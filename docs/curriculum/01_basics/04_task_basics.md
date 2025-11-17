# 4. タスクの基本

## 学習目標

このセクションを完了すると、以下ができるようになります:
- タスクデコレータのオプションを理解する
- 様々な方法でタスクに引数を渡す
- タスクの戻り値を効果的に使用する
- タスクの命名規則とベストプラクティスを理解する

## タスクデコレータの詳細

### 基本的な使い方

```python
from celery_app import app

@app.task
def simple_task():
    return 'Hello'
```

### デコレータのオプション

```python
@app.task(
    name='custom.task.name',           # カスタムタスク名
    bind=True,                          # タスクインスタンスをバインド
    max_retries=3,                      # 最大リトライ回数
    default_retry_delay=60,             # リトライ間隔（秒）
    time_limit=300,                     # タイムリミット（秒）
    soft_time_limit=240,                # ソフトタイムリミット
    ignore_result=False,                # 結果を保存するか
    track_started=True,                 # 開始状態を追跡
    acks_late=True,                     # レイトACKを有効化
    reject_on_worker_lost=True,         # ワーカー停止時にタスクを拒否
)
def advanced_task(x, y):
    return x + y
```

## タスクへの引数渡し

### 1. 位置引数（Positional Arguments）

```python
@app.task
def add(x, y):
    return x + y

# 実行
result = add.delay(10, 20)
print(result.get())  # 30
```

### 2. キーワード引数（Keyword Arguments）

```python
@app.task
def create_user(username, email, age=None):
    return {
        'username': username,
        'email': email,
        'age': age
    }

# 実行方法1: 位置引数
result = create_user.delay('alice', 'alice@example.com', 25)

# 実行方法2: キーワード引数
result = create_user.delay(
    username='bob',
    email='bob@example.com',
    age=30
)

# 実行方法3: apply_asyncで明示的に
result = create_user.apply_async(
    args=['charlie', 'charlie@example.com'],
    kwargs={'age': 35}
)
```

### 3. 可変長引数

```python
@app.task
def sum_all(*args):
    """任意の数の引数を合計"""
    return sum(args)

# 実行
result = sum_all.delay(1, 2, 3, 4, 5)
print(result.get())  # 15

@app.task
def create_report(**kwargs):
    """任意のキーワード引数を受け取る"""
    return {
        'report_type': kwargs.get('report_type', 'default'),
        'format': kwargs.get('format', 'pdf'),
        'options': kwargs
    }

# 実行
result = create_report.delay(
    report_type='sales',
    format='excel',
    include_charts=True,
    date_range='last_month'
)
```

### 4. 複雑なデータ構造

```python
@app.task
def process_data(data_dict):
    """辞書を受け取って処理"""
    total = data_dict.get('total', 0)
    items = data_dict.get('items', [])
    return {
        'total': total,
        'item_count': len(items),
        'processed': True
    }

# 実行
data = {
    'total': 1000,
    'items': [
        {'id': 1, 'name': 'Item 1'},
        {'id': 2, 'name': 'Item 2'}
    ],
    'metadata': {
        'source': 'api',
        'timestamp': '2024-01-01'
    }
}
result = process_data.delay(data)
```

### 5. リスト処理

```python
@app.task
def process_items(items):
    """リストの各要素を処理"""
    results = []
    for item in items:
        # 処理のロジック
        processed = item.upper() if isinstance(item, str) else item
        results.append(processed)
    return results

# 実行
items = ['apple', 'banana', 'cherry']
result = process_items.delay(items)
print(result.get())  # ['APPLE', 'BANANA', 'CHERRY']
```

## 引数のシリアライゼーション

### JSON（デフォルト）

```python
# celery_app.py
app.conf.update(
    task_serializer='json',
    result_serializer='json',
    accept_content=['json']
)
```

**利点**: 安全、言語間の互換性
**制限**: datetime、set、カスタムオブジェクトは直接送信不可

### 対応方法

```python
from datetime import datetime
import json

@app.task
def process_with_date(date_str):
    """文字列として日付を受け取る"""
    date = datetime.fromisoformat(date_str)
    # 処理...
    return date.strftime('%Y-%m-%d')

# 実行
now = datetime.now()
result = process_with_date.delay(now.isoformat())

# または、カスタムエンコーダを使用
class DateTimeEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, datetime):
            return obj.isoformat()
        return super().default(obj)
```

### Pickle（注意が必要）

```python
app.conf.update(
    task_serializer='pickle',
    result_serializer='pickle',
    accept_content=['pickle', 'json']
)
```

**警告**: セキュリティリスクがあるため、信頼できる環境でのみ使用

## タスクの戻り値

### 1. シンプルな値

```python
@app.task
def get_number():
    return 42

@app.task
def get_string():
    return 'Hello, World!'

@app.task
def get_boolean():
    return True
```

### 2. 辞書

```python
@app.task
def get_user_info(user_id):
    return {
        'id': user_id,
        'name': 'Alice',
        'email': 'alice@example.com',
        'created_at': '2024-01-01'
    }

# 使用
result = get_user_info.delay(123)
user_info = result.get()
print(user_info['name'])  # 'Alice'
```

### 3. リスト

```python
@app.task
def get_squares(n):
    return [i * i for i in range(n)]

# 使用
result = get_squares.delay(5)
squares = result.get()
print(squares)  # [0, 1, 4, 9, 16]
```

### 4. 複雑なデータ構造

```python
@app.task
def analyze_data(data):
    return {
        'summary': {
            'total': len(data),
            'average': sum(data) / len(data) if data else 0
        },
        'details': {
            'min': min(data) if data else None,
            'max': max(data) if data else None
        },
        'raw_data': data
    }

# 使用
result = analyze_data.delay([10, 20, 30, 40, 50])
analysis = result.get()
print(analysis['summary']['average'])  # 30.0
```

### 5. 戻り値なし（None）

```python
@app.task
def send_email(to, subject, body):
    # メール送信処理
    send_mail(to, subject, body)
    # 戻り値なし

# 結果を保存する必要がない場合
@app.task(ignore_result=True)
def log_event(event_type, data):
    # ログ記録
    logger.info(f'{event_type}: {data}')
```

## bindオプション - タスクインスタンスへのアクセス

### 基本的な使い方

```python
@app.task(bind=True)
def debug_task(self):
    print(f'Request: {self.request}')
    print(f'Task ID: {self.request.id}')
    print(f'Task Name: {self.request.task}')
    return 'Done'
```

### タスク情報の取得

```python
@app.task(bind=True)
def show_task_info(self, x, y):
    info = {
        'task_id': self.request.id,
        'task_name': self.name,
        'args': self.request.args,
        'kwargs': self.request.kwargs,
        'retries': self.request.retries,
    }
    return info

# 実行
result = show_task_info.delay(10, 20)
print(result.get())
# {
#     'task_id': 'abc-123-...',
#     'task_name': 'tasks.show_task_info',
#     'args': [10, 20],
#     'kwargs': {},
#     'retries': 0
# }
```

### リトライでの使用

```python
@app.task(bind=True, max_retries=3)
def fetch_data(self, url):
    try:
        response = requests.get(url)
        response.raise_for_status()
        return response.json()
    except requests.RequestException as exc:
        # リトライ
        raise self.retry(exc=exc, countdown=5)
```

### カスタム状態の更新

```python
@app.task(bind=True)
def long_process(self, items):
    total = len(items)
    for i, item in enumerate(items):
        # 進捗を更新
        self.update_state(
            state='PROGRESS',
            meta={
                'current': i,
                'total': total,
                'percent': int((i / total) * 100)
            }
        )
        # 処理
        process_item(item)

    return {'status': 'completed', 'total': total}
```

## タスクの命名

### 自動命名

```python
# tasks.py
@app.task
def send_email(to, subject):
    pass

# タスク名: 'tasks.send_email'
```

### カスタム命名

```python
@app.task(name='email.send')
def send_email(to, subject):
    pass

# タスク名: 'email.send'

@app.task(name='users.registration.welcome_email')
def send_welcome_email(user_id):
    pass

# タスク名: 'users.registration.welcome_email'
```

### 命名規則のベストプラクティス

```python
# アプリ名.モジュール.アクション
@app.task(name='myapp.orders.process')
def process_order(order_id):
    pass

@app.task(name='myapp.orders.send_confirmation')
def send_order_confirmation(order_id):
    pass

# カテゴリ.リソース.アクション
@app.task(name='email.user.welcome')
def send_user_welcome_email(user_id):
    pass

@app.task(name='email.order.confirmation')
def send_order_email(order_id):
    pass
```

## タスクのドキュメント化

### ドックストリング

```python
@app.task
def process_payment(order_id, amount, payment_method):
    """
    支払い処理を実行する

    このタスクは注文の支払い処理を非同期で実行します。
    支払いゲートウェイとの通信を行い、結果を返します。

    Args:
        order_id (int): 処理する注文のID
        amount (float): 支払い金額（税込）
        payment_method (str): 支払い方法 ('credit_card', 'bank_transfer', etc.)

    Returns:
        dict: 支払い結果
            - transaction_id (str): トランザクションID
            - status (str): 'success' または 'failed'
            - message (str): ステータスメッセージ

    Raises:
        PaymentGatewayError: 支払いゲートウェイとの通信エラー
        InvalidAmountError: 無効な金額が指定された場合

    Example:
        >>> result = process_payment.delay(12345, 1000.0, 'credit_card')
        >>> payment_result = result.get()
        >>> print(payment_result['status'])
        'success'
    """
    # 実装...
    pass
```

### タイプヒント

```python
from typing import Dict, List, Optional

@app.task
def calculate_statistics(
    data: List[float],
    include_median: bool = False
) -> Dict[str, float]:
    """
    データの統計情報を計算

    Args:
        data: 数値のリスト
        include_median: 中央値を含めるかどうか

    Returns:
        統計情報の辞書
    """
    result = {
        'mean': sum(data) / len(data),
        'min': min(data),
        'max': max(data),
    }

    if include_median:
        sorted_data = sorted(data)
        n = len(sorted_data)
        result['median'] = sorted_data[n // 2]

    return result
```

## 実践例: 実用的なタスク

### 例1: ユーザー登録処理

```python
@app.task(
    name='users.send_welcome_email',
    max_retries=3,
    default_retry_delay=300
)
def send_welcome_email(user_id: int) -> Dict[str, str]:
    """
    新規ユーザーにウェルカムメールを送信

    Args:
        user_id: ユーザーID

    Returns:
        送信結果
    """
    try:
        user = get_user_from_db(user_id)
        send_email(
            to=user['email'],
            subject='Welcome to our service!',
            template='welcome.html',
            context={'name': user['name']}
        )
        return {
            'status': 'sent',
            'user_id': user_id,
            'email': user['email']
        }
    except Exception as exc:
        # エラーログ
        logger.error(f'Failed to send welcome email to user {user_id}: {exc}')
        raise
```

### 例2: 画像処理

```python
@app.task(
    name='media.process_image',
    bind=True,
    time_limit=600
)
def process_uploaded_image(
    self,
    image_id: int,
    sizes: List[tuple] = [(150, 150), (300, 300), (600, 600)]
) -> Dict:
    """
    アップロードされた画像を処理

    Args:
        image_id: 画像ID
        sizes: 生成するサムネイルサイズのリスト

    Returns:
        処理結果
    """
    total_sizes = len(sizes)
    thumbnails = []

    for i, size in enumerate(sizes):
        # 進捗更新
        self.update_state(
            state='PROGRESS',
            meta={
                'current': i + 1,
                'total': total_sizes,
                'size': size
            }
        )

        # サムネイル生成
        thumbnail_url = create_thumbnail(image_id, size)
        thumbnails.append({
            'size': size,
            'url': thumbnail_url
        })

    return {
        'image_id': image_id,
        'thumbnails': thumbnails,
        'status': 'completed'
    }
```

### 例3: データエクスポート

```python
@app.task(
    name='reports.export_data',
    bind=True,
    ignore_result=False,
    track_started=True
)
def export_user_data(
    self,
    user_ids: List[int],
    format: str = 'csv'
) -> Dict[str, str]:
    """
    ユーザーデータをエクスポート

    Args:
        user_ids: エクスポートするユーザーIDのリスト
        format: エクスポート形式 ('csv' または 'json')

    Returns:
        エクスポートファイルの情報
    """
    import csv
    import tempfile
    from datetime import datetime

    total_users = len(user_ids)
    filename = f'user_export_{datetime.now():%Y%m%d_%H%M%S}.{format}'

    with tempfile.NamedTemporaryFile(mode='w', delete=False) as f:
        if format == 'csv':
            writer = csv.DictWriter(f, fieldnames=['id', 'name', 'email'])
            writer.writeheader()

            for i, user_id in enumerate(user_ids):
                # 進捗更新
                if i % 100 == 0:
                    self.update_state(
                        state='PROGRESS',
                        meta={'current': i, 'total': total_users}
                    )

                user = get_user_from_db(user_id)
                writer.writerow(user)

        filepath = f.name

    # ファイルをストレージにアップロード
    upload_url = upload_to_storage(filepath, filename)

    return {
        'filename': filename,
        'url': upload_url,
        'total_records': total_users,
        'format': format
    }
```

## まとめ

このセクションで学んだこと:

- ✅ タスクデコレータの様々なオプション
- ✅ 引数の渡し方（位置引数、キーワード引数、可変長引数）
- ✅ シリアライゼーションの理解
- ✅ 戻り値の設計
- ✅ `bind=True`でタスクインスタンスにアクセス
- ✅ タスクの命名規則
- ✅ ドキュメント化のベストプラクティス

## 練習問題

### 問題1: 複数引数のタスク

複数の数値を受け取り、その平均値を返すタスク`calculate_average`を作成してください。

```python
# 期待される動作
result = calculate_average.delay([10, 20, 30, 40, 50])
print(result.get())  # 30.0
```

### 問題2: キーワード引数のタスク

ユーザー情報（名前、メール、年齢）を受け取り、フォーマットされた文字列を返すタスク`format_user_info`を作成してください。

```python
# 期待される動作
result = format_user_info.delay(
    name='Alice',
    email='alice@example.com',
    age=25
)
print(result.get())
# "Name: Alice, Email: alice@example.com, Age: 25"
```

### 問題3: bindを使ったタスク

タスクIDを含む結果を返すタスク`task_with_id`を作成してください。

```python
# 期待される動作
result = task_with_id.delay('test')
info = result.get()
print(info['task_id'])  # タスクIDが表示される
```

## 次のステップ

基礎編はこれで完了です！

次は「[実践編](../02_practical/05_task_execution.md)」に進み、より高度なタスクの実行方法について学びます。

## 参考リソース

- [Celery Task Reference](https://docs.celeryproject.org/en/stable/reference/celery.app.task.html)
- [Task Best Practices](https://docs.celeryproject.org/en/stable/userguide/tasks.html#best-practices)
- [Serializers](https://docs.celeryproject.org/en/stable/userguide/calling.html#serializers)
