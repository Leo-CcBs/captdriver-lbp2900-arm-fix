# captdriver: fix LBP2900 hang on ARM64 (aarch64)

Patched copy of **captdriver** — a free CAPT driver for Canon LBP2900 / LBP3000 / LBP3010 series laser printers — with a fix for a bug that makes printing **hang forever after the first page** on ARM64 systems.

Tested on UOS Desktop 20 (aarch64) with a Canon LBP2900.

## Symptoms

- `lpstat -p` shows the printer stuck at `printing` / `Processing page N...` indefinitely
- Multi-page jobs print only the first page; the rest stay queued
- Occasionally one blank page comes out, then the job hangs
- Reinstalling the printer helps for exactly one test page

## Root cause

`captdriver 0.1.3` contains several **unbounded wait loops**. If the printer reports an unexpected status — a transient "no paper" flag, a delayed status update — the filter never returns and the CUPS job stays in "printing" forever.

| File | Function | Waits for |
|---|---|---|
| `prn_lbp2900.c` | `lbp2900_page_epilogue` | `page_out == page_decoding` |
| `prn_lbp2900.c` | `lbp2900_job_epilogue` | `page_completed == page_decoding` |
| `prn_lbp2900.c` | `lbp2900_page_prologue` | `CAPT_FL_BUFFERFULL` clear |
| `prn_lbp2900.c` | `lbp2900_wait_user` | panel button press |
| `capt-status.c` | `capt_wait_ready` / `capt_wait_xready` | `CAPT_FL_BUSY` clear |

Additionally, `rastertocapt.c` retried a failed page **without a limit**, so one bad page could be reprinted over and over.

## Fix

1. **Timeouts** on every wait loop: 120 s for status waits, 60 s for the button wait. On timeout the driver logs an `ERROR: CAPT: ... timeout` line and continues instead of hanging.
2. **Retry limit**: a page is retried at most 2 times; after that it is skipped with an error log instead of being re-sent forever (`CAPT_MAX_PAGE_RETRIES`).
3. Small build fixes for ARM64: `_POSIX_C_SOURCE`, missing prototypes, an extra CAPT constant.

The complete diff is `0001-fix-hang.patch` (249 lines).

## Build

Requires CUPS headers (`libcups2-dev` / `cups-devel`) and a C compiler.

```bash
cd src
gcc -std=c99 -O2 -c capt-command.c capt-status.c generic-ops.c \
    hiscoa-common.c hiscoa-compress.c paper.c printer.c \
    prn_lbp2900.c rastertocapt.c
gcc -o rastertocapt *.o -lcups -lcupsimage -lpthread
```

If your distro has no `libcups.so` symlink (runtime lib only), link explicitly:

```bash
gcc -o rastertocapt *.o -l:libcups.so.2 -l:libcupsimage.so.2 -lpthread
```

## Install

```bash
sudo install -o root -g root -m 755 rastertocapt /usr/lib/cups/filter/rastertocapt
sudo systemctl restart cups

# register the printer (adjust the USB URI to your device)
URI=$(lpinfo -v | awk '/^direct usb:\/\// && tolower($0) ~ /lbp2900/ {print $2; exit}')
sudo lpadmin -p Canon-LBP2900 -E -v "$URI" -P Canon-LBP2900.ppd
sudo lpadmin -d Canon-LBP2900
```

## Notes

- The printer may still emit one blank page and pause ~60 s when it reports a transient "no paper" / button state. The job now **recovers by itself** instead of hanging. If this happens often, check the paper pickup roller.
- Do not run the original Canon `ccpd` daemon together with this driver — it grabs the USB device.
- This driver talks CAPT directly over `usb://`; `ccpd` is not required.

## License

GPLv3 — see `COPYING`.

Based on **captdriver** by Alexey Galakhov (and contributors), with the hang-fix patch from this repository.
