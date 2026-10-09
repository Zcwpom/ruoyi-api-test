# 若依（RuoYi-Vue）接口测试项目

拿开源的若依后台管理系统当被测系统，练了一遍接口测试的完整流程：
看接口文档 → 设计用例 → 写断言 → 找问题 → 提缺陷单 → 做自动化回归。

- 覆盖系统管理、系统监控等 **6 个模块、37 个接口**
- 写了 **110 条断言**，用 Newman 命令行跑全量回归，**耗时 6 秒**
- 发现 **7 个问题**，其中 3 个写了正式缺陷报告，1 个是高危

## 一、关键数据

| 项目 | 数值 |
| --- | --- |
| 被测系统 | RuoYi-Vue v3.9.2（Spring Boot + Spring Security + JWT） |
| 接口数量 | 37 个 |
| 断言数量 | 110 条（实跑 109 条，有 1 条因为验证码关掉了没执行） |
| 发现的问题 | 7 个（高危 1 / 中 3 / 低 3） |
| 正式缺陷报告 | 3 份 |
| 全量回归耗时 | 6 秒 |
| 用到的工具 | Postman、Newman、MySQL、Redis、Git |

## 二、目录结构

```
.
├── postman/                                   # 集合和环境变量
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
| 没做性能测试 | 现在只有响应时间的兜底断言，压测得用 JMeter |
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