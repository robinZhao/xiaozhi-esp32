# 情绪监听 + 主动安慰功能 (Emotion Monitor) 实现计划

## Context

在设备空闲状态下，使能监听周围人的情绪。通过 SER（语音情绪识别）对原始音频做推理，检测到消极情绪时，服务器自动触发 LLM 生成安慰话术并通过 TTS 播放——**设备直接利用已有的 kDeviceStateSpeaking 状态播放 TTS，播放完自动回到 MONITORING 继续监听**。如果听到唤醒词，立即退出情绪监听，进入正常对话模式。

**核心约束：**
- ESP32 算力有限，情绪分析需依赖服务器端完成
- 唤醒词检测优先级始终最高，不能被情绪监听阻塞
- 功能需通过 Kconfig 开关控制，默认关闭
- 需兼顾功耗管理
- 检测坏情绪时直接利用现有 Speaking 状态回复，不引入新状态

---

## 架构设计

### 不新增 DeviceState，而是在 Idle 状态内增加子模式

通过新类 `DeviceEmotionMonitor` 封装所有情绪监听逻辑，在 `kDeviceStateIdle` 状态内运行。这样：
- 不需要修改 `device_state_machine.cc` 的转换表
- 唤醒词从 Idle → Connecting 的流程保持不变
- `HandleWakeWordDetectedEvent` 只需在最前面调用 `emotion_monitor_.OnWakeWordDetected()` 即可确保优先级

### DeviceEmotionMonitor 状态机

```
DISABLED ──(enable)──> IDLE_WAITING ──(grace period 到期)──> STARTING ──(channel opened)──> MONITORING
                            ^                                                                    |
                            |               (服务器 TTS 播放完)                                     | (wake word detected)
                            |                    │                                                |
                            v                    v                                                v
                       COOLDOWN ←──(服务器触发 LLM 安慰)── SPEAKING_COMFORT ←───────────────────+── (退出 idle 状态)
```

- **IDLE_WAITING**: 进入 idle 后等待可配置的静默期 (默认 30s)，避免对话刚结束就立刻开始监听
- **STARTING**: 打开音频通道，准备开始监听
- **MONITORING**: 音频持续发送到服务器，服务器做 SER 情绪分析。如果检测到消极情绪，服务器自动触发 LLM 生成安慰话术 + TTS
- **SPEAKING_COMFORT**: 小智进入 Speaking 状态播放安慰话术（由服务器 TTS 驱动）。播放完自动回到 MONITORING
- **COOLDOWN**: 刚说完安慰话，冷却期内不再触发

### 音频管道共存

在 MONITORING 状态下，同时设置 `AS_EVENT_WAKE_WORD_RUNNING` 和 `AS_EVENT_AUDIO_PROCESSOR_RUNNING`。`AudioInputTask` 的现有逻辑 (audio_service.cc:269-281) 已支持同时向唤醒词和音频处理器发送数据，无需修改。

---

## 文件修改清单

### 1. 新建文件: `main/device_emotion_monitor.h`

声明 `DeviceEmotionMonitor` 类：
- 内部状态枚举：DISABLED, IDLE_WAITING, STARTING, MONITORING, COOLDOWN
- 公开方法：`Initialize()`, `SetEnabled()`, `OnEnterIdle()`, `OnExitIdle()`, `OnWakeWordDetected()`, `OnTick()`, `HandleServerResponse()`, `IsPreventingSleep()`
- 私有成员：Application 指针、状态计数器、配置参数（静默期、冷却期、最大提醒次数）、最后检测到的情绪

### 2. 新建文件: `main/device_emotion_monitor.cc`

实现 `DeviceEmotionMonitor` 的状态机：
- `OnEnterIdle()`: 从 DISABLED 转为 IDLE_WAITING，开始计时
- `OnTick()`: 驱动状态机转换
  - IDLE_WAITING + 到期 → STARTING → 调用 `OpenAudioChannel()`
  - STARTING + 通道已打开 → MONITORING → 调用 `EnableVoiceProcessing(true)`, `SendStartListening(kListeningModeEmotionMonitor)`
  - COOLDOWN + 到期 → MONITORING
- `HandleServerResponse()`: 解析 `{"type":"emotion_monitor","emotion":"sad","negative":true}`，如果消极且在冷却期外，由服务器自动触发 LLM 生成安慰话术并 TTS 播放
- `OnWakeWordDetected()`: 立即关闭音频通道、停止语音处理、重置状态为 IDLE_WAITING
- `OnExitIdle()`: 关闭通道，清理状态

### 3. 修改: `main/application.h`

- 新增 `#include "device_emotion_monitor.h"`
- 新增成员 `DeviceEmotionMonitor emotion_monitor_`
- 新增事件位 `#define MAIN_EVENT_EMOTION_MONITOR (1 << 13)`

### 4. 修改: `main/application.cc`

变更点：
- **Initialize()** (约 line 61): 调用 `emotion_monitor_.Initialize(this)`
- **Run() 事件循环** (约 line 180): 在 ALL_EVENTS 中加入 `MAIN_EVENT_EMOTION_MONITOR`，添加对应的 `if (bits & MAIN_EVENT_EMOTION_MONITOR)` 处理调用 `emotion_monitor_.OnTick()`
- **HandleStateChangedEvent()** (约 line 860):
  - `kDeviceStateIdle` case: 在现有逻辑末尾添加 `emotion_monitor_.OnEnterIdle()`
  - 所有非 idle case (Connecting, Listening, Speaking, Activating, WifiConfiguring 等): 在 case 开头添加 `emotion_monitor_.OnExitIdle()`
- **HandleWakeWordDetectedEvent()** (line 780): 在函数最开头添加 `emotion_monitor_.OnWakeWordDetected()`
- **OnIncomingJson** (约 line 521): 在消息类型解析中添加 `emotion_monitor` 类型：
  ```cpp
  } else if (strcmp(type->valuestring, "emotion_monitor") == 0) {
      emotion_monitor_.HandleServerResponse(root);
  }
  ```
- **CanEnterSleepMode()** (line 1057): 添加 `emotion_monitor_.IsPreventingSleep()` 检查
- **CLOCK_TICK 处理** (约 line 248): 添加 `emotion_monitor_.OnTick()` 调用
- **MAIN_EVENT_SEND_AUDIO** (约 line 220): 在情绪监听模式下也需要发送音频（当前只有在 `kDeviceStateListening`/`kDeviceStateSpeaking` 时音频通道才会打开）。需确保 MONITORING 状态下音频也能正常发送。

### 5. 修改: `main/protocols/protocol.h`

- 在 `ListeningMode` 枚举中添加 `kListeningModeEmotionMonitor`

### 6. 修改: `main/protocols/protocol.cc`

- 在 `SendStartListening()` 中添加 `emotion_monitor` 模式映射

### 7. 修改: `main/protocols/websocket_protocol.cc`

- 在 `GetHelloMessage()` 的 features 对象中添加 `emotion_monitor` 标志 (受 `#ifdef CONFIG_ENABLE_EMOTION_MONITOR` 保护)

### 8. 修改: `main/protocols/mqtt_protocol.cc`

- 同上，在 `GetHelloMessage()` 的 features 中添加 `emotion_monitor` 标志

### 9. 修改: `main/Kconfig.projbuild`

在 `menu "Xiaozhi Assistant"` 内（`endmenu` 之前）添加：

```kconfig
config ENABLE_EMOTION_MONITOR
    bool "Enable Emotion Monitoring in Idle State"
    default n
    help
        When enabled, the device will monitor ambient conversations for
        negative emotional content during idle periods.

config EMOTION_MONITOR_IDLE_WAIT_SECONDS
    int "Idle Wait Before Monitoring (seconds)"
    default 30
    range 5 300
    depends on ENABLE_EMOTION_MONITOR

config EMOTION_MONITOR_COOLDOWN_SECONDS
    int "Reminder Cooldown (seconds)"
    default 120
    range 30 600
    depends on ENABLE_EMOTION_MONITOR

config EMOTION_MONITOR_MAX_REMINDERS
    int "Max Reminders Per Session"
    default 5
    range 1 20
    depends on ENABLE_EMOTION_MONITOR
```

### 10. 修改: `main/CMakeLists.txt`

- 在源文件列表中添加 `device_emotion_monitor.cc`

---

## 唤醒词优先级保证

1. **音频管道无冲突**: MONITORING 状态下音频同时送给唤醒词和音频处理器，唤醒词检测不受影响
2. **事件处理优先**: `HandleWakeWordDetectedEvent` 第一行即调用 `OnWakeWordDetected()`，立即关闭情绪监听通道
3. **状态机兜底**: 任何离开 Idle 的状态转换都会触发 `OnExitIdle()`

---

## 服务器端情绪分析 (SER)

### 为什么用 SER 而不是 ASR + 文本分析

ASR 转文字会丢失语调、语速、音高、能量等副语言特征，这些恰恰是情绪识别的关键信号。应该直接用 **语音情绪识别 (Speech Emotion Recognition, SER)** 对原始音频做推理。

### 服务器端 SER 管线

```
设备音频流 (Opus, 16kHz, mono)
        │
        ▼
   Opus 解码 → 16kHz PCM
        │
        ▼
   音频缓冲 (滑动窗口, 2-5秒片段, 步长可配置)
        │
        ▼
   特征提取 (MFCC / 语谱图 / 基频 / 能量 / 语速)
        │
        ▼
   SER 模型推理
   ├── 经典方案: CNN + LSTM/Transformer 对语谱图分类
   └── 前沿方案: wav2vec 2.0 / HuBERT / emotion2vec 预训练模型微调
        │
        ▼
   情绪分类 + 置信度
   {emotion: "sad", negative: true, confidence: 0.87}
        │
        ▼
   滑动平均去抖 (避免单帧抖动)
        │
        ▼
   → 服务器检测到消极情绪 → 自动触发 LLM 生成安慰话术
        │
        ▼
   → 服务器将安慰话术 TTS 编码为音频流
        │
        ▼
   → 返回设备 {"type":"emotion_monitor","emotion":"sad","negative":true,"confidence":0.87}
```

### 安慰话术触发流程

```
设备 MONITORING 状态（通道保持）
    → 服务器 SER 检测到消极情绪
    → 服务器 LLM 生成安慰话术（如 "你看起来心情不太好，想和我聊聊吗？"）
    → 服务器 TTS 编码为 Opus 音频流
    → 设备收到 tts:start → 进入 kDeviceStateSpeaking
    → 设备播放 TTS 音频
    → 设备收到 tts:stop → 进入 kDeviceStateListening（对话模式，等待用户回应）
    → 用户说话 → 正常 ASR 对话
    → 对话结束 → kDeviceStateIdle
    → emotion_monitor_ 重置为 IDLE_WAITING → 静默期 → MONITORING
```

### 和对话模式的区别

| | 对话模式 | 情绪监听模式 |
|---|---|---|
| 触发方式 | 唤醒词触发 | 空闲自动开启 |
| 音频处理 | ASR → LLM → TTS | SER 模型 |
| 服务端响应 | TTS 音频流 + LLM 文本 | SER 结果 + 可能触发 LLM 安慰 TTS |
| 设备是否回复语音 | 是 | 是（自动进入 Speaking 播放安慰语） |
| 播放完行为 | 回到 Listening（对话模式） | 回到 Listening（对话模式） |
| 推理成本 | 高（LLM） | 低（SER）或中（SER+LLM） |
| start-listening mode | auto/manual/realtime | emotion_monitor |

---

## 验证步骤

1. **编译验证**: `idf.py build`，确认 `CONFIG_ENABLE_EMOTION_MONITOR=n` 时编译通过且行为不变
2. **唤醒词优先级测试**: 情绪监听进行中 → 说唤醒词 → 设备应立即停止监听，进入 Connecting → Listening 正常对话
3. **静默期测试**: 对话结束后设备进入 idle → 应在静默期内保持纯唤醒词检测，不打开音频通道
4. **提醒冷却测试**: 连续触发消极情绪 → 第二次应在冷却期外才触发安慰
5. **sleep 防护测试**: MONITORING 状态下 `CanEnterSleepMode()` 应返回 false
6. **协议验证**: hello 消息包含 `emotion_monitor: true` feature flag；start-listening 发送 `mode: "emotion_monitor"`
