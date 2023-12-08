---
layout: post
title: 有关Kubernetes的面试题
subtitle: 面试可能会遇到的问题
author: FatGuy010
permalink: /K8sInterviewQuestions
tags: [ 面试 ]
---



## 什么是pod ？

1. 在docker中支持以containers容器的方式部署应用,但是一个容器只能部署一个软件应用,实际情况中如果想让一个软件正常运行通常需要部署多个应用配合运行才能对外提供功能,docker中针对这种情况会把这些应用部署为一组容器,比较繁琐
2. Pod是k8s中最小的部署单元,一个pod中可以运行一个或多个容器,这些容器共享存储、网络、以及怎样运行这些容器的声明,一般不直接创建Pod，而是创建一些工作负载由工作负载来创建Pod
3. pod 内的容器都是平等的



## 直接创建 pod的缺点 ?

1. 直接创建的pod,该pod中的容器如果宕机异常,有恢复功能, 但是这个pod如果宕机异常不会有恢复功能,所以创建一些工作负载由他们来创建Pod,实现pod的异常恢复功能
2. 在直接部署pod时,虽然可以通过podIP属性指定该pod的访问ip,但是集群网络是私有的,该地址只能在集群内部访问,如果想允许外部访问需要创建Service来公开Pod的网络终结点



## Pod 中的多容器协同

1. 什么是多容器协同: 在一个pod中部署多个Container应用容器,多个应用容器配合工作

   - 网络：每个Pod都会被分配一个唯一的IP地址**,Pod中的所有容器共享网络空间**,包括IP地址和端口。Pod内部的容器可以使用localhost互相通信。Pod中的容器与外界通信时，必须分配共享网络资源（例如使用宿主机的端口映射）

   - 存储: 可以Pod指定多个共享的Volume。Pod中的所有容器共享volume。Volume也可以用来持久化Pod中的存储资源，以防容器重启后文件丢失

2. 多容器协同的好处: 

- 举个不太恰当的例子,服务间相互调用现在有a,b,c,d四个服务,d是配合工作的附加服务,多个服务间是通过d服务进行通信的,这样在pod中就可以部署为"a,d",“b,d”,"c,d"在pod内部abc三个主服务都附加一个d服务

- 优点:

  - 将相关的服务放在同一个 Pod 中可以共享资源,例如网络和存储，避免了在不同 Pod 之间进行跨节点通信的开销

  - 更高的可靠性：这种部署方式可以增加服务之间的稳定性和可靠性,可以通过本地的 localhost 直接通信，而且共享相同的生命周期

  - 简化部署和管理：将相关服务打包到同一个 Pod 中可以简化部署和管理过程。例如可以通过 Deployment 或 StatefulSet 来管理 Pod，可以实现集中的配置、更新和扩展，同时降低了管理成本

  - 安全性增强：因为打包到同一个pod的服务之间的通信是通过 localhost 进行的，不需要经过网络暴露，减少了一些潜在的攻击



## Pod 的组成与paush (重要)

1. 一个pod中可以部署多个容器，多个容器之间共享网络，通过localhost就可以通信,共享文件资源。每个pod中除了运行的应用容器外，每个pod内部都存在一个特殊的Pause容器。

   [![pi2AgpD.png](https://z1.ax1x.com/2023/12/08/pi2AgpD.png)](https://imgse.com/i/pi2AgpD)

2. Pasu 容器通过 `kubectl  get pods` 是看不到的，需要在 对应节点的下通过 `docker ps | grep` "pod名称"才可看到

3. 每个pod内部都会部署一个Pause特殊容器,pod在启动容器时会优先启动这个特殊容器,可以将这个特殊容器看成用来管理同一pod下的其它应用容器的,它负责了: **创建和占用pod的NetworkNamespace网络命名空间和PID命名空间:**

   - 简单解释就是pause容器在启动时,会为 pod 生成一个独立的网络环境和进程环境,并将这些环境与其他 pod 隔离开,其中:

   - 创建和占用pod的NetworkNamespace网络命名空间,网络命名空间是Linux的一种技术,通过pause为pod分配一个虚拟 IP 地址分配一个独立的虚拟网络接口和 IP 地址,当前pod中的其他用户容器就可以加入到这个网络命名空间中,从而共享同一个IP和网络接口,提供了网络通信环境,使得它们可以通过 localhost 进行通信,也可以访问外部网络

   - 创建和占用pod 的 PID 命名空间，并为自己分配一个 PID 为 1 的进程，pod 中的其他用户容器就可以加入到这个 PID 命名空间中,共享同一个进程 ID 空间,通过共享进程ID空间实现了: 1pod 内的容器之间的进程监控和管理, 2pod 内的容器之间的信号传递, 3pod 内的容器之间的僵尸进程的回收避免了资源泄漏(注意好像在1.8版本中被修改默认情况下是禁用的)

4. 回收僵尸进程的步骤如下：

   - 首先，当 pod 中的某个用户容器启动时，它会加入到 pause 容器创建和占用的 PID 命名空间中，从而共享同一个进程 ID 空间

   - 当 pod 中的某个用户容器中的某个子进程退出时，会变成一个僵尸进程，也就是已经结束运行，但是还没被父进程回收

   - 由于 pod 中的所有容器都在同一个 PID 命名空间中，所以都可以看到这个僵尸进程，并且可以向它发送 SIGCHLD 信号，通知它的父进程回收它

   - pause 容器是 pod 中第一个启动的容器，并且分配了 PID 为 1 的进程，它会成为 pod 中所有孤儿进程（没有父进程或者父进程已经退出的进程）的父进程。因此，当 pause 容器收到 SIGCHLD 信号时，它会调用 wait() 函数来回收僵尸进程，并释放其占用的资源

- ### 面试题: 

为什么一个pod中的多个容器可以共享网络:

因为在pod在部署容器时,每个应用都会多部署一个对应的paush容器,通过这个paush容器设置来设置当前实际容器的网络

- ### 面试题: 

如果pod中的容器宕机重启,ip会变吗?

不会,底层会通过对应的paush去设置网络,但是如果重建 pod 则IP 可能会变化



## Pod 的生命周期

1. pod对象从创建到终止的这段时间范围称为pod的生命周期,主要包含:

   - pod创建

   - 运行初始化容器（initContainer）

   - 运行主容器（main container）过程

   - 容器启动后钩子postStart执行，容器终止前preStop钩子

   - 容器的存活性探测（liveness probe），就绪性探测（readness probe）

   - pod终止

2. 在整个声明周期中，Pod会出现5种状态(相位)

   - 挂起Pending：挨批server已经创建了pod资源对象，但它尚未被调度完成或者仍处于下载镜像的过程中

   - 运行中Running：pod已经被调度至某节点，并且所有容器都已经被kubectl创建完成

   - 成功Succeed：pod中的所有容器都已经成功终止并且不会被重启

   - 失败Failed：所有容器都已经停止，但至少有一个容器终止失败，即容器返回了非0的退出状态

   - 未知Unknown：apiserver无法正常获取到pod对象的状态信息，通常由网络通信失败所致

3. **Pod的创建过程详解:**

   - 用户通过kubectl或其他api客户端提交需要创建的pod信息给apiServer

   - apiServer开始生成pod对象的信息，并将信息存入etcd，然后返回确认信息至客户端

   - apiServer开始反映etcd中的pod对象的变化，其他组件使用watch机制来跟踪检查apiServer上的变动

   - scheduler发现有新的pod对象要创建，开始为pod分配足迹并将结果更新只apiServer

   - node节点上的kubectl发现有pod调度过来，尝试调用docker启动容器，并将结果回送至apiServer

   - apiServer将接收到的pod状态信息存入etcd中

4. Pod的终止过程详解:

   - 用户向apiServer发送删除pod对象的命令,执行`kubectl delete {pod_id}`删除老pod

   - 在删除pod时好像有一个在宽限期默认30秒，此时pod被视为dead

   - 后续apiServer接收到删除命令后,会在etd中将根据pod_id将老pod标记为terminating退出中

   - kube-proxy监听endPoint,当发现pod对象转为terminating状态,会触发iptables-restore,生成全新的iptables规则(上线pod时会执行多次,下线pod时好些只会执行一次),新的iptables DNAT规则中就会删除当前pod_id,将当前pod提出负载均衡

   - 如果当前pod对象定义了preStop钩子处理器，则在其标记为terminating后即会以同步的方式执行

   - kubelet监听到etcd中pod status状态变化,当发现属于当前节点时,走针对pod的实际退出流程—>preStop执行pod的退出回调–>关闭监听端口–>处理现有连接–>完成退出

   - 注意kube-proxy检查到下线pod命令执行删除地址动作与kubelet执行删除实际pod是同步执行的,所以可能会出现pod已经实际删除了,但是地址还存在endpoint列表中,造成请求异常



## 静态Pod

kl8s中pod分为静态pod与动态pod,我们使用k8s指定部署的应用pod称为动态pod,而k8s这个服务中的基础设施启动需要的pod称为静态pod, 在/etc/kubernetes/manifests位置放了创建这些静态pod的Pod.yaml文件，机器启动kubelet自己就把他启动起来。静态Pod一直守护在他的这个机器上



## 探针

​	上面了解到pod是有状态有生命周期的,通过这个状态又延伸出了重启策略, 重启底层实际就基于探针实现的,通过探测和重启策略实现了服务的健康检查,0宕机

​	在容器的containers中存在三个属性: **startupProbe启动探针, livenessProbe存活探针, readinessProbe就绪探针**

- startupProbe启动探针: 用来探测当前容器是否启动成功

- livenessProbe存活探针: 用来判断当前容器是否存活,例如当探测到容器不存活时,会重新拉起

- readinessProbe就绪探针: 用来探测当前容器是否就绪,能否能够对外提供服务,以调用服务负载均衡为例,当接收到请求后如果通过该探针探测到某个服务节点不可用,则不会将该节点加入负载均衡

​	探针支持的三种设置方法(与钩子相同)

- exec: 通过钩子程序执行命令

- httpGet:通过钩子发送http get请求

- tcpSocket: 容器创建之后连接tcp端口进行指定操作



1. ### livenessProbe存活探针

   - livenessProbe存活探针用于判断容器是不是健康，如果不满足健康条件，Kubelet 将根据 Pod 中设置的 restartPolicy 重启策略来判断，Pod 是否要进行重启。

   - LivenessProbe按照配置去探测 ( 进程、或者端口、或者命令执行后是否成功等等)，来判断容器是不是正常。如果探测不到，代表容器不健康（可以配置连续多少次失败才记为不健康），则 kubelet 会杀掉该容器，并根据容器的重启策略做相应的处理。

   - 如果未配置存活探针，则默认容器启动为Success通过状态。即Success后pod状态是RUNING

2. ### readinessProbe就绪探针

   - readinessProbe就绪探针，用于判断容器内的程序是否存活或者说是否健康，是否启动完成并就绪,正常对外提供服务

   - 容器启动后会按照readinessProbe的配置进行探测, 探测成功返回 Success。pod的READY状态变为 true，更新pod成功数量比如1/1，否则还是0/1。

   - 若未配置就绪探针，则默认容器启动后状态Success。此时pod、pod关联的Service、EndPoint 等资源都会进入Ready 状态,进行相关设置

   - 后续程序运行中还可以通过readinessProbe继续监测, 如果探测失败,更新Pod 的 Ready 状态变为 false，系统则会在对应的Service关联的 EndPoint 列表中去除此pod地址，实现服务的异常踢除。如果 Pod 恢复为 Ready 状态。将再会被加回 Endpoint 列表。kube-proxy也将有概率通过负载机制会引入流量到此pod中

3. ### startupProbe启动探针

   - 启动探针时 k8s 在1.16版本后增加startupProbe探针，主要解决在复杂的程序中readinessProbe、livenessProbe探针无法更好的判断程序是否启动、是否存活。进而引入startupProbe探针为readinessProbe、livenessProbe探针服务,

   - startupProbe探针与另两种区别: 如果三个探针同时存在，先执行startupProbe探针，其他两个探针将会被暂时禁用，直到pod满足startupProbe探针配置的条件，其他2个探针启动，如果不满足按照规则重启容器, 另外两种探针在容器启动后，会按照配置，直到容器消亡才停止探测，而startupProbe探针只是在容器启动后按照配置满足一次后，不在进行后续的探测

4. ### 就绪、存活两种探针的区别

   - ReadinessProbe 和 livenessProbe 可以使用相同探测方式，只是对 Pod 的处置方式不同：

   - readinessProbe 当检测失败后，将 Pod 的 IP:Port 从对应的 EndPoint 列表中删除。

   - livenessProbe 当检测失败后，将杀死容器并根据 Pod 的重启策略来决定作出对应的措施

5. ### startupProbe的存在意义？

   - startupProbe 和 livenessProbe 最大的区别就是startupProbe在探测认为成功之后就不会继续探测了，而livenessProbe在pod的生命周期中一直在探测,并且其它探针是在startupProbe认为成功后才会执行,。如果只设置livenessProbe探针会存在如下问题： "一个服务如果前期启动需要很长时间，那么它后面死亡未被发现的时间就越长，为什么会这么说呢？假设我们一个服务A启动完成需要2分钟，那么我们如下开始定义livenessProbe,5s就会根据重启策略进行一次重启，这个时候你会发现pod一直会陷入死循环"

6. ~~~yaml
   livenessProbe:
     httpGet:
       path: /test
       prot: 80
   failureThreshold: 1
   initialDelay: 5
   periodSeconds: 5
   ~~~

   - 为解决以上问题修改为,使用启动探针startupProbe,程序有605s=300s的启动时间，当startupProbe探针探测成功之后，才会被livenessProbe接管，这样在运行中出问题livenessProbe就能在15=5s内发现。如果启动探测是3分钟内还没有探测成功，则接受Pod的重启策略进行重启

7. ~~~yaml
   livenessProbe:
     httpGet:
       path: /test
       prot: 80
   failureThreshold: 1
   initialDelay: 5
   periodSeconds: 5
   
   startupProbe:
     httpGet:
       path: /test
       prot: 80
   failureThreshold: 60
   initialDelay: 5
   periodSeconds: 5
   ~~~



## Pod的tolerations容忍策略相关

1. Pod中可以通过tolerations进行容忍相关设置,实现了如下功能

   - 调度约束：通过设置tolerations容忍,可以将pod调度安装到指定节点上
   - 弹性和容错性：在节点出现故障或维护时，节点上的Pod可能需要迁移到其他节点上。通过定义tolerations，可以确保Pod仍然能够被分配到指定节点上,并且可以设置等待迁移时间

2. Pod下tolerations内部是一个数组属性,可以设置多个容忍规则,其中内部包含:

   1. key：设置需要匹配的污点(注意此处除了可以设置自定义污点外,也可以设置k8s内部预定义的污点,实现某些特殊功能)

   2. operator：指定如何与节点上的污点进行比较。常用的匹配操作符有以下几种：

      - Equal：要求污点的键值与toleration规则的键值完全相等。

      - Exists：只要存在与toleration规则的键相同的污点键，即可匹配成功。

3. value：匹配的值,污点也是有值的,当污点匹配成功后,匹配污点的值,如果未指定该属性，则默认为""空字符串。

4. effect：设置匹配成功后的执行规则,可以设置为以下几种：

   - NoSchedule：当Pod满足toleration规则时，将不会被调度到带有匹配污点的节点上。

   - PreferNoSchedule：当Pod满足toleration规则时，尽量不要被调度到带有匹配污点的节点上，但并不是绝对禁止调度

   - NoExecute：当Pod满足toleration规则时，如果该节点上已经存在该污点，则会从该节点上驱逐（删除）Pod。

5. tolerationSeconds：定义容忍时间，用于指定容忍期限。当一个节点上的污点超过了容忍时间，Pod将被驱逐。该属性是可选的，如果未指定，默认为tolerationSeconds: null，表示没有容忍期限

   例如：

6. ~~~yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: my-pod
   spec:
     containers:
       - name: my-container
         image: nginx:latest
     tolerations:  # 指定tolerations部分
       - key: "critical-node"  # 指定需要容忍的污点
         operator: "Equal"  # 指定如何与节点上的污点进行比较,当前是做等值判断
         value: "true"  # 指定节点上污点的预期值判断,默认为""空字符串
         effect: "NoSchedule"  # 当匹配规则满足时的作用效果,当前表示为不调度安装到存在该污点的节点
   ~~~

7. ### k8s内置的污点与节点异常容忍时间问题

   1. k8s中内置了多种污点,可以将这些污点设置到tolerations的key属性上实现一些特殊功能,例如

      - node.kubernetes.io/not-ready: 用于表示节点当前处于不可用状态
      - node.kubernetes.io/unreachable：用于标记节点不可达。当节点无法与集群通信时，该污点会自动添加到节点上，以确保不会将新的Pod调度到该节点上。
      - node.kubernetes.io/out-of-disk：用于标记节点磁盘空间不足。当节点的可用磁盘空间不足时，该污点会自动添加到节点上，防止将新的Pod调度到可能导致更多磁盘使用的节点上。
      - node.kubernetes.io/memory-pressure：用于标记节点内存压力过大。当节点的可用内存资源不足时，该污点会自动添加到节点上，以避免将新的Pod调度到可能导致更多内存使用的节点上。
      - node.kubernetes.io/disk-pressure：用于标记节点磁盘压力过大。当节点的磁盘压力过大时，该污点会自动添加到节点上，以避免将新的Pod调度到可能导致更多磁盘使用的节点上。
      - node.kubernetes.io/network-unavailable：用于标记节点网络不可用。当节点的网络无法正常工作时，该污点会自动添加到节点上，以防止将新的Pod调度到网络不可用的节点上。

   2. 以节点异常容忍时间为例举例说明内置污点的使用,首先什么是节点的异常容忍时间,意思是k8s集群中如果某个节点意外宕机,在发生宕机时,该节点上部署的应用会自动迁移到其它可用节点,迁移过程可能需要一些时间,取决与硬件性能,网络性能,镜像拉取时间等等,为此k8s引入了一个称为"容忍时间"Toleration的概念,用于控制每个Pod在发生迁移时,允许等待的时间，默认为300秒,如果在这个等待时间内没有重新调度到其它节点完成运行,这个pod将会标记为丢失状态(注意这个等待时间只对已经允许的pod有效,对于新部署的pod无效)

   3. 如何修改容忍时间,或者如何缩短容忍时间(不要修改过短,要考虑服务的启动时间,如果过短考虑网络延迟问题,会出现pod反复迁移),通过部署pod的tolerations属性设置

      - 首先在节点发生异常时,会自动给节点添加 node.kubernetes.io/not-ready表示当前节点处于不可用状态的污点,和node.kubernetes.io/unreachable节点通信异常表示当前节点不可达的污点

      - 在pod模板中的tolerations属性中,通过key设置匹配这两个污点,设置不在存在该污点的节点上调度安装

      - 在pod模板中的tolerations属性中,通过tolerationSeconds指定容忍时间,当前不设置默认为300秒,既在pod发生迁移时如果指定时间内为迁移成功,该pod会标记为丢失状态

        例如：

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
spec:
  containers:
    - name: my-container
      image: nginx
  tolerations:
    - key: node.kubernetes.io/unreachable #匹配的污点,该污点是k8s内置的
      operator: Exists #匹配规则,Exists表示存在
      effect: NoSchedule
      tolerationSeconds: 300  # 设置容忍时间默认300秒
    - key: node.kubernetes.io/not-ready #匹配的污点,该污点是k8s内置的
      operator: Exists #匹配规则,Exists表示存在
      effect: NoSchedule #效果,NoSchedule表示如果匹配成功,则在对应的节点上驱逐,也就是不会调度安装到对应的节点上
      tolerationSeconds: 300  # 设置容忍时间默认300秒
~~~



## K8s 面试题整理:

**1、 k8s是什么？请说出你的了解**？**

答：Kubernetes是一个针对容器应用，进行自动部署，弹性伸缩和管理的开源系统。主要功能是生产环境中的容器编排。

K8S是Google公司推出的，它来源于由Google公司内部使用了15年的Borg系统，集结了Borg的精华。



**2、 K8s架构的组成是什么？**

答：和大多数分布式系统一样，K8S集群至少需要一个主节点（Master）和多个计算节点（Node）。

- 主节点主要用于暴露API，调度部署和节点的管理；
- 计算节点运行一个容器运行环境，一般是docker环境（类似docker环境的还有rkt），同时运行一个K8s的代理（kubelet）用于和master通信。计算节点也会运行一些额外的组件，像记录日志，节点监控，服务发现等等。计算节点是k8s集群中真正工作的节点。

K8S架构细分：

1、Master节点（默认不参加实际工作）：

- Kubectl：客户端命令行工具，作为整个K8s集群的操作入口；
- Api Server：在K8s架构中承担的是“桥梁”的角色，作为资源操作的唯一入口，它提供了认证、授权、访问控制、API注册和发现等机制。客户端与k8s群集及K8s内部组件的通信，都要通过Api Server这个组件；
- Controller-manager：负责维护群集的状态，比如故障检测、自动扩展、滚动更新等；
- Scheduler：负责资源的调度，按照预定的调度策略将pod调度到相应的node节点上；
- Etcd：担任数据中心的角色，保存了整个群集的状态；

2、Node节点：

- Kubelet：负责维护容器的生命周期，同时也负责Volume和网络的管理，一般运行在所有的节点，是Node节点的代理，当Scheduler确定某个node上运行pod之后，会将pod的具体信息（image，volume）等发送给该节点的kubelet，kubelet根据这些信息创建和运行容器，并向master返回运行状态。（自动修复功能：如果某个节点中的容器宕机，它会尝试重启该容器，若重启无效，则会将该pod杀死，然后重新创建一个容器）；

- Kube-proxy：Service在逻辑上代表了后端的多个pod。负责为Service提供cluster内部的服务发现和负载均衡（外界通过Service访问pod提供的服务时，Service接收到的请求后就是通过kube-proxy来转发到pod上的）；

container-runtime：是负责管理运行容器的软件，比如docker

- Pod：是k8s集群里面最小的单位。每个pod里边可以运行一个或多个container（容器），如果一个pod中有两个container，那么container的USR（用户）、MNT（挂载点）、PID（进程号）是相互隔离的，UTS（主机名和域名）、IPC（消息队列）、NET（网络栈）是相互共享的。



**3、 容器和主机部署应用的区别是什么？**

答：容器的中心思想就是秒级启动；一次封装、到处运行；这是主机部署应用无法达到的效果，但同时也更应该注重容器的数据持久化问题。

另外，容器部署可以将各个服务进行隔离，互不影响，这也是容器的另一个核心概念。



**4、请说一下kubernetes针对pod资源对象的健康监测机制？**

答：K8s中对于pod资源对象的健康状态检测，提供了三类probe（探针）来执行对pod的健康监测：

1. livenessProbe探针

   ​	可以根据用户自定义规则来判定pod是否健康，如果livenessProbe探针探测到容器不健康，则kubelet会根据其重启策略来决定是否重启，如果一个容器不包含livenessProbe探针，则kubelet会认为容器的livenessProbe探针的返回值永远成功。

2. ReadinessProbe探针

   ​	同样是可以根据用户自定义规则来判断pod是否健康，如果探测失败，控制器会将此pod从对应service的endpoint列表中移除，从此不再将任何请求调度到此Pod上，直到下次探测成功。

3. startupProbe探针

   ​	启动检查机制，应用一些启动缓慢的业务，避免业务长时间启动而被上面两类探针kill掉，这个问题也可以换另一种方式解决，就是定义上面两类探针机制时，初始化时间定义的长一些即可。

每种探测方法能支持以下几个相同的检查参数，用于设置控制检查时间：

- initialDelaySeconds：初始第一次探测间隔，用于应用启动的时间，防止应用还没启动而健康检查失败

- periodSeconds：检查间隔，多久执行probe检查，默认为10s；

- timeoutSeconds：检查超时时长，探测应用timeout后为失败；

- successThreshold：成功探测阈值，表示探测多少次为健康正常，默认探测1次。

上面两种探针都支持以下三种探测方法：

1. Exec：通过执行命令的方式来检查服务是否正常，比如使用cat命令查看pod中的某个重要配置文件是否存在，若存在，则表示pod健康。反之异常。
2. Httpget：通过发送http/htps请求检查服务是否正常，返回的状态码为200-399则表示容器健康（注http get类似于命令curl -I）。
3. tcpSocket：通过容器的IP和Port执行TCP检查，如果能够建立TCP连接，则表明容器健康，这种方式与HTTPget的探测机制有些类似，tcpsocket健康检查适用于TCP业务。

在上述的yaml配置文件中，两类探针都使用了，在容器启动5秒后，kubelet将发送第一个readinessProbe探针，这将连接容器的8080端口，如果探测成功，则该pod为健康，十秒后，kubelet将进行第二次连接。

除了readinessProbe探针外，在容器启动15秒后，kubelet将发送第一个livenessProbe探针，仍然尝试连接容器的8080端口，如果连接失败，则重启容器。

探针探测的结果无外乎以下三者之一：

- Success：Container通过了检查；
- Failure：Container没有通过检查；
- Unknown：没有执行检查，因此不采取任何措施（通常是没有定义探针检测，默认为成功）。



**5、 如何控制滚动更新过程？**

答：可以通过下面的命令查看到更新时可以控制的参数：

- maxSurge：此参数控制滚动更新过程，副本总数超过预期pod数量的上限。可以是百分比，也可以是具体的值。默认为1。
- maxUnavailable：此参数控制滚动更新过程中，不可用的Pod的数量。



**6、什么是Kubernetes？它的主要目标是什么？**

答：Kubernetes是一个开源容器编排平台，用于自动化部署、扩展和管理容器化应用程序。它的主要目标是简化容器化应用的部署和管理，并提供弹性、可靠的应用程序编排。



**7、什么是Pod？**

答：Pod是Kubernetes的最小调度和部署单元。它是一个包含一个或多个容器的逻辑主机，这些容器共享网络和存储资源，并且在同一主机上共享生命周期。



**8、什么是ReplicaSet？**

答：ReplicaSet是Kubernetes的控制器之一，用于确保在集群中运行指定数量的Pod副本。如果Pod的数量少于指定的副本数，ReplicaSet将创建新的Pod副本；如果Pod的数量多于指定的副本数，ReplicaSet将删除多余的Pod。



**9、什么是Deployment？**

答：Deployment是Kubernetes的控制器之一，用于声明性地管理Pod副本集。它允许定义Pod模板、副本数和更新策略，使得应用程序的部署和更新变得简单可控。



**10、什么是Service？**

答：Service是Kubernetes的抽象层，用于暴露应用程序的一组Pod。它为这些Pod提供稳定的网络终结点，并允许它们通过服务发现进行通信。



**11、什么是命名空间（Namespace）？**

答：命名空间是一种在Kubernetes集群中创建多个虚拟集群的机制。它可以用于隔离和管理不同的应用程序、团队或环境。



**12、如何进行应用程序的水平扩展？**

答：可以使用Deployment的副本数字段来进行水平扩展。通过增加副本数，Kubernetes会创建更多的Pod副本以应对负载增加。



**13、如何在Kubernetes中进行滚动更新（Rolling Update）？**

答：可以通过更新Deployment的Pod模板来进行滚动更新。Kubernetes会逐步替换现有的Pod副本，确保在整个更新过程中应用程序的可用性。



**14、如何在Kubernetes中进行滚动回滚（Rollback）？**

答：可以使用Deployment的回滚功能来进行滚动回滚。通过指定回滚到的特定修订版本或回滚到上一个修订版本，Kubernetes会自动恢复旧的Pod副本。



**15、什么是Kubernetes的水平自动扩展（Horizontal Pod Autoscaling）？**

答：水平自动扩展是Kubernetes的功能之一，根据应用程序的负载自动调整Pod副本数。它基于CPU利用率或自定义指标来进行自动扩展。



**16、如何进行存储卷（Volume）的使用？**

答：可以使用存储卷将持久化数据附加到Pod中。Kubernetes支持多种类型的存储卷，如空白存储卷、主机路径、持久卷等。



**17、什么是ConfigMap和Secret？**

答：ConfigMap用于存储应用程序的配置数据，而Secret用于存储敏感信息，如密码、API密钥等。它们可以作为环境变量、命令行参数或挂载到容器中使用。



**18、什么是亲和性（Affinity）和反亲和性（Anti-Affinity）？**

答：亲和性和反亲和性是Pod调度的约束条件。通过使用亲和性，可以将Pod调度到指定的节点；通过使用反亲和性，可以避免将Pod调度到指定的节点。



**19、什么是DaemonSet？**

答：DaemonSet是Kubernetes的控制器之一，用于在集群中的每个节点上运行一个Pod副本。它适用于在集群中的每个节点上运行系统级别的守护进程。



**20、什么是Ingress？**

答：Ingress是Kubernetes的资源之一，用于将外部流量路由到集群内的服务。它可以提供负载均衡、SSL终止、路径基于的路由等功能。



**21、什么是持久卷（Persistent Volume）和持久卷声明（Persistent Volume Claim）？**

答：持久卷是一种Kubernetes资源，用于提供独立于Pod的持久化存储。持久卷声明用于请求持久卷，使得Pod可以访问持久化存储。



**22、什么是Init容器（Init Container）？**

答：Init容器是Pod中的一个额外容器，用于在主应用程序容器启动之前运行初始化任务。它可以用于数据准备、配置下载等任务。



**23、如何在Kubernetes中进行配置文件的安全管理？**

答：可以使用Secret来安全地管理敏感配置信息，如数据库密码、API密钥等。可以通过加密、访问控制和密钥轮换等措施来确保Secret的安全性。



**24、如何监控Kubernetes集群？**

答：可以使用Kubernetes内置的指标和日志系统，如kube-state-metrics、Heapster和EFK堆栈，来监控集群的运行状态和性能。



**25、如何进行跨集群部署和管理？**

答：可以使用Kubernetes Federation或Kubernetes多集群（Multi-cluster）解决方案来进行跨集群部署和管理。



**25、什么是Kubernetes的生命周期钩子（Lifecycle Hook）？**

答：生命周期钩子是Pod中的回调函数，可以在容器的生命周期事件发生时触发。它们可以用于在容器启动、停止或失败时执行定制化操作。



**26、什么是Pod的探针（Probe）？**

答：Pod的探针用于定期检查容器的健康状态。Kubernetes支持三种类型的探针：存活探针（Liveness Probe）、就绪探针（Readiness Probe）和启动探针（Startup Probe）。



**27、什么是Kubernetes的安全性措施？**

答：Kubernetes提供了多种安全性措施，如访问控制、网络策略、身份验证和授权、安全上下文等。此外，还可以使用第三方工具和插件来增强Kubernetes的安全性。



**29、什么是容器资源限制（Resource Limit）和容器资源请求（Resource Request）？**

答：容器资源限制用于限制容器使用的CPU和内存资源。容器资源请求用于向调度器声明容器所需的CPU和内存资源。



**30、什么是Kubernetes中的水平和垂直扩展？**

答：水平扩展（Horizontal Scaling）指的是增加Pod副本数来处理更多的负载。垂直扩展（Vertical Scaling）指的是增加或减少单个Pod的资源限制。



**31、什么是Kubernetes的节点亲和性（Node Affinity）？**

答：节点亲和性用于将Pod调度到具有特定标签或节点选择器匹配的节点上。它可以用于确保Pod运行在特定类型的节点上，如SSD存储节点或GPU节点。



**32、什么是Kubernetes的事件（Event）？**

答：事件是Kubernetes集群中发生的重要操作或状态更改的记录。可以使用kubectl命令或API查看集群中的事件。



**33、什么是Helm？**

答：Helm是Kubernetes的包管理工具，用于简化应用程序的部署和管理。它允许定义和版本化应用程序的Charts（图表），并通过Helm命令进行安装、升级和删除。



**34、如何进行Kubernetes集群的高可用性配置？**

答：可以使用Kubernetes的Master节点高可用性（HA）模式，通过配置多个Master节点实现集群的高可用性。这可以通过使用负载均衡器、备份ETCD数据存储等方法来实现。



**35、什么是Kubernetes的状态管理器（StatefulSet）？**

答：StatefulSet是Kubernetes的控制器之一，用于管理有状态应用程序的部署。它确保Pod的稳定网络标识和有序启动、停止，适用于数据库、队列等有状态应用程序。



**36、什么是Kubernetes的自定义资源定义（Custom Resource Definition，CRD）？**

答：CRD允许用户将自定义资源（Custom Resources）引入到Kubernetes中。这使得用户可以扩展Kubernetes的API和控制器，以支持自定义的资源类型。



**37、什么是Kubernetes的配置管理工具？**

答：Kubernetes提供了多种配置管理工具，如kubectl、kubeconfig文件、ConfigMap、Secret、Helm等。这些工具可以用于管理和传递应用程序的配置信息。



**38、什么是Kubernetes的网络模型？**

答：Kubernetes的网络模型基于容器间和容器与外部的通信。每个Pod都具有唯一的IP地址，并且可以通过服务和Ingress来实现内部和外部的网络通信。



**39、什么是Kubernetes的升级策略？**

答：Kubernetes的升级策略指定了如何处理应用程序的升级。它包括滚动更新、蓝绿部署、金丝雀发布等不同的升级方式。



**40、什么是Kubernetes的监控和日志记录解决方案？**

答：Kubernetes提供了多种监控和日志记录解决方案，如Prometheus、Grafana、ELK堆栈等。这些工具可以用于监控集群的性能指标和应用程序日志。



**41、请解释一下 Kubernetes 的主要组件。**

答：Kubernetes的主要组件包括：Master组件（API Server、Controller Manager、Scheduler）和Node组件（kubelet、kube-proxy、容器运行时）。



**42、如何在 Kubernetes 中扩展应用程序？**

答：可以使用ReplicaSet、Deployment或Horizontal Pod Autoscaler（HPA）来扩展应用程序。



**43、怎样从一个镜像创建一个 Pod？**

答：可以使用kubectl命令行工具或编写一个Pod的YAML文件，然后使用kubectl apply命令创建Pod。



**44、如何将应用程序部署到 Kubernetes？**

答：可以使用Deployment、StatefulSet或DaemonSet等资源对象来部署应用程序。



**45、如何水平扩展 Deployment？**

答：可以通过更改Deployment的副本数来水平扩展应用程序。例如，使用kubectl scale命令或更改Deployment的replicas字段。



**46、怎样在 Kubernetes 中进行服务发现？**

可以使用Kubernetes的Service对象来进行服务发现。Pod可以通过Service的DNS名称进行通信。



**47、如何进行滚动更新（Rolling Update）？**

答：可以通过更新Deployment的Pod模板或修改RollingUpdate策略来执行滚动更新。



**48、什么是 PVC（Persistent Volume Claim）？**

答：PVC是用于声明对持久卷（Persistent Volume）的需求的对象，它允许Pod使用持久化存储。



**49、怎样进行容器间通信？**

答：可以使用Pod的内部IP地址和端口号进行容器间通信。此外，也可以使用Service对象来提供稳定的网络访问。



**50、如何进行集群内部的日志收集？**

答：可以使用Kubernetes的日志收集器（如Fluentd、Prometheus）来收集集群中各个Pod的日志。



**51、什么是亲和性调度（Affinity Scheduling）？**

答：亲和性调度是一种机制，用于将Pod调度到特定的节点或一组节点，以便满足特定的调度策略。



**52、怎样进行水平自动扩展（Horizontal Autoscaling）？**

答：可以使用Horizontal Pod Autoscaler（HPA）对象来自动根据CPU或其他指标水平扩展Pod副本的数量。



**53、如何进行安全访问控制（RBAC）？**

答：可以使用Kubernetes的Role-Based Access Control（RBAC）机制来定义和管理用户对集群资源的访问权限。



**54、什么是 Sidecar 容器？**

答：Sidecar容器是与主应用程序容器共同运行的辅助容器，用于提供额外的功能或服务。



**55、怎样进行跨集群部署？**

答：可以使用Kubernetes的Federation机制来管理跨多个集群的应用程序部署和资源。



**56、如何进行存储卷的扩展和快照？**

答：可以使用Kubernetes的存储类（StorageClass）和持久卷声明（Persistent Volume Claim）来管理存储卷的扩展和快照。



**57、什么是 Pod 的生命周期？**

答：Pod的生命周期包括Pending、Running、Succeeded、Failed和Unknown等阶段。



**58、怎样进行热更新（Hot Deployment）？**

答：可以使用滚动更新策略，将新版本的应用程序容器逐步替换旧版本的容器，实现热更新。



**59、什么是 Downward API？**

答：Downward API是一种机制，用于将Pod的元数据（如标签、注解、环境变量等）注入到容器中。



**60、怎样进行资源限制和配额管理？**

答：可以使用Kubernetes的资源限制（Resource Limit）和配额（Resource Quota）来控制Pod使用的资源。



**61、什么是 CNI（Container Network Interface）？**

答：CNI是一种规范，用于定义容器运行时和网络插件之间的接口，以实现容器网络的配置和管理。