---
title: utils
date: 2018-10-29 15:31:19
tags: utils
categories: utils
---
### 查看网卡UUID, MAC
```
UUID:  
	CentOS7:  
		nmcli con show  
	CentOS6:  
		nmcli con list  
MAC:  
	CentOS7:  
		nmcli dev show  
	CentOS6:  
		nmcli dev list  
```

### 修改网卡为eth0
```
vim /etc/default/grub  
GRUB_CMDLINE_LINUX 添加 net.ifnames=0 biosdevname=0

grub2-mkconfig -o /boot/grub2/grub.cfg  
```
<!-- more -->

### PS1配置
```
export PS1="[\[\e[36;1m\]\t\e[0m \u@\h \W]\\$ "
```

PS1的常用参数以及含义:
```
\d ：代表日期，格式为weekday month date，例如："Mon Aug 1"
\H ：完整的主机名称
\h ：仅取主机名中的第一个名字
\t ：显示时间为24小时格式，如：HH：MM：SS
\T ：显示时间为12小时格式
\A ：显示时间为24小时格式：HH：MM
\u ：当前用户的账号名称
\v ：BASH的版本信息
\w ：完整的工作目录名称
\W ：利用basename取得工作目录名称，只显示最后一个目录名
\# ：下达的第几个命令
\$ ：提示字符，如果是root用户，提示符为 # ，普通用户则为 $
```
```
\[\e[字体色号;背景色号m\]........\[\e[0m\]
```
```
字体	背景
30  	40 黑色
31  	41 红色
32  	42 绿色
33  	43 黄色
34  	44 蓝色
35  	45 紫红色
36  	46 青蓝色
37  	47 白色
```

### git 操作
```
git rebase -i origin/master

修改commit
git commit --amend

继续rebase
git rebase --continue

自动合并commit
git commit --fixup  
git rebase -i --autosquash 
```

- pick就是cherry-pick
- reword 就是在cherry-pick的同时你可以编辑commit message，它会在执行的时候跳出一个界面让你编辑信息，当你退出的时候，会继续执行命令
- edit 麻烦点，cherry-pick同时 ，会停止，让你编辑信息，完了后，你要用git rebase --continue命令继续执行，相对上面来说有点麻烦，感觉没必要啊。
- squash，合并此条记录到前一个记录中，并把commit message也合并进去 。
- fixup ，合并此条记录到前一个记录中，但是忽略此条commit message

### linux监控
##### uptime  
快速查看机器的负载情况   1分钟、5分钟、15分钟的平均负载情况  
16:29:00 up 23:52,  6 users,  load average: 17.33, 13.04, 11.07

##### vmstat 1
r：等待在CPU资源的进程数  
free：系统可用内存数  
si, so：交换区写入和读取的数量  
us, sy, id, wa, st：这些都代表了CPU时间的消耗  

##### mpstat -P ALL 1
每个CPU的占用情况

##### pidstat 1
输出进程的CPU占用率

##### sar -n DEV 1
网络设备的吞吐率

##### sar -n TCP,ETCP 1
active/s：每秒本地发起的TCP连接数，既通过connect调用创建的TCP连接；  
passive/s：每秒远程发起的TCP连接数，即通过accept调用创建的TCP连接；  
retrans/s：每秒TCP重传数量；

##### sar
怀疑CPU存在瓶颈，可用 sar -u 和 sar -q 等来查看  
怀疑内存存在瓶颈，可用 sar -B、sar -r 和 sar -W 等来查看  
怀疑I/O存在瓶颈，可用 sar -b、sar -u 和 sar -d 等来查看  

### mdadm 命令
mdadm --zero-superblock /dev/sdc  
mdadm -Ds > /etc/mdadm.conf

### 批量重命名
在Debian或者Ubuntu环境下使用的语法是：  
```
rename 's/stringx/stringy/' files
```
而在CentOS下或者RedHat下是：
```
rename stringx stringy files
```

### 热插拔磁盘
获取编号 
```
lsscsi
```
热插 
```
echo "scsi add-single-device a b c d" >/proc/scsi/scsi 
```
重新扫描 
```
echo "- - -" > /sys/class/scsi_host/host2/scan  
for i in /sys/class/scsi_host/host*/scan;do echo "- - -" >$i;done
```
热拔
```
echo "scsi remove-single-device a b c d" >/proc/scsi/scsi  
ls /sys/bus/scsi/drivers/sd/2\:0\:1\:0/  
echo 1 > /sys/bus/scsi/drivers/sd/2\:0\:1\:0/delete 
```

### posix_fadvise fallocate fdatasync
posix_fadvise  
fallocate(fd, FALLOC_FL_KEEP_SIZE | FALLOC_FL_PUNCH_HOLE  
fdatasync

### 修改文件大小
truncate -s <size> <file>  
fallocate -l <size> <file>

### valgrind
valgrind --track-fds=yes --leak-check=full --undef-value-errors=yes ./a.out

### core文件
```
ulimit -c unlimited 大小不受限制
sysctl kernel.core_pattern | grep 'kernel.core_pattern = corepath' > /dev/null
sysctl -e kernel.core_pattern=corepath > /dev/null
/etc/sysctl.conf   kernel.core_pattern = /opt/core/core-%e-%p-%s
/proc/sys/kernel/core_pattern
```

### 编译so连接路径
-Wl,rpath=<your_lib_dir>

### 清缓存
```
#echo 3 > /proc/sys/vm/drop_caches
要达到释放缓存的目的，我们首先需要了解下关键的配置文件/proc/sys/vm/drop_caches。这个文件中记录了缓存释放的参数，默认值为0，也就是不释放缓存。他的值可以为0~3之间的任意数字，代表着不同的含义：

0 – 不释放
1 – 释放页缓存
2 – 释放dentries和inodes
3 – 释放所有缓存

知道了参数后，我们就可以根据我们的需要，使用下面的指令来进行操作。

首先我们需要使用sync指令，将所有未写的系统缓冲区写到磁盘中，包含已修改的 i-node、已延迟的块 I/O 和读写映射文件。否则在释放缓存的过程中，可能会丢失未保存的文件。

sync
接下来，我们需要将需要的参数写进/proc/sys/vm/drop_caches文件中，比如我们需要释放所有缓存，就输入下面的命令：

echo 3 > /proc/sys/vm/drop_caches
此指令输入后会立即生效，可以查询现在的可用内存明显的变多了。

要查询当前缓存释放的参数，可以输入下面的指令： 
cat /proc/sys/vm/drop_caches
```

### 查看内存条数
```
查看内存条数命令：
dmidecode | grep -A16 "Memory Device$"

其中第一行用全局角度描述系统使用的内存状况：
total——总物理内存
used——已使用内存，一般情况这个值会比较大，因为这个值包括了cache+应用程序使用的内存
free——完全未被使用的内存
shared——应用程序共享内存
buffers——缓存，主要用于目录方面,inode值等（ls大目录可看到这个值增加）
cached——缓存，用于已打开的文件
note:
    total=used+free
    used=buffers+cached (maybe add shared also)

第二行描述应用程序的内存使用：
前个值表示-buffers/cache——应用程序使用的内存大小，used减去缓存值
后个值表示+buffers/cache——所有可供应用程序使用的内存大小，free加上缓存值
note:
   -buffers/cache=used-buffers-cached
   +buffers/cache=free+buffers+cached

第三行表示swap的使用：
used——已使用
free——未使用
```














