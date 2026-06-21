# lab-maid

基于 Ansible 的 homelab 自动化运维工具，纯 push 模式，由自动化面板定时触发 playbook 管理目标主机。

## 架构

```
自动化面板 (panel / cron / 任意调度器)
        │
        ▼ 定时执行 ansible-playbook
控制器 (Ansible 控制节点)
        │
        ├── SSH ──> PVE 物理机 (ZFS 快照, SMART 采集)
        ├── SSH ──> PostgreSQL 虚拟机 (备份, 物理备份前置)
        └── SSH ──> VPS (vnstat 流量同步)
```

- **纯 push**：控制器通过 SSH 直接执行，目标主机不需要 cron/systemd timer
- **配置分散**：每个 role 独立管理 `defaults/main.yml`
- **外部调度**：面板执行 `ansible-playbook playbooks/<name>.yml`

## 快速开始

### 1. 安装依赖

```bash
# Ansible + 必需 collection
pip install ansible
ansible-galaxy collection install community.postgresql

# 控制器端（仅 smart-collect 需要）
pip install psycopg2-binary
```

### 2. 配置

```bash
# 复制示例配置
cp inventory.example.yml inventory.yml
cp group_vars/all/main.example.yml group_vars/all/main.yml
cp group_vars/all/vault.example.yml group_vars/all/vault.yml
cp host_vars/pve-host.example.yml host_vars/pve-host.yml
# ... 按需复制其他 host_vars

# 加密密钥文件
ansible-vault encrypt group_vars/all/vault.yml
```

### 3. 初始化目标主机

```bash
ansible-playbook playbooks/bootstrap.yml
```

### 4. 配置面板定时任务

| 任务 | 命令 | 建议频率 |
|------|------|----------|
| ZFS 快照 | `ansible-playbook playbooks/zfs-snapshot.yml` | 每小时 |
| SMART 采集 | `ansible-playbook playbooks/smart-collect.yml` | 每 6 小时 |
| 数据库备份 | `ansible-playbook playbooks/postgres-backup.yml` | 每天 |
| 物理备份前置 | `ansible-playbook playbooks/pg-start-backup.yml` | 备份窗口前 |
| 流量同步 | `ansible-playbook playbooks/traffic-sync.yml` | 每小时 |

## 任务说明

### ZFS 快照 (`zfs_snapshots`)

在 PVE 主机上创建 ZFS 数据集快照，按保留天数自动清理过期快照。

```yaml
zfs_snapshots_datasets:
  - name: tank/data
    recursive: true
  - name: tank/data/sub
    recursive: false
zfs_snapshots_prefix: "autosnap"
zfs_snapshots_retention_days: 30
```

- 支持递归快照（`-r`）
- 单次最多删除 100 个快照（安全上限）
- 递归父和子数据集重叠时自动去重

### SMART 监控 (`smart_monitoring`)

在 PVE 主机上运行 `smartctl`，采集磁盘 SMART 数据存入 PostgreSQL。

```yaml
smart_monitoring_target: "pve-host"
smart_monitoring_drives:
  - /dev/sda
  - /dev/nvme0
smart_monitoring_pg_table: "smart_stats"
```

- 支持 ATA 和 NVMe 设备
- 完整 `smartctl -a -j` JSON 存入 `JSONB` 列
- 自动建表（幂等 `CREATE TABLE IF NOT EXISTS`）

### PostgreSQL 备份 (`postgres_backups`)

在 DB 主机本地执行 `pg_dumpall | zstd`，导出全部数据库、角色和表空间，压缩为单个文件。

```yaml
postgres_backups_user: "postgres"
postgres_backups_host: ""   # 留空 = Unix socket
postgres_backups_compression: "auto"  # zstd > gzip > xz 自动检测
```

- 可选 NAS 推送：备份文件通过控制器中转 `fetch` → `scp` 到 NAS，推完删本地
- 压缩工具自动检测：`zstd` → `gzip` → `pigz` → `xz`
- 各压缩工具独立可配压缩级别

### PG 物理备份前置 (`pg_start_backup`)

在单个 psql 会话中执行 `pg_backup_start()` → 等待 → `pg_backup_stop()`，为外部物理备份工具提供一致的备份窗口。

```yaml
pg_start_backup_label: "scheduled_backup"
pg_start_backup_delay_seconds: 600  # 10 分钟备份窗口
```

### 流量同步 (`traffic_monitor`)

从 vnstat SQLite 数据库增量同步流量数据到 PostgreSQL。

```yaml
traffic_monitor_source_path: /var/lib/vnstat/vnstat.db
traffic_monitor_pg_table: "network_traffic"
```

- 增量同步：`ON CONFLICT (timestamp) DO NOTHING`
- 自动时区转换（本地 → UTC）

## 配置层级

变量优先级（低 → 高）：

1. `roles/<role>/defaults/main.yml` — 默认值
2. `group_vars/all/main.yml` — 全局
3. `group_vars/<group>.yml` — 按组
4. `host_vars/<host>.yml` — 按主机
5. `--extra-vars` — 命令行

敏感信息（密码、token）放 `group_vars/all/vault.yml`，用 `ansible-vault encrypt` 加密。

## 目录结构

```
lab-maid/
├── ansible.cfg
├── inventory.yml           # 从 .example.yml 复制
├── group_vars/all/         # 从 .example.yml 复制
├── host_vars/              # 从 .example.yml 复制
├── roles/
│   ├── common/             # 基础设施（通知、工具安装）
│   ├── zfs_snapshots/      # ZFS 快照
│   ├── smart_monitoring/   # SMART 采集
│   ├── postgres_backups/   # PG 逻辑备份
│   ├── pg_start_backup/    # PG 物理备份前置
│   └── traffic_monitor/    # 流量同步
├── playbooks/
│   ├── zfs-snapshot.yml
│   ├── smart-collect.yml
│   ├── postgres-backup.yml
│   ├── pg-start-backup.yml
│   ├── traffic-sync.yml
│   └── adhoc/
└── assets/tools/           # 预编译工具（如静态 zstd）
```

## 依赖项

| 依赖 | 用途 |
|------|------|
| Ansible ≥ 2.14 | 核心引擎 |
| `community.postgresql` | `postgresql_query` 模块 |
| `psycopg2-binary` | 控制器端 PG 连接（仅 smart-collect） |
| PVE 主机：`zfs`, `smartmontools` | ZFS 快照、SMART 采集 |
| DB 主机：`pg_dumpall`, `zstd`/`gzip` | 数据库备份 |
| VPS 主机：`vnstat`, `sqlite3` | 流量数据源 |

## 开发

```bash
# Lint
ansible-lint -c .ansible-lint
yamllint -c .yamllint .

# 语法检查
ansible-playbook playbooks/zfs-snapshot.yml --syntax-check

# 预提交
pre-commit run --all-files
```

## 许可证

MIT
