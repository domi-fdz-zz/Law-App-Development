# Data Boundary（Law App）

一个**本地优先**的美国隐私与数据使用合规**初步审查工具**。用户用自然语言描述一个数据使用场景（例如"我们想用 GEO 上的人类 RNA-seq 数据训练商业模型"），工具会抽取结构化事实、让用户确认，然后由**确定性规则**给出结论，并附上可核验的一手法律来源。

> ⚠️ 本工具只识别**可能适用**的法律与义务，**不构成法律意见**，也不判断某项使用是否合法。覆盖范围仅限已建模的美国隐私 / 数据使用法律，不涉及版权、知识产权、GDPR / PIPL 等非美国法律。未被标记不等于"已放行"。

当前版本：`0.1.0-alpha.1`

---

## 核心设计原则

- **结论由规则决定，不由模型决定。** LLM 只负责事实抽取、叙述解释、来源发现和基于报告的问答；适用性判断和最终裁定由 `privacy/gate.py` 中的确定性逻辑产生。
- **人在回路（human-in-the-loop）。** 模型抽取的事实会以可编辑卡片的形式展示给用户，确认/修正后才进入评估。
- **引文必须可核验。** 模型给出的每一段法条引文都会到官方一手来源（政府 / 立法机构域名白名单）抓取原文比对，无法逐字核验的引文直接丢弃。
- **"未知"是一等公民。** 关键事实缺失时结论为 `insufficient`，而不是默认放行。
- **密钥不出本机。** API key 以 `0600` 权限保存在 `~/.config/data-boundary/config.json`，永远不会返回给浏览器（只显示掩码）。

## 工作流程

```
自然语言场景
   │  ① 抽取 (LLM)        → 结构化事实 + 规范化复述
   ▼
用户确认 / 修改事实卡片
   │  ② 评估
   ├─ gate      确定性适用性判断 + 豁免路径（去标识化、合同层等）+ 按用途（研究/商业）给出裁定
   ├─ retrieve  对每部适用法律：推理 + 义务，附引文、原文摘录与官方链接
   ├─ case-scan 针对个案的深度扫描，可发现已建模范围之外的权威来源
   ├─ verify    到官方来源逐字核验引文
   └─ narrate   生成针对个案的解释（不得改变规则裁定）
   ▼
报告 + 右侧基于报告的问答助手
```

裁定等级（由严到宽）：`restricted` → `requires_approval` → `conditionally_allowed` → `insufficient` → `allowed`

## 已建模的法律

HIPAA、Common Rule、FDA 人体受试者法规、GINA、FTC Act §5、COPPA、FERPA、GLBA、VPPA、CCPA/CPRA、弗吉尼亚 VCDPA、伊利诺伊 BIPA、华盛顿 My Health My Data Act。来源登记见 [`privacy/source_registry.json`](privacy/source_registry.json)。

## 项目结构

```
.
├── __init__.py          包入口、版本号、本地端口 (7788)
├── cli.py               命令行：serve / app / info
├── main.py              FastAPI 应用与全部 API 端点
├── config.py            LLM 服务配置（DeepSeek / OpenAI / Claude / Qwen / 自定义）
├── desktop.py           pywebview 原生桌面窗口
├── desktop_entry.py     PyInstaller 打包入口
├── privacy/             当前主产品：隐私 / 数据使用审查
│   ├── schema.py        七维度评估输入（object/actor/use_case/basis/...）
│   ├── gate.py          确定性适用性闸门与裁定合成
│   ├── pipeline.py      两阶段流程：抽取 → 评估（检索、扫描、核验、叙述）
│   ├── source_discovery.py  来源发现与校验提示词、官方域名白名单
│   └── source_registry.json 法律来源登记
├── core/                旧版合规引擎（路由、规则引擎、证据、报告生成）
├── domain_packs/        旧版领域规则包：biodata（active）、genetic_testing、
│                        drug_procurement、agricultural_genomics、custom
├── models/schemas.py    旧版引擎的数据模型
├── data/                前端单页 UI（datause.html 为当前主界面）
└── tests/               pytest 测试
```

## 安装与运行

需要 Python 3.11+。代码以包名 `app` 进行导入（如 `from app.main import app`），因此克隆时请把目录命名为 `app`：

```bash
git clone https://github.com/domi-fdz-zz/Law-App-Development.git app
python -m venv .venv && source .venv/bin/activate
pip install fastapi uvicorn pydantic httpx click python-multipart
pip install pywebview   # 可选：原生桌面窗口
pip install pytest      # 可选：运行测试
```

在 `app` 的**上一级目录**运行：

```bash
python -m app.cli serve        # 启动 Web 版并打开浏览器 http://localhost:7788/
python -m app.cli app          # 以原生桌面窗口运行（需要 pywebview）
python -m app.cli info         # 查看版本与当前 LLM 配置（不显示密钥）
```

### 配置 LLM

在界面的 **Settings** 中选择服务商并填写 API key，或使用环境变量（优先级：环境变量 > 配置文件 > 服务商预设）：

| 变量 | 说明 |
|---|---|
| `CCA_LLM_ENDPOINT` | OpenAI 兼容的 `/chat/completions` 端点 |
| `CCA_LLM_MODEL` | 模型名 |
| `CCA_LLM_API_KEY` | API key |
| `CCA_CONFIG_DIR` | 自定义配置目录 |

事实抽取、评估叙述、来源发现和问答都需要配置 key。

## 主要 API

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/` | 主界面（数据使用审查） |
| POST | `/api/datause/extract` | ① 自然语言 → 结构化事实 |
| POST | `/api/datause/assess` | ② 已确认事实 → 裁定报告 |
| POST | `/api/datause/scope` | 左侧预检助手（帮助梳理场景，不给结论） |
| POST | `/api/datause/chat` | 右侧报告问答助手（解释结论，不推翻结论） |
| GET/POST | `/api/settings` | 读取 / 保存 LLM 配置 |
| POST | `/api/test_connection` | 测试 LLM 连接 |
| GET | `/health` | 健康检查 |

旧版合规引擎端点：`/compliance`、`/api/domain-packs`、`/api/assessments/normalize`、`/api/assessments`。启动后可在 `/docs` 查看完整的交互式 API 文档。

## 测试

在 `app` 的上一级目录运行：

```bash
python -m pytest app/tests
```

测试使用 FastAPI `TestClient`，不需要网络和真实 API key。
