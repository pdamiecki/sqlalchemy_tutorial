# Moduł 07 — Modele deklaratywne (mapowanie klas)

W tym module przestajesz patrzeć na bazę danych jak na worek wierszy i zaczynasz pracować na **obiektach Pythona**. Dowiesz się, dlaczego baza i Python „nie dogadują się” same z siebie, jak działa `DeclarativeBase`, po co istnieją `Mapped[]` i `mapped_column()`, jak pozbyć się powtarzalności przez `Annotated` i `type_annotation_map`, jak zbudować modele z automatycznym konstruktorem (`MappedAsDataclass`) oraz jak złożyć z nich wieloplikowy projekt z mixinami i wspólną `MetaData`. Na końcu zapiszesz ten sam schemat sklepu, który w module 03 powstał w czystym Core, jako komplet modeli deklaratywnych — i porównasz wygenerowany DDL, żeby zobaczyć, że pod spodem dzieje się dokładnie to samo.

> 🟡 **Poziom:** średni
> ⏱ **Czas:** ~180 minut
> 📚 **Wymagania wstępne:** [01 — Wprowadzenie](01_wprowadzenie.md), [02 — Środowisko i Engine](02_srodowisko_i_engine.md), [03 — MetaData, Table, DDL](03_metadata_ddl.md), [04 — select() i Result](04_select_i_result.md), [05 — DML](05_dml.md), [06 — Joiny i zaawansowane SQL](06_joiny_i_zaawansowane_sql.md)
> 📄 **Czego dotyczy ten plik:** warstwy ORM. Do tej pory pisałeś `Table(...)` ręcznie i czytałeś `Row`. Tutaj po raz pierwszy definiujesz **klasy**, a SQLAlchemy sama wygeneruje z nich tabele.

---

## Spis treści

1. [Wiersz to nie obiekt — problem impedancji obiektowo-relacyjnej](#1-wiersz-to-nie-obiekt--problem-impedancji-obiektowo-relacyjnej)
2. [`DeclarativeBase` — od klasy Pythona do tabeli](#2-declarativebase--od-klasy-pythona-do-tabeli)
3. [`Mapped[...]` i `mapped_column()`](#3-mapped-i-mapped_column)
4. [`Annotated` i `type_annotation_map` — koniec z powtarzaniem](#4-annotated-i-type_annotation_map--koniec-z-powtarzaniem)
5. [`__tablename__`, `__table_args__` i praca z gotową `Table`](#5-__tablename__-__table_args__-i-praca-z-gotową-table)
6. [Modyfikatory `mapped_column()` — pełny przegląd](#6-modyfikatory-mapped_column--pełny-przegląd)
7. [`MappedAsDataclass` — modele z automatycznym `__init__`](#7-mappedasdataclass--modele-z-automatycznym-__init__)
8. [Mixiny i `@declared_attr`](#8-mixiny-i-declared_attr)
9. [`registry`, `metadata` i podział projektu na pliki](#9-registry-metadata-i-podział-projektu-na-pliki)
10. [Typowanie i `mypy` na modelach](#10-typowanie-i-mypy-na-modelach)
11. [Przykład obowiązkowy: Core kontra ORM na tym samym schemacie](#11-przykład-obowiązkowy-core-kontra-orm-na-tym-samym-schemacie)
12. [Podsumowanie](#podsumowanie)
13. [Ćwiczenia](#ćwiczenia)
14. [Najczęstsze błędy i jak je czytać](#najczęstsze-błędy-i-jak-je-czytać)
15. [Słowniczek modułu](#słowniczek-modułu)
16. [Dalsze czytanie](#dalsze-czytanie)
17. [Co dalej](#co-dalej)

---

## 1. Wiersz to nie obiekt — problem impedancji obiektowo-relacyjnej

### 1.1 Dwie reprezentacje tego samego

W module 04 czytałeś dane tak:

```python
# examples/07_row_vs_object.py
from sqlalchemy import create_engine, text

engine = create_engine("sqlite:///:memory:")

with engine.begin() as conn:
    conn.execute(text("CREATE TABLE customer (id INTEGER PRIMARY KEY, name TEXT)"))
    conn.execute(
        text("INSERT INTO customer (id, name) VALUES (1, 'Anna'), (2, 'Bartek')")
    )

with engine.connect() as conn:
    result = conn.execute(text("SELECT id, name FROM customer"))
    for row in result:
        print(row.id, row.name)
```

To działa i to jest dobre narzędzie. Ale zauważ, czym jest `row`: to **krotka z etykietami**. Nie ma metod, nie ma historii, nie ma pojęcia „jestem klientem o numerze 1”. Jest to tylko zrzut dwóch wartości z jednego zapytania. Gdy zapytanie zwróci dwa razy tego samego klienta (np. w joincie), dostaniesz dwa niezależne obiekty, o których Python nie wie, że dotyczą tej samej osoby.

Teraz spójrz na to, co chciałbyś napisać jako programista aplikacji:

```python
# ... (pominięte dla zwięzłości — pełny kod w przykładzie obowiązkowym)
customer = Customer(name="Anna")
order = Order(customer=customer)
customer.addresses[0].city = "Kraków"
```

Chcesz mówić językiem swojej domeny: klient, zamówienie, adres. Chcesz, żeby obiekt miał metody (`customer.total_spent()`), żeby zmiana atrybutu była widoczna dla innych części programu, i żeby `customer is customer` było prawdą, gdy mówimy o tym samym rekordzie.

Baza danych natomiast mówi językiem tabel, kolumn, kluczy obcych i wierszy. To dwa różne światy.

> 💡 **Analogia — katalog fiszek i żywy człowiek**
>
> Wyobraź sobie bibliotekę, która prowadzi kartotekę czytelników. Każdy czytelnik ma **fiszkę**: nazwisko, numer, data zapisu. Fiszek nie da się „zapytać”, fiszka nie ma pamięci ani zachowania. Gdy chcesz wiedzieć, ile książek wypożyczył pan Nowak, musisz przejrzeć wszystkie inne fiszki.
>
> Teraz wyobraź sobie **człowieka** — pana Nowaka, który siedzi na krześle, pamięta, co wypożyczył, i może odpowiedzieć na pytanie. Możesz zmienić jego adres i każdy, kto na niego patrzy, widzi nowy adres.
>
> Wiersz w bazie to fiszka. Obiekt Pythona to człowiek. **Mapowanie obiektowo-relacyjne** (ang. *object-relational mapping*, ORM) to tłumacz, który utrzymuje między nimi zgodność: gdy zmieniasz coś w człowieku, tłumacz idzie i poprawia fiszkę; gdy czytasz fiszkę, tłumacz tworzy człowieka.

### 1.2 Cztery rodzaje rozjazdu

Ten rozjazd ma nazwę: **impedancja obiektowo-relacyjna** (ang. *object-relational impedance mismatch*). „Impedancja” to termin pożyczony z elektroniki — oznacza opór na styku dwóch różnych środowisk. Rozjazdów jest kilka i warto je znać po nazwie, bo każdy z nich SQLAlchemy rozwiązuje inaczej.

| Rodzaj rozjazdu | Świat relacyjny | Świat obiektowy |
|---|---|---|
| **Tożsamość** | Wiersz identyfikuje klucz główny; dwa odczyty tego samego wiersza to dwa „zestawy wartości” | Obiekt identyfikuje się przez `is`; dwa odczyty tego samego bytu powinny dać **ten sam obiekt** |
| **Cykl życia** | Wiersz istnieje od `INSERT` do `DELETE` | Obiekt istnieje od utworzenia w pamięci do usunięcia przez garbage collector |
| **Relacje** | Powiązania to klucze obce i joiny, wynikiem jest płaska tabela | Powiązania to referencje: `order.customer` zwraca **obiekt**, nie liczbę |
| **Dziedziczenie i kompozycja** | Brak dziedziczenia; najbliżej: dodatkowa tabela lub kolumna „typ” | Klasy dziedziczą, mają polimorfizm |

Do tego dochodzi różnica w typach: `decimal.Decimal` vs `NUMERIC`, `datetime` ze strefą i bez, `list` vs tabela potomna.

> 🧠 **Dlaczego tak jest** — bazy relacyjne powstały w latach 70. i projektowano je tak, żeby wydajnie **przeszukiwać zbiory**, a nie reprezentować pojedyncze byty. Model obiektowy projektowano tak, żeby **opisywać zachowanie**. Te dwa cele są różne i żadne z nich nie jest „lepsze” — dlatego potrzebny jest tłumacz, a nie zwycięzca.

### 1.3 Od czego uciekamy: ręczne mapowanie

Zobacz, ile pracy kosztuje ręczne zbudowanie obiektów z wierszy. To jest kod, który ORM pisze za ciebie:

```python
# examples/07_manual_mapping.py
"""Ręczne mapowanie wierszy na obiekty — po to istnieje ORM."""
from __future__ import annotations

import decimal
from dataclasses import dataclass, field

from sqlalchemy import create_engine, text

engine = create_engine("sqlite:///:memory:")

with engine.begin() as conn:
    conn.execute(text("CREATE TABLE customer (id INTEGER PRIMARY KEY, name TEXT)"))
    conn.execute(
        text(
            "CREATE TABLE product (id INTEGER PRIMARY KEY, name TEXT, price NUMERIC)"
        )
    )
    conn.execute(text("CREATE TABLE orders (id INTEGER PRIMARY KEY, customer_id INT)"))
    conn.execute(
        text(
            "CREATE TABLE order_item (id INTEGER PRIMARY KEY, order_id INT, "
            "product_id INT, quantity INT)"
        )
    )
    conn.execute(text("INSERT INTO customer (id, name) VALUES (1, 'Anna')"))
    conn.execute(
        text("INSERT INTO product (id, name, price) VALUES (10, 'Kubek', 24.99)")
    )
    conn.execute(text("INSERT INTO orders (id, customer_id) VALUES (100, 1)"))
    conn.execute(
        text(
            "INSERT INTO order_item (id, order_id, product_id, quantity) "
            "VALUES (1000, 100, 10, 3)"
        )
    )


@dataclass
class Product:
    id: int
    name: str
    price: decimal.Decimal


@dataclass
class OrderItem:
    id: int
    product: Product
    quantity: int

    @property
    def subtotal(self) -> decimal.Decimal:
        return self.product.price * self.quantity


@dataclass
class Order:
    id: int
    items: list[OrderItem] = field(default_factory=list)


with engine.connect() as conn:
    rows = conn.execute(
        text(
            """
            SELECT o.id AS order_id, oi.id AS item_id, oi.quantity,
                   p.id AS product_id, p.name AS product_name, p.price
            FROM orders o
            JOIN order_item oi ON oi.order_id = o.id
            JOIN product p ON p.id = oi.product_id
            WHERE o.customer_id = :cid
            """
        ),
        {"cid": 1},
    ).all()

    # ...i teraz trzeba ręcznie "poskładać" płaskie wiersze w graf obiektów:
    orders: dict[int, Order] = {}
    for row in rows:
        order = orders.get(row.order_id)
        if order is None:
            order = Order(id=row.order_id)
            orders[row.order_id] = order
        order.items.append(
            OrderItem(
                id=row.item_id,
                product=Product(
                    id=row.product_id, name=row.product_name, price=row.price
                ),
                quantity=row.quantity,
            )
        )

for order in orders.values():
    print(order.id, sum(i.subtotal for i in order.items))
```

Zwróć uwagę na trzy rzeczy:

1. Ręcznie odtwarzasz relację („ten sam `order_id` → ten sam obiekt zamówienia”).
2. Ręcznie konwertujesz typy.
3. Gdyby zapytanie zmieniło kształt (np. dodano `LEFT JOIN`), cała pętla składająca mogłaby się zepsuć.

ORM robi dokładnie to — tylko raz, poprawnie i dla każdego modelu w projekcie.

> ⚠️ **Pułapka — „ORM zastępuje SQL”**
>
> To najczęstsze nieporozumienie. ORM **nie zastępuje** SQL-a — ORM **generuje** SQL. W tym kursie przy każdym zapytaniu pokazujemy wyemitowany SQL (`🔬 Pod maską`), bo bez umiejętności czytania SQL-a nie da się zdiagnozować problemów wydajnościowych (moduł 11 i 17). Osoba, która nie zna SQL-a, a używa ORM-a, jest jak kierowca, który nie wie, co to hamulec — jedzie, dopóki nie trzeba zatrzymać.

---

## 2. `DeclarativeBase` — od klasy Pythona do tabeli

### 2.1 Pierwszy model

Zaczniemy od absolutnego minimum. Ten plik jest kompletny i uruchamialny.

```python
# examples/07_first_model.py
"""Najprostszy model deklaratywny w SQLAlchemy 2.0."""
from __future__ import annotations

from sqlalchemy import create_engine
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    """Klasa bazowa dla wszystkich modeli w projekcie."""


class Customer(Base):
    __tablename__ = "customer"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]


engine = create_engine("sqlite:///:memory:", echo=False)
Base.metadata.create_all(engine)

print(type(Customer.__table__))
print(Customer.__table__.columns.keys())
```

Wynik:

```text
<class 'sqlalchemy.sql.schema.Table'>
['id', 'name']
```

Trzy rzeczy, które właśnie się stały:

- Klasa `Customer` **nie jest** zwykłą klasą Pythona — SQLAlchemy dopisała jej atrybuty `__table__`, `__mapper__`, `__mapper_args__` i podmieniła atrybuty `id`/`name` na tzw. **atrybuty instrumentowane** (ang. *instrumented attributes*).
- `Base.metadata` to zwykła `MetaData` z modułu 03 — ta sama, którą znasz z Core. Zawiera teraz obiekt `Table` wygenerowany z klasy.
- `Base.metadata.create_all(engine)` działa dokładnie tak, jak w module 03, bo operuje na `MetaData`.

> 🔬 **Pod maską — DDL wygenerowany z klasy**

```python
# examples/07_first_model_ddl.py
from sqlalchemy import create_engine
from sqlalchemy.schema import CreateTable

from examples.first_model import Customer  # klasa z poprzedniego przykładu

engine = create_engine("sqlite:///:memory:")
print(CreateTable(Customer.__table__).compile(engine))
```

```sql
CREATE TABLE customer (
	id INTEGER NOT NULL, 
	name VARCHAR NOT NULL, 
	PRIMARY KEY (id)
)
```

Zwróć uwagę na `NOT NULL` przy `name`. W SQL-u kolumna jest domyślnie **nullable** (może być pusta), a tutaj jest odwrotnie — o tym w punkcie 3.2.

### 2.2 Co dzieje się przy imporcie

Proces „deklaratywny” to nie magia, to zwykłe mechanizmy Pythona. Kolejność jest taka:

```text
1. Definiujesz `class Base(DeclarativeBase)`.
      → DeclarativeBase tworzy obiekt `registry` (rejestr mapperów).
      → registry zakłada (lub dostaje) obiekt `MetaData`.

2. Definiujesz `class Customer(Base)`.
      → Python wywołuje `Base.__init_subclass__` — to hook Pythona,
        który odpala się w momencie DEFINICJI klasy, nie jej utworzenia.
      → Declarative skanuje `Customer.__annotations__` i `Customer.__dict__`,
        szukając pól `Mapped[...]` oraz `mapped_column()`.

3. Powstaje obiekt `Table("customer", metadata, ...)` — taki sam jak w module 03.

4. Powstaje obiekt `Mapper` — to on pamięta, że atrybut `Customer.name`
   odpowiada kolumnie `customer.name`.

5. SQLAlchemy podmienia `Customer.id` i `Customer.name` na deskryptory
   (atrybuty instrumentowane), które obsługują odczyt, zapis i śledzenie zmian.
```

Punkt 5 jest kluczowy dla zrozumienia ORM-a: `customer.name` to nie jest zwykłe pole obiektu. Gdy piszesz `customer.name = "Anna"`, nie modyfikujesz słownika `__dict__` w sposób, o którym Python sam by wiedział — robi to deskryptor, który **rejestruje fakt zmiany** dla mechanizmu Unit of Work (moduł 08).

> 💡 **Analogia — klasa bazowa jako wzór formularza**
>
> `DeclarativeBase` to nie jest „baza danych”, tylko **wzór formularza urzędowego**. Każdy model to jeden formularz wypełniony według tego wzoru. `registry` to sekretariat, który prowadzi rejestr wszystkich formularzy, a `MetaData` to segregator, w którym trzymane są gotowe opisy tabel.

> 🧠 **Dlaczego tak jest** — `__init_subclass__` to standardowy hook Pythona (PEP 487), dostępny od wersji 3.6. SQLAlchemy nie potrzebuje metaklasy z zaklęciami — wystarczy jej fakt, że Python informuje klasę bazową o powstaniu podklasy. Dzięki temu modele są „normalnymi” klasami: można je dziedziczyć, opisywać typami i testować.

### 2.3 Dlaczego klasa bazowa, a nie dekorator

W SQLAlchemy 1.x istniały dwie drogi: `declarative_base()` (funkcja zwracająca klasę bazową) i dekorator `@mapper_registry.mapped`. W 2.0 preferowaną drogą jest dziedziczenie po `DeclarativeBase`, ponieważ:

- **Działa z typowaniem statycznym.** `class Base(DeclarativeBase)` jest zrozumiałe dla `mypy` i dla edytora. `declarative_base()` zwraca `Any`, więc tracisz podpowiedzi.
- **Pozwala konfigurować globalnie.** Na klasie bazowej ustawiasz `metadata`, `type_annotation_map` i `__allow_unmapped__` raz dla całego projektu.
- **Jest jawna.** Widzisz w kodzie, że klasa jest modelem, bo dziedziczy po `Base`.

> ⚠️ **Pułapka — `declarative_base()` w starych poradnikach**
>
> W internecie wciąż pełno jest przykładów zaczynających się od `Base = declarative_base()`. To API z 1.x. Działa w 2.0, ale jest oznaczone jako przestarzałe (ang. *legacy*) i nie daje typowania. Jeżeli kopiujesz przykład i widzisz `declarative_base()`, `Column(...)` zamiast `mapped_column(...)` albo `session.query(...)` — trafiłeś na materiał z 1.x i połowa rzeczy w nim może się nie zgadzać z tym kursem.

---

## 3. `Mapped[...]` i `mapped_column()`

### 3.1 Typ wywnioskowany z adnotacji

Porównaj dwa zapisy. Oba są poprawne w 2.0:

```python
# examples/07_two_styles.py
from sqlalchemy import Integer, String
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class ProductA(Base):
    """Styl „klasyczny” — typ podany w mapped_column()."""

    __tablename__ = "product_a"

    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(120), nullable=False)
    price = mapped_column(Numeric(10, 2), nullable=False)


class ProductB(Base):
    """Styl 2.0 — typ wynika z adnotacji Mapped[...]."""

    __tablename__ = "product_b"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(120))
    price: Mapped[decimal.Decimal] = mapped_column(Numeric(10, 2))
```

Zalety stylu 2.0:

| Aspekt | `mapped_column(Integer, ...)` | `Mapped[int] = mapped_column(...)` |
|---|---|---|
| Podpowiedzi w edytorze | brak — atrybut jest `Any` | pełne — `product.id` to `int` |
| `mypy` | nie sprawdzi nic | sprawdzi przypisania i odczyty |
| Nullability | trzeba powtórzyć `nullable=` | wynika z adnotacji |
| Ryzyko literówki | duże (dwa razy ten sam typ) | małe (jeden typ) |

Adnotacja `Mapped[T]` znaczy dosłownie: „ten atrybut jest mapowany na kolumnę, a jego typ w Pythonie to `T`”. SQLAlchemy sprawdza, czy `T` da się znaleźć w tablicy typów. Domyślna tablica wygląda tak:

```python
# Domyślne mapowanie typów Pythona na typy SQLAlchemy (uproszczone)
type_map = {
    bool:               Boolean(),
    bytes:              LargeBinary(),
    datetime.date:      Date(),
    datetime.datetime:  DateTime(),
    datetime.time:      Time(),
    datetime.timedelta: Interval(),
    decimal.Decimal:    Numeric(),
    float:              Float(),
    int:                Integer(),
    str:                String(),
    uuid.UUID:          Uuid(),
}
```

Zwróć uwagę na dwie rzeczy:

- **`dict` i `list` nie ma na tej liście.** Jeżeli napiszesz `Mapped[dict[str, Any]]`, SQLAlchemy **nie** zgadnie, że chodzi o JSON — zgłosi błąd. Musisz dopisać to mapowanie samodzielnie (punkt 4.3).
- **`str` → `String()` bez długości.** Na PostgreSQL to `VARCHAR` bez limitu, na SQLite `VARCHAR`. Jeżeli chcesz `VARCHAR(120)`, musisz podać długość jawnie: `mapped_column(String(120))`.

> 🧠 **Dlaczego tak jest** — SQLAlchemy nie wie, czy Twoje `str` to imię (60 znaków), opis (bez limitu) czy kod pocztowy (6 znaków). Zgadywanie byłoby wygodne, ale niebezpieczne: zmiana limitu długości w kodzie to zmiana schematu bazy. Dlatego domyślnie wybiera typ „bezpieczny” (bez długości), a precyzję zostawia Tobie.

### 3.2 Nullability — `Mapped[str]` kontra `Mapped[str | None]`

To najważniejsza reguła tego rozdziału:

```python
# examples/07_nullability.py
from __future__ import annotations

from sqlalchemy import create_engine
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy.schema import CreateTable


class Base(DeclarativeBase):
    pass


class Customer(Base):
    __tablename__ = "customer"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str]                       # NOT NULL
    nickname: Mapped[str | None]             # NULL dopuszczalne
    notes: Mapped[str | None] = mapped_column(nullable=True)  # to samo, jawnie


engine = create_engine("sqlite:///:memory:")
print(CreateTable(Customer.__table__).compile(engine))
```

```sql
CREATE TABLE customer (
	id INTEGER NOT NULL, 
	email VARCHAR NOT NULL, 
	nickname VARCHAR, 
	notes VARCHAR, 
	PRIMARY KEY (id)
)
```

**Adnotacja mówi o nullability, nie SQL.** W SQL-u domyślnie kolumna może być pusta (`NULL`). W deklaratywnym SQLAlchemy jest odwrotnie:

| Adnotacja | Znaczenie dla Pythona | Znaczenie dla bazy |
|---|---|---|
| `Mapped[str]` | zawsze `str` | `NOT NULL` |
| `Mapped[str \| None]` | `str` albo `None` | dopuszcza `NULL` |

> ⚠️ **Pułapka — `| None` dopisane „dla spokoju `mypy`”**
>
> To najczęstszy cichy błąd w projektach z SQLAlchemy. Scenariusz: `mypy` marudzi, że nie da się przypisać `None`, więc programista dopisuje `| None` do adnotacji — i **przy okazji zmienia schemat bazy**. Potem w migracji (moduł 16) pojawia się `ALTER TABLE ... DROP NOT NULL`, którego nikt się nie spodziewał, a dane, które miały być zawsze wypełnione, zaczynają być puste.
>
> Reguła: `| None` w adnotacji to **decyzja projektowa o schemacie**, a nie kosmetyka typowania. Jeżeli atrybut ma być wymagany, a `mypy` marudzi, popraw logikę, a nie adnotację.

Jeżeli podasz `nullable` jawnie, wygrywa wartość jawna. To bywa przydatne (np. kolumna `Mapped[str | None]`, którą chcesz mieć `NOT NULL` z powodu nałożonego `CHECK`), ale jest też źródłem rozjazdu między tym, co mówi kod, a tym, co mówi baza. W praktyce: **nie mieszaj obu mechanizmów w jednym modelu.**

### 3.3 Domyślne wartości: `default`, `insert_default`, `server_default`, `onupdate`

Cztery różne parametry, cztery różne momenty działania. To najbardziej myląca część `mapped_column()`.

```python
# examples/07_defaults.py
from __future__ import annotations

import datetime

from sqlalchemy import DateTime, func, text
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class Article(Base):
    __tablename__ = "article"

    id: Mapped[int] = mapped_column(primary_key=True)

    # 1. default: wartość wstawiana przez PYTHON, gdy atrybutu nie ustawiono
    status: Mapped[str] = mapped_column(default="draft")

    # 2. insert_default: jak default, ale NIE jest wartością domyślną
    #    w konstruktorze (istotne przy MappedAsDataclass — punkt 7)
    views: Mapped[int] = mapped_column(insert_default=0)

    # 3. server_default: wartość wstawiana przez BAZĘ, gdy kolumny
    #    nie ma w INSERT-cie
    created_at: Mapped[datetime.datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now()
    )

    # 4. onupdate: wartość wstawiana przez PYTHON przy każdym UPDATE
    updated_at: Mapped[datetime.datetime | None] = mapped_column(
        DateTime(timezone=True), onupdate=func.now()
    )

    # 5. server_onupdate: informacja dla SQLAlchemy, że baza sama aktualizuje
    #    kolumnę (np. przez trigger) — SQLAlchemy NIE generuje tej wartości
    revision: Mapped[int] = mapped_column(server_default=text("1"))
```

Kiedy który:

| Parametr | Kto wstawia | Kiedy | Czy widoczne w `__init__` (dataclass) |
|---|---|---|---|
| `default` | Python (SQLAlchemy) | przy `INSERT`, jeśli wartość nieustawiona | tak |
| `insert_default` | Python (SQLAlchemy) | przy `INSERT`, jeśli wartość nieustawiona | nie |
| `server_default` | baza | przy `INSERT`, jeśli kolumny nie ma w `INSERT` | nie |
| `onupdate` | Python (SQLAlchemy) | przy każdym `UPDATE` | nie |
| `server_onupdate` | baza (trigger) | zależy od bazy | nie |

> 🔬 **Pod maską — `default` kontra `server_default`**
>
> `default="draft"` nie pojawia się w SQL-u — SQLAlchemy wysyła wartość jako **parametr**:
>
> ```sql
> INSERT INTO article (status, views, revision) VALUES (?, ?, ?)
> -- parametry: ('draft', 0, ...)
> ```
>
> `server_default=func.now()` trafia do **DDL** i przy wstawianiu kolumny może w ogóle nie być w `INSERT`:
>
> ```sql
> -- fragment CREATE TABLE:
> created_at DATETIME DEFAULT (CURRENT_TIMESTAMP) NOT NULL
> ```
>
> Konsekwencja praktyczna: `default` jest liczony na maszynie aplikacji, `server_default` na maszynie bazy. Jeżeli wstawiasz dane z innego narzędzia (np. `psql`, skrypt importujący, inna aplikacja), **tylko `server_default` zadziała**. Dlatego kolumny techniczne (`created_at`, `status` domyślny) w dobrze zaprojektowanym schemacie mają `server_default`, a `default` zostaje tam, gdzie wartość musi powstać w Pythonie.

> ⚠️ **Pułapka — `server_default` musi być wyrażeniem SQL**
>
> `server_default="now()"` **nie zadziała** w sposób, jakiego oczekujesz: zostanie potraktowane jako stały napis `'now()'`, a nie wywołanie funkcji. Poprawnie: `server_default=func.now()` albo `server_default=text("now()")`. Wyjątkiem są proste literały, gdzie baza i tak dostanie napis — `server_default="draft"` jest w porządku.

### 3.4 `Mapped[list["X"]]` — zapowiedź relacji

W modelu mogą pojawić się atrybuty, które nie są kolumnami, lecz **relacjami**:

```python
# examples/07_relationship_preview.py
from __future__ import annotations

from sqlalchemy import ForeignKey
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    pass


class Author(Base):
    __tablename__ = "author"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]

    books: Mapped[list["Book"]] = relationship(back_populates="author")


class Book(Base):
    __tablename__ = "book"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str]
    author_id: Mapped[int] = mapped_column(ForeignKey("author.id"))

    author: Mapped["Author"] = relationship(back_populates="books")
```

Trzy rzeczy do zapamiętania na teraz (pełne omówienie w module 09):

- `author_id: Mapped[int] = mapped_column(ForeignKey("author.id"))` — to **kolumna**; trafia do tabeli.
- `author: Mapped["Author"] = relationship(...)` — to **nie kolumna**; nie trafia do DDL, jest tylko sposobem poruszania się po grafie obiektów.
- `Mapped[list["Book"]]` — cudzysłowy są konieczne, bo klasa `Book` jeszcze nie istnieje w momencie definiowania `Author`. Python zapisuje adnotację jako napis i rozwiązuje ją później.

> 🧠 **Dlaczego tak jest** — SQLAlchemy musi odróżnić „kolumnę” od „relacji”. Rozróżnia po tym, czy po prawej stronie stoi `mapped_column()`, czy `relationship()`. Gdybyś napisał `author: Mapped["Author"]` bez prawej strony, SQLAlchemy próbowałoby utworzyć kolumnę typu… nieznanego — i zgłosiłoby błąd. Adnotacja mówi *co*, prawa strona mówi *jak*.

> 🧪 **Ćwiczenie** — dopisz do `Article` z punktu 3.3 kolumnę `slug: Mapped[str]`, która ma być unikalna i indeksowana, ale **nie** ma być przekazywana w konstruktorze obiektu. (Podpowiedź: `mapped_column(..., init=False)` — szczegóły w punkcie 7.3.)

---

## 4. `Annotated` i `type_annotation_map` — koniec z powtarzaniem

### 4.1 `Annotated[str, 50]`

Wyobraź sobie projekt, w którym 40 kolumn to `String(255)` — e-maile, nazwy, tytuły. Pisanie `mapped_column(String(255))` czterdzieści razy to nie tylko nuda, to ryzyko: raz napiszesz `String(255)`, raz `String(250)`, a migracja wygeneruje `ALTER TYPE` bez powodu.

Python ma na to narzędzie: `typing.Annotated` (PEP 593). To opakowanie, które dokleja do typu dodatkowe metadane:

```python
from typing import Annotated

str255 = Annotated[str, 255]   # "str, ale z metadanymi: 255"
```

SQLAlchemy rozumie ten zapis: `Annotated[str, 255]` znaczy „użyj typu `String` z długością 255”.

```python
# examples/07_annotated_basic.py
from __future__ import annotations

import decimal
from typing import Annotated

from sqlalchemy import create_engine, Numeric, String
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy.schema import CreateTable

str120 = Annotated[str, 120]
money = Annotated[decimal.Decimal, 12]  # tylko metadana — patrz niżej!


class Base(DeclarativeBase):
    pass


class Product(Base):
    __tablename__ = "product"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str120]                      # → String(120)
    price: Mapped[money] = mapped_column(Numeric(12, 2))  # jawnie, bo metadana to 12
```

Uwaga: `Annotated[str, 120]` to skrót obsługiwany wprost. `Annotated[Decimal, 12]` **nie** ma takiego skrótu — metadana `12` nie mówi SQLAlchemy, że chodzi o `Numeric(12, 2)`. Aby uzyskać taki efekt, trzeba użyć drugiej, mocniejszej formy.

### 4.2 Całe „kształty kolumn” w `Annotated`

`Annotated` może przenosić nie tylko liczbę, ale cały obiekt `mapped_column()`:

```python
# examples/07_annotated_shapes.py
from __future__ import annotations

import datetime
from typing import Annotated

from sqlalchemy import DateTime, ForeignKey, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship

# Gotowe "kształty" kolumn, definiowane raz i używane w całym projekcie.
intpk = Annotated[int, mapped_column(primary_key=True)]
category_fk = Annotated[int, mapped_column(ForeignKey("category.id"))]
created_at = Annotated[
    datetime.datetime,
    mapped_column(DateTime(timezone=True), server_default=func.now()),
]
required_name = Annotated[str, mapped_column(String(120), nullable=False)]


class Base(DeclarativeBase):
    pass


class Category(Base):
    __tablename__ = "category"

    id: Mapped[intpk]
    name: Mapped[required_name]
    created_at: Mapped[created_at] = mapped_column(init=False)
```

Ten mechanizm bywa nazywany **deklaratywnymi mixinami zorientowanymi na typ** — zastępuje większość zastosowań `@declared_attr` (punkt 8).

> 💡 **Analogia — pieczątki w urzędzie**
>
> Zamiast pisać za każdym razem odręcznie „niniejszym zaświadcza się, że…” — masz pieczątkę. `Annotated[int, mapped_column(primary_key=True)]` to pieczątka „KLUCZ GŁÓWNY”. Przyłożenie jej do dowolnej kolumny daje spójny efekt, a gdy trzeba zmienić treść pieczątki, zmieniasz ją w jednym miejscu.

### 4.3 `type_annotation_map`

Pieczątki działają na pojedynczych adnotacjach. `type_annotation_map` działa **globalnie** — mówi, jak tłumaczyć typy Pythona na typy SQL w całym projekcie:

```python
# examples/07_type_map.py
from __future__ import annotations

import datetime
import decimal
import uuid
from typing import Annotated, Any

from sqlalchemy import JSON, BigInteger, DateTime, MetaData, Numeric, String, Uuid
from sqlalchemy.dialects import postgresql
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

# Warianty str o różnych długościach — jako osobne "typy"
str60 = Annotated[str, 60]
str255 = Annotated[str, 255]

NAMING_CONVENTION = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}


class Base(DeclarativeBase):
    metadata = MetaData(naming_convention=NAMING_CONVENTION)

    type_annotation_map = {
        # warianty str
        str60: String(60),
        str255: String(255),
        # nadpisanie wbudowanego mapowania
        decimal.Decimal: Numeric(12, 2),
        int: BigInteger(),
        datetime.datetime: DateTime(timezone=True),
        uuid.UUID: Uuid(as_uuid=True),
        # JSON — trzeba dodać ręcznie!
        dict[str, Any]: JSON,
        # wariant dla PostgreSQL
        str: String().with_variant(postgresql.CITEXT(), "postgresql"),
    }


class Customer(Base):
    __tablename__ = "customer"

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    email: Mapped[str255]                 # → VARCHAR(255)
    full_name: Mapped[str60]              # → VARCHAR(60)
    preferences: Mapped[dict[str, Any]]   # → JSON
    balance: Mapped[decimal.Decimal]      # → NUMERIC(12, 2)
    created_at: Mapped[datetime.datetime] # → DATETIME WITH TIME ZONE (PG)
```

Trzy praktyczne zasady:

1. **Nadpisuj wbudowane mapowanie świadomie.** `int: BigInteger()` znaczy „każde `Mapped[int]` w projekcie to `BIGINT`”. To wygodne, ale ma konsekwencje: klucze główne urosną do 8 bajtów. Rób to, gdy wiesz, dlaczego.
2. **`dict` i `list` trzeba dodać samodzielnie.** Bez tego `Mapped[dict[str, Any]]` zgłosi `ArgumentError`. Świadomie: nie każdy słownik ma trafić do bazy jako JSON.
3. **`with_variant` pozwala pisać kod przenośny.** `String().with_variant(postgresql.CITEXT(), "postgresql")` znaczy: „na PostgreSQL użyj `CITEXT` (porównywanie bez rozróżniania wielkości liter), na wszystkim innym `VARCHAR`”.

> 🆕 **SQLAlchemy 2.1** — w 2.1 rozszerzono obsługę typów w `type_annotation_map` o typy opcjonalne (np. `Mapped[dict[str, Any] | None]`), dzięki czemu nie trzeba już deklarować osobnego wpisu dla wariantu z `None`. W 2.0 dla takiego przypadku trzeba było wpisać obie wersje osobno.

> 🧪 **Ćwiczenie** — zdefiniuj `slug = Annotated[str, mapped_column(String(80), unique=True, index=True)]` i użyj go w dwóch różnych modelach. Sprawdź w DDL, czy w obu powstał indeks unikalny.

---

## 5. `__tablename__`, `__table_args__` i praca z gotową `Table`

### 5.1 `__tablename__`

`__tablename__` to nazwa tabeli w bazie. Dwie uwagi:

- **Nie ma wartości domyślnej.** Jeżeli jej nie podasz, SQLAlchemy zgłosi `ArgumentError` w momencie definicji klasy. To celowe: zgadywanie nazwy tabeli z nazwy klasy prowadzi do niespodzianek (`CustomerOrder` → `customerorder`? `customer_order`?).
- **Nazwa klasy i nazwa tabeli to dwie różne rzeczy.** Klasa `Order` może mapować się na tabelę `orders`, klasa `User` na `app_user` (bo `user` to słowo zastrzeżone w PostgreSQL). Możesz też całkowicie rozdzielić nazwę atrybutu i nazwę kolumny: `mapped_column("full_name")` z atrybutem `fullname`.

```python
# examples/07_tablename.py
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class User(Base):
    __tablename__ = "app_user"  # inna nazwa niż klasa — i to jest OK

    id: Mapped[int] = mapped_column(primary_key=True)
    # atrybut w Pythonie: fullname, kolumna w bazie: full_name
    fullname: Mapped[str] = mapped_column("full_name")
```

### 5.2 `__table_args__`

`__table_args__` to krotka (ang. *tuple*) zawierająca wszystko, co nie jest kolumną, a dotyczy tabeli: ograniczenia, indeksy i opcje tabeli.

```python
# examples/07_table_args.py
from __future__ import annotations

import decimal

from sqlalchemy import (
    CheckConstraint,
    ForeignKey,
    Index,
    Numeric,
    String,
    UniqueConstraint,
)
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class Product(Base):
    __tablename__ = "product"

    id: Mapped[int] = mapped_column(primary_key=True)
    sku: Mapped[str] = mapped_column(String(32))
    name: Mapped[str] = mapped_column(String(120))
    price: Mapped[decimal.Decimal] = mapped_column(Numeric(12, 2))
    category_id: Mapped[int] = mapped_column(ForeignKey("category.id"))
    is_active: Mapped[bool] = mapped_column(default=True)

    __table_args__ = (
        # 1. ograniczenia — jak w module 03
        UniqueConstraint("sku", name="uq_product_sku"),
        CheckConstraint("price >= 0", name="ck_product_price_non_negative"),
        # 2. indeks złożony — kolejność kolumn ma znaczenie!
        Index("ix_product_category_active", "category_id", "is_active"),
        # 3. OSTATNI element może być słownikiem z opcjami tabeli
        {"schema": None, "comment": "Katalog produktów"},
    )
```

Reguły, o których łatwo zapomnieć:

- Ostatni element krotki, jeżeli jest słownikiem, traktowany jest jako **argumenty słownikowe** dla `Table(...)`. Krotka może zawierać tylko jeden taki słownik i tylko na końcu.
- Wewnątrz `__table_args__` kolumny podajesz **po nazwie jako napisy** (`"category_id"`), a nie jako atrybuty (`category_id`) — w momencie definiowania klasy obiektu kolumny jeszcze nie ma. Gdybyś chciał odwołać się do atrybutu, potrzebujesz `@declared_attr` (punkt 8.4).

> 🔬 **Pod maską — indeks z `Index` w `__table_args__`**
>
> ```sql
> CREATE INDEX ix_product_category_active ON product (category_id, is_active)
> ```
>
> Indeks złożony `(category_id, is_active)` przyspieszy zapytanie `WHERE category_id = ? AND is_active = ?` oraz `WHERE category_id = ?`, ale **nie** przyspieszy `WHERE is_active = ?`. To ta sama zasada, co w module 03: indeks działa jak spis treści — przydaje się, gdy szukasz po pierwszej kolumnie.

### 5.3 `__table__` — gdy tabela już istnieje

Czasem masz `Table` zbudowane w Core (np. z modułu 03, albo z refleksji nad istniejącą bazą) i chcesz tylko „doczepić” do niej klasę. Służy do tego `__table__`:

```python
# examples/07_table_attribute.py
from __future__ import annotations

from sqlalchemy import Column, Integer, MetaData, String, Table, create_engine
from sqlalchemy.orm import DeclarativeBase, Mapped


class Base(DeclarativeBase):
    pass


# 1. Tabela zdefiniowana po staremu — w Core.
legacy_customer = Table(
    "legacy_customer",
    Base.metadata,
    Column("id", Integer, primary_key=True),
    Column("full_name", String(120), nullable=False),
)


# 2. Klasa, która NIE definiuje kolumn — bierze je z gotowej Table.
class LegacyCustomer(Base):
    __table__ = legacy_customer

    # 3. Adnotacje są opcjonalne, ale poprawiają typowanie
    id: Mapped[int]
    full_name: Mapped[str]

    def greet(self) -> str:
        return f"Cześć, {self.full_name}!"


engine = create_engine("sqlite:///:memory:")
Base.metadata.create_all(engine)
print(sorted(LegacyCustomer.__table__.columns.keys()))
```

```text
['full_name', 'id']
```

Kiedy to się przydaje:

| Sytuacja | Rozwiązanie |
|---|---|
| Istniejąca, duża baza, której nie chcesz opisywać od zera | refleksja (`autoload_with=engine`) + `__table__` |
| Tabela zdefiniowana w innym module w Core | `__table__ = ta_tabela` |
| Potrzebujesz tej samej tabeli w dwóch mapperach | `__table__` + `registry` |
| Nowy projekt | pisz modele deklaratywne, nie `Table` |

> ⚠️ **Pułapka — mieszanie `__table__` i kolumn w klasie**
>
> Jeżeli użyjesz `__table__`, **nie możesz** jednocześnie deklarować kolumn przez `mapped_column()` — SQLAlchemy zgłosi `ArgumentError: Can't add additional column ... when using __table__`. Możesz natomiast dodawać relacje, metody i właściwości. To rozróżnienie jest zdrowe: kolumny pochodzą z tabeli, zachowanie z klasy.

### 5.4 Naming convention razem z `DeclarativeBase`

W module 03 poznałeś `naming_convention`. Z deklaratywnym SQLAlchemy działa tak samo — i **jest jeszcze ważniejsze**, bo bez niego `alembic autogenerate` (moduł 16) nie potrafi rozpoznać ograniczeń i generuje migracje typu „usuń i dodaj od nowa”.

```python
# examples/07_base_with_conventions.py
from sqlalchemy import MetaData
from sqlalchemy.orm import DeclarativeBase

NAMING_CONVENTION = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}


class Base(DeclarativeBase):
    metadata = MetaData(naming_convention=NAMING_CONVENTION)
```

Efekt: `UniqueConstraint("sku")` bez `name=` i tak dostanie nazwę `uq_product_sku`, a `ForeignKey("category.id")` — `fk_product_category_id_category`. To nie kosmetyka: nazwy ograniczeń są kontraktem między kodem a bazą.

> 🧠 **Dlaczego tak jest** — PostgreSQL i SQLite nazywają ograniczenia automatycznie, ale **inaczej** i nieprzewidywalnie. Bez konwencji ta sama migracja uruchomiona na dwóch maszynach może próbować usunąć ograniczenia o różnych nazwach. Konwencja ustawiona raz na `MetaData` rozwiązuje to na zawsze.

---

## 6. Modyfikatory `mapped_column()` — pełny przegląd

Poniżej pełna sygnatura z podziałem na grupy. Nie musisz jej znać na pamięć — wróć tu, gdy będziesz szukać konkretnego parametru.

```python
def mapped_column(
    __name_pos=None, __type_pos=None, /, *args,
    # --- kolumna (przekazywane do Column) ---
    name=None, type_=None, autoincrement="auto", nullable=..., primary_key=False,
    unique=None, index=None, default=..., insert_default=..., onupdate=None,
    server_default=None, server_onupdate=None, comment=None, quote=None,
    system=False, doc=None, key=None, info=None, active_history=False,
    # --- ORM ---
    deferred=..., deferred_group=None, deferred_raiseload=None,
    use_existing_column=False,
    # --- dataclass (punkt 7) ---
    init=..., repr=..., default_factory=..., compare=..., kw_only=..., hash=...,
    # --- inne ---
    sort_order=..., dataclass_metadata=..., **kw,
) -> MappedColumn[Any]: ...
```

### 6.1 Kolumny odroczone: `deferred`

`deferred=True` znaczy: „nie pobieraj tej kolumny domyślnie; dociągnij ją dopiero, gdy ktoś sięgnie po atrybut”.

```python
# examples/07_deferred.py
from __future__ import annotations

from sqlalchemy import Text
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class Article(Base):
    __tablename__ = "article"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str]
    # Wielka kolumna z treścią — pobierana tylko na żądanie.
    body: Mapped[str] = mapped_column(Text, deferred=True)
```

> 🔬 **Pod maską — zapytanie bez `body`**
>
> ```sql
> SELECT article.id, article.title FROM article WHERE article.id = ?
> ```
>
> Gdy po raz pierwszy odczytasz `article.body`, poleci drugie zapytanie:
>
> ```sql
> SELECT article.body FROM article WHERE article.id = ?
> ```

Kiedy używać: kolumny rzadko potrzebne (treść artykułu, blob, długi JSON). Kiedy **nie** używać: gdy zawsze i tak ich potrzebujesz — wtedy generujesz dodatkowe zapytania (odmiana problemu N+1, moduł 11).

### 6.2 `sort_order`

`sort_order` (dodane w 2.0.4) steruje kolejnością kolumn w generowanym `Table`. Przydaje się w mixinach: chcesz, żeby `id` było pierwsze, a kolumny audytowe na końcu — niezależnie od tego, jak ułożysz kod.

```python
# examples/07_sort_order.py
from __future__ import annotations

import datetime

from sqlalchemy import DateTime, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class IdMixin:
    id: Mapped[int] = mapped_column(primary_key=True, sort_order=-100)


class TimestampMixin:
    created_at: Mapped[datetime.datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), sort_order=100
    )


class Book(IdMixin, TimestampMixin, Base):
    __tablename__ = "book"

    title: Mapped[str]
```

```sql
CREATE TABLE book (
	id INTEGER NOT NULL, 
	title VARCHAR NOT NULL, 
	created_at DATETIME DEFAULT (CURRENT_TIMESTAMP) NOT NULL, 
	PRIMARY KEY (id)
)
```

> ⚠️ **Pułapka — `sort_order` a `MappedAsDataclass`**
>
> `sort_order` zmienia kolejność **kolumn w tabeli**, a nie kolejność argumentów w `__init__`. Jeżeli używasz `MappedAsDataclass` i mixinu z polem mającym domyślną wartość, Python zgłosi `TypeError: non-default argument follows default argument`, bo pola z mixinu trafiają przed pola klasy. Rozwiązanie: `kw_only=True` (punkt 8.5).

### 6.3 Pozostałe parametry, o których warto wiedzieć

| Parametr | Do czego służy | Kiedy używać |
|---|---|---|
| `use_existing_column=True` | użyj kolumny już zdefiniowanej w klasie nadrzędnej, zamiast tworzyć nową | dwa mixiny deklarujące tę samą kolumnę; rzadkie |
| `active_history=True` | przy przypisaniu wartości zapamiętaj **starą** wartość | audyt zmian, `@validates` z porównaniem (moduł 13) |
| `info={...}` | dowolne metadane aplikacji, niewidoczne dla bazy | własne narzędzia, np. flagowanie kolumn wrażliwych |
| `key="..."` | nazwa atrybutu Pythona inna niż nazwa kolumny | mapowanie dziedziczonych tabel |
| `comment="..."` | komentarz kolumny (`COMMENT ON COLUMN` w PostgreSQL) | dokumentacja schematu na poziomie bazy |
| `doc="..."` | opis w dokumentacji wygenerowanej | biblioteki i frameworki |
| `deferred_group="..."` | grupa odroczeń, można je włączać razem | modele z kilkoma „ciężkimi” grupami kolumn |
| `deferred_raiseload=True` | dostęp do odroczonej kolumny rzuca wyjątek | wymuszenie jawnego ładowania w testach |
| `autoincrement="auto"` | domyślnie: `True` dla pojedynczego klucza całkowitoliczbowego | zmieniaj tylko przy kluczach złożonych |

> 🧠 **Dlaczego tak jest** — `mapped_column()` nie jest osobnym bytem: pod spodem buduje zwykłe `Column(...)` z modułu 03 i opakowuje je w `MappedColumn`, który dokłada informacje o adnotacji i o zachowaniu ORM. Dlatego wszystkie parametry z `Column()` działają tutaj identycznie — nie musisz uczyć się dwóch API.

---

## 7. `MappedAsDataclass` — modele z automatycznym `__init__`

### 7.1 Problem: brak konstruktora

Zwykły model deklaratywny **nie ma** generowanego `__init__`. To znaczy, że `Customer(name="Anna")` nie zadziała — chyba że sam napiszesz konstruktor.

```python
# examples/07_no_init.py
class Customer(Base):
    __tablename__ = "customer"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]


c = Customer(name="Anna")  # TypeError: Customer() takes no arguments
```

SQLAlchemy **celowo** tego nie robi, bo ORM nie wymaga konstruktora: możesz utworzyć obiekt i ustawiać atrybuty po kolei, a Unit of Work i tak wykryje zmiany.

### 7.2 Włączenie

`MappedAsDataclass` sprawia, że każda klasa dziedzicząca dostaje generowany `__init__`, `__repr__`, `__eq__` — dokładnie jak `@dataclass` z biblioteki standardowej.

```python
# examples/07_dataclass_models.py
from __future__ import annotations

import decimal

from sqlalchemy.orm import DeclarativeBase, Mapped, MappedAsDataclass, mapped_column


class Base(MappedAsDataclass, DeclarativeBase):
    """Każdy model dziedziczący po tej klasie jest dataclassą."""


class Product(Base):
    __tablename__ = "product"

    # init=False: klucz nadaje baza, nie konstruktor
    id: Mapped[int] = mapped_column(primary_key=True, init=False)

    name: Mapped[str]
    price: Mapped[decimal.Decimal] = mapped_column(default=decimal.Decimal("0.00"))
    description: Mapped[str | None] = mapped_column(default=None)


p = Product(name="Kubek", price=decimal.Decimal("24.99"))
print(p)
print(p.id, p.description)
```

```text
Product(name='Kubek', price=Decimal('24.99'), description=None)
None None
```

Trzy rzeczy warte zauważenia:

1. **`init=False` przy `id`.** Klucz główny nadaje baza (autoinkrementacja) — nie ma sensu wymagać go w konstruktorze. Bez `init=False` musiałbyś pisać `Product(id=0, name=...)`.
2. **`default=` zamiast zwykłego `= wartość`.** W `MappedAsDataclass` musisz użyć `mapped_column(default=...)`. Składnia znana z `@dataclass` (`price: Decimal = Decimal("0")`) **nie jest obsługiwana** — SQLAlchemy wymaga jawnego parametru.
3. **`__repr__` działa od razu.** Bez `MappedAsDataclass` musiałbyś go napisać sam. W praktyce to ogromna oszczędność przy debugowaniu.

### 7.3 Kolejność argumentów i `kw_only`

Zwykłe dataclasses mają regułę: pole bez wartości domyślnej nie może stać po polu z wartością domyślną. Ta sama reguła obowiązuje tutaj:

```python
# examples/07_dataclass_order.py
class Product(Base):
    __tablename__ = "product"

    id: Mapped[int] = mapped_column(primary_key=True, init=False)
    name: Mapped[str]                                    # brak domyślnej — OK, jest pierwsze
    price: Mapped[decimal.Decimal] = mapped_column(default=decimal.Decimal("0"))
    description: Mapped[str | None] = mapped_column(default=None)   # z domyślną — OK, jest dalej
```

Gdybyś zamienił `name` i `price`, Python zgłosi `TypeError`. Rozwiązania:

| Rozwiązanie | Jak |
|---|---|
| Ustawić `kw_only=True` na klasie bazowej | `class Base(MappedAsDataclass, DeclarativeBase, kw_only=True)` — wszystkie pola stają się keyword-only, kolejność przestaje mieć znaczenie |
| Ustawić `kw_only=True` na konkretnym polu | `mapped_column(default=0, kw_only=True)` |
| Uporządkować pola ręcznie | pola obowiązkowe najpierw, opcjonalne na końcu |

W projektach produkcyjnych najczęstszy wybór to `kw_only=True` globalnie: kod jest odporny na przestawianie pól, a wywołania `Product(name="Kubek", price=...)` są czytelniejsze niż pozycyjne.

### 7.4 `default`, `default_factory` i `insert_default` — tabela prawdy

Przy `MappedAsDataclass` znaczenie `default` się zmienia. To najczęstsze źródło zamieszania:

| Konstrukcja | Działa z dataclass? | Działa bez dataclass? | Przyjmuje wartość? | Przyjmuje funkcję? | Ustawia obiekt od razu? |
|---|---|---|---|---|---|
| `default` | ✅ | ✅ | ✅ | tylko bez dataclass | ✅ (tylko z dataclass) |
| `insert_default` | ✅ | ✅ | ✅ | ✅ | ❌ |
| `default_factory` | ✅ | ❌ | ❌ | ✅ | ✅ (tylko z dataclass) |

```python
# examples/07_dataclass_defaults.py
from __future__ import annotations

import datetime

from sqlalchemy import func
from sqlalchemy.orm import Mapped, MappedAsDataclass, mapped_column


class Base(MappedAsDataclass, DeclarativeBase):
    pass


class Article(Base):
    __tablename__ = "article"

    id: Mapped[int] = mapped_column(primary_key=True, init=False)

    # default: stała wartość, widoczna od razu na obiekcie
    status: Mapped[str] = mapped_column(default="draft")

    # default_factory: funkcja wołana przy tworzeniu obiektu (dataclass-only!)
    tags: Mapped[list[str]] = mapped_column(default_factory=list)

    # insert_default: wartość użyta w INSERT, ale NIE na obiekcie
    views: Mapped[int] = mapped_column(insert_default=0, init=False)

    # server_default: wartość nadana przez bazę
    created_at: Mapped[datetime.datetime] = mapped_column(
        server_default=func.now(), init=False
    )


a = Article()
print(a.status, a.tags, a.views, a.created_at)
```

```text
draft [] None None
```

Zapamiętaj to zdanie: **`default_factory` jest parametrem wyłącznie dla dataclass i działa tylko przy tworzeniu obiektu w Pythonie.** Jeśli wstawiasz dane przez `insert(Article)` z Core (bez tworzenia obiektu), `default_factory` **nie zadziała**. Do takich przypadków jest `insert_default`.

> 🆕 **SQLAlchemy 2.1 — zmiana zachowania domyślnych wartości w dataclassach**
>
> W 2.0 wartości z `mapped_column(default=...)` były wpisywane do `__dict__` obiektu już w konstruktorze. Miało to poważny skutek uboczny: jeśli tworzyłeś obiekt przez klucz obcy zamiast przez relację —
>
> ```python
> order = Order(customer_id=5)   # zamiast Order(customer=customer_obj)
> ```
>
> — domyślna wartość `customer=None` (pochodząca z `relationship(default=...)`) **nadpisywała** poprawnie ustawiony `customer_id`, i do bazy szło `NULL`. Ten sam problem dotykał `Session.merge()`.
>
> W 2.1 zachowanie zostało zmienione: domyślne wartości **nie są już wpisywane do `__dict__`** obiektu. Zamiast tego są dostarczane przez deskryptor przy odczycie atrybutu, a w konstruktorze używany jest specjalny znacznik `DONT_SET`, który oznacza „nie ustawiono”:
>
> ```python
> # 2.1 — konstruktor zachowuje się tak:
> def __init__(self, related_id=DONT_SET, related=DONT_SET): ...
> ```
>
> Efekt praktyczny:
>
> ```python
> p = Parent(related_id=5)
> p.__dict__        # {'related_id': 5, '_sa_instance_state': ...}
> p.related         # None (dostarczone przez deskryptor, nie z __dict__)
> ```
>
> Dodatkowo w 2.1 domyślna wartość jest dostępna na obiekcie **także wtedy, gdy pole ma `init=False`** — czyli nie występuje w konstruktorze wcale.
>
> Jeśli z jakichś powodów potrzebujesz starego zachowania z 2.0, dokumentacja migracyjna opisuje, jak je przywrócić. W nowym kodzie nie rób tego — nowe zachowanie jest zgodne z intuicją: „default to default kolumny”, a nie „default to wartość wpisana do obiektu”.

### 7.5 Czego `MappedAsDataclass` nie obsługuje

| Funkcja dataclasses | Status |
|---|---|
| `init`, `repr`, `eq`, `order`, `unsafe_hash` | ✅ obsługiwane |
| `kw_only`, `match_args` | ✅ obsługiwane (Python 3.10+) |
| `frozen` | ❌ nieobsługiwane |
| `slots` | ❌ nieobsługiwane |
| `field(default_factory=...)` z `dataclasses` | ❌ używaj `mapped_column(default_factory=...)` |
| Składnia `x: int = 5` (bez `mapped_column`) | ❌ używaj `mapped_column(default=5)` |

> ⚠️ **Pułapka — `frozen=True` i `slots=True`**
>
> Jeżeli próbujesz użyć `@dataclass(frozen=True)` razem z modelem ORM, pamiętaj: SQLAlchemy musi móc modyfikować atrybuty, żeby śledzić zmiany i przypisywać klucze po `INSERT`. Niemutowalny model ORM to sprzeczność. Jeżeli potrzebujesz niemutowalnej struktury, użyj osobnej klasy (value object) i mapuj ją przez `TypeDecorator` (moduł 12) albo `composite()` (moduł 13).

> 🧪 **Ćwiczenie** — przerób model `Article` z punktu 3.3 na wersję z `MappedAsDataclass`, ustaw `kw_only=True` globalnie i sprawdź, czy `Article(status="published")` działa bez podawania pozostałych pól.

---

## 8. Mixiny i `@declared_attr`

### 8.1 Zwykły mixin wystarcza w 90% przypadków

Mixin to klasa, której nie mapujesz na tabelę, ale której atrybuty „wpadają” do klas, które po niej dziedziczą. W SQLAlchemy 2.0 **zwykłe kolumny w mixinie działają bez żadnych dekoratorów**:

```python
# examples/07_mixins_basic.py
from __future__ import annotations

import datetime

from sqlalchemy import DateTime, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class IdMixin:
    id: Mapped[int] = mapped_column(primary_key=True, sort_order=-100)


class TimestampMixin:
    created_at: Mapped[datetime.datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), sort_order=100
    )
    updated_at: Mapped[datetime.datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
        onupdate=func.now(),
        sort_order=101,
    )


class Book(IdMixin, TimestampMixin, Base):
    __tablename__ = "book"

    title: Mapped[str]


print(Book.__table__.columns.keys())
```

```text
['id', 'title', 'created_at', 'updated_at']
```

> 💡 **Analogia — pieczątka na dokumencie**
>
> Mixin to pieczątka nagłówkowa, którą przykładasz do każdego dokumentu. Dokument `Book` ma nagłówek „ID, utworzono, zmieniono” dlatego, że przyłożono do niego dwie pieczątki. Każdy dokument dostaje **własną kopię** treści pieczątki — nie dzielą jednej kolumny.

### 8.2 Kiedy potrzebny `@declared_attr`

Są rzeczy, których w momencie definiowania mixinu jeszcze nie znasz — bo zależą od klasy, która po nim dziedziczy. Wtedy używasz `@declared_attr`:

```python
# examples/07_declared_attr.py
from __future__ import annotations

from sqlalchemy import ForeignKey
from sqlalchemy.orm import DeclarativeBase, Mapped, declared_attr, mapped_column, relationship


class Base(DeclarativeBase):
    pass


class RefTargetMixin:
    """Mixin dodający kolumnę i relację wskazującą na dowolną tabelę."""

    @declared_attr
    def target_id(cls) -> Mapped[int]:
        return mapped_column(ForeignKey("target.id"))

    @declared_attr
    def target(cls) -> Mapped["Target"]:
        return relationship("Target")
```

Kiedy `@declared_attr` jest **konieczne**:

| Przypadek | Dlaczego |
|---|---|
| `mapped_column(ForeignKey(...))` w mixinie | obiekt `ForeignKey` nie może być współdzielony między tabelami — każda potrzebuje własnej kopii |
| `relationship()` w mixinie | mapper musi wiedzieć, do której klasy należy relacja |
| `__tablename__` w mixinie | nazwa zależy od klasy |
| `__table_args__` w mixinie | j.w. |
| `__mapper_args__` w mixinie | j.w. |

Kiedy `@declared_attr` jest **zbędne**: zwykłe kolumny bez `ForeignKey`, metody, właściwości (`@property`).

### 8.3 `IdMixin`, `TimestampMixin`, `SoftDeleteMixin` — wersje produkcyjne

```python
# examples/07_mixins_production.py
"""Mixiny, które w tej lub zbliżonej formie znajdziesz w realnych projektach."""
from __future__ import annotations

import datetime
import uuid
from typing import Any

from sqlalchemy import DateTime, ForeignKey, String, Uuid, func
from sqlalchemy.orm import (
    DeclarativeBase,
    Mapped,
    declared_attr,
    mapped_column,
    relationship,
)

NAMING_CONVENTION = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}


class Base(DeclarativeBase):
    metadata = MetaData(naming_convention=NAMING_CONVENTION)
    type_annotation_map = {dict[str, Any]: JSON}


# --- 1. Klucz główny ---------------------------------------------------
class IntIdMixin:
    id: Mapped[int] = mapped_column(primary_key=True, sort_order=-100)


class UuidIdMixin:
    """Uuid jako klucz: nie ujawnia liczby rekordów i jest bezpieczny przy scalaniu."""

    id: Mapped[uuid.UUID] = mapped_column(
        Uuid(as_uuid=True), primary_key=True, default=uuid.uuid4, sort_order=-100
    )


# --- 2. Znaczniki czasu ------------------------------------------------
class TimestampMixin:
    created_at: Mapped[datetime.datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), sort_order=100
    )
    updated_at: Mapped[datetime.datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
        onupdate=func.now(),
        sort_order=101,
    )


# --- 3. „Kto utworzył” --------------------------------------------------
class CreatedByMixin:
    @declared_attr
    def created_by_id(cls) -> Mapped[int | None]:
        return mapped_column(ForeignKey("app_user.id"), sort_order=102)

    @declared_attr
    def created_by(cls) -> Mapped["AppUser | None"]:
        return relationship("AppUser", foreign_keys=[cls.created_by_id])
```

Trzy uwagi projektowe:

1. **`IntIdMixin` kontra `UuidIdMixin`.** Nie mieszaj ich w jednym projekcie bez powodu — spójność kluczy upraszcza kod (moduł 12 omawia wady i zalety `Uuid`).
2. **`sort_order` w mixinach.** Ustawia kolejność kolumn w DDL, ale ma konsekwencje przy `MappedAsDataclass` (punkt 8.5).
3. **`foreign_keys=[cls.created_by_id]`.** Bez tego SQLAlchemy nie wie, po której kolumnie łączyć — klasa `AppUser` może być wskazywana przez kilka kolumn.

### 8.4 `__table_args__` w mixinie — `declared_attr.directive`

Jeżeli mixin ma dodać ograniczenie lub indeks, użyj `@declared_attr.directive`:

```python
# examples/07_mixin_table_args.py
from __future__ import annotations

from sqlalchemy import CheckConstraint, Index
from sqlalchemy.orm import declared_attr


class ActiveFlagMixin:
    @declared_attr.directive
    def __table_args__(cls) -> tuple[Index, CheckConstraint]:
        return (
            Index(f"ix_{cls.__tablename__}_active", "is_active"),
            CheckConstraint("is_active IN (0, 1)", name="active_is_boolean"),
        )
```

Efekt: każda klasa dziedzicząca po `ActiveFlagMixin` dostaje własny indeks i własne ograniczenie, z nazwą zależną od swojej tabeli.

> ⚠️ **Pułapka — kolizja `__table_args__` w dziedziczeniu**
>
> Jeżeli klasa dziedziczy po **dwóch** mixinach, które definiują `__table_args__`, ostatni wygrywa — pierwszy zostanie **cicho pominięty**. Nie ma tu automatycznego łączenia krotek. Rozwiązanie: jeden mixin zarządza `__table_args__`, albo każda klasa definiuje je sama i wywołuje pomocnicze funkcje z mixinów.

### 8.5 Mixin + `MappedAsDataclass` = klasyczna pułapka

To bardzo częsty problem w projektach, które używają obu mechanizmów:

```python
# examples/07_mixin_dataclass_trap.py
from __future__ import annotations

import datetime

from sqlalchemy import DateTime, func
from sqlalchemy.orm import DeclarativeBase, Mapped, MappedAsDataclass, mapped_column


class Base(MappedAsDataclass, DeclarativeBase, kw_only=True):
    pass


class TimestampMixin:
    # UWAGA: tutaj NIE ma kw_only=True, więc pole ma "domyślną" wartość
    # w rozumieniu dataclasses — i to psuje kolejność argumentów
    created_at: Mapped[datetime.datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now()
    )


class Book(TimestampMixin, Base):
    __tablename__ = "book"

    id: Mapped[int] = mapped_column(primary_key=True, init=False)
    title: Mapped[str]  # brak domyślnej — a jest PO polu z domyślną!


# TypeError: non-default argument 'title' follows default argument
```

Naprawa — dwie drogi:

```python
# Wariant A: kw_only w mixinie (zalecany przy kw_only=True na Base)
class TimestampMixin:
    created_at: Mapped[datetime.datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), kw_only=True
    )


# Wariant B: init=False — pole w ogóle nie trafia do konstruktora
class TimestampMixin:
    created_at: Mapped[datetime.datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), init=False
    )
```

> 🧠 **Dlaczego tak jest** — Python buduje `__init__` dataclass, przechodząc pola w kolejności MRO (kolejności rozwiązywania metod). Pola z mixinów wypadają **przed** polami klasy. Jeśli mixin ma pole z domyślną wartością, a klasa — bez, Python widzi klasyczny błąd „argument bez domyślnej po argumencie z domyślną”. SQLAlchemy nie może tego obejść, bo generuje zwykłą dataclassę.
>
> **Reguła praktyczna:** w mixinach używanych razem z `MappedAsDataclass` każde pole techniczne (`created_at`, `id`, `updated_at`) powinno mieć `kw_only=True` albo `init=False`.

---

## 9. `registry`, `metadata` i podział projektu na pliki

### 9.1 `registry`

`registry` to obiekt, który prowadzi rejestr wszystkich mapperów i wie, jak tłumaczyć adnotacje. `DeclarativeBase` tworzy go automatycznie — ale możesz go stworzyć samodzielnie, gdy potrzebujesz większej kontroli:

```python
# examples/07_registry.py
from __future__ import annotations

import decimal
from typing import Any

from sqlalchemy import JSON, MetaData, Numeric, String
from sqlalchemy.orm import Mapped, registry

# 1. Własny registry z konfiguracją globalną
mapper_registry = registry(
    metadata=MetaData(),
    type_annotation_map={
        decimal.Decimal: Numeric(12, 2),
        dict[str, Any]: JSON,
        str: String(255),
    },
)

# 2. Klasa bazowa z tego registry
Base = mapper_registry.generate_base()


class Customer(Base):
    __tablename__ = "customer"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]


print(mapper_registry.mappers)          # [<Mapper at ...; Customer>]
print(Base.metadata is mapper_registry.metadata)  # True
```

Dostępne metody `registry`:

| Metoda / atrybut | Do czego |
|---|---|
| `registry.generate_base()` | tworzy klasę bazową (odpowiednik `DeclarativeBase`) |
| `registry.mapped` | dekorator: `@registry.mapped` na zwykłej klasie |
| `registry.map_imperatively(Klasa, table)` | mapuje istniejącą klasę na istniejącą tabelę |
| `registry.metadata` | przypisana `MetaData` |
| `registry.mappers` | kolekcja mapperów |
| `registry.configure()` | wymusza skonfigurowanie wszystkich mapperów |
| `registry.dispose()` | usuwa wszystkie mapowania (przydatne w testach) |

### 9.2 Wiele `MetaData` — gdy naprawdę potrzebujesz

Domyślnie wszystkie modele dzielą jedną `MetaData` (tę z klasy bazowej). Czasem trzeba to rozdzielić:

```python
# examples/07_two_metadata.py
from __future__ import annotations

from sqlalchemy import MetaData
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

core_metadata = MetaData()
analytics_metadata = MetaData()


class CoreBase(DeclarativeBase):
    metadata = core_metadata


class AnalyticsBase(DeclarativeBase):
    metadata = analytics_metadata


class Customer(CoreBase):
    __tablename__ = "customer"

    id: Mapped[int] = mapped_column(primary_key=True)


class SalesFact(AnalyticsBase):
    __tablename__ = "sales_fact"

    id: Mapped[int] = mapped_column(primary_key=True)
```

Kiedy to ma sens:

| Sytuacja | Rozdzielać `MetaData`? |
|---|---|
| Dwie różne bazy danych (OLTP + hurtownia) | ✅ tak |
| Ten sam PostgreSQL, ale osobne schematy zarządzane osobno | ✅ czasem |
| Migracje z Alembicem dla różnych części systemu | ✅ tak (osobne `target_metadata`) |
| Zwykły projekt jednej bazy | ❌ nie — jedna `MetaData` to mniej kłopotów |

> ⚠️ **Pułapka — dwa `MetaData` i klucz obcy między nimi**
>
> Jeżeli tabela z `core_metadata` ma `ForeignKey("sales_fact.id")`, a `sales_fact` jest w `analytics_metadata`, `create_all` nie zadziała — SQLAlchemy nie potrafi posortować tabel z różnych `MetaData`. Klucze obce **muszą** być w obrębie jednej `MetaData`. Alternatywa: `metadata.schema` + `schema_translate_map` albo osobne połączenie.

### 9.3 Struktura pakietu `models/`

Monolityczny `models.py` z 40 klasami jest nie do utrzymania. Typowa, sprawdzona struktura:

```text
app/
├── db/
│   ├── __init__.py          # eksportuje Base
│   ├── base.py              # Base + naming convention + type_annotation_map
│   └── session.py           # engine, sessionmaker (moduł 08)
├── models/
│   ├── __init__.py          # importuje WSZYSTKIE modele (ważne!)
│   ├── mixins.py            # IdMixin, TimestampMixin, SoftDeleteMixin
│   ├── customer.py          # Customer
│   ├── product.py           # Product, Category
│   └── order.py             # Order, OrderItem
└── main.py
```

```python
# app/db/base.py
from __future__ import annotations

from typing import Any

from sqlalchemy import JSON, MetaData, Numeric, String
from sqlalchemy.orm import DeclarativeBase

NAMING_CONVENTION = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}


class Base(DeclarativeBase):
    metadata = MetaData(naming_convention=NAMING_CONVENTION)
    type_annotation_map = {
        dict[str, Any]: JSON,
        str: String(255),
    }
```

```python
# app/models/__init__.py
"""Importuje wszystkie modele, aby `Base.metadata` znało pełny schemat."""
from app.models.customer import Customer
from app.models.order import Order, OrderItem
from app.models.product import Category, Product

__all__ = ["Category", "Customer", "Order", "OrderItem", "Product"]
```

```python
# app/models/customer.py
from __future__ import annotations

from sqlalchemy.orm import Mapped, mapped_column

from app.db.base import Base


class Customer(Base):
    __tablename__ = "customer"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(unique=True)
    full_name: Mapped[str]
```

> 🧠 **Dlaczego `models/__init__.py` musi importować wszystkie modele** — deklaratywne mapowanie dzieje się w momencie **definicji klasy**, czyli przy imporcie modułu. Jeżeli `app/models/order.py` nie zostanie zaimportowany, klasa `Order` nigdy nie powstanie, a `Base.metadata` nie będzie zawierać tabeli `orders`. `create_all` utworzy niekompletny schemat, a `alembic autogenerate` wygeneruje migrację, która **usunie** tabelę `orders` z bazy. Ten błąd zdarza się w realnych projektach i jest trudny do zauważenia w testach, które i tak importują modele pośrednio.

### 9.4 Importy cykliczne

Gdy modele są w osobnych plikach, a mają relacje między sobą, łatwo wpaść w cykl importów:

```text
customer.py  →  importuje  →  order.py
order.py     →  importuje  →  customer.py     ← cykl!
```

Trzy sprawdzone rozwiązania:

```python
# Rozwiązanie A: cudzysłowy w relacjach — NAJPROSTSZE i najczęstsze
# app/models/order.py
from __future__ import annotations

from sqlalchemy import ForeignKey
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.db.base import Base


class Order(Base):
    __tablename__ = "order"

    id: Mapped[int] = mapped_column(primary_key=True)
    customer_id: Mapped[int] = mapped_column(ForeignKey("customer.id"))

    # NIE importujemy Customer — wystarczy napis
    customer: Mapped["Customer"] = relationship(back_populates="orders")
```

```python
# Rozwiązanie B: TYPE_CHECKING — gdy potrzebujesz typu w adnotacji
# app/models/customer.py
from __future__ import annotations

from typing import TYPE_CHECKING

from sqlalchemy.orm import Mapped, relationship

from app.db.base import Base

if TYPE_CHECKING:
    from app.models.order import Order


class Customer(Base):
    __tablename__ = "customer"

    id: Mapped[int] = mapped_column(primary_key=True)
    orders: Mapped[list["Order"]] = relationship(back_populates="customer")
```

```python
# Rozwiązanie C: konfiguracja relacji przez registry (rzadkie)
# app/db/base.py — po zdefiniowaniu wszystkich klas:
from app.models.customer import Customer
from app.models.order import Order

# relacje dokładane po fakcie — tylko dla nietypowych przypadków
```

| Metoda | Kiedy stosować |
|---|---|
| Cudzysłów `"ClassName"` | zawsze — to domyślna, najprostsza droga |
| `TYPE_CHECKING` + `from __future__ import annotations` | gdy edytor/`mypy` potrzebuje typu do podpowiedzi |
| Import w `__init__.py` | gdy chcesz mieć wygodny dostęp: `from app.models import Order` |
| Konfiguracja po fakcie | tylko w bardzo skomplikowanych przypadkach (cykl przez trzy moduły) |

> ⚠️ **Pułapka — `from __future__ import annotations` i `Mapped[]`**
>
> `from __future__ import annotations` sprawia, że **wszystkie** adnotacje stają się napisami i są rozwiązywane leniwie. SQLAlchemy to obsługuje — ale wymaga, żeby typy były **rozwiązywalne** w module, w którym zdefiniowano klasę. Jeżeli w adnotacji użyjesz typu zaimportowanego tylko pod `TYPE_CHECKING`, a nie w cudzysłowie, wszystko zadziała (bo napis nie jest rozwiązywany). Jeżeli jednak SQLAlchemy będzie musiało ten napis rozwiązać (np. dla `Mapped[list[Order]]` bez cudzysłowów), zgłosi `NameError: Could not de-stringify annotation`.

---

## 10. Typowanie i `mypy` na modelach

### 10.1 Co `Mapped[]` daje narzędziom

Adnotacje w stylu 2.0 to nie dekoracja — one realnie pracują:

```python
# examples/07_typing_demo.py
from __future__ import annotations

from sqlalchemy import select
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class Product(Base):
    __tablename__ = "product"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    price: Mapped[decimal.Decimal]
    description: Mapped[str | None]


stmt = select(Product.name, Product.price)

# W edytorze i w mypy:
#   - stmt jest typu Select[tuple[str, Decimal]]
#   - p.name to str, p.description to str | None
#   - p.id = "abc"  →  błąd typowania
```

Korzyści w praktyce:

- **Autouzupełnianie.** `product.` pokazuje tylko prawdziwe kolumny, nie `__dict__` i `_sa_instance_state`.
- **Wykrywanie literówek.** `product.nmae` to błąd `mypy`, nie `AttributeError` w produkcji.
- **Sprawdzanie typów w zapytaniach.** `where(Product.price > "dużo")` zostanie zgłoszone jako błąd.
- **Refaktoryzacja.** Zmiana nazwy atrybutu jest bezpieczna, bo narzędzie widzi wszystkie użycia.

### 10.2 Pułapki typowania

| Pułapka | Objaw | Rozwiązanie |
|---|---|---|
| `Mapped[Optional[str]]` w 2.0 vs `Mapped[str \| None]` | w 2.0 oba działają; `\| None` wymaga Pythona ≥ 3.10 | używaj `\| None` (Python 3.11+) |
| `Mapped[str]` bez wartości domyślnej | `mypy` nie narzeka, ale przy tworzeniu obiektu brakuje wartości | to nie błąd — obiekt może istnieć bez wartości, dopóki nie trafi do bazy |
| Zwykła adnotacja bez `Mapped` | `sqlalchemy.exc.ArgumentError: Type annotation for "X.y" can't be correctly interpreted` | użyj `Mapped[...]`, `ClassVar[...]` albo `__allow_unmapped__ = True` |
| `Mapped[list[str]]` bez mapowania na JSON | `ArgumentError` | dodaj `dict`/`list` do `type_annotation_map` |
| Atrybut nie-kolumna z adnotacją | błąd jak wyżej | `ClassVar[int]` dla stałych klasowych |

```python
# examples/07_classvar.py
from typing import ClassVar

from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class Product(Base):
    __tablename__ = "product"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]

    # 1. Stała klasowa — NIE jest kolumną.
    MAX_NAME_LENGTH: ClassVar[int] = 120

    # 2. Atrybut instancji poza mapowaniem — wymaga __allow_unmapped__
    __allow_unmapped__ = True
    _cache: dict[str, str]

    def __init__(self, name: str) -> None:
        self.name = name
        self._cache = {}
```

> 🧠 **Dlaczego tak jest** — SQLAlchemy skanuje **wszystkie** adnotacje w klasie. Nie umie odróżnić „adnotacja opisująca kolumnę” od „adnotacja dla `mypy`”. Dlatego wprowadzono kontener `Mapped[]` jako znacznik intencji. Gdy adnotacja nie ma `Mapped`, SQLAlchemy domyślnie zgłasza błąd — bo w 99% przypadków oznacza to zapomniany `Mapped`. `ClassVar` to standardowy sposób Pythona na powiedzenie „to pole klasowe, nie instancyjne”, więc SQLAlchemy je honoruje.
>
> **Nie ustawiaj `__allow_unmapped__ = True` globalnie bez powodu.** To wyłącza ochronę przed najczęstszym błędem w ORM-owym kodzie — zapomnianym `Mapped`.

### 10.3 Uruchomienie `mypy`

```bash
# instalacja
pip install "sqlalchemy[mypy]" mypy

# sprawdzenie całego projektu
mypy --strict app/

# sprawdzenie jednego pliku
mypy app/models/customer.py
```

Od SQLAlchemy 2.0 **wtyczka do `mypy` nie jest potrzebna** dla modeli w stylu 2.0. Wtyczka (`plugins = sqlalchemy.ext.mypy.plugin`) istnieje wyłącznie dla kodu w starym stylu 1.x. Jeżeli zaczynasz nowy projekt — nie dodawaj jej.

Zalecana konfiguracja w `pyproject.toml`:

```toml
# pyproject.toml
[tool.mypy]
python_version = "3.11"
strict = true
warn_unused_ignores = true
warn_redundant_casts = true
plugins = []          # dla stylu 2.0 — puste!
```

> 🆕 **SQLAlchemy 2.1** — w 2.1 rozszerzono typowanie generyczne: `Result` i `Row` korzystają z parametryzacji variadic (PEP 646), co pozwala precyzyjnie opisać liczbę i typy kolumn w wyniku zapytania. Dodatkowo 2.1 wymaga **Pythona 3.11+** — jeżeli utrzymujesz kod na 3.10, zostajesz na linii 2.0.

> 🧪 **Ćwiczenie** — uruchom `mypy --strict` na pliku `07_dataclass_models.py` z punktu 7.2. Napraw wszystkie zgłoszone problemy. Ile z nich dotyczyło `Mapped`, a ile zwykłego typowania Pythona?

---

## 11. Przykład obowiązkowy: Core kontra ORM na tym samym schemacie

Ten rozdział to domknięcie modułu. Zapisujemy **ten sam schemat sklepu** w dwóch wersjach i porównujemy wygenerowany DDL. Celem jest zobaczenie, że ORM nie jest osobnym światem — to generator tabel, które już znasz z modułu 03.

### 11.1 Wersja Core — przypomnienie z modułu 03

```python
# examples/07_core_version.py
"""Schemat sklepu w czystym Core — odpowiednik modułu 03."""
from __future__ import annotations

import datetime
import decimal

from sqlalchemy import (
    CheckConstraint,
    Column,
    DateTime,
    ForeignKey,
    Index,
    Integer,
    MetaData,
    Numeric,
    String,
    Table,
    UniqueConstraint,
    func,
)

metadata = MetaData()

category = Table(
    "category",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String(80), nullable=False, unique=True),
    Column("parent_id", Integer, ForeignKey("category.id"), nullable=True),
)

customer = Table(
    "customer",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("email", String(255), nullable=False, unique=True),
    Column("full_name", String(120), nullable=False),
    Column("nickname", String(60), nullable=True),
    Column("is_active", Integer, nullable=False, server_default="1"),
    Column(
        "created_at",
        DateTime(timezone=True),
        nullable=False,
        server_default=func.now(),
    ),
)

product = Table(
    "product",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("sku", String(32), nullable=False),
    Column("name", String(120), nullable=False),
    Column("price", Numeric(12, 2), nullable=False),
    Column("category_id", Integer, ForeignKey("category.id"), nullable=False),
    Column("is_active", Integer, nullable=False, server_default="1"),
    UniqueConstraint("sku", name="uq_product_sku"),
    CheckConstraint("price >= 0", name="ck_product_price_non_negative"),
    Index("ix_product_category_active", "category_id", "is_active"),
)

order = Table(
    "order",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("customer_id", Integer, ForeignKey("customer.id"), nullable=False),
    Column(
        "placed_at",
        DateTime(timezone=True),
        nullable=False,
        server_default=func.now(),
    ),
    Column("status", String(20), nullable=False, server_default="new"),
)

order_item = Table(
    "order_item",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("order_id", Integer, ForeignKey("order.id"), nullable=False),
    Column("product_id", Integer, ForeignKey("product.id"), nullable=False),
    Column("quantity", Integer, nullable=False),
    Column("unit_price", Numeric(12, 2), nullable=False),
    UniqueConstraint("order_id", "product_id", name="uq_order_item_order_product"),
)
```

### 11.2 Wersja ORM — `models.py`

```python
# examples/07_models.py
"""Modele deklaratywne sklepu — SQLAlchemy 2.0."""
from __future__ import annotations

import datetime
import decimal
from typing import Any

from sqlalchemy import (
    JSON,
    CheckConstraint,
    DateTime,
    ForeignKey,
    Index,
    MetaData,
    Numeric,
    String,
    UniqueConstraint,
    func,
)
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship

# --------------------------------------------------------------------------
# 1. Konwencja nazewnictwa — jedna, globalna, dla całego projektu
# --------------------------------------------------------------------------
NAMING_CONVENTION = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}


# --------------------------------------------------------------------------
# 2. Klasa bazowa — wspólna dla wszystkich modeli
# --------------------------------------------------------------------------
class Base(DeclarativeBase):
    metadata = MetaData(naming_convention=NAMING_CONVENTION)

    type_annotation_map = {
        dict[str, Any]: JSON,
        decimal.Decimal: Numeric(12, 2),
    }


# --------------------------------------------------------------------------
# 3. Mixiny — pola techniczne, wspólne dla wielu tabel
# --------------------------------------------------------------------------
class IdMixin:
    id: Mapped[int] = mapped_column(primary_key=True, sort_order=-100)


class TimestampMixin:
    created_at: Mapped[datetime.datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
        sort_order=100,
    )


class ActiveMixin:
    is_active: Mapped[bool] = mapped_column(
        default=True,
        server_default="1",  # SQLite/MySQL; na PostgreSQL użyj text("true")
        sort_order=101,
    )


class SoftDeleteMixin:
    """Zamiast usuwać rekord, oznaczamy go jako usunięty."""

    deleted_at: Mapped[datetime.datetime | None] = mapped_column(
        DateTime(timezone=True),
        default=None,
        index=True,
        sort_order=102,
    )

    @property
    def is_deleted(self) -> bool:
        return self.deleted_at is not None

    def soft_delete(self, when: datetime.datetime | None = None) -> None:
        self.deleted_at = when or datetime.datetime.now(datetime.UTC)


# --------------------------------------------------------------------------
# 4. Modele
# --------------------------------------------------------------------------
class Category(IdMixin, Base):
    __tablename__ = "category"

    name: Mapped[str] = mapped_column(String(80), unique=True)
    parent_id: Mapped[int | None] = mapped_column(ForeignKey("category.id"))

    products: Mapped[list["Product"]] = relationship(back_populates="category")
    # Relacje omawiamy w pełni w module 09 — tutaj tylko zapowiedź.


class Customer(IdMixin, TimestampMixin, ActiveMixin, SoftDeleteMixin, Base):
    __tablename__ = "customer"

    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    full_name: Mapped[str] = mapped_column(String(120))
    nickname: Mapped[str | None] = mapped_column(String(60))
    preferences: Mapped[dict[str, Any]] = mapped_column(default=dict)

    orders: Mapped[list["Order"]] = relationship(back_populates="customer")

    @property
    def display_name(self) -> str:
        """Metoda domenowa — dowód, że obiekt to nie tylko worek pól."""
        return self.nickname or self.full_name


class Product(IdMixin, ActiveMixin, Base):
    __tablename__ = "product"

    sku: Mapped[str] = mapped_column(String(32))
    name: Mapped[str] = mapped_column(String(120))
    price: Mapped[decimal.Decimal]  # → Numeric(12, 2) z type_annotation_map
    category_id: Mapped[int] = mapped_column(ForeignKey("category.id"))

    category: Mapped["Category"] = relationship(back_populates="products")

    __table_args__ = (
        UniqueConstraint("sku", name="uq_product_sku"),
        CheckConstraint("price >= 0", name="ck_product_price_non_negative"),
        Index("ix_product_category_active", "category_id", "is_active"),
    )

    @property
    def price_with_vat(self) -> decimal.Decimal:
        return (self.price * decimal.Decimal("1.23")).quantize(
            decimal.Decimal("0.01")
        )


class Order(IdMixin, TimestampMixin, Base):
    __tablename__ = "order"

    customer_id: Mapped[int] = mapped_column(ForeignKey("customer.id"))
    placed_at: Mapped[datetime.datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now()
    )
    status: Mapped[str] = mapped_column(String(20), default="new")

    customer: Mapped["Customer"] = relationship(back_populates="orders")
    items: Mapped[list["OrderItem"]] = relationship(back_populates="order")

    @property
    def total(self) -> decimal.Decimal:
        return sum((item.subtotal for item in self.items), decimal.Decimal("0"))


class OrderItem(IdMixin, Base):
    __tablename__ = "order_item"

    order_id: Mapped[int] = mapped_column(ForeignKey("order.id"))
    product_id: Mapped[int] = mapped_column(ForeignKey("product.id"))
    quantity: Mapped[int]
    unit_price: Mapped[decimal.Decimal]

    order: Mapped["Order"] = relationship(back_populates="items")
    product: Mapped["Product"] = relationship()

    __table_args__ = (
        UniqueConstraint(
            "order_id", "product_id", name="uq_order_item_order_product"
        ),
    )

    @property
    def subtotal(self) -> decimal.Decimal:
        return self.unit_price * self.quantity
```

### 11.3 Porównanie wygenerowanego DDL

```python
# examples/07_ddl_preview.py
"""Drukuje DDL wygenerowany z modeli ORM — do porównania z wersją Core."""
from __future__ import annotations

from sqlalchemy import create_engine
from sqlalchemy.schema import CreateIndex, CreateTable

from examples.models import Base

engine = create_engine("sqlite:///:memory:", echo=False)

for table in Base.metadata.sorted_tables:
    print(str(CreateTable(table).compile(engine)).strip())
    for index in sorted(table.indexes, key=lambda i: i.name or ""):
        print(str(CreateIndex(index).compile(engine)).strip())
    print("-" * 70)
```

Fragment wyniku:

```sql
CREATE TABLE category (
	id INTEGER NOT NULL, 
	name VARCHAR(80) NOT NULL, 
	parent_id INTEGER, 
	PRIMARY KEY (id), 
	UNIQUE (name), 
	CONSTRAINT fk_category_parent_id_category FOREIGN KEY(parent_id) REFERENCES category (id)
)
CREATE INDEX ix_category_name ON category (name)
----------------------------------------------------------------------
CREATE TABLE customer (
	id INTEGER NOT NULL, 
	email VARCHAR(255) NOT NULL, 
	full_name VARCHAR(120) NOT NULL, 
	nickname VARCHAR(60), 
	created_at DATETIME NOT NULL, 
	is_active BOOLEAN NOT NULL, 
	deleted_at DATETIME, 
	preferences JSON NOT NULL, 
	PRIMARY KEY (id), 
	UNIQUE (email), 
	CONSTRAINT ck_customer_active_is_boolean CHECK (is_active IN (0, 1))
)
----------------------------------------------------------------------
CREATE TABLE product (
	id INTEGER NOT NULL, 
	sku VARCHAR(32) NOT NULL, 
	name VARCHAR(120) NOT NULL, 
	price NUMERIC(12, 2) NOT NULL, 
	category_id INTEGER NOT NULL, 
	is_active BOOLEAN NOT NULL, 
	PRIMARY KEY (id), 
	CONSTRAINT uq_product_sku UNIQUE (sku), 
	CONSTRAINT ck_product_price_non_negative CHECK (price >= 0), 
	CONSTRAINT fk_product_category_id_category FOREIGN KEY(category_id) REFERENCES category (id)
)
CREATE INDEX ix_product_category_active ON product (category_id, is_active)
----------------------------------------------------------------------
```

Wnioski z porównania:

| Aspekt | Core (`Table`) | ORM (klasy) |
|---|---|---|
| Wynikowy DDL | identyczny (przy tych samych parametrach) | identyczny |
| Czytelność schematu | jedna tabela = jedna deklaracja, widać wszystko naraz | jedna klasa = jedna tabela, ale kolumny rozproszone między mixiny |
| Zachowanie | brak — tabele nie mają metod | `order.total`, `customer.display_name`, `product.price_with_vat` |
| Typowanie | brak | pełne `Mapped[...]` |
| Relacje | trzeba pisać joiny ręcznie | `order.customer` zwraca obiekt |
| Wykorzystanie w aplikacji | świetne do raportów i hurtowni | świetne do logiki biznesowej |
| Ryzyko | literówka w nazwie kolumny wychodzi dopiero w runtime | literówka wychodzi w `mypy` |

Zwróć uwagę na dwie różnice, które **nie** wynikają z ORM, a z naszych decyzji projektowych:

- `preferences JSON` pojawiło się dzięki wpisowi w `type_annotation_map` — w wersji Core trzeba było jawnie napisać `JSON()`.
- `ck_customer_active_is_boolean` w wersji ORM pochodzi z `ActiveMixin`; w wersji Core nie było tego ograniczenia.

### 11.4 Jak uruchomić

```bash
# 1. Środowisko
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install "SQLAlchemy>=2.0"

# 2. Struktura
mkdir -p examples
touch examples/__init__.py
# skopiuj pliki 07_models.py i 07_ddl_preview.py do examples/

# 3. Uruchomienie podglądu DDL
python -m examples.07_ddl_preview

# 4. Utworzenie bazy na dysku (SQLite)
python -c "
from sqlalchemy import create_engine
from examples.models import Base
engine = create_engine('sqlite:///shop.db')
Base.metadata.create_all(engine)
print('Utworzono:', sorted(Base.metadata.tables))
"
```

Na PostgreSQL wystarczy zmienić URL i sterownik:

```bash
pip install "psycopg[binary]"
python -c "
from sqlalchemy import create_engine
from examples.models import Base
engine = create_engine('postgresql+psycopg://user:pass@localhost:5432/shop')
Base.metadata.create_all(engine)
"
```

> ⚠️ **Pułapka — `create_all` to nie migracje**
>
> `Base.metadata.create_all(engine)` **tworzy brakujące tabele i nie robi nic więcej**. Jeżeli tabela już istnieje, nie zostanie zmieniona — nawet jeśli dodałeś kolumnę w modelu. Do zmiany istniejącego schematu służy Alembic (moduł 16). Trzymanie `create_all` w kodzie produkcyjnym jest antywzorcem, który omawiamy w module 24.

> 🧪 **Ćwiczenie** — dodaj do modelu `Product` kolumnę `weight_grams: Mapped[int | None]` z `CheckConstraint("weight_grams > 0")` w `__table_args__`. Sprawdź w DDL, że ograniczenie dostało nazwę zgodną z konwencją. Następnie spróbuj wygenerować różnicę przez Alembic (`alembic revision --autogenerate`) i zobacz, jak wygląda migracja.

---

## Podsumowanie

Zabierz ze sobą te punkty:

1. **Wiersz to nie obiekt.** Baza myśli zbiorami i wierszami; Python myśli obiektami i zachowaniem. ORM jest tłumaczem, nie zamiennikiem SQL-a.
2. **`DeclarativeBase` to punkt startowy.** Dziedziczysz po niej, a SQLAlchemy zamienia klasy na `Table` i `Mapper` w momencie **definicji klasy**, nie w momencie utworzenia obiektu.
3. **`Mapped[T]` mówi dwa razy:** jaki jest typ Pythona i czy kolumna może być `NULL`. `Mapped[str]` → `NOT NULL`, `Mapped[str | None]` → `NULL`.
4. **`mapped_column()` to `Column()` z modułu 03 plus warstwa ORM.** Wszystkie parametry z Core działają identycznie, dodatkowo masz `deferred`, `sort_order`, `init`, `default_factory`.
5. **`default` to Python, `server_default` to baza.** Dane wstawiane z innego narzędzia nie zobaczą `default`.
6. **`Annotated` i `type_annotation_map` usuwają powtarzalność.** `Annotated[str, 120]` działa od razu; `dict`/`list` → JSON musisz dodać sam.
7. **`__tablename__` jest obowiązkowe.** `__table_args__` przyjmuje ograniczenia, indeksy i słownik opcji na końcu.
8. **`MappedAsDataclass` generuje `__init__` i `__repr__`, ale zmienia znaczenie `default`.** Pola bez domyślnej wartości muszą poprzedzać te z domyślną — albo użyj `kw_only=True`.
9. **Mixiny to pieczątki.** Zwykłe kolumny działają bez dekoratorów; `@declared_attr` jest konieczne dla `ForeignKey`, `relationship` i `__table_args__`.
10. **`models/__init__.py` musi importować wszystkie modele.** Bez tego `create_all` utworzy niekompletny schemat, a `autogenerate` zaproponuje usunięcie brakujących tabel.
11. **Nie używaj `__allow_unmapped__` bez potrzeby.** To wyłącza ochronę przed najczęstszym błędem w modelach — zapomnianym `Mapped`.
12. **Konwencja nazewnictwa ustawiona raz na `MetaData` oszczędza godziny przy migracjach.**

---

## Ćwiczenia

### Zadanie 1 — mixin „kto i kiedy utworzył rekord” (łatwe)

Dodaj do modelu `Product` (z przykładu obowiązkowego) informacje o tym, kto i kiedy utworzył rekord. Wymagania:

- Nowy mixin `AuditMixin` z polami `created_by_id: Mapped[int | None]` (klucz obcy do `customer.id`) i `created_at` (już masz w `TimestampMixin` — nie duplikuj).
- Pole `created_by_id` ma mieć `sort_order` ustawiony tak, aby w tabeli pojawiło się **po** `is_active`.
- Dodaj relację `created_by` do `Customer` — użyj `foreign_keys=[cls.created_by_id]`, żeby SQLAlchemy wiedziało, po której kolumnie łączyć.
- Wydrukuj DDL i sprawdź kolejność kolumn oraz nazwę klucza obcego.

### Zadanie 2 — rozbicie monolitu na pakiet (średnie)

Rozbij plik `examples/07_models.py` na pakiet:

```text
app/
├── db/base.py          # Base + NAMING_CONVENTION + type_annotation_map
├── models/__init__.py  # importy wszystkich modeli
├── models/mixins.py    # IdMixin, TimestampMixin, ActiveMixin, SoftDeleteMixin
├── models/catalog.py   # Category, Product
├── models/customer.py  # Customer
└── models/order.py     # Order, OrderItem
```

Wymagania:

- Żaden plik nie może importować drugiego w sposób tworzący cykl.
- `app/models/__init__.py` importuje wszystko i eksportuje przez `__all__`.
- `app/db/base.py` nie importuje żadnego modelu.
- Po refaktoryzacji `Base.metadata.tables` musi zawierać **dokładnie te same sześć tabel**, co przed nią.
- Napisz test, który to sprawdza: `assert set(Base.metadata.tables) == {...}`.

### Zadanie 3 — porównanie czterech stylów (trudne)

Zdefiniuj ten sam model `Article` na cztery sposoby i wypisz dla każdego DDL oraz sygnaturę `__init__`:

| Wariant | Konfiguracja |
|---|---|
| A | zwykły `DeclarativeBase`, bez `__init__` |
| B | `DeclarativeBase` + ręcznie napisany `__init__` |
| C | `MappedAsDataclass` bez `kw_only` |
| D | `MappedAsDataclass` z `kw_only=True` globalnie |

Model ma pola: `id` (PK, autoinkrementacja), `title: str`, `slug: str` (unikalny, indeksowany), `body: str | None`, `views: int` (domyślnie 0), `created_at`.

Dla każdego wariantu odpowiedz pisemnie:

1. Czy `Article(title="x", slug="x")` działa? A `Article("x", "x")`?
2. Czy `repr(article)` daje coś sensownego bez pisania `__repr__`?
3. Czy `article.views` jest `0`, czy `None`, tuż po utworzeniu obiektu?
4. Które warianty są odporne na przestawienie kolejności pól w klasie?

### Rozwiązania

#### Rozwiązanie zadania 1

```python
# examples/07_ex1_audit_mixin.py
from __future__ import annotations

import datetime
from typing import Any

from sqlalchemy import (
    JSON,
    CheckConstraint,
    DateTime,
    ForeignKey,
    Index,
    MetaData,
    Numeric,
    String,
    UniqueConstraint,
    create_engine,
    func,
)
from sqlalchemy.orm import (
    DeclarativeBase,
    Mapped,
    declared_attr,
    mapped_column,
    relationship,
)
from sqlalchemy.schema import CreateTable

NAMING_CONVENTION = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}


class Base(DeclarativeBase):
    metadata = MetaData(naming_convention=NAMING_CONVENTION)
    type_annotation_map = {
        dict[str, Any]: JSON,
        decimal.Decimal: Numeric(12, 2),
    }


class IdMixin:
    id: Mapped[int] = mapped_column(primary_key=True, sort_order=-100)


class ActiveMixin:
    is_active: Mapped[bool] = mapped_column(
        default=True, server_default="1", sort_order=101
    )


class AuditMixin:
    """Kto utworzył rekord. `sort_order=102` → po `is_active`."""

    @declared_attr
    def created_by_id(cls) -> Mapped[int | None]:
        return mapped_column(ForeignKey("customer.id"), sort_order=102)

    @declared_attr
    def created_by(cls) -> Mapped["Customer | None"]:
        return relationship("Customer", foreign_keys=[cls.created_by_id])


class Customer(IdMixin, Base):
    __tablename__ = "customer"

    email: Mapped[str] = mapped_column(String(255), unique=True)
    full_name: Mapped[str] = mapped_column(String(120))


class Product(IdMixin, ActiveMixin, AuditMixin, Base):
    __tablename__ = "product"

    sku: Mapped[str] = mapped_column(String(32))
    name: Mapped[str] = mapped_column(String(120))
    price: Mapped[decimal.Decimal]
    category_id: Mapped[int] = mapped_column(ForeignKey("category.id"))

    __table_args__ = (
        UniqueConstraint("sku", name="uq_product_sku"),
        CheckConstraint("price >= 0", name="ck_product_price_non_negative"),
        Index("ix_product_category_active", "category_id", "is_active"),
    )


engine = create_engine("sqlite:///:memory:")
print(CreateTable(Product.__table__).compile(engine))
```

Fragment DDL (kolejność kolumn wynika z `sort_order`):

```sql
CREATE TABLE product (
	id INTEGER NOT NULL, 
	sku VARCHAR(32) NOT NULL, 
	name VARCHAR(120) NOT NULL, 
	price NUMERIC(12, 2) NOT NULL, 
	category_id INTEGER NOT NULL, 
	is_active BOOLEAN NOT NULL, 
	created_by_id INTEGER, 
	PRIMARY KEY (id), 
	CONSTRAINT uq_product_sku UNIQUE (sku), 
	CONSTRAINT ck_product_price_non_negative CHECK (price >= 0), 
	CONSTRAINT fk_product_category_id_category FOREIGN KEY(category_id) REFERENCES category (id), 
	CONSTRAINT fk_product_created_by_id_customer FOREIGN KEY(created_by_id) REFERENCES customer (id)
)
```

**Dlaczego tak:** `@declared_attr` jest konieczne, bo `ForeignKey` nie może być współdzielony między tabelami — każda klasa musi dostać własny obiekt. `foreign_keys=[cls.created_by_id]` jest konieczne, bo `Customer` może być wskazywany przez więcej niż jedną kolumnę (np. `order.customer_id` i `product.created_by_id`), a SQLAlchemy nie zgadnie, o którą chodzi.

**Alternatywa:** można by użyć `Annotated` z `mapped_column(ForeignKey(...))` — to działa dla kolumny, ale **nie** dla relacji, więc mixin i tak potrzebowałby `@declared_attr` dla `created_by`. Dla spójności zostajemy przy wersji z `@declared_attr` w obu polach.

**Minipułapka:** gdyby `AuditMixin` stał **przed** `ActiveMixin` w liście klas bazowych, `created_by_id` i tak wylądowałoby po `is_active`, bo decyduje `sort_order`, nie kolejność dziedziczenia. Bez `sort_order` kolejność zależałaby od MRO — i byłaby nieprzewidywalna dla czytelnika.

#### Rozwiązanie zadania 2

```python
# app/db/base.py
from __future__ import annotations

import decimal
from typing import Any

from sqlalchemy import JSON, MetaData, Numeric, String
from sqlalchemy.orm import DeclarativeBase

NAMING_CONVENTION = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}


class Base(DeclarativeBase):
    metadata = MetaData(naming_convention=NAMING_CONVENTION)
    type_annotation_map = {
        dict[str, Any]: JSON,
        decimal.Decimal: Numeric(12, 2),
        str: String(255),
    }
```

```python
# app/models/mixins.py
from __future__ import annotations

import datetime

from sqlalchemy import DateTime, func
from sqlalchemy.orm import Mapped, mapped_column


class IdMixin:
    id: Mapped[int] = mapped_column(primary_key=True, sort_order=-100)


class TimestampMixin:
    created_at: Mapped[datetime.datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), sort_order=100
    )


class ActiveMixin:
    is_active: Mapped[bool] = mapped_column(
        default=True, server_default="1", sort_order=101
    )


class SoftDeleteMixin:
    deleted_at: Mapped[datetime.datetime | None] = mapped_column(
        DateTime(timezone=True), default=None, index=True, sort_order=102
    )

    @property
    def is_deleted(self) -> bool:
        return self.deleted_at is not None
```

```python
# app/models/catalog.py
from __future__ import annotations

import decimal
from typing import Any

from sqlalchemy import CheckConstraint, ForeignKey, Index, UniqueConstraint
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.db.base import Base
from app.models.mixins import ActiveMixin, IdMixin


class Category(IdMixin, Base):
    __tablename__ = "category"

    name: Mapped[str]
    parent_id: Mapped[int | None] = mapped_column(ForeignKey("category.id"))

    products: Mapped[list["Product"]] = relationship(back_populates="category")


class Product(IdMixin, ActiveMixin, Base):
    __tablename__ = "product"

    sku: Mapped[str]
    name: Mapped[str]
    price: Mapped[decimal.Decimal]
    category_id: Mapped[int] = mapped_column(ForeignKey("category.id"))

    category: Mapped["Category"] = relationship(back_populates="products")

    __table_args__ = (
        UniqueConstraint("sku", name="uq_product_sku"),
        CheckConstraint("price >= 0", name="ck_product_price_non_negative"),
        Index("ix_product_category_active", "category_id", "is_active"),
    )
```

```python
# app/models/customer.py
from __future__ import annotations

import datetime
from typing import Any

from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.db.base import Base
from app.models.mixins import ActiveMixin, IdMixin, SoftDeleteMixin, TimestampMixin


class Customer(IdMixin, TimestampMixin, ActiveMixin, SoftDeleteMixin, Base):
    __tablename__ = "customer"

    email: Mapped[str] = mapped_column(unique=True, index=True)
    full_name: Mapped[str]
    nickname: Mapped[str | None]
    preferences: Mapped[dict[str, Any]] = mapped_column(default=dict)

    orders: Mapped[list["Order"]] = relationship(back_populates="customer")

    @property
    def display_name(self) -> str:
        return self.nickname or self.full_name
```

```python
# app/models/order.py
from __future__ import annotations

import datetime
import decimal

from sqlalchemy import DateTime, ForeignKey, UniqueConstraint, func
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.db.base import Base
from app.models.mixins import IdMixin, TimestampMixin


class Order(IdMixin, TimestampMixin, Base):
    __tablename__ = "order"

    customer_id: Mapped[int] = mapped_column(ForeignKey("customer.id"))
    placed_at: Mapped[datetime.datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now()
    )
    status: Mapped[str] = mapped_column(default="new")

    customer: Mapped["Customer"] = relationship(back_populates="orders")
    items: Mapped[list["OrderItem"]] = relationship(back_populates="order")

    @property
    def total(self) -> decimal.Decimal:
        return sum((item.subtotal for item in self.items), decimal.Decimal("0"))


class OrderItem(IdMixin, Base):
    __tablename__ = "order_item"

    order_id: Mapped[int] = mapped_column(ForeignKey("order.id"))
    product_id: Mapped[int] = mapped_column(ForeignKey("product.id"))
    quantity: Mapped[int]
    unit_price: Mapped[decimal.Decimal]

    order: Mapped["Order"] = relationship(back_populates="items")
    product: Mapped["Product"] = relationship()

    __table_args__ = (
        UniqueConstraint(
            "order_id", "product_id", name="uq_order_item_order_product"
        ),
    )

    @property
    def subtotal(self) -> decimal.Decimal:
        return self.unit_price * self.quantity
```

```python
# app/models/__init__.py
"""Import wszystkich modeli — bez tego `Base.metadata` będzie niekompletne."""
from app.models.catalog import Category, Product
from app.models.customer import Customer
from app.models.order import Order, OrderItem

__all__ = ["Category", "Customer", "Order", "OrderItem", "Product"]
```

```python
# tests/test_schema_complete.py
from app.db.base import Base
from app.models import *  # noqa: F403  — wymusza rejestrację wszystkich modeli


def test_all_tables_registered() -> None:
    expected = {"category", "customer", "order", "order_item", "product"}
    assert set(Base.metadata.tables) == expected


def test_no_cycles_in_imports() -> None:
    """Prosty test: import pakietu nie rzuca ImportError ani cyklu."""
    import importlib

    for module in (
        "app.db.base",
        "app.models.mixins",
        "app.models.catalog",
        "app.models.customer",
        "app.models.order",
    ):
        importlib.import_module(module)
```

**Dlaczego tak:** cykl importów rozwiązujemy przez `from __future__ import annotations` (adnotacje stają się napisami) oraz `Mapped["Customer"]` w cudzysłowach (SQLAlchemy rozwiązuje napisy leniwie, po zarejestrowaniu wszystkich klas). Plik `order.py` nie importuje `customer.py` ani `catalog.py` wcale — wystarczy mu nazwa tabeli w `ForeignKey` i nazwa klasy w cudzysłowie.

**Alternatywa:** gdybyś chciał mieć pełne podpowiedzi typów w `order.py`, dodałbyś `if TYPE_CHECKING: from app.models.customer import Customer`. To nie tworzy cyklu w runtime, bo `TYPE_CHECKING` jest `False` poza sprawdzaniem typów.

**Minipułapka:** test `test_all_tables_registered` **musi** importować `app.models` (albo `app.models.__init__`). Jeżeli zaimportujesz tylko `app.db.base`, `Base.metadata` będzie puste i test przejdzie trywialnie dla pustego zbioru — a właściwie go oblituje. To dokładnie ten sam błąd, który w produkcji powoduje „brakującą tabelę” w `create_all`.

#### Rozwiązanie zadania 3

```python
# examples/07_ex3_styles.py
"""Cztery style definiowania tego samego modelu — porównanie."""
from __future__ import annotations

import datetime

from sqlalchemy import DateTime, func
from sqlalchemy.orm import DeclarativeBase, Mapped, MappedAsDataclass, mapped_column

# --------------------------------------------------------------------------
# Wariant A — zwykły DeclarativeBase, bez __init__
# --------------------------------------------------------------------------
class BaseA(DeclarativeBase):
    pass


class ArticleA(BaseA):
    __tablename__ = "article_a"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str]
    slug: Mapped[str] = mapped_column(unique=True, index=True)
    body: Mapped[str | None]
    views: Mapped[int] = mapped_column(default=0)
    created_at: Mapped[datetime.datetime] = mapped_column(server_default=func.now())


# --------------------------------------------------------------------------
# Wariant B — ręcznie napisany __init__
# --------------------------------------------------------------------------
class BaseB(DeclarativeBase):
    pass


class ArticleB(BaseB):
    __tablename__ = "article_b"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str]
    slug: Mapped[str] = mapped_column(unique=True, index=True)
    body: Mapped[str | None]
    views: Mapped[int] = mapped_column(default=0)
    created_at: Mapped[datetime.datetime] = mapped_column(server_default=func.now())

    def __init__(
        self,
        title: str,
        slug: str,
        body: str | None = None,
        views: int = 0,
    ) -> None:
        self.title = title
        self.slug = slug
        self.body = body
        self.views = views

    def __repr__(self) -> str:
        return f"ArticleB(title={self.title!r}, slug={self.slug!r})"


# --------------------------------------------------------------------------
# Wariant C — MappedAsDataclass bez kw_only
# --------------------------------------------------------------------------
class BaseC(MappedAsDataclass, DeclarativeBase):
    pass


class ArticleC(BaseC):
    __tablename__ = "article_c"

    id: Mapped[int] = mapped_column(primary_key=True, init=False)
    title: Mapped[str]                                  # brak domyślnej → pierwsze
    slug: Mapped[str] = mapped_column(unique=True, index=True)  # brak domyślnej → drugie
    body: Mapped[str | None] = mapped_column(default=None)
    views: Mapped[int] = mapped_column(default=0)
    created_at: Mapped[datetime.datetime] = mapped_column(
        server_default=func.now(), init=False
    )


# --------------------------------------------------------------------------
# Wariant D — MappedAsDataclass z kw_only=True
# --------------------------------------------------------------------------
class BaseD(MappedAsDataclass, DeclarativeBase, kw_only=True):
    pass


class ArticleD(BaseD):
    __tablename__ = "article_d"

    id: Mapped[int] = mapped_column(primary_key=True, init=False)
    title: Mapped[str]
    slug: Mapped[str] = mapped_column(unique=True, index=True)
    body: Mapped[str | None] = mapped_column(default=None)
    views: Mapped[int] = mapped_column(default=0)
    created_at: Mapped[datetime.datetime] = mapped_column(
        server_default=func.now(), init=False
    )
```

| Pytanie | A | B | C | D |
|---|---|---|---|---|
| `Article(title="x", slug="x")` | ❌ `TypeError` | ✅ | ✅ | ✅ |
| `Article("x", "x")` | ❌ | ✅ | ✅ | ❌ (`kw_only`) |
| sensowny `repr` bez pisania | ❌ (`<ArticleA object at 0x...>`) | ✅ (ręcznie) | ✅ (generowany) | ✅ (generowany) |
| `article.views` po utworzeniu | `None` (nieustawione!) | `0` | `0` | `0` |
| odporne na przestawienie pól | ✅ (nie ma `__init__`) | ✅ | ❌ | ✅ |

**Dlaczego tak:** w wariancie A `views` jest `None`, a nie `0`, bo `default=0` to **domyślna wartość dla kolumny**, nie dla atrybutu Pythona — zostanie użyta dopiero przy `INSERT`. To zachowanie jest spójne i celowe (punkt 3.3), ale zaskakuje początkujących. W wariantach C i D `default=0` działa **podwójnie**: jest wartością domyślną w konstruktorze dataclassy **oraz** domyślną wartością kolumny.

**Alternatywa:** wariant B (ręczny `__init__`) jest wciąż sensowny, gdy potrzebujesz walidacji lub przekształceń w konstruktorze — np. automatycznego generowania `slug` z `title`. Żaden z pozostałych wariantów tego nie zrobi, bo `MappedAsDataclass` generuje konstruktor, którego nie możesz nadpisać bez utraty wygody.

**Minipułapka:** w wariancie C **kolejność pól ma znaczenie** — `title` i `slug` muszą stać przed `body` i `views`, bo nie mają domyślnej wartości. Dodanie nowego wymaganego pola na końcu klasy natychmiast zepsuje `__init__` z `TypeError`. W wariancie D ten problem nie występuje, dlatego `kw_only=True` jest zalecane w projektach, które będą się rozwijać.

---

## Najczęstsze błędy i jak je czytać

| Komunikat / objaw | Przyczyna | Naprawa |
|---|---|---|
| `ArgumentError: Mapper ... could not assemble any primary key columns` | brak `mapped_column(primary_key=True)` na jakimkolwiek polu | dodaj klucz główny, np. `id: Mapped[int] = mapped_column(primary_key=True)` |
| `ArgumentError: Class <Customer> does not have a __table__ or __tablename__` | zapomniane `__tablename__` | dodaj `__tablename__ = "customer"` |
| `ArgumentError: Type annotation for "X.y" can't be correctly interpreted for Annotated Declarative Table form` | adnotacja bez `Mapped[...]` (np. pole pomocnicze lub relacja w starym stylu) | użyj `Mapped[...]`, `ClassVar[...]` lub (ostatecznie) `__allow_unmapped__ = True` |
| `ArgumentError: Could not locate SQLAlchemy Core type for Python type: <class 'dict'>` | brak wpisu dla `dict`/`list` w `type_annotation_map` | dodaj `dict[str, Any]: JSON` (albo `postgresql.JSONB`) do mapy typów |
| `NameError: Could not de-stringify annotation 'list[Book]'` | `from __future__ import annotations` + adnotacja bez cudzysłowów, której nie da się rozwiązać | użyj `Mapped[list["Book"]]` albo zaimportuj typ w runtime |
| `sqlalchemy.exc.InvalidRequestError: Table 'x' is already defined for this MetaData instance` | dwa modele z tą samą wartością `__tablename__` w jednej `MetaData` | zmień `__tablename__` albo użyj `extend_existing=True` (ostateczność) |
| `TypeError: non-default argument 'title' follows default argument` | `MappedAsDataclass` + pole bez domyślnej po polu z domyślną (często z mixinu) | `kw_only=True` na klasie bazowej albo `kw_only=True`/`init=False` na polu z mixinu |
| `TypeError: Customer() takes no arguments` | zwykły model deklaratywny nie generuje `__init__` | użyj `MappedAsDataclass` albo napisz `__init__` ręcznie, albo ustawiaj atrybuty po utworzeniu |
| `ArgumentError: Can't add additional column 'name' when using __table__` | klasa ma `__table__` i jednocześnie deklaruje kolumny | przenieś kolumny do `Table(...)` albo usuń `__table__` |
| `ArgumentError: Column object 'x' already assigned to Table 'y'` | ten sam obiekt `Column`/`mapped_column` użyty w dwóch klasach (klasyczny błąd przy mixinach bez `@declared_attr`) | owiń deklarację w `@declared_attr` albo użyj `use_existing_column=True` |
| `sqlalchemy.exc.NoForeignKeysError: Could not determine join condition` | relacja bez jednoznacznego klucza obcego (dwie kolumny wskazują tę samą tabelę) | dodaj `foreign_keys=[...]` do `relationship()` |
| `ImportError: cannot import name 'X' from partially initialized module` | cykl importów między plikami modeli | użyj `Mapped["X"]` w cudzysłowie, `TYPE_CHECKING` lub przenieś wspólne rzeczy do `db/base.py` |
| Brakująca tabela w `create_all` | moduł z modelem nie został zaimportowany | zaimportuj wszystkie modele w `models/__init__.py` |
| Kolumna jest `NULL` w bazie, choć wydawała się wymagana | w adnotacji jest `| None` | usuń `| None` z adnotacji (i sprawdź, czy logika nie wymaga `None`) |
| `alembic autogenerate` proponuje `DROP TABLE` dla istniejącej tabeli | model nie został zaimportowany przy generowaniu migracji | ustaw `target_metadata` na `Base.metadata` i importuj wszystkie modele w `env.py` |

---

## Słowniczek modułu

| Termin (EN) | Polski odpowiednik | Wyjaśnienie |
|---|---|---|
| Annotated Declarative | deklaratywne mapowanie z adnotacjami | styl 2.0, w którym typ kolumny i nullability wynikają z `Mapped[...]` |
| attribute instrumentation | instrumentacja atrybutów | podmiana zwykłych atrybutów klasy na deskryptory, które śledzą zmiany |
| `DeclarativeBase` | klasa bazowa deklaratywna | klasa, po której dziedziczą wszystkie modele; uruchamia proces mapowania |
| declarative mixin | mixin deklaratywny | klasa niebędąca modelem, której pola wpadają do klas dziedziczących |
| `@declared_attr` | atrybut deklarowany | dekorator dla pól mixinu, które muszą powstać osobno dla każdej klasy |
| eager / deferred loading | ładowanie natychmiastowe / odroczone | czy kolumna jest pobierana razem z obiektem, czy dopiero na żądanie |
| `insert_default` | domyślna wartość wstawiania | wartość używana w `INSERT`, niewidoczna w konstruktorze obiektu |
| instrumented attribute | atrybut instrumentowany | atrybut modelu obsługiwany przez SQLAlchemy, np. `Customer.name` |
| impedance mismatch | impedancja obiektowo-relacyjna | naturalny rozjazd między modelem obiektowym a relacyjnym |
| `Mapped[T]` | typ mapowany | kontener adnotacji: „ten atrybut jest kolumną typu `T`” |
| `MappedAsDataclass` | mapowany jako dataclass | klasa bazowa włączająca generowanie `__init__`, `__repr__`, `__eq__` |
| mapper | mapper | obiekt tłumaczący klasę Pythona na tabelę i z powrotem |
| metadata | metadane | zbiór definicji tabel; w module 03 poznałeś ją jako `MetaData` |
| naming convention | konwencja nazewnictwa | reguły nadawania nazw ograniczeniom i indeksom |
| nullability | dopuszczalność wartości NULL | czy kolumna może być pusta; w 2.0 wynika z `| None` |
| registry | rejestr | obiekt prowadzący rejestr mapperów i mapę typów |
| `server_default` | domyślna wartość serwera | wartość nadawana przez bazę przy `INSERT` |
| `server_onupdate` | aktualizacja po stronie serwera | informacja, że bazowa kolumna jest zmieniana przez trigger |
| `sort_order` | kolejność sortowania | parametr sterujący kolejnością kolumn w tabeli (2.0.4+) |
| `type_annotation_map` | mapa typów z adnotacji | słownik „typ Pythona → typ SQLAlchemy”, używany przy `Mapped[]` |
| Unit of Work | jednostka pracy | mechanizm zbierający zmiany i wysyłający je w jednej transakcji (moduł 08) |
| `__allow_unmapped__` | zezwól na niezmapowane | flaga wyłączająca błąd przy adnotacjach bez `Mapped[]` |
| `__table__` | tabela | gotowy obiekt `Table`, na który mapowana jest klasa |
| `__table_args__` | argumenty tabeli | krotka z ograniczeniami, indeksami i opcjami tabeli |

---

## Dalsze czytanie

Dokumentacja oficjalna (SQLAlchemy 2.0):

- **Mapping Classes with Declarative** — wprowadzenie do stylu deklaratywnego i `DeclarativeBase`: <https://docs.sqlalchemy.org/en/20/orm/declarative_mapping.html>
- **Declarative Mapping Styles** — porównanie stylów (klasa bazowa, dekorator, imperatywny): <https://docs.sqlalchemy.org/en/20/orm/declarative_styles.html>
- **Table Configuration with Declarative** — `mapped_column()`, `Annotated`, `type_annotation_map`, `__table_args__`, `__table__`: <https://docs.sqlalchemy.org/en/20/orm/declarative_tables.html>
- **Integration with dataclasses and attrs** — pełne omówienie `MappedAsDataclass`, `kw_only`, `default_factory`: <https://docs.sqlalchemy.org/en/20/orm/dataclasses.html>
- **Composing Mapped Hierarchies with Mixins** — mixiny i `@declared_attr` w praktyce: <https://docs.sqlalchemy.org/en/20/orm/mixins.html>
- **Class Mapping API** — pełna sygnatura `mapped_column()`, `registry`, `DeclarativeBase`: <https://docs.sqlalchemy.org/en/20/orm/mapping_api.html>

Migracje i nowości:

- **SQLAlchemy 2.0 — Major Migration Guide** — sekcja o `__allow_unmapped__` i przejściu ze stylu 1.x: <https://docs.sqlalchemy.org/en/20/changelog/migration_20.html>
- **What's New in SQLAlchemy 2.1** — m.in. zmiana zachowania domyślnych wartości w dataclassach: <https://docs.sqlalchemy.org/en/21/changelog/migration_21.html>

> 💡 **Jak czytać dokumentację SQLAlchemy** — to jedna z najlepiej napisanych dokumentacji w ekosystemie Pythona, ale ma stromy próg wejścia. Praktyczna rada: każda strona ma na górze przykłady, a dopiero niżej pełny opis parametrów. Czytaj przykłady, kopiuj do edytora i uruchamiaj. Dopiero gdy coś nie działa — schodź do opisu parametru.

---

## Co dalej

Masz teraz modele: klasy, które wiedzą, jak wyglądają w bazie i jak się zachowują w Pythonie. Ale model to tylko opis — nic jeszcze nie zostało zapisane ani odczytane. Do tego potrzebujesz obiektu, który **śledzi zmiany na obiektach, grupuje je w transakcję i wysyła do bazy w odpowiedniej kolejności**. Tym obiektem jest `Session` — i to jest temat [modułu 08](08_sesja_cykl_zycia.md).

W module 08 odpowiemy na pytania, które pojawiły się mimochodem tutaj: dlaczego `Product(name="Kubek")` nie trafia od razu do bazy, kiedy dokładnie powstaje `INSERT`, co znaczy, że obiekt jest „przywiązany do sesji”, i dlaczego dwa zapytania o tego samego klienta zwracają **ten sam obiekt Pythona**.

<!-- koniec modułu 07 -->