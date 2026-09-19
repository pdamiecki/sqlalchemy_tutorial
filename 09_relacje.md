# Moduł 09 — Relacje: jak powiązać tabele w spójny graf

W tym module nauczysz się łączyć tabele w Pythonie tak, aby kod pracował na obiektach, a nie na ręcznie dopasowanych identyfikatorach. Zaczniemy od problemu, który relacje rozwiązują, potem zbudujemy pierwszą relację jeden-do-wielu, a skończymy na kaskadach, tabelach asocjacyjnych z własnymi danymi i strategiach ładowania, które decydują o tym, czy Twoja aplikacja działa szybko, czy wysyła sto zapytań tam, gdzie wystarczy jedno.

> **Poziom:** 🟠 zaawansowany · **Czas:** ~180 min
> **Wymagania wstępne:** [`07_modele_deklaratywne.md`](07_modele_deklaratywne.md), [`08_sesja_cykl_zycia.md`](08_sesja_cykl_zycia.md)
> **Czego dotyczy ten plik:** definiowania relacji w modelach deklaratywnych (`relationship()`), kardynalności 1:N, N:1, 1:1 i N:M, kaskad, usuwania danych po stronie bazy i Pythona, oraz strategii ładowania relacji.

## Spis treści

- [1. Problem: dwie tabele, jedna historia](#1-problem-dwie-tabele-jedna-historia)
- [2. Pierwsza relacja: jeden-do-wielu](#2-pierwsza-relacja-jeden-do-wielu)
- [3. Parametry relationship() — pełny przegląd](#3-parametry-relationship--pełny-przegląd)
- [4. Jeden-do-jednego](#4-jeden-do-jednego)
- [5. Wiele-do-wielu](#5-wiele-do-wielu)
- [6. Association Object — gdy tabela pośrednia ma własne dane](#6-association-object--gdy-tabela-pośrednia-ma-własne-dane)
- [7. Relacje samo-referencyjne](#7-relacje-samo-referencyjne)
- [8. Kaskady — kto za kim sprząta](#8-kaskady--kto-za-kim-sprząta)
- [9. Kto usuwa dane: Python czy baza?](#9-kto-usuwa-dane-python-czy-baza)
- [10. Dwie relacje do tej samej tabeli](#10-dwie-relacje-do-tej-samej-tabeli)
- [11. viewonly i synonym](#11-viewonly-i-synonym)
- [12. Strategie ładowania — lazy=...](#12-strategie-ładowania--lazy)
- [13. Modyfikowanie kolekcji i skąd bierze się N+1](#13-modyfikowanie-kolekcji-i-skąd-bierze-się-n1)
- [14. Pełny przykład: biblioteka](#14-pełny-przykład-biblioteka)
- [Podsumowanie](#podsumowanie)
- [Ćwiczenia](#ćwiczenia)
- [Najczęstsze błędy i jak je czytać](#najczęstsze-błędy-i-jak-je-czytać)
- [Słowniczek modułu](#słowniczek-modułu)
- [Dalsze czytanie](#dalsze-czytanie)
- [Co dalej](#co-dalej)

---

## 1. Problem: dwie tabele, jedna historia

Zacznijmy od sytuacji, którą znasz już z modułu 07. Mamy tabelę `author` i tabelę `book`. W tabeli `book` jest kolumna `author_id`, która wskazuje na `author.id`. To jest **klucz obcy** (*foreign key*) — liczba, która mówi: „ten wiersz należy do tamtego wiersza”.

Klucz obcy sam w sobie jest jednak tylko liczbą. Gdy piszesz kod w Pythonie, nie chcesz operować na liczbach:

```python
# Tak wygląda praca BEZ relacji — ręczne sklejanie po identyfikatorach
author = session.get(Author, 1)
book = Book(title="Zbrodnia i kara", author_id=author.id)   # sam wpisujesz FK
session.add(book)
session.commit()

# A żeby przejść z książki do autora, musisz zapytać bazę ręcznie:
book = session.get(Book, 7)
author = session.get(Author, book.author_id)   # drugie zapytanie, ręcznie
```

To działa, ale ma cztery wady, które zobaczysz dopiero po napisaniu dwustu linii takiego kodu:

1. **Musisz pamiętać o kolejności.** Nie możesz ustawić `book.author_id`, zanim autor nie ma `id`. A `id` pojawia się dopiero po `flush`. To jest dokładnie ten problem, który moduł 08 opisywał jako „kolejkowanie zmian”.
2. **Musisz pamiętać o spójności.** Jeśli zmienisz `book.author_id` na 5, ale w pamięci nadal trzymasz obiekt autora o `id=3` w jakiejś zmiennej, masz dwa różne światy w jednym programie.
3. **Każde przejście z jednej encji do drugiej to ręczne zapytanie.** W pętli po 100 książkach zrobisz 100 dodatkowych zapytań — nawet o tym nie wiedząc.
4. **Kod nie wyraża intencji.** `book.author_id = author.id` to instrukcja techniczna. `book.author = author` to zdanie z języka domeny: „ta książka ma tego autora”.

Relacje w SQLAlchemy rozwiązują wszystkie cztery problemy naraz. Ale zanim pokażemy API, musimy ustalić jedno: **relacja w SQLAlchemy to nie to samo co klucz obcy w bazie**.

> 💡 **Analogia — drzwi widziane z dwóch stron**
>
> Wyobraź sobie, że między dwoma pokojami są drzwi. Stojąc w pokoju A, widzisz drzwi prowadzące do pokoju B. Stojąc w pokoju B, widzisz dokładnie te same drzwi, prowadzące do pokoju A. To jedne drzwi, ale z dwóch stron wyglądają inaczej: z jednej strony jest klamka i szyld „do biblioteki”, z drugiej — „do kuchni”.
>
> Klucz obcy `book.author_id` to **fizyczne drzwi** — jeden fakt w bazie danych. Relacje `Book.author` i `Author.books` to **dwa sposoby patrzenia na te same drzwi** z wnętrza Pythona. SQLAlchemy nie tworzy dwóch drzwi. Tworzy dwa uchwyty do tych samych drzwi.

Konsekwencja praktyczna jest ogromna i wraca w całym module: **relacja nie jest kolumną**. `Book.author` nie istnieje w bazie danych. Istnieje `book.author_id`. `Book.author` to obiekt Pythona, który SQLAlchemy potrafi wyprodukować na podstawie tej kolumny.

> 🧠 **Dlaczego tak jest**
>
> Baza danych zna tylko tabele, wiersze i wartości. Nie zna pojęcia „obiekt autora”. ORM musi więc zbudować warstwę tłumaczącą: kolumna → atrybut, wiersz → obiekt, klucz obcy → relacja. Ta warstwa nazywa się **mapper** (mapujący). Relacje to część konfiguracji mappera, a nie część schematu bazy. Dlatego `relationship()` nie generuje żadnego `ALTER TABLE` ani `CREATE TABLE` — nie zmienia schematu. Zmienia tylko sposób, w jaki Python rozmawia z już istniejącym schematem.

---

## 2. Pierwsza relacja: jeden-do-wielu

Najczęstsza relacja w praktyce: **jeden autor ma wiele książek, każda książka ma jednego autora**. W żargonie: jeden-do-wielu (*one-to-many*) z perspektywy autora i wiele-do-jednego (*many-to-one*) z perspektywy książki. To ta sama relacja, oglądana z dwóch stron drzwi.

### 2.1. Model

```python
# examples/09_step1_one_to_many.py
from __future__ import annotations

from sqlalchemy import ForeignKey, String, create_engine
from sqlalchemy.orm import DeclarativeBase, Mapped, Session, mapped_column, relationship


class Base(DeclarativeBase):
    pass


class Author(Base):
    __tablename__ = "author"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(120))

    # Strona "jeden": kolekcja książek.
    books: Mapped[list[Book]] = relationship(back_populates="author")


class Book(Base):
    __tablename__ = "book"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))

    # Kolumna w bazie — to jest jedyna rzecz, która realnie istnieje w SQL.
    author_id: Mapped[int] = mapped_column(ForeignKey("author.id"))

    # Strona "wiele": pojedynczy obiekt autora.
    author: Mapped[Author] = relationship(back_populates="books")


engine = create_engine("sqlite:///:memory:", echo=True)
Base.metadata.create_all(engine)
```

Zwróć uwagę na cztery rzeczy, bo każda z nich jest nośnikiem informacji dla SQLAlchemy:

| Element | Co mówi SQLAlchemy |
|---|---|
| `Mapped[list[Book]]` | To jest strona „jeden”. Kolekcja, czyli relacja jeden-do-wielu. |
| `Mapped[Author]` (bez `list`) | To jest strona „wiele”. Pojedynczy obiekt, czyli wiele-do-jednego. |
| `ForeignKey("author.id")` na `author_id` | To jest kolumna łącząca. Bez niej SQLAlchemy nie wie, po czym łączyć. |
| `back_populates="author"` / `back_populates="books"` | Te dwa atrybuty to dwie strony tej samej relacji. Trzymaj je w synchronizacji. |

To ostatnie jest nowością względem starszych wersji: w SQLAlchemy 1.x i w wielu tutorialach z internetu zobaczysz `backref="books"` zamiast `back_populates`. Wyjaśniamy różnicę w sekcji 2.4.

### 2.2. Praca na obiektach — bez ani jednego `_id`

```python
# examples/09_step1_usage.py
from datetime import date

with Session(engine) as session:
    author = Author(name="Fiodor Dostojewski")
    author.books.append(Book(title="Zbrodnia i kara", published_on=date(1866, 1, 1)))
    author.books.append(Book(title="Bracia Karamazow", published_on=date(1880, 1, 1)))

    session.add(author)     # dodajemy TYLKO autora
    session.commit()

    print(author.id)                       # 1  — id pojawiło się po flushu
    print(author.books[0].author_id)       # 1  — SQLAlchemy ustawiło FK samo!
    print(author.books[0].author is author)  # True
```

Trzy zdania warte zatrzymania:

- **Dodaliśmy tylko autora.** Nie musieliśmy wywoływać `session.add()` na książkach. Zrobiła to za nas kaskada `save-update`, która jest domyślnie włączona (sekcja 8).
- **Nigdy nie wpisaliśmy `author_id`.** SQLAlchemy wie, że `book.author_id` to kolumna łącząca relacji `Book.author`, więc przy `flush` przenosi tożsamość obiektu autora na kolumnę klucza obcego.
- **`book.author is author` zwraca `True`.** To nie przypadek — to **identity map** z modułu 08. Ten sam wiersz bazy to zawsze ten sam obiekt Pythona w danej sesji.

> 🔬 **Pod maską — co poleciało do bazy**
>
> Przy `session.commit()` SQLAlchemy wykonał (w uproszczeniu):
>
> ```sql
> INSERT INTO author (name) VALUES (?)
> -- parametry: ('Fiodor Dostojewski',)
>
> INSERT INTO book (title, published_on, author_id) VALUES (?, ?, ?)
> -- parametry: ('Zbrodnia i kara', '1866-01-01', 1)
>
> INSERT INTO book (title, published_on, author_id) VALUES (?, ?, ?)
> -- parametry: ('Bracia Karamazow', '1880-01-01', 1)
> ```
>
> Kolejność jest istotna i nie jest przypadkowa. SQLAlchemy **posortował** operacje: najpierw `INSERT` do `author`, bo dopiero po nim istnieje `id=1`, którego potrzebują wiersze w `book`. To jest dokładnie ta „jednostka pracy” z modułu 08 — kolejka zmian, którą ktoś musi ułożyć w poprawną kolejność. W tym przypadku ułożył ją algorytm sortowania topologicznego po zależnościach kluczy obcych.

### 2.3. Odczytywanie w obie strony

```python
# examples/09_step1_read.py
from sqlalchemy import select

with Session(engine) as session:
    # Od strony "wiele" do "jeden" — przejście po relacji
    book = session.scalars(select(Book).where(Book.title == "Zbrodnia i kara")).one()
    print(book.author.name)          # Fiodor Dostojewski

    # Od strony "jeden" do "wiele" — iteracja po kolekcji
    author = session.get(Author, 1)
    for b in author.books:
        print(b.title)
```

I tu pojawia się pierwszy poważny temat wydajnościowy tego modułu.

> 🔬 **Pod maską — leniwe ładowanie (lazy loading)**
>
> `book.author.name` **nie jest** odczytem z pamięci. To dodatkowe zapytanie do bazy:
>
> ```sql
> SELECT author.id, author.name FROM author WHERE author.id = ?
> -- parametry: (1,)
> ```
>
> Podobnie `for b in author.books`:
>
> ```sql
> SELECT book.id, book.title, book.author_id FROM book WHERE ? = book.author_id
> -- parametry: (1,)
> ```
>
> SQLAlchemy domyślnie ładuje relacje **leniwie** (*lazy*): nie pobiera ich, dopóki nie poprosisz o dostęp. Zaleta: nie pobierasz danych, których nie użyjesz. Wada: nie wiesz, ile zapytań wysyłasz, dopóki nie policzysz. To źródło problemu N+1, do którego wrócimy w sekcji 13 i w całości w module 11.

### 2.4. `back_populates` kontra `backref`

Oba parametry robią pozornie to samo: łączą dwie strony relacji. Różnica jest istotna.

**`back_populates`** — jawne po obu stronach. Piszesz dwa razy, ale widzisz całość:

```python
class Author(Base):
    books: Mapped[list["Book"]] = relationship(back_populates="author")

class Book(Base):
    author: Mapped["Author"] = relationship(back_populates="books")
```

**`backref`** — jedna strona tworzy drugą automatycznie:

```python
class Book(Base):
    author: Mapped["Author"] = relationship(backref="books")
    # SQLAlchemy samo doda atrybut Author.books
```

Kiedy `backref` wciąż ma sens? W trzech sytuacjach:

- w krótkich skryptach jednorazowych, gdzie liczy się zwięzłość;
- gdy relacja „w drugą stronę” nie jest częścią publicznego API i chcesz ją ukryć;
- w starszym kodzie, który już tak działa (migracja nie jest obowiązkowa).

Kiedy `back_populates` jest wyraźnie lepsze?

1. **Typowanie i IDE.** Przy `backref` atrybut `Author.books` nie istnieje w źródle, więc `mypy` go nie widzi, a edytor nie podpowie. Przy `back_populates` masz pełne `Mapped[list[Book]]`.
2. **Czytelność.** Chcesz wiedzieć, gdzie jest druga strona, nie zgadując.
3. **Konfiguracja.** Przy `backref` część parametrów trzeba przekazać przez `backref(...)`, co rozjeżdża konfigurację na dwa miejsca.
4. **Wielokrotne relacje.** Gdy tabela ma trzy relacje do tej samej tabeli, `backref` staje się nieczytelny.

W SQLAlchemy 2.0 `backref` nadal działa i nie jest formalnie przestarzały, ale dokumentacja konsekwentnie używa `back_populates`, i my też tak zrobimy w całym kursie.

> ⚠️ **Pułapka — brak `back_populates` powoduje „rozjazd w pamięci”**
>
> ```python
> class Author(Base):
>     books: Mapped[list["Book"]] = relationship()          # brak back_populates
>
> class Book(Base):
>     author: Mapped["Author"] = relationship()             # brak back_populates
> ```
>
> Teraz:
>
> ```python
> author = Author(name="X")
> book = Book(title="Y")
>
> book.author = author
> print(author.books)        # [] — pusta lista!
> ```
>
> Dwa atrybuty zachowują się jak dwie **niezależne** relacje, choć wskazują tę samą kolumnę. To najbardziej mylący błąd w całym module, bo **baza zapisze się poprawnie** (obie relacje zapisują ten sam `author_id`), ale w pamięci Twoje obiekty kłamią. Objawy: „dodałem książkę do autora, ale `author.books` jest puste”, „usunąłem z kolekcji, ale obiekt nadal ma autora”.
>
> **Naprawa:** zawsze dodawaj `back_populates` po obu stronach. Jeśli zapomnisz, SQLAlchemy ostrzeże Cię komunikatem `SAWarning: relationship 'Book.author' will copy column ... which conflicts with relationship(s) ...` — o tym w tabeli błędów na końcu.

> 🧪 **Ćwiczenie — w miejscu**
>
> Dodaj do modelu klasę `Publisher` (wydawca) i relację jeden-do-wielu: jeden wydawca ma wiele książek. Nie ustawiaj `book.publisher_id` ręcznie — sprawdź, czy `Book.publisher` i `Publisher.books` są spójne po `author.books[0].publisher = wydawca`.

---

## 3. Parametry relationship() — pełny przegląd

`relationship()` ma kilkadziesiąt parametrów. W praktyce używa się kilkunastu. Poniżej te, które naprawdę spotkasz, z podziałem na kategorie.

### 3.1. Parametry łączące dwie strony

| Parametr | Do czego służy |
|---|---|
| `back_populates` | Nazwa atrybutu po drugiej stronie relacji. Domyślnie brak — i wtedy strony są niezależne. |
| `backref` | Skrót: tworzy drugą stronę automatycznie. Relikt 1.x, ale działa. |
| `foreign_keys` | Które kolumny traktować jako klucz obcy. Obowiązkowe, gdy jest ich kilka. |
| `primaryjoin` | Ręczny warunek łączenia (`ON`). Używany, gdy warunek nie wynika z kluczy obcych. |
| `secondary` | Tabela pośrednia dla relacji wiele-do-wielu. |
| `secondaryjoin` | Warunek łączenia z tabelą pośrednią, gdy `primaryjoin` nie wystarcza. |
| `remote_side` | Dla relacji samo-referencyjnych: która kolumna jest „dalszą” stroną (rodzicem). |
| `uselist` | Wymusza pojedynczy obiekt (`False`) zamiast kolekcji. Używane przy 1:1 i przy `Mapped` z adnotacją. |
| `order_by` | Sortowanie kolekcji przy każdym ładowaniu. |

### 3.2. Parametry zachowania

| Parametr | Do czego służy |
|---|---|
| `cascade` | Co ma się dziać z obiektami powiązanymi przy zapisie i usunięciu. |
| `passive_deletes` | Czy SQLAlchemy ma ładować dzieci, żeby je usunąć, czy zaufać bazie. |
| `lazy` | Strategia ładowania: `"select"`, `"joined"`, `"selectin"`, `"raise"`, `"write_only"`. |
| `viewonly` | Relacja tylko do odczytu — nie zapisuje zmian. |
| `innerjoin` | Czy eager loading ma używać `INNER JOIN` zamiast `LEFT OUTER JOIN`. |
| `single_parent` | Wymusza, że obiekt ma tylko jednego rodzica. Używane przy `delete-orphan` na 1:1. |

### 3.3. Parametry typowe dla N:M

| Parametr | Do czego służy |
|---|---|
| `secondary` | Nazwa tabeli asocjacyjnej (obiekt `Table` albo string). |
| `secondaryjoin` | Dodatkowy warunek przy łączeniu z tabelą pośrednią. |

> 🧠 **Dlaczego tak dużo parametrów?**
>
> Bo `relationship()` musi rozwiązać problem, który w SQL jest trywialny, a w obiektach niejednoznaczny. W SQL piszesz `FROM book JOIN author ON book.author_id = author.id`. W Pythonie piszesz tylko `relationship()` — i SQLAlchemy musi **odgadnąć**, że chodzi o `author_id`. Gdy istnieje dokładnie jedna ścieżka kluczy obcych, odgadnie. Gdy istnieją dwie (np. `author_id` i `editor_id`), zgadnąć nie może i wymaga `foreign_keys`. Gdy kluczy obcych nie ma wcale, wymaga `primaryjoin`. To nie nadmiarowość — to jedyny sposób, żeby „magia” była przewidywalna.

---

## 4. Jeden-do-jednego

Relacja jeden-do-jednego (*one-to-one*) to jeden-do-wielu, w którym druga strona ma zagwarantowaną co najwyżej jedną pozycję. Klasyczny przykład: użytkownik i jego profil.

W SQL realizuje się to tak samo jak 1:N, z jednym dodatkiem: **kolumna klucza obcego musi być unikalna**. Bez tego baza pozwoli na dwa profile dla jednego użytkownika, a Python będzie miał problem: relacja „jeden” zobaczy dwa obiekty i nie będzie wiedział, który zwrócić.

```python
# examples/09_step2_one_to_one.py
from __future__ import annotations

from datetime import date

from sqlalchemy import ForeignKey, String, create_engine
from sqlalchemy.orm import DeclarativeBase, Mapped, Session, mapped_column, relationship


class Base(DeclarativeBase):
    pass


class Member(Base):
    __tablename__ = "member"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(120))

    # Strona "jeden" relacji 1:1. uselist=False => pojedynczy obiekt, nie lista.
    profile: Mapped["MemberProfile | None"] = relationship(
        back_populates="member",
        uselist=False,
        cascade="all, delete-orphan",
        single_parent=True,
    )


class MemberProfile(Base):
    __tablename__ = "member_profile"

    id: Mapped[int] = mapped_column(primary_key=True)
    # unique=True jest KLUCZOWE — bez tego to nie jest relacja 1:1.
    member_id: Mapped[int] = mapped_column(
        ForeignKey("member.id", ondelete="CASCADE"),
        unique=True,
    )
    nickname: Mapped[str] = mapped_column(String(60))
    joined_on: Mapped[date] = mapped_column(default=date.today)

    member: Mapped[Member] = relationship(back_populates="profile")
```

Cztery elementy warte omówienia:

1. **`uselist=False`** — mówi SQLAlchemy: „z tej strony nie ma listy, jest jeden obiekt albo `None`”. Bez tego `member.profile` byłoby listą, co byłoby mylące i niezgodne z adnotacją `Mapped[MemberProfile | None]`.
2. **`unique=True`** — gwarancja po stronie bazy. Bez tego SQLAlchemy przy ładowaniu `uselist=False` zgłosi ostrzeżenie, gdy znajdzie więcej niż jeden wiersz, i po cichu zwróci pierwszy. Bazy danych nie da się oszukać — jeśli chcesz 1:1, wymuś to indeksem unikalnym.
3. **`cascade="all, delete-orphan"`** — usunięcie użytkownika usuwa jego profil; odłączenie profilu od użytkownika też go usuwa (bo jest „sierotą”, *orphan*).
4. **`single_parent=True`** — wymagane przy `delete-orphan` w relacji, która nie jest kolekcją. Bez tego SQLAlchemy zgłosi `ArgumentError`, bo nie może stwierdzić, że profil ma tylko jednego właściciela.

> 🔬 **Pod maską — odczyt profilu**
>
> ```python
> member = session.get(Member, 1)
> print(member.profile.nickname)
> ```
>
> Wygeneruje (leniwe ładowanie, jak przy 1:N):
>
> ```sql
> SELECT member_profile.id, member_profile.member_id, member_profile.nickname, ...
> FROM member_profile
> WHERE ? = member_profile.member_id
> -- parametry: (1,)
> ```
>
> Różnica względem 1:N jest tylko w tym, co SQLAlchemy zrobi z wynikiem: przy `uselist=False` weźmie pierwszy wiersz jako obiekt. Dlatego bez `unique=True` na kolumnie bazy wynik jest niedeterministyczny.

> ⚠️ **Pułapka — 1:1 bez unikalnego klucza obcego**
>
> Jeśli zapomnisz `unique=True`, kod będzie działał w testach (bo testy mają po jednym profilu), a na produkcji zacznie zwracać losowe rekordy. Objaw: „nickname użytkownika zmienia się bez powodu”. To nie jest problem SQLAlchemy — to problem schematu. Reguła: **relacja 1:1 zawsze z unikalnym kluczem obcym**.

---

## 5. Wiele-do-wielu

Książka może należeć do wielu kategorii, a kategoria obejmować wiele książek. W relacyjnej bazie nie da się tego zapisać w dwóch tabelach — potrzebna jest trzecia, tak zwana **tabela asocjacyjna** (*association table*) albo tabela łącząca.

> 💡 **Analogia — lista obecności**
>
> Wyobraź sobie konferencję. Masz listę uczestników i listę sesji. Żeby wiedzieć, kto jest na jakiej sesji, nie dopisujesz nazwisk do listy sesji (bo sesja może mieć stu uczestników) ani nie dopisujesz sesji do uczestnika (bo uczestnik może pójść na pięć sesji). Zakładasz **trzecią kartkę**: lista obecności, na której każdy wiersz to jedna para „uczestnik–sesja”. Ta kartka nie zawiera nic więcej niż dwie kolumny — i to jest właśnie tabela asocjacyjna.

### 5.1. Prosta tabela asocjacyjna

```python
# examples/09_step3_many_to_many.py
from __future__ import annotations

from sqlalchemy import Column, ForeignKey, String, Table, create_engine
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    pass


# Tabela asocjacyjna: zwykły obiekt Table, NIE klasa modelu.
# Nie ma własnego id — klucz główny to para (book_id, category_id).
book_category = Table(
    "book_category",
    Base.metadata,
    Column("book_id", ForeignKey("book.id", ondelete="CASCADE"), primary_key=True),
    Column("category_id", ForeignKey("category.id", ondelete="CASCADE"), primary_key=True),
)


class Category(Base):
    __tablename__ = "category"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(60), unique=True)

    books: Mapped[list["Book"]] = relationship(
        secondary=book_category,
        back_populates="categories",
    )


class Book(Base):
    __tablename__ = "book"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))

    categories: Mapped[list[Category]] = relationship(
        secondary=book_category,
        back_populates="categories",
    )
```

Trzy rzeczy do zapamiętania:

1. **Tabela asocjacyjna to `Table`, nie klasa.** Nie ma jej w kodzie jako model, bo nie ma własnych danych poza parą kluczy. Nie potrzebujesz `Base.metadata` w definicji osobno — `Base.metadata` jako drugi argument rejestruje ją w schemacie.
2. **`primary_key=True` na obu kolumnach** — to klucz złożony. Zapewnia, że nie da się dwa razy dodać tej samej pary. To jest Twoja ochrona przed duplikatami.
3. **`secondary=book_category`** — ten jeden parametr zmienia relację z 1:N w N:M. SQLAlchemy od tej pory wie, że między `book` i `category` trzeba przechodzić przez trzecią tabelę.

Praca z N:M wygląda dokładnie jak z listą:

```python
# examples/09_step3_usage.py
from sqlalchemy import select
from sqlalchemy.orm import Session

with Session(engine) as session:
    drama = Category(name="Dramat")
    classic = Category(name="Klasyka")

    book = Book(title="Zbrodnia i kara")
    book.categories.extend([drama, classic])   # albo .append() dwa razy

    session.add(book)
    session.commit()

    # Zapytanie "od drugiej strony" — książki w kategorii "Dramat"
    drama_db = session.scalars(
        select(Category).where(Category.name == "Dramat")
    ).one()
    print([b.title for b in drama_db.books])   # ['Zbrodnia i kara']
```

> 🔬 **Pod maską — wstawienie i odczyt N:M**
>
> Przy `session.commit()`:
>
> ```sql
> INSERT INTO category (name) VALUES (?)
> -- ('Dramat',)
> INSERT INTO category (name) VALUES (?)
> -- ('Klasyka',)
> INSERT INTO book (title) VALUES (?)
> -- ('Zbrodnia i kara',)
> INSERT INTO book_category (book_id, category_id) VALUES (?, ?)
> -- (1, 1)
> INSERT INTO book_category (book_id, category_id) VALUES (?, ?)
> -- (1, 2)
> ```
>
> Zwróć uwagę: **nie ma `UPDATE`**. Kategorie nie mają kolumny w `book`. Cała relacja żyje w tabeli pośredniej, a SQLAlchemy wstawia tam wiersze dopiero wtedy, gdy obie strony mają już swoje `id` — dlatego `INSERT INTO book_category` jest na końcu.
>
> Przy odczycie `drama_db.books`:
>
> ```sql
> SELECT book.id, book.title
> FROM book
> JOIN book_category ON book.id = book_category.book_id
> WHERE ? = book_category.category_id
> -- parametr: (1,)
> ```

> ⚠️ **Pułapka — duplikaty przy zapytaniach przez N:M**
>
> Zapytanie `select(Book).join(Book.categories).where(Category.name == "Klasyka")` zwróci książkę **raz**, bo filtr jest na kategorii. Ale `select(Book).join(Book.categories)` **bez filtra** zwróci książkę tyle razy, ile ma kategorii. To poprawne zachowanie SQL, nie błąd SQLAlchemy — JOIN zawsze produkuje wiersz na każdą pasującą kombinację. Rozwiązania: `distinct()`, `select(Book).where(Book.categories.any(Category.name == "Klasyka"))`, albo `selectinload`. Wrócimy do tego w module 10 i 11.

### 5.2. Kiedy tabela pośrednia naprawdę potrzebuje własnych kolumn

Prosta tabela asocjacyjna ma tylko dwie kolumny. To wystarcza, gdy relacja jest „czysta”: książka jest w kategorii albo nie. Ale bardzo często relacja sama w sobie niesie informację:

- **polubienia**: kto, co i **kiedy** polubił;
- **role użytkownika w projekcie**: użytkownik, projekt i **rola** (`owner`, `editor`, `viewer`);
- **oceny**: książka, czytelnik i **ocena** (1–5) plus **data**;
- **stan magazynowy w zamówieniu**: produkt, zamówienie i **liczba sztuk** oraz **cena z momentu zakupu**.

W takich przypadkach tabela pośrednia przestaje być „kartką z parą identyfikatorów” i staje się pełnoprawną encją. To wzorzec **Association Object** (obiekt asocjacyjny), omówiony w następnej sekcji.

> 🧠 **Dlaczego nie dopisać kolumny do tabeli pośredniej bez tworzenia klasy?**
>
> Technicznie można — `Table` przyjmie dowolną liczbę kolumn. Problem pojawia się przy zapisie: SQLAlchemy przy relacji `secondary` zarządza tabelą pośrednią **automatycznie**, wstawiając i usuwając wiersze bez Twojej wiedzy. Jeśli chcesz ustawić `liked_at`, musiałbyś pisać ręczne `INSERT` do tabeli pośredniej obok pracy na obiektach — i natychmiast tracisz cały zysk z ORM. Dlatego konwencja jest jednoznaczna: **tabela pośrednia bez danych → `Table`; tabela pośrednia z danymi → pełna klasa**.

---

## 6. Association Object — gdy tabela pośrednia ma własne dane

Rozbudujmy przykład biblioteki: chcemy wiedzieć, **kiedy** książka została przypisana do kategorii i **kto** to zrobił. Tabela `book_category` dostaje dwie dodatkowe kolumny. Wtedy staje się modelem.

```python
# examples/09_step4_association_object.py
from __future__ import annotations

from datetime import datetime

from sqlalchemy import DateTime, ForeignKey, String, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    pass


class BookCategory(Base):
    """Pełnoprawna encja: wiersz tabeli pośredniej z własnymi danymi."""

    __tablename__ = "book_category"

    book_id: Mapped[int] = mapped_column(
        ForeignKey("book.id", ondelete="CASCADE"), primary_key=True
    )
    category_id: Mapped[int] = mapped_column(
        ForeignKey("category.id", ondelete="CASCADE"), primary_key=True
    )
    added_at: Mapped[datetime] = mapped_column(
        DateTime, server_default=func.now(), nullable=False
    )
    added_by: Mapped[str | None] = mapped_column(String(120))

    # Relacje do obu "końców" — to dzięki nim poruszamy się po grafie.
    book: Mapped["Book"] = relationship(back_populates="category_links")
    category: Mapped["Category"] = relationship(back_populates="book_links")


class Book(Base):
    __tablename__ = "book"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))

    # "Prawdziwa" relacja: praca przez obiekt asocjacyjny.
    category_links: Mapped[list[BookCategory]] = relationship(
        back_populates="book",
        cascade="all, delete-orphan",
    )

    # Wygodny widok "tylko do czytania" — lista kategorii bez tabeli pośredniej.
    categories: Mapped[list["Category"]] = relationship(
        secondary="book_category",
        viewonly=True,
    )


class Category(Base):
    __tablename__ = "category"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(60), unique=True)

    book_links: Mapped[list[BookCategory]] = relationship(back_populates="category")
```

Dwie relacje na `Book` pełnią różne role:

| Atrybut | Typ | Do czego |
|---|---|---|
| `Book.category_links` | `list[BookCategory]` | Zapis. Tu ustawiasz `added_by` i czytasz `added_at`. |
| `Book.categories` | `list[Category]` | Odczyt. Wygodny skrót, gdy nie interesują Cię dane pośrednie. |

Praca z takim modelem:

```python
# examples/09_step4_usage.py
from sqlalchemy.orm import Session

with Session(engine) as session:
    book = Book(title="Bracia Karamazow")
    classic = Category(name="Klasyka")

    # Zapis: budujemy obiekt asocjacyjny i przypinamy go do książki.
    book.category_links.append(
        BookCategory(category=classic, added_by="librarian@example.com")
    )

    session.add(book)
    session.commit()

    # Odczyt po stronie "wygodnej":
    print([c.name for c in book.categories])          # ['Klasyka']

    # Odczyt po stronie "bogatej":
    link = book.category_links[0]
    print(link.added_by, link.added_at)               # librarian@example.com 2026-...
```

> ⚠️ **Pułapka — `secondary` z `viewonly=True` jest obowiązkowe**
>
> Gdyby `Book.categories` nie miało `viewonly=True`, SQLAlchemy próbowałoby zarządzać tabelą `book_category` **jednocześnie** przez `category_links` i przez `categories`. Dwa mechanizmy pisałyby do tej samej tabeli, co kończy się ostrzeżeniami `SAWarning` o nakładających się relacjach i nieprzewidywalnym SQL. Reguła: **przy Association Object widok przez `secondary` zawsze z `viewonly=True`**.
>
> Dodatkowo: `secondary="book_category"` podajemy jako **string**, bo klasa `BookCategory` jest mapowana do tej samej tabeli — a SQLAlchemy musi wiedzieć, że chodzi o tabelę, nie o klasę. String rozwiązuje problem kolejności definicji.

> 🧠 **Dlaczego `secondary` przyjmuje string?**
>
> Bo SQLAlchemy musi znaleźć tabelę w `MetaData` po nazwie, a nie klasę w rejestrze. W chwili przetwarzania `Book` klasa `Category` może jeszcze nie istnieć — string jest odroczony do momentu konfiguracji mappera, kiedy wszystko jest już zdefiniowane. Ten sam mechanizm działa dla `order_by="Book.title"` i `remote_side`.

> 🧪 **Ćwiczenie — w miejscu**
>
> Dodaj do `BookCategory` kolumnę `is_primary: Mapped[bool]` (domyślnie `False`), oznaczającą kategorię główną książki. Napisz zapytanie zwracające dla każdej książki jej kategorię główną. Zastanów się, czy lepiej zrobić to przez `category_links` (filtrując po `is_primary`), czy przez osobne `ForeignKey` na `Book`.

---

## 7. Relacje samo-referencyjne

Czasem tabela wskazuje sama na siebie. Klasyczny przykład: kategoria ma nadrzędną kategorię, a ta z kolei swoją nadrzędną. Albo: pracownik ma przełożonego, który też jest pracownikiem.

```python
# examples/09_step5_self_referential.py
from __future__ import annotations

from sqlalchemy import ForeignKey, String
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    pass


class Category(Base):
    __tablename__ = "category"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(60))

    parent_id: Mapped[int | None] = mapped_column(
        ForeignKey("category.id", ondelete="SET NULL")
    )

    # Strona "wiele": kategoria ma jedną kategorię nadrzędną (albo żadnej).
    parent: Mapped["Category | None"] = relationship(
        back_populates="children",
        remote_side="Category.id",
    )

    # Strona "jeden": kategoria ma wiele podkategorii.
    children: Mapped[list["Category"]] = relationship(
        back_populates="parent",
        order_by="Category.name",
    )
```

Dlaczego potrzebne jest `remote_side`?

> 💡 **Analogia — drzewo genealogiczne na jednej kartce**
>
> Wyobraź sobie, że wszystkie osoby w rodzinie są na jednej wielkiej liście, a każda ma rubrykę „rodzic: numer wiersza”. Gdy patrzysz na wiersz, widzisz dwa kierunki: **w górę** („mój rodzic”) i **w dół** („moje dzieci”). Ale z samej rubryki nie wynika, który kierunek jest który — rubryka to tylko liczba. `remote_side` mówi SQLAlchemy: „strona wskazywana przez klucz obcy to **rodzic**, więc wiersze, które wskazują na mnie, to **dzieci**”.

Bez `remote_side` SQLAlchemy zgłosi `AmbiguousForeignKeysError` — nie wie, po której stronie relacji postawić „jeden”, a po której „wiele”. Z `remote_side="Category.id"` sprawa jest jasna: `Category.id` to strona „jeden” (rodzic), a `parent_id` to strona „wiele”.

Alternatywa to `remote_side=[id]` — lista obiektów kolumn w zasięgu klasy. Oba warianty są równoważne; string jest bezpieczniejszy przy `from __future__ import annotations` i rozbiciu modeli na pliki.

> 🔬 **Pod maską — wczytanie drzewa**
>
> ```python
> root = session.scalars(select(Category).where(Category.parent_id.is_(None))).one()
> for child in root.children:
>     print(child.name)
> ```
>
> ```sql
> SELECT category.id, category.name, category.parent_id
> FROM category WHERE category.parent_id IS NULL
>
> SELECT category.id, category.name, category.parent_id
> FROM category WHERE ? = category.parent_id ORDER BY category.name
> -- parametr: (1,)
> ```
>
> Zwróć uwagę: `parent_id` jest w bazie **nullable** (`Mapped[int | None]`), bo korzeń drzewa nie ma rodzica. To jest jeden z tych przypadków, w których adnotacja typowa `int | None` **jest** deklaracją `nullable=True` — mechanizm opisany w module 07.

> ⚠️ **Pułapka — rekurencja bez końca**
>
> Jeśli przez pomyłkę ustawisz kategorię jako swojego własnego rodzica (`cat.parent = cat`), SQLAlchemy nie zgłosi błędu przy zapisie — baza też nie (klucz obcy jest spełniony). Problem pojawi się dopiero przy odczycie drzewa rekurencyjnie: nieskończona pętla. Reguła: **walidację cykli w hierarchii rób w kodzie aplikacji**, najlepiej w `@validates` (moduł 13) albo w warstwie serwisowej (moduł 19). Baza nie obroni Cię przed cyklem.

---

## 8. Kaskady — kto za kim sprząta

Kaskada to reguła mówiąca, co zrobić z obiektami powiązanymi, gdy coś dzieje się z obiektem głównym. Parametr `cascade` przyjmuje string z listą nazw, np. `"all, delete-orphan"`.

> 💡 **Analogia — co zabrać ze sobą, gdy burzę dom**
>
> Wyobraź sobie, że burzysz stary dom. Masz listę decyzji: czy zabrać meble (są częścią domu)? Czy zabrać sąsiada (nie jest)? Czy jeśli meble zostały wyniesione na podwórko i nikt ich nie chce, to czy są śmieciem (sierotami)? Kaskada w SQLAlchemy to dokładnie ta lista decyzji, zapisana raz, w jednym miejscu, zamiast w pięćdziesięciu miejscach kodu.

### 8.1. Dostępne wartości

| Wartość | Znaczenie |
|---|---|
| `save-update` | Obiekty dodane do sesji przez relację też są zapisywane. **Domyślnie włączone.** |
| `merge` | `session.merge()` propaguje się na obiekty powiązane. **Domyślnie włączone.** |
| `delete` | Usunięcie rodzica usuwa dzieci. **Domyślnie WYŁĄCZONE.** |
| `delete-orphan` | Dziecko odłączone od rodzica jest usuwane. Wymaga `delete`. |
| `refresh-expire` | `session.refresh()`/`expire()` propaguje się na dzieci. **Domyślnie włączone.** |
| `expunge` | `session.expunge()` propaguje się na dzieci. **Domyślnie wyłączone.** |
| `all` | Skrót na `save-update, merge, refresh-expire, expunge, delete`. **Nie zawiera `delete-orphan`.** |
| `all, delete-orphan` | Pełny zestaw razem z usuwaniem sierot. |

Trzy rzeczy, które zaskakują najczęściej:

1. **`delete` nie jest domyślne.** Jeśli nic nie ustawisz, usunięcie autora **nie usunie jego książek** — a SQLAlchemy dodatkowo spróbuje ustawić `book.author_id = NULL`, co przy kolumnie `NOT NULL` zakończy się `IntegrityError`. To celowa decyzja projektowa: biblioteka nie usuwa danych, o które nie prosiłeś.
2. **`all` nie zawiera `delete-orphan`.** Musisz je dopisać ręcznie. To również celowe — usuwanie sierot jest zbyt agresywne, żeby włączać je domyślnie.
3. **`delete-orphan` ma sens tylko po stronie „jeden”.** Na stronie „wiele” (wiele-do-jednego) SQLAlchemy zgłosi błąd.

```python
# examples/09_step6_cascade_ok.py
class Author(Base):
    __tablename__ = "author"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(120))

    books: Mapped[list["Book"]] = relationship(
        back_populates="author",
        cascade="all, delete-orphan",   # prawidłowo: strona "jeden"
    )
```

```python
# examples/09_step6_cascade_bad.py
class Book(Base):
    __tablename__ = "book"
    id: Mapped[int] = mapped_column(primary_key=True)
    author_id: Mapped[int] = mapped_column(ForeignKey("author.id"))

    author: Mapped["Author"] = relationship(
        back_populates="books",
        cascade="all, delete-orphan",   # BŁĄD: strona "wiele"
    )
```

> ⚠️ **Pułapka — `delete-orphan` po stronie „wiele”**
>
> SQLAlchemy zgłosi `ArgumentError` o treści zbliżonej do:
>
> ```text
> For many-to-one relationship Book.author, delete-orphan cascade is normally configured
> only on the "one" side of a one-to-many relationship, and not on the "many" side of a
> many-to-one relationship.
> ```
>
> Logika jest prosta: „sierota” to obiekt, który stracił **rodzica**. Strona „wiele” nie ma dzieci, więc nie może mieć sierot. Gdybyś chciał, żeby usunięcie autora usuwało książki, kaskadę ustawiasz na `Author.books` — nie odwrotnie. *(Dokładne brzmienie komunikatu może się nieznacznie różnić między wydaniami 2.0.x.)*

### 8.2. Kiedy `delete-orphan` jest zbawienny

- **Wartości osadzone w agregacie.** Pozycje zamówienia nie mają sensu bez zamówienia. Jeśli usuniesz pozycję z `order.items`, chcesz, żeby zniknęła z bazy.
- **Relacje 1:1.** Profil bez użytkownika jest bezsensowny.
- **Tabele asocjacyjne z danymi (Association Object).** Gdy odłączasz kategorię od książki, wiersz pośredni powinien zniknąć.
- **Encje zależne w agregatach DDD.** Tam `delete-orphan` to wręcz wzorzec architektoniczny (moduł 19).

### 8.3. Kiedy `delete-orphan` jest niebezpieczny

- **Gdy obiekt może być współdzielony.** Wyobraź sobie tag używany przez tysiąc artykułów. Jeśli włączysz `delete-orphan` na `Article.tags`, to **odłączenie tagu od jednego artykułu usunie tag globalnie** — a wraz z nim jego powiązania ze wszystkimi pozostałymi artykułami. To katastrofa, którą łatwo przeoczyć w testach z jednym artykułem.
- **Gdy „odłączenie” jest normalną operacją.** Jeśli często przenosisz obiekty między rodzicami, `delete-orphan` może usuwać je w trakcie przenoszenia.
- **Gdy dane mają wartość historyczną.** Historia wypożyczeń nie powinna znikać, gdy książka zostanie usunięta z katalogu.

> 🧠 **Dlaczego współdzielone obiekty i `delete-orphan` się nie lubią**
>
> `delete-orphan` pyta: „czy ten obiekt ma jeszcze jakiegokolwiek rodzica?”. Przy relacji wiele-do-wielu z prostą tabelą pośrednią obiekt może mieć wielu rodziców. SQLAlchemy obsługuje ten przypadek tylko wtedy, gdy powie się mu `single_parent=True` — i wtedy odłączenie od jednego rodzica staje się odłączeniem od jedynego, czyli usunięciem. To prawie nigdy nie jest to, czego chcesz. Dlatego `delete-orphan` stosuje się do relacji, gdzie każdy obiekt ma dokładnie jednego właściciela.

> 🧪 **Ćwiczenie — w miejscu**
>
> Mając model biblioteki z sekcji 14, zastanów się: co się stanie po `book.categories.remove(kategoria)` przy `cascade="all, delete-orphan"` na tej relacji? Odpowiedź zapisz, a potem sprawdź w kodzie. (Podpowiedź: usunie się wiersz w `book_category`, a nie kategoria.)

---

## 9. Kto usuwa dane: Python czy baza?

Mamy dwie możliwości usuwania powiązanych danych:

- **Python** — SQLAlchemy ładuje dzieci do pamięci i wystawia `DELETE` dla każdego z osobna.
- **Baza** — w kluczu obcym ustawiamy `ON DELETE CASCADE`, a SQLAlchemy wystawia jedno `DELETE` na rodzica.

```python
# examples/09_step7_db_side_delete.py
class Author(Base):
    __tablename__ = "author"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(120))

    books: Mapped[list["Book"]] = relationship(
        back_populates="author",
        cascade="all, delete-orphan",
        passive_deletes=True,        # nie ładuj dzieci, zaufaj bazie
    )


class Book(Base):
    __tablename__ = "book"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))

    author_id: Mapped[int] = mapped_column(
        ForeignKey("author.id", ondelete="CASCADE")   # baza usunie sama
    )

    author: Mapped[Author] = relationship(back_populates="books")
```

Porównanie:

| Aspekt | Kaskada w Pythonie | `ON DELETE CASCADE` + `passive_deletes=True` |
|---|---|---|
| Liczba zapytań przy usunięciu autora z 100 książkami | ~201 (`SELECT` + 100 `DELETE book` + 100 `DELETE review` + `DELETE author`) | 1 (`DELETE author`) |
| Pamięć | Ładuje wszystkie dzieci do identity map | Nie ładuje niczego |
| Kontrola w kodzie | Pełna — możesz dodać logikę, audyt, warunki | Brak — baza usuwa bez pytania |
| Wymaga migracji | Nie | Tak (zmiana klucza obcego) |
| Ryzyko | Wolne przy dużych zbiorach | Nieodwracalne, jeśli się pomylisz |
| Widoczność w logach aplikacji | Widać każdy usunięty wiersz | Widać tylko jedno `DELETE` |

Trzy poziomy „passywności”:

| Wartość | Zachowanie |
|---|---|
| `passive_deletes=False` (domyślnie) | SQLAlchemy ładuje dzieci i sam je usuwa (albo ustawia FK na `NULL`). |
| `passive_deletes=True` | Przy usunięciu rodzica SQLAlchemy **nie ładuje** dzieci — zakłada, że baza ma `ON DELETE CASCADE`. Dzieci już załadowane do sesji i tak zostaną obsłużone. |
| `passive_deletes="all"` | SQLAlchemy nigdy nie ładuje ani nie modyfikuje dzieci przy usuwaniu. Wymaga `ON DELETE CASCADE` albo `ON DELETE SET NULL`. |

> ⚠️ **Pułapka — `passive_deletes=True` bez `ondelete` w bazie**
>
> Jeśli ustawisz `passive_deletes=True`, ale w `ForeignKey` nie ma `ondelete="CASCADE"`, to baza przy usunięciu autora zostawi książki z nieistniejącym `author_id` — albo, jeśli klucz obcy jest wymuszany, zgłosi `IntegrityError` z bazy, którego SQLAlchemy nie zrozumie w kontekście Pythona. Objaw: `IntegrityError: FOREIGN KEY constraint failed` przy `session.delete(author)`, choć żadnej książki nie usuwałeś.
>
> Reguła: **`passive_deletes=True` idzie w parze z `ondelete="CASCADE"` (albo `SET NULL`)**. To dwie strony jednej decyzji.

> 🧠 **Dlaczego `cascade="all, delete-orphan"` i `passive_deletes=True` mogą współistnieć**
>
> Kaskada `delete-orphan` działa w momencie, gdy odłączasz obiekt od kolekcji (np. `author.books.remove(book)`). `passive_deletes` działa przy usunięciu **rodzica**. To dwa różne zdarzenia. Możesz więc mieć jedno i drugie: odłączone książki usuwane po stronie Pythona, a usunięcie całego autora obsłużone jednym `DELETE` przez bazę. Uwaga jednak: przy `passive_deletes=True` SQLAlchemy nie wykryje sierot w momencie usuwania rodzica — jeśli książka była „sierotą” w sensie logicznym, ale nigdy nie została odłączona, baza ją usunie tak czy owak dzięki `CASCADE`.

**Który wariant wybrać?** Praktyczna zasada:

- Małe zbiory, potrzebny audyt, logika biznesowa przy usuwaniu → kaskada w Pythonie.
- Duże zbiory, hierarchia danych, wydajność → `ON DELETE CASCADE` + `passive_deletes=True`.
- W obu przypadkach: **zawsze przetestuj usuwanie na realistycznych danych**, bo błąd w kaskadach objawia się dopiero na produkcji.

---

## 10. Dwie relacje do tej samej tabeli

Wróćmy do biblioteki. Chcemy wiedzieć nie tylko, kto **napisał** książkę, ale też kto ją **zredagował**. Obie osoby to `User` (albo `Author`). Powstają dwa klucze obce do tej samej tabeli:

```python
# examples/09_step8_two_fks.py
from __future__ import annotations

from sqlalchemy import ForeignKey, String
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    pass


class Person(Base):
    __tablename__ = "person"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(120))

    written_books: Mapped[list["Book"]] = relationship(
        back_populates="author",
        foreign_keys="Book.author_id",     # jawnie wskazujemy kolumnę
    )
    edited_books: Mapped[list["Book"]] = relationship(
        back_populates="editor",
        foreign_keys="Book.editor_id",
    )


class Book(Base):
    __tablename__ = "book"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))

    author_id: Mapped[int] = mapped_column(ForeignKey("person.id"))
    editor_id: Mapped[int | None] = mapped_column(ForeignKey("person.id"))

    author: Mapped[Person] = relationship(
        back_populates="written_books",
        foreign_keys=[author_id],
    )
    editor: Mapped[Person | None] = relationship(
        back_populates="edited_books",
        foreign_keys=[editor_id],
    )
```

> ⚠️ **Pułapka — `AmbiguousForeignKeysError`**
>
> Bez `foreign_keys` SQLAlchemy nie może rozstrzygnąć, czy `Book.author` łączy się przez `author_id`, czy przez `editor_id`. Komunikat:
>
> ```text
> sqlalchemy.exc.AmbiguousForeignKeysError: Could not determine join condition between
> parent/child tables on relationship Book.author - there are multiple foreign key paths
> linking the tables. Specify the 'foreign_keys' argument, providing a list of those
> columns which should be counted as containing a foreign key reference to the parent table.
> ```
>
> Naprawa: `foreign_keys="Book.author_id"` (string) albo `foreign_keys=[author_id]` (lista kolumn w zasięgu klasy). Oba warianty są poprawne. Przy `from __future__ import annotations` i rozbiciu na pliki bezpieczniejszy jest string.

> ⚠️ **Pułapka — ostrzeżenie o nakładających się relacjach**
>
> Przy dwóch relacjach do tej samej tabeli możesz zobaczyć:
>
> ```text
> SAWarning: relationship 'Book.editor' will copy column person.id to column book.editor_id,
> which conflicts with relationship(s): 'Book.author' (copies person.id to column
> book.author_id). If this is not the intention, consider if these relationships should be
> linked with back_populates, or if viewonly=True should be applied to one or more of them.
> ```
>
> To ostrzeżenie pojawia się, gdy SQLAlchemy podejrzewa, że dwie relacje mogą próbować pisać do tych samych kolumn — najczęściej dlatego, że brakuje `back_populates` albo `foreign_keys`. **Nie ignoruj go.** W naszym przykładzie `back_populates` i `foreign_keys` są ustawione, więc ostrzeżenia nie będzie. Jeśli jednak budujesz relację tylko do czytania (np. `all_books_by_any_person`), dodaj `viewonly=True` i ostrzeżenie zniknie.

---

## 11. viewonly i synonym

### 11.1. `viewonly=True`

Relacja oznaczona jako `viewonly` jest **tylko do odczytu**. SQLAlchemy ją ładuje, ale nigdy nie zapisuje zmian wprowadzonych przez ten atrybut.

Kiedy to ma sens?

1. **Widok przez `secondary` przy Association Object** (sekcja 6) — obowiązkowo.
2. **Relacja pomocnicza do raportów** — np. `Author.recent_books` z `primaryjoin` filtrującym po dacie.
3. **Gdy druga strona relacji jest już zarządzana** — żeby nie było dwóch mechanizmów piszących do tej samej kolumny.
4. **Gdy relacja jest zbudowana na wyrażeniu, którego nie da się odwrócić** — np. łączenie po funkcji.

```python
# examples/09_step9_viewonly.py
from sqlalchemy import func, select
from sqlalchemy.orm import relationship

class Book(Base):
    # ...
    reviews: Mapped[list["Review"]] = relationship(
        back_populates="book",
        cascade="all, delete-orphan",
    )

    # Relacja tylko do odczytu — nie zarządzamy nią, tylko czytamy.
    recent_reviews: Mapped[list["Review"]] = relationship(
        primaryjoin="and_(Book.id == Review.book_id, Review.rating >= 4)",
        viewonly=True,
        order_by="Review.created_at.desc()",
    )
```

> ⚠️ **Pułapka — modyfikowanie relacji `viewonly`**
>
> ```python
> book.recent_reviews.append(Review(rating=5))   # NIE zadziała!
> ```
>
> Zmiana trafi do pamięci (kolekcja to zwykła lista), ale **nie zostanie zapisana do bazy**. To najgorszy rodzaj błędu: brak wyjątku, brak ostrzeżenia, ciche zgubienie danych. W SQLAlchemy 2.0 przy próbie modyfikacji takiej kolekcji pojawia się ostrzeżenie, ale nie polegaj na nim — **traktuj `viewonly` jako kontrakt, którego nie wolno łamać**.

### 11.2. `synonym()`

`synonym()` tworzy alias atrybutu. Używa się go, gdy chcesz mieć dwie nazwy na tę samą rzecz — np. nazwę techniczną i nazwę domenową.

```python
# examples/09_step9_synonym.py
from sqlalchemy import String
from sqlalchemy.orm import Mapped, mapped_column, synonym

class Member(Base):
    __tablename__ = "member"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(120))

    # Kolumna w bazie nazywa się "email", ale atrybut domenowy to "email_address".
    email_address: Mapped[str] = mapped_column("email", String(200), unique=True)

    # Alias: działa też w zapytaniach (select(Member.email)).
    email = synonym("email_address")
```

Zastosowania:

- **Migracja nazewnictwa bez migracji schematu.** Zmieniasz nazwę atrybutu w Pythonie, ale zostawiasz nazwę kolumny w bazie.
- **Czytelność w zapytaniach.** `select(Member.email)` zamiast `select(Member.email_address)`.
- **Zgodność z zewnętrznym kontraktem.** Kod klienta używa `obj.email`, a Twoja domena mówi `email_address`.

> 🧠 **Dlaczego nie robić tego przez `@property`?**
>
> Bo `@property` nie działa w zapytaniach. `select(Member.email)` z `@property` nie zadziała — SQLAlchemy nie wie, jak zamienić property na kolumnę. `synonym()` jest zintegrowany z mapperem, więc działa i w Pythonie, i w SQL. To ten sam mechanizm, co `hybrid_property` z modułu 13, tylko prostszy (bez logiki).

---

## 12. Strategie ładowania — lazy=...

Parametr `lazy` decyduje, **kiedy** SQLAlchemy pobiera dane relacji. To najważniejszy parametr wydajnościowy w całym module.

| Wartość | Kiedy ładuje | Ile zapytań przy N obiektach | Kiedy używać |
|---|---|---|---|
| `"select"` (domyślnie) | Przy pierwszym dostępie do atrybutu | $O(1 + N)$ | Małe zbiory, relacje rzadko używane |
| `"joined"` | Natychmiast, `LEFT OUTER JOIN` w tym samym zapytaniu | $O(1)$ | Relacje wiele-do-jednego, 1:1 |
| `"selectin"` | Natychmiast, drugie zapytanie z `IN (...)` | $O(2)$ | Kolekcje (jeden-do-wielu, N:M) |
| `"subquery"` | Natychmiast, podzapytanie | $O(2)$ | Rzadko; historycznie do kolekcji |
| `"raise"` | Nigdy — zgłasza błąd | — | Wymuszenie jawnego ładowania |
| `"write_only"` | Nigdy (kolekcja tylko do zapisu) | — | Bardzo duże kolekcje |
| `"dynamic"` | Relikt 1.x | — | Nie używać — zastąpione przez `write_only` |

```python
# examples/09_step10_lazy_joined.py
class Book(Base):
    __tablename__ = "book"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    author_id: Mapped[int] = mapped_column(ForeignKey("author.id"))

    # Wiele-do-jednego: joined jest zwykle najlepszym wyborem.
    author: Mapped["Author"] = relationship(
        back_populates="books",
        lazy="joined",
    )


class Author(Base):
    __tablename__ = "author"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(120))

    # Kolekcja: selectin jest zwykle najlepszym wyborem.
    books: Mapped[list[Book]] = relationship(
        back_populates="author",
        lazy="selectin",
    )
```

> 🔬 **Pod maską — `lazy="select"` vs `lazy="selectin"`**
>
> Przy `lazy="select"` i pętli po 100 książkach:
>
> ```sql
> SELECT book.id, book.title, book.author_id FROM book        -- 1 zapytanie
> SELECT author.id, author.name FROM author WHERE author.id = ?   -- × 100
> ```
>
> Razem **101 zapytań**. To jest problem N+1.
>
> Przy `lazy="selectin"`:
>
> ```sql
> SELECT book.id, book.title, book.author_id FROM book        -- 1 zapytanie
> SELECT author.id, author.name FROM author
>   WHERE author.id IN (?, ?, ?, ..., ?)                      -- 1 zapytanie
> ```
>
> Razem **2 zapytania** — niezależnie od tego, czy książek jest 10, 100 czy 10 000. To jest cała różnica.
>
> Przy `lazy="joined"`:
>
> ```sql
> SELECT book.id, book.title, book.author_id,
>        author_1.id, author_1.name
> FROM book LEFT OUTER JOIN author AS author_1 ON author_1.id = book.author_id
> ```
>
> Jedno zapytanie, ale szersze — każdy wiersz książki powtarza dane autora.

### 12.1. Wybór strategii na poziomie zapytania

Ustawienie `lazy` na relacji to **domyślna** polityka. Możesz ją nadpisać dla konkretnego zapytania:

```python
# examples/09_step10_query_level.py
from sqlalchemy import select
from sqlalchemy.orm import joinedload, selectinload

# Autor zawsze ładowany JOIN-em, książki przez IN (...)
stmt = (
    select(Book)
    .options(joinedload(Book.author), selectinload(Book.categories))
)

books = session.scalars(stmt).unique().all()
```

Trzy uwagi:

1. **`.unique()` jest wymagane**, gdy używasz `joinedload` na **kolekcji** — bo JOIN produkuje duplikaty wierszy, a SQLAlchemy musi je zdeduplikować w obiektach. Bez tego dostaniesz `InvalidRequestError`. Przy `joinedload` na relacji wiele-do-jednego `.unique()` nie jest konieczne (jeden wiersz = jedna książka), ale nie zaszkodzi.
2. **Dwie `joinedload` na dwóch kolekcjach jednocześnie** produkują „wybuch kartezjański” — iloczyn rozmiarów. Przy 100 kategoriach i 50 recenzjach jedna książka da 5000 wierszy. Dlatego kolekcje ładujemy przez `selectinload`.
3. **`lazy="raise"`** to świetne zabezpieczenie w testach: każda próba leniwego ładowania zgłosi błąd zamiast cicho wysyłać zapytanie. Dzięki temu regresja wydajnościowa zostanie złapana w CI, a nie na produkcji.

> 🧪 **Ćwiczenie — w miejscu**
>
> Ustaw `lazy="raise"` na `Author.books` i spróbuj wykonać pętlę `for b in author.books`. Zobacz komunikat błędu. Potem dodaj `selectinload(Author.books)` do zapytania i sprawdź, że działa.

Pełne omówienie strategii ładowania, w tym `defer`, `load_only`, `raiseload`, `with_loader_criteria` i `contains_eager`, znajduje się w module 11.

> 🆕 **SQLAlchemy 2.1 — `selectinload`**
>
> W 2.1 `selectinload` zyskał dwa ulepszenia: parametr `chunksize`, który dzieli `IN (...)` na porcje (przydatne, gdy lista identyfikatorów jest tak długa, że baza odrzuca zapytanie albo plan staje się zły), oraz `omit_join` dla relacji wiele-do-wielu, który pozwala pominąć dodatkowy `JOIN` do tabeli pośredniej. W 2.0.5x nie ma tych parametrów.

---

## 13. Modyfikowanie kolekcji i skąd bierze się N+1

Kolekcje w SQLAlchemy zachowują się jak listy, ale mają dodatkową logikę: rejestrują zmiany i przekazują je do jednostki pracy.

### 13.1. Trzy operacje i ich skutki

```python
# examples/09_step11_collections.py
with Session(engine) as session:
    author = Author(name="Nowy Autor")
    b1 = Book(title="Książka 1")
    b2 = Book(title="Książka 2")

    # 1. append — dodanie do kolekcji
    author.books.append(b1)
    author.books.append(b2)

    session.add(author)
    session.flush()
    # INSERT author; INSERT book × 2 (z author_id ustawionym automatycznie)

    # 2. remove — odłączenie
    author.books.remove(b2)
    session.flush()
    # Bez delete-orphan: UPDATE book SET author_id = NULL WHERE id = 2
    # Z delete-orphan:    DELETE FROM book WHERE id = 2

    # 3. clear — opróżnienie kolekcji
    author.books.clear()
    session.flush()
    # Analogicznie, dla każdego elementu
```

| Operacja | Bez `delete-orphan` | Z `delete-orphan` |
|---|---|---|
| `append(x)` | `INSERT`/`UPDATE` z FK na rodzica | to samo |
| `remove(x)` | `UPDATE x SET fk = NULL` | `DELETE x` |
| `clear()` | `UPDATE ... SET fk = NULL` dla każdego | `DELETE` dla każdego |
| `usunięcie rodzica` | `UPDATE ... SET fk = NULL` (lub błąd przy `NOT NULL`) | `DELETE` dzieci + `DELETE` rodzica |

> ⚠️ **Pułapka — `remove` bez `delete-orphan` przy kolumnie `NOT NULL`**
>
> Jeśli `book.author_id` jest `NOT NULL` (bo `Mapped[int]`, nie `Mapped[int | None]`), to `author.books.remove(book)` spróbuje ustawić `author_id = NULL` i baza zgłosi:
>
> ```text
> sqlalchemy.exc.IntegrityError: (sqlite3.IntegrityError) NOT NULL constraint failed: book.author_id
> ```
>
> To nie jest błąd SQLAlchemy — to sygnał, że próbujesz zrobić rzecz niedozwoloną przez schemat. Chcesz albo `delete-orphan` (usuń książkę), albo kolumnę nullable (książka bez autora), albo przenieść książkę do innego autora zamiast usuwać.

### 13.2. Skąd bierze się N+1 w pętli po kolekcji

To najczęstszy realny problem wydajnościowy w aplikacjach z ORM. Klasyczny kod:

```python
# examples/09_step11_n_plus_1_bad.py
# ❌ ŹLE — 1 + N zapytań
with Session(engine) as session:
    authors = session.scalars(select(Author)).all()        # 1 zapytanie
    for author in authors:                                  # N iteracji
        for book in author.books:                           # N dodatkowych zapytań!
            print(f"{author.name}: {book.title}")
```

Przy 50 autorach to 51 zapytań. Przy 5000 autorów — 5001. Każde zapytanie to runda do bazy: wysłanie, planowanie, wykonanie, przesłanie wyniku. Na lokalnym SQLite tego nie zauważysz. Na PostgreSQL przez sieć — 200 ms zamieni się w 20 sekund.

```python
# examples/09_step11_n_plus_1_good.py
# ✅ DOBRZE — 2 zapytania, niezależnie od liczby autorów
from sqlalchemy.orm import selectinload

with Session(engine) as session:
    stmt = select(Author).options(selectinload(Author.books))
    authors = session.scalars(stmt).all()                   # 1 zapytanie
    for author in authors:
        for book in author.books:                           # z pamięci, 0 zapytań
            print(f"{author.name}: {book.title}")
```

> 🔬 **Pod maską — porównanie**
>
> **Wersja zła:**
>
> ```sql
> SELECT author.id, author.name FROM author                       -- 1
> SELECT book.id, book.title, book.author_id FROM book
>   WHERE ? = book.author_id                                       -- × N
> ```
>
> **Wersja dobra:**
>
> ```sql
> SELECT author.id, author.name FROM author                       -- 1
> SELECT book.author_id AS book_author_id, book.id, book.title
> FROM book WHERE book.author_id IN (?, ?, ?, ...)                 -- 1
> ```
>
> Druga wersja wysyła **jedno** zapytanie z listą identyfikatorów. Koszt: $O(1)$ rund do bazy, plus $O(N)$ transferu danych (który i tak musisz ponieść).

### 13.3. Trzy sposoby na uniknięcie N+1

| Sposób | Kiedy stosować | Wada |
|---|---|---|
| `lazy="selectin"` na relacji | Gdy relacja jest używana niemal zawsze | Ładuje kolekcję nawet, gdy nie jest potrzebna |
| `selectinload()` w zapytaniu | Gdy tylko niektóre ścieżki kodu potrzebują relacji | Trzeba pamiętać o dodaniu w każdym miejscu |
| `lazy="raise"` + jawne opcje | W projektach, gdzie wydajność jest krytyczna | Więcej kodu, ale zero niespodzianek |

> 🧠 **Dlaczego leniwe ładowanie w ogóle istnieje, jeśli jest takie ryzykowne?**
>
> Bo jest wygodne i często optymalne. Wyobraź sobie stronę z listą autorów, gdzie pokazujesz tylko nazwiska — po co ładować 50 000 książek? `lazy="select"` sprawia, że płacisz tylko za to, czego użyjesz. Problem pojawia się, gdy używasz relacji **w pętli** — wtedy leniwe ładowanie zamienia się w lawinę zapytań. Rozwiązaniem nie jest wyłączenie leniwości, ale świadome ładowanie tam, gdzie wiesz, że będziesz potrzebować danych.

---

## 14. Pełny przykład: biblioteka

Zbudujmy kompletny model, który łączy wszystko z tego modułu: 1:N, 1:1, N:M, Association Object, composite key, kaskady i samo-referencję.

```python
# examples/09_library_full.py
"""Kompletny model biblioteki — moduł 09 kursu SQLAlchemy.

Uruchomienie:
    pip install "SQLAlchemy>=2.0"
    python examples/09_library_full.py
"""
from __future__ import annotations

from datetime import date, datetime

from sqlalchemy import (
    CheckConstraint,
    Date,
    DateTime,
    ForeignKey,
    String,
    Table,
    Column,
    Text,
    create_engine,
    func,
    select,
)
from sqlalchemy.orm import (
    DeclarativeBase,
    Mapped,
    Session,
    mapped_column,
    relationship,
    selectinload,
)


class Base(DeclarativeBase):
    pass


# --------------------------------------------------------------------------
# 1. Tabela asocjacyjna N:M bez własnych danych: książka <-> kategoria
# --------------------------------------------------------------------------
book_category = Table(
    "book_category",
    Base.metadata,
    Column("book_id", ForeignKey("book.id", ondelete="CASCADE"), primary_key=True),
    Column(
        "category_id",
        ForeignKey("category.id", ondelete="CASCADE"),
        primary_key=True,
    ),
)


# --------------------------------------------------------------------------
# 2. Association Object: książka <-> tag z datą i autorem przypisania
# --------------------------------------------------------------------------
class BookTag(Base):
    """Wiersz pośredni z własnymi danymi — pełnoprawna encja."""

    __tablename__ = "book_tag"

    book_id: Mapped[int] = mapped_column(
        ForeignKey("book.id", ondelete="CASCADE"), primary_key=True
    )
    tag_id: Mapped[int] = mapped_column(
        ForeignKey("tag.id", ondelete="CASCADE"), primary_key=True
    )
    added_at: Mapped[datetime] = mapped_column(
        DateTime, server_default=func.now(), nullable=False
    )
    added_by: Mapped[str | None] = mapped_column(String(120))

    book: Mapped["Book"] = relationship(back_populates="tag_links")
    tag: Mapped["Tag"] = relationship(back_populates="book_links")


class Tag(Base):
    __tablename__ = "tag"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(60), unique=True)

    book_links: Mapped[list[BookTag]] = relationship(
        back_populates="tag",
        cascade="all, delete-orphan",
    )


# --------------------------------------------------------------------------
# 3. Hierarchia kategorii — relacja samo-referencyjna
# --------------------------------------------------------------------------
class Category(Base):
    __tablename__ = "category"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(60))

    parent_id: Mapped[int | None] = mapped_column(
        ForeignKey("category.id", ondelete="SET NULL")
    )

    parent: Mapped["Category | None"] = relationship(
        back_populates="children",
        remote_side="Category.id",
    )
    children: Mapped[list["Category"]] = relationship(
        back_populates="parent",
        order_by="Category.name",
    )

    books: Mapped[list["Book"]] = relationship(
        secondary=book_category,
        back_populates="categories",
    )

    __table_args__ = (
        # Nazwa kategorii unikalna w obrębie rodzica — typowy wymóg domenowy.
        # (SQLite traktuje NULL-e jako różne, więc korzenie nie kolidują.)
        CheckConstraint("length(name) > 0", name="ck_category_name_not_empty"),
    )


# --------------------------------------------------------------------------
# 4. Autor — strona "jeden" relacji 1:N
# --------------------------------------------------------------------------
class Author(Base):
    __tablename__ = "author"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(120), index=True)

    books: Mapped[list["Book"]] = relationship(
        back_populates="author",
        cascade="all, delete-orphan",
        order_by="Book.title",
    )

    def __repr__(self) -> str:
        return f"Author(id={self.id!r}, name={self.name!r})"


# --------------------------------------------------------------------------
# 5. Książka — węzeł centralny grafu
# --------------------------------------------------------------------------
class Book(Base):
    __tablename__ = "book"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200), index=True)
    isbn: Mapped[str | None] = mapped_column(String(13), unique=True)
    published_on: Mapped[date | None] = mapped_column(Date)

    author_id: Mapped[int] = mapped_column(ForeignKey("author.id", ondelete="CASCADE"))
    author: Mapped[Author] = relationship(back_populates="books")

    categories: Mapped[list[Category]] = relationship(
        secondary=book_category,
        back_populates="categories",   # UWAGA: patrz ostrzeżenie niżej
        lazy="selectin",
    )

    tag_links: Mapped[list[BookTag]] = relationship(
        back_populates="book",
        cascade="all, delete-orphan",
        lazy="selectin",
    )

    reviews: Mapped[list["Review"]] = relationship(
        back_populates="book",
        cascade="all, delete-orphan",
        passive_deletes=True,
    )

    loans: Mapped[list["Loan"]] = relationship(back_populates="book")

    def __repr__(self) -> str:
        return f"Book(id={self.id!r}, title={self.title!r})"


# --------------------------------------------------------------------------
# 6. Czytelnik — 1:1 z profilem, 1:N z wypożyczeniami i recenzjami
# --------------------------------------------------------------------------
class Member(Base):
    __tablename__ = "member"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(120))
    email: Mapped[str] = mapped_column("email", String(200), unique=True)
    joined_on: Mapped[date] = mapped_column(Date, default=date.today)

    profile: Mapped["MemberProfile | None"] = relationship(
        back_populates="member",
        uselist=False,
        cascade="all, delete-orphan",
        single_parent=True,
    )
    loans: Mapped[list["Loan"]] = relationship(back_populates="member")
    reviews: Mapped[list["Review"]] = relationship(back_populates="member")


class MemberProfile(Base):
    __tablename__ = "member_profile"

    id: Mapped[int] = mapped_column(primary_key=True)
    member_id: Mapped[int] = mapped_column(
        ForeignKey("member.id", ondelete="CASCADE"),
        unique=True,          # to czyni relację 1:1
    )
    nickname: Mapped[str] = mapped_column(String(60))
    bio: Mapped[str | None] = mapped_column(Text)

    member: Mapped[Member] = relationship(back_populates="profile")


# --------------------------------------------------------------------------
# 7. Wypożyczenie — wiele-do-jednego do książki i czytelnika
# --------------------------------------------------------------------------
class Loan(Base):
    __tablename__ = "loan"

    id: Mapped[int] = mapped_column(primary_key=True)
    book_id: Mapped[int] = mapped_column(ForeignKey("book.id"))
    member_id: Mapped[int] = mapped_column(ForeignKey("member.id"))

    loaned_on: Mapped[date] = mapped_column(Date, default=date.today)
    due_on: Mapped[date] = mapped_column(Date)
    returned_on: Mapped[date | None] = mapped_column(Date)

    book: Mapped[Book] = relationship(back_populates="loans")
    member: Mapped[Member] = relationship(back_populates="loans")

    __table_args__ = (
        CheckConstraint("due_on >= loaned_on", name="ck_loan_due_after_loan"),
    )


# --------------------------------------------------------------------------
# 8. Recenzja — KLUCZ ZŁOŻONY (book_id, member_id)
# --------------------------------------------------------------------------
class Review(Base):
    __tablename__ = "review"

    book_id: Mapped[int] = mapped_column(
        ForeignKey("book.id", ondelete="CASCADE"), primary_key=True
    )
    member_id: Mapped[int] = mapped_column(
        ForeignKey("member.id"), primary_key=True
    )
    rating: Mapped[int]
    comment: Mapped[str | None] = mapped_column(Text)
    created_at: Mapped[datetime] = mapped_column(
        DateTime, server_default=func.now(), nullable=False
    )

    book: Mapped[Book] = relationship(back_populates="reviews")
    member: Mapped[Member] = relationship(back_populates="reviews")

    __table_args__ = (
        CheckConstraint("rating BETWEEN 1 AND 5", name="ck_review_rating_range"),
    )
```

> ⚠️ **Uwaga do kodu powyżej — literówka do wyłapania**
>
> W `Book.categories` jest `back_populates="categories"`, a powinno być `back_populates="books"` (bo po stronie `Category` atrybut nazywa się `books`). To celowy przykład błędu z sekcji 2.4 — SQLAlchemy zgłosi `ArgumentError: relationship 'Book.categories' ... does not reference a mapped attribute`. **Poprawna wersja: `back_populates="books"`.** Zapamiętaj ten wzorzec: `back_populates` zawsze wskazuje nazwę atrybutu po drugiej stronie, nie nazwę klasy ani tabeli.

### 14.1. Skrypt demonstracyjny

```python
# examples/09_library_demo.py
from datetime import date, timedelta

from sqlalchemy import select

engine = create_engine("sqlite:///library.db", echo=False)
Base.metadata.create_all(engine)

with Session(engine) as session:
    # --- Wstawianie przez obiekty: ZERO ręcznych *_id -------------------
    classic = Category(name="Klasyka")
    drama = Category(name="Dramat")
    classic.children.append(drama)          # hierarchia samo-referencyjna

    dostoevsky = Author(name="Fiodor Dostojewski")
    book = Book(title="Zbrodnia i kara", published_on=date(1866, 1, 1), isbn="9788307032327")
    book.categories.extend([classic, drama])          # N:M przez secondary
    book.tag_links.append(BookTag(tag=Tag(name="rosyjska"), added_by="librarian@lib.pl"))
    dostoevsky.books.append(book)

    member = Member(name="Anna Kowalska", email="anna@example.com")
    member.profile = MemberProfile(nickname="anka", bio="Czyta dużo.")

    session.add_all([dostoevsky, member])
    session.commit()

    print("id autora:", dostoevsky.id)
    print("author_id książki ustawione automatycznie:", book.author_id)
    print("kategorie:", [c.name for c in book.categories])
    print("tagi:", [link.tag.name for link in book.tag_links])
    print("profil:", member.profile.nickname)

    # --- Recenzja z kluczem złożonym ------------------------------------
    session.add(Review(book=book, member=member, rating=5, comment="Arcydzieło."))
    session.commit()

    # --- Usuwanie autora: analiza SQL -----------------------------------
    print("\n--- DELETE autora ---")
    session.delete(dostoevsky)
    session.commit()
```

### 14.2. Analiza wygenerowanego SQL przy usuwaniu autora

Uruchom skrypt z `echo=True` i zobaczysz (w uproszczeniu, kolejność może się różnić między wersjami):

```sql
-- 1. Załaduj książki autora (potrzebne, bo nie ma passive_deletes na Author.books)
SELECT book.id, book.title, book.author_id FROM book WHERE ? = book.author_id;

-- 2. Załaduj recenzje tych książek (delete-orphan + brak passive_deletes? -- jest!)
--    Book.reviews ma passive_deletes=True, więc ten SELECT się NIE pojawi,
--    o ile książki nie są już w identity map. Jeśli są -- SQLAlchemy je obsłuży.

-- 3. Usuń wiersze asocjacyjne N:M dla każdej książki
DELETE FROM book_category WHERE book_category.book_id = ? AND book_category.category_id = ?;

-- 4. Usuń książki
DELETE FROM book WHERE book.id = ?;

-- 5. Usuń autora
DELETE FROM author WHERE author.id = ?;
```

Wnioski z tego, co widać:

| Obserwacja | Wniosek |
|---|---|
| `SELECT book ... WHERE author_id = ?` | SQLAlchemy **musi** załadować dzieci, żeby je usunąć. To koszt. |
| `DELETE FROM book_category` dla każdej pary | Relacja `secondary` zawsze ładuje wiersze pośrednie, żeby je usunąć. `passive_deletes` nie pomoże. |
| Brak `DELETE FROM review` | Bo `Book.reviews` ma `passive_deletes=True` + `ondelete="CASCADE"` w bazie. |
| Brak `DELETE FROM loan` | Bo `Loan` **nie ma** kaskady — jeśli byłyby wypożyczenia, baza zgłosiłaby `IntegrityError`. |

Ostatni punkt jest kluczowy dla bezpieczeństwa danych: **historia wypożyczeń nie znika razem z książką**. To celowa decyzja domenowa. Ale oznacza, że usunięcie książki z historią zakończy się błędem:

```text
sqlalchemy.exc.IntegrityError: (sqlite3.IntegrityError) FOREIGN KEY constraint failed
```

Naprawa: `Loan.book_id` z `ondelete="RESTRICT"` (jawnie) i logika biznesowa „archiwizuj książkę zamiast usuwać” (`is_archived`), albo `ondelete="SET NULL"` z `Mapped[int | None]`. To dobry moment, żeby przypomnieć sobie moduł 05 i „miękkie usuwanie”.

> 🧠 **Dlaczego SQLAlchemy ładuje dzieci, zamiast po prostu wystawić `DELETE`**
>
> Bo kaskady są regułami **Pythona**, a nie SQL-a. `delete-orphan` wymaga wiedzy, które obiekty są sierotami. `save-update` wymaga wiedzy, które obiekty trzeba zapisać. SQLAlchemy nie może tego zgadnąć bez załadowania danych — chyba że powiesz mu `passive_deletes`, czyli „zaufaj bazie”. To jest dokładnie ta różnica, którą omówiliśmy w sekcji 9.

---

## Podsumowanie

1. **Relacja to nie kolumna.** `Book.author` nie istnieje w bazie. Istnieje `book.author_id`. `relationship()` konfiguruje mappera, nie schemat.
2. **Dwie strony tej samej relacji.** `back_populates` po obu stronach utrzymuje spójność pamięci. Bez tego baza zapisze się poprawnie, ale obiekty będą kłamać.
3. **Kardynalność wynika z adnotacji.** `Mapped[list[X]]` → strona „jeden”. `Mapped[X]` → strona „wiele”. `uselist=False` → 1:1 (plus `unique=True` w bazie!).
4. **N:M wymaga trzeciej tabeli.** Prosta para kluczy → `Table` + `secondary`. Własne kolumny → Association Object z `viewonly=True` na widoku `secondary`.
5. **Samo-referencja wymaga `remote_side`.** Bez tego SQLAlchemy nie wie, po której stronie jest „jeden”.
6. **`delete` NIE jest domyślne.** Domyślnie działa tylko `save-update`, `merge`, `refresh-expire`. Usuwanie musisz włączyć świadomie.
7. **`delete-orphan` tylko po stronie „jeden”.** Po stronie „wiele” to błąd konfiguracji. I uwaga na obiekty współdzielone — tam `delete-orphan` jest niebezpieczny.
8. **`passive_deletes=True` idzie w parze z `ondelete="CASCADE"`.** Razem dają jedno zapytanie zamiast setek; osobno dają błędy integralności.
9. **Dwie relacje do tej samej tabeli wymagają `foreign_keys`.** Inaczej `AmbiguousForeignKeysError`.
10. **`lazy` decyduje o wydajności.** `select` = $O(1+N)$, `selectin`/`joined` = $O(1)$–$O(2)$. Kolekcje → `selectin`, wiele-do-jednego → `joined`, testy → `raise`.

---

## Ćwiczenia

### Zadanie 1 (łatwe) — hierarchia kategorii

Rozbuduj model z sekcji 14 tak, aby kategorie miały pełną hierarchię, i napisz funkcję:

```python
def print_tree(category: Category, indent: int = 0) -> None:
    """Wypisz drzewo kategorii z wcięciami."""
```

Funkcja ma działać na relacji `Category.children`, ale **nie może generować N+1 zapytań** przy wywołaniu na korzeniu drzewa o 5 poziomach. Zdecyduj, którą strategię ładowania zastosować.

### Zadanie 2 (średnie) — polubienia jako Association Object

Dodaj do modelu relację „polubienia”: czytelnik może polubić książkę, a polubienie ma datę i (opcjonalnie) komentarz. Użyj wzorca Association Object. Napisz:

1. Modele `BookLike`, relacje dwustronne i widok `viewonly` na `Book.liked_by`.
2. Zapytanie zwracające 10 najczęściej polubionych książek z liczbą polubień.
3. Zapytanie sprawdzające, czy dany czytelnik polubił daną książkę — w jednym zapytaniu, bez ładowania obiektów.

### Zadanie 3 (trudne) — usuwanie autora w dwóch wariantach

Dla modelu z sekcji 14 napisz dwa skrypty:

- **Wariant A:** kaskady po stronie Pythona. Usuń autora z 500 książkami i zmierz czas oraz policz zapytania.
- **Wariant B:** `ondelete="CASCADE"` na `Book.author_id` + `passive_deletes=True` na `Author.books`. To samo usunięcie.

Porównaj wyniki w tabeli. Wyjaśnij, dlaczego różnica rośnie wraz z liczbą książek, i w jakich sytuacjach wybrałbyś wariant A, mimo że jest wolniejszy.

---

### Rozwiązania

<details>
<summary><b>Zadanie 1 — hierarchia kategorii</b></summary>

Problem: `Category.children` domyślnie ładuje się leniwie. Rekurencyjne `print_tree` wywoła jedno zapytanie na każdy węzeł — dokładnie N+1, tylko ukryte w rekurencji.

Rozwiązanie: `lazy="selectin"` na `Category.children`. Wtedy pierwsze zapytanie pobiera korzeń, drugie pobiera **wszystkie** kategorie z `parent_id IN (id korzenia, ...)`.

```python
# examples/09_ex1_tree.py
from sqlalchemy import select
from sqlalchemy.orm import Session, selectinload


class Category(Base):
    # ... (reszta jak w sekcji 14)
    children: Mapped[list["Category"]] = relationship(
        back_populates="parent",
        order_by="Category.name",
        lazy="selectin",      # <-- kluczowa zmiana
    )


def print_tree(category: Category, indent: int = 0) -> None:
    print("  " * indent + category.name)
    for child in category.children:
        print_tree(child, indent + 1)


with Session(engine) as session:
    root = session.scalars(
        select(Category).where(Category.parent_id.is_(None))
    ).one()
    print_tree(root)
```

**Ale uwaga:** `selectin` ładuje tylko **jeden poziom** w głąb przy każdym zapytaniu. Przy 5 poziomach i tak powstaną dodatkowe zapytania — po jednym na poziom. To wciąż $O(\text{głębokość})$, a nie $O(\text{liczba węzłów})$, czyli ogromna poprawa.

**Alternatywa dla głębokich drzew:** rekurencyjne CTE (`with_recursive`) w jednym zapytaniu — moduł 06. Albo `join_depth` na relacji, które każe SQLAlchemy zagnieżdżać eager loading do określonej głębokości:

```python
children: Mapped[list["Category"]] = relationship(
    back_populates="parent",
    order_by="Category.name",
    lazy="joined",
    join_depth=4,        # eager load 4 poziomy w głąb
)
```

**Pułapka:** `join_depth` z `joined` na kolekcji generuje duży JOIN i wymaga `.unique()`. Przy szerokim drzewie lepiej zostać przy `selectin` i zaakceptować kilka zapytań.

</details>

<details>
<summary><b>Zadanie 2 — polubienia jako Association Object</b></summary>

```python
# examples/09_ex2_likes.py
from __future__ import annotations

from datetime import datetime

from sqlalchemy import DateTime, ForeignKey, String, func, select, func as sa_func
from sqlalchemy.orm import Mapped, mapped_column, relationship


class BookLike(Base):
    __tablename__ = "book_like"

    book_id: Mapped[int] = mapped_column(
        ForeignKey("book.id", ondelete="CASCADE"), primary_key=True
    )
    member_id: Mapped[int] = mapped_column(
        ForeignKey("member.id", ondelete="CASCADE"), primary_key=True
    )
    liked_at: Mapped[datetime] = mapped_column(
        DateTime, server_default=func.now(), nullable=False
    )
    note: Mapped[str | None] = mapped_column(String(280))

    book: Mapped["Book"] = relationship(back_populates="likes")
    member: Mapped["Member"] = relationship(back_populates="likes")


class Book(Base):
    # ... (reszta jak w sekcji 14)
    likes: Mapped[list[BookLike]] = relationship(
        back_populates="book",
        cascade="all, delete-orphan",
    )
    liked_by: Mapped[list["Member"]] = relationship(
        secondary="book_like",
        viewonly=True,          # OBOWIĄZKOWE przy Association Object
    )


class Member(Base):
    # ... (reszta jak w sekcji 14)
    likes: Mapped[list[BookLike]] = relationship(back_populates="member")
```

**1. Modele** — powyżej. Zwróć uwagę na `viewonly=True` na `liked_by`: bez tego dwa mechanizmy pisałyby do `book_like`.

**2. Top 10 najczęściej polubionych książek:**

```python
stmt = (
    select(Book, sa_func.count(BookLike.member_id).label("like_count"))
    .join(BookLike, BookLike.book_id == Book.id)
    .group_by(Book.id)
    .order_by(sa_func.count(BookLike.member_id).desc())
    .limit(10)
)

with Session(engine) as session:
    for book, like_count in session.execute(stmt):
        print(f"{book.title}: {like_count} polubień")
```

**3. Czy czytelnik polubił książkę — jedno zapytanie, zero obiektów:**

```python
from sqlalchemy import exists

stmt = select(
    exists().where(
        (BookLike.book_id == 7) & (BookLike.member_id == 3)
    )
)

with Session(engine) as session:
    liked: bool = session.scalar(stmt)
    print(liked)     # True / False
```

**Dlaczego `exists()` jest tu lepsze niż `session.get(BookLike, (7, 3))`?** Bo `get` zwraca **cały obiekt** i wstawia go do identity map. Przy sprawdzaniu tysiąca polubień to tysiące obiektów w pamięci, których nigdy nie użyjesz. `exists()` zwraca jedną wartość logiczną i nie tworzy niczego w sesji.

**Pułapka:** `Book.likes` ma `cascade="all, delete-orphan"`, ale `Member.likes` — nie. Usunięcie czytelnika z polubieniami zgłosi `IntegrityError`, mimo że w `BookLike.member_id` jest `ondelete="CASCADE"` — bo kaskada po stronie Pythona nadpisuje zachowanie bazy dla załadowanych obiektów. Konsekwencja: albo dodaj `passive_deletes=True` na `Member.likes`, albo zaakceptuj, że usunięcie czytelnika wymaga najpierw usunięcia polubień.

</details>

<details>
<summary><b>Zadanie 3 — usuwanie autora: dwa warianty</b></summary>

```python
# examples/09_ex3_delete_variants.py
import time

from sqlalchemy import event, ForeignKey, select
from sqlalchemy.engine import Engine
from sqlalchemy.orm import Session, relationship


# --- Licznik zapytań -----------------------------------------------------
query_count = 0


@event.listens_for(Engine, "before_cursor_execute")
def _count_queries(conn, cursor, statement, parameters, context, executemany):
    global query_count
    query_count += 1


def reset_counter() -> None:
    global query_count
    query_count = 0


# --- WARIANT A: kaskada w Pythonie --------------------------------------
class AuthorA(Base):
    __tablename__ = "author_a"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(120))

    books: Mapped[list["BookA"]] = relationship(
        back_populates="author",
        cascade="all, delete-orphan",
    )


class BookA(Base):
    __tablename__ = "book_a"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    author_id: Mapped[int] = mapped_column(ForeignKey("author_a.id"))

    author: Mapped[AuthorA] = relationship(back_populates="books")


# --- WARIANT B: kaskada w bazie -----------------------------------------
class AuthorB(Base):
    __tablename__ = "author_b"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(120))

    books: Mapped[list["BookB"]] = relationship(
        back_populates="author",
        cascade="all, delete-orphan",
        passive_deletes=True,          # <-- nie ładuj dzieci
    )


class BookB(Base):
    __tablename__ = "book_b"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    author_id: Mapped[int] = mapped_column(
        ForeignKey("author_b.id", ondelete="CASCADE")   # <-- baza usunie
    )

    author: Mapped[AuthorB] = relationship(back_populates="books")
```

Pomiar (500 książek, SQLite w pliku):

```python
def measure(model_author, model_book, label: str) -> None:
    with Session(engine) as session:
        author = model_author(name="Test")
        author.books = [model_book(title=f"Książka {i}") for i in range(500)]
        session.add(author)
        session.commit()
        author_id = author.id

    with Session(engine) as session:
        author = session.get(model_author, author_id)
        reset_counter()
        start = time.perf_counter()
        session.delete(author)
        session.commit()
        elapsed = time.perf_counter() - start

    print(f"{label}: {query_count} zapytań, {elapsed * 1000:.1f} ms")


measure(AuthorA, BookA, "A (kaskada w Pythonie)")
measure(AuthorB, BookB, "B (kaskada w bazie) ")
```

**Typowe wyniki (SQLite, 500 książek):**

| Wariant | Liczba zapytań | Czas |
|---|---|---|
| A — kaskada w Pythonie | ~502 (`SELECT` + 500 `DELETE` + `DELETE`) | ~180 ms |
| B — `ON DELETE CASCADE` | 2 (`SELECT author` + `DELETE author`) | ~4 ms |

**Dlaczego różnica rośnie z liczbą książek?** Wariant A jest liniowy w liczbie dzieci: $O(N)$ rund do bazy, każda z własnym narzutem (serializacja, planowanie, transakcja, fsync na SQLite). Wariant B jest stały: $O(1)$ rund, a baza usuwa wiersze po indeksie klucza obcego wewnętrznie.

**Kiedy mimo wszystko wybrać wariant A?**

1. **Audyt.** Chcesz zapisać w `audit_log`, kto usunął którą książkę — musisz mieć obiekty w rękach.
2. **Logika warunkowa.** Nie wszystkie książki mają zostać usunięte: część przenosisz do innego autora, część archiwizujesz.
3. **Brak kontroli nad schematem.** Pracujesz z bazą, do której nie możesz dodać `ON DELETE CASCADE` (legacy, polityka DBA).
4. **Zdarzenia aplikacyjne.** Chcesz wysłać zdarzenie do kolejki (np. „unieważnij rezerwacje”) dla każdej usuwanej książki.
5. **Brak migracji.** Wariant B wymaga migracji Alembic zmieniającej klucz obcy — wariant A nie.

**Pułapka:** mieszanie obu wariantów. Jeśli masz `ondelete="CASCADE"` **i** kaskadę w Pythonie **bez** `passive_deletes=True`, to SQLAlchemy i tak załaduje dzieci, a baza i tak je usunie — czyli zapłacisz za jedno i drugie. Reguła: `ondelete="CASCADE"` **zawsze** łącz z `passive_deletes=True`.

</details>

---

## Najczęstsze błędy i jak je czytać

| Komunikat | Przyczyna | Naprawa |
|---|---|---|
| `AmbiguousForeignKeysError: Could not determine join condition ... there are multiple foreign key paths linking the tables` | Dwie lub więcej kolumn FK między tymi samymi tabelami, brak `foreign_keys` | Dodaj `foreign_keys="Book.author_id"` (string) albo `foreign_keys=[author_id]` po obu stronach relacji |
| `NoForeignKeysError: Could not determine join condition ... there are no foreign keys linking these tables` | Brak `ForeignKey` na kolumnie łączącej, brak `primaryjoin` | Dodaj `ForeignKey(...)` do kolumny albo jawny `primaryjoin="Book.author_id == Author.id"` |
| `ArgumentError: For many-to-one relationship Book.author, delete-orphan cascade is normally configured only on the "one" side ...` | `delete-orphan` ustawione po stronie „wiele” | Przenieś kaskadę na stronę „jeden” (`Author.books`) |
| `ArgumentError: relationship 'Book.categories' does not reference a mapped attribute` | `back_populates` wskazuje nazwę, której nie ma po drugiej stronie | Sprawdź, jak nazywa się atrybut po drugiej stronie relacji (np. `books`, nie `categories`) |
| `ArgumentError: Mapper ... has no property 'x'` przy `back_populates` | Literówka w nazwie atrybutu albo druga strona nie jest jeszcze zmapowana | Porównaj nazwy atrybutów po obu stronach |
| `SAWarning: relationship 'X.y' will copy column ... which conflicts with relationship(s) ...` | Dwie relacje mogą pisać do tej samej kolumny FK | Dodaj `foreign_keys`, `back_populates`, albo `viewonly=True` na jednej z relacji |
| `InvalidRequestError: The unique() method must be invoked on this Result, as it contains results that include joined eager loads against collections` | `joinedload` na kolekcji bez `.unique()` | Dodaj `.unique()` po `session.scalars(...)` — albo użyj `selectinload` |
| `DetachedInstanceError: Parent instance <Book> is not bound to a Session; lazy load operation of attribute 'author' cannot proceed` | Obiekt poza sesją próbuje doładować relację leniwie | Załaduj relację wcześniej (`selectinload`/`joinedload`) albo trzymaj obiekt w otwartej sesji; szczegóły w module 11 |
| `IntegrityError: NOT NULL constraint failed: book.author_id` | `remove()` z kolekcji przy kolumnie FK `NOT NULL` i bez `delete-orphan` | Dodaj `delete-orphan`, zmień kolumnę na nullable, albo przenieś obiekt zamiast usuwać |
| `IntegrityError: FOREIGN KEY constraint failed` przy `session.delete(...)` | Istnieją wiersze odwołujące się do usuwanego obiektu, brak kaskady | Dodaj `cascade="all, delete-orphan"` w Pythonie albo `ondelete` w bazie + `passive_deletes` |
| `FlushError: New instance <Review> with identity key (<class 'Review'>, (1, 1), None) conflicts with persistent instance` | Dwa obiekty z tym samym kluczem złożonym w jednej sesji | Sprawdź, czy nie tworzysz duplikatu; użyj `session.get(Review, (book_id, member_id))` przed `add` |
| `InvalidRequestError: 'Book.author' is not available due to lazy='raise'` | Relacja z `lazy="raise"` nie została załadowana jawnie | Dodaj `selectinload(Book.author)` / `joinedload(Book.author)` w zapytaniu |
| `ArgumentError: This relationship is not a single parent, ... single_parent=True` | `delete-orphan` na relacji, gdzie obiekt może mieć wielu rodziców (N:M, 1:1 bez `uselist=False`) | Dodaj `single_parent=True` przy 1:1, albo zrezygnuj z `delete-orphan` przy N:M |
| `InvalidRequestError: Object '<Book>' is already attached to session '1' (this is '2')` | Ten sam obiekt Pythona trafia do dwóch sesji | Użyj `session.merge()` albo pracuj na kopiach; nie współdziel obiektów między sesjami |

*(Dokładne brzmienie komunikatów może się nieznacznie różnić między wydaniami 2.0.x — SQLAlchemy dopisuje na końcu link do `sqlalche.me`, który prowadzi do opisu konkretnego błędu.)*

---

## Słowniczek modułu

| Termin (EN) | Polski odpowiednik | Wyjaśnienie |
|---|---|---|
| association object | obiekt asocjacyjny | Klasa mapowana na tabelę pośrednią N:M, gdy ta tabela ma własne kolumny |
| association table | tabela asocjacyjna | Trzecia tabela łącząca dwie tabele w relacji wiele-do-wielu |
| back_populates | — | Nazwa atrybutu po drugiej stronie relacji; jawnie łączy dwie strony |
| backref | — | Skrót tworzący drugą stronę relacji automatycznie; relikt 1.x |
| cascade | kaskada | Reguła propagowania operacji (zapis, usunięcie) na obiekty powiązane |
| delete-orphan | usuń sierotę | Kaskada: obiekt odłączony od rodzica jest usuwany |
| eager loading | ładowanie zachłanne | Pobranie relacji od razu, w tym samym lub dodatkowym zapytaniu |
| foreign key | klucz obcy | Kolumna wskazująca wiersz w innej tabeli |
| identity map | mapa tożsamości | Rejestr sesji: jeden wiersz = jeden obiekt Pythona |
| lazy loading | ładowanie leniwe | Pobranie relacji dopiero przy pierwszym dostępie do atrybutu |
| many-to-many (N:M) | wiele-do-wielu | Relacja, w której obie strony mają kolekcje |
| many-to-one (N:1) | wiele-do-jednego | Strona z pojedynczym obiektem i kluczem obcym |
| one-to-many (1:N) | jeden-do-wielu | Strona z kolekcją obiektów |
| one-to-one (1:1) | jeden-do-jednego | 1:N z unikalnym kluczem obcym i `uselist=False` |
| orphan | sierota | Obiekt, który stracił rodzica w relacji z `delete-orphan` |
| passive_deletes | pasywne usuwanie | SQLAlchemy nie ładuje dzieci, ufa `ON DELETE` w bazie |
| primaryjoin | warunek łączenia | Ręcznie podany warunek `ON` dla relacji |
| remote_side | strona odległa | Wskazuje, która kolumna jest „rodzicem” w relacji samo-referencyjnej |
| secondary | tabela pośrednia | Tabela używana do przejścia w relacji N:M |
| self-referential | samo-referencyjna | Relacja, w której tabela wskazuje sama na siebie |
| selectinload | — | Strategia ładowania: osobne zapytanie z `IN (...)` |
| joinedload | — | Strategia ładowania: `LEFT OUTER JOIN` w tym samym zapytaniu |
| single_parent | jeden rodzic | Wymuszenie, że obiekt ma tylko jednego właściciela; wymagane przy `delete-orphan` w 1:1 |
| synonym | synonim | Alias atrybutu działający zarówno w Pythonie, jak i w zapytaniach |
| uselist | — | `False` wymusza pojedynczy obiekt zamiast kolekcji |
| viewonly | tylko odczyt | Relacja, której SQLAlchemy nie zapisuje |

---

## Dalsze czytanie

- **Podstawowe wzorce relacji** (1:N, N:1, 1:1, N:M, self-referential): <https://docs.sqlalchemy.org/en/20/orm/basic_relationships.html>
- **Pełne API `relationship()`** (wszystkie parametry, opisane po kolei): <https://docs.sqlalchemy.org/en/20/orm/relationship_api.html>
- **Kaskady — kompletny przewodnik po `cascade` i `passive_deletes`**: <https://docs.sqlalchemy.org/en/20/orm/cascades.html>
- **Konfiguracja warunków łączenia** (`foreign_keys`, `primaryjoin`, `remote_side`): <https://docs.sqlalchemy.org/en/20/orm/join_conditions.html>
- **Relacje samo-referencyjne — osobny rozdział z przykładami drzew**: <https://docs.sqlalchemy.org/en/20/orm/self_referential.html>
- **Strategie ładowania relacji** (zapowiedź modułu 11): <https://docs.sqlalchemy.org/en/20/orm/queryguide/relationships.html>
- **`association_proxy` — wygodny dostęp przez tabelę pośrednią**: <https://docs.sqlalchemy.org/en/20/orm/extensions/associationproxy.html>
- **Kolekcje i ich zachowanie** (`append`, `remove`, `clear`, zdarzenia kolekcji): <https://docs.sqlalchemy.org/en/20/orm/collections.html>
- **Wersja 2.1 tych samych stron** (dla porównania, m.in. `selectinload.chunksize`): <https://docs.sqlalchemy.org/en/21/orm/basic_relationships.html>

---

## Co dalej

Masz już pełny graf modeli: 1:N, 1:1, N:M, Association Object, samo-referencje i kaskady. Umiesz powiedzieć, **jak** dane są powiązane. Nie umiesz jeszcze powiedzieć, **co z nich wyciągnąć** — a to jest zadanie zapytań. W module 10 przejdziemy od relacji do `select()` w wersji ORM-owej: jak łączyć encje, jak zwracać krotki, jak pisać ORM-owe `UPDATE` i `DELETE`, i jak uniknąć pułapek typu `synchronize_session`.

➡️ **[`10_zapytania_orm.md`](10_zapytania_orm.md)**

<!-- koniec modułu 09 -->