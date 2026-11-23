
**Stacja środowiskowa na ESP32 z alarmem i sterowaniem przez MQTT**

Projekt DIY łączący elektronikę analogową, cyfrową i IoT: ESP32 mierzy temperaturę i natężenie światła, pokazuje je na wyświetlaczu LCD, sygnalizuje stan diodami i brzęczykiem oraz wysyła dane przez Wi-Fi do brokera MQTT. Urządzeniem można też sterować zdalnie (np. z telefonu).

> **Status: w trakcie realizacji.** Zaprojektowana jest architektura i piny, w KiCadzie powstaje schemat (gotowy blok z termistorem). Firmware i budowa na płytce stykowej są następnym krokiem. Tabela statusu na dole pokazuje, co jest zrobione.

## Funkcje (docelowe)

- Pomiar **temperatury** (termistor NTC 10k) i **natężenia światła** (fotorezystor) przez ADC ESP32.
- Wyświetlanie odczytów na **LCD 1602 (I2C)**, przełączanie ekranów przyciskiem.
- **Pasek 8 diod LED** przez rejestr przesuwny 74HC595 pokazuje poziom oświetlenia.
- **Dioda RGB** zmienia kolor z temperaturą (zimno / OK / gorąco).
- **Alarm**: brzęczyk aktywny sterowany tranzystorem NPN (PN2222) po przekroczeniu progu temperatury.
- **Potencjometr** ustawia próg alarmu na żywo.
- **MQTT**: publikowanie pomiarów i subskrypcja komend (zmiana progu, wyciszenie alarmu, tekst na LCD).

## Sprzęt

| Element | Rola |
|---|---|
| ESP32 DevKit (WROOM-32, 38 pin, USB-C) | mikrokontroler, Wi-Fi |
| Termistor NTC 10k + rezystor 10k | pomiar temperatury (dzielnik napięcia) |
| Fotorezystor + rezystor 10k | pomiar światła (dzielnik napięcia) |
| Potencjometr | ustawianie progu alarmu |
| LCD 1602 + adapter I2C (PCF8574) | wyświetlacz |
| 74HC595 + 8 diod LED + rezystory 220 Ω | wskaźnik poziomu światła |
| Dioda RGB + rezystory 220 Ω | wskaźnik temperatury |
| Tranzystor PN2222 + rezystor 1k + brzęczyk aktywny | alarm |
| Przycisk | zmiana ekranu / wyciszenie |
| Płytka stykowa, przewody | montaż |

## Przypisanie pinów ESP32

| Funkcja | Pin ESP32 | Uwagi |
|---|---|---|
| Termistor (dzielnik) | GPIO34 | ADC1; wejście tylko do odczytu |
| Fotorezystor (dzielnik) | GPIO35 | ADC1 |
| Potencjometr | GPIO32 | ADC1 |
| LCD I2C SDA / SCL | GPIO21 / GPIO22 | zasilanie 5V z VIN |
| 74HC595 DATA / CLOCK / LATCH | GPIO23 / GPIO18 / GPIO5 | SRCLR do 3V3, OE do GND |
| LED RGB (R, G, B) | GPIO25, GPIO26, GPIO27 | każdy kolor przez 220 Ω |
| Brzęczyk (przez tranzystor) | GPIO14 | rezystor 1k do bazy |
| Przycisk | GPIO13 | `INPUT_PULLUP` |

**Decyzje projektowe**

- Wszystkie czujniki analogowe są na **ADC1**, bo ADC2 nie działa przy włączonym Wi-Fi.
- Dzielniki napięcia zasilane są z **3,3 V**, bo wejścia ADC ESP32 nie tolerują 5 V.
- Termistor jest po stronie 3V3, a rezystor stały po stronie GND, więc napięcie na wejściu ADC **rośnie** wraz z temperaturą.
- Piny GPIO6–GPIO11 są zajęte przez pamięć flash i nie są używane.

## Termistor: jak to działa

NTC zmienia rezystancję wraz z temperaturą (około 33 kΩ w 0°C, 10 kΩ w 25°C, 3,6 kΩ w 50°C). ADC mierzy napięcie, więc zmianę oporu zamieniam na napięcie dzielnikiem:

```
3V3 ──[ NTC 10k ]──┬──[ 10k ]── GND
                   │
                GPIO34
```

`V = 3,3 V × 10k / (R_NTC + 10k)`. Firmware przelicza odczyt na rezystancję, a potem na temperaturę wzorem β (równanie Steinharta-Harta w uproszczonej postaci). Zależność jest nieliniowa, więc zwykłe skalowanie nie wystarcza.


## Tematy MQTT

| Temat | Kierunek | Wartość |
|---|---|---|
| `dom/stacja/temperatura` | publikacja | np. `23.4` (°C) |
| `dom/stacja/swiatlo` | publikacja | np. `71` (%) |
| `dom/stacja/alarm` | publikacja | `ON` / `OFF` |
| `dom/stacja/prog/set` | subskrypcja | próg alarmu w °C |
| `dom/stacja/alarm/mute` | subskrypcja | `1` wycisza alarm |
| `dom/stacja/lcd/set` | subskrypcja | tekst na wyświetlaczu |

Planowane: wiadomość *Last Will* informująca, że stacja jest offline, oraz automatyczne wznawianie połączenia Wi-Fi i MQTT.

## Struktura repozytorium

```
.
├── README.md
├── kicad/ #pliki kicad oraz dokumentacja
├── docs/
│   └── images/                            # zrzuty schematu
└── firmware/                              # kod ESP32 (do dodania)
```

Symbol `ESP32_DevKit_38pin.kicad_sym` narysowałem samodzielnie na podstawie pinoutu mojej płytki, bo gotowego symbolu nie ma w standardowej bibliotece KiCada. Dodaje się go przez *Preferencje → Zarządzaj bibliotekami symboli → Biblioteki projektu*.

## Plan prac i status

- [x] Dobór pinów i architektura
- [x] Własny symbol ESP32 w KiCadzie
- [x] Schemat: zasilanie i dzielnik z termistorem
- [ ] Schemat: fotorezystor, potencjometr, LCD, 74HC595, RGB, alarm
- [ ] Symulacja analogowych bloków w ngspice
- [ ] Połączenie Wi-Fi i MQTT (publikacja testowa)
- [ ] Odczyt czujników i przeliczanie na °C i %
- [ ] Obsługa LCD, diod i alarmu
- [ ] Sterowanie zdalne przez MQTT
- [ ] Zdjęcia i opis gotowego układu


## Technologie

ESP32 (Arduino / PlatformIO), MQTT (Mosquitto, biblioteka PubSubClient), KiCad, ngspice.