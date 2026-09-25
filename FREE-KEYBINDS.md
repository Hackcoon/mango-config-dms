# Free keybinds (fury) — generated 2026-09-25 from `mango/config.conf` (126 binds, DMS-only)

> Regenerate after any bind change: `rg -n "^(bind|mousebind|axisbind)=" ~/.config/mango/config.conf`.
> Bare-`SUPER` letters are EXHAUSTED — all 26 taken.

Key names are mango/XKB style (`semicolon` = `;`, `slash` = `/`,
`apostrophe` = `'`, `grave` = `` ` ``, `minus` = `-`, `equal` = `=`).

## SUPER (single keys — nothing left)

| Status | Keys |
|---|---|
| Taken | `a b c d e f g h i j k l m n o p q r s t u v w x y z, semicolon (;), 1-9, space, return, tab, comma, period, arrows` |
| FREE | ``0, slash (/), apostrophe ('), minus (-), equal (=), bracketleft ([), bracketright (]), backslash (\), grave (`)`` |

## SUPER+SHIFT (best layer for new binds)

| Status | Keys |
|---|---|
| Taken | `b c f g h i j n o p q r s u v y z, 1-9, return, tab, period, bracketleft, bracketright, arrows` |
| FREE | `a, d, e, k, l, m, t, w, x, 0, comma (,), semicolon (;), slash (/), apostrophe ('), minus (-), equal (=), backslash (\), grave (`), space` |

> Vicinae candidates (verified free 2026-09-25): `SUPER+SHIFT+space`, `SUPER+SHIFT+d` (old rofi slot). `SUPER+ALT+space` is TAKEN (`toggle_all_floating`), `SUPER+CTRL+space` is TAKEN (`togglefloating`).

## SUPER+CTRL

| Status | Keys |
|---|---|
| Taken | `d f h j k l u, space, return, tab, 1-9, arrows` |
| FREE | `a, b, c, e, g, i, m, n, o, p, q, r, s, t, v, w, x, y, z, 0, comma (,), period (.), semicolon (;), slash (/), apostrophe ('), minus (-), equal (=), brackets, backslash (\), grave (`)` |

## SUPER+ALT (wide open except shell-swap + float-all)

| Status | Keys |
|---|---|
| Taken | `l, space, 1-4, comma (,), period (.), arrows` |
| FREE | `a b c d e f g h i j k m n o p q r s t u v w x y z, 0, 5-9, semicolon (;), slash (/), apostrophe ('), minus (-), equal (=), brackets, backslash (\), grave (`), return, tab` |

> DMS-only variant: no `keymode=noctalia` section, no `SUPER+ALT+N/D` shell-swap, no `XF86` Noctalia binds. `SUPER+ALT+D/N` are FREE here (taken in dual).

## Triple layers (nearly untouched)

- `SUPER+CTRL+SHIFT`: only `S` and `Tab` taken — everything else free.
- `SUPER+ALT+SHIFT`: only `comma` and `period` taken — everything else free.

## Other

- `F1-F12`: NO function key is bound on ANY layer — all free.
- `Print` combos (`NONE`/`SHIFT`/`ALT` + `Print`) are all taken (screenshots).
- `ALT+Tab` / `ALT+SHIFT+Tab` taken (window cycling). Bare `ALT+letter` otherwise free but risky (apps use ALT). `ALT+space` free (vicinae alt option).
- `CTRL+ALT+P` taken (power menu), `CTRL+ALT+SUPER+B` taken (bar toggle).
- `SUPER+wheel` taken (tag hopping). `SUPER+click/drag` taken (move/resize/float).

## Recommendations for the next binds

1. `SUPER+SHIFT+<letter>` first (`a d e k l m t w x` free, single extra key).
2. `SUPER+slash` or `SUPER+apostrophe` for launcher-style popups (prime, mnemonic-adjacent).
3. `SUPER+ALT+<letter>` when the SHIFT variant is taken (e.g. `SUPER+ALT+V` was the neovim runner-up).
4. Bare `SUPER` letters are exhausted — `J` now runs the special workspace trio, not the old calculator.
