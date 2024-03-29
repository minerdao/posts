# Aleo Testnet3 如何参与

## 1 几个问题
#### 1.1 真的开始了吗？
目前Aleo三测2阶段软启动上线，但是没有奖励积分。官方放出来一个CPU版本的代码，让大家先测试。不过目前的版本稍加改动，就可以用显卡来跑，只是效率不高。

[MinerDAO](https://github.com/minerdao)团队在官方的版本上做了一些改动，增加了证明效率统计、默认支持显卡的版本，欢迎大家尝鲜：https://github.com/minerdao/snarkOS

#### 1.2 什么时候开始？
2周后将公布Zprize算法比赛的结果 — 主要是msm算法在显卡和fpga的实现，这个比赛结果公布以后，同时代码合并完毕，才会启动真正的测试。
所以正式开始出奖励积分可能会在2周以后，估计会是1个月后（因为他们还要过感恩节🦃️，不像我们这么卷...）。

#### 1.3 积分现在有没有用？
现在的积分肯定是没用的，矿池用来秀肌肉、厂商吹牛用的。

#### 1.4 网络现在是什么情况
网络现在很容易挂，如果有测试过的朋友，刚开始启动，是连不上官方beacon节点的。目前官方总共有6个固定的beacon节点，都在美国。
```rust
let bootstrap = [
    "164.92.111.59:4133",  // 科罗拉多
    "159.223.204.96:4133", // 德克萨斯
    "167.71.219.176:4133", // 纽约
    "157.245.205.209:4133",// 俄勒冈
    "134.122.95.106:4133", // 加利福尼亚
    "161.35.24.55:4133",   // 新泽西
];
```
连不上可能有2个原因：
- `4133`和`3033`这2个端口没有正常开启，如果是机房的机器，需要路由器上做好NAT映射；如果是云服务器，安全组需要检查一下；
- 端口正常的情况下，启动后需要等待大概10分钟左右，就可以连上`beacon`节点了。

其他可能的情况欢迎各位大佬补充。

## 2 Testnet3如何挖？

#### 2.1 beacon, client, prover, validator都是干什么的？
Aleo Testnet3 目前共有4种类型的节点，简单说明一下，后面发文章详细介绍。
- client: 应用节点，应用开发者通过client节点，将Aleo应用部署到链上，应用部署及执行过程产生的交易都进入了内存池(Memory Pool)；
- beacon: 信标节点，主要做以下3件事：
  - 从内存池中读取交易，生成待证明待难题(Coinbase Puzzle)；
  - 收到证明结果后，通知validator验证证明结果；
  - 多个被验证的交易产生后，打包生成区块。
- prover: 证明节点，完成零知识证明的计算，也就是Aleo网络中的矿工，prover从beacon节点获取待证明的难题，完成证明后提交结果，然后由beacon通知validator验证；
- validator: 验证节点，用来验证矿工(prover)的证明结果；

#### 2.2 用什么样的机器？
有什么机器用什么机器，目前Filecoin的算力机可以直接拿来跑，但是只能用一张显卡。

Intel低端的CPU也可以试试，对比一下效率。

#### 2.3 怎么挖？
编译前，机器上先预装rust1.65
```sh
git clone https://github.com/minerdao/snarkOS.git --depth 1
cd snarkOS

# 编译
cargo install --path .

# 如果遇到target权限不够，或者依赖库未安装的情况，可以运行以下命令编译:
sudo ./build_ubuntu.sh

# 创建钱包
./target/release/snarkos account new

# 启动Prover，输入上面创建地址的私钥
./run-prover.sh
```
#### 2.4 如何启用GPU?
修改`snarkOS/Cargo.toml/`里面的`[workspace.dependencies.snarkvm]`,

将
```toml
features = ["circuit", "console", "parallel"]
```
改为
```toml
features = ["circuit", "console", "parallel", "cuda"]
```