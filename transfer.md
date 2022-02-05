# 前言

最近刚好弄到一台新加坡的机器，打算趁这个机会写一篇文章记录下转移过程

本次操作的机器都是 `Oracle` 的机器

**请根据自身机器和环境自行摸索，此教程顾及不了全部机器，只能作为参考。**

# 环境

## 需要转移的机器

机器架构：`#AMD` `#1C1G` `#Linux` `#Ubuntu` `#x86_64`
<details>
<summary>详细信息</summary>

```
root@instance-20211003-1301 
--------------------------- 
OS: Ubuntu 20.04.3 LTS x86_64 
Host: KVM/QEMU (Standard PC (i440FX + PIIX, 1996) pc-i440fx-4.2) 
Kernel: 5.11.0-1027-oracle 
Uptime: 4 days, 20 hours, 48 mins 
Packages: 677 (dpkg), 4 (snap) 
Shell: bash 5.0.17 
Terminal: pypy3 
CPU: AMD EPYC 7551 (2) @ 1.996GHz
```
| key | value |
| --- | ----- |
| Python 版本 | `3.7.10` |
| Telethon 版本 | `1.24.0` |
| Redis 数据库 | `有` |

对，我 PagerMaid 用的是 `pypy3.7`，所以有一些部分可能会有区别。

</details>

---

## 目标机器

机器架构：`#ARM` `#4C24G` `#Linux` `#Ubuntu` `#AARCH64`
<details>
<summary>详细信息</summary>

```
root@pagermaid 
-------------- 
OS: Ubuntu 20.04.3 LTS aarch64 
Host: KVM Virtual Machine virt-4.2 
Kernel: 5.4.0-97-generic 
Uptime: 1 hour, 21 mins 
Packages: 710 (dpkg) 
Shell: bash 5.0.17 
Resolution: 1024x768 
CPU: (4) 
GPU: 00:01.0 Red Hat, Inc. Virtio GPU 
Memory: 384MiB / 23996MiB
```
</details>

---

# 设置环境

首先，先给目标机器设置一下 `PagerMaid` 环境
\
我这里根据[安装软件包](https://github.com/Xtao-Labs/PagerMaid-Modify/wiki/Ubuntu-16.04-%E5%AE%89%E8%A3%85%E8%AF%A6%E8%A7%A3#%E5%AE%89%E8%A3%85%E8%BD%AF%E4%BB%B6%E5%8C%85)的教程把需要的软件包安装一下。
\
至于安装什么软件包就看你需要什么功能了，我是全部安装的。

建议使用 `Ubuntu OS`, 因为可以缩短安装时间（毕竟全部都有现成的软件包）

个人建议别做死尝试 `Oracle Linux`，会把时间浪费在除错上面

## 依赖包架构不同

有些依赖包可能没有，比如 `imagemagick`，当遇到时会发生以下错误

```sh
ubuntu@pagermaid:/var/lib$ sudo apt-get install imagemagick -y
Reading package lists... Done
Building dependency tree       
Reading state information... Done
Package imagemagick is not available, but is referred to by another package.
This may mean that the package is missing, has been obsoleted, or
is only available from another source

E: Package 'imagemagick' has no installation candidate
```

解决方法也很简单，就是找个相同架构的就行了

其它依赖包也是这样处理，不过我建议要是没找到可信任的源就自己编译好了。

## 创建 PyPy 环境

- `PyPy` 的[安装教程（英文）](https://doc.pypy.org/en/latest/install.html)

`PyPy` 最新版本可以在[官网](https://www.pypy.org/download.html)取得，记得按照自己机器的型号下载并安装
\
我这里安装的是 `pypy3.8`，架构 `aarch64`，请替换成属于自己机器的架构
\
我个人习惯把二进制文件放在 `/usr/local`

我执行的指令：

```sh
cd /tmp
wget https://downloads.python.org/pypy/pypy3.8-v7.3.7-aarch64.tar.bz2
tar -xf pypy3.8-v7.3.7-aarch64.tar.bz2 
sudo mv pypy3.8-v7.3.7-aarch64 /usr/local
sudo ln -s /usr/local/pypy3.8-v7.3.7-aarch64/bin/pypy /usr/local/bin/pypy
sudo ln -s /usr/local/pypy3.8-v7.3.7-aarch64/bin/pypy3 /usr/local/bin/pypy3
sudo ln -s /usr/local/pypy3.8-v7.3.7-aarch64/bin/pypy3.8 /usr/local/bin/pypy3.8
sudo ln -s /usr/local/pypy3.8-v7.3.7-aarch64/bin/python /usr/local/bin/python
sudo ln -s /usr/local/pypy3.8-v7.3.7-aarch64/bin/python3 /usr/local/bin/python3
sudo ln -s /usr/local/pypy3.8-v7.3.7-aarch64/bin/python3.8 /usr/local/bin/python3.8
pypy3 -m ensurepip
pypy3 -m pip install -U pip wheel
```

<details>
<summary>安装记录档</summary>

```bash
sam@pagermaid:/tmp$ cd /tmp
sam@pagermaid:/tmp$ wget https://downloads.python.org/pypy/pypy3.8-v7.3.7-aarch64.tar.bz2
--2022-02-05 09:03:46--  https://downloads.python.org/pypy/pypy3.8-v7.3.7-aarch64.tar.bz2
Resolving downloads.python.org (downloads.python.org)... 151.101.192.175, 151.101.128.175, 151.101.64.175, ...
Connecting to downloads.python.org (downloads.python.org)|151.101.192.175|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 28871528 (28M) [application/x-tar]
Saving to: 'pypy3.8-v7.3.7-aarch64.tar.bz2'

pypy3.8-v7.3.7-aarch64.tar.bz2         100%[===========================================================================>]  27.53M  --.-KB/s    in 0.1s    

2022-02-05 09:03:46 (230 MB/s) - 'pypy3.8-v7.3.7-aarch64.tar.bz2' saved [28871528/28871528]

sam@pagermaid:/tmp$ tar -xf pypy3.8-v7.3.7-aarch64.tar.bz2
sam@pagermaid:/tmp$ sudo mv pypy3.8-v7.3.7-aarch64 /usr/local
sam@pagermaid:/tmp$ sudo ln -s /usr/local/pypy3.8-v7.3.7-aarch64/bin/pypy /usr/local/bin/pypy
sam@pagermaid:/tmp$ sudo ln -s /usr/local/pypy3.8-v7.3.7-aarch64/bin/pypy3 /usr/local/bin/pypy3
sam@pagermaid:/tmp$ sudo ln -s /usr/local/pypy3.8-v7.3.7-aarch64/bin/pypy3.8 /usr/local/bin/pypy3.8
sam@pagermaid:/tmp$ sudo ln -s /usr/local/pypy3.8-v7.3.7-aarch64/bin/python /usr/local/bin/python
sam@pagermaid:/tmp$ sudo ln -s /usr/local/pypy3.8-v7.3.7-aarch64/bin/python3 /usr/local/bin/python3
sam@pagermaid:/tmp$ sudo ln -s /usr/local/pypy3.8-v7.3.7-aarch64/bin/python3.8 /usr/local/bin/python3.8
sam@pagermaid:/tmp$ pypy3 -m ensurepip
Looking in links: /tmp/tmpp494xmm5
Processing ./tmpp494xmm5/setuptools-56.0.0-py3-none-any.whl
Processing ./tmpp494xmm5/pip-21.1.1-py3-none-any.whl
Installing collected packages: setuptools, pip
  WARNING: The scripts pip3 and pip3.8 are installed in '/usr/local/pypy3.8-v7.3.7-aarch64/bin' which is not on PATH.
  Consider adding this directory to PATH or, if you prefer to suppress this warning, use --no-warn-script-location.
Successfully installed pip-21.1.1 setuptools-56.0.0
sam@pagermaid:/tmp$ pypy3 -m pip install -U pip wheel
Requirement already satisfied: pip in /usr/local/pypy3.8-v7.3.7-aarch64/lib/pypy3.8/site-packages (21.1.1)
Collecting pip
  Downloading pip-22.0.3-py3-none-any.whl (2.1 MB)
     |################################| 2.1 MB 19.2 MB/s 
Collecting wheel
  Downloading wheel-0.37.1-py2.py3-none-any.whl (35 kB)
Installing collected packages: wheel, pip
  WARNING: The script wheel is installed in '/usr/local/pypy3.8-v7.3.7-aarch64/bin' which is not on PATH.
  Consider adding this directory to PATH or, if you prefer to suppress this warning, use --no-warn-script-location.
  Attempting uninstall: pip
    Found existing installation: pip 21.1.1
    Uninstalling pip-21.1.1:
      Successfully uninstalled pip-21.1.1
  WARNING: The scripts pip, pip3 and pip3.8 are installed in '/usr/local/pypy3.8-v7.3.7-aarch64/bin' which is not on PATH.
  Consider adding this directory to PATH or, if you prefer to suppress this warning, use --no-warn-script-location.
Successfully installed pip-22.0.3 wheel-0.37.1
```

</details>

## 创建虚拟环境

为了让 `PagerMaid-Modify` 不污染 `PyPy` 的环境，我们得创建一个虚拟环境，专门给 `PGM`。

我执行的指令：

```sh
cd /var/lib/pagermaid/
sudo pypy3 -m venv venv
source venv/bin/activate
```

<details>
<summary>执行记录档</summary>

```sh
sam@pagermaid:/tmp# cd /var/lib/pagermaid/
sam@pagermaid:/var/lib/pagermaid# sudo pypy3 -m venv venv
sam@pagermaid:/var/lib/pagermaid# source venv/bin/activate
(venv) sam@pagermaid:/var/lib/pagermaid#
```

</details>

然后按照正常的[安装依赖包](https://github.com/Xtao-Labs/PagerMaid-Modify/wiki/Ubuntu-16.04-%E5%AE%89%E8%A3%85%E8%AF%A6%E8%A7%A3#%E5%AE%89%E8%A3%85%E8%BD%AF%E4%BB%B6%E5%8C%85)流程即可。

在你准备安装之前，你肯定会遇到下面的问题，请先把下面的问题看完再安装。

## 安装依赖包遇到 lxml 报错了

在执行 `pip3 install -r requirements.txt` 的时候会遇到以下情况

<details>
<summary>执行记录</summary>

```sh
(venv) sam@pagermaid:/var/lib/pagermaid# pip3 install -r requirements.txt
Collecting psutil>=5.8.0
  Downloading psutil-5.9.0.tar.gz (478 kB)
     |████████████████████████████████| 478 kB 17.2 MB/s 
Collecting PyQRCode>=1.2.1
  Downloading PyQRCode-1.2.1.zip (41 kB)
     |████████████████████████████████| 41 kB 549 kB/s 
Collecting pypng>=0.0.20
  Downloading pypng-0.0.21-py3-none-any.whl (48 kB)
     |████████████████████████████████| 48 kB 5.8 MB/s 
Collecting pyzbar>=0.1.8
  Downloading pyzbar-0.1.8-py2.py3-none-any.whl (28 kB)
Collecting emoji>=1.2.0
  Downloading emoji-1.6.3.tar.gz (174 kB)
     |████████████████████████████████| 174 kB 4.2 MB/s 
Collecting email-validator>=1.1.3
  Downloading email_validator-1.1.3-py2.py3-none-any.whl (18 kB)
Collecting youtube_dl>=2021.6.6
  Downloading youtube_dl-2021.12.17-py2.py3-none-any.whl (1.9 MB)
     |████████████████████████████████| 1.9 MB 24.1 MB/s 
Collecting PyYAML>=5.4.1
  Downloading PyYAML-6.0.tar.gz (124 kB)
     |████████████████████████████████| 124 kB 19.6 MB/s 
  Installing build dependencies ... done
  Getting requirements to build wheel ... done
    Preparing wheel metadata ... done
Collecting redis>=3.5.3
  Downloading redis-4.1.2-py3-none-any.whl (173 kB)
     |████████████████████████████████| 173 kB 20.0 MB/s 
Collecting coloredlogs>=15.0.1
  Downloading coloredlogs-15.0.1-py2.py3-none-any.whl (46 kB)
     |████████████████████████████████| 46 kB 4.6 MB/s 
Collecting requests[socks]>=2.25.1
  Downloading requests-2.27.1-py2.py3-none-any.whl (63 kB)
     |████████████████████████████████| 63 kB 1.8 MB/s 
Collecting httpx>=0.21.3
  Downloading httpx-0.22.0-py3-none-any.whl (84 kB)
     |████████████████████████████████| 84 kB 2.4 MB/s 
Collecting pytz>=2021.1
  Downloading pytz-2021.3-py2.py3-none-any.whl (503 kB)
     |████████████████████████████████| 503 kB 14.7 MB/s 
Collecting cowpy>=1.1.0
  Downloading cowpy-1.1.4.tar.gz (27 kB)
  Installing build dependencies ... done
  Getting requirements to build wheel ... done
    Preparing wheel metadata ... done
Collecting translators>=5.0.1
  Downloading translators-5.0.1-py3-none-any.whl (23 kB)
Collecting gTTS>=2.2.2
  Downloading gTTS-2.2.3-py3-none-any.whl (25 kB)
Collecting gTTS-token>=1.1.4
  Downloading gTTS-token-1.1.4.tar.gz (3.9 kB)
Collecting wordcloud>=1.8.1
  Downloading wordcloud-1.8.1.tar.gz (220 kB)
     |████████████████████████████████| 220 kB 12.1 MB/s 
Collecting Telethon>=1.24.0
  Downloading Telethon-1.24.0-py3-none-any.whl (528 kB)
     |████████████████████████████████| 528 kB 17.8 MB/s 
Collecting Pillow>=8.2.0
  Downloading Pillow-9.0.1.tar.gz (49.5 MB)
     |████████████████████████████████| 49.5 MB 8.2 MB/s 
Collecting python-magic>=0.4.24
  Downloading python_magic-0.4.25-py2.py3-none-any.whl (13 kB)
Collecting Pygments>=2.9.0
  Downloading Pygments-2.11.2-py3-none-any.whl (1.1 MB)
     |████████████████████████████████| 1.1 MB 43.7 MB/s 
Collecting speedtest-cli>=2.1.3
  Downloading speedtest_cli-2.1.3-py2.py3-none-any.whl (23 kB)
Collecting GitPython>=3.1.17
  Downloading GitPython-3.1.26-py3-none-any.whl (180 kB)
     |████████████████████████████████| 180 kB 48.1 MB/s 
Collecting Werkzeug>=2.0.1
  Downloading Werkzeug-2.0.2-py3-none-any.whl (288 kB)
     |████████████████████████████████| 288 kB 46.9 MB/s 
Collecting Flask>=2.0.1
  Downloading Flask-2.0.2-py3-none-any.whl (95 kB)
     |████████████████████████████████| 95 kB 5.3 MB/s 
Collecting Flask-SQLAlchemy>=2.5.1
  Downloading Flask_SQLAlchemy-2.5.1-py2.py3-none-any.whl (17 kB)
Collecting Flask-Login>=0.5.0
  Downloading Flask_Login-0.5.0-py2.py3-none-any.whl (16 kB)
Collecting Flask-Bcrypt>=0.7.1
  Downloading Flask-Bcrypt-0.7.1.tar.gz (5.1 kB)
Collecting Flask-WTF>=0.15.1
  Downloading Flask_WTF-1.0.0-py3-none-any.whl (12 kB)
Collecting WTForms>=2.3.3
  Downloading WTForms-3.0.1-py3-none-any.whl (136 kB)
     |████████████████████████████████| 136 kB 49.8 MB/s 
Collecting cheroot>=8.5.2
  Downloading cheroot-8.6.0-py2.py3-none-any.whl (104 kB)
     |████████████████████████████████| 104 kB 54.0 MB/s 
Collecting python-socks[asyncio]>=1.2.4
  Downloading python_socks-2.0.3-py3-none-any.whl (49 kB)
     |████████████████████████████████| 49 kB 7.8 MB/s 
Collecting certifi>=2021.5.30
  Downloading certifi-2021.10.8-py2.py3-none-any.whl (149 kB)
     |████████████████████████████████| 149 kB 52.4 MB/s 
Collecting magic_google>=0.2.9
  Downloading magic_google-0.2.9.tar.gz (4.1 kB)
Collecting sentry-sdk>=1.5.2
  Downloading sentry_sdk-1.5.4-py2.py3-none-any.whl (143 kB)
     |████████████████████████████████| 143 kB 14.7 MB/s 
Collecting analytics-python>=1.4.0
  Downloading analytics_python-1.4.0-py2.py3-none-any.whl (15 kB)
Collecting beautifulsoup4>=4.9.3
  Downloading beautifulsoup4-4.10.0-py3-none-any.whl (97 kB)
     |████████████████████████████████| 97 kB 8.6 MB/s 
Collecting apscheduler>=3.8.1
  Downloading APScheduler-3.8.1-py2.py3-none-any.whl (59 kB)
     |████████████████████████████████| 59 kB 8.0 MB/s 
Collecting dnspython>=1.15.0
  Downloading dnspython-2.2.0-py3-none-any.whl (266 kB)
     |████████████████████████████████| 266 kB 33.4 MB/s 
Collecting idna>=2.0.0
  Downloading idna-3.3-py3-none-any.whl (61 kB)
     |████████████████████████████████| 61 kB 7.6 MB/s 
Collecting deprecated>=1.2.3
  Downloading Deprecated-1.2.13-py2.py3-none-any.whl (9.6 kB)
Collecting packaging>=20.4
  Downloading packaging-21.3-py3-none-any.whl (40 kB)
     |████████████████████████████████| 40 kB 6.7 MB/s 
Collecting humanfriendly>=9.1
  Downloading humanfriendly-10.0-py2.py3-none-any.whl (86 kB)
     |████████████████████████████████| 86 kB 6.6 MB/s 
Collecting charset-normalizer
  Downloading charset_normalizer-2.0.11-py3-none-any.whl (39 kB)
Collecting sniffio
  Downloading sniffio-1.2.0-py3-none-any.whl (10 kB)
Collecting rfc3986[idna2008]<2,>=1.3
  Downloading rfc3986-1.5.0-py2.py3-none-any.whl (31 kB)
Collecting httpcore<0.15.0,>=0.14.5
  Downloading httpcore-0.14.7-py3-none-any.whl (68 kB)
     |████████████████████████████████| 68 kB 7.5 MB/s 
Collecting PyExecJS>=1.5.1
  Downloading PyExecJS-1.5.1.tar.gz (13 kB)
```

</details>

```sh
Collecting lxml>=4.5.0
  Downloading lxml-4.7.1.tar.gz (3.2 MB)
     |████████████████████████████████| 3.2 MB 30.6 MB/s 
    ERROR: Command errored out with exit status 1:
     command: /var/lib/pagermaid/venv/bin/pypy3 -c 'import io, os, sys, setuptools, tokenize; sys.argv[0] = '"'"'/tmp/pip-install-46dyma7v/lxml_712ef404893b40fb96f19d5160662846/setup.py'"'"'; __file__='"'"'/tmp/pip-install-46dyma7v/lxml_712ef404893b40fb96f19d5160662846/setup.py'"'"';f = getattr(tokenize, '"'"'open'"'"', open)(__file__) if os.path.exists(__file__) else io.StringIO('"'"'from setuptools import setup; setup()'"'"');code = f.read().replace('"'"'\r\n'"'"', '"'"'\n'"'"');f.close();exec(compile(code, __file__, '"'"'exec'"'"'))' egg_info --egg-base /tmp/pip-pip-egg-info-9bs3x1tf
         cwd: /tmp/pip-install-46dyma7v/lxml_712ef404893b40fb96f19d5160662846/
    Complete output (3 lines):
    Building lxml version 4.7.1.
    Building without Cython.
    Error: Please make sure the libxml2 and libxslt development packages are installed.
    ----------------------------------------
WARNING: Discarding https://files.pythonhosted.org/packages/84/74/4a97db45381316cd6e7d4b1eb707d7f60d38cb2985b5dfd7251a340404da/lxml-4.7.1.tar.gz#sha256=a1613838aa6b89af4ba10a0f3a972836128801ed008078f8c1244e65958f1b24 (from https://pypi.org/simple/lxml/) (requires-python:>=2.7, !=3.0.*, !=3.1.*, !=3.2.*, !=3.3.*, != 3.4.*). Command errored out with exit status 1: python setup.py egg_info Check the logs for full command output.
  Downloading lxml-4.6.5.tar.gz (3.2 MB)
     |████████████████████████████████| 3.2 MB 26.8 MB/s
...  # 下面重复
```

首先，不要慌，先 `Ctrl+C` 退出安装依赖包过程

我们重点放在这句

`Error: Please make sure the libxml2 and libxslt development packages are installed.`

也就是说我们没安装 `libxml2` 和 `libxslt`，那么先安装就可以了吧？

指令（[来源参考](https://stackoverflow.com/a/5178444)）：

`Ubuntu`:

```sh
sudo apt-get install libxml2-dev libxslt-dev python-dev -y
```

`Oracle Linux`:

```sh
sudo yum install libxslt-devel libxml2-devel -y
```

### matplotlib 安装问题

大部分都是因为没安装足够或软件包版本太低，尝试以下指令

`Ubuntu`:

```sh
sudo apt install libxml2-dev libxslt-dev python-dev gcc g++ make libjpeg8-dev zlib1g-dev -y
```

接下来安装教程继续操作，修改配置文件即可

# 转移

## 备份并关闭

在[登录账号](https://github.com/Xtao-Labs/PagerMaid-Modify/wiki/Ubuntu-16.04-%E5%AE%89%E8%A3%85%E8%AF%A6%E8%A7%A3#%E7%99%BB%E5%BD%95%E8%B4%A6%E5%8F%B7)之前，我们先备份一下我们旧的 `PagerMaid` 文件

### 备份

在 `PagerMaid` 中输入 `-backup` 指令

![image.png](https://s2.loli.net/2022/02/05/zqJiRnFLTpgCQm3.png)

完成后，会保存到 `/var/lib/pagermaid/pagermaid_backup.tar.gz`（如果设置了 Log 频道将会看到此文件）

![image.png](https://s2.loli.net/2022/02/05/BqcEAJeDb96ZNrY.png)

### 关闭

接下来，我们就可以关掉旧的 `PagerMaid` 了，不然等下可能会出现问题。
\
记得输入以下指令确保之后不会误启动

```sh
sudo systemctl stop pagermaid
sudo systemctl disable pagermaid
```

## 启动新的PGM并恢复备份

### 设置账号

首先，按照正常的[登录账号](https://github.com/Xtao-Labs/PagerMaid-Modify/wiki/Ubuntu-16.04-%E5%AE%89%E8%A3%85%E8%AF%A6%E8%A7%A3#%E7%99%BB%E5%BD%95%E8%B4%A6%E5%8F%B7)流程登录一次。
\
然后，也按照[进程守护](https://github.com/Xtao-Labs/PagerMaid-Modify/wiki/Ubuntu-16.04-%E5%AE%89%E8%A3%85%E8%AF%A6%E8%A7%A3#%E8%BF%9B%E7%A8%8B%E5%AE%88%E6%8A%A4)让 `PagerMaid` 可以自启动。

**注意：**`进程守护`中的 `/usr/bin/python3` 记得改成 `/var/lib/pagermaid/venv/bin/python3`

### 恢复备份

现在，我们可以恢复旧 `PagerMaid` 的备份了。

如果你的 Log 频道有上传备份文件，那你就可以对着备份文件进行 `-recovery` 指令，

否则，请自行将备份路径的文件上传到新的机器的*相同路径上*
\
不过，上传后并不需要对着备份文件回复了，可以直接输入 `-recovery` 指令

这里的例子是 `Log 频道有上传备份文件`

![image.png](https://s2.loli.net/2022/02/05/aqInd59xmo3YXvR.png)

然后 `PagerMaid` 就会开始下载并恢复备份，完成后，整个转移流程就完成了。

![image.png](https://s2.loli.net/2022/02/05/vSrnNw1OYMdh3fZ.png)

![image.png](https://s2.loli.net/2022/02/05/Ww9BeOHuoXsUAtZ.png)

# 最后

首先，我想吐槽一下，这玩意的安装难度对于小白来说有点困难，尤其是在安装依赖包上面
\
我在 `Oracle Linux` 上尝试过安装，结果变成了源码编译地狱，浪费了我一天的时间

后面好不容易更换到了 `Ubuntu OS`，还是有些地方没软件包安装的，必须手动排除错误并安装，
\
又浪费了我几个小时，真难受。

不过，结果还是令人兴奋的，最后成功把 `PagerMaid` 转移到了新加坡的机器上，这延迟也是非常舒服

![image.png](https://s2.loli.net/2022/02/05/mBeqzMG9uXk7dNh.png)

如果在安装或者设置的过程中遇到了任何问题，请第一时间打开你的搜索引擎找一下是不是有人遇到过和你一样的问题
\
而不是在群里吹半个小时。（当然如果到最后还是解决不了的话可以到群里问）

![image.png](https://s2.loli.net/2022/02/05/JjfdiAc36UBXvmq.png)
