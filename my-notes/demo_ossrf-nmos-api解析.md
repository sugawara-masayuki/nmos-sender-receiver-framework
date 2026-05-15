# Demo ossrf-nmos-api　解析.md
- ./build/Debug/cpp/demos/ossrf-nmos-api/ossrf-nmos-api -f ./cpp/demos/ossrf-nmos-api/config/nmos_config.json

```
この例では、NMOS側とGStreamer側の両方で、1つのビデオ/RAWレシーバーと2つのビデオ/RAWセンダーを作成する方法を示します。レシーバーはどちらのセンダーにも接続できるため、異なる出力を確認できます。NMOSオーディオリソースを作成することは可能ですが、GStreamerのオーディオサポートはまだ実装されていません。
```
- nmos-registryには現れる。
- 接続制御するとどうなる？
    - 見た目は正常に動いている。
    - 送受すると何が起きるかが分からない。

## 関係する（可能性がある）設定ファイル：コンテナの方
- /cpp/demos/ossrf-nmos-api/config/nmos_config.json
- /cpp/demos/config/nmos_plugin_node_config_receiver.json
- /cpp/demos/config/nmos_plugin_node_config_sender.jso
- /cpp/demos/nmos-cpp-node/config.jsonの設定は？

- アドレスの統一 ホストの実アドレス？

- docker networkはどうなってる？
    - ossrf-devはhost network
    - なので、コンテナはホストが持っているアドレスを使わなければならない
        - 127.0.0.1
        - 10.190.9.167 全てこれ？
        - 172.~
    - 10.190.9.167に統一してmock動作はOK
- 実動は、ReceiverはGSteamerでautovideosinkになっている模様
- ホストに現れないのはX11の問題か？
- xhost +local:docker
- yaml DISPLAY: -> DISPLAY=

## ソースファイル
- /cpp/demos/ossrf-nmos-api/
    - main.cpp — メインソースファイル
    - CMakeLists.txt — ビルド定義
    - nmos_config.json — 設定ファイル

- 全体解析
````
Read [](file:///home/nmos/nmos-sender-receiver-framework/cpp/demos/ossrf-nmos-api/main.cpp#1-1), lines 1 to 100

Read [](file:///home/nmos/nmos-sender-receiver-framework/cpp/demos/ossrf-nmos-api/CMakeLists.txt)

Read [](file:///home/nmos/nmos-sender-receiver-framework/cpp/demos/ossrf-nmos-api/main.cpp#100-100), lines 100 to 184

Read [](file:///home/nmos/nmos-sender-receiver-framework/cpp/demos/ossrf-nmos-api/config/nmos_config.json#1-1), lines 1 to 100

Read [](file:///home/nmos/nmos-sender-receiver-framework/cpp/demos/ossrf-nmos-api/config/nmos_config.json#100-100), lines 100 to 118

このプログラムの動作を以下に解析します。

---

## `ossrf-nmos-api` 動作解析

### 概要

NMOS（Networked Media Open Specifications）に準拠した送受信デモプログラムです。NMOSレジストリにノード・デバイスを登録し、GStreamerを使ったメディアの送信・受信を行います。

---

### 起動フロー

```
main()
 └─ 引数: -f <config_file> が必須
 └─ run() → load_configuration_from_file() でJSONを読込
          → bisect::gst::initializer() でGStreamer初期化
          → go() で本処理
```

---

### `go()` の処理シーケンス

#### 1. NMOSクライアント初期化
- 設定JSONから `node.id` と `node.configuration` を取り出す
- `nmos_client_t::create()` でNMOSクライアントを生成 → NMOSレジストリへの登録が行われる
- `nmos_client->add_device()` でデバイスをNMOSに登録

#### 2. レシーバー登録（最大2つ）
- 設定の `receivers` 配列を順番に処理
- 各レシーバーについて `nmos_client->add_receiver()` を呼びNMOSに登録
- コールバックが設定され、接続要求（SDP受信）時に `create_gst_receiver_plugin()` でGStreamerパイプラインを起動

#### 3. センダー登録（最大2つ）
- 設定の `senders` 配列を順番に処理
- 各センダーについて `nmos_client->add_sender()` を呼びNMOSに登録
- `create_gst_sender_plugin()` でGStreamerパイプラインを即時起動
  - センダー1: 25 fps
  - センダー2: 15 fps

#### 4. インタラクティブ停止シーケンス（キー入力待ち）
```
[キー入力] → センダー停止 (remove_sender)
[キー入力] → レシーバー停止 (remove_receiver)
[キー入力] → デバイス削除 (remove_resource)
```

---

### 設定ファイル (nmos_config.json) の内容

| 項目 | 内容 |
|------|------|
| ノード | ID: `0aad3458-...`、NMOSレジストリ: `10.190.9.167:8010` |
| デバイス | ID: `b9b85f97-...`、ラベル: `OSSRF Device` |
| レシーバー1 | `video/raw`、インターフェース: `eno1 (10.190.9.167)` |
| レシーバー2 | `audio/L24`、インターフェース: `eno1 (10.190.9.167)` |
| センダー1 | `video/raw`、640x480 YCbCr-4:2:2 50fps → マルチキャスト `239.10.10.10:5004` |
| センダー2 | `audio/L24`、2ch 48kHz → マルチキャスト `239.10.10.11:5005` |

---

### 使用ライブラリ

- **`ossrf_nmos_api`** — NMOSクライアント（登録・管理）
- **`ossrf_gstreamer_api`** — GStreamer送受信プラグイン
- **`bisect_nmoscpp`** — NMOSノード設定構造体
- **`bisect_expected`** / **`bisect_json`** — エラー処理・JSON操作ユーティリティ
- **`nlohmann_json`** — JSONパース
````

- レシーバーパイプライン解析
````
映像レシーバー（ST 2110-20）のGStreamerパイプラインを解析します。

---

## 映像レシーバー GStreamer パイプライン解析

### パイプライン構成図

```
udpsrc → rtpjitterbuffer → queue1 → rtpvrawdepay → queue2 → videoconvert → autovideosink
```

---

### 各エレメント詳細

#### 1. `udpsrc` — UDP受信
- **役割**: マルチキャストUDPパケットを受信
- **設定パラメータ** (SDPから取得):
  - `address` — 送信先IPアドレス（マルチキャストアドレス、例: `239.10.10.10`）
  - `port` — 送信先ポート（例: `5004`）
  - `multicast-iface` — 受信インターフェース（例: `eno1`）
  - `auto-multicast` = `TRUE`
- **Caps（メディア形式指定）**:
  ```
  application/x-rtp,
    media=video,
    clock-rate=90000,
    encoding-name=RAW,
    width=<SDPから>,
    height=<SDPから>,
    sampling=<SDPから e.g. YCbCr-4:2:2>,
    depth=10
  ```

#### 2. `rtpjitterbuffer` — ジッターバッファ
- **役割**: ネットワーク揺らぎ（ジッター）を吸収し、RTPパケットを再順序付け

#### 3. `queue` (queue1) — バッファキュー1
- **役割**: jitterbuffer と depay の間のスレッド分離
- `max-size-time` = 200,000 ns（200 µs）
- `max-size-buffers` = 0（無制限）
- `max-size-bytes` = 0（無制限）

#### 4. `rtpvrawdepay` — RTPデペイロード
- **役割**: RFC 4175 に従い、RTPパケットから生のビデオフレームデータを取り出す（デペイロード）

#### 5. `queue` (queue2) — バッファキュー2
- **役割**: depay と videoconvert の間のスレッド分離
- queue1 と同じ設定

#### 6. `videoconvert` — 色空間変換
- **役割**: YCbCr などの入力フォーマットをシンクが受け付けるフォーマットに変換

#### 7. `autovideosink` — 映像表示
- **役割**: 利用可能なビデオ出力（X11, Wayland, OpenGL等）を自動選択して映像を表示

---

### データフロー

```
NMOSコントローラ
    │ SDP (接続情報)
    ▼
receiver_activation_callback()
    │ SDPパース → IPアドレス・ポート・映像フォーマット取得
    ▼
create_gst_receiver_plugin()
    │ capabilities == "video/raw" → create_gst_st2110_20_plugin()
    ▼
create_gstreamer_pipeline()
    │ UDP受信 → RTPデペイロード → 表示
    ▼
pipeline_.run_loop()  ← 非同期ループで実行継続
```

---

### 規格対応

| 項目 | 内容 |
|------|------|
| 映像規格 | **SMPTE ST 2110-20** (非圧縮映像over IP) |
| RTPペイロード | **RFC 4175** (Raw Video) |
| クロックレート | **90,000 Hz** (映像RTP標準) |
| ビット深度 | **10bit** 固定 |
| 色空間 | SDPから動的取得（例: `YCbCr-4:2:2`） |

---

### 注意点

- `capabilities` の先頭要素のみチェックしており、複数ケイパビリティには未対応（TODO コメントあり）
- `autovideosink` を使用しているため、映像出力先は実行環境に依存（GUI不要環境では失敗する可能性あり）
- セカンダリネットワーク（冗長系）の `receiver_settings::secondary` は構造体に定義されているが、現パイプラインではプライマリのみ使用
``
````

- GSreamerパイプラインソース
```
cpp/libs/ossrf_gstreamer_api/lib/src/
├── sender/
│   ├── sender_plugin.cpp          ← エントリポイント
│   ├── st2110_20_sender_plugin.cpp ← 映像パイプライン
│   ├── st2110_20_sender_plugin.h
│   ├── st2110_30_sender_plugin.cpp ← 音声パイプライン
│   └── st2110_30_sender_plugin.h
└── receiver/
    ├── receiver_plugin.cpp          ← エントリポイント
    ├── st2110_20_receiver_plugin.cpp ← 映像パイプライン
    ├── st2110_20_receiver_plugin.h
    ├── st2110_30_receiver_plugin.cpp ← 音声パイプライン
    └── st2110_30_receiver_plugin.h
```
- パイプライン内容
````
はい。`gst-launch-1.0` のようなパイプライン文字列ではなく、GStreamerの **C API** を使って各エレメントをプログラム的に生成・接続しています。

```cpp
// 各エレメントを個別に生成
auto* source       = gst_element_factory_make("udpsrc", NULL);
auto* jitter_buffer = gst_element_factory_make("rtpjitterbuffer", NULL);
auto* depay        = gst_element_factory_make("rtpvrawdepay", NULL);
auto* videoconvert = gst_element_factory_make("videoconvert", NULL);
auto* sink         = gst_element_factory_make("autovideosink", NULL);

// パイプラインに追加してリンク
gst_bin_add_many(GST_BIN(pipeline), source, ...);
gst_element_link_many(source, jitter_buffer, queue1, depay, queue2, videoconvert, sink, NULL);
```

`gst-launch-1.0` で書くと等価なのは:

```bash
gst-launch-1.0 udpsrc address=239.10.10.10 port=5004 ... \
  ! rtpjitterbuffer \
  ! queue \
  ! rtpvrawdepay \
  ! queue \
  ! videoconvert \
  ! autovideosink
```

C APIで書くことで、パラメータを実行時に動的に設定したり、コールバックでパイプラインを制御したりできる利点があります。
````

- センダーパイプライン解析
````
映像センダー（ST 2110-20）のGStreamerパイプラインを解析します。

---

## 映像センダー GStreamer パイプライン解析

### パイプライン構成図

```
videotestsrc → capsfilter → queue → rtpvrawpay → udpsink
```

---

### 各エレメント詳細

#### 1. `videotestsrc` — テスト映像ソース
- **役割**: テストパターンの映像フレームを生成（実カメラの代わり）
- **設定パラメータ**:
  - `pattern` — テストパターンのenum値（`videotestsrc` の `pattern` プロパティ）
    - センダー1: `pattern=25`（`main.cpp` 118行目）
    - センダー2: `pattern=15`（`main.cpp` 124行目）
  - フレームレートは capsfilter 未設定のため `videotestsrc` のデフォルト（30fps）に従う

> **注意**: `main.cpp` の引数 `25` / `15` は **fps ではなく `videotestsrc` の pattern enum値**。  
> 実際のフレームレートは `nmos_config.json` の `frame_rate` が `video_info_t` に読み込まれるが、  
> 現パイプラインでは capsfilter に framerate が設定されていないため、ネゴシエーションに委ねられる。

#### 2. `capsfilter` — メディア形式制約
- **役割**: `videotestsrc` の出力フォーマットを強制指定する
- **設定 Caps**:
  ```
  video/x-raw,
    format=UYVP,
    width=<config から>,
    height=<config から>
  ```
  - `UYVP` = 10bit packed YCbCr 4:2:2（ST 2110-20 の標準フォーマット）
  - width / height は `nmos_config.json` の sender の `width` / `height` から取得

#### 3. `queue` — バッファキュー
- **役割**: capsfilter と rtpvrawpay の間のスレッド分離
- `max-size-time` = 200,000 ns（200 µs）
- `max-size-buffers` = 0（無制限）
- `max-size-bytes` = 0（無制限）

#### 4. `rtpvrawpay` — RTPペイロード化
- **役割**: RFC 4175 に従い、生ビデオフレームを RTP パケットにカプセル化（ペイロード化）

#### 5. `udpsink` — UDP送信
- **役割**: RTP パケットをマルチキャスト UDP で送出
- **設定パラメータ** (`nmos_config.json` から取得):
  - `host` — 送信先IPアドレス（例: `239.10.10.10`）
  - `port` — 送信先ポート（例: `5004`）
  - `auto-multicast` = `TRUE`
  - `multicast-iface` — 送出インターフェース名（例: `eno1`）

---

### データフロー

```
nmos_config.json（senders配列）
    │ メディア情報（width, height, format, network など）
    ▼
create_gst_sender_plugin(config_json, pattern)
    │ translate_json() → sender_settings + video_info_t
    ▼
create_gst_st2110_20_plugin(settings, format, pattern)
    ▼
create_gstreamer_pipeline(pattern)
    │ テスト映像生成 → フォーマット変換 → RTPペイロード化 → UDP送出
    ▼
pipeline_.run_loop()  ← 非同期ループで継続送信
```

---

### `gst-launch-1.0` 等価コマンド

```bash
gst-launch-1.0 \
  videotestsrc pattern=25 \
  ! capsfilter caps="video/x-raw,format=UYVP,width=640,height=480" \
  ! queue max-size-time=200000 max-size-buffers=0 max-size-bytes=0 \
  ! rtpvrawpay \
  ! udpsink host=239.10.10.10 port=5004 auto-multicast=true multicast-iface=eno1
```

---

### 規格対応

| 項目 | 内容 |
|------|------|
| 映像規格 | **SMPTE ST 2110-20**（非圧縮映像 over IP） |
| RTPペイロード | **RFC 4175**（Raw Video） |
| ビット深度 | **10bit packed**（`UYVP` フォーマット） |
| 色空間 | **YCbCr 4:2:2** |
| 映像ソース | `videotestsrc`（テストパターン） |

---

### 注意点

- `videotestsrc` はテスト用ソースのため、実運用では実カメラ入力（`v4l2src` 等）に差し替える必要がある
- `capsfilter` に framerate が設定されていないため、実際の送出レートは GStreamer のネゴシエーション依存
- `video_info_t` に `exact_framerate` / `chroma_sub_sampling` / `structure`（interlace mode）が含まれるが、  
  現パイプラインでは capsfilter に反映されていない（将来の拡張余地）
- セカンダリネットワーク（冗長系）は未実装（プライマリのみ使用）
````

