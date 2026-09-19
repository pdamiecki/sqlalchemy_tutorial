# Moduł 08 — Sesja: cykl życia obiektów i transakcji

W tym module poznasz `Session` — serce warstwy ORM. Dowiesz się, dlaczego samo `engine` nie wystarcza do pracy na obiektach, czym jest „notes zmian" i „koszyk zakupowy" w wydaniu SQLAlchemy, jak obiekty przechodzą ze stanu *transient* do *detached*, dlaczego dwa zapytania potrafią zwrócić **ten sam** obiekt Pythona i — co najważniejsze — **w którym dokładnie momencie** Twój kod zamienia się w SQL lecący do bazy. Po tym module przestaniesz się dziwić, że „dodałem obiekt, a w bazie nic nie ma".

| | |
|---|---|
| **Poziom** | 🟡 średni |
| **Czas** | ~180 minut (plus ćwiczenia) |
| **Wymagania wstępne** | [`02_srodowisko_i_engine.md`](02_srodowisko_i_engine.md) (Engine, połączenia, transakcje w Core), [`05_dml.md`](05_dml.md) (INSERT/UPDATE/DELETE, `commit`, `rollback`), [`07_modele_deklaratywne.md`](07_modele_deklaratywne.md) (klasy `Base`, `Mapped`, `mapped_column`) |
| **Czego dotyczy plik** | Obiekt `Session`, fabryka `sessionmaker`, tryby pracy sesji, stany obiektu, identity map, Unit of Work, `flush` vs `commit`, `expire_on_commit`, odczyt i usuwanie obiektów, sesja w kontekście aplikacji |
| **Czego NIE ma w tym pliku** | Relacji między tabelami (`relationship`) — to [`09_relacje.md`](09_relacje.md); strategii ładowania i problemu N+1 — to [`11_ladowanie_i_n_plus_1.md`](11_ladowanie_i_n_plus_1.md); trybu `async` — to [`15_asynchronicznosc.md`](15_asynchronicznosc.md) |
| **Baza** | SQLite (`sqlite://` w pamięci lub plik) — zero konfiguracji, uruchamiasz wszystko lokalnie |

## Spis treści

1. [Problem: dlaczego samo `engine` nie wystarcza](#1-problem-dlaczego-samo-engine-nie-wystarcza)
2. [Czym jest `Session` — model mentalny](#2-czym-jest-session--model-mentalny)
3. [Tworzenie sesji: `Session` i `sessionmaker`](#3-tworzenie-sesji-session-i-sessionmaker)
4. [Trzy tryby pracy sesji](#4-trzy-tryby-pracy-sesji)
5. [Stany obiektu — pełny cykl życia](#5-stany-obiektu--pełny-cykl-życia)
6. [Identity map — rejestr mieszkańców](#6-identity-map--rejestr-mieszkańców)
7. [Unit of Work — kolejka zmian](#7-unit-of-work--kolejka-zmian)
8. [`flush()` kontra `commit()`](#8-flush-kontra-commit)
9. [`expire_on_commit` — najważniejszy przełącznik tego modułu](#9-expire_on_commit--najważniejszy-przełącznik-tego-modułu)
10. [Czytanie obiektów z bazy](#10-czytanie-obiektów-z-bazy)
11. [Obiekty z zewnątrz: `merge()`, `add_all()`, `delete()`](#11-obiekty-z-zewnątrz-merge-add_all-delete)
12. [Sesja w kontekście aplikacji](#12-sesja-w-kontekście-aplikacji)
13. [Pełny przykład: symulator dziennika zmian](#13-pełny-przykład-symulator-dziennika-zmian)
14. [Podsumowanie](#podsumowanie)
15. [Ćwiczenia](#ćwiczenia)
16. [Najczęstsze błędy i jak je czytać](#najczęstsze-błędy-i-jak-je-czytać)
17. [Słowniczek modułu](#słowniczek-modułu)
18. [Dalsze czytanie](#dalsze-czytanie)
19. [Co dalej](#co-dalej)

---

## 1. Problem: dlaczego samo `engine` nie wystarcza

W module 02 nauczyłeś się wykonywać SQL przez `engine.connect()` i `conn.execute(text(...))`. W module 07 zdefiniowałeś klasy Pythona odpowiadające tabelom. Spróbujmy teraz połączyć jedno z drugim **bez** `Session` i zobaczmy, co się psuje.

Załóżmy, że chcemy: (a) dopisać książkę, (b) dopisać drugą, (c) sprawdzić, czy pierwsza już jest w bazie, (d) zmienić tytuł tej pierwszej i (e) wszystko to cofnąć, jeśli krok (d) się nie uda.

```python
# examples/08_without_session.py
from sqlalchemy import create_engine, text

engine = create_engine("sqlite:///library.db", echo=True)

with engine.begin() as conn:  # jedna transakcja na cały blok
    conn.execute(
        text("INSERT INTO books (title, isbn) VALUES (:t, :i)"),
        {"t": "Solaris", "i": "978-83-08-06026-2"},
    )
    conn.execute(
        text("INSERT INTO books (title, isbn) VALUES (:t, :i)"),
        {"t": "Eden", "i": "978-83-08-06027-9"},
    )

    # Chcę teraz pracować z tymi książkami jak z obiektami Pythona...
    row = conn.execute(
        text("SELECT id, title FROM books WHERE isbn = :i"),
        {"i": "978-83-08-06026-2"},
    ).one()
    print(row.id, row.title)  # Row, nie Book
```

Widzisz problemy? Jest ich pięć i wszystkie są poważne:

1. **Nie ma obiektów.** `row` to `Row` — krotka z etykietami. Nie ma metod, nie ma walidacji, nie ma typów. Żeby dostać `Book`, musisz ręcznie napisać `Book(id=row.id, title=row.title, ...)`.
2. **Nie ma pamięci zmian.** Jeśli zmienisz `book.title = "..."`, SQLAlchemy nie ma pojęcia, że coś się zmieniło — to zwykły atrybut Pythona na zwykłym obiekcie.
3. **Nie ma automatycznego UPDATE.** Musisz sam pamiętać, które obiekty zmieniłeś, i sam napisać `UPDATE ... WHERE id = ...`.
4. **Nie ma tożsamości.** Dwa zapytania o tę samą książkę dadzą dwa różne obiekty Pythona. Zmienisz jeden, drugi będzie miał stare dane — i nie zauważysz tego, dopóki nie wyślesz obu do bazy.
5. **Nie ma kolejności.** Przy kluczach obcych musisz ręcznie pilnować, żeby INSERT rodzica poszedł przed INSERT-em dziecka.

> 💡 **Analogia — budowa domu.** `engine` to firma budowlana z bazą sprzętu i ekipami. Możesz zadzwonić i powiedzieć: „przywieźcie cegły" (`conn.execute(...)`) — i to zadziała. Ale gdy chcesz wybudować piętro, potrzebujesz **kierownika budowy z notesem**: zapisuje, co ma być zrobione, pilnuje kolejności (najpierw fundamenty, potem ściany), zbiera poprawki i dopiero gdy notes jest kompletny — wysyła ekipy. Tym notesem jest `Session`.

`Session` to jeden obiekt, który rozwiązuje wszystkie pięć problemów naraz. To nie jest „nakładka na połączenie" — to **menedżer stanu** między Twoimi obiektami Pythona a wierszami w bazie.

> 🧠 **Dlaczego tak jest — trzy zadania w jednym.** `Session` pełni trzy role, które w innych bibliotekach bywają rozdzielone:
> 1. **Rejestr tożsamości** (*identity map*) — „każdy wiersz ma dokładnie jeden obiekt Pythona w tej sesji".
> 2. **Jednostka pracy** (*unit of work*) — „pamiętam, co dodałeś, co zmieniłeś i co usunąłeś, i wyślę to w optymalnej kolejności".
> 3. **Zakres transakcji** (*transaction scope*) — „wszystko, co zrobisz, dzieje się w jednej transakcji, którą można zatwierdzić albo cofnąć".
>
> Zrozumienie tych trzech ról to 80% zrozumienia ORM. Reszta modułu to ich konsekwencje.

---

## 2. Czym jest `Session` — model mentalny

### 2.1 Sesja to nie połączenie

Najczęstsze nieporozumienie na starcie: „sesja = połączenie z bazą". **Nie.** Połączenie to zasób sieciowy, którego jest mało (pula ma np. 5 sztuk) i który trzeba szybko oddawać. Sesja to lekki obiekt Pythona — możesz ich tworzyć setki.

Sesja **pożycza** połączenie z puli dopiero wtedy, gdy naprawdę potrzebuje wysłać SQL, i **oddaje** je, gdy transakcja się kończy. Widać to w logu: samo `Session(engine)` nie generuje ani jednej linii SQL.

```python
# examples/08_session_is_not_connection.py
from sqlalchemy import create_engine
from sqlalchemy.orm import Session

engine = create_engine("sqlite://", echo=True)

session = Session(engine)          # ZERO SQL w logu
print(engine.pool.status())        # np. Pool size: 5  Connections in pool: 0 ...
session.close()                    # ZERO SQL w logu
```

> 💡 **Analogia — wypożyczalnia samochodów.** Sesja to kierowca, połączenie to samochód. Kierowca może stać w kolejce po auto (pula), ale sam fakt „bycia kierowcą" nie zajmuje samochodu. Auto dostaje dopiero, gdy rusza w trasę.

### 2.2 Druga analogia: notes naczelnika zmiany

Wyobraź sobie naczelnika zmiany na budowie z notesem:

| W notesie | Odpowiednik w sesji |
|---|---|
| „zamów 200 cegieł" | `session.add(book)` |
| „zmień kolor ściany w pokoju 3" | `book.title = "..."` (użycie atrybutu!) |
| „rozbiórka ścianki w kuchni" | `session.delete(book)` |
| „wypisz listę zadań ekipom" | `session.flush()` |
| „koniec dnia, wszystko zatwierdzone" | `session.commit()` |
| „wszystko anulujemy" | `session.rollback()` |

Kluczowe: **napisanie czegoś w notesie nie zmienia budynku.** Zmienia go dopiero moment, w którym ekipy dostaną polecenia. Ta różnica między „zapisane w notesie" a „wysłane do wykonania" to cała różnica między `add()` a `flush()`.

### 2.3 Trzecia analogia: koszyk zakupowy

`session.add(book)` to wrzucenie produktu do koszyka. Produkt **jeszcze nie jest Twój** — leży w koszyku i możesz go wyjąć. `session.flush()` to moment, w którym kasjer zaczyna skanować (baza wie o towarze, dostał numer). `session.commit()` to **kasa i paragon** — transakcja zamknięta, towar jest Twój, nie ma odwrotu. `session.close()` to wyjście ze sklepu: koszyk zniknął, ale to, co kupiłeś, zostaje w domu.

> ⚠️ **Pułapka — najważniejsza w całym module.** `session.add(book)` **nie wysyła żadnego SQL**. Ani `add()`, ani przypisanie atrybutu, ani `session.delete()` nie dotykają bazy. SQL leci dopiero przy `flush()` (jawnym lub automatycznym) lub `commit()`. 90% zgłoszeń „dodałem obiekt i go nie ma" to brak `commit()` (albo brak `add()` — oba błędy opisujemy w sekcji [Najczęstsze błędy](#najczęstsze-błędy-i-jak-je-czytać)).

### 2.4 Co sesja pamięta o każdym obiekcie

Dla każdego obiektu, którym zarządza, sesja trzyma wewnętrzny rekord stanu (`InstanceState`). Możesz go obejrzeć funkcją `inspect()`:

```python
# examples/08_inspect_state.py
from sqlalchemy import create_engine, inspect
from sqlalchemy.orm import Session

from examples.models import Book  # Base + Book z modułu 07

engine = create_engine("sqlite:///library.db")

book = Book(title="Solaris", isbn="978-83-08-06026-2")
state = inspect(book)

print(state.transient)    # True  — nikt o nim nie wie
print(state.pending)      # False
print(state.persistent)   # False
print(state.detached)     # False
print(state.identity)     # None  — brak klucza głównego
print(state.key)          # None  — brak tożsamości w sesji
```

`inspect(book)` zwraca `InstanceState` — „kartotekę" obiektu. To ona wie, czy obiekt jest w sesji, czy ma już wiersz w bazie i czy został zmodyfikowany. Wszystko, co dalej robimy, to manipulowanie tym rekordem.

> 🧪 **Ćwiczenie.** Utwórz obiekt `Book`, wywołaj `inspect(book)` i wypisz wszystkie pola `InstanceState` (`vars(state)`), które nie zaczynają się od podkreślenia. Spróbuj odgadnąć, do czego służy `state.modified` i `state.expired`, zanim przeczytasz dalszą część modułu.

---

## 3. Tworzenie sesji: `Session` i `sessionmaker`

### 3.1 Najprostsza droga

```python
# examples/08_first_session.py
from sqlalchemy import create_engine, select
from sqlalchemy.orm import Session

from examples.models import Book

engine = create_engine("sqlite:///library.db")

with Session(engine) as session:                    # 1. sesja związana z engine
    book = session.get(Book, 1)                     # 2. odczyt po kluczu głównym
    print(book.title if book else "brak książki o id=1")
```

Pierwszy argument pozycyjny `Session(...)` to `bind` — obiekt, do którego sesja będzie wysyłać SQL. Może to być `Engine` albo bezpośrednio `Connection` (o tym drugim za chwilę).

### 3.2 Dlaczego fabryka, a nie jedna sesja

Gdybyś trzymał jedną sesję jako globalną zmienną, miałbyś trzy problemy:

- **Współdzielenie między wątkami.** `Session` nie jest bezpieczna wątkowo — dwie funkcje w dwóch wątkach używające tej samej sesji to gwarantowany bałagan (a w trybie `async` — wyjątek, patrz [`15_asynchronicznosc.md`](15_asynchronicznosc.md)).
- **Rosnąca pamięć.** Identity map rośnie z każdym odczytem. Sesja żyjąca tygodniami trzyma w pamięci miliony obiektów.
- **Transakcje.** Sesja ma jedną transakcję naraz. Chcesz osobne transakcje na osobne operacje — potrzebujesz osobnych sesji.

Rozwiązaniem jest **fabryka**: obiekt, który pamięta konfigurację i produkuje nowe sesje na żądanie.

```python
# examples/08_sessionmaker.py
from sqlalchemy import create_engine
from sqlalchemy.orm import Session, sessionmaker

engine = create_engine("sqlite:///library.db", echo=False)

SessionLocal = sessionmaker(
    bind=engine,              # do czego wysyłać SQL
    class_=Session,           # jakiej klasy użyć (domyślnie Session)
    expire_on_commit=False,   # omówimy w sekcji 9
    autoflush=True,           # omówimy w sekcji 8
)

# Każde wywołanie tworzy NOWĄ, niezależną sesję:
with SessionLocal() as session:
    ...  # transakcja nr 1

with SessionLocal() as session:
    ...  # transakcja nr 2, zupełnie inna
```

> 💡 **Analogia — fabryka kluczy.** `sessionmaker` to maszyna do wycinania kluczy. Ustawiasz ją raz (jaki wzór, jaki materiał) i potem wyciskasz klucze jeden po drugim. Każdy klucz jest osobny — jeden zgubiony nie psuje pozostałych. `SessionLocal` to maszyna, `SessionLocal()` to świeży klucz.

### 3.3 Konwencja nazewnictwa

W projektach przyjęło się:

- `engine` — jeden na proces, tworzony raz przy starcie.
- `SessionLocal` (albo `session_factory`, `async_session_maker`) — fabryka, tworzona raz.
- `session` — lokalna zmienna w funkcji, krótko żyjąca.

Zauważ, że `sessionmaker` **nie jest klasą** — to instancja klasy `sessionmaker`, która przy wywołaniu zwraca `Session`. Dlatego piszemy `SessionLocal()`, a nie `SessionLocal`. Typowo adnotuje się to jako:

```python
from sqlalchemy.orm import sessionmaker

SessionLocal: sessionmaker[Session] = sessionmaker(bind=engine)
```

### 3.4 `Session(engine)` czy `Session()` z bindowaniem

Możliwe są trzy warianty:

```python
# A. Wiązanie w konstruktorze — najczęstsze i najczystsze
with Session(engine) as session:
    ...

# B. Wiązanie po utworzeniu — dla kodu, który tworzy sesję wcześniej
session = Session()
session.bind = engine          # albo session.configure(bind=engine)
...

# C. Wiązanie do konkretnego połączenia — zaawansowane (testy, UoW)
with engine.connect() as conn:
    with Session(bind=conn) as session:
        ...                    # sesja użyje TEGO połączenia i jego transakcji
```

Wariant C jest ważny, choć rzadko używany na co dzień: pozwala **współdzielić jedną transakcję** między kodem ORM a kodem Core. Użyjemy go w modułach 18 (testy) i 21 (Unit of Work).

> ⚠️ **Pułapka.** Gdy wiążesz sesję do `Connection`, który **już ma rozpoczętą transakcję**, SQLAlchemy musi zdecydować, co zrobić z tą transakcją. Steruje tym parametr `join_transaction_mode` (domyślnie `"conditional_savepoint"` — sesja „wchodzi" w transakcję połączenia). Jeśli zobaczysz w logu `SAVEPOINT`, to właśnie to. Nie zmieniaj tego parametru bez potrzeby.

---

## 4. Trzy tryby pracy sesji

To sekcja, którą trzeba przeczytać uważnie, bo różnice są subtelne, a konsekwencje — poważne.

### 4.1 Tryb A: `with Session(engine) as session:` — tylko zamknięcie

```python
# examples/08_mode_a.py
from sqlalchemy import create_engine
from sqlalchemy.orm import Session

from examples.models import Book

engine = create_engine("sqlite:///library.db")

with Session(engine) as session:
    session.add(Book(title="Solaris", isbn="978-83-08-06026-2"))
    # ... i koniec bloku
```

Co się dzieje przy wyjściu z bloku? Wywoływane jest `session.close()`, które:

1. **przewija** bieżącą transakcję (`ROLLBACK`) — czyli **wszystkie zmiany przepadają**,
2. oddaje pożyczone połączenie do puli,
3. usuwa wszystkie obiekty z sesji (stają się *detached*).

W trybie A **musisz sam wywołać `commit()`**, inaczej dane nie trafią do bazy. To tryb dla kodu, który sam zarządza transakcją (np. chce commitować warunkowo, w pętli, albo wykonać `rollback` w `except`).

### 4.2 Tryb B: `with Session(engine) as session, session.begin():` — transakcja automatyczna

```python
# examples/08_mode_b.py
from sqlalchemy import create_engine
from sqlalchemy.orm import Session

from examples.models import Book

engine = create_engine("sqlite:///library.db")

with Session(engine) as session, session.begin():
    session.add(Book(title="Solaris", isbn="978-83-08-06026-2"))
    session.add(Book(title="Eden", isbn="978-83-08-06027-9"))
    # commit() wywoła się SAM przy wyjściu z bloku
    # a jeśli poleci wyjątek — sam rollback()
```

To **zalecany tryb domyślny**. `session.begin()` zwraca obiekt transakcji (`SessionTransaction`), który jako kontekst menedżera:

- przy normalnym wyjściu wywołuje `commit()`,
- przy wyjątku wywołuje `rollback()` i **przepuszcza wyjątek dalej**.

Wszystko albo nic — dokładnie to, czego chcesz w 95% przypadków.

> 🧠 **Dlaczego tak jest — „albo wszystko, albo nic" jest regułą, nie wyjątkiem.** Pomyśl o przelewie bankowym: nie może być tak, że pieniądze zniknęły z jednego konta i nie pojawiły się na drugim. Transakcja to obietnica, że zestaw operacji wykona się w całości lub wcale. Tryb B wpisuje tę obietnicę w strukturę bloku `with`, więc nie musisz o niej pamiętać.

### 4.3 Tryb C: ręczne zarządzanie

```python
# examples/08_mode_c.py
from sqlalchemy import create_engine
from sqlalchemy.orm import Session

from examples.models import Book

engine = create_engine("sqlite:///library.db")
session = Session(engine)

try:
    with session.begin():                 # albo session.commit() ręcznie
        session.add(Book(title="Solaris", isbn="978-83-08-06026-2"))
except Exception:
    session.rollback()                    # w trybie z with to zbędne
    raise
finally:
    session.close()                       # ZAWSZE — inaczej wyciek połączeń
```

Tego wzorca używaj tylko wtedy, gdy naprawdę potrzebujesz sesji w zmiennej (np. w klasie repozytorium z modułu 20). Kluczowa zasada: **`close()` musi się wykonać zawsze**. Bez tego połączenie nie wróci do puli i po kilkudziesięciu żądaniach aplikacja „zawiesi się" na oczekiwaniu na wolne połączenie.

### 4.4 Porównanie

| | Tryb A (`with Session`) | Tryb B (`with Session, session.begin()`) | Tryb C (ręcznie) |
|---|---|---|---|
| Kiedy commit | ręcznie | automatycznie przy wyjściu | ręcznie |
| Kiedy rollback przy wyjątku | **nie** (trzeba samemu) | automatycznie | ręcznie |
| Kiedy `close()` | automatycznie | automatycznie | ręcznie (`finally`) |
| Ryzyko wycieku połączenia | brak | brak | **wysokie** |
| Typowe użycie | skrypty sterujące transakcją | aplikacje, serwisy, endpointy | klasy, repozytoria, kod biblioteczny |

> 🔬 **Pod maską — jak wygląda początek transakcji.** W logu `echo=True` zobaczysz przy pierwszej operacji sesji:
> ```text
> BEGIN (implicit)
> SELECT books.id, books.title, books.isbn FROM books WHERE books.id = ?
> [generated in 0.00031s] (1,)
> ```
> `BEGIN (implicit)` to „autobegin" — sesja sama rozpoczęła transakcję, bo wykryła pierwszą operację wymagającą bazy. Nie musisz nigdzie pisać `BEGIN`; SQLAlchemy robi to za Ciebie leniwie (lazy — dopiero gdy trzeba).

> 🧪 **Ćwiczenie.** Uruchom ten sam skrypt trzy razy — w trybie A bez `commit()`, w trybie A z `commit()`, w trybie B. W każdym wypadku wypisz `session.scalar(select(func.count()).select_from(Book))` w **nowej** sesji po zakończeniu bloku. Kiedy liczba wynosi 0, a kiedy 1?

---

## 5. Stany obiektu — pełny cykl życia

Każdy mapowany obiekt Pythona znajduje się w jednym z pięciu stanów. To nie jest teoria — od tego stanu zależy, co sesja zrobi z obiektem.

### 5.1 Diagram przejść

```text
                        session.add(obj)
   ┌───────────┐  ───────────────────────────►  ┌───────────┐
   │ TRANSIENT │                                │  PENDING  │
   │  (ulotny) │  ◄───────────────────────────  │ (oczekujący)│
   └───────────┘      rollback() / expunge()     └─────┬─────┘
         ▲                                            │
         │                                            │ flush() → INSERT
         │                                            ▼
         │                                      ┌────────────┐
         │                                      │ PERSISTENT │ ◄──── session.get()
         │                                      │ (trwały)   │       SELECT z bazy
         │                                      └──┬──────┬──┘
         │                                         │      │
         │        session.delete(obj) + flush()    │      │  session.close()
         │                    ┌────────────────────┘      │  session.expunge(obj)
         │                    ▼                           │
         │              ┌───────────┐                     │
         │              │  DELETED  │                     │
         │              │ (usuwany) │                     │
         │              └─────┬─────┘                     │
         │                    │ commit()                  │
         │                    ▼                           ▼
         │              (wiersz znika z bazy)      ┌────────────┐
         └──────────────────────────────────────── │  DETACHED  │
                                                 │ (odłączony)│
                                                 └────────────┘
```

### 5.2 Transient (ulotny)

Obiekt istnieje tylko w pamięci Pythona. Sesja nic o nim nie wie, w bazie nie ma wiersza, nie ma klucza głównego.

```python
book = Book(title="Solaris", isbn="978-83-08-06026-2")
state = inspect(book)
print(state.transient)   # True
print(book.id)           # None — klucz nadaje baza, nie Python
```

### 5.3 Pending (oczekujący)

Po `session.add(book)` obiekt trafia do kolejki `session.new`. **Nadal nie ma SQL** — ale sesja już wie, że przy najbliższym fluszu ma wysłać INSERT.

```python
session = Session(engine)
session.add(book)
state = inspect(book)
print(state.pending)     # True
print(state.transient)   # False
print(len(session.new))  # 1
print(book.id)           # nadal None!
```

> 💡 **Analogia — kolejka w urzędzie.** `pending` to moment, w którym wziąłeś numerek. Jesteś w systemie, urzędnik wie, że istniejesz, ale sprawa nie jest jeszcze załatwiona. Dopiero `flush()` to moment, w którym okienko Cię wywołało.

### 5.4 Persistent (trwały)

Po fluszu obiekt ma wiersz w bazie i klucz główny. Od tego momentu sesja **śledzi każdą zmianę** jego atrybutów.

```python
session.flush()
state = inspect(book)
print(state.persistent)  # True
print(book.id)           # 1 — baza nadała klucz
print(state.identity)    # (1,) — tożsamość w bazie
print(state.key)         # (Book, (1,), None) — tożsamość w sesji
```

Ważne: od tego momentu **nie musisz nic robić**, żeby zapisać zmianę.

```python
book.title = "Solaris (wydanie drugie)"   # ZERO SQL
session.flush()                            # dopiero teraz: UPDATE books SET title=...
```

To jest cała magia ORM: przypisanie atrybutu jest rejestrowane przez mechanizm *instrumentacji* (atrybuty klas mapowanych to nie zwykłe atrybuty Pythona, tylko deskryptory, które zapisują zmianę w `InstanceState`).

> 🔬 **Pod maską — co dokładnie leci przy zmianie tytułu.**
> ```text
> UPDATE books SET title=? WHERE books.id = ?
> [generated in 0.00028s] ('Solaris (wydanie drugie)', 1)
> ```
> Zauważ dwie rzeczy: (1) SQLAlchemy aktualizuje **tylko zmienione kolumny** — nie wysyła całego wiersza; (2) używa parametrów wiązanych (`?`), więc nie ma mowy o wstrzyknięciu SQL.

### 5.5 Deleted (usuwany)

`session.delete(book)` przenosi obiekt do stanu `deleted`. Wiersz zniknie z bazy przy najbliższym fluszu.

```python
session.delete(book)
print(len(session.deleted))   # 1
print(inspect(book).deleted)  # True
session.flush()
# DELETE FROM books WHERE books.id = ?
```

Po `commit()` obiekt wraca do stanu *detached* i reprezentuje już nieistniejący wiersz. Próba ponownego `session.add()` takiego obiektu to typowy błąd — patrz tabela błędów.

### 5.6 Detached (odłączony)

Obiekt jest w pełni funkcjonalnym obiektem Pythona, ale **sesja już nim nie zarządza**. Nie ma automatycznych UPDATE-ów, nie ma dostępu do bazy.

Obiekt staje się *detached* po:

- `session.close()`,
- `session.expunge(obj)` (pojedynczy) / `session.expunge_all()`,
- `session.rollback()` — dla obiektów, które nie były jeszcze w bazie,
- przekazaniu do innego wątku/procesu (obiekt nie jest serializowalny razem z sesją!).

```python
with Session(engine) as session:
    book = session.get(Book, 1)
    print(inspect(book).persistent)   # True

print(inspect(book).detached)         # True — sesja zamknięta
print(book.title)                     # działa! atrybut był załadowany
```

> ⚠️ **Pułapka — `DetachedInstanceError`.** Dopóki atrybut był już wczytany, dostęp do niego na obiekcie detached działa. Ale gdy SQLAlchemy musi **dociągnąć** wartość (np. relację leniwie ładowaną — moduł 09/11, albo atrybut wygaszony po `commit` — sekcja 9), rzuci `DetachedInstanceError`. To najczęstszy błąd początkujących w aplikacjach webowych: obiekt wyjeżdża z sesji i przy serializacji do JSON wybucha.

### 5.7 Tabela zbiorcza

| Stan | `inspect(obj).X` | W bazie? | Śledzony? | Jak tu trafić |
|---|---|---|---|---|
| transient | `transient` | nie | nie | `Book(...)` |
| pending | `pending` | nie | tak | `session.add(obj)` |
| persistent | `persistent` | tak | tak | `session.flush()` / odczyt z bazy |
| deleted | `deleted` | tak (jeszcze) | tak | `session.delete(obj)` |
| detached | `detached` | tak | nie | `session.close()`, `expunge()` |

> 🧪 **Ćwiczenie.** Napisz funkcję `print_state(obj: object) -> None`, która wypisuje stan obiektu w czytelnej formie (np. `"persistent, id=1, zmieniony"`). Użyj jej w skrypcie, który przechodzi przez wszystkie pięć stanów po kolei, wypisując stan po każdej operacji.

---

## 6. Identity map — rejestr mieszkańców

### 6.1 Problem, który rozwiązuje

Wyobraź sobie aplikację, w której w trzech miejscach odczytujesz książkę o `id=1`:

```python
a = session.get(Book, 1)
b = session.get(Book, 1)
a.title = "Nowy tytuł"
print(b.title)   # co powinno tu być?
```

Bez identity map `a` i `b` to dwa różne obiekty. `b.title` zwróciłoby stary tytuł — mimo że w sesji „obowiązuje" już nowy. Jeszcze gorzej: przy commicie sesja nie wiedziałaby, który obiekt jest źródłem prawdy.

**Identity map** to słownik wewnątrz sesji: `(klasa, klucz główny) → obiekt Pythona`. Dzięki niemu sesja **gwarantuje**, że w obrębie jednej sesji istnieje dokładnie **jeden** obiekt Pythona reprezentujący dany wiersz.

```python
# examples/08_identity_map.py
from sqlalchemy import create_engine, select
from sqlalchemy.orm import Session

from examples.models import Book

engine = create_engine("sqlite:///library.db", echo=True)

with Session(engine) as session:
    a = session.get(Book, 1)
    b = session.get(Book, 1)

    print(a is b)                        # True — ten sam obiekt!
    print(len(session.identity_map))     # 1

    c = session.scalars(select(Book).where(Book.id == 1)).one()
    print(c is a)                        # True — mimo że SELECT naprawdę poszedł
```

Zwróć uwagę na ostatnią linię. `select(...)` **naprawdę wykonało zapytanie SQL** (zobaczysz je w logu), ale jego wynik nie stworzył nowego obiektu — SQLAlchemy rozpoznał klucz `(1,)` i „wpasował" wiersz w istniejący obiekt `a`.

> 🧠 **Dlaczego tak jest — tożsamość kontra cache.** To bardzo ważne rozróżnienie:
> - **Identity map NIE jest cachem wyników zapytań.** Zapytanie zawsze idzie do bazy (chyba że użyjesz `session.get()`, które najpierw sprawdza mapę).
> - Identity map gwarantuje **unikalność obiektu**, nie **aktualność danych**.
>
> Innymi słowy: SQL leci do bazy, ale wynik trafia do istniejącego obiektu — a SQLAlchemy **domyślnie nie nadpisuje** już załadowanych atrybutów. To zachowanie jest celowe (chroni Twoje niezapisane zmiany), ale ma skutek uboczny: dane w obiekcie mogą być starsze niż w bazie.

> 💡 **Analogia — rejestr mieszkańców.** W bloku mieszkalnym jest jeden rejestr: „lokal 1 → pan Kowalski". Niezależnie od tego, ile razy zapytasz administrację o lokal 1, dostaniesz **tę samą osobę**, a nie kopię pana Kowalskiego. Gdyby administracja tworzyła kopię przy każdym pytaniu, po pięciu pytaniach miałbyś pięciu panów Kowalskich i nie wiedziałbyś, który jest prawdziwy.

### 6.2 `session.get()` — jedyna metoda, która korzysta z mapy

`session.get(Klasa, pk)` działa tak:

1. Sprawdź identity map. Jeśli obiekt jest tam **i nie jest wygaszony** — zwróć go, **bez SQL**.
2. Jeśli go nie ma albo jest wygaszony — wyślij `SELECT ... WHERE id = ?` i zmapuj wynik.
3. Jeśli wiersza nie ma — zwróć `None`.

```python
with Session(engine) as session:
    b1 = session.get(Book, 1)     # SELECT leci
    b2 = session.get(Book, 1)     # ZERO SQL
    b3 = session.get(Book, 999)   # SELECT leci, zwraca None
```

### 6.3 Konsekwencja pierwsza: spójność tożsamości

Dzięki identity map możesz porównywać obiekty operatorem `is` i używać ich jako kluczy w słownikach. To bardzo praktyczne:

```python
books_by_author: dict[Book, list[str]] = {}
```

### 6.4 Konsekwencja druga: dane mogą być nieświeże

Scenariusz: dwie sesje, dwa połączenia (baza plikowa, nie `:memory:`).

```python
# examples/08_stale_data.py
from sqlalchemy import create_engine, select
from sqlalchemy.orm import Session

from examples.models import Book

engine = create_engine("sqlite:///library.db")

with Session(engine) as s1, Session(engine) as s2:
    book_1 = s1.get(Book, 1)                       # s1 ładuje tytuł "Solaris"

    book_2 = s2.get(Book, 1)
    book_2.title = "Solaris — wydanie poprawione"  # s2 zmienia tytuł
    s2.commit()                                    # ... i zapisuje

    again = s1.scalars(select(Book).where(Book.id == 1)).one()
    print(again is book_1)                         # True
    print(again.title)                             # "Solaris" — STARE dane!
```

Zapytanie poszło do bazy i baza zwróciła nowy tytuł — ale SQLAlchemy **nie nadpisał** już załadowanego atrybutu. To zachowanie jest spójne z ideą izolacji transakcji, ale bywa zaskakujące.

Jeśli naprawdę chcesz zobaczyć świeże dane w istniejącym obiekcie, masz trzy narzędzia:

```python
s1.refresh(book_1)                        # SELECT i nadpisanie atrybutów
s1.expire(book_1)                         # wygaś atrybuty; następny dostęp → SELECT
s1.expire(book_1, ["title"])              # wygaś tylko jeden atrybut
```

Albo — na poziomie zapytania — `execution_options(populate_existing=True)`:

```python
stmt = (
    select(Book)
    .where(Book.id == 1)
    .execution_options(populate_existing=True)   # nadpisz istniejący obiekt danymi z bazy
)
```

> ⚠️ **Pułapka — długo żyjąca sesja to rosnący słownik.** Identity map nigdy sama nie „zapomina" obiektów. Sesja żyjąca przez cały dzień pracy procesu trzyma w pamięci każdy wczytany wiersz. **Reguła: sesja powinna żyć krótko** — jeden request, jedno zadanie, jedna operacja. Dokładnie o tym mówi sekcja 12 i moduł 21.

> 🧪 **Ćwiczenie.** Zmierz `len(session.identity_map)` po każdym z 1000 odczytów (`session.get(Book, i)`) w jednej sesji. Potem powtórz eksperyment z nową sesją na każdy odczyt. Porównaj zużycie pamięci funkcją `tracemalloc`.

---

## 7. Unit of Work — kolejka zmian

### 7.1 Trzy kolejki

Sesja prowadzi trzy zbiory obiektów:

| Atrybut | Co zawiera | Odpowiadający SQL |
|---|---|---|
| `session.new` | obiekty `pending` — do wstawienia | `INSERT` |
| `session.dirty` | obiekty `persistent` ze zmienionymi atrybutami | `UPDATE` |
| `session.deleted` | obiekty oznaczone do usunięcia | `DELETE` |

```python
# examples/08_three_queues.py
from sqlalchemy import create_engine
from sqlalchemy.orm import Session

from examples.models import Book

engine = create_engine("sqlite:///library.db")

with Session(engine) as session, session.begin():
    existing = session.get(Book, 1)

    new_book = Book(title="Eden", isbn="978-83-08-06027-9")
    session.add(new_book)

    existing.title = "Solaris (wyd. 2)"

    print(len(session.new))       # 1  -> INSERT
    print(len(session.dirty))     # 1  -> UPDATE
    print(session.is_modified(existing))   # True
    # commit() wyśle oba polecenia
```

> 🧠 **Dlaczego tak jest — „brudny" znaczy zmieniony.** `dirty` nie oznacza „obiekt, który trzeba sprawdzić". SQLAlchemy dokładnie wie, **który atrybut** się zmienił (dzięki instrumentacji z sekcji 5). Możesz to obejrzeć:
>
> ```python
> from sqlalchemy.orm import attributes
>
> history = attributes.get_history(existing, "title")
> print(history.added)      # ['Solaris (wyd. 2)']  — nowa wartość
> print(history.deleted)    # ['Solaris']           — stara wartość
> print(history.unchanged)  # []
> ```
>
> Ten mechanizm wykorzystamy w module 13 do budowy dziennika audytu.

### 7.2 Sortowanie i łączenie operacji

Flush to nie „pętla po obiektach i wysyłanie po jednym". SQLAlchemy buduje **plan flushu**:

1. **Sortowanie topologiczne** po zależnościach kluczy obcych — najpierw tabele niezależne, potem zależne. Dzięki temu nigdy nie wysyła `INSERT` do tabeli `loans`, zanim istnieje wiersz w `books`.
2. **Kolejność w obrębie tabeli**: najpierw `INSERT`, potem `UPDATE`, potem `DELETE`. (Dlatego usunięcie i ponowne dodanie rekordu o tym samym kluczu w jednej transakcji bywa kłopotliwe — patrz tabela błędów.)
3. **Łączenie w partie.** Wiele `INSERT`-ów do tej samej tabeli idzie jednym poleceniem — to funkcja *insertmanyvalues*.

> 🔬 **Pod maską — dwa INSERT-y w jednym poleceniu.** Gdy dodasz dwie książki i wykonasz `flush()`, w logu na SQLite 3.35+ zobaczysz coś takiego:
> ```text
> INSERT INTO books (title, isbn, copies_total, copies_available)
> VALUES (?, ?, ?, ?), (?, ?, ?, ?) RETURNING id
> [generated in 0.00042s] [('Solaris', '978-...', 1, 1), ('Eden', '978-...', 1, 1)]
> ```
> Jedna runda do bazy zamiast dwóch — a przy 10 000 rekordów to różnica rzędu sekund. Gdyby baza nie umiała zwrócić wygenerowanych kluczy (`RETURNING`), SQLAlchemy wysłałby osobne `INSERT`-y dla każdego wiersza, bo po każdym musi poznać nadane `id`.

### 7.3 Autoflush — flush, o którym nie wiedziałeś

`autoflush` (domyślnie `True`) oznacza: **zanim sesja wykona zapytanie, najpierw wyśle oczekujące zmiany**. Powód jest prozaiczny: gdyby tego nie robiła, zapytanie mogłoby nie zobaczyć danych, które właśnie dodałeś.

```python
# examples/08_autoflush.py
from sqlalchemy import create_engine, select
from sqlalchemy.orm import Session

from examples.models import Book

engine = create_engine("sqlite:///library.db", echo=True)

with Session(engine) as session, session.begin():
    session.add(Book(title="Niezwyciężony", isbn="978-83-08-06028-6"))
    print("session.new:", len(session.new))        # 1

    # Ten SELECT spowoduje AUTOFLUSH — INSERT poleci PRZED zapytaniem:
    found = session.scalars(select(Book).where(Book.title == "Niezwyciężony")).all()
    print("znaleziono:", len(found))               # 1  ← bez autoflush byłoby 0
    print("session.new:", len(session.new))        # 0  ← kolejka opróżniona
```

W logu zobaczysz kolejność: `INSERT ...`, potem `SELECT ...`. To autoflush w akcji.

Wyłączenie autoflush daje odwrotny efekt:

```python
with Session(engine, autoflush=False) as session, session.begin():
    session.add(Book(title="Niezwyciężony", isbn="978-83-08-06028-6"))
    found = session.scalars(select(Book).where(Book.title == "Niezwyciężony")).all()
    print(len(found))     # 0! Zapytanie nie widzi jeszcze niezsynchronizowanej zmiany
```

> ⚠️ **Pułapka.** `autoflush=False` jest czasem zalecany jako „optymalizacja" — i faktycznie bywa przydatny przy profilowaniu albo przy bardzo długich transakcjach. Ale zmienia **semantykę** Twojego kodu: zapytania przestają widzieć świeże zmiany. Jeśli go wyłączasz, musisz sam wywoływać `session.flush()` przed każdym zapytaniem, które ma zależeć od niezapisanych zmian.

> 🆕 **SQLAlchemy 2.1 — autoflush działa bezwarunkowo.** W 2.0 istniały sytuacje, w których autoflush był pomijany (szczegóły opisuje changelog 2.1). W 2.1 zachowanie jest bezwarunkowe: jeśli `autoflush=True` i sesja ma oczekujące zmiany, zapytanie przez sesję zostanie poprzedzone flushem. **Praktyczny wniosek:** nie buduj logiki na założeniu „autoflush tu nie zadziała" — kod, który dziś działa dzięki takiemu pominięciu, po aktualizacji do 2.1 zacznie wysyłać dodatkowy SQL (albo — co gorsza — ujawni błąd integralności, który wcześniej był ukryty).

> 🧪 **Ćwiczenie.** Uruchom skrypt z tej sekcji w czterech wariantach: `autoflush=True/False` × `flush()` ręczny przed zapytaniem / bez. Zapisz w tabelce, ile wierszy widzi zapytanie w każdym z czterech przypadków. Wyciągnij regułę.

---

## 8. `flush()` kontra `commit()`

To najważniejsze rozróżnienie w module. Wiele osób miesza te dwa pojęcia i potem nie rozumie, dlaczego dane „znikają".

### 8.1 `flush()` — wyślij SQL, nie kończ transakcji

`flush()` robi trzy rzeczy:

1. Buduje plan flushu (sekcja 7.2) i wysyła `INSERT`/`UPDATE`/`DELETE` do bazy.
2. **Nadaje klucze główne** obiektom `pending` — po fluszu `book.id` jest już znane.
3. Nie kończy transakcji. Zmiany są widoczne **tylko wewnątrz tej transakcji** — inne połączenia ich nie widzą, a `rollback()` je cofnie.

```python
with Session(engine) as session, session.begin():
    book = Book(title="Solaris", isbn="978-83-08-06026-2")
    session.add(book)

    print(book.id)     # None — jeszcze nic nie poszło
    session.flush()
    print(book.id)     # 1 — INSERT już był, klucz nadany

    # ... i możemy to wszystko cofnąć:
    session.rollback()
```

Po `rollback()` obiekt `book` wraca do stanu *transient* (jeśli był tylko dodany) i traci klucz. Sprawdź: `inspect(book).transient` → `True`.

> 💡 **Analogia — wysłanie zamówienia do magazynu.** `flush()` to moment, w którym magazyn dostaje listę i zaczyna kompletować. `commit()` to moment, w którym paczka jest zapakowana, opłacona i wysłana. Między jednym a drugim możesz jeszcze wszystko odwołać — nawet jeśli magazyn już zaczął pracę.

### 8.2 `commit()` — zakończ transakcję na dobre

`commit()` robi cztery rzeczy:

1. **Wywołuje `flush()`** — jeśli masz oczekujące zmiany, nie musisz flushować ręcznie.
2. Wysyła `COMMIT` do bazy — dane są trwałe, inne połączenia je widzą.
3. **Wygasza wszystkie obiekty** w sesji (jeśli `expire_on_commit=True`, czyli domyślnie) — o tym cała sekcja 9.
4. **Kończy transakcję.** Kolejna operacja na sesji automatycznie rozpocznie **nową** transakcję (autobegin).

```python
# examples/08_flush_vs_commit.py
from sqlalchemy import create_engine, func, select
from sqlalchemy.orm import Session

from examples.models import Book

engine = create_engine("sqlite:///library.db", echo=True)

with Session(engine) as session:
    book = Book(title="Solaris", isbn="978-83-08-06026-2")
    session.add(book)
    session.flush()
    print("Po flush, w tej samej transakcji:", book.id)     # 1

    session.commit()
    print("Po commit:", book.id)                            # nadal 1 (choć wygaszony)

    # Nowa transakcja startuje sama przy następnej operacji
    total = session.scalar(select(func.count()).select_from(Book))
    print("Liczba książek:", total)
```

> 🔬 **Pod maską — kolejność zdarzeń w logu.**
> ```text
> BEGIN (implicit)                                     ← autobegin
> INSERT INTO books (title, isbn, ...) VALUES (?, ?, ...)
> [generated in 0.00031s] ('Solaris', '978-...', 1, 1)
> COMMIT                                               ← commit() kończy transakcję
> BEGIN (implicit)                                     ← nowa transakcja przy kolejnym zapytaniu
> SELECT count(*) AS count_1 FROM books
> [generated in 0.00024s] ()
> ROLLBACK                                             ← close() przewija pustą transakcję
> ```

### 8.3 Kiedy `flush` dzieje się sam

Flush jest wywoływany automatycznie w czterech sytuacjach:

| Sytuacja | Dlaczego |
|---|---|
| Przed zapytaniem (autoflush, domyślnie `True`) | żeby zapytanie widziało Twoje zmiany |
| Wewnątrz `commit()` | żeby zamienić kolejkę zmian w SQL |
| `session.begin_nested()` | żeby otworzyć savepoint na aktualnym stanie |
| Zmiana klucza głównego lub operacja na relacji wymagająca synchronizacji | o tym w module 09 |

### 8.4 `begin_nested()` — savepoint

Czasem chcesz mieć w środku dużej transakcji „podtransakcję", którą można cofnąć bez przewijania całości. Do tego służy savepoint.

```python
# examples/08_savepoint.py
from sqlalchemy import create_engine
from sqlalchemy.orm import Session

from examples.models import Book

engine = create_engine("sqlite:///library.db")

with Session(engine) as session, session.begin():
    session.add(Book(title="Solaris", isbn="978-83-08-06026-2"))

    try:
        with session.begin_nested():          # SAVEPOINT
            session.add(Book(title="Eden", isbn="978-83-08-06026-2"))  # duplikat ISBN!
            # commit savepointu nie dojdzie do skutku — leci IntegrityError
    except Exception as exc:
        print(f"Pozycja odrzucona, reszta transakcji żyje: {type(exc).__name__}")

    session.add(Book(title="Niezwyciężony", isbn="978-83-08-06027-9"))

# W bazie są Solaris i Niezwyciężony; Eden został cofnięty.
```

To wzorzec „import partii danych z pominięciem błędnych wierszy" — wrócimy do niego w module 05 (DML) i 24.

> ⚠️ **Pułapka — `session.begin()` dwa razy.** Gdy sesja ma już rozpoczętą transakcję (np. przez autobegin przy pierwszym `session.get()`), wywołanie `session.begin()` rzuci `InvalidRequestError: A transaction is already begun on this Session`. W trybie B jest to bezpieczne, bo `with Session(...) as session, session.begin():` startuje na świeżej sesji. Ale w trybie C musisz pilnować, żeby nie wywołać `begin()` dwa razy.

> 🧪 **Ćwiczenie.** Napisz skrypt, który w jednej transakcji dodaje trzy książki, ale drugą dodaje wewnątrz `begin_nested()` i celowo wywołuje tam błąd (np. duplikat `isbn`). Sprawdź w bazie, które książki się zapisały.

---

## 9. `expire_on_commit` — najważniejszy przełącznik tego modułu

### 9.1 Co robi

Po `commit()` SQLAlchemy **domyślnie wygasza** (`expire`) wszystkie obiekty w sesji. „Wygaszenie" oznacza: wartości atrybutów są wyrzucane z pamięci obiektu, ale obiekt **zostaje** w sesji i zna swój klucz główny. Kolejny dostęp do dowolnego atrybutu powoduje **automatyczny SELECT** i ponowne wypełnienie.

```python
# examples/08_expire_on_commit.py
from sqlalchemy import create_engine
from sqlalchemy.orm import Session

from examples.models import Book

engine = create_engine("sqlite:///library.db", echo=True)

with Session(engine) as session:
    book = session.get(Book, 1)          # SELECT
    book.title = "Solaris (wyd. 2)"

    session.commit()                     # UPDATE + COMMIT
    # W tym momencie obiekt jest WYGASZONY:
    # print(book.title)                  # ... spowodowałoby kolejny SELECT!

    print(book.title)                    # SELECT books.id, ... WHERE books.id = 1
```

> 🔬 **Pod maską — log po commicie.**
> ```text
> UPDATE books SET title=? WHERE books.id = ?
> [generated in 0.00026s] ('Solaris (wyd. 2)', 1)
> COMMIT
> BEGIN (implicit)                        ← nowa transakcja (bo odczyt wymaga bazy)
> SELECT books.id, books.title, books.isbn, books.copies_total, books.copies_available
> FROM books WHERE books.id = ?
> [generated in 0.00022s] (1,)
> ```

### 9.2 Dlaczego domyślnie `True`

Bo to **bezpieczniejsze**. Po commicie baza mogła zmienić dane, o których SQLAlchemy nie wie:

- `server_default` (wartość nadana przez bazę, np. `created_at = now()`),
- `onupdate` po stronie bazy,
- **triggery**,
- zmiany dokonane przez inną transakcję, która zdążyła się zatwierdzić,
- wygenerowane klucze i kolumny obliczane.

Gdyby obiekt nie był wygaszany, pokazywałby dane, których już nie ma. `expire_on_commit=True` mówi: „po commicie nie ufam pamięci, sprawdzę w bazie".

> 💡 **Analogia — paragon i kasa.** Po wyjściu ze sklepu (commit) wyrzucasz z pamięci ceny, które zapamiętałeś przy półce. Jeśli ktoś Cię zapyta „ile kosztował ten chleb?", sprawdzasz na paragonie (SELECT), a nie zgadujesz z pamięci.

### 9.3 Dlaczego `False` bywa wygodne — i niebezpieczne

W aplikacjach webowych `expire_on_commit=False` jest bardzo częste, bo:

- serializacja odpowiedzi (np. Pydantic) nie może wykonywać I/O,
- obiekt opuszczający sesję (detached) musi mieć dane w pamięci,
- dodatkowe SELECT-y po każdym commicie to strata czasu i źródło problemu N+1 (moduł 11).

```python
# examples/08_expire_false.py
from sqlalchemy import create_engine
from sqlalchemy.orm import Session, sessionmaker

engine = create_engine("sqlite:///library.db")

SessionLocal = sessionmaker(bind=engine, expire_on_commit=False)

with SessionLocal() as session, session.begin():
    book = session.get(Book, 1)
    book.title = "Solaris (wyd. 3)"

# Po wyjściu z bloku obiekt jest DETACHED, ale ma dane w pamięci:
print(book.title)     # "Solaris (wyd. 3)" — bez SQL, bez błędu
print(book.id)        # 1
```

Cena, jaką płacisz:

- **Ryzyko nieświeżych danych.** Jeśli w bazie jest trigger zmieniający `updated_at`, obiekt go nie zobaczy.
- **Ryzyko nadpisania cudzych zmian.** Obiekt ma stare wartości i przy następnym UPDATE może nadpisać coś, co zmienił ktoś inny.
- **Kłopot przy `server_default`.** Dodajesz wiersz, bazę prosisz o `created_at`, a w obiekcie masz `None` — bo nie ma odświeżenia.

> ⚠️ **Pułapka — mieszanie `expire_on_commit=False` z `server_default`.** To klasyczna kombinacja, która produkuje obiekty z `None` tam, gdzie w bazie jest wartość. Jeśli używasz `expire_on_commit=False` i kolumn wypełnianych przez bazę, po `flush()` wywołaj `session.refresh(obj)` dla tych kolumn.

> 🧠 **Dlaczego tak jest — to nie jest ustawienie „wydajność vs poprawność".** `expire_on_commit=False` nie jest szybszą wersją `True`. Zmienia **kontrakt**: „obiekt po commicie odzwierciedla bazę" staje się „obiekt po commicie odzwierciedla bazę tak, jak ją widziałem przed commitem". Wybieraj świadomie — w aplikacji webowej to zwykle właściwy wybór, w skryptach wsadowych z regułami w bazie — nie.

### 9.4 `refresh()` i `expire()` — sterowanie ręczne

```python
session.refresh(book)                  # SELECT i nadpisanie WSZYSTKICH atrybutów
session.refresh(book, ["created_at"])  # SELECT tylko wskazanych kolumn
session.expire(book)                   # wygaś wszystko (bez SELECT-a teraz)
session.expire(book, ["title"])        # wygaś jeden atrybut
```

> 🧪 **Ćwiczenie.** Uruchom ten sam skrypt dwa razy — z `expire_on_commit=True` i `False`. Policz, ile zapytań `SELECT` leci **po** commicie w każdym wariancie, przy dostępie do trzech obiektów. Zapisz liczby i wyciągnij wniosek dla aplikacji webowej.

---

## 10. Czytanie obiektów z bazy

### 10.1 Przegląd metod

| Metoda | Zwraca | Kiedy używać |
|---|---|---|
| `session.get(Book, pk)` | obiekt albo `None` | znasz klucz główny; korzysta z identity map |
| `session.get_one(Book, pk)` | obiekt albo wyjątek `NoResultFound` | znasz klucz i brak wiersza to **błąd** |
| `session.scalars(stmt)` | `ScalarResult` (obiekty) | zwykłe zapytania ORM |
| `session.scalar(stmt)` | pierwsza kolumna pierwszego wiersza | agregacje, `COUNT`, `EXISTS` |
| `session.execute(stmt)` | `Result` (wiersze) | zapytania zwracające kolumny, `INSERT ... RETURNING` |

```python
# examples/08_reading.py
from sqlalchemy import create_engine, func, select
from sqlalchemy.orm import Session

from examples.models import Book

engine = create_engine("sqlite:///library.db", echo=True)

with Session(engine) as session:
    # 1. Po kluczu głównym — najprościej i najszybciej
    book = session.get(Book, 1)
    print(book.title if book else "brak")

    # 2. Po kluczu, ale brak wiersza to błąd
    try:
        book = session.get_one(Book, 999)
    except Exception as exc:
        print("get_one:", type(exc).__name__)          # NoResultFound

    # 3. Zapytanie ORM — pełne obiekty
    titles = session.scalars(select(Book.title).order_by(Book.title)).all()
    print(titles)                                      # ['Eden', 'Solaris', ...]

    # 4. Zapytanie ORM — encje
    books = session.scalars(select(Book).where(Book.copies_available > 0)).all()
    print(len(books))

    # 5. Skalar — jedna wartość
    total = session.scalar(select(func.count()).select_from(Book))
    print("Liczba książek:", total)
```

> 🧠 **Dlaczego tak jest — `scalars` czy `scalar`?** Różnica jest jednoliterowa, a pomyłka kosztowna:
> - `scalars()` → **wiele** obiektów (zwraca `ScalarResult`, na którym wołasz `.all()`, `.first()`, `.one()`).
> - `scalar()` → **jedna wartość** (już wywołane `.first()` i wyciągnięta pierwsza kolumna).
>
> Praktyczna reguła: jeśli w `select()` masz jedną kolumnę albo agregację → `scalar()`. Jeśli encję lub wiele kolumn → `scalars()`.

### 10.2 `Result` i jego metody

`ScalarResult` i `Result` mają ten sam zestaw metod „pobierających":

| Metoda | Zachowanie przy 0 wierszy | Zachowanie przy >1 wierszy |
|---|---|---|
| `.all()` | `[]` | wszystkie |
| `.first()` | `None` | pierwszy |
| `.one()` | `NoResultFound` | `MultipleResultsFound` |
| `.one_or_none()` | `None` | `MultipleResultsFound` |
| `.unique()` | — | usuwa duplikaty obiektów (wymagane przy `joinedload` kolekcji — moduł 11) |

> ⚠️ **Pułapka — `Result` jest jednorazowy.** Nie możesz „przewinąć" wyniku dwa razy. Po `.all()` obiekt wyniku jest wyczerpany; kolejne `.all()` zwróci pustą listę (a w niektórych przypadkach `ResourceClosedError`). Zawsze przypisz wynik do zmiennej: `books = session.scalars(stmt).all()`.

### 10.3 `expunge()` — ręczne odłączenie

```python
with Session(engine) as session:
    book = session.get(Book, 1)
    session.expunge(book)          # obiekt staje się detached, sesja go zapomina
    print(inspect(book).detached)  # True
    session.expunge_all()          # odłącz wszystko (przydatne przed długim przetwarzaniem)
```

`expunge_all()` bywa użyteczne w zadaniach wsadowych: po przetworzeniu partii obiektów odłączasz je, żeby identity map nie rosła. Uwaga — tracisz też śledzenie zmian, więc najpierw `flush()`.

> 🧪 **Ćwiczenie.** Wczytaj 10 000 książek w jednej sesji i zmierz `len(session.identity_map)` oraz pamięć przez `tracemalloc`. Powtórz, wywołując `session.expunge_all()` co 1000 wierszy. Jaka jest różnica?

---

## 11. Obiekty z zewnątrz: `merge()`, `add_all()`, `delete()`

### 11.1 `add_all()` — dodawanie hurtem

```python
books = [
    Book(title="Solaris", isbn="978-83-08-06026-2"),
    Book(title="Eden", isbn="978-83-08-06027-9"),
    Book(title="Niezwyciężony", isbn="978-83-08-06028-6"),
]
session.add_all(books)   # to samo co trzy razy session.add()
session.commit()
```

`add_all()` **nie wysyła SQL** — tylko dodaje obiekty do kolejki. Flush (jeden, połączony) nastąpi przy commicie.

### 11.2 `delete()` — usuwanie

```python
with Session(engine) as session, session.begin():
    book = session.get(Book, 1)
    session.delete(book)
    # DELETE poleci przy commicie
```

Możesz też usunąć bez wczytywania obiektu, zapytaniem DML (to szybsze i nie dotyka identity map):

```python
from sqlalchemy import delete

with Session(engine) as session, session.begin():
    session.execute(delete(Book).where(Book.copies_available == 0))
```

To tak zwany **ORM-enabled DELETE** — omówimy go dokładnie w module 10. Uwaga na `synchronize_session`: przy DML przez `session.execute()` sesja musi zdecydować, co zrobić z obiektami w pamięci, które właśnie zniknęły z bazy.

> ⚠️ **Pułapka — usuwanie obiektu *transient*.** `session.delete(book)` na obiekcie, który nigdy nie był w sesji, rzuci `InvalidRequestError: Instance '<Book at 0x...>' is not persisted`. Najpierw `add()`, potem `delete()` — albo od razu DML.

### 11.3 `merge()` — obiekty z zewnątrz

To metoda, która ratuje w aplikacjach webowych. Scenariusz: dane przyszły z API (JSON), z formularza albo z innego procesu. Masz obiekt Pythona z ustawionym `id`, ale **nie należy on do żadnej sesji**.

```python
# examples/08_merge.py
from sqlalchemy import create_engine
from sqlalchemy.orm import Session

from examples.models import Book

engine = create_engine("sqlite:///library.db")

# 1. Obiekt pochodzi z zewnątrz (np. zdeserializowany z JSON-a)
incoming = Book(id=1, title="Solaris — nowy tytuł", isbn="978-83-08-06026-2",
                copies_total=3, copies_available=2)

# 2. Obiekt jest DETACHED — nie należy do żadnej sesji
from sqlalchemy import inspect
print(inspect(incoming).transient)     # True — nikt o nim nie wie

with Session(engine) as session, session.begin():
    persistent = session.merge(incoming)

    print(persistent is incoming)      # False! merge zwraca INNY obiekt
    print(inspect(persistent).persistent)  # True
    # merge wykonał: SELECT (jeśli trzeba) + UPDATE przy commicie
```

Najważniejsze fakty o `merge()`:

1. **Nie dodaje** przekazanego obiektu do sesji. Zwraca **inny** obiekt — taki, którym sesja zarządza (istniejący albo nowo utworzony).
2. Dla obiektu z kluczem, który już istnieje w bazie, merge **kopiuje atrybuty** na obiekt w sesji i zaplanuje `UPDATE`.
3. Dla obiektu bez klucza (lub z nieistniejącym kluczem) merge **tworzy nowy** obiekt w sesji i zaplanuje `INSERT`.
4. Domyślnie SQLAlchemy najpierw sprawdza bazę (`SELECT`) — chyba że obiekt jest już w identity map.

> 💡 **Analogia — przepisanie danych na nowy formularz.** `merge()` to nie „wrzucenie starego dokumentu do segregatora". To przepisanie treści ze starego dokumentu na nowy formularz, który już leży w segregatorze. Stary dokument zostaje na biurku — możesz go wyrzucić albo zachować.

**Kiedy używać `merge()`, a kiedy nie:**

| Sytuacja | Rozwiązanie |
|---|---|
| Obiekt wczytany w tej samej sesji i zmodyfikowany | nic nie rób — sesja sama wykryje zmiany |
| Obiekt z innej sesji (np. cache, wątek) | `merge()` |
| Dane z JSON-a/API z kluczem głównym | `merge()` — ale rozważ DTO i ręczne wypełnienie encji (moduł 19) |
| Dane z JSON-a **bez** klucza | `add()` nowego obiektu |
| PATCH — aktualizacja tylko części pól | lepiej `session.get()` + przypisanie pól; `merge()` nadpisałby wszystko |

> ⚠️ **Pułapka — `merge()` i brakujące pola.** `merge()` kopiuje **wszystkie** atrybuty, także te, których nie ustawiłeś. Jeśli obiekt z zewnątrz ma `copies_total=0` jako wartość domyślną, bo „nie wiedział" o tym polu, merge wyzeruje prawdziwą wartość w bazie. Przy aktualizacjach częściowych zawsze wczytaj obiekt i ustaw tylko zmienione pola.

> 🧪 **Ćwiczenie.** Zaimplementuj funkcję `upsert_book(session: Session, payload: dict[str, object]) -> Book`, która na podstawie słownika z JSON-a albo aktualizuje istniejącą książkę, albo tworzy nową. Użyj `session.get()` i ręcznego przypisania pól — a potem napisz drugą wersję z `merge()` i porównaj zachowanie dla PATCH-a z jednym polem.

---

## 12. Sesja w kontekście aplikacji

### 12.1 Jedna sesja na jednostkę pracy

Reguła praktyczna, którą powtarzamy w całym kursie:

| Kontekst | Jednostka pracy | Wzorzec |
|---|---|---|
| Skrypt CLI | jedno uruchomienie (albo jeden etap) | `with Session(engine) as s, s.begin():` |
| Endpoint HTTP | jedno żądanie | sesja z zależności (FastAPI `Depends`) — moduł 22 |
| Zadanie w kolejce | jedno zadanie | sesja tworzona na początku zadania |
| Test | jeden test | fixture z `sessionmaker` — moduł 18 |
| Zadanie wsadowe (batch) | jedna partia (np. 1000 rekordów) | sesja per partia, `expunge_all()` między partiami |

### 12.2 Dlaczego nie dzielić sesji między wątkami

`Session` **nie jest bezpieczna wątkowo**. Trzy konkretne problemy:

1. **Transakcja i połączenie.** Sesja ma jedno połączenie i jedną transakcję naraz. Dwa wątki piszące w tej samej transakcji nie mają pojęcia o swoich zmianach.
2. **Identity map.** To zwykły słownik bez blokad. Równoczesny zapis z dwóch wątków może go uszkodzić.
3. **Stan `InstanceState`.** Każdy obiekt ma jeden rekord stanu — dwa wątki modyfikujące ten sam obiekt to wyścig.

W trybie `async` jest jeszcze ostrzej: `AsyncSession` zgłasza błąd, jeśli zostanie użyta z dwóch zadań równolegle (szczegóły w module 15).

### 12.3 `scoped_session` — czym jest i kiedy ma sens

`scoped_session` to opakowanie, które trzyma **jedną sesję na zakres** (domyślnie: na wątek). Wywołanie `SessionLocal()` zwraca zawsze tę samą sesję w obrębie wątku.

```python
# examples/08_scoped_session.py
from sqlalchemy import create_engine
from sqlalchemy.orm import scoped_session, sessionmaker

engine = create_engine("sqlite:///library.db")

session_factory = sessionmaker(bind=engine)
Session = scoped_session(session_factory)      # nazwa zwyczajowa: duże "S"

def handle_request() -> None:
    session = Session()                        # ta sama sesja w tym wątku
    ...
    Session.remove()                           # KLUCZOWE: sprzątanie po żądaniu

# W innym wątku Session() zwróci już inną sesję
```

> 🧠 **Dlaczego tak jest — `scoped_session` to relikt epoki bez wstrzykiwania zależności.** Powstał, gdy frameworki nie miały porządnego mechanizmu DI. W nowoczesnym kodzie (FastAPI, `dependency-injector`, ręczne fabryki z modułu 21) **nie potrzebujesz `scoped_session`** — przekazujesz sesję jawnie jako argument. Zostawiamy go w kursie, bo:
> - spotkasz go w istniejących projektach (Flask, stare aplikacje),
> - musisz wiedzieć, że **wymaga `remove()`**, inaczej wycieka połączenia.
>
> **Zasada:** jeśli nie masz konkretnego powodu, używaj jawnego przekazywania sesji (`sessionmaker` + parametr funkcji).

### 12.4 Zapowiedź: sesja w architekturze

W module 21 zbudujemy `UnitOfWork` — klasę, która tworzy sesję, udostępnia repozytoria i sama pilnuje granicy transakcji:

```python
# Zapowiedź — pełna implementacja w module 21
with UnitOfWork(session_factory) as uow:
    book = uow.books.get(1)
    uow.books.add(Book(...))
    # commit lub rollback dzieje się w __exit__
```

Na razie zapamiętaj intencję: **sesja jest zasobem o jasnym początku i końcu**, a nie globalną zmienną.

> 🧪 **Ćwiczenie.** Napisz program z dwoma wątkami (`threading.Thread`), w którym każdy wątek wczytuje i modyfikuje inną książkę. Najpierw spróbuj użyć **jednej wspólnej sesji** — zobacz, co się dzieje. Potem popraw kod tak, aby każdy wątek miał własną sesję. Zapisz obserwacje.

---

## 13. Pełny przykład: symulator dziennika zmian

Ten skrypt jest sercem modułu. Uruchom go, a zobaczysz w logu **dokładnie**, w których momentach leci SQL.

```python
# examples/08_session_lifecycle.py
"""Symulator dziennika zmian: kiedy SQLAlchemy wysyła SQL, a kiedy nie."""

from __future__ import annotations

from sqlalchemy import String, create_engine, func, inspect, select
from sqlalchemy.orm import DeclarativeBase, Mapped, Session, mapped_column


class Base(DeclarativeBase):
    pass


class Book(Base):
    __tablename__ = "books"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    isbn: Mapped[str] = mapped_column(String(20), unique=True)
    copies_total: Mapped[int] = mapped_column(default=1)
    copies_available: Mapped[int] = mapped_column(default=1)

    def __repr__(self) -> str:
        return f"<Book id={self.id} title={self.title!r}>"


def stage(label: str) -> None:
    """Wypisuje wyraźny nagłówek etapu, żeby nie zgubić się w logu SQL."""
    print(f"\n{'=' * 70}\n=== {label}\n{'=' * 70}")


engine = create_engine("sqlite://", echo=True)   # baza w pamięci, log SQL włączony
Base.metadata.create_all(engine)                  # DDL — tabele

# ---------------------------------------------------------------------------
stage("1. TRANSIENT — obiekt istnieje tylko w Pythonie")
book = Book(title="Solaris", isbn="978-83-08-06026-2")
state = inspect(book)
print(f"transient={state.transient} pending={state.pending} id={book.id!r}")

with Session(engine) as session:
    # -----------------------------------------------------------------------
    stage("2. add() — obiekt w kolejce, ZERO SQL")
    session.add(book)
    state = inspect(book)
    print(f"pending={state.pending} new={len(session.new)} id={book.id!r}")

    # -----------------------------------------------------------------------
    stage("3. flush() — INSERT leci, klucz zostaje nadany")
    session.flush()
    state = inspect(book)
    print(f"persistent={state.persistent} new={len(session.new)} id={book.id}")

    # -----------------------------------------------------------------------
    stage("4. zmiana atrybutu — ZERO SQL (na razie)")
    book.title = "Solaris (wydanie drugie)"
    print(f"dirty={len(session.dirty)} is_modified={session.is_modified(book)}")

    # -----------------------------------------------------------------------
    stage("5. AUTOFLUSH — zapytanie wymusza wysłanie zmian")
    session.add(Book(title="Niezwyciężony", isbn="978-83-08-06027-9"))
    print(f"przed zapytaniem: new={len(session.new)}, dirty={len(session.dirty)}")

    found = session.scalars(select(Book).order_by(Book.id)).all()
    print(f"po zapytaniu: znaleziono={len(found)}, new={len(session.new)}")
    print(f"wszystkie obiekty to te same instancje: {all(b is not None for b in found)}")

    # -----------------------------------------------------------------------
    stage("6. commit() — COMMIT + wygaśnięcie obiektów")
    session.commit()
    print(f"expired po commicie: {inspect(book).expired}")

    # -----------------------------------------------------------------------
    stage("7. odczyt wygaszonego atrybutu — SELECT odświeżający")
    print(f"book.title -> {book.title!r}")

    # -----------------------------------------------------------------------
    stage("8. identity map — ten sam obiekt Pythona")
    a = session.get(Book, book.id)     # obiekt jest wygaszony -> SELECT
    b = session.get(Book, book.id)     # świeży -> ZERO SQL
    print(f"a is b -> {a is b}, rozmiar identity_map={len(session.identity_map)}")

    # -----------------------------------------------------------------------
    stage("9. expire() i refresh()")
    session.expire(book)
    print(f"expired po expire(): {inspect(book).expired}")
    print(f"book.id -> {book.id}")     # SELECT
    session.refresh(book)              # SELECT niezależnie od stanu
    print("refresh() wykonany")

    # -----------------------------------------------------------------------
    stage("10. delete() — obiekt trafia do kolejki usunięć")
    session.delete(b)
    print(f"deleted={len(session.deleted)}, stan={inspect(b).deleted}")

# ---------------------------------------------------------------------------
stage("11. Po zamknięciu sesji obiekt jest DETACHED")
print(f"detached={inspect(book).detached}")
print(f"book.title -> {book.title!r} (dane wciąż w pamięci)")

# ---------------------------------------------------------------------------
stage("12. Brak commit = brak zmian")
with Session(engine) as session:
    session.add(Book(title="Eden", isbn="978-83-08-06028-6"))
    # UWAGA: brak commit() -> przy wyjściu z bloku leci ROLLBACK

with Session(engine) as session:
    total = session.scalar(select(func.count()).select_from(Book))
    print(f"Liczba książek w bazie: {total}")   # 0 — wszystko zostało cofnięte
```

**Jak uruchomić:**

```bash
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install "SQLAlchemy>=2.0"
python examples/08_session_lifecycle.py
```

**Co zaobserwujesz** (skrót z logu):

| Etap | Linie SQL w logu |
|---|---|
| 1–2 | *(brak — tylko obiekty Pythona)* |
| 3 | `INSERT INTO books (...) VALUES (?, ?, ?, ?)` |
| 4 | *(brak)* |
| 5 | `INSERT INTO books ...` (autoflush), potem `SELECT books...` |
| 6 | `UPDATE books SET title=? WHERE books.id = ?`, potem `COMMIT` |
| 7 | `BEGIN (implicit)`, `SELECT books.id, ... WHERE books.id = ?` |
| 8 | `SELECT ... WHERE books.id = ?` (dla `a`), *(brak dla `b`)* |
| 9 | `SELECT ...` z `expire()`, `SELECT ...` z `refresh()` |
| 10 | `DELETE FROM books WHERE books.id = ?` |
| 11 | `ROLLBACK` (z `close()`), brak SQL przy odczycie atrybutu |
| 12 | `ROLLBACK` — Eden nigdy nie trafił do bazy |

> 🔬 **Pod maską — cały cykl życia w jednym logu.** Zwróć uwagę na dwie rzeczy, które są sednem tego modułu:
> 1. **Ilość SQL jest znacznie mniejsza niż liczba operacji w Pythonie.** Dziesięć operacji na obiektach to kilka poleceń SQL.
> 2. **Każde polecenie SQL ma jasną przyczynę** — flush, autoflush, commit, wygaśnięcie, odczyt. Nie ma „magii": jest notes i są momenty, w których notes zamienia się w polecenia.

---

## Podsumowanie

1. **`Session` to nie połączenie.** To menedżer stanu: rejestr tożsamości (identity map), kolejka zmian (unit of work) i zakres transakcji. Połączenie pożycza z puli dopiero wtedy, gdy wysyła SQL.
2. **Trzy role w jednym obiekcie** wyjaśniają wszystkie „dziwne" zachowania ORM: dlaczego dwa zapytania dają ten sam obiekt, dlaczego zmiana atrybutu nie leci od razu do bazy i dlaczego `close()` cofa zmiany.
3. **`add()`, zmiana atrybutu i `delete()` nie wysyłają SQL.** SQL leci przy `flush()` (jawnym albo automatycznym) oraz przy `commit()`. Brak `commit()` to najczęstsza przyczyna „danych, których nie ma".
4. **Pięć stanów obiektu:** transient → pending → persistent → deleted → detached. Sprawdzasz je przez `inspect(obj).transient/.pending/.persistent/.deleted/.detached`.
5. **Identity map gwarantuje unikalność obiektu, nie aktualność danych.** Zapytanie może pójść do bazy, a i tak nie nadpisać już załadowanych atrybutów. Do wymuszenia świeżości służą `refresh()`, `expire()` i `populate_existing=True`.
6. **Flush sortuje i łączy operacje.** SQLAlchemy wysyła INSERT-y przed UPDATE-ami, respektuje kolejność kluczy obcych i potrafi połączyć wiele wstawień w jedno polecenie.
7. **Autoflush to bezpiecznik spójności**, nie optymalizacja. Wyłączenie go zmienia semantykę zapytań. W SQLAlchemy 2.1 autoflush działa bezwarunkowo.
8. **`expire_on_commit=True` (domyślnie) chroni przed nieświeżymi danymi** kosztem dodatkowych SELECT-ów po commicie. `False` jest wygodne w aplikacjach webowych, ale przenosi odpowiedzialność za świeżość danych na Ciebie.
9. **`session.get()` korzysta z identity map, `scalars()`/`scalar()` nie** — ale wynik zapytania i tak trafia do istniejących obiektów.
10. **Sesja żyje krótko i nie dzieli się między wątkami.** Jedna sesja = jedna jednostka pracy (request, zadanie, partia). `scoped_session` wymaga `remove()` i w nowoczesnym kodzie jest rzadko potrzebne.

---

## Ćwiczenia

### Ćwiczenie 1 — Autoflush pod kontrolą (łatwe)

Masz kod:

```python
with Session(engine) as session, session.begin():
    session.add(Book(title="Nowa", isbn="111"))
    count = session.scalar(select(func.count()).select_from(Book))
    print(count)
```

**Pytanie:** ile książek wypisze `count` przy `autoflush=True`, a ile przy `False`? Odpowiedz **najpierw bez uruchamiania**, potem sprawdź.

### Ćwiczenie 2 — `expire_on_commit` w praktyce (średnie)

Napisz skrypt, który:

1. tworzy `sessionmaker` z `expire_on_commit=True`, a potem drugi z `False`,
2. w obu wariantach: wczytuje trzy książki, zmienia tytuł jednej, robi `commit()`, a następnie wypisuje tytuły wszystkich trzech,
3. zlicza, ile poleceń `SELECT` poleciało **po** commicie w każdym wariancie.

Wnioski zapisz w tabeli: liczba zapytań, ryzyko nieświeżych danych, wygoda w aplikacji webowej.

### Ćwiczenie 3 — Własne narzędzie „dziennik zmian" (trudne)

Zbuduj funkcję pomocniczą:

```python
def dump_pending(session: Session) -> str:
    """Zwraca czytelny opis tego, co sesja ma do wysłania."""
```

Powinna wypisywać: liczbę i identyfikatory obiektów w `session.new`, `session.dirty`, `session.deleted`, a dla obiektów z `dirty` — **nazwy zmienionych atrybutów** (użyj `sqlalchemy.orm.attributes.get_history`). Następnie użyj jej w skrypcie, który wykonuje 6 różnych operacji i po każdej wypisuje stan kolejki.

### Rozwiązania

#### Rozwiązanie 1

- **`autoflush=True` (domyślnie): `1`.**
  Zapytanie `select(func.count())` idzie przez sesję, więc przed jego wykonaniem autoflush wysyła `INSERT` nowo dodanej książki. Baza widzi 1 wiersz.
- **`autoflush=False`: `0`.**
  INSERT jeszcze nie poleciał, a zapytanie widzi tylko to, co jest w bazie.

> 🧠 Warto zauważyć, że **nie chodzi o to, że baza „nie widzi" własnej transakcji** — chodzi o to, że SQL **jeszcze nie został wysłany**. Gdybyś wywołał `session.flush()` ręcznie, `count` wyniósłby `1` także przy `autoflush=False`.

#### Rozwiązanie 2

```python
# examples/08_solution_expire.py
from sqlalchemy import String, create_engine, event, select
from sqlalchemy.orm import DeclarativeBase, Mapped, Session, mapped_column, sessionmaker


class Base(DeclarativeBase):
    pass


class Book(Base):
    __tablename__ = "books"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    isbn: Mapped[str] = mapped_column(String(20), unique=True)


engine = create_engine("sqlite:///library.db")
Base.metadata.create_all(engine)

# Szybkie wypełnienie danych startowych
with Session(engine) as s, s.begin():
    if s.scalar(select(Book.id).limit(1)) is None:
        s.add_all([Book(title=f"Książka {i}", isbn=f"isbn-{i}") for i in range(1, 4)])


def count_selects(session: Session, fn) -> tuple[object, int]:
    """Uruchamia fn i zlicza polecenia SELECT wykonane w tym czasie."""
    counter = {"n": 0}

    @event.listens_for(session, "do_orm_execute")
    def _count(state) -> None:              # type: ignore[no-untyped-def]
        if state.is_select:
            counter["n"] += 1

    result = fn()
    event.remove(session, "do_orm_execute", _count)
    return result, counter["n"]


for expire in (True, False):
    Maker = sessionmaker(bind=engine, expire_on_commit=expire)

    def scenario() -> list[str]:
        with Maker() as session:
            books = session.scalars(select(Book).order_by(Book.id)).all()
            books[0].title = f"Zmieniony (expire={expire})"
            session.commit()
            return [b.title for b in books]     # dostępy po commicie

    titles, selects = count_selects(Maker(), scenario)
    print(f"expire_on_commit={expire}: SELECT-ów={selects}, tytuły={titles}")
```

Typowy wynik:

```text
expire_on_commit=True:  SELECT-ów=5, tytuły=['Zmieniony (expire=True)', 'Książka 2', 'Książka 3']
expire_on_commit=False: SELECT-ów=1, tytuły=['Zmieniony (expire=False)', 'Książka 2', 'Książka 3']
```

Interpretacja: przy `True` po commicie każde dotknięcie atrybutu to osobny `SELECT` (3 obiekty = 3 odświeżenia, plus zapytanie wstępne i ewentualne sprawdzenie). Przy `False` nie ma żadnego odświeżenia — ale obiekty pokazują stan z chwili sprzed commitu.

| | `expire_on_commit=True` | `expire_on_commit=False` |
|---|---|---|
| SELECT-y po commicie | wiele (po jednym na obiekt przy pierwszym dostępie) | zero |
| Świeżość danych | gwarantowana | stan z chwili commitu |
| Ryzyko w web | obiekty detached wybuchają `DetachedInstanceError` | wygodne, ale trzeba pilnować świeżości |
| Kiedy wybrać | skrypty, reguły w bazie, triggery | aplikacje webowe z serializacją odpowiedzi |

#### Rozwiązanie 3

```python
# examples/08_solution_dump_pending.py
from __future__ import annotations

from sqlalchemy import String, create_engine, inspect, select
from sqlalchemy.orm import (
    DeclarativeBase,
    Mapped,
    Session,
    attributes,
    mapped_column,
)


class Base(DeclarativeBase):
    pass


class Book(Base):
    __tablename__ = "books"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    isbn: Mapped[str] = mapped_column(String(20), unique=True)
    copies_available: Mapped[int] = mapped_column(default=1)


def dump_pending(session: Session) -> str:
    """Czytelny opis kolejki zmian w sesji."""

    def describe(obj: object) -> str:
        state = inspect(obj)
        identity = state.identity if state.identity else ("?",)
        return f"{type(obj).__name__}{identity}"

    lines: list[str] = []

    if session.new:
        lines.append(f"  new     ({len(session.new)}): {[describe(o) for o in session.new]}")
    if session.dirty:
        lines.append(f"  dirty   ({len(session.dirty)}):")
        for obj in session.dirty:
            changed: list[str] = []
            for key in inspect(obj).attrs.keys():
                history = attributes.get_history(obj, key)
                if history.added and history.deleted:
                    changed.append(f"{key}: {history.deleted[0]!r} -> {history.added[0]!r}")
                elif history.added and history.unchanged:
                    changed.append(f"{key}: (ustawione na) {history.added[0]!r}")
            lines.append(f"    {describe(obj)} -> {changed or ['(brak zmian atrybutów)']}")
    if session.deleted:
        lines.append(f"  deleted ({len(session.deleted)}): {[describe(o) for o in session.deleted]}")

    return "\n".join(lines) if lines else "  (kolejka pusta)"


engine = create_engine("sqlite:///library.db")
Base.metadata.create_all(engine)

with Session(engine) as session, session.begin():
    session.add_all([Book(title=f"Książka {i}", isbn=f"isbn-{i}") for i in range(1, 4)])

with Session(engine) as session:
    print("1) świeża sesja:")
    print(dump_pending(session))

    session.add(Book(title="Nowa", isbn="new-1"))
    print("2) po add():")
    print(dump_pending(session))

    existing = session.scalars(select(Book).order_by(Book.id)).first()
    existing.title = "Zmieniony tytuł"                       # type: ignore[union-attr]
    existing.copies_available = 7                            # type: ignore[union-attr]
    print("3) po modyfikacji istniejącego obiektu:")
    print(dump_pending(session))

    session.flush()
    print("4) po flush():")
    print(dump_pending(session))

    session.delete(existing)                                 # type: ignore[arg-type]
    print("5) po delete():")
    print(dump_pending(session))

    session.commit()
    print("6) po commit():")
    print(dump_pending(session))
```

Przykładowe wyjście:

```text
3) po modyfikacji istniejącego obiektu:
  new     (1): ['Book(?,)']
  dirty   (1):
    Book(1,) -> ['title: 'Książka 1' -> 'Zmieniony tytuł'', 'copies_available: 1 -> 7']
```

Warto zwrócić uwagę na krok 6: po `commit()` kolejka jest pusta, bo flush już wysłał wszystko, a transakcja została zamknięta.

---

## Najczęstsze błędy i jak je czytać

| Komunikat / objaw | Przyczyna | Naprawa |
|---|---|---|
| *„Dodałem obiekt, ale w bazie go nie ma"* (brak błędu!) | brak `commit()` — przy wyjściu z `with Session(...)` leci `ROLLBACK` | użyj `with Session(...) as s, s.begin():` albo wywołaj `s.commit()` |
| *„Zmieniłem atrybut, ale baza ma starą wartość"* | brak `commit()` **albo** brak `add()` (obiekt poza sesją) | upewnij się, że obiekt jest `persistent` (`inspect(obj).persistent`) i wywołaj `commit()` |
| `DetachedInstanceError: Instance <Book at 0x...> is not bound to a Session; attribute refresh operation cannot proceed` | obiekt opuścił sesję (close/expunge), a kod próbuje dociągnąć atrybut, który nie był załadowany (relacja leniwa, atrybut wygaszony po commicie) | ładuj relacje eagerly (moduł 11), ustaw `expire_on_commit=False`, albo pracuj na DTO w obrębie sesji |
| `InvalidRequestError: Object '<Book at 0x...>' is already attached to session '2' (this is '1')` | ten sam obiekt próbujesz dodać do drugiej sesji | użyj `session.merge(obj)` albo odtwórz obiekt z danych |
| `PendingRollbackError: This Session's transaction has been rolled back due to a previous exception during flush.` | wcześniej poleciał wyjątek (np. `IntegrityError`), a Ty wykonujesz kolejne operacje bez `rollback()` | po przechwyceniu wyjątku zrób `session.rollback()` (albo `with session.begin():` zamiast ręcznego sterowania) |
| `InvalidRequestError: This session is in 'prepared' state; no further SQL can be emitted within this transaction.` | próba wykonania SQL między `commit()` a zakończeniem dwufazowego commitu, albo operacja wewnątrz zdarzenia flushu | nie wykonuj zapytań w listenerach `before_flush`/`after_flush`; nie mieszaj `two_phase` z dalszą pracą na sesji |
| `InvalidRequestError: Instance '<Book at 0x...>' is not persisted` | `session.delete(obj)` na obiekcie, który nigdy nie był w sesji (`transient`) | najpierw `session.add(obj)`, potem `delete`; albo użyj DML `delete(Book).where(...)` |
| `FlushError: Instance <Book at 0x...> has a NULL identity key.` | wstawiasz wiersz z ręcznie ustawionym kluczem `None` do kolumny, która **nie** jest autoinkrementowana, albo do kolumny z kluczem złożonym bez wszystkich części | sprawdź, czy `primary_key=True` i autoinkrementacja są poprawnie skonfigurowane; dla UUID użyj `default=uuid4` |
| `NoResultFound` / `MultipleResultsFound` | `.one()` na wyniku pustym albo z wieloma wierszami | użyj `.one_or_none()`, `.first()` albo popraw warunek zapytania |
| `ResourceClosedError: This result object is closed.` | odczyt z `Result` po jego wyczerpaniu (np. drugie `.all()`) | przypisz wynik do zmiennej: `rows = session.scalars(stmt).all()` |
| `InvalidRequestError: A transaction is already begun on this Session.` | ręczne `session.begin()` na sesji, która już rozpoczęła transakcję (autobegin) | używaj wzorca `with Session(...) as s, s.begin():` na świeżej sesji |
| Objaw: aplikacja „zawiesza się" po kilkudziesięciu żądaniach, `engine.pool.status()` pokazuje `Checked out: 5` | brak `session.close()` — połączenia nie wracają do puli | zawsze `with Session(...)` albo `close()` w `finally`; przy `scoped_session` — `Session.remove()` |
| Objaw: proces zużywa coraz więcej pamięci | sesja żyje zbyt długo, identity map rośnie | jedna sesja na jednostkę pracy; `session.expunge_all()` między partiami |
| Objaw: dane w obiekcie różnią się od danych w bazie | identity map nie nadpisuje już załadowanych atrybutów | `session.refresh(obj)`, `session.expire(obj)` lub `populate_existing=True` |

---

## Słowniczek modułu

| Termin (EN) | Polski | Wyjaśnienie |
|---|---|---|
| `Session` | sesja | Obiekt zarządzający stanem encji: identity map, kolejka zmian, zakres transakcji. Nie jest połączeniem z bazą. |
| `sessionmaker` | fabryka sesji | Konfigurowalny „producent" nowych sesji. Tworzysz go raz na aplikację. |
| `bind` | wiązanie | Wskazanie, do czego sesja ma wysyłać SQL: `Engine` albo `Connection`. |
| `scoped_session` | sesja o zakresie | Opakowanie trzymające jedną sesję na wątek (albo inny zdefiniowany zakres). Wymaga `remove()`. |
| identity map | rejestr tożsamości | Słownik `(klasa, klucz) → obiekt`, gwarantujący unikalność obiektu w obrębie sesji. |
| unit of work | jednostka pracy | Mechanizm zbierania zmian i wysyłania ich w optymalnej kolejności w jednej transakcji. |
| `flush()` | zrzut / wypchnięcie | Wysłanie oczekujących INSERT/UPDATE/DELETE do bazy. Nie kończy transakcji. |
| `commit()` | zatwierdzenie | Flush + `COMMIT` + wygaśnięcie obiektów. Kończy transakcję. |
| `rollback()` | wycofanie | Cofnięcie wszystkich zmian transakcji; obiekty `pending` wracają do stanu `transient`. |
| autobegin | automatyczny start | Sesja sama rozpoczyna transakcję przy pierwszej operacji wymagającej bazy. |
| `autoflush` | automatyczny zrzut | Flush wykonywany przed zapytaniem, żeby zapytanie widziało niezapisane zmiany. |
| `expire` | wygaśnięcie | Wyrzucenie wartości atrybutów z pamięci obiektu; następny dostęp wywoła SELECT. |
| `expire_on_commit` | wygaszanie po commicie | Ustawienie (domyślnie `True`) mówiące, czy po `commit()` wygasić wszystkie obiekty. |
| `refresh()` | odświeżenie | Wymuszenie SELECT-a i nadpisanie atrybutów obiektu danymi z bazy. |
| `expunge()` | odłączenie | Ręczne usunięcie obiektu z sesji; obiekt staje się detached. |
| `merge()` | scalenie | Skopiowanie stanu obiektu zewnętrznego do obiektu zarządzanego przez sesję. Zwraca inny obiekt. |
| transient | ulotny | Stan obiektu: istnieje tylko w Pythonie, sesja o nim nie wie. |
| pending | oczekujący | Stan obiektu: dodany do sesji, brak wiersza w bazie. |
| persistent | trwały | Stan obiektu: ma wiersz w bazie i jest śledzony przez sesję. |
| deleted | usuwany | Stan obiektu: oznaczony do usunięcia, wiersz zniknie przy fluszu. |
| detached | odłączony | Stan obiektu: istnieje w Pythonie, ale sesja nim nie zarządza. |
| identity key | klucz tożsamości | Para `(klasa, klucz główny)`, po której sesja rozpoznaje obiekt. |
| savepoint | punkt zapisu | `session.begin_nested()` — podtransakcja, którą można cofnąć bez przewijania całości. |
| `DetachedInstanceError` | błąd odłączonej instancji | Wyjątek przy próbie dociągnięcia danych obiektu, który nie należy do sesji. |

---

## Dalsze czytanie

- **Session Basics** — podstawy: cykl życia, `begin`, `commit`, `close`, `expire_on_commit`: <https://docs.sqlalchemy.org/en/20/orm/session_basics.html>
- **State Management** — stany obiektu, identity map, `merge`, `refresh`, `expunge`: <https://docs.sqlalchemy.org/en/20/orm/session_state_management.html>
- **Transactions and Connection Management** — autobegin, savepointy, `join_transaction_mode`: <https://docs.sqlalchemy.org/en/20/orm/session_transaction.html>
- **Session API** — pełna dokumentacja parametrów `Session` i `sessionmaker`: <https://docs.sqlalchemy.org/en/20/orm/session_api.html>
- **Contextual/Thread-local Sessions** — `scoped_session` i jego pułapki: <https://docs.sqlalchemy.org/en/20/orm/contextual.html>
- **Session Events** — `before_flush`, `after_flush`, `before_commit` (rozwinięcie w module 13): <https://docs.sqlalchemy.org/en/20/orm/events.html#session-events>
- **Glossary** — definicje pojęć: <https://docs.sqlalchemy.org/en/20/glossary.html>
- **What's New in SQLAlchemy 2.1** — zmiany, w tym autoflush: <https://docs.sqlalchemy.org/en/21/changelog/whatsnew_21.html>

## Co dalej

Wiesz już, jak sesja zarządza pojedynczymi obiektami: kiedy wysyła SQL, jak śledzi zmiany i dlaczego obiekty przechodzą przez pięć stanów. Ale dotąd nasze encje były **samotne** — książka nie wiedziała nic o autorze, a wypożyczenie o czytelniku.

W module 09 zajmiemy się relacjami: `relationship()`, kardynalnościami (1:1, 1:N, N:M), kaskadami i tabelami asocjacyjnymi. Zobaczysz też, jak relacje zmieniają plan flushu i dlaczego dodanie obiektu do kolekcji może wysłać SQL wcześniej, niż się spodziewasz.

➡️ Przejdź do [`09_relacje.md`](09_relacje.md).

<!-- koniec modułu 08 -->