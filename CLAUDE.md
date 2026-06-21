# CLAUDE.md — Lab-Maid 项目规范

## 项目概述

基于 Ansible 的 homelab 自动化运维工具，通过自动化面板定时触发 playbook，以纯 push 模式管理目标主机。

**目标环境**：1 台 PVE 物理机（运行 3 VM + 2 LXC）+ 1 台 VPS，共约 6 个管理目标。

## 架构原则

- **纯 push 模式**：控制器通过 SSH 直接执行 playbook，不在目标主机部署 timer/cron
- **外部调度**：由自动化面板定时执行 `ansible-playbook` 命令
- **配置分散**：每个 role 管理自己的 `defaults/main.yml`，通用功能封装在 `common` role
- **通知在 task 级别**：由 `common` role 提供可复用的通知 task（Telegram + Email），各 playbook 显式调用

## 目录结构

```
lab-maid/
├── ansible.cfg
├── inventory.yml                    # 单文件清单（通过 .gitignore 排除，提交 .example.yml）
├── inventory.example.yml
├── requirements.yml
│
├── group_vars/
│   └── all/
│       ├── main.yml                 # 全局配置（.gitignore 排除，提交 main.example.yml）
│       └── vault.yml                # 密钥（.gitignore 排除，提交 vault.example.yml）
│
├── host_vars/                       # 主机特定配置（*.yml 排除，提交 *.example.yml）
│   ├── pve-host.example.yml
│   ├── db-vm.example.yml
│   └── vps.example.yml
│
├── roles/
│   ├── common/                      # 可复用基础设施（通知、日志、工具函数）
│   ├── zfs_snapshots/               # ZFS 定期快照
│   ├── smart_monitoring/            # SMART 数据采集与告警
│   └── postgres_backups/            # PostgreSQL 备份
│
├── playbooks/
│   ├── zfs-snapshot.yml             # 定时：ZFS 快照
│   ├── smart-collect.yml            # 定时：SMART 采集
│   ├── postgres-backup.yml          # 定时：数据库备份
│   └── adhoc/                       # 手动运维操作
│       ├── restore-db.yml
│       └── health-check.yml
│
├── .ansible-lint
├── .yamllint
├── .editorconfig
├── .pre-commit-config.yaml
├── .gitignore
├── .github/workflows/
│   └── lint.yml
├── LICENSE                          # MIT
└── README.md
```

## 命名规范

### Playbook：`<动作>-<对象>.yml`（kebab-case）
- `zfs-snapshot.yml`、`smart-collect.yml`、`postgres-backup.yml`
- 面板直接通过文件名调用，不需要额外映射

### Role：snake_case（Ansible 导入约束）
- `zfs_snapshots`、`smart_monitoring`、`postgres_backups`、`common`

### 变量：`<role>_<what>_<modifier>`（全 snake_case）
```yaml
zfs_snapshots_hourly_retention: 24
smart_monitoring_drives: [/dev/sda, /dev/sdb]
postgres_backups_databases: [nextcloud, gitea]
```

## 配置管理

### 个人信息保护 —— `.example` 模式

涉及个人信息的文件提交 `.example.yml` 示例，真实文件加入 `.gitignore`：

| 文件 | 处理 |
|------|------|
| `inventory.yml` | `inventory.example.yml` 提交，真实文件排除 |
| `host_vars/*.yml` | `*.example.yml` 提交，真实文件排除 |
| `group_vars/all/main.yml` | `main.example.yml` 提交，真实文件排除 |
| `group_vars/all/vault.yml` | `vault.example.yml` 提交，真实文件排除 |

所有配置文件统一走 `.example` 模式。即使不包含敏感信息，个人偏好（保留策略、阈值、压缩格式等）也不应随项目提交，避免更新时产生冲突或覆盖他人配置。

### 变量优先级（低→高）
1. `roles/<role>/defaults/main.yml` — 角色默认值，用户可覆盖
2. `group_vars/all/main.yml` — 全局配置
3. `group_vars/<group>.yml` — 按组覆盖
4. `host_vars/<host>.yml` — 按主机覆盖
5. `--extra-vars` — 命令行紧急覆盖

### `defaults/` vs `vars/` 的选择
- **defaults/main.yml**：可被外部覆盖的配置（用户应能调整的参数）
- **vars/main.yml**：角色内部常量（外部不应覆盖）

## 代码风格

- YAML 缩进：**2 空格**
- 每个 task **必须有 `name:`**（ansible-lint 强制）
- shell/command 模块仅在必要时使用，优先使用专用模块（`zfs`、`postgresql_db` 等）
- task 保持幂等性，二次执行 `changed=0`

## 提交规范

使用 **Conventional Commits**：
```
<type>(<scope>): <描述>

feat(zfs): add hourly snapshot retention
fix(smart): handle nvme drive attribute parsing
docs: add role documentation
chore(ci): add ansible-lint github action
refactor(common): extract notify logic to handler
```

Type：`feat` / `fix` / `docs` / `chore` / `refactor` / `test`

**提交署名**：末尾使用 `Generated-by: Claude Code (deepseek-v4-pro)`，不使用 Co-authored-by。

## 工具链

| 工具 | 用途 | 配置文件 |
|------|------|----------|
| ansible-lint | Ansible 最佳实践检查 | `.ansible-lint` |
| yamllint | YAML 语法与风格检查 | `.yamllint` |
| pre-commit | git commit 前自动检查 | `.pre-commit-config.yaml` |
| GitHub Actions | push/PR 时 CI 检查 | `.github/workflows/lint.yml` |

## Playbook 与主机组映射

| Playbook | 目标组 | 说明 |
|----------|--------|------|
| `zfs-snapshot.yml` | `pve` | 需要 `become: true`，ZFS 操作需 root |
| `smart-collect.yml` | `localhost` | smartctl 通过 `delegate_to` 到目标 |
| `postgres-backup.yml` | `db` | pg_dumpall 在 DB 主机本地执行，需要 `become: true` |
| `pg-start-backup.yml` | `db` | psql 单会话 backup_start/stop |
| `traffic-sync.yml` | `vps` | sqlite3+psql 在源主机本地执行 |

## 依赖项

```yaml
# requirements.yml
collections:
  - name: community.postgresql  # postgresql_query 模块
```

控制器端（仅 smart-collect 需要）：`pip install psycopg2-binary`

## 实践经验

### 时间戳：用 `date` 命令，不要用 `ansible_date_time`

`ansible.cfg` 中 `fact_caching` 会缓存 facts，`ansible_date_time` 在缓存期内不更新。备份/快照的命名时间戳必须用 `command: date +%Y_%m_%d-%H%M%S`。

### become 渗透

Playbook 级别的 `become: true` 会渗透到 `delegate_to: localhost` 的任务。需要在这些任务上显式加 `become: false`。

### zstd 覆盖

`zstd -o` 不覆盖已有文件，需要加 `-f` 参数。配合 `creates:` 实现幂等。

### Unix socket + peer 认证

PG 的 `peer` 认证需要 OS 用户和 PG 用户同名。用 `localhost` TCP 连接走 `scram-sha-256` 密码认证更可靠。psql 会自动协商服务端支持的最高安全级别，无需角色代码改动。

### ZFS 快照递归

父数据集 `recursive: true` 的子快照是独立 ZFS 实体，逐个 `zfs destroy`（不带 `-r`）删除即可。重叠数据集在删除列表里用 `unique | sort` 去重。

### Shell 可移植性

目标机 `/bin/sh` 可能是 dash（Debian/Ubuntu），避免 bash 专有语法：
- `10#08` → 用 `awk` 处理数字
- `[-+]` 字符类 → 用 `case` 匹配
- pipe 中的 `while` → subshell 吞变量，改用临时文件

## 扩展新功能的标准化流程

添加新任务类型时按以下步骤，不修改已有代码：
1. `mkdir -p roles/<new_task>/{defaults,tasks,templates}`
2. 写 `defaults/main.yml`（可配置参数）
3. 写 `tasks/main.yml`（执行逻辑）
4. 创建 `playbooks/<new-task>.yml`（入口 playbook）
5. 在 `group_vars/all/main.yml` 添加对应变量
6. 面板中配置定时命令

## 许可证

MIT
