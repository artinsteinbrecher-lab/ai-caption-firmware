# AI字幕辅助设备 — 固件

基于 [xiaozhi-esp32](https://github.com/78/xiaozhi-esp32) 完整源码修改的实时语音转字幕设备固件，专为听障人士设计的辅助工具。

## 项目简介

本项目在原版小智 ESP32 固件基础上进行裁剪和修改，去除了 AI 对话、LLM、TTS 等功能，只保留：麦克风采集语音 → WebSocket 上传后端 → 后端调用阿里云百炼实时 ASR → 流式返回识别文本 → 屏幕显示字幕。

## 适用硬件

- **立创实战派 ESP32-S3**（lichuang-dev 板型）— 嘉立创商城售卖的成品板子，内置 SPI LCD 屏（ST7789, 320×240）、MEMS 麦克风、扬声器，开箱即用
- 也可用于立创实战派 ESP32-C3（lichuang-c3-dev），但 C3 无 PSRAM、内存紧张，建议优先选 S3

## 修改文件清单（共 4 个文件）

| 文件 | 修改内容 |
|------|----------|
| `main/application.h` | 新增 `caption_mode_` 标志、重连定时器成员、`StartReconnectTimer()` 等方法声明 |
| `main/application.cc` | 断线指数退避自动重连（1s→2s→4s→8s→16s→30s）、caption 模式角色控制、idle 状态不清空字幕、静音超时不弹唤醒词 |
| `main/display/lcd_display.cc` | 默认样式分支（`#else`）添加 caption 全屏居中换行显示；WeChat 样式分支添加标签复用避免每帧重建 |
| `main/display/oled_display.cc` | 添加 else 分支恢复 `content_left_` 可见性 |

## 关键修复

### P0 修复（影响设备可用性）

- **断线自动重连**：网络断开后设备显示"重连中..."，自动按指数退避策略重连（1s→2s→4s→8s→16s→30s），最多重试 10 次，重连成功后字幕继续更新
- **静音超时不泄漏连接**：caption 模式下静音超时直接关闭连接，不触发 AI 对话流程
- **LVGL 对象复用**：避免每次 ASR 中间结果都 `lv_obj_clean` 重建全部 LVGL 对象，改为复用已有标签直接更新文本

### P1 修复（影响稳定性）

- **默认样式 caption 显示**：立创实战派 S3 使用 DEFAULT 消息样式，不启用 `CONFIG_USE_WECHAT_MESSAGE_STYLE`，原版代码的 caption 处理只在 WeChat 分支中，在目标硬件上完全不生效。本次修复在 `#else` 默认分支中添加了完整的 caption 显示逻辑
- **OLED 恢复显示**：caption 消息结束后恢复 `content_left_` 的可见性和布局

## 编译方法

1. 安装 ESP-IDF v5.4 或以上版本
2. 设置目标芯片：`idf.py set-target esp32s3`
3. 选择板型配置：`idf.py menuconfig` → XiaZhi Board → 选择 `lichuang-dev`
4. 编译：`idf.py -DBOARD=lichuang-dev build`
5. 烧录：`idf.py -DBOARD=lichuang-dev flash`

## 配套后端

后端代码请见：[ai-caption-server](https://github.com/artinsteinbrecher-lab/ai-caption-server)

## 使用文档

详细使用手册（含服务端搭建、固件烧录、故障排查）请见：[ai-caption](https://github.com/artinsteinbrecher-lab/ai-caption)

## 致谢

基于以下开源项目修改：
- [78/xiaozhi-esp32](https://github.com/78/xiaozhi-esp32) — 小智 ESP32 原版固件
