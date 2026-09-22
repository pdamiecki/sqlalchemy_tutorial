# Moduł 18 — Testowanie kodu ze SQLAlchemy

Testy to jedyny sposób, w jaki możesz **udowodnić**, że Twój kod działa — a nie tylko w to wierzyć. W tym module nauczysz się pisać szybkie, izolowane i wiarygodne testy dla kodu korzystającego z SQLAlchemy. Dowiesz się, jak skonfigurować bazę testową (SQLite w pamięci i PostgreSQL w kontenerze), jak zapewnić izolację między testami, jak budować fabryki danych, jak testować zachowania specyficzne dla ORM (kaskady, walidatory, zdarzenia, blokady optymistyczne) oraz jak wykrywać regresje wydajnościowe, licząc zapytania SQL. Na koniec złożysz wszystko w jeden działający `conftest.py` plus zestaw ośmiu testów.

---

**Poziom:** 🔴 architektoniczny
**Czas:** ~180 minut
**Wymagania wstępne:**

- `08_sesja_cykl_zycia.md` — cykl życia sesji, `flush` vs `commit`, transakcje.
- `09_relacje.md` — kaskady, `relationship`, tabele asocjacyjne.
- `11_ladowanie_i_n_plus_1.md` — strategie ładowania i problem N+1 (kluczowy dla testów liczby zapytań).
- `14_transakcje_i_wspolbieznosc.md` — blokada optymistyczna, `StaleDataError`.
- Mile widziane: `15_asynchronicznosc.md` — dla sekcji o testach async.

**Czego dotyczy plik:** strategii testowania warstwy danych napisanej na SQLAlchemy 2.0+, konfiguracji środowiska testowego, izolacji, fabryk danych i testowania zachowań specyficznych dla ORM.

---

## Spis treści

- [Po co testować warstwę danych](#po-co-testować-warstwę-danych)
- [pytest w pięć minut](#pytest-w-pięć-minut)
- [Baza testowa — czym testować](#baza-testowa--czym-testować)
- [Izolacja testów — trzy strategie](#izolacja-testów--trzy-strategie)
- [Wersjonowanie schematu w testach](#wersjonowanie-schematu-w-testach)
- [Fabryki danych](#fabryki-danych)
- [Testowanie zachowań specyficznych dla SQLAlchemy](#testowanie-zachowań-specyficznych-dla-sqlalchemy)
- [Testowanie zapytań i liczby zapytań](#testowanie-zapytań-i-liczby-zapytań)
- [Testy asynchroniczne](#testy-asynchroniczne)
- [Mockowanie — kiedy pomaga, kiedy szkodzi](#mockowanie--kiedy-pomaga-kiedy-szkodzi)
- [Dane testowe a produkcja](#dane-testowe-a-produkcja)
- [Struktura testów w projekcie](#struktura-testów-w-projekcie)
- [Pełny przykład — conftest.py i osiem testów](#pełny-przykład--conftestpy-i-osiem-testów)
- [Podsumowanie](#podsumowanie)
- [Ćwiczenia](#ćwiczenia)
- [Najczęstsze błędy i jak je czytać](#najczęstsze-błędy-i-jak-je-czytać)
- [Słowniczek modułu](#słowniczek-modułu)
- [Dalsze czytanie](#dalsze-czytanie)
- [Co dalej](#co-dalej)

---

## Po co testować warstwę danych

Zacznijmy od pytania, które brzmi banalnie, ale w praktyce rzadko dostaje dobrą odpowiedź: **co właściwie chcemy udowodnić testami?**

> 💡 **Analogia — nie testujemy silnika, testujemy trasę.**
> Wyobraź sobie, że kupiłeś samochód. Nie rozkręcasz silnika, żeby sprawdzić, czy tłoki poruszają się w odpowiedniej kolejności — to robi producent na linii montażowej. Ty sprawdzasz coś innego: czy samochód jedzie tam, gdzie chcesz, czy skręca, gdy kręcisz kierownicą, czy hamuje, gdy wciskasz hamulec. Testujesz **trasę i prowadzenie**, nie **konstrukcję silnika**.
>
> W naszym świecie „silnikiem” jest SQLAlchemy. Silnik ma własne, bardzo obszerne testy (tysiące przypadków w repozytorium SQLAlchemy). Ty masz przetestować **swoją trasę**: swoje modele, swoje zapytania, swoją logikę biznesową.

Ta analogia prowadzi do konkretnej listy. Oto, czego **nie testujemy**:

- Nie testujemy, czy `select(Book).where(Book.id == 1)` zwróci książkę — to jest funkcja biblioteki, udokumentowana i przetestowana.
- Nie testujemy, czy SQLAlchemy poprawnie wygeneruje `JOIN` — to zadanie dialektu.
- Nie testujemy, czy SQLite obsługuje `INSERT` — to zadanie bazy danych.
- Nie testujemy `commit()` jako takiego.

A oto, co **testujemy**:

- Czy nasz model deklaratywny faktycznie ma kolumnę `isbn` o typie `String` i ograniczenie unikalności (mapowanie).
- Czy walidator `@validates` odrzuca zły e-mail i normalizuje poprawny (logika).
- Czy kaskada `delete-orphan` rzeczywiście usuwa recenzje z bazy (zachowanie).
- Czy repozytorium zwraca książki w ustalonej kolejności (kontrakt).
- Czy zapytanie w ścieżce krytycznej nie generuje N+1 (regresja wydajnościowa).
- Czy blokada optymistyczna wyłapuje konflikt dwóch użytkowników (logika biznesowa).
- Czy błąd bazy jest tłumaczony na błąd aplikacyjny, a nie wycieka do API.

Zauważ wspólny mianownik: **testujemy kod, który sami napisaliśmy**. To nie przypadek — to jedyna sensowna granica.

### Piramida testów a warstwa danych

> 💡 **Analogia — piramida testów to piramida egzaminów.**
> Wyobraź sobie, że uczysz się do egzaminu. Masz szybkie fiszki (odpowiedź w 2 sekundy — sprawdzasz słówka), kartkówki (2 minuty — sprawdzasz zrozumienie jednego tematu) i egzamin próbny (2 godziny — sprawdzasz całość w warunkach bojowych). Nie zdajesz egzaminu, robiąc tylko fiszki, ale nie robisz egzaminu próbnego sto razy dziennie — bo trwa za długo.
>
> Tak samo z testami: dużo szybkich (jednostkowe), mniej średnich (integracyjne), kilka długich (end-to-end).

| Poziom | Czas jednego testu | Czy dotyka bazy? | Co sprawdza w warstwie danych |
|---|---|---|---|
| **Jednostkowe** | milisekundy | ❌ nie | czysta logika bez sesji: obliczenia, walidacja w `@validates`, funkcje pomocnicze, `TypeDecorator.process_bind_param` |
| **Integracyjne** | 10–200 ms | ✅ tak, baza testowa | modele, relacje, kaskady, zapytania, repozytoria, granice transakcji |
| **End-to-end** | sekundy | ✅ prawdziwa infrastruktura | działające API, migracje, pełny przepływ „od HTTP do HTTP” |

W praktyce **90% testów warstwy danych to testy integracyjne z bazą w pamięci**. I to jest w porządku — właśnie po to SQLite in-memory istnieje.

> 🧠 **Dlaczego tak jest — baza w pamięci jest „darmowa”.**
> Testy integracyjne są zwykle drogie, bo wymagają stawiania prawdziwej infrastruktury. SQLite w pamięci usuwa ten koszt: baza nie istnieje na dysku, nie ma procesu serwera, nie ma sieci. Cały koszt to zaalokowanie kilku kilobajtów RAM. Dzięki temu możesz mieć **tysiące** testów integracyjnych, które uruchamiają się w kilka sekund.

> ⚠️ **Pułapka — pokrycie kodu nie jest celem.**
> „Mamy 98% pokrycia!” brzmi dumnie, dopóki nie odkryjesz, że 80% tych testów to sprawdzanie getterów, a żaden nie sprawdza, co się dzieje przy dwóch równoczesnych rezerwacjach. Pokrycie mierzy, **ile linii zostało wykonanych**, a nie **ile zachowań zostało zweryfikowanych**. Używaj `pytest-cov` jako narzędzia do szukania *dziur*, nie jako celu samego w sobie.

> 🧪 **Ćwiczenie — w miejscu.**
> Zajrzyj do swojego ostatniego projektu (albo do kodu z modułów 09–11). Wypisz pięć rzeczy, które poszłyby źle, gdyby ktoś zmienił jedną kolumnę w modelu. Przy każdej napisz jedno zdanie: „ten test by to złapał, gdyby sprawdzał…”. To Twoja lista priorytetów na ten moduł.

---

## pytest w pięć minut

`pytest` to standardowy framework testowy w Pythonie. Jest prostszy niż `unittest` (który wymaga klas i metod `assertEqual`), a mimo to potężniejszy — głównie dzięki *fixtures*.

### Instalacja i pierwszy test

```bash
pip install pytest pytest-cov
```

```python
# tests/test_pierwszy.py
def dodaj(a: int, b: int) -> int:
    return a + b


def test_dodawanie() -> None:
    # pytest rozumie samo `assert` — nie potrzebujesz specjalnych metod
    assert dodaj(2, 3) == 5


def test_dodawanie_jest_przemienne() -> None:
    assert dodaj(2, 3) == dodaj(3, 2)
```

Uruchomienie:

```bash
pytest -q            # -q = quiet, mniej hałasu
pytest tests/test_pierwszy.py::test_dodawanie   # jeden konkretny test
pytest -k "dodawanie"                            # testy, których nazwa zawiera frazę
```

Konwencje, które pytest rozpoznaje automatycznie:

- pliki: `test_*.py` lub `*_test.py`,
- funkcje: `test_*`,
- klasy: `Test*` (bez `__init__`).

### Fixtures — przygotowanie i sprzątanie

Fixture to funkcja, która *dostarcza* coś testowi. Brzmi jak zwykła funkcja pomocnicza, ale ma trzy cechy, których zwykła funkcja nie ma:

1. pytest **wstrzykuje** ją do testu na podstawie nazwy argumentu,
2. może mieć **zakres** (scope) — od pojedynczego testu do całej sesji pytest,
3. może mieć część „sprzątającą” po `yield` — to gwarantuje, że sprzątanie wykona się nawet gdy test rzuci wyjątek.

> 💡 **Analogia — fixture to kelner przygotowujący stolik.**
> Zanim usiądziesz, kelner nakrywa stolik (przygotowanie), a po Twoim wyjściu sprząta (sprzątanie). Nie musisz tego robić sam w każdym teście. A jeśli test „wyszedł z hukiem” (rzucił wyjątek), kelner i tak posprząta — bo sprzątanie jest w `finally`-owej części `yield`.

```python
# tests/test_fixtures.py
import pytest


@pytest.fixture
def lista_ksiazek() -> list[str]:
    # przygotowanie
    ksiazki = ["Wiedźmin", "Solaris", "Ferdydurke"]
    yield ksiazki
    # sprzątanie (tutaj nic nie trzeba, ale pokazuje mechanizm)
    ksiazki.clear()


def test_ksiazek_jest_trzy(lista_ksiazek: list[str]) -> None:
    assert len(lista_ksiazek) == 3
```

Zwróć uwagę: w argumencie testu jest `lista_ksiazek` — pytest widzi nazwę, znajduje fixture i **sam** ją wywołuje.

### `conftest.py` — współdzielone fixtures

Jeśli fixture ma być dostępna w wielu plikach testowych, umieszczasz ją w `conftest.py`. pytest automatycznie importuje ten plik dla wszystkich testów w danym katalogu i podkatalogach.

```text
tests/
├── conftest.py          # fixtures widoczne dla wszystkich testów poniżej
├── test_models.py
└── integration/
    ├── conftest.py      # dodatkowe fixtures tylko dla testów integracyjnych
    └── test_repository.py
```

> 🧠 **Dlaczego tak jest — `conftest.py` nie wymaga importu.**
> Nie piszesz `from conftest import db_session`. pytest zbiera pliki `conftest.py` podczas „schodzenia” po katalogach i rejestruje ich fixtures globalnie. To wygoda, ale też źródło zamieszania: jeśli masz trzy pliki `conftest.py` na różnych poziomach i fixture o tej samej nazwie, wygrywa ta **najbliżej** testu. Nazywaj fixtures precyzyjnie, żeby nie walczyć z tym mechanizmem.

### Parametryzacja — jeden test, wiele przypadków

```python
# tests/test_parametryzacja.py
import pytest


@pytest.mark.parametrize(
    ("tekst", "oczekiwany"),
    [
        ("  ALA@Example.COM ", "ala@example.com"),
        ("jan@kowalski.pl", "jan@kowalski.pl"),
        ("test@test.io", "test@test.io"),
    ],
)
def test_normalizacja_emaila(tekst: str, oczekiwany: str) -> None:
    assert tekst.strip().lower() == oczekiwany
```

Parametryzacja to jedyny sposób, by nie kopiować tego samego testu pięć razy. W wynikach pytest zobaczysz pięć osobnych przypadków: `test_normalizacja_emaila[tekst0-oczekiwany0]`, itd.

### `tmp_path` i `monkeypatch` — dwa fixtures wbudowane

```python
# tests/test_wbudowane.py
from pathlib import Path


def test_zapis_pliku(tmp_path: Path) -> None:
    # tmp_path to unikalny katalog tymczasowy, tworzony per test i usuwany po nim
    plik = tmp_path / "dane.txt"
    plik.write_text("hello", encoding="utf-8")
    assert plik.read_text(encoding="utf-8") == "hello"


def test_zmienna_srodowiskowa(monkeypatch) -> None:
    # monkeypatch zmienia stan globalny na czas testu i przywraca go po
    monkeypatch.setenv("DATABASE_URL", "sqlite:///:memory:")
    import os

    assert os.environ["DATABASE_URL"] == "sqlite:///:memory:"
    # po zakończeniu testu zmienna jest automatycznie przywrócona
```

`tmp_path` jest szczególnie przydatny przy testach z bazą plikową — daje Ci czysty katalog na plik `.db` bez martwienia się o sprzątanie.

### `pytest-cov` — pokrycie kodu

```bash
pytest --cov=app --cov-report=term-missing
```

`--cov=app` mierzy pokrycie pakietu `app`, `term-missing` pokazuje, **które konkretnie linie** nie zostały wykonane. To znacznie cenniejsze niż sam procent.

### Markery — oznaczanie testów

```python
# tests/test_slow.py
import pytest


@pytest.mark.slow
def test_pelna_migracja() -> None:
    ...


@pytest.mark.integration
def test_repozytorium() -> None:
    ...
```

Konfiguracja w `pyproject.toml`:

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
markers = [
    "slow: testy trwające dłużej niż sekundę",
    "integration: testy wymagające bazy danych",
]
```

Uruchamianie selektywne:

```bash
pytest -m "not slow"     # bez wolnych testów — szybka pętla lokalna
pytest -m integration    # tylko integracyjne — w CI
```

> ⚠️ **Pułapka — nierozpoznany marker to cichy problem.**
> Bez sekcji `markers` w konfiguracji pytest wypisze tylko ostrzeżenie (`PytestUnknownMarkWarning`), ale testy **przejdą**. Problemy pojawiają się, gdy w CI użyjesz `-m slow` i okaże się, że wszystkie testy mają literówkę w nazwie markera. Zawsze deklaruj markery.

> 🧪 **Ćwiczenie — w miejscu.**
> Napisz fixture `przykladowy_engine`, która zwraca `create_engine("sqlite://")` i sprząta po sobie przez `yield` + `dispose()`. Uruchom z argumentem `--setup-show`, żeby zobaczyć, w jakiej kolejności pytest wykonuje przygotowanie i sprzątanie.

---

## Baza testowa — czym testować

To najważniejsza sekcja techniczna modułu. Wybór bazy testowej determinuje szybkość, wiarygodność i „produkcyjność” Twoich testów.

### SQLite w pamięci — wariant podstawowy

```python
# tests/conftest.py (fragment)
from collections.abc import Iterator

import pytest
from sqlalchemy import Engine, create_engine
from sqlalchemy.pool import StaticPool

from tests.models import Base


@pytest.fixture(scope="session")
def engine() -> Iterator[Engine]:
    engine = create_engine(
        # 1) "sqlite://" oznacza bazę w pamięci, prywatną dla tego procesu Pythona.
        #    Uwaga: "sqlite:///" (trzy ukośniki) to baza *plikowa* w katalogu
        #    bieżącym — dwie zupełnie różne rzeczy!
        "sqlite://",
        # 2) Domyślny pool dla SQLite to QueuePool/NULL, który po zamknięciu
        #    połączenia zwraca je do puli. Baza ":memory:" żyje WEWNĄTRZ jednego
        #    połączenia — gdy połączenie zostanie zwrócone/zamknięte, dane znikają.
        #    StaticPool trzyma JEDNO połączenie na zawsze i wydaje je wszystkim.
        poolclass=StaticPool,
        # 3) SQLAlchemy może używać tego samego połączenia z różnych wątków
        #    (np. TestClient FastAPI, pytest-xdist). SQLite domyślnie tego zabrania
        #    (parametr check_same_thread). Wyłączamy tę kontrolę — bezpiecznie,
        #    dopóki wszystkie operacje idą przez jedno połączenie z StaticPool.
        connect_args={"check_same_thread": False},
    )
    # 4) Tworzymy schemat raz na całą sesję testową — najszybciej.
    Base.metadata.create_all(engine)
    yield engine
    # 5) dispose() zamyka pulę; dla StaticPool zamyka to jedno połączenie.
    engine.dispose()
```

**Wyjaśnijmy każdy parametr po kolei, bo to klasyczne miejsce na błędy.**

> 🔬 **Pod maską — co się dzieje bez `StaticPool`.**
> Załóżmy, że użyliśmy `create_engine("sqlite://")` bez `poolclass`. Pierwsze zapytanie pobiera z puli połączenie **P1**, w którym baza w pamięci zawiera już nasze tabele. Gdy połączenie wraca do puli, to jest w porządku. Ale gdy zostanie **zamknięte** (np. przez `engine.dispose()` albo po przekroczeniu czasu życia puli), kolejne żądanie dostaje **nowe połączenie P2** — a SQLite tworzy dla niego **nową, pustą bazę w pamięci**. Efekt: `OperationalError: no such table: books` losowo, w teście, który wcześniej przechodził.
>
> `StaticPool` rozwiązuje to brutalnie: jeden obiekt połączenia, wydawany każdemu, zawsze. Brak zamknięć, brak utraty danych.

> 🧠 **Dlaczego tak jest — `check_same_thread` to zabezpieczenie SQLite, nie SQLAlchemy.**
> Biblioteka `sqlite3` (standardowa) jest domyślnie skonfigurowana tak, że obiekt połączenia można używać **tylko w tym wątku, w którym go utworzono**. SQLAlchemy respektuje to i przekazuje parametr dalej. W testach zdarza się, że połączenie powstaje w jednym wątku (np. fixture), a używa go inny (np. wątek roboczy `TestClient`). Wyłączenie kontroli jest bezpieczne **wyłącznie** dlatego, że i tak wszystkie operacje idą przez jedno połączenie z `StaticPool`. Nie wyłączaj `check_same_thread` w kodzie produkcyjnym „na wszelki wypadek”.

> ⚠️ **Pułapka — SQLite w pamięci zniknie, gdy dasz się ponieść refaktorowi.**
> Najczęstszy scenariusz: wszystko działa, dopóki ktoś nie doda `poolclass=NullPool` „bo czytał, że to dobre dla testów”. `NullPool` tworzy nowe połączenie na każde żądanie i zamyka je natychmiast — czyli **każde zapytanie widzi inną, pustą bazę**. Testy przechodzą losowo albo padają na `no such table`. Jeśli musisz mieć `NullPool`, użyj bazy plikowej w `tmp_path`.

### Baza plikowa w `tmp_path` — gdy potrzebujesz wielu połączeń

`StaticPool` ma jedną wadę: **jest tylko jedno połączenie**, więc nie możesz mieć dwóch niezależnych, równoległych transakcji. A to jest potrzebne do testów współbieżności i blokad.

```python
# tests/conftest.py (fragment)
from pathlib import Path


@pytest.fixture
def file_engine(tmp_path: Path) -> Iterator[Engine]:
    db_file = tmp_path / "test.db"
    engine = create_engine(f"sqlite:///{db_file}")
    # Domyślny pool dla bazy plikowej to QueuePool — wiele równoległych połączeń,
    # więc dwie sesje mogą mieć dwie niezależne transakcje.
    Base.metadata.create_all(engine)
    yield engine
    engine.dispose()
```

`tmp_path` gwarantuje, że każdy test dostaje **własny, czysty katalog**, a plik bazy nie zanieczyszcza repozytorium. Po zakończeniu testu pytest kasuje katalog.

> ⚠️ **Pułapka — SQLite ma jedno miejsce do pisania.**
> Nawet z bazą plikową SQLite pozwala na **jednego piszącego** w danej chwili. Dwie sesje mogą czytać równolegle, ale zapis drugiej poczeka lub rzuci `database is locked`. Do testów prawdziwej współbieżności (wiele piszących) potrzebujesz PostgreSQL. Do testu blokady optymistycznej (sekwencyjne zapisy z odczytem „nieświeżych” danych) SQLite w zupełności wystarczy.

### PostgreSQL w Dockerze przez `testcontainers`

Testy na SQLite są szybkie, ale **nie są wiarygodne dla kodu produkcyjnego na PostgreSQL**. Różnice są realne i bolesne:

| Zjawisko | SQLite | PostgreSQL |
|---|---|---|
| `RETURNING` | wspierane w nowszych wersjach (≥ 3.35) | natywnie, wszędzie |
| Typy | dynamiczne (afinityzacja) | ścisłe |
| `FOR UPDATE` | brak / ignorowane | pełne wsparcie |
| Współbieżność | jeden piszący | MVCC, wielu piszących |
| Funkcje (`date_trunc`, `jsonb`, `ILIKE`) | brak / inne | pełne |
| `ON CONFLICT` | wspierane, inne warianty | pełne |
| `ALTER TABLE` | bardzo ograniczone | pełne |

Jeśli Twój kod używa `jsonb`, `ILIKE`, okien funkcyjnych z `PARTITION BY`, `FOR UPDATE` albo `CREATE INDEX CONCURRENTLY` — testowanie tylko na SQLite da Ci **fałszywe poczucie bezpieczeństwa**.

```python
# tests/test_postgres.py
import pytest
from sqlalchemy import Engine, create_engine
from sqlalchemy.orm import Session
from testcontainers.postgres import PostgresContainer

from tests.models import Base


@pytest.fixture(scope="session")
def postgres_engine() -> Engine:
    # PostgresContainer startuje prawdziwy serwer PostgreSQL w kontenerze Dockera.
    # Wymaga zainstalowanego i działającego Dockera na maszynie CI/deweloperskiej.
    with PostgresContainer("postgres:16-alpine") as postgres:
        url = postgres.get_connection_url().replace(
            "postgresql+psycopg2://", "postgresql+psycopg://"
        )
        engine = create_engine(url)
        Base.metadata.create_all(engine)
        yield engine
        engine.dispose()


@pytest.mark.integration
def test_jsonb_operators(postgres_engine: Engine) -> None:
    with Session(postgres_engine) as session:
        # tutaj testy zachowań specyficznych dla PostgreSQL
        ...
```

> 💡 **Analogia — SQLite to szkicownik, PostgreSQL to warsztat.**
> Świetnie ćwiczysz rysunek w szkicowniku: szybko, tanio, bez przygotowań. Ale zanim oddasz projekt na produkcję, musisz zobaczyć go w prawdziwym warsztacie — z materiałami, narzędziami i ograniczeniami, które w szkicowniku nie istnieją. Strategia optymalna: **SQLite dla wszystkich testów w pętli lokalnej + jedno uruchomienie na PostgreSQL w CI**. Dzięki temu masz i szybkość, i prawdę.

> ⚠️ **Pułapka — `testcontainers` w CI to kontener w kontenerze.**
> Jeśli Twoje CI samo działa w kontenerze (typowe dla GitHub Actions), potrzebujesz „Docker-in-Docker” albo sidecara z Dockerem. To bywa konfiguracyjną udręką. Alternatywa: `services` w GitHub Actions — uruchamiasz PostgreSQL jako *usługę* i podajesz URL do niej, bez `testcontainers`. Dla lokalnego developmentu `testcontainers` jest jednak bezkonkurencyjny.

> 🧪 **Ćwiczenie — w miejscu.**
> Uruchom `pytest --setup-show tests/test_podstawowe.py` i sprawdź w logu, ile razy wykonuje się `create_all`. Następnie zmień `scope="session"` na `scope="function"` i porównaj czas. Zapamiętaj różnicę — wrócimy do niej przy strategiach izolacji.

---

## Izolacja testów — trzy strategie

Testy muszą być **niezależne od kolejności** i **niezależne od siebie nawzajem**. Jeśli test `test_usun_ksiazke` przechodzi tylko wtedy, gdy wcześniej wykonano `test_dodaj_ksiazke`, to nie masz testów — masz loterię.

Istnieją trzy strategie izolacji. Każda ma inne koszty i inne zastosowania.

### Strategia A — `create_all` / `drop_all` per test

```python
# tests/conftest.py (wariant A)
@pytest.fixture
def db_session_a(engine: Engine) -> Iterator[Session]:
    with Session(engine) as session:
        yield session
    # po teście czyścimy cały schemat i tworzymy od nowa
    Base.metadata.drop_all(engine)
    Base.metadata.create_all(engine)
```

**Koszt:** wysoki (DDL jest drogie, zwłaszcza na PostgreSQL, gdzie rozbiórka schematu trwa dziesiątki milisekund).
**Zysk:** absolutna czystość i prostota.
**Kiedy:** gdy w teście zmieniasz schemat (rzadko) albo gdy testy są bardzo nieliczne.

### Strategia B — transakcja zewnętrzna + savepoint + rollback per test

To **standard branżowy** i strategia, którą recommendation SQLAlchemy opisuje w sekcji *„Joining a Session into an External Transaction”*.

```python
# tests/conftest.py (wariant B)
from sqlalchemy.orm import Session


@pytest.fixture
def db_session(engine: Engine) -> Iterator[Session]:
    # 1) Otwieramy połączenie i transakcję "zewnętrzną" (outer transaction).
    connection = engine.connect()
    transaction = connection.begin()

    # 2) Tworzymy sesję związaną z TYM połączeniem. Dzięki
    #    join_transaction_mode="create_savepoint" każde session.begin()
    #    (i automatyczny autobegin) tworzy SAVEPOINT wewnątrz transakcji
    #    zewnętrznej, a session.commit() zwalnia savepoint — nie kończy
    #    transakcji zewnętrznej.
    session = Session(bind=connection, join_transaction_mode="create_savepoint")

    try:
        yield session
    finally:
        # 3) Zamykamy sesję i wycofujemy CAŁĄ transakcję zewnętrzną.
        #    Wszystko, co test zapisał (nawet przez commit), znika.
        session.close()
        transaction.rollback()
        connection.close()
```

> 🧠 **Dlaczego `join_transaction_mode="create_savepoint"` robi różnicę.**
> Bez tego parametru `session.commit()` próbowałby zatwierdzić **transakcję zewnętrzną** — czyli połączenie przestałoby być w transakcji, a późniejszy `transaction.rollback()` rzuciłby `InvalidRequestError` albo po cichu nic nie zrobiłby (dane zostawałyby w bazie!).
>
> W SQLAlchemy 2.0 wprowadzono `join_transaction_mode`, które rozwiązuje ten problem deklaratywnie. Dostępne tryby:
> - `"rollback_only"` (domyślny) — sesja robi rollback zewnętrznej transakcji; commit przenosi kontrolę,
> - `"create_savepoint"` — **to jest tryb dla testów**: każda operacja sesji dzieje się w savepoincie,
> - `"control_fully"` — sesja przejmuje pełną kontrolę nad transakcją,
> - `"conditional_savepoint"` — savepoint tylko gdy sterownik go wspiera.

> 🔬 **Pod maską — jak wygląda SQL w strategii B.**
> Gdy test wykonuje `session.commit()`, SQLAlchemy emituje:
> ```sql
> -- zamiast COMMIT:
> RELEASE SAVEPOINT sa_savepoint_1
> ```
> A gdy test kończy się i fixture robi `transaction.rollback()`:
> ```sql
> ROLLBACK
> ```
> Baza cofa **wszystko** od początku, w tym zmiany, które test „zatwierdził”. Z punktu widzenia testu świat wygląda normalnie, ale na poziomie fizycznym nic nie zostało utrwalone.

**Koszt:** bardzo niski (jedno `BEGIN` + savepointy + jedno `ROLLBACK`).
**Zysk:** izolacja pełna, a sesja zachowuje się jak produkcyjna (commit, rollback, autoflush działają).
**Kiedy:** **domyślnie, zawsze**, o ile testy nie muszą widzieć commitu innych sesji.

> ⚠️ **Pułapka — strategia B nie działa, gdy potrzebujesz prawdziwego commitu.**
> Test blokady optymistycznej wymaga, żeby dwie sesje miały **osobne połączenia i osobne transakcje**, z prawdziwym commitem. W strategii B obie sesje siedzą w tej samej transakcji zewnętrznej, więc nie zobaczą nawzajem swoich zmian — i test będzie „przechodził”, mimo że nic nie sprawdza. Dla takich testów używaj osobnego fixture z bazą plikową (patrz `file_engine` wyżej).

### Strategia C — `TRUNCATE` / `DELETE` między testami

```python
# tests/conftest.py (wariant C)
from sqlalchemy import text


@pytest.fixture
def db_session_c(engine: Engine) -> Iterator[Session]:
    with Session(engine) as session:
        yield session
    # Czyścimy dane, ale zostawiamy schemat.
    with engine.begin() as connection:
        for table in reversed(Base.metadata.sorted_tables):
            connection.execute(table.delete())
```

Uwaga na dwa szczegóły:

- `Base.metadata.sorted_tables` zwraca tabele w kolejności **tworzenia** (odpowiednio posortowane zależnościami FK). Do usuwania potrzebujesz odwrotnej — stąd `reversed(...)`.
- `TRUNCATE` nie istnieje w SQLite. W PostgreSQL możesz użyć `TRUNCATE ... RESTART IDENTITY CASCADE`, co jest znacznie szybsze niż `DELETE` przy dużych tabelach.

**Koszt:** średni (rośnie z liczbą tabel i wierszy).
**Zysk:** działa z prawdziwymi commitami, wymaga tylko wspólnego schematu.
**Kiedy:** gdy testy robią prawdziwe commity, ale chcesz szybkości.

### Porównanie — kompletna tabela decyzyjna

| Kryterium | A: `create_all/drop_all` | B: savepoint + rollback | C: truncate |
|---|---|---|---|
| Szybkość | najwolniejsza | najszybsza | średnia |
| Prawdziwe commity widoczne dla innych sesji | ❌ | ❌ | ✅ |
| Zachowanie produkcyjne sesji | częściowe | ✅ pełne | ✅ pełne |
| Wymaga wspólnego schematu wcześniej | nie | tak | tak |
| Włącza wersjonowanie numerów `id` | ✅ reset | ✅ reset | tylko `RESTART IDENTITY` |
| Odporność na zmiany schematu w teście | ✅ | ❌ | ❌ |
| Ryzyko „przecieku” danych | brak | brak | przy błędzie fixture |

> ⚠️ **Pułapka — `create_all` na poziomie modułu zamiast w sesyjnym fixture.**
> Typowy błąd to wołanie `Base.metadata.create_all(engine)` na poziomie modułu `conftest.py`. Skutek: kod uruchamia się przy **importowaniu** pliku, a więc w momencie zbierania testów — niezależnie od tego, czy uruchamiasz testy w danym module. Do tego, jeśli równolegle działa `pytest-xdist`, wszystkie workery wykonają `create_all` na tej samej bazie, walcząc o DDL. Trzymaj całą inicjalizację **wewnątrz fixtures** (najlepiej `scope="session"`).

> 🆕 **SQLAlchemy 2.1 — mniej rund przy tworzeniu schematu.**
> W 2.1 `MetaData.create_all()` ma wewnętrznie zoptymalizowaną ścieżkę sprawdzania istnienia tabel (parametr nazwany `has_multi_table` w changelogu), co redukuje liczbę rund `SELECT` do bazy podczas inicjalizacji. Na PostgreSQL z 50 tabelami to realna oszczędność czasu w CI. W 2.0.x ten narzut wciąż istnieje.

> 🧪 **Ćwiczenie — w miejscu.**
> Zaimplementuj na swoim testowym schemacie wszystkie trzy strategie. Uruchom `pytest --durations=10`, żeby zobaczyć czasy. Zapisz różnicę w tabeli „strategia → czas” w komentarzu w `conftest.py`. To wiedza, która zostanie Ci na lata.

---

## Wersjonowanie schematu w testach

Skoro w module 16 nauczyłeś się pisać migracje Alembic, pojawia się pytanie: **czy testy powinny tworzyć schemat przez `create_all()` czy przez `alembic upgrade head`?**

### `create_all()` — szybko, mniej realistycznie

```python
Base.metadata.create_all(engine)
```

**Zalety:** błyskawiczne, nie wymaga Alembica, schemat zawsze zgodny z modelami (bo z nich powstaje).
**Wady:** nie testuje migracji, schemat może być inny niż w produkcji (np. gdy migracje mają ręczne poprawki, które nie znajdują się w modelach).

### Migracje Alembic — realistycznie, wolniej

```python
# tests/migration/conftest.py
import subprocess
from pathlib import Path

import pytest
from sqlalchemy import Engine, create_engine

from tests.models import Base  # tylko do porównania, nie do create_all


@pytest.fixture
def migrated_engine(tmp_path: Path, monkeypatch) -> Engine:
    db_file = tmp_path / "migrated.db"
    url = f"sqlite:///{db_file}"
    monkeypatch.setenv("DATABASE_URL", url)
    # Uruchamiamy migracje jak w prawdziwym wdrożeniu.
    subprocess.run(["alembic", "upgrade", "head"], check=True)
    engine = create_engine(url)
    yield engine
    engine.dispose()
```

**Zalety:** testuje to, co faktycznie uruchomi się na produkcji.
**Wady:** wolniejsze, wymaga katalogu `alembic/`.

### Strategia hybrydowa — rekomendowana

- **99% testów** używa `create_all()` (szybko).
- **Osobny, oznaczony markerem `slow` zestaw testów** uruchamia migracje i sprawdza:
  1. czy `alembic upgrade head` przechodzi na czystej bazie,
  2. czy `alembic downgrade -1` (i dalej do zera) przechodzi bez błędów,
  3. czy schemat po migracjach odpowiada modelom — to kluczowy test „rozjazdu”,
  4. czy migracje danych faktycznie przenoszą dane (dla migracji, które modyfikują zawartość, nie tylko strukturę).

Test zgodności schematu z modelami to złoty interes:

```python
# tests/migration/test_schema_drift.py
import pytest
from alembic.autogenerate import compare_metadata
from alembic.migration import MigrationContext
from sqlalchemy import Engine

from tests.models import Base

pytestmark = pytest.mark.slow


def test_brak_rozjazdu_schematu(migrated_engine: Engine) -> None:
    with migrated_engine.connect() as connection:
        context = MigrationContext.configure(connection)
        # compare_metadata porównuje stan bazy ze stanem modeli.
        # Pusta lista = brak różnic = schemat i modele są zgodne.
        diff = compare_metadata(context, Base.metadata)
        assert diff == [], f"Wykryto rozjazd schematu: {diff}"
```

Ten test uratuje Ci życie przy zmianie typu kolumny albo zapomnianej migracji — i to zanim kod trafi na produkcję.

> ⚠️ **Pułapka — `create_all()` obok Alembica w jednym projekcie.**
> Jeśli aplikacja uruchamia `create_all()` na starcie, a osobno używasz Alembica, dochodzi do rozjazdu: baza ma schemat z modeli, ale tabela `alembic_version` nie istnieje albo wskazuje starą rewizję. Efekt: `alembic upgrade head` próbuje dodać kolumnę, która już istnieje → `DuplicateColumnError`. Zasada: **na produkcji schemat tworzy wyłącznie Alembic; `create_all()` istnieje tylko na potrzeby testów.**

> 🆕 **SQLAlchemy 2.1 + Alembic 1.19 — nazwane ograniczenia CHECK.**
> Nowe wersje Alembica w autogenerate wykrywają **nazwane** ograniczenia `CHECK`. To istotne dla testu dryfu z powyższego przykładu: jeśli masz `CheckConstraint("rating BETWEEN 1 AND 5", name="ck_reviews_rating")`, autogenerate poprawnie zauważy różnicę między bazą a modelem. W starszych wersjach takie różnice bywały niewidoczne — dlatego zawsze nadawaj nazwy ograniczeniom (moduł 03, sekcja o `naming_convention`).

> 🧪 **Ćwiczenie — w miejscu.**
> Napisz test `test_migracja_danych_przenosi_wartosci`, który: (1) tworzy rekord w starej strukturze, (2) uruchamia konkretną rewizję z migracją danych, (3) sprawdza, że wartość trafiła do nowej kolumny. Uruchom go dwukrotnie — powinien być idempotentny.

---

## Fabryki danych

Test bez danych to test, który nic nie robi. Potrzebujesz sposobu na tworzenie obiektów w typowy, powtarzalny sposób.

### Ręczne funkcje fabryczne — prostota przede wszystkim

```python
# tests/factories.py
from datetime import datetime

from sqlalchemy.orm import Session

from tests.models import Author, Book, Category, Loan, Member, Review


def make_author(name: str = "Ursula K. Le Guin") -> Author:
    return Author(name=name)


def make_book(
    title: str = "Wyprawa",
    isbn: str = "978-0-000-00000-0",
    author: Author | None = None,
) -> Book:
    # Jeśli nie podano autora, tworzymy nowego z unikalną nazwą.
    # Dzięki temu dwa wywołania nie kolidują na UNIQUE(author.name).
    if author is None:
        author = make_author(name="Autor Testowy")
    return Book(title=title, isbn=isbn, author=author)


def make_member(email: str = "czytelnik@example.com") -> Member:
    return Member(name="Czytelnik Testowy", email=email)


def seed_library(session: Session, authors: int = 3, books_per_author: int = 2) -> list[Author]:
    """Buduje spójny 'świat' danych: N autorów, po M książek każdy."""
    result: list[Author] = []
    for i in range(authors):
        author = make_author(name=f"Autor {i}")
        for j in range(books_per_author):
            author.books.append(make_book(title=f"Ksiazka {i}-{j}", isbn=f"isbn-{i}-{j}"))
        session.add(author)
        result.append(author)
    session.flush()
    return result
```

> 🧠 **Dlaczego tak jest — nie wyliczaj ręcznie `id` ani nie licz na stałe wartości.**
> Kusi, żeby napisać `Book(id=1, ...)`. To pułapka: `id` zależy od kolejności testów, od tego, czy poprzedni test coś wstawił, i od użytej strategii izolacji (savepoint może cofać sekwencje inaczej niż truncate). Zamiast tego:
> - nie podawaj `id` — baza nada je sama,
> - pobieraj obiekty **przez zapytanie** (`session.scalars(select(Book).where(Book.isbn == "isbn-0-0")).one()`),
> - przy wartościach `UNIQUE` (e-mail, ISBN) twórz je parametrycznie lub z licznikiem, żeby uniknąć kolizji.

### `factory_boy` — gdy fabryki stają się złożone

```python
# tests/factories_fb.py
import factory
from factory.alchemy import SQLAlchemyModelFactory

from tests.models import Author, Book


class AuthorFactory(SQLAlchemyModelFactory):
    class Meta:
        model = Author
        sqlalchemy_session_persistence = "flush"  # nie commituj, tylko flush!

    name = factory.Sequence(lambda n: f"Autor {n}")


class BookFactory(SQLAlchemyModelFactory):
    class Meta:
        model = Book
        sqlalchemy_session_persistence = "flush"

    title = factory.Faker("sentence", nb_words=3)
    isbn = factory.Sequence(lambda n: f"978-0-000-{n:05d}")
    author = factory.SubFactory(AuthorFactory)
```

Aby fabryki wiedziały, której sesji użyć, podłącza się je do fixtures w `conftest.py` (wzorzec kanoniczny z dokumentacji `factory_boy`):

```python
# tests/conftest.py (fragment)
import pytest
from factory.alchemy import SQLAlchemyModelFactory

from tests.factories_fb import AuthorFactory, BookFactory


@pytest.fixture(autouse=True)
def _podlacz_fabryki(db_session):
    # `autouse=True` sprawia, że fixture wykonuje się dla każdego testu
    # automatycznie — nie musisz dopisywać jej do sygnatur.
    for fabryka in (AuthorFactory, BookFactory):
        fabryka._meta.sqlalchemy_session = db_session
    yield
```

`sqlalchemy_session_persistence = "flush"` jest kluczowe: fabryka dodaje obiekt do sesji i robi `flush`, ale **nie commituje** — więc strategia izolacji B działa dalej.

### `polyfactory` — fabryki z adnotacji typów

Dla modeli dataclass-based (albo `MappedAsDataclass`) `polyfactory` potrafi wygenerować fabrykę automatycznie z typów:

```python
# tests/factories_pf.py
from polyfactory.factories.sqlalchemy_factory import SQLAlchemyFactory

from tests.models import Member


class MemberFactory(SQLAlchemyFactory[Member]):
    __model__ = Member
    __set_relationships__ = False
    # Wymuszamy deterministyczne wartości tam, gdzie chcemy je kontrolować.
    email = "unikalny@example.com"
```

> 💡 **Analogia — fabryka to foremka do ciastek.**
> Ciasto (klasa) jest jedno, ale foremka (fabryka) pozwala wyciąć z niego ciastko w sekundę, z domyślnym nadzieniem, które możesz zmienić dla konkretnego egzemplarza: `make_book(title="Inny tytuł")`. Bez foremki w każdym teście wyrabiałbyś ciasto od nowa.

> ⚠️ **Pułapka — „świat danych” w `conftest` na poziomie sesji.**
> Kusząca optymalizacja: utwórz jednego, wspólnego „świata” danych dla wszystkich testów (`scope="session"`), żeby nie powtarzać `seed_library` w każdym teście. Problem: testy, które modyfikują dane, psują świat innym testom — a kolejność zależy od tego, jak pytest posortował pliki. Zasada: **świat danych per test**. Jeśli tworzenie go jest drogie, skróć go do minimum potrzebnego danemu testowi.

> 🧪 **Ćwiczenie — w miejscu.**
> Napisz funkcję `make_loan(book, member)`, która: (a) tworzy wypożyczenie, (b) ustawia `borrowed_at` na stałą datę za pomocą parametru, (c) pozostawia `returned_at = None`. Użyj jej w dwóch testach i sprawdź, czy nie kolidują.

---

## Testowanie zachowań specyficznych dla SQLAlchemy

To serce modułu. Zachowania ORM — kaskady, walidatory, zdarzenia, blokady, transakcje — to Twoja własna konfiguracja, więc to **Twoja** odpowiedzialność, żeby były przetestowane.

### Kaskady i usuwanie

Model z modułu 09 ma `Book.reviews` z `cascade="all, delete-orphan"`. Co to znaczy „sierota” (orphan)? Recenzja, która straciła swoją książkę.

```python
# tests/test_models.py
from sqlalchemy import select

from tests.models import Book, Review
from tests.factories import make_author, make_book


def test_delete_orphan_usuwie_recenzje_po_odlaczeniu(db_session) -> None:
    book = make_book(author=make_author())
    book.reviews.append(Review(rating=5, comment="Świetna"))
    db_session.add(book)
    db_session.flush()

    assert db_session.scalars(select(Review)).all() != []

    # Usuwamy recenzję z kolekcji — nie wołamy session.delete()!
    book.reviews.clear()
    db_session.flush()

    # Dzięki delete-orphan recenzja zostaje usunięta z bazy.
    assert db_session.scalars(select(Review)).all() == []
```

> 🔬 **Pod maską — co SQLAlchemy wyemituje po `book.reviews.clear()` + `flush()`.**
> ```sql
> DELETE FROM reviews WHERE reviews.id = ?
> ```
> Bez `delete-orphan` (samo `cascade="all"`) wiersz w `reviews` **zostałby**, tylko `book_id` wskazywałoby na nieistniejącą książkę albo SQLAlchemy ustawiłoby je na `NULL` — o ile kolumna jest nullable. To klasyczne źródło „duchów” w bazie.

Testujemy też kaskadę przy usunięciu encji nadrzędnej:

```python
def test_usuniecie_ksiazki_usuwa_recenzje(db_session) -> None:
    book = make_book(author=make_author())
    book.reviews.append(Review(rating=3, comment="Przeciętna"))
    db_session.add(book)
    db_session.flush()
    review_id = book.reviews[0].id

    db_session.delete(book)
    db_session.flush()

    assert db_session.get(Review, review_id) is None
```

> ⚠️ **Pułapka — kaskada w Pythonie nie jest tym samym co `ondelete="CASCADE"`.**
> W modelu masz i `cascade="all, delete-orphan"` (kaskada ORM, w Pythonie), i `ondelete="CASCADE"` (kaskada w bazie). Test powyżej przechodzi dzięki kaskadzie ORM — SQLAlchemy sam wyśle `DELETE FROM reviews`. Ale jeśli ktoś usunie książkę **bezpośrednio w SQL-u** (`DELETE FROM books WHERE id=1`), zadziała wyłącznie `ondelete` z bazy. Testuj **oba** scenariusze:
> - kaskada ORM: przez `session.delete(book)`,
> - kaskada w bazie: przez `connection.execute(delete(Book).where(Book.id == book_id))`.
> To dwie różne ścieżki i obie muszą działać.

### Walidatory `@validates`

Walidator działa **w momencie przypisania wartości do atrybutu** — niezależnie od tego, czy obiekt jest w sesji. To ważna cecha: test nie potrzebuje bazy.

```python
# tests/test_walidacja.py
import pytest

from tests.models import Member


def test_walidator_normalizuje_email(db_session) -> None:
    member = Member(name="Ala", email="  ALA@Example.COM ")
    assert member.email == "ala@example.com"

    db_session.add(member)
    db_session.flush()
    assert member.email == "ala@example.com"


def test_walidator_odrzuca_email_bez_malpy() -> None:
    # Walidator działa natychmiast przy tworzeniu obiektu — baza nie jest potrzebna.
    with pytest.raises(ValueError, match="e-mail"):
        Member(name="Bob", email="to-nie-jest-email")
```

> ⚠️ **Pułapka — `@validates` NIE działa przy bulk update.**
> ```python
> session.execute(update(Member).where(Member.id == 1).values(email="ZLY"))
> ```
> Ten kod **ominie** walidator, bo SQLAlchemy w ogóle nie tworzy obiektu `Member` — wysyła SQL bezpośrednio. Jeśli dane muszą być zawsze poprawne, niezależnie od ścieżki zapisu, walidacja musi też istnieć w bazie (`CHECK`, trigger) albo w warstwie serwisowej. Test na to jest bezcenny:
> ```python
> def test_bulk_update_omija_walidator(db_session) -> None:
>     member = Member(name="Ala", email="ala@example.com")
>     db_session.add(member)
>     db_session.flush()
>     # Ten zapis przejdzie, mimo że walidator by go odrzucił.
>     db_session.execute(
>         update(Member).where(Member.id == member.id).values(email="bez-malpy")
>     )
>     db_session.flush()
>     assert db_session.get(Member, member.id).email == "bez-malpy"
> ```
> Ten test nie jest „testem błędu” — jest **dokumentacją architektury**. Pokazuje przyszłemu programiście, gdzie walidacja NIE działa, i zmusza do decyzji: gdzie umieścić pilnowanie niezmienników.

### Zdarzenia `before_flush` (audyt)

Powiązane zdarzenie sesji (`Session`) uruchamia się przed wysłaniem zmian do bazy. Typowe zastosowanie: audyt „kto i kiedy”.

```python
# tests/test_audit.py
from sqlalchemy import event
from sqlalchemy.orm import Session

from tests.factories import make_author, make_book


def test_before_flush_zanotowal_zmiane(db_session, monkeypatch) -> None:
    # Rejestrujemy listener dynamicznie, tylko na czas tego testu.
    notatki: list[str] = []

    def zapisz(session: Session, flush_context, instances) -> None:
        for obj in session.dirty:
            notatki.append(f"UPDATE {type(obj).__name__}#{obj.id}")

    event.listen(db_session, "before_flush", zapisz)
    try:
        book = make_book(author=make_author())
        db_session.add(book)
        db_session.flush()

        book.title = "Nowy tytuł"
        db_session.flush()

        assert any("UPDATE Book" in n for n in notatki)
    finally:
        # KLUCZOWE: usuwamy listener, żeby nie "przeciekł" do innych testów.
        event.remove(db_session, "before_flush", zapisz)
```

> 🧠 **Dlaczego tak jest — dlaczego akurat `before_flush`.**
> Masz trzy bliskie zdarzenia: `before_flush` (obiekty już zaktualizowane w pamięci, SQL jeszcze nie wysłany), `after_flush` (SQL wysłany, ale transakcja otwarta), `after_flush_postexec` (po wykonaniu wszystkich poleceń, w tym kaskad). Dla audytu najczęściej chcesz `before_flush`, bo tam widzisz **intencję** i możesz jeszcze zmodyfikować obiekty (np. dopisać wiersz audytu), a sesja doda go do tej samej transakcji.

> ⚠️ **Pułapka — listener zarejestrowany globalnie przecieka między testami.**
> `event.listen(SomeClass, "before_flush", ...)` działa **na klasie**, nie na instancji sesji. Zarejestrowany raz, zostaje na zawsze w całym procesie pytest. Jeśli zrobisz to w teście, wszystkie kolejne testy (i ich fixtures) będą miały aktywny listener — z nieprzewidywalnymi skutkami. Reguły bezpieczeństwa:
> 1. Rejestruj listener na **konkretnej instancji sesji** (`event.listen(db_session, ...)`), nie na klasie.
> 2. Zawsze zdejmuj go w `finally` albo w fixture ze `yield`.
> 3. Jeśli listener jest produktowy (np. audyt w aplikacji), rejestruj go raz przy starcie aplikacji i napisz test, który sprawdza, że **nie zarejestrowano go dwukrotnie**.

### Transakcje i wyjątki

```python
# tests/test_transakcje.py
import pytest
from sqlalchemy import select
from sqlalchemy.exc import IntegrityError

from tests.factories import make_author, make_book
from tests.models import Book


def test_duplikat_isbn_rzuca_integrity_error(db_session) -> None:
    autor = make_author()
    db_session.add(make_book(author=autor, isbn="978-DUP"))
    db_session.flush()

    with pytest.raises(IntegrityError, match="UNIQUE"):
        db_session.add(make_book(author=make_author(name="Inny"), isbn="978-DUP"))
        db_session.flush()

    # Po IntegrityError sesja wymaga rollbacku — inaczej kolejny flush
    # rzuci PendingRollbackError.
    db_session.rollback()
    assert db_session.scalars(select(Book).where(Book.isbn == "978-DUP")).one()
```

> 💡 **Analogia — po błędzie transakcji sesja jest jak rozbite lustro.**
> Możesz je jeszcze trzymać w rękach, ale nie zobaczysz w nim nic sensownego. Po `IntegrityError` (i każdym innym błędzie w transakcji) sesja przechodzi w stan, w którym musi nastąpić `rollback()`, zanim wykonasz kolejną operację. To nie fanaberia SQLAlchemy — tak działa SQL: transakcja, która napotkała błąd, jest „zatruta” i nie może kontynuować w PostgreSQL.

### Blokada optymistyczna i `StaleDataError`

Blokada optymistyczna (moduł 14) polega na tym, że każdy wiersz ma numer wersji, a każda aktualizacja podnosi go o 1 i sprawdza, czy wersja się zgadza. Jeśli nie — ktoś zdążył zmienić wiersz w międzyczasie.

```python
# tests/test_optimistic_lock.py
from datetime import datetime

import pytest
from sqlalchemy import select
from sqlalchemy.exc import StaleDataError
from sqlalchemy.orm import sessionmaker

from tests.models import Author, Book, Loan, Member


def test_blokada_optymistyczna_wykrywa_konflikt(file_engine) -> None:
    factory = sessionmaker(bind=file_engine)

    # Przygotowanie danych — wymaga prawdziwego commitu, bo dwie sesje
    # muszą zobaczyć ten sam wiersz w osobnych transakcjach.
    with factory.begin() as setup:
        book = Book(title="Solaris", isbn="978-1", author=Author(name="Stanisław Lem"))
        member = Member(name="Jan", email="jan@example.com")
        setup.add(Loan(book=book, member=member))

    with factory() as session_a, factory() as session_b:
        loan_a = session_a.scalars(select(Loan)).one()
        loan_b = session_b.scalars(select(Loan)).one()
        assert loan_a.version == 1

        # Sesja B zatwierdza jako pierwsza: version 1 -> 2.
        loan_b.returned_at = datetime(2026, 9, 20, 12, 0)
        session_b.commit()

        # Sesja A próbuje zapisać na podstawie nieaktualnej wersji.
        loan_a.returned_at = datetime(2026, 9, 20, 13, 0)
        with pytest.raises(StaleDataError):
            session_a.commit()
```

> 🔬 **Pod maską — SQL, który wywołał `StaleDataError`.**
> ```sql
> UPDATE loans
>    SET returned_at = ?, version = 2
>  WHERE loans.id = ? AND loans.version = 1
> -- rowcount = 0 -> SQLAlchemy podnosi StaleDataError
> ```
> Zwróć uwagę: warunek `version = 1` to serce mechanizmu. Baza w tym momencie ma `version = 2`, więc zapytanie nie modyfikuje **żadnego** wiersza. SQLAlchemy wykrywa `rowcount == 0` i rzuca wyjątek. Bez `version_id_col` ten sam `UPDATE` po prostu nadpisałby zmianę z sesji B — i nikt by się o tym nie dowiedział.

> 🆕 **SQLAlchemy 2.1 — greenlet nie instaluje się automatycznie.**
> Jeśli używasz `AsyncSession`, w 2.1 musisz jawnie zainstalować `pip install "sqlalchemy[asyncio]"`, bo `greenlet` przestał być instalowany jako zależność. To wpływa na konfigurację środowiska testowego CI: jeśli testy async przechodziły na 2.0, a po podbiciu do 2.1 nagle lecą z `MissingGreenlet`, sprawdź najpierw, czy `greenlet` jest w zależnościach.

---

## Testowanie zapytań i liczby zapytań

To sekcja, która zamienia „testy funkcjonalne” w **testy regresji wydajnościowej**. Sam wynik zapytania to za mało — zapytanie może być poprawne, a mimo to zabijać aplikację.

### Fixture licząca zapytania

```python
# tests/conftest.py (fragment)
from collections.abc import Iterator

from sqlalchemy import Engine, event


@pytest.fixture
def query_counter(engine: Engine) -> Iterator[dict[str, int]]:
    """Zlicza polecenia SQL wykonywane na silniku w trakcie testu."""
    licznik = {"n": 0}

    def policz(conn, cursor, statement, parameters, context, executemany) -> None:
        licznik["n"] += 1

    event.listen(engine, "before_cursor_execute", policz)
    try:
        yield licznik
    finally:
        event.remove(engine, "before_cursor_execute", policz)
```

Teraz każdy test może zażądać `query_counter` i zrobić z niego asercję:

```python
# tests/test_zapytania.py
from sqlalchemy import select
from sqlalchemy.orm import selectinload

from tests.factories import seed_library
from tests.models import Author


def test_selectinload_nie_ma_n_plus_1(db_session, query_counter) -> None:
    seed_library(db_session, authors=5, books_per_author=2)
    db_session.flush()
    query_counter["n"] = 0  # zerujemy po przygotowaniu danych

    stmt = select(Author).options(selectinload(Author.books))
    autorzy = db_session.scalars(stmt).all()
    assert len(autorzy) == 5
    assert sum(len(a.books) for a in autorzy) == 10

    # Dokładnie 2 zapytania: autorzy + książki przez IN (...)
    assert query_counter["n"] == 2


def test_lazy_loading_w_petli_powoduje_n_plus_1(db_session, query_counter) -> None:
    seed_library(db_session, authors=5, books_per_author=2)
    db_session.flush()
    query_counter["n"] = 0

    autorzy = db_session.scalars(select(Author)).all()
    # Dotknięcie relacji w pętli uruchamia osobne zapytanie na każdego autora.
    for autor in autorzy:
        _ = autor.books

    # 1 (autorzy) + 5 (po jednym na autora) = 6
    assert query_counter["n"] == 6
```

> 🧠 **Dlaczego tak jest — asercja na liczbę zapytań to najtańsza ochrona przed regresją wydajnościową.**
> Test funkcjonalny `assert len(autorzy) == 5` przejdzie zarówno dla wersji z `selectinload`, jak i bez niego. Test na liczbę zapytań — nie. Jeśli w przyszłości ktoś usunie `selectinload` z jednego zapytania w kodzie, ten test natychmiast to pokaże. Liczba zapytań jest **kontraktem** równie ważnym jak zwracane wartości.

> 🔬 **Pod maską — SQL dla obu strategii.**
> ```sql
> -- selectinload: dwa zapytania
> SELECT authors.id, authors.name FROM authors;
> SELECT books.id, books.title, books.isbn, books.author_id
>   FROM books WHERE books.author_id IN (?, ?, ?, ?, ?);
>
> -- lazy loading w pętli: 1 + N zapytań
> SELECT authors.id, authors.name FROM authors;
> SELECT books.id, books.title, ... FROM books WHERE books.author_id = ?;  -- powtarzane N razy
> ```

### `raiseload("*")` — bezpiecznik domyślny w testach

Zdarza się, że zapominasz o eager loadingu w jednym z 20 endpointów i problem pojawia się dopiero na produkcji, przy 10 000 rekordów. Możesz temu zapobiec, ustawiając w testach **zakaz** leniwego ładowania.

```python
# tests/test_raiseload.py
import pytest
from sqlalchemy import select
from sqlalchemy.exc import InvalidRequestError
from sqlalchemy.orm import raiseload

from tests.factories import seed_library
from tests.models import Author


def test_raiseload_blokuje_przypadkowe_lazy_loading(db_session) -> None:
    seed_library(db_session, authors=2, books_per_author=1)
    db_session.flush()

    author = db_session.scalars(
        select(Author).options(raiseload(Author.books))
    ).one()

    # Dotknięcie relacji, której nie załadowano jawnie, rzuci wyjątek —
    # zamiast po cichu wysłać zapytanie.
    with pytest.raises(InvalidRequestError, match="lazy='raise'"):
        _ = author.books
```

Możesz to zrobić jeszcze bardziej rygorystycznie — globalnie dla testów:

```python
# tests/conftest.py (fragment)
@pytest.fixture
def session_bez_lazy(db_session, engine):
    # Ustawia raiseload("*") jako domyślną politykę ładowania dla całej sesji.
    from sqlalchemy.orm import raiseload

    db_session.execute(select(1).options(raiseload("*")))
    yield db_session
```

> ⚠️ **Pułapka — `raiseload("*")` domyślnie w testach frustruje na początku.**
> Pierwsze uruchomienie zestawu testów z `raiseload("*")` zwykle wywala połowę z nich — bo kod faktycznie polegał na leniwym ładowaniu. To jest **dokładnie ta wartość**: testy pokazują, gdzie Twoja aplikacja wysyła zapytania `N+1`, zanim zrobi to w produkcji. Wprowadzaj `raiseload` stopniowo: najpierw na testy nowych endpointów, potem na całość.

> 🧪 **Ćwiczenie — w miejscu.**
> Zbuduj test, który tworzy fixture `query_counter` i sprawdza, że zapytanie o `Member` z jego `loans` z `selectinload` generuje **2** zapytania. Następnie usuń `selectinload` i zobacz, jak test zmienia wynik na `1 + N` — bez zmiany wyniku funkcjonalnego zapytania.

---

## Testy asynchroniczne

Testy `AsyncSession` mają jedno specyficzne wyzwanie: **pętla zdarzeń** (`event loop`).

### Konfiguracja

```bash
pip install pytest-asyncio aiosqlite "sqlalchemy[asyncio]"
```

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
```

Tryb `"auto"` sprawia, że:

- funkcje `async def test_*` są automatycznie testami asynchronicznymi,
- fixtures `async def` są automatycznie obsługiwane — nie musisz dekorować ich `@pytest_asyncio.fixture`.

### Fixtures dla `AsyncSession`

```python
# tests/conftest.py (fragment)
import pytest
import pytest_asyncio
from sqlalchemy.ext.asyncio import (
    AsyncEngine,
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)
from sqlalchemy.pool import StaticPool

from tests.models import Base


@pytest_asyncio.fixture
async def async_engine() -> AsyncEngine:
    engine = create_async_engine(
        # Dla async SQLite potrzebny jest sterownik aiosqlite — "sqlite+aiosqlite".
        "sqlite+aiosqlite://",
        # StaticPool ma tu dokładnie to samo znaczenie co w wersji sync:
        # bez niego każde nowe połączenie dostałoby PUSTĄ bazę w pamięci.
        poolclass=StaticPool,
    )
    async with engine.begin() as connection:
        # DDL jest synchroniczne, więc uruchamiamy je przez run_sync.
        await connection.run_sync(Base.metadata.create_all)
    yield engine
    await engine.dispose()


@pytest_asyncio.fixture
async def async_session(async_engine: AsyncEngine) -> AsyncSession:
    factory = async_sessionmaker(async_engine, expire_on_commit=False)
    async with factory() as session:
        yield session
```

> 🧠 **Dlaczego tak jest — `expire_on_commit=False` jest tu niemal obowiązkowe.**
> W trybie async nie możesz uruchomić **nieświadomego** IO. Gdy `expire_on_commit=True` (domyślne), po `commit()` wszystkie atrybuty obiektu zostają wygaszone — a Twój następny `book.title` próbuje dociągnąć dane z bazy. W trybie sync to działa (ukryte zapytanie), w async rzuca `MissingGreenlet`. W `AsyncSession` domyślnie ustawia się więc `expire_on_commit=False`. Zapamiętaj tę różnicę — w testach async powinieneś jawnie ją ustawić, żeby nie polegać na niejawnej konfiguracji.

### Test asynchroniczny

```python
# tests/test_async.py
import pytest
from sqlalchemy import select

from tests.factories import seed_library
from tests.models import Author


@pytest.mark.asyncio
async def test_async_repozytorium_zwraca_autorow(async_session) -> None:
    seed_library(async_session, authors=2)
    await async_session.commit()

    result = await async_session.scalars(select(Author))
    autorzy = result.all()
    assert len(autorzy) == 2
```

### „Session is bound to a different event loop”

> ⚠️ **Pułapka — sesja stworzona w jednej pętli, użyta w innej.**
> To najczęstszy błąd w testach async. Wygląda mniej więcej tak:
>
> ```
> RuntimeError: Task got Future attached to a different loop
> ```
>
> Przyczyna: fixture o `scope="session"` (jedna na cały przebieg pytest) tworzy `AsyncEngine` i sesję w **jednej** pętli zdarzeń. pytest-asyncio domyślnie używa **innej** pętli dla każdego testu (albo dla każdej funkcji). Obiekty `asyncio` są przywiązane do konkretnej pętli — nie da się ich przenieść.
>
> Rozwiązania:
> 1. **Najprostsze:** wszystkie fixtures async mają `scope="function"` — jedna pętla na test, wszystko spójne.
> 2. **Szybsze:** ustaw `loop_scope` fixture na `"session"` (pytest-asyncio ≥ 0.24 udostępnia `pytest_asyncio.fixture(loop_scope="session")` oraz konfigurację `asyncio_default_fixture_loop_scope = "session"` w `pyproject.toml`) i utrzymuj jedną pętlę dla całej sesji testowej. Uwaga: wtedy musisz też oznaczyć testy `@pytest.mark.asyncio(loop_scope="session")`.
> 3. **Alternatywa:** użyj `anyio` jako backendu — `pytest-anyio` ma podobny mechanizm zakresu pętli.

> ⚠️ **Pułapka — `await` na `commit()` bez `rollback()` w `finally`.**
> Jeśli test asynchroniczny rzuci wyjątek w połowie transakcji, sesja zostaje w stanie wymagającym rollbacku. W sync to mniej bolesne, w async potrafi prowadzić do niezrozumiałego `MissingGreenlet` w kolejnej operacji. Wzorzec:
>
> ```python
> try:
>     await session.commit()
> except Exception:
>     await session.rollback()
>     raise
> ```

> 🆕 **SQLAlchemy 2.1 — autoflush w `Session` działa bezwarunkowo.**
> W 2.1 autoflush działa **bezwarunkowo** — SQLAlchemy zrezygnowało z pewnych warunków pomijania autoflush, które istniały wcześniej. Praktyczna konsekwencja dla testów: jeśli opierałeś się na tym, że w określonych sytuacjach SQL *nie* był wysyłany przed zapytaniem, w 2.1 możesz zobaczyć **więcej** zapytań, niż oczekiwałeś. Asercje na liczbę zapytań (`assert query_counter["n"] == 2`) są odporne na tę zmianę tylko wtedy, gdy nie polegają na niejawnym braku flushu — a więc testuj intencję, nie artefakt.

> 🧪 **Ćwiczenie — w miejscu.**
> Skonfiguruj fixtures async w `scope="function"` i policz, ile testów przechodzi. Następnie zmień na `loop_scope="session"` (z odpowiednim markerem) i porównaj czas. Zapisz obie liczby.

---

## Mockowanie — kiedy pomaga, kiedy szkodzi

Mockowanie (podstawianie obiektu atrapą) to potężne narzędzie, które bardzo łatwo zamienić w antywzorzec.

> 💡 **Analogia — mock to dubler aktora.**
> Jeśli scena wymaga, żeby aktor skoczył z dachu, zatrudniasz kaskadera. Nie zatrudniasz kaskadera do **grania głównej roli**. Mockowanie ma sens na **granicy systemu** (płatności, e-mail, zewnętrzne API), gdzie prawdziwa realizacja jest niedostępna, droga albo niedeterministyczna. Nie ma sensu w środku Twojej logiki.

### Kiedy mockowanie jest OK

```python
# tests/test_mock_ok.py
from unittest.mock import MagicMock


def test_wysylka_emaila_po_rejestracji(monkeypatch) -> None:
    # Klient SMTP to granica systemu — nie chcemy wysyłać prawdziwych e-maili.
    fake_smtp = MagicMock()
    monkeypatch.setattr("app.services.email.smtp_client", fake_smtp)

    # ... wywołanie serwisu rejestracji ...

    fake_smtp.send.assert_called_once()
```

Dobre kandydatury do mockowania:

- klienty zewnętrznych API (płatności, wysyłka e-mail, SMS),
- systemy plików (choć tu lepszy jest `tmp_path`),
- zegar (choć tu lepszy jest `freezegun`),
- funkcje losujące (choć tu lepszy jest wstrzyknięty `random.Random(seed)`),
- operacje niedeterministyczne (UUID, adresy IP).

### Kiedy mockowanie jest antywzorcem

```python
# ❌ ANTYWZORZEC — mockujemy Session
def test_repozytorium_z_mockiem():
    session = MagicMock()
    session.scalars.return_value.all.return_value = [MagicMock()]
    repo = BookRepository(session)
    result = repo.list_active()
    # Ten test sprawdza... co? Że repo woła scalars()? Nic więcej.
```

Dlaczego to antywzorzec:

1. **Testujesz mockę, nie kod.** Sprawdzasz, że `scalars` zostało wywołane — ale nie, czy SQL jest poprawny, czy relacje działają, czy wyszły właściwe dane.
2. **Mock nie ma logiki.** Prawdziwy `Session` robi identity map, autoflush, kaskady, mapowanie typów. Mock tego nie robi — więc test „przechodzi” nawet gdy kod jest zepsuty.
3. **Zmiana w kodzie = zmiana w mocku.** Za każdym razem, gdy zmienisz implementację, musisz poprawić mock. To koszt bez wartości.

> ⚠️ **Pułapka — kaskadowe atrapy (`MagicMock().foo().bar().baz()`).**
> Magia `MagicMock` polega na tym, że **każdy** atrybut istnieje i **każde** wywołanie działa. To sprawia, że literówka w nazwie metody przechodzi niezauważona:
>
> ```python
> session.scalrs(...)   # literówka! Prawdziwy Session rzuciłby AttributeError.
> ```
> W mocku to działa. Dlatego **nigdy** nie mockuj `Session`. Zamiast tego użyj prawdziwej bazy w pamięci — jest równie szybka, a daje prawdziwe zachowania.

> 🧠 **Dlaczego tak jest — realna baza w pamięci jest lepsza od mocka niemal zawsze.**
> Mock jest szybki. SQLite w pamięci jest **równie szybki** (rzędu milisekund), a daje Ci: prawdziwe ograniczenia (UNIQUE, FK), prawdziwe transakcje, prawdziwe kaskady, prawdziwe mapowanie typów. Jedyna sytuacja, w której mock jest lepszy, to granica zewnętrzna (patrz wyżej) — i tam zwykle chodzi nie o szybkość, a o niedostępność.

> 🧪 **Ćwiczenie — w miejscu.**
> Znajdź w swoich testach (albo w kodzie z poprzednich modułów) jedno miejsce, gdzie mockujesz coś wewnętrznego. Spróbuj zastąpić to testem integracyjnym na SQLite. Zauważ, ile dodatkowych błędów prawdziwa baza potrafi złapać.

---

## Dane testowe a produkcja

Testy powinny być **deterministyczne** — ten sam kod ma dawać ten sam wynik zawsze, na każdej maszynie.

### Czas — `freezegun`

```python
# tests/test_czas.py
from datetime import datetime

from freezegun import freeze_time


@freeze_time("2026-09-20 12:00:00")
def test_wypozyczenie_ma_proper_due_date(db_session) -> None:
    # Każde wywołanie datetime.now() w kodzie zwróci zamrożoną datę.
    loan = Loan(book=book, member=member)
    assert loan.borrowed_at == datetime(2026, 9, 20, 12, 0)
```

Alternatywa dla `freezegun` to wstrzykiwanie „zegara” jako zależności:

```python
# app/clock.py
from datetime import datetime
from typing import Protocol


class Clock(Protocol):
    def now(self) -> datetime: ...


class SystemClock:
    def now(self) -> datetime:
        return datetime.now()


class FrozenClock:
    def __init__(self, moment: datetime) -> None:
        self._moment = moment

    def now(self) -> datetime:
        return self._moment
```

Wstrzykiwanie jest bardziej inwazyjne, ale działa też w teście async i nie wymaga „magii” podmiany globalnych funkcji.

### Losowość — deterministyczne `uuid` i `random`

```python
# tests/test_losowosc.py
import uuid


def test_deterministyczny_uuid(monkeypatch) -> None:
    # Generujemy przewidywalne UUID-y, żeby testy dały się odtworzyć.
    licznik = iter(range(1000))

    def fake_uuid4() -> uuid.UUID:
        return uuid.UUID(int=next(licznik))

    monkeypatch.setattr(uuid, "uuid4", fake_uuid4)
    assert str(uuid.uuid4()) == "00000000-0000-0000-0000-000000000000"
```

### Anonimizacja danych produkcyjnych

Bywa, że chcesz testować na realistycznych danych. Wtedy:

1. weź kopię bazy produkcyjnej,
2. **anonimizuj** dane osobowe (e-maile, nazwiska, telefony, adresy, numery PESEL),
3. zapisz kopię w oddzielnej, izolowanej bazie (nigdy na produkcji!),
4. używaj jej jako fixture.

Anonimizacji nie robi się przez `UPDATE users SET email = 'x'` — bo zostawiasz korelacje (dwóch użytkowników z tym samym e-mailem wygląda inaczej po prostym nadpisaniu). Użyj biblioteki albo funkcji odwracalnej mapy (`hashlib.sha256` + sól).

> ⚠️ **Pułapka — dane produkcyjne w repozytorium.**
> Najczęstszy wyciek danych w małych projektach: ktoś eksportuje z bazy `.csv` do `tests/fixtures/` i commituje. Nawet jeśli dane „wyglądają na testowe”, zawsze sprawdź, czy nie ma tam prawdziwych e-maili, numerów, adresów. Reguła: `git grep` pod kątem `@` w katalogu `tests/fixtures` przed każdym `git push`.

> 🧪 **Ćwiczenie — w miejscu.**
> Sprawdź, czy jakikolwiek test w Twoim projekcie zależy od aktualnej godziny (`datetime.now()` bez `freeze_time`). Jeśli tak — zamroź czas i zobacz, czy testy nadal przechodzą. Test zależny od zegara to test, który padnie w nocy.

---

## Struktura testów w projekcie

Dobra struktura katalogów testów komunikuje intencję lepiej niż komentarze.

```text
tests/
├── conftest.py                 # fixtures wspólne: engine, db_session, query_counter
├── factories.py                # ręczne funkcje fabryczne
├── models.py                   # modele wspólne dla testów
├── unit/
│   ├── conftest.py             # fixtures dla testów bez bazy
│   ├── test_validators.py      # czysta logika walidatorów
│   ├── test_type_decorator.py  # process_bind_param / process_result_value
│   └── test_services_pure.py   # logika bez IO
├── integration/
│   ├── conftest.py             # fixtures z bazą (file_engine, migrated_engine)
│   ├── test_models.py          # mapowanie, relacje, kaskady
│   ├── test_repository.py      # zapytania, paginacja
│   ├── test_transactions.py    # granice transakcji, rollbacki
│   └── test_optimistic_lock.py # blokady
├── async/
│   ├── conftest.py             # async_engine, async_session
│   └── test_async_repository.py
├── migration/
│   ├── conftest.py
│   └── test_schema_drift.py    # sprawdzenie zgodności migracji z modelami
└── e2e/
    ├── conftest.py
    └── test_api_flow.py        # testy end-to-end na prawdziwej bazie
```

### Markery i CI

```toml
# pyproject.toml
[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"
addopts = "-ra --strict-markers"
markers = [
    "slow: testy wolne (migracje, 10k+ rekordów)",
    "integration: testy wymagające bazy danych",
    "e2e: testy end-to-end",
]
```

```yaml
# .github/workflows/ci.yml (fragment)
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install -e ".[test]"
      - name: Szybkie testy (bez wolnych)
        run: pytest -m "not slow" -q
      - name: Testy wolne (PostgreSQL, migracje)
        run: pytest -m "slow" -q
        env:
          DATABASE_URL: postgresql+psycopg://postgres:test@localhost:5432/test
```

### `pytest-xdist` — testy równoległe

```bash
pip install pytest-xdist
pytest -n auto -q
```

`-n auto` użyje wszystkich rdzeni procesora. **Uwaga:** każdy worker to osobny proces, więc:

- SQLite w pamięci jest **osobny dla każdego workera** — to dobrze, daje izolację automatycznie,
- PostgreSQL z jedną bazą jest **wspólny** — musisz zapewnić izolację (savepointy działają, ale nie na wszystkich poziomach),
- `testcontainers` dwukrotnie podniesie kontener — użyj markera, żeby takie testy szły tylko w jednym workerze (`-n0` lub `pytest-xdist` z `loadgroup`).

> ⚠️ **Pułapka — testy, które przechodzą tylko w izolacji albo tylko razem.**
> Objawy: `pytest test_a.py` przechodzi, `pytest` (cały zestaw) pada; albo odwrotnie. Przyczyny: (a) stan współdzielony między testami (baza plikowa bez izolacji), (b) listener złapany globalnie, (c) zmienne środowiskowe ustawione w jednym teście i nieprzywrócone, (d) zamrożone czas w jednym teście „wycieka”. Lekarstwo: `pytest -p no:randomly` (jeśli używasz randomizacji kolejności) → uruchom kilka razy; `pytest -x --lf` (fail-fast, last-failed) do zawężenia; i ostatecznie `pytest --forked` (uruchamia każdy test w osobnym procesie — brutalne, ale skuteczne).
>
> **Lepsze lekarstwo:** uruchom `pytest -p randomly --randomly-seed=12345` i powtarzaj z rosnącym ziarnem. Jeśli testy padają przy różnych kolejnościach — masz stan współdzielony. Nie „naprawiaj” tego wyłączaniem randomizacji: napraw prawdziwy problem.

> 🧪 **Ćwiczenie — w miejscu.**
> Dodaj `pytest-randomly`, uruchom testy trzy razy z różnymi ziarnami i sprawdź, czy któryś pada. Jeśli tak — znajdź, który stan jest współdzielony.

---

## Pełny przykład — `conftest.py` i osiem testów

Poniżej kompletny, uruchamialny zestaw. Zakłada, że masz strukturę:

```text
mypackage/                     # pusty plik mypackage/__init__.py
tests/
├── __init__.py
├── conftest.py
├── models.py
└── test_library.py
pyproject.toml
```

Instalacja:

```bash
pip install "SQLAlchemy>=2.0" pytest pytest-cov pytest-asyncio aiosqlite \
            "sqlalchemy[asyncio]" freezegun
```

### `tests/models.py`

```python
# tests/models.py
from __future__ import annotations

from datetime import datetime

from sqlalchemy import Column, ForeignKey, String, Table, UniqueConstraint, func
from sqlalchemy.orm import (
    DeclarativeBase,
    Mapped,
    mapped_column,
    relationship,
    validates,
)


class Base(DeclarativeBase):
    """Wspólna klasa bazowa dla wszystkich modeli w testach."""


# Tabela asocjacyjna dla relacji wiele-do-wielu Book <-> Category.
book_category = Table(
    "book_category",
    Base.metadata,
    Column("book_id", ForeignKey("books.id", ondelete="CASCADE"), primary_key=True),
    Column("category_id", ForeignKey("categories.id", ondelete="CASCADE"), primary_key=True),
)


class Author(Base):
    __tablename__ = "authors"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(120), unique=True)

    books: Mapped[list[Book]] = relationship(back_populates="author")

    def __repr__(self) -> str:
        return f"Author(id={self.id!r}, name={self.name!r})"


class Category(Base):
    __tablename__ = "categories"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(80), unique=True)

    books: Mapped[list[Book]] = relationship(
        secondary=book_category, back_populates="categories"
    )


class Book(Base):
    __tablename__ = "books"
    __table_args__ = (UniqueConstraint("isbn", name="uq_books_isbn"),)

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    isbn: Mapped[str] = mapped_column(String(20))
    author_id: Mapped[int] = mapped_column(ForeignKey("authors.id"))

    author: Mapped[Author] = relationship(back_populates="books")
    categories: Mapped[list[Category]] = relationship(
        secondary=book_category, back_populates="books"
    )
    reviews: Mapped[list[Review]] = relationship(
        back_populates="book", cascade="all, delete-orphan"
    )
    loans: Mapped[list[Loan]] = relationship(back_populates="book")


class Review(Base):
    __tablename__ = "reviews"

    id: Mapped[int] = mapped_column(primary_key=True)
    book_id: Mapped[int] = mapped_column(ForeignKey("books.id", ondelete="CASCADE"))
    rating: Mapped[int]
    comment: Mapped[str | None] = mapped_column(String(500))

    book: Mapped[Book] = relationship(back_populates="reviews")

    @validates("rating")
    def validate_rating(self, key: str, value: int) -> int:
        if not 1 <= value <= 5:
            raise ValueError("Ocena musi być w zakresie 1-5")
        return value


class Member(Base):
    __tablename__ = "members"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(120))
    email: Mapped[str] = mapped_column(String(200), unique=True)

    loans: Mapped[list[Loan]] = relationship(back_populates="member")

    @validates("email")
    def validate_email(self, key: str, value: str) -> str:
        if "@" not in value:
            raise ValueError("Nieprawidłowy adres e-mail")
        return value.strip().lower()


class Loan(Base):
    __tablename__ = "loans"

    id: Mapped[int] = mapped_column(primary_key=True)
    book_id: Mapped[int] = mapped_column(ForeignKey("books.id"))
    member_id: Mapped[int] = mapped_column(ForeignKey("members.id"))
    borrowed_at: Mapped[datetime] = mapped_column(server_default=func.now())
    returned_at: Mapped[datetime | None]

    # version_id_col: każdy UPDATE sprawdza, czy wersja się nie zmieniła.
    version: Mapped[int] = mapped_column(default=1)

    book: Mapped[Book] = relationship(back_populates="loans")
    member: Mapped[Member] = relationship(back_populates="loans")

    __mapper_args__ = {"version_id_col": version}
```

### `tests/conftest.py`

```python
# tests/conftest.py
from __future__ import annotations

from collections.abc import Iterator
from pathlib import Path
from typing import Any

import pytest
import pytest_asyncio
from sqlalchemy import Engine, event, create_engine
from sqlalchemy.ext.asyncio import (
    AsyncEngine,
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)
from sqlalchemy.orm import Session
from sqlalchemy.pool import StaticPool

from tests.models import Author, Base, Book


# --------------------------------------------------------------------------- #
# SYNCHRONICZNE ŚRODOWISKO
# --------------------------------------------------------------------------- #
@pytest.fixture(scope="session")
def engine() -> Iterator[Engine]:
    """Silnik SQLite w pamięci, wspólny dla całej sesji testowej.

    StaticPool sprawia, że wszystkie operacje używają jednego połączenia —
    dzięki temu baza ":memory:" nie znika między zapytaniami.
    """
    engine = create_engine(
        "sqlite://",  # baza w pamięci (nie plik!)
        poolclass=StaticPool,
        connect_args={"check_same_thread": False},
    )
    Base.metadata.create_all(engine)
    yield engine
    engine.dispose()


@pytest.fixture
def db_session(engine: Engine) -> Iterator[Session]:
    """Sesja w transakcji zewnętrznej — rollback po każdym teście.

    join_transaction_mode="create_savepoint" oznacza, że session.commit()
    zwalnia savepoint, a nie kończy transakcji zewnętrznej. Dzięki temu
    wszystko, co test zapisał, znika po teście, a sesja zachowuje się
    jak produkcyjna.
    """
    connection = engine.connect()
    transaction = connection.begin()
    session = Session(bind=connection, join_transaction_mode="create_savepoint")
    try:
        yield session
    finally:
        session.close()
        transaction.rollback()
        connection.close()


@pytest.fixture
def file_engine(tmp_path: Path) -> Iterator[Engine]:
    """Silnik na bazie plikowej — dla testów wymagających wielu połączeń."""
    db_file = tmp_path / "test.db"
    engine = create_engine(f"sqlite:///{db_file}")
    Base.metadata.create_all(engine)
    yield engine
    engine.dispose()


@pytest.fixture
def query_counter(engine: Engine) -> Iterator[dict[str, int]]:
    """Liczy polecenia SQL wykonane na silniku w trakcie testu."""
    licznik: dict[str, int] = {"n": 0}

    def policz(
        conn: Any,
        cursor: Any,
        statement: str,
        parameters: Any,
        context: Any,
        executemany: bool,
    ) -> None:
        licznik["n"] += 1

    event.listen(engine, "before_cursor_execute", policz)
    try:
        yield licznik
    finally:
        event.remove(engine, "before_cursor_execute", policz)


# --------------------------------------------------------------------------- #
# FABRYKI DANYCH
# --------------------------------------------------------------------------- #
@pytest.fixture
def make_author():
    """Zwraca funkcję tworzącą autora z unikalną nazwą (licznikiem)."""
    licznik = {"n": 0}

    def _make(name: str | None = None) -> Author:
        licznik["n"] += 1
        return Author(name=name or f"Autor {licznik['n']}")

    return _make


@pytest.fixture
def make_book():
    licznik = {"n": 0}

    def _make(author: Author, title: str | None = None) -> Book:
        licznik["n"] += 1
        return Book(
            title=title or f"Ksiazka {licznik['n']}",
            isbn=f"isbn-{licznik['n']:05d}",
            author=author,
        )

    return _make


@pytest.fixture
def seeded_library(db_session, make_author, make_book) -> list[Author]:
    """Trzech autorów, po dwie książki każdy."""
    autorzy: list[Author] = []
    for _ in range(3):
        autor = make_author()
        autor.books.extend([make_book(author=autor), make_book(author=autor)])
        db_session.add(autor)
        autorzy.append(autor)
    db_session.flush()
    return autorzy


# --------------------------------------------------------------------------- #
# ASYNCHRONICZNE ŚRODOWISKO
# --------------------------------------------------------------------------- #
@pytest_asyncio.fixture
async def async_engine() -> AsyncEngine:
    engine = create_async_engine(
        "sqlite+aiosqlite://",
        poolclass=StaticPool,
    )
    async with engine.begin() as connection:
        await connection.run_sync(Base.metadata.create_all)
    yield engine
    await engine.dispose()


@pytest_asyncio.fixture
async def async_session(async_engine: AsyncEngine) -> AsyncSession:
    factory = async_sessionmaker(async_engine, expire_on_commit=False)
    async with factory() as session:
        yield session
```

### `tests/test_library.py` — osiem testów

```python
# tests/test_library.py
from __future__ import annotations

from datetime import datetime

import pytest
from sqlalchemy import select, update
from sqlalchemy.exc import IntegrityError, InvalidRequestError, StaleDataError
from sqlalchemy.orm import raiseload, selectinload, sessionmaker

from tests.models import Author, Book, Loan, Member, Review


# --------------------------------------------------------------------------- #
# 1. CRUD — tworzenie i odczyt
# --------------------------------------------------------------------------- #
def test_crud_tworzenie_i_odczyt(db_session, make_author, make_book) -> None:
    autor = make_author(name="Stanisław Lem")
    db_session.add(make_book(author=autor, title="Solaris"))
    db_session.flush()

    ksiazka = db_session.scalars(
        select(Book).where(Book.title == "Solaris")
    ).one()
    assert ksiazka.author.name == "Stanisław Lem"


# --------------------------------------------------------------------------- #
# 2. Ograniczenie UNIQUE — IntegrityError i rollback
# --------------------------------------------------------------------------- #
def test_duplikat_isbn_rzuca_integrity_error(db_session, make_author, make_book) -> None:
    autor = make_author()
    db_session.add(make_book(author=autor, title="A"))
    db_session.flush()
    zajete_isbn = db_session.scalars(select(Book.isbn)).one()

    with pytest.raises(IntegrityError, match="UNIQUE"):
        db_session.add(
            Book(title="B", isbn=zajete_isbn, author=make_author(name="Inny"))
        )
        db_session.flush()

    db_session.rollback()


# --------------------------------------------------------------------------- #
# 3. Kaskada delete-orphan
# --------------------------------------------------------------------------- #
def test_delete_orphan_usuwie_recenzje(db_session, make_author, make_book) -> None:
    ksiazka = make_book(author=make_author())
    ksiazka.reviews.append(Review(rating=5, comment="Świetna"))
    db_session.add(ksiazka)
    db_session.flush()
    assert db_session.scalars(select(Review)).all() != []

    ksiazka.reviews.clear()
    db_session.flush()

    assert db_session.scalars(select(Review)).all() == []


# --------------------------------------------------------------------------- #
# 4. Walidator @validates — normalizacja i odrzucenie
# --------------------------------------------------------------------------- #
def test_walidator_emaila_normalizuje_i_odrzuca(db_session) -> None:
    czlonek = Member(name="Ala", email="  ALA@Example.COM ")
    assert czlonek.email == "ala@example.com"

    db_session.add(czlonek)
    db_session.flush()
    assert czlonek.email == "ala@example.com"

    with pytest.raises(ValueError, match="e-mail"):
        Member(name="Bob", email="to-nie-jest-email")


# --------------------------------------------------------------------------- #
# 5. Blokada optymistyczna — StaleDataError
# --------------------------------------------------------------------------- #
def test_blokada_optymistyczna_wykrywa_konflikt(file_engine, make_author, make_book) -> None:
    factory = sessionmaker(bind=file_engine)

    with factory.begin() as setup:
        ksiazka = make_book(author=make_author(), title="Baza")
        czlonek = Member(name="Jan", email="jan@example.com")
        setup.add(Loan(book=ksiazka, member=czlonek))

    with factory() as sesja_a, factory() as sesja_b:
        poz_a = sesja_a.scalars(select(Loan)).one()
        poz_b = sesja_b.scalars(select(Loan)).one()
        assert poz_a.version == 1

        poz_b.returned_at = datetime(2026, 9, 20, 12, 0)
        sesja_b.commit()  # version 1 -> 2

        poz_a.returned_at = datetime(2026, 9, 20, 13, 0)
        with pytest.raises(StaleDataError):
            sesja_a.commit()


# --------------------------------------------------------------------------- #
# 6. Liczba zapytań — regresja wydajnościowa (N+1)
# --------------------------------------------------------------------------- #
def test_selectinload_ogranicza_liczbe_zapytan(
    db_session, seeded_library, query_counter
) -> None:
    query_counter["n"] = 0

    autorzy = db_session.scalars(
        select(Author).options(selectinload(Author.books))
    ).all()
    assert len(autorzy) == 3
    assert sum(len(a.books) for a in autorzy) == 6
    # 2 zapytania: autorzy + ich książki (IN).
    assert query_counter["n"] == 2


def test_lazy_loading_w_petli_generuje_n_plus_1(
    db_session, seeded_library, query_counter
) -> None:
    query_counter["n"] = 0

    autorzy = db_session.scalars(select(Author)).all()
    for autor in autorzy:
        _ = autor.books

    # 1 + 3 = 4 zapytania.
    assert query_counter["n"] == 4


# --------------------------------------------------------------------------- #
# 7. raiseload — bezpiecznik przeciw przypadkowemu leniwemu ładowaniu
# --------------------------------------------------------------------------- #
def test_raiseload_blokuje_przypadkowe_lazy(db_session, seeded_library) -> None:
    autor = db_session.scalars(
        select(Author).options(raiseload(Author.books))
    ).one()

    with pytest.raises(InvalidRequestError, match="lazy='raise'"):
        _ = autor.books


# --------------------------------------------------------------------------- #
# 8. Async — repozytorium i commit
# --------------------------------------------------------------------------- #
@pytest.mark.asyncio
async def test_async_zapis_i_odczyt(async_session, make_author, make_book) -> None:
    autor = make_author(name="Ursula K. Le Guin")
    async_session.add(make_book(author=autor, title="Wyprawa"))
    await async_session.commit()

    wynik = await async_session.scalars(select(Book))
    ksiazki = wynik.all()
    assert len(ksiazki) == 1
    assert ksiazki[0].title == "Wyprawa"
```

Uruchomienie:

```bash
pytest -q
# 8 passed in ~0.4s
```

> 🔬 **Pod maską — co faktycznie leci do SQLite w teście 5.**
> ```sql
> -- faza przygotowania
> INSERT INTO authors (name) VALUES (?);
> INSERT INTO books (title, isbn, author_id) VALUES (?, ?, ?);
> INSERT INTO members (name, email) VALUES (?, ?);
> INSERT INTO loans (book_id, member_id, returned_at, version) VALUES (?, ?, ?, 1);
> COMMIT;
>
> -- sesja_b
> SELECT ... FROM loans;
> UPDATE loans SET returned_at = ?, version = 2 WHERE loans.id = ? AND loans.version = 1;
> COMMIT;
>
> -- sesja_a (nieaktualna wersja)
> SELECT ... FROM loans;
> -- przy commit: rowcount = 0 -> StaleDataError
> ```

> ⚠️ **Pułapka — test 5 nie działa w strategii savepoint.**
> Ten test celowo używa `file_engine`, a nie `db_session`. Gdybyś użył `db_session`, obie sesje siedziałyby w tej samej transakcji zewnętrznej — `sesja_b.commit()` zwolniłoby tylko savepoint, a `sesja_a` nie zobaczyłaby nowej wersji. Test przeszedłby z fałszywym zadowoleniem. **Blokady testuj na osobnych połączeniach.**

> 🧪 **Ćwiczenie — w miejscu.**
> Dodaj do zestawu dziewiąty test: `test_bulk_update_omija_walidator`, używający `update(Member)` na `db_session`. Sprawdź, czy potrafisz przewidzieć wynik, zanim go uruchomisz.

---

## Podsumowanie

1. **Testujemy swój kod, nie bibliotekę.** SQLAlchemy ma własne testy; Ty weryfikujesz modele, zapytania, logikę biznesową i mapowanie.
2. **`pytest` + fixtures to fundament.** `yield` dzieli fixture na przygotowanie i sprzątanie; `conftest.py` udostępnia je w całym projekcie; parametryzacja eliminuje kopiowanie.
3. **SQLite w pamięci + `StaticPool` + `check_same_thread=False`** to domyślne środowisko testowe. Bez `StaticPool` każde połączenie widzi pustą bazę — i to jest najczęstszy błąd początkujących.
4. **Strategia izolacji B (transakcja zewnętrzna + `join_transaction_mode="create_savepoint"`)** to standard: szybka, dokładna, a sesja zachowuje się produkcyjnie.
5. **Prawdziwe commity widoczne dla kilku sesji wymagają osobnych połączeń** — używaj bazy plikowej w `tmp_path` lub PostgreSQL.
6. **`create_all()` do 99% testów, migracje Alembic do testów dryfu schematu.** Test `compare_metadata` wyłapuje rozjazd między modelami a migracjami.
7. **`query_counter` to najtańsza ochrona przed regresją wydajnościową.** `assert query_counter["n"] == 2` mówi więcej niż dziesięć testów funkcjonalnych.
8. **`raiseload("*")` wymusza jawny eager loading.** Świetnie wykrywa N+1 jeszcze przed produkcją.
9. **Nigdy nie mockuj `Session`.** Prawdziwa baza w pamięci daje więcej i jest równie szybka.
10. **Async rządzi się swoimi prawami:** `expire_on_commit=False`, jedna pętla zdarzeń, `pytest-asyncio` z `asyncio_mode = "auto"`, świadomy scope fixtures.

---

## Ćwiczenia

### Ćwiczenie 1 — test regresji N+1 (łatwe)

Masz funkcję `list_orders_with_items(session)` (w repozytorium zamówień), która zwraca listę zamówień z ich pozycjami:

```python
def list_orders_with_items(session: Session) -> list[Order]:
    stmt = select(Order).options(selectinload(Order.items))
    return list(session.scalars(stmt).all())
```

**Zadanie:** napisz test, który **udowadnia**, że w tej funkcji nie ma N+1. Test ma:
1. przygotować 10 zamówień, każde z 3 pozycjami,
2. wyzerować licznik,
3. wywołać funkcję,
4. asertować **dokładną** liczbę zapytań.

Następnie dodaj „test strażniczy”: zmień implementację na wersję bez `selectinload` i sprawdź, czy test pada. Zapisz oba wyniki w komentarzu.

<details>
<summary><strong>Rozwiązanie</strong></summary>

```python
# tests/test_orders_regression.py
import pytest
from sqlalchemy import select
from sqlalchemy.orm import Session, selectinload

from tests.models import Order  # zakładamy model z relacją Order.items


def list_orders_with_items(session: Session) -> list[Order]:
    stmt = select(Order).options(selectinload(Order.items))
    return list(session.scalars(stmt).all())


def list_orders_with_items_broken(session: Session) -> list[Order]:
    # Wersja celowo zepsuta — bez eager loadingu.
    return list(session.scalars(select(Order)).all())


def test_brak_n_plus_1(db_session, query_counter) -> None:
    for i in range(10):
        zam = Order(number=f"ZAM-{i}")
        zam.items.extend([OrderItem(sku=f"SKU-{i}-{j}") for j in range(3)])
        db_session.add(zam)
    db_session.flush()
    query_counter["n"] = 0

    zamowienia = list_orders_with_items(db_session)
    assert len(zamowienia) == 10
    assert sum(len(z.items) for z in zamowienia) == 30

    # 2 zapytania: 1 na zamówienia + 1 na pozycje (IN ...).
    assert query_counter["n"] == 2


def test_wykrywa_regresje_n_plus_1(db_session, query_counter) -> None:
    for i in range(10):
        zam = Order(number=f"ZAM-{i}")
        zam.items.extend([OrderItem(sku=f"SKU-{i}-{j}") for j in range(3)])
        db_session.add(zam)
    db_session.flush()
    query_counter["n"] = 0

    zamowienia = list_orders_with_items_broken(db_session)
    # Wynik funkcjonalny jest poprawny — tylko wydajność jest zła.
    assert len(zamowienia) == 10
    # Ale liczba zapytań zdradza problem.
    assert query_counter["n"] > 2
```

**Komentarz:** pierwszy test to kontrakt wydajnościowy. Drugi pokazuje, że **liczenie zapytań potrafi złapać coś, czego nie złapie żadna asercja na wynik**. Gdybyś w przyszłości usunął `selectinload` przez przypadek, tylko drugi test (a raczej jego negacja) by to pokazał. W prawdziwym projekcie trzymaj więc ten pierwszy test w zestawie i nie „upraszczaj” go do sprawdzania samego wyniku.

</details>

### Ćwiczenie 2 — test migracji (średnie)

Zakładając, że masz w projekcie strukturę Alembic i migrację, która dodaje kolumnę `books.published_year` z migracją danych:

**Zadanie:** napisz test, który:
1. uruchamia `alembic upgrade` do rewizji **przed** dodaniem kolumny,
2. dodaje wiersz książki w starej strukturze,
3. uruchamia `alembic upgrade head`,
4. sprawdza, że kolumna `published_year` istnieje i że wartość **została przeniesiona** (albo ustawiona na `NULL`, jeśli migracja danych tego nie robiła),
5. wykonuje `alembic downgrade` z powrotem i sprawdza, że stara struktura jest odtworzona (i kolumny już nie ma).

Uruchomienie: `pytest -m slow -q`.

<details>
<summary><strong>Rozwiązanie</strong></summary>

```python
# tests/migration/test_publikacja.py
import subprocess
from pathlib import Path

import pytest
from sqlalchemy import create_engine, inspect, text

pytestmark = pytest.mark.slow


def _alembic(komenda: list[str], url: str) -> None:
    subprocess.run(
        ["alembic", *komenda],
        check=True,
        env={"DATABASE_URL": url, "PATH": "/usr/bin:/usr/local/bin"},
    )


def test_migracja_dodaje_kolumne_i_przenosi_dane(
    tmp_path: Path, monkeypatch
) -> None:
    db_file = tmp_path / "migracja.db"
    url = f"sqlite:///{db_file}"
    monkeypatch.setenv("DATABASE_URL", url)

    # 1. Rewizja przed migracją (podmień na nazwę Twojej rewizji).
    rewizja_przed = "abc123"
    _alembic(["upgrade", rewizja_przed], url)

    # 2. Wstawiamy wiersz w starej strukturze — tutaj gołym SQL-em,
    #    żeby nie zależeć od aktualnych modeli.
    engine = create_engine(url)
    with engine.begin() as conn:
        conn.execute(
            text(
                "INSERT INTO authors (name) VALUES ('Lem')"
            )
        )
        conn.execute(
            text(
                "INSERT INTO books (title, isbn, author_id) "
                "VALUES ('Solaris', 'isbn-1', 1)"
            )
        )

    # 3. Upgrade do najnowszej rewizji.
    _alembic(["upgrade", "head"], url)

    # 4. Sprawdzamy strukturę i dane.
    inspector = inspect(engine)
    kolumny = {c["name"] for c in inspector.get_columns("books")}
    assert "published_year" in kolumny

    with engine.connect() as conn:
        rok = conn.execute(
            text("SELECT published_year FROM books WHERE isbn = 'isbn-1'")
        ).scalar_one()

    # Wartość przeniesiona przez migrację danych.
    assert rok == 1961

    # 5. Downgrade i sprawdzenie odwrotności.
    _alembic(["downgrade", rewizja_przed], url)
    kolumny_po = {c["name"] for c in inspector.get_columns("books")}
    assert "published_year" not in kolumny_po

    engine.dispose()
```

**Komentarz:** test celowo używa `text()` i surowego SQL, żeby **nie zależeć od aktualnych modeli** — modele zmieniają się z czasem, a migracja musi być testowana w swoim ówczesnym kontekście. Druga rzecz: ustawienie zmiennej środowiskowej `DATABASE_URL` wewnątrz `subprocess` gwarantuje, że `env.py` Alembica wskaże testową bazę, a nie produkcyjną. To zabezpieczenie chroni przed katastrofą, gdy `.env` ma nieaktualny URL.

</details>

### Ćwiczenie 3 — pełna izolacja i test async (trudne)

Masz projekt z: synchronicznym repozytorium `BookRepository`, asynchronicznym `AsyncBookRepository`, a także serwisem, który używa obu. Zbuduj **kompletny** `conftest.py` obejmujący:

1. `engine` (session-scoped, SQLite in-memory z `StaticPool`),
2. `db_session` (savepoint + rollback),
3. `file_engine` (dla testów współbieżności),
4. `async_engine` i `async_session` (osobne, z `pytest_asyncio`),
5. `query_counter`,
6. `seeded_library`.

Następnie napisz test, który **udowadnia**, że `AsyncBookRepository` wykonuje dokładnie tyle samo zapytań co jego synchroniczny odpowiednik dla tej samej operacji.

<details>
<summary><strong>Rozwiązanie</strong></summary>

`conftest.py` znajdziesz powyżej w sekcji „Pełny przykład” — użyj go bez zmian, dodając fixture do liczenia zapytań w wersji async:

```python
# tests/conftest.py (dodatek)
import pytest_asyncio
from sqlalchemy import event
from sqlalchemy.ext.asyncio import AsyncEngine


@pytest_asyncio.fixture
async def async_query_counter(async_engine: AsyncEngine):
    licznik = {"n": 0}

    def policz(conn, cursor, statement, parameters, context, executemany) -> None:
        licznik["n"] += 1

    # Event dla AsyncEngine rejestruje się na jego wewnętrznym, sync engine.
    event.listen(async_engine.sync_engine, "before_cursor_execute", policz)
    try:
        yield licznik
    finally:
        event.remove(async_engine.sync_engine, "before_cursor_execute", policz)
```

Kluczowa subtelność: `AsyncEngine` jest **nakładką** na `Engine`. Zdarzenia takie jak `before_cursor_execute` są rejestrowane na `async_engine.sync_engine`, nie na samym `AsyncEngine`. Bez tego niuansu testy „nie widzą” zapytań i licznik zawsze pokazuje zero.

Test porównawczy:

```python
# tests/test_parity.py
import pytest
from sqlalchemy import select
from sqlalchemy.orm import selectinload

from tests.models import Author


def list_authors_sync(session) -> list[Author]:
    return list(
        session.scalars(select(Author).options(selectinload(Author.books))).all()
    )


async def list_authors_async(session) -> list[Author]:
    result = await session.scalars(
        select(Author).options(selectinload(Author.books))
    )
    return list(result.all())


def test_sync_wersja_zapytan(db_session, seeded_library, query_counter) -> None:
    query_counter["n"] = 0
    autorzy = list_authors_sync(db_session)
    assert len(autorzy) == 3
    liczba_sync = query_counter["n"]
    assert liczba_sync == 2


@pytest.mark.asyncio
async def test_async_wersja_ma_tyle_samo_zapytan(
    async_session, async_query_counter
) -> None:
    # Przygotowanie świata danych w wersji async.
    autorzy = []
    for i in range(3):
        autor = Author(name=f"Autor {i}")
        autor.books.extend(
            [Book(title=f"Ksiazka {i}-{j}", isbn=f"isbn-{i}-{j}") for j in range(2)]
        )
        async_session.add(autor)
        autorzy.append(autor)
    await async_session.commit()

    async_query_counter["n"] = 0
    wynik = await list_authors_async(async_session)
    assert len(wynik) == 3
    # Ten sam kontrakt: 2 zapytania, niezależnie od trybu sync/async.
    assert async_query_counter["n"] == 2
```

**Komentarz:** ten test ma konkretną wartość inżynierską — jeśli ktoś przepisze repozytorium z sync na async (albo odwrotnie) i wprowadzi regresję N+1 tylko w jednej wersji, test to wyłapie. Zauważ też, że **liczymy zapytania w obu trybach tą samą logiką**, tylko z innym punktem rejestracji eventu. Spójność miar daje porównywalne wyniki.

</details>

---

## Najczęstsze błędy i jak je czytać

| Komunikat / objaw | Przyczyna | Naprawa |
|---|---|---|
| `sqlite3.ProgrammingError: SQLite objects created in a thread can only be used in that same thread` | Brak `check_same_thread=False`, a połączenie użyte w innym wątku | Dodaj `connect_args={"check_same_thread": False}` przy `create_engine` **razem z** `poolclass=StaticPool` |
| `OperationalError: no such table: books` (losowo, w teście który wcześniej działał) | Baza SQLite `:memory:` bez `StaticPool` — każde nowe połączenie dostaje pustą bazę | Dodaj `poolclass=StaticPool`; alternatywnie użyj bazy plikowej w `tmp_path` |
| `sqlalchemy.exc.PendingRollbackError: This Session's transaction has been rolled back due to a previous exception during flush` | Test złapał wyjątek po `flush()` i nie wywołał `rollback()` | Zawsze po złapanym błędzie w transakcji: `session.rollback()` przed kolejną operacją |
| `sqlalchemy.orm.exc.DetachedInstanceError: Instance <Book...> is not bound to a Session` | Test zamknął sesję i próbuje odczytać leniwą relację poza nią | Załaduj relację jawnie (`selectinload`) lub trzymaj sesję otwartą przez czas użycia obiektu; ustaw `expire_on_commit=False`, jeśli potrzebujesz obiektu po commicie |
| `sqlalchemy.exc.InvalidRequestError: 'Author.books' is not available due to lazy='raise'` (gdy użyto `raiseload`) | Test użył `raiseload(...)` i nie załadował relacji jawnie | Dodaj `selectinload(Author.books)` do `select()` albo — jeśli leniwe ładowanie jest tu OK — usuń `raiseload` z tego konkretnego testu |
| `RuntimeError: Task got Future attached to a different loop` | Fixture o scope `session` tworzy `AsyncEngine` w jednej pętli, a test działa w innej | Ustaw fixture w `scope="function"` albo skonfiguruj `loop_scope="session"` w `pyproject.toml` i oznacz testy odpowiednim markerem |
| `sqlalchemy.exc.StaleDataError: UPDATE statement on table 'loans' expected to update 1 row(s); 0 were matched` | Blokada optymistyczna: wiersz został zmieniony przez inną sesję | To **zamierzone** zachowanie. Obsłuż wyjątek w kodzie (retry lub informacja dla użytkownika). Nie „naprawiaj” przez usunięcie `version_id_col` |
| `sqlite3.OperationalError: database is locked` | Dwa równoległe zapisy w SQLite; brak wsparcia dla wielu piszących | Skróć transakcje; użyj PostgreSQL dla testów współbieżności; w warstwie plikowej włącz `PRAGMA journal_mode=WAL` |
| `sqlalchemy.exc.IntegrityError: UNIQUE constraint failed: books.isbn` | Duplikat wartości z ograniczeniem `UNIQUE` — brak `rollback()` po złapaniu | `session.rollback()` po przechwyceniu; w kodzie aplikacji tłumacz na wyjątek domenowy |
| `ValueError: Ocena musi być w zakresie 1-5` wycieka z testu | `@validates` rzucił wyjątek, a test go nie złapał | Owiń `pytest.raises(...)`, jeśli testujesz walidację; jeśli nie — popraw dane testowe |
| `ScopeMismatch: You tried to access the function scoped fixture ... with a session scoped request object` | Fixture o mniejszym scope (function) użyta w fixture o większym scope (session) | Zmień scope jednej z fixtures albo rozbij zależność. Częste przy `engine` (session) i `db_session` (function) |
| `PytestUnknownMarkWarning: Unknown pytest.mark.slow` | Marker użyty w kodzie, nie zadeklarowany w konfiguracji | Dodaj sekcję `markers` w `pyproject.toml` |
| `AttributeError: 'Session' object has no attribute 'query'` w nowym kodzie | Kod próbuje starego API 1.x (`session.query`) | Przepisz na `session.scalars(select(...))` / `session.execute(select(...))`. Sprawdź moduł A4 dla tabeli migracji |
| Testy przechodzą lokalnie, padają w CI na PostgreSQL | Kod korzysta z zachowania SQLite (np. dynamicznych typów, braku `FOR UPDATE`) | Uruchom testy na PostgreSQL jako oddzielny marker; różnice dialektów są realne, nie „kosmetyczne” |
| Testy przechodzą w izolacji, padają razem | Współdzielony stan: baza plikowa bez izolacji, listener globalny, zmienne środowiska nieprzywrócone | Użyj strategii savepoint, zdejmuj listenery w `finally`, używaj `monkeypatch` zamiast `os.environ[...] = ...` |

---

## Słowniczek modułu

| Termin (EN) | Polski odpowiednik | Wyjaśnienie |
|---|---|---|
| fixture | uchwyt / przygotowanie | Funkcja dostarczająca zasób testowi; może mieć część sprzątającą po `yield` |
| `conftest.py` | plik konfiguracyjny testów | Plik, którego fixtures pytest automatycznie udostępnia testom w danym katalogu i podkatalogach |
| scope (fixture) | zakres | Jak długo żyje fixture: `function` (domyślnie), `class`, `module`, `package`, `session` |
| in-memory database | baza w pamięci | Baza trzymana wyłącznie w RAM; znika po zamknięciu połączenia (SQLite) |
| connection pool | pula połączeń | Zbiór gotowych połączeń do bazy, z którego aplikacja pożycza i zwraca połączenia |
| `StaticPool` | pula statyczna | Pula trzymająca jedno połączenie na zawsze; kluczowa dla SQLite `:memory:` |
| `check_same_thread` | kontrola wątku | Parametr `sqlite3` zabraniający używania połączenia w innym wątku niż utworzone |
| savepoint | punkt zapisu | Znacznik wewnątrz transakcji, do którego można częściowo wycofać zmiany |
| outer transaction | transakcja zewnętrzna | Transakcja otwarta poza sesją, w której sesja działa jako zagnieżdżona |
| isolation (test) | izolacja testów | Zapewnienie, że testy nie wpływają na siebie i nie zależą od kolejności |
| truncate | opróżnienie tabeli | Szybkie usunięcie wszystkich wierszy tabeli z zachowaniem struktury |
| factory (test) | fabryka danych | Funkcja lub klasa tworząca typowe obiekty testowe z domyślnymi wartościami |
| `factory_boy` | biblioteka fabryk | Popularna biblioteka do budowania obiektów testowych z `SubFactory`, `Faker`, sekwencjami |
| `polyfactory` | biblioteka fabryk | Alternatywa dla `factory_boy`, generująca fabryki z adnotacji typów |
| seed data | dane zasiewowe | Wstępnie przygotowany „świat” danych, na którym pracuje test |
| `before_cursor_execute` | zdarzenie przed wykonaniem kursora | Event SQLAlchemy wywoływany tuż przed wysłaniem SQL do bazy; używany do liczenia zapytań |
| `raiseload` | ładowanie z błędem | Polityka ładowania relacji rzucająca wyjątek, gdy relacja nie została jawnie załadowana |
| N+1 problem | problem N+1 | Antywzorzec polegający na wykonaniu 1 zapytania głównego i N dodatkowych (jedno na obiekt) |
| query counter | licznik zapytań | Mechanizm (najczęściej event) zliczający zapytania wykonane w trakcie testu |
| `StaleDataError` | błąd nieaktualnych danych | Wyjątek SQLAlchemy przy niezgodności wersji w blokadzie optymistycznej |
| optimistic locking | blokada optymistyczna | Strategia wykrywania konfliktów przez numer wersji, bez blokowania wiersza w bazie |
| `pytest-asyncio` | wtyczka asynchroniczna | Plugin obsługujący testy `async def` i asynchroniczne fixtures |
| event loop | pętla zdarzeń | Mechanizm `asyncio` zarządzający zadaniami asynchronicznymi; sesja i silnik są z nią związane |
| `loop_scope` | zakres pętli | W pytest-asyncio: jak długo żyje pętla zdarzeń (`function` lub `session`) |
| mocking | mockowanie | Podmiana realnego obiektu atrapą (np. `MagicMock`) w celu izolacji od zależności |
| schema drift | rozjazd schematu | Sytuacja, w której schemat bazy różni się od modeli lub migracji |
| `compare_metadata` | porównanie metadanych | Funkcja Alembica zwracająca listę różnic między bazą a modelami |
| `testcontainers` | kontenery testowe | Biblioteka uruchamiająca tymczasowe kontenery (np. PostgreSQL) na czas testów |
| anonimizacja | anonimizacja | Zastąpienie danych osobowych wartościami neutralnymi w danych testowych |
| `freezegun` | zamrażarka czasu | Biblioteka podmieniająca zegar systemowy na stałą datę w trakcie testu |
| marker (pytest) | znacznik | Etykieta testu (`@pytest.mark.slow`) umożliwiająca selektywne uruchamianie |
| `pytest-xdist` | równoległość pytest | Wtyczka uruchamiająca testy równolegle w wielu procesach |
| CI (Continuous Integration) | ciągła integracja | Automatyczne uruchamianie testów przy każdej zmianie kodu w repozytorium |

---

## Dalsze czytanie

- SQLAlchemy — ORM Quick Start: <https://docs.sqlalchemy.org/en/20/orm/quickstart.html>
- SQLAlchemy — Joining a Session into an External Transaction (strategia savepoint): <https://docs.sqlalchemy.org/en/20/orm/session_transaction.html#joining-a-session-into-an-external-transaction-such-as-for-test-suites>
- SQLAlchemy — pula połączeń (`StaticPool`, `QueuePool`, `NullPool`): <https://docs.sqlalchemy.org/en/20/core/pooling.html>
- SQLAlchemy — dialekt SQLite (ograniczenia, `check_same_thread`): <https://docs.sqlalchemy.org/en/20/dialects/sqlite.html>
- SQLAlchemy — VERSIONING (blokada optymistyczna, `version_id_col`): <https://docs.sqlalchemy.org/en/20/orm/versioning.html>
- SQLAlchemy — relationship loading techniques (`selectinload`, `joinedload`, `raiseload`): <https://docs.sqlalchemy.org/en/20/orm/queryguide/relationships.html>
- SQLAlchemy — ORM Events (`before_flush`, `after_flush`): <https://docs.sqlalchemy.org/en/20/orm/events.html>
- SQLAlchemy — asyncio (`AsyncEngine`, `AsyncSession`, `run_sync`): <https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html>
- pytest — dokumentacja: <https://docs.pytest.org/en/stable/>
- pytest-asyncio — dokumentacja i konfiguracja `loop_scope`: <https://pytest-asyncio.readthedocs.io/>
- `factory_boy` — dokumentacja (w tym `SQLAlchemyModelFactory`): <https://factoryboy.readthedocs.io/>
- `polyfactory` — dokumentacja: <https://polyfactory.litestar.dev/>
- `testcontainers-python` — dokumentacja: <https://testcontainers-python.readthedocs.io/>
- Alembic — dokumentacja (autogenerate, `compare_metadata`): <https://alembic.sqlalchemy.org/en/latest/>
- `freezegun` — repozytorium: <https://github.com/spulec/freezegun>

---

## Co dalej

Masz teraz komplet narzędzi do tego, żeby **udowodnić**, że warstwa danych działa: szybkie testy integracyjne, fabryki danych, licznik zapytań jako kontrakt wydajnościowy i strategia testowania migracji. Umiesz też odróżnić test, który coś sprawdza, od testu, który tylko „przechodzi”.

W następnym module zejdziemy poziom wyżej — od testów pojedynczych zachowań do **architektury**. Zastanowimy się, gdzie kończy się model ORM, a zaczyna model domenowy, jak oddzielić bazę od logiki biznesowej i jak zaprojektować warstwy tak, żeby dały się testować bez mockowania — bo właśnie teraz, gdy umiesz testować warstwę danych, zobaczysz wyraźnie, że część Twoich testów sprawdza zbyt dużo na raz.

➡️ **Następny moduł:** `19_warstwy_i_data_mapper.md` — warstwy, modele domenowe i Data Mapper.

<!-- koniec modułu 18 -->