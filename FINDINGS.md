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
| Custom kernel module | **Likely** | Drop `.ko` in `/tvservice/modules` (if modsign off) |
| Boot own kernel/Linux | **Hard** | Blocked by eFUSE secure boot; needs bootloader/TEE exploit or glitch |

## Next steps
1. **Confirm the real TV has an accessible UART header** (3–4 pins near the SoC, 3.3 V) → get root shell.
2. From root: `mount -o remount,rw /dev/mmcblk0p18 /tvservice`, inspect, back up, then stage custom binaries/modules.
3. Static analysis of the boot/verify logic in **MBOOT** (Ghidra/REA) to check *when* RFS is verified (every boot vs post-OTA only) — a gap there could widen options.
4. Check kernel `.ko` loading: whether `module.sig_enforce` / CONFIG_MODULE_SIG is set (decompile kernel or read dtb/cmdline).
5. For "fix issues": diff this M0609 against the exact P0305 build later, and identify the config/app files behind the specific bugs.

_Artifacts: `firmware/M0609/parts/` (carved partitions), `RFS_root/` (extracted rootfs), `partition_map.txt`._
