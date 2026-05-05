# Ansible 自动化编写规范

本项目遵循 [Red Hat Automation Good Practices](https://github.com/redhat-cop/automation-good-practices) 的核心原则，以确保基础设施代码的可维护性、可读性和灵活性。

## 1. 命名规范
- **任务命名**: 每个 `task` 必须有清晰、简洁且具备描述性的 `name`。
- **变量命名**: 使用小写字母和下划线 (`snake_case`)。全局变量应在 `group_vars/all.yml` 中定义，并带有清晰的前缀。

## 2. 模块化与独立运行 (Tags)
- **标签使用**: 所有逻辑功能块（如“换源”、“安装依赖”、“配置代理”）必须标记 `tags`。
- **独立性**: 支持通过 `--tags` 参数运行特定任务而不干扰其他流程。
  - 例如：`ansible-playbook site.yml --tags apt-mirror` 仅执行换源操作。

## 3. 幂等性 (Idempotency)
- 任务必须是幂等的。重复运行 Playbook 不应对系统产生副作用。
- 尽量使用 Ansible 内置模块（如 `replace`, `template`, `apt`），避免直接使用 `shell` 或 `command` 模块，除非没有对应功能的模块。

## 4. 变量与配置管理
- **禁止硬编码**: 所有的 IP 地址、域名、路径和版本号必须通过变量管理。
- **敏感信息**: 严禁将密码或 Token 写入 Playbook，使用 `ansible-vault` 或通过外部注入。

## 5. 错误处理
- 关键任务应考虑使用 `failed_when` 或 `ignore_errors`（需加注释说明原因）。
- 使用 `assert` 模块对前置条件（如磁盘空间、网络通透性）进行校验。

## 6. 目录结构
- 随着项目增长，应从单文件 Playbook 迁移到 `roles` 结构，实现逻辑剥离。
- `site.yml` 作为主入口，通过 `import_playbook` 串联各阶段任务。
