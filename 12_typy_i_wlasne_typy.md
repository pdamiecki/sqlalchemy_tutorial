# Moduł 12 — Typy danych i własne typy

Typ kolumny to najbardziej niedoceniana decyzja projektowa w całym modelowaniu danych. Wygląda niewinnie — jedno słowo w `mapped_column()` — a potem przez dwa lata decyduje o tym, czy kwoty się zgadzają, czy strefy czasowe da się porównać, czy zapytanie po kluczu JSON wykorzysta indeks, i czy migracja na PostgreSQL będzie bolesna. W tym module przejdziemy od przeglądu wbudowanych typów, przez konkretne pułapki, aż do momentu, w którym sami napiszemy typ, jakiego SQLAlchemy nie ma — i zrobimy to poprawnie, z pamięcią podręczną zapytań i obsługą różnych dialektów.

---

**Poziom:** 🟠 zaawansowany
**Czas:** ~150 minut
**Wymagania wstępne:** [`03_metadata_ddl.md`](03_metadata_ddl.md) (typy kolumn w Core), [`07_modele_deklaratywne.md`](07_modele_deklaratywne.md) (`Mapped` + `mapped_column`), [`11_ladowanie_i_n_plus_1.md`](11_ladowanie_i_n_plus_1.md) (kontekst ładowania i wydajności)
**Czego dotyczy ten plik:** sposobu, w jaki pojedyncza wartość Pythona zamienia się w wartość w bazie i z powrotem — oraz tego, jak tę konwersję rozszerzyć własnym kodem.

---

## Spis treści

- [1. Typ kolumny to kontrakt między dwoma światami](#1-typ-kolumny-to-kontrakt-między-dwoma-światami)
- [2. Przegląd typów: Python ↔ SQLAlchemy ↔ SQL](#2-przegląd-typów-python--sqlalchemy--sql)
- [3. Czas i strefy czasowe](#3-czas-i-strefy-czasowe)
- [4. JSON i JSONB](#4-json-i-jsonb)
- [5. Mutowalność: dlaczego zmiana w słowniku znika](#5-mutowalność-dlaczego-zmiana-w-słowniku-znika)
- [6. Enum](#6-enum)
- [7. UUID jako klucz główny](#7-uuid-jako-klucz-główny)
- [8. LargeBinary: pliki w bazie](#8-largebinary-pliki-w-bazie)
- [9. TypeDecorator: własny typ](#9-typedecorator-własny-typ)
- [10. Trzy poziomy walidacji](#10-trzy-poziomy-walidacji)
- [11. Typy specyficzne dla dialektu i przenośność](#11-typy-specyficzne-dla-dialektu-i-przenośność)
- [12. Pełny przykład: profil czytelnika](#12-pełny-przykład-profil-czytelnika)
- [Podsumowanie](#podsumowanie)
- [Ćwiczenia](#ćwiczenia)
- [Najczęstsze błędy i jak je czytać](#najczęstsze-błędy-i-jak-je-czytać)
- [Słowniczek modułu](#słowniczek-modułu)
- [Dalsze czytanie](#dalsze-czytanie)
- [Co dalej](#co-dalej)

---

## 1. Typ kolumny to kontrakt między dwoma światami

Zacznijmy od problemu, zanim pokażemy rozwiązanie.

Python zna obiekty: liczby całkowite, teksty, daty, słowniki, obiekty klas. Baza danych zna własne typy: `INTEGER`, `VARCHAR`, `TIMESTAMP`, `JSONB`. To dwa różne światy z różnymi zestawami możliwości. Python ma `Decimal`, który umie reprezentować dokładnie 0,10; baza PostgreSQL też ma `NUMERIC`, który to potrafi — ale SQLite nie ma osobnego typu i zapisze to jako tekst lub liczbę zmiennoprzecinkową. Python ma `datetime` ze strefą czasową; SQLite nie ma osobnego typu na datę i zapisze ją jako napis.

Ktoś musi pilnować tłumaczenia w obie strony. Ten ktoś to **typ kolumny** (ang. *column type*, *type*).

> 💡 **Analogia — Typ jako celnik na przejściu granicznym**
>
> Wyobraź sobie granicę między dwoma państwami. Po jednej stronie mówi się „obiekt Pythona”, po drugiej — „wartość SQL”. Na granicy stoi celnik, który:
> - przy eksporcie (zapisywaniu) sprawdza dokument obiektu i wystawia go w formacie, jaki uznaje kraj docelowy — to jest `process_bind_param`;
> - przy imporcie (odczycie) bierze to, co przyszło z bazy, i odtwarza pełnoprawny obiekt Pythona — to jest `process_result_value`.
>
> Każdy wbudowany typ SQLAlchemy to taki gotowy, wbudowany celnik. W tym module nauczymy się zatrudniać własnych.

Dlaczego to nie jest trywialne? Bo konwersja w obie strony nie zawsze jest odwracalna i nie zawsze oczywista:

- Zapisujesz `datetime` ze strefą do kolumny, która strefy nie przechowuje. Informacja o strefie zostaje po cichu **zgubiona**. Odczytasz z powrotem inny obiekt, niż zapisałeś.
- Zapisujesz `Decimal("0.10")` do kolumny typu `Float`. Przy odczycie dostajesz `0.1` jako liczbę zmiennoprzecinkową, która nie umie dokładnie reprezentować wielu ułamków dziesiętnych. Po kilku tysiącach operacji bilans przestaje się zgadzać.
- Zapisujesz słownik do kolumny JSON, potem zmieniasz jeden klucz i zapisujesz ponownie. Baza nie dostaje `UPDATE`, bo SQLAlchemy nie zauważył zmiany.

Wszystkie trzy problemy to problemy **typu**, nie problemy zapytania ani sesji. Dlatego ten moduł ma znaczenie.

Warto też od początku rozróżnić trzy rzeczy, które łatwo pomieszać:

| Pytanie | Odpowiada za to |
|---|---|
| Czy Python zamieni `Decimal` na to, co rozumie PostgreSQL? | typ kolumny (np. `Numeric`) |
| Czy baza odrzuci literówkę we wpisie statusu? | ograniczenie `CHECK` albo natywny typ `ENUM` |
| Czy użytkownik zobaczy komunikat „podaj poprawny e-mail”? | walidacja warstwy aplikacji (Pydantic, `@validates`) |

Ten moduł zajmuje się głównie pierwszą kolumną, ale punkt 10 pokaże, gdzie przebiega granica między nimi i dlaczego trzymanie całej walidacji w typie kolumny to zły pomysł.

> 🧠 **Dlaczego tak jest — jedna wartość, trzy warstwy**
>
> Gdy piszesz `reader.email == "a@b.pl"`, SQLAlchemy musi:
> 1. zamienić obiekt `"a@b.pl"` na parametr wiązany (`bind parameter`) właściwego typu — tu działa `bind_processor` typu;
> 2. wysłać `UPDATE ... SET email = %(email)s` przez sterownik bazy (DBAPI);
> 3. przy odczycie wziąć surowy wynik sterownika i zamienić go na obiekt Pythona — tu działa `result_processor`.
>
> Każdy z tych kroków może mieć własną logikę, i to jest miejsce, w którym wkracza `TypeDecorator`.

---

## 2. Przegląd typów: Python ↔ SQLAlchemy ↔ SQL

Poniższa tabela to mapa, do której będziesz wracać. Kolumna „SQLite” i „PostgreSQL” pokazuje, w co typ SQLAlchemy zamienia się na konkretnym dialekcie — bo ta sama deklaracja w kodzie daje różne typy w bazie.

| Python | SQLAlchemy | SQLite | PostgreSQL | Kiedy używać |
|---|---|---|---|---|
| `bool` | `Boolean` | `INTEGER` (0/1) | `BOOLEAN` | flagi, przełączniki (`is_active`, `is_deleted`) |
| `int` | `Integer` | `INTEGER` | `INTEGER` | klucze główne, liczniki do ~2 mld |
| `int` | `BigInteger` | `INTEGER` | `BIGINT` | duże liczniki, klucze przy >2 mld wierszy |
| `int` | `SmallInteger` | `INTEGER` | `SMALLINT` | rzadko; gdy naprawdę liczy się miejsce |
| `float` | `Float` | `REAL` | `DOUBLE PRECISION` | pomiary fizyczne, współrzędne, ML — nigdy pieniądze |
| `Decimal` | `Numeric(18, 2)` | `NUMERIC(18,2)` | `NUMERIC(18,2)` | pieniądze, stawki, wszystko, co musi się zgadzać co do grosza |
| `str` | `String(255)` | `VARCHAR(255)` | `VARCHAR(255)` | krótki tekst o znanym limicie: e-mail, kod pocztowy, ISIN |
| `str` | `Text` | `TEXT` | `TEXT` | długi tekst bez limitu: opis, treść artykułu |
| `str` | `postgresql.CITEXT` | — | `CITEXT` | tekst porównywany bez rozróżniania wielkości liter (e-mail, nazwa użytkownika) |
| `datetime.datetime` | `DateTime` | `DATETIME` (tekst) | `TIMESTAMP WITHOUT TIME ZONE` | znacznik czasu bez odniesienia do strefy |
| `datetime.datetime` | `DateTime(timezone=True)` | `DATETIME` (tekst, bez offsetu!) | `TIMESTAMP WITH TIME ZONE` | prawie zawsze to, czego chcesz |
| `datetime.date` | `Date` | `DATE` | `DATE` | data bez godziny: termin zwrotu, dzień wypłaty |
| `datetime.time` | `Time` | `TIME` | `TIME` | godzina bez daty: godziny otwarcia |
| `datetime.timedelta` | `Interval` | brak natywnego (liczba) | `INTERVAL` | odstępy czasu: długość wypożyczenia |
| `bytes` | `LargeBinary` | `BLOB` | `BYTEA` | małe binaria: hash hasła, klucz publiczny, podpis |
| `dict` / `list` | `JSON` | `TEXT` (lub `JSON`) | `JSON` | elastyczne struktury, których jeszcze nie znamy |
| `dict` / `list` | `postgresql.JSONB` | `JSONB` (🆕 2.1) | `JSONB` | to samo, ale z indeksami i zapytaniami po kluczach |
| `uuid.UUID` | `Uuid` | `CHAR(32)` | `UUID` | klucze publiczne, identyfikatory w URL-ach |
| `enum.Enum` | `Enum(..., native_enum=False)` | `VARCHAR` (+ `CHECK`) | `VARCHAR` (+ `CHECK`) | status, kategoria, typ zdarzenia |
| `enum.Enum` | `Enum(...)` | `VARCHAR` (+ `CHECK`) | natywny typ `CREATE TYPE` | to samo, ale wymuszane przez bazę |
| `list` | `postgresql.ARRAY` | — (użyj `JSON`) | `ARRAY` | tablice liczb lub tekstów |
| `ipaddress` | `postgresql.INET` | — | `INET` | adresy IP w logach i ACL |
| `object` | `PickleType` | `BLOB` | `BYTEA` | ostateczność — nie używaj w produkcji |

> ⚠️ **Pułapka — `String(50)` przy pustej adnotacji**
>
> W stylu 2.0 adnotacja `Mapped[str]` bez parametru mapuje się na `String` **bez długości**, co na PostgreSQL daje `VARCHAR` (bez limitu), a w wielu innych bazach — `VARCHAR(255)` albo wręcz błędnie utworzoną kolumnę. Jeśli naprawdę chcesz limit, napisz go jawnie: `mapped_column(String(50))`. Jeśli nie chcesz limitu, napisz jawnie `Text`. Niejawne „coś pomiędzy” to źródło rozjazdu między środowiskami.

Trzy rekomendacje, które warto zapamiętać z tej tabeli od razu, bo są najczęstszą przyczyną problemów produkcyjnych:

1. **Pieniądze nigdy w `Float`.** Zawsze `Numeric(precision, scale)` — i to nie z ostrożności, ale z powodu arytmetyki binarnej. `Decimal("0.1") + Decimal("0.2")` daje dokładnie `Decimal("0.3")`; ich odpowiedniki `float` dają `0.30000000000000004`.
2. **`DateTime(timezone=True)` domyślnie.** Kolumna czasu bez strefy wymaga od każdego czytelnika kodu zgadywania, w jakiej strefie jest wartość. Zobacz punkt 3.
3. **`String` vs `Text` to nie „krótki vs długi tekst”.** To decyzja o tym, czy chcesz, żeby baza odrzucała zbyt długie wartości, czy nie. `String(10)` na kodzie pocztowym to darmowy `CHECK`, który nie wymaga migracji przy zmianie reguły walidacji.

---

## 3. Czas i strefy czasowe

To temat, który kosztował więcej błędów produkcyjnych niż jakikolwiek inny w tej tabeli, więc poświęcimy mu osobny rozdział.

### 3.1. Dwa rodzaje `datetime`

Python zna dwa rodzaje obiektów `datetime`:

- **naiwny** (ang. *naive*) — `datetime(2026, 9, 19, 12, 0)`. Nie wie, czy to południe w Warszawie, w Tokio czy na księżycu. Jego atrybut `tzinfo` to `None`.
- **świadomy** (ang. *aware*) — `datetime(2026, 9, 19, 12, 0, tzinfo=timezone.utc)`. Wie dokładnie, w której strefie jest wyrażony.

> 💡 **Analogia — Zegarek bez nazwy miasta**
>
> „Spotkajmy się o 15:00” to zdanie naiwne. Jeśli druga osoba jest w innym mieście, nie wie, o której się spotkać. „Spotkajmy się o 15:00 czasu warszawskiego” to zdanie świadome — jednoznaczne niezależnie od tego, gdzie jesteś. Baza danych ma dokładnie ten sam problem. `TIMESTAMP WITH TIME ZONE` to dopisanie nazwy miasta do godziny.

Porównywanie obiektu naiwnego ze świadomym w Pythonie kończy się wyjątkiem `TypeError: can't compare offset-naive and offset-aware datetimes`. Porównywanie dwóch naiwnych obiektów, które pochodzą z różnych stref, nie kończy się niczym — po prostu daje błędny wynik, cicho.

### 3.2. Zasada: w bazie zawsze UTC

Jedna reguła, której warto się trzymać bez wyjątków: **w bazie przechowujemy czas w UTC, a strefę użytkownika stosujemy dopiero przy wyświetlaniu.** Powody:

- UTC nie ma czasu letniego, więc nie ma niejednoznacznych godzin (raz w roku godzina 2:30 istnieje dwa razy w strefach z DST).
- Porównania i sortowania między rekordami z różnych stref są poprawne.
- Logi z różnych serwerów można ustawić w jednej linii czasu.

Kolumna mówi bazie, że wartość jest świadoma, ale nie zapisuje strefy użytkownika — i to jest w porządku, bo strefa użytkownika to dana prezentacyjna, nie dana domenowa.

```python
# examples/12_datetime_columns.py
from datetime import datetime, timezone

from sqlalchemy import DateTime, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class Reader(Base):
    __tablename__ = "reader"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(unique=True)

    # Wartość w bazie jest świadoma strefy.
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
        nullable=False,
    )
    # Aktualizowana przez bazę przy każdym UPDATE.
    updated_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
        onupdate=func.now(),
    )
```

### 3.3. `default` vs `server_default` vs `func.now()`

Trzy sposoby na wypełnienie kolumny wartością domyślną wyglądają podobnie, a robią zupełnie różne rzeczy. To klasyczne miejsce na pomyłkę.

| Mechanizm | Kto generuje wartość | Kiedy trafia do bazy | Widoczna w obiekcie przed `flush`? | Uwagi |
|---|---|---|---|---|
| `default=utcnow` (funkcja Pythona) | proces Pythona | w momencie `INSERT`, jako parametr | tak, po `flush` | zegar aplikacji; przy wielu serwerach mogą się różnić |
| `default=datetime.now(timezone.utc)` (wartość) | Python przy imporcie modułu | ta sama wartość dla wszystkich wierszy! | tak | błąd — data zamrożona w chwili uruchomienia |
| `server_default=func.now()` | serwer bazy | w momencie `INSERT` | nie | zegar jednego autorytetu; wymaga odświeżenia obiektu, by zobaczyć wartość |
| `onupdate=func.now()` | serwer bazy | przy każdym `UPDATE` | nie | działa tylko dla operacji przez ORM/UPDATE; nie przy `executemany` poza ORM-em |

> ⚠️ **Pułapka — `default=datetime.now()` bez nawiasów**
>
> Najczęstszy błąd w tym miejscu: `default=datetime.now` (referencja do funkcji) jest poprawne i wywoła się przy każdym wstawieniu. `default=datetime.now()` (wywołanie) jest błędne — wartość zostanie obliczona raz, w chwili importu modułu, i każde wstawienie użyje tej samej daty. Jeśli w bazie wszystkie rekordy mają identyczny `created_at` co do sekundy, szukaj właśnie tu.

Gdy potrzebujesz wartości od razu po wstawieniu, a używasz `server_default`, musisz albo użyć `returning`, albo odświeżyć obiekt:

```python
# examples/12_returning_created_at.py
from sqlalchemy import insert, select
from sqlalchemy.orm import Session

from examples_models import Reader, engine

with Session(engine) as session:
    # Wariant 1: INSERT ... RETURNING — wartość przychodzi w tym samym zapytaniu.
    result = session.execute(
        insert(Reader).values(email="a@example.com").returning(Reader.created_at)
    )
    created_at = result.scalar_one()
    print("created_at z RETURNING:", created_at)

    # Wariant 2: odświeżenie już istniejącego, wygasłego obiektu.
    reader = session.scalars(select(Reader)).first()
    if reader is not None:
        session.refresh(reader)
        print("created_at po refresh:", reader.created_at)
```

> 🔬 **Pod maską**
>
> ```sql
> -- Wariant 1
> INSERT INTO reader (email) VALUES (%(email)s) RETURNING reader.created_at
>
> -- Wariant 2
> SELECT reader.id, reader.email, reader.created_at, reader.updated_at
> FROM reader WHERE reader.id = %(pk)s
> ```
>
> `server_default=func.now()` w DDL zamienia się na `DEFAULT now()`. Uwaga: dla SQLite `func.now()` da `CURRENT_TIMESTAMP`, który zwraca czas w UTC jako napis — to zachowanie różni się od PostgreSQL, gdzie `now()` zwraca `timestamptz`.

### 3.4. SQLite nie przechowuje stref — i to trzeba zaadresować

SQLite nie ma typu daty. Wszystko jest napisem. SQLAlchemy konwertuje `datetime` do napisu ISO, ale **bez informacji o offsetie**. Odczytany z powrotem obiekt będzie naiwny. Jeśli potem porównasz go ze świadomym `datetime.now(timezone.utc)` — dostaniesz `TypeError` w jednym środowisku, a w innym (gdy zapomnisz porównywać) ciche błędy.

Rozwiązanie: **`TypeDecorator`, który normalizuje czas do UTC po obu stronach.** To pierwszy powód, dla którego warto umieć pisać własne typy — pokażemy go w całości w punkcie 9.3.

> 🆕 **SQLAlchemy 2.1**
>
> W linii 2.1 udoskonalono obsługę typów dat w kontekście deklaratywnym, w tym mapowanie `dataclass` (domyślne wartości nie lądują już w `__dict__` instancji, co wcześniej mogło maskować brakujące pola). Nie zmienia to jednak zasady: strefa czasu to coś, co musisz zaprojektować świadomie.

### 3.5. `Arrow` i `pendulum` — czy warto?

Biblioteki `arrow` i `pendulum` oferują wygodniejsze API do dat niż wbudowany `datetime`. Zalety: lepsze parsowanie, strefy jako pierwsza klasa, operacje typu „początek miesiąca”. Wady w kontekście SQLAlchemy:

- SQLAlchemy nie zna tych typów. Jeśli w modelu masz `Mapped[arrow.Arrow]`, musisz napisać `TypeDecorator`, który konwertuje w obie strony — a to dodatkowy kod, który trzeba utrzymywać i testować.
- `arrow`/`pendulum` jako zależność wchodzi do każdej warstwy aplikacji, także tam, gdzie nie jest potrzebna.

Rekomendacja praktyczna: **w modelach ORM trzymaj wbudowany `datetime` w UTC.** Konwersję i formatowanie strefy użytkownika rób dopiero przy serializacji odpowiedzi (Pydantic, warstwa API). W ten sposób baza i ORM pozostają „głupie i przewidywalne”, a cała wiedza o strefach siedzi w jednym miejscu — albo w typie `TypeDecorator`, albo w schemacie Pydantic.

> 🧪 **Ćwiczenie — Sprawdź, gdzie gubisz strefę**
>
> Napisz krótki skrypt, który zapisuje do SQLite (i do PostgreSQL, jeśli masz) `datetime.now(timezone.utc)`, a potem go odczytuje. Wypisz `value`, `value.tzinfo` i `value.utcoffset()` przed zapisem i po odczycie. Zobacz, w którym momencie `tzinfo` staje się `None`. To doświadczenie utrwali regułę lepiej niż akapit tekstu.

---

## 4. JSON i JSONB

Kolumny JSON pozwalają zapisać w bazie słownik albo listę bez definiowania dla nich osobnych tabel. To potężne narzędzie — i jedno z najczęściej nadużywanych.

### 4.1. Kiedy JSON w bazie ma sens

Dobry wybór:

- **Ustawienia i preferencje użytkownika** — klucze pojawiają się i znikają, nie chcemy migracji dla każdej nowej opcji.
- **Odpowiedzi z zewnętrznych API**, które chcemy zachować w niezmienionej formie (np. surowa odpowiedź z bramki płatności do celów audytowych).
- **Metadane zdarzeń analitycznych** — różne typy zdarzeń mają różne pola.
- **Atrybuty produktu zależne od kategorii** — rower ma „rozmiar ramy”, a narty mają „długość”, i nie chcemy trzydziestu kolumn, z których zawsze 27 jest `NULL`.

Zły wybór:

- **Pola, po których często filtrujesz i łączysz tabele.** Jeśli `preferences["city"]` jest używane w `WHERE` w połowie zapytań, powinno być kolumną.
- **Pola wymagające ograniczeń integralności.** JSON nie ma kluczy obcych ani `NOT NULL` na poziomie klucza.
- **Dane, które mają własną tożsamość i cykl życia** — jeśli „adres dostawy” ma datę utworzenia i jest współdzielony między zamówieniami, to osobna tabela.

> 💡 **Analogia — JSON jako pudełko „różne”**
>
> Kolumna JSON to pudełko z etykietą „różne” w szufladzie z dokumentami. Jest świetne, gdy naprawdę nie wiesz, co w nim będzie, i gdy zajrzysz do niego raz na tydzień. Jest fatalne, gdy co drugi dokument musisz wyjąć właśnie z tego pudełka — za każdym razem wysypujesz całą zawartość na biurko, żeby znaleźć jedną kartkę.

### 4.2. `JSON` a `JSONB`

PostgreSQL ma dwa typy: `JSON` (przechowuje tekst w oryginalnej postaci) i `JSONB` (przechowuje sparsowaną strukturę binarną). Różnice są istotne:

| Cecha | `JSON` | `JSONB` |
|---|---|---|
| Zachowanie kolejności kluczy | tak | nie |
| Zachowanie duplikatów kluczy i formatowania | tak | nie |
| Można indeksować (GIN) i wydajnie filtrować | nie | tak |
| Szybkość zapisu | nieco szybsza | nieco wolniejsza (parsowanie) |
| Szybkość odczytu i zapytań po kluczach | wolniejsza | szybsza |
| Rekomendacja | tylko gdy potrzebujesz dokładnej kopii tekstu | domyślny wybór do danych strukturalnych |

W praktyce: **używaj `JSONB`**. Wyjątek stanowi sytuacja, gdy musisz zachować „surową” odpowiedź z API 1:1, na przykład do weryfikacji podpisu kryptograficznego.

> 🆕 **SQLAlchemy 2.1**
>
> Linia 2.1 rozszerza wsparcie dla typu `JSONB` również na dialekt SQLite. Wcześniej `JSONB` był konstrukcją czysto PostgreSQL-ową, a na SQLite trzeba było używać `JSON`. W 2.0 trzymaj się `JSON` na SQLite i `postgresql.JSONB` na PostgreSQL — albo napisz `TypeDecorator` z `load_dialect_impl` (punkt 9.4), który sam wybierze właściwy typ.

### 4.3. Zapytania po zawartości JSON

PostgreSQL oferuje operatory `->` (zwraca JSON) i `->>` (zwraca tekst). SQLAlchemy wystawia je w sposób przenośny przez indeksowanie kolumny:

```python
# examples/12_json_query.py
from sqlalchemy import JSON, select
from sqlalchemy.dialects import postgresql, sqlite
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class Reader(Base):
    __tablename__ = "reader"

    id: Mapped[int] = mapped_column(primary_key=True)
    preferences: Mapped[dict[str, object]] = mapped_column(JSON, default=dict)


# Ścieżka w JSON-ie: preferences -> 'theme' jako tekst.
stmt = select(Reader.id).where(Reader.preferences["theme"].as_string() == "dark")

# Zobaczmy, jaki SQL wygeneruje każdy dialekt dla TEGO SAMEGO wyrażenia.
print(stmt.compile(dialect=postgresql.dialect(), compile_kwargs={"literal_binds": True}))
print(stmt.compile(dialect=sqlite.dialect(), compile_kwargs={"literal_binds": True}))
```

> 🔬 **Pod maską — dwa dialekty, jedno wyrażenie**
>
> ```sql
> -- PostgreSQL
> SELECT reader.id FROM reader
> WHERE (reader.preferences ->> 'theme') = 'dark'
>
> -- SQLite
> SELECT reader.id FROM reader
> WHERE (json_extract(reader.preferences, '$.theme')) = 'dark'
> ```
>
> Widzisz sens abstrakcji typu: piszesz raz, a SQLAlchemy wybiera właściwą funkcję dialektu. To nie jest magia — to `TypeEngine` z metodami `literal_processor` i operatorami zdefiniowanymi per dialekt.

Przydatne operatory PostgreSQL:

| Operator | Znaczenie | Przykład |
|---|---|---|
| `->` | element lub pole jako JSON | `data -> 'items'` |
| `->>` | element lub pole jako tekst | `data ->> 'status'` |
| `#>` | ścieżka jako JSON | `data #> '{a,b}'` |
| `#>>` | ścieżka jako tekst | `data #>> '{a,b}'` |
| `@>` | zawiera (JSON po lewej zawiera JSON po prawej) | `data @> '{"status":"paid"}'` |
| `<@` | jest zawarte w | `'{"a":1}' <@ data` |
| `?` | klucz istnieje | `data ? 'city'` |
| `?|` | którykolwiek z kluczy istnieje | `data ?| array['a','b']` |
| `?&` | wszystkie klucze istnieją | `data ?& array['a','b']` |

W SQLAlchemy operatory niestandardowe uzyskasz przez `.op()`:

```python
# examples/12_json_ops.py
from sqlalchemy import JSON, select
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class Order(Base):
    __tablename__ = "order"
    id: Mapped[int] = mapped_column(primary_key=True)
    data: Mapped[dict[str, object]] = mapped_column(JSON)


stmt = select(Order.id).where(Order.data.op("@>")({"status": "paid"}))
# SELECT "order".id FROM "order" WHERE "order".data @> %(data_1)s
```

### 4.4. Indeksy na JSON-ie (PostgreSQL)

Indeks GIN (ang. *Generalized Inverted Index*) pozwala zapytaniom typu `@>`, `?` i `?|` działać bez skanowania całej tabeli. W SQLAlchemy tworzymy go przez `Index` z `postgresql_using="gin"`:

```python
# examples/12_json_index.py
from sqlalchemy import JSON, Index
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class Order(Base):
    __tablename__ = "order"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(index=True)
    data: Mapped[dict[str, object]] = mapped_column(JSONB)

    __table_args__ = (
        # Indeks po całym dokumencie — przyspiesza operatory @>, ?, ?|, ?&.
        Index("ix_order_data_gin", "data", postgresql_using="gin"),
        # Indeks po konkretnej ścieżce i typie jsonb_path_ops — mniejszy, ale
        # obsługuje wyłącznie operator @>.
        Index(
            "ix_order_data_path_ops",
            "data",
            postgresql_using="gin",
            postgresql_ops={"data": "jsonb_path_ops"},
        ),
        # Indeks po konkretnym polu, gdy filtrujemy je bardzo często.
        Index("ix_order_data_status", data["status"].astext),
    )
```

> 🔬 **Pod maską**
>
> ```sql
> CREATE INDEX ix_order_data_gin ON "order" USING gin (data);
> CREATE INDEX ix_order_data_path_ops
>     ON "order" USING gin (data jsonb_path_ops);
> CREATE INDEX ix_order_data_status ON "order" ((data ->> 'status'));
> ```
>
> Trzeci indeks to „indeks po wyrażeniu”. Uwaga na pułapkę: **musisz pisać `WHERE` dokładnie tym samym wyrażeniem**, inaczej PostgreSQL nie skorzysta z indeksu. `WHERE data ->> 'status' = 'paid'` pójdzie po indeksie; `WHERE data::text LIKE '%paid%'` już nie.

### 4.5. Zapach projektowy: JSON jako wymówka

Kilka sygnałów, że JSON w bazie jest używany nie tam, gdzie trzeba:

- Ten sam klucz JSON występuje w `WHERE` w większości zapytań do tabeli.
- W kodzie istnieje funkcja, która „waliduje, czy JSON ma odpowiednie klucze” — to znaczy, że to nie jest JSON, to jest tabela.
- Nie da się napisać `JOIN`a po zawartości JSON-a bez rzutowania i kombinowania.
- Do JSON-a trafiają dane, które mają własny `id` i relacje z innymi tabelami.

> 🧪 **Ćwiczenie — Zapach czy nie?**
>
> Zaklasyfikuj jako „JSON” albo „osobna tabela”: (a) preferencje powiadomień użytkownika (e-mail, SMS, push, częstotliwość), (b) pozycje zamówienia, (c) surowa odpowiedź z bramki płatniczej do audytu, (d) adresy dostawy klienta (wielu klientów, wiele adresów), (e) parametry techniczne produktu zależne od kategorii.
>
> *Odpowiedzi:* (a) JSON — klucze są rzadko filtrowane; (b) osobna tabela — mają ilości, ceny, relacje; (c) JSON — niezmienny surowy zapis; (d) osobna tabela — mają tożsamość i współdzielenie; (e) JSON — z natury niejednorodne.

---

## 5. Mutowalność: dlaczego zmiana w słowniku znika

To jeden z najczęściej zgłaszanych „błędów SQLAlchemy”, który w rzeczywistości jest konsekwencją świadomej decyzji projektowej.

### 5.1. Problem

Zapiszmy użytkownika z preferencjami, a potem zmieńmy jedno pole w słowniku — tak, jak to naturalnie robimy w Pythonie:

```python
# examples/12_mutation_problem.py
from sqlalchemy import JSON, create_engine, select
from sqlalchemy.orm import DeclarativeBase, Mapped, Session, mapped_column


class Base(DeclarativeBase):
    pass


class Reader(Base):
    __tablename__ = "reader"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(unique=True)
    # Uwaga: zwykły JSON, bez żadnego rozszerzenia.
    preferences: Mapped[dict[str, object]] = mapped_column(JSON, default=dict)


engine = create_engine("sqlite://", echo=False)
Base.metadata.create_all(engine)

with Session(engine) as session:
    reader = Reader(email="a@example.com", preferences={"theme": "light"})
    session.add(reader)
    session.commit()

    # Zmiana "w miejscu" (in-place) — dokładnie tak, jak lubi Python.
    reader.preferences["theme"] = "dark"

    print("Czy sesja widzi zmianę?", session.is_modified(reader))  # False!
    session.commit()

with Session(engine) as session:
    stored = session.scalars(select(Reader)).one()
    print("Zapisana wartość:", stored.preferences)  # {'theme': 'light'} — zmiana przepadła
```

Wynik: `False` i `{'theme': 'light'}`. Zmiana **nie została zapisana**, mimo że obiekt w pamięci Pythona ma `'dark'`.

### 5.2. Dlaczego tak jest

> 💡 **Analogia — Kartka w koszulce**
>
> Wyobraź sobie, że SQLAlchemy prowadzi dla każdego obiektu teczkę i w niej — kartkę z aktualnym stanem. Kiedy robisz `obj.attr = value`, SQLAlchemy dostaje komunikat „ktoś podmienił kartkę” i zapisuje, że coś się zmieniło. Ale kiedy robisz `obj.preferences["theme"] = "dark"`, podmieniasz tylko treść w środku kartki. Teczka nie dostaje żadnego powiadomienia. Nikt nie wie, że kartka wygląda inaczej.
>
> `MutableDict` to taka plastikowa koszulka, która sama informuje teczkę: „ktoś edytował moją zawartość”.

Mechanizm techniczny: SQLAlchemy instrumentuje dostęp do **atrybutów** (to te `instrumented attribute` z modułu 7), ale nie instrumentuje wnętrza obiektów, które przechowujesz. Liczba, napis, data — to wartości niezmienne, więc `=` zawsze je podmienia. Słownik i lista są mutowalne, więc `[]` i `[] =` to operacje na tym samym obiekcie, do którego SQLAlchemy nie ma wglądu.

### 5.3. Rozwiązanie pierwsze: `MutableDict` / `MutableList` / `MutableSet`

Rozszerzenie `sqlalchemy.ext.mutable` dostarcza typy opakowujące, które informują sesję o każdej zmianie wnętrza.

```python
# examples/12_mutable_json.py
from sqlalchemy import JSON, create_engine, select
from sqlalchemy.ext.mutable import MutableDict
from sqlalchemy.orm import DeclarativeBase, Mapped, Session, mapped_column


class Base(DeclarativeBase):
    pass


class Reader(Base):
    __tablename__ = "reader"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(unique=True)

    # MutableDict.as_mutable(JSON) opakowuje JSON w słownik,
    # który powiadamia sesję o zmianach struktury.
    preferences: Mapped[dict[str, object]] = mapped_column(
        MutableDict.as_mutable(JSON),
        default=dict,
    )


engine = create_engine("sqlite://", echo=True)
Base.metadata.create_all(engine)

with Session(engine) as session:
    reader = Reader(email="a@example.com", preferences={"theme": "light"})
    session.add(reader)
    session.commit()

    # Teraz zmiana w miejscu JEST wykrywana.
    reader.preferences["theme"] = "dark"
    reader.preferences["locale"] = "pl"
    print("Czy sesja widzi zmianę?", session.is_modified(reader))  # True
    session.commit()

with Session(engine) as session:
    stored = session.scalars(select(Reader)).one()
    print("Zapisana wartość:", stored.preferences)
```

> 🔬 **Pod maską**
>
> ```sql
> -- INSERT przy pierwszym commicie
> INSERT INTO reader (email, preferences) VALUES (?, ?)
> -- UPDATE przy drugim commicie — to jest to, czego brakowało
> UPDATE reader SET preferences = ? WHERE reader.id = ?
> ```
>
> Widzisz w logu `echo=True`, że przy drugim `commit()` pojawia się `UPDATE`. W wersji bez `MutableDict` — nie pojawił się żaden SQL. To najlepszy sposób, żeby przekonać samego siebie, że problem jest realny.

`MutableList` i `MutableSet` działają analogicznie:

```python
# examples/12_mutable_list.py
from sqlalchemy import JSON
from sqlalchemy.ext.mutable import MutableList
from sqlalchemy.orm import Mapped, mapped_column


class Report(Base):
    __tablename__ = "report"

    id: Mapped[int] = mapped_column(primary_key=True)
    metrics: Mapped[list[str]] = mapped_column(
        MutableList.as_mutable(JSON), default=list
    )
    tags: Mapped[list[str]] = mapped_column(MutableList.as_mutable(JSON), default=list)
```

> ⚠️ **Pułapka — `MutableDict` na zagnieżdżonym słowniku**
>
> `MutableDict` wykrywa zmiany **na najwyższym poziomie**. Jeżeli masz strukturę `{"notifications": {"email": True}}` i wykonasz `prefs["notifications"]["email"] = False`, `MutableDict` tego **nie zauważy** — wewnętrzny słownik nie jest już opakowany. Rozwiązania: (a) spłaszczyć strukturę, (b) po każdej głębokiej zmianie wywołać `flag_modified`, (c) użyć własnego `TypeDecorator` z rekurencyjnym opakowaniem (trudniejsze, ale możliwe).

### 5.4. Rozwiązanie drugie: `flag_modified`

Gdy z jakiegoś powodu nie możesz użyć `Mutable*` (na przykład odczytujesz dane, których strukturę kontroluje API zewnętrzne), powiedz sesji wprost, że atrybut się zmienił:

```python
# examples/12_flag_modified.py
from sqlalchemy.orm import Session
from sqlalchemy.orm.attributes import flag_modified

from examples_models import Reader, engine

with Session(engine) as session:
    reader = session.get(Reader, 1)
    assert reader is not None

    reader.preferences["theme"] = "dark"
    # Wymuś oznaczenie atrybutu jako zmienionego.
    flag_modified(reader, "preferences")
    session.commit()
```

`flag_modified` to narzędzie „młotek” — działa, ale trzeba pamiętać o jego wywołaniu w każdym miejscu. `MutableDict` jest bezpieczniejszy, bo działa automatycznie, i dlatego jest domyślnym wyborem.

### 5.5. Rozwiązanie trzecie: nie mutować, tylko podmieniać

Najprostsza i najbardziej „funkcyjna” strategia: **zamiast modyfikować, twórz nowy obiekt.**

```python
# examples/12_replace_instead_of_mutate.py
reader.preferences = {**reader.preferences, "theme": "dark"}
```

Przypisanie do atrybutu jest zawsze wykrywane, więc nie potrzebujesz ani `MutableDict`, ani `flag_modified`. Wadą jest tworzenie nowego obiektu przy każdej zmianie i utrata tożsamości referencji — ale w wielu projektach to najbardziej czytelne rozwiązanie.

> 🧪 **Ćwiczenie — Pokaż różnicę**
>
> Zmień w skrypcie z punktu 5.1 jeden element: zamiast `reader.preferences["theme"] = "dark"` wpisz `reader.preferences = {**reader.preferences, "theme": "dark"}`. Uruchom z `echo=True` i sprawdź, czy pojawi się `UPDATE`. Następnie przywróć wersję z `MutableDict` i sprawdź, że efekt jest identyczny.

---

## 6. Enum

`Enum` to typ reprezentujący skończony zbiór dopuszczalnych wartości: status zamówienia, typ zdarzenia, kategoria. W bazie danych można go zrealizować na trzy sposoby i każdy ma inne konsekwencje.

### 6.1. Trzy realizacje

| Realizacja | DDL | Zalety | Wady |
|---|---|---|---|
| `VARCHAR` bez ograniczeń | `VARCHAR(20)` | najprostsza, zero migracji przy nowych wartościach | literówka wchodzi bez oporu |
| `VARCHAR` + `CHECK` | `VARCHAR(20) CHECK (status IN ('a','b'))` | baza wymusza poprawność, przenośne | rozszerzenie zbioru wymaga migracji |
| natywny `ENUM` | `CREATE TYPE ... AS ENUM (...)` | najbardziej „typowo”, oszczędne miejsce | migracja bolesna (`.ALTER TYPE`), trudne wycofywanie, kłopoty z ORM |

SQLAlchemy domyślnie używa drugiej opcji na bazach, które nie mają natywnego `ENUM`, i natywnego `ENUM` na PostgreSQL — chyba że jawnie wyłączysz `native_enum`.

```python
# examples/12_enum_model.py
import enum

from sqlalchemy import Enum
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class ReaderStatus(enum.Enum):
    ACTIVE = "active"
    BLOCKED = "blocked"
    PENDING = "pending"


class Reader(Base):
    __tablename__ = "reader"

    id: Mapped[int] = mapped_column(primary_key=True)

    status: Mapped[ReaderStatus] = mapped_column(
        Enum(
            ReaderStatus,
            native_enum=False,
            create_constraint=True,
            length=20,
            values_callable=lambda e: [m.value for m in e],
            validate_strings=True,
            name="ck_reader_status",
        ),
        default=ReaderStatus.PENDING,
        nullable=False,
    )
```

Zatrzymajmy się nad tymi parametrami, bo każdy z nich odpowiada za konkretny problem:

- **`native_enum=False`** — wymusza `VARCHAR + CHECK` nawet na PostgreSQL. Nasza rekomendacja dla większości projektów: brak natywnego `ENUM` jest mniejszym złem niż operacje `ALTER TYPE` na produkcji.
- **`create_constraint=True`** — **bardzo ważne**: od SQLAlchemy 1.4 domyślna wartość tego parametru to `False`. Bez niego dostajesz goły `VARCHAR` **bez `CHECK`** i żadnej ochrony przed literówką. Jeśli chcesz ograniczenie, musisz je włączyć jawnie.
- **`length=20`** — długość kolumny `VARCHAR`. Bez tego SQLAlchemy wyliczy ją z najdłuższej wartości, co brzmi sprytnie, ale przy dodaniu dłuższej wartości w przyszłości kolumna okaże się za krótka.
- **`values_callable=lambda e: [m.value for m in e]`** — domyślnie SQLAlchemy zapisuje **nazwy** elementów enuma (`"ACTIVE"`, `"PENDING"`), a nie ich wartości. Jeśli w kodzie masz `ReaderStatus.ACTIVE = "active"` i w bazie chcesz `"active"` (małymi literami, w formacie przyjaznym API), ten `values_callable` jest niezbędny.
- **`validate_strings=True`** — przy operacjach na surowych napisach (np. `where(Reader.status == "nieznany")`) SQLAlchemy podniesie `LookupError` zamiast po cichu wygenerować zapytanie zwracające zero wierszy.

> 🔬 **Pod maską**
>
> ```sql
> CREATE TABLE reader (
>     id INTEGER NOT NULL,
>     status VARCHAR(20) DEFAULT 'pending' NOT NULL,
>     PRIMARY KEY (id),
>     CONSTRAINT ck_reader_status CHECK (status IN ('active', 'blocked', 'pending'))
> );
> ```
>
> Zwróć uwagę, że `DEFAULT` ma wartość `'pending'` — czyli wartość (`value`), nie nazwę (`name`). To skutek `values_callable`. Bez niego zobaczyłbyś `DEFAULT 'PENDING'`.

> ⚠️ **Pułapka — `Enum` i migracje**
>
> Dodanie nowej wartości do enuma w kodzie **nie zmienia automatycznie ograniczenia `CHECK` w bazie**. Bez migracji baza będzie odrzucać nową wartość błędem `IntegrityError`. Jeśli używasz natywnego `ENUM` na PostgreSQL, potrzebujesz `ALTER TYPE ... ADD VALUE`, który w starszych wersjach nie działał w transakcji. Przy `native_enum=False` potrzebujesz `ALTER TABLE ... DROP CONSTRAINT` i `ADD CONSTRAINT` z nową listą — prościej i bezpieczniej. Moduł [`16_alembic_migracje.md`](16_alembic_migracje.md) pokazuje, jak to zautomatyzować.

### 6.2. Alternatywa: tabela słownikowa („living enum”)

Zamiast wpisywać dopuszczalne wartości w `CHECK`, można trzymać je w osobnej tabeli:

```python
# examples/12_lookup_table.py
from sqlalchemy import ForeignKey, String, select
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    pass


class StatusRef(Base):
    """Tabela słownikowa: dopuszczalne wartości statusu."""

    __tablename__ = "status_ref"

    code: Mapped[str] = mapped_column(String(20), primary_key=True)
    label: Mapped[str] = mapped_column(String(100))
    sort_order: Mapped[int] = mapped_column(default=0)
    is_terminal: Mapped[bool] = mapped_column(default=False)


class Reader(Base):
    __tablename__ = "reader"

    id: Mapped[int] = mapped_column(primary_key=True)
    status_code: Mapped[str] = mapped_column(
        ForeignKey("status_ref.code"), default="pending"
    )

    status: Mapped[StatusRef] = relationship(lazy="joined")
```

Plusy: nową wartość dodajesz przez `INSERT`, nie przez migrację schematu; masz miejsce na etykiety, tłumaczenia i kolejność wyświetlania; klucz obcy chroni integralność równie dobrze jak `CHECK`. Minusy: jedno więcej złączenie przy każdym odczycie (jeśli potrzebujesz etykiety) i trochę więcej kodu.

| Kryterium | `Enum` | Tabela słownikowa |
|---|---|---|
| Dodanie wartości | migracja schematu | `INSERT` |
| Etykiety i tłumaczenia | osobna tabela albo stałe w kodzie | naturalne miejsce |
| Zapytania w Pythonie | porównanie z `ReaderStatus.ACTIVE` | porównanie z napisami albo `JOIN` |
| Wydajność odczytu | brak złączenia | złączenie (lub cache) |
| Porządek wyświetlania | wymaga sortowania w kodzie | kolumna `sort_order` |
| Bezpieczeństwo typów w kodzie | pełne (`enum.Enum`) | słabsze (napisy) |

Rekomendacja praktyczna: **użyj `Enum` dla wartości, które naprawdę są stałe i niewiele się zmieniają** (np. `BookFormat.HARDCOVER / PAPERBACK / EBOOK`), a **tabeli słownikowej, gdy zbiór wartości jest zarządzany przez biznes** i zmienia się często (kategorie produktów, statusy workflow).

---

## 7. UUID jako klucz główny

Do tej pory używaliśmy kluczy `Integer` z automatycznym numerowaniem (ang. *autoincrement*). Alternatywą jest `Uuid`.

### 7.1. Kiedy UUID, a kiedy `Integer`

| Kryterium | `Integer` (autoinkrementacja) | `Uuid` |
|---|---|---|
| Rozmiar w PostgreSQL | 4 lub 8 bajtów | 16 bajtów |
| Kolejność wstawiania zgodna z indeksem | tak | nie (losowy) |
| Ujawnia liczbę rekordów (`/books/42` → 42. książka) | tak | nie |
| Bezpieczne przy scalaniu baz i generowaniu offline | nie | tak |
| Czytelność w logach i debugowaniu | wysoka | niska |
| Koszt indeksu w B-tree | niski | wyższy przy losowym UUIDv4 |

> 💡 **Analogia — Numer pesel a numer klienta**
>
> Klucz `Integer` to numer porządkowy w kolejce: krótki, wygodny, ale każdy domyśla się, ilu było przed nim. UUID to numer klienta nadany losowo: dłuższy, brzydszy, ale nie zdradza niczego o kolejności ani liczbie osób w systemie.

### 7.2. Natywny `Uuid` i jego zachowanie na dialektach

SQLAlchemy 2.0 udostępnia **generyczny typ `Uuid`**, który sam wybiera realizację zależnie od dialektu:

```python
# examples/12_uuid_model.py
import uuid

from sqlalchemy import Uuid, create_engine
from sqlalchemy.dialects import postgresql, sqlite
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class Book(Base):
    __tablename__ = "book"

    id: Mapped[uuid.UUID] = mapped_column(
        Uuid(as_uuid=True), primary_key=True, default=uuid.uuid4
    )
    title: Mapped[str] = mapped_column(index=True)


from sqlalchemy.schema import CreateTable

print(CreateTable(Book.__table__).compile(dialect=postgresql.dialect()))
print(CreateTable(Book.__table__).compile(dialect=sqlite.dialect()))
```

> 🔬 **Pod maską**
>
> ```sql
> -- PostgreSQL
> CREATE TABLE book (
>     id UUID NOT NULL,
>     title VARCHAR NOT NULL,
>     PRIMARY KEY (id)
> );
>
> -- SQLite: brak typu UUID, więc SQLAlchemy używa CHAR(32)
> CREATE TABLE book (
>     id CHAR(32) NOT NULL,
>     title VARCHAR NOT NULL,
>     PRIMARY KEY (id)
> );
> ```
>
> Wariant `Uuid(as_uuid=False)` przechowywałby wartość jako `str`. Dla spójności z adnotacją `Mapped[uuid.UUID]` używaj `as_uuid=True` (jest to wartość domyślna).

### 7.3. Problem losowego UUIDv4 w indeksach

Losowy UUIDv4 w kluczu głównym powoduje, że kolejne wstawiane wiersze lądują w **losowych miejscach drzewa B-tree** indeksu. Skutki:

- Wstawianie wymaga częstszego dzielenia stron indeksu (ang. *page split*) i rozgrzewa cache mniej efektywnie.
- Przy dużej tabeli (dziesiątki milionów wierszy) koszt wstawiania rośnie w porównaniu do sekwencyjnego `BIGINT`.

Rozwiązania:

1. **Zostaw `BigInteger` jako klucz techniczny i dodaj `Uuid` jako kolumnę publiczną.** Masz szybkie wstawianie i nieujawniający identyfikator. To najczęstszy wzorzec produkcyjny.
2. **Użyj UUIDv7 albo ULID**, które są sortowalne czasowo (prefiks z timestampem) — zachowują większość zalet UUID i większość zalet sekwencyjnego klucza. SQLAlchemy nie ma tego typu wbudowanego, ale to doskonały kandydat na `TypeDecorator` (patrz punkt 9).

```python
# examples/12_uuid_plus_int.py
import uuid

from sqlalchemy import BigInteger, Uuid
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class Order(Base):
    __tablename__ = "order"

    # Wewnętrzny, szybki klucz techniczny — nigdy nie opuszcza aplikacji.
    id: Mapped[int] = mapped_column(BigInteger, primary_key=True, autoincrement=True)
    # Publiczny identyfikator — pojawia się w URL-ach, API, e-mailach.
    public_id: Mapped[uuid.UUID] = mapped_column(
        Uuid(as_uuid=True), unique=True, default=uuid.uuid4, index=True
    )
```

> 🧪 **Ćwiczenie — Ile ujawnia twój klucz?**
>
> Otwórz dokumentację dowolnego publicznego API (np. GitHub, Stripe) i sprawdź, czy identyfikatory zasobów wyglądają jak liczby sekwencyjne, jak UUID, czy jak prefiksowane identyfikatory (`cus_...`, `pi_...`). Zastanów się, dlaczego duże serwisy prawie zawsze wybierają trzecią opcję.

---

## 8. LargeBinary: pliki w bazie

`LargeBinary` mapuje `bytes` Pythona na `BLOB` w SQLite i `BYTEA` w PostgreSQL. Nazwa brzmi zachęcająco — „duże dane binarne” — ale rzadko chcemy tam wpychać duże pliki.

### 8.1. Kiedy `LargeBinary` ma sens

- **Hash hasła** (`bcrypt`, `argon2`) — 60–100 bajtów, idealne.
- **Klucz publiczny lub mały certyfikat** — setki bajtów do kilku KB.
- **Podpis kryptograficzny** — dziesiątki bajtów.
- **Mały, niezmienny plik, który jest częścią rekordu** — np. wygenerowana miniatura 20 KB, którą chcesz trzymać razem z rekordem, żeby zachować atomowość.

### 8.2. Kiedy zdecydowanie nie

- Zdjęcia, wideo, PDF-y w rozmiarach megabajtowych.
- Cokolwiek, co ma być serwowane przez CDN.
- Cokolwiek, co ma być strumieniowane i cache'owane na brzegu.

Powody: kopie zapasowe bazy puchną (i przestają mieścić się w oknie backupu), `SELECT *` nieopatrznie wciąga megabajty do pamięci, migracje stają się wolne, a replikacja bazy zaczyna przenosić dane, które nigdy nie powinny tam trafić.

> 💡 **Analogia — Sejf a magazyn**
>
> Baza danych to sejf, nie magazyn. Trzymasz w sejfie rzeczy, których integralność ma być chroniona razem z resztą dokumentów. Meble trzymasz w magazynie i przechowujesz tylko numer kwitu. Jeśli w sejfie zaczniesz trzymać meble, przestaniesz się w nim mieścić i przestaniesz go otwierać na czas.

Wzorzec: plik trafia do magazynu obiektowego (S3, GCS, MinIO, Azure Blob), a w bazie zapisujesz **klucz obiektu, rozmiar, sumę kontrolną i typ MIME**. Wtedy baza pozostaje lekka, a plik jest dostępny przez szybką sieć dostarczania treści.

```python
# examples/12_file_reference.py
from sqlalchemy import BigInteger, String
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class Attachment(Base):
    """Załącznik przechowywany w magazynie obiektowym, nie w bazie."""

    __tablename__ = "attachment"

    id: Mapped[int] = mapped_column(primary_key=True)
    owner_id: Mapped[int] = mapped_column(index=True)
    storage_key: Mapped[str] = mapped_column(String(255), unique=True)
    original_name: Mapped[str] = mapped_column(String(255))
    content_type: Mapped[str] = mapped_column(String(100))
    size_bytes: Mapped[int] = mapped_column(BigInteger)
    checksum_sha256: Mapped[str] = mapped_column(String(64))
```

### 8.3. Gdy naprawdę musisz przechować binaria: strumieniowanie

Jeśli już musisz trzymać dane binarne w bazie, unikaj wciągania ich w całości do pamięci. PostgreSQL udostępnia duże obiekty (ang. *Large Object*, LOB) z API strumieniowym; SQLAlchemy wystawia je przez `psycopg`:

```python
# examples/12_large_object.py
# Uwaga: działa tylko na PostgreSQL i wymaga sterownika psycopg (3.x).
from sqlalchemy import create_engine, text

engine = create_engine("postgresql+psycopg://user:pass@localhost/db")

with engine.begin() as conn:
    # Rejestracja dużego obiektu z finalizacją przy commicie.
    oid = conn.execute(text("SELECT lo_from_bytea(0, %(data)s)"), {"data": b"..."}).scalar_one()
    print("OID dużego obiektu:", oid)
```

W praktyce zamiast się w to zagłębiać, rozważ magazyn obiektowy. LOB to rozwiązanie dla wąskiej klasy problemów (np. bardzo duże dokumenty wymagające transakcyjnej spójności z rekordem).

---

## 9. TypeDecorator: własny typ

To serce tego modułu. `TypeDecorator` pozwala opakować istniejący typ SQLAlchemy i dodać własną konwersję w obie strony. Elastyczność jest tu bardzo duża: możesz dodać walidację, normalizację, szyfrowanie, reprezentację wartości, a nawet wybrać różny typ dla różnych dialektów.

### 9.1. Anatomia `TypeDecorator`

Klasa dziedzicząca po `TypeDecorator` implementuje (nadpisuje) kilka metod:

| Metoda | Kiedy jest wywoływana | Do czego służy |
|---|---|---|
| `__init__` | przy tworzeniu instancji typu | przekazanie parametrów do typu bazowego |
| `impl` (atrybut klasy) | zawsze | **typ bazowy**, który realizuje faktyczny zapis/odczyt |
| `load_dialect_impl(dialect)` | raz na dialekt w trakcie kompilacji | wybór innego typu bazowego dla konkretnego dialektu |
| `process_bind_param(value, dialect)` | przed wysłaniem wartości do bazy | konwersja Python → wartość zrozumiała dla bazy |
| `process_literal_param(value, dialect)` | przy kompilacji z `literal_binds=True` | to samo, ale dla literału w SQL (np. na potrzeby testów i DDL) |
| `process_result_value(value, dialect)` | po odczytaniu wartości z bazy | konwersja wartość z bazy → obiekt Pythona |
| `python_type` (właściwość) | przez narzędzia i ORM | deklaracja typu Pythona reprezentowanego przez ten typ |
| `cache_ok` (atrybut klasy) | raz przy ładowaniu klasy | czy typ można cache'ować w kompilatorze SQL |

Metody `process_bind_param` i `process_result_value` są symetryczne — pamiętaj o obu. Jeśli zapomnisz o `process_result_value`, zapis będzie działać, a odczyt zwróci surową wartość z bazy i zobaczysz coś dziwnego.

### 9.2. Przykład 1: `Money`

Zbudujmy typ na pieniądze. Wymagania:

- W bazie: `NUMERIC(18, 2)`.
- W Pythonie: `Decimal`.
- Zawsze zaokrąglone do dwóch miejsc, zawsze dokładne.
- Przyjmuje `int`, `float` i `str`, ale konwertuje je na `Decimal` bezpieczną ścieżką (przez `str`, nie przez binarną reprezentację `float`).

```python
# examples/12_money.py
from decimal import ROUND_HALF_UP, Decimal

from sqlalchemy import Numeric, TypeDecorator
from sqlalchemy.engine import Dialect


class Money(TypeDecorator[Decimal]):
    """Kwota pieniężna: w bazie NUMERIC, w Pythonie Decimal zaokrąglony do groszy."""

    # Typ bazowy. Numeric z asdecimal=True zwraca Decimal, nie float.
    impl = Numeric
    # Wymagane, aby SQLAlchemy mogło cache'ować skompilowaną formę zapytań.
    cache_ok = True

    def __init__(self, precision: int = 18, scale: int = 2) -> None:
        # Przekazujemy parametry do typu bazowego Numeric(precision, scale).
        super().__init__(precision=precision, scale=scale)

    # --- Konwersja Python -> baza -----------------------------------------
    def process_bind_param(self, value: object, dialect: Dialect) -> Decimal | None:
        if value is None:
            return None
        # Akceptujemy też int, float i str — ale zawsze konwertujemy przez str,
        # aby nie wprowadzać błędu reprezentacji zmiennoprzecinkowej.
        if isinstance(value, (int, float, str)):
            value = Decimal(str(value))
        if not isinstance(value, Decimal):
            raise TypeError(f"Money oczekuje liczby lub Decimal, dostało {type(value)!r}")
        return value.quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)

    # --- Konwersja baza -> Python -----------------------------------------
    def process_result_value(self, value: object, dialect: Dialect) -> Decimal | None:
        if value is None:
            return None
        if isinstance(value, Decimal):
            return value.quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)
        return Decimal(str(value)).quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)

    # --- Typ Pythona dla narzędzi -----------------------------------------
    @property
    def python_type(self) -> type[Decimal]:
        return Decimal
```

Użycie w modelu:

```python
# examples/12_money_model.py
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

from examples_12_money import Money


class Base(DeclarativeBase):
    pass


class Reader(Base):
    __tablename__ = "reader"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(unique=True)
    # Domyślna wartość 0.00 — jawnie jako Decimal, nie jako int.
    unpaid_fines: Mapped[Money] = mapped_column(
        Money(18, 2), default=Decimal("0.00")
    )
```

> 🔬 **Pod maską**
>
> ```sql
> CREATE TABLE reader (
>     id INTEGER NOT NULL,
>     email VARCHAR NOT NULL,
>     unpaid_fines NUMERIC(18, 2) NOT NULL,
>     PRIMARY KEY (id)
> );
>
> INSERT INTO reader (email, unpaid_fines) VALUES (%(email)s, %(unpaid_fines)s);
> -- %(unpaid_fines)s zostanie wysłane jako Decimal('5.00')
> ```

Zwróć uwagę na `cache_ok = True` w kodzie. Wyjaśnijmy, dlaczego to nie jest opcjonalny dodatek.

### 9.3. `cache_ok` — dlaczego to nie jest ozdoba

SQLAlchemy, żeby być szybkie, cache'uje wynik **kompilacji** zapytania SQL. Klucz cache'a bierze się między innymi z typów użytych w zapytaniu. Żeby typ mógł trafić do klucza, SQLAlchemy musi wiedzieć, że jego instancje dają spójny wynik — czyli że `Money(18, 2)` zawsze znaczy to samo i nie ma ukrytych stanów, które zmieniają SQL.

Jeżeli tego nie potwierdzisz, dostaniesz ostrzeżenie w logu:

```
SAWarning: TypeDecorator Money() will not produce a cache key because
the cache_ok flag is not set. This can significantly degrade performance...
```

I to nie jest teoretyczne ostrzeżenie. Bez `cache_ok = True` **każde** wystąpienie tego typu w zapytaniu omija cache kompilacji, co przy pętli wykonującej tysiące podobnych zapytań oznacza realny narzut na procesor i pamięć.

Warunki, przy których `cache_ok = True` jest bezpieczne: klasa nie ma stanu, który zmienia SQL; jeśli ma parametry, to wyłącznie skalarne wartości (liczby, napisy, `Enum`), a nie obiekty, które mogą się zmieniać po zbudowaniu typu. Nasze `Money(18, 2)` spełnia ten warunek — parametry są liczbami i są przekazywane do `impl`.

> ⚠️ **Pułapka — `cache_ok = True` na typie ze stanem mutowalnym**
>
> Jeśli typ dostaje w konstruktorze słownik z konfiguracją i gdzieś go modyfikujesz, `cache_ok = True` może doprowadzić do trudnych do znalezienia błędów, w których inne zapytania korzystają z nieaktualnego klucza cache. Zasada: `cache_ok = True` tylko dla typów bezstanowych albo z parametrami, których nie ruszasz po zbudowaniu. Nigdy nie implementuj `__eq__`/`__hash__` na typie, jeśli nie wiesz dokładnie, co robisz — psuje klucz cache'a.

### 9.4. Przykład 2: `UTCDateTime`

Wróćmy do problemu z punktu 3.4: SQLite gubi strefy, a my chcemy zawsze trzymać UTC i zawsze odtwarzać świadomy obiekt.

```python
# examples/12_utc_datetime.py
from datetime import datetime, timezone

from sqlalchemy import DateTime, TypeDecorator
from sqlalchemy.engine import Dialect


class UTCDateTime(TypeDecorator[datetime]):
    """Zawsze zapisuje i zwraca świadomy datetime w UTC."""

    # W bazie trzymamy DateTime(timezone=True) — PostgreSQL to uszanuje.
    impl = DateTime(timezone=True)
    cache_ok = True

    def process_bind_param(self, value: object, dialect: Dialect) -> datetime | None:
        if value is None:
            return None
        if not isinstance(value, datetime):
            raise TypeError(f"UTCDateTime oczekuje datetime, dostało {type(value)!r}")
        if value.tzinfo is None:
            # Naiwny datetime traktujemy jako już wyrażony w UTC.
            # Możesz zdecydować inaczej i rzucać wyjątek — to kwestia polityki.
            return value.replace(tzinfo=timezone.utc)
        return value.astimezone(timezone.utc)

    def process_result_value(self, value: object, dialect: Dialect) -> datetime | None:
        if value is None:
            return None
        if not isinstance(value, datetime):
            raise TypeError(f"Baza zwróciła {type(value)!r}, oczekiwano datetime")
        if value.tzinfo is None:
            # SQLite zwraca napis bez strefy — doklejamy UTC świadomie.
            return value.replace(tzinfo=timezone.utc)
        return value.astimezone(timezone.utc)
```

To jest dokładnie ten wzorzec, który warto mieć w każdym projekcie: **jeden typ, jedna reguła, zero niespodzianek**. Aplikacja nie musi pamiętać o `.astimezone(timezone.utc)` w pięćdziesięciu miejscach — robi to typ.

> 🧠 **Dlaczego tak jest — strefa to decyzja projektowa, nie właściwość danych**
>
> Moment w czasie jest obiektywny. Sposób jego zapisania (ze strefą czy bez) jest umowny. Gdy umowa jest rozproszona po całym kodzie, spotykają się dwa różne poglądy i powstaje błąd. Gdy umowa jest w jednym typie, każdy zapis i odczyt przechodzi przez ten sam celnik — i nie ma gdzie się pomylić.

### 9.5. Przykład 3: `Slug` z normalizacją

Pokażmy jeszcze typ, który nie tylko konwertuje, ale też **normalizuje** wartość.

```python
# examples/12_slug.py
import re
import unicodedata

from sqlalchemy import String, TypeDecorator
from sqlalchemy.engine import Dialect

_SLUG_ALLOWED = re.compile(r"[^a-z0-9]+")


class Slug(TypeDecorator[str]):
    """Adres URL-owy: małe litery, myślniki, bez znaków diakrytycznych."""

    impl = String
    cache_ok = True

    def __init__(self, length: int = 200) -> None:
        super().__init__(length)

    @staticmethod
    def normalize(value: str) -> str:
        # "Zażółć gęślą jaźń" -> "zazolc-gesla-jazn"
        ascii_only = (
            unicodedata.normalize("NFKD", value)
            .encode("ascii", "ignore")
            .decode("ascii")
        )
        slug = _SLUG_ALLOWED.sub("-", ascii_only.lower()).strip("-")
        return slug or "n-a"

    def process_bind_param(self, value: object, dialect: Dialect) -> str | None:
        if value is None:
            return None
        if not isinstance(value, str):
            raise TypeError(f"Slug oczekuje tekstu, dostało {type(value)!r}")
        return self.normalize(value)

    def process_result_value(self, value: object, dialect: Dialect) -> str | None:
        # Na odczycie nic nie zmieniamy — w bazie już jest znormalizowane.
        return None if value is None else str(value)
```

Ta sama idea — znormalizować dane **w jednym miejscu**, przed zapisem — dotyczy e-maili (lowercase), kodów pocztowych, numerów telefonów, numerów NIP i kodów ISIN. To wzorzec, który ratuje przed sytuacją, w której `Jan@example.com` i `jan@example.com` traktowane są jako różne konta.

### 9.6. Przykład 4: typ zależny od dialektu

Jeśli naprawdę chcesz mieć `JSONB` na PostgreSQL i `JSON` gdzie indziej, `load_dialect_impl` jest właściwym miejscem — nie warunek `if` w kodzie aplikacji.

```python
# examples/12_portable_json.py
from sqlalchemy import JSON, TypeDecorator
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.engine import Dialect, TypeEngine


class PortableJSON(TypeDecorator[dict]):
    """JSONB na PostgreSQL, JSON w pozostałych dialektach."""

    impl = JSON
    cache_ok = True

    def load_dialect_impl(self, dialect: Dialect) -> TypeEngine[object]:
        if dialect.name == "postgresql":
            return dialect.type_descriptor(JSONB())
        return dialect.type_descriptor(JSON())

    def process_bind_param(self, value: object, dialect: Dialect) -> object:
        return value

    def process_result_value(self, value: object, dialect: Dialect) -> object:
        return value
```

Teraz w modelu piszesz po prostu `Mapped[dict[str, object]] = mapped_column(PortableJSON)` i zapominasz o różnicach dialektów. To czysty zysk: jedna decyzja, jedno miejsce, brak rozgałęzień w kodzie biznesowym.

> 🧪 **Ćwiczenie — Ile dialektów obsłuży twój typ?**
>
> Weź klasę `PortableJSON` i dodaj obsługę dialektu SQLite w taki sposób, aby w 2.0 używał `JSON`, a w 2.1 (jeśli jest dostępny) mógł użyć `JSONB`. Zauważ, że musisz porównywać wersje SQLAlchemy — albo po prostu zostaw `JSON` i wyjaśnij w komentarzu, dlaczego świadomie rezygnujesz z rozwoju.

---

## 10. Trzy poziomy walidacji

Naturalnym odruchem po nauce `TypeDecorator` jest chęć walidowania wszystkiego w typie. To błąd. Walidacja ma trzy poziomy i każdy odpowiada za coś innego.

| Poziom | Realizacja | Widzi kontekst sesji? | Widzi inne pola obiektu? | Koszt | Odpowiedni do |
|---|---|---|---|---|---|
| 1. Typ kolumny | `TypeDecorator.process_bind_param` | nie | nie | zerowy przy odczycie | normalizacja i konwersja (UTC, slug, decimal), tanie kontrole zakresu |
| 2. Model ORM | `@validates` (moduł 13) | nie do końca | tak, w ograniczonym zakresie | mały | spójność między polami encji, wartości wyliczane |
| 3. Schemat wejściowy | Pydantic (`v2`), ręczna walidacja | nie | tak | mały, ale tylko na wejściu | komunikaty dla użytkownika, formaty, limity długości, wzorce |
| 4. Baza danych | `CHECK`, `NOT NULL`, `UNIQUE`, `FOREIGN KEY` | nie | nie | zerowy | ostateczna linia obrony, integralność danych |

Zasada praktyczna:

- **Normalizacja i konwersja → typ kolumny.** `Jan@Example.com` ma być zapisane jako `jan@example.com` niezależnie od tego, która ścieżka kodu wykonała `INSERT`.
- **Komunikaty dla użytkownika → schemat wejściowy (Pydantic).** Użytkownik ma dostać „Podaj adres e-mail w formacie nazwa@domena.pl”, a nie „ValueError z TypeDecorator”.
- **Integralność danych → baza.** Nawet jeśli ktoś ominie ORM (skrypt, ręczny `UPDATE`), baza ma nie dopuścić do niespójności.

Warto też wiedzieć, gdzie walidacja NIE działa: `@validates` nie jest wywoływany przy masowych aktualizacjach przez `session.execute(update(...))`, a `TypeDecorator` — mimo że działa wszędzie — nie dostaje informacji o innych kolumnach. Jeśli reguła brzmi „data_zwrotu musi być późniejsza niż data_wypożyczenia”, to nie jest miejsce na `TypeDecorator`; to miejsce na walidację modelu albo ograniczenie w bazie.

Przykład podziału ról na jednej domenie — e-mail:

```python
# examples/12_three_levels_email.py
#
# 1. Typ: normalizacja + ograniczenie długości na poziomie kolumny.
#
from sqlalchemy import String, TypeDecorator
from sqlalchemy.engine import Dialect


class Email(TypeDecorator[str]):
    """E-mail zapisywany zawsze małymi literami, bez spacji wokół."""

    impl = String
    cache_ok = True

    def __init__(self, length: int = 254) -> None:
        super().__init__(length)

    def process_bind_param(self, value: object, dialect: Dialect) -> str | None:
        if value is None:
            return None
        if not isinstance(value, str):
            raise TypeError("Email oczekuje tekstu")
        return value.strip().lower()

    def process_result_value(self, value: object, dialect: Dialect) -> str | None:
        return None if value is None else str(value)


#
# 2. Schemat Pydantic: komunikaty dla użytkownika i format.
#
# from pydantic import BaseModel, EmailStr
#
# class ReaderCreate(BaseModel):
#     email: EmailStr
#
#
# 3. Baza: limit długości i unikalność.
#
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class Reader(Base):
    __tablename__ = "reader"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(Email(254), unique=True, nullable=False)
```

Warto zauważyć, że te trzy warstwy nie konkurują — uzupełniają się. Pydantic daje ładny komunikat w API, typ `Email` gwarantuje, że każdy `INSERT` (także ze skryptu administracyjnego) zapisze wartość w tej samej postaci, a `unique=True` i `NOT NULL` pilnują integralności na samym końcu.

> ⚠️ **Pułapka — walidacja tylko w Pydantic**
>
> Jeśli jedyną walidacją jest schemat Pydantic, to każdy skrypt migracyjny, konsument kolejki, zadanie cron i panel administracyjny stanie się dziurą w walidacji. Schemat chroni kontrakt HTTP, nie dane. Dane chroni baza plus normalizacja w typie.

---

## 11. Typy specyficzne dla dialektu i przenośność

SQLite i PostgreSQL mają różne możliwości. SQLite jest projektowana jako baza „wszystko jest napisem, prawie wszystko jest dozwolone”. PostgreSQL ma bogatą paletę typów: `JSONB`, `ARRAY`, `INET`, `CIDR`, `TSVECTOR`, `Range`, `HSTORE`, `CITEXT`. SQLAlchemy udostępnia je w modułach `sqlalchemy.dialects.postgresql`.

| Typ | Moduł | Co reprezentuje | Odpowiednik przenośny |
|---|---|---|---|
| `postgresql.JSONB` | `sqlalchemy.dialects.postgresql` | JSON binarny, indeksowalny | `JSON` |
| `postgresql.JSON` | `sqlalchemy.dialects.postgresql` | JSON tekstowy | `JSON` |
| `postgresql.ARRAY` | `sqlalchemy.dialects.postgresql` | tablica wartości | `JSON` (lista) |
| `postgresql.INET` | `sqlalchemy.dialects.postgresql` | adres IP (v4/v6) | `String(45)` |
| `postgresql.CIDR` | `sqlalchemy.dialects.postgresql` | sieć IP | `String(50)` |
| `postgresql.TSVECTOR` | `sqlalchemy.dialects.postgresql` | wektor wyszukiwania pełnotekstowego | `Text` + wyszukiwanie w aplikacji |
| `postgresql.CITEXT` | `sqlalchemy.dialects.postgresql` | tekst bez rozróżniania wielkości liter | `String` + `func.lower()` |
| `postgresql.HSTORE` | `sqlalchemy.dialects.postgresql` | słownik klucz-wartość (przestarzały) | `JSON` |
| `postgresql.INT4RANGE`, `NUMRANGE`, `DATERANGE`, `TSTZRANGE` | `sqlalchemy.dialects.postgresql` | zakresy wartości | dwie kolumny `_from` / `_to` |

### 11.1. `with_variant` — jeden model, wiele dialektów

Gdy chcesz mieć przenośny model bez pisania własnego `TypeDecorator`, możesz użyć `with_variant`. Mówi on SQLAlchemy: „na PostgreSQL używaj tego typu, wszędzie indziej — tamtego”.

```python
# examples/12_with_variant.py
from sqlalchemy import String, Uuid
from sqlalchemy.dialects.postgresql import CITEXT, INET
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class User(Base):
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(primary_key=True)
    # Na PostgreSQL: CITEXT (porównania bez rozróżniania wielkości liter).
    # Wszędzie indziej: VARCHAR(254).
    email: Mapped[str] = mapped_column(String(254).with_variant(CITEXT(), "postgresql"))
    # Na PostgreSQL: INET. Wszędzie indziej: VARCHAR(45) (mieści IPv6 z prefiksem).
    last_ip: Mapped[str | None] = mapped_column(
        String(45).with_variant(INET(), "postgresql"), nullable=True
    )
```

> 🔬 **Pod maską**
>
> ```sql
> -- PostgreSQL
> CREATE TABLE user_account (
>     id SERIAL NOT NULL,
>     email CITEXT NOT NULL,
>     last_ip INET,
>     PRIMARY KEY (id)
> );
>
> -- SQLite
> CREATE TABLE user_account (
>     id INTEGER NOT NULL,
>     email VARCHAR(254) NOT NULL,
>     last_ip VARCHAR(45),
>     PRIMARY KEY (id)
> );
> ```
>
> W SQLite `CITEXT` nie istnieje, więc SQLAlchemy użyje wariantu bazowego. Uwaga jednak: wariant bazowy nie zachowuje semantyki `CITEXT` (porównywanie bez wielkości liter). Jeśli chcesz mieć identyczne zachowanie, musisz normalizować wartości w typie (patrz `Email` w punkcie 10).

### 11.2. Kiedy typy dialektu, a kiedy przenośność

Trzy strategie, każda z inną ceną:

1. **Tylko przenośne typy.** Model działa identycznie na SQLite i PostgreSQL, ale tracisz indeksy GIN na JSON-ie, wyszukiwanie pełnotekstowe i typy sieciowe. Dobre dla bibliotek i narzędzi wielobazowych.
2. **Typy dialektu z fallbackiem `with_variant`.** Model jest przenośny na poziomie DDL, ale bogate możliwości działają tylko na PostgreSQL. Dobre dla aplikacji, które chcą działać w testach na SQLite i na produkcji na PostgreSQL.
3. **Typy dialektu bez fallbacku.** Model wymaga PostgreSQL. Testy muszą działać na PostgreSQL (`testcontainers`, Dockera). Najbardziej realistyczne i najczęstsze w poważnych projektach.

> 🧠 **Dlaczego tak jest — testowanie na tej samej bazie co produkcja**
>
> Jeśli testujesz na SQLite, a wdrażasz na PostgreSQL, to testujesz inny produkt. Różnice potrafią być subtelne: `RETURNING` w starszych wersjach SQLite, brak `FOR UPDATE NOWAIT`, inne rzutowanie typów, inna semantyka porównania napisów. Moduł [`18_testowanie.md`](18_testowanie.md) pokazuje, jak uruchomić PostgreSQL w kontenerze na czas testów. Reguła: SQLite świetnie służy do nauki i szybkich testów jednostkowych logiki, ale testy integracyjne z bazą powinny chodzić na docelowym silniku.

> 🧪 **Ćwiczenie — Zbadaj swój model**
>
> Weź model z modułu 09 i sprawdź, jaki DDL wygeneruje na dwóch dialektach: `CreateTable(Table).compile(dialect=sqlite.dialect())` i `...postgresql.dialect())`. Wypisz różnice i oceń, czy któraś z nich wpłynie na działanie aplikacji (np. czy gdzieś polegasz na kolejności napisów albo na typie danych przy porównaniu).

---

## 12. Pełny przykład: profil czytelnika

Zbierzmy wszystko w jeden uruchamialny skrypt. Model zawiera:

- `Uuid` jako klucz publiczny,
- `Email` jako `TypeDecorator` z normalizacją,
- `Enum` statusu czytelnika,
- `JSON` z `MutableDict` na preferencje,
- `Money` na nieuregulowane kary,
- `UTCDateTime` na znaczniki czasu.

Skrypt na końcu udowadnia dwie rzeczy: że zmiana wnętrza JSON-a jest zapisywana, oraz że da się filtrować po ścieżce JSON w przenośny sposób.

```python
# examples/12_reader_profile.py
from __future__ import annotations

import enum
import uuid
from datetime import datetime, timezone
from decimal import ROUND_HALF_UP, Decimal

from sqlalchemy import (
    JSON,
    Enum,
    Numeric,
    String,
    TypeDecorator,
    Uuid,
    create_engine,
    select,
)
from sqlalchemy.dialects import postgresql, sqlite
from sqlalchemy.engine import Dialect
from sqlalchemy.ext.mutable import MutableDict
from sqlalchemy.orm import DeclarativeBase, Mapped, Session, mapped_column
from sqlalchemy.schema import CreateTable


# ----------------------------------------------------------------------
# 1. Własne typy
# ----------------------------------------------------------------------
class Email(TypeDecorator[str]):
    """E-mail zapisywany małymi literami, z obciętymi spacjami."""

    impl = String
    cache_ok = True

    def __init__(self, length: int = 254) -> None:
        super().__init__(length)

    def process_bind_param(self, value: object, dialect: Dialect) -> str | None:
        if value is None:
            return None
        return str(value).strip().lower()

    def process_result_value(self, value: object, dialect: Dialect) -> str | None:
        return None if value is None else str(value)


class UTCDateTime(TypeDecorator[datetime]):
    """Zawsze świadomy datetime w UTC — po obu stronach."""

    impl = __import__("sqlalchemy").DateTime(timezone=True)
    cache_ok = True

    def process_bind_param(self, value: object, dialect: Dialect) -> datetime | None:
        if value is None:
            return None
        dt = value  # type: ignore[assignment]
        if dt.tzinfo is None:  # type: ignore[union-attr]
            return dt.replace(tzinfo=timezone.utc)  # type: ignore[union-attr]
        return dt.astimezone(timezone.utc)  # type: ignore[union-attr]

    def process_result_value(self, value: object, dialect: Dialect) -> datetime | None:
        if value is None:
            return None
        dt = value  # type: ignore[assignment]
        if dt.tzinfo is None:  # type: ignore[union-attr]
            return dt.replace(tzinfo=timezone.utc)  # type: ignore[union-attr]
        return dt.astimezone(timezone.utc)  # type: ignore[union-attr]


class Money(TypeDecorator[Decimal]):
    """Kwota pieniężna: NUMERIC(18,2) w bazie, Decimal w Pythonie."""

    impl = Numeric
    cache_ok = True

    def __init__(self, precision: int = 18, scale: int = 2) -> None:
        super().__init__(precision=precision, scale=scale)

    def process_bind_param(self, value: object, dialect: Dialect) -> Decimal | None:
        if value is None:
            return None
        if isinstance(value, (int, float, str)):
            value = Decimal(str(value))
        if not isinstance(value, Decimal):
            raise TypeError(f"Money oczekuje Decimal, dostało {type(value)!r}")
        return value.quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)

    def process_result_value(self, value: object, dialect: Dialect) -> Decimal | None:
        if value is None:
            return None
        return Decimal(str(value)).quantize(
            Decimal("0.01"), rounding=ROUND_HALF_UP
        )


# ----------------------------------------------------------------------
# 2. Model
# ----------------------------------------------------------------------
class Base(DeclarativeBase):
    pass


class ReaderStatus(enum.Enum):
    ACTIVE = "active"
    BLOCKED = "blocked"
    PENDING = "pending"


class Reader(Base):
    __tablename__ = "reader"

    id: Mapped[uuid.UUID] = mapped_column(
        Uuid(as_uuid=True), primary_key=True, default=uuid.uuid4
    )
    email: Mapped[str] = mapped_column(Email(254), unique=True, nullable=False)

    status: Mapped[ReaderStatus] = mapped_column(
        Enum(
            ReaderStatus,
            native_enum=False,
            create_constraint=True,
            length=20,
            values_callable=lambda e: [m.value for m in e],
            validate_strings=True,
            name="ck_reader_status",
        ),
        default=ReaderStatus.PENDING,
        nullable=False,
    )

    # MutableDict gwarantuje, że zmiana klucza w słowniku wygeneruje UPDATE.
    preferences: Mapped[dict[str, object]] = mapped_column(
        MutableDict.as_mutable(JSON), default=dict, nullable=False
    )

    unpaid_fines: Mapped[Decimal] = mapped_column(
        Money(18, 2), default=Decimal("0.00"), nullable=False
    )

    created_at: Mapped[datetime] = mapped_column(
        UTCDateTime, default=lambda: datetime.now(timezone.utc), nullable=False
    )


# ----------------------------------------------------------------------
# 3. Demonstracja
# ----------------------------------------------------------------------
def main() -> None:
    engine = create_engine("sqlite://", echo=False)

    # Pokaż DDL dla dwóch dialektów — bez uruchamiania PostgreSQL.
    print("=== DDL: SQLite ===")
    print(CreateTable(Reader.__table__).compile(dialect=sqlite.dialect()))
    print("=== DDL: PostgreSQL ===")
    print(CreateTable(Reader.__table__).compile(dialect=postgresql.dialect()))

    Base.metadata.create_all(engine)

    with Session(engine) as session:
        reader = Reader(
            email="  Anna.Kowalska@Example.COM  ",
            preferences={"theme": "light", "notifications": {"email": True}},
        )
        session.add(reader)
        session.commit()
        print("Znormalizowany e-mail:", reader.email)
        print("Status:", reader.status)
        print("Data (świadoma):", reader.created_at, reader.created_at.tzinfo)

        reader_id = reader.id

        # --- Test 1: mutacja JSON-a -------------------------------
        reader.preferences["theme"] = "dark"
        print("is_modified po zmianie JSON:", session.is_modified(reader))
        session.commit()

    with Session(engine) as session:
        # --- Test 2: czytamy z powrotem ---------------------------
        stored = session.get(Reader, reader_id)
        assert stored is not None
        print("Preferencje po odczycie:", stored.preferences)
        print("Wartość 'theme' zapisana? ",
              stored.preferences["theme"] == "dark")

        # --- Test 3: operacja na pieniądzach -----------------------
        stored.unpaid_fines = stored.unpaid_fines + Decimal("12.5")
        session.commit()
        print("Kara po dodaniu 12.50:", stored.unpaid_fines)

        # --- Test 4: filtrowanie po ścieżce JSON (przenośne) -------
        stmt = select(Reader.email).where(
            Reader.preferences["theme"].as_string() == "dark"
        )
        print("SQL dla PostgreSQL:")
        print(stmt.compile(
            dialect=postgresql.dialect(),
            compile_kwargs={"literal_binds": True},
        ))
        print("SQL dla SQLite:")
        print(stmt.compile(
            dialect=sqlite.dialect(),
            compile_kwargs={"literal_binds": True},
        ))
        found = session.scalars(stmt).all()
        print("Znalezieni czytelnicy z motywem 'dark':", found)


if __name__ == "__main__":
    main()
```

**Jak uruchomić:**

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install "SQLAlchemy>=2.0"
python examples/12_reader_profile.py
```

Oczekiwany przebieg: najpierw dwa bloki DDL pokazujące różnice między dialektami, potem sekwencja wydruków — znormalizowany e-mail (`anna.kowalska@example.com`), status `ReaderStatus.PENDING`, świadoma data z `tzinfo` równym `UTC`, `is_modified` równe `True` po zmianie JSON-a, zapisana wartość `'dark'`, kwota kary `Decimal('12.50')` oraz dwa różne zapytania SQL (dla PostgreSQL z `->>` i dla SQLite z `json_extract`).

Zwróć uwagę na jeden szczegół w kodzie: w `UTCDateTime` użyliśmy `__import__("sqlalchemy").DateTime(...)` tylko po to, by uniknąć dodawania kolejnej linii do importów w tym konkretnym skrócie. W prawdziwym projekcie napisz po prostu `from sqlalchemy import DateTime` i użyj `impl = DateTime(timezone=True)` — to czytelniejsze i nie zaskakuje czytelnika.

> 🧪 **Ćwiczenie — Rozszerz profil**
>
> Dodaj do modelu `Reader` dwie rzeczy: (a) kolumnę `locale: Mapped[str]` z typem `String(5)` i `server_default='pl_PL'`, (b) pole `last_login_at: Mapped[datetime | None]` z typu `UTCDateTime`. Napisz zapytanie, które znajduje czytelników, którzy nie logowali się od 30 dni — pamiętaj o strefie czasu przy porównaniu.

---

## Podsumowanie

1. **Typ kolumny to kontrakt.** Odpowiada za konwersję Python ↔ SQL. Zły typ powoduje ciche błędy, nie wyjątki.
2. **Pieniądze zawsze w `Numeric`, nigdy w `Float`.** Używaj `Decimal`, konwertuj z `float` przez `str`, nie przez binarną reprezentację.
3. **Czas zawsze w UTC, kolumna zawsze `timezone=True`.** SQLite nie przechowuje stref — użyj `TypeDecorator`, jeśli chcesz działać na obu bazach.
4. **`default` (Python) vs `server_default` (baza) to różne rzeczy.** Wybieraj świadomie; `server_default=func.now()` daje jeden autorytet czasu.
5. **JSON ma sens dla danych niejednorodnych i rzadko filtrowanych.** Jeśli filtrujesz po kluczu w każdym zapytaniu — to nie jest JSON, to jest kolumna.
6. **Mutacja wnętrza JSON-a nie jest wykrywana.** Używaj `MutableDict`/`MutableList`, `flag_modified` albo podmieniaj całą wartość.
7. **`Enum` zapisuje nazwy, nie wartości** — chyba że podasz `values_callable`. I pamiętaj: `create_constraint=False` jest domyślne od 1.4.
8. **`Uuid` nie ujawnia liczby rekordów, ale psuje sekwencyjność indeksu.** Rozważ parę `BIGINT` (techniczny) + `Uuid` (publiczny).
9. **`cache_ok = True` na każdym własnym typie bezstanowym.** To nie dekoracja, to warunek korzystania z cache'u kompilacji.
10. **Normalizacja w typie, komunikaty w schemacie, integralność w bazie.** Trzy poziomy walidacji robią trzy różne rzeczy i nie zastępują się nawzajem.

---

## Ćwiczenia

### Zadanie 1 — `EncryptedString`

Zaimplementuj `TypeDecorator` o nazwie `EncryptedString`, który przechowuje napisy w bazie w formie zaszyfrowanej, a w Pythonie zwraca jawny tekst.

Wymagania:

- Typ bazowy: `String(512)`.
- Szyfrowanie musi być odwracalne (nie hash) — do demonstracji wystarczy prosty szyfr XOR z kluczem, ale napisz komentarz, że **na produkcji użyłbyś `cryptography.fernet.Fernet` albo szyfrowania na poziomie bazy/KMS**, a klucz nigdy nie może być zahardkodowany w kodzie.
- `cache_ok = True`.
- `process_bind_param` przyjmuje `str | None`, zwraca `str | None`.
- `process_result_value` odwraca operację.

### Zadanie 2 — JSON + indeks GIN + zapytanie po ścieżce

Rozszerz model `Order` o pole `data` typu `postgresql.JSONB` i dodaj:

1. Indeks GIN na całym dokumencie.
2. Indeks po wyrażeniu `(data ->> 'status')`.
3. Zapytanie `select(Order.id).where(Order.data["status"].as_string() == "paid")`.
4. Zapytanie korzystające z operatora `@>` (`Order.data.op("@>")({"status": "paid", "currency": "PLN"})`).
5. Wypisz skompilowany SQL dla dialektu PostgreSQL dla obu zapytań.
6. W komentarzu wyjaśnij, dlaczego indeks GIN z `jsonb_path_ops` obsłużyłby tylko jedno z tych zapytań.

### Zadanie 3 — `PositiveMoney`

Napisz podklasę `Money` o nazwie `PositiveMoney`, która dodatkowo odrzuca wartości ujemne.

- Konstruktor bez zmian.
- W `process_bind_param` po zaokrągleniu sprawdź, czy kwota jest nieujemna; jeśli nie, podnieś `ValueError` z jasnym komunikatem.
- W komentarzu wyjaśnij, dlaczego ta walidacja **nie** zastępuje ograniczenia `CHECK (kwota >= 0)` w bazie: chodzi o to, kto może pominąć walidację (np. `psql`, skrypt migracyjny, inny serwis).
- Czy `cache_ok` nadal może być `True`? Uzasadnij.

---

### Rozwiązania

#### Rozwiązanie 1

```python
# examples/12_solutions_encrypted_string.py
from sqlalchemy import String, TypeDecorator
from sqlalchemy.engine import Dialect


class EncryptedString(TypeDecorator[str]):
    """Napisy przechowywane w bazie w formie odwracalnie zaszyfrowanej.

    Szyfrowanie demonstracyjne: XOR bajt po bajcie z kluczem. NIE używaj tego
    w produkcji. Na produkcji: `cryptography.fernet.Fernet` albo szyfrowanie
    po stronie bazy/KMS, a klucz wyłącznie ze zmiennej środowiskowej lub
    menedżera sekretów.
    """

    impl = String
    cache_ok = True

    def __init__(self, length: int = 512, key: str = "demo-key") -> None:
        super().__init__(length)
        # Klucz jest parametrem budującym typ. Jest niezmienny po utworzeniu
        # typu, więc nie psuje cache_ok — ale UWAGA: dwa typy z różnymi
        # kluczami to dwa różne klucze cache'a. W praktyce klucz powinien
        # pochodzić z konfiguracji środowiska i być ten sam w całej aplikacji.
        self._key_bytes = key.encode("utf-8")

    def _xor(self, data: bytes) -> bytes:
        key = self._key_bytes
        return bytes(b ^ key[i % len(key)] for i, b in enumerate(data))

    def process_bind_param(self, value: object, dialect: Dialect) -> str | None:
        if value is None:
            return None
        if not isinstance(value, str):
            raise TypeError(f"EncryptedString oczekuje tekstu, dostało {type(value)!r}")
        import base64

        return base64.b64encode(self._xor(value.encode("utf-8"))).decode("ascii")

    def process_result_value(self, value: object, dialect: Dialect) -> str | None:
        if value is None:
            return None
        import base64

        raw = base64.b64decode(str(value).encode("ascii"))
        return self._xor(raw).decode("utf-8")
```

**Dlaczego tak:** szyfrowanie jest symetryczne, więc konwersja w obie strony jest dokładnie odwrotna. `base64` jest potrzebne, żeby zaszyfrowane bajty dały się bezpiecznie zapisać w kolumnie tekstowej.

**Alternatywa:** jeśli masz dostęp do `cryptography`, użyj `Fernet` — daje uwierzytelnione szyfrowanie (nie tylko odwracalne) i wbudowaną rotację kluczy.

**Minipułapka:** szyfrowanie w kolumnie **uniemożliwia zapytania po treści**. `WHERE email = 'a@b.pl'` nie zadziała, bo w bazie jest inny napis. Jeśli potrzebujesz filtrować po tym polu, dodaj osobną kolumnę z hashem (np. `encrypted_email_hash`) i porównuj po niej.

#### Rozwiązanie 2

```python
# examples/12_solutions_json_gin.py
from sqlalchemy import Index, select
from sqlalchemy.dialects import postgresql
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class Order(Base):
    __tablename__ = "order"

    id: Mapped[int] = mapped_column(primary_key=True)
    data: Mapped[dict[str, object]] = mapped_column(JSONB)

    __table_args__ = (
        # 1. Indeks GIN po całym dokumencie JSONB.
        Index("ix_order_data_gin", "data", postgresql_using="gin"),
        # 2. Indeks po wyrażeniu (data ->> 'status').
        Index("ix_order_data_status", data["status"].astext),
    )


# 3. Zapytanie po ścieżce.
stmt_path = select(Order.id).where(Order.data["status"].as_string() == "paid")

# 4. Zapytanie z operatorem @>.
stmt_contains = select(Order.id).where(
    Order.data.op("@>")({"status": "paid", "currency": "PLN"})
)

print(stmt_path.compile(
    dialect=postgresql.dialect(), compile_kwargs={"literal_binds": True}
))
print(stmt_contains.compile(
    dialect=postgresql.dialect(), compile_kwargs={"literal_binds": True}
))
```

Wygenerowany SQL:

```sql
-- stmt_path
SELECT "order".id FROM "order" WHERE ("order".data ->> 'status') = 'paid'

-- stmt_contains
SELECT "order".id FROM "order" WHERE "order".data @> '{"status": "paid", "currency": "PLN"}'
```

**Dlaczego tak:** `- >>` wyciąga pojedyncze pole jako tekst, a indeks po wyrażeniu doskonale to obsługuje. `@>` sprawdza zawieranie całych dokumentów i tu lepiej wypada indeks GIN.

**Dlaczego `jsonb_path_ops` obsłużyłby tylko drugie:** `jsonb_path_ops` indeksuje wyłącznie „ścieżki do wartości” i obsługuje tylko operator `@>`. Nigdy nie przyspieszy `data ->> 'status' = ...`, bo ten operator nie jest zawieraniem dokumentu. Indeks `jsonb_path_ops` jest mniejszy i szybszy niż domyślny `jsonb_ops`, ale jego zastosowanie jest węższe.

**Minipułapka:** indeks po wyrażeniu działa tylko wtedy, gdy w `WHERE` użyjesz **dokładnie tego samego wyrażenia**. `WHERE data ->> 'status' = 'paid'` pójdzie po indeksie; `WHERE (data ->> 'status')::text = 'paid'` już nie.

#### Rozwiązanie 3

```python
# examples/12_solutions_positive_money.py
from decimal import ROUND_HALF_UP, Decimal

from sqlalchemy.engine import Dialect

from examples_12_money import Money


class PositiveMoney(Money):
    """Kwota pieniężna, która nigdy nie może być ujemna."""

    cache_ok = True

    def process_bind_param(self, value: object, dialect: Dialect) -> Decimal | None:
        amount = super().process_bind_param(value, dialect)
        if amount is None:
            return None
        if amount < 0:
            raise ValueError(
                f"PositiveMoney nie przyjmuje kwot ujemnych (otrzymano {amount})"
            )
        return amount.quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)
```

**Dlaczego tak:** cała konwersja i zaokrąglanie są odziedziczone z `Money`; podklasa dokłada tylko jedną regułę. To pokazuje, że `TypeDecorator` można rozszerzać przez dziedziczenie — dokładnie jak każdą inną klasę.

**Dlaczego to nie zastępuje `CHECK`:** ograniczenie w typie działa tylko wtedy, gdy wartość przechodzi przez SQLAlchemy. Ręczny `UPDATE` z `psql`, skrypt migracyjny wykonujący `op.execute`, inny serwis piszący do tej samej tabeli, import z CSV — wszystkie te ścieżki omijają typ. Ograniczenie `CHECK (kwota >= 0)` działa niezależnie od tego, kto pisze. Wniosek: walidacja w typie daje dobry komunikat błędu, `CHECK` daje gwarancję integralności. Potrzebujesz obu.

**Czy `cache_ok = True`?** Tak — klasa nie ma stanu, a jej zachowanie deterministycznie zależy wyłącznie od argumentów konstruktora (których nie ma). Dwa `PositiveMoney()` są zawsze równoważne z punktu widzenia zapytania.

---

## Najczęstsze błędy i jak je czytać

| Komunikat / objaw | Przyczyna | Naprawa |
|---|---|---|
| `SAWarning: TypeDecorator ... will not produce a cache key` | brak `cache_ok = True` we własnym typie | dodaj `cache_ok = True` (jeśli typ jest bezstanowy) |
| `TypeError: can't compare offset-naive and offset-aware datetimes` | odczytany z bazy `datetime` jest naiwny, a porównujesz ze świadomym | użyj `UTCDateTime` albo `DateTime(timezone=True)` i normalizuj UTC |
| `TypeError: SQLite DateTime type only accepts Python datetime and date objects` | do kolumny `Date`/`DateTime` trafia np. `str` | sprawdź typ wartości; użyj `TypeDecorator` do parsowania napisów |
| Zmiana w `dict`/`list` nie zapisuje się, brak `UPDATE` w logu | SQLAlchemy nie wykrywa mutacji wnętrza | `MutableDict.as_mutable(JSON)` albo `flag_modified`, albo podmiana całej wartości |
| `IntegrityError: CHECK constraint failed: ck_reader_status` | nowa wartość enuma nie jest w `CHECK` w bazie | dopisz migrację rozszerzającą `CHECK` (`16_alembic_migracje.md`) |
| `LookupError: 'XYZ' is not among the defined enum values` | porównanie z napisem spoza enuma i `validate_strings=True` | używaj `ReaderStatus.ACTIVE`, nie `"active"` |
| Wszystkie wiersze mają identyczną datę utworzenia | `default=datetime.now()` zamiast `datetime.now` (wywołanie w chwili importu) | zamień na referencję do funkcji lub `server_default=func.now()` |
| Bilans różni się o grosze | kwoty w `Float` zamiast `Numeric` | zmień typ na `Numeric(18, 2)`, użyj `Decimal` |
| `psycopg.errors.UndefinedObject: type "citext" does not exist` | `CITEXT` wymaga rozszerzenia `citext` w bazie | `CREATE EXTENSION citext;` albo użyj `String` z `func.lower()` |
| Zapytanie po JSON-ie nie używa indeksu (wolne) | inny operator niż ten, który obsługuje indeks, albo rzutowanie typu | dopasuj warunek do indeksu; dodaj indeks po wyrażeniu dokładnie na używanym wyrażeniu |
| Po zmianie typu kolumny w kodzie aplikacja dalej widzi stary typ | zmiana modelu bez migracji; kolumna w bazie pozostała stara | napisz migrację (`16_alembic_migracje.md`), nie polegaj na `create_all` |
| `ValueError: A value is required for bind parameter` | w `insert().values()` brak wartości dla kolumny bez `nullable`/`default` | dodaj wartość, `default` albo zrób kolumnę `nullable=True` |

---

## Słowniczek modułu

| Termin (EN) | Polski | Wyjaśnienie |
|---|---|---|
| `TypeDecorator` | dekorator typu | klasa opakowująca istniejący typ i dodająca własną konwersję |
| `impl` | typ bazowy | typ realizujący faktyczny zapis/odczyt (np. `Numeric`, `String(30)`) |
| `process_bind_param` | przetwarzanie parametru wiązanego | konwersja Python → baza przed wysłaniem |
| `process_result_value` | przetwarzanie wartości wyniku | konwersja baza → Python po odczytaniu |
| `load_dialect_impl` | wybór implementacji dialektu | pozwala użyć innego typu bazowego dla PostgreSQL niż SQLite |
| `cache_ok` | znacznik cache'owalności | pozwala SQLAlchemy cache'ować skompilowaną formę zapytań |
| `python_type` | typ Pythona | deklaracja typu reprezentowanego przez własny typ |
| `bind parameter` | parametr wiązany | znak zapytania albo `:nazwa` w SQL, którego wartość podstawia sterownik |
| `literal_binds` | wartość wstawiona wprost | opcja `compile()` wstawiająca wartości bezpośrednio do SQL |
| `timezone-aware datetime` | data świadoma strefy | obiekt wiedzący, w jakiej strefie jest wyrażony |
| `naive datetime` | data naiwna | obiekt bez informacji o strefie |
| `zone` / `tzinfo` | strefa / informacja o strefie | dane do wyliczenia przesunięcia względem UTC |
| UTC | uniwersalny czas skoordynowany | wspólna linia czasu bez czasu letniego |
| `server_default` | domyślna wartość po stronie serwera | wartość generowana przez bazę, nie przez Pythona |
| `onupdate` | wartość przy aktualizacji | np. `onupdate=func.now()` |
| `MutableDict` / `MutableList` | słownik/listа mutowalna | opakowania zawiadamiają sesję o zmianach wnętrza |
| `flag_modified` | oznacz jako zmieniony | ręczne powiadomienie sesji o zmianie atrybutu |
| `JSONB` | binarny JSON | typ PostgreSQL: sparsowany, indeksowalny |
| `GIN index` | indeks uogólniony odwrócony | struktura przyspieszająca zapytania po JSON, `tsvector` i tablicach |
| `@>` | zawiera | operator PostgreSQL sprawdzający, czy dokument zawiera inny dokument |
| `->` / `->>` | wyciągnij jako JSON / jako tekst | operatory dostępu do pola JSON |
| `jsonb_path_ops` | operatory ścieżki JSON | wariant indeksu GIN obsługujący wyłącznie `@>` |
| `CHECK constraint` | ograniczenie sprawdzające | warunek, który musi spełniać wartość w kolumnie |
| `native_enum` | typ enumeracyjny natywny | `CREATE TYPE ... AS ENUM` w PostgreSQL |
| `values_callable` | funkcja zwracająca wartości | decyduje, co jest zapisywane: nazwy czy wartości enuma |
| `Uuid` | identyfikator uniwersalnie unikalny | 128-bitowy identyfikator; `UUID` na PostgreSQL, `CHAR(32)` na SQLite |
| `B-tree index` | indeks drzewiasty | standardowy indeks bazy danych, wrażliwy na kolejność wstawiania |
| `LOB` | duży obiekt | mechanizm przechowywania dużych danych binarnych, np. w PostgreSQL |
| `CITEXT` | tekst bez wielkości liter | rozszerzenie PostgreSQL: porównania ignorują wielkość liter |
| `INET` / `CIDR` | adres sieci / podsieni | typy PostgreSQL na adres IP i sieć |
| `with_variant` | z wariantem | mechanizm wyboru alternatywnego typu dla wskazanego dialektu |
| `sequence` | sekwencja | obiekt bazy generujący kolejne liczby (postać autoinkrementacji) |

---

## Dalsze czytanie

- Typy wbudowane i ich parametry:
  `https://docs.sqlalchemy.org/en/20/core/type_basics.html`
- Tworzenie własnych typów (`TypeDecorator`):
  `https://docs.sqlalchemy.org/en/20/core/custom_types.html`
- API typu (`TypeEngine`, operatory, procesory):
  `https://docs.sqlalchemy.org/en/20/core/type_api.html`
- Typy dialektu PostgreSQL (w tym `JSONB`, `ARRAY`, `INET`, `CITEXT`, `TSVECTOR`, zakresy):
  `https://docs.sqlalchemy.org/en/20/dialects/postgresql.html`
- Typy i osobliwości dialektu SQLite (obsługa dat, `BLOB`):
  `https://docs.sqlalchemy.org/en/20/dialects/sqlite.html`
- Rozszerzenie `sqlalchemy.ext.mutable` (`MutableDict`, `MutableList`, `MutableSet`):
  `https://docs.sqlalchemy.org/en/20/orm/extensions/mutable.html`
- `Enum` — parametry `native_enum`, `create_constraint`, `values_callable`:
  `https://docs.sqlalchemy.org/en/20/core/type_basics.html#sqlalchemy.types.Enum`
- Dekorator typów na PostgreSQL — `with_variant` i warianty:
  `https://docs.sqlalchemy.org/en/20/core/type_api.html#sqlalchemy.types.TypeEngine.with_variant`

---

## Co dalej

Typy odpowiadają za to, **jak pojedyncza wartość zamienia się w dane i z powrotem**. Nie odpowiadają za to, co się dzieje, gdy obiekt zmienia stan — ani za reguły, które muszą być spełnione w obrębie całego modelu. Tym zajmiemy się w module 13, gdzie poznamy zdarzenia SQLAlchemy (`before_flush`, `before_insert`), walidację przy pomocy `@validates`, właściwości hybrydowe (`hybrid_property`), które działają jednocześnie w Pythonie i w SQL, oraz `association_proxy` dla wygodnego dostępu do relacji wiele-do-wielu.

Przejdź do [13_zdarzenia_i_hybrydy.md](13_zdarzenia_i_hybrydy.md).

<!-- koniec modułu 12 -->