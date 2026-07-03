# Sealos 模板支持外部数据库的通用改造教程

本文档说明如何把一个已经内置数据库的 Sealos 应用模板，改造成“默认使用模板自带数据库，也可通过 inputs 切换到外部数据库”的通用形态。

推荐做法是：外部数据库只暴露一个连接串输入框。模板内部再从连接串解析出 host、port、username、password、database。这样用户界面更简洁，也更适合迁移到其它带数据库的应用。

## 改造目标

- 默认保持原模板行为，继续创建并使用模板自带数据库。
- 新增一个布尔 input，例如 `use_external_mysql`，决定是否使用外部数据库。
- 外部数据库只新增一个连接串 input，例如 `external_mysql_url`。
- 启用外部数据库时，不再渲染模板自带数据库的 ServiceAccount、Role、RoleBinding、Cluster 等资源。
- 外部数据库路径创建同名连接 Secret，应用容器和初始化 Job 统一读取这个 Secret。
- 启动脚本解析连接串，得到应用需要的数据库环境变量。

## 连接串格式

MySQL 推荐格式：

```text
mysql://user:password@mysql.example.com:3306/app
```

PostgreSQL 推荐格式：

```text
postgresql://user:password@postgres.example.com:5432/app
```

可以允许 query 参数，但模板脚本通常会忽略：

```text
mysql://user:password@mysql.example.com:3306/app?charset=utf8mb4
```

注意：如果使用纯 shell 解析连接串，密码中尽量不要包含未转义的 `/` 这类会破坏 URL 路径结构的字符。必须支持复杂密码时，建议在镜像中提供更完整的解析工具，或改回多字段输入。

## 第一步：新增 inputs

在 `spec.inputs` 中新增布尔开关和连接串输入。

```yaml
inputs:
  use_external_mysql:
    description: 'Use an external MySQL database instead of creating the built-in MySQL database.'
    type: boolean
    default: 'false'
    required: false
  external_mysql_url:
    description: 'External MySQL connection string, for example mysql://user:password@mysql.example.com:3306/app.'
    type: string
    default: ''
    required: true
    if: inputs.use_external_mysql === 'true'
```

PostgreSQL 可以改成：

```yaml
inputs:
  use_external_postgresql:
    description: 'Use an external PostgreSQL database instead of creating the built-in PostgreSQL database.'
    type: boolean
    default: 'false'
    required: false
  external_postgresql_url:
    description: 'External PostgreSQL connection string, for example postgresql://user:password@postgres.example.com:5432/app.'
    type: string
    default: ''
    required: true
    if: inputs.use_external_postgresql === 'true'
```

## 第二步：条件渲染内置数据库资源

把原模板中创建内置数据库的资源包起来。通常包括数据库 ServiceAccount、Role、RoleBinding、Cluster，以及只服务于内置数据库的其它资源。

```yaml
---
${{ if(inputs.use_external_mysql === 'false') }}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ${{ defaults.app_name }}-mysql

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ${{ defaults.app_name }}-mysql

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ${{ defaults.app_name }}-mysql

---
apiVersion: apps.kubeblocks.io/v1alpha1
kind: Cluster
metadata:
  name: ${{ defaults.app_name }}-mysql
spec:
  # 保留原数据库 Cluster 配置
${{ else() }}

---
apiVersion: v1
kind: Secret
metadata:
  name: ${{ defaults.app_name }}-mysql-conn-credential
  labels:
    app: ${{ defaults.app_name }}-mysql
    cloud.sealos.io/app-deploy-manager: ${{ defaults.app_name }}-mysql
type: Opaque
stringData:
  url: "${{ inputs.external_mysql_url }}"
${{ endif() }}
```

关键点是：外部数据库路径仍然创建 `${{ defaults.app_name }}-mysql-conn-credential`，但只保存 `url`。内置数据库路径继续使用 KubeBlocks 自动生成的同名 Secret，其中通常包含 `host`、`port`、`username`、`password`。

## 第三步：改造初始化 Job

初始化 Job 需要兼容两种输入来源：

- 内置数据库模式：从 KubeBlocks Secret 读取 `host`、`port`、`username`、`password`，数据库名使用模板默认值。
- 外部数据库模式：从 Secret 读取 `url`，在脚本中解析出连接参数。

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: ${{ defaults.app_name }}-mysql-init
spec:
  completions: 1
  template:
    spec:
      automountServiceAccountToken: false
      containers:
        - name: mysql-init
          image: mysql:8.0
          imagePullPolicy: IfNotPresent
          env:
            ${{ if(inputs.use_external_mysql === 'true') }}
            - name: MYSQL_URL
              valueFrom:
                secretKeyRef:
                  name: ${{ defaults.app_name }}-mysql-conn-credential
                  key: url
            ${{ else() }}
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: ${{ defaults.app_name }}-mysql-conn-credential
                  key: password
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: ${{ defaults.app_name }}-mysql-conn-credential
                  key: username
            - name: MYSQL_HOST
              valueFrom:
                secretKeyRef:
                  name: ${{ defaults.app_name }}-mysql-conn-credential
                  key: host
            - name: MYSQL_PORT
              valueFrom:
                secretKeyRef:
                  name: ${{ defaults.app_name }}-mysql-conn-credential
                  key: port
            - name: DATABASE_NAME
              value: app
            ${{ endif() }}
          command:
            - /bin/sh
            - -c
            - |
              if [ -n "${MYSQL_URL:-}" ]; then
                mysql_url="${MYSQL_URL#mysql://}"
                if [ "${mysql_url}" = "${MYSQL_URL}" ]; then
                  echo "MYSQL_URL must use mysql://user:password@host:port/database format." >&2
                  exit 1
                fi
                mysql_url="${mysql_url%%\?*}"
                mysql_auth="${mysql_url%@*}"
                mysql_host_db="${mysql_url##*@}"
                if [ "${mysql_auth}" = "${mysql_url}" ] || [ -z "${mysql_auth}" ] || [ "${mysql_host_db}" = "${mysql_url}" ]; then
                  echo "MYSQL_URL must include user, password, host, and database." >&2
                  exit 1
                fi
                if [ "${mysql_auth#*:}" = "${mysql_auth}" ]; then
                  echo "MYSQL_URL must include a password." >&2
                  exit 1
                fi
                MYSQL_USER="${mysql_auth%%:*}"
                MYSQL_PASSWORD="${mysql_auth#*:}"
                mysql_host_port="${mysql_host_db%%/*}"
                DATABASE_NAME="${mysql_host_db#*/}"
                if [ "${DATABASE_NAME}" = "${mysql_host_db}" ] || [ -z "${DATABASE_NAME}" ]; then
                  echo "MYSQL_URL must include a database path." >&2
                  exit 1
                fi
                MYSQL_HOST="${mysql_host_port%%:*}"
                MYSQL_PORT="${mysql_host_port#*:}"
                if [ "${MYSQL_PORT}" = "${MYSQL_HOST}" ]; then
                  MYSQL_PORT=3306
                fi
              fi

              until mysql -h${MYSQL_HOST} -P${MYSQL_PORT} -u${MYSQL_USER} -p${MYSQL_PASSWORD} -e "SELECT 1" >/dev/null 2>&1; do
                echo "MySQL is not ready, retrying..."
                sleep 2
              done
              mysql -h${MYSQL_HOST} -P${MYSQL_PORT} -u${MYSQL_USER} -p${MYSQL_PASSWORD} \
                -e "CREATE DATABASE IF NOT EXISTS \`${DATABASE_NAME}\` DEFAULT CHARACTER SET utf8mb4 DEFAULT COLLATE utf8mb4_unicode_ci;" \
                || mysql -h${MYSQL_HOST} -P${MYSQL_PORT} -u${MYSQL_USER} -p${MYSQL_PASSWORD} "${DATABASE_NAME}" -e "SELECT 1"
      restartPolicy: Never
  backoffLimit: 0
  ttlSecondsAfterFinished: 300
```

如果外部数据库账号通常没有建库权限，这种写法会先尝试建库；建库失败后再验证目标库可连接。如果你的应用要求外部数据库必须提前创建，也可以把整个初始化 Job 放进 `${{ if(inputs.use_external_mysql === 'false') }}`，只在内置数据库模式运行。

## 第四步：改造应用容器 env

应用容器也按同样方式切换 Secret key：

```yaml
env:
  ${{ if(inputs.use_external_mysql === 'true') }}
  - name: APP_DB_URL
    valueFrom:
      secretKeyRef:
        name: ${{ defaults.app_name }}-mysql-conn-credential
        key: url
  ${{ else() }}
  - name: APP_DB_HOST
    valueFrom:
      secretKeyRef:
        name: ${{ defaults.app_name }}-mysql-conn-credential
        key: host
  - name: APP_DB_PORT
    valueFrom:
      secretKeyRef:
        name: ${{ defaults.app_name }}-mysql-conn-credential
        key: port
  - name: APP_DB_USER
    valueFrom:
      secretKeyRef:
        name: ${{ defaults.app_name }}-mysql-conn-credential
        key: username
  - name: APP_DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: ${{ defaults.app_name }}-mysql-conn-credential
        key: password
  - name: APP_DB_NAME
    value: app
  ${{ endif() }}
```

如果应用原生支持连接串，可以直接把 `APP_DB_URL` 传进去。否则需要在启动脚本里解析连接串，然后导出应用原本需要的变量。

```sh
if [ -n "${APP_DB_URL:-}" ]; then
  db_url="${APP_DB_URL#mysql://}"
  db_url="${db_url%%\?*}"
  db_auth="${db_url%@*}"
  db_host_db="${db_url##*@}"
  APP_DB_USER="${db_auth%%:*}"
  APP_DB_PASSWORD="${db_auth#*:}"
  db_host_port="${db_host_db%%/*}"
  APP_DB_NAME="${db_host_db#*/}"
  APP_DB_HOST="${db_host_port%%:*}"
  APP_DB_PORT="${db_host_port#*:}"
  if [ "${APP_DB_PORT}" = "${APP_DB_HOST}" ]; then
    APP_DB_PORT=3306
  fi
fi
```

随后继续使用应用原来的配置逻辑：

```sh
export APP_JDBC_URL="jdbc:mysql://${APP_DB_HOST}:${APP_DB_PORT}/${APP_DB_NAME}"
export APP_DB_USER APP_DB_PASSWORD
```

## 第五步：补齐端口和数据库名

很多旧模板只写了数据库 host：

```properties
db_host=${DB_HOST}
```

改造后建议包含端口：

```properties
db_host=${DB_HOST}:${DB_PORT}
```

如果配置文件通过 `sed` 替换，也要补齐端口和数据库名：

```sh
sed -i "s#\${DB_HOST}#${DB_HOST}#g" /tmp/app.properties
sed -i "s#\${DB_PORT}#${DB_PORT}#g" /tmp/app.properties
sed -i "s#\${DB_NAME}#${DB_NAME}#g" /tmp/app.properties
```

## PostgreSQL 改造差异

PostgreSQL 的整体思路相同：

- input 使用 `use_external_postgresql` 和 `external_postgresql_url`。
- Secret 名称通常是 `${{ defaults.app_name }}-pg-conn-credential`。
- 外部连接串格式使用 `postgresql://user:password@host:5432/database`。
- 初始化镜像使用 `postgres:<version>`。
- 连接测试命令使用 `psql`。

PostgreSQL 不支持 `CREATE DATABASE IF NOT EXISTS`，生产模板中建议先查询数据库是否存在，或只做连接验证。

## 搜索和改造范围

改造前先搜索所有数据库引用：

```sh
rg -n "mysql|postgres|database|DB_HOST|DB_PORT|DB_NAME|conn-credential|secretKeyRef" template/<app> -S
```

重点确认：

- 应用主容器已改造。
- 初始化 Job、迁移 Job、worker、cron Job 都已改造。
- 内置数据库资源全部被条件包住。
- 外部数据库路径创建同名连接 Secret。
- 外部模式不再引用内置数据库 Secret 的 `host`、`port`、`username`、`password` key。
- 内置模式不依赖外部连接串 input。

## 验证两种渲染路径

至少验证两种场景：

- `use_external_mysql=false`：应包含内置数据库 Cluster，不应包含外部数据库 URL Secret。
- `use_external_mysql=true`：不应包含内置数据库 Cluster，应包含保存 `url` 的外部数据库 Secret。

可以用轻量脚本做粗略 YAML 解析验证。根据实际 input 名称替换条件字符串。

```sh
python3 -c 'from pathlib import Path
import yaml
path = Path("template/<app>/index-with-external-db.yaml")
lines = path.read_text().splitlines()
for mode in ("false", "true"):
    out = []
    keep = True
    in_block = False
    for line in lines:
        s = line.strip()
        if s == "${{ if(inputs.use_external_mysql === '\''false'\'') }}":
            in_block = True
            keep = mode == "false"
            continue
        if s == "${{ else() }}" and in_block:
            keep = mode == "true"
            continue
        if s == "${{ endif() }}" and in_block:
            keep = True
            in_block = False
            continue
        if keep:
            out.append(line)
    docs = [d for d in yaml.safe_load_all("\n".join(out)) if d is not None]
    print(mode, len(docs), [d.get("kind") for d in docs])
'
```

这类脚本只能检查 YAML 结构，不能替代平台渲染和真实部署验证。

## 最终检查清单

- [ ] 新增布尔 input，默认值保持原行为。
- [ ] 外部数据库只暴露一个连接串 input。
- [ ] 连接串描述中写清楚格式示例。
- [ ] 内置数据库资源被 `${{ if(...) }}` 条件包住。
- [ ] 外部数据库路径创建同名连接 Secret，且 Secret 中保存 `url`。
- [ ] 外部模式下应用和 Job 读取 `url`，并在脚本中解析。
- [ ] 内置模式下应用和 Job 继续读取 KubeBlocks Secret 的 `host`、`port`、`username`、`password`。
- [ ] 数据库名、host、port、username、password 都能正确传入应用。
- [ ] 配置文件或启动脚本里的连接 URL 已包含端口。
- [ ] 两种模式都能通过 YAML 结构解析。
