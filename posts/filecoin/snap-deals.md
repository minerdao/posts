# Snap Deals 操作指南

## 1. 关于Snap Deals
Snap Deals是Filecoin Network15之后提供的一个将原有CC扇区升级并存入订单数据，而无需重新封装的一个新功能。

## 2. Snap Deals如何存订单

#### 2.1 更改扇区状态
```bash
lotus-miner sectors snap-up 2
```

```json
> ID  State       OnChain  Active  Expiration                   Deals  DealWeight  VerifiedPower
> 2   Available   YES      YES     1549517 (in 10 weeks 1 day)  CC
```

#### 2.2 修改Miner配置
注意Miner的配置文件中，需要设置`MakeNewSectorForDeals = false`，否则订单将会创建新的扇区进行封装，不会存入已有的扇区。

#### 2.3 导入订单
客户端发订单，然后Miner导入订单：
```bash
lotus-miner storage-deals import-data bafyreiapr4djmjsdfs424n3dfyhtesizpvnqpe7dtc6n2iloeo6pu4efm4 /data/car/baga6ea4seaqaw4j4spzjg7gkdh42gae6zoa42buyxlgvekhp3fpi2t4ym233idy.car
```

如果订单导入后提示扇区剩余时间小于订单的存储时间，需要执行下面的目录，延长扇区周期：
```bash
lotus-miner sectors extend --new-expiration 3826102 56088 && lotus-miner sectors match-pending-pieces
```

## 3. 遇到的问题
#### 3.1 关于Snap Deals的时间 [TDOO]

#### 3.2 `sector xxxx must have deals, but does not`错误解决
https://github.com/filecoin-project/lotus/issues/8686

需要合并 https://github.com/filecoin-project/lotus/pull/9310

目前最新的master分支已经合并了该PR。