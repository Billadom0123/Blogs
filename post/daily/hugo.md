+++
date = '2026-08-26T13:25:31+08:00'
draft = false
title = '安装Hugo'
+++

# 安装Hugo全过程

考虑到Wordpress对于markdown的支持不是很好，且当前服务器已经千疮百孔，因此将服务器重装，并将Hugo作为服务器中的第一个自建服务。

本文为安装Hugo的一些重要过程。

## 安装Snap

根据官方doc，安装Hugo推荐使用Snap管理器，拥有易于安装，易于升级降级等诸多便利。而当前服务器安装的镜像为CentOS，CentOS的官方仓库并未收录Snap。  
因此在尝试使用 `yum install snap` 时，出现了如下报错：

``` shell
No match for argument: snap
Error: Unable to find a match: snap
```

为此，必须要先开启 `epel` ，即 `Extra Packages for Enterprise Linux`:
``` shell
yum install epel-release
```

在安装了epel后，再次安装snap即可成功安装.此外，根据教程，安装完成后还需要开启snapd套接字和符号链接（方便快速访问snap安装的软件或包）：
``` shell
sudo systemctl enable --now snapd.socket
sudo ln -s /var/lib/snapd/snap /snap
```

## 安装Hugo

接下来便是安装Hugo了，命令也十分简单：
``` shell
snap install hugo
```
其中默认安装的是hugo的 `extended` 版本，且会使用 `strict` 模式  

Snap的隔离模式主要有三种：`strict`, `classic`, `devmode`. 作为使用者一般只需要考虑strict和classic  
其中 `strict` 为严格模式，默认只允许程序访问`/home/<username>`的目录，而其它目录的允许访问则需要通过**接口**允许访问权限。  
而 `classic` 则宽松的多，允许程序访问全路径，没有任何限制。但由于其权限过高，因此在安装时需要使用 `--classic` 参数，且包本身应当通过审核（此外还需要包本身支持classic）。  

在安装完成后，即可按照Hugo官方所给的教程执行命令了：
``` shell
hugo new site quickstart
cd quickstart
git init
git submodule add https://github.com/theNewDynamic/gohugo-theme-ananke.git themes/ananke
echo "theme = 'ananke'" >> hugo.toml
hugo server
```

但是当执行 `hugo new site` 的时候，产生了如下错误：
``` shell
-bash: hugo: command not found
```

在 `which hugo` 查找无果和确认已经安装后，在跟ai的聊天中发现，snap安装后是需要自己将其加入环境变量的

因此需要修改 `~/.bashrc` ：
``` shell
echo 'export PATH=$PATH:/snap/bin' >> ~/.bashrc
source ~/.bashrc  # 立即刷新bashrc
```
在此之后，便可以正常的进行new site了

但是在执行完`hugo server`的时候，发现尽管端口已经开放，但是仍无法在公网访问该服务，在一番询问后得知：`hugo server` 默认绑定的地址为127.0.0.1，不会映射到公网，需要：
``` shell
hugo server --bind=0.0.0.0
```

在配置好后，访问公网，此时已经可以看到内容，但是是“Page Not Found”  
在经过排查后得出，可能是hugo本身的权限问题：我将hugo的目录放在了非用户目录下，因此使用默认 `strict` 模式的 hugo 自然无法进行正常生成资源  

但是当使用了 `--classic` 重新安装后，还是无法正常访问。  

在跟ai聊后，它再次提醒了我：一个包是否能用classic，还得看它本身是否支持`classic`，使用`snap list`后可以在`Notes`里查看，如果是classic一般会有专门的标记。  

查看后发现并没有，因此只能用严格模式放在用户目录或snap根目录下了（当然事实上使用挂载也可以，但这会比较复杂，且不易于管理，故作罢）  

但是依旧可以创建一个软链接，方便之后快速访问。

## nginx

本以为这些全部完成后就可以完整体验了，但是依旧无法访问。
那么问题便来到了nginx上。
尽管已经做好了相关的配置文件：
``` nginx
server {
    listen 80;
    listen [::]:80;
    server_name md.billadom.top;

    # return 301 https://$server_name$request_uri;

    location / {
        proxy_pass http://127.0.0.1:1313;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

```

但是当实际应用时，却依旧无法正常重定向，展示404的画面。
在排查后最后发现，默认安装的nginx中，`nginx.conf`默认接管了所有的`server_name`：
``` nginx
    server {
        listen       80;
        listen       [::]:80;
        server_name  _;
        root         /usr/share/nginx/html;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        error_page 404 /404.html;
        location = /404.html {
        }

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
        }
    }
```
因此尽管新的配置已被接入，却依旧被default的配置接管了。

将这一部分注释掉后再用`nginx -s reload`重启nginx即可正常开启网页
