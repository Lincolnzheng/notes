# 4G 联网、HTTP 对时与 MQTT 连接

以 HN_COVER 工程为例。STM32 通过 USART2 向 4G 模块发送 AT 指令，模块负责接入运营商网络、发 HTTP 请求和收发 MQTT 报文。

```text
模块上电 → SIM 就绪 → 网络注册 → 激活 PDP 获取 IP
  → HTTP GET 获取平台时间 → STM32 写 RTC、计算登录密码
  → 配置 MQTT → 打开服务器连接 → MQTT 登录 → 订阅主题
```

`TX` 表示 STM32 发送，`RX` 表示模块返回。下面是按源码编排的演示日志。每行 `#` 后的文字是注释，串口只发送 `TX` 行中注释前的内容。IP、账号、设备编号和时间均为示例。手动串口联调时，AT 指令逐条发送，工程使用 `\r\n` 结束指令行。

## 1. 4G 联网

STM32 通过 `ENLTE` 控制模块供电，以 `115200` 波特率发送 `AT` 探测。源码在进入网络初始化后会再发一次 `AT`。

```text
TX  AT  # 上电后探测模块是否能通过串口通信
RX  OK  # 模块响应正常，进入网络初始化
TX  AT  # 网络初始化开始时再次确认模块在线
RX  OK  # 模块响应正常，可以关闭回显
TX  ATE0  # 关闭 AT 命令回显
RX  OK  # 回显设置成功，继续检查 SIM 卡
TX  AT+CPIN?  # 查询 SIM 卡状态
RX  +CPIN: READY  # SIM 卡已就绪，源码据此继续
RX  OK  # SIM 卡查询命令结束
TX  AT+CGREG?  # 查询分组数据网络注册状态
RX  +CGREG: 0,1  # 已在本地网络注册，末位为 5 时表示漫游注册
RX  OK  # 网络注册查询命令结束
TX  AT+QMTCLOSE=0  # 关闭 MQTT 客户端 0 的旧网络连接
RX  ERROR  # 此处源码接受 ERROR，继续清理普通套接字
TX  AT+QICLOSE=0  # 关闭普通 TCP/IP 套接字 0
RX  ERROR  # 此处源码接受 ERROR，继续设置 APN
TX  AT+QICSGP=1,1,"CMNET","","",0  # 设置 PDP 上下文 1，使用 IPv4 和 CMNET APN，关闭 APN 认证
RX  OK  # APN 参数设置成功
TX  AT+QIDEACT=1  # 停用编号 1 的旧 PDP 上下文
RX  OK  # 停用命令完成，可以重新激活
TX  AT+QIACT=1  # 激活 PDP 上下文 1，建立 4G 数据连接
RX  OK  # 激活成功，可以查询分配的 IP
TX  AT+QIACT?  # 第一次查询已激活的 PDP 上下文
RX  +QIACT: 1,1,1,"10.64.12.34"  # 上下文 1 已激活，模块的 IPv4 地址为 10.64.12.34
RX  OK  # 第一次查询结束
TX  AT+QIACT?  # 第二次查询，源码会从回包中检查 IP
RX  +QIACT: 1,1,1,"10.64.12.34"  # 再次返回运营商分配给模块的 IP
RX  OK  # 第二次查询结束，IP 校验通过后进入 HTTP 对时
```

- `ATE0` 关闭回显。`CPIN?` 等 `READY`。`CGREG?` 查询注册状态，源码接受 `+CGREG: 0,1`（本地注册）或 `+CGREG: 0,5`（漫游注册）。
- `QMTCLOSE=0` 清理 MQTT 客户端 0 的旧连接，`QICLOSE=0` 清理普通套接字 0。当前工程对这两条指令以及 `QIDEACT=1` 接受 `OK` 或 `ERROR`。
- `QICSGP=1,1,"CMNET","","",0` 设置 PDP 上下文 1，使用 IPv4 和默认 APN `CMNET`。两个空字符串是 APN 用户名和密码，末尾 `0` 表示无 APN 认证。配置了专用 APN 时，源码使用保存的 APN 参数。
- `QIDEACT=1` 停用旧的 PDP 上下文，`QIACT=1` 激活它，`QIACT?` 查询分配结果。源码查询两次，第二次还校验 IP。`10.64.12.34` 是模块获分配的 IP，MQTT 服务器 IP 在 `QMTOPEN` 里另填。

源码入口为 `User/app/mqtt.c` 的 `wait_start_up()`、`network_init()` 和 `send_QICSGP()`。

## 2. HTTP GET 和对时

4G 数据连接就绪后，使用 `QHTTPURL` 设置 URL，`QHTTPGET` 发请求，`QHTTPREAD` 读取响应。示例 URL `http://192.0.2.10/time` 长 22 字节。

```text
TX  AT+QHTTPURL=22  # 告知模块接下来要输入 22 字节的 URL
RX  CONNECT  # 模块已准备接收 URL
TX  http://192.0.2.10/time  # 发送对时 URL 本身，共 22 字节
RX  OK  # 模块已接收 URL，可以发起 GET

TX  AT+QHTTPGET=10  # 发起 HTTP GET，等待结果的参数为 10 秒
RX  OK  # 模块接受命令，继续等待 HTTP 结果
RX  +QHTTPGET: 0,200,18  # 模块执行成功，HTTP 状态为 200，正文长 18 字节

TX  AT+QHTTPREAD=10  # 读取 HTTP 响应，接收数据包之间最多等待 10 秒
RX  CONNECT  # 模块开始输出响应内容
RX  2026-11-1214:30:00  # 平台返回的时间正文，STM32 从中解析日期和时间
RX  OK  # HTTP 响应输出结束，继续等读取结果
RX  +QHTTPREAD: 0  # 读取成功，STM32 解析时间并写入 RTC
```

- `QHTTPURL=<字节数>` 返回 `CONNECT` 后输入 URL。长度只计算 URL 本身。
- `QHTTPGET=10` 的 `10` 是等待 GET 结果的超时秒数。`+QHTTPGET: 0,200,18` 里的 `0` 是模块执行成功，`200` 是 HTTP 状态码，`18` 是示例正文的字节数。源码等到 `+QHTTPGET: 0,200` 才读正文。
- `QHTTPREAD=10` 将响应输出到串口，`10` 是接收数据包之间的最大等待秒数。`CONNECT` 后是响应内容，源码等 `+QHTTPREAD: 0` 后解析时间并写入 RTC。

HTTP 对时是本项目在 MQTT 登录前的业务步骤。STM32 取平台时间的 `yyyyMMddHH` 作 HMAC 密钥，以平台分配的 MQTT 原始密码作消息，计算 HMAC-SHA256，再编码为 64 个小写十六进制字符。示例密钥是 `2026111214`，原始密码是 `DemoPassword`，计算结果见下一节。

通用 HTTP GET 流程可以复用来访问其他 URL。当前 OTA 下载代码使用 `QHTTPURL` 设置固件地址，`QHTTPGETEX=<超时>,<偏移>,<长度>` 请求文件片段，再用 `QHTTPREAD` 读取固件字节。源码入口为 `http_get_time()` 和 `User/app/upgrade.c` 的 `http_file_analysis()`。

## 3. MQTT 连接、登录和订阅

示例 MQTT 地址为 `198.51.100.20:1883`，客户端 ID 为 `demo-client`，用户名为 `demo-user`，设备编号为 `90010000000000001`。实际值由平台提供。

```text
TX  AT+QMTCFG="keepalive",0,180  # 给 MQTT 客户端 0 设置 180 秒保活
RX  OK  # 保活设置成功
TX  AT+QMTCFG="recv/mode",0,0,1  # 设置客户端 0 直接输出接收消息，并带上正文长度
RX  OK  # 接收模式设置成功，可以打开 MQTT 网络连接

TX  AT+QMTOPEN=0,"198.51.100.20",1883  # 用客户端 0 连接指定服务器 IP 和端口
RX  OK  # 模块接受命令，继续等待连接结果
RX  +QMTOPEN: 0,0  # 网络连接已打开，可以发送 MQTT 登录命令

TX  AT+QMTCONN=0,"demo-client","demo-user","71ba6280b824aa858c1e1cd9b0aa7ac827744a2f6cf067c37bf5d510fdc81541"  # 发送客户端 ID、用户名和对时后计算的 HMAC 密码
RX  OK  # 模块接受命令，继续等待服务端登录结果
RX  +QMTCONN: 0,0,0  # 服务端接受登录，可以开始订阅

TX  AT+QMTSUB=0,1,"/v1/devices/90010000000000001/cloud/app/rsp",0  # 订阅 OTA 平台响应主题，请求 QoS 0
RX  OK  # 模块接受命令，继续等待订阅结果
RX  +QMTSUB: 0,1,0,0  # 第一条订阅成功，获准 QoS 0
TX  AT+QMTSUB=0,1,"/v1/devices/90010000000000001/cloud/parameter/rsp",0  # 订阅参数请求的平台响应主题，请求 QoS 0
RX  OK  # 模块接受命令，继续等待订阅结果
RX  +QMTSUB: 0,1,0,0  # 第二条订阅成功，MQTT 初始化完成
```

- `QMTCFG="keepalive",0,180` 设置 MQTT 客户端 0 的保活时间。当前源码配置为 180 秒，项目的 V0.16 平台协议要求 20 秒。
- `QMTCFG="recv/mode",0,0,1` 设置客户端 0，接收模式 `0` 表示通过 `+QMTRECV` 直接带出消息，末尾 `1` 表示同时带出正文长度。
- `QMTOPEN=<客户端索引>,<服务器地址>,<端口>` 打开网络连接。先回 `OK`，再等 `+QMTOPEN: 0,0` 确认连接已打开。
- `QMTCONN=<客户端索引>,<clientId>,<用户名>,<密码>` 登录 MQTT。先回 `OK`，再等 `+QMTCONN: 0,0,0` 确认登录成功。这里填对时后计算出的密码。
- `QMTSUB=<客户端索引>,<报文编号>,<主题>,<QoS>` 订阅主题。当前源码依次发送两条订阅命令，每条都等 `+QMTSUB: 0,1,0,0`。这两个主题分别接收平台的升级与参数响应。

两个主题订阅成功后，`mqtt_conn()` 才返回成功。`Core/Src/main.c` 随后调用 `gateway_upgrade()`。发送业务 JSON 时用 `AT+QMTPUBEX`，等 `>` 后输入指定字节数的正文，收到的订阅消息由模块通过 `+QMTRECV` 主动通知。

指令顺序和判定条件来自 HN_COVER 源码。以上是串口演示，实物的支持情况与回包格式以模块固件和实测日志为准。
