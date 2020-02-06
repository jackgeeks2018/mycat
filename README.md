# 使用docker创建mycat mysql主从服务器

配置文件已经全部写好 基本找下面流程走一遍就能直接用

注意：mycat 和 mysql使用的字符集编码全部是 <code>utf8mb4</code>

```shell
% cd ~ #切换到主目录
% git clone https://github.com/liuwel/docker-mycat.git
% tree docker-mycat
├ compose
│   ├ docker-compose.yml
│   ├ master
│   │   └ Dockerfile
│   ├ mycat
│   │   ├ Dockerfile
│   │   └ Mycat-server-1.6.5-release-20180122220033-linux.tar.gz
├ config
│   ├ hosts
│   ├ mycat
│   │   ├ autopartition-long.txt
│   │   ├ auto-sharding-long.txt
│   │   ├ auto-sharding-rang-mod.txt
│   │   ├ cacheservice.properties
│   │   ├ ehcache.xml
│   │   ├ index_to_charset.properties
│   │   ├ log4j2.xml
│   │   ├ migrateTables.properties
│   │   ├ myid.properties
│   │   ├ partition-hash-int.txt
│   │   ├ partition-range-mod.txt
│   │   ├ rule.xml
│   │   ├ schema.xml
│   │   ├ sequence_conf.properties
│   │   ├ sequence_db_conf.properties
│   │   ├ sequence_distributed_conf.properties
│   │   ├ sequence_time_conf.properties
│   │   ├ server.xml
│   │   ├ sharding-by-enum.txt
│   │   ├ wrapper.conf
│   │   ├ zkconf
│   │   │   ├ autopartition-long.txt
│   │   │   ├ auto-sharding-long.txt
│   │   │   ├ auto-sharding-rang-mod.txt
│   │   │   ├ cacheservice.properties
│   │   │   ├ ehcache.xml
│   │   │   ├ index_to_charset.properties
│   │   │   ├ partition-hash-int.txt
│   │   │   ├ partition-range-mod.txt
│   │   │   ├ rule.xml
│   │   │   ├ schema.xml
│   │   │   ├ sequence_conf.properties
│   │   │   ├ sequence_db_conf.properties
│   │   │   ├ sequence_distributed_conf-mycat_fz_01.properties
│   │   │   ├ sequence_distributed_conf.properties
│   │   │   ├ sequence_time_conf-mycat_fz_01.properties
│   │   │   ├ sequence_time_conf.properties
│   │   │   ├ server-mycat_fz_01.xml
│   │   │   ├ server.xml
│   │   │   └ sharding-by-enum.txt
│   │   └ zkdownload
│   │       └ auto-sharding-long.txt
│   ├ mycat-logs
│   │   ├ mycat.log
│   │   ├ mycat.pid
│   │   └ wrapper.log
│   ├ mysql-master
│   │   ├ conf.d
│   │   │   ├ client.cnf
│   │   │   ├ docker.cnf
│   │   │   └ mysql.cnf
│   │   ├ my.cnf
│   │   ├ mysql.cnf
│   │   └ mysql.conf.d
│   │       └ mysqld.cnf
│   ├ mysql-s1
│   │   ├ conf.d
│   │   │   ├ client.cnf
│   │   │   ├ docker.cnf
│   │   │   └ mysql.cnf
│   │   ├ my.cnf
│   │   ├ mysql.cnf
│   │   └ mysql.conf.d
│   │       └ mysqld.cnf
│   └ mysql-s2
│       ├ conf.d
│       │   ├ client.cnf
│       │   ├ docker.cnf
│       │   └ mysql.cnf
│       ├ my.cnf
│       ├ mysql.cnf
│       └ mysql.conf.d
│           └ mysqld.cnf
└ README.md
19 directories, 69 files
```
#### mysql 主从服务器的配置已经写在config对应的目录中 
mysql-m1 : 主服务器 IP:172.18.0.2 

mysql-s1 : 从服务器slave1 IP:172.18.0.3

mysql-s2 : 从服务器slave2 IP:172.18.0.4

mycat    : Mycat服务器 IP:172.18.0.5  

### 修改hosts文件 添加解析
```shell
% sudo vi /etc/hosts
# docker-mycat m1:mysql-master主服务器 s1,s2：mysql-slave 从服务器
# mycat mycat中间件服务器
172.18.0.2      m1
172.18.0.3      s1
172.18.0.4      s2
172.18.0.5      mycat
127.0.0.1       local
```

### docker-compose.yml配置文件
```shell
% cd ~/docker-mycat/compose
% cat docker-compose.yml
```
```yml
version: '2'
services:
  m1:
    image: mysql:5.6
    container_name: m1
    volumes:
      - ../config/mysql-m1/:/etc/mysql/:ro
      #- /etc/localtime:/etc/localtime:ro
      - ../config/hosts:/etc/hosts:ro
    ports:
      - "3309:3306"
    networks:
      mysql:
        ipv4_address: 172.18.0.2
    ulimits:
      nproc: 65535
    hostname: m1
    mem_limit: 512m
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: root
  s1:
      image: mysql:5.6
      container_name: s1
      volumes:
        - ../config/mysql-s1/:/etc/mysql/:ro
       # - /etc/localtime:/etc/localtime:ro
        - ../config/hosts:/etc/hosts:ro
      ports:
        - "3307:3306"
      networks:
        mysql:
          ipv4_address: 172.18.0.3
      links:
        - m1
      ulimits:
        nproc: 65535
      hostname: s1
      mem_limit: 512m
      restart: always
      environment:
        MYSQL_ROOT_PASSWORD: root
  s2:
    image: mysql:5.6
    container_name: s2
    volumes:
      - ../config/mysql-s2/:/etc/mysql/:ro
      #- /etc/localtime:/etc/localtime:ro
      - ../config/hosts:/etc/hosts:ro
    ports:
      - "3308:3306"
    links:
      - m1
    networks:
      mysql:
        ipv4_address: 172.18.0.4
    ulimits:
      nproc: 65535
    hostname: s2
    mem_limit: 512m
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: root
  mycat:
    build: ./mycat
    container_name: mycat
    volumes:
      - ../config/mycat/:/mycat/conf/:ro
      - ../log/mycat-logs/:/mycat/logs/:rw
      #- /etc/localtime:/etc/localtime:ro
      - ../config/hosts:/etc/hosts:ro
    ports:
      - "8066:8066"
      - "9066:9066"
    links:
      - m1
      - s1
      - s2
    networks:
      mysql:
        ipv4_address: 172.18.0.5
    ulimits:
      nproc: 65535
    hostname: mycat
    mem_limit: 512m
    restart: always
networks:
  mysql:
    driver: bridge
    ipam:
      driver: default
      config:
      - subnet: 172.18.0.0/24
        gateway: 172.18.0.1

```
### Build 镜像
```shell
% sudo docker-compose build m1 s1 s2
Building m1
Step 1/4 : FROM mysql:5.7.17
 ---> 9546ca122d3a
 ...
Successfully built cffffead5570
Successfully tagged compose_s2:latest
```
### 运行 docker mysql主从数据库 (mysql数据库密码在yml文件里面)
```shell
% sudo docker-compose up -d m1 s1 s2  
Creating m1                             
Creating s2                             
Creating s1                             
```
### mysql主从配置
#### 配置m1主服务器
```shell
sudo docker exec -it m1 /bin/bash
root@m1:/# mysql -uroot -proot
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 3
Server version: 5.7.17-log MySQL Community Server (GPL)

Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
mysql>
```
已经进入m1主服务器mysql 命令行 

创建用于主从复制的用户repl
```shell
mysql> create user repl;
```
给repl用户授予slave的权限
```shell
mysql> GRANT REPLICATION SLAVE ON *.* TO 'repl'@'172.18.0.%' IDENTIFIED BY 'repl';
Query OK, 0 rows affected, 1 warning (0.00 sec)
```
刷新
```shell
mysql> FLUSH TABLES;
Query OK, 0 rows affected (0.00 sec)
```
查看binlog状态 记录File 和 Position 状态稍后从库配置的时候会用
```shell
mysql>  show master status;
+-------------------+----------+--------------+------------------+-------------------+
| File              | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+-------------------+----------+--------------+------------------+-------------------+
| master-bin.000003 |      644 |              |                  |                   |
+-------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```
#### 配置从库s1 s2
进入s1 shell
```shell
% sudo docker exec -it s1 /bin/bash
root@s1:/# mysql -uroot -proot
mysql> change master to master_host='m1',master_port=3306,master_user='repl',master_password='repl',master_log_file='master-bin.000003',master_log_pos=644;
Query OK, 0 rows affected, 2 warnings (0.05 sec)
mysql> start slave;
Query OK, 0 rows affected (0.00 sec)
mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: m1
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: master-bin.000004
          Read_Master_Log_Pos: 489
               Relay_Log_File: mysqld-relay-bin.000002
                Relay_Log_Pos: 284
        Relay_Master_Log_File: master-bin.000004
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 489
              Relay_Log_Space: 458
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 2
                  Master_UUID: 613ed8a2-48bf-11ea-8291-0242ac120002
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
```
注意：show slave status\G;后Slave_IO_Running，Slave_SQL_Running两个值要是 Yes

进入s2 后同s1操作

### mysql主从配置完成 现在测试一下
登陆主数据库 创建masterdb数据库 (这个数据库名在稍后的mycat里面会用到)
```shell
% mysql -uroot -ptoot -hm1
MySQL [(none)]> create database db1;create database db2;create database db3;
Query OK, 1 row affected (0.01 sec)
```
进入从库看看数据库是否创建
```shell
% mysql -uroot -proot -hs1
MySQL [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| db1                |
| db2                | 
| db3                |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.00 sec)
```
可以看到从库也已经创建成功了 到这里msyql的主从已经配置完成了

接下来是mycat的配置其实在 ~/config/mycat 里面已经配置好了直接就可以用了

看下schama.xml配置文件  
```shell 
% cat ~/config/mycat/schema.xml
```

```xml
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">
	<schema name="masterdb" checkSQLschema="false" sqlMaxLimit="100" >
                <table name="t_user" dataNode="dn1,dn2,dn3" rule="crc32slot">
                    <childTable name="t_admin" joinKey="user_id" parentKey="id" />
                </table>
        </schema>
        <dataNode name="dn1" dataHost="masterDH" database="db1" />
        <dataNode name="dn2" dataHost="masterDH" database="db2" />
        <dataNode name="dn3" dataHost="masterDH" database="db3" />
	<dataHost name="masterDH" maxCon="1000" minCon="10" balance="1"
			  writeType="0" dbType="mysql" dbDriver="native" switchType="1" slaveThreshold="100">
		<heartbeat>select user()</heartbeat>
		<writeHost host="m1" url="172.18.0.2:3306" user="root" password="root">
			<readHost host="s1" url="172.18.0.3:3306" user="root" password="root" />
			<readHost host="s2" url="172.18.0.4:3306" user="root" password="root" />
		</writeHost>
	</dataHost>
</mycat:schema>
              
```
server.xml 配置文件 
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- - - Licensed under the Apache License, Version 2.0 (the "License");
        - you may not use this file except in compliance with the License. - You
        may obtain a copy of the License at - - http://www.apache.org/licenses/LICENSE-2.0
        - - Unless required by applicable law or agreed to in writing, software -
        distributed under the License is distributed on an "AS IS" BASIS, - WITHOUT
        WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. - See the
        License for the specific language governing permissions and - limitations
        under the License. -->
<!DOCTYPE mycat:server SYSTEM "server.dtd">
<mycat:server xmlns:mycat="http://io.mycat/">
        <system>
        <property name="charset">utf8mb4</property>
        <property name="useSqlStat">0</property>  <!-- 1为开启实时统计、0为关闭 -->
        <property name="useGlobleTableCheck">0</property>  <!-- 1为开启全加班一致性检测、0为关闭 -->

                <property name="sequnceHandlerType">2</property>
      <!--  <property name="useCompression">1</property>--> <!--1为开启mysql压缩协议-->
        <!--  <property name="fakeMySQLVersion">5.6.20</property>--> <!--设置模拟的MySQL版本号-->
        <!-- <property name="processorBufferChunk">40960</property> -->
        <!--
        <property name="processors">1</property>
        <property name="processorExecutor">32</property>
         -->
                <!--默认为type 0: DirectByteBufferPool | type 1 ByteBufferArena-->
                <property name="processorBufferPoolType">0</property>
                <!--默认是65535 64K 用于sql解析时最大文本长度 -->
                <!--<property name="maxStringLiteralLength">65535</property>-->
                <!--<property name="sequnceHandlerType">0</property>-->
                <!--<property name="backSocketNoDelay">1</property>-->
                <!--<property name="frontSocketNoDelay">1</property>-->
                <!--<property name="processorExecutor">16</property>-->
                <!--
                        <property name="serverPort">8066</property> <property name="managerPort">9066</property>
                        <property name="idleTimeout">300000</property> <property name="bindIp">0.0.0.0</property>
                        <property name="frontWriteQueueSize">4096</property> <property name="processors">32</property> -->
                <!--分布式事务开关，0为不过滤分布式事务，1为过滤分布式事务（如果分布式事务内只涉及全局表，则不过滤），2为不
过滤分布式事务,但是记录分布式事务日志-->
                <property name="handleDistributedTransactions">0</property>
                        <!--
                        off heap for merge/order/group/limit      1开启   0关闭
                -->
                <property name="useOffHeapForMerge">1</property>
                <!--
                        单位为m
                -->
                <property name="memoryPageSize">1m</property>
                <!--
                        单位为k
                -->
                <property name="spillsFileBufferSize">1k</property>
                <property name="useStreamOutput">0</property>
                <!--
                        单位为m
                -->
                <property name="systemReserveMemorySize">384m</property>
                <!--是否采用zookeeper协调切换  -->
                <property name="useZKSwitch">true</property>
        </system>

        <!-- 全局SQL防火墙设置
        <firewall>
           <whitehost>
              <host host="172.18.0.2" user="root"/>
              <host host="172.18.0.3" user="root"/>
                                <host host="172.18.0.4" user="root"/>
           </whitehost>
       <blacklist check="false">
       </blacklist>
        </firewall>-->
        <user name="root">
                <property name="password">root</property>
                <property name="schemas">masterdb</property>

                <!-- 表级 DML 权限设置 -->
                <!--
                <privileges check="false">
                        <schema name="TESTDB" dml="0110" >
                                <table name="tb01" dml="0000"></table>
                                <table name="tb02" dml="1111"></table>
                        </schema>
                </privileges>
                 -->
        </user>
</mycat:server>
```
看一下rule.xml


```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- - - Licensed under the Apache License, Version 2.0 (the "License"); `
	- you may not use this file except in compliance with the License. - You 
	may obtain a copy of the License at - - http://www.apache.org/licenses/LICENSE-2.0 
	- - Unless required by applicable law or agreed to in writing, software - 
	distributed under the License is distributed on an "AS IS" BASIS, - WITHOUT 
	WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. - See the 
	License for the specific language governing permissions and - limitations 
	under the License. -->
<!DOCTYPE mycat:rule SYSTEM "rule.dtd">
<mycat:rule xmlns:mycat="http://io.mycat/">
	<tableRule name="rule1">
		<rule>
			<columns>id</columns>
			<algorithm>func1</algorithm>
		</rule>
	</tableRule>
<tableRule name="rule2">
	<rule>
		<columns>user_id</columns>
		<algorithm>func1</algorithm>
	</rule>
</tableRule>

<tableRule name="sharding-by-intfile">
	<rule>
		<columns>sharding_id</columns>
		<algorithm>hash-int</algorithm>
	</rule>
</tableRule>
<tableRule name="auto-sharding-long">
	<rule>
		<columns>id</columns>
		<algorithm>rang-long</algorithm>
	</rule>
</tableRule>
<tableRule name="mod-long">
	<rule>
		<columns>id</columns>
		<algorithm>mod-long</algorithm>
	</rule>
</tableRule>
<tableRule name="sharding-by-murmur">
	<rule>
		<columns>id</columns>
		<algorithm>murmur</algorithm>
	</rule>
</tableRule>
<tableRule name="crc32slot">
	<rule>
		<columns>id</columns>
		<algorithm>crc32slot</algorithm>
	</rule>
</tableRule>
<tableRule name="sharding-by-month">
	<rule>
		<columns>create_time</columns>
		<algorithm>partbymonth</algorithm>
	</rule>
</tableRule>
<tableRule name="latest-month-calldate">
	<rule>
		<columns>calldate</columns>
		<algorithm>latestMonth</algorithm>
	</rule>
</tableRule>

<tableRule name="auto-sharding-rang-mod">
	<rule>
		<columns>id</columns>
		<algorithm>rang-mod</algorithm>
	</rule>
</tableRule>

<tableRule name="jch">
	<rule>
		<columns>id</columns>
		<algorithm>jump-consistent-hash</algorithm>
	</rule>
</tableRule>

<function name="murmur"
	class="io.mycat.route.function.PartitionByMurmurHash">
	<property name="seed">0</property><!-- 默认是0 -->
	<property name="count">2</property><!-- 要分片的数据库节点数量，必须指定，否则没法分片 -->
	<property name="virtualBucketTimes">160</property><!-- 一个实际的数据库节点被映射为这么多虚拟节点，默认是160倍，也就是虚拟节点数是物理节点数的160倍 -->
	<!-- <property name="weightMapFile">weightMapFile</property> 节点的权重，没有指定权重的节点默认是1。以properties文件的格式填写，以从0开始到count-1的整数值也就是节点索引为key，以节点权重值为值。所有权重值必须是正整数，否则以1代替 -->
	<!-- <property name="bucketMapPath">/etc/mycat/bucketMapPath</property> 
	     			用于测试时观察各物理节点与虚拟节点的分布情况，如果指定了这个属性，会把虚拟节点的murmur hash值与物理节点的映射按行输出到这个文件，没有默认值，如果不指定，就不会输出任何东西 -->
</function>

<function name="crc32slot"
		  class="io.mycat.route.function.PartitionByCRC32PreSlot">
	<property name="count">3</property><!-- 要分片的数据库节点数量，必须指定，否则没法分片 -->
</function>
<function name="hash-int"
	class="io.mycat.route.function.PartitionByFileMap">
	<property name="mapFile">partition-hash-int.txt</property>
</function>
<function name="rang-long"
	class="io.mycat.route.function.AutoPartitionByLong">
	<property name="mapFile">autopartition-long.txt</property>
</function>
<function name="mod-long" class="io.mycat.route.function.PartitionByMod">
	<!-- how many data nodes -->
	<property name="count">2</property>
</function>

<function name="func1" class="io.mycat.route.function.PartitionByLong">
	<property name="partitionCount">8</property>
	<property name="partitionLength">128</property>
</function>
<function name="latestMonth"
	class="io.mycat.route.function.LatestMonthPartion">
	<property name="splitOneDay">24</property>
</function>
<function name="partbymonth"
	class="io.mycat.route.function.PartitionByMonth">
	<property name="dateFormat">yyyy-MM-dd</property>
	<property name="sBeginDate">2015-01-01</property>
</function>

<function name="rang-mod" class="io.mycat.route.function.PartitionByRangeMod">
    	<property name="mapFile">partition-range-mod.txt</property>
</function>

<function name="jump-consistent-hash" class="io.mycat.route.function.PartitionByJumpConsistentHash">
	<property name="totalBuckets">3</property>
</function>
</mycat:rule>
```




### mycat

```shell
% cd ~/docker-mycat/compose
% sudo docker-compose up -d mycat
```

### 整体测试
```shell
% mysql -uroot -proot -hmycat -P8066
```
```shell
MySQL \[(none)\]> show databases;
+----------+
| DATABASE |
+----------+
| masterdb |
+----------+
1 row in set (0.00 sec)
```

### 测试数据
```shell
MySQL [(none)]> use masterdb                                                         
Database changed                                                                     

---SQL相关脚本
drop table if exists t_admin;
drop table if exists t_user;

create table t_user(
    id int not null ,
    name varchar(32),
    primary key (id)
);

create table t_admin(
    id int not null,
    name varchar(32),
    user_id int,
    primary key (id)
);

insert into t_user(id, name) values(1, 'aaa');
insert into t_user(id, name) values(2, 'bbb');
insert into t_user(id, name) values(3, 'ccc');
insert into t_user(id, name) values(4, 'ddd');
insert into t_user(id, name) values(5, 'eee');

insert into t_admin(id, name, user_id) values(1, 'admin1', 1);
insert into t_admin(id, name, user_id) values(2, 'admin2', 1);
insert into t_admin(id, name, user_id) values(3, 'admin3', 2);
insert into t_admin(id, name, user_id) values(4, 'admin4', 2);
insert into t_admin(id, name, user_id) values(5, 'admin5', 3);
insert into t_admin(id, name, user_id) values(6, 'admin6', 3);
insert into t_admin(id, name, user_id) values(7, 'admin7', 4);
insert into t_admin(id, name, user_id) values(8, 'admin8', 4);
insert into t_admin(id, name, user_id) values(9, 'admin9', 5);
insert into t_admin(id, name, user_id) values(10, 'admin0', 5);

-- 查询语法：
select * from t_user left join t_admin on t_user.id = t_admin.user_id
--查询结果：
+----+------+-------+------+--------+---------+
| id | name | _slot | id   | name   | user_id |
+----+------+-------+------+--------+---------+
|  3 | ccc  | 32411 |    5 | admin5 |       3 |
|  3 | ccc  | 32411 |    6 | admin6 |       3 |
|  5 | eee  | 27566 |    9 | admin9 |       5 |
|  5 | eee  | 27566 |   10 | admin0 |       5 |
|  1 | aaa  | 44983 |    1 | admin1 |       1 |
|  1 | aaa  | 44983 |    2 | admin2 |       1 |
|  2 | bbb  | 65037 |    3 | admin3 |       2 |
|  2 | bbb  | 65037 |    4 | admin4 |       2 |
|  4 | ddd  | 68408 |    7 | admin7 |       4 |
|  4 | ddd  | 68408 |    8 | admin8 |       4 |
+----+------+-------+------+-------+                                                
```

注意：！！！

如果调整rule.xml分片规则请删除ruledata目录

感谢：

### [liuwel](https://github.com/liuwel/docker-mycat)

### [尚学堂](http://www.sxtbz.com/)

