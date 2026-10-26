# CTF Writeup：Reverse - game_flag

## 一、题目信息

| 项目 | 内容 |
|------|------|
| 题目名称 | game_flag |
| 方向 | Reverse（逆向工程） |
| 附件 | `game_flag.exe` |
| 考察点 | PyInstaller 打包识别、pyc 反编译、程序逻辑分析 |
| Flag | `flag{Welcome_to_reverse}` |

## 二、解题思路概览

拿到 exe 附件后，先做用DIE做**文件类型识别**，发现它并非普通 C/C++ 编译产物，而是 **PyInstaller 打包的 Python 程序**。这类题目的标准套路是：

```
识别 PyInstaller → 解包出 pyc → 反编译成 py → 读源码分析逻辑 → 得到 flag
```

本题目的是考察"能不能识破 PyInstaller 外壳 + 引用反编译工具还原源码"，属于**入门级逆向**。

## 三、详细解题过程

### 1. 文件类型识别

把 `game_flag.exe` 拖进 **DIE（Detect It Easy）**，或直接用 010 Editor 查看文件头/尾部特征字符串。

关键特征：文件尾部存在 `MEI` 标记和 `PYZ-00.pyz` 相关字符串，DIE 识别为：![alt text](image.png)

```
PyInstaller[modified]
```

判定：这是一个用 **PyInstaller 打包**、并带有 **Python 3.8 运行时**的 exe，主逻辑是 Python 编写。

### 2. 解包出 pyc

使用 **pyinstxtractor-ng**（PyInstaller 解包工具）提取内部资源。

在 Windows 11 环境下，从 Release 页面下载 `pyinstxtractor-ng.exe`（注意：无后缀的是 Linux/macOS 版本，要下 `.exe`）。因工具来自 GitHub、无数字签名，运行时会被 **Microsoft Defender SmartScreen** 拦截：

- 在文件 **右键 → 属性 → 常规 → 勾选"解除锁定"** 后即可运行；或点击弹窗的"更多信息 → 仍要运行"。

执行解包：

```bash
cd C:\Users\ftl22\Downloads
pyinstxtractor-ng.exe game_flag.exe
```

输出关键信息：

```
[+] Pyinstaller version: 2.1+
[+] Python version: 3.8
[+] Length of package: 5513765 bytes
[+] Found 59 files in CArchive
[+] Possible entry point: game_flag.pyc
[+] Found 74 files in PYZ archive
[+] Successfully extracted pyinstaller archive: game_flag.exe
```

解包生成 `game_flag.exe_extracted/` 目录，其中：

- `game_flag.pyc` —— **程序主逻辑（入口点）**，反编译目标
- `PYZ-00.pyz` / `PYZ-00.pyz_extracted/` —— 打包的第三方/标准库模块
- `python38.dll`、各类 `.pyd`、`.dll` —— Python 运行时依赖

> 提示：如果多次解包，同名文件会以 UUID 后缀重存（如 `game_flag_adc500a3-...pyc`），内容一致，任选一份即可。

### 3. 反编译 pyc → py

`game_flag.pyc` 是 **Python 3.8** 编译的字节码，需要匹配版本的反编译工具。尝试过的路径：

**方案一：uncompyle6（失败）**

```bash
python -m uncompyle6 game_flag.pyc > game_flag.py
```

报错：

```
ImportError: bad magic number in 'struct': b'U\r\r\n'
```

原因：本机为 Python 3.11，与 pyc 的 3.8 版本不匹配，工具链解析标准库时崩溃。

**方案二：decompyle3（成功）**

```bash
python -m pip install decompyle3
cd C:\Users\ftl22\Downloads\game_flag.exe_extracted
C:\Users\ftl22\AppData\Local\Programs\Python\Python311\Scripts\decompyle3.exe game_flag.pyc > game_flag.py
```

> 避坑记录：
> - `python -m decompyle3` 会报 `'decompyle3' is a package and cannot be directly executed`，必须**直接调用 `decompyle3.exe` 命令行脚本**。
> - 运行前务必 `cd` 到 pyc 所在目录，否则找不到文件。
> - 若环境曾出现 `bad magic number`，清理 `Python311\Lib\__pycache__` 缓存后 `import struct` 可恢复。

### 4. 分析源码

反编译得到的核心逻辑如下（忽略 decompyle3 的调试输出）：

```python
import sys

def is_console_mode():
    return sys.stdin.isatty()

def main():
    if not is_console_mode():
        print("错误：请通过命令行窗口运行本程序")
        sys.exit(1)
    user_input = input("Please enter the content: ").strip()
    print("flag{Welcome_to_reverse}" if user_input.lower() == "enter the game" else "Invalid input")

if __name__ == "__main__":
    main()
```

逻辑一目了然：

1. **`is_console_mode()`** 检查程序是否在**命令行窗口（真终端）**中运行。直接双击 exe 时 `isatty()` 返回 False，程序会拒绝运行并退出。
2. 在命令行中运行后，提示输入内容。
3. 若输入内容（**转小写后**）等于 `enter the game`，则输出 flag。

### 5. 获取 flag

在 CMD 中运行程序并输入指定内容：

```bash
cd C:\Users\ftl22\Downloads
game_flag.exe
```

出现提示后在终端输入：

```
Please enter the content: enter the game
```

输出：

```
flag{Welcome_to_reverse}
```

> 注意：输入 `Enter The Game`、`ENTER THE GAME` 等均可，因为代码里做了 `.lower()` 转换。

## 四、Flag

```
flag{Welcome_to_reverse}
```

## 五、总结与知识点

本题是 **PyInstaller 逆向的入门模板题**，完整走通了一条标准链路，可提炼为通用解题流程：

| 步骤 | 工具 | 要点 |
|------|------|------|
| 1. 识别外壳 | DIE / 010 Editor | 关注文件尾 `MEI`、`PYZ-00.pyz` 特征 |
| 2. 解包 | pyinstxtractor-ng | 认准 Python 版本；注意 SmartScreen 拦截 |
| 3. 反编译 | decompyle3 / uncompyle6 / pylingual | **工具版本必须匹配 pyc 的 Python 版本** |
| 4. 审计源码 | 文本编辑器 | 找输入处理、比较、加密逻辑 |

**踩坑经验：**

1. **版本匹配是第一原则**。pyc 是 3.8 的，就别指望用 3.11 环境的 uncompyle6 硬解；要么装对应版本解释器，要么用在线工具（pylingual.io）。
2. **`python -m` 不是万能的**。有些包没提供 `__main__.py`，必须调用其安装目录下的命令行脚本。
3. **程序环境检查要留意**。本题用 `sys.stdin.isatty()` 做了"必须命令行运行"的限制，反向提示我们**拿到 exe 后优先在终端运行观察行为**。
4. **PyInstaller 打包 ≠ 代码被保护**。它只是把 pyc 塞进可执行文件，反编译后源码几乎原样还原，所以此类题目的难度仅在"工具链是否用对"。
