# TechniSat VIOLA – ESP32-C3-Umbau

Dieses Projekt ersetzt die ursprüngliche Licht- und Touch-Steuerung der TechniSat-VIOLA-Lampe teilweise durch einen ESP32-C3. Die Lampe behält ihre LED-Leistungselektronik und den Jieli-Audio/Bluetooth-Chip; der ESP32 übernimmt die beiden Lichtkanäle, die Touch-Auswertung und die schaltbare Versorgung des Jieli.

Der aktuelle produktive ESPHome-Stand befindet sich in [`viola_lampe.yaml`](viola_lampe.yaml).

## Ziel des Umbaus

Die ursprüngliche Lampe hatte zwei wesentliche Nachteile:

1. Die letzte Lichtfarbe und Helligkeit wurden nicht zuverlässig gespeichert.
2. Beim Einschalten startete sie immer mit der hellsten Stufe.
3. Der Jieli verband sich automatisch mit Bluetooth-Geräten, auch wenn diese Verbindung nicht gewünscht war.
4. Die Lampe hatte keine geeignete Smart-Home- bzw. Home-Assistant-Integration.

Der Umbau soll deshalb:

- Warmweiß und Kaltweiß per PWM steuern,
- die Helligkeit in Home Assistant dimmbar machen,
- den letzten WW/KW-Zustand speichern,
- den Jieli bei Bedarf vollständig stromlos schalten,
- eine eigene Touch-Elektrode über einen TTP223 verwenden,
- verhindern, dass sich der Jieli ungefragt automatisch mit Bluetooth-Geräten verbindet,
- Bluetooth und Audio bei Bedarf weiterhin gezielt einschalten können,
- die Lampe in Home Assistant bzw. ein Smart-Home-System integrieren. Die originale
  Lampe bot dafür keine geeignete Smart-Home-Schnittstelle.

## Übersicht zu entfernende Widerstände

Für den beschriebenen Umbau werden vier physische Widerstände bzw. 0-Ω-Brücken entfernt:

| Bauteil | Funktion im Original | Warum entfernen? | Neue Lösung |
|---|---|---|---|
| **R6** | Jieli-PWM-Verbindung zu einem Lichtkanal | Jieli soll diesen Kanal nicht mehr steuern | ESP32 GPIO4 über 6,8 kΩ zum WW-Basisknoten |
| **R11** | Jieli-PWM-Verbindung zum zweiten Lichtkanal | Jieli soll diesen Kanal nicht mehr steuern | ESP32 GPIO5 über 6,8 kΩ zum KW-Basisknoten |
| **R8** | 0-Ω-Verbindung der originalen Touch-Elektrode zu Jieli Pin 11 | Originaler Jieli-Touch wird nicht mehr verwendet | TTP223 an eigener Elektrode, Ausgang an GPIO1 |
| **R23/R27** | Niederohmige Verbindung der 5-V-Zuleitung zu Jieli Pin 12 | Jieli-Versorgung soll schaltbar werden | BC327/BC337-Hochseitenschalter an GPIO0 |

Bei **R23/R27** war die Beschriftung auf der Platine nicht eindeutig lesbar. Gemeint ist das Bauteil, dessen eine Seite direkt mit Jieli Pin 12 und dessen andere Seite mit der USB-/5-V-Zuleitung verbunden ist. Es wird nur dieses eine Bauteil entfernt – nicht zusätzlich R29, R30 oder andere Widerstände der Lichttreiber.

Nach dem Entfernen sollten die ursprünglichen Verbindungen messbar getrennt sein:

- Jieli Pin 1/2 sind von den beiden Licht-Basisknoten getrennt.
- Jieli Pin 11 ist von der Touch-Elektrode getrennt.
- Jieli Pin 12 ist von der direkten 5-V-Zuleitung getrennt.

Die ausgelöteten Bauteile sollten vor dem Einbau des ESP32 fotografiert und die jeweiligen Pads markiert werden. Die neue Verdrahtung wird an den nun freien bzw. lampenseitigen Pads angeschlossen.

## Aktueller Hardwarestand

### ESP32-C3

Verwendet wird ein ESP32-C3, im finalen Aufbau als kompakter C3-mini.

| ESP32-Pin | Funktion | Anschluss |
|---|---|---|
| GPIO0 | Jieli-Versorgung | Ansteuerung des BC337/BC327-Schalters |
| GPIO1 | Touch-Eingang | TTP223-Ausgang `I/O` |
| GPIO4 | WW-PWM | über 6,8 kΩ zum WW-Basisknoten |
| GPIO5 | KW-PWM | über 6,8 kΩ zum KW-Basisknoten |
| 5V | Versorgung | an die 5-V-Versorgung der Lampe |
| GND | Masse | gemeinsame Masse mit Lampe und TTP223 |

GPIO6 wurde im finalen Aufbau vermieden, weil beim Booten ein kurzzeitiger Pegel beobachtet wurde. GPIO0 zeigt eventuell ebenfalls einen sehr kurzen Pegelwechsel beim Booten; das ist bei der Jieli-Versorgung akzeptabel, da dadurch keine gefährliche externe Last geschaltet wird.

### Versorgung

- Die 5-V-Versorgung der Lampe versorgt auch den ESP32 über dessen `5V`-Pin.
- ESP32-GND, Lampen-GND und TTP223-GND sind verbunden.
- Es bleiben keine zwei logisch getrennten Massepotentiale bestehen.
- Der Jieli bekommt seine 5 V nicht dauerhaft direkt, sondern über den BC327-Hochseitenschalter.

### Lichtkanäle

Die ursprünglichen Jieli-PWM-Verbindungen wurden an R6/R11 getrennt. Die beiden externen Steuerleitungen werden über normale Widerstände an die vorhandenen Transistor-Basisknoten geführt:

```text
ESP32 GPIO4 ── 6,8 kΩ ── WW-Basisknoten (Q2/R30/L1)
ESP32 GPIO5 ── 6,8 kΩ ── KW-Basisknoten (Q1/R29/L2)
```

Die beiden 6,8-kΩ-Widerstände sind wichtig. Ein ESP32-GPIO darf nicht direkt an die Lampen- bzw. Transistorsteuerung angeschlossen werden.

Die gemessenen Widerstände waren:

- WW: 6,81 kΩ
- KW: 6,85 kΩ

Die vorhandenen Pulldown-Widerstände der Lampenschaltung bleiben erhalten.

### Jieli-Versorgungsschalter

Der Jieli wird mit einem PNP/NPN-Gespann auf der 5-V-Seite geschaltet:

```text
                         +5 V Lampe
                            │
                     E  ┌───┴───┐
                        │ BC327 │ PNP
                     C  └───┬───┘
                            │
                       Jieli Pin 12

BC327-Basis ── 1 kΩ ── BC337-Collector
BC327-Basis ── 10 kΩ ── +5 V

BC337-Emitter ───────── GND
BC337-Basis ── 4,7 kΩ ── ESP32 GPIO0
BC337-Basis ── ca. 100 kΩ ── GND
```

Aufgebaut wurden nominal 4,7 kΩ und 100 kΩ; gemessen wurden ungefähr 4,67 kΩ und 91,9 kΩ.

Die Transistoren wurden mit der flachen Seite von vorne betrachtet so verwendet:

```text
links = Emitter   mitte = Base   rechts = Collector
```

Das gilt nur für die hier verwendeten BC327/BC337-Gehäuse und sollte bei anderen Transistoren erneut geprüft werden.

Der ursprüngliche Jieli-Versorgungsweg über den kleinen Widerstand bzw. die Ferrit-/Widerstandsstelle R23/R27 wurde entfernt. Die geschaltete Seite des BC327 geht direkt zum Jieli-Versorgungspunkt an Pin 12. Vor dem endgültigen Einbau wurden etwa 5,1 V auf der Lampenseite und etwa 5,03 V am Jieli gemessen.

### Touch

Der ursprüngliche Touch-Anschluss zum Jieli wurde über R8 getrennt; die ursprüngliche Touch-Spirale ist damit nicht mehr elektrisch mit Pin 11 des Jieli verbunden.

Der TTP223 wird so angeschlossen:

```text
TTP223 VCC ── 3,3 V ESP32
TTP223 GND ── gemeinsamer GND
TTP223 I/O ── ESP32 GPIO1
```

Die TTP223-Platine ist auf `AB = 00` eingestellt: direkter, nicht verriegelnder High-Ausgang.

Der eigentliche Sensoreingang des TTP223 ist der IC-Pin `I` (Pin 3). Auf der verwendeten Platine führt er zum rechten Pad des unbestückten C2. Dort kann eine externe Elektrode angeschlossen werden.

Die originale Touch-Spirale verursachte unterhalb der Lampenplatine Störeinkopplungen und falsche TTP223-Impulse. Daher wurde die Touch-Elektrode an eine Position außerhalb der unmittelbaren Nähe der Lampenplatine verlegt. Falls nötig, sollte zusätzlich ein Kunststoff-Abstandshalter verwendet werden.

## ESPHome-Bedienlogik

Die produktive Firmware ist [`viola_lampe.yaml`](viola_lampe.yaml).

### Touch bei ausgeschalteter Lampe

```text
1. Touch
   → gespeicherten WW/KW-Zustand wiederherstellen

weiterer Touch innerhalb einer Sekunde:
2. Touch → 100 % Kaltweiß
3. Touch → 100 % Neutralweiß (WW und KW jeweils 100 %)
4. Touch → 100 % Warmweiß
```

Nach der Auswahl von Warmweiß ist die Farbwahlfolge beendet.

Ein langer Touch dimmt den aktuell gewählten Zustand. Beide aktiven Kanäle werden proportional reduziert, aber höchstens bis 1 %. Zum Ausschalten wird anschließend ein kurzer separater Touch verwendet.

### Touch bei eingeschalteter Lampe

Wenn das Farbwahlfenster abgelaufen ist:

```text
kurzer Touch → aktuellen WW/KW-Zustand speichern und Lampe ausschalten
langer Touch → aktuelle Farbe dimmen, mindestens bis 1 %
```

### Home Assistant

Die Firmware stellt unter anderem folgende Entitäten bereit:

- `VIOLA Warmweiß`
- `VIOLA Kaltweiß`
- `VIOLA Touch`
- `VIOLA Jieli Versorgung`
- `VIOLA Lampe letzter Zustand`

Der Schalter **VIOLA Lampe letzter Zustand** stellt beim Einschalten die gespeicherten WW/KW-Werte wieder her. Beim Ausschalten speichert er den aktuellen Zustand und schaltet beide Lichtkanäle aus.

Die gespeicherten Werte sind ESPHome-Globals mit `restore_value: yes`. Die ESPHome-Defaults für das Flash-Schreibintervall bleiben absichtlich unverändert, um unnötige Flash-Schreibzyklen zu vermeiden. Eine unmittelbar nach einer Änderung erfolgende Stromunterbrechung kann deshalb theoretisch noch den vorherigen gespeicherten Zustand hinterlassen.

## PWM und Frequenz

Die finale Firmware verwendet:

```yaml
frequency: 12000Hz
```

Bei niedrigeren Frequenzen war ein deutliches Spulen-/Schaltgeräusch hörbar. Mit 12 kHz war das PWM-Piepen praktisch nicht mehr hörbar. Die externe Ansteuerung arbeitet mit 3,3-V-GPIO-Pegeln und den vorgeschalteten 6,8-kΩ-Widerständen.

Die ursprüngliche Messreihe zeigte unter anderem:

- ungefähr 65 kHz bis 68 kHz in bestimmten Betriebszuständen,
- etwa 2,0 kHz in anderen gedimmten Zuständen,
- unterschiedliche Tastverhältnisse für die Dimmstufen.

Für die neue ESP32-Steuerung wird die ursprüngliche Jieli-PWM jedoch nicht mehr übernommen; der ESP32 erzeugt die eigene PWM.

## Bekannte Geräusche und Störungen

Während der Arbeiten wurde ein leises periodisches Pochen bzw. Schaltgeräusch beobachtet. Es trat teilweise auch ohne angeschlossenen ESP32 und bei ausgeschalteter LED auf. Die Touch-Spirale nahm dieses Signal auf, wenn sie direkt unter der bestromten Lampenplatine lag.

Die praktikable Lösung ist:

1. Touch-Elektrode von der Lampenplatine weg positionieren.
2. Leitungen zwischen Elektrode und TTP223 möglichst kurz halten.
3. Bei Bedarf Kunststoff als elektrische Abschirmung/Abstand verwenden.
4. Keine direkte Verbindung zwischen der originalen Touchleitung und dem Jieli wiederherstellen.

Nach Einbau der 6,8-kΩ-GPIO-Widerstände wurden Q1/Q2 nicht mehr heiß, sondern nur noch leicht warm. Bei jedem endgültigen Aufbau sollte nach einigen Minuten Betrieb geprüft werden, ob die Temperatur stabil bleibt und keine Komponente ungewöhnlich heiß wird.

## Flashen

Im Projektverzeichnis wird normalerweise eine eigene `secrets.yaml` mit den WLAN-Zugangsdaten benötigt. Diese Datei gehört nicht ins Repository.

Beispiel:

```bash
cd /home/codex/codex/viola
esphome run viola_lampe.yaml
```

Vor dem Flashen:

- prüfen, dass der richtige ESP32-C3 angeschlossen ist,
- sicherstellen, dass die Lampe nicht versehentlich über einen GPIO direkt gespeist wird,
- kontrollieren, dass die beiden 6,8-kΩ-Widerstände in den PWM-Leitungen vorhanden sind,
- beim ersten Test die Temperatur von Q1/Q2 und der Umgebung beobachten.

## Mess- und Testdateien

Die übrigen YAML-Dateien sind Diagnose- bzw. historische Testkonfigurationen:

| Datei | Zweck |
|---|---|
| `viola_jieli_switch_test.yaml` | Test des GPIO0-Jieli-Versorgungsschalters |
| `viola_pwm_control_test.yaml` | Früher PWM-Steuertest für WW/KW |
| `viola_pwm_measure.yaml` | Frequenz- und Tastgradmessung an GPIO4/GPIO5 |
| `viola_pwm_frequency.yaml` | Frequenzmessung an GPIO4 |
| `viola_pwm_width.yaml` | Messung der HIGH-Pulsbreite an GPIO4 |
| `viola_pwm_average.yaml` | ADC-Mittelwertmessung an GPIO4 |
| `viola_level_measure.yaml` | ADC-Maximalpegelmessung an GPIO4 |
| `viola_pwm_rmt.yaml` | Rohsignal-/Pulsdiagnose über den RMT-Empfänger |

Diese Dateien dürfen nicht mit dem produktiven Aufbau verwechselt werden. Die Messkonfigurationen waren nur zulässig, weil das Lampensignal über geeignete Widerstände bzw. den Spannungsteiler angeschlossen wurde. Ein Lampensignal darf nicht direkt an einen ESP32-GPIO gelegt werden.

## Fotos und Dokumentation

- [`IMG20260928120217.jpg`](IMG20260928120217.jpg) – Platinenaufnahme
- [`IMG20260928120217-markings.jpg`](IMG20260928120217-markings.jpg) – markierte Leiterbahnen, Pads und Bauteile
- [`lochraster-oben.jpg`](lochraster-oben.jpg) – Oberseite der Lochrasterplatine
- [`lochraster-unten.jpg`](lochraster-unten.jpg) – Unterseite der Lochrasterplatine

## Wichtige Designentscheidungen

- Der Jieli wird nicht umprogrammiert oder gehackt; er bleibt für Bluetooth und Audio zuständig.
- USB-D+/D− wurden als Steuerweg verworfen. Die Verbindung am Lampenboard lieferte am Rechner keine zuverlässig nutzbare USB-Kommunikation.
- Der Jieli-Touch wird nicht mehr verwendet. Der TTP223 übernimmt ausschließlich die Touch-Erkennung.
- Ein Logic-Level-Konverter ist für die finale PWM-Ausgabe nicht erforderlich, weil der ESP32 die Lampentransistoren über die externen Widerstände mit 3,3 V ansteuert.
- Die ursprüngliche LED-Leistungselektronik der Lampe bleibt erhalten; der ESP32 ersetzt nur die Steuersignale.

## Offene Punkte

- Die endgültige mechanische Befestigung und elektrische Abschirmung der Touch-Elektrode kann bei Bedarf noch optimiert werden.
- Bei Änderungen an PWM-Frequenz, Widerstandswerten oder Versorgung sollte die Temperatur erneut kontrolliert werden.
- Falls der gespeicherte Zustand unmittelbar nach einer Änderung und einem Stromausfall verloren geht, ist das eine Folge des absichtlich beibehaltenen ESPHome-Flash-Schreibintervalls und kein Hinweis auf einen Fehler der Lichtsteuerung.
