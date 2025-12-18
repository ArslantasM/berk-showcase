# iAts - BERK ile Yeniden Tasarım

##  Proje Özeti

Bu döküman, [iAts (Arduino Antenna Tracking System)](https://github.com/akari-tun/iAts) açık kaynak projesinin BERK programlama dili ile nasıl modernize edilebileceğini ve iyileştirilebileceğini detaylı olarak açıklar.

**Kaynak Proje:** https://github.com/akari-tun/iAts  
**Lisans:** GPL-3.0  
**Orijinal Geliştirici:** akari-tun  
**BERK Adaptasyonu:** BERK Geliştirici Topluluğu

---

##  iAts Nedir?

iAts (intelligent Antenna tracking System), FPV (First Person View) uçuşları için tasarlanmış otomatik anten takip sistemidir (AAT - Automatic Antenna Tracker).

### Temel İşlev

```
┌─────────────────────────────────────────────────────────────────┐
│                        iAts Sistem Mimarisi                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   UÇAK (TX)                              YER İSTASYONU (RX)     │
│   ┌──────────┐                           ┌──────────────────┐   │
│   │ GPS      │                           │ Anten Platformu  │   │
│   │ Modülü   │──▶ Konum Verisi ──▶       │ ┌────────────┐   │   │
│   └──────────┘    (Audio Ch.)            │ │ Pan Servo  │   │   │
│        │                                 │ │ (360°)     │   │   │
│        ▼                                 │ └────────────┘   │   │
│   ┌──────────┐                           │ ┌────────────┐   │   │
│   │ TCM3105  │    ─────────────────▶     │ │ Tilt Servo │   │   │
│   │ Modem    │    Video + Audio          │ │ (180°)     │   │   │
│   └──────────┘                           │ └────────────┘   │   │
│                                          │       │          │   │
│                                          │       ▼          │   │
│                                          │ ┌────────────┐   │   │
│                                          │ │ Yönlü Anten│   │   │
│                                          │ │ (23dB)     │   │   │
│                                          │ └────────────┘   │   │
│                                          └──────────────────┘   │
│                                                  │              │
│                                                  ▼              │
│                                          ┌──────────────────┐   │
│                                          │ Bluetooth → App  │   │
│                                          │ (HC05)           │   │
│                                          └──────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Mevcut Donanım Bileşenleri

| Bileşen | Model | Açıklama |
|---------|-------|----------|
| MCU | Arduino Pro Mini (ATmega328P) | 8-bit, 16MHz, 32KB Flash, 2KB RAM |
| IMU | MPU6050 | 6-axis (Gyro + Accelerometer) |
| Pusula | HMC5883L / QMC5883 | 3-axis magnetometer |
| Bluetooth | HC05 | Serial Bluetooth 2.0 |
| Modem | TCM3105 | Audio FSK modem |
| Pan Servo | MG995 (360°) | Sürekli dönüş |
| Tilt Servo | MG995 (180°) | Açısal konum |
| Ekran | SSD1306 OLED | 128x64 I2C |

---

##  Mevcut Sistemin Analizi

### Yazılım Mimarisi (C++)

```cpp
// Orijinal kod yapısı (basitleştirilmiş)
void loop() {
    // 1. Sensör okuma
    readCompass();
    readAccelerometer();
    
    // 2. Kalman filtre
    kalmanFilter();
    
    // 3. GPS verisi parse
    if (Serial.available()) {
        parseNMEA();
    }
    
    // 4. Açı hesaplama
    calculateBearing();
    calculateElevation();
    
    // 5. Servo kontrol
    updateServos();
    
    // 6. Bluetooth iletişim
    sendTelemetry();
}
```

### Tespit Edilen Sorunlar

| Sorun | Açıklama | Etki |
|-------|----------|------|
| **Nondeterministic Timing** | `loop()` polling modeli, sabit zamanlama garantisi yok | Servo jitter, sensör aliasing |
| **Kaynak Kısıtı** | 2KB RAM, karmaşık algoritmalar için yetersiz | Kalman matris boyutu sınırlı |
| **Float Emülasyonu** | ATmega328P'de HW FPU yok | Yavaş trigonometri (~100µs/op) |
| **Blocking I/O** | `Serial.read()` bloklayıcı | GPS parse sırasında servo donması |
| **Tip Güvenliği Yok** | C++ manual memory, no borrow checker | Buffer overflow riski |
| **Debug Zorluğu** | Serial.print debugging | Gerçek zamanlı analiz yok |

---

##  BERK ile İyileştirme Stratejisi

### Aşama 1: Platform Yükseltme

```
ATmega328P (8-bit, 16MHz, 2KB RAM)
              │
              ▼
ESP32-WROOM-32 (32-bit dual-core, 240MHz, 520KB RAM)
   veya
STM32F411 (32-bit, 100MHz, 128KB RAM, HW FPU)
```

### Aşama 2: Mimari Dönüşüm

```
┌─────────────────────────────────────────────────────────────────┐
│                    BERK iAts Mimarisi                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    RTOS Semantik Katman                  │    │
│  │  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌──────────┐│    │
│  │  │ Sensör    │ │ Navigasyon│ │ Servo     │ │ İletişim ││    │
│  │  │ Görevi    │ │ Görevi    │ │ Görevi    │ │ Görevi   ││    │
│  │  │ P:1 T:5ms │ │ P:2 T:10ms│ │ P:3 T:20ms│ │ P:4 T:50ms│    │
│  │  └─────┬─────┘ └─────┬─────┘ └─────┬─────┘ └────┬─────┘│    │
│  └────────┼─────────────┼─────────────┼────────────┼──────┘    │
│           │             │             │            │            │
│  ┌────────▼─────────────▼─────────────▼────────────▼──────┐    │
│  │                Zero-Copy Message Bus                    │    │
│  │              (SPSC Ring Buffers)                        │    │
│  └─────────────────────────┬──────────────────────────────┘    │
│                            │                                    │
│  ┌─────────────────────────▼──────────────────────────────┐    │
│  │                    HAL Abstraction                      │    │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌───────┐ │    │
│  │  │hal::i2c│ │hal::uart│ │hal::pwm│ │hal::gpio│ │hal::bt││    │
│  │  └────────┘ └────────┘ └────────┘ └────────┘ └───────┘ │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Aşama 3: Algoritma İyileştirmeleri

| Algoritma | Orijinal | BERK Versiyonu |
|-----------|----------|----------------|
| Kalman Filtre | 2D basit | Extended Kalman (std::linalg) |
| Haversine | Integer approx | Full precision (std::math) |
| PID Kontrolör | P-only | Full PID + anti-windup |
| Sensör Füzyon | Complementary | Madgwick/Mahony AHRS |

---

##  Proje Dosya Yapısı

```
iAts/
├── README.md                    # Bu döküman
├── docs/
│   ├── hardware_schematic.md    # Donanım şeması
│   ├── protocol_spec.md         # Protokol tanımları
│   └── performance_analysis.md  # Performans analizi
├── src/
│   ├── main.berk               # Ana program
│   ├── config.berk             # Konfigürasyon
│   ├── tasks/
│   │   ├── sensor_task.berk    # Sensör okuma görevi
│   │   ├── navigation_task.berk # Navigasyon hesaplama
│   │   ├── servo_task.berk     # Servo kontrol
│   │   └── comm_task.berk      # İletişim görevi
│   ├── drivers/
│   │   ├── mpu6050.berk        # IMU sürücüsü
│   │   ├── hmc5883l.berk       # Pusula sürücüsü
│   │   └── servo.berk          # Servo sürücüsü
│   ├── algorithms/
│   │   ├── kalman.berk         # Kalman filtre
│   │   ├── navigation.berk     # Navigasyon algoritmaları
│   │   └── pid.berk            # PID kontrolör
│   └── protocols/
│       ├── nmea.berk           # NMEA parser
│       ├── mavlink.berk        # MAVLink parser
│       └── ltm.berk            # LTM protokolü
└── tests/
    ├── test_kalman.berk        # Kalman testleri
    ├── test_navigation.berk    # Navigasyon testleri
    └── test_servo.berk         # Servo testleri
```

---

##  Beklenen Kazanımlar

### Performans İyileştirmeleri

| Metrik | Arduino (Orijinal) | BERK/ESP32 | İyileşme |
|--------|-------------------|------------|----------|
| CPU Frekansı | 16 MHz | 240 MHz | **15x** |
| RAM | 2 KB | 520 KB | **260x** |
| Float İşlem | ~100 µs | ~0.1 µs (HW FPU) | **1000x** |
| Loop Hızı | ~50 Hz | ~1000 Hz | **20x** |
| Sensör Okuma | 10 ms | 0.5 ms | **20x** |
| Kalman Update | 5 ms | 0.1 ms | **50x** |
| Servo Update | 20 ms (jittery) | 1 ms (deterministic) | **20x** + jitter-free |

### Kalite İyileştirmeleri

| Özellik | Arduino | BERK |
|---------|---------|------|
| Tip Güvenliği | ❌ Manual | ✅ Compile-time |
| Memory Safety | ❌ Buffer overflow riski | ✅ Borrow checker |
| Determinism | ❌ Polling | ✅ RTOS semantik |
| Debugging | ❌ Serial.print | ✅ LSP + breakpoints |
| Concurrency | ❌ Single-threaded | ✅ Multi-task |
| Power Management | ❌ Always-on | ✅ Deep sleep |

### Yeni Özellikler (BERK ile Mümkün)

1. **WiFi/BLE Native**: HC05 yerine ESP32 native → app latency ↓
2. **OTA Update**: Kablosuz firmware güncelleme
3. **Web Dashboard**: Dahili HTTP server
4. **Telemetri Logging**: SD kart veya SPIFFS
5. **Multi-Tracker Sync**: Mesh networking
6. **AI Prediction**: Uçuş yolu tahmini (CUIO-lite)

---

##  İlgili Dosyalar

- [Prototip Kaynak Kodları](src/)
- [Donanım Şeması](docs/hardware_schematic.md)
- [Performans Analizi](docs/performance_analysis.md)
- [Test Dosyaları](tests/)

---

##  Lisans

Bu proje GPL-3.0 lisansı altındadır (orijinal iAts projesiyle uyumlu).

---

*Bu döküman BERK Programlama Dili geliştirme ekibi tarafından hazırlanmıştır.*
