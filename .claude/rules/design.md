# Design Rules — LAR-E

> Full spec: `docs/design/lare-design-prototype.html` and `landing-mockup-v5.html`. No colours, fonts, shadows, or components outside this document.

**Philosophy:** Warm neumorphism. Depth from light/shadow only — no borders. Film grain always on. Inter only. Warm cream base. Ask: *Does this feel made by someone who understands money and relationships?*

---

## CSS Variables — copy verbatim into `:root`

```css
:root {
  --bg:             #EDE7DC;
  --surface:        #F5F0E7;
  --sh-dark:        rgba(163, 140, 105, 0.55);
  --sh-light:       rgba(255, 255, 255, 0.80);
  --neu-raised:     9px 9px 16px var(--sh-dark), -9px -9px 16px var(--sh-light);
  --neu-raised-lg:  12px 12px 22px var(--sh-dark), -12px -12px 22px var(--sh-light);
  --neu-inset:      inset 6px 6px 12px var(--sh-dark), inset -6px -6px 12px var(--sh-light);
  --neu-inset-deep: inset 10px 10px 20px var(--sh-dark), inset -10px -10px 20px var(--sh-light);
  --neu-sm:         5px 5px 10px var(--sh-dark), -5px -5px 10px var(--sh-light);
  --golden:  #D4A44C;  --golden-light: #F5E6C8;
  --ocean:   #7BA8A8;  --palm:         #8FAF82;
  --coral:   #D4896A;  --ink:          #2A2520;
  --drift:   #8A8078;  --mist:         #B5AFA6;
  --night-sand: #1C1916;
  --font: 'Inter', system-ui, sans-serif;
  --r-card: 32px;  --r-btn: 16px;  --r-sm: 12px;
}
```

---

## Typography

| Class | Weight | Size | Case | Tracking | Use |
|-------|--------|------|------|----------|-----|
| `.t-display` | 900 | 64–96px | UPPER | -0.04em | Hero greetings, auth wordmark |
| `.t-heading` | 900 | 18–52px | UPPER | -0.03em | Section titles, stat numbers |
| `.t-subhead` | 700 | context | UPPER | -0.02em | Card sub-titles |
| `.t-label` | 500 | **10px** | UPPER | 0.12em | Labels, nav, badges — never resize |
| `.t-body` | 400 | 14px | normal | — | Descriptions, conversation text |

`.t-muted` → `var(--drift)` · `.t-ghost` → `var(--mist)`. No serif. No size below 9px. Body line-height 1.6, display 0.92.

---

## Shadow Rules

Light from top-left always. Shadows only work on `var(--bg)` — never on white or other backgrounds.

| State | Shadow |
|-------|--------|
| Card default | `var(--neu-raised)` |
| Card hover | `var(--neu-raised-lg)` + `translateY(-2px)` |
| Input / tab container | `var(--neu-inset)` |
| Input focus | `var(--neu-inset-deep)` |
| Small (avatar, icon-well) | `var(--neu-sm)` |
| Button active press | `var(--neu-inset)` + `translateY(0.5px)` |

---

## Film Grain — required on every surface

```css
body::before {
  content: ''; position: fixed; top: -50%; left: -50%;
  width: 200%; height: 200%;
  background-image: url("data:image/svg+xml,%3Csvg viewBox='0 0 256 256' xmlns='http://www.w3.org/2000/svg'%3E%3Cfilter id='n'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='0.85' numOctaves='4' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='100%25' height='100%25' filter='url(%23n)'/%3E%3C/svg%3E");
  opacity: 0.065; pointer-events: none; mix-blend-mode: multiply;
  z-index: 9999; animation: grain 0.5s steps(1) infinite;
}
```
Opacity stays `0.065`. `mix-blend-mode: multiply` required. No `will-change: transform`.

---

## Component Quick-Reference

**Buttons** — base: `11px/700/0.08em/UPPER, border-radius:var(--r-btn), padding:13px 24px`, no border. Variants:
- `.btn-primary` — golden bg, `--night-sand` text
- `.btn-secondary` — `var(--bg)` + `var(--neu-raised)`
- `.btn-ghost` — transparent, no shadow, hover gets `var(--neu-sm)`
- `.btn-danger` — `var(--bg)` + coral text + `var(--neu-raised)`
- `.btn-sm` 9px/18px · `.btn-lg` 12px/36px

**Cards** — `.card`: `border-radius:var(--r-card); box-shadow:var(--neu-raised); padding:32px`. `.card-sm`: `border-radius:20px; padding:18px 24px`. Stat card number: `52px/900/-0.04em`.

**Inputs** — always inset (`var(--neu-inset)`). Focus: `var(--neu-inset-deep)`. No borders. Placeholder: `var(--mist)`. `border-radius:var(--r-btn); padding:14px 18px`.

**Badges** — `9px/700/0.10em/UPPER, padding:3px 9px, border-radius:9999px`. Status colours: active=palm · draft=mist · pending=golden · paused=coral · completed=ocean. Tier: A=golden · B=ocean · C=mist.

**Tabs** — container: `var(--neu-inset)`. Active tab: `var(--neu-raised)`. Both `10px/700/UPPER`. Filter chips smaller (`7px 14px` padding, `var(--neu-sm)` active).

**Nav** — sticky 64px. Links: `10px/500/0.10em/UPPER`, `var(--drift)` default → hover/active `var(--neu-sm)` + `var(--ink)`. Wordmark: `18–20px/900/-0.04em/UPPER` + `lare-face.png` at `mix-blend-mode:multiply`.

**Conversation bubbles** — inbound: `var(--bg)` + `var(--neu-raised)`, `border-bottom-left-radius:5px`. Outbound: golden gradient, `--night-sand` text, `border-bottom-right-radius:5px`. Entry animation: `opacity:0 translateY(8px)` → `opacity:1` at 0.3s.

**Upload zone** — `var(--neu-inset-deep)`, `border:2px dashed rgba(163,140,105,0.28)`, hover → `border-color:var(--golden)`.

**Icon wells** — always recessed (`var(--neu-inset-deep)` 48px / `var(--neu-inset)` 36px). Emoji or SVG only.

**Stepper** — done node: golden bg. Current: `var(--neu-sm)` + `outline:2px solid var(--golden); outline-offset:2px`. Pending: `var(--neu-inset)`.

**Progress bar** — track: `var(--neu-inset)`. Fill: `linear-gradient(90deg, var(--golden), #e8b860)`.

**Tier bar** — A=golden · B=ocean · C=mist segments, widths set inline as %.

---

## Colour Usage

| Colour | Use | Not for |
|--------|-----|---------|
| `--golden` | CTAs, Tier A, active, selected, progress | Body copy |
| `--ocean` | Tier B, replies, completed | CTAs |
| `--palm` | Success, appointments, active badge | Errors |
| `--coral` | Danger, paused, sign out | Decoration |
| `--ink` | All primary text, headings | Backgrounds |
| `--drift` | Labels, metadata, secondary | Headings |
| `--mist` | Placeholders, ghost, disabled | Readable labels |
| `--night-sand` | Text on `--golden` backgrounds | Text on `--bg` |

---

## Layout

Container: `max-width:1200px; padding:0 48px`. Page: `padding-top:56px; padding-bottom:96px`. Nav: 64px sticky. Grid gaps: 20px. Breakpoints: 960px (4/3-col → 2-col), 600px (all → 1-col, padding → 20px).

---

## lare-face.png

Always `mix-blend-mode: multiply`. Only on `--bg`. Sizes: auth 92px · nav 28–30px · hero 220px · CTA 84px · widget 44px circle.

---

## Key Animations

| Name | Duration | Effect |
|------|----------|--------|
| `face-develop` | 2.4s ease-out | blur+saturate darkroom reveal — auth/hero |
| `breathe` | 4–5s infinite | `scale(1)↔scale(1.022)` — face illustration |
| `bubble-in` | 0.3s | `translateY(8px) opacity:0` → resting — chat bubbles |
| `grain` | 0.5s steps(1) infinite | random translate — film flicker |
| `pulse-ring` | 3s infinite | golden box-shadow expand/fade — CTA ring |
| `scroll-pulse` | context | scaleY reveal/fade — scroll hint |

---

## Forbidden — never do any of these

- No white backgrounds — `var(--bg)` only
- No borders — depth from shadows only
- No flat colour for elevation — shadows separate layers
- No single-colour box-shadow — always dark + light pair
- No serif fonts — Inter only
- No colours outside the palette
- No `font-size` below 9px
- No bright/saturated colours
- No `border-radius` below 6px on interactive elements
- No `outline` except stepper current node
- Never remove the film grain
- Never apply neumorphic shadows on non-`--bg` backgrounds
