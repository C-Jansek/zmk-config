# ZMK home-row mods vs Kanata `tap-hold-release` — research notes

Research only; no keymap changes made. Written 2026-09-22 against ZMK `main` (development docs, "You're viewing the documentation for the development version") and Kanata `main`.

Question: how can ZMK home-row mods be made to behave like Kanata's
`(tap-hold-release $tap-time $hold-time tap hold)` (tap-time = hold-time = 200 in
`~/github/kanata/kanata.kbd`), and how can key repeat on a quick re-press be enabled?

---

## 1. Kanata behavior breakdown (what `tap-hold-release` actually does)

Source: Kanata config docs, `tap-hold` section —
https://github.com/jtroo/kanata/blob/main/docs/config.adoc (action reference ~lines 2018–2130,
variant table ~2055–2120).

Syntax: `(tap-hold-release $tap-repress-timeout $hold-timeout $tap-action $hold-action)`

The user's config (`kanata.kbd` lines 11–27) uses `tap-time 200`, `hold-time 200`, e.g.
`(tap-hold-release $tap-time $hold-time a lalt)` for a/s/d/f → alt/gui/shift/ctrl and
j/k/l/; → ctrl/shift/gui/alt.

Exact timing rules (from the docs' variant descriptions and `tap-hold` description):

| Event sequence | Result |
|---|---|
| Key released before `hold-timeout` elapses, no other key involved | **tap** action |
| Key still held when `hold-timeout` elapses | **hold** action (at timeout) |
| Another key **pressed and released** while the tap-hold key is still held | **hold** action fires early (this is what makes the variant "release"-flavored; a press-only does *not* decide it) |
| Another key pressed but not yet released | **no decision yet** — Kanata waits; the decision comes from the other key's release, the tap-hold key's release, or the timeout |
| press → release → press again within `tap-repress-timeout` | the **tap action is output immediately and held** (gives OS auto-repeat), even if the second press outlasts `hold-timeout` |

Notes:

- The two parameters have **different roles**: the first (`tap-repress-timeout`, "tap-time")
  only governs the quick re-press/repeat window; the second (`hold-timeout`, "hold-time")
  only governs the hold timeout. They are not one shared timer.
- Kanata also has a global/per-action `require-prior-idle` option and richer variants
  (`tap-hold-release-keys`, `tap-hold-opposite-hand`, etc.) — see the variant table in the
  same doc section. The user's config does not use them.
- The docs also flag the classic `f24` trick needed to make repeats work in multi-key
  setups (`(multi f24 (tap-hold ...))`) — the user's config already does this
  (`kanata.kbd` lines 19–26), which is part of why repeat feels reliable there.

## 2. Current ZMK setup and why it misbehaves

Current state (`config/sofle_choc_pro.keymap`):

- Line 86: home row uses the predefined `&mt` behavior: `&mt LALT A`, `&mt LGUI S`,
  `&mt LSHFT D`, `&mt LCTRL F`, `&mt RCTRL J`, `&mt RSHFT K`, `&mt RGUI L`.
- A custom `ht_ralt_semi` (balanced, tapping-term 200) is only used for `;` (line 86).

**Key finding — the premise in the task needs a correction:** `&mt` does *not* default to
`balanced`. The `zmk,behavior-hold-tap` binding defaults `flavor` to
**`hold-preferred`** (`zmk,behavior-hold-tap.yaml` line 34:
`default: "hold-preferred"`), and the docs state mod-tap is "configured with the
'hold-preferred' flavor" by default:
https://zmk.dev/docs/keymaps/behaviors/hold-tap#interrupt-flavors.

Consequences:

1. **Accidental mods:** with `hold-preferred`, *any other key pressed* while the home-row
   key is held resolves the hold immediately — even if the other key is released 20 ms
   later within a fast typing roll. In `app/src/behaviors/behavior_hold_tap.c`,
   `decide_hold_preferred()` maps `HT_OTHER_KEY_DOWN → STATUS_HOLD_INTERRUPT`
   (~line 337–347). Kanata's `tap-hold-release` instead waits for the other key's
   *release* before firing the hold. This is exactly the observed difference
   (rolls → unwanted modifiers).
2. **No repeat:** `quick-tap-ms` defaults to `-1` (disabled) — `zmk,behavior-hold-tap.yaml`
   lines 19–22 (`default: -1`). So a quick double-tap of a home-row key produces two
   discrete taps and never holds the tap keycode, so the host never auto-repeats.
   (The custom `ht_ralt_semi` is `balanced` + 200 ms but also has no `quick-tap-ms`.)

Note on repeat mechanics in general: ZMK has no firmware auto-repeat. Host auto-repeat
happens when a keycode stays *down* in the HID report. For a plain `&kp` key that is
trivial. For a hold-tap, the tap keycode only stays down when the hold-tap resolves to a
tap while the finger is still on the key — which only happens *immediately*, before any
timeout/interrupt, via the quick-tap path (below).

## 3. Options matrix — every relevant ZMK mechanism

All of these are properties of the `zmk,behavior-hold-tap` binding
(`app/dts/bindings/behaviors/zmk,behavior-hold-tap.yaml` on `main`):

```
bindings; tapping-term-ms; quick-tap-ms; require-prior-idle-ms; flavor;
hold-while-undecided; hold-while-undecided-linger; retro-tap;
hold-trigger-key-positions; hold-trigger-on-release
```

(yaml lines 8–51; deprecated aliases `tapping_term_ms`, `quick_tap_ms`, `global-quick-tap`
also exist.)

Decision logic is in `app/src/behaviors/behavior_hold_tap.c` (`main`). Decision moments:
own key down/up (`HT_KEY_DOWN`/`HT_KEY_UP`), other key down/up (`HT_OTHER_KEY_DOWN`/
`HT_OTHER_KEY_UP`), timer (`HT_TIMER_EVENT`), quick-tap (`HT_QUICK_TAP`); dispatched at
~line 783 (`ev->state ? HT_OTHER_KEY_DOWN : HT_OTHER_KEY_UP`) and on timer expiry
(`behavior_hold_tap_timer_work_handler`, ~line 842).

### 3.1 Flavors (`flavor = "..."`) — the tap-vs-hold decider

Semantics verified in source, `decide_*` functions ~lines 282–347:

| Flavor | Own key released (before term) | Other key pressed | Other key pressed **and released** | Timer expires |
|---|---|---|---|---|
| `hold-preferred` (&mt default) | tap | **hold** | hold | hold |
| `balanced` | tap | *(undecided, keeps waiting)* | **hold** | hold |
| `tap-preferred` (&lt default) | tap | *(undecided)* | *(undecided)* | hold |
| `tap-unless-interrupted` | tap | **hold** | (already hold) | **tap** |

Mapping to Kanata `tap-hold-release`:

- `balanced` is the **exact** match for the decision rules: hold fires on
  other-key press+release (early hold), on timeout, and tap on own release. Both systems
  also leave the decision pending while another key is merely held down.
- `hold-preferred` (what `&mt` currently is) fires hold on other-key **press** —
  more aggressive than Kanata; this is the misfire source.
- `tap-preferred` is stricter than Kanata (hold only via timeout) — slower modifier
  chords, even fewer misfires.
- `tap-unless-interrupted` inverts the logic; not a Kanata match.

### 3.2 `quick-tap-ms` — repeat on quick re-press

Docs: https://zmk.dev/docs/keymaps/behaviors/hold-tap#quick-tap-ms — "If you press a tapped
hold-tap again within `quick-tap-ms` milliseconds of the first press, it will always
trigger the tap behavior… a quick tap-then-hold can be used to hold it down to delete long
parts of text." Default disabled (`default: -1`, yaml line 22).

Implementation (`behavior_hold_tap.c:143-149`, `is_quick_tap()`):

- Timing is **press-to-press** of the same position (ZMK counts from first *press*, not
  release — docs note the QMK `QUICK_TAP_TERM` difference), matching Kanata's
  tap-repress window (Kanata's worked example is also press→release→press within the
  window → tap held).
- On quick-tap the hold-tap decides tap *at key down* (`decide_hold_tap(..., HT_QUICK_TAP)`
  in `on_hold_tap_binding_pressed`, ~line 625), so the tap keycode goes down immediately
  and stays down until release → the **host OS auto-repeats**. No separate ZMK repeat
  config is needed.
- Nuance: in `is_quick_tap()`, `require-prior-idle-ms` is checked **first** (any recent key
  → treat as quick tap), then the same-position `quick-tap-ms` check. So setting
  `require-prior-idle-ms` subsumes/overrides part of the `quick-tap-ms` window.
- The last-tap bookkeeping (`last_tapped`, `store_last_tapped`, ~lines 111–142) only
  records non-modifier key presses and hold-tap taps — i.e. exactly the typing activity
  Kanata's repress window also tracks.

Kanata equivalence: `quick-tap-ms = <200>` reproduces `tap-time 200` repress behavior.
(Kanata's `f24` workaround for repeat through tap-hold has no ZMK analogue — and none is
needed; ZMK's mechanism is built into the hold-tap itself.)

### 3.3 `require-prior-idle-ms` — "global quick tap"

Docs: https://zmk.dev/docs/keymaps/behaviors/hold-tap#require-prior-idle-ms — "If a
hold-tap is pressed within `require-prior-idle-ms` of another non-modifier key … the
hold-tap will always resolve in a tap… the hold-tap immediately resolves to a tap on key
press." Default `-1` (disabled), yaml lines 28–30.

Not in the user's Kanata config (Kanata has it as `require-prior-idle` /
"automatic fast typing layer", see https://github.com/jtroo/kanata/discussions/1425), but
in ZMK it is the standard companion that removes both residual misfires and typing latency
with `balanced`. It makes mods harder to trigger mid-fast-typing (documented tradeoff).

### 3.4 Positional hold-tap: `hold-trigger-key-positions` + `hold-trigger-on-release`

Docs: https://zmk.dev/docs/keymaps/behaviors/hold-tap#positional-hold-tap-and-hold-trigger-key-positions
and #hold-trigger-on-release.

- With `hold-trigger-key-positions = <...>` set, a first other-key press at a position
  *not* in the list forces a tap, regardless of flavor. Recommended for home-row mods so
  same-hand rolls don't become mods.
- By default the trigger key is evaluated on **press**; `hold-trigger-on-release;`
  delays the evaluation to the other key's **release** — this is precisely the
  "wait for release" character of `tap-hold-release`, applied per key position, and it
  also lets same-hand mods combine.
- This is orthogonal to the flavor (it overrides the flavor's decision, `decide_positional_hold()`
  ~line 425) and is the ZMK analogue of Kanata's `tap-hold-release-keys` / opposite-hand
  variants (secondary: kanata docs variant table; https://github.com/jtroo/kanata/issues/1602).
- Cost: `hold-trigger-key-positions` are *key position indexes* numbered by keymap order —
  per-keymap bookkeeping, and it blocks same-hand modifier chords unless those positions
  are whitelisted.

### 3.5 Separate tap-time vs hold-time — **not supported**

The binding has a single `tapping-term-ms` (`zmk,behavior-hold-tap.yaml` lines 12–16).
There is no property for a distinct hold timeout vs decision timeout. However, this is not
a loss for this use case: Kanata's `tap-time` only governs the *repress* window (which ZMK
covers with `quick-tap-ms`) and `hold-time` governs the hold timeout (which ZMK covers
with `tapping-term-ms`). With both set to 200 in Kanata, ZMK needs
`tapping-term-ms = <200>` + `quick-tap-ms = <200>` — one property per Kanata timer, no
shared timer in practice.

### 3.6 `hold-while-undecided` / `hold-while-undecided-linger` (eager mods)

Docs: #hold-while-undecided, #hold-while-undecided-linger. The hold binding is pressed
*immediately* on key-down and released again before the tap if the hold-tap resolves to a
tap. Kanata equivalent concept is eager mods / hold-while-undecided (kanata discussion
1602 table). Useful for Shift+Click / mouse use; adds a small risk of stray modifier
events (docs note Alt/Gui side effects). Optional, not part of a Kanata match.

### 3.7 `retro-tap`

Docs: #retro-tap — if the hold-tap resolved to hold *only by timeout* and is released
with no other key pressed, the **tap** is emitted instead of press+release of the mod.
Kanata's `tap-hold-release` does *not* do this (it outputs the hold action). Including
`retro-tap` therefore diverges from Kanata but is a common comfort tweak for home-row
mods. Optional.

### 3.8 Adjacent mechanisms (not hold-tap, briefly)

- **Sticky key / one-shot mods** `&sk LALT`, `&sk LGUI` — press once, mod applies to next
  key then expires: https://zmk.dev/docs/keymaps/behaviors/sticky-key. Different model
  from tap-hold; no timing ambiguity, but an extra keypress.
- **Sticky layer** `&sl SYM` — already used in this keymap for SYM/NUM:
  https://zmk.dev/docs/keymaps/behaviors/sticky-layer.
- **Mod-morph** — already used here (`mm_comma` etc.); shifts tap output on a modifier,
  unrelated to hold-vs-tap timing: https://zmk.dev/docs/keymaps/behaviors/mod-morph.
- **Key Repeat behavior** `&key_repeat` re-sends the last pressed keycode on a dedicated
  key — not relevant to hold-tap repeat: https://zmk.dev/docs/keymaps/behaviors/key-repeat.
- The `&lt` layer-tap variant of hold-tap supports all the same properties if the
  Kanata `fnl` (fn hold / layer-toggle) pattern is ever mirrored on the keyboard.

### 3.9 Community/secondary sources (marked secondary)

- precondition, "A guide to home row mods" — flavor comparison (QMK/ZMK/Kanata mapping,
  `balanced` = Permissive Hold = `tap-hold-release`): https://precondition.github.io/home-row-mods
  (discussion: https://github.com/precondition/precondition.github.io/discussions/26)
- urob's "Timeless Home Row Mods" ZMK config guide (balanced + require-prior-idle +
  positional + hold-trigger-on-release recipe): https://github.com/urob/zmk-config
  (summary thread: https://www.reddit.com/r/KeyboardLayouts/comments/1dz6oyy/…)
- ZMK docs' own "timeless home-row mods" example (primary): see hold-tap docs, "Homerow
  Mods" custom example — it ships exactly `balanced` + `require-prior-idle-ms` +
  `quick-tap-ms` + `hold-trigger-key-positions` + `hold-trigger-on-release`.
- Kanata home-row-mod samples & discussions (secondary):
  https://github.com/jtroo/kanata/blob/main/cfg_samples/home-row-mod-basic.kbd,
  https://github.com/jtroo/kanata/discussions/1455, https://github.com/jtroo/kanata/discussions/1425.

## 4. Recommended config (closest match to Kanata `tap-hold-release`)

Recipe: **`flavor = "balanced"`** (== tap-hold-release decision rules) **+
`tapping-term-ms = <200>`** (== hold-time) **+ `quick-tap-ms = <200>`** (== tap-time,
gives repress/auto-repeat) **+ `require-prior-idle-ms = <125…150>`** (recommended extra:
kills residual same-hand roll misfires and typing latency — strictly *more* tap-biased
than the Kanata config, adjust to taste). Positional hold-tap (`hold-trigger-key-positions`
+ `hold-trigger-on-release`) is optional hardening on top, per the docs' "timeless
home-row mods" recipe.

```dts
/ {
  behaviors {
    // Balanced == Kanata tap-hold-release: hold fires when another key is
    // pressed AND released while this key is held, or on timeout; tap on own release.
    hml: home_row_mod_left {
      compatible = "zmk,behavior-hold-tap";
      #binding-cells = <2>;
      flavor = "balanced";
      tapping-term-ms = <200>;
      quick-tap-ms = <200>;
      require-prior-idle-ms = <125>;
      bindings = <&kp>, <&kp>;
    };
    hmr: home_row_mod_right {
      compatible = "zmk,behavior-hold-tap";
      #binding-cells = <2>;
      flavor = "balanced";
      tapping-term-ms = <200>;
      quick-tap-ms = <200>;
      require-prior-idle-ms = <125>;
      bindings = <&kp>, <&kp>;
    };
  };
};
```

Keymap changes (BASE, line 86):

```dts
// before
&kp ESC  &mt LALT A  &mt LGUI S  &mt LSHFT D  &mt LCTRL F  &kp G    &kp H  &mt RCTRL J  &mt RSHFT K  &mt RGUI L  &ht_ralt_semi RALT 0  &kp RET

// after
&kp ESC  &hml LALT A  &hml LGUI S  &hml LSHFT D  &hml LCTRL F  &kp G    &kp H  &hmr RCTRL J  &hmr RSHFT K  &hmr RGUI L  &hmr_ralt_semi RALT 0  &kp RET
```

Keep `ht_ralt_semi` (or clone it as `hmr_ralt_semi`) with the same new options and
`bindings = <&kp>, <&mm_semi>;`. Optionally apply the same options to the sym-layer
`&mt` bindings (line 106) for consistency.

Optional extra (positional / "timeless" style, per docs example): add
`hold-trigger-key-positions = <…>;` (opposite-hand + thumb positions) and
`hold-trigger-on-release;` to each behavior. Tradeoffs: eliminates nearly all same-hand
misfires, but requires maintaining key-position indexes and blocks same-hand mod chords
unless whitelisted.

Tradeoffs of the recommended (non-positional) setup vs Kanata:

- `require-prior-idle-ms` is not in the Kanata config, so behavior is *slightly* more
  tap-biased than Kanata: mods pressed during fast typing resolve as taps until there's
  a ~125 ms pause first. If that interferes with intentional holds while typing, raise
  it or drop the property (the Kanata-exact core is balanced + the two timers).
- `quick-tap-ms` is per-behavior and press-to-press; a genuinely held tap keycode
  auto-repeats at the host's repeat rate (identical to Kanata's repress+OS repeat).
- Single-timer concern is moot here: Kanata's tap-time maps to `quick-tap-ms`, hold-time
  to `tapping-term-ms`.
- No `.conf` changes needed; nothing Kconfig-side governs these properties. If logging
  decisions during tuning, `CONFIG_ZMK_LOG_LEVEL_DBG` shows the `decided …` lines.

## 5. Sources

Primary:

1. ZMK Hold-Tap Behavior docs (development version):
   https://zmk.dev/docs/keymaps/behaviors/hold-tap (v0.3 mirror:
   https://v0-3-branch.zmk.dev/docs/keymaps/behaviors/hold-tap)
   — mod-tap default flavor hold-preferred; flavor definitions; quick-tap-ms,
   require-prior-idle-ms, positional hold-tap / hold-trigger-on-release, hold-while-undecided,
   retro-tap, "timeless home-row mods" example.
2. `zmk,behavior-hold-tap.yaml` (zmkfirmware/zmk, main):
   https://github.com/zmkfirmware/zmk/blob/main/app/dts/bindings/behaviors/zmk,behavior-hold-tap.yaml
   — property list and defaults: `quick-tap-ms` default −1 (lines 19–22),
   `require-prior-idle-ms` default −1 (28–30), `flavor` default `"hold-preferred"` (31–37),
   `retro-tap` (44), `hold-trigger-on-release` (50), single `tapping-term-ms` (12–16).
3. `behavior_hold_tap.c` (zmkfirmware/zmk, main):
   https://github.com/zmkfirmware/zmk/blob/main/app/src/behaviors/behavior_hold_tap.c
   — `is_quick_tap()` (143–149, require-prior-idle checked before same-position
   quick-tap), `decide_balanced()` (282–299, hold on `HT_OTHER_KEY_UP`),
   `decide_hold_preferred()` (337–347, hold on `HT_OTHER_KEY_DOWN`),
   `decide_tap_preferred()` (301–315), `decide_tap_unless_interrupted()` (319–334),
   timer scheduling (636–637), other-key dispatch (783), timer decision (842).
4. Kanata config docs, `tap-hold` reference (jtroo/kanata, main):
   https://github.com/jtroo/kanata/blob/main/docs/config.adoc — `tap-hold-release` =
   "Activate `$hold-action` early if held and another input key is pressed and released";
   tap-repress-timeout/hold-timeout parameter semantics; press→release→press example;
   variant table (`tap-hold-release-keys`, `require-prior-idle` option).
5. ZMK Key Repeat behavior docs (for completeness):
   https://zmk.dev/docs/keymaps/behaviors/key-repeat
6. ZMK Sticky Key docs: https://zmk.dev/docs/keymaps/behaviors/sticky-key;
   Sticky Layer: https://zmk.dev/docs/keymaps/behaviors/sticky-layer;
   Mod-Morph: https://zmk.dev/docs/keymaps/behaviors/mod-morph

Secondary (community, clearly non-authoritative):

7. precondition, "A guide to home row mods": https://precondition.github.io/home-row-mods
8. urob's Timeless Home Row Mods: https://github.com/urob/zmk-config
9. Kanata discussions on HRM tuning / automatic fast typing layer:
   https://github.com/jtroo/kanata/discussions/1455,
   https://github.com/jtroo/kanata/discussions/1425,
   https://github.com/jtroo/kanata/issues/1602 (flavor cross-comparison table)
10. Local configs referenced: `config/sofle_choc_pro.keymap` (this repo),
    `~/github/kanata/kanata.kbd`.
