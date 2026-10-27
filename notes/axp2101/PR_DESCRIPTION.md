# PR draft (not opened, waiting on the owner's approval)

**Title:** esp32: don't let a talk on the AXP2101's power key outlast its hold-off

**Head:** `Loanyy:fix/axp2101-hold-off-during-talk` → `facebookincubator:main`

---

## Problem

On the boards that talk on the AXP2101's power key, holding talk for more than 10 s cuts power to the board. There is no clean shutdown, and the voice note being recorded is lost (#277, reproduced on a Waveshare ESP32-S3-Touch-AMOLED-1.75C). These boards are:

- the Waveshare ESP32-S3-Touch-AMOLED-1.75C (PWR, read through GPIO3)
- the M5Stack CoreS3 (PWR is its only button, read from the PMU's key latch)
- the Waveshare ESP32-S3-Touch-AMOLED-2.16 (PWR talks as well as KEY3)

## Root cause

- `esp32/components/muse/muse_pmu.c:77` sets REG27[3:2] (OFFLEVEL) to `PKEY_OFF_10S`, which is 10 s, the longest the chip offers. The comment on it (`muse_pmu.c:49`) says that is "longer than the 8 s talk limit".
- The talk limit is `MAX_SECS 15` (`esp32/components/muse/muse_voice.c:48`).
- The AXP2101 powers off when its key is held past OFFLEVEL, if REG22[1] allows that. Nothing in the firmware sets REG22, so the PMU's default applies (an eFuse value; on the 1.75C it is evidently on).
- The talk buttons that are affected:
  - 1.75C: `.talk_button = "top"`, `boards/board_waveshare_s3_175c.c:237` and `:264`
  - CoreS3: `boards/board_m5stack_cores3.c:126`
  - 2.16: `boards/board_waveshare_s3_216.c:211`, and `:231`, where PWR sets `pwr_held`

### Why not just cap the talk at ~9 s?

The PMU counts how long the key is physically held, not how long the firmware records. If talk stops at 9 s and the user keeps holding, the board still powers off at 10 s. Only two things prevent it:

- (A) don't talk on the PMU key, or
- (B) disable the PMU's long-press power-off while that key is held.

The CoreS3 has no other button, so it can only use (B).

## Datasheet evidence

These are from the X-Powers AXP2101 datasheet V1.0, as hosted by M5Stack: <https://m5stack.oss-cn-shenzhen.aliyuncs.com/resource/docs/datasheet/core/K128%20CoreS3/AXP2101_Datasheet_V1.0_en.pdf>.

§6.13.2.19, REG 22: PWROFF_EN (p. 36):

| Bit | Description | R/W | Reset | Default |
|---|---|---|---|---|
| 7:3 | | RO | / | 0 |
| 2 | DIE Over-Temperature(LEVEL2) as POWEROFF Source enable. 0: disable 1: enable | RW | POR | 1b |
| 1 | **PWRON > OFFLEVEL as POWEROFF Source enable. 0: disable 1: enable** | RW | POR | EFUSE |
| 0 | Function Select when btn_pwroff_en=1 and button power-off occur. 0: Power-off 1: Restart | RW | POR | EFUSE |

§6.13.2.24, REG 27: IRQLEVEL/OFFLEVEL/ONLEVEL setting (p. 38): "OFFLEVEL configuration, bits 3:2: 00: 4s 01: 6s 10: 8s 11: 10s", POR default 01b.

§6.5.4.3, Power Off (p. 20): "POK. AXP2101 can be powered off by pressing and holding POK for a period of time that longer than "OFFLEVEL". The function can be configured by REG22H[1] …". The power-off sources it lists besides this one are REG10H[0] (soft off), VSYS below VOFF or VBUS above 7 V, DCDC under- or over-voltage, and die over-temperature. None of them is a key hold.

§6.13.2.7, REG 10 (p. 32): bit 2 is "PWRON 16s to shut the PMIC enable", POR default 0b. This is the only other key-hold shutdown. It is off by default and nothing in this repo sets it.

The Reset column is "POR" for REG22 and REG27, so the PMU keeps them across ESP32 resets and soft power-off. It only resets them on a power-on reset (battery removed). That is why init has to restore REG22[1].

XPowersLib agrees on every bit: `enableLongPressShutdown()` and `disableLongPressShutdown()` set and clear bit 1 of `XPOWERS_AXP2101_PWROFF_EN` (0x22), and `setPowerKeyPressOffTime()` writes bits 3:2 of 0x27 (<https://github.com/lewisxhe/XPowersLib/blob/master/src/XPowersAXP2101.hpp>).

## Fix

- **`muse_pmu.c`/`.h`: new `muse_pmu_set_key_shutdown(bool on)`.** It is a read-modify-write of REG22 bit 1 only, and returns `ESP_ERR_INVALID_STATE` without touching the bus if there is no PMU.
  - `muse_pmu_init()` turns the bit on at every boot and keeps REG22's other bits. A reset in the middle of a held talk would otherwise leave it off.
  - The init log line now shows REG22 and REG21 (PWROFF status, the cause of the last power-off), which helps bench checks.
  - The stale "8 s talk limit" comment is fixed.
- **Each affected board's `poll_buttons()`** turns key shutdown off on PWR's press edge and back on at its release. That runs in the input task, never in an ISR, on the same task and I2C bus that already poll the PMU's key latch and battery status. The bus is the BSP's `i2c_master` bus, which serialises transactions.
  - **1.75C:** follows the debounced GPIO3 edges.
  - **CoreS3:** follows the PMU latch. When press and release arrive in one read, the state stays as it was. That matches how `talk_button()` orders them: a tap stays released, and a release followed by a re-press stays held.
  - **2.16:** only the PWR path does this, not KEY3. A whole PWR press seen in one read is dropped, as before, so it changes nothing.
- **Tracking the physical key, not voice state.** A talk that `MAX_SECS` ends while PWR is still held stays protected until release.
- **A failed write** is logged once per edge, and the talk goes ahead. With no PMU it does nothing.
- **Unchanged:** the 1.75 (non-C), where BOOT talks and a long PWR hold still powers off; the C6 boards (PWR is aux there); every other board.
- **Kept:** the 10 s hold-off still works whenever PWR isn't held, and whenever the firmware isn't polling buttons. That includes a wedged firmware, which never sees the press, so the bit stays on.

### Behaviour changes worth a reviewer's eye

1. Because PWR is the talk key on these three boards, a healthy firmware no longer powers off on a 10 s PWR hold. On the 1.75C and 2.16, holding BOOT (aux) for 1.5 s still powers off, and so does Settings → Power off; on the CoreS3, Settings → Power off.
2. `muse_pmu_init()` now explicitly turns REG22[1] on for every AXP2101 board. If some board's eFuse default had it off, a 10 s hold of an aux PWR (C6 boards) would now power off where it didn't before. The existing comment and log already assumed this was on ("hardware PWR hold-off 10 s").
3. On the 2.16, while Wi-Fi naps, `wait_buttons` sleeps on KEY3 and BOOT for up to 10 s (`NAP_WAIT_MS`), because the PMU's IRQ isn't wired. A PWR press there can go unseen until the next poll, so this fix can't cover a PWR talk begun in that state. KEY3 is unaffected.
4. If the firmware wedges while PWR is held, the hold-to-off fallback stays disabled until the next reset or power-on reset.

## Alternative (A)

If you'd rather not have PMU writes on the talk path, the 1.75C could swap its side buttons so that BOOT talks and PWR is aux, as the 1.75 does. That would also help #65 (PWR produced no input events on the reporter's 1.75C). The 2.16 could stop mapping PWR to talk, since KEY3 already talks. The CoreS3 has only PWR, so (B) is its only option.

## Tests

`esp32/tests/test_muse_pmu.py`, with `tests/muse_pmu_harness.c` and `tests/pmu_fakes/`:

- **`MusePmu`** compiles the real `muse_pmu.c` against a fake `i2c_master` with a 256-byte register file, plus minimal `esp_log`, `esp_check` and `esp_err` fakes. It checks that:
  - init sets REG27[3:2]=11 and REG22[1]=1 and keeps every other bit, including when a reset left bit 1 off;
  - the setter touches only REG22 bit 1, for all 8 starting values;
  - read and write errors are returned, and a failed read writes nothing;
  - without a PMU the result is `ESP_ERR_INVALID_STATE`, with no bus traffic.
- **`PowerKeyTalkBoards`** extracts each board's production `poll_buttons()`, in the style of `test_core2_buttons.py`, and runs it against GPIO and PMU-latch fakes:
  - **1.75C:** a PWR press turns shutdown off once, a 15 s hold leaves it off, and release turns it on. BOOT is ignored, a release from a PWR held at boot is harmless, and a failed write still reports the talk.
  - **1.75:** BOOT talks, and PWR (aux) never touches it.
  - **CoreS3:** press and release, a 15 s hold, a tap inside one latch read, a release and re-press inside one read while held, and a failed write.
  - **2.16:** KEY3 never touches it. PWR press and release work, a whole PWR press inside one read is dropped, and KEY3 held across a PWR press is handled.

**Before and after.** Against unpatched upstream (1a1682f):

- The CoreS3, 1.75C and 2.16 board tests fail with ``Assertion `calls == 1 && !shutdown_on' failed``.
- `MusePmu` fails to build (`muse_pmu_set_key_shutdown` doesn't exist).
- Built with a stub setter, the init case fails at ``Assertion `s_regs[PWROFF_EN] == 0x07' failed``: REG27 is set, REG22[1] is not.
- The 1.75 non-C test passes both before and after; it guards the unaffected board.

With the patch, all 8 pass with gcc and clang (`-Wall -Wextra -Werror`).

Full host suite, run as CI runs it (IDF v6.0.1, `libmbedtls-dev` and `pkg-config`, `idf.py -B build-host-tests reconfigure`, FreeInk at 2c047df): 837 tests, 1 failure, 3 skipped, against 829 tests, 1 failure, 3 skipped on unpatched 1a1682f. The difference is exactly the 8 new tests.

- The failure, `test_link_epaper_status.test_sticky_board_wiring` (`font.fallback = &muse_font_cjk_16;` not found in `muse_ui.c`), fails identically before and after. It isn't touched here.
- The skips are libvpx, libx264 and ppk2-api, none of which the CI job installs.

Firmware builds (ESP-IDF v6.0.1, CI's flags): `muse-waveshare-s3-175c`, `muse-m5stack-cores3` and `muse-waveshare-s3-216` all build.

- **Warnings:** none in any changed file. The 1.75C and 2.16 have 0 warnings in total. The CoreS3's 11 are all FreeRTOS `atomic.h` unused-function warnings from LVGL's managed `lv_freertos.c`.
- **Size:** each app is 0x211000 with 48% of its 4 MB slot free.
- The unpatched CoreS3 build has the same 11 warnings, line for line.
- Two boards whose behaviour shouldn't change, but which compile the changed `muse_pmu.c`, also build: `muse-waveshare-s3-175` (0 warnings) and `muse-waveshare-c6-18` (RISC-V; its 11 warnings are the same `atomic.h` ones).

## Not tested on hardware

Nothing here was run on hardware. Before this PR is opened, it gets checked on a 1.75C: a 12–14 s and a 20 s PWR hold on battery, short talks, BOOT and Settings power-off, and the boot log's `PWROFF_EN` and `last off`.

These can't be checked here:
- **The AXP2101's timing:** that clearing REG22[1] a few ms after the press stops an OFFLEVEL power-off already being counted. The datasheet doesn't say when the PMU samples the bit.
- **The CoreS3 and the 2.16**, which have no hardware here.
- **The wedged-firmware fallback**, which can't be forced on a stock build.

Fixes #277
