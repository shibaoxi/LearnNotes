# JOB

## job 概念

Job 会创建一个或者多个 Pods，并确保指定数量的 Pods 成功终止。 随着 Pods 成功结束，Job 跟踪记录成功完成的 Pods 个数。 当数量达到指定的成功个数阈值时，任务（即 Job）结束。 删除 Job 的操作会清除所创建的全部 Pods。

一种简单的使用场景下，你会创建一个 Job 对象以便以一种可靠的方式运行某 Pod 直到完成。 当第一个 Pod 失败或者被删除（比如因为节点硬件失效或者重启）时，Job 对象会启动一个新的 Pod。

## job运行单个实例

```yml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job
spec:
  template:
    metadata:
      labels:
        app: batch-job  #如果没有指定pod选择器，则根据pod模板中的标签创建
    spec:
      restartPolicy: OnFailure  #job不能使用Always 为默认的重新启动策略
      containers:
      - name: main
        image: davidshi/batch-job
```

## job运行多个实例

- 顺序运行job pod

```yml
apiVersion: batch/v1
kind: Job
metadata:
  name: multi-completion-batch-job
spec:
  completions: 5  #将completions设置为5，将使此作业顺序运行5个pod
  template:
    metadata:
      labels:
        app: batch-job  
    spec:
      restartPolicy: OnFailure  #job不能使用Always 为默认的重新启动策略
      containers:
      - name: main
        image: davidshi/batch-job
```

- 并行运行job pod

```yml
apiVersion: batch/v1
kind: Job
metadata:
  name: multi-completion-batch-job
spec:
  completions: 5  #将completions设置为5，这项任务必须确保5个pod成功完成
  parallelism: 2  #最多两个pod可以并行运行
  template:
    metadata:
      labels:
        app: batch-job  
    spec:
      restartPolicy: OnFailure  #job不能使用Always 为默认的重新启动策略
      containers:
      - name: main
        image: davidshi/batch-job
```

- 定期运行job pod

```yml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: batch-job-every-fifteen-minutes
spec:
  schedule: "0,15,30,45 * * * *"  #这项工作应该每天在每小时0，15，30和45分钟运行
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: periodic-batch-job
        spec:
          restartPolicy: OnFailure
          containers:
          - name: main
            image: davidshi/batch-job
```

时间表从左到右包含分钟、小时、每月中的第几天、月、星期几。

```bash
0,15,30,45 * * * *
```

这意味着每小时的0、15、30和45分钟

```bash
0，30 * 1 * *
```

每隔30分钟运行一次，仅在每月的第一天运行

```bash
0 3 * * 0
```

每个星期天的3AM运行 星期天用0表示
