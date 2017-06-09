# Heapster monitor java by custom metrics

Heapster是kubernetes的监控组件, 它负责收集Node上Cadvisor提供的监控信息, 推送给上游监控组件. 除了缺省的监控项(cpu/memory/net/storage/fs), Cadvisor还提供应用监控功能, 用户可以通过自定义监控项扩展监控范围. 而Heapster默认是支持收集Cadvisor的应用监控指标. 下面是一个例子, 展示怎样通过自定义应用监控项, 让Heapster收集部署在K8S集群中的java应用监控信息(通过jmx).

## Cadvisor application metrics[^1]
Cadvisor在收集Container监控信息的时候, 会检测是否存在以`io.cadvisor.metric`开头的Label(值指向一个配置文件), 如果存在, Cadvisor会认为这个Container有应用监控指标需要收集, 他就会去读取Label指向的配置文件, 配置文件的格式:

```
{
  "endpoint" : "http://localhost:1234/metrics",
  "metrics_config" : [
    "jvm_memory_pool_bytes_used",
    "jvm_threads_current",
    "tomcat_threadpool_currentthreadsbusy"
  ]
}
```

上面是一个Promethus格式的endpoint, 输出信息是结构化的, Cadvisor会请求配置文件中的endpoint, 获取对应的监控项; 当然, 如果你希望收集endpoint提供的所有监控项, 省略metrics_config.

另外, Cadvisor还支持非结构化的信息, 具体就不展开了.

## K8S custom metrics
上面描述了Cadvisor的应用监控扩展, 而在K8S中, 这个功能被称为Custom metrics, 该功能缺省是关闭的, 需要在kubelet中增加启动参数`enable-custom-metrics`才能启用该功能.

kubelet在创建container的时候, 会做两件事情: 首先, 它会确认是否该功能已经启用, 其次, 检测Pod是否有Volume mount在`/etc/custom-metrics`, 同时, 在该目录下存在`definition.json`这个文件. 满足上述条件, 他会在该Container上增加Label: `io.cadvisor.metric.promethus=/etc/custom-metrics/definition.json`. 有了这个Label, Cadvisor才会对该Container进行应用监控指标收集.

## endpoint
在K8S中, Cadvisor运行在Host网络, 在请求配置文件中的endpoint时, 需要考虑这个问题, 我们需要把PodIP注入到配置文件中(当然你也可以用NodePort这种方式, 但由于端口冲突问题, 基本不可行).

## Java应用监控
监控一个运行Java应用的Pod, 我们需要Jmx, 这里我们用[jmx_exporter](https://github.com/prometheus/jmx_exporter), jmx_exporter能把Jmx监控信息转换成Promethus风格的输出, 便于Cadvisor收集.

我们需要考虑怎么把配置文件放入Pod中, 注意上面提到的: `/etc/custom-metrics`必须是一个Volume mount, 所以把配置文件打包到Image的做法行不通. 目前可行的做法有三种: ConfigMap, initContainer, emptyDir Volume. 这里我们用emptyDir volume(configMap个人觉得不太方便, initContainer也是emptyDir Volume的变形).

下面我们用docker.io/tomcat做baseimage, 制作支持custom metrics的image.

我们需要增加修改catalina.sh增加jmx支持, 同时把配置文件注入:

```
echo "{\"endpoint\":\"http://$POD_IP:1234/metrics\"}" >/etc/custom-metrics/definition.json
JAVA_OPTS="$JAVA_OPTS -javaagent:$CATALINA_HOME/lib/jmx_prometheus_javaagent-0.9.jar=1234:$CATALINA_HOME/lib/config.yaml"
```

增加[jmx-exporter jar](https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.9/jmx_prometheus_javaagent-0.9.jar)和配置文件[config.yaml](https://github.com/prometheus/jmx_exporter/blob/master/example_configs/tomcat.yml)到image.

Dockerfile内容:
```
[root@primeton-tacy-k8s-master custom_metrics]# cat Dockerfile
FROM tomcat:8.0.44-jre8-alpine
ADD  jmx_prometheus_javaagent-0.9.jar /usr/local/tomcat/lib/
ADD  config.yaml /usr/local/tomcat/lib/
ADD  catalina.sh /usr/local/tomcat/bin/
```

我制作好的imagey已经push在docker.io上, 可以直接用来测试, 测试方法:

```
[root@primeton-tacy-k8s-master 1.5]# cat tomcat-metrics.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    run: tomcat-metrics
  name: tomcat-metrics
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      run: tomcat-metrics
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        run: tomcat-metrics
    spec:
      containers:
      - image: tacylee/tomcat:8.0.44-jre8-alpine
        imagePullPolicy: IfNotPresent
        name: tomcat-metrics
        resources: {}
        terminationMessagePath: /dev/termination-log
        volumeMounts:
        - mountPath: /etc/custom-metrics
          name: custommetrics
        env:
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - name: custommetrics
        emptyDir: {}

[root@primeton-tacy-k8s-master 1.5]# kubectl create -f tomcat-metrics.yaml

[root@primeton-tacy-k8s-master 1.5]# kubectl get pods -o wide
NAME                             READY     STATUS    RESTARTS   AGE       IP                NODE
heapster-b95w3                   1/1       Running   0          22h       192.168.172.239   primeton-tacy-k8s-node2
tomcat-metrics-133681138-kphcm   1/1       Running   0          1d        192.168.172.237   primeton-tacy-k8s-node2

[root@primeton-tacy-k8s-master 1.5]# curl http://localhost:8080/api/v1/proxy/nodes/primeton-tacy-k8s-node2/stats/summary
...
      "userDefinedMetrics": [
       {
        "name": "jvm_threads_deadlocked_monitor",
        "type": "gauge",
        "units": "",
        "time": "2017-06-09T13:38:10Z",
        "value": 0
       },
       {
        "name": "tomcat_bytesreceived_total",
        "type": "cumulative",
        "units": "",
        "time": "2017-06-09T13:38:10Z",
        "value": 0
       },
       {
        "name": "jmx_config_reload_success_total",
        "type": "cumulative",
        "units": "",
        "time": "2017-06-09T13:38:10Z",
        "value": 0
       },
...
```

看到userDefinedMetrics就可以了.

最后, 你需要部署heapster组件, 直接用log sink验证, 能看到有`custom/jmx_config_reload_failure_total = 0.000000`类似的指标.

```
[root@primeton-tacy-k8s-master heapster]# cat heapster.yaml
apiVersion: v1
kind: ReplicationController
metadata:
  labels:
    k8s-app: heapster-test
    name: heapster
  name: heapster
spec:
  replicas: 1
  selector:
    k8s-app: heapster-test
  template:
    metadata:
      labels:
        k8s-app: heapster-test
    spec:
      containers:
      - name: heapster-test
        image: tacylee/heapster-openfalcon:v1.2
        imagePullPolicy: Always
        command:
        - /heapster
        - --source=kubernetes.summary_api:https://kubernetes.default
        - --sink=log
        - -logtostderr=1
```


[^1]: [Collecting Application Metrics with cAdvisor](https://github.com/google/cadvisor/blob/master/docs/application_metrics.md)
