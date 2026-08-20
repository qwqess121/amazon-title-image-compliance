# amazon-title-image-compliance

按亚马逊 2025-01-21 生效的**商品名称新规**（或需求人自定义规则）批量清洗/改写商品标题、批量重命名商品图片，并输出合规报告。规则由 `rules.json` 配置驱动，可被需求人附件覆盖；配合 SP-API Feeds 可批量上传到亚马逊店铺。

**双模式**：
- **合规模式 `clean-titles`**：确定性规则清洗（超长/禁用符号/促销语/重词/全大写），保证符合硬规则。
- **改写模式 `optimize-titles`**：集成 `amazon-listing-optimization`（nexscope-ai/Amazon-Skills, MIT）标题方法论——`[品牌] + [主关键词] + [属性] + [差异化]`，主关键词前置、≤200 字符、禁促销语，输出 before→after。

---

## 目录结构

```
amazon-title-image-compliance/
├── SKILL.md                        ← skill 说明（Agent 读取此文件识别能力）
├── config/
│   └── rules.example.json          ← 规则配置模板（需求人规则来了改这里）
├── examples/
│   ├── products.csv                ← 合规模式样例（5 条违规标题）
│   ├── products_optimize.csv       ← 改写模式样例（含品牌+主关键词）
│   └── map.csv                     ← 图片重命名映射样例
├── references/
│   └── rules.md                    ← 默认规则与覆盖说明
└── scripts/
    ├── amazon_compliance.py        ← 核心引擎（4 个子命令）
    └── selftest.py                 ← 一键自测
```

---

## 一、Claude 怎么安装这个 skill

本 skill 采用 **Anthropic Agent Skills 格式**（一个文件夹 + `SKILL.md`），Claude Code / Claude Desktop 及其它支持该格式的 Agent（Codex、Cursor、Windsurf、OpenClaw 等）均可直接安装。

### 方式 1：git clone（推荐）

```bash
# 1) 克隆仓库
git clone https://github.com/qwqess121/amazon-title-image-compliance.git

# 2) 复制到 Claude 的 skills 目录
#    Windows（PowerShell）：
Copy-Item -Recurse amazon-title-image-compliance $HOME\.claude\skills\
#    macOS / Linux：
cp -r amazon-title-image-compliance ~/.claude/skills/
```

### 方式 2：手动下载

1. 打开 https://github.com/qwqess121/amazon-title-image-compliance → **Code → Download ZIP**
2. 解压得到文件夹 `amazon-title-image-compliance`
3. 把整个文件夹放进 Claude 的 skills 目录（见下方位置说明）

### 安装位置

| 使用场景 | 目录 |
|---|---|
| 所有项目通用（个人级） | `~/.claude/skills/amazon-title-image-compliance/` |
| 仅当前项目（项目级） | `<项目根目录>/.claude/skills/amazon-title-image-compliance/` |

> 装到 `~/.claude/skills/` 后，所有项目的 Claude 都能用；装到项目 `.claude/skills/` 则只在该项目生效。

### 验证安装

```bash
# 1) 确认 SKILL.md 在位
ls ~/.claude/skills/amazon-title-image-compliance/SKILL.md

# 2) 一键自测（会临时生成样例数据跑通全链路）
cd ~/.claude/skills/amazon-title-image-compliance
python scripts/selftest.py
# 期望输出: [PASS] make-rules / clean-titles / optimize-titles / rename-images → 全部通过 ✅
```

### 在 Claude 里使用

安装后**重启 Claude 会话**（新会话才会加载新 skill），然后直接说：
- "批量改亚马逊标题，符合新标题格式"
- "按亚马逊规则批量改图片命名"
- "检查这些标题是否符合亚马逊新规"
- "把店铺标题清洗成 200 字符内、去符号、去重词"

Claude 会自动匹配本 skill 并执行。

---

## 二、命令行用法（不依赖 Agent 也可用）

```bash
# 导出默认规则模板
python scripts/amazon_compliance.py make-rules --out rules.json

# 合规模式：批量清洗标题 → cleaned.csv（含合规报告）
python scripts/amazon_compliance.py clean-titles \
  --input products.csv --output cleaned.csv \
  --rules rules.json --key-col sku --title-col item_name

# 改写模式：品牌 + 主关键词前置 → optimized.csv（before→after）
python scripts/amazon_compliance.py optimize-titles \
  --input products.csv --output optimized.csv \
  --brand-col brand --keyword-col primary_keyword --rules rules.json

# 批量重命名图片（复制到输出目录，不删原件）
python scripts/amazon_compliance.py rename-images \
  --mapping map.csv --image-dir ./imgs --output-dir ./renamed --rules rules.json
```

依赖：Python 3.8+，纯标准库；读 `.xlsx` 需 `pip install openpyxl`。

---

## 三、规则配置（需求人规则怎么覆盖）

规则全部由 `rules.json` 驱动，需求人附件规则只需改配置，引擎逻辑不动：

| 字段 | 默认 | 说明 |
|---|---|---|
| `title.max_length` | 200 | 标题最大字符数（部分类目更短，如 80/150） |
| `title.forbidden_symbols` | `! $ ? _ { } ^ ¬ ¦ ~ # < > *` | 禁用符号（品牌名含符号时删掉该符号） |
| `title.exempt_small_words` | in/on/over/with/and/or/for/the/a/an/of/to/by/from | 豁免小词（不判重、非首词小写） |
| `title.promo_phrases` | free shipping / best seller / 100% quality ... | 促销语库，命中整段移除 |
| `title.single_word_promo` | super / premium / new / best / top ... | 单字促销词（改写模式剔除） |
| `title.max_word_repeat` | 2 | 同一词允许次数（>2 移除） |
| `image.key_field` | sku | 图片命名主键（可改 asin） |
| `image.format` / `image.padding` | jpg / 2 | 扩展名、附图序号位数（`_01`） |

完整说明见 `references/rules.md`。

---

## 四、批量上传到亚马逊店铺

本 skill 负责「出合规结果」，上架走 `linkfox-amazon-store-operations`（SP-API 通道）：

1. **授权前置**：需 `LINKFOX_AGENT_API_KEY` + 店铺 OAuth（`store_tokens.py` 取 `accessToken`）
2. **整理 flat file**：`cleaned.csv` / `optimized.csv` 转成 `item_sku` + `item_name` 模板
3. **Feeds 上传**：`create_feed_document` → `upload_feed_document` → `create_feed` → `get_feed` 轮询 → `get_feed_document`
4. **无 API 备选**：卖家中心「库存 > 批量上传商品 > 部分更新」手动上传

> 上传前必须确认 `sellerId / sku / ASIN / marketplaceIds / feedType` 等关键参数，写操作不可误改误删。

---

## 官方依据

- 亚马逊商品名称新规：2025-01-21 生效，适用所有卖家与品类（媒体类目除外），覆盖所有站点。
- 各站点/类目细节以卖家平台帮助页「商品名称要求和指南」为准。

## License

MIT
