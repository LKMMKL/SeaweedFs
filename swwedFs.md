**主要对官方文档大部分进行了翻译，并加入了自己的了解，可以与[官方文档](https://github.com/igool/seaweedfs)结构学习**

**读该文档时，可先阅读文档底部的Seaweed拓扑结构，有利于了解该文件系统**



[SeaweedFs 拓扑结构](#jump)



## 说明

SeaweedFs 是一个简单的，高扩展的分布式文件系统。它有两个特点：

​     1.存储亿万级的文件

​     2.快速提供文件

SeaweedFs 选择实现键值映射，而不是支持全POSIX语义的文件系统。类似与“NOSQL",也可以称为”NOFS“。  

SeaweedFs 选择管理中央主文件中的文件卷`卷是硬盘上的存储区域`,并且使用卷服务管理文件和元数据，而不是直接使用中央主文件管理所有元数据。它减轻了来自中央主文件的并发压力并把文件元数据传输到卷服务器的内存中，只需要一次磁盘读取操作即可实现实现更快的文件访问！(所以。。为什么只需要一次)

SeaweedFs 只需要40字节的磁盘存储空间来存储一个文件元数据。它的查找非常简单，复杂度为O(1),欢迎你根据自身使用情况来挑战它的性能。



## 附加功能

* 可以选择不复制或者不同的复制功能，支持机架和数据中心。

* 主服务器故障自动转移，没有单点故障问题。

* 自动压缩取决于文件扩展类型。<!-- MIME: 文件的扩展类型 -->

* 自动压缩并回收删除或更新后的磁盘空间。

* 同一集群中的服务器可以有不同的磁盘空间，不同的文件系统，不同的操作系统。

* 添加/删除服务器不会导致任何数据进行重新平衡的操作。

* 可选的Filer服务器通过http提供"普通"的目录和文件。

* 对于jpeg类型的图片，可以选择固定方向。

* 支持Etag，Accept-Range，Last-Modified等。   

  ​

## 使用示例

默认情况下，主节点运行再9333端口上，卷节点运行再8080端口上。现在我开启1个主节点和两个卷节点，分别运行再8080和8081端口上。理想情况下，它们应该运行在不同的机器上，这里我只使用本机做一个例子。

SeaweedFs 使用HTTP REST操作读，写，删除行为。返回结果是JSON或者JSONP格式。

### 开启主服务器

~~~
> weed master
~~~

### 开启卷服务器

~~~
> weed volume -dir="/tmp/data1" -max=5  -mserver="localhost:9333" -port=8080 &
> weed volume -dir="/tmp/data2" -max=10 -mserver="localhost:9333" -port=8081 &
~~~

**-max 表示卷服务器中最多可以存在的卷数量，一个dat文件代表一个卷。**



### 写文件

上传文件。首先要发送一个HTTP POST，PUT或者GET请求到 `/dir/assign` ,获取fid和卷服务器地址。

~~~
> curl -X POST http://localhost:9333/dir/assign
{"fid":"5,017f524ccc","url":"127.0.0.1:8082","publicUrl":"127.0.0.1:8082","count":1}
~~~

然后，利用返回的结果进行存储文件。发送一个HTTP PUT 或 POST请求到 `url + '/' + fid`。

~~~
curl -X PUT -F file=@C:\Users\lkm\Desktop\untitled.jpg http://127.0.0.1:8082/5,017f524ccc
{"name":"untitled.jpg","size":54584}
~~~



可以直接使用submit,则不需要再提前获取fid,它会先进行存储，再返回fid。

```
curl -F file=@C:\Users\lkm\Desktop\untitled.jpg http://127.0.0.1:9333/submit
{"fid":"6,02ee9e1d62","fileName":"untitled.jpg","fileUrl":"127.0.0.1:8082/6,02ee9e1d62","size":54584}
```

#### collection

启动 volumeServer

~~~
weed volume -dir="D:\ZhongYe\seaweedfs\seaweedfs\2" -max=5 -mserver="localhost:9333" -port=8083
~~~

分配fid，**它会将所有空余卷服务器的卷都整合成一个collection。**

~~~
curl -X POST http://localhost:9333/dir/assign?collection=pictures
{"fid":"10,039b0b8e05","url":"127.0.0.1:8083","publicUrl":"127.0.0.1:8083","count":1}
~~~

~~~
curl -X POST http://localhost:9333/dir/assign?collection=map
{"error":"No free volumes left!"}
~~~

~~~
curl -X POST http://localhost:9333/dir/assign?collection=pictures
{"fid":"7,042c5762d4","url":"127.0.0.1:8083","publicUrl":"127.0.0.1:8083","count":1}
~~~

1. 因为默认情况下，VolumeServer 启动时， 未申请任何 Volume，当第一次 `/dir/assign `的时候， 会分配 Volume，因为`weed volume `的参数 `-max=5 `所以一次性分配 5 个 Volume ，并且这 5 个 Volume 的 Collection 属性都是 pictures， 甚至可以看到卷服务器下：

​      ![](D:\ZhongYe\seaweedfs\seaweedfs\oll.png)

可以看出每个卷的文件名以 Collection 来命名。

2. 因为已有的 5 个 Volume 的 Collection 属性都是 pictures， 所以此时如果需要 `/dir/assign `一个非 pictures Collection 的 Fid 时失败，如果有非coolection的volume存在，则成功。
3. 当申请一个属于 pictures Collection 的 Fid 成功。也就是在每次申请 Fid 时，会针对 Collection 进行检查，来保证存入 Volume 的每个 Needle 所属的 Collection 一致。 在实际应用中可以通过 Collection 来类别的片。

---



更新文件，只需要再重复发送一次请求



查看文件。

~~~
http://127.0.0.1:8082/5,017f524ccc
~~~



删除文件。发送一个HTTP DELETE请求，带上fid参数

~~~
>curl -X DELETE http://127.0.0.1:8082/5,017f524ccc
~~~



### 保存 fid

现在，我们可以把上面得到的fid 5,017f524ccc，保存到数据库当中。

~~~
fid 5,017f524ccc 可以分成三部分
volumeId: 5 ，32bit,卷id,卷的数量很多，具体存到哪个卷服务器的哪个卷都是随机的。
NeedleId: 01 ，64bit,文件标识，全局唯一NeedleId，每个存储的文件都不一样(除了互为备份的)。
Cookie: 9d90e98a ，32bit,Cookie值，为了安全起见，防止恶意攻击。
~~~

你可以选择使用自己的格式如 <volume id, file key, file cookie>存储fid,也可以直接使用String存储fid。

理论上，如果使用String存储，需要 8+1+16+8=33个字节，使用char(33)就足够了。

如果空间真的是一个问题，您可以以自己的格式存储文件ID。 您需要一个4字节的整数用于卷ID，8字节长的数字用于文件密钥，4字节的整数用于文件Cookie。 所以16个字节就足够了（绰绰有余）。



### 读取文件

首先，根据volumeId查找卷服务器的URL:

~~~
curl http://localhost:9333/dir/lookup?volumeId=5
{"volumeId":"5","locations":[{"url":"127.0.0.1:8082","publicUrl":"127.0.0.1:8082"}]}
~~~

然而，通常情况下，卷服务器的数量不会很多，你可以对结果进行缓存。文件读取依赖于复制类型，一个卷有多个副本，随机进行读取。

~~~
http://127.0.0.1:8082/5,017f524ccc
~~~

注意，读取文件的url是可选的，上面只是其中一种方式。

如果你想用一种更容易理解的方式，以下方式也可以

~~~
http://127.0.0.1:8082/5,017f524ccc.jpg
http://127.0.0.1:8082/5/017f524ccc/lkm.jpg
http://127.0.0.1:8082/5/017f524ccc
~~~

如果你想要获取一定大小的图片，你可以带上参数

~~~
http://127.0.0.1:8082/5,017f524ccc？width=100&height=100
~~~

### 机架和数据中心

SeaweedFs 在卷的层次上应用复制策略，你可以在获取fid时指定复制策略： 

~~~
curl -X POST http://localhost:9333/dir/assign?replication=001
~~~

以下是复制策略的参数

~~~
000 :  不复制
001 ： 在相同的机架复制一次
010 ： 在不同的机架上复制一次，使用相同的数据中心
100 ： 在不同的数据中心复制
200 :  在两个不同的数据中心复制两次
110 ： 在不同的机架复制一次，在不同的数据中心复制一次
~~~

更多细节可以查看[WIKI](https://github.com/chrislusf/seaweedfs/wiki/Replication)。

### 分配具体的数据中心

可以在卷服务器启动时，指定数据中心：

~~~
weed volume -dir=/tmp/1 -port=8080 -dataCenter=dc1
weed volume -dir=/tmp/2 -port=8081 -dataCenter=dc2
~~~

主服务可以通过卷服务器的IP地址和weed.conf文件指定数据中心。

获取file key时也可以限制在某个数据中心

~~~
http://localhost:9333/dir/assign?dataCenter=dc1
~~~



## 结构

​       通常，分布式文件系统将每个文件拆分为块，并且中央主服务器将文件名和块索引映射到块句柄，以及每个块服务器具有哪些块。

​       这有一个缺点，即中央主服务器无法高效地处理许多小文件，并且由于所有读请求都需要通过块主服务器，所以许多并发Web用户的响应速度会很慢。

​        SeaweedFS选择管理主服务器中的数据卷，而不是管理组块。每个数据卷大小为32GB，并且可以容纳大量文件。每个存储节点可以有很多数据卷。所以主节点只需要存储关于卷的元数据，这是相当少量的数据，大部分时间是相当静态的。

​        实际的文件元数据存储在卷服务器上的每个卷中。由于每个卷服务器只管理自己磁盘上文件的元数据，每个文件只有16个字节，因此所有文件访问都可以从内存中读取文件元数据，只需要一次磁盘操作即可实际读取文件数据。

​        为了比较，请考虑Linux中的xfs inode结构是536字节。



## 主服务器和卷服务器

该架构非常简单。数据存储在存储节点的卷上。一个卷服务器管理众多卷，且都支持基本访问。

主服务器包含卷Id到卷服务器的映射。这相当于静态信息，可以被缓存。

在每个写入请求上，主服务器还会生成一个文件密钥，这是一个不断增长的64位无符号整数。 由于写入请求不像读取请求那么繁忙，所以一个主服务器应该能够很好地处理并发。



## 写和读取文件

读取文件可以从卷中读取，也可以从缓存中读取。



## 存储规格

默认情况下，每个卷大小最多是32G,可以通过修改代码将容量修改为64/128G。

卷的数量是2^32。所以整个文件系统大小为32G*2^32=128EiB。



## 节省内存



------

接下来的是各个文件系统的比较分析，由于没有接触过，就暂不翻译了。

------





## <div id="jump">SeaweedFs 拓扑结构</div>

以下是SweedFs的拓扑结构，有助于理解该文件系统的运行

**SeaweedFs 整体拓扑结构**

![SweedFs 整体拓扑结构](https://github.com/LKMMKL/SeaweedFs/blob/master/sweed3.PNG)



​    master管理多个volume服务器，每个volume服务器管理多个volume（volume是一个个磁盘分区,也可以称为一个存储节点）,每个volume由一个superBlock（磁盘分区头部）和若干个Needle组成，每个Needle则代表一个小文件，例如图片和短视频，因此上传的每个文件都有不一样的NeedleId。

​    分布式文件系统数据节点存储数据时，会创建出若干个大文件(volume)，用于存储小文件，例如文件，短视频等等；在seaweedfs中，大文件就称为Volume。



**master 拓扑**

![](https://github.com/LKMMKL/SeaweedFs/blob/master/sweed5.JPG)  

​     **每个 MasterServer 通过 Topology 维护多个 VolumeServer 。**

​      seaweedfs拓扑结构主要有三个概念，数据中心(DataCenter)，机架(Rack)，数据节点(DataNode)；这样可以很灵活配置不同数据中心，同一个数据中心下不同机架，同一机架下不同的数据节点；数据都是存储在DataNode中。

------

具体流程：

当客户端需要存储数据时

1. 需要先给M-Master发送请求，获取该文件存储的DataNode地址，文件存储的VolumeID以及文件fid，从fid中找到NeedleId。
2. 然后客户端接着将文件发送至从Master获取到的DataNode，DataNode会根据VolumeID找到相应的Volume，再找到对应的Needle,将小文件存到该Needle。

------



## 集群

首先开启三个master

~~~
mater1:启动master1后，由于只有这一个master，因此将自己选为leader,master2、master3启动后，加入到以master1为leader的 master集群。
weed -v=3 master -port=9333 -mdir=D:\ZhongYe\seaweedfs\seaweedfs\mdata -  peers=localhost:9333,localhost:9334,localhost:9335 -defaultReplication=100

master2:
weed -v=3 master -port=9334 -mdir=D:\ZhongYe\seaweedfs\seaweedfs\mdata2 -peers=localhost:9333,localhost:9334,localhost:9335 -defaultReplication=100

master3:
weed -v=3 master -port=9335 -mdir=D:\ZhongYe\seaweedfs\seaweedfs\mdata3 -peers=localhost:9333,localhost:9334,localhost:9335 -defaultReplication=100
~~~

再开启三个volumeServer,设置不同的dataCenter

~~~
weed volume -port=8081 -dir=D:\ZhongYe\seaweedfs\seaweedfs\1 -mserver=localhost:9333 -dataCenter=dc1

weed volume -port=8081 -dir=D:\ZhongYe\seaweedfs\seaweedfs\2 -mserver=localhost:9333 -dataCenter=dc1

weed volume -port=8081 -dir=D:\ZhongYe\seaweedfs\seaweedfs\3 -mserver=localhost:9333 -dataCenter=dc2
~~~

这个时候卷里面还没有dat , idx文件

存储文件

~~~
curl -F file=@C:\Users\lkm\Desktop\untitled.jpg http://localhost:9333/submit
{"fid":"11,053fdc0fa6","fileName":"untitled.jpg","fileUrl":"127.0.0.1:8082/11,053fdc0fa6","size":54584}

文件存储在id为11的volume中，文件夹2和文件夹3中都有11.dat,因此文件在这两个文件夹中都有存储。
~~~

读取文件

~~~
//使用8082 和 8083 都可以访问到文件
curl http://127.0.0.1:8082/11,053fdc0fa6
curl http://127.0.0.1:8083/11,053fdc0fa6

//如果使用8081，得不到文件内容，只有一个链接
curl http://127.0.0.1:8081/11,053fdc0fa6
<a href="http://127.0.0.1:8082/11,053fdc0fa6">Moved Permanently</a>

//可以使用 -L 进行重定向，这样可以访问到文件
curl -L http://127.0.0.1:8081/11,053fdc0fa6

//如果使用master进行读取，master会轮询式返回8082和8083，这里同样可以使用重定向
curl http://127.0.0.1:9333/11,053fdc0fa6
<a href="http://127.0.0.1:8082/11,053fdc0fa6">Moved Permanently</a>.

curl http://127.0.0.1:9333/11,053fdc0fa6
<a href="http://127.0.0.1:8083/11,053fdc0fa6">Moved Permanently</a>.
~~~

删除文件

~~~
//文件夹2和文件夹3都进行删除，无论使用volume还是master都无法访问文件
curl -X DELETE http://127.0.0.1:8082/11,053fdc0fa6
{"size":54584}

//可以查询文件详细信息
curl -i  http://127.0.0.1:8082/11,053fdc0fa6
HTTP/1.1 404 Not Found
Date: Thu, 20 Aug 2015 08:13:28 GMT
Content-Length: 0
Content-Type: text/plain; charset=utf-8
~~~

手工清除：

删除hello.txt后，volume server下的数据文件的size却并没有随之减小，别担心，这就是weed-fs的处理方法，这些数据删除后遗留下来的空洞需要手工清除（对数据文件 进行手工紧缩）

~~~
curl "http://localhost:9335/vol/vacuum"
{"Topology":{"DataCenters":[{"Free":8,"Id":"dc1","Max":14,"Racks":[{"DataNodes":[{"Free":4,"Max":7,"PublicUrl":"127.0.0.1:8081","Url":"127.0.0.1:8081","Volumes":3},{"Free":4,"Max":7,"PublicUrl":"127.0.0.1:8082","Url":"127.0.0.1:8082","Volumes":3}],”Free”:8,”Id”:”DefaultRack”,”Max”:14}]},{“Free”:1,”Id”:”dc2″,”Max”:7,”Racks”:[{"DataNodes":[{"Free":1,"Max":7,"PublicUrl":"127.0.0.1:8083","Url":"127.0.0.1:8083","Volumes":6}],”Free”:1,”Id”:”DefaultRack”,”Max”:7}]}],”Free”:9,”Max”:21,”layouts”:[{"collection":"","replication":"100","ttl":"","writables":[1,2,3,4,5,6]}]},"Version":"0.70 beta"}

查看dat文件，size减小了。
~~~



**一致性**

在分布式系统中，“一致性”是永恒的难题。weed-fs支持replication，其多副本的数据一致性需要保证。

weed-fs理论上采用了是一种“强一致性”的策略，即：

存储文件时，当多个副本都存储成功后，才会返回成功；任何一个副本存储失败，此次存储操作则返回失败。

删除文件时，当所有副本都删除成功后，才返回成功；任何一个副本删除失败，则此次删除操作返回失败。

