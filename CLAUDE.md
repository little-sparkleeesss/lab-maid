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
