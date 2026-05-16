# PostgreSQL 15 远程服务器安装文档

> 日期: 2026-05-16
> 适用环境: Ubuntu 远程服务器

## 一、安装 PostgreSQL 15

```bash
# 更新系统
sudo apt update && sudo apt upgrade -y

# 通过官方脚本添加 PostgreSQL APT 仓库（自动识别 Ubuntu 版本）
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh -y

# 确认可用版本
apt-cache search postgresql-15
# 输出: postgresql-15 - The World's Most Advanced Open Source Relational Database

# 安装
sudo apt install -y postgresql-15 postgresql-contrib-15

# 验证
sudo systemctl status postgresql
```

## 二、创建用户和数据库

```bash
# 修改 postgres 超级用户密码
sudo -u postgres psql -c "ALTER USER postgres PASSWORD 'lemon2judy';"

# 创建应用用户和数据库
sudo -u postgres psql <<SQL
CREATE USER platform WITH PASSWORD 'lemon2judy';
CREATE DATABASE share_compoter OWNER platform ENCODING 'UTF8';
GRANT ALL PRIVILEGES ON DATABASE share_compoter TO platform;
\c share_compoter
GRANT ALL ON SCHEMA public TO platform;
SQL
```

## 三、开启远程访问

```bash
# 1. 配置监听所有地址
sudo sed -i "s/#listen_addresses = 'localhost'/listen_addresses = '*'/" /etc/postgresql/15/main/postgresql.conf

# 2. 允许密码认证
echo "host    all             all             0.0.0.0/0               md5" | sudo tee -a /etc/postgresql/15/main/pg_hba.conf

# 3. 开放 5432 端口
sudo ufw allow 5432/tcp

# 4. 重启
sudo systemctl restart postgresql
```

## 四、云服务器安全组配置

如果服务器是云主机（阿里云/腾讯云/华为云/AWS 等），需要在**安全组**中放行 5432 端口。云安全组和服务器 `ufw` 是两层防火墙，两层都要放行：

```
你的电脑 → 云安全组（5432 放行）→ ufw（5432 放行）→ PostgreSQL
```

**操作步骤：**

1. 登录云厂商控制台
2. 找到对应服务器的 **安全组** 设置
3. 添加 **入站规则**：
   - 协议：TCP
   - 端口：5432
   - 来源：`0.0.0.0/0`（或填入你当前的公网 IP）

> 如果不想暴露给全网，可以填写 `ip4.me` 或 `ip.sb` 查到的当前公网 IP/32。但考虑到 IP 会变化，每次变了都需要更新安全组规则。

## 五、验证远程连接

```bash
# 在你本地开发机执行
psql -h <服务器IP> -U platform -d share_compoter -W
# 输入密码后进入 psql 即成功
```

## 六、安全加固

```bash
# 1. 限制最大连接数
sudo sed -i "s/max_connections = 100/max_connections = 50/" /etc/postgresql/15/main/postgresql.conf

# 2. 开启慢查询日志
sudo tee -a /etc/postgresql/15/main/postgresql.conf <<'EOF'
log_destination = 'stderr'
logging_collector = on
log_directory = '/var/log/postgresql'
log_statement = 'ddl'
log_duration = on
log_min_duration_statement = 1000
EOF

# 重启生效
sudo systemctl restart postgresql
```

## 七、自动备份

```bash
sudo mkdir -p /var/backups/postgresql
sudo tee /etc/cron.daily/pg-backup <<'SCRIPT'
#!/bin/bash
BACKUP_DIR="/var/backups/postgresql"
FILE="$BACKUP_DIR/share_compoter_$(date +%Y%m%d).sql.gz"
pg_dump -U platform share_compoter | gzip > "$FILE"
find "$BACKUP_DIR" -name "*.sql.gz" -mtime +7 -delete
SCRIPT
sudo chmod +x /etc/cron.daily/pg-backup
```

## 八、安全注意事项

- 应用用户 `platform` 使用强密码，不要用弱密码
- 不要暴露 `postgres` 超级用户给应用连接
- `platform` 用户权限限定在 `share_compoter` 库

## 九、管理平台用到的 PG 特性

| 特性 | 用途 |
|------|------|
| 聚合函数 `COUNT/SUM/AVG` | Dashboard 统计卡片 |
| 窗口函数 `ROW_NUMBER()/RANK()` | 设备排名、消费排行 |
| `date_trunc()` | 按天/周/月聚合营收 |
| `CASE WHEN` | 状态分布统计 |
| `FILTER` 子句 | 单次查询多维度聚合 |
| `MATERIALIZED VIEW` | Dashboard 数据预聚合 |
| `EXPLAIN ANALYZE` | 慢查询分析 |
| `pg_stat` 系统视图 | 监控连接数、表大小 |
