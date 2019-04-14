# 1.  yum方式安装数据库
rpm -ivh https://repo.mysql.com//mysql80-community-release-el7-2.noarch.rpm
yum install mysql-community-server -y
service mysqld start
service mysqld status
systemctl enable mysqld


# 2.  tar.xz安装（8.0.15）和起多个数据库实例
BASE_DIR=/mysql/
DATA_DIR=/data/
port=3306
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

cat > my.cnf <<EOF
[client]
port = 3306
socket = /data/mysql_3306/mysql.sock
default-character-set = utf8mb4

[mysqld]
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

character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
init_connect = 'SET NAMES utf8mb4'
character-set-client-handshake = FALSE

binlog_format = row
server-id = 33060
log-bin = /data/mysql_3306/binlog
expire_logs_days = 3

explicit_defaults_for_timestamp=1

log_timestamps = SYSTEM

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


mv my.cnf ${BASE_DIR}/mysql8/etc/
cp ${BASE_DIR}/mysql8/etc/my.cnf my.cnf
sed -i "s/3306/${port}/g" my.cnf
mv -f my.cnf ${BASE_DIR}/mysql8/etc/my${port}.cnf
cd ${BASE_DIR}
id mysql || (echo -e "please create mysql user " && exit 1)
chown -R mysql:mysql ${BASE_DIR}/mysql8
cd ${BASE_DIR}/mysql8
./bin/mysqld --defaults-file=${BASE_DIR}mysql8/etc/my${port}.cnf --basedir=${BASE_DIR} --datadir=${DATA_DIR}mysql_${port} --initialize-insecure
[ $? -ne 0 ] && echo -e "install mysql${port} failed"
echo -e "install mysql${port} successful"
echo -e "for convenience please create alias to /root/.bashrc"
echo "alias mysql${port}=\"${BASE_DIR}/mysql8/bin/mysql -S ${DATA_DIR}/mysql_${port}/mysql.sock\"" >> /etc/.bashrc
source /etc/.bashrc

./bin/mysqld_safe --defaults-file=${BASE_DIR}mysql8/etc/my3306.cnf --user=mysql &
 
mysql${port}
create user 'admin'@'%' identified by 'admin';
grant all privileges on *.* to admin@'%' with grant option;


## 3. 主从设置
# master：（show master status）
create user slave@'%' identified by 'slave';
GRANT SELECT,REPLICATION CLIENT,REPLICATION SLAVE  ON *.* TO 'slavel'@'%' ;
# slave：(show slave status\G)
CHANGE MASTER TO MASTER_HOST='192.168.137.251',MASTER_USER='mescanal',MASTER_PASSWORD='mescanal',MASTER_PORT=3306,MASTER_AUTO_POSITION = 1;
CHANGE MASTER TO master_log_file='mysql-bin.000267',master_log_pos=824388534;
start slave;
stop slave;
# 从库只同步库和跳过错误
replicate-do-db = ucenter     #同步的数据库名字
slave-skip-errors=all
