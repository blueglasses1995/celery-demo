# ワーカーアーキテクチャの詳細

## ワーカープロセスの構造

```
Celery Worker Process Hierarchy:
═══════════════════════════════════════════════════════════════════

┌──────────────────────────────────────────────────────────────────┐
│                    Main Process (Parent)                          │
│                    PID: 1234                                      │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │              Component Manager                              │ │
│  │  - 各コンポーネントのライフサイクル管理                      │ │
│  │  - Graceful shutdown coordination                           │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │              Consumer (celery.worker.consumer)              │ │
│  │  ┌──────────────────────────────────────────────────────┐ │ │
│  │  │ Connection: Redis/RabbitMQ                           │ │ │
│  │  │ - broker_connection                                  │ │ │
│  │  │ - connection_pool                                    │ │ │
│  │  └──────────────────────────────────────────────────────┘ │ │
│  │                                                              │ │
│  │  ┌──────────────────────────────────────────────────────┐ │ │
│  │  │ Event Loop (kombu.asynchronous)                      │ │ │
│  │  │ - select/epoll for I/O multiplexing                 │ │ │
│  │  │ - Message dispatch                                   │ │ │
│  │  └──────────────────────────────────────────────────────┘ │ │
│  │                                                              │ │
│  │  ┌──────────────────────────────────────────────────────┐ │ │
│  │  │ QoS Manager (Quality of Service)                    │ │ │
│  │  │ - Prefetch count management                          │ │ │
│  │  │ - Flow control                                       │ │ │
│  │  └──────────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │          Pool Manager (billiard.pool)                       │ │
│  │  - Worker pool process lifecycle                           │ │
│  │  - Task dispatch to worker processes                       │ │
│  │  - Result collection                                        │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │          Beat (Scheduler) - Optional                        │ │
│  │  - Periodic task scheduling                                 │ │
│  │  - Schedule file management                                 │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │          Event Dispatcher - Optional                        │ │
│  │  - Sends worker/task events to monitoring                  │ │
│  │  - Flower, custom monitoring tools                          │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │          Timer Service                                      │ │
│  │  - ETA tasks scheduling                                     │ │
│  │  - Periodic housekeeping                                    │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │          Control Command Handler                            │ │
│  │  - Remote control commands (shutdown, pool_restart, etc)   │ │
│  └────────────────────────────────────────────────────────────┘ │
└────────────────────────┬─────────────────────────────────────────┘
                         │
                         │ Spawns worker pool processes
                         ▼
┌──────────────────────────────────────────────────────────────────┐
│                    Worker Pool (Child Processes)                  │
│                                                                    │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │ Worker Process 1│  │ Worker Process 2│  │ Worker Process N│ │
│  │  PID: 1235      │  │  PID: 1236      │  │  PID: 1237      │ │
│  │                 │  │                 │  │                 │ │
│  │ ┌─────────────┐ │  │ ┌─────────────┐ │  │ ┌─────────────┐ │ │
│  │ │Execute task │ │  │ │Execute task │ │  │ │ Idle        │ │ │
│  │ └─────────────┘ │  │ └─────────────┘ │  │ └─────────────┘ │ │
│  │                 │  │                 │  │                 │ │
│  │ Task count: 42  │  │ Task count: 39  │  │ Task count: 0   │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│                                                                    │
│  Communication: multiprocessing.Queue / Pipe                      │
└──────────────────────────────────────────────────────────────────┘
```

## Prefork Pool の詳細実装

```
Prefork Pool Architecture (Default):
═══════════════════════════════════════════════════════════════════

┌──────────────────────────────────────────────────────────────────┐
│                        Main Process                               │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │           Task Queue (multiprocessing.Queue)                │ │
│  │  ┌──────┬──────┬──────┬──────┬──────┐                     │ │
│  │  │Task 1│Task 2│Task 3│Task 4│Task 5│ ...                 │ │
│  │  └──────┴──────┴──────┴──────┴──────┘                     │ │
│  │  ↑                                                          │ │
│  │  │ Producer: Consumer thread puts tasks here              │ │
│  │  │                                                          │ │
│  │  └─ Backed by pipe: /tmp/pymp-XXX                         │ │
│  └────────────────────────────────────────────────────────────┘ │
│                          │                                        │
│                          │ Shared memory pipe                     │
│                          ▼                                        │
└──────────────────────────────────────────────────────────────────┘
                           │
            ┌──────────────┼──────────────┐
            │              │              │
            ▼              ▼              ▼
┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│ Worker Process 1 │ │ Worker Process 2 │ │ Worker Process N │
│                  │ │                  │ │                  │
│ ┌──────────────┐ │ │ ┌──────────────┐ │ │ ┌──────────────┐ │
│ │ Event Loop   │ │ │ │ Event Loop   │ │ │ │ Event Loop   │ │
│ │              │ │ │ │              │ │ │ │              │ │
│ │ while True:  │ │ │ │ while True:  │ │ │ │ while True:  │ │
│ │   task =     │ │ │ │   task =     │ │ │ │   task =     │ │
│ │   queue.get()│ │ │ │   queue.get()│ │ │ │   queue.get()│ │
│ │   execute()  │ │ │ │   execute()  │ │ │ │   execute()  │ │
│ └──────────────┘ │ │ └──────────────┘ │ │ └──────────────┘ │
│                  │ │                  │ │                  │ │
│ Python process   │ │ Python process   │ │ Python process   │ │
│ Separate memory  │ │ Separate memory  │ │ Separate memory  │ │
│ No GIL conflict  │ │ No GIL conflict  │ │ No GIL conflict  │ │
└──────────────────┘ └──────────────────┘ └──────────────────┘
            │              │              │
            └──────────────┼──────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│           Result Queue (multiprocessing.Queue)                    │
│  ┌──────┬──────┬──────┐                                         │
│  │Res 1 │Res 2 │Res 3 │ ...                                     │
│  └──────┴──────┴──────┘                                         │
│                                                                    │
│  Main process collects results                                   │
└──────────────────────────────────────────────────────────────────┘

Worker Lifecycle:
═══════════════════════════════════════════════════════════════════

Initialization:
┌────────────────────────────────────────────────────────────────┐
│ 1. Main process spawns N workers                               │
│    for i in range(concurrency):                                │
│      p = multiprocessing.Process(target=worker_main)           │
│      p.start()                                                  │
│                                                                  │
│ 2. Each worker process:                                         │
│    - Imports task modules                                       │
│    - Sets up signal handlers                                    │
│    - Connects to result backend (own connection)               │
│    - Enters event loop                                          │
└────────────────────────────────────────────────────────────────┘

Steady State:
┌────────────────────────────────────────────────────────────────┐
│ Worker N event loop:                                            │
│   while not shutdown_flag:                                      │
│     try:                                                         │
│       # Blocking wait for task                                  │
│       request = task_queue.get(timeout=1.0)                    │
│                                                                  │
│       # Execute task                                            │
│       result = execute_task(request)                            │
│                                                                  │
│       # Send result back                                        │
│       result_queue.put(result)                                  │
│                                                                  │
│       # Check if max_tasks_per_child reached                   │
│       tasks_executed += 1                                       │
│       if tasks_executed >= max_tasks_per_child:                │
│         break  # Exit to be restarted                           │
│                                                                  │
│     except Empty:                                               │
│       continue  # Timeout, check shutdown flag                  │
│                                                                  │
│     except Exception as exc:                                    │
│       logger.error(f'Worker error: {exc}')                     │
│       result_queue.put(WorkerLostError(exc))                   │
└────────────────────────────────────────────────────────────────┘

Shutdown:
┌────────────────────────────────────────────────────────────────┐
│ 1. Graceful shutdown (SIGTERM):                                │
│    - shutdown_flag = True                                       │
│    - Complete current task                                      │
│    - Exit event loop                                            │
│    - Cleanup resources                                          │
│                                                                  │
│ 2. Force shutdown (SIGKILL):                                   │
│    - Immediate termination                                      │
│    - Current task lost                                          │
│    - No cleanup                                                 │
└────────────────────────────────────────────────────────────────┘
```

## Gevent Pool の詳細実装

```
Gevent Pool Architecture:
═══════════════════════════════════════════════════════════════════

┌──────────────────────────────────────────────────────────────────┐
│                    Single Python Process                          │
│                    (Event-driven, Cooperative)                    │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │              Gevent Hub (Event Loop)                        │ │
│  │  - epoll/kqueue for I/O multiplexing                       │ │
│  │  - Greenlet scheduling                                      │ │
│  └────────────────────────────────────────────────────────────┘ │
│                          │                                        │
│                          │ Schedules greenlets                    │
│                          ▼                                        │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │            Greenlet Pool (gevent.pool.Pool)                 │ │
│  │                                                              │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │ │
│  │  │Greenlet 1│  │Greenlet 2│  │Greenlet 3│  │Greenlet N│  │ │
│  │  │          │  │          │  │          │  │          │  │ │
│  │  │ Task A   │  │ Task B   │  │ Task C   │  │ Idle     │  │ │
│  │  │ (I/O)    │  │ (Running)│  │ (Waiting)│  │          │  │ │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘  │ │
│  │                                                              │ │
│  │  Same memory space, lightweight context switching          │ │
│  │  Cooperative multitasking (explicit yield points)          │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                    │
│  Memory Layout:                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Heap (Shared by all greenlets)                            │ │
│  │  ┌──────────────────────────────────────────────────────┐ │ │
│  │  │ Python objects, task data, etc.                      │ │ │
│  │  └──────────────────────────────────────────────────────┘ │ │
│  │                                                              │ │
│  │  Stack (Per greenlet, small ~1KB-8KB)                      │ │
│  │  ┌────────┐ ┌────────┐ ┌────────┐                         │ │
│  │  │G1 stack│ │G2 stack│ │GN stack│ ...                     │ │
│  │  └────────┘ └────────┘ └────────┘                         │ │
│  └────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘

Execution Flow:
═══════════════════════════════════════════════════════════════════

┌────────────────────────────────────────────────────────────────┐
│ Task A: HTTP Request                                            │
│   1. Start: socket.send(request)                               │
│   2. Would block → yield to hub                                │
│      ┌─────────────────────────────────────────────────────┐  │
│      │ Gevent patches socket.send():                        │  │
│      │   if would_block:                                    │  │
│      │     gevent.sleep(0)  # Yield to hub                 │  │
│      │     # Hub schedules other greenlets                  │  │
│      │   return data                                        │  │
│      └─────────────────────────────────────────────────────┘  │
│   3. Hub switches to Task B                                    │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│ Task B: Database Query                                          │
│   1. db.execute(query)                                          │
│   2. Waiting for DB → yield                                    │
│   3. Hub switches to Task C                                    │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│ Task C: Computation (CPU-bound)                                │
│   1. result = heavy_calculation()                              │
│   2. No I/O → runs to completion                               │
│   3. Blocks all other greenlets! (GIL held)                    │
│      ⚠️ Problem: Gevent is bad for CPU-bound tasks            │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│ Event ready (e.g., HTTP response arrived)                      │
│   1. epoll notifies hub: socket readable                       │
│   2. Hub resumes Task A greenlet                               │
│   3. Task A: data = socket.recv()                              │
│   4. Task A completes                                           │
└────────────────────────────────────────────────────────────────┘

Advantages:
- 軽量: 1000+ 同時greenlets可能
- 低オーバーヘッド: コンテキストスイッチが速い
- メモリ効率: 共有メモリ空間

Disadvantages:
- CPU-bound tasksで全体がブロック
- Monkey patchingが必要（標準ライブラリの置き換え）
- デバッグが難しい
```

## プリフェッチとACKの詳細

```
Prefetch and ACK Mechanism:
═══════════════════════════════════════════════════════════════════

Configuration:
worker_prefetch_multiplier = 4
concurrency = 4
→ Total prefetch = 4 * 4 = 16 messages

State Diagram:
┌────────────────────────────────────────────────────────────────┐
│                       Broker Queue                              │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ [M1] [M2] [M3] [M4] [M5] [M6] [M7] [M8] [M9] [M10] ...  │ │
│  └──────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────┘
              │
              │ Prefetch 16 messages
              ▼
┌────────────────────────────────────────────────────────────────┐
│                    Worker Local Buffer                          │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ Prefetched (not yet ACKed):                              │ │
│  │ [M1][M2][M3][M4][M5][M6][M7][M8]                         │ │
│  │ [M9][M10][M11][M12][M13][M14][M15][M16]                 │ │
│  └──────────────────────────────────────────────────────────┘ │
│              │                                                  │
│              │ Dispatch to pool                                │
│              ▼                                                  │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ Worker Pool (4 workers):                                 │ │
│  │  Worker1: Executing M1   (unacked)                       │ │
│  │  Worker2: Executing M2   (unacked)                       │ │
│  │  Worker3: Executing M3   (unacked)                       │ │
│  │  Worker4: Executing M4   (unacked)                       │ │
│  └──────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────┘

Flow with task_acks_late=False (Early ACK):
═══════════════════════════════════════════════════════════════════

T=0s    Worker receives M1 from broker
        │
        ├─> Immediately ACK to broker
        │   broker.ack(delivery_tag_1)
        │   ✓ M1 removed from broker queue
        │
        └─> Dispatch M1 to pool
            │
            └─> Worker1 starts executing M1

T=5s    Worker1 crashes mid-execution
        │
        └─> ❌ Task M1 is LOST (already ACKed)
            No recovery possible

Flow with task_acks_late=True (Late ACK):
═══════════════════════════════════════════════════════════════════

T=0s    Worker receives M1 from broker
        │
        ├─> Do NOT ACK yet
        │   M1 remains in broker's "unacked" set
        │
        └─> Dispatch M1 to pool
            │
            └─> Worker1 starts executing M1

T=10s   Worker1 completes M1 successfully
        │
        ├─> Store result in backend
        │
        └─> ACK to broker
            broker.ack(delivery_tag_1)
            ✓ M1 removed from broker

Alternative: Worker1 crashes at T=5s
        │
        ├─> Broker detects connection lost
        │
        └─> M1 still in "unacked" set
            ├─> Broker re-queues M1
            └─> Another worker will process M1
                ✓ Task is NOT lost

Visibility Timeout (Redis specific):
═══════════════════════════════════════════════════════════════════

Redis doesn't have native "unacked" concept, so Celery implements it:

T=0s    Worker receives M1
        │
        ├─> ZADD unacked <timestamp> M1_id
        │   Add to sorted set with current time as score
        │
        └─> Execute M1

Background thread: Visibility checker
        │
        │ Every 10 seconds:
        │   old_timestamp = now() - visibility_timeout
        │   lost_tasks = ZRANGEBYSCORE unacked 0 old_timestamp
        │
        │   for task in lost_tasks:
        │     RPUSH queue task_message  # Re-queue
        │     ZREM unacked task_id       # Remove from unacked
        │
        └─> Ensures tasks don't get lost

T=15s   M1 completes
        │
        └─> ZREM unacked M1_id
            Remove from unacked set

Configuration Impact:
═══════════════════════════════════════════════════════════════════

Small worker_prefetch_multiplier (e.g., 1):
┌────────────────────────────────────────────────────────────────┐
│ Advantages:                                                     │
│ - Fair distribution (long tasks don't block others)            │
│ - Better for long-running tasks                                │
│ - Lower memory usage                                            │
│                                                                  │
│ Disadvantages:                                                  │
│ - More network round-trips                                     │
│ - Lower throughput for short tasks                             │
└────────────────────────────────────────────────────────────────┘

Large worker_prefetch_multiplier (e.g., 100):
┌────────────────────────────────────────────────────────────────┐
│ Advantages:                                                     │
│ - Fewer network round-trips                                    │
│ - Higher throughput for short tasks                            │
│                                                                  │
│ Disadvantages:                                                  │
│ - Unfair distribution (one worker may hoard tasks)            │
│ - Higher memory usage                                           │
│ - Bad for long-running tasks                                   │
└────────────────────────────────────────────────────────────────┘
```

## まとめ

Celeryワーカーは複雑なマルチプロセス/マルチスレッドアーキテクチャを持ち、以下の主要コンポーネントで構成されています：

- **Main Process**: メッセージ受信、ディスパッチ、結果収集
- **Worker Pool**: タスク実行（prefork/gevent/threads）
- **QoS Manager**: フロー制御とプリフェッチ管理
- **Event System**: モニタリングとコントロール

プリフェッチとACKメカニズムを理解することで、信頼性とパフォーマンスのトレードオフを適切に調整できます。
