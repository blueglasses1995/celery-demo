# プロジェクト4: Webスクレイピングシステム

## プロジェクト概要

複数サイトを並列クローリングし、レート制限とリトライを実装した分散スクレイピングシステムを構築します。

## 実装

### タスク定義

```python
# tasks/scraping_tasks.py
from celery import group
from celery_app import app
import requests
from bs4 import BeautifulSoup

@app.task(
    bind=True,
    autoretry_for=(requests.RequestException,),
    retry_kwargs={'max_retries': 5},
    retry_backoff=True,
    rate_limit='10/m'  # レート制限
)
def scrape_page(self, url):
    """Webページをスクレイピング"""
    try:
        response = requests.get(url, timeout=30)
        response.raise_for_status()

        soup = BeautifulSoup(response.content, 'html.parser')

        # データ抽出
        data = {
            'url': url,
            'title': soup.find('title').text,
            'links': [a['href'] for a in soup.find_all('a', href=True)],
            'scraped_at': timezone.now()
        }

        # データベースに保存
        ScrapedData.objects.create(**data)

        return data

    except Exception as exc:
        logger.error(f'Failed to scrape {url}: {exc}')
        raise

@app.task
def scrape_multiple_urls(urls):
    """複数URLを並列スクレイピング"""
    job = group(scrape_page.s(url) for url in urls)
    result = job.apply_async()
    return result

@app.task
def scheduled_scraping():
    """定期的なスクレイピング（Celery Beatで実行）"""
    urls = get_urls_to_scrape()
    scrape_multiple_urls.delay(urls)
```

### スケジュール設定

```python
# celery_app.py
from celery.schedules import crontab

app.conf.beat_schedule = {
    'scrape-daily': {
        'task': 'tasks.scraping_tasks.scheduled_scraping',
        'schedule': crontab(hour=2, minute=0),
    },
}
```

## まとめ

- ✅ 分散クローリング
- ✅ レート制限とリトライ
- ✅ 定期スクレイピング

## カリキュラム完了

おめでとうございます！Celeryの学習カリキュラムをすべて完了しました。

学んだスキル:
- ✅ Celeryの基礎概念
- ✅ タスクの実行と状態管理
- ✅ ワーカーとブローカーの理解
- ✅ 高度なワークフローとエラーハンドリング
- ✅ 本番環境でのデプロイと運用
- ✅ 実践的なプロジェクト構築

これで、Celeryを使った非同期タスク処理システムを設計・実装・運用できるようになりました！
