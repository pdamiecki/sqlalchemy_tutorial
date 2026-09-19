# Moduł 17 — Wydajność: mierz, diagnozuj, naprawiaj

Do tej pory nauczyłeś się pisać kod ze SQLAlchemy *poprawnie*. Ten moduł uczy pisać go *szybko* — ale nie przez zgadywanie, tylko przez pomiar. Zobaczysz, jak w praktyce wygląda diagnoza „ta strona ładuje się 3 sekundy”, jak odróżnić problem sieciowy od problemu z zapytaniem, jak użyć `EXPLAIN`, jak porównać cztery strategie wstawiania 100 tysięcy wierszy i jak zbudować licznik zapytań, który w kilka linijkach pokaże Ci prawdę o Twoim ORM-ie. Najważniejsza teza całego modułu brzmi: **optymalizacja bez pomiaru to nie optymalizacja, a loteria**. Po tym module będziesz mieć w rękach zestaw narzędzi, który zamienia „wydaje mi się, że to jest wolne” w „wiem, że to jest wolne, bo zmierzyłem 41 ms i 312 zapytań”.

Ten moduł jest trudny nie dlatego, że API jest skomplikowane (jest proste), ale dlatego, że wymaga **dyscypliny myślenia**. Wydajność to dziedzina, w której intuicja zawodzi najczęściej. Rzeczy, które wyglądają na drogie, są tanie (jedno duże zapytanie z JOIN-em), a rzeczy, które wyglądają na darmowe, są katastrofalne (pętla `for book in books: print(book.author.name)`). Nauczysz się więc przede wszystkim *patrzeć na liczby*.

---

| | |
|---|---|
| **Poziom** | 🔴 architektoniczny |
| **Czas** | ~180 minut (plus ~40 minut na warsztat pomiarowy) |
| **Wymagania wstępne** | [`02_srodowisko_i_engine.md`](02_srodowisko_i_engine.md) — pula połączeń, `echo`; [`08_sesja_cykl_zycia.md`](08_sesja_cykl_zycia.md) — `expire_on_commit`, identity map; [`10_zapytania_orm.md`](10_zapytania_orm.md) — `select()` w ORM; [`11_ladowanie_i_n_plus_1.md`](11_ladowanie_i_n_plus_1.md) — strategie ładowania (ten moduł zakłada, że znasz problem N+1) |
| **Czego dotyczy plik** | Metodologia pomiaru, indeksy, operacje masowe, strumieniowanie, cache kompilacji, cykl życia sesji a wydajność, konfiguracja połączeń, antywzorce i checklist przeglądu |
| **Czego ten moduł NIE jest** | Nie jest kursem tuningu PostgreSQL. Administracja bazą (VACUUM, autovacuum, partycjonowanie, replikacja) to osobna, głęboka dziedzina |
| **Zasada nadrzędna** | Każda liczba w tym module jest **przykładowa** i pochodzi z konkretnego, skromnego środowiska. Twoje liczby będą inne. Wnioski *jakościowe* (co jest 3× szybsze, a co 300×) są przenośne; liczby bezwzględne — nie |

---

## Spis treści

1. [Metodologia: lekarz, nie szaman](#metodologia-lekarz-nie-szaman)
2. [Cztery największe dźwignie wydajności](#cztery-największe-dźwignie-wydajności)
3. [Indeksy: kiedy pomagają, kiedy szkodzą](#indeksy-kiedy-pomagają-kiedy-szkodzą)
4. [Wstawianie masowe: cztery strategie](#wstawianie-masowe-cztery-strategie)
5. [Masowa aktualizacja i usuwanie](#masowa-aktualizacja-i-usuwanie)
6. [Odczyty dużych zbiorów: `all()` vs strumień](#odczyty-dużych-zbiorów-all-vs-strumień)
7. [Cache kompilacji SQL](#cache-kompilacji-sql)
8. [Sesja, `expire_on_commit` i rosnąca identity map](#sesja-expire_on_commit-i-rosnąca-identity-map)
9. [Wzorce zapytań: mniej danych, mniej rund](#wzorce-zapytań-mniej-danych-mniej-rund)
10. [Konfiguracja połączeń i sieć](#konfiguracja-połączeń-i-sieć)
11. [Antywzorce wydajnościowe](#antywzorce-wydajnościowe)
12. [Warsztat pomiarowy: 1 000 000 wierszy](#warsztat-pomiarowy-1-000-000-wierszy)
13. [Checklist optymalizacyjny](#checklist-optymalizacyjny)

---

## Metodologia: lekarz, nie szaman

### Problem, który rozwiązujemy

Wyobraź sobie, że przychodzisz do lekarza i mówisz: „boli mnie głowa”. Lekarz, który od razu bez pytania wypisuje silny lek przeciwbólowy, to szaman. Lekarz, który najpierw mierzy ciśnienie, pyta o sen, zleca badania krwi i dopiero potem coś przepisuje — to profesjonalista.

Programiści optymalizujący „na wyczucie” są szamanami. Zmieniają `lazy="select"` na `joinedload`, dodają trzy indeksy, włączają cache — i nie mają pojęcia, czy cokolwiek pomogło, czy tylko przesunęli problem w inne miejsce.

> 💡 **Analogia — wizyta u lekarza.** Pomiar wydajności ma trzy etapy i każdy z nich odpowiada pytaniu lekarza:
> 1. **„Co pana boli?”** — profilowanie: *gdzie* tracimy czas? (endpoint, funkcja, pojedyncze zapytanie)
> 2. **„Jak bardzo?”** — pomiar: ile czasu, ile zapytań, ile bajtów, ile wierszy?
> 3. **„Od kiedy?”** — regresja: co się zmieniło, że wcześniej było dobrze? (nowy kod, nowe dane, gorszy plan zapytania)
>
> Dopiero po tych trzech odpowiedziach wolno Ci napisać jedną linię kodu „optymalizującego”.

### Reguła jednej zmiany

To najważniejsza praktyczna reguła tego modułu:

> 🧠 **Dlaczego tak jest.** Jeśli zmienisz naraz pięć rzeczy i aplikacja przyspieszy o 30%, nie wiesz, która zmiana dała efekt, a która go pogorszyła (i tylko suma wyszła na plus). Zmieniaj **jedną rzecz, mierz, zapisz wynik, idź dalej**. Mikroskopijnie nudne, ale to jedyny sposób, żeby zbierać wiedzę zamiast zgadywać.

### Narzędzie 1: `echo` — najprostszy podgląd SQL

Najprostszy sposób, żeby zobaczyć, co naprawdę dzieje się w bazie:

```python
# examples/17_echo.py
from sqlalchemy import create_engine, select
from sqlalchemy.orm import Session

from models import Book

engine = create_engine("sqlite:///:memory:", echo=True)

with Session(engine) as session:
    books = session.scalars(select(Book)).all()
```

`echo=True` wypisuje na konsolę każde zapytanie SQL wraz z czasem wykonania i informacją o parametrach. To wystarcza, żeby zobaczyć najczęstszy problem świata ORM — powtarzające się zapytanie w pętli.

> 🆕 **SQLAlchemy 2.1** — log wygląda inaczej
> W 2.1 log `echo` został uporządkowany i zawiera bardziej czytelne znaczniki czasu oraz oznaczenie, czy wykonanie było „executemany”. Samo włączenie działa identycznie (`echo=True` albo `logging`), ale czytanie logu jest przyjemniejsze.

### Narzędzie 2: logging zamiast `echo`

`echo=True` to przełącznik globalny dla całego silnika. W aplikacji produkcyjnej chcesz logować **selektywnie** i włączać to na życzenie:

```python
# examples/17_logging.py
import logging
from sqlalchemy import create_engine

logger = logging.getLogger("sqlalchemy.engine")
logger.setLevel(logging.INFO)

if not logger.handlers:
    handler = logging.StreamHandler()
    handler.setFormatter(logging.Formatter("%(asctime)s [%(levelname)s] %(message)s"))
    logger.addHandler(handler)

engine = create_engine("sqlite:///:memory:")
```

Poziomy mają znaczenie:

| Poziom | Co pokazuje |
|---|---|
| `INFO` | Każde wykonane zapytanie, z czasem i statusem |
| `DEBUG` | Dodatkowo: wynikowe wiersze, parametry, kolejność ładowania relacji |

> ⚠️ **Pułapka.** `echo=True` zostawione w kodzie produkcyjnym to najczęstsza „niewidoczna” degradacja wydajności. Logowanie do pliku na dysku sieciowym, w pętli, przy tysiącach zapytań na sekundę — potrafi zjeść więcej czasu niż same zapytania. Wymuś przez konfigurację (`echo` czytane z ENV, domyślnie `False`) i sprawdź w code review.

### Narzędzie 3: licznik zapytań — narzędzie numer jeden

Czasami nie interesuje Cię *jaki* SQL leci, tylko **ile razy**. Do tego służy licznik zapytań oparty na zdarzeniach (eventach) silnika:

```python
# examples/17_query_counter.py
from __future__ import annotations

import time
from contextlib import contextmanager
from dataclasses import dataclass, field
from typing import Iterator

from sqlalchemy import Engine, event


@dataclass
class QueryStats:
    """Zbiorczy wynik pomiaru zapytań."""

    count: int = 0
    total_s: float = 0.0
    statements: list[str] = field(default_factory=list)

    def summary(self) -> str:
        avg_ms = (self.total_s / self.count * 1000) if self.count else 0.0
        return (
            f"zapytania={self.count}  czas_bazy={self.total_s * 1000:.1f} ms  "
            f"średnio={avg_ms:.2f} ms/zapytanie"
        )


@contextmanager
def count_queries(
    engine: Engine, *, keep_statements: bool = False, repeat_fast: bool = False
) -> Iterator[QueryStats]:
    """Kontekst menedżer mierzący liczbę i łączny czas zapytań na silniku."""
    stats = QueryStats()
    started_at: dict[int, float] = {}

    def before(conn, cursor, statement, parameters, context, executemany):
        started_at[id(cursor)] = time.perf_counter()

    def after(conn, cursor, statement, parameters, context, executemany):
        start = started_at.pop(id(cursor), None)
        stats.count += 1
        if start is not None:
            stats.total_s += time.perf_counter() - start
        if keep_statements:
            stats.statements.append(statement)

    event.listen(engine, "before_cursor_execute", before)
    event.listen(engine, "after_cursor_execute", after)
    try:
        yield stats
    finally:
        event.remove(engine, "before_cursor_execute", before)
        event.remove(engine, "after_cursor_execute", after)
```

Użycie:

```python
from sqlalchemy import select
from sqlalchemy.orm import Session

from models import Book

with Session(engine) as session, count_queries(engine) as stats:
    books = session.scalars(select(Book).limit(50)).all()
    for book in books:
        _ = book.author.name  # leniwe ładowanie relacji!

print(stats.summary())
# zapytania=51  czas_bazy=418.6 ms  średnio=8.21 ms/zapytanie
```

> 🔬 **Pod maską.** Powyższy kod wykonał **jedno** zapytanie po książki i **pięćdziesiąt** zapytań po autorów. Klasyczne N+1 w pełnej krasie, widoczne w liczniku po pół sekundzie pracy. Naprawa tego to `selectinload(Book.author)`, po której licznik pokaże `2`.

Tego licznika będziesz używać przez cały moduł. Zbuduj go raz i trzymaj w `tests/conftest.py` (moduł [`18_testowanie.md`](18_testowanie.md)) — test, który mówi „ta funkcja nie może wykonać więcej niż 3 zapytań”, to najtańsze zabezpieczenie przed regresją wydajności, jakie możesz mieć.

### Narzędzie 4: `time.perf_counter` — mierzenie kodu Pythona

```python
import time
from sqlalchemy import select
from sqlalchemy.orm import Session

with Session(engine) as session:
    start = time.perf_counter()
    rows = session.scalars(select(Book)).all()
    elapsed = time.perf_counter() - start

print(f"odczyt {len(rows)} książek: {elapsed * 1000:.1f} ms")
```

`time.perf_counter()` mierzy czas monotoniczny o najwyższej dostępnej rozdzielczości — to właściwe narzędzie do mierzenia krótkich fragmentów. `time.time()` mierzy czas zegarowy, który może się cofnąć przy zmianie zegara systemowego.

> ⚠️ **Pułapka.** Jeden pomiar to nie pomiar. Pierwsze uruchomienie kodu jest wolniejsze (import, kompilacja, zimny cache, brak danych w pamięci bazy). Zawsze mierz **kilka razy** i bierz medianę lub minimum. Prosty helper:

```python
import statistics
import time
from collections.abc import Callable
from typing import TypeVar

T = TypeVar("T")


def bench(func: Callable[[], T], *, repeat: int = 5) -> tuple[T, float]:
    """Uruchamia `func` `repeat` razy, zwraca wynik i medianę czasu w ms."""
    timings: list[float] = []
    result: T | None = None
    for _ in range(repeat):
        start = time.perf_counter()
        result = func()
        timings.append((time.perf_counter() - start) * 1000)
    assert result is not None
    return result, statistics.median(timings)
```

### Narzędzie 5: `cProfile` i `py-spy` — gdzie znika czas

`cProfile` to wbudowany profiler Pythona. Pokazuje, *które funkcje* zajęły najwięcej czasu:

```bash
python -m cProfile -s cumulative examples/17_slow_report.py | head -30
```

Flaga `-s cumulative` sortuje wyniki po czasie skumulowanym (włącznie z funkcjami wywoływanymi wewnątrz), więc na górze zobaczysz prawdziwych sprawców.

> ⚠️ **Pułapka.** Profilowanie pokaże Ci, że najwięcej czasu zajmuje `psycopg` albo `sqlite3` — bo tam ORM *czeka na bazę*. To nie znaczy, że sterownik jest winny. Znaczy to, że **spędzasz czas w bazie**, i trzeba sprawdzić, *ile* jest tych zapytań i *jakie* są. Dlatego licznik zapytań i `EXPLAIN` są ważniejsze od profilera Pythona przy diagnozie warstwy danych.

`py-spy` to zewnętrzny profiler, który potrafi podłączyć się do **działającego** procesu bez jego zatrzymywania:

```bash
pip install py-spy
py-spy top --pid 12345          # podgląd "na żywo", jak top
py-spy record -o profile.svg --pid 12345 --duration 30
```

To narzędzie ratunkowe na produkcji: zamiast zastanawiać się, „co robi ten proces”, patrzysz na stos wywołań w czasie rzeczywistym.

### Narzędzie 6: `EXPLAIN` — co planuje baza

Najważniejsze narzędzie do diagnozy **pojedynczego zapytania**. Baza nie wykonuje SQL-a dosłownie — najpierw układa *plan*: czytać tabelę sekwencyjnie czy przez indeks, w jakiej kolejności łączyć tabele, jak sortować.

Dla SQLite:

```python
from sqlalchemy import text

with engine.connect() as conn:
    plan = conn.execute(
        text("EXPLAIN QUERY PLAN SELECT * FROM loans WHERE book_id = :bid"),
        {"bid": 42},
    ).all()

for row in plan:
    print(row)
# (2, 0, 0, 'SEARCH loans USING INDEX ix_loans_book_id (book_id=?)')
```

Dla PostgreSQL:

```python
with engine.connect() as conn:
    rows = conn.execute(
        text("EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM loans WHERE book_id = :bid"),
        {"bid": 42},
    ).all()
for row in rows:
    print(row[0])
```

Czytanie planu to umiejętność, którą zdobywa się z czasem. Na początek zapamiętaj dwa słowa-klucze:

| W planie widzisz | Znaczenie | Dobrze czy źle? |
|---|---|---|
| `SEARCH ... USING INDEX` (SQLite) / `Index Scan` (PostgreSQL) | Baza użyła indeksu, żeby trafić w wiersze | ✅ Dobrze |
| `SCAN ...` (SQLite) / `Seq Scan` (PostgreSQL) | Baza czyta całą tabelę od początku do końca | ⚠️ Zależy — czasem to optymalne (mała tabela, czytasz większość wierszy), a czasem to sygnał braku indeksu |
| `USE TEMP B-TREE FOR ORDER BY` (SQLite) / `Sort` (PostgreSQL) | Baza sortuje wyniki „w locie”, bo indeks nie pasuje do `ORDER BY` | ⚠️ Drogo na dużych zbiorach |

> 🧠 **Dlaczego tak jest.** Indeks to jak skorowidz na końcu książki. Bez niego, żeby znaleźć wszystkie wzmianki o „SQLAlchemy”, musisz przekartkować całą książkę (sekundowe skanowanie). Z indeksem idziesz prosto do właściwych stron. Ale skorowidz nie pomoże, jeśli szukasz „wszystkich słów zawierających literę a” — indeks obsługuje konkretne wartości, nie dowolne wzorce. Stąd osobna historia z `LIKE '%...%'`.

### Narzędzie 7: `EXPLAIN ANALYZE` — plan z rzeczywistym czasem

Sam `EXPLAIN` pokazuje *plan*, ale nie mówi, ile trwało. `EXPLAIN ANALYZE` (PostgreSQL) **wykonuje** zapytanie i pokazuje realny czas oraz liczbę przetworzonych wierszy:

```
Seq Scan on loans  (cost=0.00..19424.00 rows=1 width=44) (actual time=89.4..89.4 rows=37 loops=1)
  Filter: (book_id = 42)
  Rows Removed by Filter: 999963
Planning Time: 0.183 ms
Execution Time: 89.512 ms
```

Czytaj to tak: baza przeczytała **milion wierszy**, żeby znaleźć **37**. To ewidentny brak indeksu. Po dodaniu indeksu plan zmieni się w `Index Scan`, a czas spadnie do kilku milisekund.

> ⚠️ **Pułapka.** `EXPLAIN ANALYZE` **wykonuje** zapytanie. Jeśli to `DELETE` albo `UPDATE`, naprawdę usuniesz dane. W PostgreSQL owiń w transakcję i wycofaj:
> ```sql
> BEGIN;
> EXPLAIN (ANALYZE) DELETE FROM loans WHERE ...;
> ROLLBACK;
> ```

### Zasada końcowa: wąskie gardło jest zawsze jedno

Optymalizowanie wszystkiego naraz to strata czasu. W praktyce w 90% przypadków problemem jest **jedna z czterech rzeczy**, opisanych w następnym rozdziale. Znajdź ją, napraw, zmierz ponownie.

---

## Cztery największe dźwignie wydajności

Kolejność ma znaczenie: od największego wpływu do najmniejszego.

### Dźwignia 1: liczba rund do bazy (N+1)

**Runda** (ang. *round trip*) to jedna pełna wymiana zdań z bazą: wysyłam zapytanie → baza wykonuje → dostaję wyniki. Każda runda to narzut sieciowy (albo IPC), parsowanie, planowanie.

> 💡 **Analogia — sklep.** Masz kupić 50 produktów. Wariant A: jedno wejście do sklepu, jedna lista, jedno wyjście. Wariant B: 50 razy wchodzisz do sklepu po jedną rzecz. Robota ta sama, ale w wariancie B spędzasz całą godzinę na samym chodzeniu. W bazie danych „chodzenie” to runda — i ona jest **nieproporcjonalnie droga** w stosunku do pracy.

Złożoność widać wprost: zamiast $O(1)$ rund robisz $O(1 + N)$ rund, gdzie $N$ to liczba obiektów. Przy 1000 książek to 1001 rund zamiast jednej.

Diagnoza i naprawa — w całości w module [`11_ladowanie_i_n_plus_1.md`](11_ladowanie_i_n_plus_1.md). Tutaj dopiszemy tylko sposób weryfikacji w liczbach:

```python
# examples/17_n_plus_1_measure.py
from sqlalchemy import select
from sqlalchemy.orm import Session, selectinload

from models import Book

with Session(engine) as session, count_queries(engine) as stats:
    books = session.scalars(select(Book).limit(200)).all()
    titles = [(b.title, b.author.name) for b in books]

print(f"leniwie:   {stats.summary()}")
# leniwie:   zapytania=201  czas_bazy=612.4 ms

with Session(engine) as session, count_queries(engine) as stats:
    stmt = select(Book).options(selectinload(Book.author)).limit(200)
    books = session.scalars(stmt).all()
    titles = [(b.title, b.author.name) for b in books]

print(f"selectinload: {stats.summary()}")
# selectinload: zapytania=2  czas_bazy=11.7 ms
```

Różnica 201 vs 2 zapytania i ~50× w czasie to typowy wynik na pierwszy rzut oka „niewinnego” kodu.

> 🆕 **SQLAlchemy 2.1** — `selectinload(chunksize=...)`
> Przy bardzo dużych zbiorach `selectinload` generuje `IN (...)` z tysiącami parametrów, co potrafi być problemem dla planera i limitów protokołu. W 2.1 możesz ograniczyć rozmiar porcji:
> ```python
> from sqlalchemy.orm import selectinload
> stmt = select(Book).options(selectinload(Book.author, chunksize=500))
> ```
> Efekt: `selectinload` wykona kilka zapytań `IN (...)` zamiast jednego gigantycznego. To wciąż $O(1)$ rund w sensie liczby „porcji relacji”, a nie „porcji obiektów” — czyli nadal skalowalne.

### Dźwignia 2: ilość przesyłanych danych

Mniej danych na rundę = szybsza runda. Brzmi banalnie, a łamane jest bez przerwy, bo `select(Book)` **zawsze** pobiera wszystkie skolumnizowane atrybuty encji, także te, których nie użyjesz.

```python
from sqlalchemy import select
from sqlalchemy.orm import load_only

# ❌ Pobiera wszystkie kolumny, w tym wielki opis i blob z okładką
stmt = select(Book)

# ✅ Pobiera tylko to, czego potrzebujesz w tym konkretnym miejscu
stmt = select(Book).options(load_only(Book.id, Book.title, Book.isbn))
```

Alternatywnie pobierz nie encje, a same kolumny:

```python
stmt = select(Book.id, Book.title)  # zwraca Row, nie encję
```

### Dźwignia 3: rozmiar transakcji

Długa transakcja trzyma blokady, uniemożliwia VACUUM w PostgreSQL i generuje w bazie bałagan (bloat). Zasada: **transakcja powinna być krótka**. Jeśli w środku transakcji wołasz zewnętrzne API, wysyłasz e-mail albo czekasz na człowieka — transakcja jest za długa.

> ⚠️ **Pułapka.** W aplikacjach webowych sesja domyślnie żyje przez cały request. Jeśli request trwa 3 sekundy (bo w środku jest wywołanie API płatności), to przez 3 sekundy masz otwartą transakcję. Rozwiązanie: podziel przypadki użycia na kilka krótszych transakcji albo wykonaj operacje zewnętrzne poza transakcją, korzystając ze wzorca *outbox* (zapowiedź w [`21_unit_of_work.md`](21_unit_of_work.md)).

### Dźwignia 4: indeksy

Ostatnia, ale nie najmniejsza. Indeks może zmienić sekundowe skanowanie w milisekundowe wyszukiwanie — albo (dodany pochopnie) spowolnić zapisy i zjeść dysk. Osobny rozdział poniżej.

---

## Indeksy: kiedy pomagają, kiedy szkodzą

### Czym jest indeks, naprawdę

> 💡 **Analogia — książka i skorowidz.** Tabela bez indeksu to książka bez skorowidza: żeby znaleźć temat, kartkujesz od pierwszej strony (skan sekwencyjny). Indeks to skorowidz na końcu książki: hasło → numery stron. Znalezienie hasła zajmuje sekundę.
>
> Ale skorowidz ma koszt: trzeba go **utrzymywać**. Gdy do druku wchodzi nowe wydanie z nowym rozdziałem, ktoś musi dopisać strony do skorowidza. W bazie: każdy `INSERT`, `UPDATE` i `DELETE` musi zaktualizować każdy indeks na tej tabeli. Dlatego „indeks na wszystko” to pewny sposób na wolną bazę przy zapisach.

Technicznie: w SQLite i PostgreSQL domyślnym typem indeksu jest B-tree (B-drzewo). Uporządkowana struktura drzewa pozwala na wyszukiwanie w czasie logarytmicznym. Kluczowa konsekwencja: **B-tree obsługuje równość, zakresy (`<`, `>`, `BETWEEN`) i prefiksy, ale nie dowolne wzorce w środku tekstu**.

### Kiedy indeks pomaga

| Scenariusz | Czy indeks pomoże |
|---|---|
| `WHERE book_id = 42` (unikalny lub nie, ale wysoka selektywność) | ✅ Bardzo |
| `WHERE returned_at IS NULL` (mało wierszy spełnia warunek) | ✅ Tak, szczególnie indeks częściowy |
| `WHERE title LIKE 'SQL%'` (wzorzec od początku) | ✅ Tak (prefix) |
| `WHERE title LIKE '%SQL%'` (wzorzec w środku) | ❌ Nie, chyba że `pg_trgm` (PostgreSQL) |
| `ORDER BY created_at DESC LIMIT 20` | ✅ Tak, indeks zgodny z sortowaniem |
| `JOIN ... ON loans.book_id = books.id` | ✅ Tak, indeks na kolumnie łączącej |
| `SELECT * FROM books` (czytasz całą tabelę) | ❌ Nie — skan jest szybszy |

> 🧠 **Dlaczego tak jest.** Indeks przechowuje *klucze* (wartości kolumny) posortowane, a obok wskaźniki do wierszy. Jeśli szukasz konkretnej wartości — idziesz drzewem prosto do niej. Jeśli szukasz `%SQL%`, nie znasz początku wzorca, więc nie wiesz, do którego miejsca drzewa zejść. Musiałbyś przejrzeć **wszystkie** klucze — czyli pracę skanu wykonujesz dodatkowo, z narzutem. Baza to wie i w takim wypadku po prostu wybierze skan.

### Indeks tworzymy w modelu

```python
# examples/17_indexes.py
from datetime import date, datetime

from sqlalchemy import ForeignKey, Index, String, text
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    pass


class Book(Base):
    __tablename__ = "books"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(300))
    isbn: Mapped[str] = mapped_column(String(20), unique=True)  # (1)
    published_year: Mapped[int] = mapped_column(index=True)      # (2)

    loans: Mapped[list["Loan"]] = relationship(back_populates="book")


class Loan(Base):
    __tablename__ = "loans"

    id: Mapped[int] = mapped_column(primary_key=True)
    book_id: Mapped[int] = mapped_column(ForeignKey("books.id"))  # (3)
    lender: Mapped[str] = mapped_column(String(120))
    loaned_at: Mapped[datetime]
    returned_at: Mapped[datetime | None]

    book: Mapped[Book] = relationship(back_populates="loans")

    __table_args__ = (
        # (4) Indeks częściowy: tylko aktywne wypożyczenia
        Index(
            "ix_loans_active",
            "book_id",
            "loaned_at",
            sqlite_where=text("returned_at IS NULL"),
            postgresql_where=text("returned_at IS NULL"),
        ),
        # (5) Indeks złożony — kolejność kolumn ma znaczenie!
        Index("ix_loans_lender_loaned_at", "lender", "loaned_at"),
    )
```

Wyjaśnienia:

1. `unique=True` automatycznie tworzy indeks unikalny — to dobrze, bo ISBN musi być unikalny i często po nim szukamy.
2. `index=True` to skrót dla prostego indeksu jednej kolumny.
3. `ForeignKey` **nie tworzy** indeksu automatycznie w większości baz! W PostgreSQL, SQLite i MySQL musisz dodać indeks ręcznie, jeśli po tej kolumnie filtrujesz lub łączysz. To najczęstsze przeoczenie w projektach.
4. **Indeks częściowy** (partial index) zawiera tylko wiersze spełniające warunek. Jeśli 95% wypożyczeń jest zwróconych, a Ty pytasz tylko o aktywne, indeks częściowy jest ~20× mniejszy i szybszy w utrzymaniu.
5. **Indeks złożony** obejmuje dwie kolumny. Kolejność ma fundamentalne znaczenie — to kolejność, w jakiej wartości są sortowane w drzewie.

> ⚠️ **Pułapka — kolejność kolumn w indeksie złożonym.** Indeks `(lender, loaned_at)` przyspieszy zapytanie `WHERE lender = 'Jan' ORDER BY loaned_at`. Ale **nie** przyspieszy zapytania `WHERE loaned_at > '2026-01-01'` bez warunku na `lender`. Reguła: indeks złożony działa dla prefiksów. To jak książka telefoniczna posortowana najpierw po nazwisku, potem po imieniu — szukanie „wszystkich Janków” przelatuje całą książkę.

### Indeks pokrywający (covering index)

Jeśli zapytanie potrzebuje tylko kolumn, które są w indeksie, baza może odpowiedzieć **wyłącznie z indeksu**, nie zaglądając do tabeli. W PostgreSQL realizuje to `INCLUDE`:

```python
Index(
    "ix_loans_book_covering",
    "book_id",
    postgresql_include=["loaned_at", "returned_at"],
)
```

Dla zapytania `SELECT loaned_at, returned_at FROM loans WHERE book_id = 42` baza nie musi dotykać tabeli — wszystko ma w indeksie. Efekt: mniejszy ruch I/O.

### Indeksy dla `LIKE '%...%'` (tylko PostgreSQL)

B-tree nie pomoże przy wzorcu w środku tekstu. Ale PostgreSQL ma rozszerzenie `pg_trgm`, które indeksuje **trigramy** — trójki znaków. Wtedy `LIKE '%sql%'` staje się wyszukiwalne.

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX ix_books_title_trgm ON books USING gin (title gin_trgm_ops);
```

Od SQLAlchemy:

```python
Index("ix_books_title_trgm", "title", postgresql_using="gin",
      postgresql_ops={"title": "gin_trgm_ops"})
```

To indeks drogi w utrzymaniu, więc stosuj tylko tam, gdzie naprawdę potrzebujesz wyszukiwania pełnotekstowego po fragmencie. W alternatywie rozważ `TSVECTOR` i `to_tsvector` (search wektorowy PostgreSQL) — lepszy do wyszukiwania słów, a nie fragmentów.

### Kiedy indeks szkodzi

- **Zapisy.** Każdy `INSERT` aktualizuje wszystkie indeksy tabeli. Tabela z 8 indeksami wstawia dane znacznie wolniej niż tabela z 2.
- **Dysk.** Indeks to fizyczna struktura na dysku. Duże indeksy tekstowe potrafią być większe niż same tabele.
- **Plan.** Zbyt wiele indeksów myli planer — musi rozważać więcej możliwości i czasem wybiera nietypową, gorszą ścieżkę.
- **Nieużywane indeksy.** Indeks, którego nikt nie używa, to czysty koszt. W PostgreSQL sprawdzisz to przez `pg_stat_user_indexes` (kolumna `idx_scan = 0`).

> 🧠 **Dlaczego tak jest.** Indeks to struktura zoptymalizowana pod **odczyt kosztem zapisu**. Płacisz przy każdym zapisie, żeby zyskać przy odczycie. Jeśli zapisujesz 100× więcej niż czytasz (np. tabela zdarzeń, logi), indeksy mogą być kontrproduktywne.

### Pomiar: przed i po

```python
# examples/17_index_experiment.py
from sqlalchemy import BigInteger, Index, String, create_engine, insert, select, text
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class BigRow(Base):
    __tablename__ = "big_rows"

    id: Mapped[int] = mapped_column(BigInteger, primary_key=True)
    title: Mapped[str] = mapped_column(String(300))


engine = create_engine("sqlite:///perf.sqlite")
Base.metadata.create_all(engine)

# 1. Wygeneruj 100 000 wierszy w porcjach
with engine.begin() as conn:
    chunk = 10_000
    for start in range(0, 100_000, chunk):
        conn.execute(
            insert(BigRow),
            [{"id": i, "title": f"Książka numer {i % 5_000}"} for i in range(start, start + chunk)],
        )

# 2. Zmierz zapytanie BEZ indeksu
with engine.begin() as conn:
    conn.execute(text("ANALYZE"))  # (SQLite) odśwież statystyki planera
    plan = conn.execute(text("EXPLAIN QUERY PLAN SELECT id FROM big_rows WHERE title = 'Książka numer 42'")).all()
    print("PLAN PRZED:", plan[0][-1])
    # PLAN PRZED: SCAN big_rows

# 3. Utwórz indeks
with engine.begin() as conn:
    conn.execute(text("CREATE INDEX ix_big_rows_title ON big_rows (title)"))

# 4. Zmierz ponownie
with engine.begin() as conn:
    conn.execute(text("ANALYZE"))
    plan = conn.execute(text("EXPLAIN QUERY PLAN SELECT id FROM big_rows WHERE title = 'Książka numer 42'")).all()
    print("PLAN PO:    ", plan[0][-1])
    # PLAN PO:     SEARCH big_rows USING INDEX ix_big_rows_title (title=?)
```

Na 100 000 wierszy i skromnym sprzęcie typowy wynik wygląda tak:

| Wariant | Czas (mediana) | Plan |
|---|---|---|
| Bez indeksu | 28–35 ms | `SCAN big_rows` |
| Z indeksem | 0.05–0.15 ms | `SEARCH ... USING INDEX` |

To poprawa rzędu **200–600×**. I to nie jest oszustwo — po prostu skanowanie 100 000 wierszy w Pythonie vs skok w B-drzewie o 17 poziomach to fundamentalnie różne operacje.

---

## Wstawianie masowe: cztery strategie

Wstawianie dużej liczby wierszy to miejsce, gdzie naiwne podejście ORM-owe kompromituje się najbardziej. Porównamy cztery strategie.

### Strategia 1: `session.add_all()` — „czysty ORM”

```python
from sqlalchemy.orm import Session

ROWS = 100_000

with Session(engine) as session:
    session.add_all(
        [Book(id=i, title=f"Book {i}", isbn=f"ISBN-{i:08d}") for i in range(ROWS)]
    )
    session.commit()
```

Co się dzieje: SQLAlchemy tworzy **100 000 obiektów Pythona** w pamięci, wpisuje je do identity map, sortuje operacje, a potem wstawia, używając mechanizmu `insertmanyvalues`. Narzut: pamięć (~100 MB+), czas tworzenia obiektów, potencjalnie dziesiątki tysięcy wierszy w `flush` na raz.

Zaleta: dostajesz gotowe encje (z `id`), masz kaskady, eventy `before_insert`, całą maszynerię ORM.

### Strategia 2: `insert().values([...])` — jedno wielkie `INSERT`

```python
from sqlalchemy import insert

rows = [{"id": i, "title": f"Book {i}", "isbn": f"ISBN-{i:08d}"} for i in range(ROWS)]

with engine.begin() as conn:
    conn.execute(insert(Book), rows)
```

To **executemany** z listą słowników. SQLAlchemy w wersji 2.0 kompiluje to do postaci `INSERT ... VALUES (...), (...), (...), ...` z wieloma zestawami parametrów, używając mechanizmu **`insertmanyvalues`**. W porównaniu do strategii 1: brak tworzenia encji, brak identity map, brak eventów ORM — czyli dużo mniej pracy w Pythonie.

### Strategia 3: `insertmanyvalues_page_size` — kontrola porcji

```python
with engine.begin() as conn:
    conn.execute(
        insert(Book),
        rows,
        execution_options={"insertmanyvalues_page_size": 5000},
    )
```

Domyślnie SQLAlchemy dobiera rozmiar porcji. Możesz go wymusić:

- `insertmanyvalues_page_size` — ile wierszy w jednym `INSERT`.
- `insertmanyvalues_max_parameters` — górny limit liczby parametrów w zapytaniu.

Dlaczego porcje mają znaczenie: **za małe** (np. 10) to tysiące rund, **za duże** trafiają w limity parametrów dialektu (SQLite ma limit zmiennych w zapytaniu, PostgreSQL limit argumentów w protokole).

> 🔬 **Pod maską.** Dla 100 000 wierszy i `page_size=1000` SQLAlchemy wykona **100 zapytań** zamiast 100 000. Widać to doskonale w liczniku:

```python
with engine.begin() as conn, count_queries(engine) as stats:
    conn.execute(insert(Book), rows, execution_options={"insertmanyvalues_page_size": 1000})
print(stats.summary())
# zapytania=100  czas_bazy=... ms
```

### Strategia 4: `COPY` (PostgreSQL) — omijanie SQL-a

PostgreSQL ma dedykowany protokół masowego ładowania: `COPY`. Nie przechodzi przez parser SQL-a, nie buduje planu dla każdego wiersza — po prostu sypie bajty do tabeli. SQLAlchemy **nie ma** API COPY; sięgasz po surowe połączenie sterownika (psycopg3):

```python
# examples/17_copy_postgres.py
# Wymaga: pip install "sqlalchemy[postgresql]"  (psycopg3)
from sqlalchemy import create_engine

engine = create_engine("postgresql+psycopg://user:pass@localhost/db")

rows = ((i, f"Book {i}", f"ISBN-{i:08d}") for i in range(100_000))

with engine.begin() as conn:
    raw = conn.connection.driver_connection  # prawdziwe połączenie psycopg
    with raw.cursor().copy("COPY books (id, title, isbn) FROM STDIN") as copy:
        for row in rows:
            copy.write_row(row)
```

> ⚠️ **Pułapka.** To kod **specyficzny dla PostgreSQL i psycopg3**. Omija SQLAlchemy całkowicie: nie ma `insertmanyvalues`, nie ma eventów, nie ma mapowania typów, nie ma `default` z Pythona. Za to jest najszybszy z możliwych.

### Tabela porównawcza (100 000 wierszy, SQLite, laptop klasy „developer”)

| Strategia | Czas | Zapytania | Zużycie pamięci Pythona | Uwagi |
|---|---|---|---|---|
| `session.add_all()` | ~4.5 s | 100 (porcjowane) | ~120 MB | Najprostsze, najwolniejsze, ale masz encje i eventy |
| `insert().values([...])` / `execute(insert, rows)` | ~1.2 s | 100 | ~25 MB | Dobry kompromis; brak obiektów ORM |
| Jak wyżej, `page_size=500` | ~1.5 s | 200 | ~25 MB | Mniej w jednym zapytaniu = więcej rund |
| Jak wyżej, `page_size=10000` | ~1.1 s | 10 | ~25 MB | Uważaj na limity parametrów |
| `COPY` (PostgreSQL) | ~0.4 s | 1 | generator (stała) | Tylko PostgreSQL, omija ORM |

> 💡 Te liczby są **orientacyjne** — Twój sprzęt, rozmiar wiersza i dostępne indeksy je zmienią. Wnioski do zapamiętania: (a) `add_all` jest najdroższe przy dużych zbiorach, (b) `insertmanyvalues` z rozsądnym `page_size` daje rząd wielkości poprawy, (c) `COPY` jest niedoścignione w PostgreSQL.

> 🧠 **Dlaczego `add_all` jest wolne.** Trzy powody naraz: tworzenie obiektów Pythona, wstawianie ich do identity map (która musi wykrywać duplikaty po kluczu), oraz sortowanie przez Unit of Work. Każdy z tych kroków wykona się 100 000 razy. `insert().values([...])` nie tworzy obiektów — pracuje na słownikach, czyli na wbudowanej prymitywnej strukturze.

> 🧪 **Ćwiczenie.** Dla tych samych 100 000 wierszy zmień `page_size` na `1`, `10`, `100`, `1000`, `10000`. Zapisz tabelę (page_size → czas, liczba zapytań). Zaobserwuj minimum i wyjaśnij je.

---

## Masowa aktualizacja i usuwanie

Ta sama zasada co przy insercie: **nie ładuj encji do pamięci, żeby je zmienić**. Pętla po obiektach w ORM to przepis na N+1 przy zapisie.

### Źle: pętla po obiektach

```python
from sqlalchemy import select
from sqlalchemy.orm import Session

with Session(engine) as session:
    for book in session.scalars(select(Book).where(Book.published_year < 1990)):
        book.title = book.title.upper()  # jedna zmiana = jeden UPDATE na commit
    session.commit()
```

Przy 50 000 starych książek: jedno `SELECT` i 50 000 `UPDATE`-ów. Czas: dziesiątki sekund.

### Dobrze: jedno `UPDATE ... WHERE`

```python
from sqlalchemy import update

with Session(engine) as session:
    result = session.execute(
        update(Book)
        .where(Book.published_year < 1990)
        .values(title=func.upper(Book.title))
    )
    session.commit()
    print(result.rowcount)  # np. 50_000
```

Baza robi to sama, jednym zapytaniem, w swoim tempie. Zamiast 50 000 rund — jedna.

### `synchronize_session` — co z obiektami już w sesji

Jeśli w tej samej sesji masz już wczytane obiekty `Book`, a wykonujesz ORM-owe `update()` przez `session.execute()`, to SQLAlchemy nie wie o zmianie w bazie. Domyślnie w 2.0 metoda ustala strategię `synchronize_session`:

| Wartość | Znaczenie | Koszt |
|---|---|---|
| `'auto'` | SQLAlchemy wybiera `'evaluate'` lub `'fetch'` zależnie od wyrażenia | Zależny |
| `'evaluate'` | SQLAlchemy próbuje ocenić warunek i zmodyfikować obiekty w pamięci | Tanie, ale nie dla każdego wyrażenia |
| `'fetch'` | Dodatkowe `SELECT` po `id` pasujących wierszy, potem aktualizacja obiektów w pamięci | Drogie (dodatkowa runda i lista id!) |
| `False` | Nie synchronizuj — obiekty w pamięci zostaną **nieaktualne** | Zero kosztu |

> ⚠️ **Pułapka.** Gdy robisz masowy `UPDATE` na milionie wierszy, `synchronize_session='fetch'` wykona `SELECT id FROM ...` po milionie identyfikatorów — czyli dokładnie to, czego chciałeś uniknąć. Wtedy użyj `synchronize_session=False` i zadbaj, żeby nie polegać na obiektach z tej samej sesji:

```python
session.execute(
    update(Book).where(Book.published_year < 1990).values(...),
    execution_options={"synchronize_session": False},
)
```

Alternatywnie natychmiast po masowej aktualizacji wykonaj `session.expire_all()` — wszelkie dalsze odczyty pobiorą świeże dane z bazy (za cenę zapytań).

### Usuwanie

```python
from sqlalchemy import delete

with Session(engine) as session:
    session.execute(delete(Loan).where(Loan.returned_at < date(2020, 1, 1)))
    session.commit()
```

Usuwanie pętlą po obiektach ma jeszcze jeden problem: kaskady Pythonowe (`delete-orphan`) i eventy `before_delete` wykonają dodatkową robotę na każdym obiekcie. Masowe `delete()` w SQL omija wszystko — usunięcie 1 miliona wierszy kosztuje bazy tyle, co jeden `DELETE`.

> 🧠 **Dlaczego masowe operacje omijają ORM.** ORM to *warstwa*, która zamienia pracę na wierszach na pracę na obiektach. Ta warstwa jest użyteczna przy pojedynczych operacjach z logiką biznesową. Przy operacjach „na całym zbiorze” jest zbędnym pośrednikiem: nie potrzebujesz 1 000 000 obiektów Pythona, żeby zmienić nazwę w 1 000 000 rzędów bazy.

> 🧪 **Ćwiczenie.** Zmierz czas aktualizacji 50 000 wierszy w trzech wariantach: (a) pętla po obiektach, (b) `session.execute(update(...))` z domyślnym `synchronize_session`, (c) to samo z `synchronize_session=False`. Zanotuj liczbę zapytań w każdym.

---

## Odczyty dużych zbiorów: `all()` vs strumień

### Problem pamięci

`session.scalars(select(Book)).all()` materializuje **wszystkie** wyniki w liście obiektów Pythona jednocześnie. Przy 1 000 000 wierszy to gigabajty pamięci i realne ryzyko `MemoryError`.

### Strumieniowanie: `yield_per`

```python
from sqlalchemy import select
from sqlalchemy.orm import Session

with Session(engine) as session:
    result = session.scalars(
        select(Book),
        execution_options={"yield_per": 1000},
    )
    for book in result:
        process(book)
```

`yield_per` mówi sterownikowi: „pobieraj wiersze w porcjach po 1000”. W efekcie biblioteka klienta nie trzyma całego zbioru w pamięci — zużycie pamięci staje się stałe, niezależne od liczby wierszy.

Można też ustawić na obiekcie `Result`:

```python
result = session.scalars(select(Book))
for book in result.yield_per(1000):
    process(book)
```

### `stream()` — jeszcze bardziej eksplicytne

```python
with Session(engine) as session:
    for book in session.stream_scalars(select(Book), execution_options={"yield_per": 500}):
        process(book)
```

`session.stream()` zwraca `Result` w trybie strumieniowym, a `session.stream_scalars()` — strumień skalarów. Semantycznie to to samo co `yield_per`, ale nazwa wprost komunikuje intencję czytelnikowi kodu.

### `partitions()` — porcje jako listy

Czasem chcesz pracować w porcjach, ale mieć każdą porcję jako **listę** (np. do `executemany` po przetworzeniu):

```python
result = session.scalars(select(Book))

for partition in result.partitions(1000):
    # partition to zwykła lista obiektów, maksymalnie 1000 elementów
    process_batch(partition)
```

### Pomiar pamięci przez `tracemalloc`

```python
# examples/17_stream_memory.py
import tracemalloc
from sqlalchemy import select
from sqlalchemy.orm import Session

from models import Book


def measure_all() -> int:
    tracemalloc.start()
    with Session(engine) as session:
        rows = session.scalars(select(Book)).all()
    current, peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()
    print(f"all():   obiektów={len(rows)}  szczyt pamięci={peak / 1024 / 1024:.1f} MB")
    return peak


def measure_stream() -> int:
    tracemalloc.start()
    seen = 0
    with Session(engine) as session:
        for _ in session.scalars(select(Book), execution_options={"yield_per": 1000}):
            seen += 1
    _, peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()
    print(f"stream:  obiektów={seen}  szczyt pamięci={peak / 1024 / 1024:.1f} MB")
    return peak
```

Typowy wynik dla 1 000 000 wierszy i encji z 6 kolumnami:

| Metoda | Szczyt pamięci | Uwagi |
|---|---|---|
| `scalars().all()` | ~1 200 MB | Wszystko w jednej liście jednocześnie |
| `yield_per(1000)` | ~5 MB | Stała pamięć, niezależnie od rozmiaru zbioru |

> ⚠️ **Pułapka — `yield_per` z eager loadingiem kolekcji.** `yield_per` działa bezpiecznie z relacjami ładowanymi przez `selectinload` (bo to osobne zapytania) oraz z relacjami skalarowymi przez `joinedload`. Ale **`joinedload` na kolekcji** przy `yield_per` to konflikt: SQLAlchemy musi złożyć obiekty z jednego zbioru wierszy, a strumieniowanie tego nie robi. Zobaczysz ostrzeżenie albo błąd. Rozwiązanie: użyj `selectinload` dla kolekcji.

> 🧠 **Dlaczego `yield_per` nie jest darmowe.** Wymaga kursora serwerowego (PostgreSQL: *named cursor*, `DECLARE ... CURSOR`) albo porcjowania po stronie klienta. To dodatkowa złożoność i (w PostgreSQL) ograniczenie: kursor trzyma transakcję otwartą przez cały czas iteracji. Przy długich pętlach z przerwami na przetwarzanie blokujesz zasoby bazy.

> 🧪 **Ćwiczenie.** Porównaj na 300 000 wierszy: (a) `scalars().all()` + `len()`, (b) `for row in session.scalars(stmt, execution_options={"yield_per": 1000}): n += 1`, (c) `for part in result.partitions(1000): n += len(part)`. Zapisz czas i szczyt pamięci.

> 🆕 **SQLAlchemy 2.1** — `yield_per` a `selectinload(chunksize)`
> Oba mechanizmy można połączyć. `chunksize` ogranicza porcję zapytania `IN (...)`, a `yield_per` porcję strumienia. W razem dają stałą pamięć klienta i przewidywalne zapytania relacyjne.

---

## Cache kompilacji SQL

### Ile kosztuje kompilacja

Każde `select(Book).where(...)` musi zostać zamienione na SQL — i to SQL **specyficzny dla dialektu** (`?` vs `$1`, `LIMIT` vs `FETCH FIRST`). To nie jest darmowe: parsowanie wyrażeń, generowanie klucza, budowa drzewa SQL, konwersja na tekst.

SQLAlchemy rozwiązuje to przez **cache skompilowanych zapytań**: pierwsze wykonanie konkretnej *struktury* zapytania kompiluje SQL, a wynik jest zapamiętywany pod kluczem zależnym od struktury. Kolejne wykonania tej samej struktury (z różnymi parametrami) korzystają z cache.

```python
from sqlalchemy import select

# Struktura jest identyczna dla wszystkich wywołań — kompiluje się RAZ
stmt = select(Book).where(Book.published_year > 2000)
session.scalars(stmt.limit(10)).all()   # kompilacja + wykonanie
session.scalars(stmt.limit(20)).all()   # cache hit — inna struktura (limit), więc nowy klucz
```

### `query_cache_size`

Rozmiar cache ustawiasz przy tworzeniu silnika:

```python
engine = create_engine("postgresql+psycopg://...", query_cache_size=1000)
```

Domyślnie 500. Zwiększenie pomaga aplikacjom z wieloma różnymi zapytaniami, ale cache pamięta każdą strukturę zapytania — więc nie przesadzaj z rozmiarem.

> 🔬 **Pod maską.** Klucz cache jest budowany z **struktury** zapytania (rodzaj wyrażenia, kolumny, operatory, typy), a nie z wartości. Dlatego `where(Book.id == 1)` i `where(Book.id == 2)` dzielą ten sam wpis cache — różnią się tylko wartościami przekazywanymi jako parametry.

### `cache_ok` — własne typy i cache

Jeśli tworzysz własny `TypeDecorator` (moduł [`12_typy_i_wlasne_typy.md`](12_typy_i_wlasne_typy.md)), musisz jawnie powiedzieć SQLAlchemy, czy jego kompilacja jest deterministyczna:

```python
from sqlalchemy import String
from sqlalchemy.types import TypeDecorator


class UpperString(TypeDecorator[str]):
    """Zapisuje tekst zawsze wielkimi literami."""

    impl = String
    cache_ok = True  # (1)

    def process_bind_param(self, value: str | None, dialect) -> str | None:
        return value.upper() if value is not None else None

    def process_result_value(self, value: str | None, dialect) -> str | None:
        return value
```

1. `cache_ok = True` oznacza: „wiem, że mój typ kompiluje się w sposób zależny wyłącznie od swojej definicji i wartości, więc cache kluczy jest bezpieczny”. Bez tego SQLAlchemy wypisze ostrzeżenie i **wyłączy** efektywne cache'owanie dla zapytań z tym typem.

> ⚠️ **Pułapka.** Jeśli Twój `TypeDecorator` zachowuje się różnie w zależności od stanu zewnętrznego (np. konfiguracji, czasu, wartości z ENV), ustawienie `cache_ok = True` jest **błędem** i doprowadzi do użycia błędnego SQL z cache. Wtedy ustaw `cache_ok = False` i zaakceptuj koszt kompilacji, albo przeprojektuj typ.

> 🆕 **SQLAlchemy 2.1** — szybsze generowanie kluczy cache
> Generowanie klucza kompilacji zostało zoptymalizowane w 2.1. W benchmarkach powtarzalnych zapytań widać kilkuprocentowy do kilkunastoprocentowy zysk CPU, co przy tysiącach zapytań na sekundę przekłada się na mniejszą liczbę rdzeni potrzebnych do obsługi aplikacji. Jeśli jesteś na 2.0 i widzisz w profilu duży udział `sqlalchemy.sql.compiler`, to jest właśnie ten obszar.

### Cache a parametry: nie cache'uj wartości w strukturze

Najgorszy antywzorzorzec wydajnościowy to wysyłanie **literałów** zamiast parametrów:

```python
# ❌ Każda wartość tworzy NOWĄ strukturę zapytania — cache bez sensu,
#     a przy okazji otwarta furtka na SQL Injection
stmt = text(f"SELECT * FROM books WHERE title = '{user_input}'")

# ✅ Jedna struktura, wartość jako parametr — cache działa, wstrzyknięcie niemożliwe
stmt = text("SELECT * FROM books WHERE title = :title")
```

W ORM-ie jest to trudniejsze do zrobienia niechcący, ale zdarzy się przy `literal_column()`, dynamicznie generowanych nazwach kolumn i przy surowym SQL-u. Zawsze: **struktura statyczna, wartości jako `bindparam`**.

---

## Sesja, `expire_on_commit` i rosnąca identity map

### Koszt `expire_on_commit=True`

Domyślnie sesja działa tak: po `commit()` **wygasa** (expire) wszystkie obiekty w identity map. Oznacza to, że pierwszy dostęp do atrybutu po commit **wykona zapytanie** do bazy.

```python
from sqlalchemy import select
from sqlalchemy.orm import Session

with Session(engine) as session:
    books = session.scalars(select(Book).limit(50)).all()
    session.commit()  # ← tu wszystkie 50 obiektów "wygasa"

    for book in books:
        print(book.title)  # ← 50 dodatkowych zapytań! Jedno na książkę.
```

> 🔬 **Pod maską.** Licznik pokaże `1 + 50 = 51` zapytań, mimo że nie zmieniliśmy ani nie dodaliśmy nic, co wpłynęłoby na tytuły. Wszystkie te zapytania to skutek `expire_on_commit=True`.

Dlaczego to w ogóle domyślne? Bo jest **bezpieczne**: po commicie widzisz świeże dane z bazy, uwzględniające `server_default`, triggery i zmiany z innych transakcji. To dobra domyślna wartość.

Kiedy wyłączyć:

```python
Session(engine, expire_on_commit=False)
```

- W aplikacjach, gdzie po commicie natychmiast serializujesz obiekty do odpowiedzi API (typowe w FastAPI).
- W async (tam `expire_on_commit=True` praktycznie uniemożliwia pracę — patrz [`15_asynchronicznosc.md`](15_asynchronicznosc.md)).

> ⚠️ **Pułapka.** `expire_on_commit=False` oznacza, że obiekty w pamięci mogą być **nieaktualne** względem bazy. Jeśli po commicie odczytujesz pola wyliczane przez bazę (`server_default`, kolumna generowana), dostaniesz starą wartość. Rozwiązanie: `session.refresh(obj)` na tych konkretnych obiektach.

### Długość życia sesji a identity map

Identity map to słownik „klucz → obiekt” trzymany przez sesję. Dla sesji żyjącej przez cały request to świetne — powtórne odczyty tego samego wiersza nie generują zapytań.

Ale identity map **rośnie** i **pamięta obiekty**. Długa sesja (CLI przetwarzający milion wierszy, worker zadań) zatrzyma wszystkie wczytane obiekty w pamięci i będzie wolniejsza z każdą iteracją.

Rozwiązania:

```python
# 1. Okresowo czyść identity map
session.expunge_all()  # usuwa wszystkie obiekty z sesji (bez zapisu)

# 2. Zamykaj i otwieraj sesję dla partii wierszy
for batch_start in range(0, total, 1000):
    with Session(engine) as session:
        rows = session.scalars(select(Book).offset(batch_start).limit(1000)).all()
        process(rows)

# 3. Strumieniowanie BEZ identity map — czysty odczyt kolumn
for row in session.execute(select(Book.id, Book.title)).partitions(1000):
    process(row)
```

> 🧠 **Dlaczego sesja krócej żyjąca jest lepsza przy przetwarzaniu masowym.** Każdy obiekt w identity map to wpis w słowniku i referencja trzymająca cały graf. Sesja „żyjąca cały dzień” w workerze to wyciek pamięci z opóźnieniem. Wzorzec: *sesja per jednostka pracy* (per request, per zadanie, per partia) — dokładnie to, co omawia [`21_unit_of_work.md`](21_unit_of_work.md).

### `autoflush=False` — kiedy pomaga w profilowaniu

Autoflush to automatyczne wysłanie oczekujących zmian przed każdym zapytaniem:

```python
with Session(engine) as session:
    session.add(Book(title="Nowa"))
    session.scalars(select(Book))  # ← autoflush wysłał INSERT zanim wykonał SELECT
```

To jest zaskakujące przy debugowaniu, bo widzisz w logu `INSERT`, którego nie wywołałeś. Wyłączenie autoflush pomaga zrozumieć przepływ:

```python
with Session(engine, autoflush=False) as session:
    session.add(Book(title="Nowa"))
    session.scalars(select(Book))  # INSERT jeszcze nie poszedł — SELECT go nie widzi!
```

> ⚠️ **Pułapka.** `autoflush=False` jest użyteczne w diagnostyce, ale w kodzie produkcyjnym potrafi wywołać subtelne błędy: dodajesz obiekt i natychmiast go wyszukujesz — i nie znajdujesz, bo flush jeszcze nie zaszedł. Jeśli wyłączasz autoflush, rób to świadomie, np. w skryptach wsadowych, i pamiętaj o jawnym `session.flush()` tam, gdzie go potrzebujesz.

> 🆕 **SQLAlchemy 2.1** — autoflush działa bezwarunkowo
> W 2.1 semantyka autoflush została uproszczona: w miejscach, gdzie wcześniej SQLAlchemy mógł warunkowo pominąć autoflush (na przykład w pewnych ścieżkach rekurencyjnych), teraz zachowanie jest jednolite. Jeżeli w Twoim kodzie polegałeś na „tu się nie zflushuje”, po aktualizacji zachowanie może się zmienić. Testy z licznikiem zapytań wykryją to natychmiast — to kolejny argument, żeby taki licznik mieć.

---

## Wzorce zapytań: mniej danych, mniej rund

### Nie pobieraj `SELECT *`

ORM-owe `select(Book)` odpowiada `SELECT books.id, books.title, books.isbn, ...` — czyli pobiera **wszystkie zmapowane kolumny**. Jeśli w tabeli jest `description` będący 20 kB tekstu, a Ty potrzebujesz tylko tytułu do listy rozwijanej, to pobierasz megabajty bez potrzeby.

```python
from sqlalchemy import select
from sqlalchemy.orm import load_only

# Lista do wyboru — potrzebujesz 2 kolumn z 9
stmt = select(Book).options(load_only(Book.id, Book.title))
```

> ⚠️ **Pułapka.** `load_only` oznacza, że **każdy dostęp** do pominiętego atrybutu spowoduje dodatkowe zapytanie (lazy load pojedynczej kolumny). Jeśli więc pomijasz `Book.isbn`, a potem w pętli odczytujesz `book.isbn`, zbudowałeś sobie N+1 ręcznie. Reguła: `load_only` na listach, gdzie naprawdę nie potrzebujesz reszty — i `raiseload` na resztę, żeby mieć pewność:

```python
from sqlalchemy.orm import load_only, raiseload

stmt = select(Book).options(
    load_only(Book.id, Book.title),
    raiseload("*"),  # każdy dostęp do niezaładowanego atrybutu = wyjątek
)
```

### Wybieraj kolumny zamiast encji w raportach

Do raportów nie potrzebujesz obiektów ORM. Potrzebujesz danych:

```python
# ❌ Ładuje pełne encje, żeby odczytać dwa pola
rows = session.scalars(select(Book)).all()
report = [(b.title, b.published_year) for b in rows]

# ✅ Baza zwraca tylko to, czego potrzebuje raport
stmt = select(Book.title, Book.published_year).where(Book.published_year > 2000)
for title, year in session.execute(stmt):
    ...
```

Zalety: mniej danych przez sieć, brak identity map, brak instancjonowania encji, brak lazy loadów.

### Agreguj w bazie, nie w Pythonie

```python
# ❌ Baza wysyła 100 000 wierszy, żeby Python policzył sumę
total = sum(loan.fee for loan in session.scalars(select(Loan)))

# ✅ Baza liczy sumę, wysyła jedną liczbę
from sqlalchemy import func

total = session.scalar(select(func.coalesce(func.sum(Loan.fee), 0)))
```

Różnica w transferze: kilobajty vs bajty. Różnica w pamięci: megabajty vs nic. Różnica w czasie: sekundy vs milisekundy.

> 🧠 **Dlaczego tak jest.** Bazy danych są **bardzo** dobre w agregacjach. To ich podstawowa kompetencja od pięćdziesięciu lat. Python jest do agregacji fatalny: musi zbuforować wszystko, utworzyć N obiektów, iterować w interpreterze. Przenoszenie pracy agregacyjnej do Pythona to jak zamawianie w hurtowni 100 000 jajek, żeby zliczyć, ile ważą.

### Nie powtarzaj zapytań w pętli o niezmienną wartość

```python
# ❌ To samo zapytanie 1000 razy, wynik zawsze ten sam
for book in books:
    category = session.scalar(
        select(Category).where(Category.id == book.category_id)
    )

# ✅ Jedno zapytanie z IN + słownik w pamięci
category_ids = {b.category_id for b in books}
categories = {
    c.id: c
    for c in session.scalars(select(Category).where(Category.id.in_(category_ids)))
}
for book in books:
    category = categories[book.category_id]
```

To ten sam wzorzec co N+1, ale na poziomie logiki aplikacji — i dokładnie tak samo kosztowny.

### Paginacja: `OFFSET` a keyset

```python
# Klasyczna paginacja OFFSET/LIMIT
stmt = select(Book).order_by(Book.id).offset(page * size).limit(size)
```

`OFFSET 500000 LIMIT 20` działa tak: baza czyta i **odrzuca** 500 000 wierszy, żeby zwrócić 20. Dla głębokich stron koszt rośnie liniowo z numerem strony.

Keyset pagination (kursorowe) idzie inaczej:

```python
# Następna porcja: wiersze o id większym od ostatnio widzianego
last_id = 500_000
stmt = select(Book).where(Book.id > last_id).order_by(Book.id).limit(size)
```

Baza wchodzi w indeks na `id` i zwraca od razu kolejne 20. Koszt stały niezależnie od głębokości.

| Metoda | Złożoność strony $k$ | Kolejność po zmianach danych |
|---|---|---|
| `OFFSET` | $O(k \cdot page\_size)$ | Stabilna, ale łatwo pominąć wiersze przy równoległych zmianach |
| Keyset | $O(page\_size)$ | Wymaga unikalnego klucza sortowania; odporna na zmiany |

> 🧠 **Dlaczego `OFFSET` jest tak drogi.** Baza musi *fizycznie przeczytać* wiersze, które odrzuca. Nie ma sposobu, żeby „pominąć” 500 000 wierszy bez ich policzenia — chyba że istnieje kursor, który pamięta, gdzie skończyliśmy. Keyset to właśnie taki kursor, tylko realizowany przez wartość klucza. Więcej o implementacji w module [`20_repository.md`](20_repository.md).

> 🧪 **Ćwiczenie.** Na tabeli 1 000 000 wierszy zmierz czas zapytań `LIMIT 20 OFFSET {0, 10000, 100000, 500000}` i zapisz tabelę. Następnie zmierz keyset pagination z tego samego „miejsca” w danych. Wyciągnij wniosek o kształcie krzywej czasu.

---

## Konfiguracja połączeń i sieć

### Wielkość puli połączeń

Pula (connection pool) to zbiór gotowych połączeń do bazy, które aplikacja wypożycza i zwraca. Tworzenie nowego połączenia (TCP, TLS, autoryzacja) trwa dziesiątki milisekund. Pula ukrywa ten koszt.

```python
from sqlalchemy import create_engine

engine = create_engine(
    "postgresql+psycopg://user:pass@localhost/db",
    pool_size=10,          # (1) bazowa liczba połączeń trzymanych w puli
    max_overflow=20,       # (2) ile dodatkowych można otworzyć ponad pool_size
    pool_timeout=30,       # (3) ile sekund czekać na wolne połączenie
    pool_recycle=1800,     # (4) po ilu sekundach odtworzyć połączenie
    pool_pre_ping=True,    # (5) sprawdź połączenie przed wydaniem
)
```

1. **`pool_size`** — liczba połączeń utrzymywanych na stałe. Dobór zależy od tego, ile równoległych requestów obsługuje aplikacja i ile równoległych połączeń zniesie baza. PostgreSQL domyślnie ma `max_connections=100` — nie chcesz wyczerpać tego jednym procesem, jeśli masz kilka instancji aplikacji.
2. **`max_overflow`** — pozwolenie na chwilowe przekroczenie bazowego rozmiaru. Połączenia „nadmiarowe” są zamykane przy zwrotach. Ustaw na rozsądną wartość, żeby skoki ruchu nie blokowały requestów, ale żeby baza nie została zalana.
3. **`pool_timeout`** — jeśli pula jest wyczerpana, klient czeka. Po tym czasie dostaniesz `TimeoutError`. Jeśli widzisz ten błąd, pulę masz za małą albo zapytania za długie.
4. **`pool_recycle`** — stare połączenia bywają zamykane przez firewall albo bazę (timeout bezczynności). Recykling zapobiega użyciu „martwego” połączenia.
5. **`pool_pre_ping`** — przed wydaniem połączenia klient wykonuje `SELECT 1` (albo równoważne). Kosztem minimalnego opóźnienia zyskujesz pewność, że połączenie żyje. To złoty standard w środowiskach z NAT-em, load balancerami i firewallami.

> 💡 **Analogia — wypożyczalnia rowerów.** Pula to rząd rowerów na stojaku. Klient przychodzi, bierze rower, wraca po przejażdżce — nie produkujesz nowego roweru przy każdym wynajmie. `pool_size` to liczba rowerów na stojaku, `max_overflow` to rowery na zapleczu (droższe w użyciu), `pool_timeout` to czas, ile klient poczeka, zanim się obrazi i pójdzie, a `pool_recycle` to przegląd techniczny, który wymienia rower, jeśli stał zbyt długo bez użycia.

### Diagnostyka puli

```python
print(engine.pool.status())
# np.: Pool size: 10  Connections in pool: 3  Current Overflow: -7  Current Checked out: 0
```

Ta linia ratuje zespół, gdy pojawia się „aplikacja zawiesza się pod obciążeniem”. Jeśli `Current Checked out` stale zbliża się do `pool_size + max_overflow`, masz wyciek połączeń (kod nie zwraca sesji!) albo zapytania trwają zbyt długo.

> ⚠️ **Pułapka.** Wyciek połączeń to klasyk: ktoś utworzył `Session()` i zapomniał `close()`. Wtedy pula stopniowo się wyczerpuje, po kilkudziesięciu minutach aplikacja „zawiesza się bez powodu”, a restart pomaga. Zawsze używaj sesji w `with Session(engine) as session:`.

### Prepared statements

Prepared statement to zapytanie sparsowane i zaplanowane **raz**, a potem wielokrotnie wykonane z różnymi parametrami. Oszczędza parsowanie i planowanie.

Dla psycopg3 (PostgreSQL):

```python
engine = create_engine(
    "postgresql+psycopg://user:pass@localhost/db",
    connect_args={"prepare_threshold": 2},  # po 2 wykonaniach zpreparuj zapytanie
)
```

W `asyncpg` prepared statements są tworzone automatycznie.

> ⚠️ **Pułapka — PgBouncer.** Jeśli łączysz się z PostgreSQL przez PgBouncer w trybie **transaction pooling**, prepared statements mogą „przeskakiwać” między połączeniami i kończyć się błędem albo niepoprawnym zachowaniem. Rozwiązanie: wyłącz prepared statements po stronie klienta (`prepare_threshold=None` w psycopg3) albo użyj trybu *session pooling*. To jest naprawdę częsta, bolesna niespodzianka przy wdrożeniach.

### Sieć: rundy vs szerokość

Dwie skrajności:

- **Wiele wąskich zapytań**: 1000 × `SELECT title FROM books WHERE id = ?`. Narzut rund dominuje absolutnie.
- **Jedno szerokie zapytanie**: `SELECT * FROM books WHERE id IN (...)` — jedna runda, ale dużo danych.

Reguła praktyczna: **na sieci rundy są droższe niż bajty**. Zmniejszaj liczbę rund (batch, `IN`, JOIN) nawet kosztem większej ilości danych w jednej rundzie — oczywiście z rozsądkiem i przy świadomości limitu rozmiaru jednej odpowiedzi.

> 🧪 **Ćwiczenie.** Zbuduj benchmark na 10 000 wierszy: (a) 10 000 × `SELECT ... WHERE id = ?`, (b) 10 zawołań `SELECT ... WHERE id IN (...)` po 1000 id, (c) jedno `SELECT ... WHERE id BETWEEN ? AND ?`. Zmierz czasy. Zobacz, jak wygląda krzywa.

---

## Antywzorce wydajnościowe

### 1. „Pobierz wszystko i filtruj w Pythonie”

```python
# ❌ Baza nie wie o filtrach, wysyła 100 000 wierszy
books = session.scalars(select(Book)).all()
recent = [b for b in books if b.published_year >= 2020]

# ✅ Baza filtruje
recent = session.scalars(select(Book).where(Book.published_year >= 2020)).all()
```

Konsekwencja pierwszego wariantu: pełny transfer tabeli, pełna instancjonowanie encji, pełna identity map, a potem ~1 ms pracy w Pythonie. To jak zamówienie wszystkich książek z biblioteki kurierem, żeby wybrać te z ostatnich trzech lat.

### 2. `session.flush()` w pętli

```python
# ❌ 1000 flushy = 1000 rund (lub więcej) plus utrata batchowania
for row in rows:
    session.add(Book(**row))
    session.flush()

# ✅ Jeden commit na porcję
for chunk in chunked(rows, 1000):
    session.add_all([Book(**r) for r in chunk])
    session.commit()
```

### 3. Relacja leniwa w pętli

Wspomniane wiele razy, ale to absolutnie najczęstszy problem:

```python
# ❌ N+1
for book in session.scalars(select(Book)):
    print(book.author.name)
```

### 4. Logowanie SQL na produkcji

`echo=True` albo `logging.level=DEBUG` z pluginem logującym wynikowe wiersze — na produkcji to katastrofa. Log debugowy na milionie wierszy potrafi zjeść setki megabajtów i wygenerować gigabajty plików. Wymuś:

```python
import os

DEBUG_SQL = os.getenv("SQL_DEBUG", "false").lower() == "true"
engine = create_engine(url, echo=DEBUG_SQL)
```

I pilnuj tego w code review. `echo=True` albo `echo_pool=True` w kodzie produkcyjnym to błąd klasy „blokujący merge”.

### 5. Wszystko w jednej sesji na zawsze

Długożyjąca sesja to rosnąca identity map, otwarte transakcje i potencjalne konflikty. Sesja żyje tak długo, jak jednostka pracy — nie dłużej.

### 6. Indeks na każdej kolumnie „na wszelki wypadek”

Zwiększa to koszt każdego `INSERT` i `UPDATE`, puchnie dysk, miesza planerowi. Indeksuj po zapytaniach, nie po przeczuciu.

### 7. Indeksy nieużywane, nieusunięte

Podobnie jak wyżej — po miesiącach pracy sprawdź w bazie, które indeksy nie były użyte ani raz i rozważ ich usunięcie.

### 8. Brak limitu na listach

`GET /books` bez `LIMIT` = klient pobiera 500 000 rekordów na żądanie i przeżywa własną wersję ataku DoS (który sam sobie zrobił). Zawsze ograniczaj: domyślnie 20, maksymalnie 200.

### 9. „Policz w pętli”

```python
# ❌ Zamiast agregacji w bazie
count = len(session.scalars(select(Book)).all())

# ✅ `COUNT(*)` w bazie
from sqlalchemy import func

count = session.scalar(select(func.count()).select_from(Book))
```

Pierwszy wariant przy milionie wierszy alokuje obiekty i trzyma je w pamięci tylko po to, żeby zwrócić jedną liczbę.

### 10. N+1 ukryte w pętli po agregatach

```python
# ❌ Jedno zapytanie na każdą kategorię
for category in session.scalars(select(Category)):
    category.count = session.scalar(
        select(func.count()).select_from(Book).where(Book.category_id == category.id)
    )

# ✅ Jedno zapytanie z GROUP BY
stmt = (
    select(Category.id, func.count(Book.id).label("book_count"))
    .outerjoin(Book)
    .group_by(Category.id)
)
counts = dict(session.execute(stmt).all())
```

To zarówno liczba rund, jak i brak agregacji w bazie — podwójny antywzorzec.

---

## Warsztat pomiarowy: 1 000 000 wierszy

### Konfiguracja warsztatu

Uruchom dwa poniższe skrypty. Pierwszy przygotowuje bazę z milionem wierszy, drugi wykonuje pięć eksperymentów. Cały warsztat zajmuje około 10–20 minut czasu (głównie na seed).

```python
# examples/17_seed_benchmark.py
"""Generator bazy: 50 000 książek, 1 000 000 wypożyczeń. Uruchom raz przed warsztatem."""
from __future__ import annotations

import random
import time
from datetime import date, datetime, timedelta

from sqlalchemy import (
    BigInteger,
    Date,
    DateTime,
    ForeignKey,
    Integer,
    String,
    create_engine,
    insert,
    text,
)
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class Book(Base):
    __tablename__ = "books"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    title: Mapped[str] = mapped_column(String(300), index=False)
    isbn: Mapped[str] = mapped_column(String(20), unique=True)
    published_year: Mapped[int] = mapped_column(Integer)


class Loan(Base):
    __tablename__ = "loans"

    id: Mapped[int] = mapped_column(BigInteger, primary_key=True)
    book_id: Mapped[int] = mapped_column(ForeignKey("books.id"))
    lender: Mapped[str] = mapped_column(String(120))
    loaned_at: Mapped[datetime] = mapped_column(DateTime)
    due_date: Mapped[date] = mapped_column(Date)
    returned_at: Mapped[datetime | None] = mapped_column(DateTime, nullable=True)
    fee: Mapped[float]  # celowo Float, żeby pokazać granice agregacji


def chunked(iterable, size: int):
    chunk = []
    for item in iterable:
        chunk.append(item)
        if len(chunk) >= size:
            yield chunk
            chunk = []
    if chunk:
        yield chunk


def main() -> None:
    engine = create_engine("sqlite:///benchmark.sqlite")
    Base.metadata.create_all(engine)

    random.seed(42)
    books_count = 50_000
    loans_count = 1_000_000

    print("Wstawianie książek...")
    start = time.perf_counter()
    with engine.begin() as conn:
        book_rows = (
            {
                "id": i,
                "title": f"Książka numer {i}",
                "isbn": f"ISBN-{i:08d}",
                "published_year": 1950 + (i % 76),
            }
            for i in range(books_count)
        )
        for chunk in chunked(book_rows, 5_000):
            conn.execute(insert(Book), chunk)
    print(f"  książki: {time.perf_counter() - start:.1f} s")

    print("Wstawianie wypożyczeń...")
    start = time.perf_counter()
    lenders = [f"Czytelnik {i}" for i in range(2_000)]
    base_date = datetime(2022, 1, 1)
    with engine.begin() as conn:
        loan_rows = (
            {
                "id": i,
                "book_id": random.randrange(books_count),
                "lender": random.choice(lenders),
                "loaned_at": base_date + timedelta(days=random.randrange(1_200)),
                "due_date": date(2025, 1, 1) + timedelta(days=random.randrange(365)),
                "returned_at": None
                if random.random() < 0.2
                else base_date + timedelta(days=random.randrange(1_400)),
                "fee": round(random.random() * 20, 2),
            }
            for i in range(loans_count)
        )
        for chunk in chunked(loan_rows, 10_000):
            conn.execute(insert(Loan), chunk)
    print(f"  wypożyczenia: {time.perf_counter() - start:.1f} s")

    with engine.begin() as conn:
        conn.execute(text("ANALYZE"))
    print("Gotowe. Baza: benchmark.sqlite")


if __name__ == "__main__":
    main()
```

```bash
python examples/17_seed_benchmark.py
```

```python
# examples/17_workshop.py
"""Pięć eksperymentów wydajnościowych na bazie z 17_seed_benchmark.py."""
from __future__ import annotations

import statistics
import time
from datetime import datetime

from sqlalchemy import create_engine, func, select, text, update
from sqlalchemy.orm import Session, load_only, selectinload

from queries import QueryStats, count_queries  # licznik z 17_query_counter.py
from seed_models import Book, Loan  # modele z 17_seed_benchmark.py

engine = create_engine("sqlite:///benchmark.sqlite")
PROBE_TITLE = "Książka numer 42"


def timed(label: str, func_to_call, repeat: int = 5) -> float:
    """Uruchamia funkcję `repeat` razy, zwraca medianę czasu w ms."""
    timings: list[float] = []
    for _ in range(repeat):
        start = time.perf_counter()
        func_to_call()
        timings.append((time.perf_counter() - start) * 1000)
    median = statistics.median(timings)
    print(f"{label:<45} {median:>10.1f} ms")
    return median


def explain(sql: str) -> str:
    with engine.connect() as conn:
        rows = conn.execute(text(f"EXPLAIN QUERY PLAN {sql}")).all()
    return rows[0][-1]


# ── Eksperyment 1: brak indeksu vs indeks na tytule ────────────────────────
def exp1_index() -> None:
    print("\n=== Eksperyment 1: indeks ===")
    query = f"SELECT id FROM books WHERE title = '{PROBE_TITLE}'"
    print("plan przed:", explain(query))
    before = timed("bez indeksu", lambda: _run_sql(query))
    with engine.begin() as conn:
        conn.execute(text("CREATE INDEX ix_books_title ON books (title)"))
        conn.execute(text("ANALYZE"))
    print("plan po:   ", explain(query))
    after = timed("z indeksem", lambda: _run_sql(query))
    print(f"przyspieszenie: {before / after:.0f}x")


def _run_sql(sql: str) -> None:
    with engine.connect() as conn:
        conn.execute(text(sql)).all()


# ── Eksperyment 2: N+1 vs selectinload ─────────────────────────────────────
def exp2_n_plus_1() -> None:
    print("\n=== Eksperyment 2: N+1 ===")
    with Session(engine) as session, count_queries(engine) as stats:
        loans = session.scalars(select(Loan).limit(200)).all()
        titles = [loan.book.title for loan in loans]
    print("leniwie:    ", stats.summary())

    with Session(engine) as session, count_queries(engine) as stats:
        stmt = select(Loan).options(selectinload(Loan.book)).limit(200)
        loans = session.scalars(stmt).all()
        titles = [loan.book.title for loan in loans]
    print("selectinload:", stats.summary())


# ── Eksperyment 3: bulk insert vs pętla ────────────────────────────────────
def exp3_bulk_insert() -> None:
    print("\n=== Eksperyment 3: insert ===")
    from seed_models import Book as B

    with engine.begin() as conn:
        conn.execute(text("DELETE FROM books WHERE id > 900000"))
        conn.execute(text("DELETE FROM loans WHERE id > 900000"))

    def via_loop() -> None:
        with engine.begin() as conn:
            for i in range(900_001, 903_001):
                conn.execute(
                    text("INSERT INTO books (id, title, isbn, published_year) VALUES (:i, :t, :isbn, :y)"),
                    {"i": i, "t": f"Book {i}", "isbn": f"ISBN-X-{i}", "y": 2024},
                )

    def via_executemany() -> None:
        with engine.begin() as conn:
            conn.execute(
                text("INSERT INTO books (id, title, isbn, published_year) VALUES (:i, :t, :isbn, :y)"),
                [{"i": i, "t": f"Book {i}", "isbn": f"ISBN-Y-{i}", "y": 2024} for i in range(904_001, 907_001)],
            )

    timed("3000 × INSERT w pętli", via_loop, repeat=1)
    timed("3000 × INSERT executemany", via_executemany, repeat=1)


# ── Eksperyment 4: all() vs strumień ───────────────────────────────────────
def exp4_stream() -> None:
    print("\n=== Eksperyment 4: all() vs strumień ===")
    import tracemalloc

    def via_all() -> int:
        tracemalloc.start()
        with Session(engine) as session:
            rows = session.scalars(select(Loan)).all()
        _, peak = tracemalloc.get_traced_memory()
        tracemalloc.stop()
        print(f"all():   szczyt={peak / 1024 / 1024:>8.1f} MB")
        return len(rows)

    def via_stream() -> int:
        tracemalloc.start()
        seen = 0
        with Session(engine) as session:
            stmt = select(Loan)
            for _ in session.scalars(stmt, execution_options={"yield_per": 2000}):
                seen += 1
        _, peak = tracemalloc.get_traced_memory()
        tracemalloc.stop()
        print(f"stream:  szczyt={peak / 1024 / 1024:>8.1f} MB")
        return seen

    timed("wszystkie wiersze (all)", via_all, repeat=1)
    timed("strumień (yield_per=2000)", via_stream, repeat=1)


# ── Eksperyment 5: agregacja w Pythonie vs w SQL ───────────────────────────
def exp5_aggregation() -> None:
    print("\n=== Eksperyment 5: agregacja ===")

    def in_python() -> float:
        with Session(engine) as session:
            fees = [loan.fee for loan in session.scalars(select(Loan))]
        return sum(fees)

    def in_sql() -> float:
        with Session(engine) as session:
            return session.scalar(select(func.coalesce(func.sum(Loan.fee), 0.0))) or 0.0

    timed("suma w Pythonie", in_python, repeat=3)
    timed("suma w SQL (SUM)", in_sql, repeat=3)


if __name__ == "__main__":
    exp1_index()
    exp2_n_plus_1()
    exp3_bulk_insert()
    exp4_stream()
    exp5_aggregation()
```

### Przykładowe wyniki

| Eksperyment | Wariant A | Wariant B | Poprawa |
|---|---|---|---|
| 1. Indeks | `SCAN books` — **31 ms** | `SEARCH ... USING INDEX` — **0.08 ms** | ~390× |
| 2. N+1 (200 wyp.) | leniwie: **200+1 zapytań, 480 ms** | `selectinload`: **2 zapytania, 14 ms** | ~34× |
| 3. Insert (3000) | pętla: **~2.1 s** | `executemany`: **~0.09 s** | ~23× |
| 4. Odczyt (1 M) | `all()`: szczyt **~1 180 MB** | strumień: szczyt **~6 MB** | ~200× mniej pamięci |
| 5. Agregacja | Python: **~4.7 s** | SQL: **~0.09 s** | ~52× |

> 💡 **Zapamiętaj kształt, nie liczby.** Wszystkie eksperymenty pokazują ten sam wzorzec: przeniesienie pracy do bazy albo ograniczenie rund daje **rzędy wielkości** poprawy. Mikrooptymalizacje (np. skrócenie nazwy zmiennej, zmiana kolejności importów) dają 5–15%. Najpierw rzędy wielkości, potem procenty.

---

## Checklist optymalizacyjny

Dziesięć pytań do zadania sobie przed każdym przeglądem kodu warstwy danych:

1. **Czy zmierzyłem?** Znasz liczbę zapytań i czas dla scenariusza, który optymalizujesz? Bez pomiaru — stop.
2. **Czy policzyłem zapytania w pętli?** Uruchom licznik na ścieżce krytycznej. Jeśli liczba zapytań zależy od liczby obiektów — masz N+1.
3. **Czy ładuję tylko potrzebne kolumny?** W raportach i listach używam projekcji kolumn albo `load_only`.
4. **Czy agregacje są w bazie?** `SUM`, `COUNT`, `MAX`, `AVG` robi baza, nie Python.
5. **Czy każde filtrowanie jest w `WHERE`?** Brak filtrowania w Pythonie po pobraniu całości.
6. **Czy są indeksy na kolumnach z `WHERE`, `JOIN` i `ORDER BY`?** Sprawdzone przez `EXPLAIN`, nie przez przeczucie.
7. **Czy istnieją nieużywane indeksy?** Przejrzałem `pg_stat_user_indexes` (PostgreSQL) lub listę indeksów SQLite.
8. **Czy `expire_on_commit` jest świadomą decyzją?** Wiem, gdzie obiekty są ponownie odczytywane po commicie.
9. **Czy odczyty masowe strumieniują?** Duże raporty używają `yield_per` albo `stream`. Nie ma `scalars().all()` na czymś, co ma 100 000 wierszy.
10. **Czy pula połączeń jest dobrana i monitorowana?** Znam `pool_size`, `max_overflow` i wiem, jak odczytać `engine.pool.status()`.

### Przed/po — przykładowy raport z optymalizacji

| Metryka | Przed | Po | Zmiana |
|---|---|---|---|
| Zapytania na request | 1 042 | 6 | −99.4% |
| Czas odpowiedzi (p95) | 3 180 ms | 148 ms | −95.3% |
| Szczyt pamięci procesu | 980 MB | 62 MB | −93.7% |
| Obciążenie CPU na request | 41 ms | 9 ms | −78.0% |
| Liczba `INSERT`-ów na import (100 k) | 100 000 | 100 | −99.9% |

> 🧠 **Dlaczego to wygląda jak magia, a nie jest.** Trzy zmiany, każda mierzona z osobna: (a) `selectinload` zamiast leniwego ładowania, (b) agregacja w SQL zamiast w Pythonie, (c) `insertmanyvalues` w imporcie. Każda daje rząd wielkości. Żadna nie wymagała niezwykłej wiedzy — tylko pomiaru i wiedzy, gdzie szukać.

---

## Podsumowanie

1. **Pomiar przed optymalizacją.** Licznik zapytań (`before_cursor_execute`), `time.perf_counter`, `EXPLAIN`, `EXPLAIN ANALYZE`. Bez tych narzędzi optymalizujesz na wyczucie, czyli prawdopodobnie w złą stronę.
2. **Zmieniaj jedną rzecz naraz** i zapisuj wynik. Jeśli zmieniasz pięć, nie wiesz, co zadziałało.
3. **Cztery dźwignie, w kolejności ważności:** liczba rund do bazy (N+1), ilość przesyłanych danych, długość transakcji, indeksy.
4. **Indeks to skorowidz, nie magia.** Pomaga na równość, zakresy i prefiksy; nie pomaga na `%wzorzec%` (chyba że `pg_trgm`). Kosztuje przy każdym zapisie.
5. **`ForeignKey` nie tworzy indeksu automatycznie** w SQLite, PostgreSQL ani MySQL. Jeśli po kolumnie łączysz lub filtrujesz — dodaj indeks ręcznie.
6. **Operacje masowe rób masowo.** `execute(insert(T), rows)` z `insertmanyvalues_page_size` jest rzędy wielkości szybsze od `add_all` przy dużych zbiorach; `UPDATE ... WHERE` jest rzędy wielkości szybsze od pętli po obiektach.
7. **Strumieniuj duże odczyty.** `yield_per`, `stream()` i `partitions()` dają stałą pamięć. `yield_per` z `joinedload` kolekcji to konflikt — używaj `selectinload`.
8. **`expire_on_commit=True` kosztuje odczyty** po commicie. Wyłącz go świadomie (API, async) i pamiętaj o `refresh` tam, gdzie potrzebujesz świeżych danych z bazy.
9. **`cache_ok = True` na własnych typach** jest obowiązkowe, jeśli chcesz korzystać z cache kompilacji. Nie ustawiaj go, jeśli typ zależy od stanu zewnętrznego.
10. **Sieć kosztuje rundami, nie bajtami.** Mniej rund wygrywa z mniejszymi zapytaniami. Paginuj keysetem, nie `OFFSET`-em, na głębokich stronach.

---

## Ćwiczenia

### Ćwiczenie 1 — Licznik zapytań w praktyce

Masz endpoint (albo funkcję) listujący 100 zamówień wraz z pozycjami. Zaimplementuj go w sposób oczywisty, używając leniwego ładowania. Zmierz liczbę zapytań i czas licznikiem z tego modułu. Następnie przepisz funkcję tak, żeby wykonała **co najwyżej 3 zapytania** — udokumentuj każdą zmianę liczbą.

### Ćwiczenie 2 — Przyspieszenie 10×

Dany jest skrypt raportowy na tabeli 1 000 000 wypożyczeń, który:
- pobiera wszystkie wypożyczenia (`scalars(select(Loan)).all()`),
- filtruje w Pythonie te z `returned_at is None`,
- dla każdego z nich pobiera książkę leniwym dostępem do relacji,
- liczy sumę opłat w Pythonie.

Twoje zadanie: przepisz go tak, żeby działał **co najmniej 10× szybciej**, i uzasadnij każdą zmianę liczbą oraz fragmentem planu zapytania. Wymagane: raport musi zwracać tę samą liczbę wierszy i tę samą sumę.

### Ćwiczenie 3 — Zaprojektuj indeksy pod realne zapytania

Dla schematu biblioteki (książki, autorzy, kategorie, wypożyczenia) zebrałeś następujące zapytania produkcyjne:

1. `SELECT * FROM loans WHERE book_id = ? AND returned_at IS NULL`
2. `SELECT * FROM loans WHERE lender = ? ORDER BY loaned_at DESC LIMIT 20`
3. `SELECT title FROM books WHERE title LIKE 'Wiedźmin%'`
4. `SELECT COUNT(*) FROM loans WHERE loaned_at BETWEEN ? AND ?`
5. `SELECT b.* FROM books b JOIN loans l ON l.book_id = b.id WHERE l.lender = ?`

Zaproponuj zestaw indeksów (maksymalnie 5). Dla każdego podaj definicję SQL/`Index(...)` w SQLAlchemy 2.0 i uzasadnienie, dlaczego pasuje do konkretnego zapytania. Zakładaj PostgreSQL.

### Rozwiązania

<details>
<summary><strong>Rozwiązanie ćwiczenia 1</strong></summary>

Wariant „oczywisty”:

```python
with Session(engine) as session, count_queries(engine) as stats:
    orders = session.scalars(select(Order).limit(100)).all()
    payload = [
        {
            "id": order.id,
            "items": [(item.sku, item.qty) for item in order.items],
        }
        for order in orders
    ]
print(stats.summary())
# zapytania=101  czas_bazy=~390 ms
```

Wariant zmierzony i naprawiony:

```python
from sqlalchemy.orm import selectinload

with Session(engine) as session, count_queries(engine) as stats:
    stmt = (
        select(Order)
        .options(selectinload(Order.items))
        .order_by(Order.id)
        .limit(100)
    )
    orders = session.scalars(stmt).all()
    payload = [
        {"id": order.id, "items": [(item.sku, item.qty) for item in order.items]}
        for order in orders
    ]
print(stats.summary())
# zapytania=2  czas_bazy=~11 ms
```

Uzasadnienie: `selectinload(Order.items)` zamienia $1+N$ zapytań na 2 (jedno po zamówienia, jedno `IN (...)`) po pozycje. Kolumn w projekcji nie zmieniam, bo raportu nie da się zbudować bez pól pozycji — więc `load_only` tu nie pomaga. Trzecie zapytanie (jeśli się pojawi) to prawdopodobnie `SELECT COUNT` na paginację — dlatego „co najwyżej 3”.

</details>

<details>
<summary><strong>Rozwiązanie ćwiczenia 2</strong></summary>

Wersja wejściowa (wolna, ~6–8 s na 1M wierszy):

```python
with Session(engine) as session:
    loans = session.scalars(select(Loan)).all()
    active = [loan for loan in loans if loan.returned_at is None]
    rows = [(loan.book.title, loan.fee) for loan in active]
    total = sum(fee for _, fee in rows)
```

Wersja naprawiona (docelowo rzędy wielkości szybciej):

```python
from sqlalchemy import func, select
from sqlalchemy.orm import selectinload

with Session(engine) as session:
    stmt = (
        select(Loan)
        .where(Loan.returned_at.is_(None))
        .options(selectinload(Loan.book).load_only(Book.title))
    )
    loans = session.scalars(stmt, execution_options={"yield_per": 2000})
    rows = [(loan.book.title, loan.fee) for loan in loans]

    total = session.scalar(
        select(func.coalesce(func.sum(Loan.fee), 0.0)).where(Loan.returned_at.is_(None))
    )
```

Cztery zmiany i uzasadnienie liczbowo-planowe:

1. **Filtr w `WHERE` zamiast w Pythonie.** Baza odrzuca nieaktywne wypożyczenia na poziomie indeksu (`ix_loans_active` z modułu), zamiast wysyłać je przez sieć. Efekt: liczba zwracanych wierszy spada z ~1 000 000 do ~200 000 (w danych generowanych 20% jest aktywnych).
2. **Indeks częściowy na `returned_at IS NULL`.** Plan zmienia się ze `SCAN loans` na `SEARCH loans USING INDEX ix_loans_active`. Czas filtrowania: sekundy → dziesiątki milisekund.
3. **`selectinload` + `load_only(Book.title)`.** Odczyty relacji przestają być N+1; dodatkowo pobieramy z książek tylko tytuł.
4. **Strumień `yield_per`.** Szczyt pamięci z ~1 GB → kilka MB, więc nie ma GC pressure.

Weryfikacja „ta sama liczba wierszy, ta sama suma”: asercje `assert len(rows) == expected_rows` i `assert abs(total - expected_total) < 0.01` na kontrolnie wyliczonych wartościach.

</details>

<details>
<summary><strong>Rozwiązanie ćwiczenia 3</strong></summary>

```python
from sqlalchemy import Index, text

# 1. Zapytanie: loans WHERE book_id = ? AND returned_at IS NULL
#    + Zapytanie 5: JOIN books/l oans po book_id
Index("ix_loans_book_id", "book_id")  # bazowy, potrzebny też do JOIN-a

# 2. Zapytanie 1: indeks częściowy — tylko aktywne wypożyczenia danego tytułu
Index(
    "ix_loans_active_by_book",
    "book_id",
    "loaned_at",
    postgresql_where=text("returned_at IS NULL"),
)

# 3. Zapytanie 2: lender + ORDER BY loaned_at DESC
Index("ix_loans_lender_loaned_at", "lender", "loaned_at")

# 4. Zapytanie 3: prefix LIKE 'Wiedźmin%' — B-tree radzi sobie z prefiksem
Index("ix_books_title", "title")

# 5. Zapytanie 4: COUNT po zakresie czasu
Index("ix_loans_loaned_at", "loaned_at")
```

Uzasadnienia:

- **`ix_loans_book_id`** obsługuje `WHERE book_id = ?` oraz JOIN po `l.book_id = b.id` (zapytanie 5). `ForeignKey` sam indeksu nie tworzy — to najczęstsze przeoczenie.
- **`ix_loans_active_by_book`** jest indeksem częściowym: zawiera tylko aktywne wypożyczenia (ok. 20% tabeli), więc jest ~5× mniejszy i szybszy w utrzymaniu niż pełny indeks dwukolumnowy. Kolejność `(book_id, loaned_at)` pozwala na przeszukanie po `book_id` i od razu odczyt posortowany po `loaned_at` w razie potrzeby.
- **`ix_loans_lender_loaned_at`** obsługuje filtr po `lender` i sortowanie `ORDER BY loaned_at DESC`. PostgreSQL może użyć go w odwrotnej kolejności (`DESC`) bez dodatkowego sortowania. Kolejność kolumn jest tu kluczowa — odwrotna (`loaned_at, lender`) nie zadziałałaby dla tego zapytania.
- **`ix_books_title`** obsługuje prefiks `LIKE 'Wiedźmin%'`. B-tree radzi sobie z prefiksem dzięki porządkowi leksykalnemu. Warunek `%Wiedźmin%` (środek) wymagałby `pg_trgm` i indeksu GIN.
- **`ix_loans_loaned_at`** obsługuje czysty zakres `BETWEEN` po czasie. Zapytanie 4 nie ma innych warunków — prosty indeks jednej kolumny jest tu optymalny.

Uwaga o ilości: to pięć indeksów, ale indeksy 3, 4 i 5 nie nakładają się na siebie (różne tabele / różne kolumny wiodące). Natomiast gdyby lista zapytań była dłuższa, warto rozważyć, czy `ix_loans_lender_loaned_at` i `ix_loans_loaned_at` nie mogą być połączone indeksem pokrywającym. Przy dużym wolumenie zapisów redukcja liczby indeksów ma realne znaczenie — sprawdź `idx_scan` w `pg_stat_user_indexes`.

</details>

---

## Najczęstsze błędy i jak je czytać

| Komunikat | Przyczyna | Naprawa |
|---|---|---|
| `TimeoutError: QueuePool limit of size 5 overflow 10 reached, connection timed out` | Pula wyczerpana: albo za mała, albo wyciek połączeń | Sprawdź `engine.pool.status()`. Upewnij się, że każda sesja jest zamykana (`with Session(...)`); rozważ zwiększenie `pool_size`/`max_overflow` albo skrócenie transakcji |
| `sqlalchemy.exc.OperationalError: (sqlite3.OperationalError) database is locked` | SQLite dopuszcza jednego piszącego; długie transakcje blokują dostęp | Skróć transakcje, włącz tryb WAL (`PRAGMA journal_mode=WAL`) lub przenieś się na PostgreSQL, jeśli potrzebujesz prawdziwej współbieżności |
| `sqlalchemy.exc.InvalidRequestError: Could not evaluate current criteria in Python: ...` | `synchronize_session='evaluate'` nie potrafi ocenić Twojego wyrażenia w ORM-owym `update()`/`delete()` | Użyj `execution_options={"synchronize_session": "fetch"}` (poprawnie, ale drogo) albo `False` (najszybciej, ale trzeba odświeżyć obiekty ręcznie) |
| `MemoryError` przy odczycie dużego zbioru | `scalars().all()` materializuje wszystkie obiekty | `yield_per`, `stream()`, `partitions()` — albo projekcja kolumn zamiast encji |
| `SELECT` pojawia się w logu po każdej iteracji pętli | Leniwe ładowanie relacji (N+1) | `selectinload`, `joinedload`, `contains_eager` — patrz [`11_ladowanie_i_n_plus_1.md`](11_ladowanie_i_n_plus_1.md) |
| Ostrzeżenie: `TypeDecorator ... will not produce a cache key` | Brak lub `cache_ok = False` na własnym `TypeDecorator` | Ustaw `cache_ok = True`, jeśli typ jest deterministyczny; w przeciwnym razie zostaw `False` i zaakceptuj koszt kompilacji |
| Nagłe spowolnienie po wdrożeniu, plan się zmienił | Nowe dane, nowe rozkłady lub nieaktualne statystyki planera | `ANALYZE` (PostgreSQL, SQLite), sprawdź `EXPLAIN ANALYZE` po zmianie; rozważ `VACUUM ANALYZE` |
| `DetachedInstanceError` przy próbie odczytu po zamknięciu sesji | Dostęp do atrybutu poza sesją — najczęściej relacja ładowana leniwie | Wczytaj relację przed zamknięciem sesji (`selectinload`) albo ustaw `expire_on_commit=False` — patrz [`08_sesja_cykl_zycia.md`](08_sesja_cykl_zycia.md) |
| `sqlalchemy.exc.CompileError` przy `LIKE` z wielkimi literami na SQLite | SQLite nie wspiera `ILIKE` natywnie; SQLAlchemy mapuje je inaczej niż PostgreSQL | Użyj `func.lower(col).like(value.lower())` albo wiedz, że `ilike` na SQLite działa tylko dla ASCII |
| Brak poprawy po dodaniu indeksu | Indeks nie pasuje do zapytania (inny prefiks kolumn, funkcja na kolumnie, `%wzorzec%`) | Sprawdź `EXPLAIN`, czy baza w ogóle używa indeksu. Indeks nie zadziała dla `WHERE func(x) = ...` — potrzebny indeks wyrażeniowy |

---

## Słowniczek modułu

| Termin (EN) | Polski odpowiednik | Wyjaśnienie |
|---|---|---|
| `round trip` | runda do bazy | Jedna pełna wymiana zdań z bazą: wysłanie zapytania → wykonanie → odbiór wyników. Każda runda to narzut sieciowy |
| `connection pool` | pula połączeń | Zbiór gotowych połączeń utrzymywanych przez aplikację, żeby nie tworzyć nowego połączenia przy każdym zapytaniu |
| `pool_size` / `max_overflow` | rozmiar bazowy / nadmiar | Minimalna i dodatkowa liczba połączeń w puli |
| `pool_pre_ping` | ping przed wydaniem | Sprawdzenie, czy połączenie żyje, przed wydaniem go z puli |
| `prepared statement` | instrukcja przygotowana | Zapytanie sparsowane i zaplanowane raz, wykonywane wielokrotnie z różnymi parametrami |
| `index scan` | skan indeksu | Odczyt danych przez indeks — szybki dla dobrze selektywnych zapytań |
| `seq scan` / `SCAN` | skan sekwencyjny | Odczyt tabeli wiersz po wierszu — właściwy przy dużych zapytaniach zbiorczych, zły przy pojedynczych trafieniach |
| `covering index` | indeks pokrywający | Indeks zawierający wszystkie kolumny potrzebne zapytaniu — baza nie musi odczytywać tabeli |
| `partial index` | indeks częściowy | Indeks zawierający tylko wiersze spełniające warunek (np. `WHERE returned_at IS NULL`) |
| `B-tree` | B-drzewo | Domyślny typ indeksu — uporządkowane drzewo, obsługuje równość, zakresy i prefiksy |
| `pg_trgm` | rozszerzenie trigramowe (PostgreSQL) | Rozszerzenie indeksujące trójki znaków, umożliwiające `LIKE '%...%'` |
| `insertmanyvalues` | wstawianie wielu wartości | Mechanizm SQLAlchemy 2.0, który kompiluje masowe wstawianie w kilkuelementowe `INSERT ... VALUES (...), (...), ...` |
| `executemany` | wykonanie wielokrotne | Wysłanie jednego zapytania z wieloma zestawami parametrów (semantyka DBAPI) |
| `COPY` | kopiowanie masowe (PostgreSQL) | Protokół szybkiego ładowania danych bez użycia parsera SQL-a |
| `yield_per` | strumieniowanie w porcjach | Tryb odczytu, który pobiera wiersze partiami i nie trzyma całego wyniku w pamięci |
| `partitions()` | porcje wyników | Metoda `Result`, zwracająca wyniki w listach o zadanym rozmiarze |
| `expire_on_commit` | wygaśnięcie po commicie | Flaga sesji: po `commit` obiekty są oznaczone jako nieaktualne i zostaną odczytane ponownie przy następnym dostępie |
| `identity map` | mapa tożsamości | Rejestr obiektów wczytanych w sesji, kluczowany po `(klasa, klucz główny)`; zapewnia, że ten sam wiersz to ten sam obiekt Pythona |
| `autoflush` | automatyczny zrzut zmian | Wysłanie oczekujących zmian (`INSERT`/`UPDATE`/`DELETE`) przed wykonaniem zapytania |
| `query_cache_size` | rozmiar cache zapytań | Liczba skompilowanych struktur zapytań trzymanych w pamięci |
| `cache_ok` | flaga bezpieczeństwa cache | Atrybut typu, mówiący SQLAlchemy: „ten typ kompiluje się deterministycznie, cache jest bezpieczny” |
| `keyset pagination` | paginacja kursorowa | Paginacja po wartości klucza (`WHERE id > last_id`) zamiast `OFFSET` — stała złożoność niezależnie od głębokości |
| `EXPLAIN ANALYZE` | plan z wykonaniem | Instrumentacja, która wykonuje zapytanie i pokazuje rzeczywisty czas oraz liczbę przetworzonych wierszy |
| `synchronize_session` | synchronizacja sesji | Strategia aktualizacji obiektów w pamięci przy masowym `update()`/`delete()` |

---

## Dalsze czytanie

- **Pula połączeń i `create_engine`** — `https://docs.sqlalchemy.org/en/20/core/engines.html`
- **Zdrowie puli połączeń, `pre_ping`, eventy połączeniowe**: `https://docs.sqlalchemy.org/en/20/core/pooling.html`
- **Wydajność ORM (`yield_per`, `partitions`, projekcje kolumn)** — `https://docs.sqlalchemy.org/en/20/orm/queryguide/index.html`
- **Optymalizacja ładowania relacji** — `https://docs.sqlalchemy.org/en/20/orm/queryguide/relationships.html`
- **`insertmanyvalues` i parametry masowego wstawiania** — `https://docs.sqlalchemy.org/en/20/core/connections.html`
- **Cache kompilacji i `cache_ok`** — `https://docs.sqlalchemy.org/en/20/core/compiler.html`
- **Nowości i zmiany w 2.1 (jeśli już jesteś na tej gałęzi)** — `https://docs.sqlalchemy.org/en/21/changelog/`
- **Modele typów i `TypeDecorator`** — `https://docs.sqlalchemy.org/en/20/core/custom_types.html`
- **PostgreSQL: `EXPLAIN`, `pg_stat_user_indexes`, `pg_trgm`** — `https://www.postgresql.org/docs/current/using-explain.html`
- **Alembic 1.19 i migracje** — `https://alembic.sqlalchemy.org/en/latest/`

---

## Co dalej

Masz narzędzia pomiaru i zestaw technik, które dają rzędy wielkości. Ale wydajność jest bezwartościowa, jeśli kod się psuje przy pierwszej zmianie wymagań. W module [`18_testowanie.md`](18_testowanie.md) zbudujesz zestaw testów, które wykryją N+1 **przed** produkcją, sprawdzą, że migracja działa w obie strony, i zabezpieczą wszystkie wnioski, które właśnie wyciągnąłeś — automatycznie, przy każdym uruchomieniu `pytest`. Licznik zapytań, który napisałeś w tym module, stanie się asercją w `conftest.py`.

<!-- koniec modułu 17 -->