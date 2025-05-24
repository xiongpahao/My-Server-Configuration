# My-Server-Configuration
在服务器上搭建Gost-v3代理和NextChat服务

(本项目的前一部分是基于已故大佬[haoel的项目](https://github.com/haoel/haoel.github.io)改编而成的，大佬R.I.P。)

## 1. 准备工作

### 1.1 安装Docker
参考docker官方的安装文档：

- [CentOS 上的 Docker CE 安装](https://docs.docker.com/install/linux/docker-ce/centos/)
- [Ubuntu 上的 Docker CE 安装](https://docs.docker.com/install/linux/docker-ce/ubuntu/)

### 1.2 开启 TCP BBR 拥塞控制算法
TCP BBR（Bottleneck Bandwidth and Round-trip propagation time）是由Google设计，于2016年发布的拥塞算法。以往大部分拥塞算法是基于丢包来作为降低传输速率的信号，而BBR则基于模型主动探测。该算法使用网络最近出站数据分组当时的最大带宽和往返时间来创建网络的显式模型。数据包传输的每个累积或选择性确认用于生成记录在数据包传输过程和确认返回期间的时间内所传送数据量的采样率。该算法认为随着网络接口控制器逐渐进入千兆速度时，分组丢失不应该被认为是识别拥塞的主要决定因素，所以基于模型的拥塞控制算法能有更高的吞吐量和更低的延迟，可以用BBR来替代其他流行的拥塞算法，例如CUBIC。Google在YouTube上应用该算法，将全球平均的YouTube网络吞吐量提高了4%，在一些国家超过了14%。

BBR之后移植入Linux内核4.9版本，并且对于QUIC可用。

如果开启，请参看 《[开启TCP BBR拥塞控制算法](https://github.com/iMeiji/shadowsocks_install/wiki/开启-TCP-BBR-拥塞控制算法) 》

### 1.3 绑定域名和证书

1）在域名提供商（如[namecheap](https://www.namecheap.com)）申请好域名，然后为子域名添加一条A记录指向VPS的IP地址，或者添加一条CNAME记录指向VPS的DNS名称。
![](images/namecheapDNSconfig.jpg)

2）使用 [Let's Encrypt](https://letsencrypt.org) 来签一个证书。首先需要在服务器上安装一个 [certbot](https://certbot.eff.org/instructions)，点击 [certbot](https://certbot.eff.org/instructions) 这个链接，你可以选择你的服务器，操作系统，然后就跟着指令走吧。

接下来，你需要申请一个证书（我们使用standalone的方式，然后，你需要输入你的电子邮件和你解析到 VPS 的域名）：

```shell
$ sudo certbot certonly --standalone
```

证书默认生成在 `/etc/letsencrypt/live/<YOUR.DOMAIN.COM/>` 目录下，证书有效期为90天，在后面我们会设置certbot自动在后台更新证书。


## 2. 搭建Gost v3代理

### 2.1 搭建服务端

[gost](https://github.com/go-gost/gost) 是一个非常强的代理服务，它可以设置成 HTTPS 代理，然后把你的服务伪装成一个Web服务器，**这比其它的流量伪装更好，也更隐蔽。这也是这里强烈推荐的一个方式**。

接下来就是启动 Gost 服务了，我们这里通过 Docker 建立 Gost 服务器。

**Note**
>此处使用的是Gost的v3版本，Gost v2版本的搭建可以参考[https://github.com/xiongpahao/Magical-Proxy](https://github.com/xiongpahao/Magical-Proxy)

```shell
#!/bin/bash

# 下面的四个参数需要改成你的
DOMAIN="YOUR_DOMAIN"
USER="username"
PASS="password"
PORT=2333

BIND_IP=0.0.0.0
CERT_DIR=/etc/letsencrypt
CERT=${CERT_DIR}/live/${DOMAIN}/fullchain.pem
KEY=${CERT_DIR}/live/${DOMAIN}/privkey.pem
sudo docker run -d --name gost \
    -v ${CERT_DIR}:${CERT_DIR}:ro \
    --net=host gogost/gost \
    -L "http2://${USER}:${PASS}@${BIND_IP}:${PORT}?certFile=${CERT}&keyFile=${KEY}&probeResistance=code:404&knock=www.google.com"
```
上面这个脚本中，需要配置：域名(`DOMAIN`), 用户名 (`USER`), 密码 (`PASS`) 和 端口号(`PORT`) 这几个变量。

关于 gost 的参数， 可以参看其文档：[Gost Wiki](https://gost.run/)，上面所设置的参数 `probeResistance=code:404` 意思是，如果服务器被探测，或是用浏览器来访问，返回404错误，也可以返回一个网页（如：`probeResistance=file:/path/to/file.txt` 或其它网站 `probeResistance=web:example.com/page.html`）

**Note**
>
> 1. 开启了探测防御功能后，当认证失败时服务器默认不会响应 `407 Proxy Authentication Required`，但某些情况下客户端需要服务器告知代理是否需要认证(例如Chrome中的 SwitchyOmega 插件)。通过knock参数设置服务器才会发送407响应。对于上面的例子，我们的`knock`参数配置的是`www.google.com`，所以，你需要先访问一下 `https://www.google.com` 让服务端返回一个 `407` 后，SwitchyOmega 才能正常工作。
>
> 2. 如果认证信息（也就是用户名和密码）中包含特殊字符，则可以（应该是必须！否则客户端一侧会有很多不兼容）通过auth参数来设置，下面是使用 `auth` 参数的例子：
>
>    ```shell
>    DOMAIN="YOU.DOMAIN.NAME"
>    USER="username"
>    PASS="password"
>    PORT=2333
>    AUTH=$(echo -n ${USER}:${PASS} | base64)
>
>    BIND_IP=0.0.0.0
>    CERT_DIR=/etc/letsencrypt
>    CERT=${CERT_DIR}/live/${DOMAIN}/fullchain.pem
>    KEY=${CERT_DIR}/live/${DOMAIN}/privkey.pem
>    sudo docker run -d --name gost \
>        -v ${CERT_DIR}:${CERT_DIR}:ro \
>        --net=host gogost/gost \
>        -L "http2://${BIND_IP}:${PORT}?auth=${AUTH}&certFile=${CERT}&keyFile=${KEY}&probeResistance=code:404&knock=www.google.> com"
>    ```

如无意外，你的服务就已经启动了。 你可以使用如下命令检查有没有启动成功：

-  `sudo docker ps` 来查看 gost 是否在运行。
-  `netstat -nolp | grep 2333` 来查看 gost 是否在监听 2333 端口。
-  `sudo docker logs gost` 来查看 gost 的日志。

#### 2.1.1 使用Cloudflare原生IP

1) 用Docker安装 Cloudflare Warp

首先在任意目录创建并编辑docker-compose.yml文件：

```shell
    vim docker-compose.yml
```
把下面这段代码粘贴进去，然后保存（键盘：ESC --> : --> wq）：

```shell
    version: "3"

services:
  warp:
    image: caomingjun/warp
    container_name: warp
    restart: always
    # add removed rule back (https://github.com/opencontainers/runc/pull/3468)
    device_cgroup_rules:
      - 'c 10:200 rwm'
    ports:
      - "1080:1080"
    environment:
      - WARP_SLEEP=2
      # - WARP_LICENSE_KEY= # optional
      # - WARP_ENABLE_NAT=1 # enable nat
    cap_add:
      # Docker already have them, these are for podman users
      - MKNOD
      - AUDIT_WRITE
      # additional required cap for warp, both for podman and docker
      - NET_ADMIN
    sysctls:
      - net.ipv6.conf.all.disable_ipv6=0
      - net.ipv4.conf.all.src_valid_mark=1
      # uncomment for nat
      # - net.ipv4.ip_forward=1
      # - net.ipv6.conf.all.forwarding=1
      # - net.ipv6.conf.all.accept_ra=2
    volumes:
      - ./data:/var/lib/cloudflare-warp
```
最后在该目录中运行以下命令：

```shell
    docker compose up -d
```

这条命令会在容器上的 1080端口 开启一个 socks5 代理，接下来查看这个容器的 ip:

```shell
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' warp
```

然后可以通过 curl 镜像来测试，例如，如果容器的 ip 是 `172.18.0.2`，则可以运行:

```shell
docker run --rm curlimages/curl --connect-timeout 2 -x "socks5://172.18.0.2:1080" ipinfo.io
```

返回的结果中 `org` 字段应该能看到 Cloudflare 相关的信息。

2) 再建立一个Gost容器，并将其流量转发给Warp

```shell
#!/bin/bash

# 下面的四个参数需要改成你的
DOMAIN="YOUR_DOMAIN"
USER="username"
PASS="password"
PORT=2334

BIND_IP=0.0.0.0
CERT_DIR=/etc/letsencrypt
CERT=${CERT_DIR}/live/${DOMAIN}/fullchain.pem
KEY=${CERT_DIR}/live/${DOMAIN}/privkey.pem
sudo docker run -d --name gost-warp \
    -v ${CERT_DIR}:${CERT_DIR}:ro \
    --net=host gogost/gost \
    -L "http2://${USER}:${PASS}@${BIND_IP}:${PORT}?certFile=${CERT}&keyFile=${KEY}&probeResistance=code:404&knock=www.google.com" \
    -F "socks5://172.18.0.2:1080"
```

上面的脚本会建立一个监听2334端口的Gost容器，并将其流量转发给Warp。

**Note**
>使用bypass参数可以对流量进行条件转发，-F "socks5://172.18.0.2:1080?bypass=~*.reddit.com"表示当且仅当访问域名包含“reddit.com”的网站时才转发流量到Warp。
>关于Gost分流参数“bypass”的使用方法可以参考[官方文档](https://gost.run/concepts/bypass/)

#### 2.1.2设置证书自动更新
使用命令`crontab -e`来编辑定时任务：
```shell
0 0 1 * * /usr/bin/certbot renew --force-renewal
5 0 1 * * /usr/bin/docker restart gost
5 0 1 * * /usr/bin/docker restart gost-warp
```

### 2.2 配置客户端
参考[https://github.com/xiongpahao/Magical-Proxy](https://github.com/xiongpahao/Magical-Proxy)

## 3. 搭建NextChat服务

### 3.1 试用Docker搭建NextChat服务（[项目主页](https://github.com/ChatGPTNextWeb/ChatGPT-Next-Web)）

使用Docker来部署：

```shell
docker pull yidadaa/chatgpt-next-web

docker run -d -p 3000:3000 \
   -e BASE_URL=http://172.17.0.3:8080 \
   -e OPENAI_API_KEY=YOUR_API_KEY \
   -e CODE=your-password \
   yidadaa/chatgpt-next-web
```

使用以下命令来确认chatgpt-next-web已经成功运行且监听3000端口：
```shell
docker ps
```
如果没有问题，此时就可以在浏览器中通过地址`http://your_vps_ip:3000`来访问NextChat了（注意需要先输入访问密码，才能使用chatGPT）。

### 3.3 通过Caddy方向代理NextChat，实现通过HTTPS访问

[Caddy](https://caddyserver.com/docs/) 可以很方便地为端口服务提供 HTTPS 支持，自动管理证书，省心省力。

以下是一个 Debian/Ubuntu 系统上使用 Caddy 的示例，其他系统请参考 [Caddy 官方文档](https://caddyserver.com/docs/)。

#### 安装 Caddy

```shell
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

#### 配置 Caddy

```shell
sudo vi /etc/caddy/Caddyfile
```

假设已经与VPS绑定的域名为 `your.domain.com`，在 Caddyfile 中添加以下内容：

```shell
your.domain.com {
    reverse_proxy localhost:3000
}
```
#### 启动 Caddy

执行以下命令启动 Caddy：

```shell
# 启动 Caddy
sudo systemctl start caddy

# 设置 Caddy 开机自启
sudo systemctl enable caddy

# 查看 Caddy 运行状态
sudo systemctl status caddy
```

如果没有问题的话，那此时就可以通过 `https://your.domain.com` 访问 NextChat 服务了。
