# Direct AHCI and DMA32 support for HDDSuperTool

This branch adds a standalone direct-AHCI path for HDDSuperTool on Linux. It
was developed to communicate with a WD ROYL drive that spun normally but never
became ATA-ready, so the normal kernel block-device and SG_IO paths were not
available.

This is experimental data-recovery software. A wrong controller address,
port, script, target, or ROM can destroy data or leave a drive unable to start.
Use a dedicated SATA controller, keep the patient disk read-only whenever
possible, and take two matching reads before writing anything.

## What changed

- Initialize the direct-I/O feature table and AHCI register offsets in the
  standalone tool.
- Connect and map the requested AHCI port when explicit HBA/port addresses
  bypass the interactive device chooser.
- Allocate command, FIS, and data buffers below 4 GiB through the existing
  kernel helper for 32-bit-limited HBAs.
- Program 64-bit AHCI CLB/FB registers as two 32-bit MMIO writes.
- Wait for `PxCI` slot completion instead of treating an all-zero `PxTFD` as a
  completed command.
- Stop the command/FIS engines before restoring addresses and releasing DMA
  memory.
- Correct the page orders passed to `free_pages()` by the helper module.
- Synchronize the script-line counter used by the shared command parser.
- Add optional AHCI command tracing with `HDDSUPERTOOL_AHCI_TRACE=1`.

## Build

On a Debian/Ubuntu system, install a compiler, the running kernel's headers,
`xxd`, libcurl development files, and the legacy libusb development package.
Then run:

```sh
make dma32
```

That produces:

- `hddsupertool-dma32`
- `driver/hddsuperclone_driver.ko`

The binary embeds the absolute module path at build time. Override it when
needed:

```sh
make dma32 DMA32_MODULE_PATH=/opt/hddsupertool/hddsuperclone_driver.ko
```

The helper module is loaded when direct mode starts and unloaded during normal
cleanup. The program therefore needs root privileges.

## Read-only recovery-state module dump

`hddscripts/wd_royl_dump_mod_patient` skips ATA IDENTIFY and validates the
ROYL header, returned module ID, plausible length, and 32-bit additive checksum
before preserving a service-area module. It does not write ROM, service-area,
or user sectors.

Use controller values obtained for your own dedicated HBA; the values below
are placeholders:

```sh
sudo ./hddsupertool-dma32 \
  --ahci \
  --hbaaddress HBA_MMIO_BASE \
  --portnumber PORT_NUMBER \
  --portaddress PORT_MMIO_BASE \
  -f ./hddscripts/wd_royl_dump_mod_patient \
  mod=0x102 \
  file==/safe/destination/mod0x102.bin
```

To inspect the direct-AHCI state machine:

```sh
sudo env HDDSUPERTOOL_AHCI_TRACE=1 ./hddsupertool-dma32 ...
```

Never copy HBA or port addresses from another machine. Confirm the PCI BAR,
AHCI port, model, serial, and cabling each time. Do not use this path through a
USB-to-SATA bridge.

## ROM writes

The upstream `wd_royl_write_rom` script remains available, but it is not a
generic PCB-swap solution. WD ROMs contain drive-specific adaptive data, and a
checksum-valid ROM from the same model can still be wrong for the patient.
Before any write:

1. Preserve two identical ROM reads when the drive permits it.
2. Validate every ROM block and resident module checksum.
3. Preserve patient-specific adaptive modules independently.
4. Verify the candidate's size, family, firmware marker, and changed ranges.
5. Read the ROM back after writing and compare it byte-for-byte before power
   cycling.

No firmware images or patient data are included in this branch.

## Tested context

- Base source commit: `7f26db3a662feb1ee3f89d2fda52c136ade6b449`
- Linux direct AHCI on a dedicated controller
- WD5000AAVS ROYL/Marvell-family recovery case

The changes solved one real recovery case but have not been broadly tested
across controllers, kernels, or drive families.
