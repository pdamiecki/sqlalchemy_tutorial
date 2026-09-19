# Moduł 16 — Migracje schematu bazy danych z Alembic

W tym module nauczysz się prowadzić **historię zmian struktury bazy danych** tak samo rygorystycznie, jak prowadzisz historię swojego kodu w Git. Zobaczysz, dlaczego `Base.metadata.create_all()` — wygodne na etapie zabawy — staje się pułapką, gdy w bazie są już prawdziwe dane klientów, oraz jak narzędzie **Alembic** pozwala dodawać kolumny, zmieniać typy, przenosić dane między tabelami i wycofywać zmiany, nie tracąc ani jednego rekordu. Nauczysz się czytać wygenerowane migracje (i nigdy nie ufać im bezkrytycznie), pisać działający `downgrade`, projektować migracje bezpieczne dla produkcji (wzorzec *expand/contract*) i uruchamiać cały proces w CI/CD. To moduł „architektoniczny”: po nim schemat bazy przestanie być czymś, co „samo się dzieje”, a stanie się artefaktem, który wersjonujesz, przeglądasz i wdrażasz świadomie.

---

**Metadane modułu**

| Pole | Wartość |
|---|---|
| **Poziom** | 🔴 architektoniczny |
| **Czas** | ~200–240 min (plus czas na ćwiczenia) |
| **Wymagania wstępne** | [`07_modele_deklaratywne.md`](07_modele_deklaratywne.md), [`09_relacje.md`](09_relacje.md), [`11_ladowanie_i_n_plus_1.md`](11_ladowanie_i_n_plus_1.md); mile widziany [`03_metadata_ddl.md`](03_metadata_ddl.md) |
| **Czego dotyczy plik** | Narzędzia Alembic, `alembic.ini`, `env.py`, `versions/`, operacje `op.*`, migracje danych, cykl pracy, wzorce bezpiecznego wdrażania na produkcję |
| **Bazy w przykładach** | SQLite (`sqlite:///library.db`) jako baza startowa, PostgreSQL 16 jako baza produkcyjna |

---

## Spis treści

1. [Czym są migracje i dlaczego `create_all()` nie wystarcza](#1-czym-są-migracje-i-dlaczego-create_all-nie-wystarcza)
2. [Instalacja i pierwsze uruchomienie](#2-instalacja-i-pierwsze-uruchomienie)
3. [Anatomia projektu Alembic](#3-anatomia-projektu-alembic)
4. [`env.py` — serce konfiguracji](#4-envpy--serce-konfiguracji)
5. [Autogenerate — co potrafi, a czego nie](#5-autogenerate--co-potrafi-a-czego-nie)
6. [Operacje w migracjach (`op.*`)](#6-operacje-w-migracjach-op)
7. [Migracje danych](#7-migracje-danych)
8. [Cykl pracy z Alembic](#8-cykl-pracy-z-alembic)
9. [Migracje w zespole i w CI/CD](#9-migracje-w-zespole-i-w-cicd)
10. [Ryzykowne zmiany na produkcji](#10-ryzykowne-zmiany-na-produkcji)
11. [Alembic w kodzie aplikacji](#11-alembic-w-kodzie-aplikacji)
12. [Alternatywy i kompromisy](#12-alternatywy-i-kompromisy)
13. [Pełny przykład: biblioteka w trzech migracjach](#13-pełny-przykład-biblioteka-w-trzech-migracjach)
14. [Podsumowanie](#podsumowanie)
15. [Ćwiczenia](#ćwiczenia)
16. [Najczęstsze błędy i jak je czytać](#najczęstsze-błędy-i-jak-je-czytać)
17. [Słowniczek modułu](#słowniczek-modułu)
18. [Dalsze czytanie](#dalsze-czytanie)
19. [Co dalej](#co-dalej)

---

## 1. Czym są migracje i dlaczego `create_all()` nie wystarcza

W module 07 zdefiniowałeś modele, a w module 03 poznałeś `Base.metadata.create_all(engine)`. To jedno polecenie sprawdza, czy w bazie istnieją tabele opisane w twoich modelach, i tworzy te, których brakuje. Przy nauce i w małych projektach jest idealne: piszesz klasę `Book`, uruchamiasz skrypt, tabela `books` pojawia się w bazie. Magia.

Ta magia kończy się w momencie, gdy w bazie zaczynają mieszkać prawdziwe dane.

> 💡 **Analogia** — `create_all()` to ekipa budowlana z jednym, bardzo konkretnym zleceniem: *„Jeśli domu nie ma, zbuduj go od zera według tego projektu”*. Sprawdza fundament, ściany i dach, których nie ma, i dostawia je. Ale gdy mieszkańcy już się wprowadzili i chcą dobudować pokój, przenieść drzwi albo wymienić instalację — ekipa rozkłada ręce. Nie umie remontować. Nie umie „zburzyć i odtworzyć od nowa”, bo mieszkańcy (twoje dane) musieliby się wyprowadzić, a nikt nie wie, czy wrócą. **Migracje to właśnie ekipa remontowa**: wykonywane po kolei, udokumentowane, odwracalne operacje, które przekształcają istniejący dom — z mieszkańcami w środku — w nowy układ.

### 1.1 Cztery scenariusze, w których `create_all()` zawodzi

Zobaczmy to konkretnie na modelu z modułu 09, gdzie mamy tabele `books`, `authors`, `loans` i kilka tysięcy rekordów. Oto cztery klasyczne sytuacje:

**Scenariusz A — nowa kolumna.** Chcesz dodać do `books` kolumnę `available_copies` (liczba dostępnych egzemplarzy). `create_all()` nic nie zrobi: tabela `books` **już istnieje**, więc ekipa budowlana uzna, że jej praca jest skończona. Nowa kolumna nigdy się nie pojawi. A gdybyś na siłę zrobił `drop_all()` + `create_all()` — skasowałeś całą bazę.

**Scenariusz B — zmiana typu.** Kolumna `isbn` jest dziś `String(20)`, a chcesz rozszerzyć ją do `String(32)`. Znowu: `create_all()` jej nie tknie. Ręcznie trzeba wykonać `ALTER TABLE` w każdym środowisku (dev, test, staging, produkcja) i nikt nie zapamięta, że nazajutrz trzeba to samo zrobić na kolejnym serwerze.

**Scenariusz C — podział tabeli.** Masz w `authors` kolumnę `name` („Jan Kowalski”) i chcesz rozdzielić ją na `first_name` oraz `last_name`. To nie jest operacja jednego polecenia SQL: musisz dodać dwie nowe kolumny, **przepisać** do nich dane ze starej, zweryfikować wynik i dopiero usunąć starą. To już jest *migracja danych*, nie tylko *migracja schematu*.

**Scenariusz D — dane, które trzeba wypełnić.** Nowa kolumna `available_copies` musi zaraz po dodaniu dostać wartość dla istniejących wierszy (np. równą `total_copies`). Sam `ALTER TABLE ... ADD COLUMN` ustawi tam `NULL`. Trzeba dorzucić `UPDATE`.

Każdy z tych scenariuszy ma wspólny mianownik: **operacja musi być zapisana jako plik w repozytorium, wykonana w ustalonej kolejności i możliwa do wycofania.** To jest dokładnie definicja migracji.

> 🧠 **Dlaczego tak jest** — baza danych jest **stanem**, a nie kodem. Kodu nie „ma” na serwerze — kod jest plikiem, który porównujesz z Gitem. Bazy nie porównasz z niczym: na każdym środowisku wygląda trochę inaczej, a różnice trzeba jakoś uzgodnić. Migracje sprowadzają ten problem do jednego pytania: *„Które numerki migracji zostały już na tym środowisku wykonane?”*. Odpowiedź trzyma w bazie sama Alembic, w jednej małej tabeli. Reszta to deterministyczne wykonanie brakujących plików.

### 1.2 Schemat vs dane — dwa rodzaje migracji

Warto od początku rozróżniać dwie rzeczy:

- **Migracja schematu (schema migration)** — zmienia *strukturę*: dodaje kolumnę, tworzy indeks, zmienia typ. Mówi o tym „kształt” tabel.
- **Migracja danych (data migration)** — zmienia *zawartość*: przenosi wartości z jednej kolumny do drugiej, wypełnia nowe pola, normalizuje e-maile na małe litery.

W praktyce oba typy żyją w tych samych plikach migracji (Alembic nie wymusza podziału), ale warto je **pisać osobno**, o czym przekonasz się w rozdziale 7.

> 💡 **Analogia** — schemat to szkielet biblioteki (regały, kategorie, numery półek), a dane to książki. Możesz przebudować regał (schemat), nie dotykając książek, i możesz przestawić książki (dane) bez zmiany regału. Ale czasem musisz jedno i drugie naraz — i wtedy porządek podyktowany jest bezpieczeństwem: najpierw nowy regał, potem przenosiny ksiąg, na końcu demontaż starego.

---

## 2. Instalacja i pierwsze uruchomienie

Alembic nie jest częścią biblioteki SQLAlchemy — to osobny pakiet, ale tworzony przez ten sam zespół i zaprojektowany wyłącznie do pracy z SQLAlchemy.

```bash
# terminal
pip install "alembic>=1.19"
# Alembic działa z SQLAlchemy, więc upewnij się, że masz je obok:
pip install "SQLAlchemy>=2.0"
```

Sprawdź, że działa:

```bash
alembic --version
# Alembic 1.19.x
```

> 🆕 **SQLAlchemy 2.1** — od serii 2.1 sterownikiem domyślnym dla URL-a `postgresql://` jest `psycopg` (wersja 3), a nie `psycopg2`. Ma to znaczenie przy konfiguracji `alembic.ini`: jeśli w dokumentacji Alembica zobaczysz historyczne `postgresql+psycopg2://`, w nowym projekcie możesz spokojnie użyć `postgresql+psycopg://`. Na SQLite nic się nie zmienia.

### 2.1 Inicjalizacja projektu

W katalogu głównym projektu wykonaj:

```bash
alembic init alembic
```

Alembic utworzy:

```text
twoj-projekt/
├── alembic.ini          ← konfiguracja (ścieżki, logowanie, URL bazy)
└── alembic/
    ├── env.py           ← skrypt uruchamiany przy każdej komendzie
    ├── script.py.mako   ← szablon nowych plików migracji
    ├── README           ← krótka notatka, można zignorować
    └── versions/        ← tutaj lądują pliki migracji (na razie pusty)
```

Katalog `alembic/` (nazwa dowolna, tu akurat taka sama jak narzędzie) nazywamy **script location** — miejscem skryptów migracji. Istnieje też gotowy szablon dla projektów asynchronicznych:

```bash
alembic init -t async alembic
```

Ten wariant od razu generuje `env.py`, który korzysta z `AsyncEngine` (patrz [`15_asynchronicznosc.md`](15_asynchronicznosc.md)) i używa `await connection.run_sync(...)`. Jeśli twoja aplikacja jest async — **używaj od razu szablonu `async`**, żeby nie przepisywać `env.py` później.

> ⚠️ **Pułapka** — Alembic nie edytuje twoich modeli ani nie importuje ich automatycznie. `alembic init` tworzy tylko szkielet infrastruktury; połączenie z twoimi klasami (`Book`, `Author`) skonfigurujesz ręcznie w `env.py`. Bez tego kroku Alembic nie ma pojęcia, jak wygląda twoja „prawda” o modelach.

### 2.2 „Jak uruchomić” — minimalny zestaw testowy

Zanim przejdziemy dalej, uruchom Alembic na pustej bazie:

```bash
# krok 1: ustaw w alembic.ini adres bazy (na razie na sztywno)
# sqlalchemy.url = sqlite:///library.db

# krok 2: wygeneruj pierwszą (pustą lub pełną) migrację
alembic revision --autogenerate -m "baseline"

# krok 3: zastosuj ją
alembic upgrade head

# krok 4: sprawdź, co jest w bazie
alembic current
```

Na krok 2 `alembic.ini` mówi jeszcze, że nie ma `target_metadata` — dostaniesz błąd. To normalne; naprawimy to w rozdziale 4. Na razie chodziło o to, byś zobaczył strukturę komend.

> 🧪 **Ćwiczenie** — uruchom `alembic init alembic` w katalogu tymczasowym i przejrzyj wszystkie utworzone pliki. Notuj, co robi każdy z nich — wrócimy do tego w rozdziale 3.

---

## 3. Anatomia projektu Alembic

Zanim zaczniemy konfigurować, przyjrzyjmy się każdemu elementowi infrastruktury. Znajomość tych plików zwraca się wielokrotnie, bo 90% problemów z Alembicem sprowadza się do błędów konfiguracji w `env.py` lub `alembic.ini`.

### 3.1 `alembic.ini`

To plik konfiguracyjny w formacie INI. Otwórz go i skup się na kilku kluczowych liniach:

```ini
# alembic.ini (fragmenty, które realnie się zmienia)

[alembic]
# Katalog ze skryptem env.py i podkatalogiem versions/.
script_location = alembic

# Adres bazy. Kolejność wykonania migration jest deterministyczna,
# więc URL tutaj to "punkt wyjścia" — env.py może go nadpisać.
sqlalchemy.url = sqlite:///library.db

# Szablon nazw plików migracji. Domyśnie "%%(rev)s_%%(slug)s".
file_template = %%(year)d%%(month).2d%%(day).2d_%%(hour).2d%%(minute).2d_%%(rev)s_%%(slug)s

# Katalog, z którego importowane są modele — potrzebne, gdy uruchamiasz
# alembic z innego katalogu niż główny projekt.
prepend_sys_path = .

# Wielokrotne linie lub katalogi migracji (zwykle niepotrzebne)
# version_path_separator = os
```

Kilka uwag praktycznych:

- `file_template` z datą i godziną (jak wyżej) daje pliki o nazwach typu `20260919_1430_a1b2c3d4_add_available_copies.py` — przy dziesiątkach migracji czytelność rośnie dramatycznie.
- `prepend_sys_path = .` zapewnia, że `import myapp.models` w `env.py` zadziała, nawet gdy uruchamiasz `alembic` z innego katalogu (np. w CI/CD).
- **Nie zostawiaj prawdziwego hasła w `alembic.ini`.** Plik idzie do repozytorium. URL bazy ustawisz w `env.py` ze zmiennej środowiskowej (rozdział 4.3).

Sekcja `[loggers]` na dole pliku konfiguruje logowanie Alembica. Domyślne ustawienie `level = WARN` jest wygodne, ale podczas nauki warto tymczasowo zmienić je na `INFO`, żeby widzieć każdy krok.

### 3.2 `alembic/env.py` — spis treści

To najważniejszy plik: uruchamiany przy **każdej** komendzie Alembica. Jego zadanie: przygotować kontekst migracji. Pełną analizę robimy w rozdziale 4; teraz tylko mapa:

```python
# alembic/env.py (ogólna struktura, uproszczona)

from alembic import context
from sqlalchemy import engine_from_config, pool

# 1. Konfiguracja obiektu Alembica (odczyt alembic.ini)
config = context.config

# 2. (Tu wczytamy modele i podłączymy target_metadata — rozdział 4)

# 3. Tryb offline: generowanie SQL-a do pliku, bez łączenia z bazą
def run_migrations_offline() -> None:
    ...

# 4. Tryb online: faktyczne połączenie i wykonanie migracji
def run_migrations_online() -> None:
    ...

if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

### 3.3 `alembic/versions/` — historia migracji

W tym katalogu ląduje każdy plik migracji. Nazwa pliku to kombinacja `revision` (unikalny identyfikator, najczęściej krótki hash) i `slug` (krótki opis tekstowy). Plik jest zwykłym skryptem Pythona z dwiema funkcjami:

```python
def upgrade() -> None:
    """Zmiany 'w przód' — to wykona alembic upgrade."""
    ...

def downgrade() -> None:
    """Zmiany 'w tył' — to wykona alembic downgrade."""
    ...
```

Powiązania między wersjami trzyma się w atrybutach modułu:

```python
revision: str = "a1b2c3d4e5f6"
down_revision: str | None = "deadbeef1234"   # poprzednia migracja w łańcuchu
branch_labels = None
depends_on = None
```

To **lista jednokierunkowa**: każda migracja (poza pierwszą) ma jeden `down_revision`, wskazujący bezpośredniego poprzednika. Alembic idzie po tym łańcuchu i ustala bieżące położenie bazy względem historii.

### 3.4 `alembic/script.py.mako`

Szablon (Mako to język szablonów) używany przez `alembic revision`. Gdy wywołujesz `alembic revision`, Alembic renderuje ten plik i zapisuje w `versions/`. Możesz go edytować — na przykład dodać domyślne importy (choć Alembic i tak wstawia potrzebne sam przy autogenerate) albo szablony komentarzy.

> 🧪 **Ćwiczenie** — uruchom `alembic revision -m "test"` (bez autogenerate — to działa bez `target_metadata`), otwórz plik w `versions/` i zaznacz palcem na ekranie, gdzie jest `revision`, gdzie `down_revision`, gdzie `upgrade`/`downgrade`. Usuń plik, żeby nie zaśmiecał historii.

---

## 4. `env.py` — serce konfiguracji

Skoro `env.py` uruchamia się przy każdej komendzie, każdy jego błąd odbija się na całym procesie. Ten rozdział buduje gotowy do produkcji `env.py` krok po kroku. Zakładamy strukturę projektu:

```text
library-app/
├── alembic.ini
├── alembic/
│   ├── env.py
│   └── versions/
├── app/
│   ├── __init__.py
│   ├── models.py       ← tutaj: Base, Book, Author, Member, Loan, ...
│   └── db.py           ← tutaj: silnik, fabryka sesji
└── pyproject.toml
```

### 4.1 `target_metadata` — połączenie z twoimi modelami

Alembic musi wiedzieć, jak wygląda twoja „prawda” o modelach, żeby porównać ją ze stanem bazy. Ta „prawda” to obiekt `MetaData`, siedzący w twojej klasie `Base`:

```python
# alembic/env.py (fragment — góra pliku)

from __future__ import annotations

from logging.config import fileConfig

from alembic import context
from sqlalchemy import engine_from_config, pool

from app.models import Base  # ← twoje modele i ich metadata

config = context.config

# Logowanie Alembica według alembic.ini
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

# To jest "prawda" o modelach, z którą Alembic porówna bazę.
target_metadata = Base.metadata
```

`Base.metadata` to ten sam obiekt `MetaData`, o którym mówiliśmy w module 07: rejestr wszystkich tabel zdefiniowanych przez klasy dziedziczące po `Base`. Wystarczy więc zaimportować `Base` i wskazać jego `metadata`. Bez tego autogenerate nie ma z czym porównywać i albo nic nie wygeneruje, albo wygeneruje „drop everything”.

> 🧠 **Dlaczego tak jest** — Alembic nie „czyta kodu”. On **importuje twój kod Pythona** i sprawdza, jakie obiekty `Table` zarejestrowały się w podanym `MetaData`. Dlatego `env.py` musi mieć dostęp do wszystkich modeli: jeśli `app/models.py` nie importuje jakiegoś pliku z modelem, to ten model po prostu nie trafi do `Base.metadata`, a Alembic uzna, że tabelę należy **usunąć**. To klasyczna przyczyna historii „autogenerate chciał zdropować pół bazy!” — omówimy ją w rozdziale 5.

### 4.2 `naming_convention` — nazwy ograniczeń mają znaczenie

Zanim pójdziemy dalej, jedna rzecz fundamentalna dla migracji. W SQL każdy indeks, klucz obcy i ograniczenie ma **nazwę**. Jeśli nie nadajesz jej jawnie, baza nadaje ją sama, a **każda baza robi to inaczej**: PostgreSQL nazwie indeks `books_isbn_key`, SQLite `sqlite_autoindex_books_1`, MySQL jeszcze inaczej.

Dla Alembica to katastrofa: gdy przyjdzie do usunięcia ograniczenia, musi znać jego nazwę. A nazwy różnią się między środowiskami, więc migracja napisana pod PostgreSQL nie zadziała na SQLite.

Rozwiązaniem jest `naming_convention` ustawione **raz, na poziomie `MetaData`**:

```python
# app/models.py

from sqlalchemy import MetaData
from sqlalchemy.orm import DeclarativeBase

# Ustalamy jednolite wzory nazw dla WSZYSTKICH ograniczeń i indeksów
# w tej bazie. Dzięki temu są one identyczne na każdej bazie i w każdym
# środowisku — co jest warunkiem koniecznym dla powtarzalnych migracji.
NAMING_CONVENTION: dict[str, str] = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}

metadata_obj = MetaData(naming_convention=NAMING_CONVENTION)


class Base(DeclarativeBase):
    metadata = metadata_obj
```

Teraz klucz obcy z `books.author_id` nazywa się zawsze `fk_books_author_id_authors`, niezależnie od bazy. Migracje przestają być zależne od dialektu.

> ⚠️ **Pułapka** — jeśli masz już bazę **bez** `naming_convention`, a teraz je dodasz, autogenerate zobaczy „wszystkie stare ograniczenia do usunięcia i wszystkie nowe do dodania”. To nie jest błąd Alembica, tylko konsekwencja zmiany nazw. Rozwiązanie: albo od początku projektu ustawiasz `naming_convention` (rób to **zawsze**), albo ręcznie piszesz migrację, która przemianowuje istniejące ograniczenia. Więcej o tym w rozdziale 5.

### 4.3 URL bazy z **env** — konfiguracja wielo-środowiskowa

Nie zapisujemy hasła do bazy w `alembic.ini` (bo trafia do Gita). Zamiast tego `env.py` nadpisuje `sqlalchemy.url` wartością ze zmiennej środowiskowej:

```python
# alembic/env.py (fragment)

import os

# Odczytujemy adres bazy ze zmiennej środowiskowej. W aplikacji webowej
# na produkcji ustawisz DATABASE_URL w konfiguracji serwera/wdrożenia.
DATABASE_URL: str | None = os.environ.get("DATABASE_URL")

if DATABASE_URL:
    # set_main_option nadpisuje wartość z alembic.ini.
    # Znak "%" w URL-u trzeba escapować, bo alembic.ini interpretuje "%":
    config.set_main_option("sqlalchemy.url", DATABASE_URL.replace("%", "%%"))
```

Dzięki temu:

```bash
# lokalnie — SQLite
DATABASE_URL="sqlite:///library.db" alembic upgrade head

# na stagingu — PostgreSQL
DATABASE_URL="postgresql+psycopg://app:secret@db-staging:5432/library" \
  alembic upgrade head
```

> 🆕 **SQLAlchemy 2.1** — wspomniany wcześniej domyślny driver `psycopg` dla `postgresql://` oznacza, że w nowych projektach nie musisz pisać `+psycopg2` ani instalować starego drivera. To upraszcza konfigurację `env.py` i `alembic.ini`. Jeśli jednak projekt produkcyjny był tworzony na `psycopg2`, zostaw jawne `postgresql+psycopg2://` — działają oba, świadomość, który jest aktywny, chroni przed niespodziankami.

### 4.4 Tryby: online i offline

Alembic ma dwa tryby pracy:

- **`offline`** („tryb SQL”) — Alembic generuje cały SQL potrzebny do przejścia ze stanu A do B i wypisuje go na wyjście (lub do pliku). **Nie łączy się z bazą.** Przydatne, gdy DBA (administrator bazy) nie daje aplikacji uprawnień do DDL, a chce zobaczyć i sam wykonać SQL.
- **`online`** („tryb połączenia”) — Alembic łączy się z bazą i wykonuje migracje operacja po operacji. To tryb domyślny, używany codziennie.

Tryby przełącza się flagą `alembic upgrade head --sql` (offline) lub `alembic upgrade head` (online).

Oto pełna, dopracowana implementacja obu funkcji:

```python
# alembic/env.py (fragmenty — pełna wersja)

from __future__ import annotations

from logging.config import fileConfig

from alembic import context
from sqlalchemy import engine_from_config, pool

from app.models import Base

config = context.config

if config.config_file_name is not None:
    fileConfig(config.config_file_name)

target_metadata = Base.metadata


def run_migrations_offline() -> None:
    """Generuje SQL bez połączenia z bazą.

    W tym trybie nie ma połączenia — konfigurujemy tylko URL i target_metadata,
    a kontekst bezpiecznie zapisze SQL do pliku/wyjścia.
    """
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
        compare_type=True,
        compare_server_default=True,
    )

    with context.begin_transaction():
        context.run_migrations()


def run_migrations_online() -> None:
    """Łączy się z bazą i wykonuje migracje rzeczywiście."""
    connectable = engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,  # jedno użycie — pula nie jest potrzebna
    )

    with connectable.connect() as connection:
        context.configure(
            connection=connection,
            target_metadata=target_metadata,
            # Kataloguje porównanie różnych typów (VARCHAR(20) != VARCHAR(32)).
            # Od Alembica 1.12 jest to domyślnie True, ale piszemy jawnie
            # dla czytelności i świadomości.
            compare_type=True,
            # Porównuj wartości server_default przy autogenerate.
            compare_server_default=True,
            # KLUCZOWE dla SQLite: ALTER TABLE tam jest ubogie, więc Alembic
            # musi "odtworzyć" tabelę pod spodem — patrz rozdział 6.3.
            render_as_batch=True,
            # Nazwa tabeli, w której Alembic trzyma historię wersji.
            version_table="alembic_version",
        )

        with context.begin_transaction():
            context.run_migrations()


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

Dwie funkcje — dwie filozofie. `run_migrations_offline` nie wie, co jest w bazie, więc generuje SQL „na ślepo”. `run_migrations_online` zna stan bazy (jeszcze przed migracją) i dlatego lepiej radzi sobie z porównaniem typów (`compare_type`) i wartości domyślnych (`compare_server_default`).

> 🧠 **Dlaczego tak jest** — `compare_type=True` działa „naprawdę” tylko w trybie online, bo Alembic musi **odczytać** aktualny typ z bazy i porównać go z deklaracją w modelu. W trybie offline tego nie wie, więc wynik jest przybliżony. Z tego powodu `autogenerate` działa **wyłącznie** w trybie online — bez połączenia nie wie, co zmienić.

### 4.5 `env.py` w stylu async

Jeśli twoja aplikacja używa `AsyncEngine` (moduł 15), `env.py` z szablonu `-t async` różni się w jednym miejscu: zamiast `engine_from_config` tworzy `create_async_engine`, a migracje uruchamia przez `run_sync`:

```python
# alembic/env.py (fragment dla szablonu async)

import asyncio

from alembic import context
from sqlalchemy.ext.asyncio import async_engine_from_config


def do_run_migrations(connection) -> None:
    context.configure(
        connection=connection,
        target_metadata=target_metadata,
        compare_type=True,
        render_as_batch=True,
    )
    with context.begin_transaction():
        context.run_migrations()


async def run_async_migrations() -> None:
    connectable = async_engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
    )
    async with connectable.connect() as connection:
        # run_sync "przepuszcza" synchroniczne migracje przez async-owy most.
        await connection.run_sync(do_run_migrations)
    await connectable.dispose()


def run_migrations_online() -> None:
    asyncio.run(run_async_migrations())
```

Reszta (`target_metadata`, `set_main_option`, tryby offline/online) jest identyczna.

### 4.6 Pozostałe parametry `context.configure`

Kilka opcji, które warto znać, nawet jeśli nie użyjesz ich od razu:

| Parametr | Domyślnie | Co robi |
|---|---|---|
| `compare_type` | `True` (Alembic ≥1.12) | Wykrywa zmiany typów kolumn |
| `compare_server_default` | `False` | Wykrywa zmiany wartości serwerowych `DEFAULT` |
| `render_as_batch` | `False` | Automatycznie opakowuje `ALTER` w `batch_alter_table` (dla SQLite) |
| `include_schemas` | `False` | Uwzględnia schematy (PostgreSQL `public`, inne) w porównaniu |
| `include_object` | `None` | Callback decydujący, co ignorować (np. tabele tymczasowe innej aplikacji) |
| `process_revision_directives` | `None` | Hook pozwalający zmodyfikować wygenerowaną migrację przed zapisem |
| `version_table` | `"alembic_version"` | Nazwa tabeli z historią wersji |
| `version_table_schema` | `None` | Schemat, w którym ta tabela ma się znajdować |

Ostatnia opcja, `include_object`, jest niezwykle przydatna w projektach, gdzie jedna baza jest współdzielona z innymi systemami: pozwala powiedzieć „nie zarządzaj tą tabelą, to nie twoje”.

---

## 5. Autogenerate — co potrafi, a czego nie

`alembic revision --autogenerate -m "opis"` to najczęściej używana komenda Alembica. Porównuje `target_metadata` (twoje modele) ze stanem bazy i tworzy plik z migracją opisującą różnice.

```bash
alembic revision --autogenerate -m "add available_copies to books"
```

Wygenerowany plik wygląda mniej więcej tak:

```python
# alembic/versions/20260919_1430_a1b2c3_add_available_copies_to_books.py
"""add available_copies to books

Revision ID: a1b2c3d4e5f6
Revises: 9f8e7d6c5b4a
Create Date: 2026-09-19 14:30:00
"""
from __future__ import annotations

from alembic import op
import sqlalchemy as sa


revision: str = "a1b2c3d4e5f6"
down_revision: str | None = "9f8e7d6c5b4a"
branch_labels = None
depends_on = None


def upgrade() -> None:
    op.add_column(
        "books",
        sa.Column("available_copies", sa.Integer(), nullable=False,
                  server_default=sa.text("0")),
    )


def downgrade() -> None:
    op.drop_column("books", "available_copies")
```

To naprawdę działa — ale tylko wtedy, gdy **sam** napisałeś model z `available_copies` i autogenerate „dogonił” bazę. Mimo to obowiązuje żelazna zasada:

> ⚠️ **Pułapka** — **ZAWSZE przeglądaj wygenerowany plik migracji.** Autogenerate to *propozycja*, nie wyrocznia. Potrafi „naprawić” coś, co ma sens, ale potrafi też zaproponować coś, co skasuje twoje dane. Wygenerowana migracja to kod, który trafi na produkcję — czytaj go tak, jak każdy inny kod, który zmienia dane.

### 5.1 Co autogenerate wykrywa

Autogenerate **dobrze** radzi sobie z:

- dodaniem i usunięciem tabel (`op.create_table` / `op.drop_table`),
- dodaniem i usunięciem kolumn,
- zmianą typu kolumny (przy `compare_type=True`),
- zmianą nullability (`nullable=False` / `True`),
- dodaniem i usunięciem indeksów,
- dodaniem i usunięciem kluczy obcych, kluczy głównych, ograniczeń unikalności,
- zmianą wartości `server_default` (przy `compare_server_default=True`),
- nazwanymi ograniczeniami CHECK — **nowość w Alembicu 1.19**, gdzie autogenerate potrafi je wykryć pod warunkiem, że mają jawną nazwę (dlatego potrzebna jest `naming_convention` z rozdziału 4.2).

### 5.2 Czego autogenerate NIE wykrywa

To lista „znanych ślepych plam”. Naucz się jej na pamięć, bo to źródło większości cichych błędów.

| Zmiana | Co widzi autogenerate | Co zrobić |
|---|---|---|
| **Zmiana nazwy kolumny** (`title` → `book_title`) | Widzi „drop `title`, add `book_title`” — **utrata danych!** | Napisać migrację ręcznie: `op.alter_column("books", "title", new_column_name="book_title")` |
| **Zmiana nazwy tabeli** | Widzi „drop starej, create nowej” — **utrata danych!** | `op.rename_table("old", "new")` |
| **Zmiana typu kolumny w SQLite** | Widzi zmianę, ale SQLite nie umie `ALTER COLUMN` | `render_as_batch` + świadomość, że to i tak przebudowa tabeli |
| **Dodanie wartości do `sa.Enum`** (PostgreSQL native enum) | Zwykle nic; „enum” to osobny typ w bazie | Ręcznie `op.execute("ALTER TYPE status ADD VALUE 'new'")` (patrz 5.3) |
| **Zmiany w widokach (`VIEW`)** | Nic (Alembic nie porównuje widoków) | `op.create_view`/`op.drop_view` ręcznie (2.1) lub surowe SQL |
| **Triggery, funkcje, procedury** | Nic | Surowy `op.execute(...)` |
| **Zmiany uprawnień (`GRANT`)** | Nic | `op.execute(...)` |
| **Zmiana kolejności kolumn** | Nic (kolejność w bazie nie ma znaczenia) | Zignorować |
| **Zmiana kolacji / zestawu znaków** | Zależy od dialektu, często nic | Ręcznie |
| **Indeksy istniejące tylko w bazie** (poza modelami) | Na ogół widzi jako „do usunięcia” | Dodać do modeli albo `include_object` |
| **`server_default` funkcji** (`now()` vs `CURRENT_TIMESTAMP`) | Często mylnie uznaje za zmianę (zależnie od dialektu) | `compare_server_default=False` lokalnie albo ręcznie ignorować |

> 💡 **Analogia** — autogenerate to asystent, który porównuje dwie fotografię: twoją deklarację (co chcesz mieć) i stan bazy (co jest). Potrafi zobaczyć, że „na drugim zdjęciu brakuje budki z gazetami” i zaproponować jej postawienie. Ale gdy przesunąłeś budkę o metr (zmiana nazwy), asystent nie zauważa, że to **ta sama budka** — widzi „brak budki tu” i „nowa budka tam” i proponuje zburzyć starą. Dlatego zawsze czytaj to, co ci podsuwa.

### 5.3 Enum i migracje — szczególny przypadek

Zacznijmy od przypomnienia: `sa.Enum` na PostgreSQL tworzy natywny typ `ENUM`. Wartości takiego typu nie da się zmienić prostym `ALTER COLUMN`. Dodanie nowej wartości wymaga polecenia `ALTER TYPE ... ADD VALUE`, a to — w zależności od wersji PostgreSQL — ma ograniczenia (do PG 11 włącznie nie mogło być częścią transakcji; od PG 12 można, ale nowa wartość nie może być użyta w tej samej transakcji).

W Alembicu wygląda to tak:

```python
# alembic/versions/20260919_1500_b2c3d4_add_status_value.py

from alembic import op


revision: str = "b2c3d4e5f6a7"
down_revision: str | None = "a1b2c3d4e5f6"
branch_labels = None
depends_on = None


def upgrade() -> None:
    # Wiele operacji na ENUM w PostgreSQL musi być poza transakcją.
    # Alembic pozwala to zrobić przez autocommit_block.
    # Uwaga: to działa TYLKO na PostgreSQL.
    with op.get_context().autocommit_block():
        op.execute("ALTER TYPE loan_status ADD VALUE IF NOT EXISTS 'overdue'")


def downgrade() -> None:
    # PostgreSQL NIE pozwala usunąć wartości z ENUM.
    # Downgrade musi więc albo odtworzyć typ, albo być celowo pusty.
    # Sensowne minimum: świadoma decyzja "nie da się tego cofnąć".
    pass
```

> ⚠️ **Pułapka** — `downgrade()` może być legalnie pusty **tylko wtedy, gdy rozumiesz, dlaczego jest pusty** i udokumentujesz to komentarzem. Nie pisz pustego `downgrade`, bo „nie chciało ci się”. Pisz go, bo operacja jest technicznie nieodwracalna na danym dialekcie. To różnica między inżynierią a bałaganem.

**Alternatywa na całe życie:** jeśli często dodajesz wartości do zbioru (status=”nowy → w realizacji → wysłany → dostarczony…”), rozważ zamiast `ENUM` **tabelę słownikową** i kolumnę `status_id` z kluczem obcym. Wtedy dodanie wartości to zwykłe `INSERT`, a nie `ALTER TYPE`.

> 🧪 **Ćwiczenie** — w projekcie bez natywnego `Enum` (SQLite) dodaj kolumnę tekstową `status` z `CheckConstraint`, a potem spróbuj w migracji dodać nową wartość do tego CHECK. Zobacz, że na SQLite oznacza to przebudowę tabeli (batch).

---

## 6. Operacje w migracjach (`op.*`)

W `upgrade`/`downgrade` nie piszesz surowego SQL — używasz API `op`, które tłumaczy twoje polecenia na dialekt bazy. Pokażemy najważniejsze operacje z „pod maską”, żebyś widział, jaki SQL realnie leci.

### 6.1 Operacje schematu

**Dodanie kolumny:**

```python
def upgrade() -> None:
    op.add_column(
        "books",
        sa.Column("pages", sa.Integer(), nullable=True),
    )
```

```sql
-- 🔬 Pod maską (PostgreSQL / SQLite)
ALTER TABLE books ADD COLUMN pages INTEGER;
```

**Dodanie kolumny `NOT NULL` z wartością domyślną** — klasyka, gdy tabela ma dane:

```python
def upgrade() -> None:
    op.add_column(
        "books",
        sa.Column(
            "available_copies",
            sa.Integer(),
            nullable=False,
            server_default=sa.text("0"),
        ),
    )
```

```sql
-- 🔬 Pod maską (PostgreSQL)
ALTER TABLE books ADD COLUMN available_copies INTEGER NOT NULL DEFAULT 0;
```

> 🧠 **Dlaczego tak jest** — kolumny `NOT NULL` bez wartości domyślnej nie da się dodać do tabeli z danymi: istniejące wiersze nie mają z czego tej wartości wziąć. Dlatego w migracji **musisz** dać `server_default` albo dodać kolumnę jako `nullable=True`, wypełnić ją, a dopiero potem ustawić `NOT NULL` (trzy osobne operacje). Wzór „add nullable → UPDATE → alter to not null” zobaczysz w rozdziale 10.

**Zmiana kolumny:**

```python
def upgrade() -> None:
    op.alter_column(
        "books",
        "title",
        existing_type=sa.String(200),
        type_=sa.String(300),
        existing_nullable=False,
        new_column_name="book_title",   # zmiana nazwy; UWAGA: na PostgreSQL
                                        # jednocześnie zmiana typu i nazwy
                                        # może wymagać rozdzielenia.
    )
```

**Indeks:**

```python
def upgrade() -> None:
    op.create_index("ix_books_isbn", "books", ["isbn"], unique=True)


def downgrade() -> None:
    op.drop_index("ix_books_isbn", table_name="books")
```

**Ograniczenie unikalności:**

```python
def upgrade() -> None:
    op.create_unique_constraint("uq_members_email", "members", ["email"])


def downgrade() -> None:
    op.drop_constraint("uq_members_email", "members", type_="unique")
```

> 🧠 **Dlaczego tak jest** — `op.drop_constraint` wymaga **nazwy** ograniczenia. To dlatego `naming_convention` (rozdział 4.2) jest tak ważne: bez niego nie wiesz, jak nazwy się nazywają — nie możesz ich usunąć.

**Tworzenie tabeli:**

```python
def upgrade() -> None:
    op.create_table(
        "categories",
        sa.Column("id", sa.Integer(), primary_key=True),
        sa.Column("name", sa.String(80), nullable=False),
        sa.Column("slug", sa.String(80), nullable=False),
        sa.UniqueConstraint("slug", name="uq_categories_slug"),
    )
```

### 6.2 Wstawianie danych masowo: `op.bulk_insert`

`op.bulk_insert` to odpowiednik `insert().values([...])` w Core (moduł 05) — pozwala wypełnić tabelę słownikową jednym poleceniem:

```python
from alembic import op
import sqlalchemy as sa

def upgrade() -> None:
    category_table = sa.table(
        "categories",
        sa.column("id", sa.Integer),
        sa.column("name", sa.String),
        sa.column("slug", sa.String),
    )
    op.bulk_insert(
        category_table,
        [
            {"id": 1, "name": "Fikcja", "slug": "fikcja"},
            {"id": 2, "name": "Literatura faktu", "slug": "non-fiction"},
            {"id": 3, "name": "Fantastyka", "slug": "fantasy"},
        ],
    )
```

Zwróć uwagę, że `sa.table(...)` tworzy lekką reprezentację tabeli **tylko na czas migracji** — nie musisz importować modeli ORM do plików migracji. To ważne: migracja powinna być **niezależna od kodu aplikacji**, który się zmienia. Migracja opisuje **stan historyczny**.

> 🧠 **Dlaczego tak jest** — migracja musi działać, gdy uruchomisz ją dajmy na to rok później, na kodzie, który w międzyczasie zmienił modele dziesięć razy. Gdyby migracja importowała `from app.models import Category`, a model `Category` zmienił się od tamtej pory, migracja przestałaby działać na tym samym schemacie. Dlatego **w migracjach nie używamy modeli ORM** — używamy albo `op.*`, albo lekkich `sa.table(...)`, albo surowego SQL przez `op.execute`.

### 6.3 `batch_alter_table` — ratunek dla SQLite

SQLite to wspaniała baza do nauki, ale ma bardzo ubogie `ALTER TABLE`: nie umie zmienić typu kolumny ani usunąć ograniczenia. Nie umie też dodać klucza obcego do istniejącej tabeli. Aby cokolwiek z tym zrobić, trzeba **przebudować całą tabelę**: utworzyć nową, skopiować dane, usunąć starą, zmienić nazwę nowej.

Alembic ma na to mechanizm `batch_alter_table`:

```python
def upgrade() -> None:
    with op.batch_alter_table("books") as batch_op:
        batch_op.alter_column("title", type_=sa.String(300))
        batch_op.create_unique_constraint("uq_books_isbn", ["isbn"])
        batch_op.drop_column("legacy_notes")
```

Dla SQLite Alembic wykona mniej więcej:

```sql
-- 🔬 Pod maską (SQLite, tryb batch)
CREATE TABLE _alembic_tmp_books (
    id INTEGER NOT NULL,
    title VARCHAR(300) NOT NULL,
    isbn VARCHAR(20),
    author_id INTEGER,
    PRIMARY KEY (id),
    CONSTRAINT uq_books_isbn UNIQUE (isbn),
    FOREIGN KEY(author_id) REFERENCES authors (id)
);
INSERT INTO _alembic_tmp_books (id, title, isbn, author_id)
    SELECT id, title, isbn, author_id FROM books;
DROP TABLE books;
ALTER TABLE _alembic_tmp_books RENAME TO books;
```

Dla PostgreSQL ta sama migracja zamieni się w zwykłe `ALTER TABLE` — `batch_alter_table` jest inteligentny i na bazach, które umieją `ALTER`, nie przebudowuje tabeli.

Dzięki `render_as_batch=True` w `env.py` **autogenerate sam** opakowuje operacje w `batch_alter_table`, gdy wykryje SQLite. Bez tego dostaniesz błąd `near "ALTER": syntax error`.

> ⚠️ **Pułapka** — `batch_alter_table` bez `render_as_batch=True` i bez świadomości, że przebudowuje tabelę, może się wydawać niewinną zmianą, a na dużych tabelach (miliony wierszy) trwać bardzo długo i blokować. Zawsze sprawdź, co migracja robi „pod maską” na twoim dialekcie.

### 6.4 `op.execute` — surowy SQL

Gdy potrzebujesz czegoś, czego nie ma w API `op` (np. specyficznej dla PostgreSQL operacji), sięgasz po surowy SQL:

```python
def upgrade() -> None:
    op.execute(
        """
        CREATE INDEX CONCURRENTLY IF NOT EXISTS ix_loans_member_returned
        ON loans (member_id, returned_at);
        """
    )
```

**Uwaga:** `CREATE INDEX CONCURRENTLY` nie może działać wewnątrz transakcji (a Alembic domyślnie każdą migrację opakowuje w transakcję). Trzeba użyć `autocommit_block()` — jak w przykładzie z `ALTER TYPE` w 5.3.

> 🔬 **Pod maską** — `op.execute("...")` przekazuje SQL niemal bez zmian do sterownika. To znaczy, że tracisz przenośność między dialektami. Używaj `op.execute` świadomie i komentuj, dla jakiego dialektu jest dany zapis (np. `# PostgreSQL 16+`).

---

## 7. Migracje danych

Osobno omawiamy migracje, które **modyfikują zawartość**, nie strukturę. To najbardziej ryzykowny gatunek migracji, bo operuje na prawdziwych danych.

### 7.1 `op.get_bind()` — dostęp do połączenia

Wewnątrz migracji możesz dostać obiekt `Connection`, tego samego, który znasz z modułu 02, i wykonywać na nim dowolne zapytania:

```python
from alembic import op
import sqlalchemy as sa

def upgrade() -> None:
    bind = op.get_bind()

    # Krok 1: bezpiecznie odczytujemy wartości, które chcemy przenieść.
    rows = bind.execute(
        sa.text("SELECT id, total_copies FROM books")
    ).all()

    # Krok 2: aktualizujemy nową kolumnę na podstawie odczytu.
    for row in rows:
        bind.execute(
            sa.text(
                "UPDATE books SET available_copies = :copies WHERE id = :id"
            ),
            {"copies": row.total_copies, "id": row.id},
        )
```

To działa, ale przy dużych tabelach jest **wolne i niebezpieczne** (pętla w Pythonie, tysiące rund do bazy). Lepiej użyć jednego `UPDATE`:

```python
def upgrade() -> None:
    op.execute(
        sa.text(
            "UPDATE books SET available_copies = total_copies "
            "WHERE available_copies = 0"
        )
    )
```

> 💡 **Analogia** — migracja danych to przenoszenie mebli po remoncie. Możesz przenosić każdą rzecz osobno (pętla w Pythonie — wolno, ale widzisz każdy ruch) albo napisać spis i zrobić to hurtem (jeden `UPDATE` — szybko, ale powinieneś potem sprawdzić, czy wszystko się zgadza). Na produkcji hurtem.

### 7.2 Dlaczego migracje danych piszemy **osobno** od migracji schematu

Trzy powody:

1. **Ryzyko i czas.** Migracja schematu jest szybka i zwykle nieodwracalnie tania. Migracja danych może trwać minuty i zależeć od zawartości. Chcesz móc ją wycofać **bez** cofania schematu.
2. **Testowalność.** Schemat testujesz raz na wielu środowiskach. Dane testujesz na kopii produkcji (bo tylko tam są realne rozmiary i przypadki brzegowe).
3. **Kolejność.** Schemat można wdrożyć szybko (zmiana jest natychmiast widoczna), a dane migruje się stopniowo. Rozdzielenie pozwala zastosować wzorzec *expand/contract* (rozdział 10.3).

W Alembicu to są po prostu **oddzielne pliki migracji**:

```text
versions/
├── 20260919_1400_add_available_copies.py       ← schemat
├── 20260919_1410_fill_available_copies.py      ← dane
└── 20260919_1420_add_index_on_isbn.py          ← schemat (indeks)
```

### 7.3 Migracje danych zawsze z zapisanym `downgrade`

`downgrade` migracji danych musi być napisany — nawet jeśli będzie „wyglądał” na nieodwracalny:

```python
def downgrade() -> None:
    # Wycofujemy tylko wypełnienie — zerujemy kolumnę.
    # Schemat cofa osobna migracja.
    op.execute("UPDATE books SET available_copies = 0")
```

Albo — gdy dane są kompletnie jednorazowe:

```python
def downgrade() -> None:
    # Celowo pusty: wypełnienie kolumny jest jednokierunkowe.
    # Cofnięcie schematu (osobna migracja) usuwa kolumnę tak czy siak,
    # więc wycofywanie danych nie ma tu sensu.
    pass
```

> ⚠️ **Pułapka** — pusta funkcja `downgrade()` z powodu 1. (kompletna nieodwracalność) i z powodu 2. (lenistwo) wyglądają identycznie w kodzie. Różnica jest w komentarzu i twojej intencji. Zawsze dopisuj jedno zdanie uzasadnienia.

### 7.4 Idempotencja migracji danych

Migrację danych dobrze jest pisać **idempotentnie** — tak, żeby powtórne uruchomienie nie zrobiło szkody. Wzorzec: tylko „dokończ to, co już częściowo się stało”:

```python
def upgrade() -> None:
    # Wypełniamy tylko wiersze, które jeszcze nie były wypełnione.
    # Gdy migracja została przerwana w połowie, powtórne uruchomienie
    # dokończy resztę bez nadpisywania wartości.
    op.execute(
        sa.text(
            "UPDATE books SET available_copies = total_copies "
            "WHERE available_copies IS NULL"
        )
    )
```

Alembic sam trzyma informację o wykonanych migracjach, więc „powtórne uruchomienie tej samej migracji” zdarza się rzadko — ale zdarza się, gdy przerwiesz migrację (Ctrl+C, kill procesu) w połowie i musisz ją uruchomić ręcznie ponownie. Idempotencja to tanio kupione bezpieczeństwo.

---

## 8. Cykl pracy z Alembic

Poznasz teraz komendy, które będziesz wydawać codziennie. Każdą z nich warto zobaczyć „w akcji” na prostym projekcie.

### 8.1 Podstawowe komendy

```bash
# Utwórz nową migrację (wykryj zmiany w stosunku do modeli)
alembic revision --autogenerate -m "add available_copies"

# Utwórz pustą migrację (napiszesz upgrade/downgrade ręcznie)
alembic revision -m "manual data backfill"

# Zastosuj wszystkie brakujące migracje
alembic upgrade head

# Cofnij ostatnią migrację
alembic downgrade -1

# Cofnij do konkretnej rewizji
alembic downgrade a1b2c3d4e5f6

# Zobacz, która migracja jest aktualnie zastosowana
alembic current

# Zobacz pełną historię
alembic history
alembic history --verbose

# Zobacz, ile migracji brakuje do head
alembic heads

# "Podstempluj" bazę bez wykonywania migracji
# (użyj ostrożnie! — patrz 8.3)
alembic stamp head

# Sprawdź, czy autogenerate miałby coś do wygenerowania
# (bez tworzenia pliku) — nowość w Alembicu 1.9+
alembic check
```

`alembic check` to jedno z najbardziej niedocenianych narzędzi. Służy do wykrywania **rozjazdu** między modelami a bazą. Idealne do wpięcia w CI:

```bash
# Jeśli ktoś zmienił model, ale nie wygenerował migracji — CI krzyczy.
alembic check
# No new upgrade operations detected.    ← dobrze
# albo:
# FAILED: New upgrade operations detected: [ ... ]   ← źle
```

### 8.2 Łańcuch rewizji i rozgałęzienia

Każda migracja wskazuje poprzednika w `down_revision`. Gdy historię widzimy jako graf, wygląda mniej więcej tak:

```text
     base (pierwsza migracja)
        │
        ▼
      a1b2c3  add available_copies
        │
        ▼
      deadbe  fill available_copies
        │
        ▼
      f6a7b8  index on isbn   ← head
```

`head` to najnowsza rewizja w łańcuchu. Problem pojawia się, gdy **dwóch programistów pracujących na dwóch gałęziach Git stworzy po jednej migracji równolegle**. Wtedy powstają **dwie głowy**:

```text
      a1b2c3
       ├──▶ deadbe  ← developer A
       └──▶ 0c9f1a  ← developer B
```

Baza wie, że jest na `deadbe`, ale druga gałąź (`0c9f1a`) jest nieznana. Alembic to wykryje:

```bash
alembic heads
# deadbe (head)
# 0c9f1a (head)
```

Trzeba „zespawać” historię w jedną głowę za pomocą migracji scalającej (merge):

```bash
alembic merge -m "merge A and B" deadbe 0c9f1a
```

To tworzy nową migrację, której `down_revision` to **krotka obu** rewizji:

```python
revision: str = "ffff0000"
down_revision = ("deadbe", "0c9f1a")
```

Od teraz historia jest liniowa: obie zmiany żyją **obok siebie**, nie jedna przed drugą, i kolejne migracje idą już od scalonej głowy. Migracja `merge` zwykle nie zawiera żadnych operacji (`upgrade` i `downgrade` są puste) — istnieje wyłącznie po to, żeby „złączyć” dwie linie.

> 🧠 **Dlaczego tak jest** — to nie jest wada narzędzia, tylko konsekwencja równoległej pracy. Alembic nie wie, że zmiana A i zmiana B „mają” jakąś kolejność — dowiaduje się dopiero od ciebie przez `merge`. W małym zespole dobrze jest wprowadzić zasadę: **każdy przed commitem uruchamia `alembic check`**, a po scaleniu gałęzi — `alembic heads`. Dwie głowy przed wprowadzeniem to sygnał do natychmiastowego `merge`.

### 8.3 `alembic stamp` — kiedy jest niebezpieczny

`alembic stamp <rev>` mówi bazie „jesteś na tej wersji”, **nie wykonując żadnej migracji**. To przydatne w konkretnym scenariuszu: gdy masz już bazę produkcyjną z pełnym schematem (utworzonym nie przez Alembic) i chcesz „wprowadzić” Alembic, wskazujesz mu wersję, którą schemat odpowiada. Alembic zapisuje tę wersję w `alembic_version` i odtąd prowadzi historię normalnie.

To również **niebezpieczne**, bo jeśli `stamp` wskaże wersję, której schemat NAPRAWDĘ nie odpowiada bazie, kolejne `upgrade` będą nakładać zmiany na zły stan. Efekt: baza rozjedzie się z historią i nikt już nie wie, co jest prawdą.

> ⚠️ **Pułapka** — `alembic stamp head` na świeżej bazie, w której nie ma tabel, sprawi, że Alembic uzna wszystkie migracje za wykonane. Potem aplikacja „nie widzi” tabel, a `create_all` (jeśli go użyjesz) utworzy je w złej kolejności. `stamp` to lekarstwo na jeden konkretny problem — nie używaj go „na wyczucie”.

### 8.4 „Downgrade zawsze napisany”

Zasada do zapamiętania na całe życie: **każda migracja z poprawnym `downgrade` to migracja, którą można bezpiecznie cofnąć na produkcji**, gdy coś pójdzie źle zaraz po wdrożeniu.

W practice widać to wcześnie, gdy uruchamiasz migrację „na test”: `upgrade head`, potem `downgrade -1`, potem znowu `upgrade head`. Jeśli któraś z tych operacji się wywala, dowód, że twoje `downgrade` **nie działa** — nawet jeśli nie rzucało wyjątku, mogło zostawić bazę w niespójnym stanie.

> 🧪 **Ćwiczenie** — weź dowolną migrację z projektu (swojego lub z tego modułu) i wykonaj cykl `upgrade head` → `downgrade -1` → `upgrade head` na lokalnej bazie z danymi. Sprawdź, że dane są nienaruszone.

---

## 9. Migracje w zespole i w CI/CD

Migracje w pojedynkę są proste. Migracje w zespole — to projektowanie procesu. Oto zasady, które warto przyjąć od pierwszego dnia.

### 9.1 Reguły pracy

1. **Migracja to kod.** Przechodzi review jak każdy kod. Czytaj uważnie, co robi, zwłaszcza `upgrade` z `DROP` albo `ALTER` zmieniającym typ.
2. **Pierwszy programista, który dodał model, pisze migrację.** Nie zostawiamy tego na później.
3. **Nie edytuj już zastosowanej migracji.** Jeśli migracja jest w `main` i została wdrożona, bazy mają wpis w `alembic_version`. Zmiana pliku oznacza, że różne bazy mają „tę samą wersję” o różnych treściach — katastrofa. Zmiany wprowadzaj nową migracją.
4. **Przed mergem uruchom `alembic check`.** CI to zrobi za ciebie. Jeśli wyjdzie „new upgrade operations detected”, brakuje migracji.
5. **Wszystkie migracje muszą przejść pełny cykl w CI:** `upgrade head` → (opcjonalnie test danych) → `downgrade base` → `upgrade head`.
6. **Numer wersji i wersja schematu nie muszą się zgadzać.** Schema wersjonuje się w Alembicu, numer aplikacji w `pyproject.toml`. Wystarczy, że istnieje mapa „ta wersja aplikacji wymaga migracji do tej rewizji”. Zwykle wystarczy: „wdrożenie zawsze robi `alembic upgrade head`”.

### 9.2 Migracje w CI

Minimalna konfiguracja CI to krok, który uruchamia pełen cykl na czystej bazie. Przykład (GitHub Actions):

```yaml
# .github/workflows/ci.yml (fragment)

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    env:
      DATABASE_URL: postgresql+psycopg://postgres:postgres@localhost:5432/test
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -e ".[dev]"
      - name: Sprawdź, czy migracje są aktualne
        run: alembic check
      - name: Pełny cykl migracji
        run: |
          alembic upgrade head
          alembic downgrade base
          alembic upgrade head
      - name: Testy
        run: pytest
```

Trzy rzeczy warte komentarza:

- `alembic check` złapie zapomnianą migrację po zmianie modelu.
- `downgrade base` (czyli cofnij wszystko) sprawdza, że wszystkie `downgrade` ze sobą współpracują. To brutalne, ale skuteczne. Jeśli masz migrację danych, której cofnięcie jest nieodwracalne — użyj `downgrade <rev>` do najstarszej odwracalnej.
- Testy napisane **po** migracjach działają na prawdziwym schemacie, nie na `create_all`.

### 9.3 Migracje na produkcji — `upgrade head` jako krok wdrożenia

Standardowy schemat wdrożenia:

```bash
# w pipeline przed startem nowej wersji aplikacji:
alembic upgrade head

# następnie start aplikacji
```

Zasada: **aplikacja startująca nie tworzy schematu**. Migracja jest osobnym krokiem, uruchamianym raz, przed uruchomieniem nowych instancji. To rozdzielenie ma dwie korzyści: (a) migrację widzisz jako osobny etap wdrożenia, w którym można zareagować na błąd, (b) w środowisku z wieloma replikami aplikacji tylko jeden proces (krok CI) ją wykonuje — nie ma wyścigu.

**Wyjście awaryjne:** jeśli migracja polegnie w połowie, najczęściej nie ma połowy — transakcja się wycofuje. Chyba że migracja robi coś sprytnego, jak `CREATE INDEX CONCURRENTLY` w `autocommit_block` — wtedy może zostawić niedokończony indeks. Dlatego te operacje muszą być idempotentne (`IF NOT EXISTS`, sprawdzenie stanu).

---

## 10. Ryzykowne zmiany na produkcji

To najważniejszy rozdział dla każdego, kto wdraża na produkcję. Mówi o tym, jak zmieniać schemat na bazie z milionami wierszy i **bez wyłączania aplikacji**.

### 10.1 Dodanie kolumny `NOT NULL` do dużej tabeli

Naiwne podejście:

```python
op.add_column(
    "books",
    sa.Column("available_copies", sa.Integer(), nullable=False),
)
```

To na tabeli z 50 mln wierszy **wykona się** (PostgreSQL 11+ potrafi zrobić metadata-only `ADD COLUMN`), ale tylko dlatego, że nie ma `DEFAULT` — wtedy wszystkie istniejące wiersze mają… nie, chwila: `NOT NULL` bez `DEFAULT` **nie da się dodać** do tabeli z danymi, chyba że tabela jest pusta. Alembic zgłosi błąd.

Poprawne, produkcyjne podejście:

```python
def upgrade() -> None:
    # Krok 1: dodaj kolumnę jako nullable z server_default.
    # PostgreSQL 11+ wykona to jako METADATA-ONLY — bez przepisywania tabeli.
    op.add_column(
        "books",
        sa.Column(
            "available_copies",
            sa.Integer(),
            nullable=True,
            server_default=sa.text("0"),
        ),
    )

    # Krok 2 (opcjonalnie): wypełnij istniejące wiersze sensowną wartością.
    op.execute(
        "UPDATE books SET available_copies = total_copies "
        "WHERE available_copies = 0"
    )

    # Krok 3: wymuś NOT NULL. PostgreSQL zweryfikuje wartości przy ALTER —
    # na dużych tabelach to skanowanie (patrz uwaga niżej).
    op.alter_column(
        "books",
        "available_copies",
        existing_type=sa.Integer(),
        nullable=False,
    )
```

> 🧠 **Dlaczego tak jest** — od PostgreSQL 11 `ADD COLUMN ... DEFAULT ...` nie przepisuje tabeli: PostgreSQL zapamiętuje wartość domyślną w metadanych i „udaje”, że wszystkie istniejące wiersze ją mają. Ale **`ALTER COLUMN ... SET NOT NULL`** na dużych tabelach musi zeskanować tabelę, żeby sprawdzić, że żaden wiersz nie jest NULL — i to może trwać minuty oraz blokować. Bezpieczną alternatywą jest dodanie **ograniczenia CHECK `NOT VALID`** i późniejsza walidacja:

```python
def upgrade() -> None:
    # Szybka operacja: dodaj ograniczenie, ale nie waliduj od razu.
    # PostgreSQL zapisze je jako NOT VALID i nie blokuje tabeli.
    op.execute(
        "ALTER TABLE books "
        "ADD CONSTRAINT ck_books_available_copies_not_null "
        "CHECK (available_copies IS NOT NULL) NOT VALID"
    )


def upgrade_validate() -> None:  # uruchamiane w osobnej, późniejszej migracji
    # Walidacja bez długiej blokady zapisów (SHARE UPDATE EXCLUSIVE).
    op.execute(
        "ALTER TABLE books VALIDATE CONSTRAINT ck_books_available_copies_not_null"
    )
```

Sekret tkwi w `NOT VALID`: PostgreSQL nie sprawdza istniejących wierszy w tej chwili, więc operacja trwa milisekundy. Dopiero `VALIDATE CONSTRAINT` skanuje tabelę (bez blokady zapisów) i oznacza ograniczenie jako aktywne.

### 10.2 `CREATE INDEX` — `CONCURRENTLY`

Zwykłe `CREATE INDEX` blokuje zapisy do tabeli na czas budowy indeksu. Na tabeli z milionami wierszy to zablokuje aplikację. PostgreSQL oferuje `CREATE INDEX CONCURRENTLY`, które buduje indeks bez blokowania zapisów (działa wolniej i nie może być w transakcji).

```python
def upgrade() -> None:
    # CREATE INDEX CONCURRENTLY nie może się wykonać w transakcji,
    # więc używamy autocommit_block().
    with op.get_context().autocommit_block():
        op.create_index(
            "ix_loans_member_returned",
            "loans",
            ["member_id", "returned_at"],
            postgresql_concurrently=True,   # opcja specyficzna dla PostgreSQL
        )


def downgrade() -> None:
    with op.get_context().autocommit_block():
        op.drop_index(
            "ix_loans_member_returned",
            table_name="loans",
            postgresql_concurrently=True,
        )
```

`op.create_index(..., postgresql_concurrently=True)` generuje:

```sql
-- 🔬 Pod maską (PostgreSQL)
CREATE INDEX CONCURRENTLY ix_loans_member_returned
    ON loans (member_id, returned_at);
```

To jedna z tych sytuacji, gdy „da się zrobić bez wyłączania aplikacji” wymaga świadomej współpracy z dialektem. Na innych bazach (MySQL 5.7+, MariaDB) też istnieją mechanizmy online; na SQLite — nie ma, bo SQLite i tak trzyma zapisy globalnie.

### 10.3 Wzorzec *expand/contract* — migracja bez przestoju

Najważniejsze narzędzie do zmian, które „wyglądają” na wymagające przerwy. Chcemy np. rozdzielić `authors.name` na `first_name` i `last_name`. Gdybyśmy zrobili to jedną migracją, w trakcie jej działania **stary kod nie wiedziałby o nowych kolumnach, a nowy nie wiedziałby o starej** — wdrożenie starych i nowych instancji aplikacji nakładałoby się w czasie.

Wzorzec *expand/contract* dzieli całą zmianę na cztery (lub pięć) kroków rozłożonych w czasie:

**Faza 1 — Expand (rozszerzenie).** Dodaj nowe kolumny (`first_name`, `last_name`) jako nullable. Stare kolumny **zostają**. Stary kod działa bez zmian. Nowy kod jeszcze nie istnieje.

```python
def upgrade() -> None:
    op.add_column("authors", sa.Column("first_name", sa.String(60), nullable=True))
    op.add_column("authors", sa.Column("last_name", sa.String(60), nullable=True))
```

**Faza 2 — Dual write (podwójny zapis).** Nowa wersja aplikacji zapisuje **do obu miejsc** — starej kolumny `name` i nowych `first_name`/`last_name`. Dzięki temu każdy nowy wiersz ma poprawną wartość w obu formatach. Odczyt wciąż ze starego.

**Faza 3 — Backfill (uzupełnienie historii).** Migracja danych wypełnia nowe kolumny dla **starych** wierszy.

```python
def upgrade() -> None:
    op.execute(
        "UPDATE authors SET "
        "first_name = split_part(name, ' ', 1), "
        "last_name = substring(name from position(' ' in name) + 1) "
        "WHERE first_name IS NULL"
    )
```

**Faza 4 — Switch reads (przełączenie odczytów).** Nowa wersja aplikacji zaczyna czytać z nowych kolumn. Stare kolumny stają się „martwe” (nikt ich nie używa), ale wciąż żyją w bazie.

**Faza 5 — Contract (zwężenie).** Po pewnym czasie, gdy stare instancje aplikacji już nie istnieją, usuwamy stare kolumny.

```python
def upgrade() -> None:
    op.drop_column("authors", "name")
```

> 💡 **Analogia** — remont kuchni w domu, w którym ciągle ktoś mieszka. Najpierw stawiasz nową szafkę obok starej (expand), potem przenosisz do niej część rzeczy (dual write), potem przenosisz resztę (backfill), potem gotujesz tylko z nowej szafki (switch reads), a na końcu wynosisz starą (contract). Nikt nie został ani na chwilę bez kuchni, a zmiana jest trwała.

To wzorzec znany ze świata bazodanowego od dawna, ale w praktyce trzeba go **rozpisać na osobne migracje** — jedna faza = jedna migracja = jeden etap wdrożenia aplikacji. Migracje danych (faza 3) piszesz **idempotentnie**, bo możesz je dokończyć na kolejnym uruchomieniu.

### 10.4 Szacowanie czasu i plan awaryjny

Przed wdrożeniem ryzykownej migracji:

- **Zrób kopię bazy** albo upewnij się, że masz działający backup z punktu w czasie.
- **Przetestuj na kopii danych produkcyjnych** (anonimizowanej — patrz [`18_testowanie.md`](18_testowanie.md)). Migracja na 100 wierszach testowych nic nie mówi o tym, co zrobi na 50 mln.
- **Oszacuj czas.** Dla `ALTER TABLE` z walidacją: `EXPLAIN ANALYZE` na fragmencie i ekstrapolacja. Dla `CREATE INDEX CONCURRENTLY`: na 1 mln wierszy zwykle sekundy, na 100 mln — dziesiątki minut.
- **Przygotuj plan rollback.** Cofnięcie migracji (`downgrade`) — jeśli jest szybkie. Albo *forward fix* — kolejna migracja naprawiająca. W scenariuszach typu „drop kolumny” rollback może być niemożliwy (dane znikają), więc rób *expand/contract*, a usunięcie kolumny zostaw na sam koniec, gdy nie ma już z czego wracać.
- **Zabezpiecz okno serwisowe** tylko wtedy, gdy naprawdę nie da się inaczej. W większości wypadków wzorce z 10.1–10.3 pozwalają go uniknąć.

> ⚠️ **Pułapka** — „mamy backup” to nie to samo co „backup jest zweryfikowany”. Backup, którego nikt nigdy nie odtworzył, nie jest backupem, tylko plikiem. Raz na kwartał rób próbne odtworzenie.

---

## 11. Alembic w kodzie aplikacji

Domyślnie Alembic uruchamia się z terminala (CLI). Ale można go też wywoływać z kodu Pythona — to przydatne w skryptach wdrożeniowych, w CLI narzędziowych czy w testach.

### 11.1 Programowe uruchomienie migracji

```python
# scripts/migrate.py

from __future__ import annotations

import os
from pathlib import Path

from alembic import command
from alembic.config import Config


def run_migrations() -> None:
    """Uruchamia migracje do head, używając konfiguracji Alembica."""
    project_root = Path(__file__).resolve().parent.parent
    alembic_cfg = Config(str(project_root / "alembic.ini"))

    # Upewnij się, że katalog alembic/ jest widoczny.
    alembic_cfg.set_main_option("script_location", str(project_root / "alembic"))

    # URL bazy — najlepiej ze zmiennej środowiskowej (bez hasła w kodzie).
    db_url = os.environ["DATABASE_URL"]
    alembic_cfg.set_main_option("sqlalchemy.url", db_url)

    command.upgrade(alembic_cfg, "head")


if __name__ == "__main__":
    run_migrations()
```

`command.upgrade`, `command.downgrade`, `command.current`, `command.stamp` — wszystkie komendy Alembica mają swoje odpowiedniki w module `alembic.command`.

### 11.2 Kiedy uruchamiać migracje przy **starcie** aplikacji

To jest pytanie, które dzieli społeczność. Rozsądna odpowiedź brzmi:

**NIE uruchamiaj migracji automatycznie przy każdym starcie aplikacji w środowisku produkcyjnym.** Powody:

1. W środowisku z N replikami aplikacji wszystkie N procesów mogłoby jednocześnie uruchomić migrację — jedna z nich wygra, pozostałe czekają lub błądzą. To nie jest deterministyczne.
2. Jeśli migracja się nie powiedzie (np. lock timeout), aplikacja nie wstanie wcale, a na stagingu widać tylko „app crashed”.
3. Utrudnia to rozdzielenie ról: osoba wdrażyjąca aplikację nawet nie wie, że migracja się odbywa.
4. Wdrożenie „rollback” wraca do poprzedniej wersji kodu, ale nie cofa migracji — więc baza zostaje nowsza niż kod. To czasem pożądane (expand/contract), a czasem katastrofa.

**Zalecenie:** migracje uruchamiaj jako **osobny krok pipeline'u**, przed startem aplikacji. W środowisku deweloperskim (jeden deweloper, jedno uruchomienie) automatyczne migracje przy starcie są wygodne — o ile jesteś świadomy, że na produkcji tak nie robisz.

```python
# NIE RÓB TEGO w main.py produkcyjnej aplikacji:
# Base.metadata.create_all(engine)

# NIE RÓB TEGO AUTOMATYCZNIE przy starcie poza środowiskiem developmentu:
# run_migrations()
```

> 🧠 **Dlaczego tak jest** — `create_all` i migracje reprezentują dwa różne modele pracy: „baza to tylko kopia mojego kodu” (dev) vs „baza to trwały stan, który przechodzi przez wersje” (produkcja). Mieszanie ich prowadzi do stanu, w którym nikt nie wie, czy baza jest aktualna i czyje zmiany są „oficjalne”. Ustal jedną regułę i trzymaj się jej.

### 11.3 Wiele baz danych

Gdy aplikacja rozmawia z kilkoma bazami (np. baza główna + hurtownia), masz dwie strategie.

**Strategia A — osobne konfiguracje Alembica.** Dwa katalogi `alembic/` i dwa `alembic.ini`. Prosto, ale duplikacja.

**Strategia B — jeden `env.py` z parametrem `-x`.** Alembic pozwala przekazać dodatkowe parametry z CLI:

```bash
alembic -x db=main upgrade head
alembic -x db=analytics upgrade head
```

W `env.py` odczytujesz je tak:

```python
# alembic/env.py (fragment)

x_args = context.get_x_argument(as_dictionary=True)
db_alias = x_args.get("db", "main")

DATABASES = {
    "main": os.environ["DATABASE_URL_MAIN"],
    "analytics": os.environ["DATABASE_URL_ANALYTICS"],
}

target_metadata_map = {
    "main": Base.metadata,
    "analytics": AnalyticsBase.metadata,
}

config.set_main_option("sqlalchemy.url", DATABASES[db_alias])
target_metadata = target_metadata_map[db_alias]
```

Wtedy dwie aplikacje mają jeden projekt Alembica, ale różne `versions/` (albo przynajmniej różne łańcuchy `down_revision`).

> 🆕 **SQLAlchemy 2.1** — `greenlet` nie instaluje się już automatycznie z SQLAlchemy, więc jeśli używasz szablonu async w Alembicu, pamiętaj o `pip install "sqlalchemy[asyncio]"`. Bez tego `run_sync` w `env.py` wysypie się z błędem importu — częsta pułapka przy migracji projektu z 2.0 na 2.1.

---

## 12. Alternatywy i kompromisy

Alembic nie jest jedynym narzędziem do migracji schematu. Krótkie porównanie, żebyś wiedział, **dlaczego** w ekosystemie SQLAlchemy wygrywa.

| Narzędzie | Jak działa | Mocne strony | Słabości |
|---|---|---|---|
| **Alembic** | Framework Python, generuje kod migracji z modeli | Ścisła integracja z SQLAlchemy, autogenerate, wsparcie dialektów | Wymaga Pythona (nie dla innych języków) |
| **SQL ręcznie** | Katalog plików `.sql` w repozytorium, wykonywane narzędziami | Zero abstrakcji, wszystkie dialekty | Brak wersjonowania stanu, brak wykrywania rozjazdów, ręczne śledzenie |
| **Flyway** | Narzędzie JVM (choć samodzielne), pliki `.sql` z numeracją | Dojrzałe, szerokie wsparcie baz, świetne w środowisku JVM | Nie generuje z modeli, brak ścisłej więzi z SQLAlchemy |
| **Liquibase** | Narzędzie JVM, pliki XML/YAML/SQL | Wsparcie wielu baz, „changesety” | Rozbudowany, ciężki, luźny związek z Pythonem |
| **Django migrations** | Wbudowane w Django, ścisły związek z modelami Django | Bezobsługowość, świetne dla Django | Nie dla SQLAlchemy (inny ORM) |
| **`create_all`** | Wbudowane w SQLAlchemy | Zero konfiguracji | Nie umie zmieniać istniejącego schematu (rozdz. 1) |

**Dlaczego w projektach SQLAlchemy standardem jest Alembic:** jedno narzędzie, jedna filozofia, jedna biblioteka do zrozumienia. `env.py` odwołuje się do tych samych modeli, których używa aplikacja. Osoba wprowadzająca się do projektu widzi jeden plik konfiguracyjny i dziesiątki plików `versions/`, a nie „SQL-e wrzucone lata temu przez kogoś, kto już odszedł”.

> ⚠️ **Pułapka** — nie próbuj „hardkorowo” migrować SQL-em ręcznym w projekcie SQLAlchemy. To działa w prostych projektach, ale traci sens, gdy modeli jest 40, mają zmienne relacje i zależy ci na wykrywaniu rozjazdów. Uczciwa cena Alembica to nauka 60 minut i jedna konfiguracja `env.py`. Zwraca się przy drugiej zmianie schematu.

---

## 13. Pełny przykład: biblioteka w trzech migracjach

Poniższy przykład zakłada bazę biblioteki z modułu 09. Warto go wygenerować samemu, krok po kroku — użyjemy trzech migracji pokazujących trzy różne typy zmian:

1. **Migracja A — schemat:** dodanie kolumny `available_copies` do `books` z `server_default`.
2. **Migracja B — dane:** wypełnienie nowej kolumny na podstawie `total_copies`.
3. **Migracja C — schemat (indeks):** indeks na `books(isbn)`.

### 13.1 Model z modułu 09 (przypomnienie)

```python
# examples/16_library_models.py

from __future__ import annotations

from sqlalchemy import CheckConstraint, ForeignKey, MetaData, String, Text
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


NAMING_CONVENTION: dict[str, str] = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}


class Base(DeclarativeBase):
    metadata = MetaData(naming_convention=NAMING_CONVENTION)


class Author(Base):
    __tablename__ = "authors"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(120))
    bio: Mapped[str | None] = mapped_column(Text())

    books: Mapped[list["Book"]] = relationship(back_populates="author")


class Book(Base):
    __tablename__ = "books"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    isbn: Mapped[str | None] = mapped_column(String(20))
    total_copies: Mapped[int] = mapped_column(default=1)
    author_id: Mapped[int] = mapped_column(ForeignKey("authors.id"))

    author: Mapped["Author"] = relationship(back_populates="books")

    __table_args__ = (
        CheckConstraint("total_copies >= 0", name="total_copies_non_negative"),
    )
```

> 🧠 **Dlaczego tak jest** — `CheckConstraint` ma jawną nazwę (`total_copies_non_negative`). Bez tego na PostgreSQL dostałaby nazwę zależną od kolejności kolumn, a na SQLite w ogóle nie dałby się odnaleźć w Alembicu. Jawna nazwa + `naming_convention` (rozdział 4.2) dają powtarzalność.

### 13.2 Konfiguracja projektu

```text
library-app/
├── alembic.ini
├── alembic/
│   ├── env.py
│   └── versions/
└── examples/
    └── 16_library_models.py
```

`env.py` ustawiamy według rozdziału 4.4, z `render_as_batch=True` (bo testujemy na SQLite), `target_metadata = Base.metadata` oraz URL-em z `DATABASE_URL`.

### 13.3 Migracja A — dodanie kolumny

Zakładamy, że w bazie istnieje już schemat z 5 tysiącami książek. Najpierw zmieniamy model:

```python
# examples/16_library_models.py (zmiana)

class Book(Base):
    __tablename__ = "books"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    isbn: Mapped[str | None] = mapped_column(String(20))
    total_copies: Mapped[int] = mapped_column(default=1)
    available_copies: Mapped[int] = mapped_column(
        default=0,
        server_default="0",
    )
    author_id: Mapped[int] = mapped_column(ForeignKey("authors.id"))
    ...
```

Generujemy migrację:

```bash
DATABASE_URL="sqlite:///library.db" \
  alembic revision --autogenerate -m "add available_copies to books"
```

Otwieramy plik i **czytamy go** (zasada z rozdziału 5!). Na SQLite wygląda tak:

```python
# alembic/versions/20260919_1430_a1b2c3_add_available_copies_to_books.py

from __future__ import annotations

from alembic import op
import sqlalchemy as sa

revision: str = "a1b2c3d4e5f6"
down_revision: str | None = "9f8e7d6c5b4a"
branch_labels = None
depends_on = None


def upgrade() -> None:
    with op.batch_alter_table("books") as batch_op:
        batch_op.add_column(
            sa.Column(
                "available_copies",
                sa.Integer(),
                server_default="0",
                nullable=False,
            )
        )


def downgrade() -> None:
    with op.batch_alter_table("books") as batch_op:
        batch_op.drop_column("available_copies")
```

Uwagi:

- Na SQLite `ALTER TABLE ADD COLUMN` z `NOT NULL` i `DEFAULT` jest **dozwolone** (SQLite specyficzne), więc `batch_alter_table` tu niekoniecznie musi się pojawić. Alembic i tak je wstawił, bo `render_as_batch=True`. To akceptowalne — i wprost pokazuje, jak „działa pod maską”.
- Na PostgreSQL ten kod wygenerowałby zwykłe `op.add_column(...)` bez `batch`, co wykonuje się szybko (metadata-only, jak mówiliśmy w 10.1).

Stosujemy migrację:

```bash
DATABASE_URL="sqlite:///library.db" alembic upgrade head
```

### 13.4 Migracja B — migracja danych

Nowa kolumna ma `sever_default` = 0, ale chcemy, żeby dla istniejących wierszy `available_copies` było równe `total_copies`. Piszemy migrację danych ręcznie:

```bash
DATABASE_URL="sqlite:///library.db" \
  alembic revision -m "backfill available_copies from total_copies"
```

I uzupełniamy:

```python
# alembic/versions/20260919_1445_b2c3d4_backfill_available_copies.py

from __future__ import annotations

from alembic import op
import sqlalchemy as sa

revision: str = "b2c3d4e5f6a7"
down_revision: str | None = "a1b2c3d4e5f6"
branch_labels = None
depends_on = None


def upgrade() -> None:
    # Idempotentnie: aktualizujemy tylko wiersze, które nie były jeszcze
    # wypełnione. Gdy migracja została przerwana w połowie, powtórzenie
    # dokończy resztę, nie nadpisując już poprawnych wartości.
    op.execute(
        sa.text(
            "UPDATE books SET available_copies = total_copies "
            "WHERE available_copies = 0"
        )
    )


def downgrade() -> None:
    # Celowo zerujemy. Nie da się odtworzyć "stanu przed migracją",
    # więc wycofanie sprowadza kolumnę do wartości domyślnej.
    op.execute(sa.text("UPDATE books SET available_copies = 0"))
```

Zwróć uwagę na `op.execute(sa.text(...))` — to bezpieczniejsze niż surowy string, bo pokazuje typ intencji (ustawiamy parametryzowane polecenie) i chroni przed literówkami w cytowaniu.

### 13.5 Migracja C — indeks

Dodajemy indeks na `books.isbn` (bo szukamy po ISBN w wyszukiwarce). W modelu:

```python
# examples/16_library_models.py (zmiana)

class Book(Base):
    __tablename__ = "books"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    isbn: Mapped[str | None] = mapped_column(String(20), index=True)
    ...
```

Autogenerate:

```bash
DATABASE_URL="sqlite:///library.db" \
  alembic revision --autogenerate -m "add index on books.isbn"
```

Wynik (SQLite):

```python
# alembic/versions/20260919_1500_c3d4e5_add_index_on_books_isbn.py

from __future__ import annotations

from alembic import op

revision: str = "c3d4e5f6a7b8"
down_revision: str | None = "b2c3d4e5f6a7"
branch_labels = None
depends_on = None


def upgrade() -> None:
    op.create_index("ix_books_isbn", "books", ["isbn"], unique=False)


def downgrade() -> None:
    op.drop_index("ix_books_isbn", table_name="books")
```

Uwaga: nazwa indeksu `ix_books_isbn`. Gdybyśmy nie mieli `naming_convention`, na SQLite indeks dostałby nazwę `ix_books_isbn` z domyślnej konwencji SQLAlchemy — a na PostgreSQL miałby tej samej nazwy domyślnej, ale nie zgadzałby się z niczym, co generujesz w innych projektach. Spójność jest wartością samą w sobie.

### 13.6 Podgląd historii i przejście w obie strony

Wyświetlamy historię:

```bash
DATABASE_URL="sqlite:///library.db" alembic history
# b2c3d4e5f6a7 -> c3d4e5f6a7b8 (head), add index on books.isbn
# a1b2c3d4e5f6 -> b2c3d4e5f6a7, backfill available_copies from total_copies
# <base> -> a1b2c3d4e5f6, add available_copies to books
```

Sprawdzamy stan:

```bash
DATABASE_URL="sqlite:///library.db" alembic current
# c3d4e5f6a7b8 (head)
```

Cofamy dwie ostatnie migracje:

```bash
DATABASE_URL="sqlite:///library.db" alembic downgrade -2
# INFO  [alembic.runtime.migration] Running downgrade c3d4e5f6a7b8 -> b2c3d4e5f6a7
# INFO  [alembic.runtime.migration] Running downgrade b2c3d4e5f6a7 -> a1b2c3d4e5f6
```

I wracamy do przodu:

```bash
DATABASE_URL="sqlite:///library.db" alembic upgrade head
# INFO  Running upgrade a1b2c3d4e5f6 -> b2c3d4e5f6a7
# INFO  Running upgrade b2c3d4e5f6a7 -> c3d4e5f6a7b8
```

Sprawdzenie „pod maską” — generujemy SQL, który zostałby wykonany:

```bash
DATABASE_URL="sqlite:///library.db" alembic upgrade head --sql
```

Ten tryb (offline) pokaże ci komplet SQL do przejścia ze stanu bieżącego do head — świetne narzędzie do review, gdy robisz change na produkcji bez dostępu do bazy.

### 13.7 „Jak uruchomić” cały przykład

```bash
# 1. Środowisko
python -m venv .venv
source .venv/bin/activate            # Windows: .venv\Scripts\activate
pip install "SQLAlchemy>=2.0" "alembic>=1.19"

# 2. Inicjalizacja Alembica w pustym katalogu projektu
alembic init alembic

# 3. Zastąp alembic/env.py zawartością z rozdziału 4.4 (z render_as_batch=True)

# 4. Ustaw bazę (SQLite) — w alembic.ini lub w DATABASE_URL
export DATABASE_URL="sqlite:///library.db"

# 5. Utwórz bazę z modeli (jednorazowo — dla szkicu)
python -c "from examples.library_models import Base; from sqlalchemy import create_engine; \
Base.metadata.create_all(create_engine('sqlite:///library.db'))"

# 6. Wykonaj trzy migracje — patrz wyżej (13.3, 13.4, 13.5)
alembic revision --autogenerate -m "add available_copies"
alembic upgrade head
alembic revision -m "backfill available_copies"
# (uzupełnij plik treścią z 13.4)
alembic upgrade head
alembic revision --autogenerate -m "add index on books.isbn"
alembic upgrade head

# 7. Podróż w obie strony
alembic history --verbose
alembic current
alembic downgrade -2
alembic upgrade head
```

> 🧪 **Ćwiczenie** — dodaj kolumnę `acquisition_date` typu `DateTime` z `server_default` `CURRENT_TIMESTAMP`, wypełnij ją dla istniejących wierszy wartością „początek tego roku” w osobnej migracji danych, a następnie dodaj indeks częściowy `WHERE acquisition_date IS NOT NULL` i przełącz bazę na head. Napisz wszystkie trzy migracje od zera — z pełnym `upgrade` i `downgrade`.

---

## Podsumowanie

1. **`create_all()` to nie migracje.** Tworzy schemat od zera i nie potrafi zmieniać istniejącego. Migracje to sekwencja udokumentowanych zmian, zastosowanych po kolei na każdym środowisku.
2. **Alembic to standard w ekosystemie SQLAlchemy.** `alembic init alembic` tworzy `alembic.ini`, `env.py`, `versions/` i `script.py.mako`. Cały proces opiera się na `env.py`.
3. **`target_metadata = Base.metadata`** to most między twoimi modelami a Alembikiem. Bez tego ani autogenerate, ani sprawdzanie rozjazdów nie działają.
4. **`naming_convention` w `MetaData`** to warunek powtarzalnych migracji. Ustaw go raz, na starcie projektu, i nigdy nie zmieniaj.
5. **Autogenerate to propozycja, nie wyrocznia.** Zmiany nazw kolumn i tabel widzi jako drop+create (utrata danych!). Zawsze czytaj wygenerowany plik.
6. **Na SQLite `render_as_batch=True`.** Bez tego `ALTER TABLE` na wiele operacji nie zadziała.
7. **`op.*` to API migracji.** `add_column`, `alter_column`, `create_index`, `create_unique_constraint`, `bulk_insert`, `execute`. Do surowych operacji — `op.execute`.
8. **Migracje danych pisz osobno** od schematu, idempotentnie i zawsze z zapisanym `downgrade` (choćby pustym z uzasadnieniem).
9. **Cykl pracy:** `revision --autogenerate` → review → `upgrade head` → test `downgrade` → `upgrade head` ponownie. Przed mergem `alembic check`. Dwie głowy → `alembic merge`.
10. **Na produkcji migracje uruchamiaj jako osobny krok wdrożenia**, nie przy starcie aplikacji. Ryzykowne zmiany rób wg wzorca *expand/contract*.

---

## Ćwiczenia

### Ćwiczenie 1 (łatwe) — Migracja dodająca indeks

W projekcie z biblioteką dodaj do tabeli `loans` indeks na parę kolumn `(member_id, returned_at)` — używany przy raporcie „aktywne wypożyczenia członka”. Napisz migrację ręcznie (bez autogenerate) z `upgrade` i `downgrade`. Przetestuj na SQLite i na PostgreSQL (jeśli masz).

### Ćwiczenie 2 (średnie) — Rozdzielenie `full_name`

W tabeli `authors` masz kolumnę `name` (String), która zawiera imię i nazwisko w jednym polu (np. „Jan Kowalski”). Napisz migrację, która:

1. Dodaje kolumny `first_name` i `last_name` (nullable).
2. Wypełnia je na podstawie wartości `name` (proste rozbicie po pierwszej spacji).
3. Ustawia je jako `NOT NULL`.
4. Usuwa kolumnę `name`.

Napisz też pełny `downgrade`, który przywraca `name` (sklejone `first_name || ' ' || last_name`). Przetestuj pełny cykl `upgrade head` → `downgrade -1` → `upgrade head`.

### Ćwiczenie 3 (trudne) — Migracja *expand/contract* bez przestoju

Zaproponuj trzyosobowy zespół pracujący nad aplikacją, w której tabela `loans` ma kolumnę `status` typu `String(20)` przechowującą status w formie tekstowej („active”, „returned”, „overdue”, …). Chcemy zastąpić ją kolumną `status_id` odwołującą się do nowej tabeli słownikowej `loan_statuses`.

Rozpisz **pełny plan migracji** (co najmniej 4 osobne kroki) w stylu *expand/contract*: co zawiera każda migracja, kiedy jest wdrażana, jaki kod aplikacji jest wtedy aktywny, w którym kroku migrujemy dane, a w którym usuwamy starą kolumnę. Narysuj prosty diagram stanu.

---

### Rozwiązania

#### Rozwiązanie 1

```python
# alembic/versions/20260919_1600_d4e5f6_index_loans_member_returned.py

from __future__ import annotations

from alembic import op

revision: str = "d4e5f6a7b8c9"
down_revision: str | None = "c3d4e5f6a7b8"
branch_labels = None
depends_on = None


def upgrade() -> None:
    # Na PostgreSQL moglibyśmy użyć postgresql_concurrently=True
    # w autocommit_block — patrz rozdział 10.2. Tu robimy wersję prostą,
    # kompatybilną ze SQLite i PostgreSQL.
    op.create_index(
        "ix_loans_member_returned",
        "loans",
        ["member_id", "returned_at"],
        unique=False,
    )


def downgrade() -> None:
    op.drop_index("ix_loans_member_returned", table_name="loans")
```

Komentarz: indeks złożony `(member_id, returned_at)` wspiera zapytania typu „aktywne wypożyczenia członka X”. Kolejność kolumn ma znaczenie — gdy filtrujesz po `member_id` i pakujesz po `returned_at`, taka kolejność jest optymalna. Odwrotna pomogłaby zapytaniom „wszystkie wypożyczenia po dacie”, które prawdopodobnie nie są potrzebne.

#### Rozwiązanie 2

Dwie migracje.

```python
# alembic/versions/20260919_1700_e5f6a7_add_first_last_name.py

from __future__ import annotations

from alembic import op
import sqlalchemy as sa

revision: str = "e5f6a7b8c9d0"
down_revision: str | None = "d4e5f6a7b8c9"
branch_labels = None
depends_on = None


def upgrade() -> None:
    # Krok 1: dodaj kolumny jako nullable.
    with op.batch_alter_table("authors") as batch_op:
        batch_op.add_column(sa.Column("first_name", sa.String(60), nullable=True))
        batch_op.add_column(sa.Column("last_name", sa.String(60), nullable=True))

    # Krok 2: wypełnij na podstawie `name`. Rozbijamy po pierwszej spacji.
    # Na PostgreSQL/Mysql działa to zachowawczo; na SQLite analogiczne funkcje
    # mają inne nazwy — dlatego robimy to w Pythonie przez bind.
    bind = op.get_bind()
    rows = bind.execute(sa.text("SELECT id, name FROM authors"))
    for row in rows:
        parts = row.name.strip().split(" ", 1)
        first = parts[0]
        last = parts[1] if len(parts) > 1 else ""
        bind.execute(
            sa.text(
                "UPDATE authors SET first_name = :f, last_name = :l "
                "WHERE id = :id"
            ),
            {"f": first, "l": last, "id": row.id},
        )

    # Krok 3: wymuś NOT NULL na first_name (last_name zostawiamy nullable,
    # bo autorzy jednoimienni istnieją).
    with op.batch_alter_table("authors") as batch_op:
        batch_op.alter_column(
            "first_name",
            existing_type=sa.String(60),
            nullable=False,
        )

    # Krok 4: usuń starą kolumnę.
    with op.batch_alter_table("authors") as batch_op:
        batch_op.drop_column("name")


def downgrade() -> None:
    # Krok A: przywróć kolumnę `name`.
    with op.batch_alter_table("authors") as batch_op:
        batch_op.add_column(sa.Column("name", sa.String(120), nullable=True))

    # Krok B: wypełnij ją zfirst_name + last_name.
    bind = op.get_bind()
    rows = bind.execute(
        sa.text("SELECT id, first_name, last_name FROM authors")
    )
    for row in rows:
        full = (row.first_name + " " + (row.last_name or "")).strip()
        bind.execute(
            sa.text("UPDATE authors SET name = :n WHERE id = :id"),
            {"n": full, "id": row.id},
        )

    # Krok C: wymuś NOT NULL na name.
    with op.batch_alter_table("authors") as batch_op:
        batch_op.alter_column(
            "name",
            existing_type=sa.String(120),
            nullable=False,
        )

    # Krok D: usuń nowe kolumny.
    with op.batch_alter_table("authors") as batch_op:
        batch_op.drop_column("first_name")
        batch_op.drop_column("last_name")
```

Uwaga do rozwiązania: w prawdziwym projekcie lepiej byłoby rozbić to na **dwie migracje** — jedną schematową (kroki 1, 3, 4) i jedną danych (krok 2), zgodnie z rozdziałem 7.2. Tu trzymamy razem, żebyś zobaczył cały wzór. Kod w Pythonie (pętla zamiast `split_part`) jest przenośny między dialektami, ale **wolny** — na produkcji z milionem wierszy lepiej użyć funkcji SQL-owych. To właśnie kompromis, o którym mówiliśmy w 7.1.

#### Rozwiązanie 3

**Cel:** zamienić kolumnę `loans.status` (String) na relację `loans.status_id` → `loan_statuses.id`, bez przestoju, na systemie z dwoma wersjami aplikacji przez jakiś czas.

**Plan (5 kroków):**

**Krok 1 — Migracja 1 (schemat, expand).**
Migracja tworzy tabelę `loan_statuses(id, code)` i wypełnia ją wartościami (`active`, `returned`, `overdue`). Do tabeli `loans` dodaje **nullable** kolumnę `status_id` (FK do `loan_statuses`). Stara kolumna `status` zostaje.
Kod aplikacji: **stary**. Nikt nic nie zapisuje do `status_id`. Wdrożenie bezpieczne, schemat „obszerniejszy” niż potrzeba.

**Krok 2 — Migracja 2 (dane, backfill).**
Migracja danych: dla każdego wiersza `loans` ustawia `status_id` na podstawie zmapowanego `status`. Idempotentnie (WHERE status_id IS NULL).

**Krok 3 — Wdrożenie aplikacji v2 (dual write).**
Aplikacja v2 pisze do **obu** kolumn: `status` i `status_id`, oraz czyta z `status_id`. Stara aplikacja v1 (jeśli jeszcze gdzieś działa) nadal czyta z `status`. Dzięki temu oba systemy widzą spójne dane.

**Krok 4 — Migracja 3 (schemat, switch).**
Po tym, jak cały ruch idzie przez v2, nowa migracja ustawia `status_id` jako `NOT NULL`.
Kod aplikacji: **v2**. Stara kolumna `status` żyje, ale nikt jej już nie czyta.

**Krok 5 — Migracja 4 (schemat, contract).**
Po okresie karencji (dni, tygodnie) migracja usuwa kolumnę `status`.
Kod aplikacji: **v2**.

**Diagram stanu (uproszczony):**

```text
Schemat:    [loans bez status_id] → [+ status_id nullable] → [status_id NOT NULL] → [- status]
Aplikacja:      v1                  v1 lub v2                v2               v2
                                (dual write)

Gdyby krok 4 i 5 zrobiono naraz:
     Schemat:  [status_id NOT NULL, bez status] w jednej migracji
     Ryzyko:   aplikacja v1 (jeszcze działająca) traci kolumnę, na której pracuje
               → błąd produkcyjny, niedostępność usługi
```

Kluczowa refleksja: *expand/contract* nie jest o migracjach, tylko o **chronologii wdrożeń**. Migracje wymuszają schemat, w którym kolejne wersje aplikacji mogą współistnieć przez jakiś okres. Gdybyś zmienił schemat „raz a dobrze” jedną migracją, zmusiłbyś do jednoczesnego (i ryzykownego) przełączenia wszystkich instancji aplikacji — co na produkcji niemal zawsze oznacza przestój.

---

## Najczęstsze błędy i jak je czytać

| Komunikat błędu (lub objaw) | Przyczyna | Naprawa |
|---|---|---|
| `Target database is not up to date.` | Baza ma rewizję wcześniejszą niż `head`, a `revision --autogenerate` uruchomiono bez `upgrade` | Zrób `alembic upgrade head` przed kolejnym `revision`, albo (gdy rozjazd jest świadomy) użyj `alembic merge` |
| `Multiple head revisions are present for given argument 'head'` | Dwie równoległe migracje (typowe przy pracy równoległej na gałęziach) | `alembic merge -m "merge heads" head1 head2` |
| `Can't locate revision identified by 'abc123'` | Migracja `abc123` została usunięta lub nie istnieje w `versions/`, a jest w `alembic_version` w bazie | Sprawdź `alembic history`, znajdź wpis w bazie i `alembic stamp` na właściwą rewizję |
| `sqlalchemy.exc.OperationalError: near "ALTER": syntax error` (SQLite) | Brak `render_as_batch=True` w `env.py` | Dodaj `render_as_batch=True` do `context.configure` |
| `sqlalchemy.exc.CompileError: (in table 'x', column 'y'): Can't generate DDL for NullType()` | Migracja ma kolumnę bez zadeklarowanego typu, w której baza sama nie może się domyślić typu | Uzupełnij typ (`sa.Integer()`, `sa.String(50)` itd.) |
| Autogenerate generuje `drop_column` + `add_column` (utrata danych!) | Zmieniłeś nazwę kolumny w modelu; Alembic nie domyśla się, że to ta sama kolumna | Napisz migrację ręcznie z `op.alter_column(..., new_column_name=...)` |
| `ValueError: Constraint must have a name` (przy `drop_constraint`) | Brak `naming_convention` albo ręczna nazwa niezgodna z konfiguracją | Ustaw `naming_convention` w `MetaData`; nazwę znajdź w bazie (`\d <table>` w psql albo `inspect(engine).get_foreign_keys(...)`) |
| `alembic.runtime.migration: ERROR [alembic.autogenerate] ...` `cannot alter type of a column used by a view or rule` (PostgreSQL) | Zmiana typu kolumny używanej w widoku | Usuń widok, zmień kolumnę, odtwórz widok (w tej samej migracji) |
| `pending`/`running` w `alembic_version` na produkcji | Migracja została przerwana w połowie i nie zdążyła zapisać `head` | Sprawdź, co zostało zrobione (`alembic current`), wycofaj ręcznie niedokończoną część albo dokończ w kolejnej migracji |
| `Target database is not up to date` mimo że kodu nie zmieniałeś | `alembic check` wykrywa rozjazd, bo w modelach dodano np. `server_default`, którego w bazie nie ma | Wygeneruj migrację `alembic revision --autogenerate` i sprawdź, co widzi |
| `relation "books" does not exist` przy `create_all`/migracji | Używasz `alembic stamp` na bazie, w której nie ma jeszcze tabel | Użyj `alembic upgrade head` (bez `stamp`) albo — świadomie — utwórz schemat i `stamp` na odpowiadającą rewizję |
| `RuntimeError: 'url' not set for connection` | Brak konfiguracji `sqlalchemy.url` w `alembic.ini` ani w `env.py` | Ustaw URL albo (co lepsze) przez `DATABASE_URL` + `config.set_main_option` |
| `sqlalchemy.exc.ArgumentError: Could not parse SQLAlchemy URL` | Zły format URL — najczęściej brak części „baza” lub zły dialekt | Sprawdź format: `dialekt+sterownik://użytkownik:hasło@host:p/baza` |
| Migracje widzą tabele, których nie ma w modelach jako „do usunięcia” | Modele nie zostały zaimportowane do `env.py` (np. `import app.models` nie ciągnie pod-modułu) | Upewnij się, że importy w `models.py` (lub `models/__init__.py`) sięgają do *wszystkich* plików z klasami |
| `greenlet` nie znaleziony (async) — nowe w 2.1 | SQLAlchemy 2.1 nie ciągnie `greenlet` domyślnie | `pip install "sqlalchemy[asyncio]"` |
| `migration 0001.py` działa, ale `alembic downgrade base` wysypuje się | Jeden z `downgrade` w łańcuchu jest niepoprawny — najczęściej usuwa kolumnę, która nie istnieje, albo zamienia kolumny w złej kolejności | Przejdź krok po kroku `alembic downgrade -1` i zobacz, na której migracji się sypie |
| `SELECT ... GROUP BY` błąd przy migracji danych używającej agregacji | Migracja pętli po rekordach i robi `UPDATE` z `GROUP BY` w różnych miejscach | Użyj jednego `UPDATE ... FROM` (PostgreSQL) lub dwóch kroków: `SELECT` do `tmp`, potem `UPDATE` |
| `alembic.ini` ma hasło, które wyciekło do Git | Konfiguracja zostawiona na sztywno w pliku | Ustaw URL ze zmiennej środowiskowej (`DATABASE_URL`); w `alembic.ini` — placeholder albo `sqlalchemy.url` bez hasła |

---

## Słowniczek modułu

| Termin (EN) | Termin (PL) | Wyjaśnienie |
|---|---|---|
| **Migration** | Migracja | Sekwencja zmian schematu bazy danych, rozumiana jako nieodłączna część kodu, wersjonowana razem z nim |
| **Schema migration** | Migracja schematu | Zmiana *struktury* bazy (dodanie kolumny, zmiana typu, utworzenie indeksu) |
| **Data migration** | Migracja danych | Zmiana *zawartości* bazy (przeniesienie wartości między kolumnami, wypełnienie nowej kolumny) |
| **Revision** | Rewizja | Pojedynczy plik migracji w Alembicu, identyfikowany przez krótki hash i łączony z poprzednikiem przez `down_revision` |
| **`head`** | Głowa | Najnowsza rewizja w łańcuchu, do której zmierza `alembic upgrade head` |
| **`down_revision`** | — | Wskaźnik do poprzedniej rewizji. Tworzy liniowy łańcuch historii (lub graf z `merge`) |
| **Branch / head split** | Rozgałęzienie / dwie głowy | Sytuacja, w której historia ma więcej niż jedną głowę (efekt pracy równoległej). Rozwiązanie: `alembic merge` |
| **`env.py`** | — | Skrypt uruchamiany przez Alembic przy każdej komendzie; konfiguruje połączenie, `target_metadata` i tryb pracy |
| **`alembic.ini`** | — | Plik konfiguracyjny Alembica (ścieżki, URL, logowanie) |
| **`versions/`** | Katalog wersji | Folder, w którym trzymane są pliki migracji |
| **`script.py.mako`** | — | Szablon nowych plików migracji, używany przez `alembic revision` |
| **`target_metadata`** | — | Obiekt `MetaData` z twoimi modelami; źródło prawdy, z którym Alembic porównuje bazę |
| **Autogenerate** | Autogenerowanie | Tryb `alembic revision --autogenerate`, który porównuje `target_metadata` ze stanem bazy i tworzy plik migracji z wykrytymi zmianami |
| **`naming_convention`** | Konwencja nazewnictwa | Reguły nadawania spójnych nazw ograniczeniom i indeksom — warunek powtarzalnych migracji |
| **`render_as_batch`** | Tryb batch | Parametr `env.py` powodujący automatyczne opakowanie operacji `ALTER` w `batch_alter_table` (niezbędne dla SQLite) |
| **`batch_alter_table`** | Przebudowa tabeli (batch) | Mechanizm Alembica, który na bazach bez rozbudowanego `ALTER TABLE` odtwarza tabelę: kopiuje dane, tworzy nową, zamienia nazwę |
| **`op.*`** | Operacje migracyjne | API używane w `upgrade`/`downgrade` do tworzenia, zmiany i usuwania obiektów schematu. Tłumaczy się na SQL dialektu |
| **`op.get_bind()`** | — | Zwraca obiekt `Connection`, na którym można wykonywać zapytania w trakcie migracji |
| **`op.bulk_insert`** | — | Masowe wstawianie wierszy (np. wypełnienie tabeli słownikowej) |
| **`op.execute`** | — | Wykonanie surowego SQL-a w migracji |
| **`compare_type`** | Porównanie typów | Parametr `context.configure` — czy autogenerate ma wykrywać zmiany typów kolumn |
| **`compare_server_default`** | Porównanie wartości domyślnych | Parametr `context.configure` — czy autogenerate ma wykrywać zmiany wartości `server_default` |
| **`server_default`** | Wartość domyślna po stronie serwera | Wartość domyślna zdefiniowana w bazie (`DEFAULT 0`), stosowana także do istniejących wierszy po `ALTER TABLE` |
| **`NULL / NOT NULL`** | — | Ograniczenie dopuszczalności wartości pustych dla kolumny |
| **`NOT VALID` / `VALIDATE CONSTRAINT`** | — | Wzorzec PostgreSQL: szybko dodaj ograniczenie bez walidacji, później zwaliduj bez długiej blokady |
| **`CONCURRENTLY`** | — | Opcja PostgreSQL (`CREATE INDEX CONCURRENTLY`) — buduje indeks bez blokowania zapisów. Nie może działać wewnątrz transakcji (wymaga `autocommit_block`) |
| **Expand/contract** | Rozszerzenie / zwężenie | Wzorzec bezpiecznych zmian schematu na produkcji: najpierw „poszerz” schemat (dodaj nowe obok starych), zmigruj dane, przełącz aplikację, na końcu usuń stare |
| **Backfill** | Uzupełnienie historii | Wypełnianie wartości nowej kolumny dla istniejących wierszy |
| **Dual write** | Podwójny zapis | Tymczasowe zapisywanie danej do dwóch miejsc (starej i nowej kolumny), by umożliwić płynne przejście między wersjami aplikacji |
| **`autocommit_block()`** | Blok bez transakcji | Kontekst Alembica pozwalający wykonać operacje wymagające braku transakcji (`CREATE INDEX CONCURRENTLY`, `ALTER TYPE ADD VALUE`) |
| **`alembic check`** | Kontrola rozjazdu | Komenda sprawdzająca, czy modele i baza są zgodne; idealna do CI |
| **`alembic stamp`** | Oznaczenie wersji | Zapisuje w bazie informację „jesteś na tej rewizji” bez wykonywania migracji. Używać ostrożnie |
| **Zero-downtime migration** | Migracja bez przestoju | Zmiana schematu, której aplikacja nie odczuwa jako przerwy w działaniu |
| **CI/CD** | Ciągła integracja i dostarczanie | Automatyzacja, w której migracje są uruchamiane jako deterministyczny etap wdrożenia |

---

## Dalsze czytanie

- Dokumentacja Alembica — wprowadzenie i samouczek: <https://alembic.sqlalchemy.org/en/latest/tutorial.html>
- Konfiguracja `env.py` (tryby online/offline, `target_metadata`, `context.configure`): <https://alembic.sqlalchemy.org/en/latest/tutorial.html#configuration>
- Autogenerate — co wykrywa, a czego nie (kluczowa lista ślepych plam): <https://alembic.sqlalchemy.org/en/latest/autogenerate.html>
- Autogenerate — porównanie typów (`compare_type`): <https://alembic.sqlalchemy.org/en/latest/autogenerate.html#comparing-types>
- Operacje `op.*` — referencja API migracji: <https://alembic.sqlalchemy.org/en/latest/ops.html>
- `batch_alter_table` (tryb batch dla SQLite): <https://alembic.sqlalchemy.org/en/latest/batch.html>
- Migracje danych i `op.get_bind()`: <https://alembic.sqlalchemy.org/en/latest/ops.html#alembic.operations.Operations.get_bind>
- `autocommit_block` i operacje nietransakcyjne: <https://alembic.sqlalchemy.org/en/latest/api/runtime.html#alembic.runtime.migration.MigrationContext.autocommit_block>
- `naming_convention` w SQLAlchemy — konfiguracja nazw ograniczeń: <https://docs.sqlalchemy.org/en/20/core/metadata.html#sqlalchemy.MetaData.params.naming_convention>
- SQLAlchemy — tworzenie schematu i relacja do migracji: <https://docs.sqlalchemy.org/en/20/core/metadata.html#creating-and-dropping-database-tables>
- PostgreSQL — `ALTER TABLE ... ADD COLUMN` jako metadata-only: <https://www.postgresql.org/docs/current/sql-altertable.html>
- PostgreSQL — `CREATE INDEX ... CONCURRENTLY` i `NOT VALID`: <https://www.postgresql.org/docs/current/sql-createindex.html>
- Nowości SQLAlchemy 2.1 (m.in. `greenlet` poza ekstra `async`): <https://docs.sqlalchemy.org/en/21/changelog/migration_21.html>

---

## Co dalej

Masz teraz narzędzie, które pozwala zmieniać schemat bazy z prawdziwą precyzją — i wiesz, jak to robić, kiedy pod ręką są dane milionów użytkowników. W module [`17_wydajnosc.md`](17_wydajnosc.md) zajmiemy się drugą stroną tej samej monety: **mierzeniem i optymalizacją** kodu, który już działa. Migracje dbają, żeby schemat był poprawny; pomiar dbają, żeby schemat i zapytania były *szybkie*. Zaczynamy od metodologii — nie zgadywać, tylko liczyć.

<!-- koniec modułu 16 -->