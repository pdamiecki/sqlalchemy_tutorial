# Moduł 23 — Projekt końcowy: wypożyczalnia sprzętu

To moduł podsumowujący cały kurs. Nie znajdziesz tu gotowego rozwiązania — znajdziesz **pełną, zamrożoną specyfikację** fikcyjnej firmy „VerteRent”, **uzasadniony model danych**, **dziesięć etapów pracy** z kryteriami ukończenia, **tabelę punktową oceny** i **listę pytań kontrolnych**. Twoim zadaniem jest zbudować aplikację samodzielnie, korzystając z modułów 01–22 jako biblioteki wiedzy. Po ukończeniu tego modułu będziesz mieć w portfolio kompletny projekt backendowy: asynchroniczne API na FastAPI, warstwę danych na SQLAlchemy 2.0, migracje Alembic, testy i przemyślaną architekturę.

> **Poziom:** 🔴 architektoniczny
>
> **Czas:** ~600 minut (10 godzin zegarowych) rozłożone na 10 etapów — realistycznie 3–6 weekendów pracy własnej
>
> **Wymagania wstępne:** całe moduły 01–22. Kluczowe: `14_transakcje_i_wspolbieznosc.md` (blokady), `16_alembic_migracje.md`, `18_testowanie.md`, `20_repository.md`, `21_unit_of_work.md`, `22_fastapi_integracja.md`. Pomocnicze: `03_metadata_ddl.md`, `07_modele_deklaratywne.md`, `09_relacje.md`, `11_ladowanie_i_n_plus_1.md`, `12_typy_i_wlasne_typy.md`
>
> **Czego dotyczy plik:** przewodnik po projekcie zaliczeniowym. Zawiera specyfikację, model danych, plan pracy i kryteria oceny — **bez kompletnego kodu aplikacji**.

---

## Spis treści

- [1. Specyfikacja biznesowa](#1-specyfikacja-biznesowa)
- [2. Model danych](#2-model-danych)
- [3. Wymagania techniczne i definicja „gotowe”](#3-wymagania-techniczne-i-definicja-gotowe)
- [4. Etapy pracy](#4-etapy-pracy)
- [5. Kryteria oceny](#5-kryteria-oceny)
- [6. Rozszerzenia dla chętnych](#6-rozszerzenia-dla-chętnych)
- [7. Wskazówki do przeglądu własnej pracy](#7-wskazówki-do-przeglądu-własnej-pracy)
- [8. Gdzie szukać pomocy](#8-gdzie-szukać-pomocy)
- [Podsumowanie](#podsumowanie)
- [Ćwiczenia](#ćwiczenia)
- [Najczęstsze błędy i jak je czytać](#najczęstsze-błędy-i-jak-je-czytać)
- [Słowniczek modułu](#słowniczek-modułu)
- [Dalsze czytanie](#dalsze-czytanie)
- [Co dalej](#co-dalej)

---

## 1. Specyfikacja biznesowa

### 1.1. Firma i problem

Wyobraź sobie wypożyczalnię sprzętu w górach. Zimą ludzie wypożyczają narty, buty, kaski i kijki; latem — rowery, kaski i sakwy. W szczycie sezonu przy ladzie stoi kolejka dziesięciu osób, a za ladą pracuje dwóch pracowników z tabletem. Każdy pracownik musi w kilka sekund odpowiedzieć na trzy pytania:

1. Czy mamy wolny egzemplarz tego, o co prosi klient?
2. Ile to będzie kosztować na trzy dni?
3. Czy ten konkretny egzemplarz jest sprawny i kiedy wrócił z ostatniego wypożyczenia?

Dziś odpowiedzi na te pytania udziela zeszyt i pamięć pracownika. Efekt: podwójne wypożyczenie tego samego roweru, zgubione rezerwacje, kłótnie o kary za przetrzymanie, brak jakiejkolwiek historii, gdy sprzęt się zepsuje. Twoim zadaniem jest zastąpić zeszyt systemem.

> 💡 **Analogia — dlaczego to nie jest „zwykła aplikacja CRUD”.**
>
> Zwykłe CRUD (create/read/update/delete) przypomina segregator z kartoteką: dodaj kartę, przeczytaj kartę, zmień kartę. Wypożyczalnia sprzętu przypomina bardziej **parking z jednym miejscem**: jeśli dwóch kierowców jednocześnie wjedzie na to samo miejsce, dochodzi do stłuczki. Cała trudność tego projektu nie leży w dodawaniu rekordów — leży w **niezmiennikach**, czyli zdaniach, które muszą być prawdziwe zawsze, niezależnie od tego, ilu klientów klika jednocześnie.

### 1.2. Aktorzy systemu

Projekt ma dokładnie dwóch aktorów. Świadomie nie dodajemy trzeciego — to decyzja zakresowa, nie zapomnienie.

| Aktor | Kto to jest | Co robi w systemie |
|---|---|---|
| **Customer** (klient) | osoba wypożyczająca sprzęt | przegląda katalog, tworzy rezerwację, widzi swoje wypożyczenia i naliczone kary |
| **Staff** (pracownik) | osoba za ladą | potwierdza wypożyczenie, rejestruje zwrot, wycofuje uszkodzony sprzęt, zarządza cennikiem |

> 🧠 **Dlaczego tak jest** — w prawdziwych systemach role są zwykle trzy lub cztery (klient, pracownik, kierownik, administrator). Ograniczamy się do dwóch, ponieważ każda dodatkowa rola to kolejna warstwa autoryzacji, której ten projekt nie ma uczyć. Uproszczenie zakresu to nie lenistwo — to inżynieria. W modułach 19–21 wielokrotnie podkreślaliśmy: lepiej zrobić mniej rzeczy dobrze niż więcej byle jak.

### 1.3. Wymagania funkcjonalne

Wymagania są **zamrożone**. Nie zmieniamy ich w trakcie. Jeśli w połowie projektu przyjdzie Ci do głowy „a może jeszcze obsługa punktów lojalnościowych?” — zapisujesz to do pliku `IDEAS.md`, ale nie implementujesz. Ta dyscyplina jest częścią zadania.

**F1 — Katalog.** Klient widzi listę kategorii (narty, buty, rowery, kaski) i w każdej kategorii listę egzemplarzy. Dla każdego egzemplarza widzi, czy jest dostępny w wybranym przedziale dat.

**F2 — Rezerwacja.** Klient podaje e-mail, identyfikator egzemplarza oraz datę i godzinę rozpoczęcia i zakończenia. System tworzy rezerwację w stanie `PENDING`. Rezerwacja `PENDING` blokuje egzemplarz na 15 minut. Klient może ją potwierdzić (przejście do `CONFIRMED`) lub anulować.

**F3 — Wypożyczenie.** Pracownik rejestruje wypożyczenie, podając identyfikator klienta (albo rezerwacji) i kod egzemplarzy. Jedno wypożyczenie może obejmować wiele egzemplarzy — klient wypożycza komplet narty + buty + kaski.

**F4 — Zwrot.** Pracownik rejestruje zwrot każdego egzemplarza osobno. System liczy karę za przetrzymanie i zapisuje ją jako rekord kary.

**F5 — Historia.** Klient widzi wszystkie swoje wypożyczenia i kary, także historyczne. Pracownik widzi pełną historię zdarzeń konkretnego egzemplarza — kto, kiedy i co z nim zrobił.

**F6 — Wycofanie sprzętu.** Pracownik może wycofać egzemplarz z eksploatacji (zepsuty, skradziony, sprzedany). Wycofany egzemplarz nie pojawia się w katalogu, ale jego historia zostaje.

**F7 — Cennik.** Cena za dobę zależy od kategorii i okresu obowiązywania. Zmiana cennika nie wpływa na już zarejestrowane wypożyczenia.

### 1.4. Niezmienniki biznesowe

To najważniejsza część specyfikacji. **Niezmiennik** to zdanie, które musi być prawdziwe w każdej chwili istnienia systemu, bez wyjątków, nawet gdy dwie osoby klikają „Zarezerwuj” w tej samej milisekundzie. Jeśli niezmiennik jest kiedykolwiek złamany, system jest zepsuty — nawet jeśli żaden użytkownik tego nie zauważył.

| # | Niezmiennik | Jak go łamiemy przez przypadek |
|---|---|---|
| **I1** | Egzemplarz nie może być jednocześnie w dwóch aktywnych wypożyczeniach. | brak blokady przy rezerwacji → klasyczny wyścig |
| **I2** | Egzemplarz nie może mieć jednocześnie aktywnej rezerwacji i aktywnego wypożyczenia. | dwa niezależne warunki sprawdzane osobno |
| **I3** | Rezerwacja niepotwierdzona w ciągu 15 minut przechodzi w stan `EXPIRED`. | brak mechanizmu wygaszania — „wygasa” tylko w teorii |
| **I4** | Kara za przetrzymanie naliczana jest za każdy **rozpoczęty** dzień przetrzymania. | liczenie pełnych dób zamiast rozpoczętych |
| **I5** | Historia zdarzeń egzemplarza jest **tylko dopisywana** (append-only) i nie zależy od jego bieżącego stanu. | aktualizacja rekordu historii zamiast dopisania nowego |
| **I6** | Klient nie może mieć dwóch aktywnych rezerwacji tego samego egzemplarza. | brak częściowego unikalnego indeksu |
| **I7** | Zwróconego pozycji wypożyczenia nie można zwrócić powtórnie. | ponowne wysłanie tego samego żądania HTTP (retry) |
| **I8** | Cena zastosowana w wypożyczeniu jest **kopiowana** z cennika w momencie wypożyczenia i już się nie zmienia. | odwołanie do bieżącego cennika przy każdym wyliczeniu |

> 🧠 **Dlaczego niezmienniki są ważniejsze niż funkcje.**
>
> Funkcje można dodać później. Złamany niezmiennik oznacza, że **dane w bazie są już błędne** i trzeba je ręcznie naprawiać. Wypożyczalnia, która wydała jeden rower dwóm osobom, nie naprawi tego commitem — musi przeprosić dwóch klientów. Dlatego w tym projekcie testy niezmienników są ważniejsze niż testy endpointów.

> 🧪 **Ćwiczenie —** Zanim przejdziesz dalej, zastanów się przez minutę: który z ośmiu niezmienników najtrudniej zagwarantować i dlaczego? Odpowiedź znajdziesz na końcu tej sekcji w komentarzu, ale najpierw spróbuj sam.
>
> *Odpowiedź:* I1 i I2. Nie dlatego, że są skomplikowane logicznie — są banalne: „sprawdź, czy nie ma aktywnego wypożyczenia, jeśli nie ma, dodaj nowe”. Trudność polega na tym, że **sprawdzenie i dodanie to dwie osobne operacje**. Między nimi ktoś inny może zrobić to samo. To jest esencja problemu współbieżności i wrócimy do niego w Etapie 5.

### 1.5. Wymagania niefunkcjonalne

| # | Wymaganie | Jak je zmierzymy |
|---|---|---|
| **N1** | Przy 100 równoległych próbach rezerwacji tego samego egzemplarza dokładnie jedna kończy się sukcesem. | skrypt współbieżnościowy z Etapu 5 |
| **N2** | p95 czasu odpowiedzi endpointu listy egzemplarzy < 300 ms przy 10 000 egzemplarzy. | `hey` lub prosty skrypt `asyncio` |
| **N3** | Brak zapytań N+1 w ścieżkach krytycznych (lista wypożyczeń z pozycjami). | licznik zapytań z modułu 11 |
| **N4** | Schemat bazy powstaje wyłącznie przez migracje; `create_all()` nie jest wywoływane w kodzie produkcyjnym. | przegląd kodu — brak `create_all` w `app/` |
| **N5** | Każda migracja ma działający `downgrade`. | `alembic upgrade head && alembic downgrade base && alembic upgrade head` |
| **N6** | Brak SQL budowanego przez konkatenację stringów. | `grep -rn "f\".*SELECT" app/` — brak wyników |
| **N7** | Pokrycie testami logiki biznesowej ≥ 80 %. | `pytest --cov=app/services` |

> ⚠️ **Pułapka — „zmierzymy później”.**
>
> Wymagania niefunkcjonalne, które nie mają zdefiniowanej metody pomiaru, nie istnieją. „System ma być szybki” to nie wymaganie. „p95 < 300 ms przy 10 000 egzemplarzy mierzone skryptem `scripts/bench_listing.py`” to wymaganie. Zauważ, że w tabeli każdy wiersz ma kolumnę „jak je zmierzymy” — to nie ozdoba, to połowa treści.

### 1.6. Poza zakresem

- Płatności online (kara jest rejestrowana, nie pobierana).
- Powiadomienia e-mail i SMS.
- Logowanie i uwierzytelnianie (na potrzeby projektu klient i pracownik są identyfikowani parametrem, nie tokenem).
- Panel administracyjny z interfejsem graficznym.
- Rezerwacje wielopozycyjne (rezerwacja dotyczy dokładnie jednego egzemplarza).
- Obsługa stref czasowych innych niż jedna, ustalona w konfiguracji.

> 🧠 **Dlaczego tak jest** — wypisanie rzeczy „poza zakresem” jest równie ważne jak wypisanie rzeczy „w zakresie”, a w praktyce inżynierskiej robi się to znacznie rzadziej. Bez takiej listy każda rozmowa o projekcie kończy się zdaniem „a to przecież trzeba dodać”. Zamrożona lista „nie robimy tego” jest tarczą chroniącą termin realizacji.

---

## 2. Model danych

### 2.1. Diagram relacji

Zanim powstanie jedna linia kodu, musimy wiedzieć, jak wygląda świat, którego ten kod dotyczy. Poniżej diagram w ASCII. Strzałka oznacza stronę „wiele”; liczba przy strzałce to krotność.

```text
                       ┌──────────────┐
                       │  Category    │  (tabela słownikowa)
                       │  id  Integer │
                       │  name        │
                       │  slug        │
                       └──────┬───────┘
                              │ 1
                    ┌─────────┴─────────┐
                    │ N                 │ N
             ┌──────▼───────┐    ┌──────▼───────┐
             │ EquipmentItem│    │   Tariff     │  (cennik historyczny)
             │ id  Integer  │    │ id  Integer  │
             │ code  UNIQUE │    │ valid_from   │
             │ ...          │    │ valid_to     │
             └──┬────┬───┬──┘    │ price_cents  │
                │    │   │       └──────────────┘
                │    │   │
      N ┌───────┘    │   └───────┐ N
        │            │ N         │
┌───────▼──────┐  ┌──▼────────┐  │
│  LoanItem    │  │ ItemStatus│  │
│ (pośrednia   │  │  Event    │  │
│  z kolumnami)│  │ (append   │  │
│  due_at      │  │  only)    │  │
│  returned_at │  └──┬────────┘  │
│  price_cents │     │ N         │ N
│  ...         │     │           │
└───┬──────────┘  ┌──▼───────────▼──┐
    │ N           │   Reservation   │
    │             │  id  Uuid       │
┌───▼──────┐      │  status Enum    │
│   Loan   │      │  expires_at     │
│ id  Uuid │      └──┬──────────────┘
│ status   │         │ N
│ ...      │      ┌──▼──────────┐
└───┬──────┘      │  Customer   │
    │ N           │  id  Uuid   │
┌───▼──────┐      │  email UNIQ │
│ Penalty  │      │  deleted_at │  (soft delete)
│ id Uuid  │◄─────┤  ...        │
└──────────┘  N:1 └─────────────┘
```

> 💡 **Analogia — dlaczego diagram przed kodem.**
>
> Diagram relacji to plan mieszkania przed zakupem mebli. Można wstawić szafę do przedpokoju i potem odkryć, że nie przechodzi przez drzwi — ale taniej jest zmierzyć przedpokój. Każda zmiana modelu danych po napisaniu migracji to „przestawianie ścian” — wymaga nowej migracji, przepisania zapytań i poprawek testów. Pół godziny nad diagramem oszczędza dzień pracy.

### 2.2. Encje — przegląd

| Encja | Rola | Klucz główny | Uwagi |
|---|---|---|---|
| `Category` | kategoria sprzętu (narty, buty, rowery) | `Integer` identity | tabela słownikowa, edytowalna przez pracownika |
| `EquipmentItem` | fizyczny egzemplarz sprzętu | `Integer` identity | `code` (kod kreskowy) jest identyfikatorem zewnętrznym |
| `Tariff` | cennik: kategoria × okres → cena za dobę | `Integer` identity | historyczny — wiersze się nie nadpisują |
| `Customer` | klient | `Uuid` | `deleted_at` — soft delete |
| `Reservation` | rezerwacja jednego egzemplarza na okres | `Uuid` | wygasa po 15 minutach |
| `Loan` | wypożyczenie (nagłówek) | `Uuid` | obejmuje 1..N egzemplarzy |
| `LoanItem` | pozycja wypożyczenia (tabela pośrednia) | `Integer` identity | **tabela asocjacyjna z kolumnami** |
| `Penalty` | naliczona kara za przetrzymanie | `Uuid` | po jednej na pozycję |
| `ItemStatusEvent` | dziennik zdarzeń egzemplarza | `Integer` identity | **append-only**, źródło historii |

> 🧠 **Dlaczego tak jest** — zwróć uwagę, że nie ma osobnej encji `Return` ani `Availability`. Zwrot to cecha pozycji wypożyczenia (`LoanItem.returned_at`), a dostępność to **zapytanie**, nie dana. Zagadnienie „gdzie trzymać stan” omawiamy szczegółowo w punkcie 2.3.4.

### 2.3. Uzasadnienie decyzji projektowych

Ta sekcja jest sercem modułu. Każdy podrozdział odpowiada na pytanie „dlaczego właśnie tak, a nie inaczej”. W prawdziwym projekcie takie uzasadnienia trafiają do dokumentu `docs/adr/` (Architecture Decision Records) — plików z decyzjami architektonicznymi. Ty też je zapiszesz.

#### 2.3.1. `Uuid` czy `Integer` jako klucz główny?

Decyzja: **mieszamy**, i to celowo.

- `Uuid` dla encji, których identyfikator **wychodzi na zewnątrz** (`Customer`, `Loan`, `Reservation`, `Penalty`).
- `Integer` identity dla encji **wewnętrznych** (`Category`, `EquipmentItem`, `Tariff`, `LoanItem`, `ItemStatusEvent`).

Argumenty za `Uuid` w warstwie zewnętrznej:

1. **Nie ujawnia skali biznesu.** Numer `12` na fakturze mówi konkurentowi, że masz 12 klientów. `Uuid` nie mówi nic.
2. **Nie da się zgadnąć.** Klient nie może podmienić `.../loans/13` na `.../loans/14` i zobaczyć cudze wypożyczenie. To nie zastępuje autoryzacji — ale to dodatkowa warstwa.
3. **Umożliwia generowanie identyfikatora po stronie aplikacji.** Możesz zbudować obiekt `Loan` z pełnym `id`, zanim baza cokolwiek wie. Przy `Integer` identity identyfikator powstaje dopiero po `INSERT` i wymaga rundy do bazy.

Argument przeciw `Uuid` — **wydajność indeksu**. Klucz główny w PostgreSQL to domyślnie drzewo B-tree, a losowe `Uuid` (wersja 4) trafiają w losowe miejsca drzewa. Efekt: przy dużej tabeli każde wstawienie powoduje podział strony indeksu. Rozwiązania:

- `Uuid` wersji 7 (uporządkowany czasowo) — jeśli warstwa infrastruktury na to pozwala.
- Wewnętrzny `Integer` jako klucz tabeli, a `Uuid` tylko jako kolumna publiczna z unikalnym indeksem.

> 💡 **Analogia — klucz w zamku i numer w rejestrze.**
>
> `Uuid` to klucz, który fizycznie nie pasuje do żadnego innego zamka na świecie. `Integer` to numer w rejestrze — wygodny do sortowania i odwołań, ale każdy może odgadnąć, ile jest wpisów i który jest najnowszy. Mieszanie obu podejść to trzymanie zamka na drzwiach i rejestru w środku.

> 🆕 **SQLAlchemy 2.1** — typ `Uuid` w 2.1 nadal działa identycznie jak w 2.0, ale seria 2.1 dodała lepsze wsparcie typowania PEP 646 dla `Row`, dzięki czemu IDE dokładniej rozumie typy zwracane przez `result.one()`. Ma to znaczenie przy rzutowaniu `Row` na DTO — kompilator statyczny widzi więcej.

> ⚠️ **Pułapka — `default=uuid.uuid4` vs `server_default`.**
>
> Jeśli ustawisz `Mapped[uuid.UUID] = mapped_column(Uuid, default=uuid.uuid4)`, identyfikator powstaje **w Pythonie**. To dobrze — dostajesz `id` natychmiast po `session.add()`. Ale migracja wygenerowana przez Alembic nie ma w bazie żadnego `DEFAULT`. Jeśli jakikolwiek kod wstawia wiersz surowym SQL-em, nie dostanie identyfikatora. Alternatywa: `server_default=sa.text("gen_random_uuid()")` (PostgreSQL, wymaga rozszerzenia `pgcrypto` w starszych wersjach). Decyzję zapisz w ADR.

#### 2.3.2. Tabela pośrednia z kolumnami — `LoanItem`

`Loan` (wypożyczenie) i `EquipmentItem` (egzemplarz) łączą się relacją wiele-do-wiele. Klasyczna tabela asocjacyjna miałaby dwie kolumny: `loan_id` i `equipment_item_id`. To za mało. Między wypożyczeniem a egzemplarzem istnieją **własne dane**:

| Kolumna | Po co |
|---|---|
| `loan_id` | FK do nagłówka wypożyczenia |
| `equipment_item_id` | FK do egzemplarza |
| `price_per_day_cents` | **snapshot** ceny w chwili wypożyczenia |
| `due_at` | termin zwrotu tego konkretnego egzemplarza |
| `returned_at` | faktyczny moment zwrotu (`NULL` = jeszcze nie zwrócony) |
| `condition_on_return` | stan przy zwrocie (Enum) |

Gdy tabela pośrednia ma własne kolumny, w ORM nie mapujemy jej jako zwykłej `Table` w argumencie `secondary=`. Mapujemy ją jako **pełnoprawną klasę** (wzorzec *Association Object*, obiekt asocjacyjny) i linkujemy relacje przez nią.

Szkic — to sygnatury, nie pełny model. Pełen model napiszesz sam w Etapie 1:

```python
# examples/23_loan_item_sketch.py (szkic — NIE kopiuj jako gotowe rozwiązanie)
from datetime import datetime
from sqlalchemy import ForeignKey, Integer
from sqlalchemy.orm import Mapped, mapped_column, relationship


class LoanItem(Base):
    """Pozycja wypożyczenia: asocjacja Loan <-> EquipmentItem z własnymi danymi."""

    __tablename__ = "loan_items"

    id: Mapped[int] = mapped_column(primary_key=True)
    loan_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("loans.id", ondelete="CASCADE"))
    equipment_item_id: Mapped[int] = mapped_column(ForeignKey("equipment_items.id"))

    # snapshot ceny — nie odwołanie do cennika (patrz I8)
    price_per_day_cents: Mapped[int]
    due_at: Mapped[datetime]
    returned_at: Mapped[datetime | None]
    # ... condition_on_return i relacje
```

> 🔬 **Pod maską** — gdy zapisujesz wypożyczenie z dwiema pozycjami przez obiekty, ORM wygeneruje trzy instrukcje w jednej transakcji, w tej kolejności:

```sql
INSERT INTO loans (id, customer_id, status, created_at) VALUES (%(id)s, ...) RETURNING loans.id;
INSERT INTO loan_items (loan_id, equipment_item_id, price_per_day_cents, due_at, returned_at)
    VALUES (%(loan_id)s, %(equipment_item_id)s, ...) RETURNING loan_items.id;
INSERT INTO loan_items (loan_id, equipment_item_id, price_per_day_cents, due_at, returned_at)
    VALUES (%(loan_id)s, %(equipment_item_id)s, ...) RETURNING loan_items.id;
```

> Zwróć uwagę na `RETURNING` — to mechanizm, który pozwala bazie odesłać wygenerowany klucz bez drugiego zapytania. W PostgreSQL działa natywnie; w SQLite od wersji 3.35. To jeden z powodów, dla których „testuję tylko na SQLite” bywa ryzykowne (patrz moduł 18).

> ⚠️ **Pułapka — `secondary=` przy tabeli z kolumnami.**
>
> Naturalny odruch po module 09 to napisanie `relationship(secondary=loan_items_table)`. To działa **tylko** wtedy, gdy tabela pośrednia ma wyłącznie dwie kolumny kluczowe. Przy dodatkowych kolumnach stracisz możliwość ustawienia i odczytania `price_per_day_cents` przez ORM — SQLAlchemy nie ma gdzie ich zapisać. Musisz użyć wzorca Association Object: klasa `LoanItem`, relacja `Loan.items: Mapped[list[LoanItem]]` i `LoanItem.equipment_item: Mapped[EquipmentItem]`.

> 🧪 **Ćwiczenie —** Zapisz w `docs/adr/0002-association-object.md` jedno zdanie: dlaczego w tym projekcie wybrałeś obiekt asocjacyjny zamiast prostej tabeli `secondary`. Jedno zdanie, nie akapit. ADR-y mają być krótkie, bo inaczej nikt ich nie przeczyta.

#### 2.3.3. Snapshot ceny — dlaczego kopiujemy, a nie odwołujemy

Niezmiennik **I8** mówi: cena w wypożyczeniu nie zmienia się, gdy zmienia się cennik. Realizujemy to, kopiując `price_per_day_cents` do `LoanItem` w momencie wypożyczenia.

Alternatywa (odwołanie do `Tariff.id`) brzmi elegancko — jedna prawda, brak duplikacji. Ale ma katastrofalną wadę: **zmiana cennika zmienia historię**. Gdyby pracownik podniósł cenę rowerów o 10 zł, wszystkie historyczne wypożyczenia z dnia na dzień „podrożałyby”. Rachunki już wystawione przestałyby się zgadzać z systemem.

> 💡 **Analogia — paragon i cennik w sklepie.**
>
> Cennik w sklepie to `Tariff`. Paragon to `LoanItem`. Gdy sklep podniesie cenę masła w poniedziałek, paragon z soboty nadal pokazuje starą cenę — bo paragon jest **kopią stanu w momencie zakupu**, nie odnośnikiem do cennika. Gdyby paragon był odnośnikiem, każda zmiana ceny przepisałaby wszystkie paragony w historii. Nikt tego nie chce.

> 🧠 **Dlaczego tak jest** — to ogólna zasada w systemach finansowych i rozliczeniowych: **dane historyczne muszą być niezmienne**. Dotyczy to cen, kursów walut, stawek VAT, prowizji. Wzorzec nazywa się *snapshot* albo *point-in-time record*. W tym projekcie stosujemy go w `LoanItem.price_per_day_cents`. W rozszerzeniach (punkt 6) zastosujesz go samo także przy karach.

> 🧪 **Ćwiczenie —** Zaproponuj test, który **dowodzi**, że I8 jest zachowany. Zapis: „utwórz wypożyczenie, zmień cennik, odczytaj wypożyczenie, sprawdź, że cena się nie zmieniła”. Ten test jest ważniejszy niż test endpointu `POST /loans` — bo chroni niezmiennik, a nie funkcję.

#### 2.3.4. Tabela historii zamiast kolumny statusu

To najciekawsza decyzja w całym projekcie. Rozważmy dwa podejścia:

**Podejście A — kolumna statusu.** `EquipmentItem.status` przyjmuje wartości `AVAILABLE`, `RESERVED`, `ON_LOAN`, `RETIRED`. Jedna kolumna, proste zapytania, szybki odczyt.

**Podejście B — tabela zdarzeń.** `ItemStatusEvent` to dziennik tylko do dopisywania: `(item_id, event_type, occurred_at, actor, loan_id, note)`. Bieżący stan to ostatnie zdarzenie lub zapytanie po tabelach operacyjnych.

Wybór: **B jako źródło prawdy, a dostępność wyliczamy zapytaniem o `Loan` i `Reservation`**. Nie dodajemy kolumny `status` do `EquipmentItem`.

| Kryterium | Kolumna statusu | Tabela zdarzeń |
|---|---|---|
| Szybkość odczytu bieżącego stanu | natychmiastowa | wymaga zapytania lub agregacji |
| Możliwość rozjazdu | wysoka — stan w bazie może kłamać | brak — stan jest pochodną faktów |
| Audyt („kto i kiedy”) | żaden | pełny, wbudowany |
| Rozliczenie sporu z klientem | „system tak mówi” | „proszę, oto cała historia z datami” |
| Złożoność kodu | niska | wyższa, ale jednorazowa |

> 🧠 **Dlaczego tak jest** — kolumna statusu to **duplikacja prawdy**. Jeśli `Loan` i `Reservation` mówią jedno, a `EquipmentItem.status` drugie, to która jest prawdziwa? Każda denormalizacja tworzy pytanie „co, jeśli się rozjadą?”. Rozjazd można ograniczyć, ale nie można go wyeliminować — zawsze znajdzie się ścieżka kodu (albo ręczna poprawka w bazie, albo przerwane wdrożenie), która zaktualizuje jedno, a nie drugie. Tabela zdarzeń nie ma tego problemu, bo **nie przechowuje stanu** — przechowuje fakty, z których stan wynika.

> 💡 **Analogia — kartoteka i dziennik pokładowy.**
>
> Kolumna statusu to tabliczka na drzwiach: „w pokoju”. Łatwo ją odwiesić i zapomnieć zmienić. Tabela zdarzeń to dziennik pokładowy samolotu: „o 8:05 wszedł pasażer, o 8:12 wyszedł”. Tabliczka może kłamać; dziennik nie, bo nikt go nie „poprawia” — tylko dopisuje kolejne wpisy.

> 🔬 **Pod maską** — „jaki jest bieżący stan egzemplarza” to w podejściu B zapytanie po faktach:

```sql
-- Czy egzemplarz ma aktywne wypożyczenie?
SELECT EXISTS (
    SELECT 1 FROM loan_items li
    WHERE li.equipment_item_id = %(item_id)s
      AND li.returned_at IS NULL
) AS has_active_loan;

-- Czy egzemplarz ma aktywną, nieprzeterminowaną rezerwację?
SELECT EXISTS (
    SELECT 1 FROM reservations r
    WHERE r.equipment_item_id = %(item_id)s
      AND r.status IN ('PENDING', 'CONFIRMED')
      AND r.expires_at > now()
) AS has_active_reservation;
```

> To dwa zapytania. Przy 10 000 egzemplarzy na liście **nie wolno** ich wykonywać w pętli — powstanie N+1 (moduł 11), tylko trzeba je złączyć w jedno zapytanie z agregacją albo `LEFT JOIN LATERAL`. To konkretne zadanie w Etapie 7.

> ⚠️ **Pułapka — historia, która nie jest tylko do dopisywania.**
>
> Łatwo napisać kod, który „poprawia” zdarzenie: `event.event_type = "RETURNED"`. Wtedy dziennik przestaje być dziennikiem. Zabezpieczenie: w modelu `ItemStatusEvent` żaden atrybut nie ma `onupdate`, a w repozytorium **nie ma** metody `update` ani `delete` dla tej encji. Ograniczenie egzekwuj kodem, nie dobrą wolą. Jeśli chcesz pójść dalej — dodaj trigger w PostgreSQL blokujący `UPDATE` i `DELETE` na tej tabeli.

> 🆕 **SQLAlchemy 2.1** — seria 2.1 dodała konstrukcje `CreateView` i `CREATE TABLE AS SELECT` w warstwie Core. W tym projekcie nie są potrzebne, ale to dobry moment, żeby wiedzieć o ich istnieniu: gdyby w Etapie 7 okazało się, że „bieżąca dostępność” jest odpytywana w każdym requestcie i zaczyna boleć, naturalnym krokiem jest widok zmaterializowany albo tabela pochodna — i właśnie te konstrukcje pozwalają zdefiniować je po stronie SQLAlchemy zamiast surowym SQL-em w migracji.

#### 2.3.5. Soft delete — gdzie tak, gdzie nie

| Encja | Decyzja | Uzasadnienie |
|---|---|---|
| `Customer` | **soft delete** (`deleted_at`) | historia wypożyczeń i kar musi pozostać; usunięcie klienta nie może usuwać faktów |
| `Tariff` | **soft delete** (`valid_to`) | cennik jest historyczny — wiersze się zamykają, nie kasują |
| `EquipmentItem` | **brak** — wycofanie przez zdarzenie `RETIRED` | egzemplarz nadal istnieje fizycznie; historia musi zostać |
| `Loan`, `LoanItem`, `Penalty` | **brak hard delete** | dokumenty rozliczeniowe — nie usuwamy |
| `Reservation` | **hard delete** wyłącznie dla `EXPIRED` starszych niż 90 dni | dane tymczasowe, brak wartości historycznej |

> 💡 **Analogia — akt osobowy i kosz na śmieci.**
>
> Soft delete to wyciągnięcie kartoteki z segregatora i wrzucenie do szuflady „archiwum”. Hard delete to wyrzucenie kartoteki do kosza i spalenie. W systemach rozliczeniowych prawie zawsze chodzi o archiwum. Prawdziwe usunięcie jest zarezerwowane dla danych, które naprawdę nie mają wartości — na przykład niepotwierdzonych rezerwacji z zeszłego roku.

> ⚠️ **Pułapka — soft delete, o którym zapomnisz w jednym miejscu.**
>
> `Customer.deleted_at IS NULL` trzeba dopisać do **każdego** zapytania o klientów. Wcześniej czy później zapomnisz — i usunięty klient pojawi się na liście. Dwa rozwiązania: (a) jedno repozytorium budujące wszystkie zapytania o klientów, (b) filtr globalny. SQLAlchemy oferuje mechanizm `with_loader_criteria`, który pozwala zastosować warunek do wszystkich zapytań danej encji. To temat z pogranicza modułów 11 i 13 — zaimplementuj go sam, jeśli chcesz podnieść jakość.

#### 2.3.6. `Enum` czy tabela słownikowa?

| Dane | Wybór | Dlaczego |
|---|---|---|
| `ReservationStatus` (`PENDING`, `CONFIRMED`, `EXPIRED`, `CANCELLED`) | `Enum` | zbiór zamknięty, zmienia się raz na kilka lat, logika zależy od każdej wartości |
| `LoanStatus` (`ACTIVE`, `CLOSED`) | `Enum` | jak wyżej; wartości dwie |
| `ItemCondition` (`NEW`, `GOOD`, `WORN`, `DAMAGED`) | `Enum` | logika progowa (np. `DAMAGED` blokuje wydanie) |
| `Category` | **tabela słownikowa** | pracownik dodaje kategorie sam, bez wdrożenia |
| `EquipmentEventType` (`RESERVED`, `LOANED`, `RETURNED`, `RETIRED`, `REPAIRED`) | `Enum` | zbiór zdarzeń zdefiniowany przez logikę aplikacji |

Reguła decyzyjna: **jeśli dodanie nowej wartości wymaga zmiany kodu — `Enum`. Jeśli dodanie nowej wartości jest operacją biznesową wykonywaną przez użytkownika — tabela słownikowa.**

> 🧠 **Dlaczego tak jest** — `Enum` w bazie danych to `VARCHAR` z ograniczeniem `CHECK` (albo natywny typ `ENUM` w PostgreSQL). Dodanie wartości wymaga migracji — czyli wdrożenia. To jest zaleta, nie wada: skoro dodanie statusu `PARTIALLY_RETURNED` zmieniłoby zachowanie aplikacji, to **nie chcemy**, żeby ktoś dodał go przez SQL bez zmiany kodu. Odwrotnie z kategoriami: dodanie „deski snowboardowe” nie zmienia ani linii kodu, więc powinno być operacją w panelu pracownika.

> ⚠️ **Pułapka — natywny `Enum` PostgreSQL i migracje.**
>
> Przy `sa.Enum(ReservationStatus, native_enum=True)` (domyślnie w PostgreSQL) dodanie wartości do istniejącego typu to osobna operacja `ALTER TYPE ... ADD VALUE`. Alembic **nie wygeneruje** jej z `--autogenerate` — zobaczysz pustą migrację i zdziwienie. Rozwiązania: (a) `native_enum=False` — wtedy wartości są w `VARCHAR` + `CHECK`, a Alembic wykrywa zmianę jako `alter_column`; (b) ręczna migracja z `op.execute("ALTER TYPE ...")`. Decyzję zapisz w ADR — to klasyczna pułapka produkcyjna opisana też w module 16.

#### 2.3.7. Indeksy — które i po co

Indeksy dodajemy dopiero po pomiarze (moduł 17), ale trzy z nich wynikają wprost z niezmienników i są znane z góry:

| Indeks | Na czym | Po co |
|---|---|---|
| **I6** — unikalny, częściowy | `reservations (equipment_item_id) WHERE status IN ('PENDING','CONFIRMED')` | blokada na poziomie bazy: klient nie może mieć dwóch aktywnych rezerwacji tego samego egzemplarza |
| **I1** — unikalny, częściowy | `loan_items (equipment_item_id) WHERE returned_at IS NULL` | **ostatnia linia obrony**: baza nie pozwoli na dwa aktywne wypożyczenia tego samego egzemplarza, nawet jeśli kod aplikacji zawiedzie |
| wydajnościowy | `loan_items (equipment_item_id, returned_at)` | sprawdzanie dostępności egzemplarza |
| wydajnościowy | `reservations (expires_at) WHERE status = 'PENDING'` | zadanie wygaszające rezerwacje |

> 💡 **Analogia — kłódka i strażnik.**
>
> Warunek w kodzie aplikacji to strażnik przy drzwiach: sprawny, ale może zasnąć, może zachorować, może dojść do wyścigu. Ograniczenie w bazie to kłódka na drzwiach: nie śpi, nie choruje, nie da się jej obejść. **Miej jedno i drugie.** Aplikacja daje ładny komunikat błędu („sprzęt już zarezerwowany”), baza daje gwarancję. Bez kłódki prędzej czy później ktoś znajdzie lukę; bez strażnika użytkownik zobaczy brzydki `IntegrityError` zamiast zrozumiałego komunikatu.

> 🔬 **Pod maską** — częściowy indeks unikalny w PostgreSQL wygląda tak:

```sql
CREATE UNIQUE INDEX uq_loan_items_active_item
    ON loan_items (equipment_item_id)
    WHERE returned_at IS NULL;
```

> W SQLAlchemy deklarujesz go w `__table_args__`:

```python
# examples/23_partial_index_sketch.py (szkic)
from sqlalchemy import Index, text

__table_args__ = (
    Index(
        "uq_loan_items_active_item",
        "equipment_item_id",
        unique=True,
        postgresql_where=text("returned_at IS NULL"),
        sqlite_where=text("returned_at IS NULL"),
    ),
)
```

> ⚠️ **Pułapka — indeks częściowy a Alembic.**
>
> Alembic z `--autogenerate` **nie zawsze** wykrywa indeksy częściowe i zmiany w warunku `WHERE`. Po wygenerowaniu migracji **przeczytaj ją zawsze** i dopisz brakujący `op.create_index` ręcznie, z parametrem `postgresql_where`. To dokładnie ta zasada z modułu 16: „zawsze przeglądaj wygenerowany plik”.

### 2.4. Podsumowanie modelu w jednym akapicie

Świat składa się z kategorii, a kategoria ma egzemplarze i cenniki. Klient robi rezerwację na jeden egzemplarz, a pracownik rejestruje wypożyczenie, które może obejmować wiele egzemplarzy — dlatego wypożyczenie ma pozycje. Każda pozycja pamięta cenę z chwili wypożyczenia i moment zwrotu. Kara jest osobnym faktem rozliczeniowym. Każde zdarzenie egzemplarza trafia do dziennika, którego nikt nie poprawia. Dostępność nie jest przechowywana — jest wyliczana. Wszystko, co mówi „nie wolno”, jest zabezpieczone i w kodzie, i w bazie.

---

## 3. Wymagania techniczne i definicja „gotowe”

### 3.1. Stack technologiczny

| Warstwa | Narzędzie | Wersja | Uwagi |
|---|---|---|---|
| Język | Python | ≥ 3.11 | unions przez `\|`, wbudowane generyki |
| ORM | SQLAlchemy | ≥ 2.0 | styl 2.0: `select()`, `Mapped`, `mapped_column` |
| Sterownik SQLite | `aiosqlite` | aktualny | szybki start, zero konfiguracji |
| Sterownik PostgreSQL | `asyncpg` | aktualny | baza docelowa |
| Migracje | Alembic | 1.19.x | autogenerate wykrywa nazwane ograniczenia `CHECK` |
| Web | FastAPI | aktualny | z `pydantic-settings` |
| Walidacja | Pydantic | v2 | `from_attributes=True` |
| Testy | pytest + pytest-asyncio | aktualne | plus `httpx.AsyncClient` do testów API |
| Jakość | ruff + mypy | aktualne | formatowanie i typowanie |
| Konteneryzacja | Docker + Docker Compose | — | PostgreSQL lokalnie |

> 🆕 **SQLAlchemy 2.1** — jeśli zdecydujesz się eksperymentować z serią 2.1, pamiętaj o trzech różnicach, które dotkną tego projektu: (1) 2.1 wymaga Pythona ≥ 3.11, (2) domyślnym sterownikiem dla `postgresql://` jest `psycopg`, a nie `psycopg2`, więc URL-e bez jawnego dialektu zachowają się inaczej, (3) `greenlet` nie instaluje się automatycznie — potrzebujesz `pip install "sqlalchemy[asyncio]"`. Projekt zaliczeniowy rób na 2.0.x; 2.1 traktuj jako eksperyment na boku.

### 3.2. Środowisko lokalne

Dwie ścieżki uruchomienia. Obydwie mają działać.

**Ścieżka A — SQLite (szybki start).** Zero konfiguracji, plik `vererent.db` w katalogu projektu. Zastrzeżenie: brak natywnego `ENUM`, inne zachowanie przy współbieżności (jeden piszący naraz), brak `SELECT FOR UPDATE` w pełnym znaczeniu, inna obsługa `ALTER TABLE` w migracjach (wymaga `render_as_batch=True`).

**Ścieżka B — PostgreSQL w Dockerze.** Docelowa. Wymagana do testu współbieżności z Etapu 5, bo SQLite nie jest w stanie pokazać realnego wyścigu między dwoma połączeniami.

```yaml
# examples/23_docker_compose_sketch.yaml (szkic)
services:
  db:
    image: postgres:17
    environment:
      POSTGRES_USER: vererent
      POSTGRES_PASSWORD: vererent
      POSTGRES_DB: vererent
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

> ⚠️ **Pułapka — rozjazd zachowań między SQLite i PostgreSQL.**
>
> Najczęstszy scenariusz porażki: cały projekt przechodzi testy na SQLite, a po zmianie URL-a na PostgreSQL sypie się przy pierwszej migracji i pierwszym zapytaniu współbieżnym. Cztery konkretne różnice, które Cię dotkną: (1) `sa.Enum` jest natywny w PostgreSQL, a w SQLite to `VARCHAR` + `CHECK`; (2) `render_as_batch=True` jest potrzebne tylko dla SQLite, ale nie zaszkodzi PostgreSQL; (3) `FOR UPDATE` w SQLite jest ignorowane po cichu — test wyścigu „przejdzie”, choć nic nie blokuje; (4) `now()` vs `CURRENT_TIMESTAMP` mają różne typy zwracane. Rozwiązanie: **testy integracyjne uruchamiaj na obu bazach** (fixture parametryzowana), a test współbieżności tylko na PostgreSQL.

### 3.3. Struktura projektu

```text
vererent/
├── app/
│   ├── __init__.py
│   ├── main.py                 # tworzenie aplikacji FastAPI, lifespan
│   ├── core/
│   │   ├── config.py           # Settings z pydantic-settings
│   │   └── exceptions.py       # wyjątki aplikacyjne
│   ├── db/
│   │   ├── base.py             # DeclarativeBase, mixiny
│   │   ├── session.py          # engine, sessionmaker, get_session
│   │   └── naming.py           # naming_convention dla MetaData
│   ├── models/
│   │   ├── category.py
│   │   ├── equipment.py        # EquipmentItem, ItemStatusEvent
│   │   ├── tariff.py
│   │   ├── customer.py
│   │   ├── reservation.py
│   │   ├── loan.py             # Loan, LoanItem
│   │   └── penalty.py
│   ├── repositories/
│   │   ├── base.py
│   │   ├── equipment.py
│   │   ├── reservation.py
│   │   ├── loan.py
│   │   └── customer.py
│   ├── services/
│   │   ├── reservation_service.py
│   │   ├── rental_service.py   # wypożyczenie i zwrot
│   │   ├── pricing_service.py
│   │   └── reporting.py
│   ├── schemes/
│   │   ├── equipment.py
│   │   ├── reservation.py
│   │   └── loan.py
│   ├── api/
│   │   ├── deps.py             # zależności: UoW, sesja, paginacja
│   │   └── routers/
│   │       ├── catalog.py
│   │       ├── reservations.py
│   │       ├── loans.py
│   │       └── reports.py
│   └── uow.py                  # UnitOfWork
├── alembic/
│   ├── env.py
│   └── versions/
├── tests/
│   ├── conftest.py
│   ├── unit/
│   ├── integration/
│   └── concurrency/
├── scripts/
│   ├── seed.py
│   ├── bench_listing.py
│   └── race_reservation.py
├── docs/adr/
├── docker-compose.yaml
├── alembic.ini
├── pyproject.toml
├── .env.example
└── README.md
```

> 🧠 **Dlaczego tak jest** — struktura katalogów jest mapą zależności. `models/` nie wie o `schemas/`; `schemas/` nie wie o `repositories/`; `api/` nie wie o `db/` bezpośrednio, tylko przez `uow.py` i `deps.py`. Jeśli złamiesz tę zasadę (np. zaimportujesz `AsyncSession` w pliku z logiką biznesową w `services/`), kod nadal będzie działać — ale przestanie być testowalny bez bazy. To jest granica, którą chronisz (moduły 19–21).

### 3.4. Definicja „gotowe”

Projekt jest gotowy, gdy **wszystkie** poniższe zdania są prawdziwe:

1. `docker compose up -d db && alembic upgrade head && uvicorn app.main:app` uruchamia aplikację na czystej bazie bez żadnego ręcznego kroku.
2. `pytest` przechodzi w całości (unit + integracja + async), na SQLite i na PostgreSQL.
3. `alembic downgrade base && alembic upgrade head` przechodzi bez błędu na czystej bazie.
4. Skrypt `scripts/race_reservation.py` przy 100 równoległych rezerwacjach tego samego egzemplarza raportuje **dokładnie jeden** sukces i 99 kontrolowanych odrzuceń.
5. Skrypt `scripts/bench_listing.py` na 10 000 egzemplarzy pokazuje p95 < 300 ms i **stałą liczbę zapytań**, niezależną od rozmiaru strony.
6. `mypy app/` nie zgłasza błędów; `ruff check app/ tests/` jest czysty.
7. `grep -rn "create_all" app/` nie zwraca nic.
8. `grep -rnE "f\".*SELECT|f'.*SELECT" app/` nie zwraca nic.
9. `README.md` pozwala osobie z zewnątrz uruchomić projekt w 10 minut bez pytania Cię o cokolwiek.
10. W `docs/adr/` są co najmniej cztery decyzje architektoniczne z krótkim uzasadnieniem.

> 💡 **Analogia — definicja „gotowe” jak lista kontrolna pilota.**
>
> Pilot nie uznaje samolotu za gotowy do startu, gdy „wygląda dobrze”. Ma listę: klapy, paliwo, hydraulika, drzwi. Jeśli którykolwiek punkt nie jest odhaczony, samolot nie startuje — nawet jeśli wszystko inne działa idealnie. Twoja definicja „gotowe” działa tak samo. Projekt, który spełnia 9 z 10 punktów, **nie jest gotowy** — i to jest cały sens takiej listy.

> 🧪 **Ćwiczenie —** Stwórz plik `CHECKLIST.md` i dosłownie wklej do niego dziesięć punktów z tej sekcji. Po każdym etapie pracy przejdź po liście i odhacz, co już działa. To nie jest biurokracja — to jedyny sposób, żeby nie odkryć w ostatnim dniu, że nie umiesz odtworzyć bazy od zera.

---

## 4. Etapy pracy

Dziesięć etapów. Każdy ma cztery części: **zadania**, **jak sprawdzić, że skończone**, **podpowiedź** i **moduł do powtórki**. Pracuj etapami. Nie zaczynaj kolejnego, dopóki poprzedni nie jest zweryfikowany — to nie jest zalecenie ostrożnościowe, to jest metoda pracy, która chroni Cię przed sytuacją „nie wiem, gdzie jest błąd, bo zmieniłem dwadzieścia rzeczy naraz”.

### Etap 0 — Środowisko, struktura projektu, konfiguracja

**Zadania**

1. Utwórz środowisko wirtualne, `pyproject.toml` z zależnościami i `ruff`/`mypy` w konfiguracji.
2. Zbuduj strukturę katalogów dokładnie według drzewa z sekcji 3.3 (puste pliki `__init__.py` są w porządku).
3. Napisz `app/core/config.py` z klasą `Settings` opartą na `pydantic-settings`: `database_url`, `app_env`, `debug`, `echo_sql`, `reservation_ttl_minutes = 15`, `penalty_per_day_cents`. Wszystko z ENV, plik `.env.example` w repozytorium, `.env` w `.gitignore`.
4. Napisz `app/db/session.py`: `create_async_engine` z warunkowym `echo`, `async_sessionmaker` z `expire_on_commit=False`, funkcja `get_session()` jako zależność FastAPI.
5. Skonfiguruj `docker-compose.yaml` z PostgreSQL.
6. Skonfiguruj `logging` tak, aby na `DEBUG` logował SQL, a na `INFO` nie.

> 🔬 **Pod maską** — `create_async_engine` z jawnym URL-em dla `asyncpg`:

```python
# examples/23_engine_sketch.py (szkic)
from sqlalchemy.ext.asyncio import AsyncEngine, create_async_engine

engine: AsyncEngine = create_async_engine(
    "postgresql+asyncpg://vererent:vererent@localhost:5432/vererent",
    echo=settings.echo_sql,        # SQL w logach tylko gdy debug
    pool_size=10,                  # pula połączeń
    max_overflow=20,               # szczytowe dobicie
    pool_pre_ping=True,            # sprawdzenie „żyje?” przed użyciem
    pool_recycle=1800,             # odświeżenie po 30 minutach
)
```

**Jak sprawdzić, że skończone**

- `python -c "from app.core.config import Settings; print(Settings())"` wypisuje konfigurację.
- Zmiana `DATABASE_URL` w `.env` zmienia bazę docelową bez dotknięcia kodu.
- `docker compose up -d db` startuje bazę; `app/db/session.py` importuje się bez błędu.
- W logach na `DEBUG` widać SQL, na `INFO` nie widać.

**Podpowiedź** — `expire_on_commit=False` jest w trybie async niemal obowiązkowe. Powód techniczny: domyślnie `True` powoduje, że po commicie SQLAlchemy próbuje odświeżyć obiekty, wykonując zapytanie — a to jest operacja wejścia/wyjścia, która w kontekście async wymaga `await`. Ponieważ nie ma tam `await`, dostaniesz `MissingGreenlet`. Szczegóły w module 15.

**Moduł do powtórki:** `22_fastapi_integracja.md`, `02_srodowisko_i_engine.md`.

---

### Etap 1 — Schemat i modele

**Zadania**

1. W `app/db/base.py` zdefiniuj `class Base(DeclarativeBase)` z `metadata` ustawioną `naming_convention`.
2. Zdefiniuj mixiny: `IdMixin` (`Integer` identity), `UuidMixin` (`Uuid` + `default=uuid.uuid4`), `TimestampMixin` (`created_at`, `updated_at` z `server_default=func.now()` i `onupdate`).
3. Napisz wszystkie modele z sekcji 2.2. Każdy w osobnym pliku, każdy w pełni otypowany.
4. Zdefiniuj relacje: `Loan.items`, `LoanItem.equipment_item`, `EquipmentItem.events`, `Customer.loans`, `Customer.reservations`, `Category.items`, `Category.tariffs`.
5. Ustaw strategie ładowania domyślne: relacje jeden-do-wielu jako `lazy="raise"` w modelach — **wymuszenie** świadomego eager loadingu.
6. Zdefiniuj `__table_args__` z indeksami częściowymi z sekcji 2.3.7.
7. Zdefiniuj `CheckConstraint` dla niezmienników, które da się wyrazić w SQL, np. `due_at > created_at`, `price_per_day_cents >= 0`.

> 🔬 **Pod maską** — nie musisz uruchamiać bazy, żeby zobaczyć DDL. Wystarczy:

```python
# examples/23_show_ddl.py
from sqlalchemy.schema import CreateTable
from sqlalchemy.dialects import postgresql
from app.db.base import Base
import app.models  # import rejestruje wszystkie modele w Base.metadata

for table in Base.metadata.sorted_tables:
    print(CreateTable(table).compile(dialect=postgresql.dialect()))
```

> `sorted_tables` sortuje tabele tak, by klucze obce wskazywały już istniejące tabele — dzięki temu nie zobaczysz błędu o brakującej tabeli. Zauważ, że importujesz pakiet `app.models`, a nie pojedyncze klasy: **efektem ubocznym importu jest rejestracja modeli w `Base.metadata`**. To klasyczna pułapka — jeśli nie zaimportujesz modułu z modelem, tabela „zniknie” z migracji (moduł 03).

**Jak sprawdzić, że skończone**

- Skrypt `scripts/show_ddl.py` drukuje pełny DDL dla wszystkich tabel bez błędów.
- W DDL widać wszystkie indeksy częściowe z warunkami `WHERE`.
- W DDL widać `FOREIGN KEY` i `CHECK`.
- `mypy app/models/` nie zgłasza błędów.

**Podpowiedź** — kolejność definiowania relacji z dwoma kluczami obcymi do tej samej tabeli wymaga jawnego `foreign_keys=[...]`. W tym projekcie nie masz takiego przypadku, ale jeśli w rozszerzeniach dodasz np. `Loan.created_by` i `Loan.closed_by` oba do `Customer` — już masz. Warto sprawdzić teraz, że rozumiesz mechanizm.

**Moduł do powtórki:** `03_metadata_ddl.md`, `07_modele_deklaratywne.md`, `09_relacje.md`, `12_typy_i_wlasne_typy.md`.

---

### Etap 2 — Migracje

**Zadania**

1. `alembic init alembic`, potem konfiguracja `alembic/env.py` pod async (`run_sync` z `connection.run_sync`) — wzorzec z modułu 16 i 15.
2. Ustaw `target_metadata = Base.metadata`, `compare_type=True`, `compare_server_default=True`.
3. Dodaj `render_as_batch=True` w `context.configure` — konieczne dla SQLite.
4. URL bazy czytany z `app.core.config.Settings` przez `config.set_main_option`, nie zapisany w `alembic.ini`.
5. Wygeneruj pierwszą migrację bez `--autogenerate`, ręcznie — chcesz mieć pełną kontrolę nad pierwszym schematem.
6. Wygeneruj drugą migrację przez `--autogenerate` i **porównaj z pierwszą**. Jeśli się różnią — to znaczy, że coś pominąłeś`.
7. Napisz `downgrade` dla obu.

**Jak sprawdzić, że skończone**

Po tym poleceniu nie ma prawa pojawić się żadna nowa migracja z niepustą treścią — to dowód, że model i baza są w zgodzie:

```bash
alembic upgrade head
alembic revision --autogenerate -m "sanity check"
# wygenerowany plik MUSI mieć puste upgrade() i downgrade()
```

- `alembic downgrade base && alembic upgrade head` przechodzi na czystej bazie.
- `alembic history` pokazuje dwa linearne komity, bez rozgałęzień.

> ⚠️ **Pułapka — „sanity check” zostawiony w repozytorium.**
>
> Wygenerowany plik kontrolny **usuń** po sprawdzeniu. Jeśli zostawisz pustą migrację w `versions/`, za tydzień ktoś (być może Ty) przejrzy historię i zgłupieje: po co komit, który nic nie robi? Gorzej — puste migracje psują narzędzia porównujące historię między środowiskami.

**Podpowiedź** — nazwy ograniczeń z `naming_convention` są kluczowe. Bez nich `--autogenerate` generuje w każdym uruchomieniu „zmienione” ograniczenia, bo nie znajduje nazwy, którą sam nadał. Konwencja z modułu 03 (`ix_%(column_0_label)s`, `uq_%(table_name)s_%(column_0_name)s`, `fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s`, `pk_%(table_name)s`, `ck_%(table_name)s_%(constraint_name)s`) rozwiązuje problem raz na zawsze.

**Moduł do powtórki:** `16_alembic_migracje.md`.

---

### Etap 3 — Dane słownikowe i seed

**Zadania**

1. `scripts/seed.py` — idempotentny skrypt zasiewający: kategorie (narty, buty, rowery, kaski), tarify na bieżący sezon, 200 egzemplarzy sprzętu z unikalnymi kodami, kilku klientów testowych.
2. Idempotentność przez `insert().on_conflict_do_nothing(index_elements=[...])` — uruchomienie skryptu dwa razy nie może zdublować danych.
3. `scripts/generate_equipment.py N` — generator `N` egzemplarzy do testów wydajnościowych (Etap 7), z realistycznym rozkładem kategorii.
4. Wszystko w jednej transakcji; przy błędzie — pełny rollback, żadnych „wpół zasianych” danych.

> 🔬 **Pod maską** — idempotentny insert 200 egzemplarzy wygląda tak:

```python
# examples/23_seed_sketch.py (szkic)
from sqlalchemy.dialects.postgresql import insert as pg_insert

stmt = pg_insert(EquipmentItem).values(rows)
stmt = stmt.on_conflict_do_nothing(index_elements=["code"])
await session.execute(stmt)
```

> Wygenerowany SQL (dla 200 wierszy — poglądowo skrócony):

```sql
INSERT INTO equipment_items (code, category_id, purchased_at, condition)
VALUES (%(code)s, %(category_id)s, %(purchased_at)s, %(condition)s)
ON CONFLICT (code) DO NOTHING;
```

> `INSERT ... ON CONFLICT` to konstrukcja PostgreSQL i SQLite. W MySQL nazywa się `INSERT ... ON DUPLICATE KEY UPDATE`, a SQLAlchemy wystawia ją przez `mysql.insert().on_duplicate_key_update()` — dialekt po dialekcie, jak to opisuje moduł 05.

**Jak sprawdzić, że skończone**

- `python scripts/seed.py` uruchomione dwa razy daje w bazie dokładnie 200 egzemplarzy, nie 400.
- `SELECT count(*) FROM equipment_items` po pierwszym i drugim uruchomieniu daje tę samą liczbę.
- W bazie są tarify z poprawnym zakresem `valid_from`/`valid_to`.

**Podpowiedź** — generowanie 200 wierszy jednym `insert().values([...])` jest znacznie szybsze niż 200 osobnych `session.add()`. Różnica w liczbie rund do bazy jest liniowa: $O(1)$ vs $O(N)$. To dokładnie ten eksperyment, który robiłeś w module 17 — tutaj masz okazję wykorzystać go w praktyce.

**Moduł do powtórki:** `05_dml.md`, `17_wydajnosc.md`.

---

### Etap 4 — Repozytoria i UoW

**Zadania**

1. `app/uow.py` — klasa `UnitOfWork` z `async_sessionmaker` w konstruktorze, kontekstem async (`__aenter__`/`__aexit__`), atrybutami repozytoriów i `commit()`/`rollback()`.
2. Protokoły repozytoriów (`typing.Protocol`) dla `EquipmentRepository`, `ReservationRepository`, `LoanRepository`, `CustomerRepository`.
3. Implementacje SQLAlchemy dla każdego repozytorium. Każda dostaje `AsyncSession` w konstruktorze. **Żadna nie commituje.**
4. Repozytorium `EquipmentRepository` z metodami: `get_by_code`, `list_available(category_id, start, end, page, size)`, `list_by_category`, `get_with_events(id)`.
5. Metoda `list_available` musi zwracać liczbę zapytań niezależną od rozmiaru strony.
6. Repozytorium `ReservationRepository` z `get_active_for_item(item_id)`, `get_active_for_customer(customer_id)`, `expire_stale(now)`.
7. Testy integracyjne repozytoriów na prawdziwej sesji (nie mockach).

> 🔬 **Pod maską** — `list_available` to najtrudniejsze zapytanie tego projektu. Musi wykluczyć egzemplarze, które mają nakładające się wypożyczenie albo aktywną rezerwację. Warunek nakładania się przedziałów to klasyk: dwa przedziały `[a, b]` i `[c, d]` **nakładają się wtedy i tylko wtedy, gdy** `a < d AND c < b`. W zapytaniu oznacza to:

```sql
SELECT e.id, e.code, ...
FROM equipment_items e
WHERE e.category_id = %(category_id)s
  AND NOT EXISTS (
      SELECT 1 FROM loan_items li
      WHERE li.equipment_item_id = e.id
        AND li.returned_at IS NULL
        AND li.due_at > %(start)s
  )
  AND NOT EXISTS (
      SELECT 1 FROM reservations r
      WHERE r.equipment_item_id = e.id
        AND r.status IN ('PENDING', 'CONFIRMED')
        AND r.expires_at > now()
        AND r.start_at < %(end)s
        AND r.end_at > %(start)s
  )
ORDER BY e.id
LIMIT %(limit)s OFFSET %(offset)s;
```

> To jest dokładnie ten rodzaj zapytania, dla którego SQLAlchemy Core błyszczy — piszesz `exists()` i `select()` zamiast sklejać stringi. Każdy warunek to metoda, każda wartość to parametr wiązany, żadnego f-stringa (N6).

**Jak sprawdzić, że skończone**

- Test: 100 egzemplarzy, 10 z nich z wypożyczeniami w danym okresie → `list_available` zwraca 90, niezależnie od `page`/`size`.
- Licznik zapytań (fixture z modułu 11) pokazuje **3 zapytania** dla dowolnej strony: `COUNT` + `SELECT` + pobranie kategorii. Jeśli widzisz 3 + N, masz N+1.
- Test UoW: wyjątek wewnątrz kontekstu powoduje rollback; obiekt nie trafia do bazy.
- `mypy app/repositories/ app/uow.py` jest czysty.

**Podpowiedź** — w `UnitOfWork` repozytoria tworzysz raz, w `__aenter__`, przekazując tę samą sesję. Dzięki temu wszystkie repozytoria działały w jednej transakcji i jednej identity map — to klucz do spójności (moduł 21). Jeśli każde repozytorium tworzyłoby własną sesję, commit w UoW nie obejmowałby ich zmian.

```python
# examples/23_uow_sketch.py (szkic)
class UnitOfWork:
    def __init__(self, session_factory: async_sessionmaker[AsyncSession]) -> None:
        self._session_factory = session_factory

    async def __aenter__(self) -> Self:
        self._session = self._session_factory()
        self.equipment = EquipmentRepository(self._session)
        # ... pozostałe repozytoria na tej samej sesji
        return self

    async def __aexit__(self, exc_type: object, exc: object, tb: object) -> None:
        if exc_type is not None:
            await self.rollback()
        await self._session.close()

    async def commit(self) -> None: ...
    async def rollback(self) -> None: ...
```

**Moduł do powtórki:** `20_repository.md`, `21_unit_of_work.md`, `11_ladowanie_i_n_plus_1.md`.

---

### Etap 5 — Przypadki użycia: rezerwacja, wypożyczenie, zwrot

To najważniejszy etap projektu. Tutaj rozstrzyga się, czy rozumiesz współbieżność, czy tylko o niej czytałeś.

**Zadania**

1. `ReservationService.create(customer_id, item_id, start, end)`:
   - sprawdź niezmienniki I1, I2, I6,
   - utwórz rezerwację ze statusem `PENDING` i `expires_at = now() + 15 min`,
   - obsłuż naruszenie indeksu częściowego (`IntegrityError`) jako kontrolowany wyjątek biznesowy.
2. `ReservationService.confirm(reservation_id)` — `PENDING → CONFIRMED`, obsługa I3 (jeśli wygasła, rzuć wyjątek domenowy).
3. `RentalService.checkout(customer_id, item_codes, days)`:
   - pobierz aktualne ceny z cennika,
   - **zablokuj egzemplarze** przez `with_for_update()`,
   - utwórz `Loan` + `LoanItem` ze snapshotem ceny i terminem zwrotu,
   - zapisz zdarzenia `LOANED` w `ItemStatusEvent`,
   - anuluj/oznacz jako wykorzystaną powiązaną rezerwację.
4. `RentalService.return_items(loan_id, item_codes, returned_at)`:
   - sprawdź I7 (nie zwracaj tego samego dwa razy),
   - ustaw `returned_at`, policz karę wg I4 (`ceil` dni przetrzymania),
   - zapisz `Penalty`, zapisz zdarzenia `RETURNED`,
   - zaktualizuj `Loan.status` na `CLOSED`, jeśli wszystkie pozycje zwrócone.
5. `RetirementService.retire(item_id, reason)` — wycofanie przez zdarzenie `RETIRED`, nie przez usunięcie.
6. Skrypt `scripts/race_reservation.py` — 100 równoległych prób na ten sam egzemplarz, oczekiwany wynik: 1 sukces, 99 odrzuceń.

> 💡 **Analogia — sprawdź i zrób, ale pod kluczem.**
>
> Wyobraź sobie jedne drzwi do magazynu i dwie osoby, które chcą wejść. Jeśli obie najpierw sprawdzą klamką, czy drzwi są otwarte, a potem obie wejdą — mamy bałagan. Rozwiązanie: **klucz w zamku**. Pierwsza osoba przekręca klucz, druga czeka. Dopiero gdy pierwsza wyjdzie i odda klucz, druga przekręca i wchodzi — i widzi, że drzwi są już zajęte. `SELECT ... FOR UPDATE` to ten klucz w zamku.

> 🔬 **Pod maską** — wersja naiwna, która **przegrywa wyścig**:

```python
# examples/23_race_bad_sketch.py (szkic — celowo BŁĘDNY)
item = await repo.get(item_id)            # SELECT ... WHERE id = 42
state = await repo.check_availability(item)  # SELECT ... EXISTS ...
if state.is_available:                    # ← tu wchodzi druga transakcja i widzi to samo
    loan = Loan(...)
    session.add(loan)
    await session.commit()                # obie commitują, obie „sukces”
```

> Wygenerowane SQL — dwie rundy, między nimi okno bez ochrony:

```sql
SELECT id, code FROM equipment_items WHERE id = 42;
SELECT EXISTS (SELECT 1 FROM loan_items WHERE equipment_item_id = 42 AND returned_at IS NULL);
INSERT INTO loans (...) VALUES (...) RETURNING loans.id;
```

> Wersja poprawna — jedna runda, blokada od początku:

```sql
SELECT id, code FROM equipment_items WHERE id = 42 FOR UPDATE;
-- od tego momentu druga transakcja CZEKA, dopóki pierwsza nie zrobi COMMIT albo ROLLBACK
INSERT INTO loans (...) VALUES (...) RETURNING loans.id;
INSERT INTO loan_items (...) VALUES (...) RETURNING loan_items.id;
```

> To właśnie `FOR UPDATE` (blokada pesymistyczna) rozwiązuje niezmienniki I1 i I2. Szczegóły — moduł 14, w szczególności sekcje o `nowait`, `skip_locked` i `of=`. Drugą linią obrony jest indeks częściowy z sekcji 2.3.7: nawet gdybyś zapomniał blokady, baza odrzuci drugi insert.

> ⚠️ **Pułapka — test współbieżności na SQLite.**
>
> SQLite **ignoruje** `FOR UPDATE`. Twój skrypt „100 równoległych rezerwacji” przejdzie na SQLite, bo SQLite i tak serializuje zapisy (jeden piszący naraz), więc wyścigu nie będzie widać. Ale nie znaczy to, że kod jest poprawny — po prostu test nie mierzy tego, co myślisz. **Test współbieżności uruchamiaj wyłącznie na PostgreSQL.** Zapisuj to w README jako jawny wymóg.

> 🧠 **Dlaczego tak jest** — blokada pesymistyczna to jedno z dwóch podejść. Drugie to blokada optymistyczna (kolumna `version`). W wypożyczalni sprzętu pesymistyczna jest lepsza, bo konflikt jest **częsty** — w szczycie sezonu dwie osoby dosłownie walczą o ten sam sprzęt. Blokada optymistyczna jest lepsza tam, gdzie konflikty są rzadkie, a transakcje długie (np. edycja formularza w przeglądarce przez 10 minut). Tabelę decyzyjną znajdziesz w module 14.

> 🧪 **Ćwiczenie —** Napisz skrypt `scripts/race_reservation.py` tak, aby **dowiódł** poprawności, a nie tylko „przeszedł”. Test musi: (a) uruchomić 100 równoległych korutyn przez `asyncio.gather`, (b) każda używa **własnej** sesji (nie wspólnej!), (c) zebrać wyniki w licznik sukcesów i odrzuceń, (d) **asercją** sprawdzić, że `sukceses == 1`. Bez asercji to nie jest test — to demonstracja.

**Jak sprawdzić, że skończone**

- `scripts/race_reservation.py` na PostgreSQL: dokładnie 1 sukces, 99 kontrolowanych wyjątków biznesowych. Uruchom pięć razy — wynik za każdym razem ten sam.
- Test karny: wypożyczenie na 3 dni, zwrot po 5 dniach i 1 godzinie → kara za 3 dni (dni 4, 5 i rozpoczęty dzień 6 — zgodnie z I4 „każdy rozpoczęty dzień”).
- Test I7: podwójny zwrot tej samej pozycji → drugi kończy się wyjątkiem domenowym, nie duplikatem w bazie.
- Test I5: historia zdarzeń egzemplarza rośnie, nigdy nie jest modyfikowana (sprawdź brak metod `update` na repozytorium zdarzeń).

**Podpowiedź** — kara liczona jako `ceil((returned_at - due_at) / 1 dzień)` daje „każdy rozpoczęty dzień”, ale **uważaj na strefy czasowe** (moduł 12). Trzymaj w bazie `DateTime(timezone=True)` w UTC, rób `ceil` na `timedelta` w UTC, dopiero na końcu konwertuj na czas lokalny do wyświetlenia. Najczęstszy błąd w tym projekcie to kara policzona o jedną dobę za dużo lub za mało z powodu strefy.

**Moduł do powtórki:** `14_transakcje_i_wspolbieznosc.md`, `13_zdarzenia_i_hybrydy.md`, `12_typy_i_wlasne_typy.md`.

---

### Etap 6 — API REST

**Zadania**

1. Schematy Pydantic v2 w `app/schemas/` z `model_config = ConfigDict(from_attributes=True)`.
2. **Zasada:** endpointy zwracają DTO, nigdy encje ORM. To chroni przed `DetachedInstanceError` i przed wyciekiem pól wewnętrznych.
3. Zależność `get_uow()` w `app/api/deps.py` — tworzy `UnitOfWork` z fabryki sesji, przekazuje do endpointu, commituje przy sukcesie, rollbackuje przy wyjątku.
4. Endpointy:

| Metoda | Ścieżka | Rola | Opis |
|---|---|---|---|
| `GET` | `/categories` | publiczny | lista kategorii |
| `GET` | `/equipment` | publiczny | lista egzemplarzy z filtrami (kategoria, dostępność w terminie), paginacja |
| `GET` | `/equipment/{code}` | publiczny | szczegóły egzemplarza |
| `GET` | `/equipment/{code}/history` | staff | historia zdarzeń |
| `POST` | `/reservations` | klient | utwórz rezerwację |
| `POST` | `/reservations/{id}/confirm` | klient | potwierdź |
| `DELETE` | `/reservations/{id}` | klient | anuluj |
| `GET` | `/customers/{id}/reservations` | klient | aktywne rezerwacje |
| `POST` | `/loans` | staff | zarejestruj wypożyczenie (wiele pozycji) |
| `POST` | `/loans/{id}/returns` | staff | zarejestruj zwrot (wiele pozycji) |
| `GET` | `/customers/{id}/loans` | klient | historia wypożyczeń |
| `POST` | `/equipment/{code}/retire` | staff | wycofaj sprzęt |
| `GET` | `/reports/utilization` | staff | raport wykorzystania |

5. Obsługa wyjątków domenowych przez `app.exception_handler` — spójny format odpowiedzi błędu.
6. Mapowanie `IntegrityError` na `409 Conflict` z czytelnym komunikatem.

> 🔬 **Pod maską** — endpoint listujący wypożyczenia klienta z pozycjami. Wersja **z N+1**:

```sql
SELECT * FROM loans WHERE customer_id = %(id)s;
SELECT * FROM loan_items WHERE loan_id = %(id_1)s;
SELECT * FROM loan_items WHERE loan_id = %(id_2)s;
SELECT * FROM loan_items WHERE loan_id = %(id_3)s;
-- ... i tak dalej, N razy
```

> Wersja poprawna — dwa zapytania, niezależnie od tego, ile jest wypożyczeń:

```sql
SELECT * FROM loans WHERE customer_id = %(id)s ORDER BY created_at DESC LIMIT 20;
SELECT * FROM loan_items WHERE loan_id IN (%(id_1)s, %(id_2)s, ..., %(id_20)s);
```

> Drugie zapytanie to `selectinload`, pierwsze to zwykłe `select`. Razem: $O(2)$ zamiast $O(1 + N)$. To nie jest optymalizacja na wyrost — to różnica między 200 ms a 4 s przy stu wypożyczeniach (moduł 11).

> ⚠️ **Pułapka — `BackgroundTasks` i sesja.**
>
> Jeśli użyjesz `BackgroundTasks` w FastAPI do wysłania e-maila po wypożyczeniu, pamiętaj: zadanie w tle wykonuje się **po** zakończeniu requestu, czyli po zamknięciu sesji. Sesji tam nie ma. Jeśli w zadaniu tła potrzebujesz danych z bazy, przekaż do niego **gotowe DTO**, nie encje i nie `AsyncSession`. W przeciwnym razie dostaniesz `DetachedInstanceError` albo `IllegalStateChangeError`. Ten sam problem dotyczy kolejki zadań (rozszerzenie w punkcie 6) — i tam rozwiązaniem jest wzorzec outbox.

**Jak sprawdzić, że skończone**

- `curl` na każdy endpoint zwraca sensowny kod i sensowny JSON.
- `POST /loans` z nieistniejącym kodem zwraca `422` lub `404`, nie `500`.
- Dwukrotne `POST /reservations` na ten sam egzemplarz dla tego samego klienta zwraca `409`.
- Dokumentacja `/docs` (Swagger) jest czytelna, wszystkie schematy mają opisane pola.
- `grep -rn "session.query" app/` nie zwraca nic (styl 2.0, spójność).

**Podpowiedź** — to najdłuższy etap i najłatwiej w nim stracić dyscyplinę architektoniczną. Pokusa: „wpiszę to jedno zapytanie bezpośrednio w endpoint, bo tak szybciej”. Konsekwencja: ten sam kod za tydzień ląduje w drugim endpoincie w innej wersji, a przy zmianie modelu trzeba pamiętać o obu. Reguła: **żaden endpoint nie zawiera zapytania SQLAlchemy ani logiki biznesowej** — tylko: walidacja wejścia, wywołanie serwisu, konwersja na DTO, zwrot.

**Moduł do powtórki:** `22_fastapi_integracja.md`, `19_warstwy_i_data_mapper.md`.

---

### Etap 7 — Raporty i optymalizacja

**Zadania**

1. `scripts/generate_equipment.py 10000` — wygeneruj 10 000 egzemplarzy, 1000 klientów, 5000 wypożyczeń historycznych i 200 aktywnych.
2. `GET /reports/utilization` — wykorzystanie sprzętu: dla każdej kategorii: liczba egzemplarzy, liczba wypożyczeń w okresie, średni czas wypożyczenia, procent wykorzystania.
3. `GET /reports/revenue` — przychód według kategorii i miesiąca, z użyciem funkcji okna (`sum() over (partition by ... order by ...)`) na narastająco.
4. `GET /equipment` — doprowadź do p95 < 300 ms na 10 000 egzemplarzy. Zmierz to skryptem `bench_listing.py`.
5. Sprawdź plan zapytania przez `EXPLAIN ANALYZE`. Dodaj indeksy, których brakuje. **Zapisz wyniki przed i po.**
6. Upewnij się, że wszystkie endpointy listujące mają ograniczony `limit` (maksymalnie 100) — brak limitu to „ładuj wszystko”.

> 🔬 **Pod maską** — raport przychodu narastająco, jedno zapytanie:

```sql
SELECT
    c.name AS category,
    date_trunc('month', li.due_at) AS month,
    sum(li.price_per_day_cents) AS revenue,
    sum(sum(li.price_per_day_cents)) OVER (
        PARTITION BY c.name
        ORDER BY date_trunc('month', li.due_at)
    ) AS revenue_running
FROM loan_items li
JOIN equipment_items e ON e.id = li.equipment_item_id
JOIN categories c ON c.id = e.category_id
GROUP BY c.name, date_trunc('month', li.due_at)
ORDER BY c.name, month;
```

> Zauważ trzy rzeczy. Pierwsza: `GROUP BY` jest wymagane, bo jest agregacja. Druga: funkcja okna nakłada się **na wynik po grupowaniu**, dlatego w `sum()` siedzi `sum()` — wewnętrzna sumuje wiersze w grupie, zewnętrzna sumuje grupy narastająco. Trzecia: `ORDER BY` wewnątrz `OVER` jest obowiązkowe. Bez niego baza może zwrócić wartości narastające w dowolnej kolejności i „running total” będzie bez sensu. To pułapka opisana w module 06.

> ⚠️ **Pułapka — mierz na realistycznych danych, nie na pustej bazie.**
>
> Zapytanie na pustej tabeli ma plan, w którym PostgreSQL wybiera pełny skan, bo to najszybsze. Po wstawieniu 10 000 wierszy plan się zmienia na indeksowy — albo odwrotnie, optymalizator decyduje, że pełny skan i tak jest lepszy. **Wniosek z pomiaru na pustej bazie jest bezwartościowy.** Zawsze mierz na danych o realistycznym rozkładzie i rozmiarze. Dobrą stroną jest to, że skrypt `generate_equipment.py` piszesz raz i używasz wielokrotnie.

> 🆕 **SQLAlchemy 2.1** — seria 2.1 dodała `selectinload(..., omit_join=True)` dla relacji wiele-do-wiele. Przy dużych grafach relacji oznacza to mniej roboczego `JOIN` i szybsze zapytania. Jeśli w raportach ładujesz kolekcje wiele-do-wiele (np. `Loan.items`), to jedna z nielicznych sytuacji, w której warto rozważyć 2.1 w projekcie eksperymentalnym — ale nie w wersji zaliczeniowej.

**Jak sprawdzić, że skończone**

- `bench_listing.py` raportuje p95 i liczbę zapytań. Liczba zapytań musi być **stała** dla rozmiarów strony 10, 50 i 100.
- W `docs/perf.md` masz tabelę „przed / po”: rozmiar danych, wskaźnik, wartość przed, wartość po, dodany indeks.
- Każdy dodany indeks ma uzasadnienie w postaci planu zapytania.
- Wszystkie endpointy listujące mają twarde `limit`.

**Podpowiedź** — najczęstszy powód wolnego listowania to nie brak indeksu, a **brak paginacji po stronie bazy**. Jeśli robisz `await session.scalars(select(EquipmentItem))` bez `LIMIT` i potem tniesz w Pythonie, to pobierasz 10 000 wierszy, mapujesz na 10 000 obiektów ORM i wyrzucasz 9 990. Mierzone w milisekundach: setki razy wolniej niż `LIMIT`. Ta sama uwaga dotyczy `count(*)` — policz w bazie, nie w Pythonie.

**Moduł do powtórki:** `17_wydajnosc.md`, `11_ladowanie_i_n_plus_1.md`, `06_joiny_i_zaawansowane_sql.md`.

---

### Etap 8 — Testy

**Zadania**

1. `tests/conftest.py`:
   - fixture `engine` (SQLite in-memory z `StaticPool`),
   - fixture `engine_pg` (parametryzowana, opcjonalna — PostgreSQL),
   - fixture `session` z izolacją przez transakcję zewnętrzną i rollback po teście,
   - fixture `uow` powiązana z sesją testową,
   - fixture `client` (`httpx.AsyncClient` z `ASGITransport`),
   - fixture `query_counter` — licznik zapytań z modułu 11.
2. Testy jednostkowe dla logiki cen i kar — bez bazy.
3. Testy integracyjne repozytoriów — na prawdziwej sesji.
4. Testy API — przez `httpx.AsyncClient`, na pełnym stosie.
5. Testy niezmienników — osobna grupa `tests/integration/test_invariants.py`, po jednym teście na każdy niezmiennik I1–I8.
6. `tests/concurrency/test_race.py` — test wyścigu, uruchamiany tylko na PostgreSQL, oznaczony markerem `@pytest.mark.pg`.
7. Test migracji: `alembic upgrade head`, wstaw dane, `alembic downgrade base`, `alembic upgrade head` — bez błędu.

> 🔬 **Pod maską** — fixture izolacji przez transakcję zewnętrzną (wzorzec z modułu 18):

```python
# tests/conftest.py (szkic)
import pytest_asyncio
from sqlalchemy.ext.asyncio import AsyncConnection, AsyncSession


@pytest_asyncio.fixture
async def session(engine) -> AsyncSession:
    async with engine.connect() as conn:
        trans = await conn.begin()          # zewnętrzna transakcja
        async with AsyncSession(bind=conn, join_transaction_mode="create_savepoint") as s:
            yield s
        await trans.rollback()              # wszystko znika po teście
```

> Każdy test dostaje **świeży widok** bazy, ale koszt `create_all` i `drop_all` ponosi się tylko raz na sesję testową. To ta różnica, o której mówiliśmy w module 18: tysiące testów działających w sekundach zamiast w minutach.

> ⚠️ **Pułapka — SQLite in-memory bez `StaticPool`.**
>
> Baza SQLite w pamięci istnieje **per połączenie**. Bez `StaticPool` każdy nowy `connect()` tworzy nową, pustą bazę — twoje tabele „znikają” między zapytaniami i testy wyglądają na zepsute bez powodu. Poprawna konfiguracja to `create_async_engine("sqlite+aiosqlite:///:memory:", poolclass=StaticPool, connect_args={"check_same_thread": False})`. To najczęstsza przyczyną tajemniczych awarii testów ORM.

**Jak sprawdzić, że skończone**

- `pytest -q` przechodzi w całości na SQLite.
- `pytest -q -m pg` przechodzi na PostgreSQL (z uruchomionym kontenerem).
- `pytest --cov=app/services --cov=app/repositories` pokazuje ≥ 80 %.
- Testy nie zależą od kolejności — uruchom `pytest --random-order` (lub w odwrotnej kolejności alfabetycznej plików), wynik musi być identyczny.
- Każdy z niezmienników I1–I8 ma test, którego nazwa zawiera numer niezmiennika.

**Podpowiedź** — testy niezmienników pisz **przed** implementacją usług. To nie jest dogmat TDD — to praktyczna korzyść: niezmiennik jest już zdefiniowany w specyfikacji, więc test napiszesz szybko, a implementacja dostanie natychmiastową informację zwrotną. „Test I1: dwa równoległe wypożyczenia tego samego egzemplarza → drugie rzuca `ItemNotAvailable`” to jedna funkcja, dziesięć linii.

**Moduł do powtórki:** `18_testowanie.md`.

---

### Etap 9 — Konteneryzacja, README, przegląd kodu

**Zadania**

1. `Dockerfile` dla aplikacji (wieloetapowy build: warstwa z zależnościami, warstwa z kodem).
2. `docker-compose.yaml` z usługami `db` i `api`, `healthcheck` dla bazy, `depends_on` z warunkiem gotowości.
3. `README.md`: opis projektu, wymagania wstępne, instrukcja uruchomienia w 10 minut, opis endpointów (albo link do `/docs`), opis testów, opis struktury projektu, **wymóg uruchomienia testu wyścigu na PostgreSQL**.
4. `docs/` — co najmniej cztery ADR-y: (a) UUID vs Integer, (b) Association Object, (c) tabela zdarzeń zamiast kolumny statusu, (d) snapshot ceny.
5. Przegląd kodu według listy z sekcji 7. Napraw wszystko, co znajdziesz.
6. `ruff check --fix app/ tests/ scripts/`, `ruff format app/ tests/ scripts/`, `mypy app/` — wszystko czyste.
7. Uruchom od zera na czystym klonie repozytorium: `git clone`, `docker compose up`, `alembic upgrade head`, `pytest`. Ma działać bez pytania Cię o cokolwiek.

> 🧠 **Dlaczego tak jest** — README pisany na końcu bywa zły, bo autor pamięta kontekst. Dlatego najprostszy test jakości README jest radykalny: **daj go do przeczytania komuś, kto nie zna projektu.** Jeśli ta osoba utknie, pytanie brzmi nie „czy ona jest nieogarnięta”, a „czego brakuje w README”. W tym projekcie czytelnik README jest częścią oceny (kryterium dokumentacji).

**Jak sprawdzić, że skończone**

- `rm -rf *.db venv && git clone ... && docker compose up -d db && alembic upgrade head && pytest` działa.
- README ma sekcję „Wymagania wstępne” z wersjami, nie z ogólnikami.
- Każdy ADR ma datę, kontekst, decyzję, konsekwencje i alternatywy — cztery elementy, nie esej.
- W repozytorium nie ma plików `*.db`, `venv/`, `.env`, `__pycache__/`.
- Historia git jest czytelna: komity mają zwięzłe, opisowe wiadomości w trybie rozkazującym („add partial index for I1” zamiast „poprawki”).

**Podpowiedź** — konteneryzacja jest ostatnia rozmyślnie. Jeśli zrobisz ją na początku, spędzisz dwa dni na walce z `Dockerfile`, zamiast na projekcie. Docker jest tu elementem dostarczenia, nie nauki.

**Moduł do powtórki:** `24_antywzorce_faq.md` (checklist przeglądu kodu), `22_fastapi_integracja.md`.

---

## 5. Kryteria oceny

Oceniasz projekt sam, ale według sztywnej tabeli. Punktacja jest maksymalna wtedy, gdy kryterium jest spełnione **demonstracyjnie** — czyli masz na to dowód (plik, linia w kodzie, wynik skryptu, zrzut planu zapytania).

| Obszar | Maks. pkt | Co daje maksimum |
|---|---|---|
| **Funkcjonalność** | 40 | Wszystkie niezmienniki I1–I8 zaimplementowane i przetestowane. Każda ścieżka biznesowa działa end-to-end. Test wyścigu raportuje dokładnie 1 sukces. Kara liczona poprawnie według I4. |
| **Jakość kodu** | 20 | Warstwy nieprzeciekające (brak importu `AsyncSession` w `services/`). Brak SQL-a w endpointach. Pełne typowanie, `mypy` czysty. Spójny styl 2.0 (`select()`, `Mapped`, `mapped_column`). Brak kodu martwego i zakomentowanego. |
| **Wydajność** | 10 | p95 < 300 ms przy 10 000 egzemplarzy (z pomiarem). Stała liczba zapytań niezależna od rozmiaru strony (z dowodem). Każdy indeks uzasadniony planem zapytania. |
| **Testy** | 15 | Pokrycie logiki biznesowej ≥ 80 %. Testy niezmienników istnieją i są nazwane numerami. Testy uruchamiają się na obu bazach. Brak testów zależnych od kolejności. Sprawdzana też liczba zapytań (regresja N+1). |
| **Dokumentacja** | 10 | README pozwala uruchomić projekt bez pomocy autora. Cztery ADR-y z kompletem elementów. Opisane decyzje o typach i indeksach. |
| **Migracje** | 5 | Migracje odtwarzają cały schemat od zera. Każda ma działający `downgrade`. Brak migracji pustych. `create_all` nigdzie w produkcyjnym kodzie. |
| **Razem** | **100** | |

### Jak interpretować wynik

| Punkty | Ocena jakościowa |
|---|---|
| 90–100 | projekt produkcyjny; możesz pokazywać go w rekrutacji jako „to zbudowałem sam” |
| 75–89 | solidny projekt portfolio; są jeszcze obszary do dopracowania |
| 60–74 | działa, ale architektura albo testy są niedokończone — wróć do Etapów 4 i 8 |
| 40–59 | funkcjonalnie niekompletny; przejdź całą sekcję 4 jeszcze raz, etapami |
| < 40 | wróć do modułów 14–22 i zbuduj mniejszą wersję projektu, potem rozszerzaj |

> ⚠️ **Pułapka — ocenianie „na wyczucie”.**
>
> Samoocena bez dowodów jest bez wartości, bo zawsze wypada dobrze. Każdy przyznany punkt musi mieć **artefakt**: linijkę w README, wynik `pytest`, zrzut `EXPLAIN ANALYZE`, wpis w ADR, liczbę z `bench_listing.py`. Zbierz je w pliku `docs/SCORECARD.md` — jedna tabela, dwie kolumny: „kryterium” i „dowód (ścieżka/komenda/wynik)”.

> 🧪 **Ćwiczenie —** Wypełnij `docs/SCORECARD.md` po Etapie 4 — czyli w połowie drogi. Zdziwisz się, ile kryteriów nie dostaje ani punktu, choć wydawało Ci się, że są „zrobione w głowie”.

---

## 6. Rozszerzenia dla chętnych

Rób je **dopiero po** uzyskaniu pełnych 100 punktów w tabeli oceny. Kolejność jest nieprzypadkowa — każde rozszerzenie zakłada działający rdzeń.

| # | Rozszerzenie | Co uczy | Sugerowany moduł do powtórki |
|---|---|---|---|
| 1 | **Kolejka zadań na wygaszanie rezerwacji** — worker (ARQ lub APScheduler) uruchamia `expire_stale()` co minutę, w osobnej sesji | zadania w tle, granice sesji poza webem, idempotencja | 21, 22 |
| 2 | **Wzorzec outbox** — zdarzenia biznesowe zapisywane w tabeli w tej samej transakcji, worker je później publikuje (e-mail, SMS) | spójność między bazą a światem zewnętrznym | 14, 21 |
| 3 | **Integracja z płatnościami** — kara jako fakturowalna pozycja z zewnętrznym identyfikatorem | idempotencja, retry, klucze idempotencji | 14 |
| 4 | **Wielojęzyczność katalogu** — nazwy kategorii w `JSONB` albo tabeli tłumaczeń | JSON, indeksy GIN, `with_variant` | 12 |
| 5 | **Panel administracyjny** — `SQLAdmin` albo prosty widok HTML z sesją per request | integracja z zewnętrzną warstwą prezentacji | 22 |
| 6 | **Wielodostępność (multi-tenant)** — jedna baza, izolacja schematami | `schema_translate_map`, granice danych | 02, 22 |
| 7 | **Raport analityczny z materializowanym widokiem** — dostępność jako widok odświeżany co minutę | widoki, `CreateView` w 2.1, kompromis świeżości i szybkości | 2.3.4, 17 |
| 8 | **Kod QR i skanowanie** — endpoint przyjmujący kod z czytnika, obsługa duplikatów skanów | idempotencja, klucze idempotencji | 14 |
| 9 | **Observability** — metryki Prometheus (liczba zapytań, p95, liczba aktywnych transakcji) | pomiar w produkcji, nie tylko benchmark | 17 |
| 10 | **Test obciążeniowy** — `locust` albo skrypt `asyncio` na 1000 wirtualnych użytkowników | zachowanie puli połączeń pod obciążeniem | 14, 17 |

> 🧠 **Dlaczego rozszerzenia numer 1 i 2 są na górze listy** — bo dotyczą dokładnie tego, czego projekt zaliczeniowy **nie** obejmuje, a co w prawdziwym systemie zawsze istnieje: coś musi dziać się **poza** cyklem request–response. Rezerwacja wygasająca sama z siebie to pierwszy raz, kiedy Twoja aplikacja działa bez użytkownika. A gdy działa bez użytkownika, pojawiają się pytania, których web nie zadaje: co jeśli worker padnie w połowie, co jeśli dwie instancje workera zrobią to samo, jak uniknąć wysłania dwóch e-maili. To najlepszy możliwy test zrozumienia granic transakcji.

> ⚠️ **Pułapka — rozszerzenia zamiast fundamentu.**
>
> Najczęstszy sposób zepsucia projektu: zaczynasz od „ciekawego” rozszerzenia (płatności! panel!), a kończysz z pięcioma niedokończonymi funkcjami i zerem testów. Rozszerzenia są nagrodą, nie odskocznią. Warunek wejścia jest twardy: **100/100 w tabeli oceny**.

---

## 7. Wskazówki do przeglądu własnej pracy

Przed uznaniem projektu za gotowy odpowiedz sobie na każde pytanie osobiście, patrząc w kod. Nie „chyba tak” — tylko „tak, w pliku X linia Y”. Jeśli nie umiesz wskazać linii, odpowiedź brzmi „nie”.

### Sesje i transakcje

1. Czy każdy request HTTP dostaje **własną** sesję, utworzoną w zależności FastAPI, i czy jest ona zamknięta po odpowiedzi? (`app/api/deps.py`)
2. Czy istnieje jakakolwiek ścieżka, w której sesja jest globalna albo modułowa? (szukaj `session = ...` poza funkcjami i klasami)
3. Czy wszystkie transakcje mają poprawnie wyznaczoną granicę — zaczynają się tam, gdzie zaczyna się przypadek użycia, i kończą przed wywołaniem czegokolwiek zewnętrznego?
4. Czy żadne repozytorium nie wywołuje `commit()`? (szukaj `.commit()` w `app/repositories/`)
5. Czy po każdym wyjątku w kontekście UoW wykonywany jest `rollback`? Sprawdź testem, nie wzrokiem.
6. Czy `expire_on_commit` jest ustawione świadomie i udokumentowane w ADR?

### Współbieżność

7. Co się stanie przy **100 równoległych rezerwacjach** tego samego egzemplarza? Czy odpowiedź to „przeczytałem skrypt i widziałem 1 sukces”, czy „wydaje mi się”?
8. Czy blokada `with_for_update()` jest ustawiona na właściwej tabeli i w tej samej transakcji, w której tworzysz wypożyczenie?
9. Czy test wyścigu działał na **PostgreSQL**, nie na SQLite?
10. Czy istnieje indeks częściowy blokujący dwa aktywne wypożyczenia tego samego egzemplarza? Pokaż `CREATE UNIQUE INDEX` w migracji.
11. Czy masz zaplanowaną obsługę `IntegrityError` i tłumaczenie jej na wyjątek domenowy?

### N+1 i wydajność

12. Czy endpoint listujący wypożyczenia pobiera pozycje w jednym zapytaniu (`selectinload`) czy w N? Pokaż liczbę zapytań z fixture.
13. Czy każda lista ma twarde `limit`?
14. Czy liczba zapytań jest **stała** dla rozmiaru strony 10, 50 i 100?
15. Czy każdy dodany indeks ma dowód w postaci `EXPLAIN ANALYZE` przed i po?

### Niezmienniki

16. Czy każdy z I1–I8 ma test, który **dowodzi** jego zachowania (asercja), a nie tylko „nie wywala się”?
17. Czy I8 (snapshot ceny) jest testowany na wypadek zmiany cennika **po** wypożyczeniu?
18. Czy I5 (append-only) jest zabezpieczony kodem (brak metod `update`/`delete`) albo triggerem?
19. Czy I4 (kara za każdy rozpoczęty dzień) jest testowany na granicy dnia, północy i zmiany strefy czasowej?

### Warstwy

20. Czy `services/` importuje cokolwiek z `sqlalchemy`? (Prawidłowa odpowiedź: tylko `AsyncSession` przekazywaną przez UoW — a jeśli nawet to nie, jeszcze lepiej.)
21. Czy `api/` zawiera jakikolwiek `select()`, `session.execute()` albo logikę biznesową?
22. Czy DTO nie zawierają encji ORM?
23. Czy istnieje cykl importów (A importuje B, B importuje A)? Sprawdź `import-linter` albo `ruff` z regułą cykli.

### Migracje i schemat

24. Czy `alembic downgrade base && alembic upgrade head` przechodzi?
25. Czy migracje odtwarzają cały schemat od zera, bez ręcznych kroków?
26. Czy w kodzie jest gdziekolwiek `create_all()`? (Odpowiedź: nie.)
27. Czy `naming_convention` jest ustawiona i widoczna w DDL ograniczeń?
28. Czy indeksy częściowe są w migracjach jawnie, z warunkiem `WHERE`?

### Odporność i higiena

29. Czy logowanie nie wycieka danych wrażliwych i czy `echo_sql` jest wyłączone na produkcji?
30. Czy projekt działa od zera na czystym klonie według własnego README?

> 💡 **Analogia — przegląd jak lista kontrolna przed wyjazdem w góry.**
>
> Przed wyjazdem nie pytasz „czy spakowałem czekan?” i nie odpowiadasz „chyba tak, coś tam wrzuciłem”. Wyjmujesz czekan i patrzysz na niego. Ta lista działa identycznie: dopiero gdy dotkniesz każdej pozycji, masz prawo wyjść. Projekt z siedmioma „chyba tak” i trzema „nie” nie jest gotowy — i lepiej, żebyś Ty to odkrył, niż osoba, która otworzy Twoje repozytorium.

> 🧪 **Ćwiczenie —** Wypisz z listy **te pytania, na które nie umiesz odpowiedzieć twierdząco**. Nie naprawiaj ich od razu — najpierw policz je. Jeśli jest ich więcej niż pięć, to znaczy, że projekt nie jest w Etapie 9, tylko w Etapie 5 albo 6. Wróć tam.

---

## 8. Gdzie szukać pomocy

Nie ma tu gotowego rozwiązania i to jest celowe. Ale nie ma też sytuacji, w której zostajesz bez wsparcia — w materiale kursu wszystkie potrzebne mechanizmy są omówione.

| Problem | Moduł | Konkretnie czego szukasz |
|---|---|---|
| Nie wiem, jak skonfigurować engine, połączenia, pulę | `02_srodowisko_i_engine.md` | sekcje o `create_engine`, `pool_pre_ping`, URL-ach |
| Nie wiem, jak zadeklarować tabelę, indeks, ograniczenie | `03_metadata_ddl.md` | `__table_args__`, `naming_convention`, indeksy |
| Nie wiem, jak napisać INSERT/UPDATE, jak działa upsert | `05_dml.md` | `on_conflict_do_update`, `returning` |
| Nie wiem, jak napisać skomplikowane zapytanie (raport, CTE) | `06_joiny_i_zaawansowane_sql.md` | `exists()`, CTE, funkcje okna |
| Nie wiem, jak napisać model deklaratywny, mixin, relację | `07`, `09` | `Mapped`, `mapped_column`, `relationship`, `Association Object` |
| Nie wiem, jak działa sesja, flush, commit, identity map | `08_sesja_cykl_zycia.md` | stany obiektu, `expire_on_commit`, autoflush |
| Nie wiem, jak pisać zapytania ORM w stylu 2.0 | `10_zapytania_orm.md` | `select()` + `scalars()`, ORM-enabled DML |
| Coś działa wolno, mam N+1 | `11_ladowanie_i_n_plus_1.md` | `selectinload`, licznik zapytań, `raiseload` |
| Nie wiem, jak obsłużyć JSON, UUID, Enum, czas | `12_typy_i_wlasne_typy.md` | `Uuid`, `Enum`, `DateTime(timezone=True)`, `TypeDecorator` |
| Nie wiem, jak zrobić audyt, automatyczne pola, `@validates` | `13_zdarzenia_i_hybrydy.md` | `before_flush`, `hybrid_property`, eventy |
| Mam wyścig, nie wiem, jak go rozwiązać | `14_transakcje_i_wspolbieznosc.md` | `with_for_update()`, poziomy izolacji, retry |
| Chcę async, dostaję `MissingGreenlet` | `15_asynchronicznosc.md` | `AsyncSession`, `AsyncAttrs`, `run_sync` |
| Nie wiem, jak napisać migrację, dowolnego `ALTER TABLE` | `16_alembic_migracje.md` | `op.add_column`, `batch_alter_table`, `render_as_batch` |
| Coś działa wolno, chcę zmierzyć i zoptymalizować | `17_wydajnosc.md` | `yield_per`, bulk, indeksy, `EXPLAIN` |
| Nie wiem, jak napisać test z bazą, jak izolować testy | `18_testowanie.md` | fixtures, `StaticPool`, izolacja transakcją |
| Mam kłopot z podziałem kodu na warstwy | `19_warstwy_i_data_mapper.md` | Data Mapper, DTO, granice warstw |
| Nie wiem, jak zorganizować repozytoria, paginację | `20_repository.md` | protokoły, keyset pagination, projekcje |
| Nie wiem, gdzie zacząć i skończyć transakcję | `21_unit_of_work.md` | `UnitOfWork`, sesja per request, granice |
| Nie wiem, jak to wszystko złożyć w FastAPI | `22_fastapi_integracja.md` | zależności, lifespan, schematy, obsługa błędów |
| Utknąłem z dziwnym komunikatem błędu | `24_antywzorce_faq.md` | tabela „komunikat → przyczyna → naprawa” |
| Potrzebuję szybkiej ściągi API | `A1_sciaga.md` | gotowe wzorce |
| Nie pamiętam, co znaczy termin | `A2_glosariusz.md` | EN → PL → definicja |
| Chcę porównać moje rozwiązanie ćwiczeń | `A3_cwiczenia_rozwiazania.md` | rozwiązania z modułów |

> 🧠 **Dlaczego tak jest** — najczęstszym sposobem porażki w projekcie zaliczeniowym nie jest brak wiedzy, tylko **brak cierpliwości do wrócenia do materiału**. Człowiek pamięta, że „gdzieś w module 14 było o blokadach”, ale zamiast wrócić, improwizuje — i pisze kod, który działa przypadkiem. Tabela powyżej istnieje po to, żeby powrót do materiału kosztował trzydzieści sekund, a nie pół godziny przeszukiwania. Korzystaj z niej bez wstydu.

> ⚠️ **Pułapka — „poszukam w internecie”.**
>
> W internecie znajdziesz mnóstwo przykładów SQLAlchemy w starym stylu (`session.query()`, `declarative_base()`, `Column(...)`). Skopiowanie takiego przykładu do projektu w stylu 2.0 **zadziała** (bo 2.0 nadal wspiera stare API), ale wprowadzi niespójność, którą krytykuje kryterium „jakość kodu”, i nauczy Cię złych nawyków. Jeśli szukasz wzorca — najpierw szukaj w kursie, potem w oficjalnej dokumentacji, a dopiero na końcu w wynikach wyszukiwania. I jeśli już kopiujesz, sprawdź, czy kod używa `select()`.

---

## Podsumowanie

1. **Projekt jest o niezmiennikach, nie o CRUD.** Osiem zdań z sekcji 1.4 jest ważniejszych niż wszystkie endpointy razem. Test niezmiennika chroni system; test endpointu chroni tylko kod.
2. **Najtrudniejszy niezmiennik to „jeden egzemplarz, jedno wypożyczenie”.** Rozwiązuje się go blokadą pesymistyczną w transakcji i indeksem częściowym w bazie. Dwie warstwy obrony, nie jedna.
3. **Dostępność jest wyliczana, nie przechowywana.** Historia zdarzeń jest tylko do dopisywania, a bieżący stan wynika z faktów. To eliminuje możliwość rozjazdu między „statusem” a rzeczywistością.
4. **Cena jest kopiowana, nie odwoływana.** Dane historyczne muszą być niezmienne — to ogólna zasada systemów rozliczeniowych, nie specyfika wypożyczalni.
5. **Architektura to warstwy, a warstwy to reguły importów.** `services/` nie zna SQLAlchemy, `api/` nie zna SQL-a. Te granice da się złamać w sekundę i to jest właśnie powód, żeby je nazwać i kontrolować.
6. **UoW tworzy jedną sesję dla wszystkich repozytoriów.** Dzięki temu commit obejmuje wszystkie zmiany przypadku użycia, a rollback cofa je w całości.
7. **Mierz, nie zgaduj.** p95, liczba zapytań, `EXPLAIN ANALYZE`, test wyścigu — każda decyzja „optymalizacyjna” bez liczby jest zgadywaniem.
8. **Test wyścigu na SQLite nie testuje niczego.** SQLite ignoruje `FOR UPDATE`, więc test przechodzi fałszywie. Baza docelowa jest częścią testu.
9. **Definicja „gotowe” jest binarna.** Dziewięć z dziesięciu punktów to nie „prawie gotowe” — to „nie gotowe”. Lista istnieje po to, żeby tego nie negocjować ze sobą.
10. **Rozszerzenia są nagrodą, nie planem.** Warunkiem wejścia jest pełna punktacja w tabeli oceny.

---

## Ćwiczenia

Trzy zadania. Pierwsze jest analityczne, drugie projektowe, trzecie weryfikacyjne. Żadne nie ma jednoznacznego rozwiązania w sensie kodu — i tak ma być, bo w projektach architektonicznych poprawna jest metoda, nie wynik.

### Zadanie 1 — Kryteria akceptacyjne dla Etapu 5

Napisz zestaw kryteriów akceptacyjnych (behavioralnych) dla przypadku użycia „wypożyczenie sprzętu przez pracownika”. Kryteria mają być zapisane w formacie **Given–When–Then** i musi ich być co najmniej osiem. Powinny pokryć: ścieżkę szczęśliwą, wszystkie niezmienniki, które ten przypadek dotyka, oraz co najmniej dwa scenariusze błędne. Każde kryterium musi być weryfikowalne automatycznym testem — czyli musi zawierać konkretne dane, nie ogólniki.

### Zadanie 2 — Plan testu współbieżności

Zaplanuj test współbieżności dla niezmiennika I1. Opisz: (a) ile korutyn uruchamiasz i dlaczego tyle, (b) jaką sesję dostaje każda korutyna i jak ją tworzysz, (c) co jest asercją i co ma być liczone, (d) jak odróżnić odrzucenie kontrolowane (wyjątek domenowy) od błędu technicznego (np. `asyncpg.TooManyConnectionsError`), (e) jak zagwarantować, że korutyny faktycznie startują **równolegle**, a nie jedna po drugiej, (f) jak sprawdzić, że test ma moc — czyli że **nie przechodzi** na wersji kodu bez blokady.

### Zadanie 3 — Audyt własnego modelu danych

Weź model danych z sekcji 2.1 i przeprowadź audyt pod kątem trzech pytań: (a) czy istnieje jakakolwiek kolumna, której wartość dałoby się wyliczyć z innych tabel (a więc jest duplikacją prawdy)? (b) czy istnieje jakakolwiek relacja, której brakuje do zaspokojenia wymagań funkcjonalnych F1–F7? (c) które kolumny wymagają `NOT NULL`, a które muszą być `NULL`-owalne, i dlaczego. Napisz raport w `docs/model-audit.md` — maksymalnie jedna strona.

### Rozwiązania

#### Rozwiązanie zadania 1 — wzorcowe kryteria

Dobre kryterium akceptacyjne ma trzy części i żadnych słów „poprawnie”, „prawidłowo”, „właściwie”, bo te słowa nic nie znaczą dla testera. Wzorcowy zestaw:

```text
K1 (ścieżka szczęśliwa, rezerwacja wykorzystana)
    Given: klient C ma potwierdzoną rezerwację R na egzemplarz E od D1 do D2
    When:  pracownik rejestruje wypożyczenie na E dla C na 3 doby
    Then:  powstaje Loan ze statusem ACTIVE, LoanItem z due_at = now + 3 dni
           i price_per_day_cents skopiowanym z bieżącego Tariff,
           rezerwacja R ma status USED,
           powstaje zdarzenie ItemStatusEvent typu LOANED

K2 (dwa wypożyczenia tego samego egzemplarza — I1)
    Given: egzemplarz E ma aktywne wypożyczenie (returned_at IS NULL)
    When:  pracownik próbuje wypożyczyć E ponownie
    Then:  operacja kończy się wyjątkiem ItemNotAvailable,
           liczba wierszy w loan_items dla E z returned_at IS NULL pozostaje 1

K3 (rezerwacja i wypożyczenie jednocześnie — I2)
    Given: egzemplarz E ma potwierdzoną rezerwację R na przyszły okres
    When:  pracownik próbuje wypożyczyć E na okres nakładający się z R
    Then:  operacja kończy się wyjątkiem ItemNotAvailable

K4 (snapshot ceny — I8)
    Given: wypożyczenie L na egzemplarz E z kategorią K z ceną 5000 gr/dobę
    When:  cennik kategorii K zmienia się na 7000 gr/dobę
    Then:  odczyt L zwraca price_per_day_cents = 5000 (wartość historyczna)

K5 (kara za przetrzymanie — I4)
    Given: wypożyczenie L z due_at = 2026-03-10T12:00:00Z
    When:  zwrot rejestrowany 2026-03-13T13:00:00Z
    Then:  powstaje Penalty z days_overdue = 3
           (dzień 11 — rozpoczęty, dzień 12 — rozpoczęty, dzień 13 — rozpoczęty)

K6 (zwrot na granicy dnia — I4, przypadek brzegowy)
    Given: wypożyczenie L z due_at = 2026-03-10T12:00:00Z
    When:  zwrot rejestrowany 2026-03-12T12:00:00Z (dokładnie 2 doby)
    Then:  powstaje Penalty z days_overdue = 2

K7 (podwójny zwrot — I7)
    Given: wypożyczenie L ze zwróconą pozycją na egzemplarz E
    When:  pracownik próbuje zarejestrować zwrot E po raz drugi
    Then:  operacja kończy się wyjątkiem ItemAlreadyReturned,
           liczba rekordów Penalty dla tej pozycji pozostaje 1

K8 (brak cennika w danym okresie — ścieżka błędna)
    Given: brak wpisu w Tariff dla kategorii K obejmującego dzisiejszą datę
    When:  pracownik rejestruje wypożyczenie egzemplarza kategorii K
    Then:  operacja kończy się wyjątkiem TariffNotFound,
           w bazie nie powstaje żaden Loan ani LoanItem
```

Cztery cechy, które odróżniają te kryteria od słabych: konkretne dane (daty, kwoty, kody), konkretny oczekiwany rezultat (status, liczba wierszy, wartość pola), pokrycie ścieżek błędnych, i **pokrycie przypadków brzegowych** (K6 — dokładnie dwie doby).

Alternatywa: gdybyś chciał zapisać to zwięźlej, możesz użyć tabeli „krok | działanie | rezultat” zamiast Given–When–Then. Kompromis: tabela jest szybsza do skanowania w code review, ale GWT wymusza precyzję kontekstu, którą łatwo zgubić w tabeli.

Minipułapka: kryterium „Then: operacja kończy się sukcesem” jest bezwartościowe, bo „sukces” nie jest zdefiniowany. Test na takie kryterium zawsze przejdzie — bo nie sprawdza niczego poza tym, że nie rzucił wyjątku.

#### Rozwiązanie zadania 2 — plan testu współbieżności

(a) **Liczba korutyn: 100.** Więcej nie wnosi nowej informacji (przy 1000 prawdopodobieństwo wyścigu nie rośnie istotnie przy 10 połączeniach w puli, a czas testu rośnie liniowo). Mniej niż 20 nie gwarantuje, że jakiekolwiek dwie korutyny faktycznie się spotkają. Sto to wartość, przy której wyścig ujawnia się praktycznie zawsze, gdy kod jest błędny.

(b) **Sesja per korutyna.** Każda korutyna tworzy własną `AsyncSession` z tej samej fabryki — inaczej nie ma współbieżności, tylko wspólny stan. Fabryka musi być skonfigurowana z `pool_size >= 100` i `max_overflow = 0`, żeby pula nie zamaskowała problemu (gdyby pula miała 5 połączeń, zapytania serializowałyby się same z siebie i test nie ujawniłby wyścigu — co jest poważnym błędem metodologicznym).

(c) **Asercja i liczenie.** Zbierasz wyniki do trzech liczników: `successes` (zwróciły rezerwację), `domain_rejections` (`ItemNotAvailable` — oczekiwane), `technical_errors` (wszystko inne). Asercja: `assert successes == 1`, `assert technical_errors == 0`, `assert successes + domain_rejections == 100`. Trzecia asercja jest ważna: gwarantuje, że żadna korutyna nie „zniknęła”.

(d) **Rozróżnienie odrzucenia kontrolowanego od błędu.** Tylko wyjątek z warstwy domenowej (`ItemNotAvailable`) wpadający do `domain_rejections` jest sukcesem testu. Wszystko inne — `asyncpg.TooManyConnectionsError`, `TimeoutError`, `IntegrityError` przeciekający jako `500` — trafia do `technical_errors` i **obla test**. To rozróżnienie jest krytyczne: bez niego test „przechodzi” także wtedy, gdy aplikacja po prostu nie wyrabia i odrzuca 99 żądań z powodu wyczerpania puli.

(e) **Gwarancja równoległości.** Nie wystarczy `asyncio.gather` — korutyny startują razem, ale mogą wykonać się sekwencyjnie, jeśli nie ma `await` w kluczowym punkcie. Rozwiązanie: wstrzyknij punkt synchronizacji tuż przed operacją krytyczną, np. `await asyncio.sleep(0)` albo `asyncio.Barrier` (Python 3.11+) — wszystkie korutyny startują dopiero, gdy wszystkie dobiegły do punktu startu. Bez tego test mierzy kolejkę, nie wyścig.

```python
# scripts/race_reservation_sketch.py (szkic)
import asyncio

async def attempt(barrier: asyncio.Barrier, factory, item_id: int) -> str:
    async with factory() as session:
        await barrier.wait()          # wszyscy startują w tej samej chwili
        try:
            await reservation_service.create(session, customer_id=1, item_id=item_id)
            return "success"
        except ItemNotAvailable:
            return "domain_rejection"

async def main() -> None:
    barrier = asyncio.Barrier(100)
    results = await asyncio.gather(*(attempt(barrier, factory, 42) for _ in range(100)))
    successes = results.count("success")
    assert successes == 1, f"oczekiwano 1 sukcesu, było {successes}"
```

(f) **Test ma moc tylko wtedy, gdy umie się wywalić.** Sposób sprawdzenia: zakomentuj `with_for_update()` w serwisie (razem z indeksem częściowym — usuń go tymczasowo z bazy), uruchom test, **musi** zobaczyć więcej niż 1 sukces. Jeśli nadal widzi 1, test nie mierzy tego, co myślisz. To najważniejszy krok całego zadania — test, który nie umie paść, nie jest testem. Zapisz ten eksperyment w `docs/perf.md` jako dowód.

Alternatywa: zamiast zakomentowania blokady możesz usunąć indeks częściowy z bazy ręcznym `DROP INDEX`. Kompromis: usunięcie tylko blokady pokazuje, że test wykrywa brak blokady; usunięcie tylko indeksu — że wykrywa brak indeksu. Oba eksperymenty są wartościowe i pokazują, która warstwa obrony działa.

Minipułapka: `asyncio.Barrier` w Pythonie 3.11 przy 100 uczestnikach jest w porządku, ale na `asyncio.Semaphore` z limitem mniejszym niż liczba korutyn „zagłodziłbyś” test — część korutyn nigdy nie dotarłaby do baru. Uważaj też na limit plików deskryptorów w systemie, jeśli testujesz pulę większą niż domyślny `ulimit -n`.

#### Rozwiązanie zadania 3 — wzorcowy audyt

**Pytanie (a): czy istnieje kolumna duplikująca prawdę?**

Przeglądaj model po modelu i zadaj każdej kolumnie pytanie: „czy da się ją wyliczyć z innych tabel?”. Wynik audytu modelu z sekcji 2.1:

| Kolumna | Da się wyliczyć? | Werdykt |
|---|---|---|
| `Loan.status` | tak — „wszystkie pozycje mają `returned_at`” | **świadomie zatrzymana denormalizacja** — dla wydajności listingu; wymaga zdarzenia synchronizującego i testu, który wykryje rozjazd |
| `Reservation.status` | częściowo — `PENDING`/`CONFIRMED` wynikają z `expires_at` | świadomie zatrzymana; `EXPIRED` nie da się wyliczyć, bo potrzebujemy wiedzieć, że **wygasła**, a nie tylko że minął czas |
| `EquipmentItem` (bez statusu) | — | brak kolumny statusu; prawdopodobnie poprawnie |
| `Penalty.amount_cents` | tak — `days_overdue * stavka` | **świadomie zatrzymana** — to fakt rozliczeniowy; przeliczenie go po zmianie stawki zmieniłoby historię (analogia z I8) |
| `LoanItem.price_per_day_cents` | tak — z `Tariff` | **świadomie zatrzymana** — snapshot rozliczeniowy (I8) |

Wniosek audytu: w modelu są cztery kolumny, które „technicznie” są duplikacją, i wszystkie cztery są celowe. Ale zauważ subtelność: dopóki nie zadasz tego pytania, nie wiesz, które denormalizacje są **uzasadnione**, a które **przypadkowe**. Audyt kończy się wpisem do ADR dla każdej zatrzymanej denormalizacji z uzasadnieniem i wymogiem: „jeśli źródło prawdy się zmienia, ta kolumna musi zostać zaktualizowana w tej samej transakcji”.

**Pytanie (b): czy brakuje relacji?**

Przejdź wymagania F1–F7 i sprawdź, czy każdy da się zrealizować w tym modelu:

- F1 (katalog z dostępnością) → `Category` → `EquipmentItem` → relacje `LoanItem`, `Reservation`. OK.
- F2 (rezerwacja) → `Reservation` z `expires_at` i `status`. OK.
- F3 (wypożyczenie wielopozycyjne) → `Loan` + `LoanItem`. OK.
- F4 (zwrot i kara) → `LoanItem.returned_at` + `Penalty`. OK.
- F5 (historia klienta i egzemplarza) → `Customer` → `Loan`, `EquipmentItem` → `ItemStatusEvent`. OK.
- F6 (wycofanie) → zdarzenie `RETIRED` w `ItemStatusEvent`. **Tu jest problem:** „wycofany egzemplarz nie pojawia się w katalogu”, a katalog filtruje po dostępności, która jest wyliczana z `Loan`/`Reservation`. Egzemplarz wycofany nie ma żadnego wypożyczenia ani rezerwacji, więc **będzie wyglądał na dostępny**. Brakuje reguły: filtr katalogu musi uwzględniać też brak zdarzenia `RETIRED`. To jest **realna luka w modelu** ujawniona audytem.
- F7 (cennik) → `Tariff`. OK.

Wniosek: brakuje jawnego sposobu na odróżnienie egzemplarza wycofanego od dostępnego. Dwie drogi naprawy: (1) dodać filtr `NOT EXISTS` na zdarzeniu `RETIRED` w zapytaniu katalogu, (2) utrzymywać denormalizowaną kolumnę `retired_at` na `EquipmentItem`. Droga pierwsza jest w duchu „nie przechowujemy stanu”; droga druga jest szybsza w odczycie. Decyzja do ADR.

**Pytanie (c): które kolumny są `NOT NULL`, które `NULL`-owalne?**

Tabela wynikowa dla najważniejszych kolumn:

| Kolumna | Nullable | Dlaczego |
|---|---|---|
| `EquipmentItem.code` | `NOT NULL` | identyfikator zewnętrzny, nigdy nie pusty |
| `EquipmentItem.retired_at` (jeśli dodana) | `NULL` | puste = nie wycofany |
| `LoanItem.returned_at` | `NULL` | `NULL` = nie zwrócony; to znaczenie niesie logikę I1 i I7 |
| `LoanItem.condition_on_return` | `NULL` | puste do chwili zwrotu |
| `Reservation.expires_at` | `NOT NULL` | zawsze istnieje, także dla `CONFIRMED` |
| `Reservation.cancelled_at` | `NULL` | puste = nie anulowana |
| `Customer.deleted_at` | `NULL` | puste = aktywny |
| `Penalty.paid_at` | `NULL` | puste = kara nieopłacona |
| `Penalty.days_overdue` | `NOT NULL` | zawsze znane w chwili naliczenia |
| `Loan.closed_at` | `NULL` | puste = wypożyczenie trwa |

Reguła ogólna: **`NULL` reprezentuje brak zdarzenia w przyszłości albo brak relacji opcjonalnej**. Jeśli kolumna jest `NULL`-owalna i nie umiesz w jednym zdaniu powiedzieć, co znaczy `NULL`, to znaczy że brakuje Ci wiedzy o modelu — nie że decyzja jest dowolna.

Minipułapka: w SQLAlchemy `Mapped[str | None]` **automatycznie** ustawia `nullable=True`, a `Mapped[str]` — `nullable=False`. Ale `Mapped[str | None]` z `mapped_column(default="")` jest sprzeczne semantycznie: domyślna wartość sugeruje, że puste nie jest możliwe. Taki model będzie działał, ale komunikuje coś przeciwnego, niż robi. To dokładnie ta klasa niespójności, którą wyłapuje audyt.

---

## Najczęstsze błędy i jak je czytać

| Komunikat błędu | Przyczyna | Naprawa |
|---|---|---|
| `sqlalchemy.exc.MissingGreenlet: greenlet_spawn has not been called` | próba leniwego ładowania relacji w kontekście async bez `await` | dodaj `selectinload`/`joinedload` do zapytania albo użyj `await obj.awaitable_attrs.relacja` (moduł 15) |
| `sqlalchemy.orm.exc.DetachedInstanceError: Instance <X> is not bound to a Session` | obiekt poza aktywną sesją, próba dostępu do niezaładowanej relacji | zwróć DTO, a nie encję; ładuj relacje eagerly; nie przekazuj encji do zadań tła |
| `sqlalchemy.exc.PendingRollbackError: This Session's transaction has been rolled back` | po wyjątku nie zrobiono `rollback`, kolejna operacja na tej samej sesji | obsłuż wyjątek w UoW i wykonaj rollback; nie kontynuuj pracy na tej samej sesji |
| `sqlalchemy.exc.IntegrityError: duplicate key value violates unique constraint "uq_loan_items_active_item"` | próba utworzenia drugiego aktywnego wypożyczenia tego samego egzemplarza (I1 złamało się?) — dobra wiadomość: baza zadziałała | przechwyć `IntegrityError`, zamień na `ItemNotAvailable`, pokaż klientowi `409`; sprawdź, dlaczego aplikacja nie złapała tego wcześniej |
| `sqlalchemy.exc.OperationalError: server closed the connection unexpectedly` | połączenie w puli „umarło” (restart bazy, timeout po stronie serwera) | ustaw `pool_pre_ping=True` i `pool_recycle` (moduł 02) |
| `sqlalchemy.exc.StatementError: SQLite DateTime type only accepts Python datetime objects` | przekazano string zamiast `datetime` — albo strefa czasowa nie ustawiona | parsuj dane wejściowe do `datetime` w schemacie Pydantic; używaj `DateTime(timezone=True)` |
| `asyncpg.TooManyConnectionsError` / `TimeoutError: connection pool exhausted` | pula mniejsza niż równoległość; wyciek połączeń | zwiększ `pool_size`/`max_overflow`; sprawdź, czy każda sesja jest zamknięta; użyj `async with` konsekwentnie |
| `alembic.util.exc.CommandError: Multiple head revisions are present` | dwie gałęzie migracji w zespole | `alembic merge heads -m "merge"` (moduł 16) |
| `alembic` nie generuje migracji dla nowej kolumny | model nie zaimportowany do `env.py` — tabela nie zarejestrowana w `Base.metadata` | dodaj import pakietu `app.models` w `env.py` lub w `base.py` |
| `TypeError: Object of type UUID is not JSON serializable` | zwracasz encję zamiast DTO, serializer nie zna `UUID` | użyj schematu Pydantic z typem `UUID`; nie zwracaj encji ORM |
| `RuntimeError: Task got Future attached to a different loop` | jedna sesja/engine używane w różnych pętlach zdarzeń lub sesja współdzielona między korutynami | sesja per korutyna; engine tworzony raz na pętlę; w testach `pytest-asyncio` z odpowiednim scope |
| `sqlalchemy.exc.InvalidRequestError: Can't operate on closed transaction inside context manager` | podwójne zamknięcie albo `commit` wewnątrz `async with session.begin()` | nie wywołuj `commit` ręcznie wewnątrz kontekstu `begin()`; zobacz moduły 08, 21 |

> ⚠️ **Pułapka — czytanie tylko pierwszej linii komunikatu.**
>
> Komunikaty SQLAlchemy są długie i mają „łańcuch przyczyn” (chained traceback). Ostatnia linia `During handling of the above exception, another exception occurred:` poprzedza **właściwą** przyczynę. Czytaj od końca, nie od początku. W praktyce: `PendingRollbackError` jest symptomem, a prawdziwa przyczyna (np. `IntegrityError`) jest wyżej w tracebacku.

---

## Słowniczek modułu

| Termin (EN) | Polski | Wyjaśnienie |
|---|---|---|
| **invariant** | niezmiennik | zdanie, które musi być prawdziwe w każdej chwili życia systemu, niezależnie od liczby równoległych operacji |
| **concurrency race** | wyścig | sytuacja, w której wynik zależy od kolejności wykonania równoległych operacji, a nie od ich treści |
| **pessimistic locking** | blokada pesymistyczna | rezerwacja wiersza na czas transakcji (`SELECT ... FOR UPDATE`), żeby nikt inny go nie zmienił |
| **optimistic locking** | blokada optymistyczna | sprawdzenie wersji wiersza przy zapisie; konflikt wykrywany na końcu, nie blokuje z góry |
| **snapshot** | kopia stanu | skopiowanie wartości (np. ceny) w momencie zdarzenia, żeby późniejsze zmiany nie zmieniały historii |
| **association object** | obiekt asocjacyjny | klasa reprezentująca tabelę pośrednią, gdy ma ona własne kolumny poza kluczami obcymi |
| **append-only log** | dziennik tylko do dopisywania | tabela, w której rejestry się tylko dodaje; nigdy nie modyfikuje ani nie usuwa |
| **denormalization** | denormalizacja | świadome przechowywanie wartości, którą dałoby się wyliczyć — dla szybkości, kosztem ryzyka rozjazdu |
| **source of truth** | źródło prawdy | miejsce, w którym dana „naprawdę” żyje; wszystkie inne miejsca to kopie |
| **partial index** | indeks częściowy | indeks obejmujący tylko wiersze spełniające warunek; używany tu do egzekwowania niezmienników |
| **soft delete** | usuwanie miękkie | oznaczenie rekordu datą usunięcia zamiast fizycznego kasowania |
| **surrogate key** | klucz sztuczny | identyfikator wygenerowany (auto-increment, UUID), nie wynikający z danych biznesowych |
| **lookup table** | tabela słownikowa | tabela z wartościami edytowalnymi przez użytkownika, bez zmian w kodzie |
| **idempotency** | idempotentność | właściwość operacji, która wykonana dwa razy daje ten sam efekt co wykonana raz |
| **p95** | — | percentyl 95; czas, poniżej którego mieści się 95 % żądań; lepsza miara niż średnia |
| **N+1** | — | wzorzec wadliwego dostępu: jedno zapytanie o listę i N zapytań o powiązane dane |
| **eager loading** | ładowanie zachłanne | pobranie powiązanych danych z góry, w tym samym zapytaniu lub w drugim zbiorczym |
| **lazy loading** | ładowanie leniwe | pobranie powiązanych danych dopiero w chwili dostępu; źródło problemu N+1 |
| **handover / checkout** | wydanie | moment, w którym fizyczny egzemplarz opuszcza magazyn i powstaje zobowiązanie |
| **unit of work** | jednostka pracy | obiekt grupujący zmiany wielu repozytoriów w jedną transakcję |
| **repository** | repozytorium | klasa tłumacząca pytania biznesowe na zapytania bazodanowe |
| **ADR** (Architecture Decision Record) | zapis decyzji architektonicznej | krótki dokument opisujący jedną decyzję: kontekst, wybór, konsekwencje, alternatywy |
| **non-functional requirement** | wymaganie niefunkcjonalne | wymaganie dotyczące jakości (wydajność, niezawodność), nie funkcji |
| **acceptance criteria** | kryteria akceptacyjne | konkretne, testowalne warunki, których spełnienie uznaje funkcję za gotową |

---

## Dalsze czytanie

Oficjalna dokumentacja SQLAlchemy 2.0 (sekcje przydatne w tym projekcie):

- [ORM Quick Start](https://docs.sqlalchemy.org/en/20/orm/quickstart.html) — najkrótsze przypomnienie składni 2.0
- [Session Basics](https://docs.sqlalchemy.org/en/20/orm/session_basics.html) — cykl życia sesji, flush, commit, stany obiektu
- [Using the Session / Transactions](https://docs.sqlalchemy.org/en/20/orm/session_transaction.html) — transakcje, savepointy, autobegin
- [ORM Querying Guide](https://docs.sqlalchemy.org/en/20/orm/queryguide/index.html) — `select()` w ORM, eager loading, `with_for_update`
- [Relationship Loading Techniques](https://docs.sqlalchemy.org/en/20/orm/queryguide/relationships.html) — strategie ładowania relacji
- [Relationship Configuration](https://docs.sqlalchemy.org/en/20/orm/relationships.html) — `relationship()`, kaskady, Association Object
- [Background: Design & Concurrency](https://docs.sqlalchemy.org/en/20/orm/session_transaction.html#session-transaction-isolation) — izolacja i współbieżność
- [Asyncio Extension](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html) — `AsyncEngine`, `AsyncSession`, `run_sync`, `AsyncAttrs`
- [Type Decorators](https://docs.sqlalchemy.org/en/20/core/custom_types.html#sqlalchemy.types.TypeDecorator) — własne typy
- [Declarative Mapping](https://docs.sqlalchemy.org/en/20/orm/declarative_tables.html) — `Mapped`, `mapped_column`, `type_annotation_map`

Dokumentacja Alembic:

- [Alembic Tutorial](https://alembic.sqlalchemy.org/en/latest/tutorial.html)
- [Autogenerate](https://alembic.sqlalchemy.org/en/latest/autogenerate.html) — co wykrywa, a czego nie
- [Working in the Offline Mode / Batch](https://alembic.sqlalchemy.org/en/latest/batch.html) — migracje dla SQLite

Dokumentacja pokrewna:

- [FastAPI — SQL (Relational) Databases](https://fastapi.tiangolo.com/tutorial/sql-databases/)
- [Pydantic v2 — Model Config](https://docs.pydantic.dev/latest/concepts/models/)
- [PostgreSQL — Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html) — `FOR UPDATE`, poziomy izolacji
- [pytest-asyncio](https://pytest-asyncio.readthedocs.io/) — fixtures async i scope

---

## Co dalej

Zbudowałeś — lub właśnie budujesz — kompletny system. Jeśli w trakcie pracy natrafiłeś na problem, którego nie umiałeś rozpoznać po komunikacie błędu, albo na sytuację, w której „wszystko wygląda dobrze, a nie działa”, następny moduł jest dla Ciebie napisany. Zawiera katalog antywzorców z objawem, przyczyną i naprawą, listę pytań do code review oraz FAQ z dwudziestoma pytaniami, które najczęściej zadają osoby po kursie.

Przejdź do [`24_antywzorce_faq.md`](24_antywzorce_faq.md) — modułu ratunkowego, do którego będziesz wracać jeszcze długo po ukończeniu kursu.

<!-- koniec modułu 23 -->