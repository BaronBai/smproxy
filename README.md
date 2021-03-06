# SMProxy
## swoole msyql proxy 一个基于mysql协议，swoole 开发的mysql数据库连接池
## 特性
- 支持读写分离
- 支持数据库连接池，能够有效解决PHP带来的数据库连接瓶颈
- 支持SQL92标准
- 遵守Mysql原生协议，跨语言，跨平台的通用中间件代理。
- 支持mysql事物
- 采用协程调度
- 支持 HandshakeV10 协议版本
- 完美兼容mysql5.6-5.7
## 设计初衷
php没有连接池，所以高并发时数据库会出现连接打满的情况，mycat等数据库中间件会出现部分sql无法使用，例如不支持批量添加等，而且过于臃肿。所以就自己编写了这个仅支持连接池和读写分离的轻量级中间件，使用swoole协程调度HandshakeV10协议转发使程序更加稳定不用像mycat一样解析所有sql包体，增加复杂度。
## 环境
* swoole 2.1+
* php 7.0+
## 安装
下载的文件直接解压即可。
## 运行
- bin/server start   : 运行服务
- bin/server stop    : 停止服务
- bin/server restart : 重启服务
- bin/server status  : 查询服务运行状态
- bin/server reload  : 平滑重启
- bin/server -h      : 帮助
## SMProxy连接测试
测试SMProxy与测试mysql完全一致，mysql怎么连接，SMProxy就怎么连接。

推荐先采用命令行测试：

mysql -uroot -p123456 -P3366 -h127.0.0.1

也可采用工具连接。


## 配置文件:
```
ROOT 当前SMProxy跟目录
```
### database.json
```Json
{
  "database": {
    "account": {
      "root": {
        "user": "root", 
        "password": "123456"
      }
    },
    "serverInfo": {
      "server1": {
        "write": {
          "host": "127.0.0.1",
          "port": 3306,
          "timeout": 0.5,
          "flag": 0,
          "account": "root"
        },
        "read": {
          "host": "127.0.0.1",
          "port": 3306,
          "timeout": 0.5,
          "flag": 0,
          "account": "root"
        }
      }
    },
    "databases": {
      "db1": {
        "serverInfo": "server1",
        "maxSpareConns": 10,
        "maxConns": 20,
        "charset": "utf-8"
      }
    }
  }
}
```
| account 账号信息 | serverInfo 服务信息 | databases 数据库连接池信息 |
| ------ | ------ | ------ |
| account.root 用户标识 与 serverInfo...account.root 对应 | serverInfo.server1 服务标识 与  databases..serverInfo 对应 | databases.db1 数据库名称 |
| account..user 用户名  | serverInfo..write 读写分离 write 写库 read 读库 | databases..serverInfo 服务信息 |
| account..password 密码  | serverInfo..host 数据库连接地址 | databases..maxSpareConns 最大空闲连接数 |
|   | serverInfo..prot 数据库端口 | databases..maxConns 最大连接数 |
|   | serverInfo..timeout 数据库超时时长(秒) | databases..charset 数据库编码格式 |
|   | serverInfo..flag TCP类型目前支持0阻塞 不支持1.非阻塞 |  |
|   | serverInfo..account  与 databases.account 对应|  |

### server.json
```Json
{
  "server": {
    "user":"root",
    "password":"123456",
    "charset":"utf8mb4",
    "host": "0.0.0.0",
    "port": "3366",
    "mode": 3,
    "sock_type": 1,
    "logs": {
      "open":true,
      "config": {
        "system": {
          "log_path": "/var/www/swoole/swoole-mysql-proxy/logs",
          "log_file": "system.log",
          "format": "Y/m/d"
        },
        "mysql": {
          "log_path": "/var/www/swoole/swoole-mysql-proxy/logs",
          "log_file": "mysql.log",
          "format": "Y/m/d"
        }
      }
    },
    "swoole": {
      "worker_num": 2,
      "max_coro_num": 16000,
      "open_tcp_nodelay": true,
      "daemonize": 0,
      "heartbeat_check_interval": 60,
      "heartbeat_idle_time": 600,
      "reload_async": true,
      "log_file": "/var/www/swoole/swoole-mysql-proxy/logs/error.log",
      "pid_file": "/var/www/swoole/swoole-mysql-proxy/logs/pid/server.pid"
    },
    "swoole_client_setting": {
      "package_max_length": 16777216
    },
    "swoole_client_sock_setting": {
      "sock_type": 1,
      "sync_type": 1
    }
  }
}
```
| user 服务用户名 | password 服务密码 | charset 服务编码 | host 链接地址 | port 服务端口 多个以,隔开 |  [mode](https://wiki.swoole.com/wiki/page/277.html) | sock_type 1 tcp | logs 日志配置 | swoole swoole配置 | swoole_client_setting 客户端配置 | swoole_client_sock_setting 客户端sock配置 |
| ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ |
|   |   |   |   |   |   |   | logs.open 日志开关  |  worker_num work进程数量 | package_max_length 最大包长  | sock_type 1.tcp  |
|   |   |   |   |   |   |   | logs.config 日志配置项 |  max_coro_num 最大携程数  |   | sync_type 1.异步  |
|   |   |   |   |   |   |   | logs.system or mysql 配置模块  |  open_tcp_nodelay 关闭Nagle合并算法  |   |   |
|   |   |   |   |   |   |   | logs..log_path 日志目录 |  daemonize 守护进程化 |   |   |
|   |   |   |   |   |   |   | logs..log_file 日志文件名 |  heartbeat_check_interval 心跳检测 |   |   |
|   |   |   |   |   |   |   | logs..format 日志日期格式 |  heartbeat_idle_time 最大空闲时间 |   |   |
|   |   |   |   |   |   |   |   |  reload_async 异步重启 |   |   |
|   |   |   |   |   |   |   |   |  log_file 日志目录 |   |   |
|   |   |   |   |   |   |   |   |  pid_file 主进程pid目录 |   |   |

## 其他学习资料
- mysql协议分析 ：https://www.cnblogs.com/davygeek/p/5647175.html
- mysql官方协议文档 ：https://dev.mysql.com/doc/internals/en/connection-phase-packets.html#packet-Protocol::Handshake
- mycat源码 ：https://github.com/MyCATApache/Mycat-Server
