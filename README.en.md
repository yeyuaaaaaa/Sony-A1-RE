English | **[简体中文](README.md)**

# Sony ILCE-1 (A1) Firmware Static Analysis & Reverse Engineering

This project is a **pure offline static audit** of Sony ILCE-1 (A1) firmware 4.00 and related components. The research focuses on the data paths for video LUTs and custom shutter sounds, low-level debug interfaces, message flows between menus and processors, and the security boundaries of the updater and boot loader.

The project does not connect to a camera, execute firmware, run QEMU, generate flashing packages or patch bytes, or provide exploit scripts or USB packet senders.

## Core findings

- **LUT path:** A1 contains .cube and binary LUT parsing, user LUT parameters, and monitoring LUT interfaces. The embedded MainCPS is part of a CodeMeter-protected runtime. Ordinary dynamic symbols and relocations did not recover a directly callable LUT provider, so the assumption that a single hidden switch is enough to enable the feature remains unconfirmed.
- **CP debug interfaces:** A1 cpapp-b.bin registers UART, GPIO, and SPI/SPI-GPIO command tables. The tables are stored by an internal CP registry, but there is currently no evidence that these commands are reachable from USB, the Multi interface, an external UART, or the boot stage.
- **Updater and Jiritsu:** The static audit confirmed local gaps including an ordinary verifyFile path that always returns success, CRC short reads, a candidate 32-bit length wraparound, temporary-file writes based on the requested length, and Jiritsu path concatenation. These observations do not prove that the complete signature chain or main firmware flashing can be bypassed.
- **Boot loader:** loader_nor.bin contains an engineering debug mode, an SPI slave handshake, GPIO configuration, CP escape handling, and a PCIe STM startup transfer path. The loader files do not confirm an A7M4-equivalent BootROM serial entry.

## Handoff for hardware researchers

Anyone continuing this work should begin with passive physical evidence gathering: identify the exact motherboard revision and interface multiplexing, then use a logic analyzer to record power-up, reset, candidate SPI/GPIO links, CP escape, and PCIe STM ready timing. Correlate those observations with loader addresses and logs. The A1 pins, baud rate, command set, and BootROM trigger condition are not confirmed, so A7M4 wiring or protocols must not be transferred by assumption. Machine-readable bootstrap evidence is available in [evidence/round4_bootstrap_audit_evidence_20261024.json](evidence/round4_bootstrap_audit_evidence_20261024.json).

## Repository structure

- README.md — Chinese entry point
- README.en.md — English entry point
- LICENSE.recommendation.md
- docs/README.md
- docs/release-guide.md
- docs/analysis/a1-vulnerability-theory-memo-20261024-en.md — [Updater and Jiritsu vulnerability theory memo](docs/analysis/a1-vulnerability-theory-memo-20261024-en.md)
- docs/analysis/a1-bootstrap-hardware-backdoor-audit-20261024-en.md — [Boot loader and hardware debug-interface audit](docs/analysis/a1-bootstrap-hardware-backdoor-audit-20261024-en.md)
- evidence/README.md
- evidence/SHA256SUMS.txt
- evidence/round4_bootstrap_audit_evidence_20261024.json

The docs/analysis directory contains reader-facing analysis reports; evidence contains the corresponding structured evidence. Original firmware, BODYDATA.DAT, decryption keys, complete ELF/BIN/SO images, and runtime dumps must not be committed to the public repository.

## Disclaimer

> This project is limited to pure offline academic research, software supply-chain research, and defensive security analysis. It does not provide flashing packages, patch bytes, exploit scripts, USB packet senders, or cracking instructions, and it does not instruct anyone to write to a device. Firmware modification, boot-mode experiments, and incorrect flashing can permanently brick a camera, cause data loss, or void the warranty. Anyone taking action on any device must independently confirm authorization, backups, and recovery conditions. The authors and contributors are not responsible for device damage, data loss, warranty issues, or other consequences.

## Publication and licensing

The research reports and curated evidence are recommended for CC BY-NC-SA 4.0, preserving attribution, non-commercial use, and share-alike requirements. Independently written analysis tools may use MIT or Apache-2.0 separately. The copyright, licensing, and redistribution boundaries for Sony firmware and derivative material require separate review; this repository does not claim ownership of or relicense proprietary content.

See docs/release-guide.md for the publication procedure and evidence/README.md for the evidence-file notes.
