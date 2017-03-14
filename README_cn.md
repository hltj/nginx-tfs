[English](README.md)

名称
====

* nginx-tfs

描述
====

* 本项目是从已停止独立开发并集成到 **[alibaba/tengine](https://github.com/alibaba/tengine)** 的 **alibaba/nginx-tfs** 项目 fork 而来。
* 相对 **alibaba/nginx-tfs** 的改动：
  * **alibaba/tengine** 的 TFS 模块的所有改动都已合并进来，并保留了原始作者及提交时间。
  * 支持 OpenResty / 官方 Nginx 1.0.x-1.11.x 以及 Tengine 1.2.1-2.2.0。
* 这个模块实现了TFS的客户端，为 TFS 提供了 [RESTful API](TFS_RESTful_API_cn.md)。TFS 的全称是 Taobao File System，是淘宝开源的一个分布式文件系统。

编译安装
=======

1. TFS模块使用了一个开源的JSON库来支持JSON，请先安装[yajl](http://lloyd.github.com/yajl/)-2.0.1 或其新版本。

2. 下载 [Nginx](http://www.nginx.org/) 或 [Tengine](http://tengine.taobao.org/)。

3. ./configure --add-module=/path/to/nginx-tfs

4. make && make install

配置示例
====

    http {
        #tfs_upstream tfs_rc {
        #    server 127.0.0.1:6100;
        #    type rcs;
        #    rcs_zone name=tfs1 size=128M;
        #    rcs_interface eth0;
        #    rcs_heartbeat lock_file=/logs/lk.file interval=10s;
        #}

        tfs_upstream tfs_ns {
            server 127.0.0.1:8100;
            type ns;
        }

        server {
              listen       7500;
              server_name  localhost;

              tfs_keepalive max_cached=100 bucket_count=10;
              tfs_log "pipe:/usr/sbin/cronolog -p 30min /path/to/nginx/logs/cronolog/%Y/%m/%Y-%m-%d-%H-%M-tfs_access.log";

              location / {
                  tfs_pass tfs://tfs_ns;
              }
        }
    }

指令
====

tfs\_upstream
----------------

**Syntax**： *tfs\_upstream name {...}*

**Default**： *none*

**Context**： *http*

配置 TFS 模块的 server 信息,这个块包括上面几个命令。

例如：

    tfs_upstream tfs_rc {
        server 127.0.0.1:6100;
        type rcs;
        rcs_zone name=tfs1 size=128M;
        rcs_interface eth0;
        rcs_heartbeat lock_file=/logs/lk.file interval=10s;
    }

    tfs_upstream tfs_ns {
        server 127.0.0.1:8100;
        type ns;
    }

server
------------

**Syntax**： *server address*

**Default**： *none*

**Context**： *tfs_upstream*

指定后端 TFS 服务器的地址，当指令 <i>type</i> 为 <i>rcs</i> 时为 RcServer 的地址，如果为 <i>ns</i> 时为 NameServer 的地址。此指令必须配置。

例如:

	server 10.0.0.1:8108;

type
----------------

**Syntax**： *type [ns | rcs]*

**Default**： *ns*

**Context**： *tfs_upstream*

设置 server 类型，类型只能为 ns 或者 rcs，如果为 ns，则指令 <i>server</i> 指定的地址为 NameServer 的地址，如果为 rcs，则为 RcServer 的地址，默认为 ns。

如需使用自定义文件名功能请设置类型为 rcs，使用自定义文件名功能需额外配置 MetaServer 和 RootServer。

rcs\_zone
--------------

**Syntax**： *rcs_zone name=n size=num*

**Default**： *none*

**Context**： *tfs_upstream*

配置 TFS 应用在 RcServer 的配置信息。若开启 RcServer（配置了 <i>type rcs</i>），则必须配置此指令。配置此指令会在共享内存中缓存 TFS 应用在 RcServer 的配置信息，并可以通过指令 <i>rcs_heartbeat</i> 来和 RcServer 进行 keepalive 以保证应用的配置信息的及时更新。

例如：

	rcs_zone name=tfs1 size=128M;

rcs\_heartbeat
--------------

**Syntax**： *rcs_heartbeat lock_file=/path/to/file interval=time*

**Default**： *none*

**Context**： *tfs_upstream*

配置 TFS 应用和 RcServer 的 keepalive，应用可通过此功能来和 RcServer 定期交互，以及时更新其配置信息。若开启 RcServer 功能（配置了 <i>type rcs</i>），则必须配置此指令。

必须配置以下参数：

lock_file=<i>/path/to/file</i>
    用于创建互斥量来确保同时只有一个 Nginx worker 保持心跳。
interval=<i>时间</i>
    设置心跳间隔。

例如：

	rcs_heartbeat lock_file=/path/to/nginx/logs/lk.file interval=10s;

rcs\_interface
----------------

**Syntax**： *rcs\_interface interface*

**Default**： *none*

**Context**： *tfs_upstream*

配置 TFS 模块使用的网卡。若开启 RcServer 功能（配置了 <i>type rcs</i>），则必须配置此指令。

例如：

	rcs_interface eth0;

tfs_pass
--------

**Syntax**： *tfs_pass name*

**Default**： *none*

**Context**： *location*

是否打开 TFS 模块功能，此指令为关键指令，决定请求是否由 TFS 模块处理，必须配置。

需要注意，这里不支持直接写 ip 地址或者域名，这里只支持指令 <i>tfs_upstream name {...} </i> 配置的 upstream，并且必须以 tfs:// 开头。

例如：


	tfs_upstream tfs_rc {
    };

	location / {
		tfs_pass tfs://tfs_rc;
	}

tfs_keepalive
-------------

**Syntax**： *tfs_keepalive max_cached=num bucket_count=num*

**Default**： *none*

**Context**： *http, server*

配置 TFS 模块使用的连接池的大小，TFS 模块的连接池会缓存 TFS 模块和后端 TFS 服务器的连接。此指令必须配置。

可以把这个连接池看作由多个 hash 桶构成的 hash 表。

必须配置以下参数：

max_cached=<i>num</i>
    设置一个哈希桶的容量。
bucket_count=<i>num</i>
    设置桶的个数。

注意：应该根据机器的内存情况来合理配置连接池的大小。

例如：

	tfs_keepalive max_cached=100 bucket_count=15;

tfs\_block\_cache\_zone
-----------------------

**Syntax**： *tfs_block_cache_zone size=num*

**Default**： *none*

**Context**： *http*

配置 TFS 模块的本地 BlockCache。配置此指令会在共享内存中缓存 TFS 中的 Block 和 DataServer 的映射关系。

注意：应根据机器的内存情况来合理配置 BlockCache大小。

例如：

	tfs_block_cache_zone size=256M;

tfs\_log
----------------

**Syntax**： *tfs_log path*

**Default**： *none*

**Context**： *http, server*

配置 TFS 访问记录。

配置此指令会以固定格式将访问 TFS 的请求记录到指定日志文件中，以便进行分析。具体格式参见代码。

例如：

	tfs_log "pipe:/usr/sbin/cronolog -p 30min /path/to/nginx/logs/cronolog/%Y/%m/%Y-%m-%d-%H-%M-tfs_access.log";

注意：cronolog 支持依赖于 tengine 提供的扩展的日志模块。

tfs\_body\_buffer\_size
-----------------------

**Syntax**： *tfs_body_buffer_size size*

**Default**： *2m*

**Context**： *http, server, location*

配置用于和后端 TFS 服务器交互时使用的的 body_buffer 的大小。默认为 2m。

例如：

	tfs_body_buffer_size 2m;

tfs\_connect\_timeout
---------------------

**Syntax**： *tfs_connect_timeout time*

**Default**： *3s*

**Context**： *http, server, location*

配置连接后端 TFS 服务器的超时时间。

tfs\_send\_timeout
------------------

**Syntax**： *tfs_send_timeout time*

**Default**： *3s*

**Context**： *http, server, location*

配置向后端TFS服务器发送数据的超时时间。

tfs\_read\_timeout
------------------

**Syntax**： *tfs_read_timeout time*

**Default**： *3s*

**Context**： *http, server, location*

配置从后端TFS服务器接收数据的超时时间。

其他
----
能支持上传文件大小决定于 <i>client_max_body_size</i> 指令配置的大小。
