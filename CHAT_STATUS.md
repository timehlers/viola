# VIOLA-Projekt – Chat-/Projektstatus

Stand: 2026-10-21

Dieses Dokument ist ein Wiedereinstiegspunkt für spätere Chats. Es ist kein vollständiger Wort-für-Wort-Export, sondern eine konsolidierte Zusammenfassung der bisherigen Entscheidungen, Messungen und des aktuellen technischen Zustands.

## Projektziel

Die TechniSat-VIOLA-Lampe soll mit einem ESP32-C3 in Home Assistant integriert werden.

Ursprüngliche Probleme der Lampe:

1. Die letzte Lichtfarbe und Helligkeit wurden nicht zuverlässig gespeichert.
2. Beim Einschalten startete sie immer mit der hellsten Stufe.
3. Der Jieli verband sich automatisch mit Bluetooth-Geräten, auch wenn diese Verbindung nicht gewünscht war.
4. Die Lampe hatte keine geeignete Smart-Home-/Home-Assistant-Integration.

Der ESP32 steuert deshalb die beiden Lichtkanäle, den Touch-Eingang und die schaltbare Versorgung des Jieli. Der Jieli bleibt für Bluetooth und Audio erhalten und wird nicht umprogrammiert.

## Wichtige Dateien

- `viola_lampe.yaml` – aktuelles produktives ESPHome
- `README.md` – ausführliche technische Projekt- und Aufbau-Dokumentation
- `lochraster-oben.jpg` / `lochraster-unten.jpg` – finale Lochrasterplatine
- `IMG20260928120217.jpg` – Platinenfoto
- `IMG20260928120217-markings.jpg` – Platinenfoto mit Markierungen
- `viola_*_test.yaml`, `viola_pwm_*.yaml`, `viola_level_measure.yaml` – historische Mess- und Testkonfigurationen

WLAN-Zugangsdaten liegen ausschließlich in einer lokalen `secrets.yaml` und dürfen nicht in dieses Statusdokument übernommen werden.

## Aktueller Hardwarestand

### ESP32-C3-Pinbelegung

| ESP32 | Funktion |
|---|---|
| GPIO0 | Jieli-Versorgung über BC337/BC327-Schalter |
| GPIO1 | TTP223-Ausgang |
| GPIO4 | WW-PWM über 6,8 kΩ |
| GPIO5 | KW-PWM über 6,8 kΩ |
| 5V | Versorgung vom 5-V-Kreis der Lampe |
| GND | gemeinsame Masse |

GPIO6 wurde wegen eines beobachteten Boot-Pegels nicht verwendet. GPIO0 kann beim Booten kurz einen Pegelwechsel zeigen; das ist für die Jieli-Versorgung akzeptabel.

### Versorgung

- Die Lampe versorgt den ESP32 über dessen `5V`-Pin.
- Lampen-GND, ESP32-GND und TTP223-GND sind gemeinsam verbunden.
- Der Jieli wird nicht dauerhaft direkt mit 5 V versorgt.

### Lichtsteuerung

R6 und R11, die ursprünglichen Jieli-PWM-Verbindungen, wurden entfernt.

```text
ESP32 GPIO4 ── 6,8 kΩ ── WW-Basisknoten (Q2/R30/L1)
ESP32 GPIO5 ── 6,8 kΩ ── KW-Basisknoten (Q1/R29/L2)
```

Gemessene externe Widerstände:

- WW: 6,81 kΩ
- KW: 6,85 kΩ

Die Widerstände sind zwingend erforderlich; die ESP32-GPIOs dürfen nicht direkt an die Lampentransistoren angeschlossen werden.

Die finale PWM-Frequenz beträgt 12 kHz. Bei niedrigeren Frequenzen war Spulen-/Schaltpiepen hörbar. Nach Einbau der Serienwiderstände wurden Q1/Q2 nur noch leicht warm und die Temperatur blieb stabil.

### Jieli-Versorgungsschalter

Verwendete Transistoren: BC327 (PNP high-side) und BC337 (NPN).

```text
+5 V Lampe ── BC327 Emitter
BC327 Collector ── Jieli Pin 12

BC327 Base ── 1 kΩ ── BC337 Collector
BC327 Base ── 10 kΩ ── +5 V

BC337 Emitter ── GND
ESP32 GPIO0 ── 4,7 kΩ ── BC337 Base
BC337 Base ── ca. 100 kΩ ── GND
```

Gemessene Werte der aufgebauten Widerstände: ca. 4,67 kΩ und 91,9 kΩ.

Die ursprüngliche niederohmige Jieli-Versorgung zwischen USB-/5-V-Zuleitung und Jieli Pin 12 wurde entfernt. Die Bezeichnung war vermutlich R23 oder R27; entscheidend ist die Verbindung, deren eine Seite an Pin 12 und deren andere Seite an der 5-V-Zuleitung liegt.

### Touch

Der originale Touch-Anschluss wurde über R8 von Jieli Pin 11 getrennt. Die originale Touch-Spirale ist elektrisch nicht mehr mit dem Jieli verbunden.

TTP223:

```text
VCC ── 3,3 V ESP32
GND ── gemeinsamer GND
I/O ── ESP32 GPIO1
```

Die Platine steht auf `AB = 00` (direkter, nicht verriegelnder High-Ausgang). Der eigentliche Sensoreingang ist IC-Pin 3 (`I`), der auf der Modulplatine am rechten Pad des unbestückten C2 erreichbar ist.

Die originale Touch-Spirale nahm unterhalb der Lampenplatine periodische Störungen auf und führte zu falschen Touch-Impulsen. Die Elektrode wurde daher von der Lampenplatine weg positioniert. Ein Kunststoff-Abstandshalter kann zusätzlich helfen.

## Vier zu entfernende Widerstände

Für den beschriebenen Umbau werden vier physische Widerstände bzw. 0-Ω-Brücken entfernt:

| Bauteil | Zweck der Entfernung |
|---|---|
| R6 | Jieli-PWM-Verbindung zu einem Lichtkanal trennen |
| R11 | Jieli-PWM-Verbindung zum zweiten Lichtkanal trennen |
| R8 | originale Touch-Elektrode von Jieli Pin 11 trennen |
| R23 oder R27 | direkte Jieli-5-V-Versorgung trennen |

R29 und R30 bleiben als Bestandteile der Lampen-Lichttreiber erhalten. Die Widerstandsbezeichnungen auf der Platine sind teilweise schwer lesbar; vor dem Auslöten immer die Verbindung mit dem Multimeter über die angeschlossenen Netze identifizieren.

## Aktuelle Touch-Bedienlogik

Bei ausgeschalteter Lampe:

```text
1. Touch       → gespeicherten WW/KW-Zustand wiederherstellen
2. Touch       → 100 % KW
3. Touch       → 100 % NW (WW und KW jeweils 100 %)
4. Touch       → 100 % WW
```

Die weiteren Touches müssen jeweils innerhalb von ungefähr einer Sekunde erfolgen. Nach WW ist die Farbfolge beendet.

Ein langer Touch dimmt den aktuellen Zustand proportional, aber nicht unter 1 %. Zum Ausschalten wird ein separater kurzer Touch verwendet.

Wenn das Farbwahlfenster abgelaufen ist, gilt:

```text
kurzer Touch → aktuellen Zustand speichern und ausschalten
langer Touch → aktuelle Farbe dimmen
```

In Home Assistant gibt es zusätzlich den Schalter `VIOLA Lampe letzter Zustand`. Einschalten stellt die gespeicherten WW/KW-Werte wieder her; Ausschalten speichert den aktuellen Zustand und schaltet die Lampe aus.

## Persistenz

`saved_ww` und `saved_kw` sind in `viola_lampe.yaml` als ESPHome-Globals mit `restore_value: yes` angelegt.

Das ESPHome-Flash-Schreibintervall bleibt absichtlich auf Default, um unnötigen Flash-Verschleiß zu vermeiden. Nach einer Änderung sollte die Lampe daher nicht unmittelbar stromlos gemacht werden, wenn garantiert der gerade eben geänderte Zustand erhalten bleiben soll.

## Nicht weiterverfolgte Ansätze

- Der Jieli wird nicht gehackt oder neu programmiert.
- USB-D+ und USB-D− wurden testweise verbunden, am Rechner aber nicht zuverlässig als nutzbares USB-Gerät erkannt; der Ansatz wurde verworfen.
- Ein Logic-Level-Konverter ist für die finale Licht-PWM nicht erforderlich.
- Der originale Jieli-Touch wird nicht wieder aktiviert.

## Sicherheit und Testhinweise

- Nie ein Lampensignal direkt an einen ESP32-GPIO anschließen.
- Die 6,8-kΩ-Serienwiderstände in den PWM-Leitungen müssen vorhanden sein.
- Vor dem Einschalten Polarität, gemeinsame Masse und Transistor-Pinout prüfen.
- Nach Änderungen mehrere Minuten bei verschiedenen Helligkeiten testen und Q1/Q2 sowie die Umgebung auf ungewöhnliche Erwärmung prüfen.
- Bei wiederkehrendem Touch-Fehlauslösen die Elektrode weiter von der Lampenplatine entfernen und die Sensorleitung verkürzen.

## Flashen

```bash
cd /home/codex/codex/viola
esphome run viola_lampe.yaml
```

Eine lokale `secrets.yaml` mit `wifi_ssid` und `wifi_password` ist erforderlich. Dieses Statusdokument enthält keine Zugangsdaten.
