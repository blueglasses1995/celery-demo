# プロジェクト1: メール送信システム

## プロジェクト概要

バルクメール送信システムを構築し、リトライ、レート制限、レポート生成を実装します。

## 学習目標

- バルクメール送信の実装
- レート制限の適用
- エラーハンドリングとリトライ
- 送信レポートの生成

## アーキテクチャ

```
[Webアプリ] → [send_bulk_emails] → [send_email (個別)] → [メールサーバー]
                                           ↓
                                    [送信履歴の記録]
                                           ↓
                                    [generate_report]
```

## 実装

### 1. タスク定義

```python
# tasks/email_tasks.py
from celery import group, chord
from celery_app import app
import logging

logger = logging.getLogger(__name__)

@app.task(
    bind=True,
    autoretry_for=(SMTPException,),
    retry_kwargs={'max_retries': 3},
    retry_backoff=True,
    rate_limit='10/m'  # 1分間に10通
)
def send_email(self, recipient, subject, body, template=None):
    """個別のメール送信"""
    try:
        if template:
            body = render_template(template, context={'body': body})

        send_mail(
            subject=subject,
            message=body,
            from_email='noreply@example.com',
            recipient_list=[recipient],
        )

        # 送信履歴を記録
        EmailLog.objects.create(
            task_id=self.request.id,
            recipient=recipient,
            subject=subject,
            status='sent',
            sent_at=timezone.now()
        )

        return {'status': 'sent', 'recipient': recipient}

    except Exception as exc:
        # エラーログ
        EmailLog.objects.create(
            task_id=self.request.id,
            recipient=recipient,
            subject=subject,
            status='failed',
            error=str(exc)
        )
        raise

@app.task
def generate_email_report(results):
    """送信レポートを生成"""
    total = len(results)
    sent = sum(1 for r in results if r.get('status') == 'sent')
    failed = total - sent

    report = {
        'total': total,
        'sent': sent,
        'failed': failed,
        'success_rate': (sent / total * 100) if total > 0 else 0
    }

    # レポートを保存またはメール送信
    save_report(report)
    notify_admin(report)

    return report

@app.task
def send_bulk_emails(recipients, subject, body, template=None):
    """バルクメール送信（コールバック付き）"""
    # chordで並列送信 → レポート生成
    job = chord(
        send_email.s(recipient, subject, body, template)
        for recipient in recipients
    )(generate_email_report.s())

    return job
```

### 2. モデル定義

```python
# models.py
from django.db import models

class EmailLog(models.Model):
    task_id = models.CharField(max_length=255, db_index=True)
    recipient = models.EmailField()
    subject = models.CharField(max_length=255)
    status = models.CharField(
        max_length=20,
        choices=[('sent', 'Sent'), ('failed', 'Failed')]
    )
    sent_at = models.DateTimeField(null=True, blank=True)
    error = models.TextField(null=True, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        indexes = [
            models.Index(fields=['recipient', 'created_at']),
            models.Index(fields=['status', 'created_at']),
        ]
```

### 3. 使用例

```python
# views.py
from django.http import JsonResponse
from tasks.email_tasks import send_bulk_emails

def send_newsletter(request):
    """ニュースレター送信"""
    # 購読者リストを取得
    subscribers = Subscriber.objects.filter(
        is_active=True
    ).values_list('email', flat=True)

    subject = 'Monthly Newsletter'
    body = get_newsletter_content()

    # バルクメール送信
    result = send_bulk_emails.delay(
        recipients=list(subscribers),
        subject=subject,
        body=body,
        template='newsletter.html'
    )

    return JsonResponse({
        'status': 'queued',
        'task_id': result.id,
        'recipients': len(subscribers)
    })
```

## 進捗トラッキング

```python
@app.task(bind=True)
def send_bulk_emails_with_progress(self, recipients, subject, body):
    """進捗付きバルクメール送信"""
    total = len(recipients)
    sent = 0
    failed = 0

    for i, recipient in enumerate(recipients):
        try:
            send_email.apply(args=[recipient, subject, body])
            sent += 1
        except Exception:
            failed += 1

        # 進捗を更新
        if i % 10 == 0:
            self.update_state(
                state='PROGRESS',
                meta={
                    'current': i + 1,
                    'total': total,
                    'sent': sent,
                    'failed': failed,
                    'percent': int((i + 1) / total * 100)
                }
            )

    return {
        'total': total,
        'sent': sent,
        'failed': failed
    }
```

## まとめ

このプロジェクトで学んだこと:
- ✅ バルクメール送信の実装
- ✅ レート制限の適用
- ✅ エラーハンドリング
- ✅ chordでコールバック
- ✅ 送信レポートの生成

## 次のプロジェクト

次のプロジェクト「[画像処理パイプライン](./22_image_pipeline.md)」では、画像処理のワークフローを学びます。
