## 1. 存储测试工具
使用fio对现有方式进行磁盘性能测试：
```
sudo apt-get install -y fio
sudo fio -filename=/nvme/1.txt -direct=1 -iodepth 1 -thread -rw=randwrite -ioengine=psync -bs=4k -size=100G -numjobs=50 -runtime=180 -group_reporting -name=rand_100write_4k 
 # /nvme/1.txt为挂载路径
 # 测试参数相同情况下需要iops达到100K以上
```
## 2. 设置madam
```
sudo mdadm --create --verbose /dev/md0 --chunk=128  --level=0 --raid-devices=5 /dev/nvme0n1 /dev/nvme2n1 ........
#根据自己机器的硬盘数量进行设置
```
```
sudo mkfs.xfs -f -d agcount=128,su=128k,sw=5 -r extsize=640k  /dev/md0
#su=128k保持不变，sw=5为硬盘数量，extsize=640k ，为su*sw
```

## 3.写入madam信息
```
sudo mdadm --detail /dev/md0
sudo vim /etc/mdadm/mdadm.conf
ARRAY /dev/md0 UUID=7f1dbda8:90550745:c70bbe0f:0ea5f37e
保存退出
sudo update-initramfs -u
```

## 4.挂载硬盘
```
sudo blkid
sudo vim /etc/fstab
/dev/disk/by-uuid/fcf9ca63-831d-4190-a465-7e2082658d94 /home/cs/md0 xfs defaults 0 0
sudo mount -a
```

tips.
若同一台机器启动两个worker进程进行封装，可以将该机器组成2个raid目录，每个worker缓存文件存放在一个目录中。

