---
title: Oracle 数据库启动阶段详解
date: 2016-07-14 20:39:47
tags: 
- startup 
- nomount
- mount
- open
category:
- Oracle DBA
---

Oracle Server主要由两部分组成：Instance 和Database 。Instance 是指一组后台进程/线程和一块共享内存区域，而 Database是指存储在磁盘上的一组物理文件。本文由数据库 如何启动入手。 

## 数据库的启动 

首先来分析一下数据库的启动过程，Oracle 数据库的启动主要包含 3 个步骤： 
 <font color="red">（1）启动数据库到 nomount 状态；</font>
 <font color="red">（2）启动数据库到 mount 状态；</font>
 <font color="red">（3）启动数据库到 open 状态。</font>
下面逐个来看看各个步骤的具体过程以其含义。 

### 1.  启动数据库到nomount 状态

在启动的第一步骤，Oracle 首先寻找**参数文件（pfile/spfile ）**，然后根据参数文件中 的设置，创建实例，分配内存，启动后台进程。 
 
在这里可以看到，<font color="red">只要拥有了一个参数文件，就可以凭之启动实例（Instance）， 这一步 骤并不需要任何控制文件或数据文件的参与。 </font>

在创建数据库时，如果在这一步骤就出现问题，那么通常可能是**系统配置（内核参数等）** 存在问题，用户需要检查是否分配了足够的系统资源等。 来看一下启动到 nomount 状态的过程： 

``` shell
[oracle@dbtest dbs]$ cd $ORACLE_HOME/dbs
[oracle@dbtest dbs]$ ls
hc_orcl.dat  init.ora  initorcl.ora  lkORCL  orapworcl  spfileorcl.ora
[oracle@dbtest dbs]$ sqlplus / as sysdba

SQL*Plus: Release 11.2.0.1.0 Production on Wed May 4 10:36:45 2016

Copyright (c) 1982, 2009, Oracle.  All rights reserved.

Connected to an idle instance.

SQL> startup nomount;
ORACLE instance started.

Total System Global Area 1152450560 bytes
Fixed Size                  2212696 bytes
Variable Size             922750120 bytes
Database Buffers          218103808 bytes
Redo Buffers                9383936 bytes
SQL> 
```

注意这里，Oracle 根据参数文件的内容，创建了 instance ，分配了相应的内存区域，启 动了相应的后台进程。 此时观察警报日志文件（`alert_<sid>.log` ; <font color="green">show parameter dump查看路径</font>），可以看到这一阶段的启动过程，读取参数 文件，应用参数启动实例，所有在参数文件中定义的非缺省参数都会记录在警报日志文件中： 

```
Starting up:
Oracle Database 11g Enterprise Edition Release 11.2.0.1.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options.
Using parameter settings in server-side spfile /u01/app/oracle/product/11.2.0/db_1/dbs/spfileorcl.ora
System parameters with non-default values:
  processes                = 150
  sga_target               = 176M
  memory_target            = 1104M
  memory_max_target        = 1104M
  control_files            = "/u01/app/oracle/oradata/orcl/control01.ctl"
  control_files            = "/u01/app/oracle/flash_recovery_area/orcl/control02.ctl"
  db_block_size            = 8192
  compatible               = "11.2.0.0.0"
  db_recovery_file_dest    = "/u01/app/oracle/flash_recovery_area"
  db_recovery_file_dest_size= 3882M
  undo_tablespace          = "UNDOTBS1"
  remote_login_passwordfile= "EXCLUSIVE"
  db_domain                = "oracle.com"
  global_names             = FALSE
  dispatchers              = "(PROTOCOL=TCP) (SERVICE=orclXDB)"
  shared_servers           = 5
  audit_file_dest          = "/u01/app/oracle/admin/orcl/adump"
  audit_trail              = "DB"
  db_name                  = "orcl"
  open_cursors             = 300
  diagnostic_dest          = "/u01/app/oracle"

```

然后后台进程依次启动： 

```
Wed May 04 10:36:55 2016
PMON started with pid=2, OS id=3128 
Wed May 04 10:36:55 2016
VKTM started with pid=3, OS id=3132 at elevated priority
VKTM running at (10)millisec precision with DBRM quantum (100)ms
Wed May 04 10:36:55 2016
GEN0 started with pid=4, OS id=3138 
Wed May 04 10:36:55 2016
DIAG started with pid=5, OS id=3142 
Wed May 04 10:36:55 2016
DBRM started with pid=6, OS id=3146 
Wed May 04 10:36:55 2016
PSP0 started with pid=7, OS id=3150 
Wed May 04 10:36:55 2016
DIA0 started with pid=8, OS id=3158 
Wed May 04 10:36:55 2016
MMAN started with pid=9, OS id=3162 
Wed May 04 10:36:55 2016
DBW0 started with pid=10, OS id=3166 
Wed May 04 10:36:55 2016
LGWR started with pid=11, OS id=3170 
Wed May 04 10:36:55 2016
CKPT started with pid=12, OS id=3175 
Wed May 04 10:36:55 2016
SMON started with pid=13, OS id=3179 
Wed May 04 10:36:55 2016
RECO started with pid=14, OS id=3184 
Wed May 04 10:36:55 2016
MMON started with pid=15, OS id=3189 
starting up 1 dispatcher(s) for network address '(ADDRESS=(PARTIAL=YES)(PROTOCOL=TCP))'...
Wed May 04 10:36:55 2016
MMNL started with pid=16, OS id=3193 
starting up 5 shared server(s) ...
ORACLE_BASE from environment = /u01/app/oracle
```

<font color="red">这里注意一下 Oracle选择参数文件的顺序:</font>

Oracle 首选`spfile<sid>.ora `文件作为启动参数文件；如果该文件不 存在，Oracle选择`spfile.ora` 文件；如果前两者都不存在，Oracle将会选择 `init<sid>.ora `文件；如果以上 3 个文件都不存在，Oracle 将无法创建和启动 instance ,Oracle将无法启动。 

用户可以在SQL*PLUS 中通过show parameter spfile 命令来检查数据库是否使用了 spfile文件，如果 value 不为Null，则数据库使用了 spfile文件： 
``` sql
SQL> show parameter spfile

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
spfile                               string      /u01/app/oracle/product/11.2.0
                                                 /db_1/dbs/spfileorcl.ora
SQL> 
```

**这时候也可以从操作系统查看启动了的后台进:**
```
[root@dbtest trace]# ps -ef|grep ora_  
oracle    3128     1  0 10:36 ?        00:00:00 ora_pmon_orcl
oracle    3132     1  0 10:36 ?        00:00:00 ora_vktm_orcl
oracle    3138     1  0 10:36 ?        00:00:00 ora_gen0_orcl
oracle    3142     1  0 10:36 ?        00:00:00 ora_diag_orcl
oracle    3146     1  0 10:36 ?        00:00:00 ora_dbrm_orcl
oracle    3150     1  0 10:36 ?        00:00:00 ora_psp0_orcl
oracle    3158     1  0 10:36 ?        00:00:00 ora_dia0_orcl
oracle    3162     1  0 10:36 ?        00:00:00 ora_mman_orcl
oracle    3166     1  0 10:36 ?        00:00:00 ora_dbw0_orcl
oracle    3170     1  0 10:36 ?        00:00:00 ora_lgwr_orcl
oracle    3175     1  0 10:36 ?        00:00:00 ora_ckpt_orcl
oracle    3179     1  0 10:36 ?        00:00:00 ora_smon_orcl
oracle    3184     1  0 10:36 ?        00:00:00 ora_reco_orcl
oracle    3189     1  0 10:36 ?        00:00:00 ora_mmon_orcl
oracle    3193     1  0 10:36 ?        00:00:00 ora_mmnl_orcl
oracle    3197     1  0 10:36 ?        00:00:00 ora_d000_orcl
oracle    3201     1  0 10:36 ?        00:00:00 ora_s000_orcl
oracle    3205     1  0 10:36 ?        00:00:00 ora_s001_orcl
oracle    3209     1  0 10:36 ?        00:00:00 ora_s002_orcl
oracle    3213     1  0 10:36 ?        00:00:00 ora_s003_orcl
oracle    3217     1  0 10:36 ?        00:00:00 ora_s004_orcl
root      3358  3253  0 10:50 pts/3    00:00:00 grep ora_
```

<font color="red">如果这3 个文件都不存在，Oracle 将无法启动： </font>

	[oracle@dbtest dbs]$ mv init.ora init.ora.bak
	[oracle@dbtest dbs]$ mv initorcl.ora initorcl.ora.bak
	[oracle@dbtest dbs]$ mv spfileorcl.ora spfileorcl.ora.bak
	[oracle@dbtest dbs]$ ls
	hc_orcl.dat  init.ora.bak  initorcl.ora.bak  lkORCL  orapworcl  spfileorcl.ora.bak
	[oracle@dbtest dbs]$ sqlplus / as sysdba

	SQL*Plus: Release 11.2.0.1.0 Production on Wed May 4 10:55:42 2016

	Copyright (c) 1982, 2009, Oracle.  All rights reserved.

	Connected to an idle instance.

	SQL> startup nomount;
	ORA-01078: failure in processing system parameters
	LRM-00109: could not open parameter file '/u01/app/oracle/product/11.2.0/db_1/dbs/initorcl.ora'


在Oracle整个启动过程中，参数文件是写在应用程序中的硬代码，按照如上顺序进行查 找，不能改变Oracle的搜索路径及行为，但是如果参数文件不在相应的位置，在Linux/UNIX 系统上，可以通过符号链接来进行重定位。 

<font color="red">在参数文件中，通常需要最少的参数是 db_name，设置了这个参数之后，数据库实例就可以启动，来看一个简单的测试： </font>
	
	SQL> ! echo "db_name=julia" > initorcl.ora
	SQL> startup nomount;
	ORACLE instance started.
	Total System Global Area  217157632 bytes
	Fixed Size                  2211928 bytes
	Variable Size             159387560 bytes
	Database Buffers           50331648 bytes
	Redo Buffers                5226496 bytes

这样，就通过了最少的参数需求启动了 Oracle实例。 

### 2.  启动数据库到mount 状态 

启动到nomount 状态以后，Oracle就可以从参数文件中获得**控制文件**的位置信息， 这一部分信息在参数文件中的记录类似如下所示（Oracle缺省会创建3 个控制文件，这 3 个控制文件的内容完全一致，是Oracle为了安全而采用的镜像手段，在生产环境中，通 常应该将3 个控制文件存放在不同的物理硬盘上，避免因为介质故障而同时损坏3 个控制 文件）： 

	SQL> show parameter control_files

	NAME                                 TYPE        VALUE
	------------------------------------ ----------- ------------------------------
	control_files                        string      /u01/app/oracle/product/11.2.0
													 /db_1/dbs/cntrlorcl.dbf
													 
在nomount 状态，可以查询`v$parameter `视图，获得控制文件信息，这部分信息来自启 动的参数文件；当数据库 mount 之后，可以查询 `v$controlfile `视图获得关于控制文件的信 息，此时，这部分信息来自控制文件： 

	[oracle@dbtest dbs]$ mv init.ora.bak init.ora
	[oracle@dbtest dbs]$ mv initorcl.ora.bak initorcl.ora
	[oracle@dbtest dbs]$ mv spfileorcl.ora.bak spfileorcl.ora
	[oracle@dbtest dbs]$ sqlplus / as sysdba

	SQL*Plus: Release 11.2.0.1.0 Production on Wed May 4 11:07:07 2016

	Copyright (c) 1982, 2009, Oracle.  All rights reserved.

	Connected to an idle instance.

	SQL> startup nomount;
	ORACLE instance started.

	Total System Global Area 1152450560 bytes
	Fixed Size                  2212696 bytes
	Variable Size             922750120 bytes
	Database Buffers          218103808 bytes
	Redo Buffers                9383936 bytes
	
	SQL> alter database mount;   

	Database altered.

	SQL> select * from v$controlfile; 

	STATUS
	-------
	NAME
	--------------------------------------------------------------------------------
	IS_ BLOCK_SIZE FILE_SIZE_BLKS
	--- ---------- --------------

	/u01/app/oracle/oradata/orcl/control01.ctl
	NO       16384            594


	/u01/app/oracle/flash_recovery_area/orcl/control02.ctl
	NO       16384            594

	STATUS
	-------
	NAME
	--------------------------------------------------------------------------------
	IS_ BLOCK_SIZE FILE_SIZE_BLKS
	--- ---------- --------------

在mount 数据库的过程中，Oracle需要找到控制文件并锁定控制文件。如果控制文件全 部丢失此时就会报出如下错误： 

	SQL> alter database mount;  
	alter database mount
	*
	ERROR at line 1:
	ORA-00205: error in identifying control file, check alert log for more info

这时候alert<sid>.log 文件中通常会记录更为详细的信息。 

因为Oracle的3 个（缺省的）控制文件内容完全相同，如果只是损失了其中 1～2 个， 可以复制完好的控制文件，更改为相应的名称，就可以启动数据库；如果丢失了所有的控制 文件，那么就需要恢复或重建控制文件来打开数据库。 

在正常Mount 数据库的过程中，数据库的警报日志文件仅记录如下信息： 

	alter database mount
	Wed May 04 11:07:44 2016
	Successful mount of redo thread 1, with mount id 1438756220
	Database mounted in Exclusive Mode
	Lost write protection disabled
	Completed: alter database mount
	
在这一步骤中，数据库需要计算**Mount id** 并将其记录在控制文件中，然后开始启动 Heartbeat（心跳），每3 秒更新一次控制文件。

<font color="red">启动到Mount 状态，数据库必须具备的另外一个重要文件是口令文件</font>，该文件位于 $ORACLE_HOME/dbs 目录下，缺省的名称为 orapw<sid> 。 <font color="green">口令文件中存放 sysdba/sysoper 用户的用户名及口令</font>： 
```
	[oracle@dbtest dbs]$ strings orapworcl 
	]\[Z
	ORACLE Remote Password file
	INTERNAL
	769C0CD849F9B8B2
	5638228DAF52805F
	[oracle@dbtest dbs]$ 
```

在数据库没有启动之前，数据库内建用户是无法通过数据库本身来验证身份的，通过口 令文件，Oracle 可以实现对用户的身份认证，在数据库未启动之前登录，进而启动数据库。 **对于口令文件，Oracle 缺省查找 orapw<sid> 文件，如果该文件不存在，则继续查找orapw 文件，如果两者都不存在，则数据库将会出现错误。 **

如果口令文件丢失，通过 orapw 工具即可重建，所以在通常的备份策略中可以不必包含 口令文件： 

```
	[oracle@dbtest dbs]$ orapwd
	Usage: orapwd file=<fname> entries=<users> force=<y/n> ignorecase=<y/n> nosysdba=<y/n>

	  where
		file - name of password file (required),
		password - password for SYS will be prompted if not specified at command line,
		entries - maximum number of distinct DBA (optional),
		force - whether to overwrite existing file (optional),
		ignorecase - passwords are case-insensitive (optional),
		nosysdba - whether to shut out the SYSDBA logon (optional Database Vault only).
		
	  There must be no spaces around the equal-to (=) character.
	[oracle@dbtest dbs]$ 

```
<font color="green">通常在Linux/UNIX 平台下，在$ORACLE_HOME/dbs 目录下，还会存在另外一个文件，该
文件命名规则为 `lk<SID>`，lk指lock ，该文件在数据库启动时创建，用于操作系统对数据库
的锁定。当数据库启动时获得锁定，数据库关闭时释放。 </font>

该文件内容通常只有一行，提示不要删除，该文件仅仅用于锁定.

### 3.  启动数据库open阶段 

由于控制文件中记录了数据库中数据文件、日志文件的位置信息、检查点信息等重要信 息，所以在数据库的 open阶段，Oracle可以根据控制文件中记录的这些信息找到这些文件， 然后进行检查点及完整性检查。 

如果不存在问题就可以启动数据库，如果存在不一致或文件丢失则需要进行恢复。 

进一步地说，实际上在数据库 open的过程中，Oracle 进行的检查中包括以下两项： 

第一次检查数据文件头中的检查点计数**（Checkpoint cnt ）**是否和控制文件中的检查点 计数（Checkpoint cnt ）一致。此步骤检查用以确认数据文件是来自同一版本，而不是从备 份中恢复而来（因为 Checkpoint Cnt 不会被冻结，会一直被修改）。 下面通过一个简单的测试来说明一下 Checkpoint Cnt的作用。 

如果检查点计数检查通过，则数据库进行第二次检查。第二次检查数据文件头的开始SCN 和控制文件中记录的该文件的结束 SCN 是否一致，如果控制文件中记录的结束 SCN 等于数据 文件头的开始 SCN，则不需要对那个文件进行恢复。 

