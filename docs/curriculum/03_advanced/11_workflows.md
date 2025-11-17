# 11. タスクチェーンとワークフロー

## 学習目標

- chain、group、chordを使ってタスクを組み合わせる
- 複雑なワークフローを構築する

## ワークフローの基本パターン

### 1. chain - 順次実行

タスクを順番に実行し、前のタスクの結果を次に渡します。

```python
from celery import chain

@app.task
def add(x, y):
    return x + y

@app.task
def multiply(x, y):
    return x * y

# chain: add(2, 2) → multiply(result, 8)
workflow = chain(add.s(2, 2), multiply.s(8))
result = workflow.apply_async()
print(result.get())  # (2 + 2) * 8 = 32
```

パイプ記法:

```python
workflow = (add.s(2, 2) | multiply.s(8))
result = workflow()
print(result.get())  # 32
```

### 2. group - 並列実行

複数のタスクを並列実行します。

```python
from celery import group

# 複数のタスクを並列実行
job = group(add.s(i, i) for i in range(10))
result = job.apply_async()
print(result.get())  # [0, 2, 4, 6, 8, 10, 12, 14, 16, 18]
```

### 3. chord - 並列実行後にコールバック

並列タスクの結果をまとめて、コールバックタスクを実行します。

```python
from celery import chord

@app.task
def sum_all(numbers):
    return sum(numbers)

# chord: 並列実行 → 結果を集約
workflow = chord(add.s(i, i) for i in range(10))(sum_all.s())
result = workflow.get()
print(result)  # sum([0, 2, 4, 6, 8, 10, 12, 14, 16, 18]) = 90
```

### 4. map - リストに対してタスクを実行

```python
from celery import group

@app.task
def process_item(item):
    return item * 2

# map
items = [1, 2, 3, 4, 5]
job = group(process_item.s(item) for item in items)
result = job.apply_async()
print(result.get())  # [2, 4, 6, 8, 10]
```

### 5. starmap - タプルを展開して実行

```python
@app.task
def add(x, y):
    return x + y

# starmap
pairs = [(1, 1), (2, 2), (3, 3)]
job = group(add.s(x, y) for x, y in pairs)
result = job.apply_async()
print(result.get())  # [2, 4, 6]
```

## 実践例

### 例1: 画像処理パイプライン

```python
@app.task
def download_image(url):
    image_data = requests.get(url).content
    return image_data

@app.task
def resize_image(image_data, size):
    resized = resize(image_data, size)
    return resized

@app.task
def upload_image(image_data):
    url = upload_to_s3(image_data)
    return url

# パイプライン
workflow = chain(
    download_image.s('https://example.com/image.jpg'),
    resize_image.s((800, 600)),
    upload_image.s()
)
result = workflow.apply_async()
print(result.get())  # アップロードされたURL
```

### 例2: データ集計

```python
@app.task
def fetch_sales_data(region):
    return get_sales_for_region(region)

@app.task
def aggregate_sales(sales_list):
    return {
        'total': sum(sales_list),
        'average': sum(sales_list) / len(sales_list),
        'regions': len(sales_list)
    }

# 複数リージョンのデータを並列取得 → 集計
regions = ['north', 'south', 'east', 'west']
workflow = chord(
    fetch_sales_data.s(region) for region in regions
)(aggregate_sales.s())

result = workflow.get()
print(result)  # {'total': 10000, 'average': 2500, 'regions': 4}
```

### 例3: ETLパイプライン

```python
@app.task
def extract_data(source):
    return fetch_from_source(source)

@app.task
def transform_data(data):
    return clean_and_transform(data)

@app.task
def load_data(data):
    save_to_database(data)
    return 'Success'

# ETLパイプライン
workflow = chain(
    extract_data.s('database'),
    transform_data.s(),
    load_data.s()
)
result = workflow.apply_async()
```

## まとめ

- ✅ `chain`: 順次実行
- ✅ `group`: 並列実行
- ✅ `chord`: 並列実行 + コールバック
- ✅ ワークフローで複雑な処理を構築

## 次のステップ

次のセクション「[エラーハンドリング](./12_error_handling.md)」では、エラー処理を学びます。
