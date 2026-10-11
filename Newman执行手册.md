# Newman 执行手册

## 一、目录结构

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

## 二、安装

```powershell
npm install -g newman newman-reporter-htmlextra
```

`newman-reporter-htmlextra` 是生成 HTML 报告用的，要一起装。

装完**新开一个终端**再验证：

```powershell
newman -v
```

能打出类似 `6.x.x` 的版本号就行了。

## 三、从 Postman 导出

**导出集合**：左侧 `Collections` → 右键集合 → `Export` → 选 `Collection v2.1`
→ 存到 `postman\RuoYi-Vue.postman_collection.json`

**导出环境**：左侧 `Environments` → 右键环境 → `Export`
→ 存到 `postman\RuoYi-Local.postman_environment.json`

⚠️ **导出后检查一下 token**：Postman 会把环境变量的当前值一起导出来。
用记事本搜一下 `token`，如果是 `eyJhbGci...` 这样的真实值，**手动清空成 `""`**。
这个文件要提交 Git，不能带真实凭证。

## 四、执行

### 全量执行（日常用这个）

```powershell
cd 'E:\学习\postman测试'

newman run postman\RuoYi-Vue.postman_collection.json -e postman\RuoYi-Local.postman_environment.json --timeout-request 10000 -r "cli,htmlextra" --reporter-htmlextra-export report\ruoyi-api-report.html
```

| 参数 | 作用 |
| --- | --- |
| `run <集合>` | 执行集合 |
| `-e <环境>` | 加载环境变量（不加会大量 401） |
| `--timeout-request 10000` | 单请求超时 10 秒，防止卡死 |
| `-r "cli,htmlextra"` | 报告器：命令行 + HTML 文件（**逗号两边要加引号**） |
| `--reporter-htmlextra-export` | HTML 报告输出路径 |

### 预期结果

```
All 103  |  Passed 96  |  Failed 7  |  Skipped 0  |  Errors 0
```

- **96 通过** = 主套件全绿
- **7 失败** = `91 已知缺陷回归` 里的断言，对应缺陷 D1 / D7 / D9，属预期红灯
- **0 errors** = 没有环境问题、没有脚本报错

判断标准：失败的 7 条必须都是 `[缺陷00X]` 开头的断言。
如果出现 401、超时、文件读不到，那是环境问题，不是「已知缺陷」。

> 接 CI 的时候，门槛设成「**不允许新增失败**」，而不是「零失败」—— 真实项目就是这么做的。

### 只跑已知缺陷回归

```powershell
newman run postman\RuoYi-Vue.postman_collection.json -e postman\RuoYi-Local.postman_environment.json --folder "91 已知缺陷回归" -r cli
```

预期：3 个请求、7 条断言全部失败。

### 带参数化执行

```powershell
newman run postman\RuoYi-Vue.postman_collection.json -e postman\RuoYi-Local.postman_environment.json -d data\users.csv -n 3 -r "cli,htmlextra" --reporter-htmlextra-export report\ruoyi-api-report-data.html
```

| 参数 | 作用 |
| --- | --- |
| `-d data\users.csv` | 指定迭代数据文件 |
| `-n 3` | 迭代 3 次（用前 3 行数据） |

⚠️ CSV 的列名要和请求里的 `{{变量名}}` 对上才会生效，对不上该变量就是空的。
比如 CSV 里有 `userName` 列，请求体里就要写 `{{userName}}`。

## 五、看 HTML 报告

双击打开 `report\ruoyi-api-report.html`。

里面有总请求数、断言通过率、总耗时，每个请求的耗时，失败的用例，以及请求/响应的原文。

实测 `03 监控与运维 → 05 服务监控 GET /monitor/server` 耗时约 **1200 ms**，
其余接口都在 5~20 ms。该接口要用 OSHI 采集 CPU / 内存 / 磁盘 / JVM 信息，
本身就偏重，属正常现象，但值得记一笔。

## 六、上传用例在 Newman 里才跑得通

`04 通用能力 → 01 文件上传 POST /common/upload` 在 **Postman GUI** 里一直跑不通
（服务端提示 `file is null`），**但在 Newman 里会自动通过**。

原因：Newman 是命令行程序，直接按集合 JSON 里的 `src` 路径
（`E:\ruoyi-test\sample-upload.txt`）读磁盘文件，不需要文件选择器。

所以那不是接口的问题，也不是路径编码的问题，是 Postman GUI 的机制。

## 七、常见报错

| 报错 | 原因 | 解决 |
| --- | --- | --- |
| `newman 不是内部或外部命令` | 全局安装后 PATH 没生效 | 新开终端；把 `npm config get prefix` 的路径加进用户 Path |
| `could not find "cli htmlextra" reporter` | PowerShell 把 `-r cli,htmlextra` 的逗号解析成了数组 | 给值加引号：`-r "cli,htmlextra"` |
| `EPERM: operation not permitted` | npm 全局目录权限不足 | 用**管理员**终端重装 |
| `unable to read file ... sample-upload.txt` | 上传用的示例文件不存在 | 确认 `E:\ruoyi-test\sample-upload.txt` 还在 |
| 大量 `401` | 若依或 Redis 没启动 | 检查 8080 / 6379 端口，确认两个窗口都开着 |
| 报告没生成 | 报告器名写错 | 必须是 `htmlextra`，且已装 `newman-reporter-htmlextra` |
| 报出集合解析错误 | 集合 JSON 被破坏 | 重新从 Postman 导出一次 |

## 八、下一步

Newman 跑通之后可以接 CI（GitHub Actions / Jenkins），大致的流水线是：

1. 拉代码
2. 用 Docker 起 MySQL + Redis
3. 导入 SQL 并启动若依后端
4. `newman run` 执行用例
5. 归档 HTML 报告
6. 有新增失败就让构建失败

这一步目前还没做，先记在这里。
