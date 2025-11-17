# 1. Celeryの概要

## 学習目標

このセクションを完了すると、以下ができるようになります:
- Celeryが何であるかを説明できる
- 非同期タスクキューの概念を理解する
- Celeryが適用できるユースケースを識別できる
- Celeryのアーキテクチャの全体像を理解する

## Celeryとは何か

Celeryは、Pythonで書かれた**分散タスクキュー**のフレームワークです。

### 主な特徴

- **非同期処理**: 時間のかかる処理をバックグラウンドで実行
- **分散実行**: 複数のワーカーマシンで処理を分散
- **スケジューリング**: 定期的なタスクの実行をサポート
- **高い信頼性**: リトライ、タイムアウト、エラーハンドリング機能
- **柔軟性**: 様々なメッセージブローカーと結果バックエンドに対応

### なぜCeleryが必要なのか

通常のWebアプリケーションでは、ユーザーのリクエストに対してすぐに応答を返す必要があります。
しかし、以下のような時間のかかる処理があると問題が発生します:

```python
# 問題のあるコード例
def upload_video(request):
    video = request.FILES['video']
    save_to_storage(video)  # 数秒かかる
    create_thumbnail(video)  # 10秒かかる
    encode_video(video)      # 数分かかる
    send_notification()      # 数秒かかる
    return "Upload complete"  # ユーザーは数分待たされる！
```

**Celeryを使った解決策:**

```python
# Celeryを使った改善例
def upload_video(request):
    video = request.FILES['video']
    save_to_storage(video)
    # 重い処理はバックグラウンドで実行
    process_video_task.delay(video.id)
    return "Upload started"  # すぐに応答を返せる
```

## 非同期タスクキューの概念

### タスクキューの仕組み

```
[Webアプリ] → [メッセージ] → [キュー] → [ワーカー] → [実行]
                                ↓
                            [結果保存]
```

1. **プロデューサー**: タスクを作成してキューに送信
2. **ブローカー**: メッセージ（タスク）を保管・配送
3. **ワーカー**: キューからタスクを取り出して実行
4. **バックエンド**: タスクの結果を保存（オプション）

### 同期処理 vs 非同期処理

**同期処理:**
```
リクエスト → 処理1 → 処理2 → 処理3 → レスポンス
[========== ユーザーは待つ ==========]
```

**非同期処理:**
```
リクエスト → タスクをキューに送信 → レスポンス
                    ↓
           [バックグラウンドで実行]
                処理1 → 処理2 → 処理3
```

## Celeryのユースケース

### 1. メール送信

```python
@app.task
def send_welcome_email(user_id):
    user = User.objects.get(id=user_id)
    send_mail(
        'Welcome!',
        f'Hi {user.name}, welcome to our service!',
        'from@example.com',
        [user.email]
    )

# ユーザー登録時
def register_user(request):
    user = create_user(request.data)
    send_welcome_email.delay(user.id)  # 非同期でメール送信
    return Response("Registration complete")
```

### 2. 画像/動画処理

```python
@app.task
def process_uploaded_image(image_id):
    image = Image.objects.get(id=image_id)
    # サムネイル生成
    create_thumbnail(image, size=(150, 150))
    # 複数サイズの生成
    create_thumbnail(image, size=(300, 300))
    create_thumbnail(image, size=(600, 600))
    # 透かしの追加
    add_watermark(image)
```

### 3. データ処理・レポート生成

```python
@app.task
def generate_monthly_report(month, year):
    data = fetch_monthly_data(month, year)
    analysis = analyze_data(data)
    report = create_pdf_report(analysis)
    send_to_managers(report)
```

### 4. Webスクレイピング

```python
@app.task
def scrape_website(url):
    html = fetch_html(url)
    data = parse_html(html)
    save_to_database(data)

# 複数サイトを並列処理
urls = ['http://example1.com', 'http://example2.com', ...]
for url in urls:
    scrape_website.delay(url)
```

### 5. 定期的なタスク

```python
from celery.schedules import crontab

# 毎日午前2時にデータベースのバックアップ
@app.task
def backup_database():
    create_backup()
    upload_to_s3()

# スケジュール設定
app.conf.beat_schedule = {
    'daily-backup': {
        'task': 'tasks.backup_database',
        'schedule': crontab(hour=2, minute=0),
    },
}
```

### 6. API呼び出しのレート制限

```python
@app.task(rate_limit='10/m')  # 1分間に10回まで
def call_external_api(data):
    response = requests.post('https://api.example.com/data', json=data)
    return response.json()
```

## Celeryのアーキテクチャ

### 基本コンポーネント

```
┌─────────────┐
│ Application │  タスクを作成するアプリケーション
└──────┬──────┘
       │ タスクを送信
       ↓
┌─────────────┐
│   Broker    │  メッセージキュー（Redis/RabbitMQ等）
│  (Message   │  タスクを保管・配送
│   Queue)    │
└──────┬──────┘
       │ タスクを取得
       ↓
┌─────────────┐
│   Worker    │  タスクを実行するプロセス
│  (Celery    │  複数のワーカーを並列実行可能
│   Worker)   │
└──────┬──────┘
       │ 結果を保存
       ↓
┌─────────────┐
│   Backend   │  タスクの結果を保存（Redis/DB等）
│  (Result    │  結果の取得・確認に使用
│   Store)    │
└─────────────┘
```

### 詳細なアーキテクチャ

```
┌───────────────────────────────────────────────────┐
│             Your Application                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │  Django  │  │  Flask   │  │  Script  │       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘       │
└───────┼─────────────┼─────────────┼──────────────┘
        │             │             │
        └─────────────┴─────────────┘
                      │ task.delay()
                      ↓
        ┌─────────────────────────────┐
        │    Message Broker           │
        │  ┌─────────────────────┐   │
        │  │   Task Queue        │   │
        │  │  [Task1][Task2]...  │   │
        │  └─────────────────────┘   │
        │     Redis / RabbitMQ        │
        └──────────┬──────────────────┘
                   │
        ┌──────────┴──────────┬───────────┐
        │                     │           │
        ↓                     ↓           ↓
┌───────────────┐    ┌───────────────┐   ...
│ Celery Worker │    │ Celery Worker │
│   Process 1   │    │   Process 2   │
│               │    │               │
│ ┌───────────┐ │    │ ┌───────────┐ │
│ │   Task    │ │    │ │   Task    │ │
│ │ Execution │ │    │ │ Execution │ │
│ └───────────┘ │    │ └───────────┘ │
└───────┬───────┘    └───────┬───────┘
        │                    │
        └────────┬───────────┘
                 ↓
        ┌─────────────────────────────┐
        │    Result Backend           │
        │  ┌─────────────────────┐   │
        │  │   Task Results      │   │
        │  │  {id: result, ...}  │   │
        │  └─────────────────────┘   │
        │     Redis / Database        │
        └─────────────────────────────┘
```

### 各コンポーネントの役割

#### 1. Application（アプリケーション）
- タスクを定義
- タスクを実行依頼（キューに送信）
- 結果を取得（オプション）

#### 2. Message Broker（メッセージブローカー）
- タスクメッセージの保管
- ワーカーへの配送
- メッセージの永続化

**主な選択肢:**
- **Redis**: 高速、シンプル、メモリベース
- **RabbitMQ**: 高機能、信頼性が高い、ディスクベース
- **Amazon SQS**: クラウドネイティブ、管理不要

#### 3. Celery Worker（ワーカー）
- キューからタスクを取得
- タスクを実行
- 結果をバックエンドに保存

**特徴:**
- 複数ワーカーを並列実行可能
- 異なるマシンで実行可能（分散処理）
- プロセス、スレッド、geventなど複数の並行処理モデル

#### 4. Result Backend（結果バックエンド）
- タスクの実行結果を保存
- タスクの状態を保存
- 結果の取得・確認に使用

**主な選択肢:**
- **Redis**: 高速、一時的な結果保存
- **Database**: 永続的な保存が必要な場合
- **無効化**: 結果が不要な場合

## データフロー

### タスク実行の流れ

```python
# 1. タスクの定義
@app.task
def add(x, y):
    return x + y

# 2. タスクの送信
result = add.delay(4, 6)  # → メッセージブローカーへ送信

# 3. ワーカーでの処理
# - ワーカーがメッセージを受信
# - タスクを実行: 4 + 6 = 10
# - 結果をバックエンドに保存

# 4. 結果の取得
print(result.get())  # → 10
```

### メッセージの構造

タスク実行時にブローカーに送信されるメッセージの例:

```json
{
  "id": "4e6e6f7a-3b4c-4c5e-9a8f-1b2c3d4e5f6a",
  "task": "myapp.tasks.add",
  "args": [4, 6],
  "kwargs": {},
  "retries": 0,
  "eta": null,
  "expires": null
}
```

## Celeryの利点

### 1. スケーラビリティ
- ワーカーを追加するだけで処理能力を増強
- 水平スケーリングが容易

### 2. 信頼性
- タスクのリトライ機能
- タイムアウト設定
- エラーハンドリング

### 3. 柔軟性
- 様々なブローカーに対応
- 複数のプログラミングパラダイムをサポート
- カスタマイズ可能

### 4. 可視性
- タスクの状態追跡
- モニタリングツール（Flower）
- ロギング機能

### 5. 開発効率
- シンプルなAPI
- Djangoなど主要フレームワークとの統合
- 豊富なドキュメント

## Celeryの制限事項

### 注意すべきポイント

1. **追加の複雑性**
   - ブローカーとワーカーの管理が必要
   - デバッグが複雑になる

2. **リソース要件**
   - ブローカー（Redis/RabbitMQ）が必要
   - ワーカープロセスのメモリ使用

3. **学習コスト**
   - 概念の理解が必要
   - ベストプラクティスの習得

4. **即時性の欠如**
   - タスクの実行は非同期
   - わずかな遅延が発生

## まとめ

- Celeryは**分散タスクキュー**フレームワーク
- **非同期処理**でユーザー体験を向上
- **ブローカー**、**ワーカー**、**バックエンド**の3つの主要コンポーネント
- メール送信、画像処理、データ処理など様々なユースケースに対応
- スケーラブルで信頼性が高いが、追加の複雑性がある

## 次のステップ

次のセクション「[環境構築](./02_environment_setup.md)」では、実際にCeleryを使えるように開発環境をセットアップします。

## 理解度チェック

以下の質問に答えられるか確認しましょう:

1. Celeryの主な用途は何ですか？
2. タスクキューの基本的な仕組みを説明できますか？
3. メッセージブローカーの役割は何ですか？
4. Celeryが適しているユースケースを3つ挙げられますか？
5. 同期処理と非同期処理の違いは何ですか？

## 参考リソース

- [Celery公式ドキュメント](https://docs.celeryproject.org/)
- [Celery GitHub](https://github.com/celery/celery)
- [分散システムの基礎](https://en.wikipedia.org/wiki/Distributed_computing)
