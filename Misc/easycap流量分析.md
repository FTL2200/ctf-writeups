# CTF WriteUp — MISC 流量分析「easycap」

> 平台：CTF²（ctf2.dascft.com）
> 分类：MISC / 流量分析
> 附件：`easycap.pcap`
> 附件大小：6802 字节
> 最终 Flag：`flag{385b87afc8671dee07550290d16a8071}`

---

## 一、题目信息

| 项目 | 内容 |
| --- | --- |
| 平台 | CTF²（ctf2.dascft.com） |
| 分类 | MISC / 流量分析 |
| 附件 | `easycap.pcap` |
| 文件大小 | 6802 字节 |
| 总包数 | 82 个 |
| 通信总量 | 5466 字节 |
| 提交格式 | `flag{...}` |

---

## 二、实验环境与工具

| 工具 | 版本 / 路径 | 用途 |
| --- | --- | --- |
| Wireshark | 任意版本 | 图形化流量分析、协议分级统计 |
| tshark | `C:\Program Files\Wireshark\tshark.exe` | 命令行字段批量提取 |
| PowerShell | 5.1 | 执行命令、十六进制转 ASCII |

---

## 三、解题过程

### 3.1 阶段一：协议分层统计

**操作**

Wireshark 打开 `easycap.pcap`，执行：

```
统计（Statistics） → 协议分级（Protocol Hierarchy）
```

**结果**

| 协议 | 分组百分比 | 分组数 | 字节百分比 | 字节数 |
| --- | --- | --- | --- | --- |
| Frame | 100.0% | 82 | 100.0% | 5466 |
| Ethernet | 100.0% | 82 | 21.0% | 1148 |
| Internet Protocol Version 4 | 100.0% | 82 | 30.0% | 1640 |
| Transmission Control Protocol | 100.0% | 82 | 48.3% | 2640 |
| **Data** | **46.3%** | **38** | **0.7%** | **38** |

**分析结论**

1. 全部 82 个包均为 `Ethernet → IPv4 → TCP`，**不存在 HTTP、DNS、TLS 等应用层协议**。说明该流量不是标准 Web 通信。
2. 底层存在 `Data` 层，共 **38 个包、38 字节**，即 TCP 载荷未被任何应用层协议解析。
3. **关键异常**：38 个包共 38 字节，平均每包仅 1 字节。正常 TCP 传输不会出现如此碎片化的载荷分布，此为典型的人为构造特征。

> 流量分析题中，**协议分层统计的异常值往往是第一线索**。此题中「Data 层 46.3% 的分组占比但仅 0.7% 的字节占比」这一矛盾比例，直接指向隐藏信道。

---

### 3.2 阶段二：数据包列表观察

**操作**

在 Wireshark 主界面观察包列表（或使用过滤器 `tcp.len == 1`）。

**结果**

```text
No.  Time        Source           Destination      Proto Len Info
8    24.723642   172.31.98.199    192.155.81.86    TCP   67  46046 → 7890 [PSH, ACK] Seq=3 AcK=1 Win=29312 Len=1
10   25.724349   172.31.98.199    192.155.81.86    TCP   67  46046 → 7890 [PSH, ACK] Seq=4 Ack=1 Win=29312 Len=1
12   26.724839   172.31.98.199    192.155.81.86    TCP   67  46046 → 7890 [PSH, ACK] Seq=5 Ack=1 Win=29312 Len=1
14   27.725043   172.31.98.199    192.155.81.86    TCP   67  46046 → 7890 [PSH, ACK] Seq=6 Ack=1 Win=29312 Len=1
16   28.725317   172.31.98.199    192.155.81.86    TCP   67  46046 → 7890 [PSH, ACK] Seq=7 Ack=1 Win=29312 Len=1
18   29.725583   172.31.98.199    192.155.81.86    TCP   67  46046 → 7890 [PSH, ACK] Seq=8 Ack=1 Win=29312 Len=1
20   30.725851   172.31.98.199    192.155.81.86    TCP   67  46046 → 7890 [PSH, ACK] Seq=9 Ack=1 Win=29312 Len=1
...
```

**观察结论**

| 特征 | 数值 | 含义 |
| --- | --- | --- |
| 通信方向 | 单向（`172.31.98.199:46046 → 192.155.81.86:7890`） | 数据单向外发 |
| TCP 标志 | `PSH, ACK` | 携带有效载荷 |
| **每包长度** | **Len = 1** | **每包恰含 1 字节数据** |
| 时间间隔 | 约 1 秒/包（24.72 → 25.72 → 26.72…） | 高度规律，人为构造 |
| 目标端口 | **7890** | Clash 默认 HTTP 代理端口 |

**分析结论**

> 该流量呈现出典型的 **隐蔽信道（Covert Channel）** 特征：
>
> - 每个 TCP 包仅携带 1 字节
> - 发送间隔严格固定约 1 秒
> - 单向传输，且目标端口为非常规服务端口
>
> 这些特征与正常应用通信不符，推断为**将一段文本按字节拆分，逐字节通过 TCP 载荷外发**。
>
> 提示：目标端口 7890 为 Clash 代理默认端口，可能是出题人模拟代理流量，也可能仅为干扰项，不影响核心分析。

---

### 3.3 阶段三：载荷批量提取

**操作（图形界面方式）**

1. 在 Wireshark 过滤器中输入：`tcp.len == 1`
2. 逐个查看每个包的 `Data` 字段（每个包仅 1 字符）
3. 按顺序记录

**缺点**：38 个包需手动逐个记录，效率低。

**操作（tshark 命令行方式）**

```powershell
& "C:\Program Files\Wireshark\tshark.exe" `
    -r "C:\Users\ftl22\Desktop\666ad926d8f138992c8d35934e45a6863636d350bf1f53442365d6573febc804\easycap.pcap" `
    -Y "tcp.len==1" `
    -T fields `
    -e tcp.payload
```

**参数说明**

| 参数 | 含义 |
| --- | --- |
| `-r <文件>` | 读取 pcap 文件 |
| `-Y "tcp.len==1"` | 显示过滤器：仅保留 TCP 载荷长度为 1 的包 |
| `-T fields` | 输出格式：仅输出指定字段值 |
| `-e tcp.payload` | 指定提取字段：TCP 负载（十六进制表示） |

**结果**

输出 38 行，每行一个两位十六进制数：

```text
46
4c
41
47
3a
33
38
35
62
38
37
61
66
63
38
36
37
31
64
65
65
30
37
35
35
30
32
39
30
64
31
36
61
38
30
37
31
0a
```

**共 38 个字节**，与协议分级统计中的 `Data: 38 包 / 38 字节` 完全吻合。

---

### 3.4 阶段四：十六进制转 ASCII

**操作**

使用 PowerShell 完成转换：

```powershell
$hex = & "C:\Program Files\Wireshark\tshark.exe" -r "文件路径" -Y "tcp.len==1" -T fields -e tcp.payload
$bytes = $hex | Where-Object { $_ -match '^[0-9a-fA-F]{2}$' } | ForEach-Object { [Convert]::ToByte($_, 16) }
-join ($bytes | ForEach-Object { [char]$_ })
```

**转换结果**

```text
FLAG:385b87afc8671dee07550290d16a8071
```

**字节对照（部分）**

| 十六进制 | ASCII |
| --- | --- |
| `46` | `F` |
| `4c` | `L` |
| `41` | `A` |
| `47` | `G` |
| `3a` | `:` |
| `33` | `3` |
| `38` | `8` |
| `35` | `5` |
| `0a` | 换行（LF） |

---

### 3.5 阶段五：Flag 提交

提取结果为：

```text
FLAG:385b87afc8671dee07550290d16a8071
```

根据题目要求的 `flag{...}` 格式，将冒号后的内容填入花括号：

```text
flag{385b87afc8671dee07550290d16a8071}
```

提交成功，题目完成。

---

## 四、完整解题链路

```text
easycap.pcap（82 包 / 5466 字节）
    │
    ├─ 协议分级统计
    │     └─ 发现 Data 层：38 包 / 38 字节（平均每包 1 字节）
    │            │
    │            └─ 异常结论：疑似隐藏信道
    │
    ├─ 包列表观察
    │     ├─ 每包 [PSH, ACK] Len=1
    │     ├─ 时间间隔约 1 秒
    │     └─ 端口 46046 → 7890
    │
    ├─ tshark 批量提取
    │     └─ -Y "tcp.len==1" -T fields -e tcp.payload
    │            │
    │            └─ 得到 38 个十六进制字节
    │
    ├─ 十六进制转 ASCII
    │     └─ FLAG:385b87afc8671dee07550290d16a8071
    │
    └─ 格式转换
          └─ flag{385b87afc8671dee07550290d16a8071}
```

---

## 五、知识点总结

### 5.1 隐蔽信道（Covert Channel）

**定义**：利用协议中「本不用于传输数据」的部分承载信息，从而绕过常规检测。

**常见类型**

| 类型 | 载体 | 识别特征 |
| --- | --- | --- |
| TCP 隐蔽信道 | TCP 载荷（每包 1 字节） | 载荷极小、间隔规律 |
| ICMP 隧道 | ICMP payload | 大量异常长度的 ICMP 包 |
| DNS 隧道 | DNS 查询域名 | 超长域名、编码字符串 |
| HTTP 隐蔽信道 | Header / Cookie | 异常头部字段 |
| 时序信道 | 包间隔时间 | 时间间隔编码二进制 |

**本题属于第一种：TCP 载荷隐蔽信道。**

### 5.2 流量分析的核心判断依据

| 判断维度 | 观察内容 | 异常表现 |
| --- | --- | --- |
| **协议层级** | 是否有应用层解析 | 只有 TCP 无 HTTP，可疑 |
| **包大小分布** | 长度是否均匀 | 全部 Len=1，异常 |
| **时间间隔** | 是否规律 | 固定约 1 秒，人为痕迹 |
| **通信方向** | 单向还是双向 | 大量单向发送，可疑 |
| **端口** | 是否为标准端口 | 非常规端口需关注 |

### 5.3 tshark 字段提取要点

| 参数 | 作用 |
| --- | --- |
| `-r` | 读取 pcap 文件 |
| `-Y` | 显示过滤器（语法同 Wireshark） |
| `-T fields` | 输出为字段模式（必须配合 `-e`） |
| `-e <字段名>` | 指定要提取的字段 |
| `-E separator=,` | 字段分隔符（导出 CSV 时使用） |

---

## 六、技术要点详解

### 6.1 为什么用协议分级统计作为起点

Wireshark 的 **Statistics → Protocol Hierarchy** 提供了全局视角：

- 展示了各协议占用的包数与字节数比例
- **比例异常**往往直接暴露题目构造方式
- 相比逐个翻包，可节省大量时间

**本题的关键信号**：

```
Data: 46.3% 的包，却只占 0.7% 的字节
```

「包占比高、字节占比极低」= **大量极小载荷的包** = 隐蔽信道。

### 6.2 为什么选择 tshark 而非手动记录

| 方式 | 38 个包的处理时间 | 出错概率 |
| --- | --- | --- |
| 手动点击记录 | 约 5-10 分钟 | 高（易漏、易错位） |
| tshark 批量提取 | 约 3 秒 | 极低 |

**结论**：流量题中，凡是需要「按顺序提取大量字段」的场景，一律使用 tshark。

### 6.3 十六进制转 ASCII 的几种方式

**方式一：PowerShell（本题使用）**

```powershell
$bytes | ForEach-Object { [char][Convert]::ToByte($_, 16) }
```

**方式二：Python**

```python
hex_list = ["46", "4c", "41", "47"]
print("".join(chr(int(h, 16)) for h in hex_list))
```

**方式三：CyberChef**

将十六进制字符串粘贴到 CyberChef，使用 `From Hex` 操作。

**方式四：在线工具**

搜索「hex to ascii」即可。

---

## 七、踩坑记录

| # | 问题现象 | 原因 | 解决方案 |
| --- | --- | --- | --- |
| 1 | `tshark: The file "easycap.pcap" doesn't exist` | 当前工作目录为 `C:\WINDOWS\system32`，使用相对路径找不到文件 | 使用完整绝对路径，或先 `cd` 至文件所在目录 |
| 2 | 手动在 GUI 中提取效率低 | 38 个包逐个查看 | 改用 tshark 批量提取 |
| 3 | 输出结果包含非十六进制行 | tshark 可能输出空行或提示信息 | 使用正则 `^[0-9a-fA-F]{2}$` 过滤 |

---

## 八、同类题型扩展

### 8.1 常见流量分析题型

| 题型 | 特征 | 核心命令/操作 |
| --- | --- | --- |
| HTTP 明文传输 | 有 HTTP 协议解析 | `-Y "http"`、导出对象 |
| 文件传输 | 大文件流 | `File → Export Objects` |
| **TCP 隐蔽信道** | **每包极小载荷** | `tcp.len==1` |
| ICMP 隧道 | 大量 ICMP 包 | `-Y "icmp" -e icmp.data` |
| DNS 隧道 | 超长域名 | `-Y "dns" -e dns.qry.name` |
| USB 流量 | `usb.capdata` | 键码还原 |
| 加密流量 | TLS 握手 | 需导入密钥解密 |

### 8.2 流量题通用排查流程

```text
① 协议分级统计（Statistics → Protocol Hierarchy）
        ↓
② 观察异常协议 / 异常比例
        ↓
③ 查看会话（Statistics → Conversations）
        ↓
④ 确定可疑 IP / 端口 / 流
        ↓
⑤ 追踪流（Follow TCP Stream）或批量提取字段
        ↓
⑥ 解码（hex / base64 / 编码还原）
```

### 8.3 常用 tshark 命令速查

```powershell
# 协议分层统计
tshark -r x.pcap -z io,phs

# 会话统计
tshark -r x.pcap -z conv,tcp

# 提取所有 TCP 载荷
tshark -r x.pcap -Y "tcp.len>0" -T fields -e tcp.payload

# 搜索关键词
tshark -r x.pcap -Y "tcp contains \"flag\"" -T fields -e frame.number -e tcp.payload

# 导出 HTTP 对象
tshark -r x.pcap --export-objects http,./out

# 追踪第 0 个流
tshark -r x.pcap -z follow,tcp,ascii,0

