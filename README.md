1.  yum方式安装数据库
rpm -ivh https://repo.mysql.com//mysql80-community-release-el7-2.noarch.rpm
yum install mysql-community-server -y
service mysqld start
service mysqld status
systemctl enable mysqld


2.  tar.xz安装（8.0.15）和起多个数据库实例
BASE_DIR=/mysql/
DATA_DIR=/data/
port=3306
# create group and user
groupadd mysql
useradd  -r -g mysql mysql
mkdir -p ${BASE_DIR}/etc
mkdir -p ${DATA_DIR}
yum install -y wget
wget https://cdn.mysql.com//Downloads/MySQL-8.0/mysql-8.0.15-linux-glibc2.12-x86_64.tar.xz -O ${BASE_DIR}/mysql-8.0.15-linux-glibc2.12-x86_64.tar.xz
xz -d mysql-8.0.15-linux-glibc2.12-x86_64.tar.xz
tar -xvf ${BASE_DIR}/mysql-8.0.15-linux-glibc2.12-x86_64.tar -C ${BASE_DIR}/
mv ${BASE_DIR}/mysql-8.0.15-linux-glibc2.12-x86_64 ${BASE_DIR}/mysql8
mkdir ${BASE_DIR}/mysql8/etc
#======================================
cat > my.cnf <<EOF
[client]
port = 3306
socket = /data/mysql_3306/mysql.sock
default-character-set = utf8mb4

[mysqld]
##### base set #####
port = 3306
socket = /data/mysql_3306/mysql.sock
datadir = /data/mysql_3306
log-error = /data/mysql_3306/error.log
user = mysql
bind_address = 0.0.0.0
init_connect ='set autocommit=1'
sql_mode = NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES
wait_timeout = 3600
interactive_timeout = 3600
read_only = 0

#skip-grant-tables
##### character set #####
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
init_connect = 'SET NAMES utf8mb4'
# skip-character-set-client-handshake=1  #use server character
character-set-client-handshake = FALSE

##### bin_log set #####
binlog_format = row
server-id = 33060
log-bin = /data/mysql_3306/binlog
expire_logs_days = 3


## update time when other coloumn was updated
explicit_defaults_for_timestamp=1

## set log use time
log_timestamps = SYSTEM

## current check position
# show variables like ‘%pct%';
innodb_max_dirty_pages_pct = 30

[mysql]
no-auto-rehash
tee = history.log
default-character-set = utf8mb4

[myisamchk]
sort_buffer_size = 8M
read_buffer = 8M
write_buffer = 8M

[mysqlhotcopy]
interactive-timeout
EOF
#======================================
mv my.cnf ${BASE_DIR}/mysql8/etc/
cp ${BASE_DIR}/mysql8/etc/my.cnf my.cnf
sed -i "s/3306/${port}/g" my.cnf
mv -f my.cnf ${BASE_DIR}/mysql8/etc/my${port}.cnf
cd ${BASE_DIR}
id mysql || (echo -e "\033[31m please create mysql user \033[0m" && exit 1)
chown -R mysql:mysql ${BASE_DIR}/mysql8
cd ${BASE_DIR}/mysql8
./bin/mysqld --defaults-file=${BASE_DIR}mysql8/etc/my${port}.cnf --basedir=${BASE_DIR} --datadir=${DATA_DIR}mysql_${port} --initialize-insecure
[ $? -ne 0 ] && echo -e "\033[31m install mysql${port} failed \033[0m"
echo -e "\033[32m install mysql${port} successful \033[0m"
echo -e "\033[32m for convenience please create alias to /root/.bashrc\033[0m"
echo "alias mysql${port}=\"${BASE_DIR}/mysql8/bin/mysql -S ${DATA_DIR}/mysql_${port}/mysql.sock\"" >> /etc/.bashrc
source /etc/.bashrc
# start mysql
./bin/mysqld_safe --defaults-file=${BASE_DIR}mysql8/etc/my3306.cnf --user=mysql &
# mysql 
mysql${port}
create user 'admin'@'%' identified by 'admin';
grant all privileges on *.* to admin@'%' with grant option;

my.cnf 配置文件
#===============================================
cat > my.cnf << EOF
[client]
port = 3306
socket = /DATA_DIR/mysql_3306/mysql.sock
default-character-set = utf8mb4

[mysqld]
##### base set #####
port = 3306
socket = /DATA_DIR/mysql_3306/mysql.sock
datadir = /DATA_DIR/mysql_3306
log-error = /DATA_DIR/mysql_3306/error.log
user = mysql
bind_address = 0.0.0.0
skip-name-resolve
#slow_query_log = 1
#long_query_time = 0.125
#general_log = 1
#general_log_file = /DATA_DIR/mysql_3306/slow_query.log
#slow_query_log_file = /DATA_DIR/mysql_3306/slow_query.log
init_connect ='set autocommit=1'
sql_mode = NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES
wait_timeout = 3600
interactive_timeout = 3600
log_warnings = 1
read_only = 0
lower_case_table_names = 1

#skip-grant-tables
##### character set #####
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
init_connect = 'SET NAMES utf8mb4'
# skip-character-set-client-handshake=1  #use server character
character-set-client-handshake = FALSE

###### innodb set #####
innodb_data_file_path = ibdata1:500M;ibdata2:1024M:autoextend
innodb_file_per_table = 1
innodb_buffer_pool_size = 2G
innodb_buffer_pool_instances = 1
innodb_old_blocks_time = 1000
innodb_log_file_size = 512M
innodb_log_files_in_group = 3
innodb_log_buffer_size = 32M
innodb_flush_log_at_trx_commit = 2
innodb_lock_wait_timeout = 30
innodb_read_io_threads = 8
innodb_write_io_threads = 8
innodb_io_capacity = 4000
innodb_thread_concurrency = 4
innodb_flush_method = O_DIRECT
innodb_open_files = 20000


##### bin_log set #####
binlog_format = row
server-id = 33060
log-bin = /DATA_DIR/mysql_3306/binlog
sync_binlog = 2
sync_master_info = 0
sync_relay_log = 0
sync_relay_log_info = 0
log_bin_trust_function_creators = 1
expire_logs_days = 3
log_slave_updates = on
#binlog-ignore-db = test

slave_parallel_type=LOGICAL_CLOCK
slave_parallel_workers=16
relay_log_recovery=ON
master_info_repository=TABLE
relay_log_info_repository=TABLE
gtid_mode = on
enforce_gtid_consistency = on


###### other set #####
memlock

## update time when other coloumn was updated
explicit_defaults_for_timestamp=1

## set log use time
log_timestamps = SYSTEM

## current check position
# show variables like ‘%pct%';
innodb_max_dirty_pages_pct = 30

key_buffer_size = 512M
max_allowed_packet = 256M
table_open_cache = 1024
sort_buffer_size = 8M
net_buffer_length = 32K
read_buffer_size = 64M
read_rnd_buffer_size = 128M
max_connections = 4000
join_buffer_size = 8M
tmp_table_size = 128M
max_connect_errors = 40000
table_open_cache = 2000
thread_cache_size = 394
max_heap_table_size = 128M

[mysqldump]
quick
max_allowed_packet = 256M

[mysql]
no-auto-rehash
tee = history.log
default-character-set = utf8mb4

[myisamchk]
key_buffer_size = 1024M
sort_buffer_size = 8M
read_buffer = 8M
write_buffer = 8M

[mysqlhotcopy]
interactive-timeout
EOF
#===========================================


3. 主从设置
# master：（show master status）
create user slave@'%' identified by 'slave';
GRANT SELECT,REPLICATION CLIENT,REPLICATION SLAVE  ON *.* TO 'slavel'@'%' ;
# slave：(show slave status\G)
CHANGE MASTER TO MASTER_HOST='192.168.137.251',MASTER_USER='mescanal',MASTER_PASSWORD='mescanal',MASTER_PORT=3306,MASTER_AUTO_POSITION = 1;
CHANGE MASTER TO master_log_file='mysql-bin.000267',master_log_pos=824388534;
start slave;
# stop slave;
# 从库只同步库和跳过错误
replicate-do-db = ucenter     #同步的数据库名字
slave-skip-errors=all
