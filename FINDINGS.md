# Hisense HE65A6100UWTS — Firmware Analysis Findings

**Target TV:** Hisense HE65A6100UWTS (A001), VIDAA, region ZAF, running `V0000.01.00a.P0305`
**Analyzed image:** `M0609` build — `usb_MICALIDVB6886.bin` (865 MB), from 4PDA (Mail.ru mirror), free.
Same board/platform (B7500 / A6140, MStar **MSD6886**), 2022-06 build. Not the exact P0305 on the TV, but the same SoC, layout, and boot chain — valid for analysis.

## Platform
- **SoC:** MStar/MediaTek **MSD6886** (U3_ALL)
- **Bootloader:** U-Boot 2011.06 (MStar MBoot), built Jun 9 2022, Hisense vendor layer
- **OS:** VIDAA Linux, **ARM 32-bit** (glibc, busybox), Linux 2.6.16+ ABI
- **Layout:** eMMC, GPT, A/B dual-system, OTA via `oad` partition (`mmcblk0p36`)

## Image format
`M0609.zip` → `MICALIDVB6886_pkg_20220609.tar.gz` → `usb_MICALIDVB6886.bin`.
The `.bin` begins with an MStar text flashing script (`# File Partition: ...`, `filepartload`, `mmc write.p`, `mmc unlzo`) that maps every partition's offset/length/compression. Full map in `firmware/M0609/partition_map.txt`. Partitions carved to `firmware/M0609/parts/`.

## Partition filesystem identification
| Partition | Dev | Content | Signed? | Writable |
|---|---|---|---|---|
| MBOOT/MBOOTBAK | — | U-Boot bootloader | **YES** | no |
| optee/opteeB | — | OP-TEE TrustZone secure OS | **YES** | no |
| armfw | — | ARM firmware | **YES** | no |
| KL/KLB | mmcblk0p? | Kernel (uImage) | **YES** | no |
| RFS/RFSB | mmcblk0p7 | **Root filesystem (SquashFS)** | **YES** | ro |
| dtb | — | Device tree | (with kernel) | no |
| **tvservice** | **mmcblk0p18** | **Apps + libs + kernel modules** | **NO** | ro→rw |
| tvconfig | mmcblk0p20 | config | NO | ro |
| tvcommon | mmcblk0p21 | | NO | rw |
| tvcustomer | mmcblk0p22 | Customer/appcache | NO | rw |
| tvdatabase | mmcblk0p23 | | NO | rw |
| certificate | mmcblk0p30 | keys/certs | NO | rw |
| modeldata/criticaldata/factory | p24/25/33 | | NO | ro |
| usrfs/optfs/varlfs | | more app FS (LZO) | NO | rw |

## Secure boot (hardware root of trust)
The bootloader RSA/SHA256-verifies the chain **MBOOT → optee/armfw → KL (kernel) → RFS (rootfs)**.
Keys are **eFUSE-backed AES/RSA** (`AES Encrypt fail by Efuse key`, `RSA Authentication Fail`, `SHA256 Authentication Fail`).
On failure: fall back to A/B other system, else `Run USB upgrade to rescue`.
- Authenticated set string: `MBOOT,KL,RFS,armfw,optee`
- Kernel cmdline: `root=/dev/mmcblk0p7 rootfstype=squashfs rw rootwait`
- **No dm-verity** (not needed — bootloader checks the whole squashfs).

→ Replacing the kernel or rootfs and booting normally = signature fail. Custom **kernel** is not feasible without the eFUSE private key (would need a bootloader/OP-TEE bug or hardware glitch).

## The opening — unsigned execution paths
Mount points the **signed** rootfs uses resolve into **unsigned** partitions:
```
/applications -> /tvservice/applications   (mmcblk0p18, unsigned)
/mslib        -> /tvservice/mslib          (unsigned)
/config       -> /tvconfig/config          (unsigned)
/Customer     -> /tvcustomer/customer      (unsigned)
```
- The main TV app stack (`svrmgr`, `MiSysSrv`, VIDAA UI) launches from `/applications/bin` = **tvservice (unsigned)**.
- Shared libraries load from `/mslib` = **tvservice (unsigned)**.
- **Kernel modules load from `/tvservice/modules/*.ko`** (`/etc/insmoddrv.sh`) = **unsigned**. If the kernel doesn't enforce module signatures (likely), this allows **custom kernel-level code**.
- No integrity check on `tvservice` found in the boot scripts.

## Root access
- `/etc/inittab`: `ttyS0::once:-/bin/sh` → **UART serial console gives a root shell with NO login/password.**
- `/etc/passwd`: `root::...:/bin/bash` (empty pw field); `/etc/shadow` root hash uses a 1-char (space) salt — weak, but not needed given open UART.
- Debug-flavored build (telnetd user, many `demo_*` accounts).

## Feasibility verdict
| Goal | Verdict | Path |
|---|---|---|
| Live root access | **Easy** | UART ttyS0 root shell (no auth) |
| Fix TV issues | **Doable** | Root → patch unsigned config/app partitions |
| Run own code / userland | **Doable** | Modify unsigned `tvservice` (apps, libs) |
| Custom kernel module | **Confirmed** | Kernel has **no module-sig verification** (CONFIG_MODULE_SIG off). Build against vermagic `4.9.44+ SMP preempt mod_unload ARMv7`, drop `.ko` in `/tvservice/modules`, `insmod`. |
| Boot own kernel/Linux | **Unconfirmed — possibly an env-config gap** | MBOOT's default `bootcmd` is plain `mmc read + bootm` (CRC only); KL/RFS RSA auth exists but is wired **only** to OTA/recovery/console (`authenticate`/`SecureBootCmd`), not the normal boot path. If the live TV's saved `bootcmd` matches, a modified kernel may boot without a sig check. **Must confirm live `bootcmd` via UART `printenv`.** Chain-of-trust every boot covers MBOOT/OPTEE/ARMFW only. |

## Next steps
1. **Confirm the real TV has an accessible UART header** (3–4 pins near the SoC, 3.3 V) → get root shell.
2. From root: `mount -o remount,rw /dev/mmcblk0p18 /tvservice`, inspect, back up, then stage custom binaries/modules.
3. ~~Static analysis of the boot/verify logic in **MBOOT**~~ → **DONE + corrected** (see Update 2026-10-14). Normal boot path does **NOT** RSA-verify KL/RFS — it's `mmc read` + `bootm` (CRC only); auth runs only at OTA/recovery. Earlier "every boot" claim was wrong.
4. ~~Check kernel `.ko` loading / module signing~~ → **DONE** (see Update below). Module signing is **off**.
5. For "fix issues": diff this M0609 against the exact P0305 build later, and identify the config/app files behind the specific bugs.
6. Extract the `tvservice` partition (LZO blob @ `0x1e51a000`, len `0x24271dd` in the `.bin`) to get the full ~30-module set referenced by `/etc/insmoddrv.sh` (only `kdrv_xc.ko`, `utpa2k.ko` are in the rootfs).

_Artifacts: `firmware/M0609/parts/` (carved partitions), `RFS_root/` (extracted rootfs), `partition_map.txt`, `finding_mimo/` (MBOOT Ghidra headless + symbol crack)._

> **Full component-by-component analysis of every partition:** see `finding_mimo/FIRMWARE_ANALYSIS.md` (2026-10-14). Covers kernel, MBOOT, rootfs (init/accounts/caps/tools/symlinks), the complete `tvservice` (50 modules, app stack, 25 OP-TEE TAs — all unsigned), OP-TEE, dtb, and all config/data/key partitions (certificate, cacert, critical/factory/model/serial data, DBs), with a consolidated access-surface summary.

## Update 2026-10-13 — static deep-dive (kernel + MBOOT auth)

### Kernel (KL) facts
- **Linux 4.9.44+**, ARM 32-bit, **uncompressed** uImage payload (`comp=none`), load/entry `0x20208000`.
  Banner: `#2 SMP PREEMPT Thu Jun 9 19:50:05 CST 2022`, gcc 4.8.3 (Sourcery CodeBench 2014.05).
- Boot cmdline (default env): `root=/dev/mmcblk0p7 rootfstype=squashfs rw rootwait`, `console=ttyS2,115200n8`, `bootdelay=0`.

### FINDINGS #4 — kernel module signing: **OFF (decisive)**
Searched the decompressed kernel image:
- `~Module signature appended~` (MODULE_SIG_STRING) — **absent**
- `mod_verify_sig`, `module_sig_check`, PKCS#7 / X.509 parser strings — **all absent**

MODULE_SIG_STRING compiles in **iff** `CONFIG_MODULE_SIG=y`; its absence means signature verification isn't in this kernel. The lone `module.sig_enforce` token is just a parameter-name string in the kernel's param/blacklist table (adjacent to `nomodule`, `module_blacklist`), not an active check.
- Shipped modules (`kdrv_xc.ko`, `utpa2k.ko`) are plain ELF with **no signature trailer**, confirming unsigned loading at boot.
- `vermagic=4.9.44+ SMP preempt mod_unload ARMv7`; no `__versions`/modversions section → **CONFIG_MODVERSIONS off**, so only the vermagic string must match (no per-symbol CRC). `insmod --force` or a matching-vermagic build loads.

→ **Custom kernel-level code via `.ko` is confirmed feasible** once you have root to write `/tvservice` and run `insmod`.

### FINDINGS #3 — MBOOT KL/RFS verification ⚠️ CORRECTED 2026-10-14
> The original claim here ("full RSA auth of KL/RFS every boot") was **wrong** — it was inferred from the mere *presence* of `do_Authenticate_FileSystem` symbols, not from tracing the boot path. Control-flow analysis (agent MiMo, Ghidra) + independent Capstone verification show the opposite. Full write-up + evidence: `finding_mimo/SECURE_BOOT_TIMING.md` and `finding_mimo/HANDOFF.md`.

**On the normal (non-OTA) boot path, MBOOT does NOT RSA/SHA-verify KL or RFS.** The default `bootcmd` is `mmc read.p … KL …; bootm` — a plain read + `bootm` (CRC/header sanity + A/B fallback only). RSA-2048 partition authentication code **exists** but is reachable **only** via the `authenticate` / `SecureBootCmd` / `readKL→bootKL` commands, which appear only in OTA/recovery env scripts (`recoverycmd = "authenticate recovery 0x25000000; bootm …"`) and the console — **not** in the default `bootcmd`.

Independently verified (Capstone call-graph on `mboot.bin`):
- VA map confirmed: cmd-table stores link VAs (`file_off+0x256F7E00`); runtime = +0x200.
- `do_bootm` (`0x257b4d24`): 65 callees, **none** crypto/auth.
- `FUN_2575d994` (partition-auth) has **exactly one** caller — the `authenticate` handler.
- **No** auth/crypto function is reachable from `do_bootm`, `MstarToKernel`, `MstarProcess`, or the `mmc` handler.
- Auth functions have **zero** data-pointer refs → cannot be invoked by indirect dispatch (closes the "transparent authenticate-on-read" caveat).
- Every-boot chain of trust (via prior **sboot** stage, eFUSE keys) covers **MBOOT + OPTEE + ARMFW** only; KL/RFS are not in it for this config.

The **partial-authentication** feature is real (env `partial_partition`, `IsDoingPartialAuthentication`, `Enable/DisablePartialAuthentication` via `setenv`/`saveenv`) but only tunes *how much* of a partition the `authenticate` command hashes — it is not on the normal boot path and not a bypass.

→ **Implication:** "verify-on-install, CRC-on-boot" design. If the **live TV's saved `bootcmd`** matches this default, a modified KL/RFS could boot without a signature failure — "boot own kernel" would be an **env-config gap, not a crypto wall.** Not yet confirmed on the real unit (this is the default env of a downloaded image). **Decisive check: UART `printenv` → read the live `bootcmd`.** The code-execution route (unsigned `tvservice` + unsigned `.ko`) stands regardless.

**Tooling note:** REA's Ghidra provider on this Windows host rejects ARM targets (x86/x64 PE only); MBOOT analysis uses the project's own Ghidra-headless output under `finding_mimo/` plus independent Capstone 5.0.7 scripts.
