# zmk-config

Personal ZMK config for a **Sofle Choc Pro** split keyboard (Xiao nRF52840 halves), built with QWERTY + home-row mods and four extra layers (sym/num/nav/sys).

## Repo layout

- `config/sofle_choc_pro.keymap` — the keymap (BASE + sym/num/nav/sys layers)
- `config/*.conf` — ZMK config options
- `scripts/flash-sofle.sh` — interactive wizard: builds firmware and flashes both halves
- `build/left|right/zephyr/zmk.uf2` — build output (not committed)

## Flashing the keyboard

```sh
./scripts/flash-sofle.sh
```

Requirements: Docker running (builds inside `zmkfirmware/zmk-build-arm:stable`), macOS or Linux.

The wizard has 3 stages:

1. **Build firmware** — runs `west build` in Docker for both `sofle_choc_pro_left` and `sofle_choc_pro_right`, output to `build/<side>/zephyr/zmk.uf2`. If firmware already exists you can reuse it; pick **rebuild after keymap changes**.
2. **Flash left half** — unplug everything, plug in only the left half, trigger bootloader mode, script copies the `.uf2` to the mounted bootloader drive.
3. **Flash right half** — same for the right half.

Entering bootloader mode for a half:

- If new firmware is already on it: toggle to the **SYS** layer (SYS key, bottom-right of BASE) and press **RST** (bottom row, next to BOOT).
- Otherwise: double-tap the physical reset button on the board.

The script polls `/Volumes` for the newly mounted drive (e.g. `XIAO-BOOT`) and copies the `.uf2` there; the drive ejects itself and the keyboard reboots. If no drive appears within 2 minutes you can type the mount path manually, or skip and drag the `.uf2` onto the drive by hand.

After flashing:

- Bluetooth pairing may be needed — reconnect via USB or re-pair "Sofle Choc Pro" in Bluetooth settings.
- If BT misbehaves: build & flash a `settings_reset` firmware on **both** halves (script prints the exact docker command at the end), then reflash normal firmware and re-pair.

## Building only (no flashing)

```sh
docker run --rm -v "$PWD":/workspaces/zmk-config -w /workspaces/zmk-config \
  zmkfirmware/zmk-build-arm:stable \
  west build -s zmk/app -b sofle_choc_pro_left -d build/left \
  -- -DZMK_CONFIG=/workspaces/zmk-config/config
```

(Repeat with `sofle_choc_pro_right`.)
