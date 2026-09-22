# Moduł 24 — Antyworce, debugowanie i FAQ

Ten moduł to skrzynka z narzędziami ratunkowymi. Zamiast uczyć nowych mechanizmów, uczy **rozpoznawania objawów**: gdy w konsoli pojawia się `DetachedInstanceError`, `PendingRollbackError` albo „strona ładuje się trzy sekundy”, powinieneś po samym komunikacie wiedzieć, który mechanizm SQLAlchemy się odezwał i jak go naprawić. Każdy problem opisujemy według jednego schematu: **objaw → przyczyna → naprawa → zapobieganie**. Na końcu znajdziesz gotowe repozytorium sześciu zepsutych skryptów do samodzielnej naprawy oraz checklistę przeglądu kodu, której użyjesz w projekcie końcowym i w pracy zawodowej.

---

**Poziom:** 🟠 średni / 🔴 zaawansowany
**Czas:** ~150 minut (plus czas na ćwiczenia — licz się z 60+ minutami eksperymentów)
**Wymagania wstępne:** wszystkie moduły 01–23; szczególnie `08_sesja_cykl_zycia.md` (stany obiektu, flush vs commit), `11_ladowanie_i_n_plus_1.md` (N+1, strategie ładowania), `14_transakcje_i_wspolbieznosc.md` (stany sesji po błędzie), `15_asynchronicznosc.md` (greenlet), `16_alembic_migracje.md` (Alembic), `18_testowanie.md` (fixtures bazodanowe), `21_unit_of_work.md` (granice transakcji)
**Czego dotyczy plik:** katalogu objawów i antywzorców SQLAlchemy 2.0+, technik diagnostycznych oraz FAQ zbierającego pytania, które wracają najczęściej.

---

## Spis treści

1. [Jak korzystać z tego modułu](#jak-korzystać-z-tego-modułu)
2. [1. DetachedInstanceError — obiekt bez domu](#1-detachedinstanceerror--obiekt-bez-domu)
3. [2. MissingGreenlet — gdy async spotyka leniwe IO](#2-missinggreenlet--gdy-async-spotyka-leniwe-io)
4. [3. „Nic się nie zapisało”](#3-nic-się-nie-zapisało)
5. [4. Duplikaty wyników](#4-duplikaty-wyników)
6. [5. N+1, czyli „strona ładuje się 3 sekundy”](#5-n1-czyli-strona-ładuje-się-3-sekundy)
7. [6. Błędy transakcyjne](#6-błędy-transakcyjne)
8. [7. IntegrityError — jak czytać komunikat bazy](#7-integrityerror--jak-czytać-komunikat-bazy)
9. [8. Problemy migracyjne](#8-problemy-migracyjne)
10. [9. Problemy z typami danych](#9-problemy-z-typami-danych)
11. [10. Problemy z połączeniami](#10-problemy-z-połączeniami)
12. [11. Problemy wydajnościowe](#11-problemy-wydajnościowe)
13. [12. Problemy z testami](#12-problemy-z-testami)
14. [13. Problemy z API i architekturą](#13-problemy-z-api-i-architekturą)
15. [14. Antywzorce projektowe](#14-antywzorce-projektowe)
16. [15. FAQ — dwadzieścia pytań](#15-faq--dwadzieścia-pytań)
17. [16. Checklist przeglądu kodu (30 punktów)](#16-checklist-przeglądu-kodu-30-punktów)
18. [17. Narzędzia diagnostyczne](#17-narzędzia-diagnostyczne)
19. [Jak NIE diagnozować](#jak-nie-diagnozować)
20. [Repozytorium bugów — sześć skryptów do naprawy](#repozytorium-bugów--sześć-skryptów-do-naprawy)
21. [Podsumowanie](#podsumowanie)
22. [Ćwiczenia](#ćwiczenia)
23. [Najczęstsze błędy i jak je czytać](#najczęstsze-błędy-i-jak-je-czytać)
24. [Słowniczek modułu](#słowniczek-modułu)
25. [Dalsze czytanie](#dalsze-czytanie)
26. [Co dalej](#co-dalej)

---

## Jak korzystać z tego modułu

Ten moduł czyta się inaczej niż pozostałe. Nie ma tu jednego przykładu „od początku do końca”, który trzeba przepisać — są **katalogi objawów**.

Przy każdym problemie znajdziesz cztery rzeczy:

- **Objaw** — dokładny komunikat albo obserwowane zachowanie. Wklej ten fragment w wyszukiwarkę albo w `Ctrl+F` w tym pliku.
- **Przyczyna** — mechanizm SQLAlchemy, który za tym stoi. Bez tego kroku naprawa byłaby kopiowaniem recepty od kolegi.
- **Naprawa** — kod, który realnie rozwiązuje problem.
- **Zapobieganie** — reguła, którą wpisujesz do checklisty, żeby problem nie wrócił za dwa tygodnie.

> 💡 **Analogia — ten moduł to SOR, nie szkoła medyczna**
> W poprzednich modułach uczyłeś się anatomii. Tutaj wchodzisz na oddział ratunkowy: na drzwiach wisi tablica „objaw → rozpoznanie → postępowanie”. Lekarz, który zna anatomię, korzysta z tablicy szybciej niż ten, który jej nie zna — ale tablica sama nie wystarczy. Dlatego każdy punkt poniżej odsyła do modułu, w którym dany mechanizm jest wyjaśniony od zera.

Zasada praktyczna: gdy napotkasz problem, **najpierw znajdź swój objaw w tym module**, a dopiero potem zmieniaj kod. Diagnoza przed leczeniem.

---

## 1. DetachedInstanceError — obiekt bez domu

### Objaw

```text
sqlalchemy.orm.exc.DetachedInstanceError: Instance <Book at 0x7f2a1c0d5e50> is not bound
to a Session; attribute refresh operation cannot proceed (Background on this error at:
https://sqlalche.me/e/20/bhk3)
```

Albo wariant „miękki”: nie ma wyjątku, ale `book.author` zwraca `None`, choć w bazie autor istnieje.

### Przyczyna

Obiekt ORM (encja) nie jest samodzielnym workiem danych. Jest **kartką przypiętą do tablicy korkowej**, którą jest sesja. Dopóki sesja żyje, obiekt może dopytać bazę o brakujące kolumny i relacje. Gdy sesja zostanie zamknięta (albo obiekt zostanie z niej usunięty przez `expunge`), obiekt staje się **odłączony (detached)** — wciąż ma to, co już zostało wczytane, ale nie może niczego dopytać.

Trzy słowa, które trzeba rozróżniać:

- **attached (przypięty)** — obiekt jest w sesji, można dopytywać bazę.
- **detached (odłączony)** — sesja już nie istnieje lub obiekt został z niej usunięty; dostęp do wczytanych pól działa, dostęp do niewczytanych wybucha.
- **expired (przedawniony)** — obiekt jest w sesji, ale jego dane uznano za nieaktualne (np. po `commit`); każde dotknięcie pola wywoła nowe zapytanie „odświeżające”.

```python
# examples/24_detached.py
from sqlalchemy import ForeignKey, String, create_engine, select
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship, Session


class Base(DeclarativeBase):
    pass


class Author(Base):
    __tablename__ = "author"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    books: Mapped[list["Book"]] = relationship(back_populates="author")


class Book(Base):
    __tablename__ = "book"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    author_id: Mapped[int] = mapped_column(ForeignKey("author.id"))
    author: Mapped["Author"] = relationship(back_populates="books")


engine = create_engine("sqlite:///:memory:")
Base.metadata.create_all(engine)

with Session(engine) as session:
    session.add(Author(name="Lem", books=[Book(title="Solaris")]))
    session.commit()

# --- scenariusz A: sesja zamknięta, wczytana tylko kolumna ---
with Session(engine) as session:
    book = session.scalars(select(Book)).one()

print(book.title)          # OK — kolumna była wczytana razem z wierszem
print(book.author.name)    # DetachedInstanceError — relacji nie ma, sesji nie ma
```

> 🔬 **Pod maską** — co się dzieje w scenariuszu A
> ```sql
> -- zapytanie z select(Book): ładowana jest tylko tabela book
> SELECT book.id, book.title, book.author_id FROM book;
> -- dostęp do book.author chciałby wyemitować:
> SELECT author.id, author.name FROM author WHERE author.id = ?;
> -- ...ale nie ma już sesji, więc SQLAlchemy odmawia i rzuca wyjątek
> ```

### Pięć scenariuszy i pięć napraw

**Scenariusz 1 — dostęp po zamknięciu bloku `with Session(...)`.**

```python
# ❌ ŹLE
with Session(engine) as session:
    book = session.scalars(select(Book)).one()
print(book.author.name)  # DetachedInstanceError

# ✅ DOBRZE: albo wczytaj relację z góry (eager loading),
stmt = select(Book).options(selectinload(Book.author))
with Session(engine) as session:
    book = session.scalars(stmt).one()
print(book.author.name)  # działa: relacja już jest w pamięci
```

**Scenariusz 2 — `expire_on_commit=True` (domyślne) plus obiekt wyciągnięty z sesji.**

```python
# ❌ ŹLE — commit przedawnia obiekty, więc każde pole trzeba dopytać
with Session(engine) as session:
    book = session.scalars(select(Book)).one()
    session.commit()          # book staje się expired
    session.expunge(book)     # ...i odłączony
print(book.title)             # DetachedInstanceError

# ✅ DOBRZE: albo odczytaj dane PRZED commitem...
with Session(engine) as session:
    book = session.scalars(select(Book)).one()
    snapshot = {"id": book.id, "title": book.title}   # zwykły dict, bez sesji
    session.commit()
# ...albo świadomie wyłącz przedawnianie tam, gdzie to bezpieczne
with Session(engine, expire_on_commit=False) as session:
    book = session.scalars(select(Book)).one()
    session.commit()
print(book.title)  # działa, bo obiekt nie został przedawniony
```

**Scenariusz 3 — sesja zamknięta, potem praca z obiektem (np. w funkcji `render()`).**

```python
# ✅ DOBRZE: konwertuj na DTO wewnątrz sesji — jeden, jawny kontrakt danych
from dataclasses import dataclass


@dataclass(frozen=True)
class BookDTO:
    id: int
    title: str
    author_name: str


def fetch_book(session: Session, book_id: int) -> BookDTO:
    stmt = select(Book).options(joinedload(Book.author)).where(Book.id == book_id)
    book = session.scalars(stmt).one()
    return BookDTO(id=book.id, title=book.title, author_name=book.author.name)
```

**Scenariusz 4 — serializacja w API.** Encja wypuszczona z sesji do warstwy HTTP (albo do Pydantic z `from_attributes=True`) próbuje dociągnąć relacje i wybucha — albo, gorzej, generuje N+1 zapytań, bo sesja wciąż żyje. Naprawa jest zawsze ta sama: **wewnątrz sesji zamień encję na strukturę danych bez „leniwych” haczyków** (DTO, `dict`, model Pydantic) i tylko ją przekazuj dalej. Szczegóły w `22_fastapi_integracja.md`.

**Scenariusz 5 — obiekt przekazany do wątku lub do zadania asynchronicznego.**

```python
# ❌ ŹLE — sesja nie jest bezpieczna wątkowo; obiekt z wątku A używany w wątku B
book = session.scalars(select(Book)).one()          # sesja w wątku głównym
with ThreadPoolExecutor() as pool:
    pool.submit(render_book, book)                  # wątek potomny
# Zamiast tego: przekaż identyfikator albo DTO.

# ✅ DOBRZE — przekazujemy tylko niezmienne dane
book_id, book_title = book.id, book.title
with ThreadPoolExecutor() as pool:
    pool.submit(render_by_id, engine, book_id)
```

> ⚠️ **Pułapka — `merge()` jako plaster**
> `session.merge(obj)` „przypina” odłączony obiekt do nowej sesji i zwraca jego kopię. To bywa przydatne (np. gdy dostajesz encję z formularza), ale jest kosztowne: `merge` wykonuje `SELECT` po kluczu głównym, potem przenosi stan atrybutów, a dla kolumn oznaczonych jako `server_default` może wymagać dodatkowego odświeżenia. Używaj go świadomie, nie jako automatycznej naprawy każdego `DetachedInstanceError`.

### Zapobieganie

1. **Encja nigdy nie opuszcza sesji.** Granica sesji = granica warstwy danych. Na zewnątrz wychodzą DTO, `dict` albo modele Pydantic.
2. **Wszystko, co będzie potrzebne poza sesją, ładuj z góry** (`joinedload` / `selectinload`).
3. W testach ustaw `lazy="raise"` na relacjach (patrz `11_ladowanie_i_n_plus_1.md`) — wtedy brak eager loadingu wychodzi natychmiast, a nie dopiero na produkcji.
4. `expire_on_commit=False` stosuj tam, gdzie wiesz, co robisz (async, odczyt-heavy API), i dokumentuj w kodzie, dlaczego.

> 🧪 **Ćwiczenie** — dopisz do skryptu `examples/24_detached.py` trzeci wariant odczytu, w którym konwertujesz obiekt na `dict` wewnątrz sesji i drukujesz go po jej zamknięciu. Sprawdź, że działa bez żadnego eager loadingu.

---

## 2. MissingGreenlet — gdy async spotyka leniwe IO

### Objaw

```text
sqlalchemy.exc.MissingGreenlet: greenlet_spawn has not been called; can't call await_only()
here. Was IO attempted in an unexpected place? (Background on this error at:
https://sqlalche.me/e/20/xd2v)
```

Często towarzyszy mu `RuntimeWarning: coroutine '...' was never awaited` albo komunikat o „attempting to refresh with lazy loading”. W praktyce objaw jest taki: **kod async wygląda poprawnie, ale wybucha przy pierwszym dostępie do relacji albo po `commit()`**.

### Przyczyna

W trybie asynchronicznym SQLAlchemy działa na „mostach” zwanych **zielonymi wątkami (greenlets)**. Biblioteka udaje synchroniczne API, a naprawdę wykonuje `await` w tle. Most działa tylko wtedy, gdy wywołanie pochodzi z **wnętrza Twojego `await`** — czyli z kodu, który SQLAlchemy objął swoim kontekstem.

Gdy coś dzieje się *poza* tym kontekstem, most nie może wykonać `await` i kończy się `MissingGreenlet`. Trzy klasyczne sytuacje:

1. **Leniwe ładowanie relacji** — `book.author` próbuje wysłać zapytanie, ale nie ma skąd wziąć `await`.
2. **Dostęp do przedawnionego obiektu po `commit()`** — `expire_on_commit=True` (domyślnie) sprawia, że kolejne dotknięcie pola to zapytanie odświeżające, a ono jest leniwe.
3. **`__repr__` albo logowanie**, które dotyka relacji — wyjątek wybucha w najbardziej zaskakującym miejscu, np. w debuggerze lub w loggerze.

> 💡 **Analogia — zielony wątek to tłumacz z ograniczonym dostępem**
> Zielony wątek (greenlet) to tłumacz, który potrafi załatwić sprawę tylko wtedy, gdy stoi obok Ciebie w urzędzie. Gdy jesteś w kolejce (w kontekście `await`), tłumacz działa. Gdy wyszedłeś z budynku (koniec kontekstu `await`) i dzwonisz do niego z prośbą „załatw to”, odpowiada: „nie ma mnie tam, nie mogę nic zrobić”. `MissingGreenlet` to dokładnie ta odpowiedź.

### Naprawa

```python
# examples/24_missing_greenlet.py
import asyncio

from sqlalchemy import ForeignKey, String, select
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship, selectinload


class Base(DeclarativeBase):
    pass


class Author(Base):
    __tablename__ = "author"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    books: Mapped[list["Book"]] = relationship(back_populates="author")

    def __repr__(self) -> str:
        # ⚠️ UWAGA: odwołanie do relacji w __repr__ to klasyczna bomba w async
        return f"Author(id={self.id!r}, name={self.name!r})"


class Book(Base):
    __tablename__ = "book"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    author_id: Mapped[int] = mapped_column(ForeignKey("author.id"))
    author: Mapped["Author"] = relationship(back_populates="books")


engine = create_async_engine("sqlite+aiosqlite:///:memory:")
SessionFactory = async_sessionmaker(engine, expire_on_commit=False)  # ← naprawa nr 1


async def main() -> None:
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    async with SessionFactory() as session:
        session.add(Author(name="Lem", books=[Book(title="Solaris")]))
        await session.commit()

    async with SessionFactory() as session:
        # ❌ to wybuchnie: leniwe ładowanie w async
        # book = await session.scalars(select(Book).limit(1))
        # print(book.one().author.name)

        # ✅ naprawa nr 2: jawne strategie ładowania zamiast leniwego
        stmt = select(Book).options(selectinload(Book.author))
        books = (await session.scalars(stmt)).all()
        for book in books:
            print(book.title, "→", book.author.name)   # działa

    await engine.dispose()


if __name__ == "__main__":
    asyncio.run(main())
```

**Naprawa nr 3 — `AsyncAttrs` + `awaitable_attrs`.** Gdy naprawdę potrzebujesz leniwego ładowania w async, SQLAlchemy daje kontrolowany sposób: poczekać na nie jawnie.

```python
from sqlalchemy.ext.asyncio import AsyncAttrs


class Book(AsyncAttrs, Base):
    __tablename__ = "book"
    # ... kolumny jak wyżej


# gdzieś w kodzie async:
book = (await session.scalars(select(Book).limit(1))).one()
author = await book.awaitable_attrs.author   # ← explicit await, most działa
print(author.name)
```

**Naprawa nr 4 — `expire_on_commit=False`.** W async jest to praktycznie obowiązkowe. Domyślne `True` przedawnia obiekty po każdym `commit`, więc kolejne dotknięcie pola to IO — a to IO musi być opakowane w `await`, czego nie da się zrobić przy zwykłym dostępie do atrybutu.

**Naprawa nr 5 — nie dotykaj relacji w `__repr__` ani w `__str__`.** To najczęstsza pułapka na produkcji: wyjątek nie pojawia się tam, gdzie pracujesz, tylko tam, gdzie ktoś zaloguje obiekt.

> 🆕 **SQLAlchemy 2.1** — w serii 2.1 `greenlet` **nie instaluje się już automatycznie** razem z SQLAlchemy. Jeśli używasz async, musisz jawnie zainstalować pakiet:
> ```bash
> pip install "sqlalchemy[asyncio]"        # pociąga greenlet
> pip install "sqlalchemy[asyncio]" asyncpg
> ```
> Bez tego zobaczysz przy imporcie `sqlalchemy.ext.asyncio` komunikat o brakującym greenlet. Dodatkowo w 2.1 uporządkowano automatyczny `autoflush` w `Session` — działa bezwarunkowo, więc nie próbuj nim sterować, żeby „uniknąć” ładowania w async.

### Zapobieganie

- W projekcie async ustaw `lazy="raise"` na **wszystkich** relacjach. Wtedy leniwe odwołanie jest błędem jawnym, z komunikatem mówiącym „użyj selectinload”, zamiast niejasnego `MissingGreenlet`.
- Ustaw `expire_on_commit=False` w `async_sessionmaker`.
- Zakaz odwołań do relacji w `__repr__`, `__str__` i w f-stringach logów.
- `await engine.dispose()` przy zamykaniu aplikacji.

> 🧪 **Ćwiczenie** — zakomentuj `selectinload(Book.author)` w powyższym skrypcie i dopisz `lazy="raise"` na relacji. Porównaj komunikat błędu z `MissingGreenlet`. Który jest łatwiejszy w diagnozie i dlaczego?

---

## 3. „Nic się nie zapisało”

### Objaw

Skrypt kończy się bez błędu, `print(len(session.new))` pokazuje dodane obiekty, ale po restarcie aplikacji rekordów nie ma. Albo: `SELECT` w kliencie bazy pokazuje pustą tabelę.

### Przyczyna

To rodzina pięciu różnych przyczyn, które dają identyczny objaw. Dlatego trzeba je po kolei wykluczyć, a nie zgadywać.

| # | Przyczyna | Dowód |
|---|---|---|
| 1 | **Brak `commit()`** | obiekty są w `session.new`/`session.dirty`, ale transakcja nigdy się nie zakończyła |
| 2 | **Brak `add()`** | obiekt jest w stanie `transient`, nie ma go w `session.new` |
| 3 | **Sesja bez „wiązania” (bind)** | `session.get_bind()` rzuca `UnboundExecutionError` przy pierwszej operacji |
| 4 | **Zmiana poza sesją** (obiekt odłączony) | nie ma wyjątku; sesja po prostu nie wie o zmianie |
| 5 | **Zmiana niewidoczna dla SQLAlchemy** (mutowalny JSON, zmiana w kolekcji bez przypisania) | `session.is_modified(obj)` zwraca `False`, choć dane się różnią |

```python
# examples/24_not_saved.py
from sqlalchemy import JSON, String, create_engine, select
from sqlalchemy.ext.mutable import MutableDict
from sqlalchemy.orm import DeclarativeBase, Mapped, Session, mapped_column


class Base(DeclarativeBase):
    pass


class Article(Base):
    __tablename__ = "article"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    # ⚠️ bez MutableDict SQLAlchemy NIE zauważy zmiany w środku słownika
    meta: Mapped[dict] = mapped_column(JSON, default=dict)


engine = create_engine("sqlite:///:memory:", echo=False)
Base.metadata.create_all(engine)

# --- przyczyna 1: brak commit ---
with Session(engine) as session:
    session.add(Article(title="A"))
    # brak commit → wyjście z bloku = rollback + close
with Session(engine) as session:
    print("po braku commit:", session.scalar(select(Article).limit(1)))  # None

# --- przyczyna 2: brak add ---
with Session(engine) as session:
    article = Article(title="B")          # transient, nie w sesji
    print("w sesji?", article in session)  # False
    session.commit()
with Session(engine) as session:
    print("po braku add:", session.scalar(select(Article).limit(1)))      # None

# --- przyczyna 4: zmiana poza sesją ---
with Session(engine) as session:
    session.add(Article(title="C"))
    session.commit()
    detached = session.scalars(select(Article)).one()
    session.expunge(detached)              # obiekt odłączony
    detached.title = "C2"                  # sesja o tym nie wie
    session.commit()
with Session(engine) as session:
    print("po zmianie poza sesją:", session.scalars(select(Article)).one().title)  # C

# --- przyczyna 5: mutowalny JSON ---
with Session(engine) as session:
    article = Article(title="D", meta={"views": 0})
    session.add(article)
    session.commit()
    article.meta["views"] = 5              # zmiana „w środku” struktury
    session.commit()
with Session(engine) as session:
    print("po zmianie w JSON:", session.scalars(select(Article)).one().meta)       # {'views': 0}
```

### Naprawa

Trzy narzędzia, które odpowiadają na pytanie „dlaczego nic się nie zapisało” bez zgadywania:

```python
# Wgląd w stan sesji: co jest w kolejce do wysłania
print("new:", session.new)      # obiekty dodane, jeszcze nie INSERT-owane
print("dirty:", session.dirty)  # obiekty zmienione, jeszcze nie UPDATE-owane
print("deleted:", session.deleted)
print("identity map:", list(session.identity_map.keys()))

# Czy SQLAlchemy widzi zmianę konkretnego obiektu?
print("zmieniony?", session.is_modified(article))

# Co dokładnie SQLAlchemy zamierza wysłać? (flush nie kończy transakcji!)
session.flush()
print("po flush:", session.new, session.dirty)
```

Dla mutowalnych struktur — dwie naprawy:

```python
# Naprawa A: MutableDict (przeźroczyste dla kodu, jeden import więcej)
from sqlalchemy.ext.mutable import MutableDict


class Article(Base):
    __tablename__ = "article"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    meta: Mapped[dict] = mapped_column(MutableDict.as_mutable(JSON), default=dict)


article.meta["views"] = 5   # teraz SQLAlchemy widzi zmianę


# Naprawa B: całościowe przypisanie (bez importu, ale trzeba pamiętać)
article.meta = {**article.meta, "views": 5}

# Naprawa C: ręczne oznaczenie obiektu jako zmienionego
from sqlalchemy.orm.attributes import flag_modified

flag_modified(article, "meta")
```

> 🔬 **Pod maską** — dlaczego brak `commit` nie boli od razu
> ```sql
> INSERT INTO article (title, meta) VALUES (?, ?);
> -- ...i to jest tylko FLUSH. Dopóki nie ma COMMIT, baza trzyma dane w transakcji:
> ROLLBACK;      -- ← to wysyła close() bez commit w kontekście menedżera sesji
> ```
> Wyjście z `with Session(...)` **bez** `commit` wykonuje `rollback()`, a potem `close()`. To nie jest błąd biblioteki — to bezpieczne domyślne zachowanie: „skoro nie powiedziałeś, że kończymy, nie zmieniam świata”.

### Zapobieganie

1. W skryptach używaj `with Session(engine) as session, session.begin():` — wtedy commit (lub rollback przy wyjątku) jest gwarantowany przez kontekst.
2. W aplikacji nigdy nie pozwól, żeby granica transakcji znajdowała się „gdzieś” — ustal ją w warstwie aplikacji (`21_unit_of_work.md`).
3. Zawsze `session.add()` **przed** modyfikacją obiektu albo używaj `merge`.
4. Dla JSON-a używaj `MutableDict`; dla kolekcji — `MutableList`. To jedna linia kodu, która oszczędza godziny debugowania.
5. Test na „czy się zapisało” pisz zawsze po **nowej sesji** — testujący w tej samej sesji widzi obiekt w identity map i nie widzi problemu.

> 🧪 **Ćwiczenie** — dodaj do `examples/24_not_saved.py` szóstą przyczynę: dodanie obiektu do `books` (relacji kolekcji) bez `session.add()` i bez kaskady `save-update`. Sprawdź, czy się zapisał, i wyjaśnij, dlaczego tak.

---

## 4. Duplikaty wyników

### Objaw

Zapytanie zwraca 5 autorów zamiast 3 (`SELECT` się zgadza, ale `len()` nie), albo leci wyjątek:

```text
sqlalchemy.exc.InvalidRequestError: The unique() method must be invoked on this Result,
as it contains results that include joined eager loads against collections
```

### Przyczyna

Trzy różne przyczyny, ten sam objaw:

1. **Brak `distinct` przy JOIN z kolekcją.** Jeśli autor ma 4 książki, `JOIN book` wyprodukuje 4 wiersze dla tego autora. Baza nie „wie”, że interesują Cię autorzy, a nie pary (autor, książka).
2. **`joinedload` po kolekcji bez `unique()`.** SQLAlchemy wykrywa wynikające z tego duplikaty i wymaga, żebyś jawnie potwierdził: „tak, chcę jeden obiekt na grupę wierszy”.
3. **Źle zbudowany JOIN** — warunek w `where` zamiast w `onclause` przy `LEFT JOIN`, co dodatkowo rozjeżdża semantykę (patrz `06_joiny_i_zaawansowane_sql.md`).

> 💡 **Analogia — lista obecności a lista par**
> `JOIN` to zestawienie dwóch list: „uczniowie” i „oceny”. Zestawienie uczniów z ocenami daje pary (uczeń, ocena). Uczeń z pięcioma ocenami pojawi się pięć razy. Gdy chcesz listę obecności, musisz powiedzieć bazie: „daj mi każdego ucznia raz” — i to jest właśnie `distinct`. Gdy natomiast chcesz uczniów razem z ocenami w jednym worku, musisz dodać „worki połącz” — i to jest `unique()` przy eager loadingu kolekcji.

### Naprawa — trzy warianty

```python
# examples/24_duplicates.py
from sqlalchemy import ForeignKey, String, create_engine, select
from sqlalchemy.orm import (
    DeclarativeBase,
    Mapped,
    Session,
    joinedload,
    mapped_column,
    relationship,
    selectinload,
)


class Base(DeclarativeBase):
    pass


class Author(Base):
    __tablename__ = "author"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    books: Mapped[list["Book"]] = relationship(back_populates="author")


class Book(Base):
    __tablename__ = "book"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    author_id: Mapped[int] = mapped_column(ForeignKey("author.id"))
    author: Mapped["Author"] = relationship(back_populates="books")


engine = create_engine("sqlite:///:memory:")
Base.metadata.create_all(engine)
with Session(engine) as session:
    session.add_all(
        [
            Author(name="Lem", books=[Book(title="Solaris"), Book(title="Niezwyciężony")]),
            Author(name="Herbert", books=[Book(title="Diuna"), Book(title="Mesjasz Diuny")]),
        ]
    )
    session.commit()

with Session(engine) as session:
    # ❌ 1. JOIN po kolekcji bez distinct — duplikaty
    stmt = select(Author).join(Author.books)
    print("z JOIN bez distinct:", len(session.scalars(stmt).all()))     # 4, nie 2!

    # ✅ 1a. distinct — jeśli naprawdę chcesz autorów „raz na każdy”
    stmt = select(Author).join(Author.books).distinct()
    print("z distinct:", len(session.scalars(stmt).all()))              # 2

    # ❌ 2. joinedload kolekcji bez unique() — wyjątek
    # stmt = select(Author).options(joinedload(Author.books))
    # session.scalars(stmt).all()   # InvalidRequestError

    # ✅ 2a. unique() — potwierdzasz, że chcesz „jeden obiekt na grupę”
    stmt = select(Author).options(joinedload(Author.books))
    authors = session.scalars(stmt).unique().all()
    print("z unique():", len(authors))                                   # 2

    # ✅ 2b. selectinload zamiast joinedload — nie generuje duplikatów wcale
    stmt = select(Author).options(selectinload(Author.books))
    authors = session.scalars(stmt).all()
    print("z selectinload:", len(authors))                               # 2
```

> 🔬 **Pod maską** — dlaczego `joinedload` produkowuje duplikaty, a `selectinload` nie
> ```sql
> -- joinedload: JOIN w jednym zapytaniu → 4 wiersze dla 2 autorów
> SELECT author.id, author.name, book_1.id, book_1.title, book_1.author_id
> FROM author LEFT OUTER JOIN book AS book_1 ON author.id = book_1.author_id;
>
> -- selectinload: dwa zapytania, drugie z IN (...)
> SELECT author.id, author.name FROM author;
> SELECT book.author_id, book.id, book.title FROM book WHERE book.author_id IN (?, ?);
> ```
> Jedno zapytanie = wiersze „rozmnożone”, więc SQLAlchemy musi zgrupować je z powrotem i to Ty musisz powiedzieć `unique()`. Dwa zapytania = brak duplikatów, ale dodatkowa runda. Dla kolekcji `selectinload` jest zwykle lepszym domyślnym wyborem.

### Zapobieganie

1. `select(Encja).join(Encja.kolekcja)` **zawsze** kończ `.distinct()` albo — lepiej — zamień na `select(Encja).where(Encja.id.in_(select(...)))`.
2. `joinedload` na kolekcji: zawsze `.unique()`.
3. `selectinload` na kolekcjach jako domyślna strategia.
4. Test, który sprawdza `len()` wyniku — duplikaty nie rzucają wyjątku przy prostym `select(Author)`, więc bez testu ich nie złapiesz.

> 🆕 **SQLAlchemy 2.1** — `selectinload` dostał parametr `omit_join` dla relacji wiele-do-wielu, pozwalający pominąć dodatkowe złączenie w zapytaniu głównym, oraz parametr `chunksize` do dzielenia klauzuli `IN (...)` na porcje. To drugie jest istotne przy bardzo długich listach identyfikatorów: bazy mają limity liczby parametrów w `IN`, a `chunksize` pozwala nad tym zapanować bez ręcznego dzielenia.

---

## 5. N+1, czyli „strona ładuje się 3 sekundy”

### Objaw

Endpoint albo skrypt działa poprawnie, ale jest wolny. Log bazy (albo `pg_stat_statements`) pokazuje ten sam wzorzec zapytania powtórzony 50, 100, 1000 razy. Czas odpowiedzi rośnie liniowo z liczbą wierszy.

### Przyczyna

To skutek leniwego ładowania w pętli. Złożoność liczby zapytań wynosi $O(1 + N)$, gdzie $N$ to liczba encji; przy optymalnym eager loadingu — $O(1)$ lub $O(2)$.

### Pełna ścieżka diagnozy — krok po kroku

Ten schemat możesz stosować zawsze, bez zastanowienia.

**Krok 1 — policz zapytania.** Bez liczby nie ma diagnozy. Gotowy licznik:

```python
# examples/24_query_counter.py
from collections.abc import Generator
from contextlib import contextmanager

from sqlalchemy import Engine, event


@contextmanager
def count_queries(engine: Engine) -> Generator[list[str], None, None]:
    """Zbiera teksty wszystkich zapytań wysłanych w bloku `with`."""
    statements: list[str] = []

    def before_cursor_execute(
        conn: object,
        cursor: object,
        statement: str,
        parameters: object,
        context: object,
        executemany: bool,
    ) -> None:
        statements.append(statement)

    event.listen(engine, "before_cursor_execute", before_cursor_execute)
    try:
        yield statements
    finally:
        event.remove(engine, "before_cursor_execute", before_cursor_execute)
```

Użycie:

```python
with count_queries(engine) as queries:
    stmt = select(Author).limit(50)
    authors = session.scalars(stmt).all()
    for author in authors:
        _ = [book.title for book in author.books]   # leniwe ładowanie w pętli

print("liczba zapytań:", len(queries))   # 1 + 50 = 51
for q in queries[:3]:
    print(q)
```

**Krok 2 — zobacz zapytania.** Ten sam licznik drukuje SQL. Zobaczysz jedno zapytanie o autorów i $N$ identycznych zapytań o książki — to sygnatura N+1.

**Krok 3 — nazwij problem.** To nie jest „wolna baza”. To nie jest „słaby serwer”. To **wzorzec dostępu**: aplikacja pyta o dane w pętli zamiast z góry.

**Krok 4 — wybierz strategię ładowania.**

| Relacja | Zalecana strategia | Efekt |
|---|---|---|
| wiele-do-jednego (książka → autor) | `joinedload` | 1 zapytanie |
| jeden-do-wielu (autor → książki) | `selectinload` | 2 zapytania |
| zagnieżdżone relacje | `selectinload(...).joinedload(...)` | 2–3 zapytania |
| dane tylko do odczytu, bardzo duże | `load_only` + strumień (`yield_per`) | 1 zapytanie, mało pamięci |

**Krok 5 — zmierz ponownie.** Licznik zapytań jest jedynym obiektywnym dowodem poprawy.

```python
# ✅ po naprawie: 2 zapytania niezależnie od liczby autorów
with count_queries(engine) as queries:
    stmt = select(Author).options(selectinload(Author.books)).limit(50)
    authors = session.scalars(stmt).all()
    for author in authors:
        _ = [book.title for book in author.books]
print("liczba zapytań:", len(queries))   # 2
```

**Krok 6 — zabezpiecz się na przyszłość.** W teście:

```python
def test_list_authors_has_no_n_plus_one(engine: Engine, session: Session) -> None:
    with count_queries(engine) as queries:
        stmt = select(Author).options(selectinload(Author.books))
        session.scalars(stmt).all()
    assert len(queries) <= 2, f"regresja N+1: {len(queries)} zapytań"
```

oraz `lazy="raise"` na relacjach w środowisku testowym — wtedy każdy przypadkowy leniwy odczyt jest błędem, a nie cichym spadkiem wydajności.

### Zapobieganie

1. **W pętli nigdy nie odwołuj się do relacji**, której nie załadowałeś z góry.
2. W endpointach listujących ustal strategię ładowania **przed** napisaniem pętli.
3. Test na liczbę zapytań dla każdej ścieżki krytycznej.
4. `lazy="raise"` w testach i dla relacji, których nigdy nie chcesz ładować leniwie.
5. Ostrzeżenie: N+1 potrafi ukryć się w miejscu, którego nie podejrzewasz — w serializatorze („serializer dotyka relacji”), w `__repr__`, w logu audytowym, w `for` w szablonie. Włącz licznik i uruchom test z danymi, które mają duże kolekcje.

> 🔬 **Pod maską** — jak wygląda sygnatura N+1 w logu
> ```sql
> SELECT author.id, author.name FROM author LIMIT 50;
> SELECT book.id, book.title, book.author_id FROM book WHERE book.author_id = 1;
> SELECT book.id, book.title, book.author_id FROM book WHERE book.author_id = 2;
> -- ... × 48 kolejnych, każdy identyczny poza parametrem
> ```
> Ten wzorzec „jedno zapytanie, potem powtarzalne zapytanie z parametrem” to najbardziej rozpoznawalna sygnatura problemu. Jeśli widzisz go w logu — nie musisz szukać dalej.

> 🧪 **Ćwiczenie** — napisz skrypt, który generuje 200 autorów po 5 książek, i zmierz czas wykonania trzech wariantów raportu: `lazy`, `joinedload`, `selectinload`. Do każdego zmierz liczbę zapytań. Zapisz trzy liczby i trzy czasy w tabeli — to Twój punkt odniesienia na przyszłość.

---

## 6. Błędy transakcyjne

### Objaw

```text
sqlalchemy.exc.PendingRollbackError: This Session's transaction has been rolled back due to a
previous exception during flush. To begin a new transaction with this Session, first issue
Session.rollback(). Original exception was: ... (Background on this error at:
https://sqlalche.me/e/20/7s2a)
```

```text
sqlalchemy.exc.InvalidRequestError: This session is in 'prepared' state; no further SQL can be
emitted within this transaction.
```

```text
sqlalchemy.exc.ResourceClosedError: This Connection is closed
```

### Przyczyna

Sesja ma **maszynę stanów transakcji**. Gdy w trakcie `flush()` wystąpi błąd bazy (np. `IntegrityError`), SQLAlchemy **musi** wycofać transakcję — to jedyny bezpieczny stan. Ale sesja nie robi tego automatycznie „za Ciebie”: oznacza się jako wymagająca rollbacku i **blokuje** dalsze operacje, dopóki nie wykonasz `session.rollback()`. To celowe: lepiej dostać jawny błąd niż wysłać połowę zmian.

Analogia: to jak kuchnia po wybuchu czajnika. Nikt nie kontynuuje gotowania, dopóki nie posprząta. `PendingRollbackError` to napis „najpierw posprzątaj”.

### Naprawa

```python
# examples/24_transactions.py
from sqlalchemy import String, create_engine, select
from sqlalchemy.exc import IntegrityError, PendingRollbackError
from sqlalchemy.orm import DeclarativeBase, Mapped, Session, mapped_column


class Base(DeclarativeBase):
    pass


class User(Base):
    __tablename__ = "user_account"
    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(200), unique=True)


engine = create_engine("sqlite:///:memory:")
Base.metadata.create_all(engine)

with Session(engine) as session:
    session.add(User(email="a@example.com"))
    session.commit()

# --- ❌ ŹLE: łapiemy wyjątek i lecimy dalej ---
with Session(engine) as session:
    try:
        session.add(User(email="a@example.com"))   # duplikat
        session.flush()
    except IntegrityError:
        print("złapane, ale...")
    try:
        session.add(User(email="b@example.com"))
        session.flush()          # PendingRollbackError!
    except PendingRollbackError as exc:
        print("PendingRollbackError:", str(exc)[:60], "...")

# --- ✅ DOBRZE: rollback jako część obsługi wyjątku ---
with Session(engine) as session:
    try:
        session.add(User(email="a@example.com"))
        session.flush()
    except IntegrityError:
        session.rollback()       # ← kluczowy krok
        print("po rollbacku sesja znów działa")
    session.add(User(email="b@example.com"))
    session.commit()
    print("użytkownicy:", session.scalars(select(User.email)).all())
```

Wzorzec `try/except/finally` dla kodu produkcyjnego:

```python
def register_user(session: Session, email: str) -> User | None:
    try:
        user = User(email=email)
        session.add(user)
        session.flush()          # wymuszamy błąd tutaj, nie przy commicie
        return user
    except IntegrityError:
        session.rollback()
        return None              # albo podnieś wyjątek domenowy: EmailAlreadyExists
    finally:
        # nic tu nie rób, chyba że naprawdę wiesz, co — zamknięcie należy do UoW
        pass
```

### Trzy warianty tego samego problemu

| Objaw | Przyczyna | Naprawa |
|---|---|---|
| `PendingRollbackError` | kontynuacja pracy po nieudanym `flush` bez rollbacku | `session.rollback()` w `except` |
| `This session is in 'prepared' state` | transakcja dwufazowa (2PC) po `prepare()` lub commit, który nie zakończył się zgodnie z oczekiwaniem | `session.rollback()`; nie używaj tej samej sesji dalej |
| `ResourceClosedError: This Connection is closed` | praca na połączeniu/sesji po `close()`, np. w `finally` przed użyciem | przenieś pracę przed `close()`; sesję otwieraj w miejscu użycia |

> ⚠️ **Pułapka — commit w niewłaściwym miejscu**
> Commit to **granica transakcji**, nie „dodatek na koniec funkcji”. Jeśli `commit()` znajduje się w środku pętli, zaczynasz i kończysz transakcję 1000 razy: to wolne, niewspółbieżne i łamie atomowość (część danych zapisana, część nie). Jeśli `commit()` jest w repozytorium **i** w UoW, dostajesz podwójny commit i transakcję pociętą na dwie — błędy trudne do znalezienia. Zasada: **commit jest w jednym miejscu w całej ścieżce wykonania** (`21_unit_of_work.md`).

### Zapobieganie

1. Ustal jedno miejsce, w którym kończy się transakcja. Zwykle jest to warstwa aplikacji / UoW.
2. Zawsze obsługuj wyjątek + `rollback` w jednym bloku — nigdy „złapię i pójdę dalej”.
3. `flush()` przed operacją, która może się nie udać, jeśli chcesz złapać błąd w miejscu wywołania, a nie dopiero przy commicie.
4. Nie loguj i nie wysyłaj e-maili wewnątrz transakcji — wydłużasz ją i narażasz na rollback po wysłaniu (wzorzec outbox, `21_unit_of_work.md`).
5. W testach: test, w którym operacja rzuca wyjątek, i sprawdzenie, że dane **nie** zostały zapisane.

> 🧪 **Ćwiczenie** — napisz funkcję `transfer_books(session, from_author, to_author, titles)`, która przenosi książki i rzuca wyjątek w połowie. Sprawdź, że po błędzie żadna książka nie zmieniła autora (rola transakcji). Potem dodaj `begin_nested()` i sprawdź, jak zmienia się zachowanie.

---

## 7. IntegrityError — jak czytać komunikat bazy

### Objaw

```text
sqlalchemy.exc.IntegrityError: (sqlite3.IntegrityError) UNIQUE constraint failed:
user_account.email
[SQL: INSERT INTO user_account (email) VALUES (?)]
[parameters: ('a@example.com',)]
```

albo dla PostgreSQL:

```text
sqlalchemy.exc.IntegrityError: (psycopg.errors.ForeignKeyViolation) insert or update on table
"book" violates foreign key constraint "book_author_id_fkey"
DETAIL:  Key (author_id)=(42) is not present in table "author".
```

### Przyczyna

Baza odmówiła zapisu, bo naruszyłby ona **ograniczenie integralności** (constraint): unikalność, klucz obcy, `NOT NULL`, `CHECK`. To nie jest błąd SQLAlchemy — SQLAlchemy tylko przekazuje komunikat dialektu. Dlatego tekst komunikatu różni się między SQLite, PostgreSQL i MySQL.

**Kluczowa umiejętność: rozpoznać typ naruszenia po treści.**

| Typ naruszenia | SQLite | PostgreSQL |
|---|---|---|
| unikalność / klucz główny | `UNIQUE constraint failed: tabela.kolumna` | `duplicate key value violates unique constraint "..."` |
| klucz obcy | `FOREIGN KEY constraint failed` | `violates foreign key constraint "..."` + `DETAIL: Key (...) is not present` |
| `NOT NULL` | `NOT NULL constraint failed: tabela.kolumna` | `null value in column "..." violates not-null constraint` |
| `CHECK` | `CHECK constraint failed: nazwa` | `violates check constraint "..."` |

### Naprawa

Dwie warstwy: **techniczna** (co zrobić z sesją) i **semantyczna** (co powiedzieć użytkownikowi).

```python
# examples/24_integrity.py
from sqlalchemy import ForeignKey, String, create_engine
from sqlalchemy.exc import IntegrityError
from sqlalchemy.orm import DeclarativeBase, Mapped, Session, mapped_column


class Base(DeclarativeBase):
    pass


class Author(Base):
    __tablename__ = "author"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100), unique=True)


class Book(Base):
    __tablename__ = "book"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    author_id: Mapped[int] = mapped_column(ForeignKey("author.id"))


class DomainError(Exception):
    """Wyjątek warstwy domenowej — nie przecieka do API jako IntegrityError."""


class AuthorAlreadyExists(DomainError):
    pass


class AuthorNotFound(DomainError):
    pass


engine = create_engine("sqlite:///:memory:")
Base.metadata.create_all(engine)


def create_author(session: Session, name: str) -> Author:
    author = Author(name=name)
    session.add(author)
    try:
        session.flush()
    except IntegrityError as exc:
        session.rollback()
        # Tłumaczymy błąd dialektu na błąd aplikacji
        if "author.name" in str(exc.orig):
            raise AuthorAlreadyExists(name) from exc
        raise DomainError(str(exc.orig)) from exc
    return author


def add_book(session: Session, title: str, author_id: int) -> Book:
    book = Book(title=title, author_id=author_id)
    session.add(book)
    try:
        session.flush()
    except IntegrityError as exc:
        session.rollback()
        if "FOREIGN KEY" in str(exc.orig).upper():
            raise AuthorNotFound(author_id) from exc
        raise DomainError(str(exc.orig)) from exc
    return book


with Session(engine) as session:
    create_author(session, "Lem")
    try:
        create_author(session, "Lem")
    except AuthorAlreadyExists:
        print("Autor już istnieje — obsłużone domenowo")
    try:
        add_book(session, "Nieistniejąca", author_id=999)
    except AuthorNotFound:
        print("Autor 999 nie istnieje — obsłużone domenowo")
```

> 🔬 **Pod maską** — skąd SQLAlchemy wie, co zawiodło
> Wyjątek `IntegrityError` trzyma **oryginalny wyjątek sterownika** w atrybucie `.orig`, a skompilowane zapytanie i parametry w atrybutach `statement`/`params` oraz w tekstowej reprezentacji. Detekcja po `str(exc.orig)` jest przenośna „w miarę”, ale krucha — komunikaty różnią się między wersjami sterowników. W projektach produkcyjnych lepiej wykrywać problem **przed** wysłaniem INSERT-a (zapytanie kontrolne albo `on_conflict_do_nothing()`) albo polegać na nazwach ograniczeń z `naming_convention` (`03_metadata_ddl.md`) i dopasowywać po nazwie, nie po treści.

### Zapobieganie

1. Ustaw `naming_convention` w `MetaData` — wtedy nazwy ograniczeń są przewidywalne we wszystkich bazach.
2. Mapuj `IntegrityError` na wyjątki domenowe **w jednym miejscu** (repozytorium albo serwis), nigdy w kontrolerze HTTP.
3. Dla operacji, które nie mogą się nie udać (idempotentne wstawianie), używaj `insert().on_conflict_do_nothing()` / `on_conflict_do_update()` — wtedy nie ma wyjątku, tylko policzalny wynik.
4. `NOT NULL` w bazie realizuj jako `Mapped[str]` + jawne `nullable=False` w kodzie. Walidacja w Pythonie bez ograniczenia w bazie to nie zabezpieczenie — to sugestia.

> ⚠️ **Pułapka — SQLite domyślnie nie egzekwuje kluczy obcych**
> W SQLite ograniczenia `FOREIGN KEY` są domyślnie **wyłączone**. Powyższy przykład zadziałał, bo... sprawdziliśmy `UNIQUE`, ale `FOREIGN KEY constraint failed` na czystym SQLite zwykle nie wystąpi! Trzeba je włączyć per połączenie:
> ```python
> from sqlalchemy import Engine, event
>
> @event.listens_for(Engine, "connect")
> def _enable_sqlite_fk(dbapi_connection: object, connection_record: object) -> None:
>     cursor = dbapi_connection.cursor()      # type: ignore[attr-defined]
>     cursor.execute("PRAGMA foreign_keys=ON")
>     cursor.close()
> ```
> To jedna z najczęstszych „cichych” różnic między środowiskiem testowym (SQLite) a produkcyjnym (PostgreSQL).

---

## 8. Problemy migracyjne

### Objaw 1 — `ALTER` nie działa na SQLite

```text
sqlalchemy.exc.OperationalError: (sqlite3.OperationalError) near "ALTER": syntax error
[SQL: ALTER TABLE book DROP COLUMN language]
```

**Przyczyna.** SQLite historycznie nie wspierał większości `ALTER TABLE`. Alembic potrafi obejść ten problem techniką **batch mode**: tworzy nową tabelę z pożądanym schematem, kopiuje dane, kasuje starą i zmienia nazwę. Ale trzeba mu to włączyć.

**Naprawa.** W `alembic/env.py`:

```python
# alembic/env.py (fragment)
def run_migrations_online() -> None:
    connectable = config.attributes.get("connection")
    # ...
    with engine.connect() as connection:
        context.configure(
            connection=connection,
            target_metadata=target_metadata,
            compare_type=True,
            render_as_batch=True,          # ← kluczowe dla SQLite
        )
```

albo ręcznie w migracji:

```python
# alembic/versions/xxxx_remove_language.py
def upgrade() -> None:
    with op.batch_alter_table("book", schema=None) as batch_op:
        batch_op.drop_column("language")


def downgrade() -> None:
    with op.batch_alter_table("book", schema=None) as batch_op:
        batch_op.add_column(sa.Column("language", sa.String(length=10), nullable=True))
```

### Objaw 2 — konflikt `heads`

```text
alembic.util.exc.CommandError: Multiple head revisions are present for given argument 'head';
please specify a specific target revision, '<branchname>@head' to narrow to a specific head,
or 'heads' for all heads
```

**Przyczyna.** Dwie osoby (albo dwie gałęzie) wygenerowały migrację z tego samego punktu. Powstały dwie niezależne linie historii — dwa „łańcuchy” zmian zamiast jednego.

**Naprawa.**

```bash
alembic heads            # pokazuje oba wierzchołki
alembic merge -m "merge heads" head1 head2
alembic upgrade head
```

**Zapobieganie.** Rebase/merge z głównej gałęzi **przed** wygenerowaniem migracji; jedna migracja na pull request; w CI krok sprawdzający `alembic heads | wc -l == 1`.

### Objaw 3 — `create_all()` i Alembic w jednym projekcie

```text
sqlalchemy.exc.OperationalError: table book already exists
```

albo — gorszy wariant, **bez błędu**: w bazie brakuje kolumny, która jest w modelu, bo `create_all` nie robi `ALTER`.

**Przyczyna.** `create_all` tworzy brakujące tabele i nic więcej. Alembic prowadzi historię zmian. Gdy oba mechanizmy działają równolegle na tej samej bazie, stan `alembic_version` rozjeżdża się z rzeczywistością: migracja myśli, że kolumna istnieje, a w bazie jej nie ma (albo odwrotnie).

**Naprawa.** Wybierz jedno źródło prawdy — **migracje**. `create_all` zostaw dla testów (`18_testowanie.md`) albo dla pierwszej wersji prototypu. Jeśli stan się rozjechał:

```bash
alembic current                     # co myśli Alembic
alembic stamp <revision>            # oznacz stan faktyczny bez wykonywania migracji
```

### Objaw 4 — migracja dropuje kolumnę z danymi

Autogenerate wygenerował `op.drop_column(...)` i `op.add_column(...)` tam, gdzie chodziło tylko o **zmianę nazwy**. Efekt: dane znikają.

**Przyczyna.** Alembic porównuje **schemat**, nie intencje. Zmiana nazwy kolumny wygląda dla niego jak „usunięto X, dodano Y”. To samo dotyczy zmiany typu na SQLite i zmian w `Enum`.

**Naprawa.** Zawsze przeglądaj wygenerowany plik i zamień drop/add na `alter_column`:

```python
def upgrade() -> None:
    op.alter_column("book", "title", new_column_name="name")
```

**Zapobieganie.** Zasada żelazna: **wygenerowana migracja to szkic, nie gotowiec**. Reguła druga: każda migracja ma napisany `downgrade`, który był przetestowany.

> 🆕 **SQLAlchemy 2.1 / Alembic 1.19** — aktualna seria Alembica (1.19.x) potrafi wykryć w `autogenerate` nazwane ograniczenia `CHECK`. Wcześniej wymagały ręcznego dopisania — co bywało źródłem „działa lokalnie, nie działa u kolegi”. Jeśli używasz Alembica w wersji starszej niż 1.19, nazwane `CheckConstraint` (i ich usuwanie) dopisuj ręcznie.

---

## 9. Problemy z typami danych

### 9.1 `datetime` naiwny vs ze strefą

**Objaw.**

```text
TypeError: can't compare offset-naive and offset-aware datetimes
```

albo — cichsze i gorsze — zapytanie zwraca puste wyniki, choć w bazie widać wiersze.

**Przyczyna.** `DateTime(timezone=True)` w modelu to **prośba**, nie gwarancja. To, czy baza faktycznie przechowuje informację o strefie, zależy od dialektu. PostgreSQL ma `TIMESTAMPTZ` i robi to dobrze. SQLite zapisuje daty jako tekst i oddaje je **bez** informacji o strefie — nawet jeśli w kodzie prosiłeś o `timezone=True`.

**Naprawa.**

```python
# examples/24_timezone.py
from datetime import UTC, datetime

from sqlalchemy import DateTime, create_engine, select
from sqlalchemy.orm import DeclarativeBase, Mapped, Session, mapped_column


class Base(DeclarativeBase):
    pass


class Event(Base):
    __tablename__ = "event"
    id: Mapped[int] = mapped_column(primary_key=True)
    # Docker Postgres: postgresql+psycopg://... — tam timezone=True naprawdę działa
    happened_at: Mapped[datetime] = mapped_column(DateTime(timezone=True))


engine = create_engine("sqlite:///:memory:")
Base.metadata.create_all(engine)

with Session(engine) as session:
    session.add(Event(happened_at=datetime.now(UTC)))   # ← zawsze UTC w kodzie
    session.commit()

with Session(engine) as session:
    event = session.scalars(select(Event)).one()
    print("typ w bazie (SQLite):", event.happened_at.tzinfo)  # None — naiwny!

# ✅ Konwencja bezpieczna dla obu baz: normalizuj przy odczycie i przy zapisie
def as_utc(value: datetime) -> datetime:
    return value.replace(tzinfo=UTC) if value.tzinfo is None else value.astimezone(UTC)
```

**Zapobieganie.** Trzy reguły: (1) w kodzie tworzysz **zawsze** `datetime.now(UTC)`, nigdy `datetime.now()`; (2) przy odczycie normalizujesz do UTC; (3) nie porównuj dat z poziomu Pythona tam, gdzie może to zrobić baza — porównuj w `where()` (w SQL obie strony są równie „naiwne” albo równie „świadome”).

### 9.2 `Numeric` vs `float`

**Objaw.**

```text
TypeError: unsupported operand type(s) for +: 'decimal.Decimal' and 'float'
```

**Przyczyna.** `Numeric` oddaje `Decimal`, `Float` — `float`. Python nie dodaje ich automatycznie.

**Naprawa.** Konsekwentnie używaj `Decimal` w całej ścieżce: w modelu, w serwisie, w DTO, w schemacie Pydantic (`condecimal` albo pole `Decimal`). Nie konwertuj „tylko w jednym miejscu”.

```python
from decimal import Decimal

from sqlalchemy import Numeric
from sqlalchemy.orm import Mapped, mapped_column


class Invoice(Base):
    __tablename__ = "invoice"
    id: Mapped[int] = mapped_column(primary_key=True)
    # ✅ pieniądze zawsze jako Numeric, nigdy Float
    amount: Mapped[Decimal] = mapped_column(Numeric(12, 2))
```

**Zapobieganie.** Reguła: **pieniądze nigdy nie są `Float`**. `Float` to błąd reprezentacji binarnej — `0.1 + 0.2 != 0.3`. Kwoty, stawki, podatki → `Numeric`. Współczynniki naukowe → `Float`.

### 9.3 `Enum` + migracje

**Objaw.** Po dodaniu nowej wartości do `Enum` w Pythonie i wdrożeniu:

```text
psycopg.errors.InvalidTextRepresentation: invalid input value for enum book_status: "archived"
```

**Przyczyna.** W PostgreSQL typ `ENUM` jest **typem w bazie**, a nie tylko ograniczeniem w kodzie. Zmiana w Pythonie nie zmienia typu w bazie. SQLite w tym miejscu jest „luźniejszy” (używa `VARCHAR` + `CHECK`), więc problem nie pojawi się lokalnie.

**Naprawa.**

```python
# Alembic, PostgreSQL: dodanie wartości do istniejącego ENUM
def upgrade() -> None:
    op.execute("ALTER TYPE book_status ADD VALUE IF NOT EXISTS 'archived'")
```

**Zapobieganie.** Dla wartości, których lista rośnie (statusy, kategorie), rozważ **tabelę słownikową** zamiast `Enum`: dodawanie wartości to wtedy zwykły `INSERT`, a nie operacja DDL. Szczegóły i porównanie kompromisów — `12_typy_i_wlasne_typy.md`.

### 9.4 JSON bez mutowalności

**Objaw.** Zmiana klucza w słowniku JSON „nie zapisuje się”. Brak wyjątku, brak śladu w logu SQL.

**Przyczyna.** SQLAlchemy śledzi **przypisania atrybutów**, nie mutacje wewnątrz struktur. `article.meta["views"] = 5` nie jest przypisaniem atrybutu `meta`, więc nie ma czego zauważyć. Analogia: kartoteka widzi, że **podmieniłeś kartkę**, ale nie widzi, że **dopisałeś coś ołówkiem** na tej samej kartce.

**Naprawa.** `MutableDict.as_mutable(JSON)` (i odpowiednio `MutableList`, `MutableSet`) albo całościowe przypisanie `article.meta = {**article.meta, "views": 5}`.

---

## 10. Problemy z połączeniami

### 10.1 „server closed the connection unexpectedly”

**Objaw.**

```text
sqlalchemy.exc.OperationalError: (psycopg.OperationalError) server closed the connection
unexpectedly
```

Zdarza się po okresie bezczynności (noc, weekend, dłuższa przerwa między żądaniami) albo za load balancerem.

**Przyczyna.** Firewall, proxy albo sama baza zamykają bezczynne połączenia. Pula połączeń aplikacji przechowuje je nadal i nie wie, że są martwe. Pierwsze użycie takiego połączenia kończy się błędem. To zjawisko „połączenia zombie”.

**Naprawa.**

```python
# examples/24_engine_options.py
from sqlalchemy import create_engine

engine = create_engine(
    "postgresql+psycopg://user:pass@localhost/db",
    pool_size=10,
    max_overflow=20,
    pool_timeout=30,
    pool_recycle=1800,      # ← recykling po 30 min: krócej niż timeout sieci/bazy
    pool_pre_ping=True,     # ← „ping” przed użyciem połączenia z puli
    echo_pool="debug",      # ← diagnostyka puli (na czas debugowania!)
)
```

`pool_pre_ping=True` wykonuje tanie sprawdzenie połączenia przy wypożyczeniu; jeśli jest martwe, SQLAlchemy je odrzuca i tworzy nowe. Koszt jest niewielki, a oszczędność czasu debugowania ogromna.

> 💡 **Analogia — `pool_recycle` to data przydatności**
> Pula połączeń to lodówka z jogurtami. `pool_recycle` to data przydatności: po jej upływie jogurt wyrzucamy, nawet jeśli wygląda dobrze. `pool_pre_ping` to powąchanie przed użyciem — tanie i ratuje przed niemiłą niespodzianką.

### 10.2 Wyciek połączeń

**Objaw.**

```text
TimeoutError: QueuePool limit of size 5 overflow 10 reached, connection timed out, timeout 30.00
```

Aplikacja „zawiesza się” po pewnym czasie albo po N żądaniach przestaje odpowiadać.

**Przyczyna.** Połączenia są wypożyczane i **nie zwracane**. Typowe źródła: sesja utworzona bez zamknięcia, `Connection` trzymane w globalnej zmiennej, brak `close()` w `finally`, sesja żyjąca dłużej niż blok kodu.

**Naprawa.** Diagnostyka i naprawa wzorca:

```python
# 1. Sprawdź stan puli
print(engine.pool.status())
# Pool size: 5  Connections in pool: 1  Current Overflow: -4  Current Checked out connections: 4

# 2. Zobacz, kto trzyma połączenia (na czas debugowania)
from sqlalchemy import event

@event.listens_for(engine, "checkout")
def _on_checkout(dbapi_conn, connection_rec, connection_proxy) -> None:
    print("CHECKOUT", id(connection_rec))

@event.listens_for(engine, "checkin")
def _on_checkin(dbapi_conn, connection_rec) -> None:
    print("CHECKIN ", id(connection_rec))
```

```python
# ❌ ŹLE — połączenie nigdy nie wraca do puli
session = Session(engine)
session.execute(select(Author))
# ...i nic więcej; brak close()

# ✅ DOBRZE — kontekst menedżer zawsze zwraca połączenie
with Session(engine) as session:
    session.execute(select(Author))
```

**Zapobieganie.** (1) Zawsze `with` dla `Session` i `Connection`. (2) `pool_timeout` ustaw świadomie — to bezpiecznik, który zamienia „zawieszenie aplikacji” w czytelny błąd. (3) W FastAPI sesja w `Depends` z `yield` i `finally`. (4) Monitoruj `engine.pool.checkedout()` w metrykach aplikacji.

> 🧠 **Dlaczego tak jest — pula jest mała celowo**
> `pool_size=5` nie znaczy „aplikacja obsłuży 5 użytkowników”. Znaczy: maksymalnie 5 jednoczesnych operacji na bazie z tego procesu, plus `max_overflow` w chwilach szczytu. Baza ma skończoną liczbę procesów roboczych; 200 połączeń z jednej aplikacji to nie wydajność, a sposób na zatkanie serwera. Mała pula + krótkie transakcje bije dużą pulę + długie transakcje.

### 10.3 `idle in transaction`

**Objaw.** Baza „zwalnia”, liczba połączeń rośnie, `pg_stat_activity` pokazuje sesje w stanie `idle in transaction`.

```sql
-- PostgreSQL: kto trzyma transakcje otwarte i jak długo?
SELECT pid, state, wait_event_type, now() - xact_start AS txn_age, left(query, 80) AS q
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY txn_age DESC;
```

**Przyczyna.** Sesja otworzyła transakcję (np. przez `flush`/`SELECT`) i nie zamknęła jej. Transakcja „wisi”, blokując `VACUUM` i trzymając blokady. To najczęstszy efekt uboczny trzymania jednej sesji przez cały request z przerwą na wywołanie zewnętrznego API.

**Naprawa.** Ogranicz czas: ustaw `statement_timeout` i `lock_timeout` po stronie połączenia, oraz skróć życie transakcji.

```python
engine = create_engine(
    "postgresql+psycopg://user:pass@localhost/db",
    connect_args={
        "options": "-c statement_timeout=5000 -c idle_in_transaction_session_timeout=10000",
    },
)
```

---

## 11. Problemy wydajnościowe

### 11.1 Indeks na kolumnie z funkcją — niewykorzystany

**Objaw.** Zapytanie na tabeli z 500 tys. wierszy robi `Seq Scan`, mimo że kolumna ma indeks. `EXPLAIN ANALYZE` pokazuje pełne przejście.

**Przyczyna.** Warunek nie pasuje do indeksu „gołej” kolumny:

```sql
WHERE lower(author.name) = 'lem'     -- indeks na author.name NIE pomoże
```

Indeks na `name` jest indeksem na wartościach kolumny, nie na wyniku funkcji. Baza nie „widzi” związku.

**Naprawa.** Indeks **funkcyjny** (wyrażeniowy) plus zapytanie w tej samej postaci:

```python
# examples/24_indexes.py
from sqlalchemy import Index, String, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class Author(Base):
    __tablename__ = "author"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100), index=True)   # zwykły indeks
    __table_args__ = (
        Index("ix_author_name_lower", func.lower(name)),          # indeks funkcyjny
    )


# ✅ warunek zgodny z indeksem funkcyjnym — PostgreSQL użyje ix_author_name_lower
stmt = select(Author).where(func.lower(Author.name) == "lem")
```

**Zapobieganie.** Zawsze pisz `EXPLAIN ANALYZE` przed i po dodaniu indeksu. Indeks bez pomiaru to wiara, nie inżynieria. Uwaga na pułapkę odwrotną: **indeksy nieużywane zwalniają zapisy** — każdy `INSERT` musi je zaktualizować.

### 11.2 `LIKE '%x%'`

**Objaw.** Wyszukiwanie fragmentu tekstu trwa sekundy na dużym zbiorze i całkowicie ignoruje indeks.

**Przyczyna.** Indeks B-tree porządkuje wartości od lewej. Wzorzec zaczynający się od `%` nie ma punktu startu — baza musi przejrzeć wszystko. Złożoność: $O(N)$ zamiast $O(\log N)$.

**Naprawa (PostgreSQL).** Rozszerzenie `pg_trgm` + indeks GIN:

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX ix_book_title_trgm ON book USING gin (title gin_trgm_ops);
```

W SQLAlchemy:

```python
from sqlalchemy import Index
from sqlalchemy.dialects.postgresql import TEXT

Index("ix_book_title_trgm", Book.__table__.c.title, postgresql_using="gin",
      postgresql_ops={"title": "gin_trgm_ops"})
```

**Alternatywa.** Pełnotekstowe wyszukiwanie (`tsvector` w PostgreSQL) — właściwe narzędzie do wyszukiwania po słowach, a nie po fragmentach.

### 11.3 `OFFSET` na dużych zbiorach

**Objaw.** Strona 1 listy odpowiada w 20 ms, strona 5000 — w 4 sekundy. Czasy rosną liniowo z numerem strony.

**Przyczyna.** `OFFSET 100000 LIMIT 20` nie znaczy „przeskocz sto tysięcy wierszy”. Baza musi **przejrzeć i odrzucić** sto tysięcy wierszy, a dopiero potem oddać dwadzieścia. Do tego przy zmieniających się danych paginacja przez offset **gubi i powtarza** elementy.

**Naprawa — paginacja keyset (kursorowa).**

```python
# ❌ OFFSET: koszt rośnie z numerem strony
stmt = select(Book).order_by(Book.id).offset(100_000).limit(20)

# ✅ KEYSET: stały koszt, niezależny od „głębokości”
last_seen_id = 100_000
stmt = select(Book).where(Book.id > last_seen_id).order_by(Book.id).limit(20)
```

> 🔬 **Pod maską**
> ```sql
> -- OFFSET 100000: plan musi wygenerować 100020 wierszy i wyrzucić 100000
> SELECT id, title FROM book ORDER BY id OFFSET 100000 LIMIT 20;
>
> -- KEYSET: baza startuje od indeksu w miejscu id > 100000
> SELECT id, title FROM book WHERE id > 100000 ORDER BY id LIMIT 20;
> ```
> Implementację `Page[T]` z oboma wariantami znajdziesz w `20_repository.md`.

### 11.4 Brak limitu w „ładuj wszystko”

**Objaw.** Aplikacja działa na 100 rekordach, po pół roku produkcji zaczyna zjadać gigabajty RAM i restartuje się pod `OOMKilled`.

**Przyczyna.** `session.scalars(select(Book)).all()` ładuje **całą tabelę** do pamięci, tworząc obiekty ORM (każdy z identity map, cyklem życia i narzutem). Pamięć rośnie liniowo z liczbą wierszy.

**Naprawa.**

```python
# ✅ strumieniowanie: stały ślad pamięciowy
for book in session.scalars(select(Book).execution_options(yield_per=1000)):
    process(book)

# ✅ agregacja w bazie zamiast w Pythonie
total = session.scalar(select(func.count()).select_from(Book))

# ✅ projekcja kolumn zamiast całych encji (raporty)
stmt = select(Book.id, Book.title).execution_options(yield_per=1000)
```

**Zapobieganie.** Każde zapytanie zwracające listę **musi** mieć limit — w API, w CLI, w zadaniu wsadowym. Domyślny limit to nie „brak limitu”, to bomba zegarowa.

---

## 12. Problemy z testami

### 12.1 SQLite in-memory bez `StaticPool`

**Objaw.**

```text
sqlalchemy.exc.OperationalError: (sqlite3.OperationalError) no such table: author
```

Mimo że `Base.metadata.create_all(engine)` zostało wywołane.

**Przyczyna.** Baza `sqlite:///:memory:` **istnieje tylko w obrębie jednego połączenia**. Domyślnie SQLAlchemy używa dla SQLite w pamięci `SingletonThreadPool`, który w jednym wątku trzyma jedno połączenie — dlatego prosty skrypt w jednym wątku działa. Wtedy wchodzi wątek testowy (np. `TestClient` FastAPI, `ThreadPoolExecutor`) i dostaje **nowe połączenie = nową, pustą bazę**. Stąd „nie ma tabeli”, choć je tworzyłeś.

**Naprawa.**

```python
# tests/conftest.py
from collections.abc import Iterator

import pytest
from sqlalchemy import StaticPool, create_engine
from sqlalchemy.orm import DeclarativeBase, Session, sessionmaker


class Base(DeclarativeBase):
    pass


@pytest.fixture(scope="session")
def engine():
    # ✅ StaticPool: JEDNO połączenie współdzielone przez wszystkich
    engine = create_engine(
        "sqlite:///:memory:",
        connect_args={"check_same_thread": False},   # jeden wątek testowy
        poolclass=StaticPool,                         # jedno połączenie, jedna baza
    )
    Base.metadata.create_all(engine)
    yield engine
    engine.dispose()


@pytest.fixture
def session(engine) -> Iterator[Session]:
    connection = engine.connect()
    transaction = connection.begin()                  # transakcja zewnętrzna
    session = Session(bind=connection, join_transaction_mode="create_savepoint")
    yield session
    session.close()
    transaction.rollback()                            # izolacja między testami
    connection.close()
```

**Zapobieganie.** Dla SQLite in-memory zawsze `StaticPool` + `check_same_thread=False`. Jeśli testujesz współbieżność prawdziwie lub chcesz wierności dialektu — używaj PostgreSQL w kontenerze (`testcontainers`), a nie SQLite.

### 12.2 Dzielona sesja między testami

**Objaw.** Testy przechodzą pojedynczo, ale nie w całości; albo kolejność uruchomienia zmienia wynik.

**Przyczyna.** Sesja utworzona w `scope="session"` (albo, gorzej, jako globalna zmienna) kumuluje stan: identity map, obiekty `pending`, przedawnienia. Jeden test widzi dane drugiego.

**Naprawa.** Fixture funkcjonalna (`scope="function"`) z transakcją zewnętrzną i rollbackiem po teście (kod wyżej). Albo — gdy potrzebna jest sesja sesyjna — czyść stan jawnie: `session.rollback()`, `session.expunge_all()`, `session.close()`.

### 12.3 Testy zależne od kolejności

**Objaw.** `pytest -q` przechodzi, `pytest -q -p no:randomly` nie; albo testy przechodzą w innym porządku na CI.

**Przyczyna.** Test zakłada istnienie danych utworzonych przez inny test (albo ich brak). To najczęściej konsekwencja braku izolacji transakcyjnej albo współdzielonej bazy plikowej.

**Naprawa.** Każdy test tworzy dokładnie te dane, których potrzebuje — najczęściej przez fixture-fabrykę:

```python
# tests/conftest.py (dopisek)
@pytest.fixture
def author_factory(session):
    def _create(name: str = "Lem", titles: list[str] | None = None):
        author = Author(name=name, books=[Book(title=t) for t in (titles or ["Solaris"])])
        session.add(author)
        session.flush()
        return author
    return _create


def test_author_has_books(session, author_factory):
    author = author_factory(titles=["Solaris", "Niezwyciężony"])
    assert len(author.books) == 2
```

**Zapobieganie.** Zakaz `session.commit()` w testach (poza testami, które jawnie sprawdzają commit); identyfikatory nigdy nie są wpisywane w kodzie testu jako literały — test nie powinien wiedzieć, że autor ma `id=1`.

### 12.4 Mockowana sesja

**Objaw.** Testy zielone, produkcja czerwona: `DetachedInstanceError` albo brak zapisu.

**Przyczyna.** `MagicMock(spec=Session)` nie ma logiki. `session.scalars(...)` zwraca kolejnego mocka — a Ty nie sprawdzasz SQL ani błędu mapowania, tylko to, że metoda została wywołana. **Testujesz mock, nie kod.**

**Naprawa.** Testy warstwy danych zawsze na **prawdziwej** sesji (SQLite/PostgreSQL). Mocki zostaw dla integracji zewnętrznych: płatności, e-mail, S3.

**Zapobieganie.** Reguła: jeśli musisz mockować `Session`, to znak, że Twoje repozytorium/serwis ma złą granicę — użyj protokołu i fake-repozytorium w pamięci (`20_repository.md`).

---

## 13. Problemy z API i architekturą

### 13.1 Encja zamiast DTO w odpowiedzi HTTP

**Objaw.** Endpoint działa, ale: (a) zwraca pola, które nie powinny wyjść (hasło, e-mail, wewnętrzne flagi), (b) wybucha `DetachedInstanceError` przy relacjach, (c) generuje N+1 w serializerze, (d) cykliczne relacje powodują problemy przy serializacji.

**Przyczyna.** Encja ORM to **obiekt połączony z bazą**, nie kontrakt API. Kontrakt API powinien być stabilny, podczas gdy model się zmienia.

**Naprawa.** DTO/schemat Pydantic budowany wewnątrz sesji:

```python
# ❌ ŹLE
@app.get("/books/{book_id}")
async def get_book(book_id: int, session: AsyncSession = Depends(get_session)):
    book = await session.get(Book, book_id)
    return book                       # encja? brak eager loadingu, wyciek pól

# ✅ DOBRZE
@app.get("/books/{book_id}", response_model=BookRead)
async def get_book(book_id: int, session: AsyncSession = Depends(get_session)):
    stmt = select(Book).options(selectinload(Book.author)).where(Book.id == book_id)
    book = (await session.scalars(stmt)).one_or_none()
    if book is None:
        raise HTTPException(status_code=404, detail="Book not found")
    return BookRead.model_validate(book, from_attributes=True)
```

### 13.2 Sesja w dwóch wątkach

**Objaw.** Losowe błędy, przemieszane dane, „obiekt należy do innej sesji”, brak deterministycznego kroku reprodukcji.

**Przyczyna.** `Session` **nie jest** bezpieczna wątkowo. Trzyma identity map i transakcję dla konkretnego połączenia, a połączenia DBAPI nie można używać równolegle. Kod wygląda poprawnie do momentu, gdy dwa wątki dotkną tej samej sesji.

**Naprawa.**

```python
# ❌ ŹLE — jedna sesja, wiele wątków
with Session(engine) as session:
    with ThreadPoolExecutor(max_workers=4) as pool:
        pool.map(lambda fid: session.get(Book, fid), book_ids)

# ✅ DOBRZE — jedna sesja na wątek, przekazujemy identyfikatory
def fetch_one(engine: Engine, book_id: int) -> str:
    with Session(engine) as session:
        book = session.get(Book, book_id)
        assert book is not None
        return book.title

with ThreadPoolExecutor(max_workers=4) as pool:
    titles = list(pool.map(lambda fid: fetch_one(engine, fid), book_ids))
```

W async to samo dotyczy zadań: **jedna `AsyncSession` na zadanie**, nigdy wspólna w `asyncio.gather` (`15_asynchronicznosc.md`).

### 13.3 Silnik tworzony per request

**Objaw.** Liczba połączeń w bazie rośnie i nigdy nie spada; pamięć rośnie; po godzinie baza odrzuca połączenia.

**Przyczyna.** `create_engine` tworzy **pulę** połączeń. Silnik w funkcji obsługującej żądanie to nowa pula na każde żądanie — połączenia tworzone, porzucane, nigdy zwalniane w sposób uporządkowany.

**Naprawa.** Silnik tworzony **raz** przy starcie aplikacji, zamykany przy wyjściu (`engine.dispose()` w `lifespan` w FastAPI). Szczegóły w `22_fastapi_integracja.md`.

### 13.4 Logika biznesowa w endpointcie

**Objaw.** Endpoint na 200 linii, z SQL-em, regułami biznesowymi, wysyłką e-maila, logowaniem i konwersją formatów. Nikt nie potrafi tego przetestować bez uruchamiania serwera HTTP.

**Przyczyna.** Brak warstw. HTTP to jeden z wielu sposobów wejścia do aplikacji (obok CLI, kolejki, zadań); reguła biznesowa nie należy do HTTP.

**Naprawa.** Endpoint: parsowanie wejścia, wywołanie przypadku użycia, mapowanie wyniku na odpowiedź. Reguła: `service`/UoW. Dane: repozytorium. Szczegóły i trzy warianty architektury — `19_warstwy_i_data_mapper.md`, `21_unit_of_work.md`.

---

## 14. Antywzorce projektowe

Tabela-katalog antywzorców. Czwarta kolumna to zamiennik, nie tylko „nie rób tak”.

| Antywzorzec | Dlaczego kusi | Co się psuje | Czym zastąpić |
|---|---|---|---|
| **`Session` jako wrapper na repo** — „repozytorium” z metodami `get`, `add`, `execute` bez żadnej logiki | wygląda jak wzorzec, a jest cienką warstwą nad sesją | podwójna abstrakcja: dwa miejsca robiące CRUD, niejasna granica transakcji | albo jawna sesja w serwisie (wariant A), albo pełne repozytorium z zapytaniami domenowymi (`20_repository.md`) |
| **UoW, który nie commit’uje** | „commit należy do wywołującego, elastycznie” | nikt nie commit’uje; dane giną; transakcja wisi w `idle in transaction` | UoW jako kontekst: `with uow: ... uow.commit()` — jeden właściciel granicy (`21_unit_of_work.md`) |
| **`create_all()` zamiast migracji** | działa od ręki | na produkcji zmiana modelu nie zmienia schematu; rozjazd stanu z Alembikiem | Alembic od pierwszego dnia (`16_alembic_migracje.md`); `create_all` tylko w testach/prototypie |
| **`text()` z f-stringiem** | najszybsze na teraz | SQL Injection; brak cache planów zapytań | `select()` albo `text()` z parametrami wiązanymi (`:param`) |
| **Globalna sesja aplikacji** | wygodne, „wszędzie dostępne” | wycieki, konflikty wątkowe, brudna transakcja dla kolejnego żądania | sesja per request / per zadanie, wstrzykiwana zależnością |
| **Globalna sesja** w kodzie async | jw. | jw. + ryzyko dwóch zadań naraz na tym samym połączeniu | `async_sessionmaker` + sesja per zadanie |
| **`echo=True` na produkcji** | łatwe debugowanie | logi puchną, dane osobowe w logach, spadek wydajności | `echo=False` + kontrolowany logger `sqlalchemy.engine` na poziomie INFO/WARN |
| **Model ORM jako model domenowy w złożonym projekcie** | mniej klas | logika biznesowa uwikłana w cykl sesji; trudne testy | mapowanie na modele domenowe + `select(col...)`/DTO (`19_warstwy_i_data_mapper.md`) |
| **Leniwe ładowanie „bo tak”** | mniej kodu | N+1, nieprzewidywalna liczba zapytań | jawne strategie ładowania w miejscu użycia; `lazy="raise"` w testach |
| **`is_modified`/`@validates` jako jedyna walidacja** | wygodne | nie działa przy bulk `update()` i `session.execute(insert(...))` | walidacja wielopoziomowa: schemat wejścia (Pydantic) → model → ograniczenia w bazie |
| **Zapytania zwracające `Select` na zewnątrz repozytorium** | „elastyczność” | przeciek abstrakcji: SQL wychodzi do warstwy HTTP | metody repozytorium z konkretnym znaczeniem biznesowym |
| **Model z 40 kolumnami ładowany zawsze w całości** | prościej | niepotrzebny transfer, wolne listy | `load_only()`, projekcje, `deferred` |
| **Trzymanie encji w cache aplikacji** | pozornie szybko | dane nieaktualne, obiekty detached, wyciek |

### Czarny charakter, który nie jest antywzorcem: singleton `Engine`

Czasem słyszysz: „globalna zmienna to antywzorzec”. W przypadku `Engine` jest odwrotnie — **`Engine` powinien być jeden na proces** (i to jest dobra praktyka):

- `Engine` zawiera **pulę połączeń**. Pula jest zasobem współdzielonym — jeśli tworzysz silnik per żądanie, tworzysz nową pulę i tracisz cały sens puli.
- `Engine` jest bezpieczny wątkowo i bezpieczny dla wielu zadań async (sam oddaje i przyjmuje połączenia).
- `Engine` jest **niezmienny po utworzeniu** — nie ma stanu sesji ani transakcji, który mógłby wyciekać.

Antywzorcem jest **globalna sesja**, nie globalny silnik. Sesja to notatnik jednej pracy; silnik to rozdzielnia prądu dla całego budynku.

> 💡 **Analogia — silnik vs sesja**
> `Engine` to rozdzielnia prądu w budynku: jedna, z określoną liczbą gniazdek (pula), dostępna dla wszystkich mieszkańców. `Session` to przedłużacz, który wziąłeś do konkretnej pracy w konkretnym pokoju. Jeden przedłużacz dla całego budynku = ktoś potknie się o kabel. Jedna rozdzielnia dla każdego pokoju z osobna = absurdalny koszt i więcej gniazdek niż prądu.

---

## 15. FAQ — dwadzieścia pytań

**1. Czy ORM zastępuje SQL?**
Nie. ORM **generuje** SQL i mapuje wyniki na obiekty. Znajomość SQL pozostaje obowiązkowa: musisz wiedzieć, jakie zapytanie powstanie (używaj `str(stmt)` i logu), czy istnieje indeks i jak wygląda plan. ORM oszczędza pisanie powtarzalnego kodu, nie usuwa bazy spod spodu.

**2. Czy muszę umieć SQL, żeby używać SQLAlchemy?**
Na start nie — moduły 01–05 tłumaczą SQL od zera i pokazują SQL generowany przez ORM. W pracy tak: bez SQL nie odróżnisz zapytania wydajnego od katastrofalnego.

**3. Czy nazwy tabel i kolumn są wrażliwe na wielkość liter?**
Zależy od bazy i to źródło przenośnych błędów. SQLite traktuje nazwy identyfikatorów bez rozróżniania wielkości liter (dla ASCII). PostgreSQL **zwija niecytowane identyfikatory do małych liter**: `Book` staje się `book`, a `"Book"` (w cudzysłowie) zostaje `Book`. Dlatego mieszanie `"Book"` i `Book` w tym samym projekcie produkuje na PostgreSQL dwóch różnych kandydatów na tę samą tabelę. **Reguła: nazwy tabel i kolumn zawsze małymi literami, z podkreśleniami** (`book`, `author_id`, `created_at`).

**4. Jak zrobić `UPDATE` bez wcześniejszego `SELECT`?**
Wykonaj `update()` przez sesję — zamiast ładować encję i modyfikować pole:

```python
from sqlalchemy import update

session.execute(
    update(Book).where(Book.title == "Solaris").values(title="Solaris (wyd. 2)")
)
session.commit()
```

Uwaga: encje, które **już są** w sesji, nie zobaczą tej zmiany automatycznie — stąd parametr `synchronize_session` (szczegóły w `10_zapytania_orm.md`).

**5. Jak szybko wyczyścić całą tabelę?**
PostgreSQL: `session.execute(text("TRUNCATE TABLE book RESTART IDENTITY CASCADE"))`. SQLite: `DELETE FROM book` (SQLite nie ma `TRUNCATE`); w SQLite odpowiednikiem szybszym od `DELETE` jest `DELETE FROM book` w transakcji lub `DROP TABLE` + `create`. W aplikacji rozważ „miękkie usuwanie” (`deleted_at`) zamiast fizycznego kasowania.

**6. Jak zresetować licznik `autoincrement`?**
PostgreSQL: `ALTER SEQUENCE book_id_seq RESTART WITH 1` (albo razem z `TRUNCATE ... RESTART IDENTITY`). SQLite: usuń i odtwórz tabelę; dla tabeli z `AUTOINCREMENT` wartość w `sqlite_sequence` można wyzerować. Nie rób tego na produkcji bez powodu — odwołania w innych tabelach mogą wskazywać na stare identyfikatory.

**7. Czy używać `Query` w SQLAlchemy 2.0?**
`Query` istnieje w 2.0 jako **spuścizna (legacy)**, ale nowe API to `select()` + `Session.execute()`/`scalars()`. Nie ucz się `Query` ani nie pisz nowego kodu w tym stylu. Jeśli masz stary kod — mapa migracji jest w dodatku `A4_migracja_i_nowosci.md`.

**8. Co z `bulk_save_objects()`?**
To metoda z kategorii „legacy bulk methods”. W 2.0 preferuj:

```python
# wstawianie wielu wierszy bez tworzenia obiektów ORM
session.execute(insert(Book), [{"title": "A", "author_id": 1}, {"title": "B", "author_id": 1}])
```

albo `session.add_all([...])` + `commit()`, jeśli chcesz mieć obiekty i cykl życia. Wybór jest kompromisem: funkcje bulk są szybsze, ale omijają identity map, eventy i `@validates`.

**9. Ile zapytań wysyła mój kod?**
Tyle, ile wyśle baza — nie zgaduj, policz. Licznik `event.listen(engine, "before_cursor_execute", ...)` z sekcji 5 to najlepsze kilka linii kodu, jakie napiszesz w tym tygodniu.

**10. Dlaczego `session.execute("SELECT 1")` rzuca wyjątek?**
W 2.0 nie wolno przekazywać surowych napisów. Potrzebujesz `text()`:

```python
from sqlalchemy import text

session.execute(text("SELECT 1"))
```

Dostaniesz `ObjectNotExecutableError: Not an executable object: 'SELECT 1'`. To celowe: napis nie ma parametrów wiązanych, więc nie ma żadnej ochrony przed wstrzyknięciem.

**11. Czy `engine` powinien być globalny?**
Tak — jeden na proces (patrz sekcja 14). Globalna powinna być **sesja**? Nigdy.

**12. Jak zmienić bazę bez zmiany kodu?**
Przez URL czytany ze zmiennej środowiskowej:

```python
import os

from sqlalchemy import create_engine

engine = create_engine(os.environ["DATABASE_URL"])
```

W testach podstawiasz inną wartość; w aplikacji czytasz z sekretów.

**13. Czym różni się `flush()` od `commit()`?**
`flush()` wysyła nagromadzone `INSERT`/`UPDATE`/`DELETE` do bazy **w ramach bieżącej transakcji** — dane nie są jeszcze widoczne dla innych połączeń i można je wycofać. `commit()` kończy transakcję, czyniąc zmiany trwałymi, i (domyślnie) przedawnia obiekty. `flush` dzieje się też automatycznie przed każdym zapytaniem (autoflush) — dlatego `id` nowego obiektu potrafi być dostępne przed commitem.

**14. Czy `create_all()` to zła praktyka?**
W aplikacji produkcyjnej — tak, bo nie umie `ALTER`. W testach i w prototypach — jak najbardziej w porządku i szybkie.

**15. Dlaczego `datetime` gubi strefę czasową na SQLite?**
Bo SQLite nie ma natywnego typu danych dla daty ze strefą — SQLAlchemy zapisuje ISO-8601 jako tekst i odczytuje z powrotem bez `tzinfo`. PostgreSQL z `TIMESTAMPTZ` zachowuje strefę. Konwencja bezpieczna dla obu: zawsze zapisuj UTC i normalizuj przy odczycie.

**16. Jak pobrać tylko część dużej kolekcji relacji?**
Nie rób tego przez relację w modelu — zrób osobne zapytanie z `limit`:

```python
stmt = (
    select(Book)
    .where(Book.author_id == author.id)
    .order_by(Book.id)
    .limit(5)
)
```

Relacja to cały zbiór powiązanych wierszy; „top 5” to zapytanie, nie relacja.

**17. Czy mogę używać dwóch sesji jednocześnie?**
Tak — każda jest niezależna, ma własną transakcję i identity map. **Ale nie przenoś obiektów między sesjami** (użyj `merge` albo identyfikatora) i nie mieszaj ich w jednej transakcji logicznej bez świadomej decyzji o dwóch połączeniach.

**18. Jak policzyć rekordy bez pobierania ich?**
```python
from sqlalchemy import func, select

total = session.scalar(select(func.count()).select_from(Book))
```
Albo, jeśli masz już zapytanie z filtrami: `select(func.count()).select_from(stmt.subquery())`.

**19. Skąd wziąć `id` nowego obiektu przed `commit()`?**
Po `flush()` baza nadaje klucz i SQLAlchemy wypełnia nim atrybut. To nie jest „magia” — to mechanizm „wstaw i wróć”: SQLAlchemy albo używa zwróconego klucza (`RETURNING`), albo `lastrowid` sterownika.

**20. Jak logować SQL na produkcji, ale nie zalewać logów?**
Nie używaj `echo=True`. Skonfiguruj logger i poziom:

```python
import logging

logging.getLogger("sqlalchemy.engine").setLevel(logging.WARNING)   # tylko problemy
# logging.getLogger("sqlalchemy.engine").setLevel(logging.INFO)    # SQL w logach
```

Osobno rozważ logger `sqlalchemy.engine.Engine` z własnym handlerem i filtrem (np. logowanie tylko zapytań dłuższych niż 200 ms przez `before_cursor_execute` + pomiar czasu). Nigdy nie loguj parametrów z danymi osobowymi.

---

## 16. Checklist przeglądu kodu (30 punktów)

Do użycia w `code review` i przy ocenie projektu końcowego. Punkty 1–10 to twarde błędy (blokują merge), 11–22 to jakość, 23–30 to wydajność i higiena.

**Poprawność (blokujące)**

1. Czy każda operacja zapisu kończy się dokładnie jednym `commit()` na ścieżce wykonania?
2. Czy po każdym złapanym `IntegrityError` (i każdym błędzie w trakcie `flush`) następuje `session.rollback()`?
3. Czy `Session`/`AsyncSession` powstaje w miejscu użycia i jest zamykana (`with`, `finally`, `Depends` z `yield`)?
4. Czy sesja nie jest współdzielona między wątkami ani między zadaniami `asyncio`?
5. Czy `Engine` jest tworzony raz na proces (a nie per funkcja/żądanie)?
6. Czy w kodzie nie ma SQL składanego z f-stringów ani `%`? Wszystkie wartości idą przez parametry wiązane?
7. Czy `expire_on_commit` jest ustawione świadomie i czy kod nie polega na dostępie do przedawnionych obiektów poza sesją?
8. Czy encje ORM nie są zwracane z warstwy HTTP/CLI na zewnątrz (jest DTO/schemat/dict)?
9. Czy `create_all()` nie jest wołany w ścieżce startu produkcji?
10. Czy migracje mają napisane i sprawdzone `downgrade`?

**Jakość i czytelność**

11. Czy w kodzie nie ma `session.query(...)` ani `Query` z 1.x?
12. Czy zapytania używają `select()` + `Session.execute()`/`scalars()` w stylu 2.0?
13. Czy każda relacja ma `back_populates` po obu stronach?
14. Czy kaskady (`cascade`, `ondelete`, `passive_deletes`) są ustawione świadomie, a nie „przez przypadek”?
15. Czy `relationship()` nie jest ładowana w pętli bez jawnej strategii?
16. Czy kolekcje są ładowane przez `selectinload` (a `joinedload` na kolekcjach ma `.unique()`)?
17. Czy w projekcie async wszystkie relacje mają `lazy="raise"` albo jawne strategie ładowania?
18. Czy zastosowano `naming_convention` w `MetaData`?
19. Czy typy kolumn są adekwatne (pieniądze → `Numeric`, nie `Float`; `NOT NULL` jawnie w `Mapped[...]` i w kolumnie)?
20. Czy `datetime` jest konsekwentnie w UTC (zapis) i normalizowany (odczyt)?
21. Czy mutowalne struktury (JSON, kolekcje) używają `Mutable*` albo całościowego przypisania?
22. Czy granice transakcji są w warstwie aplikacji, a nie w repozytorium i nie w kontrolerze HTTP?

**Wydajność i higiena**

23. Czy każde zapytanie zwracające listę ma limit?
24. Czy listy bez paginacji keyset są uzasadnione (mały zbiór)?
25. Czy raporty używają projekcji kolumn (`select(Book.id, Book.title)`) zamiast pełnych encji?
26. Czy w testach istnieje asercja na liczbę zapytań dla ścieżek krytycznych?
27. Czy indeksy odpowiadają realnym predykatom zapytań (w tym indeksy funkcyjne i złożone z właściwą kolejnością kolumn)?
28. Czy pula połączeń jest skonfigurowana (`pool_size`, `max_overflow`, `pool_recycle`, `pool_pre_ping`) i czy `pool_timeout` jest świadomie ustawiony?
29. Czy `echo`/`echo_pool` są wyłączone na produkcji, a SQL w logach jest ograniczony?
30. Czy dane wrażliwe nie trafiają do logów zapytań ani do komunikatów błędów zwracanych do klienta?

---

## 17. Narzędzia diagnostyczne

Sześć narzędzi, których użyjesz w 95% przypadków.

### 17.1 `echo` i `logging`

```python
# Szybko, w skrypcie: wszystko widać, ale wszystko też ląduje na stdout
engine = create_engine("sqlite:///:memory:", echo=True, echo_pool="debug")

# W aplikacji: precyzyjnie, per logger
import logging

logging.basicConfig()
logging.getLogger("sqlalchemy.engine").setLevel(logging.INFO)
```

`echo_pool="debug"` pokazuje wypożyczenia i zwroty połączeń — to pierwsze narzędzie, gdy podejrzewasz wyciek połączeń.

### 17.2 Licznik zapytań przez `sqlalchemy.event`

```python
# examples/24_query_counter.py — pełna implementacja w sekcji 5
with count_queries(engine) as queries:
    ...
print(len(queries))
```

Wariant „wolne zapytania”:

```python
import time

from sqlalchemy import event


@event.listens_for(engine, "before_cursor_execute")
def _before(conn, cursor, statement, parameters, context, executemany) -> None:
    conn.info.setdefault("query_start_time", []).append(time.perf_counter())


@event.listens_for(engine, "after_cursor_execute")
def _after(conn, cursor, statement, parameters, context, executemany) -> None:
    total = time.perf_counter() - conn.info["query_start_time"].pop(-1)
    if total > 0.2:
        print(f"WOLNE ({total:.3f}s): {statement[:120]}")
```

### 17.3 `EXPLAIN` / `EXPLAIN ANALYZE`

```python
from sqlalchemy import text

stmt = select(Book).where(Book.title == "Solaris")
print(stmt)                                                     # co wyślemy
sql = str(stmt.compile(engine, compile_kwargs={"literal_binds": True}))
rows = session.execute(text(f"EXPLAIN ANALYZE {sql}")).all()     # UWAGA: dev only
for row in rows:
    print(row[0])
```

W `psql`/`pgcli` po prostu `EXPLAIN ANALYZE` wklejone zapytanie. Czego szukać: `Seq Scan` na dużej tabeli, różnica między szacowaną a rzeczywistą liczbą wierszy (dowód nieaktualnych statystyk), `Nested Loop` na dużych zbiorach.

### 17.4 `pg_stat_statements` — ranking najdroższych zapytań

```sql
SELECT calls, mean_exec_time, total_exec_time, left(query, 100) AS query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

To daje odpowiedź na pytanie „co naprawdę obciąża bazę”, a nie „co mi się wydaje wolne”.

### 17.5 Klienty bazy: `sqlite3`, `pgcli`, `DBeaver`

```bash
sqlite3 app.db ".schema book"          # odczyt schematu
sqlite3 app.db "PRAGMA foreign_keys;"  # czy włączone (domyślnie 0!)
pgcli postgresql://user@localhost/db   # wygodny klient PostgreSQL
```

### 17.6 Debugger

`breakpoint()` w kodzie to wciąż niedoceniane narzędzie: w miejscu wyjątku sprawdzisz `session.new`, `session.dirty`, `inspect(obj).persistent`, `engine.pool.status()`. Sześć sekund w debuggerze oszczędza godzinę czytania dokumentacji.

---

## Jak NIE diagnozować

To sekcja celowo odwrotna do zwykłych „pułapek”. Jeśli łapiesz się na którymś z tych zachowań — zatrzymaj się.

1. **Zgadywanie.** „Pewnie trzeba dodać `expire_on_commit=False`” — bez przeczytania pełnego komunikatu i bez sprawdzenia, w którym miejscu wybucha. Komunikat zawiera link `sqlalche.me/e/20/...` prowadzący do opisanej przyczyny; to nie jest ozdoba.
2. **Zmiana wielu rzeczy naraz.** Dodanie `selectinload` + `expire_on_commit=False` + `lazy="joined"` w jednym commicie sprawia, że nie wiesz, co pomogło, a co wprowadziło nowy problem. **Jedna zmiana → jeden pomiar.**
3. **Gaszenie objawów.** `try: ... except Exception: pass` wokół zapytań, `expire_on_commit=False` wsadzone globalnie, `pool_size=100` jako odpowiedź na wyciek połączeń. Objawy znikają, choroba zostaje. `pool_size=100` przy 5-procesowej bazie nie leczy wycieku — tylko przesuwa moment awarii.
4. **Optymalizacja bez pomiaru.** „Przepiszę na `joinedload`, będzie szybciej” — bez licznika zapytań i bez `EXPLAIN`. Często jest wolniej: dwie kolekcje w `joinedload` dają kartezjański wybuch.
5. **Testowanie na pustej bazie.** Plan zapytania na 100 wierszach to nie plan na 5 mln. Statystyki PostgreSQL zmieniają plan, gdy tabela rośnie.
6. **Diagnozowanie na produkcji przez `echo=True`.** Ogromne logi, dane osobowe w plikach, spadek wydajności. Diagnozuj na kopii; na produkcji tylko metryki i logi wolnych zapytań bez parametrów.
7. **Zmiana modelu bez migracji**, żeby „szybko sprawdzić”. To najkrótsza droga do rozjazdu schematu między środowiskami.
8. **Obwinianie SQLAlchemy.** Zdecydowana większość „wolnych zapytań ORM” jest wolna z tych samych powodów, dla których byłoby wolne ręczne SQL: brak indeksu, brak limitu, zły wzorzec dostępu. ORM nie chroni przed złym zapytaniem — tylko przed ręcznym mapowaniem wierszy.

> 🧠 **Dlaczego tak jest — trzy pytania zamiast dziesięciu hipotez**
> Zanim cokolwiek zmienisz, odpowiedz na trzy pytania: (1) **Ile zapytań** poszło i jakich? (2) **Jaki był plan** najdroższego z nich? (3) **Ile danych** przez nie przepłynęło? Odpowiedzi na te trzy pytania rozwiązują około 90% problemów wydajnościowych i połowę problemów z poprawnością.

---

## Repozytorium bugów — sześć skryptów do naprawy

Celowe, zepsute skrypty edukacyjne. Każda funkcja **reprodukuje dokładnie jeden** opisany wyżej problem i zawiera komentarz z informacją, jak go naprawić. Twoje zadanie: uruchomić, zrozumieć, naprawić.

Wymagania: `pip install "SQLAlchemy>=2.0" aiosqlite` (SQLite w wersji standardowej Pythona wystarczy).

```python
# examples/24_bug_repository.py
"""Repozytorium sześciu zepsutych skryptów — ćwiczenie 'znajdź i napraw'.

Uruchamiaj poszczególne punkty:
    python examples/24_bug_repository.py 1
    python examples/24_bug_repository.py 2
    ...
"""

from __future__ import annotations

import asyncio
import sys
from concurrent.futures import ThreadPoolExecutor

from sqlalchemy import ForeignKey, JSON, String, create_engine, event, select
from sqlalchemy.engine import Engine
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from sqlalchemy.ext.mutable import MutableDict
from sqlalchemy.orm import (
    DeclarativeBase,
    Mapped,
    Session,
    joinedload,
    mapped_column,
    relationship,
    selectinload,
)


class Base(DeclarativeBase):
    pass


class Author(Base):
    __tablename__ = "author"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    books: Mapped[list["Book"]] = relationship(back_populates="author")


class Book(Base):
    __tablename__ = "book"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    author_id: Mapped[int] = mapped_column(ForeignKey("author.id"))
    meta: Mapped[dict] = mapped_column(JSON, default=dict)      # BUG 6: brak MutableDict
    author: Mapped["Author"] = relationship(back_populates="books")


def make_engine() -> Engine:
    engine = create_engine("sqlite:///:memory:")
    Base.metadata.create_all(engine)
    with Session(engine) as session:
        session.add(
            Author(
                name="Lem",
                books=[Book(title="Solaris"), Book(title="Niezwyciężony"), Book(title="Golem XIV")],
            )
        )
        session.commit()
    return engine


# ---------------------------------------------------------------------------
# BUG 1 — dane nie są zapisywane
# ---------------------------------------------------------------------------
def bug_1_not_saved(engine: Engine) -> None:
    """OBJAW: brak błędu, ale po wyjściu z bloku rekordów nie ma w bazie.
    PRZYCZYNA: brak session.commit() — zamknięcie sesji robi rollback.
    NAPRAWA: dodaj `session.commit()` albo użyj `with Session(engine) as s, s.begin():`.
    """
    with Session(engine) as session:
        session.add(Author(name="Nowy"))
        # ← TU BRAKUJE COMMIT
    with Session(engine) as session:
        count = session.scalar(select(__import__("sqlalchemy").func.count()).select_from(Author))
        print(f"[BUG 1] autorów w bazie: {count}  (oczekiwane 2, a nie 1)")


# ---------------------------------------------------------------------------
# BUG 2 — DetachedInstanceError
# ---------------------------------------------------------------------------
def bug_2_detached(engine: Engine) -> None:
    """OBJAW: DetachedInstanceError przy dostępie do relacji.
    PRZYCZYNA: dostęp do leniwej relacji po zamknięciu sesji.
    NAPRAWA: `selectinload(Book.author)` albo konwersja na DTO wewnątrz sesji.
    """
    with Session(engine) as session:
        book = session.scalars(select(Book).limit(1)).one()
    print(book.title)            # OK — kolumna wczytana
    print(book.author.name)      # ← wybuchnie: DetachedInstanceError


# ---------------------------------------------------------------------------
# BUG 3 — N+1
# ---------------------------------------------------------------------------
def bug_3_n_plus_one(engine: Engine) -> None:
    """OBJAW: 1 + N zapytań, liniowy wzrost czasu odpowiedzi.
    PRZYCZYNA: leniwe ładowanie kolekcji w pętli.
    NAPRAWA: `.options(selectinload(Author.books))` przed pętlą.
    """
    statements: list[str] = []

    def _before(conn, cursor, statement, parameters, context, executemany) -> None:
        statements.append(statement)

    event.listen(engine, "before_cursor_execute", _before)
    try:
        with Session(engine) as session:
            authors = session.scalars(select(Author)).all()
            for author in authors:                     # ← leniwe ładowanie w pętli
                _ = len(author.books)
    finally:
        event.remove(engine, "before_cursor_execute", _before)
    print(f"[BUG 3] zapytań: {len(statements)}  (oczekiwane 2, przy 1 autorze wystarczy 1)")


# ---------------------------------------------------------------------------
# BUG 4 — duplikaty / InvalidRequestError
# ---------------------------------------------------------------------------
def bug_4_duplicates(engine: Engine) -> None:
    """OBJAW: InvalidRequestError 'The unique() method must be invoked on this Result'.
    PRZYCZYNA: joinedload przeciwko kolekcji bez unique().
    NAPRAWA: `.unique()` po scalars() ALBO selectinload zamiast joinedload.
    """
    with Session(engine) as session:
        stmt = select(Author).options(joinedload(Author.books))
        authors = session.scalars(stmt).all()      # ← wybuchnie
    print(f"[BUG 4] autorów: {len(authors)}")


# ---------------------------------------------------------------------------
# BUG 5 — SQLite in-memory w innym wątku
# ---------------------------------------------------------------------------
def _read_count_in_thread(engine: Engine) -> int:
    from sqlalchemy import func

    with Session(engine) as session:
        return session.scalar(select(func.count()).select_from(Book)) or 0


def bug_5_memory_db_thread(engine: Engine) -> None:
    """OBJAW: 'no such table: book' mimo wcześniejszego create_all().
    PRZYCZYNA: SQLite :memory: + brak StaticPool → wątek potomny dostaje PUSTĄ bazę.
    NAPRAWA: create_engine(..., poolclass=StaticPool, connect_args={'check_same_thread': False}).
    """
    with ThreadPoolExecutor(max_workers=1) as pool:
        count = pool.submit(_read_count_in_thread, engine).result()
    print(f"[BUG 5] książek widzianych w wątku: {count}")


# ---------------------------------------------------------------------------
# BUG 6 — mutowalny JSON bez MutableDict
# ---------------------------------------------------------------------------
def bug_6_json_not_modified(engine: Engine) -> None:
    """OBJAW: zmiana klucza w JSON nie zapisuje się; brak śladu w logu SQL.
    PRZYCZYNA: SQLAlchemy nie śledzi mutacji wewnątrz struktur.
    NAPRAWA: MutableDict.as_mutable(JSON) albo całościowe przypisanie
             `book.meta = {**book.meta, 'views': 7}`.
    """
    with Session(engine) as session:
        book = session.scalars(select(Book).limit(1)).one()
        book.meta["views"] = 7                     # ← SQLAlchemy tego nie widzi
        session.commit()
    with Session(engine) as session:
        book = session.scalars(select(Book).limit(1)).one()
        print(f"[BUG 6] meta po commicie: {book.meta}  (oczekiwane {{'views': 7}})")


# ---------------------------------------------------------------------------
# BONUS — wersja async buga 2 (MissingGreenlet), gdy chcesz więcej
# ---------------------------------------------------------------------------
async def bonus_missing_greenlet() -> None:
    """OBJAW: MissingGreenlet przy book.author w kodzie async.
    NAPRAWA: selectinload + expire_on_commit=False (+ lazy='raise' jako bezpiecznik).
    """
    engine = create_async_engine("sqlite+aiosqlite:///:memory:")
    factory = async_sessionmaker(engine, expire_on_commit=False)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    async with factory() as session:
        session.add(Author(name="Lem", books=[Book(title="Solaris")]))
        await session.commit()
    async with factory() as session:
        book = (await session.scalars(select(Book).limit(1))).one()
        print(book.author.name)      # ← MissingGreenlet
    await engine.dispose()


DISPATCH = {
    "1": bug_1_not_saved,
    "2": bug_2_detached,
    "3": bug_3_n_plus_one,
    "4": bug_4_duplicates,
    "5": bug_5_memory_db_thread,
    "6": bug_6_json_not_modified,
}


def main() -> None:
    which = sys.argv[1] if len(sys.argv) > 1 else "1"
    if which == "async":
        asyncio.run(bonus_missing_greenlet())
        return
    engine = make_engine()
    DISPATCH[which](engine)


if __name__ == "__main__":
    main()
```

> 🧪 **Ćwiczenie** — uruchom wszystkie sześć punktów po kolei (`python examples/24_bug_repository.py 1` … `6`), zapisz komunikat każdego błędu, a następnie napraw je **wszystkie jednocześnie** w kopii pliku `24_bug_repository_fixed.py`. Warunek: żaden z punktów nie może wyświetlić komunikatu wyjątku, a punkty 1, 3 i 6 muszą wypisać wartości zgodne z „oczekiwane”.

---

## Podsumowanie

1. **Diagnoza przed leczeniem.** Każdy problem w tym module ma objaw, mechanizm i naprawę. Zaczynaj od objawu i mechanizmu, nie od zmiany kodu.
2. **`DetachedInstanceError` to pytanie „gdzie jest sesja”**, nie „co zepsułem w modelu”. Encja nie opuszcza granicy sesji — na zewnątrz wychodzą DTO.
3. **`MissingGreenlet` w async rozwiązujesz trzema krokami:** `expire_on_commit=False`, jawne strategie ładowania (`selectinload`), zakaz odwołań do relacji w `__repr__`.
4. **„Nic się nie zapisało” ma pięć typowych przyczyn** i wszystkie sprawdzisz trzema komendami: `session.new`, `session.dirty`, `session.is_modified(obj)`.
5. **Duplikaty to znak JOIN-a po kolekcji albo `joinedload` bez `unique()`.** Dla kolekcji domyślnie wybieraj `selectinload`.
6. **N+1 diagnozujesz licznikiem zapytań.** Jeśli nie masz liczby, nie masz diagnozy. Optymalna liczba zapytań dla listy z jedną relacją to 2, nie 51.
7. **Po każdym błędzie w trakcie `flush` należy się `session.rollback()`.** Bez tego dostaniesz `PendingRollbackError` i sesję, która nie chce już nic robić.
8. **`IntegrityError` to komunikat bazy, nie SQLAlchemy.** Tłumacz go na wyjątki domenowe w jednym miejscu i nie pozwól mu dolecieć do kontrolera HTTP.
9. **Migracje: przeglądaj wygenerowany plik, pisz `downgrade`, włącz `render_as_batch` dla SQLite** i nigdy nie łącz `create_all` z Alembikiem w jednym środowisku.
10. **Globalny `Engine` jest dobry, globalna `Session` jest zła.** To nie sprzeczność — to różnica między rozdzielnią prądu a notatnikiem.
11. **Wydajność: ogranicz liczbę rund do bazy, ilość przesyłanych danych i czas trwania transakcji.** Trzy dźwignie, które odpowiadają za większość wyników.

---

## Ćwiczenia

### Ćwiczenie 1 — Napraw sześć bugów

Uruchom `examples/24_bug_repository.py` dla punktów 1–6, zanotuj komunikaty i napraw wszystkie sześć w osobnym pliku. Twoje rozwiązanie musi spełniać warunki:

- punkt 1 wypisuje 2 (a nie 1),
- punkt 2 kończy się bez wyjątku,
- punkt 3 raportuje 2 zapytania dla 1 autora i nie więcej niż 2 dla N autorów,
- punkt 4 kończy się bez wyjątku, a liczba autorów wynosi 1,
- punkt 5 działa w wątku potomnym (bez „no such table”),
- punkt 6 wypisuje `{'views': 7}`.

### Ćwiczenie 2 — Własna checklista

Na podstawie doświadczeń z projektu końcowego (`23_projekt_koncowy.md`) dopisz do checklisty z sekcji 16 **pięć własnych punktów**. Każdy punkt musi mieć formę pytania, na które da się odpowiedzieć „tak/nie”, oraz krótkie uzasadnienie, dlaczego to ma znaczenie w Twoim projekcie. Przykład dobrego punktu: „Czy każdy endpoint listujący ma jawnie ustawioną strategię ładowania relacji, które serializuję?”.

### Ćwiczenie 3 — Symulacja awarii produkcyjnej

Przygotuj skrypt, który doprowadza do:

1. wycieku połączeń (polegającego na tym, że `pool.status()` pokazuje rosnącą liczbę `Checked out connections`),
2. `PendingRollbackError` po nieudanym `flush`,
3. `idle in transaction` (na PostgreSQL; na SQLite pokaż, czym różni się model blokowania).

Do każdej awarii napisz (a) minimalny kod reprodukujący, (b) sposób wykrycia (który licznik, które zapytanie do `pg_stat_activity`, który fragment logu), (c) naprawę.

### Rozwiązania

**Ćwiczenie 1 — klucz napraw (pełny kod jest dopuszczalny, ale spróbuj najpierw sam):**

1. Dodaj `session.commit()` na końcu bloku (albo użyj `with Session(engine) as session, session.begin():`).
2. Zamień `select(Book).limit(1)` na `select(Book).options(selectinload(Book.author)).limit(1)`; alternatywnie zbuduj `BookDTO` wewnątrz sesji.
3. Dodaj `.options(selectinload(Author.books))` do zapytania o autorów. Liczba zapytań spadnie z $1+N$ do 2.
4. Dodaj `.unique()` przed `.all()`: `session.scalars(stmt).unique().all()`. Albo zamień `joinedload` na `selectinload`.
5. Wydziel silnik do fixture z `poolclass=StaticPool` i `connect_args={"check_same_thread": False}`:
   ```python
   from sqlalchemy import StaticPool

   engine = create_engine(
       "sqlite:///:memory:",
       connect_args={"check_same_thread": False},
       poolclass=StaticPool,
   )
   ```
6. W modelu zamień `JSON` na `MutableDict.as_mutable(JSON)`; alternatywnie w kodzie `book.meta = {**book.meta, "views": 7}`; trzeci wariant — `flag_modified(book, "meta")`.

Typowe błędy w tym ćwiczeniu: naprawa punktu 1 przez `expire_on_commit=False` (nie pomoże — to nie problem przedawnienia, a brak commitu), naprawa punktu 4 przez `distinct()` (nie pomoże — wyjątek leci z warstwy wyników, nie z SQL), naprawa punktu 5 przez `check_same_thread=False` **bez** `StaticPool` (wtedy każdy wątek dostaje nowe połączenie, czyli nową, pustą bazę — objaw zostaje).

**Ćwiczenie 2 — kryteria oceny własnej checklisty:** punkty muszą być weryfikowalne (tak/nie), odnosić się do konkretnego ryzyka z Twojego projektu i nie dublować punktów 1–30 z listy bazowej. Dobre uzupełnienia dotyczą rzeczy specyficznych dla Twojej domeny: „Czy stan magazynowy jest aktualizowany w tej samej transakcji co rezerwacja?”, „Czy każdy raport ma zdefiniowany zakres dat, żeby nie skanować całej historii?”.

**Ćwiczenie 3 — szkic rozwiązania:**

```python
# Wyciek połączeń: zapamiętaj sesje bez close()
leaked: list[Session] = []
for _ in range(20):
    leaked.append(Session(engine))          # ❌ brak close() → połączenia zajęte
# Wykrycie:
print(engine.pool.status())                  # rośnie "Checked out connections"
# Naprawa: `with Session(engine) as session:` albo `session.close()` w finally

# PendingRollbackError — patrz skrypt w sekcji 6
# Wykrycie: komunikat błędu wskazuje na nieudany flush i brak rollbacku
# Naprawa: session.rollback() w except

# idle in transaction (PostgreSQL)
# Wykrycie:
#   SELECT pid, state, now() - xact_start FROM pg_stat_activity
#   WHERE state = 'idle in transaction';
# Naprawa: krótsze transakcje, idle_in_transaction_session_timeout, brak wywołań
#          zewnętrznych API wewnątrz transakcji
```

Na SQLite nie zobaczysz `idle in transaction` w tej samej postaci: SQLite blokuje całą bazę na czas zapisu i nie prowadzi tej statystyki — sama blokada jest objawem (drugi proces dostaje `database is locked`). To dobra ilustracja, dlaczego testowanie modelu współbieżności na SQLite bywa mylące.

---

## Najczęstsze błędy i jak je czytać

Tabela-indeks tego modułu. Jeśli nie wiesz, gdzie szukać — zacznij tutaj.

| Komunikat / objaw | Gdzie szukać | Pierwsza naprawa |
|---|---|---|
| `DetachedInstanceError` | [§1](#1-detachedinstanceerror--obiekt-bez-domu) | eager loading albo DTO wewnątrz sesji |
| `MissingGreenlet` | [§2](#2-missinggreenlet--gdy-async-spotyka-leniwe-io) | `expire_on_commit=False` + `selectinload` + `lazy="raise"` |
| brak zapisu bez błędu | [§3](#3-nic-się-nie-zapisało) | sprawdź `session.new`/`dirty`, dodaj `commit()` |
| `The unique() method must be invoked` | [§4](#4-duplikaty-wyników) | `.unique()` albo `selectinload` |
| wolne listy, powtarzalne zapytania w logu | [§5](#5-n1-czyli-strona-ładuje-się-3-sekundy) | `selectinload`/`joinedload` + licznik zapytań w teście |
| `PendingRollbackError` | [§6](#6-błędy-transakcyjne) | `session.rollback()` w `except` |
| „session is in 'prepared' state” | [§6](#6-błędy-transakcyjne) | `rollback()` i nie używaj dalej tej sesji |
| `IntegrityError: UNIQUE constraint failed` | [§7](#7-integrityerror--jak-czytać-komunikat-bazy) | mapuj na wyjątek domenowy, `rollback` |
| `IntegrityError` + klucz obcy nic nie zgłasza (SQLite) | [§7](#7-integrityerror--jak-czytać-komunikat-bazy) | włącz `PRAGMA foreign_keys=ON` |
| `near "ALTER": syntax error` | [§8](#8-problemy-migracyjne) | `render_as_batch=True` |
| `Multiple head revisions are present` | [§8](#8-problemy-migracyjne) | `alembic merge` |
| `table ... already exists` / brakująca kolumna | [§8](#8-problemy-migracyjne) | wybierz migracje, usuń `create_all` z produkcji |
| `TypeError: can't compare offset-naive and offset-aware` | [§9](#9-problemy-z-typami-danych) | zapisuj UTC, normalizuj przy odczycie |
| `unsupported operand type(s) for +: 'Decimal' and 'float'` | [§9](#9-problemy-z-typami-danych) | konsekwentny `Decimal` w całej ścieżce |
| `invalid input value for enum` | [§9](#9-problemy-z-typami-danych) | `ALTER TYPE ... ADD VALUE` albo tabela słownikowa |
| zmiana w JSON się nie zapisuje | [§9](#9-problemy-z-typami-danych) | `MutableDict` / przypisanie całości |
| `server closed the connection unexpectedly` | [§10](#10-problemy-z-połączeniami) | `pool_pre_ping=True`, `pool_recycle` |
| `QueuePool limit of size ... reached` | [§10](#10-problemy-z-połączeniami) | znajdź wyciek; `with` dla każdej sesji |
| `idle in transaction` w `pg_stat_activity` | [§10](#10-problemy-z-połączeniami) | krótsze transakcje + `idle_in_transaction_session_timeout` |
| zapytanie ignoruje indeks | [§11](#11-problemy-wydajnościowe) | `EXPLAIN` + indeks funkcyjny/trigram |
| strona 5000 ładuje się 4 sekundy | [§11](#11-problemy-wydajnościowe) | keyset pagination zamiast `OFFSET` |
| `no such table` w testach | [§12](#12-problemy-z-testami) | `StaticPool` + `check_same_thread=False` |
| testy zależne od kolejności | [§12](#12-problemy-z-testami) | transakcja zewnętrzna + fabryki danych |
| `DetachedInstanceError` w serializerze HTTP | [§13](#13-problemy-z-api-i-architekturą) | DTO wewnątrz sesji |
| dane przemieszane, brak determinizmu | [§13](#13-problemy-z-api-i-architekturą) | jedna sesja na wątek / na zadanie |

---

## Słowniczek modułu

| Termin (EN) | Polski odpowiednik | Wyjaśnienie |
|---|---|---|
| **detached** | odłączony | obiekt ORM, który nie jest przypięty do sesji; wczytane pola działają, dociąganie relacji nie |
| **expired** | przedawniony | obiekt w sesji, którego dane uznano za nieaktualne; dostęp do pola wywołuje zapytanie |
| **greenlet** | zielony wątek | lekki „wątek” biblioteki `greenlet`, na którym SQLAlchemy buduje most sync↔async |
| **`MissingGreenlet`** | brak zielonego wątku | wyjątek oznaczający IO poza kontekstem `await` w trybie async |
| **idle in transaction** | bezczynna transakcja | stan połączenia w PostgreSQL: transakcja otwarta, ale nic się nie dzieje; blokuje `VACUUM` |
| **N+1** | zapytanie N+1 | wzorzec dostępu: jedno zapytanie plus $N$ zapytań w pętli |
| **connection leak** | wyciek połączeń | wypożyczone połączenie nie wraca do puli; pula się wyczerpuje |
| **`StaticPool`** | pula statyczna | pula z jednym współdzielonym połączeniem; standard dla SQLite `:memory:` w testach |
| **`pool_pre_ping`** | sprawdzenie połączenia przed użyciem | tanie badanie żywotności połączenia wypożyczanego z puli |
| **`pool_recycle`** | recykling połączeń | maksymalny wiek połączenia w puli; starsze są wymieniane |
| **keyset pagination** | paginacja kursorowa | stronicowanie przez `WHERE id > :last ORDER BY id LIMIT n`; stały koszt |
| **`idle` / `Seq Scan`** | skan sekwencyjny | plan zapytania czytający całą tabelę; tu: dowód braku użytecznego indeksu |
| **functional index** | indeks funkcyjny | indeks na wyniku wyrażenia, np. `lower(name)`; zapytanie musi używać tego samego wyrażenia |
| **`render_as_batch`** | tryb wsadowy Alembica | technika migracji dla SQLite: nowa tabela, kopiowanie danych, podmiana |
| **`merge()`** | scalanie | przypisanie odłączonego obiektu do sesji przez kopię znalezioną po kluczu |
| **`MutableDict` / `MutableList`** | słownik/listа śledzona | rozszerzenia typów, które informują SQLAlchemy o mutacjach wewnątrz struktury |
| **`autoflush`** | automatyczny zrzut | wysłanie nagromadzonych zmian przed zapytaniem; w 2.1 działa bezwarunkowo |
| **`lazy="raise"`** | leniwe ładowanie jako błąd | konfiguracja relacji: każde leniwe odwołanie rzuca wyjątek; bezpiecznik przed N+1 |

---

## Dalsze czytanie

- `sqlalchemy` — strona „Errors” z odnośnikami do wszystkich kodów błędów (linki `sqlalche.me/e/20/...` z komunikatów prowadzą tutaj): https://docs.sqlalchemy.org/en/20/errors.html
- Zarządzanie stanem sesji, `DetachedInstanceError`, `expire_on_commit`: https://docs.sqlalchemy.org/en/20/orm/session_state_management.html
- Transakcje sesji, savepointy, stany po błędzie: https://docs.sqlalchemy.org/en/20/orm/session_transaction.html
- Ładowanie relacji i N+1 (`selectinload`, `joinedload`, `raiseload`): https://docs.sqlalchemy.org/en/20/orm/queryguide/relationships.html
- Pula połączeń, `pool_pre_ping`, `pool_recycle`, `StaticPool`: https://docs.sqlalchemy.org/en/20/core/pooling.html
- Rozszerzenie asyncio (`AsyncSession`, `run_sync`, `MissingGreenlet`): https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html
- Dialekt SQLite (ograniczenia `ALTER TABLE`, `PRAGMA foreign_keys`): https://docs.sqlalchemy.org/en/20/dialects/sqlite.html
- Dokumentacja główna 2.0: https://docs.sqlalchemy.org/en/20/
- Dokumentacja główna 2.1 (nowości serii 2.1): https://docs.sqlalchemy.org/en/21/
- Alembic — tryb wsadowy (`render_as_batch`): https://alembic.sqlalchemy.org/en/latest/batch.html
- Alembic — autogenerate, rozgałęzienia, `heads`/`merge`: https://alembic.sqlalchemy.org/en/latest/autogenerate.html
- PostgreSQL — `EXPLAIN`, planowanie zapytań: https://www.postgresql.org/docs/current/using-explain.html
- PostgreSQL — monitorowanie, `pg_stat_activity`, `pg_stat_statements`: https://www.postgresql.org/docs/current/monitoring-stats.html

---

## Co dalej

Znasz już pełny zestaw objawów, mechanizmów i napraw — to ostatni moduł merytoryczny kursu. Kolejne pliki to dodatki: `A1_sciaga.md` (ściąga wzorców i fragmentów kodu, do której będziesz wracać codziennie), a potem `A2_glosariusz.md`, `A3_cwiczenia_rozwiazania.md` z rozwiązaniami wszystkich ćwiczeń kursu, `A4_migracja_i_nowosci.md` (jeśli utrzymujesz kod z 1.x albo chcesz śledzić zmiany w 2.1) oraz `A5_zasoby.md`.

Przejdź do `A1_sciaga.md`.

<!-- koniec modułu 24 -->