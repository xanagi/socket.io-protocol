
# socket.io プロトコル

  このドキュメントでは Socket.IO プロトコルを説明します. 参照 JavaScript 実装については、
  [socket.io-parser](https://github.com/learnboost/socket.io-parser),
  [socket.io-client](https://github.com/learnboost/socket.io-client)
  および [socket.io](https://github.com/learnboost/socket.io)
  を確認してください.

## プロトコルバージョン

  **現在のプロトコルリビジョン:** `4`.

## パーサー API

### Parser#Encoder

  socket.io パケットを engine.io-transportable 形式にエンコードするオブジェクトです.
  唯一のパブリックメソッドは Encoder#encode です.

#### Encoder#encode(Object:packet, Function:callback)

  `Packet` オブジェクトを engine.io 互換エンコーディングの配列にエンコードします.
  オブジェクトが純粋な JSON であれば、配列は、ただ一つの要素である、socket.io
  でエンコードされた文字列を含みます. オブジェクトがバイナリーデータ (ArrayBuffer, Buffer,
  Blob, もしくは File) を含んでいれば、配列の最初の要素は、パケットを表すメタデータと、
  最初のパケットでバイナリーデータが保持されていたプレースホルダーを含むJSONからなる
  文字列になります.残りの要素は、後でデコードするときにプレースホルダーに埋め込むための
  生のバイナリーデータです.ぇ

  コールバック関数は、エンコードされた配列を唯一の引数として呼ばれます.
  socket.io-parser の実装では、コールバックは、トランスポートのために、配列の各要素を
  engine.io に書き出します. あらゆる実装で、配列の各要素はシーケンシャルにトランスポート
  されることが期待されています.

### Parser#Decoder

  engine.io からのデータを、完全な socket.io パケットにデコードするオブジェクトです.

  Decoder を使う場合に想定されるワークフローは、engine.io からどんなエンコードされたデータ
  を受け取った場合でも、すぐに `Decoder#add` メソッドを呼び、Decoder の 'decoded' イベントを
  リッスンして、完全にデコードされたパケットを処理するというものです.

#### Decoder#add(Object:encoding)

  engine.io トランスポートからのエンコードされたオブジェクトを、一つデコードします.
  非バイナリーパケットの場合は、一つの encoding 引数が完全なパケットを再構成するために
  使われます. パケットのタイプが `BINARY_EVENT` か `ACK` であれば、元のパケットの
  バイナリーデータの断片ごとに、1回ずつ add が追加で呼ばれることが期待されています.
  最後のエンコードされたバイナリーデータが add に渡されて初めて、完全な socket.io パケットが
  再構築されます.

  パケットが完全にデコードされた後、Decoder は、デコードされたパケットを唯一の引数として、
  'decoded' イベントを (Emitter 経由で) 発行します. このイベントのリスナーは、その
  パケットが準備完了状態になったものとして扱うべきです.

#### Decoder#destroy()

  Decoder インスタンスのリソースの後始末をします. デコード中に disconnect イベントが発生
  した場合などに、メモリリークを防ぐために呼ばれるべきです.

### Parser#types

  パケットタイプキーの配列です.

### Packet

  個々のパケットは、`nsp` キーと `type` キーを持つ、通常の `Object` として表現されます.
  `nsp` キーは、パケットが属するネームスペース ("多重化" を参照) を示します.
  `type` キーは以下のいずれか一つです:

  - `Packet#CONNECT` (`0`)
  - `Packet#DISCONNECT` (`1`)
  - `Packet#EVENT` (`2`)
  - `Packet#ACK` (`3`)
  - `Packet#ERROR` (`4`)
  - `Packet#BINARY_EVENT` (`5`)
  - `Packet#BINARY_ACK` (`6`)

#### EVENT

  - `data` (`Array`) 引数のリストです. 最初の要素はイベント名です. 引数リストは、
    任意のサイズのオブジェクトや配列など、`JSON` をデコードして得られるどんなタイプの
    フィールドでも含むことができます.

  - `id` (`Number`) `id` 識別子があれば、このイベントの受信を通知してもらう
    ことを、サーバーが期待していることを示します.

#### BINARY_EVENT

  - `data` (`Array`) `EVENT` `data` 参照. ただし追加で、どの引数も、非JSONの任意の
    バイナリーデータを含むことができます. エンコーディングの場合、バイナリーデータは
    Buffer, ArrayBuffer, Blob, File のいずれかであると考えられます.
    デコーディングの場合は、全てのバイナリーデータのサーバー側は Buffer です。モダンな
    クライアントでは、バイナリーデータは ArrayBuffer です. バイナリーをサポートしていない
    古いブラウザでは、それぞれのバイナリーデータの要素は、`{base64: true, data: <base64_bin_encoding>}`
    のようなオブジェクトに置き換えられます. `BINARY_EVENT` もしくは `ACK` パケットが
    最初にデコードされるときは、全てのバイナリーデータの要素はプレースホルダーであり、
    追加で `Decoder#add` が呼ばれるたびに埋められていきます.
  - `id` (`Number`) `EVENT` `id` 参照.

#### ACK

  - `data` (`Array`) `EVENT` `data` 参照. 上記の `EVENT` と同じように文字列としてエンコードされます.
    ACK 関数がバイナリーデータとともに呼ばれない場合に使われるべきです.
  - `id` (`Number`) `EVENT` `id` 参照.

#### BINARY_ACK

  - `data` (`Array`) `ACK` `data` 参照. ACK 関数の引数がバイナリーデータを含んでいる
    場合に使われます. パケットを上述の `BINARY_EVENT` スタイルでエンコードします.
  - `id` (`Number`) `EVENT` `id` 参照.

#### ERROR

  - `data` (`Mixed`) エラーデータ

## トランスポート

  socket.io プロトコルは、様々なトランスポート上で配信されることができます.
  [socket.io-client](http://github.com/learnboost/socket.io-client)
  は、ブラウザと Node.JS 用のプロトコルを
  [engine.io-client](http://github.com/learnboost/engine.io-client)
  の上に、実装したものです.

  [socket.io](http://github.com/learnboost/socket.io) は、
  [engine.io](http://github.com/learnboost/engine.io)
  の上の、プロトコルのサーバー実装です.

## 多重化

  Socket.IO は多重化をビルトインでサポートしています. これは、それぞれのパケットは、
  パス文字列 (`/this` のような) で特定できるある名前空間に、必ず属している
  ということを意味します. `Packet` オブジェクト内の、パス文字列に対応するキーは、`nsp` です.

  socket.io のトランスポート接続が確立されたら、`/` 名前空間への接続を試みることが想定
  されています (つまり、サーバーは、クライアントが　`/` 名前空間に `CONNECT` パケットを
  送信したかのように振舞います).

  同じトランスポートの下にある複数のソケットの多重化をサポートするために、クライアントは
  任意の名前空間のURI (例えば `/another`) に、追加で `CONNECT` パケットを送信する
  ことができます.

  サーバーが、その名前空間への `CONNECT` パケットに応答したら、多重化されたソケットが
  接続されたとみなされます.

  あるいは、サーバーから多重化されたソケットの接続エラー (認証エラーなど) が発生したことを
  示す、 `ERROR` パケットが返ってくることもあります.
  関連するエラーペイロードは、エラーによって様々で、ユーザー定義であることもあります.

  ある `nsp` に対する `CONNECT` パケットがサーバーに受信されたら、クライアントは
  `EVENT` パケットの送受信をすることができるようになります. サーバーもしくはクライアントが
   `id` フィールド付きの `EVENT` パケットを受け取ったら、そのパケットが受信されたことを
  保証するために、`ACK` パケットが送信されることが期待されます.

## License

MIT
