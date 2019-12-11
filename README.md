# sequoiadb-docker
教程地址：http://doc.sequoiadb.com/cn/sequoiadb-cat_id-1562752047-edition_id-0


第一步：创建4个容器
 # 192.168.100.220
 docker run -itd -p 11810:11810 -p 11800:11800  -p 8000:8000 -p 11790:11790  -p 11780:11780 --privileged=true  --restart=always  -v  /nodedata/sequoiadb/master_data:/opt/sequoiadb/database --name coord_catalog1 --hostname coord_catalog1 sequoiadb/sequoiadb:latest
  docker run -itd -p 11820:11820 -p 11830:11830  -p 11840:11840 --privileged=true  --restart=always -v /nodedata/sequoiadb/sdb_data1:/opt/sequoiadb/database --name sdb_data1 --hostname sdb_data1 sequoiadb/sequoiadb:latest
# 192.168.100.221
docker run -itd -p 11810:11810 -p 11800:11800  -p 8000:8000 -p 11790:11790  -p 11780:11780 --privileged=true  --restart=always  -v  /nodedata/sequoiadb/master_data:/opt/sequoiadb/database --name coord_catalog2  --hostname coord_catalog2 sequoiadb/sequoiadb:latest

 docker run -itd   -p 11820:11820 -p 11830:11830  -p 11840:11840  --privileged=true  --restart=always -v /nodedata/sequoiadb/sdb_data2:/opt/sequoiadb/database --name sdb_data2 --hostname sdb_data2 sequoiadb/sequoiadb:latest
# 192.168.100.222
docker run -itd -p 11810:11810 -p 11800:11800  -p 8000:8000 -p 11790:11790  -p 11780:11780 --privileged=true  --restart=always  -v  /nodedata/sequoiadb/master_data:/opt/sequoiadb/database --name coord_catalog3  --hostname coord_catalog3  sequoiadb/sequoiadb:latest

 docker run -itd   -p 11820:11820 -p 11830:11830  -p 11840:11840   --privileged=true  --restart=always -v /nodedata/sequoiadb/sdb_data3:/opt/sequoiadb/database --name sdb_data3 --hostname sdb_data3 sequoiadb/sequoiadb:latest




查看4个容器的ip地址

docker inspect coord_catalog | grep IPAddress |awk 'NR==2 {print $0}'
docker inspect sdb_data1 | grep IPAddress |awk 'NR==2 {print $0}'
docker inspect sdb_data2 | grep IPAddress |awk 'NR==2 {print $0}'
docker inspect sdb_data3 | grep IPAddress |awk 'NR==2 {print $0}'

第二步：部署 SequoiaDB 集群
----------------------------------------
1 、修改database目录权限
chown  sdbadmin database/
chgrp sdbadmin_group database/
2、修改/etc/hosts 
192.168.2.194	coord_catalog1
192.168.2.196	sdb_data1
192.168.2.131	coord_catalog2
192.168.2.134	sdb_data2
192.168.2.66	coord_catalog3
192.168.2.69	sdb_data3
192.168.2.195   mysql_bsxq_m1
192.168.2.130   mysql_bsxq_m2




-------------编码和编辑节点均高可用-----------------------------------	

docker exec coord_catalog "/init.sh" \
      --coord='coord_catalog1:11810,coord_catalog2:11810,coord_catalog3:11810' \
      --catalog='coord_catalog1:11800,coord_catalog2:11800,coord_catalog3:11800' \
	--data='group1=sdb_data1:11820,sdb_data2:11820,sdb_data3:11820;group2=sdb_data2:11830,sdb_data3:11830,sdb_data1:11830;group3=sdb_data3:11840,sdb_data1:11840,sdb_data2:11840'
	

	
	
输出：************ Deploy SequoiaDB ************************
************ Deploy SequoiaDB ************************
Create catalog: coord_catalog1:11800
Create catalog: coord_catalog2:11800
Create catalog: coord_catalog3:11800
Create coord:   coord_catalog1:11810
Create coord:   coord_catalog2:11810
Create coord:   coord_catalog3:11810
Create data:    sdb_data1:11820
Create data:    sdb_data2:11820
Create data:    sdb_data3:11820
Create data:    sdb_data2:11830
Create data:    sdb_data3:11830
Create data:    sdb_data1:11830
Create data:    sdb_data3:11840
Create data:    sdb_data1:11840
Create data:    sdb_data2:11840

	
第三步：创建mysql容器	
//主节点
docker run -itd -p 3306:3306 -v /nodedata/sequoiadb/mysql:/opt/sequoiasql/mysql/database --privileged=true --name mysql_bsxq_m1 --hostname mysql_bsxq_m1 sequoiadb/sequoiasql-mysql:latest
//可作为读节点
docker run -itd -p 3306:3306 -v /nodedata/sequoiadb/mysql:/opt/sequoiasql/mysql/database --privileged=true --name mysql_bsxq_m2 --hostname mysql_bsxq_m2 sequoiadb/sequoiasql-mysql:latest
查看容器ip地址
$ docker inspect mysql_bsxq | grep IPAddress | awk 'NR==2 {print $0}'
第四步：将 MySQL 实例注册入协调节点
chown  sdbadmin database/
chgrp sdbadmin_group database/
在opt/sequoiasql/mysql 路径下
mkdir binlog
chown  sdbadmin binlog/
chgrp sdbadmin_group binlog/
docker exec mysql_master "/init.sh" --port=3306 --coord='coord_catalog1:11810'



service sequoiasql-mysql status


【问题描述】  
用户通过快速部署安装集群，没有安装 SAC 服务。所以问能否再单独安装 SAC 服务？ 

【解决办法】  
在集群的所有机器上执行命令：“ps -ef | grep sdbom”，查看机器上是否有SAC服务进程。如果是通过快速部署的方式启动的集群，那应该是没有这个进程的。确认没有SAC服务进程后，执行以下步骤：  
1. 启动SAC服务进程，在sdb shell执行命令：  
>var oma = new Oma( "localhost", 11790 )  
>oma.createOM( 11780, "/opt/sequoiadb/database/sms/11780", { httpname: 8000, wwwpath: "/opt/sequoiadb/web" } )  
>oma.startNode( 11780 )  
2. 将sequoiadb.xxx.run包和sequoiasql.xxx.run包拷贝到目录“/opt/sequoiadb/packet”下，并赋予它们可读权限  
3. 此时SAC服务进程已经准备就绪，接下来进行web端操作。SAC服务进程只需要在一台机器上运行即可，目前它不是分布式的  
SAC单独安装，在3.0版本以前可以发现SequoiaDB存储服务，但是无法发现SQL实例，在3.2.2版本之后才有发现SQL实例的功能 

mysql 双主架构
https://www.cnblogs.com/zping/p/7928805.html
GRANT REPLICATION SLAVE ON *.* TO 'RepUser'@'%'identified by 'bsxq123456';


查看serverid   show variables like 'ser%';
查看是否开启binlog      重启mysql后    show variables like '%log_bin%'； Value 为 ON即可
 
查询binlog 变动信息   show binlog events;
auto.cnf
master1
########## add by huanglin
server_id = 1 # 两天机器的serverid不能相同。
log-bin = /opt/sequoiasql/mysql/binlog/mysql-bin
binlog_format = mixed
relay-log=/opt/sequoiasql/mysql/binlog/bsxq-reply-bin
relay-log-index=/opt/sequoiasql/mysql/binlog/bsxq-reply-bin.index
auto-increment-offset = 1
auto-increment-increment = 1

master2
########## add by huanglin
server_id=3
log-bin = /opt/sequoiasql/mysql/binlog/mysql-bin
binlog_format = mixed
relay-log=/opt/sequoiasql/mysql/binlog/master1-reply-bin
relay-log-index=/opt/sequoiasql/mysql/binlog/master1-reply-bin.index
auto-increment-offset = 1
auto-increment-increment = 1
log-slave-updates=true


CHANGE MASTER TO MASTER_USER='RepUser',MASTER_HOST='mysql_bsxq_m1',MASTER_PASSWORD='bsxq123456',MASTER_PORT=3306,MASTER_LOG_FILE='mysql-bin.000001',MASTER_LOG_POS=437;  
CHANGE MASTER TO MASTER_USER='RepUser',MASTER_HOST='mysql_bsxq_m2',MASTER_PASSWORD='bsxq123456',MASTER_PORT=3306,MASTER_LOG_FILE='mysql-bin.000001',MASTER_LOG_POS=437;  
常用命令：

start slave;
stop salve;
show slave status\G;
show table status;
show master status;
show full processlist;
