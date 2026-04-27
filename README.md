# centos-mihomo-docker-
centos内核安装mihomo，并设置docker可以使用


### 1、安装及测试

```bash
mkdir ~/clash && cd ~/clash

# 下载（compatible 版兼容旧内核，适合你的 CentOS 7）
VERSION=
wget https://github.com/MetaCubeX/mihomo/releases/download/v1.19.16/mihomo-linux-amd64-compatible-v1.19.16.gz

# 解压
gzip -d mihomo-linux-amd64-compatible-v1.19.16.gz

# 重命名 + 加执行权限
mv mihomo-linux-amd64-compatible-v1.19.16 mihomo
chmod +x mihomo

# 验证
./mihomo -v
```


如果提示：mihomo: 未找到命令...

把它移到系统路径里，以后直接用 `mihomo` 命令：

```bash
mv /安装路径/mihomo /usr/local/bin/mihomo

# 然后就可以直接用了
mihomo -v
```


然后

```bash
cd /data/clash

# 下载订阅配置
wget -O config.yaml "你的订阅链接"

# 启动
mihomo -d /data/clash
```


新开一个终端窗口，执行：

```bash
export http_proxy=http://127.0.0.1:7890
export https_proxy=http://127.0.0.1:7890

curl -I https://www.google.com

# 返回 HTTP/2 200 就说明代理工作正常。
```


### 2、设置开机自启

**第一步：创建 systemd 服务文件**

```bash
vi /etc/systemd/system/mihomo.service
```


写入以下内容：

```bash
[Unit]
Description=mihomo proxy service
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/mihomo -d /data1/clash
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```


**第二步：启用并启动**

```bash
# 先杀掉当前手动启动的 mihomo
pkill mihomo

# 重载 systemd 配置
systemctl daemon-reload

# 开机自启
systemctl enable mihomo

# 立即启动
systemctl start mihomo

# 查看状态
systemctl status mihomo
```

状态显示 `Active: active (running)` 就成功了。

------

**常用管理命令：**

```bash
systemctl stop mihomo      # 停止
systemctl restart mihomo   # 重启
journalctl -u mihomo -f    # 实时查看日志
```




### **3、web面板**

访问：

```
https://metacubex.github.io/metacubexd/
```


然后后端地址填你服务器的 **IP**：

```bash
http://x.x.x.x:9090
```




### 4、docker使用

```bash
# 创建 docker 代理配置目录
mkdir -p /etc/systemd/system/docker.service.d

# 创建代理配置文件
cat > /etc/systemd/system/docker.service.d/proxy.conf << EOF
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:7890"
Environment="HTTPS_PROXY=http://127.0.0.1:7890"
Environment="NO_PROXY=localhost,127.0.0.1"
EOF

# 重载配置并重启 docker
systemctl daemon-reload
systemctl restart docker

# 测试
docker pull imagesname
```


以后所有 `docker pull` 都会走代理，包括：

- `docker pull`
- `docker build`（构建时拉取基础镜像）
- `docker-compose up`（拉取镜像时）

**唯一注意的是**：这是 Docker 守护进程的代理，解决的是**拉取镜像**的问题。

容器内部的网络请求不走这个代理，如果容器内也需要代理，需要在 `docker run` 时单独传环境变量：docker run -e http_proxy=http://127.0.0.1:7890 -e https_proxy=http://127.0.0.1:7890 ...



若想让配置失效：

删掉配置文件然后重启 Docker 就行：

```
rm /etc/systemd/system/docker.service.d/proxy.conf

systemctl daemon-reload
systemctl restart docker
```


或者如果以后还可能用到，不想删文件，可以只把文件内容清空或注释掉：

```
vi /etc/systemd/system/docker.service.d/proxy.conf
```


把三行 `Environment` 前面加 `#` 注释掉，然后同样重启 Docker。




## Docker Pull 代理快速开关方案

### 1. 编辑 ~/.bashrc

```bash
vi ~/.bashrc
```

在文件末尾添加：

```bash
alias docker-proxy-on='mkdir -p /etc/systemd/system/docker.service.d && printf "[Service]\nEnvironment=\"HTTP_PROXY=http://127.0.0.1:7890\"\nEnvironment=\"HTTPS_PROXY=http://127.0.0.1:7890\"\nEnvironment=\"NO_PROXY=localhost,127.0.0.1\"\n" > /etc/systemd/system/docker.service.d/proxy.conf && systemctl daemon-reload && systemctl restart docker && echo "Docker 代理已开启"'

alias docker-proxy-off='rm -f /etc/systemd/system/docker.service.d/proxy.conf && systemctl daemon-reload && systemctl restart docker && echo "Docker 代理已关闭"'
```

### 2. 生效配置

```bash
source ~/.bashrc
```

### 3. 日常使用

```bash
docker-proxy-on    # 开启代理
docker pull xxx    # 拉取镜像
docker-proxy-off   # 关闭代理
```

### 4. 验证是否生效

```bash
# 查看配置文件是否存在
ls -la /etc/systemd/system/docker.service.d/

# 查看文件内容
cat /etc/systemd/system/docker.service.d/proxy.conf

# 查看 Docker 是否读取到代理环境变量
systemctl show docker | grep -i proxy
```

> **注意：** 代理地址 `127.0.0.1:7890` 根据你本机代理端口自行修改。
