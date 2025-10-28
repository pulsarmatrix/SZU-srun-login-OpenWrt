# OpenWrt 路由器 Srun 自动登录

本教程基于[深圳大学深澜校园网登录python脚本(Matt-Dong123/SZU-srun-login)](https://github.com/Matt-Dong123/SZU-srun-login)，解决将其部署在 OpenWrt 路由器上时遇到的问题。

**本教程没有修改原项目的代码，需要自行修改！！！**

**本教程没有修改原项目的代码，需要自行修改！！！**

**本教程没有修改原项目的代码，需要自行修改！！！**

## 1. 原脚本无法联网原因分析

直接在OpenWrt上使用原脚本，会遇到无法找到校园网登录网页地址问题，其实是校园网认证系统的“强制门户”机制导致的问题。

- **问题一：DNS 劫持失败**

  - **原理**：Srun 系统依赖“DNS 劫持”。当用户（未登录）访问任何网址时，它本应拦截用户的 DNS 请求，并劫持目标网址的 IP 为“登录页”的 IP。

  - **现状**：当用户的 OpenWrt 路由器去请求 DNS 时，Srun 系统无法正常响应。它本应劫持用户*手机/电脑*的请求，而不是*路由器本身*的请求。因此不返回 IP，导致 DNS 解析失败。

- **问题二：IP 地址错配**

  - **原理**：`apis.py` 中的 `get_ip()` 函数会访问登录页，从 HTML 中抓取用户当前的 IP。Srun 系统需要这个 IP 来进行用户认证。

  - **现状**：如果用户在路由器*后面*（LAN 口）运行脚本，`get_ip()` 抓到的是用户的内网 IP（如 `192.168.x.x`），而不是 Srun 想要的路由器 WAN 口 IP。Srun 不识别这个 `192` 开头的 IP，自然会认证失败。

## 2. 解决方案

### 步骤一：修复 DNS（手动指定IP）

用 `hosts` 文件绕过 DNS，指定系统登录页的 IP。

1. **拔掉路由器**，将电脑（非OpenWrt设备）**直连**校园网口。

2. **不要登录**，在终端运行 `ping net.szu.edu.cn`。

3. 记下返回的真实 IP 地址（假设是 `10.1.2.3`）。

4. 把电脑**接回路由器 LAN 口**。

5. **修改运行脚本的Openwrt设备 `hosts` 文件**，在终端运行

```bash
vi /etc/hosts
```

6. 在文件末尾添加一行（IP 换成刚刚 ping 到的）：

```
10.1.2.3   net.szu.edu.cn
```

### 步骤二：修复 IP（硬编码）

修改 `apis.py`，禁止它自动抓取 IP，直接告诉它正确的 IP。

1. 登录 OpenWrt 管理后台，找到路由器从校园网获取的 **WAN 口 IP**。

2. 打开 `apis.py` 文件。

3. 找到 `login` 方法：

   ```python
   # 修改前:
   def login(self, username, password):
       self.username = username
       self.password = password

       self.get_ip()
       self.get_token()
       self.get_login_responce()
   ```
   
4. **修改代码**，注释掉 `get_ip()`，并手动指定 `self.ip`：

   ```python
   # 修改后:
   def login(self, username, password):
       self.username = username
       self.password = password

       # 1. 注释掉自动获取 IP
       # self.get_ip() 
       
       # 2. 硬编码路由器 WAN 口 IP
       self.ip = "172.31.**.**" 
       
       # 3. 保留后续步骤
       self.get_token()
       self.get_login_responce()
   ```
   
5. **重要**：若这个 WAN 口 IP 变化，需要回来修改这个文件。

## 3. 部署到 OpenWrt（持久运行）

现在把修改好的脚本部署到 OpenWrt 路由器上，让它在后台“无人值守”地运行。

### 步骤一：准备环境

1. 登录 OpenWrt 终端（SSH）。

2. 安装 Python 和 `requests` 包：

```bash
opkg update
opkg install python3 python3-requests
```

### 步骤二：后台运行（`nohup`）

`nohup` 是 OpenWrt (BusyBox) 自带的工具，非常轻量，适合后台任务。

1. 进入脚本目录：

```bash
cd /root/SZU-srun-login-OpenWrt/
```
   
2. 使用 `nohup` 启动脚本：

```bash
nohup python3 main.py > /dev/null 2>&1 &
```

### 步骤三：检查与停止

- **检查运行**：`ps | grep main.py`

- **停止脚本**：`kill <找到的进程号PID>`

## 4. 开机自启动

使用 OpenWrt 经典的 `rc.local` 脚本，实现开机自动执行上面的 `nohup` 命令。

1. 编辑 `rc.local` 文件：

```bash
vi /etc/rc.local
```
   
2. 在文件末尾的 `exit 0` 这一行 **之前**，加入启动命令（使用绝对路径）：

```bash
# 在 exit 0 之前加入下面这行:
nohup python3 /root/SZU-srun-login-OpenWrt/main.py > /dev/null 2>&1 &
exit 0
```

3. **警告**：末尾的 `&` 符号**至关重要**！如果忘了加，脚本（一个无限循环）会卡住整个开机进程。

4. 保存文件并重启路由器（`reboot`）进行测试。
