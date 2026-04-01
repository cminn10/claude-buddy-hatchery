# Claude Buddy Hatchery

A reverse-engineered reimplementation of the [Claude Code](https://docs.anthropic.com/en/docs/claude-code) companion hatching algorithm. Explore the buddy system, hunt for rare companions, or crack an ID that gives you the exact buddy you want.

English | [中文](./README.zh-CN.md)

## Quick Start

```bash
curl -fsSL https://raw.githubusercontent.com/cminn10/claude-buddy-hatchery/main/hatch.py -o hatch.py
python3 hatch.py --crack          # interactive mode — pick your dream buddy
python3 hatch.py --hunt           # explore random rare companions
python3 hatch.py                  # hatch a random buddy
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

## How It Works

Claude Code 2.1.89 introduced `/buddy` — a virtual companion that lives in your terminal. The companion's traits (species, rarity, stats, etc.) are **deterministically derived from your account identity**, not random per hatch.

The algorithm chain:

```
account ID + "friend-2026-401"  →  Bun.hash (wyhash)  →  32-bit seed
                                                              ↓
                                                        SplitMix32 PRNG
                                                              ↓
                                                  rarity, species, eye, hat,
                                                  shiny, stats, name hint
```

Only `name` and `personality` come from an LLM call during hatching. Everything else is a pure function of the hash seed.

### Account types and seed source

The binary determines which ID to use as the seed with this priority chain:

```javascript
oauthAccount?.accountUuid ?? userID ?? "anon"
```

| Account type | Seed field | ID format | Persists across sessions? |
|---|---|---|---|
| **Subscription** (Pro/Team) | `oauthAccount.accountUuid` | UUID (`xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`) | Yes, until next OAuth login |
| **API key** (custom routing) | `userID` | hex64 (64 hex chars) | Yes |

Both fields live in `.claude.json`. The `--crack` command generates IDs in both formats.

### What's in the binary

| Component | Details |
|-----------|---------|
| **Species** | duck, goose, blob, cat, dragon, octopus, owl, penguin, turtle, snail, ghost, axolotl, capybara, cactus, robot, rabbit, mushroom, chonk |
| **Rarity** | common (60%), uncommon (25%), rare (10%), epic (4%), legendary (1%) |
| **Shiny** | 1% chance, independent of rarity |
| **Stats** | DEBUGGING, PATIENCE, CHAOS, WISDOM, SNARK — budget scales with rarity |
| **Hats** | crown, tophat, propeller, halo, wizard, beanie, tinyduck |
| **Eyes** | `·` `✦` `×` `◉` `@` `°` |

## Commands

### `python3 hatch.py [seed]` — hatch a buddy

```bash
python3 hatch.py                    # random seed
python3 hatch.py "any-string"       # deterministic from seed
```

### `--hunt [traits]` — search for rare companions

Brute-force random seeds to find companions matching target traits.

```bash
python3 hatch.py --hunt                     # interactive — pick from menus
python3 hatch.py --hunt legendary           # find any legendary
python3 hatch.py --hunt "shiny capybara"    # find a shiny capybara
python3 hatch.py --hunt "legendary dragon"  # find a legendary dragon
```

### `--crack [traits]` — crack an ID (requires [Bun](https://bun.sh))

Find a cracked ID that produces your dream companion. Outputs both UUID (for subscription accounts) and hex64 (for API key accounts).

```bash
python3 hatch.py --crack                            # interactive — pick from menus
python3 hatch.py --crack "legendary shiny dragon"   # silent mode
```

Interactive mode lets you select rarity, species, shiny, hat, and eye from menus. Silent mode parses traits from the string.

```
Cracking UUID (for subscription accounts)...
Cracking: legendary shiny dragon (format: uuid)
Odds: ~1/180,000 per attempt — using Bun.hash (native speed)

Cracked in 9,174 attempts (0s)

Cracked accountUuid:
3148d224-4d36-bc8b-955e-02db9b17d7dd

──────────────────────────────────────────────────
Cracking hex64 (for API key accounts)...

Cracked userID:
83d020d639c814a3a2217d69bac9f486e622fd0f16894c193601d65c2f0653b4
```

### `--apply <cracked_id> [config_path]` — patch config

Automatically patches the right field in `.claude.json` based on your account type:

- **If `oauthAccount` exists** in your config → patches `oauthAccount.accountUuid` (expects UUID)
- **If no `oauthAccount`** → patches `userID` (expects hex64)

Also deletes the `companion` key so `/buddy` will re-hatch.

```bash
# Auto-detect config location (prompts if multiple found)
python3 hatch.py --apply "3148d224-4d36-bc8b-955e-02db9b17d7dd"

# Explicit config path
python3 hatch.py --apply "3148d224-4d36-bc8b-955e-02db9b17d7dd" ~/.claude/.claude.json
```

Config file is searched in this order:
1. `$CLAUDE_CONFIG_DIR/.claude.json` (if env var is set)
2. `~/.claude/.claude.json` (default `CLAUDE_CONFIG_DIR`)
3. `~/.claude.json`

### Manual apply (without `--apply`)

If you prefer to edit the config manually:

1. Find your `.claude.json` (see locations above)
2. **Subscription account:** set `oauthAccount.accountUuid` to the cracked UUID
3. **API key account:** set `userID` to the cracked hex64
4. Delete the `"companion"` key (so `/buddy` re-hatches with new traits)
5. Run `/buddy` in Claude Code

> **Note:** The name and personality are generated by an LLM call during hatching, so those will be unique each time. The species, rarity, stats, shiny, hat, and eye are deterministic from the seed.

## How the crack works

1. **The hash is only 32-bit.** `Bun.hash()` returns a wyhash truncated to uint32 — just ~4.3 billion possible seeds.
2. **Traits are deterministic from the seed.** Given a seed, the PRNG sequence (and thus all companion traits) are fixed.
3. **Brute-force is fast.** For legendary+shiny+dragon (~1/180k odds), a native Bun script finds a matching ID in under a second.

### Known limitation: subscription accounts

For subscription users, `oauthAccount.accountUuid` is set by the OAuth login flow. Modifying this field works and persists **until the next OAuth login/token refresh**, at which point it gets overwritten with the real account UUID.

**Workaround:** after each OAuth login, re-run `--apply` with your cracked UUID. The buddy bones are cached per session, so once you've applied and run `/buddy`, the companion sticks for that session.

## The hash red herring

The binary contains *two* hash implementations:

```javascript
function aN4(H) {
  if (typeof Bun < "u")
    return Number(BigInt(Bun.hash(H)) & 0xffffffffn);  // ← Bun runtime path
  // FNV-1a fallback (dead code in compiled binary)
  let _ = 2166136261;
  for (let q = 0; q < H.length; q++)
    _ ^= H.charCodeAt(q), _ = Math.imul(_, 16777619);
  return _ >>> 0;
}
```

Since Claude Code is compiled with Bun, it always takes the `Bun.hash()` (wyhash) path. The FNV-1a code is dead. We initially implemented FNV-1a, got wrong results, and discovered the real hash by testing against a known companion.

## Requirements

- Python 3.10+
- [Bun](https://bun.sh) (required for `--crack` and `--apply` verification; `--hunt` and basic hatching use it when available, fall back to FNV-1a otherwise)

## Disclaimer

This is a fun reverse-engineering project for educational purposes. The buddy system is an April 2026 feature of Claude Code by Anthropic. All trademarks belong to their respective owners.
