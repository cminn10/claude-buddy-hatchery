# Claude Buddy Hatchery

对 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) 编程伙伴孵化算法的逆向工程复刻。探索扭蛋系统、搜索稀有伙伴，或破解出一个能让你获得梦想伙伴的 ID。

[English](./README.md) | 中文

## 快速开始

```bash
curl -fsSL https://raw.githubusercontent.com/cminn10/claude-buddy-hatchery/main/hatch.py -o hatch.py
python3 hatch.py --crack          # 交互模式 — 选择你的梦想伙伴
python3 hatch.py --hunt           # 探索随机稀有伙伴
python3 hatch.py                  # 随机孵化一个伙伴
```

```
  ┌────────────────────────────────────────┐
  │ ★★★★★ LEGENDARY         DRAGON        │
  │                                        │
  │                  -+-                    │
  │                /^\  /^\                 │
  │               <  ×  ×  >               │
  │               (   ~~   )               │
  │                `-vvvv-´                 │
  │                                        │
  │  Biscuit                               │
  │  "A legendary dragon of few words."    │
  │  ✨ SHINY ✨                            │
  │                                        │
  │  DEBUGGING  ███████░░░  69             │
  │  PATIENCE   █████░░░░░  53             │
  │  CHAOS      ███████░░░  74             │
  │  WISDOM     ████████░░  79             │
  │  SNARK      ██████████ 100             │
  └────────────────────────────────────────┘
```

## 原理

Claude Code 2.1.89 引入了 `/buddy` 功能 — 一个住在终端里的虚拟编程伙伴。伙伴的属性（物种、稀有度、数值等）是**由账号 ID 确定性生成的**，不是每次随机抽取。

算法链路：

```
账号 ID + "friend-2026-401"  →  Bun.hash (wyhash)  →  32位种子
                                                          ↓
                                                    SplitMix32 PRNG
                                                          ↓
                                              稀有度、物种、眼睛、帽子、
                                              闪光、数值、名字提示
```

只有 `名字` 和 `性格描述` 来自 LLM 生成。其他所有属性都是哈希种子的纯函数。

### 账号类型与种子来源

二进制中通过以下优先级链决定使用哪个 ID 作为种子：

```javascript
oauthAccount?.accountUuid ?? userID ?? "anon"
```

| 账号类型 | 种子字段 | ID 格式 | 跨会话持久化？ |
|---|---|---|---|
| **订阅账号**（Pro/Team） | `oauthAccount.accountUuid` | UUID（`xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`） | 是，直到下次 OAuth 登录 |
| **API Key**（自定义路由） | `userID` | hex64（64位十六进制字符） | 是 |

两个字段都在 `.claude.json` 中。`--crack` 命令会同时生成两种格式的 ID。

### 从二进制中提取的数据

| 组件 | 详情 |
|------|------|
| **物种** | duck, goose, blob, cat, dragon, octopus, owl, penguin, turtle, snail, ghost, axolotl, capybara, cactus, robot, rabbit, mushroom, chonk |
| **稀有度** | common（60%）、uncommon（25%）、rare（10%）、epic（4%）、legendary（1%） |
| **闪光** | 1% 概率，独立于稀有度 |
| **数值** | DEBUGGING、PATIENCE、CHAOS、WISDOM、SNARK — 预算随稀有度提升 |
| **帽子** | crown, tophat, propeller, halo, wizard, beanie, tinyduck |
| **眼睛** | `·` `✦` `×` `◉` `@` `°` |

## 命令

### `python3 hatch.py [seed]` — 孵化伙伴

```bash
python3 hatch.py                    # 随机种子
python3 hatch.py "any-string"       # 从种子确定性生成
```

### `--hunt [traits]` — 搜索特定伙伴

暴力搜索随机种子，寻找匹配目标属性的伙伴。

```bash
python3 hatch.py --hunt                     # 交互模式 — 从菜单选择
python3 hatch.py --hunt legendary           # 寻找任意传说
python3 hatch.py --hunt "shiny capybara"    # 寻找闪光水豚
python3 hatch.py --hunt "legendary dragon"  # 寻找传说龙
```

### `--crack [traits]` — 破解 ID（需要 [Bun](https://bun.sh)）

找到一个能生成你梦想伙伴的 ID。同时输出 UUID（订阅账号）和 hex64（API Key 账号）两种格式。

```bash
python3 hatch.py --crack                            # 交互模式 — 从菜单选择
python3 hatch.py --crack "legendary shiny dragon"   # 静默模式
```

交互模式可以依次选择稀有度、物种、闪光、帽子和眼睛。静默模式从字符串中解析属性。

```
破解 UUID（订阅账号）...
Cracking: legendary shiny dragon (format: uuid)
Odds: ~1/180,000 per attempt

Cracked in 9,174 attempts (0s)

Cracked accountUuid:
3148d224-4d36-bc8b-955e-02db9b17d7dd

──────────────────────────────────────────────────
破解 hex64（API Key 账号）...

Cracked userID:
83d020d639c814a3a2217d69bac9f486e622fd0f16894c193601d65c2f0653b4
```

### `--apply <cracked_id> [config_path]` — 写入配置

根据账号类型自动修改 `.claude.json` 中的正确字段：

- **如果配置中有 `oauthAccount`** → 修改 `oauthAccount.accountUuid`（需要 UUID 格式）
- **如果没有 `oauthAccount`** → 修改 `userID`（需要 hex64 格式）

同时删除 `companion` 键，让 `/buddy` 重新孵化。

```bash
# 自动检测配置位置（如果有多个会提示选择）
python3 hatch.py --apply "3148d224-4d36-bc8b-955e-02db9b17d7dd"

# 指定配置文件路径
python3 hatch.py --apply "3148d224-4d36-bc8b-955e-02db9b17d7dd" ~/.claude/.claude.json
```

配置文件按以下顺序搜索：
1. `$CLAUDE_CONFIG_DIR/.claude.json`（如果设置了环境变量）
2. `~/.claude/.claude.json`（默认 `CLAUDE_CONFIG_DIR`）
3. `~/.claude.json`

### 手动修改（不使用 `--apply`）

如果你想手动编辑配置文件：

1. 找到你的 `.claude.json`（位置见上文）
2. **订阅账号：** 将 `oauthAccount.accountUuid` 设为破解的 UUID
3. **API Key 账号：** 将 `userID` 设为破解的 hex64
4. 删除 `"companion"` 键（让 `/buddy` 用新属性重新孵化）
5. 在 Claude Code 中运行 `/buddy`

> **注意：** 名字和性格描述由 LLM 在孵化时生成，每次都不同。物种、稀有度、数值、闪光、帽子和眼睛由种子确定性决定。

## 破解原理

1. **哈希只有 32 位。** `Bun.hash()` 返回的 wyhash 被截断为 uint32 — 只有约 43 亿种可能的种子。
2. **属性由种子确定。** 给定种子，PRNG 序列（以及所有伙伴属性）是固定的。
3. **暴力破解很快。** 对于 legendary+shiny+dragon（约 1/18万 概率），原生 Bun 脚本在不到一秒内就能找到匹配的 ID。

### 已知限制：订阅账号

对于订阅用户，`oauthAccount.accountUuid` 由 OAuth 登录流程设置。修改该字段后**在下次 OAuth 登录/令牌刷新之前都有效**，之后会被覆盖为真实的账号 UUID。

**解决方法：** 每次 OAuth 登录后，重新运行 `--apply` 即可。Buddy 的骨架数据在每个会话中只计算一次并缓存，所以一旦 apply 并运行了 `/buddy`，伙伴就会在该会话中保持不变。

## 哈希的烟幕弹

二进制中包含*两种*哈希实现：

```javascript
function aN4(H) {
  if (typeof Bun < "u")
    return Number(BigInt(Bun.hash(H)) & 0xffffffffn);  // ← Bun 运行时路径
  // FNV-1a 回退（编译后的二进制中是死代码）
  let _ = 2166136261;
  for (let q = 0; q < H.length; q++)
    _ ^= H.charCodeAt(q), _ = Math.imul(_, 16777619);
  return _ >>> 0;
}
```

由于 Claude Code 使用 Bun 编译，它始终走 `Bun.hash()`（wyhash）路径。FNV-1a 代码是死代码。我们最初实现了 FNV-1a，得到了错误的结果，通过对比已知伙伴的输出才发现了真正的哈希函数。

## 环境要求

- Python 3.10+
- [Bun](https://bun.sh)（`--crack` 和 `--apply` 验证必需；`--hunt` 和基本孵化在有 Bun 时使用，否则回退到 FNV-1a）

## 免责声明

本项目是出于教育目的的逆向工程研究。Buddy 系统是 Anthropic 的 Claude Code 2026年4月推出的功能。所有商标归其各自所有者所有。
