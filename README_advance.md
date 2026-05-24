# meridis - 技術仕様

[English](README_advance_EN.md) / Japanese

**meridis**の詳細な技術仕様、設定ファイルの構造、カスタマイズ方法を解説します。  
システムを拡張したい、独自の設定で動かしたい方はこちらをご覧ください。  
<BR>

## 目次

- [仕様](#仕様)
- [ロボット動作を管理する](#ロボット動作を管理する)
  - [アーキテクチャ図](#アーキテクチャ図)
  - [コマンド](#コマンド)
  - [オプション](#オプション)
  - [動作](#動作)
  - [マネージャーオプション（--mgr）](#マネージャーオプション--mgr)
    - [mgr_sim2real.json](#mgr_sim2realjson)
    - [mgr_real2sim.json](#mgr_real2simjson)
    - [mgr_ai2real.json](#mgr_ai2realjson)
  - [ネットワークオプション（--network）](#ネットワークオプション--network)
  - [フットオプション(--foot)](#フットオプション--foot)
- [ライブラリの詳細](#ライブラリの詳細)
  - [redisデータの送信](#redisデータの送信)
  - [redisデータの受信](#redisデータの受信)
  - [redisデータのプロット](#redisデータのプロット)

 
---

## 仕様

## ロボット動作を管理する

**meridis_manager.py**

- `meridis_manager.py` はシミュレーション環境と実機ロボットの間でリアルタイムデータ交換を実現する中核的なブリッジ機能を提供します。
- Mujocoをベースとするシミュレーションプログラム**merimujoco**を公開しています。`meridis_manager.py`と組み合わせる操作手順を「クイックスタート」で説明しています<br>
  https://github.com/holypong/merimujoco


![merimujoco](image/merimujoco.png)

### アーキテクチャ図

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
    ROB[ロボット実機] --> MM;
    MM --> SIM;
    MM --> ROB;
    MM --> MCP;
```

### コマンド

```bash
python meridis_manager.py --mgr MGR_FILE --network NETWORK_FILE --foot FOOT_MODE
```

### オプション

- `--mgr`（デフォルト: `mgr_sim2real.json`）: マネージャー設定 JSON ファイルのパス
- `--network`（デフォルト: `network.json`）: ネットワーク設定 JSON ファイルのパス  
- `--foot`（デフォルト: `off`）: 足部データの詳細スケーリング設定（`off`/`on`）

### 動作

- 起動時にマネージャー設定ファイルとネットワーク設定ファイルを読み込みます。設定ファイルが見つからない場合はエラーメッセージを表示してプログラムを終了します。
- マネージャー設定の `data_flow` に基づいて、Redis→UDP、UDP→Redis の各方向でのデータ転送を制御します。
- Redis のハッシュデータ（90要素）を読み取り、Meridim90 バイナリフォーマットに変換して UDP パケットとして送信します。
- UDP で受信した Meridim90 データをパースし、Redis のハッシュとして書き戻します。
- `--foot on` オプション時は以下のスケーリングを適用します：
  - 関節角度データ（インデックス21,23,25,...45 および 51,53,55,...75）: 1/100 スケーリング
  - 左足位置データ（インデックス46-49）: 1/100000 スケーリング（mm×100→m変換）
  - 右足位置データ（インデックス76-79）: 1/100000 スケーリング（mm×100→m変換）
- `--foot off` オプション時は、インデックス21-80の偶数インデックスすべてに 1/100 スケーリングを適用します。
- 転送処理は継続的なループで実行され、リアルタイムでのデータ同期を実現します。


### マネージャーオプション（--mgr）

#### キーの役割
`--mgr`オプションでは以下のキーを扱います。すべてHASH型です。

| キー | Write | Read | 役割 |
|------------|----------------|----------------|---------|
| `meridis_sim_pub` | シミュレータ | 実機 / MCP | シミュレータ主導での制御 |
| `meridis_calc_pub` | 動作生成プログラム | シミュレータ / 実機 | 数値制御 |
| `meridis_console_pub` | Meridian_console | シミュレータ / 実機 | コンソールUIからの操作 |
| `meridis_mgr_pub` | meridis_manager | シミュレータ / 実機 / MCP | 実機主導の制御 |
| `meridis_ai_pub` | AIエージェント+MCP | シミュレータ / 実機 | AI主導の制御 |


#### mgr_sim2real.json

Meridianを搭載したロボット実機がある場合、シミュレーションロボットのダンスの動きをロボット実機に同期させることができます。

以下は `mgr_sim2real.json` の例<br>
（実際のファイルはリポジトリ内のものを参照してください）：
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

    Note right of HOST: キー: meridis_ai_pub
    HOST->>R: HSET meridis_ai_pub // ホストが指令を格納
    Note right of R: キー: meridis_ai_pub
    SIM->>R: HGETALL meridis_ai_pub  // シミュレータが指令を取得
    Note right of R: キー: meridis_sim_pub
    SIM->>R: HSET meridis_sim_pub // シミュレータが演算結果を格納
    ROB->>MMD: UDP受信  // 実機から実行結果を受信
    Note right of R: キー: meridis_sim_pub
    MMD->>R: HGETALL meridis_sim_pub  // meridis-manager が演算結果を取得
    MMD->>ROB: UDP送信  // 実機へ演算結果を送信
```

#### mgr_real2sim.json

Meridianを搭載したロボット実機がある場合、ロボット実機の関節の動きをシミュレーション上のロボットで再現できます。

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

#### mgr_ai2real.json

Meridianを搭載したロボット実機がある場合、AIエージェントがMCPサーバー経由でロボット実機の動きを指令します。

以下は `mgr_ai2real.json` の例です（実際のファイルはリポジトリ内のものを参照してください）：

```json
{
  "redis": {
    "host": "127.0.0.1",
    "port": 6379
  },
  "redis_keys": {
    "read": "meridis_ai_pub",
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

    Note right of HOST: キー: meridis_ai_pub
    HOST->>R: HSET meridis_ai_pub // ホストが指令を格納
    ROB->>MMD: UDP受信  // 実機から実行結果を受信
    Note right of R: キー: meridis_ai_pub
    MMD->>R: HGETALL meridis_ai_pub  // meridis-manager が指令を取得
    Note right of R: キー: meridis_mgr_pub
    MMD->>R: HSET meridis_mgr_pub  // meridis-manager が実行結果を格納
    MMD->>ROB: UDP送信  // 実機へ指令を送信
    Note right of HOST: キー: meridis_mgr_pub
    HOST->>R: HGETALL meridis_mgr_pub // ホストが実行結果を取得
    HOST->>MON: ストレージ格納 // ホストが実行結果・設定をメモリ・ファイルに格納
    HOST->>MON: ストレージ取得 // ホストが実行結果・設定をメモリ・ファイルから取得

```

### ネットワークオプション（--network）
- udp
  - PC側からみた送信側・受信側のIP・PORTを設定してください。

- `network.json`の例<br>
  - send: ロボット実機のIPアドレスとポート<br>
  - recv: PCのIPアドレスとポート
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

### フットオプション(--foot)
足の逆運動学計算に基づく位置データのスケーリングを制御するオプションです。足のXYZ位置を登録する処理が入っている場合に使用します。

- `off` (デフォルト): 全サーボ位置データ（インデックス 21-80 の偶数番）を 1/100 スケーリングする（従来動作）

- `on`: 関節角度データと足位置データを個別にスケーリングする：
  - 関節角度データ（インデックス 21,23,25,...45 および 51,53,55,...75）: 1/100 スケーリング
  - 左足位置データ（インデックス 46-49）: 1/100000 スケーリング（mm×100→m変換）
  - 右足位置データ（インデックス 76-79）: 1/100000 スケーリング（mm×100→m変換）

具体的な実装は `meridis_manager.py` の `write_redis_data()` 関数内にあります：

```python
# --foot on の場合：関節角度は1/100、足位置は1/100000（mm×100→m）
data[21:47:2] /= 100.0        # 関節角度（2ステップ刻み）
data[46:50] /= 100000.0       # 左足位置 x,y,z,r (mm×100 → m)
data[51:77:2] /= 100.0        # 関節角度（2ステップ刻み）
data[76:80] /= 100000.0       # 右足位置 x,y,z,r (mm×100 → m)

# --foot off の場合：従来通りの処理
data[21:81:2] /= 100.0        # 全サーボ位置データ（2ステップ刻み）
```


---
## ライブラリの詳細

- 以降のライブラリの詳細説明は、プログラム開発時の参考としてください。

### redisデータの送信

**redis_transfer.py**

- `redis_transfer.py` は Redisサーバーの指定キーにMeridian形式のハッシュデータを書き込むためのデータ転送ユーティリティです。
- 主に他のアプリケーションからライブラリとしてコールされますが、テスト用途で単体実行も可能です。
- Redis接続の動作確認やデータ書き込みのテスト機能を提供します。

#### コマンド

```bash
python redis_transfer.py [--host HOST] [--port PORT] [--key KEY]
```

#### オプション

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

#### 例

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


### redisデータの受信

**redis_receiver.py**

- `redis_receiver.py` は Redisサーバーの指定キーに保存されたMeridian形式のハッシュデータを取得するためのユーティリティです。
- 主に他のアプリケーションからライブラリとしてコールされますが、単体でも実行可能です。
- Redis接続の動作確認やデータ取得のテスト用途にも利用できます。

#### コマンド

```bash
python redis_receiver.py [--host HOST] [--port PORT] [--key KEY] [--window SEC]
```

#### オプション

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


### redisデータのプロット

**redis_plotter.py**

- `redis_plotter.py` は Redisサーバーに格納されたMeridian形式のハッシュデータをリアルタイムでグラフ表示するためのビジュアライゼーションツールです。
- ロボットの関節角度や足部位置の時系列変化をリアルタイムで監視・デバッグできます。
- matplotlib を使用したアニメーション表示により、データの変化を直感的に把握できます。

#### コマンド

```bash
python redis_plotter.py [--width WIDTH] [--height HEIGHT] [--window WINDOW] [--log on/off] [--redis-key KEY] [--display MODE]
```

#### オプション

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

#### 注記

- ネットワーク設定は `network.json` から自動読み込みされ、Redis接続先が決定されます。設定ファイルが存在しない場合はプログラムを終了します。
- 各関節は Meridim90 配列の特定インデックスにマッピングされており、コード内の `joint_to_meridis` 辞書で管理されています。
- 表示範囲は関節角度が ±90度、IMUデータが ±10度 で固定設定されています。

#### 例

```bash
python redis_plotter.py --width 12 --height 8 --window 10 --log on --display foot
```

実装の詳細や利用可能なクラス・関数については `redis_plotter.py` を参照してください（`RedisPlotter`、`get_joint_data_series`、`update_plot` など）。



## データ収集ツール：redis_logger.py

PADコントローラのボタンをトリガとして、Redisからリアルタイムにデータを収集し `log/logs-YYYYMMDDHHMM.csv` に保存するスタンドアロンツールです。

### 使い方

```bash
python redis_logger.py --btn 1                              # ボタン値=1 の間だけ録画
python redis_logger.py --btn 512 --redis redis-mgr.json    # 実機用Redis設定で録画
python redis_logger.py --btn 3 --interval 20               # ポーリング間隔 20 ms
python redis_logger.py --btn 1 --redis-key meridis_sim_pub # Redisキーを直接指定
```

### オプション

| オプション | デフォルト | 説明 |
|---|---|---|
| `--btn` | （必須） | 録画トリガとなる PAD ボタン値（Meridim90[15] の整数値） |
| `--redis` | `redis.json` | Redis接続設定JSONファイル |
| `--redis-key` | JSON の `redis_keys.read` | 読み取るRedisキー名（省略時はJSONから取得） |
| `--interval` | `10.0` ms | ポーリング間隔 |

### 動作仕様

- ボタン値が `--btn` と一致している間だけバッファにデータを蓄積
- ボタン値が変化するか上限（10000行）に達したら `log/` に自動保存
- Ctrl+C で中断した場合も残バッファを保存
- 保存形式は `buf_input.csv` と同じ Meridim90 生データ（ヘッダーなし・90列）

---

