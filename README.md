# openshift（kubernetes）上elasticsearch群集编排包括持久化与横向扩展


### 提前测试相关镜像的可下载性
```
docker pull registry.dataos.io/library/elasticsearch:2.3 #版本自行修改
```

### 创建Service和DeploymentConfig

**cat es-svc-node1.yaml**

```
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: es
    name: es1
  name: es1
spec:
  ports:
  - name: port-9200
    port: 9200
    protocol: TCP
    targetPort: 9200
  - name: port-9300
    port: 9300
    protocol: TCP
    targetPort: 9300
  selector:
    app: es
    node: es1
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}

```

**es-dc-node1.yaml**

```
apiVersion: v1
kind: DeploymentConfig
metadata:
  annotations:
    openshift.io/deployment.cancelled: "3"
  creationTimestamp: null
  labels:
    node: es1
  name: es-cluster1
spec:
  replicas: 1
  selector:
    node: es1
  strategy:
    resources: {}
    rollingParams:
      intervalSeconds: 1
      maxSurge: 25%
      maxUnavailable: 25%
      timeoutSeconds: 600
      updatePeriodSeconds: 1
    type: Rolling
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: es
        node: es1
    spec:
      containers:
      - args:
        - -Des.cluster.name=elasticsearch-cluster #群集名
        - -Des.node.name=node-1   #该节点名称
        - -Des.discovery.zen.ping.multicast.enabled=false  #关闭组播，防止影响其他elasticsearch群集
        - -Des.discovery.zen.ping_timeout=120s
        - -Des.discovery.zen.minimum_master_nodes=2  #master不少于两个
        - -Des.client.transport.ping_timeout=60s
        - -Des.discovery.zen.ping.unicast.hosts=es1,es2,es3   #定义三台
        image: registry.dataos.io/library/elasticsearch:2.3
        imagePullPolicy: IfNotPresent
        name: es-cluster1
        resources: {}
        terminationMessagePath: /dev/termination-log
        volumeMounts:
        - mountPath: /usr/share/elasticsearch/data
          name: storage
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - name: storage
        persistentVolumeClaim:
          claimName: elastic-storage-1       #以glusterfs方式的持久化卷
  test: false
  triggers:
  - type: ConfigChange
status: {}
```

### 以上为一个节点的
其他两个节点service 和 deploymentconfig 请查看es-svc-node2.yaml 、es-svc-node3.yaml 、es-dc-node2.yaml 、 es-dc-node3.yaml

### 最后创建群集service，用于kibana

cat es-svc-cluster.yaml

```
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: es
  name: es
spec:
  ports:
  - name: port-9200
    port: 9200
    protocol: TCP
    targetPort: 9200
  - name: port-9300
    port: 9300
    protocol: TCP
    targetPort: 9300
  selector:
    app: es
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
```


### 测试：
```
oc expose svc es #用于获取route，即$URL
```


##### 1.查询索引
```
curl http://$URL/syslog
curl http://$URL/_aliases 
```
##### 2.写入数据测试

```
curl -XPUT http://$URL/syslog -d '{  "user": "kimchy" }'


curl -XPUT http://$URL/syslog11 -d '
{
    "user": "11111",
    "post_date": "2016-09-27T13:12:00",
    "message": "Trying out Elasticsearch, su far so good?"
}'
```

##### 3.写入的数据查询

```
curl -XGET http://$URL/syslog
curl -XGET http://$URL/syslog11
```

##### 4.删除所有pod

