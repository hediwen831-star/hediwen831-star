# 何帝文 · 安全工程作品集

> 网络工程（网络安全方向）在读，目标岗位：**渗透测试 / 红队**。
>
> 下面三个项目不是三份互不相干的代码 —— 它们互相咬合成一个闭环：
> **靶场提供确定的目标，平台证明 PoC 能被自动化消费，插件补上「人工验证」那一环。**

| 项目 | 定位 | 技术栈 | 规模 |
|---|---|---|---|
| **[attack-surface](https://github.com/hediwen831-star/attack-surface)** | 外网攻击面测绘与漏洞验证平台 | Python 3.11+ · asyncio · SQLAlchemy | 318 个单元测试 · ruff 零告警 |
| **[vulnlab](https://github.com/hediwen831-star/vulnlab)** | Web 漏洞靶场与 PoC 基准集 | PHP 7.3/7.4/8.1 · SQLite · 原生 | 12 个场景 · 75 条 CI 断言 |
| **[burp-idor-detector](https://github.com/hediwen831-star/burp-idor-detector)** | Burp 越权（IDOR）检测插件 | Java 21 · Burp Montoya API | 81 个单元测试 · v0.1.0 |

---

## 闭环长什么样

```
        ┌──────────────────────┐              ┌──────────────────────┐
        │      vulnlab         │   机读 PoC    │    attack-surface    │
        │                      │ ───────────► │                      │
        │  12 个漏洞场景        │              │  YAML PoC 引擎        │
        │  每个都写成三档：      │              │  （白名单 DSL，无 eval）│
        │   low  有漏洞         │ ◄─────────── │                      │
        │   medium 能被绕过      │  基准 / 误报诱饵 │  负向对照校验防误报    │
        │   high 已修复         │              │  资产变更 diff        │
        └──────────────────────┘              └──────────────────────┘
                                                       ▲
                                                       │ 同一套「负向对照」思路
                                                       ▼
                                        ┌──────────────────────────────┐
                                        │     burp-idor-detector       │
                                        │  从真实流量里三态对比：        │
                                        │   ① 基准  ② 换身份重放         │
                                        │   ③ 不存在的 ID 对照           │
                                        └──────────────────────────────┘
```

**为什么靶场的 `high` 档重要**：它是**天然的误报诱饵**。
引擎在那里报漏洞，就说明判定逻辑有问题。这比「报告列出 50 个问题」有用得多。

---

## 1. attack-surface

> 把「资产发现 → 服务识别 → 指纹匹配 → 漏洞验证 → 报告」做成一条可插拔的异步流水线。

**核心是自研的 YAML PoC 引擎** —— 这是它与「调个现成扫描器」的区别所在：

- **白名单 DSL，不用 `eval`** —— PoC 是数据不是代码。有一条测试专门守着这条线
  （`test_dsl_rejects_non_whitelisted_syntax`）
- **负向对照校验** —— 每个 PoC 带一条「随机路径」的对照请求，
  排除「检测特征被检测行为本身制造出来」这类误报
- **泛解析基线检测** —— 先摸清目标是否对任意子域都返回同样内容，再判断子域是否真实存在
- **资产变更 diff** —— 两次扫描之间的新增/消失资产

除了 CLI，还有一个 FastAPI 写的只读看板 + REST 接口（默认只绑回环，非回环绑定必须带 token）。

**规模**：318 个单元测试（**全部离线可跑**，DNS/HTTP 用 monkeypatch 隔离）、
覆盖率 65%、ruff 零告警、Docker Compose 一键启动。

📖 [README](https://github.com/hediwen831-star/attack-surface) ·
[架构与技术栈详解（用了什么 / 为什么用它 / 核心逻辑怎么跑）](https://github.com/hediwen831-star/attack-surface/blob/main/ARCHITECTURE.md)

---

## 2. vulnlab

> 用 PHP 从零写的漏洞靶场。**每个漏洞都凑齐四件套：三档源码 + Writeup + 机读 PoC + 修复对照。**

12 个场景，覆盖 OWASP Top 10 的主体：

```
SQL 注入    XSS（反射型）  文件上传      SSRF
命令注入    文件包含（LFI） PHP 反序列化  XXE
JWT 伪造    CSRF           SSTI         越权（IDOR）
```

**三档结构不是难度分级，是三种不同的失败方式**：

| | 这一档演示什么 |
|---|---|
| **low** | 完全没有防护 |
| **medium** | 有防护，但**防护依赖了请求方可控的信息**，或者**抓错了特征** |
| **high** | 换方向：**取消能力**而不是加强过滤 |

举几个 medium 档的真实绕过 —— 它们的共同结构比单个技巧更值得看：

- **SQL 注入**：黑名单删 `union` → 双写 `ununionion`（删除动作把片段拼了回来）
- **反序列化**：类名黑名单 → 小写 `cache`（PHP 找类时不区分大小写）
- **XXE**：过滤请求体里的 `<!ENTITY` → 实体声明挪到**外部 DTD**（过滤器看不见，解析器会加载）
- **SSTI**：黑名单挡括号 → 反引号 `` `cmd` ``（运算符不需要括号）
- **CSRF**：检查 `Referer` → 攻击者用 `no-referrer` 去掉它，或注册一个含目标域名的自己人域名
- **越权**：归属校验写对了 → 但「我是谁」来自请求参数 `uid`

> **防护所依据的那个信息，如果是请求方可以自由决定的，那它就挡不住请求方。**

CI 里 75 条断言守着这批资产不被改坏，每条都分三类：
**漏洞场景**（没被改坏）、**对照组**（证明防护确实在工作）、**反向保护**（修复没有过度）。

📖 [README](https://github.com/hediwen831-star/vulnlab) ·
[12 份 Writeup](https://github.com/hediwen831-star/vulnlab/tree/main/writeups) ·
[设计文档：技术栈与教学设计](https://github.com/hediwen831-star/vulnlab/blob/main/docs/DESIGN.md)

---

## 3. burp-idor-detector

> 越权访问的自动化检测插件。从 Burp 的真实流量里挑出「像资源标识的参数」，
> 用**第二身份**重放，通过**三态对比**判断服务端是否校验了资源归属。

核心是**三态对比**：

```
① 基准请求        用自己的身份访问自己的资源
② 换身份重放      用 A 的身份访问 B 的资源
③ 不存在的 ID 对照 用一个一定不存在的 ID 再试一次   ← 关键
```

第 ③ 步挡掉的是同类工具最大的误报来源：很多站点对不存在的 ID 也返回一个
「长得差不多」的错误页，只比对 ①② 就会把它判成越权。

> 这和 attack-surface 里的「负向对照校验」是同一件事：
> **排除「检测特征被检测行为本身制造出来」的可能。**

**设计上值得一提的**：判定逻辑（`ResponseComparator`）刻意与 Burp API 解耦，
所以 81 个测试里绝大多数能脱离 Burp 单独跑。

📖 [README](https://github.com/hediwen831-star/burp-idor-detector) ·
[设计文档：越权检测的判定模型](https://github.com/hediwen831-star/burp-idor-detector/blob/main/docs/DESIGN.md)

---

## 两条命令跑通闭环

```bash
# 1) 起靶场（需要 PHP 7.2+，零其他依赖）
cd vulnlab && ./start.sh          # Windows: start.bat

# 2) 用平台引擎检测靶场（另开一个终端）
cd attack-surface && pip install -e .
python -m asp.cli poc run http://127.0.0.1:8080 --dir ../vulnlab/pocs
```

实测输出：

```
执行 PoC: 18   耗时: 3.7s
命中: 27

critical  vulnlab-command-injection          /cmdi/low.php                   1.00
critical  vulnlab-ssti-template-injection    /ssti/low.php                   1.00
high      vulnlab-xxe-external-entity        /xxe/low.php                    1.00
high      vulnlab-idor-broken-access-control /idor/low.php?order_id=1003      1.00
high      vulnlab-csrf-state-change          /csrf/medium.php                1.00
```

**12 个场景的机读 PoC 全部命中**，且 `high` 档一处未报 —— 这就是「基准集」的作用。

---

## 这套东西能证明什么

1. **不是「会用工具」，是「能设计工具」** ——
   每个设计决策都能讲出取舍：为什么用 asyncio、为什么 PoC 引擎坚持不用 `eval`、
   为什么负向对照要拿「不存在的 ID」而不是「另一个真实 ID」
2. **有真实的调试记录** —— 开发中实际踩到过「报错信息回显 payload 导致特征被自己制造出来」
   （误报 6 → 3）、「CI 断言和机读 PoC 是两套载荷，PoC 坏了 CI 却全绿」这些坑，都修掉了
3. **工程化完整** —— 318 + 81 个离线可跑的单测、ruff 零告警、GitHub Actions CI、
   配置校验、结构化日志、Docker 一键启动
4. **有闭环** —— 三个项目互为验证，而不是三堆不相干的代码

---

## 免责声明

以上项目均仅供**授权范围内的安全测试与学习**使用。

- `vulnlab` 是**故意包含漏洞**的教学代码，**禁止部署到公网**
- `attack-surface` 内置限速与可识别前缀标记，**禁止用于未授权目标**
- `burp-idor-detector` 只应在你已获得授权的目标上使用

使用者需自行确保合规。
