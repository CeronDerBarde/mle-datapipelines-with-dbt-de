# Datenpipelines mit dbt

Dieses Repository ist eine praktische Einführung in **dbt (data build tool)**. Du baust eine kleine Analysepipeline auf Basis der NYC-Yellow-Taxi-Fahrtdaten: zuerst auf einer lokalen PostgreSQL-Datenbank, dann in ein Docker-Image verpackt und optional auf DuckDB als schlanke alternative Laufzeitumgebung.

## Lernziele

Am Ende dieses Repositorys kannst du:

- ein einsteigerfreundliches dbt-Projekt auf Basis von Postgres bauen.
- dasselbe dbt-Projekt in Docker ausführen.
- dasselbe Projekt mit nur kleinen Änderungen an der Konfiguration auf DuckDB umstellen.
- verstehen, wie Staging-, Intermediate- und Mart-Schicht in einer einfachen Analysepipeline zusammenspielen.

## Lernpfad

| Datei / Ordner | Beschreibung |
|---|---|
| [**01 - Einführung in dbt**](01-dbt-intro.md) | Ein dbt-Projekt auf Basis von Postgres bauen und testen. |
| [**02 - dbt in Docker verpacken**](02-dockerize-dbt.md) | Dasselbe Projekt in Docker mit Postgres ausführen. |
| [**03 - DuckDB (optional)**](03-OPTIONAL-duckdb.md) | Dasselbe Projekt mit DuckDB ausführen. |

### Weitere Ordner und Dateien

| Datei / Ordner | Beschreibung |
|---|---|
| [**solutions**](solutions/) | Referenzstände am Ende jedes Kapitels. |
| [**assets**](assets/) | Abbildungen, auf die das Repository verweist. |
| [**pyproject.toml**](pyproject.toml) | Projektkonfiguration und Abhängigkeiten. |
| [**uv.lock**](uv.lock) | Lock-Datei für die Abhängigkeiten. |

## Voraussetzungen

- **Docker Desktop**: nötig, um den lokalen PostgreSQL-Container auszuführen, den der ganze Kurs nutzt. Folg der [Installationsanleitung](https://docs.docker.com/get-docker/), falls du es noch nicht hast. Stell sicher, dass es **installiert ist und läuft**, bevor du mit der Anleitung beginnst.

> [!NOTE]
> **Unter Windows:** Die Befehle in dieser Anleitung nutzen Bash-Syntax. Führ sie mit **Git Bash** aus (installiert mit [Git for Windows](https://git-scm.com/downloads/win)), nicht mit PowerShell.

## Setup

Verwende [uv](https://docs.astral.sh/uv/getting-started/installation/) 0.13.5 oder neuer
und Python 3.13.17 oder ein neueres Patch-Release von 3.13, wie in
[`.python-version`](.python-version) festgelegt.

Bau vorhandene Docker-Images nach diesem Update mit `--pull --no-cache` neu,
um ihre Systemzertifikate und Abhängigkeiten zu aktualisieren.

> [!NOTE]
> In allen Schritten ist Text in spitzen Klammern wie `<repo-name>` ein **Platzhalter**. Ersetze ihn samt den Klammern `< >` durch deinen eigenen Wert. Aus `cd <repo-name>` wird zum Beispiel `cd mle-datapipelines-with-dbt`.

### 1. Repository als Template verwenden

Klick auf GitHub auf **Use this template**.

Beim Erstellen des Repositorys:

- Leg dich selbst als **Owner** fest
- Wähl einen Namen für das Repository
- Deaktiviere **Include all branches**
- Klick auf **Create repository**

> [!IMPORTANT]
> Wenn du zu zweit oder in einer Gruppe arbeitest, erledigt nur **eine Person** diesen Schritt.

---

### 2. Collaborators hinzufügen (nur bei Paar- oder Gruppenarbeit)

Wenn du mit Teammitgliedern arbeitest:

1. Öffne das Repository auf GitHub
2. Geh zu **Settings → Collaborators**
3. Füg deine Teammitglieder als Collaborators hinzu
4. Teil den Link zum Repository mit deinem Team

Die Teammitglieder sollten die Einladung annehmen, bevor sie weitermachen.

---

### 3. Repository klonen

Kopier die SSH-URL über den Button **Code** auf GitHub und führ dann Folgendes aus:

```bash
git clone <copied-ssh-url>
```

Die kopierte SSH-URL hat die Form `git@github.com:<your-username>/<repo-name>.git`.

---

### 4. In den Projektordner wechseln und Abhängigkeiten installieren

Das installiert alle Abhängigkeiten und erstellt eine virtuelle Umgebung in `.venv/`.

```bash
cd <repo-name>
uv sync
```

---

### 5. Repository in VS Code öffnen

> [!NOTE]
> Achte darauf, VS Code im Stammverzeichnis des Projekts zu öffnen, damit es die von `uv sync` erstellte Umgebung automatisch erkennt.

Starte VS Code im Stammverzeichnis des Projekts:

```bash
code .
```

Jeder Befehl in der Anleitung läuft über `uv run`, deshalb wird diese Umgebung automatisch genutzt.

## Anleitung

Die Hauptanleitung in [**01-dbt-intro**](./01-dbt-intro.md) ist in sich abgeschlossen. Sie zeigt dir, wie du:

1. einen lokalen Postgres-Container startest.
2. die Yellow-Taxi-Quelldaten von Januar 2025 in `public.yellow_taxi` lädst.
3. auf Basis dieser Quelle ein dbt-Projekt baust, testest und dokumentierst.

Betrachte Kapitel 01 als Source of Truth für das erste Setup. Die Kapitel 02 und 03 setzen beide voraus, dass du diesen Workspace schon fertig hast; Kapitel 03 ist dabei optional.

## Lösungen

Die Endstände der Kapitel findest du in [**solutions**](./solutions/README.md):

- [**solutions/01-dbt-intro**](./solutions/01-dbt-intro): Endstand von Kapitel 01
- [**solutions/02-dockerize-dbt**](./solutions/02-dockerize-dbt): Endstand von Kapitel 02
- [**solutions/03-OPTIONAL-duckdb**](./solutions/03-OPTIONAL-duckdb): Optionaler Endstand von Kapitel 03

Nutz sie als Referenzstände, wenn du deine Arbeit mit einer validierten Version des Projekts vergleichen willst. Sie ergänzen die Tutorials, sollen aber die Schritt-für-Schritt-Anleitungen nicht ersetzen.
