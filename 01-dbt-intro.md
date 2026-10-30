# Einführung in dbt: Lokale Postgres-Pipeline bauen

Diese Anleitung zeigt, wie du von rohen NYC-Taxi-Fahrtdaten zu einem funktionierenden **dbt**-Projekt auf Basis von PostgreSQL kommst. Am Ende hast du eine Pipeline, die du so nutzen kannst:

1. mit `dbt debug` ansprechen,
2. mit `dbt seed` befüllen,
3. mit `dbt run` bauen,
4. mit `dbt test` validieren,
5. mit `dbt docs generate` dokumentieren.

## Was ist dbt?

**dbt (data build tool)** ist ein Transformations-Framework für Analytics Engineering. Es setzt auf deinem Data Warehouse auf, und du definierst damit Transformationen als versionierte SQL-Modelle.

dbt ist besonders nützlich in Workflows mit **ELT (Extract, Load, Transform)**:

- Rohdaten werden zuerst ins Warehouse geladen.
- dbt transformiert diese Rohdaten in sauberere, nützlichere Modelle.
- Tests und Dokumentation liegen neben dem SQL, das die Pipeline definiert.

```mermaid
flowchart LR
    subgraph ELT
        elt1["Extrahieren"] --> elt2["Rohdaten ins Warehouse laden"] --> elt3["Im Warehouse mit dbt transformieren"]
    end
```

dbt ist beliebt, denn damit lassen sich Transformationen im Warehouse leicht:

- in Git prüfen,
- testen und dokumentieren,
- in wiederverwendbare Schichten gliedern,
- in verschiedenen Umgebungen wie `dev` und `prod` ausführen.

## Wie dbt in eine Pipeline passt

Ein dbt-Projekt modelliert Daten meist in Schichten.

| Schicht | Zweck |
| --- | --- |
| **Quelle / Rohdaten** | Von außen geladene Daten |
| **Staging** | Leichte Bereinigung und Umbenennung von Spalten |
| **Intermediate** | Wiederverwendbare Geschäftslogik oder Aggregationen |
| **Mart** | Ergebnisse für Stakeholder |

```mermaid
flowchart LR
    raw["public.yellow_taxi"] --> stg["stg_yellow_taxi"]
    stg --> int1["int_avg_tips"]
    stg --> int2["int_avg_tips_per_payment_type"]
    seed["Seed payment_type"] --> int2
    int1 --> mart["fct_best_tips"]
    int2 --> mart
```

In diesem Tutorial nutzen wir **dbt Core** mit dem Postgres-Adapter `dbt-postgres`. Wenn du später verwaltete Orchestrierung erkunden willst, sieh dir [dbt Cloud](https://www.getdbt.com/) an.

## Bevor du anfängst

Erstell im Stammverzeichnis dieses Repositorys einen neuen Ordner für diese Anleitung:

```bash
mkdir dbt-intro
cd dbt-intro
```

Weil `dbt-intro/` im Repository liegt, findet `uv run` die Umgebung des Repositorys automatisch. So kann jeder `python`- und `dbt`-Befehl in diesem Tutorial über `uv run` laufen, ohne dass du etwas von Hand aktivierst, und zwar auf dieselbe Weise unter macOS, Windows und Linux. Außerdem verändert das Tutorial so nichts Bestehendes, und du kannst deine Ergebnisse nach den Schritten unten leichter mit der Musterlösung vergleichen.

Prüf, bevor es weitergeht, ob Docker läuft:

```bash
docker version
```

Wenn Docker noch nicht läuft, starte Docker Desktop oder deinen lokalen Docker-Daemon, bevor du weitermachst.

## 1. PostgreSQL in Docker starten

Starte einen lokalen Postgres-Container:

```bash
docker run -d \
  --name ny-taxi-db \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=ny_taxi \
  -p 5432:5432 \
  postgres:17
```

Erwartetes Ergebnis:

- Docker gibt eine lange Container-ID aus,
- ein Container namens `ny-taxi-db` läuft jetzt.

Warte, bis die Datenbank bereit ist:

```bash
wait_for_postgres() {
  local deadline=$((SECONDS + 60))
  while (( SECONDS < deadline )); do
    if docker exec ny-taxi-db pg_isready -t 1 -U postgres -d ny_taxi; then
      return 0
    fi
    sleep 1
  done
  printf '%s\n' 'PostgreSQL was not ready within 60 seconds. Check: docker logs ny-taxi-db' >&2
  return 1
}
wait_for_postgres
```

Daran erkennst du den Erfolg:

- die Schleife endet von selbst,
- Postgres gibt `accepting connections` aus.

Wenn Port `5432` auf deinem Rechner schon belegt ist, ordne einen anderen Host-Port zu, etwa `5433:5432`, und nutz dann genau diesen Host-Port überall in diesem Tutorial:

- im Python-Bootstrap-Skript unten,
- in `~/.dbt/profiles.yml`.

## 2. Rohe Quelltabelle herunterladen und laden

Erstell einen kleinen Datenordner und lade die Parquet-Datei für Januar 2025 herunter:

```bash
mkdir -p data
curl -L \
  -o data/yellow_tripdata_2025-01.parquet \
  https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2025-01.parquet
```

Lade diese Parquet-Datei jetzt in `public.yellow_taxi` in Postgres:

```bash
uv run python - <<'PY'
import pyarrow.parquet as pq
from time import perf_counter
from sqlalchemy import create_engine, text

engine = create_engine(
    "postgresql+psycopg2://postgres:postgres@localhost:5432/ny_taxi"
)
parquet_file = pq.ParquetFile("data/yellow_tripdata_2025-01.parquet")
expected_rows = parquet_file.metadata.num_rows
started_at = perf_counter()
loaded_rows = 0
print(f"Loading {expected_rows:,} rows into public.yellow_taxi", flush=True)

with engine.begin() as conn:
    conn.execute(text("drop table if exists public.yellow_taxi"))

first_batch = True
for batch_number, batch in enumerate(parquet_file.iter_batches(batch_size=50000), start=1):
    batch_started_at = perf_counter()
    df = batch.to_pandas()
    df.to_sql(
        "yellow_taxi",
        engine,
        schema="public",
        if_exists="replace" if first_batch else "append",
        index=False,
        method="multi",
        chunksize=5000,
    )
    first_batch = False
    loaded_rows += len(df)
    print(
        f"Batch {batch_number}: {loaded_rows:,}/{expected_rows:,} rows loaded; "
        f"batch {perf_counter() - batch_started_at:.1f}s, "
        f"elapsed {perf_counter() - started_at:.1f}s",
        flush=True,
    )

with engine.begin() as conn:
    row_count = conn.execute(
        text("select count(*) from public.yellow_taxi")
    ).scalar_one()

assert row_count == expected_rows, (
    f"Row count mismatch: Parquet contains {expected_rows:,} rows, "
    f"but PostgreSQL contains {row_count:,}."
)
print(f"Loaded {row_count} rows into public.yellow_taxi; row-count validation passed.")
engine.dispose()
PY
```

Dieser Ladeschritt kann auf einem Laptop eine Weile dauern. Jeder abgeschlossene Batch meldet den Fortschritt und die bisher vergangene Zeit. Die abschließende Assertion vergleicht die Zeilenzahl in der Datenbank mit den Parquet-Metadaten; dass das Einfügen abgeschlossen ist, beweist allein noch nicht, dass alles vollständig geladen wurde.

Wenn du die komplette Datei für Januar 2025 lädst, sollte die Zeilenzahl **3.475.226** betragen.

Daran erkennst du den Erfolg:

- das Python-Skript endet ohne Exception,
- es gibt `Loaded 3475226 rows into public.yellow_taxi` aus.

Du kannst die Zeilenzahl auch direkt über Docker prüfen:

```bash
docker exec -it ny-taxi-db \
  psql -U postgres -d ny_taxi \
  -c "select count(*) as row_count from public.yellow_taxi;"
```

### Erwartete Quelltabelle

Bevor du mit dbt anfängst, stell sicher, dass die Quelltabelle `public.yellow_taxi` existiert und diese Spalten enthält:

```text
VendorID
tpep_pickup_datetime
tpep_dropoff_datetime
passenger_count
trip_distance
RatecodeID
store_and_fwd_flag
PULocationID
DOLocationID
payment_type
fare_amount
extra
mta_tax
tip_amount
tolls_amount
improvement_surcharge
total_amount
congestion_surcharge
Airport_fee
cbd_congestion_fee
```

Dafür kannst du Folgendes ausführen:

```bash
docker exec ny-taxi-db \
  psql -U postgres -d ny_taxi \
  -c "\d public.yellow_taxi"
```

Genau dieses Schema erwartet das Staging-Modell weiter unten.

## 3. dbt-Projekt initialisieren

Erstell ein neues dbt-Projekt (ohne die interaktiven Eingabeaufforderungen für das Profil):

```bash
uv run dbt init dbt_intro --skip-profile-setup
```

Das erstellt:

- einen Projektordner namens `dbt_intro`,
- eine Projektkonfigurationsdatei namens `dbt_project.yml`,
- die üblichen Startordner und Beispielmodelle.

Wechsle in den Projektordner:

```bash
cd dbt_intro
```

## 4. Startbeispiel aufräumen

`dbt init` erstellt Startmodelle als Beispiel. Wir ersetzen sie durch unsere eigenen Modelle für das Tutorial.

Lösch den Beispielordner:

```bash
rm -rf models/example
```

Öffne dann `dbt_project.yml` in einem Editor deiner Wahl und entferne den Block `example:` unter `models.dbt_intro`:

```yaml
models:
  dbt_intro:
    example:
      +materialized: view
```

Die Kommentare drumherum können bleiben. Wenn du dieses Aufräumen überspringst, funktioniert dbt trotzdem, gibt aber eine Warnung zu ungenutzter Konfiguration aus.

## 5. `profiles.yml` erstellen

Öffne `~/.dbt/profiles.yml` in deinem bevorzugten Editor. Wenn du ein Beispiel für die Kommandozeile willst:

```bash
nano ~/.dbt/profiles.yml
```

Oder in VS Code:

```bash
code ~/.dbt/profiles.yml
```

Wenn die Datei noch nicht existiert, erstell sie. Stell dann sicher, dass sie so aussieht:

```yaml
dbt_intro:
  outputs:

    dev:
      dbname: ny_taxi
      host: localhost
      pass: postgres
      port: 5432
      schema: dbt_dev
      threads: 4
      type: postgres
      user: postgres

    prod:
      dbname: ny_taxi
      host: localhost
      pass: postgres
      port: 5432
      schema: dbt_prod
      threads: 4
      type: postgres
      user: postgres

  target: dev
```

Dieses explizite, dateibasierte Setup lässt sich leichter kopieren, prüfen und auf Fehler untersuchen, als wenn du die Werte über die Eingabeaufforderungen von `dbt init` eingibst.

### Warum diese Schemanamen wichtig sind

dbt nutzt das Zielschema als Präfix, wenn du in einem Modell zusätzlich ein eigenes Schema festlegst. Das heißt:

- ein Modell mit `schema='stg'` landet in `dbt_dev_stg`,
- ein Modell mit `schema='int'` landet in `dbt_dev_int`,
- ein Modell mit `schema='driver'` landet in `dbt_dev_driver`.

Das ist das Standardverhalten und so zu erwarten.

## 6. Verbindung testen

Führ Folgendes aus:

```bash
uv run dbt debug
```

Du solltest Folgendes sehen:

- `profiles.yml file [OK found and valid]`
- `dbt_project.yml file [OK found and valid]`
- `Connection test: [OK connection ok]`
- `All checks passed!`

Daran erkennst du den Erfolg:

- dbt kann sowohl `dbt_project.yml` als auch `~/.dbt/profiles.yml` lesen,
- die Postgres-Verbindung funktioniert, ohne dass du sonst etwas ändern musst.

Es sollte ungefähr so aussehen:

![dbt_debug_success](assets/01-dbt_debug.png)

## 7. Projektstruktur verstehen

Im Ordner des dbt-Projekts siehst du diese Hauptverzeichnisse:

- [dbt_project.yml](https://docs.getdbt.com/reference/dbt_project.yml): Konfiguration auf Projektebene
- [models](https://docs.getdbt.com/docs/build/models): SQL-Modelle, die dbt in Tabellen oder Views umwandelt
- [seeds](https://docs.getdbt.com/docs/build/seeds): CSV-Dateien, die dbt ins Warehouse lädt
- [tests](https://docs.getdbt.com/docs/build/data-tests): Eigene SQL-Tests
- [snapshots](https://docs.getdbt.com/docs/build/snapshots): Historie für langsam veränderliche Daten
- [macros](https://docs.getdbt.com/docs/build/jinja-macros): Wiederverwendbare Jinja-Logik
- [analyses](https://docs.getdbt.com/docs/build/analyses): Analytisches Ad-hoc-SQL, das versioniert, aber nicht als Modell materialisiert wird

Zentrale Einstellungen in `dbt_project.yml`, die du kennen solltest:

- `name`: Name des Projekts
- `profile`: Welches Profil aus `profiles.yml` genutzt wird
- `model-paths`, `seed-paths`, `snapshot-paths`, `macro-paths`: Wo dbt nach Ressourcen sucht
- `clean-targets`: Verzeichnisse, die `dbt clean` entfernt

## 8. Staging-Schicht erstellen

Erstell den Ordner:

```bash
mkdir -p models/staging
```

Definiere zuerst die Quelle in `models/staging/src_yellow_taxi.yml`:

```yaml
version: 2

sources:
  - name: src_yellow_taxi
    schema: public
    tables:
      - name: yellow_taxi
```

Das sagt dbt: „Meine Rohdaten liegen in `public.yellow_taxi`.“

Erstell jetzt `models/staging/stg_yellow_taxi.sql`:

```sql
{{ config(materialized='table', schema='stg') }}

select
    "VendorID" as vendor_id,
    tpep_pickup_datetime as pickup_datetime,
    tpep_dropoff_datetime as dropoff_datetime,
    passenger_count,
    trip_distance,
    "RatecodeID" as rate_code_id,
    store_and_fwd_flag,
    "PULocationID" as pickup_location_id,
    "DOLocationID" as dropoff_location_id,
    payment_type,
    fare_amount,
    extra,
    mta_tax,
    tip_amount,
    tolls_amount,
    improvement_surcharge,
    total_amount,
    congestion_surcharge,
    "Airport_fee" as airport_fee,
    cbd_congestion_fee
from {{ source('src_yellow_taxi', 'yellow_taxi') }}
```

Dieses Modell bereinigt die Daten nur leicht:

- es liest direkt aus der Rohdatenquelle,
- es benennt Spalten mit gemischter Groß- und Kleinschreibung in `snake_case` um,
- es wendet noch keine Geschäftslogik an.

Wenn du die Runs in der Entwicklung später begrenzen willst, kannst du Folgendes hinzufügen:

```sql
{% if target.name == 'dev' %}
limit 100
{% endif %}
```

## 9. Seed-Datei hinzufügen

Seeds sind kleine CSV-Dateien, die dbt für Lookups und Referenzdaten ins Warehouse laden kann.

Erstell `seeds/payment_type.csv`:

```csv
id,payment_type
1,Credit Card
2,Cash
3,No Charge
4,Dispute
5,Unknown
6,Voided Trip
```

Mit diesem Lookup machen wir aus den Codes der Zahlungsarten lesbare Labels.

## 10. Intermediate-Modelle erstellen

Erstell den Intermediate-Ordner:

```bash
mkdir -p models/intermediate
```

Erstell `models/intermediate/int_avg_tips_per_payment_type.sql`:

```sql
{{ config(materialized='table', schema='int') }}

with avg_tips as (
    select
        payment_type,
        avg(tip_amount) as avg_tip_amount
    from {{ ref('stg_yellow_taxi') }}
    where payment_type between 1 and 6
    group by 1
),
payment_type_lookup as (
    select
        id as payment_type_id,
        payment_type
    from {{ ref('payment_type') }}
)

select
    payment_type_lookup.payment_type,
    avg_tips.avg_tip_amount
from avg_tips
join payment_type_lookup
    on payment_type_lookup.payment_type_id = avg_tips.payment_type
```

Warum filtern wir auf `payment_type between 1 and 6`?

- Im Rohdatensatz kommt `payment_type = 0` vor.
- Unsere Seed-Datei bildet absichtlich nur `1` bis `6` ab.
- Mit dem Filter an dieser Stelle passt die Logik des Tutorials zur Lookup-Tabelle.

Erstell `models/intermediate/int_avg_tips.sql`:

```sql
{{ config(materialized='table', schema='int') }}

select
    avg(tip_amount) as avg_tip_amount
from {{ ref('stg_yellow_taxi') }}
where payment_type between 1 and 6
```

Dieses Modell berechnet das durchschnittliche Trinkgeld insgesamt über die bekannten Zahlungsarten.

## 11. Mart-Modell erstellen

Erstell den Mart-Ordner:

```bash
mkdir -p models/mart/driver
```

Erstell `models/mart/driver/fct_best_tips.sql`:

```sql
{{ config(materialized='table', schema='driver') }}

with avg_tip as (
    select
        avg_tip_amount
    from {{ ref('int_avg_tips') }}
)

select
    payment_type,
    avg_tip_amount,
    case
        when avg_tip_amount < (select avg_tip_amount from avg_tip) then 'bad tip'
        when avg_tip_amount > (select avg_tip_amount from avg_tip) then 'good tip'
        else 'average tip'
    end as tip_status
from {{ ref('int_avg_tips_per_payment_type') }}
```

Dieses Modell vergleicht das durchschnittliche Trinkgeld pro Zahlungsmethode mit dem durchschnittlichen Trinkgeld insgesamt und kennzeichnet das Ergebnis.

### Abhängigkeitsgraph der Modelle

```mermaid
flowchart LR
    src["source('src_yellow_taxi', 'yellow_taxi')"] --> stg["stg_yellow_taxi"]
    stg --> intAvg["int_avg_tips"]
    stg --> intByType["int_avg_tips_per_payment_type"]
    seed["Seed payment_type"] --> intByType
    intAvg --> mart["fct_best_tips"]
    intByType --> mart
```

## 12. Projekt ausführen

Jetzt, wo die Projektdateien bereitliegen, führ die Pipeline der Reihe nach aus.

### Seed laden

```bash
uv run dbt seed
```

Erwartetes Ergebnis:

- dbt lädt `payment_type.csv`,
- die Tabelle landet im Zielschema, in `dev` wird sie also zu `dbt_dev.payment_type`.

### Alle Modelle bauen

```bash
uv run dbt run
```

Erwartetes Ergebnis in `dev`:

- `dbt_dev_stg.stg_yellow_taxi`
- `dbt_dev_int.int_avg_tips`
- `dbt_dev_int.int_avg_tips_per_payment_type`
- `dbt_dev_driver.fct_best_tips`

Daran erkennst du den Erfolg:

- alle vier Modelle enden mit `OK`,
- dbt endet mit `Completed successfully`.

### Nur die Staging-Schicht ausführen

```bash
uv run dbt run --select path:models/staging
```

### Nur Staging- und Intermediate-Schicht ausführen

```bash
uv run dbt run --select path:models/staging path:models/intermediate
```

## 13. Realistische Tests hinzufügen

Generische Tests in dbt stehen in einer YAML-Datei für das Schema. Erstell `models/staging/staging_schema.yml`:

```yaml
version: 2

models:
  - name: stg_yellow_taxi
    description: Staging model for January 2025 yellow taxi trips.
    columns:
      - name: vendor_id
        description: Vendor identifier from the source data.
        data_tests:
          - not_null
      - name: pickup_datetime
        description: Pickup timestamp.
        data_tests:
          - not_null
      - name: dropoff_datetime
        description: Dropoff timestamp.
        data_tests:
          - not_null
      - name: trip_distance
        description: Trip distance in miles.
        data_tests:
          - not_null
      - name: store_and_fwd_flag
        description: Whether the trip record was held in vehicle memory.
        data_tests:
          - accepted_values:
              arguments:
                values: ['Y', 'N']
      - name: payment_type
        description: Payment type code.
        data_tests:
          - not_null
          - relationships:
              arguments:
                to: ref('payment_type')
                field: id
              config:
                where: "payment_type between 1 and 6"
      - name: fare_amount
        description: Metered fare amount.
        data_tests:
          - not_null
      - name: total_amount
        description: Final charged amount.
        data_tests:
          - not_null

seeds:
  - name: payment_type
    columns:
      - name: id
        data_tests:
          - not_null
          - unique
      - name: payment_type
        data_tests:
          - not_null
          - unique
```

Diese Tests sind bewusst realistisch für Daten von Taxifahrten.

Wir nutzen `unique` **nicht** für Staging-Felder wie `vendor_id`, `rate_code_id`, `pickup_location_id`, `dropoff_location_id` oder `payment_type`, weil sich diese Werte über viele Fahrten hinweg wiederholen und der Test sofort fehlschlagen würde.

Beim Seed ist es anders: Jede ID muss genau ein eindeutiges Label haben. Seine vier Tests verhindern fehlende oder doppelte Lookup-Werte. Der [relationships-Test](https://docs.getdbt.com/reference/resource-properties/data-tests#relationships) prüft, dass jeder Zahlungscode aus dem Staging, den die Intermediate-Modelle behalten (1 bis 6), im Seed existiert. So verwirft der Join mit dem Lookup diese Codes nicht unbemerkt; eindeutige Seed-IDs verhindern, dass er Treffer vervielfacht. Codes außerhalb dieses Bereichs bleiben absichtlich ausgeschlossen. Insgesamt sind es 13 Tests.

Führ die Tests aus:

```bash
uv run dbt test
```

Erwartetes Ergebnis:

- alle konfigurierten Tests bestehen,
- dbt meldet jeweils `PASS` und endet mit `Completed successfully`.

## 14. Dokumentation generieren

dbt kann aus den Metadaten deiner Modelle eine durchsuchbare Projektdoku machen.

Generiere die Artefakte:

```bash
uv run dbt docs generate
```

Das schreibt Artefakte der Dokumentation wie `manifest.json` und `catalog.json` ins Verzeichnis `target/`.

Stell die Doku lokal bereit:

```bash
uv run dbt docs serve
```

Drück im Terminal Ctrl+C, um den Server zu beenden.

## 15. Checkliste zur Überprüfung

Nutz diese Checkliste am Ende des Tutorials:

1. `docker exec -it ny-taxi-db psql -U postgres -d ny_taxi -c "select count(*) from public.yellow_taxi;"` gibt die erwartete Zeilenzahl zurück.
2. `uv run dbt debug` endet mit `All checks passed!`.
3. `uv run dbt seed` lädt `dbt_dev.payment_type`.
4. `uv run dbt run` baut die Staging-, Intermediate- und Mart-Modelle.
5. `uv run dbt test` endet damit, dass alle Tests bestehen.
6. `uv run dbt docs generate` schreibt die Artefakte der Doku nach `target/`.

Wenn alle sechs Prüfungen erfolgreich sind, funktioniert deine lokale dbt-Pipeline von Anfang bis Ende.
