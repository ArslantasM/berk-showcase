# iAts BERK - Donanım Şeması

Bu doküman, iAts antenna tracker için ESP32 tabanlı BERK implementasyonunun donanım bağlantılarını açıklar.

##  ESP32 Pin Bağlantıları

### Genel Bakış

```
                           ┌─────────────────────┐
                           │      ESP32-WROOM    │
                           │                     │
        MPU6050 SDA   ──── │ GPIO21 (I2C SDA)    │
        HMC5883L SDA       │                     │
                           │                     │
        MPU6050 SCL   ──── │ GPIO22 (I2C SCL)    │
        HMC5883L SCL       │                     │
                           │                     │
        GPS TX        ──── │ GPIO16 (UART2 RX)   │
        GPS RX        ──── │ GPIO17 (UART2 TX)   │
                           │                     │
        Pan Servo     ──── │ GPIO25 (PWM)        │
        Tilt Servo    ──── │ GPIO26 (PWM)        │
                           │                     │
        Status LED    ──── │ GPIO2 (Built-in)    │
                           │                     │
                           │ GND ────────────────│─── GND Common
                           │ 3.3V ───────────────│─── Sensors VCC
                           │ VIN ────────────────│─── 5V Power In
                           └─────────────────────┘
```

##  Bileşen Listesi

### Ana Bileşenler

| Bileşen | Model | Miktar | Açıklama |
|---------|-------|--------|----------|
| MCU | ESP32-WROOM-32 | 1 | Ana işlemci |
| IMU | MPU6050 | 1 | 6-axis Gyro + Accelerometer |
| Magnetometer | HMC5883L veya QMC5883 | 1 | 3-axis Compass |
| GPS | NEO-6M veya uyumlu | 1 | NMEA çıkışlı GPS |
| Pan Servo | MG995/MG996R | 1 | Yatay hareket |
| Tilt Servo | MG995/MG996R | 1 | Dikey hareket |

### Güç Kaynağı

| Bileşen | Özellik |
|---------|---------|
| Ana Güç | 7-12V DC, min 2A |
| Servo Güç | 5V 3A ayrı regülatör |
| Lojik Güç | 3.3V (ESP32 regülatör) |

### Opsiyonel

| Bileşen | Kullanım |
|---------|----------|
| OLED Display | Durum gösterimi (I2C) |
| SD Kart Modülü | Veri kaydı (SPI) |
| Buzzer | Sesli uyarılar |

##  Detaylı Bağlantılar

### I2C Bus (3.3V)

```
ESP32                MPU6050              HMC5883L
─────                ───────              ────────
GPIO21 (SDA) ────────┬─ SDA ──────────────── SDA
                     │
GPIO22 (SCL) ────────┼─ SCL ──────────────── SCL
                     │
3.3V ────────────────┼─ VCC ──────────────── VCC
                     │
GND ─────────────────┴─ GND ──────────────── GND

MPU6050:
  AD0 → GND (Adres: 0x68)

HMC5883L:
  DRDY → Bağlanmayabilir (polling mode)
```

**Önemli Notlar:**
- I2C hatlarına 4.7kΩ pull-up dirençleri eklenebilir
- Kısa kablo kullanın (<10cm)
- QMC5883 kullanılıyorsa adres 0x0D olacaktır

### UART (GPS)

```
ESP32                    GPS Module (NEO-6M)
─────                    ───────────────────
GPIO16 (UART2 RX) ────── TX
GPIO17 (UART2 TX) ────── RX
3.3V ─────────────────── VCC
GND ──────────────────── GND
```

**GPS Ayarları:**
- Baud rate: 9600 (varsayılan) veya 115200
- Protokol: NMEA 0183
- Güncelleme hızı: 5-10 Hz önerilir

### Servo Bağlantıları

```
ESP32                    Servo Power          Pan Servo (MG995)
─────                    ───────────          ─────────────────
GPIO25 ──────────────────────────────────────── Signal (Orange)
                         5V ─────────────────── VCC (Red)
GND ─────────────────────GND ────────────────── GND (Brown)

GPIO26 ──────────────────────────────────────── Signal (Tilt Servo)
                         5V ─────────────────── VCC (Tilt Servo)
GND ─────────────────────GND ────────────────── GND (Tilt Servo)
```

**Servo Güç Notları:**
- ⚠️ Servoları ESP32'den beslemeyin! Ayrı 5V regülatör kullanın.
- Her MG995 servo ~1.5A peak çekebilir
- 5V 3A regülatör önerilir (örn: LM2596)
- GND'ler ortak olmalı

### PWM Konfigürasyonu

```c
// BERK config.berk'ten
yapı PinConfig yap
    servo_pan_pin: 25,   // LEDC Channel 0
    servo_tilt_pin: 26,  // LEDC Channel 1
    // PWM: 50Hz, 16-bit çözünürlük
son
```

##  Mekanik Montaj

### Pan-Tilt Mekanizması

```
        ┌─────────────────────────────┐
        │     Tilt Servo Platform     │
        │  ┌─────────────────────┐    │
        │  │   ◄── Antenna ──►   │    │
        │  │                     │    │
        │  └─────────┬───────────┘    │
        │            │                │
        │     [Tilt Servo]            │
        │            │                │
        └────────────┼────────────────┘
                     │
              [Pan Servo]
                     │
        ─────────────┴─────────────
              Taban (Sabit)
```

### Sensör Yerleşimi

- **MPU6050/HMC5883L**: Pan ekseni üzerinde, tilt hareketi ile birlikte dönmeli
- **GPS**: Montaj tabanından en az 10cm uzakta, gökyüzüne açık
- **ESP32**: Koruyucu kutu içinde, ısıdan uzak

##  Güç Dağıtımı

```
                    ┌─────────────────────────────────┐
                    │           Güç Girişi             │
                    │          7-12V DC 3A            │
                    └───────────────┬─────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
              ┌─────▼─────┐   ┌─────▼─────┐   ┌─────▼─────┐
              │  ESP32    │   │  Servo    │   │ 3.3V LDO  │
              │ VIN → 3.3V│   │ 5V 3A Reg │   │ (Backup)  │
              └───────────┘   └─────┬─────┘   └─────┬─────┘
                    │               │               │
              Lojik Güç        Servo Güç      Sensör Güç
              (ESP32)          (Servolar)     (Opsiyonel)
                    │               │               │
                    └───────────────┴───────────────┘
                                    │
                              GND (Ortak)
```

### Güç Hesaplamaları

| Bileşen | Tipik Akım | Max Akım |
|---------|------------|----------|
| ESP32 | 80mA | 500mA (WiFi TX) |
| MPU6050 | 4mA | 4mA |
| HMC5883L | 1mA | 2mA |
| GPS (NEO-6M) | 45mA | 67mA |
| Pan Servo | 200mA | 1500mA |
| Tilt Servo | 200mA | 1500mA |
| **Toplam** | **530mA** | **~3.6A** |

##  Montaj Adımları

### 1. PCB/Breadboard Hazırlığı

1. ESP32 dev board'u merkeze yerleştirin
2. I2C sensörleri breadboard'un bir tarafına koyun
3. Güç regülatörünü ısı dağılımı için köşeye yerleştirin

### 2. I2C Bağlantıları

1. SDA ve SCL hatlarını bağlayın
2. 3.3V ve GND bağlayın
3. I2C adresleri kontrol edin (scanner ile)

### 3. UART Bağlantıları

1. GPS modülünü bağlayın (TX→RX, RX→TX dikkat!)
2. GPS'i açık alana çıkarın
3. Fix alana kadar bekleyin (ilk sefer 1-5 dakika)

### 4. Servo Bağlantıları

1. Ayrı servo güç kaynağını bağlayın
2. GND'leri birleştirin
3. Sinyal kablolarını ESP32'ye bağlayın
4. Servoları test edin (center konumu)

### 5. Final Test

1. Güç verin
2. LED durumunu kontrol edin
3. Seri monitörden logları izleyin
4. Sensör verilerini doğrulayın

##  PCB Tasarımı (Opsiyonel)

Prototip aşamasından sonra özel PCB tasarımı için:

### Önerilen PCB Özellikleri

- 2 katman yeterli
- En az 1oz bakır (güç hatları için)
- USB-C şarj girişi
- JST konektörler (servolar için)
- Programlama header'ı
- Reset ve Boot butonları

### Kritik Layout Kuralları

1. I2C hatlarını kısa tutun
2. Servo güç hatlarını geniş yapın
3. Dijital ve analog GND'leri bir noktada birleştirin
4. MPU6050'yi titreşimden uzak tutun
5. HMC5883L'yi motorlardan uzak tutun (manyetik girişim)

##  Koruma

### ESD Koruması

- TVS diyotları UART ve I2C hatlarında
- USB girişinde ESD koruması

### Aşırı Akım Koruması

- Polyfuse ana güç girişinde
- Her servo hattında sigorta

### Gürültü Filtrasyonu

- 100nF kapasitör her VCC-GND çiftinde
- 10μF elektrolitik güç girişlerinde
- Ferrit boncuk servo güç hattında

---

*Not: Bu şema referans amaçlıdır. Gerçek uygulamada bileşen datasheetlerini kontrol edin ve güvenlik önlemlerini alın.*
