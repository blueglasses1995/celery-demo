# 10. タスクのスケジューリング

## 学習目標

- Celery Beatを使って定期的なタスクを実行する
- crontab形式のスケジュールを理解する
- 動的なスケジュール管理を習得する

## Celery Beat とは

Celery Beatは、定期的にタスクをスケジュールするためのスケジューラーです。

```
Celery Beat → [定期的にタスクを送信] → Broker → Worker
```

## 基本的な使い方

### スケジュールの定義

```python
# celery_app.py
from celery import Celery
from celery.schedules import crontab

app = Celery('myapp')

app.conf.beat_schedule = {
    # 30秒ごとに実行
    'add-every-30-seconds': {
        'task': 'tasks.add',
        'schedule': 30.0,
        'args': (16, 16)
    },

    # 毎日午前2時に実行
    'cleanup-daily': {
        'task': 'tasks.cleanup',
        'schedule': crontab(hour=2, minute=0),
    },

    # 毎週月曜日の午前9時
    'weekly-report': {
        'task': 'tasks.generate_report',
        'schedule': crontab(hour=9, minute=0, day_of_week=1),
        'args': ('weekly',),
    },
}
```

### Beatの起動

```bash
# 別のターミナルでBeatを起動
celery -A celery_app beat --loglevel=info

# ワーカーも必要
celery -A celery_app worker --loglevel=info
```

## スケジュールの種類

### 1. 秒単位のスケジュール

```python
# 10秒ごと
'task-name': {
    'task': 'tasks.my_task',
    'schedule': 10.0,
}

# 1分ごと
'task-name': {
    'task': 'tasks.my_task',
    'schedule': 60.0,
}
```

### 2. timedelta

```python
from datetime import timedelta

'task-name': {
    'task': 'tasks.my_task',
    'schedule': timedelta(minutes=30),
}
```

### 3. crontab

```python
from celery.schedules import crontab

# 毎時0分に実行
'hourly-task': {
    'task': 'tasks.my_task',
    'schedule': crontab(minute=0),
}

# 毎日午前2時
'daily-task': {
    'task': 'tasks.my_task',
    'schedule': crontab(hour=2, minute=0),
}

# 毎週月曜日
'weekly-task': {
    'task': 'tasks.my_task',
    'schedule': crontab(hour=0, minute=0, day_of_week=1),
}

# 毎月1日
'monthly-task': {
    'task': 'tasks.my_task',
    'schedule': crontab(hour=0, minute=0, day_of_month=1),
}

# 複数の時刻（0時、6時、12時、18時）
'four-times-daily': {
    'task': 'tasks.my_task',
    'schedule': crontab(hour='0,6,12,18', minute=0),
}

# 範囲指定（平日9-17時の毎時）
'business-hours': {
    'task': 'tasks.my_task',
    'schedule': crontab(hour='9-17', minute=0, day_of_week='mon-fri'),
}
```

### crontabのパラメータ

```python
crontab(
    minute='*',         # 0-59
    hour='*',           # 0-23
    day_of_week='*',    # 0-6 (0=日曜日) または mon,tue,wed,thu,fri,sat,sun
    day_of_month='*',   # 1-31
    month_of_year='*',  # 1-12
)
```

## 実践例

### 例1: データベースのバックアップ

```python
# tasks.py
@app.task
def backup_database():
    """データベースをバックアップ"""
    timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
    filename = f'backup_{timestamp}.sql'

    # バックアップ処理
    create_backup(filename)
    upload_to_s3(filename)

    return f'Backup created: {filename}'

# celery_app.py
app.conf.beat_schedule = {
    'backup-daily': {
        'task': 'tasks.backup_database',
        'schedule': crontab(hour=2, minute=0),  # 毎日午前2時
    },
}
```

### 例2: レポート生成

```python
# tasks.py
@app.task
def generate_daily_report():
    """日次レポート生成"""
    yesterday = date.today() - timedelta(days=1)
    data = fetch_data_for_date(yesterday)
    report = create_report(data)
    send_to_managers(report)
    return f'Report generated for {yesterday}'

@app.task
def generate_weekly_report():
    """週次レポート生成"""
    # 処理...
    pass

# celery_app.py
app.conf.beat_schedule = {
    'daily-report': {
        'task': 'tasks.generate_daily_report',
        'schedule': crontab(hour=8, minute=0),  # 毎日午前8時
    },
    'weekly-report': {
        'task': 'tasks.generate_weekly_report',
        'schedule': crontab(hour=9, minute=0, day_of_week=1),  # 毎週月曜午前9時
    },
}
```

### 例3: キャッシュのクリア

```python
@app.task
def clear_expired_cache():
    """期限切れキャッシュをクリア"""
    cleared = cache.delete_expired()
    return f'Cleared {cleared} expired items'

app.conf.beat_schedule = {
    'clear-cache-hourly': {
        'task': 'tasks.clear_expired_cache',
        'schedule': crontab(minute=0),  # 毎時0分
    },
}
```

### 例4: ヘルスチェック

```python
@app.task
def health_check():
    """システムのヘルスチェック"""
    checks = {
        'database': check_database(),
        'redis': check_redis(),
        'api': check_external_api(),
    }

    for service, status in checks.items():
        if not status:
            send_alert(f'{service} is down!')

    return checks

app.conf.beat_schedule = {
    'health-check': {
        'task': 'tasks.health_check',
        'schedule': 300.0,  # 5分ごと
    },
}
```

## 動的スケジュール管理

### DatabaseSchedulerを使用

```bash
pip install django-celery-beat
```

```python
# settings.py (Django)
INSTALLED_APPS = [
    ...
    'django_celery_beat',
]

# celery_app.py
app.conf.beat_scheduler = 'django_celery_beat.schedulers:DatabaseScheduler'
```

これにより、管理画面からスケジュールを動的に追加・変更できます。

### プログラムでスケジュールを追加

```python
from django_celery_beat.models import PeriodicTask, IntervalSchedule
from datetime import timedelta

# インターバルスケジュールを作成
schedule, created = IntervalSchedule.objects.get_or_create(
    every=10,
    period=IntervalSchedule.SECONDS,
)

# タスクを追加
PeriodicTask.objects.create(
    interval=schedule,
    name='Import data every 10 seconds',
    task='tasks.import_data',
)
```

## Beatの運用

### 単一インスタンスで実行

**重要**: Beatは必ず1つのインスタンスのみ実行してください。複数起動するとタスクが重複実行されます。

```bash
# ワーカーとBeatを別々に起動（推奨）
celery -A celery_app worker --loglevel=info
celery -A celery_app beat --loglevel=info
```

### スケジュールファイル

```bash
# スケジュールファイルの場所を指定
celery -A celery_app beat \
    --loglevel=info \
    --schedule=/var/run/celery/celerybeat-schedule
```

### Beatとワーカーを同時起動（開発用のみ）

```bash
# 開発環境でのみ使用
celery -A celery_app worker --beat --loglevel=info
```

**注意**: 本番環境では推奨されません。

## まとめ

- ✅ Celery Beatで定期タスクを実行
- ✅ `crontab`で柔軟なスケジューリング
- ✅ DatabaseSchedulerで動的管理
- ✅ Beatは単一インスタンスのみ実行

## 次のステップ

次のセクション「[タスクチェーンとワークフロー](./11_workflows.md)」では、複数タスクの連携を学びます。
