# 18650 Akku-Tester

[![Projektstatus](https://img.shields.io/badge/Status-funktionsfähig-brightgreen)](https://github.com/)
[![Hardware](https://img.shields.io/badge/Hardware-Arduino%20%7C%20TFT-blue)](https://www.arduino.cc/)

Ein kompakter Arduino-basierter Tester zur Messung der Spannung eines 18650-Lithium-Ionen-Akkus mit Anzeige von Spannung, Ladezustand, Akkusymbol und Status auf einem ST7735-TFT-Display.

## Features

- Messung der Akkuspannung über den Analog-Eingang A0
- Glättung durch Mittelwertbildung aus zehn Messungen
- Prozentberechnung anhand der gemessenen Spannung
- Farbige Statusanzeige:
  - Kein Akku
  - Tiefentladen
  - Akku leer
  - Akku normal
  - Akku voll
- Grafische Darstellung des Ladezustands
- Aktualisierung bei relevanten Spannungsänderungen
- Kompatibel mit Arduino-Boards mit 5-V-Logik und SPI-Schnittstelle

## Hardware-Komponenten

| Komponente | Beschreibung |
|---|---|
| Arduino Uno R3 | Mikrocontroller und Versorgung |
| ST7735 TFT-Display | 1,44-Zoll-Farbdisplay mit SPI-Schnittstelle |
| 18650-Akkuhalter | Aufnahme für einen einzelnen 18650-Akku |
| 18650 Li-Ion-Akku | Zu messende Energiequelle |
| 10 kΩ Widerstand | Pull-Down zwischen A0 und GND; verhindert Floating-Pins und Zufallswerte bei entferntem Akku |
| Anschlusskabel | Verbindung zwischen Akkuhalter, Arduino und Display |
| Optional: Spannungsteiler | Erforderlich, wenn die Eingangsspannung den zulässigen Analogbereich überschreiten kann |

## Schaltplan & Pinbelegung

### Pinout

| Funktion | Anschluss |
|---|---|
| TFT CS | Arduino D10 |
| TFT RST | Arduino D9 |
| TFT DC | Arduino D8 |
| TFT MOSI | Arduino D11 / SPI MOSI |
| TFT SCK | Arduino D13 / SPI SCK |
| TFT VCC | 5 V oder entsprechend der Display-Spezifikation |
| TFT GND | GND |
| Akku-Plus | Arduino A0 |
| Akku-Minus | Arduino GND |
| Pull-Down-Widerstand | 10 kΩ zwischen A0 und GND |

### Text-Schaltbild

```text
                         +----------------------+
                         |     Arduino Uno      |
                         |                      |
          TFT CS -------| D10                  |
          TFT RST ------| D9                   |
          TFT DC -------| D8                   |
          TFT MOSI -----| D11 / MOSI           |
          TFT SCK ------| D13 / SCK             |
          TFT VCC ------| 5 V                  |
          TFT GND ------| GND                  |
                         |                      |
             Akku-Plus ----| A0                   |
             Akku-Minus ---| GND                  |
                         +----------------------+

       18650-Akku
       +  --------------------------> A0
       -  --------------------------> GND

          Pull-Down-Widerstand
          A0  ------------------[ 10 kΩ ]-----> GND
```

        Der 10 kΩ Pull-Down-Widerstand ist direkt zwischen A0 und GND angeschlossen. Ist kein Akku eingelegt, zieht er den Analogeingang definiert auf 0 V und verhindert dadurch Floating-Pins sowie zufällige Messwerte. Bei eingelegtem Akku bildet der Widerstand zusammen mit dem Akkuanschluss die Messschaltung; der Messwert wird weiterhin an A0 erfasst.

> [!WARNING]
> Der Analog-Eingang A0 darf niemals eine Spannung außerhalb des zulässigen Eingangsbereichs des verwendeten Arduino-Boards erhalten. Beim Arduino Uno liegt dieser Bereich standardmäßig zwischen 0 V und 5 V. Ein einzelner 18650-Akku erreicht maximal etwa 4,2 V und kann bei direktem Anschluss grundsätzlich innerhalb dieses Bereichs liegen. Bei mehreren Zellen, einer externen Spannungsquelle oder einem anderen Referenzspannungspegel muss ein korrekt dimensionierter Spannungsteiler verwendet werden. Lithium-Ionen-Akkus dürfen nicht überladen, tiefentladen, kurzgeschlossen oder beschädigt betrieben werden.

## Benötigte Bibliotheken

Die folgenden Bibliotheken müssen über den Arduino Library Manager installiert werden:

- [Adafruit GFX Library](https://github.com/adafruit/Adafruit-GFX-Library)
- [Adafruit ST7735 and ST7789 Library](https://github.com/adafruit/Adafruit-ST7735-Library)
- SPI ist Bestandteil der Arduino-Standardbibliothek

## Code

```cpp
#include <Adafruit_GFX.h>
#include <Adafruit_ST7735.h>
#include <SPI.h>

// Pin-Definitionen für das ST7735 TFT
#define TFT_CS     10
#define TFT_RST     9
#define TFT_DC      8
#define AKKU_PIN   A0

Adafruit_ST7735 tft = Adafruit_ST7735(TFT_CS, TFT_DC, TFT_RST);

// Hilfsvariablen für die Glättung & Update-Logik
float letzteSpannung = -1.0;

void setup() {
  // Display initialisieren
  tft.initR(INITR_144GREENTAB);
  tft.fillScreen(ST77XX_BLACK);
  tft.setRotation(1); // Querformat (0-3)

  // Feste Rahmen und Titel zeichnen
  tft.setTextColor(ST77XX_WHITE);
  tft.setTextSize(1);
  tft.setCursor(8, 6);
  tft.println("18650 AKKU TESTER");
  tft.drawFastHLine(0, 18, 128, ST77XX_WHITE);
}

// Funktion zur Umrechnung der Spannung in Prozent
int berechneProzent(float volt) {
  if (volt >= 4.20) return 100;
  if (volt <= 3.00) return 0;

  if (volt > 3.85) {
    return map(volt * 100, 385, 420, 50, 100);
  } else if (volt > 3.65) {
    return map(volt * 100, 365, 385, 20, 50);
  } else {
    return map(volt * 100, 300, 365, 0, 20);
  }
}

// Funktion zum Zeichnen des Akkusymbols
void zeichneAkkuIcon(
  int x,
  int y,
  int w,
  int h,
  int prozent,
  uint16_t farbe
) {
  tft.drawRect(x, y, w, h, ST77XX_WHITE);
  tft.fillRect(x + w, y + (h / 4), 3, h / 2, ST77XX_WHITE);
  tft.fillRect(x + 2, y + 2, w - 4, h - 4, ST77XX_BLACK);

  int maxBreite = w - 4;
  int fuellBreite = map(prozent, 0, 100, 0, maxBreite);

  if (fuellBreite > 0) {
    tft.fillRect(x + 2, y + 2, fuellBreite, h - 4, farbe);
  }
}

void loop() {
  // 10 Messungen durchführen und Mittelwert bilden
  long summe = 0;

  for (int i = 0; i < 10; i++) {
    summe += analogRead(AKKU_PIN);
    delay(5);
  }

  float rawValue = summe / 10.0;
  float spannung = rawValue * (5.0 / 1023.0);

  if (abs(spannung - letzteSpannung) > 0.02) {
    letzteSpannung = spannung;

    int prozent = 0;
    uint16_t statusFarbe = ST77XX_WHITE;
    String statusText = "";

    if (spannung < 0.6) {
      prozent = 0;
      statusFarbe = ST77XX_WHITE;
      statusText = "KEIN AKKU";
    } else if (spannung < 3.0) {
      prozent = 0;
      statusFarbe = ST77XX_RED;
      statusText = "TIEFENTLADEN!";
    } else if (spannung < 3.5) {
      prozent = berechneProzent(spannung);
      statusFarbe = ST77XX_RED;
      statusText = "AKKU LEER";
    } else if (spannung < 3.9) {
      prozent = berechneProzent(spannung);
      statusFarbe = ST77XX_YELLOW;
      statusText = "AKKU NORMAL";
    } else {
      prozent = berechneProzent(spannung);
      statusFarbe = ST77XX_GREEN;
      statusText = "AKKU VOLL";
    }

    zeichneAkkuIcon(12, 28, 100, 32, prozent, statusFarbe);

    tft.fillRect(0, 68, 128, 60, ST77XX_BLACK);

    // Spannung und Prozent anzeigen
    tft.setCursor(8, 72);
    tft.setTextSize(2);
    tft.setTextColor(ST77XX_CYAN);
    tft.print(spannung, 2);
    tft.print("V ");

    tft.setTextColor(statusFarbe);
    tft.print(prozent);
    tft.print("%");

    // Reste aus vorherigen längeren Texten in der Zeile löschen
    tft.print("  ");

    tft.setCursor(12, 100);
    tft.setTextSize(1);
    tft.setTextColor(statusFarbe);
    tft.println(statusText);
  }

  delay(500);
}
```

## Lizenz / Hinweis

Dieses Projekt wird ohne Gewährleistung bereitgestellt. Die Nutzung erfolgt auf eigene Verantwortung. Besonders beim Umgang mit Lithium-Ionen-Akkus sind geeignete Schutzmaßnahmen und ein sicherer mechanischer Aufbau erforderlich.

