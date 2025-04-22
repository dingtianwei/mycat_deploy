# MySQL双主 + MyCat部署指南

## 步骤1: 启动容器

```bash
docker-compose up -d
```

## 步骤2: 配置MySQL主从复制

### 创建复制用户
在两台MySQL上创建复制用户:

```bash
# 在Master1上
docker exec -it mysql_master1 mysql -uroot -proot_password -e "CREATE USER 'repl'@'%' IDENTIFIED BY 'repl_password';"
docker exec -it mysql_master1 mysql -uroot -proot_password -e "GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';"
docker exec -it mysql_master1 mysql -uroot -proot_password -e "FLUSH PRIVILEGES;"

# 在Master2上
docker exec -it mysql_master2 mysql -uroot -proot_password -e "CREATE USER 'repl'@'%' IDENTIFIED BY 'repl_password';"
docker exec -it mysql_master2 mysql -uroot -proot_password -e "GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';"
docker exec -it mysql_master2 mysql -uroot -proot_password -e "FLUSH PRIVILEGES;"
```

### 配置Master1 -> Master2复制

```bash
# 获取Master1的状态
docker exec -it mysql_master1 mysql -uroot -proot_password -e "SHOW MASTER STATUS\G"
```

记下返回的 File 和 Position 值，然后在Master2上执行:

```bash
docker exec -it mysql_master2 mysql -uroot -proot_password -e "CHANGE MASTER TO MASTER_HOST='mysql_master1', MASTER_USER='repl', MASTER_PASSWORD='repl_password', MASTER_LOG_FILE='这里填写File的值', MASTER_LOG_POS=这里填写Position的值;"
docker exec -it mysql_master2 mysql -uroot -proot_password -e "START SLAVE;"
```

### 配置Master2 -> Master1复制

```bash
# 获取Master2的状态
docker exec -it mysql_master2 mysql -uroot -proot_password -e "SHOW MASTER STATUS\G"
```

记下返回的 File 和 Position 值，然后在Master1上执行:

```bash
docker exec -it mysql_master1 mysql -uroot -proot_password -e "CHANGE MASTER TO MASTER_HOST='mysql_master2', MASTER_USER='repl', MASTER_PASSWORD='repl_password', MASTER_LOG_FILE='这里填写File的值', MASTER_LOG_POS=这里填写Position的值;"
docker exec -it mysql_master1 mysql -uroot -proot_password -e "START SLAVE;"
```

### 验证复制状态

```bash
docker exec -it mysql_master1 mysql -uroot -proot_password -e "SHOW SLAVE STATUS\G"
docker exec -it mysql_master2 mysql -uroot -proot_password -e "SHOW SLAVE STATUS\G"
```

两台服务器的 Slave_IO_Running 和 Slave_SQL_Running 都应该显示 "Yes"。

## 步骤3: 为MyCat创建必要的用户权限

```bash
# 在Master1上创建用户
docker exec -it mysql_master1 mysql -uroot -proot_password -e "CREATE USER 'mycat_user'@'%' IDENTIFIED BY 'mycat_pass';"
docker exec -it mysql_master1 mysql -uroot -proot_password -e "GRANT ALL PRIVILEGES ON *.* TO 'mycat_user'@'%';"
docker exec -it mysql_master1 mysql -uroot -proot_password -e "FLUSH PRIVILEGES;"

# 在Master2上创建用户
docker exec -it mysql_master2 mysql -uroot -proot_password -e "CREATE USER 'mycat_user'@'%' IDENTIFIED BY 'mycat_pass';"
docker exec -it mysql_master2 mysql -uroot -proot_password -e "GRANT ALL PRIVILEGES ON *.* TO 'mycat_user'@'%';"
docker exec -it mysql_master2 mysql -uroot -proot_password -e "FLUSH PRIVILEGES;"
```

## 步骤4: 测试MyCat连接

连接到MyCat:

```bash
mysql -h127.0.0.1 -P8066 -umycat -pmycat -Dtestdb
```

## 步骤5: 测试高可用切换

1. 在MyCat中创建测试表并插入数据:

```sql
CREATE TABLE test (id INT PRIMARY KEY, name VARCHAR(20));
INSERT INTO test VALUES (1, 'Test Data 1');
```

2. 验证数据已同步到两台MySQL服务器。

3. 模拟主服务器故障:

```bash
docker stop mysql_master1
```

4. 在MyCat中继续写入数据:

```sql
INSERT INTO test VALUES (3, 'Test After Failover');
```

5. 验证数据写入到了Master2。

6. 恢复Master1，检查是否同步:

```bash
docker start mysql_master1
```

## 配置说明

MyCat配置中的关键参数:

- `balance="1"`: 表示读操作负载均衡类型，1代表读写分离
- `writeType="0"`: 表示写操作分发方式，0代表只在第一个writeHost写入
- `switchType="1"`: 表示自动切换，1代表自动切换
- `slaveThreshold="100"`: 从库延迟阈值

MySQL配置中的关键参数:

- `auto-increment-increment = 2`: 自增步长设为2
- `auto-increment-offset = 1/2`: 不同服务器使用不同偏移，避免主键冲突
