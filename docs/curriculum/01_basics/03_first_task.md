# 3. 最初のタスク

## 学習目標

このセクションを完了すると、以下ができるようになります:
- Celeryタスクを定義する
- タスクをキューに送信する
- ワーカーを起動してタスクを実行する
- タスクの結果を取得する

## Hello Worldタスク

最もシンプルなCeleryタスクから始めましょう。

### プロジェクト構造

```
celery-hello/
├── celery_app.py   # Celeryアプリケーション
└── tasks.py        # タスク定義
```

### ステップ1: Celeryアプリケーションの作成

```python
# celery_app.py
from celery import Celery

# Celeryインスタンスの作成
app = Celery(
    'hello',                              # アプリ名
    broker='redis://localhost:6379/0',    # メッセージブローカー
    backend='redis://localhost:6379/0'    # 結果バックエンド
)

# 基本設定
app.conf.update(
    task_serializer='json',
    accept_content=['json'],
    result_serializer='json',
    timezone='Asia/Tokyo',
    enable_utc=True,
)
```

### ステップ2: タスクの定義

```python
# tasks.py
from celery_app import app

@app.task
def hello():
    """最もシンプルなタスク"""
    return 'Hello, Celery!'

@app.task
def greet(name):
    """引数を受け取るタスク"""
    return f'Hello, {name}!'
```

### ステップ3: ワーカーの起動

ターミナルで以下を実行:

```bash
celery -A celery_app worker --loglevel=info
```

**出力の見方:**

```
 -------------- celery@hostname v5.3.4 (emerald-rush)
--- ***** -----
-- ******* ---- Linux-...
- *** --- * ---
- ** ---------- [config]
- ** ---------- .> app:         hello:0x...
- ** ---------- .> transport:   redis://localhost:6379/0
- ** ---------- .> results:     redis://localhost:6379/0
- *** --- * --- .> concurrency: 8 (prefork)
-- ******* ---- .> task events: OFF
--- ***** -----
 -------------- [queues]
                .> celery           exchange=celery(direct) key=celery

[tasks]                          ← 登録されたタスク
  . tasks.greet
  . tasks.hello

[2024-XX-XX XX:XX:XX,XXX: INFO/MainProcess] Connected to redis://localhost:6379/0
[2024-XX-XX XX:XX:XX,XXX: INFO/MainProcess] mingle: searching for neighbors
[2024-XX-XX XX:XX:XX,XXX: INFO/MainProcess] mingle: all alone
[2024-XX-XX XX:XX:XX,XXX: INFO/MainProcess] celery@hostname ready.
                                ↑ ワーカーが起動完了
```

### ステップ4: タスクの実行

別のターミナルまたはPythonスクリプトで:

```python
from tasks import hello, greet

# タスクの実行
result = hello.delay()
print(result.get())  # 'Hello, Celery!'

# 引数付きタスクの実行
result = greet.delay('Alice')
print(result.get())  # 'Hello, Alice!'
```

### ステップ5: ワーカーのログを確認

ワーカーのターミナルに以下のようなログが表示されます:

```
[2024-XX-XX XX:XX:XX,XXX: INFO/MainProcess] Task tasks.hello[abc-123-def] received
[2024-XX-XX XX:XX:XX,XXX: INFO/ForkPoolWorker-1] Task tasks.hello[abc-123-def] succeeded in 0.001s: 'Hello, Celery!'

[2024-XX-XX XX:XX:XX,XXX: INFO/MainProcess] Task tasks.greet[xyz-456-uvw] received
[2024-XX-XX XX:XX:XX,XXX: INFO/ForkPoolWorker-2] Task tasks.greet[xyz-456-uvw] succeeded in 0.001s: 'Hello, Alice!'
```

## タスクの実行方法

Celeryには複数のタスク実行方法があります。

### 1. delay() - シンプルな非同期実行

最も一般的な方法:

```python
from tasks import greet

# タスクをキューに送信
result = greet.delay('Bob')

# タスクIDを取得
print(result.id)  # 'abc-123-def-456-...'

# 結果を待機して取得
print(result.get())  # 'Hello, Bob!'
```

### 2. apply_async() - 高度な非同期実行

より詳細な制御が可能:

```python
from tasks import greet
from datetime import datetime, timedelta

# 基本的な使い方
result = greet.apply_async(args=['Charlie'])

# 10秒後に実行
result = greet.apply_async(
    args=['David'],
    countdown=10
)

# 特定の時刻に実行
eta = datetime.now() + timedelta(hours=1)
result = greet.apply_async(
    args=['Eve'],
    eta=eta
)

# 有効期限を設定（1時間以内に実行されなければ破棄）
result = greet.apply_async(
    args=['Frank'],
    expires=3600
)
```

### 3. 同期実行（テスト用）

タスクを即座に実行（キューを使用しない）:

```python
from tasks import greet

# apply()で同期実行
result = greet.apply(args=['Grace'])
print(result.get())  # すぐに 'Hello, Grace!'
```

### delay() vs apply_async() の比較

```python
# delay() - シンプル
result = task.delay(arg1, arg2)

# apply_async() - 同等の処理
result = task.apply_async(args=[arg1, arg2])

# apply_async() - より高度な制御
result = task.apply_async(
    args=[arg1, arg2],
    kwargs={'key': 'value'},
    countdown=10,
    expires=3600,
    retry=True,
    retry_policy={
        'max_retries': 3,
        'interval_start': 0,
        'interval_step': 0.2,
        'interval_max': 0.2,
    }
)
```

## 実践例: 様々なタスク

### 例1: 計算タスク

```python
# tasks.py
from celery_app import app

@app.task
def add(x, y):
    """2つの数値を加算"""
    return x + y

@app.task
def multiply(x, y):
    """2つの数値を乗算"""
    return x * y

@app.task
def power(base, exponent):
    """累乗を計算"""
    return base ** exponent
```

使用例:

```python
from tasks import add, multiply, power

# 基本的な計算
result = add.delay(10, 20)
print(result.get())  # 30

result = multiply.delay(5, 6)
print(result.get())  # 30

result = power.delay(2, 10)
print(result.get())  # 1024
```

### 例2: 時間のかかるタスク

```python
# tasks.py
from celery_app import app
import time

@app.task
def long_running_task(duration):
    """指定秒数待機するタスク"""
    print(f'Starting task (will take {duration} seconds)')
    time.sleep(duration)
    print('Task completed')
    return f'Completed after {duration} seconds'

@app.task
def process_data(data_size):
    """データ処理のシミュレーション"""
    result = []
    for i in range(data_size):
        # 処理のシミュレーション
        time.sleep(0.01)
        result.append(i * i)
    return result
```

使用例:

```python
from tasks import long_running_task, process_data

# 非同期実行により、すぐに制御が返ってくる
result = long_running_task.delay(10)
print('Task submitted')  # すぐに表示

# 完了を待たずに次の処理へ
result2 = process_data.delay(100)

# 後で結果を取得
print(result.get())   # 10秒待ってから結果を返す
print(result2.get())  # データ処理の結果
```

### 例3: 外部API呼び出し

```python
# tasks.py
from celery_app import app
import requests

@app.task
def fetch_url(url):
    """URLからデータを取得"""
    response = requests.get(url)
    return {
        'status_code': response.status_code,
        'content_length': len(response.content),
        'url': url
    }

@app.task
def send_notification(message, webhook_url):
    """Webhookで通知を送信"""
    payload = {'text': message}
    response = requests.post(webhook_url, json=payload)
    return response.status_code
```

使用例:

```python
from tasks import fetch_url, send_notification

# 複数のURLを並列で取得
urls = [
    'https://example.com',
    'https://example.org',
    'https://example.net'
]

results = []
for url in urls:
    result = fetch_url.delay(url)
    results.append(result)

# すべての結果を取得
for result in results:
    data = result.get()
    print(f"{data['url']}: {data['status_code']}")
```

### 例4: ファイル処理

```python
# tasks.py
from celery_app import app
import os

@app.task
def count_lines(filename):
    """ファイルの行数をカウント"""
    with open(filename, 'r') as f:
        lines = len(f.readlines())
    return lines

@app.task
def process_csv(filename):
    """CSVファイルを処理"""
    import csv

    data = []
    with open(filename, 'r') as f:
        reader = csv.DictReader(f)
        for row in reader:
            # データ処理のロジック
            data.append(row)

    return {
        'filename': filename,
        'rows_processed': len(data)
    }
```

## タスクの状態確認

### ready() - タスクが完了したか確認

```python
from tasks import long_running_task

result = long_running_task.delay(5)

# すぐに確認
print(result.ready())  # False

# 5秒待つ
import time
time.sleep(5)

print(result.ready())  # True
```

### successful() - タスクが成功したか確認

```python
result = add.delay(1, 2)
result.get()  # タスク完了を待つ

print(result.successful())  # True
```

### failed() - タスクが失敗したか確認

```python
@app.task
def divide(x, y):
    return x / y

result = divide.delay(10, 0)  # ゼロ除算エラー

try:
    result.get()
except ZeroDivisionError:
    print(result.failed())  # True
```

### state - タスクの状態を取得

```python
result = long_running_task.delay(10)

print(result.state)  # 'PENDING'
# しばらく待つ...
print(result.state)  # 'SUCCESS'
```

タスクの状態:
- `PENDING`: タスクが送信された（デフォルト）
- `STARTED`: タスクが開始された
- `SUCCESS`: タスクが成功
- `FAILURE`: タスクが失敗
- `RETRY`: タスクをリトライ中
- `REVOKED`: タスクがキャンセルされた

## インタラクティブなテスト

### Pythonスクリプトでのテスト

```python
# test_tasks.py
from tasks import hello, greet, add

def main():
    print('=== Celery Task Test ===')

    # テスト1: 引数なしタスク
    print('\n1. Testing hello()')
    result = hello.delay()
    print(f'Task ID: {result.id}')
    print(f'Result: {result.get()}')

    # テスト2: 引数ありタスク
    print('\n2. Testing greet("World")')
    result = greet.delay('World')
    print(f'Result: {result.get()}')

    # テスト3: 複数タスクの並列実行
    print('\n3. Testing parallel execution')
    results = []
    for i in range(5):
        result = add.delay(i, i * 2)
        results.append(result)

    print('All tasks submitted')
    for i, result in enumerate(results):
        print(f'add({i}, {i*2}) = {result.get()}')

if __name__ == '__main__':
    main()
```

実行:

```bash
python test_tasks.py
```

### IPythonでのインタラクティブテスト

```python
# IPythonを起動
ipython

# タスクをインポート
from tasks import *

# タスクを実行
result = hello.delay()

# 結果を取得
result.get()

# タスクIDを確認
result.id

# 状態を確認
result.state
```

## ワーカーの起動オプション

### 基本的な起動

```bash
celery -A celery_app worker --loglevel=info
```

### ログレベルの変更

```bash
# デバッグ情報を表示
celery -A celery_app worker --loglevel=debug

# 警告以上のみ表示
celery -A celery_app worker --loglevel=warning

# エラーのみ表示
celery -A celery_app worker --loglevel=error
```

### 並行処理数の指定

```bash
# 4つのワーカープロセスで実行
celery -A celery_app worker --concurrency=4

# 自動（CPUコア数に基づく）
celery -A celery_app worker --concurrency=auto
```

### ワーカーに名前を付ける

```bash
celery -A celery_app worker --loglevel=info --hostname=worker1@%h
```

### 特定のキューのみ処理

```bash
celery -A celery_app worker --loglevel=info --queues=high_priority,default
```

## ベストプラクティス

### 1. タスクは純粋関数に

タスクは可能な限り副作用を持たない純粋関数にします。

```python
# Good
@app.task
def calculate_total(items):
    return sum(item['price'] for item in items)

# Avoid - グローバル状態に依存
total = 0

@app.task
def add_to_total(value):
    global total
    total += value
    return total
```

### 2. タスクは小さく保つ

大きなタスクは分割します。

```python
# Better
@app.task
def process_user(user_id):
    return process_single_user(user_id)

# Avoid
@app.task
def process_all_users():
    for user in User.objects.all():  # 数万件ある場合
        process_user(user)
```

### 3. わかりやすいタスク名

```python
# Good
@app.task(name='users.send_welcome_email')
def send_welcome_email(user_id):
    pass

# OK (自動生成される名前)
@app.task
def send_welcome_email(user_id):
    pass  # 名前: tasks.send_welcome_email
```

### 4. タスクにドキュメントを追加

```python
@app.task
def process_payment(order_id, amount):
    """
    支払い処理を実行する

    Args:
        order_id (int): 注文ID
        amount (float): 支払い金額

    Returns:
        dict: 処理結果（transaction_id, status等）
    """
    # 処理...
    pass
```

## よくある間違い

### 間違い1: 結果を待たずに使用

```python
# Wrong
result = add.delay(1, 2)
print(result)  # <AsyncResult: ...>  数値ではない！

# Correct
result = add.delay(1, 2)
print(result.get())  # 3
```

### 間違い2: ワーカーが起動していない

タスクを送信してもワーカーが起動していないと実行されません。

```bash
# 必ずワーカーを起動
celery -A celery_app worker --loglevel=info
```

### 間違い3: 循環インポート

```python
# tasks.py
from celery_app import app
from models import User  # OK

# celery_app.py
from celery import Celery
from tasks import *  # NG - 循環インポート
```

## まとめ

このセクションで学んだこと:

- ✅ `@app.task`デコレータでタスクを定義
- ✅ `task.delay()`で非同期実行
- ✅ `result.get()`で結果を取得
- ✅ ワーカーの起動とログの確認
- ✅ タスクの状態確認方法
- ✅ ベストプラクティス

## 練習問題

1. 2つの文字列を連結するタスク`concat(s1, s2)`を作成してください
2. リストの要素数をカウントするタスク`count_items(items)`を作成してください
3. 1から指定された数までの合計を計算するタスク`sum_range(n)`を作成してください
4. 上記のタスクをすべて実行し、結果を確認してください

## 次のステップ

次のセクション「[タスクの基本](./04_task_basics.md)」では、タスクのより高度な機能について学びます。

## 参考リソース

- [Celery Tasks](https://docs.celeryproject.org/en/stable/userguide/tasks.html)
- [Calling Tasks](https://docs.celeryproject.org/en/stable/userguide/calling.html)
