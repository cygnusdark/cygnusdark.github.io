---
title: Flink Checkpoint 机制
date: 2023-07-01
categories:  大数据
tags:
  - Flink
---

## checkpoint 的过程包含了 JobManager 和 Taskmanager 端 task 的执行过程，按照步骤为：	
1. 在 JobManager 端构建 ExecutionGraph 过程中会创建 CheckpointCoordinator，这是负责 checkpoint 的核心实现类，同时会给 job 添加一个监听器 CheckpointCoordinatorDeActivator（只有设置了 checkpoint 才会注册这个监听器），在 JobManager 端开始进行任务调度的时候，会对 job 的状态进行转换，由 CREATED 转成 RUNNING，job 监听器 CheckpointCoordinatorDeActivator 就开始启动 checkpoint 的定时任务了，最终会调用 CheckpointCoordinator.startCheckpointScheduler ()


2. CheckpointCoordinator 会部署一个定时任务，用于周期性的触发 checkpoint，这个定时任务就是 ScheduledTrigger，在触发 checkpoint 之前先做一遍检查，检查当前正在处理的 checkpoint 是否超过设置的最大并发 checkpoint 数量，检查 checkpoint 的间隔是否达到设置的两次 checkpoint 的时间间隔，在都没有问题的情况下向所有的 source task 去触发 checkpoint，远程调用 TaskManager 的 triggerCheckpoint () 方法


3. TaskManager 的 triggerCheckpoint () 方法首先获取到 source task（即 SourceStreamTask），调用 Task.triggerCheckpointBarrier ()，triggerCheckpointBarrier () 会异步的去执行一个独立线程，这个线程来负责 source task 的 checkpoint 执行。checkpoint 的核心实现在 StreamTask.performCheckpoint () 方法中，该方法主要有三个步骤：
	1. 在 checkpoint 之前做一些准备工作，通常情况下 operator 在这个阶段是不做什么操作的
	2. 立即向下游广播 CheckpointBarrier，以便使下游的 task 能够及时的接收到 CheckpointBarrier 也开始进行 checkpoint 的操作
	3. 开始进行状态的快照，即 checkpoint 操作。
	注意以上操作都是在同步代码块里进行的，获取到的这个 lock 锁就是用于 checkpoint 的锁，checkpoint 线程和 task 任务线程用的是同一把锁，在进行 performCheckpoint () 时，task 任务线程是不能够进行数据处理的

4. checkpoint 的执行过程是一个异步的过程，保证不能因为 checkpoint 而影响了正常数据流的处理。StreamTask 里的每个 operator 都会创建一个 OperatorSnapshotFutures，OperatorSnapshotFutures 里包含了执行 operator 状态 checkpoint 的 FutureTask，然后由另一个单独的线程异步的来执行这些 operator 的实际 checkpoint 操作，就是执行这些 FutureTask。这个异步线程叫做 AsyncCheckpointRunnable，checkpoint 的执行就是将状态数据推送到远程的存储介质中


5. 对于非 Source Task，checkpoint 的标志性开始在接收到上游的 CheckpointBarrier，方法在 StreamTask 中的 CheckpointBarrierHandler.getNextNonBlocked ()。CheckpointBarrierHandler 会根据 CheckpointingMode 模式不同生成不同的 Handler，如果是 EXACTLY_ONCE，就会生成 BarrierBuffer，会进行 barrier 对齐，保证数据的一致性，BarrierBuffer 中的 CachedBufferBlocker 是用来缓存 barrier 对齐时从被阻塞 channel 接收到的数据。如果 CheckpointingMode 是 AT_LEAST_ONCE，那就会生成 BarrierTracker，不会进行 barrier 对齐，而是继续处理数据，在接收到上游 task 所有的 CheckpointBarrier 才开始进程 checkpoint，这样就会 checkpoint (n) 的状态会包含 checkpoint (n+1) 的数据，数据不一致。非 Source Task 的 checkpoint 执行跟步骤 3、4 是一样的，只不过触发的线程是 Task 工作线程，跟 source task 不一样


6. Task 在执行完 checkpoint 后会向 JobManager 上报 checkpoint 的元数据信息，JobManager 端的 CheckpointCoordinator 会调用 PendingCheckpoint.acknowledgeTask () 方法，该方法就是将 task 上报的元数据信息（checkpoint 的路径地址，状态数据大小等等）添加到 PendingCheckpoint 里


7. task 的 checkpoint 会一直进行到 sink task。JobManager 如果接收到了全部 task 上报的的 Ack 消息，就执行 completePendingCheckpoint ()，会将 checkpoint 元数据信息进行持久化，然后通知所有的 task 进行 commit 操作，一般来说，task 的 commit 操作其实不需要做什么，但是像那种 TwoPhaseCommitSinkFunction，比如 FlinkKafkaProducer 就会进行一些事物的提交操作等，或者像 FlinkKafkaConsumer 会进行 offset 的提交


8. 所有 task 执行完 commit 操作后（实际上执行的是 operator.notifyCheckpointComplete () 方法），一个完整的 checkpoint 流程就完成了