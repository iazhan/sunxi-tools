# `sunxi-fel` with SPI NAND read support

Fork of [sunxi-tools](https://github.com/linux-sunxi/sunxi-tools) that adds two **read-only**
SPI NAND commands to `sunxi-fel`, on top of upstream `master`. Everything else is upstream code.

Branch: **`f1c100s-spinand`** (based on `master`).
The single functional commit is also exported as a plain patch in
[`patches/`](patches/) so it can be re-applied on a newer upstream tree.

## Why

Upstream `sunxi-fel spiflash-*` speaks **SPI NOR only**. It cannot read SPI NAND:

* `spiflash-read` issues a bare `0x03` with a 24-bit byte address. On SPI NAND `0x03` is
  **READ-FROM-CACHE**, not a continuous read: it never loads a page. You get the page that
  happens to sit in the cache register after reset — for every offset, at every address.
* SPI NAND needs a two-phase access: load the page into the cache (`0x13`), then read the cache
  (`0x03`). No amount of address arithmetic turns the NOR command into a NAND read.

This is the same gap the SPI NAND framework in U-Boot / Linux had: a table-driven probe fails on
unknown IDs, whereas FEL does not need a chip table at all — a blind read only needs the page
size, and picks the rest up from the data itself.

## What is added

| command | behaviour |
|---|---|
| `sunxi-fel spinand-id` | sends `0x9F` (READ ID) and prints the raw 8 bytes it clocks back |
| `sunxi-fel spinand-read <offset> <length> <file>` | reads `length` bytes of the NAND **data area** starting at byte `offset` into `file` |

```console
$ sunxi-fel spinand-id
SPI NAND raw ID (0x9F) = 7F 7F 7F 00 00 00 C8 01
$ sunxi-fel spinand-read 0 0x200000 first2M.bin
```

Notes on the ID: the byte order depends on how the controller shifts the command out, so the
output is printed **raw** — read it as a human and locate the vendor/device pair
(e.g. `C8 01` = GigaDevice, device `0x01`). The ID is evidence, not a prerequisite: the read
command never looks at it.

## How the read works

1. `0x13` PAGE-READ with a 24-bit row address — the value is the **page number itself**
   (`page = offset / 2048`), *not* `page << 12`. Getting this wrong shows up as "every large
   offset returns page 0".
2. wait for the page load on the host side (~1 ms, covers the 250 µs `tR` of common parts),
3. `0x03` READ-FROM-CACHE with the 16-bit column address,
4. copy `column..page_end` out, advance, repeat.

Assumptions and consequences:

* **2048-byte pages** — the dominant SPI NAND geometry. A chip with 4096-byte pages will produce
  visibly shifted/garbled data; that is the signal to change the constant.
* the **spare/OOB area is not read** — only the data area. No ECC engine is involved, so the bytes
  are exactly what the flash stores (raw read).
* data is transferred through the existing SRAM batch engine
  (`fel-remotefunc-spi-data-transfer.h`), one page per remote call, so throughput is bound by FEL
  round-trips (~32 KB/s on a full-speed USB link in a VM) — not by the SPI clock.

## Safety

* Only **read** opcodes are issued: `0x9F`, `0x13`, `0x03`.
  No write-enable / program / erase opcode exists in the added code, and no
  `spinand-write` / `spinand-erase` command is added — unlike `spiflash-write`, there is no
  path here that can modify the flash.
* The commands do not touch the environment, the bootloader or any medium: they only DMA a helper
  into SRAM/DRAM and read the SPI bus.

## Build

```bash
sudo apt install libusb-1.0-0-dev libfdt-dev pkg-config
make
./sunxi-fel
```

`libfdt-dev` is required by current upstream `master` (`fit_image.c`); older trees did not need
it. `libusb-1.0-dev` is the only other dependency (no zlib).

## How this was verified

Board: Allwinner **F1C100S** + GigaDevice SPI NAND, **128 MiB**, 2 KiB pages, entered via BROM FEL.

* `spinand-id` → vendor/device `C8 01`.
* `spinand-read 0 2048` → starts with `06 00 00 ea 65 47 4f 4e 2e 42 54 30` (`eGON.BT0`, the
  Allwinner boot-0 header).
* `spinand-read 0 0x200000` → `md5 = f5efcdeb3f3c2d14762a43686c553f5d`.
* full chip (128 MiB) → `sha256 = cf385a7d7f7c175b440895452d6257f6ce25e26ed0bdf07a572a1cb06ba2d333`,
  last non-`0xFF` byte at 125.54 MiB.
* The same flash was dumped independently through the Linux kernel MTD/ECC path and with
  `xfel spinand read`; all three dumps are byte-identical.

## Files changed

```
 fel-spiflash.c | 123 ++++++++++++++++++++++++++++++++++++++++++++++++++++++++-
 fel-spiflash.h |   5 +++
 fel.c          |  10 +++++
 3 files changed, 137 insertions(+), 1 deletion(-)
```

* `fel-spiflash.c` — `spinand_exec()` (build one command batch and run it) plus
  `aw_fel_spinand_read()` / `aw_fel_spinand_info()`, inserted after `aw_fel_spiflash_read()`;
  and two lines added to `aw_fel_spiflash_help()` so the new commands appear in the usage text.
* `fel-spiflash.h` — prototypes for the two new functions.
* `fel.c` — `spinand-id` / `spinand-read` dispatch branches next to `spiflash-info`.

The identical change is kept as a plain patch in
[`patches/0001-fel-spiflash-add-SPI-NAND-read-support.patch`](patches/), so it can be applied to
any tree (`git apply`) and committed there — useful when rebasing onto a newer upstream:

```
fel-spiflash: add SPI NAND read/info commands

SPI NAND cannot be read with the NOR command set: 0x03 alone is
READ-FROM-CACHE and never loads a page, so spiflash-read returns the
cache-register contents at every offset. Add the two-phase access
instead: 0x13 PAGE-READ (24-bit row address = the page number itself)
followed by 0x03 READ-FROM-CACHE (column address inside the page).

Two read-only commands are added:
  spinand-id                    print the raw 0x9F response
  spinand-read addr len file    dump the NAND data area to file

Only read opcodes (0x9F/0x13/0x03) are issued; no program/erase path is
added. A page size of 2048 bytes is assumed, only the data area is
dumped (no OOB/ECC), and the ID is reported as evidence only - the read
path never consults a chip table.
```


## Rebasing on newer upstream

The three touch points above are small and independent of the rest of the file, so
`git rebase upstream/master` should stay trivial. If upstream ever grows its own SPI NAND
support, this branch becomes unnecessary.

## License

GPL-2.0 (same as upstream, see `LICENSE.md`). The added code is a downstream addition; original
sunxi-tools copyright and license headers are untouched.
