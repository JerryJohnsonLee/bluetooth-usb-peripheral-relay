# Fix keyboard and mouse HID mapping for extended keys and buttons

## Summary

- Fix incorrect USB HID codes for Page Up, Page Down, and End keys that made them non-functional
- Fix F11/F12 mapping (Linux codes 87/88 were incorrectly assumed to be sequential with F1-F10)
- Add full numpad support (NumLock, KP0-KP9, KP Enter, KP +/-/*/.)
- Add missing keys: PrintScreen, ScrollLock, Pause, Menu/Application
- Add mouse side button support (5 buttons instead of 3) with BTN_FORWARD/BTN_BACK aliases
- Add horizontal scroll wheel support (REL_HWHEEL / AC Pan) for mice like the Logitech MX Master 3S
- Update mouse HID descriptor and report format from 4 to 5 bytes
- Fix incorrect Linux input event code constants to match `input-event-codes.h`

## Problem

Several key categories and mouse features were broken in the Bluetooth-to-USB relay:

**Keyboard:**
- Page Up/Down produced Up/Down Arrow instead (both mapped to the same HID codes 0x52/0x51)
- End key produced Page Up (mapped to HID 0x4B instead of 0x4D)
- F11/F12 were unmapped — the F-key loop mapped Linux codes 59-70 sequentially, but F11=87 and F12=88 in Linux (codes 69/70 are NumLock/ScrollLock)
- Numpad keys had no mappings at all

**Mouse:**
- HID descriptor only declared 3 buttons (left/middle/right), so side button bits were silently ignored by the host OS
- No horizontal scroll axis in the descriptor, making the MX Master 3S thumb wheel non-functional

## Changes

### `internal/relay/keyboard.go`
- Fix navigation cluster HID codes: Page Up → 0x4B, End → 0x4D, Page Down → 0x4E
- Fix F-key loop to only cover F1-F10 (codes 59-68), add explicit F11 (87→0x44) and F12 (88→0x45)
- Add 17 numpad key mappings (NumLock, KP0-9, KP operators)
- Add PrintScreen (0x46), ScrollLock (0x47), Pause (0x48), Menu (0x65)
- Fix all `KEY_*` constants to match Linux `input-event-codes.h`

### `internal/relay/mouse.go`
- Expand report from `[4]byte` to `[5]byte` (new byte 4 = horizontal wheel)
- Add `case 6` (REL_HWHEEL) handler for horizontal scroll
- Extend button range from 272-276 to 272-278, mapping BTN_FORWARD(277)→bit 4 and BTN_BACK(278)→bit 3
- Update `validateEvent` to accept the extended button range

### `scripts/setup_gadgets.sh`
- Update mouse `report_length` from 4 to 5
- Replace mouse HID descriptor: 5 buttons (was 3), X/Y, vertical wheel, horizontal wheel (AC Pan via Consumer usage page 0x0C)

### `internal/relay/keyboard_test.go`
- Add test cases for F1, F11, F12, Home, End, Page Up, Page Down, Up Arrow, NumLock, KP5, KP Enter, and unmapped key handling

### `internal/relay/mouse_test.go`
- Update all existing test expected reports from 4 to 5 bytes
- Add tests for horizontal scroll (REL_HWHEEL), side buttons (BTN_SIDE, BTN_EXTRA), and aliased buttons (BTN_FORWARD, BTN_BACK)
- Add validation tests for extended button range and horizontal wheel events

## Deployment

After merging, on the Raspberry Pi:
1. Re-run `sudo bash scripts/setup_gadgets.sh` to update the USB HID descriptors
2. Rebuild and reinstall: `task service:install`

## Test plan

- [ ] Run `go test ./...` — all tests pass
- [ ] Verify F1-F12 keys work on Filco Majestouch Convertible
- [ ] Verify Home/End/Page Up/Page Down produce correct actions
- [ ] Verify numpad keys (with NumLock on/off)
- [ ] Verify MX Master 3S side buttons (back/forward) work
- [ ] Verify MX Master 3S horizontal thumb wheel scrolls horizontally
