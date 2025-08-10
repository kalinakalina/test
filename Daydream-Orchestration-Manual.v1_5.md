# Daydream Orchestration Manual — v1.5 (Payoneer)

**Generated:** 2025-08-10T06:38:05.984931Z

## Overview
Daydream uses three cooperating agents to turn a prompt or screenshot into pixel-faithful, DS-compliant UI:
1. **Gatherer** — collects/normalizes inputs (prompt, screenshots, product name), enriches with context (DS tokens/components/flows).
2. **Planner** — selects a **hard‑wired template** and emits a **plan.json** (no free-styling).
3. **Builder** — renders from **plan.json** using **only DS tokens/components** and runs **acceptance checks**. If anything is missing, it fails fast with `NEEDS_ICON`, `NEEDS_ASSET`, `TOKEN_VIOLATION`, or `LAYOUT_ACCEPTANCE_FAIL`.

### Golden Rules
- *Minimal logic, maximal imitation.*
- Builder never invents styles or icons. It only uses the DS.
- If Planner is uncertain, route to a **fallback template family** (FormPage, EntityList, DetailPage).

---

## Agent APIs (for ChatGPT-5 API integration)

### Router (tiny step before Gatherer)
**Input:** free‑text prompt (or screenshot tag), optional screen hint  
**Output:** `{intent, entities, confidence}`  
**Intents:** `home`, `activity`, `convert`, `send`, `request`, `add_funds`, `select_method`, `cards`, `settings`

### Gatherer
- Attaches **DS master** (`Payoneer-DS-Master.v1_5.json`) to the context.
- If screenshot present: fingerprints to closest hard‑wired template and extracts props (currency, amounts, titles, dates) with rules-of-thumb.
- Emits: `{intent, entities, ds_version, assets}`

### Planner
- Picks **templateId** from DS templates by intent + entities.
- Fills props, leaves placeholders where uncertain.
- Emits **plan.json** with:
```json
{
  "template": "payoneer.home.v2",
  "tokensLock": true,
  "props": {
    "HeaderRow.name": "Hi Elinor",
    "CurrencyTile[0].flag": "US",
    "CurrencyTile[0].currencyCode": "USD",
    "CurrencyTile[0].label": "USD balance",
    "CurrencyTile[0].amount": "$240,000.00"
  },
  "acceptance": ["..."],
  "gaps": []
}
```

### Builder
- Validates: **no inline styles**, **only DS tokens**, **icon map resolved**.
- Renders HTML/JSX.
- Runs acceptance checks from template + DS-wide guardrails.
- On failure, returns structured error with a **fix hint** for Planner.

### Validator (post-render)
- Asserts acceptance again; emits success/failure + diffs (e.g., currency code not end-aligned).

---

## Common Flows (hard‑wired)
- **Home** → `payoneer.home.v2`
- **Activity** → `payoneer.activity.v1`
- **Convert / Manage Currencies** → `payoneer.convert.v1`
- **Send** → `payoneer.send.v1` (+ `payoneer.send.v1.review`)
- **Request** → `payoneer.request.v1`
- **Add Funds** → `payoneer.addfunds.v1` (+ empty variant)

**Principle:** When in doubt, **fail fast** and return gaps for Planner to fix (don’t improvise).

---

## DS Guardrails (Builder MUST enforce)
- Only use **tokens** from DS (colors, spacing, radii, shadows, typography).
- Only use **components** specified (HeaderRow, CurrencyTile, CurrencyBadge, SectionHeader, TransactionList, DateChip, TabBar, Card, Field, Select, Input, StickyActions, EmptyState, Divider).
- Use **icon tokens** from `tokens.icon.map` (`home`, `activity`, `actions`, `cards`, `notification`, `send`, `receive`, `exchange`, `bank`, `search`, `filter`).

**If a required icon is missing:** return `NEEDS_ICON(name)`.

---

## Example End-to-End (Prompt-first)
**Prompt:** “open convert and set 250 eur to usd”  
1. Router → intent `convert`, entities `{amount: '€250', from:'EUR', to:'USD'}`  
2. Gatherer → loads DS v1.5, returns assets.  
3. Planner → template `payoneer.convert.v1`, fills tokens: EUR→USD tiles, rate placeholder, CTA 'Continue'.  
4. Builder → renders with CurrencyBadge (flag-left 32px, code right-aligned minWidth 42), sticky CTA, Actions tab active.  
5. Validator → checks acceptance list, returns success.

---

## Error Codes
- `TOKEN_VIOLATION` — a color/spacing/font outside DS was used.
- `LAYOUT_ACCEPTANCE_FAIL` — a required element is missing or misaligned.
- `NEEDS_ICON(name)` — icon not mapped in sprite.
- `NEEDS_ASSET(path)` — referenced asset missing.

---

## Orchestration JSON (drop-in)
See `Orchestration-Chain.json` in the v1.5 kit (same as v1.4 with icon/token references updated).

---

## Delivery Contract
- DS is the authority. Planner & Builder use **only** what’s in the DS.
- If Planner can’t fill a prop, leave placeholder + add to `gaps`.
- If Builder can’t resolve an icon, return `NEEDS_ICON` (no substitutes).
