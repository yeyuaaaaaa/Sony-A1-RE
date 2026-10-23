# A1 4.00 Boot Loader and Hardware Debug-Interface Audit

## Audit boundary

This report uses only offline A1 4.00 extracts. It does not connect to a camera, run the loader, run CP, run QEMU, or transfer A7M4 addresses, wiring, or commands to A1.

Key inputs:

| File | SHA-256 | Description |
|---|---|---|
| loader_nor.bin | 03facf97b5becd640b4dd9248308c827d763695e6dbf905a8f9ee1205abdadb2 | AArch64; load address and entry point 0xFE049000 |
| loader_spi.bin | 314d372d0d2b9883c3b9dc04b40389edb3100e6d5ef529633cc427c3d0cab72e | 39,184 bytes; no clear UART/serial-debug strings found |
| cpapp-b.bin | 5c8528f46f5a8aa49179089c82b5944ef6b260a859e30a93f53a9b46c4208b12 | ARM Thumb CP image |
| tomco.txt | 2a51c104695392454a930da8368fac53101efd10b862f9538eb6994b7da948ef | A1 CP static configuration containing console_enable=N and uart_baudrate=460800 |

The current file set has no separate BootROM image. loader_nor.bin is a later-stage boot loader and must not be presented as BootROM.

## 1. loader_nor entry and privilege initialization

At entry 0xFE049000, the code first sets VBAR_EL1/EL2/EL3, SCR_EL3, HCR_EL2, CPACR_EL1, MAIR_EL3, and TCR_EL3, clears the relevant SCTLR registers, builds page tables and exception vectors, and then enters internal initialization. This confirms high-privilege startup code, but it does not show a secret serial protocol or an external authentication bypass.

## 2. Plaintext search results

The strings UART, UART_DEBUG, SERIAL, FASTBOOT, BOOTROM, JTAG, and SWD were not found in loader_nor.bin or loader_spi.bin. The only usb hit in loader_nor.bin is the usbchrg configuration string, which cannot be interpreted as a USB debug command.

This is only negative evidence at the string layer. BootROM may not be included in the file, and a protocol may use numeric tables or encrypted data.

## 3. Engineering-debug and hardware-control paths that do exist in loader_nor

The following are static facts about the A1 loader itself:

- 0xABCC-0xAC7C reads configuration fields, constructs 0xF101D008 + (field << 12), writes a bit mask, reads status, and prints DEBUG_MODE_SW-related information. The corresponding log string is near file offset 0x14386.
- 0xAC90-0xACD8 accesses the 0xF102E008 region, writes state, and reads bits.
- File offset 0x14331 contains LDR:BOOT_EXT(SPI_BOOT_MODE). The related code uses boot-state or configuration fields; it does not prove that an external GPIO level can directly hijack startup.
- File offsets 0x135B0, 0x135D8, and 0x13602 contain CP_ESCAPE@cetus, cpapp-b.bin@cetus, and backup.bin@cetus transfer-wait logs.
- File offsets 0x14E21 and 0x14E96 contain SPI:Slave initialize and spi_handshake_gpio_slave_init.
- File offsets 0x13740, 0x13771, and 0x13799 contain pcie_stm_comm_ready.
- The string table contains BKID_LDR_CFG_ADJUST_BOOTMODE, BKID_RAM_ADDR_DEBUGER_MODE, BKID_LDR_DEBUGER_MODE, and BKID_LDR_CFG_EMERGENCY_DETECT.

These paths show that the loader supports an engineering debug mode, SPI slave handshake, GPIO configuration, and PCIe STM startup transfer. They may still serve only factory or internal boot flows.

## 4. CP debug command tables are closed to the registration layer

The CP image contains more than isolated strings. Three command tables use 12-byte records containing a category or attribute pointer, a command-name pointer, and a Thumb function address. The common registration call is 0x5E7D9C.

### UART table

- Table address: 0x108FC40 (CP-image-relative VA).
- Registration site: 0x4EB968-0x4EB972 constructs 0x108FC40 and branches to 0x5E7D9C.
- Record count: 18, range 0xF87C40-0xF87D0C.
- Example command names and functions:
  - uart_dump -> 0x4EA795
  - pbl_uart_open -> 0x4EA869
  - pbl_uart_read_buffer -> 0x4EACB1
  - pbl_uart_txstart -> 0x4EAE05
  - pbl_uart_testmode_ena -> 0x4EB611
  - pbl_uart_test_read_buffer -> 0x4EB681
  - pbl_uart_test_write_buffer -> 0x4EB7C5
  - debug_memdump_enable -> 0x4EB8B9

### GPIO table

- Table address: 0x108F434.
- Registration site: 0x4E0CA4-0x4E0CAE constructs the table address and branches to 0x5E7D9C.
- Record count: 23, range 0xF87434-0xF8753C.
- Commands include pbl_gpio_set_func, pbl_gpio_set_sel, pbl_gpio_get_sel, pbl_gpio_set_dir, pbl_gpio_get_dir, pbl_gpio_set_enable, pbl_gpio_get_enable, pbl_gpio_set_data, pbl_gpio_get_data, pbl_gpio_set_oen, pbl_gpio_set_ien, pbl_gpio_set_drv, pull-up and pull-down controls, and register checks.

### SPI/SPI-GPIO table

- Table address: 0x108F8C0.
- Registration site: 0x4E72CC-0x4E72D6 constructs the table address and branches to 0x5E7D9C.
- Record count: 22, range 0xF878C0-0xF879BC.
- Commands include pbl_spi_open, pbl_spi_start, pbl_spi_read_data, pbl_spi_write_data, pbl_spi_set_gpio_hiz, pbl_spi_read_write_data, status_check, and several pbl_gpio helpers.

0x5E7D9C itself only stores the command tables in a CP global registry array. Static analysis does not prove that the registry is directly driven by USB, the Multi interface, PCIe, or an external UART.

## 5. Boundary of the A7M4 reference

The local reference work/reference/a7m4_custom_firmware/README.md explicitly says that A7M4 needs a custom cable and an RP2040 boot bridge, and that the project does not contain an interactive UART console, UART command line, or serial update-mode switch. It describes a serial transport into fastboot. A7M4 wiring, signal levels, commands, and payloads cannot be used to infer A1.

## 6. Graded conclusions

| Proposition | Conclusion | Confidence |
|---|---|---|
| The A1 loader contains engineering-debug and hardware-control paths | Confirmed | High |
| A1 CP contains UART/GPIO/SPI debug commands registered in internal command tables | Confirmed | High |
| These commands are reachable from external USB or the Multi serial interface | Unknown; static breakpoint | Low |
| A1 has the same serial boot mode as A7M4 | Unknown; cannot be inferred | Low |
| A directly hijackable BootROM backdoor has been identified | Unconfirmed | Low |

Final conclusion: **A physical logic analyzer is required to capture boot logs.** Until the A1 motherboard schematic and reliable interface multiplexing are available, no pin, baud rate, command, or boot timing can be specified. Hardware work should first passively record the timing from power-up to CP_ESCAPE, SPI slave, and PCIe STM ready, then correlate it with loader register accesses. Do not transmit data to an unknown interface first.

## 7. Minimum physical evidence needed to revive the project

1. Motherboard revision and interface multiplexing documentation, identifying the physical links for spi_handshake_gpio_slave_init, CP escape, and PCIe STM.
2. Passive logic-analyzer recordings of reset, candidate SPI/GPIO, and candidate serial lines during boot; capture levels, clocks, direction, and timestamps without injecting data.
3. Boot logs or error codes that can be correlated with loader-relative addresses, especially SPI_BOOT_MODE, CP_ESCAPE, and pcie_stm_comm_ready events.
4. If read-only memory access becomes available later, first verify the loader runtime regions 0xF101D008 + (field << 12) and 0xF102E008, plus the CP command registry candidate at 0x108FC40. These are module or runtime address candidates, not confirmed physical addresses.
5. The BootROM itself or chip-vendor startup documentation must still be found; the current loader file cannot substitute for that evidence.
