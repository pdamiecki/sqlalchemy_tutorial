# Moduł 14 — Transakcje i współbieżność

W tym module przechodzimy do tematu, który oddziela aplikacje „działające na moim laptopie” od aplikacji, które nie gubią danych, gdy jednocześnie korzysta z nich tysiąc osób. Nauczysz się, czym naprawdę jest transakcja (bez akademickiego żargonu), jak cztery obietnice ACID przekładają się na konkretne zachowania bazy, jak wybrać poziom izolacji i kiedy jest to w ogóle potrzebne, jak świadomie blokować wiersze (pesymistycznie i optymistycznie), jak wykrywać i naprawiać wyścigi (race conditions) oraz jak zaprojektować operacje, które można bezpiecznie ponowić. Zakończymy warsztatem, w którym pięć równoległych procesów próbuje kupić ostatnią sztukę towaru — i tylko jeden powinien wygrać.

| | |
|---|---|
| **Poziom** | 🔴 architektoniczny |
| **Czas** | ~210 minut nauki + ~90 minut ćwiczeń |
| **Wymagania wstępne** | [`08_sesja_cykl_zycia.md`](08_sesja_cykl_zycia.md) — sesja, flush, commit, granice transakcji sesji · [`10_zapytania_orm.md`](10_zapytania_orm.md) — `select()`, `session.execute()` · [`13_zdarzenia_i_hybrydy.md`](13_zdarzenia_i_hybrydy.md) — zdarzenia sesji |
| **Czego dotyczy ten plik** | Projektowanie bezpiecznych operacji współbieżnych: ACID, poziomy izolacji, savepointy, `with_for_update()`, wersjonowanie optymistyczne, idempotencja, deadlocki, długie transakcje, pula połączeń, MVCC, testowanie wyścigów |
| **Bazy używane w przykładach** | SQLite (uruchomienie bez konfiguracji) oraz **PostgreSQL 16+** (tam, gdzie zachowanie zależy od dialektu — a w tym module zależy bardzo często) |

> ⚠️ **Pułapka na wejściu — przeczytaj to najpierw.** SQLite nie ma blokad wierszowych i nie obsługuje klauzuli `FOR UPDATE`. Wszystkie przykłady dotyczące blokowania pesymistycznego uruchomisz sensownie **tylko na PostgreSQL**. Na SQLite uruchomią się bez błędu, ale pokażą inne wyniki — i to też jest pouczające. Wszędzie, gdzie zachowanie zależy od dialektu, oznaczam to jawnie.

## Spis treści

1. [Transakcja od zera — cztery obietnice ACID](#1-transakcja-od-zera--cztery-obietnice-acid)
2. [Poziomy izolacji](#2-poziomy-izolacji)
3. [Ustawianie poziomu izolacji w SQLAlchemy](#3-ustawianie-poziomu-izolacji-w-sqlalchemy)
4. [Zagnieżdżone transakcje, savepointy i `join_transaction_mode`](#4-zagnieżdzone-transakcje-savepointy-i-join_transaction_mode)
5. [Blokowanie pesymistyczne: `with_for_update()`](#5-blokowanie-pesymistyczne-with_for_update)
6. [Blokowanie optymistyczne: kolumna wersji](#6-blokowanie-optymistyczne-kolumna-wersji)
7. [Wyścigi i idempotencja](#7-wyscigi-i-idempotencja)
8. [Deadlocki](#8-deadlocki)
9. [Długie transakcje — najczęstszy zabójca wydajności](#9-dlugie-transakcje--najczestszy-zabojca-wydajnosci)
10. [Pula połączeń pod obciążeniem](#10-pula-polaczen-pod-obciazeniem)
11. [MVCC w PostgreSQL kontra model blokujący SQLite](#11-mvcc-w-postgresql-kontra-model-blokujacy-sqlite)
12. [Testowanie współbieżności](#12-testowanie-wspolbieznosci)
13. [Warsztat obowiązkowy: „kup ostatnią sztukę” w czterech wersjach](#13-warsztat-obowiazkowy-kup-ostatnia-sztuke-w-czterech-wersjach)
14. [Podsumowanie](#podsumowanie)

---

## 1. Transakcja od zera — cztery obietnice ACID

### 1.1 Problem: przelew, który znika w połowie

Wyobraź sobie, że piszesz kod przelewu bankowego. Potrzebujesz dwóch operacji:

1. zabierz 100 zł z konta Ani,
2. dodaj 100 zł na konto Bartka.

Między tymi dwiema operacjami istnieje ułamek sekundy. Co się stanie, jeśli dokładnie w tym momencie wyłączą prąd w serwerowni? Albo jeśli baza odrzuci drugie polecenie, bo skończyło się miejsce na dysku? Albo jeśli Twój proces zostanie ubity przez system (OOM killer) po pierwszym `UPDATE`?

Bez żadnych zabezpieczeń: Ania traci 100 zł, Bartek nic nie dostaje, a 100 zł rozpływa się w powietrzu. To nie jest hipotetyczny scenariusz — to najczęstsza klasa błędów w systemach finansowych, które „działały, dopóki nie zaczęły działać pod obciążeniem”.

> 💡 **Analogia — przelew bankowy jako koperta.** Transakcja to koperta, do której wkładasz wszystkie polecenia związane z jedną sprawą. Baza nie wykonuje ich „pojedynczo i od razu”. Zbiera je, a potem albo **przykleja całą kopertę i wysyła** (commit), albo **wrzuca kopertę do niszczarki** (rollback). Nie ma stanu pośredniego „pół koperty wysłane”. Ta analogia wystarczy do zbudowania modelu mentalnego — teraz zdejmiemy ją i zamienimy na precyzję.

**Transakcja** to logiczna jednostka pracy: zbiór operacji, który baza traktuje jako nierozdzielną całość. Kluczowe słowo: *logiczna*. Baza może wewnętrznie wykonać te operacje w dowolnej kolejności i w dowolnym momencie, ale z punktu widzenia każdego innego użytkownika systemu efekt jest taki, jakby stały się jednym, niepodzielnym zdarzeniem.

### 1.2 ACID jako cztery obietnice

ACID to akronim czterech obietnic, które baza składa Twojej aplikacji. Nie są to „funkcje”, które się włącza i wyłącza — to kontrakt, który baza realizuje (albo nie, i wtedy mówimy o „bazie niezgodnej z ACID”, jak niektóre bazy NoSQL w trybie domyślnym).

**A — Atomicity (atomowość).** Wszystko albo nic. Jeśli transakcja zawiera 5 poleceń i piąte się nie powiedzie, pierwsze cztery zostaną cofnięte. Nie ma „częściowego sukcesu”.

**C — Consistency (spójność).** Transakcja przenosi bazę ze stanu poprawnego do stanu poprawnego. „Poprawny” definiujesz Ty — przez ograniczenia (`CHECK`, `FOREIGN KEY`, `UNIQUE`, `NOT NULL`) i przez logikę aplikacji. Baza gwarantuje, że nie zaczniesz z poprawnych danych i nie skończysz z niepoprawnymi. Uwaga na nieporozumienie: spójność **nie** oznacza, że baza zna Twoje reguły biznesowe. Jeśli nie zapiszesz reguły „stan magazynu nigdy nie może być ujemny” jako ograniczenia, baza o niej nie wie i jej nie ochroni.

**I — Isolation (izolacja).** Transakcje nie widzą nawzajem swoich niezakończonych zmian. W praktyce oznacza to, że w środku transakcji widzisz świat „zamrożony” na tyle, na ile pozwala wybrany poziom izolacji (o tym cała sekcja 2). To najbardziej subtelna z czterech obietnic i źródło 90% błędów współbieżności w realnych systemach.

**D — Durability (trwałość).** Po `COMMIT` dane przetrwają awarię zasilania, restart serwera i awarię procesu. Baza zapisuje wcześniej transakcję do trwałego dziennika (write-ahead log, WAL), zanim powie Ci „gotowe”. To dlatego `COMMIT` jest droższy niż `SELECT` — czeka na dysk.

> 🧠 **Dlaczego tak jest — kompromis „szybko czy bezpiecznie”.** Każda z tych obietnic kosztuje. Trwałość kosztuje zapis na dysk. Izolacja kosztuje blokady albo wersjonowanie. Dlatego w bazach istnieją „poziomy izolacji”: im wyższy, tym więcej gwarancji i tym mniej równoległości. Inżynieria polega na wybraniu **najniższego poziomu, który nadal jest poprawny** dla danego przypadku użycia — a nie na ustawieniu wszędzie najwyższego „na wszelki wypadek”.

### 1.3 Co się dzieje przy awarii — scenariusze

| Scenariusz | Bez transakcji | Z transakcją |
|---|---|---|
| Wyjątek w kodzie po pierwszym `UPDATE` | Pierwszy `UPDATE` zostaje w bazie | Całość cofnięta |
| Zerwanie połączenia w połowie | Część zmian utrwalona | Baza cofa transakcję przy rozłączeniu |
| Restart serwera bazy | Część zmian może zostać | WAL odtwarza lub cofa całość |
| Równoległy odczyt „w połowie” operacji | Widzi stan pośredni | Nie widzi zmian do `COMMIT` |
| OOM / `kill -9` procesu aplikacji | Część zmian zostaje | Baza wykrywa rozłączenie i cofa |

Ostatni wiersz jest ważny i często mylony: **to nie aplikacja decyduje o rollbacku po nagłym zabiciu procesu — robi to baza**, wykrywając zamknięcie połączenia TCP. Dlatego nawet jeśli Twój kod nie ma `try/except`, baza nie zostawi „pół transakcji”. Ale zostawi transakcję **zawieszoną** — czyli połączenie w stanie `idle in transaction`, dopóki nie wykryje, że klient zniknął. Do tego wrócimy w sekcji 9.

### 1.4 Jak to wygląda w SQLAlchemy — poziom Core

W SQLAlchemy Core transakcja to kontekst `engine.begin()`. Uruchom to i przeczytaj komentarze obok logu SQL:

```python
# examples/14_01_acid_basics.py
"""ACID w praktyce: dwie instrukcje jako jedna nierozdzielna całość."""

from sqlalchemy import create_engine, text

engine = create_engine("sqlite:///acid_demo.db", echo=True)

with engine.begin() as conn:  # BEGIN
    conn.execute(
        text("CREATE TABLE IF NOT EXISTS account (id INTEGER PRIMARY KEY, balance INTEGER NOT NULL)")
    )
    conn.execute(text("DELETE FROM account"))
    conn.execute(text("INSERT INTO account (id, balance) VALUES (1, 100), (2, 50)"))

print("--- przelew poprawny ---")
with engine.begin() as conn:  # BEGIN
    conn.execute(text("UPDATE account SET balance = balance - 30 WHERE id = 1"))
    conn.execute(text("UPDATE account SET balance = balance + 30 WHERE id = 2"))
    # COMMIT (kontekst wychodzi bez wyjątku)

print("--- przelew przerwany wyjątkiem ---")
try:
    with engine.begin() as conn:  # BEGIN
        conn.execute(text("UPDATE account SET balance = balance - 30 WHERE id = 1"))
        raise RuntimeError("awaria w połowie przelewu")
        # poniższa linia nigdy się nie wykona
        conn.execute(text("UPDATE account SET balance = balance + 30 WHERE id = 2"))
except RuntimeError as exc:
    print(f"Przechwycono: {exc}")
    # engine.begin() wykonał już ROLLBACK

with engine.connect() as conn:
    print(conn.execute(text("SELECT id, balance FROM account ORDER BY id")).all())
```

> 🔬 **Pod maską.** `engine.begin()` emituje `BEGIN` (albo `BEGIN (implicit)` — zależy od dialektu) przy wejściu do bloku i `COMMIT` przy wyjściu bez wyjątku. Przy wyjątku — `ROLLBACK`. W SQLite dialekt pysqlite dodatkowo ma własne zasady rozpoczynania transakcji (o tym w sekcji 11).

Wynik drugiej części: salda pozostają `(1, 70)` i `(2, 80)` — pierwszy `UPDATE` z nieudanej transakcji został cofnięty. To jest atomowość w działaniu.

> ⚠️ **Pułapka — `engine.connect()` bez `begin()`.** W SQLAlchemy 2.0 `engine.connect()` samo w sobie rozpoczyna transakcję (autobegin), ale **musisz** ją zakończyć: albo `conn.commit()`, albo `conn.rollback()`. Jeśli po prostu wyjdziesz z bloku `with`, transakcja zostanie **cofnięta**, nie zatwierdzona. To zmiana względem 1.x, gdzie często polegało się na niejawnym commicie przy zamknięciu. Zasada praktyczna: przy zapisach używaj `engine.begin()`, a `engine.connect()` tylko do odczytów.

---

## 2. Poziomy izolacji

### 2.1 Analogia: kilku edytorów tego samego dokumentu

Wyobraź sobie dokument w chmurze, nad którym pracuje pięć osób. Ktoś wpisuje zdanie, ktoś inny zmienia nagłówek, jeszcze ktoś usuwa akapit. Jak zapobiec temu, żebyście się wzajemnie nie „nadpisali”? Możesz:

- **pozwolić wszystkim pisać jednocześnie i nie przejmować się bałaganem** (niska izolacja, maksymalna równoległość),
- **dać każdemu własną kopię dokumentu, a przy zapisie sprawdzić, czy nic się nie zmieniło** (izolacja snapshotowa),
- **wpuścić do dokumentu jedną osobę naraz i zamknąć drzwi** (pełna serializacja, minimalna równoległość).

To dokładnie trzy strategie, które realizują poziomy izolacji. Każda jest poprawna — dla innego problemu.

### 2.2 Cztery anomalie

Zanim omówimy poziomy, musimy nazwać choroby, które te poziomy mają leczyć. Wszystkie cztery wynikają z tego, że dwie transakcje wykonują się naprzemiennie.

**Dirty read (brudny odczyt).** Transakcja A czyta dane zapisane przez transakcję B, która **jeszcze nie zatwierdziła** swoich zmian. Jeśli B się wycofa, A działa na danych, które nigdy nie istniały. Analogia: czytasz notatkę kogoś, kto jeszcze ją pisze i zaraz ją wyrzuci do kosza.

**Non-repeatable read (odczyt niepowtarzalny).** W tej samej transakcji czytasz ten sam wiersz dwukrotnie i dostajesz dwie różne wartości, bo ktoś pomiędzy je zmienił i zatwierdził. Analogia: sprawdzasz cenę w sklepie, idziesz po portfel, wracasz — cena jest inna.

**Phantom read (odczyt widmowy).** W tej samej transakcji wykonujesz to samo zapytanie zakresowe (`WHERE stock > 0`) dwukrotnie i dostajesz **inną liczbę wierszy**, bo ktoś pomiędzy dodał lub usunął wiersz pasujący do warunku. To „duch” — nie zmienił się żaden z wierszy, które widziałeś, ale pojawił się nowy.

**Write skew (przekrzywienie zapisu).** Najbardziej podstępna. Dwie transakcje czytają wspólny stan, każda na jego podstawie podejmuje decyzję i każda zapisuje **inny** wiersz. Każda z osobna jest poprawna. Razem łamią niezmiennik.

> 💡 **Analogia do write skew — dwóch lekarzy na dyżurze.** Reguła szpitala: „na dyżurze musi być co najmniej jeden lekarz”. Dyżurują Ania i Bartek. Ania patrzy na grafik, widzi dwoje, myśli „mogę się zwolnić” i wypisuje się. Bartek robi dokładnie to samo w tej samej milisekundzie. Oboje zatwierdzają. Efekt: na dyżurze nie ma nikogo. Żadna transakcja nie nadpisała danych drugiej — dlatego zwykłe blokady wierszy tego nie łapią.

Z write skew wiąże się jeszcze **lost update (zgubiona aktualizacja)**, czyli klasyk „odczytaj–zmodyfikuj–zapisz”: dwie transakcje czytają `stock = 1`, obie liczą `1 - 1 = 0` i obie zapisują `0`. Sprzedano dwie sztuki, w bazie zostało zero, choć powinno być minus jeden — a przy dobrym ograniczeniu `CHECK` po prostu jeden z zapisów powinien się nie udać.

### 2.3 Tabela: anomalia × poziom izolacji

Standard SQL definiuje cztery poziomy. Tabela poniżej pokazuje, **jakie anomalie pozostają możliwe** na każdym poziomie.

| Poziom izolacji | Dirty read | Non-repeatable read | Phantom read | Lost update | Write skew |
|---|---|---|---|---|---|
| `READ UNCOMMITTED` | ✅ możliwy | ✅ możliwy | ✅ możliwy | ✅ możliwy | ✅ możliwy |
| `READ COMMITTED` | ❌ | ✅ możliwy | ✅ możliwy | ✅ możliwy | ✅ możliwy |
| `REPEATABLE READ` *(standard SQL)* | ❌ | ❌ | ✅ możliwy | ✅ możliwy | ✅ możliwy |
| `REPEATABLE READ` *(PostgreSQL)* | ❌ | ❌ | ❌ | ❌ | ✅ możliwy |
| `SERIALIZABLE` | ❌ | ❌ | ❌ | ❌ | ❌ |

> ⚠️ **Pułapka — PostgreSQL jest „lepszy niż standard”, a to myli.** W standardzie SQL `REPEATABLE READ` dopuszcza phantom reads. PostgreSQL implementuje ten poziom jako **snapshot isolation**: cała transakcja widzi jedną, niezmienną migawkę bazy z momentu jej rozpoczęcia. Dzięki temu phantomy i lost updates **nie występują** na `REPEATABLE READ`. Ale write skew nadal występuje — i to jest powód, dla którego „ustawmy REPEATABLE READ i problem zniknie” jest błędnym wnioskiem. Zawsze sprawdzaj tabelę dla **konkretnej bazy**, nie dla standardu.

> ⚠️ **Pułapka — `READ UNCOMMITTED` w PostgreSQL nie istnieje.** PostgreSQL akceptuje to słowo, ale realizuje je jako `READ COMMITTED`. Nie możesz więc w PostgreSQL „przyspieszyć” aplikacji, obniżając izolację do `READ UNCOMMITTED`. W MySQL (InnoDB) i w SQL Server ten poziom faktycznie coś zmienia.

### 2.4 Domyślne poziomy w praktyce

| Baza | Domyślny poziom | Uwagi |
|---|---|---|
| PostgreSQL 16/17 | `READ COMMITTED` | Każde **polecenie** widzi świeżą migawkę; w obrębie jednego polecenia migawka jest stała |
| MySQL 8 (InnoDB) | `REPEATABLE READ` | Snapshot isolation dla odczytów, blokady przy zapisach |
| SQL Server | `READ COMMITTED` | Domyślnie z blokadami (nie MVCC) |
| SQLite | brak sensownego wyboru | Cała baza blokowana przy zapisie; patrz sekcja 11 |
| Oracle | `READ COMMITTED` | Snapshot per polecenie, podobnie jak PostgreSQL |

Dla większości aplikacji webowych `READ COMMITTED` w PostgreSQL jest **właściwym** wyborem i nie należy go zmieniać. Zmienia się go tylko wtedy, gdy konkretna operacja ma wymaganie, którego `READ COMMITTED` nie spełnia — i wtedy ustawia się wyższy poziom **punktowo**, dla tej jednej operacji.

> 🧠 **Dlaczego tak jest — `READ COMMITTED` a „każde polecenie osobno”.** W PostgreSQL na `READ COMMITTED` dwie instrukcje w tej samej transakcji mogą zobaczyć różne stany świata. To brzmi strasznie, ale w praktyce jest bardzo wygodne: nie musisz się bać, że Twoja transakcja „zamrozi” się na minutę i będzie pracować na nieaktualnych danych. Snapshot trzyma się tylko na czas jednego polecenia. Konsekwencja: jeśli w jednej transakcji robisz `SELECT` → liczysz coś w Pythonie → `UPDATE`, to między `SELECT` a `UPDATE` świat mógł się zmienić. To jest dokładnie źródło lost update.

### 2.5 Jak samodzielnie zobaczyć różnicę

Najlepszy sposób nauki: uruchom dwie sesje `psql` (albo dwa skrypty Python) i wykonaj naprzemiennie poniższe polecenia. To doświadczenie zapamiętasz lepiej niż tabelę.

```text
# Terminal 1 (A)                              # Terminal 2 (B)
BEGIN;
UPDATE account SET balance = balance - 100
  WHERE id = 1;
                                              BEGIN;
                                              SELECT balance FROM account WHERE id = 1;
                                              -- na READ COMMITTED: nadal 100 (A nie zatwierdziła)
COMMIT;
                                              SELECT balance FROM account WHERE id = 1;
                                              -- teraz 0
```

W PostgreSQL na `READ COMMITTED` druga transakcja nie zobaczy zmian, dopóki pierwsza nie zatwierdzi. Ale uwaga — jeśli A i B **jednocześnie modyfikują ten sam wiersz**, B zostanie **zablokowana** do momentu zakończenia A, a następnie PostgreSQL ponownie oceni warunek `WHERE` na już zaktualizowanym wierszu. To zachowanie nazywa się *re-check* i jest jednym z powodów, dla których `UPDATE ... WHERE warunek` jest bezpieczniejszy niż `SELECT` → modyfikacja w Pythonie → `UPDATE`.

> 🧪 **Ćwiczenie — zbadaj re-check.** W PostgreSQL, na `READ COMMITTED`, wykonaj w A: `BEGIN; UPDATE product SET stock = 5 WHERE id = 1;` (bez commita). W B: `BEGIN; UPDATE product SET stock = stock - 1 WHERE id = 1 AND stock > 0;`. Co robi B? Czeka, czy od razu się kończy? Co się dzieje po `COMMIT` w A? Odpowiedź uzasadnij na podstawie mechanizmu re-check.

---

## 3. Ustawianie poziomu izolacji w SQLAlchemy

### 3.1 Uczciwe ostrzeżenie o API

> ⚠️ **Pułapka — `Session(isolation_level=...)` NIE istnieje.** `Session` (sesja ORM) jest *fasadą* nad silnikiem i połączeniem — celowo nie wystawia poziomu izolacji jako własnego parametru. Poziom izolacji ustawia się na `Engine` lub na `Connection`. W dokumentacji SQLAlchemy jest to powiedziane wprost: sesja „acts as a facade for engines and connections, but does not expose transaction isolation directly”. Jeśli szukasz sposobu na ustawienie izolacji per transakcja — użyjesz `session.connection(execution_options={...})` (sekcja 3.3).

### 3.2 Globalnie — na `Engine`

To najprostszy i najczęściej wystarczający sposób: cała aplikacja działa na jednym poziomie izolacji.

```python
# examples/14_02_isolation_engine.py
"""Ustawienie poziomu izolacji na silniku — dotyczy wszystkich połączeń."""

from sqlalchemy import create_engine, text
from sqlalchemy.orm import sessionmaker

# PostgreSQL: poziom ustawiany raz, przy tworzeniu puli połączeń.
engine = create_engine(
    "postgresql+psycopg://app:secret@localhost:5432/shop",
    isolation_level="REPEATABLE READ",
    echo=True,
)

SessionLocal = sessionmaker(bind=engine, expire_on_commit=False)

with SessionLocal() as session:
    # W logu zobaczysz: BEGIN ISOLATION LEVEL REPEATABLE READ (implicit)
    print(session.scalars(text("SELECT 1")).all())
```

Wariant drugi, gdy chcesz mieć **dwa** silniki na różnych poziomach, ale dzielić jedną pulę połączeń. `Engine.execution_options()` tworzy płytką kopię silnika — dzieli z rodzicem tę samą pulę, ale ma inne opcje wykonania:

```python
# examples/14_03_isolation_two_engines.py
"""Jeden pool, dwa poziomy izolacji — wzorzec 'transakcyjny' vs 'autocommit'."""

from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

engine = create_engine("postgresql+psycopg://app:secret@localhost:5432/shop")

# Kopia silnika dzieląca pulę połączeń, ale z inną izolacją.
strict_engine = engine.execution_options(isolation_level="SERIALIZABLE")

DefaultSession = sessionmaker(bind=engine)          # READ COMMITTED
StrictSession = sessionmaker(bind=strict_engine)    # SERIALIZABLE
```

### 3.3 Per transakcja — przez `Session.connection()`

To jest sposób na ustawienie izolacji **dla jednej konkretnej transakcji**, gdy reszta aplikacji działa na domyślnym poziomie. Kluczowe: `session.connection()` musisz wywołać **zanim** wykonasz jakąkolwiek operację w tej sesji.

```python
# examples/14_04_isolation_per_transaction.py
"""Izolacja SERIALIZABLE tylko dla jednej, krytycznej operacji."""

from sqlalchemy import select
from sqlalchemy.orm import Session

from models import Account  # przykładowy model z poprzednich modułów


def transfer_strict(session: Session, from_id: int, to_id: int, amount: int) -> None:
    """Przelew w izolacji SERIALIZABLE — uruchom PRZED jakąkolwiek operacją w sesji."""
    # UWAGA: to musi być pierwsza operacja na tej sesji.
    session.connection(execution_options={"isolation_level": "SERIALIZABLE"})

    src = session.scalars(select(Account).where(Account.id == from_id)).one()
    dst = session.scalars(select(Account).where(Account.id == to_id)).one()
    if src.balance < amount:
        raise ValueError("brak środków")

    src.balance -= amount
    dst.balance += amount
    session.commit()
    # Po commicie połączenie wraca do puli z PRZYWRÓCONYM domyślnym poziomem.
```

> 🧠 **Dlaczego tak jest — izolacja „resetuje się” po commicie.** Poziom izolacji jest własnością **transakcji w bazie**, nie połączenia. Gdy transakcja się kończy, SQLAlchemy przywraca na połączeniu poziom domyślny, zanim odda je do puli. Dzięki temu kolejny użytkownik tego samego połączenia nie odziedziczy „cudzej” izolacji. To nie jest szczegół implementacyjny — to gwarancja, na której możesz polegać.

> ⚠️ **Pułapka — nie zmieniaj izolacji w trakcie transakcji.** Dokumentacja SQLAlchemy ostrzega wprost: poziomu izolacji **nie można bezpiecznie zmieniać na połączeniu, na którym transakcja już się rozpoczęła**. Bazy danych nie potrafią zmienić izolacji trwającej transakcji, a różne sterowniki zachowują się w tym obszarze niekonsekwentnie. Jeśli spróbujesz, możesz dostać `InvalidRequestError` albo — gorzej — ciche zignorowanie ustawienia.

### 3.4 Poziom Core — `Connection.execution_options()`

```python
# examples/14_05_isolation_core.py
"""Izolacja na poziomie pojedynczego połączenia (Core)."""

from sqlalchemy import create_engine, text

engine = create_engine("postgresql+psycopg://app:secret@localhost:5432/shop")

with engine.connect() as conn:
    conn = conn.execution_options(isolation_level="SERIALIZABLE")
    # UWAGA: execution_options() zwraca NOWY obiekt Connection — trzeba go użyć!
    conn.execute(text("SELECT count(*) FROM orders"))
    conn.commit()
```

> ⚠️ **Pułapka — `execution_options()` zwraca nowy obiekt.** To nie jest metoda modyfikująca `self`. `conn.execution_options(...)` bez przypisania wyniku do zmiennej to klasyczny błąd „napisałem, a nie działa”. Dotyczy to również `Select.execution_options()` i `Session.connection(execution_options=...)` (tu na szczęście wynik nie jest potrzebny).

Dostępne wartości to łańcuchy: `"AUTOCOMMIT"`, `"READ COMMITTED"`, `"READ UNCOMMITTED"`, `"REPEATABLE READ"`, `"SERIALIZABLE"`. Nie wszystkie dialekty obsługują wszystkie — sprawdź w dokumentacji dialektu, jeśli pracujesz z czymś egzotycznym.

### 3.5 `AUTOCOMMIT` — i dlaczego to nie jest poziom izolacji

`isolation_level="AUTOCOMMIT"` to specjalna wartość, która **wyłącza transakcje** na sterowniku: każde polecenie zatwierdza się samo. Używa się jej do operacji administracyjnych (`CREATE INDEX CONCURRENTLY`, `VACUUM`, `ALTER TYPE ... ADD VALUE`), które nie mogą działać w transakcji.

```python
# examples/14_06_autocommit.py
"""AUTOCOMMIT: operacje administracyjne, które nie mogą działać w transakcji."""

from sqlalchemy import create_engine, text

engine = create_engine("postgresql+psycopg://app:secret@localhost:5432/shop")

with engine.connect().execution_options(isolation_level="AUTOCOMMIT") as conn:
    conn.execute(text("CREATE INDEX CONCURRENTLY ix_orders_created_at ON orders (created_at)"))
```

> ⚠️ **Pułapka — `AUTOCOMMIT` to nie „szybszy tryb”.** Wyłączenie transakcji oznacza, że **nie masz atomowości**. Dwa polecenia w trybie `AUTOCOMMIT` to dwie niezależne transakcje. Jeśli pierwsze się powiedzie, a drugie nie, nie ma jak tego cofnąć. W aplikacji biznesowej `AUTOCOMMIT` powinien być rzadkością, zarezerwowaną dla migracji i administracji.

> 🆕 **SQLAlchemy 2.1 — autoflush w sesji działa bezwarunkowo.** W 2.1 zmieniono semantykę `autoflush` w `Session`: jest teraz stosowany bezwarunkowo, co usuwa pewną klasę zaskakujących zachowań, w których flush „nie odpalał się” w kontekstach, w których programista się go spodziewał. Jeśli w 2.0 miałeś kod polegający na tym, że autoflush **nie** zadziała w jakimś nietypowym miejscu, po aktualizacji do 2.1 może zachować się inaczej. Sprawdź to w testach.

---

## 4. Zagnieżdżone transakcje, savepointy i `join_transaction_mode`

### 4.1 Po co komu transakcja w transakcji

Skoro transakcja jest nierozdzielna, po co „transakcja w transakcji”? Odpowiedź: żeby móc **cofnąć tylko część pracy**, nie tracąc reszty.

> 💡 **Analogia — zapisywanie gry w środku poziomu.** Przechodzisz długi poziom. Nie chcesz, żeby pojedynczy nieudany skok kosztował Cię cały poziom. Więc zapisujesz stan w środku (savepoint). Jeśli skok się nie uda, wracasz do ostatniego zapisu — a nie do początku poziomu.

**Savepoint** to znacznik w środku transakcji. Możesz się do niego cofnąć (`ROLLBACK TO SAVEPOINT`), nie przerywając całej transakcji. W SQLAlchemy obsługuje to `Session.begin_nested()`.

### 4.2 `begin_nested()` w praktyce

```python
# examples/14_07_savepoint.py
"""Import wierszy: jeden błędny wiersz nie psuje całego importu."""

from sqlalchemy import create_engine, select, text
from sqlalchemy.orm import Session

engine = create_engine("sqlite:///import_demo.db")

with engine.begin() as conn:
    conn.execute(
        text(
            "CREATE TABLE IF NOT EXISTS customer ("
            " id INTEGER PRIMARY KEY, email TEXT NOT NULL UNIQUE)"
        )
    )

ROWS = [
    {"email": "ania@example.com"},
    {"email": "bartek@example.com"},
    {"email": "ania@example.com"},  # duplikat -> IntegrityError
    {"email": "celina@example.com"},
]

with Session(engine) as session:
    session.execute(text("DELETE FROM customer"))

    for index, row in enumerate(ROWS):
        try:
            with session.begin_nested():  # SAVEPOINT
                session.execute(
                    text("INSERT INTO customer (email) VALUES (:email)"),
                    {"email": row["email"]},
                )
            print(f"OK      wiersz {index}: {row['email']}")
        except Exception as exc:  # noqa: BLE001 - demonstracja
            # savepoint został już cofnięty przez kontekst begin_nested()
            print(f"ODRZUCONY wiersz {index}: {row['email']} ({type(exc).__name__})")

    session.commit()  # zatwierdza wszystko poza odrzuconymi wierszami

with Session(engine) as session:
    emails = session.scalars(text("SELECT email FROM customer ORDER BY email")).all()
    print("W bazie:", emails)
```

> 🔬 **Pod maską.** Emitowany SQL wygląda tak:
>
> ```sql
> BEGIN (implicit)
> SAVEPOINT sa_savepoint_1
> INSERT INTO customer (email) VALUES (?)
> RELEASE SAVEPOINT sa_savepoint_1
> SAVEPOINT sa_savepoint_2
> INSERT INTO customer (email) VALUES (?)
> RELEASE SAVEPOINT sa_savepoint_2
> ...
> ROLLBACK TO SAVEPOINT sa_savepoint_3   -- to dla duplikatu
> SAVEPOINT sa_savepoint_4
> INSERT INTO customer (email) VALUES (?)
> RELEASE SAVEPOINT sa_savepoint_4
> COMMIT
> ```
>
> Zwróć uwagę na kolejność: `SAVEPOINT`, potem operacja, potem `RELEASE SAVEPOINT`. Dopiero `COMMIT` na końcu utrwala wszystko. To dokładnie ten sam wzorzec, który zastosujesz w imporcie CSV z modułu 05 — tylko tam pokazany od strony Core.

> ⚠️ **Pułapka — po błędzie w transakcji PostgreSQL odmawia dalszej pracy.** W PostgreSQL, gdy polecenie w transakcji zakończy się błędem, cała transakcja wchodzi w stan „aborted” i **każde kolejne polecenie** kończy się komunikatem `current transaction is aborted, commands ignored until end of transaction block`. Bez savepointu nie da się kontynuować. Z savepointem — da się, bo `ROLLBACK TO SAVEPOINT` przywraca transakcję do stanu używalności. To jest praktyczne uzasadnienie istnienia `begin_nested()`.

### 4.3 Co się dzieje z `commit()` wewnątrz savepointu — zmiana w 2.0

> 🧠 **Dlaczego tak jest — zachowanie 2.0 odwraca zachowanie 1.x.** W SQLAlchemy 2.0 `session.commit()` **zawsze** zatwierdza **najbardziej zewnętrzną** transakcję — nawet jeśli jesteś w środku `begin_nested()`. W 1.x bywało inaczej (commit mógł „uwalniać” savepoint). Dokumentacja mówi o tym wprost: „this is a SQLAlchemy 2.0 specific behavior that is reversed from the 1.x series”. Praktyczny wniosek: **savepoint zatwierdzasz i cofasz przez obiekt zwrócony z `begin_nested()`**, a nie przez `session.commit()`. Najbezpieczniej używać `begin_nested()` jako kontekstu (`with`) i nie mieszać go z ręcznym `commit()`.

### 4.4 `twophase=True` — commit dwufazowy

**Two-phase commit (2PC)** to mechanizm koordynujący commit **na kilku bazach jednocześnie**. Baza A i baza B mają zatwierdzić zmiany razem — albo żadna. Realizuje się to w dwóch etapach: najpierw wszystkie bazy „przygotowują się” (`PREPARE TRANSACTION`), potem wszystkie zatwierdzają.

```python
# examples/14_08_twophase.py
"""Commit dwufazowy — tylko dla PostgreSQL i MySQL."""

from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

engine_a = create_engine("postgresql+psycopg://app:secret@db-a:5432/shop")
engine_b = create_engine("postgresql+psycopg://app:secret@db-b:5432/analytics")

TwoPhaseSession = sessionmaker(binds={None: engine_a, object: engine_b}, twophase=True)
```

> ⚠️ **Pułapka — 2PC to rzadko dobry pomysł.** Commit dwufazowy wymaga trwałego dziennika przygotowanych transakcji, blokuje zasoby między etapami i bywa źródłem „utkniętych” transakcji wymagających ręcznej interwencji administratora. Jeśli rozważasz 2PC, to zwykle znak, że potrzebujesz wzorca **outbox** (zapisz intencję w tej samej transakcji, a potem wykonaj efekt asynchronicznie). W praktyce biznesowej 2PC stosuje się znacznie rzadziej, niż sugeruje jego obecność w dokumentacji.

### 4.5 `join_transaction_mode` — nowość 2.0, którą warto znać

Co się dzieje, gdy przekażesz sesji `Connection`, na którym **już** trwa transakcja? Odpowiedź zależy od parametru `join_transaction_mode` sesji (albo `sessionmaker`). To parametr dodany w 2.0 i jego zrozumienie jest ważne głównie w testach.

| Wartość | Zachowanie | Kiedy używać |
|---|---|---|
| `"conditional_savepoint"` *(domyślne)* | Jeśli na połączeniu jest już SAVEPOINT → utwórz własny savepoint; jeśli nie ma → zachowaj się jak `"rollback_only"` | Kompatybilność wsteczna z 1.x; nie zalecane jako świadomy wybór |
| `"rollback_only"` | Sesja przejmuje kontrolę nad transakcją **tylko** dla `rollback()`; `commit()` sesji nie jest propagowany | Gdy chcesz mieć pewność, że `commit()` sesji nie zatwierdzi cudzej transakcji |
| `"create_savepoint"` | Sesja **zawsze** używa `Connection.begin_nested()` do realizacji swojego BEGIN/COMMIT/ROLLBACK | **Testy** — zewnętrzna transakcja pozostaje nietknięta |
| `"control_fully"` | Sesja przejmuje pełną kontrolę nad istniejącą transakcją | Gdy wiesz dokładnie, co robisz |

```python
# examples/14_09_join_transaction_mode.py
"""Wzorzec testowy: sesja 'żyje' w savepoincie zewnętrznej transakcji."""

import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import Session, sessionmaker

engine = create_engine("postgresql+psycopg://app:secret@localhost:5432/shop_test")

@pytest.fixture
def db_session():
    """Każdy test dostaje sesję, która po zakończeniu nie zostawia śladu."""
    connection = engine.connect()
    transaction = connection.begin()  # transakcja zewnętrzna, nigdy niecommitowana

    session = Session(
        bind=connection,
        join_transaction_mode="create_savepoint",
        expire_on_commit=False,
    )
    try:
        yield session
    finally:
        session.close()
        transaction.rollback()  # wszystko, co test zrobił, znika
        connection.close()
```

Ten wzorzec rozwiniemy w module 18 (testowanie). Tutaj ważne jest, żebyś zrozumiał **dlaczego** działa: `"create_savepoint"` sprawia, że `commit()` w kodzie testowanym zatwierdza tylko savepoint, a nie prawdziwą transakcję — więc dane nie wyciekają między testami.

### 4.6 Autobegin — gdzie naprawdę zaczyna się transakcja

> 🧠 **Dlaczego tak jest — „samo się zaczyna”, ale nie bez powodu.** W SQLAlchemy 2.0 sesja ma włączony **autobegin**: transakcja rozpoczyna się automatycznie przy pierwszej operacji, która wymaga połączenia (`execute`, `get`, `flush`). Nie musisz pisać `session.begin()`. To wygodne, ale ma konsekwencję: **moment, w którym transakcja się zaczyna, nie jest widoczny w kodzie**. A ponieważ im dłużej trwa transakcja, tym więcej blokad i tym większe ryzyko konfliktów — musisz o tym myśleć świadomie.

Możesz wyłączyć autobegin (`Session(autobegin=False)`) i wtedy **musisz** jawnie wywołać `session.begin()`. W kodzie produkcyjnym to rzadko potrzebne, ale w testach i w kodzie, w którym granice transakcji mają być widoczne jak na dłoni — bywa pomocne.

> ⚠️ **Pułapka — `session.execute(select(...))` zaczyna transakcję, o której zapomniałeś.** Bardzo częsty scenariusz: w handlerze HTTP robisz `session.get(...)`, potem idziesz do zewnętrznego API (2 sekundy), potem drugie zapytanie, potem `commit()`. Transakcja trwa 2,5 sekundy, choć nie ma po temu żadnego powodu. Jeśli w międzyczasie dotknęła wiersza przez `with_for_update()`, przez 2,5 sekundy blokujesz innych. O tym, jak temu zapobiegać, sekcja 9.

> 🧪 **Ćwiczenie — policz transakcje.** Napisz skrypt, który włącza `echo=True` i wykonuje: `session.get(Book, 1)`, `session.scalars(select(Book)).all()`, `session.commit()`, `session.get(Book, 1)`. Policz w logu wystąpienia `BEGIN` i `COMMIT`. Ile transakcji powstało? Które z nich są dla Ciebie zaskoczeniem?

---

## 5. Blokowanie pesymistyczne: `with_for_update()`

### 5.1 Analogia: klucz do toalety

Na stacji benzynowej jest jedna toaleta i jeden klucz na haczyku. Jeśli chcesz skorzystać, bierzesz klucz. Ktoś inny, kto przyjdzie po Tobie, **czeka** — nie może wejść, bo klucza nie ma. Kiedy wychodzisz i odwieszasz klucz, następna osoba wchodzi.

To jest **blokada pesymistyczna**: zakładasz z góry, że konflikt się zdarzy, więc go blokujesz. Kosztuje czekanie, ale daje pewność.

Blokada optymistyczna (sekcja 6) to odwrotność: nie blokujesz niczego, tylko **przy zapisie sprawdzasz, czy nikt Cię nie ubiegł**. Jeśli ubiegł — zgłaszasz błąd i próbujesz ponownie albo prosisz użytkownika o odświeżenie. Zakładasz, że konflikty są rzadkie.

### 5.2 `with_for_update()`

W SQLAlchemy blokadę wiersza zakłada metoda `with_for_update()` na zapytaniu `select()`:

```python
# examples/14_10_for_update.py
"""Blokada wiersza na czas transakcji (PostgreSQL)."""

from sqlalchemy import create_engine, select
from sqlalchemy.orm import Session

from models import Product  # Mapped[int] id, Mapped[str] name, Mapped[int] stock

engine = create_engine("postgresql+psycopg://app:secret@localhost:5432/shop")

stmt = select(Product).where(Product.id == 1).with_for_update()
print(stmt)
# SELECT product.id, product.name, product.stock
# FROM product
# WHERE product.id = %(id_1)s
# FOR UPDATE

with Session(engine) as session:
    product = session.scalars(stmt).one()  # wiersz zablokowany DO KOŃCA TRANSAKCJI
    product.stock -= 1
    session.commit()  # dopiero tutaj blokada znika
```

> 🔬 **Pod maską.** `with_for_update()` dodaje do `SELECT` klauzulę `FOR UPDATE`. Blokada **nie** jest związana z metodą `with_for_update()` — jest związana z **transakcją**. Wiersz pozostaje zablokowany aż do `COMMIT` albo `ROLLBACK`. To znaczy, że trzymanie obiektu „w ręce” po `SELECT ... FOR UPDATE` przez 10 sekund to 10 sekund, przez które nikt inny nie tknie tego wiersza. Bardzo łatwo o tym zapomnieć.

### 5.3 Warianty blokady

| Wariant | SQL | Znaczenie |
|---|---|---|
| `with_for_update()` | `FOR UPDATE` | Zablokuj wiersz do zapisu; inni czekają |
| `with_for_update(read=True)` | `FOR SHARE` | Zablokuj do odczytu; inni mogą czytać, ale nie zapisywać |
| `with_for_update(nowait=True)` | `FOR UPDATE NOWAIT` | Nie czekaj — jeśli zablokowany, natychmiast rzuć błąd |
| `with_for_update(skip_locked=True)` | `FOR UPDATE SKIP LOCKED` | Pomiń zablokowane wiersze i weź następny wolny |
| `with_for_update(of=Product)` | `FOR UPDATE OF product` | W zapytaniu z JOIN-em zablokuj wiersze tylko z tej tabeli |
| `with_for_update(key_share=True)` | `FOR NO KEY UPDATE` | PostgreSQL: blokada słabsza niż `FOR UPDATE`, nie blokuje kluczy obcych |

`skip_locked` zasługuje na szczególną uwagę, bo rozwiązuje klasyczny problem kolejek: „pięciu workerów, tysiąc zadań, żaden nie może przetwarzać tego samego zadania”.

```python
# examples/14_11_queue_skip_locked.py
"""Wzorzec kolejki: każdy worker bierze inne zadanie, bez czekania."""

from sqlalchemy import create_engine, select
from sqlalchemy.orm import Session

from models import Job  # Mapped[int] id, Mapped[str] status

engine = create_engine("postgresql+psycopg://app:secret@localhost:5432/jobs")


def claim_one_job(session: Session) -> Job | None:
    """Zabierz jedno wolne zadanie. Jeśli wszystkie zajęte — zwróć None."""
    stmt = (
        select(Job)
        .where(Job.status == "pending")
        .order_by(Job.id)
        .limit(1)
        .with_for_update(skip_locked=True)  # <- kluczowy fragment
    )
    job = session.scalars(stmt).first()
    if job is None:
        return None
    job.status = "running"
    session.commit()
    return job
```

> 💡 **Analogia — `skip_locked` to „nie stój w kolejce, weź inny numerek”.** Zwykłe `FOR UPDATE` to czekanie pod drzwiami, aż ktoś wyjdzie. `SKIP LOCKED` to rozejrzenie się po sali i zajęcie pierwszego wolnego krzesła. Dla zadań, które są niezależne od siebie (przetwórz e-mail, wygeneruj PDF), to idealne. Dla operacji, które muszą zachować kolejność — wręcz przeciwskazane.

> ⚠️ **Pułapka — `FOR UPDATE` na SQLite nie działa.** SQLite nie ma blokad wierszowych. Dialekt SQLAlchemy **nie wygeneruje** klauzuli `FOR UPDATE` (jest pomijana, bo baza jej nie rozumie). Twój kod się uruchomi, ale nie uzyska żadnej ochrony przed wyścigiem. Jeśli piszesz testy na SQLite dla kodu używającego `with_for_update()`, testy **nie sprawdzą** tego, co myślisz, że sprawdzają. To jeden z najczęstszych fałszywych sygnałów „zielonych testów”. Rozwiązanie: testy współbieżności uruchamiaj na PostgreSQL (moduł 18, `testcontainers`).

> ⚠️ **Pułapka — `nowait` i `skip_locked` nie są dostępne wszędzie.** PostgreSQL i MySQL 8 obsługują oba. Starsze bazy — niekoniecznie. Sprawdzaj dokumentację dialektu.

### 5.4 Przykład: zabierz jedną sztukę z magazynu, bez wyścigu

```python
# examples/14_12_take_one_stock.py
"""Zabranie sztuki z magazynu z blokadą pesymistyczną."""

from sqlalchemy import create_engine, select
from sqlalchemy.orm import Session

from models import Product

engine = create_engine("postgresql+psycopg://app:secret@localhost:5432/shop")


def take_one_stock(session: Session, product_id: int) -> bool:
    """Zwraca True, jeśli udało się zabrać sztukę. Wymaga PostgreSQL."""
    product = session.scalars(
        select(Product).where(Product.id == product_id).with_for_update()
    ).one()

    if product.stock <= 0:
        session.rollback()  # zwolnij blokadę natychmiast
        return False

    product.stock -= 1
    session.commit()
    return True
```

Zwróć uwagę na `session.rollback()` w gałęzi „brak towaru”. To **nie** jest zbędne: dzięki niemu blokada zwalnia się od razu, a nie po zamknięciu sesji. Bez tego wiersz pozostaje zablokowany do momentu wyjścia z bloku `with`, co przy dużej liczbie nieudanych prób potrafi zablokować cały system.

### 5.5 `SELECT ... FOR UPDATE` kontra `UPDATE ... RETURNING`

Czy musisz najpierw czytać? Często nie. Jeśli chcesz tylko „zabierz jedną sztukę, jeśli jest”, możesz to zrobić **jednym** poleceniem:

```python
# examples/14_13_atomic_update.py
"""Atomowa aktualizacja warunkowa — bez SELECT, bez blokady jawnej."""

from sqlalchemy import create_engine, update
from sqlalchemy.orm import Session

from models import Product

engine = create_engine("postgresql+psycopg://app:secret@localhost:5432/shop")


def take_one_stock_atomic(session: Session, product_id: int) -> bool:
    """Jedno polecenie: zmień, jeśli warunek jest spełniony. Zwróć, czy się udało."""
    stmt = (
        update(Product)
        .where(Product.id == product_id, Product.stock > 0)
        .values(stock=Product.stock - 1)
        .returning(Product.stock)
    )
    row = session.execute(stmt).first()
    session.commit()
    return row is not None  # None => warunek nie był spełniony, nikt nic nie zmienił
```

> 🔬 **Pod maską.**
>
> ```sql
> UPDATE product
> SET stock = product.stock - 1
> WHERE product.id = %(id_1)s AND product.stock > 0
> RETURNING product.stock
> ```
>
> Dlaczego to jest bezpieczne bez `FOR UPDATE`? Bo `UPDATE` w PostgreSQL **sam zakłada blokadę** na aktualizowanym wierszu. Dwie równoległe transakcje nie wykonają tego `UPDATE` jednocześnie: druga poczeka, a po odblokowaniu PostgreSQL **ponownie oceni warunek `WHERE`** na świeżej wersji wiersza (ten sam *re-check*, o którym mówiliśmy w sekcji 2.5). Jeśli `stock` spadło do 0, warunek `stock > 0` nie będzie spełniony, `UPDATE` nie zmieni żadnego wiersza, `RETURNING` nic nie zwróci — i dostaniesz `None`. Dokładnie to chciałeś.

| Kryterium | `SELECT ... FOR UPDATE` + zmiana w Pythonie | `UPDATE ... WHERE warunek RETURNING` |
|---|---|---|
| Liczba rund do bazy | 2 (SELECT, UPDATE) | 1 |
| Wymaga odczytu danych do aplikacji | Tak | Nie |
| Możliwość uruchomienia logiki biznesowej przed zapisem | Tak | Nie (tylko warunek w SQL) |
| Odporność na lost update | Tak (blokada) | Tak (re-check) |
| Zgodność ze SQLite | Nie (brak `FOR UPDATE`) | Tak (SQLite obsługuje `RETURNING` od 3.35) |
| Blokada trzymana długo | Ryzyko, jeśli logika jest wolna | Minimalna — blokada tylko na czas `UPDATE` |

**Zasada praktyczna:** jeśli operację da się wyrazić jako warunkowy `UPDATE`, wyraź ją jako warunkowy `UPDATE`. Sięgaj po `SELECT ... FOR UPDATE` dopiero wtedy, gdy musisz najpierw **odczytać dane**, podjąć decyzję w Pythonie i na tej podstawie zapisać.

> 🧪 **Ćwiczenie — przepisz na jedno polecenie.** Operacja: „oznacz zamówienie jako opłacone, jeśli jest w statusie `pending` i jego kwota się zgadza”. Napisz wersję z `SELECT` + sprawdzeniem w Pythonie + `UPDATE` oraz wersję z jednym `UPDATE ... WHERE status='pending' AND amount=:amount RETURNING id`. Porównaj liczbę rund i wygenerowany SQL.

---

## 6. Blokowanie optymistyczne: kolumna wersji

### 6.1 Idea

> 💡 **Analogia — numer wersji dokumentu.** Wysyłasz komuś umowę w wersji 3. Kolega ją edytuje, Ty też. Przy zapisie system sprawdza: „czy plik na serwerze to nadal wersja 3?”. Jeśli ktoś w międzyczasie zapisał wersję 4 — Twój zapis jest odrzucany, bo pracowałeś na nieaktualnych danych. Nie blokowałeś nikogo, ale nie nadpisałeś cudzej pracy.

SQLAlchemy implementuje to natywnie przez `version_id_col` w `__mapper_args__`.

```python
# examples/14_14_optimistic.py
"""Blokada optymistyczna: kolumna wersji pilnowana przez SQLAlchemy."""

from sqlalchemy import create_engine, Integer, String
from sqlalchemy.orm import DeclarativeBase, Mapped, Session, mapped_column
from sqlalchemy.exc import StaleDataError


class Base(DeclarativeBase):
    pass


class Product(Base):
    __tablename__ = "product"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    stock: Mapped[int]

    # SQLAlchemy sam inkrementuje tę kolumnę przy każdym UPDATE
    # i dokłada ją do warunku WHERE.
    version_id: Mapped[int] = mapped_column(Integer, nullable=False, default=1)

    __mapper_args__ = {"version_id_col": version_id}


engine = create_engine("sqlite:///optimistic_demo.db")
Base.metadata.create_all(engine)

# Dwie sesje czytają TEN SAM wiersz.
session_a = Session(engine)
session_b = Session(engine)

product_a = session_a.get(Product, 1)
product_b = session_b.get(Product, 1)
print(f"wersja widziana przez A: {product_a.version_id}")
print(f"wersja widziana przez B: {product_b.version_id}")

# A wygrywa wyścig.
product_a.stock -= 1
session_a.commit()
print("A zapisała, nowa wersja:", product_a.version_id)

# B próbuje zapisać na podstawie nieaktualnych danych.
product_b.stock -= 1
try:
    session_b.commit()
except StaleDataError as exc:
    print(f"B odrzucona: {type(exc).__name__}: {exc}")
    session_b.rollback()  # OBOWIĄZKOWE
```

> 🔬 **Pod maską.** Emitowany SQL to:
>
> ```sql
> UPDATE product
> SET stock = %(stock)s, version_id = %(version_id)s
> WHERE product.id = %(id)s AND product.version_id = %(version_id_1)s
> ```
>
> SQLAlchemy wysyła **starą** wartość `version_id` w klauzuli `WHERE` i nową w `SET`. Następnie sprawdza `rowcount`. Jeśli baza zmieniła 0 wierszy — znaczy to, że ktoś inny zdążył zmienić ten wiersz i wersja już się nie zgadza. Wtedy leci `StaleDataError`. Zwróć uwagę: SQLAlchemy nie wie, **kto** zmienił wiersz ani **jak** — wie tylko, że wersja się rozjechała.

### 6.2 Co robić po `StaleDataError`

Trzy strategie, w kolejności od najprostszej:

1. **Powiadom użytkownika.** W aplikacji webowej: „Ktoś inny zmodyfikował ten rekord. Odśwież stronę i spróbuj ponownie.” To jest całkowicie akceptowalne i często najlepsze rozwiązanie przy edycji formularzy.
2. **Odśwież i ponów automatycznie.** `session.rollback()`, `session.refresh(obj)`, ponowne zastosowanie zmian. Działa tylko wtedy, gdy zmiany nie kolidują logicznie.
3. **Scal (merge).** Wybierz, które pola wygrywają. Wymaga wiedzy biznesowej — nie da się tego zrobić „mechanicznie”.

> ⚠️ **Pułapka — `StaleDataError` zostawia sesję w złym stanie.** Po tym wyjątku **musisz** wykonać `session.rollback()`. Bez tego sesja jest w stanie „pending rollback” i każde kolejne użycie rzuci `PendingRollbackError`. To dotyczy zresztą **każdego** wyjątku rzuconego w trakcie `flush` lub `commit` — nie tylko `StaleDataError`. Zapamiętaj regułę: **wyjątek z `commit()` ⇒ natychmiastowy `rollback()`**.

> ⚠️ **Pułapka — `expire_on_commit=False` plus konflikt = nieaktualne dane w pamięci.** Jeśli używasz `expire_on_commit=False` (typowe w FastAPI), obiekty po commicie **nie są odświeżane**. W scenariuszu: próba zapisu → `StaleDataError` → `rollback()` → ponowna próba na tym samym obiekcie — obiekt nadal trzyma wartości, które uważa za aktualne, choć w bazie są inne. Rozwiązanie: po `rollback()` wywołaj `session.refresh(obj)` albo — czyściej — **odrzuć obiekt i pobierz go na nowo**.

### 6.3 Wersjonowanie przez `updated_at`

Alternatywa dla osobnej kolumny `version_id`: użycie kolumny `updated_at` jako znacznika wersji.

```python
# examples/14_15_updated_at_version.py
"""Wersjonowanie czasem modyfikacji — bez dodatkowej kolumny."""

from datetime import datetime

from sqlalchemy import DateTime, func
from sqlalchemy.orm import Mapped, mapped_column

from base import Base  # Twoja klasa bazowa z poprzednich modułów


class Document(Base):
    __tablename__ = "document"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str]
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
        onupdate=func.now(),
    )

    __mapper_args__ = {"version_id_col": updated_at}
```

> ⚠️ **Pułapka — rozdzielczość zegara.** `updated_at` działa jako znacznik wersji tylko wtedy, gdy ma wystarczającą rozdzielczość. Jeśli dwa zapisy nastąpią w tej samej mikrosekundzie (na szybkim sprzęcie — realne), kolizja nie zostanie wykryta. W PostgreSQL `timestamptz` ma rozdzielczość mikrosekundową, więc ryzyko jest małe, ale nie zerowe. Osobna kolumna `Integer` z licznikiem jest **deterministyczna** i to jest jej główna przewaga.

| Kryterium | `version_id_col` (Integer) | `version_id_col` (`updated_at`) |
|---|---|---|
| Determinizm | Pełny | Zależny od rozdzielczości zegara |
| Informacja dla człowieka | „wersja 7” | „ostatnia zmiana 14:32:07” |
| Dodatkowa kolumna w bazie | Tak | Nie (jeśli i tak masz `updated_at`) |
| Łatwość diagnozy konfliktu | Trudniejsza | Łatwiejsza (widać, kiedy zmieniono) |

### 6.4 Pesymistyczna czy optymistyczna?

| Sytuacja | Rekomendacja |
|---|---|
| Edycja formularza przez człowieka, konflikty rzadkie | **Optymistyczna** — brak blokad, brak czekania |
| Krótka, krytyczna operacja, wysoka konkurencja o ten sam wiersz | **Pesymistyczna** (`FOR UPDATE`) albo atomowy warunkowy `UPDATE` |
| Licznik, stan magazynu, saldo konta | **Atomowy warunkowy `UPDATE`** (jedna runda, brak blokady w aplikacji) |
| Kolejka zadań | `FOR UPDATE SKIP LOCKED` |
| Operacja wymagająca odczytu + decyzji w Pythonie + zapisu | **Pesymistyczna** (`FOR UPDATE`) |
| Konflikt powinien być widoczny dla użytkownika | **Optymistyczna** — `StaleDataError` to sygnał, nie awaria |

> 🧠 **Dlaczego tak jest — „blokada to zasób, nie zabezpieczenie”.** Blokada pesymistyczna nie chroni danych lepiej niż optymistyczna — obie dają ten sam poziom poprawności. Różnica jest w **koszcie**: pesymistyczna kosztuje czekanie i zużywa połączenia z puli; optymistyczna kosztuje powtórzenia i wymaga obsługi konfliktu w kodzie. Wybierasz tańszą dla swojego profilu obciążenia.

---

## 7. Wyścigi i idempotencja

### 7.1 Podwójne kliknięcie „Zapłać”

> 💡 **Analogia — przycisk windy.** Naciskasz przycisk, winda nie przyjeżdża w pół sekundy, naciskasz jeszcze trzy razy. Winda przyjedzie raz, ale jeśli Twoja aplikacja nie jest na to przygotowana, może naliczyć cztery opłaty.

To najczęstszy scenariusz produkcji: użytkownik klika „Zapłać”, sieć się zawiesza, on klika ponownie. Bez zabezpieczenia powstają dwie płatności. Rozwiązanie nazywa się **kluczem idempotencji (idempotency key)**: klient generuje unikalny identyfikator operacji, a serwer gwarantuje, że ten sam klucz wykona się tylko raz.

```python
# examples/14_16_idempotency.py
"""Idempotentna płatność: klucz unikalny w bazie jako ostateczna linia obrony."""

import uuid
from decimal import Decimal

from sqlalchemy import create_engine, String, Numeric, select
from sqlalchemy.dialects.postgresql import insert as pg_insert
from sqlalchemy.orm import DeclarativeBase, Mapped, Session, mapped_column


class Base(DeclarativeBase):
    pass


class Payment(Base):
    __tablename__ = "payment"

    id: Mapped[int] = mapped_column(primary_key=True)
    idempotency_key: Mapped[str] = mapped_column(String(64), unique=True, index=True)
    amount: Mapped[Decimal] = mapped_column(Numeric(12, 2))
    status: Mapped[str] = mapped_column(String(20), default="created")


engine = create_engine("postgresql+psycopg://app:secret@localhost:5432/shop")
Base.metadata.create_all(engine)


def create_payment(session: Session, key: str, amount: Decimal) -> int:
    """Utwórz płatność raz. Powtórne wywołanie z tym samym kluczem zwraca istniejącą."""
    stmt = (
        pg_insert(Payment)
        .values(idempotency_key=key, amount=amount, status="created")
        .on_conflict_do_nothing(index_elements=["idempotency_key"])
        .returning(Payment.id)
    )
    new_id = session.execute(stmt).scalar_one_or_none()

    if new_id is not None:
        session.commit()
        return new_id

    # Konflikt: płatność już istnieje — zwróć jej identyfikator.
    session.rollback()
    existing_id = session.scalars(
        select(Payment.id).where(Payment.idempotency_key == key)
    ).one()
    return existing_id
```

> 🔬 **Pod maską.**
>
> ```sql
> INSERT INTO payment (idempotency_key, amount, status)
> VALUES (%(key)s, %(amount)s, %(status)s)
> ON CONFLICT (idempotency_key) DO NOTHING
> RETURNING payment.id
> ```
>
> Kluczowy szczegół: **unikalny indeks na `idempotency_key` jest gwarancją, nie walidacja w Pythonie**. Możesz sprawdzać w Pythonie `SELECT` przed `INSERT` — ale między `SELECT` a `INSERT` zawsze istnieje okno czasowe, w którym ktoś inny zdąży wstawić ten sam wiersz. Tylko baza potrafi zrobić sprawdzenie i wstawienie atomowo.

W SQLite odpowiednik jest niemal identyczny, tylko import inny:

```python
# examples/14_16b_idempotency_sqlite.py
from sqlalchemy.dialects.sqlite import insert as sqlite_insert

stmt = (
    sqlite_insert(Payment)
    .values(idempotency_key=key, amount=amount, status="created")
    .on_conflict_do_nothing(index_elements=["idempotency_key"])
    .returning(Payment.id)
)
```

> ⚠️ **Pułapka — `ON CONFLICT DO UPDATE` przy idempotencji to zwykle błąd.** Jeśli zamienisz `DO NOTHING` na `DO UPDATE`, powtórne wywołanie **zmodyfikuje** istniejący rekord. Dla płatności to katastrofa: drugie kliknięcie mogłoby nadpisać kwotę lub status. Do idempotencji używa się `DO NOTHING` (albo `DO UPDATE` **tylko** na polach technicznych, jak `last_seen_at`).

> 🆕 **SQLAlchemy 2.1 — wielokrotne klauzule `ON CONFLICT` w SQLite.** W 2.1 dodano obsługę wielu klauzul `ON CONFLICT` dla dialektu SQLite, co pozwala modelować bardziej złożone scenariusze konfliktów w jednym poleceniu. W 2.0 musisz się ograniczyć do jednej klauzuli na `INSERT`.

### 7.2 Wzorzec „reserve → confirm”

Gdy operacja jest długa (wywołanie bramki płatniczej, rezerwacja w systemie zewnętrznym), trzymanie transakcji przez cały czas jej trwania jest niedopuszczalne. Wzorzec:

1. **Reserve.** W krótkiej transakcji: sprawdź dostępność i utwórz rekord rezerwacji ze statusem `reserved` i czasem wygaśnięcia. Atomowo, jednym `UPDATE ... WHERE`.
2. **Wywołaj system zewnętrzny.** Poza transakcją. Trwa sekundy.
3. **Confirm.** W kolejnej krótkiej transakcji: zmień status na `confirmed`.
4. **Sprzątanie.** Zadanie w tle usuwa rezerwacje wygasłe (`status='reserved' AND expires_at < now()`).

```python
# examples/14_17_reserve_confirm.py
"""Reserve -> confirm: transakcja nigdy nie trwa dłużej niż kilka milisekund."""

from datetime import datetime, timedelta, timezone

from sqlalchemy import create_engine, select, update
from sqlalchemy.orm import Session

from models import Reservation, Seat

engine = create_engine("postgresql+psycopg://app:secret@localhost:5432/cinema")

RESERVATION_TTL = timedelta(minutes=15)


def reserve_seat(session: Session, seat_id: int, user_id: int) -> int | None:
    """Krok 1: atomowa rezerwacja. Zwraca id rezerwacji lub None."""
    now = datetime.now(timezone.utc)

    # Atomowo: zajmij miejsce tylko jeśli jest wolne LUB rezerwacja wygasła.
    stmt = (
        update(Seat)
        .where(
            Seat.id == seat_id,
            (Seat.status == "free")
            | ((Seat.status == "reserved") & (Seat.reserved_until < now)),
        )
        .values(status="reserved", reserved_until=now + RESERVATION_TTL)
        .returning(Seat.id)
    )
    if session.execute(stmt).first() is None:
        session.rollback()
        return None

    reservation = Reservation(seat_id=seat_id, user_id=user_id, status="reserved")
    session.add(reservation)
    session.commit()
    return reservation.id


def confirm_reservation(session: Session, reservation_id: int, paid: bool) -> None:
    """Krok 3: potwierdzenie po udanej płatności — osobna, krótka transakcja."""
    reservation = session.get(Reservation, reservation_id)
    if reservation is None:
        raise LookupError("nie ma takiej rezerwacji")
    if reservation.status != "reserved":
        return  # już potwierdzona — idempotencja

    reservation.status = "confirmed" if paid else "cancelled"
    seat = session.get(Seat, reservation.seat_id)
    seat.status = "taken" if paid else "free"
    session.commit()
```

> 🧠 **Dlaczego tak jest — „transakcja to nie miejsce na czekanie”.** Każda sekunda transakcji to sekunda trzymanej blokady i sekunda zajętego połączenia z puli. Wywołanie HTTP do bramki płatniczej w środku transakcji to gwarantowany problem przy obciążeniu: pula połączeń się skończy, inni użytkownicy będą czekać, a baza zgłosi `idle in transaction`. Wzorzec reserve→confirm przenosi „czekanie” **poza** transakcję, a stan pośredni (`reserved`) uczynia widocznym i naprawialnym.

> ⚠️ **Pułapka — rezerwacja bez czasu wygaśnięcia to wyciek.** Jeśli użytkownik zamknął przeglądarkę po rezerwacji, a nie ma mechanizmu wygasania, miejsce jest zablokowane na zawsze. Zawsze dodawaj `expires_at`/`reserved_until` i zadanie sprzątające. To ten sam problem co `idle in transaction`, tylko na poziomie logiki biznesowej.

---

## 8. Deadlocki

### 8.1 Jak powstaje deadlock

> 💡 **Analogia — wąskie przejście w dwóch kierunkach.** Ania wchodzi w zaułek z lewej, Bartek z prawej. Oboje sięgają po drabinę stojącą pośrodku. Ania trzyma drabinę i czeka, aż Bartek się cofnie. Bartek trzyma drabinę i czeka, aż Ania się cofnie. Nikt się nie ruszy, dopóki ktoś nie ustąpi.

W bazie wygląda to tak:

```text
Transakcja A:                          Transakcja B:
BEGIN;                                 BEGIN;
UPDATE account SET ... WHERE id = 1;   UPDATE account SET ... WHERE id = 2;
-- blokuje wiersz 1                     -- blokuje wiersz 2

UPDATE account SET ... WHERE id = 2;   UPDATE account SET ... WHERE id = 1;
-- CZEKA na B                          -- CZEKA na A  ==> DEADLOCK
```

Baza wykrywa cykl oczekiwań (PostgreSQL robi to po `deadlock_timeout`, domyślnie 1 s) i **zabija jedną z transakcji**, zgłaszając błąd `deadlock detected` (SQLSTATE `40P01`). Druga może kontynuować.

**Kluczowa obserwacja:** deadlock to nie „awaria bazy”. To normalna, oczekiwana sytuacja w systemie ze współbieżnością. Baza rozwiązuje ją deterministycznie, a Twoim zadaniem jest **poprawnie zareagować** — czyli ponowić operację.

### 8.2 Jak minimalizować deadlocki

| Technika | Dlaczego działa |
|---|---|
| **Stała kolejność blokowania** | Jeśli wszystkie transakcje dotykają wierszy w tej samej kolejności (np. rosnące `id`), cykl oczekiwań nie może powstać |
| **Krótkie transakcje** | Mniejsze okno czasowe na kolizję |
| **`FOR UPDATE NOWAIT`** | Zamiast czekać — natychmiastowy błąd, obsłużony przez retry |
| **Blokowanie zbiorcze zamiast punktowego** | Jeden `UPDATE ... WHERE id IN (...)` zamiast pętli po wierszach |
| **Unikanie indeksów bez potrzeby** | Zapis do tabeli z wieloma indeksami blokuje więcej zasobów w nieoczywistej kolejności |
| **`READ COMMITTED` zamiast `SERIALIZABLE`** | Mniej blokad, mniej okazji do cyklu (ale inne gwarancje!) |

**Stała kolejność** to najskuteczniejsza i najczęściej pomijana technika. Wystarczy sortować identyfikatory przed aktualizacją:

```python
# examples/14_18_consistent_lock_order.py
"""Blokowanie w stałej kolejności — deadlock staje się niemożliwy."""

from sqlalchemy import select
from sqlalchemy.orm import Session

from models import Account


def transfer(session: Session, from_id: int, to_id: int, amount: int) -> None:
    """Przelew z blokowaniem kont w kolejności rosnącego id."""
    # Kolejność jest ZAWSZE taka sama, niezależnie od kierunku przelewu.
    first_id, second_id = sorted((from_id, to_id))

    locked = session.scalars(
        select(Account).where(Account.id.in_([first_id, second_id])).order_by(Account.id).with_for_update()
    ).all()
    by_id = {account.id: account for account in locked}

    src, dst = by_id[from_id], by_id[to_id]
    if src.balance < amount:
        session.rollback()
        raise ValueError("brak środków")

    src.balance -= amount
    dst.balance += amount
    session.commit()
```

> 🔬 **Pod maską.**
>
> ```sql
> SELECT account.id, account.balance
> FROM account
> WHERE account.id IN (%(id_1)s, %(id_2)s)
> ORDER BY account.id
> FOR UPDATE
> ```
>
> `ORDER BY account.id` w połączeniu z `FOR UPDATE` sprawia, że PostgreSQL zakłada blokady w kolejności rosnącego `id` — zawsze tak samo, niezależnie od tego, kto komu przelewa. Cykl nie może powstać.

> ⚠️ **Pułapka — sortowanie w Pythonie nie wystarczy.** Jeśli napiszesz `sorted()` na liście `id`, ale potem wykonasz `session.get(Account, id)` w pętli **po nieposortowanej** liście, kolejność blokad znowu będzie losowa. Sortuj tę listę, której faktycznie używasz do blokowania — albo lepiej, zrób jeden `SELECT ... WHERE id IN (...) ORDER BY id FOR UPDATE` jak wyżej.

### 8.3 Gotowy dekorator `@retry_on_deadlock`

Deadlocki i błędy serializacji (`40001`, `40P01`) są **przejściowe** — po ponowieniu operacja zwykle się udaje. Oto kompletny, produkcyjnie użyteczny dekorator:

```python
# examples/14_19_retry.py
"""Retry dla błędów przejściowych: deadlock, serialization failure, locked DB."""

from __future__ import annotations

import functools
import logging
import random
import time
from collections.abc import Callable
from typing import Any, TypeVar

from sqlalchemy.exc import DBAPIError, OperationalError

logger = logging.getLogger(__name__)

F = TypeVar("F", bound=Callable[..., Any])

# PostgreSQL: 40P01 = deadlock_detected, 40001 = serialization_failure
RETRYABLE_PG_CODES = frozenset({"40P01", "40001"})

# SQLite nie ma kodów SQLSTATE — rozpoznajemy po treści komunikatu.
RETRYABLE_MESSAGES = ("deadlock detected", "database is locked", "could not serialize access")


def is_retryable(exc: DBAPIError) -> bool:
    """Czy ten błąd warto ponowić?"""
    orig = exc.orig
    pgcode = getattr(orig, "pgcode", None)  # psycopg / psycopg2
    if pgcode in RETRYABLE_PG_CODES:
        return True
    message = str(orig).lower()
    return any(fragment in message for message in RETRYABLE_MESSAGES)


def retry_on_deadlock(
    max_attempts: int = 5,
    base_delay: float = 0.05,
    max_delay: float = 2.0,
) -> Callable[[F], F]:
    """Ponów funkcję, jeśli rzuci przejściowym błędem bazy.

    WAŻNE: funkcja musi sama tworzyć świeżą sesję przy każdym wywołaniu.
    Po błędzie sesja jest w stanie wymagającym rollbacku i nie nadaje się do użycia.
    """

    def decorator(func: F) -> F:
        @functools.wraps(func)
        def wrapper(*args: Any, **kwargs: Any) -> Any:
            last_exc: Exception | None = None
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except (OperationalError, DBAPIError) as exc:
                    if not is_retryable(exc) or attempt == max_attempts:
                        raise
                    last_exc = exc
                    # Exponential backoff + jitter (rozrzut), żeby nie zsynchronizować retry.
                    delay = min(base_delay * (2 ** (attempt - 1)), max_delay)
                    delay *= 0.5 + random.random()
                    logger.warning(
                        "Próba %d/%d nieudana (%s). Ponawiam za %.3f s.",
                        attempt,
                        max_attempts,
                        type(exc).__name__,
                        delay,
                    )
                    time.sleep(delay)
            raise RuntimeError("nieosiągalne") from last_exc

        return wrapper  # type: ignore[return-value]

    return decorator
```

Użycie — zwróć uwagę, że **sesja jest tworzona wewnątrz funkcji**, a nie przekazywana z zewnątrz:

```python
# examples/14_19b_retry_usage.py
from sqlalchemy.orm import Session, sessionmaker

from examples_retry import retry_on_deadlock  # dekorator powyżej

SessionLocal = sessionmaker(bind=engine, expire_on_commit=False)


@retry_on_deadlock(max_attempts=5)
def transfer_money(from_id: int, to_id: int, amount: int) -> None:
    """Każda próba dostaje ŚWIEŻĄ sesję — to jest warunek poprawności."""
    with SessionLocal() as session:
        try:
            # ... logika przelewu, włącznie z commit() ...
            session.commit()
        except Exception:
            session.rollback()
            raise  # dekorator zdecyduje, czy ponowić
```

> ⚠️ **Pułapka — retry bez `rollback` to katastrofa.** To najczęstszy błąd w implementacjach retry. Po błędzie bazy sesja jest „zatruta”: w PostgreSQL transakcja jest w stanie *aborted* i każde kolejne polecenie kończy się `current transaction is aborted`. Jeśli po prostu ponowisz operację na tej samej sesji, dostaniesz serię mylących błędów zamiast naprawy. Dwa rozwiązania: (a) `rollback()` przed ponowieniem, (b) — lepiej — **świeża sesja na każdą próbę**, jak w przykładzie powyżej.

> ⚠️ **Pułapka — retry bez limitu to pętla nieskończona.** Zawsze ustawiaj `max_attempts` i zawsze dodawaj **jitter** (losowy rozrzut opóźnienia). Bez jittera sto równoległych transakcji ponowi próbę w tej samej milisekundzie i znowu się zderzy. To klasyczny efekt *thundering herd*.

> 🧪 **Ćwiczenie — wywołaj deadlock celowo.** Napisz dwa skrypty, które w odwrotnej kolejności aktualizują dwa wiersze `account` w PostgreSQL (z `sleep` w środku, żeby wydłużyć okno). Uruchom je równocześnie. Zobacz komunikat błędu i kod SQLSTATE. Następnie napraw kod, sortując `id`, i uruchom ponownie — deadlock nie powinien wystąpić.

---

## 9. Długie transakcje — najczęstszy zabójca wydajności

### 9.1 `idle in transaction`

Gdy aplikacja rozpoczyna transakcję i przestaje wysyłać polecenia (bo np. wywołuje zewnętrzne API, albo po prostu zapomniała o `commit`), połączenie wchodzi w stan **`idle in transaction`**. Baza widzi otwartą transakcję, której nikt nie zamyka. Konsekwencje:

- blokady założone w tej transakcji są nadal aktywne,
- PostgreSQL nie może usunąć „martwych” wersji wierszy, które ta transakcja mogłaby jeszcze zobaczyć — narasta **bloat**,
- połączenie z puli jest zajęte i niedostępne dla innych.

Diagnoza w PostgreSQL:

```sql
SELECT pid, usename, state, state_change, now() - state_change AS idle_for, query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY idle_for DESC;
```

### 9.2 VACUUM i bloat — dlaczego to ma znaczenie dla Ciebie

PostgreSQL realizuje MVCC (sekcja 11): `UPDATE` nie modyfikuje wiersza w miejscu, tylko tworzy **nową wersję** wiersza i oznacza starą jako nieaktualną. Stare wersje są usuwane przez proces `VACUUM`. Ale `VACUUM` nie może usunąć wersji, która jest jeszcze potrzebna jakiejkolwiek otwartej transakcji.

Długa transakcja (albo kilka sesji `idle in transaction`) blokuje `VACUUM`. Tabela puchnie: zajmuje więcej miejsca, skanowanie trwa dłużej, indeksy rosną. Efekt: aplikacja, która „działała szybko przez rok”, nagle zaczyna odpowiadać w 3 sekundy — a przyczyną jest jedna sesja otwarta od rana.

> 🧠 **Dlaczego tak jest — „nieczytelny pokój blokuje sprzątanie”.** Sprzątacz nie może wyrzucić gazety, jeśli ktoś właśnie ją czyta. W PostgreSQL każda otwarta transakcja to „ktoś, kto czyta”. Dopóki nie skończy, stare wersje muszą zostać na miejscu.

### 9.3 Timeouts — pas bezpieczeństwa

Nigdy nie licz na to, że Twoja aplikacja nie popełni błędu. Ustaw limity po stronie bazy:

```python
# examples/14_20_timeouts.py
"""Timeouty po stronie połączenia — tanie zabezpieczenie przed zawieszeniem."""

from sqlalchemy import create_engine, event
from sqlalchemy.engine import Connection

engine = create_engine("postgresql+psycopg://app:secret@localhost:5432/shop")

# Twardy limit na każde polecenie: 5 sekund.
# Po przekroczeniu PostgreSQL anuluje zapytanie i zwróci błąd.
engine = engine.execution_options(
    connect_args={
        "options": "-c statement_timeout=5000 -c lock_timeout=2000 -c idle_in_transaction_session_timeout=30000"
    }
)
```

| Parametr | Co robi | Sensowna wartość |
|---|---|---|
| `statement_timeout` | Maksymalny czas jednego polecenia | 5–30 s dla API; dłużej dla zadań wsadowych |
| `lock_timeout` | Maksymalny czas czekania na blokadę | 1–3 s — lepiej dostać błąd niż wisieć |
| `idle_in_transaction_session_timeout` | Zabija sesje bezczynne w transakcji | 30–60 s |
| `transaction_timeout` *(PG 17+)* | Maksymalny czas całej transakcji | Zależnie od operacji |

> ⚠️ **Pułapka — `idle_in_transaction_session_timeout` zabija połączenie, nie tylko transakcję.** Po przekroczeniu limitu PostgreSQL **zamyka sesję**. Twój kod dostanie `OperationalError` przy następnej próbie użycia. To jest zamierzone zachowanie — ale wymaga, żeby kod umiał się po tym pozbierać (rollback, nowe połączenie z puli). Bez tego „zabezpieczenie” zamienia powolny problem w głośną awarię.

> ⚠️ **Pułapka — sesja otwarta na czas całego requestu HTTP.** W wielu frameworkach (FastAPI, Flask) sesja jest tworzona w zależności i zamykana po odpowiedzi. Brzmi dobrze — dopóki request nie zawiera wywołania zewnętrznego API, generowania PDF-a albo wysyłki e-maila. Wtedy transakcja trwa sekundy. Rozwiązanie: (a) dziel request na kilka krótkich transakcji, (b) używaj `session.commit()`/`session.rollback()` po każdej logicznej jednostce, (c) rozważ sesję bez autobegin i jawne `begin()`, żebyś **widział** granice.

---

## 10. Pula połączeń pod obciążeniem

### 10.1 Po co pula

> 💡 **Analogia — wypożyczalnia rowerów.** Nawiązanie połączenia z bazą to jak zbudowanie roweru od zera: zamówienie ramy, montaż, pompowanie kół. Zajmuje dziesiątki milisekund. Pula połączeń to wypożyczalnia: rowery już stoją gotowe, klient przychodzi i odjeżdża w milisekundę. Ale jeśli wszystkich 10 rowerów jest wypożyczonych, jedenasty klient **czeka** — albo odchodzi.

| Parametr | Znaczenie | Typowe wartości |
|---|---|---|
| `pool_size` | Ile połączeń trzymamy stale otwartych | 5–20 (zależnie od liczby workerów) |
| `max_overflow` | Ile połączeń **dodatkowych** można otworzyć ponad `pool_size` | 10–20 |
| `pool_timeout` | Jak długo czekać na wolne połączenie, zanim rzucić `TimeoutError` | 5–30 s |
| `pool_pre_ping` | Przed wydaniem połączenia sprawdź, czy żyje | `True` w produkcji |
| `pool_recycle` | Zamknij połączenie starsze niż N sekund | 1800 (30 min) |
| `pool_use_lifo` | Wydawaj ostatnio zwrócone połączenie | `True` — mniej połączeń bezczynnych |

```python
# examples/14_21_pool_config.py
"""Konfiguracja puli — świadome decyzje, nie kopiowanie domyślnych."""

from sqlalchemy import create_engine

engine = create_engine(
    "postgresql+psycopg://app:secret@localhost:5432/shop",
    pool_size=10,
    max_overflow=20,
    pool_timeout=10,          # po 10 s czekania -> TimeoutError
    pool_pre_ping=True,       # chroni przed "połączeniem zombie"
    pool_recycle=1800,        # odświeżaj co 30 min
    pool_use_lifo=True,       # ostatnio zwrócone używane jako pierwsze
)

print(engine.pool.status())
# Pool size: 10  Connections in pool: 0  Current Overflow: 0  Current Checked out connections: 0
```

### 10.2 Połączenie zombie

**Połączenie zombie** to połączenie, które SQLAlchemy uważa za otwarte, ale po drugiej stronie już nie istnieje. Typowe przyczyny: firewall z limitem bezczynności, restart bazy, timeout po stronie serwera proxy. Objaw: pierwsze zapytanie na takim połączeniu kończy się `OperationalError: server closed the connection unexpectedly`.

Rozwiązania: `pool_pre_ping=True` (tanie sprawdzenie przed wydaniem połączenia) oraz `pool_recycle` krótszy niż najkrótszy timeout w infrastrukturze.

> ⚠️ **Pułapka — `pool_pre_ping` nie jest darmowe.** Wysyła dodatkowe polecenie na każdym wypożyczeniu połączenia. Przy bardzo wysokim ruchu i krótkich zapytaniach koszt może być odczuwalny. Ale w większości aplikacji jest to **znacznie** tańsze niż obsługa błędu „połączenie padło”. Włączaj domyślnie.

### 10.3 PgBouncer — pooling przed poolingiem

Gdy masz 50 instancji aplikacji po 20 połączeń, to 1000 połączeń do PostgreSQL. Każde połączenie to osobny proces w bazie — 1000 procesów to setki megabajtów RAM i degradacja planisty. **PgBouncer** to zewnętrzny pooler, który utrzymuje np. 50 prawdziwych połączeń i multiplexuje na nich tysiące połączeń aplikacji.

| Tryb PgBouncer | Kiedy zwraca połączenie do puli | Konsekwencja |
|---|---|---|
| **session pooling** | Po rozłączeniu klienta | Zgodny ze wszystkim, ale słabo pomaga |
| **transaction pooling** | Po zakończeniu każdej transakcji | Najczęstszy wybór; **uwaga na `SET`** |
| **statement pooling** | Po każdym poleceniu | Nie działa z transakcjami — praktycznie nieużywany |

> ⚠️ **Pułapka — `transaction pooling` a `SET` i `prepared statements`.** W trybie transaction pooling Twoje polecenie może trafić na inne połączenie fizyczne niż poprzednie. Dlatego: (a) `SET` wykonane w jednej transakcji **nie obowiązuje** w następnej — ustawiaj parametry przez `options` w URL albo przez `SET LOCAL` wewnątrz transakcji; (b) prepared statements muszą być obsługiwane z uwagą — w psycopg3 sterownik domyślnie zarządza nimi bezpiecznie (tryb `prepare_threshold`), ale konfiguracje z `prepare=True` na poziomie dialektu mogą sprawić problem. Zawsze testuj konfigurację z PgBouncerem, zamiast zakładać, że „powinno działać”.

> ⚠️ **Pułapka — poziom izolacji a PgBouncer.** Jeśli ustawiasz izolację przez `execution_options` na połączeniu, a pooler zwraca połączenie po każdej transakcji, mechanizm „resetu izolacji po commicie” opiera się na tym, że SQLAlchemy wykona odpowiednie polecenie przed oddaniem połączenia. W trybie transaction pooling połączenie może zostać oddane **wcześniej**. Nie polegaj na „wyciekającej” izolacji — ustawiaj ją jawnie na silniku lub per transakcja.

---

## 11. MVCC w PostgreSQL kontra model blokujący SQLite

### 11.1 MVCC — jak PostgreSQL pozwala czytać bez blokowania

**MVCC (Multi-Version Concurrency Control, wielowersyjna kontrola współbieżności)** to strategia, w której baza nie nadpisuje wierszy w miejscu. Każda zmiana tworzy **nową wersję** wiersza. Transakcja widzi wersję, która odpowiada jej migawce.

> 💡 **Analogia — dokumenty w segregatorze z datami.** Nie gumkujesz strony i nie piszesz na niej od nowa. Wkładasz **nową kartkę** z nową datą. Ktoś, kto przegląda segregator z datą „wczoraj”, nadal widzi starą kartkę. Dopiero po zamknięciu wszystkich przeglądających starych kartek można je wyrzucić (VACUUM).

Konsekwencje praktyczne:

- **Czytelnicy nie blokują pisarzy** i odwrotnie. `SELECT` nigdy nie czeka na `UPDATE`.
- **Zapis do tego samego wiersza blokuje.** Dwie transakcje modyfikujące ten sam wiersz: druga czeka.
- **Stare wersje zajmują miejsce** do czasu `VACUUM` — stąd bloat i znaczenie długich transakcji.
- **Snapshot trwa tak długo, jak transakcja.** Im dłużej, tym więcej wersji musi być zachowanych.

### 11.2 SQLite — jeden piszący naraz

SQLite to zupełnie inny model: **jedna baza = jeden plik**, a zapisy są serializowane. Nie ma MVCC dla zapisów. W trybie domyślnym (journal rollback) zapisujący zakłada **blokadę całej bazy**; inni czytelnicy mogą czytać w tym czasie w trybie WAL, ale **drugi piszący musi czekać**.

| Cecha | PostgreSQL | SQLite |
|---|---|---|
| Blokady wierszowe | Tak (`FOR UPDATE`) | Nie ma |
| `SELECT ... FOR UPDATE` | Tak | Ignorowane przez dialekt |
| Wielu piszących równocześnie | Tak | Nie — jeden na raz |
| Poziomy izolacji | 4 (realnie 3) | Nie ma sensownego wyboru |
| MVCC dla odczytów | Tak | Częściowo (WAL) |
| Błąd przy konflikcie zapisu | Czekanie, potem deadlock/serialization error | `database is locked` |
| Zastosowanie | Produkcja, wielu użytkowników | Prototypy, aplikacje desktopowe, testy |

### 11.3 Tryb WAL i `BEGIN IMMEDIATE`

Domyślny dziennik SQLite (rollback journal) blokuje bazę na czas zapisu i blokuje czytelników. **WAL (Write-Ahead Logging)** pozwala czytelnikom czytać w trakcie zapisu — ale nadal tylko jeden piszący naraz.

```python
# examples/14_22_sqlite_wal.py
"""SQLite w trybie WAL + BEGIN IMMEDIATE: przewidywalne blokady."""

from sqlalchemy import create_engine, event, text

engine = create_engine("sqlite:///app.db")

# WAL to ustawienie TRWAŁE bazy — wystarczy raz, ale bezpiecznie jest powtarzać.
with engine.begin() as conn:
    conn.execute(text("PRAGMA journal_mode=WAL"))
    conn.execute(text("PRAGMA busy_timeout=5000"))  # czekaj 5 s zamiast od razu rzucać błąd


# BEGIN IMMEDIATE: bierz blokadę zapisu od razu, a nie dopiero przy pierwszym INSERT.
# Dzięki temu SQLite nie zgłasza "database is locked" przy eskalacji blokady.
@event.listens_for(engine, "connect")
def _sqlite_on_connect(dbapi_connection, connection_record):  # noqa: ANN001
    # Wyłącz własne zarządzanie transakcjami w pysqlite...
    dbapi_connection.isolation_level = None


@event.listens_for(engine, "begin")
def _sqlite_begin_immediate(conn):  # noqa: ANN001
    # ...i przejmij je sam, zawsze zaczynając od blokady zapisu.
    conn.exec_driver_sql("BEGIN IMMEDIATE")
```

> 🧠 **Dlaczego tak jest — pysqlite ma „własne zdanie” o transakcjach.** Sterownik `sqlite3` z biblioteki standardowej historycznie sam zarządza transakcjami: domyślnie nie wysyła `BEGIN` przy starcie transakcji, tylko **przed pierwszym `INSERT`/`UPDATE`/`DELETE`**. To prowadzi do zaskakującego zachowania: `SELECT` w transakcji działa w trybie autocommit, a blokada zapisu pojawia się dopiero w połowie pracy — i wtedy, gdy trzeba ją **eskalować** z odczytu do zapisu, może wystąpić `database is locked`. Powyższy wzorzec (`isolation_level=None` + własny `BEGIN IMMEDIATE`) to zalecany przez dokumentację SQLAlchemy sposób uzyskania przewidywalnego zachowania. To jest **specyficzne dla SQLite/pysqlite** — nie przenoś tego na PostgreSQL.

> ⚠️ **Pułapka — `database is locked` w testach.** Jeśli widzisz ten błąd, to prawie zawsze znaczy, że dwie sesje próbują pisać. Rozwiązania: (a) `PRAGMA busy_timeout`, (b) `BEGIN IMMEDIATE`, (c) — najczęściej najlepsze — **przełącz testy współbieżności na PostgreSQL**. Testy na SQLite, które „przechodzą”, mogą wcale nie sprawdzać współbieżności, bo SQLite serializuje zapisy za Ciebie.

### 11.4 Kiedy SQLite w ogóle wystarcza

SQLite jest **znakomitą** bazą — dla właściwego zastosowania:

- aplikacja desktopowa lub mobilna (jeden użytkownik, jedno urządzenie),
- narzędzie CLI, skrypt analityczny,
- prototyp, MVP, nauka,
- aplikacja z przewagą odczytów, z rzadkimi zapisami,
- testy automatyczne (choć nie testy współbieżności!),
- cache lokalny, magazyn danych dla jednego procesu.

Nie używa się jej, gdy: wielu użytkowników pisze równocześnie, potrzebujesz blokad wierszowych, replikacji, kopii zapasowej na żywo, partycjonowania, albo gdy rozmiar danych przekracza kilkaset GB.

> 🧪 **Ćwiczenie — sprawdź, że SQLite ignoruje `FOR UPDATE`.** Uruchom na SQLite dwa skrypty równolegle, w których oba wykonują `SELECT ... with_for_update()` na tym samym wierszu, modyfikują go i commitują. Zaobserwuj, że końcowy stan jest „zły” (obaj nadpisali), a żaden nie dostał błędu. Następnie uruchom to samo na PostgreSQL — jeden z nich poczeka, a wynik będzie poprawny.

---

## 12. Testowanie współbieżności

### 12.1 Zasada: nie da się przetestować współbieżności jednym wątkiem

Test, który uruchamia operacje sekwencyjnie, **nie wykryje** wyścigu. Potrzebujesz:

1. **Kilku niezależnych połączeń** — jedno połączenie to jedna transakcja, więc nie ma konkurencji.
2. **Synchronizacji startu** — wszystkie „wątki” muszą zacząć w tym samym momencie (`threading.Barrier`).
3. **Sztucznego opóźnienia** w krytycznym miejscu, żeby poszerzyć okno wyścigu.
4. **Bazy, która faktycznie blokuje** — PostgreSQL.

```python
# examples/14_23_race_demo.py
"""Reprodukcja wyścigu: 10 'kupujących' walczy o ostatnią sztukę.

Uruchom na PostgreSQL. Na SQLite zobaczysz inne (błędne) wyniki —
i to też jest wartościowe doświadczenie.
"""

from __future__ import annotations

import threading
from concurrent.futures import ThreadPoolExecutor

from sqlalchemy import Integer, String, create_engine, func, select, text
from sqlalchemy.orm import DeclarativeBase, Mapped, Session, mapped_column


class Base(DeclarativeBase):
    pass


class Product(Base):
    __tablename__ = "product"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50))
    stock: Mapped[int] = mapped_column(Integer, nullable=False)


PURCHASES: list[int] = []  # wynik zbiorczy (lista tylko dla wątku głównego)
LOCK = threading.Lock()
BARRIER = threading.Barrier(10)  # wszyscy startują równocześnie


def main() -> None:
    engine = create_engine("postgresql+psycopg://app:secret@localhost:5432/race_demo")
    Base.metadata.drop_all(engine)
    Base.metadata.create_all(engine)

    with Session(engine) as session:
        session.add(Product(id=1, name="Rower górski", stock=1))  # UWAGA: 1 sztuka
        session.commit()

    def buy(worker_id: int) -> bool:
        """Wersja BŁĘDNA: odczytaj, pomyśl, zapisz."""
        BARRIER.wait()  # synchronizacja startu
        with Session(engine) as session:
            product = session.get(Product, 1)
            if product.stock > 0:
                # Sztuczne opóźnienie — poszerza okno wyścigu.
                # Bez tego wątek zdąży zapisać, zanim inny odczyta.
                import time

                time.sleep(0.05)
                product.stock -= 1
                session.commit()
                with LOCK:
                    PURCHASES.append(worker_id)
                return True
            session.rollback()
            return False

    with ThreadPoolExecutor(max_workers=10) as executor:
        results = list(executor.map(buy, range(10)))

    with Session(engine) as session:
        final_stock = session.scalar(select(Product.stock).where(Product.id == 1))

    print(f"Udane zakupy: {len(PURCHASES)} (oczekiwane: 1)")
    print(f"Stan magazynu na końcu: {final_stock} (oczekiwane: 0)")


if __name__ == "__main__":
    main()
```

> 🔬 **Pod maską — dlaczego to się psuje.** Sekwencja na `READ COMMITTED` w PostgreSQL:
>
> ```text
> Wątek 1: SELECT ... FOR? (nie!) -> widzi stock=1
> Wątek 2: SELECT -> widzi stock=1
> Wątek 3: SELECT -> widzi stock=1
> ...
> Wątek 1: UPDATE product SET stock=0 WHERE id=1   -- blokada, potem commit
> Wątek 2: UPDATE product SET stock=0 WHERE id=1   -- czeka, potem NADPISUJE na 0
> Wątek 3: UPDATE product SET stock=0 WHERE id=1   -- czeka, potem NADPISUJE na 0
> ```
>
> Każdy wątek zapisał `0`. Skończony stan: `stock = 0`, ale **dziesięciu** klientów dostało potwierdzenie zakupu. To jest lost update w najczystszej postaci. Zwróć uwagę, że baza **nie zgłosiła błędu** — bo z jej punktu widzenia wszystko było poprawne: nikt nie naruszył ograniczenia, nikt nie zablokował się na dłużej niż chwilę.

### 12.2 Miary, które warto zbierać

| Miara | Co mówi |
|---|---|
| Liczba udanych operacji | Czy nie ma „nadmiarowych sukcesów” (lost update) |
| Stan końcowy zasobu | Czy nie ma wartości niemożliwych (ujemny magazyn) |
| Liczba wyjątków i ich typy | Czy kod radzi sobie z konfliktem, czy się wywala |
| Łączny czas wykonania | Czy blokady nie zabiły przepustowości |
| Liczba ponowień (retry) | Czy retry działa i nie wchodzi w pętlę |

> ⚠️ **Pułapka — test współbieżności, który „przechodzi”, nic nie znaczy.** Jeśli testy są szybkie, wątki nie zdążą się zderzyć i wszystko wygląda poprawnie. Dlatego w teście **musi** być sztuczne opóźnienie (`time.sleep`) w krytycznym miejscu albo `Barrier` na starcie. Test bez tych elementów to test sekwencyjny w przebraniu. Uczciwie: testy współbieżności są zawsze probabilistyczne — mogą wykryć problem, ale nie mogą udowodnić jego braku. Ich rolą jest **regresja**: gdy ktoś usunie blokadę, test powinien to złapać.

---

## 13. Warsztat obowiązkowy: „kup ostatnią sztukę” w czterech wersjach

To najważniejszy fragment modułu. Uruchom wszystkie cztery wersje na PostgreSQL i porównaj wyniki. Każda wersja to **to samo wymaganie biznesowe** rozwiązane inaczej.

### 13.1 Schemat

```python
# examples/14_24_shop_models.py
"""Modele wspólne dla wszystkich czterech wersji."""

from datetime import datetime

from sqlalchemy import DateTime, Integer, String, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class Product(Base):
    __tablename__ = "product"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50))
    stock: Mapped[int] = mapped_column(Integer, nullable=False)
    version_id: Mapped[int] = mapped_column(Integer, nullable=False, default=1)

    __mapper_args__ = {"version_id_col": version_id}


class Purchase(Base):
    __tablename__ = "purchase"

    id: Mapped[int] = mapped_column(primary_key=True)
    product_id: Mapped[int]
    buyer: Mapped[str] = mapped_column(String(50))
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now()
    )
```

### 13.2 Wersja A — błędna (read-modify-write)

```python
# examples/14_24a_naive.py
"""WERSJA A — BŁĘDNA. Oczekiwany wynik: 10 udanych zakupów, magazyn = 0."""

import time

from sqlalchemy import create_engine
from sqlalchemy.orm import Session

from examples_shop_models import Base, Product, Purchase


def buy_naive(engine, product_id: int, buyer: str) -> bool:
    with Session(engine) as session:
        product = session.get(Product, product_id)
        if product.stock <= 0:
            session.rollback()
            return False
        time.sleep(0.05)  # okno wyścigu
        product.stock -= 1
        session.add(Purchase(product_id=product_id, buyer=buyer))
        session.commit()
        return True
```

**Dlaczego jest błędna:** między `session.get()` a `session.commit()` istnieje okno, w którym dziesięciu innych kupujących widzi ten sam `stock = 1`. Wszyscy zapisują `stock = 0`. `CHECK (stock >= 0)` nie pomoże — zero jest poprawne.

### 13.3 Wersja B — `FOR UPDATE`

```python
# examples/14_24b_for_update.py
"""WERSJA B — pesymistyczna. Oczekiwany wynik: 1 udany zakup, magazyn = 0."""

import time

from sqlalchemy import select
from sqlalchemy.orm import Session

from examples_shop_models import Product, Purchase


def buy_for_update(engine, product_id: int, buyer: str) -> bool:
    with Session(engine) as session:
        product = session.scalars(
            select(Product).where(Product.id == product_id).with_for_update()
        ).one()

        if product.stock <= 0:
            session.rollback()  # zwolnij blokadę NATYCHMIAST
            return False

        time.sleep(0.05)  # to opóźnienie jest teraz bezpieczne — inni CZEKAJĄ
        product.stock -= 1
        session.add(Purchase(product_id=product_id, buyer=buyer))
        session.commit()
        return True
```

> 🔬 **Pod maską.** W logu zobaczysz naprzemiennie: `SELECT ... FOR UPDATE` (jedna transakcja wchodzi), potem długą ciszę (pozostałe czekają), potem `UPDATE`, `INSERT`, `COMMIT`, i dopiero wtedy kolejna transakcja dostaje `SELECT ... FOR UPDATE` i widzi `stock = 0`. To dokładnie pożądane zachowanie — ale zauważ koszt: dziewięciu klientów czeka 50 ms każdy. Przy `sleep(1)` zamiast `sleep(0.05)` ostatni czekałby 9 sekund. **Blokada pesymistyczna działa, ale jej koszt rośnie liniowo z liczbą oczekujących.**

### 13.4 Wersja C — blokada optymistyczna

```python
# examples/14_24c_optimistic.py
"""WERSJA C — optymistyczna. Oczekiwany wynik: 1 udany zakup, magazyn = 0."""

from sqlalchemy.exc import StaleDataError
from sqlalchemy.orm import Session

from examples_shop_models import Product, Purchase


def buy_optimistic(engine, product_id: int, buyer: str) -> bool:
    with Session(engine) as session:
        product = session.get(Product, product_id)
        if product.stock <= 0:
            session.rollback()
            return False

        product.stock -= 1
        session.add(Purchase(product_id=product_id, buyer=buyer))
        try:
            session.commit()
            return True
        except StaleDataError:
            # Ktoś nas ubiegł. To nie awaria — to informacja.
            session.rollback()
            return False
```

> 🔬 **Pod maską.** SQLAlchemy wygeneruje:
>
> ```sql
> UPDATE product
> SET stock = %(stock)s, version_id = %(version_id)s
> WHERE product.id = %(id)s AND product.version_id = %(version_id_1)s
> ```
>
> Dziewięć transakcji zmieni **zero wierszy**, bo wersja już się zmieniła. SQLAlchemy to wykryje i rzuci `StaleDataError`. **Uwaga na pułapkę:** te dziewięć transakcji nie czekało na siebie nawzajem — wszystkie pracowały równolegle i dopiero przy `commit()` się zderzyły. To jest przewaga optymistycznej: brak blokad, brak czekania. Koszt: obsługa wyjątku w kodzie.

### 13.5 Wersja D — atomowy warunkowy `UPDATE` + `ON CONFLICT`

```python
# examples/14_24d_atomic.py
"""WERSJA D — atomowa. Oczekiwany wynik: 1 udany zakup, magazyn = 0."""

from sqlalchemy import update
from sqlalchemy.orm import Session

from examples_shop_models import Product, Purchase


def buy_atomic(engine, product_id: int, buyer: str) -> bool:
    with Session(engine) as session:
        stmt = (
            update(Product)
            .where(Product.id == product_id, Product.stock > 0)
            .values(stock=Product.stock - 1)
            .returning(Product.id)
        )
        updated = session.execute(stmt).first()
        if updated is None:
            session.rollback()
            return False

        session.add(Purchase(product_id=product_id, buyer=buyer))
        session.commit()
        return True
```

Ta wersja robi dokładnie jedną rundę do bazy dla sprawdzenia i zapisu. Nie potrzebuje `FOR UPDATE`, nie potrzebuje kolumny wersji, nie czeka. `ON CONFLICT` dochodzi tu dodatkowo, gdy chcemy **idempotencji po kluczu kupującego** (żeby ten sam klient nie kupił dwa razy przez podwójne kliknięcie) — dokładnie tak, jak w sekcji 7.1.

### 13.6 Wyniki — tabela porównawcza

Uruchomienie na PostgreSQL, 10 równoległych wątków, `stock = 1`, opóźnienie 50 ms w krytycznym miejscu:

| Wersja | Udane zakupy | Stan magazynu | Rekordów `purchase` | Czas łączny | Wyjątki | Ocena |
|---|---|---|---|---|---|---|
| **A — naiwna** | 10 | 0 | 10 | ~0,1 s | 0 | ❌ **Sprzedano 10 sztuk, była 1** |
| **B — `FOR UPDATE`** | 1 | 0 | 1 | ~0,5 s | 0 | ✅ Poprawna, ale 9 wątków czekało |
| **C — optymistyczna** | 1 | 0 | 1 | ~0,15 s | 9 × `StaleDataError` | ✅ Poprawna, szybka, wymaga obsługi wyjątku |
| **D — atomowa** | 1 | 0 | 1 | ~0,1 s | 0 | ✅ **Najlepsza** — 1 runda, brak czekania |

> 🧠 **Dlaczego tak jest — pozorna przewaga wersji A.** Wersja A jest najszybsza i „nie rzuca błędów”. To jest pułapka, w którą wpada większość zespołów: kod działa, testy przechodzą (bo testy są sekwencyjne), aplikacja jest szybka — i dopiero po roku ktoś zauważa, że w bazie jest 10 000 sprzedanych rowerów, których nie ma na stanie. **Poprawność nie objawia się błędem.** Dlatego testy współbieżności są obowiązkowe dla operacji na współdzielonych zasobach.

> ⚠️ **Pułapka — `expire_on_commit=False` i brak odświeżenia po konflikcie.** W wersji C po `StaleDataError` i `rollback()` obiekt `product` w pamięci nadal ma stare wartości. Jeśli spróbujesz ponowić operację na tym samym obiekcie, będzie on „myślał”, że `stock = 1`, choć w bazie jest 0. **Zawsze po `rollback()` pobierz obiekt na nowo** albo wywołaj `session.refresh(product)`. Wersja D jest odporna na ten problem, bo nie trzyma stanu w obiekcie — czyta wynik wprost z `RETURNING`.

> ⚠️ **Pułapka — `SERIALIZABLE` jako „srebrna kula”.** Kuszące: ustawmy `SERIALIZABLE` globalnie i problemy znikną. Znikną — razem z przepustowością. PostgreSQL implementuje `SERIALIZABLE` przez *serializable snapshot isolation* (SSI), które **aktywni wykrywa konflikty i wymaga ponowień** (`40001 serialization_failure`). Przy dużym ruchu możesz zobaczyć lawinę retry. Do tego SSI utrzymuje predykatowe blokady, które rosną z liczbą odczytów. Praktyka: używaj `SERIALIZABLE` **punktowo**, dla konkretnej operacji wymagającej ochrony przed write skew, i zawsze z gotowym retry.

---

## Podsumowanie

1. **Transakcja to koperta, nie pojedyncze polecenie.** `engine.begin()` i `with Session(...) as s, s.begin():` dają atomowość: wszystko albo nic. Wyjątek w środku = automatyczny `ROLLBACK`.
2. **ACID to cztery obietnice, z których każda kosztuje.** Izolacja kosztuje blokady albo wersjonowanie, trwałość kosztuje zapis na dysk. Wybieraj **najniższy wystarczający** poziom izolacji, nie najwyższy.
3. **PostgreSQL domyślnie działa na `READ COMMITTED`** i to jest właściwy wybór dla większości aplikacji. Każde polecenie widzi świeżą migawkę; w obrębie jednego polecenia migawka jest stała, a `UPDATE` wykonuje *re-check* warunku.
4. **`REPEATABLE READ` w PostgreSQL to snapshot isolation** — mocniejszy niż standard SQL (chroni przed phantomami i lost update), ale **nie chroni przed write skew**.
5. **`Session(isolation_level=...)` nie istnieje.** Izolację ustawiasz na `Engine`, na `Connection`, albo per transakcja przez `session.connection(execution_options={"isolation_level": ...})` — i to **zanim** wykonasz cokolwiek innego w tej sesji.
6. **Savepoint (`session.begin_nested()`) pozwala cofnąć część transakcji** bez utraty całości — niezbędne w PostgreSQL, gdzie błąd w transakcji blokuje dalszą pracę (`current transaction is aborted`).
7. **W 2.0 `session.commit()` zawsze zatwierdza najbardziej zewnętrzną transakcję** — savepoint zatwierdzasz przez obiekt z `begin_nested()`.
8. **`with_for_update()` to klucz do toalety** — skuteczny, ale kosztuje czekanie innych. `skip_locked=True` zamienia czekanie w „weź inne zadanie” (idealne dla kolejek). Na SQLite `FOR UPDATE` **nie działa wcale**.
9. **Jeśli operację da się wyrazić jako warunkowy `UPDATE ... WHERE ... RETURNING`, zrób to.** Jedna runda do bazy, brak jawnej blokady, odporność na lost update. To domyślny wybór dla liczników i stanów magazynowych.
10. **Blokada optymistyczna (`version_id_col`) nie blokuje nikogo** — wykrywa konflikt przy zapisie (`StaleDataError`). Idealna do edycji formularzy. Po `StaleDataError` **obowiązkowo** `rollback()`.
11. **Unikalny indeks w bazie to jedyna prawdziwa gwarancja idempotencji.** Sprawdzenie w Pythonie zawsze ma okno czasowe. `INSERT ... ON CONFLICT DO NOTHING` + `RETURNING` to gotowy wzorzec.
12. **Deadlock to normalne zjawisko, nie awaria.** Minimalizuj stałą kolejnością blokowania, obsługuj retry z **jitterem** i zawsze na **świeżej sesji** (albo po `rollback()`).
13. **Długie transakcje to najczęstsza przyczyna degradacji.** Ustaw `statement_timeout`, `lock_timeout`, `idle_in_transaction_session_timeout`. Pamiętaj, że długie transakcje blokują `VACUUM` i powodują bloat.
14. **`pool_pre_ping=True` i `pool_recycle` chronią przed połączeniami zombie.** `pool_size` musi być świadomą decyzją, a PgBouncer w trybie transaction pooling wymaga ostrożności z `SET` i prepared statements.
15. **Nie da się przetestować współbieżności jednym wątkiem.** Potrzebujesz `Barrier`, sztucznego `sleep` i — najlepiej — PostgreSQL. Testy na SQLite dają fałszywe poczucie bezpieczeństwa.

---

## Ćwiczenia

### Ćwiczenie 1 (łatwe) — zbadaj izolację empirycznie

Napisz skrypt, który w dwóch wątkach na PostgreSQL wykonuje:

- **Wątek A:** `BEGIN` → `UPDATE account SET balance = balance - 100 WHERE id = 1` → `sleep(1)` → `COMMIT`.
- **Wątek B:** `sleep(0.3)` → `BEGIN` → `SELECT balance FROM account WHERE id = 1` → wypisz → `COMMIT`.

Uruchom go dwa razy: raz z `isolation_level="READ COMMITTED"`, raz z `"REPEATABLE READ"`. Wyjaśnij różnicę w wyniku, odwołując się do pojęcia migawki (snapshot).

### Ćwiczenie 2 (średnie) — idempotentny endpoint płatności

Zaimplementuj funkcję `process_payment(session, idempotency_key, amount, account_id)`, która:

1. Jest bezpieczna przy równoległym wywołaniu z tym samym kluczem — dokładnie jedna płatność powstaje w bazie.
2. Zwraca ten sam `payment_id` przy powtórnym wywołaniu.
3. Jeśli saldo konta jest niewystarczające, nie tworzy płatności i zwraca informację o braku środków.
4. Nie tworzy ujemnego salda nawet przy 50 równoległych wywołaniach.

Wymagania: użyj `ON CONFLICT`, warunkowego `UPDATE` i unikalnego indeksu. Napisz test z `ThreadPoolExecutor` (50 zadań, saldo pozwalające na 10 płatności) i udowodnij, że dokładnie 10 się powiodło, a saldo nie jest ujemne.

### Ćwiczenie 3 (trudne) — napraw deadlock przez uporządkowanie blokad

Masz funkcję `rebalance(session, account_ids: list[int])`, która:

- blokuje wszystkie konta z listy przez `with_for_update()`,
- wyrównuje ich salda (przenosi nadwyżki do kont z deficytem).

Funkcja działa, ale przy równoległym wywołaniu z **odwrotnie posortowanymi** listami identyfikatorów powoduje deadlocki (sprawdź to, uruchamiając 20 równoległych wywołań w pętli z losową kolejnością `account_ids`).

Twoje zadanie:

1. Zreprodukuj deadlock i pokaż komunikat błędu wraz z kodem SQLSTATE.
2. Napraw go **dwoma** sposobami: (a) przez uporządkowanie blokad, (b) przez dodanie retry z `@retry_on_deadlock`.
3. Porównaj oba rozwiązania: w jakim scenariuszu każde jest lepsze? Czy da się je połączyć?

---

### Rozwiązania

<details>
<summary><strong>Rozwiązanie 1 — badanie izolacji</strong></summary>

```python
# solutions/14_1_isolation_probe.py
"""Dowód empiryczny: READ COMMITTED vs REPEATABLE READ."""

import threading
import time

from sqlalchemy import create_engine, text
from sqlalchemy.orm import Session

URL = "postgresql+psycopg://app:secret@localhost:5432/shop"

results: dict[str, list[int]] = {"b_reads": []}
barrier = threading.Barrier(2)


def run(engine) -> None:
    barrier.wait()

    def writer() -> None:
        with Session(engine) as session:
            session.execute(text("UPDATE account SET balance = balance - 100 WHERE id = 1"))
            time.sleep(1.0)
            session.commit()

    def reader() -> None:
        time.sleep(0.3)
        with Session(engine) as session:
            first = session.scalar(text("SELECT balance FROM account WHERE id = 1"))
            time.sleep(0.5)
            second = session.scalar(text("SELECT balance FROM account WHERE id = 1"))
            results["b_reads"].append((first, second))
            session.rollback()

    threads = [threading.Thread(target=writer), threading.Thread(target=reader)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()


for level in ("READ COMMITTED", "REPEATABLE READ"):
    engine = create_engine(URL, isolation_level=level)
    results["b_reads"].clear()
    with engine.begin() as conn:
        conn.execute(text("UPDATE account SET balance = 100 WHERE id = 1"))
    run(engine)
    first, second = results["b_reads"][0]
    print(f"{level:16} -> odczyt 1: {first}, odczyt 2: {second}")
```

**Wyjaśnienie.**

- **`READ COMMITTED`:** każdy `SELECT` widzi świeżą migawkę. Drugi odczyt następuje po `COMMIT` pisarza, więc widzi nową wartość. Wynik: `(100, 0)`. To *non-repeatable read* — w tej samej transakcji dwie różne wartości.
- **`REPEATABLE READ`:** cała transakcja widzi jedną migawkę z momentu jej rozpoczęcia. Drugi odczyt widzi tę samą, „zamrożoną” wartość. Wynik: `(100, 100)`.

Uwaga na szczegół: pisarz w `REPEATABLE READ` **nie** zostanie zablokowany przez czytelnika (PostgreSQL MVCC nie blokuje czytelników), ale **czytelnik zobaczy stan sprzed zmiany**. Gdyby czytelnik próbował zapisać zmieniony przez kogoś wiersz, dostałby `40001 could not serialize access due to concurrent update`.

**Minipułapka:** `results["b_reads"]` jest zapisywany tylko z jednego wątku, więc nie potrzebuje blokady — ale w ogólnym przypadku dopisywanie do listy z wielu wątków wymaga `threading.Lock`. Nie zakładaj, że „akurat tutaj” jest bezpiecznie, jeśli nie potrafisz tego uzasadnić.

</details>

<details>
<summary><strong>Rozwiązanie 2 — idempotentna płatność</strong></summary>

```python
# solutions/14_2_payment.py
"""Idempotentna płatność: unikalny klucz + atomowy warunkowy UPDATE."""

from decimal import Decimal

from sqlalchemy import Numeric, String, UniqueConstraint, create_engine, select, update
from sqlalchemy.dialects.postgresql import insert as pg_insert
from sqlalchemy.orm import DeclarativeBase, Mapped, Session, mapped_column


class Base(DeclarativeBase):
    pass


class Account(Base):
    __tablename__ = "account"

    id: Mapped[int] = mapped_column(primary_key=True)
    balance: Mapped[Decimal] = mapped_column(Numeric(14, 2), nullable=False)


class Payment(Base):
    __tablename__ = "payment"
    __table_args__ = (UniqueConstraint("idempotency_key", name="uq_payment_idem"),)

    id: Mapped[int] = mapped_column(primary_key=True)
    idempotency_key: Mapped[str] = mapped_column(String(64), index=True)
    account_id: Mapped[int]
    amount: Mapped[Decimal] = mapped_column(Numeric(14, 2))
    status: Mapped[str] = mapped_column(String(20))


class InsufficientFunds(Exception):
    """Brak środków na koncie."""


def process_payment(
    session: Session, idempotency_key: str, amount: Decimal, account_id: int
) -> int:
    """Zwraca id płatności. Powtórne wywołanie z tym samym kluczem jest bezpieczne."""
    # Krok 1: spróbuj utworzyć płatność. Unikalny indeks rozstrzyga wyścig.
    insert_stmt = (
        pg_insert(Payment)
        .values(
            idempotency_key=idempotency_key,
            account_id=account_id,
            amount=amount,
            status="pending",
        )
        .on_conflict_do_nothing(index_elements=["idempotency_key"])
        .returning(Payment.id)
    )
    payment_id = session.execute(insert_stmt).scalar_one_or_none()

    if payment_id is None:
        # Płatność już istnieje — zwróć ją. NIE powtarzaj obciążenia.
        session.rollback()
        existing = session.scalars(
            select(Payment).where(Payment.idempotency_key == idempotency_key)
        ).one()
        return existing.id

    # Krok 2: atomowo obciąż konto, jeśli ma środki. Re-check w UPDATE chroni
    # przed wyścigiem i przed ujemnym saldem.
    debit_stmt = (
        update(Account)
        .where(Account.id == account_id, Account.balance >= amount)
        .values(balance=Account.balance - amount)
        .returning(Account.balance)
    )
    new_balance = session.execute(debit_stmt).scalar_one_or_none()

    if new_balance is None:
        # Brak środków: cofnij CAŁĄ transakcję, więc płatność też nie powstanie.
        session.rollback()
        raise InsufficientFunds(f"konto {account_id} nie ma {amount}")

    session.execute(
        update(Payment).where(Payment.id == payment_id).values(status="completed")
    )
    session.commit()
    return payment_id
```

**Dlaczego tak.**

1. **`ON CONFLICT DO NOTHING` + `RETURNING`** rozstrzyga wyścig o klucz idempotencji **atomowo** — nie ma okna między sprawdzeniem a wstawieniem.
2. **Warunkowy `UPDATE ... WHERE balance >= amount`** jest jedynym miejscem, które decyduje o obciążeniu. Nie ma `SELECT` przed nim, więc nie ma okna na lost update. To jest ta „jedna runda”, o której mówiliśmy w sekcji 5.5.
3. **Rollback cofa wszystko** — w tym wstawioną płatność. Dzięki temu nie zostaje „płatność pending” dla transakcji, która się nie udała.

**Alternatywa:** zamiast `RETURNING` można sprawdzić `rowcount` po `UPDATE`. Wtedy jednak nie dowiesz się, jakie jest nowe saldo bez dodatkowego `SELECT`. `RETURNING` jest wygodniejszy i obsługiwany przez PostgreSQL oraz SQLite ≥ 3.35.

**Minipułapka:** jeśli `amount <= 0`, `WHERE balance >= amount` przepuściłby operację i saldo **wzrosłoby**. Walidacja `amount > 0` musi być osobno — albo w kodzie, albo jako `CheckConstraint("amount > 0")`. Nie zakładaj, że warunek biznesowy jest oczywisty dla bazy.

**Test współbieżności:**

```python
# solutions/14_2b_payment_test.py
from concurrent.futures import ThreadPoolExecutor
from decimal import Decimal

with Session(engine) as session:
    session.add(Account(id=1, balance=Decimal("100.00")))
    session.commit()


def attempt(i: int) -> str:
    with Session(engine) as session:
        try:
            process_payment(session, f"key-{i}", Decimal("10.00"), 1)
            return "ok"
        except InsufficientFunds:
            return "brak-srodkow"


with ThreadPoolExecutor(max_workers=20) as pool:
    outcomes = list(pool.map(attempt, range(50)))

print(f"udane: {outcomes.count('ok')}")           # oczekiwane: 10
print(f"brak środków: {outcomes.count('brak-srodkow')}")  # oczekiwane: 40

with Session(engine) as session:
    balance = session.scalar(select(Account.balance).where(Account.id == 1))
    print(f"saldo: {balance}")                    # oczekiwane: 0.00
```

</details>

<details>
<summary><strong>Rozwiązanie 3 — naprawa deadlocka</strong></summary>

**Krok 1 — reprodukcja.**

```python
# solutions/14_3a_deadlock_repro.py
"""Reprodukcja deadlocka: losowa kolejność blokowania."""

import random
from concurrent.futures import ThreadPoolExecutor

from sqlalchemy import select
from sqlalchemy.orm import Session

from solutions_models import Account


def rebalance_broken(session: Session, account_ids: list[int]) -> None:
    """Wersja BŁĘDNA: blokuje w kolejności, w jakiej dostała identyfikatory."""
    accounts = session.scalars(
        select(Account).where(Account.id.in_(account_ids)).with_for_update()
    ).all()
    # ... logika wyrównania sald ...
    session.commit()


def worker(seed: int) -> str:
    ids = [1, 2, 3, 4]
    random.Random(seed).shuffle(ids)
    with Session(engine) as session:
        try:
            rebalance_broken(session, ids)
            return "ok"
        except Exception as exc:  # noqa: BLE001
            session.rollback()
            return f"{type(exc).__name__}: {str(exc)[:60]}"


with ThreadPoolExecutor(max_workers=20) as pool:
    results = list(pool.map(worker, range(200)))

deadlocks = [r for r in results if "deadlock" in r.lower()]
print(f"deadlocki: {len(deadlocks)} z 200")
if deadlocks:
    print(deadlocks[0])
```

Oczekiwany komunikat (PostgreSQL):

```text
OperationalError: (psycopg.errors.DeadlockDetected) deadlock detected
DETAIL: Process 12345 waits for ShareLock on transaction 67890; blocked by process 54321.
```

**Krok 2a — naprawa przez uporządkowanie blokad.**

```python
# solutions/14_3b_deadlock_ordered.py
def rebalance_ordered(session: Session, account_ids: list[int]) -> None:
    """Blokuj w STAŁEJ kolejności rosnącego id — cykl oczekiwań niemożliwy."""
    ordered_ids = sorted(set(account_ids))  # <- jedyna zmiana
    accounts = session.scalars(
        select(Account)
        .where(Account.id.in_(ordered_ids))
        .order_by(Account.id)  # <- i to
        .with_for_update()
    ).all()
    # ... logika wyrównania sald ...
    session.commit()
```

**Krok 2b — naprawa przez retry.**

```python
# solutions/14_3c_deadlock_retry.py
from examples_retry import retry_on_deadlock


@retry_on_deadlock(max_attempts=5, base_delay=0.02)
def rebalance_retrying(account_ids: list[int]) -> None:
    """Każda próba na ŚWIEŻEJ sesji — warunek poprawności retry."""
    with Session(engine) as session:
        try:
            accounts = session.scalars(
                select(Account).where(Account.id.in_(account_ids)).with_for_update()
            ).all()
            # ... logika wyrównania sald ...
            session.commit()
        except Exception:
            session.rollback()
            raise
```

**Krok 3 — porównanie.**

| Kryterium | Uporządkowanie blokad | Retry |
|---|---|---|
| Eliminuje deadlocki | **Tak, całkowicie** (dla tego wzorca) | Nie — tylko je obsługuje |
| Wymaga zmiany logiki biznesowej | Nie | Nie |
| Dodaje opóźnienia przy konflikcie | Nie | Tak (backoff) |
| Działa przy deadlockach z innych przyczyn | Nie (tylko dla tej klasy) | **Tak** |
| Ryzyko | Brak | Pętla, jeśli limit źle dobrany |
| Efekt uboczny | Brak | Operacja może wykonać się dwukrotnie! |

> ⚠️ **Krytyczna uwaga o retry.** Retry powtarza **całą funkcję**, więc jeśli funkcja ma efekty uboczne poza bazą (wysłany e-mail, wywołane API płatnicze), te efekty mogą wystąpić dwa razy. To jest powód, dla którego retry **musi** obejmować wyłącznie operacje bazodanowe albo być połączony z kluczem idempotencji (sekcja 7.1).

**Czy da się je połączyć? Tak — i to jest najlepsza praktyka.** Uporządkowanie blokad minimalizuje prawdopodobieństwo deadlocka do zera dla znanych wzorców. Retry obsługuje deadlocki, których nie przewidziałeś (np. wynikające z kolejności indeksów w bazie, z triggerów albo z kodu, który ktoś dopisał później). Stosuj oba: pierwsze jako projekt, drugie jako siatkę bezpieczeństwa.

**Minipułapka:** `sorted(set(account_ids))` — użyj `set`, żeby pozbyć się duplikatów. Bez tego `SELECT ... WHERE id IN (1, 1, 2)` zwróci dwa wiersze (albo — w zależności od konstrukcji — jeden), a `sorted` na liście z duplikatami nie zmieni kolejności blokad, ale może Cię zmylić przy debugowaniu.

</details>

---

## Najczęstsze błędy i jak je czytać

| Komunikat / objaw | Przyczyna | Naprawa |
|---|---|---|
| `StaleDataError: UPDATE statement on table 'x' expected to update 1 row(s); 0 were matched` | Ktoś zmienił wiersz między Twoim odczytem a zapisem (blokada optymistyczna) | `session.rollback()` i ponów na świeżych danych; poinformuj użytkownika |
| `PendingRollbackError: This Session's transaction has been rolled back due to a previous exception during flush` | Wyjątek w `flush`/`commit` bez `rollback()` | Wywołaj `session.rollback()` w `except` **zawsze** |
| `This session is in 'prepared' state; no further SQL can be emitted within this transaction` | Wyjątek w transakcji, sesja czeka na rollback | `session.rollback()` i rozpocznij od nowa |
| `current transaction is aborted, commands ignored until end of transaction block` (PostgreSQL) | Błąd polecenia wewnątrz transakcji; PostgreSQL blokuje dalszą pracę | Użyj `begin_nested()` dla operacji, które mogą się nie udać; albo cofnij całą transakcję |
| `deadlock detected` (SQLSTATE `40P01`) | Cykl oczekiwań na blokady | Uporządkuj blokady + `@retry_on_deadlock` ze świeżą sesją |
| `could not serialize access due to concurrent update` (`40001`) | `REPEATABLE READ` / `SERIALIZABLE` wykrył konflikt | Retry — to oczekiwane przy tych poziomach izolacji |
| `OperationalError: server closed the connection unexpectedly` | Połączenie zombie (firewall, restart bazy, pooler) | `pool_pre_ping=True`, `pool_recycle` krótszy niż timeouty w infrastrukturze |
| `TimeoutError: QueuePool limit of size 5 overflow 10 reached` | Pula wyczerpana — zapytania trwają za długo albo sesje nie są zamykane | Znajdź wyciek sesji; skróć transakcje; rozważ zwiększenie `pool_size` |
| `database is locked` (SQLite) | Dwie transakcje próbują pisać | `PRAGMA busy_timeout`, `BEGIN IMMEDIATE`, albo przenieś testy na PostgreSQL |
| `IdleInTransactionSessionTimeout` / połączenie zerwane po bezczynności | Transakcja otwarta bezczynnie dłużej niż limit | Skróć transakcje; nie trzymaj sesji przez wywołania zewnętrzne |
| `SELECT ... FOR UPDATE` nie blokuje niczego | Uruchamiasz na SQLite — dialekt pomija `FOR UPDATE` | Testuj współbieżność na PostgreSQL |
| Wynik operacji jest poprawny w testach, ale w produkcji dane się „gubią” | Testy sekwencyjne; brak `Barrier` i `sleep` w krytycznym miejscu | Dodaj test współbieżności z 10+ wątkami i sztucznym opóźnieniem |
| `InvalidRequestError: This connection is already in a transaction` | Próba zmiany izolacji po rozpoczęciu transakcji | Ustaw izolację przez `session.connection(execution_options=...)` **przed** pierwszym zapytaniem |
| `InvalidRequestError: Connection is closed` | Użycie `conn.execution_options()` bez przypisania wyniku | `conn = conn.execution_options(...)` |
| Operacja wykonana dwa razy (dwie płatności, dwa e-maile) | Brak idempotencji; retry powtórzył efekt uboczny | Klucz idempotencji + unikalny indeks; retry tylko na operacji bazodanowej |
| Bloat tabeli, `VACUUM` nie nadąża | Długie transakcje i sesje `idle in transaction` | `idle_in_transaction_session_timeout`; krótkie transakcje |
| Deadlock pojawia się mimo sortowania `id` w Pythonie | Blokowanie nie idzie w posortowanej kolejności (`get()` w pętli po niesortowanej liście) | Jeden `SELECT ... WHERE id IN (...) ORDER BY id FOR UPDATE` |
| Retry nie działa, błędy się mnożą | Ponawianie na tej samej, „zatrutej” sesji | Świeża sesja na każdą próbę albo `rollback()` przed ponowieniem |
| `Float` w kwotach daje błędy zaokrągleń przy obciążeniach | Typ `Float` zamiast `Numeric` | `Numeric(12, 2)`; patrz moduł 12 |

---

## Słowniczek modułu

| Termin (EN) | Polski odpowiednik | Wyjaśnienie |
|---|---|---|
| ACID | ACID | Cztery obietnice transakcji: atomowość, spójność, izolacja, trwałość |
| Atomicity | atomowość | Wszystko albo nic — brak stanów pośrednich |
| Consistency | spójność | Transakcja przenosi bazę ze stanu poprawnego do poprawnego |
| Isolation | izolacja | Transakcje nie widzą nawzajem swoich niezakończonych zmian |
| Durability | trwałość | Po `COMMIT` dane przetrwają awarię |
| Isolation level | poziom izolacji | Ustawienie określające, jak silnie transakcje są od siebie odseparowane |
| Dirty read | brudny odczyt | Odczyt danych z niezatwierdzonej transakcji |
| Non-repeatable read | odczyt niepowtarzalny | Dwa identyczne odczyty w jednej transakcji dają różne wartości |
| Phantom read | odczyt widmowy | Dwa identyczne odczyty zakresowe dają różną liczbę wierszy |
| Write skew | przekrzywienie zapisu | Dwie transakcje łamią niezmiennik, zapisując różne wiersze |
| Lost update | zgubiona aktualizacja | Dwie transakcje nadpisują się nawzajem w schemacie odczyt-zmiana-zapis |
| Snapshot isolation | izolacja migawkowa | Transakcja widzi jedną, niezmienną migawkę bazy |
| MVCC | wielowersyjna kontrola współbieżności | Zmiany tworzą nowe wersje wierszy zamiast nadpisywać w miejscu |
| SAVEPOINT | punkt zapisu | Znacznik w transakcji, do którego można się cofnąć bez jej kończenia |
| Two-phase commit (2PC) | zatwierdzanie dwufazowe | Koordynacja commitu na kilku bazach jednocześnie |
| Pessimistic locking | blokada pesymistyczna | Zakładasz konflikt i blokujesz z góry (`FOR UPDATE`) |
| Optimistic locking | blokada optymistyczna | Nie blokujesz, sprawdzasz przy zapisie (`version_id_col`) |
| `FOR UPDATE` | — | Klauzula SQL blokująca wiersze do końca transakcji |
| `SKIP LOCKED` | — | Pomija zablokowane wiersze zamiast czekać — wzorzec kolejki |
| `NOWAIT` | — | Nie czekaj na blokadę — natychmiastowy błąd |
| `version_id_col` | kolumna wersji | Kolumna, którą SQLAlchemy inkrementuje i weryfikuje przy `UPDATE` |
| `StaleDataError` | — | Wyjątek: wiersz zmieniony przez kogoś innego (blokada optymistyczna) |
| Idempotency key | klucz idempotencji | Unikalny identyfikator operacji gwarantujący jednokrotne wykonanie |
| `ON CONFLICT` | — | PostgreSQL/SQLite: obsługa konfliktu unikalności przy `INSERT` |
| `RETURNING` | — | Zwrócenie danych ze zmodyfikowanych wierszy w jednym poleceniu |
| Deadlock | zakleszczenie | Cykl oczekiwań na blokady; baza zabija jedną transakcję |
| Backoff | wycofywanie opóźnienia | Rosnące opóźnienie między ponowieniami |
| Jitter | rozrzut losowy | Losowa korekta opóźnienia, żeby nie synchronizować ponowień |
| `idle in transaction` | bezczynny w transakcji | Sesja z otwartą transakcją, która nic nie robi |
| Bloat | rozdęcie | Nadmiarowe, nieusuwalne wersje wierszy w PostgreSQL |
| VACUUM | — | Proces PostgreSQL usuwający nieaktualne wersje wierszy |
| WAL | dziennik zapisu wyprzedzającego | Mechanizm trwałości; w SQLite również tryb pracy |
| Connection pool | pula połączeń | Zbiór gotowych połączeń wielokrotnego użytku |
| `pool_pre_ping` | — | Sprawdzenie żywotności połączenia przed wydaniem |
| Connection zombie | połączenie zombie | Połączenie „otwarte” po stronie klienta, nieistniejące w bazie |
| PgBouncer | — | Zewnętrzny pooler dla PostgreSQL |
| Transaction pooling | poolowanie transakcyjne | Tryb PgBouncera zwracający połączenie po każdej transakcji |
| Race condition | wyścig | Błąd wynikający z nieprzewidywalnej kolejności operacji równoległych |
| Barrier | bariera | Narzędzie synchronizacji wątków — wszyscy startują razem |
| Re-check | ponowna ocena warunku | PostgreSQL ocenia `WHERE` ponownie po odblokowaniu wiersza |
| `statement_timeout` | limit czasu polecenia | Maksymalny czas wykonania jednego polecenia w PostgreSQL |
| `lock_timeout` | limit czasu blokady | Maksymalny czas oczekiwania na blokadę |
| SSI | serializable snapshot isolation | Metoda realizacji `SERIALIZABLE` w PostgreSQL |

---

## Dalsze czytanie

**SQLAlchemy — dokumentacja oficjalna (wersja 2.0):**

- Transactions and Connection Management — izolacja, savepointy, 2PC, `join_transaction_mode`: <https://docs.sqlalchemy.org/en/20/orm/session_transaction.html>
- Working with Engines and Connections — `execution_options`, `isolation_level`, pula: <https://docs.sqlalchemy.org/en/20/core/connections.html>
- Connection Pooling — wszystkie parametry puli: <https://docs.sqlalchemy.org/en/20/core/pooling.html>
- Configuring a Version Counter — blokada optymistyczna: <https://docs.sqlalchemy.org/en/20/orm/versioning.html>
- Row Locking with `with_for_update()` — pełna lista opcji: <https://docs.sqlalchemy.org/en/20/orm/queryguide/select.html>
- SQLite dialect — izolacja, savepointy, `BEGIN IMMEDIATE`: <https://docs.sqlalchemy.org/en/20/dialects/sqlite.html>
- PostgreSQL dialect — `ON CONFLICT`, typy specyficzne: <https://docs.sqlalchemy.org/en/20/dialects/postgresql.html>
- Session API — `join_transaction_mode`, `twophase`, parametry konstruktora: <https://docs.sqlalchemy.org/en/20/orm/session_api.html>

**SQLAlchemy 2.1 (nowości):**

- What's New in SQLAlchemy 2.1: <https://docs.sqlalchemy.org/en/21/changelog/whatsnew_21.html>
- Transactions and Connection Management (2.1): <https://docs.sqlalchemy.org/en/21/orm/session_transaction.html>

**PostgreSQL:**

- Transaction Isolation — tabela anomalii i poziomów: <https://www.postgresql.org/docs/current/transaction-iso.html>
- Explicit Locking — `FOR UPDATE`, `SKIP LOCKED`, `NOWAIT`: <https://www.postgresql.org/docs/current/explicit-locking.html>
- Client Connection Defaults — timeouty: <https://www.postgresql.org/docs/current/runtime-config-client.html>
- Routine Vacuuming — bloat i długie transakcje: <https://www.postgresql.org/docs/current/routine-vacuuming.html>

**SQLite:**

- Transactions — `BEGIN IMMEDIATE`, `BEGIN EXCLUSIVE`: <https://www.sqlite.org/lang_transaction.html>
- Write-Ahead Logging: <https://www.sqlite.org/wal.html>
- File Locking And Concurrency: <https://www.sqlite.org/lockingv3.html>

**Alembic (kontekst migracji z modułu 16):**

- Dokumentacja Alembic: <https://alembic.sqlalchemy.org/en/latest/>

---

## Co dalej

Nauczyłeś się, że bezpieczeństwo danych pod obciążeniem jest kwestią **projektu**, nie „magicznego ustawienia”: wybierasz poziom izolacji, mechanizm blokowania i strategię obsługi konfliktu świadomie, mierząc przy tym koszt. Wiesz już, że SQLite serializuje zapisy, że `FOR UPDATE` na nim nie działa, że retry wymaga świeżej sesji i że unikalny indeks jest jedyną prawdziwą gwarancją idempotencji.

W module [`15_asynchronicznosc.md`](15_asynchronicznosc.md) zajmiemy się drugim wymiarem współbieżności: **współbieżnością w obrębie jednego procesu**. Zobaczysz, jak `AsyncEngine` i `AsyncSession` pozwalają obsłużyć wiele zapytań równolegle bez blokowania pętli zdarzeń, dlaczego `asyncio.gather` z jedną sesją to błąd, skąd bierze się `MissingGreenlet` i jak uczciwie zmierzyć, czy async rzeczywiście przyspiesza Twoją aplikację. Wszystkie mechanizmy blokowania z tego modułu pozostaną aktualne — zmieni się tylko sposób, w jaki je wywołujesz.

<!-- koniec modułu 14 -->