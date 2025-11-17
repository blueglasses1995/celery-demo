# タスク実行の詳細フロー

## 概要

タスクがどのように実行されるのか、ブローカーからワーカーでの処理、結果の保存まで、詳細なフローを解説します。

## 完全な実行フロー

```
┌──────────────────────────────────────────────────────────────────┐
│                    Complete Task Execution Flow                   │
└──────────────────────────────────────────────────────────────────┘

Phase 1: Task Submission (Producer)
═══════════════════════════════════════════════════════════════════

┌─────────────┐
│ Application │  task.delay(4, 6)
└──────┬──────┘
       │
       │ 1. Generate task_id (UUID4)
       │    "550e8400-e29b-41d4-a716-446655440000"
       │
       │ 2. Build task message
       ▼
┌─────────────────────────────────────────────────────────────────┐
│ Task Message Structure:                                          │
│ {                                                                 │
│   "task": "myapp.tasks.add",                                     │
│   "id": "550e8400-...",                                          │
│   "args": [4, 6],                                                │
│   "kwargs": {},                                                  │
│   "retries": 0,                                                  │
│   "eta": null,              # 実行時刻指定                       │
│   "expires": null,          # 有効期限                           │
│   "callbacks": null,        # 成功時コールバック                 │
│   "errbacks": null,         # 失敗時コールバック                 │
│   "chord": null,            # chordグループID                    │
│   "group": null,            # グループID                         │
│   "utc": true,                                                   │
│   "origin": "gen1@hostname",  # 送信元                           │
│   "root_id": "550e8400-...",  # ルートタスクID (chain用)         │
│   "parent_id": null,          # 親タスクID                       │
│   "argsrepr": "(4, 6)",       # 引数の文字列表現（ログ用）       │
│   "kwargsrepr": "{}",                                            │
│   "timelimit": [null, null],  # [soft_limit, hard_limit]        │
│   "taskset": null,                                               │
│   "lang": "py",                                                  │
│   "delivery_info": {                                             │
│     "exchange": "celery",                                        │
│     "routing_key": "celery"                                      │
│   }                                                              │
│ }                                                                 │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            │ 3. Serialize (JSON/msgpack)
                            │ 4. Compress (optional)
                            │ 5. Send to broker
                            ▼
                    ┌───────────────┐
                    │     Broker    │
                    │  (Redis/AMQP) │
                    └───────┬───────┘
                            │
                            │ Message queued
                            │


Phase 2: Task Retrieval (Worker - Consumer Thread)
═══════════════════════════════════════════════════════════════════

                    ┌───────────────┐
                    │     Broker    │
                    └───────┬───────┘
                            │
                            │ Worker polls for messages
                            ▼
┌────────────────────────────────────────────────────────────────┐
│                    Celery Worker Process                        │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │           Consumer Thread (MainProcess)                   │ │
│  │                                                            │ │
│  │  1. BLPOP/basic.consume - Blocking wait for message      │ │
│  │     (with timeout to allow for graceful shutdown)         │ │
│  │                                                            │ │
│  │  2. Message Received                                      │ │
│  │     ┌──────────────────────────────────────────────────┐ │ │
│  │     │ Raw message from broker                          │ │ │
│  │     │ Headers: content-type, delivery_tag, etc         │ │ │
│  │     │ Body: compressed/serialized task data            │ │ │
│  │     └──────────────────────────────────────────────────┘ │ │
│  │                                                            │ │
│  │  3. Decompress (if compressed)                            │ │
│  │     gzip.decompress(body) if 'gzip' in headers           │ │
│  │                                                            │ │
│  │  4. Deserialize                                           │ │
│  │     message = json.loads(decompressed_body)              │ │
│  │                                                            │ │
│  │  5. Validate Message                                      │ │
│  │     - task name exists?                                   │ │
│  │     - required fields present?                            │ │
│  │     - message signature valid? (if enabled)               │ │
│  │                                                            │ │
│  │  6. Check ETA (if specified)                              │ │
│  │     if message['eta']:                                    │ │
│  │       eta_time = parse_iso8601(message['eta'])           │ │
│  │       if eta_time > now():                                │ │
│  │         # Re-queue with countdown                         │ │
│  │         delay = (eta_time - now()).total_seconds()       │ │
│  │         return  # Process later                           │ │
│  │                                                            │ │
│  │  7. Check Expires                                         │ │
│  │     if message['expires']:                                │ │
│  │       expires_time = parse_iso8601(message['expires'])   │ │
│  │       if expires_time < now():                            │ │
│  │         # Task expired, reject                            │ │
│  │         logger.warning('Task expired')                    │ │
│  │         ack_message()                                     │ │
│  │         return                                            │ │
│  │                                                            │ │
│  │  8. Build Task Request                                    │ │
│  │     request = Request(                                    │ │
│  │       id=message['id'],                                   │ │
│  │       task=message['task'],                               │ │
│  │       args=message['args'],                               │ │
│  │       kwargs=message['kwargs'],                           │ │
│  │       delivery_info=message['delivery_info'],             │ │
│  │       ...                                                  │ │
│  │     )                                                      │ │
│  │                                                            │ │
│  │  9. Update State: PENDING → STARTED (if track_started)   │ │
│  │     if app.conf.task_track_started:                       │ │
│  │       backend.store_result(                               │ │
│  │         task_id=request.id,                               │ │
│  │         result=None,                                      │ │
│  │         status='STARTED'                                  │ │
│  │       )                                                    │ │
│  │                                                            │ │
│  │  10. Dispatch to Worker Pool                              │ │
│  │      pool.apply_async(                                    │ │
│  │        execute_task,                                      │ │
│  │        args=(request,),                                   │ │
│  │        callback=on_success,                               │ │
│  │        error_callback=on_failure                          │ │
│  │      )                                                     │ │
│  └──────────────────┬───────────────────────────────────────┘ │
│                     │                                          │
│                     │ Task dispatched to pool                  │
│                     ▼                                          │
└────────────────────────────────────────────────────────────────┘


Phase 3: Task Execution (Worker Pool Process)
═══════════════════════════════════════════════════════════════════

┌────────────────────────────────────────────────────────────────┐
│              Worker Pool (Separate Process)                     │
│                                                                  │
│  Process: ForkPoolWorker-1                                      │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │                                                            │ │
│  │  1. Receive Task Request from main process                │ │
│  │     (via multiprocessing.Queue or pipe)                   │ │
│  │                                                            │ │
│  │  2. Set up execution context                              │ │
│  │     - Install signal handlers (SIGTERM, SIGUSR1)          │ │
│  │     - Set time limits (soft/hard)                         │ │
│  │     - Prepare task instance                               │ │
│  │                                                            │ │
│  │  3. Time Limit Setup                                      │ │
│  │     ┌────────────────────────────────────────────────┐   │ │
│  │     │ if soft_time_limit:                            │   │ │
│  │     │   signal.signal(SIGUSR1, _soft_timeout)        │   │ │
│  │     │   signal.alarm(soft_time_limit)                │   │ │
│  │     │                                                  │   │ │
│  │     │ if hard_time_limit:                            │   │ │
│  │     │   # OS timer for hard limit                    │   │ │
│  │     │   threading.Timer(                              │   │ │
│  │     │     hard_time_limit,                           │   │ │
│  │     │     os.kill, args=(os.getpid(), SIGTERM)      │   │ │
│  │     │   ).start()                                     │   │ │
│  │     └────────────────────────────────────────────────┘   │ │
│  │                                                            │ │
│  │  4. Import and Retrieve Task Function                     │ │
│  │     task_cls = app.tasks[request.task]                   │ │
│  │     # "myapp.tasks.add" → actual function object         │ │
│  │                                                            │ │
│  │  5. Bind Task Instance (if bind=True)                     │ │
│  │     if task_cls.bind:                                     │ │
│  │       task_instance = task_cls()                          │ │
│  │       task_instance.request = request                     │ │
│  │       # self.request.id, self.request.retries, etc       │ │
│  │                                                            │ │
│  │  6. Pre-task Execution Hooks                              │ │
│  │     task_cls.on_before_start(task_id, args, kwargs)      │ │
│  │                                                            │ │
│  │  7. Execute Task Function                                 │ │
│  │     ┌────────────────────────────────────────────────┐   │ │
│  │     │ try:                                           │   │ │
│  │     │   if task_cls.bind:                           │   │ │
│  │     │     result = task_instance.run(              │   │ │
│  │     │       *request.args,                          │   │ │
│  │     │       **request.kwargs                        │   │ │
│  │     │     )                                          │   │ │
│  │     │   else:                                        │   │ │
│  │     │     result = task_cls(                        │   │ │
│  │     │       *request.args,                          │   │ │
│  │     │       **request.kwargs                        │   │ │
│  │     │     )                                          │   │ │
│  │     │                                                 │   │ │
│  │     │   # Task function executes here               │   │ │
│  │     │   # e.g., return 4 + 6 = 10                   │   │ │
│  │     │                                                 │   │ │
│  │     │ except SoftTimeLimitExceeded:                 │   │ │
│  │     │   # Soft timeout, allow cleanup               │   │ │
│  │     │   task_cls.on_soft_timeout()                  │   │ │
│  │     │   raise                                        │   │ │
│  │     │                                                 │   │ │
│  │     │ except Exception as exc:                      │   │ │
│  │     │   # Task raised an exception                  │   │ │
│  │     │   if task_cls.autoretry_for:                  │   │ │
│  │     │     if isinstance(exc, task_cls.autoretry_for):│   │ │
│  │     │       # Automatic retry                       │   │ │
│  │     │       task_instance.retry(exc=exc)            │   │ │
│  │     │   raise                                        │   │ │
│  │     │                                                 │   │ │
│  │     │ finally:                                       │   │ │
│  │     │   # Cancel timers                             │   │ │
│  │     │   signal.alarm(0)                             │   │ │
│  │     └────────────────────────────────────────────────┘   │ │
│  │                                                            │ │
│  │  8. Post-task Execution Hooks                             │ │
│  │     task_cls.after_return(                                │ │
│  │       status='SUCCESS',                                   │ │
│  │       retval=result,                                      │ │
│  │       task_id=request.id,                                 │ │
│  │       args=request.args,                                  │ │
│  │       kwargs=request.kwargs                               │ │
│  │     )                                                      │ │
│  │                                                            │ │
│  │  9. Return Result to Main Process                         │ │
│  │     return {                                               │ │
│  │       'status': 'SUCCESS',                                │ │
│  │       'result': 10,                                        │ │
│  │       'task_id': '550e8400-...',                          │ │
│  │       'traceback': None                                    │ │
│  │     }                                                      │ │
│  └──────────────────┬───────────────────────────────────────┘ │
└────────────────────┼────────────────────────────────────────────┘
                     │
                     │ Result returned via callback
                     ▼


Phase 4: Result Storage & ACK (Back to Consumer Thread)
═══════════════════════════════════════════════════════════════════

┌────────────────────────────────────────────────────────────────┐
│           Consumer Thread - Result Callback                     │
│                                                                  │
│  1. Receive Result from Pool                                    │
│     result_info = {                                             │
│       'status': 'SUCCESS',                                      │
│       'result': 10,                                             │
│       'task_id': '550e8400-...',                               │
│     }                                                            │
│                                                                  │
│  2. Store Result in Backend                                     │
│     ┌──────────────────────────────────────────────────────┐  │
│     │ if not task.ignore_result:                           │  │
│     │   backend.store_result(                              │  │
│     │     task_id=result_info['task_id'],                  │  │
│     │     result=result_info['result'],                    │  │
│     │     status='SUCCESS',                                 │  │
│     │     traceback=None,                                   │  │
│     │     request=request                                   │  │
│     │   )                                                    │  │
│     │                                                         │  │
│     │   # Redis backend:                                    │  │
│     │   key = f"celery-task-meta-{task_id}"                │  │
│     │   value = {                                           │  │
│     │     "status": "SUCCESS",                              │  │
│     │     "result": 10,                                     │  │
│     │     "traceback": null,                                │  │
│     │     "children": [],                                   │  │
│     │     "date_done": "2024-01-01T12:00:00.000000"        │  │
│     │   }                                                    │  │
│     │   redis.set(key, json.dumps(value))                  │  │
│     │   redis.expire(key, result_expires)  # TTL           │  │
│     └──────────────────────────────────────────────────────┘  │
│                                                                  │
│  3. Execute Callbacks (if any)                                  │
│     if request.callbacks:                                       │
│       for callback in request.callbacks:                        │
│         callback.apply_async(args=[result])                     │
│                                                                  │
│  4. Notify Chord (if part of chord)                             │
│     if request.chord:                                           │
│       chord_unlock(request.chord, result)                       │
│                                                                  │
│  5. Send Task Events (if monitoring enabled)                    │
│     if worker.send_events:                                      │
│       dispatcher.send('task-succeeded', uuid=task_id, result=10)│
│                                                                  │
│  6. ACK Message to Broker                                       │
│     ┌──────────────────────────────────────────────────────┐  │
│     │ if task_acks_late:                                    │  │
│     │   # ACK after task completed                          │  │
│     │   broker.ack(delivery_tag)                            │  │
│     │ else:                                                  │  │
│     │   # ACK was sent when task started (early ACK)       │  │
│     │                                                         │  │
│     │ # Redis:                                              │  │
│     │ ZREM unacked task_id                                  │  │
│     │                                                         │  │
│     │ # RabbitMQ:                                           │  │
│     │ channel.basic_ack(delivery_tag=delivery_tag)          │  │
│     └──────────────────────────────────────────────────────┘  │
│                                                                  │
│  7. Update Worker Stats                                         │
│     worker.stats['total_tasks_succeeded'] += 1                 │
│     worker.stats['tasks_per_second'].append(timestamp)          │
│                                                                  │
│  8. Check for Worker Restart                                    │
│     if task.max_tasks_per_child:                                │
│       if worker.tasks_processed >= max_tasks_per_child:         │
│         # Schedule worker restart                               │
│         pool.restart_worker(worker_pid)                         │
│                                                                  │
└────────────────────────────────────────────────────────────────┘


Phase 5: Result Retrieval (Client)
═══════════════════════════════════════════════════════════════════

┌─────────────┐
│ Application │  result.get()
└──────┬──────┘
       │
       │ 1. Poll backend for result
       ▼
┌────────────────────────────────────────────────────────────────┐
│  Result Retrieval Logic                                         │
│                                                                  │
│  AsyncResult.get(timeout=None, interval=0.5):                  │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  while True:                                              │ │
│  │    # 1. Check task state                                 │ │
│  │    meta = backend.get_task_meta(task_id)                 │ │
│  │                                                            │ │
│  │    # Redis:                                               │ │
│  │    key = f"celery-task-meta-{task_id}"                   │ │
│  │    meta = json.loads(redis.get(key) or '{}')             │ │
│  │                                                            │ │
│  │    # 2. Check status                                      │ │
│  │    if meta['status'] == 'SUCCESS':                       │ │
│  │      return meta['result']  # 10                         │ │
│  │                                                            │ │
│  │    elif meta['status'] == 'FAILURE':                     │ │
│  │      exception = meta.get('exception')                   │ │
│  │      traceback = meta.get('traceback')                   │ │
│  │      raise exception                                      │ │
│  │                                                            │ │
│  │    elif meta['status'] in ('PENDING', 'STARTED'):        │ │
│  │      # Still processing                                   │ │
│  │      if timeout and time.time() - start_time > timeout:  │ │
│  │        raise TimeoutError()                               │ │
│  │      time.sleep(interval)  # Wait and retry              │ │
│  │      continue                                             │ │
│  │                                                            │ │
│  │    elif meta['status'] == 'RETRY':                       │ │
│  │      # Task is being retried                             │ │
│  │      time.sleep(interval)                                 │ │
│  │      continue                                             │ │
│  └──────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────┘
```

## タスク状態の遷移

```
State Transition Diagram:
═══════════════════════════════════════════════════════════════════

┌─────────┐
│ PENDING │  (Initial state, task submitted)
└────┬────┘
     │
     │ task_track_started=True
     ▼
┌─────────┐
│ STARTED │  (Task received by worker)
└────┬────┘
     │
     ├─────────────┬──────────────┬──────────────┐
     │             │              │              │
     ▼             ▼              ▼              ▼
┌─────────┐  ┌─────────┐   ┌─────────┐   ┌─────────┐
│PROGRESS │  │ SUCCESS │   │ FAILURE │   │  RETRY  │
│(Custom) │  │         │   │         │   │         │
└─────────┘  └─────────┘   └────┬────┘   └────┬────┘
                                 │              │
                                 │              │
                                 │              └──┐
                                 │                 │
                                 ▼                 │
                           ┌─────────┐            │
                           │ REVOKED │            │
                           │(Killed) │            │
                           └─────────┘            │
                                                   │
                    ┌──────────────────────────────┘
                    │ retry()
                    ▼
              ┌─────────┐
              │ PENDING │  (Re-queued)
              └────┬────┘
                   │
                   └──> (Starts again)

State Details:
═══════════════════════════════════════════════════════════════════

PENDING:
- デフォルト状態
- バックエンドに何も記録されていない
- result.get()は待機状態

STARTED:
- task_track_started=True の場合のみ
- ワーカーがタスクを受信
- バックエンドに{"status": "STARTED"}を保存

PROGRESS:
- カスタム状態
- update_state(state='PROGRESS', meta={...})で設定
- 進捗情報を含む

SUCCESS:
- タスクが正常に完了
- バックエンドに結果を保存
- result.get()は結果を返す

FAILURE:
- タスクが例外を発生
- バックエンドにexception、tracebackを保存
- result.get()は例外を再発生

RETRY:
- タスクがリトライ中
- リトライカウント、次回実行時刻を保存
- 自動的にPENDINGに戻る

REVOKED:
- タスクがキャンセルされた
- result.revoke()で設定
- ワーカーは実行をスキップ
```

## リトライの詳細メカニズム

```
Retry Mechanism Deep Dive:
═══════════════════════════════════════════════════════════════════

Initial Execution:
┌────────────────────────────────────────────────────────────────┐
│ @app.task(bind=True, max_retries=3)                             │
│ def unreliable_task(self):                                      │
│     try:                                                         │
│         result = call_external_api()                            │
│         return result                                            │
│     except APIException as exc:                                 │
│         # Retry with exponential backoff                        │
│         countdown = 2 ** self.request.retries                   │
│         raise self.retry(exc=exc, countdown=countdown)          │
└────────────────────────────────────────────────────────────────┘

Retry Flow:
═══════════════════════════════════════════════════════════════════

Attempt 1:  retries=0
│
├─> APIException raised
│
├─> self.retry() called
│   ├─> countdown = 2^0 = 1 second
│   ├─> Check max_retries: 0 < 3 ✓
│   │
│   ├─> Build retry message:
│   │   {
│   │     "task": "unreliable_task",
│   │     "id": "550e8400-...",  # Same task ID
│   │     "retries": 1,            # Incremented
│   │     "eta": now() + 1s,       # Countdown applied
│   │     "args": [...],
│   │     "kwargs": {...},
│   │     "exception": "APIException: Connection timeout"
│   │   }
│   │
│   ├─> Update state to RETRY:
│   │   backend.store_result(
│   │     task_id="550e8400-...",
│   │     result=None,
│   │     status='RETRY',
│   │     exception="APIException: Connection timeout",
│   │     traceback=traceback_str
│   │   )
│   │
│   └─> Re-queue task:
│       apply_async(countdown=1)  # 1秒後に再実行
│
│ (Wait 1 second)
│
▼

Attempt 2:  retries=1
│
├─> APIException raised again
│
├─> self.retry() called
│   ├─> countdown = 2^1 = 2 seconds
│   ├─> Check max_retries: 1 < 3 ✓
│   ├─> retries = 2
│   └─> Re-queue with countdown=2s
│
│ (Wait 2 seconds)
│
▼

Attempt 3:  retries=2
│
├─> APIException raised again
│
├─> self.retry() called
│   ├─> countdown = 2^2 = 4 seconds
│   ├─> Check max_retries: 2 < 3 ✓
│   ├─> retries = 3
│   └─> Re-queue with countdown=4s
│
│ (Wait 4 seconds)
│
▼

Attempt 4:  retries=3
│
├─> APIException raised again
│
├─> self.retry() called
│   ├─> countdown = 2^3 = 8 seconds
│   ├─> Check max_retries: 3 < 3 ✗  # Max retries exceeded!
│   │
│   └─> Raise MaxRetriesExceededError:
│       ┌────────────────────────────────────────────────────┐
│       │ MaxRetriesExceededError:                           │
│       │   "Can't retry unreliable_task[550e8400-...]       │
│       │    Retry limit (3) exceeded."                      │
│       │                                                     │
│       │ Wrapped exception: APIException                    │
│       └────────────────────────────────────────────────────┘
│
├─> Task marked as FAILURE
│   backend.store_result(
│     task_id="550e8400-...",
│     result=None,
│     status='FAILURE',
│     exception="MaxRetriesExceededError: ...",
│     traceback=full_traceback
│   )
│
└─> result.get() raises MaxRetriesExceededError
```

## タイムアウトの実装

```
Time Limit Implementation:
═══════════════════════════════════════════════════════════════════

@app.task(time_limit=300, soft_time_limit=240)
def long_running_task():
    # 処理...
    pass

Worker Pool Process (UNIX):
┌────────────────────────────────────────────────────────────────┐
│                                                                  │
│  1. Soft Time Limit (240 seconds)                               │
│     ┌──────────────────────────────────────────────────────┐  │
│     │ signal.signal(signal.SIGUSR1, _soft_timeout_handler) │  │
│     │                                                         │  │
│     │ def _soft_timeout_handler(signum, frame):             │  │
│     │     raise SoftTimeLimitExceeded()                     │  │
│     │                                                         │  │
│     │ # Set alarm for 240 seconds                           │  │
│     │ signal.alarm(240)                                      │  │
│     └──────────────────────────────────────────────────────┘  │
│                                                                  │
│  2. Hard Time Limit (300 seconds)                               │
│     ┌──────────────────────────────────────────────────────┐  │
│     │ # Separate timer thread                               │  │
│     │ def _hard_timeout():                                  │  │
│     │     os.kill(os.getpid(), signal.SIGTERM)             │  │
│     │                                                         │  │
│     │ timer = threading.Timer(300, _hard_timeout)           │  │
│     │ timer.daemon = True                                   │  │
│     │ timer.start()                                          │  │
│     └──────────────────────────────────────────────────────┘  │
│                                                                  │
│  3. Task Execution Timeline                                     │
│                                                                  │
│  T=0s     Task starts                                           │
│  │                                                               │
│  │        [Task processing...]                                  │
│  │                                                               │
│  T=240s   SIGUSR1 delivered (soft limit)                        │
│  │        SoftTimeLimitExceeded raised                          │
│  │        ┌────────────────────────────────────────────────┐  │
│  │        │ try:                                           │  │
│  │        │   long_running_task()                          │  │
│  │        │ except SoftTimeLimitExceeded:                  │  │
│  │        │   # Task can cleanup                           │  │
│  │        │   cleanup_resources()                          │  │
│  │        │   save_partial_progress()                      │  │
│  │        │   raise  # Re-raise for retry                  │  │
│  │        └────────────────────────────────────────────────┘  │
│  │                                                               │
│  │        If task doesn't handle SoftTimeLimitExceeded...       │
│  │                                                               │
│  T=300s   SIGTERM delivered (hard limit)                        │
│  │        Process killed immediately                            │
│  │        No cleanup possible                                   │
│  X        [Process terminated]                                  │
│                                                                  │
│  Result: Task marked as FAILURE with TimeLimitExceeded          │
└────────────────────────────────────────────────────────────────┘

Windows (No signal support):
┌────────────────────────────────────────────────────────────────┐
│  Alternative: billiard library                                  │
│  - Uses multiprocessing.Process.terminate()                    │
│  - Less graceful than UNIX signals                              │
│  - Soft timeout not supported                                   │
└────────────────────────────────────────────────────────────────┘
```

## まとめ

タスク実行は以下の5つのフェーズで構成されます：

1. **Task Submission**: アプリケーションがタスクをシリアライズしてブローカーに送信
2. **Task Retrieval**: ワーカーがメッセージを取得・検証・デシリアライズ
3. **Task Execution**: ワーカープールプロセスでタスク関数を実行
4. **Result Storage**: 結果をバックエンドに保存し、メッセージをACK
5. **Result Retrieval**: クライアントが結果を取得

この複雑なフローを理解することで、パフォーマンス問題のデバッグやカスタマイズが容易になります。
