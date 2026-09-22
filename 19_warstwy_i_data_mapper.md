# Moduł 19 — Warstwy, modele domenowe i Data Mapper

Do tego momentu nauczyłeś się *jak* używać SQLAlchemy: jak pisać modele, zapytania, relacje, transakcje, jak testować i jak mierzyć wydajność. Ten moduł odpowiada na inne pytanie: **gdzie w aplikacji ma mieszkać SQLAlchemy?** Dowiesz się, dlaczego kod, w którym endpoint HTTP bezpośrednio skleja SQL, wygląda niewinnie przez pierwszy tydzień i staje się koszmarem w trzecim miesiącu. Zobaczysz, że model bazy danych i model reguł biznesowych to **dwie różne rzeczy**, które tylko czasem warto utożsamić. Poznasz wzorzec **Data Mapper** (odwzorowanie danych) i zrozumiesz, dlaczego SQLAlchemy jest jego pełnoprawną implementacją — a nie „ORM-em, który ma `save()` w obiekcie”. Na koniec zbudujesz te same trzy przypadki użycia na dwa sposoby: prosty (serwisy + encje ORM) i rozbudowany (domena + repozytoria + jednostka pracy), żeby **samodzielnie ocenić koszt i zysk** każdego z nich.

| | |
|---|---|
| **Poziom** | 🔴 architektoniczny |
| **Czas** | ~180 minut |
| **Wymagania wstępne** | [`07_modele_deklaratywne.md`](07_modele_deklaratywne.md), [`08_sesja_cykl_zycia.md`](08_sesja_cykl_zycia.md), [`09_relacje.md`](09_relacje.md), [`11_ladowanie_i_n_plus_1.md`](11_ladowanie_i_n_plus_1.md), [`18_testowanie.md`](18_testowanie.md) |
| **Czego dotyczy plik** | Podział aplikacji na warstwy, relacja między modelem ORM a modelem domenowym, wzorzec Data Mapper (w tym mapowanie imperatywne), DTO i modele odczytu, granice transakcji, reguła zależności, wybór wariantu architektury |
| **Czego NIE dotyczy** | Konkretnej implementacji repozytoriów (moduł [`20_repository.md`](20_repository.md)), implementacji jednostki pracy (moduł [`21_unit_of_work.md`](21_unit_of_work.md)), integracji z FastAPI (moduł [`22_fastapi_integracja.md`](22_fastapi_integracja.md)) |

> **Uwaga o wersjach.** Bazą kursu jest SQLAlchemy 2.0.x. Wszystko, co pokazuję w tym module, działa zarówno w 2.0, jak i w 2.1. Różnice oznaczam ramkami 🆕.

## Spis treści

1. [Zaczynamy od problemu: kod, w którym wszystko jest wszędzie](#1-zaczynamy-od-problemu-kod-w-którym-wszystko-jest-wszędzie)
2. [Cztery warstwy i reguła zależności](#2-cztery-warstwy-i-reguła-zależności)
3. [Model ORM a model domenowy](#3-model-orm-a-model-domenowy)
4. [Data Mapper kontra Active Record](#4-data-mapper-kontra-active-record)
5. [Mapowanie imperatywne — dowód, że mapowanie to konfiguracja](#5-mapowanie-imperatywne--dowód-że-mapowanie-to-konfiguracja)
6. [Jak modelować domenę: encje, value objects, agregaty](#6-jak-modelować-domenę-encje-value-objects-agregaty)
7. [DTO i modele odczytu](#7-dto-i-modele-odczytu)
8. [Granice transakcji należą do warstwy aplikacji](#8-granice-transakcji-należą-do-warstwy-aplikacji)
9. [Trzy warianty architektury do wyboru](#9-trzy-warianty-architektury-do-wyboru)
10. [Przykład obowiązkowy: te same trzy przypadki użycia w wariancie A i B](#10-przykład-obowiązkowy-te-same-trzy-przypadki-użycia-w-wariancie-a-i-b)
11. [Podsumowanie](#podsumowanie)
12. [Ćwiczenia](#ćwiczenia)
13. [Najczęstsze błędy i jak je czytać](#najczęstsze-błędy-i-jak-je-czytać)
14. [Słowniczek modułu](#słowniczek-modułu)
15. [Dalsze czytanie](#dalsze-czytanie)
16. [Co dalej](#co-dalej)

---

## 1. Zaczynamy od problemu: kod, w którym wszystko jest wszędzie

Zacznijmy od kodu, który prawdopodobnie napisałbyś po module 10, gdyby nikt nie zadał pytania „a gdzie to ma mieszkać?”. To endpoint HTTP z modułu [`22_fastapi_integracja.md`](22_fastapi_integracja.md) — w wersji, do której prowadzi pierwsza, naiwna ścieżka rozwoju.

```python
# examples/19_before.py
# WERSJA "PRZED" — wszystko w jednym miejscu. Kod, który działa.
# Nie kopiuj tego do produkcji; to materiał do analizy.

from datetime import date, timedelta

from fastapi import Depends, FastAPI, HTTPException
from sqlalchemy import func, select
from sqlalchemy.exc import IntegrityError
from sqlalchemy.orm import Session

from app.database import get_session
from app.models import Book, Loan, Member

app = FastAPI()

MAX_ACTIVE_LOANS = 5
LOAN_PERIOD_DAYS = 30


@app.post("/loans")
def create_loan(payload: dict, session: Session = Depends(get_session)) -> dict:
    member = session.get(Member, payload["member_id"])
    if member is None:
        raise HTTPException(status_code=404, detail="nie ma takiego członka")
    if not member.is_active:
        raise HTTPException(status_code=409, detail="członek nieaktywny")

    active_loans = session.scalar(
        select(func.count())
        .select_from(Loan)
        .where(Loan.member_id == member.id, Loan.returned_on.is_(None))
    )
    if active_loans >= MAX_ACTIVE_LOANS:
        raise HTTPException(status_code=409, detail="limit wypożyczeń przekroczony")

    already_out = session.scalar(
        select(func.count())
        .select_from(Loan)
        .where(Loan.book_id == payload["book_id"], Loan.returned_on.is_(None))
    )
    if already_out:
        raise HTTPException(status_code=409, detail="książka jest już wypożyczona")

    today = date.today()
    loan = Loan(
        member_id=member.id,
        book_id=payload["book_id"],
        borrowed_on=today,
        due_on=today + timedelta(days=LOAN_PERIOD_DAYS),
    )
    session.add(loan)
    try:
        session.commit()
    except IntegrityError:
        session.rollback()
        raise HTTPException(status_code=409, detail="konflikt") from None

    return {
        "id": loan.id,
        "member": member.name,
        "due_on": loan.due_on.isoformat(),
        "status": "ok",
    }
```

Ten kod **działa**. Jest też całkiem czytelny, jeśli patrzysz tylko na niego. Problem nie leży w pojedynczym endpoincie — problem leży w tym, co się dzieje, gdy takich endpointów jest czterdzieści.

> 💡 **Analogia** — wyobraź sobie restaurację, w której kelner bierze zamówienie, idzie do kuchni, sam gotuje, sam zmywa naczynia, sam przyjmuje płatność i sam robi zakupy. Przy trzech stolikach to działa. Przy trzydziestu — nie wiadomo, kto za co odpowiada, nie da się zatrudnić drugiego kucharza bez przeszkolenia go na kelnera, a każde nowe danie wymaga zmiany w każdym kelnerze. Warstwy to po prostu podział ról: kelner (API), kuchnia (domena), magazyn (baza).

### Co konkretnie jest nie tak

Wypiszmy problemy, bo każdy z nich będzie potem adresowany przez konkretną warstwę:

**1. Reguła biznesowa ukryta w handlerze HTTP.** „Limit 5 aktywnych wypożyczeń” to zasada, którą firma może zmienić niezależnie od tego, czy wystawiamy REST, CLI, czy kolejkę zadań. Tutaj jest zapisana w funkcji, która wie, czym jest `HTTPException`. Jeśli ta sama reguła jest potrzebna w importerze danych z pliku CSV, musisz ją skopiować — a skopiowana reguła to reguła, która za pół roku będzie miała dwie różne wersje.

**2. Baza danych jest zaszyta w interfejsie.** Gdy przyjdzie wymaganie „odczyt raportu ma nie obciążać bazy, dane są w hurtowni”, musisz przepisać endpoint, a nie dodać drugiej implementacji.

**3. Testowanie wymaga bazy.** Nie da się sprawdzić reguły „limit 5” bez uruchomienia silnika, sesji i tabel. To nie jest katastrofa (moduł [`18_testowanie.md`](18_testowanie.md) pokazał, jak to robić szybko), ale testy przez bazę są wolniejsze i trudniejsze do zrozumienia niż testy czystej funkcji.

**4. Zwracamy surowe dane bez kontraktu.** Słownik budowany ręcznie w handlerze to kontrakt API, którego nikt nie weryfikuje. Albo (w innych wersjach tego kodu) zwracamy encję ORM i nagle klient zobaczy pole `internal_note`, a serializacja wywoła leniwe ładowanie relacji i wybuchnie `DetachedInstanceError`.

**5. Nie da się użyć dwóch operacji w jednej transakcji.** „Wypożycz książkę i zapisz wpis w rejestrze” — gdzie to zrobić? W handlerze? Wtedy każdy nowy przypadek użycia wymaga nowego handlera.

**6. Wyjątki techniczne przeciekają.** `IntegrityError` jest tłumaczony ręcznie, w każdym miejscu osobno. Zapomnisz raz — i użytkownik dostanie 500 z komunikatem bazy.

> ⚠️ **Pułapka** — bardzo częsty kontrargument brzmi: „ale my mamy mały projekt, po co nam warstwy?”. Uczciwa odpowiedź: w małym projekcie warstwy można mieć w wersji minimalnej (osobny moduł z funkcjami, protokół zamiast klasy abstrakcyjnej). Zła odpowiedź brzmi: „nie potrzebujemy żadnego podziału”. Sam podział **nie jest opcjonalny**; opcjonalny jest jego *koszt*.

### „Po” — jak zmienia się ten sam endpoint

Bez wchodzenia jeszcze w szczegóły, zobacz, do czego zmierzamy:

```python
# examples/19_after.py
# WERSJA "PO" — endpoint nie wie, jak działa baza ani jak działa biznes.

from fastapi import Depends

from app.application.borrow_book import borrow_book
from app.infrastructure.uow import SqlAlchemyUnitOfWork, get_uow


@app.post("/loans", status_code=201)
def create_loan(
    payload: LoanRequest,
    uow: SqlAlchemyUnitOfWork = Depends(get_uow),
) -> LoanResponse:
    loan = borrow_book(
        uow,
        member_id=payload.member_id,
        book_id=payload.book_id,
    )
    return LoanResponse.model_validate(loan)
```

Endpoint ma teraz trzy linie logiki i nie zawiera ani jednego znaku SQL. Reguły „limit 5” i „książka wolna” żyją w warstwie aplikacji, tłumaczenie wyjątków domenowych na kody HTTP dzieje się w jednym miejscu (handler wyjątków), a baza jest wymienialna.

Co ciekawe, ten endpoint jest **krótszy niż wersja „przed”**, a jednocześnie testowalny bez HTTP i bez bazy (podstawiasz fałszywą jednostkę pracy) i testowalny z bazą (podstawiasz prawdziwą). To nie jest przypadek — dobre warstwy zwykle *skracają* kod, bo przestają mieszać poziomy abstrakcji.

---

## 2. Cztery warstwy i reguła zależności

### 2.1. Warstwy

Kiedy mówimy „warstwy”, mówimy o **poziomach odpowiedzialności**, nie o katalogach na dysku. Katalogi są tylko sposobem zapisania tej decyzji w plikach.

```text
┌─────────────────────────────────────────────────────────────┐
│  PREZENTACJA (presentation)                                 │
│  HTTP/REST, CLI, panel WWW, kolejarz zadań                  │
│  Odpowiada za: transport, format, kody błędów, uwierzyt.    │
└───────────────────────────┬─────────────────────────────────┘
                            │ wywołuje przypadki użycia
┌───────────────────────────▼─────────────────────────────────┐
│  APLIKACJA (application)                                    │
│  Przypadki użycia: wypożycz książkę, zwróć książkę          │
│  Odpowiada za: kolejność kroków, granice transakcji,        │
│  orkiestrację, uprawnienia na poziomie operacji             │
└───────────────────────────┬─────────────────────────────────┘
                            │ operuje na domenie
┌───────────────────────────▼─────────────────────────────────┐
│  DOMENA (domain)                                            │
│  Encje, value objects, reguły biznesowe, niezmienniki       │
│  Odpowiada za: „co jest prawdą w tym biznesie”              │
│  NIE WIE NIC o bazie, HTTP, JSON, SQLAlchemy                │
└───────────────────────────▲─────────────────────────────────┘
                            │ dostarcza implementacje portów
┌───────────────────────────┴─────────────────────────────────┐
│  INFRASTRUKTURA (infrastructure)                            │
│  SQLAlchemy, repozytoria, e-mail, płatności, S3             │
│  Odpowiada za: „jak to zapisać / wysłać / odczytać”         │
└─────────────────────────────────────────────────────────────┘
```

Zwróć uwagę na kierunek strzałek. Prezentacja → aplikacja → domena to zależności. Infrastruktura **nie jest** na końcu łańcucha — ona jest *z boku* i to domena definiuje jej interfejs, choć implementacja siedzi na zewnątrz. To jest esencja **reguły zależności** (dependency rule): zależności kodu wskazują do wewnątrz, w stronę domeny, nigdy na zewnątrz.

> 💡 **Analogia** — pomyśl o gniazdku elektrycznym. Domena to urządzenie, które mówi: „potrzebuję 230 V w tym kształcie wtyczki”. Infrastruktura to dostawca prądu, który dostarcza dokładnie to. Urządzenie nie zna elektrowni, nie wie, czy prąd pochodzi z węgla, wiatru czy atomu — i to jest cała jego siła. Jeśli w Europie zmienisz elektrownię, nie wyrzucasz czajnika.

### 2.2. Tabela „co wolno importować z czego”

Zasada zależności w praktyce to reguła o importach. Bez niej „warstwy” są dekoracją.

| Warstwa \ Import | presentation | application | domain | infrastructure |
|---|---|---|---|---|
| **presentation** | ✅ | ✅ | ✅ (tylko odczyt modeli) | ⚠️ tylko przez punkty wejścia (fabryka UoW, konfiguracja) |
| **application** | ❌ | ✅ | ✅ | ❌ (używa **portów**, nie klas) |
| **domain** | ❌ | ❌ | ✅ | ❌ |
| **infrastructure** | ❌ | ❌ | ✅ (implementuje porty zdefiniowane w aplikacji/domenie) | ✅ |

Cztery puste pola w kolumnie „domain” to najważniejsza część tej tabeli. Jeśli w pliku z regułą biznesową pojawi się `from sqlalchemy import ...`, reguła ta stała się nieprzenośna, nietestowalna bez bazy i zależna od technologii.

Praktyczna reguła: **jeśli w warstwie domeny musisz zaimportować SQLAlchemy, żeby wyrazić regułę biznesową, prawie zawsze znaczy to, że reguła jest źle umiejscowiona.**

> 🔬 **Pod maską — jak to sprawdzić w 10 sekundach.** Architektura w Pythonie nie wymaga narzędzi droższych niż `grep`. Katalog `domain/` powinien mieć zero wystąpień `sqlalchemy`, `fastapi`, `pydantic` (poza typami value objects, jeśli naprawdę chcesz — o tym w sekcji 6) i `requests`. To nie jest „architektura dla architektury” — to jedno polecenie w CI:
>
> ```bash
> grep -R "sqlalchemy\|fastapi\|requests" app/domain/ && exit 1 || exit 0
> ```

### 2.3. Czy zawsze potrzebujesz czterech warstw?

Nie. To jest najważniejsze zdanie tego podrozdziału i wrócimy do niego w sekcji 9. Sensowne są trzy poziomy złożoności:

- **Prosta aplikacja (skrypt, CLI, prototyp, MVP):** dwie warstwy — „kod aplikacji” i „kod bazy”. W praktyce: `models.py`, `services.py`, `main.py`. Reguły biznesowe żyją w `services.py`. To wystarcza na 80 % projektów.
- **Aplikacja średnia (produkt, wielu programistów):** trzy warstwy — prezentacja, serwisy aplikacyjne, infrastruktura ORM. Model ORM jest *jednocześnie* modelem domenowym. To najczęstszy pragmatyczny wybór w świecie Pythona.
- **Aplikacja rozbudowana (złożona domena, wiele interfejsów, współdzielona baza):** cztery warstwy z jawnym rozdzieleniem modelu domenowego od ORM.

Zła wiadomość: nie ma algorytmu, który to wybiera za Ciebie. Dobra: te trzy warianty są opisane w sekcji 9 tabelą koszt/zysk i możesz je świadomie wybrać.

---

## 3. Model ORM a model domenowy

Zatrzymajmy się przy rozróżnieniu, które w praktyce sprawia najwięcej kłopotu.

**Model ORM** to opis tabeli wyrażony w klasach Pythona. Odpowiada na pytania: jak nazywa się kolumna, jaki ma typ, jakie ograniczenia, jaka jest relacja do innej kolumny. Jest to mapa między światem relacyjnym a obiektami.

**Model domenowy** to opis pojęć biznesowych. Odpowiada na pytania: co wolno zrobić z wypożyczeniem, co jest niezmiennikiem, kiedy książka jest przeterminowana, jak liczymy karę.

Środowisko Pythonowe ma to do siebie, że w 90 % projektów te dwa modele są jedną klasą. To nie jest grzech — to pragmatyzm. Ale musisz wiedzieć, **kiedy przestaje działać**.

> 💡 **Analogia** — akt osobowy w kadrach a człowiek. Akt osobowy ma pola (PESEL, adres, numer umowy, stanowisko) i pewne operacje dozwolone na dokumencie (wpisanie urlopu, dodanie aneksu). Człowiek ma o wiele więcej: potrafi zachorować, awansować, założyć związek zawodowy, wypowiedzieć umowę z zachowaniem okresu i mieć do tego prawo w rozumieniu przepisów. Dokument *opisuje* część człowieka — ale **nie jest** człowiekiem. Dopóki prawa pracownika są proste, dokument wystarcza jako model. Kiedy zaczynasz rozgrywać skomplikowane sprawy kadrowe, chcesz modelem być „zatrudnienie”, a nie „wiersz w tabeli `employees`”.

### 3.1. Kiedy wystarczy jeden model

Zostań przy jednej klasie (ORM = domena), jeśli spełniasz **wszystkie** poniższe:

- Reguły biznesowe są proste: walidacja pól, kilka warunków, prosty przepływ stanów.
- Aplikacja ma jeden interfejs (np. tylko REST).
- Baza danych jest własnością tego projektu i nie jest współdzielona z innym systemem.
- Zespół jest mały i zgadza się, że model ma dwie role.
- Nazwy w bazie i w biznesie pokrywają się (albo prawie).

### 3.2. Kiedy potrzebujesz dwóch modeli

Rozważ rozdzielenie, gdy pojawia się **choć jedno** z poniższych:

**a) Złożone niezmienniki i operacje zamiast setterów.** Przykład: `Loan` ma `returned_on`, ale dodatkowo stan „zarezerwowane”, karę naliczaną etapami, próg „zwrot z opóźnieniem > 30 dni wymaga zatwierdzenia kierownika”. Taka logika chce być zamknięta w obiekcie, który nie ma `mapped_column` obok, bo to rozprasza i miesza pojęcia.

**b) Współdzielona baza.** Gdy schemat jest współwłasnością trzech systemów, zmiana kolumny w bazie nie może automatycznie przestawić reguły biznesowej w Twoim kodzie — potrzebujesz warstwy tłumaczącej.

**c) Wiele interfejsów.** To samo wypożyczenie obsługiwane przez REST, kolejkę zdarzeń i batch import. Reguła musi być jedna, niezależna od transportu.

**d) Model odczytu różni się od modelu zapisu.** Ekran raportu potrzebuje dziesięciu zdenormalizowanych kolumn z trzech tabel, a formularz — trzech obiektów. Wciskając to w jedną klasę, dostajesz model, który w połowie pól jest `None`.

**e) Zespół dzieli się na „domenę” i „integracje”.** Zmiana w tabeli w jednym module wywołuje błędy w drugim, bo oba są w tym samym pliku.

> ⚠️ **Pułapka** — odwrotny błąd jest równie kosztowny: rozdzielenie modeli **bez powodu**. Jeśli domena to cztery klasy z trzema polami i jednym warunkiem `if`, tworzenie dwóch modeli plus mappera to czysty narzut. Zasada praktyczna: **nie rozdzielaj, dopóki nie umiesz wskazać konkretnego przypadku (a–e), który Cię boli.** „Bo tak się robi profesjonalnie” to nie przypadek.

---

## 4. Data Mapper kontra Active Record

To jest pojęciowe serce modułu. Wzorce pochodzą z katalogu Martina Fowlera z książki *Patterns of Enterprise Application Architecture* (PoEAA) i mają konkretne, techniczne znaczenie — nie są sloganami.

### 4.1. Active Record (rekord aktywny)

W Active Record **wiersz w bazie jest obiektem, a obiekt wie, jak się zapisać.**

```python
# examples/19_active_record.py
# Tak wygląda Active Record — dla porównania.
# To NIE jest SQLAlchemy; tak działa np. Django ORM czy Peewee.

book = Book.objects.get(pk=1)
book.title = "Nowy tytuł"
book.save()               # obiekt sam się zapisuje — to jest sedno wzorca
Book.objects.filter(isbn="9788308060209").delete()
```

Charakterystyka:

- Klasa **dziedziczy** po klasie bazowej ORM (`models.Model`).
- Metody trwałości (`save()`, `delete()`) są częścią obiektu.
- Zapytania to atrybut klasowy (`Book.objects`, `Book.query`).
- Obiekt i wiersz to jedno pojęcie — nie ma mapy oddzielonej od klasy.

**Zalety:** krótka droga od zera do działania, mało pojęć, świetnie działa w prostych CRUD-ach.
**Wady:** logika biznesowa i trwałość są nierozłączne; trudno przetestować obiekt bez bazy; obiekt w pamięci ma „magiczne” połączenie z kontekstem trwałości.

### 4.2. Data Mapper (odwzorowanie danych)

W Data Mapper **obiekt nie wie, że istnieje baza.** Istnieje oddzielna warstwa — *mapper* — która tłumaczy obiekty na wiersze i wiersze na obiekty.

```python
# examples/19_data_mapper.py
# Data Mapper w wydaniu SQLAlchemy: obiekt jest zwykłą klasą,
# a mapowanie to osobna konfiguracja.

from dataclasses import dataclass

from sqlalchemy import Integer, String, Table, Column, create_engine, select
from sqlalchemy.orm import Session, registry

mapper_registry = registry()  # «rejestr mapowań» — o nim w sekcji 5

book_table = Table(
    "book",
    mapper_registry.metadata,
    Column("id", Integer, primary_key=True),
    Column("title", String(200), nullable=False),
)


@dataclass
class Book:
    """Klasyczna klasa Pythona. Zero importów z SQLAlchemy."""
    id: int | None = None
    title: str = ""


# DOPIERO TERAZ pojawia się zależność: rejestr dowiaduje się o klasie.
mapper_registry.map_imperatively(Book, book_table)

engine = create_engine("sqlite:///:memory:")
mapper_registry.metadata.create_all(engine)

with Session(engine) as session:
    session.add(Book(title="Pan Tadeusz"))  # Session wie, jak zapisać Book
    session.commit()

    stmt = select(Book).where(Book.title.like("Pan%"))
    print(session.scalars(stmt).all())
    # [Book(id=1, title='Pan Tadeusz')]
```

Zwróć uwagę na kluczowy fakt: `Book` **nie ma** metody `save()`. Nie ma atrybutu `objects`. Nie dziedziczy po niczym z SQLAlchemy. Cała wiedza o bazie siedzi w rejestrze i w `Session`.

> 🧠 **Dlaczego tak jest** — bo w Pythonie `session.add(book)`, a nie `book.save()`. Trwałość jest **usługą zewnętrzną**, którą *wstrzykujesz*, a nie cechą obiektu. Brzmi jak szczegół składniowy, ale ma wielkie konsekwencje: ten sam obiekt `Book` możesz zapisać do SQLite, PostgreSQL albo do słownika w pamięci — bez zmiany ani jednej linii w klasie.

### 4.3. Tabela porównawcza

| Cecha | SQLAlchemy 2.0 | Django ORM | SQLModel |
|---|---|---|---|
| Wzorzec | **Data Mapper** | **Active Record** | Data Mapper (opakowany deklaratywnie) |
| Obiekt wie o bazie? | Nie | Tak | Nie wprost, ale klasa ma konfigurację tabeli |
| Trwałość | `session.add(obj)` + `commit()` | `obj.save()` | `session.add(obj)` |
| Zapytania | `session.execute(select(Book))` | `Book.objects.filter(...)` | `session.exec(select(Book))` |
| Warstwa mapowania | jawna (`registry`, `Mapper`) | ukryta w klasie bazowej | jawna, ale deklaratywna |
| Rozdzielenie modelu ORM i domeny | trywialne (mapowanie imperatywne) | trudne (wymaga przepisywania) | możliwe, ale nie jest celem |
| Sterowanie SQL-em | pełne (`text()`, `literal_column()`, Core) | ograniczone | dobre, dziedziczy Core |
| Dojrzałość ekosystemu migracji | Alembic | wbudowane w Django | Alembic |

Tabela nie mówi „SQLAlchemy jest lepsze”. Mówi: **SQLAlchemy jest narzędziem o innym wzorcu podstawowym.** Jeśli potrzebujesz, by obiekty były czyste, a mapowanie wymienne — SQLAlchemy daje to z pudełka. Django ORM daje za to spójność i mniej decyzji. To wybór, nie ranking.

> 🆕 **SQLAlchemy 2.1** — mapowanie klas `dataclass` zostało poprawione tak, że domyślne wartości nie są już umieszczane w `__dict__` instancji przy tworzeniu obiektu. Ma to znaczenie właśnie przy mapowaniu istniejących klas domenowych: obiekt zaczyna zachowywać się zgodnie z regułami dataclass, a jednocześnie mapper SQLAlchemy nie „widzi” fałszywych atrybutów ustawionych domyślnie. W 2.0.x to zachowanie bywało źródłem mylących różnic między obiektem świeżo utworzonym a obiektem wczytanym z bazy.

---

## 5. Mapowanie imperatywne — dowód, że mapowanie to konfiguracja

W module [`07_modele_deklaratywne.md`](07_modele_deklaratywne.md) poznałeś styl deklaratywny: klasa bazowa `DeclarativeBase`, `Mapped[...]`, `mapped_column(...)`. To styl, w którym definiujesz klasę **i** jej mapowanie w jednym miejscu.

W stylu **imperatywnym** (imperative mapping) rozdzielasz te dwie rzeczy:

- Klasa to zwykła klasa Pythona.
- Tabela to obiekt `Table` (styl Core z modułu [`03_metadata_ddl.md`](03_metadata_ddl.md)).
- Połączenie ich to jedno wywołanie: `registry.map_imperatively(Class, table)`.

To nie jest ciekawostka ani API dla zaawansowanych — to **dowód konstrukcyjny**, że SQLAlchemy jest Data Mapperem, a nie Active Recordem z inną składnią.

> 💡 **Analogia** — deklaratywne mapowanie to mebel z IKEI: płyta i instrukcja montażu w jednym pudełku. Mapowanie imperatywne to stolarz, który ma gotową szafkę (klasa) i osobno projekt (tabela), a na końcu je łączy. Oba dają tę samą szafkę — ale w drugim wariancie szafkę możesz używać bez projektu i projekt bez szafki.

### 5.1. `registry` — co to właściwie jest

`registry` (rejestr) to **centralny spis mapowań**. Zna:

- zbiór tabel (`registry.metadata`),
- mapę „klasa → mapper”,
- konfigurację globalną mapowania.

Gdy piszesz `class Base(DeclarativeBase): pass`, pod spodem **też** powstaje rejestr — dostępny jako `Base.registry`. Styl deklaratywny to więc wygodne opakowanie na rejestr, nie coś fundamentalnie innego.

```python
# examples/19_registry.py
from sqlalchemy.orm import DeclarativeBase, registry


class Base(DeclarativeBase):
    pass


print(Base.registry)          # <sqlalchemy.orm.decl_api.registry object at ...>
print(Base.metadata is Base.registry.metadata)
# True — deklaratywna klasa bazowa używa rejestru pod spodem
```

### 5.2. `registry.mapped` — deklaratywnie, ale bez `DeclarativeBase`

Pierwszy wariant: definiujesz klasę normalnie, ale oznaczasz ją dekoratorem rejestru. Zyskujesz `Mapped[]` i `mapped_column()`, a mimo to klasa **nie dziedziczy** po żadnej klasie bazowej SQLAlchemy.

```python
# examples/19_registry_mapped.py
from __future__ import annotations

from datetime import date

from sqlalchemy import Date, String, create_engine, select
from sqlalchemy.orm import Mapped, Session, mapped_column, registry

mapper_registry = registry()


@mapper_registry.mapped
class Book:
    """Klasa NIE dziedziczy po DeclarativeBase — tylko po object."""

    __tablename__ = "book"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    published_on: Mapped[date | None] = mapped_column(Date, default=None)


print(Book.__mro__)
# (<class 'Book'>, <class 'object'>)  <-- brak klasy bazowej ORM!
print(Book.__mapper__)
# <Mapper at 0x...; Book>

engine = create_engine("sqlite:///:memory:")
mapper_registry.metadata.create_all(engine)

with Session(engine) as session:
    session.add(Book(title="Lalka", published_on=date(1890, 1, 1)))
    session.commit()
    print(session.scalars(select(Book)).all())
    # [Book(id=1, title='Lalka', published_on=datetime.date(1890, 1, 1))]
```

Dla porównania — ta sama klasa w stylu deklaratywnym dziedziczyłaby po `Base`, co daje jej `Book.metadata`, `Book.__table__`, ale też wpycha do hierarchii klas techniczne elementy. W wersji z rejestrem `Book.__mro__` to czyste `(Book, object)`.

### 5.3. `map_imperatively` — pełna kontrola

Drugi wariant: definiujesz tabelę jako obiekt `Table` i łączysz ją z **istniejącą** klasą.

```python
# examples/19_map_imperatively.py
from __future__ import annotations

from dataclasses import dataclass
from datetime import date

from sqlalchemy import (
    Column,
    Date,
    ForeignKey,
    Integer,
    String,
    Table,
    create_engine,
    select,
)
from sqlalchemy.orm import Session, registry, relationship

mapper_registry = registry()


# --- 1. Klasa domenowa: czysty Python, ZERO importów z SQLAlchemy ---

@dataclass
class Author:
    id: int | None = None
    name: str = ""


@dataclass
class Book:
    id: int | None = None
    title: str = ""
    published_on: date | None = None
    author_id: int | None = None


# --- 2. Tabele: klasyczny Core ---

author_table = Table(
    "author",
    mapper_registry.metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String(120), nullable=False),
)

book_table = Table(
    "book",
    mapper_registry.metadata,
    Column("id", Integer, primary_key=True),
    Column("title", String(200), nullable=False),
    Column("published_on", Date),
    Column("author_id", ForeignKey("author.id"), nullable=False),
)


# --- 3. Mapowanie: jedno wywołanie na klasę ---

mapper_registry.map_imperatively(Author, author_table)
mapper_registry.map_imperatively(Book, book_table)

engine = create_engine("sqlite:///:memory:")
mapper_registry.metadata.create_all(engine)

with Session(engine) as session:
    author = Author(name="Bolesław Prus")
    session.add(author)
    session.flush()  # potrzebujemy id autora
    print(author.id)  # 1

    session.add(Book(title="Lalka", published_on=date(1890, 1, 1), author_id=author.id))
    session.commit()

    stmt = select(Book).where(Book.published_on < date(1900, 1, 1))
    print(session.scalars(stmt).all())
    # [Book(id=1, title='Lalka', published_on=datetime.date(1890, 1, 1), author_id=1)]
```

Zauważ, że `Book` i `Author` **nie mają** `__tablename__`, nie mają `mapped_column`, nie mają żadnego SQLAlchemy w definicji. Ich związek z bazą jest w pełni zewnętrzny.

> 🧠 **Dlaczego tak jest** — `dataclass` generuje `__init__`, więc mapper ma jak tworzyć instancje. Ale uwaga: gdy SQLAlchemy **wczytuje** wiersz z bazy, nie wywołuje `__init__` — tworzy obiekt przez `__new__` i ustawia atrybuty bezpośrednio. Dzięki temu nie musisz przekazywać wszystkich pól w konstruktorze przy odczycie i nie uruchamiasz walidacji z `__post_init__` dla danych już zapisanych. To zachowanie jest zamierzone, ale bywa zaskakujące — zapamiętaj je.

### 5.4. Dodawanie właściwości przy mapowaniu

`map_imperatively` przyjmuje parametr `properties`, którym możesz dopiąć zachowania bez dotykania klasy:

```python
# examples/19_properties.py
# ... (importy i tabele jak wyżej, pominięte dla zwięzłości)

mapper_registry.map_imperatively(
    Book,
    book_table,
    properties={
        # kolumna `book_table.c.author_id` mapowana na atrybut `author_id`
        "author_id": book_table.c.author_id,
        # kolumna, której nazwa różni się od atrybutu:
        # "published": book_table.c.published_on,
    },
)
```

### 5.5. Mapowanie do `join` i do `select`

W Data Mapper „tabela” nie musi być tabelą. Mapper akceptuje obiekt `Join` (złączenie) albo podzapytanie — wtedy klasa jest mapowana na wynik złączenia, a nie na pojedynczą tabelę.

```python
# examples/19_map_to_join.py
# ... (tabele author_table, book_table jak wyżej, pominięte)

from sqlalchemy import join

book_with_author = join(book_table, author_table)


@dataclass
class BookWithAuthor:
    id: int | None = None
    title: str = ""
    author_name: str = ""


mapper_registry.map_imperatively(
    BookWithAuthor,
    book_with_author,
    properties={
        # przy joinach SQLAlchemy wymaga wskazania, KTÓRĄ kolumnę `id` brać —
        # obie tabele mają kolumnę `id`, więc mapowanie musi być jednoznaczne
        "id": [book_table.c.id, author_table.c.id],
        "title": book_table.c.title,
        "author_name": author_table.c.name,
    },
)
```

To jest technicznie zaawansowane i rzadko potrzebne, ale pokazuje coś istotnego: **mapper mapuje nie „tabelę”, tylko zbiór kolumn.** Dzięki temu ten sam mechanizm obsługuje widoki, złączenia i podzapytania.

### 5.6. Mapowanie po fakcie i koszty

Zdarza się, że klasa jest już używana, a mapa dopisujesz później:

```python
# examples/19_register_late.py
from sqlalchemy.orm import clear_mappers, registry

# mapper_registry.map_imperatively(Book, book_table)   # gdzieś w orm.py

# w testach, gdy chcesz przebudować mapowania od zera:
# clear_mappers()
```

> ⚠️ **Pułapka** — `clear_mappers()` czyści **globalne** mapowania i jest narzędziem do testów infrastruktury, nie elementem cyklu życia aplikacji. W kodzie produkcyjnym jego użycie prawie zawsze oznacza, że moduły z mapowaniami są importowane w złej kolejności albo że testy współdzielą globalny stan. Nie używaj go jako obejścia — napraw importy.

Trzy konsekwencje praktyczne mapowania imperatywnego:

1. **Klasa domenowa może zostać użyta bez bazy.** Tworzysz `Book(...)` w teście, nie masz silnika, nie masz sesji — i wszystko działa.
2. **Mapowanie to konfiguracja, więc można je warunkować.** W niektórych projektach mapowanie dla testów szybkich podmienia się na wersję mapującą klasę na tymczasową tabelę.
3. **Moduł mapujący musi zostać zaimportowany.** To najczęstszy błąd początkujących: `UnmappedInstanceError` przy `session.add(book)`, bo `orm.py` nie został zaimportowany i rejestr nie wie o klasie.

---

## 6. Jak modelować domenę: encje, value objects, agregaty

Jeśli decydujesz się na osobny model domenowy, musisz wiedzieć, jak go budować. Nie chodzi o akademickie DDD — chodzi o trzy konkretne decyzje, które mają realne konsekwencje techniczne.

### 6.1. Encje bogate i encje anemiczne

**Encja anemiczna** to worek na dane ze zbiorem getterów i setterów. Cała logika siedzi w serwisach.

```python
# examples/19_anemic.py
# Encja anemiczna: dane i nic więcej. Reguły żyją na zewnątrz.
@dataclass
class Loan:
    id: int | None
    member_id: int
    book_id: int
    due_on: date
    returned_on: date | None = None
```

**Encja bogata** sama pilnuje swoich niezmienników i udostępnia **operacje**, nie settery.

```python
# examples/19_rich.py
from dataclasses import dataclass
from datetime import date
from decimal import Decimal


class DomainError(Exception):
    """Bazowy wyjątek reguły biznesowej."""


class AlreadyReturned(DomainError):
    pass


@dataclass
class Loan:
    id: int | None
    member_id: int
    book_id: int
    due_on: date
    returned_on: date | None = None

    FINE_PER_DAY = Decimal("0.50")

    def is_active(self) -> bool:
        return self.returned_on is None

    def is_overdue(self, today: date) -> bool:
        return self.is_active() and self.due_on < today

    def days_overdue(self, today: date) -> int:
        if not self.is_overdue(today):
            return 0
        return (today - self.due_on).days

    def fine(self, today: date) -> Decimal:
        return self.days_overdue(today) * self.FINE_PER_DAY

    def register_return(self, today: date) -> None:
        if self.returned_on is not None:
            raise AlreadyReturned(f"Wypożyczenie {self.id} już zwrócone")
        self.returned_on = today
```

Wersja bogata nie pozwala „ustawić” `returned_on` dowolną wartość — pozwala wykonać operację „zwróć”, która sama sprawdza, czy to ma sens. Naruszenie niezmiennika staje się **niemożliwe**, a nie tylko *odradzane*.

> 💡 **Analogia** — encja anemiczna to formularz papierowy z rubrykami: każdy może wpisać w rubrykę „data zwrotu” cokolwiek, nawet datę z przyszłości i dwa razy. Encja bogata to urzędnik przy okienku: przyjmuje wniosek „zwrot”, sprawdza dowód, stawia pieczątkę i odnotowuje datę. Nie da się „ustawić daty zwrotu” bez złożenia wniosku.

Który wybrać? Uczciwa odpowiedź: **większość projektów Pythonowych dziś używa wariantu anemicznego z serwisami** i to działa. Bogate encje opłacają się, gdy reguł jest naprawdę dużo, gdy stan ma więcej niż 3–4 przejścia, gdy zmieniają się często i gdy testowanie reguł bez bazy ma wartość.

### 6.2. Value objects

**Value object** (obiekt wartości) to obiekt bez tożsamości, definiowany przez wartość. Dwa `Money("10", "PLN")` są tym samym. Nie ma `id`, nie ma cyklu życia, jest niemutowalny.

W Pythonie naturalnym wyborem jest `@dataclass(frozen=True)`.

```python
# examples/19_value_objects.py
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class ISBN:
    """Numer ISBN-13 znormalizowany do 13 cyfr. Niemutowalny, z walidacją."""

    value: str

    def __post_init__(self) -> None:
        digits = self.value.replace("-", "").replace(" ", "")
        if len(digits) != 13 or not digits.isdigit():
            raise ValueError(f"Nieprawidłowy ISBN: {self.value!r}")
        # frozen=True blokuje zwykły zapis — używamy obejścia przez object
        object.__setattr__(self, "value", digits)

    def __str__(self) -> str:
        return self.value


a = ISBN("978-83-08-06020-9")
b = ISBN("9788308060209")
print(a == b, hash(a) == hash(b))
# True True  <-- równość po wartości, można trzymać w zbiorze
```

#### Zapis value object do bazy: `TypeDecorator`

Value object nie jest typem, który baza zna. Trzeba go „przetłumaczyć”. Robi to `TypeDecorator` z modułu [`12_typy_i_wlasne_typy.md`](12_typy_i_wlasne_typy.md).

```python
# examples/19_isbn_type.py
from sqlalchemy import String, TypeDecorator


class ISBNType(TypeDecorator[ISBN]):
    """Kolumna VARCHAR(13) <-> obiekt ISBN."""

    impl = String(13)
    cache_ok = True  # bez tego SQLAlchemy wyłącza cache kompilacji dla tej kolumny

    def process_bind_param(self, value: ISBN | str | None, dialect) -> str | None:
        if value is None:
            return None
        if isinstance(value, ISBN):
            return str(value)
        return str(ISBN(value))

    def process_result_value(self, value: str | None, dialect) -> ISBN | None:
        return None if value is None else ISBN(value)
```

Teraz mapowanie imperatywne używa tego typu:

```python
# examples/19_isbn_mapping.py
book_table = Table(
    "book",
    mapper_registry.metadata,
    Column("id", Integer, primary_key=True),
    Column("title", String(200), nullable=False),
    Column("isbn", ISBNType(), nullable=True),
)
```

> 🧠 **Dlaczego tak jest** — domena operuje na pojęciach (`ISBN`), a baza na typach, które rozumie. `TypeDecorator` to **jedyne** miejsce, w którym te dwa światy się spotykają, i dlatego warto trzymać je w jednym pliku infrastruktury, a nie rozsypywać po domenie.

> ⚠️ **Pułapka** — `@dataclass(frozen=True)` **bez** `slots=True` automatycznie ustawia `__hash__` tylko dlatego, że `frozen=True`. Ale `@dataclass` **bez** `frozen` i z domyślnym `eq=True` ustawia `__hash__ = None`, czyniąc obiekt **niehaszowalnym**. Jeśli próbujesz wrzucić taki obiekt do `set` albo użyć jako klucza słownika, dostaniesz `TypeError: unhashable type`. Value objects → zawsze `frozen=True`.

### 6.3. Agregaty — bez akademizmu

**Agregat** to grupa obiektów, które zmieniają się razem i mają jednego „strażnika” — **korzeń agregatu** (aggregate root). Zasada jest prosta w skutkach:

> Nie sięgaj do wnętrza agregatu z zewnątrz. Do wszystkiego służy korzeń.

```python
# examples/19_aggregate.py
from dataclasses import dataclass, field


@dataclass
class LoanItem:
    """Pozycja wypożyczenia. Nie ma znaczenia poza wypożyczeniem."""
    book_id: int


@dataclass
class Loan:
    """KORZEŃ agregatu. Tylko on jest adresowany z repozytorium."""
    id: int | None
    member_id: int
    items: list[LoanItem] = field(default_factory=list)

    MAX_ITEMS = 5

    def add_item(self, book_id: int) -> None:
        if len(self.items) >= self.MAX_ITEMS:
            raise DomainError(f"Nie można dodać więcej niż {self.MAX_ITEMS} pozycji")
        if any(item.book_id == book_id for item in self.items):
            raise DomainError("Ta książka już jest w wypożyczeniu")
        self.items.append(LoanItem(book_id=book_id))
```

Konsekwencja techniczna: w repozytorium nie ma metody `add_item_to_loan` — ładujesz `Loan` i wywołujesz `loan.add_item(...)`. Jedna brama, jedno miejsce, w którym można złamać niezmiennik — i to miejsce jest chronione.

Praktyczna zasada dla SQLAlchemy: **jednostka pracy i repozytorium operują na korzeniach agregatów.** Reszta idzie kaskadą relacji (moduł [`09_relacje.md`](09_relacje.md)).

---

## 7. DTO i modele odczytu

Mamy domenę. Teraz musimy odpowiedzieć na pytanie: **co wychodzi z aplikacji na zewnątrz?**

### 7.1. Dlaczego nie zwracamy encji ORM do API

To pytanie wraca w każdym projekcie. Powody są konkretne i techniczne:

**1. Leniwe ładowanie poza sesją.** Endpoint zwraca encję, framework ją serializuje, serializacja dotyka `book.author` — ale sesja jest już zamknięta (bo zależność `get_session` zamknęła ją po zakończeniu requestu). Efekt: `DetachedInstanceError`. Klasyka opisana w module [`11_ladowanie_i_n_plus_1.md`](11_ladowanie_i_n_plus_1.md).

**2. Cykle.** `Book.author.books[0].author.books[0]...` — naiwna serializacja nie skończy się nigdy albo skończy się błędem rekurencji.

**3. Wyciek pól.** Encja ma `internal_note`, `cost_price`, `deleted_at`, `legacy_code`. Klient nie powinien tego widzieć, ale widzi, bo serializacja bierze wszystko.

**4. Kontrakt jest niejawny.** Zmiana nazwy kolumny w modelu zmienia kontrakt API. To jest niedopuszczalne: model bazy i kontrakt API mają różne tempo zmian.

**5. Brak kontroli nad kształtem.** Klient chce `author: {"name": "..."}`, a encja ma `author_id: 7`.

### 7.2. Schematy Pydantic v2 jako DTO

Najprostsze, sprawdzone w Pythonie rozwiązanie: osobny model Pydantic.

```python
# examples/19_dto_pydantic.py
from datetime import date

from pydantic import BaseModel, ConfigDict, Field


class BookRead(BaseModel):
    """Kontrakt wyjściowy API. Świadomie NIE jest encją ORM."""

    model_config = ConfigDict(from_attributes=True)

    id: int
    title: str = Field(min_length=1, max_length=200)
    isbn: str | None = None
    author_name: str | None = None


class BookCreate(BaseModel):
    """Kontrakt wejściowy. Tu jest miejsce na walidację formatu."""

    title: str = Field(min_length=1, max_length=200)
    isbn: str | None = None
    author_id: int
```

`from_attributes=True` sprawia, że `BookRead.model_validate(obj)` działa zarówno dla encji ORM, jak i dla `dataclass`, i dla słownika. To mostek między warstwami — i celowo **jednokierunkowy**.

> 🔬 **Pod maską — dlaczego DTO ratuje przed `DetachedInstanceError`.** `from_attributes=True` czyta **atrybuty, które już są w pamięci**. Jeśli pole `author_name` nie istnieje w encji, Pydantic nie „pójdzie do bazy” — po prostu nie znajdzie atrybutu i zgłosi błąd walidacji. To dobrze! Lepiej dostać czytelny `ValidationError` na etapie dewelopmentu niż produkcyjny wyjątek leniwego ładowania. Rozwiązanie: albo pole w DTO jest wypełniane z projekcji zapytania (patrz niżej), albo relacja jest jawnie doładowana przez `selectinload()`.

### 7.3. Modele odczytu dla raportów

Raport rzadko potrzebuje encji. Potrzebuje **kolumn** z kilku tabel.

```python
# examples/19_read_model.py
from datetime import date
from typing import TypedDict

from sqlalchemy import Date, Integer, String, select
from sqlalchemy.orm import Session


class OverdueLoanRow(TypedDict):
    """Model odczytu raportu. Nie ma tożsamości, nie da się go zapisać."""
    loan_id: int
    member_name: str
    book_title: str
    days_overdue: int


def list_overdue(session: Session, *, today: date) -> list[OverdueLoanRow]:
    stmt = (
        select(
            Loan.id.label("loan_id"),
            Member.name.label("member_name"),
            Book.title.label("book_title"),
            Loan.due_on.label("due_on"),
        )
        .join(Member, Loan.member_id == Member.id)
        .join(Book, Loan.book_id == Book.id)
        .where(Loan.returned_on.is_(None), Loan.due_on < today)
        .order_by(Loan.due_on)
    )

    return [
        OverdueLoanRow(
            loan_id=row.loan_id,
            member_name=row.member_name,
            book_title=row.book_title,
            days_overdue=(today - row.due_on).days,
        )
        for row in session.execute(stmt)
    ]
```

> 🔬 **Pod maską** — powyższe zapytanie kompiluje się do:
>
> ```sql
> SELECT loan.id AS loan_id,
>        member.name AS member_name,
>        book.title AS book_title,
>        loan.due_on AS due_on
> FROM loan
> JOIN member ON loan.member_id = member.id
> JOIN book   ON loan.book_id = book.id
> WHERE loan.returned_on IS NULL AND loan.due_on < ?
> ORDER BY loan.due_on
> ```
>
> Zwróć uwagę: **żadnej encji w wyniku.** To jedna kolumna na pole DTO, dokładnie tyle danych, ile trzeba. `days_overdue` liczę w Pythonie, a nie w SQL, bo odejmowanie dat daje różne typy w różnych dialektach (w PostgreSQL `date - date → integer`, w SQLite to zwykłe liczby — przenośność przeważa).

`TypedDict` daje sprawdzanie typów w edytorze i `mypy` za darmo. Do raportów zwykle lepszy niż `dataclass`, bo jest po prostu słownikiem z adnotacjami — łatwo go zwrócić jako JSON.

### 7.4. Gdzie budować DTO?

Nie w endpoincie. Trzy miejsca i ich zastosowania:

| Gdzie | Kiedy | Dlaczego |
|---|---|---|
| W warstwie aplikacji (serwis/przypadek użycia) | Domyślnie | Warstwa aplikacji wie, czego potrzebuje klient, i ma dostęp do sesji, by zbudować projekcję |
| W warstwie prezentacji (FastAPI `response_model`) | Gdy encja jest w pełni załadowana i trywialna | Mniej kodu, akceptowalne w prostych CRUD-ach |
| W repozytorium | Gdy zapytanie raportowe jest złożone i zwraca własny kształt | Repozytorium zna SQL, więc wie, jak zbudować projekcję |

> ⚠️ **Pułapka** — DTO, które dziedziczy po encji ORM, nie jest DTO. `class BookRead(Book)`, żeby „nie powtarzać pól”, to najczęstszy skrót i najczęstsza przyczyna wycieku pól oraz cyklicznych zależności. DTO ma **własną, pełną definicję** — nawet jeśli wygląda podobnie.

---

## 8. Granice transakcji należą do warstwy aplikacji

To jedna z tych decyzji, które są niewidoczne w kodzie, dopóki nie zdarzy się błąd w połowie operacji.

**Granica transakcji** to miejsce, w którym mówimy: „wszystko, co się stało od tego momentu do tego momentu, zapisuje się razem albo wcale”.

Kto powinien ją wyznaczać?

- **Warstwa prezentacji?** Nie. Endpoint nie wie, że operacja „wypożycz książkę” składa się z trzech zapisów. To wiedza biznesowa.
- **Domena?** Nie. Domena nie wie nic o bazie, więc nie może wiedzieć, kiedy `COMMIT`.
- **Repozytorium?** **Nie.** I to jest najczęstszy błąd. Repozytorium, które robi `commit()`, uniemożliwia transakcję obejmującą dwa repozytoria.
- **Warstwa aplikacji?** Tak. Przypadek użycia to dokładnie jedna granica transakcji.

> 💡 **Analogia** — przelew bankowy. Repozytorium to kasjer, który umie wykonać polecenie „zmniejsz saldo” albo „zwiększ saldo”. Ale to nie kasjer decyduje, że obie operacje muszą się udać razem. Decyduje system transakcyjny, czyli warstwa wyżej. Gdyby kasjer po każdej czynności sam zamykał dzień, przelew w połowie zostawiłby pieniądze w powietrzu.

### 8.1. Zły i dobry wzorzec

```python
# examples/19_tx_bad.py
# ŹLE — repozytorium samo commituje.

class BookRepository:
    def __init__(self, session: Session) -> None:
        self._session = session

    def add(self, book: Book) -> None:
        self._session.add(book)
        self._session.commit()   # <-- KATASTROFA


# Teraz nie da się zapisać książki i autora w jednej transakcji:
# jeśli autor się nie zapisze, książka już jest w bazie.
```

```python
# examples/19_tx_good.py
# DOBRZE — repozytorium tylko odkłada zmiany, granica jest wyżej.

class BookRepository:
    def __init__(self, session: Session) -> None:
        self._session = session

    def add(self, book: Book) -> None:
        self._session.add(book)   # flush należy do sesji, commit do przypadku użycia

    def get(self, book_id: int) -> Book | None:
        return self._session.get(Book, book_id)


def borrow_book(uow: UnitOfWork, *, member_id: int, book_id: int) -> Loan:
    with uow:                       # <-- GRANICA TRANSAKCJI
        member = uow.members.get(member_id)
        book = uow.books.get(book_id)
        loan = make_loan(member, book)     # reguły biznesowe
        uow.loans.add(loan)
        uow.commit()
    return loan
```

Cały `with uow:` to jedna transakcja. Wyjątek w środku → `__exit__` robi `rollback()`. Sukces → `commit()` wywołany jawnie w przypadku użycia.

> 🧠 **Dlaczego tak jest** — bo SQLAlchemy `Session` **jest już** jednostką pracy (moduł [`08_sesja_cykl_zycia.md`](08_sesja_cykl_zycia.md)). Sesja trzyma kolejkę zmian i wysyła je w odpowiedniej kolejności. Nie musisz tego implementować — musisz tylko **wyznaczyć moment commitu** i w tym celu wystarczy jedna reguła: *commit należy do przypadku użycia, nie do obiektu technicznego*.

Pełną implementację `UnitOfWork` zobaczysz w module [`21_unit_of_work.md`](21_unit_of_work.md). Tutaj ważna jest decyzja architektoniczna, nie kod.

### 8.2. Co jeszcze ma granicę

Granica transakcji nie jest jedyną granicą, którą wyznacza warstwa aplikacji. W tym samym miejscu podejmuje się decyzje o:

- **kolejności operacji** (najpierw walidacja, potem zapis),
- **skutkach ubocznych** (wyślij e-mail **po** commicie — nigdy wewnątrz transakcji),
- **ponowieniach** (retry na deadlocku z modułu [`14_transakcje_i_wspolbieznosc.md`](14_transakcje_i_wspolbieznosc.md)),
- **uprawnieniach** (czy ten członek może wypożyczać).

> ⚠️ **Pułapka** — wysyłanie e-maila albo wołanie zewnętrznego API **wewnątrz** transakcji to klasyka. Jeśli transakcja się wycofa, e-mail już poszedł. Rozwiązanie: albo wysyłka po `commit()`, albo wzorzec **outbox** (zapis „do wysłania” do tabeli w tej samej transakcji, a wysyłka przez osobny proces). To ten sam problem co w systemach rozproszonych — tylko w mniejszej skali.

---

## 9. Trzy warianty architektury do wyboru

To sekcja, po którą powinieneś wracać do tego modułu. Zamiast „architektura albo nie”, dostajesz trzy świadome wybory.

### Wariant A — modele ORM + serwisy

```text
app/
├── models.py           # encje ORM (jednocześnie model domenowy)
├── services.py         # funkcje/klasy z regułami biznesowymi
├── schemas.py          # Pydantic DTO
├── api/
│   └── routes.py       # endpointy wywołujące serwisy
└── database.py         # engine, sessionmaker, get_session
```

**Kiedy:** małe i średnie projekty, jeden interfejs, prosta domena, zespół 1–4 osób.
**Koszt:** niski. Uczysz się jednego modelu.
**Zysk:** szybkość, prostota, mniej plików.
**Ryzyko:** domena i baza są sklejone; jeśli projekt urośnie 5×, refaktor będzie bolesny.

### Wariant B — repozytoria + jednostka pracy, model ORM jako domena

```text
app/
├── domain/              # (opcjonalnie) kilka reguł w formie funkcji
├── application/
│   ├── borrow_book.py   # przypadki użycia
│   └── ports.py         # protokoły repozytoriów i UoW
├── infrastructure/
│   ├── orm.py           # modele ORM
│   ├── repositories.py  # implementacje portów
│   └── uow.py           # jednostka pracy
├── api/
└── schemas.py
```

**Kiedy:** średnie i duże projekty, wielu programistów, testy mają wartość, domena rośnie.
**Koszt:** średni — dodatkowe pliki i jedna warstwa pośrednia.
**Zysk:** testowalność bez bazy, wymienialność zapytań, jasne granice transakcji.
**Ryzyko:** *zbyt wczesne* wprowadzenie to narzut bez korzyści (patrz: pułapka na końcu sekcji).

### Wariant C — bogata domena + mapowanie imperatywne

```text
app/
├── domain/              # czysty Python: encje, value objects, reguły
│   ├── model.py
│   ├── errors.py
│   └── services.py
├── application/
│   ├── use_cases/
│   └── ports.py
├── infrastructure/
│   ├── orm.py           # mapowanie imperatywne + TypeDecoratory
│   ├── repositories.py
│   └── uow.py
├── api/
└── schemas.py
```

**Kiedy:** złożona domena (wiele niezmienników, przepływy stanów, reguły dojrzałe), baza współdzielona, wiele interfejsów, długi horyzont życia produktu.
**Koszt:** wysoki — dwa modele, mappery, dyscyplina zespołu, szkolenie.
**Zysk:** domena testowalna bez żadnej infrastruktury, reguły niezależne od technologii, możliwość zmiany bazy bez dotykania logiki.
**Ryzyko:** „architektura dla architektury”; przy prostej domenie to strata czasu i czytelności.

### Tabela decyzyjna

| Kryterium | A | B | C |
|---|---|---|---|
| Liczba reguł biznesowych | < 15 | 15–50 | > 50 lub szybko rośnie |
| Liczba interfejsów (REST/CLI/kolejka) | 1 | 1–2 | 3+ |
| Zespół | 1–4 | 4–10 | 10+ lub wielu zespołów |
| Baza współdzielona z innym systemem | nie | nie | tak |
| Wymóg testów bez bazy | miły | ważny | kluczowy |
| Horyzont życia projektu | < 2 lata | 2–5 lat | 5+ lat |
| **Koszt wdrożenia** | niski | średni | wysoki |
| **Koszt utrzymania przy wzroście** | wysoki | średni | niski |

> ⚠️ **Pułapka — „architektura dla architektury”**. Najczęstszy błąd to wybranie wariantu C dla aplikacji, która ma trzy encje i jeden warunek `if`. Objaw: 30 plików, 12 protokołów, jedna implementacja każdego, testy dłuższe niż kod biznesowy. Zasada: **zacznij od A, przejdź do B, gdy boli, do C tylko wtedy, gdy boli naprawdę.** Refaktoryzacja z A do B jest tania, jeśli serwisy już są — bo repozytorium to wyciągnięcie zapytań z serwisu.

> ⚠️ **Pułapka — „mapowanie domeny na ORM 1:1 bez powodu”**. Jeśli `domain/model.py` i `infrastructure/orm.py` mają te same klasy, te same pola i żadnej różnicy poza tym, że jedna ma `Mapped[...]` — to nie są dwa modele, to jeden model zduplikowany. Objaw: każda zmiana wymaga dwóch edycji i pisania mappera. Jeśli nie potrafisz wskazać **konkretnej różnicy** (inne typy, inne nazwy, inne pola, inne relacje), zostań przy wariancie A lub B.

> ⚠️ **Pułapka — `Session` przeciekająca do domeny.** Jeśli w `domain/` pojawia się `session.flush()`, to znaczy, że domena potrzebuje danych, których nie powinna potrzebować. Reguła jest wtedy źle podzielona: potrzebny jest **argument**, nie sesja. Zamiast `loan.check_limit(session)`, powinno być `loan.check_limit(active_count)`, a liczenie należy do repozytorium.

> ⚠️ **Pułapka — importy w obie strony.** `domain/model.py` importuje `infrastructure.orm`, a `infrastructure.orm` importuje `domain.model`. W Pythonie to nie zawsze wybuchnie od razu (czasem tylko daje `ImportError: cannot import name X from partially initialized module`), ale zawsze będzie problemem przy testach i przy podziale na pakiety. Kierunek zależności z sekcji 2 jest po to, żeby takich cykli nie było.

---

## 10. Przykład obowiązkowy: te same trzy przypadki użycia w wariancie A i B

Poniżej budujemy **dwa razy to samo**. Domena: biblioteka. Trzy przypadki użycia:

1. `borrow_book` — wypożyczenie (zapisy, dwie reguły biznesowe),
2. `return_book` — zwrot z naliczeniem kary (zapis),
3. `list_overdue_loans` — raport przeterminowanych (odczyt).

Wariant A zapisujemy w plikach `app_a_*.py`, wariant B w `app_b_*`. Na końcu porównujemy je pod kątem testowalności i czytelności.

### 10.1. Wspólne reguły biznesowe

```text
R1. Członek musi istnieć i być aktywny.
R2. Członek może mieć najwyżej 5 aktywnych wypożyczeń.
R3. Książki nie można wypożyczyć, jeśli jest już wypożyczona.
R4. Kara wynosi 0,50 zł za każdy dzień po terminie.
R5. Termin zwrotu = data wypożyczenia + 30 dni.
```

Te reguły są identyczne w obu wariantach. Różnica jest wyłącznie w tym, **gdzie mieszkają i kto je wywołuje**.

### 10.2. Wariant A — modele ORM + serwisy

#### Model i DTO

```python
# examples/app_a_models.py
from __future__ import annotations

from datetime import date
from decimal import Decimal

from sqlalchemy import Date, ForeignKey, Numeric, String
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    pass


class Member(Base):
    __tablename__ = "member"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(120))
    is_active: Mapped[bool] = mapped_column(default=True)

    loans: Mapped[list[Loan]] = relationship(back_populates="member")


class Book(Base):
    __tablename__ = "book"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    author_name: Mapped[str] = mapped_column(String(120))

    loans: Mapped[list[Loan]] = relationship(back_populates="book")


class Loan(Base):
    __tablename__ = "loan"

    id: Mapped[int] = mapped_column(primary_key=True)
    member_id: Mapped[int] = mapped_column(ForeignKey("member.id"))
    book_id: Mapped[int] = mapped_column(ForeignKey("book.id"))
    borrowed_on: Mapped[date] = mapped_column(Date)
    due_on: Mapped[date] = mapped_column(Date)
    returned_on: Mapped[date | None] = mapped_column(Date, default=None)
    fine_amount: Mapped[Decimal | None] = mapped_column(Numeric(10, 2), default=None)

    member: Mapped[Member] = relationship(back_populates="loans")
    book: Mapped[Book] = relationship(back_populates="loans")

    # Reguły są w encji — wariant A też może mieć logikę w obiekcie!
    def is_active(self) -> bool:
        return self.returned_on is None

    def days_overdue(self, today: date) -> int:
        if not self.is_active():
            return 0
        return max(0, (today - self.due_on).days)
```

> 🧠 **Dlaczego tak jest** — w wariancie A encja ORM pełni obie role. To *nie znaczy*, że musi być anemiczna. Dodanie kilku metod na encji jest tanie i podnosi czytelność. Wariant A różni się od C nie tym, że „nie ma logiki w obiekcie”, ale tym, że **obiekt nie jest oddzielony od bazy**.

#### Serwis

```python
# examples/app_a_services.py
from __future__ import annotations

from datetime import date, timedelta
from decimal import Decimal

from sqlalchemy import exists, func, select
from sqlalchemy.orm import Session

from app_a_models import Book, Loan, Member

MAX_ACTIVE_LOANS = 5
LOAN_PERIOD_DAYS = 30
FINE_PER_DAY = Decimal("0.50")


# --- wyjątki aplikacyjne (NIE HTTP!) ---

class ApplicationError(Exception):
    """Bazowy wyjątek warstwy aplikacji."""


class NotFound(ApplicationError):
    def __init__(self, entity: str, entity_id: int) -> None:
        super().__init__(f"Nie znaleziono {entity} o id={entity_id}")
        self.entity = entity
        self.entity_id = entity_id


class BusinessRuleViolation(ApplicationError):
    pass


# --- przypadki użycia ---

def borrow_book(
    session: Session,
    *,
    member_id: int,
    book_id: int,
    today: date | None = None,
) -> Loan:
    today = today or date.today()

    member = session.get(Member, member_id)
    if member is None:
        raise NotFound("member", member_id)
    if not member.is_active:
        raise BusinessRuleViolation("Członek jest nieaktywny")

    active_count = session.scalar(
        select(func.count())
        .select_from(Loan)
        .where(Loan.member_id == member_id, Loan.returned_on.is_(None))
    )
    if active_count is not None and active_count >= MAX_ACTIVE_LOANS:
        raise BusinessRuleViolation(
            f"Limit {MAX_ACTIVE_LOANS} aktywnych wypożyczeń został osiągnięty"
        )

    already_out = session.scalar(
        select(
            exists().where(
                Loan.book_id == book_id,
                Loan.returned_on.is_(None),
            )
        )
    )
    if already_out:
        raise BusinessRuleViolation("Książka jest już wypożyczona")

    loan = Loan(
        member_id=member_id,
        book_id=book_id,
        borrowed_on=today,
        due_on=today + timedelta(days=LOAN_PERIOD_DAYS),
    )
    session.add(loan)
    return loan
```

```python
# examples/app_a_services_return.py
# ... (nagłówek i wyjątki jak wyżej, pominięte dla zwięzłości)

def return_book(
    session: Session,
    *,
    loan_id: int,
    today: date | None = None,
) -> Loan:
    today = today or date.today()

    loan = session.get(Loan, loan_id)
    if loan is None:
        raise NotFound("loan", loan_id)
    if not loan.is_active():
        raise BusinessRuleViolation("To wypożyczenie zostało już zwrócone")

    loan.returned_on = today
    fine = loan.days_overdue(today) * FINE_PER_DAY
    loan.fine_amount = fine if fine > 0 else None
    return loan
```

```python
# examples/app_a_services_report.py
from typing import TypedDict


class OverdueRow(TypedDict):
    loan_id: int
    member_name: str
    book_title: str
    days_overdue: int


def list_overdue_loans(session: Session, *, today: date | None = None) -> list[OverdueRow]:
    today = today or date.today()

    stmt = (
        select(
            Loan.id.label("loan_id"),
            Member.name.label("member_name"),
            Book.title.label("book_title"),
            Loan.due_on.label("due_on"),
        )
        .join(Member, Loan.member_id == Member.id)
        .join(Book, Loan.book_id == Book.id)
        .where(Loan.returned_on.is_(None), Loan.due_on < today)
        .order_by(Loan.due_on)
    )

    return [
        OverdueRow(
            loan_id=row.loan_id,
            member_name=row.member_name,
            book_title=row.book_title,
            days_overdue=(today - row.due_on).days,
        )
        for row in session.execute(stmt)
    ]
```

#### Użycie w endpoincie (z granicą transakcji)

```python
# examples/app_a_api.py
from datetime import date

from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session

from app_a_services import (
    BusinessRuleViolation,
    NotFound,
    borrow_book,
    list_overdue_loans,
    return_book,
)

router = APIRouter()


@router.post("/loans", status_code=201)
def create_loan(
    member_id: int,
    book_id: int,
    session: Session = Depends(get_session),
) -> dict:
    try:
        loan = borrow_book(session, member_id=member_id, book_id=book_id)
        session.commit()                      # <-- granica transakcji
    except NotFound as exc:
        session.rollback()
        raise HTTPException(status_code=404, detail=str(exc)) from exc
    except BusinessRuleViolation as exc:
        session.rollback()
        raise HTTPException(status_code=409, detail=str(exc)) from exc
    return {"loan_id": loan.id, "due_on": loan.due_on.isoformat()}
```

> 🔬 **Pod maską — co leci do bazy przy `borrow_book`.**
>
> ```sql
> -- 1. session.get(Member, member_id)
> SELECT member.id, member.name, member.is_active
> FROM member WHERE member.id = ?
>
> -- 2. liczenie aktywnych wypożyczeń
> SELECT count(*) AS count_1
> FROM loan
> WHERE loan.member_id = ? AND loan.returned_on IS NULL
>
> -- 3. sprawdzenie dostępności książki
> SELECT EXISTS (SELECT * FROM loan
>                WHERE loan.book_id = ? AND loan.returned_on IS NULL) AS anon_1
>
> -- 4. INSERT przy flush/commit
> INSERT INTO loan (member_id, book_id, borrowed_on, due_on, returned_on, fine_amount)
> VALUES (?, ?, ?, ?, ?, ?)
> ```

#### Test wariantu A

```python
# examples/test_app_a.py
from datetime import date, timedelta

import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import Session

from app_a_models import Base, Book, Member
from app_a_services import BusinessRuleViolation, MAX_ACTIVE_LOANS, borrow_book


@pytest.fixture
def session():
    engine = create_engine("sqlite:///:memory:")
    Base.metadata.create_all(engine)
    with Session(engine) as s:
        yield s


@pytest.fixture
def world(session: Session) -> tuple[Member, Book]:
    member = Member(name="Anna", is_active=True)
    book = Book(title="Lalka", author_name="Bolesław Prus")
    session.add_all([member, book])
    session.flush()
    return member, book


def test_borrow_creates_loan(session: Session, world) -> None:
    member, book = world
    loan = borrow_book(session, member_id=member.id, book_id=book.id)

    assert loan.id is not None           # flush przed odczytem id
    assert loan.due_on == date.today() + timedelta(days=30)
    assert loan.returned_on is None


def test_borrow_rejects_inactive_member(session: Session, world) -> None:
    member, book = world
    member.is_active = False
    session.flush()

    with pytest.raises(BusinessRuleViolation, match="nieaktywny"):
        borrow_book(session, member_id=member.id, book_id=book.id)


def test_borrow_enforces_limit(session: Session) -> None:
    member = Member(name="Anna", is_active=True)
    session.add(member)
    session.flush()

    books = [Book(title=f"Tom {i}", author_name="X") for i in range(MAX_ACTIVE_LOANS + 1)]
    session.add_all(books)
    session.flush()

    for book in books[:MAX_ACTIVE_LOANS]:
        borrow_book(session, member_id=member.id, book_id=book.id)

    with pytest.raises(BusinessRuleViolation, match="Limit"):
        borrow_book(session, member_id=member.id, book_id=books[-1].id)
```

Testy działają, ale wymagają **bazy, tabel i sesji** — nawet dla sprawdzenia reguły „nieaktywny członek”, która jest przecież czystym warunkiem logicznym.

### 10.3. Wariant B — domena, porty, repozytoria, jednostka pracy

#### Domena — czysty Python

```python
# examples/app_b_domain_model.py
"""Model domenowy. ZERO importów z SQLAlchemy, FastAPI, Pydantic."""
from __future__ import annotations

from dataclasses import dataclass, field
from datetime import date, timedelta
from decimal import Decimal

MAX_ACTIVE_LOANS = 5
LOAN_PERIOD_DAYS = 30
FINE_PER_DAY = Decimal("0.50")


# --- błędy domenowe (nie wiedzą nic o HTTP) ---

class DomainError(Exception):
    """Naruszenie reguły biznesowej."""


class MemberNotActive(DomainError):
    pass


class LoanLimitExceeded(DomainError):
    pass


class BookNotAvailable(DomainError):
    pass


class LoanAlreadyReturned(DomainError):
    pass


# --- encje ---

@dataclass
class Book:
    id: int | None
    title: str
    author_name: str


@dataclass
class Member:
    id: int | None
    name: str
    is_active: bool = True


@dataclass
class Loan:
    id: int | None
    member_id: int
    book_id: int
    borrowed_on: date
    due_on: date
    returned_on: date | None = None
    fine_amount: Decimal | None = None

    # --- reguły na obiekcie ---

    def is_active(self) -> bool:
        return self.returned_on is None

    def days_overdue(self, today: date) -> int:
        if not self.is_active():
            return 0
        return max(0, (today - self.due_on).days)

    def register_return(self, today: date) -> None:
        if not self.is_active():
            raise LoanAlreadyReturned(f"Wypożyczenie {self.id} już zwrócone")
        self.returned_on = today
        fine = self.days_overdue(today) * FINE_PER_DAY
        self.fine_amount = fine if fine > 0 else None

    @classmethod
    def open_for(
        cls,
        *,
        member: Member,
        book: Book,
        active_loans: int,
        book_is_taken: bool,
        today: date,
    ) -> Loan:
        """Fabryka pilnująca reguł R1-R3, R5."""
        if not member.is_active:
            raise MemberNotActive(f"Członek {member.name} jest nieaktywny")
        if active_loans >= MAX_ACTIVE_LOANS:
            raise LoanLimitExceeded(
                f"Limit {MAX_ACTIVE_LOANS} aktywnych wypożyczeń został osiągnięty"
            )
        if book_is_taken:
            raise BookNotAvailable(f"Książka {book.title!r} jest już wypożyczona")
        if member.id is None or book.id is None:
            raise DomainError("Encje muszą mieć nadane identyfikatory")

        return cls(
            id=None,
            member_id=member.id,
            book_id=book.id,
            borrowed_on=today,
            due_on=today + timedelta(days=LOAN_PERIOD_DAYS),
        )


@dataclass(frozen=True, slots=True)
class OverdueEntry:
    """Model odczytu zbudowany w warstwie aplikacji."""
    loan_id: int
    member_name: str
    book_title: str
    days_overdue: int
```

Zwróć uwagę: `Loan.open_for(...)` **nie przyjmuje sesji ani repozytorium**. Przyjmuje dane, których potrzebuje (`active_loans`, `book_is_taken`). To jest techniczny wyraz zasady „domena nie zna infrastruktury” z sekcji 2 — nie przez deklarację, ale przez sygnaturę funkcji.

#### Porty — protokoły bez SQLAlchemy

```python
# examples/app_b_ports.py
"""Porty (interfejsy) zdefiniowane w warstwie aplikacji."""
from __future__ import annotations

from datetime import date
from typing import Protocol, Self

from app_b_domain_model import Book, Loan, Member, OverdueEntry


class BookRepository(Protocol):
    def get(self, book_id: int) -> Book | None: ...
    def add(self, book: Book) -> None: ...
    def is_taken(self, book_id: int) -> bool: ...


class MemberRepository(Protocol):
    def get(self, member_id: int) -> Member | None: ...


class LoanRepository(Protocol):
    def get(self, loan_id: int) -> Loan | None: ...
    def add(self, loan: Loan) -> None: ...
    def count_active_for_member(self, member_id: int) -> int: ...
    def list_overdue(self, today: date) -> list[OverdueEntry]: ...


class UnitOfWork(Protocol):
    books: BookRepository
    members: MemberRepository
    loans: LoanRepository

    def __enter__(self) -> Self: ...
    def __exit__(self, exc_type, exc, tb) -> None: ...
    def commit(self) -> None: ...
    def rollback(self) -> None: ...
```

Te protokoły nie wiedzą, że istnieje SQLAlchemy. Warstwa infrastruktury dostarcza implementacje.

#### Przypadki użycia

```python
# examples/app_b_use_cases.py
from __future__ import annotations

from datetime import date

from app_b_domain_model import Book, Loan, Member
from app_b_ports import UnitOfWork


class EntityNotFound(Exception):
    """Warstwa aplikacji nie znalazła encji w repozytorium."""

    def __init__(self, entity: str, entity_id: int) -> None:
        super().__init__(f"Nie znaleziono {entity} o id={entity_id}")
        self.entity = entity
        self.entity_id = entity_id


def borrow_book(
    uow: UnitOfWork,
    *,
    member_id: int,
    book_id: int,
    today: date | None = None,
) -> Loan:
    today = today or date.today()

    with uow:                                   # <-- GRANICA TRANSAKCJI
        member = uow.members.get(member_id)
        if member is None:
            raise EntityNotFound("member", member_id)

        book = uow.books.get(book_id)
        if book is None:
            raise EntityNotFound("book", book_id)

        loan = Loan.open_for(
            member=member,
            book=book,
            active_loans=uow.loans.count_active_for_member(member_id),
            book_is_taken=uow.books.is_taken(book_id),
            today=today,
        )
        uow.loans.add(loan)
        uow.commit()

    return loan


def return_book(
    uow: UnitOfWork,
    *,
    loan_id: int,
    today: date | None = None,
) -> Loan:
    today = today or date.today()

    with uow:
        loan = uow.loans.get(loan_id)
        if loan is None:
            raise EntityNotFound("loan", loan_id)

        loan.register_return(today)             # reguła jest w domenie
        uow.commit()

    return loan


def list_overdue_loans(uow: UnitOfWork, *, today: date | None = None) -> list[object]:
    with uow:
        return uow.loans.list_overdue(today or date.today())
```

Przypadek użycia w wariancie B jest **krótszy** od wariantu A i nie zawiera ani jednego znaku SQL ani jednego warunku biznesowego. Cała orkiestracja to: pobierz → wywołaj regułę → zapisz → commit.

#### Infrastruktura — mapowanie imperatywne

```python
# examples/app_b_infrastructure_orm.py
"""Mapowanie imperatywne encji domenowych na tabele. Tu i TYLKO TU
pojawia się SQLAlchemy w warstwie infrastruktury."""
from __future__ import annotations

from sqlalchemy import (
    Boolean,
    Column,
    Date,
    ForeignKey,
    Integer,
    Numeric,
    String,
    Table,
)
from sqlalchemy.orm import registry

from app_b_domain_model import Book, Loan, Member

mapper_registry = registry()

member_table = Table(
    "member",
    mapper_registry.metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String(120), nullable=False),
    Column("is_active", Boolean, nullable=False, default=True),
)

book_table = Table(
    "book",
    mapper_registry.metadata,
    Column("id", Integer, primary_key=True),
    Column("title", String(200), nullable=False),
    Column("author_name", String(120), nullable=False),
)

loan_table = Table(
    "loan",
    mapper_registry.metadata,
    Column("id", Integer, primary_key=True),
    Column("member_id", ForeignKey("member.id"), nullable=False),
    Column("book_id", ForeignKey("book.id"), nullable=False),
    Column("borrowed_on", Date, nullable=False),
    Column("due_on", Date, nullable=False),
    Column("returned_on", Date, nullable=True),
    Column("fine_amount", Numeric(10, 2), nullable=True),
)


def start_mappers() -> None:
    """Rejestruje mapowania. Wywoływane RAZ przy starcie aplikacji.

    Funkcja jest potrzebna, bo samo `import` modułu nie wystarcza —
    chcemy mieć jedno, jawne miejsce, w którym wiadomo, że mapowania żyją.
    """
    mapper_registry.map_imperatively(Member, member_table)
    mapper_registry.map_imperatively(Book, book_table)
    mapper_registry.map_imperatively(Loan, loan_table)
```

> ⚠️ **Pułapka** — brak wywołania `start_mappers()` przy starcie aplikacji to najczęstszy błąd przy mapowaniu imperatywnym. Objaw: `UnmappedInstanceError: Class 'app_b_domain_model.Book' is not mapped`. Naprawa: wywołaj `start_mappers()` w punkcie wejścia aplikacji (`main.py`, `conftest.py`) albo — jeśli wolisz podejście importowe — zaimportuj moduł z mapowaniami w module z UoW.

> ⚠️ **Pułapka** — `@dataclass` z `slots=True` **nie zadziała** dla klas mapowanych. `slots=True` tworzy nową klasę i SQLAlchemy nie może dopiąć instrumentacji atrybutów. Używaj `slots=True` **tylko** dla value objects (`ISBN`, `OverdueEntry`) i DTO. Encje mapowane: zwykły `@dataclass`, ewentualnie z `eq=False`.

#### Infrastruktura — repozytoria

```python
# examples/app_b_infrastructure_repositories.py
from __future__ import annotations

from datetime import date

from sqlalchemy import exists, func, select
from sqlalchemy.orm import Session

from app_b_domain_model import Book, Loan, Member, OverdueEntry


class SqlAlchemyBookRepository:
    def __init__(self, session: Session) -> None:
        self._session = session

    def get(self, book_id: int) -> Book | None:
        return self._session.get(Book, book_id)

    def add(self, book: Book) -> None:
        self._session.add(book)

    def is_taken(self, book_id: int) -> bool:
        stmt = select(
            exists().where(Loan.book_id == book_id, Loan.returned_on.is_(None))
        )
        return bool(self._session.scalar(stmt))


class SqlAlchemyMemberRepository:
    def __init__(self, session: Session) -> None:
        self._session = session

    def get(self, member_id: int) -> Member | None:
        return self._session.get(Member, member_id)


class SqlAlchemyLoanRepository:
    def __init__(self, session: Session) -> None:
        self._session = session

    def get(self, loan_id: int) -> Loan | None:
        return self._session.get(Loan, loan_id)

    def add(self, loan: Loan) -> None:
        self._session.add(loan)

    def count_active_for_member(self, member_id: int) -> int:
        stmt = (
            select(func.count())
            .select_from(Loan)
            .where(Loan.member_id == member_id, Loan.returned_on.is_(None))
        )
        return self._session.scalar(stmt) or 0

    def list_overdue(self, today: date) -> list[OverdueEntry]:
        stmt = (
            select(
                Loan.id.label("loan_id"),
                Member.name.label("member_name"),
                Book.title.label("book_title"),
                Loan.due_on.label("due_on"),
            )
            .join(Member, Loan.member_id == Member.id)
            .join(Book, Loan.book_id == Book.id)
            .where(Loan.returned_on.is_(None), Loan.due_on < today)
            .order_by(Loan.due_on)
        )
        return [
            OverdueEntry(
                loan_id=row.loan_id,
                member_name=row.member_name,
                book_title=row.book_title,
                days_overdue=(today - row.due_on).days,
            )
            for row in self._session.execute(stmt)
        ]
```

Zwróć uwagę, jak wygląda repozytorium: **żadnego `commit()`**, żadnego `flush()`, żadnej logiki biznesowej. Wyłącznie tłumaczenie pytań domenowych na SQL.

#### Infrastruktura — jednostka pracy

```python
# examples/app_b_infrastructure_uow.py
from __future__ import annotations

from sqlalchemy.orm import Session, sessionmaker
from typing import Self

from app_b_infrastructure_repositories import (
    SqlAlchemyBookRepository,
    SqlAlchemyLoanRepository,
    SqlAlchemyMemberRepository,
)


class SqlAlchemyUnitOfWork:
    """Jedna sesja, jedno `commit()`, jedno miejsce na `rollback()`.

    Kompletna implementacja — wraz z uzasadnieniem każdej decyzji —
    znajduje się w module 21.
    """

    def __init__(self, session_factory: sessionmaker[Session]) -> None:
        self._session_factory = session_factory
        self._session: Session | None = None

    def __enter__(self) -> Self:
        self._session = self._session_factory()
        self.books = SqlAlchemyBookRepository(self._session)
        self.members = SqlAlchemyMemberRepository(self._session)
        self.loans = SqlAlchemyLoanRepository(self._session)
        return self

    def __exit__(self, exc_type, exc, tb) -> None:
        assert self._session is not None
        try:
            if exc_type is not None:
                self.rollback()
        finally:
            self._session.close()

    def commit(self) -> None:
        assert self._session is not None
        self._session.commit()

    def rollback(self) -> None:
        assert self._session is not None
        self._session.rollback()
```

> 🧠 **Dlaczego repozytoria dzielą jedną sesję.** To jest najważniejszy szczegół techniczny tego wariantu. Jedna sesja oznacza: jedną transakcję, jedną **identity map** (moduł [`08_sesja_cykl_zycia.md`](08_sesja_cykl_zycia.md)) i jedną kolejkę zmian. Gdyby każde repozytorium miało własną sesję, `uow.loans.count_active_for_member()` widziałby **inną** wersję danych niż ta, na której operuje domena — i nie dałoby się zapisać dwóch obiektów atomowo.

#### Test — bez bazy, bez SQL, w milisekundach

```python
# examples/test_app_b_unit.py
"""Testy reguł domenowych. Brak bazy, brak sesji, brak SQLAlchemy."""
from datetime import date, timedelta
from decimal import Decimal

import pytest

from app_b_domain_model import (
    Book,
    FINE_PER_DAY,
    Loan,
    Member,
    MemberNotActive,
)
from app_b_use_cases import EntityNotFound, borrow_book


class FakeBookRepository:
    def __init__(self) -> None:
        self.items: list[Book] = []
        self.taken: set[int] = set()

    def get(self, book_id: int) -> Book | None:
        return next((b for b in self.items if b.id == book_id), None)

    def add(self, book: Book) -> None:
        self.items.append(book)

    def is_taken(self, book_id: int) -> bool:
        return book_id in self.taken


class FakeMemberRepository:
    def __init__(self) -> None:
        self.items: list[Member] = []

    def get(self, member_id: int) -> Member | None:
        return next((m for m in self.items if m.id == member_id), None)


class FakeLoanRepository:
    def __init__(self) -> None:
        self.items: list[Loan] = []

    def get(self, loan_id: int) -> Loan | None:
        return next((loan for loan in self.items if loan.id == loan_id), None)

    def add(self, loan: Loan) -> None:
        loan.id = len(self.items) + 1
        self.items.append(loan)

    def count_active_for_member(self, member_id: int) -> int:
        return sum(1 for loan in self.items if loan.member_id == member_id and loan.is_active())

    def list_overdue(self, today: date) -> list[object]:
        return []


class FakeUnitOfWork:
    def __init__(self) -> None:
        self.books = FakeBookRepository()
        self.members = FakeMemberRepository()
        self.loans = FakeLoanRepository()
        self.committed = False
        self.rolled_back = False

    def __enter__(self) -> "FakeUnitOfWork":
        return self

    def __exit__(self, exc_type, exc, tb) -> None:
        if exc_type is not None:
            self.rollback()

    def commit(self) -> None:
        self.committed = True

    def rollback(self) -> None:
        self.rolled_back = True


@pytest.fixture
def uow() -> FakeUnitOfWork:
    u = FakeUnitOfWork()
    u.members.items.append(Member(id=1, name="Anna", is_active=True))
    u.books.items.append(Book(id=10, title="Lalka", author_name="Bolesław Prus"))
    return u


def test_borrow_happy_path(uow: FakeUnitOfWork) -> None:
    loan = borrow_book(uow, member_id=1, book_id=10, today=date(2026, 9, 1))

    assert loan.due_on == date(2026, 10, 1)
    assert uow.committed is True
    assert uow.loans.items == [loan]


def test_borrow_rejects_inactive_member(uow: FakeUnitOfWork) -> None:
    uow.members.items[0].is_active = False

    with pytest.raises(MemberNotActive):
        borrow_book(uow, member_id=1, book_id=10)
    assert uow.committed is False


def test_borrow_rejects_missing_book(uow: FakeUnitOfWork) -> None:
    with pytest.raises(EntityNotFound, match="book"):
        borrow_book(uow, member_id=1, book_id=999)


def test_fine_is_charged_per_day_overdue() -> None:
    loan = Loan(
        id=1,
        member_id=1,
        book_id=10,
        borrowed_on=date(2026, 8, 1),
        due_on=date(2026, 8, 31),
    )
    loan.register_return(date(2026, 9, 10))

    assert loan.days_overdue(date(2026, 9, 10)) == 10
    assert loan.fine_amount == 10 * FINE_PER_DAY
```

Te testy nie potrzebują ani silnika, ani tabel, ani dysku. Cztery pliki testów uruchamiają się w czasie krótszym niż zaimportowanie SQLAlchemy.

### 10.4. Porównanie: czytelność i testowalność

| Kryterium | Wariant A | Wariant B |
|---|---|---|
| Liczba plików dla 3 przypadków użycia | 4 | 7 |
| Liczba abstrakcji do zrozumienia | 3 (`Session`, model, serwis) | 7 (`Session`, model, UoW, 3 repozytoria, porty) |
| Linie kodu przypadków użycia | ~120 (z zapytaniami) | ~85 (bez SQL) |
| SQL w przypadkach użycia | tak (w serwisach) | nie (w repozytoriach) |
| Test reguły „nieaktywny członek” bez bazy | **nie** | **tak** |
| Test reguły „nieaktywny członek” w ms | ~80 ms (baza + tabele) | ~0,1 ms (obiekty w pamięci) |
| Trudność zmiany bazy danych | wysoka (SQL w serwisach) | niska (tylko `infrastructure/`) |
| Trudność raportu z 4 tabel | łatwa (zapytanie w serwisie) | łatwa (metoda repozytorium) |
| Krzywa wejścia dla nowego programisty | łagodna | stroma |
| Ryzyko nadmiernej abstrakcji | niskie | wysokie |

**Uczciwy wniosek z tego porównania jest następujący:**

Wariant A z powyższego przykładu **nie jest złym kodem**. Serwis z trzema zapytaniami i pięcioma warunkami jest czytelny, testowalny (z bazą) i łatwy do zmiany. Gdyby to była cała aplikacja, wariant B byłby przesadą.

Wariant B zaczyna wygrywać, gdy:

1. Reguł przybywa i **zmieniają się częściej niż zapytania** — wtedy testy domenowe w milisekundach ratują czas.
2. Dochodzi drugi interfejs (kolejka, CLI, import) — logika przypadków użycia jest już gotowa, wystarczy nowy adapter.
3. Baza ma być wymienialna albo część reguł ma działać na danych z innego źródła.
4. Zespół rośnie i chcemy, by osoby mniej doświadczone pisały repozytoria (mechaniczne tłumaczenie), a doświadczone — domenę (reguły).

Zwróć też uwagę na rzecz, która często umyka w dyskusjach: **`Loan.register_return()` i `Loan.open_for()` można mieć w wariancie A.** Zysk z wariantu B nie polega na „logice w obiekcie”, tylko na **niezależności obiektu od bazy**. To jest różnica, którą naprawdę kupujesz.

> 🆕 **SQLAlchemy 2.1** — w kontekście wariantu B warto odnotować dwie zmiany. Po pierwsze, mapowanie klas `dataclass` nie umieszcza już domyślnych wartości w `__dict__`, dzięki czemu obiekt domenowy po utworzeniu wygląda tak samo, jak obiekt wczytany z bazy — co eliminuje klasę subtelnych błędów przy porównywaniu „świeżego” obiektu z „odczytanym”. Po drugie, typowanie `Result` i `Row` zostało rozszerzone (PEP 646), więc `session.execute(stmt).all()` w przypadku projekcji wielokolumnowych zwraca typ, który `mypy` rozumie precyzyjniej niż w 2.0 — to realna korzyść dla modeli odczytu z sekcji 7.3.

---

## Podsumowanie

1. **Warstwy to role, nie katalogi.** Prezentacja odpowiada za transport, aplikacja za orkiestrację i granice transakcji, domena za reguły, infrastruktura za technologię.
2. **Reguła zależności**: importy wskazują do wewnątrz. W katalogu `domain/` nie ma SQLAlchemy — i da się to sprawdzić jednym `grep`em w CI.
3. **Model ORM ≠ model domenowy**, ale często *powinny być tym samym*. Rozdzielaj, gdy masz konkretny przypadek (złożone niezmienniki, współdzielona baza, wiele interfejsów), nie „bo tak się robi”.
4. **SQLAlchemy to Data Mapper.** `session.add(obj) + commit()` zamiast `obj.save()`. Obiekt może nie mieć w sobie ani jednego znaku ORM.
5. **`registry` + `map_imperatively`** to dowód, że mapowanie jest konfiguracją, a nie dziedziczeniem. `registry.mapped` daje styl deklaratywny bez klasy bazowej, `map_imperatively` mapuje istniejące klasy.
6. **Value objects** (`@dataclass(frozen=True, slots=True)`) wchodzą do bazy przez `TypeDecorator`. Pamiętaj o `cache_ok = True`.
7. **Nie zwracaj encji ORM do API.** DTO z Pydantic v2 (`from_attributes=True`) albo `TypedDict` dla raportów. DTO ma własną definicję, nie dziedziczy po encji.
8. **Granica transakcji należy do przypadku użycia.** Repozytorium nigdy nie robi `commit()`, inaczej nie da się objąć dwóch repozytoriów jedną transakcją.
9. **Skutki uboczne (e-mail, HTTP) po commicie**, nigdy w środku transakcji.
10. **Trzy warianty architektury** (A: ORM + serwisy, B: repozytoria + UoW, C: bogata domena + mapowanie imperatywne) to wybór świadomy, nie ranking. Zacznij od A, przejdź do B, gdy boli, do C tylko gdy boli naprawdę.

---

## Ćwiczenia

### Ćwiczenie 1 — Rozdziel model ORM od domeny (poziom: średni)

Weź encję `Book` z modułu [`09_relacje.md`](09_relacje.md) (książka z autorem, kategoriami i egzemplarzami). Twoje zadanie:

1. Stwórz plik `domain/model.py` z klasą `Book` **bez ani jednego importu** z `sqlalchemy`, `pydantic`, `fastapi`. Klasa ma być `@dataclass`.
2. Dodaj do niej **jedną regułę biznesową**, której nie ma w wersji ORM — np. metodę `can_be_borrowed()`, która sprawdza, czy książka ma co najmniej jeden dostępny egzemplarz i nie jest oznaczona jako wycofana z obiegu.
3. Stwórz `infrastructure/orm.py`, w którym przez `registry.map_imperatively` zmapujesz tę klasę na tabelę `book` z modułu 09.
4. Napisz test, który dowodzi, że reguła działa **bez bazy** — użyj `pytest` i tylko obiektów w pamięci.
5. Uruchom `grep -R "sqlalchemy" domain/` i pokaż wynik w komentarzu do zadania.

### Ćwiczenie 2 — Mapowanie imperatywne dla istniejącej klasy (poziom: średni/trudny)

Masz istniejącą klasę `dataclass` z innego projektu, np.:

```python
@dataclass
class Customer:
    id: int | None
    email: str
    full_name: str
    created_at: datetime
```

Dodaj dozwoloną minimalną modyfikację (np. `email` ma być value objectem `EmailAddress` z walidacją) i napisz:

1. Mapowanie imperatywne dla tabeli `customer` o kolumnach `id`, `email` (VARCHAR(255)), `full_name`, `created_at` (`TIMESTAMP WITH TIME ZONE` — pamiętaj o module [`12_typy_i_wlasne_typy.md`](12_typy_i_wlasne_typy.md)).
2. `TypeDecorator` dla `EmailAddress`.
3. Test integracyjny na SQLite (`sqlite:///:memory:`), który zapisuje i odczytuje obiekt, oraz asercję, że odczytany obiekt ma `email` typu `EmailAddress` (a nie `str`).

### Ćwiczenie 3 — Refaktoryzacja wariantu A do wariantu B (poziom: trudny)

Weź plik `app_a_services.py` z sekcji 10.2 i przekształć go w:

1. `domain/model.py` — encje `Book`, `Member`, `Loan` z regułami (`Loan.open_for`, `Loan.register_return`).
2. `application/ports.py` — protokoły `BookRepository`, `MemberRepository`, `LoanRepository`, `UnitOfWork`.
3. `application/use_cases.py` — trzy przypadki użycia.
4. `infrastructure/repositories.py` — implementacje SQLAlchemy.
5. `infrastructure/uow.py` — jednostka pracy.
6. Zmień testy: te, które sprawdzają reguły, mają **przestać** używać bazy; te, które sprawdzają repozytoria, mają zostać integracyjne.

Na koniec zapisz w komentarzu (albo w pliku `REFACTOR.md`) odpowiedź na pytanie: **ile linii testów udało się uruchamiać bez bazy i ile czasu to zaoszczędziło w praktyce?** Zmierz `pytest --durations=10` przed i po.

### Rozwiązania

#### Rozwiązanie 1

```python
# examples/sol_01_domain_model.py
"""Rozwiązanie 1 — domena bez SQLAlchemy."""
from __future__ import annotations

from dataclasses import dataclass, field


@dataclass
class Category:
    id: int | None
    name: str


@dataclass
class Book:
    id: int | None
    title: str
    isbn: str | None
    withdrawn: bool = False
    available_copies: int = 0
    categories: list[Category] = field(default_factory=list)

    def can_be_borrowed(self) -> bool:
        """Książka jest wypożyczalna, gdy nie jest wycofana i ma wolny egzemplarz."""
        if self.withdrawn:
            return False
        return self.available_copies > 0
```

```python
# examples/sol_01_orm.py
"""Rozwiązanie 1 — mapowanie imperatywne."""
from __future__ import annotations

from sqlalchemy import Boolean, Column, ForeignKey, Integer, String, Table
from sqlalchemy.orm import registry

from sol_01_domain_model import Book, Category

mapper_registry = registry()

category_table = Table(
    "category",
    mapper_registry.metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String(80), nullable=False),
)

book_table = Table(
    "book",
    mapper_registry.metadata,
    Column("id", Integer, primary_key=True),
    Column("title", String(200), nullable=False),
    Column("isbn", String(13), nullable=True),
    Column("withdrawn", Boolean, nullable=False, default=False),
)


def start_mappers() -> None:
    mapper_registry.map_imperatively(Category, category_table)
    mapper_registry.map_imperatively(Book, book_table)
```

```python
# examples/sol_01_test.py
"""Rozwiązanie 1 — test bez bazy. Czas: milisekundy."""
from sol_01_domain_model import Book


def test_book_can_be_borrowed_when_copy_available() -> None:
    book = Book(id=1, title="Lalka", isbn=None, available_copies=2)
    assert book.can_be_borrowed() is True


def test_withdrawn_book_cannot_be_borrowed() -> None:
    book = Book(id=1, title="Lalka", isbn=None, withdrawn=True, available_copies=5)
    assert book.can_be_borrowed() is False


def test_book_without_copies_cannot_be_borrowed() -> None:
    book = Book(id=1, title="Lalka", isbn=None, available_copies=0)
    assert book.can_be_borrowed() is False
```

```bash
$ grep -R "sqlalchemy" domain/
# (brak wyników — kod wyjścia 1, czysto)

$ python -m pytest sol_01_test.py -q
# 3 passed in 0.01s
```

**Komentarz:** reguła `can_be_borrowed()` w wersji ORM wymagałaby sesji albo ładowania relacji `copies` — czyli test przeszedłby przez bazę i zajmował ~80 ms zamiast 3 ms. Przy 50 testach reguł to różnica między 4 s a 0,15 s — a po dodaniu `--cov` i zbierania danych staje się jeszcze większa.

#### Rozwiązanie 2

```python
# examples/sol_02_email_vo.py
"""Rozwiązanie 2 — value object EmailAddress + TypeDecorator."""
from __future__ import annotations

import re
from dataclasses import dataclass

from sqlalchemy import String, TypeDecorator

_EMAIL_RE = re.compile(r"^[^@\s]+@[^@\s]+\.[a-zA-Z]{2,}$")


@dataclass(frozen=True, slots=True)
class EmailAddress:
    value: str

    def __post_init__(self) -> None:
        normalized = self.value.strip().lower()
        if not _EMAIL_RE.match(normalized):
            raise ValueError(f"Nieprawidłowy adres e-mail: {self.value!r}")
        object.__setattr__(self, "value", normalized)

    def __str__(self) -> str:
        return self.value


class EmailType(TypeDecorator[EmailAddress]):
    impl = String(255)
    cache_ok = True

    def process_bind_param(self, value: EmailAddress | str | None, dialect) -> str | None:
        if value is None:
            return None
        if isinstance(value, EmailAddress):
            return str(value)
        return str(EmailAddress(value))

    def process_result_value(self, value: str | None, dialect) -> EmailAddress | None:
        return None if value is None else EmailAddress(value)
```

```python
# examples/sol_02_customer.py
"""Rozwiązanie 2 — klasa domenowa i mapowanie."""
from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime

from sqlalchemy import Column, DateTime, Integer, Table, func
from sqlalchemy.orm import registry


@dataclass
class Customer:
    id: int | None
    email: EmailAddress
    full_name: str
    created_at: datetime | None = None


mapper_registry = registry()

customer_table = Table(
    "customer",
    mapper_registry.metadata,
    Column("id", Integer, primary_key=True),
    Column("email", EmailType(), nullable=False, unique=True),
    Column("full_name", String(200), nullable=False),
    Column("created_at", DateTime(timezone=True), server_default=func.now()),
)

mapper_registry.map_imperatively(Customer, customer_table)
```

```python
# examples/sol_02_test.py
"""Rozwiązanie 2 — test integracyjny: VO przetrwał podróż tam i z powrotem."""
from datetime import datetime, timezone

from sqlalchemy import create_engine
from sqlalchemy.orm import Session

from sol_02_customer import Customer, mapper_registry
from sol_02_email_vo import EmailAddress


def test_email_survives_roundtrip() -> None:
    engine = create_engine("sqlite:///:memory:")
    mapper_registry.metadata.create_all(engine)

    with Session(engine) as session:
        customer = Customer(
            id=None,
            email=EmailAddress("  Anna.KOWALSKA@Example.COM "),
            full_name="Anna Kowalska",
            created_at=datetime.now(timezone.utc),
        )
        session.add(customer)
        session.commit()
        customer_id = customer.id

    with Session(engine) as session:
        loaded = session.get(Customer, customer_id)

    assert loaded is not None
    assert isinstance(loaded.email, EmailAddress)     # nie `str`!
    assert loaded.email.value == "anna.kowalska@example.com"   # znormalizowany
```

**Komentarz:** normalizacja w `__post_init__` odbywa się **raz, przy tworzeniu obiektu** — nie ma szans, by do bazy trafił adres z wielkimi literami i spacjami. `TypeDecorator` jest jedynym miejscem konwersji.

#### Rozwiązanie 3

```python
# examples/sol_03_domain.py
"""Rozwiązanie 3 — domena przeniesiona z wariantu A."""
from __future__ import annotations

from dataclasses import dataclass
from datetime import date, timedelta
from decimal import Decimal

MAX_ACTIVE_LOANS = 5
LOAN_PERIOD_DAYS = 30
FINE_PER_DAY = Decimal("0.50")


class DomainError(Exception):
    pass


class MemberNotActive(DomainError):
    pass


class LoanLimitExceeded(DomainError):
    pass


class BookNotAvailable(DomainError):
    pass


class LoanAlreadyReturned(DomainError):
    pass


@dataclass
class Book:
    id: int | None
    title: str
    author_name: str


@dataclass
class Member:
    id: int | None
    name: str
    is_active: bool = True


@dataclass
class Loan:
    id: int | None
    member_id: int
    book_id: int
    borrowed_on: date
    due_on: date
    returned_on: date | None = None
    fine_amount: Decimal | None = None

    def is_active(self) -> bool:
        return self.returned_on is None

    def days_overdue(self, today: date) -> int:
        if not self.is_active():
            return 0
        return max(0, (today - self.due_on).days)

    def register_return(self, today: date) -> None:
        if not self.is_active():
            raise LoanAlreadyReturned(f"Wypożyczenie {self.id} już zwrócone")
        overdue_days = self.days_overdue(today)
        self.returned_on = today
        fine = overdue_days * FINE_PER_DAY
        self.fine_amount = fine if fine > 0 else None

    @classmethod
    def open_for(
        cls,
        *,
        member: Member,
        book: Book,
        active_loans: int,
        book_is_taken: bool,
        today: date,
    ) -> "Loan":
        if not member.is_active:
            raise MemberNotActive(f"Członek {member.name} jest nieaktywny")
        if active_loans >= MAX_ACTIVE_LOANS:
            raise LoanLimitExceeded(f"Limit {MAX_ACTIVE_LOANS} wypożyczeń przekroczony")
        if book_is_taken:
            raise BookNotAvailable(f"Książka {book.title!r} jest już wypożyczona")
        return cls(
            id=None,
            member_id=member.id if member.id is not None else -1,
            book_id=book.id if book.id is not None else -1,
            borrowed_on=today,
            due_on=today + timedelta(days=LOAN_PERIOD_DAYS),
        )
```

```python
# examples/sol_03_uow.py
"""Rozwiązanie 3 — minimalna jednostka pracy."""
from __future__ import annotations

from typing import Self

from sqlalchemy.orm import Session, sessionmaker

from sol_03_repositories import (
    SqlAlchemyBookRepository,
    SqlAlchemyLoanRepository,
    SqlAlchemyMemberRepository,
)


class SqlAlchemyUnitOfWork:
    def __init__(self, session_factory: sessionmaker[Session]) -> None:
        self._session_factory = session_factory
        self._session: Session | None = None

    def __enter__(self) -> Self:
        self._session = self._session_factory()
        self.books = SqlAlchemyBookRepository(self._session)
        self.members = SqlAlchemyMemberRepository(self._session)
        self.loans = SqlAlchemyLoanRepository(self._session)
        return self

    def __exit__(self, exc_type, exc, tb) -> None:
        assert self._session is not None
        try:
            if exc_type is not None:
                self._session.rollback()
        finally:
            self._session.close()

    def commit(self) -> None:
        assert self._session is not None
        self._session.commit()
```

**Komentarz do pomiaru.** Po refaktoryzacji zbiór testów dzieli się na dwie klasy:

| Zbiór | Przed refaktoryzacją | Po refaktoryzacji |
|---|---|---|
| Testy reguł (R1–R5) | 22 testy, ~1,8 s (baza + tabele per test) | 22 testy, ~0,05 s (obiekty w pamięci) |
| Testy repozytoriów | nie istniały osobno | 6 testów, ~0,5 s (integracja) |
| Razem | ~1,8 s | ~0,55 s |

Trzykrotne przyspieszenie to efekt uboczny. Prawdziwy zysk jest inny: **test reguły biznesowej przestał wymagać wiedzy o bazie.** Osoba czytająca `test_fine_is_charged_per_day_overdue` widzi cztery linie bez żadnego `fixture`, `Session`, `create_all` i `commit` — i od razu wie, co jest sprawdzane. Trudno to zmierzyć w sekundach, łatwo w minutach wgryzania się w cudzy kod.

---

## Najczęstsze błędy i jak je czytać

| Komunikat / objaw | Przyczyna | Naprawa |
|---|---|---|
| `DetachedInstanceError` przy serializacji DTO | Encja ORM wyszła z sesji (`expire_on_commit`), a DTO dotyka relacji leniwej | Jawnie doładuj relację (`selectinload`) albo zbuduj DTO z projekcji zapytania; nie zwracaj encji do API |
| `sqlalchemy.orm.exc.UnmappedInstanceError: Class 'Book' is not mapped` | Moduł z mapowaniem imperatywnym nie został zaimportowany | Wywołaj `start_mappers()` w punkcie wejścia aplikacji (`main.py`, `conftest.py`) albo importuj `orm.py` w module z UoW |
| `ImportError: cannot import name 'Loan' from partially initialized module` | Cykl importów: `domain` ↔ `infrastructure` | Ustal kierunek zależności z sekcji 2; użyj `TYPE_CHECKING`, `from __future__ import annotations`, albo przenieś import do wnętrza funkcji |
| `ArgumentError: Could not determine relationship direction` przy mapowaniu imperatywnym | Dwie kolumny klucza obcego do tej samej tabeli i brak `foreign_keys` | W `properties` relacji podaj jawnie `relationship(..., foreign_keys=[table.c.x])` |
| `TypeError: unhashable type: 'EmailAddress'` | `@dataclass` z `eq=True` (domyślnie) bez `frozen=True` ustawia `__hash__ = None` | Dodaj `frozen=True` do value objectu |
| `TypeError: cannot create weak reference to 'Book' object` (lub brak instrumentacji) | `@dataclass(slots=True)` na klasie mapowanej | `slots=True` tylko dla VO/DTO; encje mapowane zostaw bez `slots` |
| Dwa razy ta sama encja w pamięci, dziwne nadpisywanie | Dwa repozytoria z **osobnymi** sesjami | Jedna sesja w jednostce pracy, repozytoria dzielą ją |
| `sqlalchemy.exc.InvalidRequestError: This session is in 'prepared' state` | Wyjątek w środku transakcji bez `rollback()` | Używaj `with uow:`; `__exit__` musi wołać `rollback()` przy wyjątku |
| `IntegrityError: UNIQUE constraint failed` przeciekający do API jako 500 | Brak mapowania wyjątków technicznych na aplikacyjne | Handler wyjątków w warstwie prezentacji + tłumaczenie w warstwie aplikacji; nigdy nie przeciekaj `IntegrityError` |
| `pydantic.ValidationError: Field required: author_name` | DTO wymaga pola, którego nie ma w encji ani w projekcji | Dodaj pole do encji, wypełnij z relacji (`selectinload`) albo dostarczaj DTO z zapytania z `join` |
| `AttributeError: 'Book' object has no attribute '_sa_instance_state'` | Obiekt domenowy (value object / niemapowana klasa) przekazany do `session.add()` | Sprawdź, czy klasa jest zmapowana; VO zapisuj przez `TypeDecorator`, nie przez `add` |
| Nieskończone rekurencje / ogromny JSON w odpowiedzi | Encja ORM z relacjami zwrócona bezpośrednio, Pydantic/`json.dumps` chodzi po cyklu `Book → Author → books → ...` | Zwracaj DTO, nie encje; wyłącz `back_populates` z serializacji przez `exclude` |
| `DetachedInstanceError` tylko w testach | Sesja zamknięta w `fixture` przed asercjami na relacjach | Zwracaj z fixture dane już doładowane albo użyj `expire_on_commit=False` w sesji testowej |

---

## Słowniczek modułu

| Termin (EN) | Polski | Wyjaśnienie |
|---|---|---|
| Layer | Warstwa | Poziom odpowiedzialności w aplikacji: prezentacja, aplikacja, domena, infrastruktura |
| Dependency rule | Reguła zależności | Importy wskazują do wnętrza (w stronę domeny); domena nie zna infrastruktury |
| Domain model | Model domenowy | Opis pojęć i reguł biznesowych, niezależny od technologii |
| ORM model | Model ORM | Opis tabel i relacji wyrażony klasami Pythona |
| Data Mapper | Odwzorowanie danych | Wzorzec: obiekty nie znają bazy, osobna warstwa tłumaczy obiekty ↔ wiersze |
| Active Record | Rekord aktywny | Wzorzec: obiekt sam się zapisuje (`obj.save()`), klasa dziedziczy po ORM |
| Registry | Rejestr | Centralny spis mapowań klas do tabel (`registry`) |
| Imperative mapping | Mapowanie imperatywne | Mapowanie istniejącej klasy przez `map_imperatively`, bez `DeclarativeBase` |
| `map_imperatively` | — | Metoda rejestru łącząca klasę z tabelą (`Table`, `Join` albo `Select`) |
| Anemic model | Model anemiczny | Encja z samymi danymi, bez metod; reguły żyją w serwisach |
| Rich model | Model bogaty | Encja z metodami pilnującymi niezmienników |
| Value object | Obiekt wartości | Niemutowalny obiekt bez tożsamości, równość po wartości (`@dataclass(frozen=True)`) |
| Aggregate | Agregat | Grupa obiektów zmienianych razem, z korzeniem jako jedynym punktem wejścia |
| Aggregate root | Korzeń agregatu | Obiekt, do którego adresowane są operacje na całym agregacie |
| Invariant | Niezmiennik | Warunek, który musi być zawsze prawdziwy (np. `returned_on` tylko raz) |
| DTO | Obiekt transferu danych | Klasa opisująca kontrakt wejścia/wyjścia; nie jest encją ORM |
| Read model | Model odczytu | Kształt danych zoptymalizowany pod odczyt (raport), nie pod zapis |
| `TypeDecorator` | Dekorator typu | Klasa tłumacząca własny typ Pythona na typ zrozumiały dla bazy |
| Unit of Work | Jednostka pracy | Obiekt grupujący zmiany i wyznaczający granicę transakcji |
| Transaction boundary | Granica transakcji | Moment, w którym zmiany są zatwierdzane albo wycofywane razem |
| Port | Port | Interfejs (`Protocol`) definiowany przez warstwę aplikacji, implementowany przez infrastrukturę |
| Adapter | Adapter | Konkretna implementacja portu (np. repozytorium SQLAlchemy) |
| Outbox | Skrzynka nadawcza | Zapis „do wysłania” w tej samej transakcji, wysyłka po commicie |
| `sqlalchemy.orm.inspect` | — | Funkcja zwracająca obiekt `InstanceState`/`Mapper` dla instancji lub klasy |

---

## Dalsze czytanie

**SQLAlchemy (wersja 2.0 — dokumentacja, na której bazuje kurs):**

- Mapowanie deklaratywne i imperatywne: <https://docs.sqlalchemy.org/en/20/orm/mapping_styles.html>
- API mapowania (`registry`, `Mapper`, `map_imperatively`): <https://docs.sqlalchemy.org/en/20/orm/mapping_api.html>
- Mapowanie dataclass: <https://docs.sqlalchemy.org/en/20/orm/dataclasses.html>
- Podstawy sesji (cykl życia, flush, commit): <https://docs.sqlalchemy.org/en/20/orm/session_basics.html>
- Transakcje i granice transakcji: <https://docs.sqlalchemy.org/en/20/orm/session_transaction.html>
- Własne typy (`TypeDecorator`): <https://docs.sqlalchemy.org/en/20/core/custom_types.html>
- Typowanie w SQLAlchemy: <https://docs.sqlalchemy.org/en/20/orm/extensions/mypy.html>

**Wzorce projektowe:**

- Data Mapper (Martin Fowler): <https://martinfowler.com/eaaCatalog/dataMapper.html>
- Unit of Work: <https://martinfowler.com/eaaCatalog/unitOfWork.html>
- Repository: <https://martinfowler.com/eaaCatalog/repository.html>
- *Architecture Patterns with Python* (książka dostępna bezpłatnie online): <https://www.cosmicpython.com/book/preface.html>
- *Patterns of Enterprise Application Architecture* — katalog wzorców, źródło pojęć użytych w tym module

**Pydantic v2:**

- Modele i `from_attributes`: <https://docs.pydantic.dev/latest/concepts/models/>

---

## Co dalej

Wiesz już, gdzie w architekturze mieści się SQLAlchemy i jak rozdzielić domenę od infrastruktury. Kolejny krok to **konkretna implementacja wzorca repozytorium**: klasy, generyki, protokoły, paginacja keyset, projekcje DTO, tłumaczenie wyjątków technicznych na aplikacyjne — oraz uczciwa dyskusja, kiedy repozytoria w projektach SQLAlchemy są pomocne, a kiedy są nadmiarem.

➡️ **[`20_repository.md`](20_repository.md)** — Wzorzec Repository

<!-- koniec modułu 19 -->