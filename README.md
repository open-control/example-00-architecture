# Open Control Framework

> Build MIDI controllers that work everywhere.

## Why?

| | |
|---|---|
| **Unopinionated** | Your hardware, your architecture |
| **Portable** | Write once, run on Teensy / ESP32 / STM32 / Desktop |
| **Progressive** | Start with 10 lines, scale to full UI |

## The Stack

```
┌─────────────────────────────────────┐
│         Your Application            │  ← Contexts, Views, Logic
├─────────────────────────────────────┤
│    Framework (platform-agnostic)    │  ← Input bindings, State, Events
├─────────────────────────────────────┤
│      HAL (Teensy/ESP32/...)         │  ← Hardware adapters
└─────────────────────────────────────┘
```

## Show Me The Code

```cpp
// Fluent input bindings - works on ANY platform with a HAL
onButton(BTN_PLAY).press().then([this]{
    midi().sendCC(0, 20, 127);
});

onButton(BTN_SETTINGS).longPress(500).then([this]{
    switchTo(ContextID::SETTINGS);
});

onEncoder(ENC_VOLUME).turn().then([this](float v){
    volume.set(v);  // Reactive state → auto UI update
});
```

## What You Get

- **Input System**: Buttons (press/release/long/double), Encoders (normalized/raw/relative)
- **Context System**: App modes with lifecycle (init → update → cleanup)
- **Reactive State**: `Signal<T>` with auto-subscriptions
- **Error Handling**: `Result<T>` for safe init chains
- **Logging**: `OC_LOG_INFO("value: {}", x)` with colors & timestamps
- **Persistence**: `Settings<T>` with CRC validation

## Get Started

| Example | What You'll Learn |
|---------|-------------------|
| [01-midi-output](../example-teensy41-01-midi-output/) | Minimal MIDI (no framework) |
| [02-encoders](../example-teensy41-02-encoders/) | Rotary encoders → MIDI CC |
| [03-buttons](../example-teensy41-03-buttons/) | Full framework with Contexts |
| [lvgl](../example-teensy41-lvgl/) | Complete UI application |

## Porting to New Hardware

Implement these interfaces and you're done:

```cpp
class MyButtonController : public oc::interface::IButton { ... };
class MyEncoderController : public oc::interface::IEncoder { ... };
class MyMidiTransport : public oc::interface::IMidi { ... };

// Configure log output (call once at boot)
oc::log::setOutput(myLogOutput);

// Provide time function to AppBuilder
builder.timeProvider([]{ return myMillis(); });
```

See [CHEATSHEET.md](./CHEATSHEET.md) for all API patterns.
