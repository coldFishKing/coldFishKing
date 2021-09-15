# Kubernetes-Qos之 Guaranteed, Burstable，Best-Effor

# Kubernetes的服务质量保证（QoS）

Kubernetes需要整体统筹平台资源使用情况、公平合理的将资源分配给相关pod容器使用，并且要保证容器生命周期内有足够的资源来保证其运行。 与此同时，由于资源发放的独占性，即资源已经分配给了某容器，同样的资源不会在分配给其他容器，对于资源利用率相对较低的容器来说，占用资源却没有实际使用（比如CPU、内存）造成了严重的资源浪费，Kubernetes需从优先级与公平性等角度综合考虑来提高资源的利用率。为了在资源被有效调度和分配的同时提高资源利用率，Kubernetes针对不同服务质量的预期，通过QoS（Quality of Service）来对pod进行服务质量管理，提供了个采用*requests*和*limits*两种类型对资源进行分配和使用限制。对于一个pod来说，服务质量体现在两个为2个具体的指标： CPU与内存。实际过程中，当NODE节点上内存资源紧张时，kubernetes会根据预先设置的不同QoS类别进行相应处理。

#### 设置资源限制的原因

如果未做过节点 nodeSelector，亲和性（node affinity）或pod亲和、反亲和性（pod affinity/anti-affinity）等[Pod高级调度策略](https://links.jianshu.com/go?to=http%3A%2F%2Fdockone.io%2Farticle%2F2635)设置，我们没有办法指定服务部署到指定机器上，如此可能会造成cpu或内存等密集型的pod全部都被分配到相同的Node上，造成资源竞争。另一方面，如果未对资源进行限制，一些关键的服务可能会因为资源竞争因OOM（Out of Memory）等原因被kill掉，或者被限制CPU使用。

#### 资源需求（Requests）和限制（ Limits）

对于每一个资源，container可以指定具体的资源需求（requests）和限制（limits），*requests*申请范围是0到node节点的最大配置，而*limits*申请范围是*requests*到无限，即0 <= requests <=Node Allocatable, requests <= limits <= Infinity。

**对于CPU，如果pod中服务使用CPU超过设置的limits，pod不会被kill掉但会被限制。如果没有设置limits，pod可以使用全部空闲的cpu资源。**

**对于内存，当一个pod使用内存超过了设置的limits，pod中container的进程会被kernel因OOM kill掉。当container因为OOM被kill掉时，系统倾向于在其原所在的机器上重启该container或本机或其他重新创建一个pod。**

#### QoS分类

Kubelet提供QoS服务质量管理，支持系统级别的OOM控制。在Kubernetes中，pod的QoS级别包括：*Guaranteed*, *Burstable*与 *Best-Effort*。下面对各级别分别进行相应说明：

**Guaranteed**：pod中的所有容器都必须对cpu和memory同时设置*limits*，如果有一个容器要设置*requests*，那么所有容器都要设置，并设置参数同*limits*一致，那么这个pod的QoS就是*Guaranteed*级别。
注：如果一个容器只指明*limit*而未设定*request*，则*request*的值等于*limit*值。

Guaranteed举例1：容器只指明了*limits*而未指明*requests*）。



```undefined
containers:
name: foo
resources:
  limits:
    cpu: 10m
    memory: 1Gi
name: bar
resources:
  limits:
    cpu: 100m
    memory: 100Mi
```

Guaranteed举例2：*requests*与*limit*均指定且值相等。



```undefined
containers:
name: foo
resources:
  limits:
    cpu: 10m
    memory: 1Gi
  requests:
    cpu: 10m
    memory: 1Gi

name: bar
resources:
  limits:
    cpu: 100m
    memory: 100Mi
  requests:
    cpu: 100m
    memory: 100Mi
```

**Burstable**: pod中只要有一个容器的*requests*和*limits*的设置不相同，该pod的QoS即为*Burstable*。举例如下：

Container bar没有指定*resources*



```undefined
containers:
name: foo
resources:
  limits:
    cpu: 10m
    memory: 1Gi
  requests:
    cpu: 10m
    memory: 1Gi

name: bar
```

*Burstable*举例2：pod中只要有一个容器没有对cpu或者memory中的request和limits都没有明确指定。



```undefined
containers:
name: foo
resources:
  limits:
    memory: 1Gi

name: bar
resources:
  limits:
    cpu: 100m
```

*Burstable*举例3：Container foo没有设置*limits*，而bar *requests*与 *limits*均未设置。



```undefined
containers:
name: foo
resources:
  requests:
    cpu: 10m
    memory: 1Gi  
name: bar
```

**Best-Effort**：如果对于全部的resources来说*requests*与*limits*均未设置，该pod的QoS即为*Best-Effort*。举例如下：



```undefined
containers:
name: foo
resources:
name: bar
resources:
```

#### 可压缩资源与不可压缩资源

Kubernetes根据资源能否伸缩进行分类，划分为可压缩资源和不可以压缩资源2种。
CPU资源是目前支持的一种可压缩资源，而内存资源和磁盘资源为目前所支持的不可压缩资源。

#### QoS优先级

3种QoS优先级从有低到高（从左向右）：

Best-Effort pods -> Burstable pods -> Guaranteed pods

#### 静态pod

在Kubernetes中有一种DaemonSet类型pod，此类pod可以常驻在某个Node上运行，由该Node上kubelet服务直接管理而无需api server介入。静态pod也无需关联任何RC，完全由kubelet服务来监控，当kubelet发现静态pod停止时，kubelet会重新启动静态pod。

#### 资源回收策略

当kubernetes集群中某个节点上可用资源比较小时，kubernetes提供了资源回收策略保证被调度到该节点pod服务正常运行。当节点上的内存或者CPU资源耗尽时，可能会造成该节点上正在运行的pod服务不稳定。Kubernetes通过kubelet来进行回收策略控制，保证节点上pod在节点资源比较小时可以稳定运行。

**可压缩资源：CPU**
在压缩资源部分已经提到CPU属于可压缩资源，当pod使用超过其设置的*limits*值时，pod中的进程会被限制使用cpu，但不会被kill。

**不可压缩资源：内存**
Kubernetes通过cgroup给pod设置QoS级别，当资源不足时先kill优先级低的pod，在实际使用过程中，通过OOM分数值来实现，OOM分数值从0-1000。

OOM分数值根据OOM_ADJ参数计算得出，对于Guaranteed级别的pod，OOM_ADJ参数设置成了-998，对于BestEffort级别的pod，OOM_ADJ参数设置成了1000，对于Burstable级别的POD，OOM_ADJ参数取值从2到999。对于kubernetes保留资源，比如kubelet，docker，OOM_ADJ参数设置成了-999，表示不会被OOM kill掉。OOM_ADJ参数设置的越大，通过OOM_ADJ参数计算出来OOM分数越高，表明该pod优先级就越低，当出现资源竞争时会越早被kill掉，对于OOM_ADJ参数是-999的表示kubernetes永远不会因为OOM而被kill掉。

**QoS pods被kill掉的场景与顺序**

- Best-Effort 类型的pods：系统用完了全部内存时，该类型pods会最先被kill掉。
- Burstable类型pods：系统用完了全部内存，且没有Best-Effort container可以被kill时，该类型pods会被kill掉。
- Guaranteed pods：系统用完了全部内存、且没有Burstable与Best-Effort container可以被kill，该类型的pods会被kill掉。
  注：如果pod进程因使用超过预先设定的*limites*而非Node资源紧张情况，系统倾向于在其原所在的机器上重启该container或本机或其他重新创建一个pod。

#### 使用建议

- 如果资源充足，可将QoS pods类型均设置为*Guaranteed*。用计算资源换业务性能和稳定性，减少排查问题时间和成本。
- 如果想更好的提高资源利用率，业务服务可以设置为*Guaranteed*，而其他服务根据重要程度可分别设置为*Burstable*或*Best-Effort*，例如filebeat。

#### 已知问题

不支持swap，当前的QoS策略假设swap已被禁止。

#### 参考资料

转自[http://dockone.io/article/2592](https://links.jianshu.com/go?to=http%3A%2F%2Fdockone.io%2Farticle%2F2592)并进行了修改

Resource Quality of Service in Kubernetes: [https://github.com/kubernetes/ ... os.md](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fkubernetes%2Fcommunity%2Fblob%2Fmaster%2Fcontributors%2Fdesign-proposals%2Fresource-qos.md)

Introduction to Quality Of Service and Service Level Agreement: [http://www.loriotpro.com/Produ ... N.htm](https://links.jianshu.com/go?to=http%3A%2F%2Fwww.loriotpro.com%2FProducts%2FOn-line_Documentation_V5%2FLoriotProDoc_EN%2FT20-Quality_of_Service%2FT20-A1_Quality_of_Service_EN.htm)

Kubelet pod level resource management: [https://github.com/kubernetes/ ... nt.md](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fkubernetes%2Fcommunity%2Fblob%2F72f87813b4c64d5ee8ac829f70edee2159871b6b%2Fcontributors%2Fdesign-proposals%2Fpod-resource-management.md)

Kubelet - Eviction Policy: [https://github.com/kubernetes/ ... on.md](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fkubernetes%2Fcommunity%2Fblob%2F72f87813b4c64d5ee8ac829f70edee2159871b6b%2Fcontributors%2Fdesign-proposals%2Fkubelet-eviction.md)