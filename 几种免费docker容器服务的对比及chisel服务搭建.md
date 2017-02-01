## 几种免费 docker 容器服务的对比 及 chisel 服务搭建

很多人说免费的总是最贵的，我认为这句话并不是完全正确的，免费的服务存在合理的应用场景。免费的服务存在其自身的合理性和商业模式，如吸引客户，为收费版提供预览，作为营销广告等等。

由于 docker 容器服务还处于快速发展阶段，所以免费的服务出现存在其必然性，有几个服务合理利用起来还是很靠谱的。本文介绍几种免费的 docker 服务并提供详细的介绍和对比分析，文中均以搭建 `chisel` 服务为例。

## chisel 服务介绍

[chisel](https://github.com/jpillora/chisel) 是一款优秀的 TCP 转 HTTP 服务，可以方便的作为流量的出口。

如转发 squid 的 `0.0.0.0:3128` 请求或转发 ss 的 `0.0.0.0:1080` 到 `https://www.example.com/chisel` 这样的子路径地址，可以很好的隐藏服务端 TCP 服务，同时可以将 HTTP 转发的流量经过多层的CDN转发来隐藏请求来源。

## 几种 Docker 免费服务概述

本文主要介绍 `bluemix` `heroku` `arukas` `openshift` 提供的 docker 服务。这四个服务都有提供类似 `https://YOUR_APP_NAME.example.com` 这样的地址作为出口，可以直接或通过免费 CDN 或自建 nginx 转发 HTTP 请求到服务的出口地址。

其中 `bluemix` `arukas` 可以使用 docker 容器的 root 用户，而 `heroku` 和 `openshift` 则需要运行在非特权用户下。

从使用的易用性上来讲 `arukas >> openshift > heroku > bluemix`，从服务提供的资源和可靠性上来讲 `bluemix > heroku > arukas >> openshift`。

## Arukas

Arukas 是一家日本的 docker 服务提供商，目前仍在公开测试阶段。登录时可以使用 GitHub 帐号登录，需要预先申请，大概会在一天左右收到账户确认邮件。

官网 [https://arukas.io/](https://arukas.io/)

后台 [https://app.arukas.io/](https://app.arukas.io/)

arukas 的优点是服务器在日本樱花，延时很低，速度也很不错。每个容器可以使用 256M 或 512M 的内存。登录后台后，新建一个应用 `matrix-test`，输入 `Image` 地址 `xdtianyu/docker:matrix`，

入口 `Endpoint` 输入一个你想要的子域名如 `matrix-test`, 环境变量 `ENV` 输入 `PORT` `8080`， 入口命令 `CMD` 输入 `/usr/sbin/enterpoint.sh`，保存后点击 `start` 按钮开启应用。

其中 `Image` 是托管在 [https://hub.docker.com/](https://hub.docker.com/) 的镜像文件，所以可以很方便的部署应用。

![docker1](https://github.com/xdtianyu/Docs/raw/master/art/docker1.png)

应用启动后即可通过入口访问 `https://matrix-test.arukascloud.io/` docker 服务，其中 `https://matrix-test.arukascloud.io/ttyd` (root:root) 可以用来登录 docker 系统 (su - (root))。

通过 `https://matrix-test.arukascloud.io/chisel` chisel 服务可以方便的代理任何主机的 tcp 服务。如在本地客户端运行 `chisel client -v https://matrix-test.arukascloud.io/chisel 3129:3128` 就可以在本地 3129 端口使用服务器的 squid 服务了。

或者使用 `chisel client -v https://matrix-test.arukascloud.io/chisel 1110:YOUR_SERVER_IP:1110` 命令将你服务器的 1110 tcp 端口通过 http 转发到本地 1110 tcp 端口。

`chisel` 可以方便的做 CDN 转发，所以可以很好的作为流量的出口来隐藏客户端 IP。下面是一段自建 nginx cdn 的配置示例

```
    location /your_sub_path {
        proxy_pass https://matrix-test.arukascloud.io/chisel/;
        resolver 8.8.8.8;
        proxy_set_header Host matrix-test.arukascloud.io;
        proxy_ssl_server_name on;
        proxy_buffering off;
        proxy_connect_timeout   10;
        proxy_send_timeout      15;
        proxy_read_timeout      20;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
```

之后就可以使用 `chisel client -v https://YOUR_CDN_DOMAIN/your_sub_path 1110:YOUR_SERVER_IP:1110` 来做转发了。关于 chisel 的使用下文中不再继续说明。关于 `docker/matrix` 的更多服务， 请参考 [docker-auto-builds/matrix](https://github.com/xdtianyu/docker-auto-builds/tree/master/matrix)

Arukas 的 docker 容器具有 root 权限，可以在容器中方便的在线安装软件。使用 `Endpoint` 即使用 `https://matrix-test.arukascloud.io` 访问还是很稳定的，只是目前该服务一些 BUG，如果出现了错误，则可以尝试新建另一个应用。

## Heroku

Heroku 是一个老牌的 PaaS 服务商，具有很高的可靠性。Heroku 每月可以使用一个 512M 内存的应用，如果应用在30分钟内没有被请求则会进入休眠模式而不再占用资源。

例如你可以同时开2个应用并被请求 12 小时，到了晚上没有任何请求都休眠 12 小时也就是一天只占用了 12x2 = 24 小时的资源，仍然时免费的。


官网 [https://www.heroku.com/](https://www.heroku.com/)

后台 [https://dashboard.heroku.com/apps](https://dashboard.heroku.com/apps)

heroku 需要绑定信用卡，关于资费的更多介绍请参考 [https://www.heroku.com/pricing](https://www.heroku.com/pricing)

Heroku 的服务需要使用 `heroku` 命令行工具来管理。参考 [https://devcenter.heroku.com/articles/heroku-cli](https://devcenter.heroku.com/articles/heroku-cli)

```shell
sudo apt-get install software-properties-common # debian only
sudo add-apt-repository "deb https://cli-assets.heroku.com/branches/stable/apt ./"
curl -L https://cli-assets.heroku.com/apt/release.key | sudo apt-key add -
sudo apt-get update
sudo apt-get install heroku
```
使用 `heroku login` 命令登录，使用 `heroku plugins:install heroku-container-registry` 命令安装容器插件。

登录容器

```
heroku container:login
```

部署 matrix 服务

```shell
git clone https://github.com/xdtianyu/docker-auto-builds
cd docker-auto-builds/matrix/

heroku create matrix-test
heroku container:push web --app matrix-test
```

完成后即可在 `https://matrix-test.herokuapp.com/` 访问应用及 chisel 服务，和 Arukas 相同，不再说明。

Heroku 的 docker 服务并没有提供 root 权限，所以应用需要可以以任何 uid 运行。

## Bluemix

Bluemix 是 IBM 提供的 PaaS 服务，具有极高的稳定性并且提供了非常充足的免费资源。

官网 [https://www.bluemix.net](https://www.bluemix.net)

后台 [https://console.ng.bluemix.net](https://console.ng.bluemix.net)

Bluemix 注册后提供一个月的 2G 免费内存，(在 softlayer) 绑定信用卡后可以继续在次月使用 512m 的免费内存。

有3个区域 北美 欧洲 悉尼可以选择，每个区域都有 512M 内存且有两个公网 ip 可以用于绑定独立的 docker 容器，而以 docker group 形式创建的容器则不需要独立的 ip，可以使用 Bluemix 提供的域名来访问。

关于价格的更多内容请参考 [https://www.ibm.com/cloud-computing/bluemix/zh/pricing](https://www.ibm.com/cloud-computing/bluemix/zh/pricing)

Bluemix 的注册和绑定信用卡比较麻烦，需要花一些时间注册 IBMId 和 softlayer，并在 softlayer 绑定信用卡。

**安装 Cloud Foundry CLI 工具**

参考 

[https://console.ng.bluemix.net/docs/containers/container_cli_cfic.html](https://console.ng.bluemix.net/docs/containers/container_cli_cfic.html)

[https://github.com/cloudfoundry/cli/releases](https://github.com/cloudfoundry/cli/releases)

```shell
wget -q -O - https://packages.cloudfoundry.org/debian/cli.cloudfoundry.org.key | sudo apt-key add -
echo "deb http://packages.cloudfoundry.org/debian stable main" | sudo tee /etc/apt/sources.list.d/cloudfoundry-cli.list
sudo apt-get update
sudo apt-get install cf-cli
```
安装容器插件

```shell
cf install-plugin https://static-ice.ng.bluemix.net/ibm-containers-linux_x64
```

登录 Cloud Foundry

```
cf login -a api.ng.bluemix.net
```

登录后台 [https://console.ng.bluemix.net/dashboard/apps](https://console.ng.bluemix.net/dashboard/apps) 并初始化项目工作区

登录容器服务

```
cf ic login
```

上传 docker 镜像并启动应用

```shell
git clone https://github.com/xdtianyu/docker-auto-builds
cd docker-auto-builds/matrix/

cf ic build -t xdtianyu/docker:matrix .
cf ic run --name=matrix-test -m 128 -p 22 -p 80 -p 443 -p 9080 -p 9443 -e PORT=9080 registry.ng.bluemix.net/xdtianyu/docker:matrix
```

进入 [后台](https://console.ng.bluemix.net/dashboard/apps) 查看容器，这时可以通过 创建应用-容器，选择你刚才上传的 docker 镜像来创建一个容器组(即可扩展)，并定义子域名和端口等信息，或者使用如下命令创建

```
cf ic group create --auto -d mybluemix.net -n matrix-test --name matrix-test -m 128 -p 9080 -e PORT=9080 --desired 1 registry.ng.bluemix.net/xdtianyu/docker:matrix
```

关于命令参数的含义请参考 [https://console.ng.bluemix.net/docs/containers/container_cli_cfic.html#container_cli_reference_cfic__group_create](https://console.ng.bluemix.net/docs/containers/container_cli_cfic.html#container_cli_reference_cfic__group_create)

之后就可以通过浏览器打开 `https://matrix-test.mybluemix.net` 来访问应用和 chisel 服务了。

Bluemix 和 Arukas 一样提供了 root 权限，同时还有免费的外部存储空间，除了部署稍麻烦一些，其他方面尤其可以给docker容器绑定2个免费的公网IP，资源供给都应该是最好的了。

## OpenShift 

OpenShift 是由 RedHat 提供的老牌 PaaS 服务，在再次改版之后只能免费使用一个月，之后会删除帐号并清除数据。删除帐号后还可以重新申请，但每次申请都需要等待大概 1-2 天的时间。

官网 [https://www.openshift.com](https://www.openshift.com)

后台 [https://console.preview.openshift.com/console/](https://console.preview.openshift.com/console/)

首先登录后台创建一个项目，之后点击 `Add to project`， 选择 `Deploy Image`， 选择 `Image name`，输入 `xdtianyu/docker:openshift` 点击搜索，输入名称例如 `matrix-test`，点击 `Create` 创建应用。

![docker2](https://github.com/xdtianyu/Docs/raw/master/art/docker2.png)

点击 `Continue to overview` 查看部署进度，等待几分钟部署完成后，点击 `Create Route`，修改 `Target Port` 为 `8080->8080`，选中 `Secure route`，点击 `Create` 创建。

之后即可通过 `https://matrix-test-matrix.44fs.preview.openshiftapps.com/` 访问应用和 chisel 服务了。

OpenShift 没有提供 root 权限，需要应用可以以任意用户 id 运行。

## 总结

Arukas 和 OpenShift 都可以从网页后台快速的创建应用，而 Heroku 和 Bluemix 则需要以来命令行工具作为辅助。而从稳定性上讲 Bluemix 和 Heroku 的服务要优胜一些，要使用哪个服务就请读者自己根据应用的场景来做判断了。
