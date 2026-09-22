# Moduł 20 — Wzorzec Repository

W tym module nauczysz się porządkować dostęp do bazy danych w jednym, przewidywalnym miejscu. Zobaczysz, jak zamienić dziesiątki rozproszonych po całym projekcie zapytań w czytelne metody o nazwach z języka biznesu („daj książkę po ISBN”, „policz wypożyczenia”), jak nie przeciekać SQL-a do warstwy API i jak testować kod dostępu do danych bez uruchamiania prawdziwej bazy. Poznasz też szczerą krytykę tego wzorca — bo repozytorium, źle zastosowane, potrafi zaszkodzić bardziej niż pomóc.

---

**Poziom:** 🔴 architektoniczny
**Czas:** ~180 minut
**Wymagania wstępne:**
- `07_modele_deklaratywne.md` — modele `Mapped` / `mapped_column`
- `08_sesja_cykl_zycia.md` — czym jest `Session`, `flush`, `commit`, identity map
- `09_relacje.md` — `relationship`, kardynalności, tabela asocjacyjna
- `10_zapytania_orm.md` — `select()`, `session.scalars()`, ORM-owe DML
- `11_ladowanie_i_n_plus_1.md` — `selectinload`, `joinedload`
- `17_wydajnosc.md` — projekcje, indeksy, dlaczego liczba zapytań ma znaczenie
- `18_testowanie.md` — `pytest`, fixtures, SQLite w pamięci
- `19_warstwy_i_data_mapper.md` — podział na warstwy i rola DTO
- `21_unit_of_work.md` — technicznie **następny** moduł, ale w tym miejscu będziemy się do niego wielokrotnie odwoływać; nie musisz go jeszcze znać, wystarczy wiedza, że transakcją zarządza ktoś „ponad” repozytorium

**Czego dotyczy plik:** wzorzec Repository — sposób organizacji kodu dostępu do danych w aplikacji korzystającej z SQLAlchemy 2.0+.

---

## Spis treści

- [1. Po co repozytorium](#1-po-co-repozytorium)
- [2. Repozytorium w wersji podstawowej](#2-repozytorium-w-wersji-podstawowej)
- [3. Repozytorium generyczne — i kiedy się nie opłaca](#3-repozytorium-generyczne--i-kiedy-się-nie-opłaca)
- [4. Abstrakcja przez protokół](#4-abstrakcja-przez-protokół)
- [5. Zapytania specyficzne i budowniczowie zapytań](#5-zapytania-specyficzne-i-budowniczowie-zapytań)
- [6. Paginacja: offset/limit kontra keyset](#6-paginacja-offsetlimit-kontra-keyset)
- [7. Specification — filtry jako obiekty](#7-specification--filtry-jako-obiekty)
- [8. Projekcje i DTO](#8-projekcje-i-dto)
- [9. Wyjątki i tłumaczenie błędów](#9-wyjątki-i-tłumaczenie-błędów)
- [10. Krytyka wzorca repozytorium w projektach SQLAlchemy](#10-krytyka-wzorca-repozytorium-w-projektach-sqlalchemy)
- [11. Testowanie repozytoriów](#11-testowanie-repozytoriów)
- [12. Pełny przykład do uruchomienia](#12-pełny-przykład-do-uruchomienia)
- [Podsumowanie](#podsumowanie)
- [Ćwiczenia](#ćwiczenia)
- [Najczęstsze błędy i jak je czytać](#najczęstsze-błędy-i-jak-je-czytać)
- [Słowniczek modułu](#słowniczek-modułu)
- [Dalsze czytanie](#dalsze-czytanie)
- [Co dalej](#co-dalej)

---

## 1. Po co repozytorium

### 1.1. Problem, który widzisz dopiero po trzecim miesiącu pracy

Wyobraź sobie, że piszesz aplikację do obsługi biblioteki. Na początku wszystko wygląda niewinnie:

```python
# examples/20_problem_before.py
from sqlalchemy import select
from sqlalchemy.orm import Session

from .models import Book


def show_book(session: Session, isbn: str) -> None:
    stmt = select(Book).where(Book.isbn == isbn)
    book = session.scalars(stmt).one_or_none()
    print(book.title if book else "brak")
```

Potem przychodzi kolejny ekran, w którym trzeba tę samą książkę pobrać. Kopiujesz cztery linijki. Potem endpoint API — kopiujesz. Potem zadanie w kolejce — kopiujesz. Po kilku tygodniach okazuje się, że w projekcie jest **siedemnaście** miejsc, w których leci `select(Book).where(Book.isbn == ...)`, i każde z nich jest odrobinę inne:

- jedno używa `.one_or_none()`,
- drugie `.first()`,
- trzecie zapomniało o odfiltrowaniu książek usuniętych (`deleted_at IS NULL`),
- czwarte dokłada `selectinload(Book.author)`, a piąte nie i kończy się błędem `DetachedInstanceError`,
- szóste zapisuje się na literówkę w nazwie kolumny i nikt tego nie zauważył, bo kod nie ma testów.

To jest **pierwszy objaw choroby**: logika dostępu do danych rozsypana po całym projekcie. Każda zmiana reguły („od teraz nie pokazujemy książek usuniętych miękkou”) wymaga dotknięcia siedemnastu miejsc. Zawsze któreś zostanie pominięte.

Drugi objaw jest subtelniejszy: **SQL wycieka do miejsc, w których nie powinien się pojawiać**. Jeśli piszesz w FastAPI endpoint, który wygląda tak:

```python
# examples/20_leak.py
@app.get("/books/{book_id}")
async def get_book(book_id: int, session: AsyncSession = Depends(get_session)):
    stmt = (
        select(Book)
        .where(Book.id == book_id)
        .options(selectinload(Book.author), selectinload(Book.categories))
    )
    book = (await session.scalars(stmt)).one_or_none()
    ...
```

to warstwa HTTP (prezentacji) zna `select`, `where`, `options`, `selectinload` i wie, że relacje są ładowane leniwie. To wiedza o bazie danych wklejona do kodu, który powinien zajmować się wyłącznie tłumaczeniem żądania na odpowiedź. Konsekwencje:

1. Nie da się zmienić bazy (np. przenieść część danych do cache'u) bez przepisywania endpointów.
2. Nie da się przetestować endpointu bez bazy.
3. Każdy programista pisze zapytania trochę inaczej — nie ma jednej wykładni.

> 💡 **Analogia — bibliotekarz zamiast samodzielnych poszukiwań.**
> Wyobraź sobie bibliotekę, w której czytelnik musi sam iść między regały, znać układ sygnatur, wiedzieć, że nowe książki stoją w innym skrzydle, a czasopisma są w piwnicy. Każdy czytelnik robi to inaczej i część się gubi.
> Repozytorium to **bibliotekarz przy ladzie**. Czytelnik mówi: „poproszę ostatnie wypożyczenia tego pana”. Bibliotekarz zna układ regałów, wie, że część zbioru jest w magazynie, i sam wybiera optymalną drogę. Czytelnik nie musi wiedzieć nic o architekturze budynku.
> Zapamiętaj tę analogię, bo wrócimy do niej przy krytyce wzorca — okaże się, że czasem „bibliotekarz” to zbędny pośrednik, gdy czytelnik i tak zaraz idzie do magazynu sam.

> 🧠 **Dlaczego tak jest.**
> SQLAlchemy daje Ci `Session` — bardzo potężne narzędzie, które umie *wszystko*: budować zapytania, śledzić zmiany obiektów, zarządzać transakcją, trzymać identity map. To jednocześnie zaleta i wada: „umie wszystko” oznacza, że każde miejsce w kodzie może użyć go dowolnie. Wzorzec repository to umowa społeczna: **zespół zgadza się, że tylko jedna warstwa rozmawia z `Session`**, a reszta kodu używa jej przez wąski, nazwany interfejs.

### 1.2. Definicja wzorca

Wzorzec **Repository** (pol. *repozytorium*) w najkrótszej wersji:

> Repozytorium to obiekt, który zachowuje się jak **kolekcja obiektów domenowych w pamięci**, a pod spodem tłumaczy operacje na tej kolekcji na operacje na trwałym magazynie danych (bazie, pliku, zdalnym API).

Kluczowe słowa w tej definicji:

- **„jak kolekcja w pamięci”** — repozytorium ma metody, które wyglądają, jakbyś operował na zwykłej liście lub słowniku Pythona: `add`, `remove`, `get`, `list`. Nie ma tam `join`, `where`, `limit`.
- **„tłumaczy”** — to repozytorium, nie jego klient, wie, że dane leżą w tabeli `book`, że trzeba zrobić `SELECT`, że połączenie obsługuje SQLAlchemy.
- **„trwałym magazynie danych”** — repozytorium nie jest przywiązane do konkretnej technologii. Możesz mieć implementację na SQLAlchemy i drugą, trzymającą dane w słowniku Pythona (tak, to legalne i bardzo przydatne — zobacz sekcję 11).

Wzorzec pochodzi z katalogu wzorców Martina Fowlera *Patterns of Enterprise Application Architecture* (PoEAA). W oryginale towarzyszy mu para: **Repository** + **Unit of Work** (moduł 21). Razem tworzą spójny system: repozytorium mówi *jak* czytać i zapisywać pojedyncze obiekty, jednostka pracy mówi *kiedy* te zmiany trafiają do bazy i w jakiej transakcji.

### 1.3. Co repozytorium daje, a czego nie daje

Zanim brniesz dalej, uczciwy bilans. Repozytorium **daje**:

| Korzyść | Konkretnie |
|---|---|
| Jedno miejsce na reguły dostępu | „Usunięte pomijamy” zapisane raz, w jednej metodzie |
| Nazwy z języka biznesu | `repo.list_overdue_loans()` zamiast trzyakapitowego `select()` |
| Testowalność bez bazy | Implementacja in-memory podstawiona w teście serwisu |
| Wymienialność magazynu | Baza → cache → zewnętrzne API bez zmiany serwisów |
| Słabsze sprzężenie z SQLAlchemy | Domena nie importuje `sqlalchemy` |

Repozytorium **nie daje** (choć czasem się to sprzedaje jako korzyść):

| Obietnica | Rzeczywistość |
|---|---|
| „Ukryje przed Tobą bazę danych” | Nie ukryje. Prędzej czy później trafisz na N+1, transakcję albo indeks. Repozytorium tylko odsuwa te decyzje |
| „Pozwoli zmienić bazę bez zmian w kodzie” | Rzadko się zdarza w praktyce. Wymiana PostgreSQL na MySQL to i tak seria poprawek w migracjach i zapytaniach |
| „Uprości kod” | Czasem tak, czasem dodaje warstwę. Zobacz sekcję 10 |
| „Zastąpi SQL” | Nie. Musisz rozumieć SQL, żeby wiedzieć, co piszesz |

> ⚠️ **Pułapka — repozytorium jako religia.**
> Najczęstszy błąd to traktowanie repozytoriów jako obowiązkowego rytuału: „skoro to dobry wzorzec, robimy repozytorium dla każdej tabeli, nawet dla dwóch metod”. Efekt: setki linijek kodu przenoszących `session.get()` do metody `get()`. To nie architektura, to biurokracja. Repozytorium dodaje wartość, gdy **jest co chronić**: gdy reguły dostępu są nietrywialne, gdy zapytań jest wiele, gdy chcemy testować logikę bez bazy. Wrócimy do tego w sekcji 10 z konkretnymi kryteriami.

---

## 2. Repozytorium w wersji podstawowej

Zbudujmy wersję minimalną, ale kompletną. Będziemy pracować na schemacie biblioteki znanym z modułu 09. Przypomnijmy go, żeby dalej wszystko było samowystarczalne.

```python
# examples/20_models.py
from __future__ import annotations

from sqlalchemy import Column, ForeignKey, String, Table
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    """Bazowa klasa deklaratywna dla wszystkich modeli."""


# Tabela asocjacyjna dla relacji wiele-do-wielu (Book <-> Category)
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

    books: Mapped[list[Book]] = relationship(back_populates="author")


class Category(Base):
    __tablename__ = "category"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(80), unique=True)


class Book(Base):
    __tablename__ = "book"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    isbn: Mapped[str] = mapped_column(String(20), unique=True, index=True)
    published_year: Mapped[int | None] = mapped_column(default=None)
    author_id: Mapped[int] = mapped_column(ForeignKey("author.id"))

    author: Mapped[Author] = relationship(back_populates="books")
    categories: Mapped[list[Category]] = relationship(secondary=book_category)
```

### 2.1. Klasa `BookRepository` — pierwsze podejście

Zanim pokażemy wersję „ładną”, spójrzmy na to, co ma robić repozytorium. Metody wynikają wprost z pytań, które zadaje aplikacja:

- „daj książkę o identyfikatorze 42”,
- „daj książkę o ISBN 978-... ”,
- „wypisz książki na stronie 3”,
- „dodaj nową książkę”,
- „usuń tę książkę”,
- „czy mamy już książkę o tym ISBN?”.

```python
# examples/20_repository_basic.py
from collections.abc import Sequence

from sqlalchemy import Select, func, select
from sqlalchemy.orm import Session

from .models import Book


class BookRepository:
    """Repozytorium książek oparte o SQLAlchemy.

    Trzyma referencję do sesji przekazanej z zewnątrz — sama jej NIE tworzy.
    """

    def __init__(self, session: Session) -> None:
        self._session = session

    def get(self, book_id: int) -> Book | None:
        """Zwraca książkę o podanym id albo None, gdy nie istnieje."""
        return self._session.get(Book, book_id)

    def get_by_isbn(self, isbn: str) -> Book | None:
        """Zwraca książkę o podanym ISBN albo None."""
        stmt = select(Book).where(Book.isbn == isbn)
        return self._session.scalars(stmt).one_or_none()

    def list(self, *, limit: int = 20, offset: int = 0) -> Sequence[Book]:
        """Zwraca porcję katalogu, posortowaną po id."""
        stmt = select(Book).order_by(Book.id).limit(limit).offset(offset)
        return self._session.scalars(stmt).all()

    def add(self, book: Book) -> Book:
        """Dodaje książkę do sesji. NIE commituje."""
        self._session.add(book)
        return book

    def remove(self, book: Book) -> None:
        """Oznacza książkę do usunięcia. NIE commituje."""
        self._session.delete(book)

    def exists(self, isbn: str) -> bool:
        """Czy istnieje książka o podanym ISBN?"""
        stmt = select(select(Book.id).where(Book.isbn == isbn).exists())
        return bool(self._session.scalar(stmt))
```

Przyjrzyjmy się po kolei każdej metodzie.

**`get`** używa `session.get()`, a nie `select()`. To celowe: `session.get()` najpierw zagląda do identity map (moduł 08) i jeśli obiekt o tym kluczu już jest w pamięci, **nie wysyła żadnego SQL-a**. Na małych i średnich zbiorach to najtańszy możliwy odczyt. Używaj `get()` zawsze, gdy szukasz po kluczu głównym.

> 🔬 **Pod maską — `session.get(Book, 42)` przy zimnej sesji**
> ```sql
> SELECT book.id, book.title, book.isbn, book.published_year, book.author_id
> FROM book
> WHERE book.id = ?
> ```
> Przy drugim wywołaniu w tej samej sesji: brak zapytania, obiekt wraca z identity map.

**`get_by_isbn`** nie może użyć `get()`, bo ISBN nie jest kluczem głównym. Dlatego budujemy `select()`. Zwróć uwagę na `.one_or_none()` zamiast `.first()`:

- `.first()` powie „weź pierwszą i nie interesuj się resztą”. Jeśli w bazie są dwa rekordy o tym samym ISBN (a nie powinno ich być — mamy `unique=True`), `.first()` cicho zwróci jeden z nich i problem zostanie niezauważony.
- `.one_or_none()` rzuci `MultipleResultsFound`, jeśli rekordów jest więcej niż jeden. To asercja niezmiennika: **„ISBN jest unikalny i chcę, żeby baza mnie pilnowała”**.

To wzorzec wart zapamiętania na całe życie: **`one_or_none()` dokumentuje założenie o danych; `first()` je ukrywa**.

**`list`** — prosta paginacja `limit`/`offset`. `order_by(Book.id)` jest obowiązkowe: bez niego bazy danych nie gwarantują żadnej kolejności, a „strona 2” może zwrócić inne wiersze niż poprzednio. Wrócimy do tego szczegółowo w sekcji 6.

**`add`** i **`remove`** — tu jest sedno sprawy. Repozytorium **nie commituje**.

> 🧠 **Dlaczego repozytorium NIE robi `commit()` — to kluczowa decyzja projektowa.**
> Wyobraź sobie przypadek użycia „wypożycz książkę”:
> 1. znajdź książkę (`BookRepository.get`),
> 2. znajdź czytelnika (`MemberRepository.get`),
> 3. utwórz rekord wypożyczenia (`LoanRepository.add`),
> 4. zmniejsz liczbę dostępnych egzemplarzy (`BookRepository.update_availability`).
>
> To **jedna** operacja biznesowa. Musi być atomowa: albo wszystkie cztery kroki się udadzą, albo żaden. Gdyby każde repozytorium commitowało osobno, to po kroku 3 mielibyśmy wypożyczenie w bazie, a po awarii w kroku 4 — książkę, której nikt nie wypożyczył, ale której nie ma na półce.
> Dlatego **granica transakcji leży wyżej**, w warstwie aplikacji (Unit of Work, moduł 21) albo w serwisie. Repozytorium tylko *zgłasza intencję* (`add`), a nie *wykonuje* jej (`commit`).
> Analogia: repozytorium to kasjer wpisujący pozycje na paragon. Paragon podpisuje i zamyka kierownik zmiany (Unit of Work), dopiero gdy wszystkie pozycje są gotowe.

Spróbujmy zobrazować to diagramem:

```text
┌─────────────────────────────────────────────────────────────┐
│  WARSTWA APLIKACJI (serwis / UseCase / endpoint)            │
│  → otwiera transakcję                                       │
│  → wywołuje repozytoria                                     │
│  → COMMIT albo ROLLBACK                                     │
└───────────────┬─────────────────────────────┬───────────────┘
                │                             │
      ┌─────────▼─────────┐         ┌─────────▼─────────┐
      │ BookRepository    │         │ MemberRepository  │
      │  (Session)        │         │  (Session)        │
      │  TYLKO add/remove │         │  TYLKO add/remove │
      │  NIE commituje    │         │  NIE commituje    │
      └─────────┬─────────┘         └─────────┬─────────┘
                │                             │
                └──────────────┬──────────────┘
                               ▼
                    TA SAMA transakcja bazodanowa
```

Kluczowe: **obie instancje repozytoriów muszą dostać tę samą sesję**. Jeśli `BookRepository` utworzy swoją sesję, a `MemberRepository` drugą, to będą to dwie niezależne transakcje i atomowość znika. To najważniejsza reguła tego modułu:

> **Repozytorium nigdy nie tworzy sesji samo. Sesję zawsze wstrzykuje ktoś z zewnątrz.**

**`exists`** pokazuje, jak zapisać pytanie o istnienie bez pobierania danych. `select(...).exists()` buduje podzapytanie `EXISTS`, a my opakowujemy je w `select()` i pytamy bazę o jedną wartość logiczną.

> 🔬 **Pod maską — `exists("978-83-000-0000-1")`**
> ```sql
> SELECT EXISTS (
>     SELECT book.id
>     FROM book
>     WHERE book.isbn = ?
> ) AS anon_1
> ```
> Baza nie pobiera ani jednego wiersza książki — tylko odpowiada „tak/nie”. Na tabeli z indeksem na `isbn` to operacja $O(\log N)$.

### 2.2. Repozytorium a `Session` — co wolno, a czego nie

Zbierzmy w tabeli, co należy do repozytorium, a co do warstwy wyżej.

| Operacja | Repozytorium | Serwis / UoW |
|---|---|---|
| `select()` — budowa zapytania | ✅ | ❌ |
| `session.get()` | ✅ | ❌ |
| `session.add()` / `delete()` | ✅ | ❌ |
| `session.flush()` | ⚠️ rzadko (patrz sekcja 9) | ✅ |
| `session.commit()` | ❌ **nigdy** | ✅ |
| `session.rollback()` | ❌ | ✅ |
| `session.begin()` | ❌ | ✅ |
| Logika biznesowa („czy można wypożyczyć”) | ❌ | ✅ |

Ta tabela jest sednem dyscypliny. Jeśli przyłapiesz się na `commit()` w repozytorium — to znak, że repozytorium przerodziło się w coś innego i straciło swoją główną zaletę: możliwość złożenia wielu operacji w jedną transakcję.

> 🧪 **Ćwiczenie — ręczne sprawdzenie atomowości.**
> Napisz skrypt, który w jednej sesji doda dwie książki jedną metodą repozytorium, a potem symuluje błąd (np. celowo wywołuje `raise RuntimeError`). Obserwuj `echo=True`: mimo że repozytorium zgłosiło dwie intencje, do bazy nie poszedł `INSERT`, bo nikt nie commituje. To namacalny dowód, że podział odpowiedzialności działa.

---

## 3. Repozytorium generyczne — i kiedy się nie opłaca

Widząc, że `BookRepository`, `AuthorRepository` i `LoanRepository` mają identyczne metody `get`, `add`, `remove`, naturalną reakcją jest chęć **usunięcia duplikacji**. Stąd pomysł na repozytorium generyczne.

```python
# examples/20_repository_generic.py
from collections.abc import Sequence
from typing import Generic, TypeVar

from sqlalchemy import select
from sqlalchemy.orm import Session

from .models import Base


ModelT = TypeVar("ModelT", bound=Base)


class GenericRepository(Generic[ModelT]):
    """Bazowe repozytorium z metodami wspólnymi dla wszystkich encji.

    Podklasy deklarują atrybut `model` — konkretną klasę ORM.
    """

    model: type[ModelT]

    def __init__(self, session: Session) -> None:
        self._session = session

    def get(self, pk: int) -> ModelT | None:
        return self._session.get(self.model, pk)

    def list(self, *, limit: int = 20, offset: int = 0) -> Sequence[ModelT]:
        stmt = select(self.model).order_by(self.model.id).limit(limit).offset(offset)
        return self._session.scalars(stmt).all()

    def add(self, entity: ModelT) -> ModelT:
        self._session.add(entity)
        return entity

    def remove(self, entity: ModelT) -> None:
        self._session.delete(entity)

    def count(self) -> int:
        from sqlalchemy import func

        stmt = select(func.count()).select_from(self.model)
        return int(self._session.scalar(stmt) or 0)


class BookRepository(GenericRepository[Book]):
    model = Book

    def get_by_isbn(self, isbn: str) -> Book | None:
        stmt = select(Book).where(Book.isbn == isbn)
        return self._session.scalars(stmt).one_or_none()
```

Zalety są realne:

- mniej kodu do przepisania przy dodaniu nowej encji,
- spójny kształt API we wszystkich repozytoriach (łatwiej się przeskakuje między plikami),
- jednolite miejsce na zmiany wspólne (np. dodanie `exists` do wszystkich naraz).

Ale to nie jest darmowe. Zobaczmy, gdzie generyczne repozytorium realnie **zawodzi**.

### 3.1. Gdzie generyczne repozytorium się nie sprawdza

**a) `model.id` jako uniwersalne założenie.** W `list` użyliśmy `self.model.id`. To zakłada, że **każda** encja ma kolumnę `id`. A gdy któraś ma klucz złożony (`loan_id`, `book_id` — jak w tabeli asocjacyjnej z własnymi kolumnami)? Generyczne repozytorium pęknie. Można to obsłużyć atrybutem `order_by_col`, ale z każdym takim wyjątkiem generyczność się rozmywa.

**b) Różne sposoby sortowania i paginacji.** Dla `Book` naturalne jest sortowanie po `title`, dla `Loan` po `due_date DESC`. Generyczne `list` tego nie wie.

**c) Zapytania specyficzne i tak lądują w podklasie.** `get_by_isbn` nie jest generyczne — to metoda `BookRepository`. Jeśli każde z pięciu repozytoriów ma trzy własne metody, „wspólna” część to i tak 30% kodu. Zysk maleje.

**d) Projekcje łamią typ generyczny.** Gdy repozytorium zwraca DTO (`BookSummary`) zamiast encji `Book` (sekcja 8), typ `ModelT` przestaje pasować. Generyczne repozytorium z definicji zakłada, że zwracamy jedną, konkretną klasę.

**e) Abstrakcja „przecieka” przez typy.** `GenericRepository[Book]` wciąż dziedziczy z SQLAlchemy-zależnej klasy. Rozwiązanie z sekcji 4 (protokoły) jest czystsze, ale trudniejsze do pogodzenia z generycznością.

> 💡 **Analogia — uniwersalny pilot do wszystkiego.**
> Generyczne repozytorium przypomina uniwersalny pilot, który obsługuje 500 modeli telewizorów. Na początku jest świetny: jedna instrukcja, wszystkie przyciski na miejscu. Ale gdy chcesz zmienić ustawienia obrazu w konkretnym telewizorze albo użyć funkcji, którą ma tylko Twój model, uniwersalny pilot okazuje się gorszy niż oryginalny. Nie da się zbudować pilota, który zna wszystkie specyficzne funkcje wszystkich telewizorów, i jednocześnie ma prosty przycisk „włącz”.

### 3.2. Rekomendacja praktyczna

Nie ma jednej odpowiedzi. Zależy od tego, jak **niejednorodne** są Twoje encje.

| Sytuacja | Rekomendacja |
|---|---|
| 3–5 encji, proste CRUD, brak złożonej domeny | **Nie rób repozytoriów wcale**, albo zrób proste klasy bez generyczności |
| 10–20 encji, dużo wspólnego CRUD, niewiele zapytań specyficznych | Generyczne repozytorium **z klasą bazową** ma sens |
| Domena z bogatymi regułami, zapytania bardzo zróżnicowane | Repozytoria per encja, bez generyczności — specyficzne API jest wartością, nie duplikacją |
| Zespół 1–2 osoby, projekt prosty | Jedna klasa-zestaw funkcji modułu (patrz sekcja 10) wystarczy |

**Reguła kciuka:** generyczność opłaca się, gdy liczba encji jest duża, a metody są naprawdę identyczne. Jeśli musisz dodawać warunki „jeśli ta klasa, to inaczej”, generyczność przestała działać — cofnij się do prostszych, konkretnych klas.

> 🆕 **SQLAlchemy 2.1 — typowanie `Result` i `Row` (PEP 646).**
> W 2.1 poprawiono adnotacje typów dla `Result` i `Row` przy użyciu składni `Variadic` z PEP 646. W praktyce: IDE i `mypy` lepiej wnioskują typy przy rozpakowywaniu wierszy i przy `select(A, B)`. Dla repozytoriów ma to znaczenie w projekcjach (sekcja 8) — mniej rzutowań `cast()`, mniej „cichych” błędów typów. W 2.0 typowanie takich konstrukcji było wymagało ręcznych adnotacji.

---

## 4. Abstrakcja przez protokół

Wróćmy na chwilę do tabeli z sekcji 1.3: „domena nie importuje `sqlalchemy`”. Jak to osiągnąć, skoro implementacja repozytorium *musi* importować `sqlalchemy`?

Odpowiedź brzmi: rozdziel **deklarację** (kontrakt, co repozytorium umie) od **implementacji** (jak to robi). W Pythonie mamy do tego `typing.Protocol` — mechanizm z Pythona 3.8+, nazywany *structural typing* (typowanie strukturalne).

```python
# examples/20_book_repository_protocol.py
from collections.abc import Sequence
from typing import Protocol, runtime_checkable

from .models import Book, BookSummary


@runtime_checkable
class BookRepository(Protocol):
    """Kontrakt repozytorium książek. Nie ma tu ani śladu SQLAlchemy."""

    def get(self, book_id: int) -> Book | None: ...

    def get_by_isbn(self, isbn: str) -> Book | None: ...

    def list(self, *, limit: int = 20, offset: int = 0) -> Sequence[Book]: ...

    def list_summaries(self, *, limit: int = 50) -> Sequence[BookSummary]: ...

    def list_after(self, last_id: int | None, *, limit: int = 20) -> Sequence[Book]: ...

    def add(self, book: Book) -> Book: ...

    def remove(self, book: Book) -> None: ...

    def exists(self, isbn: str) -> bool: ...
```

Zwróć uwagę, co tu mamy i czego tu nie ma:

- **Jest**: nazwy metod, sygnatury, typy argumentów i wyniku. To jest cały kontrakt.
- **Nie ma**: żadnego `Session`, żadnego `select`, żadnego importu z `sqlalchemy`. Kontrakt czyta się jak opis „co umie bibliotekarz”, a nie „jak wygląda budynek biblioteki”.

Teraz implementacja. Ponieważ zdefiniowaliśmy kontrakt jako strukturalny, implementacja **nie musi dziedziczyć** po protokole — wystarczy, że spełnia jego strukturę (ma wymagane metody o pasujących sygnaturach).

```python
# examples/20_book_repository_sqlalchemy.py
from collections.abc import Sequence

from sqlalchemy import select
from sqlalchemy.orm import Session, selectinload

from .models import Author, Book, BookSummary


class SqlAlchemyBookRepository:
    """Implementacja BookRepository na SQLAlchemy. Spełnia protokół strukturalnie."""

    def __init__(self, session: Session) -> None:
        self._session = session

    def get(self, book_id: int) -> Book | None:
        return self._session.get(Book, book_id)

    def get_by_isbn(self, isbn: str) -> Book | None:
        stmt = select(Book).where(Book.isbn == isbn)
        return self._session.scalars(stmt).one_or_none()

    def list(self, *, limit: int = 20, offset: int = 0) -> Sequence[Book]:
        stmt = (
            select(Book)
            .options(selectinload(Book.author), selectinload(Book.categories))
            .order_by(Book.id)
            .limit(limit)
            .offset(offset)
        )
        return self._session.scalars(stmt).all()

    def list_summaries(self, *, limit: int = 50) -> Sequence[BookSummary]:
        stmt = (
            select(Book.id, Book.title, Author.name.label("author_name"))
            .join(Book.author)
            .order_by(Book.title)
            .limit(limit)
        )
        return [
            BookSummary(id=row.id, title=row.title, author_name=row.author_name)
            for row in self._session.execute(stmt)
        ]

    def list_after(self, last_id: int | None, *, limit: int = 20) -> Sequence[Book]:
        stmt = select(Book).order_by(Book.id).limit(limit)
        if last_id is not None:
            stmt = stmt.where(Book.id > last_id)
        return self._session.scalars(stmt).all()

    def add(self, book: Book) -> Book:
        self._session.add(book)
        return book

    def remove(self, book: Book) -> None:
        self._session.delete(book)

    def exists(self, isbn: str) -> bool:
        stmt = select(select(Book.id).where(Book.isbn == isbn).exists())
        return bool(self._session.scalar(stmt))
```

> 🧠 **Dlaczego typowanie strukturalne jest tu kluczowe.**
> Gdybyśmy kazali `SqlAlchemyBookRepository` dziedziczyć po `BookRepository` (jak interfejs w Javie czy klasa abstrakcyjna), to implementacja SQLAlchemy musiałaby importować `BookRepository`, a więc **zależeć od kontraktu**. Może się wydawać, że to OK — kontrakt nie importuje SQLAlchemy, więc „brud” nie ma jak wrócić.
> Problem pojawia się, gdy chcemy mieć **implementację in-memory** (sekcja 11) i **trzecią, modułową**. Dziedziczenie po wspólnym interfejsie jest wówczas wygodne, ale wiąże hierarchię klas. `Protocol` jest luźniejszy: wystarczy, że metody się zgadzają. Nie ma dziedziczenia, nie ma wspólnego przodka, nie ma ryzyka, że kontrakt „wciągnie” implementację.

### 4.1. Kontrakt a runtime

`runtime_checkable` pozwala napisać:

```python
isinstance(SqlAlchemyBookRepository(session), BookRepository)
```

Ale uwaga — sprawdzenie działa tylko po nazwach metod, nie po sygnaturach. Dla naszych potrzeb to wystarczy i rzadko się przydaje w kodzie produkcyjnym; `runtime_checkable` to głównie wygoda przy debugowaniu. Realną kontrolę typów robi `mypy` lub `pyright`, statycznie.

### 4.2. Gdzie jest granica — `Book` w protokole

I tu trzeba być uczciwym. W powyższym protokole zwracamy `Book` — czyli **klasę ORM** (dziedziczącą po `Base`, która jest z `sqlalchemy`). To znaczy, że kontrakt nadal zależy od SQLAlchemy — tylko przez typ encji, nie przez `Session`.

Czy to problem? Dla większości projektów — nie. Trzy realistyczne podejścia:

1. **Pragmatyczne (tu i teraz).** Zostawiamy `Book` jako encję ORM i mówimy: „domena i modele ORM to jedna rzecz”. To podejście z wariantu A z modułu 19. Uczciwe, proste, wystarczające dla 90% aplikacji.
2. **Pełne rozdzielenie.** Domena ma własną klasę `Book` (np. `@dataclass`), a repozytorium mapuje encję ORM na obiekt domenowy. Protokół operuje na domenie, implementacja SQLAlchemy tłumaczy. Więcej pracy, warte rozważenia tylko przy złożonej domenie.
3. **Pośrednie — DTO na wyjściu.** Kontrakt zwraca `BookSummary` (czysty `dataclass`, patrz sekcja 8) zamiast encji tam, gdzie to możliwe. Wtedy to, co przechodzi przez granicę, nie zawiera ani ziarenka SQLAlchemy.

Jesteśmy po module 19, więc pamiętasz, że wybór między tymi wariantami to decyzja architektoniczna, nie techniczna. W tym module pokazujemy wariant 1 jako bazę i wariant 3 tam, gdzie ma największy sens (raporty, listy).

> 🔬 **Pod maską — `list_summaries()` nie budzi relacji.**
> ```sql
> SELECT book.id, book.title, author.name AS author_name
> FROM book
> JOIN author ON author.id = book.author_id
> ORDER BY book.title
> LIMIT ?
> ```
> Jedno zapytanie, żadnych `selectinload`, żadnego ryzyka N+1. Wiersz (`Row`) zamieniamy na `BookSummary` w Pythonie — encja ORM w ogóle nie powstaje. To najtańszy sposób listowania danych.

---

## 5. Zapytania specyficzne i budowniczowie zapytań

### 5.1. Metody zamiast filtrów

Wyobraź sobie, że repozytorium wystawia jedną metodę:

```python
def list(self, *, author_id=None, category_id=None, published_after=None,
         only_available=None, order_by="id", limit=20, offset=0): ...
```

Taka sygnatura to **antywzorzec zwany „metodą szwajcarskiego scyzoryka”**. Wady:

- 8 parametrów, z których 12 kombinacji jest niespójnych (np. `only_available` działa tylko z `category_id`),
- nie da się zwalidować kombinacji bez dodatkowego kodu,
- wołający nie wie, co dokładnie robi `list(author_id=5, order_by="title")`,
- każde nowe kryterium zmienia jedną wspólną metodę i dotyka wszystkich jej wywołań.

Lepiej jest zapisać **konkretne pytania biznesowe jako konkretne metody**:

```python
# examples/20_repository_specific.py
from collections.abc import Sequence
from datetime import date

from sqlalchemy import func, select
from sqlalchemy.orm import Session, selectinload

from .models import Book, BookSummary, Loan


class BookRepository:
    def __init__(self, session: Session) -> None:
        self._session = session

    def list_by_author(self, author_id: int, *, limit: int = 20) -> Sequence[Book]:
        """Książki danego autora — to pytanie biznesowe, nie filtr techniczny."""
        stmt = (
            select(Book)
            .where(Book.author_id == author_id)
            .options(selectinload(Book.categories))
            .order_by(Book.title)
            .limit(limit)
        )
        return self._session.scalars(stmt).all()

    def list_published_between(self, start: int, end: int) -> Sequence[Book]:
        """Książki wydane w podanym zakresie lat włącznie."""
        stmt = (
            select(Book)
            .where(Book.published_year.between(start, end))
            .order_by(Book.published_year.desc(), Book.title)
        )
        return self._session.scalars(stmt).all()

    def top_sellers(self, *, since: date, limit: int = 10) -> Sequence[BookSummary]:
        """10 najczęściej wypożyczanych książek od podanej daty.

        Zwraca DTO, nie encje — to raport, nie obiekt do edycji.
        """
        stmt = (
            select(
                Book.id,
                Book.title,
                func.count(Loan.id).label("loan_count"),
            )
            .join(Loan, Loan.book_id == Book.id)
            .where(Loan.loaned_at >= since)
            .group_by(Book.id, Book.title)
            .order_by(func.count(Loan.id).desc())
            .limit(limit)
        )
        return [
            BookSummary(id=row.id, title=row.title, author_name=f"{row.loan_count} wypożyczeń")
            for row in self._session.execute(stmt)
        ]
```

Zapytaj siebie przy każdej nowej metodzie: **„czy ta nazwa to pytanie, które zadałby człowiek z działu, a nie z działu IT?”**

- „książki autora X” ✅ — tak pyta bibliotekarz.
- „książki z author_id = 5 i published_year >= 2000” ⚠️ — tak pyta programista sięgając do SQL.

Metody repozytorium powinny być nazwane *z perspektywy użytkownika procesu*.

> ⚠️ **Pułapka — repozytorium staje się warstwą logiki biznesowej.**
> Kiedyś spotkasz metodę `can_borrow(book_id, member_id) -> bool`. To **logika biznesowa**, a jej miejsce jest w serwisie (warstwa aplikacji) lub w modelu domenowym, nie w repozytorium. Repozytorium ma *znajdować dane*. „Czy można wypożyczyć” wymaga sprawdzenia regulaminu, limitu aktywnych wypożyczeń, historii kar — wszystko to nie jest dostępem do danych, tylko regułami. Repozytorium, które zaczyna „wiedzieć”, robi się trudne do przetestowania i złamane zostaje rozdzielenie odpowiedzialności.

### 5.2. Budowniczy zapytań — kiedy ma sens

Czasem naprawdę potrzebujesz dynamicznej kompozycji: użytkownik wybiera 4 z 7 filtrów na ekranie. Wtedy przydaje się obiekt-budowniczy, który **stopniowo** składa warunki, zamiast zmuszać do napisania metody na każdą kombinację.

```python
# examples/20_query_builder.py
from collections.abc import Sequence
from datetime import date

from sqlalchemy import Select, select

from .models import Book


class BookQuery:
    """Budowniczy zapytań o książki.

    Każda metoda mutuje stan i zwraca `self`, żeby dało się łańcuchować.
    Na końcu `.to_select()` zwraca gotowy obiekt `Select`.
    """

    def __init__(self) -> None:
        self._stmt: Select[tuple[Book]] = select(Book)

    def by_author(self, author_id: int) -> "BookQuery":
        self._stmt = self._stmt.where(Book.author_id == author_id)
        return self

    def published_after(self, year: int) -> "BookQuery":
        self._stmt = self._stmt.where(Book.published_year >= year)
        return self

    def title_contains(self, fragment: str) -> "BookQuery":
        self._stmt = self._stmt.where(Book.title.ilike(f"%{fragment}%"))
        return self

    def order_by_title(self) -> "BookQuery":
        self._stmt = self._stmt.order_by(Book.title)
        return self

    def limit(self, n: int) -> "BookQuery":
        self._stmt = self._stmt.limit(n)
        return self


class BookRepository:
    def __init__(self, session) -> None:
        self._session = session

    def search(self, query: BookQuery) -> Sequence[Book]:
        return self._session.scalars(query.to_select()).all()
```

Użycie wygląda tak:

```python
repo = BookRepository(session)
query = BookQuery().by_author(3).published_after(2000).order_by_title().limit(20)
books = repo.search(query)
```

> 💡 **Analogia — zestaw klocków vs gotowy mebel.**
> Metoda `list_by_author` to gotowy mebel ze sklepu: dostajesz konkretną rzecz, nie składasz nic. Budowniczy to zestaw klocków: składasz dowolną konfigurację, ale *musisz* wiedzieć, jak je łączyć (i to Ty odpowiadasz za sensowność złożenia). Oba podejścia są dobre — dla różnych potrzeb. Gotowy mebel jest wygodniejszy, gdy potrzebujesz dokładnie tego jednego mebla. Klocki są lepsze, gdy potrzebujesz 40 różnych mebli.

**Ale zwróć uwagę na pułapkę związaną z budowniczym.** Trzy problemy:

1. **Budowniczy wie o `Book`** — więc SQLAlchemy wycieka przez niego do wołającego, jeśli ten posługuje się `BookQuery` bezpośrednio.
2. **Nie da się łatwo przetestować in-memory**, bo `Search(BookQuery)` i tak wymaga sesji do wykonania.
3. **`to_select()` może trafić na zewnątrz** — i wtedy ktoś zrobi `session.execute(query.to_select())`, omijając repozytorium (przeciek abstrakcji).

Dlatego budowniczy ma sens tylko wtedy, gdy **żyje w całości po stronie repozytorium** — albo jako wewnętrzny mechanizm, albo jako obiekt przekazywany przez interfejs, ale zbudowany w kontrolowanej warstwie.

Alternatywa, którą warto znać: **kompozycja warunków** przez `and_`/`or_` i funkcję zwracającą listę wyrażeń.

```python
# examples/20_filter_composition.py
from sqlalchemy import ColumnElement, and_

from .models import Book


def build_book_filter(
    author_id: int | None = None,
    min_year: int | None = None,
) -> ColumnElement[bool] | None:
    """Zwraca pojedyncze wyrażenie bool albo None, jeśli brak filtrów."""
    conditions: list[ColumnElement[bool]] = []
    if author_id is not None:
        conditions.append(Book.author_id == author_id)
    if min_year is not None:
        conditions.append(Book.published_year >= min_year)
    return and_(*conditions) if conditions else None
```

Ta forma jest bardziej „surowa” niż budowniczy, ale też bardziej przejrzysta — widać, że kompozycja to zwykłe łączenie warunków.

---

## 6. Paginacja: offset/limit kontra keyset

### 6.1. Paginacja `limit`/`offset`

Najprostsza i najbardziej znana. Baza danych przeskakuje pierwsze `offset` wierszy i zwraca następne `limit`:

```python
# examples/20_pagination_offset.py
from collections.abc import Sequence
from dataclasses import dataclass
from typing import Generic

from sqlalchemy import func, select
from sqlalchemy.orm import Session

from .models import Book, ModelT


@dataclass(frozen=True, slots=True)
class Page(Generic[ModelT]):
    """Strona wyników: porcja danych + metadane paginacji."""

    items: Sequence[ModelT]
    total: int
    page: int
    size: int

    @property
    def pages(self) -> int:
        """Ile stron mieści się w całości. Zawsze co najmniej 1."""
        if self.size <= 0:
            return 1
        return max(1, -(-self.total // self.size))  # dzielenie w górę


class BookRepository:
    def __init__(self, session: Session) -> None:
        self._session = session

    def list_page(self, *, page: int = 1, size: int = 20) -> Page[Book]:
        if page < 1:
            raise ValueError("page musi być >= 1")
        if size < 1:
            raise ValueError("size musi być >= 1")

        total = int(
            self._session.scalar(select(func.count()).select_from(Book)) or 0
        )

        stmt = (
            select(Book)
            .order_by(Book.id)
            .limit(size)
            .offset((page - 1) * size)
        )
        items = self._session.scalars(stmt).all()
        return Page(items=items, total=total, page=page, size=size)
```

**Zalety:** prostota, naturalne wsparcie dla „skocz do strony 17”, policzalna liczba stron.

**Wady:** dwie, obie poważne przy dużych zbiorach.

> 🧠 **Dlaczego `OFFSET` jest drogi.**
> Baza danych nie ma czarodziejskiego sposobu, żeby „przeskoczyć” wiersze — musi przejść przez wszystkie poprzedzające i je odrzucić. Dla `OFFSET 1000000 LIMIT 20` baza w praktyce wykona pracę proporcjonalną do miliona wierszy, a zwróci 20. To jak przeglądanie stron książki palcem od pierwszej, żeby dojść do 400. — Koszt rośnie liniowo z numerem strony: $O(\text{offset})$.
> Po drugie, przy zmieniającym się zbiorze wyniki są **niestabilne**. Jeśli między żądaniem strony 1 i 2 ktoś doda nową książkę na początku, to książka, którą widziałeś ostatnio na dole strony 1, wyskoczy znowu na górze strony 2. Użytkownik zobaczy duplikat.

> 🔬 **Pod maską — `OFFSET 1000 LIMIT 20`**
> ```sql
> SELECT book.id, book.title, book.isbn, book.published_year, book.author_id
> FROM book
> ORDER BY book.id
> LIMIT 20 OFFSET 1000
> ```
> Baza przejdzie przez 1020 wierszy, żeby zwrócić 20. Na tabeli z 10 mln rekordów: katastrofa, zwłaszcza gdy zamierzasz skoczyć na stronę 5000.

### 6.2. Paginacja keyset (kursorowa)

Rozwiązanie: zamiast „przeskocz N wierszy” mówimy „daj 20 wierszy po tym konkretnym wierszu”. Do tego potrzebujemy **klucza sortowania**, który jest unikalny i stabilny — najczęściej `id`.

```python
# examples/20_pagination_keyset.py
from collections.abc import Sequence
from dataclasses import dataclass

from sqlalchemy import select
from sqlalchemy.orm import Session

from .models import Book


@dataclass(frozen=True, slots=True)
class BookCursor:
    """Kursor: punkt, od którego zaczynamy następną stronę.

    Trzyma ostatnie id z poprzedniej strony. `None` znaczy "od początku".
    """

    last_id: int | None


@dataclass(frozen=True, slots=True)
class BookPage:
    items: Sequence[Book]
    next_cursor: BookCursor | None


class BookRepository:
    def __init__(self, session: Session) -> None:
        self._session = session

    def list_page(self, cursor: BookCursor | None = None, *, limit: int = 20) -> BookPage:
        stmt = select(Book).order_by(Book.id).limit(limit + 1)  # +1, by wykryć "jest więcej"

        if cursor is not None and cursor.last_id is not None:
            stmt = stmt.where(Book.id > cursor.last_id)

        rows = self._session.scalars(stmt).all()

        has_more = len(rows) > limit
        items = list(rows[:limit])

        next_cursor = (
            BookCursor(last_id=items[-1].id) if has_more and items else None
        )
        return BookPage(items=items, next_cursor=next_cursor)
```

Zwróć uwagę na kilka szczegółów:

1. **Pobieramy `limit + 1` wierszy.** Jeśli wrócą wszystkie `limit + 1`, wiemy, że jest jeszcze coś dalej. To sztuczka pozwalająca uniknąć osobnego zapytania `COUNT(*)`.
2. **`WHERE id > cursor.last_id`** zamiast `OFFSET`. Baza z indeksem na `id` znajdzie wiersz w czasie logarytmicznym i przejdzie kilka następnych — koszt jest stały $O(\log N + \text{limit})$, niezależnie od tego, na której stronie jesteś.
3. **Kursor jest przekazywany do klienta** (np. jako pole `next_cursor` w JSON) i wraca w następnym żądaniu. Klient trzyma „zakładkę”, nie numer strony.

> 🔬 **Pod maską — keyset na stronie 5000**
> ```sql
> SELECT book.id, book.title, book.isbn, book.published_year, book.author_id
> FROM book
> WHERE book.id > 987654
> ORDER BY book.id
> LIMIT 21
> ```
> Baza skacze bezpośrednio do indeksu na `id`, zaczyna od 987654 i bierze 21 następnych. **Czas zapytania nie zależy od tego, na której stronie jesteś.** Różnica na 1 mln wierszy: `OFFSET` rośnie do setek milisekund, keyset trzyma się kilku.

### 6.3. Kiedy którą paginację

| Kryterium | offset/limit | keyset |
|---|---|---|
| Skok do strony N | ✅ trywialne | ❌ niemożliwe |
| Liczba stron widoczna w UI | ✅ z `COUNT(*)` | ⚠️ tylko „następna” |
| Stały czas na dużej stronie | ❌ $O(\text{offset})$ | ✅ $O(\log N)$ |
| Stabilność przy zmianach danych | ❌ duplikaty/pominięcia | ✅ stabilny |
| Złożoność implementacji | ✅ prosta | ⚠️ wymaga kursora i decyzji o sortowaniu |
| Sortowanie po wielu kolumnach | ✅ łatwe | ⚠️ wymaga złożonego kursora (id + inne klucze) |

**Rekomendacja:** w interfejsach „nieskończonego przewijania” i przy dużych zbiorach stosuj keyset. W panelach administracyjnych, gdzie użytkownik chce „stronę 3 z 12”, stosuj offset/limit z ograniczeniem numeru strony (np. do 1000).

> ⚠️ **Pułapka — keyset bez pełnego klucza sortowania.**
> Jeśli sortujesz po `published_year` (nieunikalne), to kursorem nie może być sama wartość roku — wiele książek ma ten sam rok. Kursor musi zawierać **cały klucz sortowania** i porównywać go jako krotkę: `WHERE (published_year, id) > (:year, :id)`. Zapominanie o tym rodzi zgubione wiersze (te, które mają ten sam rok, ale większe id niż ostatnio widziane). W SQLite i PostgreSQL porównania krotek działają natywnie; w MySQL wymagają innego zapisu.

> 🆕 **SQLAlchemy 2.1 — `selectinload` z `omit_join` i `chunksize`.**
> Jeśli Twoje repozytorium ładuje kolekcje przez `selectinload`, w 2.1 możesz zredukować liczbę zapytań w relacjach wiele-do-wielu dzięki opcji `omit_join` (pomija dodatkowe `JOIN` do tabeli pośredniej, gdy i tak masz już dane) oraz sterować rozmiarem porcji `IN (...)` przez `chunksize`. Dla paginowanych list z relacjami to realna oszczędność — nie zmienia sygnatury repozytorium, ale wpływa na liczbę rund do bazy. W 2.0 te opcje nie były dostępne w tej formie.

---

## 7. Specification — filtry jako obiekty

Czasem chcesz **przekazać warunek do repozytorium jako obiekt**, a nie zaszyć go w nazwie metody. Służy temu wzorzec **Specification** (specyfikacja), znany z dziedziny Domain-Driven Design.

Idea: filtr to obiekt z metodą, która potrafi przełożyć się na wyrażenie SQL.

```python
# examples/20_specification.py
from abc import ABC, abstractmethod
from collections.abc import Sequence

from sqlalchemy import ColumnElement, and_, or_
from sqlalchemy.orm import Session

from .models import Book


class Specification(ABC):
    """Abstrakcyjna specyfikacja: obiekt, który umie zbudować warunek SQL."""

    @abstractmethod
    def to_expression(self) -> ColumnElement[bool]: ...

    def __and__(self, other: "Specification") -> "Specification":
        return _AndSpec(self, other)

    def __or__(self, other: "Specification") -> "Specification":
        return _OrSpec(self, other)


class _AndSpec(Specification):
    def __init__(self, left: Specification, right: Specification) -> None:
        self._left = left
        self._right = right

    def to_expression(self) -> ColumnElement[bool]:
        return and_(self._left.to_expression(), self._right.to_expression())


class _OrSpec(Specification):
    def __init__(self, left: Specification, right: Specification) -> None:
        self._left = left
        self._right = right

    def to_expression(self) -> ColumnElement[bool]:
        return or_(self._left.to_expression(), self._right.to_expression())


class AuthorIs(Specification):
    def __init__(self, author_id: int) -> None:
        self._author_id = author_id

    def to_expression(self) -> ColumnElement[bool]:
        return Book.author_id == self._author_id


class PublishedAfter(Specification):
    def __init__(self, year: int) -> None:
        self._year = year

    def to_expression(self) -> ColumnElement[bool]:
        return Book.published_year >= self._year


class TitleContains(Specification):
    def __init__(self, fragment: str) -> None:
        self._fragment = fragment

    def to_expression(self) -> ColumnElement[bool]:
        return Book.title.ilike(f"%{self._fragment}%")
```

Użycie może wyglądać tak:

```python
spec = AuthorIs(3) & PublishedAfter(2000)
spec2 = TitleContains("python") | AuthorIs(7)
```

A repozytorium akceptuje specyfikację:

```python
# examples/20_specification_repo.py
from collections.abc import Sequence

from sqlalchemy import select
from sqlalchemy.orm import Session

from .models import Book
from .specification import Specification


class BookRepository:
    def __init__(self, session: Session) -> None:
        self._session = session

    def find(self, spec: Specification, *, limit: int = 100) -> Sequence[Book]:
        stmt = select(Book).where(spec.to_expression()).order_by(Book.id).limit(limit)
        return self._session.scalars(stmt).all()
```

### 7.1. Kiedy Specification naprawdę pomaga

Specification ma sens, gdy:

- **filtry są komponowane przez użytkownika** (zaawansowane wyszukiwanie, kreator raportów), a kombinacji jest zbyt wiele, by każda miała własną metodę;
- **filtry są używane w wielu kontekstach** — ta sama specyfikacja „aktywne wypożyczenie” może być używana w raporcie, w tle, w eksporcie CSV;
- **chcesz testować filtry niezależnie** od repozytorium (specyfikacja to zwykły obiekt, można go sprawdzić w izolacji).

Kiedy **nie** pomaga:

- gdy repozytorium ma 5 metod i każda z nich to konkretne pytanie biznesowe — specyfikacja wprowadzi zbędną warstwę,
- gdy filtry zależą od kontekstu wykonania (np. „użytkownik widzi tylko swoje”) — wtedy lepiej, by repozytorium samo doklejało warunek, a nie zdawało się na wołającego,
- gdy implementacje in-memory muszą powtarzać logikę specyfikacji — muszą ją *interpretować*, a to już trudne (trzeba zrobić odpowiednik `to_expression` w Pythonie).

> 💡 **Analogia — formularz wyszukiwania vs menu.**
> Menu (lista metod repozytorium) jest świetne, gdy opcji jest kilka i wiadomo, do czego służą. Specification to kafelki z filtrami w sklepie internetowym: użytkownik sam układa kombinację (rozmiar, kolor, cena, marka). Jeśli sklep ma 3 produkty i jeden filtr, kafelki są przesadą. Jeśli sklep ma 300 000 produktów i 20 możliwych kryteriów — kafelki (kompozycja) są jedynym sensownym rozwiązaniem.

> ⚠️ **Pułapka — specyfikacja zna encje ORM.**
> W powyższej implementacji `AuthorIs(3)` bezpośrednio odwołuje się do `Book.author_id`. To znaczy, że specyfikacje **nie są niezależne od SQLAlchemy** — przeciągają `ColumnElement` i `Book` do kodu, który miał być czysty. To realne ograniczenie: specyfikacje to element *infrastruktury*, nie domeny. Trzymaj je w pakiecie repozytoriów, nie w modelu domenowym.

---

## 8. Projekcje i DTO

### 8.1. Problem — pełna encja do wyświetlenia trzech pól

Wyobraź sobie ekran „lista książek”, który pokazuje: tytuł, autora i liczbę wypożyczeń. Jeśli pobierzesz pełne encje `Book` z `selectinload(Book.author)` i policzysz wypożyczenia w Pythonie, to:

- ściągniesz z bazy **wszystkie** kolumny `book` (w tym te, których nie używasz),
- utworzysz pełnometrażowe obiekty ORM, każdy z identity map, każdy ze stanem zmian,
- zbudujesz tymczasową grafę relacji tylko po to, żeby wypisać tytuł i nazwisko,
- przy 10 000 książek zjadasz pamięć, choć potrzebowałeś 3 kolumn.

### 8.2. Rozwiązanie — projekcja kolumn do DTO

**DTO** (ang. *Data Transfer Object*) to prosty obiekt, który przenosi dane między warstwami, bez zachowania, bez metod dziedzinowych. W Pythonie to zwykle `@dataclass`.

```python
# examples/20_dto.py
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class BookSummary:
    """Projekcja: tylko to, co potrzebuje lista."""

    id: int
    title: str
    author_name: str
    loan_count: int = 0
```

A repozytorium zwraca DTO z `select(...)` — bez tworzenia encji:

```python
# examples/20_repository_projections.py
from collections.abc import Sequence
from datetime import date

from sqlalchemy import func, select
from sqlalchemy.orm import Session

from .dto import BookSummary
from .models import Author, Book, Loan


class BookQueryRepository:
    def __init__(self, session: Session) -> None:
        self._session = session

    def list_summaries(self, *, limit: int = 50) -> Sequence[BookSummary]:
        """Lista książek z nazwą autora — jedno zapytanie, same potrzebne kolumny."""
        stmt = (
            select(Book.id, Book.title, Author.name.label("author_name"))
            .join(Book.author)
            .order_by(Book.title)
            .limit(limit)
        )
        return [
            BookSummary(id=row.id, title=row.title, author_name=row.author_name)
            for row in self._session.execute(stmt)
        ]

    def top_borrowed(self, *, since: date, limit: int = 10) -> Sequence[BookSummary]:
        """Top wypożyczane książki — agregacja robiona w bazie, nie w Pythonie."""
        stmt = (
            select(
                Book.id,
                Book.title,
                Author.name.label("author_name"),
                func.count(Loan.id).label("loan_count"),
            )
            .join(Book.author)
            .join(Loan, Loan.book_id == Book.id)
            .where(Loan.loaned_at >= since)
            .group_by(Book.id, Book.title, Author.name)
            .order_by(func.count(Loan.id).desc())
            .limit(limit)
        )
        return [
            BookSummary(
                id=row.id,
                title=row.title,
                author_name=row.author_name,
                loan_count=row.loan_count,
            )
            for row in self._session.execute(stmt)
        ]
```

> 🔬 **Pod maską — `list_summaries()`**
> ```sql
> SELECT book.id, book.title, author.name AS author_name
> FROM book
> JOIN author ON author.id = book.author_id
> ORDER BY book.title
> LIMIT ?
> ```
> Tylko trzy kolumny. Żadnej encji `Book`, żadnej `Author` w identity map. Dla listy 50 pozycji — jedno zapytanie, minimalny transfer.

### 8.3. Mapowanie `Row` a encje — jasna decyzja

Repozytorium ma wybór i musi być konsekwentne:

| Zwracamy | Kiedy | Konsekwencje |
|---|---|---|
| Encję ORM (`Book`) | Użycie w logice biznesowej: zmiana stanu, relacje, `session.add` | Pełna moc ORM, ale trzeba uważać na `DetachedInstanceError` i N+1 |
| DTO (`BookSummary`) | Odczyt do wyświetlenia, raport, eksport | Szybkie, lekkie, **ale niemodyfikowalne** — nie zapiszesz zmian przez sesję |
| `Row` | ❌ **Nigdy na wyjściu** | Przecieka szczegóły SQL i typy kolumn do wołającego |

Ostatni wiersz tabeli to twarda reguła. `Row` to struktura wewnętrzna `Result`. Jeśli repozytorium zwraca `Row`, to wołający musi wiedzieć, że `row.author_name` może być aliasingiem, że kolejność kolumn ma znaczenie przy rozpakowywaniu, że typy są te z bazy. To nie jest już repozytorium — to cienka nakładka tłumacząca `execute`. Mapuj `Row` na DTO albo na encję **wewnątrz** repozytorium.

> 💡 **Analogia — notatka ze sklepu, nie paragon fiskalny.**
> DTO to notatka na kartce: „Mleko, 3,50 zł”. Zawiera dokładnie to, co chciałeś zapamiętać. `Row` to paragon fiskalny: zawiera wszystko, co system sprzedaży uznał za potrzebne — NIP, numer kasy, datę z dokładnością do sekundy, pozycje w kolejności skanowania. Paragon jest precyzyjny, ale niekomfortowy do czytania i uzależnia Cię od tego, jak sklep skonstruował urządzenie. Jeśli wręczysz komuś paragon zamiast kartki, to on — nie Ty — musi znać układ paragonu.

> ⚠️ **Pułapka — DTO bez `frozen` i `slots`.**
> Bez `frozen=True` DTO jest mutowalne i ktoś (być może w przyszłości Ty) zmieni w nim pole, licząc, że zmiana zapisze się do bazy. Bez `slots=True` każda instancja DTO waży więcej (ma `__dict__`). Dla 10 000 wierszy to realna różnica pamięciowa. Używaj obu, jeśli nie masz konkretnego powodu, by nie.

---

## 9. Wyjątki i tłumaczenie błędów

### 9.1. Problem — `IntegrityError` z głębi SQLAlchemy

Załóżmy, że dodajesz książkę o ISBN, który już istnieje. Baza (i unikalny indeks, który zdefiniowaliśmy w module 20) odrzuci zapis:

```text
sqlalchemy.exc.IntegrityError: (sqlite3.IntegrityError) UNIQUE constraint failed:
book.isbn
[SQL: INSERT INTO book (title, isbn, published_year, author_id) VALUES (?, ?, ?, ?)]
```

Wyobraź sobie teraz, że ten wyjątek **ląduje bezpośrednio w handlerze API** i zostaje zamieniony na odpowiedź HTTP 500 z pełnym tracebackiem. Cztery problemy:

1. Klient dowiaduje się, że w bazie jest tabela `book` z kolumną `isbn` — to informacja techniczna, niepotrzebna, a czasem szkodliwa (ujawnianie struktury).
2. Ten sam `IntegrityError` obsługuje **każde** naruszenie ograniczenia (unikalność, klucz obcy, not-null) — nie da się z niego poznać, co dokładnie się stało, bez parsowania stringa.
3. Warstwa biznesowa nie wie, co się stało — dostaje surowy wyjątek biblioteki.
4. Ten sam wyjątek może przyjść z różnych miejsc — musisz go łapać wszędzie.

### 9.2. Rozwiązanie — tłumaczenie na wyjątki aplikacyjne

Zdefiniujmy własny mały pakiet wyjątków domenowych i mapujmy na nie błędy techniczne:

```python
# examples/20_exceptions.py


class RepositoryError(Exception):
    """Bazowy błąd repozytorium — wyjątek z warstwy dostępu do danych."""


class DuplicateISBN(RepositoryError):
    """ISBN już istnieje w katalogu."""

    def __init__(self, isbn: str) -> None:
        super().__init__(f"Książka o ISBN {isbn} już istnieje")
        self.isbn = isbn


class DuplicateEmail(RepositoryError):
    """Konto z takim e-mailem już istnieje."""

    def __init__(self, email: str) -> None:
        super().__init__(f"Konto z e-mailem {email} już istnieje")
        self.email = email


class ForeignKeyViolation(RepositoryError):
    """Dołączenie do nieistniejącego rekordu."""


class RepositoryNotFound(RepositoryError):
    """Oczekiwany rekord nie istnieje."""
```

Repozytorium tłumaczy:

```python
# examples/20_repository_translate.py
from sqlalchemy.exc import IntegrityError
from sqlalchemy.orm import Session

from .exceptions import DuplicateISBN
from .models import Book


class BookRepository:
    def __init__(self, session: Session) -> None:
        self._session = session

    def add_and_flush(self, book: Book) -> Book:
        """Dodaje książkę i od razu próbuje zapisać (flush).

        Wykrywa naruszenie unikalności ISBN i tłumaczy je na DuplicateISBN.
        NIE commituje — granica transakcji jest wyżej.
        """
        self._session.add(book)
        try:
            self._session.flush()
        except IntegrityError as exc:
            self._session.rollback()
            if "book.isbn" in str(exc.orig):
                raise DuplicateISBN(book.isbn) from exc
            raise
        return book
```

### 9.3. Gdzie tłumaczyć — repozytorium czy serwis?

To pytanie nurtuje każdego projektanta. Uczciwa odpowiedź: **zależy od tego, kiedy chcesz wyłapać błąd i jak precyzyjnie**.

| Miejsce tłumaczenia | Kiedy dobra decyzja | Kiedy zła |
|---|---|---|
| **W repozytorium**, przy `flush()` | Gdy błąd dotyczy jednego, konkretnego zapisu (ISBN, e-mail, slug) i chcesz dać wołającemu precyzyjną informację | Gdy repozytorium obsługuje wiele tabel i tłumaczenie musi znać kontekst |
| **W repozytorium**, przy `commit()` | ❌ Nigdy — repozytorium nie commituje | — |
| **W serwisie / Unit of Work** (przy `commit()`) | Gdy błąd jest wynikiem złożonej operacji (np. wypożyczenie naruszyło niezmiennik), gdy chcesz objąć tłumaczeniem kilka repozytoriów | Gdy serwis musi parsować string `IntegrityError` — brzydkie |
| **W globalnym handlerze** API | Jako ostatnia linia obrony — zamiana nieoczekiwanych błędów na `500` i log | Zawsze, gdy spodziewasz się konkretnego błędu — wtedy jest już za późno na precyzję |

**Praktyczna reguła:** jeśli błąd ma **konkretne imię w Twoim języku domeny** („ISBN już istnieje”, „e-mail zajęty”), repozytorium może go przetłumaczyć — ale tylko wtedy, gdy robi to *bezpośrednio przy operacji*, która ten błąd może wywołać. W przeciwnym razie serwis.

> ⚠️ **Pułapka — `flush()` w repozytorium zmienia semantykę transakcji.**
> Jeśli repozytorium wywołuje `flush()`, to w tym momencie SQL leci do bazy. Nie ma jeszcze `commit`, więc można jeszcze wycofać — ale błąd `IntegrityError` może pojawić się wtedy, gdy **połowa** operacji złożonej jest już wysłana. Musisz wtedy wywołać `rollback()` i pamiętać, że po rollbacku sesja jest w stanie „do naprawy”, a dalsze operacje wymagają nowej sesji lub ostrożnego wznowienia. W praktyce: albo repozytorium samo nigdy nie flushuje (i wtedy wszystkie błędy lądują w UoW, gdzie są obsługiwane z pełnym kontekstem), albo flushuje i wtedy **musi obsłużyć swój własny rollback** — jak w powyższym przykładzie.
> Wybór jest projektowy. Prostsza reguła: **niech repozytorium nie flushuje** (poza metodami, które jawnie nazywają to w nazwie: `add_and_flush`). Wtedy `commit()` w UoW jest jedynym miejscem, gdzie moga pojawić się `IntegrityError`, i wszystko obsługujesz w jednym miejscu. Zobaczysz to w module 21.

> 🧠 **Dlaczego repozytorium nie powinno „przeciekać” `IntegrityError`.**
> Wyobraź sobie, że piszesz serwis, który ma zwrócić ładny komunikat „Ten ISBN jest już w katalogu”. Jeśli repozytorium rzuca `sqlalchemy.exc.IntegrityError`, to serwis musi:
> - wiedzieć, że w ogóle istnieje SQLAlchemy,
> - zaimportować `IntegrityError` z `sqlalchemy.exc`,
> - przeanalizować string, żeby dowiedzieć się, o które ograniczenie chodzi.
> To trzy sposoby na to, żeby warstwa aplikacji zależała od biblioteki bazy danych. Tłumaczenie wyjątków w repozytorium to jedna z niewielu rzeczy, które jednoznacznie opłacają się także w małym projekcie.

---

## 10. Krytyka wzorca repozytorium w projektach SQLAlchemy

Teraz część najważniejsza, którą pomija 90% poradników o wzorcach: **kiedy repozytorium szkodzi**.

### 10.1. „`Session` to już Unit of Work i Query Object”

Prawda w 100%. SQLAlchemy `Session`:

- **jest Unit of Work** — zbiera zmiany, sortuje je, batchuje, wysyła na `flush`,
- **jest Query Object** — `select()` buduje zapytanie, `session.execute()` je wykonuje,
- **jest Identity Map** — gwarantuje, że ten sam wiersz daje ten sam obiekt Pythona,
- **jest transakcją** — `session.begin()`, `commit()`, `rollback()`.

Jeśli dodać do tego repozytorium, którego jedyne zadanie to wywołać `session.get()`, to mamy **trzy warstwy pośrednie dla operacji, która sprowadza się do jednego wywołania**. To jest tzw. **ceremonialny wzorzec** (pattern for the sake of pattern).

> 💡 **Analogia — sekretarka, która powtarza Twoje zdanie.**
> Repozytorium, które ma tylko `get`, `add`, `remove` i przekazuje je do `Session`, przypomina sekretarkę, która nic nie robi poza powtórzeniem Twojego zdania do kolegi obok: „Powiedz mu, że chcę kawę” — „On mówi, że chce kawę”. Nic nie wnosi poza dodatkowym krokiem. Dopiero gdy sekretarka **ma wiedzę, którą się opłaca mieć** (zna rozkład dnia szefa, umie odfiltrować nieaktualne sprawy) — jest wartościowa. Repozytorium ma sens, gdy *chroni regułę*, a nie gdy przekazuje wywołanie.

### 10.2. Kiedy repozytorium się opłaca — lista kontrolna

Zadaj sobie te pytania. Im więcej „tak”, tym bardziej repozytorium ma sens.

| Pytanie | Znaczenie |
|---|---|
| Czy logika dostępu jest nietrywialna? (filtry, reguły „miękkiego usuwania”, widoczność wg roli) | Repozytorium daje jedno miejsce na regułę |
| Czy te same zapytania są używane w wielu miejscach? | Repozytorium redukuje duplikację |
| Czy chcesz testować logikę biznesową bez bazy? | Implementacja in-memory staje się możliwa |
| Czy przewidujesz zmianę magazynu (cache, zewnętrzne API, podział danych)? | Repozytorium izoluje zmianę |
| Czy domena jest złożona i chcesz trzymać SQLAlchemy na uboczu? | Repozytorium (z protokołem) pozwala |
| Czy masz więcej niż 5–7 encji? | Repozytoria per encja zaczynają być spójne |

### 10.3. Kiedy repozytorium szkodzi — lista kontrolna

| Objaw | Dlaczego źle |
|---|---|
| Repozytorium dla każdej tabeli, nawet dla 2 metod | Ceremonializm, utrudnia nawigację |
| Repozytorium zwraca `Select` („bo klient może dograć filtry”) | Abstrakcja wycieka — repozytorium nic nie chroni |
| `commit()` w repozytorium | Łamie atomowość operacji złożonych, blokuje UoW |
| `Session` tworzona w konstruktorze repozytorium | Uniemożliwia podzielenie transakcji między repozytoria |
| Logika biznesowa w repozytorium | Repozytorium staje się serwisem, gubi sens |
| Konwersja encji na DTO w repozytorium, ale i tak wsadzane do encji | Dwie prawdy o tym samym |
| „Grube” repozytoria z 40 metodami | Trudne do zrozumienia, wraca problem sprzed repozytorium |

### 10.4. Wersja minimalistyczna — moduł z funkcjami

W małych projektach (1–2 osoby, do 10 encji) często lepsza jest wersja minimalna: **moduł z funkcjami** zamiast klasy.

```python
# examples/20_repository_module.py
"""Dostęp do danych w wersji modułowej — bez klasy repozytorium.

Funkcje przyjmują sesję jako argument. To spełnia cel (jedno miejsce
na reguły dostępu) bez ceremonii klasy i bez stanu.
"""
from collections.abc import Sequence

from sqlalchemy import select
from sqlalchemy.orm import Session, selectinload

from .models import Book


def get_book(session: Session, book_id: int) -> Book | None:
    return session.get(Book, book_id)


def get_book_by_isbn(session: Session, isbn: str) -> Book | None:
    stmt = select(Book).where(Book.isbn == isbn)
    return session.scalars(stmt).one_or_none()


def list_books(session: Session, *, limit: int = 20, offset: int = 0) -> Sequence[Book]:
    stmt = (
        select(Book)
        .options(selectinload(Book.author))
        .order_by(Book.id)
        .limit(limit)
        .offset(offset)
    )
    return session.scalars(stmt).all()


def add_book(session: Session, book: Book) -> Book:
    session.add(book)
    return book


def remove_book(session: Session, book: Book) -> None:
    session.delete(book)


def book_exists(session: Session, isbn: str) -> bool:
    stmt = select(select(Book.id).where(Book.isbn == isbn).exists())
    return bool(session.scalar(stmt))
```

**Zalety:** brak stanu, brak hierarchii, brak problemu z „jak utworzyć repozytorium”, zero ceremonii. **Wady:** sesja jest wszędzie widoczna (jej typ pojawia się w sygnaturach), brak naturalnego miejsca dla fake'a in-memory (choć można zrobić protokół nad funkcjami), mniej „architektoniczny” wygląd — co dla części recenzentów kodu ma znaczenie.

To jest **równie poprawna** architektura jak klasy. W małym projekcie wybierz ją śmiało. W dużym — klasy mają tę przewagę, że tworzą jasne granice i można je wstrzykiwać (moduł 21).

### 10.5. Realne doświadczenia zespołów

Zebrałem najczęstsze wzorce z publicznych dyskusji i doświadczeń ludzi pracujących z SQLAlchemy:

1. **„Zaczęliśmy od generycznego repozytorium, po roku mieliśmy 8 podklas z nadpisanymi metodami. Sens generyczności zniknął.”** — typowe, potwierdza sekcję 3.
2. **„Największa wartość repozytoriów to tłumaczenie wyjątków i jedno miejsce na reguły widoczności.”** — to dwie rzeczy, które trudno zrobić inaczej.
3. **„Dla prostego CRUD-u repozytorium było czystą stratą. Uprościliśmy do funkcji i nikt nie tęsknił za klasami.”**
4. **„Używamy repozytoriów, bo mamy 3 interfejsy do tej samej bazy: API, CLI i worker. Jedno miejsce na zapytania bardzo pomaga.”**
5. **„Problem zaczął się, gdy repozytoria zaczęły commitować. Wtedy transakcja złożona z 3 repozytoriów przestała być transakcją.”** — wraca do sedna.

Wniosek: repozytorium to narzędzie, nie obowiązek. Ma sens tam, gdzie ma co chronić.

> 🧪 **Ćwiczenie — audyt własnego projektu.**
> Otwórz kod, który napisałeś do tej pory w kursie (moduły 07–18). Policz, ile razy pojawia się `select(Book)` lub `select(...)`. Jeśli więcej niż 3 razy i w więcej niż jednym pliku — to konkretny sygnał, że repozytorium pomoże. Zapisz, jakie reguły (filtry, `deleted_at`, `selectinload`) się powtarzają.

---

## 11. Testowanie repozytoriów

### 11.1. Dwa rodzaje testów, dwa różne cele

- **Testy integracyjne** repozytorium: sprawdzają, że **SQL jest poprawny** — że zapytanie zwraca właściwe wiersze, że złączenia działają, że `selectinload` nie generuje duplikatów. Wymagają prawdziwej (choćby in-memory SQLite) bazy.
- **Testy jednostkowe serwisu**: sprawdzają, że **logika biznesowa** jest poprawna. Repozytorium jest tu podstawiane fake'iem — żadnej bazy.

Te dwa rodzaje testów są komplementarne. Test integracyjny na SQLite nie powie Ci, czy Twoja logika „można wypożyczyć” działa, bo nie testuje logiki. Test jednostkowy z fake'iem nie powie Ci, czy SQL jest poprawny, bo SQL nie jest wykonywany.

### 11.2. Fake repository — implementacja in-memory

```python
# examples/20_fake_book_repository.py
from collections.abc import Sequence

from .dto import BookSummary
from .exceptions import DuplicateISBN
from .models import Book


class InMemoryBookRepository:
    """Implementacja repozytorium w pamięci, do testów jednostkowych.

    Trzyma książki w słowniku indeksowanym po id oraz drugi słownik ISBN->id
    do szybkiego wyszukiwania po ISBN (odpowiednik indeksu unikalnego w bazie).
    """

    def __init__(self) -> None:
        self._by_id: dict[int, Book] = {}
        self._id_by_isbn: dict[str, int] = {}
        self._next_id = 1

    def get(self, book_id: int) -> Book | None:
        return self._by_id.get(book_id)

    def get_by_isbn(self, isbn: str) -> Book | None:
        book_id = self._id_by_isbn.get(isbn)
        return self._by_id.get(book_id) if book_id is not None else None

    def list(self, *, limit: int = 20, offset: int = 0) -> Sequence[Book]:
        ordered = [self._by_id[k] for k in sorted(self._by_id)]
        return ordered[offset : offset + limit]

    def list_summaries(self, *, limit: int = 50) -> Sequence[BookSummary]:
        ordered = [self._by_id[k] for k in sorted(self._by_id)][:limit]
        return [
            BookSummary(id=b.id, title=b.title, author_name="—")
            for b in ordered
        ]

    def list_after(self, last_id: int | None, *, limit: int = 20) -> Sequence[Book]:
        ids = [k for k in sorted(self._by_id) if last_id is None or k > last_id]
        return [self._by_id[k] for k in ids[:limit]]

    def add(self, book: Book) -> Book:
        if book.isbn in self._id_by_isbn:
            raise DuplicateISBN(book.isbn)
        book.id = self._next_id
        self._next_id += 1
        self._by_id[book.id] = book
        self._id_by_isbn[book.isbn] = book.id
        return book

    def remove(self, book: Book) -> None:
        if book.id in self._by_id:
            del self._by_id[book.id]
            self._id_by_isbn.pop(book.isbn, None)

    def exists(self, isbn: str) -> bool:
        return isbn in self._id_by_isbn
```

> ⚠️ **Pułapka — fake musi wiernie odtwarzać reguły.**
> `InMemoryBookRepository.add` sam pilnuje unikalności ISBN i sam nadaje `id` (bo baza robi to za nas w prawdziwej implementacji). Gdyby fake tego nie robił, testy przechodziłyby na fake'u, a **prawdziwy kod łamałby się na bazie** — bo serwis, który założył, że `id` już jest, dostałby `None`. To klasyczna pułapka **„test green, prod red”**.
> Zasada: fake musi być **testowany** tymi samymi testami co implementacja SQLAlchemy (testy kontraktowe, patrz niżej) — inaczej rozjeżdża się z prawdą.

### 11.3. Testy kontraktowe — wspólny zestaw testów dla obu implementacji

Skoro mamy dwie implementacje tego samego protokołu, warto napisać **jedną suitę testów**, którą uruchamiamy na obu. To gwarantuje, że fake jest wierny.

```python
# tests/test_book_repository_contract.py
import pytest

from examples.models import Base, Book
from examples.pagination import BookCursor


class BookRepositoryContract:
    """Zestaw testów kontraktowych. Podklasy dostarczają `make_repo()`."""

    def make_repo(self):
        raise NotImplementedError

    def make_book(self, isbn: str = "978-83-000-0000-1") -> Book:
        return Book(title="Wzorce", isbn=isbn, published_year=2002)

    def test_add_and_get_by_id(self):
        repo = self.make_repo()
        book = repo.add(self.make_book())
        fetched = repo.get(book.id)
        assert fetched is not None
        assert fetched.isbn == book.isbn

    def test_get_by_isbn_returns_none_for_missing(self):
        repo = self.make_repo()
        assert repo.get_by_isbn("000") is None

    def test_duplicate_isbn_raises(self):
        from examples.exceptions import DuplicateISBN

        repo = self.make_repo()
        repo.add(self.make_book("ISBN-A"))

        with pytest.raises(DuplicateISBN):
            repo.add(self.make_book("ISBN-A"))

    def test_list_after_returns_next_page(self):
        repo = self.make_repo()
        books = [repo.add(self.make_book(f"ISBN-{i}")) for i in range(25)]

        first = repo.list_after(None, limit=10)
        assert len(first) == 10

        second = repo.list_after(first[-1].id, limit=10)
        assert len(second) == 10
        assert second[0].id > first[-1].id

    def test_exists(self):
        repo = self.make_repo()
        repo.add(self.make_book("ISBN-X"))
        assert repo.exists("ISBN-X") is True
        assert repo.exists("ISBN-Y") is False
```

Dwie konkretne klasy testowe dla obu implementacji:

```python
# tests/test_book_repository_contract.py (cd.)
from sqlalchemy import create_engine
from sqlalchemy.orm import Session, sessionmaker
from sqlalchemy.pool import StaticPool

from examples.fake_book_repository import InMemoryBookRepository
from examples.models import Base
from examples.repository_sqlalchemy import SqlAlchemyBookRepository


class TestInMemoryRepository(BookRepositoryContract):
    def make_repo(self) -> InMemoryBookRepository:
        return InMemoryBookRepository()


class TestSqlAlchemyRepository(BookRepositoryContract):
    def make_repo(self) -> SqlAlchemyBookRepository:
        engine = create_engine(
            "sqlite://",
            connect_args={"check_same_thread": False},
            poolclass=StaticPool,
        )
        Base.metadata.create_all(engine)
        self._engine = engine
        session = Session(engine)
        self._session = session
        return SqlAlchemyBookRepository(session)
```

Wyjaśnienie techniczne, bo to ważne:

- **`sqlite://`** to baza w pamięci. Każde nowe połączenie to **pusta** baza — dlatego nie wystarczy samo `create_engine`.
- **`poolclass=StaticPool`** mówi: trzymaj jedno połączenie i przekazuj je dalej, nie otwieraj nowych. Bez tego każda sesja dostanie nowe, puste połączenie i test padnie na `no such table`.
- **`check_same_thread=False`** wyłącza ograniczenie SQLite dotyczące wątków — `pytest` czasem uruchamia rzeczy w innym wątku, a nasze połączenie jest współdzielone.

> 🔬 **Pod maska — dlaczego `StaticPool` jest tu konieczny.** SQLite in-memory żyje dopóki otwarte jest połączenie (albo do momentu jego zamknięcia). `sqlite://` domyślnie używa `SingletonThreadPool` albo `NullPool` (zależnie od wersji i sterownika), co może prowadzić do tego, że każde nowe połączenie nie ma tabel, które stworzyłeś w innym. `StaticPool` gwarantuje jedno wspólne połączenie na wszystkie operacje — więc schemat i dane są powtarzalne.

### 11.4. Test integracyjny na SQLite — konkretny przykład

```python
# tests/test_book_repository_integration.py
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import Session
from sqlalchemy.pool import StaticPool

from examples.models import Author, Base, Book
from examples.repository_sqlalchemy import SqlAlchemyBookRepository


@pytest.fixture
def session():
    engine = create_engine(
        "sqlite://",
        connect_args={"check_same_thread": False},
        poolclass=StaticPool,
    )
    Base.metadata.create_all(engine)
    with Session(engine) as s:
        yield s


@pytest.fixture
def repo(session: Session) -> SqlAlchemyBookRepository:
    return SqlAlchemyBookRepository(session)


def test_get_by_isbn_returns_book_with_author(repo, session):
    author = Author(name="Martin Fowler")
    book = Book(title="PoEAA", isbn="978-0-321-12742-6", author=author)
    session.add(book)
    session.commit()

    found = repo.get_by_isbn("978-0-321-12742-6")

    assert found is not None
    assert found.title == "PoEAA"
    assert found.author.name == "Martin Fowler"
```

### 11.5. Test jednostkowy serwisu z fake'iem

```python
# tests/test_borrow_service.py
from examples.fake_book_repository import InMemoryBookRepository
from examples.models import Book


class BorrowService:
    """Serwis demonstracyjny — używa tylko kontraktu repozytorium."""

    def __init__(self, books) -> None:
        self._books = books

    def register_new_book(self, title: str, isbn: str) -> bool:
        """Rejestruje książkę. Zwraca False, gdy ISBN już istnieje."""
        from examples.exceptions import DuplicateISBN

        try:
            self._books.add(Book(title=title, isbn=isbn))
        except DuplicateISBN:
            return False
        return True


def test_register_new_book_returns_true():
    repo = InMemoryBookRepository()
    service = BorrowService(repo)

    assert service.register_new_book("Nowa", "ISBN-1") is True
    assert repo.exists("ISBN-1") is True


def test_register_duplicate_returns_false():
    repo = InMemoryBookRepository()
    service = BorrowService(repo)
    service.register_new_book("Pierwsza", "ISBN-1")

    assert service.register_new_book("Druga", "ISBN-1") is False
    assert len(repo.list(limit=100)) == 1
```

Ten test **nie potrzebuje bazy** w ogóle. Można go uruchomić w milisekundach, można go uruchomić bez SQLite, bez storywera. To jest cała nagroda za wprowadzenie protokołu: testujesz logikę bez infrastruktury.

> ⚠️ **Pułapka — fake, który odbiega od implementacji.**
> Bez testów kontraktowych prawie zawsze zdarza się, że fake i implementacja SQLAlchemy rozjadą się po kilku miesiącach. Dodajesz do prawdziwej implementacji filtr „nie pokazuj usuniętych”, zapominasz o fake'u — i testy przechodzą, ale system w production zwraca inne dane. **Zawsze pisz testy kontraktowe** dla obu implementacji.

> 🆕 **SQLAlchemy 2.1 — lepsze komunikaty błędów sesji.**
> W 2.1 poprawiono komunikaty o stanie sesji (np. po rollbacku, w trybie prepared). Przy testach repozytoriów, które robią flush i rollback (sekcja 9), to realna oszczędność czasu — komunikat mówi wprost, że sesja wymaga rollbacku, zamiast rzucać mniej zrozumiałą wiadomość. Nie zmienia API, więc nie musisz zmieniać kodu testów.

---

## 12. Pełny przykład do uruchomienia

Zbierzmy wszystko w kompletny, samowystarczalny przykład. Katalog `examples/`:

```text
examples/
├── models.py
├── dto.py
├── exceptions.py
├── book_repository.py       # protokół
├── book_repository_sql.py   # implementacja SQLAlchemy
├── fake_book_repository.py  # implementacja in-memory
└── demo_repository.py       # skrypt demonstracyjny
```

```python
# examples/models.py
from __future__ import annotations

from sqlalchemy import Column, ForeignKey, String, Table
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    pass


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

    books: Mapped[list[Book]] = relationship(back_populates="author")


class Category(Base):
    __tablename__ = "category"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(80), unique=True)


class Book(Base):
    __tablename__ = "book"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    isbn: Mapped[str] = mapped_column(String(20), unique=True, index=True)
    published_year: Mapped[int | None] = mapped_column(default=None)
    author_id: Mapped[int | None] = mapped_column(ForeignKey("author.id"), default=None)

    author: Mapped[Author | None] = relationship(back_populates="books")
    categories: Mapped[list[Category]] = relationship(secondary=book_category)
```

Plik demonstracyjny:

```python
# examples/demo_repository.py
"""Uruchamianie: python -m examples.demo_repository  (z katalogu nadrzędnego)."""
from sqlalchemy import create_engine
from sqlalchemy.orm import Session
from sqlalchemy.pool import StaticPool

from examples.book_repository_sql import SqlAlchemyBookRepository
from examples.exceptions import DuplicateISBN
from examples.models import Author, Base, Book


def main() -> None:
    engine = create_engine(
        "sqlite://",
        connect_args={"check_same_thread": False},
        poolclass=StaticPool,
        echo=True,  # pokazujemy SQL — w produkcji echo=False
    )
    Base.metadata.create_all(engine)

    with Session(engine) as session, session.begin():
        author = Author(name="Martin Fowler")
        session.add(author)
        session.flush()  # potrzebujemy autora z id — albo użyjemy relacji obiektowej

        session.add_all(
            [
                Book(title="PoEAA", isbn="978-0-321-12742-6", published_year=2002, author=author),
                Book(title="Refactoring", isbn="978-0-201-48567-7", published_year=1999, author=author),
            ]
        )

    with Session(engine) as session:
        repo = SqlAlchemyBookRepository(session)

        first = repo.list(limit=1)
        print("Pierwsza książka:", first[0].title)

        page = repo.list_after(first[-1].id, limit=1)
        if page:
            print("Następna:", page[0].title)

        print("Czy jest ISBN?", repo.exists("978-0-201-48567-7"))

        for summary in repo.list_summaries(limit=5):
            print(f"[{summary.id}] {summary.title} — {summary.author_name}")

        try:
            session.add(Book(title="Dubel", isbn="978-0-201-48567-7"))
            session.flush()
        except DuplicateISBN as exc:
            print("Oczekiwany błąd:", exc)
```

Aby uruchomić, potrzeba `sqlalchemy>=2.0`. Wystarczy:

```bash
python -m pip install "sqlalchemy>=2.0"
python -m examples.demo_repository
```

> 🔬 **Pod maską — `list_summaries()` w działaniu.** Zamiast 3 zapytań (książki, autor, relacje) dostajesz **jedno** — z `JOIN author` i trzema kolumnami. To wzorzec, do którego będziesz wracał przy każdej liście w aplikacji: listy są dla DTO, nie dla encji.

---

## Podsumowanie

Zabierz ze sobą te punkty:

1. **Repozytorium = bibliotekarz.** Zamienia rozproszone zapytania na nazwane metody z języka biznesu. Chroni reguły dostępu i isola SQL-a od reszty aplikacji.
2. **Nigdy nie commituj w repozytorium.** Granica transakcji leży wyżej — w serwisie lub Unit of Work (moduł 21). Pozwala to złożyć kilka repozytoriów w jedną atomową operację.
3. **Wstrzykuj sesję z zewnątrz.** Repozytorium nie tworzy własnej `Session`. Inaczej nie da się objąć dwóch repozytoriów jedną transakcją.
4. **Repozytorium nie zwraca `Select` ani `Row`.** To przeciek abstrakcji. Na wyjściu: encja albo DTO.
5. **Generyczne repozytorium opłaca się tylko dla dużych, jednorodnych zbiorów.** Gdy pojawia się „jeśli ta klasa, to inaczej” — generyczność przestała działać.
6. **Protokół (`typing.Protocol`) daje testowalność.** Kontrakt bez SQLAlchemy, implementacja SQLAlchemy, fake in-memory — trzy części układanki.
7. **Paginacja: offset/limit dla UI, keyset dla stabilności i wydajności.** Keyset wymaga pełnego klucza sortowania.
8. **DTO dla raportów i list, encje dla logiki.** Encje ORM mają stan i identity map — nie przechodzą dobrze przez granice warstw.
9. **Tłumacz wyjątki.** `IntegrityError` nie może dopłynąć do API. Zamień na własne wyjątki z nazwą biznesową.
10. **Testuj kontraktem, nie mockiem.** Te same testy na SQLAlchemy i fake'u. Wtedy fake nie rozjeżdża się z prawdą.
11. **Repozytorium to narzędzie, nie obowiązek.** W prostym CRUD-zie funkcje modułowe są równie dobre, a mniej ceremonii.

---

## Ćwiczenia

### Ćwiczenie 1 — rozbudowa protokołu i fake'a

Dodaj do protokołu `BookRepository` metodę `list_by_author(self, author_id: int, *, limit: int = 50) -> Sequence[Book]`:

1. Zaimplementuj ją w `SqlAlchemyBookRepository` (wykorzystaj `select(Book).where(Book.author_id == author_id)`).
2. Zaimplementuj ją w `InMemoryBookRepository`.
3. Dodaj do testów kontraktowych test, który dodaje trzech autorów i sprawdza, że metoda zwraca tylko książki wybranego.

### Ćwiczenie 2 — keyset z kursorem złożonym

Rozszerz paginację keyset tak, aby sortować po `(published_year DESC, id ASC)`. Kursor musi teraz trzymać obie wartości. Napisz warunek `WHERE` porównujący całą krotkę — w SQLite i PostgreSQL użyj porównania krotek w formie `(pub_year, id) < (:year, :id)` (zwróć uwagę na kierunek przy `DESC`). Napisz test kontraktowy, który dodaje 30 książek z powtórzonymi latami i sprawdza, że kolejne strony nie gubią ani nie dublują wierszy.

### Ćwiczenie 3 — projekcja z agregacją

Napisz metodę repozytorium `list_top_authors(self, *, limit: int = 10) -> Sequence[AuthorStats]` zwracającą DTO z nazwą autora i liczbą jego książek w katalogu. Zapytanie realizuj jednym `select` z `JOIN` i `GROUP BY` (bez ładowania encji `Author` do listy). Napisz test integracyjny, który dowodzi, że zapytanie zwraca dokładnie `limit` wierszy, w kolejności malejącej liczby książek.

### Rozwiązania

**Ćwiczenie 1 — rozbudowa protokołu.**

```python
# (fragment) examples/book_repository.py
def list_by_author(self, author_id: int, *, limit: int = 50) -> Sequence[Book]: ...
```

W implementacji SQLAlchemy:

```python
def list_by_author(self, author_id: int, *, limit: int = 50) -> Sequence[Book]:
    stmt = (
        select(Book)
        .where(Book.author_id == author_id)
        .options(selectinload(Book.categories))
        .order_by(Book.title)
        .limit(limit)
    )
    return self._session.scalars(stmt).all()
```

W fake:

```python
def list_by_author(self, author_id: int, *, limit: int = 50) -> Sequence[Book]:
    books = [b for b in self._by_id.values() if b.author_id == author_id]
    books.sort(key=lambda b: (b.title, b.id))
    return books[:limit]
```

Test kontraktowy:

```python
def test_list_by_author_returns_only_selected_author(self):
    repo = self.make_repo()
    a1 = repo.add(self.make_book("ISBN-A1"))
    a1.author_id = 1
    a2 = repo.add(self.make_book("ISBN-A2"))
    a2.author_id = 2
    a3 = repo.add(self.make_book("ISBN-A3"))
    a3.author_id = 1

    result = repo.list_by_author(1)
    assert {b.isbn for b in result} == {"ISBN-A1", "ISBN-A3"}
```

Zwróć uwagę, jak trik z przypisaniem `author_id` po `add` działa w obu implementacjach: fake nie waliduje FK, SQLAlchemy też nie (nie ma pełnego FK check w SQLite bez `PRAGMA`), więc test jest wierny dla obu.

**Ćwiczenie 2 — keyset z kursorem złożonym.**

```python
@dataclass(frozen=True, slots=True)
class BookCursor:
    published_year: int
    id: int
```

Warunek:

```python
from sqlalchemy import tuple_

...

def list_after(self, cursor: BookCursor | None, *, limit: int = 20) -> Sequence[Book]:
    stmt = (
        select(Book)
        .order_by(Book.published_year.desc(), Book.id.asc())
        .limit(limit + 1)
    )
    if cursor is not None:
        # (published_year, id) < (cursor.year, cursor.id)  <-- uwaga na kierunek DESC
        stmt = stmt.where(
            tuple_(Book.published_year, Book.id) < (cursor.published_year, cursor.id)
        )
    rows = self._session.scalars(stmt).all()
    ...
```

**Trudność:** porównanie krotek musi odzwierciedlać kolejność sortowania — `DESC` na pierwszym składniku oznacza `<`, ale to działa tylko wtedy, gdy drugi składnik w krotce również rośnie w tym samym kierunku. W praktyce porównania krotek używa się głównie z jednolitym kierunkiem sortowania (wszystkie ASC lub wszystkie DESC). Przy mieszanym trzeba rozbić warunek na część ostrą i „ogonek”:

```sql
WHERE (published_year < :y)
   OR (published_year = :y AND id > :id)
```

Wersja z `tuple_` jest czytelniejsza, ale wymaga spójnego kierunku. Wart przetestowania na obu dialektach.

**Ćwiczenie 3 — projekcja z agregacją.**

```python
# examples/dto.py
@dataclass(frozen=True, slots=True)
class AuthorStats:
    author_id: int
    author_name: str
    book_count: int
```

```python
# examples/book_repository_sql.py
def list_top_authors(self, *, limit: int = 10) -> Sequence[AuthorStats]:
    stmt = (
        select(
            Author.id.label("author_id"),
            Author.name.label("author_name"),
            func.count(Book.id).label("book_count"),
        )
        .join(Book, Book.author_id == Author.id)
        .group_by(Author.id, Author.name)
        .order_by(func.count(Book.id).desc())
        .limit(limit)
    )
    return [
        AuthorStats(
            author_id=row.author_id,
            author_name=row.author_name,
            book_count=row.book_count,
        )
        for row in self._session.execute(stmt)
    ]
```

Test:

```python
def test_top_authors_returns_limited_sorted(session):
    a1 = Author(name="A")
    a2 = Author(name="B")
    session.add_all([a1, a2])
    session.flush()
    session.add_all(
        [Book(title=f"B{i}", isbn=f"I{i}", author_id=a1.id) for i in range(5)]
        + [Book(title="C", isbn="CI", author_id=a2.id)]
    )
    session.commit()

    repo = SqlAlchemyBookRepository(session)
    stats = repo.list_top_authors(limit=1)
    assert len(stats) == 1
    assert stats[0].author_name == "A"
    assert stats[0].book_count == 5
```

**Zwróć uwagę:** nie tworzymy obiektów `Author` w wyniku — tylko `AuthorStats`. To udowadnia, że projekcja działa i nie obciąża identity map.

---

## Najczęstsze błędy i jak je czytać

| Komunikat / objaw | Przyczyna | Naprawa |
|---|---|---|
| `DetachedInstanceError` przy dostępie do `book.author` | Encja 
| `TypeError: Instance and class checks can only be used with @runtime_checkable protocols` | `isinstance(obj, BookRepository)` na protokole bez `@runtime_checkable` | Albo dodaj `@runtime_checkable` do protokołu, albo (lepiej) nie sprawdzaj typów w runtime — polegaj na `mypy` |
| `AttributeError: 'Select' object has no attribute 'all'` | Repozytorium zwróciło `Select` na zewnątrz i ktoś próbuje na nim wołać metody `Result` | Zwracaj `Sequence[Book]` / `Book`; `Select` zostaw wewnątrz repozytorium |
| `DetachedInstanceError` mimo `selectinload` | Ładowanie działało, ale potem sesja została zamknięta, a warstwa wyżej odczytała **inną** relację (np. `Book.categories`, której nie eager-loadowałeś) | Eager-loaduj wszystkie relacje używane wyżej, albo zwracaj DTO. `raiseload("*")` w testach wyłapie przeoczenia |
| `NoResultFound: No row was found when one was required` | Użycie `session.scalars(stmt).one()` tam, gdzie dopuszczalna jest nieobecność | Zamień na `.one_or_none()` i zwróć `None`; `NoResultFound` zostaw tam, gdzie brak wiersza jest błędem niezmiennika |
| `MultipleResultsFound` | `get_by_isbn` zwraca `.one()`, a w bazie są duplikaty (brak ograniczenia `unique`) | Dodaj `unique=True` na kolumnę + migrację ([Moduł 16](16_alembic_migracje.md)); to repozytorium nie może zagwarantować |
| `InvalidRequestError: This session is in 'inactive' state` | Sesja została `close()`owana, a potem ktoś na niej pracował (typowe przy współdzieleniu sesji między warstwami) | Nie zamykaj sesji w repozytorium; sesją zarządza warstwa wyżej (UoW / dependency) |
| Testy „przechodzą” na fake, a padają na bazie | Fake nie odtwarza semantyki bazy (kaskady, ograniczenia, kolejność `ORDER BY` przy równych kluczach) | Test kontraktowy (jedna suite, dwie implementacje) — sekcja 13.4. Fake pokrywa tylko to, czego serwis faktycznie używa |
| Repozytorium „działa”, ale zapytania są dwa razy wolniejsze niż były | Dołożenie `selectinload` na domyślnej ścieżce, gdzie relacja nie była potrzebna | Zmieniaj strategię ładowania per metoda, nie globalnie; mierz liczbę zapytań ([Moduł 11](11_ladowanie_i_n_plus_1.md), [Moduł 17](17_wydajnosc.md)) |
| Duplikaty w `list_*` po dołączeniu tabeli N:M | JOIN rozmnożył wiersze (jedna książka × trzy kategorie = trzy wiersze) | Dodaj `.distinct()` do zapytania albo użyj `Book.categories.any(...)` (skorelowane `EXISTS`) zamiast JOIN-a |
| `ImportError` / cykl importów między `models` a `repositories` | Model importuje repozytorium (np. dla adnotacji), repozytorium importuje model | Adnotacje pod `from __future__ import annotations` lub `TYPE_CHECKING`; repozytorium nigdy nie jest importowane przez model |

Kilka uwag do czytania tych komunikatów:

- **Zawsze czytaj `__cause__`.** Nasze `raise IsbnAlreadyExists(...) from exc` zachowuje oryginalny `IntegrityError` jako przyczynę. W tracebacku zobaczysz „The above exception was the direct cause of the following exception” — to twoja wskazówka, że błąd techniczny został przetłumaczony celowo, a nie zgubiony.
- **Rozróżnij „błąd techniczny” od „błędu domenowego”.** Pierwszy naprawiasz w repozytorium albo w bazie, drugi w serwisie. Jeśli widzisz `IntegrityError` w warstwie API — masz przeciek, nie bug.
- **`DetachedInstanceError` to najczęstszy wyjątek w projektach z repozytorium.** Pojawia się prawie zawsze wtedy, gdy repozytorium zwróciło encję, a warstwa wyżej odczytała relację, której nie załadowano. Rozwiązanie jest projektowe, nie „łatka”: albo zwracasz DTO, albo eager-loadujesz przy zapytaniu, albo utrzymujesz sesję przy życiu przez request.
- **`MissingGreenlet` w async to ten sam problem ubrany inaczej.** Jeśli repozytorium jest async i zwraca encję z niezaładowaną relacją, leniwe odczytanie relacji w `async def` wywoła `MissingGreenlet` ([Moduł 15](15_asynchronicznosc.md)). Wniosek ten sam: eager-load w repozytorium albo DTO.

> ⚠️ **Pułapka:** Zbyt szerokie `except IntegrityError: raise IsbnAlreadyExists(...)` łapie **każde** naruszenie integralności — także błąd klucza obcego albo `NOT NULL`. Użytkownik dostanie komunikat o ISBN, choć problem jest zupełnie inny. Rozpoznawaj konkretne ograniczenie (po nazwie, nie po całym komunikacie), a nieobsłużone przypadki przepuszczaj dalej przez `raise`.

---

## Słowniczek modułu

| Termin (EN) | Polski odpowiednik | Wyjaśnienie |
|---|---|---|
| Repository | Repozytorium | Warstwa udostępniająca dane w języku domeny; wygląda jak kolekcja obiektów, a pod spodem jest baza danych |
| Protocol | Protokół | Typ strukturalny z `typing.Protocol`: definiuje zestaw metod bez implementacji; obiekt spełnia go, jeśli ma odpowiednie metody, bez dziedziczenia |
| Structural typing | Typowanie strukturalne | Zgodność typów sprawdzana po kształcie obiektu (metody, atrybuty), a nie po deklarowanym dziedziczeniu |
| Fake repository | Repozytorium zaślepka | Implementacja w pamięci używana w testach jednostkowych zamiast prawdziwej bazy |
| DTO (Data Transfer Object) | Obiekt transferu danych | Lekka struktura danych przenosząca dane między warstwami; bez relacji i cyklu życia encji |
| Projection | Projekcja | Zapytanie pobierające tylko wybrane kolumny, zwykle prosto do DTO — zamiast pełnej encji |
| Query Builder | Budowniczy zapytań | Obiekt składający zapytanie z klocków (filtry, sortowania) w czytelny łańcuch wywołań |
| Specification | Specyfikacja | Obiekt opisujący warunek filtrujący; komponowalny (`And`, `Or`) i testowalny osobno |
| Pagination | Paginacja | Dzielenie zbioru wyników na strony |
| Offset pagination | Paginacja offsetowa | Stronicowanie przez `LIMIT` + `OFFSET`; proste, ale koszt rośnie z numerem strony |
| Keyset pagination | Paginacja kluczowa (kursorowa) | Stronicowanie „po ostatnim kluczu”: `WHERE id > :last ORDER BY id LIMIT n`; stały koszt |
| Cursor | Kursor | Wartość oznaczająca pozycję w wynikach (np. ostatnie `id`); pozwala wrócić do miejsca bez liczenia przesunięcia |
| Cursor stability | Stabilność kursora | Gwarancja, że sortowanie jest deterministyczne (klucz unikalny); brak tego powoduje gubienie lub dublowanie wierszy |
| `Page[T]` | Strona wyników | Typ generyczny: elementy + liczba stron + rozmiar strony + numer strony |
| `total` | Liczba wszystkich pozycji | Wynik `COUNT(*)` — potrzebny dla klasycznej paginacji, pomijalny przy keyset |
| Abstraction leak | Przeciek abstrakcji | Sytuacja, w której szczegóły implementacji (np. `Select`, `Session`) są widoczne dla warstwy wyżej |
| Facade wrapper | Fasada-opakowanie | Klasa, która nie ukrywa niczego, tylko przekazuje wywołania dalej; częsty antywzorzec repozytorium |
| Unit of Work | Jednostka pracy | Warstwa zarządzająca transakcją i zestawem zmian; w SQLAlchemy rolę tę pełni `Session`. Osobny wzorzec omawiamy w [Module 21](21_unit_of_work.md) |
| Identity map | Mapa tożsamości | Rejestr sesji gwarantujący, że dany wiersz ma jedną instancję Pythona w obrębie sesji |
| Eager loading | Ładowanie zachłanne | Wczytanie relacji od razu, w tym samym lub dodatkowym z góry zaplanowanym zapytaniu (`joinedload`, `selectinload`) |
| Lazy loading | Ładowanie leniwe | Wczytanie relacji dopiero przy pierwszym dostępie; wygodne, ale źródło problemu N+1 |
| N+1 problem | Problem N+1 | Wykonanie 1 zapytania po zbiór i N zapytań po relacje każdego elementu; typowa przyczyna spowolnienia |
| `raiseload` | Wymuszenie jawnego ładowania | Opcja ładowania, która rzuca wyjątek przy próbie leniwego odczytu — zabezpieczenie przed regresją wydajności |
| `selectinload` | Ładowanie przez `IN` | Strategia: jedno dodatkowe zapytanie `WHERE id IN (...)` dla całej kolekcji |
| `joinedload` | Ładowanie przez JOIN | Strategia: dołączenie relacji do głównego zapytania przez LEFT OUTER JOIN |
| Domain exception | Wyjątek domenowy | Wyjątek wyrażający regułę biznesową (np. `IsbnAlreadyExists`), niezależny od technologii |
| `IntegrityError` | Błąd integralności | Wyjątek SQLAlchemy przy naruszeniu ograniczeń bazy (unikalność, klucz obcy, `NOT NULL`) |
| Transaction boundary | Granica transakcji | Miejsce w kodzie, w którym rozpoczyna się i kończy transakcja; należy do warstwy aplikacji, nie repozytorium |
| `flush` | Wypchnięcie zmian | Wysłanie przygotowanych zmian do bazy w obrębie bieżącej transakcji, bez zatwierdzenia |
| `commit` | Zatwierdzenie | Trwałe zapisanie transakcji w bazie; kończy bieżącą transakcję |
| `rollback` | Wycofanie | Cofnięcie bieżącej transakcji; po błędzie przywraca sesję do stanu używalności |
| Dependency Injection | Wstrzykiwanie zależności | Przekazywanie zależności (np. repozytorium, sesji) przez konstruktor zamiast tworzenia ich wewnątrz klasy |
| Contract test | Test kontraktu | Zestaw testów wspólny dla wielu implementacji tego samego interfejsu |
| Fake in memory | Implementacja w pamięci | Repozytorium działające na słownikach/listach, używane w testach i prototypach |
| Detached instance | Instancja odłączona | Obiekt encji, który nie jest już powiązany z żadną sesją; nie wolno mu już leniwie doładowywać relacji |

---

## Dalsze czytanie

**Oficjalna dokumentacja SQLAlchemy (2.0 i 2.1):**

- [ORM Quick Start](https://docs.sqlalchemy.org/en/20/orm/quickstart.html) — najszybsze wprowadzenie do modeli, `Session` i `select()`
- [Session Basics](https://docs.sqlalchemy.org/en/20/orm/session_basics.html) — cykl życia sesji, `flush`, `commit`, `rollback`, autobegin
- [ORM Querying Guide](https://docs.sqlalchemy.org/en/20/orm/queryguide/index.html) — kompletny przewodnik po `select()` w ORM, w tym `mapping()` i projekcje
- [Relationship Loading Techniques](https://docs.sqlalchemy.org/en/20/orm/queryguide/relationships.html) — `selectinload`, `joinedload`, `raiseload`, strategie ładowania
- [Relationship Configuration](https://docs.sqlalchemy.org/en/20/orm/relationships.html) — kardynalności, kaskady, `back_populates`
- [SQLAlchemy 2.0 — What's New](https://docs.sqlalchemy.org/en/20/changelog/whatsnew_20.html) — tło transformacji API z 1.4 na 2.0
- [Migrating to SQLAlchemy 2.0](https://docs.sqlalchemy.org/en/20/changelog/migration_20.html) — tabela „stare → nowe” (przydatna, gdy czytasz starszy kod)
- [What's New in SQLAlchemy 2.1](https://docs.sqlalchemy.org/en/21/changelog/whatsnew_21.html) — nowości omawiane w ramkach 🆕 w tym kursie
- [SQLAlchemy — licznik zdarzeń i `before_cursor_execute`](https://docs.sqlalchemy.org/en/20/core/events.html) — jak zbudować własny licznik zapytań do testów

**Wzorce architektoniczne:**

- Martin Fowler, [Repository](https://martinfowler.com/eaaCatalog/repository.html) — definicja oryginalna, tło historyczne
- Martin Fowler & Eric Evans, [Specification](https://martinfowler.com/apsupp/spec.pdf) — artykuł o specyfikacjach, źródło wzorca z sekcji 8
- [Cosmic Python (Architecture Patterns with Python)](https://www.cosmicpython.com/book/preface.html) — bezpłatna książka; repozytorium, Unit of Work i „dependency inversion” w Pythonie na przykładach bliskich SQLAlchemy
- [Patterns of Enterprise Application Architecture](https://martinfowler.com/books/eaa.html) — źródło wzorców Data Mapper, Repository, Unit of Work, Query Object

**Testowanie i narzędzia:**

- [pytest — dokumentacja](https://docs.pytest.org/en/stable/) — fixtures, `conftest.py`, parametryzacja, markery
- [Protocol i strukturalne typowanie — dokumentacja Pythona](https://docs.python.org/3/library/typing.html#typing.Protocol) — oficjalne wyjaśnienie `Protocol` i `@runtime_checkable`
- [Alembic — dokumentacja](https://alembic.sqlalchemy.org/en/latest/) — migracje, prawdziwe utrzymanie schematu (moduł 16)
- [Pydantic v2 — dokumentacja](https://docs.pydantic.dev/latest/) — schematy wejścia/wyjścia dla API, konwersja z encji (`from_attributes`)

---

## Co dalej

Masz teraz pełny obraz tego, jak oddzielić intencje biznesowe od techniki trwałości: protokół, dwie implementacje (SQL i pamięć), projekcje DTO, paginację keysetową i tłumaczenie wyjątków. Wiesz też, że repozytorium bywa przerostem — i potrafisz uzasadnić, kiedy je wprowadzasz. Zostało jedno pytanie: **kto commituje?** W [Module 21 — `21_unit_of_work.md`](21_unit_of_work.md) zbudujesz Unit of Work, który zepnie wiele repozytoriów w jedną transakcję, zdefiniuje granice przypadków użycia i pokaże, jak poprawnie zamykać oraz wycofywać zmiany w aplikacji webowej, CLI i workerze.

<!-- koniec modułu 20 -->