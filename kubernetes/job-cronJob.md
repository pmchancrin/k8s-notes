# Job 分类
**Batch Job:** 计算业务/离线业务，跑完就退出；  
**Long Running Job**：长作业。

# Job

```
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: resouer/ubuntu-bc 
        command: ["sh", "-c", "echo 'scale=10000; 4*a(1)' | bc -l "]
      restartPolicy: Never
  backoffLimit: 4

```

**restartPolicy** 在Job里面只允许被设置为Never和OnFailure；而在Deployment对象里，只需运行被设置为Always。  

restartPolicy为Never，当Job失败会不断尝试创建一个新的Pod。  

restartPolicy为OnFailure，当Job失败后，会尝试重启Pod，不会重新创建。  

spec.backoffLimit字段可以限制创建的次数，Job重新创建的间隔是指数增长的。   

spec.activeDeadlineSeconds是设置最长运行时间，超过这个时间就会被终止。 

spec.parallelism，Job任意时间最多可以启动多少个Pod同时运行。  
spec.completions，Job完成的Pod数目，就是最终用户需要的Pod数目。

## Demo

### Demo 1

外部管理器+Job模板

最通用的方式：

```
apiVersion: batch/v1
kind: Job
metadata:
  name: process-item-$ITEM
  labels:
    jobgroup: jobexample
spec:
  template:
    metadata:
      name: jobexample
      labels:
        jobgroup: jobexample
    spec:
      containers:
      - name: c
        image: busybox
        command: ["sh", "-c", "echo Processing item $ITEM && sleep 5"]
      restartPolicy: Never
```

外部脚本或者程序控制$ITEM变量，这些Job都又同一个标签。

```
$ mkdir ./jobs
$ for i in apple banana cherry
do
  cat job-tmpl.yaml | sed "s/\$ITEM/$i/" > ./jobs/job-$i.yaml
done

```

### demo 2

拥有固定任务数目的并行Job

```
apiVersion: batch/v1
kind: Job
metadata:
  name: job-wq-1
spec:
  completions: 8
  parallelism: 2
  template:
    metadata:
      name: job-wq-1
    spec:
      containers:
      - name: c
        image: myrepo/job-wq-1
        env:
        - name: BROKER_URL
          value: amqp://guest:guest@rabbitmq-service:5672
        - name: QUEUE
          value: job1
      restartPolicy: OnFailure

```
这种方式，只关心指定任务数 completions，不关心parallelism数。 任务可以从工作队列里面获取，生成者-消费者模式。

```
/* job-wq-1 的伪代码 */
queue := newQueue($BROKER_URL, $QUEUE)
task := queue.Pop()
process(task)
exit

```

# CronJob

CronJob 定时任务，是一个Job对象控制器。

```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure

```

*/1 * * * *  从0开始，每1个单位时间执行一次，分，时，日，月，星期

concurrencyPolicy=Allow，默认情况，Job可以同时存在；  
concurrencyPolicy=Forbid，不会创建新Pod，该创建周期被跳过；  
concurrencyPolicy=Replace，新产生的Job会替换旧的、没执行完的Job。  
spec.startingDeadlineSeconds，多少秒里，miss的数目达到100次，这个Job就不会再被创建。 （Job创建失败，就会被记miss）  
