# 优化技巧

## Linux下的优化

==重要的事情：在docker/k8s环境下，基本上不需要考虑优化的问题，因为几乎任何优化设置都是无效并且可能具有反效果。实测不采用负载均衡的情况下，单一节点的k8s的性能并不如同样配置的vps。哪怕用openvz作为生产环境，我个人都不会推荐使用docker作为生产环境==

### systemd + Lighttpd + sock(Unix Socket)

>   注意：circus并不推荐在FreeBSD环境下使用，可能会发生circus的假死问题。因此不推荐直接使用。

Lighttpd用于处理所有的静态文件，而动态的部分通过转发模组转发至PyMIO处理（小贴士：Lighttpd亦可作为负载均衡使用）。从而实现静态文件和动态内容的物理性隔离。此笔记仅适配单机单站点运行，如需要负载均衡，可在此配置的基础上进行调整。

根据实际需求，可以自行选择使用`circus`或是直接由python执行。例如只使用web服务和定时器时，可以考虑直接使用python直接执行的方式搭配`cron`以节省服务器资源开销，如果是小机，强烈建议直接使用。

操作系统则推荐Ubuntu/Debian/ArchLinux，非常不推荐CentOS，如需使用CentOS，建议切换至RockyLinux。

**警告：这个优化不适用于docker/k8s！！！在docker/k8s下反而性能非常低下！！！**

#### 技术选型

正如之前所述，根据自己的实际情况来选择是否使用circus

#### 配置文件

##### circus.ini

**注意：stderr_stream.filename里如果指定为目录中的某个文件，则此目录必须存在。**

`<project_name>`请自行替换为实际的项目名称

需要事先使用`root`权限建立对应的目录后，使用`chown`移交至运行pymio项目的账户下。如果使用安装脚本，可忽略这步。

**警告：不建议直接使用root权限运行pymio项目！**

```ini
[circus]
endpoint = tcp://127.0.0.1:5555
pubsub_endpoint = tcp://127.0.0.1:5556
stream_backend = gevent
debug = False
loglevel = info

[watcher:webapp]
cmd = python
args = -u mio/pymio.pyc --ds=/run/<project_name>/pymio.sock
stop_children = True
copy_env = True
copy_path = True
singleton = True
stop_signal = SIGINT
stderr_stream.class = FileStream
stderr_stream.filename = logs/www.log
stderr_stream.max_bytes = 10485760
stderr_stream.backup_count = 2
graceful_timeout = 10

[env]
MIO_CONFIG = production
MIO_PORT = 5000
PYTHONIOENCODING = utf-8

[env:webapp]
MIO_LIMIT_CPU = 0
```

##### service文件

备注：可以使用`install_service.sh`脚本快速配置。

此文件为线上实际运行的文件，使用的是pyenv编译的python版本。如不使用pyenv，请自行调整配置。

路径默认位于`/usr/local/www/`下，请确保路径一致。如路径不一致，请自行调整

`<project_name>`请自行替换为实际的项目名称

`<project_runner>`请自行替换为运行此项目的用户名。推荐使用非`root`账户运行以提高项目安全性。

下同

```ini
[Unit]
Description=<project_name>
After=network.target

[Service]
User=<project_runner>
LimitNPROC=infinity
LimitNOFILE=infinity
LimitFSIZE=infinity
LimitCPU=infinity
LimitAS=infinity
ExecStart=/usr/local/www/<project_runner>/<project_name>/run_circusd.sh
ExecStop=/usr/local/.pyenv/shims/circusctl quit

[Install]
WantedBy=multi-user.target
```

###### install_service.sh 的使用方法

**注意：此安装脚本搭配的是pyenv编译的python版本，如不使用pyenv，请自行调整**。

```shell
bash installer/install_service.sh <project_runner> <project_name>
```

