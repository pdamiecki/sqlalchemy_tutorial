# Moduł 01 — Czym jest SQLAlchemy i po co istnieje

W tym module nie nauczysz się jeszcze pisać zapytań. Nauczysz się czegoś ważniejszego: **zrozumiesz, jaki problem rozwiązuje SQLAlchemy** i dlaczego w ogóle ktoś poświęcił lata na jej stworzenie. Zobaczysz, jak wygląda praca z bazą danych „gołymi rękami” i jak kolejne problemy pojawiają się jeden po drugim: najpierw niebezpieczne sklejanie zapytań ze stringów, potem ręczne przepisywanie wierszy z bazy na obiekty Pythona, wreszcie różnice między bazami, przez które ten sam kod działa na SQLite, a wywraca się na PostgreSQL. Dowiesz się, że SQLAlchemy to dwa narzędzia w jednym — **Core** (skrzynka z narzędziami do SQL-a) i **ORM** (automat, który mapuje obiekty Pythona na tabele) — i kiedy które z nich jest właściwym wyborem. Na koniec zobaczysz mapę całego kursu, żebyś od pierwszej chwili wiedział, gdzie w materiałach szukać odpowiedzi na swoje pytania.

---

**Blok metadanych**

- **Poziom:** 🟢 podstawowy
- **Czas:** ~90 minut
- **Wymagania wstępne:** umiejętność pisania i uruchamiania prostych skryptów w Pythonie (zmienne, funkcje, klasy, pętle, `pip`). **Nie** zakładam znajomości SQL, baz danych ani ORM.
- **Czego dotyczy plik:** `01_wprowadzenie.md` — moduł teoretyczno-orientacyjny. Nie zawiera jeszcze pełnego kodu produkcyjnego, ale zawiera **działające, uruchamialne przykłady**, na których zobaczysz każdy omawiany problem.
- **Poprzedni moduł:** [`00_README.md`](00_README.md) · **Następny moduł:** [`02_srodowisko_i_engine.md`](02_srodowisko_i_engine.md)

---

## Spis treści

- [Zaczynamy: dane, które muszą przetrwać](#zaczynamy-dane-które-muszą-przetrwać)
  - [Pierwsze podejście: plik tekstowy](#pierwsze-podejście-plik-tekstowy)
  - [Drugie podejście: baza danych i język SQL](#drugie-podejście-baza-danych-i-język-sql)
  - [Trzecie podejście: Python rozmawia z bazą bezpośrednio](#trzecie-podejście-python-rozmawia-z-bazą-bezpośrednio)
- [Problem 1: SQL sklejany stringami i wstrzyknięcie SQL](#problem-1-sql-sklejany-stringami-i-wstrzyknięcie-sql)
- [Problem 2: ręczne mapowanie wierszy na obiekty](#problem-2-ręczne-mapowanie-wierszy-na-obiekty)
- [Problem 3: jedna baza to nie każda baza](#problem-3-jedna-baza-to-nie-każda-baza)
- [SQLAlchemy jako tłumacz](#sqlalchemy-jako-tłumacz)
- [Dwie warstwy: Core i ORM](#dwie-warstwy-core-i-orm)
- [Trzy poziomy abstrakcji — to samo zadanie na trzy sposoby](#trzy-poziomy-abstrakcji--to-samo-zadanie-na-trzy-sposoby)
- [Filozofia „DBAPI + dialekt + kompilator”](#filozofia-dbapi--dialekt--kompilator)
- [Kiedy NIE używać ORM](#kiedy-nie-używać-orm)
- [Jak SQLAlchemy wypada na tle alternatyw](#jak-sqlalchemy-wypada-na-tle-alternatyw)
- [Ekosystem — co jeszcze przyda ci się w praktyce](#ekosystem--co-jeszcze-przyda-ci-się-w-praktyce)
- [Historia i wersjonowanie: dlaczego stare przykłady nie działają](#historia-i-wersjonowanie-dlaczego-stare-przykłady-nie-działają)
- [Mapa całego kursu](#mapa-całego-kursu)
- [Słownik startowy](#słownik-startowy--12-pojęć-które-wrócą-w-każdym-module)
- [Podsumowanie](#podsumowanie)
- [Ćwiczenia](#ćwiczenia)
- [Najczęstsze błędy i jak je czytać](#najczęstsze-błędy-i-jak-je-czytać)
- [Słowniczek modułu](#słowniczek-modułu)
- [Dalsze czytanie](#dalsze-czytanie)
- [Co dalej](#co-dalej)

---

## Zaczynamy: dane, które muszą przetrwać

Wyobraź sobie, że piszesz program dla małej biblioteki osiedlowej. Program ma pamiętać, kto wypożyczył jaką książkę i kiedy. Napiszesz funkcję `wypozycz(tytul, czytelnik)` i drugą `zwroc(tytul, czytelnik)`. Wszystko działa elegancko — dopóki nie zamkniesz okna terminala.

Tu pojawia się pierwsze, fundamentalne rozróżnienie, od którego zaczyna się cała informatyka danych:

- **Dane w pamięci** (zmienne, listy, słowniki) żyją tak długo, jak długo działa proces Pythona. Wyłączenie programu = dane znikają.
- **Dane trwałe** (persistent) muszą przeżyć restart programu, restart komputera i awarię zasilania.

Program, który „zapomina” wszystko po zamknięciu, jest bezużyteczny dla księgowej, która rano chce zobaczyć, kto ma zaległe książki. Potrzebujemy **pamięci trwałej**. I tu zaczyna się droga, na której końcu znajdziemy SQLAlchemy.

> 💡 **Analogia** — Pamięć operacyjna to tablica suchościeralna w sali konferencyjnej: świetna do pracy bieżącej, ale ktoś ją zetrze po spotkaniu. Baza danych to segregator w szafie pancernej: zapis trwa, jest uporządkowany, można go przeszukiwać i chronić hasłem.

### Pierwsze podejście: plik tekstowy

Najprostszy pomysł, jaki przychodzi do głowy: zapisujmy dane do pliku. Brzmi rozsądnie, więc spróbujmy.

```python
# examples/01_plik_tekstowy.py
"""Najprostsza 'baza danych' świata: dopisywanie linii do pliku."""

from datetime import date


def dodaj_wypozyczenie(sciezka: str, czytelnik: str, tytul: str, kiedy: date) -> None:
    """Dopisuje jedno wypożyczenie jako linię tekstu."""
    with open(sciezka, "a", encoding="utf-8") as f:
        f.write(f"{kiedy.isoformat()}|{czytelnik}|{tytul}\n")


def znajdz_wypozyczenia(sciezka: str, czytelnik: str) -> list[str]:
    """Szuka wypożyczeń danego czytelnika."""
    wynik: list[str] = []
    with open(sciezka, encoding="utf-8") as f:
        for linia in f:
            data, kto, tytul = linia.rstrip("\n").split("|")
            if kto == czytelnik:
                wynik.append(f"{data}: {tytul}")
    return wynik


if __name__ == "__main__":
    dodaj_wypozyczenie("wypozyczenia.txt", "alicja", "Wiedźmin", date(2026, 9, 1))
    dodaj_wypozyczenie("wypozyczenia.txt", "bob", "Solaris", date(2026, 9, 2))
    print(znajdz_wypozyczenia("wypozyczenia.txt", "alicja"))
    # ['2026-09-01: Wiedźmin']
```

**Jak uruchomić:**

```bash
python examples/01_plik_tekstowy.py
```

To działa. Przez pierwszy tydzień. Potem okazuje się, że:

1. **Coś się zepsuje w środku pliku** — jedna linia bez separatora `|` i cały odczyt się wywraca (`ValueError: not enough values to unpack`). Nie ma mechanizmu „wszystko albo nic”.
2. **Wyszukiwanie jest liniowe** — musimy przeczytać cały plik, żeby znaleźć jeden rekord. Przy 200 000 wypożyczeń to zauważalne.
3. **Nie ma współbieżności** — dwie kopie programu piszące jednocześnie mogą się nadpisać. Nie ma zamków, nie ma transakcji.
4. **Nie ma typów** — `kiedy` zapisujemy jako tekst i musimy wierzyć, że zawsze będzie to poprawna data. Nikt tego nie pilnuje.
5. **Każde zapytanie trzeba napisać samemu** — „pokaż 10 najpopularniejszych książek z ostatniego miesiąca” to kilkadziesiąt linii Pythona i cała logika w głowie programisty.
6. **Brak relacji** — jak zapisać, że wypożyczenie dotyczy konkretnego egzemplarza książki, a nie tylko jej tytułu? Trzeba wymyślać własne identyfikatory i pilnować ich ręcznie.

Wszystkie te problemy wynikają z jednego źródła: **plik tekstowy nie wie nic o strukturze danych**. To worek bajtów, a nie zbiór rekordów o określonych polach. Potrzebujemy czegoś, co zna strukturę.

### Drugie podejście: baza danych i język SQL

**Baza danych** (database) to program, którego jedynym zadaniem jest przechowywanie i udostępnianie danych w uporządkowanej, opisanej z góry strukturze. Bazy relacyjne — a o takich tu mówimy — przechowują dane w **tabelach**.

Zanim przejdziemy dalej, cztery pojęcia, bez których nic nie zrozumiesz:

| Pojęcie | Analogia do arkusza kalkulacyjnego | Znaczenie techniczne |
|---|---|---|
| **Tabela** (table) | cały arkusz | nazwany zbiór rekordów o tej samej strukturze |
| **Wiersz / rekord** (row, record) | jeden wiersz arkusza | jeden konkretny obiekt: jedna książka, jedno wypożyczenie |
| **Kolumna / pole** (column, field) | nagłówek kolumny | nazwany atrybut o określonym typie, np. `title TEXT` |
| **Klucz główny** (primary key) | numer wiersza, ale nadawany przez nas | kolumna jednoznacznie identyfikująca rekord; nigdy się nie powtarza |

> 💡 **Analogia** — Tabela to kartoteka z fiszkami, gdzie każda fiszka ma dokładnie te same rubryki: nazwisko, tytuł, data. Baza danych to szafa na takie kartoteki, plus bibliotekarz, który błyskawicznie znajduje fiszkę po dowolnej rubryce.

Do rozmowy z bazą relacyjną służy język **SQL** (Structured Query Language, strukturalny język zapytań). SQL to język deklaratywny — nie mówisz komputerowi *jak* szukać, mówisz *co* chcesz dostać. To olbrzymia różnica: `for linia in f:` mówi „przejdź po kolei przez wszystkie elementy”, a `SELECT ... WHERE` mówi „daj mi te wiersze, które pasują” — i nie interesuje cię, czy baza użyje indeksu, skanowania w pamięci czy trzech równoległych wątków.

Pięć poleceń SQL, których potrzebujesz do zrozumienia tego modułu:

```sql
-- CREATE: zdefiniuj strukturę tabeli
CREATE TABLE loans (
    id        INTEGER PRIMARY KEY,
    reader    TEXT    NOT NULL,
    title     TEXT    NOT NULL,
    loan_date DATE    NOT NULL
);

-- INSERT: dodaj wiersz
INSERT INTO loans (reader, title, loan_date)
VALUES ('alicja', 'Wiedźmin', '2026-09-01');

-- SELECT: przeczytaj wiersze (gwiazdka = wszystkie kolumny)
SELECT id, title, loan_date FROM loans WHERE reader = 'alicja';

-- UPDATE: zmień istniejące wiersze
UPDATE loans SET title = 'Wiedźmin (tom 1)' WHERE id = 1;

-- DELETE: usuń wiersze
DELETE FROM loans WHERE id = 1;
```

Zwróć uwagę na słowo `NOT NULL` — to **ograniczenie** (constraint). Mówi bazie: „w tej kolumnie nie wolno zostawić pustego miejsca”. Baza sama tego upilnuje, niezależnie od tego, jaki program się do niej podłączy. To jedna z największych zalet bazy nad plikiem tekstowym: **reguły są egzekwowane na poziomie magazynu danych**, więc nawet błędny program ich nie obejdzie.

Trwałość również jest rozwiązana: bazy relacyjne mają mechanizm **transakcji**, czyli grupowania operacji w niepodzielną całość. Albo zapiszą się wszystkie, albo żadna. Jeśli w połowie serii zapisów zabraknie prądu, baza wstanie dokładnie w stanie sprzed transakcji. Nie musisz tego teraz rozumieć w szczegółach — moduł 14 poświęcony jest transakcjom w całości — ale zapamiętaj, że to jedna z rzeczy, których samodzielnie w pliku tekstowym nie zbudujesz (albo zbudujesz po tygodniach pracy, gorzej).

### Trzecie podejście: Python rozmawia z bazą bezpośrednio

Bazy relacyjne (SQLite, PostgreSQL, MySQL) są napisane w C i komunikują się przez własny protokół. Python nie umie z nimi rozmawiać bezpośrednio. Potrzebny jest pośrednik — **sterownik** (driver), czyli biblioteka Pythona implementująca ten protokół.

Dla SQLite sterownik jest wbudowany w Pythona i nazywa się `sqlite3`. Zobaczmy pełny, działający przykład: tworzenie tabeli, wstawianie i odczytywanie danych.

```python
# examples/01_raw_sqlite3.py
"""Praca z bazą SQLite bez żadnej biblioteki pośredniej."""

import sqlite3
from datetime import date

# Połączenie = otwarcie pliku bazy (jeśli nie istnieje, zostanie utworzony)
conn = sqlite3.connect("library.db")
cur = conn.cursor()

# 1. Definiujemy strukturę
cur.execute(
    """
    CREATE TABLE IF NOT EXISTS loans (
        id        INTEGER PRIMARY KEY,
        reader    TEXT NOT NULL,
        title     TEXT NOT NULL,
        loan_date TEXT NOT NULL
    )
    """
)

# 2. Wstawiamy dwa wiersze – UWAGA: przez parametry ?, nie przez f-string!
cur.execute(
    "INSERT INTO loans (reader, title, loan_date) VALUES (?, ?, ?)",
    ("alicja", "Wiedźmin", date(2026, 9, 1).isoformat()),
)
cur.execute(
    "INSERT INTO loans (reader, title, loan_date) VALUES (?, ?, ?)",
    ("bob", "Solaris", date(2026, 9, 2).isoformat()),
)

# 3. Zatwierdzamy zmiany. Bez tego nic nie zostanie zapisane!
conn.commit()

# 4. Czytamy – wynik to lista krotek, bo sqlite3 nic nie wie o klasach
for row in cur.execute("SELECT id, reader, title, loan_date FROM loans ORDER BY id"):
    print(row)
    # (1, 'alicja', 'Wiedźmin', '2026-09-01')
    # (2, 'bob', 'Solaris', '2026-09-02')

conn.close()
```

**Jak uruchomić:**

```bash
python examples/01_raw_sqlite3.py     # utworzy plik library.db w bieżącym katalogu
rm library.db                          # posprzątaj, jeśli chcesz zacząć od nowa
```

To już prawdziwa baza danych: z transakcją (`commit`), ze strukturą (`NOT NULL`), z trwałym zapisem. Zauważ jednak dwie rzeczy, które za chwilę staną się bohaterami tego modułu:

1. Wynik zapytania to **krotka** `(1, 'alicja', 'Wiedźmin', '2026-09-01')`. Nie ma nazw. Nie ma typu. Nie ma metod. Żeby dostać tytuł, musisz napisać `row[2]` i **pamiętać**, że tytuł jest na pozycji drugiej.
2. Zapytanie to **string**. Kusi, żeby wstawić do niego zmienną przez f-string. A to prosta droga do katastrofy.

Te dwa problemy — plus trzeci, o którym jeszcze nie mówiliśmy — to dokładnie te problemy, które rozwiązuje SQLAlchemy.

---

## Problem 1: SQL sklejany stringami i wstrzyknięcie SQL

Zacznijmy od najpoważniejszego, bo dotyczy bezpieczeństwa. Wyobraź sobie funkcję, która szuka wypożyczeń po nazwie czytelnika. Programista chce, żeby było „elastycznie”, więc pisze tak:

```python
# examples/01_unsafe_fstring.py
"""⚠️ UWAGA: ten kod jest CELOWO niebezpieczny. Nigdy tak nie pisz!

Służy wyłącznie do zademonstrowania ataku typu SQL Injection.
Nie uruchamiaj go na prawdziwych danych.
"""

import sqlite3


def znajdz_wypozyczenia_niebezpiecznie(reader: str) -> list[tuple]:
    """Wyszukiwanie po czytelniku – wersja ze wstrzyknięciem SQL."""
    conn = sqlite3.connect("library.db")
    #  ZŁO: dane od użytkownika stają się częścią kodu SQL
    sql = f"SELECT id, title FROM loans WHERE reader = '{reader}'"
    print("WYSŁANY SQL:", sql)
    return conn.execute(sql).fetchall()
```

Dla zwykłego wejścia `"alicja"` powstanie taki SQL — i wszystko wygląda dobrze:

```sql
SELECT id, title FROM loans WHERE reader = 'alicja'
```

Ale wejście pochodzi od użytkownika — z formularza na stronie internetowej, z parametru URL, z pliku CSV, z API. A użytkownik może wpisać coś takiego:

```text
x' UNION SELECT username, password_hash FROM users -- 
```

I wtedy wygenerowany SQL staje się tym:

```sql
SELECT id, title FROM loans WHERE reader = 'x' UNION SELECT username, password_hash FROM users -- '
```

Po kolei, co się stało:

- Cudzysłów po `x` **zamknął** literał tekstowy wcześniej, niż zakładał programista. Wyszukiwana wartość skończyła się na pustym `x`.
- Słowo `UNION` dołączyło do wyniku **drugie zapytanie** — pobierające loginy i skróty haseł z zupełnie innej tabeli.
- Dwa myślniki `--` oznaczają w SQL **komentarz**. Wszystko, co następuje dalej (czyli samotny cudzysłów zamykający, który zostawił programista), jest ignorowane.

Efekt: funkcja, która miała zwrócić tytuły wypożyczonych książek, zwraca listę loginów i skrótów haseł. Atakujący jednym wpisaniem w pole wyszukiwania zabrał właśnie bazę użytkowników.

To jest **wstrzyknięcie SQL** (SQL injection, SQLi): sytuacja, w której dane wprowadzone przez użytkownika zostają zinterpretowane jako kod. W OWASP Top 10 — czyli w katalogu najgroźniejszych luk w aplikacjach webowych — wstrzyknięcia zajmują jedno z czołowych miejsc od ponad dwudziestu lat.

> ⚠️ **Pułapka** — „Ale ja nie piszę aplikacji webowej, tylko skrypt do własnych danych”. Wstrzyknięcie SQL nie wymaga internetu. Wystarczy, że dane wejściowe pochodzą z zewnątrz: z pliku konfiguracyjnego, z argumentu `sys.argv`, z pola w arkuszu CSV, z odpowiedzi API. Jeśli kiedykolwiek zamienisz skrypt w usługę dla innych, dług bezpieczeństwa zostanie odziedziczony razem z kodem.

### Dlaczego parametry `?` są bezpieczne

Wróć na moment do przykładu `01_raw_sqlite3.py`. Tam napisaliśmy:

```python
cur.execute(
    "INSERT INTO loans (reader, title, loan_date) VALUES (?, ?, ?)",
    ("alicja", "Wiedźmin", "2026-09-01"),
)
```

Znaki `?` to **parametry wiązane** (bound parameters, placeholders). Działają zupełnie inaczej niż f-string. Program wysyła do bazy **dwie osobne rzeczy**:

1. **Treść zapytania** — szkielet SQL z dziurami.
2. **Wartości** — osobno, jako dane, nigdy nie sklejone z tekstem SQL.

Baza danych najpierw **parsuje i planuje** zapytanie, a dopiero potem podstawia wartości w oznaczone miejsca. Wartość nie może więc „uciec” z literału tekstowego i stać się słowem kluczowym SQL. Nawet jeśli użytkownik wpisze `x' UNION SELECT ...`, baza potraktuje to jako **jeden długi, dziwny string** do porównania z kolumną `reader` — i po prostu nic nie znajdzie.

To podejście nazywa się **parametryzacją zapytań** i jest fundamentalne. Od tego miejsca w całym kursie trzymaj się jednej żelaznej reguły:

> **Nigdy nie wstawiaj danych do tekstu SQL. Zawsze przekazuj je jako parametr.**

W SQLAlchemy ta zasada nie jest „dobrą praktyką, o której trzeba pamiętać” — jest **wbudowana w sam sposób konstruowania zapytań**. W module 04 zobaczysz, że wszystkie warunki pisze się jako `where(Book.title == zmienna)`, a SQLAlchemy sam generuje dla nich parametry wiązane. Nie ma tam miejsca, w którym naturalnie wpisałoby się f-string. To przykład ogólniejszej idei, którą przewija się w całym kursie:

> 💡 **Analogia** — Ręczne pisanie SQL-a z f-stringiem jest jak tłumaczenie zdania z polskiego na angielski, wklejając do środka cudze słowa bez sprawdzenia kontekstu: „Widziałem [DROP TABLE] w parku”. Parametryzacja to wręczanie tłumaczowi dwóch oddzielnych rzeczy: „zdanie brzmi tak: «Widziałem ___ w parku»” oraz „w puste miejsce wstaw słowo *ptaka*”. Tłumacz sam zdecyduje, w jakim przypadku je postawić — bo zna gramatykę, a ty nie musisz.

Zauważ, że problem wstrzyknięcia nie jest winą `sqlite3`. Biblioteka ostrzega: *„You can only execute one statement at a time”* i wymaga przekazywania parametrów osobno. Problem jest w programiście, który sięgnął po f-string, bo „tak prościej”. SQLAlchemy wyręcza go w tej decyzji — i to jest jej pierwszą wielką wartością.

> 🔬 **Pod maską** — Sprawdziliśmy to w module 01 na SQLite. Warto wiedzieć, że tego samego problemu nie da się rozwiązać fragmentarycznie: parametryzacja albo jest wszędzie, albo dziura zostaje w jednym miejscu i wystarcza do ataku. Moduł 04 pokaże, jak sprawdzić wygenerowany SQL i upewnić się, że dane naprawdę są wiązane, a nie wklejone.

---

## Problem 2: ręczne mapowanie wierszy na obiekty

Drugi problem nie zagraża bezpieczeństwu — zagraża zdrowiu psychicznemu programisty. Wróć myślą do zapytania i jego wyniku:

```python
row = (1, "alicja", "Wiedźmin", "2026-09-01")
print(row[2])       # 'Wiedźmin' – ale skąd mam wiedzieć, że to indeks 2?
print(row[3][:4])   # '2026' – prawda, data jest stringiem, więc sam ją parsuję
```

Wiersz z bazy to krotka bez nazw. Trzy konsekwencje:

1. **Kruchość.** Jeśli ktoś doda kolumnę na początku `SELECT`, wszystkie indeksy się przesuwają. Kod nadal się uruchomi, ale będzie czytał złe pola — a błąd tego typu jest bardzo trudny do znalezienia, bo nie rzuca wyjątku.
2. **Brak typów.** Data to string, kwota to string, prawda/fałsz to liczba 0/1. Trzeba konwertować ręcznie, w każdym miejscu osobno.
3. **Brak zachowania.** Krotka nie ma metod. Nie możesz napisać `loan.ile_dni_temu()`. Cała logika musi stać *obok* danych, w luźnych funkcjach, i za każdym razem na nowo „rozpakować” surowy wiersz.

Programiści radzą sobie, pisząc własne klasy i własne funkcje mapujące. Oto obraz tego, jak to wygląda:

```python
# examples/01_manual_mapping.py
"""Jak wygląda ORM napisany ręcznie. Tak, naprawdę tak się to robiło."""

import sqlite3
from dataclasses import dataclass
from datetime import date


@dataclass
class Loan:
    """Ręczna klasa 'encja' – odpowiednik wiersza z tabeli loans."""
    id: int | None
    reader: str
    title: str
    loan_date: date

    def ile_dni_temu(self, dzis: date | None = None) -> int:
        dzis = dzis or date.today()
        return (dzis - self.loan_date).days


def wiersz_na_loan(row: tuple) -> Loan:
    """Ręczne mapowanie: krotka z bazy -> obiekt Pythona."""
    return Loan(
        id=row[0],
        reader=row[1],
        title=row[2],
        loan_date=date.fromisoformat(row[3]),  # ręczna konwersja typu!
    )


def loan_na_wiersz(loan: Loan) -> tuple:
    """Ręczne mapowanie w drugą stronę: obiekt -> parametry do SQL."""
    return (loan.reader, loan.title, loan.loan_date.isoformat())


def zapisz(conn: sqlite3.Connection, loan: Loan) -> None:
    """Ręczne wstawienie obiektu do bazy."""
    conn.execute(
        "INSERT INTO loans (reader, title, loan_date) VALUES (?, ?, ?)",
        loan_na_wiersz(loan),
    )


if __name__ == "__main__":
    conn = sqlite3.connect(":memory:")   # baza w pamięci – znika po zamknięciu
    conn.execute(
        """
        CREATE TABLE loans (
            id        INTEGER PRIMARY KEY,
            reader    TEXT NOT NULL,
            title     TEXT NOT NULL,
            loan_date TEXT NOT NULL
        )
        """
    )
    zapisz(conn, Loan(id=None, reader="alicja", title="Wiedźmin",
                      loan_date=date(2026, 9, 1)))
    conn.commit()

    (row,) = conn.execute("SELECT id, reader, title, loan_date FROM loans").fetchall()
    loan = wiersz_na_loan(row)
    print(loan)                       # Loan(id=1, reader='alicja', title='Wiedźmin', ...)
    print(loan.ile_dni_temu(date(2026, 9, 10)))   # 9

    conn.close()
```

**Jak uruchomić:**

```bash
python examples/01_manual_mapping.py
```

Popatrz, ile pracy wymagało coś, co powinno być oczywiste: *„wczytaj wiersz i zrób z niego obiekt”*. Trzy funkcje, ręczne indeksy `row[0]`, `row[3]`, ręczne `date.fromisoformat`, ręczne odwzorowanie w drugą stronę. A to tylko cztery kolumny! Teraz wyobraź sobie tabelę z dwudziestoma kolumnami, sześć tabel ze sobą powiązanych, i pięćdziesiąt zapytań w aplikacji. Każda zmiana w schemacie bazy oznacza ręczną poprawkę kilkudziesięciu miejsc w kodzie.

> 💡 **Analogia** — Ręczne mapowanie to jak bibliotekarz, który za każdym razem, gdy ktoś prosi o książkę, dostaje z magazynu **kartkę z opisem**: *„pozycja 1: powieść, pozycja 2: fantasy, pozycja 3: 420 stron”*. Bibliotekarz musi za każdym razem tłumaczyć tę kartkę na myśl „to jest *Wiedźmin*”. ORM (Object-Relational Mapper, mapowanie obiektowo-relacyjne) to bibliotekarz, który **dostaje samą książkę** — od razu z tytułem, autorem i stronami, gotową do podania czytelnikowi.

I to jest druga wielka wartość SQLAlchemy: **mapowanie** (mapping). Zamiast pisać `row[2]`, piszesz `book.title`. Zamiast `date.fromisoformat(row[3])`, piszesz `book.loan_date` i dostajesz obiekt `datetime.date`. Zamiast pamiętać, że klucz obcy czytelnika jest w kolumnie `reader_id`, piszesz `loan.reader` i dostajesz **obiekt czytelnika** — z jego nazwiskiem, adresem i historią wypożyczeń.

---

## Problem 3: jedna baza to nie każda baza

Trzeci problem jest najbardziej podstępny, bo nie ujawnia się na twoim komputerze. Ujawnia się dopiero na produkcji.

Załóżmy, że piszesz aplikację, która na etapie nauki i testów działa na SQLite, a na serwerze ma działać na PostgreSQL. Wygląda to rozsądnie: SQLite nie wymaga instalacji, PostgreSQL jest solidniejszy pod obciążeniem. Problem w tym, że **SQL w każdej bazie jest trochę inny**. To nie jest jeden język — to rodzina dialektów, jak polski, czeski i słowacki: bliskie, ale nie identyczne.

Kilka realnych różnic, na które natkniesz się w pierwszym tygodniu pracy:

| Zagadnienie | SQLite | PostgreSQL |
|---|---|---|
| **Parametry wiązane** | `?` (qmark) | `%(nazwa)s` (pyformat) lub `$1` |
| **Cytowanie identyfikatorów** | `"nazwa"` działa, ale `[nazwa]` też | tylko `"nazwa"` |
| **Auto-numerowanie klucza** | `INTEGER PRIMARY KEY` samo się numeruje | wymaga typu `SERIAL` lub `GENERATED ... AS IDENTITY` |
| **Typ logiczny** | nie ma; używa się `INTEGER` 0/1 | natywny `BOOLEAN` |
| **Typ daty** | `TEXT` / `REAL` (brak prawdziwego typu daty) | natywny `DATE`, `TIMESTAMP` |
| **Wielkość liter w LIKE** | domyślnie wrażliwe (poza ASCII) | `LIKE` wrażliwe, `ILIKE` niewrażliwe |
| **RETURNING** | obsługiwane od wersji 3.35 (2021) | obsługiwane od zawsze |
| **Funkcje okna** | obsługiwane od 3.25 | obsługiwane od zawsze |
| **Współbieżność** | jeden piszący naraz | wielu piszących (MVCC) |
| **Operacje na JSON-ie** | podstawowe, od 3.38 (2.1: nowy `JSONB`) | pełne, z indeksami GIN i operatorami `@>`, `->>` |

Jeśli napiszesz SQL ręcznie, musisz napisać **dwie wersje każdego nietrywialnego zapytania** — a przy trzech bazach (SQLite do testów, PostgreSQL na produkcji, MySQL u klienta) trzy wersje. I pilnować, żeby się nie rozjechały.

> ⚠️ **Pułapka** — „Będę pisać SQL tylko pod SQLite, przecież działa”. To najczęstszy scenariusz, w którym projekt umiera. Testy przechodzą na SQLite, a przy pierwszym uruchomieniu na PostgreSQL sypie się połowa zapytań. Baza testowa, która zachowuje się inaczej niż produkcyjna, daje **fałszywe poczucie bezpieczeństwa** — gorzej niż brak testów. Moduł 18 pokaże, jak to rozwiązać rozsądnie (Testcontainers, PostgreSQL w Dockerze).

I tu dochodzimy do trzeciej wartości SQLAlchemy: **przenośność** (portability). Piszesz jedno wyrażenie Pythona. SQLAlchemy ma w środku **dialekty** — opis, jak to samo wyrażenie zapisać w danym języku SQL. Chcesz SQLite? Będzie `INTEGER PRIMARY KEY`. Chcesz PostgreSQL? Będzie `SERIAL` i `RETURNING`. Chcesz MySQL? Będzie `AUTO_INCREMENT`. Twój kod aplikacji się nie zmienia.

> 🔬 **Pod maską** — Zobacz to na żywo. Poniższy kod buduje **jedno** wyrażenie i kompiluje je do dwóch różnych dialektów. Nie łączy się z żadną bazą — samo tłumaczenie.

```python
# examples/01_compile_dialects.py
"""Jedno wyrażenie, dwa różne SQL-e. Serce przenośności SQLAlchemy."""

from sqlalchemy import Integer, String, column, select, table
from sqlalchemy.dialects import postgresql, sqlite

# Lekka deklaracja 'ad hoc' – w prawdziwym projekcie użyjemy klas (moduł 07)
books = table("books", column("id", Integer), column("title", String))

stmt = (
    select(books.c.id, books.c.title)
    .where(books.c.title.ilike("wied%"))   # ILIKE – 'i' jak insensitive
    .limit(5)
)

print("=== SQLite ===")
print(stmt.compile(dialect=sqlite.dialect()))
# SELECT books.id, books.title
# FROM books
# WHERE lower(books.title) LIKE lower(?)      <-- ręczna emulacja ILIKE
# LIMIT ? OFFSET ?

print()
print("=== PostgreSQL ===")
print(stmt.compile(dialect=postgresql.dialect()))
# SELECT books.id, books.title
# FROM books
# WHERE books.title ILIKE %(title_1)s         <-- natywny ILIKE
# LIMIT %(param_1)s OFFSET %(param_2)s
```

**Jak uruchomić:**

```bash
pip install "SQLAlchemy>=2.0"
python examples/01_compile_dialects.py
```

Trzy różnice widoczne od razu, bez uruchamiania bazy:

1. **`ILIKE` vs `lower(...) LIKE lower(...)`** — PostgreSQL ma natywny operator niewrażliwego porównania; SQLite nie ma, więc SQLAlchemy **emuluje** go, owijając obie strony w `lower()`. Ten sam kod, dwa różne (i oba **poprawne**) SQL-e.
2. **Styl parametrów** — `?` dla SQLite, `%(title_1)s` dla PostgreSQL. SQLAlchemy zna wymagania sterownika i sam wybiera właściwy format. Ty nigdy nie wpisujesz tych znaków ręcznie.
3. **Nazwy parametrów są generowane** — `title_1`, `param_1`. To dowód, że wartości nie są wklejane w tekst, a przekazywane osobno. Mechanizm bezpieczeństwa z poprzedniej sekcji działa tu automatycznie.

To jest dobry moment na zdanie, które warto zapamiętać na cały kurs:

> 🧠 **Dlaczego tak jest** — SQLAlchemy **nie ukrywa SQL-a**. Ona go **generuje**. Nadal potrzebujesz rozumieć SQL, żeby wiedzieć, co się dzieje — i cały ten kurs będzie konsekwentnie pokazywał wygenerowany SQL przy każdym zapytaniu. Różnica jest taka, że piszesz go raz, w Pythonie, bezpiecznie i przenośnie, a nie za każdym razem od nowa, ręcznie i z błędami.

---

## SQLAlchemy jako tłumacz

Podsumujmy trzy problemy i ich rozwiązania:

| Problem | Rozwiązanie w SQLAlchemy |
|---|---|
| Sklejanie SQL ze stringów → wstrzyknięcia | Zapytania budowane z wyrażeń Pythona; parametry wiązane generowane automatycznie |
| Ręczne mapowanie wierszy na obiekty | Deklaratywne modele: jedna klasa Pythona = jedna tabela, jeden obiekt = jeden wiersz |
| Różnice między bazami | Warstwa dialektów: jedno wyrażenie kompilowane do SQL właściwego dla bazy |

Najlepsza analogia do całości jest jedna:

> 💡 **Analogia** — SQLAlchemy to **tłumacz przysięgły** stojący między tobą a bazą danych. Ty mówisz po Pythonie: *„dodaj nowego czytelnika o nazwisku Kowalska”*. Baza rozumie tylko po SQL-u. Tłumacz zamienia twoje zdanie na poprawne SQL w dialekcie tej konkretnej bazy — a gdy baza odpowie (*„dodałem, oto numer identyfikacyjny: 42”*), tłumacz zamienia odpowiedź z powrotem na obiekt Pythona. Tłumacz **nie decyduje za ciebie**, co chcesz powiedzieć — nadal musisz wiedzieć, co chcesz osiągnąć. Ale nie musisz znać gramatyki drugiego języka.

Trzy rzeczy, które warto od razu rozdzielić, bo początkujący je mylą:

**Co robi SQLAlchemy:**
- buduje tekst SQL z wyrażeń Pythona (bezpiecznie, z parametrami),
- tłumaczy SQL na dialekt konkretnej bazy,
- zamienia wiersze na obiekty i odwrotnie,
- śledzi zmiany w obiektach i wie, co trzeba zapisać,
- zarządza pulą połączeń do bazy i transakcjami.

**Co robi baza danych:**
- przechowuje dane na dysku,
- egzekwuje ograniczenia (`NOT NULL`, klucze obce, unikalność),
- planuje i wykonuje zapytania (wybiera indeksy, łączy tabele),
- zapewnia transakcyjność i współbieżność,
- pilnuje uprawnień.

**Co robi programista (czyli ty):**
- decyduje, jakie dane i w jakiej strukturze przechowywać,
- pisze logikę biznesową,
- wybiera, kiedy użyć ORM, a kiedy zejść niżej (Core, `text()`),
- projektuje transakcje,
- mierzy wydajność i optymalizuje.

> 🧠 **Dlaczego tak jest** — Ta granica bywa rozmyta w rozmowach. „SQLAlchemy jest wolna, bo generuje głupie zapytania” — nie, to baza planuje zapytanie. „Baza nie pozwoliła mi zapisać obiektu” — nie, to SQLAlchemy nie wysłała `INSERT`, bo obiekt nie był podpięty do sesji. Rozróżnianie tych trzech warstw uratuje cię dziesiątki razy w debugowaniu (a moduł 24 poświęcony jest w całości czytaniu błędów).

---

## Dwie warstwy: Core i ORM

SQLAlchemy to **jedna biblioteka, ale dwa sposoby pracy**. Możesz używać ich osobno albo razem, w tym samym pliku.

### Core — skrzynka z narzędziami

**Core** to warstwa wyrażeń SQL. Nie ma tu klas mapowanych na tabele. Jest `Table` (opis tabeli), `select()` (zapytanie), `insert()`, `update()`, `delete()`. Pracujesz na **wierszach i kolumnach**, a nie na obiektach.

> 💡 **Analogia** — Core to **skrzynka z narzędziami** stolarza: dłuta, młotki, poziomica. Każde narzędzie robi dokładnie jedną rzecz, dobrze i przewidywalnie. Możesz zrobić dowolnie skomplikowaną szafkę — ale musisz sam wiedzieć, których narzędzi użyć i w jakiej kolejności.

Zalety Core:
- **Pełna kontrola** nad każdym fragmentem SQL-a.
- **Przewidywalność** — wiesz dokładnie, jakie zapytanie poleci.
- **Wydajność** — brak narzutu związanego ze śledzeniem obiektów.
- **Zapytania analityczne** — złożone agregacje, funkcje okna, CTE pisze się tu naturalnie.
- **Praca z istniejącą bazą** — refleksja pozwala czytać schemat, którego nie tworzyłeś.

Cena:
- Wyniki to wiersze, nie obiekty: `row.title`, nie `book.title` (choć Core też daje nazwane wiersze — o tym w module 04).
- Sam musisz pamiętać o kolejności operacji, o transakcjach, o konwersjach typów.
- Nie ma „śledzenia zmian” — jeśli zmienisz dane w pamięci, musisz sam napisać `update()`.

### ORM — automat z pilotem

**ORM** to warstwa zbudowana **na Core**. Dodaje mapowanie klas Pythona na tabele oraz **sesję** (Session) — obiekt, który śledzi obiekty, wie, które są nowe, które zmienione, i sam decyduje, kiedy wysłać `INSERT` albo `UPDATE`.

>  **Analogia** — ORM to **automat z pilotem** zamiast skrzynki z narzędziami. Mówisz: „zapisz ten obiekt”, „znajdź wszystkie książki tego autora”, „usuń to zamówienie z pozycjami”. Automat sam wybierze narzędzia, sam je poukłada w odpowiedniej kolejności, sam dopilnuje, żeby nie zapisać połowy zmiany. Wygodnie i szybko — ale jeśli chcesz zrobić coś bardzo niestandardowego, musisz zajrzeć do skrzynki pod spodem.

Zalety ORM:
- **Czytelność** — `book.author.name` zamiast trzech ręcznych JOIN-ów.
- **Model domenowy** — obiekty mają metody i zachowanie, a nie tylko dane.
- **Automatyczne zapisywanie** — zmieniasz atrybut, sesja wie o zmianie.
- **Relacje** — klucze obce stają się naturalnymi referencjami do obiektów.
- **Spójność** — jedna transakcja, jedna tożsamość obiektu (`identity map`, moduł 08).

Cena:
- Trzeba zrozumieć **cykl życia sesji** (moduł 08) — najczęstsze źródło błędów początkujących.
- Łatwo nieświadomie napisać N+1 zapytań (moduł 11) — klasyczna pułapka wydajnościowa.
- Abstrakcja „ukrywa” część SQL-a, więc można nie zauważyć, że coś jest wolne.

> ⚠️ **Pułapka** — Najczęstsze nieporozumienie brzmi: „użyję ORM, więc nie muszę znać SQL-a”. To nieprawda i to kosztowna nieprawda. ORM **nie zwalnia z wiedzy o SQL-u** — zwalnia z pisania go ręcznie. Wyobraź sobie kierowcę, który twierdzi, że skoro ma automatyczną skrzynię biegów i nawigację, nie musi rozumieć, co się dzieje, gdy samochód zaczyna się ślizgać na zakręcie. Wszystkie problemy wydajnościowe, wszystkie błędy „zapytanie zwraca nie to, co trzeba” i cała diagnostyka opierają się na umiejętności przeczytania wygenerowanego SQL-a. Dlatego w tym kursie **każdy przykład ORM-owy ma podany wygenerowany SQL**.

### Tabela decyzyjna: warstwa → kiedy używać → przykład

| Warstwa | Kiedy używać | Przykład zastosowania |
|---|---|---|
| **Surowy SQL** / `text()` | Skomplikowane, ręcznie dopracowane zapytania; praca z rzeczami, których SQLAlchemy nie modeluje (procedury składowane, komendy administracyjne, `VACUUM`, `EXPLAIN`); migracje danych w Alembicu | `ALTER TABLE ...`, `CREATE INDEX CONCURRENTLY`, własne `EXPLAIN ANALYZE` |
| **SQLAlchemy Core** | Raporty i agregacje; operacje masowe (import 100 tys. wierszy); zapytania do istniejącej bazy używanej także przez inne aplikacje; praca bez modeli obiektowych; pełna kontrola nad `INSERT`/`UPDATE` | „Sprzedaż w podziale na miesiące i kategorię”, „Zaktualizuj ceny wszystkich produktów o 5%” |
| **SQLAlchemy ORM** | Zwykła logika aplikacji: tworzenie, wyszukiwanie, modyfikowanie encji; praca na grafie obiektów z relacjami; walidacja na poziomie modelu; przypadki użycia dla użytkownika | „Zarejestruj nowego czytelnika”, „Pokaż wypożyczenia tego czytelnika z tytułami książek”, „Złóż zamówienie z pozycjami” |

Doprecyzujmy „operacje masowe”, bo to typowy punkt nieporozumień. ORM jest zoptymalizowany pod **pojedyncze encje i ich relacje**. Gdy musisz wstawić 200 000 wierszy z pliku CSV, śledzenie każdego obiektu w sesji to koszt, którego nie potrzebujesz. Wtedy używamy Core-owego `insert().values([...])` — w module 05 zobaczysz to obok siebie z `add_all()` i porównasz czasy.

Praktyczna reguła na start, zanim poznasz niuanse: **domyślnie ORM; schodź do Core, gdy masz dobre uzasadnienie**. Nie odwrotnie. Próba napisania całej aplikacji w Core mści się na czytelności, a próba napisania importu miliona rekordów przez ORM — na wydajności.

---

## Trzy poziomy abstrakcji — to samo zadanie na trzy sposoby

Najlepiej różnicę widać na identycznym zadaniu. **Cel:** zapisz w bazie informację, że czytelnik *alicja* wypożyczył *Wiedźmina* dnia 2026-09-01, a następnie odczytaj tytuły jej wypożyczeń.

### Poziom 0: surowy SQL (przez `sqlite3`)

```python
cur.execute(
    "INSERT INTO loans (reader, title, loan_date) VALUES (?, ?, ?)",
    ("alicja", "Wiedźmin", "2026-09-01"),
)
conn.commit()

rows = cur.execute(
    "SELECT title FROM loans WHERE reader = ?", ("alicja",)
).fetchall()
# [('Wiedźmin',)]
print([row[0] for row in rows])   # ['Wiedźmin']
```

- Znasz SQL? Tak. Znasz tabelę i jej kolumny? Musisz.
- Bezpieczne? Tak — bo **pamiętałeś** o parametrach `?`.
- Ile rzeczy trzeba pamiętać? Trzy: parametryzacja, `commit`, `row[0]`.

### Poziom 1: SQLAlchemy Core

```python
with engine.begin() as conn:                     # transakcja automatycznie
    conn.execute(
        insert(loans).values(
            reader="alicja", title="Wiedźmin",
            loan_date=date(2026, 9, 1),          # prawdziwy obiekt date!
        )
    )
    rows = conn.execute(
        select(loans.c.title).where(loans.c.reader == "alicja")
    ).all()
    print([r.title for r in rows])               # ['Wiedźmin']
```

- `engine.begin()` **sam** otwiera transakcję i sam ją zatwierdza przy wyjściu z bloku — nie da się zapomnieć o `commit`.
- Bezpieczeństwo jest **domyślne** — nie musisz pamiętać o `?`, bo nie ma tam miejsca na wklejenie wartości.
- Data jest **obiektem `date`**, nie stringiem. Konwersją zajmuje się typ kolumny.
- Wynik ma **nazwy**: `r.title` zamiast `r[0]`.
- **Nadal** musisz znać tabelę `loans` i jej kolumny — definicja jest w `Table(...)` z modułu 03.

### Poziom 2: SQLAlchemy ORM

```python
with Session(engine) as session, session.begin():
    session.add(Loan(reader="alicja", title="Wiedźmin",
                     loan_date=date(2026, 9, 1)))
    # nie ma INSERT-a – sesja sama ustali, że trzeba go wysłać

with Session(engine) as session:
    loans_found = session.scalars(
        select(Loan).where(Loan.reader == "alicja")
    ).all()
    print([loan.title for loan in loans_found])   # ['Wiedźmin']
    # loans_found[0] to obiekt klasy Loan – z metodami i relacjami
```

- Zamiast wiersza dostajesz **obiekt klasy `Loan`**.
- Nie piszesz `INSERT` — mówisz `add()` i sesja decyduje, jak to zrealizować.
- `Loan.reader` to **atrybut instrumentowany** (`instrumented attribute`) — czyli atrybut klasy, o którym SQLAlchemy wie i potrafi go przełożyć na kolumnę SQL. To pojęcie pojawia się w module 07.
- Jeśli `Loan` ma relację do `Reader`, możesz napisać `loan.reader` i dostać obiekt czytelnika — **bez pisania JOIN-a**.

### Diagram trzech poziomów

```text
ZADANIE: „zapisz wypożyczenie książki 'Wiedźmin' przez alicję”

┌─ POZIOM 0: surowy SQL ────────────────────────────────────────────┐
│  cur.execute("INSERT INTO loans (...) VALUES (?, ?, ?)", dane)    │
│  conn.commit()                                                    │
│  Znasz SQL: musi. Znasz tabelę: musi. Bezpieczeństwo: ręczne.     │
│  Wynik: krotka (1, 'alicja', 'Wiedźmin', '2026-09-01')            │
├─ POZIOM 1: SQLAlchemy Core ───────────────────────────────────────┤
│  with engine.begin() as conn:                                     │
│      conn.execute(insert(loans).values(reader="alicja", ...))     │
│  Znasz tabelę: musi. Bezpieczeństwo: domyślne.                    │
│  Wynik: Row(title='Wiedźmin')  ← nazwane pola, typy zachowane     │
├─ POZIOM 2: SQLAlchemy ORM ────────────────────────────────────────┤
│  session.add(Loan(reader="alicja", title="Wiedźmin", ...))        │
│  Znasz tabele: nie – znasz KLASY. Bezpieczeństwo: domyślne.       │
│  Wynik: Loan(id=1, reader='alicja', title='Wiedźmin', ...)        │
│         ← obiekt z metodami, relacjami, zachowaniem               │
└───────────────────────────────────────────────────────────────────┘
```

Ten sam efekt, trzy różne koszty. Koszt poznawczy (ile musisz wiedzieć), koszt pisania (ile linii kodu) i koszt elastyczności (jak łatwo zrobić coś nietypowego). Wszystkie trzy poziomy są **legalne i potrzebne** — SQLAlchemy nie zmusza cię do wyboru jednego na stałe. W module 06 zobaczysz zapytania analityczne, które czyściej wyglądają w Core, a w module 22 — endpointy API, gdzie ORM wygrywa bezapelacyjnie.

> 🧪 **Ćwiczenie** — Wyobraź sobie, że musisz policzyć średnią liczbę dni przetrzymania książki w podziale na kategorię, dla 300 000 wypożyczeń. Który poziom wybierzesz i dlaczego? Odpowiedź rozwijamy w rozwiązaniach ćwiczeń na końcu modułu.

---

## Filozofia „DBAPI + dialekt + kompilator”

Jest jeden fragment architektury SQLAlchemy, który warto zrozumieć już teraz, bo bez niego nie zrozumiesz żadnego komunikatu o błędzie. Oto pełna droga, jaką przebywa twoje polecenie od kodu Pythona do dysku.

```text
    Twój kod Pythona
    ┌───────────────────────────────────────────────────────────┐
    │  loan = Loan(reader="alicja", title="Wiedźmin", ...)      │
    │  session.add(loan)                                        │
    └───────────────────────────────────────────────────────────
                            │
       ┌────────────────────┴────────────────────
       │                                         │
   (ORM) mapowanie obiektu na wiersz        (Core) wyrażenie SQL
       │  Loan -> {"reader": ..., "title": ...}   Insert(...)
       └────────────────────┬────────────────────┘
                            │
                            ▼
    ┌───────────────────────────────────────────────────────────┐
    │  KOMPILATOR + DIALEKT   (warstwa sqlalchemy.sql / dialect)│
    │  Wyrażenie Pythona  →  tekst SQL + słownik parametrów     │
    │  "INSERT INTO loans (reader, title) VALUES (?, ?)"        │
    │  {"reader": "alicja", "title": "Wiedźmin"}                │
    └───────────────────────────────────────────────────────────
                            │
                            ▼
    ┌───────────────────────────────────────────────────────────
    │  DBAPI   (sqlite3 | psycopg | psycopg2 | asyncpg | ...)   │
    │  Biblioteka Pythona mówiąca protokołem danej bazy         │
    └───────────────────────────────────────────────────────────┘
                            │  protokół sieciowy (TCP) / plik
                            ▼
    ┌───────────────────────────────────────────────────────────
    │  SERWER BAZY DANYCH   (SQLite | PostgreSQL | MySQL)       │
    │  Parsowanie → plan → wykonanie → odpowiedź                │
    ───────────────────────────────────────────────────────────┘
```

Trzy elementy z tego diagramu wymagają wyjaśnienia.

### DBAPI — wspólny język sterowników Pythona

**DBAPI** (DataBase API) to nie biblioteka, tylko **standard**. PEP 249 opisuje zestaw metod, które musi mieć każdy sterownik bazy danych dla Pythona: `connect()`, `cursor()`, `execute()`, `fetchall()`, `commit()`, `rollback()`. Dlatego `sqlite3` i `psycopg` mają taki sam interfejs — i dlatego SQLAlchemy może obsługiwać dziesiątki baz, nie pisząc osobnego kodu dla każdej. Wystarczy, że sterownik przestrzega standardu.

Zapamiętaj tę nazwę, bo pojawi się w dwóch ważnych miejscach:

1. **Błędy.** Komunikat `sqlite3.OperationalError: no such table: loans` pochodzi **z DBAPI**, a nie z SQLAlchemy. SQLAlchemy go przechwytuje, opakowuje w `OperationalError` i dodaje kontekst (jakie zapytanie i jakie parametry zostały wysłane). Umiejętność rozdzielenia „to mówi sterownik” od „to mówi SQLAlchemy” to podstawa debugowania.
2. **Sterowniki.** W adresie połączenia (`02_srodowisko_i_engine.md`) zawsze wybierasz sterownik: `postgresql+psycopg://` znaczy „PostgreSQL przez sterownik `psycopg`”. Zmiana `+psycopg` na `+psycopg2` to zmiana DBAPI przy niezmienionym reszcie kodu.

### Dialekt — opis różnic między bazami

**Dialekt** (dialect) to klasa opisująca jedną konkretną bazę i jeden sterownik. Wie:

- jak nazywają się typy danych (`String(50)` → `VARCHAR(50)` w PostgreSQL, `VARCHAR(50)` w SQLite, ale `NUMBER` w Oracle),
- jak wygląda auto-numerowanie klucza,
- jakiego stylu parametrów używa sterownik (`?`, `%(name)s`, `$1`),
- czego baza nie umie, a co trzeba emulować (jak zobaczyliśmy wyżej z `ILIKE`),
- jakie komendy są dostępne natywnie (`RETURNING`, `ON CONFLICT`, `LATERAL`).

Wybierasz dialekt **raz** — w adresie połączenia:

```python
create_engine("sqlite:///library.db")                    # dialekt: sqlite
create_engine("postgresql+psycopg://user:pw@localhost/db")  # dialekt: postgresql
```

Od tego momentu cały kod działa bez zmian. To jest sensowna definicja przenośności: **kod aplikacji nie wie, z jaką bazą rozmawia**.

> 🆕 **SQLAlchemy 2.1** — Dwie zmiany dotyczące sterowników, o których warto wiedzieć, choć nie wchodzimy w szczegóły:
> - Przy adresie `postgresql://` (bez jawnego `+sterownik`) domyślnym sterownikiem jest teraz **`psycopg`** (wersja 3), a nie starszy `psycopg2`. Analogicznie `oracle://` domyślnie wybiera `oracledb`. W wersji 2.0 domyślnym wyborem był `psycopg2`.
> - `greenlet` — biblioteka potrzebna do trybu asynchronicznego (moduł 15) — **nie jest już instalowana automatycznie**; trzeba jawnie `pip install "sqlalchemy[asyncio]"`.
>
> Praktyczny wniosek: jeśli pracujesz na 2.0, po aktualizacji do 2.1 warto **jawnie** dopisać sterownik w adresie (`postgresql+psycopg://`), żeby nie zależeć od domyślnej wartości.

### Kompilator — moment, w którym Python staje się SQL-em

**Kompilacja** to zamiana wyrażenia SQLAlchemy (obiekt `Insert`, `Select`, `Update`) na tekst SQL plus słownik parametrów. Dzieje się to **przed** wysłaniem do bazy, zwykle tuż przed wykonaniem — i jest cache'owana (moduł 17 pokaże, jak to przyspiesza pracę).

To dlatego można zapytać: „jaki SQL zostanie wysłany?” bez łączenia się z bazą:

```python
print(stmt)          # niepełny SQL – parametry jako :x
print(stmt.compile(dialect=postgresql.dialect()))   # SQL w danym dialekcie
```

Ta możliwość jest jedną z najcenniejszych rzeczy w SQLAlchemy. Dzięki niej:

- **uczysz się** — widzisz, co naprawdę robi twoje wyrażenie, i jednocześnie uczysz się SQL-a;
- **debugujesz** — jeśli wynik jest dziwny, patrzysz na zapytanie;
- **optymalizujesz** — widzisz, czy nie brakuje warunku albo czy nie ma niepotrzebnego JOIN-a;
- **piszesz testy** — możesz testować wygenerowany SQL bez uruchamiania bazy.

W tym kursie pojawia się to w każdej sekcji oznaczonej:

> 🔬 **Pod maską** — tak oznaczamy miejsce, gdzie pokazujemy wygenerowany SQL. Czytaj te bloki uważnie — to najszybsza droga do biegłości w SQL-u, jaką znam.

---

## Kiedy NIE używać ORM

Uczciwy kurs musi powiedzieć nie tylko, kiedy narzędzia użyć, ale i kiedy go odłożyć na bok. ORM nie jest uniwersalnym rozwiązaniem. Oto sytuacje, w których Core albo nawet `text()` z ręcznie napisanym SQL-em jest lepszym wyborem.

### Raporty analityczne i hurtownie danych

Zapytanie typu „przychód według miesiąca, kohorty klientów i kategorii produktu, z porównaniem rok do roku, w 12 kolumnach” w ORM staje się nieczytelne. Trzeba użyć `func.*`, `case()`, grupowań, CTE i funkcji okna — a wtedy cała wartość ORM (mapowanie obiektów) znika, bo nikt nie chce zwracać tego jako encji. To zapytanie zwraca **liczby, nie obiekty**.

```text
ORM pomaga, gdy:  „daj mi obiekt Zamówienie z jego pozycjami”
ORM przeszkadza:  „daj mi 14 kolumn zagregowanych w 3 wymiarach”
```

W takiej sytuacji Core-owe `select()` z kolumnami albo wręcz `text()` z ręcznie dopracowanym SQL-em jest czytelniejszy i szybszy. To typowe dla narzędzi BI, dashboardów i zadań ETL.

### Operacje masowe (ETL, import, migracje danych)

Wstawianie miliona wierszy przez `add_all()` oznacza, że sesja musi utworzyć milion obiektów w pamięci, przypisać im tożsamości, śledzić stan każdego z nich i wygenerować dla nich instrukcje. To ogromny narzut. Core-owy `insert().values([...])` albo natywny `COPY` w PostgreSQL jest o rząd wielkości szybszy. W module 17 zobaczysz konkretne pomiary.

### Praca z istniejącą bazą, której nie kontrolujesz

Zdarza się integracja z bazą legacy, używaną przez trzy inne aplikacje, z tabelami o nazwach typu `T_ORD_HDR` i kolumnami bez typów. Mapowanie tego na ładne klasy bywa kosztowne i mało sensowne — szczególnie gdy i tak używasz tylko 4 kolumn z 60. Core z refleksją (moduł 03) pozwala czytać schemat bez budowania modelu.

### Procedury składowane, specyficzne funkcje, komendy administracyjne

Bazy oferują mnóstwo rzeczy, które nie mapują się na model obiektowy: `EXPLAIN`, `VACUUM`, tworzenie indeksów, wywoływanie procedur składowanych, partycjonowanie. Tu ORM nie ma nic do zaproponowania — używasz `text()` z parametrami, zachowując bezpieczeństwo, ale pisząc SQL.

### Ekstremalna optymalizacja

Gdy wiesz, że w danym miejscu liczy się każda mikrosekunda — np. w handlerze wywoływanym 50 000 razy na sekundę — narzut mapowania obiektów może być odczuwalny. Ale uwaga na kolejność: **najpierw zmierz**. W module 17 zobaczysz, że w 95% przypadków wąskim gardłem nie jest mapowanie, a liczba rund do bazy (N+1) lub brak indeksu. Optymalizowanie mapowania, gdy problem leży w zapytaniu, to marnowanie czasu.

### Trzy argumenty, które słyszałem i które są NIEPRAWDĄ

> ⚠️ **Pułapka** — „ORM jest zawsze wolniejszy od ręcznego SQL-a”. Nieprawda w praktyce. Generowany SQL jest zwykle równie dobry jak napisany ręcznie, często lepszy (bo SQLAlchemy nie zapomni o parametrach). Rzeczywisty koszt ORM to narzut w Pythonie: tworzenie obiektów, śledzenie zmian, mapowanie. Ten narzut jest realny, ale w typowej aplikacji webowej ginie w porównaniu z opóźnieniem sieci do bazy. Zmierz, zanim uwierzysz w tę opinię.
>
> „ORM zwalnia z nauki SQL-a”. Nieprawda — patrz sekcja wyżej. Co więcej, SQLAlchemy jest zaprojektowana tak, że **uczy SQL-a**, pokazując ci każde wygenerowane zapytanie.
>
> „SQLAlchemy to biblioteka dla dużych systemów”. Nieprawda. Do skryptu na 300 linii też się nada — a jeśli ten skrypt będzie rósł, nie będziesz musiał go przepisywać.

### Tabela decyzyjna: ORM czy Core?

| Sytuacja | Wybór | Uzasadnienie |
|---|---|---|
| CRUD w aplikacji webowej | **ORM** | Czytelność, relacje, walidacja, transakcje |
| Przypadki użycia z logiką biznesową | **ORM** | Obiekty mogą mieć metody i niezmienniki |
| Raport z agregacjami | **Core** | Zwraca liczby, nie encje; czytelniejszy w jednym miejscu |
| Import > 10 000 wierszy | **Core** | Brak narzutu śledzenia obiektów |
| Złożone zapytanie analityczne z CTE i oknami | **Core** lub `text()` | ORM dokłada złożoność bez korzyści |
| Procedura składowana, `EXPLAIN`, DDL administracyjne | `text()` | Nie da się tego zamodelować obiektowo |
| Praca z bazą legacy, używasz 4 kolumn z 60 | **Core** z refleksją | Modelowanie całości to strata czasu |
| Aplikacja mieszana (np. API + cron z raportami) | **Oba** | Ta sama biblioteka, dwa style — to nie problem, to funkcja |

---

## Jak SQLAlchemy wypada na tle alternatyw

Nie chcesz wierzyć na słowo, że SQLAlchemy jest dobra. Słusznie. Oto uczciwe porównanie z realnymi opcjami, jakie masz w Pythonie.

### Surowy DBAPI (`sqlite3`, `psycopg`)

Zero zależności, pełna kontrola, minimalny narzut. Ale: piszesz SQL ręcznie (i ryzykujesz wstrzyknięciem), mapujesz wiersze ręcznie, sam zarządzasz transakcjami, kod jest nieprzenośny między bazami. Sensowne dla bardzo małych skryptów albo dla kodu, który faktycznie jest SQL-em (np. narzędzie administracyjne).

### Django ORM

Dojrzały, świetnie zintegrowany z Django, ma wbudowane migracje i całe narzędziownictwo. Ale jest **wbudowany w framework** — nie użyjesz go sensownie poza Django. Reprezentuje wzorzec **Active Record**: obiekt sam wie, jak się zapisać (`book.save()`). SQLAlchemy reprezentuje **Data Mapper**: obiekt nie wie o bazie, a mapowaniem zajmuje się osobna warstwa. To różnica architektoniczna, nie estetyczna — wracamy do niej w module 19.

### SQLModel

Biblioteka łącząca SQLAlchemy z Pydanticiem. Znakomita, gdy budujesz API na FastAPI i chcesz, żeby jeden model był jednocześnie modelem bazy i schematem walidacji wejścia/wyjścia. Zbudowana **na** SQLAlchemy — więc wszystko, co tu poznasz, jest jej podstawą. Minusy: mniejsza kontrola nad złożonymi scenariuszami, mniejsza społeczność, mniej elastyczne relacje. Dojrzały wybór dla prostych i średnich API.

### Peewee

Lekki ORM, prosty, przyjemny w użyciu. Dobry do małych i średnich projektów, mniej „enterprise”: mniej elastyczny w trudnych przypadkach, mniejsza kontrola nad SQL-em, tryb asynchroniczny w formie nakładki.

### Tortoise ORM

Natywnie asynchroniczny od podstaw, inspirowany Django ORM. Ciekawy dla projektów w pełni async. Mniejsza dojrzałość niż SQLAlchemy, mniejszy ekosystem, mniej narzędzi do migracji i debugowania.

### Tabela porównawcza

| Kryterium | SQLAlchemy 2.x | Django ORM | SQLModel | Peewee | Tortoise |
|---|---|---|---|---|---|
| Wzorzec | Data Mapper | Active Record | Data Mapper | Active Record | Active Record |
| Dojrzałość | ⭐⭐⭐⭐⭐ (od 2006) | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| Użycie poza frameworkiem | tak | praktycznie nie | tak | tak | tak |
| Asynchroniczność | natywna (`AsyncSession`) | częściowa (od 4.1) | natywna | przez nakładkę | natywna |
| Migracje | Alembic (osobne narzędzie, klasa sama w sobie) | wbudowane | Alembic | wbudowane (proste) | Aerich |
| Kontrola nad SQL-em | bardzo duża (`text()`, Core, ORM) | średnia | przez SQLAlchemy | mała/średnia | mała |
| Sposób pracy z wieloma bazami | natywny (bindy) | ograniczony | ograniczony | ograniczony | słaby |
| Ekosystem | ogromny | ogromny (Django) | rosnący | mały | mały |

**Jak to czytać:** SQLAlchemy wygrywa tam, gdzie potrzebujesz **kontroli, przenośności i życia poza frameworkiem**. Django ORM wygrywa, gdy cała aplikacja to Django i chcesz maksymalnej integracji. SQLModel wygrywa dla prostego API na FastAPI, gdy nie potrzebujesz niuansów. Peewee i Tortoise są sensowne dla mniejszych projektów o jasnych wymaganiach — ale mają mniejszy margines na rozbudowę.

Interesujący fakt praktyczny: **kilka z tych bibliotek korzysta z SQLAlchemy pod spodem** albo jest z nią zintegrowanych. To pokazuje, że SQLAlchemy nie konkuruje z całym światem — jest warstwą infrastrukturalną, na której inni budują.

---

## Ekosystem — co jeszcze przyda ci się w praktyce

SQLAlchemy nie jest samotną wyspą. W kursie użyjemy kilku bibliotek z jej otoczenia; pozostałe warto znać ze słyszenia.

| Biblioteka | Do czego | Kiedy sięgasz |
|---|---|---|
| **Alembic** | Migracje schematu bazy — wersjonowanie zmian struktury tak jak kodu | Od pierwszego dnia prawdziwego projektu (moduł 16) |
| **SQLModel** | Połączenie SQLAlchemy + Pydantic w jeden model | Proste API FastAPI |
| **sqlalchemy-utils** | Gotowe typy i helpery (typy domenowe, kolumny z wyborem, narzędzia do refleksji) | Gdy brakuje ci drobiazgów, ale nie chcesz ich pisać |
| **alembic-utils** | Dodatki do Alembica (widoki, funkcje, triggery w migracjach) | Gdy migracje obejmują obiekty inne niż tabele |
| **FastAPI** | Framework webowy; natywnie współpracuje z sesją jako zależnością | Budowa API (moduł 22) |
| **Pydantic v2** | Walidacja i serializacja danych (schematy wejścia/wyjścia) | Granica API — nie pozwól, żeby encje ORM wychodziły na zewnątrz (moduł 19) |
| **pytest** | Testy; integruje się z SQLAlchemy przez fixtures | Zawsze (moduł 18) |
| **factory_boy / polyfactory** | Fabryki danych testowych | Gdy ręczne tworzenie obiektów w testach staje się męczące |
| **testcontainers** | Prawdziwy PostgreSQL w Dockerze do testów | Gdy testy na SQLite przestają wystarczać |
| **GeoAlchemy2** | Typy przestrzenne (GIS) na bazie PostGIS | Aplikacje mapowe |
| **SQLAdmin** | Panel administracyjny generowany z modeli | Szybki back-office |
| **pandas** | Analiza danych; `read_sql` przyjmuje Connection SQLAlchemy | Analiza i eksport danych |

Najważniejsza z tej listy jest **Alembic**, i to nie przypadek. Zacznij używać go **od pierwszego dnia**, gdy tylko projekt przestanie być zabawką. Powód jest brutalny: jeśli raz zmienisz schemat bazy przez `create_all()` na bazie z danymi produkcyjnymi, dowiesz się, co znaczy „straciliśmy dane klientów”. Nie ma drugiej takiej lekcji w tym kursie, której lepiej byłoby uniknąć. Moduł 16 poświęcony jest temu w całości.

---

## Historia i wersjonowanie: dlaczego stare przykłady nie działają

To sekcja, którą czytelnicy najczęściej pomijają — i najczęściej tego żałują. Powód: **90% przykładów SQLAlchemy w internecie jest napisane w starym API** i nie zadziała z tym, czego nauczysz się w kursie.

### Skrócona historia

**SQLAlchemy 1.0 (2015) i wcześniejsze** — powstawało API „1.x”: `Session.query()` jako główny sposób budowania zapytań, `declarative_base()` jako fabryka klasy bazowej, `Column()` bez typowania. Ten styl dominuje w starszych tutorialach, na Stack Overflow i w książkach do dziś.

**SQLAlchemy 1.4 (2021) — pomost.** Wprowadzono „nowe” API 2.0-owe obok starego, żeby dało się migrować stopniowo. Pojawiło się ostrzeżenie `SQLALCHEMY_WARN_20`, które wskazywało miejsca do poprawy. Wtedy też async stał się realnie użyteczny.

**SQLAlchemy 2.0 (2023) — nowe API.** Duże uporządkowanie:

- `select()` jest **jedynym** sposobem budowania zapytań — identycznie w Core i ORM (`Session.query()` nadal działa jako relikt, ale dokumentacja nazywa go *legacy* i nie jest zalecany);
- `DeclarativeBase` zamiast `declarative_base()`;
- `Mapped[int]` i `mapped_column()` zamiast `Column(Integer)` — modele stają się czytelne dla `mypy` i edytora;
- `engine.execute()` **usunięte** — trzeba jawnie otworzyć połączenie (`with engine.begin() as conn:`);
- wymóg Pythona 3.7+ (wtedy); async jako pełnoprawny tryb.

**SQLAlchemy 2.1 (2026) — dopracowanie.** Nie jest to przełom, ale zmiany są odczuwalne:

> 🆕 **SQLAlchemy 2.1** — najważniejsze zmiany, o których warto wiedzieć:
> - **Wymaga Pythona 3.11+**. Jeśli utrzymujesz starszy projekt na 3.9/3.10, 2.1 nie wchodzi w grę.
> - **Domyślne sterowniki**: `postgresql://` → `psycopg` (wersja 3) zamiast `psycopg2`; `oracle://` → `oracledb`.
> - **`greenlet` nie jest instalowany automatycznie** — tryb async wymaga `pip install "sqlalchemy[asyncio]"`.
> - **Wsparcie t-stringów** z Pythona 3.14 (nowy mechanizm szablonów stringów) — pozwala wstawiać parametry w tekst zapytania w sposób formalnie bezpieczny.
> - **Nowe konstrukcje DDL**: `CreateView` i `CREATE TABLE AS SELECT` — tworzenie widoków i tabel pochodnych bez pisania `text()`.
> - **Lepsze typowanie** dla `Result` i `Row` zgodne z PEP 646 (variadic generics) — edytor dokładniej podpowiada typy przy rozpakowywaniu wierszy.
> - **Poprawione mapowanie dataclass**: klasy `MappedAsDataclass` nie dostają już domyślnych wartości wpisanych do `__dict__` w nieoczekiwany sposób.
> - **`selectinload`** zyskuje opcję `omit_join` dla relacji wiele-do-wielu oraz parametr `chunksize` do dzielenia zapytań `IN (...)` na porcje.
> - **`autoflush` w sesji działa bezwarunkowo** — usunięto przypadki, w których był pomijany.
>
> Materiał tego kursu jest pisany pod **2.0 jako stan domyślny**, a wszystko, co jest specyficzne dla 2.1, oznaczamy ramką taką jak ta. Dzięki temu kurs działa na obu wersjach.

### Czego dotyczy ten kurs

Kurs uczy **API 2.0 i późniejszego**. Konsekwencje praktyczne:

- **Nigdy** nie używamy `Session.query()`.
- **Nigdy** nie używamy `declarative_base()` jako zalecanej drogi.
- **Nigdy** nie używamy `engine.execute()`.
- **Zawsze** piszemy `select()`, `Mapped[]`, `mapped_column()`, `session.execute()` / `session.scalars()`.
- Jeśli musimy wspomnieć API 1.x (głównie w dodatku `A4_migracja_i_nowosci.md`), robimy to **jawnie jako o przestarzałym**.

### Jak rozpoznać stary przykład

Jeśli zobaczysz w kodzie którąkolwiek z tych rzeczy — to API 1.x i musisz go zaktualizować:

| Sygnał w kodzie | Co to znaczy |
|---|---|
| `session.query(Book).all()` | Styl 1.x; zamień na `session.scalars(select(Book)).all()` |
| `query.filter(...)` | Styl 1.x; zamień na `stmt.where(...)` |
| `declarative_base()` | Styl 1.x; zamień na klasę `DeclarativeBase` |
| `engine.execute(...)` | **Usunięte w 2.0** — nie zadziała wcale |
| `Column("title", String(50))` wewnątrz klasy modelu | Styl 1.x; zamień na `Mapped[str]` + `mapped_column(String(50))` |
| `backref="books"` | Styl 1.x; zamień na `back_populates` (obu stronach) |
| `lazy="dynamic"` | Styl 1.x; zamień na `lazy="write_only"` |
| `bulk_save_objects()` | Styl 1.x; zamień na `session.execute(insert(...), list_of_dicts)` |
| `from sqlalchemy.ext.declarative import declarative_base` | Moduł `ext.declarative` jest przestarzały |
| `Session(engine, future=True)` | `future` było potrzebne w 1.4; w 2.0 jest domyślne i zbędne |

> 💡 **Analogia** — Kopiowanie przykładu w API 1.x i zdziwienie, że „nie działa”, przypomina wzięcie przepisu z lat 80. i narzekanie, że piekarnik nie ma funkcji, której ten przepis wymaga. Przepis nie jest zły — jest nieaktualny. Dokumentacja SQLAlchemy ma osobny rozdział o migracji (`migration_20.html`), a dodatek A4 tego kursu to praktyczna tabela „przed → po” dla kilkudziesięciu wzorców.

> 🧠 **Dlaczego tak jest** — Dlaczego twórcy zdecydowali się na taką rewolucję? Bo chcieli **jednego stylu zapytań** dla Core i ORM, jawnego typowania (dzięki czemu edytor i `mypy` rozumieją twoje modele) oraz solidnego fundamentu pod async. Wersja 1.4 była pomostem, który pozwolił migrować stopniowo — i jest dobrym przykładem tego, jak duży projekt infrastrukturalny może zmienić API bez zabijania użytkowników.

---

## Mapa całego kursu

Ostatnia rzecz w tym module: gdzie czego szukać. Nie ucz się tego na pamięć — wróć tu, gdy poczujesz się zagubiony.

### Część I — orientacja i Core (moduły 01–06)

| # | Plik | Czego dotyczy |
|---|---|---|
| 01 | `01_wprowadzenie.md` | **Ten plik.** Po co SQLAlchemy, Core vs ORM, przenośność |
| 02 | `02_srodowisko_i_engine.md` | Instalacja, `Engine`, adresy połączeń, pula połączeń |
| 03 | `03_metadata_ddl.md` | `MetaData`, `Table`, `Column`, typy, ograniczenia, refleksja |
| 04 | `04_select_i_result.md` | `select()`, `Result`, filtrowanie, sortowanie, agregacje |
| 05 | `05_dml.md` | `insert`, `update`, `delete`, `RETURNING`, upsert, transakcje w Core |
| 06 | `06_joiny_i_zaawansowane_sql.md` | JOIN, podzapytania, CTE, funkcje okna |

### Część II — ORM (moduły 07–11)

| # | Plik | Czego dotyczy |
|---|---|---|
| 07 | `07_modele_deklaratywne.md` | `DeclarativeBase`, `Mapped`, `mapped_column`, mixiny |
| 08 | `08_sesja_cykl_zycia.md` | `Session`, identity map, Unit of Work, cykl życia obiektu |
| 09 | `09_relacje.md` | `relationship`, kardynalności, kaskady, strategie ładowania |
| 10 | `10_zapytania_orm.md` | `select()` w ORM, `session.scalars()`, ORM-owy DML |
| 11 | `11_ladowanie_i_n_plus_1.md` | `joinedload`, `selectinload`, diagnoza N+1 |

### Część III — warsztat inżyniera (moduły 12–18)

| # | Plik | Czego dotyczy |
|---|---|---|
| 12 | `12_typy_i_wlasne_typy.md` | `TypeDecorator`, JSON, `Uuid`, `Enum`, typy mutowalne |
| 13 | `13_zdarzenia_i_hybrydy.md` | Zdarzenia, `@validates`, `hybrid_property`, `association_proxy` |
| 14 | `14_transakcje_i_wspolbieznosc.md` | Izolacja, blokady, wyścigi, deadlocki, retry |
| 15 | `15_asynchronicznosc.md` | `AsyncEngine`, `AsyncSession`, greenlet, pułapki |
| 16 | `16_alembic_migracje.md` | Migracje schematu, `autogenerate`, wdrożenia |
| 17 | `17_wydajnosc.md` | Profilowanie, bulk, indeksy, strumieniowanie, cache |
| 18 | `18_testowanie.md` | pytest, izolacja testów, fabryki, testy async |

### Część IV — architektura i wzorce (moduły 19–24)

| # | Plik | Czego dotyczy |
|---|---|---|
| 19 | `19_warstwy_i_data_mapper.md` | Warstwy, modele domenowe vs ORM, DTO, Data Mapper |
| 20 | `20_repository.md` | Wzorzec Repository, paginacja, Specification |
| 21 | `21_unit_of_work.md` | Unit of Work, sesja per request, dependency injection |
| 22 | `22_fastapi_integracja.md` | Aplikacja REST od zera na SQLAlchemy + FastAPI |
| 23 | `23_projekt_koncowy.md` | Projekt końcowy — przewodnik, nie rozwiązanie |
| 24 | `24_antywzorce_faq.md` | Antywzorce, debugowanie, FAQ, checklisty |

### Dodatki (A1–A5)

- `A1_sciaga.md` — ściąga: najczęstsze wzorce w tabelach
- `A2_glosariusz.md` — glosariusz EN–PL (150+ haseł)
- `A3_cwiczenia_rozwiazania.md` — rozwiązania wszystkich zadań
- `A4_migracja_i_nowosci.md` — migracja z 1.x, nowości 2.1, roadmapa
- `A5_zasoby.md` — dokumentacja, książki, narzędzia, dalsza nauka

**Gdzie czego szukać w razie konkretnego problemu:**

- *„Nie wiem, jak to zapisać do bazy”* → 05, 08
- *„Zapytanie zwraca nie to, co trzeba”* → 04, 10, 24
- *„Aplikacja jest wolna”* → 11, 17, 24
- *„Nie wiem, jak zmienić schemat”* → 16
- *„Jak to zorganizować w projekcie”* → 19, 20, 21, 22
- *„Coś się dzieje samo i nie wiem dlaczego”* → 08, 13, 24

---

## Słownik startowy — 12 pojęć, które wrócą w każdym module

| # | Termin (EN) | Po polsku | Znaczenie w jednym zdaniu |
|---|---|---|---|
| 1 | **Engine** | silnik | Obiekt reprezentujący bazę danych w twojej aplikacji; tworzy się go raz i służy do tworzenia połączeń (moduł 02). |
| 2 | **Connection** | połączenie | Pojedyncze, otwarte połączenie z bazą; działa w kontekście transakcji (moduł 02). |
| 3 | **Dialect** | dialekt | Zbiór reguł opisujących, jak wyrażenie SQLAlchemy przełożyć na konkretny język bazy (SQLite, PostgreSQL…) (moduł 01). |
| 4 | **DBAPI** | sterownik bazy | Biblioteka Pythona (`sqlite3`, `psycopg`, `asyncpg`) mówiąca protokołem konkretnej bazy; standard PEP 249 (moduł 01). |
| 5 | **MetaData** | metadane | Katalog zbierający definicje tabel; „teczka” trzymająca cały schemat w kodzie (moduł 03). |
| 6 | **Schema** | schemat | Opis struktury bazy: jakie tabele, jakie kolumny, jakie ograniczenia (moduł 03). |
| 7 | **Core** | rdzeń | Warstwa SQLAlchemy operująca na tabelach i wyrażeniach SQL, bez mapowania obiektów (moduł 01). |
| 8 | **ORM** | mapowanie obiektowo-relacyjne | Warstwa SQLAlchemy mapująca klasy Pythona na tabele i obiekty na wiersze (moduł 01). |
| 9 | **Session** | sesja | Obiekt ORM, który śledzi obiekty, zarządza transakcją i decyduje, kiedy wysłać SQL (moduł 08). |
| 10 | **Transaction** | transakcja | Niepodzielna grupa operacji: albo wykonają się wszystkie, albo żadna (moduł 14). |
| 11 | **Migration** | migracja | Wersjonowana zmiana struktury bazy, zapisana jako plik i możliwa do cofnięcia (moduł 16). |
| 12 | **Repository** | repozytorium | Warstwa pośrednicząca między logiką aplikacji a zapytaniami do bazy (moduł 20). |

---

## Podsumowanie

Zabierz ze sobą te punkty:

1. **Trwałość danych wymaga bazy danych.** Plik tekstowy nie ma typów, ograniczeń, transakcji ani współbieżności. Każdy z tych braków kosztuje wcześniej czy później.
2. **Sklejanie SQL-a ze stringów prowadzi do wstrzyknięcia SQL.** Parametryzacja (przekazywanie danych osobno od zapytania) to nie dobra praktyka, ale warunek bezpieczeństwa. SQLAlchemy wymusza ją konstrukcyjnie.
3. **Ręczne mapowanie wierszy na obiekty jest uciążliwe i kruche.** ORM zastępuje `row[2]` przez `book.title` i prowadzi ewidencję konwersji typów za ciebie.
4. **SQL w każdej bazie jest inny.** Warstwa dialektów pozwala pisać jedno wyrażenie i kompilować je do SQL właściwego dla SQLite, PostgreSQL czy MySQL.
5. **SQLAlchemy to dwie warstwy w jednej bibliotece.** Core daje narzędzia i kontrolę; ORM dodaje obiekty, relacje i automatyczne zapisywanie. Można używać obu w jednym projekcie.
6. **ORM nie zwalnia z nauki SQL-a.** Kurs konsekwentnie pokazuje wygenerowany SQL — to najszybsza droga do prawdziwej biegłości.
7. **ORM nie zawsze jest właściwym wyborem.** Raporty analityczne, importy masowe, praca z bazami legacy i komendy administracyjne lepiej wychodzą w Core lub `text()`.
8. **Domyślnie: ORM; schodź do Core, gdy masz uzasadnienie.** Odwrotna strategia kosztuje albo czytelność, albo wydajność.
9. **Kurs uczy API 2.0+.** `select()`, `DeclarativeBase`, `Mapped`, `mapped_column()` — nie `Session.query()` ani `declarative_base()`.
10. **SQLAlchemy 2.1 to dopracowanie, nie rewolucja** — ale wymaga Pythona 3.11+ i warto jawnie wybierać sterowniki w adresie połączenia.

---

## Ćwiczenia

### wiczenie 1 — Co Core, a co ORM?

Dla każdego zadania wskaż, czy zrobiłbyś je w **ORM**, w **Core**, czy przez **`text()`**, i uzasadnij w jednym–dwóch zdaniach.

1. Zarejestrowanie nowego czytelnika w bibliotece wraz z jego danymi kontaktowymi.
2. Wygenerowanie raportu: „10 najczęściej wypożyczanych tytułów w podziale na miesiąc i kategorię, z porównaniem rok do roku”.
3. Import 500 000 wypożyczeń z historycznego pliku CSV.
4. Dodanie indeksu na kolumnę `reader_id` w tabeli `loans`.
5. Wyświetlenie strony z wypożyczeniami czytelnika — razem z tytułami książek i nazwiskami autorów.
6. Wycofanie (DELETE) wszystkich wypożyczeń starszych niż 5 lat — jednorazowa operacja czyszcząca.
7. Sprawdzenie rozmiaru tabeli w PostgreSQL (`pg_total_relation_size`).

### Ćwiczenie 2 — Znajdź błędy

Poniższy kod wygląda jak coś skopiowanego z bloga z 2018 roku. Wypisz **wszystkie** elementy API 1.x oraz problem bezpieczeństwa, a następnie napisz wersję w stylu 2.0.

```python
# examples/01_exercise_legacy.py
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import Session
from sqlalchemy import Column, Integer, String

Base = declarative_base()
engine = create_engine("sqlite:///library.db", future=True)
session = Session(engine)

class Book(Base):
    __tablename__ = "books"
    id = Column(Integer, primary_key=True)
    title = Column(String(200))
    author = Column(String(100))

result = engine.execute("SELECT * FROM books WHERE title LIKE '%" + "wied" + "%'")
for row in result:
    print(row[0], row[2])

books = session.query(Book).filter(Book.author == "Sapkowski").all()
for b in books:
    print(b.title)
```

### wiczenie 3 — Uzasadnij wybór

Twoja aplikacja obsługuje dziennie 5 000 wypożyczeń i 40 000 odczytów katalogu. Twój kolega proponuje przepisanie wszystkiego na ręczne `text()` zamiast ORM, argumentując: „ORM jest wolniejszy”. Napisz krótką odpowiedź (10–15 zdań) z **co najmniej dwoma konkretnymi argumentami**, w której:

- wskazujesz, gdzie w tej aplikacji faktycznie leży potencjalne wąskie gardło,
- proponujesz, jak **zweryfikować** tezę kolegi, zamiast dyskutować na opinie,
- wskazujesz, w których konkretnych miejscach warto byłoby zejść do Core.

### Rozwiązania

<details>
<summary><strong>Rozwiązanie 1 — Co Core, a co ORM?</strong></summary>

1. **ORM.** Rejestracja to operacja na jednej encji z relacjami (adres, dane kontaktowe) i z logiką biznesową (unikalność e-maila, walidacja wieku). Naturalne dla modelu obiektowego.
2. **Core.** To zapytanie zwraca liczby w wielu wymiarach, nie encje. ORM dodałby warstwę `func.*` i `case()` bez żadnej korzyści z mapowania obiektów. Prawdopodobnie dodatkowo warto rozważyć zmaterializowany widok albo tabelę agregatów (moduł 17).
3. **Core** (lub wręcz natywne narzędzie importu bazy). Śledzenie 500 000 obiektów w sesji to zbędny narzut pamięci i czasu. Moduł 05 pokazuje porównanie wydajności.
4. **`text()`** (albo Alembic). DDL nie jest operacją na danych. W praktyce nie robisz tego „w kodzie aplikacji”, tylko w migracji albo jednorazowym skrypcie administracyjnym.
5. **ORM.** Typowa ścieżka czytania z relacjami — dokładnie to, do czego ORM służy. Uwaga na N+1, którą rozwiązujesz przez `selectinload` (moduł 11).
6. **Core.** Masowa operacja `DELETE` na nieznanej z góry liczbie wierszy. Nie ma sensu ładować encji do pamięci po to, żeby je usunąć — wystarczy jedno `delete().where(...)` (moduł 05).
7. **`text()`.** Funkcja administracyjna specyficzna dla PostgreSQL. SQLAlchemy nie ma (i nie powinna mieć) abstrakcji na wszystko.

</details>

<details>
<summary><strong>Rozwiązanie 2 — Znajdź błędy</strong></summary>

**Znalezione problemy:**

| # | Problem | Dlaczego to źle |
|---|---|---|
| 1 | `from sqlalchemy.ext.declarative import declarative_base` | Moduł `ext.declarative` jest przestarzały — emituje ostrzeżenie, należy używać `DeclarativeBase` z `sqlalchemy.orm`. |
| 2 | `declarative_base()` | Styl 1.x. Zalecane: klasa bazowa dziedzicząca po `DeclarativeBase`. |
| 3 | `future=True` w `create_engine` | Było potrzebne w 1.4, żeby przełączyć się na nowe API. W 2.0 jest domyślne i zbędne. |
| 4 | `engine.execute(...)` | **Usunięte w SQLAlchemy 2.0.** Podniesie `AttributeError`. Trzeba użyć `with engine.begin() as conn:` albo `with engine.connect() as conn:`. |
| 5 | `"SELECT * FROM books WHERE title LIKE '%" + "wied" + "%'"` | **Wstrzyknięcie SQL.** Wartość jest sklejana z tekstem zapytania. To najpoważniejszy błąd w tym kodzie. |
| 6 | `print(row[0], row[2])` | Ręczne indeksy krotek — kruche i nieczytelne. W 2.0 zwracany `Row` pozwala pisać `row.title`. |
| 7 | `Column(Integer, primary_key=True)` w klasie modelu | Styl 1.x. Zalecane: `id: Mapped[int] = mapped_column(primary_key=True)`. |
| 8 | `session.query(Book).filter(...)` | Styl 1.x. W 2.0: `session.scalars(select(Book).where(...))`. |
| 9 | `session = Session(engine)` na poziomie modułu | Globalna sesja — poważny antywzorzec (moduł 21). Powinna być tworzona w bloku `with`. |
| 10 | Brak `nullable=False` przy `title` i `author` | Autorzy modelu prawdopodobnie zakładają, że tytuł zawsze istnieje — a baza tego nie wymusza. Niejawna nieścisłość. |
| 11 | Brak `__repr__` w modelu | Bez tego `print(book)` da `<Book object at 0x...>`. Drobiazg, ale bardzo utrudnia debugowanie. |
| 12 | Brak zamknięcia / kontekstu menedżera | Sesja i połączenie nigdy nie są zamykane ani zatwierdzane — potencjalny wyciek połączeń. |

**Wersja w stylu 2.0:**

```python
# examples/01_exercise_legacy_fixed.py
"""Ten sam kod w stylu SQLAlchemy 2.0."""

from sqlalchemy import create_engine, select, text
from sqlalchemy.orm import DeclarativeBase, Mapped, Session, mapped_column


class Base(DeclarativeBase):
    """Klasa bazowa dla wszystkich modeli – następca declarative_base()."""


class Book(Base):
    __tablename__ = "books"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column("title")
    author: Mapped[str] = mapped_column("author")

    def __repr__(self) -> str:
        return f"Book(id={self.id!r}, title={self.title!r}, author={self.author!r})"


engine = create_engine("sqlite:///library.db")
Base.metadata.create_all(engine)   # tylko dla przykładu – w projektach użyj Alembica

# 1) Zapytanie 'surowe' – teraz przez text() z parametrem wiązanym
with engine.connect() as conn:
    rows = conn.execute(
        text("SELECT id, title, author FROM books WHERE title LIKE :pattern"),
        {"pattern": "%wied%"},
    ).all()
    for row in rows:
        print(row.id, row.author)          # nazwane pola, nie row[0] / row[2]

# 2) To samo przez ORM
with Session(engine) as session:
    books = session.scalars(select(Book).where(Book.author == "Sapkowski")).all()
    for book in books:
        print(book)
```

Zwróć uwagę na dwie rzeczy w wersji poprawionej:

- **`text()` z `:pattern`** — to nie jest ochrona przez grzeczność. SQLAlchemy **odmówi** wykonania zapytania, jeśli parametr nie zostanie przekazany; nie da się „zapomnieć” o parametryzacji.
- **`SELECT id, title, author`** zamiast `SELECT *` — w kodzie produkcyjnym jawnie wymieniamy kolumny, żeby wynik nie zależał od kolejności ani od tego, czy ktoś doda kolumnę.

</details>

<details>
<summary><strong>Rozwiązanie 3 — Uzasadnij wybór</strong></summary>

Przykładowa odpowiedź:

> Rozumiem intencję, ale zanim przepiszemy 15 000 linii kodu, chciałbym sprawdzić, czy problem w ogóle dotyczy ORM-a. W tej aplikacji mamy 40 000 odczytów dziennie i 5 000 zapisów — czyli w przybliżeniu jedno zapytanie na sekundę w szczycie i pół zapisu na sekundę. To obciążenie, przy którym **narzut mapowania obiektów w Pythonie nie jest wąskim gardłem** — jest nim raczej liczba rund do bazy i sposób, w jaki budujemy listy. Gdyby faktycznym problemem było mapowanie, to przepisanie na `text()` pomogłoby o kilka procent; gdyby problemem jest N+1, to przepisanie na `text()` pomogłoby spektakularnie, ale **ten sam efekt dałoby dodanie `selectinload` w 20 miejscach** — a nie przepisanie wszystkiego.
>
> Proponuję weryfikację zamiast opinii. Pierwszy krok: włączyć logowanie SQL (`echo=True`) na środowisku testowym z realistyczną ilością danych i policzyć zapytania na najpopularniejszych ścieżkach: lista katalogu, szczegół książki, „moje wypożyczenia”. Drugi krok: uruchomić te ścieżki z profilowaniem (`py-spy`, `cProfile`) i zobaczyć, gdzie faktycznie spędzamy czas — w Pythonie, w bazie, czy w oczekiwaniu na odpowiedź. Trzeci krok: uruchomić `EXPLAIN (ANALYZE, BUFFERS)` dla trzech najwolniejszych zapytań i sprawdzić plany. To zajmie pół dnia i da odpowiedź, której żadna z nas nie ma teraz.
>
> Jeśli okaże się, że problemem jest wydajność zapytań, proponuję **nie** przepisywanie całości, a trzy punktowe zmiany: (1) dodać eager loading tam, gdzie wykryjemy N+1; (2) wybrać tylko potrzebne kolumny przez `load_only()` w endpointach listujących katalog — bo pobieranie całych encji z opisami i okładkami tylko po to, żeby pokazać tytuł, to marnotrawstwo; (3) napisać raporty w Core, bo one faktycznie zwracają agregaty, a nie encje. Zostawilibyśmy ORM tam, gdzie daje wartość: w logice wypożyczeń, regułach dostępności, walidacji i transakcjach. Taki zakres zmiany zajmie dwa dni zamiast trzech tygodni i jest mierzalny — będziemy wiedzieć, czy pomógł.

</details>

---

## Najczęstsze błędy i jak je czytać

Błędy w tej sekcji są typowe dla pierwszych dni pracy z SQLAlchemy i wynikają głównie z kopiowania starych przykładów. Ucz się czytać komunikaty, nie tylko je naprawiać.

| Komunikat / objaw | Przyczyna | Naprawa |
|---|---|---|
| `ModuleNotFoundError: No module named 'sqlalchemy'` | Biblioteka nie jest zainstalowana w aktywnym środowisku wirtualnym | `pip install "SQLAlchemy>=2.0"` w **aktywnym** venv; sprawdź `python -c "import sqlalchemy; print(sqlalchemy.__version__)"` |
| `AttributeError: 'Engine' object has no attribute 'execute'` | Kod w stylu 1.x — `engine.execute()` **zostało usunięte w 2.0** | `with engine.begin() as conn: conn.execute(stmt)` lub `with engine.connect() as conn:` |
| `SADeprecationWarning: The Engine.execute() method is considered legacy` | Jak wyżej, ale uruchamiane na 1.4 | Traktuj ostrzeżenie jako listę zadań do przepisania przed aktualizacją do 2.0 |
| `ArgumentError: Textual SQL expression 'SELECT ...' should be explicitly declared as text('SELECT ...')` | Przekazano string tam, gdzie SQLAlchemy wymaga wyrażenia lub `text()` | Owiń: `conn.execute(text("SELECT ..."))` — nigdy nie sklejaj z f-stringiem |
| `ObjectNotExecutableError: Not an executable object: [...]` | Do `execute()` trafiła lista albo obiekt, który nie jest wyrażeniem SQL | Sprawdź, co przekazujesz; dla listy parametrów użyj `conn.execute(stmt, list_of_dicts)` |
| `sqlite3.OperationalError: no such table: books` | Tabela nie istnieje — nie wywołano `create_all` albo wskazano inny plik bazy | Sprawdź ścieżkę w adresie (`sqlite:///library.db` a nie `sqlite://library.db`) i utwórz schemat |
| `sqlite3.OperationalError` / `psycopg.OperationalError: connection ...` | Błąd **sterownika**, nie SQLAlchemy — baza nieosiągalna, złe hasło, zła nazwa hosta | Zweryfikuj adres połączenia; pamiętaj, że komunikat pochodzi z DBAPI |
| `InvalidRequestError: Object '<Book at 0x...>' is already attached to session ...` | Ten sam obiekt dodany do dwóch sesji | Używaj jednej sesji na jednostkę pracy (moduł 08, 21) |
| `DetachedInstanceError: Instance <Book> is not bound to a Session` | Próba odczytu relacji obiektu poza sesją | Wczytaj relacje jawnie (`selectinload`) albo zamień encję na DTO przed wyjściem z sesji (moduł 11, 19) |
| `UnsupportedCompilationError` | Wykorzystujesz funkcję, której wybrany dialekt nie obsługuje | Sprawdź, czy dana konstrukcja istnieje w twojej bazie; użyj `with_variant` albo `text()` dla specyfiki dialektu |
| Kod działa, ale wygląda jak z bloga z 2018 | `session.query()`, `declarative_base()`, `backref`, `lazy="dynamic"` | Użyj tabeli „Jak rozpoznać stary przykład” z tego modułu; szczegóły w `A4_migracja_i_nowosci.md` |

> 🧠 **Dlaczego tak jest** — Dobra praktyka na cały kurs: gdy widzisz błąd, zadaj sobie pytanie *„która warstwa to mówi?”*. `sqlite3.OperationalError` mówi sterownik. `IntegrityError` mówi baza (przez sterownik). `ArgumentError` i `InvalidRequestError` mówi SQLAlchemy. `AttributeError` to zwykle twój kod. Ta jedna umiejętność skraca debugowanie o połowę.

---

## Słowniczek modułu

| Termin (EN) | Polski odpowiednik | Wyjaśnienie |
|---|---|---|
| **Bound parameter** | parametr wiązany | Miejsce w zapytaniu SQL (`?`, `%(name)s`) wypełniane wartością przekazaną osobno; podstawa bezpieczeństwa przed SQL injection |
| **Constraint** | ograniczenie | Reguła wymuszana przez bazę: `NOT NULL`, unikalność, klucz obcy, `CHECK` |
| **Core** | rdzeń | Warstwa SQLAlchemy operująca na tabelach i wyrażeniach SQL, bez mapowania klas na tabele |
| **Data Mapper** | mapownik danych | Wzorzec, w którym obiekty nie wiedzą o bazie, a tłumaczeniem zajmuje się osobna warstwa — wzorzec SQLAlchemy |
| **Active Record** | rekord aktywny | Wzorzec, w którym obiekt sam się zapisuje (`book.save()`) — wzorzec Django ORM, Peewee, Tortoise |
| **DBAPI** | sterownik bazy danych | Biblioteka Pythona implementująca protokół bazy (`sqlite3`, `psycopg`, `asyncpg`); standard PEP 249 |
| **DDL** | język definicji danych | Polecenia definiujące strukturę: `CREATE`, `ALTER`, `DROP` |
| **DML** | język manipulacji danymi | Polecenia operujące na danych: `SELECT`, `INSERT`, `UPDATE`, `DELETE` |
| **Dialect** | dialekt | Klasa opisująca, jak wyrażenia SQLAlchemy przełożyć na konkretną bazę i sterownik |
| **Driver** | sterownik | Synonim DBAPI w kontekście adresu połączenia (`postgresql+psycopg://`) |
| **Engine** | silnik | Główny obiekt reprezentujący bazę w aplikacji; fabryka połączeń, tworzony raz na proces |
| **ETL** | ekstrakcja, transformacja, ładowanie | Proces masowego przenoszenia danych między systemami; typowe miejsce, gdzie ORM przeszkadza |
| **Identity map** | mapa tożsamości | Rejestr w sesji gwarantujący, że jeden wiersz to w każdym momencie jeden obiekt Pythona |
| **Instrumented attribute** | atrybut instrumentowany | Atrybut klasy modelu, który SQLAlchemy „owinął” i o którym wie, że odpowiada kolumnie |
| **Legacy API** | przestarzałe API | Styl 1.x (`Session.query()`, `declarative_base()`); działa, ale dokumentacja go nie zaleca |
| **Mapping** | mapowanie | Przypisanie klasy Pythona do tabeli i jej atrybutów do kolumn |
| **Migration** | migracja | Wersjonowana, wykonywalna zmiana struktury bazy (narzędzie: Alembic) |
| **ORM** | mapowanie obiektowo-relacyjne | Warstwa mapująca obiekty Pythona na wiersze tabel i odwrotnie |
| **Persistence** | trwałość | Właściwość danych polegająca na przetrwaniu restartu programu i awarii |
| **Portability** | przenośność | Możliwość uruchomienia tego samego kodu na różnych bazach bez zmian |
| **Primary key** | klucz główny | Kolumna jednoznacznie identyfikująca wiersz tabeli |
| **Reflection** | refleksja | Odczytanie istniejącego schematu bazy do obiektów SQLAlchemy |
| **Row** | wiersz wyniku | Obiekt zwracany przez `Result`, mający nazwane pola (`row.title`) i zachowujący się jak krotka |
| **Schema** | schemat | Opis struktury bazy danych: tabele, kolumny, typy, ograniczenia |
| **Session** | sesja | Obiekt ORM śledzący obiekty, zarządzający transakcją i decydujący o wysyłce SQL |
| **SQL injection** | wstrzyknięcie SQL | Atak polegający na tym, że dane wejściowe zostają zinterpretowane jako kod SQL |
| **Transaction** | transakcja | Niepodzielna grupa operacji: wszystko albo nic |

---

## Dalsze czytanie

Oficjalna dokumentacja (bez zgadywania adresów — wszystkie poniższe istnieją):

- **Wprowadzenie i filozofia** — <https://docs.sqlalchemy.org/en/20/intro.html> — sekcja opisująca dokładnie to, o czym mówi ten moduł: po co istnieje SQLAlchemy i jak myśli o problemach.
- **Samouczek Core** — <https://docs.sqlalchemy.org/en/20/tutorial/index.html> — „Unified Tutorial”, najlepszy punkt startowy do praktyki.
- **ORM Quick Start** — <https://docs.sqlalchemy.org/en/20/orm/quickstart.html> — minimalny, kompletny przykład ORM w stylu 2.0; warto przeczytać po module 07.
- **„What's New in SQLAlchemy 2.0”** — <https://docs.sqlalchemy.org/en/20/changelog/whatsnew_20.html> — lista zmian, które ukształtowały API, którego się uczysz.
- **„Migrating to SQLAlchemy 2.0”** — <https://docs.sqlalchemy.org/en/20/changelog/migration_20.html> — jeśli masz stary kod; praktyczna tabela „przed → po”.
- **Dokumentacja dialektów** — <https://docs.sqlalchemy.org/en/20/dialects/index.html> — co SQLAlchemy wspiera dla SQLite, PostgreSQL, MySQL i innych.
- **Słownik pojęć** — <https://docs.sqlalchemy.org/en/20/glossary.html> — oficjalne definicje wszystkich terminów z tego modułu i kilkuset innych.
- **FAQ** — <https://docs.sqlalchemy.org/en/20/faq/index.html> — realne pytania z list mailingowych; sekcje o wydajności i sesji są szczególnie wartościowe.
- **Alembic** — <https://alembic.sqlalchemy.org/en/latest/> — dokumentacja narzędzia do migracji, o którym mówiliśmy w sekcji o ekosystemie.

Kursy i materiały, które warto znać ze słyszenia (bez reklam konkretnych pozycji): oficjalne tutoriale „SQLAlchemy Unified Tutorial” w formie notatników, dokumentacja PostgreSQL (sekcja „The SQL Language”) dla osób, które chcą zrozumieć SQL od strony bazy, oraz blog „Cosmic Python” dla wzorców architektonicznych, do których wrócimy w modułach 19–21.

---

## Co dalej

Wiesz już, po co istnieje SQLAlchemy, jak różni się Core od ORM, skąd bierze się przenośność między bazami i dlaczego stare przykłady z internetu nie działają. Ale jeszcze nie napisałeś ani jednej linii kodu z tą biblioteką.

W module [`02_srodowisko_i_engine.md`](02_srodowisko_i_engine.md) zbudujesz środowisko pracy i utworzysz pierwszy obiekt `Engine` — serce każdej aplikacji korzystającej z SQLAlchemy. Dowiesz się, jak skonstruowany jest adres połączenia, dlaczego tworzenie połączenia z bazą jest „drogie” i po co istnieje pula połączeń, oraz wykonasz pierwsze prawdziwe zapytanie, widząc jednocześnie jego SQL. To będzie moment, w którym cała teoria z tego modułu zamieni się w działający kod.

<!-- koniec modułu 01 -->