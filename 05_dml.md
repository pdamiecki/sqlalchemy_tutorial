# Moduł 05 — Modyfikowanie danych: INSERT, UPDATE, DELETE

Do tej pory baza była dla nas magazynem, z którego tylko czytaliśmy. Ten moduł zmienia kierunek przepływu: uczymy się **zapisywać** dane. Poznasz trzy instrukcje, na których stoi każda aplikacja — dodawanie (`INSERT`), zmienianie (`UPDATE`) i usuwanie (`DELETE`) — a przede wszystkim zrozumiesz, co SQLAlchemy robi *pod spodem*, gdy podajesz mu tysiąc słowników naraz. Dowiesz się, czym jest transakcja i dlaczego „zapisało się tylko pół danych” to najczęstszy błąd początkujących, jak wstawić wiersz i od razu odebrać wygenerowany klucz (`RETURNING`), jak zrobić import idempotentny (uruchomiony dwa razy daje ten sam efekt) oraz jak poradzić sobie ze słowem „upsert”, które w każdej bazie wygląda inaczej. Moduł zamyka pełny, uruchamialny przykład: importer pliku CSV do bazy z deduplikacją i raportem.

---

## Metadane modułu

| | |
|---|---|
| **Poziom** | 🟡 średni |
| **Czas** | ~150 minut (z ćwiczeniami) |
| **Wymagania wstępne** | [`03_metadata_ddl.md`](03_metadata_ddl.md) — umiesz zdefiniować tabele w `MetaData` i utworzyć je w bazie. [`04_select_i_result.md`](04_select_i_result.md) — umiesz pisać `select()`, rozumiesz `Result` i parametry wiązane. |
| **Czego dotyczy plik** | Warstwy **Core** (bez ORM). Zapis danych na poziomie instrukcji SQL: `insert()`, `update()`, `delete()`, transakcje, konflikty, wydajność masowych zapisów. |
| **Baza w przykładach** | SQLite (`sqlite:///library.db`) — zero konfiguracji. Wszędzie, gdzie zachowanie jest specyficzne dla dialektu, jest to **wyraźnie zaznaczone**. |
| **Nie dotyczy** | `Session`, obiektów ORM, `synchronize_session`. Te tematy pojawiają się dopiero w modułach 08 i 10 — tutaj tylko je zapowiadamy. |

---

## Spis treści

1. [Zanim zaczniemy: dlaczego zapis jest trudniejszy niż odczyt](#zanim-zaczniemy-dlaczego-zapis-jest-trudniejszy-niż-odczyt)
2. [\`insert()\` — wstawianie wierszy](#insert--wstawianie-wierszy)
3. [Pod maską: co dzieje się, gdy podajesz 10 000 słowników](#pod-maską-co-dzieje-się-gdy-podajesz-10-000-słowników)
4. [\`RETURNING\` — wstaw i wróć](#returning--wstaw-i-wróć)
5. [\`update()\` — zmienianie istniejących wierszy](#update--zmienianie-istniejących-wierszy)
6. [\`delete()\` — usuwanie wierszy](#delete--usuwanie-wierszy)
7. [Transakcje w Core](#transakcje-w-core)
8. [Upsert: gdy wiersz już istnieje](#upsert-gdy-wiersz-już-istnieje)
9. [Ile wierszy zmieniłem? \`rowcount\`](#ile-wierszy-zmieniłem-rowcount)
10. [Klucze obce i kolejność operacji](#klucze-obce-i-kolejność-operacji)
11. [Masowe operacje: trzy podejścia](#masowe-operacje-trzy-podejścia)
12. [Pełny przykład: import z pliku CSV](#pełny-przykład-import-z-pliku-csv)
13. [Podsumowanie](#podsumowanie)
14. [Ćwiczenia](#ćwiczenia)
15. [Najczęstsze błędy i jak je czytać](#najczęstsze-błędy-i-jak-je-czytać)
16. [Słowniczek modułu](#słowniczek-modułu)
17. [Dalsze czytanie](#dalsze-czytanie)
18. [Co dalej](#co-dalej)

---

## Zanim zaczniemy: dlaczego zapis jest trudniejszy niż odczyt

Odczyt jest bezpieczny. Jeśli pomylisz warunek w `SELECT`, zobaczysz po prostu inne dane — najwyżej ktoś powie „raport jest dziwny”. Nie ma czego zepsuć.

Zapis jest niebezpieczny. Jeśli pomylisz warunek w `UPDATE`, możesz nadpisać ceny wszystkich książek w bazie. Jeśli pominiesz warunek w `DELETE`, usuniesz całą tabelę. Wyniki pracy programisty zajmują sekundy, a odtwarzanie danych z kopii zapasowej — godziny.

Dlatego w tym module proporcje są odwrotne niż przy `select()`: mniej „jak to napisać”, a więcej „jak to zrobić bezpiecznie i wydajnie”.

>  **Analogia — ołówek, długopis i guma**
> `SELECT` to czytanie notatki — możesz czytać ją sto razy, nic się nie zmieni. `INSERT` to dopisanie nowej linijki — potrzebujesz wolnego miejsca i musisz wiedzieć, *gdzie* dopisać. `UPDATE` to gumka i ołówek — zamazujesz coś i piszesz na tym samym miejscu, a najczęstszy wypadek to zamazanie nie tej linijki. `DELETE` to nożyczki: nie ma „cofnij”. Cała reszta tego modułu to techniki trzymania nożyczek w bezpieczny sposób (transakcje) i pisania szybko, ale bez wypadków (masowe operacje).

Trzy rodziny instrukcji SQL mają swoje nazwy, które warto rozróżniać od pierwszego dnia:

| Skrót | Rozwinięcie | Co robi | Instrukcje |
|---|---|---|---|
| **DDL** | *Data Definition Language* | definiuje **strukturę** bazy | `CREATE TABLE`, `ALTER TABLE`, `DROP TABLE` |
| **DML** | *Data Manipulation Language* | zmienia **dane** w środku | `INSERT`, `UPDATE`, `DELETE` |
| **DQL** | *Data Query Language* | **czyta** dane | `SELECT` |

> 🧠 **Dlaczego tak jest** — DDL i DML to różne światy, bo mają różne konsekwencje. `ALTER TABLE ... DROP COLUMN` zmienia *kształt* magazynu i wymaga migracji (moduł 16). `UPDATE` zmienia zawartość i wymaga transakcji (ten moduł). SQLAlchemy też to rozdziela: DDL opisujesz obiektami `Table`/`MetaData` (moduł 03), a DML konstrukcjami `insert()`, `update()`, `delete()` z `sqlalchemy`.

Zaczynamy od szkicu schematu, którego będziemy używać przez cały moduł — to ta sama biblioteka, którą napotkasz w module 09, tylko zapisana w stylu Core:

```python
# examples/05_schema.py
"""Schemat biblioteki używany w całym module 05."""

from sqlalchemy import (
    Column,
    DateTime,
    ForeignKey,
    Integer,
    MetaData,
    Numeric,
    String,
    Table,
    func,
    text,
)

metadata = MetaData()

authors = Table(
    "authors",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String(100), nullable=False),
    Column("country", String(50)),
)

books = Table(
    "books",
    metadata,
    Column("id", Integer, primary_key=True),
    # ISBN to numer identyfikujący książkę na świecie — naturalny klucz biznesowy.
    Column("isbn", String(13), nullable=False, unique=True),
    Column("title", String(200), nullable=False),
    Column("author_id", ForeignKey("authors.id", ondelete="CASCADE"), nullable=False),
    Column("published_year", Integer),
    Column("price", Numeric(8, 2)),
    # server_default: wartość nadawana PO STRONIE BAZY, gdy nie podasz nic.
    Column("stock", Integer, nullable=False, server_default=text("0")),
)

members = Table(
    "members",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("email", String(255), nullable=False, unique=True),
    Column("full_name", String(200), nullable=False),
    Column("joined_at", DateTime, server_default=func.now()),
)

loans = Table(
    "loans",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("book_id", ForeignKey("books.id"), nullable=False),
    Column("member_id", ForeignKey("members.id"), nullable=False),
    Column("loaned_at", DateTime, server_default=func.now()),
    Column("returned_at", DateTime),
)
```

> ⚠️ **Pułapka** — nazwa tabeli `'loans'` jest po angielsku, ale **wszystkie identyfikatory w kodzie** są po angielsku. Trzymaj się jednej konwencji: identyfikatory i nazwy kolumn po angielsku, komentarze i wyjaśnienia po polsku. Mieszanie języków w nazwach kolumn (`cena_ksiazki`, `BookTitle`) to najszybszy sposób na projekt, którego nikt nie chce czytać.

---

## `insert()` — wstawianie wierszy

Funkcja `insert()` pochodzi z pakietu `sqlalchemy` i tworzy obiekt `Insert` — **opis** instrukcji, nie samo wykonanie. To rozróżnienie jest fundamentalne i wraca w każdym module: konstruujesz *instrukcję*, a wykonujesz ją dopiero, przekazując do `conn.execute()`.

>  **Analogia — zamówienie u kelnera**
> `insert(books).values(...)` to zapisanie zamówienia na kartce. `conn.execute(...)` to zaniesienie kartki do kuchni. Możesz napisać zamówienie, obejrzeć je, wydrukować (`print(stmt)`), a nawet wyrzucić do kosza — dopóki nie zaniesiesz go do kuchni, nic się nie dzieje. Ten podział jest ogromnie wygodny: pozwala logować instrukcje, testować je i przekazywać dalej jako parametr funkcji.

### Najprostsza forma

```python
# examples/05_insert_one.py
"""Wstawianie pojedynczego wiersza — z podejrzeniem wygenerowanego SQL."""

from sqlalchemy import create_engine, insert, select

from examples._schema import authors, books, metadata  # nasz schemat z 05_schema.py

engine = create_engine("sqlite:///library.db", echo=False)
metadata.create_all(engine)

stmt = insert(authors).values(name="Ursula K. Le Guin", country="USA")

# 🔬 Instrukcja jeszcze nie poszła do bazy — to tylko tekst.
print(stmt)

# INSERT INTO authors (name, country) VALUES (:name, :country)

with engine.begin() as conn:
    result = conn.execute(stmt)

print(result.rowcount)  # 1
```

Trzy rzeczy warte uwagi:

1. **`values()` przyjmuje nazwy kolumn jako argumenty nazwane.** `name=` odnosi się do kolumny `name`. Jeśli użyjesz klucza, którego nie ma w tabeli, SQLAlchemy zgłosi `CompileError` — dopiero przy kompilacji, nie przy budowaniu.
2. **`print(stmt)` pokazuje `:name` i `:country`** — to parametry wiązane, dokładnie ten mechanizm, który chroni przed wstrzyknięciem SQL (moduł 04). Wartości nigdy nie trafiają do tekstu instrukcji.
3. **`result.rowcount`** daje liczbę wstawionych wierszy. Wrócimy do tego szczegółowo w sekcji 9.

Jeśli chcesz zobaczyć instrukcję z **podstawionymi wartościami** (np. do wklejenia w konsolę bazy albo do logu diagnostycznego), użyj kompilacji z `literal_binds`:

```python
from sqlalchemy.dialects import sqlite

print(stmt.compile(dialect=sqlite.dialect(), compile_kwargs={"literal_binds": True}))

# INSERT INTO authors (name, country) VALUES ('Ursula K. Le Guin', 'USA')
```

> ️ **Pułapka** — `literal_binds` służy **tylko do podglądu**. Nigdy nie buduj tym mechanizmem zapytań „do wykonania” przez `exec_driver_sql()`. To jest dokładnie ta droga, która prowadzi z powrotem do wstrzyknięcia SQL. Podgląd tak, wykonanie nie.

### Wiele wierszy w jednym `values()`

Zamiast listy argumentów nazwanych możesz podać **listę słowników**:

```python
# examples/05_insert_many_values.py
stmt = insert(authors).values(
    [
        {"name": "J.R.R. Tolkien", "country": "UK"},
        {"name": "Stanisław Lem", "country": "PL"},
        {"name": "Isaac Asimov", "country": "USA"},
    ]
)

print(stmt)
# INSERT INTO authors (name, country) VALUES (:name, :country), (:name_1, :country_1), (:name_2, :country_2)
```

To jest **jedna instrukcja SQL** z trzema grupami `VALUES`. Nazywa się to czasem *multi-values insert*. Zaleta: jedna runda do bazy zamiast trzech. Ograniczenia:

- wszystkie słowniki muszą mieć **ten sam zestaw kluczy**,
- dialekt musi wspierać wielowartościowy `INSERT` (SQLite, PostgreSQL, MySQL/MariaDB, SQL Server — tak; Oracle — nie),
- liczba grup nie może przekroczyć limitu parametrów sterownika (np. w niektórych konfiguracjach MSSQL jest to ~2100 parametrów; przy 8 kolumnach to ok. 260 wierszy na instrukcję).

>  **Dlaczego tak jest** — baza nie „magazynuje” tanio samych instrukcji. Każde `INSERT` z osobna to: przygotowanie planu, zajęcie miejsca, wpisanie do dziennika transakcji (WAL), potwierdzenie. Sto osobnych `INSERT`-ów potrafi trwać sto razy dłużej niż jeden `INSERT` ze sto grupami `VALUES`, mimo że danych jest tyle samo. Runda do bazy jest kosztowna — i to jest motyw przewodni całej sekcji 3.

### `INSERT ... FROM SELECT`

Czasem dane, które chcesz wstawić, już są w bazie — tylko w innej tabeli. Wtedy nie ma sensu wyciągać ich do Pythona i wysyłać z powrotem. Służy do tego `from_select()`:

```python
# examples/05_insert_from_select.py
"""Kopiujemy autorów amerykańskich do tabeli archiwum."""
from sqlalchemy import Column, Integer, String, MetaData, Table, insert, select

metadata = MetaData()
archive_authors = Table(
    "archive_authors",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String(100), nullable=False),
    Column("country", String(50)),
)

stmt = insert(archive_authors).from_select(
    ["name", "country"],
    select(authors.c.name, authors.c.country).where(authors.c.country == "USA"),
)

print(stmt)
# INSERT INTO archive_authors (name, country)
# SELECT authors.name, authors.country FROM authors WHERE authors.country = :country_1

with engine.begin() as conn:
    conn.execute(stmt)
```

Zwróć uwagę na dwa argumenty `from_select()`:

- **lista nazw kolumn docelowych** — `["name", "country"]`,
- **instrukcja `select()`** — musi zwracać tę samą liczbę kolumn w tej samej kolejności.

> 🔬 **Pod maską** — to jedyny moment w tym module, w którym dane nie przechodzą przez Pythona. Baza czyta z jednej tabeli i pisze do drugiej wewnątrz siebie. Przy milionach wierszy różnica w wydajności jest rzędu setek razy, bo znika wąskie gardło przesyłu przez sieć i pamięć procesu. Jeśli kiedykolwiek przyłapiesz się na pisaniu `rows = conn.execute(select(...)).all()` tuż przed `conn.execute(insert(...), rows)`, zastanów się, czy nie da się tego zapisać jednym `from_select()`.

### Wstawianie z listą parametrów — wariant podstawowy

Jest jeszcze trzeci sposób, odmienny od poprzednich: przekazujesz instrukcję **bez** `values()`, a listę słowników podajesz jako drugi argument `execute()`.

```python
# examples/05_insert_params.py
from sqlalchemy import insert

with engine.begin() as conn:
    result = conn.execute(
        insert(authors),
        [
            {"name": "Terry Pratchett", "country": "UK"},
            {"name": "Neil Gaiman", "country": "UK"},
        ],
    )
```

Wizualnie wygląda podobnie do `values([...])`, ale technicznie to **zupełnie inny mechanizm**. W tym miejscu przechodzimy do sekcji, która jest sercem tego modułu.

---

## Pod maską: co dzieje się, gdy podajesz 10 000 słowników

To prawdopodobnie najważniejsza sekcja modułu. Brzmi technicznie, ale rozumiejąc ją raz, będziesz wiedzieć, dlaczego jeden wariant importu trwa 2 sekundy, a inny 3 minuty — bez zgadywania.

> 💡 **Analogia — dostawa do magazynu**
> Wyobraź sobie, że musisz dostarczyć do magazynu 1000 paczek. Masz trzy strategie:
> 1. **Jeden kurs na paczkę** — wracasz do magazynu 1000 razy. Bezpiecznie, ale drogo: samo dojście zajmuje więcej niż rozładunek.
> 2. **Jeden kurs, jedna wielka paleta** — pakujesz wszystkie 1000 paczek na jedną paletę. Najszybciej, ale paleta musi się zmieścić w drzwiach (limit parametrów).
> 3. **Kilkanaście kursów po ~100 paczek** — kompromis. I dokładnie to robi SQLAlchemy domyślnie.

### Trzy tryby wykonania

Gdy wywołujesz `conn.execute(instrukcja, lista_parametrów)` albo `conn.execute(instrukcja_wielowartościowa)`, SQLAlchemy musi zdecydować, jak rozmówić się ze sterownikiem bazy (DBAPI — *DataBase API*, standardowy interfejs Pythona do baz). Są trzy drogi:

| Tryb | Kiedy się włącza | Ile instrukcji SQL leci do bazy |
|---|---|---|
| **`execute`** | jeden zestaw parametrów | 1 |
| **`executemany`** | lista zestawów parametrów, bez `RETURNING` | 1 instrukcja wykonana N razy po stronie sterownika |
| **`insertmanyvalues`** | lista zestawów parametrów + `RETURNING`, na dialekcie, który to wspiera | $\lceil N / \text{page\_size} \rceil$ instrukcji wielowartościowych |

Najciekawszy jest trzeci. Wyjaśnijmy go, bo to jest „ukryta optymalizacja”, która w 2.0 działa domyślnie i o której większość początkujących nawet nie wie.

### `executemany` — standard sterownika

`executemany` to metoda w standardzie DBAPI:

```python
cursor.executemany("INSERT INTO authors (name, country) VALUES (?, ?)", lista_krotek)
```

Sterownik wysyła do serwera **jedną instrukcję z parametrami** i tablicę wartości. Baza wykonuje ją N razy. To dużo lepsze niż pętla po stronie Pythona (bo mniej rund do bazy — zazwyczaj jedna, zależnie od sterownika), ale ma poważny problem: **nie da się w prosty sposób odebrać wygenerowanych kluczy głównych**. W SQLite i PostgreSQL nie masz jak zapytać „a jakie `id` dostały te trzy wiersze, które właśnie wstawiłem?”.

> 🧠 **Dlaczego tak jest** — `executemany` powstał w czasach, gdy bazy nie miały eleganckiego mechanizmu zwracania danych z `INSERT`. Standard zakłada „wsadź i zapomnij”. A aplikacje bardzo często potrzebują odwrotnie: „wsadź i powiedz mi, co powstało”.

### `insertmanyvalues` — mechanizm SQLAlchemy 2.0

SQLAlchemy rozwiązuje ten problem, **przepisując** instrukcję. Zamiast wysyłać jedno `INSERT` z jednym `VALUES`, skleja kilka zestawów wartości w jedno `VALUES` i dokłada `RETURNING`:

```sql
-- To, co napisałeś (jeden zestaw parametrów + RETURNING):
INSERT INTO books (isbn, title, author_id) VALUES (?, ?, ?) RETURNING books.id

-- To, co SQLAlchemy faktycznie wyśle dla 1000 słowników:
INSERT INTO books (isbn, title, author_id) VALUES
    (?, ?, ?), (?, ?, ?), ..., (?, ?, ?)     -- po 1000 wierszy na instrukcję
RETURNING books.id
```

Zamiast 1000 rund do bazy masz $\lceil 1000 / 1000 \rceil = 1$ rundę. Domyślny `page_size` wynosi **1000**.

> 🔬 **Pod maską** — czy `insertmanyvalues` się włączy, decydują trzy warunki, sprawdzane automatycznie:
> 1. dialekt wspiera wielowartościowy `INSERT` (`dialect.use_insertmanyvalues`),
> 2. instrukcja zawiera `RETURNING`,
> 3. backend wspiera `RETURNING` przy `executemany` (`dialect.insert_executemany_returning`).
>
> Wbudowane dialekty SQLite, PostgreSQL, MySQL/MariaDB i SQL Server mają to włączone. Oracle ma `False`, bo realizuje „executemany z RETURNING” natywnie, innym mechanizmem (i nie wspiera wielowartościowego `INSERT`). Domyślny dialekt ma `False`.
>
> Możesz to sprawdzić na żywo:
> ```python
> print(engine.dialect.use_insertmanyvalues)                    # True dla sqlite/pg
> print(engine.dialect.use_insertmanyvalues_wo_returning)       # czy także bez RETURNING
> print(engine.dialect.insertmanyvalues_page_size)              # 1000
> ```

Jest jeszcze flaga `use_insertmanyvalues_wo_returning`. Domyślnie `False` w większości dialektów, ale np. dialekt `psycopg2` ustawia ją na `True`:

| Ustawienie | Znaczenie |
|---|---|
| `use_insertmanyvalues` | czy w ogóle używać tego mechanizmu (domyślnie `True` dla sqlite, pg, mysql/mariadb, mssql) |
| `use_insertmanyvalues_wo_returning` | czy używać go **także** dla `INSERT`-ów **bez** `RETURNING` |

### Kontrola rozmiaru partii

`page_size` możesz zmienić na trzech poziomach: przy tworzeniu silnika, na połączeniu albo na pojedynczej instrukcji.

```python
# examples/05_page_size.py
from sqlalchemy import create_engine, insert

# 1) Globalnie, dla całego silnika.
engine = create_engine("sqlite:///library.db", insertmanyvalues_page_size=500)

# 2) Na obiekcie zdefiniowanej instrukcji.
stmt = (
    insert(books)
    .returning(books.c.id)
    .execution_options(insertmanyvalues_page_size=2_000)
)

# 3) Na połączeniu — wszystkie kolejne instrukcje na nim.
with engine.connect() as conn:
    tuned = conn.execution_options(insertmanyvalues_page_size=250)
```

> ⚠️ **Pułapka** — nie zwiększaj `page_size` „na wszelki wypadek”. Limit nie jest narzucony przez SQLAlchemy, ale przez **liczbę parametrów, które sterownik potrafi przyjąć w jednej instrukcji** (`insertmanyvalues_max_parameters`, np. 32700 w SQLite albo 65535 w PostgreSQL). Przy szerokich tabelach (20 kolumn × 5000 wierszy = 100 000 parametrów) albo przekroczysz limit, albo zbudujesz instrukcję o rozmiarze kilku megabajtów, którą serwer będzie parsował dłużej, niż trwałby sam zapis. Zaczynaj od wartości domyślnej i zmieniaj tylko po pomiarze — mierzenie jest tematem modułu 17.
>
> Praktyczna wskazówka: **wąskie tabele → można zwiększyć do 2000–5000; szerokie (dużo kolumn tekstowych) → zostaw 1000, a nawet zmniejsz do 250–500.**

>  **Ćwiczenie** — zbuduj tabelę z jedną kolumną `Integer` i drugą `String(5000)`. Wstaw 5000 wierszy przy `page_size` domyślnym i przy `page_size=2000`. Zmierz `time.perf_counter()` w obu przypadkach i porównaj. Co się stało i dlaczego?

---

## `RETURNING` — wstaw i wróć

Klucze główne nadawane przez bazę (autoincrement, `IDENTITY`, sekwencje) są najczęstszym powodem, dla którego potrzebujesz `RETURNING`. Bez niego po wstawieniu wiersza nie wiesz, jaki `id` dostał — i musisz go dopytywać osobnym `SELECT`.

> 💡 **Analogia — kwit z magazynu**
> Wysyłasz paletę towaru i chcesz mieć na dokumencie numery przyjęcia każdej paczki. Możesz albo zapytać po fakcie „jaki numer dostała paczka o tym opisie?” (to działa tylko wtedy, gdy opis jest unikalny — `SELECT` po `isbn`), albo poprosić o **kwit wypisany automatycznie w momencie przyjęcia**. `RETURNING` to ten kwit.

```python
# examples/05_returning_insert.py
from sqlalchemy import insert, select

with engine.begin() as conn:
    stmt = insert(books).values(
        isbn="9780898598910",
        title="The Left Hand of Darkness",
        author_id=1,
        published_year=1969,
        price=39.90,
    ).returning(books.c.id)

    print(stmt)
    # INSERT INTO books (isbn, title, author_id, published_year, price)
    # VALUES (:isbn, :title, :author_id, :published_year, :price)
    # RETURNING books.id

    new_id = conn.execute(stmt).scalar_one()
    print(new_id)  # 1
```

Zwróć uwagę na `scalar_one()` — to metoda z `Result`, którą znasz z modułu 04. `RETURNING` zwraca pełnoprawny `Result`, więc masz do dyspozycji `all()`, `first()`, `mappings()` i resztę.

### Zwracanie całych wierszy

`returning()` przyjmuje nie tylko kolumny, ale i całe tabele — a nawet dowolne wyrażenia:

```python
# Cały wiersz z wszystkimi kolumnami (uwaga: stock został nadany przez serwer).
stmt = insert(books).values(...).returning(books)
row = conn.execute(stmt).one()
print(row)  # (1, '9780898598910', 'The Left Hand of Darkness', 1, 1969, Decimal('39.90'), 0)
```

Możesz też łączyć kolumny i wyrażenia, np. `returning(books.c.id, books.c.stock + 1)`.

### `RETURNING` przy `UPDATE` i `DELETE`

To mniej znana, ale bardzo użyteczna możliwość. `update()` i `delete()` mają dokładnie te same metody `returning()`:

```python
# examples/05_returning_update_delete.py
from sqlalchemy import delete, update

stmt = (
    update(books)
    .where(books.c.author_id == 1)
    .values(price=books.c.price * 1.10)
    .returning(books.c.id, books.c.title, books.c.price)
)

print(stmt)
# UPDATE books SET price=(books.price * :price_1)
# WHERE books.author_id = :author_id_1
# RETURNING books.id, books.title, books.price
```

```python
stmt = delete(members).where(members.c.joined_at < "2020-01-01").returning(members.c.id)

print(stmt)
# DELETE FROM members WHERE members.joined_at < :joined_at_1 RETURNING members.id
```

> 🧠 **Dlaczego tak jest** — bez `RETURNING` przy `DELETE` najgorszym scenariuszem jest „skasowałem i nie wiem co”. Trzeba najpierw zrobić `SELECT`, zapisać `id` w Pythonie, potem `DELETE ... WHERE id IN (...)`. To dwie rundy do bazy i okno, w którym ktoś inny może zmienić dane pomiędzy. `DELETE ... RETURNING` robi to atomowo: jedno polecenie, jedna runda, spójny wynik.

### Kolejność zwracanych wierszy

Tu jest subtelność, która potrafi zepsuć kod w produkcji.

> ⚠️ **Pułapka** — **nie ma gwarancji, że wiersze z `RETURNING` wrócą w kolejności, w jakiej podałeś parametry.** Standard SQL tego nie obiecuje. Jeśli wstawiasz 100 wierszy i chcesz przypisać zwrócone `id` do konkretnych obiektów wejściowych, w ogólnym przypadku nie możesz tego zrobić przez `zip(rows_in, rows_out)`.
>
> SQLAlchemy 2.0.10 dodała na to lekarstwo: parametr `sort_by_parameter_order` przy `returning()`:
>
> ```python
> stmt = (
>     insert(books)
>     .returning(books.c.id, sort_by_parameter_order=True)
> )
> ```
>
> Naucz się tego jako reguły: **potrzebujesz dopasowania wejście↔wyjście? ustaw `sort_by_parameter_order=True`**. Kosztuje to dodatkową pracę po stronie bazy (nierzadko wykonanie po jednym wierszu, np. w SQLite) — więc używaj tylko wtedy, gdy naprawdę mapujesz wyniki na wejście.

> 🆕 **SQLAlchemy 2.1** — sprawdzanie dostępności `RETURNING` jest w 2.1 lepiej ujawniane w typach i komunikatach; nie zmienia się samo API `returning()`. Jeżeli używasz 2.1, pamiętaj o zmianie domyślnego sterownika dla `postgresql://` na `psycopg` (w 2.0 domyślnym był `psycopg2`). Nie wpływa to na samo `RETURNING`, ale zmienia URL-e i sposób instalacji zależności.

---

## `update()` — zmienianie istniejących wierszy

`update()` tworzy obiekt `Update`. Składnia jest zdumiewająco podobna do `insert()` — łączy je wspólna klasa bazowa i wspólna metoda `values()`. Ale jest jedna zasadnicza różnica: **`update()` bez `where()` modyfikuje całą tabelę**.

>  **Analogia — list wyborczy**
> `where()` w `UPDATE` to zakres odpowiedzialności. „Zmień cenę książek autorstwa Lema” i „zmień cenę wszystkich książek” to ta sama instrukcja, różniąca się jednym zdaniem. Wyobraź sobie, że wysyłasz e-mail: brak adresata oznacza „wyślij do wszystkich” — i tak samo działa `UPDATE` bez `where()`.

### Warunek jest wszystkim

```python
# examples/05_update_where.py
from sqlalchemy import update

# ✅ Zmiana jednego pola w jednej książce.
stmt = (
    update(books)
    .where(books.c.isbn == "9780898598910")
    .values(price=45.00)
)

print(stmt)
# UPDATE books SET price=:price WHERE books.isbn = :isbn_1
```

Porównanie z wersją „śmiertelnie niebezpieczną”:

```python
#  To zmieni cenę WSZYSTKICH książek w bazie.
stmt = update(books).values(price=45.00)

print(stmt)
# UPDATE books SET price=:price
```

> ⚠️ **Pułapka** — SQLAlchemy **nie ostrzega** przed `update()` bez `where()`. Uznaje, że wiesz, co robisz — czasem naprawdę chcesz hurtowej zmiany. Wypracuj nawyk: pisz `.where()` **natychmiast**, w tej samej linii, w której piszesz `update(...)`. Zanim dopiszesz `values()` — masz już warunek? Dobrze. Zaczynasz od `values()`? Zatrzymaj się.

### Wiele kolumn i wyrażenia

`values()` przyjmuje wiele kolumn naraz, a wartością może być **wyrażenie SQL**, nie tylko stała:

```python
# examples/05_update_expressions.py
from sqlalchemy import bindparam, func, update

# Kilka kolumn jednocześnie.
stmt = (
    update(books)
    .where(books.c.id == 7)
    .values(title="Nowy tytuł", published_year=1971, price=49.90)
)

# Wartość liczona po stronie bazy: podwyżka o 10%.
stmt = update(books).where(books.c.stock > 0).values(price=books.c.price * 1.10)
print(stmt)
# UPDATE books SET price=(books.price * :price_1) WHERE books.stock > :stock_1

# Licznik: zwiększ stan magazynowy o 5.
stmt = update(books).where(books.c.id == 7).values(stock=books.c.stock + 5)
print(stmt)
# UPDATE books SET stock=(books.stock + :stock_1) WHERE books.id = :id_1

# Wartość z innej kolumny tego samego wiersza.
stmt = update(books).values(title=func.upper(books.c.title))
print(stmt)
# UPDATE books SET title=upper(books.title)

# Wyrażenie warunkowe: obniżka tylko dla pozycji starszych niż 1990.
stmt = (
    update(books)
    .values(price=books.c.price * 0.5)
    .where(books.c.published_year < 1990)
)
```

> 🧠 **Dlaczego tak jest** — wyrażenie `books.c.price * 1.10` wykonuje się **w bazie**, nie w Pythonie. To ma dwie ogromne zalety. Po pierwsze, jest **atomowe** — nikt nie może wcisnąć się między odczyt ceny a zapis nowej ceny (o tym w module 14, przy wyścigach). Po drugie, nie musisz wcześniej pobierać wartości: `SELECT` + obliczenie + `UPDATE` to trzy rundy i okno wyścigu, a `UPDATE books SET price = price * 1.1` to jedna runda i zero wyścigu. To wzorzec, do którego będziesz wracał dziesiątki razy.

### `UPDATE ... FROM` — aktualizacja na podstawie innych tabel

PostgreSQL, SQL Server i MySQL wspierają rozszerzoną formę `UPDATE`, w której można wspomnieć dodatkowe tabele. W SQLAlchemy **w 2.0 nie ma na to osobnej metody** — mechanizm włącza się automatycznie, gdy w `where()` pojawi się druga tabela:

```python
# examples/05_update_from.py
from sqlalchemy import and_, update
from sqlalchemy.dialects import postgresql

# Podnieś cenę książek autorów z Polski o 10%.
stmt = (
    update(books)
    .where(books.c.author_id == authors.c.id)      # ← druga tabela w WHERE
    .where(authors.c.country == "PL")
    .values(price=books.c.price * 1.10)
)

print(stmt.compile(dialect=postgresql.dialect()))
# UPDATE books SET price=(books.price * %(price_1)s)
# FROM authors
# WHERE books.author_id = authors.id AND authors.country = %(country_1)s
```

Zwróć uwagę: w SQL na końcu pojawiło się `FROM authors`. Tabelę „dodał” dialekt PostgreSQL, widząc `authors` w warunku. Na SQLite ta instrukcja się nie skompiluje — SQLite nie zna `UPDATE ... FROM` w tej formie.

> ⚠️ **Pułapka (dialekty)** — `UPDATE ... FROM` **nie jest dostępne w SQLite i Oracle**. Kod, który działa na PostgreSQL, wywali się tam z błędem składni. Jeśli piszesz kod przenośny między bazami, użyj **podzapytania skorelowanego** — działa wszędzie:
>
> ```python
> # Przenośna alternatywa: podzapytanie skalarne.
> from sqlalchemy import select
>
> przecena = (
>     select(authors.c.country)
>     .where(authors.c.id == books.c.author_id)
>     .scalar_subquery()
> )
> stmt = update(books).where(przecena == "PL").values(price=books.c.price * 1.10)
> print(stmt)
> # UPDATE books SET price=(books.price * :price_1)
> # WHERE (SELECT authors.country FROM authors WHERE authors.id = books.author_id) = :country_1
> ```
>
> Mniej czytelne, ale działa na każdej bazie. Zawsze rozważ, czy Twoja aplikacja naprawdę musi być przenośna — to pytanie architektoniczne i wrócimy do niego w module 19.

### `ordered_values()` — kolejność ma znaczenie (MySQL)

W MySQL kolejność przypisań w klauzuli `SET` wpływa na wynik, bo kolejne wyrażenia widzą już zmienione kolumny. Gdy zależy Ci na tej kolejności, użyj `ordered_values()` z sekwencją krotek (w Pythonie 3.7+ słowniki też są uporządkowane, ale `ordered_values()` jest jednoznaczne w zamiarze):

```python
from sqlalchemy import update

stmt = update(books).ordered_values(
    (books.c.y, 20),
    (books.c.x, books.c.y + 10),   # widzi już y == 20
)
```

Poza MySQL nie ma to znaczenia.

---

## `delete()` — usuwanie wierszy

Zasada jest identyczna jak przy `update()`: **bez `where()` usuwamy wszystko**.

```python
# examples/05_delete.py
from sqlalchemy import delete

# ✅ Usuń jednego członka.
stmt = delete(members).where(members.c.email == "stary@example.com")
print(stmt)
# DELETE FROM members WHERE members.email = :email_1

# ⚠️ Usuń WSZYSTKICH członków.
stmt = delete(members)
print(stmt)
# DELETE FROM members
```

### Usuwanie z warunkami w innych tabelach

Podobnie jak przy `UPDATE`, wystarczy wspomnieć drugą tabelę w `where()`:

```python
# examples/05_delete_multi.py
from sqlalchemy import delete
from sqlalchemy.dialects import mysql, postgresql

# Usuń członków, którzy mają nieoddane wypożyczenia.
stmt = (
    delete(members)
    .where(members.c.id == loans.c.member_id)
    .where(loans.c.returned_at.is_(None))
)

print(stmt.compile(dialect=postgresql.dialect()))
# DELETE FROM members USING loans
# WHERE members.id = loans.member_id AND loans.returned_at IS NULL
```

> ⚠️ **Pułapka (dialekty)** — powyższa forma działa na PostgreSQL i MySQL, ale **nie w SQLite**. Dla SQLite użyj `IN` z podzapytaniem:
>
> ```python
> from sqlalchemy import select
>
> zadluzeni = (
>     select(loans.c.member_id).where(loans.c.returned_at.is_(None))
> )
> stmt = delete(members).where(members.c.id.in_(zadluzeni))
> print(stmt)
> # DELETE FROM members
> # WHERE members.id IN (SELECT loans.member_id FROM loans WHERE loans.returned_at IS NULL)
> ```

>  **SQLAlchemy 2.1** — `Delete.using()`
> Do 2.1 wielotabelowe `DELETE` było **wnioskowane** z klauzuli `WHERE` — co działa dla prostych przypadków, ale nie pozwala zapisać jawnego złączenia. Wersja 2.1 dodaje metodę `Delete.using()`:
>
> ```python
> from sqlalchemy import delete
>
> stmt = (
>     delete(members)
>     .using(
>         members.outerjoin(loans, members.c.id == loans.c.member_id)
>     )
>     .where(loans.c.returned_at.is_(None))
> )
> ```
>
> Na MySQL/MariaDB wyrenderuje się jako `DELETE FROM members USING members LEFT OUTER JOIN loans ON ... WHERE ...`, a na PostgreSQL — w analogicznej, właściwej dla niego formie `DELETE ... USING`. To funkcja specyficzna dla backendów obsługujących wielotabelowe `DELETE`. W 2.0 użyj wariantu z `IN (podzapytanie)` — jest przenośny i w praktyce równie szybki dla większości przypadków.

---

## Transakcje w Core

Zbliżamy się do najważniejszego pojęcia całego modułu. Jeśli zapamiętasz z tego pliku jedną rzecz, niech to będzie ta sekcja.

> 💡 **Analogia — przelew bankowy**
> Z Twojego konta schodzi 100 zł, na konto odbiorcy wpływa 100 zł. Co się stanie, jeśli w połowie tej operacji zabraknie prądu? Albo oba ruchy się wykonają, albo żaden. Nie ma opcji „zabrano mi 100 zł i nigdzie ich nie ma”.
> Ta obietnica nazywa się **transakcją**. Baza danych gwarantuje, że grupa operacji albo wykona się w całości, albo w ogóle. Trzy pozostałe obietnice transakcji opisuje skrót **ACID** (wyjaśniony w słowniczku na końcu modułu).

### `engine.begin()` — domyślny, właściwy sposób

```python
# examples/05_transaction_begin.py
from sqlalchemy import create_engine, insert

engine = create_engine("sqlite:///library.db")

with engine.begin() as conn:
    conn.execute(insert(authors).values(name="Stanisław Lem", country="PL"))
    conn.execute(insert(books).values(
        isbn="9780156027601", title="Solaris", author_id=4, published_year=1961
    ))
# Wyjście z bloku `with` = COMMIT.
```

Co dokładnie się dzieje:

1. Wchodząc w blok, SQLAlchemy pobiera połączenie z puli i wysyła `BEGIN`.
2. Wykonuje Twoje instrukcje.
3. Wychodząc z bloku **bez wyjątku** → `COMMIT`.
4. Wychodząc z bloku **z wyjątkiem** → `ROLLBACK` i ponowne podniesienie wyjątku.

To ostatnie jest bezcenne. Zapomnij o `try/except/finally` — wystarczy, że wyjątek wyleci z bloku, a baza sama wróci do stanu sprzed.

```python
# examples/05_transaction_rollback.py
from sqlalchemy import insert

try:
    with engine.begin() as conn:
        conn.execute(insert(authors).values(name="Autor A", country="PL"))
        # Wymuszamy błąd: autor_id, którego nie ma → naruszenie klucza obcego.
        conn.execute(insert(books).values(
            isbn="0000000000001", title="Nieistniejący", author_id=99999
        ))
except Exception as exc:
    print("Transakcja wycofana:", exc)

# Autor A NIE istnieje w bazie. Cała transakcja została cofnięta.
```

>  **Pod maską** — przy `echo=True` zobaczysz sekwencję:
> ```text
> BEGIN (implicit)
> INSERT INTO authors (name, country) VALUES (?, ?)  [('Autor A', 'PL')]
> INSERT INTO books (isbn, title, author_id) VALUES (?, ?, ?)  [...]
> ROLLBACK
> ```
> Zwróć uwagę na `(implicit)`. Oznacza to, że `BEGIN` nie został wysłany osobnym poleceniem — baza weszła w tryb transakcji automatycznie, a SQLAlchemy tylko to zaraportowało. To tzw. **autobegin**, o którym za chwilę.

### `commit()` i `rollback()` ręcznie

`engine.begin()` to wygodny skrót. Czasem potrzebujesz sterować transakcją ręcznie — np. gdy transakcja ma trwać dłużej niż jeden blok kodu:

```python
# examples/05_transaction_manual.py
with engine.connect() as conn:
    trans = conn.begin()          # jawnie zaczynamy
    try:
        conn.execute(insert(authors).values(name="Autor B", country="UK"))
        # ... dziesiątki operacji ...
        trans.commit()
    except Exception:
        trans.rollback()
        raise
```

> ⚠️ **Pułapka** — jeśli używasz `conn.begin()` i zapomnisz o `commit()`, **wychodząc z bloku `with engine.connect()` dane zostaną wycofane**. `Connection` nie commituje automatycznie — tylko `engine.begin()` to robi. To najczęstsza przyczyna zdania „uruchomiłem skrypt, nie było błędu, ale w bazie nic nie ma”. Narzędzia do diagnozy tego przypadku są w sekcji 15.

### `begin_nested()` — savepointy

**Savepoint** to „punkt kontrolny” wewnątrz transakcji. Możesz się cofnąć do niego, nie tracąc wszystkiego, co wydarzyło się przed nim.

> 💡 **Analogia — zapis w grze**
> Transakcja to całe przejście poziomu. Savepoint to zapis w połowie. Jeśli zginiesz na końcu, wczytujesz zapis, a nie zaczynasz grę od nowa. Kluczowe: **savepoint żyje tylko wewnątrz transakcji**. Koniec transakcji = wszystkie savepointy znikają.

```python
# examples/05_savepoint.py
from sqlalchemy import insert
from sqlalchemy.exc import IntegrityError

rows = [
    {"isbn": "1111111111111", "title": "Książka 1", "author_id": 1},
    {"isbn": "1111111111111", "title": "Duplikat ISBN", "author_id": 1},  # ← konflikt
    {"isbn": "2222222222222", "title": "Książka 3", "author_id": 1},
]

with engine.begin() as conn:
    dodane, pominiete = 0, 0
    for row in rows:
        try:
            with conn.begin_nested():          # ← SAVEPOINT
                conn.execute(insert(books).values(**row))
            dodane += 1
        except IntegrityError:
            pominiete += 1                     # ← ROLLBACK TO SAVEPOINT
    print(f"dodane={dodane}, pominiete={pominiete}")  # dodane=2, pominiete=1

# Na końcu: COMMIT — książki 1 i 3 są w bazie.
```

Wygenerowany SQL:

```text
BEGIN (implicit)
SAVEPOINT sa_savepoint_1
INSERT INTO books (isbn, title, author_id) VALUES (?, ?, ?)
RELEASE SAVEPOINT sa_savepoint_1
SAVEPOINT sa_savepoint_2
INSERT INTO books (isbn, title, author_id) VALUES (?, ?, ?)
ROLLBACK TO SAVEPOINT sa_savepoint_2
SAVEPOINT sa_savepoint_3
INSERT INTO books (isbn, title, author_id) VALUES (?, ?, ?)
RELEASE SAVEPOINT sa_savepoint_3
COMMIT
```

> 🧠 **Dlaczego tak jest** — bez savepointu pierwszy błąd (duplikat ISBN) zepsułby **całą** transakcję na PostgreSQL: po błędzie transakcja wchodzi w stan „aborted” i żadne dalsze polecenie nie przejdzie, dopóki nie zrobisz `ROLLBACK`. Savepoint daje Ci „cofnij tylko ten kawałek i idź dalej”. To technika, którą będziesz stosować przy imporcie danych wiersz po wierszu, gdy chcesz pominąć wadliwe rekordy, a nie przerywać całego procesu.

### Autobegin — co się dzieje, gdy nic nie zrobisz

> 🧠 **Dlaczego tak jest** — w SQLAlchemy 2.0 nie ma trybu „autocommit” znanego z 1.x. Zamiast tego działa **autobegin**: przy pierwszym poleceniu na połączeniu SQLAlchemy automatycznie rozpoczyna transakcję, jeśli żadna nie jest otwarta. Możesz to sprawdzić:
>
> ```python
> with engine.connect() as conn:
>     print(conn.in_transaction)          # False
>     conn.execute(select(1))
>     print(conn.in_transaction)          # True ← transakcja zaczęła się sama
> ```
>
> Konsekwencja praktyczna: **nie ma już „połowicznego zapisu”**. W starym świecie `engine.execute("INSERT ...")` działało natychmiast. W 2.0 każde zapytanie żyje w transakcji, którą trzeba zakończyć commitem. `engine.execute()` zresztą już nie istnieje w 2.0 — usunięto je razem z całym modelem autocommit. To jest zmiana, o której piszę najczęściej w kodzie migrowanym z 1.x.

> ⚠️ **Pułapka** — nie otwieraj drugiej „zwykłej” transakcji na tym samym połączeniu. `conn.begin()` wywołane, gdy transakcja już trwa, zgłosi `InvalidRequestError`. Do zagnieżdżania służy **wyłącznie** `begin_nested()`. Pamiętaj też: zagnieżdżone `engine.begin()` nie tworzy nowej transakcji — SQLAlchemy użyje istniejącej.

---

## Upsert: gdy wiersz już istnieje

Scenariusz jest codzienny: importujesz dane z pliku, który może zawierać rekordy już obecne w bazie. Chcesz „wstaw, a jeśli już jest — zaktualizuj”. Ta operacja nazywa się **upsert** (od *update + insert*).

> 💡 **Analogia — segregator z aktami**
> Dostajesz nowy komplet dokumentów. Dla osoby, której akta już masz, **podmieniasz kartkę**; dla nowej — **zakładasz nową teczkę**. Nie chcesz ani duplikatów (dwie teczki na tę samą osobę), ani wywracania całego segregatora na podłogę (rollback), gdy trafisz na osobę, którą już znasz.

### `ON CONFLICT DO NOTHING`

Import obsługuje `sqlalchemy.dialects.sqlite.insert` i `sqlalchemy.dialects.postgresql.insert`. To **nie jest** ogólny `insert()` — metoda `on_conflict_*` istnieje tylko na tych wariantach dialektowych.

```python
# examples/05_upsert_nothing.py
from sqlalchemy.dialects.sqlite import insert as sqlite_insert

stmt = (
    sqlite_insert(authors)
    .values(name="Stanisław Lem", country="PL")
    .on_conflict_do_nothing(index_elements=["name"])
)

print(stmt)
# INSERT INTO authors (name, country) VALUES (:name, :country)
# ON CONFLICT (name) DO NOTHING

with engine.begin() as conn:
    conn.execute(stmt)
```

Wariant PostgreSQL:

```python
from sqlalchemy.dialects.postgresql import insert as pg_insert

stmt = (
    pg_insert(authors)
    .values(name="Stanisław Lem", country="PL")
    .on_conflict_do_nothing(index_elements=["name"])
)
```

Warto wiedzieć: PostgreSQL dopuszcza też wariant „bez wskazania kolumn”, który po prostu ignoruje *każdy* konflikt:

```python
stmt = pg_insert(authors).values(...).on_conflict_do_nothing()
# ON CONFLICT DO NOTHING
```

> 🧠 **Dlaczego podajemy `index_elements`** — baza musi wiedzieć, **co konkretnie** uznać za konflikt. `ON CONFLICT (name)` znaczy „jeśli istnieje wiersz o tej samej wartości w kolumnie `name` (i w tym samym unikalnym indeksie), nie rób nic”. Bez wskazania kolumny baza sprawdza wszystkie ograniczenia unikalności. Dla `on_conflict_do_update` wskazanie kolumny jest obowiązkowe — inaczej baza nie wie, których wierszy dotyczy aktualizacja.

### `ON CONFLICT DO UPDATE` i `excluded`

```python
# examples/05_upsert_update.py
from sqlalchemy.dialects.sqlite import insert as sqlite_insert

stmt = (
    sqlite_insert(books)
    .values(
        isbn="9780156027601",
        title="Solaris (wyd. nowe)",
        author_id=4,
        published_year=1961,
        price=54.90,
    )
    .on_conflict_do_update(
        index_elements=["isbn"],
        set_={
            # `excluded` = "to, co CHCIAŁEŚ wstawić".
            "title": "Solaris (wyd. nowe)",
            "price": 54.90,
        },
    )
)

print(stmt)
# INSERT INTO books (isbn, title, author_id, published_year, price)
# VALUES (:isbn, :title, :author_id, :published_year, :price)
# ON CONFLICT (isbn) DO UPDATE SET title = excluded.title, price = excluded.price
```

`excluded` to specjalna „pseudo-tabela” dostępna w klauzuli `DO UPDATE`. Zawiera wiersz, który **próbowałeś** wstawić, ale się nie udało z powodu konfliktu. Dzięki niej możesz pisać aktualizacje oparte na nowych danych:

```python
from sqlalchemy.dialects.sqlite import insert as sqlite_insert

# Bezpieczna aktualizacja licznika: nie nadpisuj, tylko dodaj.
stmt = (
    sqlite_insert(books)
    .values(isbn="9780156027601", title="Solaris", author_id=4, stock=10)
    .on_conflict_do_update(
        index_elements=["isbn"],
        set_={"stock": books.c.stock + 10},     # dołóż do tego, co już jest
    )
)
```

Albo — najczęstszy wzorzec „weź po prostu nowe wartości”:

```python
stmt = (
    sqlite_insert(books)
    .values(...)
    .on_conflict_do_update(
        index_elements=["isbn"],
        set_={"title": stmt.excluded.title, "price": stmt.excluded.price},
    )
)
```

Możesz też zawęzić zakres konfliktu warunkiem na indeksie (`index_where=`) albo ograniczyć samą aktualizację (`where=` w `on_conflict_do_update`) — przydaje się to przy indeksach częściowych.

### Różnice dialektów

| Baza | Składnia | Uwagi |
|---|---|---|
| **SQLite** | `INSERT ... ON CONFLICT (kol) DO NOTHING / DO UPDATE` | Dostępne od SQLite 3.24 (2018). Wymaga wskazania kolumny lub indeksu. Od niedawna wspiera także wiele klauzul `ON CONFLICT` w jednej instrukcji. |
| **PostgreSQL** | `INSERT ... ON CONFLICT (kol) DO NOTHING / DO UPDATE` | Najbogatsza wersja: `index_elements`, `index_where`, `constraint=` (po nazwie ograniczenia), `where` w `DO UPDATE`. Wspiera `excluded`. |
| **MySQL / MariaDB** | `INSERT ... ON DUPLICATE KEY UPDATE` | **Inna składnia** — nie ma `excluded`. SQLAlchemy 2.0 udostępnia to w dialekcie MySQL jako `on_duplicate_key_update()`. |
| **SQL Server** | `MERGE` | Jeszcze inny mechanizm, obsługiwany osobnym konstruktem. |
| **Oracle** | `MERGE` | Jak wyżej. |
| **SQLite (starsze niż 3.24)** | brak | Trzeba ręcznie: `SELECT` → decyzja → `UPDATE` lub `INSERT` w jednej transakcji. |

> ⚠️ **Pułapka (dialekty)** — to jest klasyczna pułapka przenośności. **Nie ma jednego uniwersalnego upsertu** w SQLAlchemy Core. Jeśli Twoja aplikacja działa na MySQL/MariaDB, szukaj `on_duplicate_key_update` w dialekcie MySQL, a nie `on_conflict_do_update`. Kod, który działa na SQLite, po przeniesieniu na MySQL wywali się z `AttributeError`, bo obiekt `mysql.insert` nie ma metody `on_conflict_do_update`.

> 🧠 **Dlaczego upsert jest ważny** — trzy powody:
> 1. **Idempotencja.** Import uruchomiony dwa razy daje ten sam efekt. To absolutna podstawa niezawodnych procesów ETL (Extract-Transform-Load) i kolejek zadań.
> 2. **Atomowość.** Klasyczne „sprawdź, czy istnieje, potem wstaw” ma okno wyścigu: między `SELECT` a `INSERT` inny proces może wstawić ten sam wiersz. Upsert to jedno polecenie — okna nie ma.
> 3. **Wydajność.** Jedna runda do bazy zamiast dwóch, a przy imporcie tysięcy rekordów to różnica rzędu dwukrotności.

---

## Ile wierszy zmieniłem? `rowcount`

`result.rowcount` zwraca liczbę wierszy dopasowanych przez instrukcję. To bardzo przydatne narzędzie, ale ma szereg ograniczeń, które trzeba znać.

```python
# examples/05_rowcount.py
from sqlalchemy import update

with engine.begin() as conn:
    result = conn.execute(
        update(books).values(price=books.c.price * 1.10).where(books.c.published_year < 1990)
    )
    print(result.rowcount)   # np. 17
```

Fakty o `rowcount`:

1. **To liczba wierszy dopasowanych przez `WHERE`, a nie faktycznie zmienionych.** Jeśli `UPDATE` ustawia tę samą wartość, która już była, `rowcount` nadal to policzy. Baza nie porównuje stanu przed i po.
2. **Nie ma gwarancji dostępności przy `RETURNING` i przy `executemany`.** To zależy od sterownika. Jeśli sterownik nie potrafi podać liczby, dostaniesz **`-1`** — nie wyjątek, nie `None`, tylko `-1`. Twój kod musi to obsłużyć.
3. **`rowcount` NIE działa dla `INSERT` dwoma sposobami tak samo.** Przy pojedynczym `INSERT` jest OK. Przy `executemany` bywa niedostępny.
4. **Dla `INSERT` i `SELECT` domyślnie nie jest zachowywany.** Jeśli potrzebujesz go dla nietypowej instrukcji, użyj opcji wykonania:
   ```python
   stmt = stmt.execution_options(preserve_rowcount=True)
   ```
5. **Dostępność sygnalizuje atrybut kursora** `result.supports_sane_rowcount`. Trzecie dialekty (bazy nierelacyjne) mogą tego w ogóle nie wspierać.

> ️ **Pułapka** — kod, który traktuje `rowcount` jako pewnik, jest kodem, który kiedyś zwróci „zaktualizowano 0 rekordów”, mimo że zaktualizował 5000. **Traktuj `rowcount` jako informację opcjonalną.** Jeśli naprawdę musisz znać dokładną liczbę zmienionych wierszy, użyj `RETURNING` i policz zwrócone wiersze:
>
> ```python
> result = conn.execute(
>     update(books).where(...).values(...).returning(books.c.id)
> )
> zmienione = len(result.all())   # dokładna liczba, niezależna od sterownika
> ```
>
> To wolniejsze (bo baza musi wygenerować pełne wiersze), ale wiarygodne. Wybieraj świadomie.

> 🧠 **Dlaczego tak jest** — `rowcount` pochodzi ze standardu DBAPI, który został zaprojektowany, zanim bazy miały `RETURNING`. Różne sterowniki implementowały go po swojemu, niektóre dobrze, inne wcale. SQLAlchemy stara się zapamiętać wartość z kursora, **zanim** kursor zostanie zamknięty (bo część sterowników nie pozwala czytać `rowcount` po fakcie) — ale nie może wyczarować tego, czego sterownik nie podaje.

---

## Klucze obce i kolejność operacji

Klucz obcy to zależność: „wiersz z tabeli `loans` musi wskazywać na istniejący wiersz w `books`”. Baza egzekwuje to przy **każdym** poleceniu, więc kolejność operacji ma znaczenie.

```python
# ❌ To się nie uda — książka jeszcze nie istnieje.
with engine.begin() as conn:
    conn.execute(insert(loans).values(book_id=999, member_id=1))  # IntegrityError

# ✅ Najpierw rodzic, potem dziecko.
with engine.begin() as conn:
    book_id = conn.execute(
        insert(books)
        .values(isbn="9780156027601", title="Solaris", author_id=4)
        .returning(books.c.id)
    ).scalar_one()

    conn.execute(insert(loans).values(book_id=book_id, member_id=1))
```

> 💡 **Analogia — fundament i ściany**
> Klucz obcy to ściana, która musi stać na fundamencie. Nie postawisz ściany, zanim nie wylejesz fundamentu. I nie zburzysz fundamentu, jeśli stoją na nim ściany — chyba że w projekcie zapisano „ściany lecą razem z fundamentem” (`ON DELETE CASCADE`).

### Cykliczne zależności

Bywa, że dwie tabele wskazują na siebie. Przykład: `members` ma kolumnę `favourite_book_id` wskazującą na `books`, a `books` ma `first_borrower_id` wskazującą na `members`. Nie da się wstawić żadnej z nich pierwszej — instrukcja w drugą stronę zawsze trafi na brakujący wiersz.

Masz trzy wyjścia:

**1. Wstaw `NULL`, potem zaktualizuj (działa wszędzie).** Najprostsze i najczęstsze:

```python
with engine.begin() as conn:
    author_id = conn.execute(
        insert(authors).values(name="Autor", country="PL").returning(authors.c.id)
    ).scalar_one()
    # book_id i reservation_id zostają NULL w pierwszym kroku
    book_id = conn.execute(
        insert(books).values(isbn="...", title="...", author_id=author_id)
        .returning(books.c.id)
    ).scalar_one()
    # dopiero teraz domykamy cykl
    conn.execute(update(authors).where(authors.c.id == author_id).values(...))
```

**2. Wstaw z jawnie podanymi kluczami.** Jeśli nie używasz autoincrement i sam nadajesz `id`, kolejność przestaje być problemem — po prostu wstawiasz obie strony z gotowymi wartościami.

**3. Ograniczenia odroczone — `DEFERRABLE` (PostgreSQL).**

### `DEFERRABLE` — odroczenie sprawdzenia ograniczenia

Standard SQL pozwala zadeklarować, że ograniczenie ma być sprawdzane **na końcu transakcji**, a nie przy każdym poleceniu.

```python
# examples/05_deferrable.py
from sqlalchemy import Column, ForeignKeyConstraint, Integer, MetaData, String, Table

metadata = MetaData()

members = Table(
    "members",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("email", String(255), nullable=False),
    Column("favourite_book_id", Integer),
)

books = Table(
    "books",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("isbn", String(13), nullable=False),
    Column("title", String(200), nullable=False),
)

# Ograniczenie klucza obcego na favourite_book_id — odroczone.
metadata.append_constraint(
    ForeignKeyConstraint(
        ["favourite_book_id"], ["books.id"],
        deferrable=True,
        initially="DEFERRED",     # sprawdzaj dopiero na COMMIT
    )
)
```

```python
with engine.begin() as conn:
    # Wstawiamy oba wiersze w dowolnej kolejności — klucze obce sprawdzimy na końcu.
    conn.execute(insert(members).values(id=1, email="a@x.pl", favourite_book_id=10))
    conn.execute(insert(books).values(id=10, isbn="...", title="Solaris"))
# Tu, przy COMMIT, PostgreSQL sprawdzi wszystkie ograniczenia naraz — i przejdzie.
```

> ️ **Pułapka** — `DEFERRABLE` **nie jest przenośne**: docelowo działa na PostgreSQL (i Oracle). SQLite i MySQL je zignorują lub zgłoszą błąd. Poza tym `initially="DEFERRED"` a `initially="IMMEDIATE"` to różne rzeczy: pierwsze sprawdza na końcu transakcji, drugie przy każdym poleceniu, ale pozwala **zmienić** tryb w transakcji przez `SET CONSTRAINTS`. Poza PostgreSQL — i w praktyce w większości projektów — wybierz wariant 1 („wstaw `NULL`, potem `UPDATE`”); jest przenośny i czytelny.

---

## Masowe operacje: trzy podejścia

Wracamy do pytania z sekcji 3 i odpowiadamy na nie konkretnymi liczbami. Załóżmy, że mamy **10 000 wierszy** do wstawienia do `books`.

### Podejście A — pętla z `execute()`

```python
# examples/05_bulk_loop.py
import time

from sqlalchemy import insert

rows = [{"isbn": f"978{i:010d}", "title": f"Książka {i}", "author_id": 1} for i in range(10_000)]

start = time.perf_counter()
with engine.begin() as conn:
    for row in rows:
        conn.execute(insert(books).values(**row))
print(f"A (pętla): {time.perf_counter() - start:.2f}s")
```

Instrukcja SQL leci **10 000 razy**. Każda ma własną rundę, własne planowanie, własny wpis w dzienniku transakcji. Na SQLite lokalnie to jeszcze znośne; przez sieć do PostgreSQL — katastrofa.

### Podejście B — `executemany` (lista parametrów)

```python
# examples/05_bulk_executemany.py
start = time.perf_counter()
with engine.begin() as conn:
    conn.execute(insert(books), rows)
print(f"B (executemany): {time.perf_counter() - start:.2f}s")
```

Sterownik dostaje **jedną instrukcję i 10 000 zestawów parametrów**. Rundy są ograniczone do liczby wewnętrznych partii sterownika. To zwykle 10–50× szybciej niż pętla.

### Podejście C — `insert().values([...])` (jedna instrukcja wielowartościowa)

```python
# examples/05_bulk_multivalues.py
start = time.perf_counter()
with engine.begin() as conn:
    conn.execute(insert(books).values(rows))
print(f"C (multi-values): {time.perf_counter() - start:.2f}s")
```

Jedna instrukcja SQL z 10 000 grup `VALUES`. Najszybciej — ale uwaga na limit parametrów. Przy 3 kolumnach × 10 000 wierszy = 30 000 parametrów, co jest już na granicy SQLite (32700) i blisko granicy PostgreSQL (65535).

### Tabela porównawcza

Poniżej typowe rzędne wielkości dla **10 000 wierszy** na lokalnym SQLite (`sqlite:///test.db`). Traktuj je jako ilustrację rzędu wielkości, nie jako wynik dokładny — Twoje liczby będą inne.

| Podejście | Instrukcji SQL | Rundy do bazy | Czas (względny) | Kiedy używać |
|---|---|---|---|---|
| **A** — pętla `execute()` | 10 000 | ~10 000 | **1,00×** (baseline) | Nigdy do masowych wstawień. Sensowne tylko, gdy każdy wiersz wymaga odrębnej logiki i osobnej obsługi błędu. |
| **B** — `executemany` | 1 (wielokrotnie wykonana) | ~10–100 | **~0,05×** | Gdy potrzebujesz **pominiąć wadliwe wiersze pojedynczo** albo nie potrzebujesz `RETURNING`. Wariant domyślny i najbezpieczniejszy przenośnie. |
| **C** — `insert().values([...])` | 1 | 1 | **~0,02×** | Gdy wszystkie wiersze są poprawne, a tabela nie jest bardzo szeroka. Najszybszy, ale cała partia leci jako jeden atomowy blok — jeden błąd = cała instrukcja odrzucona. |
| **D** — `insertmanyvalues` z `RETURNING` | $\lceil N/1000 \rceil$ | ~10 | **~0,03×** | Gdy potrzebujesz zwrotki kluczy głównych. Automatyczne, gdy użyjesz `.returning()` i podasz listę parametrów. |
| **E** — `COPY` (tylko PostgreSQL) | 1 | 1 | **~0,005×** | Dziesiątki milionów wierszy. Wymaga surowego sterownika (`psycopg` `copy_expert`), nie przechodzi przez SQLAlchemy Core. Temat modułu 17. |

**Wniosek:** sam wybór między A, B i C to różnica rzędu **50×**. To nie jest mikrooptymalizacja — to różnica między „import trwa 2 sekundy” a „import trwa 100 sekund”.

> 🧠 **Dlaczego pętla jest tak wolna** — każda runda do bazy to nie tylko transfer danych. To:
> - przejście przez stos TCP (albo przez syscall, przy SQLite),
> - sparsowanie tekstu instrukcji przez serwer,
> - zaplanowanie wykonania (plan),
> - zajęcie blokad,
> - zapis do dziennika transakcji (WAL),
> - potwierdzenie.
>
> Przy 10 000 rundach powtarzasz to wszystko 10 000 razy dla danych, które zmieściłyby się w jednej paczce. Sama „stała opłata” za rundę jest tu większa niż koszt samego wstawienia.

> ️ **Pułapka** — **nie używaj `insert().values(lista)` do bardzo szerokich tabel**. Przy 30 kolumnach i 10 000 wierszy budujesz instrukcję SQL z 300 000 parametrów. SQLite ma limit 32700 parametrów (`SQLITE_MAX_VARIABLE_NUMBER`), PostgreSQL 65535. Przekroczenie limitu = błąd wykonania. Rozwiązanie: **dziel dane na partie po stronie Pythona** albo użyj `executemany` (podejście B), które partiami zajmuje się samo.
>
> ```python
> def w_partiach(seq, rozmiar):
>     for i in range(0, len(seq), rozmiar):
>         yield seq[i : i + rozmiar]
>
> with engine.begin() as conn:
>     for partia in w_partiach(rows, 1000):
>         conn.execute(insert(books).values(partia))
> ```

> ⚠️ **Pułapka** — kolejna, bardzo podstępna: **`values()` kopiuje struktury przy budowaniu instrukcji, ale lista parametrów przekazana do `execute()` jest odczytywana w momencie wykonania.** Jeżeli po wykonaniu zmodyfikujesz słowniki na liście, nie wpłynie to na już wykonaną instrukcję — ale jeżeli trzymasz **jeden obiekt `Insert`** jako zmienną globalną i budujesz go raz z `values(...)`, a potem używasz wielokrotnie, kolejne wykonania wyślą **te same, zamrożone wartości**. Zawsze buduj instrukcję na nowo albo używaj wariantu „instrukcja + lista parametrów”.

---

## Pełny przykład: import z pliku CSV

Zbierzmy wszystko w jeden uruchamialny program. Założenia:

- mamy plik `books.csv` z kolumnami `isbn,title,author_name,country,published_year,price`,
- chcemy **deduplikować autorów** po nazwisku (jeśli istnieją, użyj ich `id`),
- chcemy **upsertować książki** po `isbn` (jeśli istnieją, zaktualizuj tytuł i cenę),
- chcemy dostać **raport**: ile autorów dodano, ile książek dodano, ile zaktualizowano, ile wierszy pominięto,
- całość w **jednej transakcji**, ale z **savepointem na wiersz**, żeby jeden popsuty rekord nie zepsuł całego importu.

### Wersja 1 — cała transakcja albo nic

Najpierw wersja prosta i bezpieczna: każdy błąd wywraca wszystko.

```python
# examples/05_import_csv_simple.py
"""Import CSV do bazy — wersja "wszystko albo nic"."""

import csv
import sys
from decimal import Decimal
from pathlib import Path

from sqlalchemy import create_engine, insert, select

from examples._schema import authors, books, metadata

DB_URL = "sqlite:///library.db"
CSV_PATH = Path("books.csv")


def wczytaj_wiersze(sciezka: Path) -> list[dict]:
    """Czyta CSV i zwraca listę słowników (wszystkie wartości jako str)."""
    with sciezka.open(newline="", encoding="utf-8") as fh:
        return list(csv.DictReader(fh))


def zbuduj_rekordy(surowe: list[dict]) -> list[dict]:
    """Normalizuje dane z CSV do postaci gotowej dla bazy."""
    rekordy = []
    for w in surowe:
        rekordy.append(
            {
                "isbn": w["isbn"].strip(),
                "title": w["title"].strip(),
                "author_name": w["author_name"].strip(),
                "country": (w.get("country") or "").strip() or None,
                "published_year": int(w["published_year"]) if w.get("published_year") else None,
                # Decimal, nie float! Pieniądze w float to błąd (moduł 12).
                "price": Decimal(w["price"]) if w.get("price") else None,
            }
        )
    return rekordy


def importuj(engine, rekordy: list[dict]) -> dict[str, int]:
    """Wykonuje cały import w JEDNEJ transakcji."""
    liczniki = {"authors_new": 0, "books_new": 0, "books_updated": 0}

    with engine.begin() as conn:                       # ← jedna transakcja
        # Krok 1: cache autorów (name → id). Jedno zapytanie zamiast tysiąca.
        istniejacy: dict[str, int] = dict(
            conn.execute(select(authors.c.name, authors.c.id)).all()
        )

        # Krok 2: autorzy — wstaw tylko tych, których nie ma.
        nowi_autorzy = []
        for r in rekordy:
            if r["author_name"] not in istniejacy:
                nowi_autorzy.append(
                    {"name": r["author_name"], "country": r["country"]}
                )
                istniejacy[r["author_name"]] = -1         # znacznik: "będzie dodany"

        if nowi_autorzy:
            # Deduplikacja wewnątrz partii — to samo nazwisko może się powtórzyć.
            unikalni: dict[str, dict] = {}
            for a in nowi_autorzy:
                unikalni.setdefault(a["name"], a)
            wynik = conn.execute(
                insert(authors).values(list(unikalni.values())).returning(authors.c.id, authors.c.name)
            )
            for row in wynik:
                istniejacy[row.name] = row.id
            liczniki["authors_new"] = len(unikalni)

        # Krok 3: książki — przez upsert po ISBN (SQLite).
        from sqlalchemy.dialects.sqlite import insert as sqlite_insert

        widziane_isbn: set[str] = set()
        do_wstawienia: list[dict] = []
        for r in rekordy:
            if r["isbn"] in widziane_isbn:
                continue                                  # duplikat w pliku — pomiń
            widziane_isbn.add(r["isbn"])
            do_wstawienia.append(
                {
                    "isbn": r["isbn"],
                    "title": r["title"],
                    "author_id": istniejacy[r["author_name"]],
                    "published_year": r["published_year"],
                    "price": r["price"],
                }
            )

        stmt = sqlite_insert(books).values(do_wstawienia)
        stmt = stmt.on_conflict_do_update(
            index_elements=["isbn"],
            set_={"title": stmt.excluded.title, "price": stmt.excluded.price},
        )
        conn.execute(stmt)

    return liczniki


def main() -> int:
    if not CSV_PATH.exists():
        print(f"Brak pliku {CSV_PATH}", file=sys.stderr)
        return 1

    engine = create_engine(DB_URL, echo=False)
    metadata.create_all(engine)

    surowe = wczytaj_wiersze(CSV_PATH)
    rekordy = zbuduj_rekordy(surowe)
    liczniki = importuj(engine, rekordy)

    print("=== Raport importu ===")
    print(f"Wierszy w pliku     : {len(surowe)}")
    print(f"Unikalnych ISBN     : {len({r['isbn'] for r in rekordy})}")
    print(f"Dodani autorzy      : {liczniki['authors_new']}")
    print(f"Książek przetworzono: {len(rekordy)}")
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

Odpowiednik dla PostgreSQL różni się **jedną linią** — importem:

```python
# PostgreSQL:
from sqlalchemy.dialects.postgresql import insert as pg_insert
stmt = pg_insert(books).values(do_wstawienia)
```

### Wersja 2 — savepoint na wiersz

Wersja 1 ma wadę: **jeden popsuty rekord przewraca cały import**. W praktyce dane z plików bywają brudne. Wersja 2 izoluje wiersze savepointami, ale robi to kosztem wydajności — bo wykonuje instrukcję raz na wiersz.

```python
# examples/05_import_csv_savepoints.py
"""Import CSV z savepointem na wiersz — jeden zły rekord nikogo nie zabije."""

import csv
from decimal import Decimal, InvalidOperation
from pathlib import Path

from sqlalchemy import create_engine, insert, select
from sqlalchemy.exc import IntegrityError, SQLAlchemyError

from examples._schema import authors, books, metadata

DB_URL = "sqlite:///library.db"
CSV_PATH = Path("books.csv")


def importuj_odpornie(engine, surowe: list[dict]) -> dict[str, int]:
    """Import wiersz po wierszu, każdy w osobnym savepoincie."""
    liczniki = {"ok": 0, "pominiete": 0, "bledne": 0}

    with engine.begin() as conn:
        # Cache autorów budujemy raz, poza pętlą.
        istniejacy: dict[str, int] = dict(
            conn.execute(select(authors.c.name, authors.c.id)).all()
        )

        for i, w in enumerate(surowe, start=1):
            try:
                with conn.begin_nested():          # ← SAVEPOINT na wiersz
                    # --- walidacja i normalizacja ---
                    isbn = (w.get("isbn") or "").strip()
                    title = (w.get("title") or "").strip()
                    author_name = (w.get("author_name") or "").strip()
                    if not isbn or not title or not author_name:
                        raise ValueError("brak wymaganego pola")

                    try:
                        price = Decimal(w.get("price") or "0")
                    except InvalidOperation as exc:
                        raise ValueError(f"zła cena: {w.get('price')!r}") from exc

                    # --- autor (dopisany w locie, jeśli trzeba) ---
                    author_id = istniejacy.get(author_name)
                    if author_id is None:
                        author_id = conn.execute(
                            insert(authors)
                            .values(name=author_name, country=(w.get("country") or None))
                            .returning(authors.c.id)
                        ).scalar_one()
                        istniejacy[author_name] = author_id

                    # --- książka ---
                    conn.execute(
                        insert(books).values(
                            isbn=isbn,
                            title=title,
                            author_id=author_id,
                            published_year=int(w["published_year"])
                            if w.get("published_year")
                            else None,
                            price=price,
                        )
                    )
                liczniki["ok"] += 1
            except IntegrityError as exc:
                # Konflikt unikalności (ISBN już istnieje) — pomijamy, to nie błąd.
                liczniki["pominiete"] += 1
                print(f"  wiersz {i}: pominięty ({exc.orig})")
            except (ValueError, SQLAlchemyError) as exc:
                # Błąd danych — też pomijamy, ale raportujemy głośniej.
                liczniki["bledne"] += 1
                print(f"  wiersz {i}: BŁĄD ({exc})")

    return liczniki


def main() -> None:
    engine = create_engine(DB_URL, echo=False)
    metadata.create_all(engine)

    with CSV_PATH.open(newline="", encoding="utf-8") as fh:
        surowe = list(csv.DictReader(fh))

    raport = importuj_odpornie(engine, surowe)
    print("=== Raport ===")
    print(f"poprawnie wstawione : {raport['ok']}")
    print(f"pominięte (duplikat): {raport['pominiete']}")
    print(f"błędne dane         : {raport['bledne']}")


if __name__ == "__main__":
    main()
```

### Jak uruchomić

```bash
# 1. Środowisko (jednorazowo)
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate

# 2. Zależności
pip install "SQLAlchemy>=2.0"

# 3. Przykładowy plik CSV
cat > books.csv <<'CSV'
isbn,title,author_name,country,published_year,price
9780156027601,Solaris,Stanisław Lem,PL,1961,54.90
9780898598910,The Left Hand of Darkness,Ursula K. Le Guin,USA,1969,39.90
9780261102354,The Fellowship of the Ring,J.R.R. Tolkien,UK,1954,79.00
9780156027601,Solaris (wyd. 2),Stanisław Lem,PL,1961,59.90
CSV

# 4. Uruchomienie
python -m examples.05_import_csv_simple
python -m examples.05_import_csv_savepoints
```

Oczekiwany wynik drugiego uruchomienia:

```text
=== Raport ===
poprawnie wstawione : 3
pominięte (duplikat): 1
błędne dane         : 0
```

> 🧠 **Którą wersję wybrać?** Reguła jest prosta i wraca w wielu miejscach tego kursu:
> - **Dane zaufane** (eksport z Twojego systemu, generator, inny serwis z kontraktem) → **wersja 1**. Szybko i wszystko albo nic.
> - **Dane niezaufane** (plik od klienta, scraping, import z legacy) → **wersja 2**, z raportem. Wolniej, ale jeden zły wiersz nie wywala 100 000 dobrych.
> - **Złoty środek**: wersja 2 w partiach. Waliduj w Pythonie w pamięci, wrzucaj partie po 500 wierszy, savepoint na **partię** zamiast na wiersz, a odrzucone partie loguj do osobnego pliku do ręcznej analizy.

> 🧪 **Ćwiczenie** — zmodyfikuj wersję 2 tak, żeby używała `begin_nested()` na **partię 500 wierszy**, a nie na pojedynczy wiersz. Zmierz czas obu wersji na 10 000 wierszy.

---

## Podsumowanie

Zabierz ze sobą te punkty — będą wracały w każdym kolejnym module:

1. **`insert()`, `update()`, `delete()` budują instrukcję; `conn.execute()` ją wykonuje.** Rozdzielenie konstrukcji od wykonania pozwala logować, testować i przekazywać instrukcje.
2. **`update()` i `delete()` bez `where()` dotykają całej tabeli.** SQLAlchemy nie ostrzega. Pisz `where()` natychmiast.
3. **`engine.begin()` to domyślny sposób pracy.** Sam robi `BEGIN`, `COMMIT` przy sukcesie i `ROLLBACK` przy wyjątku. Używaj go zawsze, gdy nie masz konkretnego powodu, by sterować transakcją ręcznie.
4. **Autobegin zastąpił autocommit.** W 2.0 każde polecenie żyje w transakcji. `engine.execute()` zostało usunięte — zamiast niego `with engine.begin() as conn:`.
5. **Savepoint (`begin_nested()`) pozwala cofnąć fragment transakcji.** Kluczowe przy importach wiersz po wierszu i przy „pomiń wadliwe rekordy”.
6. **Trzy podejścia do masowych zapisów różnią się ~50×.** Pętla `execute()` → `executemany` (lista parametrów) → `insert().values([...])` (jedna instrukcja). Wybieraj świadomie.
7. **`insertmanyvalues` włącza się automatycznie przy `RETURNING` + lista parametrów.** Przepisuje partię na wielowartościowy `INSERT` i ogranicza liczbę rund do $\lceil N / \text{page\_size} \rceil$ (domyślnie 1000).
8. **`RETURNING` daje zwrotkę bez drugiego zapytania** i działa też przy `UPDATE` i `DELETE`. Kolejność zwracanych wierszy nie jest gwarantowana — jeśli mapujesz na wejście, użyj `sort_by_parameter_order=True`.
9. **`rowcount` bywa `-1`.** To liczba wierszy **dopasowanych**, nie zmienionych. Traktuj jako informację opcjonalną; dokładną liczbę daj `RETURNING`.
10. **Upsert nie jest przenośny.** SQLite/PostgreSQL mają `on_conflict_*`, MySQL `on_duplicate_key_update`, SQL Server/Oracle `MERGE`. Import idempotentny to obowiązek, nie ozdoba.
11. **Domyślnie Django, SQLAlchemy i bazy — trzy warstwy problemów.** Wartości liczone w bazie (`books.c.price * 1.10`) są atomowe i wolne od wyścigów; wartości liczone w Pythonie — nie.
12. **`DEFERRABLE`, `UPDATE ... FROM`, `DELETE ... USING` i wielotabelowe `DELETE` są specyficzne dla dialektu.** Kod przenośny używa podzapytań skorelowanych i `IN (...)`.

---

## Ćwiczenia

### wiczenie 1 — Idempotentny import katalogu (łatwe)

Napisz funkcję `import_authors(engine, rows)` przyjmującą listę słowników `{"name": ..., "country": ...}`. Funkcja ma:

- wstawić brakujących autorów,
- **nie tworzyć duplikatów**, jeśli funkcja zostanie uruchomiona dwa razy z tymi samymi danymi,
- zwrócić słownik `{"inserted": int, "skipped": int}`.

Załóż, że kolumna `authors.name` ma ograniczenie `unique` (dodaj je do schematu). Sprawdź, że drugie uruchomienie zwraca `inserted=0`.

### Ćwiczenie 2 — Miękkie usuwanie (średnie)

Zamiast fizycznie usuwać książki, wprowadź **miękkie usuwanie**: kolumna `deleted_at` (`DateTime`, `nullable=True`).

1. Dodaj kolumnę do tabeli `books` w schemacie (pamiętaj — w prawdziwym projekcie robisz to migracją, moduł 16; tutaj możesz odtworzyć bazę od zera).
2. Napisz funkcję `soft_delete_book(conn, book_id)`, która ustawia `deleted_at = func.now()`.
3. Napisz funkcję `list_books(conn, include_deleted: bool = False)`, która domyślnie **pomija** usunięte książki.
4. Napisz funkcję `purge_deleted(conn, older_than_days: int) -> int`, która **fizycznie** usuwa książki usunięte dawniej niż N dni i zwraca ich liczbę.
5. Wyjaśnij w komentarzu, dlaczego `purge_deleted` nie może opierać się na `rowcount`.

### Ćwiczenie 3 — Import z deduplikacją i raportem jakościowym (trudne)

Rozszerz importer z sekcji 12 tak, aby:

1. Obsługiwał **plik z duplikatami wewnątrz siebie** (ten sam `isbn` dwa razy) — pierwsze wystąpienie wygrywa, drugie trafia do raportu jako `duplicate_in_file`.
2. Obsługiwał **konflikt między plikiem a bazą** — jeśli książka istnieje i jej cena w bazie **jest wyższa** niż w pliku, zaktualizuj; jeśli niższa — pomiń i zalicz do `skipped_lower_price`.
3. Działał w partiach po 500 wierszy, z savepointem na partię: wadliwa partia jest odrzucana w całości, ale reszta importu idzie dalej.
4. Zwracał raport jako `TypedDict` z polami: `inserted`, `updated`, `duplicate_in_file`, `skipped_lower_price`, `rejected_batches`, `rejected_rows`.
5. Po imporcie wypisywał 5 najdroższych książek jakościowo błędnych (odrzuconych) — do ręcznej analizy.

Wskazówka: przy partiach nie da się już użyć jednego `RETURNING` dla całego pliku — rozważ, jak zbudować słownik `isbn → id` dla istniejących książek **jednym** zapytaniem na początku.

### Rozwiązania

#### Rozwiązanie 1 — Idempotentny import katalogu

Najprostsza poprawna droga to `ON CONFLICT DO NOTHING` po unikalnej kolumnie i policzenie, ile wierszy faktycznie powstało przez `RETURNING`.

```python
# solutions/05_ex1.py
"""Idempotentny import autorów — ON CONFLICT DO NOTHING + RETURNING."""

from typing import TypedDict

from sqlalchemy import Column, Integer, MetaData, String, Table, create_engine, insert
from sqlalchemy.dialects.sqlite import insert as sqlite_insert
from sqlalchemy.engine import Engine


class Counters(TypedDict):
    inserted: int
    skipped: int


metadata = MetaData()

authors = Table(
    "authors",
    metadata,
    Column("id", Integer, primary_key=True),
    # unique + NOT NULL: to jest nasz "kontrakt deduplikacji".
    Column("name", String(100), nullable=False, unique=True),
    Column("country", String(50)),
)


def import_authors(engine: Engine, rows: list[dict]) -> Counters:
    """Wstawia brakujących autorów. Drugie uruchomienie nic nie zmienia."""
    if not rows:
        return Counters(inserted=0, skipped=0)

    # Deduplikacja wewnątrz partii — baza i tak by je odrzuciła,
    # ale wtedy nie wiedzielibyśmy, ile z nich było duplikatami wejścia.
    unikalne: dict[str, dict] = {}
    for r in rows:
        unikalne.setdefault(r["name"], r)

    stmt = (
        sqlite_insert(authors)
        .values(list(unikalne.values()))
        .on_conflict_do_nothing(index_elements=["name"])
        .returning(authors.c.id)          # ← zwróci TYLKO faktycznie wstawione
    )

    with engine.begin() as conn:
        wstawione = len(conn.execute(stmt).all())

    return Counters(
        inserted=wstawione,
        skipped=len(rows) - wstawione,
    )
```

**Dlaczego tak, a nie inaczej.** Trzy alternatywy i ich problemy:

- *Pętla `SELECT` + `INSERT`*: działa, ale jest wolna i ma okno wyścigu między odczytem a zapisem. Przy równoległych importach tworzy duplikaty, jeśli nie ma ograniczenia `unique` — a jeśli jest, wywala się wyjątkiem.
- *Pętla `INSERT` + `except IntegrityError`*: dużo rund do bazy i wyjątki jako sterowanie przepływem. Czytelne, ale wolne.
- *`ON CONFLICT DO NOTHING` + `RETURNING`*: jedna instrukcja, atomowa, sama mówi, ile się udało. To wzorzec, który warto zapamiętać raz na zawsze.

**Minipułapka.** `RETURNING` przy `ON CONFLICT DO NOTHING` zwraca **tylko wiersze faktycznie wstawione**. To dokładnie to, czego chcesz, ale łatwo pomylić się w drugą stronę: przy `on_conflict_do_update` `RETURNING` zwróci **wszystkie** wiersze — i te wstawione, i te zaktualizowane. Nie da się ich rozróżnić po wyniku `RETURNING`; do tego trzeba `xmax = 0` (PostgreSQL) albo osobnego znacznika.

#### Rozwiązanie 2 — Miękkie usuwanie

```python
# solutions/05_ex2.py
"""Miękkie usuwanie książek — z purge i uczciwym raportowaniem."""

from datetime import timedelta

from sqlalchemy import (
    Column, DateTime, ForeignKey, Integer, MetaData, Numeric, String, Table,
    delete, func, insert, select, update,
)
from sqlalchemy.engine import Connection

metadata = MetaData()

authors = Table(
    "authors", metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String(100), nullable=False, unique=True),
)

books = Table(
    "books", metadata,
    Column("id", Integer, primary_key=True),
    Column("isbn", String(13), nullable=False, unique=True),
    Column("title", String(200), nullable=False),
    Column("author_id", ForeignKey("authors.id"), nullable=False),
    Column("price", Numeric(8, 2)),
    # NULL = aktywna książka. Data = usunięta (miękko) o tej godzinie.
    Column("deleted_at", DateTime, nullable=True),
)


def soft_delete_book(conn: Connection, book_id: int) -> bool:
    """Oznacza książkę jako usuniętą. Zwraca True, jeśli faktycznie zmieniła stan."""
    stmt = (
        update(books)
        .where(books.c.id == book_id)
        .where(books.c.deleted_at.is_(None))        # ← nie nadpisuj daty drugi raz!
        .values(deleted_at=func.now())
        .returning(books.c.id)
    )
    return conn.execute(stmt).first() is not None


def list_books(conn: Connection, include_deleted: bool = False) -> list[dict]:
    """Lista książek. Domyślnie bez usuniętych."""
    stmt = select(books)
    if not include_deleted:
        stmt = stmt.where(books.c.deleted_at.is_(None))
    return [dict(row._mapping) for row in conn.execute(stmt.order_by(books.c.id))]


def purge_deleted(conn: Connection, older_than_days: int) -> int:
    """Fizycznie usuwa książki usunięte dawniej niż N dni. Zwraca liczbę."""
    granica = func.now() - func.julianday(f"{older_than_days} days")  # patrz komentarz
    # UWAGA (SQLite): powyższe jest uproszczeniem na potrzeby przykładu.
    # Przenośnie i czytelnie lepiej użyć daty liczonej w Pythonie:
    stmt = (
        delete(books)
        .where(books.c.deleted_at.isnot(None))
        .where(books.c.deleted_at < func.datetime("now", f"-{older_than_days} days"))
        .returning(books.c.id)
    )
    return len(conn.execute(stmt).all())
```

**Dlaczego tak, a nie inaczej.**

- `soft_delete_book` ma **dwa** warunki: `id` i `deleted_at IS NULL`. Dzięki temu wywołanie drugi raz nie nadpisuje oryginalnej daty usunięcia. Dodatkowo `RETURNING` mówi nam, czy operacja faktycznie coś zmieniła — `rowcount` mógłby skłamać (pamiętasz: liczy **dopasowane**, nie zmienione).
- `list_books` ma domyślnie wykluczać usunięte. To zasada projektowa: **filtr „aktywności” powinien być domyślny**, a włączenie usuniętych — świadomą decyzją. Odwrotnie prowadzi do wycieków danych w API (moduł 22).
- `purge_deleted` używa `RETURNING` i liczy zwrócone wiersze.

**Minipułapka (i odpowiedź na pytanie z zadania).** `purge_deleted` **nie może** opierać się na `rowcount`, bo `DELETE` z `RETURNING` może — w zależności od sterownika — nie zwracać `rowcount` (dostaniesz `-1`). Co gorsza, `-1` wygląda jak „usunąłem minus jeden wiersz”, więc kod `if rowcount > 0` będzie działał przez przypadek, a `if rowcount == 5` — nigdy. `RETURNING` zwraca pełne wiersze, więc `len(result.all())` jest jednoznaczne i przenośne.

Druga pułapka: `ON DELETE CASCADE`. Jeśli dodasz fizyczne usunięcie książki, a jest do niej podłączona tabela `loans` z `ondelete="CASCADE"` — `purge_deleted` skasuje także historię wypożyczeń. Przy miękkim usuwaniu najczęściej **nie chcesz** kaskady na tabeli historii. Dodaj wtedy klucz obcy **bez** `ondelete`, żeby baza zablokowała purge, gdy istnieje historia.

#### Rozwiązanie 3 — Import z deduplikacją i raportem jakościowym

```python
# solutions/05_ex3.py
"""Import CSV w partiach z savepointem na partię i raportem jakościowym."""

import csv
from collections.abc import Iterator
from decimal import Decimal, InvalidOperation
from pathlib import Path
from typing import TypedDict

from sqlalchemy import create_engine, insert, select
from sqlalchemy.dialects.sqlite import insert as sqlite_insert
from sqlalchemy.engine import Connection, Engine
from sqlalchemy.exc import SQLAlchemyError

from solutions._schema import authors, books, metadata

BATCH_SIZE = 500


class ImportReport(TypedDict):
    inserted: int
    updated: int
    duplicate_in_file: int
    skipped_lower_price: int
    rejected_batches: int
    rejected_rows: list[str]        # opisy odrzuconych rekordów


def partiami(seq: list, rozmiar: int) -> Iterator[list]:
    for i in range(0, len(seq), rozmiar):
        yield seq[i : i + rozmiar]


def _posprzataj(conn: Connection, surowe: list[dict]) -> tuple[list[dict], int]:
    """Walidacja + deduplikacja wewnątrz partii. Zwraca (dobre, liczba duplikatów)."""
    dobre: list[dict] = []
    widziane: set[str] = set()
    duplikaty = 0

    for w in surowe:
        isbn = (w.get("isbn") or "").strip()
        title = (w.get("title") or "").strip()
        author_name = (w.get("author_name") or "").strip()
        if not (isbn and title and author_name):
            raise ValueError(f"brak wymaganego pola w: {w!r}")
        if isbn in widziane:
            duplikaty += 1
            continue
        widziane.add(isbn)

        try:
            price = Decimal(w.get("price") or "0")
        except InvalidOperation as exc:
            raise ValueError(f"zła cena {w.get('price')!r}") from exc

        dobre.append(
            {
                "isbn": isbn,
                "title": title,
                "author_name": author_name,
                "country": (w.get("country") or "").strip() or None,
                "published_year": int(w["published_year"]) if w.get("published_year") else None,
                "price": price,
            }
        )
    return dobre, duplikaty


def import_batches(engine: Engine, surowe: list[dict]) -> ImportReport:
    raport = ImportReport(
        inserted=0, updated=0, duplicate_in_file=0,
        skipped_lower_price=0, rejected_batches=0, rejected_rows=[],
    )

    with engine.begin() as conn:
        # 1) Słowniki istniejących danych — JEDNO zapytanie na cały import.
        autorzy_cache: dict[str, int] = dict(
            conn.execute(select(authors.c.name, authors.c.id)).all()
        )
        ceny_cache: dict[str, Decimal] = dict(
            conn.execute(select(books.c.isbn, books.c.price)).all()
        )

        for numer, partia in enumerate(partiami(surowe, BATCH_SIZE), start=1):
            try:
                with conn.begin_nested():              # ← savepoint na PARTIĘ
                    dobre, duplikaty = _posprzataj(conn, partia)
                    raport["duplicate_in_file"] += duplikaty

                    # 2) Autorzy z partii — tylko brakujący, jednym multi-values INSERT.
                    brakujacy = {
                        r["author_name"]: {"name": r["author_name"], "country": r["country"]}
                        for r in dobre
                        if r["author_name"] not in autorzy_cache
                    }
                    if brakujacy:
                        wynik = conn.execute(
                            insert(authors)
                            .values(list(brakujacy.values()))
                            .returning(authors.c.id, authors.c.name)
                        )
                        for row in wynik:
                            autorzy_cache[row.name] = row.id

                    # 3) Podział książek na "wstaw" i "zaktualizuj".
                    do_wstawienia, do_aktualizacji = [], []
                    for r in dobre:
                        stara_cena = ceny_cache.get(r["isbn"])
                        if stara_cena is None:
                            do_wstawienia.append(r)
                        elif r["price"] > stara_cena:
                            do_aktualizacji.append(r)
                        else:
                            raport["skipped_lower_price"] += 1

                    # 4) Wstawianie — jedno multivalues INSERT na partię.
                    if do_wstawienia:
                        rekordy = [
                            {
                                "isbn": r["isbn"],
                                "title": r["title"],
                                "author_id": autorzy_cache[r["author_name"]],
                                "published_year": r["published_year"],
                                "price": r["price"],
                            }
                            for r in do_wstawienia
                        ]
                        conn.execute(insert(books).values(rekordy))
                        raport["inserted"] += len(rekordy)

                    # 5) Aktualizacje — upsert po ISBN, tylko wyższa cena.
                    if do_aktualizacji:
                        stmt = sqlite_insert(books).values(
                            [
                                {
                                    "isbn": r["isbn"],
                                    "title": r["title"],
                                    "author_id": autorzy_cache[r["author_name"]],
                                    "published_year": r["published_year"],
                                    "price": r["price"],
                                }
                                for r in do_aktualizacji
                            ]
                        )
                        stmt = stmt.on_conflict_do_update(
                            index_elements=["isbn"],
                            set_={
                                "title": stmt.excluded.title,
                                "price": stmt.excluded.price,
                                "published_year": stmt.excluded.published_year,
                            },
                        )
                        conn.execute(stmt)
                        raport["updated"] += len(do_aktualizacji)
                        for r in do_aktualizacji:
                            ceny_cache[r["isbn"]] = r["price"]

            except (ValueError, SQLAlchemyError) as exc:
                # Cała partia leci do raportu. Reszta importu idzie dalej.
                raport["rejected_batches"] += 1
                raport["rejected_rows"].append(f"partia {numer}: {exc}")

    return raport


def main() -> None:
    engine = create_engine("sqlite:///library_import.db")
    metadata.create_all(engine)

    with Path("books.csv").open(newline="", encoding="utf-8") as fh:
        surowe = list(csv.DictReader(fh))

    raport = import_batches(engine, surowe)

    print("=== Raport importu ===")
    for klucz in ("inserted", "updated", "duplicate_in_file", "skipped_lower_price", "rejected_batches"):
        print(f"{klucz:>22}: {raport[klucz]}")

    if raport["rejected_rows"]:
        print("\n--- Odrzucone partie ---")
        for opis in raport["rejected_rows"][:5]:
            print(" ", opis)

    # 5 najdroższych książek z odrzuconych partii — do ręcznej analizy.
    # (Poniżej: przykładowe filtrowanie rekordów, które nie przeszły walidacji.)
    problematyczne: list[tuple[Decimal, str]] = []
    for w in surowe:
        try:
            problematyczne.append((Decimal(w.get("price") or "0"), w.get("title", "?")))
        except (InvalidOperation, TypeError):
            problematyczne.append((Decimal("-1"), w.get("title", "?")))

    print("\n--- 5 najdroższych rekordów wymagających uwagi ---")
    for cena, tytul in sorted(problematyczne, reverse=True)[:5]:
        print(f"  {cena:>8}  {tytul}")


if __name__ == "__main__":
    main()
```

**Dlaczego tak, a nie inaczej.**

- **Partia zamiast wiersza.** Savepoint na wiersz oznaczałby 10 000 savepointów — a każdy to dwie dodatkowe rundy (`SAVEPOINT` + `RELEASE`). Przy 500-wierszowych partiach jest ich 20, a wciąż izolujesz ryzyko: jedna brudna partia nie wywraca importu.
- **Cache zamiast zapytań w pętli.** `autorzy_cache` i `ceny_cache` budowane **raz**, jednym `SELECT` na cały import. To dokładnie odwrotność problemu N+1 (moduł 11) — i największa pojedyncza optymalizacja tego kodu.
- **Rozdzielenie „wstaw” od „aktualizuj”.** Moglibyśmy użyć samego `on_conflict_do_update` na wszystkim, ale wtedy `updated` zawierałby też wiersze, które nie zmieniły ceny. Rozdzielenie daje nam **dokładny raport** i pozwala zastosować warunek biznesowy (`tylko jeśli cena wyższa`).
- **`ON CONFLICT` wewnątrz partii.** Zabezpiecza przed duplikatami w `do_wstawienia`, gdyby cache był nieaktualny. To „pasek i szelki” — ale przy imporcie danych to dobra postawa.

**Minipułapka.** Przy partiach **nie da się użyć `RETURNING` do mapowania wejście↔wyjście**, jeśli równocześnie robisz `ON CONFLICT DO UPDATE` — `RETURNING` zwraca wtedy wszystkie wiersze (i wstawione, i zaktualizowane), bez informacji który jest który. Dlatego `ceny_cache` budujemy z **danych wejściowych**, a nie z odpowiedzi bazy. Jeśli naprawdę musisz rozróżnić insert od update w jednej instrukcji, na PostgreSQL użyj `RETURNING books.id, (xmax = 0) AS was_inserted` — to trik specyficzny dla PostgreSQL, który nie ma odpowiednika w SQLite.

---

## Najczęstsze błędy i jak je czytać

| Komunikat / objaw | Przyczyna | Naprawa |
|---|---|---|
| `sqlalchemy.exc.InvalidRequestError: A transaction is already begun on this Connection` | Wywołano `conn.begin()` na połączeniu, które ma już otwartą transakcję (np. w środku bloku `engine.begin()`). | Użyj `conn.begin_nested()` (savepoint) albo nie otwieraj drugiej transakcji — korzystaj z istniejącej. |
| `sqlalchemy.exc.CompileError: Unconsumed column names: cena` | W `values()` podano nazwę kolumny, której nie ma w tabeli (literówka albo brak kolumny w schemacie). | Sprawdź nazwy w definicji `Table`. `print(tabela.c.keys())` pokaże dostępne. |
| **Skrypt działa bez błędu, ale w bazie nic nie ma** | Użyto `engine.connect()` bez `commit()` — przy wyjściu z bloku następuje `ROLLBACK`. | Używaj `engine.begin()` albo dodaj jawny `trans.commit()`. Sprawdź `echo=True` w logu — brak `COMMIT` to potwierdzenie. |
| `sqlalchemy.exc.IntegrityError: (sqlite3.IntegrityError) UNIQUE constraint failed: books.isbn` | Wstawiasz wiersz z wartością już istniejącą w unikalnej kolumnie. | Użyj `on_conflict_do_nothing` / `on_conflict_do_update`, albo obsłuż wyjątek i pomiń wiersz (wzorzec savepoint). |
| `sqlalchemy.exc.IntegrityError: FOREIGN KEY constraint failed` | Klucz obcy wskazuje na nieistniejący wiersz w tabeli nadrzędnej (albo kolejność wstawiania jest zła). | Wstaw najpierw wiersz nadrzędny, odbierz `id` przez `RETURNING` i dopiero wtedy dziecko. |
| `sqlalchemy.exc.OperationalError: no such table: books` | Tabela nie została utworzona (`metadata.create_all()` nie wywołane albo wywołane na innym silniku/pliku bazy). | Wywołaj `metadata.create_all(engine)`. Sprawdź, czy URL wskazuje ten sam plik — `sqlite:///a.db` i `sqlite:///./a.db` to różne ścieżki. |
| `AttributeError: 'Insert' object has no attribute 'on_conflict_do_nothing'` | Użyto ogólnego `insert()` z `sqlalchemy` zamiast wariantu dialektowego. | Zaimportuj `from sqlalchemy.dialects.sqlite import insert` (albo `postgresql`). |
| `sqlalchemy.exc.ProgrammingError: syntax error at or near "FROM"` przy `UPDATE` | `UPDATE ... FROM` na bazie, która tego nie wspiera (SQLite). | Użyj podzapytania skorelowanego (`scalar_subquery()`) albo `IN (SELECT ...)`. |
| `result.rowcount` zwraca `-1` | Sterownik nie podaje liczby wierszy dla tego typu instrukcji (często przy `executemany` lub `RETURNING`). | Nie polegaj na `rowcount`. Licz zwrócone wiersze przez `RETURNING`. |
| `sqlite3.OperationalError: too many SQL variables` | `insert().values([...])` z partią większą niż limit parametrów (32700 w SQLite, 65535 w PostgreSQL). | Dziel listę na partie po stronie Pythona albo użyj `executemany` (instrukcja + lista parametrów). |
| **Import 10 000 wierszy trwa minutę** | Pętla z `conn.execute()` na każdy wiersz — 10 000 rund do bazy. | Zbierz wiersze w listę i wykonaj jeden `conn.execute(insert(tabela), rows)` albo `insert(tabela).values(rows)`. |
| `sqlalchemy.exc.ResourceClosedError: This result object is closed` | Próba odczytania `Result` po commicie/zamknięciu połączenia. | Odczytaj wynik **przed** wyjściem z bloku `with`, albo użyj `.all()` w środku. |
| `sqlalchemy.exc.PendingRollbackError: This Session's transaction has been rolled back due to a previous exception during flush` | Wyjątek w środku transakcji bez `rollback`. (Dotyczy ORM — zapowiedź modułu 08.) | Otocz operację `engine.begin()`/`try...except...rollback`; w ORM wywołaj `session.rollback()`. |
| `sqlalchemy.exc.NoReferencedTableError: Foreign key associated with column 'books.author_id' could not find table 'authors'` | Definicja `ForeignKey("authors.id")` przed utworzeniem tabeli `authors` w `metadata`. | Zdefiniuj tabelę nadrzędną wcześniej albo użyj `metadata.append_constraint(...)` po zdefiniowaniu obu. |
| **`RETURNING` nie działa na MySQL** | MySQL (nie MariaDB) nie wspiera `RETURNING` przed MySQL 8.0.19 w ograniczonym zakresie, a sterowniki często nie implementują `executemany` z `RETURNING`. | Odbierz `id` z `result.lastrowid` albo użyj sekwencji/`UUID` generowanego po stronie aplikacji. |
| **Wartości w `insert().values(lista)` są „zamrożone” po modyfikacji listy** | `values()` kopiuje dane w momencie budowania instrukcji. | Buduj instrukcję na nowo dla każdej partii, albo używaj wariantu „instrukcja + lista parametrów w `execute()`”. |

---

## Słowniczek modułu

| Termin (EN) | Polski odpowiednik | Wyjaśnienie |
|---|---|---|
| **ACID** | ACID | Cztery gwarancje transakcji: *Atomicity* (wszystko albo nic), *Consistency* (baza przechodzi między poprawnymi stanami), *Isolation* (transakcje nie widzą swoich połowicznych efektów), *Durability* (po commicie dane przeżyją awarię). |
| **autobegin** | automatyczne rozpoczęcie | Mechanizm 2.0: przy pierwszym poleceniu SQLAlchemy sam wysyła `BEGIN`, jeśli żadna transakcja nie trwa. Zastąpił usunięty tryb autocommit z 1.x. |
| **batch / page** | partia / strona | Grupa wierszy wysyłana w jednej instrukcji. `insertmanyvalues_page_size` to rozmiar takiej partii (domyślnie 1000). |
| **commit** | zatwierdzenie | Trwałe zapisanie zmian transakcji w bazie. Po commicie `ROLLBACK` już nie cofnie zmian. |
| **DBAPI** | DBAPI | Standardowy interfejs Pythona do baz danych (PEP 249). `sqlite3`, `psycopg`, `asyncpg` to implementacje DBAPI. |
| **DML** | DML | Język manipulacji danymi: `INSERT`, `UPDATE`, `DELETE`. Zmienia zawartość tabel, nie ich strukturę. |
| **`excluded`** | (pseudo-tabela) | W `ON CONFLICT DO UPDATE` udostępnia wiersz, który próbowano wstawić — pozwala odwołać się do „nowych” wartości. |
| **`executemany`** | wykonanie wielokrotne | Metoda DBAPI wykonująca jedną instrukcję dla listy zestawów parametrów. Jedna runda (lub kilka partii), brak zwrotki przy braku `RETURNING`. |
| **`insertmanyvalues`** | (mechanizm) | Funkcja SQLAlchemy 2.0: przepisuje `INSERT` z `RETURNING` i wieloma parametrami na wielowartościowy `INSERT ... VALUES (...), (...)` w partiach. |
| **idempotentność** | idempotentność | Właściwość operacji: wykonana dwa razy daje ten sam skutek co raz. Kluczowa dla importów i kolejek zadań. |
| **`INSERT ... SELECT`** | wstaw z zapytania | Wstawianie wierszy bezpośrednio z wyniku `SELECT`, bez przechodzenia przez Pythona. `Insert.from_select()`. |
| **klucz obcy (FK)** | klucz obcy | Kolumna wskazująca na wiersz w innej tabeli. Baza pilnuje, że wskazywany wiersz istnieje. |
| **`rowcount`** | liczba wierszy | Liczba wierszy **dopasowanych** przez `UPDATE`/`DELETE`. Może być `-1`, gdy sterownik tego nie podaje. |
| **`RETURNING`** | klauzula zwrotna | Część `INSERT`/`UPDATE`/`DELETE`, która zwraca dane ze zmienionych wierszy w tym samym poleceniu. |
| **`ROLLBACK`** | wycofanie | Cofnięcie całej transakcji. Baza wraca do stanu sprzed `BEGIN`. |
| **savepoint** | punkt kontrolny | Znacznik wewnątrz transakcji (`SAVEPOINT`/`ROLLBACK TO SAVEPOINT`), na który można się cofnąć bez utraty wcześniejszych zmian. `begin_nested()`. |
| **upsert** | wstaw-lub-aktualizuj | Operacja „jeśli wiersz istnieje, zaktualizuj; jeśli nie, wstaw”. Składnia zależy od dialektu. |
| **transakcja** | transakcja | Grupa poleceń wykonująca się jako nierozdzielna całość. |
| **WAL** | dziennik zapisu (Write-Ahead Log) | Mechanizm bazy zapisujący zmiany najpierw do dziennika, a potem do plików danych. Powód, dla którego `COMMIT` jest szybki, a `UPDATE` hurtowy wolny. |
| **wide row / narrow row** | wiersz szeroki / wąski | Wiersz szeroki = dużo kolumn lub dużo danych tekstowych. Wpływa na limity parametrów i rozmiar instrukcji. |
| **`DEFERRABLE`** | odroczone | Atrybut ograniczenia: sprawdzane na końcu transakcji, a nie przy każdym poleceniu. PostgreSQL/Oracle. |

---

## Dalsze czytanie

- **Tutorial — INSERT** (podstawy wstawiania, `RETURNING`, wiele parametrów): <https://docs.sqlalchemy.org/en/20/tutorial/data_insert.html>
- **Tutorial — UPDATE i DELETE** (aktualizacje skorelowane, `UPDATE..FROM`, `rowcount`, `RETURNING`): <https://docs.sqlalchemy.org/en/20/tutorial/data_update.html>
- **Core — Insert, Updates, Deletes** (pełna dokumentacja konstruktów, w tym `from_select`, `ordered_values`, `sort_by_parameter_order`): <https://docs.sqlalchemy.org/en/20/core/dml.html>
- **Silniki i połączenia** — sekcja o „Insert Many Values”, kontroli partii i transakcjach: <https://docs.sqlalchemy.org/en/20/core/connections.html>
- **Dialekt SQLite — `ON CONFLICT`** (upsert, `index_elements`, `index_where`): <https://docs.sqlalchemy.org/en/20/dialects/sqlite.html>
- **Dialekt PostgreSQL — `INSERT ... ON CONFLICT`** (upsert, `excluded`, `constraint=`): <https://docs.sqlalchemy.org/en/20/dialects/postgresql.html>
- **Ograniczenia** — `ForeignKey` z `ondelete`, `deferrable`, `initially`: <https://docs.sqlalchemy.org/en/20/core/constraints.html>
- **Migracja 1.4 → 2.0** — co zastąpiło `engine.execute()` i tryb autocommit: <https://docs.sqlalchemy.org/en/20/changelog/migration_20.html>
- **🆕 SQLAlchemy 2.1 — `Delete.using()` i pozostałe nowości**: <https://docs.sqlalchemy.org/en/21/changelog/migration_21.html>
- ** SQLAlchemy 2.1 — dokumentacja DML**: <https://docs.sqlalchemy.org/en/21/core/dml.html>

---

## Co dalej

Nauczyłeś się zapisywać dane bezpiecznie (transakcje, savepointy), wydajnie (masowe operacje, `insertmanyvalues`) i odpornie (upsert, `RETURNING`). Umiesz już wykonać każdą pojedynczą operację na jednej tabeli — i wiesz, jak zrobić to sto tysięcy razy bez czekania.

Brakuje jednak najtrudniejszej części SQL-a: **łączenia danych z wielu tabel**. W module 06 poznamy `JOIN`, aliasy, podzapytania, wyrażenia CTE (w tym rekurencyjne — czyli drzewa i hierarchie) oraz funkcje okna. Tam też okaże się, dlaczego warunek w `ON` a warunek w `WHERE` przy `LEFT JOIN` to dwie zupełnie różne instrukcje — i jak to jedno rozróżnienie zmienia cały wynik raportu.

➡️ **[`06_joiny_i_zaawansowane_sql.md`](06_joiny_i_zaawansowane_sql.md)** — JOIN-y, podzapytania, CTE i funkcje okna.
⬅️ **[`04_select_i_result.md`](04_select_i_result.md)** — `select()`, filtry, agregacje i obiekt `Result`.

<!-- koniec modułu 05 -->