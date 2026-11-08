# SlinkyInjector

一个最小化、可审计的 `SlinkyNative.dll` 注入器。只做一件事：把指定的 DLL 注入到 Minecraft 的 `java.exe` 进程里，然后把客户端的日志实时打出来。

> 给不想运行来路不明的第三方启动器的人：约 600 行 C#（大半是 P/Invoke 声明和注释），无混淆、无壳、无网络代码，只调用文档化的 Win32 API。

---

## 设计边界

这不是通用注入器，几个限制是**故意**的：

| 限制 | 说明 |
|---|---|
| 只认 Minecraft | 目标必须是 `java.exe` / `javaw.exe`，且拥有一个标题含 `Minecraft` 的可见窗口；其它进程一律拒绝 |
| 无网络 | 代码里没有任何 socket / WinINet / WinHTTP 导入，也不解析域名 |
| 无持久化 | 不写注册表、不装服务、不加启动项、不自更新 |
| 不做免杀 | 不混淆、不加壳、不改 AMSI/ETW、不 unhook ntdll |
| 不内置载荷 | 仓库**不包含** `SlinkyNative.dll`，需要你自己提供 |

---

## 工作原理

```
SlinkyInjector.exe
  ├─ EnumWindows           找标题含 "Minecraft" 的可见窗口
  ├─ Toolhelp32            确认该窗口属于 java.exe / javaw.exe
  ├─ Copy                  SlinkyNative.dll → %TEMP%\slinky-<pid>\
  ├─ OpenProcess
  ├─ VirtualAllocEx        在目标进程里分配一块 RW 内存
  ├─ WriteProcessMemory    写入 DLL 的 UTF-16 路径
  ├─ CreateRemoteThread    LoadLibraryW(路径)
  └─ tail slinky-native.log 直到出现 FINAL STATUS: RUNNING
```

注入用的就是最经典的 `LoadLibraryW` 远程线程，和原版启动器同一套机制。DLL 之后怎么挂到 JVM（`JNI_GetCreatedJavaVMs` + JVMTI）不属于本仓库。

DLL 会把自己的日志和 payload jar 写在**自己所在的目录**，所以必须先把它复制到一个可写目录（`%TEMP%\slinky-<pid>\`）再注入——这也是原版启动器的做法。

---

## 构建

只需要 Windows 自带的 .NET Framework 4.x（Win10/11 默认有），不需要 VS、不需要 SDK、不需要 NuGet。

```powershell
.\build.ps1
```

如果提示“禁止运行脚本”：

```powershell
powershell -ExecutionPolicy Bypass -File .\build.ps1
```

手动编译也行：

```powershell
& "$env:WINDIR\Microsoft.NET\Framework64\v4.0.30319\csc.exe" `
  /nologo /platform:x64 /optimize+ /target:exe `
  /out:build\SlinkyInjector.exe src\*.cs
```

产物：`build\SlinkyInjector.exe`（约 16 KB，x64）。

---

## 用法

```
SlinkyInjector.exe                    注入同目录下的 SlinkyNative.dll
SlinkyInjector.exe --dll <path>       指定 DLL
SlinkyInjector.exe --wait <seconds>   等 Minecraft 出现（先开 loader 再开游戏）
SlinkyInjector.exe --dry-run          只找目标，不注入
SlinkyInjector.exe --list             列出所有带标题的窗口及其进程
SlinkyInjector.exe --probe <pid>      检查对该进程有没有注入所需的权限
SlinkyInjector.exe --help
```

典型输出：

```
[*] target : java.exe (pid 15760)
[*] window : LOCAL OFFLINE | Badlion Minecraft Client v4.4.4 (1.8.9)
[*] payload: C:\...\SlinkyLoader\SlinkyNative.dll (6183424 bytes)
[*] staged : C:\Users\...\AppData\Local\Temp\slinky-15760\SlinkyNative.dll
[+] DLL loaded into the game (module handle 0xC0F00000)
[*] waiting for the client to report...
    [..] payload appended to the system class loader search
    [..] [java] Runtime: Badlion+OptiFine (official names), 20729 mappings
    [..] [java] Patched 14 classes (0 failed, 1 warnings)
    [..] [java] Slinky Semantic 0.5.0-settings initialized with 66 modules
    FINAL STATUS: RUNNING
```

退出码：

| 码 | 含义 |
|---|---|
| 0 | 注入成功且 `FINAL STATUS: RUNNING` / `RELOADED` |
| 1 | 没找到目标 / DLL 不存在 |
| 2 | 参数错误 |
| 3 | 注入失败（`OpenProcess` / `CreateRemoteThread` 等） |
| 5 | DLL 报告了非 RUNNING 的终态（如 `ALREADY_RUNNING`、`FAILED`） |
| 6 | 超时没等到 `FINAL STATUS` |

---

## 已知坑：DLL 必须支持目标 JVM

**这是最容易踩的一条。** JDK 8 和 JDK 17 的系统类加载器不是一回事（`URLClassLoader` vs `BuiltinClassLoader`），老构建的 DLL 在 Java 17 上会把 jar 挂不进去，表现为：

```
NoClassDefFoundError: dev/slinky/inject/Bootstrap
```

随后 JVM 在 `jdk.internal.loader.BuiltinClassLoader.findResourceAsStream` 里 `ACCESS_VIOLATION` 直接崩掉，并生成 `hs_err_pid*.log`。

实测：同一个 Temurin 17 运行时，一个 9.5 MB 的旧构建 100% 崩（注入早、注入晚都一样），换成支持 17 的 6.18 MB 构建后一次成功。**报错时先看 `%TEMP%\slinky-<pid>\slinky-native.log` 停在哪一行**：

- 停在 `payload ... written` → DLL 挂载类加载器失败，换 DLL
- 根本没有这个文件 → DLL 没被加载（权限不足、或目标不是 Java 进程）

---

## 网络安全

启动器本身没有网络代码。真正的风险面是**被注入的 payload**：它跑在 `java.exe` 里，和游戏共用同一个 socket，所以无法只掐掉外挂而保留联机。

- 只想断启动器：`scripts\block-network.ps1`（需管理员）
- 想连 java 一起断（单人离线才建议）：`scripts\block-network.ps1 -JavaExe "<...>\java.exe"`

---

## 目录结构

```
src/
  Program.cs           CLI 解析、目标选择、流程编排
  ProcessInjector.cs   OpenProcess → VirtualAllocEx → WriteProcessMemory → CreateRemoteThread
  NativeMethods.cs     P/Invoke 声明
  ProcessTable.cs      Toolhelp32 快照（PID → 映像名）
  WindowEnumerator.cs  EnumWindows 封装
  LogTailer.cs         跟踪 slinky-native.log
scripts/
  block-network.ps1    Windows 防火墙出站拦截
build.ps1              用系统自带 csc 编译
.github/workflows/     CI：windows-latest 上编译并上传产物
```

---

## 免责声明

本项目仅供学习 Windows 进程注入与 JVM 挂载机制、以及在**自己的机器**上研究使用。

- 在多数服务器上使用外挂违反游戏 / 服务器服务条款，账号被封自负
- 请勿用于商业用途、对抗性环境或任何未经授权的系统
- 被注入的 DLL 由使用者自行提供，其行为与本仓库无关

---

## License

MIT，见 [LICENSE](LICENSE)。

---

## English

A ~600-line, auditable loader that injects a given `SlinkyNative.dll` into the Minecraft `java.exe` process and tails the client log.

Hard boundaries: only targets `java.exe`/`javaw.exe` owning a visible window whose title contains "Minecraft"; no network code; no persistence; no anti-detection tricks; the payload DLL is not included.

Build with the `csc.exe` that ships with .NET Framework 4.x (`.\build.ps1`), run `SlinkyInjector.exe --wait 300` before starting the game. See the Chinese sections above for details.
