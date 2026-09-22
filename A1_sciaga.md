# Moduł A1 — Ściąga kursu

Jednostronicowa (no, dwunastostronicowa) referencja do całego kursu: importy, model, sesja, zapytania, zapis, relacje, ładowanie, typy, async, Alembic, testy, architektura, migracja z 1.x i pułapki. Do wydruku i do trzymania obok klawiatury. **To nie jest moduł do nauki od zera** — to skrót do materiału z modułów 01–24.

| | |
|---|---|
| **Poziom** | 🟡 średni (zakłada przeczytanie modułów 01–24) |
| **Czas** | ~30 min przeglądu, wraca się do niej wielokrotnie |
| **Wymagania wstępne** | [`00_README.md`](00_README.md), [`01_wprowadzenie.md`](01_wprowadzenie.md) |
| **Zakres** | SQLAlchemy 2.0.x (baseline); nowości 2.1 w ramkach 🆕 |
| **Uwaga** | Ściąga jest celowo zwarta — nie realizuje modułowego limitu objętości. Jej rolą jest gęstość informacji, nie narracja. |

---

## Spis treści

1. [Importy „na start”](#1-importy-na-start)
2. [Model w stylu 2.0](#2-model-w-stylu-20)
3. [Sesja — cykl życia](#3-sesja--cykl-życia)
4. [Zapytania — 15 wzorców `select()`](#4-zapytania--15-wzorców-select)
5. [Zapis danych](#5-zapis-danych)
6. [Relacje](#6-relacje)
7. [Eager loading](#7-eager-loading)
8. [Typy](#8-typy)
9. [Async](#9-async)
10. [Alembic](#10-alembic)
11. [Testy](#11-testy)
12. [Architektura](#12-architektura)
13. [API 1.x → 2.0](#13-api-1x--20)
14. [Pułapki](#14-pułapki)
- [Podsumowanie](#podsumowanie)
- [Ćwiczenia](#ćwiczenia)
- [Najczęstsze błędy i jak je czytać](#najczęstsze-błędy-i-jak-je-czytać)
- [Słowniczek modułu](#słowniczek-modułu)
- [Dalsze czytanie](#dalsze-czytanie)
- [Co dalej](#co-dalej)

---

## 1. Importy „na start”

### Core

```python
# examples/a1_imports_core.py
# Silnik, połączenia, pool
from sqlalchemy import create_engine, Engine, Connection, text
from sqlalchemy.pool import QueuePool, NullPool, StaticPool

# Schemat / DDL
from sqlalchemy import MetaData, Table, Column, Index, inspect
from sqlalchemy import (
    Integer, BigInteger, SmallInteger, String, Text, Boolean,
    Numeric, Float, DateTime, Date, Time, Interval,
    LargeBinary, JSON, Uuid, Enum as SAEnum,
)
from sqlalchemy import (
    ForeignKey, PrimaryKeyConstraint, UniqueConstraint,
    CheckConstraint, ForeignKeyConstraint,
)

# Zapytania / wyrażenia
from sqlalchemy import (
    select, insert, update, delete, union, union_all,
    intersect, except_, exists, case, cast, func, literal,
    literal_column, bindparam, and_, or_, not_, null, true, false,
)

# Typy własne
from sqlalchemy.types import TypeDecorator, UserDefinedType

# Zdarzenia i wyjątki
from sqlalchemy import event
from sqlalchemy.exc import IntegrityError, OperationalError, DBAPIError
```

### ORM

```python
# examples/a1_imports_orm.py
from sqlalchemy.orm import (
    DeclarativeBase, Mapped, mapped_column, mapped_as_dataclass,
    relationship, backref, Session, sessionmaker, scoped_session,
    aliased, selectinload, joinedload, subqueryload, immediateload,
    noload, raiseload, contains_eager, defaultload, defer, undefer,
    load_only, MappedAsDataclass, declared_attr, validates,
    column_property, synonym, attribute_mapped_collection,
    foreign, remote, reconstructor,
)
from sqlalchemy.orm import configure_mappers, object_session, inspect as orm_inspect

# Rozszerzenia
from sqlalchemy.ext.asyncio import (
    create_async_engine, async_sessionmaker, AsyncSession, AsyncAttrs,
)
from sqlalchemy.ext.hybrid import hybrid_property
from sqlalchemy.ext.associationproxy import association_proxy
from sqlalchemy.ext.mutable import MutableDict, MutableList, MutableSet
from sqlalchemy.ext.declarative import AbstractConcreteBase, ConcreteBase
from sqlalchemy.sql import Select   # pod typowanie Select[tuple[Book]]

# Dialekty (tylko gdy naprawdę potrzebne)
from sqlalchemy.dialects.postgresql import (
    insert as pg_insert, JSONB, ARRAY, UUID as PG_UUID, INET, CITEXT,
)
from sqlalchemy.dialects.sqlite import insert as sqlite_insert
```

> 🆕 **SQLAlchemy 2.1** — dla URL-a `postgresql://` domyślnym driverem jest **`psycopg`** (2.0 wybierał `psycopg2`), a dla `oracle://` — `oracledb`. Wymaga Pythona ≥ 3.11. `greenlet` nie instaluje się już automatycznie: do async potrzebujesz jawnie `pip install "sqlalchemy[asyncio]"`.

---

## 2. Model w stylu 2.0

```python
# examples/a1_model.py
from __future__ import annotations
from datetime import date, datetime
from sqlalchemy import ForeignKey, String, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    """Wspólna klasa bazowa: rejestr (registry) + MetaData."""
    pass


class Author(Base):
    __tablename__ = "author"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(120), index=True)
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())

    books: Mapped[list[Book]] = relationship(
        back_populates="author", cascade="all, delete-orphan"
    )


class Book(Base):
    __tablename__ = "book"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200), index=True)
    published: Mapped[date | None]                       # nullable=True
    price: Mapped[float]                                 # nullable=False
    author_id: Mapped[int] = mapped_column(
        ForeignKey("author.id", ondelete="CASCADE"), index=True
    )

    author: Mapped[Author] = relationship(back_populates="books")
```

### Reguły nullability w `Mapped[]`

| Adnotacja | `nullable` | Uwaga |
|---|---|---|
| `Mapped[str]` | `False` | **domyślnie NOT NULL** |
| `Mapped[str \| None]` | `True` | wymaga `Optional` w typie |
| `Mapped[list[X]]` | (dotyczy relacji) | kolekcja |
| `Mapped[X]` dla relacji | `False` | pojedyncza encja |
| `Mapped[X]` + `deferred=True` | — | kolumna ładowana na żądanie |

### Najczęstsze argumenty `mapped_column()`

| Argument | Znaczenie |
|---|---|
| `primary_key=True` | klucz główny |
| `ForeignKey("tabela.kolumna")` | klucz obcy (najlepiej z `ondelete=`) |
| `String(50)` / `Text` | VARCHAR / TEXT |
| `nullable=False` | zwykle zbędne — wynika z `Mapped[]` |
| `unique=True` / `index=True` | UNIQUE / CREATE INDEX |
| `default=...` | wartość nadawana przez **Pythona** przy INSERT |
| `server_default=...` | wartość nadawana przez **bazę** (DDL) |
| `onupdate=...` | Python przy UPDATE |
| `server_onupdate=...` | wymaga triggera po stronie bazy |
| `doc=...` / `comment=...` | dokumentacja / COMMENT ON |
| `deferred=True` | nie pobieraj domyślnie |
| `sort_order=...` | kolejność w `__init__` dataclass |

> ⚠️ **Pułapka** — `default=func.now()` da Ci *wyrażenie SQL* wstawiane przez Pythona; `server_default=func.now()` da `DEFAULT now()` w DDL. Dla kolumny „czas utworzenia” chcesz zwykle tego drugiego, bo działa też przy INSERT-ach z poza aplikacji.

---

## 3. Sesja — cykl życia

### Cykl w 6 wierszach

```python
# examples/a1_session_lifecycle.py
SessionLocal = sessionmaker(bind=engine, expire_on_commit=False)   # 1 fabryka (raz na aplikację)

with SessionLocal.begin() as session:          # 2 BEGIN + gwarantowany COMMIT/ROLLBACK i close()
    book = session.get(Book, 1)                # 3 odczyt → BEGIN otwarty (jeśli był brak)
    book.title = "Wydanie drugie"              # 4 zmiana TYLKO w pamięci (identity map)
    session.flush()                            # 5 UPDATE wysłany, obiekt nadal persistent
# 6 __exit__ → COMMIT (bez wyjątku) albo ROLLBACK (z wyjątkiem), potem close()
```

### Diagram

```text
Session lifecycle
────────────────────────────────────────────────────────────────
Session(bind=engine)                     obiekt w pamięci, brak transakcji
   │
   ├─ add(obj)          ──►  obj: transient ──► pending   (w kolejce INSERT)
   │
   ├─ execute(select)   ──►  autobegin: BEGIN; na PostgreSQL też SET ISOLATION
   │                         autoflush: najpierw wysyła pending
   │
   ├─ flush()           ──►  INSERT/UPDATE/DELETE; pending ──► persistent
   │                         identity map zyskuje klucze z RETURNING
   │
   ├─ commit()          ──►  COMMIT; przy expire_on_commit=True atrybuty wygaszone
   │                         (kolejny odczyt = nowe SELECT)
   │
   ├─ rollback()        ──►  ROLLBACK; obiekty wracają do stanu z bazy
   │
   └─ close()           ──►  połączenie wraca do puli; obiekty ──► detached
```

### Stany obiektu

| Stan | `inspect(obj).transient/…` | Kiedy |
|---|---|---|
| **transient** | `transient` | utworzony `Book(...)`, nigdy nie dodany |
| **pending** | `pending` | `session.add(obj)`, jeszcze bez `flush` |
| **persistent** | `persistent` | po `flush`/`commit`, jest w sesji, ma tożsamość w bazie |
| **deleted** | `deleted` | po `session.delete(obj)`, przed `flush` |
| **detached** | `detached` | sesja zamknięta / `expunge` — bez dostępu do leniwych relacji |

### Typowe operacje na sesji

| Cel | API |
|---|---|
| Otwórz/zamknij | `with SessionLocal() as s:` |
| Transakcja jawna | `with s.begin():` / `s.begin_nested()` |
| Odczyt po PK | `s.get(Book, 1)` / `s.get_one(Book, 1)` |
| Odczyt z JOIN-em | `s.execute(stmt).scalars().unique().all()` |
| Jedna wartość | `s.scalar(stmt)` / `s.scalars(stmt).one()` |
| Dodanie | `s.add(obj)` / `s.add_all([...])` |
| Ręczny flush | `s.flush()` |
| Odświeżenie z bazy | `s.refresh(obj, ["price"])` |
| Wygaszenie | `s.expire(obj)` / `s.expire_all()` |
| Wypięcie z sesji | `s.expunge(obj)` |
| Scal z inną sesją | `s.merge(obj)` |
| Uchwyt sesji obiektu | `object_session(obj)` |

> 🧠 **Dlaczego tak jest** — `commit()` domyślnie ustawia `expire_on_commit=True`. Po commicie każdy atrybut jest „wygaszony” i pierwszy dostęp uruchamia nowe `SELECT`. To daje świeże dane, ale kosztuje rundy do bazy i rodzi `DetachedInstanceError` poza sesją. Dlatego w aplikacjach webowych niemal zawsze ustawia się `expire_on_commit=False`.

---

## 4. Zapytania — 15 wzorców `select()`

Wspólne przygotowanie:

```python
# examples/a1_queries_setup.py
from sqlalchemy import select, func, exists, literal, case, cast, desc, asc
from sqlalchemy.orm import Session, joinedload

def show(stmt, dialect):  # 🔬 Pod maską — podgląd SQL-a
    print(stmt.compile(dialect=dialect, compile_kwargs={"literal_binds": True}))
```

| # | Cel | Kod |
|---|---|---|
| 1 | Wszystkie encje | `stmt = select(Book)` |
| 2 | Tylko kolumny (raport) | `stmt = select(Book.id, Book.title)` |
| 3 | Filtr równościowy | `stmt = select(Book).where(Book.price > 20)` |
| 4 | Filtr należenia | `stmt = select(Book).where(Book.author_id.in_([1, 2, 3]))` |
| 5 | Wzorzec tekstowy | `stmt = select(Book).where(Book.title.ilike("%python%"))` |
| 6 | Warunki złożone | `stmt = select(Book).where(and_(Book.price > 10, or_(Book.published.is_(None), Book.published >= date(2020, 1, 1))))` |
| 7 | Sortowanie | `stmt = select(Book).order_by(desc(Book.published), asc(Book.title))` |
| 8 | Paginacja offsetowa | `stmt = select(Book).order_by(Book.id).limit(20).offset(40)` |
| 9 | Jeden wiersz lub nic | `book = session.scalars(stmt).one_or_none()` |
| 10 | JOIN po relacji | `stmt = select(Book).join(Book.author).where(Author.name == "Lem")` |
| 11 | LEFT OUTER JOIN | `stmt = select(Book).outerjoin(Book.author)` |
| 12 | Agregacja + grupowanie | `stmt = select(Author.name, func.count(Book.id)).join(Book).group_by(Author.name).having(func.count(Book.id) > 2)` |
| 13 | Istnienie (anti-join) | `stmt = select(Author).where(~exists().where(Book.author_id == Author.id))` |
| 14 | `IN` z podzapytaniem | `sub = select(Book.author_id).where(Book.price > 100); stmt = select(Author).where(Author.id.in_(sub))` |
| 15 | Kolumna wyliczana + `case` | `stmt = select(Book.title, case((Book.price > 50, "droga"), else_="tania").label("kategoria"))` |

### Wykonanie i odbiór wyniku

```python
# examples/a1_queries_result.py
stmt = select(Book).where(Book.price > 20).order_by(Book.title)

rows = session.execute(stmt).scalars().all()          # list[Book]
one  = session.execute(stmt).scalars().first()        # Book | None
many = session.execute(select(Book.id, Book.title))   # Row(Book.id, Book.title)
for book_id, title in many:                           # rozpakowanie krotki
    ...

dicts = session.execute(select(Book.id, Book.title)).mappings().all()   # list[RowMapping]
count = session.scalar(select(func.count()).select_from(Book))
```

### Kiedy `.unique()`

| Sytuacja | `.unique()` |
|---|---|
| `joinedload` na relacji-kolekcji | **wymagane** |
| `joinedload` na relacji pojedynczej | nie |
| `selectinload` | nie |
| Zapytanie zwracające encje + JOIN do kolekcji | **wymagane** |
| Zapytanie zwracające kolumny | nie |

> 🔬 **Pod maską** — dla wzorca nr 3:

```sql
SELECT book.id, book.title, book.published, book.price, book.author_id
FROM book
WHERE book.price > %(price_1)s
```

`price_1` to **parametr wiązany** (`bindparam`). Wartość nigdy nie trafia do tekstu SQL-a — to jest mechanizm obrony przed SQL Injection i jednocześnie powód, dla którego SQLAlchemy może cache'ować skompilowane zapytania.

> ⚠️ **Pułapka** — `select(Book).join(Book.author)` bez `distinct()` zwróci duplikaty, gdy autor ma kilka pasujących pozycji i JOIN puchnie. Z kolei `Result` jest jednorazowy: po `.all()` nie „przewiniesz” go ponownie.

---

## 5. Zapis danych

| Operacja | Wzorzec |
|---|---|
| Dodanie encji | `session.add(book)` |
| Dodanie wielu encji | `session.add_all([b1, b2, b3])` |
| Wysłanie bez commita | `session.flush()` |
| Zakończenie transakcji | `session.commit()` / `session.rollback()` |
| Aktualizacja przez obiekt | `book.price = 39.99` (bez `save()`!) |
| Usunięcie | `session.delete(book)` |
| Bulk INSERT z dictów | `session.execute(insert(Book), [{"title": "A", "price": 10.0}, {...}])` |
| Bulk INSERT jednym VALUES | `session.execute(insert(Book), [{...}, {...}])` |
| INSERT z RETURNING | `ids = session.scalars(insert(Book).returning(Book.id)).all()` |
| ORM UPDATE (bulk) | `session.execute(update(Book).where(Book.price < 10).values(price=10))` |
| ORM DELETE (bulk) | `session.execute(delete(Book).where(Book.title == "X"))` |
| Upsert PostgreSQL | `from sqlalchemy.dialects.postgresql import insert; \n stmt = insert(Book).values(...).on_conflict_do_update(index_elements=[Book.isbn], set_={"price": ...})` |
| Upsert SQLite | `from sqlalchemy.dialects.sqlite import insert; stmt = insert(Book).values(...).on_conflict_do_nothing(index_elements=[Book.isbn])` |
| `INSERT ... SELECT` | `session.execute(insert(Archive).from_select(["title"], select(Book.title).where(...)))` |

### Surowy SQL — tylko przez `text()` z parametrami

```python
# examples/a1_raw_sql.py
from sqlalchemy import text

session.execute(
    text("UPDATE book SET price = :price WHERE id = :id"),
    [{"price": 19.90, "id": 1}, {"price": 24.50, "id": 2}],   # executemany
)
```

### Transakcje

```python
# examples/a1_transactions.py
with engine.begin() as conn:            # Core: BEGIN ... COMMIT
    conn.execute(text("INSERT INTO ..."))

with session.begin():                   # ORM: transakcja wokół bloku
    session.add(book)

with session.begin_nested():            # SAVEPOINT
    session.add(risky)
    # wyjątek tutaj wycofuje tylko savepoint
```

> 🔬 **Pod maską** — upsert na PostgreSQL (2.0):

```sql
INSERT INTO book (isbn, title, price) VALUES (%(isbn)s, %(title)s, %(price)s)
ON CONFLICT (isbn) DO UPDATE SET price = excluded.price
```

> ⚠️ **Pułapka** — `session.execute(update(...))` **nie aktualizuje obiektów w identity map**. Jeśli po bulk UPDATE sięgniesz po `session.get(Book, 1)`, możesz zobaczyć nieświeży obiekt. Foruj `synchronize_session="fetch"` (kosztuje dodatkowe SELECT-y) albo `session.expire_all()`.

---

## 6. Relacje

### Argumenty `relationship()`, których naprawdę użyjesz

| Argument | Do czego |
|---|---|
| `back_populates="…"` | dwustronne wiązanie (preferowane zamiast `backref`) |
| `backref="…"` | jednokierunkowe definiowanie drugiej strony |
| `secondary=Link.__table__` | tabela asocjacyjna N:M |
| `uselist=False` | wymuszenie 1:1 |
| `cascade="all, delete-orphan"` | usuwanie sierot |
| `passive_deletes=True` | oddaj kaskadę bazie (`ondelete="CASCADE"`) |
| `lazy="selectin"` | domyślna strategia ładowania inna niż `select` |
| `lazy="raise"` | zakaz leniwego ładowania (bezpiecznik) |
| `lazy="write_only"` | olbrzymie kolekcje tylko do zapisu |
| `order_by=…` | deterministyczna kolejność kolekcji |
| `foreign_keys=[…]` | gdy FK jest kilka (np. `author` i `editor`) |
| `primaryjoin=…`, `secondaryjoin=…` | niestandardowe warunki |
| `remote_side=[…]` | wymagane dla relacji samo-referencyjnej |
| `viewonly=True` | relacja tylko do odczytu |
| `innerjoin=True` | `joinedload` zawsze jako INNER JOIN |

### Gotowe wzory

**1:N — autor → książki**

```python
class Author(Base):
    __tablename__ = "author"
    id: Mapped[int] = mapped_column(primary_key=True)
    books: Mapped[list[Book]] = relationship(
        back_populates="author", cascade="all, delete-orphan"
    )

class Book(Base):
    __tablename__ = "book"
    id: Mapped[int] = mapped_column(primary_key=True)
    author_id: Mapped[int] = mapped_column(ForeignKey("author.id", ondelete="CASCADE"))
    author: Mapped[Author] = relationship(back_populates="books")
```

**1:1 — użytkownik ↔ profil**

```python
class Profile(Base):
    __tablename__ = "profile"
    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("user.id"), unique=True)
    user: Mapped["User"] = relationship(back_populates="profile")

class User(Base):
    __tablename__ = "user"
    id: Mapped[int] = mapped_column(primary_key=True)
    profile: Mapped["Profile | None"] = relationship(
        back_populates="user", uselist=False, cascade="all, delete-orphan"
    )
```

**N:M — książka ↔ tag**

```python
book_tag = Table(
    "book_tag", Base.metadata,
    Column("book_id", ForeignKey("book.id", ondelete="CASCADE"), primary_key=True),
    Column("tag_id", ForeignKey("tag.id", ondelete="CASCADE"), primary_key=True),
)

class Book(Base):
    ...
    tags: Mapped[list["Tag"]] = relationship(secondary=book_tag, back_populates="books")
```

**N:M z własnymi kolumnami (Association Object)** — gdy tabela pośrednia ma np. `added_at` lub `rating`, użyj **trzeciej encji** i połącz dwa 1:N zamiast `secondary`. Nigdy nie próbuj dodać kolumn do `secondary` — SQLAlchemy ich nie zmapuje.

**Samo-referencja — hierarchia kategorii**

```python
class Category(Base):
    __tablename__ = "category"
    id: Mapped[int] = mapped_column(primary_key=True)
    parent_id: Mapped[int | None] = mapped_column(ForeignKey("category.id"))
    children: Mapped[list["Category"]] = relationship(
        back_populates="parent", remote_side=[id]
    )
    parent: Mapped["Category | None"] = relationship(back_populates="children")
```

### Kaskady — ściąga

| Kaskada | Efekt w Pythonie |
|---|---|
| `save-update` | (domyślna) dodanie rodzica dodaje dzieci |
| `merge` | `merge` propaguje się na dzieci |
| `delete` | usunięcie rodzica usuwa dzieci (w ORM) |
| `delete-orphan` | usunięcie z kolekcji = DELETE dziecka |
| `refresh-expire` | `expire` propaguje się |
| `all` | `save-update, merge, refresh-expire, expunge, delete` |

> 🆕 **SQLAlchemy 2.1** — poprawiono mapowanie dataclass: domyślne wartości nie trafiają już do `__dict__` na instancji, co naprawiało mylące zachowania `MappedAsDataclass` przy `default_factory`.

---

## 7. Eager loading

| Przypadek | Opcja | Liczba zapytań | Uwagi |
|---|---|---|---|
| 1:N, jedno zapytanie, `unique()` wymagane | `joinedload(Author.books)` | **1** | LEFT OUTER JOIN; nie łącz z `yield_per` |
| Duża kolekcja, mniejsze ryzyko puchnięcia | `selectinload(Author.books)` | **1 + 1** | `WHERE author_id IN (...)` |
| Zagnieżdżone | `joinedload(A.bs).selectinload(B.cs)` | 1 + 1 | kompozycja na ścieżce |
| Wykorzystaj już istniejący JOIN | `contains_eager(Author.books)` | **1** | JOIN musi być w zapytaniu |
| Relacja pojedyncza, 1:1 | `joinedload(Book.author)` | 1 | najprostszy zysk |
| Zabronienie leniwego ładowania | `raiseload(Author.books)` | — | rzuca `InvalidRequestError` |
| „Nie ładuj wcale” | `noload(Author.books)` | 1 | atrybut pusty |
| Tylko wybrane kolumny | `load_only(Book.title)` | 1 | mniejszy transfer |
| Odroczenie kolumny | `defer(Book.description)` | 1 + na żądanie | przydatne przy LOB |
| Relacja tylko do zapisu | `lazy="write_only"` | — | dla olbrzymich kolekcji |

### Diagnostyka N+1

```python
# examples/a1_n_plus_one.py
from sqlalchemy import event

counter = {"n": 0}

@event.listens_for(engine, "before_cursor_execute")
def _count(conn, cursor, statement, params, context, executemany):
    counter["n"] += 1
    print(f"[{counter['n']:03d}] {statement.splitlines()[0]}")

counter["n"] = 0
books = session.scalars(select(Book)).all()
for b in books:
    _ = b.author.name          # leniwe ładowanie: 1 zapytanie na książkę → N+1
print(f"Zapytań: {counter['n']}")   # dla 100 książek: 101
```

> 🆕 **SQLAlchemy 2.1** — `selectinload` obsługuje parametr `chunksize` (porcjowanie `IN (...)` przy bardzo dużych zbiorach) oraz przywrócony `omit_join` dla relacji wiele-do-wiele. Sprawdź aktualną sygnaturę w dokumentacji 2.1 przed użyciem — nie zakładaj jej z pamięci.

> ⚠️ **Pułapka** — dwa `joinedload` na dwie **kolekcje** w jednym zapytaniu generują iloczyn kartezjański. Dla dwóch kolekcji użyj `selectinload`.

---

## 8. Typy

| Python | SQLAlchemy | SQLite | PostgreSQL |
|---|---|---|---|
| `int` | `Integer` | INTEGER | INTEGER |
| `int` (duże) | `BigInteger` | INTEGER | BIGINT |
| `str` (krótkie) | `String(255)` | VARCHAR(255) | VARCHAR(255) |
| `str` (długie) | `Text` | TEXT | TEXT |
| `bool` | `Boolean` | BOOLEAN (0/1) | BOOLEAN |
| `float` | `Float` | REAL | DOUBLE PRECISION |
| `Decimal` | `Numeric(12, 2)` | NUMERIC | NUMERIC ✅ natywnie |
| `datetime.datetime` | `DateTime(timezone=True)` | DATETIME (bez TZ!) | TIMESTAMPTZ |
| `datetime.date` | `Date` | DATE | DATE |
| `datetime.time` | `Time` | TIME | TIME |
| `datetime.timedelta` | `Interval` | (emulowane) | INTERVAL |
| `bytes` | `LargeBinary` | BLOB | BYTEA |
| `uuid.UUID` | `Uuid` | CHAR(32) | UUID (natywny) |
| `dict` / `list` | `JSON` | JSON (TEXT) | JSON |
| `enum.Enum` | `Enum(MyEnum)` | VARCHAR + CHECK | natywny TYPE |
| `set` | `ARRAY`/`JSON` | (przez JSON) | `ARRAY` / `JSONB` |

### `TypeDecorator` w 10 liniach

```python
# examples/a1_typedecorator.py
from sqlalchemy import String
from sqlalchemy.types import TypeDecorator


class LowercaseEmail(TypeDecorator[str]):
    """E-mail zawsze normalizowany do małych liter — po obu stronach."""
    impl = String(255)
    cache_ok = True                      # ← bez tego SQLAlchemy nie cache'uje SQL-a!

    def process_bind_param(self, value: str | None, dialect) -> str | None:
        return value.lower() if value is not None else None

    def process_result_value(self, value: str | None, dialect) -> str | None:
        return value.lower() if value is not None else None
```

### Mutowalny JSON

```python
# examples/a1_mutable_json.py
from sqlalchemy import JSON
from sqlalchemy.ext.mutable import MutableDict
from sqlalchemy.orm import Mapped, mapped_column

class User(Base):
    __tablename__ = "user"
    id: Mapped[int] = mapped_column(primary_key=True)
    prefs: Mapped[dict] = mapped_column(MutableDict.as_mutable(JSON), default=dict)

user.prefs["theme"] = "dark"   # z MutableDict zostanie wykryte i zapisane
```

> ⚠️ **Pułapka** — bez `MutableDict` zmiana klucza w słowniku **nie zostanie zapisana**: SQLAlchemy śledzi *przypisanie atrybutu*, nie mutację obiektu w miejscu. To samo dotyczy list w `JSON` (`MutableList`).

> 🆕 **SQLAlchemy 2.1** — wsparcie dla t-stringów z Pythona 3.14 oraz nowe konstrukcje `CreateView` i `CREATE TABLE AS SELECT`.

---

## 9. Async

```python
# examples/a1_async.py
import asyncio
from collections.abc import AsyncIterator
from sqlalchemy import select
from sqlalchemy.ext.asyncio import (
    AsyncSession, async_sessionmaker, create_async_engine,
)

engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/db",
    echo=False,
    pool_size=10,
    max_overflow=20,
    pool_pre_ping=True,
)
SessionLocal = async_sessionmaker(engine, expire_on_commit=False)


async def get_books() -> list[Book]:
    async with SessionLocal() as session:
        result = await session.execute(
            select(Book).options(selectinload(Book.author)).order_by(Book.title)
        )
        return list(result.scalars().all())


async def main() -> None:
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    books = await get_books()
    print(len(books))
    await engine.dispose()


if __name__ == "__main__":
    asyncio.run(main())
```

| Wzorzec | Kod |
|---|---|
| Sesja jako dependency | `async def dep() -> AsyncIterator[AsyncSession]: async with SessionLocal() as s: yield s` |
| DDL / sync API | `await conn.run_sync(Base.metadata.create_all)` |
| Strumieniowanie | `async for row in (await session.stream(stmt)).scalars(): ...` |
| Relacje leniwe | `class Base(AsyncAttrs, DeclarativeBase): ...` → `await obj.awaitable_attrs.books` |
| Bezpiecznik | `lazy="raise"` w modelach (async nie toleruje leniwego I/O) |

> ⚠️ **Pułapka** — w trybie async **leniwe ładowanie nie działa** poza kontekstem greenlet. Dostaniesz `MissingGreenlet`. Rozwiązania: (a) jawny eager loading, (b) `AsyncAttrs` + `awaitable_attrs`, (c) `lazy="raise"` jako domyślna polityka w projekcie. Nigdy nie ustawiaj `expire_on_commit=True` w async — każdy po-commitowy dostęp do atrybutu wywoła I/O.

> 🆕 **SQLAlchemy 2.1** — `greenlet` nie jest już instalowany automatycznie razem z pakietem. Dla async potrzebujesz `pip install "sqlalchemy[asyncio]"`. Zmiana dotyczy minimalnych instalacji i obrazów Docker.

---

## 10. Alembic

### Osiem poleceń

| Polecenie | Efekt |
|---|---|
| `alembic init alembic` | tworzy strukturę katalogu `alembic/` i `alembic.ini` |
| `alembic revision -m "opis"` | ręczny plik migracji (pusty szkielet) |
| `alembic revision --autogenerate -m "opis"` | porównuje modele z bazą i generuje migrację |
| `alembic upgrade head` | wdraża wszystkie brakujące wersje |
| `alembic upgrade +1` / `downgrade -1` | krok w przód / w tył o jedną wersję |
| `alembic current` | pokazuje bieżącą wersję w bazie |
| `alembic history --verbose` | lista wersji i relacji przodków |
| `alembic stamp head` | znaczy bazę jako „na `head`” bez wykonywania migracji |
| `alembic heads` / `alembic merge -m "…" heads` | obsługa rozgałęzień |

### Sześć operacji `op.*`

```python
# alembic/versions/xxxx_add_isbn.py
def upgrade() -> None:
    op.add_column("book", sa.Column("isbn", sa.String(13), nullable=True))
    op.alter_column("book", "price", existing_type=sa.Float(), nullable=False)
    op.create_index("ix_book_isbn", "book", ["isbn"], unique=True)
    op.create_unique_constraint("uq_book_isbn", "book", ["isbn"])
    op.drop_column("book", "legacy_code")

def downgrade() -> None:
    op.add_column("book", sa.Column("legacy_code", sa.String(20)))
    op.execute("UPDATE book SET legacy_code = ''")
    op.drop_constraint("uq_book_isbn", "book", type_="unique")
    op.drop_index("ix_book_isbn", table_name="book")
    op.alter_column("book", "price", existing_type=sa.Float(), nullable=True)
    op.drop_column("book", "isbn")
```

| Operacja | Uwaga |
|---|---|
| `op.create_table(...)` | pełny DDL; przy autogenerate sprawdź typy |
| `op.add_column(...)` | dla dużej tabeli: najpierw `nullable=True`, dane, potem `NOT NULL` |
| `op.alter_column(...)` | **na SQLite wymaga `batch_alter_table`** |
| `op.drop_column(...)` | nieodwracalna utrata danych przy złym `downgrade` |
| `op.create_index(...)` | na produkcji PostgreSQL `postgresql_concurrently=True` |
| `op.bulk_insert(...)` / `op.execute(...)` | migracje danych i surowy SQL |
| `op.batch_alter_table(...)` | emulacja ALTER na SQLite |

### `env.py` — trzy linie, które musisz mieć

```python
# alembic/env.py (fragmenty)
config.set_main_option("sqlalchemy.url", os.environ["DATABASE_URL"])
from myapp.models import Base
target_metadata = Base.metadata

context.configure(
    connection=connection,
    target_metadata=target_metadata,
    compare_type=True,          # wykrywaj zmiany typów
    render_as_batch=True,       # ← obowiązkowe dla SQLite
)
```

> ⚠️ **Pułapka** — bez `naming_convention` w `MetaData` autogenerate generuje przy każdej zmianie nazwy ograniczeń `drop_constraint` + `create_constraint`, co na produkcji bywa destrukcyjne. Ustaw konwencję raz, przed pierwszą migracją.

> 🆕 **SQLAlchemy 2.1 / Alembic 1.19** — `autogenerate` wykrywa nazwane ograniczenia `CHECK`, co wcześniej wymagało ręcznego dopisywania. Nadal **nie** wykrywa zmian w wartościach `Enum`.

---

## 11. Testy

```python
# tests/conftest.py
from collections.abc import Iterator
import pytest
from sqlalchemy import Engine, create_engine
from sqlalchemy.orm import Session
from sqlalchemy.pool import StaticPool
from myapp.models import Base


@pytest.fixture(scope="session")
def engine() -> Engine:
    eng = create_engine(
        "sqlite://",                                  # :memory:
        connect_args={"check_same_thread": False},    # sesja testowa w innym wątku
        poolclass=StaticPool,                         # ← JEDNO połączenie = jedna baza
    )
    Base.metadata.create_all(eng)
    return eng


@pytest.fixture
def session(engine: Engine) -> Iterator[Session]:
    """Test w transakcji zewnętrznej: rollback po każdym teście."""
    connection = engine.connect()
    transaction = connection.begin()
    session = Session(bind=connection, join_transaction_mode="create_savepoint")
    try:
        yield session
    finally:
        session.close()
        transaction.rollback()
        connection.close()
```

| Strategia izolacji | Koszt | Kiedy |
|---|---|---|
| `create_all` / `drop_all` per test | wysoki | małe schematy |
| Transakcja zewnętrzna + rollback | **niski** | domyślna rekomendacja |
| `TRUNCATE` między testami | średni | gdy testy muszą commitować |
| Migracje Alembic per test | najwyższy | testy samych migracji |

| Pułapka w testach | Naprawa |
|---|---|
| `sqlite://` bez `StaticPool` | każde połączenie tworzy **nową, pustą** bazę |
| Dzielona sesja między testami | fixture per test (`yield`) |
| `commit()` w teście psuje izolację | `begin_nested` + savepoint |
| Testy zależne od kolejności | deterministyczne dane, `freezegun`, stałe UUID |
| Async fixtury i pętla zdarzeń | `pytest-asyncio`, `loop_scope="session"` |

> 🧠 **Dlaczego w testach nie mockujemy `Session`** — mock zwraca to, co mu każesz zwrócić, więc test nie sprawdza zachowania SQLAlchemy ani zapytań. Testuj na prawdziwej bazie (SQLite `:memory:` albo PostgreSQL w kontenerze), a mockuj wyłącznie to, co wychodzi na zewnątrz: płatności, e-mail, HTTP.

---

## 12. Architektura

```text
┌──────────────────────────────────────────────────────────────┐
│  Prezentacja    API / CLI / UI         (nie zna SQLAlchemy)  │
├──────────────────────────────────────────────────────────────┤
│  Aplikacja      use cases, serwisy     (granica transakcji)  │
├──────────────────────────────────────────────────────────────┤
│  Domena         reguły biznesowe       (czysty Python)       │
├──────────────────────────────────────────────────────────────┤
│  Repozytoria    Session, select(), UoW (tu dzieje się SQL)   │
├──────────────────────────────────────────────────────────────┤
│  Infrastruktura Engine, modele ORM, Alembic (SQLAlchemy)     │
├──────────────────────────────────────────────────────────────┤
│  Baza danych    PostgreSQL / SQLite    (fizyczne dane)       │
└──────────────────────────────────────────────────────────────┘
```

| Co | Gdzie to umieścić | Czego tam NIE umieszczać |
|---|---|---|
| `select()` / `update()` | repozytorium | w endpoincie API, w modelu domenowym |
| `commit()` / `rollback()` | Unit of Work / serwis aplikacyjny | w repozytorium |
| `session.add()` | serwis / UoW (przez repozytorium) | w widoku / handlerze HTTP |
| Reguły biznesowe | domena lub serwis | w `@validates` (to walidacja formatu) |
| Mapowanie encja → JSON | Pydantic schema | w modelu ORM |
| `create_engine` | raz na proces | w funkcji obsługującej request |
| Migracje | Alembic | `create_all()` na produkcji |

### Szkielet `UnitOfWork`

```python
# examples/a1_uow.py
from contextlib import AbstractContextManager
from sqlalchemy.orm import Session, sessionmaker

class UnitOfWork(AbstractContextManager["UnitOfWork"]):
    def __init__(self, session_factory: sessionmaker[Session]) -> None:
        self._factory = session_factory
        self.session: Session | None = None

    def __enter__(self) -> "UnitOfWork":
        self.session = self._factory()
        return self

    def __exit__(self, exc_type, exc, tb) -> None:
        assert self.session is not None
        try:
            if exc_type is None:
                self.session.commit()
            else:
                self.session.rollback()
        finally:
            self.session.close()
```

> 💡 **Analogia** — warstwy to kuchnia (domena), magazyn (baza) i kelner (API). Kelner nie chodzi do magazynu po ziemniaki — mówi kuchni „stolik 4, danie dnia”, a kuchnia sięga do magazynu przez repozytorium.

---

## 13. API 1.x → 2.0

| 1.x (przestarzałe) | 2.0+ |
|---|---|
| `declarative_base()` | `class Base(DeclarativeBase): ...` |
| `Column(Integer, primary_key=True)` w modelu | `id: Mapped[int] = mapped_column(primary_key=True)` |
| `session.query(Book).filter(...)` | `session.scalars(select(Book).where(...))` |
| `Query.get(Book, 1)` | `session.get(Book, 1)` |
| `Query.filter_by(...)` | `select(Book).where(Book.title == ...)` |
| `Query.count()` | `session.scalar(select(func.count()).select_from(Book))` |
| `engine.execute(...)` | `with engine.begin() as conn: conn.execute(...)` |
| `Session(autocommit=True)` | `engine.begin()` / `Session.begin()` |
| `session.bulk_save_objects(...)` | `session.execute(insert(Book), [dicts...])` |
| `relationship(lazy="dynamic")` | `lazy="write_only"` lub osobne zapytanie |
| `backref="books"` | `back_populates="books"` (jawnie z obu stron) |
| `sqlalchemy.ext.declarative.declarative_base` | `sqlalchemy.orm.DeclarativeBase` |
| `create_engine(..., future=True)` | zbędne — 2.0 to domyślny styl |
| `Query.join(..., aliased=True)` | `aliased()` jawnie |
| Obiekty `Row` jak krotka | `Row`/`RowMapping` w stylu 2.0, `_mapping` |
| `Session.execute(select(...)).fetchall()` | `.scalars().all()` / `.all()` |
| `select([Book])` (lista) | `select(Book)` (argumenty pozycyjne) |
| `join(Book.author)` niejawne | jawne `join(Book.author)` z `onclause` przy potrzebie |
| `Query.all()` zwraca encje przy joinie | wymaga `unique()` przy `joinedload` kolekcji |
| `Session.bind` ustawiany dynamicznie | `sessionmaker(bind=...)` / `Session(bind=...)` |

> 🆕 **SQLAlchemy 2.1** — ulepszone typowanie PEP 646 sprawia, że `Result[tuple[Book]]` i `Row[tuple[int, str]]` lepiej współpracują z type checkerami; dzięki temu `mypy`/`pyright` łapią więcej błędów bez rzutowań.

---

## 14. Pułapki

1. `Mapped[str]` oznacza `NOT NULL`; jeśli kolumna ma być opcjonalna, musisz napisać `Mapped[str | None]`.
2. `flush()` ≠ `commit()` — `flush` wysyła SQL, ale transakcja pozostaje otwarta i można ją wycofać.
3. Modyfikacja obiektu poza sesją nie zostanie wykryta — brak sesji = brak `unit of work`.
4. Zmiana klucza w słowniku `JSON` bez `MutableDict` **nie zostanie zapisana**.
5. `joinedload` na kolekcji bez `.unique()` wywali się komunikatem o duplikatach z powodu identity map.
6. Dwa `joinedload` na dwóch kolekcjach naraz dają wybuch kartezjański — użyj `selectinload`.
7. `session.execute(update(...))` nie odświeża obiektów w identity map — pamiętaj o `synchronize_session`.
8. `detached instance` po `commit` z `expire_on_commit=True` jest normalnym zachowaniem, nie błędem.
9. Testy na `sqlite://:memory:` **bez** `StaticPool` używają innej, pustej bazy na każde połączenie.
10. `create_all()` na produkcji i Alembic w tym samym projekcie = rozjazd stanu schematu.
11. `ondelete="CASCADE"` w `ForeignKey` to kaskada **bazy**; `cascade="all, delete-orphan"` to kaskada **SQLAlchemy** — to nie to samo.
12. `default=func.now()` to wartość nadawana przez aplikację; `server_default=func.now()` to `DEFAULT now()` w DDL.
13. `TypeDecorator` bez `cache_ok = True` wyłącza cache kompilacji i znacząco spowalnia powtarzalne zapytania.
14. Async nie znosi leniwego ładowania — każdy dostęp do niezaładowanej relacji skończy się `MissingGreenlet`.
15. `text()` z f-stringiem to SQL Injection; jedyne poprawne podstawianie wartości to parametry wiązane (`:name`).

---

## Podsumowanie

- **Model w stylu 2.0** to `DeclarativeBase` + `Mapped[]` + `mapped_column()`; nullability wynika z adnotacji, nie z `nullable=`.
- **Sesja** to jednocześnie identity map, unit of work i granica transakcji — trzy role w jednym obiekcie.
- **`select()`** obsługuje identycznie Core i ORM; `session.execute()` plus `scalars()`/`scalar()`/`mappings()` pokrywa praktycznie wszystkie przypadki.
- **Zawsze pokazuj SQL** przy nauce i debugowaniu (`echo=True`, `stmt.compile()`); ORM bez zrozumienia emisji SQL to lot w chmurach.
- **N+1 to najczęstsza przyczyna „wolnego ORM”**; diagnostyka przez licznik `before_cursor_execute` zajmuje pięć minut i oszczędza dni.
- **Eager loading dobiera się do przypadku**: `joinedload` dla relacji pojedynczych, `selectinload` dla kolekcji, `contains_eager` gdy JOIN już masz.
- **Async zmienia reguły**: brak leniwego I/O, `expire_on_commit=False`, jawny eager loading, `pip install "sqlalchemy[asyncio]"`.
- **Migracje to kod produkcyjny** — z przeglądem autogenerate, zawsze napisanym `downgrade`, konwencją nazw ograniczeń i `render_as_batch` dla SQLite.
- **Warstwy mają znaczenie**: repozytorium wie *jak* zdobyć dane, UoW wie *kiedy* zatwierdzić, domena wie *po co*.
- **Kolejność optymalizacji**: zmierz → policz zapytania → zmniejsz transfer → dodaj indeks. Nigdy odwrotnie.

---

## Ćwiczenia

1. **Rozgrzewka (5 min).** Otwórz własny plik z modelem z modułu 07 i sprawdź, ile kolumn faktycznie ma `nullable=True`. Wypisz każdą kolumnę z `Mapped[X | None]` i oceń, czy to zamierzone.
2. **Diagnostyka (20 min).** Weź dowolny skrypt ładujący listę encji i uruchom go z licznikiem `before_cursor_execute`. Zapisz liczbę zapytań dla 100 rekordów, dodaj `selectinload` i porównaj. Zapisz oba wyniki w komentarzu w kodzie.
3. **Architektura (30 min).** Dla jednego przypadku użycia z modułu 22 napisz, w której warstwie umieściłbyś: (a) warunek „klient nie może mieć dwóch aktywnych wypożyczeń”, (b) `commit`, (c) `select()` ze złączeniem trzech tabel, (d) serializację do JSON. Odpowiedź: domena/serwis, UoW/serwis, repozytorium, warstwa prezentacji.

### Rozwiązania

**1.** Kolumna z `Mapped[str | None]` zawsze ma `nullable=True`; kolumna z `Mapped[str]` — `nullable=False`. Typowa pomyłka to `Mapped[str | None]` wszędzie „na wszelki wypadek”, co owocuje `NULL`-ami w bazie i kodem z `if x is not None` rozsianym po całym projekcie. Zasada: domyślnie `Mapped[str]`, `| None` tylko gdy naprawdę dopuszczasz brak wartości. Weryfikacja w locie:

```python
for col in Book.__table__.columns:
    print(col.name, col.type, "nullable=", col.nullable)
```

**2.** Referencyjna implementacja licznika:

```python
from sqlalchemy import event

counter = {"n": 0}

@event.listens_for(engine, "before_cursor_execute")
def _count(conn, cursor, stmt, params, ctx, executemany):
    counter["n"] += 1
```

Następnie:

```python
counter["n"] = 0
books = session.scalars(select(Book)).all()
for b in books:
    _ = b.author.name
print("bez eager:", counter["n"])           # 1 + N

counter["n"] = 0
books = session.scalars(select(Book).options(selectinload(Book.author))).all()
for b in books:
    _ = b.author.name
print("selectinload:", counter["n"])        # 2
```

**3.** (a) domena lub serwis aplikacyjny — to reguła, nie technologia; (b) Unit of Work — jedna granica dla całego przypadku użycia, nie w repozytorium; (c) repozytorium — to jest miejsce, gdzie wolno znać SQLAlchemy; (d) warstwa prezentacji (schemat Pydantic) — nigdy nie wypuszczaj encji ORM poza aplikację, bo niosą leniwe relacje i wyciekają wewnętrzne pola.

> ⚠️ **Pułapka przy rozwiązywaniu ćwiczenia 3** — jeśli w komentarzu napiszesz „commit w repozytorium”, to znaczy, że masz repozytorium odpowiedzialne za transakcję, a wtedy dwie operacje w jednym przypadku użycia nie są atomowe. Commit należy do UoW.

---

## Najczęstsze błędy i jak je czytać

| Komunikat | Przyczyna | Naprawa |
|---|---|---|
| `DetachedInstanceError: Instance <Book> is not bound to a Session` | dostęp do leniwej relacji po `close()`/`commit()` z `expire_on_commit=True` | eager loading, DTO, `expire_on_commit=False`, `session.refresh()` |
| `MissingGreenlet: greenlet_spawn has not been called` | leniwe I/O w trybie async | `selectinload`/`joinedload`, `AsyncAttrs`, `lazy="raise"` |
| `InvalidRequestError: The unique() method must be invoked` | `joinedload` na kolekcji bez `.unique()` | `session.execute(stmt).scalars().unique().all()` |
| `sqlalchemy.exc.PendingRollbackError` | wcześniejszy wyjątek bez `rollback()` | `rollback()` w `except`, użyj `with session.begin()` |
| `This session is in 'prepared' state` | użycie sesji po błędzie w transakcji 2PC | zamknij i otwórz nową sesję |
| `IntegrityError: UNIQUE constraint failed: book.isbn` | naruszenie `unique`/`unique constraint` | łap `IntegrityError`, mapuj na błąd aplikacyjny, użyj upsertu |
| `IntegrityError: FOREIGN KEY constraint failed` | wstawiasz dziecko przed rodzicem albo kasujesz rodzica z dziećmi | `ondelete="CASCADE"` w FK lub kolejność operacji |
| `NoResultFound` / `MultipleResultsFound` | `.one()` na 0 lub >1 wierszach | rozstrzygnij: `one_or_none()`, `first()`, albo popraw filtr |
| `OperationalError: no such table: book` | `create_all()` nie widzi modeli (import kolejności) albo zła baza (`:memory:`) | zaimportuj moduły modeli, `StaticPool` w testach |
| `StatementError: (builtins.TypeError) ... not a datetime` | `datetime` naiwny vs świadomy strefy | `DateTime(timezone=True)` + UTC zawsze |
| `ArgumentError: Mapper ... could not assemble any primary key` | brak `primary_key=True` lub brak FK w N:M | dodaj PK do tabeli asocjacyjnej |
| `sqlalchemy.exc.CompileError: Dialect ... does not support` | użycie `JSONB`, `ARRAY`, `LATERAL` na SQLite | `with_variant` albo testy na PostgreSQL |
| `InvalidRequestError: Can't use FOR UPDATE clause with eager loading` | `with_for_update` z `joinedload` | użyj `selectinload` albo rozbij zapytanie |
| `OperationalError: too many clients` / `pool_size exceeded` | wyciek połączeń — sesja bez `close()` | `with session:` / `try/finally`, `pool_pre_ping=True` |

---

## Słowniczek modułu

| Termin (EN) | PL | Wyjaśnienie |
|---|---|---|
| session | sesja | obszar pracy: identity map + kolejka zmian + transakcja |
| identity map | mapa tożsamości | rejestr obiektów w sesji kluczowany po `(klasa, PK)` |
| unit of work | jednostka pracy | wzorzec: wszystkie zmiany trafiają do bazy razem |
| flush | wypchnięcie | wysłanie SQL do bazy bez kończenia transakcji |
| expire | wygaszenie | oznaczenie atrybutów jako wymagających odczytu z bazy |
| detached | odłączony | obiekt poza sesją — brak dostępu do leniwych relacji |
| eager loading | ładowanie zachłanne | pobierz relacje już teraz (1–2 zapytania) |
| lazy loading | ładowanie leniwe | pobierz relację przy pierwszym dostępie |
| N+1 problem | problem N+1 | `1 + N` zapytań zamiast jednego |
| cascade | kaskada | propagacja operacji z rodzica na dzieci |
| secondary | tabela asocjacyjna | tabela łącząca w relacji N:M |
| back_populates | wsteczne wypełnianie | jawna dwustronna synchronizacja relacji |
| upsert | wstaw-lub-aktualizuj | `INSERT ... ON CONFLICT` |
| RETURNING | zwracanie wierszy | `INSERT/UPDATE ... RETURNING` — wynik operacji zapisu |
| bulk insert | wstawianie masowe | wiele wierszy w jednym zapytaniu |
| bindparam | parametr wiązany | bezpieczna wartość podstawiana jako `:name` |
| DDL / DML | — | definicja schematu / operacje na danych |
| savepoint | punkt zapisu | zagnieżdżona transakcja możliwa do wycofania |
| optimistic lock | blokada optymistyczna | kolumna wersji wykrywa konflikt zapisu |
| TypeDecorator | dekorator typu | własna konwersja Python ↔ SQL |
| hybrid_property | właściwość hybrydowa | jedna logika dla Pythona i dla SQL |
| association_proxy | proxy asocjacji | skrót dostępu przez tabelę pośrednią |
| repository | repozytorium | warstwa zdobywania danych przez API domeny |
| Unit of Work | jednostka pracy | granica transakcji dla przypadku użycia |
| DTO | obiekt transferu danych | obiekt wyniku bez zależności od ORM |
| Data Mapper | mapowanie danych | wzorzec: obiekt nie wie o bazie, mapa tłumaczy |
| autogenerate | autogenerowanie | Alembic porównuje modele z bazą i pisze migrację |

---

## Dalsze czytanie

- SQLAlchemy — ORM Quick Start (2.0): <https://docs.sqlalchemy.org/en/20/orm/quickstart.html>
- SQLAlchemy — ORM Querying Guide: <https://docs.sqlalchemy.org/en/20/orm/queryguide/index.html>
- SQLAlchemy — Relationship Configuration: <https://docs.sqlalchemy.org/en/20/orm/relationships.html>
- SQLAlchemy — Relationship Loading Techniques: <https://docs.sqlalchemy.org/en/20/orm/queryguide/relationships.html>
- SQLAlchemy — Session Basics: <https://docs.sqlalchemy.org/en/20/orm/session_basics.html>
- SQLAlchemy — Asyncio Extension: <https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html>
- SQLAlchemy — Core Tutorial: <https://docs.sqlalchemy.org/en/20/tutorial/index.html>
- SQLAlchemy — What's New in 2.1: <https://docs.sqlalchemy.org/en/21/changelog/whatsnew_21.html>
- Alembic — Tutorial: <https://alembic.sqlalchemy.org/en/latest/tutorial.html>
- Alembic — Autogenerating Migrations: <https://alembic.sqlalchemy.org/en/latest/autogenerate.html>

---

## Co dalej

Ściąga domyka kurs: moduły uczyły *dlaczego* i *kiedy*, ten plik pomaga *jak* w trybie natychmiastowym. Wracaj do niej po każdym dłuższym projekcie, żeby odświeżyć mapę API, a różnice wersji śledź w module [`A4_migracja_i_nowosci.md`](A4_migracja_i_nowosci.md). Jeśli szukasz zasobów do dalszej nauki, przejdź do [`A5_zasoby.md`](A5_zasoby.md).

> 🧪 **Ćwiczenie końcowe** — wydrukuj strony 1–4 i połóż obok klawiatury. Wszystko, czego używasz rzadziej niż raz w tygodniu, zostaw w pliku; wszystko, czego używasz codziennie, masz w palcach.

<!-- koniec modułu A1 -->