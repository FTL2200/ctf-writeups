# CTF Pwn 入门 NC 题 Writeup

## 一、题目信息

- **题目名称**：Pwn入门第一课之 NC ~
- **分值**：434 pts
- **类型**：Pwn / 入门
- **实例入口**：`ctf.wanxze.space:32918`
- **考察点**：netcat 连接、远程 shell 交互、基础 Linux 命令

## 二、赛前分析

题目名字直接点了方向：**"NC"**，即 netcat，是 Pwn 题最基础的远程接入工具。

拿到题目后第一反应是把地址粘浏览器，但访问 `https://ctf.wanxze.space:32918` 直接报错：

```
ERR_SSL_PROTOCOL_ERROR（此网站无法提供安全连接）
```

**原因**：这个端口跑的是 **nc 服务（裸 TCP）**，不是 HTTP/HTTPS 网页服务。浏览器默认用 TLS 去握手，而服务端只返回纯文本，握手失败，于是浏览器直接拒绝连接。

**结论**：这不是网页题，必须用 TCP 客户端（nc / pwntools）连接，不能丢进浏览器。

## 三、环境准备

Windows 系统默认**不带 nc**，直接执行：

```
C:\Users\ftl22>nc ctf.wanxze.space 32918
'nc' 不是内部或外部命令，也不是可运行的程序或批处理文件。
```

因此改用 Python 生态的 **pwntools**（CTF Pwn 标准工具）。

安装命令（清华源加速）：

```bash
pip install pwntools -i https://pypi.tuna.tsinghua.edu.cn/simple
```

验证安装：

```bash
python -c "import pwn; print(pwn.__version__)"
```

本次安装版本：**pwntools-4.15.0**。

## 四、第一次连接（踩坑记录）

先用一行命令快速探测端口：

```bash
python -c "from pwn import *;r=remote('ctf.wanxze.space',32918);print(r.recvall(timeout=3))"
```

输出：

```
[+] Opening connection to ctf.wanxze.space on port 32918: Done
[x] Receiving all data: 68B
[+] Receiving all data: Done (70B)
[*] Closed connection to ctf.wanxze.space port 32918
b'G1v3 y0u sh3ll & try t0 g3t fl4g!\r\nn0t1ce: press ctrl+c/d to quit!\r\n$ '
```

**关键信号**：服务端返回了一个 `$ ` 提示符，说明它给我们开了一个 **远程 shell**，正等着我们输命令：

- `G1v3 y0u sh3ll & try t0 g3t fl4g!` → 给你一个 shell，去拿 flag
- `n0t1ce: press ctrl+c/d to quit!` → 想退出按 Ctrl+C / Ctrl+D
- `$ ` → shell 提示符，等待输入

**踩坑点**：`recvall()` 的语义是"收完数据就关闭连接"。但 shell 是**持续等待输入**的，`recvall` 收满 3 秒超时后主动断线，根本没机会执行命令，所以只拿到了欢迎语。

> 经验教训：**凡是服务端返回 `$ ` / `# ` 这类提示符的题，一律用交互式连接，绝不能用 recvall。**

## 五、正确利用过程

### 5.1 编写利用脚本

切换到 `C:\Users\ftl22\Desktop` 目录，新建 `exp.py`：

```python
from pwn import *

r = remote('ctf.wanxze.space', 32918)
r.interactive()
```

### 5.2 运行并交互

在 VSCode 终端（PowerShell）运行：

```bash
python exp.py
```

进入交互模式，依次执行：

```bash
$ ls
challenge  flag

$ cat flag
flag{f6ec529e-8930-4049-8358-711aa9bc1d34}

$ [*] Got EOF while reading in interactive
```

### 5.3 结果

成功拿到 flag：

```
flag{f6ec529e-8930-4049-8358-711aa9bc1d34}
```

提交后提示**答案正确**。

## 六、解题脚本（可复用版）

```python
from pwn import *

context.log_level = 'info'

# 1. 连接远程实例
r = remote('ctf.wanxze.space', 32918)

# 2. 等待 shell 提示符
r.recvuntil(b'$ ')

# 3. 枚举目录，定位 flag 文件
r.sendline(b'ls')
print('[DIR]', r.recv(timeout=2).decode(errors='ignore'))

# 4. 读取 flag
r.sendline(b'cat flag')
print('[FLAG]', r.recv(timeout=2).decode(errors='ignore'))

r.close()
```

运行：

```bash
python exp.py
```

## 七、工具清单

做这道题（以及后续 Pwn 入门题）需要以下几样工具：

| 工具 | 用途 | 获取方式 |
| --- | --- | --- |
| **Python 3.11** | 运行 pwntools 脚本 | 已安装（本机 Python 3.11 64-bit） |
| **pwntools 4.15.0** | 核心利用框架，替代 nc 连接远程 shell | `pip install pwntools -i https://pypi.tuna.tsinghua.edu.cn/simple` |
| **VSCode** | 编写调试 exp.py | 已安装 |
| **Windows Terminal / PowerShell** | 执行脚本 | 系统自带 |
| **netcat（nc）** | 备用连接工具（Windows 无自带） | 可下载 nc.exe，或用 Nmap 自带 ncat 替代 |
| **WSL / Linux 环境** | 长期做 Pwn 题推荐，交互体验更好 | 可选，Microsoft Store 安装 Ubuntu |

**本机实际用到**：Python 3.11 + pwntools 4.15.0 + VSCode，已足够完成本题。

## 八、知识点总结

1. **Pwn 题实例入口不是网页**——带端口的地址要用 TCP 客户端连，浏览器报 `ERR_SSL_PROTOCOL_ERROR` 是典型"用错客户端"信号。
2. **Windows 无 nc**——用 pwntools 的 `remote()` 替代，底层纯 Python socket，无需系统 nc。
3. **recvall vs interactive**——服务端给 shell 时 `recvall` 会提前断连；交互式 shell 必须用 `interactive()` 或 `recvuntil + sendline` 流程。
4. **识别 shell 提示符**——返回内容里的 `$ ` / `# ` 是"可以下发命令"的强烈信号。
5. **文件定位套路**——先 `ls` 看目录，再 `cat` 目标文件；常见文件名有 `flag`、`flag.txt`、`/flag`。

---

这题思路典型：**"看到端口 → 别用浏览器 → 用 nc/pwntools 连 → 有 shell 就直接读 flag"**，是 Pwn 入门必迈的一道坎。
