# Spacemesh 集群P盘教程

本教程基于`Ubuntu22.04`，`go1.20 linux/amd64`，目前仅支持Ubuntu环境。

## 1 编译代码
### 1.1 编译准备
```sh
sudo apt update && sudo apt install -y git git-lfs make curl build-essential unzip wget ocl-icd-opencl-dev unzip libudev-dev
```
注意go版本需要大于1.19。

### 1.2 编译显卡P盘程序 - postcli
首先编译显卡P盘脚本[postcli](https://github.com/spacemeshos/post/tree/develop/cmd/postcli)。
注意要根据最新的tag拉取代码，`git clone`默认分支的不一定是稳定版本的代码。
例如：拉取tag为`v0.8.9`的代码。
```sh
git clone https://github.com/spacemeshos/post.git && cd post
git reset --hard v0.8.9
make postcli
```
编译好的二进制文件位于`post/build`目录下。

### 1.3 编译扫盘/节点程序 - go-spacemesh
```sh
git clone https://github.com/spacemeshos/go-spacemesh.git && cd go-spacemesh
git reset --hard v1.0.6
make build
```
编译好的二进制文件位于`go-spacemesh/build`目录下。

### 1.4 编译钱包命令行 - smcli
```sh
git clone https://github.com/spacemeshos/smcli.git && cd smcli
git reset --hard v1.0.10
make build
```
smcli也可以直接去官方下载编译好的：https://github.com/spacemeshos/smcli/releases。

## 2 集群P盘流程
这里主要分享多机器集群P盘的详细操作流程，主要包括以下步骤：

- 使用`smcli`创建钱包并备份助记词和钱包密码；
- 使用编译好的go-spacemesh进行初始化；
- 使用postcli的 `-printNumFiles` 计算生成的`.bin`文件数量；
- 计算分段索引，将文件分段并分发给多台机器；
- 按照计算好的分段索引在每台机器上启动P盘；
- 等所有机器P盘完成后，将生成的文件合并到运行go-spacemesh的机器上；
- 重新启动go-spacemesh，开始扫盘并生成证明。

### 2.1 创建钱包
如果已经有钱包地址，比如通过smapp创建的钱包，可跳过本节，直接开始2.2初始化。

使用上面编译好的smcli来创建钱包，注意要备份好助记词和钱包密码。
```sh
./smcli wallet create
```
钱包创建完毕后，会输出到一个`/path/to/wallet_2023-07-21T07-17-52.946Z.json`文件中。

下一步执行导入钱包，并输入创建时的密码，就能看到刚才创建的钱包地址（格式为：sm1xxxxxxxxx）。
```sh
./smcli wallet read /path/to/wallet_2023-07-21T07-17-52.946Z.json
```

### 2.2 初始化
先用编译好的`go-spacemesh`初始化：
```sh
./go-spacemesh --config config.mainnet.json --smeshing-start --smeshing-coinbase sm1xxxxxxx --smeshing-opts-numunits 5 --smeshing-opts-provider 0 --smeshing-opts-datadir /mnt/spacemesh/post_data --data-folder /mnt/spacemesh/node_data
```
该命令会启动一个Spacemesh节点，并同时启动P盘，参数说明：
- `--config`: 节点配置文件，通过`wget https://smapp.spacemesh.network/config.mainnet.json`获取；
- `--smeshing-start`: 启动P盘，如果不加此选项，则只同步节点，不会启动P盘；
- `--smeshing-coinbase`: 收益地址，`sm1`开头，通过`smcli`创建；
- `--smeshing-opts-numunits`: P盘的单元数量，和下面postcli中的`-numUnits`必须相等；
- `--smeshing-opts-provider`: 指定用显卡还是CPU，P盘时用显卡，参数值为显卡ID`0、1、2...`，扫盘时用CPU，参数值为`4294967295`；
- `--smeshing-opts-datadir`: P盘文件存储路径；
- `--data-folder`: 节点文件存储路径；

本文主要介绍多台机器如何搭建集群P盘，如果只有一台机器(即单机solo模式)，上面的命令运行以后，等待P盘完成即可。

如果要多台机器在同一个节点上同时P盘，启动上面的初始化命令等开始生成`.bin`文件后，就可以先退出了。因为我们要使用postcli来多台机器同时P盘，`go-spacemesh`只负责同步区块节点、扫盘及提交证明。

如果退出时，`--smeshing-opts-datadir`目录下已经生成了`postdata_0.bin`文件，建议删除。

初始化完成后，将在`--smeshing-opts-datadir`生成2个文件：
- `key.bin`: 节点私钥文件，其中保存了初始化以后的节点私钥；
- `postdata_metadata.json`: 节点元数据文件，其中包含了`NodeId`和`CommitmentAtxId`，这2个值都是下来P盘需要用到的。

### 2.3 计算P盘文件数
SpaceMesh是以`numUnits`为基本的存储单元，每个`numUnits = 64GB`，P好的文件是`postdata_xxx.bin`格式的文件，文件大小取决于postcli启动时`-maxFileSize`参数指定的文件大小，默认是4G。

通过postcli的 `-printNumFiles` 来计算最终生成多少bin文件。
```sh
./postcli -numUnits 5 -printNumFiles
80
```
输出结果为80，即共生成80个.bin文件，每个文件大小为4G，共320G。

如果要指定P盘文件大小，则需增加`-maxFileSize`参数，如P 8GB的文件，输出结果为40：
```sh
./postcli -numUnits 5 -maxFileSize=8589934592 -printNumFiles
40
```

### 2.4 计算分段索引
针对多台机器基于同一个NodeID P盘的情况，postcli提供了分段P盘`subset`功能，更多信息也可参照[postcli subset文档](https://github.com/spacemeshos/post/tree/develop/cmd/postcli#initializing-a-subset-of-post-data)。

Subset是把要P的文件分段并分发给多台机器来跑，通过`-fromFile`，`-toFile`来设置开始及结束的文件索引。

例如，上面的`-numUnits=5`总共需要生成80个.bin文件，如果平分给4台机器，则每台机器的`-fromFile`和`-toFile`分别为：
机器| -fromFile | - toFile
------|-------:|------:
机器1 | 0  | 19
机器2 | 20 | 39
机器3 | 40 | 59
机器4 | 60 | 79

启动命令分别为：

- 机器1 (0 - 19)
```sh
./postcli -provider=0 -commitmentAtxId=[cat postdata_metadata.json | jq -r '.CommitmentAtxId' |  base64 -d | xxd -p -c 32] -id=[cat postdata_metadata.json | jq -r '.NodeId' |  base64 -d | xxd -p -c 32] -numUnits=5 -fromFile=0 -toFile=19 -datadir=/mnt/spacemesh/post_data
```

- 机器2 (20 - 39)
```sh
./postcli -provider=0 -commitmentAtxId=[cat postdata_metadata.json | jq -r '.CommitmentAtxId' |  base64 -d | xxd -p -c 32] -id=[cat postdata_metadata.json | jq -r '.NodeId' |  base64 -d | xxd -p -c 32] -numUnits=5 -fromFile=20 -toFile=39 -datadir=/mnt/spacemesh/post_data
```

- 机器3 (40 - 59)
```sh
./postcli -provider=0 -commitmentAtxId=[cat postdata_metadata.json | jq -r '.CommitmentAtxId' |  base64 -d | xxd -p -c 32] -id=[cat postdata_metadata.json | jq -r '.NodeId' |  base64 -d | xxd -p -c 32] -numUnits=5 -fromFile=40 -toFile=59 -datadir=/mnt/spacemesh/post_data
```

- 机器4 (60 - 79)
```sh
./postcli -provider=0 -commitmentAtxId=[cat postdata_metadata.json | jq -r '.CommitmentAtxId' |  base64 -d | xxd -p -c 32] -id=[cat postdata_metadata.json | jq -r '.NodeId' |  base64 -d | xxd -p -c 32] -numUnits=5 -fromFile=60 -toFile=79 -datadir=/mnt/spacemesh/post_data
```
**⚠️ 注意将`[cat postdata_metadata.json | jq -r '.CommitmentAtxId' |  base64 -d | xxd -p -c 32]`和`[cat postdata_metadata.json | jq -r '.NodeId' |  base64 -d | xxd -p -c 32]`替换为实际运行该命令的输出结果**。

### 2.5 启动P盘
根据计算好的分段索引，在subset的每台机器上，按照分段索引启动，启动参数说明：
- `-provider` 指定P盘的显卡ID，0、1、2，默认为0，如果有多张显卡，建议启动多个进程分别指定`-provider`；
- `-commitmentAtxId` 提交PoET证明的地址，通过以下命令获取：  
  `cat postdata_metadata.json | jq -r '.CommitmentAtxId' |  base64 -d | xxd -p -c 32`

- `-id` 节点ID(NodeID)，通过以下命令获取：  
  `cat postdata_metadata.json | jq -r '.NodeId' |  base64 -d | xxd -p -c 32`

- `-numUnits` P盘文件单元数，和节点初始化时候的`--smeshing-opts-numunits`保持一致；
- `-datadir` P盘完成后的文件保存目录。

### 2.6 合并P盘文件
等所有subset的机器P盘完成后，需要将每台机器上生成的文件，合并到运行`go-spacemesh`服务机器的`--smeshing-opts-datadir`路径下，然后重新启动`go-spacemesh`就可以开始扫盘并生成证明了。

**注意**`.bin`文件合并后，需要在每个运行postcli机器的`postdata_metadata.json`文件里，找到一个全局最小的nonce，作为`go-spacemesh`的`--smeshing-opts-datadir`路径下`postdata_metadata.json`的nonce （有的postcli进程可能找不到nonce）。

启动后如果文件完整，扫码文件时会输出类似下面的日志：

```sh
2023-07-23T15:57:24.849+0800	INFO	fb26a.post	initialization: file already initialized	{"node_id": "fb26a9d2da5626ded24027da14054bf0fbf8886bd7ec4a29d05ee2fdd44edddd", "module": "post", "fileIndex": 0, "currentNumLabels": 268435456, "targetNumLabels": 268435456, "startPosition": 0}
2023-07-23T15:57:24.849+0800	INFO	fb26a.post	initialization: file already initialized	{"node_id": "fb26a9d2da5626ded24027da14054bf0fbf8886bd7ec4a29d05ee2fdd44edddd", "module": "post", "fileIndex": 1, "currentNumLabels": 268435456, "targetNumLabels": 268435456, "startPosition": 268435456}
...
```

### 2.7 扫盘并提交证明
文件扫描完毕后，开始读取文件并生成证明，生成证明开始的日志类似：
```sh
2023-07-23T15:57:59.995+0800	INFO	fb26a.post calculating proof of work for nonces 0..144
```

扫盘持续时间根据机器配置和磁盘速率而定，可通过每隔5秒钟输出磁盘I/O：
`iostat -dmt /dev/md0 5`

据我个人测试观察，整个扫盘并生成证明的过程，在刚开始会有10-20分钟的准备时间(20T数据)，然后开始有磁盘I/O。

**扫盘时间 = (总容量 / 磁盘读速率) + 30分钟的准备时间**

可根据自己的磁盘速率，估算扫盘及生成证明的时间。

比如磁盘读写为2G/s，那么20T数据的扫盘+证明时间大致为：`(20 * 1024) / 2 /3600`约为2.8小时，加上准备时间，大约3小时。

扫盘完成后，会输出类似下面的日志：
```sh
2023-07-23T16:07:27.469+0800	INFO	fb26a.post	Found proof for nonce: 110, pow: 54043195528453627 with...
```

下来是生成证明及初始化过程，完成后将在`--smeshing-opts-datadir`目录下生成`post.bin`文件。

等停留在下面内容时，就表明整个过程已完成，耐心等待注册即可。
```sh
2023-07-23T16:07:27.736+0800	INFO	fb26a.atxBuilder	building new atx challenge
```

## 3 常用命令
- 计算P盘文件数
```sh
./postcli -numUnits 5 -printNumFiles
```

- Get commitmentAtxId
```sh
cat postdata_metadata.json | jq -r '.CommitmentAtxId' |  base64 -d | xxd -p -c 32
```

- Get NodeId
```sh
cat postdata_metadata.json | jq -r '.NodeId' |  base64 -d | xxd -p -c 32

grpcurl -plaintext localhost:9092 spacemesh.v1.ActivationService.Highest | jq -r '.atx.id.id' |  base64 -d | xxd -p -c 32
```

- 查看节点状态
```sh
grpcurl -plaintext localhost:9092 spacemesh.v1.NodeService.Status
```

持续更新中...

## 4 常见问题

### 4.1 P盘速度怎样的？
在NVMe U.2 * 5 raid0上的显卡的P盘速度:

显卡       | 文件大小 | 用时
-----------|-------:|------:
RTX 3090   | 4G | 19分钟
RTX 3080   | 4G | 23分钟
RTX 2080Ti | 4G | 32分钟

欢迎大家持续补充其他显卡的速度。

### 4.2 多个Node能用一个`--smeshing-coinbase`钱包地址吗？
可以的，只要是有效的钱包地址都可以。

### 4.3 用postcli已经P好的文件，可以移走吗？
不要移走，会影响nonce的寻找，等一批跑完后方可移走。

<!-- 一台p好的数据，怎么加载到新电脑上，还是那个账号 -->

持续更新中...

## 5 资源链接
- [go-spacemesh](https://github.com/spacemeshos/go-spacemesh)
- [post-cli](https://github.com/spacemeshos/post/blob/develop/cmd/postcli/README.md)
- [post-rs](https://github.com/spacemeshos/post-rs.git)
- [gpu post](https://github.com/spacemeshos/gpu-post)
- [区块浏览器](https://explorer.spacemesh.io/overview)
- [钱包命令行工具](https://github.com/spacemeshos/smcli)
- [经济模型](https://spacemesh.io/blog/spacemesh-economics-intro)
- [奖励及发放](https://spacemesh.io/start/)
- [奖励释放测算](https://docs.google.com/spreadsheets/d/1apyWCnf5wXzFik4BGvNckLi8lCAkRSbgsaWJoY_qCng/edit#gid=0)
- [创世时间线](https://spacemesh.io/blog/genesis-timeline/)
- [显卡P盘脚本](https://github.com/spacemeshos/post/tree/develop/cmd/postcli)
- [多显卡P盘教程](https://simeononsecurity.ch/other/efficient-spacemesh-mining-multiple-gpus-guide/#linux)
- [多线程P盘脚本](https://github.com/fourierism/post)

## 6 加入社群
MinerDAO社区聚集了Filecoin, Aleo, Spacemesh等当前热门挖矿项目的矿工、开发者、投资人。  
我们为矿工和开发者提供技术交流、算法优化、资源合作、新项目研究等，欢迎大家加入讨论。

- 微信号: maxvint (备注: MinerDAO)  

  <img src="https://raw.githubusercontent.com/minerdao/posts/master/images/wechat-max.png?20230727" width="200">

- [Telegram交流群](https://t.me/joinchat/TOGYnsZ2itA0NGZl)
