# HOV-ESP32 Firmware Update – UART Protokoll Anpassung

## Aufgabe
Aktualisiere die HOV-ESP32 Firmware, damit sie das neue erweiterte UART-Feedback-Protokoll
von der STM32-Firmware (hoverboard-firmware-hack-FOC) korrekt empfängt und Fehlermeldungen
im Klartext in der UI anzeigt.

---

## Neues STM32 SerialFeedback Struct (Referenz, 24 Bytes)

Das STM32 sendet jetzt folgendes Struct (little-endian, 115200 Baud, 8N1, alle 10 ms):

```
Offset  Größe  Typ       Name           Beschreibung
------  -----  -------   ----------     ----------------------------------
  0      2     uint16    start          Frame-Start-Marke: 0xABCD
  2      2     int16     cmd1           input1-Befehl         [-1000, 1000]
  4      2     int16     cmd2           input2-Befehl         [-1000, 1000]
  6      2     int16     speedR_meas    Rechter Motorspeed    [rpm]
  8      2     int16     speedL_meas    Linker  Motorspeed    [rpm]
 10      2     int16     batVoltage     Batteriespannung * 100 (3600 = 36.00 V)
 12      2     int16     boardTemp      Boardtemperatur  * 10  ( 358 = 35.8 °C)
 14      2     uint16    cmdLed         LED / Sideboard Flags (Bitmask)
 16      2     int16     left_dc_curr   Linker  DC-Busstrom * 100 (150 = 1.50 A)
 18      2     int16     right_dc_curr  Rechter DC-Busstrom * 100
 20      1     uint8     errCodeL       Linker  Motor Fehlercode (Bitmask, s.u.)
 21      1     uint8     errCodeR       Rechter Motor Fehlercode (Bitmask, s.u.)
 22      2     uint16    checksum       XOR über alle Felder oben
------  -----
        24     Bytes gesamt
```

### Fehlercode-Bitmask (errCodeL / errCodeR)
```
Bit 0 (Wert 1): Hall-Sensor-Fehler: Zustand 000 (alle LOW)  → Kabelbruch / Stecker lose
Bit 1 (Wert 2): Hall-Sensor-Fehler: Zustand 111 (alle HIGH) → Kurzschluss / Kabelbruch
Bit 2 (Wert 4): Motor blockiert: Stillstand trotz hohem Sollwert (Rotor gesperrt / Überlast)

Kombinationen:
  0 = kein Fehler
  1 = Hall 000
  2 = Hall 111
  3 = Hall 000 + Hall 111 (beide gleichzeitig → schwerer Verdrahtungsfehler)
  4 = Motor blockiert
  5 = Hall 000 + blockiert
  6 = Hall 111 + blockiert
  7 = alle drei Fehler
```

### Checksumme
```cpp
checksum = start ^ (uint16_t)cmd1 ^ (uint16_t)cmd2
         ^ (uint16_t)speedR_meas ^ (uint16_t)speedL_meas
         ^ (uint16_t)batVoltage  ^ (uint16_t)boardTemp
         ^ cmdLed
         ^ (uint16_t)left_dc_curr ^ (uint16_t)right_dc_curr
         ^ errCodeL ^ errCodeR;
```

---

## Änderungen in der ESP32-Firmware

### 1. SerialFeedback Struct aktualisieren

Ersetze das bisherige `SerialFeedback`-Struct durch:

```cpp
#pragma pack(push, 1)
typedef struct {
  // ---- EFeru original (unverändert) ----
  uint16_t  start;           // 0xABCD
  int16_t   cmd1;
  int16_t   cmd2;
  int16_t   speedR_meas;
  int16_t   speedL_meas;
  int16_t   batVoltage;      // * 100
  int16_t   boardTemp;       // * 10
  uint16_t  cmdLed;
  // ---- Erweiterung ----
  int16_t   left_dc_curr;    // * 100
  int16_t   right_dc_curr;   // * 100
  uint8_t   errCodeL;
  uint8_t   errCodeR;
  uint16_t  checksum;
} SerialFeedback;
#pragma pack(pop)
```

**Wichtig:** `#pragma pack(push, 1)` verwenden, damit kein Padding eingebaut wird.
`sizeof(SerialFeedback)` muss exakt **24** ergeben.

---

### 2. Empfangspuffer anpassen

Den Empfangspuffer auf die neue Struct-Größe aktualisieren:

```cpp
static uint8_t  s_feedbackBuf[sizeof(SerialFeedback)];
static uint8_t  s_feedbackIdx = 0;
```

---

### 3. Checksummen-Prüfung aktualisieren

```cpp
uint16_t checksum = fb->start
  ^ (uint16_t)fb->cmd1        ^ (uint16_t)fb->cmd2
  ^ (uint16_t)fb->speedR_meas ^ (uint16_t)fb->speedL_meas
  ^ (uint16_t)fb->batVoltage  ^ (uint16_t)fb->boardTemp
  ^ fb->cmdLed
  ^ (uint16_t)fb->left_dc_curr ^ (uint16_t)fb->right_dc_curr
  ^ fb->errCodeL ^ fb->errCodeR;

if (checksum != fb->checksum) return;
```

---

### 4. Neue Felder auswerten

Nach der Checksummen-Prüfung die neuen Felder übernehmen:

```cpp
// Vorhandene Felder (unverändert)
l_speedR     = fb->speedR_meas;
l_speedL     = fb->speedL_meas;
l_batVoltage = fb->batVoltage;
l_boardTemp  = fb->boardTemp;
l_stm32Cmd1  = fb->cmd1;
l_stm32Cmd2  = fb->cmd2;
l_stm32FlagsRaw = fb->cmdLed;

// Neue Felder: Ströme mit Tiefpassfilter glätten
const float kCurrAlpha = 0.15f;
l_dcCurrLeftFilt  += ((float)fb->left_dc_curr  - l_dcCurrLeftFilt)  * kCurrAlpha;
l_dcCurrRightFilt += ((float)fb->right_dc_curr - l_dcCurrRightFilt) * kCurrAlpha;
l_dcCurrLeft  = (int16_t)lroundf(l_dcCurrLeftFilt);
l_dcCurrRight = (int16_t)lroundf(l_dcCurrRightFilt);

// Neue Felder: Fehlercodes
l_errCodeL = fb->errCodeL;
l_errCodeR = fb->errCodeR;
```

Neue globale Variablen deklarieren:
```cpp
static uint8_t  l_errCodeL = 0;
static uint8_t  l_errCodeR = 0;
```

---

### 5. Fehlermeldungen im Klartext in der UI anzeigen

Hilfsfunktion zum Umwandeln eines Fehlerbitfelds in einen lesbaren String:

```cpp
/**
 * Gibt einen Klartext-String für einen motor-Fehlercode zurück.
 * code: errCodeL oder errCodeR aus SerialFeedback
 */
String motorErrorText(uint8_t code) {
  if (code == 0) return "OK";
  String msg = "";
  if (code & 0x01) msg += "Hall-000 (Kabelbruch?) ";
  if (code & 0x02) msg += "Hall-111 (Kurzschluss?) ";
  if (code & 0x04) msg += "Motor blockiert ";
  msg.trim();
  return msg;
}
```

In der UI-Rendering-Funktion (z.B. `drawUI()` oder `loop()`):

```cpp
// Fehleranzeige nur wenn Fehler vorhanden
if (l_errCodeL != 0 || l_errCodeR != 0) {
  // Beispiel: Anzeige auf Serial / Display / OLED / WebUI
  String errL = motorErrorText(l_errCodeL);
  String errR = motorErrorText(l_errCodeR);

  // Für Serial-Log:
  Serial.printf("[FEHLER] Links:  %s\n", errL.c_str());
  Serial.printf("[FEHLER] Rechts: %s\n", errR.c_str());

  // Für OLED / TFT (Beispiel, ggf. anpassen):
  display.setTextColor(RED);
  display.setCursor(0, errorYPos);
  display.printf("L: %s", errL.c_str());
  display.setCursor(0, errorYPos + lineHeight);
  display.printf("R: %s", errR.c_str());

  // Für WebUI (JSON-Response):
  // { "errL": "Hall-000 (Kabelbruch?)", "errR": "OK" }
}
```

---

### 6. Stromwerte in der UI anzeigen

```cpp
// Ströme: Wert ist * 100, also durch 100.0f teilen für Ampere-Anzeige
float currL_A = l_dcCurrLeft  / 100.0f;
float currR_A = l_dcCurrRight / 100.0f;

Serial.printf("Strom L: %.2f A  Strom R: %.2f A\n", currL_A, currR_A);

// Für OLED:
display.printf("iL:%.1fA iR:%.1fA", currL_A, currR_A);
```

---

## Timing-Übersicht

| Parameter         | Wert                             |
|-------------------|----------------------------------|
| Baudrate          | 115200 Baud                      |
| Strukt-Größe      | 24 Bytes                         |
| Übertragungszeit  | ~2.1 ms (24 × 10 Bit / 115200)   |
| Sendeintervall    | 10 ms (STM32 main_loop × 2)      |
| Reserve           | 7.9 ms → **kein Engpass**        |

---

## Kompatibilität mit dem Original EFeru-Protokoll

Die ersten 16 Bytes (bis inkl. `cmdLed`) sind **identisch** mit dem Upstream-EFeru-Protokoll.
ESP32-Code, der das alte 18-Byte-Protokoll nutzte, muss auf 24 Bytes umgestellt werden.
