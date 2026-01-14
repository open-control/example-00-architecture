# Open Control - API Cheatsheet

## Input Bindings

```cpp
// ─────────────────────────────────────────────────────────────
// Buttons
// ─────────────────────────────────────────────────────────────
onButton(id).press().then([this]{});
onButton(id).release().then([this]{});
onButton(id).longPress(500).then([this]{});           // ms threshold
onButton(id).doubleTap(300).then([this]{});           // ms window

// Conditional
onButton(id).press().when([this]{ return isArmed; }).then([this]{});

// ─────────────────────────────────────────────────────────────
// Encoders
// ─────────────────────────────────────────────────────────────
onEncoder(id).turn().then([this](float value){});     // value depends on mode

// Modes (via EncoderAPI)
encoders().setMode(id, EncoderMode::NORMALIZED);  // [0.0, 1.0] bounded
encoders().setMode(id, EncoderMode::RAW);         // accumulated ticks
encoders().setMode(id, EncoderMode::RELATIVE);    // delta per detent

encoders().setBounds(id, 0.0f, 100.0f);           // custom range
encoders().setDelta(id, 0.01f);                   // step for RELATIVE
```

## Context Lifecycle

```cpp
class MyContext : public oc::context::IContext {
public:
    // Required APIs (validated at registration)
    static constexpr oc::context::Requirements REQUIRES{
        .button = true,
        .encoder = true,
        .midi = true,
        .frames = false
    };

    bool initialize() override {
        // Setup bindings, load state
        return true;  // false = init failed
    }

    void update() override {
        // Per-frame logic (called at APP_HZ)
    }

    void cleanup() override {
        // Teardown before context switch
    }

    const char* getName() const override { return "My"; }
};

// Registration
app.registerContext<MyContext>(ContextID::MAIN, "Main");

// Switching (from within a context)
switchTo(ContextID::CONFIG);
switchToDefault();
```

## Reactive State

```cpp
// ─────────────────────────────────────────────────────────────
// Signals
// ─────────────────────────────────────────────────────────────
oc::state::Signal<float> volume{0.5f};
oc::state::SignalString trackName;

volume.set(0.75f);
float v = volume.get();

// Subscribe (returns Subscription - store it!)
auto sub = volume.subscribe([this](const float& v){
    updateUI(v);
});

// Bind multiple signals (fluent API)
std::vector<oc::state::Subscription> subs_;

oc::state::bind(subs_)
    .on(state.name,    [this](const char* n) { nameLabel_.setText(n); })
    .on(state.volume,  [this](float v) { knob_.setValue(v); });
```

## Error Handling

```cpp
// ─────────────────────────────────────────────────────────────
// Result<T>
// ─────────────────────────────────────────────────────────────
oc::core::Result<void> init() {
    if (!hw.begin()) {
        return oc::core::Result<void>::err({oc::core::ErrorCode::HARDWARE_INIT_FAILED, "SPI"});
    }
    return oc::core::Result<void>::ok();
}

// Usage
if (auto r = device.init(); !r) {
    OC_LOG_ERROR("Failed: {}", r.error().detail);
}

// Chain
if (auto r = initA(); !r) return r;
if (auto r = initB(); !r) return r;
return oc::core::Result<void>::ok();

// With value
oc::core::Result<int> compute() { return oc::core::Result<int>::ok(42); }
int val = result.valueOr(0);  // default if error
```

## Logging

```cpp
// Configuration (once at boot)
#include <oc/log/Log.hpp>
oc::log::setOutput(myOutput);  // HAL provides Output struct

// Usage (requires OC_LOG defined)
OC_LOG_DEBUG("x={}", x);    // [12ms] DEBUG: x=42      (cyan)
OC_LOG_INFO("Ready");       // [15ms] INFO: Ready      (green)
OC_LOG_WARN("Low: {}%", p); // [20ms] WARN: Low: 5%    (yellow)
OC_LOG_ERROR("Fail");       // [25ms] ERROR: Fail      (red)

{ OC_LOG_SCOPE("init"); code(); }  // [init] 45ms
```

## Persistence

```cpp
// ─────────────────────────────────────────────────────────────
// Settings<T>
// ─────────────────────────────────────────────────────────────
struct MySettings {
    uint8_t channel = 1;
    float volume = 0.5f;
    char name[32] = "Default";
};

oc::hal::teensy::EEPROMBackend storage;
oc::state::Settings<MySettings> settings(storage, 0x0000, 1);  // addr, version

settings.load();
settings.data().volume;                             // read
settings.modify([](auto& s){ s.volume = 0.8f; });   // write
settings.save();
settings.factoryReset();
```

## AppBuilder (Teensy)

```cpp
#include <oc/hal/teensy/Teensy.hpp>

std::optional<oc::app::OpenControlApp> app;

void setup() {
    app = oc::hal::teensy::AppBuilder()
        .midi()
        .frames()  // USB Serial with COBS framing
        .encoders(encoderDefs)
        .buttons(buttonDefs)
        .inputConfig({
            .longPressMs = 500,
            .doubleTapWindowMs = 300,
            .debounceMs = 5
        });

    app->registerContext<MyContext>(ContextID::MAIN, "Main");

    if (auto r = app->begin(); !r) { /* error */ }
}

void loop() { app->update(); }
```

## HAL Interfaces (for porting)

```cpp
// Implement these interfaces in oc::hal namespace
class MyButtons : public oc::hal::IButtonController {
    oc::core::Result<void> init() override;
    void update(uint32_t currentTimeMs) override;
    bool isPressed(oc::hal::ButtonID id) const override;
    void setCallback(ButtonCallback cb) override;
};

class MyEncoders : public oc::hal::IEncoderController {
    oc::core::Result<void> init() override;
    void update() override;
    float getPosition(oc::hal::EncoderID id) const override;
    void setPosition(oc::hal::EncoderID id, float value) override;
    void setMode(oc::hal::EncoderID id, oc::hal::EncoderMode mode) override;
    void setBounds(oc::hal::EncoderID id, float min, float max) override;
    void setDelta(oc::hal::EncoderID id, float delta) override;
    void setCallback(EncoderCallback cb) override;
};

class MyMidi : public oc::hal::IMidiTransport {
    oc::core::Result<void> init() override;
    void update() override;
    void sendCC(uint8_t channel, uint8_t cc, uint8_t value) override;
    void sendNoteOn(uint8_t channel, uint8_t note, uint8_t velocity) override;
    void sendNoteOff(uint8_t channel, uint8_t note, uint8_t velocity) override;
    void sendSysEx(const uint8_t* data, size_t length) override;
    void setOnCC(CCCallback cb) override;
    void setOnNoteOn(NoteCallback cb) override;
    void setOnNoteOff(NoteCallback cb) override;
};

// Configure logging output
oc::log::Output myOutput{
    .printChar = [](char c) { MySerial.print(c); },
    .printStr = [](const char* s) { MySerial.print(s); },
    .printInt32 = [](int32_t v) { MySerial.print(v); },
    .printUint32 = [](uint32_t v) { MySerial.print(v); },
    .printFloat = [](float v) { MySerial.print(v, 2); },
    .printBool = [](bool v) { MySerial.print(v ? "true" : "false"); },
    .getTimeMs = []() { return myMillis(); }
};
oc::log::setOutput(myOutput);
```
