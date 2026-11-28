# 何帝文 · 安全工程作品集

网络工程（网络安全方向）在读，目标岗位是渗透测试 / 红队。

这里是我主要在做的三个项目：一个 Web 漏洞靶场、一个攻击面扫描平台、一个 Burp 越权检测插件。靶场给平台提供确定的测试目标，插件则把同一套「先做对照、再下结论」的方法用在真实流量上。

| 项目 | 做什么 | 技术栈 | 规模 |
|---|---|---|---|
| **[attack-surface](https://github.com/hediwen831-star/attack-surface)** | 外网攻击面测绘与漏洞验证 | Python 3.11+ · asyncio · SQLAlchemy | 434 个单元测试 · 88% 覆盖率 · ruff 无告警 |
| **[vulnlab](https://github.com/hediwen831-star/vulnlab)** | Web 漏洞靶场与 PoC 基准集 | PHP 7.3 / 7.4 / 8.1 · SQLite | 13 个场景 · 171 条 CI 断言 |
| **[burp-idor-detector](https://github.com/hediwen831-star/burp-idor-detector)** | Burp 越权（IDOR）检测插件 | Java 21 · Burp Montoya API | 94 个单元测试 · v0.1.1 |

## attack-surface

外网攻击面测绘与漏洞验证平台。资产发现、服务识别、指纹匹配、PoC 验证、报告导出做成一条异步流水线，每次扫描落库，可以对比前后两次结果。

几个实现上的选择：

- PoC 用 YAML 描述，匹配条件走白名单 DSL，不执行任意代码
- 命中后向同目录随机路径再发一次请求，排除「任何路径都返回 200」这类误报
- 子域枚举前先做泛解析基线，避免把泛解析记录当成资产
- 每次扫描独立成任务，资产变更可以 diff

![ASP 看板](https://raw.githubusercontent.com/hediwen831-star/attack-surface/main/docs/images/dashboard.png)

图是对 vulnlab 的实测结果：31 处命中，13 严重 / 16 高危 / 2 中危。

测试 434 个，覆盖率 88%，ruff 无告警。CI 跑 Python 3.11 / 3.12 / 3.13 和 Windows，另有 Docker 构建任务。

[README](https://github.com/hediwen831-star/attack-surface) ·
[架构说明](https://github.com/hediwen831-star/attack-surface/blob/main/ARCHITECTURE.md)

## vulnlab

PHP 写的 Web 漏洞靶场，13 个场景。每个场景分三档：`low` 没有防护，`medium` 有防护但能绕过，`high` 是修好之后的实现。

每个场景配四样东西：三档源码、Writeup、机读 YAML PoC、修复对照。`high` 档页面功能正常，正常请求照常返回，只是漏洞路径被堵住。

medium 档的绕过大多属于同一类问题：防护依赖了请求方能控制的信息。例如：

- SQL 注入：黑名单删除 `union`，双写 `ununionion` 又拼了回来
- 反序列化：类名黑名单不区分大小写，小写 `cache` 即可绕过
- XXE：过滤器只看请求体，实体声明挪到外部 DTD 就看不见了
- CSRF：只查 `Referer`，客户端可以不发送
- 越权：归属校验写对了，但「我是谁」来自请求参数 `uid`

![VulnLab 首页](https://raw.githubusercontent.com/hediwen831-star/vulnlab/main/docs/images/lab-index.png)

CI 有 171 条断言，分三类：漏洞场景没被改坏、防护确实生效、修复没有把功能删掉。

[README](https://github.com/hediwen831-star/vulnlab) ·
[13 份 Writeup](https://github.com/hediwen831-star/vulnlab/tree/main/writeups) ·
[设计文档](https://github.com/hediwen831-star/vulnlab/blob/main/docs/DESIGN.md)

## burp-idor-detector

Burp 插件。从流量里挑出「像资源标识」的参数，用第二个身份重放，再用一个不存在的 ID 做对照，判断服务端有没有校验资源归属。

比对前会先做标准化：去掉 CSRF token、时间戳、traceId 这类每次都变的内容，再算 5-gram Jaccard 相似度，加上状态码和 Content-Type 的权重。

判定分三种情况：

| 换身份后 | 对照请求 | 结论 |
|---|---|---|
| 相似 | 不相似 | 疑似越权 |
| 相似 | 相似 | 接口对任何 ID 都返回固定内容，误报，丢弃 |
| 不相似 | 任意 | 权限校验生效 |

插件默认不主动发请求：只识别参数、写日志。需要在面板里配置第二身份并勾选「启用主动重放」才会开始检测。只对 `GET` / `HEAD` / `OPTIONS` 生效，每个 host 有候选配额和请求间隔。

94 个测试，判定逻辑与 Burp API 解耦，大部分可以脱离 Burp 跑。Release 里提供编译好的 jar。

[README](https://github.com/hediwen831-star/burp-idor-detector) ·
[设计文档](https://github.com/hediwen831-star/burp-idor-detector/blob/main/docs/DESIGN.md)

## 跑一遍

```bash
# 1) 起靶场（需要 PHP 7.2+）
cd vulnlab && ./start.sh          # Windows: start.bat

# 2) 另开一个终端，用平台引擎扫靶场
cd attack-surface && pip install -e .
python -m asp.cli poc run http://127.0.0.1:8080 --dir ../vulnlab/pocs
```

实测输出：

```
执行 PoC: 21   耗时: 5.8s
命中: 31

critical  vulnlab-command-injection          /cmdi/low.php                   1.00
critical  vulnlab-ssti-template-injection    /ssti/low.php                   1.00
high      vulnlab-xxe-external-entity        /xxe/low.php                    1.00
high      vulnlab-idor-broken-access-control /idor/low.php?order_id=1003      1.00
high      vulnlab-csrf-state-change          /csrf/medium.php                1.00
```

13 个场景的 PoC 全部命中，`high` 档没有误报。

## 免责声明

以上项目仅用于授权范围内的安全测试与学习。

- `vulnlab` 是故意包含漏洞的教学代码，不要部署到公网
- `attack-surface` 内置限速与可识别前缀标记，不要用于未授权目标
- `burp-idor-detector` 只应在已获得授权的目标上使用

使用者需自行确保合规。
