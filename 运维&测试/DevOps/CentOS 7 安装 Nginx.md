## 1. 前言

Nginx (engine x) 是一个高性能的 HTTP 和反向代理服务器，也是一个 IMAP/POP3/SMTP 服务器。。 本例演示 CentOS 7 下安装和配置 Nginx 的基本步骤。

### 1.1 环境说明

CentOS 7（Minimal Install）

```shell
$ cat /etc/redhat-release 
CentOS Linux release 7.5.1804 (Core) 
```

## 2. 步骤

### 2.1 步骤 1: 添加 yum 源

Nginx 不在默认的 yum 源中，可以使用 epel 或者官网的 yum 源，本例使用官网的 yum 源。

```shell
$ sudo rpm -ivh http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm
```

安装完 yum 源之后，可以查看一下。

```shell
$ sudo yum repolist
Loaded plugins: fastestmirror, langpacks
Loading mirror speeds from cached hostfile
 * base: mirrors.aliyun.com
 * extras: mirrors.aliyun.com
 * updates: mirrors.aliyun.com
repo id                          repo name                          status
base/7/x86_64                    CentOS-7 - Base                    9,911
extras/7/x86_64                  CentOS-7 - Extras                    368
nginx/x86_64                     nginx repo                           108
updates/7/x86_64                 CentOS-7 - Updates                 1,041
repolist: 11,428
```

可以发现 `nginx repo` 已经安装到本机了。

### 2.2 步骤 2: 安装

yum 安装 Nginx，非常简单，一条命令。

```shell
$ sudo yum install -y nginx
```

### 2.3 步骤 3: 配置 Nginx 服务

设置开机启动。

```shell
$ sudo systemctl enable nginx
```

启动服务。

```shell
$ sudo systemctl start nginx
```

停止服务。

```shell
$ sudo systemctl restart nginx
```

重新加载，因为一般重新配置之后，不希望重启服务，这时可以使用重新加载。

```shell
$ sudo systemctl reload nginx
```

### 2.4 步骤 4: 打开防火墙端口

默认 CentOS7 使用的防火墙 firewalld 是关闭 http 服务的（打开 80 端口）。

```sh
$ sudo firewall-cmd --zone=public --permanent --add-service=http
success
$ sudo firewall-cmd --reload
success
```

打开之后，可以查看一下防火墙打开的所有的服务

```shell
$ sudo sudo firewall-cmd --list-service
ssh dhcpv6-client http
```

可以看到，系统已经打开了 http 服务。

### 2.5 反向代理

Nginx 是一个很方便的反向代理，配置反向代理可以参考 [Module ngx_http_proxy_module](http://nginx.org/en/docs/http/ngx_http_proxy_module.html) 。本文不做累述。

需要指出的是 CentOS 7 的 SELinux，使用反向代理需要打开网络访问权限。

```shell
$ sudo setsebool httpd_can_network_connect 1 
```

打开网络权限之后，反向代理可以使用了。

### 2.6 绑定其他端口

Nginx 默认绑定的端口是 http 协议的默认端口，端口号为：`80`，如果需要绑定其他端口，需要注意 SELinux 的配置

例如：绑定 8081 端口，但是会发现无法启动，一般的报错如下

```shell
YYYY/MM/DD hh:mm:ss [emerg] 46123#0: bind() to 0.0.0.0:8081 failed (13: Permission denied)
```

此时需要更改 SELinux 的设置。我们使用 SELinux 的管理工具 `semanage` 进行操作，比较方便。

安装 `semanage` 使用如下命令

```shell
$ sudo yum install policycoreutils-python
```

然后查看是否有其他协议类型使用了此端口

```shell
$ sudo semanage port -l | grep 8081
transproxy_port_t              tcp      8081
```

返回了结果，表明已经被其他类型占用了，类型为 `transproxy_port_t`。

我们还要查看一下 Nginx 的在 SELinux 中的类型 `http_port_t` 绑定的端口

```shell
$ sudo semanage port -l | grep http_port_t
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
```

第一行 `http_port_t` 中没有包含 `8081` 这个端口。因此需要修改 `8081` 端口到 `http_port_t` 类型中。

```shell
$ sudo semanage port -m -p tcp -t http_port_t 8081
```

如果没有其他协议类型使用想要绑定的端口，如 `8001`，则我们只要新增到 SELinux 中即可。

```shell
$ sudo semanage port -l | grep 8001
$ sudo semanage port -a -p tcp -t http_port_t 8001
```

此时，重新启动 Nginx 即可。

### 2.7 域名转发

添加你需要转发的域名，内容如下：

```
server {
    listen 80;
    server_name domain1.com;
    location / {
        proxy_pass http://localhost:10001;
    }
}

server {
    listen 80;
    server_name domain2.com;
    location / {
        proxy_pass http://localhost:10002;
    }
}
```

这个实现的效果就是http://domain1.com转发到http://localhost:10001上，http://domain2.com转发到http://localhost:10002上，不在这两个域名之外的也转发到第一个server上。

这里要注意，如果proxy_pass写的是http://localhost，那么网站的主机头就是localhost，而不是你写的那个server_name。

重新加载，因为一般重新配置之后，不希望重启服务，这时可以使用重新加载。

```shell
$ sudo systemctl reload nginx
```

## 3. 结论

本文演示了 CentOS 7 下 yum 安装 Nginx，配置服务等。

## 4. 参考资料

[Install Nginx Binary Releases](https://www.nginx.com/resources/wiki/start/topics/tutorials/install/)
[Module ngx_http_proxy_module](http://nginx.org/en/docs/http/ngx_http_proxy_module.html)
[Using NGINX and NGINX Plus with SELinux](https://www.nginx.com/blog/using-nginx-plus-with-selinux/)