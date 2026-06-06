# 解决SolidWorks 2026安装过程中的各种报错

## 问题描述

* 我在使用 `SolidWorks.2026.SP0.Premium.DVD.iso` 镜像安装时，出现 `vsta_setup.exe` 安装报错。
* 在安装管理程序生成注册表项 `HKLM\Software\Wow6432Node\SolidWorks` 时遭遇权限拒绝（Access is denied）。
* 在前置组件阶段，安装 `VBA 7.1` 和 `WPT` 等包时，进度条回退并提示“系统策略禁止这个安装”（错误代码 1625）。
* Flexnet 许可服务器后台脚本启动失败，提示服务无法启动（错误代码 3534）。
* 安装完成后启动主程序，直接报错“无法获得下列许可 SOLIDWORKS Design Standard。无效的(不一致的)使用许可号码。(-8,544,0)”。

## 处理过程

### 1. 解决注册表死锁与 Flexnet 服务无法启动

由于 Windows 服务的 Session 0 权限隔离以及 `.bat` 脚本所在网盘下载目录的安全限制，导致服务秒退。此外，由于之前的异常安装，导致相关注册表节点权限锁死。

* 发现问题：命令行提示 `NET HELPMSG 3534` 且无法修改注册表节点。
* 解决过程：

1. 使用管理员 PowerShell 执行 `sc.exe delete "SolidWorks Flexnet Server"` 彻底抹除死锁的旧服务。
2. 放弃系统服务注入，改用 Windows 自带的 **“任务计划程序”**，设置为“计算机启动时”，以最高权限执行 `lmgrd.exe -z -c sw_d_SSQ.lic`，成功实现开机后台静默授权。
3. 对于锁死的注册表，利用微软官方工具 `PsExec`（`psexec.exe -i -s powershell.exe`）获取最高级别的 SYSTEM 权限，强行覆写注册表完成清理。

### 2. 解决 vsta_setup.exe 安装报错（可能）

* 发现问题：
抓取到的 `dd_vsta_setup_*.log` 里面有一行：

```Log
[cite_start][3B78:44E8][2026-06-02T15:10:33]e000: Error 0x80070659: Failed to install MSI package. [cite: 1]
[cite_start][3070:5520][2026-06-02T15:10:33]e000: Error 0x80070659: Failed to configure per-machine MSI package. [cite: 1]
```

* 喂给AI帮我解释：

> *errorCode: 0x80070659 代表“系统策略禁止这个安装”。Windows 的安全机制拦截了通过常规方式静默安装的 .msi 运行库包。*

* 解决过程：
由于 VSTA（Visual Studio Tools for Applications）仅用于高级宏脚本二次开发，纯 3D 建模完全用不到，因此采用“掉包计”：

1. 进入 `PreReqs\VSTA17\` 目录，将 `vsta_setup.exe` 重命名为 `vsta_setup_BAK.exe` 备份。
2. 下载 `https://www.microsoft.com/zh-cn/download/details.aspx?id=105123` 到该目录并改名为 `vsta_setup.exe`。
3. 当 SolidWorks 安装器尝试使用静默参数调用它时，新版VSTA会正常走安装流程，从而达到非原版依赖但能跑的状态。

### 3. 解决 VBA 及 WPT 等 MSI 组件报 1625 错误

* 发现问题：
抓取到的 `SummaryIMLog_*.txt` 里面有一行：

```Log
[cite_start]17:45:42	Error	Status	157	0	"系统策略禁止这个安装。请与系统管理员联系。 [cite: 2]
[cite_start]D:\BaiduNetdiskDownload\SolidWorks.2026.SP0.Premium.DVD\PreReqs\VBA\vba71.msi" [cite: 2]
[cite_start]17:45:42	Error	Step	154	0	"** Install FAILED with code 1625" [cite: 2]
```

* 喂给AI帮我解释：

> *错误代码 1625 是典型的 Windows 现代安全子系统拦截机制。由于你的安装包在网盘下载目录下（带有 Mark of the Web 网络风险标记），且 SolidWorks 安装程序 (setup.exe) 试图在后台静默拉起这些 MSI 包，触发了 Windows 的 ASR（攻击面减少）规则或 Smart App Control 机制。系统由于没有 UI 弹窗让用户授权，直接一刀切判定为拒绝 (Deny)。*

* 解决过程：
放弃一切命令行静默安装。直接进入安装包的 `PreReqs` 目录下对应的 `VBA`、`WPT`、`CEF` 和 `LoginMgr` 文件夹，**纯手工双击运行**里面的 `.msi` 文件。通过 Windows 图形界面的 UAC 弹窗点击“是”完成人工授权。手动安装完毕后，重新运行 SolidWorks 的主 `setup.exe`，安装管理器检测到环境已满足，直接跳过并顺利安装了几十 GB 的核心主程序。

### 4. 解决启动报 (-8, 544, 0) 许可错误

* 发现问题：
主程序安装完成后，通过 `list.log` 核对目录发现，虽然核心程序 `SLDWORKS.exe` 已成功部署在自定义的 `D:\SolidWorks2026` 路径下 ，但仍是未破解的官方原版，无法识别本地搭建的服务器。

* 解决过程：
由于安装时修改了默认路径，导致破解包内原有的树状目录结构无法自动覆盖。

1. 直接进入破解文件夹 `_SolidSQUAD_` 深处提取劫持补丁核心文件 `netapi32.dll`。
2. 将其单独复制并粘贴到 `D:\SolidWorks2026\` 根目录下，确保与 `SLDWORKS.exe` 同级存放以实现劫持 。
3. 双击导入 `SolidSQUADLoaderEnabler.reg` 注册表，软件顺利启动，不再报错。

## 总结

SolidWorks 2026 采用的是老旧的“静默分发 MSI”的安装架构，这种方式极其容易触发 Windows 11 现代安全机制（特权隔离、MotW 标记拦截）。
遇到 `1625` 或 `0x80070659` 这类“策略禁止”错误，最高效的解法就是顺应微软“必须使用图形化Desktop操作UAC授权”的安全逻辑：**核心依赖组件（如 VBA/WPT）采用纯手工双击通过 UAC 安装**；**无用组件（如 VSTA）使用 cmd 掉包伪装跳过**。最后理清自定义安装路径，精准投递破解补丁即可通关。