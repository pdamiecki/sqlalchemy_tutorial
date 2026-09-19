I'll verify a few async API details against the official docs before writing, so nothing in the module is invented.



# Moduł 15 — Asynchroniczność: SQLAlchemy w świecie `async`/`await`

W tym module przestawisz SQLAlchemy na tryb asynchroniczny. Dowiesz się, dlaczego samo dodanie `async`/`await` do kodu nie czyni go szybszym, dlaczego zwykły sterownik bazy danych unieważnia całą korzyść z asynchroniczności i jak zbudować aplikację, w której sto zapytań do bazy leci równolegle, a nie jedno po drugim. Poznasz `create_async_engine()`, `AsyncSession`, `async_sessionmaker`, mechanizm `run_sync`, bibliotekę `greenlet` kryjącą się pod spodem oraz najbardziej irytujący błąd tego trybu — `MissingGreenlet`. Nauczysz się też, kiedy async naprawdę pomaga, a kiedy jest tylko dodatkową złożonością bez zysku.

---

> **Poziom:** 🔴 architektoniczny
>
> **Czas:** ~180 minut (plus 60–90 minut na ćwiczenia)
>
> **Wymagania wstępne:**
> - [`11_ladowanie_i_n_plus_1.md`](11_ladowanie_i_n_plus_1.md) — strategie ładowania relacji (`selectinload`, `joinedload`, `lazy="raise"`) są tu fundamentem,
> - [`08_sesja_cykl_zycia.md`](08_sesja_cykl_zycia.md) — cykl życia sesji, `flush`, `commit`, `expire_on_commit`,
> - [`14_transakcje_i_wspolbieznosc.md`](14_transakcje_i_wspolbieznosc.md) — transakcje, pula połączeń, granice transakcji,
> - [`02_srodowisko_i_engine.md`](02_srodowisko_i_engine.md) — `Engine`, URL-e połączeń, pula połączeń,
> - podstawowa znajomość `async`/`await` w Pythonie (moduł zawiera powtórkę w sekcji 1, więc nie musisz być ekspertem).
>
> **Czego dotyczy ten plik:** tryb asynchroniczny (`sqlalchemy.ext.asyncio`) w SQLAlchemy 2.0.x. Wszystkie przykłady są uruchamialne lokalnie na SQLite (`aiosqlite`), a warianty produkcyjne pokazujemy na PostgreSQL (`asyncpg`).

---

## Spis treści

1. [Async w piętnaście minut — powtórka dla laika](#1-async-w-piętnaście-minut--powtórka-dla-laika)
2. [Dlaczego blokujące IO zabija asynchroniczność](#2-dlaczego-blokujące-io-zabija-asynchroniczność)
3. [Sterowniki asynchroniczne — wymiana silnika pod maską](#3-sterowniki-asynchroniczne--wymiana-silnika-pod-maską)
4. [`create_async_engine()` — pierwszy silnik async](#4-create_async_engine--pierwszy-silnik-async)
5. [`AsyncSession` i `async_sessionmaker`](#5-asyncsession-i-async_sessionmaker)
6. [DDL w async — `run_sync` i zielone wątki](#6-ddl-w-async--run_sync-i-zielone-wątki)
7. [Pod maską: `greenlet` i most sync ↔ async](#7-pod-maską-greenlet-i-most-sync--async)
8. [`MissingGreenlet` — dlaczego leniwe ładowanie wybucha](#8-missinggreenlet--dlaczego-leniwe-ładowanie-wybucha)
9. [Trzy strategie radzenia sobie z relacjami w async](#9-trzy-strategie-radzenia-sobie-z-relacjami-w-async)
10. [Strumieniowanie dużych zbiorów](#10-strumieniowanie-dużych-zbiorów)
11. [`AsyncEngine` i `Engine` w jednym projekcie](#11-asyncengine-i-engine-w-jednym-projekcie)
12. [Anulowanie, timeouty i sesja po błędzie](#12-anulowanie-timeouty-i-sesja-po-błędzie)
13. [Współbieżność: `gather`, `TaskGroup` i `Semaphore`](#13-współbieżność-gather-taskgroup-i-semaphore)
14. [Wydajność: kiedy async naprawdę pomaga](#14-wydajność-kiedy-async-naprawdę-pomaga)
15. [Alembic i testy w trybie async](#15-alembic-i-testy-w-trybie-async)
16. [Kompletny przykład — CLI pobierające dane równolegle](#16-kompletny-przykład--cli-pobierające-dane-równolegle)
17. [Podsumowanie](#podsumowanie)
18. [Ćwiczenia](#ćwiczenia)
19. [Najczęstsze błędy i jak je czytać](#najczęstsze-błędy-i-jak-je-czytać)
20. [Słowniczek modułu](#słowniczek-modułu)
21. [Dalsze czytanie](#dalsze-czytanie)
22. [Co dalej](#co-dalej)

---

## 1. Async w piętnaście minut — powtórka dla laika

Zacznijmy od problemu, który asynchroniczność rozwiązuje. Nie od składni.

### 1.1 Problem: program, który stoi i czeka

Wyobraź sobie restaurację, w której zatrudniono jednego kelnera o bardzo sztywnej zasadzie: **obsługuje jeden stolik naraz, od początku do końca**. Podchodzi do stolika numer 1, przyjmuje zamówienie, idzie do kuchni, czeka, aż danie się ugotuje, zanosi je, wraca, przyjmuje deser, czeka, zanosi… i dopiero wtedy idzie do stolika numer 2.

Gdy kuchnia gotuje pierwsze danie przez 15 minut, kelner stoi bezczynnie. Nie robi nic pożytecznego — po prostu czeka. W tym czasie 19 pozostałych stolików nie zostało nawet zauważonych.

To jest **dokładnie** sytuacja zwykłego, synchronicznego programu w Pythonie wykonującego zapytanie do bazy danych:

```python
# programs/15_sync_wait.py
import time

def call_database() -> str:
    """Symulacja zapytania, które trwa 0.1 s (sieć + baza)."""
    time.sleep(0.1)  # <- przez ten czas wątek NIC nie robi
    return "wynik"

start = time.perf_counter()
for _ in range(20):
    call_database()
print(f"20 zapytań sekwencyjnie: {time.perf_counter() - start:.2f} s")
# 20 zapytań sekwencyjnie: 2.01 s
```

Zauważ: 2 sekundy to prawie wyłącznie **czekanie**. Procesor przez ten czas odpoczywa.

> 💡 **Analogia — kelner z asynchronicznością.** Teraz wyobraź sobie kelnera, który podchodzi do stolika 1, przyjmuje zamówienie, **przekazuje je kucharzowi i od razu idzie dalej**. Podchodzi do stolika 2, 3, 4… Gdy kuchnia dzwoni „danie dla stolika 1 gotowe!”, kelner przerywa to, co robi, zanosi danie i wraca do przerwanej czynności. Jeden kelner, dwadzieścia stolików, żadnego stania bezczynnie.
>
> Formalnie: **asynchroniczność to umiejętność porzucenia zadania w momencie, gdy trzeba na coś poczekać, i zajęcia się w tym czasie czymś innym.**

### 1.2 Cztery pojęcia, które musisz znać

**Korutyna (coroutine).** Funkcja zadeklarowana słowem `async def`. Wywołanie takiej funkcji **nie wykonuje jej treści** — tworzy obiekt, który trzeba uruchomić. Analogia: przepis kulinarny na kartce to nie obiad; trzeba jeszcze kogoś, kto go wykona.

```python
# programs/15_coroutine.py
async def hello() -> str:
    return "cześć"

# To NIE wypisze nic — powstaje tylko obiekt korutyny.
hello()

# To wykonuje treść:
import asyncio
print(asyncio.run(hello()))
# cześć
```

**`await`.** Znacznik „w tym miejscu mogę poczekać”. `await` mówi pętli zdarzeń: *to zadanie musi teraz oddać sterowanie, bo czeka na wynik; zajmij się innymi*. Możesz użyć `await` **wyłącznie** wewnątrz funkcji `async def`.

> ⚠️ **Pułapka — `await` poza `async def`.** Napisanie `await` w zwykłej funkcji kończy się błędem składni `SyntaxError: 'await' outside async function`. To najczęstszy błąd początkujących i — co gorsza — jego wersja odwrotna: **zapomniany `await`**, który nie daje błędu, tylko ciche „nic się nie stało” (o tym w sekcji 12).

**Pętla zdarzeń (event loop).** Mózg całego mechanizmu. Utrzymuje listę zadań gotowych do wykonania, uruchamia je, a gdy zadanie zgłosi „czekam”, odkłada je na bok i bierze następne. Gdy dane zadanie jest gotowe do wznowienia (np. przyszła odpowiedź z bazy), pętla wraca do niego. W praktyce pętla działa w **jednym wątku** — nie ma tu żadnej magii wielowątkowości.

**Zadanie (`Task`).** Korutyna *zaplanowana* do wykonania w pętli. Różnica jest istotna:

| Pojęcie | Co to jest | Kto uruchamia |
|---|---|---|
| Korutyna | obiekt zwrócony przez `async def` | nikt, dopóki jej nie zaplanujesz |
| `Task` | korutyna zaplanowana w pętli, biegnie „w tle” | pętla zdarzeń |
| Pętla zdarzeń | silnik, który przełącza zadania | `asyncio.run()` / framework |

`asyncio.run(coro)` to standardowy sposób uruchomienia programu asynchronicznego: tworzy nową pętlę zdarzeń, wykonuje podaną korutynę do końca i zamyka pętlę. Wywołanie `asyncio.run()` **nie może być zagnieżdżone** — w środku działającej pętli dostaniesz `RuntimeError: asyncio.run() cannot be called from a running event loop`.

> 🧠 **Dlaczego tak jest — jedno wątko, wiele zadań.** W asynchroniczności nie ma wątków ani blokad. Jest jeden wątek i jedna pętla, która bardzo szybko przełącza zadania. Konsekwencja: **żadne zadanie nie może blokować pętli**, bo zablokuje wszystkie pozostałe. To nie jest drobiazg — to fundament, na którym opiera się cała reszta tego modułu.

### 1.3 Tabela translacji: sync → async

W SQLAlchemy zmiana trybu jest mechaniczna i dotyczy niewielu nazw:

| Synchronicznie | Asynchronicznie |
|---|---|
| `create_engine(...)` | `create_async_engine(...)` |
| `Session` | `AsyncSession` |
| `sessionmaker(...)` | `async_sessionmaker(...)` |
| `with engine.connect() as conn:` | `async with engine.connect() as conn:` |
| `conn.execute(stmt)` | `await conn.execute(stmt)` |
| `conn.run_sync(...)` — nie dotyczy | `await conn.run_sync(...)` |
| `engine.dispose()` | `await engine.dispose()` |
| `session.execute(stmt)` | `await session.execute(stmt)` |
| `session.commit()` / `.rollback()` / `.close()` | `await session.commit()` / `.rollback()` / `.close()` |
| `session.get(Book, 1)` | `await session.get(Book, 1)` |
| `session.scalars(stmt)` | `await session.scalars(stmt)` |
| `session.add(obj)` | `session.add(obj)` — **bez `await`!** |
| `session.flush()` | `await session.flush()` |
| `session.refresh(obj)` | `await session.refresh(obj)` |
| `session.delete(obj)` | `await session.delete(obj)` |
| `with session.begin():` | `async with session.begin():` |

Zapamiętaj tę tabelę — to 90% praktycznej wiedzy o tym module. Reszta to konsekwencje i pułapki.

> ⚠️ **Pułapka — `session.add()` bez `await`.** `add()`, `add_all()`, `expunge()`, `expunge_all()`, `expire_all()` to metody **synchroniczne**, bo nie dotykają bazy — tylko pamięć. Jeśli napiszesz `await session.add(obj)`, dostaniesz `TypeError: object NoneType can't be used in 'await' expression`. Odwrotnie: jeśli zapomnisz `await` przy `commit()`, obiekt korutyny zostanie wyrzucony do śmieci, Python wypisze ostrzeżenie `RuntimeWarning: coroutine 'AsyncSession.commit' was never awaited`, a dane **nie zostaną zapisane**.

> 🧪 **Ćwiczenie.** Bez uruchamiania — powiedz, które z poniższych wywołań wymagają `await`: `session.rollback()`, `session.get(Book, 1)`, `session.identity_map`, `session.flush()`, `session.merge(book)`, `engine.dispose()`. (Odpowiedź: `rollback`, `get`, `flush`, `merge`, `dispose` — tak; `identity_map` to zwykły atrybut, nie.)

---

## 2. Dlaczego blokujące IO zabija asynchroniczność

Wróćmy do kelnera. Wyobraź sobie teraz, że kelner ma przy sobie **jedną jedyną słuchawkę telefonu** i musi sam zadzwonić do kuchni, żeby zapytać o danie. A kuchnia odbiera dopiero po 15 minutach. Kelner stoi z słuchawką przy uchu — nie może iść do stolika 2, 3 ani 4.

To jest właśnie **blokujące IO** (blocking I/O — blokujące wejście/wyjście). Synchroniczna biblioteka, która wykonuje operację wejścia/wyjścia, **zajmuje cały wątek** na czas oczekiwania. Nie ma jak powiedzieć pętli zdarzeń „poczekaj, ja tu sobie postoję”.

> 🧠 **Dlaczego tak jest — mechanizm jest głębszy, niż wygląda.** `time.sleep(1)` nie oddaje sterowania pętli zdarzeń. `socket.recv()` nie oddaje. `psycopg2.execute()` nie oddaje. Wszystkie one wywołują **blokujące wywołanie systemowe**, które wstrzymuje cały wątek Pythona. A skoro pętla zdarzeń działa w jednym wątku — wstrzymuje się również pętla, a więc **wszystkie** zadania w aplikacji.

Klasyczny przykład katastrofy:

```python
# programs/15_blocking_disaster.py
import asyncio
import time


async def dobra_robota(name: str) -> None:
    await asyncio.sleep(0.1)
    print(f"zadanie {name} zrobione")


async def zla_robota(name: str) -> None:
    time.sleep(0.1)  # BLOKUJE PĘTLĘ!
    print(f"zadanie {name} zrobione")


async def main() -> None:
    start = time.perf_counter()
    await asyncio.gather(*(dobra_robota(str(i)) for i in range(10)))
    print(f"asynchronicznie: {time.perf_counter() - start:.2f} s")
    # asynchronicznie: 0.10 s  <- wszystkie 10 zadań naraz

    start = time.perf_counter()
    await asyncio.gather(*(zla_robota(str(i)) for i in range(10)))
    print(f"z time.sleep:   {time.perf_counter() - start:.2f} s")
    # z time.sleep:   1.01 s   <- gather NIE pomógł!


asyncio.run(main())
```

`asyncio.gather()` uruchamia dziesięć zadań równolegle — ale tylko dlatego, że `asyncio.sleep()` **oddaje sterowanie**. W wersji z `time.sleep()` zadania są zaplanowane równolegle, ale wykonują się jedno po drugim, bo pierwsze blokuje pętlę na 0,1 s.

### 2.1 Lista „wrogów”

| Blokuje pętlę | Oddaje sterowanie |
|---|---|
| `time.sleep()` | `await asyncio.sleep()` |
| `requests.get()` | `await httpx.AsyncClient().get()` / `aiohttp` |
| `psycopg2` (v2) | `asyncpg`, `psycopg` (v3, tryb async) |
| `sqlite3` (wbudowany) | `aiosqlite` |
| `open(...).read()` (dysk) | `aiofiles` / `await asyncio.to_thread(...)` |
| `bcrypt.hashpw()` (CPU!) | `await asyncio.to_thread(bcrypt.hashpw, ...)` |
| `session.execute(stmt)` (sync) | `await session.execute(stmt)` |

> 💡 **Analogia — praca na zapleczu.** Jeśli musisz naprawdę coś przekopać łopatą (kosztowna operacja CPU), nie robisz tego na sali, tylko idziesz na zaplecze i wynajmujesz dodatkową osobę (`asyncio.to_thread`). Sala pozostaje wolna. Analogia działa też dla odwrotnej sytuacji: asynchroniczność pomaga, gdy **czekasz**; nie pomaga, gdy **liczysz**.

> ⚠️ **Pułapka — „dodam async i będzie szybciej”.** To nieprawda i warto to powiedzieć wprost: dla **jednego** zapytania do bazy tryb async jest *odrobinę wolniejszy* od sync (koszt pętli zdarzeń i mostu `greenlet`). Async wygrywa dopiero wtedy, gdy masz **wiele** równoległych operacji wejścia/wyjścia. Szczegóły z liczbami w sekcji 14.

---

## 3. Sterowniki asynchroniczne — wymiana silnika pod maską

Skoro blokujące IO psuje async, to oczywiste, że **sterownik bazy danych też musi być asynchroniczny**. Sterownik (driver) to biblioteka, która faktycznie rozmawia z bazą — SQLAlchemy nie mówi do bazy bezpośrednio, tylko przez warstwę zwaną **DBAPI**.

> 💡 **Analogia — tłumacz i telefon.** SQLAlchemy jest tłumaczem, który pisze zdanie po zdaniu po polsku (SQL). Sterownik to telefon, przez który ten tekst leci. Jeśli telefon jest stary i działa tylko w trybie „połącz i czekaj, aż ktoś odbierze” (blokującym), żadne sztuczki tłumacza nie pomogą. Potrzebujesz nowego telefonu, który sam zgłasza, kiedy rozmówca jest gotowy.

### 3.1 Kto jest kim

| Baza | Sterownik sync (klasyczny) | Sterownik async | URL SQLAlchemy |
|---|---|---|---|
| PostgreSQL | `psycopg2` / `psycopg` | `asyncpg` | `postgresql+asyncpg://` |
| PostgreSQL | — | `psycopg` (v3, tryb async) | `postgresql+psycopg://` (z `create_async_engine`) |
| SQLite | `sqlite3` (wbudowany) | `aiosqlite` | `sqlite+aiosqlite://` |
| MySQL/MariaDB | `mysqlclient` | `aiomysql` / `asyncmy` | `mysql+aiomysql://` / `mysql+asyncmy://` |
| Oracle | `cx_Oracle` | `oracledb` | `oracle+oracledb://` |

> 🆕 **SQLAlchemy 2.1** — domyślnym sterownikiem dla `postgresql://` jest teraz `psycopg` (wersja 3), a nie `psycopg2`; dla `oracle://` — `oracledb`. Jeśli więc w 2.1 napiszesz `create_engine("postgresql://...")` bez nazwy sterownika, dostaniesz `psycopg`. W 2.0 domyślnym pozostaje `psycopg2`.

> 🧠 **Dlaczego `psycopg` (v3) pojawia się dwa razy.** Ten sam sterownik obsługuje **oba tryby**. SQLAlchemy rozpoznaje, czy używasz go z `create_engine()` (wtedy wybiera wersję synchroniczną), czy z `create_async_engine()` (wtedy asynchroniczną). Możesz też nazwać tryb jawnie: `postgresql+psycopg_async://`.

### 3.2 Instalacja

```bash
# SQLite w trybie async (najprostszy start, zero konfiguracji bazy)
pip install "SQLAlchemy>=2.0" aiosqlite

# PostgreSQL w trybie async
pip install "SQLAlchemy>=2.0" asyncpg

# PostgreSQL przez psycopg v3 (sync + async w jednej bibliotece)
pip install "SQLAlchemy>=2.0" "psycopg[binary]"

# Zawsze warto dołożyć greenlet jawnie (patrz niżej)
pip install "sqlalchemy[asyncio]"
```

> 🆕 **SQLAlchemy 2.1** — `greenlet` **nie jest już instalowany automatycznie** jako zależność. W 2.0 instalował się domyślnie na popularnych platformach (`x86_64`, `aarch64`, `amd64`, `win32`). W 2.1 musisz sam dołożyć `pip install "sqlalchemy[asyncio]"`, inaczej przy pierwszym `await` dostaniesz błąd importu `greenlet`.

> ⚠️ **Pułapka — instalacja na Apple M1 i egzotycznych architekturach.** Dokumentacja SQLAlchemy wyraźnie ostrzega, że dla części platform (m.in. Apple M1) `greenlet` nie ma gotowego koła binarnego (wheel) i musi być zbudowany ze źródeł — co wymaga bibliotek deweloperskich Pythona. Ratunek: `pip install sqlalchemy[asyncio]`, który wymusi instalację `greenlet`, oraz upewnienie się, że masz kompilator i nagłówki Pythona.

> 🧪 **Ćwiczenie.** Sprawdź, czy masz zainstalowane wszystko: `python -c "import greenlet, aiosqlite, sqlalchemy; print(sqlalchemy.__version__)"`. Jeśli to polecenie działa, jesteś gotowy na dalszą część modułu.

---

## 4. `create_async_engine()` — pierwszy silnik async

Silnik asynchroniczny tworzy się niemal tak samo jak synchroniczny — z jednym warunkiem: **dialekt w URL-u musi być asynchroniczny**.

```python
# examples/15_engine.py
from __future__ import annotations

from sqlalchemy.ext.asyncio import AsyncEngine, create_async_engine

# SQLite w pliku — najprostszy start, nic nie trzeba uruchamiać
SQLITE_URL = "sqlite+aiosqlite:///library.db"

# PostgreSQL — wariant produkcyjny
POSTGRES_URL = "postgresql+asyncpg://app:secret@localhost:5432/library"


def make_engine(url: str = SQLITE_URL, *, echo: bool = False) -> AsyncEngine:
    """Tworzy AsyncEngine — jeden na całą aplikację."""
    return create_async_engine(
        url,
        echo=echo,                    # logowanie SQL do stdout
        pool_pre_ping=True,           # sprawdź połączenie przed użyciem
        pool_size=10,                 # bazowa liczba połączeń w puli
        max_overflow=20,              # ile połączeń „awaryjnych” ponad pool_size
        pool_recycle=1800,            # odświeżaj połączenia starsze niż 30 min
        pool_timeout=30,              # ile sekund czekać na wolne połączenie
    )
```

Zestaw parametrów jest **niemal identyczny** jak w `create_engine()` — to celowe. Różnice:

1. **URL musi wskazywać dialekt async.** `create_async_engine("postgresql://...")` zakończy się błędem `InvalidRequestError: The asyncio extension requires an async driver to be used. The loaded 'psycopg2' is not async.`
2. **Pula połączeń jest asynchroniczna.** Domyślnie SQLAlchemy użyje `AsyncAdaptedQueuePool` — czyli zwykłej `QueuePool`, ale z połączeniami adaptowanymi do wywołań asynchronicznych. Z punktu widzenia konfiguracji `pool_size` i `max_overflow` działają identycznie jak w sync.
3. **Wywołanie `dispose()` wymaga `await`.**

### 4.1 `await engine.dispose()` — i dlaczego to naprawdę ważne

```python
# examples/15_dispose.py
import asyncio

from sqlalchemy import text
from sqlalchemy.ext.asyncio import create_async_engine


async def main() -> None:
    engine = create_async_engine("sqlite+aiosqlite:///library.db")
    async with engine.connect() as conn:
        await conn.execute(text("select 1"))
    await engine.dispose()   # <- ZAWSZE, jeśli silnik nie żyje do końca procesu


asyncio.run(main())
```

> 🧠 **Dlaczego tak jest — i dlaczego `dispose()` to nie kosmetyka.** W trybie synchronicznym SQLAlchemy potrafi domknąć połączenia w destruktorze (`__del__`). W trybie asynchronicznym **nie może** — destruktor nie ma jak wywołać `await`. Jeśli więc porzucisz silnik bez `dispose()`, połączenia zostaną otwarte do momentu zamknięcia pętli zdarzeń, a na wyjściu zobaczysz ostrzeżenia w stylu `RuntimeError: Event loop is closed`. Dokumentacja SQLAlchemy wprost zaleca `await engine.dispose()`, gdy silnik jest tworzony w zasięgu funkcji, a nie na poziomie całego procesu.

> ⚠️ **Pułapka — silnik tworzony w każdym żądaniu.** To błąd analogiczny do modułu 02, tylko kosztowniejszy: każdy nowy `AsyncEngine` ma **własną pulę połączeń**. W aplikacji webowej tworzenie silnika w handlerze oznacza otwieranie nowych połączeń TCP do bazy przy każdym żądaniu. Silnik tworzymy **raz** przy starcie aplikacji i przekazujemy dalej.

### 4.2 Pula połączeń w świecie async — trzy przypadki

| Przypadek | Zalecana pula | Dlaczego |
|---|---|---|
| Zwykła aplikacja webowa (asyncpg) | `AsyncAdaptedQueuePool` (domyślna) | Połączenia są kosztowne, warto je recyklingować |
| PgBouncer w trybie `transaction` | `NullPool` | Podwójne poolowanie psuje przygotowane zapytania |
| Wiele pętli zdarzeń / testy | `NullPool` | Połączenia nie mogą wędrować między pętlami |

```python
# examples/15_nullpool.py
from sqlalchemy.ext.asyncio import create_async_engine
from sqlalchemy.pool import NullPool

# PgBouncer (transaction pooling) — wyłączamy pulę po stronie aplikacji
engine = create_async_engine(
    "postgresql+asyncpg://app:secret@localhost:6432/library",
    poolclass=NullPool,
    connect_args={"statement_cache_size": 0},  # asyncpg: bez cache przygotowanych zapytań
)
```

> 🧠 **Dlaczego `NullPool` przy PgBouncerze.** PgBouncer w trybie `transaction` może przekazać kolejne zapytanie z tej samej sesji klienta do **innego** procesu PostgreSQL. Przygotowane zapytania (prepared statements) zbudowane na jednym połączeniu są wtedy nieważne dla drugiego — stąd `statement_cache_size=0` i brak lokalnego poolowania.

> 🆕 **SQLAlchemy 2.1** — `autoflush` w `Session` działa bezwarunkowo. W 2.0 istniały ścieżki, w których `autoflush` był pomijany; jeśli opierałeś się na tym zachowaniu, w 2.1 zobaczysz więcej `flush` niż dotąd. W trybie async oznacza to więcej wywołań `await` pod maską — warto sprawdzić liczbę zapytań w testach.

---

## 5. `AsyncSession` i `async_sessionmaker`

`AsyncSession` to asynchroniczny odpowiednik `Session`. Najważniejsza rzecz do zapamiętania: **to nie jest sesja, która myśli inaczej** — to ta sama sesja, opakowana tak, aby każda operacja wejścia/wyjścia była `await`-owalna.

> 💡 **Analogia — notes naczelnika zmiany, tylko z krótkofalówką.** W module 08 sesja była notesem, do którego naczelnik zmiany zapisywał intencje („dodaj”, „zmień”, „usuń”), a `flush` to moment przekazania poleceń wykonawcom. `AsyncSession` to ten sam notes, ale naczelnik ma krótkofalówkę: każdy kontakt z wykonawcami wymaga naciśnięcia przycisku (`await`) i chwilowego odłożenia notesu.

### 5.1 Fabryka sesji

```python
# examples/15_session.py
from __future__ import annotations

from sqlalchemy.ext.asyncio import (
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)

engine = create_async_engine("sqlite+aiosqlite:///library.db")

# Fabryka — tworzona RAZ na aplikację, przekazywana dalej
SessionLocal = async_sessionmaker(
    bind=engine,
    class_=AsyncSession,        # domyślne, można pominąć
    expire_on_commit=False,     # patrz sekcja 5.2
    autoflush=True,             # domyślne zachowanie
)
```

`async_sessionmaker` ma także metodę `begin()`, która od razu otwiera sesję **i** transakcję:

```python
# examples/15_sessionmaker_begin.py
async with SessionLocal.begin() as session:   # sesja + transakcja, commit na wyjściu
    session.add(Book(title="Nowa książka", author_id=1))
# tutaj transakcja jest już zatwierdzona, sesja zamknięta
```

To wygodny skrót odpowiadający `async with SessionLocal() as session, session.begin():`.

### 5.2 `expire_on_commit=False` — dlaczego w async to prawie obowiązek

Przypomnijmy, co robi `expire_on_commit` (moduł 08): po `commit()` SQLAlchemy **unieważnia** wszystkie atrybuty obiektów w sesji, żeby następny odczyt pobrał świeże dane z bazy. Odczyt unieważnionego atrybutu wywołuje zapytanie SQL — czyli IO.

W trybie synchronicznym to po prostu dodatkowe zapytanie. W trybie asynchronicznym to **nielegalne IO w miejscu, w którym nie ma `await`**:

```python
# examples/15_expire_problem.py
async with SessionLocal() as session:
    book = await session.get(Book, 1)
    await session.commit()

    # expire_on_commit=True: obiekt został unieważniony.
    # Ten wiersz próbuje wywołać zapytanie SQL bez await.
    print(book.title)
    # MissingGreenlet: greenlet_spawn has not been called;
    # can't call await_only() here. Was IO attempted in an unexpected place?
```

Ustawienie `expire_on_commit=False` sprawia, że po `commit()` obiekty zachowują swoje wartości i ten kod po prostu działa.

> 🧠 **Dlaczego tak jest — „nie ma gdzie czekać”.** Pętla zdarzeń może oddać sterowanie tylko w miejscu oznaczonym `await`. Odczyt `book.title` to zwykłe odwołanie do atrybutu — Python nie ma tam żadnej korutyny. SQLAlchemy musi więc albo zablokować pętlę (czego robić nie wolno), albo zgłosić błąd. Zgłasza błąd. To świadoma decyzja projektowa: *żadnego ukrytego IO w trybie async*.

> ⚠️ **Pułapka — `expire_on_commit=False` nie jest bez kosztu.** Obiekty mogą być nieaktualne względem bazy. Jeśli po `commit()` potrzebujesz świeżych danych, użyj jawnie `await session.refresh(obj)` — wtedy IO jest widoczne w kodzie i opatrzone `await`.

### 5.3 Pełny, minimalny przykład ORM w async

```python
# examples/15_async_orm_minimal.py
"""Minimalny, kompletny przykład ORM w trybie asynchronicznym."""
from __future__ import annotations

import asyncio
from datetime import date

from sqlalchemy import ForeignKey, String, select
from sqlalchemy.ext.asyncio import (
    AsyncAttrs,
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship, selectinload


class Base(AsyncAttrs, DeclarativeBase):
    pass


class Author(Base):
    __tablename__ = "authors"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(120))

    books: Mapped[list[Book]] = relationship(back_populates="author", lazy="raise")


class Book(Base):
    __tablename__ = "books"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    published: Mapped[date | None]
    author_id: Mapped[int] = mapped_column(ForeignKey("authors.id"))

    author: Mapped[Author] = relationship(back_populates="books", lazy="raise")


async def main() -> None:
    engine = create_async_engine("sqlite+aiosqlite:///library.db", echo=True)
    SessionLocal = async_sessionmaker(engine, expire_on_commit=False)

    # 1) DDL — wymaga run_sync (patrz sekcja 6)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
        await conn.run_sync(Base.metadata.create_all)

    # 2) Zapis
    async with SessionLocal() as session:
        async with session.begin():
            author = Author(name="Ursula K. Le Guin")
            author.books = [
                Book(title="The Dispossessed", published=date(1974)),
                Book(title="The Left Hand of Darkness", published=date(1969)),
            ]
            session.add(author)

    # 3) Odczyt z jawnym eager loadingiem
    async with SessionLocal() as session:
        stmt = select(Author).options(selectinload(Author.books)).order_by(Author.id)
        authors = (await session.scalars(stmt)).all()
        for a in authors:
            print(a.name, "->", [b.title for b in a.books])

    await engine.dispose()


if __name__ == "__main__":
    asyncio.run(main())
```

> 🔬 **Pod maska — co faktycznie poleci do bazy.** Uruchamiając powyższy kod z `echo=True`, zobaczysz (SQLite, skrócone):

```sql
BEGIN (implicit)
CREATE TABLE authors (
    id INTEGER NOT NULL,
    name VARCHAR(120) NOT NULL,
    PRIMARY KEY (id)
)
CREATE TABLE books (
    id INTEGER NOT NULL,
    title VARCHAR(200) NOT NULL,
    published DATE,
    author_id INTEGER NOT NULL,
    PRIMARY KEY (id),
    FOREIGN KEY(author_id) REFERENCES authors (id)
)
COMMIT

BEGIN (implicit)
INSERT INTO authors (name) VALUES (?) RETURNING id
[generated in 0.00021s] ('Ursula K. Le Guin',)
INSERT INTO books (title, published, author_id) VALUES (?, ?, ?) RETURNING id
[generated in 0.00018s] ('The Dispossessed', '1974-01-01', 1)
INSERT INTO books (title, published, author_id) VALUES (?, ?, ?)
('The Left Hand of Darkness', '1969-01-01', 1)
COMMIT

BEGIN (implicit)
SELECT authors.id, authors.name FROM authors ORDER BY authors.id
SELECT books.author_id AS books_author_id, books.id AS books_id,
       books.title AS books_title, books.published AS books_published
FROM books WHERE books.author_id IN (?,)
[generated in 0.00015s] (1,)
ROLLBACK
```

Trzy rzeczy warte uwagi:

1. **`RETURNING id`** — SQLite (i PostgreSQL, i MariaDB) zwraca nowe `id` bez dodatkowego zapytania. To zasługa nowoczesnego `RETURNING`, obsługiwanego automatycznie przez SQLAlchemy 2.0.
2. **`SELECT ... IN (?,)`** — to `selectinload`, o którym więcej w sekcji 9.
3. **`ROLLBACK` na końcu** — sesja zamknęła się bez `commit`, więc otwarta transakcja czytająca została wycofana. To normalne i nie oznacza błędu.

> ⚠️ **Pułapka — brak `RETURNING` na MySQL 8.** Jeśli baza nie wspiera `RETURNING` (starsze MySQL), wartości generowane po stronie serwera (np. `server_default=func.now()`) **nie będą dostępne** na świeżo wstawionym obiekcie. Rozwiązanie: `Mapper.eager_defaults` w opcjach mapowania. Warto o tym pamiętać, bo w async nie można po prostu „doczytać” atrybutu — trzeba to zrobić jawnie przez `await session.refresh(obj)`.

> 🧪 **Ćwiczenie.** Uruchom powyższy przykład, a potem zmień `expire_on_commit=False` na `True` i dodaj na końcu bloku odczytu `print(authors[0].name)` **po** `await session.commit()`. Zobacz, jaki błąd się pojawi i przeczytaj jego treść.

---

## 6. DDL w async — `run_sync` i zielone wątki

Pierwsza rzecz, która zaskakuje wszystkich: **`Base.metadata.create_all` nie ma wersji asynchronicznej**.

```python
# examples/15_ddl_wrong.py
async with engine.begin() as conn:
    Base.metadata.create_all(conn)   # BŁĄD: to jest funkcja synchroniczna
```

Dlaczego? Bo `MetaData.create_all()` powstało lata przed asynchronicznością i jest zbudowane wokół synchronicznego `Connection`. Zamiast tworzyć drugą, asynchroniczną kopię każdej funkcji DDL, SQLAlchemy dostarcza jedno uniwersalne narzędzie: **`run_sync()`**.

```python
# examples/15_ddl_right.py
async with engine.begin() as conn:
    await conn.run_sync(Base.metadata.drop_all)
    await conn.run_sync(Base.metadata.create_all)
```

`run_sync(fn, *args)` mówi: *weź tę synchroniczną funkcję i uruchom ją w takim środowisku, w którym jej synchroniczne wywołania do sterownika zostaną automatycznie zamienione na `await`*.

> 💡 **Analogia — tłumacz symultaniczny na konferencji.** Funkcja `create_all` to prelegent, który mówi po „staremu” i nie zna nowego języka. `run_sync` to kabina tłumacza: prelegent mówi normalnie, a tłumacz na bieżąco przekłada każde zdanie na nowy język. Prelegent nawet nie wie, że mówi w innym języku.

To samo dotyczy refleksji schematu i inspektora:

```python
# examples/15_inspector.py
from sqlalchemy import inspect


def use_inspector(conn):
    inspector = inspect(conn)
    return inspector.get_table_names()


async def main() -> None:
    async with engine.connect() as conn:
        tables = await conn.run_sync(use_inspector)
        print(tables)
```

> 🧠 **Dlaczego tak jest — nie ma asynchronicznego `Inspector`.** Dokumentacja SQLAlchemy jest tu jednoznaczna: nie istnieje jeszcze asynchroniczna wersja `Inspector`. Ale można go użyć w kontekście async **właśnie dzięki `run_sync()`**.

### 6.1 Co jeszcze wymaga `run_sync`

| Operacja | Wymaga `run_sync`? |
|---|---|
| `Base.metadata.create_all` / `drop_all` | ✅ tak |
| `inspect(conn)` i metody inspektora | ✅ tak |
| `Base.metadata.reflect` | ✅ tak |
| `conn.execute(select(...))` | ❌ nie, `await` wystarczy |
| `conn.exec_driver_sql("...")` | ❌ nie, `await` wystarczy |
| Własne funkcje używające `session.execute` | ✅ tak, przez `AsyncSession.run_sync` |

> 🆕 **SQLAlchemy 2.1** — w tej serii pojawiły się nowe konstrukcje DDL, m.in. `CreateView` (tworzenie widoków) oraz `CREATE TABLE AS SELECT`. W trybie async korzystasz z nich przez `await conn.run_sync(...)` albo `await conn.execute(...)`, zależnie od tego, czy dana konstrukcja jest wyrażeniem wykonywalnym, czy funkcją DDL.

> 🧪 **Ćwiczenie.** Napisz funkcję `def dump_schema(conn) -> list[str]`, która zwraca nazwy kolumn tabeli `books` (`inspect(conn).get_columns("books")`), i wywołaj ją przez `await conn.run_sync(dump_schema)`.

---

## 7. Pod maską: `greenlet` i most sync ↔ async

To najtrudniejsza, ale i najciekawsza część modułu. Warto ją zrozumieć, bo bez tego `MissingGreenlet` będzie dla ciebie magicznym błędem.

### 7.1 Problem: SQLAlchemy ma dwie dekady kodu synchronicznego

SQLAlchemy Core i ORM to setki tysięcy linii kodu napisanych w stylu synchronicznym. Przepisanie tego na `async`/`await` oznaczałoby albo duplikację całej bazy kodu, albo wieloletni projekt. Zamiast tego SQLAlchemy zrobiło coś sprytnego: **udaje, że jest synchroniczna, a pod spodem przełącza się na `await`**.

Do tego potrzebna jest biblioteka `greenlet`.

> 💡 **Analogia — zielone wątki to „wątki współpracujące”.** Zwykłe wątki (`threading`) przełącza system operacyjny, w dowolnym momencie, bez pytania kodu o zgodę. `greenlet` to lekki, *współpracujący* wątek: przełączenie następuje tylko wtedy, gdy kod sam o to poprosi. To dokładnie ta właściwość, której potrzebuje SQLAlchemy: kod synchroniczny działa sobie spokojnie, a w momencie dotknięcia sterownika następuje kontrolowane przełączenie do pętli zdarzeń.

### 7.2 Jak to działa krok po kroku

```text
     Twój kod                 SQLAlchemy                 Sterownik
     (async def)              (sync, w greenlet)         (asyncpg/aiosqlite)
         │                          │                          │
         │  await session.execute() │                          │
         ├─────────────────────────►│                          │
         │                          │  tworzy greenlet,        │
         │                          │  wchodzi do niego        │
         │                          │                          │
         │                          │  cursor.execute()        │
         │                          ├─────────────────────────►│
         │                          │                          │  (blokujące
         │                          │                          │   wywołanie
         │                          │                          │   sterownika)
         │                          │  ┌───────────────────────┤
         │                          │  │ await_only()          │
         │                          │  │ = przełącz do pętli   │
         │  ◄───────────────────────┼──┘                       │
         │  pętla robi inne zadania │                          │
         │                          │                          │
         │                          │  ◄───────────────────────┤
         │                          │  dane wróciły            │
         ├─────────────────────────►│                          │
         │  wynik (Result)          │                          │
```

Kluczowe ogniwo to `await_only()`. To funkcja z `greenlet`, która w środku synchronicznego kodu **przełącza sterowanie do pętli zdarzeń i czeka na wynik korutyny**. Działa jednak **wyłącznie wtedy, gdy działamy wewnątrz greenleta utworzonego przez SQLAlchemy**.

> 🧠 **Dlaczego tak jest — i skąd `MissingGreenlet`.** `await_only()` sprawdza, czy istnieje aktywny kontekst greenleta (`greenlet_spawn`). Jeśli nie — nie ma jak oddać sterowania pętli, więc funkcja zgłasza błąd. Komunikat brzmi:

```text
MissingGreenlet: greenlet_spawn has not been called; can't call await_only() here.
Was IO attempted in an unexpected place?
```

To nie jest błąd SQLAlchemy. To **Twój kod próbował wykonać IO w miejscu, w którym nie ma greenleta** — czyli poza `await session.execute()` i poza `run_sync()`.

### 7.3 Zdarzenia i `sync_engine` — konsekwencja architektury

Skoro SQLAlchemy jest pod spodem synchroniczna, to **zdarzenia (events) też są synchroniczne**. Nie ma „asynchronicznych handlerów zdarzeń”.

| Chcesz podpiąć event do… | Użyj jako celu |
|---|---|
| `AsyncEngine` (instancja) | `engine.sync_engine` |
| `AsyncConnection` | `conn.sync_connection` lub `conn.sync_engine` |
| `AsyncSession` (instancja) | `session.sync_session` |
| Wszystkie `AsyncSession` | klasa `Session` |
| `async_sessionmaker` | `sessionmaker` + `async_sessionmaker(sync_session_class=...)` |

```python
# examples/15_events_async.py
from sqlalchemy import event
from sqlalchemy.ext.asyncio import create_async_engine

engine = create_async_engine("sqlite+aiosqlite:///library.db")


@event.listens_for(engine.sync_engine, "connect")
def on_connect(dbapi_connection, connection_record) -> None:
    """Zdarzenie synchroniczne — ale wywoływane w kontekście async."""
    print("Nowe połączenie DBAPI:", dbapi_connection)
```

Ważna konsekwencja praktyczna: **licznik zapytań z modułu 11 działa w async bez zmian**. Podpinamy go do `engine.sync_engine` i możemy liczyć zapytania w testach asynchronicznych.

```python
# examples/15_query_counter.py
from sqlalchemy import event
from sqlalchemy.ext.asyncio import AsyncEngine


class QueryCounter:
    """Licznik zapytań SQL — działa również w trybie async."""

    def __init__(self) -> None:
        self.queries: list[str] = []

    def attach(self, engine: AsyncEngine) -> None:
        @event.listens_for(engine.sync_engine, "before_cursor_execute")
        def _count(conn, cursor, statement, parameters, context, executemany) -> None:
            self.queries.append(statement)

    def reset(self) -> None:
        self.queries.clear()

    @property
    def count(self) -> int:
        return len(self.queries)
```

> ⚠️ **Pułapka — event, który potrzebuje `await`.** Jeśli handler zdarzenia musi wywołać asynchroniczną metodę sterownika (np. `set_type_codec()` w asyncpg), użyj `AdaptedConnection.run_async()`:

```python
@event.listens_for(engine.sync_engine, "connect")
def register_custom_types(dbapi_connection, *args) -> None:
    dbapi_connection.run_async(
        lambda connection: connection.set_type_codec(
            "MyCustomType", encoder, decoder, schema="pg_catalog"
        )
    )
```

To dokładny odpowiednik `run_sync()`, tylko w drugą stronę: pozwala wywołać coś `await`-owalnego z wnętrza synchronicznego handlera.

> 🧪 **Ćwiczenie.** Podłącz licznik zapytań do `engine.sync_engine`, wykonaj `select(Author).options(selectinload(Author.books))` i sprawdź, ile zapytań poleciało. Czy zgadza się z tym, co pokazywał moduł 11?

---

## 8. `MissingGreenlet` — dlaczego leniwe ładowanie wybucha

To jest błąd numer jeden w tym module. Zobaczmy go w działaniu.

```python
# examples/15_missing_greenlet.py
"""Reprodukcja błędu MissingGreenlet."""
from __future__ import annotations

import asyncio

from sqlalchemy import select
from sqlalchemy.ext.asyncio import async_sessionmaker, create_async_engine

from models import Author, Base  # modele z lazy="select" (domyślne)


async def main() -> None:
    engine = create_async_engine("sqlite+aiosqlite:///library.db")
    SessionLocal = async_sessionmaker(engine, expire_on_commit=False)

    async with SessionLocal() as session:
        # Brak eager loadingu — kolekcja .books NIE jest załadowana
        authors = (await session.scalars(select(Author))).all()

        for a in authors:
            # Ten wiersz próbuje wykonać SELECT bez await.
            print(a.books)
            # MissingGreenlet: greenlet_spawn has not been called;
            # can't call await_only() here. Was IO attempted in an
            # unexpected place?

    await engine.dispose()


asyncio.run(main())
```

### 8.1 Trzy scenariusze, które kończą się tym błędem

| Scenariusz | Co próbuje zrobić | Dlaczego brak greenleta |
|---|---|---|
| Leniwe ładowanie relacji (`a.books`) | `SELECT ... WHERE author_id = ?` | odczyt atrybutu, żadnego `await` w pobliżu |
| Odczyt unieważnionego atrybutu po `commit()` (`expire_on_commit=True`) | `SELECT ...` po `id` | odczyt atrybutu |
| Odczyt odroczonej kolumny (`deferred=True`) | `SELECT kolumna ...` | odczyt atrybutu |

Wspólny mianownik: **IO ukryte za zwykłym odwołaniem do atrybutu**.

> 🧠 **Dlaczego tak jest — filozofia „żadnego ukrytego IO”.** W świecie asynchronicznym panuje żelazna zasada: *każda linia kodu, która może wykonać IO, musi być widoczna jako `await`*. Leniwe ładowanie łamie tę zasadę — IO dzieje się przy `print(a.books)`. SQLAlchemy wybiera więc głośny błąd zamiast cichego blokowania pętli. To dobra decyzja, choć bolesna dla kodu przenoszonego z wersji synchronicznej.

> ⚠️ **Pułapka — leniwe ładowanie w `__repr__`.** To najbardziej podstępny wariant tego błędu. Jeśli masz:

```python
def __repr__(self) -> str:
    return f"<Author {self.name} books={len(self.books)}>"
```

…to **każde** `print(author)` — a także logowanie, debugger, a nawet komunikat błędu — może wywołać `MissingGreenlet`. Zalecenie: w modelach używanych asynchronicznie `__repr__` powinno odwoływać się wyłącznie do kolumn własnej tabeli, nigdy do relacji.

```python
def __repr__(self) -> str:
    # BEZPIECZNE: tylko kolumny własnej tabeli
    return f"Author(id={self.id!r}, name={self.name!r})"
```

### 8.2 Jak czytać ten błąd

```text
MissingGreenlet: greenlet_spawn has not been called; can't call await_only() here.
Was IO attempted in an unexpected place?
```

Czytamy go tak:

1. **„Was IO attempted”** — SQLAlchemy pyta: czy Twój kod próbował wykonać operację wejścia/wyjścia?
2. **„in an unexpected place”** — czyli poza `await session.execute()` i poza `run_sync()`.
3. **Gdzie szukać:** w ostatniej linii Twojego kodu przed stack trace. Prawie zawsze jest to odczyt relacji, odczyt odroczonej kolumny albo odczyt po `commit()`.

> 🧪 **Ćwiczenie.** W przykładzie powyżej zamień `print(a.books)` na `print(a.name)`. Błąd znika, bo `name` to zwykła kolumna załadowana razem z obiektem. Zastanów się, dlaczego `a.author_id` też działa, a `a.author` już nie.

---

## 9. Trzy strategie radzenia sobie z relacjami w async

Dokumentacja SQLAlchemy wymienia trzy podejścia. Omówmy je w kolejności od najbardziej zalecanego.

### 9.1 Strategia A: eager loading (zalecana)

Zasada: **ładuj wszystko, czego potrzebujesz, w jednym `await`**.

```python
# examples/15_eager.py
from sqlalchemy import select
from sqlalchemy.orm import selectinload

stmt = (
    select(Author)
    .options(selectinload(Author.books))     # kolekcja -> selectinload
    .order_by(Author.id)
)
authors = (await session.scalars(stmt)).all()

for a in authors:
    print(a.name, [b.title for b in a.books])   # bez IO, bez MissingGreenlet
```

Dlaczego `selectinload`, a nie `joinedload`? Bo w async wygrywa strategia, która nie mnoży wierszy i nie wymaga `unique()`. `selectinload` wykonuje jedno dodatkowe zapytanie z `IN (...)`, co jest przewidywalne i wydajne. `joinedload` też działa, ale przy kolekcjach trzeba pamiętać o `.unique()`:

```python
from sqlalchemy.orm import joinedload

stmt = select(Author).options(joinedload(Author.books))
authors = (await session.scalars(stmt)).unique().all()   # .unique() obowiązkowe!
```

> 🔬 **Pod maska — `selectinload` vs `joinedload`.**

```sql
-- selectinload: 2 zapytania
SELECT authors.id, authors.name FROM authors ORDER BY authors.id
SELECT books.author_id AS books_author_id, books.id AS books_id, books.title AS books_title
FROM books WHERE books.author_id IN (?, ?, ?)

-- joinedload: 1 zapytanie, ale wiersze się powtarzają
SELECT authors.id, authors.name, books_1.id, books_1.title, books_1.author_id
FROM authors LEFT OUTER JOIN books AS books_1 ON authors.id = books_1.author_id
ORDER BY authors.id
```

Przy `joinedload` autor z trzema książkami pojawi się w wyniku trzy razy — dlatego `.unique()` jest konieczne, żeby SQLAlchemy skleiło to z powrotem w jeden obiekt.

> 🆕 **SQLAlchemy 2.1** — `selectinload()` zyskuje dwa usprawnienia: parametr `chunksize` (dzielenie `IN (...)` na porcje — przydatne, gdy lista identyfikatorów jest ogromna i przekracza limit parametrów sterownika) oraz `omit_join` dla relacji wiele-do-wielu, który potrafi pominąć niepotrzebne złączenie i wykonać `selectinload` jeszcze taniej.

### 9.2 Strategia B: `AsyncAttrs` i `awaitable_attrs`

Czasem naprawdę nie wiesz z góry, czego będziesz potrzebować. Wtedy z pomocą przychodzi mixin `AsyncAttrs`, dodany w **SQLAlchemy 2.0.13**.

```python
# examples/15_async_attrs.py
from sqlalchemy.ext.asyncio import AsyncAttrs
from sqlalchemy.orm import DeclarativeBase


class Base(AsyncAttrs, DeclarativeBase):
    """Baza z obsługą awaitable_attrs."""
```

Teraz każdy atrybut można odczytać jako `await`-owalny:

```python
# examples/15_awaitable_attrs.py
author = await session.get(Author, 1)

# Zamiast a.books (co rzuciłoby MissingGreenlet):
books = await author.awaitable_attrs.books
for b in books:
    print(b.title)
```

`awaitable_attrs` to **fasada nad tym samym mechanizmem, którego używa `run_sync()`** — czyli SQLAlchemy tworzy greenlet, wykonuje leniwe ładowanie i oddaje sterowanie. IO jest tu jawne (widzisz `await`), więc zasada „żadnego ukrytego IO” jest zachowana.

> ⚠️ **Pułapka — `awaitable_attrs` to nie darmowy optymizm.** Każde `await author.awaitable_attrs.books` w pętli to **osobne zapytanie** — czyli klasyczny N+1 z modułu 11, tylko ubrany w `await`. Jeśli iterujesz po 100 autorach, zrobisz 100 zapytań. `selectinload` zrobi jedno.

### 9.3 Strategia C: `lazy="raise"` (bezpieczna postawa w projekcie)

Najbezpieczniejsze podejście w projekcie async: **zabroń leniwego ładowania u źródła**.

```python
# examples/15_lazy_raise.py
class Author(Base):
    __tablename__ = "authors"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(120))

    books: Mapped[list[Book]] = relationship(
        back_populates="author",
        lazy="raise",      # <- próba leniwego ładowania = czytelny błąd
    )
```

Teraz `author.books` bez wcześniejszego eager loadingu zgłosi `InvalidRequestError` z komunikatem w stylu *„lazy load operation of attribute 'books' cannot proceed”* — a nie tajemniczy `MissingGreenlet`. To **znacznie lepszy komunikat błędu**: od razu mówi, co zrobić.

Dodatkowo w testach można użyć globalnego zabezpieczenia:

```python
# tests/test_lazy_guard.py
from sqlalchemy import select
from sqlalchemy.orm import raiseload

stmt = select(Author).options(raiseload("*"))   # żadna relacja nie może się leniwie ładować
```

> 💡 **Analogia — gaśnica zamiast alarmu.** `MissingGreenlet` to alarm przeciwpożarowy, który wyje po całym budynku i nie mówi, gdzie się pali. `lazy="raise"` to gaśnica przy drzwiach z etykietą „tu może się palić”. Obie są potrzebne, ale ta druga oszczędza czas.

### 9.4 Kolekcje „tylko do zapisu”

Jeśli kolekcja jest bardzo duża i nigdy nie chcesz jej ładować, użyj `lazy="write_only"`:

```python
# examples/15_write_only.py
from sqlalchemy.orm import WriteOnlyMapped

class Author(Base):
    __tablename__ = "authors"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(120))

    books: WriteOnlyMapped[Book] = relationship(back_populates="author")
```

Taka kolekcja **nigdy nie wykonuje IO niejawnie** — można do niej dopisywać (`author.books.add(book)`), a odczytywać trzeba jawnym zapytaniem (`select(Book).where(Book.author_id == author.id)`). To najbardziej „async-friendly” rozwiązanie, bo całkowicie usuwa problem ukrytego IO.

> 🧠 **Dlaczego `write_only` pasuje do async jak ulał.** Dokumentacja SQLAlchemy nazywa tę cechę „fully compatible with asyncio” i „should be preferred” nad starymi relacjami `lazy="dynamic"`. Stare `lazy="dynamic"` **nie działa** w async domyślnie — można je wywołać tylko przez `run_sync()` albo przez atrybut `.statement`:

```python
# Relikt 1.x — obejście, NIE zalecane
user = await session.get(User, 42)
addresses = (await session.scalars(user.addresses.statement)).all()
```

> ⚠️ **Pułapka — kaskada `all` w async.** Dokumentacja ostrzega: unikaj `cascade="all"`, bo zawiera `refresh-expire`. Przy `AsyncSession.refresh()` oznacza to, że obiekty powiązane zostaną **unieważnione**, ale niekoniecznie odświeżone — zostaną w stanie „expired”, a ich odczyt wywoła `MissingGreenlet`. Zamiast `all` wymień kaskady jawnie: `cascade="save-update, merge, delete"`.

> 🧪 **Ćwiczenie.** Zmień w `models.py` relację `Author.books` z `lazy="raise"` na `lazy="select"` i uruchom skrypt z sekcji 8.1. Następnie zmień z powrotem i dodaj `selectinload`. Porównaj komunikaty błędów i wyciągnij wniosek, który jest bardziej pomocny.

---

## 10. Strumieniowanie dużych zbiorów

Przy dużych zbiorach `(await session.scalars(stmt)).all()` ładuje **wszystko do pamięci**. Przy milionie wierszy to koniec aplikacji. Rozwiązanie: **strumieniowanie** — czytanie wyników partiami, tak jak czyta się długą książkę rozdział po rozdziale, a nie zjadając ją jednym kęsem.

> 💡 **Analogia — taśma produkcyjna.** `all()` to załadowanie całej ciężarówki cegieł na raz. `stream()` to taśma: cegły jadą jedna po drugiej, a Ty bierzesz tyle, ile aktualnie potrzebujesz.

### 10.1 `stream()` i `stream_scalars()`

```python
# examples/15_streaming.py
from __future__ import annotations

import asyncio

from sqlalchemy import select
from sqlalchemy.ext.asyncio import async_sessionmaker, create_async_engine

from models import Author, Book


async def main() -> None:
    engine = create_async_engine("sqlite+aiosqlite:///library.db")
    SessionLocal = async_sessionmaker(engine, expire_on_commit=False)

    async with SessionLocal() as session:
        stmt = select(Book).order_by(Book.id)

        # stream_scalars -> AsyncScalarResult, obiekty ORM
        result = await session.stream_scalars(stmt)
        async for book in result:
            print(book.title)
            # ... tu przetwarzasz jeden wiersz naraz, pamięć nie rośnie

    await engine.dispose()


asyncio.run(main())
```

Dla wyników wielokolumnowych użyj `stream()`:

```python
# examples/15_stream_rows.py
from sqlalchemy import func, select

stmt = (
    select(Book.author_id, func.count().label("books_count"))
    .group_by(Book.author_id)
    .order_by(Book.author_id)
)

result = await session.stream(stmt)
async for row in result:
    print(row.author_id, row.books_count)   # Row — dostęp po nazwie
```

### 10.2 Wariant z `async with` — automatyczne zamknięcie

Dokumentacja SQLAlchemy pokazuje, że `stream()` można użyć jako **asynchronicznego kontekst menedżera** — wtedy `AsyncResult.close()` zostanie wywołane **bezwarunkowo, nawet jeśli iteracja zostanie przerwana wyjątkiem**:

```python
# examples/15_stream_context.py
async with session.stream(stmt) as result:
    async for row in result:
        if row.author_id > 100:
            break     # wcześniejsze przerwanie — wynik zostanie zamknięty
```

To istotne, bo przerwanie iteracji `break`-em bez zamknięcia wyniku trzyma otwarty kursor po stronie serwera i blokuje połączenie z puli.

> ⚠️ **Pułapka — iteracja po wyjściu z bloku.** Jeśli spróbujesz iterować po `AsyncResult` **po** opuszczeniu bloku `async with`, dostaniesz `InvalidRequestError`, bo połączenie wróciło już do puli. Wynik strumieniowy żyje tylko wewnątrz swojego bloku.

### 10.3 `yield_per` w async

`yield_per` mówi sterownikowi, ile wierszy pobierać naraz. Można go ustawić na dwa sposoby:

```python
# Sposób 1: przez execution_options w zapytaniu
stmt = select(Book).order_by(Book.id).execution_options(yield_per=500)

# Sposób 2: przez metodę na wyniku (również na AsyncResult)
result = await session.stream_scalars(stmt)
result = result.yield_per(500)
```

Uwaga na różnicę: `yield_per` na **zapytaniu** wpływa na sposób pobierania wierszy ze sterownika; `yield_per` na **wyniku** ustawia dodatkowo partycjonowanie wyników ORM (pomaga uniknąć kumulacji obiektów w identity map).

```python
# examples/15_partitions.py
result = await session.stream_scalars(select(Book).order_by(Book.id))
async for partition in result.partitions(500):
    for book in partition:
        process(book)     # 500 obiektów naraz, potem pamięć się zwalnia
```

> 🧠 **Dlaczego `partitions()` pomaga w ORM.** Przy strumieniowaniu SQLAlchemy domyślnie trzyma obiekty w **identity map** sesji. Przy milionie wierszy identity map rośnie do miliona obiektów — i pamięć znów eksploduje. `partitions(n)` (albo `yield_per(n)`) mówi: po każdych `n` wierszach „zapomnij” już przetworzone obiekty.

> ⚠️ **Pułapka — `yield_per` + `joinedload` kolekcji.** Ta kombinacja jest niepoprawna: `joinedload` zwielokrotnia wiersze, więc „partia 500 wierszy” może zawierać tylko 100 unikalnych obiektów. SQLAlchemy zgłosi błąd `InvalidRequestError`. W trybie strumieniowym używaj `selectinload`, a nie `joinedload` na kolekcjach.

> 🧪 **Ćwiczenie.** Wygeneruj 100 000 wierszy do tabeli `books` (skrypt w sekcji 16 pokazuje, jak), a następnie porównaj zużycie pamięci między `(await session.scalars(stmt)).all()` a pętlą `async for` po `stream_scalars`. Użyj `tracemalloc`.

---

## 11. `AsyncEngine` i `Engine` w jednym projekcie

Czy w jednym projekcie można mieć **oba** silniki? Tak — i to często najlepsze rozwiązanie.

```python
# examples/15_both_engines.py
"""Ten sam URL, dwa silniki: sync dla narzędzi, async dla aplikacji."""
from sqlalchemy import create_engine
from sqlalchemy.ext.asyncio import create_async_engine

# Aplikacja webowa / worker — async
async_engine = create_async_engine("postgresql+asyncpg://app:secret@localhost/library")

# CLI, skrypty administracyjne, Alembic — sync
sync_engine = create_engine("postgresql+psycopg://app:secret@localhost/library")
```

### 11.1 Kiedy używać obu

| Kontekst | Tryb | Uzasadnienie |
|---|---|---|
| Serwer HTTP (FastAPI, aiohttp) | async | Setki równoległych żądań, IO to wąskie gardło |
| Worker kolejki zadań | async | Równoległe przetwarzanie niezależnych zadań |
| CLI, skrypty migracyjne | sync | Prostszy kod, brak zysku z async |
| Alembic | sync (lub `-t async`) | Migracje są sekwencyjne — async nic nie daje |
| Testy jednostkowe | sync (szybciej) | Mniej boilerplate'u, krótszy czas startu |
| Testy integracyjne ścieżek async | async | Muszą odzwierciedlać produkcję |

> 💡 **Analogia — dwie kuchnie w restauracji.** Sala (aplikacja webowa) pracuje asynchronicznie, bo kelnerzy obsługują wielu gości. Ale magazyn (CLI, migracje) pracuje sekwencyjnie — jeden pracownik, jedna lista zadań, zero potrzeby przełączania kontekstu.

### 11.2 Współdzielenie konfiguracji

Najlepsza praktyka: **jedno źródło prawdy o konfiguracji**, dwa silniki z niego zbudowane.

```python
# examples/15_config.py
from __future__ import annotations

import os
from dataclasses import dataclass

from sqlalchemy import Engine, create_engine
from sqlalchemy.ext.asyncio import AsyncEngine, create_async_engine


@dataclass(frozen=True)
class DbConfig:
    """Konfiguracja bazy — jedno miejsce, dwa tryby."""

    host: str
    port: int
    user: str
    password: str
    database: str

    @property
    def async_url(self) -> str:
        return (
            f"postgresql+asyncpg://{self.user}:{self.password}"
            f"@{self.host}:{self.port}/{self.database}"
        )

    @property
    def sync_url(self) -> str:
        return (
            f"postgresql+psycopg://{self.user}:{self.password}"
            f"@{self.host}:{self.port}/{self.database}"
        )

    @classmethod
    def from_env(cls) -> "DbConfig":
        return cls(
            host=os.environ.get("DB_HOST", "localhost"),
            port=int(os.environ.get("DB_PORT", "5432")),
            user=os.environ["DB_USER"],
            password=os.environ["DB_PASSWORD"],
            database=os.environ.get("DB_NAME", "library"),
        )


def make_async_engine(cfg: DbConfig, *, echo: bool = False) -> AsyncEngine:
    return create_async_engine(
        cfg.async_url,
        echo=echo,
        pool_pre_ping=True,
        pool_size=10,
        max_overflow=20,
    )


def make_sync_engine(cfg: DbConfig, *, echo: bool = False) -> Engine:
    return create_engine(cfg.sync_url, echo=echo, pool_pre_ping=True)
```

To rozwiązanie ma jeszcze jedną zaletę: **Alembic** może korzystać z `sync_url`, a aplikacja z `async_url` — obie z tego samego `DbConfig`.

> ⚠️ **Pułapka — wiele pętli zdarzeń i jeden `AsyncEngine`.** Dokumentacja SQLAlchemy jest tu jednoznaczna: nie należy współdzielić tego samego `AsyncEngine` między różnymi pętlami zdarzeń (dotyczy to np. łączenia asyncio z wielowątkowością). Jeśli silnik przechodzi z jednej pętli do drugiej, **najpierw** wywołaj `await engine.dispose()`. W przeciwnym razie dostaniesz `RuntimeError: Task got Future attached to a different loop`. Jeśli silnik **musi** być współdzielony między pętlami, skonfiguruj go z `poolclass=NullPool`, żeby żadne połączenie nie było używane dwa razy.

> 🆕 **SQLAlchemy 2.1** — wymagany jest Python ≥ 3.11. Jeśli planujesz używać wolnych wątków (free-threaded builds Pythona), pamiętaj, że `AsyncSession` nadal **nie jest bezpieczna** dla współbieżnych zadań — zasada „jedna sesja na zadanie” obowiązuje niezależnie od wersji Pythona.

---

## 12. Anulowanie, timeouty i sesja po błędzie

W asynchroniczności każde zadanie można **anulować**. To potężne narzędzie — i źródło subtelnych błędów.

### 12.1 `asyncio.timeout()` (Python 3.11+)

```python
# examples/15_timeout.py
from __future__ import annotations

import asyncio

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker


async def fetch_authors(
    session_factory: async_sessionmaker[AsyncSession],
    seconds: float = 2.0,
) -> list[str]:
    """Pobiera autorów, ale nie czeka dłużej niż `seconds` sekund."""
    async with session_factory() as session:
        try:
            async with asyncio.timeout(seconds):
                result = await session.scalars(select(Author.name))
                return result.all()
        except TimeoutError:
            # Sesja mogła zostać przerwana w połowie operacji —
            # trzeba ją wyczyścić, zanim zostanie zamknięta.
            await session.rollback()
            raise
```

Trzy rzeczy warte podkreślenia:

1. **`asyncio.timeout()` to kontekst menedżer** (od Pythona 3.11). Wcześniej używało się `asyncio.wait_for()`.
2. **`TimeoutError`** — od Pythona 3.11 `asyncio.TimeoutError` jest aliasem wbudowanego `TimeoutError`. Możesz łapać ten drugi.
3. **`await session.rollback()` w `except`** — to nie jest kosmetyka, o czym za chwilę.

### 12.2 Co się dzieje z sesją po anulowaniu

> 🧠 **Dlaczego tak jest — anulowanie przychodzi w dowolnym miejscu.** `CancelledError` może zostać wstrzyknięte w sesję **w środku** wysyłania zapytania, w środku `flush`, w środku `commit`. Sesja zostaje wtedy w stanie „niedokończonej transakcji” i każda kolejna operacja zgłosi `PendingRollbackError` lub `InvalidRequestError: This session is in 'prepared' state`. Jedynym wyjściem jest jawne wyczyszczenie stanu.

Wzorzec bezpieczny:

```python
# examples/15_cancellation.py
async def safe_operation(session_factory) -> None:
    session = session_factory()
    try:
        async with session.begin():
            await do_something(session)
    except (TimeoutError, asyncio.CancelledError):
        await session.rollback()   # wyczyść stan
        raise
    finally:
        await session.close()      # zawsze zwolnij połączenie do puli
```

> ⚠️ **Pułapka — anulowanie a `finally` bez `await`.** W `finally` bloku asynchronicznego musisz użyć `await session.close()`, a nie `session.close()`. Zwykłe `session.close()` zwróci obiekt korutyny, która nigdy się nie wykona — a połączenie nie wróci do puli. To klasyczny **wyciek połączeń**, który objawia się dopiero po kilkudziesięciu minutach jako `TimeoutError: QueuePool limit of size 10 overflow 20 reached`.

### 12.3 `statement_timeout` i `lock_timeout` — timeouty po stronie bazy

`asyncio.timeout()` anuluje **Twoje** oczekiwanie. Ale zapytanie po stronie PostgreSQL nadal może biec i trzymać blokady. Dlatego warto ustawić timeout również w bazie:

```python
# examples/15_db_timeouts.py
from sqlalchemy import text
from sqlalchemy.ext.asyncio import create_async_engine

engine = create_async_engine(
    "postgresql+asyncpg://app:secret@localhost/library",
    connect_args={
        "server_settings": {
            "statement_timeout": "5000",    # 5 s na zapytanie
            "lock_timeout": "2000",         # 2 s na zdobycie blokady
            "idle_in_transaction_session_timeout": "10000",
        }
    },
)
```

Alternatywnie można ustawić je per transakcja:

```python
async with engine.begin() as conn:
    await conn.execute(text("SET LOCAL statement_timeout = '3s'"))
```

> 🧠 **Dlaczego to ma znaczenie bardziej w async niż w sync.** W aplikacji synchronicznej wolne zapytanie blokuje jeden wątek. W asynchronicznej wolne zapytanie blokuje **jedno połączenie z puli** — a tych jest `pool_size + max_overflow`. Kilka zawieszonych zapytań potrafi wyczerpać całą pulę i zatrzymać całą aplikację, mimo że pętla zdarzeń jest wolna.

> 🧪 **Ćwiczenie.** Napisz funkcję, która wykonuje `select(pg_sleep(5))` na PostgreSQL z `asyncio.timeout(1)`, a następnie sprawdź, w jakim stanie jest sesja po przechwyceniu `TimeoutError` — przed i po `await session.rollback()`.

---

## 13. Współbieżność: `gather`, `TaskGroup` i `Semaphore`

To sekcja, w której async zaczyna zarabiać na siebie.

### 13.1 Zasada numer jeden: **jedna sesja na zadanie**

Dokumentacja SQLAlchemy mówi to wprost i z ostrzeżeniem:

> A single instance of `AsyncSession` is not safe for use in multiple, concurrent tasks.

`AsyncSession` to **mutowalny, stanowy obiekt** reprezentujący **jedną** transakcję. Współdzielenie go między równoległe zadania to jak danie jednemu kelnerowi dwudziestu stolików **z jednym notesem i jedną krótkofalówką** — zamówienia się pomieszają.

```python
# examples/15_gather_wrong.py
"""ANTYWZORZEC: jedna sesja w wielu zadaniach."""
import asyncio

from sqlalchemy import func, select


async def bad_parallel(session: AsyncSession) -> list[int]:
    async def one_query() -> int:
        # Ta sama sesja używana równolegle!
        return await session.scalar(select(func.count()).select_from(Book))

    return await asyncio.gather(*(one_query() for _ in range(20)))
    # InvalidRequestError: This session is provisioning a new connection;
    # concurrent operations are not permitted
```

```python
# examples/15_gather_right.py
"""WZORZEC: sesja per zadanie."""
import asyncio

from sqlalchemy import func, select
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker


async def good_parallel(
    session_factory: async_sessionmaker[AsyncSession],
    n: int = 20,
) -> list[int]:
    async def one_query() -> int:
        async with session_factory() as session:      # własna sesja
            return await session.scalar(select(func.count()).select_from(Book))

    return list(await asyncio.gather(*(one_query() for _ in range(n))))
```

> 🧠 **Dlaczego to działa.** Każde zadanie dostaje **własną sesję**, więc własną transakcję i własne połączenie z puli. Pętla zdarzeń przełącza je w momentach `await`, a pula ogranicza liczbę jednoczesnych połączeń. To cała magia — żadnej magii nie ma.

> ⚠️ **Pułapka — SQLAlchemy nie zawsze Cię złapie.** W niektórych scenariuszach współdzielona sesja nie zgłosi błędu, tylko zwróci **błędne wyniki** albo pomiesza transakcje. Dlatego reguła „jedna sesja na zadanie” musi być dyscypliną zespołu, nie tylko nadzieją na błąd.

### 13.2 `asyncio.gather()` vs `asyncio.TaskGroup`

| Cecha | `gather()` | `TaskGroup` (Python 3.11+) |
|---|---|---|
| Składnia | `await asyncio.gather(*coros)` | `async with asyncio.TaskGroup() as tg:` |
| Wyniki | zwracane jako lista | przez `task.result()` |
| Zachowanie przy błędzie | pozostałe zadania **biegną dalej** | pozostałe zadania są **anulowane** |
| Obsługa wyjątków | `return_exceptions=True` | `ExceptionGroup` |
| Zalecenie | proste przypadki | nowy kod, bezpieczniejsze domyślne |

```python
# examples/15_taskgroup.py
import asyncio

from sqlalchemy import func, select
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker


async def fetch_all(session_factory: async_sessionmaker[AsyncSession]) -> list[int]:
    results: list[int] = []

    async with asyncio.TaskGroup() as tg:
        tasks = [
            tg.create_task(count_books(session_factory)) for _ in range(20)
        ]
    # Po wyjściu z bloku wszystkie zadania są zakończone
    # (albo wyjątek przerwał całą grupę).

    results.extend(t.result() for t in tasks)
    return results


async def count_books(session_factory: async_sessionmaker[AsyncSession]) -> int:
    async with session_factory() as session:
        return await session.scalar(select(func.count()).select_from(Book)) or 0
```

> 💡 **Analogia — grupa wycieczkowa.** `gather()` to wycieczka bez przewodnika: jeśli jedna osoba się zgubi, reszta idzie dalej. `TaskGroup` to wycieczka z przewodnikiem: jeśli ktoś się zgubi, cała grupa się zatrzymuje i wraca po niego. W aplikacji zwykle wolisz to drugie.

### 13.3 Ograniczanie równoległości — `Semaphore` i pula połączeń

Uruchomienie 1000 zadań naraz **nie znaczy** 1000 równoległych połączeń — pula (`pool_size=10`, `max_overflow=20`) wpuści maksymalnie 30. Reszta będzie czekać na `pool_timeout` (domyślnie 30 s), a potem poleci `TimeoutError`.

Często lepiej **samemu** ograniczyć równoległość:

```python
# examples/15_semaphore.py
from __future__ import annotations

import asyncio
from collections.abc import Sequence

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker


async def fetch_books_for_authors(
    session_factory: async_sessionmaker[AsyncSession],
    author_ids: Sequence[int],
    *,
    concurrency: int = 5,
) -> list[list[str]]:
    """Pobiera książki autorów, ale nie więcej niż `concurrency` zapytań naraz."""
    semaphore = asyncio.Semaphore(concurrency)

    async def one(author_id: int) -> list[str]:
        async with semaphore:                    # <- ogranicznik
            async with session_factory() as session:
                stmt = select(Book.title).where(Book.author_id == author_id)
                return list((await session.scalars(stmt)).all())

    async with asyncio.TaskGroup() as tg:
        tasks = [tg.create_task(one(aid)) for aid in author_ids]

    return [t.result() for t in tasks]
```

> 🧠 **Dlaczego `Semaphore` a nie tylko pula.** Pula ogranicza liczbę **połączeń**, ale nie liczbę **zadań**. Przy 1000 zadań czekających na połączenie rośnie zużycie pamięci (1000 korutyn, 1000 kontekstów) i rośnie ryzyko `pool_timeout`. `Semaphore` ogranicza liczbę zadań **przed** sięgnięciem po połączenie — czyli ogranicza całą kolejkę, a nie tylko jej końcówkę.

> ⚠️ **Pułapka — `Semaphore` trzymany przez cały blok `async with session_factory()`.** W powyższym kodzie semafor obejmuje całe zapytanie — to zamierzone. Ale gdybyś trzymał semafor podczas **długiego przetwarzania** danych poza bazą, blokowałbyś dostęp innym zadaniom. Zwalniaj semafor jak najszybciej — najlepiej tuż po pobraniu danych.

> 🧪 **Ćwiczenie.** Uruchom `fetch_books_for_authors` z `concurrency=1`, `5` i `20`, mierząc czas wykonania dla 50 autorów. Wyjaśnij, dlaczego `concurrency=20` nie jest dwa razy szybsze od `concurrency=10`, jeśli pula ma `pool_size=10` i `max_overflow=20`.

---

## 14. Wydajność: kiedy async naprawdę pomaga

Czas na uczciwe liczby. Poniższe wartości są **przykładowe i rzędu wielkości** — zależą od sprzętu, sieci, bazy i sterownika. Ale proporcje są pouczające.

### 14.1 Dla jednego zapytania: async jest wolniejszy

| Wariant | Czas na 1 zapytanie | Uwaga |
|---|---|---|
| sync (psycopg) | ~1.20 ms | prosty, bez narzutu |
| async (asyncpg) | ~1.35 ms | + narzut pętli i greenleta |

Różnica jest niewielka, ale **spójna**: przy pojedynczym zapytaniu async przegrywa. Nie ma tu nic do wygrania — jest tylko jeden klient i jedno czekanie.

> ⚠️ **Pułapka — „przepiszę na async, będzie szybciej”.** Przepisanie istniejącej, sekwencyjnej aplikacji na async **bez** zwiększenia równoległości zapytań da **zero zysku albo regresję**. Zysk pojawia się tylko tam, gdzie naprawdę jest współbieżność.

### 14.2 Dla wielu zapytań: różnica jest dramatyczna

Zakładając 100 zapytań, każde po 10 ms czasu bazy:

| Wariant | Czas całkowity | Obliczenie |
|---|---|---|
| sync, sekwencyjnie | ~1.00 s | $100 \times 10\,\text{ms}$ |
| async, sekwencyjnie (`await` w pętli) | ~1.05 s | prawie to samo + narzut |
| async, równolegle (`gather`, 20 zadań) | ~0.06 s | $\lceil 100/20 \rceil \times 10\,\text{ms}$ |
| async, równolegle (bez limitu) | ~0.01 s | ograniczone pulą i CPU bazy |

> 🧠 **Dlaczego sekwencyjny async nie pomaga.** To najczęstsze nieporozumienie. `await` w pętli `for` **nie tworzy równoległości** — po prostu grzecznie czeka, oddając sterowanie pętli, która… nie ma nic innego do roboty. Równoległość trzeba **jawnie stworzyć** (`gather`, `TaskGroup`, `create_task`).

```python
# examples/15_sequential_vs_parallel.py
"""Ilustracja różnicy między await w pętli a równoległością."""
from __future__ import annotations

import asyncio
import time

from sqlalchemy import func, select
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker


async def one_query(session: AsyncSession) -> int:
    return await session.scalar(select(func.count()).select_from(Book)) or 0


async def sequential(factory: async_sessionmaker[AsyncSession], n: int = 20) -> float:
    """Jedna sesja, jedno zapytanie po drugim."""
    start = time.perf_counter()
    async with factory() as session:
        for _ in range(n):
            await one_query(session)
    return time.perf_counter() - start


async def parallel(factory: async_sessionmaker[AsyncSession], n: int = 20) -> float:
    """Sesja per zadanie, wszystkie zapytania naraz."""
    start = time.perf_counter()

    async def task() -> int:
        async with factory() as session:
            return await one_query(session)

    async with asyncio.TaskGroup() as tg:
        for _ in range(n):
            tg.create_task(task())

    return time.perf_counter() - start
```

### 14.3 Kiedy NIE używać async

| Sytuacja | Wybór |
|---|---|
| Aplikacja CLI wykonująca 5 zapytań | sync — prościej i wystarczająco szybko |
| Skrypt analityczny jednorazowy | sync — async nic nie daje |
| Zespół nie zna `async`/`await` | sync — krzywa uczenia się async jest stroma |
| Biblioteki zależne wymagają `Session` sync | sync — albo adapter, ale to koszt |
| Aplikacja webowa z setkami żądań | **async** |
| Worker przetwarzający tysiące niezależnych zadań | **async** |
| Mikroserwis z dużą liczbą równoległych wywołań do innych serwisów i bazy | **async** |

> 💡 **Analogia — wybór między rowerem a ciężarówką.** Rower (sync) jest prostszy, tańszy i świetny na krótkich trasach. Ciężarówka (async) przewiezie 10 ton, ale potrzebujesz prawa jazdy, paliwa i serwisu. Nie kupuj ciężarówki, żeby zawieźć list na pocztę.

> 🧪 **Ćwiczenie.** Uruchom `sequential()` i `parallel()` z `n=20` na swojej bazie. Zapisz oba czasy i policz przyspieszenie. Następnie zmień `pool_size` na `2` i powtórz — wyjaśnij wynik.

---

## 15. Alembic i testy w trybie async

### 15.1 Alembic w trybie async

Migracje są operacją sekwencyjną i **nie potrzebują** async. Masz dwa wyjścia:

**Wyjście A (zalecane): użyj silnika synchronicznego w `env.py`.** Wystarczy, że `env.py` zbuduje URL synchroniczny. Wtedy działa standardowa, dobrze znana konfiguracja z modułu 16.

**Wyjście B: szablon async.** Alembic dostarcza gotowy szablon:

```bash
alembic init -t async migrations
```

Wygenerowany `env.py` zawiera (w istotnej części):

```python
# migrations/env.py (fragment — tryb async)
import asyncio

from sqlalchemy import pool
from sqlalchemy.engine import Connection
from sqlalchemy.ext.asyncio import async_engine_from_config


def do_run_migrations(connection: Connection) -> None:
    context.configure(connection=connection, target_metadata=target_metadata)
    with context.begin_transaction():
        context.run_migrations()


async def run_async_migrations() -> None:
    connectable = async_engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,          # <- brak puli w migracjach
    )
    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)   # <- most greenlet
    await connectable.dispose()


def run_migrations_online() -> None:
    asyncio.run(run_async_migrations())
```

> 🧠 **Dlaczego `run_sync(do_run_migrations)`.** Dokładnie ten sam mechanizm, co w sekcji 6: `context.run_migrations()` to funkcja synchroniczna, którą trzeba uruchomić w kontekście greenleta, żeby jej wewnętrzne wywołania sterownika mogły zostać zamienione na `await`.

> ⚠️ **Pułapka — `NullPool` w `env.py`.** W szablonie async pojawia się `poolclass=pool.NullPool`. To celowe: migracja otwiera jedno połączenie, kończy je i zamyka. Trzymanie puli w procesie migracyjnym jest zbędne, a przy `asyncio.run()` tworzącym i zamykającym pętlę — wręcz szkodliwe (połączenia z martwej pętli).

### 15.2 Testy asynchroniczne

```bash
pip install pytest pytest-asyncio
```

Konfiguracja w `pyproject.toml`:

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"          # testy async def działają bez dekoratora
```

> ⚠️ **Pułapka — tryb `auto` a `pytest-asyncio` w nowych wersjach.** Przy `asyncio_mode = "auto"` **fixtury asynchroniczne** i tak wymagają dekoratora `@pytest_asyncio.fixture` (nie `@pytest.fixture`). Bez tego zobaczysz `RuntimeError: coroutine ... was never awaited` albo dziwne błędy zależne od kolejności testów.

```python
# tests/conftest.py
from __future__ import annotations

from collections.abc import AsyncIterator

import pytest
import pytest_asyncio
from sqlalchemy.ext.asyncio import (
    AsyncEngine,
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)
from sqlalchemy.pool import StaticPool

from app.models import Base


@pytest_asyncio.fixture
async def engine() -> AsyncIterator[AsyncEngine]:
    """SQLite in-memory współdzielony przez cały test (StaticPool!)."""
    engine = create_async_engine(
        "sqlite+aiosqlite:///:memory:",
        poolclass=StaticPool,                        # jedna baza, nie 20 pustych
        connect_args={"check_same_thread": False},
    )
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    await engine.dispose()


@pytest_asyncio.fixture
async def session(engine: AsyncEngine) -> AsyncIterator[AsyncSession]:
    factory = async_sessionmaker(engine, expire_on_commit=False)
    async with factory() as session:
        yield session


@pytest_asyncio.fixture
async def session_factory(engine: AsyncEngine) -> async_sessionmaker[AsyncSession]:
    """Fabryka dla testów równoległości — sesja per zadanie."""
    return async_sessionmaker(engine, expire_on_commit=False)
```

Przykładowy test:

```python
# tests/test_authors.py
import pytest
from sqlalchemy import select

from app.models import Author, Book


@pytest.mark.asyncio
async def test_selectinload_avoids_n_plus_one(session, query_counter) -> None:
    session.add(
        Author(
            name="Ursula K. Le Guin",
            books=[Book(title="A"), Book(title="B"), Book(title="C")],
        )
    )
    await session.commit()
    session.expunge_all()

    query_counter.reset()
    stmt = select(Author).options(selectinload(Author.books))
    authors = (await session.scalars(stmt)).all()

    assert authors[0].books        # dostęp bez MissingGreenlet
    assert query_counter.count <= 2   # 1 na autorów + 1 na książki
```

> 🧠 **Dlaczego `StaticPool` jest tu kluczowe.** W SQLite baza `:memory:` istnieje **wyłącznie w obrębie jednego połączenia**. Bez `StaticPool` każde nowe połączenie dostaje **świeżą, pustą bazę** — i testy padają z `no such table: authors`, mimo że `create_all` się wykonało. `StaticPool` trzyma jedno połączenie i oddaje je zawsze to samo.

> ⚠️ **Pułapka — SQLite w testach to nie PostgreSQL.** Testy async na SQLite nie sprawdzą: zachowania `RETURNING` (starsze wersje), poziomów izolacji, `SELECT FOR UPDATE`, `ON CONFLICT`, zachowania przy realnej współbieżności. Ścieżki krytyczne warto uruchamiać na PostgreSQL (np. przez `testcontainers`). To zapowiedź modułu 18.

> 🆕 **SQLAlchemy 2.1** — poprawiono mapowanie dataclass: domyślne wartości nie lądują już w `__dict__` obiektu. Jeśli w testach porównujesz `obj.__dict__` albo używasz `dataclasses.asdict()`, zobaczysz różnicę względem 2.0.

> 🧪 **Ćwiczenie.** Napisz test, który dowodzi, że `await author.awaitable_attrs.books` w pętli po 10 autorach wykonuje 11 zapytań, a `selectinload` — tylko 2. Użyj `QueryCounter` z sekcji 7.3.

---

## 16. Kompletny przykład — CLI pobierające dane równolegle

Zbierzmy wszystko w jeden, uruchamialny program. Znajdziesz tu: tworzenie silnika, `run_sync` dla DDL, sesję per zadanie, `TaskGroup`, `Semaphore`, pomiar czasu i wariant na SQLite oraz PostgreSQL.

```python
# examples/15_async_cli.py
"""
CLI pobierające dane 20 razy — porównanie podejść.

Uruchomienie:
    pip install "SQLAlchemy>=2.0" aiosqlite
    python examples/15_async_cli.py

Wariant PostgreSQL:
    pip install asyncpg
    DATABASE_URL="postgresql+asyncpg://app:secret@localhost/library" \
        python examples/15_async_cli.py
"""
from __future__ import annotations

import asyncio
import os
import time
from datetime import date

from sqlalchemy import ForeignKey, String, func, select
from sqlalchemy.ext.asyncio import (
    AsyncAttrs,
    AsyncEngine,
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship, selectinload

DATABASE_URL = os.environ.get("DATABASE_URL", "sqlite+aiosqlite:///library_async.db")
N_TASKS = 20


# ---------------------------------------------------------------------------
# 1. Modele
# ---------------------------------------------------------------------------
class Base(AsyncAttrs, DeclarativeBase):
    pass


class Author(Base):
    __tablename__ = "authors"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(120))

    books: Mapped[list[Book]] = relationship(back_populates="author", lazy="raise")

    def __repr__(self) -> str:
        return f"Author(id={self.id!r}, name={self.name!r})"


class Book(Base):
    __tablename__ = "books"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    published: Mapped[date | None]
    author_id: Mapped[int] = mapped_column(ForeignKey("authors.id"))

    author: Mapped[Author] = relationship(back_populates="books", lazy="raise")

    def __repr__(self) -> str:
        return f"Book(id={self.id!r}, title={self.title!r})"


# ---------------------------------------------------------------------------
# 2. Silnik i fabryka sesji
# ---------------------------------------------------------------------------
def build_engine() -> AsyncEngine:
    return create_async_engine(
        DATABASE_URL,
        echo=False,
        pool_pre_ping=True,
        pool_size=10,
        max_overflow=10,
    )


# ---------------------------------------------------------------------------
# 3. DDL i dane startowe
# ---------------------------------------------------------------------------
async def prepare(engine: AsyncEngine) -> None:
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
        await conn.run_sync(Base.metadata.create_all)


async def seed(factory: async_sessionmaker[AsyncSession]) -> None:
    authors = [
        Author(
            name=f"Autor {i:02d}",
            books=[Book(title=f"Książka {i}-{j}", published=date(2000 + j)) for j in range(3)],
        )
        for i in range(1, 11)
    ]
    async with factory() as session:
        async with session.begin():
            session.add_all(authors)


# ---------------------------------------------------------------------------
# 4. Trzy podejścia do pobrania danych
# ---------------------------------------------------------------------------
async def approach_sequential(factory: async_sessionmaker[AsyncSession]) -> float:
    """Jedna sesja, 20 zapytań jedno po drugim."""
    start = time.perf_counter()
    async with factory() as session:
        for _ in range(N_TASKS):
            await session.scalar(select(func.count()).select_from(Book))
    return time.perf_counter() - start


async def approach_one_session_parallel(factory: async_sessionmaker[AsyncSession]) -> float:
    """ANTYWZORZEC: jedna sesja współdzielona przez równoległe zadania."""
    start = time.perf_counter()
    async with factory() as session:
        async def one() -> int:
            return await session.scalar(select(func.count()).select_from(Book)) or 0

        try:
            await asyncio.gather(*(one() for _ in range(N_TASKS)))
        except Exception as exc:  # noqa: BLE001 — pokazujemy intencję
            print(f"  [!] błąd przy współdzielonej sesji: {type(exc).__name__}: {exc}")
    return time.perf_counter() - start


async def approach_session_per_task(
    factory: async_sessionmaker[AsyncSession],
    *,
    concurrency: int = 10,
) -> float:
    """WZORZEC: sesja per zadanie + ograniczenie równoległości."""
    start = time.perf_counter()
    semaphore = asyncio.Semaphore(concurrency)

    async def one() -> int:
        async with semaphore:
            async with factory() as session:
                return await session.scalar(select(func.count()).select_from(Book)) or 0

    async with asyncio.TaskGroup() as tg:
        for _ in range(N_TASKS):
            tg.create_task(one())

    return time.perf_counter() - start


# ---------------------------------------------------------------------------
# 5. Odczyt z relacjami — selectinload kontra awaitable_attrs
# ---------------------------------------------------------------------------
async def read_with_selectinload(factory: async_sessionmaker[AsyncSession]) -> None:
    async with factory() as session:
        stmt = select(Author).options(selectinload(Author.books)).order_by(Author.id)
        authors = (await session.scalars(stmt)).all()
        for a in authors[:3]:
            print(f"  selectinload: {a.name} -> {[b.title for b in a.books]}")


async def read_with_awaitable_attrs(factory: async_sessionmaker[AsyncSession]) -> None:
    async with factory() as session:
        authors = (await session.scalars(select(Author).order_by(Author.id).limit(3))).all()
        for a in authors:
            books = await a.awaitable_attrs.books      # IO jest jawne
            print(f"  awaitable_attrs: {a.name} -> {[b.title for b in books]}")


# ---------------------------------------------------------------------------
# 6. Program główny
# ---------------------------------------------------------------------------
async def main() -> None:
    engine = build_engine()
    factory = async_sessionmaker(engine, expire_on_commit=False)

    print(f"Baza: {engine.url}\n")
    await prepare(engine)
    await seed(factory)

    print(f"--- {N_TASKS} zapytań, różne podejścia ---")
    t_seq = await approach_sequential(factory)
    print(f"  sekwencyjnie (1 sesja):        {t_seq:.3f} s")

    t_bad = await approach_one_session_parallel(factory)
    print(f"  równolegle (1 sesja, błąd):    {t_bad:.3f} s")

    t_good = await approach_session_per_task(factory, concurrency=10)
    print(f"  równolegle (sesja per zadanie):{t_good:.3f} s")
    print(f"  -> przyspieszenie: {t_seq / t_good:.1f}x\n")

    print("--- odczyt relacji ---")
    await read_with_selectinload(factory)
    await read_with_awaitable_attrs(factory)

    await engine.dispose()


if __name__ == "__main__":
    asyncio.run(main())
```

Przykładowy wynik na SQLite (`aiosqlite`):

```text
Baza: sqlite+aiosqlite:///library_async.db

--- 20 zapytań, różne podejścia ---
  sekwencyjnie (1 sesja):        0.041 s
  równolegle (1 sesja, błąd):    0.038 s
  [!] błąd przy współdzielonej sesji: InvalidRequestError: This session is provisioning a new connection; concurrent operations are not permitted
  równolegle (sesja per zadanie):0.019 s
  -> przyspieszenie: 2.2x

--- odczyt relacji ---
  selectinload: Autor 01 -> ['Książka 1-0', 'Książka 1-1', 'Książka 1-2']
  awaitable_attrs: Autor 01 -> ['Książka 1-0', 'Książka 1-1', 'Książka 1-2']
```

Na PostgreSQL z `asyncpg` i realnym opóźnieniem sieciowym przyspieszenie jest znacznie większe — tam async pokazuje pełnię możliwości.

> 🔬 **Pod maska — SQL z `selectinload`.** W powyższym programie zobaczysz (z `echo=True`):

```sql
SELECT authors.id, authors.name FROM authors ORDER BY authors.id
SELECT books.author_id AS books_author_id, books.id AS books_id,
       books.title AS books_title, books.published AS books_published
FROM books WHERE books.author_id IN (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
```

Jedno zapytanie na autorów, jedno na książki — niezależnie od liczby autorów.

> 🧪 **Ćwiczenie.** Uruchom program z `DATABASE_URL` wskazującym PostgreSQL. Zmień `concurrency` na `1`, `5`, `20` i `50`. Znajdź punkt, w którym zwiększanie równoległości przestaje pomagać. Wyjaśnij wynik w kontekście `pool_size + max_overflow` (u nas: 20).

---

## Podsumowanie

1. **Asynchroniczność to nie wielowątkowość.** Jeden wątek, jedna pętla zdarzeń, wiele zadań przełączanych w miejscach `await`. Analogia kelnera obsługującego 20 stolików jest tu najbliższa prawdy.
2. **Blokujące IO unieważnia cały mechanizm.** `time.sleep()`, `requests`, `psycopg2`, `sqlite3` — wszystko to blokuje pętlę i wszystkie zadania. Dlatego sterownik bazy **musi** być asynchroniczny: `asyncpg`, `psycopg` v3, `aiosqlite`, `aiomysql`.
3. **`create_async_engine()` bierze ten sam zestaw parametrów co `create_engine()`**, ale wymaga dialektu async w URL-u. Silnik tworzymy raz i zamykamy jawnie przez `await engine.dispose()` — w async nie ma destruktora, który by to zrobił.
4. **`AsyncSession` to ta sama sesja, opakowana.** Metody dotykające bazy wymagają `await`; metody czysto pamięciowe (`add`, `expunge`) nie. `expire_on_commit=False` jest w async praktycznie obowiązkiem.
5. **DDL wymaga `run_sync()`.** `create_all`, `drop_all`, `reflect`, `inspect` to funkcje synchroniczne — uruchamiamy je w kontekście greenleta przez `await conn.run_sync(...)`.
6. **`greenlet` to most.** Kod synchroniczny SQLAlchemy działa wewnątrz zielonego wątku, a w miejscu dotknięcia sterownika następuje kontrolowane przełączenie do pętli. Poza tym kontekstem `await_only()` nie działa — stąd `MissingGreenlet`.
7. **`MissingGreenlet` = próba ukrytego IO.** Trzy źródła: leniwe ładowanie relacji, odczyt po `commit()` z `expire_on_commit=True`, odczyt odroczonej kolumny. Ratunek: `selectinload`, `expire_on_commit=False`, `AsyncAttrs.awaitable_attrs`, `lazy="raise"`.
8. **`lazy="raise"` to najlepszy komunikat błędu, jaki możesz sobie sprawić.** Zamienia tajemniczy `MissingGreenlet` na jednoznaczne „ta relacja nie jest załadowana”.
9. **Strumieniowanie zamiast `all()`.** `await session.stream_scalars(stmt)` + `async for` + `yield_per`/`partitions` pozwala przetwarzać miliony wierszy bez zjadania pamięci.
10. **Jedna sesja na zadanie — zawsze.** `AsyncSession` nie jest bezpieczna dla współbieżnych zadań. Współdzielona sesja to albo `InvalidRequestError`, albo ciche, błędne wyniki.
11. **Równoległość trzeba stworzyć jawnie.** `await` w pętli `for` nic nie przyspiesza. Dopiero `asyncio.TaskGroup` (lub `gather`) plus `Semaphore` i świadomy dobór `pool_size` dają realny zysk.
12. **Async nie jest szybszy dla pojedynczego zapytania.** Wybieraj go tam, gdzie jest realna współbieżność: serwer HTTP, worker kolejki, agregator wielu wywołań.

---

## Ćwiczenia

### Ćwiczenie 1 — Przepisanie skryptu z modułu 11 na async (poziom: średni)

Poniższy kod synchroniczny z modułu 11 czyta 100 autorów i wypisuje ich książki. Zawiera klasyczny N+1. **Przepisz go na async** tak, aby (a) działał bez `MissingGreenlet`, (b) wykonywał dokładnie 2 zapytania, (c) zamykał silnik.

```python
# punkt wyjścia — wersja synchroniczna z N+1
from sqlalchemy import select
from sqlalchemy.orm import Session

with Session(engine) as session:
    authors = session.scalars(select(Author)).all()      # 1 zapytanie
    for a in authors:
        print(a.name, [b.title for b in a.books])        # 100 zapytań!
```

Wymagania dodatkowe:
- użyj `async_sessionmaker` z `expire_on_commit=False`,
- dodaj licznik zapytań podpięty do `engine.sync_engine` i wypisz jego wynik,
- obsłuż brak autorów bez błędu.

### Ćwiczenie 2 — Timeout i anulowanie (poziom: trudny)

Napisz funkcję `fetch_or_fail(session_factory, stmt, seconds)`, która:
1. wykonuje `stmt` w własnej sesji,
2. przerywa po `seconds` sekundach przez `asyncio.timeout()`,
3. po przekroczeniu czasu **wycofuje transakcję**, zamyka sesję i rzuca własny wyjątek `FetchTimeout`,
4. w `finally` **zawsze** zamyka sesję przez `await session.close()`.

Napisz test (lub skrypt), który dowodzi, że po timeout sesja jest w stanie umożliwiającym ponowne użycie fabryki, a liczba połączeń w puli wróciła do poziomu wyjściowego (`engine.pool.status()`).

### Ćwiczenie 3 — Równoległy agregator z limitem (poziom: ekspert)

Zbuduj funkcję `aggregate(session_factory, author_ids, *, concurrency)`, która:
1. dla każdego `author_id` pobiera liczbę książek (`SELECT count(*) ... WHERE author_id = ?`),
2. wykonuje zapytania równolegle, ale nigdy więcej niż `concurrency` naraz,
3. używa **sesji per zadanie**,
4. zwraca `dict[int, int]` (mapowanie autor → liczba książek),
5. działa w `TaskGroup`, więc błąd w jednym zadaniu anuluje pozostałe.

Następnie uruchom ją dla 200 autorów z `concurrency` równym 1, 5, 10, 25, 100 i zestaw wyniki w tabeli: `concurrency`, czas, liczba błędów. Wyjaśnij, dlaczego od pewnego momentu czas przestaje maleć.

### Rozwiązania

#### Rozwiązanie 1

```python
# solutions/15_ex1.py
from __future__ import annotations

import asyncio

from sqlalchemy import event, select
from sqlalchemy.ext.asyncio import (
    AsyncEngine,
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)
from sqlalchemy.orm import selectinload

from models import Author, Base


class QueryCounter:
    def __init__(self) -> None:
        self.statements: list[str] = []

    def attach(self, engine: AsyncEngine) -> None:
        @event.listens_for(engine.sync_engine, "before_cursor_execute")
        def _count(conn, cursor, statement, parameters, context, executemany) -> None:
            self.statements.append(statement)


async def main() -> None:
    engine = create_async_engine("sqlite+aiosqlite:///library.db")
    factory = async_sessionmaker(engine, expire_on_commit=False)
    counter = QueryCounter()
    counter.attach(engine)

    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    async with factory() as session:
        stmt = (
            select(Author)
            .options(selectinload(Author.books))     # <- klucz do 2 zapytań
            .order_by(Author.id)
        )
        authors = (await session.scalars(stmt)).all()

        if not authors:
            print("Brak autorów w bazie.")
        for a in authors:
            print(a.name, [b.title for b in a.books])

    print(f"Zapytania: {len(counter.statements)}")
    for s in counter.statements:
        print(" -", " ".join(s.split())[:90])

    await engine.dispose()


asyncio.run(main())
```

**Dlaczego tak:** `selectinload` wykonuje jedno dodatkowe zapytanie z `IN (...)`, niezależnie od liczby autorów. `expire_on_commit=False` nie jest tu potrzebne (brak `commit`), ale zostawiamy je spójnie z resztą projektu.

**Alternatywa:** `joinedload(Author.books)` + `.unique()` — jedno zapytanie zamiast dwóch, ale wiersze się mnożą. Przy 100 autorach × 3 książki to 300 wierszy w jednym zapytaniu. Kompromis: mniej rund do bazy, więcej danych w sieci.

**Minipułapka:** jeśli dodasz `print(a)` zamiast `print(a.name)` w pętli, a `__repr__` modelu odwołuje się do `self.books`, dostaniesz `MissingGreenlet` **mimo** poprawnego `selectinload` — bo `__repr__` na obiekcie spoza wyniku (np. w logu błędu) nadal próbuje leniwego ładowania.

#### Rozwiązanie 2

```python
# solutions/15_ex2.py
from __future__ import annotations

import asyncio
from typing import Any

from sqlalchemy import Select
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker


class FetchTimeout(Exception):
    """Własny wyjątek — nie przeciekamy TimeoutError do warstwy wyżej."""


async def fetch_or_fail(
    session_factory: async_sessionmaker[AsyncSession],
    stmt: Select[Any],
    seconds: float = 2.0,
) -> list[Any]:
    session = session_factory()
    try:
        async with asyncio.timeout(seconds):
            result = await session.scalars(stmt)
            return list(result.all())
    except TimeoutError as exc:
        # Anulowanie mogło trafić sesję w środku operacji — wyczyść stan.
        await session.rollback()
        raise FetchTimeout(f"Zapytanie przekroczyło {seconds} s") from exc
    except asyncio.CancelledError:
        await session.rollback()
        raise
    finally:
        await session.close()      # ZAWSZE await — inaczej wyciek połączeń
```

**Dlaczego tak:** trzy rzeczy naraz: (1) `asyncio.timeout` ogranicza czas oczekiwania, (2) `await session.rollback()` czyści stan sesji po przerwanym IO, (3) `finally: await session.close()` gwarantuje zwrot połączenia do puli nawet przy `CancelledError`.

**Alternatywa:** `asyncio.wait_for(coro, timeout=...)` — działa też na Pythonie 3.10, ale jest mniej czytelne przy zagnieżdżonych operacjach i nie tworzy własnego kontekstu czasowego.

**Minipułapka:** `except TimeoutError` w Pythonie 3.11+ łapie również `asyncio.TimeoutError` (są aliasami). Jeśli wspierasz 3.10, musisz łapać `asyncio.TimeoutError`.

#### Rozwiązanie 3

```python
# solutions/15_ex3.py
from __future__ import annotations

import asyncio
import time
from collections.abc import Sequence

from sqlalchemy import func, select
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker

from models import Book


async def aggregate(
    session_factory: async_sessionmaker[AsyncSession],
    author_ids: Sequence[int],
    *,
    concurrency: int = 10,
) -> dict[int, int]:
    """Zwraca mapowanie author_id -> liczba książek, z ograniczoną równoległością."""
    semaphore = asyncio.Semaphore(concurrency)
    results: dict[int, int] = {}

    async def one(author_id: int) -> tuple[int, int]:
        async with semaphore:
            async with session_factory() as session:      # sesja per zadanie
                stmt = select(func.count()).select_from(Book).where(
                    Book.author_id == author_id
                )
                count = await session.scalar(stmt)
                return author_id, count or 0

    async with asyncio.TaskGroup() as tg:
        tasks = [tg.create_task(one(aid)) for aid in author_ids]

    for task in tasks:
        aid, count = task.result()
        results[aid] = count

    return results


async def benchmark(session_factory, author_ids, levels=(1, 5, 10, 25, 100)) -> None:
    print(f"{'concurrency':>12} | {'czas [s]':>9} | {'błędy':>6}")
    print("-" * 34)
    for level in levels:
        start = time.perf_counter()
        errors = 0
        try:
            await aggregate(session_factory, author_ids, concurrency=level)
        except Exception as exc:  # noqa: BLE001
            errors += 1
            print(f"  błąd przy concurrency={level}: {type(exc).__name__}")
        elapsed = time.perf_counter() - start
        print(f"{level:>12} | {elapsed:>9.3f} | {errors:>6}")
```

**Dlaczego tak:** `Semaphore` ogranicza **zadania**, pula ogranicza **połączenia**. Ograniczając zadania, ograniczamy też kolejkę oczekujących korutyn i ryzyko `pool_timeout`.

**Alternatywa:** bez `Semaphore`, ale z `pool_size` dobranym do spodziewanej równoległości. Prostsze, ale mniej przewidywalne — pula nie chroni przed tysiącem zadań czekających w kolejce.

**Minipułapka:** przy `concurrency=100` i `pool_size=10, max_overflow=10` część zadań przekroczy `pool_timeout` (domyślnie 30 s) i zgłosi `TimeoutError: QueuePool limit of size 10 overflow 10 reached, connection timed out`. W tabeli zobaczysz to jako rosnącą liczbę błędów. Wniosek: **nigdy nie ustawiaj `concurrency` wyżej niż `pool_size + max_overflow`** bez dobrego powodu.

---

## Najczęstsze błędy i jak je czytać

| Komunikat błędu | Przyczyna | Naprawa |
|---|---|---|
| `MissingGreenlet: greenlet_spawn has not been called; can't call await_only() here.` | Próba ukrytego IO: leniwe ładowanie relacji, odczyt po `commit()` z `expire_on_commit=True`, odczyt odroczonej kolumny | Dodaj `selectinload`/`joinedload`; ustaw `expire_on_commit=False`; użyj `await obj.awaitable_attrs.rel`; ustaw `lazy="raise"` dla czytelniejszego błędu |
| `InvalidRequestError: This session is provisioning a new connection; concurrent operations are not permitted` | Jedna `AsyncSession` współdzielona przez równoległe zadania | Sesja per zadanie: `async with factory() as session:` wewnątrz każdego zadania |
| `RuntimeError: Task got Future attached to a different loop` | `AsyncEngine` współdzielony między pętlami zdarzeń | `await engine.dispose()` przed zmianą pętli; albo `poolclass=NullPool` |
| `RuntimeError: asyncio.run() cannot be called from a running event loop` | Zagnieżdżone `asyncio.run()` | Użyj `await` na korutynie; w testach `pytest-asyncio` |
| `RuntimeWarning: coroutine 'AsyncSession.commit' was never awaited` | Brak `await` przy metodzie sesji | Dodaj `await` — to cichy błąd: dane nie zostały zapisane |
| `TypeError: object NoneType can't be used in 'await' expression` | `await` na metodzie synchronicznej, np. `await session.add(obj)` | Usuń `await` z `add`, `add_all`, `expunge`, `expire_all` |
| `TypeError: object method can't be used in 'await' expression` | `await` na metodzie buforującej `AsyncResult`, np. `await result.all()` | `AsyncResult.all()` jest synchroniczna — wołaj bez `await` |
| `InvalidRequestError: The asyncio extension requires an async driver to be used` | Synchroniczny sterownik w `create_async_engine` | Zmień URL: `postgresql+asyncpg://`, `sqlite+aiosqlite://` |
| `PendingRollbackError` / `InvalidRequestError: This session is in 'prepared' state` | Poprzednia transakcja zakończyła się błędem bez `rollback` | `await session.rollback()` w `except`/`finally` |
| `DetachedInstanceError` | Obiekt poza sesją, odczyt relacji | Eager loading **przed** zamknięciem sesji; albo `AsyncAttrs` w zasięgu sesji |
| `sqlalchemy.exc.InvalidRequestError: lazy load operation of attribute 'books' cannot proceed` | `lazy="raise"` zadziałał | Dodaj `selectinload`/`joinedload` — to **poprawny** sygnał |
| `TimeoutError: QueuePool limit of size 10 overflow 20 reached` | Wyczerpana pula — za dużo równoległych zadań albo wyciek połączeń | Ogranicz `concurrency`; sprawdź, czy wszędzie jest `await session.close()` |
| `RuntimeError: Event loop is closed` (ostrzeżenie na wyjściu) | Brak `await engine.dispose()` | Dodaj `await engine.dispose()` przy zamykaniu aplikacji |
| `no such table: authors` w testach na `:memory:` | Każde połączenie dostaje własną, pustą bazę | `poolclass=StaticPool` w `create_async_engine` dla testów |
| `RuntimeError: coroutine ... was never awaited` w fixture | `@pytest.fixture` zamiast `@pytest_asyncio.fixture` dla `async def` | Zmień dekorator; rozważ `asyncio_mode = "auto"` |
| `InterfaceError: another operation is in progress` | Dwie operacje na tym samym połączeniu jednocześnie | Sesja per zadanie; nie używaj `gather` z jedną sesją |

---

## Słowniczek modułu

| Termin (EN) | Polski odpowiednik | Wyjaśnienie |
|---|---|---|
| `async def` | funkcja asynchroniczna | Deklaracja korutyny — funkcji, którą można zawiesić w miejscu `await` |
| `await` | — | Znacznik „tu mogę oddać sterowanie pętli i poczekać na wynik” |
| coroutine | korutyna | Obiekt zwrócony przez `async def`; nie wykonuje się, dopóki nie zostanie zaplanowany |
| event loop | pętla zdarzeń | Silnik przełączający zadania; w asyncio działa w jednym wątku |
| task | zadanie | Korutyna zaplanowana do wykonania w pętli |
| `asyncio.run()` | — | Tworzy pętlę, wykonuje korutynę do końca, zamyka pętlę; nie może być zagnieżdżone |
| blocking I/O | blokujące wejście/wyjście | Operacja zajmująca cały wątek na czas oczekiwania; wróg async |
| `asyncio.gather()` | — | Uruchamia korutyny równolegle; przy błędzie pozostałe **nie** są anulowane |
| `asyncio.TaskGroup` | grupa zadań | Kontekst menedżer (3.11+); przy błędzie anuluje pozostałe zadania |
| `asyncio.Semaphore` | semafor | Ogranicznik liczby jednocześnie wykonujących się zadań |
| `asyncio.timeout()` | limit czasu | Kontekst menedżer (3.11+) przerywający blok po zadanym czasie |
| cancellation | anulowanie | Wstrzyknięcie `CancelledError` do zadania w dowolnym punkcie `await` |
| driver | sterownik | Biblioteka rozmawiająca z bazą (DBAPI); musi być async: `asyncpg`, `aiosqlite` |
| `create_async_engine()` | — | Tworzy `AsyncEngine`; wymaga dialektu async w URL-u |
| `AsyncEngine` | silnik asynchroniczny | Odpowiednik `Engine`; wymaga `await engine.dispose()` |
| `AsyncConnection` | połączenie asynchroniczne | Odpowiednik `Connection`; `await conn.execute(...)` |
| `AsyncSession` | sesja asynchroniczna | Odpowiednik `Session`; metody IO wymagają `await` |
| `async_sessionmaker` | fabryka sesji async | Odpowiednik `sessionmaker`; tworzona raz na aplikację |
| `expire_on_commit=False` | — | Nie unieważniaj obiektów po `commit()`; w async praktycznie obowiązek |
| `run_sync()` | — | Uruchamia funkcję synchroniczną w kontekście greenleta (dla DDL, refleksji, inspektora) |
| greenlet | zielony wątek | Lekki, współpracujący wątek; most między kodem sync SQLAlchemy a sterownikiem async |
| `greenlet_spawn` | — | Wejście w kontekst greenleta; jego brak powoduje `MissingGreenlet` |
| `MissingGreenlet` | — | Błąd: próba ukrytego IO poza kontekstem greenleta |
| `AsyncAttrs` | — | Mixin (2.0.13+) dodający `awaitable_attrs` |
| `awaitable_attrs` | atrybuty awaitowalne | `await obj.awaitable_attrs.relacja` — jawne, `await`-owalne leniwe ładowanie |
| `lazy="raise"` | — | Strategia ładowania zgłaszająca błąd przy próbie leniwego ładowania |
| `lazy="write_only"` | kolekcja tylko do zapisu | Kolekcja, która nigdy nie wykonuje niejawnego IO |
| `selectinload()` | — | Eager loading przez dodatkowe `SELECT ... IN (...)`; zalecany w async |
| `joinedload()` | — | Eager loading przez `LEFT OUTER JOIN`; przy kolekcjach wymaga `.unique()` |
| `stream()` | strumieniowanie | `await session.stream(stmt)` → `AsyncResult` z kursorem po stronie serwera |
| `stream_scalars()` | — | `await session.stream_scalars(stmt)` → `AsyncScalarResult` (obiekty ORM) |
| `AsyncResult` | — | Asynchroniczny odpowiednik `Result`; obsługuje `async for` i `async with` |
| `yield_per` | — | Pobieranie i przetwarzanie wyników partiami o zadanym rozmiarze |
| `partitions(n)` | partycje | Iteracja po wyniku w partiach po `n` elementów |
| `NullPool` | pula pusta | Pula nieprzechowująca połączeń; dla PgBouncera i wielu pętli |
| `StaticPool` | pula statyczna | Jedno połączenie wielokrotnego użytku; dla SQLite `:memory:` |
| `AsyncAdaptedQueuePool` | — | Domyślna pula dla async — `QueuePool` z adaptacją połączeń |
| `sync_engine` | — | Atrybut `AsyncEngine` dający dostęp do synchronicznego silnika (dla eventów) |
| `sync_session` | — | Atrybut `AsyncSession` dający dostęp do synchronicznej sesji |
| concurrency | współbieżność | Wiele zadań postępujących naprzemiennie; nie to samo co równoległość |
| parallelism | równoległość | Wykonywanie w tym samym czasie (np. wiele rdzeni); asyncio jej nie daje |
| backpressure | presja zwrotna | Ograniczanie napływu pracy (tu: `Semaphore`, `pool_size`) |

---

## Dalsze czytanie

**SQLAlchemy 2.0 — asyncio (dokumentacja podstawowa):**
- [Asynchronous I/O (asyncio)](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html) — cała sekcja, obowiązkowa lektura
- [Synopsis — ORM](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html#synopsis-orm) — kompletny przykład z `AsyncAttrs`
- [Preventing Implicit IO when Using AsyncSession](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html#preventing-implicit-io-when-using-asyncsession) — sekcja, którą warto przeczytać dwa razy
- [Using AsyncSession with Concurrent Tasks](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html#using-asyncsession-with-concurrent-tasks)
- [Using multiple asyncio event loops](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html#using-multiple-asyncio-event-loops)
- [Running Synchronous Methods and Functions under asyncio](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html#running-synchronous-methods-and-functions-under-asyncio) — o `run_sync()`
- [Using events with the asyncio extension](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html#using-events-with-the-asyncio-extension)
- [Using the Inspector to inspect schema objects](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html#using-the-inspector-to-inspect-schema-objects)
- [Using asyncio scoped session](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html#using-asyncio-scoped-session)

**SQLAlchemy 2.0 — tematy powiązane:**
- [Relationship Loading Techniques](https://docs.sqlalchemy.org/en/20/orm/queryguide/relationships.html) — `selectinload`, `joinedload`, `raiseload`
- [Write Only Relationships](https://docs.sqlalchemy.org/en/20/orm/queryguide/relationships.html#write-only-relationships)
- [Connection Pooling](https://docs.sqlalchemy.org/en/20/core/pooling.html) — `NullPool`, `StaticPool`, `pool_size`
- [Is the Session thread-safe? Is AsyncSession safe to share in concurrent tasks?](https://docs.sqlalchemy.org/en/20/orm/session_basics.html#is-the-session-thread-safe-is-asyncsession-safe-to-share-in-concurrent-tasks)
- [What's New in SQLAlchemy 2.0](https://docs.sqlalchemy.org/en/20/changelog/whatsnew_20.html)

**SQLAlchemy 2.1:**
- [What's New in SQLAlchemy 2.1](https://docs.sqlalchemy.org/en/21/changelog/whatsnew_21.html)
- [Asynchronous I/O (asyncio) — wersja 2.1](https://docs.sqlalchemy.org/en/21/orm/extensions/asyncio.html)

**Sterowniki:**
- [asyncpg](https://magicstack.github.io/asyncpg/current/)
- [aiosqlite](https://aiosqlite.omnilib.dev/en/stable/)
- [psycopg 3](https://www.psycopg.org/psycopg3/docs/)

**Python asyncio:**
- [Coroutines and Tasks](https://docs.python.org/3/library/asyncio-task.html) — `TaskGroup`, `timeout`, `gather`
- [Synchronization Primitives](https://docs.python.org/3/library/asyncio-sync.html) — `Semaphore`
- [Developing with asyncio](https://docs.python.org/3/library/asyncio-dev.html) — debugowanie, typowe pułapki

**Narzędzia:**
- [Alembic — Using Asyncio with Alembic](https://alembic.sqlalchemy.org/en/latest/cookbook.html#using-asyncio-with-alembic)
- [pytest-asyncio](https://pytest-asyncio.readthedocs.io/en/latest/)

---

## Co dalej

Masz już komplet narzędzi do pracy z bazą w każdym trybie: wiesz, jak działa Core, jak działa ORM, jak działa async, jak ładować relacje, jak transakcjonować i jak mierzyć wydajność. Zostało jedno pytanie, na które jak dotąd odpowiadaliśmy ręcznie: **co się dzieje, gdy schemat bazy musi się zmienić** — gdy dochodzi kolumna, zmienia się typ, a dane produkcyjne trzeba zachować.

W module [`16_alembic_migracje.md`](16_alembic_migracje.md) zajmiemy się wersjonowaniem schematu z Alembic: `autogenerate`, `env.py` (również w wariancie async z sekcji 15), migracje danych, `render_as_batch` dla SQLite oraz wzorzec `expand/contract` pozwalający wdrażać zmiany bez przerwy w działaniu aplikacji.

<!-- koniec modułu 15 -->