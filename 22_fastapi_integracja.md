# Moduł 22 — SQLAlchemy w aplikacji webowej: FastAPI od modelu do endpointu

Ten moduł składa w całość wszystko, czego nauczyłeś się wcześniej: modele, sesję, transakcje, async, repozytoria i jednostkę pracy. Zbudujemy z tego jedną, działającą aplikację REST — bibliotekę z książkami, autorami i wypożyczeniami. Nauczysz się, gdzie w aplikacji webowej tworzyć silnik bazy, jak wstrzykiwać sesję do endpointu, jak zamienić wyjątki bazy danych w sensowne odpowiedzi HTTP, jak nie zwracać encji „na żywca” do klienta i jak zabezpieczyć operację wypożyczenia przed wyścigiem dwóch użytkowników klikających w tym samym momencie. Efektem końcowym jest projekt w siedmiu plikach, który uruchomisz jedną komendą, razem z migracjami i testami.

**Poziom:** 🔴 architektoniczny
**Czas:** ~210 minut
**Wymagania wstępne:**
[Moduł 07 — Modele deklaratywne](07_modele_deklaratywne.md) ·
[Moduł 11 — Ładowanie relacji i N+1](11_ladowanie_i_n_plus_1.md) ·
[Moduł 14 — Transakcje i współbieżność](14_transakcje_i_wspolbieznosc.md) ·
[Moduł 15 — Tryb asynchroniczny](15_asynchronicznosc.md) ·
[Moduł 16 — Migracje z Alembic](16_alembic_migracje.md) ·
[Moduł 20 — Wzorzec Repository](20_repository.md) ·
[Moduł 21 — Unit of Work](21_unit_of_work.md)

**Plik dotyczy:** integracji SQLAlchemy 2.0+ z FastAPI — architektury warstwowej aplikacji webowej, cyklu życia silnika i sesji, schematów Pydantic v2, obsługi błędów, transakcji w endpointach, wariantu asynchronicznego, migracji we wdrożeniu i wydajności API.

---

## Spis treści

- [22.1 Dlaczego aplikacja, a nie skrypt](#221-dlaczego-aplikacja-a-nie-skrypt)
- [22.2 Architektura projektu — kto za co odpowiada](#222-architektura-projektu--kto-za-co-odpowiada)
- [22.3 Konfiguracja: jedna prawda o środowisku](#223-konfiguracja-jedna-prawda-o-srodowisku)
- [22.4 Cykl życia aplikacji: silnik tworzony raz](#224-cykl-zycia-aplikacji-silnik-tworzony-raz)
- [22.5 Zarządzanie sesją w żądaniu HTTP](#225-zarzadzanie-sesja-w-zadaniu-http)
- [22.6 Modele, schematy Pydantic i problem „oderwanej encji”](#226-modele-schematy-pydantic-i-problem-oderwanej-encji)
- [22.7 Repozytorium: zapytania w jednym miejscu](#227-repozytorium-zapytania-w-jednym-miejscu)
- [22.8 Serwis: granica transakcji i reguły biznesowe](#228-serwis-granica-transakcji-i-reguly-biznesowe)
- [22.9 Router i siedem endpointów](#229-router-i-siedem-endpointow)
- [22.10 Obsługa błędów: od wyjątku domenowego do JSON-a](#2210-obsluga-bledow-od-wyjatku-domenowego-do-json-a)
- [22.11 Operacja złożona: wypożyczenie z blokadą wiersza](#2211-operacja-zlozona-wypozyczenie-z-blokada-wiersza)
- [22.12 Async w FastAPI: `async def`, `def`, wątki i zadania w tle](#2212-async-w-fastapi-async-def-def-watki-i-zadania-w-tle)
- [22.13 Migracje i dane słownikowe we wdrożeniu](#2213-migracje-i-dane-slownikowe-we-wdrozeniu)
- [22.14 Wydajność API: N+1, limity, cache, monitoring](#2214-wydajnosc-api-n1-limity-cache-monitoring)
- [22.15 Bezpieczeństwo i higiena](#2215-bezpieczenstwo-i-higiena)
- [22.16 Alternatywy: Flask, Django, CLI](#2216-alternatywy-flask-django-cli)
- [22.17 Kompletny projekt: jeden plik, który działa](#2217-kompletny-projekt-jeden-plik-ktory-dziala)
- [22.18 Testy integracyjne API](#2218-testy-integracyjne-api)
- [22.19 Jak uruchomić cały projekt](#2219-jak-uruchomic-caly-projekt)
- [Podsumowanie](#podsumowanie)
- [Ćwiczenia](#cwiczenia)
- [Najczęstsze błędy i jak je czytać](#najczestsze-bledy-i-jak-je-czytac)
- [Słowniczek modułu](#slowniczek-modulu)
- [Dalsze czytanie](#dalsze-czytanie)
- [Co dalej](#co-dalej)

---

## 22.1 Dlaczego aplikacja, a nie skrypt

W modułach 07–21 pisaliśmy krótkie skrypty: utwórz silnik, otwórz sesję, dodaj książkę, zamknij. Taki skrypt żyje kilkaset milisekund i kończy się razem z procesem. Aplikacja webowa żyje godzinami, obsługuje setki żądań naraz i każde z nich potrzebuje własnej sesji, własnej transakcji i własnego połączenia z bazą — ale **nie własnego silnika**.

> 💡 Analogia — Wyobraź sobie restaurację. Silnik bazy danych to kuchnia z piecem, wentylacją i zapasem gazu: buduje się ją raz i działa przez cały dzień. Sesja to jedno zamówienie na jednym stoliku: kelner bierze kartkę, notuje, co potrzeba, zanosi do kuchni i wraca z gotowym daniem. Gdyby kelner budował nową kuchnię przy każdym zamówieniu, restauracja padłaby po kwadransie. A gdyby wszyscy kelnerzy notowali na **jednej** kartce, dania pomieszałyby się i nikt nie wiedziałby, co właściwie jest w zamówieniu.

Z tego obrazka wynikają trzy zasady, które będą nam towarzyszyć przez cały moduł:

1. **Silnik (`AsyncEngine`) — jeden na proces aplikacji.** Tworzymy go przy starcie, zamykamy przy zakończeniu.
2. **Sesja (`AsyncSession`) — jedna na żądanie HTTP.** Rozpoczyna się, gdy przychodzi request; kończy, gdy budujemy odpowiedź.
3. **Transakcja — jedna na przypadek użycia.** „Zapisz książkę” to jedna transakcja. „Wypożycz książkę i zmniejsz stan magazynu” to również jedna transakcja, choć dotyka dwóch tabel.

> 🧠 Dlaczego tak jest — FastAPI obsługuje żądania współbieżnie. Jeśli dwie korutyny podzielą jedną sesję, ich operacje trafią do jednej transakcji i jednej mapy tożsamości (identity map), wzajemnie się nadpisując. Objaw bywa paskudny: użytkownik A widzi w odpowiedzi dane użytkownika B. Nie da się tego naprawić „uważnym pisaniem kodu” — jedyne rozwiązanie to osobna sesja na żądanie. Objęliśmy to już w [module 08](08_sesja_cykl_zycia.md), teraz przekładamy na konkretny framework.

Zanim przejdziemy do kodu, ustalmy jeszcze, czym różni się praca z FastAPI od pracy z Flashem czy Django. FastAPI nie ma wbudowanego ORM-a ani globalnego kontekstu żądania w stylu `flask.g`. Zamiast tego dostajesz system **wstrzykiwania zależności** (dependency injection): deklarujesz, czego potrzebuje endpoint, a framework wywołuje Twoją funkcję z gotowymi obiektami. Sesja to dokładnie taka zależność.

---

## 22.2 Architektura projektu — kto za co odpowiada

Pierwsza decyzja nie dotyczy kodu, ale **układu plików**. Warstwy, które omówiliśmy w [module 19](19_warstwy_i_data_mapper.md), muszą mieć fizyczne odzwierciedlenie w katalogach — inaczej po pół roku nikt nie odróżni repozytorium od serwisu.

```text
library-api/
├── alembic.ini
├── alembic/
│   ├── env.py                  # konfiguracja Alembica (async!)
│   └── versions/               # pliki migracji
├── app/
│   ├── __init__.py
│   ├── main.py                 # create_app(), lifespan, rejestracja routerów
│   ├── cli.py                  # komendy administracyjne (seed danych słownikowych)
│   ├── api/
│   │   ├── __init__.py
│   │   ├── deps.py             # get_session(), get_uow() — zależności FastAPI
│   │   └── v1/
│   │       ├── __init__.py
│   │       ├── books.py        # router: książki
│   │       ├── authors.py      # router: autorzy
│   │       └── loans.py        # router: wypożyczenia
│   ├── core/
│   │   ├── config.py           # Settings (pydantic-settings)
│   │   ├── db.py               # build_engine(), build_session_factory()
│   │   └── errors.py           # wyjątki domenowe + handlery
│   ├── models/
│   │   ├── __init__.py         # Base + reeksport modeli
│   │   ├── base.py             # DeclarativeBase, mixiny, naming convention
│   │   ├── author.py
│   │   ├── book.py
│   │   └── loan.py
│   ├── repositories/
│   │   ├── books.py
│   │   ├── authors.py
│   │   └── loans.py
│   ├── services/
│   │   ├── books.py            # przypadki użycia: dodaj książkę, wypożycz...
│   │   └── uow.py              # UnitOfWork
│   └── schemas/
│       ├── __init__.py
│       ├── common.py           # Page[T], odpowiedzi błędów
│       ├── author.py
│       ├── book.py
│       └── loan.py
├── tests/
│   ├── conftest.py
│   └── test_api_books.py
├── .env.example
├── pyproject.toml
└── README.md
```

> 💡 Analogia — Pomyśl o restauracji z jednym dużym, otwartym pomieszczeniem, w którym kucharze, kelnerzy i klienci mieszają się bez podziału. Pierwszy dzień jest cudownie efektywny. Trzeci — katastrofalny. Katalogi to ściany: nie po to, żeby utrudniać ruch, ale żeby każdy wiedział, gdzie są jego narzędzia i czego nie wolno mu dotykać.

Reguła, którą warto wywiesić na ścianie, brzmi: **zależności płyną w jedną stronę**.

| Warstwa | Co robi | Czego jej nie wolno |
|---|---|---|
| `api/` (router) | parsuje HTTP, waliduje wejście, woła serwis, mapuje wynik na schemat | pisać zapytań SQL/ORM, znać `select()` |
| `services/` | realizuje przypadek użycia, decyduje o granicy transakcji, rzuca wyjątki domenowe | znać `Request`, `HTTPException`, `JSONResponse` |
| `repositories/` | buduje zapytania, tłumaczy parametry na warunki `where()` | commitować, znać reguł biznesowych |
| `models/` | opisuje schemat i mapowanie | znać Pydantic, FastAPI |
| `schemas/` | opisuje kontrakt HTTP (JSON) | znać `Session` |
| `core/` | konfiguracja, silnik, błędy | znać konkretne endpointy |

```text
   HTTP request
        │
        ▼
 ┌──────────────┐   Pydantic   ┌──────────────┐   obiekty   ┌───────────────┐
 │   router     │ ───────────▶ │   service    │ ──────────▶ │  repository   │
 │  (api/v1)    │ ◀─────────── │  (transakcja)│ ◀────────── │  (select())   │
 └──────────────┘   schematy   └──────────────┘    encje    └───────────────┘
        │                             │                            │
        │ HTTPException               │ wyjątki domenowe           │ SQL
        ▼                             ▼                            ▼
   JSON response              handlery błędów              AsyncSession ──▶ baza
```

> ⚠️ Pułapka — Najczęstszy grzech początkujących to „router, który robi wszystko”: w funkcji endpointu powstaje `select()`, obliczenia i commit. Wygląda niewinnie przy jednym endpoincie, ale po dwudziestu zaczynasz kopiować te same warunki filtrowania do trzech miejsc i nikt nie wie, gdzie kończy się transakcja. Warstwy są tanie w utrzymaniu dokładnie dlatego, że są nudne.

---

## 22.3 Konfiguracja: jedna prawda o środowisku

Zanim gdziekolwiek wstawimy `create_async_engine`, musimy wiedzieć **jak** i **z czym** się łączyć. Konfiguracja w kodzie („na sztywno”) to problem, który ujawnia się dopiero przy wdrożeniu: na produkcji hasło inne, pula większa, `echo` wyłączony. Rozwiązanie: jeden obiekt `Settings`, czytany ze zmiennych środowiskowych i pliku `.env`.

```python
# app/core/config.py
from functools import lru_cache

from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    """Cała konfiguracja aplikacji w jednym miejscu.

    Wartości można nadpisać zmiennymi środowiskowymi z prefiksem APP_,
    np. APP_DATABASE_URL, APP_SQL_ECHO=true.
    """

    model_config = SettingsConfigDict(
        env_file=".env",
        env_prefix="APP_",
        extra="ignore",
    )

    # Baza danych
    database_url: str = "sqlite+aiosqlite:///./library.db"
    sql_echo: bool = False
    db_pool_size: int = 5
    db_max_overflow: int = 10
    db_pool_recycle: int = 1800
    db_statement_timeout_ms: int = 5000

    # Aplikacja
    debug: bool = False
    cors_origins: list[str] = []
    default_loan_days: int = 30
    page_size_max: int = 100


@lru_cache
def get_settings() -> Settings:
    """Zwraca jeden, współdzielony obiekt konfiguracji (tworzony raz)."""
    return Settings()
```

Plik `.env.example` (wersjonowany w repozytorium, **bez** prawdziwych sekretów):

```bash
# .env.example — skopiuj do .env i uzupełnij
# SQLite (szybki start, nic nie trzeba instalować):
APP_DATABASE_URL=sqlite+aiosqlite:///./library.db

# PostgreSQL (produkcja):
# APP_DATABASE_URL=postgresql+asyncpg://library_app:haslo@localhost:5432/library

APP_DEBUG=false
APP_SQL_ECHO=false
APP_DB_POOL_SIZE=5
APP_DB_MAX_OVERFLOW=10
APP_CORS_ORIGINS=["http://localhost:3000"]
APP_DEFAULT_LOAN_DAYS=30
```

Trzy rzeczy warte zauważenia:

- `@lru_cache` na `get_settings()` sprawia, że ustawienia parsujemy **raz na proces**. To ważne nie tylko dla wydajności: silnik tworzymy w `lifespan` na podstawie tych samych ustawień, których użyje wszystko inne.
- `extra="ignore"` chroni przed wywaleniem aplikacji z powodu dodatkowej zmiennej środowiskowej w systemie (np. `APP_` w nazwie zupełnie innego narzędzia).
- `cors_origins` jest listą — `pydantic-settings` sparsuje ją z JSON-a w zmiennej środowiskowej.

> 💡 Analogia — `Settings` to tablica rozdzielcza w piwnicy budynku. Nie chodzisz po mieszkaniach i nie przestawiasz bezpieczników w każdym z osobna; jest jedno miejsce, w którym widać cały stan instalacji, i jedno miejsce, w którym dokonuje się zmian.

### Profile: dev, test, prod

Nie potrzeba do tego osobnych klas — wystarczą różne pliki `.env` i różne wartości w środowisku wdrożeniowym.

| Ustawienie | dev | test | prod |
|---|---|---|---|
| `database_url` | `sqlite+aiosqlite:///./library.db` | `sqlite+aiosqlite://` (in-memory) | `postgresql+asyncpg://…` |
| `sql_echo` | `true` (widzisz SQL) | `false` | `false` (**nigdy** `true`) |
| `debug` | `true` | `false` | `false` |
| `db_pool_size` | 5 | nie dotyczy | 10–20 (zależnie od liczby replik) |
| migracje | ręcznie, `alembic upgrade head` | `Base.metadata.create_all()` | `alembic upgrade head` w kroku wdrożenia |

---

## 22.4 Cykl życia aplikacji: silnik tworzony raz

FastAPI od wersji 0.93 udostępnia `lifespan` — kontekst, którego część przed `yield` wykonuje się przy starcie aplikacji, a część po `yield` przy jej zamykaniu. To jest właściwe miejsce na silnik.

```python
# app/core/db.py
from typing import Any

from sqlalchemy.ext.asyncio import (
    AsyncEngine,
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)

from app.core.config import Settings


def build_engine(settings: Settings) -> AsyncEngine:
    """Tworzy silnik asynchroniczny. Wywoływane RAZ na proces aplikacji."""
    kwargs: dict[str, Any] = {"echo": settings.sql_echo}

    # Parametry puli mają sens tylko dla baz sieciowych (PostgreSQL, MySQL).
    # SQLite (aiosqlite) używa NullPool/StaticPool, które nie przyjmują
    # pool_size ani max_overflow — przekazanie ich skończy się TypeError.
    if settings.database_url.startswith("postgresql"):
        kwargs.update(
            pool_size=settings.db_pool_size,
            max_overflow=settings.db_max_overflow,
            pool_pre_ping=True,  # wykrywa "martwe" połączenia po przerwie w sieci
            pool_recycle=settings.db_pool_recycle,
        )
    elif settings.database_url.startswith("sqlite"):
        # Bezpiecznik dla plikowego SQLite przy testach i pracy lokalnej.
        kwargs["connect_args"] = {"timeout": 30}

    return create_async_engine(settings.database_url, **kwargs)


def build_session_factory(engine: AsyncEngine) -> async_sessionmaker[AsyncSession]:
    """Fabryka sesji. expire_on_commit=False jest w async praktycznie obowiązkowe."""
    return async_sessionmaker(
        bind=engine,
        class_=AsyncSession,
        expire_on_commit=False,
        autoflush=True,
    )
```

Teraz `lifespan` i fabryka aplikacji:

```python
# app/main.py
import logging
from collections.abc import AsyncIterator
from contextlib import asynccontextmanager

from fastapi import FastAPI

from app.api.v1 import authors, books, loans
from app.core.config import get_settings
from app.core.db import build_engine, build_session_factory
from app.core.errors import register_exception_handlers

logger = logging.getLogger("library.api")


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[None]:
    settings = get_settings()
    engine = build_engine(settings)

    # Silnik i fabrykę sesji trzymamy na app.state — zależności FastAPI
    # odczytają je z request.app.state, więc nie potrzebujemy zmiennych globalnych.
    app.state.engine = engine
    app.state.session_factory = build_session_factory(engine)

    logger.info("Baza gotowa: dialekt=%s", engine.dialect.name)
    try:
        yield
    finally:
        await engine.dispose()
        logger.info("Pula połączeń zamknięta")


def create_app() -> FastAPI:
    settings = get_settings()
    app = FastAPI(
        title="Library API",
        version="1.0.0",
        lifespan=lifespan,
        # Dokumentacja interaktywna na produkcji bywa niechciana:
        docs_url="/docs" if settings.debug else None,
        redoc_url=None,
    )
    app.include_router(books.router, prefix="/api/v1")
    app.include_router(authors.router, prefix="/api/v1")
    app.include_router(loans.router, prefix="/api/v1")
    register_exception_handlers(app)
    return app


app = create_app()
```

> 🧠 Dlaczego tak jest — `lifespan` jest wywoływany dokładnie raz, gdy proces wstaje. Gdybyśmy umieścili tworzenie silnika w funkcji endpointu, każdy request budowałby nową pulę połączeń: kilkadziesiąt TCP-handshake’ów na sekundę, wyciek gniazd i najprawdopodobniej wyczerpanie limitu połączeń po stronie serwera bazy. Objaw: aplikacja działa świetnie lokalnie i przewraca się pod obciążeniem z komunikatem „too many clients already”.

`engine.dispose()` w `finally` zamyka wszystkie połączenia z puli. Bez tego proces nie zawsze kończy się czysto — zostają połączenia w stanie `idle` po stronie PostgreSQL, które trzymają zasoby przez `idle_session_timeout`.

> 🆕 SQLAlchemy 2.1 — Dwa fakty, które zmieniają ten fragment. Po pierwsze, `greenlet` nie jest już instalowany razem z SQLAlchemy, więc do async potrzebujesz jawnego `pip install "sqlalchemy[asyncio]"` — w przeciwnym razie dostaniesz `ValueError: the greenlet library is required to use this function`. Po drugie, w 2.1 domyślnym sterownikiem dla URL-a `postgresql://` jest `psycopg` (wersja 3), a nie `psycopg2`, więc skrócony URL działa „od razu” z nowocześniejszym sterownikiem. W 2.0.x `postgresql://` nadal wskazuje na `psycopg2` i musisz pisać `postgresql+psycopg://`, aby wymusić wersję 3.

> 🧪 Ćwiczenie — Dodaj do `lifespan` logowanie liczby połączeń w puli (`engine.pool.status()`) przed `yield` i po zakończeniu. Uruchom aplikację, wykonaj kilka żądań i sprawdź, czy liczba otwartych połączeń rośnie, czy wraca do poziomu bazowego.

---

## 22.5 Zarządzanie sesją w żądaniu HTTP

Sesja jest zasobem, który musi zostać zamknięty po każdym żądaniu — inaczej połączenie wróci do puli dopiero przy garbage collection, a przy dużym ruchu pula się wyczerpie. FastAPI pozwala napisać to raz, jako **zależność z `yield`**.

```python
# app/api/deps.py
from collections.abc import AsyncIterator

from fastapi import Request
from sqlalchemy.ext.asyncio import AsyncSession

from app.services.uow import UnitOfWork


async def get_session(request: Request) -> AsyncIterator[AsyncSession]:
    """Jedna sesja na jedno żądanie HTTP.

    Kod po `yield` wykonuje się PO zbudowaniu odpowiedzi (od wersji FastAPI 0.106
    wyjątki z endpointu też tu docierają, więc rollback działa).
    """
    factory = request.app.state.session_factory
    async with factory() as session:
        try:
            yield session
        except Exception:
            # Wyjątek z endpointu/serwisu: cofamy wszystko, co wydarzyło się
            # w tej transakcji, i pozwalamy wyjątkowi polecieć dalej do handlera.
            await session.rollback()
            raise
        # `async with` zamyka sesję (zwraca połączenie do puli) w każdym scenariuszu.


async def get_uow(request: Request) -> AsyncIterator[UnitOfWork]:
    """Jednostka pracy: sesja + zestaw repozytoriów + kontrola transakcji."""
    factory = request.app.state.session_factory
    async with UnitOfWork(factory()) as uow:
        yield uow
```

### Gdzie postawić `commit()`?

To najważniejsza decyzja architektoniczna w tym rozdziale i jedna z tych, na których ludzie najczęściej się przewracają. Rozważmy trzy możliwości.

| Wariant | Gdzie commit | Zaleta | Wada |
|---|---|---|---|
| **A. W zależności (commit-on-success)** | w `get_session` po `yield` | kod endpointu jest krótki, „wszystko samo się zapisuje” | commit odbywa się **po zbudowaniu odpowiedzi** — błąd bazy nie da się już zamienić na czytelny kod HTTP; trudno wyrazić „jedna operacja = dwie transakcje” |
| **B. W serwisie** | w metodzie serwisu realizującej przypadek użycia | granica transakcji jest widoczna tam, gdzie ma sens; błędy mapują się na HTTP naturalnie | trzeba pamiętać o `commit()` w każdej metodzie zapisującej |
| **C. W jednostce pracy (UoW)** | w `UnitOfWork.__aexit__` | jedna reguła dla wszystkich przypadków użycia; repozytoria i serwisy są „transakcyjnie nieświadome” | wymaga dyscypliny: serwis nie może commitować sam |

W tym module przyjmujemy **B dla prostych przypadków użycia i C dla złożonych**. Wariant A pokazujemy tylko po to, żebyś rozpoznał go w cudzym kodzie i wiedział, na co uważać.

```python
# WARIANT A — pokazany wyłącznie jako antywzorzec; NIE używamy go w projekcie.
async def get_session_autocommit(request: Request) -> AsyncIterator[AsyncSession]:
    factory = request.app.state.session_factory
    async with factory() as session:
        try:
            yield session
            await session.commit()  # za późno na obsługę IntegrityError w endpointcie
        except Exception:
            await session.rollback()
            raise
```

> ⚠️ Pułapka — Wariant A wygląda elegancko, dopóki nie trafisz na naruszenie ograniczenia unikalności. `commit()` wykonuje się już po tym, jak FastAPI zserializowało odpowiedź. Handler `IntegrityError` zamieni wtedy błąd na `409`, ale w logach zobaczysz, że aplikacja „zwróciła 409 na żądanie, które już odpowiedziało 201”. Klient nigdy nie dowie się, że transakcja się nie utrwaliła. Dlatego com mitujemy tam, gdzie wiemy, że jesteśmy w środku przypadku użycia.

### Wariant synchroniczny — dla porównania

Jeżeli piszesz aplikację na Flasce albo świadomie rezygnujesz z async, obrazek jest identyczny; zmieniają się tylko dwa słowa:

```python
# Wariant SYNC (np. dla Flask/Django-nie-ORM/CLI). Porównaj z wersją async powyżej.
from collections.abc import Iterator

from sqlalchemy.orm import Session, sessionmaker


def get_session_sync(session_factory: sessionmaker[Session]) -> Iterator[Session]:
    with session_factory() as session:  # Session jako kontekst sam robi close()
        try:
            yield session
        except Exception:
            session.rollback()
            raise
```

Różnica, którą warto zapamiętać: `Session` (sync) zamyka i rollbackuje automatycznie przy `with`; `AsyncSession` również obsługuje `async with`, ale **każda** operacja wymaga `await` — o tym w sekcji 22.12.

---

## 22.6 Modele, schematy Pydantic i problem „oderwanej encji”

### Modele

Nasza domena to biblioteka: autorzy, książki i wypożyczenia. Zaczynamy od wspólnej bazy deklaratywnej z konwencją nazewnictwa ograniczeń — bez niej Alembic generuje migracje z nazwami typu `None_1`, których nie da się potem usunąć w sposób przenośny.

```python
# app/models/base.py
from datetime import datetime

from sqlalchemy import DateTime, MetaData, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

NAMING_CONVENTION = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}


class Base(DeclarativeBase):
    metadata = MetaData(naming_convention=NAMING_CONVENTION)


class TimestampMixin:
    """Kolumny techniczne: kiedy rekord powstał i kiedy był ostatnio zmieniony."""

    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), nullable=False
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
        onupdate=func.now(),
        nullable=False,
    )
```

```python
# app/models/book.py
from typing import TYPE_CHECKING

from sqlalchemy import ForeignKey, String, Text
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.models.base import Base, TimestampMixin

if TYPE_CHECKING:  # tylko dla typowania — nie tworzy cyklicznego importu w runtime
    from app.models.author import Author
    from app.models.loan import Loan


class Book(TimestampMixin, Base):
    __tablename__ = "books"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200), nullable=False, index=True)
    isbn: Mapped[str] = mapped_column(String(20), nullable=False, unique=True)
    published_year: Mapped[int | None]
    copies_total: Mapped[int] = mapped_column(default=1, server_default="1", nullable=False)
    copies_available: Mapped[int] = mapped_column(default=1, server_default="1", nullable=False)

    author_id: Mapped[int] = mapped_column(
        ForeignKey("authors.id", ondelete="CASCADE"), index=True
    )
    author: Mapped["Author"] = relationship(back_populates="books", lazy="raise")

    loans: Mapped[list["Loan"]] = relationship(
        back_populates="book",
        cascade="all, delete-orphan",
        passive_deletes=True,  # usuwanie dzieci zostawiamy bazie (ON DELETE CASCADE)
        lazy="raise",
    )
```

Autor i wypożyczenie wyglądają analogicznie:

```python
# app/models/author.py
from typing import TYPE_CHECKING

from sqlalchemy import String, Text
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.models.base import Base, TimestampMixin

if TYPE_CHECKING:
    from app.models.book import Book


class Author(TimestampMixin, Base):
    __tablename__ = "authors"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(120), nullable=False, index=True)
    bio: Mapped[str | None] = mapped_column(Text)

    books: Mapped[list["Book"]] = relationship(
        back_populates="author",
        cascade="all, delete-orphan",
        passive_deletes=True,
        lazy="raise",
    )
```

```python
# app/models/loan.py
from datetime import datetime

from sqlalchemy import DateTime, ForeignKey, String
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.models.base import Base, TimestampMixin


class Loan(TimestampMixin, Base):
    __tablename__ = "loans"

    id: Mapped[int] = mapped_column(primary_key=True)
    book_id: Mapped[int] = mapped_column(
        ForeignKey("books.id", ondelete="CASCADE"), index=True
    )
    member_name: Mapped[str] = mapped_column(String(120), nullable=False)
    due_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
    returned_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))

    book: Mapped["Book"] = relationship(back_populates="loans", lazy="raise")
```

> 🧠 Dlaczego `lazy="raise"` w każdej relacji — W aplikacji asynchronicznej leniwe ładowanie jest niemożliwe w większości miejsc (nie ma gdzie „doczekać” operacji wejścia/wyjścia). Zamiast pozwolić, by zapomniane `selectinload()` wybuchło w produkcji jako `MissingGreenlet`, ustawiamy relacje tak, aby **każdy** dostęp bez jawnego wcześniejszego załadowania zgłaszał czytelny `InvalidRequestError: 'Book.author' is not available due to lazy='raise'`. Błąd pojawia się w testach, nie u klienta.

> ⚠️ Pułapka — `lazy="raise"` i `cascade="all, delete-orphan"` to konfiguracja, która potrafi się wzajemnie blokować. Przy `await session.delete(book)` SQLAlchemy musi zajrzeć do kolekcji `book.loans`, żeby usunąć osierocone dzieci — ale `lazy="raise"` zabrania załadowania tej kolekcji. Ratunkiem jest `passive_deletes=True` (użyte powyżej): skoro baza i tak skasuje wiersze przez `ON DELETE CASCADE`, SQLAlchemy nie musi nic ładować. Cena: obiekty `Loan` pozostające w mapie tożsamości staną się nieaktualne — po skasowaniu książki wywołaj `await session.expire_all()` albo pracuj na świeżej sesji.

### Schematy Pydantic v2

Encja ORM opisuje **tabelę**. Schemat Pydantic opisuje **kontrakt HTTP**. To dwie różne odpowiedzialności i nie wolno ich mieszać — konsekwencje omówimy w [module 19](19_warstwy_i_data_mapper.md), a w praktyce wygląda to tak:

```python
# app/schemas/common.py
from typing import Generic, TypeVar

from pydantic import BaseModel, ConfigDict, Field

T = TypeVar("T")


class Page(BaseModel, Generic[T]):
    """Uniwersalna strona wyników — wspólny kształt dla wszystkich list."""

    items: list[T]
    total: int
    limit: int
    offset: int
    has_next: bool


class ErrorBody(BaseModel):
    code: str
    message: str


class ErrorResponse(BaseModel):
    error: ErrorBody


class ORMModel(BaseModel):
    """Baza dla schematów odczytywanych z encji SQLAlchemy."""

    model_config = ConfigDict(from_attributes=True)
```

```python
# app/schemas/book.py
from datetime import datetime

from pydantic import BaseModel, Field

from app.schemas.common import ORMModel


class AuthorRead(ORMModel):
    id: int
    name: str
    bio: str | None = None


class BookCreate(BaseModel):
    title: str = Field(min_length=1, max_length=200)
    isbn: str = Field(min_length=10, max_length=20, pattern=r"^[0-9Xx\-]+$")
    published_year: int | None = Field(default=None, ge=1450, le=2100)
    author_id: int = Field(ge=1)
    copies_total: int = Field(default=1, ge=1, le=10_000)


class BookUpdate(BaseModel):
    # Wszystkie pola opcjonalne = PATCH. Brak pola ≠ pole ustawione na null!
    title: str | None = Field(default=None, min_length=1, max_length=200)
    isbn: str | None = Field(default=None, min_length=10, max_length=20)
    published_year: int | None = Field(default=None, ge=1450, le=2100)
    copies_total: int | None = Field(default=None, ge=1, le=10_000)


class BookRead(ORMModel):
    id: int
    title: str
    isbn: str
    published_year: int | None
    copies_total: int
    copies_available: int
    created_at: datetime
    author: AuthorRead


class BookSlim(ORMModel):
    """Lekki widok książki — bez autora, gdy nie jest potrzebny."""

    id: int
    title: str
    isbn: str
```

> 💡 Analogia — Encja to **akt osobowy** w segregatorze: zawiera wszystko, co urząd wie o człowieku, łącznie z odręcznymi notatkami na marginesie. Schemat Pydantic to **formularz urzędowy**, który wydajesz interesantowi: tylko wybrane pola, w ustalonym układzie, bez notatek. Gdybyś wydawał interesantom akta, dwa razy w tygodniu wyciekłyby dane, których nie chciałeś ujawniać.

### „Oderwana encja” — dlaczego nie zwracamy obiektów ORM

Sesja kończy się razem z żądaniem. Obiekt `Book`, który po tym pozostaje w pamięci, jest **oderwany** (detached). Dostęp do jego niezaładowanej relacji kończy się `DetachedInstanceError` (sync) lub `MissingGreenlet` (async). Dlatego `BookRead.author` wymaga, aby autor był załadowany **przed** zwróceniem odpowiedzi. Macie dwa narzędzia:

1. **Eager loading** — `options(joinedload(Book.author))` w repozytorium. Proste i wydajne dla relacji wiele-do-jednego.
2. **DTO** — repozytorium zwraca gotowy `BookRead` zbudowany z wybranych kolumn (`select(Book.id, Book.title, ...)`). Więcej kodu, ale kontrakt API jest niezależny od modelu ORM.

W tym projekcie stosujemy rozwiązanie pierwsze dla odczytów encji i drugie dla raportów. Poniżej działający dowód, że oba podejścia dają ten sam JSON:

```python
# examples/22_detached_vs_dto.py
import asyncio
from datetime import datetime

from sqlalchemy import select
from sqlalchemy.ext.asyncio import async_sessionmaker, create_async_engine
from sqlalchemy.orm import joinedload

from app.models import Base, Author, Book
from app.schemas.book import BookRead


async def main() -> None:
    engine = create_async_engine("sqlite+aiosqlite://")
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    factory = async_sessionmaker(engine, expire_on_commit=False)

    async with factory() as session:
        author = Author(name="Stanisław Lem")
        session.add(Book(title="Solaris", isbn="9780156027602", author=author, copies_total=3,
                         copies_available=3))
        await session.commit()

    # 1) EAGER LOADING — encja z załadowaną relacją, gotowa do serializacji.
    async with factory() as session:
        stmt = select(Book).options(joinedload(Book.author)).where(Book.isbn == "9780156027602")
        book = await session.scalar(stmt)
        assert book is not None
        payload = BookRead.model_validate(book)  # from_attributes=True czyta z encji
        print("z encji:", payload.model_dump()["author"]["name"])

    # 2) DTO — encja w ogóle nie opuszcza sesji; poza sesją mamy zwykłe dane.
    async with factory() as session:
        stmt = (
            select(
                Book.id, Book.title, Book.isbn, Book.published_year,
                Book.copies_total, Book.copies_available,
                Book.created_at.label("created_at"),
                Author.name.label("author_name"),
            )
            .join(Author, Book.author_id == Author.id)
            .where(Book.isbn == "9780156027602")
        )
        row = (await session.execute(stmt)).mappings().one()
        print("z DTO:", row["author_name"], row["title"], row["created_at"])

    await engine.dispose()


if __name__ == "__main__":
    asyncio.run(main())
```

> ⚠️ Pułapka — `from_attributes=True` **nie** chroni przed leniwym ładowaniem. Pydantic uprzejmie sięgnie po `book.author`, co przy `lazy="raise"` da `InvalidRequestError` (a przy `lazy="select"` w async — `MissingGreenlet`). Jeżeli w logach widzisz błąd przy serializacji odpowiedzi, a nie przy zapytaniu, to znak, że relacja nie została załadowana eageralnie. Naprawa to jedna linia w repozytorium: `options(joinedload(...))`.

> 🆕 SQLAlchemy 2.1 — Dla zagnieżdżonych chainów ładowania 2.1 poprawia deterministyczne łączenie opcji ładowania, gdy tę samą relację ładujesz różnymi ścieżkami (`joinedload(A.b).selectinload(B.c)` razem z `selectinload(A.b)`). W 2.0.x ostatnia opcja potrafiła „wygrywać” w nieoczywisty sposób. Dodatkowo `selectinload()` zyskał `omit_join` dla relacji wiele-do-wielu — pozwala pominąć dodatkowe złączenie używane wcześniej do odfiltrowania kluczy, co zmniejsza rozmiar zapytania.

---

## 22.7 Repozytorium: zapytania w jednym miejscu

Repozytorium z [modułu 20](20_repository.md) przenosimy bez zmian koncepcyjnych — dostosowujemy tylko typ sesji na `AsyncSession`.

```python
# app/repositories/books.py
from collections.abc import Sequence

from sqlalchemy import ColumnElement, Select, func, select
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.orm import joinedload

from app.models import Book


class BookRepository:
    """Wszystkie zapytania dotyczące książek. Nie commituje — to nie jego rola."""

    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    def _conditions(
        self,
        *,
        q: str | None,
        author_id: int | None,
        available_only: bool,
    ) -> list[ColumnElement[bool]]:
        """Buduje listę warunków wielokrotnego użytku (licznik + strona wyników)."""
        conditions: list[ColumnElement[bool]] = []
        if q:
            pattern = f"%{q}%"
            conditions.append(Book.title.ilike(pattern) | Book.isbn.ilike(pattern))
        if author_id is not None:
            conditions.append(Book.author_id == author_id)
        if available_only:
            conditions.append(Book.copies_available > 0)
        return conditions

    async def get(self, book_id: int) -> Book | None:
        stmt: Select = (
            select(Book).options(joinedload(Book.author)).where(Book.id == book_id)
        )
        return await self._session.scalar(stmt)

    async def list_page(
        self,
        *,
        limit: int,
        offset: int,
        q: str | None = None,
        author_id: int | None = None,
        available_only: bool = False,
    ) -> tuple[Sequence[Book], int]:
        conditions = self._conditions(q=q, author_id=author_id, available_only=available_only)

        total = await self._session.scalar(
            select(func.count()).select_from(Book).where(*conditions)
        )
        stmt = (
            select(Book)
            .where(*conditions)
            .options(joinedload(Book.author))  # ← bez tego BookRead.author wybuchnie
            .order_by(Book.title, Book.id)     # ← deterministyczne sortowanie!
            .limit(limit)
            .offset(offset)
        )
        rows = (await self._session.scalars(stmt)).all()
        return rows, int(total or 0)

    async def exists_isbn(self, isbn: str) -> bool:
        stmt = select(select(Book.id).where(Book.isbn == isbn).exists())
        return bool(await self._session.scalar(stmt))

    async def add(self, book: Book) -> Book:
        self._session.add(book)
        await self._session.flush()  # nadaje id bez kończenia transakcji
        return book

    async def delete(self, book: Book) -> None:
        await self._session.delete(book)
```

Autorzy:

```python
# app/repositories/authors.py
from collections.abc import Sequence

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.orm import selectinload

from app.models import Author, Book


class AuthorRepository:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def get(self, author_id: int) -> Author | None:
        return await self._session.get(Author, author_id)

    async def list_all(self) -> Sequence[Author]:
        return (await self._session.scalars(select(Author).order_by(Author.name))).all()

    async def get_with_books(self, author_id: int) -> Author | None:
        """Autor WRAZ z kolekcją książek — kolekcja wymaga selectinload, nie joinedload."""
        stmt = (
            select(Author)
            .options(selectinload(Author.books))
            .where(Author.id == author_id)
        )
        return await self._session.scalar(stmt)

    async def list_books(self, author_id: int) -> Sequence[Book]:
        stmt = select(Book).where(Book.author_id == author_id).order_by(Book.title)
        return (await self._session.scalars(stmt)).all()

    def add(self, author: Author) -> Author:
        self._session.add(author)
        return author
```

> 🔬 Pod maską — Dla żądania `GET /api/v1/books?limit=20&available_only=true` repozytorium wyemituje dokładnie dwa zapytania (licznik + strona). To jest SQL dla PostgreSQL:

```sql
-- 1) licznik
SELECT count(*) AS count_1
FROM books
WHERE books.copies_available > 0;

-- 2) strona wyników (jedno zapytanie, autor dociągnięty przez LEFT OUTER JOIN)
SELECT books.id, books.title, books.isbn, books.published_year,
       books.copies_total, books.copies_available, books.author_id,
       books.created_at, books.updated_at,
       authors_1.id AS id_1, authors_1.name, authors_1.bio
FROM books
LEFT OUTER JOIN authors AS authors_1 ON authors_1.id = books.author_id
WHERE books.copies_available > 0
ORDER BY books.title, books.id
LIMIT 20 OFFSET 0;
```

> 🧠 Dlaczego tak jest — `joinedload` na relacji wiele-do-jednego nie tworzy kartezjańskiego „wybuchu”: każdy wiersz `books` pasuje najwyżej do jednego autora, więc liczba wierszy się nie zmienia. Dlatego **nie** potrzebujemy `.unique()` ani drugiego zapytania. Odwrotnie przy kolekcjach: `joinedload(Author.books)` zwielokrotnia wiersze i wymaga `unique()` — dlatego dla kolekcji używamy `selectinload`.

---

## 22.8 Serwis: granica transakcji i reguły biznesowe

Serwis to miejsce, w którym „techniczne” operacje zyskują sens biznesowy. Tutaj decydujemy, co jest jedną transakcją, i tutaj rzucamy wyjątki domenowe, które routery zamienią na kody HTTP.

Najpierw wyjątki:

```python
# app/core/errors.py
import logging

from fastapi import FastAPI, Request, status
from fastapi.responses import JSONResponse
from sqlalchemy.exc import IntegrityError

logger = logging.getLogger("library.errors")


class DomainError(Exception):
    """Bazowy wyjątek domenowy. Warstwa HTTP nie zna szczegółów SQLAlchemy."""

    code: str = "domain_error"
    status_code: int = status.HTTP_400_BAD_REQUEST

    def __init__(self, message: str = "") -> None:
        self.message = message or self.code
        super().__init__(self.message)


class NotFoundError(DomainError):
    code = "not_found"
    status_code = status.HTTP_404_NOT_FOUND


class ConflictError(DomainError):
    code = "conflict"
    status_code = status.HTTP_409_CONFLICT


class BookNotFound(NotFoundError):
    def __init__(self, book_id: int) -> None:
        super().__init__(f"Nie znaleziono książki o id={book_id}")


class AuthorNotFound(NotFoundError):
    def __init__(self, author_id: int) -> None:
        super().__init__(f"Nie znaleziono autora o id={author_id}")


class DuplicateISBN(ConflictError):
    code = "duplicate_isbn"

    def __init__(self, isbn: str) -> None:
        super().__init__(f"Książka o ISBN {isbn} już istnieje")


class NoCopiesAvailable(ConflictError):
    code = "no_copies_available"

    def __init__(self, book_id: int) -> None:
        super().__init__(f"Brak dostępnych egzemplarzy książki o id={book_id}")


def register_exception_handlers(app: FastAPI) -> None:
    """Jedno miejsce, w którym wyjątki zamieniają się w kontrakt HTTP."""

    @app.exception_handler(DomainError)
    async def _domain_error(_: Request, exc: DomainError) -> JSONResponse:
        logger.info("Wyjątek domenowy: %s (%s)", exc.code, exc.message)
        return JSONResponse(
            status_code=exc.status_code,
            content={"error": {"code": exc.code, "message": exc.message}},
        )

    @app.exception_handler(IntegrityError)
    async def _integrity_error(_: Request, exc: IntegrityError) -> JSONResponse:
        # Siatka bezpieczeństwa: reguły sprawdzone w serwisie, ale wyścig dwóch
        # żądań mógł wyprzedzić walidację. Nie logujemy SQL-a (może zawierać dane).
        logger.warning("Naruszenie ograniczenia bazy: %s", type(exc.orig).__name__)
        return JSONResponse(
            status_code=status.HTTP_409_CONFLICT,
            content={
                "error": {
                    "code": "integrity_error",
                    "message": "Operacja narusza ograniczenia bazy danych.",
                }
            },
        )

    @app.exception_handler(Exception)
    async def _unhandled(_: Request, exc: Exception) -> JSONResponse:
        logger.exception("Nieobsłużony błąd serwera")
        return JSONResponse(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            content={
                "error": {
                    "code": "internal_error",
                    "message": "Wewnętrzny błąd serwera.",
                }
            },
        )
```

> 🧠 Dlaczego tak jest — Kolejność jest tu kluczowa. Wyjątek leci z endpointu → zależność `get_session` łapie go w `except`, robi `rollback()` i **ponownie rzuca** → dopiero potem przechwytuje go handler i zamienia na JSON. Gdyby rollback był tylko w handlerze, sesja wracałaby do stanu „prepared” i kolejne żądanie na tym samym połączeniu mogłoby się przewrócić. Rollback należy do właściciela sesji, czyli zależności.

Teraz serwis CRUD-owy:

```python
# app/services/books.py
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.errors import BookNotFound, AuthorNotFound, ConflictError, DuplicateISBN
from app.models import Book
from app.repositories.authors import AuthorRepository
from app.repositories.books import BookRepository
from app.schemas.book import BookCreate, BookUpdate
from app.schemas.common import Page


class BookService:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session
        self._books = BookRepository(session)
        self._authors = AuthorRepository(session)

    async def list_books(
        self,
        *,
        limit: int,
        offset: int,
        q: str | None = None,
        author_id: int | None = None,
        available_only: bool = False,
    ) -> Page[Book]:
        rows, total = await self._books.list_page(
            limit=limit, offset=offset, q=q, author_id=author_id, available_only=available_only
        )
        return Page(
            items=list(rows),
            total=total,
            limit=limit,
            offset=offset,
            has_next=offset + len(rows) < total,
        )

    async def get_book(self, book_id: int) -> Book:
        book = await self._books.get(book_id)
        if book is None:
            raise BookNotFound(book_id)
        return book

    async def create_book(self, data: BookCreate) -> Book:
        author = await self._authors.get(data.author_id)
        if author is None:
            raise AuthorNotFound(data.author_id)
        if await self._books.exists_isbn(data.isbn):
            raise DuplicateISBN(data.isbn)

        book = Book(
            title=data.title,
            isbn=data.isbn,
            published_year=data.published_year,
            copies_total=data.copies_total,
            copies_available=data.copies_total,
            author=author,  # ustawiamy relację, nie sam author_id — autor jest już w pamięci
        )
        await self._books.add(book)
        await self._session.commit()  # ← GRANICA TRANSAKCJI: koniec przypadku użycia
        return book

    async def update_book(self, book_id: int, data: BookUpdate) -> Book:
        book = await self.get_book(book_id)
        changes = data.model_dump(exclude_unset=True)  # PATCH: tylko przesłane pola

        if "isbn" in changes and changes["isbn"] != book.isbn:
            if await self._books.exists_isbn(changes["isbn"]):
                raise DuplicateISBN(changes["isbn"])

        if "copies_total" in changes:
            # Reguła biznesowa: nie można zmniejszyć całkowitej liczby egzemplarzy
            # poniżej liczby aktualnie wypożyczonych.
            delta = changes["copies_total"] - book.copies_total
            if book.copies_available + delta < 0:
                raise ConflictError(
                    "Nie można zmniejszyć liczby egzemplarzy poniżej liczby wypożyczonych."
                )
            book.copies_available += delta

        for field, value in changes.items():
            setattr(book, field, value)

        await self._session.commit()
        return book

    async def delete_book(self, book_id: int) -> None:
        book = await self.get_book(book_id)
        await self._books.delete(book)
        await self._session.commit()
```

> ⚠️ Pułapka — `data.model_dump(exclude_unset=True)` to jedyny poprawny sposób na PATCH. Bez `exclude_unset` pole `published_year=None` (nieprzesłane) nadpisałoby w bazie istniejącą wartość, bo `model_dump()` zwraca wszystkie pola z wartościami domyślnymi. Objaw: „wysłałem tylko tytuł, a zniknął mi rok wydania”.

---

## 22.9 Router i siedem endpointów

```python
# app/api/v1/books.py
from typing import Annotated

from fastapi import APIRouter, Depends, Query, status
from sqlalchemy.ext.asyncio import AsyncSession

from app.api.deps import get_session
from app.schemas.book import BookRead, BookCreate, BookUpdate
from app.schemas.common import Page
from app.services.books import BookService

router = APIRouter(prefix="/books", tags=["books"])

SessionDep = Annotated[AsyncSession, Depends(get_session)]


@router.get("", response_model=Page[BookRead], summary="Lista książek (paginacja + filtry)")
async def list_books(
    session: SessionDep,
    q: Annotated[str | None, Query(max_length=100, description="Szuka w tytule i ISBN")] = None,
    author_id: Annotated[int | None, Query(ge=1)] = None,
    available_only: bool = False,
    limit: Annotated[int, Query(ge=1, le=100)] = 20,
    offset: Annotated[int, Query(ge=0)] = 0,
) -> Page[BookRead]:
    page = await BookService(session).list_books(
        limit=limit, offset=offset, q=q, author_id=author_id, available_only=available_only
    )
    return Page[BookRead](
        items=[BookRead.model_validate(item) for item in page.items],
        total=page.total,
        limit=page.limit,
        offset=page.offset,
        has_next=page.has_next,
    )


@router.get("/{book_id}", response_model=BookRead, summary="Szczegóły książki")
async def get_book(book_id: int, session: SessionDep) -> BookRead:
    book = await BookService(session).get_book(book_id)
    return BookRead.model_validate(book)


@router.post(
    "",
    response_model=BookRead,
    status_code=status.HTTP_201_CREATED,
    summary="Dodaj książkę",
)
async def create_book(payload: BookCreate, session: SessionDep) -> BookRead:
    book = await BookService(session).create_book(payload)
    return BookRead.model_validate(book)


@router.patch("/{book_id}", response_model=BookRead, summary="Zaktualizuj książkę")
async def update_book(book_id: int, payload: BookUpdate, session: SessionDep) -> BookRead:
    book = await BookService(session).update_book(book_id, payload)
    return BookRead.model_validate(book)


@router.delete(
    "/{book_id}",
    status_code=status.HTTP_204_NO_CONTENT,
    summary="Usuń książkę",
)
async def delete_book(book_id: int, session: SessionDep) -> None:
    await BookService(session).delete_book(book_id)
```

Router autorów dostarcza szósty endpoint:

```python
# app/api/v1/authors.py
from typing import Annotated

from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession

from app.api.deps import get_session
from app.repositories.authors import AuthorRepository
from app.schemas.book import BookSlim

router = APIRouter(prefix="/authors", tags=["authors"])

SessionDep = Annotated[AsyncSession, Depends(get_session)]


@router.get("/{author_id}/books", response_model=list[BookSlim], summary="Książki autora")
async def list_author_books(author_id: int, session: SessionDep) -> list[BookSlim]:
    books = await AuthorRepository(session).list_books(author_id)
    return [BookSlim.model_validate(book) for book in books]
```

Siódmy endpoint — wypożyczenie — omówimy osobno w sekcji 22.11, bo wymaga blokady wiersza.

> 🔬 Pod maską — Dla `POST /api/v1/books` z ciałem `{"title": "Solaris", "isbn": "9780156027602", "author_id": 1, "copies_total": 3}` aplikacja wykonuje **cztery** rundy do bazy:

```sql
-- 1) pobranie autora (selectin? nie: session.get, więc PK lookup)
SELECT authors.id, authors.name, authors.bio, authors.created_at, authors.updated_at
FROM authors WHERE authors.id = 1;

-- 2) sprawdzenie ISBN (EXISTS)
SELECT EXISTS (SELECT books.id FROM books WHERE books.isbn = '9780156027602') AS anon_1;

-- 3) wstawienie książki
INSERT INTO books (title, isbn, published_year, copies_total, copies_available, author_id,
                   created_at, updated_at)
VALUES ('Solaris', '9780156027602', NULL, 3, 3, 1, now(), now())
RETURNING books.id, books.created_at, books.updated_at;

-- 4) commit
COMMIT;
```

Warto to policzyć: cztery podróże do bazy na jedno żądanie to normalna cena czytelności i poprawnych komunikatów błędów. Jeśli endpoint jest bardzo gorący, można zamienić kroki 1–2 na jedno zapytanie `INSERT ... SELECT` z warunkami — ale dopiero po pomiarze. Zasada z [modułu 17](17_wydajnosc.md) obowiązuje bez wyjątków: najpierw liczba, potem optymalizacja.

---

## 22.10 Obsługa błędów: od wyjątku domenowego do JSON-a

Kontrakt błędów warto ustalić raz i trzymać się go we wszystkich endpointach:

```json
{
  "error": {
    "code": "no_copies_available",
    "message": "Brak dostępnych egzemplarzy książki o id=7"
  }
}
```

| Sytuacja | Warstwa, która wykrywa | Wyjątek / kod | HTTP |
|---|---|---|---|
| niepoprawne ciało żądania (brak pola, zły typ) | Pydantic | `RequestValidationError` | `422` |
| brak zasobu | serwis | `BookNotFound` | `404` |
| duplikat ISBN | serwis | `DuplicateISBN` | `409` |
| nie można spełnić reguły biznesowej | serwis | `ConflictError` | `409` |
| wyścig, naruszenie ograniczenia bazy | baza → `IntegrityError` | handler `IntegrityError` | `409` |
| błąd programisty | wszędzie | dowolny inny | `500` + log |

> 🧠 Dlaczego walidacja w dwóch miejscach ma sens — Pydantic sprawdza **kształt** (czy pole istnieje, czy rok jest liczbą, czy ISBN pasuje do wzorca). Serwis sprawdza **sens** (czy autor istnieje, czy ISBN nie jest zajęty, czy liczba egzemplarzy nie spadnie poniżej zera). Baza sprawdza **niezmiennik** (unikalność, klucz obcy) i jest ostatnią linią obrony przy wyścigu. Trzy poziomy, trzy różne scenariusze awarii. Próba upchnięcia wszystkiego w jednym poziomie zawsze kończy się dziurą.

> ⚠️ Pułapka — `IntegrityError` nie mówi elegancko, **co** naruszyłeś. Komunikat PostgreSQL wygląda tak:

```text
asyncpg.exceptions.UniqueViolationError: duplicate key value violates unique constraint "uq_books_isbn"
DETAIL:  Key (isbn)=(9780156027602) already exists.
```

Da się z tego wyciągnąć nazwę ograniczenia (dlatego `naming_convention` z sekcji 22.6 jest tak ważne!), ale to parsowanie tekstu sterownika. Poprawna kolejność: **najpierw** sprawdź w serwisie i rzuć `DuplicateISBN`, a `IntegrityError` traktuj jako siatkę bezpieczeństwa dla wyścigu, nie jako mechanizm walidacji.

---

## 22.11 Operacja złożona: wypożyczenie z blokadą wiersza

Teraz najciekawszy przypadek biznesowy i miejsce, w którym wracamy do [modułu 14](14_transakcje_i_wspolbieznosc.md). Scenariusz: dwóch użytkowników próbuje wypożyczyć ostatni egzemplarz tej samej książki w tej samej milisekundzie. Naiwna implementacja wygląda tak:

```python
# ⛔ ANTYWZORZEC — wyścig: dwóch czytelników zobaczy copies_available == 1
book = await session.get(Book, book_id)
if book.copies_available > 0:
    book.copies_available -= 1
    session.add(Loan(...))
```

Oba żądania odczytają `copies_available == 1`, oba przejdą warunek, oba zapiszą zero — i powstanie **dwa** wypożyczenia przy jednym egzemplarzu. Naprawiamy to dwiema warstwami: blokadą wiersza (`SELECT ... FOR UPDATE`) i kontrolą stanu w bazie.

```python
# app/services/uow.py
from types import TracebackType

from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker

from app.repositories.authors import AuthorRepository
from app.repositories.books import BookRepository
from app.repositories.loans import LoanRepository


class UnitOfWork:
    """Jedna transakcja, jedna sesja, wszystkie repozytoria.

    `async with UnitOfWork(session)`:
      - brak wyjątku  → COMMIT
      - wyjątek       → ROLLBACK (i wyjątek leci dalej)
      - zawsze        → CLOSE (zwrot połączenia do puli)
    """

    def __init__(self, session: AsyncSession) -> None:
        self.session = session
        self.books = BookRepository(session)
        self.authors = AuthorRepository(session)
        self.loans = LoanRepository(session)

    async def __aenter__(self) -> "UnitOfWork":
        return self

    async def __aexit__(
        self,
        exc_type: type[BaseException] | None,
        exc: BaseException | None,
        tb: TracebackType | None,
    ) -> None:
        try:
            if exc_type is None:
                await self.session.commit()
            else:
                await self.session.rollback()
        finally:
            await self.session.close()
```

```python
# app/repositories/loans.py
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.models import Book, Loan


class LoanRepository:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def get_book_for_update(self, book_id: int) -> Book | None:
        """Blokuje wiersz książki do końca transakcji (SELECT ... FOR UPDATE)."""
        stmt = select(Book).where(Book.id == book_id).with_for_update()
        return await self._session.scalar(stmt)

    def add(self, loan: Loan) -> Loan:
        self._session.add(loan)
        return loan

    async def list_active(self, *, limit: int, offset: int) -> list[Loan]:
        stmt = (
            select(Loan)
            .where(Loan.returned_at.is_(None))
            .order_by(Loan.due_at, Loan.id)
            .limit(limit)
            .offset(offset)
        )
        return list((await self._session.scalars(stmt)).all())
```

```python
# app/services/loans.py
from datetime import UTC, datetime, timedelta

from app.core.errors import BookNotFound, NoCopiesAvailable
from app.models import Loan
from app.schemas.loan import LoanCreate
from app.services.uow import UnitOfWork


class LoanService:
    """Serwis operujący wewnątrz jednostki pracy — NIE commituje sam."""

    def __init__(self, uow: UnitOfWork, default_days: int = 30) -> None:
        self._uow = uow
        self._default_days = default_days

    async def borrow(self, data: LoanCreate) -> Loan:
        book = await self._uow.loans.get_book_for_update(data.book_id)
        if book is None:
            raise BookNotFound(data.book_id)
        if book.copies_available <= 0:
            raise NoCopiesAvailable(data.book_id)

        book.copies_available -= 1  # UPDATE books SET copies_available = ... (po commicie)

        days = data.days or self._default_days
        loan = Loan(
            book_id=book.id,
            member_name=data.member_name,
            due_at=datetime.now(UTC) + timedelta(days=days),
        )
        self._uow.loans.add(loan)
        await self._uow.session.flush()  # nadaje loan.id, DML leci do bazy
        return loan
```

```python
# app/api/v1/loans.py
from typing import Annotated

from fastapi import APIRouter, Depends, status

from app.api.deps import get_uow
from app.core.config import get_settings
from app.schemas.loan import LoanCreate, LoanRead
from app.services.loans import LoanService
from app.services.uow import UnitOfWork

router = APIRouter(prefix="/loans", tags=["loans"])

UowDep = Annotated[UnitOfWork, Depends(get_uow)]


@router.post("", response_model=LoanRead, status_code=status.HTTP_201_CREATED)
async def create_loan(payload: LoanCreate, uow: UowDep) -> LoanRead:
    settings = get_settings()
    service = LoanService(uow, default_days=settings.default_loan_days)
    loan = await service.borrow(payload)
    # Zwracamy schemat zbudowany z encji, ZANIM UnitOfWork zamknie transakcję.
    return LoanRead.model_validate(loan)
```

> 🔬 Pod maską — `with_for_update()` w PostgreSQL generuje:

```sql
SELECT books.id, books.title, books.isbn, books.published_year,
       books.copies_total, books.copies_available, books.author_id,
       books.created_at, books.updated_at
FROM books
WHERE books.id = 7
FOR UPDATE;
-- ... UPDATE books SET copies_available = 2 ... ;
-- ... INSERT INTO loans (...) VALUES (...) RETURNING loans.id, loans.created_at ...;
-- COMMIT;   ← dopiero tutaj blokada pęka
```

Pierwsza transakcja trzyma blokadę wiersza do `COMMIT`. Druga **czeka** na zwolnienie blokady, a potem odczytuje już zaktualizowaną wartość `copies_available = 2` i — jeśli to było ostatnie zero — rzuca `NoCopiesAvailable` (409). Wyścig rozwiązany w bazie, nie w aplikacji.

> ⚠️ Pułapka — **SQLite nie obsługuje `FOR UPDATE`.** Dialekt SQLAlchemy po prostu pomija klauzulę (nie zgłasza błędu, ale i nie blokuje wiersza). Wniosek dla testów: test „dwóch równoległych wypożyczeń” na SQLite niczego nie dowiedzie. Albo piszemy go na PostgreSQL (testcontainers — patrz [moduł 18](18_testowanie.md)), albo testujemy samą regułę `copies_available <= 0` w izolacji, świadomie deklarując, że warstwa współbieżności pozostaje nieprzetestowana. Druga opcja jest akceptowalna tylko wtedy, gdy jest zapisana, a nie przemilczana.

> 💡 Analogia — Blokada wiersza to **kłódka na pojedynczej kartce w katalogu bibliotecznym**, nie na całym katalogu. Kolejny bibliotekarz może obsługiwać inne książki, ale ten jeden wpis musi poczekać, aż skończysz. Gdybyśmy zamiast tego zablokowali całą tabelę (`LOCK TABLE`), jedna wypożyczająca osoba zatrzymałaby całą bibliotekę.

> 🧪 Ćwiczenie — Wypisz w komentarzu w kodzie trzy scenariusze, w których `FOR UPDATE` **nie** wystarczy do poprawnego naliczenia kary za przetrzymanie książki (podpowiedź: czas, strefy czasowe, zwroty warunkowe przychodzące w innej kolejności niż wypożyczenia).

---

## 22.12 Async w FastAPI: `async def`, `def`, wątki i zadania w tle

FastAPI pozwala napisać endpoint dwoma sposobami — i wybór między nimi nie jest kosmetyczny.

| | `async def` | `def` |
|---|---|---|
| Wątek | pętla zdarzeń (główny wątek) | threadpool (AnyIO, domyślnie 40 wątków) |
| Sesja | `AsyncSession` | `Session` (sync) |
| Zachowanie przy blokującym IO | **blokuje całą aplikację** | blokuje jeden wątek z puli |
| Kiedy używać | async driver (`asyncpg`, `aiosqlite`), dużo operacji IO | sterownik sync, biblioteka bez async (np. stare SDK) |

> 🧠 Dlaczego tak jest — Pętla zdarzeń to jeden wątek, który obsługuje wszystkie korutyny. Jeżeli w endpointcie `async def` wykonasz blokującą operację (np. `requests.get()` albo synchroniczne zapytanie na `psycopg2`), zatrzymujesz **wszystkie** równoległe żądania aplikacji. FastAPI tego nie wykryje — zobaczysz tylko, że „pod obciążeniem wszystko zwalnia”. Odwrotnie: endpoint `def` jest wykonywany w threadpoolu, więc blokowanie jest tam dozwolone, ale kończy się dławieniem przy ~40 równoległych żądaniach.

W tym projekcie jedziemy jedną drogą: `async def` + `AsyncSession`. Konsekwencje są poważne i trzeba je znać:

1. **`await` przed każdą operacją sesji.** `await session.get(...)`, `await session.commit()`, `await session.flush()`, `await session.delete(obj)`. Zapomniany `await` na `session.flush()` daje korutynę, która nigdy się nie wykona — i najczęściej cichy `RuntimeWarning: coroutine was never awaited`.
2. **`expire_on_commit=False`.** Po commicie sesja domyślnie wygasza obiekty, żeby przy następnym odczycie pobrać świeże dane. W async taki odczyt oznacza operację wejścia/wyjścia, której nie da się wykonać bez `await` — i wybucha `MissingGreenlet`. Wyłączając wygaszanie, obiekt zachowuje wartości, które sam ustawiłeś.
3. **Leniwe ładowanie nie działa.** Stąd `lazy="raise"` w modelach i jawne `joinedload`/`selectinload`.

```python
# examples/22_sync_threadpool.py — pokazuje cenę blokowania pętli zdarzeń
import asyncio

from fastapi import FastAPI
from fastapi.concurrency import run_in_threadpool

app = FastAPI()


@app.get("/blocking")
async def blocking() -> dict[str, str]:
    # ⛔ Blokuje pętlę zdarzeń na 0.5 s — wszystkie inne żądania stoją.
    await asyncio.sleep(0)  # miejsce po to, żeby było widać, o co chodzi
    asyncio.get_running_loop().run_in_executor  # (w prawdziwym kodzie: requests.get(...))
    return {"status": "ok"}


@app.get("/threadpool")
async def threadpool_ok() -> dict[str, str]:
    # ✅ Ciężka/blokująca praca oddelegowana do wątku.
    await run_in_threadpool(sum, range(10_000_000))
    return {"status": "ok"}
```

### Zadania w tle

`BackgroundTasks` wykonuje się po wysłaniu odpowiedzi — ale **nie przedłuża życia zależności**. Sesja z `get_session` zostanie zamknięta, zanim zadanie wystartuje.

```python
from fastapi import BackgroundTasks


@router.post("/books/{book_id}/notify")
async def notify_about_book(
    book_id: int,
    session: SessionDep,
    background: BackgroundTasks,
) -> dict[str, str]:
    book = await BookService(session).get_book(book_id)
    title = book.title  # ✅ skopiuj DANE, nie encję

    async def send_email(title: str) -> None:
        # Nowa sesja, bo poprzednia już nie żyje. Albo lepiej: osobna kolejka zadań.
        await email_gateway.send(f"Nowość w bibliotece: {title}")

    background.add_task(send_email, title)
    return {"status": "kolejka"}
```

> ⚠️ Pułapka — Przekazanie do `BackgroundTasks` **obiektu encji** (`background.add_task(send_email, book)`) działa, dopóki zadanie nie spróbuje dotknąć niezaładowanej relacji: sesja jest już zamknięta, obiekt oderwany, a przy `lazy="raise"` dostaniesz `InvalidRequestError`. Jeszcze gorzej: przy `Session` sync bez `expire_on_commit=False` dostaniesz `DetachedInstanceError`. Reguła bez wyjątków: do zadań w tle przekazujemy **proste dane** (identyfikatory, stringi, słowniki), nie encje. A poważne zadania (`BackgroundTasks` ginie razem z procesem przy restarcie) kierujemy do kolejki — Celery, Dramatiq, Arq, RQ.

> 🆕 SQLAlchemy 2.1 — Dwie zmiany istotne dla tego rozdziału. `autoflush` w `Session` działa bezwarunkowo — wcześniej istniały ścieżki, w których zdarzenie `before_flush` mogło zostać pominięte; jeśli budowałeś logikę na tych zdarzeniach (moduł 13), zachowanie będzie teraz bardziej przewidywalne. Druga: w 2.1 trzeba jawnie zainstalować `"sqlalchemy[asyncio]"`, bo `greenlet` nie przychodzi „gratis”. Objaw przy starcie aplikacji: `ValueError: the greenlet library is required to use this function`.

---

## 22.13 Migracje i dane słownikowe we wdrożeniu

### Nie uruchamiaj `create_all()` przy starcie

`Base.metadata.create_all()` jest świetne w testach i w pierwszym prototypie, ale na produkcji jest niebezpieczne z czterech powodów:

1. Nie wykonuje zmian — brakuje kolumny w kodzie? `create_all` jej nie doda, bo tabela już istnieje. Aplikacja wstanie i wywali się na pierwszym zapytaniu.
2. Przy N replikach (Kubernetes, AWS ECS) N procesów równolegle próbuje modyfikować schemat.
3. Nie ma historii zmian ani `downgrade`.
4. Nie da się „zatrzymać” migracji — jest wpisana w start procesu, więc nieudana migracja blokuje podniesienie aplikacji.

Zamiast tego: **`alembic upgrade head` jako osobny krok wdrożenia** — job, krok pipeline’u, `initContainer`, `migrate` w etykiecie wdrożeniowej. Aplikacja startuje dopiero, gdy schemat jest gotowy.

```bash
# Krok 1: inicjalizacja szablonu async (raz w projekcie)
alembic init -t async alembic

# Krok 2: utworzenie migracji
alembic revision --autogenerate -m "books, authors, loans"

# Krok 3: przegląd pliku w alembic/versions/... — ZAWSZE ręcznie!

# Krok 4: wdrożenie
alembic upgrade head
```

### `env.py` w wersji async

```python
# alembic/env.py
"""Konfiguracja Alembica dla projektu asynchronicznego.

Ręcznie zmienione względem szablonu `alembic init -t async alembic`:
  * URL bierzemy z Settings (zmienne środowiskowe), a nie z alembic.ini;
  * dołączamy `compare_type` i `compare_server_default` (autogenerate widzi więcej);
  * włączamy `render_as_batch` dla SQLite (bez tego `ALTER COLUMN` jest no-opem).
"""

import asyncio
from logging.config import fileConfig

from alembic import context
from sqlalchemy import pool
from sqlalchemy.engine import Connection
from sqlalchemy.ext.asyncio import async_engine_from_config

from app.core.config import get_settings
from app.models import Base  # import PAKIETU — rejestruje wszystkie modele w metadata!

config = context.config

# Jedno źródło prawdy o środowisku: te same Settings, których używa aplikacja.
settings = get_settings()
config.set_main_option("sqlalchemy.url", settings.database_url)

if config.config_file_name is not None:
    fileConfig(config.config_file_name)

target_metadata = Base.metadata


def run_migrations_offline() -> None:
    """Tryb offline: generuje SQL do pliku, bez łączenia się z bazą."""
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
        compare_type=True,
        compare_server_default=True,
        render_as_batch=url.startswith("sqlite"),
    )
    with context.begin_transaction():
        context.run_migrations()


def do_run_migrations(connection: Connection) -> None:
    """Ciało migracji — wykonuje się w wątku roboczym pętli zdarzeń."""
    context.configure(
        connection=connection,
        target_metadata=target_metadata,
        compare_type=True,
        compare_server_default=True,
        # SQLite nie umie większości ALTER TABLE — Alembic odtwarza tabelę.
        # PostgreSQL to ignoruje, więc flaga jest bezpieczna dla obu baz.
        render_as_batch=connection.dialect.name == "sqlite",
    )
    with context.begin_transaction():
        context.run_migrations()


async def run_async_migrations() -> None:
    """Silnik jednorazowy (NullPool) — migracja to jeden, krótki proces."""
    connectable = async_engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)
    await connectable.dispose()


def run_migrations_online() -> None:
    asyncio.run(run_async_migrations())


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

Trzy rzeczy, o które potykają się wszyscy zaczynający:

1. **`from app.models import Base`** — importuj cały pakiet modeli, nie pojedyncze klasy. Jeśli w `app/models/__init__.py` nie ma `from app.models.loan import Loan`, to `Loan` nie trafi do `Base.metadata` i autogenerate „nie zauważy” tabeli `loans`. Objaw: migracja generuje wszystko oprócz jednej tabeli, a po wdrożeniu endpoint zwraca `no such table: loans`.
2. **`alembic.ini`** — pozostaw w nim `sqlalchemy.url = ` pusty. `env.py` i tak nadpisuje tę wartość, a trzymanie prawdziwego hasła w pliku wersjonowanym to klasyczny wyciek sekretów.
3. **Adnotacja `# przeglądaj ręcznie`** — autogenerate wykrywa dodanie i usunięcie tabeli, dodanie kolumny, indeks i klucz obcy, ale **zmianę nazwy kolumny** widzi jako „usuń starą, dodaj nową” (utrata danych!), a w SQLite nie wykryje zmiany typu. Zawsze otwórz wygenerowany plik i przeczytaj go linia po linii, zanim uruchomisz `upgrade`.

> 🧠 Dlaczego tak jest — Alembic 1.19.x potrafi wykryć nazwane ograniczenia `CHECK`, ale tylko jeśli mają nazwę — a mają ją dzięki `naming_convention` z sekcji 22.6. To kolejny powód, dla którego konwencja nazewnictwa jest ustawiana w pierwszym dniu projektu, a nie „kiedyś”.

### Dane słownikowe: inicjalizator, nie `lifespan`

Świat rzeczywisty ma dane, które nie są danymi użytkownika: listy statusów, opcje do list rozwijanych, parametry domyślne. Interfejs potrzebuje ich do wyświetlenia etykiet. Trzymamy je w tabeli słownikowej i wypełniamy **osobnym poleceniem**, wykonywanym jako krok wdrożenia — z tego samego powodu, dla którego migracje nie startują przy starcie aplikacji.

```python
# app/models/dictionary.py
from sqlalchemy import String, UniqueConstraint
from sqlalchemy.orm import Mapped, mapped_column

from app.models.base import Base, TimestampMixin


class DictionaryEntry(TimestampMixin, Base):
    """Wpis słownikowy: etykiety, opcje interfejsu, parametry nienależące do kodu."""

    __tablename__ = "dictionary_entries"
    __table_args__ = (
        UniqueConstraint("kind", "code", name="uq_dictionary_entries_kind_code"),
        {"comment": "Słowniki interfejsu; wypełniane skryptem seed."},
    )

    id: Mapped[int] = mapped_column(primary_key=True)
    kind: Mapped[str] = mapped_column(String(50), nullable=False, index=True)
    code: Mapped[str] = mapped_column(String(50), nullable=False)
    label: Mapped[str] = mapped_column(String(200), nullable=False)
    sort_order: Mapped[int] = mapped_column(default=0, nullable=False)
```

Seeding musi być **idempotentny** — uruchomiony dwa razy ma dać ten sam stan. Dwa podejścia, oba poprawne, ale o różnym koszcie:

```python
# app/cli.py
"""Komendy administracyjne. Uruchamiaj jako krok wdrożenia, nie w lifespan!

    python -m app.cli seed          # wypełnij słowniki (idempotentnie)
    python -m app.cli seed --demo   # dodatkowo dane demonstracyjne
"""

import asyncio

import typer
from sqlalchemy import select, text
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.config import get_settings
from app.core.db import build_engine, build_session_factory
from app.models.dictionary import DictionaryEntry

app = typer.Typer(help="Narzędzia administracyjne Library API")

DICTIONARY_SEED: list[dict[str, object]] = [
    {"kind": "loan_period_days", "code": "14", "label": "2 tygodnie", "sort_order": 1},
    {"kind": "loan_period_days", "code": "30", "label": "1 miesiąc", "sort_order": 2},
    {"kind": "loan_period_days", "code": "60", "label": "2 miesiące", "sort_order": 3},
    {"kind": "book_condition", "code": "new", "label": "Nowa", "sort_order": 1},
    {"kind": "book_condition", "code": "good", "label": "Dobra", "sort_order": 2},
    {"kind": "book_condition", "code": "worn", "label": "Zużyta", "sort_order": 3},
]


async def _seed_dictionary(session: AsyncSession) -> tuple[int, int]:
    """Wstawia brakujące wpisy. Zwraca (dodane, pominięte)."""
    added = skipped = 0
    for entry in DICTIONARY_SEED:
        exists = await session.scalar(
            select(DictionaryEntry.id).where(
                DictionaryEntry.kind == entry["kind"],
                DictionaryEntry.code == entry["code"],
            )
        )
        if exists is not None:
            skipped += 1
            continue
        session.add(DictionaryEntry(**entry))  # type: ignore[arg-type]
        added += 1
    await session.commit()
    return added, skipped


async def _run_seed(with_demo: bool) -> None:
    settings = get_settings()
    engine = build_engine(settings)
    factory = build_session_factory(engine)

    async with factory() as session:
        # Blokada doradcza PostgreSQL: jeśli wystartuje 5 replik naraz,
        # tylko jedna wykona seed, reszta poczeka i zobaczy gotowe dane.
        # Na SQLite ten SELECT jest nieszkodliwym, szybkim zapytaniem —
        # dlatego dekorujemy go warunkiem po dialekcie.
        if engine.dialect.name == "postgresql":
            await session.execute(text("SELECT pg_advisory_xact_lock(725122)"))
        added, skipped = await _seed_dictionary(session)
        typer.echo(f"Słowniki: dodano={added}, pominięto={skipped}")
        if with_demo:
            typer.echo("Dane demonstracyjne: pominięte w tym module (patrz moduł 23).")

    await engine.dispose()


@app.command()
def seed(
    demo: bool = typer.Option(False, "--demo", help="Dodaj dane demonstracyjne"),
) -> None:
    """Wypełnia dane słownikowe (bezpieczne do wielokrotnego uruchomienia)."""
    asyncio.run(_run_seed(demo))


if __name__ == "__main__":
    app()
```

> 💡 Analogia — Seed słowników to **kartki informacyjne wywieszone w bibliotece** („wypożyczenie na 2 tygodnie”, „książka zużyta”). Nie wpisuje ich żaden bibliotekarz w trakcie pracy; przywozi je raz osoba, która urządza placówkę. A jeśli przyjedzie dwa razy, nie naklei tych samych kartek dwukrotnie — bo ma listę kontrolną.

> 🆕 SQLAlchemy 2.1 — Dla seederów pisanych „hurtem” przydają się dwie rzeczy. Po pierwsze, w SQLite można teraz wyrazić **wiele klauzul `ON CONFLICT` w jednym `INSERT`**, co pozwala złożyć idempotentny zapis jednym poleceniem zamiast pętli `SELECT + INSERT` (na PostgreSQL taka składnia była dostępna wcześniej). Po drugie, 2.1 wydajniej generuje klucze cache dla takich masowych wstawień, więc seeder wstawiający tysiące wpisów nagrzewa się szybciej.

> ⚠️ Pułapka — Umieszczenie seeda w `lifespan` to zaproszenie do problemów na produkcji: przy pięciu replikach pięć procesów wykonuje te same `INSERT`-y w tej samej milisekundzie. Na PostgreSQL skończy się to `IntegrityError` i wywaleniem połowy replik przy starcie; na SQLite (plikowym) – blokadą zapisu i timeoutem. Reguła: **stan bazy zmienia się wyłącznie w krokach wdrożenia i migracjach.**

### Migracje przy wielu instancjach

Gdy aplikacja działa na N replikach, `alembic upgrade head` **nie może** być wykonywany przez każdą z nich. Standardowe rozwiązania, od najczęstszego:

| Wzorzec | Jak działa | Kiedy wybrać |
|---|---|---|
| Osobny job wdrożeniowy | pipeline uruchamia migrację; rollout startuje po jej sukcesie | Kubernetes `Job`, GitHub Actions, GitLab CI, AWS ECS „run task” |
| `initContainer` | kontener przed aplikacją wykonuje `upgrade head` i kończy się | Kubernetes; ryzyko: N kontenerów naraz, jeśli brak koordynacji |
| Blokada doradcza w migracji | migracja na starcie, ale pod `pg_advisory_lock` | gdy nie da się wywołać osobnego kroku |
| Migracje manualne | człowiek uruchamia przed wdrożeniem | audytowane systemy regulowane; wymaga dyscypliny i checklisty |

Alembic domyślnie **nie zakłada** żadnej blokady — dwa równoległe `upgrade head` mogą wejść sobie w drogę. Dlatego najbezpieczniejszy jest wzorzec pierwszy: jedno miejsce w pipeline, jedno uruchomienie, brak wyścigu.

```bash
# Typowy krok wdrożenia w pipeline (przykład ogólny)
# 1. Migracje schematu
APP_DATABASE_URL="$PROD_DB_URL" alembic upgrade head

# 2. Dane słownikowe
APP_DATABASE_URL="$PROD_DB_URL" python -m app.cli seed

# 3. Dopiero teraz nowa wersja aplikacji
kubectl set image deployment/library-api api=registry/library-api:1.4.0
```

> 🧪 Ćwiczenie — Dopisz do `app/cli.py` komendę `check`, która sprawdza, czy wersja schematu w bazie zgadza się z `head` Alembica, i kończy się kodem wyjścia `1`, jeśli nie. Podłącz ją jako pierwszy krok skryptu uruchomieniowego aplikacji na produkcji. Podpowiedź: `alembic script` i `alembic.runtime.migration.MigrationContext.get_current_revision()`.

---

## 22.14 Wydajność API: N+1, limity, cache, monitoring

W aplikacji webowej wydajność sprowadza się do trzech liczb: **ile rund do bazy**, **ile bajtów przez sieć** i **jak długo trzymasz transakcję**. Reszta to szczegóły implementacji.

### Przypadek 1: lista wypożyczeń z tytułami książek

Najgorszy i najczęstszy błąd to pętla, w której dociągamy dane po jednym rekordzie:

```python
# ⛔ ANTYWZORZEC — 100 wypożyczeń = 201 rund do bazy
@router.get("/loans/naive", response_model=list[LoanRead])
async def list_loans_naive(session: SessionDep, limit: int = Query(100, le=100)) -> list[LoanRead]:
    loans = await LoanRepository(session).list_active(limit=limit, offset=0)
    result: list[LoanRead] = []
    for loan in loans:
        book = await session.get(Book, loan.book_id)  # ← osobne zapytanie na każdy wiersz
        result.append(LoanRead.model_validate({**loan.__dict__, "book_title": book.title}))
    return result
```

Poprawka to jedno zapytanie z jawnym ładowaniem dwóch poziomów relacji:

```python
# app/repositories/loans.py (metoda dodana do klasy LoanRepository)
from sqlalchemy.orm import joinedload

    async def list_active_with_books(self, *, limit: int, offset: int) -> list[Loan]:
        """Jedno zapytanie: wypożyczenie + książka + autor książki."""
        stmt = (
            select(Loan)
            .where(Loan.returned_at.is_(None))
            .options(joinedload(Loan.book).joinedload(Book.author))
            .order_by(Loan.due_at, Loan.id)
            .limit(limit)
            .offset(offset)
        )
        return list((await self._session.scalars(stmt)).all())
```

> 🔬 Pod maską — `joinedload` na łańcuchu dwóch relacji wiele-do-jednego generuje jedno zapytanie z dwoma `LEFT OUTER JOIN`. Liczba zwróconych wierszy jest identyczna jak liczba encji `Loan`, więc `.unique()` nie jest potrzebne:

```sql
SELECT loans.id, loans.book_id, loans.member_name, loans.due_at, loans.returned_at,
       loans.created_at, loans.updated_at,
       books_1.id AS id_1, books_1.title, books_1.isbn, books_1.author_id, ...,
       authors_1.id AS id_2, authors_1.name, authors_1.bio, ...
FROM loans
LEFT OUTER JOIN books AS books_1 ON books_1.id = loans.book_id
LEFT OUTER JOIN authors AS authors_1 ON authors_1.id = books_1.author_id
WHERE loans.returned_at IS NULL
ORDER BY loans.due_at, loans.id
LIMIT 100 OFFSET 0;
```

| Endpoint (100 rekordów) | Liczba zapytań | Czas lokalny (SQLite, przybliżenie) |
|---|---|---|
| naiwna pętla z `session.get()` na wiersz | 201 | ~180 ms |
| `joinedload(Loan.book).joinedload(Book.author)` | 1 | ~6 ms |
| `selectinload(Loan.book)` z osobnym zapytaniem | 2 | ~8 ms |

Liczby zmierzone na kursowym zbiorze danych (100 wypożyczeń, SQLite w pamięci). Względna różnica utrzymuje się na PostgreSQL: to nie kwestia dialektu, a liczby rund w protokole.

> 🧠 Dlaczego tak jest — Każde zapytanie to nie tylko praca bazy, ale i pełna podróż w sieci: wysłanie SQL-a, sparsowanie go, zaplanowanie, pobranie z cache’u lub dysku, serializacja wyników, transfer, deserializacja w Pythonie. Przy 200 rundach te stałe koszty zjadają wszystko, nawet jeśli baza odpowiada w pół milisekundy.

### Licznik zapytań: mierz, nie zgaduj

```python
# app/core/observability.py
import contextvars
import logging
from collections.abc import Awaitable, Callable

from fastapi import FastAPI, Request, Response
from sqlalchemy import event
from sqlalchemy.ext.asyncio import AsyncEngine

_logger = logging.getLogger("library.sql")

# ContextVar trzyma LISTĘ, a nie int. Dlaczego? Bo mutacja obiektu w liście
# jest widoczna w kopii kontekstu utworzonej dla zadania obsługującego żądanie -
# a przypisanie nowej wartości już nie. Pętla zdarzeń i middleware FastAPI
# wykonują się w różnych kopiach kontekstu.
_counter: contextvars.ContextVar[list[int] | None] = contextvars.ContextVar(
    "sql_counter", default=None
)


def start_counting() -> None:
    """Rozpoczyna liczenie od zera w bieżącym kontekście (żądanym/teście)."""
    _counter.set([0])


def queries_so_far() -> int:
    counter = _counter.get()
    return counter[0] if counter is not None else 0


def register_query_counter(engine: AsyncEngine) -> None:
    """Liczy zapytania SQL. Zdarzenia podłączamy do engine.sync_engine!

    AsyncEngine to tylko opakowanie - sam nie emituje zdarzeń SQLAlchemy.
    """

    @event.listens_for(engine.sync_engine, "before_cursor_execute")
    def _before(conn, cursor, statement, parameters, context, executemany) -> None:
        counter = _counter.get()
        if counter is not None:
            counter[0] += 1
        if _logger.isEnabledFor(logging.DEBUG):
            _logger.debug("SQL[%s]: %s", "many" if executemany else "one", statement)


def register_query_count_middleware(app: FastAPI, *, expose_header: bool) -> None:
    """Dokłada nagłówek X-Query-Count - tylko w trybie debug."""

    @app.middleware("http")
    async def _count_queries(
        request: Request, call_next: Callable[[Request], Awaitable[Response]]
    ) -> Response:
        start_counting()
        response = await call_next(request)
        if expose_header:
            response.headers["X-Query-Count"] = str(queries_so_far())
        return response
```

W `create_app()` rejestrujemy licznik po zbudowaniu silnika — czyli w `lifespan`:

```python
    app.state.engine = engine
    app.state.session_factory = build_session_factory(engine)
    register_query_counter(engine)
    logger.info(...)
```

Od tego momentu masz twardą liczbę. Uruchom aplikację z `APP_SQL_ECHO=true` i `--debug`, zrób żądanie `GET /api/v1/loans` i sprawdź nagłówek `X-Query-Count`. Jeśli zobaczysz więcej niż trzy, wiesz dokładnie, gdzie szukać.

### Limity, których nie da się ominąć

Zabezpieczenie przed „chcę całą tabelę” musi działać po stronie serwera, niezależnie od tego, co przyśle klient:

```python
# app/schemas/common.py (fragment)
PAGE_SIZE_MAX = 100


def clamp_limit(limit: int, *, default: int = 20, maximum: int = PAGE_SIZE_MAX) -> int:
    if limit < 1:
        return default
    return min(limit, maximum)
```

W endpointcie deklarujemy `Query(ge=1, le=PAGE_SIZE_MAX)`, a w serwisie dodatkowo przepuszczamy wartość przez `clamp_limit` — Pydantic zwróci `422`, a logika biznesowa i tak nigdy nie dostanie wartości spoza zakresu. Podwójne zabezpieczenie jest tanie.

> ⚠️ Pułapka — `OFFSET` na dużych zbiorach jest kosztowny: `OFFSET 100000 LIMIT 20` każe bazie przejść i odrzucić 100 tysięcy wierszy. Dla list, które mają być przeglądane „głęboko”, stosuje się paginację keyset (`WHERE id > :last_id ORDER BY id LIMIT n`), o której mówiliśmy w [module 20](20_repository.md). W API trzeba wtedy wystawić kursory (`next_cursor`) zamiast `offset`. Ćwiczenie 1 prowadzi właśnie przez tę zmianę.

### Cache danych słownikowych

Dane słownikowe są małe, rzadko się zmieniają i są potrzebne przy każdym renderowaniu listy opcji. Idealny kandydat na cache w pamięci procesu:

```python
# app/repositories/dictionaries.py
from collections.abc import Sequence
from functools import lru_cache

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncEngine, async_sessionmaker

from app.models.dictionary import DictionaryEntry


@lru_cache(maxsize=64)
def _read_sync_cache_placeholder() -> None:  # placeholder pokazujący ograniczenie
    raise RuntimeError("Funkcja asynchroniczna nie może być cache'owana przez lru_cache.")


class DictionaryRepository:
    """Odczyt słowników z krótkim cache'em procesowym."""

    def __init__(self, session_factory: async_sessionmaker) -> None:
        self._factory = session_factory
        self._cache: dict[str, Sequence[DictionaryEntry]] = {}

    async def entries(self, kind: str) -> Sequence[DictionaryEntry]:
        cached = self._cache.get(kind)
        if cached is not None:
            return cached
        async with self._factory() as session:
            stmt = (
                select(DictionaryEntry)
                .where(DictionaryEntry.kind == kind)
                .order_by(DictionaryEntry.sort_order, DictionaryEntry.code)
            )
            rows = tuple((await session.scalars(stmt)).all())
        self._cache[kind] = rows
        return rows

    def invalidate(self, kind: str | None = None) -> None:
        """Wywołaj po zmianie słowników (np. z komendy administracyjnej)."""
        if kind is None:
            self._cache.clear()
        else:
            self._cache.pop(kind, None)
```

> ⚠️ Pułapka — `@lru_cache` **nie zadziała** na metodzie asynchronicznej ani na funkcji zwracającej encje ORM: encje poza sesją stają się oderwane, a `lru_cache` na korutynie zapamiętuje obiekt korutyny, co jest błędem czasu wykonania. Dlatego cache trzymamy jako zwykły słownik w obiekcie długożyjącym (np. w `app.state`) i unieważniamy go jawnie po zmianie danych. Jeśli słowniki zmieniają się w czasie działania systemu, użyj `cachetools.TTLCache` zamiast ręcznego słownika.

### Zasady wydajności API zebrane w tabeli

| Zasada | Co daje | Kiedy łamana świadomie |
|---|---|---|
| Lista zawsze z `limit` i walidacją górnej granicy | brak „pobierz wszystko” | nigdy |
| Relacje w liście jawnie ładowane (`joinedload` / `selectinload`) | brak N+1 | gdy Dominika zwracasz DTO z projekcją kolumn |
| Licznik zapytań w CI | regresja wydajności wyłapana w pull requeście | — |
| Transakcja kończona przed serializacją odpowiedzi | krótkie blokady | gdy potrzebujesz `RETURNING` po commicie |
| Agregacje liczone w bazie | mniej danych przez sieć | gdy logika agregacji jest w Pythonie z powodów biznesowych |
| Słowniki w cache procesowym | mniej zapytań na gorącej ścieżce | gdy słowniki zmieniają się często |

---

## 22.15 Bezpieczeństwo i higiena

Pięć nawyków, które odróżniają kod produkcyjny od prototypu.

**1. Nigdy nie składaj SQL-a ze stringów.** API 2.0 domyślnie wiąże parametry, ale `text()` z f-stringiem omija to zabezpieczenie:

```python
# ⛔ Nigdy tak:
stmt = text(f"SELECT * FROM books WHERE title = '{title}'")

# ✅ Tak — parametr wiązany, sterownik wysyła wartość osobno od zapytania:
stmt = text("SELECT * FROM books WHERE title = :title")
rows = (await session.execute(stmt, {"title": title})).all()

# ✅ Jeszcze lepiej — Core/ORM, który sam wie, co związać:
rows = (await session.scalars(select(Book).where(Book.title == title))).all()
```

**2. Loguj identyfikatory, nie treść.** `member_name` to dane osobowe. Log `INFO` typu `Wypożyczenie utworzone: loan_id=41 book_id=7` jest diagnostycznie wystarczający. W `except` używaj `logger.exception`, ale nigdy nie loguj całego ciała żądania ani parametrów zapytań na produkcji.

**3. `echo=True` to narzędzie developerskie.** Na produkcji `APP_SQL_ECHO=false`. Logowanie każdego zapytania plus parametry potrafi spowolnić aplikację o kilkadziesiąt procent i zapchać dysk w kilka dni.

**4. Konto bazy z minimalnymi uprawnieniami.**

```sql
-- PostgreSQL: konto aplikacji nie potrzebuje prawa do DROP czy CREATE.
CREATE ROLE library_app LOGIN PASSWORD 'zmien-mnie';
GRANT CONNECT ON DATABASE library TO library_app;
GRANT USAGE ON SCHEMA public TO library_app;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO library_app;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO library_app;

-- Aplikacja nie zmienia schematu - migracje uruchamia inne konto:
-- library_migrator (właściciel tabel, prawa CREATE/ALTER/DROP).
```

Rozdzielenie konta aplikacji i konta migracji to jeden z niewielu zabezpieczeń, które nie kosztuje ani linii kodu: jeśli konto aplikacji nie ma prawa `DROP`, żadna pomyłka w kodzie nie skasuje tabeli.

**5. Nie wyciekaj wnętrzności w odpowiedziach.** Handler `Exception` z sekcji 22.8 zwraca generyczne `internal_error`, a szczegóły trafiają do logu. Klient nie powinien nigdy zobaczyć stack trace’u, nazwy tabeli ani ciągu połączenia.

Dodatkowo, na poziomie API:

- CORS wyłącznie z listy dozwolonych źródeł z konfiguracji; `allow_origins=["*"]` razem z `allow_credentials=True` jest zabronione przez przeglądarki i słusznie.
- `docs_url` wyłączony na produkcji (`if settings.debug`), tak jak w `create_app()`.
- Twarde limity `limit` (ochrona przed prostym DoS-em przez „pobierz milion wierszy”).
- `.env` w `.gitignore`, `.env.example` bez prawdziwych sekretów.

```gitignore
# .gitignore (fragment)
.env
*.db
__pycache__/
.pytest_cache/
.ruff_cache/
```

> 🧠 Dlaczego tak jest — Bezpieczeństwo w aplikacji bazodanowej rozgrywa się na granicach: między przeglądarką a API (CORS, limity), między API a bazą (uprawnienia, parametry wiązane) i między kodem a logami (dane osobowe). Każda z tych granic jest tania do pilnowania na starcie i koszmarnie droga do naprawienia po incydencie.

---

## 22.16 Alternatywy: Flask, Django, CLI

Wzorzec „silnik raz, sesja na jednostkę pracy” jest uniwersalny. Zmienia się tylko mechanizm, którym framework przekazuje sesję do kodu.

### Flask + `scoped_session`

Flask obsługuje żądania synchronicznie, w jednym wątku, i ma wbudowany kontekst aplikacji/żądania. Sesję przypina się do kontekstu aplikacji:

```python
# Wariant FLASK (sync) — pokazany dla kontrastu. W projekcie kursowym używamy FastAPI.
from collections.abc import Iterator

from flask import Flask, g
from sqlalchemy import create_engine
from sqlalchemy.orm import Session, scoped_session, sessionmaker

engine = create_engine("postgresql+psycopg://user:pass@localhost/library", pool_pre_ping=True)
SessionLocal = scoped_session(sessionmaker(bind=engine, expire_on_commit=False))

app = Flask(__name__)


def get_session() -> Session:
    """Jedna sesja na kontekst aplikacji (Flask trzyma ją w `g`)."""
    if "db" not in g:
        g.db = SessionLocal()
    return g.db


@app.teardown_appcontext
def close_session(exc: BaseException | None) -> None:
    session = g.pop("db", None)
    if session is None:
        return
    if exc is None:
        session.commit()  # wariant commit-on-success - świadomie omówiony w 22.5
    else:
        session.rollback()
    session.close()
    SessionLocal.remove()  # czyści rejestr scoped_session dla tego wątku
```

Różnice wobec FastAPI:

| Aspekt | FastAPI | Flask |
|---|---|---|
| Przekazanie sesji | `Depends(get_session)` — jawnie w sygnaturze | globalny `g` / `scoped_session` — niejawnie |
| Zakres | `yield` w zależności, także z async | `teardown_appcontext` |
| Wątki | jeden wątek na żądanie z threadpoola (dla `def`) | jeden wątek na żądanie (Werkzeug) |
| Testowanie | nadpisanie zależności / `app.state` | `app.test_client()` + `g` |

`scoped_session` działa przez wątek (thread-local). W aplikacji async jest reliktem — pętla zdarzeń nie ma „wątków żądania”, a zadania przeskakują między kontekstami. Dlatego w FastAPI przekazujemy sesję wprost.

### Django

Django ma ORM zbudowany wokół innej filozofii: model dziedziczy po `models.Model`, a zapytania wychodzą z klasy (`Book.objects.filter(...)`). To wzorzec **Active Record**, a nie Data Mapper, o którym mówiliśmy w [module 19](19_warstwy_i_data_mapper.md). Konsekwencje praktyczne:

- nie ma obiektu `Session` ani `UnitOfWork` dostarczanego przez ORM — granicą jest `transaction.atomic()`;
- migracje są częścią frameworka (`makemigrations`), a nie osobnym narzędziem;
- leniwe zapytania (`QuerySet` jest lazy) zmieniają rachunek N+1 na jeszcze mniej przewidywalny;
- w Django async jest warstwą dodaną (`sync_to_async`, `aget`), a nie trybem podstawowym.

Nie ma tu lepszej i gorszej strony — są różne kompromisy. Jeżeli potrzebowalibyśmy elastyczności zapytań i świadomego sterowania transakcjami, SQLAlchemy daje narzędzia, których Django ORM nie ma. Jeżeli budujemy klasyczną aplikację CRUD z administratorem, Django jest szybszą drogą.

### CLI: jedna sesja na jedno polecenie

Aplikacja CLI to trzeci kontekst, obok webu i workera. Zasada ta sama: sesja jest krótka, transakcja dopięta do polecenia.

```python
# Wariant CLI (typer + async) — jedna sesja na uruchomienie polecenia
import asyncio

import typer
from sqlalchemy import select

from app.core.config import get_settings
from app.core.db import build_engine, build_session_factory
from app.models import Book

app = typer.Typer()


@app.command()
def list_books() -> None:
    """Wypisuje dostępne książki (jedna transakcja, jedna sesja)."""

    async def _run() -> None:
        engine = build_engine(get_settings())
        factory = build_session_factory(engine)
        try:
            async with factory() as session:
                stmt = select(Book).where(Book.copies_available > 0).order_by(Book.title)
                for book in await session.scalars(stmt):
                    typer.echo(f"{book.id:>4}  {book.title}")
        finally:
            await engine.dispose()

    asyncio.run(_run())


if __name__ == "__main__":
    app()
```

> 💡 Analogia — Framework to **rodzaj lokalu**: bar szybkiej obsługi (Flask, CLI), samoobsługowy bufet (FastAPI z wstrzykiwaniem) albo restauracja z gotowym menu (Django). Ten sam personel, te same kuchnie — ale inaczej się wydaje polecenia i inaczej rozlicza rachunek.

---

## 22.17 Kompletny projekt: jeden plik, który działa

Zanim złożysz paczkę `app/`, warto zobaczyć całe okablowanie w jednym pliku. Poniższy kod to **działające, samodzielne API** — te same warstwy, tylko bez podziału na moduły. Zapisz go jako `examples/api_minimal.py`.

```python
# examples/api_minimal.py
"""Minimalne, w pełni uruchamialne API biblioteki (SQLite + async SQLAlchemy).

Uruchomienie:
    pip install "fastapi[standard]" "sqlalchemy[asyncio]" aiosqlite
    uvicorn examples.api_minimal:app --reload
    curl http://127.0.0.1:8000/api/v1/books

Uwaga: katalog `examples/` traktujemy jako pakiet (namespace package działa,
ale `touch examples/__init__.py` usuwa wszelkie niespodzianki przy imporcie).
"""

from collections.abc import AsyncIterator
from datetime import UTC, datetime, timedelta
from typing import Annotated

from fastapi import APIRouter, Depends, FastAPI, Query, status
from pydantic import BaseModel, ConfigDict, Field
from sqlalchemy import ForeignKey, String, func, select
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from sqlalchemy.orm import (
    DeclarativeBase,
    Mapped,
    Select,
    joinedload,
    mapped_column,
    relationship,
)

DATABASE_URL = "sqlite+aiosqlite:///./minimal.db"


# --------------------------------------------------------------------------- modele
class Base(DeclarativeBase):
    pass


class Author(Base):
    __tablename__ = "authors"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(120), nullable=False)

    books: Mapped[list["Book"]] = relationship(back_populates="author", lazy="raise")


class Book(Base):
    __tablename__ = "books"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200), nullable=False, index=True)
    isbn: Mapped[str] = mapped_column(String(20), nullable=False, unique=True)
    copies_available: Mapped[int] = mapped_column(default=1, nullable=False)

    author_id: Mapped[int] = mapped_column(ForeignKey("authors.id"), index=True)
    author: Mapped["Author"] = relationship(back_populates="books", lazy="raise")


# ------------------------------------------------------------------------- schematy
class AuthorRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    name: str


class BookCreate(BaseModel):
    title: str = Field(min_length=1, max_length=200)
    isbn: str = Field(min_length=10, max_length=20)
    author_id: int = Field(ge=1)
    copies_available: int = Field(default=1, ge=0, le=1000)


class BookRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    title: str
    isbn: str
    copies_available: int
    author: AuthorRead


class Page(BaseModel):
    items: list[BookRead]
    total: int
    limit: int
    offset: int


# ---------------------------------------------------------------------- repozytorium
class BookRepository:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def get(self, book_id: int) -> Book | None:
        stmt: Select[tuple[Book]] = (
            select(Book).options(joinedload(Book.author)).where(Book.id == book_id)
        )
        return await self._session.scalar(stmt)

    async def list_page(
        self, *, limit: int, offset: int, q: str | None
    ) -> tuple[list[Book], int]:
        conditions = []
        if q:
            conditions.append(Book.title.ilike(f"%{q}%"))
        total = await self._session.scalar(
            select(func.count()).select_from(Book).where(*conditions)
        )
        stmt = (
            select(Book)
            .where(*conditions)
            .options(joinedload(Book.author))
            .order_by(Book.title, Book.id)
            .limit(limit)
            .offset(offset)
        )
        rows = list((await self._session.scalars(stmt)).all())
        return rows, int(total or 0)

    async def exists_isbn(self, isbn: str) -> bool:
        stmt = select(select(Book.id).where(Book.isbn == isbn).exists())
        return bool(await self._session.scalar(stmt))

    async def add(self, book: Book) -> Book:
        self._session.add(book)
        await self._session.flush()
        return book


# -------------------------------------------------------------------------- serwis
class BookService:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session
        self._repo = BookRepository(session)

    async def get(self, book_id: int) -> Book:
        book = await self._repo.get(book_id)
        if book is None:
            raise LookupError(f"Nie znaleziono książki id={book_id}")
        return book

    async def list_books(
        self, *, limit: int, offset: int, q: str | None
    ) -> tuple[list[Book], int]:
        return await self._repo.list_page(limit=limit, offset=offset, q=q)

    async def create(self, data: BookCreate) -> Book:
        if await self._repo.exists_isbn(data.isbn):
            raise ValueError(f"ISBN {data.isbn} już istnieje")
        author = await self._session.get(Author, data.author_id)
        if author is None:
            raise LookupError(f"Nie znaleziono autora id={data.author_id}")
        book = Book(
            title=data.title,
            isbn=data.isbn,
            copies_available=data.copies_available,
            author=author,
        )
        await self._repo.add(book)
        await self._session.commit()  # granica transakcji przypadku użycia
        return book


# -------------------------------------------------------------------------- router
router = APIRouter(prefix="/api/v1")

engine = create_async_engine(DATABASE_URL, echo=False)
SessionFactory = async_sessionmaker(engine, expire_on_commit=False, autoflush=True)


async def get_session() -> AsyncIterator[AsyncSession]:
    async with SessionFactory() as session:
        try:
            yield session
        except Exception:
            await session.rollback()
            raise


SessionDep = Annotated[AsyncSession, Depends(get_session)]


@router.get("/books", response_model=Page, tags=["books"])
async def list_books(
    session: SessionDep,
    q: Annotated[str | None, Query(max_length=100)] = None,
    limit: Annotated[int, Query(ge=1, le=100)] = 20,
    offset: Annotated[int, Query(ge=0)] = 0,
) -> Page:
    books, total = await BookService(session).list_books(limit=limit, offset=offset, q=q)
    return Page(
        items=[BookRead.model_validate(book) for book in books],
        total=total,
        limit=limit,
        offset=offset,
    )


@router.get("/books/{book_id}", response_model=BookRead, tags=["books"])
async def get_book(book_id: int, session: SessionDep) -> BookRead:
    try:
        book = await BookService(session).get(book_id)
    except LookupError as exc:
        from fastapi import HTTPException

        raise HTTPException(status_code=404, detail=str(exc)) from exc
    return BookRead.model_validate(book)


@router.post(
    "/books",
    response_model=BookRead,
    status_code=status.HTTP_201_CREATED,
    tags=["books"],
)
async def create_book(payload: BookCreate, session: SessionDep) -> BookRead:
    from fastapi import HTTPException

    try:
        book = await BookService(session).create(payload)
    except ValueError as exc:
        raise HTTPException(status_code=409, detail=str(exc)) from exc
    except LookupError as exc:
        raise HTTPException(status_code=404, detail=str(exc)) from exc
    return BookRead.model_validate(book)


# --------------------------------------------------------------------------- aplikacja
async def lifespan(_: FastAPI) -> AsyncIterator[None]:
    async with engine.begin() as conn:
        # W przykładzie edukacyjnym; w projekcie docelowym zastępuje to Alembic.
        await conn.run_sync(Base.metadata.create_all)
    yield
    await engine.dispose()


app = FastAPI(title="Minimal Library API", lifespan=lifespan)
app.include_router(router)
```

Dla wersji pakietowej brakuje nam jeszcze schematów wypożyczenia, do których odwołaliśmy się w sekcji 22.11:

```python
# app/schemas/loan.py
from datetime import datetime

from pydantic import BaseModel, Field

from app.schemas.common import ORMModel


class LoanCreate(BaseModel):
    book_id: int = Field(ge=1)
    member_name: str = Field(min_length=2, max_length=120)
    days: int | None = Field(default=None, ge=1, le=365)


class LoanRead(ORMModel):
    id: int
    book_id: int
    member_name: str
    due_at: datetime
    returned_at: datetime | None
    created_at: datetime


class LoanWithBook(ORMModel):
    """Odczyt rozszerzony — wymaga joinedload(Loan.book).joinedload(Book.author)."""

    id: int
    due_at: datetime
    member_name: str
    book_title: str
    author_name: str
```

`LoanWithBook` nie da się zbudować z samej encji (nie ma pól `book_title` ani `author_name`). To celowe: na potrzeby raportów używamy DTO — patrz następny krok:

```python
# app/repositories/loans.py (metoda projekcji do raportu)
from sqlalchemy import select

    async def list_active_report(self, *, limit: int, offset: int) -> list[dict[str, object]]:
        """Projekcja: zwraca słowniki gotowe do zbudowania raportu."""
        from app.models import Author, Book, Loan

        stmt = (
            select(
                Loan.id,
                Loan.due_at,
                Loan.member_name,
                Book.title.label("book_title"),
                Author.name.label("author_name"),
            )
            .join(Book, Loan.book_id == Book.id)
            .join(Author, Book.author_id == Author.id)
            .where(Loan.returned_at.is_(None))
            .order_by(Loan.due_at, Loan.id)
            .limit(limit)
            .offset(offset)
        )
        return [dict(row) for row in (await self._session.execute(stmt)).mappings().all()]
```

```python
# app/api/v1/loans.py (endpoint raportowy — dopełnia listę siedmiu)
@router.get("/reports/active-loans", response_model=list[LoanWithBook])
async def active_loans_report(
    uow: UowDep,
    limit: Annotated[int, Query(ge=1, le=200)] = 50,
    offset: Annotated[int, Query(ge=0)] = 0,
) -> list[LoanWithBook]:
    rows = await uow.loans.list_active_report(limit=limit, offset=offset)
    return [LoanWithBook.model_validate(row) for row in rows]
```

> 🧠 Dlaczego `dict(row)` — `RowMapping` z SQLAlchemy zachowuje się niemal jak słownik, ale nie jest słownikiem. `model_validate(row)` na `LoanWithBook` z `from_attributes=True` też by zadziałało (Pydantic czyta pola przez `getattr`), ale jawne `dict(...)` jest bardziej przewidywalne przy debugowaniu i nie zależy od wewnętrznych metod `Row`.

### Struktura pakietu: co gdzie ostatecznie trafia

| Plik | Zawartość | Sekcja, w której powstał |
|---|---|---|
| `app/core/config.py` | `Settings`, `get_settings()` | 22.3 |
| `app/core/db.py` | `build_engine()`, `build_session_factory()` | 22.4 |
| `app/core/errors.py` | wyjątki domenowe, `register_exception_handlers()` | 22.8 |
| `app/core/observability.py` | licznik zapytań, middleware | 22.14 |
| `app/models/base.py` | `Base`, `TimestampMixin`, `NAMING_CONVENTION` | 22.6 |
| `app/models/{author,book,loan}.py` | encje | 22.6 |
| `app/models/dictionary.py` | tabela słownikowa | 22.13 |
| `app/schemas/{common,author,book,loan}.py` | kontrakty HTTP | 22.6, 22.17 |
| `app/repositories/{books,authors,loans,dictionaries}.py` | zapytania | 22.7, 22.14 |
| `app/services/{uow,books,loans}.py` | przypadki użycia, granice transakcji | 22.8, 22.11 |
| `app/api/deps.py` | `get_session()`, `get_uow()` | 22.5 |
| `app/api/v1/{books,authors,loans}.py` | routery, 7 endpointów | 22.9, 22.11, 22.17 |
| `app/main.py` | `create_app()`, `lifespan` | 22.4 |
| `app/cli.py` | seed słowników | 22.13 |
| `alembic/env.py` | migracje async | 22.13 |
| `tests/conftest.py`, `tests/test_api_books.py` | testy | 22.18 |

Siedem endpointów naszego API:

| Metoda i ścieżka | Kod | Rola |
|---|---|---|
| `GET /api/v1/books` | 200 | lista z paginacją i filtrami |
| `GET /api/v1/books/{id}` | 200 / 404 | szczegóły |
| `POST /api/v1/books` | 201 / 409 | utworzenie |
| `PATCH /api/v1/books/{id}` | 200 / 404 / 409 | częściowa aktualizacja |
| `DELETE /api/v1/books/{id}` | 204 / 404 | usunięcie |
| `GET /api/v1/authors/{id}/books` | 200 | książki autora |
| `POST /api/v1/loans` | 201 / 404 / 409 | wypożyczenie z blokadą wiersza |

---

## 22.18 Testy integracyjne API

Testujemy **cały przepływ**: od żądania HTTP, przez router, serwis i repozytorium, do bazy i z powrotem. Nie mockujemy `Session` — to byłby test mocka, nie testu ([moduł 18](18_testowanie.md)).

```python
# tests/conftest.py
from collections.abc import AsyncIterator

import pytest
from httpx import ASGITransport, AsyncClient
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from sqlalchemy.pool import StaticPool

from app.core.config import get_settings
from app.main import create_app
from app.models import Base


@pytest.fixture
async def app_instance():
    """Aplikacja z bazą w pamięci.

    Uwaga: httpx.ASGITransport NIE uruchamia zdarzeń lifespan, więc silnik
    i fabrykę sesji ustawiamy na app.state ręcznie. To także wygodne:
    każdy test dostaje czystą bazę.
    """
    get_settings.cache_clear()

    engine = create_async_engine(
        "sqlite+aiosqlite:///:memory:",
        # Bez tego aiosqlite odmówi współdzielenia połączenia między wątkami.
        connect_args={"check_same_thread": False},
        # StaticPool = jedno połączenie na cały silnik. Bez tego każde nowe
        # połączenie do ":memory:" dostaje WŁASNĄ, PUSTĄ bazę i tabele znikają.
        poolclass=StaticPool,
    )
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    application = create_app()
    application.state.engine = engine
    application.state.session_factory = async_sessionmaker(
        engine, class_=AsyncSession, expire_on_commit=False
    )
    yield application
    await engine.dispose()


@pytest.fixture
async def client(app_instance) -> AsyncIterator[AsyncClient]:
    transport = ASGITransport(app=app_instance)
    async with AsyncClient(transport=transport, base_url="http://test") as http_client:
        yield http_client


@pytest.fixture
async def seeded(client: AsyncClient) -> dict[str, int]:
    """Autor + dwie książki; zwraca identyfikatory do dalszych żądań."""
    # Autor nie ma endpointu POST w tym module, więc tworzymy go przez sesję testową.
    # (W projekcie końcowym dodasz POST /authors - patrz ćwiczenie 2.)
    from sqlalchemy import select

    from app.models import Author, Book

    factory = client._transport.app.state.session_factory  # type: ignore[attr-defined]
    async with factory() as session:
        author = Author(name="Stanisław Lem")
        session.add(author)
        await session.flush()
        session.add_all(
            [
                Book(title="Solaris", isbn="9780156027602", author_id=author.id,
                     copies_total=2, copies_available=2),
                Book(title="Bajki robotów", isbn="9788308062290", author_id=author.id,
                     copies_total=1, copies_available=0),
            ]
        )
        await session.commit()
        author_id = author.id

    book_id = (await client.get("/api/v1/books", params={"q": "Solaris"})).json()["items"][0]["id"]
    return {"author_id": author_id, "book_id": book_id}
```

> ⚠️ Pułapka — Dostęp do sesji w fixture przez `client._transport.app` to skrót, który zależy od wewnętrznych atrybutów `httpx`. W prawdziwym projekcie udostępniaj `app_instance` jako osobną fixture zależną od `client` (albo trzymaj sesję testową w fixture `db_session`), zamiast sięgać do prywatnego pola transportu. Pokazany wariant jest czytelny w materiale kursowym, ale w kodzie produkcyjnym warto go zamienić na jawną zależność między fixtures.

Pięć testów:

```python
# tests/test_api_books.py
import pytest
from httpx import AsyncClient

pytestmark = pytest.mark.asyncio


async def test_create_book_returns_201_with_author(client: AsyncClient, seeded: dict) -> None:
    payload = {
        "title": "Niezwyciężony",
        "isbn": "9788375780633",
        "author_id": seeded["author_id"],
        "copies_available": 3,
    }
    response = await client.post("/api/v1/books", json=payload)

    assert response.status_code == 201
    body = response.json()
    assert body["title"] == "Niezwyciężony"
    assert body["author"]["name"] == "Stanisław Lem"  # relacja załadowana eager


async def test_list_books_paginates_and_filters(client: AsyncClient, seeded: dict) -> None:
    response = await client.get("/api/v1/books", params={"limit": 1, "offset": 0})

    assert response.status_code == 200
    body = response.json()
    assert body["total"] == 2
    assert len(body["items"]) == 1
    assert body["limit"] == 1


async def test_get_missing_book_returns_404(client: AsyncClient) -> None:
    response = await client.get("/api/v1/books/999999")

    assert response.status_code == 404
    assert "Nie znaleziono" in response.json()["detail"]


async def test_duplicate_isbn_returns_409(client: AsyncClient, seeded: dict) -> None:
    payload = {
        "title": "Solaris (wydanie drugie)",
        "isbn": "9780156027602",  # ten sam ISBN co istniejąca książka
        "author_id": seeded["author_id"],
        "copies_available": 1,
    }
    response = await client.post("/api/v1/books", json=payload)

    assert response.status_code == 409
    assert "już istnieje" in response.json()["detail"]


async def test_patch_only_touches_supplied_fields(client: AsyncClient, seeded: dict) -> None:
    book_id = seeded["book_id"]
    response = await client.patch(f"/api/v1/books/{book_id}", json={"title": "Solaris II"})

    assert response.status_code == 200
    body = response.json()
    assert body["title"] == "Solaris II"
    assert body["isbn"] == "9780156027602"  # nieprzesłane pole zostało nietknięte
```

A szósty test — ten, który mierzy liczbę zapytań — pokazuje, jak wyłapać N+1 w CI:

```python
# tests/test_api_performance.py
import pytest
from httpx import AsyncClient

from app.core.observability import queries_so_far, start_counting

pytestmark = pytest.mark.asyncio


async def test_book_list_does_not_have_n_plus_one(client: AsyncClient, seeded: dict) -> None:
    start_counting()
    response = await client.get("/api/v1/books", params={"limit": 50})

    assert response.status_code == 200
    assert len(response.json()["items"]) == 2
    # 1 zapytanie COUNT + 1 zapytanie z LEFT OUTER JOIN = 2. Twardy limit!
    assert queries_so_far() <= 2, f"Zbyt wiele zapytań SQL: {queries_so_far()}"
```

> 🧠 Dlaczego to ma sens — Test liczby zapytań nie zależy od danych (przy dwóch książkach i przy dwóch tysiącach asercja jest ta sama), nie zależy od maszyny i nie zależy od dialektu. Jeżeli ktoś w przyszłości usunie `joinedload(Book.author)` z repozytorium, test padnie natychmiast — jeszcze przed wdrożeniem, a nie po pierwszym zgłoszeniu od klienta, że „lista ładuje się trzy sekundy”.

> 🧪 Ćwiczenie — Dodaj test, który sprawdza, że wypożyczenie ostatniego egzemplarza kończy się `201`, a kolejna próba tego samego wypożyczenia zwraca `409` z kodem `no_copies_available`. Pamiętaj, że SQLite ignoruje `FOR UPDATE`.

---

## 22.19 Jak uruchomić cały projekt

```bash
# 1. Środowisko
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate

# 2. Zależności
pip install "sqlalchemy[asyncio]" "fastapi[standard]" pydantic-settings \
            alembic aiosqlite asyncpg typer httpx pytest pytest-asyncio

# 3. Konfiguracja
cp .env.example .env               # domyślnie SQLite: nie trzeba nic zmieniać

# 4. Migracje i dane słownikowe
alembic upgrade head
python -m app.cli seed

# 5. Uruchomienie (SQLite)
uvicorn app.main:app --reload

# 6. Testy
pytest -q
```

Ścieżka PostgreSQL różni się jednym plikiem `.env` i dwoma poleceniami:

```bash
# PostgreSQL: uruchom kontener i ustaw URL
docker run --rm -p 5432:5432 -e POSTGRES_PASSWORD=secret -e POSTGRES_DB=library \
           postgres:17-alpine

# .env:
# APP_DATABASE_URL=postgresql+asyncpg://postgres:secret@localhost:5432/library

alembic upgrade head
python -m app.cli seed
uvicorn app.main:app --reload
```

Sprawdzenie, że wszystko działa, oraz to, co dokładnie się dzieje pod spodem:

```bash
# 1. Dodaj autora wprost w bazie (nie ma jeszcze endpointu POST /authors)
psql "$APP_DATABASE_URL" -c "INSERT INTO authors (name) VALUES ('Ursula K. Le Guin');"

# 2. Utwórz książkę przez API
curl -X POST http://127.0.0.1:8000/api/v1/books \
     -H 'Content-Type: application/json' \
     -d '{"title":"Czarnoksiężnik z Archipelagu","isbn":"9788370750150","author_id":1,
          "copies_available":2}'

# 3. Sprawdź listę
curl 'http://127.0.0.1:8000/api/v1/books?limit=5&available_only=false'
```

Odpowiedź na żądanie 3 (dla PostgreSQL):

```json
{
  "items": [
    {
      "id": 1,
      "title": "Czarnoksiężnik z Archipelagu",
      "isbn": "9788370750150",
      "published_year": null,
      "copies_total": 2,
      "copies_available": 2,
      "created_at": "2026-09-20T10:14:03.221904+00:00",
      "author": {"id": 1, "name": "Ursula K. Le Guin", "bio": null}
    }
  ],
  "total": 1,
  "limit": 5,
  "offset": 0,
  "has_next": false
}
```

A w logu aplikacji (przy `APP_SQL_ECHO=true`) zobaczysz dokładnie te dwa zapytania, o których mówiliśmy w sekcji 22.7: `SELECT count(*)` oraz `SELECT ... LEFT OUTER JOIN authors ... ORDER BY books.title, books.id LIMIT 5 OFFSET 0`.

> ⚠️ Pułapka — Wariant SQLite z `sqlite+aiosqlite:///./library.db` zachowuje się inaczej niż PostgreSQL w trzech miejscach: (1) ignoruje `FOR UPDATE`, więc testy współbieżności są bezwartościowe; (2) nie ma typów `JSONB`, `ARRAY`, `INET`; (3) blokuje całą bazę na czas zapisu, więc równoległe żądania zapisujące układają się w kolejkę. Traktuj SQLite jako wygodne środowisko do nauki i szybkich testów reguł biznesowych, a PostgreSQL jako jedyne wiarygodne środowisko testowania współbieżności.

---

## Podsumowanie

1. **Silnik raz na proces, sesja raz na żądanie, transakcja raz na przypadek użycia.** To trzy niezależne decyzje i wszystkie trzy trzeba podjąć świadomie.
2. **`lifespan` to jedyne właściwe miejsce** na `create_async_engine()` i `build_session_factory()`, a `engine.dispose()` w `finally` zamyka pulę przy zakończeniu procesu.
3. **Zależność z `yield`** (`get_session`) jest właścicielem sesji: ona robi `rollback` przy wyjątku i `close` zawsze. Handler wyjątków robi tylko tłumaczenie na HTTP.
4. **Granica transakcji należy do serwisu lub jednostki pracy**, nie do routera i nie do repozytorium. Wariant „commit po odpowiedzi” jest kuszący i niepoprawny.
5. **Modele ORM i schematy Pydantic to dwie różne warstwy.** `from_attributes=True` nie chroni przed leniwym ładowaniem — relacje muszą być załadowane eageralnie lub zamienione na DTO.
6. **W async leniwe ładowanie nie istnieje.** `lazy="raise"` w modelach zamienia ciche błędy produkcyjne w głośne błędy w testach.
7. **`expire_on_commit=False`** jest w async praktycznie obowiązkowe — inaczej po commicie każdy odczyt atrybutu próbuje wykonać operację wejścia/wyjścia bez `await`.
8. **Wyjątki domenowe to twój kontrakt HTTP.** `IntegrityError` traktuj jako siatkę bezpieczeństwa dla wyścigów, nie jako mechanizm walidacji.
9. **Operacje złożone potrzebują blokady wiersza** (`with_for_update()`), a błąd reguły biznesowej zamień na `409`. Pamiętaj, że SQLite tę klauzulę ignoruje.
10. **Migracje i seed słowników to krok wdrożenia**, nigdy `lifespan`. Przy wielu replikach jedno uruchomienie, najlepiej z blokadą doradczą.
11. **Mierz liczbę zapytań.** Licznik w `before_cursor_execute` podpięty do `engine.sync_engine` plus asercja `queries_so_far() <= 2` to najtańszy test wydajnościowy, jaki możesz napisać.
12. **Zadania w tle dostają dane, nie encje.** Sesja kończy się razem z żądaniem, a `BackgroundTasks` startuje po odpowiedzi.

---

## Ćwiczenia

### Ćwiczenie 1 — Endpoint raportowy z agregacją i paginacją keyset

Dodaj endpoint `GET /api/v1/reports/books-by-author`, który zwraca listę autorów z liczbą książek i liczbą dostępnych egzemplarzy, posortowaną malejąco po liczbie książek, z paginacją **keyset** (kursor zamiast `offset`).

Wymagania:

- parametry: `limit` (1–100, domyślnie 20) oraz `after` (identyfikator autora, od którego zaczynamy);
- sortowanie: `books_count DESC, author_id ASC` — kursor musi być dwuelementowy, inaczej przy równych liczbach książek zgubisz lub zdublujesz wiersze;
- odpowiedź: `items` (id, nazwa autora, liczba książek, liczba dostępnych egzemplarzy) oraz `next_cursor`;
- liczba zapytań na żądanie: **jedno**. Agregacja i strona wyników w jednym `SELECT` z `GROUP BY` i `HAVING`.

### Ćwiczenie 2 — Podłącz jednostkę pracy do istniejącego endpointu

Endpoint `POST /api/v1/books` korzysta dziś z `BookService`, który sam commituje. Przepisz go tak, aby działał wewnątrz `UnitOfWork`, a `BookService` **nie zawierał ani jednego `commit()`**.

Wymagania:

- użyj zależności `get_uow` zamiast `get_session`;
- `BookService.__init__` przyjmuje `UnitOfWork`, nie `AsyncSession`;
- dodaj drugą operację w tej samej transakcji — wpis audytowy (`DictionaryEntry` z `kind='audit'` i `code=f'book_created:{book_id}'`) — aby było widać, że obie zmiany są atomowe;
- napisz test, który udowadnia atomowość: podmień tworzenie wpisu audytowego na operację rzucającą wyjątek i sprawdź, że książka **nie** została utrwalona.

### Ćwiczenie 3 — Licznik zapytań jako brama w CI

Dodaj test parametryzowany, który dla trzech endpointów listujących (`/books`, `/authors/{id}/books`, `/reports/active-loans`) sprawdza górne limity liczby zapytań, i podłącz go do pipeline’u jako osobny krok „performance”.

Wymagania:

- limity: `/books` ≤ 2, `/authors/{id}/books` ≤ 2, `/reports/active-loans` ≤ 1;
- test ma być niezależny od liczby rekordów w bazie (użyj 50 książek i 10 autorów);
- komunikat błędu ma wskazywać zmierzoną liczbę i limit, żeby diagnoza nie wymagała ponownego uruchomienia.

### Rozwiązania

#### Rozwiązanie 1 — raport z agregacją i keyset

```python
# app/schemas/report.py
from pydantic import BaseModel


class AuthorStats(BaseModel):
    author_id: int
    author_name: str
    books_count: int
    available_total: int


class AuthorStatsPage(BaseModel):
    items: list[AuthorStats]
    next_cursor: tuple[int, int] | None  # (books_count, author_id)
```

```python
# app/repositories/authors.py — metoda dodana do AuthorRepository
from sqlalchemy import func, or_, select, tuple_

    async def stats_page(
        self,
        *,
        limit: int,
        after: tuple[int, int] | None = None,
    ) -> list[tuple[int, str, int, int]]:
        """Autorzy z agregatami, strona w jednym zapytaniu (keyset pagination).

        Kursor to para (books_count, author_id) — przy równych liczbach książek
        o kolejności decyduje id, więc strona nie gubi ani nie dubluje wierszy.
        """
        from app.models import Author, Book

        books_count = func.count(Book.id).label("books_count")
        available_total = func.coalesce(func.sum(Book.copies_available), 0).label(
            "available_total"
        )

        stmt = (
            select(Author.id, Author.name, books_count, available_total)
            .outerjoin(Book, Book.author_id == Author.id)
            .group_by(Author.id, Author.name)
            .order_by(books_count.desc(), Author.id.asc())
            .limit(limit)
        )

        if after is not None:
            last_count, last_id = after
            stmt = stmt.having(
                or_(
                    books_count < last_count,
                    (books_count == last_count) & (Author.id > last_id),
                )
            )

        rows = (await self._session.execute(stmt)).all()
        return [(row[0], row[1], int(row[2]), int(row[3])) for row in rows]
```

```python
# app/api/v1/reports.py
from typing import Annotated

from fastapi import APIRouter, Depends, Query
from sqlalchemy.ext.asyncio import AsyncSession

from app.api.deps import get_session
from app.repositories.authors import AuthorRepository
from app.schemas.report import AuthorStats, AuthorStatsPage

router = APIRouter(prefix="/reports", tags=["reports"])

SessionDep = Annotated[AsyncSession, Depends(get_session)]


@router.get("/books-by-author", response_model=AuthorStatsPage)
async def books_by_author(
    session: SessionDep,
    limit: Annotated[int, Query(ge=1, le=100)] = 20,
    after: Annotated[str | None, Query(description="kursor: books_count,author_id")] = None,
) -> AuthorStatsPage:
    cursor: tuple[int, int] | None = None
    if after:
        count_str, id_str = after.split(",", maxsplit=1)
        cursor = (int(count_str), int(id_str))

    rows = await AuthorRepository(session).stats_page(limit=limit, after=cursor)
    items = [
        AuthorStats(author_id=r[0], author_name=r[1], books_count=r[2], available_total=r[3])
        for r in rows
    ]
    next_cursor = (items[-1].books_count, items[-1].author_id) if len(items) == limit else None
    return AuthorStatsPage(items=items, next_cursor=next_cursor)
```

Dlaczego keyset jest tu obowiązkowy, a nie tylko ładny: autorzy mają często identyczne liczby książek, a `ORDER BY books_count DESC` z `offset` na zmieniających się danych gwarantuje, że użytkownik zobaczy część wierszy dwukrotnie albo nie zobaczy ich wcale. Zapis `(books_count, author_id)` w kursorze czyni porządek całkowitym — a porządek całkowity to jedyna podstawa, na której paginacja może być poprawna.

#### Rozwiązanie 2 — księgowanie w jednostce pracy

```python
# app/services/books.py — wersja z UnitOfWork (bez commitów w serwisie)
from app.services.uow import UnitOfWork


class BookServiceUoW:
    def __init__(self, uow: UnitOfWork) -> None:
        self._uow = uow

    async def create_book(self, data: BookCreate) -> Book:
        author = await self._uow.authors.get(data.author_id)
        if author is None:
            raise AuthorNotFound(data.author_id)
        if await self._uow.books.exists_isbn(data.isbn):
            raise DuplicateISBN(data.isbn)

        book = Book(
            title=data.title,
            isbn=data.isbn,
            published_year=data.published_year,
            copies_total=data.copies_total,
            copies_available=data.copies_total,
            author=author,
        )
        await self._uow.books.add(book)
        await self._uow.session.flush()  # nadaje book.id wewnątrz transakcji

        # Druga zmiana w TEJ SAMEJ transakcji — atomowość.
        self._uow.session.add(
            DictionaryEntry(
                kind="audit",
                code=f"book_created:{book.id}",
                label=f"Utworzono książkę {book.title}",
                sort_order=0,
            )
        )
        return book
```

```python
# app/api/v1/books.py — endpoint na jednostce pracy
@router.post(
    "",
    response_model=BookRead,
    status_code=status.HTTP_201_CREATED,
    summary="Dodaj książkę (w jednej transakcji z wpisem audytowym)",
)
async def create_book_uow(payload: BookCreate, uow: UowDep) -> BookRead:
    book = await BookServiceUoW(uow).create_book(payload)
    return BookRead.model_validate(book)
```

```python
# tests/test_uow_atomicity.py
import pytest
from httpx import AsyncClient
from sqlalchemy import func, select

from app.models import Book
from app.models.dictionary import DictionaryEntry

pytestmark = pytest.mark.asyncio


async def test_failed_audit_rolls_back_book(client: AsyncClient, monkeypatch) -> None:
    """Gdy druga operacja padnie, pierwsza też musi zniknąć."""
    from app.services import books as books_module

    async def boom(*args, **kwargs):
        raise RuntimeError("symulowany błąd audytu")

    monkeypatch.setattr(books_module, "DictionaryEntry", boom)  # type: ignore[arg-type]

    payload = {
        "title": "Książka widmo",
        "isbn": "9999999999999",
        "author_id": 1,
        "copies_available": 1,
    }
    response = await client.post("/api/v1/books", json=payload)
    assert response.status_code == 500  # błąd nieprzewidziany → 500 + log

    factory = client._transport.app.state.session_factory  # type: ignore[attr-defined]
    async with factory() as session:
        count = await session.scalar(
            select(func.count()).select_from(Book).where(Book.isbn == "9999999999999")
        )
        entries = await session.scalar(select(func.count()).select_from(DictionaryEntry))
    assert count == 0
    assert entries == 0
```

Warto zwrócić uwagę na jedną rzecz w tym teście: `monkeypatch.setattr` podmienia klasę `DictionaryEntry` na funkcję, więc `DictionaryEntry(kind=...)` w serwisie zgłasza wyjątek. `UnitOfWork.__aexit__` widzi wyjątek, wykonuje `rollback()` i **ponownie rzuca** — dlatego `IntegrityError`/`RuntimeError` dociera do handlera i zamienia się w `500`. To jest dokładnie ten przepływ, który opisaliśmy w sekcji 22.8.

#### Rozwiązanie 3 — brama wydajnościowa w CI

```python
# tests/test_performance_budget.py
import pytest
from httpx import AsyncClient
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.observability import queries_so_far, start_counting
from app.models import Author, Book

pytestmark = pytest.mark.asyncio

BUDGETS = {
    "/api/v1/books?limit=50": 2,
    "/api/v1/reports/active-loans?limit=50": 1,
}


@pytest.fixture
async def many_rows(app_instance) -> None:
    """50 książek i 10 autorów — liczby dobrane tak, by N+1 na pewno się ujawnił."""
    factory = app_instance.state.session_factory
    async with factory() as session:
        authors = [Author(name=f"Autor {i}") for i in range(10)]
        session.add_all(authors)
        await session.flush()
        session.add_all(
            [
                Book(
                    title=f"Książka {i}",
                    isbn=f"978000000{i:04d}",
                    copies_total=2,
                    copies_available=1,
                    author_id=authors[i % 10].id,
                )
                for i in range(50)
            ]
        )
        await session.commit()


@pytest.mark.parametrize(("url", "budget"), BUDGETS.items())
async def test_query_budget(client: AsyncClient, many_rows: None, url: str, budget: int) -> None:
    start_counting()
    response = await client.get(url)
    measured = queries_so_far()

    assert response.status_code == 200
    assert measured <= budget, f"{url}: zmierzono {measured} zapytań, budżet to {budget}"
```

```yaml
# .github/workflows/ci.yml (fragment - krok wydajnościowy po testach jednostkowych)
      - name: Testy
        run: pytest -q --ignore=tests/test_performance_budget.py

      - name: Budżet zapytań SQL
        run: pytest -q tests/test_performance_budget.py
```

Rozdzielenie kroków jest celowe: gdy brama wydajnościowa padnie, od razu widać w interfejsie CI, że problem jest wydajnościowy, a nie funkcjonalny. Nowy programista w zespole nie musi czytać logu, żeby zrozumieć, co się stało.

---

## Najczęstsze błędy i jak je czytać

| Komunikat / objaw | Przyczyna | Naprawa |
|---|---|---|
| `MissingGreenlet: greenlet_spawn has not been called; can't call await_only() here` | leniwe ładowanie relacji w kodzie async; dostęp do atrybutu poza `await` | dodaj `joinedload()`/`selectinload()` w repozytorium; ustaw `lazy="raise"` w modelu, by błąd wychwycić wcześniej |
| `DetachedInstanceError: Instance <Book> is not bound to a Session` | obiekt użyty po zamknięciu sesji (serializacja, zadanie w tle, inny wątek) | zwróć DTO albo załaduj relacje przed zamknięciem; do zadań w tle przekazuj dane, nie encje |
| `InvalidRequestError: 'Book.author' is not available due to lazy='raise'` | dostęp do relacji bez jawnego załadowania | dodaj `.options(joinedload(Book.author))` do zapytania |
| `RuntimeWarning: coroutine 'AsyncSession.flush' was never awaited` | brak `await` przy `flush()`, `commit()`, `delete()`, `execute()` | dopisz `await` (i sprawdź, czy metoda naprawdę jest asynchroniczna) |
| `ValueError: the greenlet library is required to use this function` (SQLAlchemy 2.1) | zainstalowano `sqlalchemy` bez `[asyncio]`; w 2.1 `greenlet` nie przychodzi domyślnie | `pip install "sqlalchemy[asyncio]"` |
| `asyncpg.exceptions.TooManyConnectionsError` / `too many clients already` | silnik tworzony per request albo `pool_size` większy niż limit serwera razy liczba replik | jeden silnik na proces w `lifespan`; `pool_size × repliki < max_connections` serwera |
| Zwracasz `422` tam, gdzie klient oczekuje `400`/`409` | walidacja Pydantic (kształt) zamiast reguły biznesowej (sens) | przenieś sprawdzenie do serwisu i rzuć wyjątek domenowy z właściwym kodem |
| `500` zamiast `409` przy duplikacie | `IntegrityError` nieobsłużony, handler generyczny łapie wszystko | dodaj handler `IntegrityError` (sekcja 22.8) **oraz** sprawdzenie w serwisie |
| `ResponseValidationError` przy zwracaniu encji | schemat oczekuje pola, którego nie ma w encji (np. `book_title` w raporcie) | użyj DTO albo zbuduj pola jawnie w serwisie |
| SQLite: `no such table: books` w testach | `:memory:` bez `StaticPool` — każde połączenie to osobna, pusta baza | dodaj `poolclass=StaticPool` i `connect_args={"check_same_thread": False}` |
| SQLite: `sqlite3.OperationalError: database is locked` | plikowy SQLite z równoległym zapisem; brak `timeout` lub WAL | ustaw `connect_args={"timeout": 30}`; na produkcji przejdź na PostgreSQL |
| `FOR UPDATE` nie działa w testach | SQLite nie obsługuje blokad wiersza — dialekt pomija klauzulę | testuj współbieżność na PostgreSQL (`testcontainers`); na SQLite testuj tylko regułę biznesową |
| `RuntimeError: Task got Future attached to a different loop` | jedna sesja/engine używane w dwóch pętlach zdarzeń (np. sesja tworzona poza fixture) | twórz silnik i sesję w obrębie tej samej pętli; w testach używaj `asyncio_mode = "auto"` i fixture o właściwym zakresie |
| Nagłówek `X-Query-Count` pokazuje `0` mimo zapytań | licznik oparty na `contextvars` bez mutowalnego kontenera — mutacja nie wraca z zadania do middleware | przechowuj stan w liście (`ContextVar[list[int]]`), tak jak w sekcji 22.14 |
| `InvalidRequestError: This session is in 'prepared' state` | jednostka pracy commitowała dwa razy albo `commit()` wykonano po wyjątku bez `rollback()` | jedno miejsce commitowania; `rollback()` w `__aexit__` przy wyjątku |
| Błąd importu cyklicznego przy `models/` | `from app.models.book import Book` w `author.py` na poziomie modułu | import pod `if TYPE_CHECKING:`; relacje opisuj anotacjami w cudzysłowie |
| `alembic revision --autogenerate` generuje drop+add zamiast rename | Alembic nie wykrywa zmiany nazwy kolumny | edytuj migrację ręcznie: `op.alter_column(..., new_column_name=...)` |

---

## Słowniczek modułu

| Termin (EN) | Polski odpowiednik | Wyjaśnienie |
|---|---|---|
| `AsyncEngine` | silnik asynchroniczny | Obiekt zarządzający pulą połączeń w trybie async; fabryka połączeń, nie połączenie. Tworzony raz na proces. |
| `AsyncSession` | sesja asynchroniczna | Jednostka pracy w trybie async; wszystkie operacje wymagają `await`. Jedna na żądanie HTTP. |
| `async_sessionmaker` | fabryka sesji (async) | Wywoływalny obiekt tworzący nowe `AsyncSession` z ustaloną konfiguracją (`expire_on_commit`, `autoflush`). |
| `lifespan` | cykl życia aplikacji | Kontekst FastAPI: kod przed `yield` wykonuje się przy starcie, po `yield` przy zamykaniu. Miejsce na silnik i `dispose()`. |
| dependency injection | wstrzykiwanie zależności | Mechanizm FastAPI: endpoint deklaruje, czego potrzebuje (`Depends(...)`), a framework dostarcza obiekt. |
| `Depends` | zależność | Deklaracja „daj mi ten obiekt”. Dla sesji i jednostki pracy używamy wariantu z `yield` (generatorowego). |
| `yield` dependency | zależność generatorowa | Zależność, która ma sekcję „po zwróceniu” — wykonywaną po zbudowaniu odpowiedzi; tu robimy `rollback`/`close`. |
| commit-on-success | zatwierdzanie po sukcesie | Wariant, w którym `commit()` wykonuje się w zależności po odpowiedzi. Pokazany jako antywzorzec. |
| `ModelConfig(from_attributes=True)` | konfiguracja modelu | Ustawienie Pydantic v2 pozwalające budować schemat z obiektu (także encji ORM) przez `getattr`. |
| DTO | obiekt transferu danych | Klasa opisująca dane wyjściowe niezależnie od encji; używana w raportach i kontraktach API. |
| `Page[T]` | strona wyników | Generyczny schemat paginacji: `items`, `total`, `limit`, `offset`, `has_next`. |
| keyset pagination | paginacja kursorowa | Stronicowanie po całkowitym porządku (`WHERE (a, b) > (x, y)`) zamiast `OFFSET`; nie gubi i nie dubluje wierszy. |
| unit of work | jednostka pracy | Obiekt grupujący sesję i repozytoria, decydujący o `commit`/`rollback` na wyjściu z kontekstu. |
| `with_for_update()` | blokada wiersza | Generuje `SELECT ... FOR UPDATE`; blokuje wiersz do końca transakcji. **Nie działa na SQLite.** |
| `IntegrityError` | błąd integralności | Wyjątek sterownika przy naruszeniu ograniczenia (unikalność, klucz obiegowy). Traktowany jako siatka bezpieczeństwa. |
| wyjątek domenowy | błąd biznesowy | Własny wyjątek aplikacji (`BookNotFound`, `DuplicateISBN`) mapowany na konkretny kod HTTP. |
| `BackgroundTasks` | zadania w tle | Wykonują się po wysłaniu odpowiedzi; **nie** przedłużają życia zależności ani sesji. |
| N+1 | problem N+1 zapytań | Wzorzec: jedno zapytanie na listę i N dodatkowych na relacje. Złożoność $O(1+N)$ zamiast $O(1)$. |
| `joinedload` / `selectinload` | ładowanie złączone / przez `IN` | Strategie jawnego ładowania relacji: jedno zapytanie z JOIN albo dwa zapytania z `IN (...)`. |
| query counter | licznik zapytań | Nasłuch `before_cursor_execute` podpięty do `engine.sync_engine`; podstawa testów wydajnościowych. |
| `engine.sync_engine` | silnik synchroniczny pod async | Zdarzenia SQLAlchemy podłącza się do niego; `AsyncEngine` sam nie emituje zdarzeń. |
| advisory lock | blokada doradcza | `pg_advisory_xact_lock()` — koordynacja procesów (np. seed) bez blokowania tabel. |
| expand/contract | rozszerz i zawęź | Wzorzec wdrożenia migracji: dodaj kolumnę, podwójny zapis, migracja danych, usuń starą kolumnę. |
| `render_as_batch` | tryb wsadowy Alembica | Odtwarzanie tabeli dla SQLite, bo nie obsługuje większości `ALTER TABLE`. |

---

## Dalsze czytanie

- **SQLAlchemy — asyncio (ORM)**: https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html
- **SQLAlchemy — podstawy sesji i transakcji**: https://docs.sqlalchemy.org/en/20/orm/session_basics.html
- **SQLAlchemy — przewodnik po zapytaniach ORM (relacje, eager loading)**: https://docs.sqlalchemy.org/en/20/orm/queryguide/relationships.html
- **SQLAlchemy — połączenia i pula połączeń (Core)**: https://docs.sqlalchemy.org/en/20/core/connections.html
- **SQLAlchemy — style mapowania (declarative, imperatywne)**: https://docs.sqlalchemy.org/en/20/orm/mapping_styles.html
- **SQLAlchemy — dialekt PostgreSQL (typy, `INSERT ... ON CONFLICT`)**: https://docs.sqlalchemy.org/en/20/dialects/postgresql.html
- **FastAPI — zależności z `yield`**: https://fastapi.tiangolo.com/tutorial/dependencies/
- **FastAPI — zdarzenia cyklu życia (`lifespan`)**: https://fastapi.tiangolo.com/advanced/events/
- **Pydantic v2 — modele i `from_attributes`**: https://docs.pydantic.dev/latest/concepts/models/
- **pydantic-settings — konfiguracja ze zmiennych środowiskowych**: https://docs.pydantic.dev/latest/concepts/pydantic_settings/
- **Alembic — autogenerate i `env.py`**: https://alembic.sqlalchemy.org/en/latest/autogenerate.html
- **Starlette — cykl życia aplikacji**: https://www.starlette.io/lifespan/
- **PostgreSQL — jawne blokady (`FOR UPDATE`, poziomy izolacji)**: https://www.postgresql.org/docs/current/explicit-locking.html

---

## Co dalej

Masz aplikację, która ma warstwy, konfigurację, migracje, obsługę błędów, blokadę wiersza i testy. Potrafisz uzasadnić każdą decyzję architektoniczną i wiesz, co się stanie, gdy dwie osoby klikną „wypożycz” w tej samej milisekundzie. To dokładnie zestaw umiejętności, którego wymaga od ciebie projekt końcowy.

W [module 23 — Projekt końcowy](23_projekt_koncowy.md) nie dostaniesz już gotowego kodu. Dostaniesz specyfikację wypożyczalni sprzętu, dziewięć etapów pracy z kryteriami ukończenia, punktację oceny i pytania, które powinieneś sobie zadać przed uznaniem projektu za gotowy. Wszystko, czego potrzebujesz, jest w modułach 01–22 — a to, co najtrudniejsze, będzie w decyzjach: gdzie postawić granicę transakcji, co wyeksponować w API i czego świadomie nie implementować.

<!-- koniec modułu 22 -->