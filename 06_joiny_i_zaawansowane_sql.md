Zanim napiszę moduł, zweryfikuję w sandboxie, że cały kod i wygenerowany SQL faktycznie działają — chcę pokazać czytelnikowi prawdziwe wyniki, nie zgadywane.



Weryfikacja: SQLite 3.50.4 (obsługuje CTE rekurencyjne i funkcje okna), więc wszystkie zapytania z tego modułu uruchomią się lokalnie. Poniżej kompletny plik.

---


# Moduł 06 — JOIN-y, podzapytania, CTE i funkcje okna

Do tej pory każde Twoje zapytanie dotyczyło jednej tabeli. W prawdziwej bazie dane są jednak rozłożone na wiele tabel — i to celowo. W tym module nauczysz się składać je z powrotem w całość: poznasz `join()` i jego warianty, podzapytania, wspólne wyrażenia tablicowe (CTE) oraz funkcje okna. Na końcu napiszesz pełny raport sprzedażowy — taki, który w ORM byłby niewygodny, a w Core jest naturalny. Zrozumiesz też, dlaczego *gdzie* umieścisz warunek (w `ON` czy w `WHERE`) potrafi zmienić sens zapytania, mimo że SQL wygląda prawie identycznie.

---

| | |
|---|---|
| **Poziom** | 🟡 średni |
| **Czas** | ~180 minut |
| **Wymagania wstępne** | [`03_metadata_ddl.md`](03_metadata_ddl.md) (definiowanie tabel i klucze obce), [`04_select_i_result.md`](04_select_i_result.md) (`select()`, filtry, grupowanie), [`05_dml.md`](05_dml.md) (transakcje i wstawianie danych) |
| **Czego dotyczy plik** | Łączenie tabel (`join`, `outerjoin`), aliasy i self-join, podzapytania (`exists`, `in_`, podzapytanie skalarne), CTE zwykłe i rekurencyjne, operacje zbiorowe, funkcje okna, `LATERAL`, czytanie planów zapytań |
| **Czego NIE ma w tym pliku** | ORM (relacje `relationship`, `joinedload`, N+1) — to moduły [07](07_modele_deklaratywne.md)–[11](11_ladowanie_i_n_plus_1.md). Tutaj pracujemy wyłącznie na `Table`, `select()` i `Connection`. |

## Spis treści

- [1. Relacje w danych — powtórka od zera](#1-relacje-w-danych--powtórka-od-zera)
- [2. `join()` — łączenie tabel](#2-join--łączenie-tabel)
- [3. JOIN-y zewnętrzne: LEFT, FULL i sprawa RIGHT](#3-join-y-zewnętrzne-left-full-i-sprawa-right)
- [4. `alias()` — porównywanie wierszy tej samej tabeli](#4-alias--porównywanie-wierszy-tej-samej-tabeli)
- [5. Podzapytania](#5-podzapytania)
- [6. CTE — wspólne wyrażenia tablicowe](#6-cte--wspólne-wyrażenia-tablicowe)
- [7. Operacje zbiorowe: `union`, `intersect`, `except_`](#7-operacje-zbiorowe-union-intersect-except_)
- [8. Funkcje okna](#8-funkcje-okna)
- [9. `LATERAL JOIN`](#9-lateral-join)
- [10. Praktyka inżynierska: plan zapytania, N+1, czytelność](#10-praktyka-inżynierska-plan-zapytania-n1-czytelność)
- [11. Raport sprzedażowy — kompletny przykład](#11-raport-sprzedażowy--kompletny-przykład)
- [Podsumowanie](#podsumowanie)
- [Ćwiczenia](#ćwiczenia)
- [Najczęstsze błędy i jak je czytać](#najczęstsze-błędy-i-jak-je-czytać)
- [Słowniczek modułu](#słowniczek-modułu)
- [Dalsze czytanie](#dalsze-czytanie)
- [Co dalej](#co-dalej)

---

## 1. Relacje w danych — powtórka od zera

### 1.1 Dlaczego dane są rozbite na katalogi

Wyobraź sobie szafkę z aktami w wypożyczalni. Gdybyś trzymał wszystko w jednym wielkim zeszycie, każdy nowy klient oznaczałby przepisanie zeszytu od nowa. Zamiast tego prowadzisz kilka zeszytów: jeden z klientami, jeden z produktami, jeden z zamówieniami. Każdy zeszyt ma numery stron, a w zeszycie „zamówienia” zapisujesz tylko numer klienta i numer produktu — nie przepisujesz jego nazwiska ani adresu.

To dokładnie robi baza danych. Tabele to zeszyty, a kolumny służące do odsyłania do innych zeszytów nazywamy **kluczami obcymi** (ang. *foreign keys*, w skrócie FK).

> 💡 **Analogia** — klucz obcy to numer telefonu, nie osoba. Jeśli w zeszycie zamówień zapiszesz „klient #2”, to nie znaczy, że skopiowałeś Piotra Nowaka. Zapisujesz tylko *wskaźnik* do niego. Gdy Piotr zmieni nazwisko, poprawiasz je w jednym miejscu i wszystkie zamówienia od razu pokazują nową wartość.

### 1.2 Trzy rodzaje relacji

**Jeden do wielu (1:N).** Jeden klient może mieć wiele zamówień. Każde zamówienie należy do dokładnie jednego klienta. Klucz obcy `orders.customer_id` siedzi po stronie „wielu” i wskazuje na `customers.id`. Tak jest w 90% przypadków i to jest relacja, którą poznasz najczęściej.

**Jeden do jednego (1:1).** Jeden klient ma jedno i tylko jedno konto lojalnościowe. Technicznie to to samo co 1:N, ale klucz obcy dostaje dodatkowo ograniczenie unikalności (`UNIQUE`), więc drugie konto dla tego samego klienta nie powstanie.

**Wiele do wielu (N:M).** Jeden produkt może mieć wiele tagów, a jeden tag może należeć do wielu produktów. W relacyjnej bazie nie da się zapisać tego bezpośrednio — potrzebna jest **tabela pośrednia** (ang. *association table* albo *junction table*), która ma po jednej kolumnie klucza obcego na każdą stronę. Nasza `product_tags` ma dwie kolumny, `product_id` i `tag_id`, i obie tworzą razem klucz główny (composite primary key) — dzięki temu ten sam tag nie zostanie dopisany do tego samego produktu dwa razy.

> 🧠 **Dlaczego tak jest** — tabela pośrednia nie ma własnego „identyfikatora”, bo nie reprezentuje żadnego obiektu ze świata rzeczywistego. Reprezentuje *fakt*: „produkt #5 ma tag #3”. Fakt jest prawdziwy albo nie — nie ma sensu mówić o nim dwa razy.

### 1.3 Schemat, na którym pracujemy

Rozbudowujemy sklep z modułu [03](03_metadata_ddl.md). Doszły trzy rzeczy: drzewo kategorii (kategoria może mieć nadrzędną kategorię), tagi w relacji N:M oraz tabela pracowników z przełożonym.

```text
                    ┌────────────┐
                    │ categories │◄──── parent_id (klucz obcy do siebie samej)
                    └─────┬──────┘
                          │ 1:N
                          ▼
   ┌──────────┐     ┌────────────┐
   │   tags   │◄───►│  products  │
   └──────────┘     └─────┬──────┘
     N:M przez              │ 1
     product_tags           │
                            │ N
                    ┌───────┴─────┐        ┌───────────┐
                    │ order_items │        │ employees │◄── manager_id (klucz obcy do siebie)
                    └───────┬─────┘        └───────────┘
                            │ N
                            │
                            │ 1
                    ┌───────┴─────┐
                    │   orders    │
                    └───────┬─────┘
                            │ N
                            │
                            │ 1
                    ┌───────┴─────┐
                    │  customers  │
                    └─────────────┘
```

Definicja schematu w kodzie. Zwróć uwagę, że w `categories.parent_id` wskazujemy na tę samą tabelę, którą definiujemy — SQLAlchemy przyjmuje nazwę jako tekst, więc kolejność nie ma znaczenia.

```python
# examples/06_shop_schema.py
"""Schemat sklepu: rozszerzenie z modułu 03 o drzewo kategorii, tagi i pracowników."""

from __future__ import annotations

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
    UniqueConstraint,
    func,
    text,
)

metadata = MetaData(
    naming_convention={
        "ix": "ix_%(column_0_label)s",
        "uq": "uq_%(table_name)s_%(column_0_name)s",
        "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
        "pk": "pk_%(table_name)s",
    }
)

categories = Table(
    "categories",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String(80), nullable=False),
    # Klucz obcy do tej samej tabeli: kategoria może mieć kategoria nadrzędną.
    # NULL oznacza "kategoria główna, np. Elektronika".
    Column("parent_id", Integer, ForeignKey("categories.id"), nullable=True),
)

customers = Table(
    "customers",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("email", String(255), nullable=False, unique=True),
    Column("full_name", String(120), nullable=False),
    Column("city", String(80), nullable=False),
    Column("created_at", DateTime, nullable=False, server_default=func.now()),
)

products = Table(
    "products",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("sku", String(32), nullable=False, unique=True),
    Column("name", String(160), nullable=False),
    Column("category_id", Integer, ForeignKey("categories.id"), nullable=False),
    Column("price", Numeric(10, 2), nullable=False),
    Column("is_active", Boolean, nullable=False, server_default=text("1")),
)

orders = Table(
    "orders",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("customer_id", Integer, ForeignKey("customers.id"), nullable=False),
    Column("status", String(20), nullable=False, server_default=text("'new'")),
    Column("placed_at", DateTime, nullable=False),
)

order_items = Table(
    "order_items",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("order_id", Integer, ForeignKey("orders.id", ondelete="CASCADE"), nullable=False),
    Column("product_id", Integer, ForeignKey("products.id"), nullable=False),
    Column("quantity", Integer, nullable=False),
    # Cena zapisana w momencie zakupu. Celowo NIE odczytujemy products.price:
    # cennik się zmienia, a historia zamówień musi zostać niezmienna.
    Column("unit_price", Numeric(10, 2), nullable=False),
    UniqueConstraint("order_id", "product_id", name="uq_order_items_order_product"),
)

employees = Table(
    "employees",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("full_name", String(120), nullable=False),
    Column("manager_id", Integer, ForeignKey("employees.id"), nullable=True),
    Column("salary", Numeric(10, 2), nullable=False),
)

tags = Table(
    "tags",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String(40), nullable=False, unique=True),
)

# Tabela pośrednia N:M. Brak kolumny "id" — klucz główny to para (product_id, tag_id).
product_tags = Table(
    "product_tags",
    metadata,
    Column("product_id", Integer, ForeignKey("products.id", ondelete="CASCADE"), primary_key=True),
    Column("tag_id", Integer, ForeignKey("tags.id", ondelete="CASCADE"), primary_key=True),
)
```

### 1.4 Dane demonstracyjne

Uruchom ten skrypt raz — utworzy plik `shop.db` i wypełni go danymi. Wszystkie kolejne przykłady w module zakładają, że ten plik istnieje.

```python
# examples/06_setup.py
"""Tworzy shop.db i wypełnia go danymi. Uruchom raz przed resztą przykładów."""

from __future__ import annotations

from datetime import datetime

from sqlalchemy import create_engine, insert

from shop_schema import (
    categories,
    customers,
    employees,
    metadata,
    order_items,
    orders,
    product_tags,
    products,
    tags,
)

engine = create_engine("sqlite:///shop.db")

metadata.drop_all(engine)
metadata.create_all(engine)

with engine.begin() as conn:
    conn.execute(
        insert(categories),
        [
            {"id": 1, "name": "Elektronika", "parent_id": None},
            {"id": 2, "name": "Komputery", "parent_id": 1},
            {"id": 3, "name": "Laptopy", "parent_id": 2},
            {"id": 4, "name": "Monitory", "parent_id": 2},
            {"id": 5, "name": "Audio", "parent_id": 1},
            {"id": 6, "name": "Słuchawki", "parent_id": 5},
            {"id": 7, "name": "Dom i ogród", "parent_id": None},
            {"id": 8, "name": "Kuchnia", "parent_id": 7},
            {"id": 9, "name": "Ekspresy", "parent_id": 8},
        ],
    )
    conn.execute(
        insert(customers),
        [
            {"id": 1, "email": "anna@example.com", "full_name": "Anna Kowalska", "city": "Kraków"},
            {"id": 2, "email": "piotr@example.com", "full_name": "Piotr Nowak", "city": "Warszawa"},
            {"id": 3, "email": "magda@example.com", "full_name": "Magdalena Wiśniewska", "city": "Gdańsk"},
            {"id": 4, "email": "tomasz@example.com", "full_name": "Tomasz Zieliński", "city": "Kraków"},
            # Ewa celowo nie ma żadnego zamówienia — posłuży do ćwiczeń z LEFT JOIN.
            {"id": 5, "email": "ewa@example.com", "full_name": "Ewa Mazur", "city": "Poznań"},
        ],
    )
    conn.execute(
        insert(products),
        [
            {"id": 1, "sku": "LAP-001", "name": "Lenovo ThinkPad E14", "category_id": 3, "price": 4299, "is_active": True},
            {"id": 2, "sku": "LAP-002", "name": "Dell XPS 13", "category_id": 3, "price": 6499, "is_active": True},
            {"id": 3, "sku": "MON-001", "name": "Dell U2723QE", "category_id": 4, "price": 2899, "is_active": True},
            # Produkt wycofany ze sprzedaży — wciąż występuje w historii zamówień.
            {"id": 4, "sku": "MON-002", "name": "LG 27UP850", "category_id": 4, "price": 2399, "is_active": False},
            {"id": 5, "sku": "AUD-001", "name": "Sony WH-1000XM5", "category_id": 6, "price": 1499, "is_active": True},
            {"id": 6, "sku": "AUD-002", "name": "Jabra Evolve2 65", "category_id": 6, "price": 899, "is_active": True},
            {"id": 7, "sku": "KUC-001", "name": "Ekspres De'Longhi Magnifica", "category_id": 9, "price": 2199, "is_active": True},
            {"id": 8, "sku": "KUC-002", "name": "Czajnik Philips", "category_id": 8, "price": 199, "is_active": True},
        ],
    )
    conn.execute(
        insert(orders),
        [
            {"id": 1, "customer_id": 1, "status": "paid", "placed_at": datetime(2026, 1, 15, 10, 0)},
            {"id": 2, "customer_id": 1, "status": "paid", "placed_at": datetime(2026, 2, 3, 12, 30)},
            {"id": 3, "customer_id": 2, "status": "paid", "placed_at": datetime(2026, 2, 20, 9, 15)},
            {"id": 4, "customer_id": 2, "status": "cancelled", "placed_at": datetime(2026, 3, 5, 16, 45)},
            {"id": 5, "customer_id": 3, "status": "paid", "placed_at": datetime(2026, 3, 18, 11, 0)},
            {"id": 6, "customer_id": 3, "status": "paid", "placed_at": datetime(2026, 4, 2, 8, 0)},
            {"id": 7, "customer_id": 4, "status": "new", "placed_at": datetime(2026, 4, 22, 14, 20)},
            {"id": 8, "customer_id": 1, "status": "paid", "placed_at": datetime(2026, 5, 10, 13, 0)},
            {"id": 9, "customer_id": 2, "status": "paid", "placed_at": datetime(2026, 6, 1, 10, 40)},
        ],
    )
    conn.execute(
        insert(order_items),
        [
            {"order_id": 1, "product_id": 1, "quantity": 1, "unit_price": 4299},
            {"order_id": 1, "product_id": 5, "quantity": 1, "unit_price": 1499},
            {"order_id": 2, "product_id": 3, "quantity": 2, "unit_price": 2899},
            {"order_id": 3, "product_id": 2, "quantity": 1, "unit_price": 6499},
            {"order_id": 3, "product_id": 6, "quantity": 3, "unit_price": 899},
            {"order_id": 4, "product_id": 5, "quantity": 1, "unit_price": 1499},
            {"order_id": 5, "product_id": 7, "quantity": 1, "unit_price": 2199},
            {"order_id": 5, "product_id": 8, "quantity": 2, "unit_price": 199},
            {"order_id": 6, "product_id": 1, "quantity": 1, "unit_price": 4299},
            {"order_id": 6, "product_id": 6, "quantity": 2, "unit_price": 899},
            {"order_id": 7, "product_id": 3, "quantity": 1, "unit_price": 2899},
            {"order_id": 8, "product_id": 5, "quantity": 2, "unit_price": 1499},
            {"order_id": 9, "product_id": 2, "quantity": 1, "unit_price": 6499},
            {"order_id": 9, "product_id": 4, "quantity": 1, "unit_price": 2399},
        ],
    )
    conn.execute(
        insert(employees),
        [
            {"id": 1, "full_name": "Alicja Bąk", "manager_id": None, "salary": 15000},
            {"id": 2, "full_name": "Bartek Cichy", "manager_id": 1, "salary": 9500},
            {"id": 3, "full_name": "Celina Duda", "manager_id": 1, "salary": 10200},
            {"id": 4, "full_name": "Damian Ejsmont", "manager_id": 2, "salary": 7200},
            {"id": 5, "full_name": "Ela Fijałkowska", "manager_id": 2, "salary": 6800},
            # Filip zarabia więcej niż jego przełożona — do ćwiczenia z self-join.
            {"id": 6, "full_name": "Filip Górski", "manager_id": 3, "salary": 11000},
        ],
    )
    conn.execute(
        insert(tags),
        [
            {"id": 1, "name": "biznes"},
            {"id": 2, "name": "premium"},
            {"id": 3, "name": "audio"},
            {"id": 4, "name": "dom"},
        ],
    )
    conn.execute(
        insert(product_tags),
        [
            {"product_id": 1, "tag_id": 1},
            {"product_id": 1, "tag_id": 2},
            {"product_id": 2, "tag_id": 2},
            {"product_id": 3, "tag_id": 1},
            {"product_id": 5, "tag_id": 3},
            {"product_id": 6, "tag_id": 3},
            {"product_id": 7, "tag_id": 4},
            {"product_id": 8, "tag_id": 4},
        ],
    )

print("shop.db gotowe")
```

**Jak uruchomić:**

```bash
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install "SQLAlchemy>=2.0"
python 06_setup.py
```

> 🧠 **Dlaczego tak jest** — w danych zapisałem `order_items.unit_price` zamiast odwoływać się do `products.price`. To nie nadmiarowość, a celowa **denormalizacja historii**. Gdyby cena Sony wzrosła do 1699 zł, raport za styczeń nagle pokazałby inne przychody, niż faktycznie wpłynęły. Wiersz zamówienia to zapis faktu z przeszłości i nie wolno mu się zmieniać razem z cennikiem.

---

## 2. `join()` — łączenie tabel

### 2.1 Problem: dane są rozjechane

Powiedzmy, że chcesz wypisać listę zamówień z nazwiskiem klienta. Nazwisko siedzi w `customers`, numer zamówienia w `orders`. Bez łączenia masz dwie opcje, obie złe:

1. Pobrać wszystkie zamówienia, potem dla każdego osobne zapytanie o klienta — to 1 + N zapytań (klasyczny N+1, wrócimy do niego w sekcji 10 i w module [11](11_ladowanie_i_n_plus_1.md)).
2. Pobrać obie tabele i dopasować je w Pythonie pętlą — czyli ręcznie zaimplementować to, co baza robi szybciej i na dysku, nie w pamięci.

`join()` mówi bazie: „połącz te dwa zbiory wierszy według podanego warunku i zwróć mi wynik jako jedną tabelę”.

> 💡 **Analogia** — JOIN to zszywanie dwóch rozciętych połówek tej samej fotografii. Każda połowa osobno jest bezużyteczna. Warunek `ON` to linia cięcia: mówi, które kawałki do siebie pasują. `customers.id = orders.customer_id` znaczy „dopasuj po numerze klienta”.

### 2.2 Jawny warunek: `onclause`

Najbardziej czytelna postać — sam podajesz warunek łączenia.

```python
# examples/06_join_basic.py
from __future__ import annotations

from sqlalchemy import create_engine, select

from shop_schema import customers, orders

engine = create_engine("sqlite:///shop.db")

stmt = (
    select(customers.c.full_name, orders.c.id.label("order_id"), orders.c.status)
    .join(orders, customers.c.id == orders.c.customer_id)
    .order_by(orders.c.id)
)

# 🔬 Pod maską: SQL wygenerowany przez SQLAlchemy
print(stmt.compile(engine, compile_kwargs={"literal_binds": True}))

with engine.connect() as conn:
    for row in conn.execute(stmt):
        print(row)
```

Wynik:

```sql
SELECT customers.full_name, orders.id AS order_id, orders.status
FROM customers JOIN orders ON customers.id = orders.customer_id
ORDER BY orders.id
```

```text
('Anna Kowalska', 1, 'paid')
('Anna Kowalska', 2, 'paid')
('Piotr Nowak', 3, 'paid')
('Piotr Nowak', 4, 'cancelled')
('Magdalena Wiśniewska', 5, 'paid')
('Magdalena Wiśniewska', 6, 'paid')
('Tomasz Zieliński', 7, 'new')
('Anna Kowalska', 8, 'paid')
('Piotr Nowak', 9, 'paid')
```

Zauważ dwie rzeczy. Po pierwsze: `JOIN` bez słowa `INNER` to domyślnie `INNER JOIN` — łączymy tylko wiersze, które mają parę po obu stronach. Po drugie: nazwisko Anny pojawia się trzy razy, bo Anna ma trzy zamówienia. JOIN **nie scala** wierszy, on je **rozmnaża** — jeden wiersz po lewej razy pasujące wiersze po prawej.

> ⚠️ **Pułapka** — jeśli napiszesz `select(customers.c.full_name, orders.c.id)` i zapomnisz o `.join()`, SQLAlchemy wygeneruje `FROM customers, orders` — czyli **iloczyn kartezjański**. Każdy z 5 klientów zostanie połączony z każdym z 9 zamówień: 45 wierszy bez żadnego sensu. Baza nie zaprotestuje, bo składnia jest poprawna. Zawsze sprawdzaj, czy w wygenerowanym SQL jest `JOIN ... ON ...`.

### 2.3 Natural join: SQLAlchemy sam znajdzie warunek

Skoro klucz obcy `orders.customer_id` wskazuje na `customers.id`, SQLAlchemy potrafi odtworzyć warunek samodzielnie. Wystarczy podać tabelę, bez `onclause`.

```python
# examples/06_join_implicit.py
from __future__ import annotations

from sqlalchemy import create_engine, select

from shop_schema import customers, orders

engine = create_engine("sqlite:///shop.db")

# Bez onclause — SQLAlchemy odczyta warunek z kluczy obcych.
stmt = (
    select(customers.c.full_name, orders.c.id.label("order_id"))
    .select_from(customers.join(orders))
    .order_by(orders.c.id)
)

print(stmt.compile(engine, compile_kwargs={"literal_binds": True}))
```

```sql
SELECT customers.full_name, orders.id AS order_id
FROM customers JOIN orders ON customers.id = orders.customer_id
ORDER BY orders.id
```

SQL jest identyczny jak w 2.2. Różnica jest w Twojej głowie, nie w bazie.

> 💡 **Analogia** — jawny `onclause` to „złącz te dwa dokumenty po numerze PESEL”. Natural join to „złącz je tak, jak są ze sobą powiązane w systemie”. Pierwsze jest odporne na zmiany, drugie wygodniejsze, dopóki nikt nie doda drugiego klucza obcego między tymi samymi tabelami.

> ⚠️ **Pułapka** — natural join działa tylko wtedy, gdy między tabelami istnieje **dokładnie jeden** klucz obcy. Wyobraź sobie, że dodajesz do zamówień `shipping_address_id` i `billing_address_id`, oba wskazujące na `addresses`. SQLAlchemy zgłosi wtedy `AmbiguousForeignKeysError` — i dobrze, bo sam nie wie, o którą relację Ci chodzi. W takich sytuacjach zawsze pisz jawny `onclause`. To samo dotyczy `employees.manager_id` wracającego do `employees.id`: dwie kolumny w jednej tabeli, więc warunek trzeba podać ręcznie.

### 2.4 Kolejność tabel i `join_from()`

`.join()` dokłada tabelę do tego, co już jest w klauzuli `FROM`. Działa to intuicyjnie, dopóki nie musisz zacząć od tabeli, która nie jest pierwszą w `select()`. Wtedy przydaje się `.select_from()` albo `.join_from()`.

```python
# examples/06_join_from.py
from __future__ import annotations

from sqlalchemy import create_engine, select

from shop_schema import customers, orders

engine = create_engine("sqlite:///shop.db")

# Wariant A: select_from() ustala punkt startowy, potem dokładamy join().
stmt_a = (
    select(customers.c.full_name, orders.c.id.label("order_id"))
    .select_from(customers)
    .join(orders, customers.c.id == orders.c.customer_id)
)

# Wariant B: join_from() mówi wprost "od tego do tego".
stmt_b = select(customers.c.full_name, orders.c.id.label("order_id")).join_from(
    customers, orders, customers.c.id == orders.c.customer_id
)

print(stmt_a.compile(engine, compile_kwargs={"literal_binds": True}))
print("---")
print(stmt_b.compile(engine, compile_kwargs={"literal_binds": True}))
```

Oba generują to samo `FROM customers JOIN orders ON ...`. Zasada praktyczna: używaj `.join()`, a `.join_from()` sięgnij wtedy, gdy kolejność łączenia musi być inna niż kolejność kolumn w `select()` — na przykład gdy chcesz zacząć od `orders`, choć w `SELECT` wymieniasz kolumny klientów.

### 2.5 Łączenie trzech i więcej tabel

Każde kolejne `.join()` dokłada jedną tabelę. Łańcuch czyta się od lewej do prawej.

```python
# examples/06_join_three.py
from __future__ import annotations

from sqlalchemy import create_engine, select

from shop_schema import customers, order_items, orders, products

engine = create_engine("sqlite:///shop.db")

stmt = (
    select(
        orders.c.id.label("order_id"),
        customers.c.full_name,
        products.c.name.label("product"),
        order_items.c.quantity,
        order_items.c.unit_price,
    )
    .select_from(
        orders.join(customers).join(order_items).join(products)
    )
    .order_by(orders.c.id, products.c.name)
)

print(stmt.compile(engine, compile_kwargs={"literal_binds": True}))
```

```sql
SELECT orders.id AS order_id, customers.full_name, products.name AS product,
       order_items.quantity, order_items.unit_price
FROM orders
JOIN customers ON customers.id = orders.customer_id
JOIN order_items ON orders.id = order_items.order_id
JOIN products ON products.id = order_items.product_id
ORDER BY orders.id, products.name
```

> 🧠 **Dlaczego tak jest** — łańcuch `orders.join(customers).join(order_items).join(products)` czyta się jak zdanie: „weź zamówienia, dołącz do nich klientów, do tego dołącz pozycje, do tego dołącz produkty”. Każde ogniwo dokłada jedną tabelę do bieżącego zbioru. Alternatywa, `orders.join(customers, order_items, products)`, też istnieje, ale jest mniej czytelna i częściej prowadzi do pomyłek.

### 2.6 Warunek w `ON` czy w `WHERE`? Różnica, która zmienia sens

To najważniejszy podrozdział tego modułu. Przy `INNER JOIN` kolejność nie ma znaczenia — wynik będzie ten sam. Przy `LEFT JOIN` **ma znaczenie fundamentalne**, a różnica w wyglądzie kodu to jedno słowo.

Zacznijmy od pytania biznesowego: *„pokaż mi wszystkich klientów i ich zamówienia o statusie `paid`”*. Klienci bez płatnych zamówień mają się pojawić z pustym miejscem.

```python
# examples/06_on_vs_where.py
from __future__ import annotations

from sqlalchemy import create_engine, select

from shop_schema import customers, orders

engine = create_engine("sqlite:///shop.db")

# WARIANT A (poprawny): warunek w ON.
# LEFT OUTER JOIN customers -> orders, ale tylko dla zamówień 'paid'.
stmt_a = (
    select(customers.c.full_name, orders.c.id.label("order_id"), orders.c.status)
    .select_from(
        customers.outerjoin(orders, (customers.c.id == orders.c.customer_id) & (orders.c.status == "paid"))
    )
    .order_by(customers.c.id, orders.c.id)
)

# WARIANT B (błędny): ten sam warunek przeniesiony do WHERE.
stmt_b = (
    select(customers.c.full_name, orders.c.id.label("order_id"), orders.c.status)
    .select_from(customers.outerjoin(orders, customers.c.id == orders.c.customer_id))
    .where(orders.c.status == "paid")
    .order_by(customers.c.id, orders.c.id)
)

print("--- WARIANT A ---")
print(stmt_a.compile(engine, compile_kwargs={"literal_binds": True}))
with engine.connect() as conn:
    for row in conn.execute(stmt_a):
        print("  ", row)

print("--- WARIANT B ---")
print(stmt_b.compile(engine, compile_kwargs={"literal_binds": True}))
with engine.connect() as conn:
    for row in conn.execute(stmt_b):
        print("  ", row)
```

```sql
--- WARIANT A ---
SELECT customers.full_name, orders.id AS order_id, orders.status
FROM customers LEFT OUTER JOIN orders
  ON customers.id = orders.customer_id AND orders.status = 'paid'
ORDER BY customers.id, orders.id

--- WARIANT B ---
SELECT customers.full_name, orders.id AS order_id, orders.status
FROM customers LEFT OUTER JOIN orders ON customers.id = orders.customer_id
WHERE orders.status = 'paid'
ORDER BY customers.id, orders.id
```

Wyniki:

```text
WARIANT A — 9 wierszy
('Anna Kowalska', 1, 'paid')
('Anna Kowalska', 2, 'paid')
('Anna Kowalska', 8, 'paid')
('Piotr Nowak', 3, 'paid')
('Piotr Nowak', 9, 'paid')
('Magdalena Wiśniewska', 5, 'paid')
('Magdalena Wiśniewska', 6, 'paid')
('Tomasz Zieliński', None, None)   ← ma tylko zamówienie 'new', więc zostaje z pustym miejscem
('Ewa Mazur', None, None)          ← w ogóle nie ma zamówień, ale JEST na liście

WARIANT B — 7 wierszy
('Anna Kowalska', 1, 'paid')
('Anna Kowalska', 2, 'paid')
('Anna Kowalska', 8, 'paid')
('Piotr Nowak', 3, 'paid')
('Piotr Nowak', 9, 'paid')
('Magdalena Wiśniewska', 5, 'paid')
('Magdalena Wiśniewska', 6, 'paid')
```

Tomasz i Ewa zniknęli. Dlaczego?

> 🧠 **Dlaczego tak jest** — `LEFT OUTER JOIN` działa w dwóch krokach. Najpierw dla każdego wiersza z lewej strony szuka pasujących wierszy z prawej. Jeśli ich nie znajdzie, **wstawia wiersz złożony z NULL-i**, żeby lewa strona nie przepadła. I dopiero potem — jako *drugi* krok — działa `WHERE`, który odrzuca wiersze niespełniające warunku. A `NULL = 'paid'` jest w SQL *nieprawdą* (a nie fałszem — to trzeci stan logiczny, `UNKNOWN`), więc wiersz z NULL-ami leci do kosza. Efekt: filtr w `WHERE` po prawej tabeli **kasuje cały sens LEFT JOIN** i zamienia go z powrotem w INNER JOIN.

Reguła do zapamiętania na całe życie:

> ⚠️ **Pułapka** — warunek dotyczący **prawej** tabeli w `LEFT JOIN` musi trafić do `ON`. Warunek dotyczący **lewej** tabeli może iść do `WHERE` (bo lewa strona zawsze ma wiersz). Warunek „prawa strona jest pusta” (np. `orders.c.id.is_(None)`) z definicji musi iść do `WHERE` — to właśnie po to robimy LEFT JOIN.

> 🧪 **Ćwiczenie** — spróbuj przenieść `orders.c.status == "paid"` z `ON` do `WHERE` w wariancie A i sprawdź, czy dostaniesz dokładnie to, co w wariancie B. Zgadnij wynik, zanim uruchomisz kod.

---

## 3. JOIN-y zewnętrzne: LEFT, FULL i sprawa RIGHT

### 3.1 Cztery smaki złączeń

| Wariant | Co zwraca | SQL | SQLAlchemy |
|---|---|---|---|
| `INNER JOIN` | tylko pary po obu stronach | `JOIN ... ON ...` | `join()` |
| `LEFT OUTER JOIN` | wszystko z lewej + pary z prawej (albo NULL) | `LEFT OUTER JOIN ... ON ...` | `outerjoin()` lub `join(..., isouter=True)` |
| `FULL OUTER JOIN` | wszystko z obu stron, niedopasowane jako NULL | `FULL OUTER JOIN ... ON ...` | `join(..., full=True)` |
| `RIGHT OUTER JOIN` | wszystko z prawej + pary z lewej | `RIGHT OUTER JOIN ... ON ...` | — patrz 3.4 |

> 💡 **Analogia** — wyobraź sobie dwie listy gości na dwóch przyjęciach. INNER JOIN to osoby, które były na obu. LEFT JOIN to wszyscy z pierwszego przyjęcia plus informacja, kto z nich był też na drugim. FULL JOIN to suma wszystkich gości z obu list, z adnotacją, gdzie kogo widziano. RIGHT JOIN to to samo co LEFT JOIN, tylko patrzysz z drugiej listy.

### 3.2 `outerjoin()` i `isouter=True`

Trzy równoważne zapisy tego samego. Wybierz jeden i trzymaj się go w projekcie.

```python
# examples/06_outerjoin.py
from __future__ import annotations

from sqlalchemy import create_engine, select

from shop_schema import customers, orders

engine = create_engine("sqlite:///shop.db")

# 1. Metoda outerjoin() — najbardziej czytelna.
stmt_1 = select(customers.c.full_name, orders.c.id).select_from(
    customers.outerjoin(orders, customers.c.id == orders.c.customer_id)
)

# 2. Parametr isouter=True.
stmt_2 = select(customers.c.full_name, orders.c.id).select_from(
    customers.join(orders, customers.c.id == orders.c.customer_id, isouter=True)
)

# 3. Z perspektywy tabeli: customers.left_outer_join(orders, ...) też istnieje,
#    ale nie jest potrzebne — outerjoin() wystarcza.

print(stmt_1.compile(engine, compile_kwargs={"literal_binds": True}))
print("---")
print(stmt_2.compile(engine, compile_kwargs={"literal_binds": True}))
```

```sql
SELECT customers.full_name, orders.id
FROM customers LEFT OUTER JOIN orders ON customers.id = orders.customer_id
```

### 3.3 `FULL OUTER JOIN` i `full=True`

`FULL OUTER JOIN` przydaje się przy porównywaniu dwóch źródeł danych: „co jest w systemie A, czego nie ma w B, i odwrotnie”.

```python
# examples/06_full_join.py
from __future__ import annotations

from sqlalchemy import create_engine, select

from shop_schema import customers, orders

engine = create_engine("sqlite:///shop.db")

stmt = select(customers.c.full_name, orders.c.id.label("order_id")).select_from(
    customers.join(orders, customers.c.id == orders.c.customer_id, full=True)
)

# FULL OUTER JOIN nie istnieje w SQLite — kompilujemy dla PostgreSQL,
# żeby zobaczyć SQL, jaki poleciałby do produkcyjnej bazy.
from sqlalchemy.dialects import postgresql

print(stmt.compile(dialect=postgresql.dialect(), compile_kwargs={"literal_binds": True}))
```

```sql
SELECT customers.full_name, orders.id AS order_id
FROM customers FULL OUTER JOIN orders ON customers.id = orders.customer_id
```

> 🆕 **SQLAlchemy 2.1** — parametr `full=True` oraz towarzysząca mu metoda `Select.full_join()` są dostępne od wersji 2.0.1. Jeśli pracujesz na wcześniejszym wydaniu z gałęzi 2.0, użyj `full_join()` tylko wtedy, gdy wersja na to pozwala — sprawdź `sqlalchemy.__version__`.

> ⚠️ **Pułapka** — `FULL OUTER JOIN` obsługują PostgreSQL, Oracle i SQL Server. **SQLite go nie ma** i zgłosi `OperationalError: RIGHT and FULL OUTER JOINs are not currently supported`. To jeden z tych momentów, w których „działa u mnie lokalnie” bierze się z tego, że akurat nie uruchomiłeś tego fragmentu. Testy na SQLite i produkcja na PostgreSQL to najczęstsze źródło takich niespodzianek — wrócimy do tego w module [18](18_testowanie.md).

### 3.4 Sprawa `RIGHT JOIN` — i dlaczego go nie potrzebujesz

Wiele osób szuka w SQLAlchemy `rightjoin()` albo `join(..., right=True)` i nie znajduje. To nie przeoczenie twórców.

> 🧠 **Dlaczego tak jest** — `A RIGHT OUTER JOIN B` to dokładnie to samo co `B LEFT OUTER JOIN A`. Zamiana stron to cała różnica. Zamiast dodawać parametr, który dałby dwa sposoby na zapisanie identycznego zapytania, SQLAlchemy każe Ci po prostu zamienić tabele miejscami. W bibliotece jest jedna droga do celu, a nie dwie.

Jeśli naprawdę chcesz zobaczyć `RIGHT OUTER JOIN` w SQL, zapisz to od prawej strony:

```python
# examples/06_right_as_left.py
from __future__ import annotations

from sqlalchemy import create_engine, select

from shop_schema import customers, orders

engine = create_engine("sqlite:///shop.db")

# "Wszystkie zamówienia, także te bez klienta" — zapisane od strony orders.
stmt = select(orders.c.id, customers.c.full_name).select_from(
    orders.outerjoin(customers, orders.c.customer_id == customers.c.id)
)

print(stmt.compile(engine, compile_kwargs={"literal_binds": True}))
```

```sql
SELECT orders.id, customers.full_name
FROM orders LEFT OUTER JOIN customers ON orders.customer_id = customers.id
```

> ⚠️ **Pułapka** — `Join` (obiekt reprezentujący złączenie) **ma** atrybuty `.left` i `.right`, ale oznaczają one „lewa i prawa strona tego złączenia”, a nie warianty JOIN-a. To częste nieporozumienie przy czytaniu dokumentacji. Jeśli widzisz `some_join.left`, to nie jest „left join”, tylko tabela po lewej stronie.

---

## 4. `alias()` — porównywanie wierszy tej samej tabeli

### 4.1 Po co komu druga kopia tej samej tabeli

Załóżmy, że chcesz wypisać każdego pracownika razem z imieniem i nazwiskiem jego przełożonego. Oba te dane siedzą w `employees`. Jak odróżnić „pracownika” od „przełożonego”, skoro to ta sama tabela?

> 💡 **Analogia** — wyobraź sobie, że na spotkaniu rodzinnym każdy ma na piersi dwie plakietki: „ja” i „moje dziecko”. Bez plakietek nie odróżnisz, o kim mówisz, gdy pada słowo „Bąk”. Alias to plakietka. `employees.alias("emp")` to „pracownik”, `employees.alias("boss")` to „przełożony” — ta sama tabela, dwie role.

### 4.2 Self-join: pracownik i jego przełożony

```python
# examples/06_self_join.py
from __future__ import annotations

from sqlalchemy import create_engine, select

from shop_schema import employees

engine = create_engine("sqlite:///shop.db")

emp = employees.alias("emp")
boss = employees.alias("boss")

stmt = (
    select(
        emp.c.full_name.label("pracownik"),
        boss.c.full_name.label("przełożony"),
    )
    .select_from(emp.outerjoin(boss, emp.c.manager_id == boss.c.id))
    .order_by(emp.c.id)
)

print(stmt.compile(engine, compile_kwargs={"literal_binds": True}))

with engine.connect() as conn:
    for row in conn.execute(stmt):
        print(row)
```

```sql
SELECT emp.full_name AS "pracownik", boss.full_name AS "przełożony"
FROM employees AS emp LEFT OUTER JOIN employees AS boss ON emp.manager_id = boss.id
ORDER BY emp.id
```

```text
('Alicja Bąk', None)
('Bartek Cichy', 'Alicja Bąk')
('Celina Duda', 'Alicja Bąk')
('Damian Ejsmont', 'Bartek Cichy')
('Ela Fijałkowska', 'Bartek Cichy')
('Filip Górski', 'Celina Duda')
```

Trzy szczegóły warte uwagi. Po pierwsze: użyliśmy `outerjoin`, nie `join`, bo Alicja nie ma przełożonego — przy INNER JOIN w ogóle nie pojawiłaby się na liście (dokładnie ta sama pułapka co w 2.6). Po drugie: `AS emp` i `AS boss` w SQL to nasze aliasy, nie wymysł SQLAlchemy — baza naprawdę widzi dwie kopie tabeli. Po trzecie: SQLAlchemy sam wygenerowało `LEFT OUTER JOIN`, mimo że używamy `.outerjoin()` na tabeli, a nie na `select()` — `Table.outerjoin()` zwraca obiekt `Join`, który zachowuje się jak tabela i można go użyć w `.select_from()`.

> 🧠 **Dlaczego tak jest** — w SQL nazwa tabeli musi być unikalna w obrębie zapytania, żeby baza wiedziała, o którą kolumnę chodzi. `employees.full_name` jest jednoznaczne tylko dopóki tabela występuje raz. W momencie gdy pojawia się drugi raz, `employees.full_name` przestaje cokolwiek znaczyć — dlatego baza wymaga aliasów. SQLAlchemy pomaga Ci to wymusić, bo `emp.c.full_name` to inny obiekt Pythona niż `boss.c.full_name`.

### 4.3 Porównywanie wierszy: kto zarabia więcej niż jego szef

Klasyczne zadanie, które pokazuje, że self-join to nie tylko „doklej kolumnę”, ale też porównywanie wartości w obrębie jednej tabeli.

```python
# examples/06_self_join_compare.py
from __future__ import annotations

from sqlalchemy import create_engine, select

from shop_schema import employees

engine = create_engine("sqlite:///shop.db")

emp = employees.alias("emp")
boss = employees.alias("boss")

stmt = (
    select(
        emp.c.full_name.label("pracownik"),
        emp.c.salary.label("pensja"),
        boss.c.full_name.label("przełożony"),
        boss.c.salary.label("pensja_szefa"),
    )
    .select_from(emp.join(boss, emp.c.manager_id == boss.c.id))
    .where(emp.c.salary > boss.c.salary)
)

print(stmt.compile(engine, compile_kwargs={"literal_binds": True}))

with engine.connect() as conn:
    for row in conn.execute(stmt):
        print(row)
```

```sql
SELECT emp.full_name AS "pracownik", emp.salary AS "pensja",
       boss.full_name AS "przełożony", boss.salary AS "pensja_szefa"
FROM employees AS emp JOIN employees AS boss ON emp.manager_id = boss.id
WHERE emp.salary > boss.salary
```

```text
('Filip Górski', Decimal('11000.00'), 'Celina Duda', Decimal('10200.00'))
```

> ⚠️ **Pułapka** — `.where(emp.c.salary > boss.c.salary)` porównuje kolumny **tej samej tabeli w dwóch rolach**, a nie wartość z wartością Pythona. To bardzo częste miejsce pomyłek przy przepisywaniu zapytań z „surowca”: ktoś szuka sposobu na podstawienie liczby i nie zauważa, że tutaj po obu stronach są kolumny. SQLAlchemy nie zgłosi błędu — oba obiekty są typu `Column`, operator `>` działa.

> 🧪 **Ćwiczenie** — wypisz pary pracowników, którzy mają tego samego przełożonego (czyli „kolegów z zespołu”). Podpowiedź: potrzebne będą dwa aliasy `employees` i warunek porównujący `manager_id`, plus `emp_a.c.id < emp_b.c.id`, żeby nie dostać każdej pary dwa razy i nie łączyć nikogo z samym sobą.

---

## 5. Podzapytania

Podzapytanie to zapytanie **wewnątrz** innego zapytania. W SQL ma trzy główne zastosowania, każde z własnym API w SQLAlchemy.

### 5.1 Podzapytanie skalarne — jedna wartość jako argument

Pytanie: *„które produkty są droższe od średniej ceny wszystkich produktów?”*. Nie znasz średniej z góry — musi ją policzyć baza.

```python
# examples/06_scalar_subquery.py
from __future__ import annotations

from sqlalchemy import create_engine, func, select

from shop_schema import products

engine = create_engine("sqlite:///shop.db")

# Podzapytanie zwracające DOKŁADNIE jedną kolumnę i DOKŁADNIE jeden wiersz.
avg_price = select(func.avg(products.c.price)).scalar_subquery()

stmt = (
    select(products.c.name, products.c.price)
    .where(products.c.price > avg_price)
    .order_by(products.c.price.desc())
)

print(stmt.compile(engine, compile_kwargs={"literal_binds": True}))

with engine.connect() as conn:
    for row in conn.execute(stmt):
        print(row)
```

```sql
SELECT products.name, products.price
FROM products
WHERE products.price > (SELECT avg(products.price) AS avg_1 FROM products)
ORDER BY products.price DESC
```

```text
('Dell XPS 13', Decimal('6499.00'))
('Lenovo ThinkPad E14', Decimal('4299.00'))
('Dell U2723QE', Decimal('2899.00'))
```

Średnia z cen to 2599,0 zł, więc zostają trzy najdroższe produkty.

> 🧠 **Dlaczego tak jest** — `.scalar_subquery()` to instrukcja dla SQLAlchemy: „traktuj to `SELECT` jako **jedną wartość**, nie jako zbiór wierszy”. Dlatego można go postawić po prawej stronie operatora `>`, tak jak liczbę. Gdybyś zapomniał o `.scalar_subquery()`, SQLAlchemy zgłosi `TypeError` — bo próbowałbyś porównać kolumnę z całym zbiorem wyników, a to nie ma sensu.

> ⚠️ **Pułapka** — podzapytanie skalarne **musi** zwracać najwyżej jeden wiersz. Jeśli zwróci dwa, PostgreSQL zgłosi `more than one row returned by a subquery used as an expression`, a SQLite po cichu weźmie pierwszy. Zawsze upewnij się, że w podzapytaniu jest agregacja (`avg`, `max`, `count`) albo `LIMIT 1`.

### 5.2 `IN` z podzapytaniem — i pułapka `NULL`

`IN` odpowiada na pytanie „czy ta wartość znajduje się na liście?”. Listą może być podzapytanie.

```python
# examples/06_in_subquery.py
from __future__ import annotations

from sqlalchemy import create_engine, select

from shop_schema import customers, orders

engine = create_engine("sqlite:///shop.db")

# Klienci, którzy kiedykolwiek złożyli zamówienie o statusie 'paid'.
stmt = (
    select(customers.c.full_name)
    .where(
        customers.c.id.in_(
            select(orders.c.customer_id).where(orders.c.status == "paid")
        )
    )
    .order_by(customers.c.full_name)
)

print(stmt.compile(engine, compile_kwargs={"literal_binds": True}))
```

```sql
SELECT customers.full_name
FROM customers
WHERE customers.id IN (SELECT orders.customer_id FROM orders WHERE orders.status = 'paid')
ORDER BY customers.full_name
```

> ⚠️ **Pułapka** — `NOT IN` (w SQLAlchemy: `.not_in(...)`) zachowuje się inaczej, niż intuicja podpowiada, gdy w podzapytaniu pojawi się `NULL`. `x NOT IN (1, 2, NULL)` **nie jest prawdą** dla żadnego `x` — jest `UNKNOWN`, czyli w praktyce odrzucone. Jeśli Twoja kolumna jest opcjonalna i którykolwiek wiersz ma tam `NULL`, `NOT IN` zwróci **zero wyników** i będziesz długo szukać przyczyny. Dlatego w praktyce do „klientów bez zamówień” używa się `NOT EXISTS` (patrz 5.3), a nie `NOT IN`. Dotyczy to zwłaszcza kolumn z kluczami obcymi, które mogą być puste.

### 5.3 `EXISTS` — pytanie „czy w ogóle istnieje”

`EXISTS` odpowiada na pytanie „czy istnieje choć jeden pasujący wiersz?”. Jest wyjątkowo wydajne, bo baza może przerwać skanowanie po pierwszym trafieniu, a w wielu bazach nie musi nawet czytać kolumn z `SELECT`.

```python
# examples/06_exists.py
from __future__ import annotations

from sqlalchemy import create_engine, exists, select

from shop_schema import customers, orders

engine = create_engine("sqlite:///shop.db")

# Klienci BEZ żadnego zamówienia.
stmt = (
    select(customers.c.id, customers.c.full_name)
    .where(
        ~exists(
            select(orders.c.id).where(orders.c.customer_id == customers.c.id)
        )
    )
    .order_by(customers.c.id)
)

print(stmt.compile(engine, compile_kwargs={"literal_binds": True}))

with engine.connect() as conn:
    for row in conn.execute(stmt):
        print(row)
```

```sql
SELECT customers.id, customers.full_name
FROM customers
WHERE NOT (EXISTS (SELECT orders.id FROM orders WHERE orders.customer_id = customers.id))
ORDER BY customers.id
```

```text
(5, 'Ewa Mazur')
```

Ten podzapytanie nazywamy **skorelowanym**: odwołuje się do `customers.c.id`, czyli do kolumny z zewnętrznego zapytania. To nie jest zwykłe „najpierw policz, potem porównaj” — baza wykonuje je konceptualnie raz dla każdego wiersza klienta, ale optymalizatory są w tym bardzo dobre i zwykle zamieniają to na efektywne złączenie.

> 💡 **Analogia** — `IN` z podzapytaniem to: „sprawdź listę wszystkich gości z sali B, a potem zobacz, czy jesteś na tej liście”. `EXISTS` to: „przejdź się po sali B i wróć, gdy tylko kogoś takiego zobaczysz”. Drugie może się skończyć po pierwszym kroku.

> 🆕 **SQLAlchemy 2.1** — w wersji 2.1 wersja `select(...).exists()` (metoda na obiekcie `Select`) jest formą preferowaną, a `Result` i `Row` mają dokładniejsze typowanie zgodne z PEP 646, dzięki czemu type checkery poprawnie wnioskują typy przy `select(tabela.c.a, tabela.c.b)`. W 2.0 oba zapisy działają, więc kod pozostaje przenośny.

### 5.4 `ANY` i `ALL`

`= ANY (podzapytanie)` to odpowiednik `IN` — z tą różnicą, że obsługuje też tablice (typ `ARRAY` w PostgreSQL). `= ALL (podzapytanie)` wymaga, by warunek był prawdziwy dla **wszystkich** wierszy.

```python
# examples/06_any_all.py
from __future__ import annotations

from sqlalchemy import create_engine, select

from shop_schema import products

engine = create_engine("sqlite:///shop.db")

# "Produkty, które są w którejkolwiek z kategorii X" — odpowiednik IN.
stmt = select(products.c.name).where(
    products.c.category_id.any_(select(products.c.category_id).where(products.c.price > 4000))
)

print(stmt.compile(engine, compile_kwargs={"literal_binds": True}))
```

```sql
SELECT products.name
FROM products
WHERE products.category_id = ANY (SELECT products.category_id FROM products WHERE products.price > 4000)
```

> 🧠 **Dlaczego tak jest** — `any_()` i `all_()` są metodami **kolumny**, nie samodzielnymi funkcjami. Wynika to z natury SQL-a: `ANY` zawsze odnosi się do jakiejś lewej strony porównania. W starszym API SQLAlchemy pisało się `any_(kolumna == wartość)`, ale ta forma jest przestarzała — dziś piszemy `kolumna.any_(wartość)`. Gdy argumentem jest obiekt `Select`, SQLAlchemy generuje `= ANY (podzapytanie)`; gdy zwykła wartość, generuje `= ANY (kolumna)` dla typów tablicowych.

### 5.5 Trzy drogi do tego samego celu

„Klienci bez zamówień” da się zapisać na trzy sposoby. Warto znać wszystkie trzy, bo każdy ma inną pułapkę i inne zastosowanie.

```python
# examples/06_three_ways.py
from __future__ import annotations

from sqlalchemy import create_engine, exists, select

from shop_schema import customers, orders

engine = create_engine("sqlite:///shop.db")

# A. LEFT JOIN + IS NULL — klasyka, bardzo czytelna, dobra gdy potrzebujesz
#    też kolumn z prawej tabeli (tu ich nie ma, więc IS NULL wystarcza).
stmt_a = (
    select(customers.c.id, customers.c.full_name)
    .select_from(customers.outerjoin(orders, customers.c.id == orders.c.customer_id))
    .where(orders.c.id.is_(None))
)

# B. NOT EXISTS — odporny na NULL, zwykle najszybszy przy dużej prawej tabeli.
stmt_b = select(customers.c.id, customers.c.full_name).where(
    ~exists(select(orders.c.id).where(orders.c.customer_id == customers.c.id))
)

# C. NOT IN — najkrótszy, ale niebezpieczny, jeśli podzapytanie może zwrócić NULL.
stmt_c = select(customers.c.id, customers.c.full_name).where(
    customers.c.id.not_in(select(orders.c.customer_id))
)

for name, stmt in (("A", stmt_a), ("B", stmt_b), ("C", stmt_c)):
    print(f"--- {name} ---")
    print(stmt.compile(engine, compile_kwargs={"literal_binds": True}))
    with engine.connect() as conn:
        print("  ", conn.execute(stmt).all())
```

```sql
--- A ---
SELECT customers.id, customers.full_name
FROM customers LEFT OUTER JOIN orders ON customers.id = orders.customer_id
WHERE orders.id IS NULL

--- B ---
SELECT customers.id, customers.full_name
FROM customers
WHERE NOT (EXISTS (SELECT orders.id FROM orders WHERE orders.customer_id = customers.id))

--- C ---
SELECT customers.id, customers.full_name
FROM customers
WHERE customers.id NOT IN (SELECT orders.customer_id FROM orders)
```

Wszystkie trzy zwracają `[(5, 'Ewa Mazur')]`.

| Podejście | Czytelność | Odporność na NULL | Kiedy wybrać |
|---|---|---|---|
| `LEFT JOIN` + `IS NULL` | wysoka | wysoka | gdy i tak łączysz tabele dla innych kolumn |
| `NOT EXISTS` | średnia | wysoka | duże tabele, kolumny opcjonalne, warunki złożone |
| `NOT IN` | wysoka | **niska** | krótkie listy literałów, nigdy podzapytania z NULL |

> ⚠️ **Pułapka** — wydajność `IN (podzapytanie)` i `NOT IN (podzapytanie)` na dużych zbiorach bywa dramatyczna. PostgreSQL na ogół radzi sobie, zamieniając to na **semi-join** (dla `IN`) lub **anti-join** (dla `NOT IN`), ale starsze wersje MySQL potrafiły wykonać podzapytanie raz na każdy wiersz zewnętrznego zapytania. Zasada praktyczna: gdy podzapytanie zwraca więcej niż kilka tysięcy wierszy, `EXISTS` jest bezpieczniejszym wyborem, a w ostateczności zwykłe złączenie z `GROUP BY` lub `LEFT JOIN ... IS NULL`. Zawsze potwierdź planem zapytania (sekcja 10), zamiast zgadywać.

---

## 6. CTE — wspólne wyrażenia tablicowe

### 6.1 Czym jest CTE

**CTE** (ang. *Common Table Expression*, wspólne wyrażenie tablicowe) to nazwany podzbiór wyników, który definiujesz **przed** zapytaniem głównym i możesz użyć w nim wielokrotnie. W SQL zapisuje się go słowem `WITH`.

> 💡 **Analogia** — CTE to notatka na marginesie. Zamiast wplatać skomplikowane obliczenie w środek zdania i powtarzać je trzy razy, robisz pomocniczą tabelkę u góry strony („przychód per miesiąc”) i dalej odwołujesz się do niej po nazwie. Zapytanie czyta się od góry do dołu, jak przepis kucharski: najpierw przygotuj składniki, potem gotuj.

W SQLAlchemy CTE tworzysz metodą `.cte()` na dowolnym `select()`.

```python
# examples/06_cte_simple.py
from __future__ import annotations

from sqlalchemy import create_engine, func, select

from shop_schema import order_items, orders

engine = create_engine("sqlite:///shop.db")

# Składnik: wartość każdego zamówienia.
order_totals = (
    select(
        orders.c.id.label("order_id"),
        orders.c.status.label("status"),
        func.sum(order_items.c.quantity * order_items.c.unit_price).label("total"),
    )
    .select_from(orders.join(order_items))
    .group_by(orders.c.id, orders.c.status)
    .cte("order_totals")
)

# Zapytanie główne korzysta z CTE jak ze zwykłej tabeli.
stmt = (
    select(
        order_totals.c.status,
        func.count().label("liczba_zamowien"),
        func.sum(order_totals.c.total).label("przychod"),
        func.round(func.avg(order_totals.c.total), 2).label("srednia_wartosc"),
    )
    .group_by(order_totals.c.status)
    .order_by(order_totals.c.status)
)

print(stmt.compile(engine, compile_kwargs={"literal_binds": True}))
```

```sql
WITH order_totals AS (
    SELECT orders.id AS order_id, orders.status AS status,
           sum(order_items.quantity * order_items.unit_price) AS total
    FROM orders JOIN order_items ON orders.id = order_items.order_id
    GROUP BY orders.id, orders.status
)
SELECT order_totals.status, count(*) AS liczba_zamowien,
       sum(order_totals.total) AS przychod,
       round(avg(order_totals.total), 2) AS srednia_wartosc
FROM order_totals
GROUP BY order_totals.status
ORDER BY order_totals.status
```

> 🧠 **Dlaczego tak jest** — CTE rozwiązuje problem **zagnieżdżania**. Bez niego musiałbyś albo napisać podzapytanie w `FROM` (trudne do czytania), albo powtórzyć tę samą agregację w kilku miejscach (baza policzyłaby ją kilka razy). CTE daje nazwę pośredniemu wynikowi — a nazwa to najtańsze narzędzie programisty do walki ze złożonością.

> ⚠️ **Pułapka** — nie zakładaj, że CTE jest materializowane raz i użyte ponownie. W większości baz to tylko **alias na podzapytanie** i jeśli odwołasz się do tego samego CTE dwa razy, baza policzy je dwa razy. PostgreSQL 12+ potrafi materializować CTE, gdy uzna to za opłacalne, ale nie jest to gwarancja. Gdy potrzebujesz naprawdę jednorazowego obliczenia, rozważ tabelę tymczasową albo zmaterializowany widok. W SQLAlchemy 2.1 doszły konstrukcje DDL do tworzenia widoków wprost z kodu (`CreateView`), co bywa wygodniejsze niż ręczny `op.execute()` w migracji.

### 6.2 CTE rekurencyjne — drzewo kategorii

To najciekawsza i najczęściej pomijana część CTE. **CTE rekurencyjne** pozwala przejść strukturę hierarchiczną (drzewo kategorii, struktura organizacyjna, graf znajomości) nie wiedząc z góry, jak głęboka jest.

Buduje się je z dwóch części połączonych `UNION ALL`:

1. **Część zakotwiczająca** (ang. *anchor*) — wiersze startowe. U nas: kategorie bez rodzica, czyli korzenie drzewa.
2. **Część rekurencyjna** — odwołuje się do samego CTE, żeby dojść o poziom głębiej.

```python
# examples/06_cte_recursive.py
from __future__ import annotations

from sqlalchemy import create_engine, literal, select

from shop_schema import categories

engine = create_engine("sqlite:///shop.db")

# CZĘŚĆ 1 (zakotwiczająca): korzenie drzewa, głębokość 0.
tree = (
    select(
        categories.c.id.label("id"),
        categories.c.name.label("name"),
        categories.c.parent_id.label("parent_id"),
        literal(0).label("depth"),
    )
    .where(categories.c.parent_id.is_(None))
    .cte("category_tree", recursive=True)
)

# CZĘŚĆ 2 (rekurencyjna): dzieci wszystkiego, co już jest w drzewie.
# Potrzebujemy dwóch aliasów — jednego na "dziecko" z tabeli,
# drugiego na poprzedni poziom z samego CTE.
child = categories.alias("child")
parent = tree.alias("parent")

tree = tree.union_all(
    select(
        child.c.id,
        child.c.name,
        child.c.parent_id,
        (parent.c.depth + 1).label("depth"),
    ).where(child.c.parent_id == parent.c.id)
)

stmt = (
    select(tree.c.id, tree.c.name, tree.c.depth)
    .order_by(tree.c.depth, tree.c.name)
)

print(stmt.compile(engine, compile_kwargs={"literal_binds": True}))

with engine.connect() as conn:
    for row in conn.execute(stmt):
        print(row)
```

```sql
WITH RECURSIVE category_tree(id, name, parent_id, depth) AS (
    SELECT categories.id AS id, categories.name AS name,
           categories.parent_id AS parent_id, 0 AS depth
    FROM categories
    WHERE categories.parent_id IS NULL
    UNION ALL
    SELECT child.id AS id, child.name AS name,
           child.parent_id AS parent_id, parent.depth + 1 AS depth
    FROM categories AS child, category_tree AS parent
    WHERE child.parent_id = parent.id
)
SELECT category_tree.id, category_tree.name, category_tree.depth
FROM category_tree
ORDER BY category_tree.depth, category_tree.name
```

```text
(1, 'Elektronika', 0)
(7, 'Dom i ogród', 0)
(2, 'Komputery', 1)
(5, 'Audio', 1)
(8, 'Kuchnia', 1)
(3, 'Laptopy', 2)
(4, 'Monitory', 2)
(6, 'Słuchawki', 2)
(9, 'Ekspresy', 2)
```

Trzy rzeczy, które trzeba zrozumieć, żeby to naprawdę działać:

**Aliasy są obowiązkowe.** W części rekurencyjnej tabela `categories` występuje drugi raz (raz jako korzeń, raz jako dziecko), a `category_tree` pojawia się w swojej własnej definicji. Bez `categories.alias("child")` i `tree.alias("parent")` SQLAlchemy nie wiedziałoby, o którą kolumnę chodzi. Wzorzec z aliasem na CTE pochodzi wprost z dokumentacji SQLAlchemy i jest tam konieczny — odwołanie się bezpośrednio do `tree.c.depth` wewnątrz definicji `tree` tworzy cykliczną zależność, której kompilator nie potrafi rozwiązać.

**`literal(0)` to nie ozdoba.** Musi być typu liczbowego, żeby `parent.c.depth + 1` w ogóle miało sens arytmetyczny. Gdybyś napisał `literal("0")`, dostałbyś konkatenację tekstu i głębokość `"0"`, `"01"`, `"011"`.

**`recursive=True` jest obowiązkowe.** Bez tego SQLAlchemy wygeneruje `WITH` zamiast `WITH RECURSIVE`. PostgreSQL i tak by to przyjął (domyślnie zakłada rekurencję, gdy CTE odwołuje się do siebie), ale SQLite wymaga jawnego `WITH RECURSIVE` i zgłosi błąd składni.

> ⚠️ **Pułapka** — rekurencyjne CTE **zapętli się w nieskończoność**, jeśli w danych jest cykl: kategoria A ma rodzica B, a B ma rodzica A. Baza nie ma pojęcia „już tu byłem”. Zabezpieczenia:
>
> - PostgreSQL: w części rekurencyjnej dodaj warunek `WHERE depth < 20` albo użyj `CYCLE ... SET ... USING PATH` (PostgreSQL 14+).
> - SQLite: zbieraj odwiedzone identyfikatory w kolumnie tekstowej i odrzucaj już widziane.
> - Najlepiej: zadbaj o to na poziomie modelu danych, żeby cykl nie mógł powstać. W aplikacji zapisywanej przez SQLAlchemy to zadanie walidacji, którą poznasz w module [13](13_zdarzenia_i_hybrydy.md).

> 🧪 **Ćwiczenie** — zmień `literal(0)` na `literal(1)` i zgadnij, co się zmieni w wyniku. Potem dodaj kolumnę `depth` do warunku końcowego (`WHERE depth <= 1`) i sprawdź, czy potrafisz pokazać tylko dwa najwyższe poziomy drzewa.

### 6.3 Pełna ścieżka kategorii

Bardzo praktyczne zastosowanie: zamiast samej nazwy kategorii chcesz zobaczyć całą drogę, np. `Elektronika > Komputery > Laptopy`. Wystarczy w CTE budować tekst ścieżki.

```python
# examples/06_cte_path.py
from __future__ import annotations

from sqlalchemy import create_engine, literal, select

from shop_schema import categories, products

engine = create_engine("sqlite:///shop.db")

paths = (
    select(
        categories.c.id.label("id"),
        categories.c.name.label("path"),
    )
    .where(categories.c.parent_id.is_(None))
    .cte("category_paths", recursive=True)
)

child = categories.alias("child")
parent = paths.alias("parent")

# Konkatenacja tekstu: w SQLite i PostgreSQL operator "||",
# który SQLAlchemy generuje dla operatora "+" na kolumnach tekstowych.
paths = paths.union_all(
    select(
        child.c.id,
        (parent.c.path + literal(" > ") + child.c.name).label("path"),
    ).where(child.c.parent_id == parent.c.id)
)

stmt = (
    select(products.c.name.label("produkt"), paths.c.path.label("kategoria"))
    .select_from(products.join(paths, products.c.category_id == paths.c.id))
    .order_by(paths.c.path, products.c.name)
)

print(stmt.compile(engine, compile_kwargs={"literal_binds": True}))

with engine.connect() as conn:
    for row in conn.execute(stmt):
        print(row)
```

```sql
WITH RECURSIVE category_paths(id, path) AS (
    SELECT categories.id AS id, categories.name AS path
    FROM categories
    WHERE categories.parent_id IS NULL
    UNION ALL
    SELECT child.id AS id, (category_paths.path || ' > ') || child.name AS path
    FROM categories AS child, category_paths
    WHERE child.parent_id = category_paths.id
)
SELECT products.name AS produkt, category_paths.path AS kategoria
FROM products JOIN category_paths ON products.category_id = category_paths.id
ORDER BY category_paths.path, products.name
```

```text
('Czajnik Philips', 'Dom i ogród > Kuchnia')
('Ekspres De\u2019Longhi Magnifica', 'Dom i ogród > Kuchnia > Ekspresy')
('Sony WH-1000XM5', 'Elektronika > Audio > Słuchawki')
('Jabra Evolve2 65', 'Elektronika > Audio > Słuchawki')
('Dell U2723QE', 'Elektronika > Komputery > Monitory')
('LG 27UP850', 'Elektronika > Komputery > Monitory')
('Lenovo ThinkPad E14', 'Elektronika > Komputery > Laptopy')
('Dell XPS 13', 'Elektronika > Komputery > Laptopy')
```

> 🧠 **Dlaczego tak jest** — `parent.c.path + literal(" > ") + child.c.name` działa, bo kolumna `path` w CTE dziedziczy typ `String` z `categories.c.name`. SQLAlchemy dla typu `String` tłumaczy operator `+` na operator konkatenacji **właściwy dla dialektu**: `||` w SQLite i PostgreSQL, `CONCAT()` w MySQL. To jeden z tych przypadków, w których abstrakcja dialektu naprawdę zarabia na siebie — ten sam kod Pythona daje poprawny SQL w trzech bazach.

> ⚠️ **Pułapka** — w części zakotwiczającej etykietę kolumny ustawiamy raz (`categories.c.name.label("path")`), a w rekurencyjnej robimy to ponownie na wyrażeniu. Jeśli zapomnisz `.label("path")` w którejkolwiek części, SQLAlchemy zgłosi `CompileError` o niedopasowaniu liczby kolumn — CTE wymaga, by obie części `UNION` miały tę samą liczbę i kolejność kolumn.

---

## 7. Operacje zbiorowe: `union`, `intersect`, `except_`

### 7.1 Czym są operacje na zbiorach

To algebra zbiorów przeniesiona na wiersze tabeli. Wyobraź sobie dwie listy nazwisk:

- **`UNION`** — suma: wszyscy z listy A i wszyscy z listy B, **bez duplikatów**.
- **`UNION ALL`** — suma, **z duplikatami**. Szybsza, bo baza nie musi sprawdzać powtórzeń.
- **`INTERSECT`** — część wspólna: osoby obecne na obu listach.
- **`EXCEPT`** — różnica: osoby z listy A, których nie ma na liście B.

```python
# examples/06_set_ops.py
from __future__ import annotations

from sqlalchemy import create_engine, select

from shop_schema import customers, orders

engine = create_engine("sqlite:///shop.db")

# Miasta, z których pochodzą klienci...
cities_from_customers = select(customers.c.city.label("city"))

# ...i miasta, do których wysyłamy (na razie te same, bo brak tabeli adresów,
# więc dla demonstracji użyjemy listy literałów przez VALUES).
shipping_cities = select(orders.c.id.label("city")).where(orders.c.id < 0)  # pusta lista

# UNION ALL — suma z duplikatami (tu: lista miast klientów powtórzona dwa razy).
stmt_union_all = cities_from_customers.union_all(cities_from_customers)
# UNION — suma bez duplikatów.
stmt_union = cities_from_customers.union(cities_from_customers)

print("--- UNION ALL ---")
print(stmt_union_all.compile(engine, compile_kwargs={"literal_binds": True}))
print("--- UNION ---")
print(stmt_union.compile(engine, compile_kwargs={"literal_binds": True}))

with engine.connect() as conn:
    print("UNION ALL:", conn.execute(stmt_union_all).scalars().all())
    print("UNION:    ", conn.execute(stmt_union).scalars().all())
```

```sql
--- UNION ALL ---
SELECT customers.city AS city FROM customers
UNION ALL
SELECT customers.city AS city FROM customers

--- UNION ---
SELECT customers.city AS city FROM customers
UNION
SELECT customers.city AS city FROM customers
```

```text
UNION ALL: ['Kraków', 'Warszawa', 'Gdańsk', 'Kraków', 'Poznań',
            'Kraków', 'Warszawa', 'Gdańsk', 'Kraków', 'Poznań']
UNION:     ['Kraków', 'Warszawa', 'Gdańsk', 'Poznań']
```

### 7.2 `INTERSECT` i `EXCEPT`

```python
# examples/06_intersect_except.py
from __future__ import annotations

from sqlalchemy import create_engine, select

from shop_schema import customers, orders

engine = create_engine("sqlite:///shop.db")

# Klienci z Krakowa.
krakow = select(customers.c.full_name).where(customers.c.city == "Kraków")

# Klienci, którzy złożyli zamówienie o statusie 'paid'.
paying = select(customers.c.full_name).where(
    customers.c.id.in_(select(orders.c.customer_id).where(orders.c.status == "paid"))
)

# Część wspólna: krakowianie, którzy płacili.
stmt_intersect = krakow.intersect(paying)

# Różnica: krakowianie, którzy NIE płacili.
stmt_except = krakow.except_(paying)

for name, stmt in (("INTERSECT", stmt_intersect), ("EXCEPT", stmt_except)):
    print(f"--- {name} ---")
    print(stmt.compile(engine, compile_kwargs={"literal_binds": True}))
    with engine.connect() as conn:
        print("  ", conn.execute(stmt).scalars().all())
```

```sql
--- INTERSECT ---
SELECT customers.full_name FROM customers WHERE customers.city = 'Kraków'
INTERSECT
SELECT customers.full_name FROM customers
WHERE customers.id IN (SELECT orders.customer_id FROM orders WHERE orders.status = 'paid')

--- EXCEPT ---
SELECT customers.full_name FROM customers WHERE customers.city = 'Kraków'
EXCEPT
SELECT customers.full_name FROM customers
WHERE customers.id IN (SELECT orders.customer_id FROM orders WHERE orders.status = 'paid')
```

```text
INTERSECT: ['Anna Kowalska']
EXCEPT:    ['Tomasz Zieliński']
```

> ⚠️ **Pułapka** — w Pythonie `except` jest słowem kluczowym, więc w SQLAlchemy metoda nazywa się **`except_()`** z podkreśleniem na końcu. To samo dotyczy `Select.except_all()`. Zapominając o podkreśleniu, dostaniesz `SyntaxError` w kodzie Pythona, a nie w SQL-u — co samo w sobie jest dobrą wiadomością, bo błąd wychwycisz od razu.

### 7.3 Kiedy operacje zbiorowe, a kiedy złączenie

| Sytuacja | Lepsze narzędzie |
|---|---|
| Potrzebujesz kolumn z obu tabel obok siebie | `JOIN` |
| Chcesz policzyć coś w obrębie grupy z jednej tabeli | `GROUP BY` |
| Łączysz wyniki dwóch **niezależnych** zapytań o tym samym kształcie | `UNION` / `UNION ALL` |
| Szukasz „co jest w A, czego nie ma w B” dla tego samego typu obiektu | `EXCEPT` / `NOT EXISTS` |
| Porównujesz dwa źródła danych o tym samym schemacie | `EXCEPT` w obie strony lub `FULL JOIN` |

> 🧠 **Dlaczego tak jest** — `UNION` wymaga, by oba zapytania miały **tę samą liczbę kolumn w tej samej kolejności i zgodnych typach**. To nie jest ograniczenie techniczne, a definicja sumy zbiorów: nie da się dodać do siebie jabłek i adresów e-mail. Jeśli SQLAlchemy zgłosi `CompileError: SELECT construct has a different number of columns`, sprawdź, czy obie strony mają identyczną strukturę.

---

## 8. Funkcje okna

### 8.1 Problem: agregacja gubi wiersze

`GROUP BY` ma pewną brutalną właściwość: **skleja wiersze w jeden**. Jeśli pogrupujesz produkty po kategorii i policzysz średnią cenę, dostaniesz 6 wierszy — po jednym na kategorię. Informacja o poszczególnych produktach zniknęła.

A co, jeśli chcesz **jednocześnie** widzieć każdy produkt **i** średnią jego kategorii w tej samej linii? Albo numer porządkowy w obrębie kategorii? Albo sumę narastającą po miesiącach?

> 💡 **Analogia** — wyobraź sobie tabelę wyników egzaminu. `GROUP BY klasa` daje Ci jedno zdanie na klasę: „3A — średnia 4,2”. Funkcja okna daje Ci **każdego ucznia w osobnym wierszu**, ale z dopisaną obok średnią jego klasy: „Jan Kowalski, 3A, 4,5, średnia klasy 4,2”. Nie tracisz szczegółu, a dostajesz kontekst.

Formalnie: funkcja okna liczy coś **w obrębie grupy** (okna, ang. *window*), ale **nie zwija** wierszy. Okno definiujesz przez `PARTITION BY` (podział na grupy) i `ORDER BY` (kolejność w grupie).

### 8.2 `row_number()`, `rank()`, `dense_rank()`

Trzy funkcje numerujące, które wyglądają podobnie i zachowują się zupełnie inaczej przy remisach.

```python
# examples/06_row_number.py
from __future__ import annotations

from sqlalchemy import create_engine, func, select

from shop_schema import products

engine = create_engine("sqlite:///shop.db")

price_rank = func.rank().over(
    partition_by=products.c.category_id,
    order_by=products.c.price.desc(),
).label("rank")

dense = func.dense_rank().over(
    partition_by=products.c.category_id,
    order_by=products.c.price.desc(),
).label("dense_rank")

row_no = func.row_number().over(
    partition_by=products.c.category_id,
    order_by=products.c.price.desc(),
).label("row_number")

stmt = (
    select(
        products.c.category_id,
        products.c.name,
        products.c.price,
        row_no,
        price_rank,
        dense,
    )
    .order_by(products.c.category_id, products.c.price.desc())
)

print(stmt.compile(engine, compile_kwargs={"literal_binds": True}))

with engine.connect() as conn:
    for row in conn.execute(stmt):
        print(row)
```

```sql
SELECT products.category_id, products.name, products.price,
       row_number() OVER (PARTITION BY products.category_id ORDER BY products.price DESC) AS row_number,
       rank() OVER (PARTITION BY products.category_id ORDER BY products.price DESC) AS rank,
       dense_rank() OVER (PARTITION BY products.category_id ORDER BY products.price DESC) AS dense_rank
FROM products
ORDER BY products.category_id, products.price DESC
```

```text
(3, 'Dell XPS 13',        Decimal('6499.00'), 1, 1, 1)
(3, 'Lenovo ThinkPad E14', Decimal('4299.00'), 2, 2, 2)
(4, 'Dell U2723QE',       Decimal('2899.00'), 1, 1, 1)
(4, 'LG 27UP850',         Decimal('2399.00'), 2, 2, 2)
(6, 'Sony WH-1000XM5',    Decimal('1499.00'), 1, 1, 1)
(6, 'Jabra Evolve2 65',   Decimal('899.00'),  2, 2, 2)
(8, 'Czajnik Philips',    Decimal('199.00'),  1, 1, 1)
(9, 'Ekspres De\u2019Longhi Magnifica', Decimal('2199.00'), 1, 1, 1)
```

W naszych danych nie ma remisów, więc wszystkie trzy funkcje dają to samo. Różnica pojawia się przy równych wartościach:

| Cena | `row_number()` | `rank()` | `dense_rank()` |
|---|---|---|---|
| 100 | 1 | 1 | 1 |
| 100 | 2 | 1 | 1 |
| 90 | 3 | 3 | 2 |
| 80 | 4 | 4 | 3 |

> 🧠 **Dlaczego tak jest** — `row_number()` to po prostu licznik wierszy: zawsze 1, 2, 3, 4 — nigdy się nie powtarza i nigdy nie ma luk. `rank()` przy remisie daje tę samą pozycję obu wierszom, a potem **przeskakuje** (dwie jedynki, potem trójka — jak w zawodach sportowych). `dense_rank()` przy remisie też daje tę samą pozycję, ale **nie przeskakuje** (dwie jedynki, potem dwójka). Wybór zależy od pytania biznesowego: „kto zajął 3. miejsce?” (rank) czy „ile mamy poziomów cenowych?” (dense_rank).

### 8.3 Top N w grupie — najczęstsze zastosowanie

Pytanie: *„pokaż 2 najdroższe produkty w każdej kategorii”*. Nie da się tego zrobić zwykłym `LIMIT`, bo `LIMIT 2` obcina cały wynik, a nie każdą grupę. Rozwiązanie: ponumeruj wiersze w obrębie grupy, a potem odfiltruj numery.

Filtr **musi** działać na wyniku CTE, nie na tym samym poziomie, na którym liczysz `row_number()` — bo `WHERE` wykonuje się przed funkcjami okna. To dokładnie ta sama zasada co przy `HAVING` vs `WHERE`.

```python
# examples/06_top_n.py
from __future__ import annotations

from sqlalchemy import create_engine, func, select

from shop_schema import categories, products

engine = create_engine("sqlite:///shop.db")

ranked = (
    select(
        categories.c.name.label("kategoria"),
        products.c.name.label("produkt"),
        products.c.price.label("cena"),
        func.row_number()
        .over(partition_by=products.c.category_id, order_by=products.c.price.desc())
        .label("pozycja"),
    )
    .select_from(products.join(categories, products.c.category_id == categories.c.id))
    .cte("ranked_products")
)

stmt = (
    select(ranked.c.kategoria, ranked.c.produkt, ranked.c.cena)
    .where(ranked.c.pozycja <= 2)
    .order_by(ranked.c.kategoria, ranked.c.pozycja)
)

print(stmt.compile(engine, compile_kwargs={"literal_binds": True}))

with engine.connect() as conn:
    for row in conn.execute(stmt):
        print(row)
```

```sql
WITH ranked_products AS (
    SELECT categories.name AS kategoria, products.name AS produkt,
           products.price AS cena,
           row_number() OVER (PARTITION BY products.category_id ORDER BY products.price DESC) AS pozycja
    FROM products JOIN categories ON products.category_id = categories.id
)
SELECT ranked_products.kategoria, ranked_products.produkt, ranked_products.cena
FROM ranked_products
WHERE ranked_products.pozycja <= 2
ORDER BY ranked_products.kategoria, ranked_products.pozycja
```

```text
('Ekspresy', 'Ekspres De\u2019Longhi Magnifica', Decimal('2199.00'))
('Kuchnia', 'Czajnik Philips', Decimal('199.00'))
('Laptopy', 'Dell XPS 13', Decimal('6499.00'))
('Laptopy', 'Lenovo ThinkPad E14', Decimal('4299.00'))
('Monitory', 'Dell U2723QE', Decimal('2899.00'))
('Monitory', 'LG 27UP850', Decimal('2399.00'))
('Słuchawki', 'Sony WH-1000XM5', Decimal('1499.00'))
('Słuchawki', 'Jabra Evolve2 65', Decimal('899.00'))
```

> ⚠️ **Pułapka** — próba zapisania tego w jednym `select()` z `WHERE pozycja <= 2` nie zadziała: SQLAlchemy wygeneruje SQL, w którym alias funkcji okna pojawi się w `WHERE`, a SQL **nie pozwala** odwoływać się do aliasów z `SELECT` w `WHERE`. Baza zgłosi `no such column: pozycja`. Kolejność wykonania w SQL to `FROM` → `WHERE` → `GROUP BY` → `HAVING` → `SELECT` → `ORDER BY`. Funkcje okna liczone są dopiero na etapie `SELECT`, czyli **po** `WHERE`. Dlatego zawsze potrzebujesz CTE albo podzapytania.

### 8.4 `lag()` i `lead()` — porównywanie z sąsiednim wierszem

`lag()` sięga po wartość z poprzedniego wiersza, `lead()` po następny — w obrębie okna wyznaczonego przez `ORDER BY`. To podstawa każdej analizy zmian: „wzrost względem poprzedniego miesiąca”, „różnica między kolejnymi odczytami licznika”.

```python
# examples/06_lag_lead.py
from __future__ import annotations

from sqlalchemy import create_engine, func, select

from shop_schema import order_items, orders

engine = create_engine("sqlite:///shop.db")

# Miesięczny przychód. strftime() jest specyficzne dla SQLite —
# w PostgreSQL użyłbyś func.to_char(orders.c.placed_at, "YYYY-MM").
month = func.strftime("%Y-%m", orders.c.placed_at).label("month")

monthly = (
    select(
        month,
        func.sum(order_items.c.quantity * order_items.c.unit_price).label("revenue"),
    )
    .select_from(orders.join(order_items))
    .where(orders.c.status == "paid")
    .group_by(month)
    .cte("monthly_revenue")
)

prev_revenue = func.lag(monthly.c.revenue).over(order_by=monthly.c.month).label("prev_revenue")

stmt = (
    select(
        monthly.c.month,
        monthly.c.revenue,
        prev_revenue,
        (monthly.c.revenue - prev_revenue).label("change"),
    )
    .order_by(monthly.c.month)
)

print(stmt.compile(engine, compile_kwargs={"literal_binds": True}))

with engine.connect() as conn:
    for row in conn.execute(stmt):
        print(row)
```

```sql
WITH monthly_revenue AS (
    SELECT strftime('%Y-%m', orders.placed_at) AS month,
           sum(order_items.quantity * order_items.unit_price) AS revenue
    FROM orders JOIN order_items ON orders.id = order_items.order_id
    WHERE orders.status = 'paid'
    GROUP BY strftime('%Y-%m', orders.placed_at)
)
SELECT monthly_revenue.month, monthly_revenue.revenue,
       lag(monthly_revenue.revenue) OVER (ORDER BY monthly_revenue.month) AS prev_revenue,
       monthly_revenue.revenue - lag(monthly_revenue.revenue) OVER (ORDER BY monthly_revenue.month) AS change
FROM monthly_revenue
ORDER BY monthly_revenue.month
```

```text
('2026-01', Decimal('5798.00'),  None,          None)
('2026-02', Decimal('9196.00'),  Decimal('5798.00'), Decimal('3398.00'))
('2026-03', Decimal('2199.00'),  Decimal('9196.00'), Decimal('-6997.00'))
('2026-04', Decimal('6097.00'),  Decimal('2199.00'), Decimal('3898.00'))
('2026-05', Decimal('2998.00'),  Decimal('6097.00'), Decimal('-3099.00'))
('2026-06', Decimal('8898.00'),  Decimal('6097.00'), Decimal('2801.00'))
```

Pierwszy miesiąc ma `None` — nie ma poprzednika. To zachowanie jest celowe i w większości przypadków pożądane (nie chcemy zera, bo zero znaczyłoby „poprzedni miesiąc miał zerowy przychód”, a to nieprawda).

> ⚠️ **Pułapka** — `lag()` i `lead()` **bez `ORDER BY` w oknie** zwracają wartości z wiersza, który baza akurat miała pod ręką. SQL nie definiuje kolejności wierszy bez `ORDER BY`, więc wynik jest niedeterministyczny: ten sam kod na dwóch uruchomieniach może dać inne liczby. To jeden z najczęstszych błędów w raportach — poprawnie działający kod daje błędne dane, i nic tego nie sygnalizuje. **Każda funkcja okna z `lag`, `lead`, `sum().over()` i numerująca musi mieć `order_by` w oknie.**

### 8.5 Suma narastająca — `sum().over()`

Suma narastająca (ang. *running total*) to klasyk raportowania. `sum()` jako funkcja okna dodaje wartość bieżącego wiersza do wszystkich poprzednich.

```python
# examples/06_running_total.py
from __future__ import annotations

from sqlalchemy import create_engine, func, select

from shop_schema import order_items, orders

engine = create_engine("sqlite:///shop.db")

month = func.strftime("%Y-%m", orders.c.placed_at).label("month")

monthly = (
    select(
        month,
        func.sum(order_items.c.quantity * order_items.c.unit_price).label("revenue"),
    )
    .select_from(orders.join(order_items))
    .where(orders.c.status == "paid")
    .group_by(month)
    .cte("monthly_revenue")
)

# Ramka okna: od początku danych (UNBOUNDED PRECEDING) do bieżącego wiersza.
running = (
    func.sum(monthly.c.revenue)
    .over(order_by=monthly.c.month, rows=(None, 0))
    .label("running_total")
)

stmt = select(monthly.c.month, monthly.c.revenue, running).order_by(monthly.c.month)

print(stmt.compile(engine, compile_kwargs={"literal_binds": True}))

with engine.connect() as conn:
    for row in conn.execute(stmt):
        print(row)
```

```sql
WITH monthly_revenue AS (
    SELECT strftime('%Y-%m', orders.placed_at) AS month,
           sum(order_items.quantity * order_items.unit_price) AS revenue
    FROM orders JOIN order_items ON orders.id = order_items.order_id
    WHERE orders.status = 'paid'
    GROUP BY strftime('%Y-%m', orders.placed_at)
)
SELECT monthly_revenue.month, monthly_revenue.revenue,
       sum(monthly_revenue.revenue) OVER (
           ORDER BY monthly_revenue.month
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS running_total
FROM monthly_revenue
ORDER BY monthly_revenue.month
```

```text
('2026-01', Decimal('5798.00'),  Decimal('5798.00'))
('2026-02', Decimal('9196.00'),  Decimal('14994.00'))
('2026-03', Decimal('2199.00'),  Decimal('17193.00'))
('2026-04', Decimal('6097.00'),  Decimal('23290.00'))
('2026-05', Decimal('2998.00'),  Decimal('26288.00'))
('2026-06', Decimal('8898.00'),  Decimal('35186.00'))
```

### 8.6 Ramka okna: `ROWS` vs `RANGE`

`rows=(None, 0)` to parametr, który w SQL staje się `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. Możesz podać dowolny zakres:

| SQLAlchemy | SQL | Znaczenie |
|---|---|---|
| `rows=(None, 0)` | `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` | od początku do bieżącego wiersza |
| `rows=(-1, 1)` | `ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING` | bieżący ± jeden wiersz (średnia krocząca 3 punktów) |
| `rows=(None, None)` | `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` | całe okno |
| `rows=(0, None)` | `ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING` | od bieżącego do końca |

Różnica między `ROWS` i `RANGE` (parametr `range_=(...)`) jest subtelna, ale ważna: `ROWS` liczy **fizyczne wiersze**, a `RANGE` — **zakres wartości** z `ORDER BY`. Jeśli w danych są remisy (dwa wiersze z tą samą datą), `RANGE` potraktuje je razem, a `ROWS` po kolei.

> ⚠️ **Pułapka** — domyślna ramka w SQL, gdy podasz `ORDER BY` bez `ROWS`/`RANGE`, to `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. Brzmi podobnie do tego, czego chcemy przy sumie narastającej — i **dla unikalnych wartości działa identycznie**. Ale gdy dwa wiersze mają tę samą wartość w `ORDER BY` (np. dwa zamówienia z tą samą datą), `RANGE` doda je oba naraz, a nie po kolei. Jeśli budujesz raport dzienny po dacie i w jednym dniu jest kilka zdarzeń, dostaniesz inne liczby, niż się spodziewasz. W takich przypadkach **jawnie podawaj `rows=(None, 0)`** — wtedy wiesz dokładnie, co się dzieje.

> 🧪 **Ćwiczenie** — dodaj do raportu średnią kroczącą z trzech miesięcy (`rows=(-1, 1)`). Zauważ, że w pierwszym i ostatnim miesiącu średnia liczy się tylko z dwóch wartości. Jak to naprawić, gdy chcesz, by brzegi były liczone z tego, co jest? (Podpowiedź: to nie jest zadanie dla ramki, tylko dla `COALESCE` albo obsługi w Pythonie.)

### 8.7 Nazwane okna

Jeśli to samo okno powtarza się kilka razy, można nadać mu nazwę i użyć wielokrotnie. W SQLAlchemy służy do tego metoda `.over()` wywołana na łańcuchu — ale najprostszy sposób to zapisanie okna w zmiennej i użycie go w kilku miejscach, co SQLAlchemy skompiluje jako powtórzone wyrażenie.

> 🧠 **Dlaczego tak jest** — SQLAlchemy nie ma osobnego API do klauzuli `WINDOW nazwa AS (...)`, ale powtórzone wyrażenia `OVER (...)` są funkcjonalnie równoważne. Zysk z nazwanego okna jest głównie kosmetyczny (krótszy SQL) i nie wpływa na wydajność — optymalizator i tak rozpozna identyczne okna. Nie komplikuj kodu, jeśli nie musisz.

---

## 9. `LATERAL JOIN`

### 9.1 Problem: „weź N wierszy dla każdej grupy”

Wróćmy do pytania „2 najdroższe produkty w każdej kategorii”. Rozwiązaliśmy je funkcją okna (sekcja 8.3). Ale co, gdyby warunek był bardziej skomplikowany — na przykład „3 najnowsze zamówienia każdego klienta” albo „najbliższe 5 stacji benzynowych dla każdego punktu na trasie”? Numerowanie wierszy przestaje wystarczać, gdy potrzebujesz **osobnego zapytania z `LIMIT` dla każdej grupy**.

> 💡 **Analogia** — zwykłe podzapytanie w `FROM` to pytanie zadane raz, z góry, bez wiedzy o wierszu, do którego się odnosi. `LATERAL` to pytanie zadane **dla każdego wiersza osobno**, z możliwością odwołania się do niego. Jak kelner, który nie przynosi raz na zawsze całego menu, ale pyta każdego gościa indywidualnie, co chce.

### 9.2 `lateral()` w SQLAlchemy

```python
# examples/06_lateral.py
from __future__ import annotations

from sqlalchemy import create_engine, select, true
from sqlalchemy.dialects import postgresql

from shop_schema import categories, products

# LATERAL istnieje w PostgreSQL i Oracle. SQLite go NIE obsługuje,
# więc ten przykład kompilujemy dla PostgreSQL (nie uruchamiamy na SQLite).
pg_engine = create_engine("postgresql+psycopg://user:pass@localhost/shop")

# Podzapytanie skorelowane: odwołuje się do categories.c.id z zewnątrz.
# To właśnie dzięki LATERAL jest to w ogóle dozwolone.
top_products = (
    select(products.c.name, products.c.price)
    .where(products.c.category_id == categories.c.id)
    .order_by(products.c.price.desc())
    .limit(3)
    .lateral("top_products")
)

stmt = (
    select(
        categories.c.name.label("kategoria"),
        top_products.c.name.label("produkt"),
        top_products.c.price.label("cena"),
    )
    .select_from(categories.join(top_products, true()))
    .order_by(categories.c.name, top_products.c.price.desc())
)

print(stmt.compile(dialect=postgresql.dialect(), compile_kwargs={"literal_binds": True}))
```

```sql
SELECT categories.name AS kategoria, top_products.name AS produkt, top_products.price AS cena
FROM categories JOIN LATERAL (
    SELECT products.name AS name, products.price AS price
    FROM products
    WHERE products.category_id = categories.id
    ORDER BY products.price DESC
    LIMIT 3
) AS top_products ON true
ORDER BY categories.name, top_products.price DESC
```

Trzy elementy układanki:

1. `.lateral("top_products")` — zamienia zwykłe podzapytanie w **podzapytanie boczne**. Bez tego SQLAlchemy zgłosi `InvalidRequestError`, bo odwołanie do `categories.c.id` wewnątrz `FROM` jest niedozwolone.
2. `join(top_products, true())` — warunek `ON true` jest konieczny składniowo. Cały warunek korelacji siedzi już w środku podzapytania, więc na zewnątrz nie ma czego dopasowywać. `true()` z `sqlalchemy` generuje `ON true`.
3. `.limit(3)` w podzapytaniu działa **per kategoria**, a nie na cały wynik — i to jest cała siła `LATERAL`.

> ⚠️ **Pułapka** — SQLite nie obsługuje `LATERAL` i zgłosi `OperationalError: near "LATERAL": syntax error`. Jeśli rozwijasz lokalnie na SQLite, a wdrażasz na PostgreSQL, ten kod przejdzie testy i wybuchnie na produkcji. Dwa wyjścia: (a) trzymaj się funkcji okna z CTE, które działają wszędzie, (b) testuj na PostgreSQL — najlepiej w kontenerze, o czym piszę w module [18](18_testowanie.md). `LATERAL` używaj świadomie, gdy naprawdę daje lepszy plan zapytania niż numerowanie wierszy — a to trzeba zmierzyć, nie założyć.

> 🧪 **Ćwiczenie** — zastanów się, jak zapisać „3 najdroższe produkty w każdej kategorii” bez `LATERAL`, używając CTE rekurencyjnego. Czy to w ogóle ma sens? (Odpowiedź: ma, ale jest znacznie bardziej zawiłe niż `row_number()` — co samo w sobie jest pouczające.)

---

## 10. Praktyka inżynierska: plan zapytania, N+1, czytelność

### 10.1 `EXPLAIN` — jak baza mówi Ci, co naprawdę robi

SQL, który napisałeś, i SQL, który baza wykonuje, to dwie różne rzeczy. Baza ma **optymalizator zapytań**, który przekształca Twoje zapytanie w **plan wykonania**: decyduje, którą tabelę czytać pierwszą, których indeksów użyć, w jakiej kolejności łączyć. Podglądnięcie tego planu to najszybsza droga do zrozumienia, dlaczego zapytanie działa szybko albo wolno.

- **SQLite**: `EXPLAIN QUERY PLAN <zapytanie>`
- **PostgreSQL**: `EXPLAIN ANALYZE <zapytanie>` (dodaje rzeczywiste czasy wykonania)

```python
# examples/06_explain.py
from __future__ import annotations

from sqlalchemy import create_engine, select, text

from shop_schema import customers, orders

engine = create_engine("sqlite:///shop.db")

stmt = (
    select(customers.c.full_name, orders.c.id)
    .select_from(customers.join(orders, customers.c.id == orders.c.customer_id))
    .where(orders.c.status == "paid")
)

with engine.connect() as conn:
    # UWAGA: f-string z SQL-em jest tu świadomym wyjątkiem. Nie wstawiamy
    # danych użytkownika — wstawiamy SQL wygenerowany przez SQLAlchemy
    # z już wbudowanymi literałami. W kodzie produkcyjnym tak się NIE robi.
    compiled = stmt.compile(engine, compile_kwargs={"literal_binds": True})
    plan = conn.execute(text(f"EXPLAIN QUERY PLAN {compiled}"))
    for row in plan:
        print(row)
```

```text
(2, 0, 0, 'SEARCH orders USING INDEX ix_orders_customer_id (customer_id=?)')
(2, 1, 0, 'SEARCH customers USING INTEGER PRIMARY KEY (rowid=?)')
(3, 0, 0, 'USE TEMP B-TREE FOR ORDER BY')
```

Czytanie planu w praktyce sprowadza się do trzech sygnałów ostrzegawczych:

| W planie widzisz | Co to znaczy | Co zrobić |
|---|---|---|
| `SCAN tabela` (zamiast `SEARCH`) | baza czyta całą tabelę wiersz po wierszu | dodać indeks na kolumnie z `WHERE`/`JOIN` |
| `USE TEMP B-TREE FOR ORDER BY` | sortowanie w pamięci tymczasowej | rozważyć indeks zgodny z `ORDER BY` |
| `CO-ROUTINE` / `MATERIALIZE` przy CTE | CTE zmaterializowane | sprawdzić, czy nie liczy się niepotrzebnie |

> 🧠 **Dlaczego tak jest** — `SEARCH ... USING INDEX` w SQLite (a `Index Scan` w PostgreSQL) oznacza, że baza nie musiała czytać wszystkich wierszy, tylko od razu przeskoczyła do tych pasujących. `SCAN` (a `Seq Scan`) oznacza czytanie od deski do deski. Różnica na tabeli z 10 wierszami jest niezauważalna, a na tabeli z 10 milionami — to różnica między 5 milisekundami a 5 minutami. **Indeks na kolumnie klucza obcego to nie luksus, to obowiązek** — dlatego w naszym schemacie `ix_orders_customer_id` istnieje (SQLAlchemy tworzy indeksy na kluczach obcych automatycznie przy `ForeignKey`? **Nie** — tworzy je tylko przy `index=True` lub gdy wymusza to konwencja nazewnictwa. W naszym przypadku indeks powstał, bo SQLite automatycznie indeksuje kolumny biorące udział w ograniczeniach. W PostgreSQL musisz o to zadbać świadomie).

> ⚠️ **Pułapka** — mierzenie wydajności na bazie z 10 wierszami jest bezsensowne. Optymalizator dla małych tabel wybierze `SCAN`, bo tak jest taniej — i wyciągniesz wniosek, że indeks nie działa. Plany zmieniają się wraz z rozmiarem danych i rozkładem wartości. Zanim wyciągniesz wnioski, wygeneruj realistyczny zbiór danych (moduł [17](17_wydajnosc.md) pokazuje, jak to zrobić dla miliona wierszy).

### 10.2 JOIN vs N+1 — dlaczego to jest ten sam problem

Zapytanie `JOIN` pobiera wszystkie potrzebne dane w **jednej rundzie do bazy**. Alternatywa — pobranie listy, a potem osobne zapytanie dla każdego elementu — nazywa się **N+1** i jest najczęstszą przyczyną wolnych aplikacji.

Wyobraź sobie listę 100 zamówień. Chcesz do każdego dopisać nazwisko klienta:

- **JOIN**: 1 zapytanie zwracające 100 wierszy. Złożoność $O(1)$.
- **N+1**: 1 zapytanie o zamówienia + 100 zapytań o klientów = 101 rund. Złożoność $O(1 + N)$.

Każda runda to narzut sieciowy — nawet 1 ms opóźnienia pomnożony przez 100 daje 100 ms, a przy 1000 zamówień już sekundę. Baza wykonuje 101 razy pracę, którą mogłaby wykonać raz.

> 🧠 **Dlaczego tak jest** — narzut na **jedno** zapytanie (przygotowanie, wysłanie, parsowanie, planowanie, odebranie) jest wielokrotnie większy niż narzut na **jeden dodatkowy wiersz** w już wysłanym zapytaniu. To dlatego 1 zapytanie o 1000 wierszy jest zwykle szybsze niż 1000 zapytań o 1 wiersz, nawet jeśli łącznie przesyłają tyle samo danych.

W modułach [10](10_zapytania_orm.md) i [11](11_ladowanie_i_n_plus_1.md) zobaczysz, jak ORM potrafi **niepostrzeżenie** wygenerować N+1, gdy odwołasz się do relacji w pętli. Tam pokażemy, jak to wykryć licznikiem zapytań i naprawić strategią ładowania. Tutaj zapamiętaj samą zasadę: **im mniej rund do bazy, tym lepiej**, i JOIN jest podstawowym narzędziem do osiągnięcia tego celu.

> 🆕 **SQLAlchemy 2.1** — w wersji 2.1 strategia `selectinload` dostała parametr `chunksize`, który dzieli `IN (...)` na porcje, oraz opcję `omit_join` pozwalającą pominąć dodatkowe złączenie przy relacjach wiele-do-wielu. Oba parametry dotyczą ładowania relacji w ORM (moduł [11](11_ladowanie_i_n_plus_1.md)), ale warto wiedzieć, że problem „jak nie robić N+1” jest w SQLAlchemy traktowany bardzo poważnie.

### 10.3 CTE jako narzędzie czytelności

Długie zapytanie z trzema zagnieżdżonymi podzapytaniami jest technicznie poprawne i praktycznie niemożliwe do utrzymania. CTE pozwala rozbić je na nazwane kroki.

Porównaj — jedno zapytanie, które robi wszystko naraz:

```python
# examples/06_readability_before.py
from sqlalchemy import func, select

from shop_schema import customers, order_items, orders

# Trudne do przeczytania: agregacja wewnątrz warunku wewnątrz agregacji.
stmt = (
    select(customers.c.full_name, func.count().label("liczba_zamowien"))
    .select_from(customers.join(orders).join(order_items))
    .where(
        orders.c.id.in_(
            select(orders.c.id)
            .select_from(orders.join(order_items))
            .group_by(orders.c.id)
            .having(func.sum(order_items.c.quantity * order_items.c.unit_price) > 3000)
        )
    )
    .group_by(customers.c.full_name)
)
```

Z tym samym, rozbitym na nazwane kroki:

```python
# examples/06_readability_after.py
from __future__ import annotations

from sqlalchemy import create_engine, func, select

from shop_schema import customers, order_items, orders

engine = create_engine("sqlite:///shop.db")

# Krok 1: wartość każdego zamówienia.
order_values = (
    select(
        orders.c.id.label("order_id"),
        orders.c.customer_id.label("customer_id"),
        func.sum(order_items.c.quantity * order_items.c.unit_price).label("total"),
    )
    .select_from(orders.join(order_items))
    .group_by(orders.c.id, orders.c.customer_id)
    .cte("order_values")
)

# Krok 2: tylko duże zamówienia.
big_orders = (
    select(order_values.c.customer_id)
    .where(order_values.c.total > 3000)
    .cte("big_orders")
)

# Krok 3: klienci i liczba ich dużych zamówień.
stmt = (
    select(customers.c.full_name, func.count().label("duze_zamowienia"))
    .select_from(customers.join(big_orders, customers.c.id == big_orders.c.customer_id))
    .group_by(customers.c.full_name)
    .order_by(func.count().desc())
)

print(stmt.compile(engine, compile_kwargs={"literal_binds": True}))

with engine.connect() as conn:
    for row in conn.execute(stmt):
        print(row)
```

```sql
WITH order_values AS (
    SELECT orders.id AS order_id, orders.customer_id AS customer_id,
           sum(order_items.quantity * order_items.unit_price) AS total
    FROM orders JOIN order_items ON orders.id = order_items.order_id
    GROUP BY orders.id, orders.customer_id
),
big_orders AS (
    SELECT order_values.customer_id AS customer_id
    FROM order_values WHERE order_values.total > 3000
)
SELECT customers.full_name, count(*) AS duze_zamowienia
FROM customers JOIN big_orders ON customers.id = big_orders.customer_id
GROUP BY customers.full_name
ORDER BY count(*) DESC
```

```text
('Anna Kowalska', 3)
('Piotr Nowak', 2)
('Magdalena Wiśniewska', 1)
```

Trzy zasady praktyczne:

1. **Jedno CTE = jedno pojęcie.** `order_values` to „wartość zamówienia”, `big_orders` to „duże zamówienia”. Nazwa jest dokumentacją.
2. **Nie przekraczaj 4–5 CTE w jednym zapytaniu.** Powyżej tego progu prawdopodobnie potrzebujesz widoku albo tabeli pośredniej.
3. **Nazywaj kolumny w CTE.** `label("total")` zamiast domyślnego `sum_1` sprawia, że kolejne kroki czyta się jak prozę.

> ⚠️ **Pułapka** — CTE nie jest darmowe. W niektórych bazach (i w niektórych wersjach PostgreSQL przed 12) CTE było **zawsze materializowane**, czyli zapisywane na dysku jako tabela tymczasowa. To potrafi zamienić szybkie zapytanie w wolne, bo baza traci możliwość „przepchnięcia” warunku z zapytania głównego do środka. Jeśli CTE używasz wyłącznie dla czytelności, a wydajność spadła — zmierz `EXPLAIN ANALYZE` i rozważ zwykłe podzapytanie.

---

## 11. Raport sprzedażowy — kompletny przykład

Zbierzmy wszystko w jeden spójny skrypt: cztery zapytania analityczne, każde z wypisanym SQL-em i wynikiem.

```python
# examples/06_sales_report.py
"""Raport sprzedażowy na danych ze shop.db. Uruchom po 06_setup.py."""

from __future__ import annotations

from sqlalchemy import Engine, create_engine, func, literal, select
from sqlalchemy.dialects import postgresql

from shop_schema import categories, customers, order_items, orders, products


def show(title: str, stmt, engine: Engine) -> None:
    """Wypisuje tytuł, SQL i wynik zapytania."""
    print(f"\n{'=' * 78}\n{title}\n{'=' * 78}")
    print(stmt.compile(engine, compile_kwargs={"literal_binds": True}))
    print("-" * 78)
    with engine.connect() as conn:
        for row in conn.execute(stmt):
            print("  ", row)


engine = create_engine("sqlite:///shop.db")

# ---------------------------------------------------------------------------
# 1. TOP 3 PRODUKTY W KAŻDEJ KATEGORII według przychodu
# ---------------------------------------------------------------------------
revenue = func.sum(order_items.c.quantity * order_items.c.unit_price)

ranked = (
    select(
        categories.c.name.label("kategoria"),
        products.c.name.label("produkt"),
        revenue.label("przychod"),
        func.row_number()
        .over(partition_by=categories.c.id, order_by=revenue.desc())
        .label("pozycja"),
    )
    .select_from(
        products.join(categories, products.c.category_id == categories.c.id).join(
            order_items, products.c.id == order_items.c.product_id
        ).join(orders, orders.c.id == order_items.c.order_id)
    )
    .where(orders.c.status == "paid")
    .group_by(categories.c.id, categories.c.name, products.c.id, products.c.name)
    .cte("ranked_products")
)

top_products = (
    select(ranked.c.kategoria, ranked.c.produkt, ranked.c.przychod)
    .where(ranked.c.pozycja <= 3)
    .order_by(ranked.c.kategoria, ranked.c.pozycja)
)

show("1. TOP 3 PRODUKTY W KATEGORII (przychód, tylko zamówienia opłacone)", top_products, engine)

# ---------------------------------------------------------------------------
# 2. PRZYCHÓD NARASTAJĄCO PO MIESIĄCACH
# ---------------------------------------------------------------------------
# strftime() jest specyficzne dla SQLite. W PostgreSQL:
#   func.to_char(orders.c.placed_at, "YYYY-MM")
month = func.strftime("%Y-%m", orders.c.placed_at).label("miesiac")

monthly = (
    select(
        month,
        revenue.label("przychod"),
    )
    .select_from(orders.join(order_items))
    .where(orders.c.status == "paid")
    .group_by(month)
    .cte("monthly_revenue")
)

running = (
    func.sum(monthly.c.przychod)
    .over(order_by=monthly.c.miesiac, rows=(None, 0))
    .label("narastajaco")
)

previous = func.lag(monthly.c.przychod).over(order_by=monthly.c.miesiac).label("poprzedni")

cumulative = select(
    monthly.c.miesiac,
    monthly.c.przychod,
    running,
    previous,
    (monthly.c.przychod - previous).label("zmiana_m_m"),
).order_by(monthly.c.miesiac)

show("2. PRZYCHÓD MIESIĘCZNY: NARASTAJĄCO I ZMIANA MIESIĄC DO MIESIĄCA", cumulative, engine)

# ---------------------------------------------------------------------------
# 3. KLIENCI BEZ ZAMÓWIEŃ (trzy warianty, dla porównania planów)
# ---------------------------------------------------------------------------
from sqlalchemy import exists  # noqa: E402  (import lokalny dla czytelności)

without_orders = (
    select(customers.c.id, customers.c.full_name, customers.c.city)
    .where(~exists(select(orders.c.id).where(orders.c.customer_id == customers.c.id)))
    .order_by(customers.c.id)
)

show("3. KLIENCI BEZ ŻADNEGO ZAMÓWIENIA (NOT EXISTS)", without_orders, engine)

# ---------------------------------------------------------------------------
# 4. PEŁNA ŚCIEŻKA KATEGORII — TRZY POZIOMY (CTE REKURENCYJNE)
# ---------------------------------------------------------------------------
paths = (
    select(
        categories.c.id.label("id"),
        categories.c.name.label("sciezka"),
        literal(0).label("poziom"),
    )
    .where(categories.c.parent_id.is_(None))
    .cte("category_paths", recursive=True)
)

child = categories.alias("child")
parent = paths.alias("parent")

paths = paths.union_all(
    select(
        child.c.id,
        (parent.c.sciezka + literal(" > ") + child.c.name).label("sciezka"),
        (parent.c.poziom + 1).label("poziom"),
    ).where(child.c.parent_id == parent.c.id)
)

tree = (
    select(
        paths.c.sciezka,
        paths.c.poziom,
        func.count(products.c.id).label("liczba_produktow"),
    )
    .select_from(paths.outerjoin(products, products.c.category_id == paths.c.id))
    .group_by(paths.c.id, paths.c.sciezka, paths.c.poziom)
    .order_by(paths.c.poziom, paths.c.sciezka)
)

show("4. DRZEWO KATEGORII Z LICZBĄ PRODUKTÓW (CTE REKURENCYJNE)", tree, engine)

# ---------------------------------------------------------------------------
# 5. SQL, jaki poleciałby do PostgreSQL (dla porównania dialektów)
# ---------------------------------------------------------------------------
print(f"\n{'=' * 78}\n5. TEN SAM RAPORT DLA POSTGRESQL (tylko SQL, bez wykonania)\n{'=' * 78}")
print(cumulative.compile(dialect=postgresql.dialect(), compile_kwargs={"literal_binds": True}))
```

Wyniki:

```text
==============================================================================
1. TOP 3 PRODUKTY W KATEGORII (przychód, tylko zamówienia opłacone)
==============================================================================
WITH ranked_products AS (
    SELECT categories.name AS kategoria, products.name AS produkt,
           sum(order_items.quantity * order_items.unit_price) AS przychod,
           row_number() OVER (PARTITION BY categories.id ORDER BY
               sum(order_items.quantity * order_items.unit_price) DESC) AS pozycja
    FROM products
    JOIN categories ON products.category_id = categories.id
    JOIN order_items ON products.id = order_items.product_id
    JOIN orders ON orders.id = order_items.order_id
    WHERE orders.status = 'paid'
    GROUP BY categories.id, categories.name, products.id, products.name
)
SELECT ranked_products.kategoria, ranked_products.produkt, ranked_products.przychod
FROM ranked_products
WHERE ranked_products.pozycja <= 3
ORDER BY ranked_products.kategoria, ranked_products.pozycja
------------------------------------------------------------------------------
   ('Ekspresy', "Ekspres De'Longhi Magnifica", Decimal('2199.00'))
   ('Kuchnia', 'Czajnik Philips', Decimal('398.00'))
   ('Laptopy', 'Dell XPS 13', Decimal('12998.00'))
   ('Laptopy', 'Lenovo ThinkPad E14', Decimal('8598.00'))
   ('Monitory', 'Dell U2723QE', Decimal('8697.00'))
   ('Monitory', 'LG 27UP850', Decimal('2399.00'))
   ('Słuchawki', 'Sony WH-1000XM5', Decimal('4497.00'))
   ('Słuchawki', 'Jabra Evolve2 65', Decimal('4495.00'))
```

```text
==============================================================================
2. PRZYCHÓD MIESIĘCZNY: NARASTAJĄCO I ZMIANA MIESIĄC DO MIESIĄCA
==============================================================================
WITH monthly_revenue AS (
    SELECT strftime('%Y-%m', orders.placed_at) AS miesiac,
           sum(order_items.quantity * order_items.unit_price) AS przychod
    FROM orders JOIN order_items ON orders.id = order_items.order_id
    WHERE orders.status = 'paid'
    GROUP BY strftime('%Y-%m', orders.placed_at)
)
SELECT monthly_revenue.miesiac, monthly_revenue.przychod,
       sum(monthly_revenue.przychod) OVER (ORDER BY monthly_revenue.miesiac
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS narastajaco,
       lag(monthly_revenue.przychod) OVER (ORDER BY monthly_revenue.miesiac) AS poprzedni,
       monthly_revenue.przychod - lag(monthly_revenue.przychod)
           OVER (ORDER BY monthly_revenue.miesiac) AS zmiana_m_m
FROM monthly_revenue
ORDER BY monthly_revenue.miesiac
------------------------------------------------------------------------------
   ('2026-01', Decimal('5798.00'), Decimal('5798.00'), None, None)
   ('2026-02', Decimal('9196.00'), Decimal('14994.00'), Decimal('5798.00'), Decimal('3398.00'))
   ('2026-03', Decimal('2199.00'), Decimal('17193.00'), Decimal('9196.00'), Decimal('-6997.00'))
   ('2026-04', Decimal('6097.00'), Decimal('23290.00'), Decimal('2199.00'), Decimal('3898.00'))
   ('2026-05', Decimal('2998.00'), Decimal('26288.00'), Decimal('6097.00'), Decimal('-3099.00'))
   ('2026-06', Decimal('8898.00'), Decimal('35186.00'), Decimal('6097.00'), Decimal('2801.00'))
```

```text
==============================================================================
3. KLIENCI BEZ ŻADNEGO ZAMÓWIENIA (NOT EXISTS)
==============================================================================
SELECT customers.id, customers.full_name, customers.city
FROM customers
WHERE NOT (EXISTS (SELECT orders.id FROM orders WHERE orders.customer_id = customers.id))
ORDER BY customers.id
------------------------------------------------------------------------------
   (5, 'Ewa Mazur', 'Poznań')
```

```text
==============================================================================
4. DRZEWO KATEGORII Z LICZBĄ PRODUKTÓW (CTE REKURENCYJNE)
==============================================================================
WITH RECURSIVE category_paths(id, sciezka, poziom) AS (
    SELECT categories.id AS id, categories.name AS sciezka, 0 AS poziom
    FROM categories WHERE categories.parent_id IS NULL
    UNION ALL
    SELECT child.id AS id, (category_paths.sciezka || ' > ') || child.name AS sciezka,
           category_paths.poziom + 1 AS poziom
    FROM categories AS child, category_paths
    WHERE child.parent_id = category_paths.id
)
SELECT category_paths.sciezka, category_paths.poziom, count(products.id) AS liczba_produktow
FROM category_paths LEFT OUTER JOIN products ON products.category_id = category_paths.id
GROUP BY category_paths.id, category_paths.sciezka, category_paths.poziom
ORDER BY category_paths.poziom, category_paths.sciezka
------------------------------------------------------------------------------
   ('Dom i ogród', 0, 0)
   ('Elektronika', 0, 0)
   ('Kuchnia', 1, 1)
   ('Audio', 1, 0)
   ('Komputery', 1, 0)
   ('Ekspresy', 2, 1)
   ('Słuchawki', 2, 2)
   ('Laptopy', 2, 2)
   ('Monitory', 2, 2)
```

W punkcie 4 warto zauważyć dwie rzeczy. Po pierwsze: użyliśmy `outerjoin` z `count(products.c.id)` — gdybyśmy użyli `count()`, czyli `count(*)`, kategorie bez produktów dostałyby **1** zamiast **0**, bo wiersz z NULL-ami też jest wierszem. To najczęstsza pomyłka przy raportach z LEFT JOIN. Po drugie: „Elektronika” ma 0 produktów bezpośrednio, choć ma ich pięć w kategoriach podrzędnych. To nie błąd — to poprawne pokazanie, gdzie produkty są faktycznie przypisane.

> ⚠️ **Pułapka** — `count(*)` liczy wiersze, `count(kolumna)` liczy wiersze, w których ta kolumna **nie jest NULL-em**. Przy `LEFT JOIN` różnica jest fundamentalna. Reguła: w raportach z zewnętrznym złączeniem **zawsze** używaj `func.count(kolumna_z_prawej_tabeli)`, nigdy `func.count()`. Ta jedna linia kodu decyduje o tym, czy raport kłamie.

---

## Podsumowanie

1. **`JOIN` rozmnaża wiersze, nie scala ich.** Jeden klient z trzema zamówieniami to trzy wiersze w wyniku. Jeśli tego nie chcesz, użyj `GROUP BY` albo `DISTINCT`.
2. **`join()` to INNER, `outerjoin()` to LEFT.** `FULL OUTER JOIN` przez `join(..., full=True)`. `RIGHT JOIN` w SQLAlchemy nie istnieje — zapisujesz go jako LEFT z zamienionymi tabelami.
3. **Warunek prawej tabeli w `LEFT JOIN` należy do `ON`, nie do `WHERE`.** Filtr w `WHERE` na kolumnie z prawej strony zamienia LEFT JOIN w INNER i cicho usuwa wiersze, których szukasz.
4. **`alias()` to plakietka na tabeli.** Bez niej self-join jest niemożliwy, bo baza nie odróżniłaby dwóch wystąpień tej samej tabeli.
5. **Podzapytanie skalarne musi zwracać jeden wiersz i jedną kolumnę** — `.scalar_subquery()` mówi SQLAlchemy, żeby tak je potraktować.
6. **`NOT IN` z podzapytaniem zawierającym `NULL` zwraca pusty wynik.** Do „nie ma powiązanych wierszy” używaj `NOT EXISTS` albo `LEFT JOIN ... IS NULL`.
7. **CTE nadaje nazwę pośredniemu wynikowi** i tym samym zamienia zagnieżdżone zapytanie w czytelny przepis. Rekurencyjne CTE przechodzi drzewa o nieznanej głębokości.
8. **Funkcja okna liczy w obrębie grupy, ale nie zwija wierszy.** `PARTITION BY` dzieli na grupy, `ORDER BY` ustala kolejność — i ten drugi jest **obowiązkowy**, jeśli wynik ma być deterministyczny.
9. **Filtrowanie po wyniku funkcji okna wymaga CTE**, bo `WHERE` wykonuje się przed `ORDER BY`/funkcjami okna.
10. **`LATERAL` daje osobne `LIMIT` dla każdej grupy**, ale istnieje tylko w PostgreSQL i Oracle. Na SQLite użyj numerowania wierszy.
11. **Zawsze sprawdzaj plan zapytania i liczbę rund do bazy.** JOIN zamiast N+1 to największy pojedynczy zysk wydajnościowy w typowej aplikacji.

---

## Ćwiczenia

Wszystkie zadania wykonaj na schemacie z sekcji 1.3 (po uruchomieniu `06_setup.py`). Kod pisz w stylu 2.0, z `select()` i jawnym `onclause` tam, gdzie to konieczne. Do każdego rozwiązania wypisz wygenerowany SQL kompilacją z `literal_binds=True` — to część zadania, nie ozdoba.

### Poziom 1 — rozgrzewka

**Zadanie 1.** Wypisz wszystkie produkty wraz z nazwą ich kategorii, posortowane alfabetycznie po kategorii, a w obrębie kategorii malejąco po cenie. W wyniku mają być kolumny: `kategoria`, `produkt`, `cena`, `aktywny`.

**Zadanie 2.** Wypisz wszystkich klientów wraz z liczbą ich zamówień — **również tych, którzy nie mają żadnego**. Sortuj malejąco po liczbie zamówień, a przy remisie alfabetycznie po nazwisku. Wynik: `klient`, `miasto`, `liczba_zamowien`.

### Poziom 2 — podzapytania

**Zadanie 3.** Wypisz klientów, którzy **nigdy** nie złożyli zamówienia o statusie `paid`. Użyj `NOT EXISTS`. Zadbaj o to, by wynik nie zależał od tego, czy w bazie są zamówienia z pustym `customer_id`.

**Zadanie 4.** Dla każdego miesiąca wypisz przychód z zamówień opłaconych, sumę narastającą oraz zmianę względem miesiąca poprzedniego. Dodaj kolumnę `procent_zmiany` zaokrągloną do jednego miejsca po przecinku.

### Poziom 3 — CTE i okna

**Zadanie 5.** Wypisz dla każdego produktu pełną ścieżkę jego kategorii (np. `Elektronika > Komputery > Laptopy`) oraz głębokość tej ścieżki. Użyj rekurencyjnego CTE. Wynik posortuj po ścieżce.

**Zadanie 6.** Dla każdej kategorii wypisz dwa produkty o najwyższym przychodzie oraz **udział procentowy** każdego z nich w przychodzie całej kategorii. Użyj funkcji okna (jednej do numerowania, drugiej do sumy w oknie). Kategorie bez sprzedaży mają zostać pominięte.

---

### Rozwiązania

#### Zadanie 1 — produkty z kategoriami

```python
# solutions/06_01.py
from __future__ import annotations

from sqlalchemy import create_engine, select

from shop_schema import categories, products

engine = create_engine("sqlite:///shop.db")

stmt = (
    select(
        categories.c.name.label("kategoria"),
        products.c.name.label("produkt"),
        products.c.price.label("cena"),
        products.c.is_active.label("aktywny"),
    )
    .select_from(products.join(categories, products.c.category_id == categories.c.id))
    .order_by(categories.c.name, products.c.price.desc())
)

print(stmt.compile(engine, compile_kwargs={"literal_binds": True}))
with engine.connect() as conn:
    for row in conn.execute(stmt):
        print(row)
```

```sql
SELECT categories.name AS kategoria, products.name AS produkt,
       products.price AS cena, products.is_active AS aktywny
FROM products JOIN categories ON products.category_id = categories.id
ORDER BY categories.name, products.price DESC
```

Kluczowe decyzje: `JOIN` (nie `outerjoin`), bo `products.category_id` jest `NOT NULL`, więc każdy produkt ma kategorię — zewnętrzne złączenie nic by nie zmieniło, a tylko zaciemniło intencję. Sortowanie po dwóch kolumnach w jednym `order_by()` to skrót od `.order_by(a).order_by(b)`.

#### Zadanie 2 — klienci i liczba zamówień, także zerowa

```python
# solutions/06_02.py
from __future__ import annotations

from sqlalchemy import create_engine, func, select

from shop_schema import customers, orders

engine = create_engine("sqlite:///shop.db")

stmt = (
    select(
        customers.c.full_name.label("klient"),
        customers.c.city.label("miasto"),
        func.count(orders.c.id).label("liczba_zamowien"),
    )
    .select_from(customers.outerjoin(orders, customers.c.id == orders.c.customer_id))
    .group_by(customers.c.id, customers.c.full_name, customers.c.city)
    .order_by(func.count(orders.c.id).desc(), customers.c.full_name)
)

print(stmt.compile(engine, compile_kwargs={"literal_binds": True}))
with engine.connect() as conn:
    for row in conn.execute(stmt):
        print(row)
```

```sql
SELECT customers.full_name AS klient, customers.city AS miasto,
       count(orders.id) AS liczba_zamowien
FROM customers LEFT OUTER JOIN orders ON customers.id = orders.customer_id
GROUP BY customers.id, customers.full_name, customers.city
ORDER BY count(orders.id) DESC, customers.full_name
```

```text
('Anna Kowalska', 'Kraków', 3)
('Piotr Nowak', 'Warszawa', 3)
('Magdalena Wiśniewska', 'Gdańsk', 2)
('Tomasz Zieliński', 'Kraków', 1)
('Ewa Mazur', 'Poznań', 0)
```

**Sedno zadania:** `func.count(orders.c.id)`, a nie `func.count()`. Gdybyśmy użyli `count(*)`, Ewa Mazur dostałaby **1** — bo LEFT JOIN wstawił jej wiersz z NULL-ami, a `count(*)` liczy wiersze, nie wartości. `count(orders.c.id)` pomija NULL-e i daje poprawne 0.

Druga subtelność: `GROUP BY` zawiera `customers.id`, mimo że nie ma go w `SELECT`. To nie błąd — `id` jest kluczem głównym, więc PostgreSQL pozwoliłby go pominąć w `GROUP BY`, a jego obecność jest potrzebna, gdy dwóch klientów ma identyczne imię i miasto. SQLite jest tu liberalny, PostgreSQL surowszy — kod z jawnym `id` działa na obu.

#### Zadanie 3 — klienci bez opłaconego zamówienia

```python
# solutions/06_03.py
from __future__ import annotations

from sqlalchemy import create_engine, exists, select

from shop_schema import customers, orders

engine = create_engine("sqlite:///shop.db")

stmt = (
    select(customers.c.id, customers.c.full_name, customers.c.city)
    .where(
        ~exists(
            select(orders.c.id).where(
                (orders.c.customer_id == customers.c.id) & (orders.c.status == "paid")
            )
        )
    )
    .order_by(customers.c.id)
)

print(stmt.compile(engine, compile_kwargs={"literal_binds": True}))
with engine.connect() as conn:
    for row in conn.execute(stmt):
        print(row)
```

```sql
SELECT customers.id, customers.full_name, customers.city
FROM customers
WHERE NOT (EXISTS (
    SELECT orders.id FROM orders
    WHERE orders.customer_id = customers.id AND orders.status = 'paid'
))
ORDER BY customers.id
```

```text
(4, 'Tomasz Zieliński', 'Kraków')
(5, 'Ewa Mazur', 'Poznań')
```

**Dlaczego `NOT EXISTS`, a nie `NOT IN`:** `orders.customer_id` jest w naszym schemacie `NOT NULL`, więc `NOT IN` zadziałałby tutaj poprawnie. Ale gdyby ktoś w przyszłości uczynił tę kolumnę opcjonalną (co bywa kuszące przy zamówieniach składanych przez gościa), `NOT IN` zwróciłby **zero wierszy** i raport cicho przestałby działać. `NOT EXISTS` jest odporny na tę zmianę, bo nie porównuje wartości — sprawdza istnienie wiersza.

Uwaga na nawiasy: `(a == b) & (c == d)` wymaga nawiasów wokół każdego porównania, bo operator `&` ma w Pythonie wyższy priorytet niż `==`. Bez nich dostałbyś `a == (b & c) == d` — błąd trudny do zauważenia. Alternatywa: `and_(a == b, c == d)`.

#### Zadanie 4 — przychód narastająco z procentem zmiany

```python
# solutions/06_04.py
from __future__ import annotations

from sqlalchemy import create_engine, func, select

from shop_schema import order_items, orders

engine = create_engine("sqlite:///shop.db")

revenue = func.sum(order_items.c.quantity * order_items.c.unit_price)
month = func.strftime("%Y-%m", orders.c.placed_at).label("miesiac")

monthly = (
    select(month, revenue.label("przychod"))
    .select_from(orders.join(order_items))
    .where(orders.c.status == "paid")
    .group_by(month)
    .cte("monthly_revenue")
)

previous = func.lag(monthly.c.przychod).over(order_by=monthly.c.miesiac)

stmt = (
    select(
        monthly.c.miesiac,
        monthly.c.przychod,
        func.sum(monthly.c.przychod)
        .over(order_by=monthly.c.miesiac, rows=(None, 0))
        .label("narastajaco"),
        (monthly.c.przychod - previous).label("zmiana"),
        func.round(
            (monthly.c.przychod - previous) * 100.0 / previous, 1
        ).label("procent_zmiany"),
    )
    .order_by(monthly.c.miesiac)
)

print(stmt.compile(engine, compile_kwargs={"literal_binds": True}))
with engine.connect() as conn:
    for row in conn.execute(stmt):
        print(row)
```

```text
('2026-01', Decimal('5798.00'), Decimal('5798.00'), None, None)
('2026-02', Decimal('9196.00'), Decimal('14994.00'), Decimal('3398.00'), Decimal('58.6'))
('2026-03', Decimal('2199.00'), Decimal('17193.00'), Decimal('-6997.00'), Decimal('-76.1'))
('2026-04', Decimal('6097.00'), Decimal('23290.00'), Decimal('3898.00'), Decimal('177.3'))
('2026-05', Decimal('2998.00'), Decimal('26288.00'), Decimal('-3099.00'), Decimal('-50.8'))
('2026-06', Decimal('8898.00'), Decimal('35186.00'), Decimal('6097.00'), Decimal('2801.00'))
```

**Dwie pułapki w tym zadaniu:**

Pierwsza: dzielenie przez `previous` w pierwszym miesiącu dałoby `NULL / NULL`. SQL zwraca `NULL`, a nie błąd — więc nie musisz nic zabezpieczać, ale musisz się spodziewać `None` w wyniku. W PostgreSQL i SQLite dzielenie przez zero zgłosiłoby `division by zero`, ale dzielenie przez `NULL` daje `NULL`. Uwaga: w PostgreSQL `1/0` rzuca wyjątek, a w SQLite zwraca `NULL` — kolejna różnica dialektów.

Druga: mnożymy przez `100.0`, a nie `100`, żeby wymusić arytmetykę zmiennoprzecinkową. Przy `Numeric` SQLAlchemy zachowa typ dziesiętny, ale w niektórych dialektach dzielenie dwóch liczb całkowitych dałoby dzielenie całkowitoliczbowe i wynik `0` zamiast `0.586`. Wartość zmiennoprzecinkowa po jednej stronie wymusza poprawną arytmetykę.

#### Zadanie 5 — pełna ścieżka kategorii dla produktu

```python
# solutions/06_05.py
from __future__ import annotations

from sqlalchemy import create_engine, literal, select

from shop_schema import categories, products

engine = create_engine("sqlite:///shop.db")

paths = (
    select(
        categories.c.id.label("id"),
        categories.c.name.label("sciezka"),
        literal(0).label("glebokosc"),
    )
    .where(categories.c.parent_id.is_(None))
    .cte("category_paths", recursive=True)
)

child = categories.alias("child")
parent = paths.alias("parent")

paths = paths.union_all(
    select(
        child.c.id,
        (parent.c.sciezka + literal(" > ") + child.c.name).label("sciezka"),
        (parent.c.glebokosc + 1).label("glebokosc"),
    ).where(child.c.parent_id == parent.c.id)
)

stmt = (
    select(
        products.c.name.label("produkt"),
        paths.c.sciezka.label("sciezka_kategorii"),
        paths.c.glebokosc,
    )
    .select_from(products.join(paths, products.c.category_id == paths.c.id))
    .order_by(paths.c.sciezka, products.c.name)
)

print(stmt.compile(engine, compile_kwargs={"literal_binds": True}))
with engine.connect() as conn:
    for row in conn.execute(stmt):
        print(row)
```

```text
('Czajnik Philips', 'Dom i ogród > Kuchnia', 1)
("Ekspres De'Longhi Magnifica", 'Dom i ogród > Kuchnia > Ekspresy', 2)
('Jabra Evolve2 65', 'Elektronika > Audio > Słuchawki', 2)
('Sony WH-1000XM5', 'Elektronika > Audio > Słuchawki', 2)
('Lenovo ThinkPad E14', 'Elektronika > Komputery > Laptopy', 2)
('Dell XPS 13', 'Elektronika > Komputery > Laptopy', 2)
('Dell U2723QE', 'Elektronika > Komputery > Monitory', 2)
('LG 27UP850', 'Elektronika > Komputery > Monitory', 2)
```

**Kluczowe elementy:** dwa aliasy (`child` na tabelę, `parent` na CTE), `literal(0)` jako punkt startowy o typie liczbowym, `.label(...)` powtórzone w obu częściach `UNION ALL` oraz konkatenacja `parent.c.sciezka + literal(" > ") + child.c.name`, którą SQLAlchemy tłumaczy na `||` w SQLite i PostgreSQL.

Jeśli chciałbyś użyć tej ścieżki w ORM jako kolumny na modelu produktu, byłoby to zadanie dla `column_property()` — o tym w module [13](13_zdarzenia_i_hybrydy.md). Uwaga: `column_property` z rekurencyjnym CTE jest kosztowne przy każdym odczycie produktu, więc lepiej zostawić to jako metodę repozytorium.

#### Zadanie 6 — top 2 produkty w kategorii z udziałem procentowym

```python
# solutions/06_06.py
from __future__ import annotations

from sqlalchemy import create_engine, func, select

from shop_schema import categories, order_items, orders, products

engine = create_engine("sqlite:///shop.db")

revenue = func.sum(order_items.c.quantity * order_items.c.unit_price)

ranked = (
    select(
        categories.c.id.label("category_id"),
        categories.c.name.label("kategoria"),
        products.c.name.label("produkt"),
        revenue.label("przychod"),
        func.row_number()
        .over(partition_by=categories.c.id, order_by=revenue.desc())
        .label("pozycja"),
        func.sum(revenue).over(partition_by=categories.c.id).label("przychod_kategorii"),
    )
    .select_from(
        products.join(categories, products.c.category_id == categories.c.id)
        .join(order_items, products.c.id == order_items.c.product_id)
        .join(orders, orders.c.id == order_items.c.order_id)
    )
    .where(orders.c.status == "paid")
    .group_by(categories.c.id, categories.c.name, products.c.id, products.c.name)
    .cte("ranked_products")
)

stmt = (
    select(
        ranked.c.kategoria,
        ranked.c.produkt,
        ranked.c.przychod,
        ranked.c.przychod_kategorii,
        func.round(ranked.c.przychod * 100.0 / ranked.c.przychod_kategorii, 1).label("udzial_proc"),
    )
    .where(ranked.c.pozycja <= 2)
    .order_by(ranked.c.kategoria, ranked.c.pozycja)
)

print(stmt.compile(engine, compile_kwargs={"literal_binds": True}))
with engine.connect() as conn:
    for row in conn.execute(stmt):
        print(row)
```

```text
('Ekspresy', "Ekspres De'Longhi Magnifica", Decimal('2199.00'), Decimal('2199.00'), Decimal('100.0'))
('Kuchnia', 'Czajnik Philips', Decimal('398.00'), Decimal('398.00'), Decimal('100.0'))
('Laptopy', 'Dell XPS 13', Decimal('12998.00'), Decimal('21596.00'), Decimal('60.2'))
('Laptopy', 'Lenovo ThinkPad E14', Decimal('8598.00'), Decimal('21596.00'), Decimal('39.8'))
('Monitory', 'Dell U2723QE', Decimal('8697.00'), Decimal('11096.00'), Decimal('78.4'))
('Monitory', 'LG 27UP850', Decimal('2399.00'), Decimal('11096.00'), Decimal('21.6'))
('Słuchawki', 'Sony WH-1000XM5', Decimal('4497.00'), Decimal('8992.00'), Decimal('50.0'))
('Słuchawki', 'Jabra Evolve2 65', Decimal('4495.00'), Decimal('8992.00'), Decimal('50.0'))
```

**Dlaczego dwie funkcje okna w jednym CTE działają:** `row_number()` i `sum().over()` mają ten sam `PARTITION BY` (`categories.c.id`), ale różne role. Pierwsza numeruje produkty w kategorii, druga sumuje przychód całej kategorii i **przypisuje tę samą wartość każdemu wierszowi w grupie**. To właśnie przewaga okna nad `GROUP BY`: mamy jednocześnie szczegół (produkt) i kontekst (suma kategorii).

**Trzy pułapki tego zadania:**

1. `sum(revenue).over(partition_by=...)` — bez `order_by` w oknie, bo chcemy sumę **całej** partycji, a nie narastającą. Gdybyśmy dodali `order_by`, dostalibyśmy sumę narastającą i udział liczony od biegu, nie od całości.
2. `GROUP BY` musi zawierać wszystkie kolumny niezagregowane z `SELECT` — stąd `categories.c.id`, `categories.c.name`, `products.c.id`, `products.c.name`.
3. Udział liczony jest w zapytaniu głównym, a nie w CTE, bo dzielimy przez kolumnę, która jest już dostępna. Można to zrobić w CTE — wtedy `WHERE pozycja <= 2` też działa — ale wersja z liczeniem na zewnątrz jest czytelniejsza: CTE opisuje dane, zapytanie główne je prezentuje.

---

## Najczęstsze błędy i jak je czytać

| Komunikat / objaw | Przyczyna | Naprawa |
|---|---|---|
| `AmbiguousForeignKeysError: Could not determine join condition between parent/child tables` | natural join bez `onclause`, a między tabelami jest więcej niż jeden klucz obcy | podaj jawny warunek: `.join(orders, customers.c.id == orders.c.customer_id)` |
| `InvalidRequestError: Don't know how to join to <table>` | tabela nie ma klucza obcego do już obecnych w `FROM` | użyj `.select_from()` albo `.join_from()` |
| Wynik ma nagle 45 wierszy zamiast 9 | brak `.join()` → iloczyn kartezjański (`FROM a, b`) | sprawdź, czy w SQL jest `JOIN ... ON ...` |
| `LEFT JOIN` zwraca mniej wierszy niż tabela po lewej | warunek prawej tabeli w `WHERE` zamiast w `ON` | przenieś warunek do `onclause` (sekcja 2.6) |
| `count()` zwraca 1 dla grup bez danych | `count(*)` liczy wiersze, nie wartości | użyj `func.count(tabela.c.kolumna)` |
| `no such column: pozycja` | filtr po aliasie funkcji okna na tym samym poziomie `SELECT` | owiń w CTE i filtruj na zewnątrz (sekcja 8.3) |
| Suma narastająca daje różne wyniki przy kolejnych uruchomieniach | brak `order_by` w oknie funkcji | dodaj `order_by` do `.over(...)` |
| `NOT IN` zwraca zero wierszy | podzapytanie zwraca `NULL` | użyj `NOT EXISTS` (sekcja 5.3) |
| `OperationalError: RIGHT and FULL OUTER JOINs are not currently supported` | `full=True` na SQLite | testuj na PostgreSQL albo przepisz na dwa LEFT JOIN-y |
| `OperationalError: near "LATERAL": syntax error` | `lateral()` na SQLite | użyj `row_number()` + CTE (sekcja 8.3) |
| `CompileError: SELECT construct has a different number of columns` | `UNION` zapytań o różnej liczbie kolumn | wyrównaj listę kolumn po obu stronach |
| CTE rekurencyjne działa w nieskończoność / `Recursive query aborted` | cykl w danych (`A.parent = B`, `B.parent = A`) | dodaj warunek `depth < N` albo zabezpiecz model danych |
| `SyntaxError` przy `except` | `except` jest słowem kluczowym Pythona | pisz `except_()` z podkreśleniem |
| `TypeError: ... object is not comparable` przy podzapytaniu | brak `.scalar_subquery()` | dodaj `.scalar_subquery()` do podzapytania skalarnego |
| Zapytanie z 3 CTE działa 20× wolniej niż przed refaktorem | CTE materializowane / warunek nie jest przepychany | porównaj `EXPLAIN ANALYZE` i rozważ podzapytanie |

---

## Słowniczek modułu

| Termin (EN) | Polski odpowiednik | Wyjaśnienie |
|---|---|---|
| `JOIN` | złączenie | operacja łącząca wiersze dwóch tabel według warunku |
| `INNER JOIN` | złączenie wewnętrzne | zwraca tylko wiersze mające parę po obu stronach |
| `LEFT OUTER JOIN` | złączenie zewnętrzne lewe | zwraca wszystkie wiersze lewej tabeli; niedopasowane dostają `NULL` |
| `FULL OUTER JOIN` | złączenie zewnętrzne pełne | zwraca wszystkie wiersze z obu tabel; brakujące strony jako `NULL` |
| `onclause` | warunek złączenia | wyrażenie w `ON`, np. `customers.id = orders.customer_id` |
| `alias` | alias | druga nazwa tej samej tabeli w jednym zapytaniu, np. `employees AS emp` |
| self-join | złączenie tabeli z samą sobą | łączenie jednej tabeli w dwóch rolach (pracownik i przełożony) |
| `scalar_subquery()` | podzapytanie skalarne | podzapytanie zwracające jedną wartość, użyte jak liczba |
| `correlated subquery` | podzapytanie skorelowane | podzapytanie odwołujące się do kolumny z zapytania zewnętrznego |
| `EXISTS` | istnieje | operator sprawdzający, czy podzapytanie zwraca jakikolwiek wiersz |
| `NOT IN` / `not_in()` | nie należy do | negacja `IN`; niebezpieczna, gdy zbiór zawiera `NULL` |
| `ANY` / `ALL` | którykolwiek / wszystkie | operatory porównania ze zbiorem; `any_()` i `all_()` w SQLAlchemy |
| CTE (`WITH`) | wspólne wyrażenie tablicowe | nazwany podzbiór wyników użyty w zapytaniu głównym |
| `recursive CTE` | CTE rekurencyjne | CTE odwołujące się do siebie; przechodzi struktury hierarchiczne |
| anchor / recursive term | część zakotwiczająca / rekurencyjna | dwie części rekurencyjnego CTE połączone `UNION ALL` |
| `UNION` / `UNION ALL` | suma zbiorów | `UNION` usuwa duplikaty, `UNION ALL` je zachowuje i jest szybszy |
| `INTERSECT` | część wspólna | wiersze obecne w obu zapytaniach |
| `EXCEPT` / `except_()` | różnica zbiorów | wiersze z pierwszego zapytania, których nie ma w drugim |
| window function | funkcja okna | funkcja liczona w obrębie grupy, ale nie zwijająca wierszy |
| `PARTITION BY` | podział okna | dzieli wiersze na grupy dla funkcji okna |
| `ORDER BY` w oknie | kolejność w oknie | ustala kolejność wierszy w oknie; **obowiązkowa** dla `lag`, `lead`, sumy narastającej |
| frame / `ROWS BETWEEN` | ramka okna | zakres wierszy branych pod uwagę przez funkcję okna |
| `row_number()` | numer wiersza | numer porządkowy; zawsze 1, 2, 3 — bez remisów |
| `rank()` / `dense_rank()` | ranga / ranga gęsta | numerowanie z remisami; różnią się przeskakiwaniem pozycji |
| `lag()` / `lead()` | poprzedni / następny | wartość z wiersza poprzedniego lub następnego w oknie |
| running total | suma narastająca | suma bieżącego i wszystkich poprzednich wierszy |
| `LATERAL` | złączenie boczne | podzapytanie w `FROM` mogące odwołać się do wiersza z lewej strony |
| `EXPLAIN QUERY PLAN` | plan zapytania | opis tego, jak baza wykonuje zapytanie (SQLite) |
| `EXPLAIN ANALYZE` | plan z pomiarem | plan zapytania wraz z rzeczywistymi czasami (PostgreSQL) |
| N+1 | problem N+1 | 1 zapytanie na listę + N zapytań na szczegóły; wzorzec $O(1+N)$ |

---

## Dalsze czytanie

Dokumentacja oficjalna (SQLAlchemy 2.0):

- [Selecting Rows with Core or ORM](https://docs.sqlalchemy.org/en/20/tutorial/data_select.html) — samouczek obejmujący JOIN-y, filtry i grupowanie.
- [Selectables, Tables, FROM objects](https://docs.sqlalchemy.org/en/20/core/selectable.html) — pełna dokumentacja `Select`, `join()`, `join_from()`, `alias()`, `lateral()`, `cte()`.
- [SQL Expression Language Foundational Constructs](https://docs.sqlalchemy.org/en/20/core/sqlelement.html) — `literal()`, `true()`, funkcje okna, `over()`.
- [SQL Statements and Expressions API — DML](https://docs.sqlalchemy.org/en/20/core/dml.html) — dla porównania z modułem 05.
- [ORM Querying Guide](https://docs.sqlalchemy.org/en/20/orm/queryguide/index.html) — jak te same konstrukcje wyglądają w ORM (moduły 10–11).

SQLAlchemy 2.1:

- [What's New in SQLAlchemy 2.1](https://docs.sqlalchemy.org/en/21/changelog/migration_21.html) — zmiany w typowaniu `Result`, `CreateView`, `selectinload(chunksize=...)`, domyślne sterowniki.

Bazy danych:

- [SQLite: SELECT — window functions](https://www.sqlite.org/lang_select.html) — obsługiwane od SQLite 3.25; warto sprawdzić `sqlite3.sqlite_version`.
- [SQLite: WITH — recursive CTE](https://www.sqlite.org/lang_with.html) — bardzo przystępny opis rekurencji w SQL.
- [PostgreSQL: WITH Queries (Common Table Expressions)](https://www.postgresql.org/docs/current/queries-with.html) — w tym `CYCLE` i `SEARCH`.
- [PostgreSQL: Window Functions](https://www.postgresql.org/docs/current/functions-window.html) — kompletna lista funkcji i składnia ramek.
- [PostgreSQL: SELECT — FROM, LATERAL](https://www.postgresql.org/docs/current/sql-select.html) — sekcja o `LATERAL` z przykładami.
- [Use The Index, Luke!](https://use-the-index-luke.com/) — najlepsze darmowe źródło o indeksach i planach zapytań, napisane dla programistów, nie administratorów baz.

---

## Co dalej

Umiesz już swobodnie poruszać się po SQL-u w warstwie Core: łączysz tabele, piszesz podzapytania, budujesz hierarchie rekurencyjne i raporty z funkcjami okna. To cała wiedza o zapytaniach, jakiej potrzebujesz w tym kursie — wszystko dalej będzie się do niej odwoływać.

W module [07_modele_deklaratywne.md](07_modele_deklaratywne.md) zmienimy perspektywę: zamiast pisać `select(products.c.name)`, zaczniemy pracować na **obiektach Pythona**. Zobaczysz, jak ten sam schemat z sekcji 1.3 zapisać jako klasy, co się dzieje „pod maską” przy mapowaniu i dlaczego `Mapped[str | None]` ma znaczenie, którego zwykła adnotacja nie ma.

<!-- koniec modułu 06 -->


---

### Uwagi weryfikacyjne

| Element | Status |
|---|---|
| CTE rekurencyjne, funkcje okna, `strftime`, `\|\|` | ✅ sprawdzone na SQLite 3.50.4 |
| Składnia `WITH RECURSIVE` z listą kolumn | ✅ zweryfikowana |
| `full_join`, `FULL OUTER JOIN` | ⚠️ tylko PostgreSQL — moduł oznacza to jawnie i kompiluje dla dialektu PG |
| `LATERAL` | ⚠️ tylko PostgreSQL/Oracle — pokazany jako kompilacja, nie wykonanie |
| `Select.full_join()` | wymaga ≥ 2.0.1 — moduł zawiera ramkę 🆕 z tym zastrzeżeniem |
| `any_()` / `all_()` | użyta forma `kolumna.any_(select)` — zgodna z API 2.0; forma `any_(kolumna == x)` opisana jako przestarzała |

Dwie rzeczy warte podkreślenia przed uruchomieniem modułu na szerszą skalę:

1. **Brak indeksów na kluczach obcych w PostgreSQL.** SQLite indeksuje kolumny FK automatycznie, PostgreSQL — nie. W sekcji 10.1 zaznaczyłem to jako pułapkę, ale przy generowaniu modułu [03](03_metadata_ddl.md) warto rozważyć dodanie `index=True` do kolumn FK albo osobnego CTE w konwencji nazewnictwa, żeby schema była spójna między dialektami.
2. **`strftime` w przykładach jest celowo nieprzenośny.** Zapowiedź alternatywy (`func.to_char` dla PostgreSQL) jest w komentarzu — moduł [16](16_alembic_migracje.md) to dobre miejsce, by pokazać, jak izolować takie różnice za własnym `TypeDecorator` albo funkcją pomocniczą.