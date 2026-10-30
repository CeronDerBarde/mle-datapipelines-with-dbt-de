# Lösungen

Dieser Ordner enthält funktionierende Referenzstände für jedes Kapitel des Tutorials.

Jeder Ordner zeigt den erwarteten Endstand, nachdem du das jeweilige Kapitel abgeschlossen hast:

- `01-dbt-intro/`: Das validierte dbt-Projekt auf Basis von Postgres aus Kapitel 01
- `02-dockerize-dbt/`: Dasselbe Projekt plus die Docker-Dateien aus Kapitel 02
- `03-OPTIONAL-duckdb/`: Dasselbe Projekt, konfiguriert für die optionale DuckDB-Laufzeitumgebung aus Kapitel 03

So nutzt du diese Referenzstände:

- Jede Datei `profiles.yml` auf oberster Ebene ist ein Beispiel dafür, was in `~/.dbt/profiles.yml` stehen sollte.
- Die Datei `dbt_prod/profiles.yml` in Kapitel 02 ist das Profil speziell für den Container, das ins Docker-Image kopiert wird.
- Diese Stände dienen als Referenz und ersetzen nicht die Anleitungen der Tutorials.

Was absichtlich fehlt:

- generierte Artefakte wie `target/` und `logs/`,
- virtuelle Umgebungen,
- heruntergeladene Parquet-Dateien,
- die Postgres-Quelltabelle aus den Kapiteln 01 und 02.
- die Datenbankdatei `ny_taxi.duckdb` aus Kapitel 03.

So bleiben die Lösungsordner schlank und zeigen trotzdem genau die von Hand geschriebenen Dateien, mit denen das Projekt funktioniert.
