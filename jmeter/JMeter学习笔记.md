# JMeter 学习手册

> 基于已有的若依（RuoYi-Vue）接口测试项目，从零学 JMeter 到完成性能测试
> 创建日期：2026-10-15 ｜ 状态：**第①步已完成**（JMeter 5.6.3 已装好；JDK 24、中文路径、非 GUI 执行、HTML 报告 **均已实测通过**）

---

## 〇、这份手册怎么用

| 节 | 内容 | 什么时候看 |
|---|---|---|
| 一 | 全局路线图（15 步） | 先看一眼，知道要去哪 |
| 二 | 第①步：安装配置 | ✅ 已完成 |
| 三 | 第②③步：第一个测试计划 | GUI 打开后照着做 |
| 四 | 第④⑤步：测试计划骨架 + token 关联 | ★ 核心 |
| 五 | 第⑨～⑫步：性能测试 | ★★ JMeter 的主场 |
| 六 | 第⑥⑦⑧步：断言 / CSV 参数化 / 控制器 | ★ 决定测得准不准 |
| 七 | 术语中英对照 | 看界面时对照 |
| 八 | 排错手册 | 出问题时 |

> 后面每完成一步，我会往这份手册里追加对应章节。

---

## 一、全局路线图

| 阶段 | 步骤 | 状态 |
|---|---|---|
| **一、环境** | ① 安装配置 JMeter | ✅ 已完成（还差环境变量） |
| **二、跑通第一个请求** | ② 认识 5 个核心组件<br>③ 第一个 HTTP 请求 | 📖 **手册第三节已写好** |
| **三、把若依项目搬进来** | ④ 测试计划骨架（请求默认值 / 请求头 / Cookie）<br>⑤ 登录 + token 关联<br>⑥ 断言<br>⑦ 参数化（CSV）<br>⑧ 逻辑控制器 + 事务控制器 | 📖 **④⑤ 见第四节<br>⑥⑦⑧ 见第六节** |
| **四、性能测试**（JMeter 主场） | ⑨ 设计压测场景<br>⑩ 读聚合报告<br>⑪ 找瓶颈<br>⑫ 写性能测试报告 | 📖 **见手册第五节** |
| **五、工程化** | ⑬ 命令行执行 + HTML 报告<br>⑭ 分布式压测（了解）<br>⑮ 接 CI | ⬜ |

### 为什么值得学

Postman 和 pytest 都已做过，但它们**做不了并发压测**。
JMeter 补上的正是「性能」这块拼图 —— 功能、自动化、性能三条线就都跑通了。

---

## 二、第①步：安装配置

### 2.1 前置条件（本机实测结果）

| 项 | 实际情况 |
|---|---|
| JAVA_HOME | `E:\develop\jdk` · **JDK 24.0.1** |
| 备用 JDK | `E:\develop\IntelliJ IDEA 2025.1.3\jbr` · **JDK 21.0.7**（IDEA 自带） |
| JMeter | 未安装 |
| 磁盘 | C: 9.3 GB ／ D: 65.3 GB ／ E: 13.8 GB |

### ⭐ 关键结论：JDK 24 可以直接用（已实测，不用换 JDK）

2026-10-15 用命令行实测：

```
> "E:\develop\jmeter\bin\jmeter.bat" --version
退出码: 0        耗时: 2.8 秒

     _    ____   _    ____ _   _ _____       _ __  __ _____ _____ _____ ____
    / \  |  _ \ / \  / ___| | | | ____|     | |  \/  | ____|_   _| ____|  _ \
   / _ \ | |_) / _ \| |   | |_| |  _|    _  | | |\/| |  _|   | | |  _| | |_) |
  / ___ \|  __/ ___ \ |___|  _  | |___  | |_| | |  | | |___  | | | |___|  _ <
 /_/   \_\_| /_/   \_\____|_| |_|_____|  \___/|_|  |_|_____| |_| |_____|_| \_\ 5.6.3
```

**结论：不需要换 JDK，也不需要装 JDK 17。** 你的 JDK 24 直接跑通。

输出里这些是常规提示，**无害，每次启动都会有**：

| 提示 | 说明 |
|---|---|
| `WARN StatusConsoleListener The use of package scanning to locate plugins is deprecated` | log4j2 的提示，JMeter 5.6 每次启动都有 |
| `WARNING: Could not create windows registry node Software\JavaSoft\Prefs` | Java Preferences 写注册表失败，不影响功能 |

> 8.1 节的备用 JDK 方案**留着**，万一后面压测遇到问题再用。

### ⭐ 进一步实测：完整跑通一次测试计划（非 GUI + HTML 报告）

光有 `--version` 不算数，用一个手写的测试计划**真跑了一遍**：

```
> jmeter -n -t "plans\01-captchaImage.jmx" -l result.jtl -j jmeter-run.log -e -o report

Created the tree successfully using E:\学习\1\jmeter\plans\01-captchaImage.jmx
Starting standalone test @ 2026 Oct 15 18:38:40 GMT+08:00
summary =      1 in 00:00:00 =    8.9/s Avg:    28 Min:    28 Max:    28 Err:     1 (100.00%)
... end of run
退出码: 0        耗时: 5 秒
```

**产物：**

| 文件 | 说明 |
|---|---|
| `plans\result.jtl` | 结果原始数据（CSV，后面分析要用） |
| `plans\report\index.html` | **HTML 仪表盘报告** —— 双击就能在浏览器打开 |
| `plans\jmeter-run.log` | 运行日志 |

> 那次 `Err: 1 (100.00%)` 是因为**若依没启动**，连接被拒 —— **这是预期的**。
> 重点是整条链路跑通了。

### 三个已验证的结论

| 结论 | 证据 |
|---|---|
| **JDK 24 完全可用** | 不只是 `--version`，完整的测试计划执行链路都通 |
| **中文路径没问题** | 计划放在 `E:\学习\1\jmeter\plans\`，正常解析并执行 |
| **中文不乱码** | jtl 里的线程组名 `线程组 - 1 个用户` 完整正确 |

> ⚠️ **最后一条要注意**：`jmeter.properties` 第 1105 行的
> `#sampleresult.default.encoding=UTF-8` **默认是注释掉的**。
> 在 JDK 24 上没问题（Java 18+ 默认 UTF-8 编码），
> **但如果你以后换回 JDK 8/11，中文就会乱码** —— 到时候把那个 `#` 去掉。

### 已经有一份可用的参考计划

`plans\01-captchaImage.jmx` —— 手写的、验证过能跑的最小测试计划。

> 这是我建好并跑通的最小示例，可以对照着自己再建一遍，
> 建完可以打开它对比，看看哪里不一样。

### 关于 GUI：日志显示启动链路是通的

我这边试着启动 GUI，进程活不过 30 秒，但 `jmeter.log` 留下了证据：

```
INFO  JMeter: Version 5.6.3
INFO  JMeter: java.version=24.0.1
INFO  JMeter: file.encoding=UTF-8
INFO  JMeter: java.awt.headless=null
INFO  JMeter: Available Processors =20
INFO  JMeterGuiLauncher: Setting LAF to: com.github.weisj.darklaf.DarkLaf...
                          ↑ 已经加载到 Swing 界面主题了
```

**它一路走到了 GUI 初始化，说明启动链路完全通。**
在我这里死掉是因为**沙箱进程无法创建桌面窗口**，不是 JMeter 或 JDK 的问题。

> **结论：GUI 需要你自己双击验证。但日志已经强烈证明它没问题。**

### 特别注意：旧窗口读不到新配的环境变量

如果配完环境变量后还报 `jmeter 不是内部或外部命令`，
**99% 是因为你还在用配之前的那个窗口。**

**关掉它，重新开一个。**

### 2.2 下载

**版本：JMeter 5.6.3**（官方最新，要求 Java 8+）

Windows 用 **`.zip`** 包：

```
https://mirrors.tuna.tsinghua.edu.cn/apache/jmeter/binaries/apache-jmeter-5.6.3.zip
```

备用官方源：

```
https://dlcdn.apache.org/jmeter/binaries/apache-jmeter-5.6.3.zip
```

> ⚠️ 别下成 `apache-jmeter-5.6.3_src.zip` —— 那是源码，要自己编译。

### 2.3 解压

**目标路径：`E:\develop\jmeter`**

| 铁律 | 原因 |
|---|---|
| 路径**不能有中文** | 启动脚本 + 日志编码都会出问题 |
| 路径**不能有空格** | `C:\Program Files` 这类会出问题 |
| **别装 C 盘** | C 盘只剩 9.3 GB |

解压后确认结构（`bin` 要直接在 `jmeter\` 下面）：

```
E:\develop\jmeter\
├── bin\jmeter.bat          ← Windows GUI 启动
├── bin\jmeter.properties   ← 配置文件（改语言、改编码在这）
├── lib\                    ← 依赖
└── lib\ext\                ← 第三方插件放这
```

> 有些解压工具会多套一层 `apache-jmeter-5.6.3\`，把里面内容移到 `jmeter\` 下。

### 2.4 配置环境变量

**方式一：图形界面**

`Win + R` → `sysdm.cpl` → 高级 → 环境变量

| 位置 | 操作 |
|---|---|
| 用户变量 | 新增 `JMETER_HOME` = `E:\develop\jmeter` |
| 用户变量 Path | 新增一行 `%JMETER_HOME%\bin` |

**方式二：PowerShell**（我用不了图形界面，这个也行）

```powershell
[Environment]::SetEnvironmentVariable('JMETER_HOME','E:\develop\jmeter','User')
$p = [Environment]::GetEnvironmentVariable('Path','User')
[Environment]::SetEnvironmentVariable('Path', "$p;%JMETER_HOME%\bin", 'User')
```

**⚠️ 配完必须新开一个命令行窗口**，旧窗口读不到新变量。

### 2.5 验证

```powershell
jmeter --version
```

最终的判断标准 —— **GUI 能不能起来**：

```powershell
cd E:\develop\jmeter\bin
.\jmeter.bat
```

首次启动要 10~30 秒（加载插件），**别以为卡死了**。

### 2.6 一键自检

装完跑这个（本手册同目录）：

```powershell
powershell -ExecutionPolicy Bypass -File "E:\学习\1\jmeter\check_jmeter_env.ps1"
```

它会检查：安装路径 / 环境变量 / Java 版本兼容性 / 若依后端状态，并告诉你下一步做什么。

### 2.7 界面语言

**建议用英文界面**，理由：

| | 中文界面 | 英文界面 |
|---|---|---|
| 上手 | 快 | 稍慢 |
| 查资料 | ❌ 网上教程、报错、文档全是英文术语 | ✅ 直接对得上 |
| 日常沟通 | ❌ 说「线程组」「取样器」容易对不上 | ✅ 通用英文术语 |

切换方式：

```
临时：Options → Choose Language → Chinese (Simplified)
永久：编辑 bin\jmeter.properties，把 #language=en 改成 language=zh_CN
```

---

## 三、第②③步：第一个测试计划

> 目标：在 JMeter 里对若依跑通 `GET /captchaImage`，并理解 5 个核心组件。

### 3.0 先认识 5 个核心组件

| JMeter 组件 | 中文 | 你 Postman 里对应什么 | 一句话理解 |
|---|---|---|---|
| Test Plan | 测试计划 | 整个集合（Collection） | 一本书 |
| Thread Group | 线程组 | Runner 里的迭代设置 | **一群虚拟用户**（压测靠它） |
| Sampler | 取样器 | 一个具体的 Request | 一次请求 |
| Config Element | 配置元件 | 集合变量、集合级鉴权 | 一处配置、多处生效 |
| Listener | 监听器 | 响应面板 + Test Results | 看结果的地方 |

**层级关系（必须先懂 —— JMeter 是自顶向下建树的）：**

```
Test Plan（测试计划）
└── Thread Group（线程组）          ← 所有请求必须挂在它下面
    ├── Config Element              ← 只对「同级」和「下级」生效
    ├── Sampler（HTTP Request）
    │    └── Assertion              ← 断言挂在「某个请求」下面
    └── Listener                    ← 结果也挂在下面
```

> ⚠️ **和 Postman 最大的思维差异**：
> Postman 是「先有请求，再组织成文件夹」；
> JMeter 是「**先建容器，再往里放东西**」—— **不先建线程组，你连请求都加不进去。**

### 3.1 开始之前：把被测系统起起来

```powershell
# 窗口 1 —— Redis（必须先起，若依要连它）
cd E:\Redis\Redis-8.10.2-Windows-x64-msys2
.\redis-server.exe redis.conf
```

```powershell
# 窗口 2 —— 若依后端
cd E:\学习\RuoYi-Vue
java -jar ruoyi-admin\target\ruoyi-admin.jar
```

验证（第三个窗口）：

```powershell
curl.exe -s http://localhost:8080/captchaImage
# 期望看到: {"msg":"操作成功","code":200,"captchaEnabled":false}
```

### 3.2 第一步：加线程组

**右键 `Test Plan` → Add → Threads (Users) → Thread Group**

三个核心参数（**这就是后面压测要调的旋钮**）：

| 字段 | 中文 | 含义 | 现在填 |
|---|---|---|---|
| Number of Threads (users) | 线程数 | **虚拟用户数** | `1` |
| Ramp-Up Period (seconds) | 加压时间 | 多少秒内把这些用户启动完 | `1` |
| Loop Count | 循环次数 | 每个用户跑几遍 | `1` |

> **理解一下**：线程数 100 / 加压 10 秒 = 每秒启动 10 个用户。
> 第⑨步设计压测场景，调的就是这三个数。

### 3.3 第二步：加 HTTP 请求默认值

**右键 `Thread Group` → Add → Config Element → HTTP Request Defaults**

| 字段 | 填 |
|---|---|
| Protocol | `http` |
| Server Name or IP | `127.0.0.1` |
| Port Number | `8080` |

**为什么先加它**：后面每加一个请求都不用重复填域名端口 ——
**等价于 Postman 的环境变量 `{{baseUrl}}`。**

### 3.4 第三步：加 HTTP 请求

**右键 `Thread Group` → Add → Sampler → HTTP Request**

| 字段 | 填 |
|---|---|
| Name | `GET /captchaImage` |
| Method | `GET` |
| Path | `/captchaImage` |

（域名端口不用填，上面的「请求默认值」会自动补上。）

### 3.5 第四步：加监听器

**右键 `Thread Group` → Add → Listener → View Results Tree**
再加一个 **Aggregate Report**

| 监听器 | 中文 | 干什么 |
|---|---|---|
| View Results Tree | 查看结果树 | 看每个请求的**请求头 / 响应体**（调脚本用） |
| Aggregate Report | 聚合报告 | 看**平均值 / 中位数 / 错误率**（压测用） |

> ⚠️ **两个都加，但用途不同** ——
> 调试阶段用「查看结果树」；**压测阶段只留「聚合报告」**，
> 因为结果树会把每个响应都存进内存，压测时开着会拖垮 JMeter 自己。

### 3.6 第五步：保存 + 跑

**先保存**：`Ctrl + S` → 存到 `E:\学习\1\jmeter\plans\`，命名 `01-第一个请求.jmx`

**再跑**：点工具栏的**绿色三角形 ▶**

跑完在「查看结果树」里点一下那个请求，右侧能看到：
- 请求头 / 请求体
- **响应数据**（应该看到 `{"msg":"操作成功","code":200,"captchaEnabled":false}`）
- 响应码 `200`

### 3.7 第六步：加响应断言

**右键 `GET /captchaImage` → Add → Assertions → Response Assertion**

| 字段 | 填 |
|---|---|
| Name | `响应断言 - 含 code:200` |
| Field to Test | `Response Data` |
| Pattern Matching Rules | 勾 **Contains** |
| Patterns to Test | `"code":200` |

**等价于你 Postman 里的：**

```js
pm.test("业务 code = 200", () => pm.expect(json.code).to.eql(200));
```

> **JMeter 断言的本质**：它**不做 JSON 解析**（那是 JSON Assertion 的事），
> 就是**在响应文本里找字符串**。简单粗暴，但够用。

### 3.8 对照验收

建完打开参考计划对比一下：

```
E:\学习\1\jmeter\plans\01-captchaImage.jmx
```

（Test Plan 上右键 → Open，选它。）

| 对比项 | 你应该建的 | 参考计划里 |
|---|---|---|
| 线程组 | 1 线程 / 1 秒 / 1 循环 | 同 |
| 请求默认值 | 127.0.0.1 : 8080 / http | 同 |
| 请求 | GET `/captchaImage` | 同 |
| 断言 | Contains `"code":200` | 同 |

**基本一致 = 你建对了。**

### 3.9 自查清单

```
□ 1. Test Plan 下有一个 Thread Group
□ 2. Thread Group 下有 HTTP Request Defaults（填了 127.0.0.1:8080）
□ 3. Thread Group 下有一个 HTTP Request（GET /captchaImage）
□ 4. 加了 View Results Tree 和 Aggregate Report 两个监听器
□ 5. 跑起来响应码 200，响应体里有 "code":200
□ 6. 加了 Response Assertion，跑完断言不报错
□ 7. 保存成 .jmx 文件
```

---

## 四、第④⑤步：测试计划骨架 + token 关联（JMeter 最核心的一章）

> 目标：把 Postman 里的「集合级自动登录」搬到 JMeter ——
> 先调登录拿 token，再把 token 传给后续所有需要鉴权的请求。
> **这一章学会了，JMeter 就入门了。**

### 4.0 先看懂两种做法的心智差异

| 环节 | Postman 怎么做 | JMeter 怎么做 |
|---|---|---|
| 拿 token | Tests 脚本 `pm.environment.set("token", json.token)` | **JSON 提取器**（元件，不写代码） |
| 变量语法 | `{{token}}` | `${token}` |
| 带上 token | 集合级 Authorization | **HTTP 信息头管理器** |
| 保证顺序 | Pre-request 脚本里判断 | **靠树的上下顺序**（根本不需要脚本） |

> **一句话**：**Postman 靠脚本，JMeter 靠元件 + 树的顺序。**
> 所以 JMeter 里你会看到很多"设置面板"，但几乎没有代码 —— 这就是它的设计哲学。

### 4.1 第一步：加 HTTP 信息头管理器

**右键 `Thread Group` → Add → Config Element → HTTP Header Manager**

添加一行：

| Name | Value |
|---|---|
| `Content-Type` | `application/json;charset=UTF-8` |

> **为什么需要**：若依的登录接口要求 `application/json`，
> 不设的话服务端收不到参数，会返回"用户名或密码错误"。
> 这一条等价于 Postman 里 Body 选 `raw → JSON`。

### 4.2 第二步：加 HTTP Cookie 管理器

**右键 `Thread Group` → Add → Config Element → HTTP Cookie Manager**

**保持默认，什么都不用填。**

> 作用：自动存取 Cookie（JSESSIONID 等）。
> 若依用 JWT 主要靠请求头，但加上它以后遇到依赖 Cookie 的接口不会莫名失败。

### 4.3 第三步：加登录请求

**右键 `Thread Group` → Add → Sampler → HTTP Request**

| 字段 | 填 |
|---|---|
| Name | `POST /login` |
| Method | `POST` |
| Path | `/login` |
| Body Data | 见下面 |

**切到 `Body Data` 页签，粘贴：**

```json
{
  "username": "admin",
  "password": "admin123",
  "code": "",
  "uuid": ""
}
```

> `code` 和 `uuid` 留空是因为**你已经把若依的验证码关掉了**。
> 如果哪天验证码又开了，这里就会返回"验证码错误" —— 顺便记一下：
> 「自动化遇到验证码拦截，我的处理是关掉测试环境验证码 / 或者走白名单接口」。

### 4.4 第四步（关键）：用 JSON 提取器把 token 抽出来

**右键 `POST /login` → Add → Post Processors → JSON Extractor**

| 字段 | 填 |
|---|---|
| Name | `提取 token` |
| **Names of created variables** | `token` |
| **JSON Path expressions** | `$.token` |
| Match No. (0 for Random) | `1` |
| Compute concatenation var | 不勾 |
| **Default Values** | `TOKEN_NOT_FOUND` |

> ✅ **我用 5.6.3 的 jar 包实际验证过**：JMeter 内置了**两个** JSON 提取器，**都不用装插件**：
>
> | 菜单名 | 语法 | 表达式写法 |
> |---|---|---|
> | **JSON Extractor** ← 本篇用这个 | JSONPath | `$.token` |
> | JSON JMESPath Extractor | JMESPath | `token` |
>
> **为什么用 JSON Extractor**：JSONPath 是主流写法，
> 你学会以后查资料能直接对上。

> 💡 **`Default Values` 是个很重要的习惯**：提取失败时变量会变成 `TOKEN_NOT_FOUND`，
> 而不是空字符串。这样后续请求报 401 时，你一眼就能看出"是提取没成功"，
> 而不是对着空白请求头猜半天。

### 4.5 第五步：发一个需要鉴权的请求

**右键 `Thread Group` → Add → Sampler → HTTP Request**

| 字段 | 填 |
|---|---|
| Name | `GET /getInfo` |
| Method | `GET` |
| Path | `/getInfo` |

**然后给它单独加一个信息头管理器**（注意是加在这个请求下面）：

**右键 `GET /getInfo` → Add → Config Element → HTTP Header Manager**

| Name | Value |
|---|---|
| `Authorization` | `Bearer ${token}` |

> **这里就是 JMeter 的「关联」**：
> - `${token}` 是 **JMeter 的变量引用语法**
> - `token` 这个名字来自 4.4 步提取器里填的 `Names of created variables`
> - 两个名字**必须一模一样**，写错一个字母就传不进去

> 📌 **作用域提醒**：元件对「同级 + 下级」生效。
> 所以这个 Authorization 头**只对 `/getInfo` 生效**，不会污染 `/login`。
> 如果放到线程组层级，登录请求也会带上它（虽然无害，但不规范）。

### 4.6 第六步：跑起来验证关联是否成功

点绿色三角形 ▶，然后在「查看结果树」里逐条检查：

| # | 点什么 | 应该看到 |
|---|---|---|
| 1 | `POST /login` | Response Data 里有 `"token":"eyJ..."` |
| 2 | `GET /getInfo` → 展开 **Request → Request Headers** | `Authorization: Bearer eyJ...`（**能展开看到真实值就说明关联成功**） |
| 3 | `GET /getInfo` 的响应码 | `200` |

**如果第 3 步是 401**，按这三条排查：

| 可能原因 | 怎么确认 |
|---|---|
| 变量名不一致 | 提取器里写 `token`，头里写 `${token}` —— 大小写也要一致 |
| 提取器的表达式错了 | 在 `/login` 的响应里肉眼确认字段名就叫 `token`（不是 `data.token`） |
| 请求头名字拼错 | 必须是 `Authorization`（不是 `Authorizaton`、`Auth`） |

### 4.7 顺序问题：为什么不用像 Postman 那样写脚本

**JMeter 按树中从上到下的顺序执行同一层级的元件。**

```
Thread Group
├── HTTP Header Manager        ← 配置元件先"登记"
├── HTTP Cookie Manager
├── POST /login                ← 先执行
│    └── JSON JMESPath Extractor  ← 登陆完立刻提取
└── GET /getInfo               ← 后执行，此时 ${token} 已经有值了
     └── HTTP Header Manager
```

**所以只要保证 `POST /login` 在 `GET /getInfo` 上面就行** —— 添加顺序天然满足。
这就是为什么 JMeter 里**不需要写"如果 token 过期就重新登录"那种脚本**：
顺序由树决定，提取由元件完成。

### 4.8 自查清单

```
□ 1. Thread Group 下有 HTTP Header Manager（Content-Type: application/json）
□ 2. Thread Group 下有 HTTP Cookie Manager
□ 3. 第一个请求是 POST /login，Body Data 填了用户名密码 JSON
□ 4. /login 下面挂了 JSON Extractor，变量名 = token，表达式 = $.token
□ 5. 第二个请求是 GET /getInfo，且它下面挂了 HTTP Header Manager
□ 6. 头里写的是 Authorization : Bearer ${token}
□ 7. 跑完：/login 返回 200 且响应里有 token
□ 8. 跑完：/getInfo 返回 200，且 Request Headers 里能看到真实 Bearer 值
```

> 8 条全绿，说明你已经掌握了 JMeter 的关联。

---

## 五、第⑨～⑫步：性能测试（JMeter 真正的主场）

> 前面几章做的事，Postman 和 pytest 也能做。
> **这一章才是 JMeter 存在的理由** —— 它能把「并发」这件事做出来。

### 5.0 先搞清「接口测试」和「性能测试」的区别

| | 接口功能测试 | 性能测试 |
|---|---|---|
| 回答什么问题 | **对不对** | **快不快、扛不扛得住** |
| 线程数 | 1 | 几十 ~ 几千 |
| 关注指标 | 断言是否通过 | **TPS / 响应时间 / 错误率** |
| 你已有的工具 | Postman、pytest | ← **JMeter** |
| 产出 | 用例、缺陷报告 | **性能测试报告** |

### 5.1 第一步：把线程组改成「持续时间」模式

功能测试时线程组填的是「循环次数」，压测要改成「压多久」。

| 字段 | 功能测试填 | **压测填** |
|---|---|---|
| Number of Threads | 1 | **20**（并发用户数） |
| Ramp-Up Period | 1 | **5**（5 秒内爬升到 20 个） |
| Loop Count | 1 | **勾上 `Infinite`**（一直跑） |
| **Scheduler** | 不勾 | ✅ **勾上** |
| **Duration (seconds)** | 空 | **20**（持续压 20 秒） |

> **为什么用「持续时间」而不是「循环次数」？**
> 压测要的是「在一段时间内系统表现如何」。
> 用固定循环次数的话，跑得快就结束得早、跑得慢就拖很久，样本量不可控。

> **Ramp-Up 为什么要设 5 秒而不是 0？**
> 瞬间 20 个用户同时上，是"冲击"；5 秒爬升，更接近真实用户陆续进来的样子。
> 想测"突发流量"就把 Ramp-Up 设小甚至设成 0。

### 5.2 第二步：加两个性能专用元件

**① 响应时间断言**（挂在每个请求下面）

**右键请求 → Add → Assertions → Duration Assertion**

| 字段 | 填 |
|---|---|
| Duration in milliseconds | `1000` |

> 意思：**这个请求超过 1 秒就判失败**。
> 这是性能测试里最常用的断言 —— 功能对但太慢，一样是不可接受的。

**② 聚合报告**（挂在线程组下面）

**右键 Thread Group → Add → Listener → Aggregate Report**

### 5.3 第三步：⚠️ 压测必须用命令行（JMeter 第一铁律）

**GUI 模式本身消耗大量资源，会严重污染压测结果。**

```powershell
jmeter -n -t "plans\03-压测演练.jmx" -l "plans\load.jtl" -e -o "plans\load-report"
```

| 参数 | 意思 |
|---|---|
| `-n` | **non-GUI，非图形模式**（压测必须加） |
| `-t` | test plan，指定 .jmx 文件 |
| `-l` | 结果数据文件（**.jtl**，CSV 格式） |
| `-e -o` | 生成 **HTML 仪表盘报告**到指定目录（目录必须不存在或为空） |

### 5.4 第四步：读结果 —— 聚合报告怎么读

**这是一次真实跑出来的结果**（20 并发 × 20 秒，被测对象是一个本地 Mock 服务）：

| 标签 | 样本 | 平均 | 中位 | **P90** | **P95** | **P99** | 最小 | 最大 | **错误率** | **TPS** |
|---|---|---|---|---|---|---|---|---|---|---|
| POST /login | 79170 | 2 | 2 | 4 | 5 | 6 | 0 | 24 | **0.00%** | **3966.6** |
| GET /getInfo | 79162 | 2 | 2 | 4 | 5 | 6 | 0 | 14 | **0.00%** | **3966.2** |
| **合计** | **158332** | 2 | 2 | 4 | 5 | 6 | 0 | 24 | **0.00%** | **7932.9** |

**每个指标什么意思：**

| 指标 | 含义 | 怎么用 |
|---|---|---|
| 样本数 | 总请求数 | 太小（<100）说明压得不够，结论不可信 |
| 平均 | 平均响应时间 | ⚠️ **别只看它** —— 几个慢请求就能把平均值拉偏 |
| 中位 | 50% 的请求快于此值 | 比平均更能代表"典型体验" |
| **P90 / P95 / P99** | 90% / 95% / 99% 的请求快于此值 | ★ **真正该看的** —— 它反映"最差的那批用户"有多差 |
| 最小 / 最大 | 极值 | 最大特别大 = 有偶发卡顿 |
| **错误率** | 失败请求占比 | 一般要求 < 0.1% |
| **TPS** | 每秒处理请求数（吞吐量） | ★ **压测的核心产出** |

> 💡 **为什么 P95 比平均重要**：
> 假设 100 个用户里 95 个 20ms、5 个 3 秒。平均只有 ~170ms 看着挺好，
> 但那 5 个人已经想砸键盘了。**P95/P99 才是用户体验的真相。**

### 5.5 第五步：三个核心指标怎么判断好坏

| 指标 | 参考标准 | 不达标说明什么 |
|---|---|---|
| **错误率** | < 0.1% | 系统有崩溃/超时/连接被拒 |
| **P95 响应时间** | 视业务而定（页面接口一般 < 1s） | 慢查询、锁竞争、连接池不够 |
| **TPS** | 看**趋势**，不看绝对值 | 并发翻倍 TPS 却没涨 → **已经到瓶颈了** |

**最关键的一句话**：

> **TPS 不随并发增长的那个点，就是系统的容量上限。**

### 5.6 第六步：用「阶梯加压」找瓶颈（这才是性能测试的正戏）

**做法**：固定持续时间（比如 30 秒），**逐步提高并发数**，每轮记一次数据。

| 并发数 | TPS | P95 | 错误率 | 观察 |
|---|---|---|---|---|
| 10 | 1200 | 12ms | 0% | 正常 |
| 20 | 2400 | 15ms | 0% | **TPS 翻倍 ≈ 线性** ✅ |
| 50 | 5800 | 40ms | 0% | 还在涨 |
| 100 | **6100** | 180ms | 0.2% | ⚠️ **TPS 几乎不涨了，响应时间跳升** |
| 200 | **5200** | 900ms | 8% | ❌ **TPS 反而下降，错误率飙高** |

**结论怎么写**：

> 「系统在 **50 并发**以内 TPS 随并发线性增长；到 **100 并发**出现瓶颈，
> TPS 停止增长且 P95 从 40ms 跳到 180ms；**200 并发时系统过载**，TPS 下降、错误率 8%。
> **建议容量上限为 50~80 并发**，需优化 {慢查询 / 连接池 / 缓存} 后复测。」

**这段话就是性能测试报告的核心价值。**

### 5.7 第七步：性能测试报告模板

```markdown
# {系统名} 性能测试报告

## 一、测试概述
- 被测系统：RuoYi-Vue v3.9.2
- 测试目的：验证核心接口在 {N} 并发下的处理能力与稳定性
- 测试时间：{日期}
- 测试环境：{CPU} / {内存} / JDK {版本} / MySQL {版本} / JMeter 5.6.3

## 二、测试场景
| 场景 | 接口 | 并发数 | 加压方式 | 持续时间 | 说明 |
|---|---|---|---|---|---|
| 登录 | POST /login | 10/20/50/100 | 5 秒爬升 | 30 秒 | 阶梯加压 |
| 获取用户信息 | GET /getInfo | 同上 | 同上 | 30 秒 | 需先登录拿 token |

## 三、测试结果
| 并发 | 接口 | 样本数 | 平均 | P95 | P99 | 错误率 | TPS |
|---|---|---|---|---|---|---|---|
| 10 | POST /login | ... | ... | ... | ... | ... | ... |
| ... | ... | ... | ... | ... | ... | ... | ... |

## 四、结果分析
- TPS 随并发的增长趋势（线性 / 提前饱和 / 下降）
- 响应时间拐点出现在哪个并发数
- 错误率的来源（超时？连接被拒？业务报错？）

## 五、结论与建议
### 结论
- 系统在 {X} 并发以内运行稳定，TPS 达到 {Y}，P95 {Z}ms
- 容量上限约 {X} 并发，超出后 {现象}

### 建议
1. {具体优化点，如：给 xxx 字段加索引 / 调大连接池 / 加缓存}
2. {扩容或限流建议}

## 六、附件
- JMeter 脚本：plans/03-压测演练.jmx
- 原始数据：load.jtl
- HTML 报告：load-report/index.html
```

> ⚠️ **写报告的红线**：
> **所有数字都必须是自己真跑出来的。** 性能数据一旦是编的，
> 被问到"环境什么配置""怎么找的瓶颈"就答不上，而且数据往往自相矛盾。

### 5.8 自查清单

```
□ 1. 线程组改成 Scheduler + Duration 模式（不是循环次数）
□ 2. 每个请求下面挂了 Duration Assertion
□ 3. 线程组下面挂了 Aggregate Report
□ 4. 用命令行 -n 跑（不是 GUI）
□ 5. 生成了 .jtl 结果文件
□ 6. 用 -e -o 生成了 HTML 仪表盘报告
□ 7. 能读出这张表：样本 / 平均 / P95 / P99 / 错误率 / TPS
□ 8. 做过阶梯加压（至少 3 个并发档位）
□ 9. 能指出瓶颈出现在哪个并发数
```

---

## 六、第⑥⑦⑧步：断言体系 / CSV 参数化 / 逻辑控制器

> 这一节对应路线图里的第⑥⑦⑧步。即使不做性能测试，**这一节也必须掌握 ——
> 它决定你的用例「测得准不准」。**

### 6.0 断言的作用域（很多人栽在这）

| 断言挂在哪 | 作用范围 |
|---|---|
| 挂在**某个请求**下面 | 只对这个请求生效 |
| 挂在**线程组**下面 | 对组内**所有请求**生效 |

**建议**：
- **公共断言**（如「HTTP 状态码 = 200」）→ 挂线程组
- **业务断言**（如「`$.code` = 200」「`$.total` > 0」）→ 挂具体请求

> ⚠️ 断言"挂了但没生效"，90% 是**挂错层级**了。

### 6.1 断言一：JSON 断言（★ 首选）

**右键请求 → Add → Assertions → JSON Assertion**

| 字段 | 填 |
|---|---|
| Assert JSON Path exists | `$.code` |
| Additionally assert value | ✅ 勾上 |
| Expected Value | `200` |
| Match as regular expression | 不勾 |

**为什么首选它**：它**先把响应解析成 JSON、按路径取值、再比较值** ——
**不受空格、换行、字段顺序影响**。

**实测教训（我真踩过的坑）**：

| 断言方式 | 面对 `{"code": 200}`（冒号后有空格） |
|---|---|
| Response Assertion 找 `"code":200` | ❌ **失败** —— 纯字符串匹配，空格不同就不认 |
| **JSON Assertion** `$.code` = `200` | ✅ **通过** —— 解析成 JSON 后比的是值 |

> 若依用 Jackson 输出的是紧凑 JSON（无空格），所以字符串断言"碰巧能用"。
> 但换成任何格式不同的服务，它立刻就挂。**能用 JSON 断言就别用字符串断言。**

### 6.2 断言二：响应断言（字符串匹配）

**右键请求 → Add → Assertions → Response Assertion**

| 字段 | 填 |
|---|---|
| Field to Test | `Response Data`（响应体）/ `Response Code`（状态码） |
| Pattern Matching Rules | `Contains`（包含） |
| Patterns to Test | 要匹配的字符串 |

**什么时候用它**：
- 响应**不是 JSON**（HTML、XML、纯文本）
- 只想粗筛（比如响应里必须出现某个业务关键字）

### 6.3 断言三：响应时间断言（Duration Assertion）

已在 **5.2 节（性能测试）** 讲过 —— 超过指定毫秒数就判失败。**压测必加。**

### 6.4 断言失败长什么样（很重要）

**断言失败不会让 HTTP 请求失败**（状态码可能还是 200），
而是把该**请求**标记为 `success=false`，并在 `failureMessage` 里写你填的自定义消息。

> **所以看结果时不要只看 HTTP 状态码** —— 要看 `success` 和断言消息。
> 这正是我上一轮把 `success=false` 当成"请求失败"的原因。

### 6.5 第⑦步：CSV 参数化

**场景**：同一套流程，换不同账号 / 数据跑多遍。

**① 准备数据文件**（`data/users-mock.csv`）：

```csv
username,password
admin,admin123
ry,admin123
test,test123
```

**② 加 CSV Data Set Config**

**右键 Thread Group → Add → Config Element → CSV Data Set Config**

| 字段 | 填 | 说明 |
|---|---|---|
| Filename | `E:\学习\1\jmeter\data\users-mock.csv` | **建议绝对路径** |
| File encoding | `UTF-8` | 有中文数据必须 |
| Variable Names | `username,password` | 和 CSV 表头对应 |
| Delimiter | `,` | |
| Ignore first line | ✅ 勾上 | 跳过表头 |
| Recycle on EOF | ✅ 勾上 | 数据用完了从头再来 |
| Stop thread on EOF | 不勾 | |
| Sharing mode | `All threads` | 所有线程共享一份 |

**③ 请求里用变量**

Body Data 把写死的值换成 `${变量名}`：

```json
{"username":"${username}","password":"${password}","code":"","uuid":""}
```

**④ 循环次数要 ≥ 数据行数**

3 行数据 → 循环次数设 `3`。

### 6.6 参数化实测结果（已验证）

**1 线程 × 3 循环 × 2 个请求 = 6 个请求，全部通过，错误率 0.00%。**

Mock 服务侧看到的账号：

```
[MOCK] /login 成功 user=admin
[MOCK] /login 成功 user=ry
[MOCK] /login 成功 user=test       ← 三个不同账号，参数化确实生效了
```

**我还加了一条「自我证明」的断言**：

```
$.token = MOCK-TOKEN-${username}
```

**如果 CSV 变量没注入进去，这条断言就会失败。它通过了 → 参数化确实生效。**

> 💡 **这是个值得偷的技巧**：写断言时**把变量自身也带上**，
> 这样"参数化没生效"会被断言直接抓出来，而不是靠你肉眼比对。

### 6.7 第⑧步：逻辑控制器（够用就行）

**① 事务控制器（Transaction Controller）★ 最有用**

**右键 Thread Group → Add → Logic Controller → Transaction Controller**

把它下面的多个请求**合并成一个事务**。压测时特别重要：

| 不加事务控制器 | 加了之后 |
|---|---|
| 报告里 `POST /login`、`GET /getInfo` 各有各的 TPS | 看到 `一整套登录流程` 一个 TPS |
| **不知道"完整业务流程"能跑多快** | ✅ 知道 |

> 勾上 `Generate parent sample` 后，报告里只显示事务的整体耗时。

**② 循环控制器（Loop Controller）**

把一组请求循环跑 N 次。和线程组的循环次数区别：
线程组循环 = 整个线程组跑 N 遍；循环控制器 = 只循环它下面那部分。

**③ If 控制器（If Controller）**

按条件决定跑不跑下面的请求（比如「只有提取到 token 才继续」）。
**但 JMeter 的思路是靠树的顺序，不是靠 if** —— 能用顺序解决就别用 if。

### 6.8 自查清单

```
□ 1. 每个请求都有 JSON 断言（$.code = 200）
□ 2. 清楚断言挂「请求下」和挂「线程组下」的区别
□ 3. CSV 文件用 UTF-8 编码 + 绝对路径
□ 4. CSV Data Set Config 勾了 Ignore first line
□ 5. 请求里用 ${变量名} 而不是写死的值
□ 6. 循环次数 ≥ CSV 行数
□ 7. 加了一条「带变量自身」的断言来验证参数化生效
□ 8. 用事务控制器把多请求流程合成一个事务
```

---

## 七、术语中英对照（看界面时对着看）

| 英文 | 中文 | 是什么 |
|---|---|---|
| Test Plan | 测试计划 | 脚本的根节点 |
| Thread Group | 线程组 | 一组虚拟用户（**并发靠它**） |
| Sampler | 取样器 | 一个具体请求，如 HTTP Request |
| Config Element | 配置元件 | 公共设置（请求默认值 / 请求头管理器） |
| Listener | 监听器 | 看结果（聚合报告 / 查看结果树） |
| Assertion | 断言 | 校验响应 |
| Pre / Post Processor | 前置 / 后置处理器 | 请求前后处理，**关联靠它** |
| Timer | 定时器 | 控制请求间隔（模拟思考时间） |
| Logic Controller | 逻辑控制器 | if / 循环 / 事务 |
| CSV Data Set Config | CSV 数据文件设置 | 参数化 |
| Transaction Controller | 事务控制器 | 把多个请求算作一个事务 |

---

## 八、排错手册

### 8.1 如果 JDK 24 不兼容

**症状对照：**

| 现象 | 判断 |
|---|---|
| 刷一堆 `WARNING: A restricted method in java.lang.System has been called` | ⚠️ 只是警告，**不影响使用** |
| 报 `UnsupportedClassVersionError` | ❌ 真不兼容 |
| 报 `Security Manager is disabled` 相关 | ❌ 真不兼容 |
| GUI 起来了但一执行就崩 | ❌ 真不兼容 |

**方案 A（推荐，不用下载）**：让 JMeter 单独用 IDEA 自带的 JDK 21。

编辑 `E:\develop\jmeter\bin\jmeter.bat`，在**最前面**加一行：

```bat
set "JAVA_HOME=E:\develop\IntelliJ IDEA 2025.1.3\jbr"
```

**方案 B**：装一个独立的 JDK 17（pytest 项目和很多工具都能用，一次投入长期受益）。

> 把完整报错发我，我给具体判断。

### 8.2 其他常见问题

| 现象 | 原因 | 解决 |
|---|---|---|
| **配了 PATH，新窗口还是「无法识别 jmeter」** | ⚠️ **`HKCU\Environment\Path` 的类型是 `REG_SZ`，Windows 不会展开里面的 `%JMETER_HOME%`** | 换成绝对路径，见下方「⚠️ 一个真实的坑」 |
| 双击 `jmeter.bat` 闪一下就没了 | 启动报错 | 在命令行里执行，才能看到报错信息 |
| GUI 字体特别小 | 高 DPI 屏幕 | 编辑 `bin\jmeter.properties`，试 `jmeter.hidpi.mode=true` |
| 启动特别慢 | 正常 | 首次加载插件，之后会快 |

### ⚠️ 一个真实的坑：`%JMETER_HOME%` 在某些情况下不会展开

**现象**：明明配了 `JMETER_HOME` 和 PATH，新开窗口敲 `jmeter` 还是「无法识别」。

**排查方法**（一行命令查出值的类型）：

```powershell
(Get-Item 'HKCU:\Environment').GetValueKind('Path')
```

| 返回值 | 含义 |
|---|---|
| `ExpandString` | ✅ `REG_EXPAND_SZ`，Windows **会**展开 `%VAR%` |
| `String` | ❌ `REG_SZ`，Windows **不会**展开，`%JMETER_HOME%\bin` 会被当成字面路径 |

**为什么会这样**：`Path` 的类型是**它第一次被创建时定下来的**。
如果之前用 `[Environment]::SetEnvironmentVariable()` 写过一次（那会写成 `REG_SZ`），
**后面即使用图形界面 `sysdm.cpl` 再改，类型也不会变**，仍然是 `REG_SZ`。

> **实测记录（2026-10-15）**：本机用图形界面加完 `%JMETER_HOME%\bin` 之后，
> `Path` 的类型**仍然是 `String`（REG_SZ）** —— 印证了上面这条。

**解决办法（推荐）—— 换成绝对路径**：

```powershell
$up = [Environment]::GetEnvironmentVariable('Path','User')
$new = $up.Replace('%JMETER_HOME%\bin', 'E:\develop\jmeter\bin')
[Environment]::SetEnvironmentVariable('Path', $new, 'User')
```

**然后必须关窗口重新开。**

**另一个办法 —— 把 PATH 改成可展开类型**（保留 `%JMETER_HOME%` 的写法）：

```powershell
$up = [Environment]::GetEnvironmentVariable('Path','User')
Set-ItemProperty -Path 'HKCU:\Environment' -Name 'Path' -Value $up -Type ExpandString
```

> 💡 **其实 PATH 不是必须的**：直接双击 `E:\develop\jmeter\bin\jmeter.bat` 也能开 GUI。
> PATH 只影响「在命令行里能不能直接敲 `jmeter`」。

### 8.3 ⚠️ JMeter 第一铁律

> **GUI 模式只用来「编辑脚本」，压测必须用命令行（non-GUI）。**
>
> 原因：GUI 本身消耗大量资源，会严重干扰压测结果。启动时弹出的那个提示就是在说这件事。

---

## 九、待补充章节

- [x] 第②③步：第一个 HTTP 请求（`GET /captchaImage`）→ **见第三节**
- [x] 第④⑤步：测试计划骨架 + token 关联 → **见第四节**
- [x] 第⑥⑦⑧步：断言 / CSV 参数化 / 控制器 → **见第六节**
- [x] 第⑨～⑫步：性能测试 → **见第五节**
- [x] 第⑬步：命令行执行 + HTML 报告 → **见 5.3 / 5.4**
- [ ] 第⑭⑮步：分布式压测 / 接 CI
