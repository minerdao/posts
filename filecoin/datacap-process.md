# Filecoin Datacap操作流程

## 1. 打包car文件
先编译打包工具，需要安装Go 1.90+。
```sh
$ git clone https://github.com/minerdao/lotus-car.git
$ cd lotus-car
$ go build -o lotus-car
```

安装完以后，执行以下命令打包car文件：
```sh
./lotus-car generate --input=/mnt/md0/1712/1712.json --parent=/mnt/md0/1712/raw --tmp-dir=/mnt/md0/tmp1 --quantity=320 --out-dir=/mnt/md0/car/dataset_1712_3_320  --out-file=/home/fil/csv/dataset_1712_3_320.csv
```
参数说明：
- **--input**：原始文件的索引文件路径，`.json`格式，通过上面lotus-car仓库中`python3 main.py -i`来生成，注意生成索引文件对时候，需要先修改`main.py`中的数据集根目录和数据集名称。
- **--parent**：原始文件所在的目录，一般放在`raw`目录下。
- **--tmp-dir**：打包过程中的临时文件路径，需要放在ssd上。
- **--quantity**：打包的car文件数量。
- **--out-dir**：car文件的保存位置。
- **--out-file**：打包完car文件后，输出的csv文件名称及路径，改文件用于client发订单。

## 2. Client发单
将上面的`--out-file`csv文件发给Client，由Client来发存储订单。

## 3. Miner接单
Miner接单需先配置好Boost，关于Boost的配置参照: https://boost.filecoin.io/getting-started/getting-started。

存储订单发送完毕以后，将生成`dataset_1711_4_3200.csv`这样的索引文件，Miner通过该文件来导入离线订单。
使用上面lotus-car仓库中的`import-deals.sh`脚本来导入订单，注意修改脚本中car文件所在的目录。
```sh
$ ./import-deals.sh dataset_1711_4_3200
# 后面跟上数据集名称即可
```