# 虚拟机扩磁盘
    # 参考: https://blog.csdn.net/weixin_44671771/article/details/123504570?utm_medium=distribute.pc_aggpage_search_result.none-task-blog-2~aggregatepage~first_rank_ecpm_v1~rank_v31_ecpm-2-123504570-null-null.pc_agg_new_rank&utm_term=%E8%99%9A%E6%8B%9F%E6%9C%BA%E6%80%8E%E4%B9%88%E5%88%86%E9%85%8D%E7%A3%81%E7%9B%98%E7%BB%99%E6%A0%B9%E7%9B%AE%E5%BD%95&spm=1000.2123.3001.4430

 fdisk  -l
 fdisk  /dev/sda
 lsblk
 fdisk  /dev/sda
    # n（新建）→p（分区类型选为主分区）→新建分区号（这里是4）→起始扇区（这里是默认得所以直接回车）→结束扇区（这里是默认得所以直接回车）→p（查看分区）→w（保存生效）

 lsblk
 pvcreate /dev/sda2
 partprobe
 partprobe  /dev/sda3
 lsblk
 pvcreate /dev/sda3
 vgextend centos /dev/sda3
 lvextend -L +27G /dev/centos/root
 pvdisplay
 lsblk
 df -h
 xfs_growfs /dev/centos/root
 df -h



## linux 增加交换空间
   参考 https://blog.csdn.net/qq_43606931/article/details/124016483

free -h
cat /proc/swaps
mkswap /swap1/swap
swapon /swap1/swap
chmod 600 /swap1/swap
vim /etc/fstab
cat /proc/sys/vm/swappiness
sysctl vm.swappiness=50
vim /etc/sysctl.conf
sysctl -p
df -h
free -h
history
