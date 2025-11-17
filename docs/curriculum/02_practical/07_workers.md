# 7. ワーカーの理解

## 学習目標

- ワーカーの役割と仕組みを理解する
- ワーカーの起動オプションを使いこなす
- 並行処理モデル（prefork、gevent、threads）を理解する
- ワーカープールを効果的に使用する

## ワーカーとは

ワーカーは、キューからタスクを取得して実行するプロセスです。

```
[ブローカー] → [ワーカー] → [タスク実行] → [結果保存]
```

### ワーカーの基本構造

```
Celery Worker
├── Main Process (親プロセス)
│   ├── メッセージ受信
│   ├── タスクのディスパッチ
│   └── 子プロセスの管理
└── Worker Pool (子プロセス群)
    ├── Worker Process 1
    ├── Worker Process 2
    ├── Worker Process 3
    └── ...
```

## ワーカーの起動

### 基本的な起動

```bash
celery -A celery_app worker --loglevel=info
```

### よく使うオプション

```bash
# ログレベル指定
celery -A celery_app worker --loglevel=debug

# 並行処理数指定
celery -A celery_app worker --concurrency=4

# ワーカー名指定
celery -A celery_app worker --hostname=worker1@%h

# 特定のキューのみ処理
celery -A celery_app worker --queues=high_priority,default

# プール実装の指定
celery -A celery_app worker --pool=gevent

# 複数オプションの組み合わせ
celery -A celery_app worker \
    --loglevel=info \
    --concurrency=8 \
    --hostname=worker1@%h \
    --queues=default \
    --max-tasks-per-child=1000
```

## 並行処理モデル

### 1. prefork (デフォルト)

マルチプロセスモデル。各タスクは独立したプロセスで実行されます。

```bash
celery -A celery_app worker --pool=prefork --concurrency=4
```

**特徴:**
- CPU密集型タスクに最適
- プロセス間で状態を共有しない
- メモリ使用量が多い
- Pythonの GIL の影響を受けない

**使用例:**
```python
@app.task
def cpu_intensive_task(n):
    """CPUを多用する計算"""
    result = 0
    for i in range(n):
        result += i ** 2
    return result
```

### 2. gevent

グリーンスレッド（協調的マルチタスク）を使用。

```bash
# geventをインストール
pip install gevent

# geventプールで起動
celery -A celery_app worker --pool=gevent --concurrency=100
```

**特徴:**
- I/O密集型タスクに最適
- 多数の同時接続を処理可能
- メモリ効率が良い
- CPU密集型タスクには不向き

**使用例:**
```python
@app.task
def fetch_url(url):
    """ネットワークI/O"""
    import requests
    response = requests.get(url)
    return response.text

# 100のURLを並列で取得
from celery import group
job = group(fetch_url.s(url) for url in urls)
result = job.apply_async()
```

### 3. eventlet

geventと似た協調的マルチタスク。

```bash
# eventletをインストール
pip install eventlet

# eventletプールで起動
celery -A celery_app worker --pool=eventlet --concurrency=100
```

### 4. threads

スレッドベースの並行処理。

```bash
celery -A celery_app worker --pool=threads --concurrency=10
```

**特徴:**
- シンプル
- メモリ共有が可能
- GILの影響を受ける（CPUバウンドには不向き）

### プールの選択ガイド

| タスクの種類 | 推奨プール | 並行数 |
|------------|-----------|--------|
| CPU密集型（計算、画像処理） | prefork | CPUコア数 |
| I/O密集型（API呼び出し、DB） | gevent/eventlet | 100-1000 |
| 混合型 | prefork | CPUコア数×2 |
| シンプルなタスク | threads | 10-20 |

## ワーカーのオプション詳細

### concurrency - 並行処理数

```bash
# 4つのワーカープロセス
celery -A celery_app worker --concurrency=4

# 自動（CPUコア数に基づく）
celery -A celery_app worker --concurrency=auto
```

### max-tasks-per-child - メモリリーク対策

```bash
# 1000タスク実行後にワーカープロセスを再起動
celery -A celery_app worker --max-tasks-per-child=1000
```

これにより、メモリリークが発生しても自動的にクリーンアップされます。

### time-limit - タスクのタイムリミット

```bash
# ハードリミット: 300秒
celery -A celery_app worker --time-limit=300

# ソフトリミット: 240秒（SIGTERMを送信）
celery -A celery_app worker --soft-time-limit=240
```

### loglevel - ログレベル

```bash
# DEBUG: 最も詳細
celery -A celery_app worker --loglevel=debug

# INFO: 標準的な情報（デフォルト）
celery -A celery_app worker --loglevel=info

# WARNING: 警告以上
celery -A celery_app worker --loglevel=warning

# ERROR: エラーのみ
celery -A celery_app worker --loglevel=error
```

### logfile - ログファイル

```bash
# ファイルにログを出力
celery -A celery_app worker --logfile=/var/log/celery/worker.log

# 標準出力（デフォルト）
celery -A celery_app worker --logfile=-
```

### pidfile - PIDファイル

```bash
celery -A celery_app worker --pidfile=/var/run/celery/worker.pid
```

### detach - バックグラウンド実行

```bash
celery -A celery_app worker --detach \
    --logfile=/var/log/celery/worker.log \
    --pidfile=/var/run/celery/worker.pid
```

## ワーカーの制御

### inspect - ワーカー情報の取得

```python
from celery_app import app

# アクティブなワーカーを取得
workers = app.control.inspect().active()
print(workers)

# 登録されているタスクを取得
tasks = app.control.inspect().registered()
print(tasks)

# 予約されているタスクを取得
reserved = app.control.inspect().reserved()
print(reserved)

# ワーカーの統計情報
stats = app.control.inspect().stats()
print(stats)
```

コマンドラインから:

```bash
# アクティブなタスク
celery -A celery_app inspect active

# 登録されているタスク
celery -A celery_app inspect registered

# ワーカーの統計
celery -A celery_app inspect stats
```

### control - ワーカーの制御

```python
# ワーカーにタスクキューを追加
app.control.add_consumer('new_queue')

# タスクキューを削除
app.control.cancel_consumer('old_queue')

# ワーカーをシャットダウン
app.control.shutdown()

# ワーカープールを縮小/拡大
app.control.pool_grow(n=2)   # 2つ追加
app.control.pool_shrink(n=1) # 1つ削減
```

コマンドラインから:

```bash
# ワーカーをシャットダウン
celery -A celery_app control shutdown

# プールサイズを変更
celery -A celery_app control pool_grow 2
celery -A celery_app control pool_shrink 1
```

## 複数ワーカーの運用

### 複数ワーカーを起動

```bash
# ワーカー1: 高優先度キュー
celery -A celery_app worker \
    --hostname=worker-high@%h \
    --queues=high_priority \
    --concurrency=4

# ワーカー2: 通常キュー
celery -A celery_app worker \
    --hostname=worker-default@%h \
    --queues=default \
    --concurrency=8

# ワーカー3: 低優先度キュー（I/O密集型）
celery -A celery_app worker \
    --hostname=worker-low@%h \
    --queues=low_priority \
    --pool=gevent \
    --concurrency=100
```

### 役割別ワーカー

```bash
# CPU密集型タスク用
celery -A celery_app worker \
    --hostname=cpu-worker@%h \
    --queues=cpu_tasks \
    --pool=prefork \
    --concurrency=4

# API呼び出し用
celery -A celery_app worker \
    --hostname=io-worker@%h \
    --queues=api_tasks \
    --pool=gevent \
    --concurrency=200

# メール送信用
celery -A celery_app worker \
    --hostname=email-worker@%h \
    --queues=email \
    --concurrency=10 \
    --rate-limit=tasks:10/m  # レート制限
```

## ワーカーのモニタリング

### イベントの有効化

```bash
celery -A celery_app worker --loglevel=info --events
```

### Flowerでのモニタリング

```bash
# Flowerのインストール
pip install flower

# 起動
celery -A celery_app flower

# ブラウザで http://localhost:5555 にアクセス
```

Flowerで確認できる情報:
- アクティブなワーカー
- 実行中のタスク
- タスクの統計
- タスクの履歴
- ワーカーの負荷

### カスタムモニタリング

```python
# monitoring.py
from celery_app import app
import time

def monitor_workers():
    """ワーカーの状態を定期的に確認"""
    inspect = app.control.inspect()

    while True:
        # アクティブなワーカー
        active = inspect.active()
        if active:
            for worker, tasks in active.items():
                print(f'{worker}: {len(tasks)} active tasks')
        else:
            print('No active workers!')

        # 統計情報
        stats = inspect.stats()
        if stats:
            for worker, stat in stats.items():
                pool = stat.get('pool', {})
                print(f'{worker}: {pool}')

        time.sleep(10)  # 10秒ごとに確認

if __name__ == '__main__':
    monitor_workers()
```

## トラブルシューティング

### 問題1: ワーカーがタスクを処理しない

**確認ポイント:**
```bash
# ワーカーが起動しているか
celery -A celery_app inspect active

# ワーカーがタスクを認識しているか
celery -A celery_app inspect registered

# キュー名が正しいか確認
celery -A celery_app inspect active_queues
```

### 問題2: メモリリーク

**対策:**
```bash
# max-tasks-per-childを設定
celery -A celery_app worker --max-tasks-per-child=100
```

### 問題3: タスクがタイムアウトする

**対策:**
```bash
# タイムリミットを延長
celery -A celery_app worker --time-limit=600
```

または個別のタスクで設定:
```python
@app.task(time_limit=600)
def long_task():
    pass
```

## ベストプラクティス

### 1. 適切な並行処理数

```bash
# CPU密集型
concurrency = CPU_CORES

# I/O密集型
concurrency = 100-1000 (gevent/eventlet)

# 混合型
concurrency = CPU_CORES * 2
```

### 2. メモリ管理

```bash
celery -A celery_app worker \
    --max-tasks-per-child=1000 \
    --max-memory-per-child=200000  # 200MB
```

### 3. ログの整理

```bash
celery -A celery_app worker \
    --loglevel=info \
    --logfile=/var/log/celery/%n%I.log
    # %n: ワーカー名
    # %I: プロセスインデックス
```

### 4. グレースフルシャットダウン

```bash
# SIGTERMを送信（推奨）
kill -TERM <PID>

# またはCeleryコマンドで
celery -A celery_app control shutdown
```

## まとめ

- ✅ ワーカーはタスクを実行するプロセス
- ✅ prefork（CPU）、gevent（I/O）、threads（シンプル）のプール
- ✅ `--concurrency`で並行処理数を制御
- ✅ `--max-tasks-per-child`でメモリリーク対策
- ✅ `inspect`と`control`でワーカーを管理・監視

## 次のステップ

次のセクション「[メッセージブローカー](./08_message_brokers.md)」では、ブローカーの詳細を学びます。
