# Celeryを使ったマイクロサービスアーキテクチャ設計パターン

## 概要

Celeryは、マイクロサービスアーキテクチャにおいて、サービス間の非同期通信、タスクの分散処理、イベント駆動アーキテクチャを実現する強力なツールです。

このドキュメントでは、実践的な設計パターンとその技術的原理を詳しく解説します。

---

## パターン1: タスクベース・マイクロサービス

### アーキテクチャ概要

各マイクロサービスが独立したCeleryワーカーとして動作し、タスクキューを通じて通信します。

```
┌─────────────────────────────────────────────────────────────────────┐
│              Task-Based Microservices Architecture                   │
└─────────────────────────────────────────────────────────────────────┘

                          ┌──────────────────┐
                          │   API Gateway    │
                          │   (Flask/Django) │
                          └────────┬─────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    │              │              │
                    ▼              ▼              ▼
         ┌─────────────────────────────────────────────────┐
         │         Message Broker (Redis/RabbitMQ)          │
         │                                                   │
         │  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
         │  │Queue:    │  │Queue:    │  │Queue:    │      │
         │  │user      │  │order     │  │payment   │      │
         │  └──────────┘  └──────────┘  └──────────┘      │
         └────┬─────────────────┬─────────────────┬────────┘
              │                 │                 │
              ▼                 ▼                 ▼
    ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
    │ User Service    │ │ Order Service   │ │ Payment Service │
    │                 │ │                 │ │                 │
    │ ┌─────────────┐ │ │ ┌─────────────┐ │ │ ┌─────────────┐ │
    │ │Celery Worker│ │ │ │Celery Worker│ │ │ │Celery Worker│ │
    │ │             │ │ │ │             │ │ │ │             │ │
    │ │ @task       │ │ │ │ @task       │ │ │ │ @task       │ │
    │ │ create_user │ │ │ │ create_order│ │ │ │ process_pay │ │
    │ │ update_user │ │ │ │ ship_order  │ │ │ │ refund      │ │
    │ └─────────────┘ │ │ └─────────────┘ │ │ └─────────────┘ │
    │                 │ │                 │ │                 │
    │ ┌─────────────┐ │ │ ┌─────────────┐ │ │ ┌─────────────┐ │
    │ │   Database  │ │ │ │   Database  │ │ │ │   Database  │ │
    │ │   users_db  │ │ │ │   orders_db │ │ │ │  payments_db│ │
    │ └─────────────┘ │ │ └─────────────┘ │ │ └─────────────┘ │
    └─────────────────┘ └─────────────────┘ └─────────────────┘
```

### 実装例

#### User Service
```python
# services/user_service/tasks.py
from celery import Celery

app = Celery('user_service', broker='redis://localhost:6379/0')

app.conf.task_routes = {
    'user_service.*': {'queue': 'user'},
}

@app.task(name='user_service.create_user')
def create_user(email, username):
    """ユーザーを作成"""
    user = User.objects.create(email=email, username=username)

    # 他のサービスに通知（イベント発行）
    from services.order_service.tasks import setup_user_account
    setup_user_account.delay(user.id)

    return {'user_id': user.id, 'status': 'created'}

@app.task(name='user_service.update_profile')
def update_profile(user_id, profile_data):
    """プロフィールを更新"""
    user = User.objects.get(id=user_id)
    user.profile.update(profile_data)
    return {'user_id': user_id, 'status': 'updated'}
```

#### Order Service
```python
# services/order_service/tasks.py
from celery import Celery

app = Celery('order_service', broker='redis://localhost:6379/0')

app.conf.task_routes = {
    'order_service.*': {'queue': 'order'},
}

@app.task(name='order_service.create_order')
def create_order(user_id, items):
    """注文を作成"""
    order = Order.objects.create(user_id=user_id, items=items)
    total = calculate_total(items)

    # Payment Serviceを呼び出し
    from services.payment_service.tasks import process_payment
    payment_result = process_payment.delay(order.id, total)

    return {'order_id': order.id, 'payment_task_id': payment_result.id}

@app.task(name='order_service.setup_user_account')
def setup_user_account(user_id):
    """新規ユーザーの注文アカウントをセットアップ"""
    OrderAccount.objects.create(user_id=user_id)
    return {'user_id': user_id, 'status': 'setup_complete'}
```

#### Payment Service
```python
# services/payment_service/tasks.py
from celery import Celery

app = Celery('payment_service', broker='redis://localhost:6379/0')

app.conf.task_routes = {
    'payment_service.*': {'queue': 'payment'},
}

@app.task(bind=True, name='payment_service.process_payment', max_retries=3)
def process_payment(self, order_id, amount):
    """支払いを処理"""
    try:
        payment = Payment.objects.create(order_id=order_id, amount=amount)

        # 外部決済ゲートウェイを呼び出し
        result = charge_credit_card(amount)

        payment.status = 'completed'
        payment.save()

        # Order Serviceに通知
        from services.order_service.tasks import confirm_order
        confirm_order.delay(order_id)

        return {'payment_id': payment.id, 'status': 'success'}

    except PaymentGatewayError as exc:
        # リトライ
        raise self.retry(exc=exc, countdown=60)
```

### 技術的原理

#### サービス間通信フロー

```
┌──────────────────────────────────────────────────────────────┐
│           Inter-Service Communication Flow                    │
└──────────────────────────────────────────────────────────────┘

1. クライアントリクエスト
   │
   ▼
┌────────────────────────────────────────────────────────┐
│ API Gateway                                             │
│   POST /api/orders                                      │
│   {user_id: 123, items: [...]}                         │
└────────────────┬───────────────────────────────────────┘
                 │
                 │ 1. order_service.create_order.delay()
                 ▼
┌────────────────────────────────────────────────────────┐
│ Message Broker                                          │
│  Queue: "order"                                         │
│  ┌──────────────────────────────────────────────────┐ │
│  │ Task: create_order(user_id=123, items=[...])    │ │
│  └──────────────────────────────────────────────────┘ │
└────────────────┬───────────────────────────────────────┘
                 │
                 │ 2. Worker retrieves task
                 ▼
┌────────────────────────────────────────────────────────┐
│ Order Service Worker                                    │
│                                                          │
│  1. Order.objects.create(...)                          │
│     order_id = 456                                      │
│                                                          │
│  2. calculate_total(items)                             │
│     total = 10000                                       │
│                                                          │
│  3. Call Payment Service                                │
│     payment_service.process_payment.delay(456, 10000)  │
└────────────────┬───────────────────────────────────────┘
                 │
                 │ 3. Cross-service task call
                 ▼
┌────────────────────────────────────────────────────────┐
│ Message Broker                                          │
│  Queue: "payment"                                       │
│  ┌──────────────────────────────────────────────────┐ │
│  │ Task: process_payment(order_id=456, amount=10000)│ │
│  └──────────────────────────────────────────────────┘ │
└────────────────┬───────────────────────────────────────┘
                 │
                 │ 4. Payment worker processes
                 ▼
┌────────────────────────────────────────────────────────┐
│ Payment Service Worker                                  │
│                                                          │
│  1. Payment.objects.create(...)                        │
│  2. charge_credit_card(10000)                          │
│  3. payment.status = 'completed'                       │
│  4. Notify Order Service                                │
│     order_service.confirm_order.delay(456)             │
└────────────────┬───────────────────────────────────────┘
                 │
                 │ 5. Confirmation
                 ▼
┌────────────────────────────────────────────────────────┐
│ Order Service Worker                                    │
│                                                          │
│  order.status = 'confirmed'                            │
│  send_confirmation_email(order_id)                     │
└────────────────────────────────────────────────────────┘
```

### メリット

✅ **疎結合**: サービス間が直接通信せず、キューを介して通信
✅ **スケーラビリティ**: 各サービスを独立してスケール可能
✅ **障害耐性**: あるサービスが停止しても、他のサービスは影響を受けない
✅ **非同期処理**: ブロッキングせずに処理を継続
✅ **リトライ機能**: 自動的に失敗したタスクを再試行

### デメリット

❌ **複雑性の増加**: サービス間の依存関係が見えにくい
❌ **デバッグの困難**: 分散トレーシングが必要
❌ **データ整合性**: 最終的整合性のみ保証（トランザクションなし）
❌ **遅延**: 同期処理より遅い

### ユースケース

- **Eコマース**: 注文、決済、在庫管理
- **ソーシャルメディア**: 投稿、通知、フィード生成
- **IoTプラットフォーム**: デバイス管理、データ収集、分析

### ブローカー選択: RabbitMQ

**推奨理由:**
- メッセージの永続性が高い
- 複雑なルーティングが可能
- 優先度キューのサポート
- トランザクション的なメッセージ配送

---

## パターン2: イベント駆動アーキテクチャ

### アーキテクチャ概要

イベントの発行と購読（Pub/Sub）パターンで、サービス間を疎結合に保ちます。

```
┌─────────────────────────────────────────────────────────────────────┐
│              Event-Driven Architecture (Pub/Sub)                     │
└─────────────────────────────────────────────────────────────────────┘

                    ┌───────────────────────┐
                    │   Event Publishers    │
                    │  (各マイクロサービス)  │
                    └───────────┬───────────┘
                                │
                                │ Events
                                ▼
         ┌──────────────────────────────────────────────┐
         │        Message Broker (RabbitMQ)             │
         │                                               │
         │  ┌────────────────────────────────────────┐  │
         │  │      Topic Exchange: "events"          │  │
         │  │                                         │  │
         │  │  routing_key patterns:                 │  │
         │  │  - user.created                        │  │
         │  │  - user.updated                        │  │
         │  │  - order.created                       │  │
         │  │  - order.completed                     │  │
         │  │  - payment.processed                   │  │
         │  └───────────┬────────────────────────────┘  │
         │              │                                │
         │              │ Bindings (routing patterns)   │
         │              │                                │
         │     ┌────────┼────────┬────────┐            │
         │     │        │        │        │            │
         │     ▼        ▼        ▼        ▼            │
         │  ┌──────┐┌──────┐┌──────┐┌──────┐          │
         │  │Queue1││Queue2││Queue3││Queue4│          │
         │  │user.*││order.*││pay.*││*.*   │          │
         │  └──────┘└──────┘└──────┘└──────┘          │
         └─────┬──────┬──────┬──────┬─────────────────┘
               │      │      │      │
               ▼      ▼      ▼      ▼
    ┌──────────────────────────────────────────────┐
    │         Event Subscribers (Workers)          │
    │                                               │
    │  ┌──────────────┐  ┌──────────────┐         │
    │  │Notification  │  │Analytics     │         │
    │  │Service       │  │Service       │         │
    │  │              │  │              │         │
    │  │Subscribe:    │  │Subscribe:    │         │
    │  │- user.*      │  │- order.*     │         │
    │  │- order.*     │  │- payment.*   │         │
    │  └──────────────┘  └──────────────┘         │
    │                                               │
    │  ┌──────────────┐  ┌──────────────┐         │
    │  │Email         │  │Audit Log     │         │
    │  │Service       │  │Service       │         │
    │  │              │  │              │         │
    │  │Subscribe:    │  │Subscribe:    │         │
    │  │- user.created│  │- *.*         │         │
    │  │- order.comp. │  │(全イベント)   │         │
    │  └──────────────┘  └──────────────┘         │
    └──────────────────────────────────────────────┘
```

### 実装例

#### イベント発行側

```python
# services/user_service/events.py
from celery import Celery

app = Celery('user_service', broker='amqp://localhost')

# Topic Exchangeの設定
app.conf.task_routes = {
    'events.*': {
        'exchange': 'events',
        'exchange_type': 'topic',
    }
}

class EventPublisher:
    """イベント発行クラス"""

    @staticmethod
    def publish_event(routing_key, event_data):
        """イベントを発行"""
        app.send_task(
            'events.publish',
            args=[event_data],
            routing_key=routing_key,
            exchange='events',
            exchange_type='topic'
        )

# User Serviceのタスク
@app.task(name='user_service.create_user')
def create_user(email, username):
    # ユーザー作成
    user = User.objects.create(email=email, username=username)

    # イベントを発行
    EventPublisher.publish_event(
        routing_key='user.created',
        event_data={
            'event_type': 'user.created',
            'user_id': user.id,
            'email': user.email,
            'username': user.username,
            'timestamp': datetime.now().isoformat()
        }
    )

    return {'user_id': user.id}

@app.task(name='user_service.update_user')
def update_user(user_id, updates):
    user = User.objects.get(id=user_id)
    user.update(updates)

    # 更新イベントを発行
    EventPublisher.publish_event(
        routing_key='user.updated',
        event_data={
            'event_type': 'user.updated',
            'user_id': user.id,
            'changes': updates,
            'timestamp': datetime.now().isoformat()
        }
    )

    return {'user_id': user_id, 'status': 'updated'}
```

#### イベント購読側

```python
# services/notification_service/subscribers.py
from celery import Celery
from kombu import Queue, Exchange

app = Celery('notification_service', broker='amqp://localhost')

# Topic Exchangeと購読キューの設定
events_exchange = Exchange('events', type='topic', durable=True)

app.conf.task_queues = (
    # user.*とorder.*イベントを購読
    Queue(
        'notification_queue',
        exchange=events_exchange,
        routing_key='user.*',
        durable=True
    ),
    Queue(
        'notification_queue',
        exchange=events_exchange,
        routing_key='order.*',
        durable=True
    ),
)

@app.task(name='notification_service.handle_user_created')
def handle_user_created(event_data):
    """user.createdイベントのハンドラ"""
    user_id = event_data['user_id']
    email = event_data['email']

    # ウェルカムメールを送信
    send_welcome_email(email)

    # プッシュ通知
    send_push_notification(user_id, 'Welcome to our service!')

    return {'status': 'notification_sent', 'user_id': user_id}

@app.task(name='notification_service.handle_order_completed')
def handle_order_completed(event_data):
    """order.completedイベントのハンドラ"""
    order_id = event_data['order_id']
    user_id = event_data['user_id']

    # 注文完了通知
    send_order_confirmation(user_id, order_id)

    return {'status': 'confirmation_sent', 'order_id': order_id}

# イベントルーター
@app.task(name='events.publish')
def route_event(event_data):
    """イベントを適切なハンドラにルーティング"""
    event_type = event_data.get('event_type')

    handlers = {
        'user.created': handle_user_created,
        'user.updated': lambda e: None,  # 何もしない
        'order.completed': handle_order_completed,
    }

    handler = handlers.get(event_type)
    if handler:
        handler.delay(event_data)
```

#### Analytics Service（全イベントを購読）

```python
# services/analytics_service/subscribers.py
from celery import Celery
from kombu import Queue, Exchange

app = Celery('analytics_service', broker='amqp://localhost')

events_exchange = Exchange('events', type='topic', durable=True)

app.conf.task_queues = (
    # 全イベント（*.*）を購読
    Queue(
        'analytics_queue',
        exchange=events_exchange,
        routing_key='#',  # すべてのイベント
        durable=True
    ),
)

@app.task(name='analytics_service.track_event')
def track_event(event_data):
    """すべてのイベントを記録"""
    Event.objects.create(
        event_type=event_data['event_type'],
        data=event_data,
        timestamp=event_data['timestamp']
    )

    # リアルタイム分析
    update_metrics(event_data)

    return {'status': 'tracked'}
```

### 技術的原理

#### Topic Exchange Routing

```
┌─────────────────────────────────────────────────────────────┐
│              RabbitMQ Topic Exchange Routing                 │
└─────────────────────────────────────────────────────────────┘

Event Published:
  routing_key: "user.created"

           ┌──────────────────┐
           │ Topic Exchange   │
           │    "events"      │
           └────────┬─────────┘
                    │
        ┌───────────┼───────────┬───────────────┐
        │           │           │               │
Binding │           │           │               │
Pattern │           │           │               │
        ▼           ▼           ▼               ▼
    ┌───────┐  ┌───────┐  ┌───────┐      ┌───────┐
    │user.* │  │order.*│  │user.  │      │  #    │
    │       │  │       │  │created│      │(全て) │
    │ ✓Match│  │×No    │  │ ✓Match│      │✓Match │
    └───┬───┘  └───────┘  └───┬───┘      └───┬───┘
        │                      │              │
        ▼                      ▼              ▼
    Queue1               Queue2          Queue3
    (Notify)            (Email)         (Analytics)

Binding Pattern Syntax:
  *  (star)  : 1単語に一致
               例: user.* → user.created, user.updated

  #  (hash)  : 0個以上の単語に一致
               例: user.# → user, user.created, user.profile.updated
```

### メリット

✅ **完全な疎結合**: パブリッシャーとサブスクライバーが互いを知らない
✅ **拡張性**: 新しいサブスクライバーを追加しても既存コードに影響なし
✅ **柔軟性**: イベントのフィルタリングが容易
✅ **監査**: すべてのイベントを記録可能
✅ **並列処理**: 複数のサブスクライバーが同時に処理

### デメリット

❌ **デバッグの困難**: イベントフローが追跡しにくい
❌ **イベント爆発**: イベントが多すぎると管理が困難
❌ **順序保証なし**: イベントの処理順序は保証されない
❌ **重複処理**: 同じイベントが複数回処理される可能性

### ユースケース

- **通知システム**: ユーザーへの各種通知
- **監査ログ**: すべての操作を記録
- **リアルタイム分析**: ユーザー行動の追跡
- **データ同期**: 複数のデータストア間での同期

### ブローカー選択: RabbitMQ

**必須理由:**
- Topic Exchangeのサポート
- 柔軟なルーティングパターン
- メッセージの永続性
- 複数サブスクライバーへのブロードキャスト

---

## パターン3: サーガパターン（分散トランザクション）

### アーキテクチャ概要

長時間実行される複雑なビジネスプロセスを、複数のステップに分割し、各ステップが成功または失敗時に補償トランザクションを実行します。

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Saga Pattern Architecture                         │
└─────────────────────────────────────────────────────────────────────┘

                        ┌──────────────┐
                        │ Saga         │
                        │ Coordinator  │
                        └──────┬───────┘
                               │
                               │ Orchestrates
                               ▼
         ┌──────────────────────────────────────────────┐
         │          Celery Workflow (Chain)             │
         │                                               │
         │  ┌────────┐    ┌────────┐    ┌────────┐    │
         │  │ Step 1 │───>│ Step 2 │───>│ Step 3 │    │
         │  │Reserve │    │Process │    │Confirm │    │
         │  │Inventory    │Payment │    │Order   │    │
         │  └────┬───┘    └────┬───┘    └────┬───┘    │
         │       │ Success     │ Success     │         │
         │       │             │             │         │
         │       │ Failure     │ Failure     │ Failure │
         │       ▼             ▼             ▼         │
         │  ┌────────┐    ┌────────┐    ┌────────┐    │
         │  │Compensate   │Compensate   │Compensate   │
         │  │(None)  │    │Unreserve│    │Refund   │   │
         │  └────────┘    │Inventory│    │& Cancel │   │
         │                └────────┘    └────────┘    │
         └──────────────────────────────────────────────┘
                               │
                               ▼
              ┌────────────────────────────────┐
              │  Services                       │
              │  ┌──────┐ ┌──────┐ ┌──────┐   │
              │  │Invent│ │Payment Payment│   │
              │  │ory   │ │       │       │   │
              │  └──────┘ └──────┘ └──────┘   │
              └────────────────────────────────┘
```

### 実装例

#### Saga Orchestrator

```python
# services/order_service/sagas.py
from celery import Celery, chain, group
from celery.exceptions import SoftTimeLimitExceeded

app = Celery('order_orchestrator', broker='redis://localhost:6379/0')

class OrderSaga:
    """注文処理のSagaオーケストレーター"""

    def __init__(self, order_id):
        self.order_id = order_id
        self.compensation_stack = []  # 補償トランザクションのスタック

    def execute(self):
        """Sagaを実行"""
        try:
            # Step 1: 在庫を予約
            inventory_result = self.reserve_inventory()
            self.compensation_stack.append(self.compensate_inventory)

            # Step 2: 支払いを処理
            payment_result = self.process_payment()
            self.compensation_stack.append(self.compensate_payment)

            # Step 3: 注文を確定
            order_result = self.confirm_order()

            # すべて成功
            return {
                'status': 'success',
                'order_id': self.order_id,
                'steps_completed': ['inventory', 'payment', 'order']
            }

        except Exception as exc:
            # 失敗したら補償トランザクションを実行
            return self.compensate(exc)

    def reserve_inventory(self):
        """在庫を予約"""
        from services.inventory_service.tasks import reserve_items
        result = reserve_items.delay(self.order_id)
        return result.get(timeout=30)

    def process_payment(self):
        """支払いを処理"""
        from services.payment_service.tasks import charge_customer
        result = charge_customer.delay(self.order_id)
        return result.get(timeout=30)

    def confirm_order(self):
        """注文を確定"""
        order = Order.objects.get(id=self.order_id)
        order.status = 'confirmed'
        order.save()
        return {'order_id': self.order_id}

    def compensate(self, error):
        """補償トランザクションを実行（ロールバック）"""
        # スタックを逆順に実行
        compensation_results = []

        for compensate_fn in reversed(self.compensation_stack):
            try:
                result = compensate_fn()
                compensation_results.append(result)
            except Exception as comp_exc:
                # 補償トランザクションの失敗をログ
                logger.error(f'Compensation failed: {comp_exc}')

        return {
            'status': 'compensated',
            'order_id': self.order_id,
            'original_error': str(error),
            'compensations': compensation_results
        }

    def compensate_inventory(self):
        """在庫予約の補償（解放）"""
        from services.inventory_service.tasks import release_reservation
        result = release_reservation.delay(self.order_id)
        return result.get(timeout=10)

    def compensate_payment(self):
        """支払いの補償（返金）"""
        from services.payment_service.tasks import refund_customer
        result = refund_customer.delay(self.order_id)
        return result.get(timeout=10)

# Celeryタスクとしてラップ
@app.task(bind=True, name='order_orchestrator.execute_order_saga')
def execute_order_saga(self, order_id):
    """注文Sagaを実行"""
    saga = OrderSaga(order_id)
    return saga.execute()
```

#### Celery Chainを使った実装（宣言的）

```python
# services/order_service/sagas_declarative.py
from celery import Celery, chain, chord
from celery.canvas import Signature

app = Celery('order_orchestrator', broker='redis://localhost:6379/0')

@app.task(bind=True, name='saga.reserve_inventory')
def reserve_inventory(self, order_id):
    """在庫を予約"""
    try:
        inventory = Inventory.reserve(order_id)
        return {'status': 'reserved', 'inventory_id': inventory.id}
    except InsufficientStockError as exc:
        # 失敗したらSagaを中止
        raise self.retry(exc=exc, max_retries=0)

@app.task(bind=True, name='saga.process_payment')
def process_payment(self, prev_result, order_id):
    """支払いを処理"""
    try:
        payment = Payment.charge(order_id)
        return {
            **prev_result,
            'status': 'payment_processed',
            'payment_id': payment.id
        }
    except PaymentFailedError as exc:
        # 補償: 在庫を解放
        compensate_inventory.delay(prev_result['inventory_id'])
        raise

@app.task(bind=True, name='saga.confirm_order')
def confirm_order(self, prev_result, order_id):
    """注文を確定"""
    try:
        order = Order.objects.get(id=order_id)
        order.status = 'confirmed'
        order.save()
        return {
            **prev_result,
            'status': 'confirmed',
            'order_id': order_id
        }
    except Exception as exc:
        # 補償: 在庫を解放 & 返金
        compensate_inventory.delay(prev_result['inventory_id'])
        compensate_payment.delay(prev_result['payment_id'])
        raise

# 補償トランザクション
@app.task(name='saga.compensate_inventory')
def compensate_inventory(inventory_id):
    """在庫予約を取り消し"""
    Inventory.release(inventory_id)
    return {'status': 'inventory_released'}

@app.task(name='saga.compensate_payment')
def compensate_payment(payment_id):
    """支払いを返金"""
    Payment.refund(payment_id)
    return {'status': 'payment_refunded'}

# Sagaワークフローを構築
def create_order_saga(order_id):
    """注文処理のSagaワークフローを構築"""
    workflow = chain(
        reserve_inventory.s(order_id),
        process_payment.s(order_id),
        confirm_order.s(order_id)
    )
    return workflow.apply_async()
```

### 技術的原理

#### Saga実行フロー（成功ケース）

```
┌──────────────────────────────────────────────────────────────┐
│              Saga Execution Flow (Success)                    │
└──────────────────────────────────────────────────────────────┘

T=0s    Saga Started: create_order_saga(order_id=123)
        │
        ├─> Step 1: reserve_inventory(123)
        │   ├─> Inventory.reserve(order_id=123)
        │   ├─> inventory_id = 456
        │   └─> ✓ SUCCESS
        │
T=2s    ├─> Step 2: process_payment(prev_result, 123)
        │   ├─> Payment.charge(order_id=123)
        │   ├─> payment_id = 789
        │   └─> ✓ SUCCESS
        │
T=5s    ├─> Step 3: confirm_order(prev_result, 123)
        │   ├─> Order.status = 'confirmed'
        │   └─> ✓ SUCCESS
        │
T=6s    └─> Saga Completed: {
                status: 'confirmed',
                order_id: 123,
                inventory_id: 456,
                payment_id: 789
            }
```

#### Saga実行フロー（失敗＆補償ケース）

```
┌──────────────────────────────────────────────────────────────┐
│         Saga Execution Flow (Failure & Compensation)          │
└──────────────────────────────────────────────────────────────┘

T=0s    Saga Started: create_order_saga(order_id=123)
        │
        ├─> Step 1: reserve_inventory(123)
        │   ├─> Inventory.reserve(order_id=123)
        │   ├─> inventory_id = 456
        │   └─> ✓ SUCCESS
        │
T=2s    ├─> Step 2: process_payment(prev_result, 123)
        │   ├─> Payment.charge(order_id=123)
        │   └─> ✗ FAILED: PaymentFailedError
        │       (クレジットカード拒否)
        │
        │   Compensation Triggered!
        │   │
T=3s    │   ├─> compensate_inventory(inventory_id=456)
        │   │   ├─> Inventory.release(456)
        │   │   └─> ✓ Inventory released
        │   │
        │   └─> Saga Failed & Compensated
        │
        └─> Result: {
                status: 'compensated',
                order_id: 123,
                error: 'PaymentFailedError',
                compensations: ['inventory_released']
            }

Final State:
  - Order: status='failed'
  - Inventory: released (available again)
  - Payment: not charged
  - User: notified of failure
```

### メリット

✅ **データ整合性**: 分散トランザクションを管理
✅ **自動ロールバック**: 失敗時に自動的に補償
✅ **ビジネスロジックの明確化**: 複雑なフローを構造化
✅ **監査証跡**: すべてのステップを記録

### デメリット

❌ **複雑性**: 補償ロジックの実装が必要
❌ **パフォーマンス**: 順次実行のため遅い
❌ **一貫性ウィンドウ**: 一時的に不整合な状態が存在
❌ **デバッグの困難**: 補償フローの追跡が難しい

### ユースケース

- **注文処理**: 在庫予約 → 支払い → 配送
- **予約システム**: 座席予約 → 支払い → チケット発行
- **マルチステップワークフロー**: 複数サービスにまたがる処理

### ブローカー選択: Redis または RabbitMQ

**Redis**: シンプルなSaga（3-5ステップ）
**RabbitMQ**: 複雑なSaga、高い信頼性が必要な場合

---

## パターン4: CQRS + イベントソーシング

### アーキテクチャ概要

コマンド（書き込み）とクエリ（読み込み）を分離し、イベントストアですべての変更を記録します。

```
┌─────────────────────────────────────────────────────────────────────┐
│              CQRS + Event Sourcing Architecture                      │
└─────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                      Command Side (Write)                     │
│                                                                │
│  ┌──────────────┐                                            │
│  │  API Gateway │                                            │
│  │  (Commands)  │                                            │
│  └──────┬───────┘                                            │
│         │ POST /commands/create-order                        │
│         ▼                                                     │
│  ┌──────────────────────────────────────────────────────┐   │
│  │         Command Handler Service                       │   │
│  │  ┌─────────────────────────────────────────────────┐ │   │
│  │  │ @app.task                                        │ │   │
│  │  │ def handle_create_order(command):                │ │   │
│  │  │   # 1. Validate command                          │ │   │
│  │  │   # 2. Execute business logic                    │ │   │
│  │  │   # 3. Generate events                           │ │   │
│  │  │   events = [                                     │ │   │
│  │  │     OrderCreatedEvent(...),                      │ │   │
│  │  │     InventoryReservedEvent(...)                  │ │   │
│  │  │   ]                                               │ │   │
│  │  │   # 4. Store events                              │ │   │
│  │  │   event_store.append(events)                     │ │   │
│  │  │   # 5. Publish events                            │ │   │
│  │  │   publish_events(events)                         │ │   │
│  │  └─────────────────────────────────────────────────┘ │   │
│  └──────────────────────┬───────────────────────────────┘   │
│                         │                                    │
│                         ▼                                    │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Event Store (Append-Only)                │   │
│  │  ┌────────────────────────────────────────────────┐  │   │
│  │  │ Event 1: OrderCreated(order_id=123, ...)      │  │   │
│  │  │ Event 2: InventoryReserved(order_id=123, ...) │  │   │
│  │  │ Event 3: PaymentProcessed(order_id=123, ...)  │  │   │
│  │  │ Event 4: OrderConfirmed(order_id=123, ...)    │  │   │
│  │  │ ...                                             │  │   │
│  │  └────────────────────────────────────────────────┘  │   │
│  └──────────────────────┬───────────────────────────────┘   │
└─────────────────────────┼────────────────────────────────────┘
                          │
                          │ Events Published
                          ▼
         ┌────────────────────────────────────────┐
         │   Message Broker (Event Bus)           │
         └────────────────┬───────────────────────┘
                          │
          ┌───────────────┼───────────────┐
          │               │               │
          ▼               ▼               ▼
┌──────────────────────────────────────────────────────────────┐
│                      Query Side (Read)                        │
│                                                                │
│  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────┐│
│  │ Read Model       │  │ Read Model       │  │ Read Model  ││
│  │ Updater 1        │  │ Updater 2        │  │ Updater 3   ││
│  │                  │  │                  │  │             ││
│  │ Subscribe to     │  │ Subscribe to     │  │ Subscribe to││
│  │ all events       │  │ order events     │  │ user events ││
│  └────────┬─────────┘  └────────┬─────────┘  └──────┬──────┘│
│           │                     │                    │        │
│           ▼                     ▼                    ▼        │
│  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────┐│
│  │ Read DB 1        │  │ Read DB 2        │  │ Read DB 3   ││
│  │ (PostgreSQL)     │  │ (Elasticsearch)  │  │ (Redis)     ││
│  │                  │  │                  │  │             ││
│  │ - Full data      │  │ - Search         │  │ - Cache     ││
│  │ - Complex queries│  │ - Analytics      │  │ - Fast reads││
│  └──────────────────┘  └──────────────────┘  └─────────────┘│
│           │                     │                    │        │
│           └─────────────────────┴────────────────────┘        │
│                                 │                              │
│                                 ▼                              │
│                      ┌──────────────────┐                     │
│                      │  API Gateway     │                     │
│                      │  (Queries)       │                     │
│                      │  GET /orders/123 │                     │
│                      └──────────────────┘                     │
└──────────────────────────────────────────────────────────────┘
```

### 実装例

#### Command Side

```python
# services/order_service/commands.py
from celery import Celery
from dataclasses import dataclass
from datetime import datetime
from typing import List

app = Celery('command_handler', broker='redis://localhost:6379/0')

# イベント定義
@dataclass
class Event:
    event_id: str
    event_type: str
    aggregate_id: str
    timestamp: datetime
    data: dict
    version: int

class OrderCreatedEvent(Event):
    def __init__(self, order_id, user_id, items):
        super().__init__(
            event_id=generate_uuid(),
            event_type='OrderCreated',
            aggregate_id=order_id,
            timestamp=datetime.now(),
            data={'user_id': user_id, 'items': items},
            version=1
        )

class PaymentProcessedEvent(Event):
    def __init__(self, order_id, payment_id, amount):
        super().__init__(
            event_id=generate_uuid(),
            event_type='PaymentProcessed',
            aggregate_id=order_id,
            timestamp=datetime.now(),
            data={'payment_id': payment_id, 'amount': amount},
            version=1
        )

# Event Store
class EventStore:
    """イベントストア（永続化）"""

    @staticmethod
    def append(events: List[Event]):
        """イベントを追加（Append-Only）"""
        for event in events:
            EventRecord.objects.create(
                event_id=event.event_id,
                event_type=event.event_type,
                aggregate_id=event.aggregate_id,
                timestamp=event.timestamp,
                data=event.data,
                version=event.version
            )

    @staticmethod
    def get_events(aggregate_id: str) -> List[Event]:
        """集約IDのすべてのイベントを取得"""
        records = EventRecord.objects.filter(
            aggregate_id=aggregate_id
        ).order_by('version')

        return [record.to_event() for record in records]

    @staticmethod
    def replay(aggregate_id: str):
        """イベントを再生して現在の状態を復元"""
        events = EventStore.get_events(aggregate_id)

        # 初期状態
        state = {'status': 'initial'}

        # イベントを順に適用
        for event in events:
            state = apply_event(state, event)

        return state

# コマンドハンドラ
@app.task(name='commands.create_order')
def handle_create_order(user_id, items):
    """注文作成コマンドを処理"""
    # 1. ビジネスロジックを実行
    order_id = generate_uuid()
    total = calculate_total(items)

    # 2. イベントを生成
    events = [
        OrderCreatedEvent(order_id, user_id, items),
    ]

    # 3. イベントストアに保存
    EventStore.append(events)

    # 4. イベントを発行（非同期）
    for event in events:
        publish_event.delay(event.__dict__)

    return {'order_id': order_id, 'status': 'created'}

@app.task(name='commands.process_payment')
def handle_process_payment(order_id, payment_method):
    """支払い処理コマンド"""
    # 1. 現在の状態を復元
    order_state = EventStore.replay(order_id)

    # 2. ビジネスルールを検証
    if order_state['status'] != 'created':
        raise InvalidStateError('Order must be in created state')

    # 3. 支払い処理
    payment_id = charge_payment(order_id, payment_method)

    # 4. イベントを生成
    events = [
        PaymentProcessedEvent(order_id, payment_id, order_state['total'])
    ]

    # 5. イベントストアに保存
    EventStore.append(events)

    # 6. イベントを発行
    for event in events:
        publish_event.delay(event.__dict__)

    return {'order_id': order_id, 'payment_id': payment_id}

@app.task(name='events.publish')
def publish_event(event_data):
    """イベントをメッセージブローカーに発行"""
    app.send_task(
        'events.consume',
        args=[event_data],
        routing_key=f"events.{event_data['event_type']}",
        exchange='events',
        exchange_type='topic'
    )
```

#### Query Side (Read Model Updater)

```python
# services/order_query/read_model_updater.py
from celery import Celery
from kombu import Queue, Exchange

app = Celery('read_model_updater', broker='redis://localhost:6379/0')

events_exchange = Exchange('events', type='topic', durable=True)

app.conf.task_queues = (
    Queue(
        'order_read_model_queue',
        exchange=events_exchange,
        routing_key='events.Order*',
        durable=True
    ),
)

# Read Model (クエリ最適化)
class OrderReadModel:
    """注文の読み込み専用モデル"""

    @staticmethod
    def create(event_data):
        """OrderCreatedイベントから作成"""
        OrderView.objects.create(
            order_id=event_data['aggregate_id'],
            user_id=event_data['data']['user_id'],
            items=event_data['data']['items'],
            status='created',
            created_at=event_data['timestamp']
        )

    @staticmethod
    def update_payment(event_data):
        """PaymentProcessedイベントで更新"""
        order = OrderView.objects.get(
            order_id=event_data['aggregate_id']
        )
        order.payment_id = event_data['data']['payment_id']
        order.amount = event_data['data']['amount']
        order.status = 'payment_processed'
        order.save()

@app.task(name='read_model.handle_order_created')
def handle_order_created(event_data):
    """OrderCreatedイベントのハンドラ"""
    OrderReadModel.create(event_data)

@app.task(name='read_model.handle_payment_processed')
def handle_payment_processed(event_data):
    """PaymentProcessedイベントのハンドラ"""
    OrderReadModel.update_payment(event_data)

# イベントルーター
@app.task(name='events.consume')
def consume_event(event_data):
    """イベントを消費して適切なハンドラにルーティング"""
    event_type = event_data['event_type']

    handlers = {
        'OrderCreated': handle_order_created,
        'PaymentProcessed': handle_payment_processed,
    }

    handler = handlers.get(event_type)
    if handler:
        handler.delay(event_data)
```

#### Query API

```python
# services/order_query/api.py
from flask import Flask, jsonify

app = Flask(__name__)

@app.route('/api/orders/<order_id>')
def get_order(order_id):
    """注文を取得（Read Modelから）"""
    order = OrderView.objects.get(order_id=order_id)
    return jsonify({
        'order_id': order.order_id,
        'user_id': order.user_id,
        'items': order.items,
        'status': order.status,
        'amount': order.amount,
        'created_at': order.created_at.isoformat()
    })

@app.route('/api/orders/search')
def search_orders():
    """注文を検索（Elasticsearchから）"""
    query = request.args.get('q')
    results = es.search(index='orders', body={
        'query': {
            'multi_match': {
                'query': query,
                'fields': ['order_id', 'user_id', 'items']
            }
        }
    })
    return jsonify(results)
```

### 技術的原理

#### イベントソーシング

```
┌──────────────────────────────────────────────────────────────┐
│              Event Sourcing: State Reconstruction             │
└──────────────────────────────────────────────────────────────┘

Event Store (order_id=123):
┌────────────────────────────────────────────────────────────┐
│ Version 1: OrderCreated                                     │
│   {order_id: 123, user_id: 456, items: [...]}             │
├────────────────────────────────────────────────────────────┤
│ Version 2: InventoryReserved                               │
│   {order_id: 123, inventory_ids: [789, 790]}              │
├────────────────────────────────────────────────────────────┤
│ Version 3: PaymentProcessed                                │
│   {order_id: 123, payment_id: 111, amount: 10000}         │
├────────────────────────────────────────────────────────────┤
│ Version 4: OrderConfirmed                                  │
│   {order_id: 123, confirmed_at: "2024-01-01T12:00:00"}    │
└────────────────────────────────────────────────────────────┘

State Reconstruction (Replay):
Initial State: {}
  │
  ├─> Apply Event 1 (OrderCreated)
  │   State: {order_id: 123, status: 'created', user_id: 456, ...}
  │
  ├─> Apply Event 2 (InventoryReserved)
  │   State: {..., inventory_reserved: true, inventory_ids: [789, 790]}
  │
  ├─> Apply Event 3 (PaymentProcessed)
  │   State: {..., payment_status: 'processed', payment_id: 111}
  │
  └─> Apply Event 4 (OrderConfirmed)
      Final State: {..., status: 'confirmed', confirmed_at: "..."}

Advantages:
- Complete audit trail (すべての変更履歴)
- Time travel (任意の時点の状態を復元)
- Event replay (障害復旧、デバッグ)
```

### メリット

✅ **完全な監査証跡**: すべての変更が記録される
✅ **タイムトラベル**: 過去の任意の時点の状態を復元
✅ **スケーラビリティ**: 読み書きを独立してスケール
✅ **柔軟なクエリ**: 複数の最適化されたRead Modelを構築
✅ **イベント駆動**: リアクティブなシステム

### デメリット

❌ **複雑性**: アーキテクチャが複雑
❌ **最終的整合性**: Read Modelの更新に遅延
❌ **イベントスキーマ**: イベント構造の変更が困難
❌ **ストレージ**: イベントが無限に増加

### ユースケース

- **金融システム**: すべての取引を記録
- **監査が必要なシステム**: コンプライアンス
- **複雑なドメインロジック**: ビジネスルールが多い
- **時系列分析**: ユーザー行動分析

### ブローカー選択: RabbitMQ

**推奨理由:**
- イベントの順序保証
- 永続性
- 複雑なルーティング

---

## ブローカー選択ガイド

### Redis を選ぶべき場合

#### 特性
- **速度**: 非常に高速（メモリベース）
- **シンプル**: セットアップが簡単
- **軽量**: リソース使用量が少ない

#### 適用パターン
✅ **パターン1: タスクベース・マイクロサービス**
  - 条件: サービス数が少ない（3-5個）、高速性重視

✅ **パターン3: サーガパターン**
  - 条件: シンプルなSaga（3-5ステップ）

#### 推奨構成
```python
# Redis Configuration for Microservices
CELERY_BROKER_URL = 'redis://localhost:6379/0'
CELERY_RESULT_BACKEND = 'redis://localhost:6379/1'

CELERY_BROKER_CONNECTION_RETRY = True
CELERY_BROKER_CONNECTION_MAX_RETRIES = 10

# タスクの並列処理を最大化
CELERYD_PREFETCH_MULTIPLIER = 4
CELERYD_CONCURRENCY = 8  # CPU cores

# メモリ管理
CELERYD_MAX_TASKS_PER_CHILD = 1000
```

#### 制限事項
❌ 優先度キューのサポートが限定的
❌ 複雑なルーティングができない
❌ メモリベースなのでデータ永続性に限界

---

### RabbitMQ を選ぶべき場合

#### 特性
- **信頼性**: 高い永続性
- **高機能**: Exchange、優先度キュー
- **スケーラビリティ**: クラスター構成

#### 適用パターン
✅ **パターン2: イベント駆動アーキテクチャ**
  - 理由: Topic Exchangeが必須

✅ **パターン4: CQRS + イベントソーシング**
  - 理由: イベントの順序保証、永続性

✅ **本番環境のすべてのパターン**
  - 理由: 高い信頼性が必要

#### 推奨構成
```python
# RabbitMQ Configuration for Microservices
CELERY_BROKER_URL = 'amqp://user:pass@localhost:5672//'
CELERY_RESULT_BACKEND = 'rpc://'  # または別のバックエンド

# 信頼性設定
CELERY_TASK_ACKS_LATE = True
CELERY_TASK_REJECT_ON_WORKER_LOST = True

# 優先度キュー
CELERY_TASK_QUEUE_MAX_PRIORITY = 10

# プリフェッチ（長時間タスク向け）
CELERYD_PREFETCH_MULTIPLIER = 1

# Exchange設定
from kombu import Exchange, Queue

CELERY_TASK_QUEUES = (
    Queue('default', Exchange('default'), routing_key='default'),
    Queue('high', Exchange('high'), routing_key='high', queue_arguments={
        'x-max-priority': 10,
    }),
)

# Topic Exchange for Events
CELERY_TASK_ROUTES = {
    'events.*': {
        'exchange': 'events',
        'exchange_type': 'topic',
    }
}
```

---

## まとめ: パターン選択マトリクス

| パターン | 複雑度 | スケーラビリティ | データ整合性 | 推奨ブローカー | ユースケース |
|---------|--------|----------------|-------------|--------------|------------|
| **タスクベース** | 低 | 中 | 最終的整合性 | Redis/RabbitMQ | 小規模、シンプル |
| **イベント駆動** | 中 | 高 | 最終的整合性 | RabbitMQ | 通知、分析 |
| **サーガ** | 高 | 中 | 強整合性 | RabbitMQ | 複雑なトランザクション |
| **CQRS+ES** | 最高 | 最高 | 最終的整合性 | RabbitMQ | 金融、監査 |

### 選択ガイド

1. **プロジェクトの規模**
   - 小規模（1-3サービス）→ パターン1 + Redis
   - 中規模（4-10サービス）→ パターン2 + RabbitMQ
   - 大規模（10+サービス）→ パターン4 + RabbitMQ

2. **データ整合性要件**
   - 緩い → パターン1, 2
   - 厳しい → パターン3, 4

3. **監査要件**
   - 不要 → パターン1, 2
   - 必要 → パターン4

4. **チームのスキル**
   - 初心者 → パターン1
   - 経験者 → パターン2, 3, 4
