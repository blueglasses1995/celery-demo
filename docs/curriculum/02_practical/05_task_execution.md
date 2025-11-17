# 5. タスクの実行方法

## 学習目標

- `delay()`と`apply_async()`の違いを理解する
- タスクのシグネチャを使いこなす
- 実行オプション（eta、countdown、expires等）を活用する
- タスクIDを効果的に管理する

## delay() vs apply_async()

### delay() - シンプルで直感的

```python
@app.task
def add(x, y):
    return x + y

# 基本的な使い方
result = add.delay(4, 6)
```

`delay()`は`apply_async()`のショートカットです:

```python
# 以下は同じ
add.delay(4, 6)
add.apply_async(args=(4, 6))
```

### apply_async() - 高度な制御

```python
from datetime import datetime, timedelta

# 1. 基本的な実行
result = add.apply_async(args=(4, 6))

# 2. 10秒後に実行
result = add.apply_async(args=(4, 6), countdown=10)

# 3. 特定の時刻に実行
eta = datetime.now() + timedelta(hours=1)
result = add.apply_async(args=(4, 6), eta=eta)

# 4. 有効期限を設定
result = add.apply_async(
    args=(4, 6),
    expires=datetime.now() + timedelta(minutes=30)
)

# 5. 特定のキューに送信
result = add.apply_async(args=(4, 6), queue='high_priority')

# 6. リトライポリシーを設定
result = add.apply_async(
    args=(4, 6),
    retry=True,
    retry_policy={
        'max_retries': 3,
        'interval_start': 0,
        'interval_step': 0.2,
    }
)
```

## タスクのシグネチャ

シグネチャは、タスクの呼び出しをオブジェクトとしてカプセル化します。

### 基本的なシグネチャ

```python
from celery import signature

# シグネチャの作成
sig = signature('tasks.add', args=(4, 6))

# または短縮形
sig = add.signature(args=(4, 6))

# さらに短く
sig = add.s(4, 6)

# 実行
result = sig.delay()
print(result.get())  # 10
```

### シグネチャのオプション

```python
# カウントダウン付き
sig = add.s(4, 6).set(countdown=10)

# キュー指定
sig = add.s(4, 6).set(queue='math_tasks')

# 複数オプション
sig = add.s(4, 6).set(
    countdown=5,
    queue='high_priority',
    expires=300
)
```

### partial() - 部分適用

```python
# 一部の引数だけを指定
partial_add = add.s(4)  # yはまだ指定していない

# 後で残りの引数を渡す
result = partial_add.delay(6)  # 4 + 6 = 10
```

### immutable() - 不変シグネチャ

```python
# 結果を次のタスクに渡さない
sig = add.si(4, 6)  # immutable signature

# チェーンで使用する場合に便利
from celery import chain

@app.task
def multiply(x, y):
    return x * y

# 通常のシグネチャ
workflow = chain(add.s(4, 6), multiply.s(2))
# add(4, 6) = 10 → multiply(10, 2) = 20

# 不変シグネチャ
workflow = chain(add.si(4, 6), multiply.si(3, 7))
# add(4, 6) = 10, multiply(3, 7) = 21 (独立して実行)
```

## 実行オプション詳細

### countdown - 遅延実行

```python
# 10秒後に実行
send_reminder.apply_async(args=[user_id], countdown=10)

# 動的な遅延
delay_seconds = calculate_delay()
task.apply_async(args=[data], countdown=delay_seconds)
```

### eta - 指定時刻に実行

```python
from datetime import datetime, timedelta

# 1時間後
future_time = datetime.now() + timedelta(hours=1)
process_order.apply_async(args=[order_id], eta=future_time)

# 毎日午前2時に実行（Celery Beatと併用）
tomorrow_2am = datetime.now().replace(
    hour=2, minute=0, second=0, microsecond=0
) + timedelta(days=1)
cleanup_task.apply_async(eta=tomorrow_2am)
```

### expires - 有効期限

```python
from datetime import datetime, timedelta

# 30分以内に実行されなければ破棄
urgent_task.apply_async(
    args=[data],
    expires=datetime.now() + timedelta(minutes=30)
)

# または秒数で指定
urgent_task.apply_async(args=[data], expires=1800)  # 30分
```

### priority - 優先度

```python
# 高優先度（0-9、値が大きいほど高優先度）
critical_task.apply_async(args=[data], priority=9)

# 低優先度
background_task.apply_async(args=[data], priority=0)
```

**注意**: RabbitMQでは優先度がサポートされていますが、Redisでは限定的です。

### retry - リトライの有効化

```python
@app.task(max_retries=3)
def api_call(url):
    response = requests.get(url)
    response.raise_for_status()
    return response.json()

# リトライポリシー付きで実行
api_call.apply_async(
    args=['https://api.example.com/data'],
    retry=True,
    retry_policy={
        'max_retries': 3,
        'interval_start': 0,      # 最初のリトライまでの秒数
        'interval_step': 0.2,      # 各リトライ間の増加秒数
        'interval_max': 0.2,       # リトライ間隔の最大秒数
    }
)
```

## タスクIDの管理

### 自動生成されるタスクID

```python
result = add.delay(4, 6)
print(result.id)  # 'abc-123-def-456...'
```

### カスタムタスクID

```python
import uuid

# UUIDを生成
custom_id = str(uuid.uuid4())

# カスタムIDでタスクを実行
result = add.apply_async(
    args=(4, 6),
    task_id=custom_id
)

print(result.id == custom_id)  # True
```

### タスクIDを使った処理の重複防止

```python
def process_user_once(user_id):
    """同じユーザーの処理を重複させない"""
    # ユーザーIDをタスクIDとして使用
    task_id = f'process_user_{user_id}'

    # 既に実行中か確認
    from celery.result import AsyncResult
    existing = AsyncResult(task_id)

    if existing.state == 'PENDING':
        # まだ実行されていないので実行
        process_user.apply_async(
            args=[user_id],
            task_id=task_id
        )
    else:
        print(f'Task for user {user_id} already exists: {existing.state}')
```

### タスクIDでの結果取得

```python
from celery.result import AsyncResult

# タスクIDから結果オブジェクトを復元
task_id = 'abc-123-def-456'
result = AsyncResult(task_id)

# 状態を確認
print(result.state)  # 'SUCCESS', 'PENDING', etc.

# 結果を取得
if result.ready():
    print(result.get())
```

## 実践例

### 例1: バッチメール送信

```python
@app.task
def send_email(to, subject, body):
    # メール送信処理
    send_mail(to, subject, body)
    return f'Email sent to {to}'

def send_bulk_emails(recipients, subject, body):
    """大量のメールを順次送信"""
    results = []

    for i, recipient in enumerate(recipients):
        # 各メールを5秒間隔で送信（レート制限対策）
        result = send_email.apply_async(
            args=[recipient, subject, body],
            countdown=i * 5
        )
        results.append(result)

    return results

# 使用例
recipients = ['user1@example.com', 'user2@example.com', ...]
results = send_bulk_emails(recipients, 'Hello', 'Welcome!')
```

### 例2: 時限タスク

```python
@app.task
def delete_temp_file(filepath):
    import os
    if os.path.exists(filepath):
        os.remove(filepath)
        return f'Deleted {filepath}'
    return f'{filepath} not found'

def create_temp_file_with_cleanup(data):
    """一時ファイルを作成し、1時間後に自動削除"""
    import tempfile
    from datetime import datetime, timedelta

    # 一時ファイル作成
    with tempfile.NamedTemporaryFile(delete=False) as f:
        f.write(data.encode())
        filepath = f.name

    # 1時間後に削除タスクをスケジュール
    delete_time = datetime.now() + timedelta(hours=1)
    delete_temp_file.apply_async(
        args=[filepath],
        eta=delete_time
    )

    return filepath
```

### 例3: 優先度付きタスク処理

```python
@app.task
def process_order(order_id, priority='normal'):
    # 注文処理
    order = get_order(order_id)
    # ...処理...
    return f'Order {order_id} processed'

def submit_order(order_id, is_premium=False):
    """注文を処理キューに追加（プレミアム会員は優先）"""
    if is_premium:
        # 高優先度で即座に実行
        result = process_order.apply_async(
            args=[order_id, 'premium'],
            priority=9,
            queue='high_priority'
        )
    else:
        # 通常優先度
        result = process_order.apply_async(
            args=[order_id, 'normal'],
            priority=5,
            queue='default'
        )

    return result
```

### 例4: リトライ付きAPI呼び出し

```python
import requests
from celery.exceptions import Retry

@app.task(bind=True, max_retries=5)
def fetch_api_data(self, url):
    """APIからデータを取得（リトライ付き）"""
    try:
        response = requests.get(url, timeout=10)
        response.raise_for_status()
        return response.json()
    except requests.RequestException as exc:
        # 指数バックオフでリトライ
        retry_count = self.request.retries
        countdown = 2 ** retry_count  # 2, 4, 8, 16, 32秒
        raise self.retry(exc=exc, countdown=countdown)

# 使用
result = fetch_api_data.apply_async(
    args=['https://api.example.com/data'],
    retry=True
)
```

### 例5: タスクの条件付き実行

```python
@app.task
def process_if_recent(data_id, max_age_minutes=30):
    """データが新しい場合のみ処理"""
    from datetime import datetime, timedelta

    data = get_data(data_id)
    age = datetime.now() - data['created_at']

    if age < timedelta(minutes=max_age_minutes):
        # 処理を実行
        result = perform_processing(data)
        return {'processed': True, 'result': result}
    else:
        # 古すぎるのでスキップ
        return {'processed': False, 'reason': 'too_old'}

# 有効期限付きで実行
result = process_if_recent.apply_async(
    args=[data_id],
    expires=1800  # 30分以内に実行されなければ破棄
)
```

## タスクのキャンセル

### revoke() - タスクの取り消し

```python
# タスクを送信
result = long_task.delay(data)

# タスクをキャンセル
result.revoke()

# 実行中のタスクも強制終了
result.revoke(terminate=True)

# シグナルを送信（デフォルト: SIGTERM）
result.revoke(terminate=True, signal='SIGKILL')
```

### タスクIDでのキャンセル

```python
from celery_app import app

# タスクIDでキャンセル
app.control.revoke('task-id-abc-123')

# 複数のタスクをキャンセル
app.control.revoke([
    'task-id-1',
    'task-id-2',
    'task-id-3'
], terminate=True)
```

## まとめ

- ✅ `delay()`は簡単、`apply_async()`は高機能
- ✅ シグネチャでタスク呼び出しをオブジェクト化
- ✅ `countdown`、`eta`、`expires`で実行タイミングを制御
- ✅ `priority`で優先度を設定
- ✅ タスクIDで結果を追跡・管理
- ✅ `revoke()`でタスクをキャンセル

## 次のステップ

次のセクション「[タスクの状態管理](./06_task_states.md)」では、タスクの状態を詳しく学びます。
