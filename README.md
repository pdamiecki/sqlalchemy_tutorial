# Kurs SQLAlchemy 2.x — od zera do architektury

> **Plik:** `README.md` (K-00) · **Poziom:** wprowadzający do całego kursu · **Czas:** ~10 min
> Wymagania wstępne: podstawy Pythona (zmienne, funkcje, klasy, `pip`). Zero wiedzy o bazach danych.

---

## Dlaczego ten kurs

**SQLAlchemy to biblioteka Pythona, która tłumaczy obiekty i kod Pythona na tabele i zapytania SQL** — dzięki niej nie musisz ręcznie sklejać tekstu zapytań ani ręcznie przepisywać wierszy z bazy na obiekty w programie. Zamiast pisać `"SELECT * FROM books WHERE id = " + str(book_id)` (co kończy się błędami i lukami bezpieczeństwa), piszesz `select(Book).where(Book.id == book_id)` — a biblioteka zamienia to na poprawny, bezpieczny SQL dopasowany do wybranej bazy danych.

**Dla kogo jest ten kurs.** Dla osoby, która umie już coś napisać w Pythonie, ale nie miała do czynienia z bazami danych albo zna je tylko ze słyszenia. Zakładamy, że nie wiesz, czym są SQL, transakcja, indeks, klucz obcy ani „wzorzec projektowy”. Wszystkie te pojęcia wprowadzamy od zera, po polsku, z analogiami i z pełnymi, uruchamialnymi przykładami. Kurs prowadzi od pierwszego `SELECT` aż do architektury produkcyjnej aplikacji webowej — bez przeskoków i bez „magii”.

**Czego ten kurs NIE jest.** SQLAlchemy to nie framework webowy (nie tworzy stron ani API — to robi np. FastAPI, o którym mówimy dopiero w MOD-21) i nie baza danych (nie przechowuje danych — to robi SQLite lub PostgreSQL). SQLAlchemy to warstwa pośrednia. Nie jest też kursem projektowania baz danych ani administracji serwerem bazy — te tematy pojawiają się tylko w zakresie, w jakim programista Pythona musi je znać.

---

## Mapa kursu

Kurs ma pięć części i 22 moduły (MOD-00 … MOD-21) plus materiały dodatkowe. Kolumna „Po module umiesz…” opisuje **umiejętność**, nie temat — to najszybszy sposób, żeby sprawdzić, czy jesteś w dobrym miejscu.

| Część | Moduły | Po module umiesz… |
|---|---|---|
| **I. Fundamenty i Core** (praca bez ORM) | MOD-00 | wyjaśnić, czym jest SQLAlchemy i kiedy użyć warstwy Core, a kiedy ORM |
| | MOD-01 | utworzyć silnik połączenia i bezpiecznie połączyć się z bazą (SQLite, PostgreSQL) |
| | MOD-02 | zdefiniować schemat tabel w Pythonie i utworzyć go w bazie |
| | MOD-03 | budować zapytania `SELECT` i odczytywać wyniki, rozumiejąc obiekt `Result` |
| | MOD-04 | zapisywać, aktualizować i usuwać dane w transakcji — bez ORM |
| | MOD-05 | łączyć tabele i pisać zapytania analityczne (JOIN, CTE, funkcje okna) |
| **II. ORM** (obiekty zamiast wierszy) | MOD-06 | zamienić tabele w klasy Pythona i pracować na obiektach |
| | MOD-07 | wyjaśnić cykl życia obiektu, rolę sesji i moment rzeczywistego zapisu |
| | MOD-08 | zamodelować relacje 1:1, 1:N i N:M oraz sterować kaskadami |
| | MOD-09 | pisać zapytania ORM w stylu 2.0 (bez przestarzałego `session.query()`) |
| | MOD-10 | wykryć i usunąć problem N+1 oraz świadomie wybierać strategię ładowania |
| | MOD-11 | dobrać właściwe typy kolumn i napisać własny typ (`TypeDecorator`) |
| | MOD-12 | rozszerzyć modele o walidację, zdarzenia i właściwości hybrydowe |
| **III. Produkcja** (niezawodność i szybkość) | MOD-13 | świadomie projektować transakcje i obsługę współbieżności |
| | MOD-14 | zbudować aplikację asynchroniczną na SQLAlchemy |
| | MOD-15 | wersjonować schemat bazy migracjami Alembic |
| | MOD-16 | mierzyć i realnie poprawiać wydajność (najpierw pomiar, potem zmiana) |
| | MOD-17 | pisać szybkie, izolowane testy warstwy danych |
| **IV. Architektura i wzorce** | MOD-18 | oddzielić model domenowy od modelu bazy (Data Mapper, warstwy) |
| | MOD-19 | zaimplementować repozytoria i specyfikacje zapytań |
| | MOD-20 | wyznaczyć granice transakcji i zarządzać sesją w architekturze |
| | MOD-21 | poskładać wszystko w produkcyjną aplikację (API REST + FastAPI) |
| **V. Materiały dodatkowe** | A1–A5 | korzystać ze ściągi, słowniczka, rozwiązań, przewodnika migracji i listy zasobów |

> 💡 **Analogia ogólna** — jeśli gubisz się w mapie, wyobraź sobie budowę domu: część I to nauka posługiwania się narzędziami i czytania projektu, część II to praca „na gotowych meblach”, część III to instalacje i odbiór techniczny, a część IV to zatrudnienie architekta i generalnego wykonawcy. Każda następna część zakłada poprzednią.

---

## Trzy ścieżki nauki

Kurs czytany po kolei zajmuje kilkadziesiąt godzin. Jeśli nie potrzebujesz wszystkiego, wybierz jedną z poniższych ścieżek. Odhaczaj moduły po kolei — kolejność ma znaczenie.

### (a) Laik / analityk danych — czytanie, pisanie i raportowanie danych

| ✔ | Moduł | Co daje |
|---|---|---|
| ☐ | MOD-00 | orientacja: po co to wszystko |
| ☐ | MOD-05 | zapytania analityczne i raporty (JOIN, agregacje) |
| ☐ | MOD-06 | praca na obiektach zamiast na wierszach |
| ☐ | MOD-07 *(przejrzyj)* | sesja — bez tego ORM nie działa, wystarczy przeczytać o cyklu życia |
| ☐ | MOD-08 | relacje — jak dane łączą się w całość |
| ☐ | MOD-13 | transakcje i integralność, żeby nie uszkodzić danych przy zapisach |

> Ta ścieżka pomija migracje, async, wydajność i architekturę. Sięgnij po nie dopiero, gdy zaczniesz budować aplikację, a nie tylko analizować dane.

### (b) Programista backendu — cały kurs po kolei

| ✔ | Zakres | Uwaga |
|---|---|---|
| ☐ | MOD-00 … MOD-21 | Czytaj bez pomijania. Moduły III i IV to część, po którą wraca się najczęściej |
| ☐ | A1, A2, A3 | Ściąga, słowniczek i rozwiązania ćwiczeń — używaj równolegle, nie na końcu |
| ☐ | A4 | Przeczytaj, jeśli spotkasz w pracy kod w starym stylu 1.x |

### (c) Architekt / doświadczony programista — przegląd i decyzje projektowe

Zaznaczono, co wystarczy **przejrzeć** (bez przepisywania ćwiczeń), a co warto przeczytać uważnie.

| ✔ | Moduł | Tryb |
|---|---|---|
| ☐ | MOD-02 | przeczytaj — jak definiuje się schemat w Pythonie |
| ☐ | MOD-06 | przeczytaj — model deklaratywny |
| ☐ | MOD-07 | przeczytaj — sesja i cykl życia obiektu |
| ☐ | MOD-09 | przejrzyj — styl zapytań 2.0 |
| ☐ | MOD-16 | przeczytaj — wydajność i metodologia pomiaru |
| ☐ | MOD-17 | przejrzyj — strategia testowania |
| ☐ | MOD-21 | przeczytaj — integracja z aplikacją produkcyjną |

> **Uzupełniająco** (poza listą podstawową): **MOD-15** (Alembic — migracje są niezbędne w każdym dojrzałym projekcie) oraz **MOD-20** (Unit of Work — granice transakcji). Bez tych dwóch modułów obraz architektury będzie niepełny.

---

## Wymagania i instalacja

**Potrzebujesz:** Python 3.11 lub nowszy, menedżer pakietów `pip` oraz — opcjonalnie — Docker (przyda się dopiero od części III, do uruchomienia PostgreSQL; wszystkie wcześniejsze moduły działają na SQLite bez żadnej konfiguracji).

Instalacja w pięciu linijkach:

```bash
python -m venv .venv
# Windows (PowerShell):  .venv\Scripts\Activate.ps1
# macOS / Linux:         source .venv/bin/activate
pip install "sqlalchemy>=2.0.30" alembic pytest
python -c "import sqlalchemy; print(sqlalchemy.__version__)"
```

> ⚠️ **Pułapka** — SQLAlchemy 2.1 (seria wydana w 2026) zmienia domyślne sterowniki baz i nie instaluje automatycznie `greenlet`. W kursie stawiamy na **2.0.x**; wszystkie różnice 2.1 są zawsze oznaczone ramką 🆕, więc starsza linia nie wywoła u Ciebie niespodzianek.

---

## Konwencje kursu

### Ramki, które napotkasz w każdym module

| Ramka | Znaczenie |
|---|---|
| 💡 **Analogia** | Życiowe porównanie, które buduje model mentalny, zanim pojawi się precyzja techniczna |
| 🧠 **Dlaczego tak jest** | Wyjaśnienie mechanizmu — sięgamy „pod podszewkę”, żeby nic nie działo się „samo” |
| ⚠️ **Pułapka** | Typowy błąd, jego objaw i sposób uniknięcia |
| 🔬 **Pod maską** | Rzeczywisty SQL, jaki zostanie wysłany do bazy dla pokazanego kodu |
| 🆕 **SQLAlchemy 2.1** | Nowość lub zmiana względem wersji 2.0 (w 2.0 nie działa lub działa inaczej) |
| 🧪 **Ćwiczenie** | Krótkie zadanie do wykonania od razu, w miejscu czytania |

### Kod i przykłady

- **Wszystkie przykłady kodu są uruchamialne** i znajdują się w katalogu `examples/`. Każdy blok kodu zaczyna się komentarzem ze ścieżką pliku, np. `# examples/06_models.py`.
- Przykłady w modułach wstępnych działają na **SQLite** (`sqlite:///:memory:` lub plik) — nie wymagają instalowania serwera bazy. Zachowania specyficzne dla PostgreSQL (lub innego dialektu) są zawsze wyraźnie oznaczone.
- Kod używa **stylu 2.0**: `select()` + `session.execute()` / `session.scalars()`, modele przez `Mapped[...]` i `mapped_column()`. Stare `session.query()` pojawia się wyłącznie jako wzmianka o przestarzałym API 1.x.
- Identyfikatory w kodzie są angielskie (`Book`, `Author`, `Loan`), komentarze i wyjaśnienia — polskie. Domena przykładów jest spójna (biblioteka/wypożyczalnia), żeby modele były znajome i nie trzeba było za każdym razem zgadywać, co reprezentują.

---

## Jak korzystać z kursu

1. **Czytaj z otwartym edytorem.** Każdy blok kodu wpisz (a raczej **przepisz**) i uruchom samodzielnie.
2. **Nie kopiuj kodu przez Ctrl+C / Ctrl+V.** Ręczne przepisywanie zmusza mózg do zauważenia importów, typów i pułapek, których kopiowanie nie ujawnia.
3. **Po każdym module zrób ćwiczenia** (sekcja „Ćwiczenia”) i porównaj z „Rozwiązaniami” dopiero po własnej próbie.
4. **Wracaj do „Pod maską”.** Jeśli nie rozumiesz SQL-a, który wygenerował ORM, wróć do odpowiedniego modułu z części I — to fundament.
5. **Nie pomijaj sekcji „Najczęstsze błędy”.** To najtańszy sposób, żeby nie tracić godzin na błędy, które ktoś już opisał.

---

## Jak zgłaszać błędy w materiale

Materiał jest dokumentem technicznym i też może zawierać błędy — literówki, nieaktualne API, przykłady, które nie działają na Twojej wersji biblioteki. Zgłaszając problem, dołącz:

1. **Numer modułu i nazwę sekcji** (np. „MOD-09, akapit o `selectinload`”),
2. **wersję SQLAlchemy i Pythona** (wynik `print(sqlalchemy.__version__)`),
3. **treść błędu** oraz **minimalny fragment kodu**, który go wywołuje.

Zgłoszenie w tej formie da się zweryfikować w kilka minut. Zgłoszenie „nie działa” — niestety nie.

---

**Kurs opiera się na oficjalnej dokumentacji SQLAlchemy:**

- SQLAlchemy 2.0 → <https://docs.sqlalchemy.org/en/20/>
- SQLAlchemy 2.1 → <https://docs.sqlalchemy.org/en/21/>
- Alembic → <https://alembic.sqlalchemy.org/en/latest/>

Zaczynaj od **[MOD-00 — Czym jest SQLAlchemy](00_wprowadzenie.md)**. Powodzenia!

<!-- koniec README -->
