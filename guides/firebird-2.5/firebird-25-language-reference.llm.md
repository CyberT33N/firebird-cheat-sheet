
# Firebird 2.5 SQL Language Reference (LLM Summary)

Dieses Dokument fasst die SQL-Sprachelemente von Firebird 2.5 zusammen, basierend auf der "Firebird 2.5 Language Reference". Es ist als Referenz für Large Language Models (LLMs) gedacht.

## SQL Language Structure

### Background to Firebird’s SQL Language

#### SQL Flavours
Firebird unterstützt verschiedene SQL-Subsets:
*   **Dynamic SQL (DSQL):** Hauptteil der Sprache für Client-Anwendungen via API. Entspricht SQL/Foundation.
*   **Procedural SQL (PSQL):** Erweiterung von DSQL um prozedurale Elemente (Variablen, Bedingungen, Schleifen) für Stored Procedures, Triggers, Functions (ab FB3) und EXECUTE BLOCK. Entspricht SQL/PSM.
*   **Embedded SQL (ESQL):** Subset von DSQL für die Einbettung in Host-Programmiersprachen mittels `gpre`. Nicht vollständig in dieser Referenz beschrieben.
*   **Interactive SQL (ISQL):** Sprache des Kommandozeilen-Tools `isql`. Basiert auf DSQL mit zusätzlichen `isql`-spezifischen Befehlen. Nicht vollständig in dieser Referenz beschrieben.

Diese Referenz konzentriert sich auf **DSQL** und **PSQL**.

#### SQL Dialects
SQL-Dialekte definieren spezifische Sprachmerkmale und können auf Datenbank- und Verbindungsebene festgelegt werden.

*   **Dialect 1:** Für Rückwärtskompatibilität mit InterBase <= v5.
    *   `DATE` speichert Datum und Zeit (äquivalent zu `TIMESTAMP` in Dialekt 3).
    *   Doppelte Anführungszeichen (`"`) können alternativ zu einfachen Anführungszeichen (`'`) für String-Literale verwendet werden (Nicht-Standard, vermeiden!).
    *   `NUMERIC`/`DECIMAL` mit Präzision > 9 werden als Fließkommazahlen gespeichert.
    *   `BIGINT` wird nicht unterstützt.
    *   Bezeichner sind immer case-insensitiv und müssen den Regeln für reguläre Bezeichner folgen.
    *   `GEN_ID` gibt 64-Bit-Werte zurück, die aber bei Abfrage durch einen Dialekt-1-Client auf 32 Bit trunkiert werden.
*   **Dialect 2:** Nur auf Client-Verbindungsebene, zum Debuggen bei Migration von Dialekt 1 zu 3.
*   **Dialect 3:** Standard und empfohlen für neue Datenbanken.
    *   `NUMERIC`/`DECIMAL` mit Präzision > 9 werden intern als skalierte 64-Bit-Integer gespeichert.
    *   `TIME` Datentyp nur für Uhrzeit.
    *   `DATE` Datentyp nur für Datum.
    *   `BIGINT` (64-Bit Integer) ist verfügbar.
    *   Doppelte Anführungszeichen (`"`) sind *nur* für Delimited Identifiers reserviert (case-sensitiv, Sonderzeichen erlaubt).
    *   String-Literale *müssen* mit einfachen Anführungszeichen (`'`) eingeschlossen werden.
    *   Generatoren (Sequences) speichern und geben 64-Bit-Werte zurück.

**Diese Referenz beschreibt die Semantik von SQL Dialect 3, sofern nicht anders angegeben.**

#### Error Conditions
Jede SQL-Anweisung endet entweder erfolgreich oder schlägt mit einer spezifischen Fehlerbedingung fehl.

### Basic Elements: Statements, Clauses, Keywords

*   **Statement:** Die primäre SQL-Konstruktion, die eine Aktion definiert (z.B. `SELECT`, `CREATE TABLE`).
*   **Clause:** Ein Teil eines Statements, der eine spezifische Direktive gibt (z.B. `WHERE`, `ORDER BY`).
*   **Keyword:** Wörter der SQL-Sprache. Einige sind *reserviert* und können nicht als reguläre Bezeichner verwendet werden (z.B. `SELECT`, `TABLE`, `ADD`). Nicht-reservierte Keywords können als Bezeichner genutzt werden, dies wird aber nicht empfohlen.
    *   Eine Liste reservierter Wörter und Keywords findet sich in Appendix C.

### Identifiers
Namen für Datenbankobjekte (Tabellen, Spalten, Prozeduren etc.).

#### Rules for Regular Object Identifiers (Dialekt 1 & 3)
*   Maximal 31 Zeichen lang.
*   Muss mit einem nicht-akzentuierten ASCII-Buchstaben (A-Z, a-z) beginnen.
*   Darf nur ASCII-Buchstaben, Ziffern (0-9), Unterstrich (`_`) und Dollarzeichen (`$`) enthalten.
*   Case-insensitiv (werden intern als Großbuchstaben gespeichert und verglichen). `fullname`, `FULLNAME`, `FullName` sind identisch.
*   Darf kein reserviertes Wort sein.

```
<name> ::= <letter> | <name><letter> | <name><digit> | <name>_ | <name>$
<letter> ::= <upper letter> | <lower letter>
<upper letter> ::= A | B | ... | Z
<lower letter> ::= a | b | ... | z
<digit> ::= 0 | 1 | ... | 9
```

#### Rules for Delimited Object Identifiers (Nur Dialekt 3)
*   Maximal 31 Zeichen lang.
*   Muss vollständig in doppelte Anführungszeichen (`"`) eingeschlossen sein (z.B. `"My Table"`).
*   Kann beliebige Zeichen enthalten, einschließlich Leerzeichen, Sonderzeichen und akzentuierte Buchstaben (abhängig vom verwendeten Zeichensatz).
*   Kann ein reserviertes Wort sein (z.B. `"SELECT"`).
*   Ist **case-sensitiv**. `"FullName"` ist *unterschiedlich* von `"fullname"` und `FullName` (letzteres ist regulär und wird als `FULLNAME` gespeichert).
*   Nachfolgende Leerzeichen werden entfernt.

```
<delimited name> ::= "<permitted_character>[<permitted_character> ...]"
```

### Literals
Direkte Darstellung von Datenwerten.

*   **Integer:** `0`, `-34`, `45`. Ab FB 2.5 auch hexadezimal: `0x080000000` (siehe Hexadecimal Format for Integer Numbers).
*   **Real (Fließkomma):** `0.0`, `-3.14`, `3.23e-23`.
*   **String:** `'text'`, `'don''t!'` (Apostroph wird durch Verdopplung escaped). Müssen in einfachen Anführungszeichen stehen (in Dialekt 3).
*   **Binary String (Hex):** `x'48656C6C6F'` (ab FB 2.5).
*   **Date:** `DATE '2018-01-19'`.
*   **Time:** `TIME '15:12:56'`.
*   **Timestamp:** `TIMESTAMP '2018-01-19 13:32:02'`.
*   **Null State:** `NULL`.

### Operators and Special Characters
Spezielle Zeichen und Kombinationen mit syntaktischer Bedeutung.

```
<special char> ::= <space> | " | % | & | ' | ( | ) | * | + | , | - | . | / | : | ; | < | = | > | ? | [ | ] | ^ | { | }
```

**Operatoren:**

```sql
<operator> ::=
  <string concatenation operator>  -- ||
  | <arithmetic operator>          -- * / + -
  | <comparison operator>          -- = <> != ~= ^= > < >= <= !> ~> ^> !< ~< ^<
  | <logical operator>              -- NOT AND OR
```
(Details siehe Expressions)

### Comments
Werden vom Parser ignoriert. Dienen der Dokumentation.

*   **Block-Kommentar:** Beginnt mit `/*`, endet mit `*/`. Kann sich über mehrere Zeilen erstrecken.
    ```sql
    /* Dies ist ein
       mehrzeiliger Kommentar */
    ```
*   **Zeilen-Kommentar:** Beginnt mit `--` und geht bis zum Ende der Zeile.
    ```sql
    SELECT * FROM T; -- Dies ist ein Zeilenkommentar
    ```

```sql
<comment> ::= <block comment> | <single-line comment>
<block comment> ::= /* <ASCII char>[<ASCII char> ...] */
<single-line comment> ::= -- <ASCII char>[<ASCII char> ...]<end line>
```

## Data Types and Subtypes

Datentypen definieren die Art der Daten, die in Spalten, Variablen oder Parametern gespeichert werden können.

**Übersicht:**

| Name                     | Size                  | Precision & Limits                      | Description (Dialect 3)                                                              |
| :----------------------- | :-------------------- | :-------------------------------------- | :----------------------------------------------------------------------------------- |
| `BIGINT`                 | 64 bits               | -2^63 to 2^63 - 1                       | 64-bit Integer. Nur Dialekt 3.                                                       |
| `BLOB`                   | Varying (up to 4GB)   | Segment size up to 64KB                 | Binary Large Object. Für große Binär- oder Textdaten. Subtypen definieren Inhalt. |
| `CHAR(n)`, `CHARACTER(n)`| n chars (max 32767 B) | 1 to 32,767 bytes                     | Feste Länge. Kürzere Strings werden mit Leerzeichen aufgefüllt (nicht gespeichert).   |
| `DATE`                   | 32 bits               | 0001-01-01 AD to 9999-12-31 AD          | Nur Datum (ohne Zeit).                                                               |
| `DECIMAL(p, s)`          | 16, 32 or 64 bits     | p: 1-18 (total digits), s: 0-p (scale)| Festkommazahl. `p` = Präzision, `s` = Nachkommastellen. Genauigkeit garantiert.     |
| `DOUBLE PRECISION`       | 64 bits               | IEEE 754 double (~15 digits)            | Fließkommazahl mit doppelter Genauigkeit.                                            |
| `FLOAT`                  | 32 bits               | IEEE 754 single (~7 digits)             | Fließkommazahl mit einfacher Genauigkeit.                                            |
| `INTEGER`, `INT`         | 32 bits               | -2,147,483,648 to 2,147,483,647        | 32-bit Integer.                                                                      |
| `NUMERIC(p, s)`          | 16, 32 or 64 bits     | p: 1-18 (total digits), s: 0-p (scale)| Festkommazahl. Ähnlich DECIMAL, Standardkonform: Präzision ist *mindestens* p.      |
| `SMALLINT`               | 16 bits               | -32,768 to 32,767                       | 16-bit Integer.                                                                      |
| `TIME`                   | 32 bits               | 00:00:00.0000 to 23:59:59.9999          | Nur Uhrzeit (ohne Datum).                                                            |
| `TIMESTAMP`              | 64 bits (2x32)        | 0001-01-01 AD to 9999-12-31 AD          | Datum und Uhrzeit.                                                                   |
| `VARCHAR(n)`, `CHAR VARYING(n)`, `CHARACTER VARYING(n)` | n chars (max 32765 B) | 1 to 32,765 bytes                     | Variable Länge. Speichert nur tatsächliche Zeichen + 2 Byte Längeninfo. `n` ist Pflicht. |

### Integer Data Types (`SMALLINT`, `INTEGER`/`INT`, `BIGINT`)

*   Ganze Zahlen unterschiedlicher Größe.
*   `BIGINT` nur in Dialekt 3 verfügbar.
*   Keine vorzeichenlosen (`unsigned`) Typen.
*   **Hexadezimalformat (ab FB 2.5):** Präfix `0x` oder `0X`. 1-8 Hex-Ziffern -> `INTEGER`, 9-16 Hex-Ziffern -> `BIGINT`. Kann implizit zu `SMALLINT` konvertiert werden, wenn der Wert im Bereich liegt.
    ```sql
    SELECT 0x6F55A09D42; -- BIGINT
    SELECT 0x80000000; -- INTEGER (-2147483648)
    SELECT 0x080000000; -- BIGINT (2147483648)
    ```

### Floating-Point Data Types (`FLOAT`, `DOUBLE PRECISION`)

*   Basierend auf IEEE 754. Speichern Vorzeichen, Exponent, Mantisse.
*   Präzision ist approximativ (`FLOAT` ~7, `DOUBLE PRECISION` ~15 Stellen).
*   Nicht empfohlen für Geldwerte oder exakte Vergleiche (Rundungsfehler). Verwende `BETWEEN` statt `=` für Vergleiche.

### Fixed-Point Data Types (`NUMERIC`, `DECIMAL`)

*   Für exakte numerische Werte, besonders geeignet für Geldwerte.
*   `NUMERIC(p, s)`, `DECIMAL(p, s)`: `p` = Präzision (Gesamtzahl der Stellen), `s` = Skala (Anzahl Nachkommastellen). `0 <= s <= p <= 18`.
*   Speicherung intern als `SMALLINT`, `INTEGER` oder `BIGINT` (Dialekt 3) oder `DOUBLE PRECISION` (Dialekt 1 für p > 9).
*   **Wichtig:** Das Verhalten von `NUMERIC` und `DECIMAL` in Firebird entspricht dem SQL-Standard `DECIMAL`: Die tatsächliche Präzision ist *mindestens* `p`. Das Speicherformat hängt von `p` ab (siehe Tabelle 2 im OCR).

### Date and Time Data Types (`DATE`, `TIME`, `TIMESTAMP`)

*   **Dialekt 3:**
    *   `DATE`: Nur Datum.
    *   `TIME`: Nur Uhrzeit (00:00:00.0000 - 23:59:59.9999).
    *   `TIMESTAMP`: Datum und Uhrzeit.
*   **Dialekt 1:**
    *   `DATE`: Speichert Datum und Uhrzeit (entspricht `TIMESTAMP` in Dialekt 3).
    *   `TIMESTAMP`: Synonym für `DATE`.
*   **Sekundenbruchteile:** Werden intern mit einer Präzision von 1/10000 Sekunden gespeichert. Die tatsächliche Präzision hängt von der Quelle ab (`CURRENT_TIME` vs. `CURRENT_TIMESTAMP`, Literal vs. Variable). `EXTRACT(MILLISECOND ...)` kann verwendet werden.
*   **Arithmetik:** Datum/Zeit-Werte können subtrahiert werden (Ergebnis ist Intervall in Tagen bzw. Sekunden). Numerische Werte können addiert/subtrahiert werden (Interpretation als Tage bei DATE/TIMESTAMP, als Sekunden bei TIME). Siehe Tabelle 5 im OCR für Details.

### Character Data Types (`CHAR`, `VARCHAR`, `NCHAR`)

*   `CHAR(n)` oder `CHARACTER(n)`: Feste Länge. Kürzere Eingaben werden mit Leerzeichen aufgefüllt. Max. 32767 Bytes.
*   `VARCHAR(n)` oder `CHARACTER VARYING(n)`: Variable Länge. Speichert tatsächliche Länge. Max. 32765 Bytes. `n` ist Pflicht.
*   `NCHAR(n)`, `NATIONAL CHARACTER [VARYING](n)`: Äquivalent zu `CHAR`/`VARCHAR`, aber mit vordefiniertem Zeichensatz (ISO8859_1).
*   **Zeichensatz (`CHARACTER SET`):** Definiert die Kodierung. Standard ist der Datenbank-Default oder `NONE`. `UTF8` unterstützt Unicode. `OCTETS` für Binärdaten.
*   **Collation (`COLLATE`):** Definiert Sortierreihenfolge und Vergleichsregeln (z.B. case-sensitiv/insensitiv). Jede Zeichensatz hat einen Default-Collation. Kann pro Spalte/Domain/Ausdruck überschrieben werden.
*   **Index Limits:** Maximale Indexschlüsselgröße hängt von Page Size und Bytes pro Zeichen ab (siehe Tabelle 7 im OCR).

### Binary Data Types (`BLOB`)

*   Binary Large Object. Für Daten unbestimmter, potenziell sehr großer Länge (bis 4GB, praktisch oft weniger, z.B. < 2GB bei 4K Page Size).
*   **Subtypen (`SUB_TYPE`):**
    *   `0` (`BINARY`): Default. Untypisierte Binärdaten (Bilder, Audio, etc.).
    *   `1` (`TEXT`): Für große Texte. Kann `CHARACTER SET` und `COLLATE` haben.
    *   `< 0`: Benutzerdefinierte Subtypen (-1 bis -32768).
*   **Segmente:** Historisch für C/`gpre`-Zugriff relevant, heute meist irrelevant.
*   **Operationen:** Zuweisung (`=`), Vergleich (`=`, `<>`, `<`, `<=`, `>`, `>=`), `IS [NOT] DISTINCT FROM`, `IS [NOT] NULL`. Partiell unterstützt (Fehler bei >32KB Suchargument): `LIKE`, `STARTING WITH`, `CONTAINING`. Aggregation (`GROUP BY`) wirkt auf BLOB-ID, nicht Inhalt.

### ARRAY Type (Abweichung vom relationalen Modell)

*   Mehrdimensionales Array fester Größe als Spaltentyp.
*   Syntax: `datatype [dim1, dim2, ...]` wobei `dim` ist `n` oder `m:n`.
    *   `n`: Obergrenze (Untergrenze 1). `n < 1` -> `n..1`, `n > 1` -> `1..n`.
    *   `m:n`: Unter- und Obergrenze (explizit).
*   Basis-Datentyp kann alles außer `BLOB` oder `ARRAY` sein.
*   Gespeichert als spezieller BLOB-Subtyp.
*   Begrenzte Unterstützung für Operationen in DSQL/PSQL. Zugriff auf Elemente über `array_column[index1, index2]`.

### Special Data Types

#### `SQL_NULL`
*   Interner Typ, nicht für Deklarationen. Wird verwendet, um den Typ von Parametern in `? IS NULL`-Prädikaten zu behandeln. Relevant für API-Programmierung.

### Conversion of Data Types

*   **Explizit:** `CAST ( value AS target_type )`
    *   `target_type` kann sein: SQL-Datentyp, `DOMAIN domainname`, `TYPE OF DOMAIN domainname`, `TYPE OF COLUMN rel.col`.
    *   Casting zu Domain prüft `CHECK`-Constraints. `TYPE OF DOMAIN` ignoriert Constraints.
    *   Casting zu `TYPE OF COLUMN` übernimmt Datentyp und ggf. Zeichensatz/Collation, aber keine Constraints oder Defaults.
    *   Tabelle 8 im OCR zeigt mögliche Konvertierungen.
*   **Shorthand Cast (Literale zu Datum/Zeit):** `DATE 'string'`, `TIME 'string'`, `TIMESTAMP 'string'`. Wird *sofort* beim Parsen ausgewertet. Für dynamische Werte (`'NOW'`, `'TODAY'`) die volle `CAST`-Syntax verwenden. Siehe Tabelle 9/10 für Formate.
*   **Implizit:**
    *   **Dialekt 1:** Häufig möglich (z.B. String zu Zahl, String zu Datum).
    *   **Dialekt 3:** Generell nicht möglich, `CAST` ist fast immer nötig.
    *   **Ausnahme (Dialekt 3):** String-Konkatenation (`||`) konvertiert Nicht-String-Operanden implizit zu Strings.

### Custom Data Types — Domains

*   Benutzerdefinierte Typen basierend auf Standard-SQL-Typen.
*   Kapseln Datentyp + Attribute: `DEFAULT`, `NOT NULL`, `CHECK`, `CHARACTER SET`, `COLLATE`.
*   Ermöglichen Wiederverwendung und zentrale Definition von Typen/Regeln.
*   Verwendbar für Tabellenspalten, PSQL-Variablen und -Parameter.
*   **Attribute können bei Spaltendefinition überschrieben werden:** `DEFAULT`, `CHECK` (additiv), `COLLATE`. `NOT NULL` kann *nicht* überschrieben werden, wenn die Domain `NOT NULL` ist. Eine nullable Domain kann für eine `NOT NULL`-Spalte verwendet werden.
*   **DDL:** `CREATE DOMAIN`, `ALTER DOMAIN`, `DROP DOMAIN`.

## Common Language Elements

Elemente, die in verschiedenen SQL-Kontexten vorkommen.

### Expressions
Konstrukte zur Auswertung, Transformation und zum Vergleich von Werten. Können enthalten:
*   Spaltennamen
*   Array-Elemente (`array[s]`)
*   Arithmetische Operatoren (`+`, `-`, `*`, `/`)
*   String-Konkatenation (`||`)
*   Logische Operatoren (`NOT`, `AND`, `OR`)
*   Vergleichsoperatoren (`=`, `<>`, `>`, `<`, etc.)
*   Vergleichsprädikate (`LIKE`, `STARTING WITH`, `CONTAINING`, `SIMILAR TO`, `BETWEEN`, `IS [NOT] NULL`, `IS [NOT] DISTINCT FROM`)
*   Existenzielle Prädikate (`IN`, `EXISTS`, `SINGULAR`, `ALL`, `ANY`, `SOME`)
*   Konstanten/Literale
*   Kontextvariablen (`CURRENT_USER`, etc.)
*   Lokale Variablen (PSQL, ESQL)
*   Positionale Parameter (`?`, nur DSQL/ESQL bei API-Nutzung)
*   Subqueries (die einen einzelnen Skalarwert liefern)
*   Funktionsaufrufe (intern oder UDF)
*   `CAST`-Ausdrücke
*   Bedingte Ausdrücke (`CASE`, `DECODE`, `IIF`, etc.)
*   Parenthesen `()` zur Gruppierung
*   `COLLATE`-Klausel für String-Vergleiche
*   `NEXT VALUE FOR sequence`

#### Constants (Literals)
*   **String:** In `' '` eingeschlossen. Apostroph wird mit `''` escaped. Max. 32767 Bytes. Zeichensatz wird angenommen oder kann via Introducer-Syntax (`_charset'string'`) angegeben werden.
*   **Hex-String (ab FB 2.5):** `x'...'` oder `X'...'`. Jedes Byte als 2 Hex-Ziffern. Default `CHARACTER SET OCTETS`. Kann via Introducer (`_charset x'...'`) interpretiert werden.
*   **Number:** Dezimalpunkt ist `.`. Kein Tausendertrennzeichen. Exponentialschreibweise (`2.34e-5`) möglich.
*   **Hex-Number (ab FB 2.5):** `0x...` oder `0X...`. 1-8 Ziffern -> `INTEGER`, 9-16 Ziffern -> `BIGINT`. Wertebereich beachten (Vorzeichenbit!).

#### SQL Operators
*   **Präzedenz:** 1. Konkatenation (`||`), 2. Arithmetik (`* / + -`), 3. Vergleich, 4. Logik (`NOT AND OR`). Gleiche Präzedenz: links nach rechts. Parenthesen `()` ändern die Reihenfolge.
*   **Arithmetisch:** `+`, `-` (unär/binär), `*`, `/`.
*   **Vergleich:** Siehe oben. Beachte `IS [NOT] DISTINCT FROM` für NULL-Vergleiche.
*   **Logisch:** `NOT`, `AND`, `OR`. Dreiwertige Logik (TRUE, FALSE, UNKNOWN/NULL).
*   **Sequence:** `NEXT VALUE FOR sequence` (SQL-konform, äquivalent zu `GEN_ID(sequence, 1)`).

#### Conditional Expressions
*   **`CASE`:**
    *   **Simple CASE:** Vergleicht einen Wert mit mehreren WHEN-Klauseln.
      ```sql
      CASE <test_expr>
        WHEN <expr1> THEN <result1>
        [WHEN <expr2> THEN <result2> ...]
        [ELSE <default_result>]
      END
      ```
    *   **Searched CASE:** Prüft mehrere boolesche Bedingungen.
      ```sql
      CASE
        WHEN <bool_expr1> THEN <result1>
        [WHEN <bool_expr2> THEN <result2> ...]
        [ELSE <default_result>]
      END
      ```
    *   Gibt den `result` der ersten passenden Bedingung zurück, sonst `default_result` oder `NULL`.
*   **`DECODE()`:** Shorthand für simple CASE.
*   **`IIF()`:** Shorthand für searched CASE mit einer Bedingung.
*   **`NULLIF()`:** Gibt NULL zurück, wenn zwei Ausdrücke gleich sind, sonst den ersten.
*   **`COALESCE()`:** Gibt den ersten Nicht-NULL-Ausdruck aus einer Liste zurück.
*   **`MAXVALUE()` / `MINVALUE()`:** Gibt Maximum/Minimum aus einer Liste zurück (NULL, wenn ein Argument NULL ist).

#### `NULL` in Expressions
*   `NULL` ist kein Wert, sondern ein Zustand (unbekannt/nicht existent).
*   Arithmetische, String-, Datum/Zeit-Operationen mit `NULL` ergeben `NULL`.
*   Logische Operationen:
    *   `NOT NULL` -> `NULL`
    *   `TRUE OR NULL` -> `TRUE`
    *   `FALSE OR NULL` -> `NULL`
    *   `NULL OR NULL` -> `NULL`
    *   `TRUE AND NULL` -> `NULL`
    *   `FALSE AND NULL` -> `FALSE`
    *   `NULL AND NULL` -> `NULL`
*   Vergleiche mit `NULL` (z.B. `col = NULL`, `NULL <> NULL`) ergeben `UNKNOWN` (was wie `NULL` behandelt wird). Verwende `IS [NOT] NULL` oder `IS [NOT] DISTINCT FROM`.

#### Subqueries
*   `SELECT`-Anweisungen innerhalb anderer Anweisungen, in `()`.
*   **Verwendung:**
    *   In der Spaltenliste (muss Skalarwert zurückgeben).
    *   In Prädikaten (`WHERE`, `HAVING`), oft mit Vergleichsoperatoren (muss Skalarwert liefern) oder Existenzprädikaten (`IN`, `EXISTS` etc.).
    *   Als Datenquelle in der `FROM`-Klausel (**Derived Table**).
    *   Als Teil einer **Common Table Expression (CTE)** (`WITH`-Klausel).
*   **Correlated Subquery:** Bezieht sich auf Spalten der äußeren Abfrage. Wird für jede Zeile der äußeren Abfrage neu ausgewertet.
*   **Scalar Subquery:** Gibt genau eine Zeile und eine Spalte zurück.

### Predicates
Logische Ausdrücke, die TRUE, FALSE oder UNKNOWN (NULL) ergeben. Werden in `WHERE`, `HAVING`, `CHECK`, `CASE`, `IIF`, `ON` verwendet.

#### Assertions
Kombinationen von Prädikaten mittels `NOT`, `AND`, `OR` und `()`.

#### Comparison Predicates
*   Standardoperatoren: `=`, `<>`, `!=`, `~=`, `^=`, `<`, `<=`, `>`, `>=`. Varianten wie `!<`, `!>` etc.
*   `BETWEEN v1 AND v2`: Prüft, ob ein Wert im inklusiven Bereich liegt. `v1` muss kleiner/gleich `v2` sein.
*   `LIKE pattern [ESCAPE char]`: Mustervergleich mit Wildcards (`%` für beliebig viele Zeichen, `_` für genau ein Zeichen). `ESCAPE` definiert ein Escape-Zeichen für Wildcards.
*   `STARTING WITH substr`: Prüft, ob ein String mit `substr` beginnt (case-sensitiv). Kann Index nutzen.
*   `CONTAINING substr`: Prüft, ob ein String `substr` enthält (case-insensitiv, außer bei akzent-sensitiver Collation). Kann Index nutzen.
*   `SIMILAR TO pattern [ESCAPE char]`: SQL-Standard-Regex-Vergleich. Muss den *gesamten* String matchen.
*   `IS [NOT] NULL`: Prüft auf `NULL`.
*   `IS [NOT] DISTINCT FROM`: Vergleichsoperator, der `NULL` als gleichwertig behandelt (`NULL IS NOT DISTINCT FROM NULL` ist TRUE).

#### Existential Predicates
*   `EXISTS (subquery)`: TRUE, wenn die Subquery mindestens eine Zeile zurückgibt.
*   `IN (value_list | subquery)`: TRUE, wenn der Wert in der Liste oder im (einespaltigen) Ergebnis der Subquery enthalten ist.
*   `SINGULAR (subquery)`: TRUE, wenn die Subquery genau eine Zeile zurückgibt.

#### Quantified Subquery Predicates
*   `<value> <op> {ALL | SOME | ANY} (subquery)`: Vergleicht einen Wert mit *allen* (`ALL`) oder *mindestens einem* (`SOME`/`ANY`) Wert aus dem (einespaltigen) Ergebnis der Subquery.
    *   `= ANY` ist äquivalent zu `IN`.
    *   `<> ALL` prüft, ob der Wert von *allen* Werten der Subquery verschieden ist.
    *   Vorsicht bei `ALL` mit leeren Subquery-Ergebnissen (ergibt immer TRUE).
    *   Vorsicht bei `ANY`/`SOME` mit leeren Subquery-Ergebnissen (ergibt immer FALSE).
    *   Vorsicht bei NULL-Werten im Subquery-Ergebnis (kann zu UNKNOWN führen).

Okay, ich mache mit dem nächsten Abschnitt weiter: Data Definition (DDL) Statements.

```markdown
## Data Definition (DDL) Statements

DDL-Anweisungen werden verwendet, um Datenbankobjekte zu erstellen, zu ändern und zu löschen. Änderungen werden erst nach einem `COMMIT` wirksam.

### DATABASE

#### CREATE DATABASE
Erstellt eine neue Datenbank.

```sql
CREATE {DATABASE | SCHEMA} <filespec>
  [USER 'username' [PASSWORD 'password']]
  [PAGE_SIZE [=] size]
  [LENGTH [=] num [PAGE[S]] -- Nur für Sekundärdateien relevant
  [SET NAMES 'charset']
  [DEFAULT CHARACTER SET default_charset [COLLATION collation]]
  [<sec_file> [<sec_file> ...]]
  [DIFFERENCE FILE 'diff_file'] -- Nur DSQL
```

**Wichtige Parameter:**
*   `<filespec>`: Pfad und Dateiname der primären Datenbankdatei (z.B. `'C:\data\mydb.fdb'` oder `'server:/path/to/db.fdb'`). Muss eindeutig sein. Kann auch ein Alias sein.
*   `USER`/`PASSWORD`: Definiert den Datenbank-Owner. Wenn weggelassen, werden Umgebungsvariablen `ISC_USER`/`ISC_PASSWORD` verwendet (falls gesetzt) oder der aktuelle Betriebssystembenutzer (bei Trusted Authentication).
*   `PAGE_SIZE`: Seitengröße in Bytes (4096, 8192, 16384). Default ist 4096.
*   `DEFAULT CHARACTER SET`: Standardzeichensatz für die Datenbank (z.B. `UTF8`, `WIN1251`, `NONE`). Beeinflusst neu erstellte String-Spalten/Domains ohne expliziten Zeichensatz.
*   `COLLATION`: Standard-Collation für den `DEFAULT CHARACTER SET`.
*   `<sec_file>`: Definiert sekundäre Datenbankdateien für Multi-File-Datenbanken (heute selten genutzt).
    ```sql
    FILE 'filepath' [LENGTH [=] num [PAGE[S]]] [STARTING [AT [PAGE]] pagenum]
    ```
*   `DIFFERENCE FILE`: Pfad für die Delta-Datei bei Verwendung von `ALTER DATABASE BEGIN BACKUP` (nBackup).
*   `SET SQL DIALECT`: Muss *vor* `CREATE DATABASE` ausgeführt werden, um eine Datenbank in Dialekt 1 zu erstellen (Default ist Dialekt 3).

**Hinweise:**
*   `SCHEMA` ist ein Synonym für `DATABASE`.
*   Die Datenbankdatei darf zum Zeitpunkt der Erstellung nicht existieren.
*   Multi-File-Datenbanken sind meist veraltet. `LENGTH` auf der primären Datei ist nicht sinnvoll.
*   Der Ersteller wird der Datenbank-Owner. Nur Administratoren und der Owner können die Datenbank verwalten/löschen.

#### ALTER DATABASE
Ändert die Dateistruktur oder den Backup-Status einer Datenbank.

```sql
ALTER {DATABASE | SCHEMA}
  [ADD <sec_file> [<sec_file> ...]]  -- Sekundärdatei(en) hinzufügen
  [ADD DIFFERENCE FILE 'diff_file']  -- Pfad für Delta-Datei setzen/ändern
  [DROP DIFFERENCE FILE]            -- Pfad für Delta-Datei entfernen
  [{BEGIN | END} BACKUP]             -- Backup-Modus starten/beenden (nur DSQL)
```

**Hinweise:**
*   Nur Administratoren können `ALTER DATABASE` ausführen.
*   `BEGIN BACKUP`: Versetzt die Datenbank in einen "Copy-Safe"-Modus. Änderungen werden in die Delta-Datei geschrieben. Die Hauptdatei kann nun sicher per Dateisystem kopiert werden. **Nicht sicher für Multi-File-Datenbanken!** Nur für Single-File verwenden.
*   `END BACKUP`: Beendet den Backup-Modus, integriert die Änderungen aus der Delta-Datei in die Hauptdatei.
*   `ADD`/`DROP DIFFERENCE FILE`: Verwaltet nur den Pfadnamen für die Delta-Datei, löscht oder erstellt die Datei nicht physisch.

#### DROP DATABASE
Löscht die aktuelle Datenbank.

```sql
DROP DATABASE
```

**Hinweise:**
*   Man muss mit der zu löschenden Datenbank verbunden sein.
*   Löscht die primäre Datei, alle sekundären Dateien und alle Shadow-Dateien.
*   Nur Administratoren können `DROP DATABASE` ausführen.
*   Alle Verbindungen zur Datenbank müssen getrennt sein.

### SHADOW
Eine exakte, seitenweise Kopie einer Datenbank, die bei Änderungen synchron gehalten wird. Dient als Hot-Standby.

#### CREATE SHADOW
Erstellt einen Shadow für die aktuelle Datenbank.

```sql
CREATE SHADOW <sh_num> [AUTO | MANUAL] [CONDITIONAL]
  'filepath' [LENGTH [=] num [PAGE[S]]]
  [<secondary_file> ...]
```

**Wichtige Parameter:**
*   `<sh_num>`: Positive Ganzzahl zur Identifizierung des Shadow-Sets.
*   `AUTO`/`MANUAL`: Verhalten, wenn der Shadow ausfällt. `AUTO` (Default): Shadowing stoppt, DB läuft weiter. `MANUAL`: DB wird unzugänglich, bis Shadow wieder da oder gedroppt ist.
*   `CONDITIONAL`: Wenn mit `AUTO` verwendet, versucht das System, den Shadow automatisch neu zu erstellen, wenn er ausfällt.
*   `filepath`: Pfad zur (primären) Shadow-Datei.
*   `LENGTH`/`<secondary_file>`: Optional für Multi-File-Shadows.

**Hinweise:**
*   Nur Administratoren können Shadows erstellen.
*   Ein Shadow kann selbst mehrteilig sein, unabhängig von der Struktur der Original-DB.
*   Seitengröße des Shadows ist identisch mit der der Original-DB.
*   Bei Ausfall der Primärdatei schaltet der Server automatisch auf einen verfügbaren Shadow um (dieser Shadow wird dann unbrauchbar).

#### DROP SHADOW
Löscht einen Shadow.

```sql
DROP SHADOW <sh_num>
```

**Hinweise:**
*   Nur Administratoren können Shadows löschen.
*   Löscht die Shadow-Dateien und entfernt den Shadow-Eintrag aus dem Datenbank-Header.

### DOMAIN
Benutzerdefinierte Datentypen.

#### CREATE DOMAIN
Erstellt eine neue Domain.

```sql
CREATE DOMAIN domain_name [AS] <datatype>
  [DEFAULT {<literal> | NULL | <context_var>}]
  [NOT NULL]
  [CHECK (<dom_condition>)]
  [COLLATE collation_name]
```

**Wichtige Parameter:**
*   `domain_name`: Eindeutiger Name für die Domain.
*   `<datatype>`: Ein SQL-Basistyp (inkl. `ARRAY`, exkl. `BLOB`-Arrays).
*   `DEFAULT`: Optionaler Standardwert (Literal oder kompatible Kontextvariable).
*   `NOT NULL`: Optional, verbietet NULL-Werte.
*   `CHECK`: Optional, eine Bedingung, die für Werte dieser Domain gelten muss. `VALUE` repräsentiert den zu prüfenden Wert. Bedingung ist erfüllt bei TRUE oder UNKNOWN (NULL).
*   `COLLATE`: Optional, spezifische Collation für String-Typen.

**Hinweise:**
*   Array-Domains können nur für Tabellenspalten verwendet werden, nicht für PSQL-Variablen.
*   Jeder verbundene Benutzer kann Domains erstellen.

#### ALTER DOMAIN
Ändert eine bestehende Domain.

```sql
ALTER DOMAIN domain_name
  [TO new_name]
  [TYPE <datatype>] -- Datentyp ändern (eingeschränkt)
  [{SET DEFAULT {<literal> | NULL | <context_var>} | DROP DEFAULT}]
  [{ADD [CONSTRAINT] CHECK (<dom_condition>) | DROP CONSTRAINT}]
  -- [TYPE <datatype>] -- Alternative Position für Typänderung
```

**Hinweise:**
*   Umbenennen (`TO new_name`) geht nur, wenn keine Abhängigkeiten bestehen.
*   Typänderung (`TYPE <datatype>`) ist nur zu kompatiblen Typen möglich, die keinen Datenverlust verursachen. Array-Typen können nicht geändert oder in Arrays umgewandelt werden.
*   `NOT NULL`-Constraint kann in FB <= 2.5 nicht hinzugefügt/entfernt werden.
*   Default-Collation kann nicht geändert werden (nur durch DROP/CREATE).
*   `ADD CHECK` fügt eine Bedingung hinzu (löscht keine bestehende). `DROP CONSTRAINT` löscht die (einzige) CHECK-Constraint der Domain.
*   Änderungen können PSQL-Code invalidieren (siehe RDB$VALID_BLR).

#### DROP DOMAIN
Löscht eine bestehende Domain.

```sql
DROP DOMAIN domain_name
```

**Hinweise:**
*   Kann nur gelöscht werden, wenn keine Tabellenspalten oder PSQL-Module sie referenzieren.
*   Jeder verbundene Benutzer kann Domains löschen (sofern keine Abhängigkeiten bestehen).

### TABLE
Relationen zur Speicherung von Daten.

#### CREATE TABLE
Erstellt eine neue Tabelle.

```sql
CREATE [GLOBAL TEMPORARY] TABLE tablename
  [EXTERNAL [FILE] 'filespec']
  (<col_def> [, {<col_def> | <tconstraint>} ...])
  [ON COMMIT {DELETE | PRESERVE} ROWS] -- Nur für GLOBAL TEMPORARY
```

**Wichtige Komponenten:**
*   `GLOBAL TEMPORARY`: Erstellt eine globale temporäre Tabelle (Metadaten persistent, Daten transaktions- oder sitzungsgebunden).
    *   `ON COMMIT DELETE ROWS` (Default): Transaktionsgebunden. Daten werden bei COMMIT/ROLLBACK gelöscht.
    *   `ON COMMIT PRESERVE ROWS`: Sitzungsgebunden. Daten bleiben bis zum Ende der Verbindung erhalten.
*   `EXTERNAL [FILE] 'filespec'`: Erstellt eine externe Tabelle, deren Daten in einer externen Textdatei fester Satzlänge gespeichert sind. Nur `INSERT` und `SELECT` möglich. Dateizugriff unterliegt `ExternalFileAccess` in `firebird.conf`.
*   `<col_def>`: Spaltendefinition.
*   `<tconstraint>`: Tabellen-Constraint.

**Spaltendefinition (`<col_def>`):**

```sql
-- Reguläre Spalte
colname {<datatype> | domainname}
  [DEFAULT {<literal> | NULL | <context_var>}]
  [NOT NULL]
  [<col_constraint>]
  [COLLATE collation_name]

-- Berechnete Spalte (Computed Field)
colname [<datatype>] -- Datentyp optional
  {COMPUTED [BY] | GENERATED ALWAYS AS} (<expression>)
```

*   Spalten können auf einem Basistyp oder einer Domain basieren.
*   `DEFAULT`: Standardwert für `INSERT`.
*   `NOT NULL`: Verbietet NULL-Werte.
*   `COLLATE`: Spezifische Collation für die String-Spalte.
*   `COMPUTED BY` / `GENERATED ALWAYS AS`: Definiert eine berechnete Spalte. Der Wert wird bei jeder Abfrage basierend auf dem Ausdruck neu berechnet.

**Spalten-Constraint (`<col_constraint>`):**

```sql
[CONSTRAINT constr_name]
{ PRIMARY KEY [<using_index>]
| UNIQUE [<using_index>]
| REFERENCES other_table [(colname)] [<using_index>]
    [ON DELETE {NO ACTION | CASCADE | SET DEFAULT | SET NULL}]
    [ON UPDATE {NO ACTION | CASCADE | SET DEFAULT | SET NULL}]
| CHECK (<check_condition>) }
```

**Tabellen-Constraint (`<tconstraint>`):**

```sql
[CONSTRAINT constr_name]
{ PRIMARY KEY (<col_list>) [<using_index>]
| UNIQUE (<col_list>) [<using_index>]
| FOREIGN KEY (<col_list>)
    REFERENCES other_table [(<col_list>)] [<using_index>]
    [ON DELETE {NO ACTION | CASCADE | SET DEFAULT | SET NULL}]
    [ON UPDATE {NO ACTION | CASCADE | SET DEFAULT | SET NULL}]
| CHECK (<check_condition>) }
```

*   **Constraints:** `PRIMARY KEY`, `UNIQUE`, `FOREIGN KEY` (`REFERENCES`), `CHECK`.
*   `CONSTRAINT constr_name`: Optionaler Name für den Constraint (bei Spalten-Constraints automatisch generiert, wenn weggelassen).
*   `PRIMARY KEY`: Eindeutiger Identifikator für eine Zeile. Impliziert `NOT NULL` und `UNIQUE`. Nur ein PK pro Tabelle. Kann mehrere Spalten umfassen (nur als Tabellen-Constraint). Erzeugt automatisch einen Index.
*   `UNIQUE`: Stellt Einzigartigkeit sicher (NULLs sind erlaubt und gelten als verschieden, außer bei identischen Nicht-NULL-Teilschlüsseln). Kann mehrere Spalten umfassen. Erzeugt automatisch einen Index.
*   `FOREIGN KEY`: Stellt referentielle Integrität sicher. Verweist auf einen `PRIMARY KEY` oder `UNIQUE KEY` in einer anderen (oder derselben) Tabelle. `ON DELETE`/`ON UPDATE`-Aktionen definieren Verhalten bei Änderungen im referenzierten Schlüssel. Erzeugt automatisch einen Index.
*   `CHECK`: Definiert eine Bedingung, die für jede Zeile wahr (oder unbekannt/NULL) sein muss.
*   `<using_index>`: (Nicht-Standard) Erlaubt, einen benutzerdefinierten Index für PK/UK/FK zu verwenden statt des automatisch generierten.
    ```sql
    USING [ASC[ENDING] | DESC[ENDING]] INDEX indexname
    ```

#### ALTER TABLE
Ändert die Struktur einer bestehenden Tabelle.

```sql
ALTER TABLE tablename
  <operation> [, <operation> ...]
```

**Mögliche Operationen (`<operation>`):**
*   `ADD <col_def>`: Fügt eine neue Spalte hinzu (regulär oder berechnet).
*   `ADD <tconstraint>`: Fügt einen neuen Tabellen-Constraint hinzu.
*   `DROP colname`: Löscht eine Spalte (nur wenn keine Abhängigkeiten bestehen).
*   `DROP CONSTRAINT constr_name`: Löscht einen benannten Constraint.
*   `ALTER [COLUMN] colname <col_mod>`: Ändert eine bestehende Spalte.

**Spaltenmodifikationen (`<col_mod>` für reguläre Spalten):**
*   `TO newname`: Benennt die Spalte um (nur wenn keine Abhängigkeiten bestehen).
*   `POSITION newpos`: Ändert die (logische) Position der Spalte.
*   `TYPE {<datatype> | domainname}`: Ändert den Datentyp (eingeschränkt, kein Datenverlust erlaubt, kein Array-Typwechsel).
*   `SET DEFAULT {<literal> | NULL | <context_var>}`: Setzt/ändert den Default-Wert.
*   `DROP DEFAULT`: Entfernt den Default-Wert.

**Spaltenmodifikationen (`<col_mod>` für berechnete Spalten):**
*   `TO newname`: Benennt die Spalte um.
*   `POSITION newpos`: Ändert die Position.
*   `[TYPE <datatype>] {COMPUTED [BY] | GENERATED ALWAYS AS} (<expression>)`: Ändert Typ und/oder Ausdruck.

**Hinweise:**
*   Nur der Tabellen-Owner und Administratoren können `ALTER TABLE` ausführen.
*   Hinzufügen/Löschen von Spalten und Ändern des Datentyps erhöhen den Metadaten-Versionszähler der Tabelle (max. 255). Backup/Restore ist zum Zurücksetzen nötig.
*   `NOT NULL`-Constraint kann nicht per `ALTER TABLE` hinzugefügt/entfernt werden.
*   Default-Collation kann nicht geändert werden.

#### DROP TABLE
Löscht eine bestehende Tabelle.

```sql
DROP TABLE tablename
```

**Hinweise:**
*   Löscht die Tabelle und alle zugehörigen Trigger und Indizes.
*   Fehlschlag, wenn Abhängigkeiten bestehen (Views, Prozeduren, Trigger, FKs anderer Tabellen).
*   Nur der Tabellen-Owner und Administratoren können `DROP TABLE` ausführen.

#### RECREATE TABLE
Erstellt eine neue Tabelle oder ersetzt eine bestehende.

```sql
RECREATE [GLOBAL TEMPORARY] TABLE tablename ... -- Rest wie CREATE TABLE
```

**Hinweise:**
*   Versucht zuerst, die bestehende Tabelle via `DROP TABLE` zu löschen. Wenn dies wegen Abhängigkeiten fehlschlägt, schlägt auch `RECREATE TABLE` fehl.
*   Nützlich in Skripten, um sicherzustellen, dass eine Tabelle existiert, ohne vorher prüfen zu müssen.

Okay, ich setze die Zusammenfassung mit dem Abschnitt über Indizes fort.

```markdown
### INDEX
Datenbankobjekte zur Beschleunigung von Suchen, Sortierungen und zur Erzwingung von Constraints (`PRIMARY KEY`, `UNIQUE`, `FOREIGN KEY`).

#### CREATE INDEX
Erstellt einen neuen Index für eine Tabelle.

```sql
CREATE [UNIQUE] [ASC[ENDING] | DESC[ENDING]]
  INDEX indexname ON tablename
  {(col [, col …]) | COMPUTED BY (<expression>)}
```

**Wichtige Parameter:**
*   `UNIQUE`: Erstellt einen Unique Index (keine doppelten Werte erlaubt, außer NULLs gemäß SQL-Standard).
*   `ASC`/`DESC`: Definiert die Sortierrichtung des Index (Default ist `ASC`).
*   `indexname`: Eindeutiger Name für den Index.
*   `tablename`: Name der Tabelle, für die der Index erstellt wird.
*   `(col [, col ...])`: Liste der Spalten, auf denen der Index basiert (Multi-Segment-Index).
*   `COMPUTED BY (<expression>)`: Erstellt einen Index basierend auf einem Ausdruck (Expression Index). Nützlich für abgeleitete Suchen (z.B. `UPPER(Name)` für case-insensitive Suche).

**Hinweise:**
*   Indizes für `PRIMARY KEY`, `UNIQUE` und `FOREIGN KEY` Constraints werden automatisch erstellt. `CREATE INDEX` ist für benutzerdefinierte Indizes zur Performance-Optimierung.
*   Indizes können nicht auf `BLOB`- oder `ARRAY`-Spalten erstellt werden (aber auf Ausdrücken, die Teile davon extrahieren, falls möglich).
*   Die maximale Länge eines Indexschlüssels beträgt 1/4 der Page Size (minus 9 Bytes für Strings). Siehe Tabelle 7 im OCR.
*   Die maximale Anzahl Indizes pro Tabelle hängt von Page Size und Spaltenanzahl ab (siehe Tabelle 26 im OCR).
*   Nur der Tabellen-Owner und Administratoren können Indizes erstellen.

#### ALTER INDEX
Aktiviert oder deaktiviert einen bestehenden Index.

```sql
ALTER INDEX indexname {ACTIVE | INACTIVE}
```

**Hinweise:**
*   `INACTIVE`: Deaktiviert den Index. Er wird vom Optimizer nicht mehr verwendet und nicht mehr aktualisiert. Die Definition bleibt erhalten. Indizes von Constraints können nicht deaktiviert werden.
*   `ACTIVE`: Aktiviert einen inaktiven Index. Der Index wird neu aufgebaut (`REBUILD`). Kann auch zum Rebuild eines bereits aktiven Index verwendet werden (nützlich für Performance bei vielen Änderungen).
*   Nur der Tabellen-Owner und Administratoren können Indizes ändern.

#### DROP INDEX
Löscht einen bestehenden Index.

```sql
DROP INDEX indexname
```

**Hinweise:**
*   Löscht nur benutzerdefinierte Indizes. Indizes, die zu Constraints gehören, können nicht direkt gelöscht werden (sie werden mit `ALTER TABLE ... DROP CONSTRAINT` entfernt).
*   Nur der Tabellen-Owner und Administratoren können Indizes löschen.

#### SET STATISTICS
Berechnet die Selektivität eines Index neu.

```sql
SET STATISTICS INDEX indexname
```

**Hinweise:**
*   Index-Statistiken (Selektivität) werden vom Optimizer zur Planerstellung genutzt.
*   Statistiken werden bei `CREATE INDEX` und `ALTER INDEX ACTIVE` automatisch gesetzt.
*   Nach vielen Datenänderungen (INSERTS, UPDATES, DELETES) können Statistiken veraltet sein. `SET STATISTICS` aktualisiert sie manuell.
*   Kann unter Last ausgeführt werden, birgt aber die Gefahr, dass die neu berechneten Statistiken sofort wieder veraltet sind.
*   Nur der Tabellen-Owner und Administratoren können Statistiken neu berechnen.

### VIEW
Virtuelle Tabellen, definiert durch eine `SELECT`-Abfrage.

#### CREATE VIEW
Erstellt eine neue View.

```sql
CREATE VIEW viewname [(<full_column_list>)]
  AS <select_statement>
  [WITH CHECK OPTION]
```

**Wichtige Parameter:**
*   `viewname`: Eindeutiger Name für die View.
*   `<full_column_list>`: Optionale Liste der Spaltennamen für die View. Wenn weggelassen, werden die Namen/Aliase aus dem `SELECT`-Statement verwendet. Muss angegeben werden, wenn Spalten im SELECT keinen eindeutigen Namen haben (z.B. bei Ausdrücken ohne Alias). Anzahl muss mit SELECT-Liste übereinstimmen.
*   `<select_statement>`: Die `SELECT`-Abfrage, die die View definiert.
*   `WITH CHECK OPTION`: Nur für *updatable views*. Stellt sicher, dass `INSERT`- oder `UPDATE`-Operationen über die View nur Zeilen erzeugen/ändern, die der `WHERE`-Klausel der View entsprechen.

**Updatable Views:**
Eine View ist automatisch updatable, wenn:
1.  Das `SELECT` nur *eine* Tabelle oder *eine* updatable View abfragt.
2.  Keine Stored Procedures im `SELECT` aufgerufen werden.
3.  Alle Spalten der Basistabelle, die in der View *nicht* enthalten sind, entweder `NULL` erlauben, einen `DEFAULT`-Wert haben oder durch einen Trigger (`BEFORE INSERT`/`UPDATE`) versorgt werden.
4.  Keine abgeleiteten Felder (Subqueries, Ausdrücke) im `SELECT` enthalten sind.
5.  Keine Aggregatfunktionen (`MIN`, `MAX`, `AVG` etc.) verwendet werden.
6.  Kein `GROUP BY` oder `HAVING` verwendet wird.
7.  Kein `DISTINCT`, `ROWS`, `FIRST` oder `SKIP` verwendet wird.

**Hinweise:**
*   Views können durch Trigger updatebar gemacht werden, auch wenn sie die obigen Kriterien nicht erfüllen. Solche Trigger müssen die Änderungen manuell an die Basistabellen weiterleiten.
*   Der Ersteller der View wird ihr Owner. Er benötigt `SELECT`-Rechte auf die Basistabellen/-Views und `EXECUTE`-Rechte auf ggf. verwendete Prozeduren. Für Updatebarkeit benötigt er entsprechende `INSERT`/`UPDATE`/`DELETE`-Rechte.

#### ALTER VIEW
Ändert die Definition einer bestehenden View.

```sql
ALTER VIEW viewname [(<full_column_list>)]
  AS <select_statement>
  [WITH CHECK OPTION]
```

**Hinweise:**
*   Syntax ist identisch mit `CREATE VIEW`.
*   Ändert die Definition, behält aber existierende Rechte und Abhängigkeiten bei.
*   Nur der View-Owner und Administratoren können `ALTER VIEW` ausführen.

#### CREATE OR ALTER VIEW
Erstellt eine neue View oder ändert eine bestehende.

```sql
CREATE OR ALTER VIEW viewname [(<full_column_list>)]
  AS <select_statement>
  [WITH CHECK OPTION]
```

**Hinweise:**
*   Kombiniert `CREATE VIEW` und `ALTER VIEW`. Nützlich in Skripten.
*   Wenn die View existiert, wird sie geändert (wie `ALTER VIEW`). Wenn nicht, wird sie erstellt (wie `CREATE VIEW`).

#### DROP VIEW
Löscht eine bestehende View.

```sql
DROP VIEW viewname
```

**Hinweise:**
*   Fehlschlag, wenn Abhängigkeiten bestehen (andere Views, Prozeduren, Trigger etc.).
*   Nur der View-Owner und Administratoren können `DROP VIEW` ausführen.

#### RECREATE VIEW
Erstellt eine neue View oder ersetzt eine bestehende.

```sql
RECREATE VIEW viewname [(<full_column_list>)]
  AS <select_statement>
  [WITH CHECK OPTION]
```

**Hinweise:**
*   Versucht zuerst, die bestehende View via `DROP VIEW` zu löschen. Wenn dies wegen Abhängigkeiten fehlschlägt, schlägt auch `RECREATE VIEW` fehl (Commit-Zeitpunkt relevant).
*   Rechte auf die View gehen bei `RECREATE` verloren (im Gegensatz zu `ALTER`).

### TRIGGER
Gespeicherte Prozeduren, die automatisch bei bestimmten Datenänderungs- (DML) oder Datenbankereignissen ausgeführt werden.

#### CREATE TRIGGER
Erstellt einen neuen Trigger.

```sql
CREATE TRIGGER trigname
  { <relation_trigger_legacy>
  | <relation_trigger_sql2003>
  | <database_trigger> }
AS
  [<declarations>]
BEGIN
  [<PSQL_statements>]
END
```

**Relation Trigger (Legacy Syntax):**
```sql
FOR {tablename | viewname}
[ACTIVE | INACTIVE]
{BEFORE | AFTER} <mutation_list>
[POSITION number]
```

**Relation Trigger (SQL:2003 Syntax):**
```sql
[ACTIVE | INACTIVE]
{BEFORE | AFTER} <mutation_list>
[POSITION number]
ON {tablename | viewname}
```

**Database Trigger:**
```sql
[ACTIVE | INACTIVE]
ON <db_event>
[POSITION number]
```

**Komponenten:**
*   `trigname`: Eindeutiger Name des Triggers.
*   `ACTIVE`/`INACTIVE`: Status des Triggers (Default `ACTIVE`).
*   `BEFORE`/`AFTER`: Phase, wann der Trigger relativ zum Ereignis feuert.
*   `<mutation_list>`: DML-Ereignis(e) (`INSERT`, `UPDATE`, `DELETE`, kombiniert mit `OR`).
*   `POSITION number`: Ausführungsreihenfolge (0-32767) relativ zu anderen Triggern für dasselbe Ereignis/Phase/Objekt. Default 0. Kleinere Nummern zuerst.
*   `ON {tablename | viewname}`: Zieltabelle/-view für Relation Trigger.
*   `ON <db_event>`: Datenbankereignis für Database Trigger.
    ```sql
    <db_event> ::=
      { CONNECT | DISCONNECT
      | TRANSACTION START | TRANSACTION COMMIT | TRANSACTION ROLLBACK }
    ```
*   `AS ... BEGIN ... END`: PSQL-Codeblock (Header mit optionalen Deklarationen, Body mit Statements).

**Kontextvariablen in DML-Triggern:**
*   `OLD.colname`: Wert der Spalte *vor* der Änderung (nur bei `UPDATE`, `DELETE`; read-only).
*   `NEW.colname`: Wert der Spalte *nach* der Änderung (nur bei `INSERT`, `UPDATE`; read/write in `BEFORE`, read-only in `AFTER`).
*   `INSERTING`, `UPDATING`, `DELETING`: Boolesche Variablen, die anzeigen, welches Ereignis den Trigger ausgelöst hat (nützlich bei `OR`-Triggern).

**Database Trigger Ausführung:**
*   `CONNECT`/`DISCONNECT`: Eigene Transaktion (Commit bei Erfolg, Rollback bei Fehler). Fehler bei `DISCONNECT` werden nicht gemeldet.
*   `TRANSACTION START`/`COMMIT`/`ROLLBACK`: Werden innerhalb der zugehörigen Benutzertransaktion ausgeführt. Verhalten bei Fehlern hängt vom Ereignis ab.

#### ALTER TRIGGER
Ändert Status, Phase, Ereignisse, Position oder Code eines Triggers.

```sql
ALTER TRIGGER trigname
  [ACTIVE | INACTIVE]
  [{BEFORE | AFTER} <mutation_list> | ON <db_event>] -- Kann nicht zwischen Relation- und DB-Trigger wechseln
  [POSITION number]
  [ AS <declarations> BEGIN [<PSQL_statements>] END ]
```

**Hinweise:**
*   Lässt nicht definierte Teile unverändert.
*   Nur der Owner des Ziels (Tabelle/View/Datenbank) und Administratoren können Trigger ändern.

#### CREATE OR ALTER TRIGGER
Erstellt oder ändert einen Trigger.

```sql
CREATE OR ALTER TRIGGER trigname ... -- Rest wie CREATE TRIGGER
```

**Hinweise:**
*   Ändert einen existierenden Trigger ohne Verlust von Rechten oder Abhängigkeiten. Erstellt ihn, falls er nicht existiert.

#### DROP TRIGGER
Löscht einen Trigger.

```sql
DROP TRIGGER trigname
```

**Hinweise:**
*   Nur der Owner des Ziels und Administratoren können Trigger löschen.

#### RECREATE TRIGGER
Erstellt oder ersetzt einen Trigger.

```sql
RECREATE TRIGGER trigname ... -- Rest wie CREATE TRIGGER
```

**Hinweise:**
*   Versucht `DROP` dann `CREATE`. Abhängigkeiten verhindern `DROP` zur Commit-Zeit. Rechte gehen verloren.

Okay, hier ist der nächste Abschnitt der Zusammenfassung, der sich mit Stored Procedures, External Functions, Filtern, Sequences und Exceptions befasst.

```markdown
### PROCEDURE (Stored Procedure)
Wiederverwendbare Code-Module, die von Clients oder anderem PSQL-Code aufgerufen werden können.

#### CREATE PROCEDURE
Erstellt eine neue Stored Procedure.

```sql
CREATE PROCEDURE procname
  [(<inparam> [, <inparam> ...])]
  [RETURNS (<outparam> [, <outparam> ...])]
AS
  [<declarations>]
BEGIN
  [<PSQL_statements>]
END
```

**Parameterdefinitionen:**
```sql
<inparam>  ::= <param_decl> [{= | DEFAULT} <value>]
<outparam> ::= <param_decl>
<param_decl> ::= paramname <type> [NOT NULL] [COLLATE collation]
<type> ::= <datatype> | [TYPE OF] domain | TYPE OF COLUMN rel.col
<value> ::= {<literal> | NULL | <context_var>}
-- <datatype> siehe CREATE DOMAIN
-- <declarations> enthält DECLARE VARIABLE und DECLARE CURSOR
```

**Wichtige Punkte:**
*   `procname`: Eindeutiger Name unter allen Prozeduren, Tabellen, Views.
*   **Input Parameter (`<inparam>`):** In Klammern nach dem Namen. Können `DEFAULT`-Werte haben (müssen dann am Ende der Liste stehen). Werden als Werte übergeben (call-by-value).
*   **Output Parameter (`<outparam>`):** In der `RETURNS`-Klausel. Sind für *selectable* Prozeduren obligatorisch.
*   **Typdefinition:** Kann SQL-Typ, Domain (mit/ohne `TYPE OF`) oder `TYPE OF COLUMN` sein.
*   `AS ... BEGIN ... END`: PSQL-Codeblock.
*   **Typen:**
    *   **Executable Procedure:** Modifiziert meist Daten, gibt optional eine einzelne Zeile via `RETURNS` zurück. Wird mit `EXECUTE PROCEDURE` aufgerufen. Enthält kein `SUSPEND`.
    *   **Selectable Procedure:** Gibt eine Ergebnismenge (0 bis n Zeilen) zurück. Muss `RETURNS` und mindestens ein `SUSPEND` im Code haben. Wird mit `SELECT` aufgerufen.

**Hinweise:**
*   Jeder Benutzer kann Prozeduren erstellen. Der Ersteller ist der Owner.

#### ALTER PROCEDURE
Ändert eine bestehende Stored Procedure (Parameter, Deklarationen, Body).

```sql
ALTER PROCEDURE procname
  [(<inparam> [, <inparam> ...])]
  [RETURNS (<outparam> [, <outparam> ...])]
AS
  [<declarations>]
BEGIN
  [<PSQL_statements>]
END
```

**Hinweise:**
*   Syntax ist identisch mit `CREATE PROCEDURE`.
*   Ändert die Definition, behält aber existierende Rechte und Abhängigkeiten bei (aber Vorsicht bei Parameteränderungen, siehe RDB$VALID_BLR).
*   Nur der Owner und Administratoren können Prozeduren ändern.

#### CREATE OR ALTER PROCEDURE
Erstellt oder ändert eine Stored Procedure.

```sql
CREATE OR ALTER PROCEDURE procname ... -- Rest wie CREATE PROCEDURE
```

**Hinweise:**
*   Kombiniert `CREATE` und `ALTER`. Behält Rechte und Abhängigkeiten bei Änderung bei.

#### DROP PROCEDURE
Löscht eine bestehende Stored Procedure.

```sql
DROP PROCEDURE procname
```

**Hinweise:**
*   Fehlschlag bei Abhängigkeiten.
*   Nur der Owner und Administratoren können Prozeduren löschen.

#### RECREATE PROCEDURE
Erstellt oder ersetzt eine Stored Procedure.

```sql
RECREATE PROCEDURE procname ... -- Rest wie CREATE PROCEDURE
```

**Hinweise:**
*   Versucht `DROP` dann `CREATE`. Abhängigkeiten verhindern `DROP` zur Commit-Zeit. Rechte gehen verloren.

### EXTERNAL FUNCTION (UDF)
Benutzerdefinierte Funktionen, implementiert in externen Bibliotheken (DLLs/Shared Objects).

#### DECLARE EXTERNAL FUNCTION
Deklariert eine UDF für die Datenbank.

```sql
DECLARE EXTERNAL FUNCTION funcname
  [<arg_type_decl> [, <arg_type_decl> ...]]
  RETURNS { <sqltype> [BY {DESCRIPTOR | VALUE}] | CSTRING(length) | PARAMETER param_num }
  [FREE_IT]
  ENTRY_POINT 'entry_point' MODULE_NAME 'library_name'
```

**Argumentdeklaration (`<arg_type_decl>`):**
```sql
  <sqltype> [{BY DESCRIPTOR} | NULL]
  | CSTRING(length) [NULL]
```

**Wichtige Parameter:**
*   `funcname`: Name der Funktion in der Datenbank (eindeutig unter Funktionen).
*   `<arg_type_decl>`: Definition der Eingabeparameter.
    *   `<sqltype>`: SQL-Datentyp (kein Array). Default: Übergabe per Referenz. `BY DESCRIPTOR` erlaubt NULL-Übergabe.
    *   `CSTRING(length)`: Null-terminierter String fester maximaler Länge.
    *   `NULL`: Parameter wird immer als NULL übergeben.
*   `RETURNS`: Definition des Rückgabewerts.
    *   `<sqltype>`: SQL-Datentyp. Default: Übergabe per Referenz. `BY VALUE` für Wertübergabe (nicht alle Typen). `BY DESCRIPTOR` für Rückgabe per Deskriptor (erlaubt NULL).
    *   `CSTRING(length)`: Null-terminierter String.
    *   `PARAMETER param_num`: Gibt den Wert des n-ten *Eingabe*-Parameters zurück (nützlich für BLOBs).
*   `FREE_IT`: Signalisiert, dass die UDF dynamisch Speicher allokiert (mit `ib_util_malloc`), der von Firebird freigegeben werden soll.
*   `ENTRY_POINT 'entry_point'`: Exportierter Name der Funktion in der Bibliothek.
*   `MODULE_NAME 'library_name'`: Name der Bibliothek (ohne Pfad/Endung). Muss im UDF-Verzeichnis liegen oder Pfad muss in `firebird.conf` konfiguriert sein (`UDFAccess`).

**Hinweise:**
*   UDFs müssen in *jeder* Datenbank deklariert werden, die sie nutzen soll.
*   Jeder Benutzer kann UDFs deklarieren.

#### ALTER EXTERNAL FUNCTION
Ändert den Entry Point oder den Modulnamen einer deklarierten UDF.

```sql
ALTER EXTERNAL FUNCTION funcname
  [ENTRY_POINT 'new_entry_point']
  [MODULE_NAME 'new_library_name']
```

**Hinweise:**
*   Ändert *nicht* die Parameter oder Rückgabetypen.
*   Jeder Benutzer kann dies tun.

#### DROP EXTERNAL FUNCTION
Entfernt die Deklaration einer UDF aus der Datenbank.

```sql
DROP EXTERNAL FUNCTION funcname
```

**Hinweise:**
*   Fehlschlag bei Abhängigkeiten.
*   Jeder Benutzer kann dies tun.

### FILTER (BLOB Filter)
Spezielle externe Funktionen zur Konvertierung zwischen BLOB-Subtypen.

#### DECLARE FILTER
Deklariert einen BLOB-Filter für die Datenbank.

```sql
DECLARE FILTER filtername
  INPUT_TYPE <sub_type> OUTPUT_TYPE <sub_type>
  ENTRY_POINT 'function_name' MODULE_NAME 'library_name'
```

**Wichtige Parameter:**
*   `filtername`: Eindeutiger Name des Filters.
*   `INPUT_TYPE <sub_type>`: Quell-BLOB-Subtyp (Nummer oder Name).
*   `OUTPUT_TYPE <sub_type>`: Ziel-BLOB-Subtyp (Nummer oder Name).
*   `ENTRY_POINT`, `MODULE_NAME`: Wie bei UDFs.

**Subtypen (`<sub_type>`):**
```sql
number | <mnemonic>
<mnemonic> ::= BINARY | TEXT | BLR | ACL | RANGES | SUMMARY | FORMAT
             | TRANSACTION_DESCRIPTION | EXTERNAL_FILE_DESCRIPTION | user_defined
```

**Hinweise:**
*   Filter müssen für jede Quell-/Ziel-Subtyp-Kombination eindeutig sein.
*   Benutzerdefinierte Subtypen müssen negative Nummern haben und können optional über `INSERT INTO RDB$TYPES` mit mnemonischen Namen verknüpft werden (vor FB3).
*   Jeder Benutzer kann Filter deklarieren.

#### DROP FILTER
Entfernt die Deklaration eines BLOB-Filters.

```sql
DROP FILTER filtername
```

**Hinweise:**
*   Jeder Benutzer kann Filter löschen.

### SEQUENCE (GENERATOR)
Objekte zur Generierung eindeutiger Ganzzahlwerte (64-bit).

#### CREATE SEQUENCE (GENERATOR)
Erstellt eine neue Sequence (oder Generator).

```sql
CREATE {SEQUENCE | GENERATOR} seq_name
```

**Hinweise:**
*   `SEQUENCE` und `GENERATOR` sind Synonyme. `SEQUENCE` ist SQL-konform.
*   Startwert ist immer 0.
*   Werte werden mit `NEXT VALUE FOR seq_name` (empfohlen, +1 Inkrement) oder `GEN_ID(seq_name, step)` (beliebiges Inkrement, 0 für aktuellen Wert) abgerufen.
*   Jeder Benutzer kann Sequences erstellen.

#### ALTER SEQUENCE
Setzt den aktuellen Wert einer Sequence.

```sql
ALTER SEQUENCE seq_name RESTART WITH new_val
```

**Wichtige Parameter:**
*   `new_val`: Neuer 64-bit Integer-Wert für die Sequence.

**Hinweise:**
*   Ändert den *aktuellen* Wert, von dem aus weiter inkrementiert wird.
*   Vorsicht: Kann die Eindeutigkeit gefährden, wenn falsch verwendet.
*   Jeder Benutzer kann Sequences ändern.

#### SET GENERATOR
(Veraltet) Setzt den aktuellen Wert eines Generators.

```sql
SET GENERATOR seq_name TO new_val
```

**Hinweise:**
*   Funktional identisch mit `ALTER SEQUENCE ... RESTART WITH`.
*   Sollte durch `ALTER SEQUENCE` ersetzt werden.

#### DROP SEQUENCE (GENERATOR)
Löscht eine Sequence (oder Generator).

```sql
DROP {SEQUENCE | GENERATOR} seq_name
```

**Hinweise:**
*   `SEQUENCE` und `GENERATOR` sind Synonyme.
*   Fehlschlag bei Abhängigkeiten (z.B. wenn in Trigger- oder Prozedurcode verwendet).
*   Jeder Benutzer kann Sequences löschen.

### EXCEPTION
Benutzerdefinierte Fehlerobjekte für PSQL.

#### CREATE EXCEPTION
Erstellt eine neue benutzerdefinierte Exception.

```sql
CREATE EXCEPTION exception_name 'message'
```

**Wichtige Parameter:**
*   `exception_name`: Eindeutiger Name der Exception (max. 31 Zeichen).
*   `message`: Standard-Fehlermeldung (max. 1021 Bytes, CHARACTER SET NONE).

**Hinweise:**
*   Wird in PSQL mit `EXCEPTION exception_name [optional_override_message];` geworfen.
*   Kann in `WHEN`-Klauseln abgefangen werden.
*   Jeder Benutzer kann Exceptions erstellen.

#### ALTER EXCEPTION
Ändert die Standard-Fehlermeldung einer Exception.

```sql
ALTER EXCEPTION exception_name 'new_message'
```

**Hinweise:**
*   Jeder Benutzer kann Exceptions ändern.

#### CREATE OR ALTER EXCEPTION
Erstellt oder ändert eine Exception.

```sql
CREATE OR ALTER EXCEPTION exception_name 'message'
```

**Hinweise:**
*   Ändert eine existierende Exception oder erstellt sie neu. Abhängigkeiten bleiben bei Änderung erhalten.

#### DROP EXCEPTION
Löscht eine benutzerdefinierte Exception.

```sql
DROP EXCEPTION exception_name
```

**Hinweise:**
*   Fehlschlag bei Abhängigkeiten (wenn in PSQL-Code verwendet).
*   Jeder Benutzer kann Exceptions löschen.

#### RECREATE EXCEPTION
Erstellt oder ersetzt eine Exception.

```sql
RECREATE EXCEPTION exception_name 'message'
```

**Hinweise:**
*   Versucht `DROP` dann `CREATE`. Abhängigkeiten verhindern `DROP` zur Commit-Zeit.

### COLLATION
Definition von Sortierreihenfolgen und Vergleichsregeln für Zeichensätze.

#### CREATE COLLATION
Macht eine bereits systemweit verfügbare Collation in der Datenbank bekannt.

```sql
CREATE COLLATION collname
  FOR charset
  [FROM basecoll | FROM EXTERNAL ('extname')]
  [NO PAD | PAD SPACE]
  [CASE [IN]SENSITIVE]
  [ACCENT [IN]SENSITIVE]
  ['<specific-attributes>']
```

**Wichtige Parameter:**
*   `collname`: Name der Collation in der Datenbank.
*   `charset`: Zeichensatz, für den die Collation gilt.
*   `FROM basecoll`: Basiert die neue Collation auf einer bereits existierenden (`basecoll`) in der DB, überschreibt aber Attribute.
*   `FROM EXTERNAL ('extname')`: Macht eine Collation aus einer externen Bibliothek (definiert in `.conf`-Dateien im `intl`-Verzeichnis) unter dem Namen `collname` verfügbar. `'extname'` ist der case-sensitive Name aus der Konfigurationsdatei. Wenn `FROM` fehlt, wird `FROM EXTERNAL ('collname')` angenommen.
*   `NO PAD`/`PAD SPACE`: Verhalten bezüglich Leerzeichen am Ende (Default: abhängig von `basecoll` oder externer Definition).
*   `CASE [IN]SENSITIVE`: Groß-/Kleinschreibung (Default: abhängig von `basecoll` oder externer Definition).
*   `ACCENT [IN]SENSITIVE`: Akzentbehandlung (Default: abhängig von `basecoll` oder externer Definition).
*   `<specific-attributes>`: Zusätzliche, Collation-spezifische Attribute (z.B. `'NUMERIC-SORT=1'`). Siehe Tabelle 57 im OCR.

**Hinweise:**
*   Erstellt keine neue Collation-Logik, sondern registriert eine vorhandene.
*   Jeder Benutzer kann Collation-Definitionen hinzufügen.

#### DROP COLLATION
Entfernt eine Collation-Definition aus der Datenbank.

```sql
DROP COLLATION collname
```

**Hinweise:**
*   Fehlschlag, wenn die Collation von Spalten, Domains etc. verwendet wird.
*   Jeder Benutzer kann Collation-Definitionen entfernen.

### CHARACTER SET

#### ALTER CHARACTER SET
Ändert die *Standard*-Collation für einen gegebenen Zeichensatz in der Datenbank.

```sql
ALTER CHARACTER SET charset
  SET DEFAULT COLLATION collation
```

**Hinweise:**
*   Ändert nur den Default für *zukünftige* Verwendungen des Zeichensatzes, wenn keine explizite `COLLATE`-Klausel angegeben wird.
*   Bestehende Objekte behalten ihre ursprüngliche Collation.

### ROLE
Ein Container für SQL-Privilegien zur einfacheren Rechteverwaltung.

#### CREATE ROLE
Erstellt eine neue Rolle.

```sql
CREATE ROLE rolename
```

**Hinweise:**
*   `rolename`: Eindeutiger Name unter allen Rollen (und idealerweise auch Benutzern).
*   Rechte werden der Rolle separat mit `GRANT` zugewiesen.
*   Die Rolle wird Benutzern mit `GRANT rolename TO username` zugewiesen.
*   Ein Benutzer muss sich explizit mit einer Rolle verbinden, um deren Rechte zu nutzen.
*   Jeder Benutzer kann Rollen erstellen und wird deren Owner.

#### ALTER ROLE
(Spezialfall) Konfiguriert das `AUTO ADMIN MAPPING` für die `RDB$ADMIN`-Rolle.

```sql
ALTER ROLE RDB$ADMIN
  { SET AUTO ADMIN MAPPING | DROP AUTO ADMIN MAPPING }
```

**Hinweise:**
*   Nur relevant für Windows-Administratoren bei Trusted Authentication.
*   `SET`: Erlaubt Windows-Admins, sich automatisch mit `RDB$ADMIN`-Rechten in dieser DB zu verbinden (wenn ohne Rolle verbunden).
*   `DROP`: Deaktiviert das automatische Mapping für diese DB.
*   Nur der DB-Owner und Administratoren können dies ausführen.

#### DROP ROLE
Löscht eine Rolle.

```sql
DROP ROLE rolename
```

**Hinweise:**
*   Entfernt die Rolle und widerruft sie implizit von allen Benutzern/Objekten, denen sie zugewiesen war.
*   Nur der Rollen-Owner und Administratoren können Rollen löschen.

### COMMENTS

#### COMMENT ON
Fügt beschreibende Kommentare zu Datenbankobjekten hinzu oder entfernt sie.

```sql
COMMENT ON <object> IS {'sometext' | NULL}
```

**Objekttypen (`<object>`):**
```sql
DATABASE
| <basic-type> objectname
| COLUMN relationname.fieldname
| PARAMETER procname.paramname

<basic-type> ::= CHARACTER SET | COLLATION | DOMAIN | EXCEPTION
               | EXTERNAL FUNCTION | FILTER | GENERATOR | INDEX
               | PROCEDURE | ROLE | SEQUENCE | TABLE | TRIGGER | VIEW
```

**Hinweise:**
*   `'sometext'`: Der Kommentartext (max. Länge variiert je nach Objekt, gespeichert in `RDB$DESCRIPTION` der jeweiligen Systemtabelle).
*   `NULL` oder `''`: Entfernt den Kommentar für das Objekt.
*   Kommentare überleben Backup/Restore.
*   Der Owner des Objekts und Administratoren können Kommentare setzen/ändern.

Okay, hier ist der nächste Teil der Zusammenfassung, der die Data Manipulation Language (DML) Statements abdeckt.

```markdown
## Data Manipulation (DML) Statements

DML-Anweisungen werden verwendet, um Daten in Datenbanktabellen und Views abzurufen, einzufügen, zu ändern und zu löschen.

### SELECT Statement
Ruft Daten aus der Datenbank ab.

**Globale Syntax:**

```sql
[WITH [RECURSIVE] <cte> [, <cte> ...]]
SELECT
  [FIRST m] [SKIP n]  -- Firebird spezifisch, siehe ROWS
  [DISTINCT | ALL] <columns>
FROM
  <source> [[AS] alias]
  [<joins>]
[WHERE <condition>]
[GROUP BY <grouping-list> [HAVING <aggregate-condition>]]
[PLAN <plan-expr>]
[UNION [DISTINCT | ALL] <other-select>]
[ORDER BY <ordering-list>]
[ROWS m [TO n]]  -- SQL Standard
[FOR UPDATE [OF <columns>]]
[WITH LOCK]
[INTO <variables>] -- Nur PSQL
```

**Wichtige Klauseln und Konzepte:**

#### FIRST, SKIP (Firebird-spezifisch)
*   `FIRST m`: Beschränkt die Ausgabe auf die ersten `m` Zeilen.
*   `SKIP n`: Überspringt die ersten `n` Zeilen der Ausgabe.
*   Kombiniert (`FIRST m SKIP n`): Überspringt `n` Zeilen, gibt dann die nächsten `m` Zeilen zurück.
*   Argumente müssen Integer-Literale, Parameter (`?`/:name) oder geklammerte Integer-Ausdrücke sein.
*   Wird durch die SQL-Standard-Klausel `ROWS` ersetzt/ergänzt.
*   Vorsicht bei `FIRST` in Subqueries von `IN`-Prädikaten bei `DELETE`/`UPDATE` (kann zu unerwartetem Verhalten führen, "unstable cursor").

#### The SELECT Columns List
*   Definiert die Spalten, die zurückgegeben werden sollen.
*   Kann sein:
    *   `*`: Alle Spalten der Quelle(n). Muss qualifiziert sein (`alias.*`), wenn nicht die einzige Quelle oder wenn mehrdeutig.
    *   Spaltennamen (`colname`, `alias.colname`). Qualifizierung nötig bei Mehrdeutigkeit (Joins).
    *   Ausdrücke (arithmetisch, String, Funktionsaufrufe, `CASE`, Subqueries, etc.). Sollten einen Alias (`AS alias`) erhalten.
    *   Literale, Kontextvariablen.
*   `DISTINCT`: Gibt nur eindeutige Zeilen zurück (Duplikate werden entfernt).
*   `ALL` (Default): Gibt alle Zeilen zurück, einschließlich Duplikaten.
*   `COLLATE collation`: Beeinflusst Sortierung und Vergleiche von String-Spalten, nicht die Ausgabe selbst.

#### The FROM Clause
*   Spezifiziert die Datenquelle(n).
*   Kann sein:
    *   Tabelle (`tablename [[AS] alias]`)
    *   View (`viewname [[AS] alias]`)
    *   Selectable Stored Procedure (`procname [(<args>)] [[AS] alias]`)
    *   Derived Table (`(<select-statement>) [[AS] alias] [(<column-aliases>)]`) - Eine Subquery, die wie eine Tabelle behandelt wird. Jede Spalte muss einen Namen haben.
    *   Common Table Expression (CTE) (siehe `WITH` Clause unten).
*   Aliase (`AS alias`) sind optional, aber oft nützlich oder notwendig (bei Subqueries, Self-Joins, oder wenn der Originalname in der Abfrage mehrdeutig ist). Wenn ein Alias vergeben wird, *muss* er anstelle des Originalnamens verwendet werden.

#### Joins
*   Kombinieren Daten aus zwei oder mehr Quellen.
*   **Typen:**
    *   `INNER JOIN` (oder nur `JOIN`): Gibt nur Zeilen zurück, für die die Join-Bedingung in beiden Quellen erfüllt ist (Schnittmenge).
    *   `LEFT [OUTER] JOIN`: Gibt alle Zeilen der linken Quelle zurück, plus passende Zeilen der rechten Quelle (oder NULLs, falls kein Match).
    *   `RIGHT [OUTER] JOIN`: Gibt alle Zeilen der rechten Quelle zurück, plus passende Zeilen der linken Quelle (oder NULLs, falls kein Match).
    *   `FULL [OUTER] JOIN`: Gibt alle Zeilen beider Quellen zurück (mit NULLs für fehlende Matches).
    *   `CROSS JOIN` (oder Komma `,` zwischen Quellen): Bildet das kartesische Produkt aller Zeilen beider Quellen. Keine `ON`/`USING`-Klausel erlaubt. Komma-Syntax vermeiden.
*   **Join-Bedingungen:**
    *   `ON <condition>`: Explizite Bedingung (beliebiger boolescher Ausdruck, meist ein Vergleich zwischen Spalten der Quellen).
    *   `USING (<column-list>)`: Shorthand für Equi-Joins bei gleichnamigen Spalten. Die genannten Spalten erscheinen nur einmal im Ergebnis (verschmolzen). Vorsicht bei OUTER Joins und `SELECT *`.
    *   `NATURAL JOIN`: Automatischer Equi-Join über *alle* Spalten, die in beiden Quellen den gleichen Namen haben. Datentypen müssen kompatibel sein. Keine `ON`/`USING`-Klausel erlaubt. Vorsicht: riskant, wenn sich Spaltennamen ändern. Nicht in Dialekt 1.
*   **Hinweis zu `NULL`:** Standard-Joins (`ON a = b`, `USING(a)`, `NATURAL`) behandeln `NULL` nicht als gleich. Verwende `ON a IS NOT DISTINCT FROM b`, um `NULL`s zu matchen.

#### The WHERE Clause
*   Filtert Zeilen *bevor* Gruppierung und Aggregation stattfindet.
*   `<search-condition>`: Ein boolescher Ausdruck, der für jede Zeile ausgewertet wird. Nur Zeilen, für die der Ausdruck TRUE ergibt, werden weiterverarbeitet.
*   Kann Parameter (`?` oder `:name`) und lokale Variablen (`:var`) enthalten.

#### The GROUP BY Clause
*   Gruppiert Zeilen mit identischen Werten in den angegebenen Spalten/Ausdrücken zu einer einzigen Ergebniszeile.
*   Wird typischerweise mit Aggregatfunktionen (`COUNT`, `SUM`, `AVG`, `MAX`, `MIN`, `LIST`) in der `SELECT`-Liste verwendet, die dann pro Gruppe ausgeführt werden.
*   **Regel:** Jede Spalte in der `SELECT`-Liste, die *keine* Aggregatfunktion ist, *muss* in der `GROUP BY`-Klausel aufgeführt sein (als Spaltenname, Alias, Position oder exakter Ausdruck).
*   Die Gruppierung kann auch nach Spalten erfolgen, die nicht in der `SELECT`-Liste stehen, oder nach konstanten Ausdrücken (obwohl letzteres meist sinnlos ist).

#### The HAVING Clause
*   Filtert Gruppen *nachdem* Gruppierung und Aggregation stattgefunden haben.
*   Bedingung kann Aggregatfunktionen, Spalten aus der `GROUP BY`-Liste und konstante Ausdrücke enthalten.
*   Kann *keine* Spalten enthalten, die nicht aggregiert sind und nicht in `GROUP BY` stehen.
*   Filterung nach nicht-aggregierten Spalten sollte effizienter in der `WHERE`-Klausel erfolgen.

#### The PLAN Clause
*   Ermöglicht dem Benutzer, den Ausführungsplan des Optimizers manuell festzulegen.
*   Nützlich zur Performance-Analyse und -Optimierung komplexer Abfragen.
*   Syntax umfasst `NATURAL` (Table Scan), `INDEX` (Index-Zugriff), `ORDER` (Sortierung via Index), `SORT` (explizite Sortierung), `JOIN` (Join-Reihenfolge und Methode), `MERGE` (Merge-Join).
*   Pläne können verschachtelt sein.
*   Syntax ist komplex, siehe Abschnitt 6.1.7 im OCR für Details.

#### UNION Clause
*   Kombiniert die Ergebnismengen von zwei oder mehr `SELECT`-Statements.
*   Alle `SELECT`-Statements müssen dieselbe Anzahl Spalten mit kompatiblen Datentypen an entsprechenden Positionen haben.
*   `UNION [DISTINCT]` (Default): Entfernt doppelte Zeilen aus dem Gesamtergebnis.
*   `UNION ALL`: Behält alle Zeilen, einschließlich Duplikaten, bei (effizienter).
*   Spaltennamen werden vom *ersten* `SELECT` übernommen.
*   Eine globale `ORDER BY`-Klausel (nur nach Spaltenposition) und `ROWS`-Klausel kann *am Ende* des gesamten `UNION`-Konstrukts stehen.

#### The ORDER BY Clause
*   Sortiert die finale Ergebnismenge.
*   `<ordering-item>`: Spaltenname (wenn kein Alias im `SELECT`), Spaltenalias, Spaltenposition (1-basiert) oder Ausdruck.
*   `COLLATE collation`: Spezifiziert die Collation für String-Sortierung.
*   `ASC` (Default) / `DESC`: Sortierrichtung.
*   `NULLS FIRST` (Default) / `NULLS LAST`: Positionierung von NULL-Werten.
*   Bei `UNION`: `ORDER BY` nur am Ende erlaubt, Sortierung nur nach Spaltenposition (Integer-Literal).

#### ROWS Clause (SQL Standard)
*   Beschränkt die Anzahl der zurückgegebenen Zeilen nach der Sortierung.
*   `ROWS m`: Gibt die ersten `m` Zeilen zurück (Äquivalent zu `FIRST m`).
*   `ROWS m TO n`: Gibt die Zeilen von `m` bis einschließlich `n` zurück (Äquivalent zu `FIRST (n - m + 1) SKIP (m - 1)`).
*   Argumente `m`, `n` können beliebige Integer-Ausdrücke sein (keine extra Klammern nötig).
*   Kann auch in `UPDATE` und `DELETE` verwendet werden (oft mit `ORDER BY`).

#### FOR UPDATE [OF] Clause
*   In FB <= 2.5: Deaktiviert nur den Prefetch-Buffer für den Cursor. Hat keine Sperrwirkung.
*   Die `OF`-Subklausel hat keine Funktion.
*   Verhalten könnte sich in zukünftigen Versionen ändern.

#### WITH LOCK Clause
*   Ermöglicht explizites pessimistisches Sperren von Zeilen (Expertenfunktion!).
*   Nur in Top-Level `SELECT` auf *einer einzelnen Tabelle*. Nicht mit Joins, Views, Aggregationen, `DISTINCT`, `UNION` etc.
*   Sperrt die selektierten Zeilen für Schreibzugriffe anderer Transaktionen bis zum Ende der eigenen Transaktion.
*   Verhalten bei Konflikten hängt von den Transaktionsparametern ab (WAIT/NOWAIT, Isolation Level). Siehe Tabelle 70.
*   In Kombination mit `FOR UPDATE` wird Zeile für Zeile beim `FETCH` gesperrt (Buffered Fetch deaktiviert).

#### INTO Clause (Nur PSQL)
*   Wird am Ende eines *Singleton SELECT* (muss 0 oder 1 Zeile zurückgeben) in PSQL verwendet, um die Spaltenwerte in lokale PSQL-Variablen zu laden.
*   Die Anzahl, Reihenfolge und Typen der Variablen müssen mit den Spalten übereinstimmen.
*   Der Doppelpunkt `:` vor Variablennamen ist optional.

```sql
SELECT Col1, Col2 FROM MyTable WHERE ID = :id INTO :var1, :var2;
```

#### WITH Clause (Common Table Expressions - CTE)
*   Definiert eine oder mehrere benannte temporäre Ergebnismengen (CTEs) in einem Präambel vor dem Haupt-`SELECT` (oder `INSERT`, `UPDATE`, `MERGE`, `DELETE`).
*   Syntax: `WITH [RECURSIVE] cte_name [(col_aliases)] AS (<cte_select>), ... <main_query>`
*   Verbessert Lesbarkeit und Struktur komplexer Abfragen. Erlaubt Rekursion.
*   **Rekursive CTEs:** Benötigen `RECURSIVE`, bestehen aus einem oder mehreren nicht-rekursiven *Anker*-Members (`UNION [DISTINCT | ALL]`) gefolgt von einem oder mehreren rekursiven Members (`UNION ALL`), die sich selbst referenzieren (genau einmal, im `FROM`). Maximale Rekursionstiefe ist 1024. Aggregate sind im rekursiven Teil nicht erlaubt.

### INSERT Statement
Fügt neue Zeilen in eine Tabelle oder eine updatable View ein.

```sql
INSERT INTO target [(<column_list>)]
  { DEFAULT VALUES | VALUES (<value_list>) | <select_stmt> }
  [RETURNING <returning_list> [INTO <variables>]] -- Nur für Einzelzeilen-Inserts
```

**Formen:**
*   `INSERT ... VALUES (<value_list>)`: Fügt genau eine Zeile mit den angegebenen Werten ein. Die Werte müssen mit `<column_list>` (falls vorhanden) oder allen Spalten der Tabelle (außer Computed) in Reihenfolge und Typ übereinstimmen.
*   `INSERT ... SELECT <select_stmt>`: Fügt null oder mehr Zeilen basierend auf dem Ergebnis der `SELECT`-Anweisung ein. Die Spalten des `SELECT` müssen mit `<column_list>` (falls vorhanden) oder allen Spalten der Tabelle übereinstimmen. **Vorsicht:** `INSERT INTO T SELECT * FROM T` führt in FB <= 2.5 zu einer Endlosschleife ("unstable cursor").
*   `INSERT ... DEFAULT VALUES`: Fügt genau eine Zeile ein, wobei für alle Spalten entweder `NULL` (falls erlaubt), der definierte `DEFAULT`-Wert oder ein Wert aus einem `BEFORE INSERT`-Trigger verwendet wird. Alle `NOT NULL`-Spalten ohne Default müssen durch einen Trigger versorgt werden.

**Klauseln:**
*   `<column_list>`: Optionale Liste der Zielspalten. Wenn weggelassen, müssen Werte für alle Spalten (außer Computed) bereitgestellt werden.
*   `RETURNING`: (Nur bei `VALUES` oder Singleton `SELECT`) Gibt Werte aus der eingefügten Zeile zurück (nach Ausführung von `BEFORE INSERT`-Triggern). Kann Spaltennamen, Ausdrücke oder `NEW.colname` enthalten. In PSQL kann `INTO :vars` verwendet werden.
*   **BLOBs:** Einfügen von BLOBs erfordert spezielle API-Behandlung oder `INSERT ... SELECT` oder Einfügen von kurzen (<32KB) String-Literalen.

### UPDATE Statement
Ändert Werte in bestehenden Zeilen einer Tabelle oder updatable View.

```sql
UPDATE target [[AS] alias]
  SET col = <value> [, col = <value> ...]
  [WHERE {<search-conditions> | CURRENT OF cursorname}] -- CURRENT OF nur PSQL/ESQL
  [PLAN <plan_items>]
  [ORDER BY <sort_items>]
  [ROWS m [TO n]]
  [RETURNING <returning_list> [INTO <variables>]] -- Nur für Einzelzeilen-Updates
```

**Klauseln:**
*   `target [[AS] alias]`: Die zu ändernde Tabelle/View, optional mit Alias.
*   `SET col = <value>, ...`: Definiert die Zuweisungen. `col` ist der Spaltenname, `<value>` ist ein Ausdruck, der den neuen Wert liefert. Eine Spalte darf nur einmal in `SET` vorkommen. Der *alte* Wert einer Spalte wird für Ausdrücke auf der rechten Seite verwendet, auch wenn die Spalte zuvor in derselben `SET`-Klausel geändert wurde (behoben in FB 2.5, konfigurierbar via `OldSetClauseSemantics`).
*   `WHERE`: Filtert die zu ändernden Zeilen.
    *   `<search-conditions>`: Normaler Suchausdruck.
    *   `CURRENT OF cursorname`: (Positioned Update) Ändert nur die Zeile, auf der der deklarierte Cursor aktuell steht (nur PSQL/ESQL).
*   `PLAN`: Optionaler Ausführungsplan.
*   `ORDER BY` / `ROWS`: Begrenzt die Anzahl der geänderten Zeilen basierend auf einer Sortierreihenfolge (nützlich z.B. für "Update top N").
*   `RETURNING`: Gibt Werte aus der geänderten Zeile zurück (nach `BEFORE UPDATE`, vor `AFTER UPDATE`). Kann `OLD.colname`, `NEW.colname`, andere Spalten und Ausdrücke enthalten. In PSQL kann `INTO :vars` verwendet werden. Nur für Updates, die maximal eine Zeile betreffen.
*   **BLOBs:** Update ersetzt immer den gesamten BLOB-Inhalt.

### UPDATE OR INSERT Statement
Fügt eine Zeile ein oder aktualisiert eine oder mehrere bestehende Zeilen.

```sql
UPDATE OR INSERT INTO target [(<column_list>)]
  VALUES (<value_list>)
  [MATCHING (<match_column_list>)]
  [RETURNING <values> [INTO <variables>]]
```

**Funktionsweise:**
1.  Prüft, ob Zeilen in `target` existieren, bei denen die Werte in den `MATCHING`-Spalten (oder im Primary Key, falls `MATCHING` fehlt) mit den entsprechenden Werten in `VALUES` übereinstimmen (`IS NOT DISTINCT FROM`-Vergleich, d.h. NULLs matchen).
2.  **Wenn Match gefunden:** Führt `UPDATE` für die übereinstimmende(n) Zeile(n) aus, wobei die Spalten aus `<column_list>` mit den Werten aus `<value_list>` gesetzt werden. **Achtung:** Wenn *mehrere* Zeilen matchen, werden alle aktualisiert (SQL-Standard würde Fehler erwarten).
3.  **Wenn kein Match gefunden:** Führt `INSERT` aus, wobei die Spalten aus `<column_list>` mit den Werten aus `<value_list>` gefüllt werden.

**Hinweise:**
*   `MATCHING` ist Pflicht, wenn die Tabelle keinen Primary Key hat.
*   `RETURNING`-Klausel ist möglich, gibt aber nur Werte zurück, wenn genau *eine* Zeile aktualisiert wurde oder eine neue Zeile eingefügt wurde. Bei mehreren Updates wird ein Fehler ausgelöst (in DSQL).

### DELETE Statement
Entfernt Zeilen aus einer Tabelle oder einer updatable View.

```sql
DELETE FROM target [[AS] alias]
  [WHERE {<search-conditions> | CURRENT OF cursorname}] -- CURRENT OF nur PSQL/ESQL
  [PLAN <plan_items>]
  [ORDER BY <sort_items>]
  [ROWS m [TO n]]
  [RETURNING <returning_list> [INTO <variables>]] -- Nur für Einzelzeilen-Deletes
```

**Klauseln:**
*   `target [[AS] alias]`: Die Tabelle/View, aus der gelöscht wird.
*   `WHERE`: Filtert die zu löschenden Zeilen (wie bei `UPDATE`).
*   `PLAN`: Optionaler Ausführungsplan.
*   `ORDER BY` / `ROWS`: Begrenzt die Anzahl der gelöschten Zeilen (wie bei `UPDATE`).
*   `RETURNING`: Gibt Werte aus der gelöschten Zeile zurück. Kann Spaltennamen (entspricht `OLD.colname`), Ausdrücke enthalten. In PSQL kann `INTO :vars` verwendet werden. Nur für Deletes, die maximal eine Zeile betreffen.

### MERGE Statement
Synchronisiert Daten zwischen einer Quell- und einer Zieltabelle/-View basierend auf einer Join-Bedingung.

```sql
MERGE INTO target [[AS] target-alias]
  USING <source> [[AS] source-alias]
  ON <join-condition>
  [ WHEN MATCHED THEN UPDATE SET col = <value> [, col = <value> ...]]
  [ WHEN NOT MATCHED THEN INSERT [(<columns>)] VALUES (<values>)]
```

**Funktionsweise:**
1.  Führt einen konzeptuellen Join zwischen `target` und `source` basierend auf `ON <join-condition>` durch.
2.  **Für jede Zeile aus `source`, die einen Match in `target` hat:** Wenn `WHEN MATCHED THEN UPDATE` vorhanden ist, wird die `UPDATE`-Operation auf der `target`-Zeile ausgeführt. **Achtung:** Wenn eine `target`-Zeile auf mehrere `source`-Zeilen passt, wird sie mehrmals aktualisiert (SQL-Standard würde Fehler erwarten).
3.  **Für jede Zeile aus `source`, die *keinen* Match in `target` hat:** Wenn `WHEN NOT MATCHED THEN INSERT` vorhanden ist, wird die `INSERT`-Operation in `target` ausgeführt.

**Hinweise:**
*   Mindestens eine `WHEN`-Klausel (`MATCHED` oder `NOT MATCHED`) muss vorhanden sein.
*   `<source>` kann eine Tabelle, View oder eine Subquery (`(<select-stmt>)`) sein.
*   `ROW_COUNT` gibt in FB <= 2.5 nicht die korrekte Anzahl betroffener Zeilen zurück.

### EXECUTE PROCEDURE Statement
Führt eine *executable* Stored Procedure aus.

```sql
EXECUTE PROCEDURE procname
  [<inparam_values>]
  [RETURNING_VALUES <output_variables>] -- Nur PSQL
```

**Parameterübergabe (`<inparam_values>`):**
*   Ohne Klammern: `value1, value2, ...`
*   Mit Klammern: `(value1, value2, ...)` (empfohlen)
*   Werte können Literale, Variablen (`:var`), Ausdrücke sein.

**Hinweise:**
*   Ruft Prozeduren auf, die typischerweise Aktionen ausführen und optional eine *einzelne* Zeile via `RETURNS` zurückgeben.
*   In PSQL können die Rückgabewerte mit `RETURNING_VALUES :var1, :var2, ...` aufgefangen werden.
*   In DSQL werden Rückgabewerte über die API in einem Buffer empfangen.
*   Aufruf einer *selectable* Prozedur hiermit liefert nur die *erste* Zeile zurück.

### EXECUTE BLOCK Statement
Führt einen anonymen Block von PSQL-Code aus. Nur in DSQL verfügbar.

```sql
EXECUTE BLOCK [(<inparams>)]
  [RETURNS (<outparams>)]
AS
  [<declarations>]
BEGIN
  [<PSQL statements>]
END
```

**Parameter:**
*   `<inparams>`: Eingabeparameter, *müssen* über Platzhalter (`?`) von der Client-Anwendung versorgt werden. `name = ? [, name = ? ...]`
*   `<outparams>`: Ausgabeparameter, definieren die Struktur der zurückgegebenen Ergebnismenge. `name <type> [, name <type> ...]`
*   `AS ... BEGIN ... END`: Wie bei Stored Procedures/Triggers.

**Hinweise:**
*   Erlaubt "on-the-fly" PSQL.
*   Wenn `RETURNS` definiert ist, muss im Body `SUSPEND` verwendet werden, um Zeilen zurückzugeben.
*   Die Ergebnismenge wird wie bei einem `SELECT` oder einer selectable Procedure verarbeitet.
*   Kein `RETURNING_VALUES` oder `INTO` wie bei `EXECUTE PROCEDURE`.
*   Parameterübergabe (`?`) erfordert API-Unterstützung im Client (z.B. nicht direkt in `isql`).

Okay, hier folgt der Abschnitt über Procedural SQL (PSQL) Statements.

```markdown
## Procedural SQL (PSQL) Statements

PSQL ist eine prozedurale Erweiterung von SQL für Stored Procedures, Triggers und `EXECUTE BLOCK`-Anweisungen.

### Elements of PSQL

*   **DML Statements with Parameters:** `SELECT`, `INSERT`, `UPDATE`, `DELETE` etc. können in PSQL verwendet werden. Parameter müssen als Input-/Output-Parameter des Moduls oder als lokale Variablen deklariert sein und werden im Statement mit einem Doppelpunkt `:` referenziert (z.B. `:my_variable`). Der Doppelpunkt ist bei Zuweisungen und Bedingungen optional.
*   **Transactions:** PSQL-Code läuft im Kontext der aufrufenden Transaktion. Eigene `COMMIT`/`ROLLBACK` sind nicht erlaubt, aber `IN AUTONOMOUS TRANSACTION DO` ermöglicht das Ausführen von Code in einer separaten, sofort committeten/gerollbackten Transaktion.
*   **Module Structure:**
    *   **Header:** Definiert Name, Input-/Output-Parameter, lokale Variablen (`DECLARE VARIABLE`), benannte Cursors (`DECLARE CURSOR`).
    *   **Body:** Ein äußerer `BEGIN...END`-Block, der PSQL- und DML-Anweisungen enthält. Kann weitere geschachtelte `BEGIN...END`-Blöcke enthalten. Anweisungen (außer `BEGIN`/`END`) enden mit Semikolon (`;`).
*   **Statement Terminators:** In Umgebungen wie `isql` muss der Standard-Terminator (`;`) temporär mittels `SET TERM` geändert werden, um PSQL-Module definieren zu können, da PSQL intern das Semikolon verwendet.

### Writing the Body Code

#### Assignment Statements
Weist einer Variablen oder einem `NEW.`-Kontextvariable einen Wert zu.

*   **Used for:** Assigning a value to a variable.
*   **Available in:** PSQL
*   **Syntax:**
    ```sql
    varname = <value_expr>;
    ```
*   **Description:** `varname` ist eine lokale Variable, ein Input- oder Output-Parameter. `<value_expr>` kann ein Literal, eine Kontextvariable, eine andere Variable oder ein beliebiger SQL-Ausdruck sein, dessen Ergebnis typkompatibel zu `varname` ist.

#### DECLARE CURSOR
Deklariert einen benannten Cursor für eine `SELECT`-Anweisung.

*   **Used for:** Declaring a named cursor.
*   **Available in:** PSQL
*   **Syntax:**
    ```sql
    DECLARE [VARIABLE] cursorname CURSOR FOR (<select_statement>) [FOR UPDATE];
    ```
*   **Description:** Bindet einen Namen (`cursorname`) an eine `SELECT`-Abfrage. Der Cursor muss später explizit mit `OPEN`, `FETCH` (in einer Schleife) und `CLOSE` verwendet werden. Ermöglicht `positioned updates/deletes` mittels `WHERE CURRENT OF cursorname`, wenn der Cursor über einer Zeile positioniert ist. `FOR UPDATE` in der Cursor-Deklaration ist optional und hat in FB <= 2.5 keine funktionale Auswirkung auf die Updatebarkeit. Der `SELECT`-Teil kann Parameter enthalten (als `:variable`).

#### DECLARE VARIABLE
Deklariert eine lokale Variable.

*   **Used for:** Declaring a local variable.
*   **Available in:** PSQL
*   **Syntax:**
    ```sql
    DECLARE [VARIABLE] varname
      {<datatype> | domain | TYPE OF {domain | COLUMN rel.col}}
      [NOT NULL] [CHARACTER SET charset] [COLLATE collation]
      [{DEFAULT | =} <initvalue>];
    ```
*   **Description:** Deklariert eine Variable mit einem Namen und Typ. Kann optional `NOT NULL`, einen Zeichensatz/Collation und einen Initialwert (`DEFAULT` oder `=`) haben. Ohne Initialwert ist die Variable initial `NULL`. Der Typ kann ein SQL-Basistyp (außer Array), eine Domain, `TYPE OF DOMAIN` oder `TYPE OF COLUMN` sein. `VARIABLE` ist optional.

#### BEGIN … END
Definiert einen Block von Anweisungen.

*   **Used for:** Delineating a block of statements.
*   **Available in:** PSQL
*   **Syntax:**
    ```sql
    [label:]
    BEGIN
      [<declarations>] -- Nur im äußersten Block von Prozeduren/Triggern
      [<PSQL statements>]
      [<WHEN ... DO ...>] -- Fehlerbehandlung, nur direkt vor END
    END
    ```
*   **Description:** Gruppiert Anweisungen. Kann leer sein. Kann geschachtelt werden. Der äußerste `BEGIN...END`-Block bildet den Body einer Prozedur oder eines Triggers. `WHEN...DO`-Klauseln zur Fehlerbehandlung können nur unmittelbar vor dem `END` stehen.

#### IF … THEN … ELSE Statement
Bedingte Ausführung von Codeblöcken.

*   **Used for:** Conditional jumps.
*   **Available in:** PSQL
*   **Syntax:**
    ```sql
    IF (<condition>)
      THEN <statement_or_block>
      [ELSE <statement_or_block>]
    ```
*   **Description:** Wenn `<condition>` TRUE ergibt, wird der `THEN`-Teil ausgeführt. Wenn FALSE oder UNKNOWN(NULL) und `ELSE` vorhanden ist, wird der `ELSE`-Teil ausgeführt. `<statement_or_block>` kann eine einzelne Anweisung (mit Semikolon) oder ein `BEGIN...END`-Block sein.

#### WHILE … DO Statement
Schleifenkonstrukt mit Vorabprüfung der Bedingung.

*   **Used for:** Looping constructs.
*   **Available in:** PSQL
*   **Syntax:**
    ```sql
    WHILE (<condition>) DO
      <statement_or_block>
    ```
*   **Description:** Der `<statement_or_block>` wird wiederholt ausgeführt, solange `<condition>` TRUE ergibt. Die Bedingung wird *vor* jeder Iteration geprüft.

#### LEAVE Statement
Beendet die aktuelle Schleife (WHILE, FOR).

*   **Used for:** Terminating a loop.
*   **Available in:** PSQL
*   **Syntax:**
    ```sql
    LEAVE [label];
    ```
*   **Description:** Springt aus der innersten `WHILE`-, `FOR SELECT`- oder `FOR EXECUTE STATEMENT`-Schleife heraus. Mit einem optionalen `label` kann auch eine benannte äußere Schleife verlassen werden. Die Ausführung wird nach dem `END`-Block der Schleife fortgesetzt.

#### EXIT Statement
Beendet die Ausführung des gesamten PSQL-Moduls (Prozedur, Trigger, Block).

*   **Used for:** Terminating module execution.
*   **Available in:** PSQL
*   **Syntax:**
    ```sql
    EXIT;
    ```
*   **Description:** Springt direkt zum finalen `END` des Moduls. Bei selectable Procedures wird SQLCODE 100 zurückgegeben. Bei executable Procedures werden die aktuellen Werte der Output-Parameter zurückgegeben.

#### SUSPEND Statement
Gibt eine Zeile aus einer selectable Procedure oder einem `EXECUTE BLOCK` zurück und pausiert die Ausführung.

*   **Used for:** Passing output to the buffer and suspending execution.
*   **Available in:** PSQL (nur in selectable Procedures und `EXECUTE BLOCK` mit `RETURNS`)
*   **Syntax:**
    ```sql
    SUSPEND;
    ```
*   **Description:** Die aktuellen Werte der `RETURNS`-Parameter werden in den Ausgabe-Buffer gestellt. Die Ausführung des Moduls wird angehalten, bis der aufrufende Prozess (z.B. `SELECT`) die nächste Zeile anfordert (`FETCH`). In executable Procedures verhält sich `SUSPEND` wie `EXIT`.

#### EXECUTE STATEMENT Statement
Führt eine dynamisch erstellte SQL-Anweisung aus.

*   **Used for:** Executing dynamically created SQL statements.
*   **Available in:** PSQL
*   **Syntax:**
    ```sql
    EXECUTE STATEMENT <sql_string_expr>
      [( <param_assignments> )]
      [WITH {AUTONOMOUS | COMMON} TRANSACTION]
      [WITH CALLER PRIVILEGES]
      [AS USER user] [PASSWORD password] [ROLE role]
      [ON EXTERNAL [DATA SOURCE] <connect_string>]
      [INTO <variables>];
    ```
*   **Description:** Führt den SQL-String in `<sql_string_expr>` aus.
    *   **Parameter:** Können im SQL-String als `:name` (named) oder `?` (positional) vorkommen. Werte werden in der `(<param_assignments>)`-Klausel zugewiesen (`param := value` für named, `value1, value2...` für positional). Named und positional nicht mischen.
    *   `INTO <variables>`: Fängt das Ergebnis eines Singleton Select auf (nur eine Zeile erlaubt). `:` vor Variablennamen ist optional.
    *   `WITH AUTONOMOUS TRANSACTION`: Führt das Statement in einer separaten Transaktion aus, die sofort committet/gerollbackt wird.
    *   `WITH COMMON TRANSACTION`: Verwendet die aktuelle Transaktion, falls möglich, sonst wie Autonomous.
    *   `WITH CALLER PRIVILEGES`: Führt das Statement mit den kombinierten Rechten des aktuellen Benutzers UND des aufrufenden Moduls aus.
    *   `AS USER/PASSWORD/ROLE`: Führt das Statement unter anderer Benutzeridentität aus (öffnet neue Verbindung, wenn `ON EXTERNAL` fehlt).
    *   `ON EXTERNAL [DATA SOURCE]`: Führt das Statement auf einer anderen Datenbank (lokal oder remote) aus. Erfordert Connection String. Verwaltet Verbindungs-Pooling. Fehler werden speziell behandelt (eds_connection/eds_statement).

#### FOR SELECT Statement
Schleifenkonstrukt zum Iterieren über das Ergebnis einer `SELECT`-Anweisung.

*   **Used for:** Looping row-by-row through a selected result set.
*   **Available in:** PSQL
*   **Syntax:**
    ```sql
    FOR <select_statement> [INTO <variables>] [AS CURSOR cursorname]
      DO <statement_or_block>
    ```
*   **Description:** Führt die `<select_statement>` aus und iteriert über jede zurückgegebene Zeile.
    *   `INTO <variables>`: Obligatorisch. Die Werte jeder Zeile werden in die angegebenen PSQL-Variablen geladen. Anzahl, Reihenfolge, Typ müssen passen. `:` optional.
    *   `AS CURSOR cursorname`: Optional. Macht einen Cursor verfügbar, um innerhalb der Schleife `WHERE CURRENT OF cursorname` für `UPDATE` oder `DELETE` zu verwenden (positioned update/delete). `cursorname` muss eindeutig sein.

#### FOR EXECUTE STATEMENT Statement
Schleifenkonstrukt zum Iterieren über das Ergebnis einer dynamisch ausgeführten `SELECT`-Anweisung.

*   **Used for:** Executing dynamically created SQL statements that return a row set.
*   **Available in:** PSQL
*   **Syntax:**
    ```sql
    FOR <execute_statement> INTO <variables>
      DO <statement_or_block>
    ```
*   **Description:** Kombiniert `EXECUTE STATEMENT` mit einer Schleife. `<execute_statement>` muss ein `SELECT` sein. Iteriert über jede Zeile des dynamischen Ergebnisses und lädt die Werte in die `INTO`-Variablen.

#### OPEN Statement
Öffnet einen deklarierten Cursor.

*   **Used for:** Opening a declared cursor.
*   **Available in:** PSQL
*   **Syntax:**
    ```sql
    OPEN cursorname;
    ```
*   **Description:** Führt die dem `cursorname` zugeordnete `SELECT`-Anweisung aus und positioniert den Cursor vor der ersten Zeile der Ergebnismenge. Eventuelle Parameter im `SELECT` erhalten die aktuellen Werte der zugehörigen PSQL-Variablen.

#### FETCH Statement
Holt die nächste Zeile aus einem offenen, deklarierten Cursor.

*   **Used for:** Fetching successive records from a data set retrieved by a cursor.
*   **Available in:** PSQL
*   **Syntax:**
    ```sql
    FETCH cursorname INTO [:]varname [, [:]varname ...];
    ```
*   **Description:** Bewegt den `cursorname` zur nächsten Zeile und lädt die Spaltenwerte in die angegebenen PSQL-Variablen.
*   **Status:** Die Kontextvariable `ROW_COUNT` ist 1, wenn eine Zeile erfolgreich geholt wurde, und 0, wenn das Ende der Ergebnismenge erreicht ist.

#### CLOSE Statement
Schließt einen offenen, deklarierten Cursor.

*   **Used for:** Closing a declared cursor.
*   **Available in:** PSQL
*   **Syntax:**
    ```sql
    CLOSE cursorname;
    ```
*   **Description:** Gibt die mit dem Cursor verbundenen Ressourcen frei.

#### IN AUTONOMOUS TRANSACTION Statement
Führt eine Anweisung oder einen Block in einer separaten Transaktion aus.

*   **Used for:** Executing a statement or a block of statements in an autonomous transaction.
*   **Available in:** PSQL
*   **Syntax:**
    ```sql
    IN AUTONOMOUS TRANSACTION DO
      <statement_or_block>
    ```
*   **Description:** Der eingeschlossene Code wird in einer neuen Transaktion ausgeführt, die die gleichen Eigenschaften wie die aufrufende Transaktion hat. Diese neue Transaktion wird am Ende des Blocks committet (bei Erfolg) oder zurückgerollt (bei Fehler), unabhängig vom Status der aufrufenden Transaktion. Nützlich für Logging oder Aktionen, die auch bei einem Rollback des Hauptteils erhalten bleiben sollen.

#### POST_EVENT Statement
Sendet eine Nachricht (Event) an lauschende Clients.

*   **Used for:** Notifying listening clients about database events in a module.
*   **Available in:** PSQL
*   **Syntax:**
    ```sql
    POST_EVENT <event_name_expr>;
    ```
*   **Description:** `<event_name_expr>` ist ein String-Ausdruck (Literal, Variable etc., max. 127 Bytes). Das Event wird registriert und an alle Clients gesendet, die sich für dieses Event registriert haben, *nachdem* die Transaktion erfolgreich committet wurde.

### Trapping and Handling Errors

#### EXCEPTION Statement
Wirft eine benutzerdefinierte oder eine System-Exception (letzteres nur innerhalb eines `WHEN`-Handlers).

*   **Used for:** Throwing a user-defined exception or re-throwing an exception.
*   **Available in:** PSQL
*   **Syntax:**
    ```sql
    EXCEPTION [exception_name [custom_message]];
    ```
*   **Description:**
    *   `EXCEPTION exception_name ['message']`: Wirft die benannte, benutzerdefinierte Exception. Die optionale 'message' (String-Ausdruck, max 1021 Bytes) überschreibt die Standardmeldung der Exception.
    *   `EXCEPTION;`: (Nur innerhalb eines `WHEN ... DO`-Blocks) Wirft die gerade abgefangene Exception erneut.

#### WHEN … DO Statement
Fängt Exceptions ab und führt einen Handler-Code aus.

*   **Used for:** Catching an exception and handling the error.
*   **Available in:** PSQL
*   **Syntax:**
    ```sql
    WHEN <error_condition> [, <error_condition> ...] DO
      <statement_or_block>
    ```
*   **Fehlerbedingungen (`<error_condition>`):**
    *   `EXCEPTION exception_name`: Fängt eine spezifische benutzerdefinierte Exception.
    *   `SQLCODE number`: Fängt einen spezifischen SQLCODE-Fehler (veraltet).
    *   `GDSCODE errcode`: Fängt einen spezifischen GDSCODE-Fehler (symbolischer Name).
    *   `ANY`: Fängt jede Exception, die nicht bereits in einem spezifischeren `WHEN` im selben Block abgefangen wurde.
*   **Description:** Muss am Ende eines `BEGIN...END`-Blocks stehen (vor `END`, nach allen anderen Statements). Wenn eine der gelisteten Fehlerbedingungen im Block auftritt, wird der normale Ablauf abgebrochen und der `DO`-Teil ausgeführt. Danach wird die Ausführung *nach* dem `END` des Blocks fortgesetzt (es sei denn, der Handler wirft selbst eine Exception oder verwendet `EXIT`/`LEAVE`).
*   **Kontext in `DO`:** Die Variablen `SQLCODE`, `GDSCODE`, `SQLSTATE` enthalten Informationen über den aufgetretenen Fehler.

Okay, hier ist der Abschnitt über die eingebauten Funktionen (Built-in Functions).

```markdown
## Built-in Functions

Firebird bietet eine Vielzahl eingebauter Funktionen für verschiedene Aufgaben. Viele davon waren in früheren Versionen externe UDFs und wurden nach und nach internalisiert. Wenn eine gleichnamige UDF in der Datenbank deklariert ist, hat diese Vorrang vor der internen Funktion.

### Context Functions

#### RDB$GET_CONTEXT()
Ruft den Wert einer Kontextvariable aus einem Namespace ab.

*   **Available in:** DSQL, PSQL (*als deklarierte UDF auch in ESQL*)
*   **Syntax:**
    ```sql
    RDB$GET_CONTEXT ('<namespace>', <varname>)
    ```
*   **Parameters:**
    | Parameter | Description                                                             |
    | :-------- | :---------------------------------------------------------------------- |
    | namespace | `'SYSTEM'`, `'USER_SESSION'`, `'USER_TRANSACTION'` (case-sensitive)     |
    | varname   | Variablenname (String-Literal, case-sensitive, max. 80 Zeichen)         |
*   **Result type:** `VARCHAR(255)`
*   **Description:** Liest den Wert der Variable `varname` aus dem angegebenen Namespace.
    *   `SYSTEM`: Read-only. Enthält vordefinierte Variablen wie `DB_NAME`, `NETWORK_PROTOCOL`, `CLIENT_ADDRESS`, `CLIENT_PID` (ab 2.5.3), `CLIENT_PROCESS` (ab 2.5.3), `CURRENT_USER`, `CURRENT_ROLE`, `ISOLATION_LEVEL`, `LOCK_TIMEOUT` (ab 2.5.3), `READ_ONLY` (ab 2.5.3), `SESSION_ID`, `TRANSACTION_ID`, `ENGINE_VERSION` (ab 2.1).
    *   `USER_SESSION`, `USER_TRANSACTION`: Read/Write. Initial leer. Variablen können mit `RDB$SET_CONTEXT` gesetzt werden. `USER_SESSION` ist verbindungsgebunden, `USER_TRANSACTION` transaktionsgebunden (überlebt `ROLLBACK TO SAVEPOINT`).
*   **Returns:** Wert der Variable als String oder `NULL`, wenn Variable in `USER_*` nicht existiert. Fehler bei Zugriff auf nicht-existente SYSTEM-Variable oder ungültigen Namespace.

#### RDB$SET_CONTEXT()
Setzt oder löscht eine Kontextvariable in einem beschreibbaren Namespace.

*   **Available in:** DSQL, PSQL (*als deklarierte UDF auch in ESQL*)
*   **Syntax:**
    ```sql
    RDB$SET_CONTEXT ('<namespace>', <varname>, <value> | NULL)
    ```
*   **Parameters:**
    | Parameter | Description                                                             |
    | :-------- | :---------------------------------------------------------------------- |
    | namespace | `'USER_SESSION'`, `'USER_TRANSACTION'` (case-sensitive)                 |
    | varname   | Variablenname (String-Literal, case-sensitive, max. 80 Zeichen)         |
    | value     | Beliebiger Wert, der zu `VARCHAR(255)` gecastet werden kann, oder `NULL` |
*   **Result type:** `INTEGER`
*   **Description:** Setzt den Wert der Variable `varname` im Namespace `namespace`. Wenn `value` `NULL` ist, wird die Variable gelöscht.
*   **Returns:** 1, wenn die Variable bereits existierte, 0, wenn sie neu erstellt wurde. Fehler bei ungültigem Namespace. Max. 1000 Variablen pro Kontext.

### Mathematical Functions

#### ABS(number)
*   **Available in:** DSQL, PSQL
*   **Result type:** Numerisch (wie Argument)
*   **Description:** Gibt den Absolutwert (Betrag) des Arguments zurück.

#### ACOS(number)
*   **Available in:** DSQL, PSQL
*   **Parameter:** `number` im Bereich [-1, 1].
*   **Result type:** `DOUBLE PRECISION`
*   **Description:** Gibt den Arkuskosinus (in Radiant) zurück. Ergebnis im Bereich [0, pi]. Gibt `NaN` zurück, wenn Argument außerhalb von [-1, 1].

#### ASIN(number)
*   **Available in:** DSQL, PSQL
*   **Parameter:** `number` im Bereich [-1, 1].
*   **Result type:** `DOUBLE PRECISION`
*   **Description:** Gibt den Arkussinus (in Radiant) zurück. Ergebnis im Bereich [-pi/2, pi/2]. Gibt `NaN` zurück, wenn Argument außerhalb von [-1, 1].

#### ATAN(number)
*   **Available in:** DSQL, PSQL
*   **Result type:** `DOUBLE PRECISION`
*   **Description:** Gibt den Arkustangens (in Radiant) zurück. Ergebnis im Bereich (-pi/2, pi/2).

#### ATAN2(y, x)
*   **Available in:** DSQL, PSQL
*   **Result type:** `DOUBLE PRECISION`
*   **Description:** Gibt den Arkustangens von `y/x` (in Radiant) zurück, berücksichtigt die Vorzeichen von `y` und `x`, um den korrekten Quadranten zu bestimmen. Ergebnis im Bereich [-pi, pi]. Entspricht dem Winkel zwischen der positiven X-Achse und dem Punkt (x, y). Fehler, wenn x und y beide 0 sind (ab FB3).

#### CEIL() / CEILING()
*   **Available in:** DSQL, PSQL
*   **Syntax:** `CEIL | CEILING (number)`
*   **Result type:** `BIGINT` (für exakte Typen), `DOUBLE PRECISION` (für Fließkomma)
*   **Description:** Gibt die kleinste ganze Zahl zurück, die größer oder gleich dem Argument ist (Aufrundung zur nächsten ganzen Zahl).

#### COS(angle)
*   **Available in:** DSQL, PSQL
*   **Parameter:** `angle` in Radiant.
*   **Result type:** `DOUBLE PRECISION`
*   **Description:** Gibt den Kosinus des Winkels zurück. Ergebnis im Bereich [-1, 1].

#### COSH(number)
*   **Available in:** DSQL, PSQL
*   **Result type:** `DOUBLE PRECISION`
*   **Description:** Gibt den hyperbolischen Kosinus zurück. Ergebnis im Bereich [1, INF].

#### COT(angle)
*   **Available in:** DSQL, PSQL
*   **Parameter:** `angle` in Radiant.
*   **Result type:** `DOUBLE PRECISION`
*   **Description:** Gibt den Kotangens des Winkels zurück.

#### EXP(number)
*   **Available in:** DSQL, PSQL
*   **Result type:** `DOUBLE PRECISION`
*   **Description:** Gibt die Exponentialfunktion e<sup>number</sup> zurück (e ≈ 2.718...).

#### FLOOR(number)
*   **Available in:** DSQL, PSQL
*   **Result type:** `BIGINT` (für exakte Typen), `DOUBLE PRECISION` (für Fließkomma)
*   **Description:** Gibt die größte ganze Zahl zurück, die kleiner oder gleich dem Argument ist (Abrundung zur nächsten ganzen Zahl).

#### LN(number)
*   **Available in:** DSQL, PSQL
*   **Parameter:** `number` muss > 0 sein.
*   **Result type:** `DOUBLE PRECISION`
*   **Description:** Gibt den natürlichen Logarithmus (Basis e) zurück. Fehler, wenn Argument <= 0.

#### LOG(x, y)
*   **Available in:** DSQL, PSQL
*   **Parameter:** `x` (Basis) und `y` müssen > 0 sein. `x` darf nicht 1 sein.
*   **Result type:** `DOUBLE PRECISION`
*   **Description:** Gibt den Logarithmus von `y` zur Basis `x` zurück. Fehler bei ungültigen Argumenten.

#### LOG10(number)
*   **Available in:** DSQL, PSQL
*   **Parameter:** `number` muss > 0 sein.
*   **Result type:** `DOUBLE PRECISION`
*   **Description:** Gibt den dekadischen Logarithmus (Basis 10) zurück. Fehler, wenn Argument <= 0.

#### MOD(a, b)
*   **Available in:** DSQL, PSQL
*   **Parameter:** `a`, `b` numerisch. `b` darf nicht 0 sein.
*   **Result type:** Integer (`SMALLINT`, `INTEGER`, `BIGINT` abhängig von `a`)
*   **Description:** Gibt den Rest der Integer-Division `a / b` zurück. Nicht-Integer-Argumente werden vor der Division gerundet.

#### PI()
*   **Available in:** DSQL, PSQL
*   **Result type:** `DOUBLE PRECISION`
*   **Description:** Gibt eine Annäherung der Kreiszahl Pi zurück.

#### POWER(x, y)
*   **Available in:** DSQL, PSQL
*   **Result type:** `DOUBLE PRECISION`
*   **Description:** Gibt `x` hoch `y` (x<sup>y</sup>) zurück.

#### RAND()
*   **Available in:** DSQL, PSQL
*   **Result type:** `DOUBLE PRECISION`
*   **Description:** Gibt eine Pseudo-Zufallszahl zwischen 0 und 1 zurück.

#### ROUND(number [, scale])
*   **Available in:** DSQL, PSQL
*   **Parameter:** `scale` (optional) ist ein Integer, der die Rundungsstelle relativ zum Dezimalpunkt angibt (0=ganze Zahl, 1=1 Nachkommastelle, -1=Zehnerstelle etc.).
*   **Result type:** `INTEGER`/`BIGINT` (skaliert) oder `DOUBLE PRECISION`.
*   **Description:** Rundet `number`. Ohne `scale` wird auf die nächste ganze Zahl gerundet. Mit `scale` wird auf die entsprechende Zehnerpotenz gerundet. Halbe Werte (x.5) werden immer von Null weg gerundet (kaufmännisches Runden).

#### SIGN(number)
*   **Available in:** DSQL, PSQL
*   **Result type:** `SMALLINT`
*   **Description:** Gibt das Vorzeichen des Arguments zurück: -1 für negativ, 0 für Null, 1 für positiv.

#### SIN(angle)
*   **Available in:** DSQL, PSQL
*   **Parameter:** `angle` in Radiant.
*   **Result type:** `DOUBLE PRECISION`
*   **Description:** Gibt den Sinus des Winkels zurück. Ergebnis im Bereich [-1, 1].

#### SINH(number)
*   **Available in:** DSQL, PSQL
*   **Result type:** `DOUBLE PRECISION`
*   **Description:** Gibt den hyperbolischen Sinus zurück.

#### SQRT(number)
*   **Available in:** DSQL, PSQL
*   **Parameter:** `number` darf nicht negativ sein.
*   **Result type:** `DOUBLE PRECISION`
*   **Description:** Gibt die Quadratwurzel zurück. Fehler, wenn Argument negativ.

#### TAN(angle)
*   **Available in:** DSQL, PSQL
*   **Parameter:** `angle` in Radiant.
*   **Result type:** `DOUBLE PRECISION`
*   **Description:** Gibt den Tangens des Winkels zurück.

#### TANH(number)
*   **Available in:** DSQL, PSQL
*   **Result type:** `DOUBLE PRECISION`
*   **Description:** Gibt den hyperbolischen Tangens zurück. Ergebnis im Bereich (-1, 1).

#### TRUNC(number [, scale])
*   **Available in:** DSQL, PSQL
*   **Parameter:** `scale` (optional) wie bei `ROUND`.
*   **Result type:** `INTEGER`/`BIGINT` (skaliert) oder `DOUBLE PRECISION`.
*   **Description:** Schneidet `number` ab. Ohne `scale` wird der Nachkommateil entfernt. Mit `scale` wird auf die entsprechende Zehnerpotenz abgeschnitten (immer Richtung Null).

### String Functions

#### ASCII_CHAR(code)
*   **Available in:** DSQL, PSQL
*   **Parameter:** `code` ist Integer im Bereich 0-255.
*   **Result type:** `CHAR(1) CHARACTER SET NONE`
*   **Description:** Gibt das ASCII-Zeichen zum gegebenen Code zurück. Gibt korrekt das NULL-Byte (`\0`) für code=0 zurück.

#### ASCII_VAL(char)
*   **Available in:** DSQL, PSQL
*   **Parameter:** `char` ist ein String oder Text-BLOB.
*   **Result type:** `SMALLINT`
*   **Description:** Gibt den ASCII-Code des *ersten* Zeichens des Strings zurück. Gibt 0 für leeren String zurück. Gibt NULL zurück, wenn Argument NULL ist. Fehler, wenn das erste Zeichen multi-byte ist.

#### BIT_LENGTH(string)
*   **Available in:** DSQL, PSQL
*   **Result type:** `INTEGER`
*   **Description:** Gibt die Länge des Strings in Bits zurück. Berücksichtigt die tatsächliche Bitlänge von Multi-Byte-Zeichen. Bei `CHAR`-Typen wird die deklarierte Länge verwendet (inkl. Füllzeichen).

#### CHAR_LENGTH() / CHARACTER_LENGTH()
*   **Available in:** DSQL, PSQL
*   **Syntax:** `CHAR_LENGTH | CHARACTER_LENGTH (string)`
*   **Result type:** `INTEGER`
*   **Description:** Gibt die Länge des Strings in Zeichen zurück. Bei `CHAR`-Typen wird die deklarierte Länge zurückgegeben. Unterstützt BLOBs.

#### HASH(string)
*   **Available in:** DSQL, PSQL
*   **Result type:** `BIGINT`
*   **Description:** Berechnet einen 64-Bit-Hashwert für den Eingabestring. Unterstützt BLOBs.

#### LEFT(string, length)
*   **Available in:** DSQL, PSQL
*   **Result type:** `VARCHAR` oder `BLOB`
*   **Description:** Gibt die ersten `length` Zeichen von `string` zurück. Unterstützt BLOBs. Wenn `length` keine ganze Zahl ist, wird kaufmännisch gerundet. Wenn `length` > Stringlänge, wird der ganze String zurückgegeben.

#### LOWER(string)
*   **Available in:** DSQL, ESQL, PSQL
*   **Result type:** `(VAR)CHAR` oder `BLOB` (wie Eingabe)
*   **Description:** Konvertiert den String in Kleinbuchstaben gemäß den Regeln des Zeichensatzes/Collation. Unterstützt BLOBs.

#### LPAD(str, endlen [, padstr])
*   **Available in:** DSQL, PSQL
*   **Result type:** `VARCHAR(endlen)` oder `BLOB`
*   **Description:** Füllt `str` von links mit `padstr` (Default: Leerzeichen) auf, bis die Gesamtlänge `endlen` erreicht ist. Wenn `str` länger als `endlen` ist, wird `str` auf `endlen` gekürzt. Unterstützt BLOBs.

#### OCTET_LENGTH(string)
*   **Available in:** DSQL, PSQL
*   **Result type:** `INTEGER`
*   **Description:** Gibt die Länge des Strings in Bytes (Oktetts) zurück. Bei `CHAR`-Typen wird die deklarierte Länge (in Bytes) zurückgegeben. Unterstützt BLOBs.

#### OVERLAY(string PLACING replacement FROM pos [FOR length])
*   **Available in:** DSQL, PSQL
*   **Result type:** `VARCHAR` oder `BLOB`
*   **Description:** Ersetzt einen Teil von `string`. Ab Position `pos` (1-basiert) werden `length` Zeichen durch `replacement` ersetzt. Wenn `FOR length` fehlt, entspricht die Länge der von `replacement`. `FOR 0` fügt `replacement` an Position `pos` ein. Unterstützt BLOBs.

#### POSITION(substr IN string) / POSITION(substr, string [, startpos])
*   **Available in:** DSQL, ESQL, PSQL
*   **Result type:** `INTEGER`
*   **Description:** Gibt die 1-basierte Startposition des ersten Vorkommens von `substr` in `string` zurück. Mit `startpos` beginnt die Suche erst ab dieser Position. Gibt 0 zurück, wenn `substr` nicht gefunden wird. Leerer `substr` gibt 1 zurück (oder `startpos`, wenn angegeben und gültig). Unterstützt BLOBs.

#### REPLACE(str, find, repl)
*   **Available in:** DSQL, PSQL
*   **Result type:** `VARCHAR` oder `BLOB`
*   **Description:** Ersetzt *alle* Vorkommen von `find` in `str` durch `repl`. Wenn `repl` leer ist, werden Vorkommen von `find` gelöscht. Unterstützt BLOBs.

#### REVERSE(string)
*   **Available in:** DSQL, PSQL
*   **Result type:** `VARCHAR`
*   **Description:** Gibt den String rückwärts zurück.

#### RIGHT(string, length)
*   **Available in:** DSQL, PSQL
*   **Result type:** `VARCHAR` oder `BLOB`
*   **Description:** Gibt die letzten `length` Zeichen von `string` zurück. Unterstützt BLOBs (mit Bugfix in 2.1.4/2.5.1 für große BLOBs). Wenn `length` keine ganze Zahl ist, wird kaufmännisch gerundet. Wenn `length` > Stringlänge, wird der ganze String zurückgegeben.

#### RPAD(str, endlen [, padstr])
*   **Available in:** DSQL, PSQL
*   **Result type:** `VARCHAR(endlen)` oder `BLOB`
*   **Description:** Füllt `str` von rechts mit `padstr` (Default: Leerzeichen) auf, bis die Gesamtlänge `endlen` erreicht ist. Wenn `str` länger als `endlen` ist, wird `str` auf `endlen` gekürzt. Unterstützt BLOBs.

#### SUBSTRING(str FROM startpos [FOR length])
*   **Available in:** DSQL, ESQL, PSQL
*   **Result type:** `VARCHAR` oder `BLOB`
*   **Description:** Extrahiert einen Teilstring aus `str`. Beginnt bei `startpos` (1-basiert). Ohne `FOR length` wird der Rest des Strings zurückgegeben. Mit `FOR length` werden maximal `length` Zeichen zurückgegeben. Unterstützt BLOBs (Bugfix für BLOBs ohne `length` in 2.1.5/2.5.1).

#### TRIM([[<where>] [what]] FROM str)
*   **Available in:** DSQL, ESQL, PSQL
*   **Parameter:**
    *   `where`: `LEADING` (führende), `TRAILING` (nachfolgende) oder `BOTH` (beide, Default).
    *   `what`: Der zu entfernende String (Default: Leerzeichen `' '`).
*   **Result type:** `VARCHAR` oder `BLOB`
*   **Description:** Entfernt Vorkommen von `what` vom Anfang, Ende oder beiden Enden von `str`. Unterstützt BLOBs.

#### UPPER(string)
*   **Available in:** DSQL, ESQL, PSQL
*   **Result type:** `(VAR)CHAR` oder `BLOB` (wie Eingabe)
*   **Description:** Konvertiert den String in Großbuchstaben gemäß den Regeln des Zeichensatzes/Collation. Unterstützt BLOBs.

Okay, hier ist der letzte Teil der Zusammenfassung über die Built-in Functions und Context Variables.

```markdown
### Date and Time Functions

#### DATEADD(amount unit TO datetime) / DATEADD(unit, amount, datetime)
*   **Available in:** DSQL, PSQL (ab 2.5 auch `WEEK` unit)
*   **Syntax:** Zwei Formen sind möglich.
*   **Parameters:**
    *   `amount`: Integer (negativ für Subtraktion).
    *   `unit`: `YEAR`, `MONTH`, `WEEK` (ab 2.5), `DAY`, `HOUR`, `MINUTE`, `SECOND`, `MILLISECOND`.
    *   `datetime`: Ein `DATE`, `TIME` oder `TIMESTAMP`-Ausdruck.
*   **Result type:** Wie `datetime`.
*   **Description:** Addiert/subtrahiert das angegebene Intervall zum/vom Datum/Zeitwert. Bei `DATE`-Typen waren Units kleiner als `DAY` vor 2.5 nicht erlaubt. Bei `TIME`-Typen sind nur `HOUR` bis `MILLISECOND` erlaubt.

#### DATEDIFF(unit FROM moment1 TO moment2) / DATEDIFF(unit, moment1, moment2)
*   **Available in:** DSQL, PSQL (ab 2.5 auch `WEEK` unit)
*   **Syntax:** Zwei Formen sind möglich.
*   **Parameters:**
    *   `unit`: `YEAR`, `MONTH`, `WEEK` (ab 2.5), `DAY`, `HOUR`, `MINUTE`, `SECOND`, `MILLISECOND`.
    *   `moment1`, `moment2`: `DATE`, `TIME` oder `TIMESTAMP`-Ausdrücke. Typen müssen kompatibel sein (DATE/TIMESTAMP gemischt erlaubt, TIME nur mit TIME).
*   **Result type:** `BIGINT`
*   **Description:** Gibt die Anzahl der vollständigen `unit`-Intervalle zwischen `moment1` und `moment2` zurück. Berücksichtigt keine kleineren Einheiten (z.B. `DATEDIFF(YEAR, '01-JAN-2023', '31-DEC-2023')` ist 0). Ergebnis ist negativ, wenn `moment2` vor `moment1` liegt.

#### EXTRACT(part FROM datetime)
*   **Available in:** DSQL, ESQL, PSQL
*   **Parameters:**
    *   `part`: `YEAR`, `MONTH`, `DAY`, `WEEKDAY` (0=So..6=Sa), `YEARDAY` (0-365), `HOUR`, `MINUTE`, `SECOND`, `MILLISECOND` (ab 2.1). `WEEK` (ISO-8601, ab 2.1).
    *   `datetime`: Ein `DATE`, `TIME` oder `TIMESTAMP`-Ausdruck.
*   **Result type:** `SMALLINT` oder `NUMERIC` (siehe Tabelle 144).
*   **Description:** Extrahiert den angegebenen Teil aus einem Datum/Zeitwert. Fehler, wenn der Teil nicht im Typ enthalten ist (z.B. `YEAR` aus `TIME`). `MILLISECOND` gibt `NUMERIC(9,1)` zurück. `SECOND` gibt `NUMERIC(9,4)` zurück (inkl. Millisekunden als Bruchteil).

### Type Casting Functions

#### CAST(expression AS target_type)
*   **Available in:** DSQL, ESQL, PSQL
*   **Syntax:** Siehe Abschnitt "Conversion of Data Types".
*   **Description:** Konvertiert einen Ausdruck explizit in einen anderen Datentyp, eine Domain oder den Typ einer Spalte. Fehler, wenn Konvertierung nicht möglich oder ungültig. Shorthand-Syntax `datatype 'literal'` nur für Datum/Zeit-Literale.

### Bitwise Functions

#### BIN_AND(num1, num2 [, ...])
*   **Available in:** DSQL, PSQL
*   **Result type:** `SMALLINT`, `INTEGER` oder `BIGINT` (abhängig von Argumenttypen).
*   **Description:** Führt eine bitweise AND-Operation auf den Integer-Argumenten durch.

#### BIN_NOT(number)
*   **Available in:** DSQL, PSQL
*   **Result type:** `SMALLINT`, `INTEGER` oder `BIGINT` (wie Argument).
*   **Description:** Führt eine bitweise NOT-Operation (Einerkomplement) durch.

#### BIN_OR(num1, num2 [, ...])
*   **Available in:** DSQL, PSQL
*   **Result type:** `SMALLINT`, `INTEGER` oder `BIGINT`.
*   **Description:** Führt eine bitweise OR-Operation durch.

#### BIN_SHL(number, shift)
*   **Available in:** DSQL, PSQL
*   **Result type:** `BIGINT`
*   **Description:** Führt einen bitweisen Left-Shift durch (`number << shift`, `number * 2^shift`).

#### BIN_SHR(number, shift)
*   **Available in:** DSQL, PSQL
*   **Result type:** `BIGINT`
*   **Description:** Führt einen arithmetischen bitweisen Right-Shift durch (`number >> shift`, `number / 2^shift`). Das Vorzeichenbit bleibt erhalten.

#### BIN_XOR(num1, num2 [, ...])
*   **Available in:** DSQL, PSQL
*   **Result type:** `SMALLINT`, `INTEGER` oder `BIGINT`.
*   **Description:** Führt eine bitweise XOR-Operation durch.

### UUID Functions (Universally Unique Identifier)

#### CHAR_TO_UUID(ascii_uuid)
*   **Available in:** DSQL, PSQL (ab 2.5)
*   **Parameter:** `ascii_uuid` ist ein 36-Zeichen String im Format `XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX`.
*   **Result type:** `CHAR(16) CHARACTER SET OCTETS`
*   **Description:** Konvertiert die hexadezimale String-Repräsentation einer UUID in den 16-Byte-Binärwert.

#### GEN_UUID()
*   **Available in:** DSQL, PSQL (ab 2.5)
*   **Result type:** `CHAR(16) CHARACTER SET OCTETS`
*   **Description:** Generiert eine neue, eindeutige 16-Byte UUID.

#### UUID_TO_CHAR(uuid)
*   **Available in:** DSQL, PSQL (ab 2.5)
*   **Parameter:** `uuid` ist ein 16-Byte `CHAR(16) CHARACTER SET OCTETS` Wert.
*   **Result type:** `CHAR(36)`
*   **Description:** Konvertiert eine 16-Byte UUID in ihre 36-Zeichen String-Repräsentation.

### Functions for Sequences (Generators)

#### GEN_ID(generator_name, step)
*   **Available in:** DSQL, ESQL, PSQL
*   **Result type:** `BIGINT`
*   **Description:** Inkrementiert (oder dekrementiert, wenn `step` negativ) den Generator `generator_name` um den Wert `step` und gibt den *neuen* Wert zurück. Wenn `step` 0 ist, wird der *aktuelle* Wert zurückgegeben, ohne ihn zu ändern. Der SQL-Standard `NEXT VALUE FOR` ist für Inkrement +1 zu bevorzugen.

### Conditional Functions (siehe auch Conditional Expressions)

#### COALESCE(expr1, expr2 [, ...])
*   Siehe oben bei Conditional Expressions.

#### DECODE(testexpr, expr1, result1 [, expr2, result2 ... ] [, defaultresult])
*   Siehe oben bei Conditional Expressions.

#### IIF(condition, resultT, resultF)
*   Siehe oben bei Conditional Expressions.

#### MAXVALUE(expr1 [, ...])
*   Siehe oben bei Conditional Expressions.

#### MINVALUE(expr1 [, ...])
*   Siehe oben bei Conditional Expressions.

#### NULLIF(expr1, expr2)
*   Siehe oben bei Conditional Expressions.

### Aggregate Functions

Diese Funktionen operieren auf Gruppen von Zeilen (definiert durch `GROUP BY` oder die gesamte Tabelle, wenn `GROUP BY` fehlt).

#### AVG([ALL | DISTINCT] expr)
*   **Available in:** DSQL, ESQL, PSQL
*   **Result type:** Numerisch (wie `expr`, aber ggf. mit höherer Präzision/Skala).
*   **Description:** Berechnet den Durchschnittswert von `expr`. `NULL`-Werte werden ignoriert. `DISTINCT` berücksichtigt jeden Wert nur einmal.

#### COUNT([ALL | DISTINCT] expr | *)
*   **Available in:** DSQL, ESQL, PSQL
*   **Result type:** `INTEGER`
*   **Description:** Zählt Zeilen oder Werte.
    *   `COUNT(*)`: Zählt alle Zeilen in der Gruppe.
    *   `COUNT(expr)`: Zählt alle Zeilen, in denen `expr` nicht `NULL` ist.
    *   `COUNT(DISTINCT expr)`: Zählt die Anzahl eindeutiger Nicht-NULL-Werte von `expr`.

#### LIST([ALL | DISTINCT] expr [, separator])
*   **Available in:** DSQL, PSQL (ab 2.5: `separator` kann Ausdruck sein)
*   **Result type:** `BLOB SUB_TYPE TEXT` (oder Subtyp von `expr`, wenn dieser BLOB ist)
*   **Description:** Konkateniert alle Nicht-NULL-Werte von `expr` zu einem einzigen String, getrennt durch `separator` (Default: Komma `,`). `DISTINCT` entfernt Duplikate (außer bei BLOB `expr`). Die Reihenfolge der Elemente ist undefiniert, es sei denn, die Eingabe wird vorsortiert (z.B. über Derived Table).

#### MAX([ALL | DISTINCT] expr)
*   **Available in:** DSQL, ESQL, PSQL
*   **Result type:** Wie `expr`.
*   **Description:** Findet den Maximalwert von `expr` in der Gruppe. `NULL`-Werte werden ignoriert. `DISTINCT` ist bedeutungslos. Unterstützt Text-BLOBs.

#### MIN([ALL | DISTINCT] expr)
*   **Available in:** DSQL, ESQL, PSQL
*   **Result type:** Wie `expr`.
*   **Description:** Findet den Minimalwert von `expr` in der Gruppe. `NULL`-Werte werden ignoriert. `DISTINCT` ist bedeutungslos. Unterstützt Text-BLOBs.

#### SUM([ALL | DISTINCT] expr)
*   **Available in:** DSQL, ESQL, PSQL
*   **Result type:** `BIGINT` oder `DOUBLE PRECISION` oder `DECIMAL`/`NUMERIC(18, s)` (abhängig vom Typ von `expr`).
*   **Description:** Berechnet die Summe von `expr`. `NULL`-Werte werden ignoriert. `DISTINCT` berücksichtigt jeden Wert nur einmal.

## Context Variables

Vordefinierte Variablen, die Informationen über den aktuellen Ausführungskontext liefern. Werden ohne `:` verwendet.

*   **`CURRENT_CONNECTION`:** `INTEGER`. Eindeutige ID der aktuellen Verbindung.
*   **`CURRENT_DATE`:** `DATE`. Aktuelles Datum des Servers. Konstant innerhalb eines PSQL-Modulaufrufs.
*   **`CURRENT_ROLE`:** `VARCHAR(31)`. Aktuell aktive Rolle des Benutzers oder `'NONE'`.
*   **`CURRENT_TIME [ (<precision>) ]`:** `TIME`. Aktuelle Uhrzeit des Servers. `precision` (0-3) für Millisekunden (Default 0). Konstant innerhalb eines PSQL-Modulaufrufs.
*   **`CURRENT_TIMESTAMP [ (<precision>) ]`:** `TIMESTAMP`. Aktuelles Datum und Uhrzeit des Servers. `precision` (0-3) für Millisekunden (Default 3). Konstant innerhalb eines PSQL-Modulaufrufs.
*   **`CURRENT_TRANSACTION`:** `INTEGER`. Eindeutige ID der aktuellen Transaktion.
*   **`CURRENT_USER`:** `VARCHAR(31)`. Name des aktuell verbundenen Benutzers. Äquivalent zu `USER`.
*   **`DELETING`:** `BOOLEAN`. (Nur in Triggern) TRUE, wenn der Trigger durch `DELETE` ausgelöst wurde.
*   **`GDSCODE`:** `INTEGER`. (Nur in `WHEN ... DO`-Blöcken) Numerischer Firebird-Fehlercode.
*   **`INSERTING`:** `BOOLEAN`. (Nur in Triggern) TRUE, wenn der Trigger durch `INSERT` ausgelöst wurde.
*   **`NEW.colname`:** Datentyp der Spalte. (Nur in `INSERT`/`UPDATE`-Triggern) Repräsentiert den neuen Wert einer Spalte. Read/write in `BEFORE`, read-only in `AFTER`.
*   **`'NOW'`:** `CHAR(3)`. String-Literal, das bei `CAST` zu Datum/Zeit den *aktuellen* Wert ergibt (im Gegensatz zu `CURRENT_*`).
*   **`OLD.colname`:** Datentyp der Spalte. (Nur in `UPDATE`/`DELETE`-Triggern) Repräsentiert den alten Wert einer Spalte. Immer read-only.
*   **`ROW_COUNT`:** `INTEGER`. Anzahl der Zeilen, die von der letzten DML-Anweisung (`INSERT`, `UPDATE`, `DELETE`, Singleton `SELECT`, `FETCH`) betroffen waren. Verhalten variiert leicht je nach Anweisung. Nicht für `EXECUTE STATEMENT`/`PROCEDURE`.
*   **`SQLCODE`:** `INTEGER`. (Nur in `WHEN ... DO`-Blöcken) SQL-Standard-Fehlercode (veraltet, meist 0 oder negativ).
*   **`SQLSTATE`:** `CHAR(5)`. (Nur in `WHEN ... DO`-Blöcken, ab 2.5.1) SQL-2003-konformer Statuscode. `'00000'` bei Erfolg, andere Codes für Warnungen/Fehler.
*   **`'TODAY'`:** `CHAR(5)`. String-Literal, das bei `CAST` zu Datum/Zeit das *aktuelle* Datum ergibt. Konstant innerhalb eines PSQL-Modulaufrufs.
*   **`'TOMORROW'`:** `CHAR(8)`. String-Literal, das bei `CAST` zu Datum/Zeit das *morgige* Datum ergibt.
*   **`UPDATING`:** `BOOLEAN`. (Nur in Triggern) TRUE, wenn der Trigger durch `UPDATE` ausgelöst wurde.
*   **`USER`:** `VARCHAR(31)`. Name des aktuell verbundenen Benutzers. Äquivalent zu `CURRENT_USER`.
*   **`'YESTERDAY'`:** `CHAR(9)`. String-Literal, das bei `CAST` zu Datum/Zeit das *gestrige* Datum ergibt.

## Transaction Control

Anweisungen zur Steuerung von Transaktionen. Werden typischerweise von Client-Anwendungen verwendet, nicht direkt in PSQL (außer Savepoints und Autonomous Transactions).

*   **`SET TRANSACTION`:** Startet und konfiguriert eine neue Transaktion (Isolation Level, Access Mode, Lock Resolution, etc.). Siehe Abschnitt 10.1.1 im OCR für Details zu Optionen wie `READ WRITE`/`READ ONLY`, `WAIT`/`NO WAIT`, `ISOLATION LEVEL SNAPSHOT | READ COMMITTED [RECORD_VERSION | NO RECORD_VERSION]`, `LOCK TIMEOUT`, `RESERVING`.
*   **`COMMIT [WORK] [RELEASE] [RETAIN [SNAPSHOT]]`:** Beendet die aktuelle Transaktion erfolgreich und macht Änderungen permanent. `RETAIN` startet die Transaktion mit derselben ID neu (Soft Commit). `RELEASE` (nur ESQL) trennt die Verbindung.
*   **`ROLLBACK [WORK] [RELEASE] [RETAIN [SNAPSHOT] | TO [SAVEPOINT] sp_name]`:** Beendet die aktuelle Transaktion erfolglos und verwirft alle Änderungen. `RETAIN` startet die Transaktion mit derselben ID neu (Soft Rollback). `TO SAVEPOINT` rollt nur bis zum genannten Savepoint zurück. `RELEASE` (nur ESQL) trennt die Verbindung.
*   **`SAVEPOINT sp_name`:** Setzt einen benannten Sicherungspunkt innerhalb der aktuellen Transaktion (nur DSQL).
*   **`RELEASE SAVEPOINT sp_name [ONLY]`:** Löscht einen benannten Savepoint (und optional alle danach erstellten, wenn `ONLY` fehlt). Gibt Ressourcen frei (nur DSQL).

## Security

*   **User Authentication:** Erfolgt über die `security2.fdb`-Datenbank. Benutzername (case-insensitiv, max 31 Zeichen), Passwort (case-sensitiv, erste 8 Zeichen signifikant).
    *   **SYSDBA:** Superuser mit vollem Zugriff. Default-Passwort 'masterkey' (unbedingt ändern!).
    *   **Trusted Authentication:** Auf POSIX (via `/etc/hosts.equiv` oder `/etc/gds_hosts.equiv`) und Windows (Parameter `Authentication=Trusted|Mixed` in `firebird.conf`) möglich. Erlaubt Betriebssystem-Benutzern (z.B. `root` auf POSIX, Administratoren auf Windows unter bestimmten Bedingungen) den Zugriff.
    *   **Database Owner:** Der Benutzer, der die Datenbank erstellt hat, hat volle Admin-Rechte *innerhalb dieser Datenbank*.
    *   **RDB$ADMIN Role:** Systemrolle. Gewährt SYSDBA-Rechte *innerhalb einer spezifischen Datenbank*, wenn einem Benutzer zugewiesen. In `security2.fdb` gewährt sie Rechte zur Benutzerverwaltung via SQL.
    *   **AUTO ADMIN MAPPING:** (Windows) Erlaubt Windows-Administratoren bei Trusted Authentication automatisch die `RDB$ADMIN`-Rolle in einer Datenbank anzunehmen (muss pro DB aktiviert/deaktiviert werden, Default: deaktiviert). SQL: `ALTER ROLE RDB$ADMIN {SET | DROP} AUTO ADMIN MAPPING`. In `security2.fdb` via `gsec -mapping {set|drop}`.
*   **SQL Statements for User Management (erfordern Admin-Rechte):**
    *   `CREATE USER username PASSWORD 'password' [FIRSTNAME '...'] [MIDDLE...] [LAST...] [GRANT ADMIN ROLE]`
    *   `ALTER USER username [SET] [PASSWORD '...'] [FIRST...] [MIDDLE...] [LAST...] [{GRANT | REVOKE} ADMIN ROLE]`
    *   `DROP USER username`
*   **SQL Privileges:** Steuern den Zugriff auf Objekte *innerhalb* einer Datenbank.
    *   **Objekte:** Tabellen, Views, Prozeduren, Rollen etc.
    *   **Privilegien:** `SELECT`, `INSERT`, `UPDATE [(cols)]`, `DELETE`, `REFERENCES [(cols)]`, `EXECUTE` (für Prozeduren), `USAGE` (für Domains, Charsets, Collations, Exceptions - nicht explizit grantbar in 2.5, implizit). `ALL [PRIVILEGES]` fasst die relevanten zusammen.
    *   **`GRANT` Statement:** Verleiht Rechte.
        ```sql
        GRANT {<privileges> ON [TABLE] object | EXECUTE ON PROCEDURE proc} TO <grantee_list> [WITH GRANT OPTION] [GRANTED BY grantor];
        GRANT <role_granted> TO <role_grantee_list> [WITH ADMIN OPTION] [GRANTED BY grantor];
        ```
    *   **`REVOKE` Statement:** Entzieht Rechte.
        ```sql
        REVOKE [GRANT OPTION FOR] {<privileges> ON [TABLE] object | EXECUTE ON PROCEDURE proc} FROM <grantee_list> [GRANTED BY grantor];
        REVOKE [ADMIN OPTION FOR] <role_granted> FROM <role_grantee_list> [GRANTED BY grantor];
        REVOKE ALL ON ALL FROM <grantee_list>; -- Nur Admin
        ```
    *   **Grantees:** `[USER] username`, `[ROLE] rolename`, `PROCEDURE procname`, `TRIGGER trigname`, `VIEW viewname`, `PUBLIC`.
    *   `PUBLIC`: Spezieller Grantee, der alle authentifizierten Benutzer repräsentiert.
    *   `WITH GRANT OPTION`: Erlaubt dem Grantee, die erhaltenen Rechte weiterzuvergeben.
    *   `WITH ADMIN OPTION`: Erlaubt dem Grantee, die erhaltene Rolle weiterzuvergeben.
    *   `GRANTED BY`: Erlaubt Admins, Rechte im Namen eines anderen Benutzers zu vergeben/entziehen. `AS` ist Synonym.

