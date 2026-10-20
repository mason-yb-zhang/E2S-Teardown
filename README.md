# 喵喵机 E2S 拆解与开发档案

> 全网首份 Paperang E2S 电子单词卡实机拆解料单与刷机指南。
> 拆解日期：2026-10-20 · 固件分析状态：待备份

---

## 1. 设备信息

| 项目 | 内容 |
|---|---|
| 产品名称 | 电子单词卡 E2S（喵喵机 / Paperang） |
| SN | `E2S0B029003474` |
| 主板板号 | `E2S_MB_V3` |
| 整机尺寸 | 86 × 39 × 13.6 mm，约 33g |
| 电池 | 450mAh（宣称典型续航 150 天） |
| 屏幕 | 2.13 英寸墨水屏，250 × 122 |
| 通讯 | 蓝牙 / Wi-Fi（**Type-C 口疑似仅充电，待验证**） |
| 发音 | 支持（D 类功放 + 扬声器） |

![产品参数](images/product_specs.jpg)

---

## 2. 主芯片料单（实拍丝印确认）

| 位号 | 芯片 | 说明 |
|---|---|---|
| U20 | **ESP32-S3R2**（丝印 `R2MNP017000 / 262022`） | 主控。QFN56，双核 Xtensa LX7，封装内 2MB PSRAM；参数表"内存 2.5MB" = 512KB SRAM + 2MB PSRAM |
| U13（背面） | **GigaDevice GD25Q16E**（`25Q16ESIG`） | SPI NOR Flash，**16Mbit = 2MB**，存放系统固件。空间极限 → 官方固件无 OTA 双分区冗余 |
| U13（主控侧） | **MKDV1GCL-AB**（米客方德 MKSEMI） | **SD NAND，1Gbit = 128MB**，LGA-8 封装，走 SD 协议。词书与用户数据全在此 |
| U14 | **NSIWAY NS4150C** | 3W 单声道 D 类音频功放，驱动扬声器 |
| — | `40.0` 晶振 | 40MHz 主晶振 |
| — | `32.768K` 晶振 | RTC 低功耗时钟 |
| U12 | 丝印 `VΛ / EZF / H44`（QFN） | 疑为电源/保护器件（Wayon 类），非主控 |
| — | `2301`（SOT-23） | CJ2301 P-MOS，疑为 NAND/外设电源开关 |

### 存储拓扑

| 器件 | 容量 | 角色 | 读取方式 |
|---|---|---|---|
| GD25Q16E | 2MB | 系统固件 | `esptool.py read_flash 0 0x200000` |
| MKDV1GCL-AB | 128MB | 词书 / 用户数据 | 自研固件挂载 SDMMC 后流式导出（LGA-8 无法夹具离线读） |

### 已排除项

- **不是 TLSR8250**（E1 的方案）：官方"双核处理器"与 TLSR8250 单核 TC32 矛盾；且 TLSR8250 无 Wi-Fi
- E1 的刷机工具链（SWS / FT232 / 网页烧录器）对 E2S **完全不适用**，E2S 走 ESP32 标准 esptool 路线

---

## 3. 主板照片

### 主控面
![主控面](images/pcb_mcu_side.jpg)

ESP32-S3R2（左下）、MKDV1GCL-AB SD NAND（中）、40MHz 主晶振、32.768K RTC 晶振、2.4G 天线及匹配网络（左上）、屏幕 FPC 座（右侧）。

![主控面全景](images/pcb_mcu_full.jpg)

### 电源/音频面
![电源音频面](images/pcb_power_audio_side.jpg)

GD25Q16E（上）、NS4150C 音频功放（中）、U12 电源器件（下）。

### 背面（Type-C 口 / 测试点区）
![背面](images/pcb_back_side.jpg)

SN 贴纸 `SN:E2S0B029003474`、规则排列的金色 pogo-pad 阵列（MARK3 附近）、按键 FPC 座、白色连接器（电池/扬声器）。

![背面全景](images/pcb_back_full.jpg)

板框丝印 `E2S_MB_V3`。

---

## 4. 刷机与固件备份

### 4.1 固件备份（最高优先级）

```bash
# 进入下载模式后：
esptool.py flash_id                          # 确认芯片与 Flash
esptool.py read_flash 0 0x200000 backup.bin  # 全量备份 2MB 固件
```

固件备份的三重价值：
1. **救砖保险** — 任何自研固件刷入前先留原厂镜像，可 `write_flash` 完整还原
2. **引脚映射** — ESP-IDF 固件中 GPIO/SPI/SDMMC/按键配置明文可反推，等价于大半张原理图
3. **协议情报** — App API 域名、OTA 地址、词书格式均可从字符串与分区表分析

风险项：若厂家启用 Flash Encryption，读出为密文（消费级概率低，待验证）。

### 4.2 进入下载模式

ESP32-S3 在复位释放瞬间采样 IO0：**低电平 = 下载模式**。整机无外露 BOOT 键，按序尝试：

1. **自动**：直接运行 esptool，S3 原生 USB-Serial-JTAG 常能自动复位进入
2. **按键**：按住某实体键插线，枚举出 `303a:1001` 即中
3. **测试点短接**（兜底）：短接 IO0 → GND 同时上电，保持 1~2 秒松开

### 4.3 两条物理链路

**Type-C 口状态：待验证。** 参数表通讯方式仅"蓝牙/Wi-Fi"，口疑似纯充电（D+/D- 悬空）。验证法：万用表蜂鸣档从 Type-C A6/B6（D+）、A7/B7（D-）打连通，重点查口旁的 R84/R85 串阻网络。

- **若口有数据线**：Type-C 直插 + IO0 短接时序即可
- **若口纯充电**（大概率）：走 UART 下载，用 FT232：

```
FT232 TX  → U0RXD (GPIO44)
FT232 RX  → U0TXD (GPIO43)
FT232 GND → GND
FT232 DTR → IO0    （esptool 自动完成 EN/IO0 时序）
FT232 RTS → EN
```

注意：此前"FTDI 不兼容刷机"的结论仅适用于 E1 的泰凌 SWS 位敲击协议；UART 下载是标准串口协议，FT232 完全适用。

### 4.4 测试点识别指南

板上 `TP1~TP35` + pogo-pad 阵列即产线烧录口。识别顺序：

1. **GND**：对 USB 口金属壳蜂鸣
2. **3V3**：开机后量电压
3. **U0TXD（GPIO43）**：上电瞬间输出 115200 8N1 ROM 启动日志；无逻辑分析仪时用万用表看"空闲高电平、上电跳变"特征
4. **IO0**：上电时按住某点接地，启动日志停在下载提示即为 IO0（识别与验证同一步）
5. **EN/RX**：对照 Espressif 官方 QFN56 封装引脚图数脚确认

### 4.5 词书导出（以软读硬）

LGA-8 封装无法使用 SOP8 夹具离线读取，正确路径：自研固件初始化 SDMMC 挂载 SD NAND → USB 串口流式导回电脑 → 同时获得官方词书格式真实样本。

---

## 5. 开源生态参照

E2S 无公开原理图（`E2S_MB_V3` 全网零结果）。功能同构的开源工程可替代：

| 项目 | 价值 |
|---|---|
| **Pi-EPD**（立创开源） | 2.7 寸墨水屏阅读器，功能含单词卡、Wi-Fi 传书，ESP32-S3 + ESP-IDF，带完整原理图 |
| **ESP32S3 墨水屏掌机**（立创开源） | 系统框图与本板同构：SDMMC、Type-C 烧录 GPIO19/20、UART0 调试 GPIO43/44 |
| **微雪 ESP32-S3-ePaper-1.54G** | 官方仓库含原理图 PDF，S3 最小系统 + 墨水屏 + SD 卡接法可直接参考 |
| **hrp518/Miaopaper** | E1（TLSR8250）电子书固件，功能对标（本档案的动机来源） |
| **atc1441/ATC_TLSR_Paper** | E1 固件底座，EPD 驱动与 BLE 协议源头 |
| **pvvx 工具链**（TlsrComSwireWriter / ATC_MiThermometer） | 仅适用 E1，E2S 不用 |
| **OpenEPaperLink** | 可刷墨水屏设备情报库（价签主控对照表） |

---

## 6. 待办清单

- [ ] 万用表验证 Type-C D+/D- 是否有网（判定纯充电口）
- [ ] 从 pogo/TP 阵列中识别 GND / U0TXD / U0RXD / IO0 / EN
- [ ] FT232 接线，`esptool.py flash_id` 打通
- [ ] `read_flash 0 0x200000` 全量备份原厂固件
- [ ] 固件分析：分区表 / GPIO 配置 / API 字符串 / 词书格式
- [ ] SD NAND 词书导出
- [ ] 自研固件方案（ESP-IDF 或 Arduino）：EPD 驱动 + SD 挂载 + BLE 传书
- [ ] 整理完整测试点对应表，回填本档案

---

## 7. 许可与说明

- 照片与料单：CC BY-SA 4.0，转载请署名
- 拆解风险自负：外壳卡扣 + 胶合，墨水屏排线脆弱；拆机可能影响保修
- 本档案仅供学习研究，词书内容版权归喵喵机/作业帮所有
