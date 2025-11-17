# 12. エラーハンドリング

## 学習目標

- タスクのリトライ機能を使いこなす
- 例外処理のベストプラクティスを理解する
- タイムアウトとエラーコールバックを活用する

## タスクのリトライ

### 基本的なリトライ

```python
@app.task(bind=True, max_retries=3)
def api_call(self, url):
    try:
        response = requests.get(url, timeout=10)
        response.raise_for_status()
        return response.json()
    except requests.RequestException as exc:
        # リトライ
        raise self.retry(exc=exc, countdown=60)  # 60秒後にリトライ
```

### 指数バックオフ

```python
@app.task(bind=True, max_retries=5)
def fetch_data(self, url):
    try:
        return requests.get(url).json()
    except Exception as exc:
        # 指数バックオフ: 2, 4, 8, 16, 32秒
        countdown = 2 ** self.request.retries
        raise self.retry(exc=exc, countdown=countdown, max_retries=5)
```

### autoretry_for - 自動リトライ

```python
@app.task(
    autoretry_for=(requests.RequestException,),
    retry_kwargs={'max_retries': 3},
    retry_backoff=True,  # 指数バックオフを有効化
    retry_backoff_max=600,  # 最大10分
    retry_jitter=True  # ランダムなジッター追加
)
def reliable_api_call(url):
    response = requests.get(url)
    response.raise_for_status()
    return response.json()
```

## 例外処理

### try-except

```python
@app.task
def process_data(data_id):
    try:
        data = fetch_data(data_id)
        result = process(data)
        save_result(result)
        return {'status': 'success', 'result': result}
    except DataNotFound as exc:
        logger.error(f'Data {data_id} not found: {exc}')
        return {'status': 'not_found', 'error': str(exc)}
    except ProcessingError as exc:
        logger.error(f'Processing failed: {exc}')
        return {'status': 'failed', 'error': str(exc)}
    except Exception as exc:
        logger.exception('Unexpected error')
        raise  # 予期しないエラーは再発生
```

### カスタム例外

```python
class RetryableError(Exception):
    """リトライ可能なエラー"""
    pass

class FatalError(Exception):
    """リトライ不可能なエラー"""
    pass

@app.task(bind=True, max_retries=3)
def robust_task(self, data):
    try:
        return process_data(data)
    except RetryableError as exc:
        # リトライ可能なエラー
        raise self.retry(exc=exc, countdown=30)
    except FatalError as exc:
        # リトライせずに失敗
        logger.error(f'Fatal error: {exc}')
        return {'status': 'fatal_error', 'error': str(exc)}
```

## タイムアウト

### ハードタイムリミット

```python
# グローバル設定
app.conf.task_time_limit = 300  # 5分

# タスクごとに設定
@app.task(time_limit=60)  # 60秒
def time_limited_task():
    # 60秒を超えるとSIGKILLで強制終了
    long_running_operation()
```

### ソフトタイムリミット

```python
from celery.exceptions import SoftTimeLimitExceeded

@app.task(soft_time_limit=30)
def task_with_cleanup():
    try:
        long_operation()
    except SoftTimeLimitExceeded:
        # クリーンアップ処理
        cleanup()
        raise
```

## エラーコールバック

### on_failure

```python
@app.task(bind=True)
def important_task(self):
    try:
        return perform_operation()
    except Exception as exc:
        # 失敗時の処理
        self.on_failure(exc, self.request.id, self.request.args, self.request.kwargs)
        raise

def on_failure(self, exc, task_id, args, kwargs, einfo):
    """タスク失敗時のコールバック"""
    send_alert(f'Task {task_id} failed: {exc}')
    log_failure(task_id, exc)
```

## 実践例

### 例1: API呼び出しのリトライ

```python
@app.task(
    bind=True,
    autoretry_for=(requests.RequestException,),
    retry_kwargs={'max_retries': 5},
    retry_backoff=True
)
def call_external_api(self, endpoint, data):
    """外部APIを呼び出す（自動リトライ付き）"""
    response = requests.post(
        f'https://api.example.com/{endpoint}',
        json=data,
        timeout=30
    )
    response.raise_for_status()
    return response.json()
```

### 例2: データベース操作のエラーハンドリング

```python
@app.task(bind=True, max_retries=3)
def update_user_data(self, user_id, data):
    try:
        with transaction.atomic():
            user = User.objects.select_for_update().get(id=user_id)
            user.update(data)
            user.save()
            return {'status': 'success', 'user_id': user_id}
    except User.DoesNotExist:
        return {'status': 'not_found', 'user_id': user_id}
    except DatabaseError as exc:
        # データベースエラーはリトライ
        raise self.retry(exc=exc, countdown=10)
```

## まとめ

- ✅ `retry()`でタスクをリトライ
- ✅ `autoretry_for`で自動リトライ
- ✅ 指数バックオフで効率的にリトライ
- ✅ タイムアウトで長時間タスクを制御

## 次のステップ

次のセクション「[タスクのルーティング](./13_routing.md)」では、キューとルーティングを学びます。
