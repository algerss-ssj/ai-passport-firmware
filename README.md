![效果示意](https://github.com/algerss-ssj/ai-passport-firmware/releases/download/v3/preview-720x960.png)

> 配图为按固件实际绘制参数（15 列 × 16 行 / 16px / #00FF41 六档衰减）**离线渲染的效果示意**，非实机拍摄。

![社区展示图](https://github.com/algerss-ssj/ai-passport-firmware/releases/download/v3/poster-community.png)

---

# AI PASSPORT 数字瀑布固件 v3（小智 2.5.2 · ESP32-C3）

> 基于官方 [xiaozhi-esp32](https://github.com/78/xiaozhi-esp32) 2.5.2，板型 `folotoy/ai-passport`，
> 在 AI PASSPORT 小卡片机上实现了「黑底 + 绿色数字雨」的 Matrix 风格界面。
> 固件保留原厂 8MB 整片回滚能力，刷坏了可以一键回原厂。

- **构建环境**：ESP-IDF v6.1（RISC-V，`riscv32-esp-elf`）
- **固件版本**：2.5.2
- **构建设置**：`python3 scripts/build.py folotoy/ai-passport --name ai-passport`
- **发布日期**：2026-10-25
- **状态**：预览版（preview）—— UI 功能可用，细节仍在打磨，见文末「已知问题」

---

## 一、适用硬件

| 项目 | 参数 |
|---|---|
| 型号 | FoloToy **AI PASSPORT**（小卡片机） |
| 主控 | ESP32-C3（RISC-V 单核 160MHz，**无 PSRAM**） |
| Flash | 8MB（`dio` / 80MHz） |
| 屏幕 | ST7789P3 **240×320** IPS（SPI） |
| 音频 | ES8311 codec + I2S 全双工（喇叭 + 麦克风） |
| 电量计 | CW2017 |
| 按键 | 3 键（GPIO0 电阻梯：确定=对话、上=音量+、下=音量−） |
| 连接 | USB-C（ESP32-C3 原生 USB-Serial-JTAG，`303a:1001`） |

> ⚠️ 只适用于 AI PASSPORT。其它小智机型（面包板 C3、S3 屏板、悦江湖四代 s3cam 等）屏幕尺寸/引脚不同，**不要刷**。

---

## 二、下载

| 文件 | 大小 | MD5 | 说明 |
|---|---|---|---|
| `rain-v3-app.bin` | 2,481,184 B (2.37 MB) | `97480345d93793502b7cdd387a08cd7d` | 只含应用，**保留 WiFi 配网与激活信息**（推荐日常升级用） |
| `rain-v3-merged.bin` | 7,616,373 B (7.26 MB) | `bfe90dc70b3783eb65fd1222605c5afd` | 整片镜像（bootloader + 分区表 + 应用 + 字库资源），从 `0x0` 一次写完 |

两个文件都不含用户数据分区（`nvs`）内容，刷入后原来的配网/激活状态不会被带入新镜像——请按下面「刷法 A / 刷法 B」选择。

---

## 三、刷机

### 0. 准备

```bash
python -m pip install -U esptool
```

插上 USB-C 线，Windows 认成 `COM3`/`COM4`…，Linux 是 `/dev/ttyACM0`，macOS 是 `/dev/cu.usbmodem*`。
本机为原生 USB-JTAG，**一般不需要按住 BOOT**。

> 如果设备上没有 `nvs` 里的配网信息，刷完还需要在小智 App 里重新配网 + 激活（见第五节）。

### 刷法 A：只刷应用（保留配网，约 30 秒）

```bash
# Linux / macOS
python -m esptool --chip esp32c3 -p /dev/ttyACM0 -b 460800 \
  --before default-reset --after hard-reset \
  write-flash 0x20000 rain-v3-app.bin

# Windows
python -m esptool --chip esp32c3 -p COM3 -b 460800 ^
  --before default-reset --after hard-reset ^
  write-flash 0x20000 rain-v3-app.bin
```

看到 `Hash of data verified.` + `Hard resetting via RTS pin...` 就是刷好了。

### 刷法 B：整片刷写（约 2~3 分钟，救砖/换机用）

```bash
python -m esptool --chip esp32c3 -p /dev/ttyACM0 -b 460800 \
  --before default-reset --after hard-reset \
  write-flash 0x0 rain-v3-merged.bin
```

整片镜像会重写 bootloader 与分区表，`nvs`（WiFi 配置）在 `0x9000~0xCFFF`，**刷整片后需要重新配网**。

> 不用整片镜像、想手工按段刷的话，地址是：
> `0x0` bootloader · `0x8000` 分区表 · `0xd000` ota_data · `0x20000` app · `0x600000` 资源(assets)

---

## 四、回滚到原厂固件

刷机前请务必备份整片 Flash（本仓库作者备份的 md5 为 `4e66ecf0801d391a19689012bb6b69cf`，8,388,608 B）。
回滚就是把它从 `0x0` 写回去：

```bash
python -m esptool --chip esp32c3 -p /dev/ttyACM0 -b 460800 \
  --before default-reset --after hard-reset \
  write-flash 0x0 original-8m.bin
```

自己备份的命令（刷机前先做）：

```bash
python -m esptool --chip esp32c3 -p /dev/ttyACM0 -b 460800 \
  read-flash 0x0 0x800000 original-8m.bin
```

---

## 五、界面说明

- **黑底绿色字符雨**：100ms 跳一格，每列速度/长度随机，字符为日文片假名 + 数字 + 少量汉字，头部满亮绿 `#00FF41`，往下逐级变暗形成长尾。
- **满屏 16 行**（每格 16px，纵向覆盖 256px，顶部 30px 留给状态栏、底部 34px 留给字幕条）。
- **状态栏 / 聊天字幕**沿用原版布局，文字叠在雨上。
- 语音交互、OTA、配网、MCP 工具全部保持官方 2.5.2 行为，**只改了显示层**。

首次使用：在小智 App 里配网 → 绑定设备 → 激活，然后喊唤醒词即可对话。

---

## 六、已知问题（预览版）

| # | 问题 | 说明 |
|---|---|---|
| 1 | 只有 **15 列**（目标 16 列） | 240px ÷ 16px = 15 列；要 16 列需把格子改成 15px 宽，会切掉字形 1px |
| 2 | 雨只在 **y=30~286** | 状态栏与字幕条区域目前留白，尚未做「满屏 320px」 |
| 3 | 字幕/状态栏**还不是绿色等宽字体** | 仍用主题字体（黑字气泡） |
| 4 | 说话时雨**不会变暗** | 目标是说话时压低雨亮度、突出字幕 |
| 5 | 个别机器首次刷入可能白屏 | 白屏=屏幕只显示了初始化时的白底，此时**断电重插**；若持续白屏请回滚（第四节）并反馈 |

---

## 七、技术记录：为什么这版不用「离屏画布」

C3 只有约 130KB 可用内部 RAM，实测数字很死板：

| 方案 | 缓冲占用 | 结果 |
|---|---|---|
| RGB565 画布 240×256 | 122,880 B | LVGL 内部 `lv_draw_add_task` 申请失败 → `taskLVGL` 卡死 → 看门狗重启 |
| I1 单色画布 240×256 | 7,684 B | 内存够，但渲染链路异常（且主题刷新后字体野指针导致崩溃） |
| L8 灰度画布 240×256 | 61,440 B | TLS 握手时 `alloc(4770 bytes) failed` → 掉线 |
| **本版：自绘（无缓冲）** | **0 B** | 在 `LV_EVENT_DRAW_MAIN` 里直接往屏幕图层 `lv_draw_label`，内存回到 129KB 空闲 |

关键结论：**TLS 握手的峰值要吃掉约 80KB**，任何 ≥40KB 的全屏画布都会把余量榨干。所以满屏雨只能「直接画」，不能「先画到缓冲再贴图」。

实测内存（本版）：启动空闲 **129KB** → 激活后 **65KB** → 运行稳定态 **76KB**，TLS 最低 **46.9KB**，无 panic、无重启。

---

## 八、构建 / 复现

```bash
git clone https://github.com/78/xiaozhi-esp32.git
cd xiaozhi-esp32
# 应用本版补丁：main/display/lcd_display.{h,cc}（瀑布实现）
source /path/to/esp-idf/export.sh     # 需要 ESP-IDF v6.1
unset http_proxy https_proxy all_proxy HTTP_PROXY HTTPS_PROXY ALL_PROXY   # 组件管理器不吃 SOCKS 代理
python3 scripts/build.py folotoy/ai-passport --name ai-passport
# 产物：build/xiaozhi.bin、build/merged-binary.bin
```

---

## 九、许可与致谢

- 固件基于 [78/xiaozhi-esp32](https://github.com/78/xiaozhi-esp32)，遵循其原仓库许可。
- 板型支持来自官方 `boards/folotoy/ai-passport`。
- 界面改动（数字瀑布）为本版新增。
