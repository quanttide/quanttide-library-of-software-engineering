Python 治理三件套：Ruff（lint + format 总入口）+ import-linter（架构边界守护）+ pip-audit（官方供应链治理），再用 mypy（类型守护）、pydeps（依赖图可视化）、vulture（死代码清理）补齐，整个观测层就能完整外包。Python 生态当前的特点是——Astral 的 Ruff + uv 已经重塑了治理面，一个 Rust 写的 Ruff 替代了 flake8、isort、pydocstyle、pyupgrade 等十几款工具，加上官方 PyPA 的 pip-audit，Python 从”工具碎片化最严重的语言“变成了”入口最清爽的语言之一“。
Python 治理工具全景表
层级	解决什么问题	推荐工具	接入成本	开源状态	备注
Lint + Format 总入口	风格、错误、复杂度、性能	Ruff（含 formatter）	极低，pyproject.toml 一份	✅ 开源（MIT）	Rust 写的，比 flake8 快 10-100 倍，规则兼容 flake8/pylint/isort 等
类型检查	类型错误、接口契约	mypy（官方倾向）或 Pyright（VSCode 默认）	中，需逐步开启 strict	✅ 开源	mypy 用 strict mode 渐进落地，是大型项目的标配
架构边界守护	分层规则、package 依赖禁令	import-linter	低，setup.cfg/pyproject.toml 一份	✅ 开源	Python 生态里最接近 ArchUnit 的工具，支持 layers/forbidden/independence 三种契约
轻量 import 限制	Ruff 内置禁止某些 import	Ruff flake8-tidy-imports	极低	✅ 开源	不需要完整架构规则时的轻量替代
循环依赖检测	破坏 import 环	import-linter + pydeps	低	✅ 开源	pydeps 直接可视化循环依赖
死代码检测	未使用函数/类/变量	vulture	低	✅ 开源	基于 AST 扫描，可配合 Ruff 的 F401 一起用
模块依赖图	看清”谁 import 谁“	pydeps	低	✅ 开源	输出 Graphviz SVG，可直接嵌文档站
安全漏洞扫描	依赖中的已知 CVE	pip-audit（PyPA 官方）	极低	✅ 官方	基于 Python Packaging Advisory Database，官方安全团队维护
静态安全扫描	代码本身的安全问题	Bandit（可集成进 Ruff）或 Semgrep	低	✅ 开源	Ruff 已支持 Bandit 规则集，无需单独装
License 合规	依赖 License 白名单	pip-licenses	低	✅ 开源	配 Ruff/pip-audit 一起构成供应链三件套
复杂度/度量	圈复杂度、函数规模	radon + Ruff pylint 兼容规则	低	✅ 开源	radon 专注复杂度度量，Ruff 的 C901 也覆盖基础场景
包/项目管理	依赖、虚拟环境、monorepo	uv（Astral）	低	✅ 开源	替代 pip/poetry/pyenv/virtualenv，已支持 workspace 管理 monorepo
深度依赖分析	复杂 import 关系编程式分析	grimp	中	✅ 开源	import-linter 底层用的就是它，可编程构建 import 图
全指标沉淀	统一报表、Quality Gate	SonarQube Community Build	中	✅ 社区版免费	官方支持 Python，覆盖 Ruff/mypy 之外的度量维度
三个核心工具的深度说明
① Ruff —— Python 治理的统一容器
这是当前 Python 生态最重要的一件工具。Ruff 用 Rust 重写了整个 lint + format 流程，一个二进制同时替代 flake8、isort、pydocstyle、pyupgrade、 autoflake、pyflakes、pycodestyle，甚至集成了 Bandit 的安全规则。它原生读取 pyproject.toml，一份配置文件就是治理事实源，IDE（VSCode/PyCharm）和 CI 完全一致。规则集支持从”只开 E/F“到”全选 ALL“的渐进式落地，大规模代码库也能在毫秒级跑完，CI 里开启 —fix 还能自动修复大量风格问题。
② import-linter —— Python 的”ArchUnit“
这是 Python 生态最值得投入的架构工具。它用一个配置文件声明”契约“，目前有三种核心契约：
• layers：分层架构，如 api → service → domain → infrastructure，强制只能向下依赖
• forbidden：禁止某模块 import 另一模块，如 domain 不得 import django.*
• independence：若干模块必须彼此独立，如各 feature 模块之间不得互相 import
CI 一条 lint-imports 就能阻断违规合并。相比手工维护架构图，这份配置本身就是可执行的架构陈述——和 go-arch-lint / dependency-cruiser 是同一思路。它的底层是 grimp 库，复杂规则可以直接用 grimp 编程式分析。
③ pip-audit —— PyPA 官方供应链治理
Python 官方打包权威机构 PyPA 出品的漏洞扫描器，基于 Python Packaging Advisory Database，扫描 requirements.txt / pyproject.toml / 已安装环境都能支持。相比第三方 Safety，它是官方出品、数据源更权威、无商业 license 顾虑，CI 一条 pip-audit 就能作为 Quality Gate。
第二大脑载体
• 底层（事实源）：Ruff 报表 + mypy 报告 + import-linter 验证结果 + pydeps 架构图 + pip-audit 漏洞清单 + vulture 死代码清单，全部从代码自动生成
• 中层（资产目录）：每个 package 一份 catalog-info.yaml 接入 Backstage，把 SonarQube 报表、pydeps 依赖图、import-linter 契约集挂到对应服务页面
• 顶层（架构陈述）：import-linter 的契约配置本身就可以渲染成架构图，配合 Structurizr DSL 维护更宏观的 C4 模型
如果 Backstage 太重，最低成本方案：Sphinx（Python 官方文档系统，支持 autodoc 自动生成 API 文档）+ mdBook/Zola 静态站点，CI 把 Ruff 报表、pydeps SVG、pip-audit 报告注入 Markdown。
三阶段落地路径
阶段一：体检（第 1 周）
一次跑齐：ruff check、mypy .、lint-imports、pydeps、pip-audit、vulture、pip-licenses。产出一份”系统健康画像“：lint 违规数、类型盲区、架构违规、循环依赖、已知漏洞、死代码、License 风险。这份画像是第二大脑的第一块砖。
阶段二：建边界（第 2–4 周）
• 用 uv workspace 把项目拆成职责清晰的多个 package（core / domain / api / infrastructure）
• 用 import-linter 写 3–5 条核心契约（分层禁令、禁止反向依赖、feature 模块独立性），先报告模式后阻断模式
• 开启 mypy strict 模式渐进落地，从 domain 层开始、逐步外扩
• 全部规则进 CI，违规阻断合并
阶段三：守护 + 陈述（第 2 个月起）
• Ruff + mypy + import-linter + pip-audit 全部强制阻断
• SonarQube Community Build 作为统一 Quality Gate
• 第二大脑站点每周自动刷新，架构图一律从 import-linter/pydeps 产出，禁止手写
与其他四种语言的对比一句话
Rust 靠编译器 + cargo 原生守护、Dart 靠一份 analysis_options.yaml、TypeScript 靠 ESLint 当容器、Go 靠官方工具链覆盖大半，而 Python 的独特优势是”一个 Astral 系 + 一个 PyPA 系“双核极简——Ruff + uv 一家承包 lint/format/包管理/monorepo，pip-audit 一家承包供应链安全，加上 import-linter 一份 YAML 做架构，总共只需要装 4-5 个工具就能跑通完整治理。代价是 Python 的动态性让 mypy 的 strict 落地比 TypeScript 慢，历史遗留代码的类型盲区需要长期投入。
一句话收束：Python 治理的杠杆点是 Ruff 的一份 pyproject.toml + import-linter 的一份契约配置，前者把 lint/format/安全规则统一进 CI，后者把架构边界写成可执行的契约——你的第二大脑同样从 CI 自动长出来。