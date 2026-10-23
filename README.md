# Sony ILCE-1 (A1) Firmware Static Analysis & Reverse Engineering

本项目是对 Sony ILCE-1（A1）4.00 固件及相关组件进行的**纯离线静态审计**。研究目标是理解视频 LUT、自定义快门声音相关的数据路径、菜单与处理器间的消息流，以及更新器和启动加载器的安全边界。

项目不连接相机、不运行固件、不运行 QEMU，不生成刷机包、补丁字节、漏洞利用脚本或 USB 发包器。

## 核心发现

- **LUT 路径：**A1 中存在 `.cube`／二进制 LUT 解析、用户 LUT 参数和监看 LUT 相关接口；嵌入的 `MainCPS` 属于 CodeMeter 保护运行时。普通动态符号和重定位中没有恢复出可直接调用的 LUT 提供者，因此“只改一个隐藏开关即可启用”的假设没有被证实。
- **CP 调试接口：**A1 的 `cpapp-b.bin` 注册了 UART、GPIO、SPI／SPI-GPIO 命令表。命令表通过 CP 内部注册器保存，但目前没有证据证明这些命令能从 USB、Multi 接口、外部 UART 或启动阶段到达。
- **更新器与 Jiritsu：**静态审计确认了普通 `verifyFile` 恒成功、CRC 短读、32 位长度相加环绕候选、临时文件按请求长度写入，以及 Jiritsu 路径拼接等局部缺口。这些观察没有证明完整签名链或主固件刷写可以被绕过。
- **启动加载器：**`loader_nor.bin` 有工程调试模式、SPI 从设备握手、GPIO 配置、CP escape 和 PCIe STM 启动传输路径；未在 loader 文件中确认 A7M4 同款 BootROM 串行入口。

## 交接给硬件研究者

如果要继续推进，第一步必须是被动物理取证：取得准确的主板版本和接口复用图，用逻辑分析仪记录上电、复位、候选 SPI/GPIO 链路、CP escape 和 PCIe STM ready 的时间关系，再与 loader 的静态地址和日志对应。当前没有确认的 A1 针脚、波特率、命令或 BootROM 触发条件，不能把 A7M4 的线序或协议直接移植过来。机器可读的启动审计证据位于 [`evidence/round4_bootstrap_audit_evidence_20261024.json`](evidence/round4_bootstrap_audit_evidence_20261024.json)。

## 仓库结构

```text
.
├── README.md
├── LICENSE.recommendation.md
├── .gitignore
├── docs/
│   ├── README.md
│   ├── release-guide.md
│   └── analysis/
│       ├── a1-vulnerability-theory-memo-20261024.md
│       └── a1-bootstrap-hardware-backdoor-audit-20261024.md
└── evidence/
    ├── README.md
    ├── SHA256SUMS.txt
    └── round4_bootstrap_audit_evidence_20261024.json
```

`docs/analysis/` 保存面向读者的分析报告；`evidence/` 保存与报告对应的结构化证据。原始固件、`BODYDATA.DAT`、解密密钥、完整 ELF／BIN／SO 镜像和运行时转储不应提交到公开仓库。

## 免责声明

> 本项目仅限纯离线学术研究、软件供应链研究和防御性安全分析。项目不提供刷机包、补丁、漏洞利用脚本、USB 发包器或破解教程，也不指导对设备执行写入。相机固件修改、启动模式尝试和错误刷写可能导致设备永久变砖、数据丢失或失去保修。任何人在任何设备上采取行动前，都必须自行确认授权、备份和恢复条件；项目作者及贡献者不对设备损坏、数据丢失、保修问题或其他后果承担责任。

## 发布与许可证

研究报告和整理后的证据建议使用 **CC BY-NC-SA 4.0**，以保留署名、非商业和相同方式共享要求。未来独立编写的分析工具可以单独使用 MIT 或 Apache-2.0。Sony 固件及其派生内容的版权、许可和再分发边界需要单独确认；本仓库不声明拥有或重新授权这些专有内容。

发布步骤见 [`docs/release-guide.md`](docs/release-guide.md)，证据文件说明见 [`evidence/README.md`](evidence/README.md)。
