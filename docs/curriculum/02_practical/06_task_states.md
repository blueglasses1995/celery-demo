# 6. タスクの状態管理

## 学習目標

- タスクの状態ライフサイクルを理解する
- AsyncResultを使って結果を取得する
- カスタム状態を作成・活用する
- 進捗状況を追跡する

## タスクの状態ライフサイクル

### 標準の状態

```python
PENDING    # タスクが送信された（デフォルト状態）
STARTED    # タスクが開始された
SUCCESS    # タスクが正常に完了
FAILURE    # タスクが失敗
RETRY      # タスクをリトライ中
REVOKED    # タスクがキャンセルされた
```

### 状態の遷移

```
PENDING → STARTED → SUCCESS
                 → FAILURE
                 → RETRY → STARTED → ...

PENDING → REVOKED
STARTED → REVOKED
```

## AsyncResult - 結果オブジェクト

### 基本的な使い方

```python
from tasks import add

# タスクを実行
result = add.delay(4, 6)

# 状態を確認
print(result.state)  # 'PENDING' or 'SUCCESS'

# 完了したか確認
print(result.ready())  # True or False

# 結果を取得（ブロッキング）
print(result.get())  # 10

# 成功したか確認
print(result.successful())  # True or False

# 失敗したか確認
print(result.failed())  # True or False
```

### get() のオプション

```python
# タイムアウト付き（10秒待つ）
try:
    result = task.delay()
    value = result.get(timeout=10)
except TimeoutError:
    print('Task did not complete in time')

# エラーを伝播させない
result = failing_task.delay()
value = result.get(propagate=False)  # 例外を発生させない

if result.failed():
    error = result.info  # 例外情報を取得
    print(f'Task failed: {error}')
```

### タスクIDから結果を取得

```python
from celery.result import AsyncResult

# タスクIDがあれば、どこからでも結果を取得できる
task_id = 'abc-123-def-456'
result = AsyncResult(task_id)

print(result.state)
if result.ready():
    print(result.get())
```

## 状態の詳細な確認

### info / result - タスクの詳細情報

```python
@app.task
def process_data(data):
    # 処理...
    return {'processed': len(data), 'timestamp': datetime.now()}

result = process_data.delay([1, 2, 3])
result.get()  # 完了を待つ

# 結果の詳細
print(result.info)
# または
print(result.result)
# {'processed': 3, 'timestamp': ...}
```

### traceback - エラー時のトレースバック

```python
@app.task
def failing_task():
    raise ValueError('Something went wrong')

result = failing_task.delay()

try:
    result.get()
except Exception:
    # トレースバックを取得
    print(result.traceback)
```

## カスタム状態

### update_state() - 状態の更新

```python
from celery import states

@app.task(bind=True)
def long_process(self, items):
    total = len(items)

    for i, item in enumerate(items):
        # カスタム状態を設定
        self.update_state(
            state='PROGRESS',
            meta={
                'current': i,
                'total': total,
                'percent': int((i / total) * 100),
                'status': f'Processing item {i+1}/{total}'
            }
        )

        # 処理
        process_item(item)
        time.sleep(1)

    return {
        'status': 'completed',
        'total': total
    }
```

### カスタム状態の取得

```python
result = long_process.delay(items)

# ポーリングで進捗を確認
import time

while not result.ready():
    if result.state == 'PROGRESS':
        info = result.info
        print(f"Progress: {info['percent']}% - {info['status']}")
    time.sleep(1)

print('Task completed!')
print(result.get())
```

## 進捗トラッキング

### 例1: ファイル処理の進捗

```python
@app.task(bind=True)
def process_file(self, filepath):
    """大きなファイルを行単位で処理"""
    # ファイルの総行数を取得
    with open(filepath, 'r') as f:
        total_lines = sum(1 for _ in f)

    # 処理
    processed = 0
    with open(filepath, 'r') as f:
        for i, line in enumerate(f, 1):
            # 処理のロジック
            process_line(line)
            processed += 1

            # 100行ごとに進捗を更新
            if i % 100 == 0:
                self.update_state(
                    state='PROGRESS',
                    meta={
                        'current': processed,
                        'total': total_lines,
                        'percent': int((processed / total_lines) * 100)
                    }
                )

    return {
        'status': 'completed',
        'lines_processed': processed
    }
```

### 例2: バッチ処理の進捗

```python
@app.task(bind=True)
def process_batch(self, item_ids):
    """アイテムのバッチ処理"""
    total = len(item_ids)
    results = []
    errors = []

    for i, item_id in enumerate(item_ids):
        try:
            result = process_item(item_id)
            results.append(result)
            status = 'success'
        except Exception as e:
            errors.append({'item_id': item_id, 'error': str(e)})
            status = 'error'

        # 進捗を更新
        self.update_state(
            state='PROGRESS',
            meta={
                'current': i + 1,
                'total': total,
                'percent': int(((i + 1) / total) * 100),
                'success_count': len(results),
                'error_count': len(errors),
                'last_status': status
            }
        )

    return {
        'status': 'completed',
        'total': total,
        'success': len(results),
        'errors': len(errors),
        'error_details': errors
    }
```

### 例3: Webアプリでの進捗表示

```python
# tasks.py
@app.task(bind=True)
def generate_report(self, report_id):
    steps = [
        'Fetching data',
        'Processing data',
        'Generating charts',
        'Creating PDF',
        'Uploading'
    ]

    for i, step in enumerate(steps):
        self.update_state(
            state='PROGRESS',
            meta={
                'current_step': i + 1,
                'total_steps': len(steps),
                'step_name': step,
                'percent': int(((i + 1) / len(steps)) * 100)
            }
        )

        # 各ステップの処理
        execute_step(step, report_id)
        time.sleep(2)  # シミュレーション

    return {'status': 'completed', 'report_id': report_id}

# Flask/Django ビュー
from flask import jsonify
from celery.result import AsyncResult

@app.route('/report/<task_id>')
def report_status(task_id):
    result = AsyncResult(task_id)

    if result.state == 'PENDING':
        response = {
            'state': result.state,
            'status': 'Task is waiting...'
        }
    elif result.state == 'PROGRESS':
        response = {
            'state': result.state,
            'current_step': result.info.get('current_step', 0),
            'total_steps': result.info.get('total_steps', 0),
            'step_name': result.info.get('step_name', ''),
            'percent': result.info.get('percent', 0)
        }
    elif result.state == 'SUCCESS':
        response = {
            'state': result.state,
            'result': result.info
        }
    else:  # FAILURE
        response = {
            'state': result.state,
            'status': str(result.info)
        }

    return jsonify(response)
```

## 複数タスクの状態管理

### ResultSet - 複数の結果をまとめて管理

```python
from celery.result import ResultSet

# 複数タスクを実行
results = [add.delay(i, i) for i in range(10)]

# ResultSetにまとめる
result_set = ResultSet(results)

# 全タスクが完了するまで待つ
result_set.join()  # または result_set.get()

# すべての結果を取得
values = result_set.results
```

### GroupResult - グループタスクの結果

```python
from celery import group

# グループタスクを実行
job = group(add.s(i, i) for i in range(10))
result = job.apply_async()

# グループ全体が完了したか確認
print(result.ready())

# 各タスクが成功したか確認
print(result.successful())

# すべての結果を取得
results = result.get()
print(results)  # [0, 2, 4, 6, 8, 10, 12, 14, 16, 18]

# 個別の結果にアクセス
for sub_result in result.results:
    print(sub_result.get())
```

## 状態の保存期間

### result_expires - 結果の有効期限

```python
# celery_app.py
app.conf.update(
    result_expires=3600,  # 1時間後に結果を削除
)

# または個別のタスクで設定
@app.task(result_expires=600)  # 10分
def temporary_task():
    return 'This result will expire soon'
```

### ignore_result - 結果を保存しない

```python
# 結果が不要なタスク
@app.task(ignore_result=True)
def send_notification(user_id, message):
    # 通知を送信
    send_push(user_id, message)
    # 戻り値は保存されない

# または実行時に指定
task.apply_async(args=[data], ignore_result=True)
```

## タスクの状態確認のベストプラクティス

### 1. ポーリング間隔を調整

```python
import time

result = long_task.delay()

# 適切な間隔でポーリング
while not result.ready():
    time.sleep(2)  # 2秒ごとに確認
    # 頻繁すぎるポーリングは避ける

print(result.get())
```

### 2. タイムアウトを設定

```python
result = task.delay()

try:
    value = result.get(timeout=60)  # 1分でタイムアウト
except TimeoutError:
    # タイムアウト時の処理
    result.revoke()  # タスクをキャンセル
    raise
```

### 3. エラーハンドリング

```python
result = risky_task.delay()

try:
    value = result.get(timeout=30)
    print(f'Success: {value}')
except Exception as exc:
    print(f'Task failed: {exc}')
    print(f'State: {result.state}')
    if result.traceback:
        print(f'Traceback:\n{result.traceback}')
```

## まとめ

- ✅ タスクの状態: PENDING, STARTED, SUCCESS, FAILURE, RETRY, REVOKED
- ✅ `AsyncResult`で結果と状態を取得
- ✅ `update_state()`でカスタム状態を設定
- ✅ 進捗トラッキングでユーザー体験を向上
- ✅ `result_expires`と`ignore_result`で効率化

## 次のステップ

次のセクション「[ワーカーの理解](./07_workers.md)」では、ワーカーの仕組みを学びます。
