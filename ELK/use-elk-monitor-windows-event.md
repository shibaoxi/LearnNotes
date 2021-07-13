# 使用Winlogbeat/Metricbeat + Elasticsearch 分析windows 事件日志和性能

## Winlogbeat + Elasticsearch

### Winlogbeat 介绍

使用Winlogbeat将Windows事件日志流传输到Elasticsearch。Winlogbeat 通过标准的 windows API 获取 windows 系统日志，常见的有 Application，Security 、System三个核心日志文件。

### 安装和配置

1. 下载并解压缩Winlogbeat

    ```shell
    #下载地址
    https://www.elastic.co/downloads/beats/winlogbeat
    ```

2. 安装Winlogbeat服务

    解压缩下载的压缩文件并重命名

    <img src="https://i.loli.net/2021/07/12/OGANY5Z31B7sct8.png" width=600 />

    在系统上执行如下命令

    ```powershell
    #进入程序所在的目录
    cd c:\winlogbeat

    powershell.exe -ExecutionPolicy UnRestricted -File .\install-service-winlogbeat.ps1
    ```

3. 配置Winlogbeat

    修改winlogbeat.yml文件，设置如下信息：

    ```shell
    # ---------------------------- Elasticsearch Output ----------------------------
    output.elasticsearch:
    # Array of hosts to connect to.
    hosts: ["192.168.100.123:9200"]

    # Protocol - either `http` (default) or `https`.
    #protocol: "https"

    # Authentication credentials - either API key or username/password.
    #api_key: "id:api_key"
    username: "elastic"
    password: "password"
    ```

    执行如下命令验证配置文件

    ```shell
    .\winlogbeat.exe test config - winlogbeat.yml -e

    #输入如下：
    2021-07-12T13:32:02.895+0800    INFO    instance/beat.go:645    Home path: [C:\winlogbeat] Config path: [C:\winlogbeat] Data path: [C:\winlogbeat\data] Logs path: [C:\winlogbeat\logs]
    2021-07-12T13:32:02.901+0800    INFO    instance/beat.go:653    Beat ID: 3b6fda1e-cdf2-4901-b421-a9fe79a5234c
    2021-07-12T13:32:02.925+0800    INFO    [beat]  instance/beat.go:981    Beat info       {"system_info": {"beat": {"path": {"config": "C:\\winlogbeat", "data": "C:\\winlogbeat\\data", "home": "C:\\winlogbeat", "logs": "C:\\winlogbeat\\logs"}, "type": "winlogbeat", "uuid": "3b6fda1e-cdf2-4901-b421-a9fe79a5234c"}}}
    2021-07-12T13:32:02.927+0800    INFO    [beat]  instance/beat.go:990    Build info      {"system_info": {"build": {"commit": "aacf9ecd9c494aa0908f61fbca82c906b16562a8", "libbeat": "7.10.2", "time": "2021-01-12T23:31:06.000Z", "version": "7.10.2"}}}
    2021-07-12T13:32:02.929+0800    INFO    [beat]  instance/beat.go:993    Go runtime info {"system_info": {"go": {"os":"windows","arch":"amd64","max_procs":2,"version":"go1.14.12"}}}
    2021-07-12T13:32:02.938+0800    INFO    [beat]  instance/beat.go:997    Host info       {"system_info": {"host": {"architecture":"x86_64","boot_time":"2021-06-21T09:38:43.04+08:00","name":"JNZKDC01","ip":["fe80::d40e:a1ec:dff2:fa60/64","192.168.100.90/24","::1/128","127.0.0.1/8"],"kernel_version":"10.0.17763.1757 (WinBuild.160101.0800)","mac":["00:15:5d:de:42:00"],"os":{"family":"windows","platform":"windows","name":"Windows Server 2019 Standard","version":"10.0","major":10,"minor":0,"patch":0,"build":"17763.1757"},"timezone":"CST","timezone_offset_sec":28800,"id":"91edbbc2-da75-4d59-b606-547a0518816d"}}}
    2021-07-12T13:32:02.940+0800    INFO    [beat]  instance/beat.go:1026   Process info    {"system_info": {"process": {"cwd": "C:\\winlogbeat", "exe": "C:\\winlogbeat\\winlogbeat.exe", "name": "winlogbeat.exe", "pid": 3752, "ppid": 5704, "start_time": "2021-07-12T13:32:02.746+0800"}}}
    2021-07-12T13:32:02.941+0800    INFO    instance/beat.go:299    Setup Beat: winlogbeat; Version: 7.10.2
    2021-07-12T13:32:02.943+0800    INFO    [index-management]      idxmgmt/std.go:184      Set output.elasticsearch.index to 'winlogbeat-7.10.2' as ILM is enabled.
    2021-07-12T13:32:02.948+0800    INFO    eslegclient/connection.go:99    elasticsearch url: http://192.168.100.123:9200
    2021-07-12T13:32:02.950+0800    INFO    [publisher]     pipeline/module.go:113  Beat name: JNZKDC01
    2021-07-12T13:32:02.951+0800    INFO    beater/winlogbeat.go:69 State will be read from and persisted to C:\winlogbeat\data\.winlogbeat.yml
    2021-07-12T13:32:03.012+0800    WARN    [cfgwarn]       registered_domain/registered_domain.go:60       BETA: The registered_domain processor is beta.
    2021-07-12T13:32:03.078+0800    WARN    [cfgwarn]       registered_domain/registered_domain.go:60       BETA: The registered_domain processor is beta.
    Config OK
    ```

4. 启动Winlogbeat

    使用以下命令启动Winlogbeat服务

    ```shell
    Start-Service winlogbeat
    ```

    使用以下命令停止Winlogbeat服务

    ```shell
    Stop-Service winlogbeat
    ```

### 在Kibana中查看分析报告

在Kibana中Security下面查看相关分析报告

<img src="https://i.loli.net/2021/07/12/SNof9Tcxg5JQMw2.png" width=600 />

## metricbeat + Elasticsearch

### 安装metricbeat服务

下载文件

```shell
#下载地址
https://www.elastic.co/downloads/beats/metricbeat
```

解压文件，并存放到目录下面

<img src="https://i.loli.net/2021/07/12/VU1F7knBQcISsLH.png" width=600 />

安装服务

```shell
powershell.exe -ExecutionPolicy UnRestricted -File .\install-service-metricbeat.ps1
```

### 配置metricbeat

打开metricbeat.yml文件

```shell
# ---------------------------- Elasticsearch Output ----------------------------
output.elasticsearch:
# Array of hosts to connect to.
hosts: ["192.168.100.123:9200"]

# Protocol - either `http` (default) or `https`.
#protocol: "https"

# Authentication credentials - either API key or username/password.
#api_key: "id:api_key"
username: "elastic"
password: "changeme"
```

### 启动metricbeat 服务

```shell
Start-Service metricbeat
```

### 登录kibana查看数据

<img src="https://i.loli.net/2021/07/13/eM71LWRVuagN6iX.png" width=600 />
