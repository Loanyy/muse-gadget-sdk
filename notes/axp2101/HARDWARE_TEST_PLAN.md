# Hardware test plan: #277 fix on the Waveshare ESP32-S3-Touch-AMOLED-1.75C

The fix is on `fix/axp2101-hold-off-during-talk` (one commit on upstream 1a1682f). This folder also has the same commit as `0001-esp32-don-t-let-a-talk-on-the-AXP2101-s-power-key-ou.patch`, for applying to your own tree.

The one thing the host tests can't show is what this bench check is for: does clearing REG22[1] a few ms *after* PWR goes down stop the AXP2101 from powering off at 10 s? The datasheet says REG22[1] gates the OFFLEVEL power-off, but it doesn't say when the PMU samples it.

## 0. Get a build onto the board

You choose one of these. Either way, keep your rollback image at hand.

### Option 1: upstream-based build (the PR as it is)

```sh
cd esp32
. ~/esp/esp-idf-v6.0.1/export.sh          # or wherever your v6.0.1 is
tools/muse/board.sh build s3              # profile waveshare-s3-175c, build dir build-muse-waveshare-s3-175c/
# equivalently, as CI does:
idf.py -B build-muse-waveshare-s3-175c -DIDF_TARGET=esp32s3 \
  -DSDKCONFIG=build-muse-waveshare-s3-175c/sdkconfig \
  -DSDKCONFIG_DEFAULTS="sdkconfig.defaults;devices/sdkconfig.muse;devices/sdkconfig.muse-waveshare-s3-175c" build
```

`tools/muse/board.sh flash s3` writes everything in `flash_args`: bootloader, partition table and app. If you want to keep your current layout and NVS (pairing, Wi-Fi) and flash only the app, as you do now:

```sh
# Which slot is running? The boot log's "Loaded app from partition at offset 0x20000"
# means ota_0 (0x20000); 0x420000 means ota_1 (partitions_muse.csv).
python -m esptool --chip esp32s3 -p /dev/ttyACM0 -b 460800 --before default-reset --after hard-reset \
  write-flash 0x20000 build-muse-waveshare-s3-175c/muse-gadget.bin
```

The upstream build signs the app with `dev_signing_key.pem`, as your builds do, assuming you didn't change the key.

### Option 2: apply the patch to your private tree

```sh
git am /path/to/0001-esp32-don-t-let-a-talk-on-the-AXP2101-s-power-key-ou.patch
# or, if your tree has drifted: git apply -3 the same file
```

It touches `muse_pmu.c`/`.h`, three board files and adds tests. Only `muse_pmu.c`, `muse_pmu.h` and `boards/board_waveshare_s3_175c.c` matter for this board. Run the host test afterwards: `cd esp32 && python3 -m unittest tests.test_muse_pmu -v`.

### Optional bench-only read-back (don't commit)

The fix logs nothing per press, on purpose. To see the register flip on every PWR edge, add this temporarily at the end of `muse_pmu_set_key_shutdown()`, before `return ESP_OK;`:

```c
    uint8_t now = 0;
    rd(REG_PWROFF_EN, &now);
    ESP_LOGI(TAG, "key shutdown %s: PWROFF_EN %02x", on ? "on" : "off", now);
```

## 1. Boot log

Watch the serial log (`idf.py -p /dev/ttyACM0 monitor`, or your usual console) from a reset.

Expected:

```
muse_pmu: AXP2101 ready (hardware PWR hold-off 10 s, PWROFF_EN xx, last off yy)
```

- `xx` must have bit 1 set: 0x02, 0x03, 0x06 or 0x07. Bit 2 (over-temperature) is 1 by default, and bit 0 is the eFuse's power-off/restart choice. **Write down the exact value.**
- `yy` is REG21, the PWROFF status, which says what powered the PMU off. Its bits:

  | Bit | Meaning |
  |---|---|
  | 0 | PWR held past OFFLEVEL (the #277 power-off) |
  | 1 | software (REG10[0], Settings → Power off) |
  | 2 | PWRON low in EN mode |
  | 3 | VSYS under voltage |
  | 4 | VBUS over voltage |
  | 5 | DCDC under voltage |
  | 6 | DCDC over voltage |
  | 7 | die over-temperature |

  It's only reset by a power-on reset (battery removed), so write it down. It may show an earlier power-off.
- The boot reaches `starting`, with no panic or reboot loop (esp32/AGENTS.md, "Before you hand back work").

**Nice cross-check (optional, do it first):** on your *current* build, reproduce #277 by holding PWR for 12 s until the board dies. Then power on with PWR and flash or boot the fixed build. The first boot log should show `last off` with bit 0 set (e.g. `01`). That proves REG21 reads what the datasheet says.

## 2. Long talk: the fix itself (on battery, USB unplugged)

This is the case the issue is about, so test on battery. Without USB there's no serial log, so judge 2a, 2c and 2d by behaviour, then plug in and read `last off` on the next boot: bit 0 must not be newly set. 2b repeats 2a on USB with the log.

| # | Do | Expect |
|---|---|---|
| 2a | Paired and idle, hold PWR (top) **12–14 s**, then release | The board **stays on**. The log shows `talk key: press`, and `talk key: release` about 12–14 s later. Recording stops at release, and the voice note uploads and gets a reply. |
| 2b | Same as 2a, on USB power | Same. |
| 2c | Hold PWR **20 s** | Recording ends at the 15 s cap (`MAX_FRAMES`, which includes the pre-roll) while PWR is still held, and the note goes. The board **stays on** through 20 s. Record what happens at release; nothing new should start. |
| 2d | Screen off / asleep on battery (wait for the display to pause, `low power: display paused` in the log), then hold PWR 12 s | Wakes on the press (GPIO3 wakes `wait_buttons`), records, and **stays on**. |
| 2e | With the bench read-back: press and release PWR | `key shutdown off: PWROFF_EN` with bit 1 clear on press, then `key shutdown on:` with bit 1 set on release. |

If 2a or 2c powers the board off, REG22[1] doesn't stop an OFFLEVEL power-off already being counted, so the fix doesn't work. Don't open the PR. Report back, and the fallback is option (A), BOOT talks.

## 3. Short talks and normal use

| # | Do | Expect |
|---|---|---|
| 3a | Tap-talk 2 s | Works as before: the note goes and a reply arrives. |
| 3b | Ten quick taps of PWR in a row, then a 12 s hold | No power-off. The register isn't left off by a missed edge (with the read-back, the last line is `on`). |
| 3c | BOOT (bottom) short press | Sleep, as before. |
| 3d | BOOT held 1.5 s ("HOLD TO POWER OFF") | Clean power-off. Then PWR turns it back on. The next boot log's `last off` should show bit 1 (software). |
| 3e | Settings → Power off | Clean power-off. PWR turns it back on. |
| 3f | Phone pairing confirm with PWR (if you can trigger a pairing) | Confirms as before (cf. #65). |

## 4. Edges worth one try each

| # | Do | Expect / record |
|---|---|---|
| 4a | Power on with PWR and **keep holding** ~12 s | Not covered by the fix: a PWR held through boot isn't a press (`s_pwr.pressed` at init), so nothing turns key shutdown off. Record whether it powers off. Either result is the same as before the fix, not a regression. |
| 4b | Reset (RST, or `esptool --after hard-reset`) **while holding PWR** in a talk, then let go and hold 12 s after boot | After the reset, init turns key shutdown back on (boot log `PWROFF_EN` bit 1 set), and a talk then protects itself as in 2a. |
| 4c | Talk 12 s while the phone/Wi-Fi is disconnected | Stays on. The offline note queues as before. |

## 5. What this bench can't show

- **M5Stack CoreS3 and Waveshare 2.16:** same mechanism through the PMU key latch, covered only by host tests. On the 2.16, a PWR talk started while Wi-Fi naps can go unseen for up to 10 s. That limitation is noted in the PR.
- **The wedged-firmware fallback.** A firmware that's stuck never sees the press, so the 10 s hold should still power off. You can't force a wedge on a stock build. Step 4a (no firmware tracking) is the closest stand-in, and the datasheet's default behaviour covers the rest.
- **A wedge while PWR is held:** the fallback stays off until a reset. Known and noted, not testable.
- **Other AXP2101 boards' eFuse defaults for REG22[1]** (C6 1.8/2.06): init now turns the bit on there too.

## What to send back

The boot-log line (`PWROFF_EN`, `last off`), pass or fail for 2a–2d and 3a–3e, what 4a did, and any unexpected log lines. Those go into the PR's "Hardware" section before it's opened.
