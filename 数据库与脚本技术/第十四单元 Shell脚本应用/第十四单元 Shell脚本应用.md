[TOC]







# 第十四单元-脚本应用案例





## 14.1 mysql备份脚本

### 14.1.1 为什么需要数据库备份？

1、保证数据安全与完整

企业的数据安全应该来说是企业的命脉，一旦丢失或造成损坏，轻则损失客户与金钱，重则倒闭（已经有前例在）。

备份的目的：为了保证数据在被人为失误、操作不当、蓄意等情况下删除或损坏后，能及时、有效的进行恢复并不会很大程度上影响到业务运行。

2、为业务提供不间断服务

实际生产环境对数据库的要求，首先就是具备7×24×365不间断服务的能力，这也是一定要备份数据库的其中原因之一。



### 14.1.2 数据库的备份方式

常用的备份方式包括以下：

**1.逻辑备份--mysqldump，mydumper**

逻辑备份其实就是利用MySQL数据库自带的mysqldump命令，或者使用第三方的工具，然后把数据库里的数据**以SQL语句的方式导出成文件的形式。**在需要恢复数据时，通过使用相关的命令（如：source ）将备份文件里的SQL语句提取出来重新在数据库中执行一遍，从而达到恢复数据的目的。

**逻辑备份的优点与使用场景**

优点：简单，易操作，自带工具方便、可靠。

使用场景：数据库数据量不大的情况可以使用，数据量比较大（超过20G左右）时备份速度比较慢，一定程度上还会影响数据库本身的性能。

实例如下：

```
mysqldump -A -B --single-transaction >/server/backup/mysql_$(date +%F).sql
```

一般备份时都会进行压缩处理，以节省磁盘空间，如下

```
mysqldump -A -B  --single-transaction ｜gzip>/server/backup/mysql_$(date +%F).sql.gz
```



**2.物理备份--Xtrabackup(热备)**

物理备份就是利用命令（如cp、tar、scp等）直接将数据库的存储数据文件复制一份或多份，分别存放在其它目录，以达到备份的效果。

这种备份方式，由于在备份时数据库还会存在数据写入的情况，一定程度上会造成数据丢失的可能性。在进行数据恢复时，需要注意新安装的数据的目录路径、版本、配置等与原数据要保持高度一致，否则同样也会有问题。

**所以，这种物理备份方式，常常需要在停机状态下进行，一般对实际生产中的数据库不太可取**。因此，此方式比较适用于数据库物理迁移，这种场景下这种方式比较高效率。

**物理备份的优点及使用场景**

优点：速度快，效率高。

场景：可用于停机维护及数据库物理迁移场景中。

实际生产环境中，具体使用哪种方式，就需要看需求与应用场景所定。





### 14.1.3 数据库备份策略

备份MYSQL数据库，每天凌晨12:00完全备份（自动），前提是数据量<20G

备份场景说明

应用场景分析：公司使用的是mysql数据库，保存着公司里所有重要的数据，如果数据库服务或者服务器出现问题，那对整个公司将是极大的损失，作为系统工程师的你，要负责数据库的备份。手动备份工作效率低下并且繁琐，所以要编写数据库备份脚本，以实现全自动备份数据库。



### 14.1.4 备份脚本实现

编写数据库备份脚本：

```shell
[root@ c6m01 ~]# vim /opt/scripts/mysqlbackup.sh

#!/bin/bash

#数据库用户和密码
user=root
passwd=123456

# 要备份的数据库名，多个数据库用空格分开
databases=(db1 db2 db3) 

# 备份文件要保存的目录
basepath='/backup/mysql/'
if [ ! -d "$basepath" ]; then
  mkdir -p "$basepath"
fi
# 循环databases数组
for db in ${databases[*]}
  do
    # 备份数据库生成SQL压缩文件--此方式是yum装mysql
    /usr/bin/mysqldump -u$user -p$passwd -B $db |gzip> $basepath$db-$(date +%Y%m%d).sql.gz
  done
# 删除7天之前的备份数据
find $basepath -mtime +7 -name "*.sql.gz"|xargs rm -f
```

添加到定时任务：

```shell
chmod +x /opt/scripts/mysqlbackup.sh

crontab -e
00 00 * * * /bin/sh /opt/scripts/mysqlbackup.sh
```





## 14.2 文件备份脚本

应用场景：对于公司重要的配置文件，为了在服务出现意外时能快速响应并第一时间恢复服务，需要定时备份公司业务中重要的文件，如公司网站的服务的配置文件、网页数据文件、公司员工工作时上传的重要文件。

编写备份脚本：

```shell
[root@ c6m01 ~]# vim /opt/scripts/file_backup.sh
#!/bin/bash

dirs=('/etc/httpd' '/etc/vsftpd')

backup_dir='/backup/data'
if [ ! -d "$backup_dir" ]; then
  mkdir -p "$backup_dir"
fi

for i in ${dirs[*]}
do
  bk_name=$(echo "$i"|awk -F '/' '{print $3}')
  cd $i && tar -zcf $backup_dir/$bk_name-$(date +%Y%m%d).tar.gz *
  if [ $? -eq 0 ]
   then
     echo "$i backup successed!!!" >>$backup_dir/backup.log
   else
     echo "$i backup failed!!!" >>$backup_dir/backup.log
  fi
done

find $backup_dir -mtime +7 -name "*.tar.gz" |xargs rm -f
```



## 14.3 服务状态监控脚本

应用场景：公司运行着ftp、mysql、http几个服务，是公司主要的几个服务，其中http服务支撑着公司的网站业务，与http相配合的mysql数据库、与公司员工文件相关联的是ftp服务，公司的系统工程师要时刻监控以上几个服务的运行状态，如服务没有运行则重启服务

实现思路与流程分析

分别检测ftp、httpd、mysql服务状态判断相关服务器状态，如没有运行则将服务启动

```shell
[root@ c6m01 ~]# vim /opt/scripts/service_check.sh
#!/bin/bash

services=('vsftpd' 'sshd' 'httpd' 'mysqld')

for i in ${services[*]}
do
    num=$(ps -ef|grep $i|grep -v grep|wc -l)
    if [ $num -eq 0 ];then
       echo -e "\033[31m $i 服务未运行... \033[0m"
       /etc/init.d/$i restart  &>/dev/null && echo "$i 服务启动成功"
       sleep 3
    else
       echo "$i 服务正在运行中..."
    fi
done
```





## 14.4 服务器运行状态监控脚本

**应用场景分析：**

为了持续观察公司服务器每天的基本运行状况，提供方便易读的集中的日志记录数据，需要结合Shell脚本和计划任务设置，定期记录不同时间段服务器的cpu负载、内存和交换空间、磁盘使用率等各种信息，以时刻监控服务器的运行负载，方便作出响应，及时处理服务器故障。

```shell
[root@ c6m01 ~]# vim /opt/scripts/monitor.sh
#!/bin/bash
nowday=`date +%Y%m%d`
cpuload=`uptime`
memuse=`free -m|grep "Mem"|awk '{print $3}'`
memfree=`free -m|grep "Mem"|awk '{print $4}'`
swapuse=`free -m|grep "Swap"|awk '{print $3}'`
swapfree=`free -m|grep "Swap"|awk '{print $4}'`
echo "当前时间：$nowday"
echo "CPU负载：$cpuload"
echo "内存已用：$memuse"
echo "内存剩余：$memfree"
echo "swap 已用$swapuse"
echo "swap 剩余$swapfree"
echo "磁盘使用情况"
df -h
echo "最近的十次登录"
last -10

```



