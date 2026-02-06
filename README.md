# redis module README

## 目次

- [redis とは](#redis-とは)
- [redis のインストール](#redis-のインストール)
  - [windows版redisのインストール](#windows版redisのインストール)
  - [WSL(Ubuntu)版redisのインストール](#wslubuntu版redisのインストール)
- [redis 動作確認](#redis-動作確認)
  - [(A) Windows/Ubuntuのローカルredisサーバーにアクセスする場合](#a-windowsubuntuのローカルredisサーバーにアクセスする場合)
    - [(1) Redis-Serverを起動](#1-redis-serverを起動)
    - [(2) Redis-Cliを起動](#2-redis-cliを起動)
  - [(B) WindowsからWSLのredisサーバーにアクセスする場合](#b-windowsからwslのredisサーバーにアクセスする場合)
    - [(1) Windows Operation](#1-windows-operation)
    - [(2) Windows Operation](#2-windows-operation)
    - [(3) WSL22.04 Operation](#3-wsl2204-operation)
    - [(4) Windows Operation](#4-windows-operation)
- [Redisキーを作成する](#redisキーを作成する)
  - [create_redis_key.py](#create_meridis_keyspy)
  - [使い方](#使い方)
  - [動作](#動作)
- [Redisキーを確認する](#redisキーを確認する)
- [ロボット動作を管理する](#ロボット動作を管理する)
  - [meridis_manager.py](#meridis_managerpy)
  - [使い方](#使い方-3)
  - [引数](#引数-2)
  - [動作](#動作-3)
  - [マネージャー設定](#マネージャー設定)
    - [Sim2Real](#sim2real)
    - [Real2Sim](#real2sim)
    - [Real](#real)
  - [ネットワーク設定](#ネットワーク設定)
  - [足データ設定](#足データ設定)
- [ライブラリの動作を確認する](#ライブラリの動作を確認する)
  - [redis_transfer.py](#redis_transferpy)
    - [使い方](#使い方-1)
    - [引数](#引数)
    - [動作](#動作-1)
    - [注記](#注記)
  - [例](#例)
  - [redis_receiver.py](#redis_receiverpy)
    - [使い方](#使い方-2)
    - [引数](#引数-1)
    - [動作](#動作-2)
    - [注記](#注記-1)
    - [例](#例-1)
  - [redis_plotter.py](#redis_plotterpy)
    - [使い方](#使い方-4)
    - [引数](#引数-3)
    - [動作](#動作-4)
    - [表示内容](#表示内容)
    - [注記](#注記-2)
  - [例](#例-2)
  - [udp_sender.py](#udp_senderpy)
    - [使い方](#使い方-5)
    - [引数](#引数-4)
    - [動作](#動作-5)
    - [注記](#注記-3)
  - [例](#例-3)

## redis とは

- Redis は高速なインメモリ型のキー・バリュー型データストアで、キャッシュ、メッセージブローカー、永続化ストアなど複数の用途で使われます。
- 本リポジトリでは主にロボット制御データ（Meridim 形式の配列）を一時的に共有・受渡しするために利用しています。

背景・目的:
- センサーデータや制御指令を低遅延でやり取りするために、ディスクI/Oの遅延を避けつつプロセス間で値を共有したい。
- 複数のプロセス（シミュレーション、実機インターフェース、監視ツール）が同じデータを容易に読み書きできる共通のバッファを提供する。

主な機能と特徴:
- 高速な読み書き（メモリ上での操作）によりリアルタイム性の高いデータ交換に適する。
- ハッシュ、リスト、セット、ソート済みセットなど多様なデータ構造をサポートしているため、用途に応じた格納方法を選べる。
- 簡易な永続化オプション（RDB/AOF）により、必要に応じてディスク保存も可能。
- パブリッシュ／サブスクライブ（Pub/Sub）やトランザクション、Lua スクリプトで柔軟な連携が可能。

本リポジトリでの使い方:
- Meridim 形式の配列は Redis のハッシュ（キー名例: `meridis`）に連番フィールド（"0"〜"89"）として格納し、`redis_receiver.py` / `redis_transfer.py` で読み書きします。
- 受信側はハッシュの値を数値（float）として扱い、送信側は整数や符号付き 16bit 値で格納する運用が想定されています。
- `Meridian_console.py` ではUIのチェックボックスでRedisのPub/Subを制御可能。SubでRedisからデータを読み取りロボット制御に反映、Pubでコンソールの制御データをRedisに送信します。

ユースケース:
- シミュレーション→実機: シミュレータが生成した制御コマンドを Redis に書き込み、インターフェースがそれを読み取って UDP 経由で実機へ送信する。
- 実機→シミュレーション: 実機から受け取ったセンサーデータを Redis に書き戻し、シミュレーションや可視化ツールがそれを読み取る。
- データ監視・ロギング: 別プロセスで Redis のハッシュやメトリクスをポーリングして可視化やログ保存を行う。

運用上の注意:
- ネットワーク越し・Docker越しに Redis を公開する場合は `protected-mode` や認証設定を適切に構成すること。
- 高頻度で大容量のデータを書き込む用途ではメモリ使用量に注意すること。

## redis のインストール

### Windows版Redisのインストール
参考：

1. 下記サイトからredisインストーラ(msi)をダウンロードする<br>
https://github.com/MicrosoftArchive/redis/releases
1. ダウンロードした「Redis-x64***.msi」をダブルクリックしてredisをインストールする。

### WSL(Ubuntu)版Redisのインストール
参考：
https://www.digitalocean.com/community/tutorials/how-to-install-and-secure-redis-on-ubuntu-20-04-ja

```bash
sudo apt update
sudo apt install redis-server
```

### MacOS版Redisのインストール
参考：
https://redis.io/docs/latest/operate/oss_and_stack/install/archive/install-redis/install-redis-on-mac-os/

```
brew install redis
```

# Redis 動作確認

## (A) Windows/Ubuntuのローカルredisサーバーにアクセスする場合

### (1) Redis-Serverを起動
```
redis-server
```

![Redis Server](image/redis-server.png)


### (2) Redis-Cliを起動
別ターミナルを開く
```
redis-cli

> redis-cli
127.0.0.1:6379> ping
PONG
127.0.0.1:6379> keys *
(empty list or set)
127.0.0.1:6379> exit
```

- Redisのクライアントからサーバーに"ping"を打ったとき"PONG"が返れば導通成功です
- 初回ではRedisのキーは空の状態です。初期状態にしたい場合は以下のコマンドを実行してください
```
> redis-cli
127.0.0.1:6379> flushall
...
127.0.0.1:6379> keys *
(empty list or set)
127.0.0.1:6379> exit
```




- [Redisキーを作成する](#redisキーを作成する)に移動します。


## (B) WindowsからWSLのredisサーバーにアクセスする場合

### (1) Windows Operation
```
redis-server
```

### (2) Windows Operation

```bash
> redis-cli -h 172.21.242.172
172.21.242.172:6379> ping
(error) DENIED Redis is running in protected mode because protected mode is enabled and no password is set for the default user. In this mode connections are only accepted from the loopback interface. 
1) If you want to connect from external computers to Redis you may adopt one of the following solutions: 
2) Alternatively you can just disable the protected mode by editing the Redis configuration file, and setting the protected mode option to 'no', and then restarting the server. 
3) If you started the server manually just for testing, restart it with the '--protected-mode no' option. 
4) Set up an authentication password for the default user. NOTE: You only need to do one of the above things in order for the server to start accepting connections from the outside.

127.0.0.1:6379> exit
```

### (3) WSL22.04 Operation
```
> redis-cli

127.0.0.1:6379> CONFIG GET protected-mode
1) "protected-mode"
2) "yes"

127.0.0.1:6379> CONFIG SET protected-mode no
OK

127.0.0.1:6379> CONFIG GET protected-mode
1) "protected-mode"
2) "no"

127.0.0.1:6379> exit
```

### (4) Windows Operation
```
> redis-cli -h 172.21.242.172

172.21.242.172:6379> ping
PONG

172.21.242.172:6379> keys "*"

127.0.0.1:6379> exit
```

- Redisのクライアントからサーバーに"ping"を打ったとき"PONG"が返れば導通成功です
- 初回ではRedisのキーは空の状態です
- [Redisキーを作成する](#redisキーを作成する) に移動します。


以上を毎回やるのがめんどくさいのでWSL内のconfを書き換えておくとよい。

```bash
sudo nano /etc/redis/redis.conf

# Redisへの外部サーバからの接続許可設定方法
- bind 127.0.0.1
+ bind 0.0.0.0

#サービスとして起動しておく
- supervised no
+ supervised systemd

# デフォルトはプロテクトがかかっているので外しておく
- protected-mode yes
+ protected-mode no
```


## Redisキーを作成する

### create_meridis_keys.py
- `create_meridis_keys.py` は Redisサーバーに必要なキーを初期化するためのコマンドラインスクリプトです。
- 起動時にRedisサーバーのIPアドレスを入力し、指定されたRedisキーをハッシュ形式で作成します。
- Meridisシステムのセットアップ時に使用します。

### 使い方
- Redisサーバーが起動している必要があります。
- 起動時にRedisサーバーのIPアドレスが聞かれるので、`127.0.0.1`を入れてください。異なるサブネットのRedisサーバーにアクセスする場合、適切なIPアドレスを入力してください。
- 入力されたIPアドレスでRedisサーバーに接続を試みます。
- キーの初期化は一度だけ行えば十分です。繰り返し実行しても既存キーは上書きされません。

```bash
python create_meridis_keys.py
Enter Redis server IP address: 127.0.0.1

...
All keys created.
```

### 動作
- 以下のキーを順次初期化します：
  - `meridis_sim_pub`
  - `meridis_calc_pub`
  - `meridis_console_pub`
  - `meridis_mgr_pub`
  - `meridis_mcp_pub`
- 各キーは90要素のハッシュとして作成され、全てのフィールドに値0が設定されます。
- キーが既に存在する場合はスキップし、メッセージを表示します。
- 接続に失敗した場合はエラーメッセージを表示してスキップします。

## Redisキーを確認する
```
redis-cli
127.0.0.1:6379> ping
PONG
127.0.0.1:6379> keys *
(empty list or set)
127.0.0.1:6379> keys *
1) "meridis_mcp_pub"
2) "meridis_console_pub"
3) "meridis_mgr_pub"
4) "meridis_calc_pub"
5) "meridis_sim_pub"
127.0.0.1:6379> exit
```

## ロボット動作を管理する
### meridis_manager.py

- `meridis_manager.py` は Redis と UDP 間の双方向データ転送を管理するデーモン型ユーティリティです。
- Redis のハッシュデータを Meridim90 フォーマットに変換して UDP 送信し、UDP で受信した Meridim90 データを Redis に書き戻します。
- 使用可能なRedisキー: `meridis_sim_pub`, `meridis_calc_pub`, `meridis_console_pub`, `meridis_mgr_pub`, `meridis_mcp_pub`
- シミュレーション環境と実機ロボットの間でリアルタイムデータ交換を実現する中核的なブリッジ機能を提供します。
  - シミュレーションサンプルプログラムとして、Mujocoをベースとする**merimujoco**を公開しています
  https://github.com/holypong/merimujoco

![merimujoco](image/merimujoco.png)


#### アーキテクチャ図

```mermaid
graph TD;
    subgraph MM [ブリッジ: meridis_manager]
        subgraph Redis [Redis]
            PK1[key1]
            PK2[key2]
        end
        subgraph UDP [UDP]
            REC[sub]
            TRA[pub]
        end
    end
    MCP[MCPサーバー] --> MM;
    SIM[シミュレータ] --> MM;
    ROB[リアルロボット] --> MM;
    MM --> SIM;
    MM --> ROB;
    MM --> MCP;
```

### 使い方

```bash
python meridis_manager.py --mgr MGR_FILE --network NETWORK_FILE --foot FOOT_MODE
```

### 引数

- `--mgr`（デフォルト: `mgr_sim2real.json`）: マネージャー設定 JSON ファイルのパス
- `--network`（デフォルト: `network.json`）: ネットワーク設定 JSON ファイルのパス  
- `--foot`（デフォルト: `off`）: 足部データの 1/100 スケーリング設定（`off`/`on`）

### 動作

- 起動時にマネージャー設定ファイルとネットワーク設定ファイルを読み込みます。設定ファイルが見つからない場合はエラーメッセージを表示してプログラムを終了します。
- マネージャー設定の `data_flow` に基づいて、Redis→UDP、UDP→Redis の各方向でのデータ転送を制御します。
- Redis のハッシュデータ（90要素）を読み取り、Meridim90 バイナリフォーマットに変換して UDP パケットとして送信します。
- UDP で受信した Meridim90 データをパースし、Redis のハッシュとして書き戻します。
- `--foot on` オプション時は、足部関連の特定インデックス範囲（21-47, 46-50, 51-77, 76-80）でのデータに 1/100 スケーリングを適用します。
- 転送処理は継続的なループで実行され、リアルタイムでのデータ同期を実現します。

### マネージャー設定

#### Sim2Real

- `mgr_sim2real.json`

以下は `mgr_sim2real.json` の例です（実際のファイルはリポジトリ内のものを参照してください）：

```json
{
  "redis": {
    "host": "127.0.0.1",
    "port": 6379
  },
  "redis_keys": {
    "read": "meridis_sim_pub",
    "write": "meridis_mgr_pub"
  },
  "data_flow": {
    "redis_to_udp": true,
    "udp_to_redis": false
  }
}
```

以下は典型的なデータフロー（Sim → Real）を Mermaid で表した図です。

```mermaid
sequenceDiagram
    participant MON as 監視/ログ
    participant HOST as ホスト(MCPサーバなど)
    participant R as Redis
    participant SIM as シミュレータ
    participant MMD as meridis-manager.py
    participant ROB as リアルロボット

    Note right of HOST: キー: meridis_mcp_pub
    HOST->>R: HSET meridis_mcp_pub // ホストが指令を格納
    Note right of R: キー: meridis_mcp_pub
    SIM->>R: HGETALL meridis_mcp_pub  // シミュレータが指令を取得
    Note right of R: キー: meridis_sim_pub
    SIM->>R: HSET meridis_sim_pub // シミュレータが演算結果を格納
    ROB->>MMD: UDP受信  // 実機から実行結果を受信
    Note right of R: キー: meridis_sim_pub
    MMD->>R: HGETALL meridis_sim_pub  // meridis-manager が演算結果を取得
    MMD->>ROB: UDP送信  // 実機へ演算結果を送信
```

### Real2Sim
- `mgr_real2sim.json`

以下は `mgr_real2sim.json` の例です（実際のファイルはリポジトリ内のものを参照してください）：

```json
{
  "redis": {
    "host": "127.0.0.1",
    "port": 6379
  },
  "redis_keys": {
    "read": "meridis_sim_pub",
    "write": "meridis_mgr_pub"
  },
  "data_flow": {
    "redis_to_udp": false,
    "udp_to_redis": true
  }
}
```

以下は典型的なデータフロー（Real → Sim）を Mermaid で表した図です。

```mermaid
sequenceDiagram
    participant MON as 監視/ログ
    participant HOST as ホスト(MCPサーバなど)
    participant R as Redis
    participant SIM as シミュレータ
    participant MMD as meridis-manager.py
    participant ROB as リアルロボット

    ROB->>MMD: UDP送信  // 実機が実行結果を送信
    Note right of R: キー: meridis_sim_pub
    MMD->>R: HGETALL meridis_sim_pub  // meridis-manager が演算結果を取得
    Note right of R: キー: meridis_mgr_pub
    MMD->>R: HSET meridis_mgr_pub  // meridis-manager が実行結果を格納
    Note right of R: キー: meridis_mgr_pub
    SIM->>R: HGETALL meridis_mgr_pub  // シミュレータが実行結果を取得
    Note right of R: キー: meridis_sim_pub
    SIM->>R: HSET meridis_sim_pub // シミュレータが演算結果を格納
```

### Real
- `mgr_mcp2real.json`

以下は `mgr_mcp2real.json` の例です（実際のファイルはリポジトリ内のものを参照してください）：

```json
{
  "redis": {
    "host": "127.0.0.1",
    "port": 6379
  },
  "redis_keys": {
    "read": "meridis_mcp_pub",
    "write": "meridis_mgr_pub"
  },
  "data_flow": {
    "redis_to_udp": true,
    "udp_to_redis": true
  }
}
```

以下は典型的なデータフロー（MCP → Real、双方向）を Mermaid で表した図です。

```mermaid
sequenceDiagram
    participant MON as 監視/ログ
    participant HOST as (MCPサーバなど)
    participant R as Redis
    participant SIM as シミュレータ
    participant MMD as meridis-manager.py
    participant ROB as リアルロボット

    Note right of HOST: キー: meridis_mcp_pub
    HOST->>R: HSET meridis_mcp_pub // ホストが指令を格納
    ROB->>MMD: UDP受信  // 実機から実行結果を受信
    Note right of R: キー: meridis_mcp_pub
    MMD->>R: HGETALL meridis_mcp_pub  // meridis-manager が指令を取得
    Note right of R: キー: meridis_mgr_pub
    MMD->>R: HSET meridis_mgr_pub  // meridis-manager が実行結果を格納
    MMD->>ROB: UDP送信  // 実機へ指令を送信
    Note right of HOST: キー: meridis_mgr_pub
    HOST->>R: HGETALL meridis_mgr_pub // ホストが実行結果を取得
    HOST->>MON: ストレージ格納 // ホストが実行結果・設定をメモリ・ファイルに格納
    HOST->>MON: ストレージ取得 // ホストが実行結果・設定をメモリ・ファイルから取得

```

### ネットワーク設定 （--network）
- udp
  - PC側からみた送信側・受信側のIP・PORTを設定してください。

```json
{
  "udp": {
    "send": {
      "ip": "192.168.0.21",
      "port": 22224
    },
    "recv": {
      "ip": "192.168.0.23",
      "port": 22222
    }
  }
}
```

### 足の設定(--foot)
足の逆運動学計算の状況をモニタリングするオプションです。足のXYZ位置を登録する処理が入っている場合のみ有用です。

- `off` (デフォルト): 送信用サーボ位置データ（index 21–80 の偶数番）を 100 倍して送信し、受信時は 1/100 に戻す処理を行う（従来動作）

- `on`: 足部分の指定範囲のみ 1/100 スケーリングを行う（コード中の範囲に準拠）

具体的な index 範囲はコードの `write_redis_data()` 内に実装されています。実際の変換ロジックや範囲を確認する場合は `meridis_manager.py` の該当関数を参照してください。

```python
  # --foot on の場合：指定されたrangeのみ1/100する
  for i in range(21, 47, 2):
      data[i] = float(data[i] / 100)
  for i in range(46, 50):   # x,y,z,(r)
      data[i] = float(data[i] / 100) 
  for i in range(51, 77, 2):
      data[i] = float(data[i] / 100)
  for i in range(76, 80):   # x,y,z,(r)
      data[i] = float(data[i] / 100)
```

## ライブラリの動作を確認する

### redis_transfer.py

- `redis_transfer.py` は Redisサーバーの指定キーにMeridian形式のハッシュデータを書き込むためのデータ転送ユーティリティです。
- 主に他のアプリケーションからライブラリとしてコールされますが、テスト用途で単体実行も可能です。
- Redis接続の動作確認やデータ書き込みのテスト機能を提供します。

#### 使い方

```bash
python redis_transfer.py [--host HOST] [--port PORT] [--key KEY]
```

#### 引数

- `--host`（デフォルト: `localhost`）: Redis サーバーのホスト名またはIPアドレス
- `--port`（デフォルト: `6379`）: Redis サーバーのポート番号
- `--key`（デフォルト: `meridis_calc_hub`）: 書き込み先となる Redis のハッシュキー名

#### 動作

- インスタンス生成時に TCP レベルの接続チェック（`socket.create_connection`）を実行し、その後 Redis の `PING` コマンドで接続確認を行います。接続タイムアウトは 0.5 秒で、到達不能なサーバーに対して迅速に失敗します。
- 指定キーが存在しない場合、90 要素のハッシュ（フィールド名: `"0"`〜`"89"`）を自動初期化します。
- `set_data()` メソッドは 90 要素の数値配列を受け取り、フィールド名を文字列化してハッシュに一括書き込みします。値は数値文字列として保存され、`redis_receiver.py` での `float()` 変換に対応します。
- テスト用の `main()` 関数では、3 回の反復処理で 90 要素のダミーデータを書き込みます。各反復で全要素を 0.1 ずつインクリメントしてデータ変化をシミュレートします。
- 書き込み処理中のエラーは適切にハンドリングされ、エラー内容が標準出力に表示されます。

#### 注記

- ハッシュのフィールド名は文字列の連番（`"0"`, `"1"`, ..., `"89"`）で統一する必要があります。受信側アプリケーションは数値順序での読み取りを前提としています。
- `connect_timeout` および `socket_timeout` パラメータは `RedisTransfer` クラスのコンストラクタで設定可能（デフォルト: 0.5秒）ですが、CLI オプションとしては現在公開されていません。
- データは Meridim90 フォーマット（90要素の数値配列）に準拠することを前提として設計されています。
- ライブラリとして使用する際は、`RedisTransfer` クラスの `set_data()` メソッドで配列データを Redis に転送できます。

### 例

```bash
# ローカルサーバーでデフォルトキーにデータ送信
python redis_transfer.py

Redis list 'meridis_calc_pub' already exists.
Starting data transfer to Redis server localhost:6379 with key 'meridis_calc_pub'
Wrote 90-element hash to 'meridis_calc_pub' (iteration 1)
Wrote 90-element hash to 'meridis_calc_pub' (iteration 2)
Wrote 90-element hash to 'meridis_calc_pub' (iteration 3)
Completed.
```

実装の詳細や利用可能なクラス・メソッドについては [redis_transfer.py](redis_transfer.py) を参照してください（`RedisTransfer`、`set_data`、`check_connection`、`initialize_hash` など）。


### redis_receiver.py

- `redis_receiver.py` は Redisサーバーの指定キーに保存されたMeridian形式のハッシュデータを取得するためのユーティリティです。
- 主に他のアプリケーションからライブラリとしてコールされますが、単体でも実行可能です。
- Redis接続の動作確認やデータ取得のテスト用途にも利用できます。

#### 使い方

```bash
python redis_receiver.py [--host HOST] [--port PORT] [--key KEY] [--window SEC]
```

#### 引数

- `--host`（デフォルト: `localhost`）: Redis サーバーのホスト名またはIPアドレス
- `--port`（デフォルト: `6379`）: Redis サーバーのポート番号
- `--key`（デフォルト: `meridis`）: 取得する Redis のハッシュキー名
- `--window`（デフォルト: `5.0`）: 時系列データの表示時間幅（秒）。内部バッファサイズに影響します

#### 動作

- 起動時に Redis サーバーへの接続テストを実行し、指定キーの存在確認を行います。接続失敗やキー不存在の場合はエラーメッセージを表示して終了します。
- 実装では TCP レベルの接続チェック（`socket.create_connection`）を先行し、その後 Redis の `PING` コマンドで確認します。接続タイムアウトは 0.5 秒で、到達不能なサーバーに対して迅速に失敗します。
- 指定ハッシュキーからデータを読み取り、フィールド名が連番文字列（`"0"`〜`"N-1"`）として格納されている前提で、値を float 型に変換して処理します。
- デフォルトでは 10 回のデータ取得ループを実行し、各ループ間で 0.5 秒待機します。データ要素数と内容を標準出力に表示します。
- Redis接続エラーやデータ変換エラーは適切にハンドリングされ、エラーメッセージとして出力されます。

#### 注記

- Redis ハッシュのフィールドは文字列の連番（`"0"`, `"1"`, ..., `"89"`）で格納される必要があります。受信側は数値順序でソートして値を読み取ります。
- `connect_timeout` および `socket_timeout` パラメータは `RedisReceiver` クラスのコンストラクタで設定可能（デフォルト: 0.5秒）ですが、現在 CLI オプションとしては公開されていません。
- 時系列データ管理機能により、指定された時間ウィンドウ内のデータを効率的にバッファリングします。
- ライブラリとして使用する際は、`RedisReceiver` クラスの `get_data()` メソッドで最新データを取得できます。

#### 例

```bash
# ローカルサーバーでデフォルト設定での動作確認
python redis_receiver.py

Redis server localhost:6379 - starting data retrieval for key 'meridis_calc_pub'
Retrieved data: 90 elements
Data: [0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9, 1.0, 1.1, 1.2, 1.3, 1.4, 1.5, 1.6, 1.7, 1.8, 1.9, 2.0, 2.1, 2.2, 2.3, 2.4, 2.5, 2.6, 2.7, 2.8, 2.9, 3.0, 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 3.7, 3.8, 3.9, 4.0, 4.1, 4.2, 4.3, 4.4, 4.5, 4.6, 4.7, 4.8, 4.9, 5.0, 5.1, 5.2, 5.3, 5.4, 5.5, 5.6, 5.7, 5.8, 5.9, 6.0, 6.1, 6.2, 6.3, 6.4, 6.5, 6.6, 6.7, 6.8, 6.9, 7.0, 7.1, 7.2, 7.3, 7.4, 7.5, 7.6, 7.7, 7.8, 7.9, 8.0, 8.1, 8.2, 8.3, 8.4, 8.5, 8.6, 8.7, 8.8, 8.9, 9.0]
...
```

実装の詳細や利用可能なクラス・メソッドについては [redis_receiver.py](redis_receiver.py) を参照してください（`RedisReceiver`、`check_connection`、`get_data`、`get_time_data` など）。


### redis_plotter.py

- `redis_plotter.py` は Redisサーバーに格納されたMeridian形式のハッシュデータをリアルタイムでグラフ表示するためのビジュアライゼーションツールです。
- ロボットの関節角度や足部位置の時系列変化をリアルタイムで監視・デバッグできます。
- matplotlib を使用したアニメーション表示により、データの変化を直感的に把握できます。

#### 使い方

```bash
python redis_plotter.py [--width WIDTH] [--height HEIGHT] [--window WINDOW] [--log on/off] [--redis-key KEY] [--display MODE]
```

#### 引数

- `--width`（デフォルト: `8`）: グラフウィンドウの幅（インチ）
- `--height`（デフォルト: `9`）: グラフウィンドウの高さ（インチ）
- `--window`（デフォルト: `5.0`）: 表示する時間幅（秒）
- `--log`（デフォルト: `off`）: ログ出力の有効/無効（`on`/`off`）
- `--redis-key`（デフォルト: `meridis`）: 読み取るRedisキー名
- `--display`（デフォルト: `joint`）: 表示モード（`joint`: 関節角度表示、`foot`: 足部位置表示）

#### 動作

- 起動時に Redis サーバーへの接続確認を行い、指定したキーの存在確認を実施します。接続できない場合はエラーメッセージを表示して終了します。
- 3つのサブプロット構成で、Base Link（IMU関連）、Right Leg、Left Leg の関節データを同時表示します。
- `joint` モード（デフォルト）では各関節の角度変化をリアルタイムプロットし、`foot` モードでは足部の位置座標（x,y,z）を表示します。
- アニメーション間隔は 10ms で、指定された時間ウィンドウ内のデータを連続表示します。
- ログモード（`--log on`）では、フレームごとにハッシュデータ（インデックス 0-89）をカンマ区切り形式でコンソール出力します。
- 一時停止/再開ボタンによりアニメーションの制御が可能です。

#### 表示内容

**Jointモード（デフォルト）:**
- Base Link: `imu_temp`, `imu_roll`, `imu_pitch`, `imu_yaw`
- Right/Left Leg: `hip_yaw`, `hip_roll`, `thigh_pitch`, `knee_pitch`, `ankle_pitch`, `ankle_roll`

**Footモード:**
- Base Link: 同上
- Foot Position: 左右の足部位置座標（`foot_x`, `foot_y`, `foot_z`）

注記

- ネットワーク設定は `network.json` から自動読み込みされ、Redis接続先が決定されます。設定ファイルが存在しない場合はプログラムを終了します。
- 各関節は Meridim90 配列の特定インデックスにマッピングされており、コード内の `joint_to_meridis` 辞書で管理されています。
- 表示範囲は関節角度が ±90度、IMUデータが ±10度 で固定設定されています。

例

```bash
python redis_plotter.py --width 12 --height 8 --window 10 --log on --display foot
```

実装の詳細や利用可能なクラス・関数については `redis_plotter.py` を参照してください（`RedisPlotter`、`get_joint_data_series`、`update_plot` など）。


### udp_sender.py

- `udp_sender.py` は Meridian_console.py のUDP受信機能をテストするためのシンプルなUDP送信/受信プログラムです。
- 10ms間隔でダミーのMeridim90データをUDP送信し、応答を受信して相互通信をシミュレートします。
- 主にローカル環境でのテストやデバッグ用途に利用できます。

### 使い方

```bash
python udp_sender.py
```
### 引数
なし

### 動作

- UDPテストツールで、Meridian_console.py に対して10ms間隔でUDPデータをPushします。
- **Meridian_console.pyにおいてRedis機能のPullを促すための「定期的なトリガ発生器」として利用できます。**
  1. Meridian_console.py を起動するとき、SEND_IP,RECV_IPともに`127.0.0.1`を指定します
  1. Meridian_console.py の `-> Redis` をチェックします
  1. Meridian_console.py の `<- Redis` をチェックします
  1. Meridian_console.py とは別スレッドで、udp_sender.py を起動します
  1. Meridian_console.py でモータを動かしたいとき、`Power`,`Python`,`Enable`をチェックしたのち、スライダー操作してください。
  1. Meridian_console.py でダンスデモを動かしたいときは、`Power`,`Python`,`Enable`,`Demo`をチェックしてください。

- 起動時にダミーのMeridim90データ（90要素のint16配列）を生成し、チェックサムを計算します。
- 10ms間隔でUDPを送信するとき、フレームカウンタをインクリメントします。
- 別スレッドでUDP受信を待機し、受信データを表示します。
- Ctrl+C で停止します。

### 注記

- データ形式はMeridim90に準拠（マスターコマンド、フレームカウンタ、チェックサム付き）。
- ローカルテスト以外ではIPアドレスを変更してください。
```python
UDP_SEND_IP = "127.0.0.1"  # Receiver IP (UDP_RECV_IP_DEF)
UDP_SEND_PORT = 22222      # Receiver port (UDP_RECV_PORT)
UDP_RECV_IP = "127.0.0.1"  # Sender IP (UDP_SEND_IP_DEF)
UDP_RECV_PORT = 22224      # Sender port (UDP_SEND_PORT)
```


### 例

```bash
python udp_sender.py
# 出力例:
# Starting UDP Sender to 192.168.3.3:22222
# Starting UDP Receiver on 127.0.0.1:22224
# Sent data: [1 0 0 ...] (checksum: 12345)
# Received from ('127.0.0.1', 22224): [1 1 0 ...] (frame: 1)
```
