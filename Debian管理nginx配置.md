# Debian管理nginx配置

## 🧑‍💻 User

我需要管理debian11上面存放在/etc/nginx/conf.d/路径下的conf文件，# EXPIRE: 2026-04-07中的2026-04-07是配置过期时间，listen 10407是监听端口，我需要能够一个http页面可以自定义输入EXPIRE与listen值来生成与删除以下格式的conf文件并通过nginx -s reload重新加载nginx配置

---

“带登录认证 + 一键清理过期配置”的完整增强版代码

---

## 🤖 Assistant

下面给你一套**完整增强版**，包含：

- 登录认证（用户名密码）
- 创建 conf
- 删除 conf
- 一键清理过期 conf
- `nginx -t && nginx -s reload`
- 端口占用检查
- 只管理 `managed_*.conf`
- 可直接部署在 Debian 11

我给你的是 **Flask 单文件后端 + 一个 HTML 模板**，简单稳定。

---

# 一、功能说明

页面支持：

1. **登录**
2. **创建配置**
   - 输入 `EXPIRE`
   - 输入 `listen`
3. **查看当前 managed 配置**
4. **删除单个配置**
5. **一键删除所有过期配置**
6. 每次修改后自动：
   - `nginx -t`
   - `nginx -s reload`

---

# 二、你的固定模板

生成的 conf 固定为：

```nginx
# EXPIRE: 2026-04-07
server {
    listen 10407;

    if ($whitelist_usergent = 0) {
        return 444;
    }

    location / {
        # 1. 设置转发目标 (注意协议是 https)
        proxy_pass http://127.0.0.1:19321;
    }
}
```

只替换：

- `EXPIRE`
- `listen`

---

# 三、项目目录

建议放这里：

```bash
/opt/nginx-conf-manager/
```

目录结构：

```bash
/opt/nginx-conf-manager/
├── app.py
└── templates/
    └── index.html
```

---

# 四、安装依赖

```bash
apt update
apt install -y python3 python3-pip sudo nginx
pip3 install flask
```

---

# 五、完整后端代码 app.py

创建：

```bash
mkdir -p /opt/nginx-conf-manager/templates
nano /opt/nginx-conf-manager/app.py
```

写入以下完整代码：

```python
import os
import re
import secrets
import shutil
import subprocess
import tempfile
from datetime import datetime, date
from functools import wraps

from flask import (
    Flask, render_template, request, redirect,
    url_for, flash, session
)

app = Flask(__name__)
app.secret_key = "please-change-to-a-long-random-secret-key"

# =========================
# 基础配置
# =========================
NGINX_CONF_DIR = "/etc/nginx/conf.d"
FILE_PREFIX = "managed_"
NGINX_BIN = "/usr/sbin/nginx"

# 登录账号密码（建议改掉）
ADMIN_USERNAME = "admin"
ADMIN_PASSWORD = "ChangeMe123!"

# =========================
# 工具函数
# =========================
def is_valid_date(date_str):
    try:
        datetime.strptime(date_str, "%Y-%m-%d")
        return True
    except ValueError:
        return False


def is_valid_port(port_str):
    if not port_str.isdigit():
        return False
    port = int(port_str)
    return 1 <= port <= 65535


def conf_filename(listen, expire):
    return f"{FILE_PREFIX}{listen}_{expire}.conf"


def conf_path(filename):
    return os.path.join(NGINX_CONF_DIR, filename)


def build_conf_content(expire, listen):
    return f"""# EXPIRE: {expire}
server {{
    listen {listen};

    if ($whitelist_usergent = 0) {{
        return 444;
    }}

    location / {{
        # 1. 设置转发目标 (注意协议是 https)
        proxy_pass http://127.0.0.1:19321;
    }}
}}
"""


def run_cmd(cmd_list):
    result = subprocess.run(cmd_list, text=True, capture_output=True)
    return result.returncode, result.stdout, result.stderr


def nginx_test_and_reload():
    code, out, err = run_cmd(["sudo", NGINX_BIN, "-t"])
    if code != 0:
        return code, out, err

    code, out2, err2 = run_cmd(["sudo", NGINX_BIN, "-s", "reload"])
    return code, (out or "") + (out2 or ""), (err or "") + (err2 or "")


def parse_conf_file(path):
    result = {
        "name": os.path.basename(path),
        "expire": "",
        "listen": "",
        "expired": False,
    }

    try:
        with open(path, "r", encoding="utf-8") as f:
            content = f.read()

        m_expire = re.search(r"^# EXPIRE:\s*([0-9]{4}-[0-9]{2}-[0-9]{2})", content, re.MULTILINE)
        if m_expire:
            result["expire"] = m_expire.group(1)

        m_listen = re.search(r"listen\s+([0-9]+);", content)
        if m_listen:
            result["listen"] = m_listen.group(1)

        if result["expire"]:
            exp_date = datetime.strptime(result["expire"], "%Y-%m-%d").date()
            result["expired"] = exp_date < date.today()

    except Exception:
        pass

    return result


def get_managed_files():
    files = []
    if not os.path.isdir(NGINX_CONF_DIR):
        return files

    for filename in sorted(os.listdir(NGINX_CONF_DIR)):
        if filename.startswith(FILE_PREFIX) and filename.endswith(".conf"):
            path = os.path.join(NGINX_CONF_DIR, filename)
            files.append(parse_conf_file(path))
    return files


def port_already_used(listen):
    listen = str(listen)

    if not os.path.isdir(NGINX_CONF_DIR):
        return False, ""

    for filename in os.listdir(NGINX_CONF_DIR):
        if not filename.endswith(".conf"):
            continue

        path = os.path.join(NGINX_CONF_DIR, filename)
        try:
            with open(path, "r", encoding="utf-8") as f:
                content = f.read()

            matches = re.findall(r"listen\s+([0-9]+);", content)
            if listen in matches:
                return True, filename
        except Exception:
            continue

    return False, ""


def login_required(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        if not session.get("logged_in"):
            return redirect(url_for("login"))
        return func(*args, **kwargs)
    return wrapper


# =========================
# 登录 / 登出
# =========================
@app.route("/login", methods=["GET", "POST"])
def login():
    if request.method == "GET":
        if session.get("logged_in"):
            return redirect(url_for("index"))
        return render_template("index.html", page="login")

    username = request.form.get("username", "").strip()
    password = request.form.get("password", "").strip()

    if secrets.compare_digest(username, ADMIN_USERNAME) and secrets.compare_digest(password, ADMIN_PASSWORD):
        session["logged_in"] = True
        flash("登录成功", "success")
        return redirect(url_for("index"))

    flash("用户名或密码错误", "error")
    return render_template("index.html", page="login")


@app.route("/logout", methods=["POST"])
@login_required
def logout():
    session.clear()
    flash("已退出登录", "success")
    return redirect(url_for("login"))


# =========================
# 页面首页
# =========================
@app.route("/", methods=["GET"])
@login_required
def index():
    files = get_managed_files()
    total = len(files)
    expired_count = sum(1 for x in files if x["expired"])
    return render_template(
        "index.html",
        page="main",
        files=files,
        total=total,
        expired_count=expired_count
    )


# =========================
# 创建配置
# =========================
@app.route("/create", methods=["POST"])
@login_required
def create_conf():
    expire = request.form.get("expire", "").strip()
    listen = request.form.get("listen", "").strip()

    if not is_valid_date(expire):
        flash("EXPIRE 日期格式不正确，应为 YYYY-MM-DD", "error")
        return redirect(url_for("index"))

    if not is_valid_port(listen):
        flash("listen 端口不合法", "error")
        return redirect(url_for("index"))

    used, used_file = port_already_used(listen)
    if used:
        flash(f"端口 {listen} 已被占用，配置文件: {used_file}", "error")
        return redirect(url_for("index"))

    filename = conf_filename(listen, expire)
    filepath = conf_path(filename)

    if os.path.exists(filepath):
        flash(f"文件已存在: {filename}", "error")
        return redirect(url_for("index"))

    content = build_conf_content(expire, listen)

    tmp_file = None
    try:
        with tempfile.NamedTemporaryFile("w", delete=False, encoding="utf-8") as f:
            f.write(content)
            tmp_file = f.name

        code, out, err = run_cmd(["sudo", "/bin/cp", tmp_file, filepath])
        if code != 0:
            flash(f"复制配置文件失败: {err or out}", "error")
            return redirect(url_for("index"))

        code, out, err = nginx_test_and_reload()
        if code != 0:
            run_cmd(["sudo", "/bin/rm", "-f", filepath])
            flash(f"nginx 校验或重载失败，已回滚: {err or out}", "error")
            return redirect(url_for("index"))

        flash(f"创建成功: {filename}", "success")
        return redirect(url_for("index"))

    except Exception as e:
        run_cmd(["sudo", "/bin/rm", "-f", filepath])
        flash(f"创建失败: {e}", "error")
        return redirect(url_for("index"))

    finally:
        if tmp_file and os.path.exists(tmp_file):
            os.remove(tmp_file)


# =========================
# 删除单个配置
# =========================
@app.route("/delete", methods=["POST"])
@login_required
def delete_conf():
    filename = request.form.get("filename", "").strip()

    if not (filename.startswith(FILE_PREFIX) and filename.endswith(".conf")):
        flash("非法文件名", "error")
        return redirect(url_for("index"))

    filepath = conf_path(filename)

    if not os.path.exists(filepath):
        flash("文件不存在", "error")
        return redirect(url_for("index"))

    backup_path = None
    try:
        with tempfile.NamedTemporaryFile(delete=False) as f:
            backup_path = f.name

        shutil.copy2(filepath, backup_path)

        code, out, err = run_cmd(["sudo", "/bin/rm", "-f", filepath])
        if code != 0:
            flash(f"删除文件失败: {err or out}", "error")
            return redirect(url_for("index"))

        code, out, err = nginx_test_and_reload()
        if code != 0:
            run_cmd(["sudo", "/bin/cp", backup_path, filepath])
            flash(f"删除后 nginx 校验失败，已回滚: {err or out}", "error")
            return redirect(url_for("index"))

        flash(f"删除成功: {filename}", "success")
        return redirect(url_for("index"))

    except Exception as e:
        if backup_path and os.path.exists(backup_path):
            run_cmd(["sudo", "/bin/cp", backup_path, filepath])
        flash(f"删除失败: {e}", "error")
        return redirect(url_for("index"))

    finally:
        if backup_path and os.path.exists(backup_path):
            os.remove(backup_path)


# =========================
# 一键清理过期配置
# =========================
@app.route("/cleanup_expired", methods=["POST"])
@login_required
def cleanup_expired():
    files = get_managed_files()
    expired_files = [x for x in files if x["expired"]]

    if not expired_files:
        flash("没有过期配置需要清理", "success")
        return redirect(url_for("index"))

    backups = []
    deleted_files = []

    try:
        # 先备份并删除所有过期文件
        for item in expired_files:
            filename = item["name"]
            filepath = conf_path(filename)

            if not os.path.exists(filepath):
                continue

            with tempfile.NamedTemporaryFile(delete=False) as f:
                backup_path = f.name

            shutil.copy2(filepath, backup_path)
            backups.append((filepath, backup_path))
            deleted_files.append(filename)

            code, out, err = run_cmd(["sudo", "/bin/rm", "-f", filepath])
            if code != 0:
                raise Exception(f"删除文件失败 {filename}: {err or out}")

        code, out, err = nginx_test_and_reload()
        if code != 0:
            # 回滚
            for filepath, backup_path in backups:
                run_cmd(["sudo", "/bin/cp", backup_path, filepath])
            flash(f"清理过期配置失败，已回滚: {err or out}", "error")
            return redirect(url_for("index"))

        flash(f"已清理过期配置: {', '.join(deleted_files)}", "success")
        return redirect(url_for("index"))

    except Exception as e:
        for filepath, backup_path in backups:
            if not os.path.exists(filepath) and os.path.exists(backup_path):
                run_cmd(["sudo", "/bin/cp", backup_path, filepath])

        flash(f"清理失败: {e}", "error")
        return redirect(url_for("index"))

    finally:
        for _, backup_path in backups:
            if os.path.exists(backup_path):
                os.remove(backup_path)


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=False)
```

---

# 六、完整前端模板 index.html

创建：

```bash
nano /opt/nginx-conf-manager/templates/index.html
```

写入：

```html
<!doctype html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>Nginx Conf 管理</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 30px;
            background: #f7f7f7;
        }
        h1, h2, h3 {
            color: #333;
        }
        .card {
            background: #fff;
            padding: 20px;
            margin-bottom: 20px;
            border-radius: 8px;
            box-shadow: 0 1px 3px rgba(0,0,0,.08);
        }
        .topbar {
            display: flex;
            justify-content: space-between;
            align-items: center;
        }
        input[type="date"], input[type="number"], input[type="text"], input[type="password"] {
            padding: 8px;
            margin-right: 10px;
            width: 220px;
        }
        button {
            padding: 8px 16px;
            cursor: pointer;
            border: none;
            border-radius: 4px;
            background: #1677ff;
            color: white;
        }
        button.danger {
            background: #d9363e;
        }
        button.secondary {
            background: #666;
        }
        table {
            width: 100%;
            border-collapse: collapse;
            background: #fff;
        }
        table th, table td {
            border: 1px solid #ddd;
            padding: 10px;
        }
        table th {
            background: #f0f0f0;
        }
        .msg {
            padding: 10px;
            margin-bottom: 10px;
            border-radius: 6px;
        }
        .success {
            background: #d4edda;
            color: #155724;
        }
        .error {
            background: #f8d7da;
            color: #721c24;
        }
        .expired {
            color: #d9363e;
            font-weight: bold;
        }
        .normal {
            color: #389e0d;
            font-weight: bold;
        }
        pre {
            background: #f3f3f3;
            padding: 12px;
            border-radius: 6px;
            overflow: auto;
        }
        .stats {
            display: flex;
            gap: 20px;
            flex-wrap: wrap;
        }
        .stat {
            background: #fafafa;
            border: 1px solid #eee;
            padding: 12px 16px;
            border-radius: 6px;
            min-width: 140px;
        }
        .login-box {
            max-width: 420px;
            margin: 80px auto;
        }
        .inline-form {
            display: inline;
        }
    </style>
</head>
<body>

{% with messages = get_flashed_messages(with_categories=true) %}
    {% if messages %}
        {% for category, message in messages %}
            <div class="msg {{ category }}">{{ message }}</div>
        {% endfor %}
    {% endif %}
{% endwith %}

{% if page == "login" %}
    <div class="card login-box">
        <h2>登录</h2>
        <form method="post" action="/login">
            <p>
                <label>用户名：</label><br>
                <input type="text" name="username" required>
            </p>
            <p>
                <label>密码：</label><br>
                <input type="password" name="password" required>
            </p>
            <p>
                <button type="submit">登录</button>
            </p>
        </form>
    </div>
{% else %}
    <div class="card">
        <div class="topbar">
            <div>
                <h1>Nginx Conf 管理</h1>
                <div>管理目录：<code>/etc/nginx/conf.d/</code></div>
            </div>
            <div>
                <form method="post" action="/logout">
                    <button type="submit" class="secondary">退出登录</button>
                </form>
            </div>
        </div>
    </div>

    <div class="card">
        <h3>统计信息</h3>
        <div class="stats">
            <div class="stat">总配置数：<b>{{ total }}</b></div>
            <div class="stat">过期配置数：<b>{{ expired_count }}</b></div>
        </div>
    </div>

    <div class="card">
        <h2>创建配置</h2>
        <form method="post" action="/create">
            <label>EXPIRE：</label>
            <input type="date" name="expire" required>

            <label>listen：</label>
            <input type="number" name="listen" min="1" max="65535" required>

            <button type="submit">创建并重载</button>
        </form>

        <h3>固定模板</h3>
        <pre># EXPIRE: YYYY-MM-DD
server {
    listen PORT;

    if ($whitelist_usergent = 0) {
        return 444;
    }

    location / {
        # 1. 设置转发目标 (注意协议是 https)
        proxy_pass http://127.0.0.1:19321;
    }
}</pre>
    </div>

    <div class="card">
        <div class="topbar">
            <h2>已管理配置</h2>
            <form method="post" action="/cleanup_expired" onsubmit="return confirm('确认一键清理所有过期配置吗？');">
                <button type="submit" class="danger">一键清理过期配置</button>
            </form>
        </div>

        <table>
            <thead>
                <tr>
                    <th>文件名</th>
                    <th>EXPIRE</th>
                    <th>listen</th>
                    <th>状态</th>
                    <th>操作</th>
                </tr>
            </thead>
            <tbody>
            {% for file in files %}
                <tr>
                    <td>{{ file.name }}</td>
                    <td>{{ file.expire }}</td>
                    <td>{{ file.listen }}</td>
                    <td>
                        {% if file.expired %}
                            <span class="expired">已过期</span>
                        {% else %}
                            <span class="normal">正常</span>
                        {% endif %}
                    </td>
                    <td>
                        <form method="post" action="/delete" class="inline-form" onsubmit="return confirm('确认删除 {{ file.name }} 吗？');">
                            <input type="hidden" name="filename" value="{{ file.name }}">
                            <button type="submit" class="danger">删除并重载</button>
                        </form>
                    </td>
                </tr>
            {% endfor %}

            {% if not files %}
                <tr>
                    <td colspan="5">暂无 managed_*.conf 配置</td>
                </tr>
            {% endif %}
            </tbody>
        </table>
    </div>
{% endif %}

</body>
</html>
```

---

# 七、创建运行用户

建议用专门用户运行：

```bash
useradd -r -s /usr/sbin/nologin nginxmgr
chown -R nginxmgr:nginxmgr /opt/nginx-conf-manager
```

---

# 八、配置 sudo 免密

编辑 sudoers：

```bash
visudo
```

加入这一行：

```sudoers
nginxmgr ALL=(root) NOPASSWD: /bin/cp, /bin/rm, /usr/sbin/nginx
```

如果你的 nginx 路径不同，先查：

```bash
which nginx
```

比如结果是 `/usr/sbin/nginx`，那就保持上面配置不变。

---

# 九、先手工测试运行

```bash
cd /opt/nginx-conf-manager
sudo -u nginxmgr python3 app.py
```

浏览器访问：

```bash
http://服务器IP:5000/login
```

默认登录账号密码是：

- 用户名：`admin`
- 密码：`ChangeMe123!`

**上线前请改掉 app.py 里的默认密码。**

---

# 十、systemd 服务

创建：

```bash
nano /etc/systemd/system/nginx-conf-manager.service
```

内容：

```ini
[Unit]
Description=Nginx Conf Manager
After=network.target

[Service]
User=nginxmgr
Group=nginxmgr
WorkingDirectory=/opt/nginx-conf-manager
ExecStart=/usr/bin/python3 /opt/nginx-conf-manager/app.py
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

启动：

```bash
systemctl daemon-reload
systemctl enable nginx-conf-manager
systemctl start nginx-conf-manager
systemctl status nginx-conf-manager
```

---

# 十一、推荐再用 nginx 反代这个管理页面

比如你不想直接暴露 5000 端口，可以加一个 nginx 配置：

```nginx
server {
    listen 8080;
    server_name _;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

然后：

```bash
nginx -t && nginx -s reload
```

访问：

```bash
http://服务器IP:8080/login
```

---

# 十二、建议你立刻改的内容

打开 `app.py`，至少改下面两项：

## 1）改 Flask secret_key

```python
app.secret_key = "please-change-to-a-long-random-secret-key"
```

改成随机长字符串。

你可以用：

```bash
python3 -c 'import secrets; print(secrets.token_hex(32))'
```

---

## 2）改登录密码

```python
ADMIN_USERNAME = "admin"
ADMIN_PASSWORD = "ChangeMe123!"
```

改成你自己的。

---

# 十三、这个版本的工作机制

## 创建配置

1. 登录后填写 EXPIRE 和 listen
2. 程序检查：
   - 日期格式
   - 端口是否合法
   - 端口是否已经被已有 `.conf` 占用
3. 生成 `/etc/nginx/conf.d/managed_xxx.conf`
4. 执行：
   - `nginx -t`
   - `nginx -s reload`
5. 如果失败，自动回滚删除新文件

## 删除单个配置

1. 删除前先备份
2. 删除文件
3. `nginx -t && nginx -s reload`
4. 如果失败，自动恢复备份

## 一键清理过期配置

1. 找出 `EXPIRE < 今天` 的 managed 配置
2. 逐个备份后删除
3. 统一执行 nginx 校验与重载
4. 如果失败，自动回滚所有已删文件

---

# 十四、注意事项

## 1）`$whitelist_usergent` 必须在 nginx 其他配置中已定义

否则 `nginx -t` 会失败。

## 2）目前登录是最简单的内置账号密码

如果你想更安全，可以继续升级成：

- 密码哈希存储
- 多用户
- IP 白名单
- HTTPS only
- CSRF 防护

## 3）端口检查是扫描 `.conf` 中的 `listen xxx;`

如果你的其他配置写法很复杂，比如：

- `listen 10407 ssl;`
- `listen [::]:10407;`
- `listen 0.0.0.0:10407;`

那我建议我再帮你把端口检测正则增强一下。

---

# 十五、我建议你下一步再做两个增强

如果你愿意，我还可以继续直接给你：

## 方案1：更安全版

- 密码哈希，不明文存储
- Session 有效期
- 登录失败次数限制
- 仅允许指定 IP 访问

## 方案2：单文件版

把 HTML 模板也内嵌到 Python，部署时只要一个 `app.py`

## 方案3：自动定时清理版

加一个 systemd timer / cron，每天自动清理过期 conf

---

如果你愿意，我下一条可以直接继续给你：

**“更安全版（密码哈希 + Session 过期 + IP 白名单）完整代码”**  
或者  
**“自动每天凌晨清理过期 conf 的 systemd timer 配置”**。

---

## 🧑‍💻 User

方案3：自动定时清理版

---

## 🤖 Assistant

下面给你做 **方案3：自动定时清理过期 conf**。  
我给你两种方式，推荐第一种：

---

# 方案选择

## 方案A：systemd timer（推荐）

优点：

- Debian 11 原生支持
- 比 cron 更容易看日志
- 更适合做服务化管理

## 方案B：cron

优点：

- 简单
- 老方案通用

---

我建议你用 **systemd timer**。

---

# 一、思路

自动任务每天定时执行一个清理脚本，流程如下：

1. 扫描 `/etc/nginx/conf.d/managed_*.conf`
2. 读取文件第一行中的：
   
   ```nginx
   # EXPIRE: YYYY-MM-DD
   ```
3. 如果过期时间 `< 今天`
   - 删除该文件
4. 删除完成后执行：
   
   ```bash
   nginx -t && nginx -s reload
   ```
5. 如果 nginx 校验失败，则自动回滚恢复已删除文件

---

# 二、推荐做法：独立清理脚本

不要直接调用 Flask 页面接口，而是写一个**独立 Python 脚本**，便于 systemd 定时运行。

目录结构变成：

```bash
/opt/nginx-conf-manager/
├── app.py
├── cleanup_expired.py
└── templates/
    └── index.html
```

---

# 三、清理脚本 cleanup_expired.py

创建文件：

```bash
nano /opt/nginx-conf-manager/cleanup_expired.py
```

写入以下代码：

```python
import os
import re
import shutil
import subprocess
import tempfile
from datetime import datetime, date

NGINX_CONF_DIR = "/etc/nginx/conf.d"
FILE_PREFIX = "managed_"
NGINX_BIN = "/usr/sbin/nginx"


def run_cmd(cmd_list):
    result = subprocess.run(cmd_list, text=True, capture_output=True)
    return result.returncode, result.stdout, result.stderr


def nginx_test_and_reload():
    code, out, err = run_cmd(["sudo", NGINX_BIN, "-t"])
    if code != 0:
        return code, out, err

    code, out2, err2 = run_cmd(["sudo", NGINX_BIN, "-s", "reload"])
    return code, (out or "") + (out2 or ""), (err or "") + (err2 or "")


def parse_expire(path):
    try:
        with open(path, "r", encoding="utf-8") as f:
            content = f.read()

        m = re.search(r"^# EXPIRE:\s*([0-9]{4}-[0-9]{2}-[0-9]{2})", content, re.MULTILINE)
        if not m:
            return None

        return datetime.strptime(m.group(1), "%Y-%m-%d").date()
    except Exception:
        return None


def get_expired_files():
    expired_files = []

    if not os.path.isdir(NGINX_CONF_DIR):
        return expired_files

    today = date.today()

    for filename in sorted(os.listdir(NGINX_CONF_DIR)):
        if not (filename.startswith(FILE_PREFIX) and filename.endswith(".conf")):
            continue

        path = os.path.join(NGINX_CONF_DIR, filename)
        exp = parse_expire(path)
        if exp and exp < today:
            expired_files.append(path)

    return expired_files


def main():
    expired_files = get_expired_files()

    if not expired_files:
        print("No expired managed conf files found.")
        return 0

    backups = []
    deleted = []

    try:
        for filepath in expired_files:
            with tempfile.NamedTemporaryFile(delete=False) as f:
                backup_path = f.name

            shutil.copy2(filepath, backup_path)
            backups.append((filepath, backup_path))

            code, out, err = run_cmd(["sudo", "/bin/rm", "-f", filepath])
            if code != 0:
                raise Exception(f"Failed to delete {filepath}: {err or out}")

            deleted.append(filepath)

        code, out, err = nginx_test_and_reload()
        if code != 0:
            for filepath, backup_path in backups:
                run_cmd(["sudo", "/bin/cp", backup_path, filepath])
            print(f"Cleanup failed, rollback completed: {err or out}")
            return 1

        print("Expired conf cleaned successfully:")
        for item in deleted:
            print(f" - {item}")
        return 0

    except Exception as e:
        for filepath, backup_path in backups:
            if not os.path.exists(filepath) and os.path.exists(backup_path):
                run_cmd(["sudo", "/bin/cp", backup_path, filepath])
        print(f"Cleanup failed: {e}")
        return 1

    finally:
        for _, backup_path in backups:
            if os.path.exists(backup_path):
                os.remove(backup_path)


if __name__ == "__main__":
    raise SystemExit(main())
```

---

# 四、赋权

```bash
chown nginxmgr:nginxmgr /opt/nginx-conf-manager/cleanup_expired.py
chmod 755 /opt/nginx-conf-manager/cleanup_expired.py
```

---

# 五、sudo 权限确认

因为这个脚本也要删文件和 reload nginx，所以 sudoers 里要有：

```sudoers
nginxmgr ALL=(root) NOPASSWD: /bin/cp, /bin/rm, /usr/sbin/nginx
```

编辑：

```bash
visudo
```

---

# 六、先手工测试

先人工执行一次：

```bash
sudo -u nginxmgr python3 /opt/nginx-conf-manager/cleanup_expired.py
```

如果没有过期文件，会输出：

```bash
No expired managed conf files found.
```

如果有过期文件，会输出类似：

```bash
Expired conf cleaned successfully:
 - /etc/nginx/conf.d/managed_10407_2024-01-01.conf
 - /etc/nginx/conf.d/managed_10408_2024-01-02.conf
```

---

# 七、创建 systemd service

创建服务文件：

```bash
nano /etc/systemd/system/nginx-conf-cleanup.service
```

内容如下：

```ini
[Unit]
Description=Cleanup expired managed nginx conf files

[Service]
Type=oneshot
User=nginxmgr
Group=nginxmgr
ExecStart=/usr/bin/python3 /opt/nginx-conf-manager/cleanup_expired.py
```

---

# 八、创建 systemd timer

创建定时器文件：

```bash
nano /etc/systemd/system/nginx-conf-cleanup.timer
```

内容如下：

```ini
[Unit]
Description=Run nginx conf cleanup daily

[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

---

# 九、启用 timer

执行：

```bash
systemctl daemon-reload
systemctl enable nginx-conf-cleanup.timer
systemctl start nginx-conf-cleanup.timer
```

查看状态：

```bash
systemctl status nginx-conf-cleanup.timer
```

查看下一次运行时间：

```bash
systemctl list-timers | grep nginx-conf-cleanup
```

---

# 十、查看执行日志

每次定时运行后，可以看日志：

```bash
journalctl -u nginx-conf-cleanup.service
```

如果要看今天的：

```bash
journalctl -u nginx-conf-cleanup.service --since today
```

如果要实时看：

```bash
journalctl -u nginx-conf-cleanup.service -f
```

---

# 十一、立即手动触发一次定时服务

你可以立即测试：

```bash
systemctl start nginx-conf-cleanup.service
```

然后看日志：

```bash
journalctl -u nginx-conf-cleanup.service -n 50 --no-pager
```

---

# 十二、如果你想改成每天凌晨 1 点执行

把 timer 改成：

```ini
[Timer]
OnCalendar=*-*-* 01:00:00
Persistent=true
```

修改后执行：

```bash
systemctl daemon-reload
systemctl restart nginx-conf-cleanup.timer
```

---

# 十三、如果你更喜欢 cron 写法

也可以不用 timer，直接用 cron。

编辑 crontab：

```bash
crontab -u nginxmgr -e
```

加入：

```cron
0 3 * * * /usr/bin/python3 /opt/nginx-conf-manager/cleanup_expired.py >> /var/log/nginx-conf-cleanup.log 2>&1
```

不过这需要考虑 `sudo` 环境和日志管理，所以我还是建议用 systemd timer。

---

# 十四、建议你顺手给 Flask 页面增加“下次自动清理时间”提示

如果你愿意，我下一步还能继续帮你做两个增强：

## 增强1

在管理页面上显示：

- 自动清理已启用
- 下次运行时间
- 上次运行结果

## 增强2

增加一个开关：

- 是否启用自动清理
- 修改清理时间

---

# 十五、最终你要执行的完整命令汇总

## 1. 创建脚本

```bash
nano /opt/nginx-conf-manager/cleanup_expired.py
```

## 2. 设置权限

```bash
chown nginxmgr:nginxmgr /opt/nginx-conf-manager/cleanup_expired.py
chmod 755 /opt/nginx-conf-manager/cleanup_expired.py
```

## 3. 测试执行

```bash
sudo -u nginxmgr python3 /opt/nginx-conf-manager/cleanup_expired.py
```

## 4. 创建 service

```bash
nano /etc/systemd/system/nginx-conf-cleanup.service
```

## 5. 创建 timer

```bash
nano /etc/systemd/system/nginx-conf-cleanup.timer
```

## 6. 启用

```bash
systemctl daemon-reload
systemctl enable nginx-conf-cleanup.timer
systemctl start nginx-conf-cleanup.timer
```

## 7. 查看

```bash
systemctl list-timers | grep nginx-conf-cleanup
journalctl -u nginx-conf-cleanup.service -n 50 --no-pager
```

---

如果你愿意，我下一条可以直接继续给你：

**“把自动清理状态显示到当前管理页面里”的完整代码修改版**。
