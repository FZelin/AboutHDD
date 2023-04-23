
### 前言

##### smr(叠瓦盘)的实现原理就不细说了，总结就是相同碟片情况下增加容量，成本下降。细分下来有三种

###### 1. DM-SMR（Drive Managed，是一个黑盒，模拟普通硬盘，一般的家用型）
###### 2. HA-SMR（Host Aware），白盒，让上层感知硬盘的状态。非常鸡肋。
###### 3. HM-SMR（Host Managed），没有盒子，全透明。全权交由上层优化操作。专业型，企业级。

```
1. 第一种其实已经在硬件层面模拟成和普通硬盘一样，直接用就行了，买硬盘时说的防止买到的叠瓦盘说的就是这种。
2. 很少碰到
3. 这种其实叫(zoned block devices)盘,需要文件系统支持才能使用，也就是我们今天主要讲解的SMR硬盘
```

### HM-SMR(zoned block devices) 硬盘的使用

###### HM-SMR硬盘的使用目前有两种方法，一种是文件系统支持，一种是通过软件把smr硬盘映射成一个普通硬盘(具体原理不知道，但应该也是和文件系统差不多的原理，就是通过一层代理，模拟出一个正常的硬盘来)

#### 1. 文件系统支持的方式

```
文件系统目前支持的有f2fs, btrfs, 推荐f2fs, linux内核需要支持zoned block devices,
推荐 debian11, 安装后直接可以用
```

==特别说明:==

==1. 用f2fs文件系统挂载硬盘，对内存需求比较高，一块14T硬盘大概需要1.5G内存==

==2. f2fs挂载相对来说是比较简单的，性能上也是最好的, 就是费内存(钱)==


##### 1.1 安装 f2fs-tools
```
apt-get -y install f2fs-tools
```
##### 1.2 格式化成f2fs
```
mkfs -t f2fs -m -f /dev/$name

# -t f2fs 指定格式化的文件系统类型
# -f 强制格式化,不加原来有文件系统的话会报错
# -m 支持zoned block devices,这个一定要加上 
```

##### 1.3 挂载硬盘

```
mount -t f2fs /dev/$name /mnt/path-to-mount
```

#### 2. 虚拟化硬盘的方式

==特别说明:==

==1. 虚拟化硬盘的方式挂载，内存要求不高，挂完24盘内存占用不到1G==

==2. 虚拟化硬盘方式相对f2fs文件系统的方式稍为麻烦一些，性能上测试写性能下降30-40%, 读性能上没有区别，最大的优点就是省内存(钱)==

##### 2.1 安装[dm-zoned-tools](https://github.com/westerndigitalcorporation/dm-zoned-tools)

```
#下载代码
git clone https://github.com/westerndigitalcorporation/dm-zoned-tools.git && cd dm-zoned-tools

#安装依赖
apt-get install pkg-config m4 autoconf automake libtool libblkid-dev kmod

#编译安装

./autogen.sh
./configure
make
make install

```

##### 2.2 格式化硬盘

```
#这个只需要格式化一次
dmzadm --format /dev/$name --force
```

##### 2.3 映射成正常硬盘

```
#这步每次开机要重新映射,这步完成后，fdisk -l, 发现会多出一块虚拟硬盘 /dev/mapper/dz-$serial  $serial为当前硬盘的序列号
dmzadm --start /dev/$name
```

##### 2.4 格式化虚拟化出来的硬盘

```
#可以用ext4文件系统，也可以用xfs这些,这步只需要操作一次
mkfs -t ext4 /dev/mapper/$serial
```

##### 2.5 挂载格式化好的虚拟硬盘

```
mount -t ext4 /dev/mapper/$serial /mnt/path-to-mount
```
