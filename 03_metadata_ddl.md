# Moduł 03 — Schemat bazy w kodzie: `MetaData`, `Table`, `Column` i DDL

W tym module przestajemy rozmawiać z bazą pojedynczymi, ręcznie pisanymi zapytaniami, a zaczynamy **opisywać strukturę bazy** — jej tabele, kolumny, typy i reguły — w kodzie Pythona. Nauczysz się, czym jest schemat bazy i dlaczego trzymanie go w jednym miejscu w kodzie ratuje projekt przed chaosem; poznasz `MetaData`, `Table`, `Column` oraz cały zestaw ograniczeń (`constraints`), które pilnują poprawności danych; zobaczysz, jak z takiego opisu wygenerować prawdziwe DDL (`CREATE TABLE`), jak odczytać istniejącą bazę z powrotem do Pythona (refleksja) i kiedy ten cały mechanizm przestaje wystarczać, wypychając nas w stronę migracji. To moduł „rysowania planu budynku” — zanim cokolwiek zbudujemy i zanim cokolwiek wypełnimy danymi.

---

**Poziom:** 🟢 podstawowy
**Czas:** ~150 minut (plus ćwiczenia, ~40 minut)
**Wymagania wstępne:** [`02_srodowisko_i_engine.md`](02_srodowisko_i_engine.md) — wiesz, jak utworzyć `Engine` i wykonać zapytanie przez `Connection`. Mile widziana (ale nie obowiązkowa) znajomość [modułu 01](01_wprowadzenie.md) — pojęcia Core i ORM.
**Czego dotyczy ten plik:** definiowania schematu bazy danych w SQLAlchemy Core — od `MetaData` przez `Table`, `Column`, typy i ograniczenia, aż po `create_all`, refleksję i świadomą decyzję „DDL w kodzie czy migracje”.

---

## Spis treści

1. [Wprowadzenie: świat bez schematu w kodzie](#wprowadzenie-świat-bez-schematu-w-kodzie)
2. [Czym jest schemat bazy danych](#czym-jest-schemat-bazy-danych)
3. [`MetaData` — teczka na definicje](#metadata--teczka-na-definicje)
4. [`Table` i `Column` — anatomia krok po kroku](#table-i-column--anatomia-krok-po-kroku)
5. [Typy danych: od Pythona do SQL](#typy-danych-od-pythona-do-sql)
6. [Ograniczenia, czyli reguły domu](#ograniczenia-czyli-reguły-domu)
7. [Klucze: naturalne i sztuczne](#klucze-naturalne-i-sztuczne)
8. [Nazwy ograniczeń i `naming_convention`](#nazwy-ograniczeń-i-naming_convention)
9. [Tworzenie schematu: `create_all`, `drop_all`, kolejność](#tworzenie-schematu-create_all-drop_all-kolejność)
10. [Refleksja: czytanie istniejącej bazy](#refleksja-czytanie-istniejącej-bazy)
11. [DDL w kodzie czy migracje?](#ddl-w-kodzie-czy-migracje)
12. [Pełny przykład: schemat sklepu](#pełny-przykład-schemat-sklepu)
13. [Podsumowanie](#podsumowanie)
14. [Ćwiczenia](#ćwiczenia)
15. [Najczęstsze błędy i jak je czytać](#najczęstsze-błędy-i-jak-je-czytać)
16. [Słowniczek modułu](#słowniczek-modułu)
17. [Dalsze czytanie](#dalsze-czytanie)
18. [Co dalej](#co-dalej)

---

## Wprowadzenie: świat bez schematu w kodzie

Wyobraź sobie, że budujesz niewielki sklep internetowy. Na początku idzie łatwo: w module 02 nauczyliśmy się wykonywać surowe zapytania przez `engine.connect()` i `text()`. Pierwsza tabela powstaje z zapytania skopiowanego z poradnika:

```python
# examples/03a_legacy_approach.py — TAK NIE ROBIĆ (pokazujemy problem, nie wzorzec)
from sqlalchemy import Engine, text


def create_customers_table(engine: Engine) -> None:
    """Tworzy tabelę klientów surowym SQL-em."""
    with engine.begin() as conn:
        conn.execute(
            text(
                """
                CREATE TABLE IF NOT EXISTS customers (
                    id INTEGER PRIMARY KEY,
                    email TEXT NOT NULL,
                    full_name TEXT NOT NULL
                )
                """
            )
        )
```

Działa. Tydzień później okazuje się, że e-mail powinien być unikalny. Dodajesz drugie zapytanie — tym razem `ALTER TABLE`. Za miesiąc ktoś inny dodaje tabelę `orders` i pisze `customer_id INTEGER` — a ponieważ zapomniał o `REFERENCES customers(id)`, baza pozwala zapisać zamówienie dla klienta, który nie istnieje. Za kolejny miesiąc ten sam e-mail ma już dwa formaty zapisu („a@b.pl” oraz „a@b.pl ”), bo nikt nie pilnował normalizacji. A kiedy trzeba postawić środowisko testowe, nikt nie wie, **które** z piętnastu rozrzuconych po projekcie zapytań `CREATE TABLE` trzeba wykonać i w jakiej kolejności.

To nie jest problem „słabego SQL-a”. To problem **braku jednego źródła prawdy o strukturze bazy**.

> 💡 **Analogia — projekt domu a dom.**
> Schemat bazy danych to **projekt architektoniczny**: rzuty pięter, wymiary drzwi, rozstaw ścian, lokalizacja pionów wodnych. Sama baza danych wypełniona tabelami i wierszami to **wybudowany dom**.
> Możesz zbudować dom bez projektu — stawiając ściany „na oko” i dorabiając drzwi tam, gdzie akurat sąsiad poprosi. Będzie stał. Ale przy pierwszym remoncie (nowe okno, przesunięcie ściany) nie będziesz wiedzieć, co jest nośne, a co nie — i każda zmiana stanie się ryzykowną improwizacją. Projekt trzymany osobno, w jednym miejscu, pozwala **przewidzieć konsekwencje zmian przed ich wykonaniem**.
> SQLAlchemy Core daje ci ten projekt w postaci kodu Pythona — czytelnego, wersjonowanego razem z resztą aplikacji i możliwego do porównania z rzeczywistym stanem budynku.

W tym module zbudujemy projekt. Nie będziemy zajmować się danymi (to zadanie modułów 04 i 05) — tylko konstrukcją.

---

## Czym jest schemat bazy danych

Zacznijmy precyzyjnie. **Schemat (schema)** w kontekście, o którym mówimy, to *opis* struktury danych: jakie tabele istnieją, jakie mają kolumny, jakiego typu, jakie reguły muszą spełniać wiersze, jak tabele są ze sobą powiązane.

Słowo „schema” ma jeszcze drugie, węższe znaczenie w PostgreSQL — „przestrzeń nazw”, czyli katalog grupujący tabele (np. `public.users`). W tym module używamy pierwszego znaczenia. To wąskie znaczenie wróci w modułach zaawansowanych, gdy będziemy pracować z wieloma aplikacjami na jednej bazie.

### Trzy warstwy prawdy o strukturze

Warto zapamiętać ten trójpodział, bo wraca przez cały kurs:

| Warstwa | Co opisuje | W jakim języku | Kto jest właścicielem |
|---|---|---|---|
| **Model pojęciowy** | „w sklepie mamy klientów, produkty i zamówienia” | język biznesu, rysunek na tablicy | zespół / analityk |
| **Schemat logiczny** | tabele, kolumny, typy, klucze, ograniczenia | SQL-owy DDL (odwzorowany w `MetaData`/`Table`) | programista |
| **Realizacja fizyczna** | pliki, strony, indeksy B-drzewa, partycje | wewnętrzna baza | baza danych / DBA |

SQLAlchemy Core operuje w warstwie drugiej. Nie musisz (i nie powinieneś) zajmować się warstwą trzecią w typowym projekcie — baza zrobi to sama.

> 🧠 **Dlaczego tak jest —** bazy danych są od dekad projektowane jako *deklaratywne*: mówisz **co** chcesz mieć („e-mail musi być unikalny”), a nie **jak** to osiągnąć („utwórz drzewo B+ o 12 poziomach i sprawdzaj przy każdym INSERT”). To fundament, który pozwala SQL-owi przetrwać 50 lat i działać szybko na sprzęcie, o którym jego autorzy nie mogli marzyć.

### DDL — język opisu struktury

**DDL** (*Data Definition Language*) to podjęzyk SQL-a służący wyłącznie do opisu i zmiany struktury. Należą do niego:

- `CREATE TABLE`, `ALTER TABLE`, `DROP TABLE` — tabele,
- `CREATE INDEX`, `DROP INDEX` — indeksy,
- `CREATE VIEW`, `DROP VIEW` — widoki,
- `CREATE SEQUENCE` — sekwencje liczbowe,
- `CREATE SCHEMA` — przestrzenie nazw.

W odróżnieniu od **DML** (*Data Manipulation Language*: `SELECT`, `INSERT`, `UPDATE`, `DELETE`), który operuje na *wierszach*, DDL operuje na *pojemnikach* na wiersze.

>  **Dlaczego to rozróżnienie jest ważne w praktyce —** DDL w większości baz jest operacją „ciężką”: często wymaga blokady tabeli (przez chwilę nikt inny nie może z niej czytać ani pisać). DML jest zwykle operacją lekką. Dlatego na produkcji `ALTER TABLE` planuje się jak wdrożenie, a `UPDATE` wykonuje się na bieżąco. Wrócimy do tego w module 16 i 17.

W SQLAlchemy tym, co odpowiada DDL-owi, jest warstwa `Table` + `MetaData`. A sam DDL można z tych obiektów *wygenerować* — i to jest kluczowa właściwość: **jedno źródło prawdy, wiele dialektów wyjściowych**.

---

## `MetaData` — teczka na definicje

### Problem: gdzie trzymać definicje tabel?

Gdyby każda tabela była niezależnym obiektem w Pythonie, prędzej czy później pojawiłby się problem: jak znaleźć „wszystkie tabele”? Jak utworzyć je wszystkie naraz? Jak sprawdzić, czy istnieją dwie tabele o tej samej nazwie? Potrzebujemy **kolekcji**.

W SQLAlchemy tę kolekcję nazywamy `MetaData`.

```python
# examples/03b_metadata_intro.py
from sqlalchemy import MetaData

metadata = MetaData()
print(type(metadata))          # <class 'sqlalchemy.sql.schema.MetaData'>
print(len(metadata.tables))    # 0 — na razie pusta
```

> 💡 **Analogia — segregator z kartami pacjentów.**
> `MetaData` to **segregator**. Tabele to **karty pacjentów** w tym segregatorze. Możesz mieć segregator z kartami pacjentów i drugi, osobny segregator z kartami pracowników (dwie niezależne bazy) — ale **jedna baza danych = jeden segregator**. Kartę można wyjąć i przenieść tylko razem z całą wiedzą o tym, do czego się odwoływała (klucze obce!), więc segregatory trzyma się raczej stabilnie.

### Co `MetaData` faktycznie robi

`MetaData` to obiekt, który:

1. **Rejestruje tabele** — każdy `Table` dołączony do niego trafia do słownika `metadata.tables`.
2. **Pilnuje unikalności nazw** — próba dodania drugiej tabeli o tej samej nazwie to błąd (`InvalidRequestError`), a nie cicha nadpiska.
3. **Zna kolejność zależności** — potrafi posortować tabele tak, by tworzyć je w kolejności uwzględniającej klucze obce (`metadata.sorted_tables`).
4. **Przechowuje konfigurację wspólną** — m.in. `naming_convention` (nazwy ograniczeń) i `schema` (domyślna przestrzeń nazw).
5. **Jest punktem wejścia do operacji masowych** — `metadata.create_all(engine)`, `metadata.drop_all(engine)`, `metadata.reflect(engine)`.

```python
# examples/03c_metadata_api.py
from sqlalchemy import MetaData

metadata = MetaData()

print(list(metadata.tables.keys()))   # [] — pusty segregator
print(list(metadata.sorted_tables))   # [] — nic do posortowania
print(metadata.info)                  # {} — worek na własne metadane (rzadko używany)
```

> ⚠️ **Pułapka —** `metadata.tables` kluczuje tabele **nazwą, a nie obiektem**. Jeśli używasz wielu schematów (przestrzeni nazw), klucz przyjmie postać `"shop.customers"`. Nie zakładaj, że klucz to zawsze prosta nazwa tabeli.

### Jedna baza = jedno `MetaData`

W prostym projekcie obowiązuje żelazna zasada: **jedna baza danych → jeden obiekt `MetaData` → jedna klasa bazowa modeli** (jeśli używasz ORM — zobaczysz to w module 07).

Jeśli zaczniesz tworzyć `MetaData()` w kilku miejscach, prędzej czy później:

- `create_all` nie zobaczy części tabel (bo są w innym segregatorze),
- klucz obcy wskaże tabelę, której nie ma w tym samym „świecie”,
- migracje (moduł 16) nie będą wiedzieć, co jest prawdą.

Dopuszczalny wyjątek: **dwie naprawdę różne bazy** (np. własna baza aplikacji + zewnętrzna baza raportowa tylko do odczytu). Wtedy dwa `MetaData` — i to jest w porządku, bo to dwa różne światy.

```python
# examples/03d_two_worlds.py
from sqlalchemy import MetaData

app_metadata = MetaData()        # baza aplikacji: tu piszemy
report_metadata = MetaData()     # baza raportowa: tylko czytamy (refleksja)

print(app_metadata is report_metadata)   # False — dwa niezależne segregatory
```

---

## `Table` i `Column` — anatomia krok po kroku

### Najprostsza tabela ever

```python
# examples/03e_first_table.py
from sqlalchemy import Column, Integer, MetaData, String, Table

metadata = MetaData()

authors = Table(
    "authors",                     # 1. nazwa tabeli w bazie
    metadata,                      # 2. segregator, do którego należy
    Column("id", Integer, primary_key=True),      # 3. kolumna
    Column("name", String(120), nullable=False),  # 4. kolejna kolumna
)

print(authors.name)                            # authors
print([c.name for c in authors.columns])       # ['id', 'name']
print(authors.c.id is authors.columns.id)      # True — dwa aliasy tego samego
```

Zatrzymajmy się i rozłóżmy to na części, bo **każdy** argument ma znaczenie.

### Argument pierwszy: nazwa tabeli

Nazwa trafia **dosłownie** do bazy jako identyfikator. Obowiązują tu reguły bazy, nie Pythona:

- PostgreSQL: nienazwane identyfikatory są zamieniane na małe litery (`Customers` → `customers`). To klasyczna pułapka przy refleksji i ręcznie pisanych zapytaniach.
- Nazwy z myślnikami, polskimi znakami czy spacjami są technicznie możliwe (cudzysłowy), ale proszą się o kłopoty. Trzymaj się konwencji `snake_case` i ASCII.

> ⚠️ **Pułapka —** `Table("Customers", ...)` w PostgreSQL createuje tabelę o nazwie `customers` (bo nie użyto cudzysłowów), ale SQLAlchemy **zapamięta** literę `C`. Refleksja tej bazy zwróci `customers`, a próba porównania „ta sama tabela czy nie” może zwrócić fałsz. Używaj `snake_case` i małych liter, a problem nie istnieje.

### Argument drugi: `MetaData`

Drugi argument pozycyjny to segregator. To on sprawia, że tabela „istnieje w świecie” projektu.

> 🧠 **Dlaczego tak jest —** `Table` nie może samodzielnie wygenerować DDL-u ani wiedzieć o innych tabelach. Dopiero `MetaData` daje mu kontekst: kto jeszcze tu jest, w jakiej kolejności tworzyć, jaka jest konwencja nazewnicza. To klasyczne rozdzielenie odpowiedzialności: `Table` opisuje **siebie**, `MetaData` opisuje **zbiór**.

### Argumenty dalsze: `Column`

`Column` to opis jednej kolumny. Sygnatura w praktyce sprowadza się do:

```text
Column(nazwa, typ, **flagi_i_ograniczenia)
```

- **nazwa** — identyfikator kolumny w bazie,
- **typ** — obiekt typu (np. `Integer()`, `String(50)`); o typach niżej,
- **flagi** — `primary_key`, `nullable`, `unique`, `index`, `default`, `server_default`, `onupdate` itd.

### Trzeci dostęp: `authors.c` i `authors.columns`

Po utworzeniu tabeli masz dwa sposoby sięgania do kolumn:

```python
# examples/03f_column_access.py
from sqlalchemy import Column, Integer, MetaData, String, Table

metadata = MetaData()
authors = Table(
    "authors",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String(120), nullable=False),
)

# Iteracja i lista nazw
print([c.name for c in authors.c])            # ['id', 'name']

# Dostęp po nazwie
print(authors.c.name.type)                    # VARCHAR(120)
print(authors.c.name.nullable)                # False
print(authors.c.id.primary_key)               # True

# Dostęp przez dict (przydatne przy budowaniu kodu dynamicznie)
print(list(authors.columns.keys()))           # ['id', 'name']

# Sprawdzenie, czy kolumna istnieje — bez wyjątku
col = authors.c.get("missing")
print(col)                                    # None
```

Te obiekty (`authors.c.name`) to **atrybuty instrumentowane** (*instrumented attributes*) — takie same jak te, których będziesz używać w warunkach `where()` w module 04. Ta sama kolumna służy więc i do opisu (DDL), i do budowania zapytań (DML). To jedna z najważniejszych decyzji projektowych SQLAlchemy.

### Ramka: `Column` a `column()`

W SQLAlchemy istnieją dwa podobne obiekty: `Column` (duże C) i `column` (małe c).

| | `Column` | `column()` |
|---|---|---|
| Do czego | definiowanie **tabeli** (DDL) | szybkie wyrażenia w zapytaniu (DML) |
| Czy należy do tabeli | tak, po dodaniu do `Table` | nie, to „wolna” kolumna |
| Czy ma pełne DDL-owe flagi | tak | nie |
| Kiedy używać | schemat, modele | jednorazowe wyrażenia, `text()`-owe hybrydy |

W praktyce: definiując schemat używasz `Column`. `column()` pojawia się rzadko i w tym kursie będziemy go unikać, żeby nie wprowadzać zamieszania.

> 🆕 **SQLAlchemy 2.1 —** w 2.1 rozszerzono lepsze typowanie (`PEP 646`) dla `Result`/`Row`, dzięki czemu statyczne narzędzia lepiej wnioskują typy kolumn przy dostępie po `select(...)`. W 2.0 działa to poprawnie, ale w węższym zakresie — nie wpływa to na działanie kodu, wyłącznie na podpowiedzi w IDE.

---

## Typy danych: od Pythona do SQL

Typ kolumny to nie kosmetyka. To **kontrakt**: co baza przyjmie, co zapamięta i co ci zwróci.

> 💡 **Analogia — typy jak pojemniki w kuchni.**
> Kolumna `String(50)` to słoik o znanej pojemności 50 ml. Kolumna `Text` to worek bez dna. Kolumna `Numeric(10, 2)` to waga kuchenna z dokładnością do 0,01 g. Kolumna `Boolean` to przełącznik: tak/nie.
> Możesz wsypać mąkę do słoika na przyprawy — w końcu są podobne — ale potem nie będziesz wiedzieć, gdzie czego szukać ani czy coś się nie wysypało. **Typ kolumny mówi bazie i przyszłym czytelnikom twojego kodu, co to za dane.**

### Tabela mapowania typów

To najważniejsza tabela w tym module. Kolumna „Python (typowy)” pokazuje, jaką wartość naturalnie wkładasz i wyciągasz.

| SQLAlchemy | Python (typowo) | SQLite (DDL) | PostgreSQL (DDL) | Uwagi |
|---|---|---|---|---|
| `Integer` | `int` | `INTEGER` | `INTEGER` / `SERIAL` dla PK | Podstawa kluczy głównych |
| `SmallInteger` | `int` | `SMALLINT` | `SMALLINT` | Gdy naprawdę liczysz bajty |
| `BigInteger` | `int` | `BIGINT` | `BIGINT` / `BIGSERIAL` | ⚠️ na SQLite **nie autoinkrementuje** PK |
| `String(n)` | `str` | `VARCHAR(n)` | `VARCHAR(n)` | `n` = limit znaków |
| `Text` | `str` | `TEXT` | `TEXT` | Bez limitu długości |
| `Unicode(n)` | `str` | `VARCHAR(n)` | `VARCHAR(n)` | Historyczne; `String` obsługuje Unicode |
| `Boolean` | `bool` | `BOOLEAN` (0/1) | `BOOLEAN` | SQLite nie ma osobnego typu |
| `Numeric(p, s)` | `Decimal` | `NUMERIC(p, s)` | `NUMERIC(p, s)` | ✅ pieniądze |
| `Float` | `float` | `FLOAT` | `FLOAT`/`DOUBLE PRECISION` | ❌ nie do pieniędzy |
| `DateTime` | `datetime.datetime` | `DATETIME` | `TIMESTAMP` | `timezone=True` w PG daje `TIMESTAMPTZ` |
| `Date` | `datetime.date` | `DATE` | `DATE` | Bez godziny |
| `Time` | `datetime.time` | `TIME` | `TIME` | Bez daty |
| `Interval` | `datetime.timedelta` | `DATETIME` (jako liczba) | `INTERVAL` | Semantyka różni się między dialektami |
| `LargeBinary` | `bytes` | `BLOB` | `BYTEA` | Pliki, obrazy, klucze kryptograficzne |
| `JSON` | `dict` / `list` | `JSON` (TEXT) | `JSON` (`JSONB` po `postgresql.JSONB`) | Serializacja automatyczna |
| `Uuid` | `uuid.UUID` | `CHAR(32)` | `UUID` | Nowość 2.0; natywna w PG |
| `Enum("a","b")` | `str` lub `enum.Enum` | `VARCHAR` + `CHECK` | natywny typ `ENUM` | Migracje utrudnione (moduł 16) |

```python
# examples/03g_types_tour.py
import uuid
from datetime import date, datetime, time, timedelta
from decimal import Decimal

from sqlalchemy import (
    JSON,
    Boolean,
    Column,
    Date,
    DateTime,
    Integer,
    Interval,
    LargeBinary,
    MetaData,
    Numeric,
    String,
    Table,
    Text,
    Time,
    Uuid,
)

metadata = MetaData()

sample = Table(
    "sample",
    metadata,
    Column("id", Uuid, primary_key=True, default=uuid.uuid4),
    Column("label", String(50), nullable=False),
    Column("body", Text, nullable=True),
    Column("is_active", Boolean, nullable=False, default=True),
    Column("price", Numeric(10, 2), nullable=False),
    Column("created_at", DateTime, nullable=False, default=datetime.utcnow),
    Column("birth_date", Date, nullable=True),
    Column("alarm_at", Time, nullable=True),
    Column("grace", Interval, nullable=True),
    Column("payload", JSON, nullable=True),
    Column("checksum", LargeBinary, nullable=True),
)

for column in sample.columns:
    print(f"{column.name:12} -> {column.type!r}")
# id           -> Uuid()
# label        -> String(length=50)
# body         -> Text()
# is_active    -> Boolean()
# price        -> Numeric(precision=10, scale=2)
# created_at   -> DateTime()
# birth_date   -> Date()
# alarm_at     -> Time()
# grace        -> Interval()
# payload      -> JSON()
# checksum     -> LargeBinary()
```

### `String(50)` czy `Text`?

To pytanie zadaje sobie każdy początkujący. Odpowiedź zależy od tego, czy długość **jest częścią kontraktu biznesowego**.

| Sytuacja | Wybór | Dlaczego |
|---|---|---|
| Kod pocztowy, ISBN, numer faktury | `String(n)` z konkretnym `n` | Długość jest regułą domenową |
| Nazwisko, e-mail, nazwa miasta | `String(120)`–`String(255)` | Praktyczna granica, chroni przed absurdami |
| Opis produktu, treść artykułu, notatka | `Text` | Naprawdę nie wiesz, jak długie będzie |
| Token, hash, klucz API | `String(64)` / `String(255)` | Format jest ustalony technicznie |

>  **Dlaczego tak jest —** PostgreSQL nie robi praktycznie żadnej różnicy wydajnościowej między `VARCHAR(n)` i `TEXT` (oba to ten sam mechanizm przechowywania). Ograniczenie długości to głównie *dokumentacja* i *walidacja za darmo*: gdy aplikacja ma błąd i próbuje wstawić 4000-znakowy „e-mail”, baza go odrzuci, zamiast wpuścić śmieć do systemu.
> W SQLite sytuacja jest jeszcze prostsza: `VARCHAR(50)` **nie jest** egzekwowane (SQLite sprawdza tylko „powinno być tekstem”). Ograniczenie długości działa tam wyłącznie jako dokumentacja. To przykład różnicy między dialektami, którą trzeba znać.

> ⚠️ **Pułapka —** kwoty trzymaj w `Numeric(10, 2)`, który w Pythonie daje `Decimal`. Typ `Float` to liczba zmiennoprzecinkowa — `0.1 + 0.2 != 0.3`. System fakturowania zbudowany na `Float` w końcu policzy komuś o grosz za dużo.

> ⚠️ **Pułapka —** `DateTime` **bez** `timezone=True` w PostgreSQL to `TIMESTAMP WITHOUT TIME ZONE`. Wstawisz czas z jednego serwera, odczytasz na drugim w innej strefie i dostaniesz ciszę oraz błędne „o 2 godziny starsze” zamówienie. Zasada praktyczna: **jeśli zapisujesz moment w czasie, używaj UTC i `DateTime(timezone=True)`**.

---

## Ograniczenia, czyli reguły domu

Jeśli typ kolumny mówi „co to za dane”, to **ograniczenie** (*constraint*) mówi „co wolno”. Ograniczenia to najtańsze ubezpieczenie, jakie możesz kupić: działają **zawsze**, niezależnie od tego, czy ktoś napisał INSERT z aplikacji, z ręcznego `psql`, ze skryptu migracyjnego czy z importu CSV.

> 💡 **Analogia — regulatory w kuchni.**
> Typ kolumny to pojemnik. Ograniczenie to **regulamin kuchni**: „nie wstawiamy surowego mięsa do szafki z sałatą”, „każdy pojemnik musi mieć etykietę”, „nic nie stoi dłużej niż 3 dni”. Regulamin działa nawet wtedy, gdy kucharz jest zmęczony i zapomniał. Aplikacja może sprawdzić reguły szybciej (i ładniej powiedzieć użytkownikowi „ten e-mail jest już zajęty”), ale to baza jest ostatnią linią obrony.

### `primary_key` — klucz główny

```python
Column("id", Integer, primary_key=True)
```

Klucz główny to kolumna (lub zestaw kolumn) **jednoznacznie identyfikująca wiersz**. Baza automatycznie:

- tworzy **indeks** na tej kolumnie (wyszukiwanie po ID jest szybkie),
- wymusza **unikalność** (nie dwa wiersze o tym samym `id`),
- wymusza **NOT NULL** (wiersz bez tożsamości nie ma sensu).

W PostgreSQL `Integer` jako PK renderuje się jako `SERIAL` (lub `BIGSERIAL` dla `BigInteger`) — baza sama nada kolejne numery. Na SQLite, kolumna zadeklarowana dokładnie jako `INTEGER PRIMARY KEY` staje się aliasem „rowid” i również numeruje się sama.

```text
-- PostgreSQL (uproszczone)
CREATE TABLE authors (
    id SERIAL NOT NULL,
    name VARCHAR(120) NOT NULL,
    PRIMARY KEY (id)
);
```

> 🧠 **Dlaczego tak jest —** w SQLAlchemy 2.0 dialekt PostgreSQL domyślnie używa `SERIAL` dla całkowitoliczbowego PK. Jeśli wolisz nowocześniejszą, standardową konstrukcję, użyj `Identity()`:
> ```python
> Column("id", Integer, Identity(), primary_key=True)
> ```
> Wygeneruje ona `GENERATED BY DEFAULT AS IDENTITY`. Obie wersje są poprawne; `SERIAL` jest starszy, ale głęboko przyjęty. Ważne dla migracji (moduł 16) — nie mieszaj ich w jednej tabeli.

> ⚠️ **Pułapka —** `BigInteger` jako klucz główny **nie zadziała** na SQLite. SQLite autoinkrementuje wyłącznie kolumnę o typie dokładnie `INTEGER PRIMARY KEY`; `BIGINT PRIMARY KEY` przyjmie `NULL` i zgłosi błąd `NOT NULL constraint failed`. Rozwiązanie: `Column("id", BigInteger().with_variant(Integer, "sqlite"), primary_key=True)`.

### `nullable` — czy wolno nic nie wpisać

```python
Column("email", String(255), nullable=False)
```

- `nullable=False` → kolumna MUSI mieć wartość (DDL: `NOT NULL`).
- `nullable=True` (domyślnie!) → kolumna może być pusta (`NULL`).

> ⚠️ **Pułapka — najczęstsza w całym module.** Domyślna wartość to `nullable=True`. Jeśli o niej zapomnisz, **każda** kolumna będzie przyjmować `NULL` — również ta, która logicznie nigdy nie powinna być pusta. Zaleta to szybkość prototypowania, wada to ciche błędy w produkcji.
> Reguła praktyczna: w kodzie produkcyjnym **pisz `nullable` jawnie przy każdej kolumnie**. Jeśli wolisz przeciwną domyślność, w SQLAlchemy 2.0 możesz użyć `MetaData()`… a dokładnie: nie ma globalnego przełącznika w Core. W ORM (moduł 07) `Mapped[str]` domyślnie daje `NOT NULL`, a `Mapped[str | None]` — kolumnę opcjonalną. Kolejny powód, by docenić adnotacje typów.

Czym jest `NULL` w ogóle? To nie „zero”, nie „pusty napis”, nie „fałsz”. To **brak informacji**. `NULL != NULL` w SQL-u — dlatego w module 04 zobaczysz, że porównanie z `NULL` wymaga `is_(None)`, a nie `== None`.

### `unique` — brak duplikatów

```python
Column("email", String(255), nullable=False, unique=True)
```

Baza utworzy unikalny indeks i odrzuci drugi wiersz z tym samym e-mailem.

> 🆕 **SQLAlchemy 2.1 / Alembic 1.19 —** Alembic 1.19.x potrafi wykrywać **nazwane** ograniczenia `CHECK` w autogenerate. Historycznie był to obszar wymagający ręcznej korekty po wygenerowaniu migracji. Zachęta: nazywaj swoje CHECK-i.

### `default` vs `server_default` — cały podrozdział

To najczęstsze źródło nieporozumień w całym dziale „schemat i dane”. Rozróżnienie wygląda banalnie, ale konsekwencje są duże.

**`default=` — domyślna wartość wyliczana w Pythonie.**

```python
Column("created_at", DateTime, nullable=False, default=datetime.utcnow)
```

Mechanizm: gdy SQLAlchemy buduje `INSERT`, brak wartości w tym polu zostaje **uzupełniony w Pythonie**, a wartość trafia do zapytania jako zwykły parametr.

```text
-- 🔬 Pod maską: INSERT z default= (wartość widoczna w zapytaniu)
INSERT INTO articles (title, created_at) VALUES (?, ?)
-- parametry: ('Tytuł', datetime.datetime(2026, 9, 18, 9, 14, 22, 413000))
```

Cecha kluczowa: **wartość nie istnieje w DDL** — nie ma jej w `CREATE TABLE`.

**`server_default=` — domyślna wartość po stronie bazy.**

```python
from sqlalchemy import func, text

Column("created_at", DateTime, nullable=False, server_default=func.now())
```

Mechanizm: wartość trafia do **DDL** jako klauzula `DEFAULT`. Aplikacja w ogóle jej nie wysyła (chyba że jawnie poda inną).

```text
-- 🔬 Pod maską: DDL zawiera DEFAULT
CREATE TABLE articles (
    id SERIAL NOT NULL,
    title VARCHAR(200) NOT NULL,
    created_at TIMESTAMP WITHOUT TIME ZONE DEFAULT now() NOT NULL,
    PRIMARY KEY (id)
);

-- 🔬 Pod maską: INSERT nie zawiera tej kolumny
INSERT INTO articles (title) VALUES (%(title)s)
```

| | `default=` | `server_default=` |
|---|---|---|
| Gdzie liczona wartość | w Pythonie (proces aplikacji) | w bazie danych |
| Czy jest w DDL | ❌ nie | ✅ tak (`DEFAULT ...`) |
| Widoczna dla innych klientów bazy | ❌ nie | ✅ tak |
| Wymaga, by ORM/Core wykonał INSERT | ✅ tak | ❌ nie |
| Typowy argument | wartość Pythona, callable (`datetime.utcnow`, `uuid.uuid4`) | `text("...")` lub `func.now()`, dla napisów też zwykły `str` |
| Kiedy używać | wartości zależne od logiki aplikacji | wartości techniczne: czas, liczniki, flagi |

> 💡 **Analogia — pieczątka czy zegar na ścianie?**
> `default=` to **pieczątka w ręce urzędnika**: coś musi ją przyłożyć (ktoś z aplikacji). Jeśli urzędnik nie przyjdzie do pracy (INSERT wykonany ręcznie z `psql` albo z innego systemu), pieczątki nie będzie.
> `server_default=` to **zegar na ścianie urzędu**: wisi tam zawsze, niezależnie od tego, kto wypełnia dokument. Każdy, kto tworzy dokument w tym urzędzie, widzi ten sam czas.

**Praktyczna rekomendacja dla nowego projektu:**

- `created_at` / `updated_at` → `server_default=func.now()` (czas bierze baza — jedna strefa, jedna prawda),
- `is_active` / licznik `version` → `server_default=text("true")` lub `text("0")` (newralgiczne przy migracjach, w których istniejące wiersze muszą dostać wartość!),
- wartości biznesowe (np. domyślny status zamówienia) → `server_default=text("'pending'")` **albo** `default="pending"` — zależnie od tego, czy istniejące wiersze mają dostać wartość przy dodaniu kolumny (zobacz moduł 16).

> 🔬 **Pod maską — jak `func.now()` staje się różnym SQL-em.**
> ```python
> from sqlalchemy import Column, DateTime, MetaData, func, Table
> from sqlalchemy.dialects import postgresql, sqlite
> from sqlalchemy.schema import CreateTable
>
> metadata = MetaData()
> t = Table("t", metadata, Column("id", DateTime, server_default=func.now()))
>
> print(CreateTable(t).compile(dialect=sqlite.dialect()))
> # CREATE TABLE t (id DATETIME DEFAULT (CURRENT_TIMESTAMP))
>
> print(CreateTable(t).compile(dialect=postgresql.dialect()))
> # CREATE TABLE t (id TIMESTAMP WITHOUT TIME ZONE DEFAULT now())
> ```
> To jest cała idea dialektów w jednym przykładzie: jedno `func.now()` w Pythonie, dwa różne DDL-e w bazach.

> ⚠️ **Pułapka — `server_default=text("now()")` na SQLite.** SQLite nie zna funkcji `now()`; zna `CURRENT_TIMESTAMP`. Taki DDL po prostu nie zadziała — a w niektórych wersjach SQLite błąd pojawi się dopiero przy `INSERT`, bo `DEFAULT (now())` jest przyjmowane bez walidacji. Używaj **`func.now()`** (SQLAlchemy przetłumaczy je na właściwą funkcję dialektu) zamiast wklejania nazwy funkcji z PostgreSQL do kodu, który ma działać na obu bazach.

> ⚠️ **Pułapka — typ wartości w `server_default`.** Ponieważ `server_default` renderuje się dosłownie, różnice dialektowe potrafią zaboleć:
> - `text("1")` → działa w SQLite, **wybuchnie** w PostgreSQL dla kolumny `BOOLEAN` (`DEFAULT 1` nie jest boolem);
> - `text("true")` → działa w PostgreSQL i w nowoczesnym SQLite (3.23+);
> - `text("uuid_generate_v4()")` → dotyczy tylko PostgreSQL;
> - `text("current_timestamp")` → działa szeroko, ale wolałbyś `func.now()`.
> Zasada: jeśli kod ma być przenośny, preferuj `func.now()` oraz `text("true")`/`text("false")` zamiast `1`/`0`.

### `onupdate` — automatyczna aktualizacja znacznika

```python
Column("updated_at", DateTime, nullable=False, server_default=func.now(), onupdate=func.now())
```

`onupdate=` jest **bliźniakiem `default=`, ale dla `UPDATE`**: gdy SQLAlchemy wykonuje aktualizację wiersza, dopisuje do zapytania aktualną wartość. Podobnie jak `default`, nie ma go w DDL.

> 🧠 **Dlaczego tak jest —** istnieje `server_onupdate=`, ale w PostgreSQL **nie generuje** klauzuli `ON UPDATE` (taka klauzula nie istnieje w tym dialekcie; PostgreSQL wymaga triggera). Jest to więc w większości dialektów konstrukcja deklaratywna/informacyjna. Jeśli naprawdę chcesz aktualizacji po stronie bazy, potrzebujesz triggera (moduł 06 i 16) albo kolumny `timestamp` ustawianej w aplikacji.
> Ważna konsekwencja: `onupdate=` **nie zadziała** przy aktualizacji wykonanej surowym SQL-em ani przez `session.execute(update(...))` (moduł 10) — Python nie bierze w niej udziału. Kolejny argument, by krytyczne znaczniki czasu ustawiać w jednym, znanym miejscu.

### `index` — szybkie szukanie

```python
Column("email", String(255), nullable=False, unique=True, index=True)
```

Indeks to dodatkowa struktura, która pozwala bazie znaleźć wiersz bez czytania całej tabeli.

> 💡 **Analogia — indeks na końcu książki.**
> Bez indeksu, żeby znaleźć słowo „transakcja”, musisz przewertować wszystkie 500 stron (*full table scan*). Z indeksem idziesz od razu na wskazaną stronę. Cena: indeks trzeba utrzymywać przy każdym dopisaniu strony — więc książki pisane na bieżąco są wolniejsze w pisaniu, ale szybsze w czytaniu.

Reguła praktyczna:

- indeksuj **klucze obce** (bo JOIN-y i „pokaż zamówienia klienta” filtrują po nich),
- indeksuj kolumny używane w `WHERE`, `ORDER BY`, `JOIN`,
- **nie** indeksuj wszystkiego — każdy indeks spowalnia zapisy i zajmuje miejsce,
- kolumny o bardzo niskiej różnorodności (np. `status` z 2 wartościami) zwykle nie potrzebują osobnego indeksu.

> 🧠 **Dlaczego tak jest —** w PostgreSQL klucz obcy **nie tworzy automatycznie indeksu** na kolumnie odwołującej się. To częste nieporozumienie (MySQL w tabelach InnoDB tworzy go). Skutek: `DELETE` na tabeli nadrzędnej musi przeskanować całą tabelę podrzędną, by sprawdzić ograniczenie — na dużych zbiorach to poważny problem. Dodawaj `index=True` na kolumnach FK.

### `ForeignKey` — powiązanie między tabelami

```python
Column("customer_id", Integer, ForeignKey("customers.id"), nullable=False)
```

Klucz obcy mówi: „wartość w tej kolumnie musi istnieć jako `id` w tabeli `customers`”. Baza tego dopilnuje — próba wstawienia zamówienia dla klienta `#999`, którego nie ma, zakończy się błędem `IntegrityError`.

Dodatkowe parametry mają praktyczne znaczenie:

```python
Column(
    "customer_id",
    Integer,
    ForeignKey(
        "customers.id",
        ondelete="CASCADE",     # gdy klient znika — jego zamówienia też
        onupdate="CASCADE",     # gdy zmieni się id klienta — FK się zaktualizuje
        deferrable=True,        # PostgreSQL: ograniczenie można odroczyć do commitu
        initially="DEFERRED",
    ),
    nullable=False,
    index=True,
)
```

| `ondelete` | Efekt przy usunięciu wiersza nadrzędnego |
|---|---|
| `"CASCADE"` | usuń wiersze podrzędne |
| `"SET NULL"` | ustaw FK na `NULL` (kolumna musi być `nullable=True`!) |
| `"RESTRICT"` / `"NO ACTION"` | zablokuj usunięcie (domyślne zachowanie) |
| `"SET DEFAULT"` | ustaw wartość domyślną |

> ️ **Pułapka — SQLite i klucze obce.** SQLite **nie wymusza** kluczy obcych, dopóki nie włączysz tego jawnie: `PRAGMA foreign_keys=ON` dla każdego połączenia. SQLAlchemy robi to za ciebie, gdy użyjesz zdarzenia `connect` (moduł 13), ale domyślnie — nie. Efekt: na SQLite możesz wstawić zamówienie dla nieistniejącego klienta i dowiesz się o tym dopiero na PostgreSQL. To główny powód, dla którego testy wyłącznie na SQLite bywają zdradliwe (moduł 18).

> ⚠️ **Pułapka — `ondelete` działa w bazie, nie w Pythonie.** `ondelete="CASCADE"` to DDL. Jeśli usuniesz obiekt z sesji ORM, SQLAlchemy może najpierw sam spróbować ustawić `NULL` na kluczach obcych (bo ma własne kaskady — moduł 09). Kaskada bazowa i kaskada ORM to dwa różne mechanizmy, które trzeba świadomie uzgodnić. Wrócimy do tego w module 09.

### `UniqueConstraint` i `CheckConstraint`

Gdy ograniczenie dotyczy **kilku kolumn** albo **wyrażenia**, nie mieści się w `Column(...)` — umieszczamy je na poziomie tabeli.

```python
from sqlalchemy import CheckConstraint, Column, Integer, MetaData, Numeric, Table, UniqueConstraint

metadata = MetaData()

order_items = Table(
    "order_items",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("order_id", Integer, nullable=False),
    Column("product_id", Integer, nullable=False),
    Column("quantity", Integer, nullable=False),
    Column("unit_price", Numeric(10, 2), nullable=False),
    # ten sam produkt może wystąpić w zamówieniu tylko raz
    UniqueConstraint("order_id", "product_id", name="order_product"),
    # ilość musi być dodatnia
    CheckConstraint("quantity > 0", name="quantity_positive"),
    # cena nie może być ujemna
    CheckConstraint("unit_price >= 0", name="unit_price_non_negative"),
)
```

> 🧠 **Dlaczego tak jest —** `UniqueConstraint("order_id", "product_id")` to ograniczenie **na parze** wartości, nie na każdej z osobna. Oznacza: „para (3, 7) może wystąpić raz; (3, 8) to inna para i też może wystąpić raz”. Gdybyś dał `unique=True` na obu kolumnach osobno, zabroniłbyś używania tego samego `product_id` w dwóch różnych zamówieniach — co jest oczywistą bzdurą.

`CheckConstraint` przyjmuje wyrażenie SQL jako tekst (lub wyrażenie SQLAlchemy). Baza sprawdza je przy każdym `INSERT` i `UPDATE`.

> ⚠️ **Pułapka —** `CHECK` w SQLite bywa ignorowany, jeśli baza została utworzona dawno lub kolumna zmieniono przez `ALTER TABLE` bez `render_as_batch` (moduł 16). Nie polegaj na `CHECK` jako jedynym mechanizmie walidacji; traktuj go jako ostatnią linię obrony, a walidację biznesową rób też w aplikacji (moduł 13).

> 🧠 **Dlaczego `CheckConstraint` są trudne w migracjach —** w wielu bazach nie da się ich „zmienić” (bo nie mają tożsamości), więc trzeba je usunąć i dodać ponownie — a to wymaga **nazwy**. Dlatego absolutnie od początku nadawaj nazwy (`name="quantity_positive"`). W SQLAlchemy pomaga w tym `naming_convention`.

### `Index` — jawne indeksy złożone

```python
from sqlalchemy import Index

# … wewnątrz Table(...)
Index("ix_orders_customer_id_placed_at", "customer_id", "placed_at")
```

Indeks złożony pomaga, gdy zapytanie filtruje po kilku kolumnach („zamówienia klienta X, posortowane po dacie”). **Kolejność kolumn ma znaczenie** — indeks `(customer_id, placed_at)` przyspiesza filtrowanie po `customer_id` i sortowanie po `placed_at`; indeks `(placed_at, customer_id)` już nie pomoże w „daj mi zamówienia klienta X”. Wrócimy do tego w module 17.

> 🧠 **`column.index=True` a `Index(...)`.** Pierwsze tworzy indeks jednej kolumny (nazwa wygenerowana z konwencji `ix`). Drugie pozwala opisać indeks wielokolumnowy, częściowy (`postgresql_where=`) czy z konkretną nazwą. Nie duplikuj — jeśli tworzysz jawne `Index` na kolumnie, nie ustawiaj jej `index=True`.

### Ramka: gdzie w Core jest `__table_args__`?

Być może słyszałeś o `__table_args__` z tutoriali SQLAlchemy. To konstrukcja **ORM-owa**, używana w klasach modeli (moduł 07) do przekazania ograniczeń i argumentów tabeli. W Core nie ma jej wcale: wszystkie ograniczenia przekazujesz **bezpośrednio do `Table(...)`** jako argumenty pozycyjne (constraints) lub przez `Index(...)`.

Zestawienie, które przyda się w module 07:

| Core (`Table`) | ORM (`__table_args__`) |
|---|---|
| `Table("t", md, ..., UniqueConstraint(...), CheckConstraint(...), Index(...))` | `__table_args__ = (UniqueConstraint(...), CheckConstraint(...), Index(...))` |
| `Table("t", md, schema="shop")` | `__table_args__ = {"schema": "shop"}` |
| `Table("t", md, ...)` na zewnątrz klasy | `Table` tworzona automatycznie z klasy |

---

## Klucze: naturalne i sztuczne

Klucz główny można wybrać na dwa sposoby i jest to decyzja projektowa, którą trudno później zmienić.

**Klucz naturalny** to coś, co już istnieje w świecie biznesu: PESEL, NIP, ISBN, adres e-mail, kod produktu SKU.

**Klucz sztuczny** (*surrogate key*) to dodatkowa kolumna bez znaczenia biznesowego: `id` jako liczba całkowita, `id` jako UUID.

| Kryterium | Klucz naturalny | Klucz sztuczny (`Integer` PK) | Klucz sztuczny (`Uuid` PK) |
|---|---|---|---|
| Zmienia się w czasie? | Często tak (e-mail!) | Nigdy | Nigdy |
| Trzeba go pokazać użytkownikowi? | Zwykle tak | Nie | Nie |
| Ujawnia liczbę rekordów? | Nie | Tak (`/users/1`, `/users/2`) | Nie |
| Koszt miejsca (PG) | Zależy | 4 bajty | 16 bajtów |
| Kolejność w indeksie | Zależna od danych | Sekwencyjna → wydajna | Losowa → fragmentacja |
| Kolizja przy scalaniu systemów | Możliwa | Bardzo prawdopodobna! | Praktycznie niemożliwa |

> 💡 **Analogia — numer PESEL a numer w kolejce.**
> Klucz naturalny to **numer PESEL**: istnieje niezależnie od tego, w jakim urzędzie go zapiszesz, ale towarzyszy człowiekowi na zawsze i zmienia się wtedy, gdy zmienia się jego sytuacja prawna (co jest koszmarem dla bazy).
> Klucz sztuczny to **numer w kolejce**: nadany losowo przy wejściu, bez znaczenia dla człowieka, ale idealny jako identyfikator techniczny. Nikt go nie „poprawi” dlatego, że się przeprowadził.

**Praktyczna rekomendacja,** którą przyjmujemy w tym kursie:

- **domyślnie `Integer` + autoinkrementacja** — szybko, prosto, wydajnie; idealne dla wewnętrznych aplikacji,
- **`Uuid`** gdy: dane są scalane z kilku systemów, identyfikatory są eksponowane w publicznym API (nie chcesz, by konkurencja liczyła twoich klientów), wiersze są generowane offline (klient mobilny tworzy zamówienie bez internetu),
- **`String` jako klucz naturalny** tylko wtedy, gdy naprawdę tego wymaga domena (np. `sku` bez możliwości zmiany), i nawet wtedy rozważ osobne `id` + `UNIQUE(sku)`.

```python
# examples/03h_id_strategies.py
import uuid

from sqlalchemy import Column, Integer, MetaData, String, Table, Uuid

metadata = MetaData()

# Wariant A: klasyczny sztuczny klucz liczbowy
users_numeric = Table(
    "users_numeric",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("email", String(255), nullable=False, unique=True),
)

# Wariant B: UUID jako klucz, generowany po stronie Pythona
users_uuid = Table(
    "users_uuid",
    metadata,
    Column("id", Uuid, primary_key=True, default=uuid.uuid4),
    Column("email", String(255), nullable=False, unique=True),
)

print(CreateTable(users_uuid).compile())
# CREATE TABLE users_uuid (
#     id CHAR(32) NOT NULL,
#     email VARCHAR(255) NOT NULL,
#     PRIMARY KEY (id),
#     UNIQUE (email)
# )
# W PostgreSQL ten sam kod wygeneruje:  id UUID NOT NULL
```

> 🆕 **SQLAlchemy 2.1 —** typ `Uuid` w 2.1 na SQLite potrafi korzystać z natywnego wsparcia UUID w nowych wersjach SQLite (lub zapisu binarnego) zamiast `CHAR(32)`, co zmniejsza rozmiar bazy. W 2.0 na SQLite obowiązuje `CHAR(32)`. Jeśli twoje dane zależą od formatu zapisu, sprawdź to przed aktualizacją SQLAlchemy.

---

## Nazwy ograniczeń i `naming_convention`

### Problem: ograniczenia bez nazw

PostgreSQL, MySQL i większość baz **same nadają nazwy** ograniczeniom, których nie nazwałeś. Brzmią wtedy mniej więcej tak:

```text
orders_customer_id_fkey
orders_status_check1
order_items_order_id_product_id_key
```

Albo, w SQLite, nie mają nazw wcale — a jeśli chcesz je zmienić, musisz odtworzyć całą tabelę.

Dlaczego to problem? Bo **migracje operują na nazwach**.

```sql
-- Żeby usunąć ograniczenie, trzeba je znać po imieniu:
ALTER TABLE orders DROP CONSTRAINT orders_status_check1;
```

Jeśli nazwy są nadawane automatycznie przez bazę w sposób, którego nie przewidzisz, to na produkcji (gdzie tabela istnieje od roku) nazwa może być inna niż w środowisku deweloperskim. Migracja działa u ciebie, a wywala się na serwerze.

### Rozwiązanie: `naming_convention`

`MetaData(naming_convention=...)` sprawia, że **każde** nowe ograniczenie dostaje nazwę według schematu, który sam opiszesz.

```python
# examples/03i_naming_convention.py
from sqlalchemy import MetaData

NAMING_CONVENTION: dict[str, str] = {
    # indeks: ix_orders_customer_id
    "ix": "ix_%(column_0_label)s",
    # ograniczenie unikalności: uq_customers_email
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    # klucz główny: pk_customers
    "pk": "pk_%(table_name)s",
    # klucz obcy: fk_orders_customer_id_customers
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    # CHECK: ck_products_price_non_negative
    "ck": "ck_%(table_name)s_%(constraint_name)s",
}

metadata = MetaData(naming_convention=NAMING_CONVENTION)
```

Dostępne tokeny (najważniejsze):

| Token | Znaczenie |
|---|---|
| `%(table_name)s` | nazwa tabeli, w której jest ograniczenie |
| `%(column_0_name)s` | nazwa pierwszej kolumny ograniczenia |
| `%(column_0_label)s` | etykieta kolumny, zwykle `tabela_kolumna` |
| `%(referred_table_name)s` | tabela, do której wskazuje klucz obcy |
| `%(constraint_name)s` | nazwa własna ograniczenia (dla `CheckConstraint` — nazwa, którą sam nadałeś) |

> 🧠 **Dlaczego token `ck` wymaga `%(constraint_name)s` —** ograniczenie `CHECK` nie ma kolumny, od której można by nazwę wyprowadzić; jego tożsamością jest własna nazwa. Dlatego w praktyce **każdy `CheckConstraint` powinien mieć `name=`**. W przeciwnym razie SQLAlchemy musi wyprowadzić nazwę z treści wyrażenia — co daje długie, brzydkie i nieprzewidywalne identyfikatory. Nazywaj CHECK-i ręcznie, zawsze.

### Jak to wygląda w DDL

```python
# examples/03j_naming_in_ddl.py
from sqlalchemy import (
    CheckConstraint,
    Column,
    ForeignKey,
    Integer,
    MetaData,
    Numeric,
    String,
    Table,
    UniqueConstraint,
)
from sqlalchemy.schema import CreateTable

metadata = MetaData(
    naming_convention={
        "ix": "ix_%(column_0_label)s",
        "uq": "uq_%(table_name)s_%(column_0_name)s",
        "ck": "ck_%(table_name)s_%(constraint_name)s",
        "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
        "pk": "pk_%(table_name)s",
    }
)

customers = Table(
    "customers",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("email", String(255), nullable=False, unique=True),
)

products = Table(
    "products",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("sku", String(32), nullable=False),
    Column("price", Numeric(10, 2), nullable=False),
    UniqueConstraint("sku"),
    CheckConstraint("price >= 0", name="price_non_negative"),
)

orders = Table(
    "orders",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("customer_id", Integer, ForeignKey("customers.id"), nullable=False),
)

for table in (customers, products, orders):
    print(CreateTable(table).compile())
```

```text
--  Pod maską (uproszczone, PostgreSQL):
CREATE TABLE customers (
    id SERIAL NOT NULL,
    email VARCHAR(255) NOT NULL,
    CONSTRAINT pk_customers PRIMARY KEY (id),
    CONSTRAINT uq_customers_email UNIQUE (email)
);

CREATE TABLE products (
    id SERIAL NOT NULL,
    sku VARCHAR(32) NOT NULL,
    price NUMERIC(10, 2) NOT NULL,
    CONSTRAINT pk_products PRIMARY KEY (id),
    CONSTRAINT uq_products_sku UNIQUE (sku),
    CONSTRAINT ck_products_price_non_negative CHECK (price >= 0)
);

CREATE TABLE orders (
    id SERIAL NOT NULL,
    customer_id INTEGER NOT NULL,
    CONSTRAINT pk_orders PRIMARY KEY (id),
    CONSTRAINT fk_orders_customer_id_customers FOREIGN KEY(customer_id)
        REFERENCES customers (id)
);
```

Zwróć uwagę: **wszystko ma nazwę, wszystko jest przewidywalne**. Migracja, która napisze `DROP CONSTRAINT ck_products_price_non_negative`, zadziała identycznie na każdym środowisku.

> ⚠️ **Pułapka — `naming_convention` ustawiaj od pierwszego dnia.** Jeśli dodasz go później, istniejące tabele w bazie mają już nadane nazwy „po staremu”. Konwencja dotyczy obiektów definiowanych w Pythonie; nie zmieni tego, co już siedzi w bazie. Możesz ustawić konwencję też per tabela (argument `naming_convention` w `Table(...)`), ale to droga do niespójności. Wybierz jedną konwencję dla całego projektu i trzymaj ją w jednym module (np. `db/meta.py`).

---

## Tworzenie schematu: `create_all`, `drop_all`, kolejność

### `metadata.create_all(engine)`

To najprostszy sposób, by przenieść projekt z kodu do bazy.

```python
# examples/03k_create_all.py
from sqlalchemy import create_engine

from examples.shop_schema import metadata  # pełny przykład niżej

engine = create_engine("sqlite:///shop.db", echo=True)

metadata.create_all(engine)

from sqlalchemy import inspect

inspector = inspect(engine)
print(inspector.get_table_names())
# ['customers', 'order_items', 'orders', 'products']
```

Co robi `create_all`:

1. bierze **wszystkie** tabele zarejestrowane w tym `MetaData`,
2. sortuje je **topologicznie** po zależnościach kluczy obcych (`metadata.sorted_tables`),
3. domyślnie sprawdza istnienie każdej tabeli (`checkfirst=True`) i tworzy tylko brakujące,
4. nie dotyka istniejących tabel — **nie doda brakującej kolumny, nie zmieni typu** (to robią migracje!).

```text
-- 🔬 Pod maską: kolejność tworzenia przy kluczach obcych
customers  →  products  →  orders  →  order_items
   ^                            ^          ^
   └── (brak zależności)        │          │
                                └──────────┴── odwołują się do customers/products/orders
```

> 🧠 **Dlaczego tak jest —** tabeli `orders` nie da się utworzyć przed `customers`, bo DDL zawiera `REFERENCES customers (id)` — baza odrzuci klucz obcy wskazujący nieistniejącą tabelę. Sortowanie topologiczne w `metadata.sorted_tables` rozwiązuje ten problem automatycznie. Gdy masz **cykliczne** zależności (tabela A wskazuje B, a B wskazuje A), sortowanie jest niemożliwe i musisz użyć `use_alter=True` na jednym z kluczy obcych — baza doda je osobnym `ALTER TABLE` już po utworzeniu tabel.

> 🔬 **Pod maską — co robi `checkfirst=True`.**
> Na PostgreSQL SQLAlchemy odpyta systemowy katalog:
> ```sql
> SELECT c.relname FROM pg_class c
> JOIN pg_namespace n ON n.oid = c.relnamespace
> WHERE n.nspname = %(schema_name)s::pg_catalog.text
>   AND c.relname = %(table_name)s::pg_catalog.text
>   AND pg_catalog.pg_table_is_visible(c.oid)
>   AND c.relkind = %(relkind)s::"char"
> ```
> Na SQLite — prostsze `PRAGMA` lub zapytanie do `sqlite_master`:
> ```sql
> PRAGMA main.table_info("customers")
> ```
> Gdy używasz `checkfirst=False`, tych zapytań nie ma — przydatne w testach (mniej rund do bazy) i w środowiskach, gdzie wiesz, że baza jest pusta.

### `engine.begin()` a ręczne DDL

`metadata.create_all(engine)` przyjmuje `engine`, ale pod spodem otwiera **transakcję** i commituje ją po zakończeniu. To ważne: jeśli tworzysz 10 tabel i siódma się nie uda, **cała operacja zostanie wycofana** (na bazach obsługujących transakcyjny DDL — PostgreSQL, SQLite tak; MySQL niestety nie, tam DDL commituje się sam).

Jeśli potrzebujesz większej kontroli, możesz podać `Connection`:

```python
# examples/03l_create_all_manual.py
from sqlalchemy import create_engine

from examples.shop_schema import metadata

engine = create_engine("sqlite:///shop.db")

with engine.begin() as conn:
    # tworzymy tylko dwie tabele, w jawnej kolejności
    metadata.tables["customers"].create(conn, checkfirst=True)
    metadata.tables["orders"].create(conn, checkfirst=True)
```

### `drop_all` — i dlaczego to niebezpieczna metoda

```python
metadata.drop_all(engine)   # USUWA WSZYSTKIE TABELE Z TEGO METADATA
```

> ⚠️ **Pułapka najwyższego ryzyka w tym module.** `metadata.drop_all(engine)` wykonuje `DROP TABLE` na wszystkich tabelach zdefiniowanych w `MetaData`. Jeśli `engine` wskazuje na bazę produkcyjną — dane są nie do odzyskania. Nie ma pytania „czy na pewno?”, nie ma kosza.
> Zabezpieczenia, które warto stosować:
> 1. Trzymaj `drop_all` wyłącznie w kodzie testów i skryptów deweloperskich (moduł 18),
> 2. Weryfikuj środowisko przed uruchomieniem:
> ```python
> import os
> assert os.environ.get("APP_ENV") in {"dev", "test"}, "drop_all tylko poza produkcją!"
> ```
> 3. Na produkcji **nigdy** nie używaj `drop_all`; do usuwania konkretnych tabel służy migracja (moduł 16), a do archiwizacji — procedura eksportu.

Kolejność `drop_all` jest odwrotnością `create_all` — SQLAlchemy usuwa najpierw tabele zależne (`order_items`), potem nadrzędne (`customers`), bo `DROP TABLE customers` przy istniejącym kluczu obcym zostałoby odrzucone (na PostgreSQL bez `CASCADE`).

### `metaData.tables` i praca z pojedynczą tabelą

```python
# examples/03m_tables_dict.py
from examples.shop_schema import metadata

print(sorted(metadata.tables.keys()))
# ['customers', 'order_items', 'orders', 'products']

orders = metadata.tables["orders"]
print(orders.c.keys())          # ['id', 'customer_id', 'status', 'placed_at']

# Sprawdzenie, czy tabela jest już zarejestrowana (bez wyjątku)
print(metadata.tables.get("missing"))   # None
```

> ⚠️ **Pułapka — import uboczny.** Definiowanie `Table(...)` na poziomie modułu to **efekt uboczny importu**: samo `import shop_schema` rejestruje tabele w `MetaData`. Jeśli utworzysz `metadata = MetaData()` i zdefiniujesz tabele w funkcji, to przy drugim wywołaniu funkcji dostaniesz `InvalidRequestError: Table 'customers' is already defined for this MetaData instance`.
> Rozwiązania: (a) definiuj schemat **raz**, na poziomie modułu (standard w projektach), (b) jeśli musisz tworzyć go dynamicznie, twórz nowy `MetaData()` w każdej iteracji, (c) używaj `extend_existing=True` tylko wtedy, gdy naprawdę chcesz nadpisać definicję (to lekarstwo na błąd, nie jego rozwiązanie).

---

## Refleksja: czytanie istniejącej bazy

Czasami nie jesteś autorem schematu. Baza istnieje od lat, jest utrzymywana przez inny zespół, a ty musisz się do niej podłączyć. SQLAlchemy potrafi **odczytać** strukturę z bazy i zbudować obiekty `Table` — to **refleksja** (*reflection*).

> 💡 **Analogia — inwentaryzacja magazynu.**
> Zamiast tworzyć magazyn od zera („chcę tu 10 regałów”), wchodzisz z notesem i **spisujesz, co już jest**: ile regałów, jak wysokie, co na których półkach. Refleksja to taka inwentaryzacja wykonywana przez program — zamiast pisać schemat ręcznie, każesz bazie go opowiedzieć.

### Dwa sposoby: `MetaData.reflect()` i `Table(..., autoload_with=)`

```python
# examples/03n_reflection.py
from sqlalchemy import MetaData, Table, create_engine

engine = create_engine("sqlite:///shop.db")

# Wariant 1: odczytaj wszystko do nowego MetaData
reflected_metadata = MetaData()
reflected_metadata.reflect(bind=engine)
print(sorted(reflected_metadata.tables.keys()))
# ['customers', 'order_items', 'orders', 'products']

# Wariant 2: odczytaj jedną tabelę
single_metadata = MetaData()
customers = Table("customers", single_metadata, autoload_with=engine)
print([c.name for c in customers.c])
# ['id', 'email', 'full_name', 'is_active', 'created_at']
print(customers.c.email.type)     # VARCHAR(255)
print(customers.primary_key.columns.keys())   # ['id']
```

> 🧠 **Dlaczego tak jest — refleksja to seria zapytań do katalogu systemowego.** SQLAlchemy przez `Inspector` odczytuje nazwy tabel, kolumny, typy, klucze główne, obce, indeksy i ograniczenia. Na PostgreSQL to zapytania do `information_schema`/`pg_catalog`; na SQLite do `PRAGMA table_info` i `sqlite_master`. Na dużej bazie `metadata.reflect()` bez filtra może być wolne — dlatego istnieje parametr `only=`.

```python
# examples/03o_reflect_selected.py
from sqlalchemy import MetaData, create_engine

engine = create_engine("sqlite:///shop.db")

metadata = MetaData()
metadata.reflect(
    bind=engine,
    only=["customers", "orders"],     # ogranicz zakres inwentaryzacji
    resolve_fks=False,                # nie dociągaj automatycznie tabel z FK
)
print(sorted(metadata.tables.keys()))   # ['customers', 'orders']
```

### `Inspector` — inspekcja bez budowania obiektów

Jeśli chcesz tylko **podejrzeć** strukturę (np. w skrypcie diagnostycznym), nie musisz tworzyć `Table`. `inspect()` zwraca obiekt `Inspector` z wygodnym API.

```python
# examples/03p_inspector.py
from sqlalchemy import create_engine, inspect

engine = create_engine("sqlite:///shop.db")
inspector = inspect(engine)

print(inspector.get_table_names())
# ['customers', 'order_items', 'orders', 'products']

for column in inspector.get_columns("customers"):
    print(f"{column['name']:12} {column['type']!s:18} nullable={column['nullable']}")
# id           INTEGER            nullable=False
# email        VARCHAR(255)       nullable=False
# full_name    VARCHAR(200)       nullable=False
# is_active    BOOLEAN            nullable=False
# created_at   DATETIME           nullable=False

print(inspector.get_pk_constraint("customers"))
# {'constrained_columns': ['id'], 'name': 'pk_customers'}

print(inspector.get_foreign_keys("orders"))
# [{'name': 'fk_orders_customer_id_customers',
#   'constrained_columns': ['customer_id'],
#   'referred_table': 'customers',
#   'referred_columns': ['id'],
#   'options': {}}]

print(inspector.get_indexes("orders"))
print(inspector.get_unique_constraints("customers"))
print(inspector.get_check_constraints("products"))
print(inspector.has_table("orders"))          # True
```

To ostatnie API (`get_check_constraints`) jest szczególnie przydatne przy weryfikacji, czy twoje ograniczenia naprawdę istnieją w bazie — i posłuży nam w ćwiczeniu 2.

> ⚠️ **Pułapka — refleksja gubi informacje, których baza nie pamięta.**
> - `Uuid` na SQLite zapisuje się jako `CHAR(32)`; refleksja zwróci `CHAR(32)`, a nie `Uuid` — stracisz informację, że w Pythonie chodziło o UUID.
> - `Enum` na PostgreSQL staje się natywnym typem; na SQLite — `VARCHAR` plus `CHECK`; refleksja zwróci różne rzeczy w obu przypadkach.
> - `default=` (Pythonowy) **nie istnieje w bazie**, więc refleksja go nie odtworzy. Zobaczysz tylko `server_default=`, jeśli był ustawiony.
> - Relacje ORM (`relationship`) nie są częścią schematu bazy — a więc refleksja ich nie odtworzy. Będziesz musiał dopisać je ręcznie (moduł 09).
> Wniosek: refleksja jest świetnym narzędziem diagnostycznym i świetnym punktem startu, ale **nie jest pełnym odtworzeniem modelu aplikacji**.

---

## DDL w kodzie czy migracje?

W tym miejscu trzeba powiedzieć coś, co bywa pomijane w tutorialach: **`create_all` nie jest narzędziem produkcyjnym do zarządzania schematem**. Jest doskonałym narzędziem do:

- szybkiego prototypu,
- środowiska deweloperskiego od zera,
- testów automatycznych (`create_all` na bazie w pamięci — moduł 18).

Nie nadaje się natomiast do **zmieniania** istniejącego schematu, bo… nic takiego nie robi. Zmiana `String(50)` na `String(100)` w kodzie i uruchomienie `create_all` **nie zmieni bazy** — SQLAlchemy zapyta „czy tabela istnieje?”, usłyszy „tak” i pójdzie dalej. Twoja baza nadal ma `VARCHAR(50)`, a kod myśli, że ma `VARCHAR(100)`.

> 💡 **Analogia — `create_all` to postawienie domu od zera; migracje to remonty.**
> Jeśli dom już stoi, a ty chcesz dobudować balkon, nie wyburzasz wszystkiego, żeby postawić dom jeszcze raz. Wchodzisz ekipą remontową, wycinasz otwór, dokładasz płytę, wzmacniasz ścianę — **z lokatorami w środku**. Tym właśnie są migracje: precyzyjnymi, wersjonowanymi zmianami na żywym obiekcie.
> `create_all` to plan „postaw dom”. Migracja to instrukcja „zmień ten konkretny fragment i oto jak to cofnąć”.

**Kiedy co stosować — tabela decyzyjna:**

| Sytuacja | Narzędzie |
|---|---|
| Globalny hackathon / prototyp | `create_all` |
| Środowisko deweloperskie resetowane codziennie | `create_all` + `drop_all` |
| Testy jednostkowe (SQLite w pamięci) | `create_all` |
| Naukowe analizy, jednorazowe skrypty | `create_all` |
| Pierwsze wdrożenie na produkcję | migracja bazowa (`alembic revision --autogenerate` na pustej bazie) |
| Każda zmiana na środowisku, które ma dane | **wyłącznie migracja** |
| Praca w zespole (każdy ma własną bazę) | **wyłącznie migracje** |
| Refaktoring schematu (rozdzielenie kolumny, zmiana typu) | migracja + migracja danych |

> 🧠 **Dlaczego nie da się „po prostu” zmieniać DDL-em z kodu —** baza z danymi to nie plik z kodem. `ALTER TABLE ... ADD COLUMN ... NOT NULL` bez wartości domyślnej jest **niewykonalne**, jeśli tabela ma jakiekolwiek wiersze (baza nie wie, co wpisać w istniejące wiersze). Podobnie zmiana typu kolumny może wymagać przepisania miliona wierszy. Migracje istnieją po to, by te operacje **opisowo i wersjonowalnie** przeprowadzić w bezpiecznej kolejności: dodaj kolumnę → wypełnij dane → ustaw `NOT NULL` → dopiero potem zmień kod aplikacji. Cały ten proces pozna…

> 🆕 **SQLAlchemy 2.1 —** w 2.1 pojawiły się nowe konstrukcje DDL-owe, m.in. `CreateView` oraz `CREATE TABLE AS SELECT` (`CreateTableAs`), pozwalające definiować widoki i tabele pochodne jako obiekty Pythona w `MetaData`. W 2.0 widoki realizuje się przez surowe `text()` lub `DDL()` — jeśli potrzebujesz ich w projekcie na 2.0, pamiętaj o tym przy planowaniu migracji.

---

## Pełny przykład: schemat sklepu

Czas złożyć wszystko w całość. Poniższy przykład jest kompletny i uruchamialny: definiuje schemat sklepu (`customers`, `products`, `orders`, `order_items`), tworzy go w pliku SQLite, wypisuje wygenerowany DDL i odczytuje schemat z powrotem przez refleksję.

**Jak uruchomić:**

```bash
pip install "SQLAlchemy>=2.0"
python examples/03_shop_schema.py
```

```python
# examples/03_shop_schema.py
"""Kompletny schemat sklepu zdefiniowany w SQLAlchemy Core."""

from __future__ import annotations

from sqlalchemy import (
    JSON,
    Boolean,
    CheckConstraint,
    Column,
    DateTime,
    Enum,
    ForeignKey,
    Index,
    Integer,
    MetaData,
    Numeric,
    String,
    Table,
    Text,
    UniqueConstraint,
    create_engine,
    func,
    inspect,
    text,
)
from sqlalchemy.schema import CreateTable

# ---------------------------------------------------------------------------
# 1. Konwencja nazewnicza — jedno miejsce decydujące o nazwach ograniczeń.
# ---------------------------------------------------------------------------
NAMING_CONVENTION: dict[str, str] = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}

metadata = MetaData(naming_convention=NAMING_CONVENTION)

# ---------------------------------------------------------------------------
# 2. Tabele. Kolejność definicji nie ma znaczenia — i tak sortujemy topologicznie.
# ---------------------------------------------------------------------------

customers = Table(
    "customers",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("email", String(255), nullable=False),
    Column("full_name", String(200), nullable=False),
    Column("is_active", Boolean, nullable=False, server_default=text("true")),
    Column("created_at", DateTime, nullable=False, server_default=func.now()),
    UniqueConstraint("email", name="email"),
    CheckConstraint("email LIKE '%@%'", name="email_shape"),
)

products = Table(
    "products",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("sku", String(32), nullable=False),
    Column("name", String(200), nullable=False),
    Column("description", Text, nullable=True),
    Column("price", Numeric(10, 2), nullable=False),
    Column("attributes", JSON, nullable=True),
    Column("is_active", Boolean, nullable=False, server_default=text("true")),
    Column("created_at", DateTime, nullable=False, server_default=func.now()),
    UniqueConstraint("sku", name="sku"),
    CheckConstraint("price >= 0", name="price_non_negative"),
    Index("ix_products_name", "name"),
)

orders = Table(
    "orders",
    metadata,
    Column("id", Integer, primary_key=True),
    Column(
        "customer_id",
        Integer,
        ForeignKey("customers.id", ondelete="CASCADE"),
        nullable=False,
        index=True,
    ),
    Column(
        "status",
        Enum("pending", "paid", "shipped", "cancelled", name="order_status"),
        nullable=False,
        server_default="pending",
    ),
    Column("note", Text, nullable=True),
    Column("placed_at", DateTime, nullable=False, server_default=func.now()),
    Index("ix_orders_customer_id_placed_at", "customer_id", "placed_at"),
)

order_items = Table(
    "order_items",
    metadata,
    Column("id", Integer, primary_key=True),
    Column(
        "order_id",
        Integer,
        ForeignKey("orders.id", ondelete="CASCADE"),
        nullable=False,
    ),
    Column(
        "product_id",
        Integer,
        ForeignKey("products.id", ondelete="RESTRICT"),
        nullable=False,
        index=True,
    ),
    Column("quantity", Integer, nullable=False),
    Column("unit_price", Numeric(10, 2), nullable=False),
    UniqueConstraint("order_id", "product_id", name="order_product"),
    CheckConstraint("quantity > 0", name="quantity_positive"),
    CheckConstraint("unit_price >= 0", name="unit_price_non_negative"),
)


# ---------------------------------------------------------------------------
# 3. Prezentacja DDL-u dla każdego dialektu.
# ---------------------------------------------------------------------------
def print_all_ddl() -> None:
    """Wypisuje DDL dla dialektu PostgreSQL (bez łączenia się z bazą)."""
    from sqlalchemy.dialects import postgresql

    print("=" * 72)
    print("DDL dla PostgreSQL (wygenerowany lokalnie, bez połączenia)")
    print("=" * 72)

    for table in metadata.sorted_tables:
        print(f"\n-- tabela: {table.name} --")
        print(str(CreateTable(table).compile(dialect=postgresql.dialect())))


def print_dependency_order() -> None:
    """Pokazuje, w jakiej kolejności SQLAlchemy utworzy tabele."""
    print("\nKolejność tworzenia (sortowanie topologiczne):")
    for position, table in enumerate(metadata.sorted_tables, start=1):
        print(f"  {position}. {table.name}")


# ---------------------------------------------------------------------------
# 4. Utworzenie schematu i refleksja.
# ---------------------------------------------------------------------------
def create_schema(database_url: str = "sqlite:///shop.db") -> None:
    engine = create_engine(database_url, echo=False)
    metadata.create_all(engine)
    engine.dispose()


def inspect_schema(database_url: str = "sqlite:///shop.db") -> None:
    """Odczytuje schemat z bazy i wypisuje jego najważniejsze elementy."""
    engine = create_engine(database_url)
    inspector = inspect(engine)

    print("\n" + "=" * 72)
    print("Refleksja: co naprawdę jest w bazie")
    print("=" * 72)

    for table_name in sorted(inspector.get_table_names()):
        print(f"\n[{table_name}]")

        pk = inspector.get_pk_constraint(table_name)
        print(f"  PK: {pk['constrained_columns']} (nazwa: {pk['name']})")

        for column in inspector.get_columns(table_name):
            nullable = "NULL" if column["nullable"] else "NOT NULL"
            print(f"  kolumna {column['name']:12} {str(column['type']):18} {nullable}")

        for fk in inspector.get_foreign_keys(table_name):
            print(
                f"  FK: {fk['constrained_columns']} -> "
                f"{fk['referred_table']}.{fk['referred_columns']} ({fk['name']})"
            )

        for unique in inspector.get_unique_constraints(table_name):
            print(f"  UNIQUE: {unique['column_names']} ({unique['name']})")

        for check in inspector.get_check_constraints(table_name):
            print(f"  CHECK: {check['sqltext']} ({check.get('name')})")

        for index in inspector.get_indexes(table_name):
            print(f"  INDEX: {index['column_names']} ({index['name']})")

    engine.dispose()


if __name__ == "__main__":
    print_all_ddl()
    print_dependency_order()

    create_schema()
    inspect_schema()
```

Fragment rzeczywistego wyjścia (SQLite, po uruchomieniu):

```text
Kolejność tworzenia (sortowanie topologiczne):
  1. customers
  2. products
  3. orders
  4. order_items

[order_items]
  PK: ['id'] (nazwa: pk_order_items)
  kolumna id           INTEGER            NOT NULL
  kolumna order_id     INTEGER            NOT NULL
  kolumna product_id   INTEGER            NOT NULL
  kolumna quantity     INTEGER            NOT NULL
  kolumna unit_price   NUMERIC(10, 2)     NOT NULL
  FK: ['order_id'] -> orders.id (fk_order_items_order_id_orders)
  FK: ['product_id'] -> products.id (fk_order_items_product_id_products)
  UNIQUE: ['order_id', 'product_id'] (uq_order_items_order_id)
  INDEX: ['product_id'] (ix_order_items_product_id)
```

> 🧪 **Ćwiczenie w miejscu —** zmień w powyższym kodzie `Column("email", String(255))` na `Column("email", String(32))`, usuń plik `shop.db` i uruchom skrypt ponownie. Potem spróbuj uruchomić go **bez** usuwania pliku. Zobacz, że `create_all` wypisuje „istnieje, pomijam” — mimo że w kodzie jest inna długość. To dokładnie ta różnica między `create_all` i migracjami, o której mówiliśmy wyżej. Wróć do `String(255)` przed kontynuowaniem.

---

## Podsumowanie

1. **Schemat bazy to projekt domu, a nie sam dom.** Trzymany w jednym miejscu w kodzie daje przewidywalność, wersjonowanie i możliwość wygenerowania DDL-u dla różnych baz.
2. **`MetaData` to segregator na definicje tabel.** Jedna baza = jedno `MetaData`. Rejestruje tabele, pilnuje unikalności nazw i zna kolejność zależności.
3. **`Table` opisuje tabelę, `Column` opisuje kolumnę.** Trzeci argument `Column` to typ, dalsze — flagi i ograniczenia.
4. **Typ kolumny to kontrakt.** Pieniądze → `Numeric` (nigdy `Float`), moment w czasie → `DateTime(timezone=True)`, długie teksty → `Text`, techniczne identyfikatory → `Integer` lub `Uuid`.
5. **`nullable` domyślnie wynosi `True`** — pisz je jawnie przy każdej kolumnie, żeby nie dopuszczać `NULL`-i tam, gdzie nie powinny wystąpić.
6. **`default=` liczy Python, `server_default=` liczy baza.** Tylko to drugie istnieje w DDL i działa dla innych klientów bazy oraz przy migracjach na istniejących danych. Do czasu używaj `func.now()`, nie `text("now()")`.
7. **`naming_convention` ustaw od pierwszego dnia.** Bez tego migracje na produkcji będą się rozjeżdżać z migracjami lokalnymi.
8. **`CHECK`-om zawsze nadawaj nazwę.** Bez nazwy nie da się ich usunąć ani zmienić w migracji.
9. **`create_all` tworzy brakujące tabele i nic więcej.** Nie dodaje kolumn, nie zmienia typów, nie przenosi danych. To narzędzie prototypu i testów, nie narzędzie wdrożeń.
10. **Refleksja to inwentaryzacja, nie odtworzenie modelu.** Odtworzy strukturę, ale zgubi typy natywne dla dialektu, Pythonowe `default` i wszystkie relacje ORM.

---

## wiczenia

### wiczenie 1 — hierarchia kategorii (poziom: podstawowy+)

Do schematu z pełnego przykładu dodaj tabelę `categories` z **samo-referencyjnym** kluczem obcym, tak aby kategorie mogły tworzyć drzewo (kategoria nadrzędna → podkategoria).

Wymagania:

1. kolumny: `id` (PK), `name` (`String(120)`, `NOT NULL`), `slug` (`String(120)`, `NOT NULL`, unikalny), `parent_id` (`Integer`, `NULL`, FK do tej samej tabeli),
2. klucz obcy z `ondelete="SET NULL"` (usunięcie rodzica nie usuwa dzieci, tylko „przypina” je do korzenia),
3. indeks na `parent_id`,
4. ograniczenie `CHECK` o nazwie `slug_shape`, wymagające, by `slug` nie zawierał spacji,
5. tabela `products` powinna zyskać kolumnę `category_id` (`Integer`, `NULL`, FK do `categories.id`, `ondelete="SET NULL"`).

Następnie: utwórz schemat w nowym pliku SQLite i wypisz `inspector.get_foreign_keys("categories")`.

### Ćwiczenie 2 — schemat w kodzie kontra schemat w bazie (poziom: średni)

Napisz funkcję, która **porównuje** schemat zdefiniowany w kodzie ze schematem rzeczywiście obecnym w bazie i zwraca listę rozbieżności.

Sygnatura:

```python
def compare_schemas(metadata: MetaData, engine: Engine) -> list[str]:
    """Zwraca listę opisów różnic między kodem a bazą (pusta lista = zgodność)."""
```

Funkcja powinna wykrywać:

1. tabele obecne w `metadata`, ale nieobecne w bazie (i odwrotnie),
2. kolumny obecne w kodzie, ale nieobecne w bazie (i odwrotnie),
3. różnice typu kolumny (porównanie po `str(type)`),
4. różnice w `nullable`.

Uwaga: porównanie typów jest trudne w ogóle, a na SQLite szczególnie (SQLite nie ma typów, tylko „powinowactwo”). Zaimplementuj porównanie **tekstowe i tolerancyjne**: znormalizuj oba typy (`str(type).lower()`), usuń długości w nawiasach i porównaj przedrostki. Skomentuj w kodzie, dlaczego dokładne porównanie typów jest w praktyce niemożliwe.

### Ćwiczenie 3 — migracja ręczna (poziom: średni+)

Mając bazę utworzoną przez `create_schema()` z pełnego przykładu **z danymi w środku** (wstaw kilka wierszy przez `engine.begin()` i `text()`), wykonaj ręcznie — bez Alembica — bezpieczną zmianę: dodaj do tabeli `customers` kolumnę `loyalty_points INTEGER NOT NULL DEFAULT 0`.

Napisz skrypt, który:

1. sprawdza, czy kolumna już istnieje (`inspector.get_columns`), i jeśli tak — kończy pracę bez błędu (idempotencja!),
2. wykonuje `ALTER TABLE customers ADD COLUMN loyalty_points INTEGER NOT NULL DEFAULT 0`,
3. dodaje kolumnę do definicji `Table` w kodzie (żeby kod i baza były zgodne),
4. na końcu używa funkcji z ćwiczenia 2, by potwierdzić zgodność.

W komentarzu wyjaśnij, dlaczego `DEFAULT 0` jest tu niezbędne.

---

### Rozwiązania

<details>
<summary><strong>Rozwiązanie ćwiczenia 1 — hierarchia kategorii</strong></summary>

```python
# examples/03_shop_hierarchy.py (fragment — dopisany do schematu z pełnego przykładu)
from sqlalchemy import CheckConstraint, Column, ForeignKey, Index, Integer, String, Table

categories = Table(
    "categories",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String(120), nullable=False),
    Column("slug", String(120), nullable=False),
    Column(
        "parent_id",
        Integer,
        ForeignKey("categories.id", ondelete="SET NULL"),
        nullable=True,
    ),
    UniqueConstraint("slug", name="slug"),
    CheckConstraint("slug NOT LIKE '% %'", name="slug_shape"),
    Index("ix_categories_parent_id", "parent_id"),
)

# Dopisujemy kolumnę do istniejącej tabeli products (w definicji, nie w bazie!)
products.append_column(
    Column(
        "category_id",
        Integer,
        ForeignKey("categories.id", ondelete="SET NULL"),
        nullable=True,
        index=True,
    )
)
```

Weryfikacja:

```python
from sqlalchemy import create_engine, inspect
from examples.shop_hierarchy import metadata

engine = create_engine("sqlite:///shop_with_categories.db")
metadata.create_all(engine)

inspector = inspect(engine)
print(inspector.get_foreign_keys("categories"))
# [{'name': 'fk_categories_parent_id_categories',
#   'constrained_columns': ['parent_id'],
#   'referred_table': 'categories',
#   'referred_columns': ['id'],
#   'options': {'ondelete': 'SET NULL'}}]
```

**Dlaczego tak, a nie inaczej:**

- `ondelete="SET NULL"` a nie `"CASCADE"` — usunięcie kategorii nadrzędnej nie powinno kasować podkategorii razem z produktami; bezpieczniej „odciąć” gałąź od korzenia.
- `parent_id` musi być `nullable=True`, bo kategoria najwyższego poziomu nie ma rodzica. Gdybyś ustawił `NOT NULL`,nie dałoby się stworzyć korzenia drzewa.
- `ForeignKey("categories.id")` — klucz obcy do **własnej** tabeli jest w pełni poprawny; SQLAlchemy obsługuje to bez problemu, a `metadata.sorted_tables` umieści `categories` i tak przed `products` (bo `products` teraz na niej zależy).

> ⚠️ **Uwaga o `append_column` —** metoda ta **dodaje kolumnę do obiektu `Table` w Pythonie** i jest wygodna w kodzie, ale **nie zmienia istniejącej tabeli w bazie**. Jeśli baza już istnieje, `create_all` pominie tabelę `products` i nowej kolumny nie będzie. To kolejny dowód na potrzebę migracji (ćwiczenie 3 i moduł 16).

> 🧠 **Alternatywa —** gdybyś chciał, by hierarchia była opcjonalna na poziomie relacji (produkt może nie mieć kategorii), `nullable=True` w `category_id` jest właściwe. Gdybyś natomiast wymagał, aby **każdy** produkt miał kategorię, dałbyś `nullable=False` — ale wtedy nie mógłbyś już wstawić produktu bez kategorii, co przy imporcie danych z CSV bywa uciążliwe.

</details>

<details>
<summary><strong>Rozwiązanie ćwiczenia 2 — porównanie schematów</strong></summary>

```python
# examples/03_schema_diff.py
"""Narzędzie diagnostyczne: porównanie schematu w kodzie z bazą."""

from __future__ import annotations

import re

from sqlalchemy import Engine, MetaData, inspect

_LENGTH_RE = re.compile(r"\(\s*\d+(\s*,\s*\d+)?\s*\)")


def _normalize_type(type_name: str) -> str:
    """Sprowadza nazwę typu do porównywalnej, tolerancyjnej postaci.

    SQLite przechowuje tylko 'powinowactwo' typów, a różne dialekty
    renderują różne nazwy dla tego samego pojęcia (INTEGER vs INT vs SERIAL).
    Dlatego: małe litery, bez długości, bez parametrów precyzji.
    """
    normalized = _LENGTH_RE.sub("", type_name.lower()).strip()
    aliases = {
        "int": "integer",
        "varchar": "string",
        "character varying": "string",
        "datetime": "timestamp",
        "bool": "boolean",
    }
    return aliases.get(normalized, normalized)


def compare_schemas(metadata: MetaData, engine: Engine) -> list[str]:
    """Zwraca listę rozbieżności między schematem w kodzie a rzeczywistą bazą."""
    differences: list[str] = []
    inspector = inspect(engine)

    code_tables = {name for name in metadata.tables if "." not in name}
    db_tables = set(inspector.get_table_names())

    for missing in sorted(code_tables - db_tables):
        differences.append(f"TABELA '{missing}': jest w kodzie, brak w bazie")

    for extra in sorted(db_tables - code_tables):
        differences.append(f"TABELA '{extra}': jest w bazie, brak w kodzie")

    for table_name in sorted(code_tables & db_tables):
        code_table = metadata.tables[table_name]

        code_columns = {col.name: col for col in code_table.columns}
        db_columns = {col["name"]: col for col in inspector.get_columns(table_name)}

        for missing in sorted(set(code_columns) - set(db_columns)):
            differences.append(
                f"KOLUMNA '{table_name}.{missing}': jest w kodzie, brak w bazie"
            )

        for extra in sorted(set(db_columns) - set(code_columns)):
            differences.append(
                f"KOLUMNA '{table_name}.{extra}': jest w bazie, brak w kodzie"
            )

        for shared in sorted(set(code_columns) & set(db_columns)):
            code_col = code_columns[shared]
            db_col = db_columns[shared]

            code_type = _normalize_type(str(code_col.type))
            db_type = _normalize_type(str(db_col["type"]))
            if code_type != db_type:
                differences.append(
                    f"TYP '{table_name}.{shared}': kod={code_type!r}, baza={db_type!r}"
                )

            # nullable w kodzie: None oznacza "nie ustawiono" -> domyślnie True
            code_nullable = True if code_col.nullable is None else bool(code_col.nullable)
            if code_nullable != bool(db_col["nullable"]):
                differences.append(
                    f"NULLABLE '{table_name}.{shared}': "
                    f"kod={code_nullable}, baza={db_col['nullable']}"
                )

    return differences


if __name__ == "__main__":
    from sqlalchemy import create_engine

    from examples.shop_schema import metadata

    engine = create_engine("sqlite:///shop.db")
    for line in compare_schemas(metadata, engine) or ["Schemat zgodny."]:
        print(line)
```

**Dlaczego tak, a nie inaczej:**

- Porównanie typów jest **tolerancyjne i tekstowe**, bo dokładne porównanie jest w praktyce niewykonalne: SQLite nie ma typów w sensie SQL-owym (tylko „powinowactwo”: `INTEGER` i `BIGINT` mają to samo powinowactwo `INTEGER`), PostgreSQL renderuje `SERIAL` tam, gdzie w kodzie było `Integer`, a `Uuid` na SQLite staje się `CHAR(32)`.
- `nullable` w SQLAlchemy Core to `bool` (domyślnie `True`), ale przy odczycie z niektórych konstrukcji może być `None` — dlatego normalizujemy do `bool`.
- Funkcja **nie sprawdza ograniczeń** (`UNIQUE`, `CHECK`, `FK`). Można je dodać analogicznie (`inspector.get_check_constraints`, `get_unique_constraints`), ale porównywanie ich tekstowych reprezentacji jest jeszcze bardziej podatne na fałszywe alarmy (różne bazy renderują `LIKE` i wyrażenia w różny sposób).
- Tolerancyjne porównanie to kompromis: łapie realne rozjazdy (brakująca kolumna, brakująca tabela), a nie zgłasza fałszywych alarmów przy różnicach dialektowych.

> ⚠️ **Minipułapka —** jeśli użyjesz tego narzędzia jako bramki w CI, **nie** traktuj różnicy typu jako twardego błędu bez weryfikacji. Zdarzają się fałszywe alarmy (np. `NUMERIC(10, 2)` vs `NUMERIC` po refleksji w SQLite). Lepiej raportować ostrzeżenie niż blokować wdrożenie.

</details>

<details>
<summary><strong>Rozwiązanie ćwiczenia 3 — ręczna migracja</strong></summary>

```python
# examples/03_manual_migration.py
"""Ręczne, idempotentne dodanie kolumny do istniejącej tabeli z danymi."""

from __future__ import annotations

from sqlalchemy import Engine, create_engine, inspect, text


def add_loyalty_points(engine: Engine) -> None:
    """Dodaje customers.loyalty_points, jeśli jeszcze nie istnieje."""
    table_name = "customers"
    column_name = "loyalty_points"

    inspector = inspect(engine)
    existing_columns = {col["name"] for col in inspector.get_columns(table_name)}

    if column_name in existing_columns:
        print(f"Kolumna {table_name}.{column_name} już istnieje — nic nie robię.")
        return

    if engine.dialect.name == "sqlite":
        # SQLite nie potrafi dodać NOT NULL bez wartości domyślnej.
        # Dzięki DEFAULT 0 istniejące wiersze od razu otrzymają sensowną wartość.
        statement = text(
            f"ALTER TABLE {table_name} "
            f"ADD COLUMN {column_name} INTEGER NOT NULL DEFAULT 0"
        )
    else:
        # PostgreSQL: trzy kroki, bo istniejące wiersze nie mają jeszcze wartości.
        statement = text(
            f"ALTER TABLE {table_name} "
            f"ADD COLUMN {column_name} INTEGER NOT NULL DEFAULT 0"
        )

    with engine.begin() as conn:
        conn.execute(statement)

    print(f"Dodano kolumnę {table_name}.{column_name}.")
    engine.dispose()


if __name__ == "__main__":
    from examples.shop_schema import compare_schemas, metadata  # noqa: F401 (patrz ćwiczenie 2)

    engine = create_engine("sqlite:///shop.db")

    with engine.begin() as conn:
        conn.execute(
            text(
                "INSERT INTO customers (email, full_name) "
                "VALUES ('anna@example.com', 'Anna Kowalska')"
            )
        )

    add_loyalty_points(engine)

    engine = create_engine("sqlite:///shop.db")
    for line in compare_schemas(metadata, engine) or ["Schemat zgodny."]:
        print(line)
```

**Dlaczego tak, a nie inaczej:**

- **`DEFAULT 0` jest niezbędne.** Bez niego `ALTER TABLE ... ADD COLUMN ... NOT NULL` na tabeli z jakimkolwiek wierszem kończy się błędem: baza nie wie, jaką wartość wpisać w istniejące wiersze. Wartość domyślna rozwiązuje to w jednym kroku. (Alternatywa dla dużych tabel: dodać kolumnę jako `NULL`, wypełnić ją w osobnej transakcji partiami, a dopiero potem ustawić `NOT NULL` — to wzorzec z modułu 16.)
- **Sprawdzenie `inspector.get_columns` na początku** czyni operację **idempotentną** — uruchomienie jej dwa razy nie wywala się z błędem „duplicate column”. To absolutnie fundamentalna cecha każdej migracji: musi móc być uruchomiona ponownie po awarii.
- **Zgodność kodu i bazy.** Po wykonaniu `ALTER TABLE` musisz dodać kolumnę **także w definicji `Table`** (np. `customers.append_column(Column("loyalty_points", Integer, nullable=False, server_default=text("0")))`) — inaczej kod będzie myślał, że kolumny nie ma, a baza będzie ją miała. To dokładnie problem, który rozwiązują migracje Alembica: łączą one zmianę w bazie ze zmianą w kodzie.
- **`engine.dialect.name`** pozwala rozgałęzić kod na dialekty, gdy DDL naprawdę się różni. W tym wypadku składnia `ADD COLUMN` jest wspólna dla SQLite i PostgreSQL, ale w innych przypadkach (np. `ALTER COLUMN TYPE`) już nie jest.

> ⚠️ **Minipułapka —** ręczne migracje są świetnym ćwiczeniem, ale w projekcie **nie zastępują Alembica**. Brakuje w nich: historii (kto, kiedy, po co), możliwości cofnięcia (`downgrade`), wykrywania rozjazdów w zespole. Ten skrypt ma sens w wyjątkowych sytuacjach (gorąca poprawka, brak narzędzia migracyjnego), a nie jako codzienna praktyka.

</details>

---

## Najczęstsze błędy i jak je czytać

| Komunikat / objaw | Przyczyna | Naprawa |
|---|---|---|
| `InvalidRequestError: Table 'x' is already defined for this MetaData instance` | Dwa `Table("x", metadata, ...)` w tym samym `MetaData` (np. funkcja tworząca schema wywołana dwa razy) | Definiuj schemat raz, na poziomie modułu; albo użyj świeżego `MetaData()` przy każdej iteracji; `extend_existing=True` tylko świadomie |
| `sqlalchemy.exc.NoReferencedTableError: Foreign key associated with column 'orders.customer_id' could not find table 'customers'` | Tabela `customers` nie jest zarejestrowana w tym samym `MetaData`, brakuje jej definicji (np. plik nie został zaimportowany) | Upewnij się, że moduły z definicjami wszystkich tabel są zaimportowane; przy rozbitym projekcie twórz pakiet z jawnym `import` lub `__init__.py` |
| `sqlalchemy.exc.CompileError: (in table 'x', column 'y'): VARCHAR requires a length on dialect sqlite` / analogiczny błąd przy `String` | Użyto `String` bez długości (`String()` zamiast `String(255)` lub `Text`) | Podaj długość albo użyj `Text` |
| `sqlite3.OperationalError: Cannot add a NOT NULL column with default value NULL` | `ALTER TABLE ADD COLUMN ... NOT NULL` bez `DEFAULT` i z istniejącymi wierszami | Dodaj `DEFAULT` albo dodaj kolumnę jako `NULL`, wypełnij, potem zacieśnij |
| `sqlite3.OperationalError: near "now": syntax error` | `server_default=text("now()")` na SQLite | Użyj `server_default=func.now()` (dialekt sam wybierze właściwą funkcję) |
| `psycopg.errors.DatatypeMismatch: column "is_active" is of type boolean but default expression is of type integer` | `server_default=text("1")` dla kolumny `Boolean` w PostgreSQL | Użyj `server_default=text("true")` (albo `text("false")`) |
| `sqlalchemy.exc.ArgumentError: Textual SQL expression '...' should be explicitly declared as text('...')` | W `server_default`/`where` przekazano zwykły napis tam, gdzie SQLAlchemy wymaga `text()` | Owinąć w `text("...")` lub użyć `func.` |
| Tabela utworzona, ale **bez** klucza obcego / ograniczenia | Ograniczenie dodano po `create_all` albo `Table` zdefiniowano w innym `MetaData`, albo kolumna miała już indeks o tej samej nazwie | Usuń i utwórz bazę na nowo (dev) albo napisz migrację (prod); sprawdź `inspector.get_foreign_keys()` |
| `sqlalchemy.exc.IntegrityError: NOT NULL constraint failed: customers.email` | Wstawiasz wiersz bez wartości dla kolumny `nullable=False` i bez `default`/`server_default` | Podaj wartość, dodaj `default=`/`server_default=` albo zmień `nullable` jeśli pole naprawdę opcjonalne |
| `sqlite3.IntegrityError: FOREIGN KEY constraint failed` — ale tylko wtedy, gdy klucze obce **są** włączone | Próba wstawienia/zmiany z kluczem obcym do nieistniejącego wiersza; na SQLite FK nie są domyślnie egzekwowane, więc ten błąd pojawia się dopiero po włączeniu `PRAGMA foreign_keys=ON` | Wstaw najpierw wiersz nadrzędny lub sprawdź istnienie FK; w testach pamiętaj, że SQLite domyślnie milczy o naruszeniach FK |
| `ProgrammingError: relation "customers" does not exist` przy `INSERT`/`SELECT` | Nie uruchomiono `create_all` / nie wykonano migracji na tym środowisku | Wykonaj `create_all` (dev) lub `alembic upgrade head` (środowiska z historią) |
| `ALTER TABLE` działa u ciebie, ale migracja wywala się na produkcji z „constraint ... does not exist” | Brak `naming_convention` — nazwy ograniczeń nadały się automatycznie i są różne w różnych środowiskach | Wprowadź `naming_convention` możliwie wcześnie; na produkcji odtwórz nazwy ręcznie w migracji |
| Po refleksji typ `Uuid` zamienił się w `CHAR(32)` | Refleksja zwraca typ fizyczny bazy, nie intencję Pythona | Traktuj refleksję jako punkt startu; popraw typy jawnie po odczycie |

---

## Słowniczek modułu

| Termin (EN) | Polski odpowiednik | Wyjaśnienie |
|---|---|---|
| `DDL` (Data Definition Language) | język definiowania danych | Podjęzyk SQL opisujący strukturę: `CREATE TABLE`, `ALTER TABLE`, `DROP TABLE`, `CREATE INDEX` |
| `DML` (Data Manipulation Language) | język manipulacji danymi | Podjęzyk SQL operujący na wierszach: `SELECT`, `INSERT`, `UPDATE`, `DELETE` |
| `schema` | schemat | Opis struktury bazy (tabele, kolumny, typy, ograniczenia). W PostgreSQL dodatkowo „przestrzeń nazw” |
| `MetaData` | metadane / segregator definicji | Kolekcja rejestrująca tabele, ich nazwy, konwencję nazewniczą i zależności |
| `Table` | tabela (obiekt opisowy) | Pythonowy opis tabeli istniejącej (lub dopiero planowanej) w bazie |
| `Column` | kolumna (obiekt opisowy) | Pythonowy opis kolumny wraz z typem, flagami i ograniczeniami |
| `constraint` | ograniczenie | Reguła egzekwowana przez bazę: `NOT NULL`, `UNIQUE`, `PRIMARY KEY`, `FOREIGN KEY`, `CHECK` |
| `primary key` | klucz główny | Kolumna (lub zestaw kolumn) jednoznacznie identyfikująca wiersz; implikuje `NOT NULL` i unikalność |
| `foreign key` | klucz obcy | Kolumna wskazująca wiersz w innej tabeli; baza pilnuje jego istnienia |
| `surrogate key` | klucz sztuczny | Dodatkowy identyfikator bez znaczenia biznesowego (np. `id`) |
| `natural key` | klucz naturalny | Identyfikator istniejący w domenie (np. e-mail, SKU) |
| `nullable` | dopuszczalność pustej wartości | `nullable=False` → `NOT NULL`, `nullable=True` (domyślnie) → kolumna przyjmuje `NULL` |
| `NULL` | brak wartości | Nie jest zerem ani pustym napisem — to informacja „nie wiem / nie dotyczy” |
| `default` | wartość domyślna w Pythonie | Uzupełniana przez SQLAlchemy przy `INSERT`; nie istnieje w DDL |
| `server_default` | wartość domyślna w bazie | Klauzula `DEFAULT` w DDL; działa dla wszystkich klientów bazy |
| `onupdate` | aktualizacja przy zmianie | Wartość dopisywana przez SQLAlchemy przy `UPDATE`; nie generuje `ON UPDATE` w DDL |
| `index` | indeks | Dodatkowa struktura przyspieszająca wyszukiwanie; kosztem rozmiaru i wolniejszych zapisów |
| `reflection` | refleksja | Odczyt istniejącej struktury bazy do obiektów SQLAlchemy (`Table(..., autoload_with=engine)`) |
| `Inspector` | inspektor | Obiekt (`inspect(engine)`) z metodami `get_columns`, `get_foreign_keys`, `get_indexes` itd. |
| `naming_convention` | konwencja nazewnicza | Szablon nazw ograniczeń i indeksów ustawiany na `MetaData` |
| `autoincrement` | automatyczne numerowanie | Nadawanie kolejnych wartości klucza głównego przez bazę |
| `Identity()` | tożsamość (standard SQL) | Nowocześniejsza alternatywa dla `SERIAL`; generuje `GENERATED ... AS IDENTITY` |
| `checkfirst` | najpierw sprawdź | Parametr `create_all`/`Table.create` — sprawdzenie istnienia tabeli przed jej utworzeniem |
| `sorted_tables` | tabele posortowane | Właściwość `MetaData` zwracająca tabele w kolejności uwzględniającej klucze obce |
| `dialect` | dialekt | Warstwa tłumacząca wspólne konstrukcje SQLAlchemy na składnię konkretnej bazy |
| `migration` | migracja | Wersjonowana, odwracalna zmiana schematu bazy (moduł 16) |

---

## Dalsze czytanie

Dokumentacja oficjalna (SQLAlchemy 2.0):

- Definiowanie schematu i `MetaData` — <https://docs.sqlalchemy.org/en/20/core/metadata.html>
- Typy danych i ich zachowanie w dialektach — <https://docs.sqlalchemy.org/en/20/core/type_basics.html>
- Ograniczenia (`Constraint`, `ForeignKey`, `CheckConstraint`, `UniqueConstraint`) — <https://docs.sqlalchemy.org/en/20/core/constraints.html>
- Nazwy ograniczeń i konwencje nazewnicze (`naming_convention`) — <https://docs.sqlalchemy.org/en/20/core/constraints.html#constraint-naming-conventions>
- Generowanie i wykonywanie DDL (`create_all`, `drop_all`, `CreateTable`) — <https://docs.sqlalchemy.org/en/20/core/ddl.html>
- Konstrukcje „wykonywalne” (`DDL`, `CreateTable`, `CreateIndex`) — <https://docs.sqlalchemy.org/en/20/core/ddl.html#sqlalchemy.schema.CreateTable>
- Refleksja bazy i `Inspector` — <https://docs.sqlalchemy.org/en/20/core/reflection.html>
- Dialekt SQLite (ograniczenia i różnice) — <https://docs.sqlalchemy.org/en/20/dialects/sqlite.html>
- Dialekt PostgreSQL (typy natywne, `JSONB`, `Identity`) — <https://docs.sqlalchemy.org/en/20/dialects/postgresql.html>
- Praca z `Engine` i pulą połączeń (powtórka z modułu 02) — <https://docs.sqlalchemy.org/en/20/core/connections.html>

Materiały pokrewne, przydatne przy schemacie:

- PostgreSQL: typy danych — <https://www.postgresql.org/docs/current/datatype.html>
- SQLite: typy i powinowactwo — <https://www.sqlite.org/datatype3.html>
- SQLite: klucze obce (`PRAGMA foreign_keys`) — <https://www.sqlite.org/foreignkeys.html>

> 🆕 **SQLAlchemy 2.1 —** jeśli pracujesz na 2.1, sprawdź „What's New” / changelog pod kątem nowych konstrukcji DDL (`CreateView`, `CREATE TABLE AS SELECT`) oraz zmian w domyślnych sterownikach (`psycopg` dla `postgresql://`, `oracledb` dla `oracle://`) i instalacji `sqlalchemy[asyncio]` (greenlet nie instaluje się już automatycznie): <https://docs.sqlalchemy.org/en/21/changelog/whatsnew_21.html>

---

## Co dalej

Masz teraz w kodzie pełny, czytelny opis struktury bazy i umiesz go wygenerować, utworzyć, obejrzeć oraz porównać z rzeczywistością. Schemat sam w sobie nie zawiera jednak ani jednego wiersza danych — jest tylko projektem domu.

W module 04 nauczymy się **czytać dane**: budować zapytania `select()` krok po kroku, filtrować, sortować, grupować i rozumieć obiekt `Result`, przez który baza oddaje ci wyniki. Nadal pozostaniemy w Core — na obiektach, nie na klasach Pythona.

➡️ Następny moduł: [`04_select_i_result.md`](04_select_i_result.md)

️ Poprzedni moduł: [`02_srodowisko_i_engine.md`](02_srodowisko_i_engine.md)

<!-- koniec modułu 03 -->