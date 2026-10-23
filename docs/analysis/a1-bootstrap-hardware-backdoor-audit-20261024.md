# A1 4.00 启动加载器与硬件调试接口审计

## 审计边界

本报告只使用 A1 4.00 的离线提取物，不连接相机、不运行 loader、不运行 CP、不运行 QEMU，也不把 A7M4 的地址、线序或命令移植到 A1。

关键输入：

| 文件 | SHA-256 | 说明 |
|---|---|---|
| `loader_nor.bin` | `03facf97b5becd640b4dd9248308c827d763695e6dbf905a8f9ee1205abdadb2` | AArch64，装载地址／入口 `0xFE049000` |
| `loader_spi.bin` | `314d372d0d2b9883c3b9dc04b40389edb3100e6d5ef529633cc427c3d0cab72e` | 39,184 字节，未见 UART/串行调试明文 |
| `cpapp-b.bin` | `5c8528f46f5a8aa49179089c82b5944ef6b260a859e30a93f53a9b46c4208b12` | ARM Thumb CP 镜像 |
| `tomco.txt` | `2a51c104695392454a930da8368fac53101efd10b862f9538eb6994b7da948ef` | A1 CP 静态配置，含 `console_enable=N`、`uart_baudrate=460800` |

当前文件集中没有单独的 BootROM 镜像。`loader_nor.bin` 是后续启动加载器，不能冒充 BootROM。

## 1. loader_nor 的入口与权限初始化

入口 `0xFE049000` 先设置 `VBAR_EL1/EL2/EL3`、`SCR_EL3`、`HCR_EL2`、`CPACR_EL1`、`MAIR_EL3`、`TCR_EL3`，清理各级 `SCTLR`，建立页表和异常向量，然后进入内部初始化。这个证据说明它是高权限启动代码，但没有显示秘密串口协议或外部认证绕过。

## 2. 明文搜索结果

在 `loader_nor.bin` 与 `loader_spi.bin` 中，未找到 `UART`、`UART_DEBUG`、`SERIAL`、`FASTBOOT`、`BOOTROM`、`JTAG`、`SWD` 等明文。`loader_nor.bin` 唯一的 `usb` 命中是 `usbchrg` 配置字符串，不能解释成 USB 调试命令。

这只是字符串层面的负面证据；BootROM 可能不在文件中，协议也可能使用数字表或加密数据。

## 3. loader_nor 中确实存在的工程调试／硬件控制路径

以下内容是 A1 loader 自身的静态事实：

- `0xABCC–0xAC7C` 一段读取配置字段，构造 `0xF101D008 + (字段 << 12)`，写入位掩码，再读取状态并输出 `DEBUG_MODE_SW` 相关信息。对应日志字符串在文件偏移 `0x14386` 附近。
- `0xAC90–0xACD8` 访问 `0xF102E008` 区域，写入状态并读取位。
- 文件偏移 `0x14331` 有 `LDR:BOOT_EXT(SPI_BOOT_MODE)`；相关代码使用启动状态／配置字段，未证明任何外部 GPIO 电平可以直接劫持启动。
- 文件偏移 `0x135B0`、`0x135D8`、`0x13602` 有 `CP_ESCAPE@cetus`、`cpapp-b.bin@cetus`、`backup.bin@cetus` 等传输等待日志。
- 文件偏移 `0x14E21`、`0x14E96` 有 `SPI:Slave initialize`、`spi_handshake_gpio_slave_init`。
- 文件偏移 `0x13740`、`0x13771`、`0x13799` 有 `pcie_stm_comm_ready`。
- 字符串表含 `BKID_LDR_CFG_ADJUST_BOOTMODE`、`BKID_RAM_ADDR_DEBUGER_MODE`、`BKID_LDR_DEBUGER_MODE`、`BKID_LDR_CFG_EMERGENCY_DETECT`。

这些路径说明 loader 支持工程调试模式、SPI 从设备握手、GPIO 配置和 PCIe STM 启动传输；它们仍可能只服务于工厂／内部启动流程。

## 4. CP 侧调试命令表已经闭合到“注册”层

CP 中不是只有孤立字符串。三个命令表都采用 12 字节记录，记录包含类别／属性指针、命令名指针和 Thumb 函数地址；共同注册调用为 `0x5E7D9C`。

### UART 表

- 表地址：`0x108FC40`（CP 镜像相对 VA）。
- 注册点：`0x4EB968–0x4EB972` 构造 `0x108FC40` 并跳到 `0x5E7D9C`。
- 记录数：18，范围 `0xF87C40–0xF87D0C`。
- 命令名／函数示例：
  - `uart_dump` → `0x4EA795`
  - `pbl_uart_open` → `0x4EA869`
  - `pbl_uart_read_buffer` → `0x4EACB1`
  - `pbl_uart_txstart` → `0x4EAE05`
  - `pbl_uart_testmode_ena` → `0x4EB611`
  - `pbl_uart_test_read_buffer` → `0x4EB681`
  - `pbl_uart_test_write_buffer` → `0x4EB7C5`
  - `debug_memdump_enable` → `0x4EB8B9`

### GPIO 表

- 表地址：`0x108F434`。
- 注册点：`0x4E0CA4–0x4E0CAE` 构造表地址并跳到 `0x5E7D9C`。
- 记录数：23，范围 `0xF87434–0xF8753C`。
- 命令覆盖 `pbl_gpio_set_func`、`pbl_gpio_set_sel`、`pbl_gpio_get_sel`、`pbl_gpio_set_dir`、`pbl_gpio_get_dir`、`pbl_gpio_set_enable`、`pbl_gpio_set_data`、`pbl_gpio_get_data`、`pbl_gpio_set_oen`、`pbl_gpio_set_ien`、`pbl_gpio_set_drv`、上下拉、寄存器检查等。

### SPI／SPI-GPIO 表

- 表地址：`0x108F8C0`。
- 注册点：`0x4E72CC–0x4E72D6` 构造表地址并跳到 `0x5E7D9C`。
- 记录数：22，范围 `0xF878C0–0xF879BC`。
- 命令覆盖 `pbl_spi_open`、`pbl_spi_start`、`pbl_spi_read_data`、`pbl_spi_write_data`、`pbl_spi_set_gpio_hiz`、`pbl_spi_read_write_data`、`status_check` 以及若干 `pbl_gpio_*` 辅助项。

`0x5E7D9C` 本身只是把命令表保存到 CP 的全局注册数组；静态分析没有证明注册数组由 USB、Multi 接口、PCIe 或外部 UART 直接驱动。

## 5. A7M4 参考的边界

本地参考 `work/reference/a7m4_custom_firmware/README.md` 明确说 A7M4 需要自制线和 RP2040 启动桥，并且该项目不包含交互式 UART 控制台、UART 命令行或串口更新模式切换。它描述的是串行传输进入 fastboot 的链路。A7M4 的线序、电平、命令和 payload 不能用于推断 A1。

## 6. 结论分级

| 命题 | 结论 | 置信度 |
|---|---|---|
| A1 loader 存在工程调试／硬件控制路径 | 已确认 | 高 |
| A1 CP 存在 UART/GPIO/SPI 调试命令并注册到内部命令表 | 已确认 | 高 |
| 这些命令可由外部 USB 或 Multi 串口到达 | 未知 | 低／静态断点 |
| A1 存在 A7M4 同款串行启动模式 | 未知，不能类推 | 低 |
| 当前已发现可直接劫持的 BootROM 后门 | 未确认 | 低 |

最终结论：**需要物理逻辑分析仪抓取启动日志。**在取得 A1 主板原理图、可靠地识别复位、时钟和接口复用前，不能指定某根针脚、波特率、命令或启动时序。后续硬件工作应先被动记录上电到 `CP_ESCAPE`／SPI slave／PCIe STM ready 的时间关系，再对照 loader 的寄存器访问；不得先向未知接口发送数据。

## 7. 未来复活项目时所需的最小物理证据

1. 主板版本和接口复用图，确认 `spi_handshake_gpio_slave_init`、CP escape 和 PCIe STM 的实际物理链路。
2. 逻辑分析仪对复位、候选 SPI/GPIO 和候选串行线的被动启动记录；记录电平、时钟、方向和时间戳，不注入数据。
3. 能把记录与 loader 相对地址对应起来的启动日志或错误码，特别是 `SPI_BOOT_MODE`、`CP_ESCAPE`、`pcie_stm_comm_ready` 事件。
4. 若以后获得只读内存访问，优先核对 loader 运行时的 `0xF101D008 + (字段 << 12)`、`0xF102E008` 状态区，以及 CP 命令注册表 `0x108FC40`；这些是模块／运行时地址候选，不是已确认的物理地址。
5. 仍需单独找到 BootROM 本体或芯片厂商启动文档；当前 loader 文件不能替代这一证据。
