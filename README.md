# My-Server-Configuration
在服务器上搭建Gost代理、NextChat+Copilot to GPT4等服务

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

证书默认生成在 `/etc/letsencrypt/live/<YOUR.DOMAIN.COM/>` 目录下，证书有效期为90天，不过不用担心，certbot会自动在后台更新证书。


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
PORT=443

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
> 2. 如果认证信息（也就是用户名和密码）中包含特殊字符，则可以（应该是必须！否则客户端一侧会有很多不兼容）通过auth参数来设置，下面是使用 `auth` 参数的例子（注意，需要 Gost 在 2.9.2+ 以上版本）：
>
>    ```shell
>    DOMAIN="YOU.DOMAIN.NAME"
>    USER="username"
>    PASS="password"
>    PORT=443
>    AUTH=$(echo -n ${USER}:${PASS} | base64)
>
>    BIND_IP=0.0.0.0
>    CERT_DIR=/etc/letsencrypt
>    CERT=${CERT_DIR}/live/${DOMAIN}/fullchain.pem
>    KEY=${CERT_DIR}/live/${DOMAIN}/privkey.pem
>    sudo docker run -d --name gost \
>        -v ${CERT_DIR}:${CERT_DIR}:ro \
>        --net=host gogost/gost \
>        -L "http2://${BIND_IP}:${PORT}?auth=${AUTH}&certFile=${CERT}&keyFile=${KEY}&probeResistance=code:404&knock=www.go   ogle.> com"
>    ```

如无意外，你的服务就启起来了。 你可以使用如下命令在检查有没有启动成功：

-  `sudo docker ps` 来查看 gost 是否在运行。
-  `netstat -nolp | grep 443` 来查看 gost 是否在监听 443 端口。
-  `sudo docker logs gost` 来查看 gost 的日志。

#### 2.1.1 使用Cloudflare原生IP

1) 用Docker安装 Cloudflare Warp

用 Docker 可以很方便地部署起一个 Cloudflare WARP Proxy，只需要一行命令:

```shell
docker run -v $HOME/.warp:/var/lib/cloudflare-warp:rw \
  --restart=always --name=cloudflare-warp e7h4n/cloudflare-warp
```

这条命令会在容器上的 40001 开启一个 socks5 代理，接下来查看这个容器的 ip:

```shell
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' cloudflare-warp
```

然后可以通过 curl 镜像来测试，例如，如果容器的 ip 是 `172.17.0.2`，则可以运行:

```shell
docker run --rm curlimages/curl --connect-timeout 2 -x "socks5://172.17.0.2:40001" ipinfo.io
```

返回的结果中 `org` 字段应该能看到 Cloudflare 相关的信息。

2）再建立一个Gost容器，并将其流量转发给Warp

```shell
#!/bin/bash

# 下面的四个参数需要改成你的
DOMAIN="YOUR_DOMAIN"
USER="username"
PASS="password"
PORT=8443

BIND_IP=0.0.0.0
CERT_DIR=/etc/letsencrypt
CERT=${CERT_DIR}/live/${DOMAIN}/fullchain.pem
KEY=${CERT_DIR}/live/${DOMAIN}/privkey.pem
sudo docker run -d --name gost \
    -v ${CERT_DIR}:${CERT_DIR}:ro \
    --net=host gogost/gost \
    -L "http2://${USER}:${PASS}@${BIND_IP}:${PORT}?certFile=${CERT}&keyFile=${KEY}&probeResistance=code:404&knock=www.google.com"
    -F "socks5://172.17.0.2:40001"
```

上面的脚本会建立一个监听8443端口的Gost容器，并将其流量转发给Warp。


### 2.2 配置客户端
参考[https://github.com/xiongpahao/Magical-Proxy](https://github.com/xiongpahao/Magical-Proxy)

## 3. 搭建NextChat+Copilot to GPT4服务

