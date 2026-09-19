# Moduł 02 — Środowisko, `Engine` i połączenia

W tym module zejdziemy z poziomu teorii (moduł `01_wprowadzenie.md`) na poziom konkretu: zainstalujesz SQLAlchemy, skonfigurujesz połączenie z bazą danych i wykonasz pierwsze prawdziwe zapytanie. Poznasz `Engine` — obiekt, który w SQLAlchemy odpowiada za komunikację z bazą — oraz zrozumiesz, **co dokładnie** dzieje się między Twoim kodem w Pythonie a plikiem bazy danych. Po tym module będziesz umieć samodzielnie przygotować działające środowisko i świadomie sterować tym połączeniem, zamiast traktować je jak czarną skrzynkę.

---

## Metadane modułu

| Pole | Wartość |
|---|---|
| **Poziom** | 🟢 podstawowy |
| **Czas** | ~120 minut (plus czas na instalację) |
| **Wymagania wstępne** | `01_wprowadzenie.md` (rozumiesz różnicę Core/ORM), umiesz uruchomić Pythona i `pip` w terminalu |
| **Zakres** | `venv`, instalacja, `Engine`, URL-e, pula połączeń, `Connection`, `text()`, logowanie, pierwsze zapytania |
| **Czego NIE ma** | definiowania tabel i modeli (to moduły `03` i `07`), sesji ORM (moduł `08`) |
| **Bazy w module** | głównie SQLite (zero konfiguracji), wariant PostgreSQL na końcu |

> 💡 **Jak czytać ten moduł.** Wszystkie przykłady są uruchamialne na SQLite, który nie wymaga instalowania żadnego serwera bazy danych — dane trzyma w zwykłym pliku. Dzięki temu możesz ćwiczyć od razu, bez „najpierw zainstaluj PostgreSQL”. Na końcu pokażemy, jak to samo zrobić z PostgreSQL, żebyś zobaczył różnicę.

---

## Spis treści

1. [Zanim zaczniemy — po co osobne środowisko](#zanim-zaczniemy--po-co-osobne-środowisko)
2. [Środowisko pracy krok po kroku](#środowisko-pracy-krok-po-kroku)
3. [`Engine` — serce komunikacji z bazą](#engine--serce-komunikacji-z-bazą)
4. [URL-e połączeń — adres bazy danych](#url-e-połączeń--adres-bazy-danych)
5. [`create_engine()` — parametry pod lupą](#create_engine--parametry-pod-lupą)
6. [Pula połączeń (connection pool)](#pula-połączeń-connection-pool)
7. [Wykonywanie SQL: `connect`, `begin`, `text()`](#wykonywanie-sql-connect-begin-text)
8. [Parametry wiązane i bezpieczeństwo](#parametry-wiązane-i-bezpieczeństwo)
9. [Logowanie — podsłuchiwanie SQL-a](#logowanie--podsłuchiwanie-sql-a)
10. [Typowe problemy zależne od systemu](#typowe-problemy-zależne-od-systemu)
11. [Pełny przykład: `first_steps.py`](#pełny-przykład-first_stepspy)
12. [Podsumowanie](#podsumowanie)
13. [Ćwiczenia](#ćwiczenia)
14. [Najczęstsze błędy i jak je czytać](#najczęstsze-błędy-i-jak-je-czytać)
15. [Słowniczek modułu](#słowniczek-modułu)
16. [Dalsze czytanie](#dalsze-czytanie)
17. [Co dalej](#co-dalej)

---

## Zanim zaczniemy — po co osobne środowisko

Wyobraź sobie, że masz szufladę z narzędziami. Jeśli wszystkie projekty — kuchenny, rowerowy i hydrauliczny — wrzucasz do jednej szuflady, to po miesiącu nie wiesz, który klucz jest do czego, a dokładając nowe narzędzie możesz przypadkiem zgubić stare. Dokładnie tak działa instalowanie bibliotek Pythona „globalnie”, na cały system.

W Pythonie każdy projekt powinien mieć **własne, izolowane środowisko** z własnym zestawem bibliotek w konkretnych wersjach. Nazywa się to **środowiskiem wirtualnym** (virtual environment, w skrócie `venv`).

> 💡 **Analogia — środowisko wirtualne to własna szuflada na narzędzia.** Projekt A potrzebuje SQLAlchemy 2.0, projekt B (stary) potrzebuje 1.4. W jednej szufladzie te dwie wersje by się kłóciły, bo nazywają się tak samo. Osobne szuflady rozwiązują problem: każdy projekt sięga tylko do swojej.

> 🧠 **Dlaczego tak jest.** Python szuka bibliotek w konkretnych katalogach. `venv` tworzy taki katalog wewnątrz folderu projektu i sprawia, że uruchomiony w nim Python widzi **tylko** biblioteki z tego katalogu. Dzięki temu eksperymenty w jednym projekcie nie psują drugiego.

To koniec teorii środowisk — teraz konkret.

---

## Środowisko pracy krok po kroku

### 1. Utworzenie katalogu projektu

Zacznij od pustego katalogu, np. `sqlalchemy-course`.

```bash
mkdir sqlalchemy-course
cd sqlalchemy-course
```

### 2. Utworzenie środowiska wirtualnego

```bash
python -m venv .venv
```

Polecenie `python -m venv .venv` mówi: „uruchom moduł `venv` i utwórz środowisko w katalogu `.venv`”. Nazwa `.venv` to konwencja (kropka na początku ukrywa katalog na systemach uniksowych), ale możesz użyć dowolnej.

> 🧪 **Ćwiczenie — rozpoznanie.** Sprawdź, jaka wersja Pythona jest używana: `python --version` (na Windows czasem `py --version`). Kurs zakłada **3.11 lub nowszy**.

### 3. Aktywacja środowiska

Aktywacja sprawia, że w bieżącej sesji terminala `python` i `pip` wskazują na wersje z `.venv`.

**Linux / macOS (bash, zsh):**

```bash
source .venv/bin/activate
```

**Windows (PowerShell):**

```powershell
.venv\Scripts\Activate.ps1
```

**Windows (cmd.exe):**

```cmd
.venv\Scripts\activate.bat
```

Po aktywacji w wierszu terminala pojawi się zwykle prefiks `(.venv)`. To znak, że jesteś w środku szuflady.

> ⚠️ **Pułapka — aktywacja działa tylko w bieżącym oknie terminala.** Otwarcie nowego okna nie dziedziczy aktywacji. Jeśli komenda `pip install` „instaluje globalnie” zamiast do projektu, prawdopodobnie zapomniałeś aktywować środowiska.

> ⚠️ **Pułapka — PowerShell i polityka wykonywania skryptów.** Na Windowsie aktywacja przez PowerShell może zwrócić błąd `... cannot be loaded because running scripts is disabled`. Wtedy uruchom jednorazowo:

```powershell
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
```

### 4. Instalacja SQLAlchemy

```bash
pip install "SQLAlchemy>=2.0"
```

Cudzysłowy wokół `SQLAlchemy>=2.0` są ważne — bez nich niektóre powłoki (np. `zsh` czy `cmd`) zinterpretują znak `>` jako przekierowanie wyjścia do pliku. Cudzysłów mówi powłoce: „to jest jeden argument dla `pip`”.

Od razu doinstalujmy to, co przyda się w kursie:

```bash
pip install alembic pytest
```

- `alembic` — narzędzie do migracji schematu bazy (zmian w strukturze tabel) — moduł `16`.
- `pytest` — framework do testów — moduł `18`.

Sterowniki baz (na razie opcjonalne, przydadzą się później):

```bash
# PostgreSQL (nowoczesny sterownik z gotowymi binariami)
pip install "psycopg[binary]"

# Wersja asynchroniczna (moduł 15)
pip install asyncpg

# Async dla SQLite (moduł 15)
pip install aiosqlite
```

> 🧠 **Dlaczego `[binary]`?** `psycopg` w wersji źródłowej wymaga kompilacji części C, co na niektórych systemach kończy się błędami braku kompilatora. Zapis `psycopg[binary]` instaluje gotowe binaria — czyli „zmontowane” już pakiety. To jak kupić mebel złożony zamiast w paczce z instrukcją.

> 🆕 **SQLAlchemy 2.1 — zmiana domyślnego sterownika.** W 2.0, gdy piszesz `postgresql://`, SQLAlchemy sięga po `psycopg2`, jeśli jest zainstalowany. W 2.1 domyślnym sterownikiem dla `postgresql://` jest `psycopg` (wersja 3), a dla `oracle://` — `oracledb`. Jeśli więc przejdziesz na 2.1 i masz stary kod oparty na `psycopg2`, jawnie dopisz sterownik w URL-u (patrz sekcja o URL-ach) albo doinstaluj `psycopg`.

> 🆕 **SQLAlchemy 2.1 — `greenlet` nie instaluje się sam.** W 2.0 pakiet `sqlalchemy[asyncio]` wciągał `greenlet` automatycznie (był potrzebny do mostu sync↔async). W 2.1 musisz go doinstalować jawnie: `pip install "sqlalchemy[asyncio]"`. Zapamiętaj to na moduł `15`.

### 5. Sprawdzenie wersji

```python
# examples/02_check_version.py
import sqlalchemy

print("Wersja SQLAlchemy:", sqlalchemy.__version__)
```

```bash
python examples/02_check_version.py
# Wersja SQLAlchemy: 2.0.36   (u Ciebie może być inna wersja punktowa)
```

Dodatkowo możesz sprawdzić, jakie dialekty i sterowniki są dostępne:

```python
# examples/02_dialects.py
from sqlalchemy.dialects import registry

for name in ("sqlite", "postgresql", "mysql", "oracle"):
    try:
        d = registry.load(name)
        print(f"{name:12} -> {d}")
    except Exception as exc:  # brak sterownika to nie błąd kursu
        print(f"{name:12} -> niedostępny ({exc.__class__.__name__})")
```

> 💡 **Analogia — dialekt to „język” danej bazy.** SQLAlchemy potrafi rozmawiać z SQLite, PostgreSQL, MySQL i wieloma innymi. Każda z nich mówi nieco innym dialektem SQL-a. Moduł ładujący ten dialekt to właśnie „tłumacz” z modułu `01`.

### 6. `requirements.txt` i `pyproject.toml`

Aby odtworzyć środowisko na innym komputerze, zapisujemy listę bibliotek.

**Wariant klasyczny — `requirements.txt`:**

```text
SQLAlchemy>=2.0,<2.1
alembic>=1.13
pytest>=8.0
```

Zapisanie aktualnie zainstalowanych wersji:

```bash
pip freeze > requirements.txt
```

Odtworzenie na nowej maszynie:

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

**Wariant nowocześniejszy — `pyproject.toml`:**

```toml
# pyproject.toml
[project]
name = "sqlalchemy-course"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "SQLAlchemy>=2.0,<2.1",
]

[project.optional-dependencies]
dev = [
    "alembic>=1.13",
    "pytest>=8.0",
]
postgres = [
    "psycopg[binary]>=3.1",
]
```

Instalacja z tym plikiem:

```bash
pip install -e ".[dev]"
```

> 🧠 **Dlaczego przypinamy górną granicę (`<2.1`) w kursie?** Bo SQLAlchemy 2.1 wprowadza zmiany (opisane w ramkach 🆕). W projekcie produkcyjnym decyzja należy do Ciebie: albo przypinasz wersję dla powtarzalności, albo pozwalasz na aktualizacje i regularnie testujesz. Kurs uczy na 2.0, a różnice sygnalizuje.

> ⚠️ **Pułapka — `pip install sqlalchemy` (małą literą) działa, ale...** Nazwy pakietów w `pip` są niewrażliwe na wielkość liter, więc instalacja przejdzie. Jednak w kodzie musisz importować `import sqlalchemy` — małymi literami. Nie mieszaj zapisu w dokumentacji: oficjalna nazwa projektu to **SQLAlchemy**, moduł importujemy jako `sqlalchemy`.

---

## `Engine` — serce komunikacji z bazą

Czas na najważniejszy obiekt w tym module.

`Engine` (silnik) to obiekt, który **zarządza komunikacją** z bazą danych. Nie jest połączeniem — jest **fabryką połączeń** i centralnym punktem konfiguracji.

> 💡 **Analogia — `Engine` to rozdzielnia elektryczna i silnik w jednym.** W budynku nie podłączasz każdego gniazdka bezpośrednio do elektrowni. Jest rozdzielnia, która wie, skąd czerpać prąd, ile obwodów utrzymać i jak je zabezpieczyć. `Engine` to taka rozdzielnia dla Twojej bazy: zna adres bazy, trzyma zestaw gotowych połączeń (pulę) i wydaje je na żądanie.

> 💡 **Druga analogia — `Engine` to centrala wypożyczalni rowerów.** Nie jeździsz na centrali. Przychodzisz, bierzesz rower (połączenie), jedziesz, oddajesz. Centrala (Engine) pilnuje, ile rowerów jest wolnych, ile wypożyczonych, i dba o ich stan techniczny.

### Trzy poziomy: `Engine`, `Connection`, `Session`

To miejsce, w którym początkujący najczęściej się gubią, bo trzy rzeczy brzmią podobnie. Rozróżnijmy je raz na zawsze.

| Obiekt | Co to jest | Analogia | Kiedy używać |
|---|---|---|---|
| `Engine` | Fabryka i konfiguracja połączeń. Tworzysz **raz na aplikację**. | Centrala wypożyczalni / rozdzielnia | Zawsze — jako punkt startowy |
| `Connection` | Jedno konkretne, otwarte połączenie do bazy. | Wypożyczony rower | Gdy chcesz świadomie zarządzać transakcją „ręcznie” (Core) |
| `Session` | Warstwa ORM nad połączeniem — „notes” zmian na obiektach. | Obsługa wypożyczalni z notesem zamówień | W pracy z ORM (moduł `08`) |

Diagram warstw:

```text
Twój kod w Pythonie
        │
        ├── (styl Core)  Engine ──► Connection ──► DBAPI ──► baza danych
        │
        └── (styl ORM)   Engine ──► Session ──► Connection ──► DBAPI ──► baza
```

> 🧠 **Dlaczego `Engine` tworzymy raz, a nie w każdej funkcji?** Bo `Engine` trzyma **pulę połączeń** (zestaw gotowych do użycia połączeń) oraz konfigurację. Utworzenie nowego `Engine` dla każdego zapytania oznacza: nową pulę, nowe połączenia, nowe uzgadnianie z bazą — cały narzut techniczny od zera. To jakby wypożyczalnia budowała nowy budynek za każdym razem, gdy ktoś chce rower.

> ⚠️ **Pułapka — `Engine` w każdej funkcji.** Ten błąd pojawia się bardzo często w kodzie początkujących:

```python
# ŹLE — nowa pula przy każdym wywołaniu
def get_user():
    engine = create_engine("sqlite:///app.db")  # tworzy nową pulę!
    with engine.connect() as conn:
        return conn.execute(...).all()
```

```python
# DOBRZE — jeden Engine na całą aplikację (moduł na poziomie)
# examples/02_engine_singleton.py
from sqlalchemy import create_engine

engine = create_engine("sqlite:///app.db")  # jeden raz


def get_user(user_id: int):
    with engine.connect() as conn:
        return conn.execute(...).all()
```

---

## URL-e połączeń — adres bazy danych

Zanim `Engine` cokolwiek zrobi, musi wiedzieć, **z którą bazą rozmawiać**. Adres przekazujemy jako tekst nazywany **URL-em połączenia**.

### Anatomia URL-a

Ogólny wzór:

```text
dialekt+sterownik://użytkownik:hasło@host:port/nazwa_bazy?opcje
```

Przykład rozłożony na części:

```text
postgresql+psycopg://alice:sekret@localhost:5432/sklep?sslmode=require
└────┬───┘└──┬───┘ └─┬─┘ └──┬─┘ └──┬───┘└─┬──┘└──┬─┘ └──────┬───────┘
 dialekt   sterownik  user  hasło  host  port   baza       opcje
```

| Składnik | Znaczenie | Przykład |
|---|---|---|
| `dialekt` | Rodzaj bazy (silnik bazy danych) | `sqlite`, `postgresql`, `mysql` |
| `+sterownik` | Konkretny konektor (DBAPI) | `+psycopg`, `+asyncpg`, `+aiosqlite` |
| `użytkownik:hasło` | Dane logowania (nie zawsze potrzebne) | `alice:sekret` |
| `host:port` | Gdzie działa baza | `localhost:5432` |
| `nazwa_bazy` | Nazwa konkretnej bazy w serwerze | `sklep` |
| `opcje` | Dodatkowe parametry po `?` | `sslmode=require` |

> 💡 **Analogia — URL to adres pocztowy bazy.** `dialekt+sterownik` to rodzaj transportu (pociąg, samolot), `host:port` to miasto i nr lokalu, `nazwa_bazy` to numer mieszkania, `hasło` to klucz do drzwi.

### SQLite — najprostszy przypadek

SQLite nie ma serwera, hosta ani hasła. Cała baza to jeden plik.

| URL | Co powstanie |
|---|---|
| `sqlite:///biblioteka.db` | Plik `biblioteka.db` w bieżącym katalogu |
| `sqlite:///data/biblioteka.db` | Plik w podkatalogu `data` |
| `sqlite:////home/user/biblioteka.db` | **Bezwzględna** ścieżka (cztery ukośniki!) |
| `sqlite://` | Baza **w pamięci RAM** (ulotna, znika po zamknięciu) |
| `sqlite:///:memory:` | Baza w pamięci RAM (jawna deklaracja) |

> ⚠️ **Pułapka — jedna ukośnik czy trzy?** To najczęstsza literówka przy SQLite:
>
> - `sqlite://` → baza **w pamięci** (nie plik!)
> - `sqlite:///foo.db` → plik `foo.db` (trzy ukośniki = „+ ścieżka względna”)
> - `sqlite:////abs/path.db` → plik o ścieżce **bezwzględnej** (cztery ukośniki)
>
> Mechanika jest prosta, gdy zapamiętasz: część po `sqlite://` to ścieżka pliku. Jeśli ścieżka zaczyna się od `/`, oznacza to położenie na dysku (a więc musisz mieć `//` w URL + `/` w ścieżce = trzy ukośniki).

> 🧠 **Dlaczego baza w pamięci jest ulotna i czemu jest przydatna?** Żyje tylko w RAM-ie procesu, który ją utworzył. Świetnie nadaje się do testów (moduł `18`) i eksperymentów — jest błyskawiczna i nie zostawia śmieci. Nie nadaje się do przechowywania czegokolwiek trwale.

### PostgreSQL i MySQL

```text
postgresql+psycopg://user:password@localhost:5432/mydb
postgresql+psycopg://user:password@localhost:5432/mydb?sslmode=require
mysql+pymysql://user:password@127.0.0.1:3306/mydb?charset=utf8mb4
```

> 🆕 **SQLAlchemy 2.1 — domyślny sterownik PostgreSQL.** W 2.0 zapis `postgresql://user:pass@host/db` oznacza „użyj domyślnego sterownika”, którym historycznie był `psycopg2`. W 2.1 domyślnym stał się `psycopg` (wersja 3). Jeśli chcesz uniknąć niespodzianek po aktualizacji, **zawsze zapisuj sterownik wprost**, np. `postgresql+psycopg://`.

### Bezpieczeństwo: hasła z ENV, nie w kodzie

Nigdy nie wstawiaj hasła do bazy wprost w kodzie i nigdy nie commituj go do repozytorium. Kod wędruje na GitHuba, na serwery CI, na czyjeś reprodukcje — a hasło wędruje razem z nim.

**Zły wzorzec:**

```python
# ŹLE — hasło w kodzie, trafi do repozytorium
engine = create_engine("postgresql+psycopg://admin:tajne123@prod.example.com/sklep")
```

**Dobry wzorzec — zmienne środowiskowe:**

```python
# examples/02_engine_from_env.py
from __future__ import annotations

import os

from sqlalchemy import create_engine, text

# Wartości wczytywane ze środowiska; lokalnie możesz je ustawić przez .env
DB_USER = os.environ.get("DB_USER", "app")
DB_PASSWORD = os.environ.get("DB_PASSWORD", "")
DB_HOST = os.environ.get("DB_HOST", "localhost")
DB_PORT = os.environ.get("DB_PORT", "5432")
DB_NAME = os.environ.get("DB_NAME", "sklep")

if not DB_PASSWORD:
    raise SystemExit("Brak DB_PASSWORD w środowisku — ustaw ją przed uruchomieniem.")

url = f"postgresql+psycopg://{DB_USER}:{DB_PASSWORD}@{DB_HOST}:{DB_PORT}/{DB_NAME}"
engine = create_engine(url, echo=True)

with engine.connect() as conn:
    print(conn.execute(text("SELECT version()")).scalar_one())
```

Ustawienie zmiennej (Linux/macOS):

```bash
export DB_PASSWORD='sekret'
python examples/02_engine_from_env.py
```

Windows PowerShell:

```powershell
$env:DB_PASSWORD = "sekret"
python examples\02_engine_from_env.py
```

> ⚠️ **Pułapka — hasło ze znakami specjalnymi w URL-u.** Znaki takie jak `@`, `:`, `/` albo `#` mają w URL-u znaczenie techniczne. Jeśli hasło brzmi `p@ss:word`, musisz je zakodować (URL-encode): `p%40ss%3Aword`. Wygodniejsze rozwiązanie: skorzystaj z `sqlalchemy.engine.URL.create(...)`, które zrobi to za Ciebie:

```python
# examples/02_url_create.py
from sqlalchemy import create_engine
from sqlalchemy.engine import URL

url = URL.create(
    drivername="postgresql+psycopg",
    username="alice",
    password="p@ss:word",  # znaki specjalne są tu bezpieczne
    host="localhost",
    port=5432,
    database="sklep",
)
print(url.render_as_string(hide_password=False))
# postgresql+psycopg://alice:p%40ss%3Aword@localhost:5432/sklep

engine = create_engine(url)
```

---

## `create_engine()` — parametry pod lupą

`create_engine()` przyjmuje URL i zestaw parametrów konfiguracyjnych. Prześledźmy te najważniejsze.

```python
# examples/02_create_engine_params.py
from sqlalchemy import create_engine

engine = create_engine(
    "postgresql+psycopg://app:sekret@localhost:5432/sklep",

    # --- Diagnostyka ---
    echo=False,             # loguj wygenerowany SQL na stdout
    echo_pool=False,        # loguj zdarzenia puli połączeń

    # --- Pula połączeń ---
    pool_size=5,            # ile połączeń trzymać stale
    max_overflow=10,        # ile dodatkowych wolno utworzyć chwilowo
    pool_timeout=30,        # ile sekund czekać na wolne połączenie
    pool_recycle=1800,      # odświeżaj połączenie po 30 minutach
    pool_pre_ping=True,     # sprawdź połączenie przed użyciem

    # --- Specyficzne dla sterownika ---
    connect_args={"connect_timeout": 5},
)
```

### `echo` i `echo_pool`

- `echo=True` — SQLAlchemy wypisuje na konsolę **każdy** SQL, który wysyła do bazy, oraz parametry. Nieocenione przy nauce i debugowaniu.
- `echo="debug"` — jak powyżej, ale z jeszcze większą ilością szczegółów.
- `echo_pool=True` — dodatkowo loguje zdarzenia puli połączeń (wypożyczenie, zwrot, utworzenie).

> ⚠️ **Pułapka — `echo=True` zostawione na produkcji.** To nie tylko zaśmieca logi, ale w niektórych konfiguracjach może **ujawnić dane wrażliwe** (parametry zapytań) w plikach logów. W środowisku produkcyjnym `echo` musi być `False`; szczegółowość reguluje się przez system logowania (sekcja niżej).

### `pool_size`, `max_overflow`, `pool_timeout`

Wyobraź sobie parking. `pool_size=5` to pięć stałych miejsc. Jeśli przyjdzie szósty samochód, tymczasowo dostawiamy krzesło obok (`max_overflow=10`) — razem maksymalnie 15 miejsc. Gdy i one się skończą, kolejny kierowca czeka maksymalnie `pool_timeout` sekund, po czym rezygnuje z błędem.

Szczegóły w sekcji o puli połączeń niżej.

### `pool_pre_ping`

Bazy danych i sieci zamykają bezczynne połączenia (np. po 5 minutach ciszy). Połączenie w puli może więc być „martwe”, mimo że wygląda na aktywne. `pool_pre_ping=True` sprawia, że SQLAlchemy **przed** wydaniem połączenia wykonuje lekkie zapytanie kontrolne (ping). Jeśli połączenie jest martwe, jest wymieniane na świeże.

> 💡 **Analogia — `pool_pre_ping` to zajrzenie do baku przed jazdą.** Krótki test, który chroni przed „autem, które nie odpali”.

> ⚠️ **Pułapka — brak `pool_pre_ping` przy długich przerwach.** Aplikacja działa całą noc bez ruchu, a rano użytkownik dostaje błąd `server closed the connection unexpectedly`. To klasyk. `pool_pre_ping=True` niemal zawsze opłaca się włączyć — koszt to jedno tanie zapytanie na wypożyczenie.

### `pool_recycle`

Kolejna defensywa: po `pool_recycle=1800` sekundach połączenie jest zamykane i tworzone od nowa, nawet jeśli wygląda zdrowo. Ustaw wartość **mniejszą** niż timeout bezczynności serwera bazy (np. jeśli baza zamyka połączenia po godzinie, ustaw 1800 = pół godziny).

### `future` — parametr, który stał się zbędny

W SQLAlchemy 1.4 wprowadzono przejściowy parametr `future=True`, który włączał „nowy styl” API 2.0. W 2.0 stał się on domyślny, więc **nie musisz go używać**. Możesz go spotkać w starych przykładach w internecie:

```python
# Relikt z 1.4 — w 2.0 niepotrzebne, ale nie zaszkodzi
engine = create_engine("sqlite:///app.db", future=True)
```

> 🧠 **Dlaczego o nim wspominamy?** Bo bardzo dużo materiałów w sieci powstało przed 2.0. Jeśli widzisz `future=True` albo `Session(engine, future=True)`, to znak, że przykład jest sprzed paru lat. W kursie go nie używamy — w 2.0 nowy styl jest po prostu stylem.

### `connect_args`

Sterowniki mają własne parametry specyficzne, których SQLAlchemy nie zna. Przekazujesz je przez słownik `connect_args`.

**SQLite i wątki:**

```python
# examples/02_sqlite_threads.py
from sqlalchemy import create_engine

engine = create_engine(
    "sqlite:///app.db",
    connect_args={"check_same_thread": False},
)
```

> ⚠️ **Pułapka — `check_same_thread=False` bez zrozumienia.** Wbudowany moduł `sqlite3` domyślnie pilnuje, by połączenie było używane tylko w tym wątku, w którym je utworzono (`check_same_thread=True`). Ustawienie `False` wyłącza tę kontrolę. Robimy to czasem w aplikacjach webowych (wiele wątków obsługuje żądania), ale **tylko** wtedy, gdy wiemy, że faktycznie chcemy współdzielić połączenie — i że nie doprowadzi to do równoczesnego pisania z dwóch wątków (SQLite tego nie lubi). Nie traktuj tego jako „naprawy błędu wątków”, ale jako **świadomą decyzję projektową**.

---

## Pula połączeń (connection pool)

### Dlaczego pula istnieje

Nawiązanie połączenia z bazą danych jest **kosztowne**. Trzeba: otworzyć gniazdo sieciowe, zalogować się (hasło, uprawnienia), uzgodnić parametry sesji, ustawić kodowanie i strefę czasową. Na dobrych łączach to kilka–kilkanaście milisekund, na słabych — znacznie więcej. Jeśli każde zapytanie otwierałoby nowe połączenie, aplikacja traciłaby mnóstwo czasu na samo „łączenie się”, zamiast na pracę.

**Pula połączeń** rozwiązuje to inaczej: tworzymy kilka połączeń raz, a potem **wypożyczamy i zwracamy** je wielokrotnie.

> 💡 **Analogia — wypożyczalnia rowerów.** Zamiast kupować nowy rower na każdą przejażdżkę i wyrzucać go po powrocie, wypożyczalnia ma stały zapas. Klient bierze, jedzie, oddaje. Kolejny klient bierze ten sam rower. Nikt nie traci czasu na produkcję roweru za każdym razem.

### Typy pul w SQLAlchemy

| Klasa puli | Zachowanie | Kiedy używać |
|---|---|---|
| `QueuePool` | Standardowa pula z kolejką i limitem; domyślna na produkcji | PostgreSQL, MySQL — bazy sieciowe |
| `NullPool` | Nie trzyma połączeń — każde otwiera i zamyka | Gdy chcesz mieć pewność co do liczby połączeń, np. przy testach lub z PgBouncer |
| `StaticPool` | Jedno współdzielone połączenie dla wszystkiego | Klasycznie do SQLite `:memory:` w testach |
| `SingletonThreadPool` | Jedno połączenie na wątek | Domyślnie: SQLite w pamięci — każde połączenie widzi tę samą bazę |
| `AssertionPool` | Dopuszcza najwyżej jedno aktywne połączenie; krzyczy, gdy jest więcej | Diagnostyka, testy „czy dobrze zwalniam połączenia” |

> 🔬 **Pod maską — sprawdź, jaką pulę ma Twój `Engine`.** Nie zgaduj, po prostu zapytaj:

```python
# examples/02_pool_type.py
from sqlalchemy import create_engine

memory_engine = create_engine("sqlite://")
file_engine = create_engine("sqlite:///app.db")

print(type(memory_engine.pool).__name__)  # zwykle SingletonThreadPool
print(type(file_engine.pool).__name__)    # zależnie od wersji — QueuePool lub NullPool
print(file_engine.pool.status())
# np. 'Pool size: 5  Connections in pool: 1  Current Overflow: -4  Checked out: 0'
```

> 🧠 **Dlaczego in-memory SQLite ma inną pulę?** Baza `:memory:` istnieje **tylko tak długo, jak istnieje połączenie, które ją utworzyło**. Gdyby pula wydała dwa różne połączenia, każde widziałoby **inną**, pustą bazę. Dlatego dla `:memory:` SQLAlchemy dba, by połączenie było współdzielone.

### Odczyt stanu puli

`engine.pool.status()` zwraca tekstowy opis: ile jest połączeń w puli, ile wypożyczonych, jaki jest bieżący „overflow”. To pierwsze narzędzie diagnostyczne, gdy podejrzewasz, że połączenia nie są zwalniane.

```python
# examples/02_pool_status.py
from sqlalchemy import create_engine, text

engine = create_engine("sqlite:///app.db", pool_size=3, max_overflow=2)

print("Przed:", engine.pool.status())

with engine.connect() as conn:
    conn.execute(text("SELECT 1"))
    print("W trakcie:", engine.pool.status())  # jedno połączenie wypożyczone

print("Po:", engine.pool.status())  # połączenie wróciło do puli
```

> ⚠️ **Pułapka — połączenia nigdy nie wrócone.** Połączenie wraca do puli, gdy blok `with ...:` się kończy. Jeśli gdzieś pobierzesz połączenie „po cichu”, bez `with`, musi być ono jawnie zamknięte (`conn.close()`). Ilość wypożyczonych połączeń będzie rosnąć, aż pula się wyczerpie, a aplikacja zacznie rzucać `TimeoutError: QueuePool limit of size ... overflow ... reached`. Diagnostyczne pierwszą myślą powinno być: „gdzie zapomniałem zamknąć połączenie?”.

---

## Wykonywanie SQL: `connect`, `begin`, `text()`

Nadszedł czas na pierwszy SQL.

### `engine.connect()` — świadome zarządzanie transakcją

```python
# examples/02_connect_example.py
from sqlalchemy import create_engine, text

engine = create_engine("sqlite:///first.db")

with engine.connect() as conn:
    result = conn.execute(text("SELECT 1 AS one"))
    row = result.one()
    print(row)  # (1,)
```

`engine.connect()` wypożycza z puli jedno połączenie i zwraca je w kontekście (`with`). Po wyjściu z bloku połączenie jest **zwracane do puli**.

### `engine.begin()` — automatyczna transakcja

>` 💡 **Czym jest transakcja?** To „grupa operacji, która albo wykona się cała, albo wcale”. Klasyczny przykład bankowy: przelew to dwie operacje (zmniejsz saldo na jednym koncie, zwiększ saldo na drugim). Nie chcemy sytuacji, w której wykona się tylko pierwsza. Transakcja gwarantuje, że obie się powiodą albo żadna — nawet jeśli w połowie nastąpi awaria. SQLAlchemy domyślnie opakowuje operacje w transakcje.

```python
# examples/02_begin_example.py
from sqlalchemy import create_engine, text

engine = create_engine("sqlite:///first.db")

with engine.begin() as conn:
    conn.execute(text("CREATE TABLE IF NOT EXISTS notes (id INTEGER PRIMARY KEY, body TEXT)"))
    conn.execute(text("INSERT INTO notes (body) VALUES (:b)"), {"b": "pierwsza notatka"})
# ...transakcja została zatwierdzona (COMMIT) automatycznie — po wyjściu z bloku
```

> 🔬 **Pod maską — co poszło do bazy.** Blok `engine.begin()` wykonuje w przybliżeniu:

```sql
BEGIN;   -- po wejściu do bloku
CREATE TABLE IF NOT EXISTS notes (id INTEGER PRIMARY KEY, body TEXT);
INSERT INTO notes (body) VALUES (?);
COMMIT;  -- po wyjściu z bloku, jeśli nie było wyjątku
```

Jeśli w bloku `engine.begin()` zostanie rzucony wyjątek, SQLAlchemy wykona `ROLLBACK` — a więc „cofnięcie transakcji” — i dopiero przepuści wyjątek dalej.

### `connect` vs `begin` — która różnica

- `engine.connect()` — po prostu połączenie. **Autobegin** włącza transakcję leniwie przy pierwszej operacji (patrz niżej).
- `engine.begin()` — połączenie **plus jawna, zarządzana transakcja**. Po sukcesie COMMIT, po błędzie ROLLBACK. Dla większości operacji zapisu to właśnie tego chcesz.

> 🧠 **Dlaczego autobegin?** W SQLAlchemy 2.0 każda sesja/połączenie „domyślnie jest w transakcji”. Gdy tylko wykonasz pierwsze zapytanie, SQLAlchemy cicho wyśle `BEGIN`. Dlatego nie musisz pisać `BEGIN` ręcznie. Ta cisza może zaskoczyć, więc nazywamy ją tu wprost: to jest **autobegin** — automatyczne rozpoczęcie transakcji.

> ⚠️ **Pułapka — długo trzymane połączenie.** Otwarcie połączenia raz, na cały czas życia aplikacji, jest złym pomysłem:

```python
# ŹLE — połączenie trzymane przez cały czas działania aplikacji
conn = engine.connect()  # wypożyczone raz i nigdy nie zwrócone
while True:
    conn.execute(text("SELECT 1"))
```

Po pierwsze: pula ubożeje o to połączenie. Po drugie: bazy danych mają własne limity na czas życia transakcji i bezczynności — połączenie trzymające otwartą transakcję godzinami to prosta droga do blokad.

### `text()` — SQL, który piszesz sam

`text()` opakowuje surowy tekst SQL w obiekt, który SQLAlchemy rozumie i umie bezpiecznie przekazać do bazy.

```python
# examples/02_text_example.py
from sqlalchemy import create_engine, text

engine = create_engine("sqlite://")

with engine.begin() as conn:
    conn.execute(text("CREATE TABLE book (id INTEGER PRIMARY KEY, title TEXT, pages INTEGER)"))
    conn.execute(
        text("INSERT INTO book (title, pages) VALUES (:title, :pages)"),
        [
            {"title": "Wiedźmin", "pages": 320},
            {"title": "Solaris", "pages": 240},
        ],
    )
    result = conn.execute(text("SELECT id, title, pages FROM book WHERE pages > :min_pages"),
                          {"min_pages": 300})
    for row in result:
        print(row)
```

> 🔬 **Pod maską — co dostanie baza.** Dla ostatniego zapytania SQLAlchemy przygotuje:

```sql
SELECT id, title, pages FROM book WHERE pages > ?
```

a parametr `:min_pages` zostanie przekazany osobno, bezpiecznie — patrz następna sekcja.

### Wstęp do `Result`

`conn.execute(...)` zwraca obiekt `Result` — to „kursor po wynikach”. Na razie zapamiętaj trzy metody:

- `.all()` — pobierz wszystkie wiersze jako listę krotek/nazwanych krotek,
- `.one()` — dokładnie jeden wiersz (jeśli zero lub więcej niż jeden → wyjątek),
- `.scalar_one()` — dokładnie jedna **kolumna** w jednym wierszu; zwraca surową wartość:

```python
# examples/02_result_intro.py
from sqlalchemy import create_engine, text

engine = create_engine("sqlite://")

with engine.connect() as conn:
    count = conn.execute(text("SELECT 1")).scalar_one()
    print(count)  # 1
```

`Result` omówimy wyczerpująco w module `04_select_i_result.md`, gdy poznamy konstruktor zapytań `select()`.

---

## Parametry wiązane i bezpieczeństwo

Jeśli z tego modułu zapamiętasz jedną rzecz o bezpieczeństwie, niech to będzie ta:

> **Nigdy nie składaj SQL-a przez f-stringi ani przez dodawanie tekstu.**

`text()` obsługuje **parametry wiązane** (bound parameters, wiązanie — przekazywanie wartości oddzielnie od struktury zapytania). Zapytanie wysyła do bazy **stały szkielet** z „dziurami” (`:nazwa`), a wartości idą kanałem pobocznym.

```python
# examples/02_bound_params.py
from sqlalchemy import create_engine, text

engine = create_engine("sqlite://")

with engine.begin() as conn:
    title = "Solaris'; DROP TABLE book; --"   # celowo złośliwy tytuł
    # POPRAWNIE — wartość przekazana jako parametr
    result = conn.execute(
        text("SELECT * FROM book WHERE title = :title"),
        {"title": title},
    )
    print(result.all())  # pusta lista — ale baza NIE została uszkodzona
```

**Dla porównania — jak NIE pisać:**

```python
# ŹLE — wstrzyknięcie SQL (SQL injection)!
title = "Solaris'; DROP TABLE book; --"
sql = f"SELECT * FROM book WHERE title = '{title}'"
conn.execute(text(sql))
```

> 🧠 **Dlaczego parametr chroni?** Bo baza najpierw **planuje** zapytanie o niezmiennej strukturze, a dopiero potem podstawia wartości jako **dane**, nie jako kod SQL. Złośliwy łańcuch `'; DROP TABLE ...` zostanie potraktowany jak zwykły tytuł, a nie jak polecenie. To jak oddzielenie formularza od adresata: sama treść listu nie zmieni adresu na kopercie.

Dozwolone warianty parametrów:

- `:nazwa` — nazwany, przekazywany jako słownik: `{"nazwa": wartość}`,
- przekazywanie słownika lub listy słowników dla wielu wierszy (`executemany`).

> 🆕 **SQLAlchemy 2.1 — t-stringi z Pythona 3.14.** Przyszłe wersje Pythona (3.14) dodają **szablony tekstowe** (t-strings) — mechanizm, który przy zachowaniu czytelności f-stringów pozwala bibliotece przejąć kontrolę nad składaniem tekstu. SQLAlchemy 2.1 potrafi z nich korzystać, dzięki czemu można pisać zapytania z wyglądem f-stringa, ale bez ryzyka wstrzyknięcia — biblioteka sama zbuduje parametry wiązane. W 2.0 tej możliwości nie ma, więc w kursie zostajemy przy `text(..., {...})`.

> ⚠️ **Pułapka — „parametryzuję przecież nazwę tabeli”.** Parametrów wiązanych **nie da się** użyć dla nazw tabel, kolumn ani fragmentów klauzul (`ORDER BY`, `ASC/DESC`). Baza musi znać strukturę przed podstawieniem wartości. Jeśli naprawdę musisz dynamicznie wybierać kolumnę (np. sortowanie), użyj jednej z metod: (a) mapowania zdefiniowanych w kodzie wartości na gotowe zapytania, (b) konstruktorów SQLAlchemy (`select()`, o których w module `04`), (c) **weryfikacji wartości względem białej listy** przed wstawieniem do tekstu.

---

## Logowanie — podsłuchiwanie SQL-a

Zamiast `echo=True` możesz skonfigurować standardowe logowanie Pythona — jest elastyczniejsze, bo pozwala filtrować poziomy, kierować logi do pliku i wyłączać je w produkcji.

```python
# examples/02_logging_setup.py
from __future__ import annotations

import logging

from sqlalchemy import create_engine, text

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s %(name)s %(levelname)s %(message)s",
)
# Interesują nas logi silnika SQLAlchemy
logging.getLogger("sqlalchemy.engine").setLevel(logging.INFO)

engine = create_engine("sqlite://")  # echo niepotrzebne — logujemy przez logging

with engine.begin() as conn:
    conn.execute(text("SELECT 1"))
```

Poziomy logowania silnika:

| Logger | Poziom | Co pokazuje |
|---|---|---|
| `sqlalchemy.engine` | `INFO` | Każdy SQL oraz liczbę zwróconych wierszy |
| `sqlalchemy.engine` | `DEBUG` | Także parametry wiązane |
| `sqlalchemy.pool` | `INFO`/`DEBUG` | Wypożyczenia, zwroty, zamykanie połączeń |
| `sqlalchemy.dialects` | `DEBUG` | Szczegóły komunikacji z dialektem |

> ⚠️ **Pułapka — `echo=True` miesza się z `logging`.** `echo=True` wypisuje logi wprost na wyjście standardowe, obok Twoich `print`. Jeśli jednocześnie skonfigurujesz `logging`, zobaczysz podwójne linie. Wybierz jedną drogę: `echo=True` do nauki i eksperymentów, `logging` do aplikacji.

> 🔬 **Pod maska — jak wygląda log.** Typowy wpis przy `INFO`:

```text
2026-09-18 12:00:03,211 sqlalchemy.engine.Engine INFO BEGIN (implicit)
2026-09-18 12:00:03,212 sqlalchemy.engine.Engine INFO SELECT 1
2026-09-18 12:00:03,212 sqlalchemy.engine.Engine INFO [raw sql] ()
2026-09-18 12:00:03,213 sqlalchemy.engine.Engine INFO COMMIT
```

Zwróć uwagę na `BEGIN (implicit)` — to właśnie **autobegin** ujawniony w logu. Słowo „implicit” (dorozumiany) mówi, że transakcja powstała automatycznie, bez Twojej jawnej komendy.

---

## Typowe problemy zależne od systemu

### Windows

1. **Ścieżki z ukośnikami.** W URL-ach używaj `/`, nie `\`:

   ```python
   # ŹLE na Windows — backslash to znak ucieczki
   create_engine("sqlite:///C:\Users\ola\app.db")

   # DOBRZE — ukośniki w przód działają na wszystkich systemach
   create_engine("sqlite:///C:/Users/ola/app.db")

   # ALTERNATYWNIE — raw string
   create_engine(r"sqlite:///C:\Users\ola\app.db")
   ```

2. **Aktywacja `venv`** — patrz sekcja o PowerShell i `Set-ExecutionPolicy`.
3. **`py` zamiast `python`** — launcher wersji Pythona; `py -3.11 -m venv .venv` bywa wygodniejsze, gdy masz kilka wersji Pythona.

### macOS

1. **Homebrew + PostgreSQL.** Jeśli instalujesz bazę przez `brew install postgresql@16`, pamiętaj o uruchomieniu usługi (`brew services start postgresql@16`) i o tym, że domyślny użytkownik to Twój login systemowy.
2. **Dwa Pythony.** Systemowy Python (Apple) i ten z `brew`/`pyenv` bywają różne. Zawsze sprawdź, którą wersję widzi `python` po aktywacji `venv`.

### Linux

1. **Brak nagłówków przy budowaniu sterowników.** Jeśli `pip install psycopg` próbuje kompilować, potrzebujesz `libpq-dev` (`sudo apt install libpq-dev`). Prościej: użyj `pip install "psycopg[binary]"`.
2. **Uprawnienia do katalogu błędów.** W Docku bywa, że kontener widzi plik SQLite jako tylko do odczytu, gdy katalog jest montowany `:ro`. Baza SQLite potrzebuje prawa **zapisu także w katalogu**, nie tylko do samego pliku — tworzy tam pliki tymczasowe.

### Wspólne — sterowniki

| Baza | Sterownik synchroniczny | Sterownik asynchroniczny |
|---|---|---|
| SQLite | `sqlite3` (wbudowany) | `aiosqlite` |
| PostgreSQL | `psycopg` (v3) lub `psycopg2` | `asyncpg` |
| MySQL | `pymysql` lub `mysqlclient` | `aiomysql` |
| Oracle | `oracledb` | `oracledb` (async) |

> ⚠️ **Pułapka — sterownik, którego nie ma.** Błąd `ModuleNotFoundError: No module named 'psycopg2'` oznacza, że URL wskazuje sterownik, którego nie zainstalowałeś. Naprawa: doinstaluj sterownik **albo** zmień go w URL (`postgresql+psycopg://`). Nie myl dialektu (`postgresql`) ze sterownikiem (`psycopg`).

---

## Pełny przykład: `first_steps.py`

Poniższy skrypt łączy wszystko: tworzy plik SQLite, wykonuje `SELECT 1`, zakłada tabelę tymczasową, wstawia dane, odczytuje je i usuwa tabelę. Wszystko z wypisaniem SQL-a.

```python
# examples/02_first_steps.py
"""Pierwsze kroki z SQLAlchemy: Engine, Connection, text() — na SQLite."""

from __future__ import annotations

from sqlalchemy import create_engine, text

# 1. Silnik — jeden na całą aplikację.
#    echo=True -> wszystkie zapytania wypisane do konsoli.
#    W pliku baza trwa między uruchomieniami; usuń first_steps.db, by zacząć od zera.
engine = create_engine("sqlite:///first_steps.db", echo=True)

# 2. Proste zapytanie bez tabeli — sprawdzamy, że komunikacja działa.
with engine.connect() as conn:
    value = conn.execute(text("SELECT 1")).scalar_one()
    print(f"SELECT 1 zwróciło: {value}")

# 3. Transakcja: utworzenie tabeli.
with engine.begin() as conn:
    conn.execute(
        text(
            """
            CREATE TABLE IF NOT EXISTS temp_notes (
                id   INTEGER PRIMARY KEY,
                body TEXT NOT NULL
            )
            """
        )
    )

# 4. Transakcja: wstawienie danych i odczyt.
with engine.begin() as conn:
    conn.execute(
        text("INSERT INTO temp_notes (body) VALUES (:body)"),
        [
            {"body": "Pierwsza notatka"},
            {"body": "Druga notatka"},
        ],
    )
    rows = conn.execute(text("SELECT id, body FROM temp_notes ORDER BY id")).all()
    for row in rows:
        print(f"#{row.id}: {row.body}")

# 5. Sprzątanie: usuń tabelę tymczasową.
with engine.begin() as conn:
    conn.execute(text("DROP TABLE temp_notes"))

# 6. Stan puli po wszystkim.
print("Stan puli:", engine.pool.status())
```

**Jak uruchomić:**

```bash
pip install "SQLAlchemy>=2.0"
python examples/02_first_steps.py
```

Oczekiwany efekt (uproszczony):

```text
SELECT 1 zwróciło: 1
... INFO BEGIN (implicit)
... INFO INSERT INTO temp_notes (body) VALUES (?)
... INFO [raw sql] ('Pierwsza notatka',)
... INFO [raw sql] ('Druga notatka',)
... INFO SELECT id, body FROM temp_notes ORDER BY id
#1: Pierwsza notatka
#2: Druga notatka
Stan puli: Pool size: ...
```

### Wariant PostgreSQL z konfiguracją przez zmienne środowiskowe

Ten sam schemat, ale dla PostgreSQL. Wymaga uruchomionej bazy i sterownika `psycopg`.

```python
# examples/02_first_steps_postgres.py
"""Pierwsze kroki z SQLAlchemy na PostgreSQL (sterownik psycopg 3)."""

from __future__ import annotations

import os

from sqlalchemy import create_engine, text
from sqlalchemy.engine import URL

url = URL.create(
    drivername="postgresql+psycopg",
    username=os.environ.get("DB_USER", "app"),
    password=os.environ.get("DB_PASSWORD", ""),
    host=os.environ.get("DB_HOST", "localhost"),
    port=int(os.environ.get("DB_PORT", "5432")),
    database=os.environ.get("DB_NAME", "sklep"),
)

engine = create_engine(
    url,
    echo=True,
    pool_pre_ping=True,
    pool_recycle=1800,
)

with engine.connect() as conn:
    print("Wersja serwera:", conn.execute(text("SELECT version()")).scalar_one())

with engine.begin() as conn:
    conn.execute(text("CREATE TEMP TABLE tmp_notes (id serial PRIMARY KEY, body text)"))
    conn.execute(text("INSERT INTO tmp_notes (body) VALUES (:b)"), {"b": "cześć"})
    print(conn.execute(text("SELECT count(*) FROM tmp_notes")).scalar_one())
# Tabela tymczasowa (TEMP) ginie sama po zamknięciu połączenia.
```

Połączenie z linią komend:

```bash
export DB_PASSWORD='sekret'
python examples/02_first_steps_postgres.py
```

> 🧠 **Uwaga o tabelach tymczasowych.** `CREATE TEMP TABLE` w PostgreSQL tworzy tabelę widoczną tylko w bieżącym połączeniu i znikającą po jego zamknięciu. Świetnie nadaje się na eksperymenty — nie śmieci w bazie. SQLite nie ma dokładnego odpowiednika, dlatego w wersji na SQLite jawnie robimy `DROP TABLE`.

---

## Podsumowanie

Zabierz ze sobą te punkty:

1. **Zawsze pracuj w `venv`.** `python -m venv .venv`, aktywacja, potem `pip install`.
2. **`Engine` to fabryka połączeń i punkt konfiguracji — twórz go raz na aplikację**, a nie w każdej funkcji.
3. **URL połączenia** ma postać `dialekt+sterownik://user:hasło@host:port/baza?opcje`. Dla SQLite: `sqlite:///plik.db` (plik) vs `sqlite://` (pamięć).
4. **Hasła trzymaj w zmiennych środowiskowych**, nigdy w kodzie. Znaki specjalne zakoduj — najłatwiej przez `URL.create(...)`.
5. **Pula połączeń** to „parking” gotowych połączeń. `pool_size`, `max_overflow`, `pool_timeout` sterują jego rozmiarem i cierpliwością.
6. **`pool_pre_ping=True`** niemal zawsze warto włączyć — chroni przed martwymi połączeniami po przerwie.
7. **`engine.begin()`** to najwygodniejszy sposób pracy z transakcją: COMMIT po sukcesie, ROLLBACK po błędzie.
8. **`text()` + parametry `:nazwa`** to jedyny bezpieczny sposób pisania surowego SQL. Nigdy nie sklejaj SQL-a f-stringami.
9. **Autobegin** — SQLAlchemy sam zaczyna transakcję przy pierwszej operacji. Widać to w logu jako `BEGIN (implicit)`.
10. **`echo` / `logging`** to Twoje lampy w ciemności. Do nauki `echo=True`, na produkcji `logging` z odpowiednim poziomem.
11. **`future=True` to relikt** z 1.4 — w 2.0 niepotrzebny.
12. **`check_same_thread=False` dla SQLite** to świadoma decyzja, nie automatyczna naprawa.

---

## Ćwiczenia

### Ćwiczenie 1 — Konfiguracja z ENV (łatwe)

Napisz skrypt, który tworzy `Engine` do SQLite, przy czym **ścieżkę do pliku bazy** pobiera ze zmiennej środowiskowej `DB_PATH`. Domyślnie (gdy zmiennej nie ma) użyj `app.db`. Skrypt powinien wypisać: typ puli, stan puli przed użyciem i po wykonaniu pojedynczego `SELECT 1`.

### Ćwiczenie 2 — `echo` a `logging` (średnie)

Przygotuj dwa uruchomienia tego samego zapytania (utworzenie tabeli i trzy wstawienia):
1. z `echo=True`,
2. z logowaniem przez `logging` na poziomie `INFO` i `DEBUG`.

Zapisz oba wyjścia do plików i porównaj: w którym widać parametry wiązane? W którym widać `BEGIN (implicit)`?

### Ćwiczenie 3 — Diagnostyka puli (trudniejsze)

Napisz skrypt, który:
1. tworzy `Engine` do SQLite (`sqlite:///pool_demo.db`) z `pool_size=2`, `max_overflow=1`,
2. wypożycza połączenia jedno po drugim w pętli i próbuje wypożyczyć więcej, niż pozwala pula,
3. po każdej próbie wypisuje `engine.pool.status()`.

Co się dzieje, gdy przekroczysz łączny limit (`pool_size + max_overflow`)? Jak zmieni się zachowanie po ustawieniu `pool_timeout=1`?

### Rozwiązania

<details>
<summary><strong>Rozwiązanie 1 — konfiguracja z ENV</strong></summary>

```python
# examples/02_ex_solution_1.py
from __future__ import annotations

import os

from sqlalchemy import create_engine, text

db_path = os.environ.get("DB_PATH", "app.db")
engine = create_engine(f"sqlite:///{db_path}", echo=False)

print("Typ puli:", type(engine.pool).__name__)
print("Przed:", engine.pool.status())

with engine.connect() as conn:
    conn.execute(text("SELECT 1"))

print("Po:", engine.pool.status())
```

Uruchomienie z domyślną ścieżką:

```bash
python examples/02_ex_solution_1.py
```

Uruchomienie z własną ścieżką:

```bash
DB_PATH=/tmp/test.db python examples/02_ex_solution_1.py
```

**Dlaczego tak?** Oddzielenie konfiguracji (gdzie jest baza) od kodu pozwala uruchamiać ten sam skrypt w różnych środowiskach bez edycji pliku. To fundament konfiguracji aplikacji (rozszerzymy to w module `22`).

**Kompromis:** `f"sqlite:///{db_path}"` jest czytelne, ale kruche, gdy `db_path` zawiera znaki specjalne. Dla SQLite to zwykle nieszkodliwe, ale dla PostgreSQL używaj `URL.create(...)`.

</details>

<details>
<summary><strong>Rozwiązanie 2 — `echo` vs `logging`</strong></summary>

```python
# examples/02_ex_solution_2_echo.py
from __future__ import annotations

from sqlalchemy import create_engine, text

engine = create_engine("sqlite:///ex2_echo.db", echo=True)

with engine.begin() as conn:
    conn.execute(text("CREATE TABLE IF NOT EXISTS t (id INTEGER PRIMARY KEY, v TEXT)"))
    conn.execute(text("INSERT INTO t (v) VALUES (:v)"), [{"v": "a"}, {"v": "b"}, {"v": "c"}])
```

```python
# examples/02_ex_solution_2_logging.py
from __future__ import annotations

import logging

from sqlalchemy import create_engine, text

logging.basicConfig(
    filename="ex2_logging.log",
    level=logging.DEBUG,
    format="%(asctime)s %(name)s %(levelname)s %(message)s",
)
logging.getLogger("sqlalchemy.engine").setLevel(logging.DEBUG)

engine = create_engine("sqlite:///ex2_logging.db")

with engine.begin() as conn:
    conn.execute(text("CREATE TABLE IF NOT EXISTS t (id INTEGER PRIMARY KEY, v TEXT)"))
    conn.execute(text("INSERT INTO t (v) VALUES (:v)"), [{"v": "a"}, {"v": "b"}, {"v": "c"}])
```

**Odpowiedź na pytanie z zadania:**

- **Parametry wiązane** zobaczysz wyraźnie w wariancie z `logging` ustawionym na `DEBUG` (w logu pojawi się linia z krotką wartości, np. `('a',)`). W wariancie `echo=True` zobaczysz sam SQL, ale parametry bywają prezentowane mniej czytelnie.
- **`BEGIN (implicit)`** pojawia się w obu wariantach, ale tylko przy odpowiednim poziomie szczegółowości. W wariancie `logging` na `INFO` zwykle zobaczysz je od razu.

**Pułapka powiązana.** `echo=True` i `logging` jednocześnie dają **podwójne** wpisy — jak dwa niezależne dyktafony nagrywające tę samą rozmowę. Wybierz jedno źródło.

</details>

<details>
<summary><strong>Rozwiązanie 3 — diagnostyka puli</strong></summary>

```python
# examples/02_ex_solution_3.py
from __future__ import annotations

from sqlalchemy import create_engine, text

engine = create_engine(
    "sqlite:///pool_demo.db",
    pool_size=2,
    max_overflow=1,
    pool_timeout=1,   # zmień na 30, by porównać zachowanie
    echo=False,
)

conns = []
try:
    for i in range(5):
        try:
            conn = engine.connect()
            conns.append(conn)
            print(f"[{i}] wypożyczono -> {engine.pool.status()}")
        except Exception as exc:
            print(f"[{i}] BŁĄD: {type(exc).__name__}: {exc}")
        finally:
            print(f"[{i}] status: {engine.pool.status()}")
finally:
    for c in conns:
        c.close()
    print("Po zwrocie:", engine.pool.status())
```

**Co się dzieje?** Łączny limit to `pool_size + max_overflow = 3`. Czwarte wypożyczenie poczeka `pool_timeout` sekund, a potem rzuci `TimeoutError: QueuePool limit of size 2 overflow 1 reached, connection timed out`.

**Eksperyment z `pool_timeout`:**

- `pool_timeout=30` — program zawiesi się na ok. 30 sekund, zanim się podda.
- `pool_timeout=1` — szybkie, wyraźne odrzucenie.

**Pułapka powiązana.** W prawdziwej aplikacji ten błąd znaczy jedno z dwóch: albo pula jest za mała dla ruchu (zwiększ `pool_size`), albo **gdzieś wycieka połączenie** i nie wraca do puli. Diagnostyka: obserwuj rosnącą liczbę „Checked out” w `pool.status()` na przestrzeni czasu.

</details>

---

## Najczęstsze błędy i jak je czytać

| Komunikat / objaw | Przyczyna | Naprawa |
|---|---|---|
| `ModuleNotFoundError: No module named 'sqlalchemy'` | Środowisko `venv` nieaktywne lub biblioteka nie zainstalowana | Aktywuj `venv`, `pip install "SQLAlchemy>=2.0"` |
| `ModuleNotFoundError: No module named 'psycopg2'` | URL wskazuje sterownik, którego nie ma | Doinstaluj sterownik lub zmień na `postgresql+psycopg://` |
| `OperationalError: unable to open database file` (SQLite) | Ścieżka do pliku nie istnieje lub katalog jest bez prawa zapisu | Utwórz katalog lub popraw URL; sprawdź uprawnienia |
| `OperationalError: database is locked` (SQLite) | Dwa procesy/wątki piszą równocześnie | Rozważ WAL, pojedynczego pisarza lub przejście na PostgreSQL |
| `SQLite objects created in a thread can only be used in that same thread` | Używasz jednego połączenia w różnych wątkach | Nie dziel połączeń między wątkami; ewentualnie świadomie `check_same_thread=False` |
| `TimeoutError: QueuePool limit of size ... reached` | Wyczerpana pula lub wyciek połączeń | Sprawdź, gdzie brakuje `close()`/`with`; zwiększ pulę świadomie |
| `server closed the connection unexpectedly` | Bezczynne połączenie zamknięte po stronie serwera | Włącz `pool_pre_ping=True` i ustaw `pool_recycle` < timeout serwera |
| `ArgumentError: Could not parse SQLAlchemy URL` | Literówka w URL (np. `postresql://`, brak `://`) | Popraw dialekt i format; użyj `URL.create(...)` |
| `InvalidRequestError: This session is in 'prepared' state` | Zły stan sesji po błędzie transakcji (moduł `08`) | Wykonaj `rollback()` i powtórz operację |
| Hasło w logach / na GitHubie | Sekret w kodzie | Przenieś do zmiennych środowiskowych, unieważnij ujawnione hasło |
| `ValueError: invalid literal for int()` przy `DB_PORT` | Zmienna środowiskowa to tekst, nie liczba | Konwertuj: `int(os.environ["DB_PORT"])` |
| Podwójne linie w konsoli | `echo=True` i `logging` jednocześnie | Wybierz jeden kanał logowania |
| `SAWarning: The Engine is being garbage collected` | `Engine` tworzony tymczasowo i porzucany | Trzymaj jeden `Engine` na aplikację |

---

## Słowniczek modułu

| Termin (EN) | Polski odpowiednik | Wyjaśnienie |
|---|---|---|
| `Engine` | silnik | Fabryka połączeń i punkt konfiguracji; tworzony raz na aplikację |
| `Connection` | połączenie | Jedno otwarte połączenie do bazy, wypożyczane z puli |
| connection pool | pula połączeń | Zestaw gotowych połączeń wielokrotnego użytku |
| driver / DBAPI | sterownik / konektor | Biblioteka Pythona tłumacząca wywołania na protokół bazy |
| dialect | dialekt | Moduł SQLAlchemy znający specyfikę konkretnej bazy |
| URL połączenia | adres połączenia | Tekst opisujący, jak i gdzie połączyć się z bazą |
| `text()` | konstruktor surowego SQL | Opakowuje ręcznie napisany SQL w obiekt dla SQLAlchemy |
| bound parameter | parametr wiązany | Wartość przekazana osobno od struktury zapytania (`:nazwa`) |
| SQL injection | wstrzyknięcie SQL | Atak polegający na wstawieniu kodu SQL przez dane wejściowe |
| transaction | transakcja | Grupa operacji wykonana cała albo wcale |
| `COMMIT` | zatwierdzenie | Zamknięcie transakcji z zapisem zmian |
| `ROLLBACK` | wycofanie | Cofnięcie całej transakcji |
| autobegin | automatyczne rozpoczęcie | Ciche rozpoczęcie transakcji przy pierwszej operacji |
| `pool_size` | rozmiar puli | Liczba stale utrzymywanych połączeń |
| `max_overflow` | nadmiar | Dodatkowe, chwilowe połączenia ponad `pool_size` |
| `pool_pre_ping` | test połączenia | Sprawdzenie żywotności połączenia przed wydaniem |
| `pool_recycle` | recykling | Czas, po którym połączenie jest odświeżane |
| `NullPool` / `StaticPool` | — | Pule o innym zachowaniu: bez trzymania połączeń / jedno współdzielone |
| `venv` | środowisko wirtualne | Izolowany zestaw bibliotek dla projektu |
| `executemany` | wykonanie wielokrotne | Jeden SQL z wieloma zestawami parametrów |
| `Result` | wynik | Obiekt reprezentujący rezultat zapytania |

---

## Dalsze czytanie

- **Oficjalna dokumentacja (2.0):** *Engine Configuration* — `https://docs.sqlalchemy.org/en/20/core/engines.html`
- **Pula połączeń:** *Connection Pooling* — `https://docs.sqlalchemy.org/en/20/core/pooling.html`
- **Praca z połączeniami i transakcjami:** *Working with Transactions and the DBAPI* — `https://docs.sqlalchemy.org/en/20/tutorial/dbapi_transactions.html`
- **Konstruktor `text()`:** *Using Textual SQL* — `https://docs.sqlalchemy.org/en/20/core/sqlelement.html#sqlalchemy.sql.expression.text`
- **Dialekt SQLite:** `https://docs.sqlalchemy.org/en/20/dialects/sqlite.html`
- **Dialekt PostgreSQL / psycopg:** `https://docs.sqlalchemy.org/en/20/dialects/postgresql.html`
- **Logowanie SQLAlchemy:** `https://docs.sqlalchemy.org/en/20/core/engines.html#configuring-logging`
- **Nowości 2.1 (dla chętnych):** `https://docs.sqlalchemy.org/en/21/changelog/migration_21.html`

---

## Co dalej

Masz działające środowisko i potrafisz połączyć się z bazą, wykonać dowolny SQL i wrócić do transakcji, gdy coś pójdzie nie tak. To solidna podstawa.

W module `03_metadata_ddl.md` przestaniemy pisać `CREATE TABLE` ręcznie w `text()`, a zaczniemy **opisywać schemat w Pythonie** — za pomocą `MetaData`, `Table` i `Column`. Zobaczysz, jak z definicji w Pythonie SQLAlchemy generuje DDL (czyli właśnie `CREATE TABLE`, `CREATE INDEX` i pokrewne), jak działa refleksja nad istniejącą bazą i dlaczego **nazwy ograniczeń** mają znaczenie już na tym etapie.

➡️ Przejdź do: **`03_metadata_ddl.md`**

<!-- koniec modułu 02 -->