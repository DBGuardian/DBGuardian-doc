# DBGuardian PostgreSQL 自动追赶说明

## 1. 背景

在 MySQL 场景里，原主恢复后通常可以通过重建复制源的方式继续追赶。

PostgreSQL 不一样。一旦从库已经执行提升，原主恢复回来后通常不再与当前主库处于同一条时间线，不能简单依靠 JDBC 层几条复制 SQL 继续追赶。常见恢复路径是：

- `pg_rewind`
- 或重新做一份副本

因此，DBGuardian 对 PostgreSQL 的自动追赶采用的是“框架判定恢复时机 + 外部脚本执行 `pg_rewind`”的方式。

---

## 2. 目标

当前这套能力解决的是下面这个场景：

1. 原主库故障
2. 原从库被提升为新主库
3. 原主库恢复上线
4. DBGuardian 检测到需要追赶
5. 自动按当前操作系统执行恢复脚本，或通过远程代理命令把脚本投递到目标数据库节点执行
6. 原主库通过 `pg_rewind` 变成当前主库的 standby

默认策略是：

- 保留当前已升主节点继续做主库
- 原主追赶完成后转为从库
- 不自动把主角色切回原主

如果显式配置 `keep-promoted-primary=false`，则策略会变成：

- 原主先追赶为当前主库的 standby
- 原主追平后再提升回主库
- 当前主再执行一次追赶，回挂到原主下面

---

## 3. 当前结构

当前自动追赶链路由三层组成：

### 3.1 方言层

`dbguardian-dialect-postgresql`

职责：

- 提供 PostgreSQL 健康检查与复制状态观测能力
- 内置跨平台恢复脚本模板
- 提供以下资源：
  - `pg-rewind-recover.sh`
  - `pg-rewind-recover.bat`

### 3.2 Spring 支撑层

`dbguardian-spring-support`

职责：

- 在恢复编排阶段识别是否进入 PostgreSQL 追赶窗口
- 判断当前数据库类型是否为 `postgresql`
- 判断是否启用 PostgreSQL 自动追赶
- 判断当前是本地执行还是远程代理执行
- 判断是否需要执行回切链路
- 执行对应脚本

### 3.3 Boot 配置层

`dbguardian-boot2-starter`
`dbguardian-boot3-starter`

职责：

- 绑定 `spring.datasource.recovery.postgresql.*` 配置
- 将配置映射到恢复执行器

---

## 4. 触发时机

PostgreSQL 自动追赶不是在正常复制观测阶段执行，而是在故障恢复编排阶段执行。

触发链路可以理解为：

```text
主库故障
-> 从库升主
-> 集群进入 SLAVE_PROMOTED
-> 原主恢复上线
-> 进入 RECOVERING_ORIGINAL_MASTER
-> 判断数据库类型为 postgresql
-> 判断已启用 PostgreSQL 自动追赶
-> 执行 pg_rewind 脚本
-> 原主变为当前主库的 standby
-> 集群回到稳定主从结构
```

当 `keep-promoted-primary=false` 时，链路会继续向下走：

```text
原主追赶完成
-> 提升原主
-> 当前主进入 RECONNECTING_PROMOTED_SLAVE
-> 当前主执行 pg_rewind
-> 当前主回挂到原主下方
-> 集群恢复为原主/原从角色分布
```

触发的必要条件：

- 当前恢复节点是原主
- 当前活动主库是故障期间被提升的新主
- `spring.datasource.recovery.enabled=true`
- `spring.datasource.recovery.auto-failback=true`
- `spring.datasource.recovery.postgresql.enabled=true`
- `spring.datasource.recovery.postgresql.data-directory` 已配置
- 已配置复制账号

---

## 5. 执行位置与脚本选择

执行器会根据当前 JVM 所在系统自动选择脚本：

- Windows
  - 执行 `pg-rewind-recover.bat`
- Linux / macOS
  - 执行 `pg-rewind-recover.sh`

如果你没有显式配置 `script-path`，默认会使用方言模块内置脚本模板。

如果你配置了 `script-path`，则优先执行你提供的脚本。

关键语义不变：

- 脚本本身始终是“在目标 PostgreSQL 节点本机执行”
- 本地模式下，由应用进程直接拉起这份脚本
- 远程模式下，由应用进程先执行 `remote-executor-command`，再由代理命令在目标 PostgreSQL 节点本机拉起脚本

---

## 6. 配置项

配置入口：

```yaml
spring:
  datasource:
    replication:
      master-host: 192.168.3.150
      master-port: 5432
      master-user: repl
      master-password: ${DBGUARDIAN_REPL_PASSWORD}
      auto-reconnect: true
    recovery:
      enabled: true
      auto-failback: true
      catchup-timeout-seconds: 300
      catchup-check-interval-seconds: 2
      postgresql:
        enabled: true
        keep-promoted-primary: false

        # 原主恢复为从库时，在原主节点执行。
        data-directory: /var/lib/postgresql/16/main
        config-file: /etc/postgresql/16/main/postgresql.conf
        script-path: /opt/dbguardian/scripts/pg-rewind-recover.sh
        remote-execution-enabled: true
        remote-executor-command: >
          ssh ubuntu@192.168.3.150 "sudo -n -u postgres env {env_sh} bash {script_path_sh}"
        remote-executor-password: ${DBGUARDIAN_SSH_PASSWORD}
        pg-ctl-command: /usr/lib/postgresql/16/bin/pg_ctl
        psql-command: /usr/lib/postgresql/16/bin/psql
        pg-rewind-command: /usr/lib/postgresql/16/bin/pg_rewind
        local-host: 127.0.0.1
        local-port: 5432
        local-database: appdb
        primary-slot-name: appdb_slot
        extra-primary-conn-info: "target_session_attrs=read-write"
        startup-wait-seconds: 60
        command-timeout-seconds: 900

        # 普通从库恢复或自动回切时，在恢复从库节点执行。
        failback-data-directory: /var/lib/postgresql/16/main
        failback-config-file: /etc/postgresql/16/main/postgresql.conf
        failback-script-path: /opt/dbguardian/scripts/pg-rewind-recover.sh
        failback-remote-execution-enabled: true
        failback-remote-executor-command: >
          ssh ubuntu@192.168.3.151 "sudo -n -u postgres env {env_sh} bash {script_path_sh}"
        failback-remote-executor-password: ${DBGUARDIAN_SSH_PASSWORD}
        failback-pg-ctl-command: /usr/lib/postgresql/16/bin/pg_ctl
        failback-psql-command: /usr/lib/postgresql/16/bin/psql
        failback-pg-rewind-command: /usr/lib/postgresql/16/bin/pg_rewind
        failback-local-host: 127.0.0.1
        failback-local-port: 5432
        failback-local-database: appdb
        failback-primary-slot-name: appdb_slot
        failback-extra-primary-conn-info: "target_session_attrs=read-write"
        failback-startup-wait-seconds: 60
        failback-command-timeout-seconds: 900
```

### 6.1 参数说明

| 配置项 | 作用 |
|---|---|
| `enabled` | 是否启用 PostgreSQL 自动追赶 |
| `keep-promoted-primary` | 是否保留当前已升主节点继续作为主库，默认 `true` |
| `data-directory` | 原主 PostgreSQL 数据目录 |
| `config-file` | 原主恢复时 `pg_ctl` 启动使用的 PostgreSQL 配置文件路径 |
| `script-path` | 恢复脚本路径；本地模式是本机路径，远程模式是目标数据库节点上的脚本路径 |
| `remote-executor-password` | 远程 SSH/Plink 密码；推荐改用密钥认证，不能作为 sudo 密码 |
| `pg-ctl-command` | `pg_ctl` 命令路径 |
| `psql-command` | `psql` 命令路径 |
| `pg-rewind-command` | `pg_rewind` 命令路径 |
| `local-host` | 恢复后本地自检连接地址 |
| `local-port` | 恢复后本地自检端口 |
| `local-database` | 恢复后本地自检数据库名 |
| `primary-slot-name` | standby 追主时使用的 slot 名称，可选 |
| `extra-primary-conn-info` | 追加到 `primary_conninfo` 的额外参数 |
| `startup-wait-seconds` | 恢复后等待节点进入 recovery 模式的时长 |
| `command-timeout-seconds` | 整个脚本执行超时时间 |
| `remote-execution-enabled` | 是否启用远程代理执行 |
| `remote-executor-command` | 远程代理命令模板，负责把脚本调度到目标数据库节点执行 |
| `failback-data-directory` | 从库恢复或回切阶段处理的目标节点数据目录；未配置时复用 `data-directory` |
| `failback-config-file` | 从库恢复或回切阶段 `pg_ctl` 使用的 PostgreSQL 配置文件；未配置时复用 `config-file` |
| `failback-script-path` | 从库恢复或回切阶段脚本路径；未配置时复用 `script-path` |
| `failback-remote-executor-password` | 从库恢复或回切阶段的远程 SSH/Plink 密码；推荐改用密钥认证，不能作为 sudo 密码 |
| `failback-pg-ctl-command` | 回切阶段 `pg_ctl` 路径；未配置时复用主恢复配置 |
| `failback-psql-command` | 回切阶段 `psql` 路径；未配置时复用主恢复配置 |
| `failback-pg-rewind-command` | 回切阶段 `pg_rewind` 路径；未配置时复用主恢复配置 |
| `failback-local-host` | 回切阶段本地自检地址；未配置时复用主恢复配置 |
| `failback-local-port` | 回切阶段本地自检端口；未配置时按节点 JDBC 端口推导 |
| `failback-local-database` | 回切阶段本地自检数据库；未配置时复用主恢复配置 |
| `failback-primary-slot-name` | 回切阶段写入的 slot 名称；未配置时复用主恢复配置 |
| `failback-extra-primary-conn-info` | 回切阶段写入 `primary_conninfo` 的附加参数 |
| `failback-startup-wait-seconds` | 回切阶段等待节点进入 recovery 的时长；未配置时复用主恢复配置 |
| `failback-command-timeout-seconds` | 回切阶段脚本超时；未配置时复用主恢复配置 |
| `failback-remote-execution-enabled` | 回切阶段是否启用远程代理执行 |
| `failback-remote-executor-command` | 回切阶段远程代理命令模板 |

### 6.2 远程命令模板占位符

执行器会在真正启动命令前，把恢复参数渲染进模板。当前支持的占位符如下：

| 占位符 | 作用 |
|---|---|
| `{script}` | 经过 shell 引用后的脚本路径，等价于 `{script_path_sh}` |
| `{script_path}` | 原始脚本路径；仅在路径不含空格、引号等特殊字符时适合直接使用 |
| `{script_path_sh}` | 适配 POSIX shell 的脚本路径引用 |
| `{env}` | 适配 POSIX shell 的环境变量导出片段，等价于 `{env_sh}` |
| `{env_sh}` | 适配 POSIX shell 的环境变量导出片段 |

`{env_cmd}`、`{env_ps}`、`{script_path_cmd}` 和 `{script_path_ps}` 不是当前执行器支持的占位符，不能写入配置。

这些环境变量里已经包含：

- 旧主/当前主数据目录
- 新主地址和端口
- 复制用户与密码
- `pg_ctl`、`psql`、`pg_rewind` 路径
- 本地自检地址、端口、数据库
- slot 名称
- 额外 `primary_conninfo`
- 启动等待时间和命令超时

---

## 7. Windows 本地执行示例

```yaml
spring:
  datasource:
    recovery:
      enabled: true
      auto-failback: true
      postgresql:
        enabled: true
        keep-promoted-primary: true
        data-directory: D:/pgsql/data
        script-path: C:/pgsql/scripts/pg-rewind-recover.bat
        pg-ctl-command: C:/pgsql/bin/pg_ctl.exe
        psql-command: C:/pgsql/bin/psql.exe
        pg-rewind-command: C:/pgsql/bin/pg_rewind.exe
        local-host: 127.0.0.1
        local-port: 5432
        local-database: postgres
        primary-slot-name: appdb_slot
        extra-primary-conn-info: "target_session_attrs=read-write"
        startup-wait-seconds: 60
        command-timeout-seconds: 900
```

适用特点：

- PostgreSQL 安装在 Windows 主机
- 应用所在进程可以直接访问 PostgreSQL 数据目录
- 应用所在进程可以直接执行 `pg_ctl.exe`、`psql.exe`、`pg_rewind.exe`

---

## 8. Linux / macOS 本地执行示例

```yaml
spring:
  datasource:
    recovery:
      enabled: true
      auto-failback: true
      postgresql:
        enabled: true
        keep-promoted-primary: true
        data-directory: /var/lib/postgresql/16/main
        script-path: /opt/dbguardian/scripts/pg-rewind-recover.sh
        pg-ctl-command: /usr/lib/postgresql/16/bin/pg_ctl
        psql-command: /usr/lib/postgresql/16/bin/psql
        pg-rewind-command: /usr/lib/postgresql/16/bin/pg_rewind
        local-host: 127.0.0.1
        local-port: 5432
        local-database: postgres
        primary-slot-name: appdb_slot
        extra-primary-conn-info: "target_session_attrs=read-write"
        startup-wait-seconds: 60
        command-timeout-seconds: 900
```

适用特点：

- PostgreSQL 安装在 Linux 或 macOS 主机
- 应用所在进程可以直接访问 PostgreSQL 数据目录
- 应用所在进程可以直接执行 `pg_ctl`、`psql`、`pg_rewind`

---

## 9. 远程代理执行示例

### 9.1 Linux SSH 示例

```yaml
spring:
  datasource:
    recovery:
      enabled: true
      auto-failback: true
      postgresql:
        enabled: true
        keep-promoted-primary: true
        data-directory: /var/lib/postgresql/16/main
        script-path: /opt/dbguardian/scripts/pg-rewind-recover.sh
        remote-execution-enabled: true
        remote-executor-command: >
          ssh postgres@10.10.10.21 "{env_sh} bash {script_path_sh}"
        pg-ctl-command: /usr/lib/postgresql/16/bin/pg_ctl
        psql-command: /usr/lib/postgresql/16/bin/psql
        pg-rewind-command: /usr/lib/postgresql/16/bin/pg_rewind
        local-host: 127.0.0.1
        local-port: 5432
        local-database: postgres
        primary-slot-name: appdb_slot
        extra-primary-conn-info: "target_session_attrs=read-write"
        startup-wait-seconds: 60
        command-timeout-seconds: 900
```

适用特点：

- 应用机与 PostgreSQL 节点分离
- 应用机可以通过 SSH 登录数据库节点
- `script-path` 指向数据库节点本机可访问的脚本路径

### 9.2 Windows PowerShell 示例

```yaml
spring:
  datasource:
    recovery:
      enabled: true
      auto-failback: true
      postgresql:
        enabled: true
        keep-promoted-primary: true
        data-directory: D:/pgsql/data
        script-path: C:/pgsql/scripts/pg-rewind-recover.bat
        remote-execution-enabled: true
        remote-executor-command: >
          powershell -Command "& { Invoke-Command -ComputerName 10.10.10.31 -ScriptBlock { param($script) {env_ps} & $script } -ArgumentList {script_path_ps} }"
        pg-ctl-command: C:/pgsql/bin/pg_ctl.exe
        psql-command: C:/pgsql/bin/psql.exe
        pg-rewind-command: C:/pgsql/bin/pg_rewind.exe
        local-host: 127.0.0.1
        local-port: 5432
        local-database: postgres
        startup-wait-seconds: 60
        command-timeout-seconds: 900
```

适用特点：

- 应用机通过 PowerShell/WinRM 代理远程执行
- 执行器负责把环境变量片段拼进模板
- 最终脚本仍在目标数据库节点本机运行

### 9.3 回切阶段远程执行示例

当你希望旧主追赶完成后自动切回原主，并且两边节点都需要远程代理执行时，可以这样配置：

```yaml
spring:
  datasource:
    recovery:
      enabled: true
      auto-failback: true
      postgresql:
        enabled: true
        keep-promoted-primary: false
        data-directory: /var/lib/postgresql/16/main
        script-path: /opt/dbguardian/scripts/pg-rewind-recover.sh
        remote-execution-enabled: true
        remote-executor-command: >
          ssh postgres@10.10.10.21 "{env_sh} bash {script_path_sh}"
        pg-ctl-command: /usr/lib/postgresql/16/bin/pg_ctl
        psql-command: /usr/lib/postgresql/16/bin/psql
        pg-rewind-command: /usr/lib/postgresql/16/bin/pg_rewind
        local-host: 127.0.0.1
        local-port: 5432
        local-database: postgres
        primary-slot-name: appdb_slot
        command-timeout-seconds: 900
        failback-data-directory: /var/lib/postgresql/16/main
        failback-script-path: /opt/dbguardian/scripts/pg-rewind-recover.sh
        failback-remote-execution-enabled: true
        failback-remote-executor-command: >
          ssh postgres@10.10.10.22 "{env_sh} bash {script_path_sh}"
        failback-pg-ctl-command: /usr/lib/postgresql/16/bin/pg_ctl
        failback-psql-command: /usr/lib/postgresql/16/bin/psql
        failback-pg-rewind-command: /usr/lib/postgresql/16/bin/pg_rewind
        failback-local-host: 127.0.0.1
        failback-local-port: 5432
        failback-local-database: postgres
        failback-primary-slot-name: appdb_slot
        failback-extra-primary-conn-info: "target_session_attrs=read-write"
        failback-startup-wait-seconds: 60
        failback-command-timeout-seconds: 900
```

这里可以把它理解成两段链路：

- 第一段：旧主恢复后，被追赶成当前主的 standby
- 第二段：旧主重新升为主，当前主再被追赶回旧主下方

---

## 10. 脚本执行的大致流程

内置脚本做的事情可以概括为：

```text
检查新主是否可写
-> 检查旧主数据目录是否存在
-> 停止旧主进程
-> 先做 pg_rewind dry-run
-> 执行 pg_rewind
-> 写入 primary_conninfo / primary_slot_name
-> 创建 standby.signal
-> 启动旧主
-> 等待旧主进入 recovery 模式
-> 成功后退出
```

这条链路的第一目标不是“切回原主”，而是“让目标节点重新加入复制链，成为当前主库的从库”。

当 `keep-promoted-primary=false` 时，框架会在这一步成功后继续走回切编排，而不是再次简单地重复同一次恢复脚本调用语义。

---

## 11. 前置条件

要让自动追赶真正可用，至少要满足这些条件：

### 11.1 PostgreSQL 层前提

- 当前主库是由原从库提升而来
- 原主与当前主之间存在共同历史
- `pg_rewind` 所需能力已满足
- 当前主库仍保留恢复所需 WAL
- 当前主库允许 replication 连接

### 11.2 运行环境前提

- 本地模式下，应用进程有权限执行脚本
- 本地模式下，应用进程有权限调用 PostgreSQL 命令
- 本地模式下，应用进程有权限访问旧主数据目录
- 远程模式下，应用进程有权限执行代理命令
- 远程模式下，代理命令有能力在目标数据库节点本机执行脚本
- 脚本路径和命令路径真实可用

### 11.3 配置前提

- 已配置 `replication.master-user`
- 已配置 `replication.master-password`
- 已启用 `recovery.postgresql.enabled`
- 已配置 `data-directory`
- 远程模式下已配置 `remote-executor-command`

---

## 12. 运行边界

这套能力有明确边界。

### 12.1 能力边界

当前支持的是：

- 框架自动识别 PostgreSQL 追赶窗口
- 自动按系统选择脚本
- 支持本地执行与远程代理执行两种模式
- 自动执行 `pg_rewind`
- 自动将原主转为当前主库的 standby
- 在 `keep-promoted-primary=false` 时继续执行回切闭环

当前不负责：

- 不自动管理 `pg_hba.conf`
- 可按配置确保物理 replication slot 存在；生产环境仍建议预创建并监控 WAL 积压
- Patroni / repmgr / pg_auto_failover 集成
- `pg_rewind` 失败后的全量重建

### 12.2 部署边界

当前实现支持两类部署：

- 应用进程与数据库实例同机，直接本地执行
- 应用进程与数据库实例分离，但应用机可以通过 SSH、WinRM、自定义 Agent 或运维平台把脚本调度到数据库节点本机执行

不建议把 `script-path` 配成“应用机路径但数据库机不存在”的形式。对远程模式来说，`script-path` 应当始终表示目标数据库节点本机的脚本路径。

---

## 13. 失败处理建议

如果 `pg_rewind` 失败，不建议强行继续恢复闭环。

推荐处理方式：

1. 保持当前已升主节点继续提供写服务
2. 原主不要重新接业务流量
3. 标记原主需要人工处理
4. 视情况选择：
   - 重新执行 `pg_rewind`
   - 直接全量重建副本

也就是说，失败后的原则是：

- 主链路不断写
- 旧主不误上线
- 不做高风险自动回切

---

## 14. 推荐使用方式

建议把 PostgreSQL 自动追赶理解成下面这个职责分层：

### 14.1 DBGuardian 负责

- 检测故障
- 执行升主
- 判断恢复阶段
- 触发自动追赶
- 在需要时继续编排回切
- 维护运行态阶段变化

### 14.2 脚本负责

- 停目标节点
- 做 `pg_rewind`
- 写 standby 配置
- 拉起目标节点
- 验证目标节点进入 recovery 状态

### 14.3 运维侧负责

- 提供可靠的 PostgreSQL 命令路径
- 提供正确的数据目录权限
- 保障 WAL 保留策略
- 保障 replication 用户权限
- 提供可用的远程代理通道

---

## 15. 推荐上线顺序

建议按下面顺序接入：

1. 先在测试环境验证脚本可独立运行
2. 再验证 DBGuardian 能否准确进入恢复阶段
3. 如需跨机执行，先单独验证远程代理命令模板
4. 再打开 `recovery.postgresql.enabled`
5. 先保留 `keep-promoted-primary=true`
6. 观察恢复完成后原主是否稳定转为 standby
7. 最后再评估是否启用 `keep-promoted-primary=false` 的自动回切闭环

---

## 16. 一句话总结

PostgreSQL 自动追赶的核心思路不是“像 MySQL 一样重挂复制”，而是：

- DBGuardian 负责判断什么时候该追赶、什么时候该回切
- `pg_rewind` 脚本负责把目标节点恢复成当前主库的从库
- 远程模式只是多了一层代理调度，脚本语义本身不变
- 默认保留已升主节点继续做主库，降低回切风险

---

## 17. 从库恢复部署与运维配置

本节用于配置正常主从运行期间的从库宕机恢复。它与原主故障转移后的自动追赶使用同一套 `pg_rewind` 脚本，但恢复目标不同。

### 17.1 从库恢复链路与数据方向

```text
从库宕机
-> 健康检查移除从库读池
-> 读请求路由到当前主库
-> 从库进程恢复
-> 在恢复从库节点执行 pg_rewind
-> 恢复从库以当前主库为 WAL 源启动
-> 健康检查确认 streaming
-> 从库重新加入读池
```

数据方向固定为：

```text
恢复从库 -> 复制 -> 当前主库
```

`pg_rewind`、`primary_conninfo` 与 `standby.signal` 都在恢复从库节点执行。DBGuardian 不会让当前主库追赶恢复从库。恢复脚本成功后，DBGuardian 仍会等待下一次健康检查确认 WAL receiver 状态为 `streaming`，之后才恢复从库读路由。

### 17.1.1 读路由状态与恢复门槛

从库故障恢复由健康检查驱动，不会在某个业务请求失败时同步重试主库。状态和路由关系如下：

| 阶段 | 从库状态 | 读请求路由 | 说明 |
|---|---|---|---|
| 正常运行 | 可用，流复制为 `streaming` | 从库 | `activeReadNodeId` 指向从库 |
| 从库宕机或复制异常 | 从可用从库池移除 | 主库 | 禁用从库读，`activeReadNodeId` 切换为当前主库 |
| 从库连接恢复，复制未确认 | 仍不在可用从库池 | 主库 | 执行 `pg_rewind` 重建复制；脚本成功本身不能恢复读路由 |
| 后续健康检查确认复制正常 | 重新加入可用从库池 | 从库 | PostgreSQL 返回 `streaming` 后，解除从库读阻断并切回从库 |

完整状态转换：

```text
正常: read -> slave
-> 从库连接失败或流复制异常
-> 移除 slave 可用池 + slaveReadsBlocked=true + activeReadNodeId=master
-> read -> master
-> 从库连接恢复
-> 在从库节点执行 pg_rewind，primary_conninfo 指向当前 master
-> 保持 read -> master
-> 下一轮健康检查确认 pg_stat_wal_receiver.status=streaming
-> 加回 slave 可用池 + slaveReadsBlocked=false + activeReadNodeId=slave
-> read -> slave
```

实现上的关键约束：

- 只要从库不在可用从库池，读请求不会被路由到从库。
- 复制重建命令返回成功只表示脚本执行成功，不表示 PostgreSQL 已建立流复制。
- 必须由**后续**健康检查读取 PostgreSQL 复制状态并确认 `streaming`，才允许恢复从库读路由。
- 若 `pg_rewind`、启动、认证、`pg_hba.conf` 或 WAL receiver 任一步失败，从库继续留在读池外，读请求持续路由到主库。
- 主库故障转移与本节的从库读降级是独立流程；本节假设当前主库仍然可用。

验证从库恢复后是否允许回流：

```sql
SELECT pg_is_in_recovery();
SELECT status FROM pg_stat_wal_receiver;
```

只有 `pg_is_in_recovery()` 为 `true` 且 `status` 为 `streaming` 时，才满足恢复从库读流量的条件。

### 17.2 节点职责和执行位置

以两节点为例：

| 节点 | 地址 | 故障后的角色 | 执行恢复脚本的位置 |
|---|---|---|---|
| `pg-master` | `192.168.3.150` | 当前主库 | 不执行从库重建脚本 |
| `pg-slave` | `192.168.3.151` | 恢复从库 | 必须在此节点执行脚本 |
| `app` | 应用服务器 | DBGuardian 进程 | 通过 SSH 调度 `pg-slave` |

脚本传入的关键环境变量：

| 环境变量 | 含义 |
|---|---|
| `DBGUARDIAN_OLD_PGDATA` | 恢复从库的数据目录 |
| `DBGUARDIAN_NEW_PRIMARY_HOST` | 当前主库地址 |
| `DBGUARDIAN_NEW_PRIMARY_PORT` | 当前主库端口 |
| `DBGUARDIAN_REPL_USER` | 复制用户名 |
| `DBGUARDIAN_REPL_PASS` | 复制密码 |

### 17.3 PostgreSQL 前置条件

恢复从库节点必须能执行以下命令。以 PostgreSQL 16 和 Debian/Ubuntu 为例：

```bash
sudo -u postgres /usr/lib/postgresql/16/bin/pg_ctl --version
sudo -u postgres /usr/lib/postgresql/16/bin/psql --version
sudo -u postgres /usr/lib/postgresql/16/bin/pg_rewind --version
```

实际安装路径不同时，必须将绝对路径填入 `application.yml`。

`pg_rewind` 要求启用数据校验和，或启用 `wal_log_hints`。推荐在每一个可能成为恢复从库的节点执行：

```sql
ALTER SYSTEM SET wal_log_hints = 'on';
```

随后重启 PostgreSQL：

```bash
sudo systemctl restart postgresql
```

确认至少满足一项：

```sql
SHOW wal_log_hints;
SHOW data_checksums;
```

```text
wal_log_hints = on
```

或：

```text
data_checksums = on
```

当前主库必须允许物理复制：

```sql
SHOW wal_level;
SHOW max_wal_senders;
SHOW max_replication_slots;
```

推荐配置：

```conf
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
```

### 17.4 复制用户、`pg_hba.conf` 和 pg_rewind 权限

在每一个可能成为当前主库的节点，以管理员执行：

```sql
CREATE ROLE repl WITH LOGIN REPLICATION PASSWORD 'replace-with-strong-password';
```

用户已经存在时：

```sql
ALTER ROLE repl WITH LOGIN REPLICATION PASSWORD 'replace-with-strong-password';
```

在每一个可能成为当前主库的节点的 `pg_hba.conf` 中，为所有可能恢复为从库的节点添加复制访问规则：

```conf
host    replication    repl    192.168.3.151/32    scram-sha-256
```

双节点环境中，两个节点都需要为对方配置对应规则。修改后加载：

```sql
SELECT pg_reload_conf();
```

DBGuardian 脚本会预检 `pg_rewind` 所需读取能力。复制用户不是超级用户时，在当前主库以超级用户执行：

```sql
GRANT EXECUTE ON FUNCTION pg_read_binary_file(text) TO repl;
GRANT EXECUTE ON FUNCTION pg_read_binary_file(text, bigint, bigint, boolean) TO repl;
GRANT EXECUTE ON FUNCTION pg_stat_file(text, boolean) TO repl;
GRANT EXECUTE ON FUNCTION pg_ls_dir(text, boolean, boolean) TO repl;
```

不同 PostgreSQL 版本的内置函数签名可能不同。若脚本输出 `pg_rewind source privileges missing`，必须以当前数据库实际存在的函数签名补充授权，不能跳过预检。

`pg_rewind` 还依赖当前主库保留从分叉点起所需 WAL。建议使用物理复制槽：

```sql
SELECT * FROM pg_create_physical_replication_slot('appdb_slot');
```

脚本会在 slot 不存在时尝试创建，但生产环境应预先创建并监控 slot 积压，避免从库长期离线导致主库磁盘被 WAL 占满。

### 17.5 部署恢复脚本

Linux PostgreSQL 节点上，将 `pg-rewind-recover.sh` 放到每一个可能作为恢复从库的节点：

```bash
sudo install -d -o postgres -g postgres -m 0750 /opt/dbguardian/scripts
sudo install -o postgres -g postgres -m 0750 \
  pg-rewind-recover.sh \
  /opt/dbguardian/scripts/pg-rewind-recover.sh
```

确认脚本可由 `postgres` 用户执行：

```bash
sudo -u postgres test -x /opt/dbguardian/scripts/pg-rewind-recover.sh
sudo -u postgres ls -l /opt/dbguardian/scripts/pg-rewind-recover.sh
```

Windows PostgreSQL 节点上，将 `pg-rewind-recover.bat` 放到每一个可能作为恢复从库的节点，例如：

```text
C:\dbguardian\scripts\pg-rewind-recover.bat
```

运行 PostgreSQL 的 Windows 服务账号必须具有脚本、数据目录和 PostgreSQL `bin` 目录的读写/执行权限。

远程模式下，`script-path` 或 `failback-script-path` 始终是目标 PostgreSQL 节点本机的绝对路径，不是应用服务器路径。

### 17.6 SSH、PuTTY/Plink 和 sudo

Linux 应用服务器推荐使用 OpenSSH 公钥认证：

```bash
ssh-keygen -t ed25519 -f ~/.ssh/dbguardian_pg_recovery
ssh-copy-id -i ~/.ssh/dbguardian_pg_recovery.pub dbguardian@192.168.3.151
ssh dbguardian@192.168.3.151 "sudo -n -u postgres true"
```

Windows 应用服务器使用密码认证时，需要安装 PuTTY 并将 `plink.exe` 加入 `PATH`。验证：

```powershell
plink -V
plink dbguardian@192.168.3.151 "hostname"
```

首次连接应人工确认目标服务器主机指纹。推荐使用 PuTTY `.ppk` 私钥，而不是将 SSH 密码写入 YAML：

```powershell
plink -i C:\Keys\dbguardian-pg-recovery.ppk dbguardian@192.168.3.151 "sudo -n -u postgres true"
```

`remote-executor-password` 和 `failback-remote-executor-password` 只用于 SSH/Plink 登录密码，**不会**自动作为 `sudo` 密码。

恢复脚本通常需要以 `postgres` 系统用户执行。推荐为专用 SSH 用户建立最小范围的免密 sudo 规则：

```bash
sudo visudo -f /etc/sudoers.d/dbguardian-postgresql-recovery
```

写入：

```sudoers
Cmnd_Alias DBGUARDIAN_PG_RECOVERY = /usr/bin/env *, /bin/bash /opt/dbguardian/scripts/pg-rewind-recover.sh

dbguardian ALL=(postgres) NOPASSWD: DBGUARDIAN_PG_RECOVERY
```

不同发行版的 `bash` 路径可能不同，可用 `command -v bash` 确认。验证：

```bash
sudo -l -U dbguardian
ssh dbguardian@192.168.3.151 "sudo -n -u postgres env DBGUARDIAN_OLD_PGDATA=/tmp/test true"
```

不要配置 `dbguardian ALL=(ALL) NOPASSWD: ALL`。

当前没有独立的 `sudo-password` 配置项。虽然可以在远程命令内通过 `printf ... | sudo -S` 传递密码，但密码会出现在 YAML、命令行或进程信息中，不适合生产环境。组织策略禁止 `NOPASSWD` 时，应使用受控运维代理、短期证书或专用凭据系统封装 sudo。

### 17.7 从库恢复配置示例

以下示例中应用运行在 Windows，数据库运行在 Linux。恢复从库是 `192.168.3.151`，当前主库由运行态自动识别：

```yaml
spring:
  datasource:
    replication:
      master-host: 192.168.3.150
      master-port: 5432
      master-user: repl
      master-password: ${DBGUARDIAN_REPL_PASSWORD}
      auto-reconnect: true
    recovery:
      enabled: true
      auto-failback: true
      catchup-timeout-seconds: 300
      catchup-check-interval-seconds: 2
      postgresql:
        enabled: true
        # 与测试项目一致：原主追平后自动回切到原主。
        keep-promoted-primary: false

        # 原主恢复为从库时，在原主节点执行。
        data-directory: /var/lib/postgresql/16/main
        config-file: /etc/postgresql/16/main/postgresql.conf
        script-path: /opt/dbguardian/scripts/pg-rewind-recover.sh
        remote-execution-enabled: true
        remote-executor-command: >
          ssh ubuntu@192.168.3.150 "sudo -n -u postgres env {env_sh} bash {script_path_sh}"
        remote-executor-password: ${DBGUARDIAN_SSH_PASSWORD}
        pg-ctl-command: /usr/lib/postgresql/16/bin/pg_ctl
        psql-command: /usr/lib/postgresql/16/bin/psql
        pg-rewind-command: /usr/lib/postgresql/16/bin/pg_rewind
        local-host: 127.0.0.1
        local-port: 5432
        local-database: appdb
        primary-slot-name: appdb_slot
        extra-primary-conn-info: "target_session_attrs=read-write"
        startup-wait-seconds: 60
        command-timeout-seconds: 900

        # 普通从库恢复或自动回切时，在恢复从库节点执行。
        # failback-* 是历史配置名，目前作为从库侧执行配置使用。
        failback-data-directory: /var/lib/postgresql/16/main
        failback-config-file: /etc/postgresql/16/main/postgresql.conf
        failback-script-path: /opt/dbguardian/scripts/pg-rewind-recover.sh
        failback-remote-execution-enabled: true
        failback-remote-executor-command: >
          ssh ubuntu@192.168.3.151 "sudo -n -u postgres env {env_sh} bash {script_path_sh}"
        failback-remote-executor-password: ${DBGUARDIAN_SSH_PASSWORD}
        failback-pg-ctl-command: /usr/lib/postgresql/16/bin/pg_ctl
        failback-psql-command: /usr/lib/postgresql/16/bin/psql
        failback-pg-rewind-command: /usr/lib/postgresql/16/bin/pg_rewind
        failback-local-host: 127.0.0.1
        failback-local-port: 5432
        failback-local-database: appdb
        failback-primary-slot-name: appdb_slot
        failback-extra-primary-conn-info: "target_session_attrs=read-write"
        failback-startup-wait-seconds: 60
        failback-command-timeout-seconds: 900
```

注意：

- 优先使用 SSH 密钥；密码应通过环境变量、密钥管理系统或外部配置注入，禁止提交到版本库。
- `{env_sh}` 与 `{script_path_sh}` 必须原样保留，执行器会在运行时替换。
- 从库恢复时，`failback-remote-executor-command` 必须指向恢复从库节点。

### 17.8 上线前验证和常见故障

从应用服务器验证 SSH、sudo、脚本和 PostgreSQL 工具：

```bash
ssh dbguardian@192.168.3.151 "sudo -n -u postgres /usr/lib/postgresql/16/bin/pg_rewind --version"
ssh dbguardian@192.168.3.151 "sudo -n -u postgres test -x /opt/dbguardian/scripts/pg-rewind-recover.sh"
```

Windows 使用 PuTTY/Plink：

```powershell
plink dbguardian@192.168.3.151 "sudo -n -u postgres /usr/lib/postgresql/16/bin/pg_rewind --version"
```

在主库确认复制槽和复制连接：

```sql
SELECT slot_name, active FROM pg_replication_slots;
SELECT application_name, client_addr, state, sync_state FROM pg_stat_replication;
```

在从库确认角色与流复制状态：

```sql
SELECT pg_is_in_recovery();
SELECT status FROM pg_stat_wal_receiver;
SELECT pg_last_wal_replay_lsn();
```

`pg_is_in_recovery()` 应为 `true`，`pg_stat_wal_receiver.status` 应为 `streaming`。

常见故障：

| 现象 | 排查方向 |
|---|---|
| `pg_rewind dry-run failed` | 检查共同时间线、`wal_log_hints`/data checksums、WAL 保留和第 17.4 节中的函数权限；无法修复时使用 `pg_basebackup` 全量重建从库。 |
| `sudo: a password is required` | sudoers 未允许非交互执行；按第 17.6 节配置最小范围 `NOPASSWD` 规则。 |
| SSH 成功但脚本未在数据库节点执行 | 检查远程配置中的脚本路径是否为 PostgreSQL 节点上的绝对路径。 |
| 从库已是 standby 但没有回到读池 | `standby` 不代表复制健康；检查 `pg_stat_wal_receiver.status`、`primary_conninfo`、`pg_hba.conf`、复制密码、slot 和网络。 |

在日志出现“从库复制重建已完成，等待后续健康检查确认流复制”后，读请求继续路由主库是预期行为；后续健康检查确认 `streaming` 并记录“从库恢复”后，才会恢复从库读路由。