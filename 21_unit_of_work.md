# Moduł 21 — Unit of Work i zarządzanie sesją

W tym module nauczysz się **świadomie wyznaczać granice transakcji** w aplikacji oraz **zarządzać cyklem życia sesji** w każdym typie programu: skrypcie, aplikacji webowej, workerze kolejki i testach. Poznasz wzorzec **Unit of Work** (jednostka pracy) i zobaczysz, że w SQLAlchemy masz już jego gotową implementację — brakuje tylko miejsca w architekturze, w którym się go używa. Nauczysz się też, dlaczego wysyłanie e-maila **wewnątrz** transakcji to klasyczny błąd i jak rozwiązać go wzorcem *outbox*. Efektem będzie kompletny, typowany kod `UnitOfWork`, dwa serwisy biznesowe z prawdziwymi niezmiennikami i test, który dowodzi, że błąd faktycznie wycofuje transakcję.

---

**Poziom:** 🔴 architektoniczny
**Czas:** ~180 minut
**Wymagania wstępne:**
- [Moduł 08 — `Session`, cykl życia obiektu, Unit of Work](08_sesja_cykl_zycia.md) — bez tego modułu ten materiał nie ma sensu
- [Moduł 14 — Transakcje, izolacja, współbieżność](14_transakcje_i_wspolbieznosc.md) — poziomy izolacji, blokady, retry
- [Moduł 18 — Testowanie kodu ze SQLAlchemy](18_testowanie.md) — fixture `db_session`
- [Moduł 19 — Warstwy, modele domenowe, Data Mapper](19_warstwy_i_data_mapper.md) — gdzie kończy się domena, a zaczyna infrastruktura
- [Moduł 20 — Wzorzec Repository](20_repository.md) — repozytoria, protokoły, paginacja

**Czego dotyczy plik:** architektury i wzorców. To moduł, w którym dotychczasowe techniki (`Session`, `commit`, repozytoria) układamy w reguły, których zespół może przestrzegać.

---

## Spis treści

- [1. Jednostka pracy — wzorzec, który już masz](#1-jednostka-pracy--wzorzec-który-już-masz)
- [2. Transakcja jako granica przypadku użycia](#2-transakcja-jako-granica-przypadku-użycia)
- [3. Implementacja `UnitOfWork`](#3-implementacja-unitofwork)
- [4. Kompozycja UoW z repozytoriami](#4-kompozycja-uow-z-repozytoriami)
- [5. Cykl życia sesji w różnych kontekstach](#5-cykl-życia-sesji-w-różnych-kontekstach)
- [6. `sessionmaker`, `scoped_session` i wątki](#6-sessionmaker-scoped_session-i-wątki)
- [7. Wstrzykiwanie zależności bez frameworka](#7-wstrzykiwanie-zależności-bez-frameworka)
- [8. Granice transakcji: co wolno i jak długo](#8-granice-transakcji-co-wolno-i-jak-długo)
- [9. Obsługa błędów i ponowienia](#9-obsługa-błędów-i-ponowienia)
- [10. Alternatywy: kiedy UoW to przesada](#10-alternatywy-kiedy-uow-to-przesada)
- [11. Diagram przepływu requestu](#11-diagram-przepływu-requestu)
- [12. Testowanie `UnitOfWork`](#12-testowanie-unitofwork)
- [Podsumowanie](#podsumowanie)
- [Ćwiczenia](#ćwiczenia)
- [Najczęstsze błędy i jak je czytać](#najczęstsze-błędy-i-jak-je-czytać)
- [Słowniczek modułu](#słowniczek-modułu)
- [Dalsze czytanie](#dalsze-czytanie)
- [Co dalej](#co-dalej)

---

## 1. Jednostka pracy — wzorzec, który już masz

### 1.1. Problem, który rozwiązujemy

Wyobraź sobie, że piszesz procedurę przyjęcia zwrotu książki w bibliotece. Musisz zrobić **trzy rzeczy naraz**:

1. oznaczyć wypożyczenie jako zwrócone (`loans.returned_at = teraz`),
2. zwiększyć stan magazynowy książki (`books.stock = stock + 1`),
3. dopisać wpis do rejestru zdarzeń (`outbox_events`).

Jeżeli po kroku 1 serwer padnie, a kroki 2 i 3 się nie wykonają, Twoja baza twierdzi, że książka wróciła, ale magazyn nadal pokazuje ją jako wypożyczoną. Rekord danych jest **niespójny**. To nie jest rzadki scenariusz: awaria procesu, restart kontenera, wyjątek w kodzie, przerwane połączenie sieciowe — wszystko to zdarza się w produkcji codziennie.

> 💡 **Analogia — jedna koperta na całą korespondencję**
> Wyobraź sobie, że sekretariat wysyła do urzędu trzy pisma w jednej sprawie. Można iść na pocztę trzy razy i wysłać każde osobno — ale wtedy istnieje ryzyko, że pierwsze dojdzie, a drugie zginie po drodze. Urząd dostanie połowę informacji i podejmie błędną decyzję. Drugie podejście: wkładasz wszystkie trzy pisma do **jednej koperty** i wysyłasz raz. Koperta dojdzie w całości albo wcale. Transakcja to taka koperta: albo wszystkie operacje zostaną zatwierdzone, albo żadna.

Wzorzec **Unit of Work** (jednostka pracy) to odpowiedź na to wyzwanie: skupiamy wszystkie zmiany z jednego logicznego zadania w jednym obiekcie, który wie, kiedy je wysłać i kiedy je wycofać.

### 1.2. Dobra wiadomość: `Session` już tym jest

W [module 08](08_sesja_cykl_zycia.md) mówiliśmy, że `Session` to „notes naczelnika zmiany na budowie”: zapisuje intencje (`add`, modyfikacje atrybutów, `delete`), a wypuszcza je do bazy w kontrolowanych momentach (`flush`). Dzisiaj dodamy drugą połowę obrazu.

`Session` w SQLAlchemy jest **kompletną implementacją wzorca Unit of Work**. W jednym obiekcie masz:

| Element wzorca Unit of Work | Odpowiednik w `Session` |
|---|---|
| Rejestr nowych obiektów | obiekty w stanie `pending` po `session.add()` |
| Rejestr zmienionych obiektów | obiekty `dirty`, śledzone przez *identity map* |
| Rejestr usuniętych obiektów | obiekty `deleted` po `session.delete()` |
| Kolejność zapisu zależności | sortowanie operacji (INSERT przed UPDATE) podczas `flush()` |
| Zatwierdzenie całości | `session.commit()` |
| Wycofanie całości | `session.rollback()` |
| Mapowanie tożsamości (jeden obiekt = jeden wiersz) | *identity map* |
| Granica transakcji | kontekst `session.begin()` / `engine.begin()` |

> 🧠 **Dlaczego tak jest** — Klasa `Session` nie jest „klientem bazy” ani „połączeniem”. Jest **buforem zmian + menedżerem transakcji**. Dlatego w tym module nie tworzymy nowego mechanizmu — tworzymy **reguły i opakowanie**, które mówią, gdzie w architekturze ten bufor powstaje, kto go zatwierdza i kto go zamyka.

### 1.3. Czego brakuje? Granica

Skoro `Session` już jest UoW, po co jeszcze jedna klasa o nazwie `UnitOfWork`? Bo sama `Session` nie odpowiada na pytania **architektoniczne**:

- Kto ją tworzy i kiedy?
- Czy repozytorium może ją zatwierdzać?
- Co się dzieje, gdy jedno repozytorium dostanie sesję A, a drugie sesję B?
- Gdzie kończy się transakcja: w serwisie, w endpoincie, w middleware?
- Co się dzieje przy wyjątku — czy ktoś pamięta o `rollback()`?

> ⚠️ **Pułapka — „dwie sesje, jedna transakcja”**
> Najczęstszy błąd architektoniczny w projektach z repozytoriami: repozytorium samo tworzy `Session` w konstruktorze. Wtedy `BookRepository()` i `MemberRepository()` mają **dwie różne sesje**, czyli **dwie różne transakcje**. Zapiszesz książkę, potem członka, wyjątek poleci przy trzecim obiekcie i… książka pozostanie zapisana. Nie ma atomowości, bo nie ma jednej koperty.
>
> Reguła na cały moduł: **jedna sesja na jeden przypadek użycia**. Zawsze.

### 1.4. Warstwy odpowiedzialności

Zanim napiszemy kod, ustalmy podział pracy. To najważniejsza tabela w tym module.

| Warstwa | Odpowiedzialność | Czego NIE robi |
|---|---|---|
| Model ORM | kształt danych, relacje, mapowanie typów | nie wie nic o transakcjach |
| Repozytorium | zapytania i operacje w obrębie jednej encji | **nie commituje**, nie tworzy sesji |
| UoW | granica transakcji, wspólna sesja dla repozytoriów | nie zawiera logiki biznesowej |
| Serwis (przypadek użycia) | reguły biznesowe, kolejność kroków | **nie commituje** (albo commituje jawnie — patrz §3.6) |
| Endpoint / CLI / worker | wejście i wyjście, mapowanie na DTO | **nie zawiera SQL ani reguł domenowych** |

> 💡 **Analogia — ekipa remontowa**
> Repozytorium to fachowiec, który umie wykonać konkretną pracę („podać ostatnie zamówienia klienta”). Serwis to kierownik budowy („wymień okna, potem pomaluj”). UoW to **kierownik zmiany**, który podpisuje protokół odbioru na koniec — albo zrywa cały protokół, gdy coś nie gra. Endpoint to recepcja, która przyjmuje zlecenie od klienta i przekazuje dalej.

> 🔬 **Pod maską — co dokładnie robi `session.commit()`**
> ```sql
> BEGIN;                         -- autobegin, start transakcji
> UPDATE loans SET returned_at=? WHERE loans.id = ?;
> UPDATE books SET stock = stock + ? WHERE books.id = ?;
> INSERT INTO outbox_events (event_type, payload, created_at) VALUES (?, ?, now());
> COMMIT;                        -- dopiero tutaj zmiany są trwałe
> ```
> Trzy instrukcje, jedna `BEGIN`, jedno `COMMIT`. To jest atomowość w praktyce. Gdyby w połowie poleciał wyjątek, a kod wykonał `ROLLBACK`, baza wróciłaby do stanu sprzed `BEGIN` — bez żadnych śladów częściowych zmian.

---

## 2. Transakcja jako granica przypadku użycia

### 2.1. Przypadek użycia to jednostka sensu

W architekturze warstwowej (moduł 19) przez „przypadek użycia” rozumiemy **jedną operację biznesową wywołaną przez użytkownika lub proces**. Przykłady:

- „Zarejestruj nowego członka biblioteki”
- „Złóż zamówienie na trzy egzemplarze”
- „Przyjmij zwrot i nalicz karę za przetrzymanie”
- „Wyślij przypomnienia o zbliżającym się terminie”

Każdy taki przypadek użycia to **jedna transakcja**. Nie pół, nie dwie, nie dwadzieścia. To reguła, którą warto zapisać w kontrybucji do repozytorium kodu jako konwencję projektu.

> 🧠 **Dlaczego tak jest** — Transakcja oznacza „stan, w którym system wygląda spójnie”. Jeżeli w połowie operacji biznesowej ktoś inny czyta bazę, zobaczy świat niedokończony: zamówienie istnieje, ale nie ma pozycji. „Częściowo złożone zamówienie” to byt, którego w domenie nie ma. Transakcja chroni **niezmienniki biznesowe** — reguły, które zawsze muszą być prawdziwe.

### 2.2. Klasyczny błąd: e-mail w środku transakcji

Rozważmy taki kod — pozornie rozsądny:

```python
# examples/21_anti_email_in_tx.py
# ⛔ TO JEST PRZYKŁAD BŁĘDU — nie kopiuj do projektu

def register_member_bad(session: Session, name: str, email: str) -> Member:
    member = Member(name=name, email=email)
    session.add(member)
    session.flush()  # żeby dostać member.id

    # ⛔ Wywołanie usługi zewnętrznej WEWNĄTRZ otwartej transakcji
    mailer.send_welcome_email(to=email, member_id=member.id)

    session.commit()
    return member
```

Co tu jest nie tak? Trzy rzeczy, każda poważna:

1. **Czas trwania transakcji.** Wywołanie HTTP do serwera SMTP trwa 200 ms – 5 s. Przez cały ten czas transakcja jest otwarta, a wiersz w `members` zablokowany. Przy 50 równoległych rejestracjach kolejka rośnie, pula połączeń się wyczerpuje, a użytkownicy widzą 504.
2. **Brak atomowości mimo pozorów.** Jeśli `send_welcome_email` się powiedzie, ale `commit()` rzuci `IntegrityError` (np. e-mail nie jest unikalny — bo sprawdziliśmy to wcześniej, ale między sprawdzeniem a zapisem ktoś inny dodał ten sam adres), to użytkownik dostanie e-mail z linkiem aktywacyjnym do konta, **którego nie ma w bazie**.
3. **Odwrotny scenariusz.** Jeśli `send_welcome_email` rzuci wyjątek, transakcja się wycofa — konto też zniknie. Ale co, jeśli mail faktycznie dotarł, a wyjątek powstał tylko przy zapisie odpowiedzi? Użytkownik już ma e-mail, konto nie istnieje.

> 💡 **Analogia — list i paczka**
> Transakcja to moment pakowania paczki. Nie dzwonisz do kuriera w połowie pakowania, nie prosisz go, żeby poczekał 5 sekund przy taśmie, aż skończysz — dzwonisz **po** zamknięciu paczki i nadaniu jej numeru. Inaczej kurier odjeżdża z pustym numerem, a klient czeka na przesyłkę, której nikt nie nada.

### 2.3. Wzorzec outbox (skrzynka nadawcza)

Rozwiązanie: zapisujemy **intencję** wysłania e-maila jako wiersz w tabeli w tej samej transakcji, a faktyczną wysyłkę wykonuje osobny proces.

```text
┌─────────────────────────────────────────────────────────────────────┐
│ TRANSAKCJA (jedna koperta)                                          │
│                                                                     │
│   INSERT INTO members (name, email)  ────────────────►              │
│   INSERT INTO outbox_events (event_type='MemberRegistered', …)      │
│                                                                     │
│                          COMMIT                                     │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│ OSOBNY PROCES: relayer outboxa                                      │
│   SELECT * FROM outbox_events WHERE processed_at IS NULL            │
│   → wysyła e-mail                                                    │
│   → UPDATE outbox_events SET processed_at = now()                    │
└─────────────────────────────────────────────────────────────────────┘
```

Zalety:

- **Atomowość** — albo istnieje konto i istnieje zdarzenie, albo nie istnieje nic.
- **Krótka transakcja** — brak czekania na HTTP.
- **Powtarzalność** — jeśli wysyłka się nie powiedzie, wiersz `processed_at IS NULL` zostanie przetworzony ponownie. Relayer **musi** być idempotentny (albo przynajmniej obsługiwać duplikaty) — to jego kontrakt.
- **Audytowalność** — masz w bazie dowód, że zdarzenie zostało zapisane w danym momencie.

> ⚠️ **Pułapka — outbox nie jest darmowy**
> Outbox dodaje tabelę, relayera, obsługę duplikatów i monitoring opóźnień. W małym projekcie, gdzie „wysyłka maila” to `smtp.send` na końcu skryptu, outbox może być przesadą. Decyzję uzasadnia się **konsekwencją niespójności**: przy rejestracjach i płatnościach użytkownik zauważy problem, przy nocnym raporcie — nie.

Zauważ jeszcze jedną rzecz istotną dla tego modułu: **kolejność operacji wewnątrz transakcji nie ma znaczenia dla atomowości**, ale ma dla czasu życia blokad. Dlatego dobre praktyki to:

1. Najpierw operacje na danych „ryzykownych” (te, które mogą rzucić konflikt).
2. Potem wypełnianie pól pochodnych i wpisy audytowe.
3. Na samym końcu wiersz outboxa.
4. `COMMIT` jak najszybciej po ostatnim `UPDATE`.

---

## 3. Implementacja `UnitOfWork`

### 3.1. Model bazy, na którym pracujemy

Zbudujmy spójną domenę wypożyczalni. Trzymamy się jej we wszystkich przykładach tego modułu.

```python
# app/models.py
from __future__ import annotations

from datetime import datetime
from decimal import Decimal

from sqlalchemy import CheckConstraint, DateTime, ForeignKey, Numeric, String, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    """Wspólna klasa bazowa — jedna instancja MetaData dla całej aplikacji."""


class Book(Base):
    __tablename__ = "books"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    isbn: Mapped[str] = mapped_column(String(20), unique=True)
    price: Mapped[Decimal] = mapped_column(Numeric(10, 2))
    stock: Mapped[int] = mapped_column(default=0)

    __table_args__ = (
        CheckConstraint("stock >= 0", name="ck_books_stock_non_negative"),
    )

    def __repr__(self) -> str:
        return f"Book(id={self.id!r}, title={self.title!r}, stock={self.stock!r})"


class Member(Base):
    __tablename__ = "members"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(120))
    email: Mapped[str] = mapped_column(String(255), unique=True)

    def __repr__(self) -> str:
        return f"Member(id={self.id!r}, email={self.email!r})"


class Order(Base):
    __tablename__ = "orders"

    id: Mapped[int] = mapped_column(primary_key=True)
    member_id: Mapped[int] = mapped_column(ForeignKey("members.id"))
    created_at: Mapped[datetime] = mapped_column(
        DateTime, server_default=func.now()
    )

    member: Mapped[Member] = relationship()
    items: Mapped[list[OrderItem]] = relationship(
        back_populates="order", cascade="all, delete-orphan"
    )


class OrderItem(Base):
    __tablename__ = "order_items"

    id: Mapped[int] = mapped_column(primary_key=True)
    order_id: Mapped[int] = mapped_column(ForeignKey("orders.id"))
    book_id: Mapped[int] = mapped_column(ForeignKey("books.id"))
    quantity: Mapped[int] = mapped_column()
    unit_price: Mapped[Decimal] = mapped_column(Numeric(10, 2))

    order: Mapped[Order] = relationship(back_populates="items")
    book: Mapped[Book] = relationship()


class OutboxEvent(Base):
    """Skrzynka nadawcza: intencje efektów ubocznych zapisane w tej samej transakcji."""

    __tablename__ = "outbox_events"

    id: Mapped[int] = mapped_column(primary_key=True)
    event_type: Mapped[str] = mapped_column(String(50))
    payload: Mapped[str] = mapped_column(String)  # JSON jako tekst — dla zwięzłości
    created_at: Mapped[datetime] = mapped_column(DateTime, server_default=func.now())
    processed_at: Mapped[datetime | None] = mapped_column(DateTime, nullable=True)
```

> 🧠 **Dlaczego `CheckConstraint` na `stock`** to nie ozdoba — to **ostatnia linia obrony** przed błędem aplikacji. Jeśli Twój serwis zapomni sprawdzić stan, baza odrzuci ujemny zapas i cała transakcja się wycofa. Niezmiennik trzymany w schemacie jest wartowniejszy niż niezmiennik trzymany w głowie programisty.

### 3.2. Fabryka sesji raz na proces

Zanim zbudujemy UoW, potrzebujemy jednego miejsca, które tworzy `Engine` i fabrykę sesji.

```python
# app/db.py
from __future__ import annotations

from collections.abc import Iterator
from contextlib import contextmanager
from pathlib import Path

from sqlalchemy import create_engine
from sqlalchemy.orm import Session, sessionmaker

DB_PATH = Path("library.db")
DATABASE_URL = f"sqlite:///{DB_PATH}"

# Engine tworzymy RAZ na proces — jest drogi i zarządza pulą połączeń (moduł 02).
engine = create_engine(
    DATABASE_URL,
    echo=False,          # True tylko lokalnie do debugowania
    pool_pre_ping=True,  # wykrywa martwe połączenia po przerwie w sieci
)

# sessionmaker to fabryka sesji: woła ją każdy, kto potrzebuje NOWEJ sesji.
session_factory: sessionmaker[Session] = sessionmaker(
    bind=engine,
    expire_on_commit=False,  # omawiamy w §8.4 — celowy wybór dla warstwy API
)


@contextmanager
def session_scope() -> Iterator[Session]:
    """Minimalny wariant: sesja + jawna granica transakcji.

    To jeszcze NIE jest Unit of Work, ale pokazuje wzorzec, z którego wyrasta.
    """
    session = session_factory()
    try:
        with session.begin():     # BEGIN … COMMIT / ROLLBACK w jednym miejscu
            yield session
    finally:
        session.close()           # ZAWSZE zwalniamy połączenie do puli
```

Points of interest: `sessionmaker[Session]` is a generic in 2.0 (so typing works), `with session.begin()` opens the transaction context and commits/rollbacks automatically, and `finally: session.close()` is what prevents pool leaks.

> 🔬 **Pod maską — co wysyła `session_scope()`**
> ```sql
> -- przy wejściu do with session.begin():
> BEGIN (implicit)
> -- ... twoje zapytania ...
> -- przy normalnym wyjściu:
> COMMIT
> -- przy wyjątku:
> ROLLBACK
> ```
> Zauważ: `session_scope` nie musi wołać `commit()` ani `rollback()` ręcznie. Kontekst `session.begin()` to robi. To jest dokładnie ten mechanizm, który opakujemy klasą.

> 🆕 **SQLAlchemy 2.1 — autoflush działa bezwarunkowo**
> W 2.0 istniały nisze, w których autoflush (automatyczne wypchnięcie oczekujących zmian przed odczytem) mógł zostać pominięty. W 2.1 `Session` realizuje autoflush konsekwentnie. Jeśli opierasz logikę na „autoflush nie zadziała w tym konkretnym miejscu”, po aktualizacji dostaniesz inne zapytania i potencjalnie inne wyniki. Więcej kontroli daje jawny `flush()` przed kodem, który **musi** widzieć świeże dane.

### 3.3. Repozytoria — wariant minimalny

W [module 20](20_repository.md) omawialiśmy protokoły i repozytoria generyczne. Tutaj pokazujemy wariant produkcyjnie użyteczny: **jedna baza generyczna + konkretne klasy z własnymi metodami**.

```python
# app/repositories.py
from __future__ import annotations

from collections.abc import Sequence
from decimal import Decimal
from typing import Generic, TypeVar

from sqlalchemy import select
from sqlalchemy.orm import Session

from app.models import Base, Book, Member, Order, OutboxEvent

ModelT = TypeVar("ModelT", bound=Base)


class SqlAlchemyRepository(Generic[ModelT]):
    """Baza dla repozytoriów. NIE commituje i NIE tworzy sesji — dostaje ją gotową."""

    model: type[ModelT]  # nadpisywane w klasach potomnych

    def __init__(self, session: Session) -> None:
        self.session = session

    def get(self, pk: int) -> ModelT | None:
        return self.session.get(self.model, pk)

    def add(self, entity: ModelT) -> None:
        """Dodaje do jednostki pracy. Zapisu NIE wykonuje — patrz §3.4."""
        self.session.add(entity)

    def list_all(self) -> Sequence[ModelT]:
        return self.session.scalars(select(self.model)).all()


class BookRepository(SqlAlchemyRepository[Book]):
    model = Book

    def get_by_isbn(self, isbn: str) -> Book | None:
        return self.session.scalars(select(Book).where(Book.isbn == isbn)).one_or_none()

    def get_locked(self, book_id: int) -> Book | None:
        """Blokada pesymistyczna wiersza (SELECT ... FOR UPDATE).

        UWAGA: SQLite nie wspiera FOR UPDATE — dialekt po cichu go pomija.
        Na PostgreSQL to realna blokada wiersza.
        """
        return self.session.scalars(
            select(Book).where(Book.id == book_id).with_for_update()
        ).one_or_none()

    def decrease_stock(self, book_id: int, quantity: int) -> None:
        book = self.get_locked(book_id)
        if book is None:
            raise BookNotFound(book_id)
        if book.stock < quantity:
            raise InsufficientStock(book_id, requested=quantity, available=book.stock)
        book.stock -= quantity


class MemberRepository(SqlAlchemyRepository[Member]):
    model = Member

    def get_by_email(self, email: str) -> Member | None:
        return self.session.scalars(
            select(Member).where(Member.email == email)
        ).one_or_none()


class OrderRepository(SqlAlchemyRepository[Order]):
    model = Order

    def add_with_items(
        self, member: Member, lines: Sequence[tuple[Book, int]]
    ) -> Order:
        order = Order(member=member)
        for book, quantity in lines:
            order.items.append(OrderItem(book=book, quantity=quantity))
        self.session.add(order)
        return order


class OutboxRepository(SqlAlchemyRepository[OutboxEvent]):
    model = OutboxEvent

    def emit(self, event_type: str, payload: str) -> OutboxEvent:
        event = OutboxEvent(event_type=event_type, payload=payload)
        self.session.add(event)
        return event
```

> ⛔ **Błąd, którego za chwilę nie popełnimy**
> Zauważ, że **żadne repozytorium nie ma metody `commit()`**. To nie przypadek — to kontrakt. Repozytorium wykonuje operacje w bieżącej sesji, a decyzję o zatwierdzeniu podejmuje UoW. Jeżeli repo zacznie commitować, natychmiast tracisz możliwość złożenia operacji z kilku repozytoriów w jedną atomową całość.

### 3.4. Klasy wyjątków domenowych

Błędy infrastruktury (`IntegrityError`) nie powinny przeciekać do API (moduł 22). Definiujemy własną hierarchię:

```python
# app/errors.py
class DomainError(Exception):
    """Baza dla wszystkich błędów domenowych."""


class BookNotFound(DomainError):
    def __init__(self, book_id: int) -> None:
        super().__init__(f"Book {book_id} not found")
        self.book_id = book_id


class InsufficientStock(DomainError):
    def __init__(self, book_id: int, requested: int, available: int) -> None:
        super().__init__(
            f"Book {book_id}: requested {requested}, available {available}"
        )
        self.book_id = book_id
        self.requested = requested
        self.available = available


class EmailAlreadyRegistered(DomainError):
    def __init__(self, email: str) -> None:
        super().__init__(f"Email {email} already registered")
        self.email = email


class MemberNotFound(DomainError):
    def __init__(self, member_id: int) -> None:
        super().__init__(f"Member {member_id} not found")
        self.member_id = member_id
```

### 3.5. Wariant A: klasa z `__enter__`/`__exit__`

Teraz gwiazda modułu.

```python
# app/uow.py
from __future__ import annotations

import json
from types import TracebackType
from typing import Self

from sqlalchemy.exc import IntegrityError
from sqlalchemy.orm import Session, sessionmaker

from app.errors import DomainError, EmailAlreadyRegistered
from app.repositories import (
    BookRepository,
    MemberRepository,
    OrderRepository,
    OutboxRepository,
)


class UnitOfWork:
    """Granica transakcji i wspólna sesja dla wszystkich repozytoriów.

    Użycie:
        with UnitOfWork(session_factory) as uow:
            ...operacje na uow.books / uow.members / uow.orders...
            uow.commit()

    Bez `commit()` wszystko zostanie wycofane przy wyjściu z bloku — to celowe.
    """

    def __init__(self, session_factory: sessionmaker[Session]) -> None:
        self._session_factory = session_factory
        self._session: Session | None = None

    # --- kontekst menedżera ------------------------------------------------

    def __enter__(self) -> Self:
        self._session = self._session_factory()
        # Wszystkie repozytoria dostają TĘ SAMĄ sesję — jedna transakcja,
        # jedna identity map (mapa tożsamości).
        self.books = BookRepository(self._session)
        self.members = MemberRepository(self._session)
        self.orders = OrderRepository(self._session)
        self.outbox = OutboxRepository(self._session)
        return self

    def __exit__(
        self,
        exc_type: type[BaseException] | None,
        exc: BaseException | None,
        tb: TracebackType | None,
    ) -> bool:
        """Sprząta po sobie. NIE tłumi wyjątków (zwraca False).

        - wyjątek  -> rollback, brak commita (jeśli nie zostało wywołane wcześniej)
        - sukces   -> nic nie commituje; commit jest jawny (§3.6)
        Zawsze zamyka sesję, żeby połączenie wróciło do puli.
        """
        try:
            if exc_type is not None:
                self.rollback()
            else:
                # Sprzątanie po operacji zakończonej powodzeniem.
                # Jeśli użytkownik nie zawołał commit(), kończymy rollbackiem:
                # „nie powiedziałeś »zatwierdź«, więc nic nie zmieniamy”.
                self.rollback_if_pending()
        finally:
            self.close()
        return False  # nie tłumimy wyjątku — leci wyżej

    # --- operacje ----------------------------------------------------------

    @property
    def session(self) -> Session:
        if self._session is None:
            raise RuntimeError("UnitOfWork nie jest otwarty — użyj 'with'")
        return self._session

    def commit(self) -> None:
        self.session.commit()

    def rollback(self) -> None:
        self.session.rollback()

    def rollback_if_pending(self) -> None:
        """Wycofuje niezatwierdzoną transakcję (jeśli taka jest)."""
        if self._session is not None and self._session.in_transaction():
            self._session.rollback()

    def close(self) -> None:
        if self._session is not None:
            self._session.close()
            self._session = None
```

> 🧠 **Dlaczego `commit()` jest jawny, a nie automatyczny**
> Wersja „commit w `__exit__`, chyba że wyjątek” jest kusząca, ale w praktyce prowadzi do zaskakujących sytuacji: fragment kodu, który miał **tylko poczytać dane**, niepostrzeżenie zatwierdza zmiany dokonane wcześniej w tym samym bloku. Jawny `uow.commit()` jest jak przycisk „Nadaj” na poczcie: nie ma wątpliwości, kto i kiedy podjął decyzję. Dzięki temu każdy `commit` w projekcie jest widoczny jednym `grep`-em.

### 3.6. Wariant B: `@contextmanager`

Krótszy, bez fasadowej klasy. Świetny, gdy repozytoria nie są potrzebne jako atrybuty:

```python
# app/uow_ctx.py
from __future__ import annotations

from collections.abc import Iterator
from contextlib import contextmanager

from sqlalchemy.orm import Session, sessionmaker

from app.repositories import BookRepository, MemberRepository, OutboxRepository


@contextmanager
def unit_of_work(
    session_factory: sessionmaker[Session],
) -> Iterator[tuple[Session, BookRepository, MemberRepository, OutboxRepository]]:
    """Atomowa granica bez klasy.

    Na końcu bloku (bez wyjątku) wykonuje commit. Przy wyjątku — rollback.
    """
    session = session_factory()
    books = BookRepository(session)
    members = MemberRepository(session)
    outbox = OutboxRepository(session)
    try:
        yield session, books, members, outbox
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()
```

Użycie wygląda tak:

```python
with unit_of_work(session_factory) as (session, books, members, outbox):
    member = members.get_by_email("ala@example.com")
    ...
```

**Które wybrać?** Tabela decyzyjna:

| Sytuacja | Rekomendacja |
|---|---|
| Wiele repozytoriów, chcesz autouzupełnianie i jawne nazwy | klasa `UnitOfWork` (wariant A) |
| 1–2 repozytoria, małe serwisy, chcesz mniej kodu | `@contextmanager` (wariant B) |
| Potrzebujesz kilku metod pomocniczych (`commit_if_dirty`, licznik operacji) | klasa |
| Serwis ma zależność „wstrzyknięty UoW” (testy z fake obiektem) | klasa przez protokół (§7) |
| Kod jednorazowy, skrypt migracyjny | `session_scope()` z §3.2 |

> 💡 **Analogia — klucz pod wycieraczką**
> Oba warianty realizują to samo: „zamek z kluczem pod wycieraczką”, czyli jedną kopertę. Klasa daje Ci dodatkową kieszeń na klucz (metody), dekorator mieści się w kieszeni spodni (mniej ceremonii). Żaden nie jest „lepszy” — są dopasowane do rozmiaru projektu.

---

## 4. Kompozycja UoW z repozytoriami

### 4.1. Dlaczego jedna sesja to nie szczegół implementacji

Najważniejsze zdanie tego modułu:

> **Repozytoria wewnątrz jednego UoW muszą dzielić tę samą `Session`.**

Trzy konsekwencje, które trzeba zrozumieć — bo wszystkie trzy bywają źródłem produkcyjnych incydentów.

**Konsekwencja 1: jedna transakcja.** Jeśli `BookRepository` i `MemberRepository` mają tę samą sesję, to `session.commit()` zatwierdzi zmiany z obu naraz. Jeśli mają różne — masz dwie transakcje i żadnej atomowości.

**Konsekwencja 2: jedna mapa tożsamości (identity map).** Dwa różne zapytania zwracają **ten sam obiekt Pythona**, jeżeli dotyczą tego samego wiersza:

```python
# examples/21_identity_map.py
with UnitOfWork(session_factory) as uow:
    book_a = uow.books.get(1)
    book_b = uow.books.get_by_isbn("978-83-0000000-1")

    # To ten sam obiekt, bo oba dotyczą tego samego wiersza w tabeli 'books'.
    assert book_a is book_b  # ✅ przechodzi tylko w obrębie jednej sesji

    book_a.stock = 99
    # book_b.stock też widzi 99 — to nie kopia, to referencja.
```

Gdyby repozytoria miały osobne sesje, `book_a is book_b` byłoby `False`, a modyfikacja jednego obiektu nie byłaby widoczna w drugim. W kodzie z 10 modułami domenowymi prowadzi to do absurdalnych błędów typu „zmieniłem stan, ale w innym miejscu widzę starą wartość”.

**Konsekwencja 3: jedna kolejność operacji.** `flush()` sortuje wszystkie oczekujące zmiany z całej sesji: najpierw INSERT-y rodziców, potem dzieci, potem UPDATE-y. To gwarancja, że klucze obce są poprawne. Przy wielu sesjach każda sortuje tylko swój podzbiór — kolejność staje się niedeterministyczna dla zależności między encjami.

### 4.2. Kontrakt repozytorium w wersji „z UoW”

Warto zapisać kontrakt wprost w `README` projektu albo w docstringu klasy bazowej:

```python
# app/repositories.py — komentarz kontraktu
"""
Kontrakt repozytorium:
  ✅ MOŻE: wykonywać select/insert/update/delete w przekazanej sesji.
  ✅ MOŻE: flush(), jeśli kolejny krok MUSI widzieć dane (np. klucz główny).
  ✅ MOŻE: rzucać wyjątki domenowe (BookNotFound, InsufficientStock).
  ❌ NIE MOŻE: tworzyć sesji ani własnego Engine.
  ❌ NIE MOŻE: wołać commit() ani rollback().
  ❌ NIE MOŻE: wywoływać usług zewnętrznych (HTTP, SMTP, kolejki).
"""
```

Ostatni punkt wraca do §2.2: żadnego I/O poza bazą w środku granicy transakcji.

### 4.3. UoW a DTO i identity map — cicha pułapka

Gdy UoW kończy pracę i sessja jest zamykana, obiekty stają się `detached` (*odłączone*). Dociągnięcie leniwej relacji poza sesją rzuci `DetachedInstanceError` (moduł 11). Dlatego **wewnątrz** UoW trzeba myśleć o tym, jak dane wychodzą na zewnątrz:

```python
# ✅ Bezpiecznie: budujesz DTO WEWNĄTRZ sesji
from dataclasses import dataclass
from decimal import Decimal


@dataclass(frozen=True)
class OrderSummary:
    order_id: int
    member_email: str
    total: Decimal


def build_summary(uow: UnitOfWork, order_id: int) -> OrderSummary:
    order = uow.orders.get(order_id)
    if order is None:
        raise OrderNotFound(order_id)
    # Wszystkie potrzebne pola czytamy TUTAJ, w zasięgu sesji.
    return OrderSummary(
        order_id=order.id,
        member_email=order.member.email,
        total=sum(item.unit_price * item.quantity for item in order.items),
    )
```

To most do modułu 22, gdzie endpoint zwraca DTO, nie encję ORM.

> ⚠️ **Pułapka — „wycieczka po relacje” w pętli**
> Powyższy kod ma ukryte ryzyko: `item.book` byłoby leniwe. Jeśli `OrderSummary` potrzebuje tytułów książek, pętla po `order.items` wygeneruje **N+1 zapytań** (moduł 11). Reguła: jeśli DTO dotyka relacji w pętli, zapytanie w repozytorium musi użyć `selectinload()`. To nie jest kwestia UoW, ale właśnie tutaj — w warstwie przypisanej do jednej transakcji — problemy N+1 najczęściej się rodzą.

---

## 5. Cykl życia sesji w różnych kontekstach

Ten rozdział to odpowiedź na pytanie „gdzie w moim programie umieszczam `with UnitOfWork(...)`”. Odpowiedź zależy od typu programu.

### 5.1. Skrypt / CLI

W skrypcie masz jedną linię wykonania: uruchomiłeś, wykonałeś, skończyłeś. Jedna sesja na cały skrypt, jedna transakcja na logiczny krok.

```python
# examples/21_cli.py
from __future__ import annotations

from app.db import session_factory
from app.uow import UnitOfWork


def main() -> None:
    # Każdy KROK CLI to osobna transakcja — użytkownik widzi częściowe efekty,
    # jeśli wykonał kilka niezależnych poleceń.
    with UnitOfWork(session_factory) as uow:
        book = uow.books.get_by_isbn("978-83-0000000-1")
        if book is None:
            raise SystemExit("Book not found")
        print(f"Before: {book.stock}")
        uow.books.decrease_stock(book.id, quantity=1)
        uow.commit()
        print(f"After:  {book.stock}")


if __name__ == "__main__":
    main()
```

**Uruchomienie:**

```bash
pip install "SQLAlchemy>=2.0"
python -m examples.21_cli
```

> 🧠 **Dlaczego po `commit()` obiekt nadal ma poprawne `stock`** — bo `sessionmaker` ma `expire_on_commit=False` (§8.4). Gdyby było `True` (wartość domyślna SQLAlchemy), odczyt `book.stock` po commicie wywołałby dodatkowe `SELECT` albo rzucił `DetachedInstanceError`, gdyby sesja była już zamknięta.

CLI ma jeszcze jedną cechę odróżniającą: **długość życia procesu**. Jeśli CLI działa 3 sekundy i kończy, wyciek połączenia nie ma znaczenia — proces umiera, a pula znika z nim. But if your CLI runs for hours as a daemon, you must think about it like a web app.

### 5.2. Aplikacja webowa — sesja na request

W aplikacji webowej request przychodzi, jest obsługiwany i wysyła odpowiedź. **Sesja musi żyć dokładnie tyle, co request** i musi zostać zamknięta, nawet jeśli endpoint rzuci wyjątek.

W FastAPI bezpośrednią implementacją jest zależność (dependency) — szczegóły zobaczysz w module 22, tu pokazujemy wariant kanoniczny:

```python
# app/web_deps.py
from __future__ import annotations

from collections.abc import Iterator

from fastapi import Depends
from sqlalchemy.orm import Session, sessionmaker

from app.db import session_factory
from app.uow import UnitOfWork


def get_uow() -> Iterator[UnitOfWork]:
    """Sesja (i UoW) per request. FastAPI zamknie generator po wysłaniu odpowiedzi."""
    with UnitOfWork(session_factory) as uow:
        yield uow
    # Po wyjściu z bloku UoW.__exit__ zawsze woła close() — brak wycieku połączeń.


def get_session(uow: UnitOfWork = Depends(get_uow)) -> Session:
    """Wariant, gdy endpoint potrzebuje gołej sesji, nie UoW."""
    return uow.session
```

Endpoint:

```python
# app/web_routes.py
from fastapi import APIRouter, Depends, HTTPException, status

from app.errors import InsufficientStock
from app.services import place_order
from app.uow import UnitOfWork
from app.web_deps import get_uow

router = APIRouter()


@router.post("/orders", status_code=status.HTTP_201_CREATED)
def create_order(
    member_id: int,
    items: list[tuple[int, int]],
    uow: UnitOfWork = Depends(get_uow),
) -> dict[str, int]:
    try:
        order = place_order(uow, member_id=member_id, items=items)
        uow.commit()
    except InsufficientStock as exc:
        raise HTTPException(status_code=409, detail=str(exc)) from exc
    return {"order_id": order.id}
```

> 🔬 **Pod maską — ile razy tworzony jest `Engine` w tej aplikacji?**
> Raz, przy imporcie `app.db`. `session_factory` też raz. Każdy request dostaje **nową `Session`** i **nowe tymczasowe połączenie z puli** (albo połączenie już istniejące, jeśli pula jest ciepła). To jest właściwy model: `Engine` to długowieczny zasób, `Session` to zasób per jednostka pracy.
>
> ```text
> proces (start)            request #1            request #2            …
>   │                          │                     │
>   ├─ Engine (pula, 5 conn) ──┼─────────────────────┼──────  długowieczny
>   │                          │                     │
>   │                    Session #1            Session #2       krótkowieczne
>   │                    (BEGIN…COMMIT)        (BEGIN…COMMIT)
>   │                    └─ close()            └─ close()
> ```

**Aplikacja async:** w module 15 używa się `async_sessionmaker` i `AsyncSession`. UoW w wariancie async wygląda analożnie, z metodami `async`:

```python
# app/uow_async.py
from __future__ import annotations

from types import TracebackType
from typing import Self

from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker


class AsyncUnitOfWork:
    def __init__(self, session_factory: async_sessionmaker[AsyncSession]) -> None:
        self._session_factory = session_factory
        self._session: AsyncSession | None = None

    async def __aenter__(self) -> Self:
        self._session = self._session_factory()
        # Inicjalizacja repozytoriów — te same obiekty, sesja async.
        return self

    async def __aexit__(
        self,
        exc_type: type[BaseException] | None,
        exc: BaseException | None,
        tb: TracebackType | None,
    ) -> bool:
        if self._session is None:
            return False
        try:
            if exc_type is not None:
                await self._session.rollback()
            elif self._session.in_transaction():
                await self._session.rollback()
        finally:
            await self._session.close()
            self._session = None
        return False

    async def commit(self) -> None:
        if self._session is not None:
            await self._session.commit()
```

> 🆕 **SQLAlchemy 2.1 — async bez `greenlet` „w zestawie”**
> W 2.1 pakiet `greenlet`, który udostępnia most sync↔async, **nie instaluje się automatycznie**. Jeśli budujesz coś na `AsyncSession`, musisz mieć `pip install "sqlalchemy[asyncio]"`. Objaw braku: `MissingGreenlet` albo błąd importu już przy tworzeniu silnika async.

### 5.3. Worker / kolejka zadań

Worker pobiera zadanie z kolejki i wykonuje je w tle. **Jedna sesja na jedno zadanie**, nie jedna na cały proces.

```python
# app/worker.py
from __future__ import annotations

from app.db import session_factory
from app.uow import UnitOfWork


def run_outbox_relay_once() -> int:
    """Przetwarza jedną porcję zdarzeń outboxa — jedno zadanie, jedna transakcja."""
    with UnitOfWork(session_factory) as uow:
        pending = uow.outbox.list_pending(limit=100)
        sent = 0
        for event in pending:
            try:
                fake_deliver(event)  # SMTP / kolejka / webhook
            except TransientDeliveryError:
                continue  # zostawiamy do następnego przebiegu
            event.mark_processed()
            sent += 1
        uow.commit()
    return sent
```

> ⚠️ **Pułapka — jeden ogromny UoW dla całej kolejki**
> „Jedno zadanie, jedna transakcja” nie znaczy „jeden pełny przebieg relayera, jedna transakcja”. Jeżeli w jednej transakcji próbujesz obsłużyć 10 000 zdarzeń, a jedno z nich rzuci wyjątek, cały przebieg przepada i nic nie wysłałeś. Batching (porcje po 100) jest tu obowiązkowy: **porcja = transakcja**, pojedyncze zdarzenie = porażka pomijalna albo dochowana do retry.

Kluczowa różnica względem web jest taka, że w workerze nie ma „użytkownika czekającego na odpowiedź”. Granicę wyznaczasz sam, kierując się tym, co jest jednostką sensu w Twoim zadaniu: jedno zdarzenie, jedna wiadomość, jedna porcja.

### 5.4. Testy — sesja na test

W [module 18](18_testowanie.md) budowaliśmy fixture `db_session` z zewnętrzną transakcją i `begin_nested` (savepoint) — po zakończeniu testu cały savepoint jest wycofywany i baza wraca do stanu sprzed testu. Z UoW wstrzykujesz **inną fabrykę**, aniżeli produkcyjna:

```python
# tests/conftest.py
from __future__ import annotations

from collections.abc import Iterator

import pytest
from sqlalchemy import create_engine, event
from sqlalchemy.orm import Session, sessionmaker
from sqlalchemy.pool import StaticPool

from app.models import Base


@pytest.fixture
def test_session_factory() -> Iterator[sessionmaker[Session]]:
    """SQLite in-memory + StaticPool: jedna baza dla całego testu."""
    engine = create_engine(
        "sqlite:///:memory:",
        connect_args={"check_same_thread": False},
        poolclass=StaticPool,  # ← bez tego każde połączenie to PUSTA baza!
    )
    Base.metadata.create_all(engine)
    factory = sessionmaker(bind=engine, expire_on_commit=False)

    yield factory

    Base.metadata.drop_all(engine)
    engine.dispose()
```

W teście podmieniasz fabrykę, nie mockujesz sesji:

```python
# tests/test_uow.py
from __future__ import annotations

import pytest
from sqlalchemy.orm import sessionmaker, Session

from app.errors import InsufficientStock
from app.models import Book, Member
from app.uow import UnitOfWork


def _seed(factory: sessionmaker[Session]) -> None:
    with UnitOfWork(factory) as uow:
        uow.books.add(Book(title="Clean Code", isbn="978-0-13", price=100, stock=1))
        uow.members.add(Member(name="Ala", email="ala@example.com"))
        uow.commit()


def test_rollback_leaves_no_trace(
    test_session_factory: sessionmaker[Session],
) -> None:
    _seed(test_session_factory)

    with pytest.raises(InsufficientStock):
        with UnitOfWork(test_session_factory) as uow:
            uow.books.decrease_stock(book_id=1, quantity=5)  # tylko 1 na stanie
            uow.commit()

    with UnitOfWork(test_session_factory) as uow:
        book = uow.books.get(1)
        assert book is not None
        assert book.stock == 1  # ✅ stan nienaruszony
```

> 🧠 **Dlaczego `_seed` musi commitować** — fixture nie używa zewnętrznej transakcji w tej wersji, więc dane muszą być realnie zatwierdzone. W wariancie z zewnętrzną transakcją (moduł 18) seed żyje w tym samym savepoincie i nie musi commitować. To, która wersja jest właściwa, zależy od tego, czy testowany kod sam wywołuje `commit()` (a w UoW wywołuje — więc wariant z zewnętrzną transakcją wymaga `join_transaction_mode="create_savepoint"`; omawiamy to w §8.5).

---

## 6. `sessionmaker`, `scoped_session` i wątki

### 6.1. Trzy poziomy

Warto rozróżnić trzy obiekty, które początkujący często myli (i słusznie):

| Obiekt | Czym jest | Kiedy tworzyć | Kiedy zamykać |
|---|---|---|---|
| `Engine` | pula połączeń + konfiguracja dialektu | raz na proces | `engine.dispose()` na zamknięciu aplikacji |
| `sessionmaker` | **fabryka** sesji (bezstanowa, bezpieczna dla wątków) | raz na proces | nie trzeba zamykać |
| `Session` | konkretna sesja = jednostka pracy | raz na przypadek użycia | zawsze `close()` |

> 💡 **Analogia — fabryka rowerów**
> `Engine` to wypożyczalnia. `sessionmaker` to automat wydający kluczyki do pojedynczych rowerów. `Session` to konkretny rower, który wziąłeś. Nie dzielisz roweru między dwie osoby jadące w różne strony — po to jest fabryka, żeby każdy dostał własny.

`sessionmaker` jest jednym z nielicznych obiektów w SQLAlchemy, które można bezpiecznie współdzielić między wątkami: nie ma stanu, tylko tworzy nowe obiekty. Podobnie `Engine`.

### 6.2. `scoped_session` — do czego powstało

`scoped_session` to opakowanie na fabrykę, które **automatycznie zwraca tę samą sesję w obrębie jednego „zakresu”** (domyślnie: jeden wątek).

```python
# examples/21_scoped.py
from __future__ import annotations

from sqlalchemy.orm import scoped_session, sessionmaker

from app.db import engine

ScopedSessions = scoped_session(sessionmaker(bind=engine))

# W wątku A:
session_a = ScopedSessions()   # tworzy sesję i zapamiętuje ją dla tego wątku
assert ScopedSessions() is session_a   # ✅ ten sam obiekt

# W wątku B (inny zakres):
# session_b = ScopedSessions()  # to byłaby INNA sesja, mimo tego samego opakowania

ScopedSessions.remove()  # zamknij sesję bieżącego zakresu
```

Problem, który `scoped_session` rozwiązuje, jest realny: w aplikacji wielowątkowej (Flask w trybie sync, Django, wątki w bibliotekach) nie chcesz przekazywać sesji przez 15 warstw funkcji — chcesz ją „wziąć z powietrza” w danym wątku. Po to jest zakres.

### 6.3. Kiedy `scoped_session` jest reliktem

Dziś — w większości projektów — `scoped_session` bywa **niepotrzebnym pośrednikiem**, i to z czterech powodów:

1. **Ukrywa granicę.** „Wziąłem sesję z powietrza” znaczy „nie wiem, gdzie ona powstała i gdzie się kończy”. Właśnie tym problemem zajmuje się ten moduł — UoW ma czynić granicę jawną.
2. **Nie wystarcza dla async.** `AsyncSession` nie korzysta ze `scoped_session` w ten sam sposób; cała koncepcja „zakresu wątku” jest obca `asyncio`. W korutynach współbieżność jest **kooperatywna w jednym wątku**, więc thread-local byłby aktywną pułapką: dwie korutyny dzieliłyby jedną sesję.
3. **Wprowadza fałszywe bezpieczeństwo.** „Thread-local” nie znaczy „bezpieczny”, gdy używasz `asyncio.to_thread` czy `ThreadPoolExecutor` — sesja może „uciec” z jednego wątku do drugiego w środku pracy.
4. **DI jest jawniejsze.** Nowoczesne aplikacje (FastAPI, Litestar) mają wbudowane wstrzykiwanie zależności (§7). Wtedy nie potrzebujesz magii wątków.

Tabela decyzyjna:

| Kontekst | Rekomendacja |
|---|---|
| Flask sync, klasyczny wielowątkowy WSGI | `scoped_session` z `remove()` w `teardown_request` — akceptowalne |
| FastAPI / Litestar (sync lub async) | `Depends(get_uow)` — **bez** `scoped_session` |
| Worker wielowątkowy (`ThreadPoolExecutor`) | sesja w każdym wątku jawnie, nie thread-local |
| Skrypt jednorazowy | zwykłe `Session(session_factory)` |
| Biblioteka współdzielona między frameworkami | protokół UoW wstrzykiwany jako argument |

> ⚠️ **Pułapka — „session is bound to a different event loop”**
> W aplikacji async tworzenie `AsyncSession` „na zapas” (np. w `__init__` serwisu) i użycie jej w innym zadaniu niż pętla zdarzeń, w której powstała, kończy się błędem albo cichym zawieszeniem. Zasada: `AsyncSession` powstaje **we wnętrzu** konkretnego wywołania `async with`, w tym samym zadaniu, w którym będzie używana.

> 🆕 **SQLAlchemy 2.1 — nowy domyślny sterownik w adresie**
> W 2.1 `postgresql://` domyślnie rozwiązuje się na `psycopg` (psycopg 3), a nie na `psycopg2`. Jeśli Twoja konfiguracja opierała się na niejawnym założeniu, że `postgresql://` oznacza psycopg2 (np. dedykowane `connect_args`), po aktualizacji zmieni się zachowanie. Zalecenie: **zawsze** pisz dialekt i sterownik jawnie, np. `postgresql+psycopg://`, `postgresql+asyncpg://`, `sqlite+pysqlite://`.

---

## 7. Wstrzykiwanie zależności bez frameworka

### 7.1. Po co wstrzykiwać zamiast importować

Skoro można zrobić `from app.db import session_factory`, po co przekazywać fabrykę? Odpowiedź jest krótka, ale fundamentalna: **testowalność**. Chcesz móc wstawić w testach inną fabrykę (bazę testową) bez `monkeypatch`-owania globalnego modułu. Jawna zależność wywołuje mniej niespodzianek niż ukryty import.

Zasada projektowa: **serwisy nie wiedzą, skąd przyszła fabryka sesji.** Znają tylko protokół (moduł 20).

### 7.2. Protokół UoW

```python
# app/uow_protocol.py
from __future__ import annotations

from typing import Protocol

from app.repositories import (
    BookRepository,
    MemberRepository,
    OrderRepository,
    OutboxRepository,
)


class UnitOfWorkProtocol(Protocol):
    """Interfejs UoW — bez śladu SQLAlchemy.

    Dzięki temu serwis można testować z implementacją in-memory,
    bez prawdziwej bazy.
    """

    books: BookRepository
    members: MemberRepository
    orders: OrderRepository
    outbox: OutboxRepository

    def commit(self) -> None: ...
    def rollback(self) -> None: ...
```

### 7.3. Kontener zależności w 40 linijkach

Nie trzeba wcale frameworka. Prosty kontener to słownik fabryk oparty na leniwej inicjalizacji:

```python
# app/container.py
from __future__ import annotations

from collections.abc import Callable
from typing import Any, TypeVar

T = TypeVar("T")


class Container:
    """Minimalny kontener DI: rejestrujesz fabryki, pobierasz instancje."""

    def __init__(self) -> None:
        self._factories: dict[type[Any], Callable[[], Any]] = {}
        self._singletons: dict[type[Any], Any] = {}

    def register(self, key: type[T], factory: Callable[[], T]) -> None:
        self._factories[key] = factory

    def register_instance(self, key: type[T], instance: T) -> None:
        self._singletons[key] = instance

    def resolve(self, key: type[T]) -> T:
        if key in self._singletons:
            return self._singletons[key]
        if key not in self._factories:
            raise LookupError(f"No factory registered for {key!r}")
        return self._factories[key]()
```

Rejestracja w `main.py`:

```python
# app/main.py (fragment konfiguracji)
from sqlalchemy.orm import Session, sessionmaker

from app.container import Container
from app.db import session_factory

container = Container()
container.register_instance(sessionmaker[Session], session_factory)
# Serwis dostaje UoW tworzony per wywołanie (factory), nie singleton:
# container.register(SomeService, lambda: SomeService(unit_of_work))
```

> 💡 **Analogia — rozdzielnia w budynku**
> Kontener DI to tablica rozdzielcza. W jednej skrzynce podpisane są kable: „prąd do kuchni”, „woda do łazienki”. Nie musisz wiedzieć, skąd dokładnie w mieście płynie prąd — wystarczy, że wiesz, że gniazdko działa. W testach podmieniasz jedno gniazdko („atrapa dostawcy”), nie całe miasto.

### 7.4. Wstrzykiwanie fabryki vs wstrzykiwanie instancji

To rozróżnienie ma konkretne znaczenie przy UoW:

| Wstrzykujesz | Kiedy | Dlaczego |
|---|---|---|
| **instancję UoW** | serwis ma być wywołany w już otwartej transakcji (jak w FastAPI dependency) | granica transakcji jest wyżej, w handlerze |
| **fabrykę UoW** | serwis sam decyduje, ile transakcji wykonuje (np. 10 niezależnych kroków) | granica jest wewnątrz serwisu — trzeba ją umieć uzasadnić |
| **gołą fabrykę sesji** | kod infrastrukturalny, skrypty, migracje | najniższa abstrakcja, najwięcej swobody (i rozgardiaszu) |

Zalecenie praktyczne: **domyślnie instancja UoW.** Wstrzykiwanie fabryki mnoży miejsca, w których powstają transakcje, a każde takie miejsce trzeba osobno przemyśleć pod kątem atomowości.

---

## 8. Granice transakcji: co wolno i jak długo

### 8.1. Reguła 3 sekund (i skąd się bierze)

Nie ma uniwersalnej liczby, ale warto mieć sygnał ostrzegawczy: **transakcja pisząca (z blokadami) powinna trwać od kilku milisekund do kilku sekund**. Wszystko dłużej to sygnał, że w środku jest coś, czego tam być nie powinno.

Typowe „grzechy” wydłużające transakcję:

| W środku transakcji | Skutek | Poprawnie |
|---|---|---|
| `requests.get(...)` do API zewnętrznego | blokada trwa tyle, co sieć zewnętrzna | pobierz dane **przed** transakcją |
| `send_email()` | blokada SMTP, brak atomowości | outbox (§2.3) |
| `time.sleep(5)` | nonsens — ale zdarza się w kodzie „poczekaj na indexing” | nigdy; to źródło deadlocków i timeoutów |
| pętla po 10 000 wierszy z `flush()` w środku | ogromna transakcja, tysiące instrukcji | batching (§5.3) |
| wczytywanie dużego pliku | trzymasz blokadę na tabeli | zapisz plik poza bazą, transakcja tylko na metadane |
| renderowanie PDF-a | CPU w środku transakcji | policz dane, zamknij transakcję, generuj PDF |

> 🧠 **Dlaczego blokada jest kosztowna** — Transakcja pisząca w PostgreSQL trzyma blokady na zmodyfikowanych wierszach oraz (w zależności od operacji) na poziomie tabeli. Dopóki się nie skończy, inni czekają albo dostają błąd. Długie transakcje powodują też efekt „puchnięcia” (bloat) bazy — PostgreSQL nie może odzyskać miejsca po starych wersjach wierszy, dopóki są one widoczne dla jakiejś otwartej transakcji.

### 8.2. Podział na kilka transakcji — kiedy jest OK

Niektóre przypadki użycia z natury są sekwencją niezależnych kroków:

```text
Import 100 000 rekordów z CSV:
├── TRANSAKCJA 1: walidacja i zapis partii 1–1000      → commit
├── TRANSAKCJA 2: walidacja i zapis partii 1001–2000   → commit
├── …
└── TRANSAKCJA N: zapis ostatniej partii               → commit

Każda partia jest spójna sama w sobie; niepowodzenie partii 5
nie unieważnia partii 1–4 (i to jest POŻĄDANE).
```

Odwrotny przypadek — **nie dziel** wbrew naturze operacji:

```text
Złożenie zamówienia:
├── TRANSAKCJA 1: zapis zamówienia        → commit
├── TRANSAKCJA 2: zapis pozycji            → commit   ⛔ zamówienie bez pozycji!
└── TRANSAKCJA 3: zmniejszenie stanu       → commit   ⛔ stan niezgodny z zamówieniem
```

Rozpoznaj różnicę: **import partiami** to sekwencja niezależnych faktów; **zamówienie z pozycjami** to jeden fakt, zapisany w trzech tabelach.

### 8.3. ACID w jednym akapicie (dla porządku)

- **A — atomicity (atomowość):** wszystko albo nic (koperta).
- **C — consistency (spójność):** po transakcji wszystkie ograniczenia schematu i niezmienniki są spełnione (np. `stock >= 0`).
- **I — isolation (izolacja):** transakcje nie widzą swoich nawzajem niedokończonych zmian (szczegóły w module 14: poziomy izolacji, anomalie).
- **D — durability (trwałość):** po `COMMIT` dane przetrwają restart, nawet nagły.

UoW realizuje literę **A** w warstwie aplikacji: jedną granicą obejmuje wszystkie zmiany z całego przypadku użycia.

### 8.4. `expire_on_commit` a UoW

Domyślnie `Session(expire_on_commit=True)`: po `commit()` wszystkie atrybuty obiektów zostają **unieważnione** i przy następnym odczycie SQLAlchemy wykonuje dodatkowe `SELECT`. To jest bezpieczne (masz świeże dane, w tym wartości wyliczone przez bazę), ale ma dwa koszty:

1. niewidoczne zapytania w kodzie, który wygląda jak czysty Python (N+1 z piekła),
2. błędy w warstwie API, gdzie obiekt jest serializowany **po** zamknięciu sesji.

Dlatego w warstwie z UoW często ustawia się `expire_on_commit=False`, ale wtedy trzeba być świadomym:

- odczyt po commicie może zwrócić wartości, których baza nie widzi (np. gdy trigger je zmienił),
- obiekt „wygląda żywy” mimo zamkniętej sesji i próba dotknięcia relacji rzuci `DetachedInstanceError`.

> ⚠️ **Pułapka — cicha zmiana pod `expire_on_commit=False`**
> Wyobraź sobie, że serwis ustawia `book.stock = 0`, potem `uow.commit()`, a następnie w innym miejscu (poza UoW) robi `book.stock += 1`, bo „book ma jeszcze zero”. Ale baza ma inną wartość — ktoś w międzyczasie ją zmienił. Bez `expire` Twój obiekt to nieaktualne lustro. Reguła: **jeśli obiekt przekracza granicę UoW, traktuj go jak DTO tylko do odczytu.** Jeśli ma być zmieniany — zacznij nowy UoW i odczytaj go ponownie (albo użyj `session.refresh(obj)`).

### 8.5. Sesja na request a transakcja na przypadek użycia

Częsty spór: „skoro mam sesję per request, czy to jedna transakcja?” Odpowiedź brzmi: **nie, chyba że tak ustawisz sesję.**

```text
Sesja per request (Session otwarta od początku do końca):
┌──────────────────────────────────────────────────────────────┐
│ with UnitOfWork(...) as uow:                                 │
│   [odczyt danych na DTO dla listy]    ← BEGIN …              │
│   [walidacja, logika biznesowa]       ← trwa ta sama tx      │
│   [commit zmian z jednej operacji]    ← COMMIT               │
│   [dalsze odczyty po commicie]        ← NOWY BEGIN (autobegin!)│
│   [uow.close()]                       ← ROLLBACK ostatniego   │
└──────────────────────────────────────────────────────────────┘
```

Zwróć uwagę na ostatnią linię: po `commit()` sesja automatycznie zaczyna **nową** transakcję przy pierwszym zapytaniu (autobegin z modułu 08). Jeśli nie zostanie zamknięta, zostanie wycofana. Dlatego:

- **Jedna sesja na request jest OK, ale jedna transakcja na request — nie zawsze.** UoW z §3.5 wycofuje niezatwierdzony stan automatycznie (metoda `rollback_if_pending`), więc nic nie przecieka.
- Jeśli chcesz, żeby po `commit()` nie zaczynała się nowa transakcja, nie wykonuj dalszych zapytań po commicie. To kwestia dyscypliny — albo jawnie zamknij UoW (`uow.close()`) i otwórz nowy.

> 🆕 **SQLAlchemy 2.1 — `CreateView`, `CREATE TABLE AS SELECT` i inne**
> 2.1 rozbudowało obsługę konstrukcji DDL rzadziej używanych, np. widoków. Dla UoW to nieistotne, ale warto wiedzieć, jeśli w Twoim projekcie UoW musi wykonywać zapytania czytające z widoków. W 2.0 widoki obsługuje się zwykle przez `Table(..., autoload_with=engine)` albo surowy `text("CREATE VIEW ...")`.

---

## 9. Obsługa błędów i ponowienia

### 9.1. Wyjątek domenowy vs wyjątek infrastruktury

Podział jest fundamentalny dla architektury:

| Typ błędu | Przykład | Kto go łapie | Co się dzieje |
|---|---|---|---|
| **Domenowy** | `InsufficientStock`, `EmailAlreadyRegistered` | serwis / endpoint | rollback, użytkownik dostaje 409/422 z czytelnym komunikatem |
| **Infrastrukturalny** | `IntegrityError` (naruszenie unikalności), `OperationalError` (padł serwer) | warstwa UoW / middleware | rollback, log, 500 albo retry |
| **Programistyczny** | `AttributeError`, `TypeError` | — | to bug; rollback i niech leci wyżej, nie maskuj |

`IntegrityError` to szczególny przypadek: pojawia się dopiero przy `flush()`, więc *żaden kod walidacji w Pythonie nie jest w stanie go uprzedzić* w warunkach współbieżności (klasyczny TOCTOU — *time of check to time of use*). Dlatego każde miejsce, gdzie może polecieć unikat, musi mieć plan:

```python
# examples/21_integrity_translation.py
from sqlalchemy.exc import IntegrityError

from app.errors import EmailAlreadyRegistered


def register_member(uow: UnitOfWork, name: str, email: str) -> Member:
    if uow.members.get_by_email(email) is not None:
        # Szybka ścieżka: sprawdzenie z rozsądnym komunikatem.
        raise EmailAlreadyRegistered(email)

    member = Member(name=name, email=email)
    uow.members.add(member)

    try:
        # Wymusza wysłanie INSERT-a; tu pojawi się ewentualny konflikt.
        uow.session.flush()
    except IntegrityError as exc:
        uow.rollback()
        # Zamieniamy błąd infrastruktury na domenowy.
        raise EmailAlreadyRegistered(email) from exc

    return member
```

> 🧠 **Dlaczego `raise ... from exc`** — zachowujesz oryginalną przyczynę (`__cause__`), więc w logach widzisz prawdziwy komunikat bazy, ale kod wołający widzi wyjątek domenowy. To jedyny sposób, żeby nie tracić informacji diagnostycznej przy jednoczesnym zachowaniu czystej granicy warstw.

### 9.2. Sesja po wyjątku — dlaczego `rollback` jest obowiązkowy

Po wyjątku z `flush()` sesja wchodzi w stan „unieważnionej transakcji”. Wygląda to tak:

```python
# examples/21_rollback_required.py
from sqlalchemy.exc import PendingRollbackError


with UnitOfWork(session_factory) as uow:
    uow.members.add(Member(name="Ala", email="dup@example.com"))
    try:
        uow.session.flush()   # IntegrityError: naruszenie unikalności
    except IntegrityError:
        pass                   # ⛔ POŁKNIĘTE bez rollback!

    # Każde kolejne zapytanie rzuci:
    # PendingRollbackError: This Session's transaction has been rolled back due
    # to a previous exception during flush. To begin a new transaction with this
    # Session, first issue Session.rollback().
    uow.books.list_all()
```

To jest **częsty błąd produkcyjny**: ktoś łapie `IntegrityError`, loguje i próbuje kontynuować, nie zdając sobie sprawy, że sesja jest w trybie „wymaga ratunku”.

Naprawa jest wbudowana w UoW: `rollback()` musi zostać wykonany, zanim sesja wróci do użycia. W wariantach z §3.5 robi to `__exit__`; w kodzie wewnątrz bloku musisz zrobić to sam:

```python
try:
    uow.session.flush()
except IntegrityError:
    uow.rollback()   # ✅ sesja znowu może pracować (rozpocznie nową transakcję)
    raise
```

Jeśli chcesz **chronić część operacji w obrębie jednej dużej transakcji**, użyj savepointu:

```python
# examples/21_savepoint.py
with UnitOfWork(session_factory) as uow:
    for index, row in enumerate(rows, start=1):
        try:
            # savepoint: osobny "mini-rollback" dla tego wiersza
            with uow.session.begin_nested():
                uow.members.add(Member(name=row["name"], email=row["email"]))
                uow.session.flush()
        except IntegrityError:
            # wiersz pominięty, ale resztę importu zachowujemy
            log_skipped(index, row)
    uow.commit()
```

> 🔬 **Pod maską — savepointy w SQL**
> ```sql
> BEGIN;
> SAVEPOINT sa_savepoint_1;
> INSERT INTO members (name, email) VALUES (?, ?);
> ROLLBACK TO SAVEPOINT sa_savepoint_1;   -- tylko ten INSERT cofnięty
> -- kolejne wiersze…
> COMMIT;                                  -- reszta importu trwała
> ```
> To najlepsze narzędzie na importy „odpornych na pojedyncze wiersze”. Uwaga na wydajność: tysiące savepointów to tysiące dodatkowych komend — przy 100 000 wierszy lepiej dzielić na porcje i używać `insert().values([...])` zamiast savepointów na każdy rekord.

### 9.3. Retry przy UoW

Współbieżne zapisy na PostgreSQL mogą kończyć się `SerializationFailureError` albo `DeadlockDetected`. To nie są błędy logiki — to kwestia „losu”. Rozwiązaniem jest ponowienie (retry) całej jednostki pracy od nowa.

```python
# app/retry.py
from __future__ import annotations

import time
from collections.abc import Callable
from typing import TypeVar

from sqlalchemy.exc import DBAPIError, OperationalError

T = TypeVar("T")

RETRYABLE_SQLSTATES = {
    "40001",  # serialization_failure
    "40P01",  # deadlock_detected
}


def _is_retryable(exc: BaseException) -> bool:
    if isinstance(exc, DBAPIError) and exc.orig is not None:
        sqlstate = getattr(exc.orig, "sqlstate", None) or getattr(exc.orig, "pgcode", None)
        if sqlstate in RETRYABLE_SQLSTATES:
            return True
    return isinstance(exc, OperationalError)  # np. chwilowa niedostępność


def with_retry(action: Callable[[], T], *, attempts: int = 3, backoff: float = 0.1) -> T:
    """Powtarza CAŁĄ operację UoW — nigdy pojedyncze zapytanie."""
    last: BaseException | None = None
    for attempt in range(attempts):
        try:
            return action()
        except BaseException as exc:  # noqa: BLE001 — świadomie szeroko
            if not _is_retryable(exc):
                raise
            last = exc
            time.sleep(backoff * (2**attempt))  # wykładniczy backoff
    assert last is not None
    raise last
```

Użycie — **kluczowa zasada**: retry obejmuje **całą funkcję operującą na UoW**, a nie poszczególne kroki. Każde ponowienie zaczyna się od świeżego UoW, bo poprzednie po błędzie jest bezużyteczne (i musi zostać zamknięte).

```python
# examples/21_retry_usage.py
from app.db import session_factory
from app.uow import UnitOfWork

# ✅ cała operacja od nowa
order_id = with_retry(lambda: _create_order_once(session_factory, member_id=1, items=items))


def _create_order_once(
    factory: sessionmaker[Session], member_id: int, items: list[tuple[int, int]]
) -> int:
    with UnitOfWork(factory) as uow:
        order = place_order(uow, member_id=member_id, items=items)
        uow.commit()
        return order.id
```

> ⚠️ **Pułapka — retry bez świeżej sesji**
> Jeśli spróbujesz „ponowić” wywołanie w tej samej sesji, dostaniesz `PendingRollbackError` albo (gorzej) ciche działanie na niezatwierdzonym, częściowo zapisanym stanie. Retry i UoW **muszą być zagnieżdżone w tej kolejności**: pętla retry na zewnątrz, UoW w środku każdej próby.

### 9.4. Idempotencja ponowienia

Ponowienie jest bezpieczne tylko, gdy operacja jest **idempotentna** — wykonana dwa razy daje ten sam skutek, co wykonana raz. W praktyce:

| Operacja | Idempotentna? | Jak ją uczynić idempotentną |
|---|---|---|
| `INSERT` z unikalnym kluczem `idempotency_key` | nie | dodać unikalny klucz + `ON CONFLICT DO NOTHING` |
| Zwiększenie licznika `stock += n` | nie | użyć stanu docelowego, nie przyrostu |
| Zmniejszenie `stock` o zamówienie | nie | powiązać z `order_id` i sprawdzać istnienie zamówienia |
| `INSERT` z naturalnym kluczem (e-mail) | tak, jeżeli jest unikalny | unikalny indeks + obsługa `IntegrityError` |

To wątek z modułu 14, ale ma bezpośredni związek z UoW: **retry + brak idempotencji = duplikaty danych.** Zawsze sprawdzaj oba razem.

---

## 10. Alternatywy: kiedy UoW to przesada

Uczciwa ocena: w wielu projektach pełny wzorzec z klasą, protokołem i DI jest nadmiarem. Trzy zdrowe warianty, uporządkowane od najprostszego.

### 10.1. Wariant 0: `session_scope()` (serwis + sesja)

Nie ma UoW — jest funkcja i sesja.

```python
# examples/21_alt_scope.py
def place_order_simple(member_id: int, items: list[tuple[int, int]]) -> int:
    with session_scope() as session:
        books = BookRepository(session)
        members = MemberRepository(session)
        orders = OrderRepository(session)
        # … logika …
        return new_order.id
```

**Kiedy to wystarcza:** aplikacja jednoosobowa, 5 endpointów, brak planów wzrostu. Zaleta: zero abstrakcji, mało kodu. Wada: serwis zna wszystkie repozytoria i wie, że istnieje `session_scope`.

### 10.2. Wariant 1: Transaction Script

Dla zadań „proceduralnych” (raport miesięczny, migracja danych) można zrezygnować z repozytoriów i pisać wszystko w jednej funkcji operującej na sesji:

```python
# examples/21_transaction_script.py
def add_late_fee_for_overdue_loans() -> int:
    with session_scope() as session:
        stmt = select(Loan).where(
            Loan.returned_at.is_(None),
            Loan.due_at < func.now(),
        )
        updated = 0
        for loan in session.scalars(stmt):
            loan.fee = compute_fee(loan)
            updated += 1
        return updated
```

**Kiedy to wystarcza:** skrypty administracyjne, migracje, jednorazowe narzędzia. Dla logiki biznesowej w serwisie — zwykle prowadzi do funkcji po 300 linii.

### 10.3. Wariant 2: tylko `sessionmaker` + `Depends`

W małej aplikacji FastAPI `Depends(get_session)` z `commit`/rollback w generatorze wystarczy. To prawie UoW, tylko bez klasy.

### 10.4. Tabela decyzyjna

| Kryterium | Wariant 0 (`session_scope`) | Wariant 2 (Depends + sesja) | UoW |
|---|---|---|---|
| Liczba przypadków użycia | < 10 | 10–50 | > 20 albo rosnąca |
| Liczba repozytoriów | 1–2 | 3–6 | dowolna |
| Zespół | 1 osoba | 2–5 osób | > 5 osób, rotacja |
| Wymóg testowania bez bazy | niski | średni | wysoki |
| Potrzebujesz protokołu UoW do fake'ów | nie | nie | tak |
| Chcesz „jedno miejsce, gdzie kończy się transakcja” | nie | częściowo | tak |
| Koszt wprowadzenia | zerowy | mały | średni (klasa + protokół + DI) |

> 🧠 **Reguła dojrzałości** — UoW wchodzi wtedy, gdy **liczba przypisów „pamiętaj o rollbacku” w głowie** staje się dłuższa niż koszt napisania 60 linii klasy. To jest granica praktyczna, nie „dobre praktyki”, bo „dobre praktyki” bez kosztu to religia, nie inżynieria.

---

## 11. Diagram przepływu requestu

Przyjrzyjmy się całej drodze żądania HTTP przez warstwę danych i zaznaczmy, gdzie zaczyna się i kończy transakcja.

```text
┌──────────────────────────────────────────────────────────────────────────┐
│ HTTP POST /orders                                                        │
└──────────────────────────────────────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ [1] Middleware / Dependency                                             │
│     with UnitOfWork(session_factory) as uow:                            │
│       │                                                                 │
│       ├─ utworzenie Session  ───────────────────────────────────►  BEGIN │
│       │                                                                 │
│       ▼                                                                 │
│ [2] Endpoint (warstwa prezentacji)                                      │
│     • walidacja Pydantic  (BEZ bazy — nie trzyma transakcji bez powodu) │
│     │                                                                   │
│     ▼                                                                   │
│ [3] Serwis (przypadek użycia)  ── granica LOGIKI ──                     │
│     place_order(uow, member_id, items):                                 │
│       • uow.members.get(member_id)                → SELECT members      │
│       • uow.books.get_locked(bid)  (FOR UPDATE)   → SELECT … FOR UPDATE │
│       • sprawdzenie niezmienników (stock >= qty)  → (błąd domenowy)     │
│       • uow.books.decrease_stock(bid, qty)        → UPDATE (pending)    │
│       • uow.orders.add_with_items(...)            → INSERT (pending)    │
│       • uow.outbox.emit("OrderPlaced", payload)   → INSERT (pending)    │
│     ── TU zaczyna się „koperta” — wszystkie zmiany w jednej transakcji  │
│     │                                                                   │
│     ▼                                                                   │
│ [4] Decyzja o granicy transakcji                                        │
│     uow.commit()  ──────────────────────────────────────────────► COMMIT│
│     LUB  wyjątek  ──────────────────────────────────────────────► ROLLBACK
│     │                                                                   │
│     ▼                                                                   │
│ [5] Mapowanie na DTO (odczyt pól WEWNĄTRZ sesji)                        │
│     │                                                                   │
│     ▼                                                                   │
│ [6] __exit__ → close() → połączenie wraca do puli  ─────────────► (brak) │
└──────────────────────────────────────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ HTTP 201 { "order_id": 42 }                                             │
└──────────────────────────────────────────────────────────────────────────┘
```

Cztery miejsca na tym diagramie są kluczowe i warto je zapamiętać jako reguły:

1. **BEGIN następuje przed pierwszym zapytaniem, nie przy tworzeniu sesji** — dzięki autobegin. Dlatego „sesja otwarta” ≠ „transakcja otwarta”.
2. **Granica logiki (serwis) nie pokrywa się z granicą transakcji (commit) dobrowolnie** — mogłyby być w tym samym miejscu, ale rozdzielenie ich pozwala np. zwrócić DTO przed commitem albo zebrać dane do odpowiedzi po commicie.
3. **Walidacja Pydantic nie powinna odbywać się w transakcji** — żądanie z błędem typu powinno zostać odrzucone, zanim otworzysz jakiekolwiek połączenie. W FastAPI walidacja działa przed wywołaniem `Depends(get_uow)`? W praktyce zależności są rozwiązywane przed wejściem w handler, więc **walidacja body** dzieje się po jej wywołaniu — ale sama sesja jeszcze nie wykonała żadnego zapytania, więc transakcja nie jest otwarta (§ reguła 1).
4. **`close()` po `commit`** zwalnia połączenie do puli, nawet jeśli nie zamkniesz procesu.

---

## 12. Testowanie `UnitOfWork`

Trzy rzeczy warto przetestować oddzielnie: **klasa UoW**, **serwisy przez UoW** i **integracja z endpointern**.

### 12.1. Test klasy UoW — czy rollback faktycznie działa

```python
# tests/test_uow_lifecycle.py
from __future__ import annotations

import pytest
from sqlalchemy.orm import Session, sessionmaker

from app.models import Book
from app.uow import UnitOfWork


def test_commit_persists(test_session_factory: sessionmaker[Session]) -> None:
    with UnitOfWork(test_session_factory) as uow:
        uow.books.add(Book(title="Refactoring", isbn="978-1", price=50, stock=3))
        uow.commit()

    with UnitOfWork(test_session_factory) as uow:
        assert uow.books.get_by_isbn("978-1") is not None


def test_exit_without_commit_rolls_back(
    test_session_factory: sessionmaker[Session],
) -> None:
    with UnitOfWork(test_session_factory) as uow:
        uow.books.add(Book(title="Refactoring", isbn="978-1", price=50, stock=3))
        # brak uow.commit() — wychodzimy z bloku

    with UnitOfWork(test_session_factory) as uow:
        assert uow.books.get_by_isbn("978-1") is None  # ✅ nic nie zostało


def test_exception_rolls_back(test_session_factory: sessionmaker[Session]) -> None:
    class Boom(Exception):
        pass

    with pytest.raises(Boom):
        with UnitOfWork(test_session_factory) as uow:
            uow.books.add(Book(title="Refactoring", isbn="978-1", price=50, stock=3))
            raise Boom  # wyjątek w środku — musi wycofać transakcję

    with UnitOfWork(test_session_factory) as uow:
        assert uow.books.get_by_isbn("978-1") is None
```

Trzeci test to dosłownie **„test na fałszywej operacji rzucającej wyjątek”** z zakresu modułu. Zauważ, jak mało jest w nim kodu: cała „sztuka” polega na tym, że operacja, która ma się nie udać, nie jest skomplikowana — i nie powinna być. Testujemy mechanizm, nie domenę.

### 12.2. Test serwisu z fake UoW (bez bazy)

Drugi poziom to test logiki biznesowej **bez prawdziwej bazy**. Do tego właśnie definiujemy protokół i fake repozytoria.

```python
# tests/test_services_fake.py
from __future__ import annotations

from decimal import Decimal

import pytest

from app.errors import InsufficientStock, MemberNotFound
from app.models import Book, Member, Order, OutboxEvent
from app.services import place_order
from app.uow_protocol import UnitOfWorkProtocol


class FakeBookRepository:
    def __init__(self, books: dict[int, Book]) -> None:
        self._books = books

    def get(self, pk: int) -> Book | None:
        return self._books.get(pk)

    def decrease_stock(self, book_id: int, quantity: int) -> None:
        book = self._books.get(book_id)
        if book is None:
            raise KeyError(book_id)
        if book.stock < quantity:
            raise InsufficientStock(book_id, quantity, book.stock)
        book.stock -= quantity


class FakeMemberRepository:
    def __init__(self, members: dict[int, Member]) -> None:
        self._members = members

    def get(self, pk: int) -> Member | None:
        return self._members.get(pk)


class FakeOrderRepository:
    def __init__(self) -> None:
        self.created: list[Order] = []

    def add_with_items(self, member: Member, lines: list[tuple[Book, int]]) -> Order:
        order = Order(member=member)
        self.created.append(order)
        return order


class FakeOutboxRepository:
    def __init__(self) -> None:
        self.events: list[OutboxEvent] = []

    def emit(self, event_type: str, payload: str) -> OutboxEvent:
        event = OutboxEvent(event_type=event_type, payload=payload)
        self.events.append(event)
        return event


class FakeUnitOfWork:
    """Implementuje tylko to, czego używa serwis — nie cały SQLAlchemy."""

    def __init__(self, books: dict[int, Book], members: dict[int, Member]) -> None:
        self.books = FakeBookRepository(books)
        self.members = FakeMemberRepository(members)
        self.orders = FakeOrderRepository()
        self.outbox = FakeOutboxRepository()
        self.committed = False

    def commit(self) -> None:
        self.committed = True

    def rollback(self) -> None:
        self.committed = False


def test_place_order_decrements_stock() -> None:
    book = Book(id=1, title="Clean Code", isbn="978-0-13", price=Decimal("100"), stock=5)
    member = Member(id=1, name="Ala", email="ala@example.com")
    uow = FakeUnitOfWork(books={1: book}, members={1: member})

    order = place_order(uow, member_id=1, items=[(1, 2)])  # type: ignore[arg-type]

    assert book.stock == 3
    assert uow.outbox.events[0].event_type == "OrderPlaced"
    assert order in uow.orders.created


def test_place_order_raises_on_missing_member() -> None:
    uow = FakeUnitOfWork(
        books={1: Book(id=1, title="X", isbn="1", price=Decimal("1"), stock=5)},
        members={},
    )

    with pytest.raises(MemberNotFound):
        place_order(uow, member_id=99, items=[(1, 1)])  # type: ignore[arg-type]
```

Zauważ, że `Mapped[...]` w modelach dotyczy **tylko mapowania ORM** — w instancjach w pamięci atrybuty są zwykłymi wartościami Pythona, więc możesz je tworzyć bez bazy. To dobry dowód na to, że model jest jednocześnie zwykłą klasą Pythona.

### 12.3. Test integracyjny na prawdziwej bazie

Ostatni poziom: prawdziwa sesja + prawdziwa transakcja. Sprawdza to, czego fake'i nie sprawdzą — zachowanie `IntegrityError`, blokad, migawek i savepointów.

```python
# tests/test_order_integration.py
from __future__ import annotations

import pytest
from sqlalchemy.orm import Session, sessionmaker

from app.errors import InsufficientStock
from app.models import Book, Member
from app.services import place_order
from app.uow import UnitOfWork


@pytest.fixture
def seeded(test_session_factory: sessionmaker[Session]) -> None:
    with UnitOfWork(test_session_factory) as uow:
        uow.books.add(Book(title="Clean Code", isbn="978-0-13", price=100, stock=2))
        uow.members.add(Member(name="Ala", email="ala@example.com"))
        uow.commit()


def test_order_reduces_stock_atomically(
    test_session_factory: sessionmaker[Session], seeded: None
) -> None:
    with UnitOfWork(test_session_factory) as uow:
        place_order(uow, member_id=1, items=[(1, 2)])
        uow.commit()

    with UnitOfWork(test_session_factory) as uow:
        assert uow.books.get(1).stock == 0
        assert len(uow.orders.list_all()) == 1
        assert len(uow.outbox.list_all()) == 1


def test_insufficient_stock_rolls_everything_back(
    test_session_factory: sessionmaker[Session], seeded: None
) -> None:
    with pytest.raises(InsufficientStock):
        with UnitOfWork(test_session_factory) as uow:
            place_order(uow, member_id=1, items=[(1, 5)])  # tylko 2 na stanie
            uow.commit()

    with UnitOfWork(test_session_factory) as uow:
        assert uow.books.get(1).stock == 2      # ✅ bez zmian
        assert len(uow.orders.list_all()) == 0  # ✅ brak zamówienia
        assert len(uow.outbox.list_all()) == 0  # ✅ brak zdarzenia
```

Drugi test to jest esencja UoW: **jedna operacja kończy się błędem, więc trzy tabele pozostają nienaruszone.** Czytelnik widzi w jednym pliku, że wzorzec rzeczywiście działa — i może to powtórzyć u siebie.

---

## Podsumowanie

1. **`Session` w SQLAlchemy już jest implementacją wzorca Unit of Work.** Nie tworzysz nowego mechanizmu — dodajesz architektoniczną obudowę i reguły.
2. **Transakcja = jeden przypadek użycia.** „Zarejestruj użytkownika i wyślij e-mail” to jeden przypadek użycia, ale **dwie** granice: jedna transakcyjna (zapis), druga asynchroniczna (outbox). Nigdy nie łącz.
3. **Wszystkie repozytoria w jednym UoW dzielą jedną sesję** — jedna transakcja, jedna identity map, jedna kolejność operacji.
4. **Repozytorium nie może commitować ani tworzyć sesji.** To nie kwestia stylu, to warunek atomowości.
5. **`with UnitOfWork(...): ... uow.commit()`** — commit jawny, rollback w `__exit__` przy wyjątku i przy wyjściu bez commita. Klasa z protokołem dla testów, `@contextmanager` dla prostoty.
6. **Jedna sesja na request (web), na zadanie (worker), na test (pytest), na krok (CLI).** Nigdy globalna zmienna modułu.
7. **`scoped_session` ma wąskie zastosowanie** — klasyczne, sync frameworki wielowątkowe. W async i w projekcie z DI jest reliktem.
8. **`IntegrityError` wymaga `rollback()`** — inaczej sesja jest w trybie „PendingRollbackError”. Savepoint (`begin_nested`) ratuje część operacji w obrębie większej transakcji.
9. **Retry musi obejmować całą jednostkę pracy**, być na zewnątrz UoW i działać tylko na operacjach idempotentnych.
10. **Walidacja walidacji nierówna.** Pydantic przed otwarciem transakcji (walidacja danych wejściowych), ograniczenia w bazie jako ostatnia linia obrony (niezmienniki), logika w serwisie jako pierwsza linia.

---

## Ćwiczenia

### Zadanie 1 — UoW dla trzech repozytoriów (poziom: podstawowy)

Napisz klasę `LibraryUnitOfWork` z metodami `commit()`, `rollback()`, `close()` i atrybutami `books`, `members`, `orders`, zainicjalizowanymi w `__enter__`. Dopisz docstring, w którym zadeklarujesz kontrakt: „Kto może commitować?”, „Kto może tworzyć sesję?”, „Czy repozytoria dostają tę samą sesję?”. Napisz też test, który potwierdza, że wyjście z bloku bez commita **nic nie zapisuje**.

### Zadanie 2 — Reguła biznesowa z prawdziwym niezmiennikiem (poziom: średni)

Zaimplementuj serwis `return_loan(uow, loan_id)` dla wypożyczenia. Wymagania:

1. Znajdź wypożyczenie; jeśli nie istnieje albo jest już zwrócone — rzuć wyjątek domenowy.
2. Ustaw `returned_at = teraz`, zwiększ stan magazynowy książki o 1.
3. Jeśli zwrot jest spóźniony (po `due_at`), nalicz opłatę: `late_fee = dni_spóźnienia * dzienna_stawka`.
4. Zapisz zdarzenie w outboxie (`OutboxEvent(event_type="LoanReturned")`).
5. Cały przypadek użycia ma być jedną transakcją — dopilnuj, żeby było to widać w kodzie.

Następnie napisz **dwa testy**: jeden na szczęśliwą ścieżkę, drugi na spóźniony zwrot. Sprawdź `stock` i obecność zdarzenia w outboxie.

### Zadanie 3 — Retry + UoW na warunkach produkcyjnych (poziom: zaawansowany)

Dany jest dekorator:

```python
# examples/21_task3_retry.py
def with_retry(action: Callable[[], T], *, attempts: int = 3, backoff: float = 0.1) -> T:
    """Powtarza CAŁĄ operację, także przy `SerializationFailureError` (SQLSTATE 40001)."""
    ...
```

Zaimplementuj `create_order_with_retry(factory, member_id, items)`, która:

1. Powtarza całą operację UoW od nowa (nie fragmenty!).
2. Po każdej porażce **loguje** numer próby, ale nie tworzy nowego `Session` w tej samej sesji.
3. Po wyczerpaniu prób rzuca ostatni wyjątek bez maskowania.
4. Jest **idempotentna** — dwukrotne wywołanie z tym samym `idempotency_key` musi dać jedno zamówienie. W tym celu dodaj kolumnę `idempotency_key: Mapped[str]` z unikalnym indeksem do `Order` i użyj `insert().on_conflict_do_nothing(index_elements=["idempotency_key"])` (dla PostgreSQL) z odpowiednikiem dla SQLite.

Opisz w komentarzu, dlaczego test idempotencji jest tu niezbędny, a nie „miły dla chętnych”.

---

### Rozwiązania

#### Zadanie 1

```python
# app/library_uow.py
from __future__ import annotations

from types import TracebackType
from typing import Self

from sqlalchemy.orm import Session, sessionmaker

from app.repositories import BookRepository, MemberRepository, OrderRepository


class LibraryUnitOfWork:
    """Granica transakcji dla wypożyczalni.

    KONTRAKT:
      - Kto może commitować?        Tylko kod poza klasą, jawnie: `uow.commit()`.
      - Kto może tworzyć sesję?     Tylko ta klasa (`__enter__`).
      - Czy repozytoria dzielą sesję? TAK — wszystkie dostają tę samą instancję.
    """

    def __init__(self, session_factory: sessionmaker[Session]) -> None:
        self._session_factory = session_factory
        self._session: Session | None = None

    def __enter__(self) -> Self:
        self._session = self._session_factory()
        self.books = BookRepository(self._session)
        self.members = MemberRepository(self._session)
        self.orders = OrderRepository(self._session)
        return self

    def __exit__(
        self,
        exc_type: type[BaseException] | None,
        exc: BaseException | None,
        tb: TracebackType | None,
    ) -> bool:
        try:
            if exc_type is not None:
                self.rollback()
            elif self._session is not None and self._session.in_transaction():
                self.rollback()   # wyjście bez commit() = wycofanie
        finally:
            self.close()
        return False

    def commit(self) -> None:
        assert self._session is not None
        self._session.commit()

    def rollback(self) -> None:
        if self._session is not None:
            self._session.rollback()

    def close(self) -> None:
        if self._session is not None:
            self._session.close()
            self._session = None
```

Test (fragment — reszta jak w §12.1):

```python
# tests/test_library_uow.py
def test_exit_without_commit_saves_nothing(
    test_session_factory: sessionmaker[Session],
) -> None:
    from app.library_uow import LibraryUnitOfWork
    from app.models import Book

    with LibraryUnitOfWork(test_session_factory) as uow:
        uow.books.add(Book(title="X", isbn="1", price=1, stock=1))
        # brak uow.commit()

    with LibraryUnitOfWork(test_session_factory) as uow:
        assert uow.books.get_by_isbn("1") is None
```

**Na co uważać w rozwiązaniu:** asercja `assert self._session is not None` jest tu użyteczna diagnostycznie, ale nie zastępuje namysłu — `commit()` wywołany poza blokiem `with` to błąd programisty, nie wyjątek domenowy. Alternatywa: rzucić `RuntimeError` (jak w §3.5) zamiast `assert`, żeby nie zniknęło, gdy ktoś uruchomi Pythona z `-O`.

#### Zadanie 2

```python
# app/services_loan.py
from __future__ import annotations

import json
from datetime import UTC, datetime
from decimal import Decimal

from app.errors import DomainError
from app.uow import UnitOfWork

DAILY_LATE_FEE = Decimal("0.50")


class LoanNotFound(DomainError):
    def __init__(self, loan_id: int) -> None:
        super().__init__(f"Loan {loan_id} not found")
        self.loan_id = loan_id


class LoanAlreadyReturned(DomainError):
    def __init__(self, loan_id: int) -> None:
        super().__init__(f"Loan {loan_id} already returned")
        self.loan_id = loan_id


def return_loan(uow: UnitOfWork, loan_id: int) -> None:
    """Jedna transakcja: oznacza zwrot, zwraca stan, nalicza opłatę, emituje outbox.

    UWAGA: sama nie commituje — robi to wołający (endpoint / test), tak jak w §3.6.
    """
    loan = uow.loans.get(loan_id)
    if loan is None:
        raise LoanNotFound(loan_id)
    if loan.returned_at is not None:
        raise LoanAlreadyReturned(loan_id)

    now = datetime.now(UTC)
    loan.returned_at = now

    # Zwrot egzemplarza do magazynu — z blokadą wiersza.
    book = uow.books.get_locked(loan.book_id)
    if book is None:
        raise DomainError(f"Book {loan.book_id} missing for loan {loan_id}")
    book.stock += 1

    late_days = max(0, (now - loan.due_at).days) if now > loan.due_at else 0
    fee = DAILY_LATE_FEE * late_days

    uow.outbox.emit(
        "LoanReturned",
        json.dumps(
            {"loan_id": loan_id, "member_id": loan.member_id, "late_fee": str(fee)}
        ),
    )
    # Wpis o opłacie w osobnej tabeli — pominięty dla zwięzłości.
```

Testy:

```python
# tests/test_return_loan.py
from __future__ import annotations

from datetime import UTC, datetime, timedelta
from decimal import Decimal

import pytest
from sqlalchemy.orm import Session, sessionmaker

from app.errors import DomainError
from app.models import Book, Loan, Member
from app.services_loan import return_loan
from app.uow import UnitOfWork


def _seed(factory: sessionmaker[Session], *, due_in_days: int) -> None:
    now = datetime.now(UTC)
    with UnitOfWork(factory) as uow:
        book = Book(title="Dune", isbn="978-1", price=50, stock=0)
        member = Member(name="Ala", email="ala@example.com")
        uow.books.add(book)
        uow.members.add(member)
        uow.session.flush()
        uow.session.add(
            Loan(
                book_id=book.id,
                member_id=member.id,
                borrowed_at=now - timedelta(days=30),
                due_at=now + timedelta(days=due_in_days),
            )
        )
        uow.commit()


def test_return_on_time_restores_stock(
    test_session_factory: sessionmaker[Session],
) -> None:
    _seed(test_session_factory, due_in_days=3)

    with UnitOfWork(test_session_factory) as uow:
        return_loan(uow, loan_id=1)
        uow.commit()

    with UnitOfWork(test_session_factory) as uow:
        assert uow.books.get(1).stock == 1
        assert uow.loans.get(1).returned_at is not None
        events = uow.outbox.list_all()
        assert events[0].event_type == "LoanReturned"


def test_return_late_computes_fee(
    test_session_factory: sessionmaker[Session],
) -> None:
    _seed(test_session_factory, due_in_days=-5)  # termin minął 5 dni temu

    with UnitOfWork(test_session_factory) as uow:
        return_loan(uow, loan_id=1)
        uow.commit()

    with UnitOfWork(test_session_factory) as uow:
        assert "5" in uow.outbox.list_all()[0].payload  # uproszczona asercja
```

**Na co uważać:** w teście nie używamy `expire_on_commit=True` (domyślnego), więc po `commit` obiekty zostają „świeże”. W drugim teście asercja na `payload` jest umyślnie uproszczona — w prawdziwym projekcie parsujesz JSON i porównujesz na `Decimal`. **Nie zapomnij o `return_Loan` ustawieniu `late_fee` w bazie** — jeśli jest osobna tabela opłat, dopisz wpis do outboxa i tabeli.

#### Zadanie 3

```python
# app/orders_with_idempotency.py
from __future__ import annotations

import logging
from collections.abc import Callable
from typing import TypeVar

from sqlalchemy import select
from sqlalchemy.exc import IntegrityError
from sqlalchemy.orm import Session, sessionmaker

from app.models import Order
from app.uow import UnitOfWork

logger = logging.getLogger(__name__)
T = TypeVar("T")


def create_order_with_retry(
    factory: sessionmaker[Session],
    member_id: int,
    items: list[tuple[int, int]],
    idempotency_key: str,
    *,
    attempts: int = 3,
) -> int:
    """Tworzy zamówienie z ponowieniem i idempotencją.

    Klucz: CAŁA operacja jest ponawiana, zawsze w świeżym UoW.
    """

    def action() -> int:
        with UnitOfWork(factory) as uow:
            # Krok 1: krótka ścieżka idempotencji — jeśli zamówienie z tym
            # kluczem już istnieje, zwracamy je i nie tworzymy nic nowego.
            existing = uow.session.scalars(
                select(Order).where(Order.idempotency_key == idempotency_key)
            ).one_or_none()
            if existing is not None:
                return existing.id

            # Krok 2: normalne złożenie zamówienia (§2.2).
            order = place_order(uow, member_id=member_id, items=items)
            order.idempotency_key = idempotency_key

            try:
                uow.session.flush()
            except IntegrityError:
                uow.rollback()
                # Ktoś inny zapisał zamówienie z tym samym kluczem w międzyczasie —
                # traktujemy to jako sukces idempotencji.
                raise _IdempotencyRetry from None

            uow.commit()
            return order.id

    for attempt in range(1, attempts + 1):
        try:
            return action()
        except _IdempotencyRetry:
            # Ponawiamy — teraz „existing” już istnieje, więc zwrócimy jego id.
            logger.info("Idempotency race detected, retrying (attempt %d)", attempt)
            continue
        except Exception as exc:
            if not _is_retryable(exc) or attempt == attempts:
                raise
            logger.warning("Retryable error, attempt %d/%d: %s", attempt, attempts, exc)

    raise RuntimeError("Unreachable")


class _IdempotencyRetry(Exception):
    """Wewnętrzny sygnał: spróbuj ponownie, bo równoległy zapis wygrał."""
```

**Dlaczego idempotencja jest niezbędna, nie opcjonalna:** każda ponowiona operacja w transakcji z zapisem ma dwa źródła duplikatów — ponowienie po błędzie aplikacji (retry) oraz ponowienie po błędzie sieci (klient wysłał request dwa razy, bo nie dostał odpowiedzi). Bez klucza idempotencji dwukrotne kliknięcie „Zamów” wygeneruje dwa zamówienia. Klucz jest kontraktem z klientem API (nagłówek `Idempotency-Key` w stylu Stripe’a) i zabezpieczeniem na poziomie bazy (unikalny indeks).

> 🆕 **SQLAlchemy 2.1 — lepsze typowanie `Result` i `Row`**
> 2.1 poprawia mapowanie typów dla `Result`/`Row` (PEP 646), więc `uow.session.scalars(...).one_or_none()` lepiej typuje zwracaną wartość w `mypy`. W 2.0 czasem trzeba pomóc adnotacją; w 2.1 robi się to rzadziej. To są dokładnie tego typu „ciche usprawnienia”, które nie zmieniają API, ale ułatwiają pracę w warstwie serwisowej.

---

## Najczęstsze błędy i jak je czytać

| Komunikat / objaw | Przyczyna | Naprawa |
|---|---|---|
| `PendingRollbackError: This Session's transaction has been rolled back due to a previous exception during flush.` | Połknięty `IntegrityError` bez `rollback()`. | Zawsze `uow.rollback()` w `except` albo użyj `begin_nested()` (savepoint) wokół ryzykownej operacji. |
| `sqlalchemy.exc.InvalidRequestError: Can't reconnect until invalid transaction is rolled back` | To samo co powyżej — sesja „zatruta” nieudanym `flush`. | `rollback()`, potem ponowne otwarcie UoW. |
| `DetachedInstanceError: Instance <Book> is not bound to a Session` | Obiekt przekroczył granicę UoW; dotknięto atrybutu po `close()`. | Buduj DTO wewnątrz UoW; `selectinload()` dla relacji; `expire_on_commit=False` + `session.refresh`. |
| `RuntimeError: Session is already flushing` | Rekurencyjne `flush` (np. event w `before_flush` wykonujący zapytanie). | Nigdy nie wołaj zapytań w `before_flush`; przenieś logikę do `before_commit` albo do serwisu. |
| `RuntimeError: UnitOfWork nie jest otwarty — użyj 'with'` (nasz błąd) | `uow.session` użyty poza blokiem `with`. | Nie przekazuj UoW poza blok; buduj DTO i zwracaj je, nie UoW. |
| Dwa zapisy: „udany pierwszy, brak drugiego” | Repozytoria mają różne sesje (jedno tworzy w `__init__`). | Jedna sesja tworzona w UoW, przekazywana do repozytoriów. |
| Użytkownik dostaje e-mail, ale konta nie ma (lub odwrotnie) | Wywołanie usługi zewnętrznej w transakcji. | Wzorzec outbox (§2.3). |
| `sqlalchemy.exc.TimeoutError: QueuePool limit of size 5 overflow 10 reached` | Wyciek połączeń (brak `close()`), długie transakcje. | `close()` w `finally`; sprawdź, czy `__exit__` zawsze zamyka; skróć transakcje. |
| `PostgreSQL: deadlock detected` | Odwrotna kolejność blokad w dwóch transakcjach. | Stała kolejność blokowania (np. zawsze `ORDER BY id`), krótkie transakcje, retry (§9.3). |
| `StatementError: (builtins.AttributeError) 'NoneType' object has no attribute …` w `before_flush` | Event operuje na obiektach `None` (np. usuniętych). | Filtruj `session.new`/`session.dirty` starannie; obsłuż `None`. |
| Test przechodzi lokalnie, na CI „database is locked” (SQLite) | SQLite nie radzi sobie z równoległym pisaniem z wielu połączeń. | Użyj `WAL` mode (`PRAGMA journal_mode=WAL`) albo PostgreSQL w CI; nie testuj współbieżności na SQLite. |
| `AttributeError: 'Session' object has no attribute 'query'` (w starych poradnikach) | Użyto API 1.x (`session.query()`). | To relikt. Użyj `select()` + `session.execute()` / `session.scalars()`. |

---

## Słowniczek modułu

| Termin (EN) | Polski odpowiednik | Wyjaśnienie |
|---|---|---|
| **Unit of Work** | jednostka pracy | Wzorzec (i klasa `Session`), który grupuje zmiany z jednego zadania i wysyła je jako jedną atomową całość. |
| **Transaction** | transakcja | Zakres operacji zatwierdzanych razem (`BEGIN`…`COMMIT`) albo wycofywanych razem (`ROLLBACK`). |
| **Atomicity** | atomowość | Wszystko albo nic. |
| **Boundary** | granica | Miejsce w kodzie, gdzie zaczyna się i kończy transakcja. |
| **Session lifecycle** | cykl życia sesji | Od `session_factory()` do `close()`. |
| **Session pool** | pula połączeń | Zbiór otwartych połączeń, z których sesje biorą jedno na czas transakcji. |
| **Autobegin** | automatyczne rozpoczęcie | SQLAlchemy sam rozpoczyna transakcję przy pierwszym zapytaniu w sesji. |
| **Autoflush** | automatyczne wypchnięcie | `flush()` wywoływany automatycznie przed zapytaniami, żeby widziały oczekujące zmiany. |
| **Autocommit** | automatyczne zatwierdzenie | W 1.x tryb, w którym każde polecenie kończyło transakcję; usunięty w 2.0. |
| **`begin_nested()`** | zagnieżdżona transakcja | Savepoint w PostgreSQL i innych bazach; punkt, do którego można się wycofać bez cofania całości. |
| **Savepoint** | punkt zapisu | Oznaczenie w transakcji umożliwiające częściowe cofnięcie. |
| **`sessionmaker`** | fabryka sesji | Obiekt, który tworzy nowe `Session`. |
| **`scoped_session`** | sesja z zakresem | Opakowanie zwracające tę samą sesję w obrębie wątku; użyteczne rzadziej niż dawniej. |
| **Identity map** | mapa tożsamości | Rejestr w sesji gwarantujący, że jeden wiersz = jeden obiekt Pythona. |
| **Detached instance** | obiekt odłączony | Obiekt poza sesją, którego leniwych relacji nie można dociągnąć. |
| **Outbox pattern** | wzorzec skrzynki nadawczej | Intencję efektu ubocznego zapisujesz w tej samej transakcji, a wykonujesz poza nią. |
| **Idempotency** | idempotencja | Właściwość: wykonanie dwa razy = wykonanie raz. |
| **Retry** | ponowienie | Mechanizm powtórzenia operacji po błędzie możliwym do ponowienia. |
| **SQLSTATE** | kod SQL stanu | Standardowy kod błędu bazy; `40001` = serialization failure, `40P01` = deadlock. |
| **IntegrityError** | błąd naruszenia więzów | Błąd bazy: naruszenie unikalności/klucza obcego/CHECK. |
| **API dependency** | zależność API | Wstrzykiwany obiekt (np. UoW), tworzony przed handlerem i zamykany po odpowiedzi. |
| **DI** (Dependency Injection) | wstrzykiwanie zależności | Przekazywanie zależności z zewnątrz zamiast tworzenia ich w środku. |

---

## Dalsze czytanie

- Oficjalna dokumentacja: „Transactions and Connection Management” — sekcja ORM w podręczniku `Session`:
  https://docs.sqlalchemy.org/en/20/orm/session_transaction.html
- Podstawy `Session`: „Session Basics”:
  https://docs.sqlalchemy.org/en/20/orm/session_basics.html
- Cykl życia i zarządzanie sesją na końcu application — „Session lifecycle patterns”:
  https://docs.sqlalchemy.org/en/20/orm/session_basics.html#session-frequently-asked-questions
- Wybór `scoped_session` i jego alternatywy:
  https://docs.sqlalchemy.org/en/20/orm/contextual.html
- Zagnieżdżone transakcje i savepointy:
  https://docs.sqlalchemy.org/en/20/orm/session_transaction.html#using-savepoint
- Wersja 2.1 — co nowego (dla osób planujących aktualizację):
  https://docs.sqlalchemy.org/en/21/changelog/migration_21.html
- Wzorzec outbox i niezawodne komunikaty w mikroserwisach (kontekst):
  https://microservices.io/patterns/data/transactional-outbox.html

---

## Co dalej

Masz teraz mechanizm, który gwarantuje spójność zmian w obrębie jednego przypadku użycia, oraz reguły, gdzie w aplikacji kończy się transakcja. W module 22 wykorzystamy to wszystko w praktyce: zbudujemy pełny projekt REST na FastAPI — z `Depends(get_uow)`, migracjami, obsługą błędów domenowych i mapowaniem encji na DTO — i zobaczymy, jak UoW wygląda w prawdziwym składzie, a nie w izolowanym pliku `uow.py`.

➡️ Następny moduł: [Moduł 22 — SQLAlchemy w aplikacji webowej (FastAPI)](22_fastapi_integracja.md)

<!-- koniec modułu 21 -->