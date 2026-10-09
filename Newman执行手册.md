# Newman 执行手册（若依 RuoYi-Vue 接口自动化）

---

## 一、Newman 是什么，有什么用

**Newman = Postman 的命令行版本**，不是另一个测试工具，而是**同一个执行引擎**。
你在 Postman 里写的集合、断言、环境变量、脚本，Newman 原样能跑，一行都不用改。

### 为什么不能只用 Postman GUI

| 场景 | Postman GUI | Newman |
| --- | --- | --- |
| 每次提交代码后自动跑 | ❌ 做不到 | ✅ 流水线里一条命令 |
| 半夜定时跑 | ❌ 得有人开着电脑点 | ✅ 定时任务 |
| 部署到 Linux 构建服务器 | ❌ 装不了图形界面 | ✅ 一个 npm 包 |
| 报告自动归档 / 分享 | ❌ 在界面里，导不出来 | ✅ 生成 HTML 文件 |
| 给流水线返回成功/失败 | ❌ 没有退出码 | ✅ 失败返回非 0，流水线自动红灯 |
| 换台电脑接着跑 | ❌ 重装、重导入、重选文件 | ✅ 两个 JSON 文件就够 |

### 接口自动化的三个阶段

```
① 设计期 —— Postman GUI（人）
   写请求、设计用例、调断言、探索性测试
        ↓ 导出为 JSON（用例资产，可进 Git）
② 执行期 —— Newman（机器）
   批量执行、生成报告、返回退出码
        ↓ 被调用
③ 调度期 —— CI/CD（Jenkins / GitHub Actions）
   决定「什么时候跑」：每次提交？每天凌晨？发版前？
```

**Postman 管设计，Newman 管执行，CI 管调度。** 只有 Postman 的话，测试就停在
「人点一下才跑一次」，进不了流水线，也就不能叫「自动化」。

### 已有资产 100% 复用

| 已有资产 | Newman 怎么用 |
| --- | --- |
| 集合（请求 + 文件夹结构） | `newman run <集合文件>` |
| 环境变量（baseUrl / token） | `-e <环境文件>` |
| 断言 | 原样执行 |
| 自动登录 + 401 自愈脚本 | 原样执行 |
| `91 已知缺陷回归` | `--folder` 单独跑 |
| `data/users.csv` | `-d` 参数做数据驱动 |

---

## 二、目录结构

执行前先把项目目录搭成这样：

```
E:\学习\postman测试\
├── postman\                          ← 从 Postman 导出
│   ├── RuoYi-Vue.postman_collection.json
│   └── RuoYi-Local.postman_environment.json
├── data\
│   └── users.csv                     ← 参数化数据
├── report\                           ← HTML 报告输出
├── 证据\                             ← 缺陷证据截图
├── 缺陷报告-D1/D7/D9.md
├── 测试记录表.md
└── Newman执行手册.md                 ← 本文件
```

建目录的命令（在 PowerShell 里执行一次）：

```powershell
$root = 'E:\学习\postman测试'
foreach ($d in @('postman','data','report')) { New-Item -ItemType Directory -Force "$root\$d" | Out-Null }
Copy-Item 'E:\学习\1\postman\users.csv' "$root\data\users.csv" -Force
Get-ChildItem $root
```

---

## 三、安装 Newman

```powershell
npm install -g newman newman-reporter-htmlextra
```

`newman-reporter-htmlextra` 是生成好看 HTML 报告的报告器，**必须一起装**。

装完**新开一个终端**再验证（让 PATH 生效）：

```powershell
newman -v
```

能打出类似 `6.x.x` 的版本号就成功了。

---

## 四、从 Postman 导出

⚠️ 必须用**你 Postman 里的那份**（手动选的上传文件路径、`91 已知缺陷回归` 文件夹、
回归断言都在里面，我工作区那份是历史基线，不要用）。

**导出集合**

左侧 `Collections` → 右键集合 → `Export` → 格式选 `Collection v2.1` → `Export`
→ 保存到 `postman\RuoYi-Vue.postman_collection.json`

**导出环境**

左侧 `Environments` → 右键 `RuoYi 本地环境` → `Export`
→ 保存到 `postman\RuoYi-Local.postman_environment.json`

### ⚠️ 导出后必做：脱敏检查

Postman 会把环境变量的**当前值**一起导出，包括真实的 `token`。
用记事本打开环境文件搜一下 `token`：

| 情况 | 处理 |
| --- | --- |
| 值是 `eyJhbGci...` 这样的真实 JWT | **手动清空成 `""`** |
| 值是空的 | 没问题 |

> 将来要提交 Git，这一步不能省。任何要进版本库的文件，提交前先过一遍凭证检查。

---

## 五、执行

### 5.1 全量执行（日常用这个）

```powershell
cd 'E:\学习\postman测试'

newman run postman\RuoYi-Vue.postman_collection.json -e postman\RuoYi-Local.postman_environment.json --timeout-request 10000 -r "cli,htmlextra" --reporter-htmlextra-export report\ruoyi-api-report.html
```

**参数说明**

| 参数 | 作用 |
| --- | --- |
| `run <集合>` | 执行集合 |
| `-e <环境>` | 加载环境变量（缺了会大量 401） |
| `--timeout-request 10000` | 单请求超时 10 秒，防止卡死 |
| `-r "cli,htmlextra"` | 报告器：命令行输出 + HTML 文件 |
| `--reporter-htmlextra-export` | HTML 报告输出路径 |

### 5.2 预期结果

```
All 103  |  Passed 96  |  Failed 7  |  Skipped 0  |  Errors 0
```

- **96 通过** = 主套件全绿
- **7 失败** = `91 已知缺陷回归` 里的断言，对应缺陷 D1 / D7 / D9，**属预期红灯**
- **0 errors** = 没有环境问题、没有脚本报错

> **判定标准**：失败的 7 条必须都是 `[缺陷00X]` 开头的断言。
> 如果出现 401、超时、文件读不到之类的失败，说明是环境问题，**不是「已知缺陷」**。

### 5.3 只跑已知缺陷回归

```powershell
newman run postman\RuoYi-Vue.postman_collection.json -e postman\RuoYi-Local.postman_environment.json --folder "91 已知缺陷回归" -r cli
```

**预期：3 个请求、7 条断言全部失败，且都是 `[缺陷00X]`。**

### 5.4 带参数化执行（数据驱动）

```powershell
newman run postman\RuoYi-Vue.postman_collection.json -e postman\RuoYi-Local.postman_environment.json -d data\users.csv -n 3 -r "cli,htmlextra" --reporter-htmlextra-export report\ruoyi-api-report-data.html
```

| 参数 | 作用 |
| --- | --- |
| `-d data\users.csv` | 指定迭代数据文件 |
| `-n 3` | 迭代 3 次（用前 3 行数据） |

⚠️ **CSV 的列名必须和请求里的 `{{变量名}}` 对应上**才会生效，否则该变量为空。
例如 CSV 有 `userName` 列，请求体里就要写 `{{userName}}`。

---

## 六、看 HTML 报告

双击打开 `report\ruoyi-api-report.html`。

| 区域 | 看什么 |
| --- | --- |
| 顶部统计 | 总请求数、断言通过率、总耗时 |
| 每个请求的耗时 | **找出慢接口** |
| 失败的 7 条 | 确认是否都是已知缺陷 |
| 请求 / 响应原文 | 可直接作为证据截图 |

**实测观察**：`03 监控与运维 → 05 服务监控 GET /monitor/server` 耗时约 **1200 ms**，
其余接口都在 5~20 ms。原因是该接口要用 OSHI 采集 CPU / 内存 / 磁盘 / JVM 信息，
本身偏重，属正常现象，但值得记一笔。

> **自动化的附带价值就是能一眼看出哪个接口慢** —— 手工点的话你根本不会有这个概念。

**这个 HTML 文件就是交付物**：能发给别人看、能进 Git 仓库、能作为简历附件。

---

## 七、关于「7 条红灯」怎么处理

两种做法，**推荐第一种**。

### 方案 A（推荐）：接受已知失败，在报告里说明

全量跑永远是 `96 通过 / 7 失败`，这 7 条对应已提缺陷。
**CI 门槛设为「不允许新增失败」**，而不是「零失败」—— 这是真实项目的做法。

优点：诚实、可追溯，材料能自圆其说。

### 方案 B：把 91 提到顶层，组合执行出「全绿」报告

把 `91 已知缺陷回归` 从 `01 用户管理` 下移到**顶层**，然后：

```powershell
newman run postman\RuoYi-Vue.postman_collection.json -e postman\RuoYi-Local.postman_environment.json --folder "00 登录与鉴权" --folder "01 用户管理" --folder "02 角色/菜单/部门/字典/参数" --folder "03 监控与运维" --folder "04 通用能力（文件/上传下载）" -r "cli,htmlextra" --reporter-htmlextra-export report\ruoyi-main-report.html
```

**期望：全绿。**

> ⚠️ 注意：`91` 现在嵌在 `01 用户管理` 里，所以 `--folder "01 用户管理"` 会把它一起带上。
> 想用方案 B，必须**先把 `91` 移到顶层**。

---

## 八、上传用例的意外收获

`04 通用能力 → 01 文件上传 POST /common/upload` 在 **Postman GUI** 里一直跑不通
（服务端提示 `file is null`），**但在 Newman 里会自动通过**。

**原因**：Newman 是命令行程序，直接按集合 JSON 里的 `src` 路径
（`E:\ruoyi-test\sample-upload.txt`）读磁盘文件，**不需要文件选择器**。

**结论**：那不是接口的问题，也不是路径编码的问题，纯粹是 Postman GUI 的机制。
这也印证了一条原则：**自动化用例应该能在无 GUI 环境下完整跑通。**

---

## 九、常见报错

| 报错 | 原因 | 解决 |
| --- | --- | --- |
| `newman 不是内部或外部命令` | 全局安装后 PATH 未生效 | 新开终端；把 `npm config get prefix` 输出的路径加进用户 Path |
| `could not find "cli htmlextra" reporter` | PowerShell 把 -r cli,htmlextra 的逗号解析成了数组 | 给值加引号：`-r "cli,htmlextra"` |
| `EPERM: operation not permitted` | npm 全局目录权限不足 | 用**管理员**终端重装 |
| `unable to read file ... sample-upload.txt` | 上传用的示例文件不存在 | 确认 `E:\ruoyi-test\sample-upload.txt` 还在 |
| 大量 `401` | 若依或 Redis 没启动 | 检查 8080 / 6379 端口，确认两个窗口都开着 |
| 报告没生成 | 报告器名写错 | 必须是 `htmlextra`，且已安装 `newman-reporter-htmlextra` |
| 报出集合解析错误 | 集合 JSON 被破坏 | 重新从 Postman 导出一次 |

---

## 十、下一步：CI 集成

Newman 能跑通后，就可以接 CI（GitHub Actions / Jenkins），流水线大致是：

1. 拉代码
2. 用 Docker 起 MySQL + Redis（工作区里那份 `ruoyi-docker-compose.yml` 就是干这个的）
3. 导入 SQL 并启动若依后端
4. `newman run` 执行用例
5. 归档 HTML 报告
6. **有新增失败则构建失败**

到这一步，「接口自动化」才算真正落地，也才能写进简历：

> 「Postman 设计用例 + Newman 命令行执行 + CI 流水线自动回归，
> 回归耗时从手工 X 分钟降到 Y 秒，失败自动通知。」

---

## 附：一句话总结

> **Postman 用来设计用例，Newman 用来执行用例，Jenkins 用来调度执行。**
> 三者是一条链：设计 → 执行 → 调度。
