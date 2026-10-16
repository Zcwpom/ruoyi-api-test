# 若依（RuoYi-Vue）接口测试项目

拿开源的若依后台管理系统当被测系统，练了一遍接口测试的完整流程：
看接口文档 → 设计用例 → 写断言 → 找问题 → 提缺陷单 → 做自动化回归。

- 覆盖登录鉴权、用户管理、系统管理、监控运维、通用能力 **5 大模块、37 个接口**
- 写了 **110 条断言**，用 Newman 命令行跑全量回归，**耗时 6 秒**
- 发现 **7 个问题**，其中 3 个写了正式缺陷报告，1 个是高危
- 用 **JMeter** 做了登录链路的压测与瓶颈定位（阶梯加压 10/20/50/100 并发）

## 一、关键数据

| 项目 | 数值 |
| --- | --- |
| 被测系统 | RuoYi-Vue v3.9.2（Spring Boot + Spring Security + JWT） |
| 接口数量 | 37 个 |
| 断言数量 | 110 条（实跑 109 条，有 1 条因为验证码关掉了没执行） |
| 发现的问题 | 7 个（高危 1 / 中 3 / 低 3） |
| 正式缺陷报告 | 3 份 |
| 全量回归耗时 | 6 秒 |
| 用到的工具 | Postman、Newman、JMeter、MySQL、Redis、Git |

## 二、目录结构

```
.
├── postman/                                   # 集合和环境变量
├── jmeter/                                    # JMeter 性能测试
│   ├── 性能测试报告-若依.md                     # 完整性能测试报告
│   ├── data/users-ruoyi.csv                    # 压测参数化数据
│   ├── plans/                                  # 6 个压测脚本
│   └── report/ruoyi-load-report/               # HTML 仪表盘报告
├── data/users.csv                             # 参数化数据
├── report/ruoyi-api-report.html               # Newman 生成的测试报告
├── 证据/                                       # 缺陷截图
├── 缺陷报告-D1/D7/D9.md                        # 3 份正式缺陷报告
├── 测试记录表.md                               # 110 条断言的实测记录
├── Newman执行手册.md                           # 命令行执行说明
└── README.md
```

## 三、主要做了什么

### 1. 集合按模块分类，鉴权配置一次就够

集合分成 `00 登录与鉴权`、`01 用户管理`、`02 角色/菜单/...`、`03 监控与运维`、
`04 通用能力` 五个文件夹。

鉴权统一配在集合根节点上（Bearer {{token}}），只有登录相关的接口单独设成
不用鉴权。这样新加接口不用一个个配请求头。

### 2. 加了自动登录，不然 token 一过期就全红

一开始必须先手动跑一遍登录，不然其他接口全是 401。后来在集合级的
Pre-request 里加了判断：如果本地没有 token，或者 token 快过期了，
就先调一次登录接口拿新的。

还有两种情况也会 401，是后面才发现的：

- **「退出登录」和「强退在线用户」这两个用例本身就会把会话弄失效** ——
  所以在这两个用例跑完之后，主动把本地的 token 清掉
- **服务端把 token 作废了（比如 Redis 重启），但本地还留着** ——
  所以又在集合级加了一层：收到 401 就把 token 清掉，重跑一次当前请求

现在 37 条用例从哪一条开始跑都行，不用管顺序。

### 3. 断言分了几个层次

每个接口大概写这几类断言：

- HTTP 状态码是不是 200
- 业务 code 是不是 200（若依把错误放在业务 code 里，HTTP 永远是 200）
- 返回结构对不对（有没有 total、rows 这些字段）
- 字段类型对不对
- 响应时间有没有超标

像“不该泄露内部信息”这种，就直接断言响应里不能出现某些内容：

```js
pm.expect(body).to.not.match(/com\.ruoyi|SysUser|java\.lang/);
```

### 4. 回归用例断言的是“修好之后应该是什么样”

`91 已知缺陷回归` 这个文件夹里的用例，是给已经提了单的缺陷做的回归。

断言写的是期望的正确行为，所以现在 **是红灯**（因为缺陷还没修）。等修好了
会自动变绿，不用改用例。比如：

```js
pm.test("[缺陷009] 不得泄露原始 SQL 语句", () => {
  pm.expect(body).to.not.include("SELECT");
});
```

主套件（00~04）保持全绿，91 单独一个文件夹放。跑的时候可以只跑主套件，
也可以一起跑但允许这 7 条红灯（相当于已知问题清单）。

### 5. 写错过一条断言，后来改掉了

D9 的回归用例里本来有一条 `code ≠ 200`。它现在确实是绿的，但绿的**原因是
SQL 报错了**，不是因为参数校验生效了。而且按我建议的修法（非法值当 asc 处理），
修好之后 code 会变成 200，这条断言**反而会变红**。

后来改成了条件断言：

```js
if (json.code !== 200) {
  pm.test("[缺陷009] 若返回错误，提示必须可读", () => {
    pm.expect(String(json.msg)).to.match(/[\u4e00-\u9fa5]/);
  });
}
```

这次明白的是：回归断言只能写“修完之后一定会成立”的东西，不能把某一种
修法写死。

### 6. 用 Newman 在命令行跑

```bash
newman run postman/RuoYi-Vue.postman_collection.json \
  -e postman/RuoYi-Local.postman_environment.json \
  --timeout-request 10000 \
  -r "cli,htmlextra" \
  --reporter-htmlextra-export report/ruoyi-api-report.html
```

两个踩过的点：

- PowerShell 里 `-r cli,htmlextra` **要加引号**，不然逗号会被当成数组，
  Newman 会去找一个叫 “cli htmlextra” 的报告器然后报错
- 文件上传的用例在 Postman 界面里一直发不出去（服务端收到的是 null），
  换成 Newman 就正常了。原因是命令行会直接按集合里记的路径读文件，
  界面版不会

### 7. 用 JMeter 做了性能测试（登录链路）

功能测试只能证明「对不对」，证明不了「快不快、扛不扛得住」。所以又用 JMeter 压了一遍登录链路。

链路：`GET /captchaImage` → `POST /login` → `GET /getInfo`

脚本里做了三件事：

- **关联**：登录接口下挂 JSON 提取器，用 JSONPath `$.token` 把 token 提成变量；
  `/getInfo` 下挂 HTTP 信息头管理器，写 `Authorization: Bearer ${token}`
- **参数化**：CSV 数据文件设置驱动 admin / ry 两个账号
- **事务**：事务控制器把三个请求合成一个「登录流程」事务，报告里能看到业务口径的 TPS

**阶梯加压结果（5 秒爬升 / 15 秒持续）：**

| 并发 | 样本数 | TPS | 平均响应 | 最大响应 | 错误率 |
| --- | --- | --- | --- | --- | --- |
| 10 | 1,793 | 119.6 | 71 ms | 281 ms | 0.00% |
| 20 | 2,098 | 137.3 | 121 ms | 554 ms | 0.00% |
| 50 | 2,248 | 146.0 | 282 ms | 1,328 ms | 0.00% |
| 100 | 2,434 | 152.1 | 538 ms | 2,068 ms | 0.00% |

并发提升 9 倍，TPS 只涨 27%，而平均响应涨了 658% —— **新增的并发基本都变成了排队**，
据此判断容量上限约 20 并发。

**单接口隔离压测（50 并发 / 15 秒）定位瓶颈：**

| 被测接口 | 样本数 | TPS | 平均响应 | 最大响应 |
| --- | --- | --- | --- | --- |
| GET /captchaImage | 380,755 | **25,324** | 1 ms | 27 ms |
| POST /login | 654 | **42.1** | **990 ms** | 1,458 ms |

`/captchaImage` 能跑 2.5 万 TPS，说明 Web 容器和网络层没问题；而 `/login` 在 10 并发
（40.1 TPS / 201 ms）和 50 并发（42.1 TPS / 990 ms）下吞吐几乎不变、响应时间翻了 5 倍，
说明登录链路上有个「串行化」的东西卡着。

结合响应时间分布很集中（152 ~ 281 ms），判断是 **BCrypt 密码校验的 CPU 密集型计算**
（若依用 `BCryptPasswordEncoder`，为了抗暴力破解刻意设计成慢的）。
所以优化方向是**水平扩容**，而不是降低加密强度。

**压测必须用命令行跑，不能用 GUI：**

```bash
jmeter -n -t "03-压测演练.jmx" -l "load.jtl" -e -o "load-report"
```

`-n` 非 GUI、`-t` 脚本、`-l` 结果数据、`-e -o` 生成 HTML 仪表盘报告。

> ⚠️ 这次是**单机压测**（JMeter 和被测系统在同一台机器上），压力机自身也吃 CPU，
> 所以测出来的容量偏保守，主要用来做**横向对比和瓶颈定位**，不是绝对值。
> 要做容量规划得在独立压力机上复测。

> 完整报告见 [jmeter/性能测试报告-若依.md](jmeter/性能测试报告-若依.md)，压测脚本与 HTML 仪表盘报告也在 [jmeter/](jmeter/) 目录下。

## 四、发现的问题

| 编号 | 问题 | 严重程度 | 状态 |
| --- | --- | --- | --- |
| **D9** | 排序参数 isAsc 没做校验，传非法值会让接口把**完整 SQL 语句、服务器路径**返回给客户端 | **高** | [报告](缺陷报告-D9-isAsc未校验导致SQL信息泄露.md) |
| **D1** | 查一个不存在的用户，返回的是 Java 异常信息，里面有包名、类名 | 中 | [报告](缺陷报告-D1-用户详情NPE信息泄露.md) |
| D3 | 查询时间范围传反了（开始时间晚于结束时间），没有报错，直接返回全部数据 | 中 | 已记录 |
| D4 | 查询时间格式写错，被忽略掉，也是返回全部数据 | 中 | 已记录 |
| D5 | pageSize 传 100000 也能执行，没有上限 | 中 | 已记录 |
| **D7** | 分页参数传了非数字，不报错，静默按默认值处理；但同一个模块的 userId 传非数字会明确报错，两套策略不一致 | 低 | [报告](缺陷报告-D7-分页参数静默降级.md) |
| D2 | 用 authRole 接口查不存在的用户，返回 code=200 还带 user:null | 低 | 已记录 |

### 另外验证通过的几项

| 检查项 | 结果 |
| --- | --- |
| SQL 注入 `userName=' or '1'='1` | ✅ 返回空列表，没被注入 |
| 排序字段注入 `orderByColumn=1,(select 1)` | ✅ 被拦截，提示“参数不符合规范” |
| 不带 token 访问 | ✅ 返回 401 |
| 用普通角色 ry 查别人的数据 | ✅ 只能看到自己的 |

> D9 那条缺陷的预期依据，就是从第 2 项来的：同一个 ORDER BY 的位置上，
> `orderByColumn` 防住了，`isAsc` 完全没防。

## 五、用例是怎么设计的

每个接口从这几个方向想测试点，不只是测“能不能通”：

| 方向 | 具体想什么 |
| --- | --- |
| 鉴权 | 不带 token、token 是假的、权限不够、只能看自己部门的数据 |
| 参数 | 必填不传、类型传错、边界值（0、-1、特别大）、超长、特殊字符、注入 |
| 业务规则 | 状态值合不合法、时间范围能不能反过来、唯一性 |
| 数据一致性 | total 和 rows 对不对得上、列表和详情一样吗、改完再查一遍 |
| 组合 | 多个条件一起传会怎样 |
| 副作用 | 会不会影响别的数据、要不要善后 |
| 性能 | 响应时间、重复提交会怎样 |

具体数据在 [测试记录表.md](测试记录表.md) 里。

## 六、怎么跑起来

### 环境要求

| 组件 | 版本 |
| --- | --- |
| JDK | 17+（我用的 24.0.1） |
| MySQL | 5.7+（我用的 8.0.37） |
| Redis | 3.0+（我用的 8.10.2） |
| Node.js | 14+（我用的 24.21.0） |
| Postman | 12.x |
| Newman | 6.x |

### 启动被测系统

```bash
git clone https://gitee.com/y_project/RuoYi-Vue.git
# 建库 ry-vue，导入 sql/ 目录下的两个 sql 文件
# 改 application-druid.yml（数据库密码）和 application.yml（ruoyi.profile 上传目录）
mvn clean package -DskipTests
java -jar ruoyi-admin/target/ruoyi-admin.jar
```

测试环境关掉验证码（方便自动化）：

```sql
UPDATE sys_config SET config_value = 'false' WHERE config_key = 'sys.account.captchaEnabled';
```

### 跑自动化

```bash
npm install -g newman newman-reporter-htmlextra

newman run postman/RuoYi-Vue.postman_collection.json \
  -e postman/RuoYi-Local.postman_environment.json \
  --timeout-request 10000 \
  -r "cli,htmlextra" \
  --reporter-htmlextra-export report/ruoyi-api-report.html
```

跑出来大概是：

```
requests           39     failed 0
assertions        109     failed 7
total run duration: 6s
```

那 7 条失败就是 91 文件夹里的，对应上面 3 个缺陷，属于预期内。

## 七、还没做的

| 事项 | 说明 |
| --- | --- |
| 没接 CI | 后面想用 GitHub Actions 或 Jenkins：起 MySQL/Redis → 启动后端 → 跑 newman → 归档报告 |
| D3/D4/D5 没写正式报告 | 已经记在测试记录表里了，和已有的几个缺陷是同类问题 |
| 参数化没全面用上 | `data/users.csv` 准备好了，但只试了一部分接口 |
| 没和代码化自动化结合 | 后面可以试试用 pytest + requests + allure 重写一遍 |

## 八、文档

| 文档 | 内容 |
| --- | --- |
| [测试记录表.md](测试记录表.md) | 每条断言的实测记录，含根因分析 |
| [缺陷报告-D1](缺陷报告-D1-用户详情NPE信息泄露.md) | 查询不存在的用户返回 Java 异常信息 |
| [缺陷报告-D7](缺陷报告-D7-分页参数静默降级.md) | 分页参数静默降级，两套校验策略 |
| [缺陷报告-D9](缺陷报告-D9-isAsc未校验导致SQL信息泄露.md) | 排序参数未校验导致 SQL 和路径泄露 |
| [Newman执行手册.md](Newman执行手册.md) | 命令行执行和 CI 接入说明 |
| [report/ruoyi-api-report.html](report/ruoyi-api-report.html) | Newman 测试报告 |
| [jmeter/性能测试报告-若依.md](jmeter/性能测试报告-若依.md) | JMeter 性能测试报告（阶梯加压 + 瓶颈定位） |
| [jmeter/plans/](jmeter/plans/) | 6 个 JMeter 压测脚本 |
| [jmeter/report/ruoyi-load-report/index.html](jmeter/report/ruoyi-load-report/index.html) | JMeter HTML 仪表盘报告 |

## 九、关于仓库里的信息

测试报告和缺陷截图里包含本地测试环境的一些信息，这里说明一下：

- **HTML 报告里的 token**：是针对 `http://localhost:8080` 本地服务生成的。若依的
  token 默认 30 分钟过期，报告生成时就已经失效了，对其他人没有用处。
- **内网 IP（10.30.16.81）**：是本机网卡的地址。10.x 属于私有网段，公网访问不到，
  也没法据此访问我的电脑。
- **admin / admin123**：若依官方文档里公开的默认账号密码，不是真实凭证，
  写在缺陷报告的复现步骤里是为了让别人能照着复现。
- **本机路径（E:\学习\...）**：D9 缺陷报告里出现的那处路径，是**接口泄露出来的
  内容本身**，属于缺陷证据，故意保留没做处理。

真实项目里这些都应该脱敏。这里因为是完全本地的练习环境，所以保留了原始数据，
方便别人对照复现。
