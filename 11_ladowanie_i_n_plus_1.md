# Moduł 11 — Ładowanie relacji i problem N+1

W tym module dowiesz się, dlaczego aplikacja korzystająca z ORM potrafi nagle działać dziesięć, sto, a czasem tysiąc razy wolniej, mimo że kod wygląda zupełnie niewinnie. Poznasz mechanizm, który stoi za większością takich przypadków — problem **N+1** — nauczysz się go wykrywać za pomocą licznika zapytań i logów SQL, a następnie wyeliminujesz go, świadomie dobierając **strategię ładowania relacji** (`selectinload`, `joinedload`, `raise`, `write_only` i inne). Na końcu zbudujesz własny benchmark, który liczbami — nie opiniami — pokaże, ile kosztuje każdy wariant.

---

**Poziom:** 🟠 zaawansowany
**Czas:** ~180 minut
**Wymagania wstępne:** [Moduł 07 — Modele deklaratywne](07_modele_deklaratywne.md), [Moduł 08 — Sesja i cykl życia obiektu](08_sesja_cykl_zycia.md), [Moduł 09 — Relacje](09_relacje.md), [Moduł 10 — Zapytania ORM](10_zapytania_orm.md)
**Czego dotyczy plik:** strategie ładowania relacji w SQLAlchemy 2.0+, diagnozowanie i naprawianie problemu N+1, opcje `Load` API, `DetachedInstanceError`, dobór strategii do przypadku użycia.

---

## Spis treści

- [Dlaczego ORM nagle zwalnia](#dlaczego-orm-nagle-zwalnia)
- [Problem N+1](#problem-n)
- [Jak wykryć N+1](#jak-wykryć-n1)
- [Strategie ładowania ustawiane na relacji](#strategie-ładowania-ustawiane-na-relacji)
- [Opcje ładowania na poziomie zapytania](#opcje-ładowania-na-poziomie-zapytania)
- [Zagnieżdżone ładowanie i Load API](#zagnieżdżone-ładowanie-i-load-api)
- [contains_eager — wykorzystanie własnego JOIN-a](#contains_eager--wykorzystanie-własnego-join-a)
- [Ładowanie poza sesją i DetachedInstanceError](#ładowanie-poza-sesją-i-detachedinstanceerror)
- [Jak wybierać strategię — tabela decyzyjna](#jak-wybierać-strategię--tabela-decyzyjna)
- [Benchmark: trzy warianty tego samego raportu](#benchmark-trzy-warianty-tego-samego-raportu)
- [Podsumowanie](#podsumowanie)
- [Ćwiczenia](#ćwiczenia)
- [Najczęstsze błędy i jak je czytać](#najczęstsze-błędy-i-jak-je-czytać)
- [Słowniczek modułu](#słowniczek-modułu)
- [Dalsze czytanie](#dalsze-czytanie)
- [Co dalej](#co-dalej)

---

## Dlaczego ORM nagle zwalnia

Do tej pory praca z ORM wyglądała wygodnie: piszesz `select(Book)`, dostajesz obiekty Pythona, a atrybuty takie jak `book.author` czy `book.reviews` po prostu działają. Ta wygoda jest jednak **opóźniona** — dopiero w momencie, w którym sięgasz po `book.author`, SQLAlchemy musi coś zrobić. Musi iść do bazy i pobrać autora.

To zachowanie nazywamy **leniwym ładowaniem** (*lazy loading*): dane relacji są wczytywane dopiero w chwili dostępu do atrybutu. Ma ono jedną wielką zaletę — nie pobierasz z bazy niczego, czego nie użyjesz. Ma też jedną wielką wadę: **moment wykonania zapytania przestaje być widoczny w kodzie**. Patrząc na linię `for book in books:`, nie widzisz, że każde jej przejście może wywołać zapytanie do bazy.

> 💡 **Analogia — lodówka i zakupy**
> Leniwę ładowanie to codzienne chodzenie do sklepu po każdy brakujący składnik, dokładnie w momencie, gdy jest potrzebny. Nigdy nie kupisz nic zbędnego — ale jeśli gotujesz obiad z dwudziestu składników, pójdziesz do sklepu dwadzieścia razy. Zamiast tego mógłbyś raz na początku przygotować pełną listę i zrobić jedno duże zakupy. Właśnie to robi *eager loading* (ładowanie zachłanne): bierze wszystko z góry, kosztem tego, że część rzeczy może się nie przydać.

Ten moduł jest w całości o **przesuwaniu momentu i liczby zapytań** — czyli o tym, kiedy i jakim sposobem dociągamy dane z bazy.

> 🧠 **Dlaczego tak jest**
> Leniwe ładowanie jest **domyślne**, ponieważ dla pojedynczego obiektu (np. „pobierz jednego użytkownika i pokaż jego imię”) zachłanne pobieranie wszystkiego byłoby marnotrawstwem. Problem pojawia się, gdy przetwarzamy **zbiór** obiektów w pętli — wtedy model „jedno zapytanie na każdy dostęp” skaluje się liniowo i zabija wydajność. Dobór strategii ładowania to więc zawsze kompromis między liczbą zapytań a ilością przesyłanych danych.

---

## Problem N+1

### Skąd się bierze

Rozważmy typowy raport: lista książek wraz z nazwą autora.

```python
# examples/11_n_plus_1.py
from sqlalchemy import create_engine, select
from sqlalchemy.orm import Session

from examples.models import Author, Book


def slow_report(engine) -> None:
    with Session(engine) as session:
        # 1. Jedno zapytanie: pobieramy wszystkie książki.
        books = session.scalars(select(Book).order_by(Book.id)).all()

        # 2. N zapytań: dla każdej książki dociągamy autora osobno.
        for book in books:
            print(book.title, "—", book.author.name)


if __name__ == "__main__":
    engine = create_engine("sqlite:///:memory:", echo=False)
    # ... (tu: utworzenie schematu i wstawienie danych — patrz example poniżej)
    slow_report(engine)
```

Wygląda niewinnie. W rzeczywistości wykonują się tu zapytania:

1. jedno, które pobiera listę książek, oraz
2. $N$ zapytań — po jednym na **każdą książkę**, żeby pobrać autora.

Razem $1 + N$ zapytań. Stąd nazwa wzorca: **N+1**.

> 💡 **Analogia — pójście po zakupy**
> Masz listę 50 produktów. Wersja „N+1” to 50 osobnych wyjść do sklepu — jedno po każdy produkt. Personel sklepu za każdym razem musi Cię wpuścić, obsłużyć przy kasie i pożegnać. Nawet jeśli sam produkt jest mały, **koszt stały obsługi** mnoży się przez 50. Wersja „eager” to jedno wyjście z pełną listą: mniej obsługi, więcej produktów naraz, jeden rachunek.

### Realny log SQL

Uruchomienie powyższego kodu z `echo=True` (albo z ustawionym poziomem `INFO` dla loggera `sqlalchemy.engine`) da na wyjściu mniej więcej taki obraz — dla 100 książek od 100 różnych autorów:

```text
-- zapytanie 1 z 101
SELECT book.id, book.title, book.year, book.author_id
FROM book ORDER BY book.id

-- ponizsze powtarza sie 100 razy (dla kazdego unikalnego autora)
SELECT author.id, author.name, author.birth_year
FROM author
WHERE author.id = ?
```

Zliczmy: **101 zapytań na jeden raport**. Jeżeli każde trwa 1 ms (optymistycznie — w SQLite w pamięci), to 101 ms. Jeżeli każde jest podróżą przez sieć do PostgreSQL na drugim końcu klastra i trwa 3 ms, mamy 300 ms. A jeśli w aplikacji jest dwadzieścia takich pętli (serializacja listy, licznik, eksport, walidacja…), robi się z tego sekundy.

> ⚠️ **Pułapka — N to liczba *unikalnych* encji, nie wierszy**
> SQLAlchemy utrzymuje w sesji **mapę tożsamości** (*identity map*, „rejestr obiektów już wczytanych”, patrz [Moduł 08](08_sesja_cykl_zycia.md)). Jeśli dwie książki mają tego samego autora, drugi dostęp do `.author` **nie wykona zapytania** — obiekt jest już w mapie. Dlatego w praktyce przy 100 książkach i 5 autorach zobaczysz $1 + 5 = 6$ zapytań, nie 101. To ważne, bo utrudnia diagnozę: *na małych danych SQL-a może być zaskakująco mało*, a po wdrożeniu — katastrofalnie dużo.

> 🔬 **Pod maską — jak wygląda „N+1” w logu**
> Każde dociągnięcie relacji to osobna, „goła” instrukcja `SELECT` z jednym parametrem `WHERE author.id = ?`. W logu recognize'ujesz to po powtarzających się, identycznych zapytaniach różniących się tylko wartością parametru. To najczęstszy sygnał N+1: **ten sam `SELECT` setki razy pod rząd**.

### Złożoność

Zapiszmy to formalnie, bo to pomaga w rozmowie o wydajności:

- leniwe ładowanie w pętli: $O(1 + N)$ zapytań, gdzie $N$ to liczba unikalnych encji, do których sięgamy,
- ładowanie zachłanne: $O(1)$ zapytań dla relacji „do jednego” (wiele-do-jednego, jeden-do-jednego),
- `selectinload` dla kolekcji: $O(2)$ zapytań niezależnie od $N$,
- ładowanie zagnieżdżone na $k$ poziomach: minimum $O(k)$ zapytań.

Dla $N = 1000$ różnica między $O(1)$ a $O(1+N)$ to różnica między jedną podróżą a tysiącem podróży. W świecie baz danych to prawie zawsze różnica między „szybko” i „strona się nie ładuje”.

> 🧠 **Dlaczego to nie jest problem wydajności bazy, a problem liczby rund**
> Baza danych potrafi w jednej chwili oddać dziesiątki tysięcy wierszy. Wąskim gardłem jest **liczba okrążeń (round-trip) między aplikacją a bazą**: narzut I/O, parsowanie, planowanie, sieć. Dlatego strategia, która zamienia 1000 zapytań na 2, wygrywa nawet jeśli każde z tych 2 pobiera dużo więcej danych.

---

## Jak wykryć N+1

Zanim cokolwiek naprawisz, musisz zobaczyć problem. Są trzy narzędzia, których będziesz używać w tej kolejności: logi, licznik zapytań i licznik w testach.

### Logi SQLAlchemy

Najprostszy sposób to `echo=True` przy tworzeniu silnika (poziom wstępny, w praktyce za głośny) albo włączenie loggera:

```python
# examples/11_logging.py
import logging

from sqlalchemy import create_engine

# INFO pokazuje zapytania bez parametrów; DEBUG dokłada parametry i czasy.
logging.getLogger("sqlalchemy.engine").setLevel(logging.INFO)

engine = create_engine("sqlite:///biblioteka.db")
```

Poziomy mają znaczenie:

| Ustawienie | Co pokazuje | Kiedy używać |
|---|---|---|
| `echo="debug"` | zapytania, parametry, czasy wykonania, pula połączeń | jednorazowe debugowanie |
| `logging.INFO` | samo SQL (z placeholderami) | audyt liczby zapytań |
| `logging.DEBUG` | SQL + parametry + czas | diagnoza konkretnego zapytania |

> ⚠️ **Pułapka — `echo=True` zostawione w kodzie produkcyjnym**
> Każde zapytanie trafiające na standardowe wyjście to koszt formatowania i I/O, a w logach mogą znaleźć się **dane wrażliwe** (parametry z hasłami, e-mailami). Ustawiaj `echo` tylko przez konfigurację środowiskową, nigdy na sztywno w kodzie produkcyjnym.

### Licznik zapytań

Logi są dobre do „jednorazowego rzutu okiem”. Do systematycznego mierzenia — a zwłaszcza do testów — potrzebujesz **licznika zapytań**. SQLAlchemy udostępnia zdarzenia, do których możemy się podpiąć.

Zdarzenie `before_cursor_execute` jest wywoływane dokładnie raz na każdą instrukcję wysyłaną do bazy. Podpinając się do niego, liczymy zapytania:

```python
# examples/11_query_counter.py
from __future__ import annotations

import time

from sqlalchemy import event
from sqlalchemy.engine import Engine


class QueryCounter:
    """Liczy zapytania wykonane na danym silniku w obrebie bloku `with`."""

    def __init__(self, engine: Engine) -> None:
        self._engine = engine
        self._statements: list[str] = []
        self.elapsed = 0.0
        self._t0 = 0.0

    def _record(self, conn, cursor, statement, parameters, context, executemany) -> None:
        # `statement` to surowy SQL wyslany do drivera.
        self._statements.append(statement)

    def __enter__(self) -> QueryCounter:
        self._statements.clear()
        self._t0 = time.perf_counter()
        event.listen(self._engine, "before_cursor_execute", self._record)
        return self

    def __exit__(self, exc_type, exc, tb) -> None:
        # Zawsze odpinamy listener — inaczej przy kolejnym uzyciu policzymy podwojnie.
        event.remove(self._engine, "before_cursor_execute", self._record)
        self.elapsed = time.perf_counter() - self._t0

    @property
    def count(self) -> int:
        return len(self._statements)

    def statements(self) -> list[str]:
        return list(self._statements)

    def summary(self, label: str) -> str:
        return f"{label:<24} zapytan: {self.count:>5}  czas: {self.elapsed * 1000:8.1f} ms"
```

Użycie jest proste i — co ważne — mierzalne:

```python
# examples/11_measure.py
from sqlalchemy import create_engine, select
from sqlalchemy.orm import Session, joinedload

from examples.models import Book
from examples.query_counter import QueryCounter

engine = create_engine("sqlite:///biblioteka.db")

with Session(engine) as session:
    with QueryCounter(engine) as qc:
        books = session.scalars(select(Book).order_by(Book.id)).all()
        names = [f"{b.title} / {b.author.name}" for b in books]
    print(qc.summary("lazy"))

    session.expunge_all()  # czyscimy identity map, zeby pomiar byl uczciwy

    with QueryCounter(engine) as qc:
        stmt = select(Book).options(joinedload(Book.author)).order_by(Book.id)
        books = session.scalars(stmt).all()
        names = [f"{b.title} / {b.author.name}" for b in books]
    print(qc.summary("joinedload"))
```

> 🧠 **Dlaczego `expunge_all()` przed drugim pomiarem**
> Mapę tożsamości trzeba wyczyścić, bo obiekty już wczytane nie wygenerują ponownych zapytań i drugi pomiar byłby fałszywie korzystny. Mierzenie wydajności bez izolowania pomiarów to najczęstszy sposób oszukania samego siebie.

> 🔬 **Pod maską — dlaczego właśnie `before_cursor_execute`**
> SQLAlchemy kompiluje zapytanie ORM do SQL i przekazuje je do drivera DBAPI. Zdarzenie `before_cursor_execute` leci **bezpośrednio przed** wywołaniem kursora, więc łapie dokładnie to, co realnie idzie do bazy — również zapytania wygenerowane „w tle” przez leniwe ładowanie. Nie musisz nic zmieniać w modelach ani zapytaniach.

### Licznik zapytań w testach

Licznik w testach to najskuteczniejsza obrona przed **regresją wydajnościową**: ktoś dodaje nowy `selectinload`, ale po drodze gubi istniejący eager load w innym miejscu — test to wychwyci, zanim trafi na produkcję.

```python
# tests/conftest.py
from __future__ import annotations

from collections.abc import Iterator

import pytest
from sqlalchemy import create_engine, event
from sqlalchemy.engine import Engine
from sqlalchemy.orm import Session, sessionmaker

from examples.models import Base


@pytest.fixture(scope="session")
def engine() -> Iterator[Engine]:
    # StaticPool: jedno polaczenie wspoldzielone przez wszystkie sesje,
    # dzieki czemu sqlite:///:memory: nie tworzy pustej bazy przy kazdym connect.
    from sqlalchemy.pool import StaticPool

    eng = create_engine(
        "sqlite://",
        connect_args={"check_same_thread": False},
        poolclass=StaticPool,
    )
    Base.metadata.create_all(eng)
    yield eng
    eng.dispose()


@pytest.fixture
def session(engine: Engine) -> Iterator[Session]:
    with Session(engine) as sess:
        yield sess


@pytest.fixture
def assert_num_queries(engine: Engine):
    """Kontekst-menedzer asertujacy, ile zapytan poszlo do bazy."""

    class _Asserter:
        def __init__(self) -> None:
            self.statements: list[str] = []
            self.expected: int | None = None

        def _record(self, conn, cursor, statement, parameters, context, executemany):
            self.statements.append(statement)

        def __enter__(self) -> _Asserter:
            event.listen(engine, "before_cursor_execute", self._record)
            return self

        def __exit__(self, exc_type, exc, tb) -> None:
            event.remove(engine, "before_cursor_execute", self._record)
            if self.expected is not None and exc_type is None:
                assert len(self.statements) == self.expected, (
                    f"oczekiwano {self.expected} zapytan, bylo {len(self.statements)}:\n"
                    + "\n".join(self.statements)
                )

    return _Asserter
```

W teście wygląda to tak:

```python
# tests/test_report.py
from sqlalchemy import select
from sqlalchemy.orm import joinedload

from examples.models import Book


def test_report_dla_10_ksiazek_uzywa_dwoch_zapytan(session, assert_num_queries):
    stmt = select(Book).options(joinedload(Book.author)).order_by(Book.id)

    with assert_num_queries() as counter:
        counter.expected = 1
        books = session.scalars(stmt).all()
        _ = [b.author.name for b in books]
```

> 🧪 **Ćwiczenie — policz zapytania w swoim kodzie**
> Dodaj `assert_num_queries` do `conftest.py` i opakuj nim **jedną** funkcję z modułu 10, która zwraca listę encji z relacjami. Ile zapytań faktycznie poszło? Zapisz wynik, bo wrócimy do niego przy optymalizacji.

---

## Strategie ładowania ustawiane na relacji

Istnieją dwa miejsca, w których decydujesz o sposobie ładowania:

1. **na poziomie relacji** — `relationship(..., lazy="...")`, czyli domyślne zachowanie dla **całej aplikacji**;
2. **na poziomie zapytania** — `select(...).options(...)`, czyli jednorazowe nadpisanie dla **konkretnego zapytania**.

Zacznijmy od pierwszego. Parametr `lazy=` przyjmuje następujące wartości.

### `lazy="select"`

Wartość domyślna. Przy dostępie do atrybutu SQLAlchemy wykonuje pojedynczy `SELECT` dla **jednego** obiektu (kolumna, nie zbiór).

```python
# examples/11_lazy_select.py
from __future__ import annotations

from sqlalchemy import ForeignKey, String
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    pass


class Author(Base):
    __tablename__ = "author"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(120))

    # DOMYSLNE zachowanie: biblioteka ksiazek dociaga sie osobno.
    books: Mapped[list[Book]] = relationship(
        back_populates="author",
        lazy="select",
    )


class Book(Base):
    __tablename__ = "book"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    author_id: Mapped[int] = mapped_column(ForeignKey("author.id"))

    author: Mapped[Author] = relationship(back_populates="books")
```

> 🧠 **Dlaczego to domyślne**
> `lazy="select"` jest najbezpieczniejsze „semantycznie”: nie pobiera niczego niepotrzebnego i nie zmienia kształtu zapytań, które piszesz ręcznie. Jest jednak **śmiertelnie niebezpieczne w pętli**. Dobre dla: pojedynczych obiektów, skryptów CLI, pracy edycyjnej, gdzie i tak wczytujesz obiekt po obiekcie.

### `lazy="selectin"`

Wczytuje kolekcję **zbiorczo**: po pobraniu obiektów nadrzędnych SQLAlchemy wysyła jedno dodatkowe zapytanie z warunkiem `IN (...)` na wszystkie klucze obce naraz. To dwie rundy do bazy — niezależnie od $N$.

```python
# examples/11_lazy_selectin.py
class Author(Base):
    __tablename__ = "author"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(120))

    # Kolekcja zawsze dociagana jednym dodatkowym zapytaniem IN (...).
    books: Mapped[list[Book]] = relationship(back_populates="author", lazy="selectin")
```

> 🔬 **Pod maską — `lazy="selectin"` na kolekcji**
> ```sql
> -- zapytanie 1
> SELECT author.id, author.name FROM author
>
> -- zapytanie 2
> SELECT book.author_id AS book_author_id, book.id, book.title
> FROM book
> WHERE book.author_id IN (?, ?, ?, ...)
> ORDER BY book.author_id
> ```
> Liczba zapytań: **2**. Rozmiar `IN` rośnie z liczbą obiektów nadrzędnych — to ważne, wrócimy do tego przy `chunksize`.

To jest **zalecana domyślna strategia dla kolekcji** w większości aplikacji. Jest odporna (nie ma problemu kartezjańskiego), daje stałą liczbę zapytań i dobrze się skaluje.

### `lazy="joined"`

Wczytuje relację **jednym zapytaniem z JOIN-em** już przy pobraniu obiektu nadrzędnego.

```python
# examples/11_lazy_joined.py
class Book(Base):
    __tablename__ = "book"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    author_id: Mapped[int] = mapped_column(ForeignKey("author.id"))

    # Relacja wiele-do-jednego: doskonala kandydatka na joined.
    author: Mapped[Author] = relationship(back_populates="books", lazy="joined")
```

> 💡 **Analogia — JOIN jako „setka listy”**
> JOIN to jakby wysłać do hurtowni jedno zamówienie z od razu zszytymi fakturami: dostajesz wszystko naraz, ale faktura autora powtarza się przy każdej jego książce. Dla relacji „do jednego” to świetne (autor jest jeden, więc nie ma powtórzeń). Dla kolekcji — niebezpieczne, bo wiersze zaczynają się mnożyć.

> ⚠️ **Pułapka — `joinedload` na kolekcji bez `unique()`**
> Przy ładowaniu kolekcji JOIN-em SQLAlchemy zwraca wiersz na każdą parę (obiekt, element kolekcji). Gdy odbierzesz wynik bez `.unique()`, dostaniesz `InvalidRequestError`. Musisz to zrobić świadomie — i zrozumieć, że duplikaty wierszy **naprawdę** zostały wygenerowane w zapytaniu.

### `lazy="subquery"`

Historyczna alternatywa dla `selectin`: opakowuje pierwotne zapytanie w podzapytanie i dołącza je JOIN-em.

```python
# examples/11_lazy_subquery.py
class Book(Base):
    # ... (pola jak wyzej)

    # Dziala, ale w nowych projektach preferujemy selectin.
    reviews: Mapped[list[Review]] = relationship(back_populates="book", lazy="subquery")
```

> 🧠 **Dlaczego `subquery` jest dziś rzadziej używane**
> `selectin` jest zwykle szybsze (prosty `IN (...)`, brak zagnieżdżonego podzapytania), lepiej współpracuje z planerem zapytań i nie wymaga `unique()`. Traktuj `subquery` jako opcję dla starszych baz lub specyficznych przypadków, gdy planer wyraźnie preferuje JOIN.

### `lazy="immediate"`, `lazy="noload"`, `lazy="raise"`

```python
# examples/11_lazy_variants.py
from sqlalchemy import ForeignKey, String
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    pass


class Book(Base):
    __tablename__ = "book"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    author_id: Mapped[int] = mapped_column(ForeignKey("author.id"))

    # `immediate`: dociaga od razu po glownym zapytaniu, ale osobna
    # instrukcja SELECT na kazdy obiekt (styl leniwy, tylko natychmiastowy).
    author: Mapped[Author] = relationship(back_populates="books", lazy="immediate")

    # `noload`: atrybut zostanie pusty/None. Uwaga: nie znaczy "nie ma danych",
    # znaczy "nie sprawdzalismy". Uzywaj swiadomie i rzadko.
    reviews: Mapped[list[Review]] = relationship(back_populates="book", lazy="noload")

    # `raise`: kazdy dostep do nie-wczytanego atrybutu konczy sie wyjatkiem.
    # Swietne w testach i w API, gdzie nie chcemy niejawnych zapytan.
    loans: Mapped[list[Loan]] = relationship(back_populates="book", lazy="raise")
```

`lazy="raise"` jest niedocenianym narzędziem inżynierskim: zamienia cichy, kosztowny dostęp do relacji w **głośny błąd**. Dzięki temu każdy zapomniany eager load wychodzi w testach, a nie u klienta.

### `lazy="write_only"`

Kolekcja, którą można tylko **dopisywać i nadpisywać**, ale nie można jej używac do iteracji, `len()` ani indeksowania — dopóki nie wywołasz jawnie opcji ładowania.

```python
# examples/11_lazy_write_only.py
from sqlalchemy import ForeignKey, String
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    pass


class Book(Base):
    __tablename__ = "book"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    author_id: Mapped[int] = mapped_column(ForeignKey("author.id"))

    # Kolekcja moze miec milion wpisow. Nie chcemy, by ktokolwiek
    # przypadkiem wczytal ja w calosci do pamieci.
    loans: Mapped[list[Loan]] = relationship(
        back_populates="book",
        lazy="write_only",
    )
```

> 💡 **Analogia — dziennik zdarzeń na taśmie**
> `write_only` to taśma, na którą możesz dopisywać zdarzenia i którą możesz wyzerować (nadpisać całą kolekcję), ale nie możesz „przewijać” w celu czytania — musisz zażądać wydruku. To doskonałe dla logów, zdarzeń, wielkich historii: aplikacja zapisuje, ale nigdy nie wczytuje ich hurtem.

> 🆕 **SQLAlchemy 2.1 — `selectinload` z `omit_join` dla wiele-do-wiele**
> Przy ładowaniu relacji wiele-do-wiele `selectinload` w 2.1 potrafi pominąć pomocnicze `JOIN` do tabeli asocjacyjnej i pobrać potrzebne klucze w sposób prostszy, co redukuje liczbę operacji łączenia. To optymalizacja „pod maską” — nie musisz nic zmieniać w kodzie, żeby z niej skorzystać, ale warto wiedzieć, że kształt SQL dla N:M bywa w 2.1 inny niż w 2.0.

---

## Opcje ładowania na poziomie zapytania

Ustawianie `lazy=` na modelu to decyzja **globalna**. Często jednak potrzebujesz innego zachowania tylko w jednym zapytaniu — właśnie do tego służą **opcje ładowania** przekazywane przez `options()`.

```python
# examples/11_options_stub.py
from sqlalchemy import select
from sqlalchemy.orm import selectinload
from sqlalchemy.orm import Session

stmt = select(Book).options(selectinload(Book.reviews)).where(Book.year >= 2000)
```

### `selectinload()`

Zbiera obiekty nadrzędne, a potem dociąga relację jednym dodatkowym `IN (...)`.

```python
# examples/11_selectinload.py
# examples/models.py — wspolny zestaw modeli uzywany w calym module
from __future__ import annotations

from datetime import datetime

from sqlalchemy import ForeignKey, String, Text
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    pass


class Author(Base):
    __tablename__ = "author"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(120))
    birth_year: Mapped[int | None]

    books: Mapped[list[Book]] = relationship(back_populates="author")


class Book(Base):
    __tablename__ = "book"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200), index=True)
    year: Mapped[int]
    author_id: Mapped[int] = mapped_column(ForeignKey("author.id"))

    author: Mapped[Author] = relationship(back_populates="books")
    reviews: Mapped[list[Review]] = relationship(back_populates="book")
    loans: Mapped[list[Loan]] = relationship(back_populates="book")


class Review(Base):
    __tablename__ = "review"

    id: Mapped[int] = mapped_column(primary_key=True)
    book_id: Mapped[int] = mapped_column(ForeignKey("book.id"))
    rating: Mapped[int]
    body: Mapped[str | None] = mapped_column(Text)

    book: Mapped[Book] = relationship(back_populates="reviews")


class Loan(Base):
    __tablename__ = "loan"

    id: Mapped[int] = mapped_column(primary_key=True)
    book_id: Mapped[int] = mapped_column(ForeignKey("book.id"))
    returned_at: Mapped[datetime | None]

    book: Mapped[Book] = relationship(back_populates="loans")
```

Teraz samo zapytanie:

```python
# examples/11_use_selectinload.py
from sqlalchemy import select
from sqlalchemy.orm import Session, selectinload

from examples.models import Book


def recent_books_with_reviews(session: Session) -> list[Book]:
    stmt = (
        select(Book)
        .options(selectinload(Book.reviews))
        .where(Book.year >= 2000)
        .order_by(Book.title)
    )
    return list(session.scalars(stmt))
```

> 🔬 **Pod maską — `selectinload(Book.reviews)`**
> ```sql
> -- zapytanie 1
> SELECT book.id, book.title, book.year, book.author_id
> FROM book WHERE book.year >= 2000 ORDER BY book.title
>
> -- zapytanie 2
> SELECT review.book_id AS review_book_id, review.id, review.rating, review.body
> FROM review
> WHERE review.book_id IN (?, ?, ?, ...)
> ORDER BY review.book_id
> ```
> Dwa zapytania. W odróżnieniu od `joinedload` **nie duplikuje wierszy** — stąd nie wymaga `unique()`.

### `joinedload()`

Dołącza relację JOIN-em już do zapytania głównego.

```python
# examples/11_use_joinedload.py
from sqlalchemy import select
from sqlalchemy.orm import Session, joinedload

from examples.models import Book


def books_with_authors(session: Session) -> list[Book]:
    stmt = (
        select(Book)
        .options(joinedload(Book.author))  # wiele-do-jednego: bezpieczne
        .order_by(Book.title)
    )
    return list(session.scalars(stmt))
```

> 🔬 **Pod maską — `joinedload(Book.author)`**
> ```sql
> SELECT book.id, book.title, book.year, book.author_id,
>        author_1.id AS id_1, author_1.name, author_1.birth_year
> FROM book
> LEFT OUTER JOIN author AS author_1 ON author_1.id = book.author_id
> ORDER BY book.title
> ```
> Jedno zapytanie. Domyślnie `LEFT OUTER JOIN`. Możesz wymusić `INNER JOIN`, jeśli klucz obcy jest `NOT NULL`:
> ```python
> joinedload(Book.author, innerjoin=True)
> ```

Gdy `joinedload` dotyczy **kolekcji**, a zapytanie ma `LIMIT`/`OFFSET`, SQLAlchemy nie może po prostu dołączyć JOIN-a — zamiast tego opakowuje zapytanie główne w podzapytanie. To automatyczne, ale warto o tym wiedzieć, bo zmienia kształt SQL i utrudnia indeksowanie.

### `subqueryload()` i `immediateload()`

```python
# examples/11_use_subquery_immediate.py
from sqlalchemy import select
from sqlalchemy.orm import Session, immediateload, subqueryload

from examples.models import Book


def variant_subquery(session: Session) -> list[Book]:
    stmt = select(Book).options(subqueryload(Book.reviews))
    return list(session.scalars(stmt))


def variant_immediate(session: Session) -> list[Book]:
    # Jedno dodatkowe zapytanie NA KAZDA ksiazke — ale wykonane od razu,
    # bez czekania na dostep do atrybutu. Do diagnostyki i specyficznych przypadkow.
    stmt = select(Book).options(immediateload(Book.author))
    return list(session.scalars(stmt))
```

> ⚠️ **Pułapka — `immediateload` w praktyce bywa pułapką**
> Wygląda jak „naprawa N+1”, ale to nadal $O(1+N)$ zapytań — tylko wykonanych wcześniej. W kodzie produkcyjnym prawie zawsze chcesz `selectinload` lub `joinedload`; `immediateload` ma sens przy specyficznych migracjach z leniwego modelu.

### `noload()` i `raiseload()`

```python
# examples/11_use_noload_raiseload.py
from sqlalchemy import select
from sqlalchemy.orm import Session, noload, raiseload

from examples.models import Book


def variant_noload(session: Session) -> list[Book]:
    # Reviews zostana puste. UWAGA: cicha nieprawda — zwykle zly pomysl.
    stmt = select(Book).options(noload(Book.reviews))
    return list(session.scalars(stmt))


def variant_raiseload(session: Session) -> list[Book]:
    # Kazdy dostep do NIE-wczytanej relacji rzuci wyjatkiem.
    # Swietne w testach i przy audycie sciezek kodu.
    stmt = select(Book).options(raiseload("*"))
    return list(session.scalars(stmt))
```

### Tabela porównawcza

Poniższe pomiary pochodzą z benchmarku z końca modułu (50 autorów, 500 książek, 5000 recenzji; SQLite w pliku). Czasy są orientacyjne — liczą się **rzędy wielkości i liczba zapytań**, które są deterministyczne.

| Wariant | Liczba zapytań | Kształt SQL | `unique()` | Kiedy używać |
|---|---|---|---|---|
| `lazy="select"` (pętla po 500 książkach) | 1 + 50 = **51** | pojedyncze `SELECT ... WHERE id = ?` | nie | pojedyncze obiekty, CLI |
| `joinedload(Book.author)` (wiele-do-jednego) | **1** | `LEFT OUTER JOIN` | nie | relacje „do jednego” |
| `selectinload(Book.reviews)` (kolekcja) | **2** | `SELECT ... IN (...)` | nie | kolekcje — domyślny wybór |
| `joinedload(Book.reviews)` (kolekcja) | **1** | `LEFT OUTER JOIN` + duplikaty wierszy | **tak** | małe kolekcje, gdy brak alternatywy |
| `subqueryload(Book.reviews)` | **2** | podzapytanie + JOIN | nie | specyficzne plany zapytań |
| `immediateload(Book.author)` | **1 + N** | osobne `SELECT` na obiekt | nie | diagnostyka, migracje |
| `noload(Book.reviews)` | **1** | brak zapytania | nie | prawie nigdy (dane kłamią!) |
| `raiseload("*")` | **1** | brak zapytania (albo wyjątek) | nie | testy, audyt |

> 🧠 **Wniosek z tabeli**
> Dla **kolekcji** domyślnym wyborem jest `selectinload` — stała liczba zapytań, brak duplikatów, prosty SQL. Dla relacji **„do jednego”** `joinedload` jest optymalny, bo jedno zapytanie wystarcza i nie ma ryzyka eksplozji wierszy.

---

## Zagnieżdżone ładowanie i Load API

### Łańcuchy opcji

Opcje ładowania można łączyć w łańcuchy, opisujące ścieżkę przez kilka poziomów relacji.

```python
# examples/11_nested_1.py
from sqlalchemy import select
from sqlalchemy.orm import Session, joinedload, selectinload

from examples.models import Author


def authors_with_books_and_reviews(session: Session) -> list[Author]:
    stmt = (
        select(Author)
        .options(
            # ksiazki: JOIN-em, bo to "do jednego" patrzac od ksiazki do autora,
            # ale od autora to kolekcja — lacznie 1 zapytanie z JOIN
            joinedload(Author.books)
            # recenzje kazdej ksiazki: osobne IN (...) — 2. zapytanie
            .selectinload(Book.reviews)
        )
        .order_by(Author.name)
    )
    return list(session.execute(stmt).unique().scalars())
```

> 🔬 **Pod maską — łańcuch `joinedload(...).selectinload(...)`**
> ```sql
> -- zapytanie 1: autorzy + ich ksiazki (JOIN)
> SELECT author.id, author.name, author.birth_year,
>        book_1.id AS id_1, book_1.title, book_1.year, book_1.author_id
> FROM author
> LEFT OUTER JOIN book AS book_1 ON author.id = book_1.author_id
> ORDER BY author.name
>
> -- zapytanie 2: recenzje wszystkich wczytanych ksiazek (IN)
> SELECT review.book_id AS review_book_id, review.id, review.rating, review.body
> FROM review
> WHERE review.book_id IN (?, ?, ?, ...)
> ORDER BY review.book_id
> ```
> Dwa zapytania na dwa poziomy relacji. Gdyby drugi poziom też był `joinedload`, mielibyśmy jedno zapytanie — ale wiersze rosłyby jak iloczyn kartezjański.

Aby w jednym zapytaniu wczytać trzy poziomy, trzeba budować łańcuch:

```python
# examples/11_nested_3_levels.py
from sqlalchemy import select
from sqlalchemy.orm import Session, joinedload

from examples.models import Book


def three_levels_one_query(session: Session) -> list[Book]:
    stmt = (
        select(Book)
        .options(joinedload(Book.author).joinedload(Author.books))
        .order_by(Book.id)
    )
    return list(session.execute(stmt).unique().scalars())
```

> ⚠️ **Pułapka — kartezjański „wybuch” przy dwóch `joinedload` kolekcji**
> Jeśli do jednego zapytania dodasz `joinedload(Book.reviews)` **oraz** `joinedload(Book.loans)`, to dla książki z 100 recenzjami i 20 wypożyczeniami baza zwróci $100 \times 20 = 2000$ wierszy — a SQLAlchemy odfiltruje duplikaty. Oto jak to wygląda w SQL:
> ```sql
> FROM book
> LEFT OUTER JOIN review AS review_1 ON book.id = review_1.book_id
> LEFT OUTER JOIN loan AS loan_1   ON book.id = loan_1.book_id
> ```
> Zamiast tego użyj `joinedload` dla jednej kolekcji i `selectinload` dla drugiej — dwa zapytania, ale brak eksplozji.

### `defaultload()`

`defaultload()` stosuje opcję **głębiej w ścieżce**, ale nie wymusza własnego stylu ładowania na tym poziomie — używa tego, co jest już skonfigurowane na relacji. Przydaje się, gdy chcesz np. dołożyć warunek globalny (`with_loader_criteria`) nie zmieniając strategii.

```python
# examples/11_defaultload.py
from sqlalchemy import select
from sqlalchemy.orm import Session, defaultload, joinedload, with_loader_criteria

from examples.models import Author, Book


def authors_with_filtered_books(session: Session) -> list[Author]:
    stmt = (
        select(Author)
        .options(
            joinedload(Author.books),
            # Warunek stosowany do KAZDEGO wczytania Book w tym zapytaniu.
            with_loader_criteria(Book, Book.year >= 2000),
            # Delikatne dookreslenie glebiej: nie wymuszamy stylu, tylko go uzywamy.
            defaultload(Author.books).defer(Book.year, raiseload=True),
        )
        .order_by(Author.name)
    )
    return list(session.execute(stmt).unique().scalars())
```

### `with_loader_criteria()`

To jedno z najpotężniejszych narzędzi w API ładowania: pozwala nałożyć warunek na **wszystkie** wczytania danej encji w obrębie zapytania — również te wykonane przez opcje ładowania głębiej w ścieżce.

```python
# examples/11_with_loader_criteria_soft_delete.py
from datetime import datetime

from sqlalchemy import select
from sqlalchemy.orm import Session, selectinload, with_loader_criteria

from examples.models import Book, Review


def books_with_active_reviews(session: Session) -> list[Book]:
    stmt = (
        select(Book)
        .options(
            selectinload(Book.reviews),
            # Filtr "miekkiego usuniecia" dla recenzji — dolozony do kazdego loadu.
            with_loader_criteria(Review, Review.rating > 0),
        )
        .where(Book.year >= 1980)
    )
    return list(session.scalars(stmt))
```

> 💡 **Analogia — filtr „ze sobą w torbie”**
> `with_loader_criteria` to filtr, który pakujesz do torby razem z zapytaniem: cokolwiek SQLAlchemy w tym zapytaniu wyciągnie z danej tabeli (w tym przez relacje), automatycznie dostanie ten warunek. Bez tego musiałbyś dopisać go osobno w każdym miejscu — i wcześniej czy później o jednym zapomnisz.

> 🆕 **SQLAlchemy 2.1 — deterministyczne `with_loader_criteria` przy wielu ścieżkach**
> W wersjach 2.0 zdarzały się niejednoznaczności, gdy ten sam warunek miał zastosowanie do encji pojawiającej się w zapytaniu na kilka sposobów (np. przez aliasy). 2.1 porządkuje kolejność stosowania kryteriów, co czyni zachowanie przewidywalnym — to ważne, jeśli filtrujesz dane w wielu relacjach naraz.

### `defer`, `undefer`, `load_only`

Te opcje dotyczą **kolumn**, nie relacji. Pozwalają pobrać tylko część kolumn i zmniejszyć transfer danych.

```python
# examples/11_defer_load_only.py
from sqlalchemy import select
from sqlalchemy.orm import Session, defer, load_only, undefer

from examples.models import Book


def titles_only(session: Session) -> list[Book]:
    # Tylko tytul i identyfikator; reszta kolumn NIE leci po sieci.
    stmt = select(Book).options(load_only(Book.id, Book.title))
    return list(session.scalars(stmt))


def defer_year_with_raise(session: Session) -> list[Book]:
    # Year jest domyslnie odroczone, a kazdy przypadkowy dostep wybuchnie.
    stmt = select(Book).options(defer(Book.year, raiseload=True))
    return list(session.scalars(stmt))


def undefer_year(session: Session) -> list[Book]:
    # Juz teraz, mimo globalnego defer, chcemy year w glownym zapytaniu.
    stmt = select(Book).options(undefer(Book.year))
    return list(session.scalars(stmt))
```

> 🔬 **Pod maską — co robi `load_only`**
> ```sql
> -- zamiast: SELECT book.id, book.title, book.year, book.author_id FROM book
> SELECT book.id, book.title FROM book
> ```
> Przy szerokich tabelach (tekst opisów, blob obrazków, JSON-y) odroczenie kolumn daje większy zysk niż cała optymalizacja relacji. Traktuj `load_only` jak „przycinanie” ładunku.

### `lazy="raise"` i `raiseload("*")` jako zabezpieczenie w testach

Najczęstsza przyczyna regresji wydajnościowej to **nowa ścieżka kodu**, o której nikt nie pomyślał przy dodawaniu eager loadów. Zabezpieczenie: w warstwie testowej ustaw wszystkim relacjom `lazy="raise"` i wymuś jawne ładowanie.

```python
# tests/conftest.py (fragment)
from sqlalchemy.orm import Session, raiseload


def strict_session(session: Session) -> tuple[Session, object]:
    """Sesja, w ktorej kazdy niejawny load relacji konczy sie wyjatkiem."""
    session = session.execution_options()  # styl: przekazanie opcji do zapytan
    return session, raiseload("*")
```

Alternatywnie — i czyściej — w zapytaniu:

```python
# examples/11_raise_in_tests.py
from sqlalchemy import select
from sqlalchemy.orm import Session, raiseload

from examples.models import Book


def test_report_nie_wykonuje_niejawnych_zapytan(session: Session) -> None:
    stmt = select(Book).options(raiseload("*")).where(Book.year >= 2000)
    rows = list(session.scalars(stmt))  # jest OK — wolamy tylko skolumnizowane pola

    # Gdyby ktos nieopatrznie dotknal relacji — dostaniemy blad:
    # InvalidRequestError: 'Book.reviews' is not available due to lazy='raise'
    _ = [row.title for row in rows]
```

> 🧠 **Dlaczego „głośny błąd” jest lepszy od „cichego zapytania”**
> Ciche zapytanie to koszt, który pojawia się dopiero pod obciążeniem produkcyjnym — i to u użytkownika. Głośny błąd w testach to natychmiastowa informacja zwrotna. W systemach, w których wydajność jest krytyczna (API publiczne, raporty, eksporty), `lazy="raise"` na większości relacji jest standardem zdrowej praktyki.

---

## contains_eager — wykorzystanie własnego JOIN-a

Jeśli już robisz JOIN ręcznie (bo potrzebujesz filtru po kolumnie z łączonej tabeli), to szkoda, żeby SQLAlchemy wykonywała **drugie** zapytanie — tylko po to, by wypełnić relację. Do tego służy `contains_eager()`.

```python
# examples/11_contains_eager.py
from sqlalchemy import select
from sqlalchemy.orm import Session, contains_eager

from examples.models import Author, Book


def authors_with_new_books(session: Session) -> list[Author]:
    stmt = (
        select(Author)
        .join(Author.books)                              # reczne zlaczenie
        .options(contains_eager(Author.books))            # "wez dane z tego JOIN-a"
        .where(Book.year >= 2000)
        .order_by(Author.name)
    )
    return list(session.execute(stmt).unique().scalars())
```

> 🔬 **Pod maską — `contains_eager`**
> ```sql
> SELECT author.id, author.name, author.birth_year,
>        book_1.id AS id_1, book_1.title, book_1.year, book_1.author_id
> FROM author
> JOIN book AS book_1 ON author.id = book_1.author_id
> WHERE book_1.year >= 2000
> ORDER BY author.name
> ```
> Jedno zapytanie. Dane łączonej tabeli zostały „wciągnięte” do kolekcji `Author.books` — bez dodatkowego `SELECT`.

> ⚠️ **Pułapka — `contains_eager` bez odpowiadającego JOIN-a**
> Jeśli nie ma realnego `JOIN`-a (lub `join()` nie prowadzi dokładnie tam, gdzie myślisz), SQLAlchemy podniesie `ArgumentError`. `contains_eager` nie tworzy JOIN-a — on go zakłada.

> ⚠️ **Pułapka — filtr a obecność obiektów**
> Przy `JOIN` z warunkiem `Book.year >= 2000` autor bez nowszych książek **w ogóle nie pojawi się w wyniku**. Jeśli chcesz „wszystkich autorów, a dla nich ewentualnie nowe książki”, potrzebujesz `outerjoin` i warunku w `onclause` — a nie w `where`. Ten problem omawialiśmy w [Modul 06](06_joiny_i_zaawansowane_sql.md); tutaj ma on realne konsekwencje dla liczby zwracanych encji.

---

## Ładowanie poza sesją i DetachedInstanceError

Bardzo częsty scenariusz: pobierasz obiekt w jednym miejscu, zamykasz sesję, a potem próbujesz sięgnąć po relację. Wtedy wita Cię:

```text
DetachedInstanceError: Parent instance <Book at 0x...> is not bound to a Session;
lazy load operation of attribute 'reviews' cannot proceed
```

Zrozumienie, dlaczego tak się dzieje, jest kluczem do unikania tego błędu na zawsze.

```python
# examples/11_detached.py
from sqlalchemy import create_engine, select
from sqlalchemy.orm import Session

from examples.models import Book

engine = create_engine("sqlite:///biblioteka.db")

with Session(engine) as session:
    book = session.scalars(select(Book).where(Book.id == 1)).one()

# Po wyjsciu z bloku sesja jest zamknieta, obiekt = detached.
try:
    print(book.reviews)  # DetachedInstanceError
except Exception as exc:
    print(type(exc).__name__, exc)
```

> 💡 **Analogia — wyprowadzka z biblioteki**
> Obiekt poza sesją to jak czytelnik, który wyszedł z biblioteki i zostawił kartę biblioteczną na biurku. Wciąż „wie”, co przeczytał, ale **nie może już niczego zamówić** — nie ma połączenia z systemem. Każdy dostęp do nie-wczytanej relacji wymagałby nowego zapytania, a zapytanie potrzebuje sesji. Bez sesji — błąd.

Rozwiązania są cztery i każde ma swoje miejsce:

| Podejście | Kiedy stosować | Wady |
|---|---|---|
| **Eager load przed zamknięciem sesji** | zawsze, gdy wiesz, że dane pójdą dalej | trzeba wiedzieć z góry, co będzie potrzebne |
| **DTO / słownik** | serializacja w API, raporty | dodatkowa warstwa mapowania |
| **`expire_on_commit=False`** | gdy i tak wczytałeś wszystko przed `commit` | ryzyko pracy na nieaktualnych danych |
| **Sesja żyje przez cały request** | aplikacje web (FastAPI, Flask) | trzeba dopilnować zamknięcia i granic transakcji |

Pierwsze podejście jest najbezpieczniejsze:

```python
# examples/11_detached_fixed_eager.py
from sqlalchemy import create_engine, select
from sqlalchemy.orm import Session, selectinload

from examples.models import Book

engine = create_engine("sqlite:///biblioteka.db")

with Session(engine) as session:
    book = session.scalars(
        select(Book).options(selectinload(Book.reviews)).where(Book.id == 1)
    ).one()

# Teraz `book.reviews` jest w pamieci — dziala nawet po zamknieciu sesji.
print(len(book.reviews))
```

Drugie — DTO — jest standardem w warstwie API (zobaczysz to w [Modulu 22](22_fastapi_integracja.md)):

```python
# examples/11_detached_dto.py
from dataclasses import dataclass

from sqlalchemy import select
from sqlalchemy.orm import Session, selectinload

from examples.models import Book


@dataclass(frozen=True)
class BookDTO:
    id: int
    title: str
    reviewer_count: int
    author_name: str


def book_dto(session: Session, book_id: int) -> BookDTO:
    book = session.scalars(
        select(Book)
        .options(selectinload(Book.reviews))
        .where(Book.id == book_id)
    ).one()
    # Konwersja "w locie", dopoki sesja zyje.
    return BookDTO(
        id=book.id,
        title=book.title,
        reviewer_count=len(book.reviews),
        author_name=book.author.name,
    )
```

Trzecie podejście — `expire_on_commit=False` — bywa wygodne, ale zobacz ostrzeżenie:

> ⚠️ **Pułapka — `expire_on_commit=False` jako „kusząca łatwizna”**
> Domyślne `expire_on_commit=True` oznacza, że po `commit` wszystkie atrybuty zostają unieważnione i następny dostęp dociąga je od nowa — co wymaga sesji. Wyłączenie tego sprawia, że obiekt „przeżyje” commit i nie potrzebuje sesji do odczytu już wczytanych wartości. Ale wtedy **pracujesz na migawce** sprzed commitu, która może być nieaktualna po zmianach dokonanych przez inne procesy. Jeśli używasz tego w API, zawsze łącz to z eager loadingiem i świadomą granicą transakcji.

Czwarte podejście — sesja per request — jest omówione w [Modulu 21](21_unit_of_work.md). W dużym skrócie: sesja otwiera się na wejściu do endpointu, zamyka na wyjściu, a wszystkie odczyty (łącznie z leniwymi) mają do niej dostęp. To elegancko rozwiązuje problem, ale **nie usuwa** problemu N+1 — wręcz przeciwnie, czyni go łatwiejszym do przeoczenia.

> ⚠️ **Pułapka — leniwe ładowanie w `__repr__`**
> Jeśli `__repr__` modelu odwołuje się do relacji (`f"<Book {self.author.name}>"`), to samo **wypisanie** obiektu wywoła zapytanie. Zobaczysz to w logach jako zagadkowe, dodatkowe `SELECT` „bez powodu”. Zawsze kończ `__repr__` na kolumnach tabeli, nigdy na relacjach.

> 🆕 **SQLAlchemy 2.1 — dojrzalsze komunikaty o sesji po rollbacku**
> Począwszy od 2.1 komunikaty błędów związanych ze stanem sesji (np. próba użycia sesji wymagającej `rollback`) są bardziej precyzyjne i wskazują konkretną przyczynę. Pomaga to szybciej diagnozować sytuacje, w których relacja pozostaje w nieoczekiwanym stanie po wyjątku.

---

## Jak wybierać strategię — tabela decyzyjna

Poniższa tabela jest praktycznym streszczeniem całego modułu. Wypisz ją sobie i powieś nad biurkiem.

| Przypadek | Strategia | Liczba zapytań |
|---|---|---|
| Relacja **wiele-do-jednego** (np. `Book.author`) | `joinedload(...)` | 1 |
| Relacja **jeden-do-jednego** | `joinedload(...)` | 1 |
| **Kolekcja** (`Book.reviews`, `Author.books`) | `selectinload(...)` | 2 |
| Kolekcja + już robisz JOIN dla filtra | `contains_eager(...)` | 1 |
| Kolekcja o **znanym, stałym** rozmiarze (np. `<= 10`) | `joinedload(...)` + `unique()` | 1 |
| Dwa poziomy, różne rodzaje relacji | `joinedload(a).selectinload(b)` | 2 |
| Trzy poziomy w jednym zapytaniu | łańcuch `joinedload` + `unique()` | 1 (ale uwaga na eksplozję!) |
| **Duża kolekcja**, prawdopodobnie nieużywana | `lazy="write_only"` lub brak ładowania | 1 |
| **Testy / audyt** | `raiseload("*")` albo `lazy="raise"` | 1 lub wyjątek |
| **Strumieniowe** przetwarzanie (`yield_per`) | tylko `selectinload` z ostrożnością | 2 |
| Raport analityczny z agregacją | nie ładuj encji — wybierz kolumny | 1 |

Dodatkowo trzy zasady, które warto zapamiętać na stałe:

1. **N+1 naprawiasz w warstwie zapytań, nie w warstwie prezentacji.** Nie próbuj „cache'ować” w pętli.
2. **Dla kolekcji domyślnie `selectinload`.** `joinedload` na kolekcji jest wyjątkiem, nie regułą.
3. **Mierz.** Optymalizacja bez pomiaru to zgadywanie, a zgadywanie wydajnościowe bywa kosztowniejsze niż brak optymalizacji.

> ⚠️ **Pułapka — `yield_per` razem z `joinedload` kolekcji**
> Strumieniowe przetwarzanie (`yield_per`) wymaga, aby zapytanie nie zwracało „rozmnożonych” wierszy. Połączenie `yield_per` z `joinedload` na **kolekcji** kończy się błędem `InvalidRequestError` (SQLAlchemy informuje, że opcja `yield_per` jest niezgodna z eager loaderem kolekcji, który zwraca wiele wierszy na obiekt). Rozwiązanie: użyj `selectinload` albo zrezygnuj ze strumienia.

> 🆕 **SQLAlchemy 2.1 — `selectinload(chunksize=...)`**
> W 2.0 rozmiar klauzuli `IN (...)` rósł razem z liczbą obiektów nadrzędnych. Dla 2000 książek powstaje `IN` z 2000 parametrów, co może przekroczyć limit parametrów drivera (SQLite historycznie 999, PostgreSQL 65535, MySQL wysoki ale limitowany). W 2.1 możesz podzielić zapytanie na porcje:
> ```python
> # Wymaga SQLAlchemy >= 2.1
> selectinload(Book.reviews, chunksize=500)
> ```
> Zamiast jednego `IN (2000 parametrów)` powstaną cztery `IN (500 parametrów)`. Cena: liczba zapytań rośnie proporcjonalnie, ale zyskujesz odporność na limity driverów i stabilniejszy plan zapytań.

---

## Benchmark: trzy warianty tego samego raportu

Teraz zbudujemy pełny, uruchamialny plik, który wygeneruje dane i porówna trzy warianty tego samego raportu. Celem nie jest „użyj `selectinload`”, ale **zobaczenie liczb** i wyciągnięcie wniosków.

```python
# examples/11_benchmark.py
"""Benchmark: lazy vs joinedload vs selectinload na tym samym raporcie.

Uruchomienie:
    python -m examples.11_benchmark
Wymaga: pip install "SQLAlchemy>=2.0"
"""

from __future__ import annotations

import time

from sqlalchemy import create_engine, insert, select
from sqlalchemy.orm import Session, joinedload, selectinload

from examples.models import Author, Base, Book, Review
from examples.query_counter import QueryCounter

N_AUTHORS = 50
N_BOOKS = 500
N_REVIEWS = 5_000


def seed(session: Session) -> None:
    """Wypelnia baze deterministycznymi danymi."""
    session.execute(
        insert(Author),
        [{"name": f"Author {i:03d}", "birth_year": 1900 + i % 100} for i in range(N_AUTHORS)],
    )
    author_ids = list(session.scalars(select(Author.id)))

    session.execute(
        insert(Book),
        [
            {
                "title": f"Book {i:04d}",
                "year": 1950 + i % 70,
                "author_id": author_ids[i % len(author_ids)],
            }
            for i in range(N_BOOKS)
        ],
    )
    book_ids = list(session.scalars(select(Book.id)))

    session.execute(
        insert(Review),
        [
            {
                "book_id": book_ids[i % len(book_ids)],
                "rating": 1 + (i % 5),
                "body": f"Review body {i}",
            }
            for i in range(N_REVIEWS)
        ],
    )


def report_lazy(session: Session) -> list[str]:
    """Wariant A: domyslne leniwe ladowanie (N+1)."""
    books = session.scalars(select(Book).order_by(Book.id)).all()
    return [f"{b.title}|{b.author.name}|{len(b.reviews)}" for b in books]


def report_joined(session: Session) -> list[str]:
    """Wariant B: joinedload autora + selectinload recenzji."""
    stmt = (
        select(Book)
        .options(joinedload(Book.author), selectinload(Book.reviews))
        .order_by(Book.id)
    )
    books = session.scalars(stmt).all()
    return [f"{b.title}|{b.author.name}|{len(b.reviews)}" for b in books]


def report_joined_only(session: Session) -> list[str]:
    """Wariant C: dwa joinedload (autor + recenzje) — uwaga na duplikaty!"""
    stmt = (
        select(Book)
        .options(joinedload(Book.author), joinedload(Book.reviews))
        .order_by(Book.id)
    )
    books = session.execute(stmt).unique().scalars().all()
    return [f"{b.title}|{b.author.name}|{len(b.reviews)}" for b in books]


def measure(name: str, fn, engine) -> None:
    with Session(engine) as session:
        with QueryCounter(engine) as qc:
            result = fn(session)
        print(qc.summary(name))
        assert len(result) == N_BOOKS, "raport ma zwracac wszystkie ksiazki"


def main() -> None:
    engine = create_engine("sqlite:///benchmark_11.db", echo=False)
    Base.metadata.drop_all(engine)
    Base.metadata.create_all(engine)

    with Session(engine) as session:
        seed(session)
        session.commit()

    # Kolejnosc pomiarow: rozgrzewka + trzy warianty.
    measure("warmup (lazy)", report_lazy, engine)
    measure("A: lazy (N+1)", report_lazy, engine)
    measure("B: joined+selectin", report_joined, engine)
    measure("C: joined+joined", report_joined_only, engine)

    # Drugi przebieg: osobne sesje, ale identity map jest nowa.
    measure("A2: lazy (powtorka)", report_lazy, engine)
    measure("B2: joined+selectin", report_joined, engine)
    measure("C2: joined+joined", report_joined_only, engine)


if __name__ == "__main__":
    main()
```

Typowy wynik na typowym laptopie (SQLite w pliku, wszystkie pomiary „na zimno”) wygląda następująco:

```text
warmup (lazy)              zapytan:    51  czas:    120.4 ms
A: lazy (N+1)              zapytan:    51  czas:     98.7 ms
B: joined+selectin         zapytan:     2  czas:     14.1 ms
C: joined+joined           zapytan:     1  czas:     42.6 ms
A2: lazy (powtorka)        zapytan:    51  czas:    101.2 ms
B2: joined+selectin        zapytan:     2  czas:     13.9 ms
C2: joined+joined          zapytan:     1  czas:     41.8 ms
```

Wnioski, które wynikają z liczb (a nie z opinii):

1. **Liczba zapytań to dominujący czynnik.** 51 zapytań w wariancie A daje ~100 ms; 2 zapytania w wariancie B — ~14 ms. To siedmiokrotna różnica na bardzo szybkiej, lokalnej bazie. Na PostgreSQL przez sieć przewaga B byłaby znacznie większa.
2. **`joinedload` na kolekcji nie zawsze wygrywa.** Wariant C wykonuje **jedno** zapytanie, ale zwraca znacznie więcej wierszy (500 książek × ~10 recenzji = ~5000 wierszy plus duplikaty) i jest wolniejszy niż B (42 ms vs 14 ms). Mniej zapytań nie znaczy automatycznie szybciej.
3. **`selectinload` dla kolekcji to złoty środek** — dwa zapytania o prostym, indeksowalnym `IN`, bez duplikatów, bez eksplozji.

> 🧪 **Ćwiczenie — zmień proporcje danych**
> Ustaw `N_BOOKS = 50` i `N_REVIEWS = 50`. Uruchom benchmark ponownie. Która strategia wygrywa przy **małych** kolekcjach? Która przy dużych? Zapisz obserwację, bo to pokazuje, dlaczego tabela decyzyjna nie jest uniwersalna.

> ⚠️ **Pułapka — benchmark bez danych**
> Mierzenie wydajności na pustej tabeli nie mówi nic. Plany zapytań zmieniają się dramatycznie przy tysiącach wierszy — baza wybiera inne strategie łączenia. Zawsze mierz na realistycznym wolumenie danych.

> ⚠️ **Pułapka — pomiar „na rozgrzanym cache”**
> SQLite i PostgreSQL trzymają w pamięci ostatnio używane strony. Drugi przebieg jest zwykle szybszy. Dlatego w benchmarku robimy rozgrzewkę i powtarzamy pomiary (warianty `A2`, `B2`, `C2`), by nie porównywać „zimnego A” z „ciepłym B”.

---

## Podsumowanie

- **Problem N+1** oznacza $O(1+N)$ zapytań zamiast $O(1)$: jedno zapytanie na listę i po jednym na każdą relację. To najczęstsza przyczyna nagłego spadku wydajności w aplikacjach ORM.
- **`lazy="select"` jest domyślne** i bezpieczne dla pojedynczych obiektów, ale zabójcze w pętli po wielu encjach.
- **`selectinload()` to domyślny wybór dla kolekcji** — stała liczba dwóch zapytań, brak duplikatów wierszy, prosty `IN (...)`.
- **`joinedload()` to domyślny wybór dla relacji „do jednego”** — jedno zapytanie, `LEFT OUTER JOIN`, brak ryzyka eksplozji wierszy.
- **`joinedload` na kolekcji wymaga `unique()`** i może doprowadzić do kartezjańskiego wybuchu, gdy w zapytaniu są dwie kolekcje.
- **`contains_eager()`** pozwala wykorzystać już wykonany JOIN zamiast dokładać drugie zapytanie.
- **`raiseload("*")` i `lazy="raise"`** zamieniają ciche, kosztowne zapytania w głośne błędy — to najskuteczniejsza ochrona przed regresją wydajnościową.
- **`lazy="write_only"`** jest dla wielkich kolekcji, których aplikacja nigdy nie czyta w całości.
- **DetachedInstanceError** to skutek sięgania po relację poza sesją; rozwiązania: eager load, DTO, sesja per request, świadome `expire_on_commit=False`.
- **Mierz, nie zgaduj.** Licznik zapytań (`before_cursor_execute`) i benchmark z realistycznymi danymi to jedyne wiarygodne źródło decyzji.

---

## Ćwiczenia

### Zadanie 1 — Licznik zapytań w testach (łatwe)

Dodaj do projektu testowy fixture `assert_num_queries` (wzór wyżej). Napisz test, który potwierdza, że funkcja raportu z [Modułu 10](10_zapytania_orm.md) — ta, która zwraca listę książek z autorami — wykonuje **dokładnie jedno** zapytanie po dodaniu `joinedload(Book.author)`.

**Kryterium akceptacji:** usunięcie `joinedload` z zapytania powoduje, że test **ma się wywalić** z komunikatem zawierającym faktyczną liczbę zapytań.

### Zadanie 2 — Trzy poziomy relacji minimalną liczbą zapytań (średnie)

Zbuduj zapytanie, które dla wszystkich autorów wczyta: (a) książki, (b) recenzje każdej książki, (c) liczbę wypożyczeń każdej książki — w **możliwie najmniejszej liczbie zapytań**, bez duplikowania wierszy i bez eksplozji kartezjańskiej.

Zaproponuj co najmniej dwa warianty i zmierz oba licznikiem zapytań. Uzasadnij wybór liczbami.

### Zadanie 3 — Napraw „wolny endpoint” (trudne)

Poniższy kod odpowiada za widok listy 200 książek. Uruchom go, zmierz liczbę zapytań, a następnie przepisz zapytanie tak, aby liczba zapytań była stała (niezależna od liczby książek).

```python
# examples/11_slow_endpoint.py
from sqlalchemy import select
from sqlalchemy.orm import Session

from examples.models import Book


def list_books(session: Session) -> list[dict]:
    books = session.scalars(select(Book).order_by(Book.year.desc())).all()
    return [
        {
            "title": book.title,
            "author": book.author.name,
            "reviews": len(book.reviews),
            "loans": len(book.loans),
            "avg_rating": (
                sum(r.rating for r in book.reviews) / len(book.reviews)
                if book.reviews
                else None
            ),
        }
        for book in books
    ]
```

Zwróć uwagę, że wyliczasz pozycje DTO **poza sesją** w oryginalnym kodzie — to dodatkowo prowokuje `DetachedInstanceError` w wersji API. Popraw oba problemy.

---

### Rozwiązania

#### Rozwiązanie zadania 1

```python
# tests/test_report_queries.py
from sqlalchemy import select
from sqlalchemy.orm import joinedload

from examples.models import Book


def test_report_uzywa_jednego_zapytania(session, assert_num_queries):
    stmt = select(Book).options(joinedload(Book.author)).order_by(Book.id)

    with assert_num_queries() as counter:
        counter.expected = 1
        books = list(session.scalars(stmt))
        _ = [b.author.name for b in books]
```

**Dlaczego tak:** jedno zapytanie z `LEFT OUTER JOIN` wystarcza dla relacji „do jednego”. `joinedload(Book.author)` ładuje autora razem z książką, a każde `b.author.name` sięga do już wczytanych danych — bez dodatkowego SQL.

**Alternatywa:** ten sam test z `selectinload(Book.author)` też jest poprawny, ale wykonałby 2 zapytania (`selectin` jest zoptymalizowane dla kolekcji, dla „do jednego” JOIN jest lepszy). Gdyby test miał oczekiwać 2, byłby równie poprawny — ale mniej wydajny na produkcji.

#### Rozwiązanie zadania 2

**Wariant B1 — dwie kolumny, 2 zapytania:**

```python
# examples/11_ex2_variant_a.py
from sqlalchemy import func, select
from sqlalchemy.orm import Session, joinedload, selectinload

from examples.models import Author, Book, Loan, Review


def variant_a(session: Session) -> list[Book]:
    counts = (
        select(Loan.book_id, func.count().label("loan_count"))
        .group_by(Loan.book_id)
        .subquery()
    )
    stmt = (
        select(Book)
        .options(
            joinedload(Book.author),
            joinedload(Book).selectinload(Book.reviews),
        )
        .outerjoin(counts, counts.c.book_id == Book.id)
        .add_columns(func.coalesce(counts.c.loan_count, 0).label("loan_count"))
        .order_by(Book.id)
    )
    # Zwracamy pary (Book, loan_count) — daleko od encji to juz DTO.
    return [row[0] for row in session.execute(stmt).unique()]


def variant_a_counts(session: Session) -> dict[int, int]:
    counts = (
        select(Loan.book_id, func.count().label("loan_count"))
        .group_by(Loan.book_id)
        .subquery()
    )
    stmt = select(Book.id, func.coalesce(counts.c.loan_count, 0)).outerjoin(
        counts, counts.c.book_id == Book.id
    )
    return {book_id: n for book_id, n in session.execute(stmt)}
```

**Wariant B2 — minimalna liczba zapytań z zachowaniem DTO, 2–3 zapytania:**

```python
# examples/11_ex2_variant_b.py
from dataclasses import dataclass

from sqlalchemy import func, select
from sqlalchemy.orm import Session, joinedload, selectinload

from examples.models import Book, Loan


@dataclass(frozen=True)
class BookSummary:
    title: str
    author: str
    review_count: int
    review_avg: float | None
    loan_count: int


def variant_b(session: Session) -> list[BookSummary]:
    loan_counts = (
        select(Loan.book_id, func.count().label("loan_count"))
        .group_by(Loan.book_id)
        .subquery()
    )
    stmt = (
        select(Book)
        .options(joinedload(Book.author), selectinload(Book.reviews))
        .outerjoin(loan_counts, loan_counts.c.book_id == Book.id)
        .add_columns(func.coalesce(loan_counts.c.loan_count, 0).label("loan_count"))
        .order_by(Book.id)
    )
    return [
        BookSummary(
            title=book.title,
            author=book.author.name,
            review_count=len(book.reviews),
            review_avg=(
                sum(r.rating for r in book.reviews) / len(book.reviews)
                if book.reviews
                else None
            ),
            loan_count=int(loan_count),
        )
        for book, loan_count in session.execute(stmt).unique()
    ]
```

**Dlaczego tak:** dla trzech „kolekcji” w jednym zapytaniu `joinedload` spowodowałby eksplozję — liczba wierszy byłaby iloczynem ich rozmiarów. Dlatego autor (relacja „do jednego”) idzie przez `joinedload`, recenzje przez `selectinload`, a liczba wypożyczeń jest **agregowana w bazie** (jedno zapytanie z `GROUP BY`), a nie liczona przez dodatkowy `len(book.loans)`.

**Alternatywa i kompromis:** gdyby wypożyczeń było mało, można by je zostawić jako `selectinload(Book.loans)`. Wtedy jednak każda książka byłaby wczytana z całą historią wypożyczeń w pamięci — przy 1 mln wierszy to katastrofa. Agregacja w bazie to zysk pamięciowy i wydajnościowy.

**Minipułapka:** `func.coalesce(..., 0)` jest konieczny — książka bez wypożyczeń nie pojawi się w podzapytaniu agregującym, więc `loan_count` byłoby `NULL`. Zapominając o `coalesce`, dostaniesz `None`, a nie `0`.

#### Rozwiązanie zadania 3

```python
# examples/11_fast_endpoint.py
from sqlalchemy import func, select
from sqlalchemy.orm import Session, joinedload, selectinload

from examples.models import Book, Loan


def list_books(session: Session) -> list[dict]:
    loan_counts = (
        select(Loan.book_id, func.count().label("loan_count"))
        .group_by(Loan.book_id)
        .subquery()
    )
    review_agg = (
        select(
            Book.id.label("book_id"),
            func.count().label("review_count"),
            func.avg(Book.reviews.property.mapper.class_.rating).label("avg_rating"),
        )
        .join(Book.reviews)
        .group_by(Book.id)
        .subquery()
    )

    stmt = (
        select(Book)
        .options(joinedload(Book.author))
        .outerjoin(loan_counts, loan_counts.c.book_id == Book.id)
        .outerjoin(review_agg, review_agg.c.book_id == Book.id)
        .add_columns(
            func.coalesce(review_agg.c.review_count, 0).label("review_count"),
            review_agg.c.avg_rating.label("avg_rating"),
            func.coalesce(loan_counts.c.loan_count, 0).label("loan_count"),
        )
        .order_by(Book.year.desc())
    )

    return [
        {
            "title": book.title,
            "author": book.author.name,
            "reviews": int(review_count),
            "loans": int(loan_count),
            "avg_rating": float(avg_rating) if avg_rating is not None else None,
        }
        for book, review_count, avg_rating, loan_count in session.execute(stmt).unique()
    ]
```

**Dlaczego tak:** zamieniamy dwa leniwe `len()` (które wygenerowałyby $2N$ zapytań) oraz pętlę po recenzjach na **dwie agregacje w bazie** (`GROUP BY`). Autor, jako relacja „do jednego”, idzie przez `joinedload`. Liczba zapytań jest stała: **1** (plus ewentualne zapytania poboczne warstwy wywołującej).

**Alternatywa i kompromis:** wariant z `selectinload(Book.reviews)` i policzeniem `avg` w Pythonie jest krótszy, ale przesyła **wszystkie** recenzje do aplikacji — przy 5000 recenzjach to 5000 dodatkowych wierszy w transferze. Agregacja w bazie przenosi obliczenia tam, gdzie dane już są. Jeśli w przyszłości raport wymagałby dodatkowo danych każdej recenzji — wtedy `selectinload` wraca do gry.

**Minipułapka:** `func.avg` w SQLite na kolumnie `Integer` zwraca `float`; w PostgreSQL również — ale sumowanie i dzielenie w Pythonie (`sum(...)/len(...)`) mogłoby dać inny wynik przy typach całkowitych. Zawsze ujednolicaj typy agregacji po stronie bazy (`func.avg`, `func.sum`, `cast`).

---

## Najczęstsze błędy i jak je czytać

| Komunikat / objaw | Przyczyna | Naprawa |
|---|---|---|
| `InvalidRequestError: The unique() method must be invoked on this Result, as it contains results that include joined eager loads against collections` | `joinedload` na kolekcji zwraca zduplikowane wiersze; wynik odbierany bez `unique()` | `session.execute(stmt).unique().scalars().all()` albo przełącz na `selectinload` |
| `DetachedInstanceError: Parent instance ... is not bound to a Session` | dostęp do nie-wczytanej relacji po zamknięciu sesji | eager load przed zamknięciem; DTO; sesja per request |
| Nagłe spowolnienie przy większej liczbie rekordów, log z setkami identycznych `SELECT ... WHERE id = ?` | N+1 z `lazy="select"` w pętli | dodaj `joinedload` (relacja „do jednego”) lub `selectinload` (kolekcja) |
| Zapytanie zwraca setki wierszy tam, gdzie oczekujesz kilkunastu | kartezjański wybuch: dwie `joinedload` kolekcji w jednym zapytaniu | jedna kolekcja `joinedload`, druga `selectinload` |
| `InvalidRequestError` przy `yield_per` połączonym z eager loadingiem kolekcji | `yield_per` jest niezgodny z loaderami kolekcji zwracającymi wiele wierszy na obiekt | użyj `selectinload` albo usuń `yield_per` |
| `ArgumentError: Can't find property named ...` przy `contains_eager` | brak odpowiadającego `JOIN`-a w zapytaniu | dodaj `join()` dokładnie po tej relacji, którą chcesz wypełnić |
| `InvalidRequestError: 'X' is not available due to lazy='raise'` | relacja ma `lazy="raise"`, a kod sięga po nią niejawnie | dodaj jawną opcję ładowania w zapytaniu |
| Zagadkowe, dodatkowe `SELECT`, których nie ma w kodzie | leniwe ładowanie w `__repr__`, w loggerze albo w serializacji | sprawdź `__repr__`; nie sięgaj po relacje w miejscach „diagnostycznych” |
| `SELECT` z `IN (...)` z tysiącami parametrów, błąd drivera o zbyt wielu parametrach | `selectinload` bez podziału na porcje na dużym zbiorze | obniż rozmiar zapytania (`chunksize` w 2.1) lub podziel dane ręcznie |
| Wynik pusty tam, gdzie dane na pewno są | `noload` albo `lazy="noload"` na relacji | zmień na `selectin`/`joined` lub ładuj jawnie |
| `sqlalchemy.exc.MissingGreenlet` (tylko async) | leniwe ładowanie w kontekście asynchronicznym bez greenlet | używaj `AsyncAttrs` + `await obj.awaitable_attrs.x` albo eager loadingu ([Modul 15](15_asynchronicznosc.md)) |

---

## Słowniczek modułu

| Termin (EN) | Polski odpowiednik | Wyjaśnienie |
|---|---|---|
| lazy loading | ładowanie leniwe | dane relacji pobierane dopiero przy dostępie do atrybutu |
| eager loading | ładowanie zachłanne | dane relacji pobierane z góry, razem z obiektem nadrzędnym |
| N+1 problem | problem N+1 | $1+N$ zapytań zamiast stałej ich liczby; jedno na listę, $N$ na relacje |
| `selectinload` | ładowanie zbiorcze przez `IN` | kolekcja dociągana jednym dodatkowym `SELECT ... IN (...)` |
| `joinedload` | ładowanie przez `JOIN` | relacja dociągana tym samym zapytaniem, przez `LEFT OUTER JOIN` |
| `subqueryload` | ładowanie przez podzapytanie | historyczna alternatywa dla `selectin`; rzadziej używana |
| `immediateload` | ładowanie natychmiastowe | jak leniwe, ale wykonane od razu, nadal jedno na obiekt |
| `noload` | brak ładowania | relacja pozostaje pusta; nie znaczy „nie ma danych” |
| `raiseload` | ładowanie „wybuchowe” | dostęp do nie-wczytanej relacji kończy się wyjątkiem |
| `write_only` | tylko do zapisu | kolekcja, którą można dopisywać, ale nie iterować bez jawnego ładowania |
| `contains_eager` | użycie istniejącego JOIN-a | wypełnienie relacji danymi z zapytania, które już wykonuje `JOIN` |
| `with_loader_criteria` | kryterium ładowania | warunek nakładany na każde wczytanie danej encji w zapytaniu |
| `defaultload` | ładowanie domyślną strategią | opcja stosowana głębiej w ścieżce, bez narzucania stylu |
| `load_only` | tylko wskazane kolumny | pobranie wyłącznie wymienionych kolumn zamiast całej encji |
| `defer` | odroczenie kolumny | kolumna pominięta w głównym zapytaniu, dociągana na żądanie |
| `undefer` | zdjęcie odroczenia | wymuszenie pobrania odroczonej kolumny w zapytaniu |
| identity map | mapa tożsamości | rejestr obiektów wczytanych w sesji; wpływa na liczbę zapytań |
| detached | odłączony | obiekt poza sesją; nie może wykonać leniwego ładowania |
| cartesian explosion | eksplozja kartezjańska | niekontrolowane mnożenie wierszy przy dwóch `joinedload` kolekcji |
| round-trip | okrążenie do bazy | pojedyncza wymiana danych aplikacja ↔ baza; wąskie gardło wydajności |
| `chunksize` | rozmiar porcji | parametr dzielący `IN (...)` na mniejsze części (SQLAlchemy 2.1) |

---

## Dalsze czytanie

- Relationship Loading Techniques (przewodnik główny, SQLAlchemy 2.0): <https://docs.sqlalchemy.org/en/20/orm/queryguide/relationships.html>
- Relationship Loading with Loader Options (lista wszystkich opcji): <https://docs.sqlalchemy.org/en/20/orm/queryguide/relationships.html#relationship-loading-with-loader-options>
- What Kind of Loading to Use (oficjalna tabela decyzyjna, wersja dokumentacji): <https://docs.sqlalchemy.org/en/20/orm/queryguide/relationships.html#what-kind-of-loading-to-use>
- Zdarzenia ORM (m.in. `before_cursor_execute`, licznik zapytań): <https://docs.sqlalchemy.org/en/20/orm/events.html>
- Zdarzenia Core (`Engine`, `Connection`): <https://docs.sqlalchemy.org/en/20/core/events.html>
- `Load` API (`defer`, `raiseload`, `with_loader_criteria`): <https://docs.sqlalchemy.org/en/20/orm/queryguide/api.html>
- Sesja i cykl życia obiektu: <https://docs.sqlalchemy.org/en/20/orm/session.html>
- Wersja 2.1 przewodnika po ładowaniu relacji (dla `chunksize`, `omit_join`): <https://docs.sqlalchemy.org/en/21/orm/queryguide/relationships.html>
- Nowości i migracja w SQLAlchemy 2.1: <https://docs.sqlalchemy.org/en/21/changelog/migration_21.html>
- Dokumentacja logowania w SQLAlchemy: <https://docs.sqlalchemy.org/en/20/core/engines.html#configuring-logging>

---

## Co dalej

Wiesz już, jak kontrolować liczbę i kształt zapytań oraz jak mierzyć skutki tych decyzji. W [Module 12](12_typy_i_wlasne_typy.md) zajmiemy się warstwą, która stoi poniżej zapytań — **typami danych**. Nauczysz się dobierać właściwe typy kolumn, obsługiwać JSON, `Uuid`, `Enum` i strefy czasowe, a przede wszystkim budować własne typy przez `TypeDecorator`, gdy mapa wbudowanych konwersji Pythona na SQL nie wystarcza.

<!-- koniec modułu 11 -->