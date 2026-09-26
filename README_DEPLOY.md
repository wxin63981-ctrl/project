# 智检云枢龙芯生产部署手册

适用目标：龙芯 LoongArch64 云主机、银河麒麟服务器操作系统、Python 3.11、MySQL 8.0、Nginx、systemd。前端在 Windows 开发机预构建，服务器只运行静态文件和后端，不依赖 Node.js。

## 一、部署文件说明

| 文件 | 用途 |
|---|---|
| `nginx/smart-maintenance.conf` | 前端静态资源、SPA 路由、API 反向代理和安全响应头 |
| `systemd/smart-maintenance-api.service` | 后端开机自启、失败重启、日志和启动前数据库升级 |
| `systemd/smart-maintenance-backup.*` | 每天 02:30 自动备份，保留 14 天 |
| `.env.production.example` | 生产环境变量模板，不含任何真实密钥 |
| `scripts/preflight_loongarch.sh` | 检查架构、系统命令、Python、内存和磁盘 |
| `scripts/check_python_dependencies.sh` | 在临时虚拟环境实装并导入全部 Python 依赖 |
| `scripts/init_database.sh` | 创建 MySQL 数据库、应用用户和最小范围授权 |
| `backend/scripts/upgrade_database.py` | 幂等数据库结构升级及版本记录 |
| `scripts/backup_mysql.sh` | 手工或定时备份、压缩、SHA-256 校验和过期清理 |
| `scripts/restore_mysql.sh` | 校验备份后恢复 MySQL |
| `scripts/install.sh` | 安装程序、虚拟环境、Nginx 和 systemd 配置 |
| `scripts/start.sh` / `stop.sh` | 启动或停止正式服务 |
| `scripts/status.sh` / `healthcheck.sh` | 状态、日志、端口和四层健康检查 |

## 二、部署前准备

### 1. 在 Windows 开发机生成前端生产包

```powershell
cd D:\ruanjianbei\complete\frontend
npm install
npm run build
Test-Path .\dist\index.html
```

最后一条必须返回 `True`。将整个 `complete` 上传到服务器，例如 `/tmp/complete`。不要上传开发用 `.env`、数据库密码、API Key、`smart.db` 或 `node_modules`。

### 2. 在龙芯服务器安装系统组件

银河麒麟版本的包名可能略有不同，先执行：

```bash
sudo dnf install -y python3 python3-pip nginx mysql mysql-server gcc python3-devel
```

若系统只提供 `yum`，将 `dnf` 替换为 `yum`。Python 必须为 3.11 或更高版本：

```bash
python3 --version
```

### 3. 做龙芯预检

```bash
cd /tmp/complete
chmod +x deploy/scripts/*.sh
sudo PYTHON_BIN=python3.11 ./deploy/scripts/preflight_loongarch.sh
PYTHON_BIN=python3.11 ./deploy/scripts/check_python_dependencies.sh
```

第二条会在 `/tmp` 创建临时虚拟环境，验证 `greenlet`、`pydantic_core` 等依赖能否在 LoongArch 上安装。必须在正式安装前通过。测试环境会自动清理，不连接业务数据库。

## 三、正式安装

### 1. 安装程序文件

```bash
cd /tmp/complete
sudo PROJECT_ROOT=/tmp/complete PYTHON_BIN=python3.11 ./deploy/scripts/install.sh
```

安装结果：

- 后端：`/opt/smart-maintenance/backend`
- 前端：`/opt/smart-maintenance/frontend`
- Python 环境：`/opt/smart-maintenance/venv`
- 部署脚本：`/opt/smart-maintenance/deploy`
- 生产环境变量：`/etc/smart-maintenance/backend.env`
- 数据库备份：`/var/backups/smart-maintenance`

安装脚本不会自动启动业务服务，也不会覆盖已经存在的生产环境变量。

### 2. 配置生产环境变量

```bash
sudo vi /etc/smart-maintenance/backend.env
```

必须替换所有 `CHANGE_ME` 和 `SERVER_IP_OR_DOMAIN`。生成随机密钥：

```bash
openssl rand -hex 32
```

注意：

- `DATABASE_URL=` 保持为空，程序才会使用 `MYSQL_*`。
- `MYSQL_PASSWORD` 是应用数据库用户密码。
- 数据库密码至少 12 位，并使用字母、数字或 `_@%+=:,.!?~-`，避免环境文件和初始化 SQL 的转义歧义。
- `MODEL_API_KEY` 填阿里云百炼 API Key。
- `BOOTSTRAP_ADMIN_PASSWORD` 至少 12 位，并同时包含大小写字母和数字。
- `ALLOWED_HOSTS` 写服务器 IP 或域名，多个值用英文逗号分隔。
- 同域名通过 Nginx 访问时，`CORS_ORIGINS=` 可以为空。
- `EMBEDDING_*` 已放入此生产模板，部署后会启用混合向量检索。

检查权限和配置：

```bash
sudo chown root:smartmaint /etc/smart-maintenance/backend.env
sudo chmod 640 /etc/smart-maintenance/backend.env
sudo /opt/smart-maintenance/deploy/scripts/validate_env.sh
```

### 3. 初始化 MySQL

```bash
if systemctl list-unit-files | grep -q '^mysqld.service'; then
  sudo systemctl enable --now mysqld
else
  sudo systemctl enable --now mysql
fi
sudo /opt/smart-maintenance/deploy/scripts/init_database.sh
```

脚本默认提示输入 MySQL `root` 密码，并依据生产环境变量创建数据库和应用用户。它不会授予全库全局权限。

检查和升级数据库：

```bash
sudo -u smartmaint bash -c 'set -a; source /etc/smart-maintenance/backend.env; set +a; /opt/smart-maintenance/venv/bin/python /opt/smart-maintenance/backend/scripts/upgrade_database.py --check'
sudo -u smartmaint bash -c 'set -a; source /etc/smart-maintenance/backend.env; set +a; /opt/smart-maintenance/venv/bin/python /opt/smart-maintenance/backend/scripts/upgrade_database.py --upgrade'
```

升级脚本可重复运行，已经执行的版本会显示“跳过”。systemd 每次启动后端前也会自动执行一次升级。

### 4. 启动并验收

```bash
sudo /opt/smart-maintenance/deploy/scripts/start.sh
```

`start.sh` 会依次检查环境变量、检查 Nginx 配置、启动后端与 Nginx、启用每日备份，并等待数据库就绪。

如服务器启用了防火墙，再开放 HTTP：

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

不要向公网开放 `3306` 和 `8000`，这两个端口只供本机 MySQL 和 Nginx 使用。

如果 `getenforce` 显示 `Enforcing`，还需允许 Nginx 读取静态目录并连接本机后端：

```bash
sudo setsebool -P httpd_can_network_connect 1
sudo semanage fcontext -a -t httpd_sys_content_t '/opt/smart-maintenance/frontend(/.*)?'
sudo restorecon -Rv /opt/smart-maintenance/frontend
```

如果没有 `semanage`，先从系统仓库安装 `policycoreutils-python-utils`，部分银河麒麟版本的包名为 `policycoreutils-python`。

浏览器打开 `http://服务器IP/`。首次登录使用 `admin` 和环境变量中的初始管理员密码，登录后立即在系统中修改密码。

验收命令：

```bash
sudo /opt/smart-maintenance/deploy/scripts/healthcheck.sh
curl -fsS http://127.0.0.1/health/ready
```

必须同时通过：systemd 后端、Nginx、FastAPI 存活、MySQL 就绪、反向代理和前端首页。

## 四、日常启停与更新

```bash
# 查看状态、最近日志、监听端口和下次备份时间
sudo /opt/smart-maintenance/deploy/scripts/status.sh

# 启动或重启完整服务
sudo /opt/smart-maintenance/deploy/scripts/start.sh

# 只停止后端，保留 Nginx
sudo /opt/smart-maintenance/deploy/scripts/stop.sh

# 同时停止后端和 Nginx
sudo /opt/smart-maintenance/deploy/scripts/stop.sh --all
```

更新版本前先备份数据库，再在新上传目录重新执行 `install.sh` 和 `start.sh`。现有 `/etc/smart-maintenance/backend.env` 不会被覆盖。

## 五、备份与恢复

### 立即备份

```bash
sudo /opt/smart-maintenance/deploy/scripts/backup_mysql.sh
ls -lh /var/backups/smart-maintenance
```

系统每天约 02:30 自动执行一次，保留 14 天。查看定时器：

```bash
systemctl list-timers smart-maintenance-backup.timer
journalctl -u smart-maintenance-backup.service -n 50 --no-pager
```

### 恢复备份

恢复会改变数据库，必须先停止后端并再做一次当前数据备份：

```bash
sudo /opt/smart-maintenance/deploy/scripts/stop.sh
sudo /opt/smart-maintenance/deploy/scripts/backup_mysql.sh
sudo /opt/smart-maintenance/deploy/scripts/restore_mysql.sh /var/backups/smart-maintenance/备份文件.sql.gz --yes
sudo -u smartmaint bash -c 'set -a; source /etc/smart-maintenance/backend.env; set +a; /opt/smart-maintenance/venv/bin/python /opt/smart-maintenance/backend/scripts/upgrade_database.py --upgrade'
sudo /opt/smart-maintenance/deploy/scripts/start.sh
```

恢复脚本会先验证 `.sha256`（若存在）和 gzip 完整性。建议定期在非生产环境做恢复演练，只有“可以恢复”的备份才是有效备份。

## 六、故障排查

| 现象 | 检查命令 | 处理方向 |
|---|---|---|
| 页面显示 `502 Bad Gateway` | `journalctl -u smart-maintenance-api -n 100 --no-pager` | 后端未启动、环境变量错误或数据库升级失败 |
| `/health` 正常但 `/health/ready` 返回 503 | `systemctl status mysqld` | MySQL 未启动、账号密码或授权主机错误 |
| 首页空白或静态资源 404 | `ls -l /opt/smart-maintenance/frontend/index.html` | Windows 未执行 `npm run build`，或上传时遗漏 `dist` |
| Nginx 启动失败 | `nginx -t` | 80 端口占用、重复 `default_server` 或配置语法错误 |
| 后端显示 Host 不允许 | 检查 `ALLOWED_HOSTS` | 补充实际 IP/域名后重启后端 |
| 百炼调用失败但本地检索仍可用 | `journalctl -u smart-maintenance-api` | 检查 API Key、外网访问和模型名称；系统会自动降级本地检索 |
| Python 依赖在龙芯安装失败 | `check_python_dependencies.sh --keep` | 安装 `gcc`、`python3-devel`，根据保留目录中的 pip 错误定位缺失编译环境 |
| 上传手册返回 413 | 检查 Nginx `client_max_body_size` | 当前默认 8 MB，可按赛题文件大小适当上调 |
| 自动备份失败 | `journalctl -u smart-maintenance-backup.service` | 检查 MySQL 客户端、数据库授权和备份目录空间 |

常用直接检查：

```bash
systemctl status smart-maintenance-api nginx mysqld
journalctl -u smart-maintenance-api -f
tail -f /var/log/nginx/smart-maintenance.error.log
ss -lntp | grep -E ':(80|8000|3306)\b'
curl -v http://127.0.0.1:8000/health/ready
curl -v http://127.0.0.1/health/ready
```

## 七、生产安全底线

- 不把 `/etc/smart-maintenance/backend.env`、API Key、数据库密码或数据库备份提交到 Git。
- 后端仅监听 `127.0.0.1:8000`，外部请求统一经过 Nginx。
- 正式环境关闭公开注册，管理员初始密码不再使用开发默认值 `123456`。
- 部署前备份，更新后执行健康检查，恢复后再次执行数据库升级。
- 有正式域名时建议在 Nginx 上增加 HTTPS 证书，并将 HTTP 重定向到 HTTPS。
