# codex-imagegen-skill

从 Codex 的 `imagegen` skill 逆向还原出来的独立副本，可直接装到 Claude Code、Codex 或其他支持 `SKILL.md` 约定的 agent 上使用。

> skill 内部名称仍然是 `imagegen` —— `SKILL.md` 的 front matter 和 `agents/openai.yaml` 里的 `$imagegen` 触发词都依赖它，不要改。仓库名只用来标识这份分发。

围绕「让模型自己判断该生成还是编辑图片、并按结构化 prompt 规格产出可用的位图资产」设计，包含生成/编辑决策树、用例 taxonomy、prompt 规格模板和一套可选的 CLI 实现。

## 目录结构

```
.claude/skills/imagegen/          # Claude Code 读取路径
.agents/skills/imagegen/          # 通用 agent skills 约定
.github/skills/imagegen/          # 随仓库分发的副本
└── SKILL.md                      # 主入口：模式、决策树、工作流、prompt 规格
    agents/openai.yaml            # 宿主 UI 元数据（显示名、简介、默认 prompt）
    references/
        prompting.md              # 两种模式共用的 prompt 原则
        sample-prompts.md         # 可直接复制的 prompt 配方（按资产类型分类）
        cli.md                    # CLI fallback 用法
        image-api.md              # API/CLI 参数速查
        codex-network.md          # CLI 模式的网络与沙箱配置
    scripts/
        image_gen.py              # CLI fallback 实现（generate / edit / generate-batch）
        remove_chroma_key.py      # 把纯色背景抠成透明通道的辅助脚本
    LICENSE.txt                   # Apache License 2.0
```

三处副本内容完全一致。只用其中一种宿主的话，删掉另外两份即可。

## 安装

```bash
git clone https://github.com/huangnaisheng/codex-imagegen-skill.git
cd codex-imagegen-skill
```

然后把 `imagegen/` 目录放进宿主的 skills 路径：

```bash
# Claude Code（个人级）
cp -r .claude/skills/imagegen ~/.claude/skills/

# Claude Code（项目级）
cp -r .claude/skills/imagegen .claude/skills/

# Codex
cp -r .claude/skills/imagegen ~/.codex/skills/
```

装好后宿主会自动读取 `SKILL.md` 的 front matter，在合适时机触发。

## 两种模式（重要）

Skill 定义了两个顶层模式：

| 模式 | 说明 | 需要 API key |
| --- | --- | --- |
| **内置工具模式**（原文优先） | 调用宿主自带的 `image_gen` 工具 | 否 |
| **CLI fallback 模式** | 走 `scripts/image_gen.py` | 是，`OPENAI_API_KEY` |

> **在 Claude Code 下的注意点**
>
> `SKILL.md` 原文假定宿主提供内置 `image_gen` 工具和 `$CODEX_HOME` 目录，这是 Codex 特有的。Claude Code 没有这个内置工具，因此**实际可用的是 CLI fallback 模式**，必须先配好 `OPENAI_API_KEY`。
>
> 想在这个环境下用得更顺，可以在 `SKILL.md` 里把「默认走内置工具」的措辞调整成「默认走 CLI」，否则模型可能先尝试不可用的内置路径。

## CLI fallback 用法

### 依赖

```bash
uv pip install openai          # 必需
uv pip install pillow          # 可选：图片检查与降采样
```

### 环境变量

```bash
export OPENAI_API_KEY="sk-..."
```

不要把 key 写进任何文件或命令参数。脚本本身不含、也不会记录 key。

如需走自定义端点（例如自建中转），设 `OPENAI_BASE_URL` 即可——底层 OpenAI SDK 会自动读取，无需改动脚本：

```bash
export OPENAI_BASE_URL="https://your-endpoint.example.com/v1"
```

### 常用命令

三个子命令：`generate`、`edit`、`generate-batch`。

```bash
CLI=".claude/skills/imagegen/scripts/image_gen.py"

# 先干跑，确认 endpoint、model、尺寸、输出路径都正确
python "$CLI" generate --prompt "a ceramic coffee mug on a linen tablecloth" --dry-run

# 真正生成
python "$CLI" generate \
  --prompt "a ceramic coffee mug on a linen tablecloth" \
  --use-case product-mockup \
  --scene "minimal studio, soft daylight" \
  --size 1536x1024 \
  --quality high \
  --out output/mug.png

# 编辑：只改指定部分，其余保持不变
python "$CLI" edit \
  --image input/mug.png \
  --prompt "replace only the background with a warm sunset gradient; keep the product and its edges unchanged" \
  --out output/mug-sunset.png

# 批量：从一个 JSONL 文件跑多个任务
python "$CLI" generate-batch --input prompts.jsonl --out-dir output/batch/
```

主要参数：`--model` `--size` `--quality` `--background` `--output-format`
`--output-compression` `--moderation` `--input-fidelity` `--mask` `--force`
`--no-augment` `--concurrency` `--max-attempts` `--dry-run`

完整清单见 [`references/cli.md`](.claude/skills/imagegen/references/cli.md) 和
[`references/image-api.md`](.claude/skills/imagegen/references/image-api.md)。

### 模型与尺寸

CLI 默认 `gpt-image-2`。常用尺寸：

| 尺寸 | 用途 |
| --- | --- |
| `1024x1024` | 方形草稿，最快 |
| `1536x1024` / `1024x1536` | 横版 / 竖版 |
| `2048x2048` | 2K 方形 |
| `3840x2160` / `2160x3840` | 4K 横版 / 竖版 |

约束：最长边 `<= 3840px`，两边均为 `16` 的倍数，长宽比 `<= 3:1`，总像素在 `655,360` 到 `8,294,400` 之间。

注意 `gpt-image-2` **不支持** `background=transparent`。需要真透明时得用 `gpt-image-1.5`，且应事先与用户确认再切换——skill 明确禁止静默换模型。

### 透明背景的两条路

1. **CLI 真透明**：`gpt-image-1.5` + `--background transparent`（需先确认换模型）
2. **色键抠图**：用 `gpt-image-2` 在纯色背景上生成，再用 `remove_chroma_key.py` 转成 alpha

```bash
python .claude/skills/imagegen/scripts/remove_chroma_key.py \
  --input output/mug-flat.png --out output/mug-transparent.png
```

## Prompt 规格

两种模式共用同一套标签式结构。写得越具体，模型越不该自由发挥：

```text
Use case: <taxonomy slug>
Asset type: <资产用在哪里>
Primary request: <用户原始需求>
Input images: <Image 1: 角色; Image 2: 角色>
Scene/backdrop: <环境>
Subject: <主体>
Style/medium: <摄影/插画/3D>
Composition/framing: <构图与画幅>
Lighting/mood: <光线与氛围>
Color palette: <色板>
Text (verbatim): "<需要逐字渲染的文字>"
Constraints: <必须保留 / 必须避免>
Avoid: <负向约束>
```

编辑类请求要每次都重申不变量（`change only X; keep Y unchanged`），否则多轮迭代容易漂移。

用例 taxonomy（slug 需与 prompt 保持一致）和更多示例见
[`references/prompting.md`](.claude/skills/imagegen/references/prompting.md)、
[`references/sample-prompts.md`](.claude/skills/imagegen/references/sample-prompts.md)。

## 已知限制

- 原文多处假定 Codex 运行时（内置 `image_gen` 工具、`view_image` 工具、`$CODEX_HOME` 输出目录）。换到其他宿主时这些描述不成立，需按上文的注意点调整。
- `scripts/image_gen.py` 是 skill 的组成部分，原文明确要求**不要修改**它。缺功能应改 `SKILL.md` 或先与使用者确认。
- 所有 Python 脚本都不含凭据，也从不打印 key。

## 许可

本目录下的 `LICENSE.txt` 为 **Apache License 2.0**。分发或再发布时请保留该文件及原始版权声明——这是 Apache-2.0 的强制要求
