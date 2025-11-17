# 2. 環境構築

## 学習目標

このセクションを完了すると、以下ができるようになります:
- Python開発環境を準備する
- Celeryをインストールする
- メッセージブローカー（RedisまたはRabbitMQ）をセットアップする
- 開発環境が正しく動作することを確認する

## 前提条件

- Python 3.7以上がインストールされていること
- 基本的なコマンドライン操作ができること
- pipまたはpip3が使えること

## ステップ1: Python環境の確認

### Pythonバージョンの確認

```bash
python --version
# または
python3 --version
```

**推奨バージョン**: Python 3.8以上

### 仮想環境の作成

プロジェクトごとに独立した環境を作成するのがベストプラクティスです。

#### venvを使用（Python標準）

```bash
# プロジェクトディレクトリの作成
mkdir celery-tutorial
cd celery-tutorial

# 仮想環境の作成
python3 -m venv venv

# 仮想環境の有効化（Linux/Mac）
source venv/bin/activate

# 仮想環境の有効化（Windows）
venv\Scripts\activate

# 有効化されると、プロンプトに(venv)と表示される
```

#### virtualenvを使用

```bash
# virtualenvのインストール（必要な場合）
pip install virtualenv

# 仮想環境の作成
virtualenv venv

# 有効化（上記と同じ）
source venv/bin/activate
```

#### pipのアップグレード

```bash
pip install --upgrade pip
```

## ステップ2: Celeryのインストール

### 基本インストール

```bash
# Celeryの最新版をインストール
pip install celery
```

### バンドル付きインストール（推奨）

Celeryは様々な依存関係を「バンドル」として提供しています。

#### Redisを使用する場合

```bash
pip install "celery[redis]"
```

#### RabbitMQを使用する場合

```bash
pip install "celery[amqp]"
```

#### その他の便利なバンドル

```bash
# 複数のバンドルを同時にインストール
pip install "celery[redis,msgpack]"

# msgpack: より高速なシリアライゼーション
# auth: 認証機能の追加
# yaml: YAML設定ファイルのサポート
```

### インストールの確認

```bash
# Celeryのバージョンを確認
celery --version

# 出力例:
# 5.3.4 (emerald-rush)
```

### requirements.txtの作成

プロジェクトの依存関係を管理するため、requirements.txtを作成します。

```bash
# 現在の依存関係を保存
pip freeze > requirements.txt
```

**requirements.txt の例:**

```txt
celery==5.3.4
redis==5.0.1
kombu==5.3.4
```

## ステップ3: メッセージブローカーのセットアップ

メッセージブローカーは必須コンポーネントです。RedisまたはRabbitMQのどちらかを選択します。

### オプションA: Redis（推奨：初心者向け）

Redisはシンプルで高速、セットアップが簡単です。

#### Redisのインストール

**Linux (Ubuntu/Debian):**

```bash
sudo apt update
sudo apt install redis-server

# Redisの起動
sudo systemctl start redis-server

# 自動起動の設定
sudo systemctl enable redis-server

# 状態確認
sudo systemctl status redis-server
```

**macOS (Homebrew):**

```bash
brew install redis

# Redisの起動
brew services start redis

# または一時的に起動
redis-server
```

**Windows:**

Windows向けには公式版がありませんが、以下の方法があります:

1. **WSL（Windows Subsystem for Linux）を使用**（推奨）
2. **Docker を使用**（後述）
3. **Microsoft版を使用**（非推奨：古い）

#### Dockerを使用したRedis

```bash
# Redisコンテナの起動
docker run -d -p 6379:6379 --name redis redis:latest

# ログの確認
docker logs redis

# 停止
docker stop redis

# 再起動
docker start redis
```

#### Redisの動作確認

```bash
# redis-cliで接続
redis-cli

# Redis CLIで以下を実行
127.0.0.1:6379> ping
PONG
127.0.0.1:6379> set test "Hello"
OK
127.0.0.1:6379> get test
"Hello"
127.0.0.1:6379> exit
```

#### Python からのRedis接続確認

```python
# test_redis.py
import redis

r = redis.Redis(host='localhost', port=6379, db=0)
r.set('test', 'Hello from Python')
print(r.get('test'))  # b'Hello from Python'
```

```bash
python test_redis.py
```

### オプションB: RabbitMQ（推奨：本番環境）

RabbitMQはより高機能で、本番環境でよく使われます。

#### RabbitMQのインストール

**Linux (Ubuntu/Debian):**

```bash
sudo apt update
sudo apt install rabbitmq-server

# RabbitMQの起動
sudo systemctl start rabbitmq-server

# 自動起動の設定
sudo systemctl enable rabbitmq-server

# 状態確認
sudo systemctl status rabbitmq-server
```

**macOS (Homebrew):**

```bash
brew install rabbitmq

# RabbitMQの起動
brew services start rabbitmq

# または一時的に起動
/usr/local/sbin/rabbitmq-server
```

#### Dockerを使用したRabbitMQ

```bash
# RabbitMQコンテナの起動（管理画面付き）
docker run -d \
  --name rabbitmq \
  -p 5672:5672 \
  -p 15672:15672 \
  rabbitmq:3-management

# ログの確認
docker logs rabbitmq
```

#### RabbitMQの管理画面

ブラウザで `http://localhost:15672` にアクセス

- **ユーザー名**: guest
- **パスワード**: guest

#### ユーザーとVHostの作成（本番環境向け）

```bash
# ユーザーの作成
sudo rabbitmqctl add_user myuser mypassword

# 権限の設定
sudo rabbitmqctl set_permissions -p / myuser ".*" ".*" ".*"

# 管理者権限の付与（必要な場合）
sudo rabbitmqctl set_user_tags myuser administrator
```

## ステップ4: プロジェクト構造の作成

基本的なプロジェクト構造を作成します。

```bash
celery-tutorial/
├── venv/              # 仮想環境
├── celery_app.py      # Celeryアプリケーション
├── tasks.py           # タスク定義
├── config.py          # 設定ファイル
└── requirements.txt   # 依存関係
```

### ディレクトリの作成

```bash
cd celery-tutorial
touch celery_app.py tasks.py config.py
```

## ステップ5: 最小限の設定ファイル作成

### config.py

```python
# config.py
import os

class Config:
    # メッセージブローカーの設定
    # Redis使用時
    broker_url = os.getenv('CELERY_BROKER_URL', 'redis://localhost:6379/0')

    # RabbitMQ使用時（コメントアウトを外す）
    # broker_url = os.getenv('CELERY_BROKER_URL', 'amqp://guest:guest@localhost:5672//')

    # 結果バックエンドの設定
    result_backend = os.getenv('CELERY_RESULT_BACKEND', 'redis://localhost:6379/0')

    # タイムゾーン
    timezone = 'Asia/Tokyo'

    # タスクの結果の有効期限（秒）
    result_expires = 3600
```

### celery_app.py

```python
# celery_app.py
from celery import Celery
from config import Config

# Celeryアプリケーションの作成
app = Celery('celery_tutorial')

# 設定の読み込み
app.config_from_object(Config)

# タスクの自動検出（オプション）
app.autodiscover_tasks(['tasks'])

if __name__ == '__main__':
    app.start()
```

## ステップ6: 動作確認

### 簡単なテストタスクの作成

```python
# tasks.py
from celery_app import app
import time

@app.task
def hello():
    return 'Hello, Celery!'

@app.task
def add(x, y):
    return x + y

@app.task
def slow_task(seconds):
    time.sleep(seconds)
    return f'Slept for {seconds} seconds'
```

### ワーカーの起動

別のターミナルウィンドウで:

```bash
# 仮想環境の有効化
source venv/bin/activate

# ワーカーの起動
celery -A celery_app worker --loglevel=info
```

**期待される出力:**

```
 -------------- celery@hostname v5.3.4 (emerald-rush)
--- ***** -----
-- ******* ---- Linux-... 2024-XX-XX XX:XX:XX
- *** --- * ---
- ** ---------- [config]
- ** ---------- .> app:         celery_tutorial:0x...
- ** ---------- .> transport:   redis://localhost:6379/0
- ** ---------- .> results:     redis://localhost:6379/0
- *** --- * --- .> concurrency: 8 (prefork)
-- ******* ---- .> task events: OFF
--- ***** -----
 -------------- [queues]
                .> celery           exchange=celery(direct) key=celery

[tasks]
  . tasks.add
  . tasks.hello
  . tasks.slow_task

[2024-XX-XX XX:XX:XX,XXX: INFO/MainProcess] Connected to redis://localhost:6379/0
[2024-XX-XX XX:XX:XX,XXX: INFO/MainProcess] mingle: searching for neighbors
[2024-XX-XX XX:XX:XX,XXX: INFO/MainProcess] mingle: all alone
[2024-XX-XX XX:XX:XX,XXX: INFO/MainProcess] celery@hostname ready.
```

### タスクの実行テスト

元のターミナルで、Pythonインタラクティブシェルを起動:

```bash
python
```

```python
>>> from tasks import hello, add, slow_task

# タスクの非同期実行
>>> result = hello.delay()
>>> result
<AsyncResult: 4e6e6f7a-3b4c-4c5e-9a8f-1b2c3d4e5f6a>

# 結果の取得
>>> result.get(timeout=10)
'Hello, Celery!'

# 引数付きタスク
>>> result = add.delay(4, 6)
>>> result.get()
10

# 時間のかかるタスク
>>> result = slow_task.delay(5)
>>> result.ready()  # 完了したか確認
False
>>> # 5秒待つ...
>>> result.ready()
True
>>> result.get()
'Slept for 5 seconds'
```

ワーカーのターミナルでタスク実行のログが表示されます:

```
[2024-XX-XX XX:XX:XX,XXX: INFO/MainProcess] Task tasks.hello[...] received
[2024-XX-XX XX:XX:XX,XXX: INFO/ForkPoolWorker-1] Task tasks.hello[...] succeeded in 0.001s: 'Hello, Celery!'
```

## ステップ7: 開発ツールのインストール（オプション）

### Flower - Celeryモニタリングツール

```bash
pip install flower
```

起動:

```bash
celery -A celery_app flower
```

ブラウザで `http://localhost:5555` にアクセス

### iPython - より使いやすいREPL

```bash
pip install ipython
```

使用:

```bash
ipython
```

## トラブルシューティング

### 問題1: ワーカーが起動しない

**エラー**: `Cannot connect to redis://localhost:6379/0`

**解決策**:
```bash
# Redisが起動しているか確認
sudo systemctl status redis-server

# または
redis-cli ping
```

### 問題2: タスクが実行されない

**確認ポイント**:
1. ワーカーが起動しているか
2. タスクがワーカーログに表示されているか
3. ブローカーの接続が正しいか

```python
# 接続テスト
from celery_app import app
print(app.connection().connect())
```

### 問題3: 結果が取得できない

**エラー**: `BackendGetMetaError`

**解決策**:
- `result_backend` が正しく設定されているか確認
- バックエンド（Redis等）が起動しているか確認

### 問題4: Windowsでワーカーが動かない

**エラー**: Celery 4.0以降、Windowsのサポートが実験的

**解決策**:
```bash
# geventを使用
pip install gevent
celery -A celery_app worker --pool=gevent --loglevel=info

# またはWSLを使用（推奨）
```

### 問題5: ポートが既に使用されている

**エラー**: `Address already in use`

**解決策**:
```bash
# Redisの場合（ポート6379）
sudo lsof -i :6379
sudo kill -9 <PID>

# RabbitMQの場合（ポート5672）
sudo lsof -i :5672
```

## 環境変数の設定（推奨）

### .envファイルの作成

```bash
# .env
CELERY_BROKER_URL=redis://localhost:6379/0
CELERY_RESULT_BACKEND=redis://localhost:6379/0
```

### python-dotenvのインストール

```bash
pip install python-dotenv
```

### 設定ファイルの更新

```python
# config.py
import os
from dotenv import load_dotenv

# .envファイルを読み込む
load_dotenv()

class Config:
    broker_url = os.getenv('CELERY_BROKER_URL', 'redis://localhost:6379/0')
    result_backend = os.getenv('CELERY_RESULT_BACKEND', 'redis://localhost:6379/0')
    timezone = 'Asia/Tokyo'
    result_expires = 3600
```

## Docker Composeを使った環境構築（推奨：チーム開発）

### docker-compose.ymlの作成

```yaml
# docker-compose.yml
version: '3.8'

services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  celery_worker:
    build: .
    command: celery -A celery_app worker --loglevel=info
    volumes:
      - .:/app
    depends_on:
      - redis
    environment:
      - CELERY_BROKER_URL=redis://redis:6379/0
      - CELERY_RESULT_BACKEND=redis://redis:6379/0

  flower:
    build: .
    command: celery -A celery_app flower
    ports:
      - "5555:5555"
    depends_on:
      - redis
      - celery_worker
    environment:
      - CELERY_BROKER_URL=redis://redis:6379/0
      - CELERY_RESULT_BACKEND=redis://redis:6379/0

volumes:
  redis_data:
```

### Dockerfileの作成

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["celery", "-A", "celery_app", "worker", "--loglevel=info"]
```

### 起動

```bash
docker-compose up -d
```

## まとめ

このセクションで以下を完了しました:

- ✅ Python仮想環境の作成
- ✅ Celeryのインストール
- ✅ メッセージブローカー（Redis/RabbitMQ）のセットアップ
- ✅ 基本的なプロジェクト構造の作成
- ✅ 動作確認とテスト
- ✅ トラブルシューティング方法の理解

## チェックリスト

環境構築が完了したか確認:

- [ ] Python 3.7以上がインストールされている
- [ ] 仮想環境が作成・有効化されている
- [ ] Celeryがインストールされている
- [ ] メッセージブローカーが起動している
- [ ] ワーカーが起動して、タスクを受け付けている
- [ ] テストタスクが正常に実行される
- [ ] 結果が正しく取得できる

## 次のステップ

次のセクション「[最初のタスク](./03_first_task.md)」では、実際にタスクを作成して実行する方法を詳しく学びます。

## 参考リソース

- [Celery Installation](https://docs.celeryproject.org/en/stable/getting-started/introduction.html)
- [Redis Documentation](https://redis.io/documentation)
- [RabbitMQ Documentation](https://www.rabbitmq.com/documentation.html)
- [Docker Documentation](https://docs.docker.com/)
