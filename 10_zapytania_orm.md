# Moduł 10 — Zapytania ORM w stylu 2.0

W tym module przestajesz pytać bazę „jak”, a zaczynasz pytać „co”. Nauczysz się budować zapytania w jednym, spójnym dialekcie `select()`, który działa identycznie w warstwie Core i ORM. Zobaczysz, dlaczego stary `Session.query()` odchodzi do lamusa i jak wygląda tabela migracyjna ze starego stylu na nowy. Opanujesz obiekt `Result` i wszystkie jego „skróty” (`scalars()`, `scalar()`, `one()`, `unique()`), nauczysz się filtrować na atrybutach Pythona, budować JOIN-y po relacjach i po kluczach obcych, używać `aliased()` do self-joinów, liczyć agregaty, sprawdzać istnienie rekordów przez `exists()` oraz wykonywać `INSERT`, `UPDATE` i `DELETE` **przez ORM** — z pełną kontrolą nad tym, co dzieje się z obiektami w pamięci (`synchronize_session`). Moduł kończy się warsztatem: dziesięć gotowych zapytań do schematu biblioteki z modułu 09, od najprostszego filtra do raportu z agregacją i wstawiania z `RETURNING`.

---

| | |
|---|---|
| **Poziom** | 🟠 zaawansowany |
| **Czas** | ~180 min (plus ~60 min na ćwiczenia) |
| **Wymagania wstępne** | [Moduł 04 — `select()` i `Result`](../course/04_select_i_result.md) (Core), [Moduł 07 — Modele deklaratywne](../course/07_modele_deklaratywne.md), [Moduł 08 — Sesja i cykl życia](../course/08_sesja_cykl_zycia.md), [Moduł 09 — Relacje](../course/09_relacje.md) |
| **Zakres pliku** | Dialekt `select()` w ORM, `Result` i jego warianty, filtrowanie, JOIN-y, `aliased()`, `contains_eager()`, agregacje, `exists()`, `union_all`, `yield_per`, ORM-owy DML (`insert`/`update`/`delete`), `synchronize_session`, bulk insert, kompozycja zapytań |
| **Baza** | SQLite (`library.db`) — zero konfiguracji. Różnice dla PostgreSQL zaznaczone jawnie |
| **Wersja** | SQLAlchemy 2.0.x; nowości 2.1 w ramkach 🆕 |

---

## Spis treści

- [1. Filozofia 2.0: jeden dialekt zapytań](#1-filozofia-20-jeden-dialekt-zapytań)
  - [1.1 Problem: dwa światy, które nie chciały się spotkać](#11-problem-dwa-światy-które-nie-chciały-się-spotkać)
  - [1.2 Co zmieniło 2.0](#12-co-zmieniło-20)
  - [1.3 Tabela migracyjna: stary styl → nowy styl](#13-tabela-migracyjna-stary-styl--nowy-styl)
  - [1.4 Dlaczego `Session.query()` odchodzi do lamusa](#14-dlaczego-sessionquery-odchodzi-do-lamusa)
  - [1.5 Pełny przepływ zapytania](#15-pełny-przepływ-zapytania)
- [2. Przygotowanie: modele i dane biblioteki](#2-przygotowanie-modele-i-dane-biblioteki)
- [3. `select(Entity)` kontra `select(Entity.kolumna)`](#3-selectentity-kontra-selectentitykolumna)
- [4. `session.execute()` i obiekt `Result`](#4-sessionexecute-i-obiekt-result)
- [5. Filtrowanie w ORM](#5-filtrowanie-w-orm)
- [6. JOIN-y w ORM](#6-join-y-w-orm)
- [7. `aliased()` i `contains_eager()`](#7-aliased-i-contains_eager)
- [8. Agregacje i sprawdzanie istnienia](#8-agregacje-i-sprawdzanie-istnienia)
- [9. Zagadnienia zaawansowane](#9-zagadnienia-zaawansowane)
- [10. DML przez ORM: INSERT, UPDATE, DELETE](#10-dml-przez-orm-insert-update-delete)
- [11. Czytelność: kompozycja zapytań](#11-czytelność-kompozycja-zapytań)
- [12. Warsztat: dziesięć zapytań do biblioteki](#12-warsztat-dziesięć-zapytań-do-biblioteki)
- [Podsumowanie](#podsumowanie)
- [Ćwiczenia](#ćwiczenia)
- [Najczęstsze błędy i jak je czytać](#najczęstsze-błędy-i-jak-je-czytać)
- [Słowniczek modułu](#słowniczek-modułu)
- [Dalsze czytanie](#dalsze-czytanie)
- [Co dalej](#co-dalej)

---

## 1. Filozofia 2.0: jeden dialekt zapytań

### 1.1 Problem: dwa światy, które nie chciały się spotkać

Wyobraź sobie, że w jednym mieście mówi się dwoma językami urzędowymi, które mają wspólne słowa, ale inną gramatykę. Mieszkańcy dogadują się, ale każde zdanie trzeba tłumaczyć dwa razy — raz dla okienka A, raz dla okienka B.

Tak właśnie wyglądało SQLAlchemy przed wersją 1.4. Istniały dwa równoległe światy:

- **Core** — pracujesz na tabelach i kolumnach: `select(book_table).where(book_table.c.year >= 1960)`. Budujesz SQL jak z klocków. Widzisz dokładnie, co leci do bazy.
- **ORM** — pracujesz na klasach: `session.query(Book).filter(Book.year >= 1960)`. Mówisz „daj mi książki”, a nie „daj mi wiersze z tabeli book”.

Oba światy miały własny zestaw metod. `select()` miał `where()`, `join()`, `group_by()`, `order_by()`. `Query` miał `filter()` (synonim `where`), `join()`, `group_by()`, `order_by()` — ale też `filter_by()`, `get()`, `count()`, `one()`, `scalar()`, `yield_per()`, `with_entities()`, `options()`… Metody o tych samych nazwach potrafiły działać inaczej. A jeśli chciałeś zrobić coś nietypowego — na przykład użyć podzapytania skorelowanego w ORM — musiałeś przeskakiwać między światami i tworzyć hybrydy:

```python
# STYL 1.x — NIE UŻYWAJ. Pokazujemy tylko, jak to wyglądało.
from sqlalchemy import func

# query() nie umiało bezpośrednio przyjąć podzapytania w kolumnach,
# więc trzeba było mieszać Query z Core:
subq = (
    session.query(func.count(Loan.id))
    .filter(Loan.member_id == Member.id)
    .correlate(Member)
    .subquery()
)
rows = session.query(Member, subq.as_scalar().label("loans")).all()
```

To działało. Ale czytelność była kiepska, a typowanie statyczne — praktycznie niemożliwe.

> 💡 **Analogia** — Pomyśl o Core jako o zestawie klocków LEGO (każdy element to tabela, kolumna, warunek), a o starym ORM jako o gotowym zestawie z instrukcją (dodaj, zmień, usuń obiekt). Problem w tym, że zestaw i klocki były w dwóch osobnych pudełkach z różnymi oznaczeniami. SQLAlchemy 2.0 wsypało oba pudełka do jednego, zachowując instrukcję — ale teraz instrukcja jest napisana tym samym językiem co klocki.

### 1.2 Co zmieniło 2.0

SQLAlchemy 2.0 postawiło jedną, twardą tezę: **jest jeden sposób budowania zapytań — `select()`**. Nie ma osobnego dialektu dla ORM. Różnica między zapytaniem „core'owym” a „ormowym” polega wyłącznie na tym, **co wstawisz do środka**:

| Co wstawiasz do `select()` | Co dostajesz | Kto to wykonuje |
|---|---|---|
| `Table` / `Column` | `Row` z kolumnami | `Connection.execute()` lub `Session.execute()` |
| klasa encji, np. `Book` | obiekty `Book` | tylko `Session.execute()` |
| encja + kolumny | `Row` z obiektem i wartościami | tylko `Session.execute()` |
| wyrażenie, np. `func.count()` | `Row` ze skalarem | oba |

To jest cała magia. `select(Book)` to zapytanie ORM-owe, bo w środku jest klasa ORM-owa. `select(Book.id, Book.title)` to zapytanie zwracające wiersze z dwiema kolumnami — i możesz je wykonać zarówno przez `Connection`, jak i przez `Session`. Ten sam `select`, ten sam `where`, ten sam `join`.

> 🧠 **Dlaczego tak jest** — Powód jest praktyczny. SQLAlchemy chciało mieć jedno miejsce, w którym definiuje się semantykę SQL (operator `==`, `in_()`, `join()`, `func.*`), i jedno miejsce, w którym definiuje się, jak wynik trafia do Pythona (obiekt ORM czy `Row`). Rozdzielenie tych dwóch spraw pozwoliło dodać w 2.0 rzeczy, które wcześniej były koszmarem: pełne typowanie (`Select[tuple[Book]]`), asynchroniczność (`AsyncSession` używa dokładnie tego samego `select()`) oraz jednolitą obsługę `RETURNING`.

### 1.3 Tabela migracyjna: stary styl → nowy styl

Jeśli kiedykolwiek trafisz na tutorial z 2018 roku (albo na kod kolegi z zespołu), zobaczysz `Query`. Poniżej mapowanie, którego będziesz potrzebować, żeby to przeczytać — oraz żeby samemu tak nie pisać.

| Stary styl (1.x) | Nowy styl (2.0) | Uwaga |
|---|---|---|
| `session.query(Book)` | `select(Book)` | Sam `select()` nic nie wykonuje — to tylko obiekt zapytania |
| `session.query(Book).all()` | `session.scalars(select(Book)).all()` | `scalars()` = „daj mi pierwszą kolumnę każdego wiersza” |
| `session.query(Book).first()` | `session.scalars(select(Book)).first()` | Zwraca `Book \| None` |
| `session.query(Book).one()` | `session.scalars(select(Book)).one()` | Rzuca wyjątek, jeśli nie dokładnie jeden |
| `session.query(Book).count()` | `session.scalar(select(func.count()).select_from(Book))` | Zobacz sekcję 8 |
| `session.query(Book).get(1)` | `session.get(Book, 1)` | Bez budowania zapytania |
| `session.query(Book).filter(...)` | `select(Book).where(...)` | `filter()` to dawny synonim `where()`; `filter_by()` nie ma odpowiednika — użyj `where()` |
| `session.query(Book).filter_by(title="X")` | `select(Book).where(Book.title == "X")` | `filter_by` przyjmował tylko `=` |
| `session.query(Book).join(Author)` | `select(Book).join(Author)` | Ta sama semantyka |
| `session.query(Book, Author).all()` | `session.execute(select(Book, Author)).all()` | Wynik to `Row(Book, Author)` |
| `session.query(Book.title).all()` | `session.execute(select(Book.title)).all()` | Zwraca `Row`, nie stringi! Uwaga na pułapkę |
| `session.query(func.count(Book.id)).scalar()` | `session.scalar(select(func.count(Book.id)))` | `session.scalar()` = pierwsza kolumna pierwszego wiersza |
| `query.with_entities(...)` | `select(...)` — po prostu wybierz kolumny | Metoda znika |
| `query.update({...})` | `session.execute(update(Book).values(...))` | ORM-owy UPDATE — sekcja 10 |
| `query.delete()` | `session.execute(delete(Book).where(...))` | ORM-owy DELETE — sekcja 10 |
| `query.options(joinedload(...))` | `select(Book).options(joinedload(...))` | Ta sama składnia `options()` |
| `query.populate_existing()` | `select(Book).execution_options(populate_existing=True)` | Sekcja 9 |
| `query.with_for_update()` | `select(Book).with_for_update()` | Sekcja 9 |
| `query.yield_per(1000)` | `select(Book).execution_options(yield_per=1000)` | Sekcja 9 |
| `query.get_or_404` (Flask-SQLAlchemy) | `session.get_one(Book, 1)` + obsługa `NoResultFound` | `get_one()` istnieje od 2.0 |

> ⚠️ **Pułapka** — Wiersz `session.query(Book.title).all()` → `session.execute(select(Book.title)).all()` jest najbardziej zdradliwy na całej liście. W starym stylu `query(Book.title)` zwracał krotki jednoelementowe, ale ludzie i tak pisali `.scalars()` na końcu. W nowym stylu `session.execute(select(Book.title))` zwraca `Row` — obiekty przypominające krotki. Jeśli chcesz same stringi, musisz dołożyć `.scalars()`. To źródło setek błędów przy migracji.

### 1.4 Dlaczego `Session.query()` odchodzi do lamusa

`Session.query()` **nadal istnieje** w SQLAlchemy 2.0 i nadal działa — nie zostanie usunięte w żadnej bliskiej wersji, bo zbyt wiele bibliotek z niego korzysta. Ale w dokumentacji ma status **legacy** (spadek, dziedzictwo) i jest opisany w osobnym dokumencie „Legacy Query API”. Powody, dla których nie powinieneś go używać w nowym kodzie:

1. **Nie obsługuje typowania.** `session.query(Book)` ma typ `Query[Book]` w najlepszym razie, ale `Query` nie jest generyczne nad kształtem wiersza. `select(Book)` ma typ `Select[tuple[Book]]` — mypy i Pylance wiedzą, że `.scalars()` da ci `ScalarResult[Book]`.
2. **Nie współpracuje z async.** `AsyncSession` nie ma metody `.query()` w ogóle. Jeśli dziś piszesz kod sync z `Query`, przepisanie na async oznacza przepisanie wszystkiego.
3. **Jest drugą implementacją tych samych rzeczy.** Każda nowa funkcja SQLAlchemy od 1.4 jest dodawana najpierw (a czasem wyłącznie) do `select()`. Przykład: `insert().returning(Book)` zwracające obiekty ORM — nie ma odpowiednika w `Query`.
4. **Miesza warstwy.** `Query` udawał, że jest częścią ORM, ale w środku owijał Core. Efektem były sytuacje, w których to samo wyrażenie działało w `Query`, a nie działało w `select()` — i odwrotnie.
5. **Jest wolniejszy w typowych przypadkach.** 2.0 ma nowy cache kompilacji SQL i szybsze generowanie kluczy; `select()` korzysta z tego w pełni.

> 🧠 **Dlaczego tak jest** — SQLAlchemy nie może „wyłączyć” `Query`, bo to by zabiło połowę ekosystemu (Flask-SQLAlchemy, starsze projekty, tutoriale). Zamiast tego stosuje strategię „miękkiej migracji”: dokumentuje stary styl osobno, nowe funkcje dodaje tylko do nowego, a społeczność stopniowo przepisuje kod. Ta sama strategia dotyczy `declarative_base()` i `Column()` w modelach.

> 🧪 **Ćwiczenie** — Otwórz plik z dowolnego projektu na GitHubie, w którym widzisz `session.query(`. Przepisz jedno zapytanie na `select()` + `session.scalars()`. Zwróć uwagę, ile miejsc wymaga `.scalars()`, a ile nie.

### 1.5 Pełny przepływ zapytania

Zanim zaczniemy pisać kod, zapamiętaj ten diagram. Wraca on w każdym module kursu.

```text
   ┌─────────────────────────────────────────────────────────────────┐
   │  TWÓJ KOD                                                       │
   │  stmt = select(Book).where(Book.year >= 1960).order_by(Book.title)│
   └───────────────────────────┬─────────────────────────────────────┘
                               │  obiekt Select[tuple[Book]]
                               │  (jeszcze NIE jest to SQL!)
                               ▼
   ┌─────────────────────────────────────────────────────────────────┐
   │  SESJA: session.scalars(stmt)                                   │
   │   1. autoflush  — jeśli w sesji są niezacommitowane zmiany,     │
   │                   najpierw leci INSERT/UPDATE (moduł 08)        │
   │   2. kompilacja — Select → SQL tekstowy + parametry wiązane     │
   │   3. wykonanie  — przez Connection z puli (moduł 02)            │
   └───────────────────────────┬─────────────────────────────────────┘
                               │  SQL + parametry
                               ▼
   ┌─────────────────────────────────────────────────────────────────┐
   │  DBAPI (sqlite3 / psycopg)  →  BAZA DANYCH                      │
   └───────────────────────────┬─────────────────────────────────────┘
                               │  wiersze
                               ▼
   ┌─────────────────────────────────────────────────────────────────┐
   │  ORM: identity map (moduł 08)                                   │
   │   • dla każdego wiersza: czy obiekt o tym PK już istnieje?      │
   │   • TAK  → zwróć TEN SAM obiekt Pythona (świeżo wypełniony)     │
   │   • NIE  → utwórz nowy obiekt, wstaw do identity map            │
   └───────────────────────────┬─────────────────────────────────────┘
                               │
                               ▼
   ┌─────────────────────────────────────────────────────────────────┐
   │  ScalarResult[Book]  →  .all() / .first() / .one() / iteracja   │
   └─────────────────────────────────────────────────────────────────┘
```

Trzy rzeczy, które warto z tego diagramu zapamiętać:

- **`select()` to opis, nie wykonanie.** Dopóki nie wywołasz `session.execute()` / `session.scalars()`, do bazy nie poleci nic. Możesz `select()` trzymać w zmiennej, przekazywać między funkcjami, modyfikować.
- **ORM wtrąca się dopiero na końcu** — po wykonaniu SQL, przy zamienianiu wierszy na obiekty.
- **`autoflush` jest po drodze.** Każde zapytanie ORM-owe może najpierw wywołać `flush()`. To wyjaśnia, dlaczego czasem w logu widzisz INSERT przed SELECT-em.

> 🔬 **Pod maską** — `select(Book).where(Book.year >= 1960)` skompilowane dla SQLite:

```python
from sqlalchemy import select
from sqlalchemy.dialects import sqlite

from models import Book

stmt = select(Book).where(Book.year >= 1960)
print(stmt.compile(dialect=sqlite.dialect(), compile_kwargs={"literal_binds": True}))
```

```sql
SELECT book.id, book.title, book.year, book.pages, book.price, book.author_id
FROM book
WHERE book.year >= 1960
```

Zwróć uwagę: `select(Book)` rozwija się do **wszystkich** kolumn mapowanych na encję. To jedna z najczęstszych przyczyn „pobieram za dużo danych” — o tym w sekcji 3.

---

## 2. Przygotowanie: modele i dane biblioteki

W całym module korzystamy z jednego schematu — biblioteki z modułu 09. Utwórz katalog `sqla_library/` i dwa pliki. Skrypty uruchamiamy **z wnętrza tego katalogu**, dzięki czemu Python widzi `models.py` i `seed.py` jako zwykłe moduły.

### 2.1 Modele

```python
# sqla_library/models.py
"""Modele biblioteki — wspólny schemat dla całego modułu 10."""

from __future__ import annotations

import warnings
from datetime import date
from decimal import Decimal

from sqlalchemy import (
    Column,
    Date,
    ForeignKey,
    Numeric,
    String,
    Table,
    Text,
)
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship

# SQLite nie ma natywnego typu DECIMAL — SQLAlchemy ostrzega przy konwersji.
# Wyciszamy to w przykładach; PostgreSQL (moduły produkcyjne) nie ma problemu.
warnings.filterwarnings(
    "ignore", message=r".*does \*not\* support Decimal objects natively.*"
)


class Base(DeclarativeBase):
    """Wspólna klasa bazowa — jeden MetaData na cały projekt."""


# Tabela asocjacyjna dla relacji wiele-do-wielu Book <-> Category (moduł 09).
book_category = Table(
    "book_category",
    Base.metadata,
    Column("book_id", ForeignKey("book.id", ondelete="CASCADE"), primary_key=True),
    Column("category_id", ForeignKey("category.id", ondelete="CASCADE"), primary_key=True),
)


class Author(Base):
    __tablename__ = "author"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(120))
    country: Mapped[str | None] = mapped_column(String(2))

    books: Mapped[list[Book]] = relationship(back_populates="author")

    def __repr__(self) -> str:
        return f"Author(id={self.id!r}, name={self.name!r})"


class Category(Base):
    __tablename__ = "category"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(60), unique=True)
    parent_id: Mapped[int | None] = mapped_column(ForeignKey("category.id"))

    # Relacja samo-referencyjna — hierarchia kategorii (moduł 09).
    parent: Mapped[Category | None] = relationship(
        back_populates="children", remote_side=[id]
    )
    children: Mapped[list[Category]] = relationship(back_populates="parent")

    books: Mapped[list[Book]] = relationship(
        secondary=book_category, back_populates="categories"
    )

    def __repr__(self) -> str:
        return f"Category(id={self.id!r}, name={self.name!r})"


class Book(Base):
    __tablename__ = "book"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    year: Mapped[int | None]
    pages: Mapped[int | None]
    price: Mapped[Decimal] = mapped_column(Numeric(8, 2))
    author_id: Mapped[int] = mapped_column(ForeignKey("author.id"))

    author: Mapped[Author] = relationship(back_populates="books")
    categories: Mapped[list[Category]] = relationship(
        secondary=book_category, back_populates="books"
    )
    reviews: Mapped[list[Review]] = relationship(back_populates="book")
    loans: Mapped[list[Loan]] = relationship(back_populates="book")

    def __repr__(self) -> str:
        return f"Book(id={self.id!r}, title={self.title!r})"


class Member(Base):
    __tablename__ = "member"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(120), unique=True)
    name: Mapped[str] = mapped_column(String(120))
    joined_at: Mapped[date] = mapped_column(Date)

    loans: Mapped[list[Loan]] = relationship(back_populates="member")
    reviews: Mapped[list[Review]] = relationship(back_populates="member")

    def __repr__(self) -> str:
        return f"Member(id={self.id!r}, email={self.email!r})"


class Loan(Base):
    __tablename__ = "loan"

    id: Mapped[int] = mapped_column(primary_key=True)
    book_id: Mapped[int] = mapped_column(ForeignKey("book.id"))
    member_id: Mapped[int] = mapped_column(ForeignKey("member.id"))
    loaned_at: Mapped[date] = mapped_column(Date)
    due_at: Mapped[date] = mapped_column(Date)
    returned_at: Mapped[date | None] = mapped_column(Date)

    book: Mapped[Book] = relationship(back_populates="loans")
    member: Mapped[Member] = relationship(back_populates="loans")


class Review(Base):
    """Klucz główny złożony: jedna opinia na książkę i członka."""

    __tablename__ = "review"

    book_id: Mapped[int] = mapped_column(ForeignKey("book.id"), primary_key=True)
    member_id: Mapped[int] = mapped_column(ForeignKey("member.id"), primary_key=True)
    rating: Mapped[int]
    comment: Mapped[str | None] = mapped_column(Text)

    book: Mapped[Book] = relationship(back_populates="reviews")
    member: Mapped[Member] = relationship(back_populates="reviews")
```

### 2.2 Dane startowe

```python
# sqla_library/seed.py
"""Buduje plik library.db z danymi biblioteki i zwraca fabrykę sesji."""

from __future__ import annotations

from datetime import date
from decimal import Decimal

from sqlalchemy import create_engine
from sqlalchemy.orm import Session, sessionmaker

from models import Author, Base, Book, Category, Loan, Member, Review

DB_FILE = "library.db"


def make_engine(echo: bool = False):
    """Silnik SQLite oparty na pliku.

    Używamy pliku, a nie ':memory:', bo chcemy móc podglądać bazę
    w osobnym procesie (np. w sqlite3 CLI) między uruchomieniami.
    """
    return create_engine(f"sqlite:///{DB_FILE}", echo=echo)


def build(echo: bool = False) -> sessionmaker[Session]:
    """Tworzy schemat, wypełnia go danymi i zwraca fabrykę sesji."""
    engine = make_engine(echo=echo)
    Base.metadata.drop_all(engine)
    Base.metadata.create_all(engine)

    # expire_on_commit=False: po commicie obiekty zachowują wartości,
    # co czyni przykłady czytelniejszymi. Konsekwencje — moduł 08.
    factory = sessionmaker(bind=engine, expire_on_commit=False)
    with factory.begin() as session:
        _populate(session)
    return factory


def _populate(session: Session) -> None:
    fiction = Category(name="Fiction")
    nonfiction = Category(name="Non-fiction")
    poetry = Category(name="Poetry", parent=fiction)
    crime = Category(name="Crime", parent=fiction)
    scifi = Category(name="Science Fiction", parent=fiction)
    essay = Category(name="Essay", parent=nonfiction)
    session.add_all([fiction, nonfiction, poetry, crime, scifi, essay])

    le_guin = Author(name="Ursula K. Le Guin", country="US")
    lem = Author(name="Stanisław Lem", country="PL")
    christie = Author(name="Agatha Christie", country="GB")
    szymborska = Author(name="Wisława Szymborska", country="PL")
    borges = Author(name="Jorge Luis Borges", country="AR")
    tokarczuk = Author(name="Olga Tokarczuk", country="PL")
    session.add_all([le_guin, lem, christie, szymborska, borges, tokarczuk])

    left_hand = Book(
        title="The Left Hand of Darkness",
        year=1969,
        pages=304,
        price=Decimal("45.90"),
        author=le_guin,
        categories=[fiction, scifi],
    )
    dispossessed = Book(
        title="The Dispossessed",
        year=1974,
        pages=341,
        price=Decimal("49.90"),
        author=le_guin,
        categories=[fiction, scifi],
    )
    earthsea = Book(
        title="A Wizard of Earthsea",
        year=1968,
        pages=183,
        price=Decimal("39.90"),
        author=le_guin,
        categories=[fiction],
    )
    solaris = Book(
        title="Solaris",
        year=1961,
        pages=204,
        price=Decimal("42.00"),
        author=lem,
        categories=[fiction, scifi],
    )
    cyberiad = Book(
        title="The Cyberiad",
        year=1965,
        pages=295,
        price=Decimal("46.50"),
        author=lem,
        categories=[fiction, scifi],
    )
    masters_voice = Book(
        title="His Master's Voice",
        year=1968,
        pages=199,
        price=Decimal("41.00"),
        author=lem,
        categories=[fiction],
    )
    orient = Book(
        title="Murder on the Orient Express",
        year=1934,
        pages=256,
        price=Decimal("34.90"),
        author=christie,
        categories=[fiction, crime],
    )
    and_then = Book(
        title="And Then There Were None",
        year=1939,
        pages=272,
        price=Decimal("36.90"),
        author=christie,
        categories=[fiction, crime],
    )
    ackroyd = Book(
        title="The Murder of Roger Ackroyd",
        year=1926,
        pages=288,
        price=Decimal("33.90"),
        author=christie,
        categories=[fiction, crime],
    )
    view = Book(
        title="View with a Grain of Sand",
        year=1957,
        pages=214,
        price=Decimal("38.00"),
        author=szymborska,
        categories=[poetry],
    )
    here = Book(
        title="Here",
        year=2009,
        pages=96,
        price=Decimal("29.90"),
        author=szymborska,
        categories=[poetry],
    )
    ficciones = Book(
        title="Ficciones",
        year=1944,
        pages=224,
        price=Decimal("44.00"),
        author=borges,
        categories=[fiction],
    )
    aleph = Book(
        title="The Aleph",
        year=1949,
        pages=210,
        price=Decimal("43.00"),
        author=borges,
        categories=[fiction],
    )
    flights = Book(
        title="Flights",
        year=2007,
        pages=403,
        price=Decimal("54.90"),
        author=tokarczuk,
        categories=[fiction],
    )
    plow = Book(
        title="Drive Your Plow Over the Bones of the Dead",
        year=2009,
        pages=274,
        price=Decimal("47.90"),
        author=tokarczuk,
        categories=[fiction, crime],
    )
    session.add_all(
        [
            left_hand, dispossessed, earthsea, solaris, cyberiad, masters_voice,
            orient, and_then, ackroyd, view, here, ficciones, aleph, flights, plow,
        ]
    )

    alice = Member(email="alice@example.com", name="Alice Nowak", joined_at=date(2023, 1, 15))
    bob = Member(email="bob@example.com", name="Bob Kowalski", joined_at=date(2023, 3, 2))
    carol = Member(email="carol@example.com", name="Carol Wiśniewska", joined_at=date(2024, 6, 11))
    dave = Member(email="dave@example.com", name="Dave Zieliński", joined_at=date(2024, 9, 30))
    erin = Member(email="erin@example.com", name="Erin Lewandowska", joined_at=date(2025, 2, 14))
    frank = Member(email="frank@example.com", name="Frank Dąbrowski", joined_at=date(2025, 7, 1))
    session.add_all([alice, bob, carol, dave, erin, frank])

    loans = [
        Loan(book=solaris, member=alice, loaned_at=date(2025, 1, 10),
             due_at=date(2025, 2, 10), returned_at=date(2025, 2, 3)),
        Loan(book=left_hand, member=alice, loaned_at=date(2025, 3, 1),
             due_at=date(2025, 4, 1), returned_at=date(2025, 3, 25)),
        Loan(book=cyberiad, member=alice, loaned_at=date(2025, 5, 5),
             due_at=date(2025, 6, 5), returned_at=None),
        Loan(book=orient, member=bob, loaned_at=date(2025, 2, 2),
             due_at=date(2025, 3, 2), returned_at=date(2025, 2, 28)),
        Loan(book=and_then, member=bob, loaned_at=date(2025, 4, 14),
             due_at=date(2025, 5, 14), returned_at=date(2025, 5, 20)),
        Loan(book=ackroyd, member=bob, loaned_at=date(2025, 6, 1),
             due_at=date(2025, 7, 1), returned_at=None),
        Loan(book=ficciones, member=carol, loaned_at=date(2025, 1, 20),
             due_at=date(2025, 2, 20), returned_at=date(2025, 2, 18)),
        Loan(book=aleph, member=carol, loaned_at=date(2025, 3, 15),
             due_at=date(2025, 4, 15), returned_at=date(2025, 4, 11)),
        Loan(book=flights, member=carol, loaned_at=date(2025, 5, 22),
             due_at=date(2025, 6, 22), returned_at=date(2025, 6, 19)),
        Loan(book=plow, member=carol, loaned_at=date(2025, 8, 3),
             due_at=date(2025, 9, 3), returned_at=None),
        Loan(book=earthsea, member=dave, loaned_at=date(2025, 2, 25),
             due_at=date(2025, 3, 25), returned_at=date(2025, 3, 30)),
        Loan(book=masters_voice, member=dave, loaned_at=date(2025, 6, 10),
             due_at=date(2025, 7, 10), returned_at=date(2025, 7, 8)),
        Loan(book=view, member=erin, loaned_at=date(2025, 4, 4),
             due_at=date(2025, 5, 4), returned_at=date(2025, 4, 30)),
        Loan(book=here, member=erin, loaned_at=date(2025, 7, 7),
             due_at=date(2025, 8, 7), returned_at=None),
        Loan(book=dispossessed, member=erin, loaned_at=date(2025, 9, 1),
             due_at=date(2025, 10, 1), returned_at=None),
        Loan(book=orient, member=frank, loaned_at=date(2025, 8, 12),
             due_at=date(2025, 9, 12), returned_at=None),
        Loan(book=left_hand, member=frank, loaned_at=date(2025, 9, 5),
             due_at=date(2025, 10, 5), returned_at=None),
    ]
    session.add_all(loans)

    session.add_all(
        [
            Review(book=left_hand, member=alice, rating=5, comment="Arcydzieło."),
            Review(book=solaris, member=alice, rating=4, comment=None),
            Review(book=cyberiad, member=bob, rating=5, comment="Bajki robotów."),
            Review(book=orient, member=bob, rating=4, comment=None),
            Review(book=flights, member=carol, rating=5, comment="Fragmentaryczna."),
            Review(book=plow, member=carol, rating=3, comment=None),
            Review(book=ficciones, member=dave, rating=5, comment="Labirynty."),
            Review(book=aleph, member=dave, rating=4, comment=None),
            Review(book=view, member=erin, rating=5, comment=None),
            Review(book=here, member=erin, rating=3, comment="Krótka."),
            Review(book=orient, member=frank, rating=2, comment="Znam film."),
            Review(book=left_hand, member=frank, rating=4, comment=None),
        ]
    )
```

### 2.3 Jak uruchamiać przykłady

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install "SQLAlchemy>=2.0"

mkdir sqla_library && cd sqla_library
# ... wklej models.py i seed.py ...

python -c "from seed import build; build(); print('baza gotowa')"
```

Od tego miejsca każdy snippet zakłada, że jesteś w katalogu `sqla_library/` i zaczyna się od:

```python
from models import Author, Book, Category, Loan, Member, Review
from seed import build
```

> 💡 **Analogia** — `models.py` to przepis na formę do ciasta, a `seed.py` to gotowe ciasto wyjęte z formy. Wszystkie dalsze eksperymenty robisz na tym samym cieście — dlatego wyniki z poszczególnych sekcji są porównywalne.

---

## 3. `select(Entity)` kontra `select(Entity.kolumna)`

To najważniejsza decyzja przy każdym zapytaniu ORM-owym: **czy chcę obiekty, czy dane?**

### 3.1 Encje: praca na obiektach

```python
# sqla_library/q01_entities.py
from sqlalchemy import select
from sqlalchemy.orm import Session

from models import Book
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    stmt = select(Book).where(Book.year >= 1960).order_by(Book.title)
    books: list[Book] = list(session.scalars(stmt))
    for book in books:
        print(book.title, "—", book.year)
```

```text
A Wizard of Earthsea — 1968
Flights — 2007
His Master's Voice — 1968
Solaris — 1961
The Cyberiad — 1965
The Dispossessed — 1974
The Left Hand of Darkness — 1969
Drive Your Plow Over the Bones of the Dead — 2009
```

Zwróć uwagę na typ zmiennej: `list[Book]`. To nie jest ozdoba — to informacja dla ciebie, dla IDE i dla mypy. Obiekty `Book` mają nie tylko dane, ale też **tożsamość** (są w identity map) i **zachowanie** (relacje, które możesz dociągnąć).

```python
    first = books[0]
    print(first.author.name)   # ← leniwe ładowanie: poleci dodatkowy SELECT
```

> 🔬 **Pod maską** — `select(Book)` generuje:

```sql
SELECT book.id, book.title, book.year, book.pages, book.price, book.author_id
FROM book
WHERE book.year >= ?
ORDER BY book.title
```

Wszystkie kolumny mapowane na `Book`. Przy 30 kolumnach w tabeli — wszystkie 30.

### 3.2 Kolumny: praca na danych

```python
# sqla_library/q02_columns.py
from sqlalchemy import select
from sqlalchemy.orm import Session

from models import Book
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    stmt = select(Book.title, Book.year).where(Book.year >= 1960).order_by(Book.title)
    rows = session.execute(stmt).all()
    for title, year in rows:
        print(f"{year}: {title}")
    print("typ wiersza:", type(rows[0]).__name__)
    print("klucze:", rows[0]._mapping.keys())
```

```text
1968: A Wizard of Earthsea
2007: Flights
...
typ wiersza: Row
klucze: RMKeyView(['title', 'year'])
```

Dostałeś `Row` — obiekt, który zachowuje się jak krotka, ale ma też nazwy kolumn. `row.title` i `row[0]` działają tak samo. Możesz go rozpakować (`title, year = row`).

> 💡 **Analogia** — `select(Book)` to zamówienie całego pudełka z zawartością: książka, jej okładka, metryczka i jeszcze instrukcja obsługi (relacje). `select(Book.title, Book.year)` to zamówienie dwóch karteczek z magazynu. Jeśli potrzebujesz tylko dwóch karteczek, po co taszczyć całe pudełko?

### 3.3 Kiedy co wybrać

| Sytuacja | Co wybrać | Dlaczego |
|---|---|---|
| Chcesz zmodyfikować obiekt i zapisać zmiany | `select(Book)` | ORM musi śledzić obiekt w sesji, żeby wykryć zmianę |
| Chcesz użyć relacji (`book.author.name`) | `select(Book)` | Tylko encja ma atrybuty relacji |
| Budujesz raport / tabelkę w UI | `select(Book.title, Book.year)` | Mniej danych z bazy, mniej pamięci, szybszy transfer |
| Liczysz coś (`count`, `sum`, `avg`) | `select(func.count())` | Nie ma sensu materializować obiektów |
| Endpoint API listujący 10 000 rekordów | kolumny lub DTO | Encje z relacjami = ryzyko N+1 (moduł 11) |
| Chcesz zwrócić encję z serializacją Pydantic | `select(Book)` + eager load | Pydantic potrzebuje atrybutów (moduł 22) |
| Sprawdzasz istnienie rekordu | `select(exists())` lub `select(Book.id)` | Nie pobieraj 40 kolumn, żeby odpowiedzieć „tak/nie” |

> 🧠 **Dlaczego tak jest** — ORM musi wykonać dodatkową pracę dla każdego obiektu: sprawdzić identity map, utworzyć instancję klasy, ustawić atrybuty przez instrumentację, dodać do `session.dirty`/`session.identity_map`, śledzić historię zmian. To jest koszt rzędu mikrosekund na obiekt — ale przy 100 000 wierszy robi się z tego sekunda. Kolumny tego kosztu nie mają: dostajesz surowe `Row`.

> 🆕 **SQLAlchemy 2.1** — Typowanie `Result` i `Row` zostało poprawione zgodnie z PEP 646 (variadic generics). W 2.0 `Row` z `select(Book.title, Book.year)` ma typ przybliżony; w 2.1 mypy potrafi wywnioskować krotkę `tuple[str, int | None]` dokładnie z kształtu zapytania. Jeśli pracujesz z mypy w trybie `strict`, to realna różnica.

### 3.4 Mieszanie: encja i kolumny razem

Możesz poprosić o jedno i drugie — to bardzo przydatne, gdy chcesz encję *oraz* policzoną wartość:

```python
# sqla_library/q03_mixed.py
from sqlalchemy import func, select
from sqlalchemy.orm import Session

from models import Book, Loan
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    loan_count = func.count(Loan.id).label("loan_count")
    stmt = (
        select(Book, loan_count)
        .outerjoin(Loan, Loan.book_id == Book.id)
        .group_by(Book.id)
        .order_by(loan_count.desc())
        .limit(5)
    )
    for book, count in session.execute(stmt):
        print(f"{count:>2}x  {book.title}")
```

```text
 3x  The Left Hand of Darkness
 2x  Murder on the Orient Express
 1x  Solaris
 1x  The Cyberiad
 1x  And Then There Were None
```

Wiersz ma dwie pozycje: obiekt `Book` (pełnoprawny, w identity map) i liczbę. Rozpakowanie `book, count = row` działa.

> ⚠️ **Pułapka** — Jeśli napiszesz `session.scalars(stmt)` na takim zapytaniu, dostaniesz **tylko obiekty `Book`** — liczba zostanie po cichu wyrzucona. `scalars()` nie zgłasza błędu przy wielokolumnowym zapytaniu; bierze pierwszą kolumnę każdego wiersza. To najczęstszy cichy błąd w ORM. Typowanie statyczne to łapie (`ScalarResult[Book]` vs `tuple[Book, int]`), ale w runtime — cisza.

---

## 4. `session.execute()` i obiekt `Result`

### 4.1 Co zwraca `execute()`

`session.execute(stmt)` **zawsze** zwraca `Result`. Kształt tego, co jest w środku, zależy od tego, co wstawiłeś do `select()`:

| `select(...)` | Zawartość `Result` | Metoda, która daje „wygodne” |
|---|---|---|
| `select(Book)` | `Row` z jednym elementem (encja) | `.scalars()` |
| `select(Book, Author)` | `Row` z dwoma elementami | `.tuples()` lub iteracja |
| `select(Book.title, Book.year)` | `Row` z dwoma kolumnami | `.all()` / `.mappings()` |
| `select(func.count())` | `Row` z jednym skalarem | `.scalar()` |

> 💡 **Analogia** — `Result` to taca kelnera: może nieść jedno danie, dwa dania albo cały zestaw. `scalars()` to polecenie „podaj mi tylko to, co jest na środku talerza”. Jeśli na talerzu jest zupa i kotlet, `scalars()` poda ci zupę i ani słowem nie wspomni o kotlecie.

### 4.2 `scalars()` i `ScalarResult`

```python
# sqla_library/q04_scalars.py
from sqlalchemy import select
from sqlalchemy.orm import Session

from models import Book
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    stmt = select(Book).where(Book.pages > 250).order_by(Book.pages)

    # Result → ScalarResult
    result = session.execute(stmt)
    scalars = result.scalars()
    print(type(scalars).__name__)          # ScalarResult

    # ScalarResult ma własne, wygodne metody:
    print(scalars.all()[:2])               # lista[Book]
    print(scalars.first())                 # Book | None
    print(len(scalars.all()))              # UWAGA: patrz pułapka niżej
```

`ScalarResult` oferuje:

| Metoda | Zwraca | Uwagi |
|---|---|---|
| `.all()` | `list[Book]` | Materializuje wszystko w pamięci |
| `.first()` | `Book \| None` | Dodaje `LIMIT 1`? **Nie** — pobiera i odrzuca resztę |
| `.one()` | `Book` | Rzuca `NoResultFound` / `MultipleResultsFound` |
| `.one_or_none()` | `Book \| None` | Rzuca tylko przy > 1 |
| `.unique()` | `ScalarResult[Book]` | Deduplikacja po tożsamości ORM (patrz 4.5) |
| `.partitions(n)` | iterator list po `n` | Strumieniowe porcjowanie |
| iteracja | `Book` | Leniwie, wiersz po wierszu |

> ⚠️ **Pułapka** — `Result` i `ScalarResult` są **jednorazowe**. To strumień z kursora bazy. Po `.all()` kursor jest wyczerpany; kolejne `.all()` zwróci pustą listę. Jeśli w powyższym przykładzie zamienisz kolejność na `scalars.first()` → `scalars.all()`, drugie wywołanie da `[]`. Zapamiętaj: **albo `.all()`, albo iteracja — nie oba**.

### 4.3 `scalar()` — jedno pole, jedna wartość

`session.scalar(stmt)` to skrót: „wykonaj, weź pierwszy wiersz, weź pierwszą kolumnę”. Idealne do agregatów.

```python
# sqla_library/q05_scalar.py
from sqlalchemy import func, select
from sqlalchemy.orm import Session

from models import Book
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    total: int | None = session.scalar(select(func.count(Book.id)))
    avg_pages: float | None = session.scalar(select(func.avg(Book.pages)))
    newest: int | None = session.scalar(select(func.max(Book.year)))

    print("książek:", total)
    print("średnia stron:", round(avg_pages or 0, 1))
    print("najnowsza:", newest)

    # Gdy nie ma wyników, scalar() zwraca None — nie rzuca wyjątku.
    missing = session.scalar(select(Book.title).where(Book.id == 9999))
    print("brak wyniku:", missing)
```

```text
książek: 15
średnia stron: 250.9
najnowsza: 2009
brak wyniku: None
```

> 🧠 **Dlaczego tak jest** — `scalar()` zwraca `None` dla pustego wyniku, bo tak działa agregat SQL: `SELECT count(*) FROM book` na pustej tabeli zwraca `0` w jednym wierszu. Gdyby `scalar()` rzucał wyjątek, kod agregujący byłby pełen `try/except`. Ale ta wygoda ma cenę: jeśli zapomnisz, że `scalar()` może zwrócić `None`, dostaniesz `TypeError` w dalszej części kodu.

### 4.4 `one()`, `one_or_none()`, `first()` — kontrakt na liczbę wyników

To jedna z najpiękniejszych rzeczy w SQLAlchemy 2.0: możesz **zakodować w zapytaniu swoje oczekiwanie** co do liczby wyników.

| Metoda | 0 wyników | 1 wynik | 2+ wyniki | Kiedy używać |
|---|---|---|---|---|
| `.one()` | `NoResultFound` | obiekt | `MultipleResultsFound` | Unikalny klucz naturalny, np. e-mail |
| `.one_or_none()` | `None` | obiekt | `MultipleResultsFound` | Opcjonalna relacja 1:1 |
| `.first()` | `None` | obiekt | pierwszy, reszta odrzucona | „Daj cokolwiek” |
| `.scalar()` | `None` | wartość | wartość z pierwszego wiersza | Agregaty |

```python
# sqla_library/q06_one.py
from sqlalchemy import select
from sqlalchemy.orm import Session
from sqlalchemy.exc import MultipleResultsFound, NoResultFound

from models import Member
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    # Dokładnie jeden wynik — e-mail jest unikalny, więc to niezmiennik.
    member = session.scalars(
        select(Member).where(Member.email == "alice@example.com")
    ).one()
    print("znaleziono:", member.name)

    # Zero wyników → None (a nie wyjątek).
    maybe = session.scalars(
        select(Member).where(Member.email == "nikt@example.com")
    ).one_or_none()
    print("one_or_none:", maybe)

    # Kontrakt złamany → wyjątek, który mówi dokładnie, co się stało.
    try:
        session.scalars(select(Member).where(Member.name.like("%a%"))).one()
    except MultipleResultsFound as exc:
        print("MultipleResultsFound:", str(exc)[:60], "...")
    except NoResultFound as exc:
        print("NoResultFound:", exc)
```

```text
znaleziono: Alice Nowak
one_or_none: None
MultipleResultsFound: Multiple rows were found when exactly one was required ...
```

> 💡 **Analogia** — `.one()` to pytanie „podaj mi **tę jedyną** osobę o numerze PESEL 123…”. Jeśli w bazie są dwie osoby z tym numerem, to nie jest problem zapytania — to **awaria danych**. Wyjątek `MultipleResultsFound` jest tu nie awarią, a alarmem pożarowym. `.first()` to pytanie „podaj mi **jakąkolwiek** osobę z listy” — nie masz prawa się zdziwić, że wynik jest dowolny.

> 🧠 **Dlaczego tak jest** — `.one()` jest **asercją niezmiennika** zapisaną w kodzie. Zamiast komentować `# tu zawsze jest jeden`, mówisz to bazie i ORM-owi. Gdy niezmiennik pęknie (np. po błędnej migracji), dowiesz się o tym natychmiast, w miejscu awarii, a nie trzy moduły dalej.

### 4.5 `unique()` — dlaczego i kiedy

To najbardziej niezrozumiała metoda w całym ORM. Wyjaśnijmy ją porządnie.

Gdy ładujesz relację **kolekcji** (np. `books`) przez `joinedload`, SQLAlchemy robi `LEFT OUTER JOIN`. Wyobraź sobie autora z 5 książkami: JOIN zwróci **5 wierszy** dla jednego autora — po jednym na książkę. Ale ty prosiłeś o autorów, nie o wiersze.

```sql
-- joinedload(Author.books): jeden autor, pięć wierszy
SELECT author.id, author.name, book_1.id, book_1.title, ...
FROM author LEFT OUTER JOIN book AS book_1 ON author.id = book_1.author_id
```

ORM widzi pięć wierszy z tym samym `author.id` i **deduplikuje je po tożsamości** — ale tylko jeśli mu na to pozwolisz przez `.unique()`. Bez tego dostaniesz pięć kopii tego samego obiektu `Author` (a właściwie ten sam obiekt Pythona pięć razy na liście).

```python
# sqla_library/q07_unique.py
from sqlalchemy import select
from sqlalchemy.orm import Session, joinedload

from models import Author
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    stmt = select(Author).options(joinedload(Author.books)).order_by(Author.name)

    # BEZ unique() — lista zawiera duplikaty (te same obiekty Pythona).
    without = session.scalars(stmt).all()
    print("bez unique():", len(without), "pozycji")

    # Z unique() — dokładnie jeden obiekt na autora.
    with_unique = session.scalars(stmt).unique().all()
    print("z unique():  ", len(with_unique), "pozycji")
```

```text
bez unique(): 15 pozycji
z unique():   6 pozycji
```

Piętnaście pozycji dla sześciu autorów — bo każdy autor pojawia się tyle razy, ile ma książek (plus autorzy bez książek po jednym razie).

> ⚠️ **Pułapka** — W SQLAlchemy 2.0 wywołanie `.unique()` jest **wymagane** przy `joinedload` kolekcji. Jeśli go pominiesz, dostaniesz `InvalidRequestError`:

```text
sqlalchemy.exc.InvalidRequestError: The unique() method must be invoked on this Result,
as it contains results that include joined eager loads against collections
```

To dobra wiadomość: 2.0 **nie pozwoli ci** po cichu dostać zduplikowanych wyników. Zła wiadomość: musisz o tym pamiętać. Zasada praktyczna — **jeśli w `options()` masz `joinedload` na relacji typu `list`, zawsze dołóż `.unique()`**.

> 🔬 **Pod maską** — deduplikacja odbywa się w Pythonie, nie w bazie. SQLAlchemy ma „znacznik” w wierszach (`__safepostfetch__`) i wie, że `(author, 1)`, `(author, 1)`, `(author, 1)` to ta sama tożsamość. Gdybyś użył `distinct()` w SQL, nie zadziałałoby — wiersze różnią się kolumnami książek. Dlatego `unique()` musi być po stronie ORM.

### 4.6 `mappings()` i `tuples()`

Dwa dodatkowe widoki na `Result`, przydatne w konkretnych sytuacjach:

```python
# sqla_library/q08_views.py
from sqlalchemy import select
from sqlalchemy.orm import Session

from models import Book
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    stmt = select(Book.title, Book.year).where(Book.year > 2000).order_by(Book.title)

    # mappings() — każdy wiersz jako słownikopodobny obiekt
    for row in session.execute(stmt).mappings():
        print(dict(row))
        # {'title': 'Flights', 'year': 2007}

    # tuples() — każdy wiersz jako prawdziwa krotka (bez nazw kolumn)
    for tup in session.execute(stmt).tuples():
        print(tup, type(tup).__name__)
        # ('Flights', 2007) tuple
```

| Widok | Element | Kiedy |
|---|---|---|
| domyślny | `Row` | Nazwy + indeksy, rozpakowywanie |
| `.mappings()` | `RowMapping` | Serializacja do JSON, `**row` |
| `.scalars()` | pierwsza kolumna | Encje, agregaty |
| `.tuples()` | `tuple` | Przekazywanie do API oczekującego krotek |

### 4.7 Skróty na poziomie sesji

Trzy metody sesji, które oszczędzają pisania:

| Metoda sesji | Odpowiednik | Zwraca |
|---|---|---|
| `session.execute(stmt)` | — | `Result` |
| `session.scalars(stmt)` | `session.execute(stmt).scalars()` | `ScalarResult[T]` |
| `session.scalar(stmt)` | `session.execute(stmt).scalar()` | `T \| None` |

```python
# sqla_library/q09_shortcuts.py
from sqlalchemy import func, select
from sqlalchemy.orm import Session

from models import Book
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    # Trzy równoważne zapisy tego samego:
    a = session.execute(select(Book)).scalars().all()
    b = session.scalars(select(Book)).all()
    c = [row[0] for row in session.execute(select(Book)).all()]
    print(len(a), len(b), len(c))   # 15 15 15

    # Trzy równoważne zapisy agregatu:
    x = session.execute(select(func.count(Book.id))).scalar()
    y = session.scalar(select(func.count(Book.id)))
    print(x, y)                     # 15 15
```

W praktyce: **`session.scalars()`** i **`session.scalar()`** są tak krótkie, że nie ma sensu pisać `execute(...).scalars()`. Wyjątki:

- gdy chcesz użyć `Result.unique()` w środku łańcucha → `session.execute(stmt).unique().scalars()`
- gdy chcesz użyć `mappings()` lub `tuples()` → `session.execute(stmt).mappings()`

### 4.8 Tabela decyzyjna — czym zakończyć zapytanie

| Chcę… | Napisz |
|---|---|
| listę encji | `session.scalars(stmt).all()` |
| jedną encję, wiem że istnieje | `session.scalars(stmt).one()` |
| jedną encję lub `None` | `session.scalars(stmt).one_or_none()` |
| pierwszą z wielu | `session.scalars(stmt).first()` |
| strumień encji (dużo danych) | `for book in session.scalars(stmt):` |
| listę krotek (raport) | `session.execute(stmt).all()` |
| listę słowników (JSON) | `[dict(r) for r in session.execute(stmt).mappings()]` |
| jeden agregat | `session.scalar(stmt)` |
| jeden wiersz z wieloma kolumnami | `session.execute(stmt).one()` |
| encję po kluczu głównym | `session.get(Book, 42)` |

> 🧪 **Ćwiczenie** — W powyższej tabeli brakuje wiersza „chcę policzyć, ile wyników zwróci zapytanie”. Zapisz, jak byś to zrobił. *(Podpowiedź: nie da się tego zrobić po wykonaniu zapytania bez pobrania wszystkich wierszy — musisz osobno zapytać bazę o `count()`.)*

---

## 5. Filtrowanie w ORM

### 5.1 `where()` i atrybuty instrumentowane

W modelach zadeklarowałeś `Mapped[int]`, `Mapped[str]`. Przy tworzeniu klasy SQLAlchemy zamienia te adnotacje na **atrybuty instrumentowane** (*instrumented attributes*) — obiekty, które zachowują się dwojako:

- **na instancji** (`book.year`) → zwracają wartość Pythona, np. `1969`
- **na klasie** (`Book.year`) → zwracają wyrażenie SQL, które można wstawić do `where()`

```python
# sqla_library/q10_instrumented.py
from sqlalchemy import select
from sqlalchemy.orm import Session

from models import Book
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    stmt = select(Book).where(Book.year >= 1960)
    print(stmt.compile(compile_kwargs={"literal_binds": True}))

    # To samo na instancji:
    with session.begin():
        probe = Book(title="X", year=2001, pages=1, price=1, author_id=1)
        print("book.year na instancji:", probe.year)   # 2001
        print("Book.year na klasie:    ", Book.year)   # book.year
```

```sql
SELECT book.id, book.title, book.year, book.pages, book.price, book.author_id
FROM book
WHERE book.year >= 1960
```

```text
book.year na instancji: 2001
Book.year na klasie:     book.year
```

> 🧠 **Dlaczego tak jest** — Python pozwala klasie sterować tym, co zwraca dostęp do atrybutu, przez deskryptory. SQLAlchemy tworzy dla każdej kolumny deskryptor, który sprawdza: „czy jesteś instancją, czy klasą?”. Na instancji oddaje wartość; na klasie oddaje `Column`-podobne wyrażenie SQL. Dzięki temu `Book.year >= 1960` w Pythonie jest legalnym wyrażeniem, które *nie* próbuje porównać `None` z liczbą — tylko buduje drzewo SQL.

> ⚠️ **Pułapka** — Porównanie **instancji** w `where()` daje bardzo mylący efekt:

```python
probe = Book(title="X", year=2001, pages=1, price=1, author_id=1)
stmt = select(Book).where(Book.year >= probe.year)   # OK — int
stmt = select(Book).where(Book.year >= probe)        # ✗ TypeError
```

Python próbuje porównać `InstrumentedAttribute` z obiektem `Book` i zgłasza `TypeError: '>=' not supported between instances of 'InstrumentedAttribute' and 'Book'`. Na szczęście — cicho działa tylko przypadek, gdy obiekt ma `__eq__` zwracający coś prawdziwego. Zawsze porównuj **kolumnę z wartością Pythona** albo **kolumnę z kolumną**.

### 5.2 Operatory porównania

| Python | SQL | Przykład |
|---|---|---|
| `==` | `=` | `Book.year == 1969` |
| `!=` | `!=` | `Book.year != 1969` |
| `<`, `<=`, `>`, `>=` | jak w SQL | `Book.pages >= 300` |
| `+`, `-`, `*`, `/` | arytmetyka SQL | `Book.price * 2` |
| `%` | `MOD` | `Book.id % 2` |

> ⚠️ **Pułapka** — `Book.year == None` **nie** wygeneruje `IS NULL`. SQLAlchemy zamienia to na `IS NULL` dla wygody, ale tylko dlatego, że jesteś w kontekście `where()`. Poza nim `Book.year == None` to wyrażenie SQL, które w Pythonie jest *prawdziwe* (bo to nie `None`), co prowadzi do absurdów typu:

```python
if Book.year == None:      # ZAWSZE prawda! To nie jest porównanie z None.
    print("to się zawsze wykona")
```

Zawsze używaj `Book.year.is_(None)` i `Book.year.is_not(None)`. To jawny, jednoznaczny zapis.

### 5.3 Tekst: `like`, `ilike`, `startswith`, `contains`

```python
# sqla_library/q11_text.py
from sqlalchemy import func, select
from sqlalchemy.orm import Session

from models import Book
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    def sql(stmt) -> str:
        return str(stmt.compile(compile_kwargs={"literal_binds": True}))

    print(sql(select(Book).where(Book.title.like("The %"))))
    print(sql(select(Book).where(Book.title.ilike("%murder%"))))
    print(sql(select(Book).where(Book.title.startswith("The"))))
    print(sql(select(Book).where(Book.title.endswith("ness"))))
    print(sql(select(Book).where(Book.title.contains("of"))))
    print(sql(select(Book).where(func.lower(Book.title) == "solaris")))
```

```sql
... WHERE book.title LIKE 'The %'
... WHERE book.title ILIKE '%murder%'
... WHERE book.title LIKE 'The' || '%'
... WHERE book.title LIKE '%' || 'ness'
... WHERE book.title LIKE '%' || 'of' || '%'
... WHERE lower(book.title) = 'solaris'
```

| Metoda | SQL | Uwaga |
|---|---|---|
| `.like(pattern)` | `LIKE` | `%` i `_` to wildcardy; **case-sensitive** w PostgreSQL |
| `.ilike(pattern)` | `ILIKE` / `lower(...) LIKE lower(...)` | PostgreSQL ma natywne `ILIKE`; SQLite dostaje wersję przez `lower()` |
| `.startswith(s)` | `LIKE 's%'` | Nie dodaje wildcarda do końca |
| `.endswith(s)` | `LIKE '%s'` | |
| `.contains(s)` | `LIKE '%s%'` | Domyślnie **case-sensitive**! |
| `.icontains(s)` | `LIKE ...` z `lower()` | 2.0 dodaje `icontains` |
| `.regexp_match(p)` | `REGEXP` / `~` | Dialektowo różne |

> ⚠️ **Pułapka** — `.contains("a")` jest **case-sensitive** i używa `LIKE`, a nie operatora „zawiera podciąg w dowolnej pozycji” z indeksem. Na tabeli z 5 milionami wierszy `WHERE title LIKE '%a%'` **nie użyje zwykłego indeksu B-tree** — baza musi przeskanować całą tabelę. Jeśli to twój hot path, potrzebujesz indeksu trigramowego (`pg_trgm` + `GIN`) w PostgreSQL albo wyszukiwarki pełnotekstowej. Szczegóły w module 17.

### 5.4 Zbiory: `in_`, `not_in`

```python
# sqla_library/q12_in.py
from sqlalchemy import select
from sqlalchemy.orm import Session

from models import Book
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    stmt = select(Book).where(Book.author_id.in_([1, 2, 6])).order_by(Book.title)
    print(str(stmt.compile(compile_kwargs={"literal_binds": True})))
    print([b.title for b in session.scalars(stmt)])

    # not_in — UWAGA na NULL-e!
    stmt2 = select(Book).where(Book.year.not_in([1969, 1974]))
    print(len(session.scalars(stmt2).all()))
```

```sql
SELECT ... FROM book WHERE book.author_id IN (1, 2, 6) ORDER BY book.title
```

> ⚠️ **Pułapka** — `NOT IN` z wartościami `NULL` w SQL jest zdradliwe. `x NOT IN (1, NULL)` **nigdy nie jest prawdą**, bo porównanie z `NULL` daje `UNKNOWN`. SQLAlchemy domyślnie pomija `None` w `not_in()` (generuje `x NOT IN (1)`), ale jeśli zbudujesz listę z podzapytania, które może zawierać `NULL`, dostaniesz pusty wynik i będziesz długo szukał przyczyny. Zasada: **kolumny w `IN` powinny być `NOT NULL`** albo filtruj `is_not(None)` jawnie.

> 💡 **Analogia** — `IN` to lista gości przy wejściu na imprezę. `NOT IN` z `NULL` to lista, na której ktoś napisał „i jeszcze jedna osoba, ale nie wiemy kto”. Ochroniarz nie wpuści nikogo, bo nie potrafi zweryfikować tego jednego wpisu.

### 5.5 NULL: `is_`, `is_not`

```python
# sqla_library/q13_null.py
from sqlalchemy import select
from sqlalchemy.orm import Session

from models import Loan
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    # Aktywne wypożyczenia (niezwrócone)
    active = select(Loan).where(Loan.returned_at.is_(None))
    print("aktywne:", len(session.scalars(active).all()))

    # Zakończone wypożyczenia
    done = select(Loan).where(Loan.returned_at.is_not(None))
    print("zwrócone:", len(session.scalars(done).all()))
```

```text
aktywne: 7
zwrócone: 10
```

`is_(None)` → `IS NULL`. `is_not(None)` → `IS NOT NULL`. Możesz też użyć `is_(True)` / `is_(False)` dla kolumn booleanowskich — w PostgreSQL to poprawna forma dla `BOOLEAN` (unikasz problemu z `= TRUE` a `NULL`).

### 5.6 Funkcje SQL w filtrach

`func` to fabryka: `func.cokolwiek(...)` generuje `cokolwiek(...)` w SQL.

```python
# sqla_library/q14_func.py
from datetime import date

from sqlalchemy import extract, func, select
from sqlalchemy.orm import Session

from models import Book, Loan
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    # Długość tytułu większa niż 20 znaków
    long_titles = select(Book).where(func.length(Book.title) > 20)
    print("długie tytuły:", len(session.scalars(long_titles).all()))

    # Wypożyczenia z roku 2025 (ekstrakcja roku z daty)
    in_2025 = select(Loan).where(extract("year", Loan.loaned_at) == 2025)
    print("wypożyczenia 2025:", len(session.scalars(in_2025).all()))

    # Wielka litera nazwiska autora — przez funkcję na kolumnie
    from models import Author
    stmt = select(Author).where(func.upper(Author.name).startswith("U"))
    print([a.name for a in session.scalars(stmt)])
```

```text
długie tytuły: 8
wypożyczenia 2025: 17
['Ursula K. Le Guin']
```

> 🔬 **Pod maską** — `func.length(Book.title)` na SQLite generuje `length(book.title)`; na PostgreSQL to samo (PostgreSQL ma `length()`). Ale `func.substr()` już się różni: PostgreSQL ma `substring()`, SQLite `substr()`. Gdy używasz `func.xxx()`, SQLAlchemy **nie tłumaczy** nazw funkcji między dialektami — przekazuje je dosłownie. To twoja odpowiedzialność.

> ⚠️ **Pułapka** — Indeks na kolumnie `title` **nie zadziała** dla `WHERE length(title) > 20` ani `WHERE lower(title) = 'x'`. Baza musi policzyć funkcję dla każdego wiersza. Rozwiązanie: indeks funkcyjny (`CREATE INDEX ... ON book (lower(title))`) w PostgreSQL. SQLite tego nie obsługuje w pełni.

### 5.7 Porównywanie atrybutów między sobą

To jedna z rzeczy, które w starym `Query` były koszmarne. W 2.0 działa naturalnie:

```python
# sqla_library/q15_attr_vs_attr.py
from sqlalchemy import select
from sqlalchemy.orm import Session

from models import Book, Loan
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    # Wypożyczenia przetrzymane dłużej niż 30 dni
    overdue = select(Loan).where(
        Loan.returned_at.is_not(None),
        (Loan.returned_at - Loan.loaned_at) > 30,
    )
    print("przetrzymane > 30 dni:", len(session.scalars(overdue).all()))

    # Książki z ceną powyżej 100 groszy za stronę
    pricey = select(Book).where(Book.price / Book.pages > 0.20)
    print("droższe niż 0.20/stronę:", len(session.scalars(pricey).all()))
```

```text
przetrzymane > 30 dni: 1
droższe niż 0.20/stronę: 11
```

> 🔬 **Pod maską** — `Loan.returned_at - Loan.loaned_at` na SQLite daje różnicę w dniach (SQLite reprezentuje daty jako teksty, ale SQLAlchemy dodaje typ `DATE` i konwertuje). Na PostgreSQL odejmowanie `date - date` zwraca `integer` (liczbę dni). **Uwaga na `timestamp`**: `timestamp - timestamp` w PostgreSQL zwraca `interval`, a nie liczbę — wtedy potrzebujesz `extract('epoch', ...)`. To typowa pułapka przy migracji SQLite → PostgreSQL.

### 5.8 `in_` z podzapytaniem

Bardzo częsty wzorzec: „daj mi A, gdzie A.id jest na liście z zapytania o B”.

```python
# sqla_library/q16_subquery_in.py
from sqlalchemy import select
from sqlalchemy.orm import Session

from models import Book, Loan
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    # Autorzy, których książki były kiedykolwiek wypożyczone
    borrowed_ids = select(Loan.book_id)
    stmt = select(Book).where(Book.id.in_(borrowed_ids)).order_by(Book.title)

    print(str(stmt.compile(compile_kwargs={"literal_binds": True})))
    print([b.title for b in session.scalars(stmt).unique()])
```

```sql
SELECT book.id, book.title, book.year, book.pages, book.price, book.author_id
FROM book
WHERE book.id IN (SELECT loan.book_id FROM loan)
ORDER BY book.title
```

Podzapytanie `select(Loan.book_id)` **nie jest wykonane osobno** — SQLAlchemy wstawia je do SQL jako podzapytanie. Jedna runda do bazy.

> 🧠 **Dlaczego tak jest** — W starym `Query` trzeba było napisać `subquery()` i ręcznie sięgać do `c`. W 2.0 `select()` w `in_()` jest automatycznie rozpoznawane jako podzapytanie. Jeśli chcesz **skorelowane** podzapytanie (odwołujące się do tabeli z zewnętrznego zapytania), użyj `.correlate(...)` lub `.scalar_subquery()` — o tym w sekcji 8.

> 🆕 **SQLAlchemy 2.1** — W 2.1 `selectinload` dostał parametr `omit_join` oraz `chunksize`, co pozwala kontrolować, jak `IN` z podzapytaniem jest budowane i dzielone na porcje. Szczegóły w module 11.

### 5.9 Łączenie warunków: `and_`, `or_`, `not_`

Domyślnie wiele `where()` łączy się przez `AND`:

```python
select(Book).where(Book.year > 1960).where(Book.pages < 300)
# → WHERE book.year > ? AND book.pages < ?
```

Dla `OR` i negacji potrzebujesz `or_`, `and_`, `not_`:

```python
# sqla_library/q17_logic.py
from sqlalchemy import and_, not_, or_, select
from sqlalchemy.orm import Session

from models import Book
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    def show(stmt) -> None:
        print(str(stmt.compile(compile_kwargs={"literal_binds": True})).split("WHERE")[-1])

    # AND — trzy sposoby zapisu, ten sam SQL
    show(select(Book).where(Book.year > 1960, Book.pages < 300))
    show(select(Book).where(Book.year > 1960).where(Book.pages < 300))
    show(select(Book).where(and_(Book.year > 1960, Book.pages < 300)))

    # OR
    show(select(Book).where(or_(Book.year < 1950, Book.pages > 350)))

    # NOT
    show(select(Book).where(not_(Book.title.startswith("The"))))

    # Kombinacja: (A AND B) OR C
    show(
        select(Book).where(
            or_(
                and_(Book.year < 1950, Book.pages > 250),
                Book.author_id == 2,
            )
        )
    )
```

```text
book.year > 1960 AND book.pages < 300
book.year > 1960 AND book.pages < 300
book.year > 1960 AND book.pages < 300
book.year < 1950 OR book.pages > 350
NOT (book.title LIKE 'The' || '%')
(book.year < 1950 AND book.pages > 250) OR book.author_id = 2
```

> ⚠️ **Pułapka** — Nie mieszaj operatorów bitowych `&` i `|` z `and_`/`or_`, jeśli nie kontrolujesz nawiasów. W Pythonie `&` ma wyższy priorytet niż `==`, więc:

```python
# ✗ ŹLE — Python policzy (Book.year > 1950) & Book.pages, potem porówna z 300
select(Book).where(Book.year > 1950 & Book.pages > 300)

# ✓ DOBRZE — jawnie, nawiasami
select(Book).where((Book.year > 1950) & (Book.pages > 300))
```

`&` i `|` w SQLAlchemy działają i są przydatne przy **dynamicznym** składaniu listy warunków (`functools.reduce(operator.and_, conditions)`), ale zawsze z nawiasami. Jeśli nie musisz — używaj `and_()` i `or_()`, bo czytają się jednoznacznie.

> 🧪 **Ćwiczenie** — Napisz zapytanie o książki, które spełniają **dokładnie jedno** z dwóch warunków: „wydane przed 1950” albo „mają więcej niż 300 stron”. *(Podpowiedź: XOR nie istnieje w SQL — zbuduj go z `or_` i `and_(not_(...), not_(...))`.)*

---

## 6. JOIN-y w ORM

JOIN to łączenie wierszy z dwóch tabel na podstawie wspólnej wartości. W ORM masz trzy sposoby na jego zapisanie — i warto znać wszystkie trzy, bo każdy jest lepszy w innej sytuacji.

### 6.1 JOIN po relacji

Najczystszy sposób. Skoro zadeklarowałeś `Book.author` jako `relationship()`, SQLAlchemy **wie**, jak połączyć te tabele.

```python
# sqla_library/q18_join_rel.py
from sqlalchemy import select
from sqlalchemy.orm import Session

from models import Author, Book
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    stmt = (
        select(Book)
        .join(Book.author)                  # ← po relacji
        .where(Author.country == "PL")
        .order_by(Book.title)
    )
    print(str(stmt.compile(compile_kwargs={"literal_binds": True})))
    print([b.title for b in session.scalars(stmt)])
```

```sql
SELECT book.id, book.title, book.year, book.pages, book.price, book.author_id
FROM book JOIN author ON author.id = book.author_id
WHERE author.country = 'PL'
ORDER BY book.title
```

```text
['Flames', 'His Master's Voice', 'Solaris', 'The Cyberiad', ...]
```

Zwróć uwagę na dwie rzeczy:

1. **Warunek `ON` został wywnioskowany** z `relationship()` — nie musiałeś pisać `Book.author_id == Author.id`.
2. **`Author` trafiła do `FROM`** automatycznie, bo pojawiła się w `where()`. SQLAlchemy widzi, że `Author` nie jest jeszcze w zapytaniu, i dokłada JOIN.

### 6.2 JOIN po kluczu obcym (bez relacji)

Jeśli nie masz relacji (albo nie chcesz jej używać), SQLAlchemy wywnioskuje warunek z klucza obcego:

```python
# sqla_library/q19_join_fk.py
from sqlalchemy import select
from sqlalchemy.orm import Session

from models import Author, Book
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    # SQLAlchemy widzi FK book.author_id → author.id i sam buduje ON
    stmt = select(Book).join(Author).where(Author.country == "US")
    print(str(stmt.compile(compile_kwargs={"literal_binds": True})).split("\n")[1])
```

```sql
FROM book JOIN author ON author.id = book.author_id
```

> ⚠️ **Pułapka** — Wnioskowanie po FK działa tylko wtedy, gdy **istnieje dokładnie jedna** ścieżka między tabelami. Jeśli `Review` ma dwa klucze obce do `Member` (np. `author_id` i `moderator_id`), `join(Member)` zgłosi `AmbiguousForeignKeysError`:

```text
sqlalchemy.exc.AmbiguousForeignKeysError: Can't determine join between 'review' and 'member';
tables have more than one foreign key constraint relationship between them.
```

Wtedy musisz podać warunek jawnie (6.3) albo użyć `foreign_keys=[...]` w relacji.

### 6.3 JOIN z jawnym warunkiem

Pełna kontrola. Przydatne, gdy:
- łączysz tabele bez klucza obcego (np. po zakresie dat),
- masz niejednoznaczne FK,
- chcesz samodzielnie zdecydować, co jest w `ON`, a co w `WHERE`.

```python
# sqla_library/q20_join_explicit.py
from sqlalchemy import select
from sqlalchemy.orm import Session

from models import Author, Book
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    stmt = (
        select(Book)
        .join(Author, Book.author_id == Author.id)
        .where(Author.country.in_(["PL", "GB"]))
        .order_by(Author.name, Book.year)
    )
    print(str(stmt.compile(compile_kwargs={"literal_binds": True})))
```

### 6.4 Co się dzieje z wynikiem: krotki encji

Jeśli w `select()` wstawisz dwie encje, dostaniesz wiersze z dwiema encjami:

```python
# sqla_library/q21_join_tuples.py
from sqlalchemy import func, select
from sqlalchemy.orm import Session

from models import Author, Book
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    # Encja + kolumna innej encji
    stmt = (
        select(Book, Author.name)
        .join(Book.author)
        .where(Author.country == "PL")
        .order_by(Book.year)
    )
    for book, author_name in session.execute(stmt):
        print(f"{book.year}  {author_name:<22} {book.title}")

    # Dwie encje
    stmt2 = select(Book, Author).join(Book.author).where(Book.year < 1930)
    for book, author in session.execute(stmt2):
        print(book.title, "→", author.name, f"({author.country})")
```

```text
1926  Agatha Christie        The Murder of Roger Ackroyd
1934  Agatha Christie        Murder on the Orient Express
...
```

> 🧠 **Dlaczego tak jest** — Każdy element `select()` staje się osobnym elementem `Row`. ORM rozpoznaje, że `Book` to encja, i zwraca **obiekt** (nie wiersz kolumn). Dzięki temu masz jednocześnie dostęp do danych z JOIN-a i do pełnoprawnego obiektu — który możesz zmodyfikować i zapisać.

> ⚠️ **Pułapka** — `session.scalars(stmt)` na zapytaniu `select(Book, Author.name)` zwróci **tylko obiekty `Book`**, po cichu gubiąc nazwę autora. To ta sama pułapka co w sekcji 3.4, tylko łatwiej ją przeoczyć przy JOIN-ach.

### 6.5 LEFT OUTER JOIN

`JOIN` (INNER) zwraca tylko wiersze, które mają dopasowanie po obu stronach. `LEFT OUTER JOIN` zwraca **wszystkie** wiersze z lewej tabeli, a dla braku dopasowania wstawia `NULL`.

Klasyczne zastosowanie: „autorzy, którzy nie mają żadnej książki”.

```python
# sqla_library/q22_left_join.py
from sqlalchemy import select
from sqlalchemy.orm import Session

from models import Author, Book
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    # ✗ ŹLE: warunek w WHERE zamienia LEFT JOIN w INNER JOIN
    wrong = (
        select(Author.name, Book.title)
        .outerjoin(Book, Book.author_id == Author.id)
        .where(Book.id.is_(None))          # ← TO JEST OK, patrz niżej
    )

    # Autorzy bez książek — klasyczny anti-join
    no_books = (
        select(Author)
        .outerjoin(Author.books)
        .where(Book.id.is_(None))
    )
    print("autorzy bez książek:", session.scalars(no_books).all())

    # Autorzy i liczba ich książek (0 dla tych bez książek)
    counts = (
        select(Author.name, func.count(Book.id))
        .outerjoin(Author.books)
        .group_by(Author.id)
        .order_by(func.count(Book.id).desc())
    )
    for name, count in session.execute(counts):
        print(f"{count:>2}  {name}")
```

```text
autorzy bez książek: []
  3  Ursula K. Le Guin
  3  Stanisław Lem
  3  Agatha Christie
  2  Wisława Szymborska
  2  Jorge Luis Borges
  2  Olga Tokarczuk
```

> ⚠️ **Pułapka** — To **najczęstszy błąd w JOIN-ach**, niezależnie od tego, czy piszesz SQL ręcznie, czy w ORM:

```python
# ✗ ŹLE — warunek na prawej tabeli w WHERE kasuje efekt LEFT JOIN
stmt = (
    select(Author.name, Book.title)
    .outerjoin(Book, Book.author_id == Author.id)
    .where(Book.year > 1960)      # ← dla autorów bez książek Book.year = NULL,
)                                  #    a NULL > 1960 to UNKNOWN → wiersz wypada!
```

Dla autorów bez książek `Book.year` jest `NULL`, a `NULL > 1960` daje `UNKNOWN`, nie `FALSE` — i wiersz nie przechodzi filtra. Efekt: dostałeś INNER JOIN, choć napisałeś `outerjoin`.

**Poprawka** — przenieś warunek do `ON`:

```python
# ✓ DOBRZE — warunek w ON zachowuje semantykę LEFT JOIN
stmt = (
    select(Author.name, Book.title)
    .outerjoin(Book, (Book.author_id == Author.id) & (Book.year > 1960))
)
```

Wyjątkiem jest warunek sprawdzający `NULL`, np. `.where(Book.id.is_(None))` — ten jest *zamierzony*, bo szukasz wierszy bez dopasowania.

> 🔬 **Pod maską** — różnica w SQL jest widoczna natychmiast:

```sql
-- Warunek w WHERE (efektywnie INNER JOIN)
FROM author LEFT OUTER JOIN book ON author.id = book.author_id
WHERE book.year > 1960

-- Warunek w ON (prawdziwy LEFT JOIN)
FROM author LEFT OUTER JOIN book ON author.id = book.author_id AND book.year > 1960
```

### 6.6 Duplikaty — `distinct()`

Gdy łączysz tabele i wybierasz tylko encję z „jeden” strony relacji, dostaniesz duplikaty:

```python
# sqla_library/q23_distinct.py
from sqlalchemy import select
from sqlalchemy.orm import Session

from models import Author, Book, Category
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    # Bez distinct — autor pojawia się raz na każdą książkę
    stmt = select(Author).join(Author.books).where(Book.year > 1960)
    print("bez distinct:", len(session.scalars(stmt).all()))

    # Z distinct — każdy autor raz
    stmt_d = select(Author).join(Author.books).where(Book.year > 1960).distinct()
    print("z distinct:  ", len(session.scalars(stmt_d).all()))

    # Przez relację N:M — każda książka raz na każdą kategorię
    stmt_c = (
        select(Book)
        .join(Book.categories)
        .where(Category.name == "Fiction")
        .distinct()
    )
    print("książki w Fiction:", len(session.scalars(stmt_c).all()))
```

```text
bez distinct: 13
z distinct:   4
książki w Fiction: 14
```

> 💡 **Analogia** — `JOIN` to rozłożenie listy zakupów na osobne paragony: jedna pozycja na produkt. `distinct()` to pytanie „ile różnych sklepów mam na tych paragonach?” — a nie „ile mam paragonów?”.

> 🧠 **Dlaczego tak jest** — `distinct()` w SQLAlchemy z encją ORM generuje `SELECT DISTINCT book.id, book.title, ...` — czyli porównuje **wszystkie kolumny**. To działa, ale na szerokich tabelach bywa kosztowne (baza musi sortować lub haszować całe wiersze). Alternatywa, gdy zależy ci na wydajności: `select(Author.id).distinct()` + drugie zapytanie, albo `exists()` zamiast JOIN-a (sekcja 8.4).

> ⚠️ **Pułapka** — `distinct()` **nie** zastępuje `unique()` przy `joinedload` kolekcji. To dwie różne warstwy:

| Problem | Warstwa | Rozwiązanie |
|---|---|---|
| Ten sam autor w wielu wierszach SQL (JOIN na książki) | SQL | `.distinct()` |
| Ten sam obiekt Pythona wielokrotnie na liście (joinedload) | ORM / Python | `.unique()` |

Możesz potrzebować obu jednocześnie. I możesz potrzebować **żadnego** — jeśli piszesz raport z kolumnami, duplikaty są tam zwykle pożądane.

### 6.7 JOIN a ładowanie relacji (zapowiedź modułu 11)

Kluczowe rozróżnienie, które bywa źródłem nieporozumień:

```python
# 1. JOIN wpływa na FROM i WHERE — ale NIE ładuje relacji.
stmt = select(Book).join(Book.author).where(Author.country == "PL")
book = session.scalars(stmt).first()
print(book.author.name)   # ← i tak poleci OSOBNY SELECT (leniwe ładowanie!)

# 2. Aby JOIN był też sposobem ładowania, użyj contains_eager() (sekcja 7.4)
#    albo joinedload() w options() (moduł 11).
```

> 🔬 **Pod maską** — pierwsze zapytanie generuje jeden SELECT z JOIN-em, ale dostęp do `book.author.name` generuje **drugi** SELECT:

```sql
-- Zapytanie 1
SELECT book.id, book.title, ... FROM book JOIN author ON author.id = book.author_id
WHERE author.country = ?

-- Zapytanie 2 (leniwe ładowanie relacji)
SELECT author.id, author.name, author.country FROM author WHERE author.id = ?
```

To jest dokładnie problem N+1, któremu poświęcony jest cały moduł 11. Zapamiętaj na razie: **JOIN w zapytaniu ≠ załadowana relacja**.

> 🧪 **Ćwiczenie** — Napisz zapytanie zwracające wszystkie książki w kategorii „Crime” wraz z nazwiskiem autora. Porównaj liczbę wierszy wyniku z liczbą książek — czy potrzebujesz `distinct()`?

---

## 7. `aliased()` i `contains_eager()`

### 7.1 Po co alias

Wyobraź sobie, że chcesz porównać tabelę z nią samą — np. „kategorie, które mają rodzica, i ich rodziców”. W SQL nie możesz napisać:

```sql
SELECT * FROM category JOIN category ON category.parent_id = category.id   -- ✗ niejednoznaczne!
```

Baza nie wie, o którą `category` chodzi w `ON`. Musisz nadać jednej z nich **alias**:

```sql
SELECT * FROM category AS child JOIN category AS parent ON child.parent_id = parent.id
```

W ORM robisz to samo przez `aliased()`.

### 7.2 Self-join

```python
# sqla_library/q24_selfjoin.py
from sqlalchemy import select
from sqlalchemy.orm import Session, aliased

from models import Category
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    child = aliased(Category)
    parent = aliased(Category)

    stmt = (
        select(child.name, parent.name)
        .join(parent, child.parent_id == parent.id)
        .order_by(parent.name, child.name)
    )
    print(str(stmt.compile(compile_kwargs={"literal_binds": True})))
    for child_name, parent_name in session.execute(stmt):
        print(f"{parent_name:<18} → {child_name}")
```

```sql
SELECT category_1.name, category_2.name
FROM category AS category_1
JOIN category AS category_2 ON category_1.parent_id = category_2.id
ORDER BY category_2.name, category_1.name
```

```text
Fiction            → Crime
Fiction            → Poetry
Fiction            → Science Fiction
Non-fiction        → Essay
```

SQLAlchemy sam nadał aliasy `category_1` i `category_2`.

> 💡 **Analogia** — Alias to identyfikator na plecaku. Kiedy w schowku stoją dwa identyczne plecaki, musisz oznaczyć je „P1” i „P2”, żeby powiedzieć „wyjmij z P1 to, co jest w P2”. Bez tego każdy rozkaz jest dwuznaczny.

### 7.3 „Wszyscy oprócz” — anty-join

Bardzo praktyczny wzorzec: „znajdź A, dla których nie istnieje B spełniające warunek”.

```python
# sqla_library/q25_antijoin.py
from sqlalchemy import select
from sqlalchemy.orm import Session, aliased

from models import Book, Loan
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    # Książki, które NIGDY nie były wypożyczone
    ever_loaned = aliased(Loan)
    stmt = (
        select(Book)
        .outerjoin(ever_loaned, ever_loaned.book_id == Book.id)
        .where(ever_loaned.id.is_(None))
        .order_by(Book.title)
    )
    print([b.title for b in session.scalars(stmt)])

    # Alternatywa przez NOT EXISTS — zwykle szybsza (sekcja 8.4)
    never = select(Book).where(~select(Loan.id).where(Loan.book_id == Book.id).exists())
    print("NOT EXISTS:", len(session.scalars(never).all()))
```

```text
['A Wizard of Earthsea', 'Ficciones', 'Here', ...]
NOT EXISTS: 5
```

Trzy równoważne sposoby na „nie istnieje”:

| Podejście | SQL | Kiedy |
|---|---|---|
| `outerjoin` + `IS NULL` | `LEFT JOIN ... WHERE x.id IS NULL` | Gdy potrzebujesz też danych z prawej tabeli |
| `NOT EXISTS` | `WHERE NOT EXISTS (SELECT ...)` | Zwykle najszybsze, czytelne intencją |
| `NOT IN (podzapytanie)` | `WHERE x NOT IN (SELECT ...)` | Uważaj na `NULL` w podzapytaniu! |

> ⚠️ **Pułapka** — `NOT IN` z podzapytaniem, które może zwrócić `NULL`, daje **zawsze pusty wynik**. Jeśli kolumna `loan.book_id` byłaby `nullable`, `Book.id NOT IN (SELECT loan.book_id FROM loan)` zwróciłoby zero wierszy — bo `id NOT IN (..., NULL)` jest `UNKNOWN`. `NOT EXISTS` tego problemu nie ma. Reguła: **do „nie istnieje” używaj `NOT EXISTS`, nie `NOT IN`**.

### 7.4 `contains_eager()`

Sytuacja: robisz JOIN, żeby filtrować po kolumnie z drugiej tabeli. JOIN już jest — więc po co ORM miałby robić drugie zapytanie, żeby dociągnąć relację? `contains_eager()` mówi: „użyj tego JOIN-a jako źródła danych dla relacji”.

```python
# sqla_library/q26_contains_eager.py
from sqlalchemy import select
from sqlalchemy.orm import Session, contains_eager

from models import Author, Book
from seed import build

factory = build()

with Session(factory.kw["bind"], expire_on_commit=False) as session:
    stmt = (
        select(Author)
        .join(Author.books)                       # JOIN już jest
        .where(Book.year > 2000)
        .options(contains_eager(Author.books))    # ← użyj go do załadowania
        .order_by(Author.name)
    )
    authors = session.scalars(stmt).unique().all()
    for author in authors:
        print(author.name, "→", [b.title for b in author.books])
        # brak dodatkowego SELECT-a — dane przyszły z JOIN-a
```

```text
Olga Tokarczuk → ['Flights', 'Drive Your Plow Over the Bones of the Dead']
Wisława Szymborska → ['Here']
```

> 🔬 **Pod maską** — Bez `contains_eager` powyższy kod wygenerowałby **dwa** zapytania: jedno z JOIN-em (do filtrowania), drugie po książki autorów. Z `contains_eager` jest **jedno**. To jedna z dwóch technik eliminacji N+1 (druga to `joinedload` z modułu 11).

> ⚠️ **Pułapka** — `contains_eager()` **wymaga**, żeby JOIN faktycznie istniał w zapytaniu. Jeśli go pominiesz, SQLAlchemy nie zgłosi błędu — po prostu dostaniesz puste kolekcje albo dane z innego JOIN-a. To cichy błąd. Druga pułapka: `contains_eager` nadpisuje strategię ładowania, więc **wyklucza się** z `joinedload` na tej samej relacji.

> 🆕 **SQLAlchemy 2.1** — `selectinload` dostał parametr `omit_join` (przydatny dla many-to-many: rezygnuje z JOIN-a na tabeli pośredniej, gdy i tak masz już dane) oraz `chunksize` (dzielenie `IN (...)` na porcje — ważne przy bardzo długich listach ID, bo PostgreSQL ma limit parametrów). `contains_eager` pozostaje głównym narzędziem, gdy JOIN już jest w zapytaniu.

> 🧪 **Ćwiczenie** — Zmień powyższy przykład tak, aby zwracał autorów **wraz z ich książkami wydanymi po 2000 roku** — tak, żeby `author.books` zawierało *tylko* te książki. *(Podpowiedź: to jest dokładnie to, przed czym ostrzega pułapka — `contains_eager` z filtrem w `where` zwróci tylko autorów z pasującymi książkami, ale kolekcja może zawierać wszystkie książki z JOIN-a. Sprawdź, co się dzieje, i zapisz wniosek.)*

---

## 8. Agregacje i sprawdzanie istnienia

### 8.1 `func.count()`

```python
# sqla_library/q27_count.py
from sqlalchemy import func, select
from sqlalchemy.orm import Session

from models import Book, Loan
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    # COUNT(*) — wszystkie wiersze
    total = session.scalar(select(func.count()).select_from(Book))

    # COUNT(kolumna) — pomija NULL-e
    with_year = session.scalar(select(func.count(Book.year)))

    # COUNT(DISTINCT kolumna)
    authors = session.scalar(select(func.count(func.distinct(Book.author_id))))

    print("wszystkich książek:", total)
    print("z rokiem:", with_year)
    print("różnych autorów:", authors)
```

```text
wszystkich książek: 15
z rokiem: 15
różnych autorów: 6
```

> ⚠️ **Pułapka** — `func.count()` **bez argumentu** wymaga `select_from()`, bo SQLAlchemy nie wie, z której tabeli liczyć:

```python
session.scalar(select(func.count()))                    # ✗ produkuje "SELECT count(*)" — bez FROM!
session.scalar(select(func.count()).select_from(Book))  # ✓
```

`select(func.count())` bez `select_from` na PostgreSQL zwróci **1** (bo `SELECT count(*)` bez `FROM` liczy jeden wiersz wirtualny) — i będziesz przekonany, że w tabeli jest jeden rekord. Zawsze dawaj `select_from()`.

### 8.2 `select_from()` — skąd liczyć

`select_from()` mówi SQLAlchemy, jaka tabela ma być w `FROM`, gdy nie wynika to z kolumn.

```python
# sqla_library/q28_select_from.py
from sqlalchemy import func, select
from sqlalchemy.orm import Session

from models import Loan, Member
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    # Ilu członków ma aktywne wypożyczenie?
    active_members = session.scalar(
        select(func.count(func.distinct(Loan.member_id))).where(Loan.returned_at.is_(None))
    )
    print("członków z aktywnym wypożyczeniem:", active_members)

    # Ilu członków NIE ma żadnego wypożyczenia? (wymaga select_from)
    members_with_loans = select(Loan.member_id)
    without = session.scalar(
        select(func.count())
        .select_from(Member)
        .where(Member.id.not_in(members_with_loans))
    )
    print("członków bez wypożyczeń:", without)
```

```text
członków z aktywnym wypożyczeniem: 5
członków bez wypożyczeń: 1
```

### 8.3 `group_by` i `having`

`GROUP BY` grupuje wiersze po wartościach kolumn. `HAVING` filtruje **grupy** (a `WHERE` filtruje wiersze **przed** grupowaniem).

```python
# sqla_library/q29_group.py
from sqlalchemy import func, select
from sqlalchemy.orm import Session

from models import Author, Book, Category, Review
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    # Liczba książek na autora — wszyscy, także bez książek
    stmt = (
        select(Author.name, func.count(Book.id).label("n"))
        .outerjoin(Author.books)
        .group_by(Author.id)
        .order_by(func.count(Book.id).desc(), Author.name)
    )
    for name, n in session.execute(stmt):
        print(f"{n:>2}  {name}")

    print()

    # Tylko kategorie z co najmniej 2 książkami (HAVING)
    stmt2 = (
        select(Category.name, func.count(Book.id).label("n"))
        .join(Category.books)
        .group_by(Category.id)
        .having(func.count(Book.id) >= 2)
        .order_by(func.count(Book.id).desc())
    )
    for name, n in session.execute(stmt2):
        print(f"{n:>2}  {name}")

    print()

    # Średnia ocena per książka — tylko książki z ≥2 opiniami
    stmt3 = (
        select(Book.title, func.avg(Review.rating).label("avg_rating"),
               func.count(Review.member_id).label("n"))
        .join(Book.reviews)
        .group_by(Book.id)
        .having(func.count(Review.member_id) >= 2)
        .order_by(func.avg(Review.rating).desc())
    )
    for title, avg, n in session.execute(stmt3):
        print(f"{float(avg):.2f}  ({n})  {title}")
```

```text
 3  Agatha Christie
 3  Stanisław Lem
 3  Ursula K. Le Guin
 2  Jorge Luis Borges
 2  Olga Tokarczuk
 2  Wisława Szymborska

 6  Fiction
 2  Crime
 1  Poetry        ← nie przechodzi HAVING (1 < 2)

 3.50  (2)  The Left Hand of Darkness
 3.00  (2)  Murder on the Orient Express
```

> 🧠 **Dlaczego tak jest** — `GROUP BY Author.id` (a nie `Author.name`) to celowy wybór. Grupowanie po kluczu głównym jest wydajniejsze (wąski typ, indeks) i — co ważniejsze — **poprawne** w PostgreSQL, który zna zasadę *functional dependency*: jeśli grupujesz po PK, możesz w `SELECT` wybrać dowolną kolumnę z tej tabeli, bo jest ona jednoznacznie wyznaczona przez PK. Grupowanie po `name` byłoby błędem, gdyby dwóch autorów miało to samo nazwisko.

> ⚠️ **Pułapka** — Na SQLite i MySQL `GROUP BY` jest „luźny”: możesz wybrać kolumnę, której nie ma w `GROUP BY`, i dostaniesz *jakąś* wartość z grupy (bez ostrzeżenia!). Ten sam kod na PostgreSQL zgłosi błąd:

```text
ERROR: column "book.title" must appear in the GROUP BY clause
       or be used in an aggregate function
```

To jedna z najważniejszych różnic między bazami. Kod, który „działa” na SQLite, może w ogóle nie ruszyć na PostgreSQL. Zasada: **grupuj po PK encji** — to działa wszędzie i jest semantycznie poprawne.

> 🔬 **Pod maską** — kolejność klauzul w SQL ma znaczenie i SQLAlchemy ją respektuje, niezależnie od kolejności wywołań w Pythonie:

```text
SELECT  →  FROM  →  JOIN  →  WHERE  →  GROUP BY  →  HAVING  →  ORDER BY  →  LIMIT
           (WHERE filtruje WIERSZE przed grupowaniem; HAVING filtruje GRUPY po grupowaniu)
```

Możesz napisać `.having(...).group_by(...)` — SQLAlchemy i tak wygeneruje poprawne `GROUP BY ... HAVING`. Ale dla czytelności pisz w kolejności SQL.

### 8.4 `exists()`

`EXISTS` to najszybszy sposób na pytanie „czy **cokolwiek** spełnia warunek?”. Baza może przerwać skanowanie po pierwszym dopasowaniu — nie musi liczyć ani materializować wyników.

```python
# sqla_library/q30_exists.py
from sqlalchemy import select
from sqlalchemy.orm import Session

from models import Book, Loan, Member
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    # Członkowie z co najmniej jednym aktywnym wypożyczeniem
    stmt = select(Member).where(
        select(Loan.id)
        .where(Loan.member_id == Member.id, Loan.returned_at.is_(None))
        .exists()
    )
    print(str(stmt.compile(compile_kwargs={"literal_binds": True})))
    print([m.name for m in session.scalars(stmt)])
```

```sql
SELECT member.id, member.email, member.name, member.joined_at
FROM member
WHERE EXISTS (SELECT loan.id FROM loan
              WHERE loan.member_id = member.id AND loan.returned_at IS NULL)
```

```text
['Alice Nowak', 'Bob Kowalski', 'Carol Wiśniewska', 'Erin Lewandowska', 'Frank Dąbrowski']
```

Zwróć uwagę na **korelację**: podzapytanie odwołuje się do `Member.id` z zapytania zewnętrznego. SQLAlchemy rozpoznaje to automatycznie — `Member` nie jest dublowane w `FROM` podzapytania.

### 8.5 Podzapytania skalarne

`.scalar_subquery()` zamienia `select()` w wyrażenie zwracające jedną wartość — możesz go użyć w `SELECT`, `WHERE` albo `ORDER BY`.

```python
# sqla_library/q31_scalar_subquery.py
from sqlalchemy import func, select
from sqlalchemy.orm import Session

from models import Book, Loan, Member
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    # Liczba wypożyczeń jako kolumna obok każdego członka
    loans_count = (
        select(func.count(Loan.id))
        .where(Loan.member_id == Member.id)
        .scalar_subquery()
    )
    stmt = (
        select(Member.name, loans_count.label("loans"))
        .order_by(loans_count.desc(), Member.name)
    )
    for name, n in session.execute(stmt):
        print(f"{n:>2}  {name}")

    print()

    # „Klienci, którzy zrobili co najmniej 3 wypożyczenia”
    stmt2 = select(Member).where(loans_count >= 3)
    print("z ≥3 wypożyczeniami:", [m.name for m in session.scalars(stmt2)])
```

```text
 4  Carol Wiśniewska
 3  Alice Nowak
 3  Bob Kowalski
 3  Erin Lewandowska
 2  Dave Zieliński
 2  Frank Dąbrowski

z ≥3 wypożyczeniami: ['Alice Nowak', 'Bob Kowalski', 'Carol Wiśniewska', 'Erin Lewandowska']
```

> 💡 **Analogia** — Podzapytanie skalarne to **przypis**: „obok każdego nazwiska dopisz, ile razy ta osoba coś wypożyczyła”. Liczysz w bazie, nie w Pythonie — i nie musisz ładować wszystkich wypożyczeń do pamięci, żeby je policzyć.

> 🔬 **Pod maską** — wygenerowany SQL:

```sql
SELECT member.name,
       (SELECT count(loan.id) FROM loan WHERE loan.member_id = member.id) AS loans
FROM member
ORDER BY loans DESC, member.name
```

Korelacja (`member.id`) działa automatycznie, bo `Member` jest w `FROM` zapytania zewnętrznego.

> 🧠 **Dlaczego tak jest** — Trzy podejścia do „policz powiązane rekordy” i ich charakterystyka:

| Podejście | SQL | Zalety | Wady |
|---|---|---|---|
| Podzapytanie skalarne | `(SELECT count(...) ...)` | Jedna runda, brak duplikatów | Może być wolne przy wielu korelacjach |
| `GROUP BY` + `JOIN` | `GROUP BY member.id` | Zwykle najszybsze | Duplikaty przy JOIN-ach; trzeba pamiętać o `outerjoin` |
| Dwa zapytania w Pythonie | `SELECT` + `SELECT ... IN (...)` | Proste | 2 rundy; ryzyko N+1 |

W praktyce dla list z licznikami wygrywa zwykle `GROUP BY`; dla pojedynczych wartości — podzapytanie skalarne. Zawsze mierz (moduł 17).

> 🧪 **Ćwiczenie** — Napisz zapytanie zwracające **książki nigdy nie zrecenzowane** trzema sposobami: przez `outerjoin` + `IS NULL`, przez `NOT EXISTS` i przez `NOT IN (podzapytanie)`. Sprawdź, czy wszystkie dają ten sam wynik, i zmierz czas wykonania.

---

## 9. Zagadnienia zaawansowane

### 9.1 `union_all`

`UNION` łączy wyniki dwóch zapytań o tym samym kształcie kolumn. `UNION ALL` nie usuwa duplikatów (i jest szybszy).

```python
# sqla_library/q32_union.py
from sqlalchemy import select, union_all
from sqlalchemy.orm import Session

from models import Book
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    # Stare i bardzo nowe książki — jeden raport
    stmt = union_all(
        select(Book.id, Book.title, Book.year).where(Book.year < 1950),
        select(Book.id, Book.title, Book.year).where(Book.year > 2000),
    ).order_by(Book.year)

    for book_id, title, year in session.execute(stmt):
        print(f"{year}  {title}")
```

```text
1926  The Murder of Roger Ackroyd
1934  Murder on the Orient Express
1939  And Then There Were None
1944  Ficciones
1949  The Aleph
2007  Flights
2009  Here
2009  Drive Your Plow Over the Bones of the Dead
```

> ⚠️ **Pułapka** — `UNION` wymaga, żeby oba zapytania miały **tę samą liczbę kolumn o zgodnych typach**. SQLAlchemy nie sprawdzi tego za ciebie — baza zgłosi błąd przy wykonaniu. Druga pułapka: **nazwy kolumn pochodzą z pierwszego zapytania**. Jeśli drugie ma inne aliasy, wynik nadal użyje tych z pierwszego. Trzecia: `ORDER BY` i `LIMIT` można stosować tylko do całości `UNION`, nie do pojedynczych części (chyba że opakujesz je w podzapytania).

Możesz też robić `UNION` na encjach:

```python
stmt = union_all(select(Book).where(Book.year < 1950), select(Book).where(Book.year > 2000))
books = session.scalars(stmt).unique().all()   # unique() — bo ORM deduplikuje encje
```

> 🧠 **Dlaczego tak jest** — `union_all` na encjach wymaga `.unique()`, bo ORM nie wie, czy chcesz duplikaty usunięte po tożsamości. Jeśli w obu częściach jest ta sama książka, `union_all` zwróci ją dwa razy — i ORM zwróci dwa razy ten sam obiekt Pythona na liście.

### 9.2 `yield_per` i strumieniowe czytanie

Domyślnie `session.scalars(stmt).all()` pobiera **wszystkie** wiersze do pamięci jako lista obiektów. Przy 500 000 wierszy to setki megabajtów. `yield_per` każe ORM-owi czytać w porcjach.

```python
# sqla_library/q33_yield_per.py
from sqlalchemy import select
from sqlalchemy.orm import Session

from models import Book
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    stmt = select(Book).order_by(Book.id).execution_options(yield_per=3)

    for i, book in enumerate(session.scalars(stmt), start=1):
        print(f"{i:>2}. {book.title}")
        # pamięć trzyma maksymalnie ~3 obiekty naraz (+ te w identity map)
```

```text
 1. The Left Hand of Darkness
 2. The Dispossessed
 3. A Wizard of Earthsea
 4. Solaris
...
```

| Sposób | Pamięć | Kiedy |
|---|---|---|
| `.all()` | Cały wynik | Do ~10 000 wierszy, wynik potrzebny w całości |
| `for x in session.scalars(stmt)` | Cały wynik (kursor) | Uwaga: to nie jest strumień! |
| `.execution_options(yield_per=N)` | ~N obiektów | Eksport, migracje, przetwarzanie wsadowe |
| `session.scalars(stmt).partitions(N)` | ~N obiektów | To samo, ale z jawnymi porcjami |

> ⚠️ **Pułapka** — Iteracja `for x in session.scalars(stmt)` **nie** jest strumieniowa w rozumieniu „stała pamięć”. Kursor DBAPI może buforować cały wynik po stronie sterownika (`psycopg` domyślnie tak robi). Dopiero `yield_per` włącza tryb serwerowego kursora i porcjowanie. Druga pułapka: `yield_per` **nie współpracuje** z `joinedload` na kolekcjach — SQLAlchemy zgłosi `InvalidRequestError`. Do strumieniowania używaj `selectinload` albo nie ładuj relacji wcale.

> 🧪 **Ćwiczenie** — Zmierz `tracemalloc.get_traced_memory()` dla `session.scalars(select(Book)).all()` i dla tego samego zapytania z `yield_per=5`. Na 15 wierszach różnica będzie śmieszna — ale mechanizm zrozumiesz. Wyniki dla 1 000 000 wierszy zobaczysz w module 17.

### 9.3 `populate_existing`

Domyślnie, gdy zapytanie zwróci obiekt, który **już jest** w identity map, SQLAlchemy **nie nadpisuje** jego atrybutów (chyba że są wygaszone przez `expire`). To wydajne, ale czasem chcesz wymusić odświeżenie:

```python
# sqla_library/q34_populate_existing.py
from sqlalchemy import select, update
from sqlalchemy.orm import Session

from models import Book
from seed import build

factory = build()

with Session(factory.kw["bind"], expire_on_commit=False) as session:
    book = session.get(Book, 1)
    print("przed:", book.title)

    # Zmiana w bazie z pominięciem sesji (np. z innego procesu)
    session.execute(
        update(Book).where(Book.id == 1).values(title="Zmienione poza ORM")
    )

    # Odczyt bez populate_existing — ORM widzi, że obiekt już jest, i nie odświeża
    same = session.scalars(select(Book).where(Book.id == 1)).one()
    print("bez populate_existing:", same.title)

    # Z populate_existing — atrybuty zostają nadpisane z bazy
    fresh = session.scalars(
        select(Book).where(Book.id == 1).execution_options(populate_existing=True)
    ).one()
    print("z populate_existing:  ", fresh.title)
```

```text
przed: The Left Hand of Darkness
bez populate_existing: The Left Hand of Darkness
z populate_existing:   Zmienione poza ORM
```

> 🧠 **Dlaczego tak jest** — Identity map to kontrakt: „w ramach tej sesji obiekt o danym PK jest **jednym** obiektem Pythona i ma spójny stan”. Bez `populate_existing` ORM chroni ten kontrakt — nie nadpisuje stanu, bo gdzieś indziej w kodzie mogłeś już na tym obiekcie pracować. `populate_existing` to jawne powiedzenie: „wiem, co robię — nadpisz”.

> 💡 **Analogia** — Identity map to tablica ogłoszeń w biurze: raz przybita kartka nie jest zdejmowana przy każdym nowym zgłoszeniu. `populate_existing` to polecenie „zerwij i przybij nową”.

### 9.4 `with_for_update`

`SELECT ... FOR UPDATE` blokuje wiersze na czas transakcji, żeby nikt inny ich nie zmienił. To podstawa „zabierz ostatnią sztukę z magazynu bez wyścigu”.

```python
# sqla_library/q35_for_update.py
from sqlalchemy import select
from sqlalchemy.orm import Session

from models import Book
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    stmt = select(Book).where(Book.id == 1).with_for_update()
    print(str(stmt.compile(compile_kwargs={"literal_binds": True})))

    # Warianty (PostgreSQL):
    #   .with_for_update(nowait=True)       → błąd zamiast czekania na blokadę
    #   .with_for_update(skip_locked=True)  → pomiń zablokowane wiersze
    #   .with_for_update(read=True)         → FOR SHARE (blokada współdzielona)
    #   .with_for_update(of=Book)           → blokuj tylko wiersze z tej tabeli
```

```sql
SELECT book.id, book.title, book.year, book.pages, book.price, book.author_id
FROM book
WHERE book.id = 1
FOR UPDATE
```

> ⚠️ **Pułapka** — **SQLite ignoruje `FOR UPDATE`** — nie zgłasza błędu, po prostu nie dodaje klauzuli. Kod, który „działa” na SQLite, na produkcji z PostgreSQL zachowa się inaczej (i odwrotnie: kod testowany na PostgreSQL może dawać inne wyniki na SQLite). Dlatego testy współbieżności **muszą** działać na docelowej bazie. Pełna dyskusja w module 14.

### 9.5 `with_loader_criteria` — szybka wzmianka

Zaawansowane narzędzie do globalnego filtrowania relacji — np. „wszystkie ładowane wypożyczenia mają być niezwrócone”. Szczegóły w module 11.

```python
from sqlalchemy.orm import with_loader_criteria

stmt = (
    select(Member)
    .options(with_loader_criteria(Loan, Loan.returned_at.is_(None)))
)
```

### 9.6 `Select` jako typ — pełne typowanie

```python
# sqla_library/q36_typing.py
from sqlalchemy import Select, select
from sqlalchemy.orm import Session

from models import Book
from seed import build


def active_books(min_year: int) -> Select[tuple[Book]]:
    """Zapytanie o książki wydane nie wcześniej niż `min_year`.

    Zwracamy sam obiekt Select — nikt go jeszcze nie wykonał.
    To pozwala komponować zapytania (sekcja 11).
    """
    return select(Book).where(Book.year >= min_year).order_by(Book.title)


factory = build()
with Session(factory.kw["bind"]) as session:
    books: list[Book] = list(session.scalars(active_books(2000)))
    print(len(books), "książek")
```

`Select[tuple[Book]]` to dokładny typ: „zapytanie, którego wiersze są krotkami zawierającymi jeden `Book`”. Dla `select(Book, Author)` typ to `Select[tuple[Book, Author]]`. Dla `select(Book.title, Book.year)` — `Select[tuple[str, int | None]]`.

> 🧠 **Dlaczego tak jest** — Funkcja zwracająca `Select` to **specyfikacja zapytania**, a nie jego wykonanie. To fundamentalna różnica względem `session.query(Book)`, które w 1.x od razu zwracało obiekt wykonawczy. Dzięki temu możesz zapytania budować warstwowo — o tym w sekcji 11.

---

## 10. DML przez ORM: INSERT, UPDATE, DELETE

### 10.1 Po co DML przez ORM

Do tej pory modyfikowałeś dane przez obiekty: `session.add(book)`, `book.price = 10`, `session.delete(book)`. To jest **Unit of Work** z modułu 08 — wygodny, ale wymaga **załadowania obiektu do pamięci**.

Czasem to jest marnotrawstwo:

```python
# ✗ ŹLE — dla zmiany ceny 5000 książek ładujesz 5000 obiektów do pamięci
books = session.scalars(select(Book).where(Book.year < 1950)).all()
for book in books:
    book.price = book.price * Decimal("1.10")
session.commit()
# Efekt: 1 SELECT + 5000 UPDATE-ów + 5000 obiektów w pamięci
```

ORM-owy DML pozwala powiedzieć bazie: **„zmień to jednym poleceniem”** — i nadal korzystać z dobrodziejstw ORM (mapowanie nazw atrybutów, obsługa `RETURNING`, synchronizacja identity map).

> 💡 **Analogia** — Unit of Work to pójście do każdego pokoju i ręczne przestawienie krzesła. ORM-owy DML to wysłanie jednego e-maila: „przestawcie krzesła we wszystkich pokojach na drugim piętrze”. Oba sposoby są poprawne — ale drugi jest znacznie szybszy, gdy pokoi jest 5000.

### 10.2 INSERT

```python
# sqla_library/q37_insert.py
from decimal import Decimal

from sqlalchemy import insert, select
from sqlalchemy.orm import Session

from models import Book
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    stmt = insert(Book).values(
        title="Nowa książka",
        year=2026,
        pages=250,
        price=Decimal("59.90"),
        author_id=1,
    )
    result = session.execute(stmt)
    print("wstawiono wierszy:", result.rowcount)

    # ID nowego wiersza — dostępne przez lastrowid / inserted_primary_key
    print("nowe id:", result.inserted_primary_key)

    session.commit()
```

```text
wstawiono wierszy: 1
nowe id: [16]
```

> 🔬 **Pod maską** — `insert(Book).values(...)` generuje `INSERT INTO book (title, year, pages, price, author_id) VALUES (...)`. Nazwy **atrybutów ORM** są mapowane na nazwy **kolumn** — jeśli masz `mapped_column("db_name")` z inną nazwą, SQLAlchemy użyje `db_name`.

> ⚠️ **Pułapka** — `insert(Book).values(...)` **nie tworzy obiektu `Book`** i nie dodaje go do identity map. Jeśli w tej samej sesji zrobisz potem `select(Book).where(Book.title == "Nowa książka")`, obiekt powstanie wtedy. Jeśli wcześniej w sesji była książka o id 16 (np. usunięta, ale wciąż w identity map) — dostaniesz ostrzeżenie o konflikcie tożsamości.

### 10.3 UPDATE

```python
# sqla_library/q38_update.py
from decimal import Decimal

from sqlalchemy import select, update
from sqlalchemy.orm import Session

from models import Author, Book
from seed import build

factory = build()

with Session(factory.kw["bind"], expire_on_commit=False) as session:
    # 1. Prosty UPDATE z wartością stałą
    stmt = update(Book).where(Book.year < 1950).values(price=Decimal("29.90"))
    result = session.execute(stmt)
    print("podrożono/zmiększono:", result.rowcount)

    # 2. UPDATE z odwołaniem do innych kolumn
    stmt2 = (
        update(Book)
        .where(Book.year >= 1950)
        .values(pages=Book.pages + 1)
    )
    print("zwiększono strony:", session.execute(stmt2).rowcount)

    # 3. UPDATE z JOIN-em po podzapytaniu (dialektowo przenośny)
    pl_author_ids = select(Author.id).where(Author.country == "PL")
    stmt3 = update(Book).where(Book.author_id.in_(pl_author_ids)).values(year=Book.year)
    print("PL:", session.execute(stmt3).rowcount)

    session.commit()

    # Sprawdź efekt
    print(session.scalars(select(Book).where(Book.year < 1950)).all())
```

```text
podrożono/zmiększono: 5
zwiększono strony: 10
PL: 5
[Book(id=7, title='Murder on the Orient Express'), ...]
```

### 10.4 DELETE

```python
# sqla_library/q39_delete.py
from sqlalchemy import delete, select
from sqlalchemy.orm import Session

from models import Book, Loan
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    # Usuń książki nigdy nie wypożyczone
    loaned_ids = select(Loan.book_id)
    stmt = delete(Book).where(Book.id.not_in(loaned_ids))
    print("usunięto książek:", session.execute(stmt).rowcount)
    session.commit()

    print("zostało:", session.scalar(select(__import__("sqlalchemy").func.count(Book.id))))
```

```text
usunięto książek: 5
zostało: 10
```

> ⚠️ **Pułapka** — `delete(Book).where(...)` **nie** uruchomi kaskad Pythona z `relationship(cascade="all, delete-orphan")`. Kaskady ORM działają tylko wtedy, gdy ORM wie o obiektach — a tutaj ich nie ładował. Jeśli masz w bazie `ON DELETE CASCADE`, baza posprząta sama. Jeśli nie — dostaniesz `IntegrityError` albo (gorzej) osierocone wiersze. To jedna z najważniejszych różnic między `session.delete(obj)` a `session.execute(delete(...))`. Szczegóły w module 14.

### 10.5 `synchronize_session` — sedno sprawy

Teraz najważniejsza część tego rozdziału.

Wyobraź sobie, że w sesji masz załadowany obiekt `book` z ceną `39.90`. Wykonujesz `UPDATE book SET price = 29.90 WHERE year < 1950` — i ten obiekt pasuje do warunku. Co ma się stać z `book.price` w pamięci?

- Nic? Wtedy `book.price` wciąż pokazuje `39.90` — **kłamstwo**. Obiekt jest „nieaktualny”.
- Zaktualizować? Trzeba wiedzieć, **które** obiekty pasowały — a to wymaga dodatkowej pracy.

Dokładnie o tym decyduje `synchronize_session`.

| Wartość | Jak działa | Koszt |
|---|---|---|
| `'auto'` (domyślne) | Na bazach z `RETURNING` → `'fetch'`; na pozostałych → `'evaluate'` | Zależny od wybranej strategii |
| `'fetch'` | Dodaje `RETURNING id` do UPDATE/DELETE (albo `SELECT` przed), potem odświeża/wygasza pasujące obiekty z identity map | 1 runda, ale dodatkowe kolumny w `RETURNING` |
| `'evaluate'` | Ocenia warunek `WHERE` w Pythonie na obiektach z sesji; **zero dodatkowych rund SQL** | Może rzucić `UnevaluatableError`; odświeża obiekty pojedynczo, jeśli są wygaszone |
| `False` | Nic nie robi | Zero kosztu, ale obiekty w sesji są **nieaktualne** |

```python
# sqla_library/q40_sync_session.py
from decimal import Decimal

from sqlalchemy import select, update
from sqlalchemy.orm import Session

from models import Book
from seed import build

factory = build()

with Session(factory.kw["bind"], expire_on_commit=False) as session:
    book = session.get(Book, 4)          # Solaris, rok 1961, cena 42.00
    print("przed:      ", book.price)

    # 1. Domyślnie ('auto' → 'fetch' na SQLite z RETURNING)
    stmt = update(Book).where(Book.id == 4).values(price=Decimal("99.00"))
    session.execute(stmt)
    print("po 'auto':  ", book.price)

    # 2. Wyłączone — obiekt NIE jest aktualizowany
    stmt = update(Book).where(Book.id == 4).values(price=Decimal("11.00"))
    session.execute(stmt, execution_options={"synchronize_session": False})
    print("po False:   ", book.price, "← nieaktualne!")

    # 3. Jawnie 'fetch'
    stmt = update(Book).where(Book.id == 4).values(price=Decimal("55.00"))
    session.execute(stmt, execution_options={"synchronize_session": "fetch"})
    print("po 'fetch': ", book.price)
```

```text
przed:       42.00
po 'auto':   99.00
po False:    99.00   ← nieaktualne!
po 'fetch':  55.00
```

Widać różnicę gołym okiem: po `False` obiekt w pamięci pokazuje `99.00`, a w bazie jest `11.00`. **To jest cichy błąd** — żaden wyjątek, żadne ostrzeżenie.

> 🔬 **Pod maską** — z `'fetch'` SQLAlchemy dokłada `RETURNING` do UPDATE:

```sql
-- 'auto' / 'fetch' na SQLite i PostgreSQL:
UPDATE book SET price = ? WHERE book.id = ? RETURNING book.id

-- 'evaluate': dokładnie to samo, bez RETURNING
UPDATE book SET price = ? WHERE book.id = ?
```

Na bazach bez `RETURNING` (MySQL) strategia `'fetch'` robi dodatkowy `SELECT id FROM book WHERE ...` **przed** UPDATE-em.

> ⚠️ **Pułapka** — `'evaluate'` może się **nie udać** na wyrażeniach, których nie umie policzyć w Pythonie:

```python
stmt = update(Book).where(func.lower(Book.title) == "solaris").values(price=Decimal("1"))
session.execute(stmt, execution_options={"synchronize_session": "evaluate"})
```

```text
sqlalchemy.exc.InvalidRequestError: Could not evaluate current criteria in Python:
"Can't evaluate expression: <Function lower at 0x...>". Specify 'fetch' or False
for the synchronization session.
```

Błąd mówi wprost, co zrobić: użyj `'fetch'` albo `False`. Zwróć uwagę, że **domyślne `'auto'` by tego nie zgłosiło** — bo na SQLite/PostgreSQL wybrałoby `'fetch'`.

> 🧠 **Dlaczego tak jest** — Trzy scenariusze, trzy właściwe odpowiedzi:

| Scenariusz | Zalecenie |
|---|---|
| W sesji masz obiekty, które mogą pasować do `WHERE` | `'auto'` (domyślne) lub `'fetch'` |
| Sesja jest „świeża” — nie ładowałeś nic z tego obszaru | `False` — oszczędzasz rundę i `RETURNING` |
| Bulk operacja na 100 000 wierszy w skrypcie wsadowym | `False` — identity map nie ma znaczenia |
| Baza bez `RETURNING` i złożony warunek `WHERE` | `'fetch'` (bo `'evaluate'` padnie) |
| Sesja z wieloma wygaszonymi obiektami | `'fetch'` — `'evaluate'` odświeży je pojedynczo (`SELECT` per obiekt!) |

> ⚠️ **Pułapka** — Dokumentacja ostrzega wyraźnie: `'evaluate'` należy unikać, gdy sesja ma **dużo wygaszonych obiektów**, bo żeby ocenić warunek `WHERE` w Pythonie, musi je najpierw odświeżyć — a to jeden `SELECT` na obiekt. Dokładnie odwrotność tego, czego chciałeś.

### 10.6 `insert().returning()` — wstaw i zwróć

```python
# sqla_library/q41_returning.py
from decimal import Decimal

from sqlalchemy import insert
from sqlalchemy.orm import Session

from models import Book
from seed import build

factory = build()

with Session(factory.kw["bind"]) as session:
    # Zwróć pełny obiekt ORM
    stmt = insert(Book).values(
        title="Powrót z gwiazd",
        year=1961,
        pages=180,
        price=Decimal("39.00"),
        author_id=2,
    ).returning(Book)

    new_book: Book = session.scalars(stmt).one()
    print(type(new_book).__name__, new_book.id, new_book.title)
    # new_book jest pełnym obiektem ORM — możesz go dalej modyfikować

    session.commit()

    # Możesz też zwrócić wybrane kolumny
    stmt2 = insert(Book).values(
        title="Niezwyciężony", year=1964, pages=190,
        price=Decimal("38.00"), author_id=2,
    ).returning(Book.id, Book.title)
    row = session.execute(stmt2).one()
    print(row.id, row.title)
    session.commit()
```

```text
Book 16 Powrót z gwiazd
17 Niezwyciężony
```

> 🔬 **Pod maską** — SQLite obsługuje `RETURNING` od wersji 3.35 (marzec 2021). Sprawdź swoją:

```bash
python -c "import sqlite3; print(sqlite3.sqlite_version)"
```

Jeśli masz starszą, `RETURNING` nie zadziała i musisz użyć `result.inserted_primary_key` albo `session.flush()` po `session.add()`.

> 🧠 **Dlaczego tak jest** — `insert().returning(Book)` jest w 2.0 **znacznie potężniejsze** niż w 1.4. Wcześniej `RETURNING` obsługiwało tylko kolumny. Teraz zwraca **pełne obiekty ORM**, dodane do identity map, gotowe do dalszej pracy. To jedna z tych zmian, które sprawiają, że migracja na 2.0 naprawdę się opłaca.

> 🆕 **SQLAlchemy 2.1** — `RETURNING` z encjami obsługuje dodatkowe opcje ładowania (`load_only`, `selectinload`) w ograniczonym zakresie, a dla `postgresql://` domyślnym driverem jest `psycopg` (v3), który obsługuje `RETURNING` z `executemany` sprawniej niż `psycopg2`.

### 10.7 Bulk: `session.execute(insert(...), list_of_dicts)`

To najszybszy sposób wstawienia wielu wierszy **przez ORM** — i najczęściej mylony z Core'owym `executemany`.

```python
# sqla_library/q42_bulk_insert.py
from decimal import Decimal

from sqlalchemy import func, insert, select
from sqlalchemy.orm import Session

from models import Book
from seed import build

factory = build()

ROWS = [
    {
        "title": f"Książka {i}",
        "year": 2000 + (i % 25),
        "pages": 100 + i,
        "price": Decimal("19.90") + i,
        "author_id": 1 + (i % 6),
    }
    for i in range(1000)
]

with Session(factory.kw["bind"]) as session:
    session.execute(insert(Book), ROWS)      # ← lista dictów jako DRUGI argument
    session.commit()

    print("w bazie:", session.scalar(select(func.count(Book.id))))
```

```text
w bazie: 1015
```

Trzy rzeczy, które warto tu wiedzieć:

1. **Klucze w dictach to nazwy atrybutów ORM**, nie nazwy kolumn. Jeśli masz `mapped_column("db_title")`, piszesz `"title"`, nie `"db_title"`.
2. **To jest `insertmanyvalues`** — nowy mechanizm 2.0, który przy bazach z `RETURNING` potrafi wstawić tysiące wierszy w jednym `INSERT ... VALUES (...), (...), (...)`.
3. **Obiekty NIE trafiają do identity map** (chyba że użyjesz `returning(Book)`).

```python
# Wariant z RETURNING — obiekty ORM wracają do sesji
with Session(factory.kw["bind"]) as session:
    books = session.scalars(insert(Book).returning(Book), ROWS).all()
    print(len(books), "obiektów ORM w identity map")
    session.commit()
```

> ⚠️ **Pułapka** — `session.execute(insert(Book), [{"a": 1}, {"b": 2}])` — dicty o **różnych kluczach** — działają w ORM (SQLAlchemy podzieli je na grupy i wygeneruje kilka `INSERT`-ów). Ale w Core (`{"dml_strategy": "raw"}`) zadziała tylko pierwszy dict — pozostałe zostaną zignorowane bez ostrzeżenia. To bardzo częsta pomyłka przy migracji z 1.x, gdzie `insert()` w sesji zachowywał się „po core'owemu”.

### 10.8 Porównanie wydajności z `add_all`

```python
# sqla_library/q43_bulk_compare.py
import time
from decimal import Decimal

from sqlalchemy import delete, insert
from sqlalchemy.orm import Session

from models import Book
from seed import build

factory = build()

ROWS = [
    {
        "title": f"T{i}",
        "year": 2000,
        "pages": 100,
        "price": Decimal("10.00"),
        "author_id": 1,
    }
    for i in range(5000)
]


def variant_add_all(session: Session) -> None:
    """Obiekty ORM jeden po drugim — pełny cykl życia encji."""
    session.add_all([Book(**row) for row in ROWS])
    session.commit()


def variant_orm_bulk(session: Session) -> None:
    """ORM bulk insert — bez tworzenia obiektów."""
    session.execute(insert(Book), ROWS)
    session.commit()


def variant_orm_bulk_returning(session: Session) -> None:
    """ORM bulk insert z RETURNING — obiekty wracają, ale bez SELECT-ów."""
    session.scalars(insert(Book).returning(Book), ROWS).all()
    session.commit()


for name, fn in [
    ("add_all", variant_add_all),
    ("bulk insert", variant_orm_bulk),
    ("bulk + RETURNING", variant_orm_bulk_returning),
]:
    with Session(factory.kw["bind"]) as session:
        session.execute(delete(Book))
        session.commit()
        start = time.perf_counter()
        fn(session)
        elapsed = time.perf_counter() - start
        print(f"{name:<20} {elapsed * 1000:>8.1f} ms")
```

Przykładowe wyniki (SQLite, 5000 wierszy, laptop klasy średniej):

```text
add_all                820.3 ms
bulk insert             74.1 ms
bulk + RETURNING       102.7 ms
```

Rzędy wielkości są powtarzalne; konkretne liczby zależą od sprzętu i bazy.

> 🧠 **Dlaczego tak jest** — `add_all` dla każdego obiektu: tworzy instancję klasy, ustawia atrybuty przez instrumentację, dodaje do `session.new`, sortuje operacje w Unit of Work, generuje `INSERT` per obiekt (chyba że `insertmanyvalues` się załączy — a załącza się tylko przy prostych przypadkach), a potem jeszcze odświeża obiekty (`expire_on_commit=True`) — czyli 5000 dodatkowych `SELECT`-ów przy następnym dostępie. Bulk insert tego wszystkiego nie robi: to jedna komenda, jedna runda (albo kilka, jeśli SQLAlchemy podzieli paczkę).

| Aspekt | `add_all` | `session.execute(insert(...), rows)` |
|---|---|---|
| Tworzy obiekty ORM | Tak | Nie (chyba że `returning(Book)`) |
| Uruchamia zdarzenia `before_insert` | Tak | Nie |
| Uruchamia `@validates` | Tak | Nie |
| Wypełnia `default` z Pythona | Tak | Nie (trzeba podać w dictach) |
| Wstawia do identity map | Tak | Nie |
| Kaskady ORM | Tak | Nie |
| Prędkość | Wolno | Szybko |

> ⚠️ **Pułapka** — Bulk insert **pomija zdarzenia ORM** (`before_insert`, `@validates`) i **nie uzupełnia** wartości z `default=` w Pythonie. Jeśli twój model polega na `@validates` do normalizacji e-maila albo na `default=lambda: datetime.now(UTC)` — bulk insert to ominie. Efekt: w bazie lądują dane w formacie, którego się nie spodziewasz. Rozwiązania: (a) uzupełniaj dicty w Pythonie przed wysłaniem, (b) użyj `server_default` zamiast `default`, (c) zostań przy `add_all` dla tych encji.

> 🧪 **Ćwiczenie** — Dodaj do modelu `@validates("title")`, który rzuca wyjątek przy tytule zaczynającym się od „X”. Sprawdź, że `add_all` zgłasza błąd, a bulk insert — nie.

---

## 11. Czytelność: kompozycja zapytań

Po dziesięciu sekcjach wiesz już, jak *napisać* zapytanie. Teraz o tym, jak napisać je tak, żeby dało się je przeczytać za pół roku.

### 11.1 Zapytania w funkcjach

Funkcja zwracająca `Select` to **specyfikacja**, nie wykonanie. Możesz ją wywołać, dodać warunek i wykonać — albo nie.

```python
# sqla_library/q44_compose.py
from sqlalchemy import Select, select
from sqlalchemy.orm import Session

from models import Author, Book
from seed import build


def books_base() -> Select[tuple[Book]]:
    """Bazowe zapytanie o książki — z podstawowym porządkowaniem."""
    return select(Book).order_by(Book.title)


def only_after(year: int) -> Select[tuple[Book]]:
    return books_base().where(Book.year > year)


def by_country(country: str) -> Select[tuple[Book]]:
    return books_base().join(Book.author).where(Author.country == country)


def long_books(min_pages: int) -> Select[tuple[Book]]:
    return books_base().where(Book.pages >= min_pages)


factory = build()
with Session(factory.kw["bind"]) as session:
    print([b.title for b in session.scalars(only_after(2000))])
    print([b.title for b in session.scalars(by_country("PL"))])
    print([b.title for b in session.scalars(long_books(300))])

    # Łączenie specyfikacji
    combined = books_base().where(Book.year > 1900).where(Book.pages > 200)
    print(len(session.scalars(combined).all()))
```

### 11.2 Kompozycja filtrów

Zamiast łańcucha `if`-ów, buduj listę warunków i połącz ją raz:

```python
# sqla_library/q45_filters.py
from sqlalchemy import ColumnElement, and_, select
from sqlalchemy.orm import Session

from models import Author, Book
from seed import build


def search_books(
    title_part: str | None = None,
    min_year: int | None = None,
    max_pages: int | None = None,
    country: str | None = None,
) -> Select[tuple[Book]]:
    """Buduje zapytanie z opcjonalnych filtrów — bez if-ów w łańcuchu."""
    stmt = select(Book).order_by(Book.title)

    conditions: list[ColumnElement[bool]] = []
    if title_part:
        conditions.append(Book.title.ilike(f"%{title_part}%"))
    if min_year is not None:
        conditions.append(Book.year >= min_year)
    if max_pages is not None:
        conditions.append(Book.pages <= max_pages)
    if country:
        stmt = stmt.join(Book.author)
        conditions.append(Author.country == country)

    if conditions:
        stmt = stmt.where(and_(*conditions))
    return stmt


factory = build()
with Session(factory.kw["bind"]) as session:
    stmt = search_books(title_part="the", min_year=1900, country="GB")
    print(str(stmt.compile(compile_kwargs={"literal_binds": True})))
    print([b.title for b in session.scalars(stmt)])
```

```sql
SELECT book.id, book.title, book.year, book.pages, book.price, book.author_id
FROM book JOIN author ON author.id = book.author_id
WHERE book.title ILIKE '%the%' AND book.year >= 1900 AND author.country = 'GB'
ORDER BY book.title
```

`and_(*conditions)` z pustą listą zwróciłoby `True` (bezwarunkowo) — dlatego `if conditions` przed `.where()`. To drobiazg, ale eliminuje jedno `if` w środku łańcucha.

> 💡 **Analogia** — To jak zamawianie pizzy przez formularz: zaznaczasz trzy opcje z dziesięciu i klikasz „zamów”. Nie ma znaczenia, których **nie** zaznaczyłeś — system składa zamówienie z tego, co jest na liście.

### 11.3 Unikanie potworka z dwunastoma `if`-ami

Kod, który wygląda tak:

```python
# ✗ ŹLE — każde dodanie filtra wymaga nowej gałęzi
if title and year and country:
    stmt = select(Book).where(Book.title.like(f"%{title}%"), Book.year == year) \
        .join(Book.author).where(Author.country == country)
elif title and year:
    stmt = select(Book).where(Book.title.like(f"%{title}%"), Book.year == year)
elif title and country:
    ...
```

jest klasycznym objawem braku kompozycji. Wersja z sekcji 11.2 ma **liniową** złożoność, a nie wykładniczą liczbę gałęzi.

Trzy zasady, które warto zapamiętać:

1. **Warunki to dane.** Trzymaj je na liście, nie w gałęziach `if`.
2. **`Select` jest niemutowalny w praktyce** — każda metoda (`where`, `join`, `order_by`) zwraca **nowy** obiekt. Dzięki temu możesz bezpiecznie przekazywać `stmt` między funkcjami.
3. **Funkcja zwracająca `Select` nie wykonuje I/O.** Możesz ją testować bez bazy.

> ⚠️ **Pułapka** — Ponieważ `Select` jest w praktyce niemutowalny, ten kod **nie działa**:

```python
stmt = select(Book)
stmt.where(Book.year > 2000)      # ✗ zwraca NOWY obiekt, którego nikt nie używa
print(session.scalars(stmt).all())  # → zwraca WSZYSTKIE książki
```

Musisz przypisać: `stmt = stmt.where(...)`. To dokładnie ta sama pułapka co z `str.replace()` w Pythonie — i tak samo cicha.

> 🧪 **Ćwiczenie** — Napisz funkcję `paginate(stmt, page: int, size: int) -> Select` zwracającą nowe zapytanie z `limit` i `offset`. Sprawdź, czy działa, gdy `page=0` i gdy zapytanie ma już własne `limit`.

---

## 12. Warsztat: dziesięć zapytań do biblioteki

Poniższy plik to **obowiązkowy przykład modułu**: dziesięć zapytań o rosnącej złożoności, od prostego filtra do raportu z agregacją, w tym jedno ORM-owe `update()` i jedno `insert().returning()`.

```python
# sqla_library/q46_workshop.py
"""Dziesięć zapytań do schematu biblioteki — przegląd całego modułu."""

from datetime import date
from decimal import Decimal

from sqlalchemy import func, insert, or_, select, update
from sqlalchemy.orm import Session

from models import Author, Book, Category, Loan, Member, Review
from seed import build

factory = build()


def main() -> None:
    with Session(factory.kw["bind"], expire_on_commit=False) as session:
        print("=" * 70)
        print("1. Książki wydane po 1960 roku, od najnowszej")
        print("=" * 70)
        stmt = select(Book).where(Book.year > 1960).order_by(Book.year.desc())
        for book in session.scalars(stmt).all()[:5]:
            print(f"  {book.year}  {book.title}")

        print()
        print("=" * 70)
        print("2. Autorzy z Polski i ich książki (JOIN po relacji)")
        print("=" * 70)
        stmt = (
            select(Author.name, Book.title, Book.year)
            .join(Author.books)
            .where(Author.country == "PL")
            .order_by(Author.name, Book.year)
        )
        for name, title, year in session.execute(stmt):
            print(f"  {name:<22} {year}  {title}")

        print()
        print("=" * 70)
        print("3. Książki w kategorii 'Crime' — ile i jakie")
        print("=" * 70)
        stmt = (
            select(Book)
            .join(Book.categories)
            .where(Category.name == "Crime")
            .order_by(Book.title)
        )
        crime = session.scalars(stmt).all()
        print(f"  znaleziono: {len(crime)}")
        for book in crime:
            print(f"  {book.title}")

        print()
        print("=" * 70)
        print("4. Autorzy bez żadnej książki (LEFT JOIN + IS NULL)")
        print("=" * 70)
        stmt = select(Author).outerjoin(Author.books).where(Book.id.is_(None))
        authors = session.scalars(stmt).all()
        print(f"  znaleziono: {len(authors)}")

        print()
        print("=" * 70)
        print("5. Liczba książek i średnia liczba stron per autor")
        print("=" * 70)
        stmt = (
            select(
                Author.name,
                func.count(Book.id).label("books"),
                func.avg(Book.pages).label("avg_pages"),
            )
            .outerjoin(Author.books)
            .group_by(Author.id)
            .order_by(func.count(Book.id).desc(), Author.name)
        )
        for name, books, avg_pages in session.execute(stmt):
            avg = f"{float(avg_pages):.0f}" if avg_pages is not None else "—"
            print(f"  {books:>2} książek, średnio {avg:>4} stron  {name}")

        print()
        print("=" * 70)
        print("6. Członkowie z co najmniej 3 wypożyczeniami (EXISTS + count)")
        print("=" * 70)
        loans_count = (
            select(func.count(Loan.id))
            .where(Loan.member_id == Member.id)
            .scalar_subquery()
        )
        stmt = (
            select(Member.name, loans_count.label("loans"))
            .where(loans_count >= 3)
            .order_by(loans_count.desc())
        )
        for name, n in session.execute(stmt):
            print(f"  {n:>2}  {name}")

        print()
        print("=" * 70)
        print("7. Top 5 autorów po liczbie wypożyczeń ich książek")
        print("=" * 70)
        stmt = (
            select(Author.name, func.count(Loan.id).label("loans"))
            .join(Author.books)
            .join(Loan, Loan.book_id == Book.id)
            .group_by(Author.id)
            .order_by(func.count(Loan.id).desc(), Author.name)
            .limit(5)
        )
        for name, n in session.execute(stmt):
            print(f"  {n:>2}  {name}")

        print()
        print("=" * 70)
        print("8. Raport: kategoria → liczba książek + średnia ocena (HAVING)")
        print("=" * 70)
        stmt = (
            select(
                Category.name,
                func.count(func.distinct(Book.id)).label("books"),
                func.avg(Review.rating).label("avg_rating"),
            )
            .join(Category.books)
            .outerjoin(Book.reviews)
            .group_by(Category.id)
            .having(func.count(func.distinct(Book.id)) >= 2)
            .order_by(func.count(func.distinct(Book.id)).desc(), Category.name)
        )
        for name, books, avg_rating in session.execute(stmt):
            rating = f"{float(avg_rating):.2f}" if avg_rating is not None else "—"
            print(f"  {books:>2} książek, średnia ocena {rating:>5}  {name}")

        print()
        print("=" * 70)
        print("9. ORM-owy UPDATE: podnieś cenę książek wydanych przed 1950 o 10%")
        print("=" * 70)
        before = session.scalars(
            select(Book).where(Book.year < 1950).order_by(Book.title)
        ).all()
        print("  przed:")
        for book in before:
            print(f"    {book.price:>7}  {book.title}")

        stmt = (
            update(Book)
            .where(Book.year < 1950)
            .values(price=Book.price * Decimal("1.10"))
            .returning(Book)
        )
        updated = session.scalars(stmt, execution_options={"synchronize_session": "fetch"})
        print("  po:")
        for book in updated:
            print(f"    {book.price:>7}  {book.title}")
        session.commit()

        print()
        print("=" * 70)
        print("10. INSERT z RETURNING: dodaj książkę i zwróć pełny obiekt ORM")
        print("=" * 70)
        stmt = (
            insert(Book)
            .values(
                title="Powrót z gwiazd",
                year=1961,
                pages=180,
                price=Decimal("39.00"),
                author_id=2,
            )
            .returning(Book)
        )
        new_book = session.scalars(stmt).one()
        print(f"  dodano: id={new_book.id}  {new_book.title!r}  ({new_book.year})")

        # Sprawdź, że obiekt jest w identity map — kolejne zapytanie zwróci TEN SAM obiekt
        same = session.get(Book, new_book.id)
        print(f"  ten sam obiekt Pythona? {same is new_book}")
        session.commit()


if __name__ == "__main__":
    main()
```

Przykładowy wynik:

```text
======================================================================
1. Książki wydane po 1960 roku, od najnowszej
======================================================================
  2009  Drive Your Plow Over the Bones of the Dead
  2009  Here
  2007  Flights
  1974  The Dispossessed
  1969  The Left Hand of Darkness

======================================================================
2. Autorzy z Polski i ich książki (JOIN po relacji)
======================================================================
  Olga Tokarczuk          2007  Flights
  Olga Tokarczuk          2009  Drive Your Plow Over the Bones of the Dead
  Stanisław Lem           1961  Solaris
  ...

======================================================================
7. Top 5 autorów po liczbie wypożyczeń ich książek
======================================================================
   5  Agatha Christie
   4  Stanisław Lem
   3  Ursula K. Le Guin
   ...

======================================================================
9. ORM-owy UPDATE: podnieś cenę książek wydanych przed 1950 o 10%
======================================================================
  przed:
      34.90  Murder on the Orient Express
      36.90  And Then There Were None
      33.90  The Murder of Roger Ackroyd
      44.00  Ficciones
      43.00  The Aleph
  po:
      38.39  Murder on the Orient Express
      40.59  And Then There Were None
      37.29  The Murder of Roger Ackroyd
      48.40  Ficciones
      47.30  The Aleph

======================================================================
10. INSERT z RETURNING: dodaj książkę i zwróć pełny obiekt ORM
======================================================================
  dodano: id=16  'Powrót z gwiazd'  (1961)
  ten sam obiekt Pythona? True
```

Ostatnia linia jest ważna: `same is new_book` → `True`. Obiekt zwrócony przez `insert().returning(Book)` został dodany do identity map, więc `session.get()` zwrócił **dokładnie ten sam obiekt Pythona**, a nie nową kopię. To esencja ORM.

> 🧪 **Ćwiczenie** — Zapytanie 8 używa `func.count(func.distinct(Book.id))`, a nie `func.count(Book.id)`. Sprawdź, co się dzieje po zmianie na prostszy `count` — i wyjaśnij dlaczego. *(Podpowiedź: `outerjoin(Book.reviews)` mnoży wiersze.)*

---

## Podsumowanie

1. **Jeden dialekt zapytań.** W 2.0 `select()` obsługuje i Core, i ORM. To, czy dostaniesz encje czy `Row`, zależy wyłącznie od tego, co wstawisz do `select()`.
2. **`Session.query()` to legacy.** Nadal działa, ale nie ma typowania, nie współpracuje z async i nie dostaje nowych funkcji. Nie używaj go w nowym kodzie.
3. **`select()` to opis, nie wykonanie.** Możesz go trzymać w zmiennej, przekazywać, komponować. SQL poleci do bazy dopiero przy `session.execute()` / `session.scalars()`.
4. **`scalars()` vs `scalar()` vs `execute()`.** Pierwsze daje encje, drugie jedną wartość, trzecie surowe `Row`. Zła metoda = cichy błąd (gubienie kolumn), nie wyjątek.
5. **`Result` jest jednorazowy.** `.all()` i iteracja wyczerpują kursor. Nie wywołuj `.all()` dwa razy.
6. **`.unique()` jest wymagane przy `joinedload` kolekcji.** Bez niego — `InvalidRequestError`. `distinct()` (SQL) i `unique()` (ORM) to dwie różne warstwy.
7. **Warunek na prawej tabeli w `WHERE` kasuje `LEFT JOIN`.** Przenieś go do `onclause` przez `.outerjoin(T, warunek)`.
8. **`aliased()` do self-joinów**, `contains_eager()` do wykorzystania istniejącego JOIN-a jako źródła relacji.
9. **`exists()` i podzapytania skalarne** liczą w bazie — nie ładuj danych do Pythona, żeby je policzyć.
10. **ORM-owy DML to jedno polecenie, nie pętla.** `session.execute(update(...))` jest rzędy wielkości szybszy niż pętla po obiektach — ale **pomija** zdarzenia ORM, walidatory i `default` z Pythona.
11. **`synchronize_session` decyduje, czy obiekty w sesji są aktualne.** `'auto'` (domyślne) jest bezpieczne, `False` jest szybkie, ale zostawia kłamstwa w pamięci.
12. **`insert().returning(Book)` zwraca pełne obiekty ORM** w identity map. To najwygodniejszy sposób wstawiania, gdy potrzebujesz wyniku.

---

## Ćwiczenia

### Ćwiczenie 1 — Top 5 autorów po liczbie wypożyczeń (z obsługą braku wypożyczeń)

Napisz funkcję `top_authors(session: Session, limit: int = 5) -> list[tuple[str, int]]`, która zwraca listę `(nazwa_autora, liczba_wypożyczeń)` posortowaną malejąco po liczbie wypożyczeń, a przy remisie alfabetycznie po nazwisku. Autorzy bez ani jednego wypożyczenia mają mieć `0` i **też** mają się znaleźć na liście (o ile zmieszczą się w `limit`).

Wymagania dodatkowe:
- Użyj `outerjoin`, żeby nie zgubić autorów bez wypożyczeń.
- Użyj `GROUP BY` po kluczu głównym.
- Zwróć `int`, nie `Decimal` ani `None`.

### Ćwiczenie 2 — Paginacja z sortowaniem po dwóch kolumnach

Napisz funkcję `browse_books(session, page, size, sort_by, descending)` zwracającą krotkę `(items, total)`, gdzie:
- `items` to lista obiektów `Book`,
- `total` to łączna liczba książek pasujących do filtrów (do wyliczenia numeru stron w UI),
- `page` jest numerowane od 1,
- `sort_by` przyjmuje jedną z wartości: `"title"`, `"year"`, `"pages"`,
- sortowanie zawsze ma drugorzędny klucz `Book.id` (żeby paginacja była **deterministyczna**).

Sprawdź, że dla `page=1, size=5` i `page=2, size=5` nie ma powtórzeń ani luk w wynikach.

### Ćwiczenie 3 — Bezpieczny bulk update z raportem

Napisz skrypt, który:
1. Podnosi ceny wszystkich książek danego autora o zadany procent, używając **ORM-owego `update()`** (jedno polecenie SQL).
2. Zwraca `RETURNING` z nowymi cenami.
3. Wypisuje raport: liczba zmienionych wierszy, stara suma cen, nowa suma cen.
4. Działa w taki sposób, że obiekty `Book` załadowane **przed** aktualizacją mają aktualne ceny **po** aktualizacji — uzasadnij wybór `synchronize_session`.
5. Rzuca własny wyjątek `NoSuchAuthorError`, jeśli autor nie istnieje.

---

### Rozwiązania

#### Rozwiązanie 1

```python
# sqla_library/ex1_top_authors.py
from sqlalchemy import func, select
from sqlalchemy.orm import Session

from models import Author, Book, Loan
from seed import build


def top_authors(session: Session, limit: int = 5) -> list[tuple[str, int]]:
    """Zwraca (nazwa autora, liczba wypożyczeń) — malejąco, z remisami alfabetycznie.

    Kluczowe decyzje:
      * outerjoin(Author.books) — autorzy bez książek nie znikają.
      * outerjoin(Loan, ...)    — książki nigdy nie wypożyczone też nie znikają.
      * count(Loan.id)          — liczy tylko NIE-NULL-e, więc 0 dla braku wypożyczeń.
      * group_by(Author.id)     — poprawne w PostgreSQL (functional dependency).
    """
    stmt = (
        select(Author.name, func.count(Loan.id).label("loans"))
        .outerjoin(Author.books)
        .outerjoin(Loan, Loan.book_id == Book.id)
        .group_by(Author.id)
        .order_by(func.count(Loan.id).desc(), Author.name)
        .limit(limit)
    )
    return [(name, int(n)) for name, n in session.execute(stmt)]


factory = build()
with Session(factory.kw["bind"]) as session:
    for name, n in top_authors(session):
        print(f"{n:>2}  {name}")
```

```text
 5  Agatha Christie
 4  Stanisław Lem
 3  Ursula K. Le Guin
 2  Olga Tokarczuk
 1  Wisława Szymborska
```

**Dlaczego `count(Loan.id)`, a nie `count(*)`:** `COUNT(*)` liczy wiersze, a `outerjoin` z pustym dopasowaniem tworzy **jeden** wiersz z `NULL`-ami. `COUNT(Loan.id)` pomija `NULL`-e, więc autor bez wypożyczeń dostaje `0`, a nie `1`. To klasyczna pułapka `LEFT JOIN` + `COUNT`.

**Alternatywa:** `func.count(func.distinct(Loan.id))` — bezpieczniejsze, gdy w zapytaniu jest więcej niż jeden JOIN, który mógłby mnożyć wiersze. Kosztuje trochę wydajności (baza musi deduplikować).

**Minipułapka:** Gdybyś użył `group_by(Author.name)` zamiast `Author.id`, dwóch autorów o tym samym nazwisku zostałoby połączonych w jedną grupę. Mało prawdopodobne w tej domenie, ale nagminne przy tabelach `user` czy `product`.

---

#### Rozwiązanie 2

```python
# sqla_library/ex2_browse.py
from sqlalchemy import Select, func, select
from sqlalchemy.orm import Session

from models import Book
from seed import build

SORTABLE = {
    "title": Book.title,
    "year": Book.year,
    "pages": Book.pages,
}


def browse_books(
    session: Session,
    page: int = 1,
    size: int = 10,
    sort_by: str = "title",
    descending: bool = False,
) -> tuple[list[Book], int]:
    """Stronicowana lista książek + łączna liczba pasujących wierszy.

    `page` liczony od 1. Sortowanie zawsze ma drugorzędny klucz Book.id,
    dzięki czemu paginacja jest deterministyczna nawet przy remisach.
    """
    if sort_by not in SORTABLE:
        raise ValueError(f"sort_by musi być jednym z: {sorted(SORTABLE)}")
    if page < 1:
        raise ValueError("page liczony od 1")
    if not 1 <= size <= 100:
        raise ValueError("size musi być w zakresie 1..100")

    column = SORTABLE[sort_by]
    order = column.desc() if descending else column.asc()

    stmt: Select[tuple[Book]] = (
        select(Book).order_by(order, Book.id.asc()).limit(size).offset((page - 1) * size)
    )
    items = list(session.scalars(stmt))

    total = session.scalar(select(func.count()).select_from(Book)) or 0
    return items, total


factory = build()
with Session(factory.kw["bind"]) as session:
    for p in (1, 2, 3):
        items, total = browse_books(session, page=p, size=5, sort_by="year", descending=True)
        titles = [f"{b.year}:{b.id}" for b in items]
        print(f"strona {p} (z {-(-total // 5)}): {titles}")
```

```text
strona 1 (z 3): ['2009:15', '2009:11', '2007:14', '1974:2', '1969:1']
strona 2 (z 3): ['1968:6', '1968:3', '1965:5', '1961:4', '1957:10']
strona 3 (z 3): ['1949:13', '1944:12', '1939:8', '1934:7', '1926:9']
```

**Dlaczego drugorzędny klucz `Book.id` jest niezbędny:** w danych są **dwie** książki z rokiem 2009 (`Here` i `Drive Your Plow...`). Bez `ORDER BY year DESC, id ASC` PostgreSQL może zwrócić je w dowolnej kolejności przy każdym wykonaniu zapytania. Efekt: `page=1` i `page=2` mogą pokazać tę samą książkę albo żadna z nich. To klasyczny błąd paginacji, który ujawnia się dopiero na produkcji, przy dużym ruchu.

**Alternatywa — keyset pagination:** zamiast `OFFSET` użyj `WHERE (year, id) < (:last_year, :last_id)`. Zaleta: `OFFSET 100000` w PostgreSQL skanuje i odrzuca 100 000 wierszy; keyset tego nie robi. Wada: nie da się skoczyć na „stronę 47” — tylko „następna”. Szczegóły w module 20.

**Minipułapka:** `session.scalar(select(func.count()).select_from(Book))` wykonuje **drugie** zapytanie. Na dużych tabelach to kosztowne — rozważ `total` tylko przy pierwszej stronie albo szacowanie z `pg_class.reltuples`.

---

#### Rozwiązanie 3

```python
# sqla_library/ex3_bulk_update.py
from decimal import Decimal

from sqlalchemy import func, select, update
from sqlalchemy.orm import Session

from models import Author, Book
from seed import build


class NoSuchAuthorError(Exception):
    """Autor o podanym identyfikatorze nie istnieje."""


def raise_prices(
    session: Session,
    author_id: int,
    percent: Decimal,
) -> dict[str, object]:
    """Podnosi ceny książek autora o `percent` (np. Decimal('10') = +10%).

    Zwraca raport ze starymi i nowymi sumami cen.
    """
    if percent <= -100:
        raise ValueError("percent nie może wynosić -100% ani mniej")

    author = session.get(Author, author_id)
    if author is None:
        raise NoSuchAuthorError(f"Autor id={author_id} nie istnieje")

    factor = Decimal(1) + percent / Decimal(100)

    # Sumy PRZED zmianą — liczone w bazie, nie w Pythonie.
    old_sum = session.scalar(
        select(func.sum(Book.price)).where(Book.author_id == author_id)
    )

    stmt = (
        update(Book)
        .where(Book.author_id == author_id)
        .values(price=Book.price * factor)
        .returning(Book.id, Book.title, Book.price)
    )
    # 'fetch' zamiast 'auto': jesteśmy na SQLite, ale chcemy zachowanie identyczne
    # jak na PostgreSQL, i chcemy odświeżone obiekty w identity map.
    rows = session.execute(stmt, execution_options={"synchronize_session": "fetch"}).all()

    new_sum = session.scalar(
        select(func.sum(Book.price)).where(Book.author_id == author_id)
    )

    return {
        "author": author.name,
        "changed": len(rows),
        "old_sum": old_sum,
        "new_sum": new_sum,
        "rows": [(r.id, r.title, r.price) for r in rows],
    }


factory = build()
with Session(factory.kw["bind"], expire_on_commit=False) as session:
    # Załaduj obiekty PRZED aktualizacją — chcemy sprawdzić, czy się odświeżą.
    before = {b.id: b.price for b in session.scalars(select(Book).where(Book.author_id == 3))}

    report = raise_prices(session, author_id=3, percent=Decimal("10"))
    print(f"Autor: {report['author']}")
    print(f"Zmienionych wierszy: {report['changed']}")
    print(f"Suma przed: {report['old_sum']}")
    print(f"Suma po:    {report['new_sum']}")
    for book_id, title, price in report["rows"]:
        print(f"  {before[book_id]:>7} → {price:>7}  {title}")

    session.commit()
```

```text
Autor: Agatha Christie
Zmienionych wierszy: 3
Suma przed: 105.7
Suma po:    116.27
   34.90 →   38.39  Murder on the Orient Express
   36.90 →   40.59  And Then There Were None
   33.90 →   37.29  The Murder of Roger Ackroyd
```

**Dlaczego `synchronize_session="fetch"`:** obiekty `Book` zostały załadowane do sesji **przed** aktualizacją (w słowniku `before`). Bez synchronizacji ich atrybuty `price` pozostałyby stare — i każdy kolejny odczyt `book.price` w tej sesji zwracałby nieaktualną wartość. Strategia `"fetch"` każe SQLAlchemy odebrać `RETURNING` z `UPDATE` i odświeżyć pasujące obiekty z identity map. To działa identycznie na SQLite (z `RETURNING`) i na PostgreSQL.

**Alternatywa `synchronize_session=False`:** szybsza o jedną kolumnę w `RETURNING`, ale wtedy `book.price` w pamięci byłby kłamstwem. Uzasadnione tylko wtedy, gdy po aktualizacji **nie czytasz** tych obiektów (np. skrypt wsadowy, który zaraz zamyka sesję) albo gdy jawnie wywołasz `session.expire_all()`.

**Minipułapka:** `session.get(Author, author_id)` to jedno zapytanie, a `select(func.sum(...))` — dwa kolejne. Możesz je zredukować do jednego `select(Author, func.sum(...))` z `GROUP BY`, ale wtedy nie odróżnisz „autor nie istnieje” od „autor istnieje, ale nie ma książek”. Trzy zapytania są tu **świadomym** wyborem na rzecz jednoznacznych błędów.

---

## Najczęstsze błędy i jak je czytać

| Komunikat / objaw | Przyczyna | Naprawa |
|---|---|---|
| `InvalidRequestError: The unique() method must be invoked on this Result, as it contains results that include joined eager loads against collections` | `joinedload` na relacji kolekcji bez `.unique()` | Dodaj `.unique()` przed `.all()` / `.first()` |
| Wynik ma tylko pierwszą kolumnę, brakuje reszty | `session.scalars(stmt)` na zapytaniu wielokolumnowym — bierze pierwszą kolumnę i cicho gubi resztę | Użyj `session.execute(stmt)` (dostaniesz `Row`) albo zawęź `select()` do jednej kolumny |
| `TypeError: '>=' not supported between instances of 'InstrumentedAttribute' and 'Book'` | Porównanie atrybutu klasy z **instancją** obiektu zamiast z jej wartością | `Book.year >= probe.year`, nie `Book.year >= probe` |
| `AttributeError: 'Book' object has no attribute 'title'` po `session.execute(select(Book.title))` | `Row` nie ma atrybutów encji — `Row.title` zwraca kolumnę, ale `Row` nie jest `Book` | Rozpakuj: `title, = row` albo użyj `.scalars()` |
| Pusty wynik przy `NOT IN (podzapytanie)` | Podzapytanie zwraca `NULL` — `x NOT IN (..., NULL)` daje `UNKNOWN` dla każdego wiersza | Użyj `~select(...).exists()` albo dodaj `is_not(None)` w podzapytaniu |
| `sqlalchemy.exc.AmbiguousForeignKeysError: Can't determine join between 'review' and 'member'` | Więcej niż jeden klucz obcy między tabelami — SQLAlchemy nie wie, którego użyć | Podaj jawny warunek: `.join(Member, Review.member_id == Member.id)` |
| `sqlalchemy.exc.InvalidRequestError: Could not evaluate current criteria in Python` | `synchronize_session='evaluate'` na wyrażeniu, którego nie da się policzyć w Pythonie (np. `func.lower(...)`) | Zmień na `'fetch'` albo `False` |
| `sqlalchemy.exc.InvalidRequestError: This session is in 'prepared' state` | Wcześniejszy błąd w sesji; brak `rollback()` | Wywołaj `session.rollback()` w `except` (moduł 14) |
| `SELECT count(*)` zwraca `1` na pustej tabeli | Brak `select_from()` — SQL `SELECT count(*)` bez `FROM` liczy jeden wiersz wirtualny | `select(func.count()).select_from(Book)` |
| `ProgrammingError: column "book.title" must appear in the GROUP BY clause` | PostgreSQL odrzuca wybór kolumn nieobjętych `GROUP BY` (SQLite to toleruje) | Grupuj po kluczu głównym encji: `.group_by(Book.id)` |
| `sqlalchemy.exc.InvalidRequestError: Can't use yield_per with joined eager loads against collections` | `yield_per` + `joinedload` kolekcji — niekompatybilne | Zamień `joinedload` na `selectinload` albo usuń `yield_per` |
| Obiekt w sesji pokazuje starą wartość po `session.execute(update(...))` | `synchronize_session=False` (albo `'evaluate'` na wygaszonych obiektach) | Użyj `execution_options={"synchronize_session": "fetch"}` |
| Wyniki „skaczą” między stronami paginacji | Brak drugorzędnego klucza w `ORDER BY` przy duplikatach wartości sortującej | Dodaj `.order_by(col, Book.id)` |
| `stmt.where(...)` nie zmienia zapytania | `Select` jest niemutowalny — zwraca nowy obiekt, nie modyfikuje istniejącego | `stmt = stmt.where(...)` |
| `sqlalchemy.exc.ArgumentError: Column expression expected, got 'list'` | Przekazanie listy zamiast rozpakowania: `.where(conds)` zamiast `.where(*conds)` | Rozpakuj: `.where(*conditions)` lub `.where(and_(*conditions))` |
| Duplikaty encji mimo `distinct()` | `joinedload` kolekcji — duplikaty po stronie ORM, nie SQL | Dodaj `.unique()` (to inna warstwa niż `distinct()`) |

---

## Słowniczek modułu

| Termin (EN) | Polski | Wyjaśnienie |
|---|---|---|
| `select()` | konstruktor zapytania | Obiekt opisujący zapytanie SQL. Jeden dialekt dla Core i ORM. Nic nie wykonuje. |
| `Select[tuple[Book]]` | typ zapytania | Typ generyczny opisujący kształt wiersza wyniku. Dla jednej encji — krotka jednoelementowa. |
| `Session.execute()` | wykonanie przez sesję | Wykonuje dowolne wyrażenie SQL, zwraca `Result`. |
| `Session.scalars()` | skrót do skalarnych wyników | `execute(stmt).scalars()` — zwraca pierwszy element każdego wiersza. |
| `Session.scalar()` | skrót do jednej wartości | Pierwsza kolumna pierwszego wiersza lub `None`. |
| `Result` | wynik | Taca z wierszami. Jednorazowa — po `.all()` kursor jest wyczerpany. |
| `ScalarResult` | wynik skalarny | Widok na `Result` zwracający pojedyncze elementy zamiast `Row`. |
| `Row` | wiersz | Krotkopodobny obiekt z nazwami kolumn i indeksami. |
| `RowMapping` | wiersz słownikowy | Wynik `.mappings()` — obsługuje `dict(row)` i `**row`. |
| `.unique()` | deduplikacja ORM | Usuwa powtórzone **obiekty Pythona** (nie wiersze SQL). Wymagane przy `joinedload` kolekcji. |
| `NoResultFound` | brak wyniku | Wyjątek `.one()`, gdy zero wierszy. |
| `MultipleResultsFound` | wiele wyników | Wyjątek `.one()` / `.one_or_none()`, gdy więcej niż jeden wiersz. |
| instrumented attribute | atrybut instrumentowany | Deskryptor, który na instancji zwraca wartość, a na klasie — wyrażenie SQL. |
| `onclause` | warunek złączenia | Warunek w `JOIN ... ON`. Różnica między `ON` a `WHERE` decyduje o semantyce `LEFT JOIN`. |
| `outerjoin()` | złączenie zewnętrzne lewe | `LEFT OUTER JOIN` — zachowuje wiersze bez dopasowania. |
| anti-join | złączenie anty | Wzorzec „A bez B”: `outerjoin` + `IS NULL` lub `NOT EXISTS`. |
| `aliased()` | alias encji | Kopia encji pod inną nazwą — do self-joinów i wielokrotnych JOIN-ów. |
| `contains_eager()` | użycie istniejącego JOIN-a | Każe ORM-owi wypełnić relację danymi z JOIN-a, który już jest w zapytaniu. |
| `distinct()` | usunięcie duplikatów | `SELECT DISTINCT` — deduplikacja **wierszy SQL**. Inna warstwa niż `unique()`. |
| `func` | fabryka funkcji SQL | `func.count()`, `func.lower()`, `func.extract()` — generuje wywołania funkcji w SQL. |
| `select_from()` | źródło zapytania | Jawnie wskazuje tabelę w `FROM`, gdy nie wynika ona z kolumn. |
| `group_by()` / `having()` | grupowanie / filtr grup | `HAVING` filtruje grupy; `WHERE` filtruje wiersze przed grupowaniem. |
| `exists()` | istnieje | `EXISTS (podzapytanie)` — najszybszy test „czy cokolwiek pasuje”. |
| `scalar_subquery()` | podzapytanie skalarne | Podzapytanie zwracające jedną wartość — użyteczne w `SELECT`, `WHERE`, `ORDER BY`. |
| `union_all()` | suma zbiorów bez deduplikacji | Łączy wyniki dwóch zapytań o tym samym kształcie kolumn. |
| `yield_per` | porcjowanie strumienia | Opcja wykonania: czyta wyniki w paczkach po N, ograniczając pamięć. |
| `populate_existing` | wymuszone odświeżenie | Opcja wykonania: nadpisz atrybuty obiektu z identity map danymi z bazy. |
| `with_for_update()` | blokada pesymistyczna | `SELECT ... FOR UPDATE` — blokuje wiersze na czas transakcji. |
| ORM-enabled DML | DML przez ORM | `insert()`/`update()`/`delete()` wykonane przez `Session` — jedno polecenie, ale bez cyklu życia obiektów. |
| `synchronize_session` | synchronizacja sesji | Strategia uzgadniania stanu obiektów w sesji ze zmianami wykonanymi przez DML: `'auto'`, `'fetch'`, `'evaluate'`, `False`. |
| `returning()` | zwracanie wierszy | Klauzula `RETURNING` — zwraca wstawione/zaktualizowane wiersze, także jako obiekty ORM. |
| bulk insert | wstawianie masowe | `session.execute(insert(Model), [dict, ...])` — szybkie, bez tworzenia obiektów ORM. |
| `insertmanyvalues` | wstawianie wielowartościowe | Mechanizm 2.0 łączący wiele `INSERT`-ów w jedno polecenie `VALUES (...), (...), ...`. |
| identity map | mapa tożsamości | Rejestr obiektów w sesji kluczowany po `(klasa, PK)`. Gwarantuje jeden obiekt Pythona na wiersz. |
| autoflush | automatyczne wypchnięcie | Przed każdym zapytaniem ORM sesja wysyła oczekujące zmiany, żeby zapytanie widziało aktualny stan. |

---

## Dalsze czytanie

- **ORM Querying Guide (główny dokument tego modułu):**
  https://docs.sqlalchemy.org/en/20/orm/queryguide/index.html
- **SELECT w ORM — pełna lista opcji i wzorców:**
  https://docs.sqlalchemy.org/en/20/orm/queryguide/select.html
- **ORM-Enabled INSERT, UPDATE, DELETE (sekcja 10 tego modułu):**
  https://docs.sqlalchemy.org/en/20/orm/queryguide/dml.html
- **Ładowanie relacji i strategie (`selectinload`, `joinedload`, `contains_eager`):**
  https://docs.sqlalchemy.org/en/20/orm/queryguide/relationships.html
- **Dodatkowe funkcje zapytań ORM (`yield_per`, `populate_existing`, `with_for_update`, `with_loader_criteria`):**
  https://docs.sqlalchemy.org/en/20/orm/queryguide/api.html
- **Legacy Query API — pełna tabela migracyjna:**
  https://docs.sqlalchemy.org/en/20/orm/queryguide/query.html
- **`Session` API — `execute`, `scalars`, `scalar`, `get`:**
  https://docs.sqlalchemy.org/en/20/orm/session_api.html
- **Core — konstruktory `select`, `insert`, `update`, `delete`, funkcje i operatory:**
  https://docs.sqlalchemy.org/en/20/core/sqlelement.html
- **Co nowego w 2.0 — kontekst zmian w API:**
  https://docs.sqlalchemy.org/en/20/changelog/whatsnew_20.html
- **Wersja 2.1 tych samych dokumentów (dla ramek 🆕):**
  https://docs.sqlalchemy.org/en/21/orm/queryguide/dml.html

---

## Co dalej

Wiesz już, jak *napisać* każde zapytanie, którego potrzebujesz — ale nie wiesz jeszcze, **ile zapytań** faktycznie poleci do bazy. W module 11 zmierzysz się z najczęstszą przyczyną „ORM jest wolny”: problemem N+1. Nauczysz się go wykrywać licznikiem zapytań i eliminować przez `selectinload`, `joinedload` i `raiseload` — z liczbami, nie z opiniami.

➡️ **[Moduł 11 — Ładowanie relacji i problem N+1](11_ladowanie_i_n_plus_1.md)**

<!-- koniec modułu 10 -->