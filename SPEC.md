# Cipherstream — 個人用暗号化動画配信仕様

## 1. この文書の位置付け

現在のローカル実装（Python CLI、AWS構築スクリプト、静的プレーヤー、テスト）に基づく仕様書。
**SPEC.md だけをリポジトリへ保存しても、実装の再作成に必要なデータ形式・動作・構築手順・制約が分かること**を目的とする。
以下のファイル名は現実装の構成を示すものであり、この文書だけを保存したリポジトリにコードが含まれることは意味しない。
実行手順は対応する実装ファイルを用意した後に使用する。

| ファイル | 役割 |
| --- | --- |
| `video.py` | `upload` / `delete` / `estimate` のPython CLI |
| `setup_aws.sh` | 初回専用のS3・CloudFront構築 |
| `requirements.txt` | Python依存 |
| `docs/index.html` | 日本語の動画一覧・プレーヤー・設定入力モーダル |
| `docs/app.js` | 設定読込、一覧、選択、キーボード操作、エラー表示 |
| `docs/player.js` | 復号、データ検証、MSE再生・シーク・バッファ管理 |
| `docs/style.css` | ダークテーマのレスポンシブUI |
| `tests/test_video.py` | 暗号化、アップロード、削除の検証 |
| `tests/test_estimate.py` | 時間見積もりの検証 |
| `tests/test_setup_aws.py` | AWS CLIを偽装した構築設定の検証 |
| `tests/player_fixture.py`, `tests/test_player.mjs` | 合成動画を使ったブラウザテスト |

個人が順番に操作する最小構成とし、過剰な抽象化を避ける。

## 2. 目的・信頼モデル

AWSを暗号文のストレージ/CDNとして利用し、**動画・catalog・manifestの平文と復号鍵をAWSに渡さない**。
ローカルPCで変換・暗号化し、ブラウザ内で復号して再生する。

```text
ローカル動画 → ffmpeg → fMP4 → AES-256-GCM → 非公開S3
                                                 ↓ OAC
                                             CloudFront
                                                 ↓ 暗号文
GitHub Pages → HTML / CSS / JavaScript → ブラウザで復号 → MSE → <video>
```

- ローカルPC、ブラウザ、GitHub Pagesから配信されるコードを信頼する。
- AWSには暗号文、nonce、認証タグ、UUID、オブジェクトのサイズ・個数、アクセス時刻などが見える。
- AWSにmaster key、動画の平文、タイトルを含む平文catalog、平文manifestを保存しない。
- CloudFront URLは秘密ではなく、第三者が暗号文を取得できることは許容する。閲覧者の認証や署名付きURLは使用しない。
- CORSは閲覧者の認証ではない。機密性は鍵の秘匿とAES-256-GCMで確保する。
- master key、鍵付きURL、AWS認証情報、平文動画をGitへコミットしない。
- 鍵を失うと復号できない。ローカルで安全にバックアップする。同じライブラリは同じ鍵を使い続ける。

## 3. 暗号化・鍵の互換仕様

master keyは暗号学的乱数による32バイトの生データ。パスワード、KDF、動画ごとの派生鍵は使用せず、全オブジェクトに同一の鍵を使う。

```text
暗号化オブジェクト = 12-byte nonce || ciphertext || 16-byte GCM tag
暗号方式           = AES-256-GCM
AAD                = なし
最小長             = 28 bytes
```

各暗号化で新しいランダムnonceを生成する。同じ鍵でnonceを再利用しない。JSONはUTF-8、非ASCII文字を保持し、余分な空白なしでシリアライズする。

```python
nonce = os.urandom(12)
encrypted = nonce + AESGCM(key).encrypt(nonce, plaintext, None)
plaintext = AESGCM(key).decrypt(encrypted[:12], encrypted[12:], None)
```

ブラウザではWeb CryptoのAES-GCM、`tagLength: 128`を使う。鍵は`extractable: false`、用途`["decrypt"]`でimportする。
認証失敗・短すぎる暗号文はエラーにする。鍵が誤っている場合にcatalogを初期化・上書きしない。

### ブラウザへの設定入力

```text
https://<URL>/#key=<BASE64URL_KEY>&cdn=https://<distribution>.cloudfront.net
```

- `key`と`cdn`はURL fragmentから`URLSearchParams`で読む。query parameterは使わない。
- 鍵は32バイトのBase64URL表現。43文字の`A-Z a-z 0-9 _ -`と、任意の末尾`=`を許容する。
- `cdn`はHTTPS origin。認証情報、query、fragment、ルート以外のパスは禁止。末尾`/`は許容する。現実装はホスト名をCloudFrontドメインに限定していない。
- 未指定または空の項目だけを一つのモーダルで入力する。指定済みの値はfragmentのものを使う。
- モーダルはEscで閉じない。送信中は二重送信を防ぎ、catalogの取得・復号・検証成功後に閉じて入力欄を空にする。失敗はモーダル内に表示する。
- 入力値をURLや永続ストレージへ書き戻さない。鍵・復号結果をlocalStorage、sessionStorage、IndexedDBへ保存しない。
- fragmentはHTTPリクエストに送信されない。ただし鍵付きURLは履歴・ブックマークに残る秘密情報として扱う。fragment内の鍵を自動除去する処理はない。
- fragment変更時はページを再読込する。不正だが空でないfragment値は自動的に入力モーダルへ切り替えず、URLの修正が必要。

## 4. S3オブジェクトとデータ形式

```text
catalog.enc
videos/<UUID-v4>/manifest.enc
videos/<UUID-v4>/init.enc
videos/<UUID-v4>/000000.enc
videos/<UUID-v4>/000001.enc
...
```

動画ごとに新規UUID v4を生成し、削除後も再利用しない。動画配下はimmutableとして扱う。
すべて`Content-Type: application/octet-stream`で保存する。

| 対象 | Cache-Control | 更新時のinvalidation |
| --- | --- | --- |
| `catalog.enc` | `no-cache` | `/catalog.enc`のみ |
| 動画配下の全オブジェクト | `public, max-age=31536000, immutable` | 行わない |

### Catalog

復号後はJSON配列。空ライブラリは`[]`。

```json
[
  {
    "id": "e536b987-3b0d-43a2-9d48-123456789abc",
    "title": "北海道旅行",
    "manifest": "/videos/e536b987-3b0d-43a2-9d48-123456789abc/manifest.enc"
  }
]
```

`title`は入力ファイルのbasenameから最後の拡張子を除いた値（Pythonの`Path.stem`）。追加順を保持する。
CLIはS3の`NoSuchKey`に限り空catalogとして扱い、アクセス拒否・復号失敗・形式不正を欠損とみなさない。
CLIの形式検証は配列・各要素がdict・`id/title/manifest`が文字列であること。
ブラウザはさらに小文字のUUID v4と、`manifest === /videos/<id>/manifest.enc`を検証する。
初回構築ではcatalogを作らないため、最初のupload前のブラウザでは取得エラーになる。

### Manifest version 1

```json
{
  "version": 1,
  "mimeType": "video/mp4; codecs=\"avc1.640028, mp4a.40.2\"",
  "duration": 8.5,
  "init": "init.enc",
  "segments": [
    {"file": "000000.enc", "start": 0.0, "duration": 4.0},
    {"file": "000001.enc", "start": 4.0, "duration": 4.0},
    {"file": "000002.enc", "start": 8.0, "duration": 0.5}
  ]
}
```

codec値は例であり、実際の出力から決める。時刻の単位は秒。

- `init`は`init.enc`固定。segment名は6桁以上の数字と`.enc`。
- `start`は先頭0から各実測durationの累積値。全体durationはsegment durationの合計。
- manifest生成時はHLSの`#EXTINF`をDecimalで累積してからJSON数値へ変換し、浮動小数の累積誤差を避ける。
- 正の有限duration、segmentファイルの存在、segment名、非空リスト、`#EXT-X-ENDLIST`、未完のEXTINFがないことを確認する。
- ブラウザはversion、MIME文字列、init名、正の有限duration、非空segments、ファイル名の重複・パス逸脱を検証する。
- ブラウザでsegment開始時刻と累積終端の誤差は0.001秒以内、合計と全体durationの誤差は0.01秒以内とする。

## 5. 動画変換

ffprobeを使わず、ffmpegのHLS/fMP4出力から独自manifestを作る。単一画質のみ。

| 項目 | 設定 |
| --- | --- |
| 入力トラック | 最初の映像`0:v:0`、最初の音声`0:a:0?`（音声なし可） |
| 映像 | `libx264`, CRF 22, `yuv420p` |
| サイズ | `pad=ceil(iw/2)*2:ceil(ih/2)*2`で奇数寸法を偶数化 |
| GOP | `-flags +cgop`, `-force_key_frames expr:gte(t,n_forced*4)`, `-sc_threshold 0` |
| 音声 | `aac`, `aac_low`, 128 kbps, 2ch, 48000 Hz |
| メタデータ | `-map_metadata -1`, `-map_chapters -1` |
| 分割 | HLS, `fmp4`, `-hls_time 4`, `vod`, `independent_segments` |
| 出力 | `init.mp4`, `%06d.m4s`, `tmp.m3u8` |

ffmpegは`-hide_banner -loglevel warning -stats -nostdin -y`で実行する。H.264 levelを固定せず、presetも明示指定しない。
約4秒のsegmentを生成するが、境界や末尾の正確な時間はHLS出力を採用する。
字幕・追加音声・追加映像は配信しない。HLS playlist自体もアップロードしない。

`init.mp4`のISO BMFF boxを解析し、`moov/trak/mdia/minf/stbl/stsd/avc1/avcC`から
profile・compatibility・levelの3バイトを読み、`avc1.<6桁の小文字hex>`を生成する。
通常サイズ、拡張64-bitサイズ、末尾までのサイズ0のboxに対応し、不正サイズを拒否する。
H.264トラックはちょうど1本を要求する。`mp4a`があれば設定済みAAC-LCに対応する`mp4a.40.2`を加える。
音声なしではAAC codecを含めない。

平文の変換結果はローカルのTemporaryDirectoryに全体を保持し、通常終了・例外・処理可能な中断時に削除する。
そのため変換後の動画全体を置ける空き容量が必要。各ファイル単位でメモリへ読み込み、暗号化して送信する。
強制終了や電源断時の一時ファイル除去は保証しない。

## 6. 動画管理CLI

Python 3.9以上、依存は以下。upload/estimateはlibx264・AAC対応ffmpegも必要。deleteには不要。

```text
boto3>=1.37.0,<2
cryptography>=42
```

動画管理のAWSアクセスはboto3のみ。AWS CLIは初回構築スクリプトだけで使用する。
認証はboto3標準設定（環境変数、`AWS_PROFILE`、IAMロールなど）。リージョンは`ap-northeast-1`固定。

```bash
python3 video.py upload <video-file> [--config aws-outputs.json] [--key-file master.key]
python3 video.py delete <video-id> [--config aws-outputs.json] [--key-file master.key]
python3 video.py estimate <video-file> [--max-seconds 60]
```

設定・鍵の既定パスはカレントディレクトリ基準。設定JSONの`BucketName`、`DistributionId`は空でない文字列を必須とする。
`Region`があってもCLIのリージョン固定値を変更しない。鍵ファイルは正確に32バイトを要求する。
成功時は終了コード0、失敗・中断は1。upload/deleteは成功UUIDを標準出力へ、進捗・診断を標準エラーへ出す。

### Upload

1. 入力ファイル・ffmpeg・鍵・設定を確認する。
2. **エンコードや書込の前に**S3からcatalogを取得・復号・検証する。
3. UUIDを生成し、UUIDとS3出力先を標準エラーへ表示する。
4. ローカルで全体をエンコードしてmanifestを生成する。
5. init、segment順、manifestの順に、個別に暗号化してS3へputする。
6. 全動画オブジェクトの保存後にcatalogへ項目を追加し、再暗号化して保存する。
7. 新規UUIDのCallerReferenceで`/catalog.enc` invalidationを要求する。完了は待たない。

ffmpegのtime・frame・speedなどを標準エラーに表示し、アップロードはsegment件数を表示する。
全体のパーセント・残り時間表示、uploadの再開機能はない。

### Delete

1. IDをUUIDとして解析し、version 4を要求して標準の小文字表記へ正規化する。
2. catalogを復号・検証して対象が存在することを確認する。未知のIDは削除しない。
3. `videos/<id>/`を`list_objects_v2`のpaginatorで最大1,000件ずつ列挙し、`delete_objects`で削除する。
4. 応答の`Errors`も確認し、全件成功後だけcatalogから対象を除いて保存する。
5. `/catalog.enc`をinvalidateする。動画本体のキャッシュは残す。

prefixが空でもcatalogに対象が残っていれば処理を完了できる。
削除直後でも既存の動画URLと鍵を持つ利用者はCloudFrontキャッシュから再生できる場合があり、即時失効の仕組みではない。

### Estimate

AWS・設定ファイル・master keyにアクセスせず、uploadと同じ設定の試験エンコードでffmpeg所要時間を見積もる。
Pythonモジュールのimportには上記Python依存が必要。

- ffmpegで入力を開き、`-map 0:v:0 -frames:v 0 -an -f null -`の出力の`Duration:`を読む。情報取得は最大10秒。長さ不明・非正ならエラー。
- 全長Dが15秒以下なら全体を1回。それ以外は長さL=`min(15, D/3)`を、開始位置`0`, `(D-L)/2`, `D-L`の3箇所で試す。
- `--max-seconds`は既定60秒、3以上の有限数。情報取得後の試験時間予算であり、コマンド全体の厳密な上限ではない。
- 残り予算を残り場面数で等分し、入力前の`-ss`と出力の`-t`で試験する。
- `-stats`を外し、`-nostats -progress pipe:1`の完結した進捗レポートから`out_time`とframeを読む。
- 時間超過はffmpegを停止・回収し、得られた進捗を使用する。処理秒数は試験長を上限とし、frameや進捗のない場面は除外。全場面が測定不能ならエラー。
- 推定秒数=`D × 有効場面の実測経過秒数合計 / 処理動画秒数合計`。速度はその逆比。
- 複数の有効場面があれば各場面の比率による最短〜最長も表示するが、信頼区間ではない。
- 起動・シーク・終了コスト、場面差、長時間処理時のCPU速度変動で誤差が出る。暗号化・S3転送時間は含めない。

## 7. 更新の整合性と復旧

upload/deleteは単一の操作者が直列に実行する。catalog更新のロック、条件付き書込、競合検知、トランザクション、アプリ独自の再試行はない。
同時実行するとcatalogの更新を失う可能性がある。

| 状況 | 現在の挙動・復旧方法 |
| --- | --- |
| catalogの復号・形式検証失敗、アクセス拒否 | 上書きせず停止。鍵・権限・データを確認 |
| uploadの途中失敗 | 部分的な暗号文がUUID配下に残り得る。自動削除しない。表示されたS3出力先を確認して整理 |
| 動画保存後にcatalog保存失敗 | catalogから参照されない動画が残り得る。再uploadは別UUIDになる |
| catalogにない孤立オブジェクト | 通常のdelete CLIでは削除不可。表示済みprefixを確認し別途整理 |
| deleteの一部失敗 | catalogは残す。原因解消後に同じIDで再実行 |
| 動画削除後にcatalog保存失敗 | 同じIDでdeleteを再実行可能 |
| catalog保存成功後、invalidationだけ失敗 | 変更は保存済みと通知。upload/deleteを繰り返さず、CloudFrontで`/catalog.enc`を無効化 |
| invalidation受付成功 | 反映完了は待たないため、一覧が一時的に古い場合がある |

## 8. AWS初回構築

`setup_aws.sh`はBash（`set -euo pipefail`）、AWS CLI v2、jqを使用する。
CloudFormationや一時JSONファイルは使わず、jqで生成したJSONをCLI引数へ渡す。`AWS_PAGER`を無効化する。

固定設定:

| 項目 | 値 |
| --- | --- |
| リージョン | `ap-northeast-1` |
| CORS許可origin | `https://<URL>`（パス・末尾スラッシュなし） |
| バケット名 | `cipherstream-<AWSアカウントID>` |
| アカウント取得 | `sts get-caller-identity` |

処理順はS3作成 → Public Access Block → OAC → Response Headers Policy → Distribution → Bucket Policy → 配備完了待機 → 設定JSON出力。

### S3・OAC

- S3は`BucketOwnerEnforced`でACL無効化。
- `BlockPublicAcls`, `IgnorePublicAcls`, `BlockPublicPolicy`, `RestrictPublicBuckets`をすべてtrueにする。
- OACは`OriginAccessControlOriginType=s3`, `SigningProtocol=sigv4`, `SigningBehavior=always`。旧OAIは使わない。
- Bucket PolicyのサービスPrincipalは`cloudfront.amazonaws.com`、Actionは`s3:GetObject`、Resourceは`arn:aws:s3:::<bucket>/*`。
- Conditionの`StringEquals`で`AWS:SourceArn`を作成したDistributionのARNに限定する。

### CloudFront・CORS

- originは`<bucket>.s3.ap-northeast-1.amazonaws.com`、IDは`EncryptedS3`。S3OriginConfigのOriginAccessIdentityは空、OAC IDを設定する。
- 有効化、HTTP/2、IPv6有効、標準CloudFront証明書、HTTPからHTTPSへredirect。
- 許可・キャッシュメソッドはGET/HEAD。TrustedSigners無効、圧縮無効。
- AWS管理cache policyは**CachingOptimized**、IDは`658327ea-f89d-4fab-a63d-7e88639e58f6`。コードには名前のコメントも付ける。
- CORSはS3ではなくCloudFront Response Headers Policyで付与する。
- 許可originは上記1個、メソッドGET/HEAD、headers `*`、credentials=false、OriginOverride=true。localhostは許可しない。
- catalogはCache-Controlやブラウザのno-store指定だけに依存せず、更新時に明示的にinvalidateする。

### 実行と出力

```bash
bash setup_aws.sh > aws-outputs.json
```

作成済みIDを随時標準エラーに出し、`cloudfront wait distribution-deployed`完了後に標準出力へ次のJSONを出す。
値は例示用のplaceholderであり、実アカウント情報や鍵は本仕様に保存しない。

```json
{
  "BucketName": "cipherstream-<account-id>",
  "Region": "ap-northeast-1",
  "DistributionId": "<distribution-id>",
  "CloudFrontUrl": "https://<distribution>.cloudfront.net",
  "OriginAccessControlId": "<oac-id>",
  "ResponseHeadersPolicyId": "<response-headers-policy-id>"
}
```

初回専用で冪等ではない。同名バケットが存在すると停止し、更新・自動ロールバックはしない。
失敗時は標準エラーのIDから作成済みリソースを確認して整理する。中断・待機タイムアウトでもAWS側の配備が継続する場合がある。

構築実行者には対象操作の権限を用意する。主な操作はS3のCreateBucket、PutBucketOwnershipControls、PutBucketPublicAccessBlock、PutBucketPolicyと、
CloudFrontのCreateOriginAccessControl、CreateResponseHeadersPolicy、CreateDistribution、GetDistribution。
動画管理には対象バケットのGetObject、PutObject、ListBucket、対象DistributionのCreateInvalidation、delete時はDeleteObjectが必要。

## 9. ブラウザプレーヤー

ビルド不要のHTML/CSSとJavaScript ES modules。外部JavaScriptライブラリは使用しない。
Web Crypto、Media Source Extensions、および対象H.264/AAC codecを扱えるブラウザが必要。配信はHTTPSとする。

### UI・取得処理

- 日本語UI、ダークテーマ、最大幅1000px、モバイル対応。ネイティブ`<video controls playsinline>`を使用する。
- 状態表示、選択中のタイトル・コピー可能なUUID、再読み込みボタン、catalog順の動画一覧、設定入力モーダルを持つ。
- タイトルなどは`textContent`で描画し、HTMLとして解釈しない。選択状態は`aria-pressed`、状態更新は`aria-live`で伝える。
- fetchは`credentials: "omit"`, `referrerPolicy: "no-referrer"`。ページにもno-referrerを設定する。
- catalogだけ`cache: "no-store"`、他は`default`。HTTP非成功はエラー。暗号文全体を取得して復号し、JSONは不正UTF-8を拒否する。
- 一覧読込後に空なら「動画がありません」と表示する。動画を選ぶとmanifestを復号・検証し、`MediaSource.isTypeSupported`を確認する。
- 自動再生を試み、拒否されたらネイティブ再生ボタンで再生できるようにする。
- 通信・復号・MSEエラーは表示し、選択中の動画全体を再読み込みできるようにする。catalogの初期読込失敗は動画再読み込みボタンの対象外。

### 再生・先読み・シーク

1. MediaSourceとobject URLを生成し、`sourceopen`を待つ。
2. manifestのMIMEで1個のSourceBufferを作り、MediaSource.durationを設定する。
3. initを取得・復号してappendする。
4. `timeupdate`と500ms間隔のタイマーで必要なsegmentを補充する。

segment開始時刻を二分探索して現在位置のindexを求める。
デコーダの境界処理のため1つ前を含め、現在segmentから3つ先までを順番に取得・復号・appendする。
SourceBufferのappend/removeは直列化し、`updateend`を待ち、error/abortを扱う。
通常は現在位置より24秒以上古いバッファをremoveする。読込済みindexだけでなく実際のbuffered範囲にsegmentの中点が残っているかも調べる。
末尾までappendしたらMediaSourceがopenの場合に`endOfStream()`を呼ぶ。

シーク時は世代番号を更新し、取得中のfetchをAbortControllerで中断する。
進行中のappendの終了を待ち、シーク先が実際にbuffered範囲内ならデータを保持する。
範囲外ならSourceBufferの状態を整理して旧バッファ・読込済み集合を消し、目的地の前のsegmentから読み直す。
旧世代の結果を追加しない。ただしシーク中に完了したappendは記録し、重複追加を避ける。

動画切替・エラー・`pagehide`ではfetchとイベントを中断し、タイマー停止、再生停止、src解除、object URL解放を行う。

### キーボード

- 右矢印で+30秒、左矢印で−10秒。0〜動画durationに丸める。
- capture phaseでネイティブ動画コントロールの矢印シークを上書きする。
- モーダル表示中、入力欄・contenteditable・textbox/slider操作中、修飾キー押下中、IME変換中は無効。
- 動画未準備、player終了済み、duration不明の場合も無効。

## 10. 再構築後の利用手順

1. 上記の実装ファイルを用意し、Python依存とffmpegをインストールする。
2. AWS認証を設定し、構築スクリプトを一度実行して`aws-outputs.json`を保存する。
3. master keyを初回のみ生成し、ローカルでバックアップする。
4. 動画をuploadする。
5. GitHub Pagesで静的ファイルを公開し、鍵とCloudFront URLを指定して開く。

```bash
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install -r requirements.txt

# 初回のみ。既存の鍵を上書きしないこと。
(umask 077; openssl rand 32 > master.key)

python3 video.py estimate ~/Videos/北海道旅行.mp4 --max-seconds 60
python3 video.py upload ~/Videos/北海道旅行.mp4
python3 video.py delete <video-id>
```

Base64URLへの変換（出力は秘密情報）:

```bash
python3 -c 'import base64; from pathlib import Path; print(base64.urlsafe_b64encode(Path("master.key").read_bytes()).rstrip(b"=").decode())'
```

GitHub Pagesの公開ブランチの`/docs`を公開元にし、独自ドメインを設定する。
別originならCloudFront Response Headers Policyも合わせる。
**SPEC.mdだけのリポジトリではプレーヤーは公開されない**。利用時は静的ファイルを別途配置・配信する必要がある。

ローカルの画面確認は鍵のあるリポジトリ全体を配信せず、静的ファイルだけを配信する。

```bash
python3 -m http.server 8000 --directory docs
```

現設定ではlocalhostから実CloudFrontへのfetchはCORSで許可されない。後述のブラウザテストはローカルfixtureで検証する。
実装を再作成する際の`.gitignore`対象は`.venv/`, `__pycache__/`, `*.pyc`, `*.key`, `aws-outputs.json`。
仕様だけを保存する運用では、コード・テスト・ローカル生成物を含めず、コミット対象を`SPEC.md`に限定する。

## 11. 検証仕様

実装を再作成した際は以下を満たすテストを用意する。AWSの本番リソースにアクセスせず、PythonではAWS操作をモックし、実ffmpegで合成動画を生成する。

```bash
python3 -m unittest discover -s tests -v
```

- 暗号化の往復、ランダムnonce、改ざん・誤鍵の拒否。誤鍵・アクセス拒否時にエンコードや書込が発生しないこと。
- 音声あり／なしのupload、codec・duration・manifestの整合性、暗号文のみの保存、catalogの最終更新、復号動画のffmpegデコード。
- 1,000件を超える削除、他動画の保持、空prefixでの再実行、部分削除失敗時のcatalog保持、不正・未知IDと誤鍵で削除しないこと。
- invalidation失敗時にも更新済みcatalogを保持すること。deleteがffmpegを要求しないこと。
- 見積もりの場面選択・重み付き計算・タイムアウト時の進捗利用・無効予算・長さ不明・一時ファイル除去・AWSや鍵なしでの実行。
- 偽のAWS CLIによる構築順序、リージョン、S3非公開設定、OAC、CORS、Bucket Policy、出力JSON、配備待機、失敗時停止。

ブラウザテストはNode.js、Playwright、Google Chrome、Python依存、ffmpegを使用する。

```bash
npm install --prefix /tmp/cipherstream-browser playwright
NODE_PATH=/tmp/cipherstream-browser/node_modules node tests/test_player.mjs
```

既定はChrome channel。`CHROME_PATH`でブラウザ実行ファイルを指定できる。
公開のテスト専用鍵と合成動画で、設定モーダル・入力値非保存・HTTPへの鍵非送信・タイトルのHTML非解釈、
先読みの制限、映像フレームの進行、バッファ内外のシーク、矢印キー、連続シーク、動画切替、音声なし、誤鍵・破損時のエラーを検証する。
テスト用の鍵を実データに使用しない。

これらは現実装の検証項目であり、本仕様書のみからテストを実行できるわけではない。
AWS上の実配備やすべてのブラウザでの動作保証を意味しない。

## 12. 現時点の対象外

複数利用者・同時更新、ユーザー認証、鍵ローテーション、パスワード/KDF、永続的な鍵保管、動画ごとの鍵、
複数画質・自動画質切替、サムネイル、字幕、複数音声、upload再開、孤立データの自動回収、
AWS構成の更新・自動撤去・ロールバック、動画キャッシュの即時失効は実装していない。
新たに実装する場合は、既存の暗号形式・データ形式との互換性と上記の信頼モデルを維持する。
