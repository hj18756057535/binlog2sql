# binlog2sql 使用记录

## 背景

需要从 MySQL binlog 中恢复误操作数据，使用 binlog2sql 工具生成回滚 SQL。

---

## 环境说明

- binlog 文件从生产服务器下载到本地
- 连接的 MySQL 实例仅用于获取表结构（无 binlog 权限）
- 最终方案：**在服务器上直接运行工具**

---

## 踩坑记录

### 本地 Windows 方案（失败）

尝试在本地 Windows 运行，遇到以下问题：

1. **密码含特殊字符 `&`**：在 cmd/PowerShell 中直接传参会被截断，尝试了 `.ps1` 脚本、参数数组等方式均无法完全解决
2. **`pymysqlreplication` 不支持读取本地 binlog 文件**：`BinLogStreamReader` 本质是模拟 slave 向 master 请求 binlog 流，无法直接读本地文件
3. **连接的 MySQL 未开启 binlog**：`SHOW MASTER STATUS` 返回空，`SHOW MASTER LOGS` 无法使用

### 服务器方案（成功）

直接在服务器上运行，连接本机 MySQL，问题逐一解决：

| 错误 | 原因 | 解决 |
|------|------|------|
| `KeyError: 255` | PyMySQL 0.7.11 不支持 MySQL 8.0 字符集 | 升级到 `PyMySQL==0.9.3` |
| `No module named 'pymysql.util'` | PyMySQL 1.x 移除了该模块，与 mysql-replication 0.13 不兼容 | 锁定 `PyMySQL==0.9.3` |
| `UnicodeDecodeError: utf-8` | 数据库字符集为 `utf8mb4`，`reversed_lines` 用 utf-8 解码失败 | 改为 `block.decode("utf-8", errors="replace")` |

---

## 最终依赖版本

```
PyMySQL==0.9.3
wheel>=0.29.0
mysql-replication==0.13
```

---

## 最终运行命令

### 生成完整回滚 SQL（含 UPDATE/DELETE/INSERT 的反转）

```bash
python binlog2sql/binlog2sql.py \
  -h 127.0.0.1 -P 3306 \
  -u root -p'your_password' \
  -d cms \
  -t srm_cp_carbon_inventory srm_cp_carbon_inventory_content srm_cp_carbon_inventory_emission_summary \
  --start-file binlog.000001 \
  --start-datetime="2026-04-01 00:00:00" \
  --stop-datetime="2026-04-23 23:59:59" \
  --flashback \
  > rollback.sql
```

### 只恢复被删除的数据（DELETE → INSERT）

```bash
python binlog2sql/binlog2sql.py \
  -h 127.0.0.1 -P 3306 \
  -u root -p'your_password' \
  -d cms \
  -t srm_cp_carbon_inventory srm_cp_carbon_inventory_content srm_cp_carbon_inventory_emission_summary \
  --start-file binlog.000001 \
  --start-datetime="2026-04-01 00:00:00" \
  --stop-datetime="2026-04-23 23:59:59" \
  --flashback \
  --sql-type DELETE \
  > rollback_insert.sql
```

---

## flashback 模式说明

| 原始操作 | 回滚 SQL |
|----------|----------|
| DELETE | INSERT |
| INSERT | DELETE |
| UPDATE | UPDATE（新旧值互换） |

---

## MySQL 服务器要求

```ini
[mysqld]
server_id     = 1
log_bin       = /var/log/mysql/mysql-bin.log
binlog_format = row
binlog_row_image = full
```

用户权限：
```sql
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'user'@'host';
```

##  mysql的binlog工具

mysqlbinlog --no-defaults \
--base64-output=DECODE-ROWS -vv \
--start-datetime="2026-03-01 00:00:00" \
--stop-datetime="2026-04-23 23:59:59" \
/fii/data/database/mysql/data/binlog.000001 > clean_binlog.txt