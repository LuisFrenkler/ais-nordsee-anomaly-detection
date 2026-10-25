# AIS Nordsee – Validierung gemeldeter Schiffstypen mit Unsupervised Learning

Im AIS-System (Automatic Identification System) tragen die Besatzungen den Schiffstyp von Hand ein. Position und Geschwindigkeit kommen dagegen automatisch per GPS. Der gemeldete Typ kann also falsch sein, durch Tippfehler, veraltete Einstellungen oder bewusste Täuschung.

**Forschungsfrage:** Lässt sich am tatsächlichen Fahrverhalten erkennen, ob der gemeldete Schiffstyp plausibel ist?

Supervised Learning scheidet aus, weil genau die Labels (die gemeldeten Typen) angezweifelt werden. Die Modelle arbeiten deshalb rein unsupervised. Der gemeldete Typ fließt nie als Feature ein, sondern dient erst am Ende als unabhängige Referenz zur Validierung.

## Daten

| Datei | Inhalt |
|---|---|
| `aisdk_2026_nordsee_500k.csv` | 500.000 AIS-Signale aus der Nordsee und westlichen Ostsee (01.01.2026, ca. 1 Stunde), Quelle: dänische Seefahrtsbehörde (AIS-DK) |
| `nordseehaefen.csv` | Koordinaten großer Nordseehäfen (OpenStreetMap / Wikipedia), zur Anreicherung der Features |

## Vorgehen

1. **Cleaning:** ungültige GPS-Werte (Latitude = 91.0) und Positionen außerhalb des Gebiets entfernen, Schiffstypen auf sechs Kategorien abbilden (Cargo, Tanker, Fishing, Passenger, Sailing/Pleasure, Tug/Towing).
2. **Aggregation pro Schiff (MMSI):** Alle Signale eines Schiffs werden zu einem Datenpunkt zusammengefasst. Das ergibt 1.394 Schiffe mit 16 Features.
3. **Features:** Abmessungen (Länge, Breite, Tiefgang), Geschwindigkeitsstatistiken (SOG), Kursvariabilität (COG), Anteile der Navigationsstatus, Position und Streuung, dazu die Distanz zum nächsten Hafen per Haversine-Formel.
4. **Drei Methoden zur Anomalie-Erkennung:**
   - **K-Means** (k = 6): Ausreißer über die Distanz zum Clusterzentrum, Schwellwert nach IQR-Regel
   - **DBSCAN** (Grid Search über eps und min_samples): Noise-Punkte gelten als Ausreißer
   - **Isolation Forest** (Hyperparameter-Tuning): Schwellwert nach IQR-Regel auf den Anomaly Scores
5. **Consensus:** Ein Schiff gilt als verdächtig, wenn mindestens zwei der drei Methoden anschlagen.
6. **Validierung:** Kreuztabelle Cluster vs. gemeldeter Typ, Plausibilitätschecks, Sensitivitätsanalyse des IQR-Faktors und Vergleich mit vs. ohne Hafendaten.

## Ergebnisse

| Methode | Silhouette | Anomalien |
|---|---|---|
| K-Means (k = 6) | 0,257 | 48 (3,4 %) |
| DBSCAN | 0,343 | 23 (1,6 %) |
| Isolation Forest | – | 58 (4,2 %) |

- **44 Schiffe** werden von mindestens zwei Methoden markiert, 14 davon von allen drei.
- Bei **25** dieser 44 Schiffe passt der gemeldete Typ nicht zum dominanten Typ ihres Clusters. **11** davon liegen in eindeutigen Clustern (mindestens 70 % ein Typ). Das sind die stärksten Verdachtsfälle.
- K-Means findet allein über das Verhalten einen Cluster, der zu **100 % aus Fischereischiffen** besteht.
- Die wichtigsten Merkmale für Anomalien sind Geschwindigkeit, Tiefgang und Schiffsgröße. Die externen Hafendaten bringen kaum etwas (Rang 14 und 16 von 16).

**Grenzen:** Die Daten decken nur rund eine Stunde am Neujahrstag ab, und über die Hälfte der Schiffe steht still. Die Verdachtsfälle sind daher Hinweise, keine Beweise für Falschmeldungen.

## Ausführen

Benötigt Python 3.11 oder neuer.

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
jupyter notebook AIS_Nordsee_Unsupervised_Final.ipynb
```

Die beiden CSV-Dateien müssen im selben Ordner wie das Notebook liegen. Ein kompletter Durchlauf dauert etwa zwei Minuten.

## Dateien

- `AIS_Nordsee_Unsupervised_Final.ipynb` – Notebook mit der kompletten Analyse
- `requirements.txt` – benötigte Python-Pakete
