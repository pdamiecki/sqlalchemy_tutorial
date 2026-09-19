# Moduł 04 — Czytanie danych: `select()` i obiekt `Result`

Ten moduł jest o najczęstszej czynności, jaką wykonuje każdy program korzystający z bazy danych: **czytaniu danych**. Nauczysz się budować zapytania `SELECT` za pomocą `select()`, dowiesz się, czym różni się zapytanie (opis tego, co chcesz dostać) od wyniku (to, co naprawdę przyszło z bazy), poznasz obiekt `Result` i jego liczne metody odczytu oraz zrozumiesz, dlaczego SQLAlchemy nigdy nie wkleja Twoich danych wprost do tekstu SQL-a — i dlaczego to bardzo dobra wiadomość. Wszystko w warstwie **Core**, czyli bez ORM-a, bez klas i bez magii: tylko tabele, kolumny i SQL.

---

> **Poziom:** 🟢 podstawowy
> **Czas:** ~150 minut
> **Wymagania wstępne:** [`03_metadata_ddl.md`](03_metadata_ddl.md) — musisz umieć zdefiniować `MetaData`, `Table` i `Column`; [`02_srodowisko_i_engine.md`](02_srodowisko_i_engine.md) — `create_engine()`, `engine.connect()`, `engine.begin()`.
> **Czego dotyczy ten plik:** budowa zapytań `select()`, filtrowanie, sortowanie, stronicowanie, agregacje i grupowanie, obiekt `Result`, wiersze (`Row`), parametry wiązane, praca z `Connection` i `Engine`.

---

## Spis treści

- [Problem: jak czytać dane z bazy](#problem-jak-czytać-dane-z-bazy)
- [Anatomia zapytania SELECT](#anatomia-zapytania-select)
- [Budowanie select()](#budowanie-select)
- [execute() i obiekt Result](#execute-i-obiekt-result)
- [Row i RowMapping: wiersz z etykietami](#row-i-rowmapping-wiersz-z-etykietami)
- [Filtrowanie: where() i operatory](#filtrowanie-where-i-operatory)
- [Sortowanie i stronicowanie](#sortowanie-i-stronicowanie)
- [Funkcje, agregacje i grupowanie](#funkcje-agregacje-i-grupowanie)
- [Parametry wiązane](#parametry-wiązane)
- [Literały i surowy SQL](#literały-i-surowy-sql)
- [Warstwy wykonawcze: Connection i Engine](#warstwy-wykonawcze-connection-i-engine)
- [Przykład kompletny: katalog produktów](#przykład-kompletny-katalog-produktów)
- [Podsumowanie](#podsumowanie)
- [Ćwiczenia](#ćwiczenia)
- [Najczęstsze błędy i jak je czytać](#najczęstsze-błędy-i-jak-je-czytać)
- [Słowniczek modułu](#słowniczek-modułu)
- [Dalsze czytanie](#dalsze-czytanie)
- [Co dalej](#co-dalej)

---

## Problem: jak czytać dane z bazy

Zanim pokażemy `select()`, zobaczmy, co dzieje się bez niego. To ćwiczenie z cyklu „najpierw problem, potem rozwiązanie”.

Masz tabelę `products` (produkty) z kolumnami `id`, `sku`, `name`, `category_id`, `price`, `stock`. Chcesz wyświetlić wszystkie aktywne produkty z kategorii 2, droższe niż 150 zł, posortowane od najdroższego. W najprostszym możliwym świecie Pythona napisałbyś tak:

```python
# examples/04_problem.py (wersja PROBLEMOWA — nie rób tak)
import sqlite3

category_id = 2
min_price = 150

connection = sqlite3.connect("shop.db")
cursor = connection.cursor()

sql = (
    "SELECT id, sku, name, price FROM products "
    f"WHERE category_id = {category_id} AND price > {min_price} "
    "AND is_active = 1 ORDER BY price DESC"
)
cursor.execute(sql)

for row in cursor.fetchall():
    print(row[1], row[2], row[3])   # co to jest row[3]? Trzeba pamiętać kolejność!
```

Ten kod ma cztery problemy i każdy z nich zniknie, gdy tylko przejdziemy na `select()`.

**Problem pierwszy: tekst SQL-a sklejany z danych.** Jeśli `category_id` pochodzi od użytkownika (np. z adresu URL), a użytkownik wpisze `0 OR 1=1`, to Twój warunek `WHERE` przestaje filtrować cokolwiek. Gorzej — wpisanie `0; DROP TABLE products; --` kończy się usunięciem tabeli. To jest **wstrzyknięcie SQL-a** (SQL injection) i jest to jeden z najczęstszych i najbardziej kosztownych błędów w historii informatyki.

**Problem drugi: wynik to krotki bez nazw.** `row[3]` to cena tylko dlatego, że pamiętasz kolejność kolumn w `SELECT`. Gdy ktoś zmieni kolejność w zapytaniu, Twój kod wyświetli cenę jako nazwę produktu — i nie zgłosi przy tym żadnego błędu.

**Problem trzeci: sterownik.** Kod napisany dla `sqlite3` nie zadziała dla PostgreSQL-a. Trzeba przepisać wszystkie wywołania, składnię placeholderów (`?` kontra `%s`) i sposób łączenia się z bazą.

**Problem czwarty: brak kompozycji.** Chcesz dodać filtr „tylko produkty z opisem”? Budujesz string metodą `+=`, robisz się warunkowy `if`, a po miesiącu nikt nie potrafi tego przeczytać ani przetestować.

SQLAlchemy rozwiązuje wszystkie cztery problemy jednocześnie: zapytanie jest **obiektem Pythona**, do którego składasz warunki metodami, dane idą **osobno od tekstu SQL-a**, wynik ma **nazwane kolumny**, a ten sam kod działa na SQLite, PostgreSQL i MySQL.

> 💡 **Analogia — zapytanie jako formularz, nie jako list.** Wyobraź sobie, że zamawiasz meble. Możesz napisać do stolarni list odręczny (sklejany SQL) — jeśli w środku zdania wpiszesz coś nieoczekiwanego, stolarz może źle zrozumieć całą treść. Albo możesz wypełnić **formularz zamówienia** z polami: „kolor”, „wymiary”, „liczba sztuk”. Stolarnia czyta pola, nie interpretuje Twoich zdań. SQLAlchemy to taki formularz: Ty wypełniasz pola, a biblioteka buduje z nich zamówienie (SQL) i wysyła je do bazy.

> 🧠 **Dlaczego tak jest — dwie warstwy: budowa i wykonanie.** W SQLAlchemy budowanie zapytania i jego wykonanie to dwa zupełnie różne etapy. `select(products).where(...)` **nic nie robi** — zwraca obiekt opisujący zapytanie. Dopiero `conn.execute(...)` wysyła ten opis do bazy. Ta rozdzielczość jest kluczowa: możesz zapytanie zbudować, sprawdzić, wypisać, zapisać do logu, przekazać dalej — bez dotykania bazy.

---

## Anatomia zapytania SELECT

Jeśli nie znasz SQL-a, nie szkodzi: `SELECT` to instrukcja „pokaż mi dane”, a jej części czytają się jak zdanie w języku angielskim. Oto pełna lista klauzul, których będziemy używać — i ich odpowiedniki w `select()`:

| SQL | Po polsku | W SQLAlchemy Core |
|---|---|---|
| `SELECT kolumny` | które kolumny chcę dostać | argumenty `select(...)` |
| `FROM tabela` | skąd | automatycznie, na podstawie kolumn |
| `WHERE warunek` | które wiersze | `.where(...)` |
| `GROUP BY kolumny` | po czym grupować | `.group_by(...)` |
| `HAVING warunek` | filtr grup | `.having(...)` |
| `ORDER BY kolumny` | w jakiej kolejności | `.order_by(...)` |
| `LIMIT n` | ile wierszy | `.limit(n)` |
| `OFFSET m` | ile wierszy pominąć | `.offset(m)` |
| `DISTINCT` | bez duplikatów | `.distinct()` |

Zobaczmy to samo zapytanie zapisane dwa razy — raz jako tekst SQL, raz jako obiekt SQLAlchemy:

```python
# examples/04_anatomia.py
from sqlalchemy import Integer, MetaData, String, Table, Column, select

metadata = MetaData()
products = Table(
    "products",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("sku", String(32), nullable=False),
    Column("name", String(120), nullable=False),
    Column("category_id", Integer, nullable=False),
    Column("price", Integer, nullable=False),
    Column("stock", Integer, nullable=False),
)

# Wersja 1: tekst SQL (dla porównania — tak NIE piszemy w aplikacji)
sql_text = """
SELECT products.sku, products.name, products.price
FROM products
WHERE products.category_id = 2
  AND products.price > 150
ORDER BY products.price DESC
LIMIT 10
"""

# Wersja 2: to samo zapytanie jako obiekt SQLAlchemy
stmt = (
    select(products.c.sku, products.c.name, products.c.price)   # SELECT ... FROM
    .where(products.c.category_id == 2, products.c.price > 150)  # WHERE
    .order_by(products.c.price.desc())                           # ORDER BY
    .limit(10)                                                   # LIMIT
)

# SQLAlchemy potrafi wypisać SQL, który powstanie z tego obiektu:
print(stmt)
```

> 🔬 **Pod maską.** `print(stmt)` wypisuje dokładnie to, co poleci do bazy (kolumny w jednej linii, kolejne klauzule w następnych — to standardowy, „ładny” format SQLAlchemy 2.0):
>
> ```sql
> SELECT products.sku, products.name, products.price
> FROM products
> WHERE products.category_id = 2 AND products.price > 150
> ORDER BY products.price DESC
> LIMIT 10
> ```
>
> Zwróć uwagę, że **nie musieliśmy podawać tabeli** w `FROM`. SQLAlchemy wie, że kolumny `products.c.sku` pochodzą z tabeli `products`, i zbudował `FROM products` samodzielnie. Gdy użyjesz kolumn z dwóch tabel, obie trafią do `FROM` (a w praktyce do `JOIN` — o tym w module 06).

> 💡 **Analogia — budowanie zapytania to rysowanie planu.** Obiekt `stmt` to plan wycieczki: „jedziemy z `products`, zatrzymujemy się tam, gdzie kategoria to 2, sortujemy po cenie”. Plan możesz pokazać komuś, poprawić, odłożyć do szuflady. `conn.execute(stmt)` to moment, w którym faktycznie wsiadasz w samochód. Ten sam plan możesz „odpalić” na SQLite i na PostgreSQL — zmienia się tylko mapa drogowa (dialekt), nie plan.

> ⚠️ **Pułapka — `.filter()` kontra `.where()`.** Spotkasz w internecie przykłady używające `.filter(...)`. Ta metoda **istnieje** w SQLAlchemy 2.0 (jako relikt z wersji 1.x i dla kompatybilności z ORM-em), ale w stylu 2.0 używamy `.where(...)`. Powód jest praktyczny: `select()` jest wspólnym API dla Core i ORM, a `.where()` to nazwa spójna z klauzulą `WHERE` w SQL-u. Jeśli piszesz nowy kod — zawsze `.where()`.

> 🆕 **SQLAlchemy 2.1 — domyślny sterownik PostgreSQL.** W 2.0 zapis `postgresql://` domyślnie wybierał sterownik `psycopg2`. W 2.1 domyślnym sterownikiem dla `postgresql://` jest **`psycopg` (wersja 3)**. Jeśli Twój projekt opiera się na starym sterowniku, w 2.1 musisz go wskazać jawnie, np. `postgresql+psycopg2://`. Ten moduł działa na SQLite, więc różnica Cię nie dotknie — ale warto ją znać, jeśli uruchamiasz te same zapytania na PostgreSQL-u.

> 🧪 **Ćwiczenie — w miejscu.** Zapisz powyższe zapytanie bez `.limit(10)`, ale z dodatkowym warunkiem „tylko produkty aktywne” (`is_active == True`). Wypisz SQL i sprawdź, czy warunki połączyły się spójnikiem `AND`.

---

## Budowanie select()

`select()` przyjmuje dowolną liczbę argumentów opisujących **co chcesz dostać**. Możesz podać:

- **całą tabelę** — `select(products)`. Wtedy wynik zawiera wszystkie kolumny tej tabeli.
- **wybrane kolumny** — `select(products.c.sku, products.c.name)`. Mniej danych po sieci, szybsze zapytanie, mniejsza pamięć.
- **wyrażenia** — `select(func.count())`, `select(products.c.price * 1.23)`.
- **nic sensownego z tabeli** — `select(1)`, `select(func.now())`. Tak, to legalne i czasem potrzebne (np. żeby sprawdzić, czy baza odpowiada).

```python
# examples/04_budowa_select.py
from sqlalchemy import MetaData, Table, Column, Integer, String, func, literal, select

metadata = MetaData()
products = Table(
    "products",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("sku", String(32), nullable=False),
    Column("name", String(120), nullable=False),
    Column("price", Integer, nullable=False),
)

# 1. Cała tabela
print(select(products))
# SELECT products.id, products.sku, products.name, products.price
# FROM products

# 2. Tylko dwie kolumny
print(select(products.c.sku, products.c.name))
# SELECT products.sku, products.name
# FROM products

# 3. Wyrażenie zamiast kolumny (cena z VAT-em)
print(select(products.c.sku, (products.c.price * 1.23).label("price_gross")))
# SELECT products.sku, products.price * 1.23 AS price_gross
# FROM products

# 4. Bez żadnej tabeli — sprawdzenie, czy baza odpowiada
print(select(1))
# SELECT 1
```

Zwróć uwagę na punkt 3: wyrażenie nie ma nazwy, więc dostaje ją od nas przez `.label()`. Bez tego SQLAlchemy wymyśli nazwę sam — i będzie to coś w rodzaju `anon_1`, co jest zupełnie bezużyteczne dla czytelnika wyników.

> 💡 **Analogia — `.label()` to etykieta na pudełku.** Baza danych zwraca kolumny „bez nazwisk”, a raczej z nazwami generowanymi przez silnik bazy. `.label("price_gross")` to naklejka, którą sam przyklejasz na pudełko: „tu jest cena brutto”. Dzięki niej odwołasz się do wartości po nazwie (`row.price_gross`), a nie po pozycji (`row[1]`).

### Zapytanie bez tabeli — po co?

`select(1)` to najprostszy sposób na sprawdzenie, że połączenie z bazą działa, zanim wykonasz właściwe zapytanie. Na SQLite i PostgreSQL-u renderuje się dokładnie tak: `SELECT 1`. Są jednak bazy, które nie pozwalają na `SELECT` bez `FROM` — Oracle wymaga tabeli systemowej `dual`. SQLAlchemy wie o tym i **dla dialektu Oracle dopisuje `FROM DUAL` automatycznie**. Ty piszesz `select(1)` i nie martwisz się o szczegóły dialektu.

```python
# Sprawdzenie „czy baza żyje” — bez tabel, bez danych
print(select(func.now().label("server_time")))
# SQLite:       SELECT CURRENT_TIMESTAMP AS server_time
# PostgreSQL:   SELECT now() AS server_time
```

> 🧠 **Dlaczego tak jest — `func.now()` to nie Python, to SQL.** `func.now()` **nie wywołuje** żadnej funkcji Pythona. Tworzy obiekt reprezentujący wywołanie funkcji w bazie danych. Jego treść zależy od dialektu: SQLite nie zna funkcji `now()`, więc dialekt tłumaczy ją na `CURRENT_TIMESTAMP`; PostgreSQL wysyła `now()`. To kolejny dowód na to, że SQLAlchemy to tłumacz — Ty mówisz „teraz”, a on tłumaczy na język konkretnej bazy.

> 🆕 **SQLAlchemy 2.1 — lepsze typowanie wyników (PEP 646).** W 2.1 typy generyczne `Result` i `Row` zostały przepisane z użyciem składni `TypeVarTuple` z PEP 646. Praktyczny skutek: edytor i `mypy` dokładniej wiedzą, ile kolumn zwraca Twoje zapytanie i jakiego typu jest każda z nich. W 2.0 typowanie było tu zgrubne (często `Row[Any]`), więc jeśli pracujesz na 2.1, zobaczysz lepsze podpowiedzi w IDE bez żadnej zmiany w kodzie.

> 🧪 **Ćwiczenie — w miejscu.** Zbuduj zapytanie zwracające `sku`, nazwę oraz cenę zaokrągloną w górę (`func.ceil(products.c.price)`). Każdej kolumnie nadaj czytelną etykietę i wypisz SQL.

---

## execute() i obiekt Result

Zbudowaliśmy zapytanie. Teraz trzeba je wykonać. Robimy to na połączeniu:

```python
# examples/04_execute.py
from sqlalchemy import create_engine, select
# ... (definicja tabeli products — jak wyżej)

engine = create_engine("sqlite:///shop.db")

with engine.connect() as conn:
    result = conn.execute(select(products.c.sku, products.c.name))
    for row in result:
        print(row.sku, row.name)
```

W tym krótkim fragmencie dzieją się trzy ważne rzeczy:

1. `engine.connect()` otwiera **połączenie** (ang. connection) do bazy — pojedynczy „kabel” do rozmowy z bazą danych. Po wyjściu z bloku `with` połączenie wraca do puli.
2. `conn.execute(stmt)` **kompiluje** zapytanie do SQL-a właściwego dla dialektu, wysyła je do sterownika (DBAPI), a następnie do bazy.
3. Wynik to obiekt **`Result`** — kursor, który czyta wiersze.

> 💡 **Analogia — `Result` to kran, nie wiadro.** Wyobraź sobie, że baza danych to zbiornik wody, a `Result` to odkręcony kran. Woda (wiersze) leci dopiero wtedy, gdy podstawiasz naczynie. Możesz podstawić małą szklankę (`fetchmany(10)`), wiadro (`all()`) albo czerpać jedną kroplę na raz (iteracja). I — co ważne — **kran nie przewija się do początku**: co wypłynęło, to wypłynęło.

### Najważniejsze metody obiektu Result

| Metoda | Co zwraca | Kiedy używać |
|---|---|---|
| `all()` | lista wszystkich wierszy | gdy wyników jest mało i chcesz je wszystkie przetworzyć |
| `first()` | pierwszy wiersz albo `None` | gdy szukasz „jednego, obojętnie którego” |
| `one()` | dokładnie jeden wiersz | gdy brak lub nadmiar wierszy to błąd (asercja niezmiennika) |
| `one_or_none()` | jeden wiersz albo `None` | gdy 0 wierszy jest w porządku, ale 2 już nie |
| `scalar()` | pierwsza kolumna pierwszego wiersza | gdy interesuje Cię jedna wartość |
| `scalars()` | strumień pierwszych kolumn | gdy wybierasz jedną kolumnę z wielu wierszy |
| `mappings()` | strumień słowników | gdy chcesz przekazać dane dalej jako dict |
| `keys()` | nazwy kolumn | gdy budujesz dynamiczny kod (np. eksport do CSV) |
| `fetchmany(n)` | do `n` wierszy | przetwarzanie partiami |
| `partitions(n)` | iteracja po partiach po `n` wierszy | eksport porcjami, praca z dużymi zbiorami |
| `unique()` | wiersze bez duplikatów (ORM) | przy eager loadingu kolekcji (moduł 11) |
| `close()` | nic (zamyka kursor) | gdy przerywasz czytanie wcześniej |

Zobaczmy je w akcji na jednym, konkretnym wyniku:

```python
# examples/04_result_api.py
from sqlalchemy import create_engine, select
# ... (definicja tabeli products)

engine = create_engine("sqlite:///shop.db")
stmt = select(products.c.sku, products.c.name).order_by(products.c.sku)

with engine.connect() as conn:
    # 1. keys() — nazwy kolumn wyniku
    result = conn.execute(stmt)
    print(result.keys())            # RMKeyView(['sku', 'name'])

    # 2. all() — wszystkie wiersze jako lista obiektów Row
    rows = result.all()
    print(rows[0])                  # ('BOK-001', 'Clean Code')
    print(len(rows))                # 12

    # 3. UWAGA: powyższy `result` jest już zużyty. Każdy kolejny odczyt
    #    wymaga ŚWIEŻEGO wyniku — dlatego wykonujemy zapytanie ponownie.
    result = conn.execute(stmt)
    print(result.first())           # ('BOK-001', 'Clean Code')
    print(result.first())           # ('BOK-002', 'Python Cookbook') — kursor idzie dalej!

    # 4. one_or_none() — dokładnie jeden wiersz albo nic
    result = conn.execute(stmt.where(products.c.sku == "GRY-003"))
    print(result.one_or_none())     # ('GRY-003', 'Wingspan')

    # 5. scalar() — pierwsza kolumna pierwszego wiersza
    result = conn.execute(select(products.c.price).where(products.c.sku == "GRY-003"))
    print(result.scalar())          # Decimal('229.00')

    # 6. scalars() — strumień wartości z jednej kolumny
    result = conn.execute(select(products.c.sku).order_by(products.c.sku))
    print(list(result.scalars())[:3])   # ['BOK-001', 'BOK-002', 'BOK-003']

    # 7. mappings() — wiersze jako słowniki
    result = conn.execute(stmt.limit(2))
    print(result.mappings().all())      # [{'sku': 'BOK-001', 'name': 'Clean Code'}, ...]

    # 8. partitions() — porcje po 5 wierszy
    result = conn.execute(stmt)
    for chunk in result.partitions(5):
        print(len(chunk))               # 5, 5, 2
```

> ⚠️ **Pułapka — `Result` to strumień, nie lista.** `Result` czyta wiersze z kursora bazy danych. Każdy odczyt **zabiera** wiersze z kursora, więc nie możesz przejść po nim dwa razy. Jeśli chcesz przetwarzać dane wielokrotnie, zrób to jawnie: `rows = result.all()` i dalej pracuj na zwykłej liście. To nie jest ograniczenie SQLAlchemy, tylko natura baz danych — silnik bazy nie trzyma dla Ciebie całego wyniku „na boku” (przy dużych zbiorach byłoby to koszmarnie drogie).

> 🧠 **Dlaczego tak jest — `one()` rzuca wyjątki i to jest jego zaleta.** `one()` wymaga **dokładnie jednego** wiersza. Jeśli baza zwróci zero wierszy, dostaniesz `NoResultFound`; jeśli dwa — `MultipleResultsFound`. Brzmi to jak utrudnianie życia, ale jest odwrotnie: to **asercja niezmiennika**. Przykład: pobierasz użytkownika po adresie e-mail, który w bazie ma ograniczenie `UNIQUE`. Jeśli `one()` kiedykolwiek zwróci dwa wiersze, to znaczy, że w systemie dzieje się coś niemożliwego — i lepiej, żeby program krzyknął od razu, niż po cichu przetworzył pierwszy z brzegu rekord. Wybieraj metodę odczytu zależnie od tego, **co w Twojej domenie jest błędem**: „brak wyniku” czy „więcej niż jeden wynik”.

> ⚠️ **Pułapka — `scalar()` kontra `scalars()`.** Jedna litera, dwa różne światy:
> - `scalar()` — **jedna wartość**: pierwsza kolumna z pierwszego wiersza. Zwraca np. `Decimal('229.00')`.
> - `scalars()` — **strumień wartości**: pierwsza kolumna z **każdego** wiersza. Zwraca obiekt `ScalarResult`, po którym iterujesz albo robisz `.all()`.
>
> Mylenie ich to najczęstszy błąd początkujących, bo obie nazwy brzmią podobnie. Zapamiętaj: liczba mnoga = wiele wierszy.

> 🧪 **Ćwiczenie — w miejscu.** Wykonaj `select(products.c.price)` (bez `where`) i sprawdź, co zwróci `scalar()`, a co `scalars().all()`. Dlaczego `scalar()` zwraca cenę tylko jednego produktu?

---

## Row i RowMapping: wiersz z etykietami

Wróćmy do drugiego problemu z naszego „problemowego” kodu: `row[3]`. SQLAlchemy zwraca wiersze jako obiekty **`Row`**.

`Row` zachowuje się jednocześnie jak **krotka** i jak **obiekt z nazwanymi polami**. Możesz więc pisać tak:

```python
# examples/04_row.py
with engine.connect() as conn:
    result = conn.execute(select(products.c.sku, products.c.name, products.c.price))
    row = result.first()

    # dostęp po nazwie — czytelnie i odpornie na zmiany kolejności
    print(row.sku)          # BOK-001
    print(row.name)         # Clean Code
    print(row.price)        # Decimal('129.00')

    # dostęp po indeksie — jak w krotce (działa, ale jest mniej czytelny)
    print(row[0], row[2])   # BOK-001 Decimal('129.00')

    # rozpakowanie jak krotkę
    sku, name, price = row
    print(f"{sku}: {name} — {price}")

    # surowa krotka
    print(row.t)            # ('BOK-001', 'Clean Code', Decimal('129.00'))
```

> 💡 **Analogia — Row to wiersz z etykietami na segregatorze.** Wyobraź sobie teczkę, w której każda kartka ma przyklejoną etykietę („sku”, „nazwa”, „cena”). Możesz wyciągnąć kartkę po numerze (trzecia kartka) albo po etykiecie („kartka z ceną”). Pierwszy sposób działa, dopóki nikt nie przełoży kartek. Drugi jest odporny na zmiany i sam się tłumaczy.

### RowMapping — gdy naprawdę potrzebujesz słownika

Czasem wiersz trzeba przekazać jako zwykły `dict`: do szablonu HTML, do JSON-a, do `csv.DictWriter`, do biblioteki, która nie zna SQLAlchemy. Służy do tego `row._mapping` — widok wiersza w postaci mapy:

```python
# examples/04_row_mapping.py
with engine.connect() as conn:
    result = conn.execute(select(products.c.sku, products.c.name, products.c.price))
    row = result.first()

    mapping = row._mapping          # RowMapping — zachowuje się jak słownik
    print(mapping["sku"])           # BOK-001
    print(list(mapping.keys()))     # ['sku', 'name', 'price']
    print(dict(mapping))            # {'sku': 'BOK-001', 'name': 'Clean Code', ...}
```

Masz też do dyspozycji `result.mappings()`, które zwraca **strumień** `RowMapping` — czyli te same słowniki, ale dla wszystkich wierszy, bez ręcznego opakowywania każdego z osobna:

```python
with engine.connect() as conn:
    result = conn.execute(select(products.c.sku, products.c.price).limit(2))
    for row in result.mappings():
        print(row["sku"], row["price"])   # dostęp jak w słowniku
```

> 🧠 **Dlaczego tak jest — nazwy kolumn pochodzą z zapytania, nie z bazy.** Klucze w `Row` i `RowMapping` biorą się z tego, co podałeś w `select()`. Jeśli wybierzesz `products.c.sku`, klucz nazywa się `sku`. Jeśli wybierzesz wyrażenie bez etykiety, klucz będzie wygenerowany (`anon_1`). Jeśli użyjesz `.label("cena_brutto")`, klucz to `cena_brutto`. Dlatego w kodzie aplikacji **prawie zawsze nadawaj etykiety wyrażeniom** — to Ty decydujesz, jak nazywają się dane w Twoim programie.

> ⚠️ **Pułapka — kolizja nazw kolumn.** Jeśli wybierzesz kolumny o tej samej nazwie z dwóch tabel (np. `products.id` i `orders.id`), SQLAlchemy musi jakoś je rozróżnić i wygeneruje nazwy typu `id_1`. Kod `row.id` zadziała dla pierwszej kolumny, ale druga będzie dostępna tylko jako `row.id_1` — co jest kruche. Rozwiązanie: nadaj etykiety (`products.c.id.label("product_id")`). Zobaczysz to w praktyce w module 06 przy łączeniu tabel.

> 🧪 **Ćwiczenie — w miejscu.** Pobierz `products.c.sku` i `func.length(products.c.name).label("name_length")` dla trzech pierwszych produktów i wypisz wynik jako listę słowników (`dict(...)` z każdego `RowMapping`).

---

## Filtrowanie: where() i operatory

`where()` przyjmuje warunki. Możesz podać ich wiele — SQLAlchemy połączy je spójnikiem `AND`:

```python
stmt = select(products.c.sku, products.c.price).where(
    products.c.category_id == 2,
    products.c.price > 150,
)
```

> 🔬 **Pod maską.**
>
> ```sql
> SELECT products.sku, products.price
> FROM products
> WHERE products.category_id = 2 AND products.price > 150
> ```
>
> Dwa warunki w jednym `where()` = `AND`. Nie musisz pisać `and_()` — wystarczy przecinek.

### Tabela operatorów

| Chcesz powiedzieć | Piszesz | SQL |
|---|---|---|
| równa się | `col == 5` | `col = 5` |
| nie równa się | `col != 5` | `col != 5` |
| większe / mniejsze | `col > 5`, `col < 5` | `col > 5`, `col < 5` |
| większe lub równe | `col >= 5` | `col >= 5` |
| jedna z listy | `col.in_([1, 2, 3])` | `col IN (1, 2, 3)` |
| nie z listy | `col.not_in([1, 2])` | `col NOT IN (1, 2)` |
| zawiera fragment | `col.like("%abc%")` | `col LIKE '%abc%'` |
| zawiera, ignorując wielkość liter | `col.ilike("%abc%")` | PostgreSQL: `col ILIKE '%abc%'` |
| w zakresie | `col.between(10, 20)` | `col BETWEEN 10 AND 20` |
| jest puste | `col.is_(None)` | `col IS NULL` |
| nie jest puste | `col.is_not(None)` | `col IS NOT NULL` |
| spełnia oba warunki | `and_(a, b)` | `a AND b` |
| spełnia którykolwiek | `or_(a, b)` | `a OR b` |
| zaprzeczenie | `not_(a)` | `NOT a` |

### Puste wartości: `is_(None)`, nie `== None`

To najczęstsza pułapka w całym SQL-u, więc powiemy to bardzo wyraźnie.

```python
# ŹLE — to nie zadziała tak, jak myślisz
stmt = select(products.c.sku).where(products.c.description == None)   # noqa: E711

# DOBRZE
stmt = select(products.c.sku).where(products.c.description.is_(None))
```

> 🧠 **Dlaczego tak jest — NULL to nie wartość, to brak wartości.** W SQL-u `NULL` oznacza „nie wiem / nie podano”. Porównanie „nie wiem = nie wiem” nie daje `True` — daje „nie wiem”. Dlatego w SQL-u istnieje osobny operator `IS NULL`, a SQLAlchemy ma na to metodę `.is_(None)`. Jeśli napiszesz `== None`, SQLAlchemy jest na tyle uprzejme, że **zamieni to na `IS NULL`** (i wyemituje ostrzeżenie), ale w kodzie wygląda to jak porównanie dwóch wartości, czym nie jest. Pisz `.is_(None)`.

### `and_`, `or_` i pułapka priorytetu operatorów

Gdy warunek jest złożony, używaj `and_()` i `or_()` — są jednoznaczne:

```python
# examples/04_and_or.py
from sqlalchemy import and_, or_, select

# (kategoria 2 i cena < 200) LUB (kategoria 3 i stan magazynowy > 3)
stmt = (
    select(products.c.sku, products.c.name)
    .where(
        or_(
            and_(products.c.category_id == 2, products.c.price < 200),
            and_(products.c.category_id == 3, products.c.stock > 3),
        )
    )
    .order_by(products.c.sku)
)
```

> 🔬 **Pod maską.** SQLAlchemy sam dodaje nawiasy tam, gdzie są potrzebne — możesz spać spokojnie:
>
> ```sql
> SELECT products.sku, products.name
> FROM products
> WHERE (products.category_id = 2 AND products.price < 200)
>    OR (products.category_id = 3 AND products.stock > 3)
> ORDER BY products.sku
> ```

Alternatywą są operatory `&` (AND) i `|` (OR). Działają, ale mają w Pythonie **wyższy priorytet niż `==`**, co prowadzi do klasycznego, mylącego błędu:

```python
# BŁĄD — to nie znaczy „kategoria = 2 AND aktywny = 1”
stmt = select(products).where(products.c.category_id == 2 & products.c.is_active == 1)
# Python czyta to jako łańcuch porównań:
#   (category_id == (2 & is_active)) and ((2 & is_active) == 1)
# i kończy się wyjątkiem:
# TypeError: Boolean value of this clause is not defined

# POPRAWNIE — nawiasy wokół każdego warunku
stmt = select(products).where(
    (products.c.category_id == 2) & (products.c.is_active == 1)   # noqa: E712
)
```

> ⚠️ **Pułapka — brak nawiasów przy `&` i `|`.** Komunikat `TypeError: Boolean value of this clause is not defined` jest jednym z najczęściej zgłaszanych przez początkujących. Nie oznacza, że SQLAlchemy „nie umie” porównać — oznacza, że Python spróbował zamienić obiekt SQL na `True`/`False` (co się dzieje w łańcuchu porównań). Lekarstwo jest zawsze takie samo: **nawiasy wokół każdego warunku** albo — jeszcze lepiej — `and_()` i `or_()`.

### `in_()` z listą — i pusta lista

```python
# examples/04_in.py
stmt = select(products.c.sku, products.c.name).where(
    products.c.sku.in_(["BOK-001", "MUZ-004"])
)
```

> 🔬 **Pod maską — uwaga, tu dzieje się coś ciekawego.** `print(stmt)` **nie pokaże** gotowej listy wartości, bo SQLAlchemy nie zna jej jeszcze w momencie kompilacji tekstu:
>
> ```sql
> SELECT products.sku, products.name
> FROM products
> WHERE products.sku IN (__[POSTCOMPILE_sku_1])
> ```
>
> Ten tajemniczy znacznik `__[POSTCOMPILE_...]` oznacza: „w tym miejscu wstawię parametry **po** skompilowaniu, na etapie wykonania”. Gdy zapytanie faktycznie poleci do bazy (np. z `echo=True`), zobaczysz:
>
> ```sql
> SELECT products.sku, products.name FROM products WHERE products.sku IN (?, ?)
> ```
>
> A gdy wymusisz podstawienie wartości do tekstu, dostaniesz pełną wersję:
>
> ```python
> print(stmt.compile(compile_kwargs={"literal_binds": True}))
> # WHERE products.sku IN ('BOK-001', 'MUZ-004')
> ```
>
> To mechanizm **expanding parameters** — dzięki niemu SQLAlchemy może użyć **jednego** skompilowanego zapytania dla list o różnych długościach, a baza może je zapamiętać (cache planów zapytań).

> ⚠️ **Pułapka — pusta lista w `in_()`.** `col.in_([])` renderuje się jako `col IN (NULL)`, a to warunek, który **nigdy nie jest prawdziwy** (porównanie z `NULL` daje „nie wiem”). Brzmi jak drobiazg, ale w praktyce: funkcja buduje listę identyfikatorów dynamicznie, lista bywa pusta, a użytkownik dostaje pusty ekran zamiast błędu. Jeśli lista może być pusta, obsłuż ten przypadek **jawnie** — albo zwróć pusty wynik bez pytania bazy, albo pomiń ten filtr w ogóle.

> 🧪 **Ćwiczenie — w miejscu.** Zbuduj zapytanie zwracające `sku` i `name` produktów, których `category_id` jest w `[1, 3]` **oraz** cena nie jest pusta. Wypisz SQL.

---

## Sortowanie i stronicowanie

Sortowanie to `.order_by()`. Możesz podać wiele kolumn — kolejność ma znaczenie, bo druga kolumna rozstrzyga remisy w pierwszej.

```python
# examples/04_order_by.py
from sqlalchemy import desc, nulls_last, select

# Najdroższe najpierw; przy równych cenach — alfabetycznie po SKU
stmt = (
    select(products.c.sku, products.c.price, products.c.stock)
    .order_by(desc(products.c.price), products.c.sku)
)
```

> 🔬 **Pod maską.**
>
> ```sql
> SELECT products.sku, products.price, products.stock
> FROM products
> ORDER BY products.price DESC, products.sku
> ```

> 🧠 **Dlaczego tak jest — dodawaj drugą kolumnę sortującą.** Jeśli sortujesz tylko po cenie, a dwie pozycje mają cenę 129,00, baza ma prawo zwrócić je w dowolnej kolejności — i ta kolejność może się zmienić między jednym zapytaniem a drugim (np. po dodaniu indeksu albo po aktualizacji statystyk). To klasyczna przyczyna „niestabilnych testów”. Druga kolumna w `ORDER BY` (np. `sku`, `id`) czyni wynik **deterministycznym**. To szczególnie ważne przy stronicowaniu — bez pełnego sortowania strona 1 i strona 2 mogą zawierać ten sam wiersz.

### NULL-e na końcu

W SQL-u `NULL` przy sortowaniu trafia zwykle na początek (bazy traktują brak wartości jako „najmniejszą”). Jeśli chcesz odwrotnie, użyj `nulls_last()`:

```python
stmt = (
    select(products.c.sku, products.c.description)
    .order_by(nulls_last(products.c.description))
)
# ORDER BY products.description ASC NULLS LAST
```

> 🧠 **Dlaczego tak jest — dialekty różnią się obsługą `NULLS LAST`.** PostgreSQL i SQLite (od wersji 3.30, rok 2019) rozumieją frazę `NULLS LAST` wprost. Starsze bazy jej nie znają, więc SQLAlchemy **emuluje** ją wyrażeniem `CASE`, które „przestawia” wiersze z `NULL`-em na koniec. Ty piszesz jedno `nulls_last()` i nie interesuje Cię, jak dialekt to rozwiąże.

### Stronicowanie: `limit` i `offset`

```python
# examples/04_pagination.py
page_size = 4
page_number = 2           # strony numerujemy od 1

stmt = (
    select(products.c.sku, products.c.price)
    .order_by(products.c.sku)
    .limit(page_size)
    .offset((page_number - 1) * page_size)
)
```

> 🔬 **Pod maską.**
>
> ```sql
> SELECT products.sku, products.price
> FROM products
> ORDER BY products.sku
> LIMIT 4 OFFSET 4
> ```

> ⚠️ **Pułapka — `OFFSET` jest kosztowny, a stronicowanie po numerze strony gubi dane.** `OFFSET 100000` nie znaczy „przeskocz do wiersza 100000”. Znaczy: „znajdź i **odrzuć** sto tysięcy wierszy, a potem pokaż następne”. Im dalej od początku, tym wolniej. Drugi problem jest subtelniejszy: jeśli w międzyczasie ktoś doda produkt, strony „przesuwają się” i część rekordów możesz zobaczyć dwukrotnie albo pominąć. Wydajniejsze i stabilniejsze jest stronicowanie po kluczu (ang. keyset pagination): `WHERE sku > :ostatni_sku ORDER BY sku LIMIT 4`. Wrócimy do tego w module 20.
>
> Uwaga dialektu: **SQLite nie potrafi użyć `OFFSET` bez `LIMIT`**. Gdy podasz sam `offset()`, dialekt SQLite dopisuje `LIMIT -1`, co dla tej bazy oznacza „bez limitu”.

### `DISTINCT` — bez duplikatów

```python
# examples/04_distinct.py
# Jakie kategorie mają jakiekolwiek produkty? (bez powtórzeń)
stmt = select(products.c.category_id).distinct().order_by(products.c.category_id)
# SELECT DISTINCT products.category_id FROM products ORDER BY products.category_id
# wynik: [1, 2, 3]
```

PostgreSQL ma dodatkowo `DISTINCT ON` — „pierwszy wiersz z każdej grupy”. W SQLAlchemy zapiszesz to, przekazując kolumny do `.distinct()`:

```python
from sqlalchemy.dialects import postgresql

stmt = (
    select(products.c.category_id, products.c.name, products.c.price)
    .distinct(products.c.category_id)          # DISTINCT ON (category_id)
    .order_by(products.c.category_id, desc(products.c.price))
)
print(stmt.compile(dialect=postgresql.dialect()))
# SELECT DISTINCT ON (products.category_id) products.category_id, products.name, products.price
# FROM products
# ORDER BY products.category_id, products.price DESC
```

To najdroższy produkt w każdej kategorii — jednym zapytaniem. **Uwaga: `DISTINCT ON` to składnia PostgreSQL-a**; na SQLite i MySQL-u taki zapis nie zadziała (tam trzeba innego podejścia, np. podzapytania — moduł 06).

> 🆕 **SQLAlchemy 2.1 — `fetch()` jako alternatywa dla `limit()`.** Metoda `.fetch(n)` renderuje standardową klauzulę SQL `FETCH FIRST n ROWS ONLY` (obsługiwaną przez Oracle, PostgreSQL i MSSQL). Uwaga: `.fetch()` **zastępuje** `.limit()`, a nie dodaje się do niego — nie używaj obu w jednym zapytaniu. Dla SQLite i MySQL pozostań przy `.limit()`.

> 🧪 **Ćwiczenie — w miejscu.** Zbuduj zapytanie o trzecią stronę wyników (po 3 produkty na stronę) posortowanych rosnąco po cenie, z `sku` jako kolumną rozstrzygającą remisy. Wypisz SQL i sprawdź, które produkty zwróci.

---

## Funkcje, agregacje i grupowanie

**Agregacja** to operacja, która zamienia wiele wierszy w jedną wartość: suma, średnia, liczba, minimum, maksimum. W SQL-u robią to funkcje agregujące, w SQLAlchemy — `func.<nazwa>`.

> 💡 **Analogia — agregacja to podsumowanie klasy.** Masz dziennik z 30 ocenami uczniów. Agregacja to pytanie „jaka jest średnia w klasie?” — z 30 liczb robi się jedna. Ale uwaga: **nie da się jednocześnie pokazać 30 nazwisk i jednej średniej** w tym samym wierszu wynikowym — chyba że pogrupujesz uczniów (np. po klasach) i policzysz średnią dla każdej grupy. Dokładnie to robi `GROUP BY`.

```python
# examples/04_aggregates.py
from sqlalchemy import func, select

stmt = select(
    func.count().label("liczba_produktow"),
    func.avg(products.c.price).label("srednia_cena"),
    func.min(products.c.price).label("cena_min"),
    func.max(products.c.price).label("cena_max"),
    func.sum(products.c.stock).label("suma_stanow"),
).select_from(products)
```

> 🔬 **Pod maską.**
>
> ```sql
> SELECT count(*) AS liczba_produktow,
>        avg(products.price) AS srednia_cena,
>        min(products.price) AS cena_min,
>        max(products.price) AS cena_max,
>        sum(products.stock) AS suma_stanow
> FROM products
> ```
>
> Na naszych danych wynik to: 12 produktów, średnia cena 161,62 zł, najtańszy 109,00 zł, najdroższy 229,00 zł, łącznie 49 sztuk w magazynie.

Zwróć uwagę na `.select_from(products)`. Bez niego SQLAlchemy nie wie, z jakiej tabeli liczyć — bo `func.count()` nie zawiera żadnej kolumny, z której można by wywnioskować `FROM`. Gdy w zapytaniu są wyłącznie agregacje, `select_from()` jest obowiązkowe.

> ⚠️ **Pułapka — `count(*)` kontra `count(kolumna)`.** `func.count()` bez argumentu liczy **wiersze** (`count(*)`), także te, w których kolumny są puste. `func.count(products.c.description)` liczy tylko wiersze, w których `description` **nie jest NULL-em**. Różnica bywa ogromna i cicho psuje raporty. Jeśli chcesz policzyć wiersze — używaj `func.count()`; jeśli chcesz policzyć „ile razy podano wartość” — `func.count(kolumna)`.

### `GROUP BY` i `HAVING`

Grupowanie dzieli wiersze na kubełki i liczy agregaty osobno dla każdego. `HAVING` to filtr **na grupach** (w odróżnieniu od `WHERE`, który filtruje **wiersze przed grupowaniem**).

```python
# examples/04_group_by.py
from sqlalchemy import func, select

stmt = (
    select(
        categories.c.name.label("kategoria"),
        func.count(products.c.id).label("liczba"),
        func.avg(products.c.price).label("srednia_cena"),
        func.sum(products.c.stock).label("suma_stanow"),
    )
    .select_from(products.join(categories, products.c.category_id == categories.c.id))
    .group_by(categories.c.name)
    .having(func.avg(products.c.price) > 130)
    .order_by(func.avg(products.c.price).desc())
)
```

> 🔬 **Pod maską.**
>
> ```sql
> SELECT categories.name AS kategoria,
>        count(products.id) AS liczba,
>        avg(products.price) AS srednia_cena,
>        sum(products.stock) AS suma_stanow
> FROM products JOIN categories ON products.category_id = categories.id
> GROUP BY categories.name
> HAVING avg(products.price) > 130
> ORDER BY avg(products.price) DESC
> ```
>
> Wynik (dwie kategorie przeszły filtr):
>
> | kategoria | liczba | srednia_cena | suma_stanow |
> |---|---|---|---|
> | Games | 4 | 196.50 | 14 |
> | Books | 4 | 164.37 | 23 |
>
> Kategoria „Music” (średnia 124,00 zł) została odrzucona przez `HAVING`.

> 🧠 **Dlaczego tak jest — `GROUP BY` powtarza wyrażenia, zamiast używać aliasów.** Zauważyłeś, że w `GROUP BY` i `ORDER BY` SQLAlchemy wypisał `avg(products.price)`, a nie `srednia_cena` z `SELECT`? To celowe. Nie wszystkie bazy pozwalają odwoływać się w `GROUP BY` do aliasów z listy `SELECT`, więc SQLAlchemy powtarza wyrażenie — dzięki temu ten sam kod działa wszędzie. Nie przejmuj się „brzydotą” wynikowego SQL-a: baza i tak optymalizuje to wewnętrznie.

**Dygresja — co to jest `JOIN`?** W zapytaniu powyżej połączyliśmy dwie tabele: `products` (zawiera `category_id`, czyli liczbę) i `categories` (zawiera `name`, czyli nazwę). `JOIN` to instrukcja „sklej wiersze obu tabel tam, gdzie `products.category_id` wskazuje na `categories.id`”. Bez tego nie moglibyśmy pokazać nazwy kategorii — w tabeli produktów mamy tylko jej identyfikator. Temu tematowi poświęcimy cały moduł 06; tutaj potrzebowaliśmy go, żeby wynik był czytelny dla człowieka.

> ⚠️ **Pułapka — kolumna w `SELECT`, której nie ma w `GROUP BY`.** Jeśli wybierzesz `products.c.name` razem z `func.count()`, a nie zgrupujesz po nazwie, baza nie wie, którą nazwę pokazać dla grupy. PostgreSQL i standard SQL **zgłoszą błąd**; SQLite i MySQL (w domyślnym trybie) **zwrócą losową wartość z grupy** — i to jest gorsze, bo błąd przejdzie niezauważony. Reguła: każda kolumna w `SELECT`, która nie jest agregatem, musi być w `GROUP BY`.

### `case()` — warunki w środku zapytania

`case()` to SQL-owy odpowiednik `if`/`elif`/`else` — ale wykonywany **w bazie**, dla każdego wiersza.

```python
# examples/04_case.py
from sqlalchemy import case, select

price_bucket = case(
    (products.c.price >= 200, "premium"),
    (products.c.price >= 150, "standard"),
    else_="budget",
).label("polka_cenowa")

stmt = (
    select(products.c.sku, products.c.price, price_bucket)
    .order_by(desc(products.c.price), products.c.sku)
)
```

> 🔬 **Pod maską.**
>
> ```sql
> SELECT products.sku, products.price,
>        CASE WHEN (products.price >= 200) THEN 'premium'
>             WHEN (products.price >= 150) THEN 'standard'
>             ELSE 'budget' END AS polka_cenowa
> FROM products
> ORDER BY products.price DESC, products.sku
> ```
>
> Wynik (pierwsze wiersze): `GRY-003 / 229.00 / premium`, `GRY-004 / 209.00 / premium`, `BOK-003 / 199.99 / standard`, …

`case()` przyjmuje pary `(warunek, wartość)` — kolejność ma znaczenie, bo pierwszy spełniony warunek wygrywa. `else_` to gałąź „w pozostałych przypadkach”; jeśli ją pominiesz, niepasujące wiersze dostaną `NULL`.

> 💡 **Analogia — `case()` to segregacja na taśmie.** Wiersze jadą taśmą i każdy trafia do pierwszego kubełka, którego warunek pasuje. Dlatego kolejność warunków jest jak kolejność sit: od najostrzejszego do najłagodniejszego.

### `cast()` i `extract()`

`cast()` zmienia typ wartości w bazie, `extract()` wyciąga część daty.

```python
# examples/04_cast_extract.py
from sqlalchemy import Integer, cast, extract, select

stmt = (
    select(
        products.c.sku,
        products.c.price,
        cast(products.c.price, Integer).label("cena_calkowita"),
        extract("year", products.c.created_at).label("rok_dodania"),
    )
    .order_by(products.c.created_at)
    .limit(3)
)
```

> 🔬 **Pod maską (dialekt SQLite).**
>
> ```sql
> SELECT products.sku, products.price,
>        CAST(products.price AS INTEGER) AS cena_calkowita,
>        CAST(STRFTIME('%Y', products.created_at) AS INTEGER) AS rok_dodania
> FROM products
> ORDER BY products.created_at
> LIMIT 3
> ```
>
> **SQLite nie ma funkcji `EXTRACT`** — dialekt SQLAlchemy tłumaczy ją na `STRFTIME('%Y', ...)` z rzutowaniem na liczbę całkowitą. Na PostgreSQL-u ten sam kod wygeneruje `EXTRACT(year FROM products.created_at)`.

> ⚠️ **Pułapka — `cast()` na SQLite obcina, nie zaokrągla.** `CAST(199.99 AS INTEGER)` daje `199`, nie `200`. Rzutowanie w SQL-u to nie zaokrąglanie. Jeśli potrzebujesz zaokrąglenia, użyj `func.round(..., 2)` albo — jeszcze lepiej — zaokrąglij w Pythonie i w bazie trzymaj wartości dokładne.

> ⚠️ **Pułapka — `Numeric` na SQLite to nie prawdziwy typ dziesiętny.** SQLite nie ma typu `DECIMAL`. SQLAlchemy obsługuje `Numeric(10, 2)`, ale pod spodem zapisuje liczby zmiennoprzecinkowe i przy zapisie `Decimal`-a wyświetla ostrzeżenie:
>
> ```
> SAWarning: Dialect sqlite+pysqlite does *not* support Decimal objects natively,
> and SQLAlchemy must convert from floating point - rounding errors and other
> issues may occur.
> ```
>
> W praktyce oznacza to: **nie licz pieniędzy na SQLite** — albo trzymaj kwoty jako liczby całkowite w groszach (`Integer`), albo pracuj na PostgreSQL-u, gdzie `NUMERIC(10,2)` jest dokładny. Odczyt z kolumny `Numeric` na SQLite zwraca `Decimal`, ale wartość przeszła przez `float`, więc może różnić się od oczekiwanej w dalekich miejscach po przecinku.

> 🧪 **Ćwiczenie — w miejscu.** Zbuduj zapytanie grupujące produkty po roku dodania (`extract("year", created_at)`) i liczące, ile produktów dodano w każdym roku. Wypisz SQL i wynik.

---

## Parametry wiązane

Wróćmy do problemu wstrzyknięcia SQL-a. Rozwiązanie nazywa się **parametr wiązany** (ang. bind parameter): zamiast wklejać wartość do tekstu SQL-a, wysyłamy ją **osobno**, a w tekście zostaje znacznik.

W SQLAlchemy nie musisz o tym myśleć — parametry wiązane powstają automatycznie:

```python
# examples/04_bind.py
from sqlalchemy import select

min_price = 150
stmt = select(products.c.sku, products.c.name).where(products.c.price > min_price)

# Kompilacja bez podstawiania wartości pokazuje parametry:
print(stmt.compile(compile_kwargs={"literal_binds": False}))
# SELECT products.sku, products.name
# FROM products
# WHERE products.price > :price_1
```

> 🔬 **Pod maską — tak to wygląda na poziomie sterownika.** Gdy zapytanie faktycznie leci do bazy (z włączonym `echo=True`), zobaczysz dwie linie: tekst SQL-a ze znacznikami oraz listę wartości przekazanych osobno:
>
> ```
> 2026-09-18 12:00:00,000 INFO sqlalchemy.engine.Engine SELECT products.sku, products.name
> FROM products
> WHERE products.price > ?
> 2026-09-18 12:00:00,000 INFO sqlalchemy.engine.Engine [generated in 0.00012s] (150,)
> ```
>
> Znak `?` to znacznik sterownika SQLite; PostgreSQL używa `%s`, inne sterowniki — `:nazwa`. **Wartość `150` nigdy nie stała się częścią tekstu SQL-a.** Nawet gdyby w zmiennej `min_price` znalazł się napis `0 OR 1=1`, baza potraktuje go jako *wartość do porównania z ceną*, a nie jako fragment kodu SQL. Wstrzyknięcie jest niemożliwe — to nie kwestia ostrożności, to kwestia architektury.

### Parametry w `text()` — surowy SQL też jest bezpieczny

Gdy piszesz surowy SQL przez `text()`, używaj składni `:nazwa` i przekazuj wartości w drugim argumencie `execute()`:

```python
# examples/04_text_params.py
from sqlalchemy import text

stmt = text(
    "SELECT sku, name, price FROM products "
    "WHERE category_id = :cat AND price >= :min_price "
    "ORDER BY price"
)

with engine.connect() as conn:
    for row in conn.execute(stmt, {"cat": 2, "min_price": 150}):
        print(f"{row.sku:<10} {row.price:>8.2f}  {row.name}")
# GRY-002    159.00  Azul
# GRY-001    189.00  Catan
# GRY-004    209.00  Ticket to Ride
# GRY-003    229.00  Wingspan
```

Możesz też nadać parametrowi typ i zachowanie, używając `bindparam()`:

```python
# examples/04_bindparam.py
from sqlalchemy import bindparam, select

# Parametr „rozszerzalny”: jedna wartość w zapytaniu, wiele wartości w liście
stmt = select(products.c.sku, products.c.name).where(
    products.c.sku.in_(bindparam("skus", expanding=True))
)

with engine.connect() as conn:
    rows = conn.execute(stmt, {"skus": ["BOK-001", "GRY-002"]}).all()
    print(len(rows))    # 2
```

> 🧠 **Dlaczego tak jest — `expanding=True` mówi SQLAlchemy „ta lista urośnie”.** Bez tego znacznika SQLAlchemy przygotowałby jedno miejsce na wartość, a Ty próbowałbyś wcisnąć tam listę. Z `expanding=True` biblioteka wie, że ma rozwinąć `IN (?)` w `IN (?, ?)` — dokładnie tyle razy, ile elementów ma lista.

### Wykonywanie tego samego zapytania wielokrotnie: `executemany`

Gdy przekażesz do `execute()` **listę słowników** zamiast jednego, SQLAlchemy użyje mechanizmu `executemany` — jednego przygotowanego zapytania i wielu zestawów parametrów:

```python
# examples/04_executemany.py
from sqlalchemy import text

with engine.begin() as conn:
    conn.execute(
        text("UPDATE products SET stock = :stock WHERE sku = :sku"),
        [
            {"sku": "BOK-001", "stock": 11},
            {"sku": "GRY-001", "stock": 4},
        ],
    )
```

> 🔬 **Pod maską — log z `echo=True`.**
>
> ```
> BEGIN (implicit)
> UPDATE products SET stock = ? WHERE sku = ?
> [generated in 0.00011s] [(11, 'BOK-001'), (4, 'GRY-001')]
> COMMIT
> ```
>
> Zwróć uwagę: **jedno** polecenie i **jedna** lista wartości. To oszczędza rundy do bazy — zamiast dwóch osobnych rozmów z serwerem jest jedna. Przy tysiącach aktualizacji różnica jest rzędu dziesiątek razy. (Wstawianie danych omawiamy w module 05 — tam SQLAlchemy ma jeszcze sprytniejszą optymalizację `insertmanyvalues` dla `INSERT ... RETURNING`.)

> ⚠️ **Pułapka — `executemany` nie służy do czytania danych.** Możesz przekazać listę parametrów do zapytania `SELECT`, ale wynik takiego wykonania jest zależny od sterownika (w praktyce dostaniesz rezultat ostatniego wykonania). Jeśli chcesz przeczytać dane dla wielu wartości, użyj `IN` z `expanding=True` albo podzapytania. `executemany` to narzędzie do **zapisywania**.

> 🧪 **Ćwiczenie — w miejscu.** Napisz skrypt, który w jednej transakcji (`engine.begin()`) aktualizuje ceny trzech produktów o +5%, podając dane jako listę słowników. Włącz `echo=True` i sprawdź w logu, czy SQLAlchemy użył `executemany`.

---

## Literały i surowy SQL

Czasem trzeba wstawić do zapytania wartość, która **nie jest parametrem** — na przykład stałą nazwę kolumny albo fragment wyrażenia. Do tego służą `literal()`, `literal_column()` i `text()`. To narzędzia ostre jak brzytwa: użyteczne, ale wymagające ostrożności.

| Narzędzie | Co wstawia | Bezpieczne dla danych użytkownika? |
|---|---|---|
| `literal(wartość)` | wartość jako parametr wiązany | ✅ tak |
| `literal_column("1 + 1")` | **dosłowny tekst SQL-a** | ❌ nie |
| `text("fragment SQL")` | **dosłowny tekst SQL-a** | ❌ nie (chyba że użyjesz `:parametrów`) |

```python
# examples/04_literals.py
from sqlalchemy import literal, literal_column, select

stmt = select(
    literal("stała wartość").label("etykieta"),
    literal_column("1 + 1").label("dwa"),
    (products.c.price * 1.23).label("cena_brutto"),
)
```

> 🔬 **Pod maską.**
>
> ```sql
> SELECT 'stała wartość' AS etykieta, 1 + 1 AS dwa, products.price * 1.23 AS cena_brutto
> FROM products
> ```
>
> `literal()` również tworzy parametr wiązany — to, że widzisz `'stała wartość'` w tekście, wynika z użycia `literal_binds`. `literal_column("1 + 1")` natomiast jest wklejone **dosłownie**.

### Dlaczego nie wolno składać SQL-a przez f-string

Zobaczmy, jak wygląda prawdziwe wstrzyknięcie i dlaczego jest tak groźne:

```python
# examples/04_injection.py (DEMONSTRACJA ATAKU — nigdy tak nie pisz!)
from sqlalchemy import text

def wyszukaj_zle(conn, fraza: str) -> list:
    # ŹLE: fraza ląduje wprost w tekście SQL-a
    sql = f"SELECT sku, name FROM products WHERE name = '{fraza}'"
    return conn.execute(text(sql)).all()

# Użytkownik wpisuje w wyszukiwarce:
fraza = "' OR 1=1 --"
# Powstaje SQL:
#   SELECT sku, name FROM products WHERE name = '' OR 1=1 --'
# Warunek 1=1 jest zawsze prawdziwy → dostajesz CAŁĄ tabelę,
# a `--` komentuje resztę zapytania, więc nic już nie ogranicza wyniku.
```

A teraz wersja bezpieczna — ta sama funkcja, ale z parametrem wiązanym:

```python
# examples/04_injection_safe.py
from sqlalchemy import text

def wyszukaj_dobrze(conn, fraza: str) -> list:
    # DOBRZE: fraza jest parametrem, nie fragmentem kodu SQL
    stmt = text("SELECT sku, name FROM products WHERE name = :fraza")
    return conn.execute(stmt, {"fraza": fraza}).all()
```

> 🧠 **Dlaczego tak jest — baza rozróżnia kod od danych.** W `text()` z parametrem `:fraza` baza dostaje dwa osobne elementy: „plan wykonania zapytania” i „wartość do wstawienia w oznaczone miejsce”. Wartość nigdy nie jest parsowana jako kod SQL, więc nie może zmienić planu. To dokładnie ta sama zasada, którą widziałeś przy `where(products.c.price > min_price)` — SQLAlchemy po prostu robi to za Ciebie automatycznie.

> ⚠️ **Pułapka — `literal_column()` z danymi od użytkownika.** `literal_column()` jest przeznaczone do **stałych, znanych Ci fragmentów** — nazw kolumn, wyrażeń arytmetycznych, fragmentów specyficznych dla dialektu. Jeśli kiedykolwiek przekażesz do niego dane pochodzące z formularza, otworzysz dokładnie tę samą dziurę, którą właśnie zamknęliśmy. Reguła jest prosta: **dane zawsze jako parametry, kod SQL zawsze jako literał w Twoim kodzie źródłowym**.

> ⚠️ **Pułapka — `literal_binds` nie zawsze zadziała.** Podstawianie wartości do tekstu (`compile_kwargs={"literal_binds": True}`) to świetne narzędzie diagnostyczne, ale dla niektórych typów danych SQLAlchemy nie umie wyrenderować literału i zgłosi `CompileError: No literal value renderer is available`. Zdarza się to przy typach własnych (`TypeDecorator` — moduł 12), interwałach i niektórych typach dialektowych. W takich sytuacjach zamiast `literal_binds` użyj `echo=True` albo logowania — zobaczysz SQL z parametrami i osobno ich wartości.

> 🧪 **Ćwiczenie — w miejscu.** Napisz funkcję, która przyjmuje nazwę kolumny sortowania od użytkownika. Dlaczego nie możesz wstawić jej jako parametru wiązanego? Jak to rozwiązać bezpiecznie? (Podpowiedź: słownik dozwolonych nazw → kolumna.)

---

## Warstwy wykonawcze: Connection i Engine

Na koniec tej części musimy uporządkować to, co robi `Engine`, a co `Connection` — i wyjaśnić, dlaczego w SQLAlchemy 2.0 **nie ma już** `engine.execute()`.

```text
    Twój kod
       │
       │  engine.connect()          ← „daj mi połączenie z puli”
       ▼
   Connection                      ← pojedyncza rozmowa z bazą, w transakcji
       │
       │  conn.execute(stmt)        ← kompilacja + wysłanie
       ▼
   Dialekt  ──►  DBAPI (sterownik)  ──►  baza danych
       ▲
       │  engine.begin()            ← to samo, ale z automatycznym COMMIT-em
```

**`Engine`** to fabryka połączeń i konfiguracja (adres bazy, pula połączeń, opcje dialektu). Tworzysz go **raz na proces** i trzymasz w jednym miejscu. `Engine` nie jest połączeniem — nie „otwiera się” i nie „zamyka” jak plik.

**`Connection`** to jedno połączenie z bazy, wydzierżawione z puli na czas bloku `with`. To na nim wykonujesz zapytania. Po wyjściu z bloku połączenie wraca do puli — **i to jest ważne dla wyników**:

```python
# examples/04_connection.py
from sqlalchemy import create_engine, select

engine = create_engine("sqlite:///shop.db")

# DOBRZE: wynik przetwarzamy wewnątrz bloku `with`
with engine.connect() as conn:
    rows = conn.execute(select(products.c.sku)).all()
print(len(rows))            # 12 — dane są już w liście, połączenie może wrócić do puli

# ŹLE: wynik odczytywany po zamknięciu połączenia
with engine.connect() as conn:
    result = conn.execute(select(products.c.sku))
# print(result.all())        # ResourceClosedError: This result object is closed
```

### `connect()` kontra `begin()`

| Sposób | Transakcja | Do czego |
|---|---|---|
| `with engine.connect() as conn:` | automatycznie rozpoczęta (autobegin), **Ty decydujesz**, czy `commit()` | odczyty, praca z „commit as you go” |
| `with engine.begin() as conn:` | rozpoczęta, **automatycznie commituje** na końcu bloku, **wycofuje** przy wyjątku | zapisy, DDL, wszystko co ma być atomowe |

> 🧠 **Dlaczego tak jest — „autobegin” w SQLAlchemy 2.0.** Gdy tylko wykonasz jakiekolwiek zapytanie na połączeniu z `connect()`, SQLAlchemy **automatycznie** rozpoczyna transakcję (w logu zobaczysz `BEGIN (implicit)`). Nie musisz pisać `BEGIN` — baza danych i tak nie wykona niczego poza transakcją. Konsekwencja jest jednak poważna: jeśli wykonasz `UPDATE` na połączeniu z `connect()` i **nie wywołasz `conn.commit()`**, to przy wyjściu z bloku zmiany zostaną **wycofane** (rollback). Kod „działał”, a dane się nie zapisały — to jedna z najczęstszych przyczyn zgłoszeń typu „SQLAlchemy nie zapisuje mi danych”.

### Koniec epoki `engine.execute()`

W SQLAlchemy 1.x można było napisać `engine.execute("SELECT ...")` i to działało. W 2.0 **ta metoda została usunięta**. Powód: „skrót” ukrywał, czyjego połączenia użyto, kiedy zaczyna się transakcja i kto ją kończy — a to właśnie te szczegóły decydują o poprawności aplikacji. API 2.0 wymusza jawność.

```python
# examples/04_engine_execute.py
from sqlalchemy import create_engine, text

engine = create_engine("sqlite:///shop.db")

# ❌ SQLAlchemy 1.x — usunięte w 2.0:
# engine.execute("SELECT 1")
# AttributeError: 'Engine' object has no attribute 'execute'

# ✅ Odczyt
with engine.connect() as conn:
    conn.execute(text("SELECT 1")).scalar()

# ✅ Zapis (transakcja z automatycznym COMMIT-em)
with engine.begin() as conn:
    conn.execute(text("UPDATE products SET stock = 0 WHERE stock < 0"))

# ✅ Gdy naprawdę chcesz wysłać surowy tekst wprost do sterownika
with engine.connect() as conn:
    conn.exec_driver_sql("SELECT count(*) FROM products").scalar()
```

> 🧠 **Dlaczego tak jest — `exec_driver_sql()` kontra `text()`.** `text()` to fragment SQL-a „w języku SQLAlchemy”: obsługuje `:parametry`, ma dostęp do typów, można go łączyć z wyrażeniami. `exec_driver_sql()` wysyła tekst **wprost do sterownika**, bez żadnego przetwarzania — parametry podajesz w stylu sterownika (`?` dla SQLite, `%s` dla PostgreSQL-a). Przydaje się do nietypowych poleceń administracyjnych i skryptów, gdzie parsowanie przez SQLAlchemy tylko przeszkadza. Do normalnej pracy używaj `text()`.

> 🆕 **SQLAlchemy 2.1 — `greenlet` nie instaluje się automatycznie.** Jeśli kiedykolwiek zechcesz użyć SQLAlchemy w trybie asynchronicznym (moduł 15), w 2.1 musisz doinstalować bibliotekę `greenlet` jawnie: `pip install "sqlalchemy[asyncio]"`. W 2.0 instalowała się „przy okazji”. Dla tego modułu (tryb synchroniczny) nie ma to znaczenia, ale warto zapamiętać, żeby nie zdziwić się przy przejściu na async.

> 🧪 **Ćwiczenie — w miejscu.** Uruchom `python -c "import sqlalchemy; print(sqlalchemy.__version__)"`, a potem wykonaj `select(1)` na połączeniu z `echo=True`. Odnajdź w logu linie `BEGIN (implicit)` i `ROLLBACK`/`COMMIT` i zastanów się, kto je wywołał.

---

## Przykład kompletny: katalog produktów

Poniższy skrypt zbiera wszystko, o czym mówiliśmy. Tworzy bazę SQLite, wypełnia ją danymi sklepu i wykonuje kilkanaście zapytań, za każdym razem wypisując SQL, który poleci do bazy.

**Jak uruchomić:**

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install "SQLAlchemy>=2.0"
python examples/04_catalog.py
```

> 📝 **Uwaga o nazwie pliku.** Pliki w tym kursie mają nazwy zaczynające się od cyfry (`04_catalog.py`), więc **nie da się ich zaimportować** jako modułu Pythona (`import 04_catalog` to błąd składni). Dlatego trzymamy cały przykład w jednym pliku. W prawdziwym projekcie nazwij plik np. `shop/schema.py` i importuj z niego tabele.

### Część 1: schemat

```python
# examples/04_catalog.py
"""Moduł 04 — czytanie danych w SQLAlchemy Core (katalog produktów)."""

from __future__ import annotations

from typing import Any

from sqlalchemy import (
    Boolean,
    Column,
    DateTime,
    ForeignKey,
    Integer,
    MetaData,
    Numeric,
    String,
    Table,
    Text,
    case,
    cast,
    create_engine,
    desc,
    extract,
    func,
    literal,
    nulls_last,
    select,
    text,
)
from sqlalchemy.engine import Engine
from sqlalchemy.sql import Select

# --------------------------------------------------------------------------
# 1. Schemat: pięć tabel sklepu
# --------------------------------------------------------------------------

metadata = MetaData()

categories = Table(
    "categories",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String(60), nullable=False, unique=True),
)

products = Table(
    "products",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("sku", String(32), nullable=False, unique=True),
    Column("name", String(120), nullable=False),
    Column("category_id", ForeignKey("categories.id"), nullable=False),
    Column("price", Numeric(10, 2), nullable=False),
    Column("stock", Integer, nullable=False, server_default=text("0")),
    Column("is_active", Boolean, nullable=False, server_default=text("1")),
    Column("description", Text, nullable=True),
    Column("created_at", DateTime, nullable=False, server_default=func.now()),
)

customers = Table(
    "customers",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String(120), nullable=False),
    Column("email", String(120), nullable=False, unique=True),
    Column("city", String(80), nullable=True),
)

orders = Table(
    "orders",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("customer_id", ForeignKey("customers.id"), nullable=False),
    Column("status", String(20), nullable=False),
    Column("created_at", DateTime, nullable=False),
)

order_items = Table(
    "order_items",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("order_id", ForeignKey("orders.id"), nullable=False),
    Column("product_id", ForeignKey("products.id"), nullable=False),
    Column("quantity", Integer, nullable=False),
    Column("unit_price", Numeric(10, 2), nullable=False),
)
```

### Część 2: dane

```python
# examples/04_catalog.py (ciąg dalszy)
# --------------------------------------------------------------------------
# 2. Dane przykładowe
#
# Wstawiamy je surowym SQL-em przez text(), bo `insert()` w Core omawiamy
# dopiero w module 05. Ten fragment nie jest tematem tego modułu — ma tylko
# przygotować grunt pod zapytania.
# --------------------------------------------------------------------------

SEED_SQL: list[str] = [
    """
    INSERT INTO categories (id, name) VALUES
        (1, 'Books'), (2, 'Games'), (3, 'Music')
    """,
    """
    INSERT INTO products
        (id, sku, name, category_id, price, stock, is_active, description, created_at)
    VALUES
        (1,  'BOK-001', 'Clean Code',         1, 129.00, 12, 1, NULL,                 '2025-11-03 10:00:00'),
        (2,  'BOK-002', 'Python Cookbook',    1, 149.50,  4, 1, 'Zbiór przepisów',    '2025-11-20 12:30:00'),
        (3,  'BOK-003', 'SQL for Smarties',   1, 199.99,  0, 1, NULL,                 '2026-01-15 09:15:00'),
        (4,  'BOK-004', 'Refactoring',        1, 179.00,  7, 0, NULL,                 '2026-02-01 08:00:00'),
        (5,  'GRY-001', 'Catan',              2, 189.00,  3, 1, NULL,                 '2025-12-05 16:40:00'),
        (6,  'GRY-002', 'Azul',               2, 159.00,  0, 1, 'Gra abstrakcyjna',   '2026-01-08 11:05:00'),
        (7,  'GRY-003', 'Wingspan',           2, 229.00,  2, 1, NULL,                 '2026-03-11 14:20:00'),
        (8,  'GRY-004', 'Ticket to Ride',     2, 209.00,  9, 1, NULL,                 '2026-04-02 10:10:00'),
        (9,  'MUZ-001', 'Kind of Blue',       3, 119.00,  5, 1, NULL,                 '2025-10-10 09:00:00'),
        (10, 'MUZ-002', 'Blue Train',         3, 129.00,  1, 1, NULL,                 '2026-01-20 15:45:00'),
        (11, 'MUZ-003', 'A Love Supreme',     3, 139.00,  0, 1, NULL,                 '2026-02-14 13:00:00'),
        (12, 'MUZ-004', 'Time Out',           3, 109.00,  6, 1, NULL,                 '2026-03-30 17:30:00')
    """,
    """
    INSERT INTO customers (id, name, email, city) VALUES
        (1, 'Anna Kowalska',   'anna@example.com',   'Kraków'),
        (2, 'Piotr Nowak',     'piotr@example.com',  'Gdańsk'),
        (3, 'Marta Zielińska', 'marta@example.com',  NULL)
    """,
    """
    INSERT INTO orders (id, customer_id, status, created_at) VALUES
        (1, 1, 'paid',      '2026-01-12 10:15:00'),
        (2, 1, 'paid',      '2026-02-03 09:40:00'),
        (3, 2, 'paid',      '2026-02-19 18:20:00'),
        (4, 3, 'cancelled', '2026-03-05 12:00:00'),
        (5, 2, 'paid',      '2026-03-21 15:30:00'),
        (6, 1, 'paid',      '2026-04-07 11:10:00'),
        (7, 3, 'paid',      '2026-04-25 20:05:00'),
        (8, 2, 'draft',     '2026-05-02 08:45:00')
    """,
    """
    INSERT INTO order_items (id, order_id, product_id, quantity, unit_price) VALUES
        (1,  1, 1,  2, 129.00),
        (2,  1, 9,  1, 119.00),
        (3,  2, 5,  1, 189.00),
        (4,  3, 2,  1, 149.50),
        (5,  3, 12, 2, 109.00),
        (6,  4, 7,  1, 229.00),
        (7,  5, 1,  1, 129.00),
        (8,  5, 10, 3, 129.00),
        (9,  6, 8,  2, 209.00),
        (10, 6, 3,  1, 199.99),
        (11, 7, 11, 1, 139.00),
        (12, 7, 6,  1, 159.00)
    """,
]

engine: Engine = create_engine("sqlite:///shop_demo.db", echo=False)


def prepare_database() -> None:
    """Tworzy schemat od zera i wypełnia go danymi."""
    metadata.drop_all(engine)
    metadata.create_all(engine)
    with engine.begin() as conn:
        for statement in SEED_SQL:
            conn.execute(text(statement))


def show(stmt: Select[Any]) -> None:
    """Wypisuje SQL, który SQLAlchemy wyśle do bazy (z podstawionymi wartościami)."""
    compiled = stmt.compile(
        dialect=engine.dialect,
        compile_kwargs={"literal_binds": True},
    )
    print(compiled)
    print()
```

Dane, na których będziemy pracować:

| id | sku | nazwa | kategoria | cena | stan | aktywny | utworzono |
|---|---|---|---|---|---|---|---|
| 1 | BOK-001 | Clean Code | Books | 129,00 | 12 | tak | 2025-11-03 |
| 2 | BOK-002 | Python Cookbook | Books | 149,50 | 4 | tak | 2025-11-20 |
| 3 | BOK-003 | SQL for Smarties | Books | 199,99 | 0 | tak | 2026-01-15 |
| 4 | BOK-004 | Refactoring | Books | 179,00 | 7 | **nie** | 2026-02-01 |
| 5 | GRY-001 | Catan | Games | 189,00 | 3 | tak | 2025-12-05 |
| 6 | GRY-002 | Azul | Games | 159,00 | 0 | tak | 2026-01-08 |
| 7 | GRY-003 | Wingspan | Games | 229,00 | 2 | tak | 2026-03-11 |
| 8 | GRY-004 | Ticket to Ride | Games | 209,00 | 9 | tak | 2026-04-02 |
| 9 | MUZ-001 | Kind of Blue | Music | 119,00 | 5 | tak | 2025-10-10 |
| 10 | MUZ-002 | Blue Train | Music | 129,00 | 1 | tak | 2026-01-20 |
| 11 | MUZ-003 | A Love Supreme | Music | 139,00 | 0 | tak | 2026-02-14 |
| 12 | MUZ-004 | Time Out | Music | 109,00 | 6 | tak | 2026-03-30 |

### Część 3: zapytania

```python
# examples/04_catalog.py (ciąg dalszy)
# --------------------------------------------------------------------------
# 3. Zapytania
# --------------------------------------------------------------------------


def demo_filtry() -> None:
    """Filtry: kategoria, zakres ceny, aktywność."""
    print("=== Filtry ===")
    stmt = (
        select(products.c.sku, products.c.name, products.c.price)
        .where(
            products.c.category_id == 1,
            products.c.is_active.is_(True),
            products.c.price.between(120, 200),
        )
        .order_by(products.c.price.desc())
    )
    show(stmt)
    with engine.connect() as conn:
        for row in conn.execute(stmt):
            print(f"  {row.sku}  {row.price:>7}  {row.name}")


def demo_fraza() -> None:
    """Szukanie po fragmencie nazwy, bez względu na wielkość liter."""
    print("=== Fraza ===")
    stmt = (
        select(products.c.sku, products.c.name)
        .where(products.c.name.ilike("%blue%"))
        .order_by(products.c.name)
    )
    show(stmt)
    with engine.connect() as conn:
        for row in conn.execute(stmt):
            print(f"  {row.sku}  {row.name}")


def demo_brak_opisu() -> None:
    """Produkty bez opisu — demonstracja is_(None)."""
    print("=== Brak opisu ===")
    stmt = (
        select(products.c.sku)
        .where(products.c.description.is_(None))
        .order_by(products.c.sku)
    )
    show(stmt)
    with engine.connect() as conn:
        print("  ", [r.sku for r in conn.execute(stmt)])


def demo_warunki_zlozone() -> None:
    """Warunki złożone: or_() i and_() z jawnymi nawiasami."""
    print("=== Warunki złożone ===")
    stmt = (
        select(products.c.sku, products.c.name, products.c.price, products.c.stock)
        .where(
            or_(
                and_(products.c.category_id == 2, products.c.price < 200),
                and_(products.c.category_id == 3, products.c.stock > 3),
            )
        )
        .order_by(products.c.sku)
    )
    show(stmt)
    with engine.connect() as conn:
        for row in conn.execute(stmt):
            print(f"  {row.sku}  {row.price:>7}  stan: {row.stock}  {row.name}")


def demo_stronicowanie() -> None:
    """Sortowanie i stronicowanie."""
    print("=== Stronicowanie ===")
    stmt = (
        select(products.c.sku, products.c.price)
        .order_by(desc(products.c.price), products.c.sku)
        .limit(4)
        .offset(4)
    )
    show(stmt)
    with engine.connect() as conn:
        for row in conn.execute(stmt):
            print(f"  {row.sku}  {row.price:>7}")


def demo_agregacje() -> None:
    """Agregacje globalne."""
    print("=== Agregacje ===")
    stmt = select(
        func.count().label("liczba_produktow"),
        func.avg(products.c.price).label("srednia_cena"),
        func.min(products.c.price).label("cena_min"),
        func.max(products.c.price).label("cena_max"),
        func.sum(products.c.stock).label("suma_stanow"),
    ).select_from(products)
    show(stmt)
    with engine.connect() as conn:
        row = conn.execute(stmt).one()
    print(f"  produktów: {row.liczba_produktow}")
    print(f"  średnia cena: {row.srednia_cena:.2f} zł")
    print(f"  od {row.cena_min:.2f} do {row.cena_max:.2f} zł")
    print(f"  łącznie w magazynie: {row.suma_stanow} szt.")


def demo_grupowanie() -> None:
    """Średnia cena w kategorii + HAVING."""
    print("=== Grupowanie ===")
    stmt = (
        select(
            categories.c.name.label("kategoria"),
            func.count(products.c.id).label("liczba"),
            func.avg(products.c.price).label("srednia_cena"),
            func.sum(products.c.stock).label("suma_stanow"),
        )
        .select_from(products.join(categories, products.c.category_id == categories.c.id))
        .group_by(categories.c.name)
        .having(func.avg(products.c.price) > 130)
        .order_by(func.avg(products.c.price).desc())
    )
    show(stmt)
    with engine.connect() as conn:
        for row in conn.execute(stmt):
            print(f"  {row.kategoria:<6} {row.liczba} produktów, "
                  f"średnia {row.srednia_cena:.2f} zł, stan {row.suma_stanow}")


def demo_kubelki() -> None:
    """CASE: własne kubełki cenowe."""
    print("=== Kubełki cenowe ===")
    polka = case(
        (products.c.price >= 200, "premium"),
        (products.c.price >= 150, "standard"),
        else_="budget",
    ).label("polka")
    stmt = (
        select(products.c.sku, products.c.price, polka)
        .order_by(desc(products.c.price), products.c.sku)
    )
    show(stmt)
    with engine.connect() as conn:
        for row in conn.execute(stmt):
            print(f"  {row.sku:<8} {row.price:>7}  {row.polka}")


def demo_parametry() -> None:
    """Parametry wiązane w surowym SQL-u."""
    print("=== Parametry ===")
    stmt = text(
        "SELECT sku, name, price FROM products "
        "WHERE category_id = :cat AND price >= :min_price "
        "ORDER BY price"
    )
    print(stmt, "\n")
    with engine.connect() as conn:
        for row in conn.execute(stmt, {"cat": 2, "min_price": 150}):
            print(f"  {row.sku:<8} {row.price:>7.2f}  {row.name}")


def demo_raport_miesieczny() -> None:
    """Mapa sprzedaży po miesiącach (tylko zamówienia opłacone)."""
    print("=== Sprzedaż po miesiącach ===")
    rok = extract("year", orders.c.created_at)
    miesiac = extract("month", orders.c.created_at)
    przychod = func.sum(order_items.c.quantity * order_items.c.unit_price)

    stmt = (
        select(rok.label("rok"), miesiac.label("miesiac"), przychod.label("przychod"))
        .select_from(orders.join(order_items, orders.c.id == order_items.c.order_id))
        .where(orders.c.status == "paid")
        .group_by(rok, miesiac)
        .order_by(rok, miesiac)
    )
    show(stmt)
    with engine.connect() as conn:
        for row in conn.execute(stmt):
            print(f"  {row.rok}-{row.miesiac:02d}   {row.przychod:>10.2f} zł")


def demo_result_api() -> None:
    """Przegląd metod obiektu Result."""
    print("=== Obiekt Result ===")
    stmt = select(products.c.sku, products.c.name).order_by(products.c.sku)
    with engine.connect() as conn:
        result = conn.execute(stmt)
        print("  keys()  :", result.keys())

        rows = result.all()                  # materializujemy wynik w liście
        print("  all()   :", len(rows), "wierszy; pierwszy:", rows[0])

        result = conn.execute(stmt)          # świeży wynik — poprzedni jest zużyty
        print("  first() :", result.first())
        print("  first() :", result.first(), "(kursor poszedł dalej)")

        result = conn.execute(stmt.limit(2))
        print("  mappings:", result.mappings().all())

        result = conn.execute(stmt)
        print("  partitions:", [len(chunk) for chunk in result.partitions(5)])

        result = conn.execute(stmt.where(products.c.sku == "GRY-003"))
        print("  one_or_none:", result.one_or_none())


if __name__ == "__main__":
    prepare_database()
    demo_filtry()
    demo_fraza()
    demo_brak_opisu()
    demo_warunki_zlozone()
    demo_stronicowanie()
    demo_agregacje()
    demo_grupowanie()
    demo_kubelki()
    demo_parametry()
    demo_raport_miesieczny()
    demo_result_api()
```

> 🔬 **Pod maską — co zobaczysz na ekranie.** Kilka najważniejszych fragmentów wyjścia:
>
> ```text
> === Filtry ===
> SELECT products.sku, products.name, products.price
> FROM products
> WHERE products.category_id = 1 AND products.is_active IS 1
>   AND products.price BETWEEN 120 AND 200
> ORDER BY products.price DESC
>
>   BOK-003   199.99  SQL for Smarties
>   BOK-002   149.50  Python Cookbook
>   BOK-001   129.00  Clean Code
> ```
>
> Na PostgreSQL-u ten sam kod wypisze `products.is_active IS true` — bo dialekt PostgreSQL-a ma prawdziwy typ logiczny, a SQLite przechowuje prawdę jako liczbę `1`.
>
> ```text
> === Sprzedaż po miesiącach ===
> SELECT CAST(STRFTIME('%Y', orders.created_at) AS INTEGER) AS rok,
>        CAST(STRFTIME('%m', orders.created_at) AS INTEGER) AS miesiac,
>        sum(order_items.quantity * order_items.unit_price) AS przychod
> FROM orders JOIN order_items ON orders.id = order_items.order_id
> WHERE orders.status = 'paid'
> GROUP BY CAST(STRFTIME('%Y', orders.created_at) AS INTEGER),
>          CAST(STRFTIME('%m', orders.created_at) AS INTEGER)
> ORDER BY CAST(STRFTIME('%Y', orders.created_at) AS INTEGER),
>          CAST(STRFTIME('%m', orders.created_at) AS INTEGER)
>
>   2026-01       377.00 zł
>   2026-02       556.50 zł
>   2026-03       516.00 zł
>   2026-04       915.99 zł
> ```
>
> Zwróć uwagę: **maja nie ma w wyniku**. To nie błąd — zamówienie z maja ma status `draft`, więc odpadło w `WHERE`. Ale to dobra ilustracja: wykres przychodu z dziurą w maju jest gorszy niż wykres z zerem w maju. Jeśli budujesz raport, który ma pokazywać każdy miesiąc, musisz zadbać o „wypełnienie dziur” w Pythonie albo w SQL-u.

> ⚠️ **Pułapka — `avg()` wraca z innej bajki niż reszta.** Zauważ, że w agregacjach używamy formatowania `:.2f` zamiast wypisywać wartość wprost. Powód: `func.avg()` może zwrócić `Decimal` albo `float` — zależnie od typu argumentu i dialektu. Formatowanie liczbowe działa dla obu, więc wynik jest przewidywalny. Jeśli w kodzie aplikacji porównujesz wynik `avg()` z inną liczbą, najpierw upewnij się, jaki typ dostajesz (`print(type(row.srednia_cena))`) — to typowa przyczyna testów, które „czasem przechodzą”.

---

## Podsumowanie

1. **`select()` buduje zapytanie, `execute()` je wykonuje.** Zapytanie to obiekt Pythona — możesz go wypisać, sprawdzić i przekazać dalej, nie dotykając bazy.
2. **`from` wynika z kolumn.** Gdy używasz wyłącznie agregatów, musisz wskazać tabelę przez `.select_from(...)`.
3. **`Result` to kursor, nie lista.** Każdy odczyt zużywa wiersze; chcesz dane dwa razy — zrób `rows = result.all()`.
4. **`one()` rzuca wyjątki celowo.** Brak wyniku lub dwa wyniki to asercja niezmiennika, a nie niedogodność.
5. **`scalar()` = jedna wartość, `scalars()` = wiele wierszy.** Jedna litera, dwie różne rzeczy.
6. **`Row` pamięta nazwy kolumn.** Odwołuj się po nazwie, nie po indeksie; wyrażeniom nadawaj `.label()`.
7. **`where()` zamiast `filter()`, `is_(None)` zamiast `== None`, nawiasy przy `&`/`|`** — trzy reguły, które oszczędzają godziny debugowania.
8. **Dane zawsze idą jako parametry wiązane.** `:param`, `bindparam()`, `expanding=True` — nigdy f-string do SQL-a.
9. **`GROUP BY` grupuje, `HAVING` filtruje grupy.** Każda niezagregowana kolumna w `SELECT` musi być w `GROUP BY`.
10. **`engine.execute()` nie istnieje w 2.0.** Używaj `engine.connect()` do odczytu i `engine.begin()` do zapisu — a w `connect()` pamiętaj o `commit()`.

---

## Ćwiczenia

Wszystkie zadania dotyczą schematu z sekcji 12 (`categories`, `products`, `customers`, `orders`, `order_items`) i tych samych danych.

**Ćwiczenie 1 — rozgrzewka: filtry i sortowanie.**
Zbuduj zapytania:
a) Wszystkie aktywne produkty z kategorii „Music” (id = 3) — kolumny `sku`, `name`, `price`, sortowane rosnąco po cenie.
b) Produkty, których nazwa zawiera „blue”, bez względu na wielkość liter — kolumny `sku`, `name`.
c) Produkty w cenie od 100 do 150 zł **lub** takie, których stan magazynowy wynosi 0 — kolumny `sku`, `price`, `stock`. **Użyj nawiasów** przy warunkach złożonych.

**Ćwiczenie 2 — raporty i stronicowanie.**
Zbuduj zapytania:
a) Pięć najdroższych produktów wraz z nazwą kategorii (użyj `join`, tak jak w `demo_grupowanie`).
b) Liczba produktów i średnia cena dla każdej kategorii — tylko dla kategorii, które mają **co najmniej 4 produkty**.
c) Produkty, które **nigdy nie zostały zamówione** (podpowiedź: `~products.c.id.in_(select(order_items.c.product_id))`).

**Ćwiczenie 3 — osiem zapytań o rosnącej trudności.**
1. `sku` i `name` wszystkich produktów, których opis jest pusty, posortowane po `sku`.
2. `sku` i `price` produktów w kategoriach 1 i 2, droższych niż 140 zł.
3. Trzy najtańsze produkty aktywne, z etykietą „tanie” — kolumny `sku`, `name`, `price`.
4. Liczba zamówień na klienta (użyj `orders` i `customers`), posortowana malejąco po liczbie.
5. Wartość każdego zamówienia (`id`, liczba pozycji, suma `quantity * unit_price`), posortowana malejąco po wartości.
6. Przychód miesięczny w 2026 roku z etykietą kwartału (`CASE` + `extract`), tylko zamówienia o statusie `paid`.
7. Produkty, których cena jest wyższa od średniej ceny wszystkich produktów (podpowiedź: `select(func.avg(products.c.price)).scalar_subquery()`).
8. Dla każdej kategorii: nazwa, liczba produktów, liczba produktów z zerowym stanem magazynowym (użyj `func.sum(case(...))`), posortowane po liczbie produktów malejąco.

### Rozwiązania

> 💡 Rozwiązania pokazują SQL i oczekiwany wynik. Zakładają ten sam schemat i te same dane co w sekcji 12 — możesz je wkleić do pliku `examples/04_catalog.py` i uruchomić.

**Ćwiczenie 1a**

```python
stmt = (
    select(products.c.sku, products.c.name, products.c.price)
    .where(products.c.category_id == 3, products.c.is_active.is_(True))
    .order_by(products.c.price)
)
# SELECT products.sku, products.name, products.price FROM products
# WHERE products.category_id = 3 AND products.is_active IS 1
# ORDER BY products.price
# → MUZ-004 109.00, MUZ-001 119.00, MUZ-002 129.00, MUZ-003 139.00
```

**Ćwiczenie 1b**

```python
stmt = (
    select(products.c.sku, products.c.name)
    .where(products.c.name.ilike("%blue%"))
    .order_by(products.c.name)
)
# SQLite:      WHERE lower(products.name) LIKE lower('%blue%')
# PostgreSQL:  WHERE products.name ILIKE '%blue%'
# → MUZ-002 Blue Train, MUZ-001 Kind of Blue
```

> ⚠️ Uwaga: gdybyś użył `like("%blue%")` (bez `i`), na **SQLite** wynik byłby identyczny — bo domyślnie `LIKE` ignoruje wielkość liter dla znaków ASCII. Na **PostgreSQL-u** `LIKE` jest wrażliwy na wielkość liter i taki warunek nie zwróciłby niczego. To jedna z najczęstszych „różnic między środowiskiem deweloperskim a produkcją”.

**Ćwiczenie 1c**

```python
stmt = (
    select(products.c.sku, products.c.price, products.c.stock)
    .where(
        or_(
            products.c.price.between(100, 150),
            products.c.stock == 0,
        )
    )
    .order_by(products.c.sku)
)
# WHERE (products.price BETWEEN 100 AND 150) OR (products.stock = 0)
# → 8 wierszy: BOK-001, BOK-002, BOK-003, GRY-002, MUZ-001, MUZ-002, MUZ-003, MUZ-004
```

**Ćwiczenie 2a**

```python
stmt = (
    select(products.c.sku, products.c.name, products.c.price, categories.c.name.label("kategoria"))
    .select_from(products.join(categories, products.c.category_id == categories.c.id))
    .order_by(desc(products.c.price))
    .limit(5)
)
# → GRY-003 Wingspan 229.00 Games
#   GRY-004 Ticket to Ride 209.00 Games
#   BOK-003 SQL for Smarties 199.99 Books
#   GRY-001 Catan 189.00 Games
#   BOK-004 Refactoring 179.00 Books
```

**Ćwiczenie 2b**

```python
stmt = (
    select(
        categories.c.name.label("kategoria"),
        func.count(products.c.id).label("liczba"),
        func.avg(products.c.price).label("srednia_cena"),
    )
    .select_from(products.join(categories, products.c.category_id == categories.c.id))
    .group_by(categories.c.name)
    .having(func.count(products.c.id) >= 4)
    .order_by(categories.c.name)
)
# → Books 4 164.37 | Games 4 196.50 | Music 4 124.00
# (w naszych danych wszystkie trzy kategorie mają po 4 produkty)
```

**Ćwiczenie 2c**

```python
stmt = (
    select(products.c.sku, products.c.name)
    .where(~products.c.id.in_(select(order_items.c.product_id)))
    .order_by(products.c.sku)
)
# WHERE products.id NOT IN (SELECT order_items.product_id FROM order_items)
# → BOK-004 Refactoring
```

> 🧠 Zwróć uwagę, że `in_()` przyjmuje nie tylko listę wartości, ale także **całe zapytanie** (`select(...)`). To podzapytanie — baza wykona je wewnętrznie i porówna z listą jego wyników. Więcej o podzapytaniach w module 06.

**Ćwiczenie 3 — rozwiązania skrótowo**

```python
# 1. Produkty bez opisu
stmt = (
    select(products.c.sku, products.c.name)
    .where(products.c.description.is_(None))
    .order_by(products.c.sku)
)
# → 9 wierszy (opis ma tylko BOK-002 i GRY-002)

# 2. Kategorie 1 i 2, cena > 140
stmt = (
    select(products.c.sku, products.c.price)
    .where(products.c.category_id.in_([1, 2]), products.c.price > 140)
    .order_by(products.c.price)
)
# → BOK-002 149.50, GRY-002 159.00, BOK-004 179.00, GRY-001 189.00,
#   BOK-003 199.99, GRY-004 209.00, GRY-003 229.00

# 3. Trzy najtańsze aktywne
stmt = (
    select(products.c.sku, products.c.name, products.c.price)
    .where(products.c.is_active.is_(True))
    .order_by(products.c.price, products.c.sku)
    .limit(3)
)
# → MUZ-004 109.00, MUZ-001 119.00, BOK-001 129.00

# 4. Liczba zamówień na klienta
stmt = (
    select(
        customers.c.name.label("klient"),
        func.count(orders.c.id).label("liczba_zamowien"),
    )
    .select_from(customers.join(orders, customers.c.id == orders.customer_id))
    .group_by(customers.c.name)
    .order_by(func.count(orders.c.id).desc(), customers.c.name)
)
# → Anna Kowalska 3, Piotr Nowak 3, Marta Zielińska 2

# 5. Wartość każdego zamówienia
stmt = (
    select(
        orders.c.id.label("zamowienie"),
        func.count(order_items.c.id).label("pozycje"),
        func.sum(order_items.c.quantity * order_items.c.unit_price).label("wartosc"),
    )
    .select_from(orders.join(order_items, orders.c.id == order_items.c.order_id))
    .group_by(orders.c.id)
    .order_by(func.sum(order_items.c.quantity * order_items.c.unit_price).desc())
)
# → 6 (2 poz., 617.99), 5 (2, 516.00), 1 (2, 377.00), 3 (2, 367.50),
#   7 (2, 298.00), 4 (1, 229.00), 2 (1, 189.00)
# Zamówienie 8 nie pojawia się — nie ma pozycji, a JOIN wewnętrzny je odrzuca.

# 6. Przychód miesięczny z kwartałem
rok = extract("year", orders.c.created_at)
miesiac = extract("month", orders.c.created_at)
kwartal = case(
    (miesiac <= 3, 1),
    (miesiac <= 6, 2),
    (miesiac <= 9, 3),
    else_=4,
).label("kwartal")
stmt = (
    select(
        rok.label("rok"),
        miesiac.label("miesiac"),
        kwartal,
        func.sum(order_items.c.quantity * order_items.c.unit_price).label("przychod"),
    )
    .select_from(orders.join(order_items, orders.c.id == order_items.c.order_id))
    .where(orders.c.status == "paid")
    .group_by(rok, miesiac, kwartal)
    .order_by(rok, miesiac)
)
# → 2026-01 kwartał 1: 377.00 | 2026-02 k1: 556.50
#   2026-03 k1: 516.00 | 2026-04 k2: 915.99

# 7. Produkty droższe od średniej
srednia = select(func.avg(products.c.price)).scalar_subquery()
stmt = (
    select(products.c.sku, products.c.name, products.c.price)
    .where(products.c.price > srednia)
    .order_by(products.c.price.desc())
)
# → 5 wierszy: GRY-003 229.00, GRY-004 209.00, BOK-003 199.99,
#   GRY-001 189.00, BOK-004 179.00   (średnia to 161.62)

# 8. Kategorie z liczbą produktów i liczbą braków
brak = func.sum(case((products.c.stock == 0, 1), else_=0)).label("bez_stanu")
stmt = (
    select(
        categories.c.name.label("kategoria"),
        func.count(products.c.id).label("liczba"),
        brak,
    )
    .select_from(products.join(categories, products.c.category_id == categories.c.id))
    .group_by(categories.c.name)
    .order_by(func.count(products.c.id).desc(), categories.c.name)
)
# → Books 4 / 1, Games 4 / 1, Music 4 / 1
```

> 🧠 **Dlaczego `func.sum(case(...))` zamiast `func.count()` z filtrem?** Bo liczymy **dwie różne rzeczy w jednym przebiegu** po danych: ile jest produktów w kategorii i ile z nich nie ma stanu. `count()` liczyłby wszystkie wiersze grupy, a my potrzebujemy warunku w środku agregacji. `sum(case(...))` to standardowy sposób na „policz tylko te, które spełniają warunek” — 1 dla spełniających, 0 dla reszty, potem suma.

---

## Najczęstsze błędy i jak je czytać

| Komunikat / objaw | Przyczyna | Naprawa |
|---|---|---|
| `AttributeError: 'Engine' object has no attribute 'execute'` | kod z SQLAlchemy 1.x; `engine.execute()` usunięto w 2.0 | `with engine.connect() as conn: conn.execute(...)` albo `engine.begin()` |
| `TypeError: Boolean value of this clause is not defined` | brak nawiasów przy `&`/`\|`; Python potraktował to jako łańcuch porównań | nawiasy wokół każdego warunku lub `and_()`/`or_()` |
| `ResourceClosedError: This result object is closed` | wynik (`Result`) czytany po wyjściu z bloku `with engine.connect()` albo po jego zużyciu | przetwarzaj wynik **wewnątrz** bloku; materializuj przez `result.all()` |
| `NoResultFound` | `one()` / `scalar_one()` nie znalazły wiersza | użyj `one_or_none()` / `first()`, albo obsłuż wyjątek — to asercja niezmiennika |
| `MultipleResultsFound` | `one()` / `scalar_one()` znalazły więcej niż jeden wiersz | dodaj warunek zawężający lub użyj `first()` |
| `InvalidRequestError: Multiple columns ...` przy `scalar_one()` | zapytanie zwraca więcej niż jedną kolumnę | wybierz jedną kolumnę albo użyj `one()` |
| `CompileError: No literal value renderer is available` | `literal_binds=True` dla typu, którego dialekt nie umie wyrenderować | użyj `echo=True` / logowania zamiast `literal_binds` |
| Pusty wynik przy `in_([])` | pusta lista renderuje się jako `IN (NULL)` — warunek nigdy nieprawdziwy | obsłuż pustą listę jawnie w kodzie |
| `SAWarning: Dialect sqlite+pysqlite does *not* support Decimal objects natively` | zapis `Decimal` do kolumny `Numeric` na SQLite | trzymaj kwoty w groszach (`Integer`) albo użyj PostgreSQL-a |
| `SAWarning: Textual SQL expression ... should be explicitly declared as text()` | do `where()`/`select()` trafił goły string | opakuj surowy SQL w `text()` |
| Dane nie zapisują się po `UPDATE` na `connect()` | brak `conn.commit()` — autobegin wycofuje zmiany przy zamknięciu | `with engine.begin() as conn:` lub jawny `conn.commit()` |
| Zapytanie zwraca kolumny `anon_1`, `count_1` | wyrażenie bez etykiety | dodaj `.label("czytelna_nazwa")` |
| Wynik zawiera duplikaty po dodaniu `join` | dołączenie tabeli 1:N mnoży wiersze | `.distinct()` albo agregacja (szczegóły w module 06) |
| `GROUP BY` zwraca dziwne wartości kolumn | kolumna w `SELECT` nie jest w `GROUP BY` (SQLite/MySQL nie zgłaszają błędu) | dodaj kolumnę do `group_by` albo usuń ją z `select` |

---

## Słowniczek modułu

| Termin (EN) | Polski odpowiednik | Wyjaśnienie |
|---|---|---|
| `Select` | zapytanie SELECT | obiekt Pythona opisujący zapytanie; nie dotyka bazy, dopóki nie zostanie wykonany |
| `Result` | wynik | kursor czytający wiersze zwrócone przez bazę; jednorazowy strumień |
| `CursorResult` | wynik kursora | `Result` zwracany przez `Connection.execute()` |
| `ScalarResult` | wynik skalarny | strumień pierwszych kolumn z każdego wiersza (efekt `scalars()`) |
| `MappingResult` | wynik mapujący | strumień wierszy w postaci słowników (efekt `mappings()`) |
| `Row` | wiersz | pojedynczy wiersz wyniku; krotka z nazwanymi polami |
| `RowMapping` | mapowanie wiersza | widok wiersza jako słownik (`row._mapping`) |
| bind parameter | parametr wiązany | wartość przekazywana do bazy osobno od tekstu SQL-a; chroni przed wstrzyknięciem |
| expanding parameter | parametr rozszerzalny | parametr dla `IN (...)`, rozwijany do tylu miejsc, ile elementów ma lista |
| post-compile | po kompilacji | etap, na którym SQLAlchemy rozwija znaczniki `__[POSTCOMPILE_...]` w konkretne parametry |
| `executemany` | wykonanie wielokrotne | jedno zapytanie i wiele zestawów parametrów w jednej rundzie do bazy |
| `literal_binds` | podstawienie literałów | tryb kompilacji wstawiający wartości w tekst SQL-a; tylko do diagnostyki |
| `label()` | etykieta | nadanie nazwy kolumnie lub wyrażeniu w wyniku |
| `where()` | warunek | klauzula `WHERE`; w stylu 2.0 zastępuje `filter()` |
| `order_by()` | porządek | klauzula `ORDER BY` |
| `nulls_last()` | puste na końcu | sortowanie z `NULL`-ami na końcu listy |
| `distinct()` | bez duplikatów | klauzula `DISTINCT`; z kolumnami daje `DISTINCT ON` (PostgreSQL) |
| aggregate function | funkcja agregująca | `count`, `sum`, `avg`, `min`, `max` — wiele wierszy w jedną wartość |
| `group_by()` / `having()` | grupowanie / filtr grup | `GROUP BY` dzieli wiersze, `HAVING` filtruje powstałe grupy |
| `scalar_subquery()` | podzapytanie skalarne | zapytanie zwracające jedną wartość, użyte wewnątrz innego zapytania |
| `case()` | wyrażenie warunkowe | SQL-owy `if`/`else` wykonywany dla każdego wiersza |
| `cast()` | rzutowanie typu | zmiana typu wartości w bazie (`CAST(x AS INTEGER)`) |
| `extract()` | wyciągnięcie części daty | rok, miesiąc, dzień z wartości czasu; na SQLite tłumaczone na `STRFTIME` |
| autobegin | automatyczne rozpoczęcie | niejawne otwarcie transakcji przy pierwszym zapytaniu na połączeniu |
| commit as you go | zatwierdzaj po drodze | styl pracy z `engine.connect()`, w którym sam wywołujesz `commit()` |
| dialect | dialekt | zestaw reguł tłumaczenia SQL-a dla konkretnej bazy (SQLite, PostgreSQL…) |
| DBAPI | sterownik bazy | biblioteka Pythona łącząca się z bazą (`sqlite3`, `psycopg`, `asyncpg`) |

---

## Dalsze czytanie

- **Core Tutorial — „Using SELECT Statements”** (najlepsze miejsce na start po tym module): `https://docs.sqlalchemy.org/en/20/tutorial/data_select.html`
- **SELECT i konstrukcje z nim związane** (pełna lista metod `Select`): `https://docs.sqlalchemy.org/en/20/core/selectable.html`
- **Praca z połączeniami, transakcje, `exec_driver_sql`**: `https://docs.sqlalchemy.org/en/20/core/connections.html`
- **Elementy języka SQL (operatory, `case`, `cast`, `extract`)**: `https://docs.sqlalchemy.org/en/20/core/sqlelement.html`
- **Referencja operatorów** (pełna tabela `like`, `in_`, `between`, `is_`): `https://docs.sqlalchemy.org/en/20/core/operators.html`
- **Typy danych i ich zachowanie na dialektach**: `https://docs.sqlalchemy.org/en/20/core/types.html`
- **Dialekt SQLite — różnice i ograniczenia**: `https://docs.sqlalchemy.org/en/20/dialects/sqlite.html`
- **Przewodnik migracji 1.x → 2.0** (dlaczego `engine.execute()` zniknęło): `https://docs.sqlalchemy.org/en/20/changelog/migration_20.html`
- **Słownik pojęć SQLAlchemy**: `https://docs.sqlalchemy.org/en/20/glossary.html`

---

## Co dalej

Wiesz już, jak **czytać** dane. W module 05 zajmiemy się ich **zmienianiem**: `insert()`, `update()`, `delete()`, `RETURNING` oraz transakcje w Core — z pełnym wyjaśnieniem, jak wstawić tysiąc wierszy w jednej rundzie i jak zrobić idempotentny import, którego można bezpiecznie uruchomić dwa razy.

➡️ **[`05_dml.md`](05_dml.md) — Zapisywanie danych: insert, update, delete**

<!-- koniec modułu 04 -->