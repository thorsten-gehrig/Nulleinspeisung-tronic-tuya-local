# Nulleinspeisung-tronic-tuya-local
Diese Home assistant Regelung versucht den Tronic-speicher (Lidl B2500) so zu regeln das er nur sinnvolle Öeistung abgibt

Für diese Steuerung wird https://github.com/Maztah/tronic-speicher-tuya-local verwendet.
Es wird ein Tasmota IR-Auge (e.g. Hichi) zur Ermittlung des aktuellen Strombezugs (bedarfs) benötigt. Dieser wird mittels Helfer / Linearer Durchschnitt (Stichprobe 200, maximalalter 5min) normalisiert als sensor.lesekopf_2_min_mittelwert_stromzahler

Die Steuerung versucht einen Batteriespeicher zu zu regeln das
a) nur Tagsüber (nach Sonnenaufgang und vor Sonnenuntergang) und wenn der SoC >5% ist - alle 2 minuten angepasst wird
b) bei Einspeisung wird der Akku auf 80W und "zuerst Laden" gestellt. Das verhindert das der Akku Leistung abgibt
c) zwischen 80-120 Watt "Bedarf" wird nicht regegelt
d) bei Einpeisung wird ein "step down" gemacht - in 50W schritten (und vielfachen)
e) bei Bedarf wird ein "step up" gemacht - in 100W schritten (und vielfachen)

Als Ergebniss sollte die Leistung nur abgegeben Werden wenn >100 Watt benötigt wird. Es gibt also keine "Nulleinspeisung", wird aber verhindert das Strom eingespeist wird.
Sobald der Akku aif 100% ist wird natürlich alles von der PV&Batterie abgegeben - so das es in diesem Fall schon zu Einspeisungen kommen wird.
