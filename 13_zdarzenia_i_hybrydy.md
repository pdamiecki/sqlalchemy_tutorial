# Moduł 13 — Zdarzenia i hybrydy

Modele, które napisałeś w poprzednich modułach, są tylko deklaracją: mówią, *jakie* dane istnieją, ale nie *co się dzieje*, gdy dane się zmieniają. Ten moduł pokazuje, jak dopisać do nich zachowanie, którego SQLAlchemy nie ma wbudowanego — automatyczne znaczniki czasu, dziennik zmian („kto i kiedy zmienił”), walidację na poziomie atrybutu oraz atrybuty, które działają identycznie w Pythonie i w SQL. Dowiesz się, **w którym dokładnie momencie** pracy z bazą uruchamia się Twój kod (bo od tego zależy, czy zadziała), jak nie doprowadzić do podwójnego wykonywania logiki i dlaczego część mechanizmów przestaje działać przy operacjach masowych.

**Poziom:** 🟠 zaawansowany
**Czas:** ~180 min czytania + ~120 min ćwiczeń
**Wymagania wstępne:**
- [`07_modele_deklaratywne.md`](07_modele_deklaratywne.md) — `DeclarativeBase`, `Mapped`, `mapped_column`, mixiny
- [`08_sesja_cykl_zycia.md`](08_sesja_cykl_zycia.md) — sesja, `flush`, `commit`, stany obiektu
- [`09_relacje.md`](09_relacje.md) — `relationship`, `cascade`, tabele asocjacyjne
- [`11_ladowanie_i_n_plus_1.md`](11_ladowanie_i_n_plus_1.md) — strategie ładowania (przyda się w 13.8)
- [`12_typy_i_wlasne_typy.md`](12_typy_i_wlasne_typy.md) — strefy czasowe, `Numeric`, własne typy

**Czego dotyczy plik:** mechanizmów rozszerzania modeli i sesji: `event.listen` / `@listens_for` (poziomy Engine, Connection, Session, Mapper), zdarzeń `before_flush`/`after_flush`/`before_commit`, atrybutów audytowych, `@validates`, `@hybrid_property`, `column_property`, `association_proxy` oraz mikro-wzorców `synonym`, `composite`, `attribute_mapped_collection`, `ordering_list`.

---

## Spis treści

- [13.1. Po co rozszerzać modele](#131-po-co-rozszerzać-modele)
- [13.2. Model zdarzeń SQLAlchemy](#132-model-zdarzeń-sqlalchemy)
- [13.3. Zdarzenia sesji — serce tego modułu](#133-zdarzenia-sesji--serce-tego-modułu)
- [13.4. Zdarzenia mappera i atrybutów](#134-zdarzenia-mappera-i-atrybutów)
- [13.5. `@validates` — walidacja na poziomie atrybutu](#135-validates--walidacja-na-poziomie-atrybutu)
- [13.6. `@hybrid_property` — jedna logika, dwa światy](#136-hybrid_property--jedna-logika-dwa-światy)
- [13.7. `column_property()` — kolumna liczona w bazie](#137-column_property--kolumna-liczona-w-bazie)
- [13.8. `association_proxy` — wygodny dostęp przez tabelę pośrednią](#138-association_proxy--wygodny-dostęp-przez-tabelę-pośrednią)
- [13.9. Mikro-wzorce stanu](#139-mikro-wzorce-stanu)
- [13.10. Gdzie umieścić logikę — tabela decyzyjna](#1310-gdzie-umieścić-logikę--tabela-decyzyjna)
- [13.11. Debugowanie zdarzeń](#1311-debugowanie-zdarzeń)
- [13.12. Pełny przykład do uruchomienia](#1312-pełny-przykład-do-uruchomienia)
- [Podsumowanie](#podsumowanie)
- [Ćwiczenia](#ćwiczenia)
- [Najczęstsze błędy i jak je czytać](#najczęstsze-błędy-i-jak-je-czytać)
- [Słowniczek modułu](#słowniczek-modułu)
- [Dalsze czytanie](#dalsze-czytanie)
- [Co dalej](#co-dalej)

---

## 13.1. Po co rozszerzać modele

Zacznijmy od problemu, a nie od API.

Wyobraź sobie, że masz już działający kod z modułu 09: klasę `Book`, `Member` i `Loan`. Wszystko się zapisuje, wszystko się odczytuje. Przychodzi jednak szef i mówi:

> „Chcę wiedzieć, kto zmienił tytuł książki i kiedy. I chcę, żeby każdy rekord miał datę ostatniej modyfikacji. I żeby nie dało się zapisać użytkownika z bezsensownym e-mailem. I żeby dało się jednym zapytaniem SQL znaleźć wszystkie wypożyczenia po terminie, ale żeby ten sam warunek dało się policzyć też w Pythonie dla obiektu, który nie jest zapisany.”

Cztery wymagania. Żadne z nich nie jest „SQL-owe” — to **zachowanie aplikacji przy okazji operacji na danych**.

Masz trzy możliwe drogi:

1. **Dopisać logikę w każdym miejscu, w którym modyfikujesz dane.** `book.title = "X"; book.updated_at = now(); session.add(AuditLog(...))`. Działa — dopóki ktoś (Ty za pół roku) nie zapomni. Wtedy w bazie są dziury w audycie, których nikt nie zauważy, dopóki nie będzie potrzebny audyt.
2. **Przenieść logikę do bazy** (triggery, `CHECK`, `DEFAULT`). Działa zawsze, ale jest niewidoczna z poziomu Pythona, trudna w testach i słabo przenośna między dialektami.
3. **Podłączyć się do cyklu życia SQLAlchemy** — wpiąć własny kod w momenty, o których biblioteka i tak wie: „obiekt zaraz zostanie zmieniony”, „za chwilę poleci COMMIT”. To jest właśnie temat tego modułu.

> 💡 **Analogia** — Zdarzenia w SQLAlchemy to jak czujniki w inteligentnym domu. Nie przebudowujesz instalacji elektrycznej ani nie chodzisz za każdym domownikiem z notesem. Montujesz czujnik ruchu, który sam reaguje na zdarzenie „ktoś wszedł”. Ty piszesz tylko to, *co ma się stać*, a nie *kiedy to sprawdzić*. I tak jak w domu: czujniki trzeba zamontować w odpowiednim miejscu (na drzwiach, nie na suficie w piwnicy) — montaż na złym poziomie zdarzeń to najczęstszy błąd początkujących.

> 🧠 **Dlaczego tak jest** — SQLAlchemy jest biblioteką *zdarzeniową* (event-driven) w środku. Cały jej cykl pracy — od nawiązania połączenia, przez budowanie SQL-a, po zwolnienie transakcji — składa się z nazwanych punktów, w których można wywołać własną funkcję. To nie jest dodatek „dla zaawansowanych”: część rzeczy w ekosystemie (np. `AuditMixin`, soft delete, automatyczne znaczniki czasu) buduje się wyłącznie w ten sposób.

---

## 13.2. Model zdarzeń SQLAlchemy

Zanim zaczniemy pisać, musimy ustalić jedno: **zdarzenia mają poziomy**. Poziom to miejsce w architekturze biblioteki, do którego się podłączasz. Jeśli podłączysz się na złym poziomie, Twój kod albo się nie uruchomi, albo uruchomi się za rzadko, albo za często.

```text
                        ┌─────────────────────────────────────────────┐
   POZIOMY ZDARZEŃ      │            TWOJA APLIKACJA                  │
                        └─────────────────────────────────────────────┘
                                        │
                 ┌──────────────────────┴───────────────────────┐
                 │                                              │
        ┌────────▼─────────┐                          ┌─────────▼──────────┐
        │   ORM (Mapper)   │                          │       Core         │
        │                  │                          │                    │
        │  • MapperEvents  │  przed_insert/update/    │  • EngineEvents    │
        │    (per encja)   │  delete, after_*         │    (connect,       │
        │  • AttributeEvents│                         │     before_cursor_ │
        │    (per kolumna) │                          │     execute, ...)  │
        │  • SessionEvents │                          │  • ConnectionEvents│
        │    (globalne)    │                          │    (begin, commit,  │
        │                  │                          │     rollback)      │
        └────────┬─────────┘                          └─────────┬──────────┘
                 │                                              │
                 └──────────────────────┬───────────────────────┘
                                        │
                             ┌──────────▼──────────┐
                             │   DBAPI (sqlite3,   │
                             │   psycopg, asyncpg) │
                             └──────────┬──────────┘
                                        │
                             ┌──────────▼──────────┐
                             │   BAZA DANYCH       │
                             └─────────────────────┘
```

Cztery poziomy, które nas interesują:

| Poziom | Obiekt docelowy (`target`) | Kiedy się uruchamia | Typowe zastosowanie |
|---|---|---|---|
| **Engine / Connection** | `Engine`, `Connection` | Na fizycznym połączeniu i przy każdej komendzie SQL | Logowanie SQL-a, `PRAGMA` dla SQLite, licznik zapytań |
| **Session** | klasa `Session` | Globalnie — dla **każdej** sesji w procesie | Audyt, znaczniki czasu, walidacje krzyżowe, cache |
| **Mapper** | Twoja klasa modelu (`Book`) | Tylko dla tej jednej encji | Wypełnianie pól technicznych, modyfikacja wartości przed zapisem |
| **Attribute** | `Book.title` (atrybut) | Gdy ktoś czyta/pisuje konkretne pole | Normalizacja wartości, walidacja pojedynczego pola |

### 13.2.1. Trzy sposoby rejestracji

Wszystkie trzy robią dokładnie to samo — różnią się wygodą:

```python
# examples/13_events_registration.py
from sqlalchemy import event
from sqlalchemy.orm import Session

# --- Wariant 1: dekorator (najczęstszy) -----------------------------------
@event.listens_for(Session, "before_flush")
def on_before_flush(session: Session, flush_context, instances) -> None:
    """Dekorator sam rejestruje funkcję i zwraca ją z powrotem."""
    print("Zaraz poleci flush")


# --- Wariant 2: jawne wywołanie (gdy funkcja już istnieje) ----------------
def log_commit(session: Session) -> None:
    print("Commit wykonany")


event.listen(Session, "after_commit", log_commit)

# --- Wariant 3: lista kilku zdarzeń naraz ---------------------------------
def on_rollback(session: Session) -> None:
    print("Rollback!")


for identifier in ("after_rollback", "after_soft_rollback"):
    event.listen(Session, identifier, on_rollback)
```

> 💡 **Analogia** — `@event.listens_for` to jak „subskrypcja newslettera” w miejscu jego tworzenia, a `event.listen` to jak dodanie odbiorcy do istniejącej listy mailingowej. Efekt ten sam, ale w wariancie 1 deklarujesz subskrypcję w chwili, gdy definiujesz funkcję.

Dwie funkcje pomocnicze, o których trzeba wiedzieć, bo ratują życie:

```python
from sqlalchemy import event
from sqlalchemy.orm import Session

# Czy ten listener jest już zarejestrowany? (chroni przed dublowaniem)
if event.contains(Session, "before_flush", on_before_flush):
    print("Już zarejestrowany — nie dodaję drugi raz")

# Usunięcie listenera (przydatne w testach i w kodzie jednorazowym)
event.remove(Session, "after_commit", log_commit)
```

### 13.2.2. Parametr `propagate`

Domyślnie zdarzenie zarejestrowane na klasie `Book` **nie** dotyczy podklas `Book`. Jeśli chcesz, żeby dotyczyło:

```python
# examples/13_propagate.py
from sqlalchemy import event
from sqlalchemy.orm import Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class Audiobook(Book):  # dziedziczy po Book
    ...


@event.listens_for(Book, "before_insert", propagate=True)
def stamp_ebook(mapper, connection, target) -> None:
    """propagate=True => działa też dla Audiobook i każdej przyszłej podklasy."""
    target.format = target.format or "unknown"
```

> ⚠️ **Pułapka** — `propagate=True` działa tylko w dół hierarchii i **tylko** dla zdarzeń mappera. Nie „propaguje” się w górę ani do innych encji. Jeśli masz 12 modeli i chcesz ten sam efekt w każdym, albo rejestrujesz zdarzenie w pętli po klasach (jak w przykładzie z 13.4), albo używasz zdarzenia sesji (jak w 13.3). Zdarzenia sesji są globalne z definicji i to jest ich główna zaleta.

> 🧪 **Ćwiczenie** — W pliku z jednym modelem `Book` zarejestruj `before_insert` na `Book`, a następnie utwórz podklasę `Audiobook` i sprawdź, czy zdarzenie się uruchamia bez `propagate=True`. Potem dodaj `propagate=True` i powtórz.

---

## 13.3. Zdarzenia sesji — serce tego modułu

To zdecydowanie najważniejsza sekcja modułu. 80% realnych zastosowań zdarzeń w SQLAlchemy to zdarzenia sesji.

### 13.3.1. Kolejność: kto, kiedy, po kim

Zanim napiszesz jedną linię kodu, musisz znać tę oś czasu. Wypisanie `session.commit()` to nie jedna operacja, a sekwencja:

```text
session.commit()
   │
   ├─(1) session.flush()                    <- jeśli są zmiany
   │    │
   │    ├─(2)  before_flush(session, flush_context, instances)
   │    │        ├─ tu WOLNO dodawać/zmieniać/usunąć obiekty; zmiany wejdą do TEGO flusa
   │    │        └─ tu działa session.new / session.dirty / session.deleted
   │    │
   │    ├─(3)  before_insert / before_update / before_delete   (MapperEvents, per obiekt)
   │    │
   │    ├─(4)  INSERT / UPDATE / DELETE  ──► SQL leci do bazy
   │    │
   │    ├─(5)  after_insert / after_update / after_delete      (MapperEvents, per obiekt)
   │    │
   │    ├─(6)  after_flush(session, flush_context)
   │    │        └─ NIE zmieniaj tu obiektów — flush jest w toku, zmiany zostaną zignorowane
   │    │
   │    └─(7)  after_flush_postexec(session, flush_context)
   │             └─ stan sesji jest już spójny po flushu; tu wolno modyfikować
   │
   ├─(8)  before_commit(session)
   │
   ├─(9)  COMMIT  ──► SQL leci do bazy (koniec transakcji)
   │
   ├─(10) after_commit(session)
   │
   └─     (sesja zaczyna NOWĄ, pustą transakcję — autobegin)
```

Osobno, poza commitem:

```text
   session.rollback()  ──►  after_rollback(session)
   session.add(obj)    ──►  after_attach(session, instance)
   obiekt persistentny → usunięty  ──►  persistent_to_deleted(session, instance)
```

> 🔬 **Pod maską** — Ta kolejność nie jest konwencją, którą można zmienić. Wynika z tego, że `flush` buduje **plan** operacji, a potem go wykonuje. `before_flush` jest wywoływane **przed** zbudowaniem planu, dlatego zmiany tam wykonane trafiają do tego samego `INSERT`/`UPDATE`. `after_flush` jest wywoływane **po** wykonaniu SQL-a, więc zmiany tam wykonane nie mają już gdzie trafić.

### 13.3.2. Pełny przykład: tabela `audit_log`

Zbudujmy dziennik zmian. Chcemy, żeby każda zmiana pola w encji objętej audytem zostawiała wiersz w osobnej tabeli — w tej samej transakcji, żeby audyt nie mógł „udowodnić” zmiany, która została wycofana.

```python
# examples/13_audit_log.py
from __future__ import annotations

from datetime import datetime, timezone

from sqlalchemy import DateTime, ForeignKey, String, Text, func, inspect
from sqlalchemy.orm import DeclarativeBase, Mapped, Session, mapped_column


class Base(DeclarativeBase):
    pass


class AuditLog(Base):
    """Jeden wiersz = jedna zmiana jednego pola."""

    __tablename__ = "audit_log"

    id: Mapped[int] = mapped_column(primary_key=True)
    entity: Mapped[str] = mapped_column(String(50))          # np. "Book"
    entity_id: Mapped[int | None]
    field: Mapped[str] = mapped_column(String(50))           # np. "title"
    old_value: Mapped[str | None] = mapped_column(Text)
    new_value: Mapped[str | None] = mapped_column(Text)
    changed_at: Mapped[datetime] = mapped_column(DateTime, server_default=func.now())
    changed_by: Mapped[str] = mapped_column(String(50), default="anonymous")

    def __repr__(self) -> str:
        return (
            f"<AuditLog {self.entity}#{self.entity_id} {self.field}: "
            f"{self.old_value!r} -> {self.new_value!r} by {self.changed_by}>"
        )


# Encje, których zmiany chcemy śledzić. Świadoma, jawna lista — nie „wszystko”.
AUDITED_ENTITIES = {"Book", "Member"}

# Pola techniczne, które nas nie interesują z punktu widzenia biznesu.
AUDIT_IGNORED_FIELDS = {"created_at", "updated_at", "id"}
```

Teraz listener. Zwróć uwagę na trzy rzeczy: filtrowanie po `session.dirty`, sprawdzanie realnej zmiany oraz odczyt historii wartości.

```python
# examples/13_audit_listener.py
from sqlalchemy import event, inspect
from sqlalchemy.orm import Session


@event.listens_for(Session, "before_flush")
def record_audit_entries(session: Session, flush_context, instances) -> None:
    """Zapisuje do audit_log każdą realną zmianę pola w encjach z AUDITED_ENTITIES."""
    actor = session.info.get("audit_actor", "anonymous")

    # session.dirty = obiekty, które SQLAlchemy uznaje za potencjalnie zmienione.
    # "Potencjalnie" jest tu kluczowe — dlatego sprawdzamy is_modified().
    for obj in session.dirty:
        if type(obj).__name__ not in AUDITED_ENTITIES:
            continue
        if not session.is_modified(obj, include_collections=False):
            continue

        mapper = type(obj).__mapper__
        state = inspect(obj)

        for attr in state.attrs:
            if attr.key in AUDIT_IGNORED_FIELDS:
                continue
            if attr.key not in mapper.columns:
                continue  # relacje pomijamy — interesują nas kolumny
            if mapper.columns[attr.key].primary_key:
                continue

            history = attr.load_history()      # obiekt History (patrz niżej)
            if not history.has_changes():
                continue

            old_value = history.deleted[0] if history.deleted else None
            new_value = history.added[0] if history.added else None
            if old_value == new_value:
                continue

            session.add(
                AuditLog(
                    entity=mapper.class_.__name__,
                    entity_id=getattr(obj, "id", None),
                    field=attr.key,
                    old_value=None if old_value is None else str(old_value),
                    new_value=None if new_value is None else str(new_value),
                    changed_by=actor,
                )
            )
```

### 13.3.3. `session.dirty`, `session.new`, `session.deleted`

Te trzy kolekcje to skróty do stanu sesji:

| Kolekcja | Co zawiera | Uwaga |
|---|---|---|
| `session.new` | Obiekty dodane przez `add()`, nigdy nie zapisane | Wszystkie zostaną wstawione (`INSERT`) |
| `session.dirty` | Obiekty **potencjalnie** zmienione | Może zawierać obiekty bez realnej zmiany! |
| `session.deleted` | Obiekty oznaczone do usunięcia | Zostaną usunięte dopiero w `flush` |

> 🧠 **Dlaczego tak jest** — SQLAlchemy nie może tanio sprawdzić „czy wartość naprawdę się zmieniła”, bo sprawdzenie wymaga porównania z wartością w bazie (dodatkowe `SELECT`) albo porównania z wartością sprzed zmiany (dodatkowy koszt pamięciowy). Zamiast tego stosuje optymistyczne założenie: *„skoro ktoś dotknął atrybutu, być może coś zmienił”*. Dlatego wchodzi do `session.dirty`. Dopiero przy budowaniu planu flusa porównuje historię i **pomija obiekty bez netto-zmian**. Wniosek praktyczny: `session.dirty` to lista podejrzanych, a nie lista winnych.

### 13.3.4. `session.is_modified()` i obiekt `History`

`is_modified(obj)` to dokładniejsze zapytanie do sesji: „czy ten obiekt ma realne zmiany?”.

```python
# examples/13_is_modified.py
from sqlalchemy.orm import Session

with Session(engine) as session:
    book = session.get(Book, 1)

    book.title = "Nowy tytuł"            # zmiana realna
    assert session.is_modified(book) is True

    book.title = book.title              # przypisanie tej samej wartości
    # Nadal trafiamy do session.dirty, ale...
    print(session.is_modified(book))     # False — brak zmian netto

    # include_collections=False (domyślnie) pomija relacje kolekcyjne.
    # Z include_collections=True dodanie elementu do listy też zostanie wykryte.
    print(session.is_modified(book, include_collections=True))
```

Drugi filar to historia atrybutu — obiekt `History`. Zwraca go `attributes.get_history()` albo `load_history()` z `AttributeState`:

```python
# examples/13_history.py
from sqlalchemy.orm.attributes import get_history
from sqlalchemy.orm.base import PASSIVE_NO_INITIALIZE

with Session(engine) as session:
    book = session.get(Book, 1)

    book.title = "Zbrodnia i kara (wyd. 2)"
    h = get_history(book, "title")

    print(h)                 # History(added=['Zbrodnia i kara (wyd. 2)'], unchanged=[], deleted=['Zbrodnia i kara'])
    print(h.has_changes())   # True
    print(h.deleted[0])      # stara wartość z bazy
    print(h.added[0])        # nowa wartość

    # Historia istnieje nawet gdy atrybut nie jest zmieniony:
    print(get_history(book, "author"))   # History(added=[], unchanged=['Fiodor Dostojewski'], deleted=[])

    # PASSIVE_NO_INITIALIZE: nie dociągaj wartości z bazy, jeśli nie jest w pamięci.
    # Szybciej, ale dostaniesz pustą historię dla niezaładowanych atrybutów.
    print(get_history(book, "author", passive=PASSIVE_NO_INITIALIZE))
```

| Pole / metoda `History` | Znaczenie |
|---|---|
| `.added` | Wartości, które „przyszły” (przypisane, dodane do kolekcji) |
| `.unchanged` | Wartości, które były i zostały |
| `.deleted` | Wartości, które „odeszły” (nadpisane, usunięte z kolekcji) |
| `.has_changes()` | Czy cokolwiek się zmieniło |
| `.is_unchanged()` | Czy nic się nie zmieniło |
| `.sum()` | Scalona wartość końcowa (przydatne przy kolekcjach) |

> 🔬 **Pod maską** — `get_history()` i `load_history()` domyślnie używają trybu `PASSIVE_OFF`, co oznacza: *„jeśli wartość nie jest załadowana, dociągnij ją z bazy”*. To znaczy, że Twój listener audytowy może wygenerować dodatkowe `SELECT`-y, jeśli audytowane atrybuty nie zostały wcześniej odczytane. To nie jest błąd — to cena za znajomość prawdziwej starej wartości. Jeśli chcesz ją wyeliminować, ładuj obiekty świadomie (`selectinload`, `load_only`) albo oznacz kolumnę jako `active_history=True` przy deklaracji, co sprawia, że SQLAlchemy pamięta starą wartość już przy przypisaniu i nie musi nic doczytywać.

### 13.3.5. Pozostałe zdarzenia sesji w praktyce

| Zdarzenie | Sygnatura | Do czego realnie służy |
|---|---|---|
| `before_flush` | `(session, flush_context, instances)` | Audyt, znaczniki czasu, walidacje krzyżowe, „dopisz brakujące” |
| `after_flush` | `(session, flush_context)` | Zdarzenia wyprowadzone, ale bez modyfikowania obiektów |
| `after_flush_postexec` | `(session, flush_context)` | Modyfikacje po zakończonym flushu (np. przeliczenie cache) |
| `before_commit` | `(session)` | Ostatnia szansa przed `COMMIT`; tu nadal wolno modyfikować obiekty (wywoła to dodatkowy flush) |
| `after_commit` | `(session)` | Unieważnienie cache, wysłanie zdarzenia do kolejki, log biznesowy |
| `after_rollback` | `(session)` | Sprzątanie po nieudanej transakcji, alerty |
| `after_attach` | `(session, instance)` | Reakcja na `session.add()` — np. wstrzyknięcie aktora do nowego obiektu |
| `persistent_to_deleted` | `(session, instance)` | Reakcja na usunięcie — np. dopisanie do audytu jako „deleted” |
| `do_orm_execute` | `(orm_execute_state)` | Globalna modyfikacja *wszystkich* zapytań ORM (soft delete, multi-tenant) |

`after_commit` bywa myląco nazwane: jest wywoływane już po tym, jak baza potwierdziła commit, i w tym momencie sesja ma otwartą nową, pustą transakcję.

```python
# examples/13_after_commit.py
@event.listens_for(Session, "after_commit")
def publish_changes(session: Session) -> None:
    """Po udanym commicie wysyłamy zdarzenia do kolejki (moduł 21: wzorzec outbox)."""
    outbox = session.info.get("outbox")
    if outbox is not None:
        for message in outbox:
            print(f"-> do kolejki: {message}")
        outbox.clear()
```

> 🆕 **SQLAlchemy 2.1** — W wersji 2.1 mechanizm `autoflush` w sesji działa **bezwarunkowo**: wcześniejsze przypadki, w których flush nie był wywoływany automatycznie zależnie od stanu sesji, zostały ujednolicone. Nie zmieniaj z tego powodu kodu, ale jeśli natrafisz na stary artykuł opisujący „autoflush nie zadziałał, bo…”, wiedz, że dotyczy starszego zachowania. `Session(autoflush=False)` nadal istnieje i nadal bywa przydatne (np. w testach).

> 🆕 **SQLAlchemy 2.1** — `after_bulk_update` i `after_bulk_delete` są w 2.0 oznaczone jako przestarzałe i docelowo znikną. Zamiast nich używaj `do_orm_execute` albo zakładaj, że operacje masowe nie generują zdarzeń per-obiekt (patrz 13.5 — to dotyczy też `@validates`).

---

## 13.4. Zdarzenia mappera i atrybutów

Zdarzenia sesji są globalne. Zdarzenia mappera są **per encja** — to ich wada (trzeba je zarejestrować dla każdej klasy) i zaleta (są precyzyjne).

```python
# examples/13_mapper_events.py
from sqlalchemy import event
from sqlalchemy.orm import Mapped, mapped_column


class Book(Base):
    __tablename__ = "book"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    created_at: Mapped[datetime | None] = mapped_column(DateTime, default=None)
    updated_at: Mapped[datetime | None] = mapped_column(DateTime, default=None)


def stamp_created(mapper, connection, target) -> None:
    """before_insert: obiekt istnieje w pamięci, ale nie ma jeszcze wiersza w bazie."""
    now = utcnow_naive()
    target.created_at = now
    target.updated_at = now


def stamp_updated(mapper, connection, target) -> None:
    """before_update: dokładnie przed zbudowaniem instrukcji UPDATE."""
    target.updated_at = utcnow_naive()


event.listen(Book, "before_insert", stamp_created)
event.listen(Book, "before_update", stamp_updated)
```

Sygnatura jest zawsze taka sama: `(mapper, connection, target)`. `mapper` to obiekt mapujący, `connection` — połączenie używane przez flush, a `target` — konkretna instancja, którą właśnie zapisujemy.

> 🔬 **Pod maską** — Ponieważ `before_insert` i `before_update` uruchamiają się **przed** zbudowaniem instrukcji SQL, każda zmiana atrybutu wykonana w środku trafi do tego samego `INSERT`/`UPDATE`. Widać to w logu: pojawia się jeden `UPDATE book SET title=?, updated_at=? WHERE book.id = ?` — a nie dwa osobne.

| Zdarzenie mappera | Kiedy | Typowy użytek |
|---|---|---|
| `before_insert` | Przed `INSERT` | `created_at`, generowanie slugów, domyślne wartości biznesowe |
| `before_update` | Przed `UPDATE` | `updated_at`, normalizacja, wersjonowanie ręczne |
| `before_delete` | Przed `DELETE` | Wsoft-delete, archiwizacja |
| `after_insert` / `after_update` / `after_delete` | Po wykonaniu SQL-a | Zdarzenia wyprowadzone (bez modyfikacji obiektu) |
| `instrument_class` | Przy tworzeniu klasy | Zaawansowane: dodawanie własnych deskryptorów |

### 13.4.1. „Realna zmiana” w praktyce

Kluczowa obserwacja z 13.3.3: **SQLAlchemy nie wyśle `UPDATE`, jeśli obiekt nie ma netto-zmian.** To znaczy, że jeśli użytkownik otworzy formularz, nic nie zmieni i kliknie „Zapisz”, `before_update` **nie zostanie wywołane** — bo nie ma czego aktualizować. To jest dokładnie to zachowanie, którego oczekujemy od `updated_at`: znacznik ma się zmieniać, gdy coś się zmieniło.

```python
# examples/13_no_op_update.py
with Session(engine) as session:
    book = session.get(Book, 1)
    original = book.updated_at

    book.title = book.title          # „zmiana” bez zmiany
    session.commit()

    # W logu NIE ma żadnego UPDATE — flush pominął obiekt bez netto-zmian.
    print(book.updated_at == original)   # True
```

> ⚠️ **Pułapka** — Wartość `updated_at` ustawiona w `before_update` **nie** zostanie zapisana, jeśli w tym samym flushu obiekt był tylko „dotknięty”, ale bez zmian. I odwrotnie: jeśli ustawisz `updated_at` ręcznie w kodzie aplikacji, *sam ten fakt* stanie się zmianą i wymusi `UPDATE`. Dlatego nie ustawiaj `updated_at` „na wszelki wypadek” w kodzie serwisowym — zostaw to zdarzeniu.

### 13.4.2. Zdarzenia atrybutów

Najniższy poziom: podsłuchiwanie konkretnego pola. Zdarzenie `set` uruchamia się, gdy ktoś przypisuje wartość.

```python
# examples/13_attribute_events.py
from sqlalchemy import event


@event.listens_for(Member.email, "set", retval=True, active_history=True)
def normalize_email(target, value, oldvalue, initiator):
    """set: uruchamia się przy każdym przypisaniu do Member.email.

    retval=True    -> zwracana wartość zastępuje przypisywaną
    active_history -> oldvalue zawiera prawdziwą starą wartość (kosztem SELECT-a)
    """
    if value is None:
        return value
    normalized = value.strip().lower()
    if "@" not in normalized:
        raise ValueError(f"Niepoprawny e-mail: {normalized!r}")
    return normalized
```

Dla kolekcji są dwa oddzielne zdarzenia: `append` i `remove`.

```python
# examples/13_collection_events.py
@event.listens_for(Book.tag_links, "append", retval=True)
def normalize_link(target, value, initiator):
    """value to obiekt BookTag dopisywany do kolekcji."""
    value.added_at = value.added_at or utcnow_naive()
    return value


@event.listens_for(Book.tag_links, "remove")
def log_removal(target, value, initiator) -> None:
    print(f"Odpięto tag {value.tag_id} od książki {target.id}")
```

> ⚠️ **Pułapka** — Zdarzenia kolekcji domyślnie działają w trybie `raw=True`, czyli dostajesz **niemodyfikowany** obiekt pośredniczący, nie końcową wartość. Jeśli chcesz pracować na obiekcie domenowym (np. na `Tag`, a nie na `BookTag`), użyj `raw=False`. Bez tego `value.tag` może być `None` i dostaniesz `AttributeError`, którego nie zrozumiesz.

> 🧠 **Dlaczego tak jest** — Zdarzenia atrybutów są niskopoziomowe i uruchamiają się zanim SQLAlchemy ustali stan obiektu. Działają także poza sesją (na obiekcie zupełnie odłączonym od bazy), dlatego nie mają dostępu do kontekstu sesji i nie mogą jej użyć do dociągnięcia danych. To jest ta sama granica, o której piszemy w 13.5 przy `@validates`.

---

## 13.5. `@validates` — walidacja na poziomie atrybutu

`@validates` to dekorator, który robi rzecz pozornie podobną do zdarzenia `set`, ale jest wygodniejszy i o wiele częściej używany.

```python
# examples/13_validates.py
from sqlalchemy.orm import Mapped, mapped_column, validates


class Member(Base):
    __tablename__ = "member"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    email: Mapped[str] = mapped_column(String(255), unique=True)

    @validates("email")
    def validate_email(self, key: str, value: str) -> str:
        """Zwrócona wartość ZASTĘPUJE przypisywaną — tu normalizujemy e-mail."""
        normalized = value.strip().lower()
        if "@" not in normalized or "." not in normalized.split("@")[-1]:
            raise ValueError(f"Niepoprawny adres e-mail: {normalized!r}")
        return normalized

    @validates("name", "email")   # jedna metoda dla wielu pól
    def reject_blank(self, key: str, value: str) -> str:
        if not value.strip():
            raise ValueError(f"Pole {key!r} nie może być puste")
        return value
```

Co się dzieje przy przypisaniu:

```python
member.email = "  Anna.Kowalska@Example.COM "
print(member.email)     # anna.kowalska@example.com

try:
    member.email = "to-nie-jest-email"
except ValueError as exc:
    print("Odrzucono:", exc)
print(member.email)     # anna.kowalska@example.com — stara wartość, atrybut nietknięty
```

> 🧠 **Dlaczego tak jest** — `@validates` uruchamia się w momencie **przypisania wartości do atrybutu**, a nie w momencie zapisu. To ma trzy konsekwencje, które trzeba znać na pamięć:

**Konsekwencja 1: nie ma kontekstu sesji.** Wewnątrz `@validates` nie wiesz, czy obiekt jest w sesji, czy nie, i nie możesz zrobić `select()`. Jeśli walidacja wymaga zapytania do bazy („czy ten ISBN już istnieje?”), `@validates` **nie jest** właściwym miejscem — to zadanie warstwy serwisowej (moduł 19) albo ograniczenia `UNIQUE` w bazie.

**Konsekwencja 2: nie działa przy operacjach masowych.** To najważniejsza pułapka tego modułu:

```python
# examples/13_validates_bulk.py
from sqlalchemy import update

# To NIE uruchomi @validates — walidacja jest metodą Pythona na instancji,
# a tutaj nie tworzymy żadnej instancji. SQL leci prosto do bazy.
session.execute(
    update(Member).where(Member.id == 1).values(email="  BŁĘDNY  ")
)
session.commit()
# W bazie wyląduje "  BŁĘDNY  " — bez normalizacji i bez walidacji.
```

**Konsekwencja 3: nie działa przy `bulk_insert_mappings` / `insert().values([...])`.** Ta sama przyczyna: brak instancji.

Wniosek: `@validates` jest walidacją *obiektową*, nie *bazodanową*. Jeśli chcesz gwarancji na poziomie danych, musisz mieć ograniczenie w bazie.

### 13.5.1. Trzy poziomy walidacji — porównanie

| Poziom | Narzędzie | Kiedy się uruchamia | Chroni przed | Nie chroni przed |
|---|---|---|---|---|
| **Python / obiekt** | `@validates` | Przy przypisaniu do atrybutu instancji | Literówki, złe formaty w kodzie aplikacji | `update()`, importem z CSV, ręcznym SQL-em, inną aplikacją |
| **Python / API** | Pydantic v2 (moduł 22) | Na wejściu do aplikacji (HTTP, CLI) | Złymi danymi z zewnątrz | Kodem wewnętrznym, migracjami, innymi klientami bazy |
| **Baza danych** | `CHECK`, `NOT NULL`, `UNIQUE`, `FK`, trigger | Przy każdym `INSERT`/`UPDATE`, niezależnie od źródła | Wszystkim — to jedyna prawdziwa gwarancja | Złożoną logiką biznesową wymagającą kontekstu |

Zalecany układ warstw: **Pydantic** daje czytelne komunikaty dla użytkownika, **`@validates`** pilnuje spójności obiektów w kodzie wewnętrznym, **ograniczenia w bazie** są ostatnią linią obrony. Trzy warstwy brzmi jak nadmiar, ale każda odpowiada na inne pytanie: „czy powiedzieć użytkownikowi *co* zrobił źle?”, „czy pozwolić temu obiektowi wejść do sesji?”, „czy dopuścić ten wiersz do tabeli?”.

> ⚠️ **Pułapka** — `@validates` **nie uruchamia się dla atrybutów ustawianych w `before_insert`/`before_update`**? Uruchamia się. Ale nie uruchamia się dla wartości pochodzących z `default=`/`server_default=` — te nadaje baza (albo SQLAlchemy poza walidacją), więc nie przechodzą przez Twoją metodę. Nie polegaj na `@validates` do sprawdzania wartości generowanych przez bazę.

```python
# examples/13_check_constraint.py
from sqlalchemy import CheckConstraint


class Loan(Base):
    __tablename__ = "loan"
    __table_args__ = (
        CheckConstraint("due_date >= borrowed_at", name="ck_loan_due_after_borrow"),
    )
    # ...
```

Taki `CHECK` zadziała niezależnie od tego, czy wiersz wstawiono przez ORM, przez `update()`, czy przez `psql`. W SQLite trzeba pamiętać o `PRAGMA foreign_keys=ON` (klucze obce), ale `CHECK` działa bez tego.

> 🧪 **Ćwiczenie** — Dodaj do `Loan` `CheckConstraint` wymagający, by `returned_at` był `NULL` albo późniejszy niż `borrowed_at`. Następnie spróbuj zapisać błędny obiekt przez `session.execute(update(...))` i sprawdź, że baza go odrzuci, mimo że `@validates` (gdyby istniało) nie zadziałałoby.

---

## 13.6. `@hybrid_property` — jedna logika, dwa światy

To najbardziej niedoceniany mechanizm SQLAlchemy. Rozwiążmy najpierw problem, z którym spotkał się każdy, kto pisał aplikacje z ORM-em.

Chcesz wiedzieć, czy wypożyczenie jest po terminie. Piszesz naturalnie:

```python
class Loan:
    @property
    def is_overdue(self) -> bool:
        if self.returned_at is not None:
            return False
        return self.due_date < date.today()
```

Działa dla jednego obiektu w pamięci. Ale gdy chcesz znaleźć **wszystkie** przeterminowane wypożyczenia, musisz albo:

- pobrać wszystkie wiersze i filtrować w Pythonie (katastrofa wydajnościowa, moduł 17),
- albo napisać drugą, równoległą wersję tej samej logiki w SQL i **utrzymywać dwie wersje w zgodzie na zawsze**.

To drugie jest źródłem klasycznych błędów: w Pythonie `due_date < date.today()`, w SQL `due_date < CURRENT_DATE` — i pewnego dnia ktoś zmienia jedną stronę bez drugiej. Wyniki zaczynają się rozjeżdżać, a nikt nie wie dlaczego.

`hybrid_property` daje jedno miejsce, w którym definiujesz **dwie implementacje tej samej reguły** — i sprawia, że Python oraz SQL używają właściwej z nich automatycznie.

```python
# examples/13_hybrid.py
from datetime import date

from sqlalchemy import ColumnElement, and_, cast, case, func, Integer
from sqlalchemy.ext.hybrid import hybrid_property


class Loan(Base):
    __tablename__ = "loan"

    id: Mapped[int] = mapped_column(primary_key=True)
    due_date: Mapped[date]
    returned_at: Mapped[date | None] = mapped_column(default=None)

    # ---------- wersja dla Pythona ----------
    @hybrid_property
    def is_overdue(self) -> bool:
        """Działa na instancji: loan.is_overdue."""
        if self.returned_at is not None:
            return False
        return self.due_date < date.today()

    # ---------- wersja dla SQL ----------
    @is_overdue.inplace.expression
    @classmethod
    def _is_overdue_expression(cls) -> ColumnElement[bool]:
        """Działa na klasie: select(Loan).where(Loan.is_overdue)."""
        return and_(cls.returned_at.is_(None), cls.due_date < func.current_date())
```

Teraz oba światy używają **tej samej nazwy**:

```python
# Python — pojedynczy obiekt
loan = session.get(Loan, 1)
if loan.is_overdue:
    print("Kara!")

# SQL — cała tabela, filtr po stronie bazy
overdue_loans = session.scalars(select(Loan).where(Loan.is_overdue)).all()
```

> 🔬 **Pod maską** — `select(Loan).where(Loan.is_overdue)` wygeneruje:

```sql
SELECT loan.id, loan.book_id, loan.member_id, loan.borrowed_at,
       loan.due_date, loan.returned_at
FROM loan
WHERE loan.returned_at IS NULL AND loan.due_date < CURRENT_DATE
```

Zwróć uwagę: SQLAlchemy **nie** wysłało całej tabeli do Pythona. `CURRENT_DATE` to wyrażenie SQL-owe (funkcja bazy, nie wartość z Pythona) i dlatego filtr działa w bazie.

### 13.6.1. Wariant `inplace` — po co `@classmethod` z podkreślnikiem

Można to zapisać na dwa sposoby:

```python
# Wariant A (używany wyżej): inplace.expression
@hybrid_property
def is_overdue(self) -> bool: ...

@is_overdue.inplace.expression          # <-- nie nadpisuje właściwości!
@classmethod
def _is_overdue_expression(cls) -> ColumnElement[bool]: ...

# Wariant B: klasyczny
class Loan(Base):
    @hybrid_property
    def is_overdue(self) -> bool: ...

    @is_overdue.expression              # <-- najpierw definiuje property, potem dokłada expression
    def is_overdue(cls) -> ColumnElement[bool]: ...
```

W wariancie A drugi element jest dekorowany „w miejscu” i **musi** mieć inną nazwę (stąd `_is_overdue_expression`) — nazwa istnieje tylko po to, żeby mieć co udekorować, i nie jest używana poza klasą. Wariant `inplace` jest czytelniejszy, bo obie definicje stoją obok siebie pod tą samą nazwą logiczną.

### 13.6.2. Przykład drugi: `remaining_days` i pułapka semantyki

Spróbujmy teraz czegoś, co wymaga **arytmetyki na datach** — i tu zaczynają się prawdziwe problemy, które chcę, żebyś zobaczył świadomie:

```python
    @hybrid_property
    def remaining_days(self) -> int:
        if self.returned_at is not None:
            return 0
        return (self.due_date - date.today()).days

    @remaining_days.inplace.expression
    @classmethod
    def _remaining_days_expression(cls) -> ColumnElement[int]:
        # SQLite: julianday() zwraca różnicę dni jako FLOAT — rzutujemy na int.
        # PostgreSQL ma to prościej: `cls.due_date - func.current_date()`.
        return cast(
            case(
                (cls.returned_at.is_not(None), 0),
                else_=func.julianday(cls.due_date) - func.julianday(func.current_date()),
            ),
            Integer,
        )
```

> ⚠️ **Pułapka** — Zwróć uwagę na dwa problemy w tej krótkiej metodzie:
> 1. **W Pythonie mamy `int`, w SQL `julianday()` zwraca `float`.** Bez `cast(...)` zapytanie `select(...).where(Loan.remaining_days > 3)` działałoby „prawie” dobrze — dopóki nie trafisz na wartość graniczną. To klasyczny przykład *niezgodności semantyki* między światami.
> 2. **`date.today()` w Pythonie to czas lokalny, `CURRENT_DATE` w SQLite to czas UTC.** W większości przypadków różnica jest niewidoczna, ale jeśli Twój serwer stoi w UTC, a aplikacja działa w Europe/Warsaw, to przez kilka godzin na dobę „Python mówi `True`, baza mówi `False`”. Ujednolicaj strefę czasową i **testuj zapytania w WHERE**, nie tylko obiekt w pamięci.

Praktyczna rada: `hybrid_property` z `expression` **testuj w obu trybach**. Test, który sprawdza tylko `loan.is_overdue`, nie wykryje, że `where(Loan.is_overdue)` zwraca coś innego.

### 13.6.3. `hybrid_method` i własne komparatory

Gdy logika przyjmuje argument, użyj `hybrid_method`:

```python
# examples/13_hybrid_method.py
from sqlalchemy.ext.hybrid import hybrid_method


class Book(Base):
    @hybrid_method
    def is_borrowed_on(self, day: date) -> bool:
        return any(loan.borrowed_at == day for loan in self.loans)

    @is_borrowed_on.inplace.expression
    @classmethod
    def _is_borrowed_on_expression(cls, day: date) -> ColumnElement[bool]:
        return Loan.book_id == cls.id and Loan.borrowed_at == day
```

`hybrid_method` jest rzadziej potrzebny, ale bywa ratunkiem przy filtrach z parametrem (np. „czy dostępne w podanym dniu”).

---

## 13.7. `column_property()` — kolumna liczona w bazie

Czasem potrzebujesz atrybutu, który **nie istnieje jako kolumna**, ale wynika z danych: liczba komentarzy pod postem, liczba wypożyczeń membera, suma pozycji zamówienia.

Pierwsza opcja — policz w Pythonie — skaluje się źle (moduł 11: N+1). Druga: policz w SQL i **udawaj, że to kolumna**.

```python
# examples/13_column_property.py
from sqlalchemy import ForeignKey, func, select
from sqlalchemy.orm import DeclarativeBase, Mapped, Session, column_property, mapped_column, relationship


class Base(DeclarativeBase):
    pass


# Comment definiujemy PIERWSZY, bo Post odwoła się do niego w wyrażeniu SQL.
class Comment(Base):
    __tablename__ = "comment"

    id: Mapped[int] = mapped_column(primary_key=True)
    post_id: Mapped[int] = mapped_column(ForeignKey("post.id"))
    body: Mapped[str]

    post: Mapped["Post"] = relationship(back_populates="comments")


class Post(Base):
    __tablename__ = "post"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str]

    # Skorelowane podzapytanie: policz komentarze dla TEGO posta.
    # W ciele klasy `id` odnosi się do kolumny zadeklarowanej linijkę wyżej.
    comment_count = column_property(
        select(func.count(Comment.id))
        .where(Comment.post_id == id)
        .correlate_except(Comment)
        .scalar_subquery()
    )

    comments: Mapped[list[Comment]] = relationship(back_populates="post")
```

> 🔬 **Pod maską** — Każde zapytanie o `Post` dociąga policzoną kolumnę:

```sql
SELECT post.id,
       post.title,
       (SELECT count(comment.id) AS count_1
        FROM comment
        WHERE comment.post_id = post.id) AS comment_count
FROM post
```

Użycie jest całkowicie naturalne:

```python
post = session.scalars(select(Post).order_by(Post.comment_count.desc())).first()
print(post.title, post.comment_count)
```

### 13.7.1. Kiedy `column_property`, a kiedy zwykłe zapytanie

| Kryterium | `column_property` | `select(func.count(...))` w zapytaniu |
|---|---|---|
| Wygoda użycia | Wysoka — wygląda jak kolumna | Wymaga jawnego zapytania i mapowania wyniku |
| Koszt | Podzapytanie **w każdym** zapytaniu o tę encję | Płacisz tylko wtedy, gdy naprawdę liczysz |
| Duże tabele | Ryzykowne — `SELECT` po `id` też dociąga podzapytanie | Bezpieczne |
| Filtrowanie | Można filtrować i sortować po tym atrybucie | Też można, ale trzeba powtórzyć wyrażenie |

Ratunek na koszt: `deferred=True`, który sprawia, że policzona kolumna jest dociągana **leniwie**, tylko gdy naprawdę po nią sięgniesz:

```python
    comment_count = column_property(
        select(func.count(Comment.id))
        .where(Comment.post_id == id)
        .correlate_except(Comment)
        .scalar_subquery(),
        deferred=True,     # domyślnie NIE ładuj
    )
```

> ⚠️ **Pułapka** — `column_property(deferred=True)` nie da się użyć w `where()`. Jeśli chcesz filtrować po liczbie komentarzy (`having`, `where`), musisz albo zrezygnować z `deferred`, albo policzyć w zapytaniu jawnie (`group_by` + `having`). Nie da się mieć jednocześnie „nie licz, dopóki nie poproszę” i „filtruj po tym w bazie” — filtr *jest* prośbą o policzenie.

> ⚠️ **Pułapka** — Jeśli użyjesz w `column_property` kolumny zdefiniowanej w tym samym ciele klasy, musisz odwołać się do obiektu kolumny (jak `id` wyżej), a nie do `Post.id`. W bardziej skomplikowanych przypadkach (np. gdy potrzebujesz odwołania do samej klasy) bezpieczniej jest dodać właściwość po zdefiniowaniu obu klas, przez `Mapper.add_property()`, albo użyć `@declared_attr`. Jeśli dostajesz `NameError` przy imporcie modeli, niemal na pewno problem jest tutaj.

---

## 13.8. `association_proxy` — wygodny dostęp przez tabelę pośrednią

Modelowanie N:M z tabelą asocjacyjną, która ma własne kolumny, wymusza używanie obiektu pośredniego. Robi się to tak (moduł 09):

```python
class Book(Base):
    tag_links: Mapped[list[BookTag]] = relationship(back_populates="book", cascade="all, delete-orphan")

class BookTag(Base):
    book_id: Mapped[int] = mapped_column(ForeignKey("book.id"), primary_key=True)
    tag_id: Mapped[int] = mapped_column(ForeignKey("tag.id"), primary_key=True)
    added_at: Mapped[datetime | None] = mapped_column(DateTime, server_default=func.now())
    tag: Mapped[Tag] = relationship(back_populates="book_links")
```

Problem: żeby dodać tag, trzeba myśleć o tabeli pośredniej, choć z punktu widzenia domeny to szczegół:

```python
# Niezręczne — domena przecieka przez tabelę asocjacyjną
book.tag_links.append(BookTag(tag=some_tag))
print([link.tag.name for link in book.tag_links])
```

`association_proxy` daje „przezroczyste okno” na drugi koniec relacji:

```python
# examples/13_association_proxy.py
from sqlalchemy.ext.associationproxy import AssociationProxy, association_proxy


class Book(Base):
    __tablename__ = "book"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))

    tag_links: Mapped[list[BookTag]] = relationship(
        back_populates="book",
        cascade="all, delete-orphan",
    )

    # Proxy: udaje zwykłą relację N:M, choć pod spodem siedzi BookTag.
    tags: AssociationProxy[list[Tag]] = association_proxy(
        "tag_links",                       # po jakiej relacji iść
        "tag",                             # do jakiego atrybutu po drugiej stronie
        creator=lambda tag: BookTag(tag=tag),   # jak zbudować obiekt pośredni z wartości
    )
```

Teraz kod domenowy wygląda tak, jakby tabela pośrednia nie istniała:

```python
book.tags.append(klasyka)              # -> tworzy BookTag(tag=klasyka)
print([tag.name for tag in book.tags]) # -> ['klasyka', 'rosyjska']

book.tags.remove(klasyka)              # -> usuwa obiekt pośredni z kolekcji
print([tag.name for tag in book.tags]) # -> ['rosyjska']
```

> 🔬 **Pod maską** — `book.tags.append(tag)` wygeneruje `INSERT` do tabeli pośredniej:

```sql
INSERT INTO book_tag (book_id, tag_id, added_at) VALUES (?, ?, ?)
```

Zwróć uwagę, że `added_at` pochodzi z `server_default` — proxy niczego nie „ukrywa”, po prostu buduje obiekt `BookTag` za Ciebie.

### 13.8.1. Ograniczenia proxy

| Problem | Wyjaśnienie |
|---|---|
| Proxy nie jest relacją | Nie da się go użyć w `joinedload(Book.tags)`. Ładujesz relację bazową: `selectinload(Book.tag_links).selectinload(BookTag.tag)`. |
| Zapytania po proxy | Nie napiszesz `select(Book).where(Book.tags.any(...))` tak swobodnie jak po relacji — musisz zejść do `tag_links` i `BookTag.tag`. |
| `creator` jest obowiązkowy przy zapisie | Bez `creator` proxy jest tylko do odczytu (próba `append` rzuci `AttributeError`). |
| Deduplikacja | `creator=lambda tag: BookTag(tag=tag)` tworzy nowy obiekt pośredni dla każdego `append`. Dodanie tego samego taga dwa razy da naruszenie klucza głównego — pilnuj tego w kodzie. |
| Nie działa przy `write_only` | Proxy wymaga wczytanej kolekcji bazowej. |

> 🆕 **SQLAlchemy 2.1** — Dla relacji wiele-do-wielu ładowanych przez obiekty pośrednie pojawiły się nowe możliwości: `selectinload(..., omit_join=True)` pozwala pominąć dodatkowe dołączenie tabeli asocjacyjnej, a parametr `chunksize` dzieli wielkie `IN (...)` na porcje. W 2.0 działa to samo zapytanie, ale bez tych optymalizacji — jeśli masz tabelę pośrednią z setkami tysięcy wierszy i wąskie gardło w `IN`, to jest konkretny powód, żeby zajrzeć do 2.1.

> 🧪 **Ćwiczenie** — Dodaj do proxy `creator`, który przy tworzeniu pośredniego wiersza ustawia `added_at` na bieżącą datę oraz powtarzalnie odrzuca próbę dodania duplikatu (podnieś `ValueError` zamiast pozwolić bazie rzucić `IntegrityError`).

---

## 13.9. Mikro-wzorce stanu

Cztery mniejsze mechanizmy. Każdy rozwiązuje konkretny, wąski problem — i każdy bywa nadużywany.

### 13.9.1. `synonym`

Drugie imię dla tego samego atrybutu. Używaj, gdy zmieniasz nazwę w kodzie, ale nie chcesz migrować kolumny:

```python
# examples/13_synonym.py
from sqlalchemy.orm import synonym


class Book(Base):
    __tablename__ = "book"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))   # kolumna w bazie: title="...""

    # W kodzie aplikacji mówimy `name`, w bazie dalej `title`.
    name = synonym("title")
```

```python
book.name = "Nowy tytuł"
print(book.title)      # Nowy tytuł
print(session.is_modified(book))   # True — to TEN SAM atrybut, nie kopia
```

`synonym` to nie kopia — to alias. Zmiana przez jedną nazwę jest widoczna przez drugą.

### 13.9.2. `composite()` — obiekt wartości z kilku kolumn

Jeśli trzy kolumny opisują jeden koncept (kwota + waluta, ulica + miasto + kod), `composite` scala je w jeden obiekt Pythona:

```python
# examples/13_composite.py
from decimal import Decimal

from sqlalchemy import Numeric, String
from sqlalchemy.orm import composite


class Money:
    """Obiekt wartości: niemutowalny w założeniu, porównywalny."""

    def __init__(self, amount: Decimal, currency: str) -> None:
        self.amount = amount
        self.currency = currency

    def __composite_values__(self) -> tuple[Decimal, str]:
        return (self.amount, self.currency)

    def __eq__(self, other: object) -> bool:
        return (
            isinstance(other, Money)
            and other.amount == self.amount
            and other.currency == self.currency
        )

    def __repr__(self) -> str:
        return f"Money({self.amount} {self.currency})"


class Product(Base):
    __tablename__ = "product"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    price_amount: Mapped[Decimal] = mapped_column(Numeric(10, 2))
    price_currency: Mapped[str] = mapped_column(String(3))

    price = composite(Money, "price_amount", "price_currency")
```

```python
product.price = Money(Decimal("39.99"), "PLN")
print(product.price_amount, product.price_currency)   # 39.99 PLN
print(product.price == Money(Decimal("39.99"), "PLN"))  # True
```

> ⚠️ **Pułapka** — `composite` bez `__eq__` sprawi, że SQLAlchemy nie wykryje zmiany, gdy podstawisz nowy obiekt o tej samej zawartości — a *wykryje* zmianę, gdy podstawisz obiekt porównywalnie inny. Zawsze implementuj `__eq__` i `__composite_values__`. I pamiętaj, że `composite` nie działa z operacjami masowymi (`update().values(price=...)`) — tam operujesz na kolumnach.

### 13.9.3. `attribute_mapped_collection` — słownik zamiast listy

Gdy w kolekcji chcesz wyszukiwać po kluczu (np. po slug), zamień listę na słownik:

```python
# examples/13_mapped_collection.py
from sqlalchemy.orm import attribute_mapped_collection


class Article(Base):
    __tablename__ = "article"
    id: Mapped[int] = mapped_column(primary_key=True)
    sections: Mapped[dict[str, "Section"]] = relationship(
        collection_class=attribute_mapped_collection("slug"),
        cascade="all, delete-orphan",
    )


class Section(Base):
    __tablename__ = "section"
    id: Mapped[int] = mapped_column(primary_key=True)
    article_id: Mapped[int] = mapped_column(ForeignKey("article.id"))
    slug: Mapped[str] = mapped_column(String(50))
    body: Mapped[str]
```

```python
article.sections["wstep"] = Section(slug="wstep", body="...")
print(article.sections.keys())    # dict_keys(['wstep'])
```

### 13.9.4. `ordering_list` — lista z automatyczną pozycją

Gdy kolejność elementów ma znaczenie i musi przetrwać w bazie:

```python
# examples/13_ordering_list.py
from sqlalchemy.ext.orderinglist import ordering_list


class Checklist(Base):
    __tablename__ = "checklist"
    id: Mapped[int] = mapped_column(primary_key=True)
    items: Mapped[list["ChecklistItem"]] = relationship(
        order_by="ChecklistItem.position",
        collection_class=ordering_list("position"),
        cascade="all, delete-orphan",
    )


class ChecklistItem(Base):
    __tablename__ = "checklist_item"
    id: Mapped[int] = mapped_column(primary_key=True)
    checklist_id: Mapped[int] = mapped_column(ForeignKey("checklist.id"))
    position: Mapped[int]
    text: Mapped[str]
```

```python
checklist.items.insert(0, ChecklistItem(text="Pierwszy krok"))
# Wszystkie kolejne elementy dostają zaktualizowaną kolumnę `position`.
```

> ⚠️ **Pułapka** — `ordering_list` utrzymuje pozycje w pamięci i przy zmianie kolejności emituje **wiele** `UPDATE`-ów (po jednym na przesunięty element). Na liście 500 elementów wstawienie jednego na początek to 500 `UPDATE`-ów. Jeśli potrzebujesz tego na dużą skalę, przemyśl model danych (np. „rzadkie” pozycje 10, 20, 30 i przepakowywanie co jakiś czas).

### 13.9.5. Kiedy mikro-wzorce są nadmiarowe

| Sytuacja | Werdykt |
|---|---|
| `synonym` na dwóch polach „bo tak wygodniej” | Prawie zawsze niepotrzebne — wymyśl jedną dobrą nazwę |
| `composite` dla pary (imię, nazwisko) | Zwykle nadmiar; `full_name` w `hybrid_property` wystarczy |
| `composite` dla (kwota, waluta) | Uzasadnione — to naprawdę jedna wartość |
| `attribute_mapped_collection` dla 3 elementów | Nadmiar; lista wystarczy |
| `ordering_list` dla uporządkowania po `created_at` | Nadmiar; użyj `order_by` |

---

## 13.10. Gdzie umieścić logikę — tabela decyzyjna

To najważniejsza tabela w module. Większość problemów z architekturą aplikacji na SQLAlchemy bierze się z umieszczenia logiki w złym miejscu.

| Miejsce | Przykład logiki | Zalety | Wady | Kiedy wybrać |
|---|---|---|---|---|
| `default=` / `server_default=` | `created_at`, licznik | Zero kodu, działa w bazie | Brak dostępu do kontekstu Pythona | Wartości domyślne |
| `@validates` | Format e-maila, zakres liczbowy | Blisko modelu, czytelne błędy | Nie działa przy operacjach masowych, brak sesji | Walidacja pojedynczego pola |
| Zdarzenie atrybutu (`set`) | Normalizacja, transformacja | Działa też poza sesją | Niskopoziomowe, brak kontekstu | Gdy `@validates` nie wystarcza (np. `oldvalue`) |
| Zdarzenie mappera | `created_at`, `updated_at`, slug | Per encja, modyfikuje dane przed SQL-em | Trzeba rejestrować dla każdej klasy | Pola techniczne encji |
| Zdarzenie sesji | Audyt, znaczniki czasu dla wielu encji | Globalne, pełny kontekst sesji | Wpływa na **wszystkie** sesje w procesie | Reguły przekrojowe |
| `hybrid_property` | „Czy po terminie”, „pełna nazwa” | Jedna logika w Pythonie i SQL | Trzeba utrzymać zgodność semantyki | Atrybuty pochodne bez dostępu do bazy |
| `column_property` | Liczba komentarzy | Wygodne, jak kolumna | Koszt w każdym zapytaniu | Małe tabele, często potrzebne wartości |
| **Warstwa serwisowa** ([moduł 19](19_warstwy_i_data_mapper.md)) | „Nie wolno wypożyczyć zajętej książki” | Pełny kontekst, testowalne, jawne | Więcej kodu, trzeba o niej pamiętać | Logika biznesowa z regułami |
| **Repozytorium** ([moduł 20](20_repository.md)) | „Wypożyczenia po terminie” | Jedno miejsce na zapytanie | Więcej warstw | Zapytania wielokrotnego użytku |
| **Baza** (`CHECK`, trigger) | `due_date >= borrowed_at` | Gwarancja niezależna od aplikacji | Niewidoczne z Pythona, trudne w testach | Integralność danych |

Reguła kciuka, którą warto zapamiętać:

> **Jeśli logika dotyczy jednego pola jednego obiektu — `@validates` lub zdarzenie atrybutu. Jeśli dotyczy wszystkich encji i wszystkich sesji — zdarzenie sesji. Jeśli wymaga decyzji biznesowej lub wiedzy o innych obiektach — warstwa serwisowa. Jeśli jest gwarancją integralności — baza.**

---

## 13.11. Debugowanie zdarzeń

Zdarzenia są potężne, ale trudne w diagnozie, bo „dzieją się same”. Oto lista problemów w kolejności od najczęstszego.

### 13.11.1. Listenery rejestrowane dwa razy

Najczęstsza przyczyna „dlaczego mam dwa wpisy w audycie?”. Moduł z listenerami zostaje zaimportowany dwa razy (np. raz w CLI, raz w testach) i `@event.listens_for` wykona się dwukrotnie. Python nie wie, że to „ta sama” funkcja.

```python
# examples/13_avoid_double_registration.py
from sqlalchemy import event
from sqlalchemy.orm import Session


def register_listeners() -> None:
    """Jedno, jawnie wywoływane miejsce rejestracji."""
    if not event.contains(Session, "before_flush", record_audit_entries):
        event.listen(Session, "before_flush", record_audit_entries)
```

Wzorzec: **nie rejestruj listenerów przy imporcie modułu z modelami.** Zrób funkcję `register_listeners()` i wywołaj ją raz w punkcie wejścia aplikacji (`main.py`, `app = FastAPI(lifespan=...)`).

> ⚠️ **Pułapka** — `event.listen(Session, ...)` jest **globalne dla procesu**. Zarejestrowany w module listener będzie działał także w testach, w skryptach migracyjnych i w narzędziach administracyjnych. Jeśli nie chcesz audytu w środowisku testowym, w teardownie testów wywołaj `event.remove(...)`.

### 13.11.2. Rekurencja: event, który modyfikuje to, co obserwuje

```python
# examples/13_recursion_bug.py   <- ANTYPRZYKŁAD, nie kopiuj
@event.listens_for(Session, "before_flush")
def touch_everything(session: Session, flush_context, instances) -> None:
    for obj in session.dirty:
        obj.updated_at = utcnow_naive()      # to też jest zmiana...
        # ...a obiekt już jest w session.dirty, więc kolejny listener
        # albo kolejny flush widzi go znowu.
```

W praktyce SQLAlchemy uruchomi `before_flush` raz na flush, więc nie ma nieskończonej pętli — ale są dwa realne objawy:

1. **Audyt loguje pola techniczne** (`updated_at` zmienia się przy każdej zmianie). Rozwiązanie: `AUDIT_IGNORED_FIELDS` (patrz 13.3.2).
2. **Wywołanie `session.flush()` wewnątrz `before_flush`** — to prawdziwy błąd. Flush nie może być reentrantny: dostaniesz wyjątek albo stan, którego nie rozumiesz. Nigdy nie wołaj `flush()` ani `commit()` w środku zdarzenia flusa.

### 13.11.3. Kolejność listenerów ma znaczenie

Dwa listenery na `before_flush` uruchamiają się **w kolejności rejestracji**. Jeśli `touch_updated_at` zarejestrujesz przed `record_audit_entries`, audyt zobaczy już zmodyfikowany `updated_at` (i, jeśli go nie wykluczysz, zapisze go jako zmianę). Rejestruj w świadomej kolejności i dokumentuj ją komentarzem.

### 13.11.4. Podczas debugowania: co widzi SQLAlchemy

Kilka narzędzi, które warto znać:

```python
# examples/13_debug_helpers.py
from sqlalchemy import event, inspect
from sqlalchemy.orm import Session


@event.listens_for(Session, "before_flush")
def debug_flush(session: Session, flush_context, instances) -> None:
    print("NEW   :", session.new)
    print("DIRTY  :", session.dirty)
    print("DELETED:", session.deleted)
    for obj in session.dirty:
        state = inspect(obj)
        print(
            f"  {type(obj).__name__}#{state.identity} "
            f"modified={state.modified} "
            f"attributes={sorted(state.unloaded)}"
        )


@event.listens_for(Session, "after_flush")
def debug_after_flush(session: Session, flush_context) -> None:
    print("Po flushu — stan sesji jest spójny, ale nie modyfikuj obiektów")


@event.listens_for(engine, "before_cursor_execute")
def count_queries(conn, cursor, statement, parameters, context, executemany) -> None:
    """Ile zapytań faktycznie poszło? (por. moduł 11 — licznik N+1.)"""
    print(f"SQL ({'executemany' if executemany else 'single'}): {statement.split()[0]}")
```

`inspect(obj)` zwraca `InstanceState` z bardzo przydatnymi polami: `.identity` (klucz główny), `.transient`, `.pending`, `.persistent`, `.detached`, `.deleted`, `.modified`, `.unloaded`, `.session_id`. To najszybszy sposób na odpowiedź „w jakim stanie jest ten obiekt?”.

> 🧠 **Dlaczego tak jest** — Zdarzenia nie mają własnego debuggera ani inspektora, bo nie są „kodem, który się wykonuje w jednym miejscu”. Są rozproszone po cyklu życia sesji. Dlatego jedyną skuteczną strategią jest **logowanie na każdym etapie** — dokładnie to, co robi debugger pokazany wyżej.

> 🧪 **Ćwiczenie** — Włącz `echo=True` na silniku i dodaj listener `before_cursor_execute`. Porównaj liczbę linii „SQL>” z liczbą operacji, które wykonałeś w kodzie. Wyjaśnij każdą nadwyżkę (podpowiedź: autoflush, audyt, dociąganie wartości).

---

## 13.12. Pełny przykład do uruchomienia

Poniższy plik demonstruje wszystkie trzy główne mechanizmy modułu naraz: `@validates` (walidacja), `hybrid_property` (logika w Pythonie i SQL) oraz `association_proxy` (dostęp przez tabelę pośrednią) — a wszystko na tle zdarzeń `before_flush` i `before_insert`/`before_update`.

```python
# examples/13_events_and_hybrids.py
"""Moduł 13 — zdarzenia, walidacja i hybrydy na jednym, spójnym przykładzie.

Uruchomienie:
    python -m venv .venv && source .venv/bin/activate    # Windows: .venv\\Scripts\\activate
    pip install "SQLAlchemy>=2.0"
    python examples/13_events_and_hybrids.py
"""

from __future__ import annotations

from datetime import date, datetime, timedelta, timezone

from sqlalchemy import (
    DateTime,
    ForeignKey,
    Integer,
    String,
    Text,
    and_,
    cast,
    case,
    create_engine,
    event,
    func,
    inspect,
    select,
)
from sqlalchemy.ext.associationproxy import AssociationProxy, association_proxy
from sqlalchemy.ext.hybrid import hybrid_property
from sqlalchemy.orm import (
    DeclarativeBase,
    Mapped,
    Session,
    mapped_column,
    relationship,
    validates,
)
from sqlalchemy.orm.attributes import get_history
from sqlalchemy.sql.elements import ColumnElement

# ---------------------------------------------------------------------------
# 1. Silnik + podsłuchiwanie SQL-a (zdarzenia poziomu Engine)
# ---------------------------------------------------------------------------
engine = create_engine("sqlite:///:memory:")


@event.listens_for(engine, "connect")
def enable_sqlite_foreign_keys(dbapi_connection, connection_record) -> None:
    """SQLite domyślnie NIE egzekwuje kluczy obcych — włączamy je ręcznie."""
    cursor = dbapi_connection.cursor()
    cursor.execute("PRAGMA foreign_keys=ON")
    cursor.close()


@event.listens_for(engine, "before_cursor_execute")
def log_sql(conn, cursor, statement, parameters, context, executemany) -> None:
    """Nasze własne echo — pełna kontrola nad formatem."""
    params = f"  -- {parameters}" if parameters else ""
    print(f"  SQL> {' '.join(statement.split())}{params}")


def utcnow_naive() -> datetime:
    """Naiwny UTC — bezpieczny dla kolumn DateTime bez strefy (SQLite)."""
    return datetime.now(timezone.utc).replace(tzinfo=None)


# ---------------------------------------------------------------------------
# 2. Modele
# ---------------------------------------------------------------------------
class Base(DeclarativeBase):
    pass


class AuditMixin:
    """Pola techniczne dla encji, które chcemy śledzić w czasie."""

    created_at: Mapped[datetime | None] = mapped_column(DateTime, default=None)
    updated_at: Mapped[datetime | None] = mapped_column(DateTime, default=None)


class AuditLog(Base):
    __tablename__ = "audit_log"

    id: Mapped[int] = mapped_column(primary_key=True)
    entity: Mapped[str] = mapped_column(String(50))
    entity_id: Mapped[int | None]
    field: Mapped[str] = mapped_column(String(50))
    old_value: Mapped[str | None] = mapped_column(Text)
    new_value: Mapped[str | None] = mapped_column(Text)
    changed_at: Mapped[datetime] = mapped_column(DateTime, server_default=func.now())
    changed_by: Mapped[str] = mapped_column(String(50), default="anonymous")

    def __repr__(self) -> str:
        return (
            f"<AuditLog {self.entity}#{self.entity_id} {self.field}: "
            f"{self.old_value!r} -> {self.new_value!r} by {self.changed_by}>"
        )


class Tag(Base):
    __tablename__ = "tag"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50), unique=True)

    book_links: Mapped[list[BookTag]] = relationship(back_populates="tag")


class BookTag(Base):
    __tablename__ = "book_tag"

    book_id: Mapped[int] = mapped_column(
        ForeignKey("book.id", ondelete="CASCADE"), primary_key=True
    )
    tag_id: Mapped[int] = mapped_column(
        ForeignKey("tag.id", ondelete="CASCADE"), primary_key=True
    )
    added_at: Mapped[datetime | None] = mapped_column(DateTime, server_default=func.now())

    book: Mapped[Book] = relationship(back_populates="tag_links")
    tag: Mapped[Tag] = relationship(back_populates="book_links")


class Book(AuditMixin, Base):
    __tablename__ = "book"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    author: Mapped[str] = mapped_column(String(100))

    tag_links: Mapped[list[BookTag]] = relationship(
        back_populates="book",
        cascade="all, delete-orphan",
    )
    tags: AssociationProxy[list[Tag]] = association_proxy(
        "tag_links", "tag", creator=lambda tag: BookTag(tag=tag)
    )
    loans: Mapped[list[Loan]] = relationship(back_populates="book")


class Member(AuditMixin, Base):
    __tablename__ = "member"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    email: Mapped[str] = mapped_column(String(255), unique=True)

    loans: Mapped[list[Loan]] = relationship(back_populates="member")

    @validates("email")
    def validate_email(self, key: str, value: str) -> str:
        """Normalizuje i waliduje e-mail przy KAŻDYM przypisaniu."""
        normalized = value.strip().lower()
        if "@" not in normalized or "." not in normalized.split("@")[-1]:
            raise ValueError(f"Niepoprawny adres e-mail: {normalized!r}")
        return normalized


class Loan(AuditMixin, Base):
    __tablename__ = "loan"

    id: Mapped[int] = mapped_column(primary_key=True)
    book_id: Mapped[int] = mapped_column(ForeignKey("book.id"))
    member_id: Mapped[int] = mapped_column(ForeignKey("member.id"))
    borrowed_at: Mapped[date] = mapped_column(default=date.today)
    due_date: Mapped[date]
    returned_at: Mapped[date | None] = mapped_column(default=None)

    book: Mapped[Book] = relationship(back_populates="loans")
    member: Mapped[Member] = relationship(back_populates="loans")

    # ---------------- hybrid_property ----------------
    @hybrid_property
    def is_overdue(self) -> bool:
        """Python: loan.is_overdue."""
        if self.returned_at is not None:
            return False
        return self.due_date < date.today()

    @is_overdue.inplace.expression
    @classmethod
    def _is_overdue_expression(cls) -> ColumnElement[bool]:
        """SQL: select(Loan).where(Loan.is_overdue)."""
        return and_(cls.returned_at.is_(None), cls.due_date < func.current_date())

    @hybrid_property
    def remaining_days(self) -> int:
        """Python: ile dni zostało (0 dla zwróconych)."""
        if self.returned_at is not None:
            return 0
        return (self.due_date - date.today()).days

    @remaining_days.inplace.expression
    @classmethod
    def _remaining_days_expression(cls) -> ColumnElement[int]:
        """SQL: odpowiednik (SQLite). W PostgreSQL: cls.due_date - func.current_date()."""
        return cast(
            case(
                (cls.returned_at.is_not(None), 0),
                else_=func.julianday(cls.due_date) - func.julianday(func.current_date()),
            ),
            Integer,
        )


# ---------------------------------------------------------------------------
# 3. Zdarzenia: znaczniki czasu (poziom mappera) i audyt (poziom sesji)
# ---------------------------------------------------------------------------
def stamp_created(mapper, connection, target) -> None:
    """before_insert: obiekt nie ma jeszcze wiersza w bazie."""
    now = utcnow_naive()
    target.created_at = now
    target.updated_at = now


def stamp_updated(mapper, connection, target) -> None:
    """before_update: tuż przed zbudowaniem instrukcji UPDATE."""
    target.updated_at = utcnow_naive()


def register_timestamp_listeners() -> None:
    """Mapper events NIE dziedziczą się po mixinach — rejestrujemy je w pętli."""
    for cls in (Book, Member, Loan):
        event.listen(cls, "before_insert", stamp_created)
        event.listen(cls, "before_update", stamp_updated)


AUDITED_ENTITIES = {"Book", "Member"}
AUDIT_IGNORED_FIELDS = {"created_at", "updated_at", "id"}


def record_audit_entries(session: Session, flush_context, instances) -> None:
    """before_flush: zapisuje realne zmiany do audit_log w TEJ SAMEJ transakcji."""
    actor = session.info.get("audit_actor", "anonymous")

    for obj in session.dirty:
        if type(obj).__name__ not in AUDITED_ENTITIES:
            continue
        if not session.is_modified(obj, include_collections=False):
            continue

        mapper = type(obj).__mapper__
        for attr in inspect(obj).attrs:
            if attr.key in AUDIT_IGNORED_FIELDS or attr.key not in mapper.columns:
                continue
            if mapper.columns[attr.key].primary_key:
                continue

            history = attr.load_history()
            if not history.has_changes():
                continue

            old_value = history.deleted[0] if history.deleted else None
            new_value = history.added[0] if history.added else None
            if old_value == new_value:
                continue

            session.add(
                AuditLog(
                    entity=mapper.class_.__name__,
                    entity_id=getattr(obj, "id", None),
                    field=attr.key,
                    old_value=None if old_value is None else str(old_value),
                    new_value=None if new_value is None else str(new_value),
                    changed_by=actor,
                )
            )


def register_audit_listener() -> None:
    if not event.contains(Session, "before_flush", record_audit_entries):
        event.listen(Session, "before_flush", record_audit_entries)


# ---------------------------------------------------------------------------
# 4. Demonstracja
# ---------------------------------------------------------------------------
def main() -> None:
    register_timestamp_listeners()
    register_audit_listener()
    Base.metadata.create_all(engine)

    with Session(engine) as session:
        print("\n=== 1. @validates: normalizacja i odrzucenie złego e-maila ===")
        member = Member(name="Anna Kowalska", email="  Anna.Kowalska@Example.COM ")
        session.add(member)
        session.flush()
        print("  Zapisany e-mail:", member.email)

        try:
            member.email = "to-nie-jest-email"
        except ValueError as exc:
            print("  Odrzucono:", exc)
        print("  E-mail po nieudanej próbie:", member.email)

        print("\n=== 2. association_proxy: tagi bez myślenia o tabeli pośredniej ===")
        book = Book(title="Zbrodnia i kara", author="Fiodor Dostojewski")
        book.tags.append(Tag(name="klasyka"))
        book.tags.append(Tag(name="rosyjska"))
        session.add(book)
        session.flush()
        print("  Tagi:", sorted(tag.name for tag in book.tags))

        print("\n=== 3. hybrid_property: jedna logika, dwa światy ===")
        session.add_all(
            [
                Loan(book=book, member=member, due_date=date.today() - timedelta(days=3)),
                Loan(book=book, member=member, due_date=date.today() + timedelta(days=7)),
                Loan(
                    book=book,
                    member=member,
                    due_date=date.today() - timedelta(days=30),
                    returned_at=date.today(),
                ),
            ]
        )
        session.flush()

        python_left = [loan for loan in book.loans if loan.is_overdue]
        sql_left = session.scalars(select(Loan).where(Loan.is_overdue)).all()
        print(f"  Python: {len(python_left)} po terminie")
        print(f"  SQL   : {len(sql_left)} po terminie")
        assert len(python_left) == len(sql_left), "Rozjazd semantyki Python/SQL!"

        print("\n=== 4. Historia atrybutu (obiekt History) ===")
        book.title = "Zbrodnia i kara (wyd. 2)"
        print("  ", get_history(book, "title"))

        print("\n=== 5. Audyt zmian (before_flush) ===")
        session.info["audit_actor"] = "anna.kowalska"
        member.name = "Anna Kowalska-Nowak"
        session.commit()

        for entry in session.scalars(select(AuditLog).order_by(AuditLog.id)):
            print("  ", entry)

        print("\n=== 6. Brak zmiany => brak UPDATE, brak wpisu w audycie ===")
        touched = session.get(Book, book.id)
        before_updated_at = touched.updated_at
        touched.title = touched.title          # przypisanie tej samej wartości
        session.commit()
        print("  updated_at bez zmian:", touched.updated_at == before_updated_at)
        print("  Liczba wpisów w audycie:", session.scalar(select(func.count(AuditLog.id))))


if __name__ == "__main__":
    main()
```

Fragment oczekiwanego wyjścia (skrócony):

```text
=== 1. @validates: normalizacja i odrzucenie złego e-maila ===
  SQL> INSERT INTO member (name, email, created_at, updated_at) VALUES (?, ?, ?, ?)  -- ('Anna Kowalska', 'anna.kowalska@example.com', '...', '...')
  Zapisany e-mail: anna.kowalska@example.com
  Odrzucono: Niepoprawny adres e-mail: 'to-nie-jest-email'
  E-mail po nieudanej próbie: anna.kowalska@example.com

=== 2. association_proxy: tagi bez myślenia o tabeli pośredniej ===
  SQL> INSERT INTO tag (name) VALUES (?)  -- ('klasyka',)
  SQL> INSERT INTO book_tag (book_id, tag_id) VALUES (?, ?)  -- (1, 1)
  Tagi: ['klasyka', 'rosyjska']

=== 3. hybrid_property: jedna logika, dwa światy ===
  SQL> SELECT loan.id, loan.book_id, ... FROM loan WHERE loan.returned_at IS NULL AND loan.due_date < CURRENT_DATE
  Python: 1 po terminie
  SQL   : 1 po terminie

=== 4. Historia atrybutu (obiekt History) ===
   History(added=['Zbrodnia i kara (wyd. 2)'], unchanged=[], deleted=['Zbrodnia i kara'])

=== 5. Audyt zmian (before_flush) ===
  SQL> UPDATE book SET title=?, updated_at=? WHERE book.id = ?
  SQL> UPDATE member SET name=?, updated_at=? WHERE member.id = ?
  SQL> INSERT INTO audit_log (entity, entity_id, field, old_value, new_value, changed_by) VALUES (?, ?, ?, ?, ?, ?)
  <AuditLog Book#1 title: 'Zbrodnia i kara' -> 'Zbrodnia i kara (wyd. 2)' by anna.kowalska>
  <AuditLog Member#1 name: 'Anna Kowalska' -> 'Anna Kowalska-Nowak' by anna.kowalska>

=== 6. Brak zmiany => brak UPDATE, brak wpisu w audycie ===
  updated_at bez zmian: True
  Liczba wpisów w audycie: 2
```

Trzy rzeczy warte zauważenia w powyższym logu:

1. **`@validates` zadziałało przy `flush`**, bo `email` zostało przypisane wcześniej do obiektu. Gdybyśmy zapisali ten sam e-mail przez `session.execute(update(...))`, w bazie wylądowałaby wersja nieznormalizowana.
2. **Hybryda dała te same wyniki w Pythonie i w SQL** — to jest cała idea i dlatego warto dopisać `assert` w kodzie.
3. **W kroku 6 nie poleciał żaden `UPDATE`** i nie powstał żaden wpis w audycie — bo SQLAlchemy porównał historię i uznał, że nie ma zmian netto. To jest ta „realna zmiana”, o której piszemy od 13.3.3.

---

## Podsumowanie

1. **Zdarzenia mają poziomy** — Engine/Connection (SQL, połączenia), Session (globalne, dla wszystkich sesji), Mapper (per encja), Attribute (per pole). Wybór złego poziomu to najczęstsza przyczyna „nie działa” albo „działa podwójnie”.
2. **`before_flush` to najważniejszy punkt zaczepienia.** Uruchamia się przed zbudowaniem planu flusa, więc zmiany tam wykonane trafiają do tego samego `INSERT`/`UPDATE`. `after_flush` jest już za późno na modyfikacje.
3. **`session.dirty` to lista podejrzanych, nie winnych.** Zawsze łącz ją z `session.is_modified(obj, include_collections=False)`, jeśli zależy Ci na realnych zmianach.
4. **`History` (`attributes.get_history`, `attr.load_history()`)** daje dostęp do wartości starej i nowej. Domyślny tryb `PASSIVE_OFF` może dociągnąć wartość z bazy — to kosztuje `SELECT`, ale daje prawdę.
5. **Audyt buduj w jednej transakcji ze zmianą.** Wpis `AuditLog` dodany w `before_flush` wyląduje w tym samym commicie — audyt nie może „udowodnić” zmiany, która została wycofana.
6. **`@validates` jest walidacją obiektową.** Normalizuje i odrzuca wartości przy przypisaniu, ale **nie działa** przy `update()`, `insert().values([...])` i imporcie danych. Gwarancje dawaj w bazie.
7. **`hybrid_property` = jedna reguła, dwie implementacje.** Zawsze używaj `inplace.expression` (albo `.expression`) i **testuj w WHERE**, nie tylko na obiekcie w pamięci.
8. **Pilnuj zgodności semantyki** między Pythonem a SQL: strefy czasowe (`date.today()` vs `CURRENT_DATE`), typy (`int` vs `float` z `julianday`), rozmiary liter (`like` vs `ilike`).
9. **`column_property` jest wygodne, ale kosztuje** — podzapytanie leci przy każdym zapytaniu o encję. Ratunkiem jest `deferred=True`, ale wtedy nie filtrujesz po tym atrybucie.
10. **`association_proxy` ukrywa szczegół, nie relację.** Pod spodem nadal jest tabela pośrednia, jej ograniczenia i jej problemy wydajnościowe.

---

## Ćwiczenia

### Ćwiczenie 1 — `updated_at`, który nie kłamie (poziom łatwy)

Do modelu `Book` z przykładu 13.12 dodaj pole `updated_at` ustawiane wyłącznie wtedy, gdy zmiana jest **realna**. Napisz skrypt, który:

1. Zmienia tytuł — i pokazuje, że `updated_at` się zmienił i `UPDATE` poleciał.
2. Przypisuje ten sam tytuł ponownie — i pokazuje, że `updated_at` **nie** zmienił się i **żaden** `UPDATE` nie poleciał.
3. Zmienia tytuł, a następnie przywraca poprzednią wartość w tej samej transakcji przed `commit()` — i wyjaśnia, jaki wynik powinien być i dlaczego.

**Wskazówka:** punkt 3 jest najciekawszy. SQLAlchemy porównuje historię *przed* flush, a nie sekwencję przypisań.

### Ćwiczenie 2 — `audit_log` dla dwóch encji (poziom średni)

Rozszerz audyt z 13.12 tak, aby:

1. Logował również **utworzenie** obiektu (`session.new`) — z `old_value=None` i `new_value=None` (albo z opisem „created”), ale tylko dla encji z listy `AUDITED_ENTITIES`.
2. Logował również **usunięcie** (zdarzenie `persistent_to_deleted`) — pole `field` ustaw na `"__deleted__"`.
3. Nie logował pól technicznych ani klucza głównego.

Napisz test, który sprawdza, że po utworzeniu, zmianie i usunięciu książki w `audit_log` są dokładnie oczekiwane wpisy.

**Wskazówka:** `persistent_to_deleted` uruchamia się w momencie, gdy obiekt *staje się* usunięty — czyli po `session.delete(obj)`, jeszcze przed flush. Sprawdź, czy `obj.id` jest wtedy dostępny.

### Ćwiczenie 3 — „Czy książka jest dostępna?” (poziom trudny)

Zaimplementuj na `Book` hybrydowe pole `is_available`, które zwraca `True`, jeśli książka nie ma ani jednego aktywnego wypożyczenia (`returned_at IS NULL`).

Wymagania:

1. Wersja Pythonowa nie może wykonywać zapytania do bazy (jeśli relacja `loans` nie jest załadowana, użyj tego, co jest w pamięci).
2. Wersja SQL-owa musi działać jako `select(Book).where(Book.is_available)` i **nie** generować N+1.
3. Dla porządku dodaj `column_property` z liczbą aktywnych wypożyczeń (`active_loans`) i pokaż, jak różni się SQL dla obu podejść.
4. Uzasadnij w komentarzu, dlaczego jedno z tych rozwiązań wybrałbyś w endpointcie API zwracającym listę 50 książek.

---

### Rozwiązania

#### Rozwiązanie 1

```python
# solutions/13_01_updated_at.py
from __future__ import annotations

from datetime import date, datetime, timedelta, timezone

from sqlalchemy import DateTime, event, func, select
from sqlalchemy.orm import DeclarativeBase, Mapped, Session, mapped_column

engine = create_engine("sqlite:///:memory:")


def utcnow_naive() -> datetime:
    return datetime.now(timezone.utc).replace(tzinfo=None)


class Base(DeclarativeBase):
    pass


class Book(Base):
    __tablename__ = "book"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str]
    updated_at: Mapped[datetime | None] = mapped_column(DateTime, default=None)


def touch_updated_at(mapper, connection, target) -> None:
    """before_update uruchamia się TYLKO wtedy, gdy SQLAlchemy realnie zaplanuje UPDATE."""
    target.updated_at = utcnow_naive()


event.listen(Book, "before_update", touch_updated_at)


def main() -> None:
    Base.metadata.create_all(engine)

    with Session(engine) as session:
        book = Book(title="Wiedźmin")
        session.add(book)
        session.commit()

        # --- 1. Realna zmiana -------------------------------------------------
        book.title = "Wiedźmin — Ostatnie życzenie"
        session.commit()
        after_real_change = book.updated_at
        print("1. Po realnej zmianie updated_at:", after_real_change)
        assert after_real_change is not None

        # --- 2. Brak realnej zmiany -------------------------------------------
        book.title = book.title        # przypisanie tej samej wartości
        session.commit()
        print("2. updated_at bez zmian?", book.updated_at == after_real_change)
        assert book.updated_at == after_real_change

        # --- 3. Zmiana i powrót do wartości wyjściowej ------------------------
        book.title = "Cokolwiek"
        book.title = "Wiedźmin — Ostatnie życzenie"   # wracamy do stanu z bazy
        session.commit()
        print("3. updated_at bez zmian?", book.updated_at == after_real_change)
        assert book.updated_at == after_real_change


if __name__ == "__main__":
    main()
```

**Dlaczego punkt 3 tak działa:** obiekt trafia do `session.dirty` przy pierwszym przypisaniu, ale SQLAlchemy porównuje **stan końcowy** z **historią względem wartości w bazie** — i widzi, że są identyczne. Nie buduje więc `UPDATE`, a skoro nie ma `UPDATE`, nie ma `before_update`. To nie jest wyjątek ani ciekawostka: to samo zachowanie chroni Cię przed lawiną pustych `UPDATE`-ów przy każdym zapisie formularza, w którym użytkownik nie zmienił ani jednego pola.

**Alternatywa:** możesz zamiast zdarzenia mappera użyć `mapped_column(DateTime, onupdate=...)`. Wtedy SQLAlchemy wywoła funkcję przy budowaniu `UPDATE`. Różnica praktyczna: `onupdate` działa również przy operacjach aktualizacji przez Core (`session.execute(update(...))`), a zdarzenie mappera już nie — bo Core nie tworzy obiektów. Wybierz `onupdate`, jeśli aktualizujesz też masowo; wybierz zdarzenie, jeśli chcesz mieć dostęp do całego obiektu i logikę warunkową.

**Minipułapka:** nie ustawiaj `updated_at` ręcznie w serwisie „dla pewności”. To natychmiast uczyni każdy zapis realną zmianą i zniweczy cały mechanizm.

#### Rozwiązanie 2

```python
# solutions/13_02_audit_insert_delete.py
from __future__ import annotations

from datetime import datetime, timezone

from sqlalchemy import DateTime, String, Text, event, func, inspect, select
from sqlalchemy.orm import DeclarativeBase, Mapped, Session, mapped_column

AUDITED_ENTITIES = {"Book", "Member"}
AUDIT_IGNORED_FIELDS = {"created_at", "updated_at", "id"}


def utcnow_naive() -> datetime:
    return datetime.now(timezone.utc).replace(tzinfo=None)


class Base(DeclarativeBase):
    pass


class AuditLog(Base):
    __tablename__ = "audit_log"

    id: Mapped[int] = mapped_column(primary_key=True)
    entity: Mapped[str] = mapped_column(String(50))
    entity_id: Mapped[int | None]
    field: Mapped[str] = mapped_column(String(50))
    old_value: Mapped[str | None] = mapped_column(Text)
    new_value: Mapped[str | None] = mapped_column(Text)
    changed_at: Mapped[datetime] = mapped_column(DateTime, default=utcnow_naive)
    changed_by: Mapped[str] = mapped_column(String(50), default="anonymous")

    def __repr__(self) -> str:
        return (
            f"<AuditLog {self.entity}#{self.entity_id} {self.field}: "
            f"{self.old_value!r} -> {self.new_value!r}>"
        )


class Book(Base):
    __tablename__ = "book"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))


class Member(Base):
    __tablename__ = "member"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))


def _log(
    session: Session,
    obj: object,
    field: str,
    old_value: str | None,
    new_value: str | None,
) -> None:
    session.add(
        AuditLog(
            entity=type(obj).__name__,
            entity_id=getattr(obj, "id", None),
            field=field,
            old_value=old_value,
            new_value=new_value,
            changed_by=session.info.get("audit_actor", "anonymous"),
        )
    )


@event.listens_for(Session, "before_flush")
def audit_updates(session: Session, flush_context, instances) -> None:
    """ZMIANY: tylko encje z listy, tylko kolumny, tylko realne różnice."""
    for obj in session.dirty:
        if type(obj).__name__ not in AUDITED_ENTITIES:
            continue
        if not session.is_modified(obj, include_collections=False):
            continue
        mapper = type(obj).__mapper__
        for attr in inspect(obj).attrs:
            if attr.key in AUDIT_IGNORED_FIELDS or attr.key not in mapper.columns:
                continue
            if mapper.columns[attr.key].primary_key:
                continue
            history = attr.load_history()
            if not history.has_changes():
                continue
            old = history.deleted[0] if history.deleted else None
            new = history.added[0] if history.added else None
            if old == new:
                continue
            _log(session, obj, attr.key, None if old is None else str(old),
                 None if new is None else str(new))

    # TWORZENIE: jeden zbiorczy wpis na obiekt (nie na każde pole).
    for obj in session.new:
        if type(obj).__name__ not in AUDITED_ENTITIES:
            continue
        _log(session, obj, "__created__", None, None)


@event.listens_for(Session, "persistent_to_deleted")
def audit_deletions(session: Session, instance) -> None:
    """USUWANIE: obiekt istnieje jeszcze w bazie, więc id() jest dostępne."""
    if type(instance).__name__ not in AUDITED_ENTITIES:
        return
    _log(session, instance, "__deleted__", None, None)
```

**Dlaczego `persistent_to_deleted`, a nie `before_delete`:** oba zadziałają, ale `persistent_to_deleted` uruchamia się natychmiast przy `session.delete(obj)`, jeszcze przed flushem, dzięki czemu `session.info["audit_actor"]` jest z pewnością jeszcze tym samym kontekstem. `before_delete` (mapper event) ma tę przewagę, że dostajesz `connection`, którym możesz wykonać dodatkowy SQL. Wybierz to drugie, jeśli audyt ma wymagać zapisu do innej bazy.

**Minipułapka:** wpis `__created__` w `session.new` powstanie również wtedy, gdy obiekt zostanie ostatecznie wycofany (`rollback`). To jest w porządku — wpis wyląduje w tej samej transakcji i też zostanie wycofany. Nie próbuj „rezerwować” audytu poza transakcją.

#### Rozwiązanie 3

```python
# solutions/13_03_is_available.py
from __future__ import annotations

from datetime import date
from sqlalchemy import ForeignKey, and_, func, select
from sqlalchemy.ext.hybrid import hybrid_property
from sqlalchemy.orm import (
    DeclarativeBase,
    Mapped,
    Session,
    column_property,
    mapped_column,
    relationship,
)
from sqlalchemy.sql.elements import ColumnElement


class Base(DeclarativeBase):
    pass


class Loan(Base):
    __tablename__ = "loan"

    id: Mapped[int] = mapped_column(primary_key=True)
    book_id: Mapped[int] = mapped_column(ForeignKey("book.id"))
    returned_at: Mapped[date | None] = mapped_column(default=None)


class Book(Base):
    __tablename__ = "book"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str]
    loans: Mapped[list[Loan]] = relationship()

    # ---- 1. Wersja Pythonowa: używa pamięci, ZERO zapytań ----------------
    @hybrid_property
    def is_available(self) -> bool:
        # Jeśli relacja nie jest załadowana, SQLAlchemy dociągnie ją leniwie
        # (N+1!) — w kodzie produkcyjnym ładuj ją jawnie, np. selectinload.
        return all(loan.returned_at is not None for loan in self.loans)

    # ---- 2. Wersja SQL-owa: NOT EXISTS, bez N+1 --------------------------
    @is_available.inplace.expression
    @classmethod
    def _is_available_expression(cls) -> ColumnElement[bool]:
        return ~(
            select(Loan.id)
            .where(and_(Loan.book_id == cls.id, Loan.returned_at.is_(None)))
            .exists()
        )

    # ---- 3. Licznik aktywnych wypożyczeń (kosztowny wariant) -------------
    active_loans = column_property(
        select(func.count(Loan.id))
        .where(Loan.book_id == id)
        .correlate_except(Loan)
        .scalar_subquery(),
        deferred=True,
    )
```

Użycie i różnica w SQL:

```python
# Wersja hydrydowa — filtr po stronie bazy, jedno zapytanie:
available_titles = session.scalars(select(Book).where(Book.is_available)).all()

# SQL:
# SELECT book.id, book.title FROM book
# WHERE NOT (EXISTS (SELECT loan.id FROM loan
#                    WHERE loan.book_id = book.id AND loan.returned_at IS NULL))

# Wersja z column_property — podzapytanie w SELECT-ie, ale TYLKO gdy
# sięgniemy po atrybut (deferred=True):
print(book.active_loans)
# SELECT (SELECT count(loan.id) FROM loan WHERE loan.book_id = book.id) AS ...
# FROM book WHERE book.id = ?
```

**Uzasadnienie dla endpointu listującego 50 książek:** wybieram wersję hydrydową (`is_available`). Po pierwsze, jest to `NOT EXISTS` w `WHERE` — baza zatrzymuje się na pierwszym znalezionym aktywnym wypożyczeniu i nie musi liczyć wszystkich. Po drugie, filtruję w bazie, więc nie pobieram 50 książek z całą ich historią wypożyczeń tylko po to, żeby odrzucić 48. Po trzecie, `column_property` wymaga policzenia *wszystkich* wypożyczeń każdej książki w wyniku, a ja potrzebuję tylko informacji „jest choć jedno”. Do widoku szczegółowego, gdzie chcę pokazać liczbę, użyłbym `column_property` (albo jawnego `func.count` w zapytaniu — moduł 17).

**Minipułapka:** `deferred=True` w `column_property` jest kluczowe. Bez niego **każde** zapytanie o `Book` — w tym lista 50 pozycji w API — dociągałoby podzapytanie liczące wypożyczenia. To dokładnie ten rodzaj cichego obciążenia, który objawia się „aplikacja działa, tylko baza ma 100% CPU”.

**Druga minipułapka:** wersja Pythonowa `is_available` używa `self.loans`, co przy liście książek wywoła N+1 (moduł 11). To nie znaczy, że jest zła — znaczy, że **nie nadaje się do pętli po kolekcji**. Hybrydy są tak dobre, jak świadomość tego, która wersja uruchomi się w danym kontekście.

---

## Najczęstsze błędy i jak je czytać

| Komunikat / objaw | Przyczyna | Naprawa |
|---|---|---|
| Dwa wpisy w audycie po jednej zmianie | Listener zarejestrowany dwukrotnie (podwójny import modułu) | `if not event.contains(...): event.listen(...)`; rejestruj w jednej funkcji `register_listeners()` |
| Zdarzenie działa w testach, choć nie powinno | Zdarzenia sesji są globalne dla procesu | Wyrejestruj w teardownie: `event.remove(Session, "before_flush", fn)` |
| `TypeError: 'bool' object is not callable` na `Loan.is_overdue` w `where()` | Hybryda bez `expression` — SQLAlchemy dostał wartość Pythona | Dodaj `@is_overdue.inplace.expression` (albo `.expression`) |
| `TypeError: Boolean value of this clause is not defined` | Użyto `and_`/`or_` zamiast `&`/`\|` w wyrażeniu SQL hybrydy | Używaj `and_(...)`, `or_(...)` w kodzie SQL |
| Walidacja „nie działa” przy imporcie z CSV | `@validates` nie uruchamia się przy `insert().values([...])` | Waliduj przed `values()` albo dodaj `CHECK` w bazie |
| `AttributeError: 'NoneType' object has no attribute 'tag'` w zdarzeniu kolekcji | Zdarzenie `append`/`remove` działa w trybie `raw=True` | Dodaj `raw=False` do `@event.listens_for` |
| `oldvalue` w zdarzeniu `set` jest `None`, choć wartość była ustawiona | Brak `active_history=True` | Dodaj `active_history=True` (koszt: dodatkowe `SELECT`) |
| `InvalidRequestError: Session is already flushing` | Wywołanie `session.flush()`/`commit()` wewnątrz zdarzenia flusa | Usuń ręczny flush ze zdarzenia |
| Modyfikacje w `after_flush` nie są zapisywane | `after_flush` jest po zbudowaniu i wykonaniu SQL-a | Przenieś logikę do `before_flush` albo do `after_flush_postexec` |
| `NameError` przy imporcie modeli z `column_property` | Odwołanie do klasy, która jeszcze nie istnieje (kolejność definicji) | Zdefiniuj encję zależną wcześniej albo użyj `Mapper.add_property()` / `@declared_attr` |
| `N+1` po dodaniu hybrydy | Wersja Pythonowa hybrydy sięga do relacji w pętli | Ładuj relacje jawnie (`selectinload`) albo filtruj w SQL (`where(Book.is_available)`) |
| Audyt loguje `updated_at` przy każdej zmianie | Listener znaczników czasu działa przed listenerem audytu | Wyklucz pola techniczne (`AUDIT_IGNORED_FIELDS`) i/lub zmień kolejność rejestracji |
| Proxy nie pozwala dodać elementu | Brak `creator=` w `association_proxy` | Dodaj `creator=lambda x: AssociationObject(...)` |
| `IntegrityError` przy `book.tags.append(ten_sam_tag)` | Proxy tworzy nowy obiekt pośredni bez sprawdzenia duplikatu | Sprawdzaj obecność: `if tag not in book.tags:` |

---

## Słowniczek modułu

| Termin (EN) | Polski odpowiednik | Wyjaśnienie |
|---|---|---|
| `event` / `listener` | zdarzenie / nasłuchiwacz | Nazwany punkt w cyklu życia SQLAlchemy i funkcja, która się w nim uruchamia |
| `event.listen` / `@listens_for` | rejestracja zdarzenia | Sposób podłączenia funkcji do zdarzenia |
| `propagate` | propagacja | Czy zdarzenie mappera ma działać również dla podklas |
| `SessionEvents` | zdarzenia sesji | Zdarzenia globalne dla każdej sesji: `before_flush`, `after_commit` itd. |
| `MapperEvents` | zdarzenia mappera | Zdarzenia per encja: `before_insert`, `before_update`, `before_delete` |
| `AttributeEvents` | zdarzenia atrybutu | Zdarzenia per pole: `set`, `append`, `remove` |
| `before_flush` | przed flushem | Punkt, w którym wolno zmieniać obiekty, a zmiany wejdą do tego samego flusa |
| `after_flush` / `after_flush_postexec` | po flushu | Zdarzenia po wykonaniu SQL-a; w pierwszym nie modyfikuj obiektów |
| `session.new` / `.dirty` / `.deleted` | nowe / brudne / usunięte | Trzy kolekcje stanu sesji |
| `is_modified()` | czy zmodyfikowany | Sprawdza **realną** zmianę obiektu (z uwzględnieniem kolumn relacji) |
| `History` | historia atrybutu | Obiekt z polami `added`, `unchanged`, `deleted` |
| `attributes.get_history()` | pobierz historię | Funkcja zwracająca `History` dla pary (obiekt, atrybut) |
| `PASSIVE_OFF` | tryb pasywny wyłączony | Domyślny: dociągnij wartość z bazy, jeśli nie ma jej w pamięci |
| `PASSIVE_NO_INITIALIZE` | nie inicjalizuj | Nie dociągaj z bazy; dla niezaładowanych atrybutów zwróci pustą historię |
| `active_history` | aktywna historia | Flaga sprawiająca, że SQLAlchemy pamięta starą wartość od razu przy przypisaniu |
| `@validates` | walidacja atrybutu | Metoda uruchamiana przy przypisaniu wartości do pola encji |
| `hybrid_property` | właściwość hybrydowa | Jedna nazwa, dwie implementacje: dla Pythona i dla SQL |
| `.expression` / `.inplace.expression` | wyrażenie SQL | Część hybrydy kompilowana do SQL-a |
| `hybrid_method` | metoda hybrydowa | Jak `hybrid_property`, ale z argumentami |
| `column_property` | właściwość kolumnowa | Atrybut wyliczany w SQL (zwykle podzapytanie skorelowane) |
| `deferred=True` | odroczony | Ładuj wartość dopiero po sięgnięciu po atrybut |
| `association_proxy` | proxy asocjacji | Przezroczysty dostęp przez tabelę pośrednią |
| `creator` | konstruktor wartości | Funkcja budująca obiekt pośredni z wartości proxy |
| `synonym` | synonim atrybutu | Drugie imię dla tego samego atrybutu (alias) |
| `composite` | kompozycja | Kilka kolumn widziane jako jeden obiekt wartości |
| `attribute_mapped_collection` | kolekcja kluczowana | Relacja kolekcyjna jako słownik |
| `ordering_list` | lista uporządkowana | Kolekcja z automatycznie utrzymywaną kolumną pozycji |
| `persistent_to_deleted` | przejście w stan usunięty | Zdarzenie sesji uruchamiane, gdy obiekt staje się usuwany |
| `do_orm_execute` | wykonaj ORM | Globalny hook na wszystkie zapytania ORM (soft delete, multi-tenant) |

---

## Dalsze czytanie

Dokumentacja oficjalna (SQLAlchemy 2.0):

- **Zdarzenia Core (Engine, Connection):** <https://docs.sqlalchemy.org/en/20/core/events.html>
- **Zdarzenia ORM (przegląd):** <https://docs.sqlalchemy.org/en/20/orm/events.html>
- **Zdarzenia sesji — pełna lista z sygnaturami:** <https://docs.sqlalchemy.org/en/20/orm/session_events.html>
- **Zdarzenia mappera:** <https://docs.sqlalchemy.org/en/20/orm/events.html#mapper-events>
- **Zdarzenia atrybutów (`set`, `append`, `remove`, `raw`, `active_history`):** <https://docs.sqlalchemy.org/en/20/orm/events.html#attribute-events>
- **`Session.is_modified`:** <https://docs.sqlalchemy.org/en/20/orm/session_api.html#sqlalchemy.orm.Session.is_modified>
- **Historia atrybutów (`History`, `get_history`):** <https://docs.sqlalchemy.org/en/20/orm/attributes.html#sqlalchemy.orm.attributes.get_history>
- **`@validates` i konfiguracja mappera:** <https://docs.sqlalchemy.org/en/20/orm/mapper_config.html#sqlalchemy.orm.validates>
- **Hybrydy (`hybrid_property`, `hybrid_method`, `inplace`):** <https://docs.sqlalchemy.org/en/20/orm/extensions/hybrid.html>
- **`column_property`:** <https://docs.sqlalchemy.org/en/20/orm/mapped_sql_expr.html#using-column-property>
- **`association_proxy`:** <https://docs.sqlalchemy.org/en/20/orm/extensions/associationproxy.html>
- **`composite`:** <https://docs.sqlalchemy.org/en/20/orm/composites.html>
- **`ordering_list`:** <https://docs.sqlalchemy.org/en/20/orm/extensions/orderinglist.html>
- **Przykład „Versioning objects” (audyt historii — rozszerzenie wzorca z 13.3):** <https://docs.sqlalchemy.org/en/20/orm/examples.html#versioning-objects>
- **SQLAlchemy 2.1 — co nowego (dla ramek 🆕):** <https://docs.sqlalchemy.org/en/21/changelog/migration_21.html>

---

## Co dalej

Umiemy już reagować na zmiany i rozszerzać modele. Ale wszystko, co zrobiliśmy w tym module, dzieje się wewnątrz jednej transakcji i w jednym procesie. Nie zadaliśmy ani razu pytania: **co się stanie, gdy dwie osoby zrobią to samo w tej samej sekundzie?** W [module 14](14_transakcje_i_wspolbieznosc.md) zajmiemy się transakcjami, poziomami izolacji, blokowaniem pesymistycznym i optymistycznym, wyścigami oraz deadlockami — i przekonasz się, że zdarzenia, które tu poznałeś, są kluczowym narzędziem przy implementacji blokady optymistycznej opartej na kolumnie wersji.

<!-- koniec modułu 13 -->