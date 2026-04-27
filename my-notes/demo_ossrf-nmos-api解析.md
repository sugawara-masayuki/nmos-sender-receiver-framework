# Demo ossrf-nmos-api　解析.md
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
    - host network
    - なので、コンテナはホストが持っているアドレスを使わなければならない
        - 127.0.0.1
        - 10.190.9.167 全てこれ？
        - 172.~
    - 10.190.9.7に統一してmock動作はOK
- 実動は、