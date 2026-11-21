# Zadání projektu: REST API pro databázi knih (`books_server`)

## 1. Úvod a cíl projektu

Cílem tohoto projektu je navrhnout a naimplementovat backendovou REST API aplikaci v jazyce **Python** s využitím frameworků **Django** a **Django REST Framework (DRF)**.

Aplikace bude sloužit jako **elektronický katalog knih a jejich autorů**. Architektura vychází z vícevrstvého návrhu (oddělení datových modelů, serializerů, pohledů/controllerů a servisní vrstvy s business logikou).

### Co si na projektu procvičíte:
- Návrh relačních datových modelů (`Author`, `Book`), indexování a výčtové typy (`TextChoices`).
- Práci s **podmíněným unikátním indexem** (`UniqueConstraint`) a konceptem **soft-delete** (skrývání záznamů místo jejich fyzického mazání).
- Tvorbu vlastních serializerů v DRF (vnořené objekty při čtení, transformace vstupních dat při zápisu, vlastní validace).
- Filtrování a vyhledávání pomocí `django-filter` (`FilterSet`) a `SearchFilter`.
- Optimalizaci databázových dotazů a prevenci **N+1 problému** pomocí `select_related`.
- Oddělení výpočetní a agregační logiky do **servisní vrstvy** (`services/`).
- Psaní automatizovaných **integračních testů** (`APITestCase`).

---

## 2. Doporučená architektura a struktura projektu

Projekt rozdělte do přehledné adresářové struktury, kde jsou pohledy (`views/`) a doménová logika (`services/`) odděleny do samostatných modulů:

```text
books_server/
├── library/                        # Hlavní aplikace databáze knih
│   ├── migrations/                 # Databázové migrace
│   ├── services/                   # Servisní vrstva (agregace, statistiky, business logika)
│   │   ├── __init__.py
│   │   ├── author_service.py       # Statistiky autorů (počet knih, celková hodnota knih)
│   │   └── book_service.py         # Souhrnné statistiky knihovny
│   ├── views/                      # Prezentační vrstva (DRF ViewSety a APIViews)
│   │   ├── __init__.py
│   │   ├── author_views.py         # CRUD pro autory, soft-delete, knihy autora
│   │   └── book_views.py           # CRUD pro knihy, filtrování a statistiky
│   ├── models.py                   # Modely Author, Book a výčet Genres
│   ├── serializer.py               # Serializery AuthorSerializer a BookSerializer
│   ├── tests.py                    # Integrační testy REST API
│   └── urls.py                     # Routování API endpointů
├── books_server/                   # Konfigurace Django projektu
│   ├── settings.py
│   └── urls.py
├── manage.py
├── requirements.txt
└── README.md
```

---

## 3. Specifikace datových modelů

### A. Výčtový typ `Genres` (Žánry knih)
Vytvořte třídu dědící z `models.TextChoices`:
- `FANTASY` (`'FANTASY'`, `'Fantasy'`)
- `SCIFI` (`'SCIFI'`, `'Sci-Fi'`)
- `DETECTIVE` (`'DETECTIVE'`, `'Detective'`)
- `HISTORY` (`'HISTORY'`, `'History'`)
- `EDUCATIONAL` (`'EDUCATIONAL'`, `'Educational'`)

### B. Model `Author` (Autor)
Reprezentuje autora knih. Záznamy autorů se z databáze fyzicky nemažou, ale skrývají pomocí příznaku `hidden`.

| Pole | Typ v Djangu | Požadavky a popis |
| :--- | :--- | :--- |
| `name` | `CharField(max_length=100)` | Jméno a příjmení autora (`db_index=True`) |
| `authorCode` | `CharField(max_length=50)` | Registrační kód autora (unikátní pouze pro `hidden=False`) |
| `birthDate` | `DateField()` | Datum narození autora |
| `country` | `CharField(max_length=50)` | Země původu autora |
| `mail` | `EmailField()` | Kontaktní e-mail autora (`blank=True, null=True`) |
| `biography` | `TextField()` | Stručný životopis nebo poznámka (`blank=True, null=True`) |
| `hidden` | `BooleanField(default=False)` | Příznak skrytí záznamu (`db_index=True`) |

> **Důležité (Podmíněný unikátní index):**
> V `class Meta` modelu `Author` definujte `UniqueConstraint` nad polem `authorCode` s podmínkou `condition=models.Q(hidden=False)`. To zajistí, že nemohou existovat dva aktivní autoři se stejným kódem, ale po skrytí autora (`hidden=True`) lze vytvořit nový aktivní záznam se stejným `authorCode`.

### C. Model `Book` (Kniha)
Reprezentuje konkrétní knihu v databázi.

| Pole | Typ v Djangu | Požadavky a popis |
| :--- | :--- | :--- |
| `isbn` | `CharField(max_length=20)` | Unikátní kód ISBN knihy (`unique=True`) |
| `title` | `CharField(max_length=255)` | Název knihy (`db_index=True`) |
| `author` | `ForeignKey(Author)` | Vazba na autora (`related_name="books"`, `on_delete=models.CASCADE` nebo `PROTECT`) |
| `genre` | `CharField(max_length=20)` | Žánr knihy (`choices=Genres.choices`, `default=Genres.FANTASY`, `db_index=True`) |
| `publishedDate` | `DateField()` | Datum vydání knihy (`db_index=True`) |
| `pages` | `PositiveIntegerField()` | Počet stran |
| `price` | `DecimalField(max_digits=10, decimal_places=2)` | Cena knihy (`db_index=True`) |
| `description` | `TextField()` | Anotace / popis knihy (`blank=True, null=True`) |

---

## 4. Stručný postup tvorby aplikace (Krok za krokem)

### Krok 1: Inicializace projektu a prostředí
1. Vytvořte virtuální prostředí (`python -m venv .venv`), aktivujte jej a nainstalujte závislosti:
   ```powershell
   pip install django djangorestframework django-filter django-cors-headers
   ```
2. Vytvořte Django projekt `books_server` a aplikaci `library`:
   ```powershell
   django-admin startproject books_server .
   python manage.py startapp library
   ```
3. V `settings.py` zaregistrujte `library`, `rest_framework`, `django_filters` a `corsheaders` do `INSTALLED_APPS` a přidejte `CorsMiddleware` do `MIDDLEWARE`.

### Krok 2: Tvorba modelů a migrace databáze
1. V `library/models.py` naimplementujte třídy `Genres`, `Author` a `Book`.
2. Vygenerujte a aplikujte migrace:
   ```powershell
   python manage.py makemigrations
   python manage.py migrate
   ```

### Krok 3: Implementace serializerů (`library/serializer.py`)
1. **`AuthorSerializer`**:
   - Přidejte pole `_id = serializers.IntegerField(source="id", read_only=True)`.
   - V metodě `validate(self, attrs)` ověřte, že v databázi neexistuje jiný aktivní autor (`hidden=False`) se stejným `authorCode`. Pokud ano, vyhoďte `serializers.ValidationError`.
2. **`BookSerializer`**:
   - Přidejte pole `_id = serializers.IntegerField(source="id", read_only=True)`.
   - Přepište `to_internal_value(self, data)`: pokud klient pošle pole `author` jako slovník `{"_id": 5}`, převeďte jej před validací na primární klíč `5` (pracujte nad kopií `data.copy()`).
   - Přepište `to_representation(self, instance)`: při čtení knihy nahraďte ID autora plně serializovaným objektem `AuthorSerializer(instance.author).data`.

### Krok 4: Implementace servisní vrstvy (`library/services/`)
1. **`BookService` (`book_service.py`)**:
   - Metoda `calculate_statistics()` spočítá pomocí Django ORM agregací (`Sum`, `Count`, `Avg`):
     - `allTimePriceSum`: celková cena všech knih v databázi (pokud žádné nejsou, vrátí `0`),
     - `currentYearPriceSum`: celková cena knih vydaných v aktuálním kalendářním roce (`publishedDate__year`),
     - `booksCount`: celkový počet knih v databázi.
2. **`AuthorService` (`author_service.py`)**:
   - Metoda `get_author_statistics()` vrátí seznam statistik pro všechny aktivní autory (`hidden=False`) pomocí `annotate`:
     - `authorId`, `authorName`, `booksCount` (počet napsaných knih) a `totalBooksValue` (součet cen jeho knih nebo `0`).

### Krok 5: Implementace pohledů a filtrů (`library/views/`)
1. **`AuthorViewSet` (`author_views.py`)**:
   - Výchozí `queryset = Author.objects.filter(hidden=False)`.
   - Přepište metodu `destroy`: místo smazání nastavte `instance.hidden = True` a uložte (`HTTP 204`).
   - Přepište metodu `update`: skryjte původní záznam (`instance.hidden = True`) a vytvořte nový aktivní záznam s upravenými daty (`HTTP 200`).
   - Přidejte `@action(detail=False, methods=['get'], url_path='statistics')`, která zavolá `AuthorService.get_author_statistics()`.
2. **`AuthorBooksView` (`author_views.py`)**:
   - Třída dědící z `ListAPIView`, která pro zadaný `authorCode` v URL vrátí všechny knihy daného autora.
3. **`BookFilter` a `BookViewSet` (`book_views.py`)**:
   - Vytvořte `BookFilter(django_filters.FilterSet)` podporující parametry: `authorID`, `genre`, `title` (`icontains`), `minPrice` (`gte`), `maxPrice` (`lte`) a `limit`.
   - V `BookViewSet` nastavte `queryset = Book.objects.select_related('author').all()` (eliminace N+1 dotazů!) a přidejte `@action(detail=False, methods=['get'], url_path='statistics')`.

### Krok 6: Routování (`library/urls.py` a `books_server/urls.py`)
1. Zaregistrujte `AuthorViewSet` (`authors`) a `BookViewSet` (`books`) do `DefaultRouter(trailing_slash=False)`.
2. Přidejte URL cestu `api/identification/<str:authorCode>/books/` pro `AuthorBooksView`.

### Krok 7: Testování (`library/tests.py`)
1. Napište integrační testy pomocí `APITestCase` pokrývající všechny případy užití uvedené níže.
2. Spusťte testy příkazem `python manage.py test`.

---

## 5. Přehled Use Cases (Případů užití)

### UC-01: Seznam a detail aktivních autorů
- **Aktér:** Uživatel / Frontendová aplikace
- **Endpointy:**
  - `GET /api/authors`
  - `GET /api/authors/<id>`
- **Popis:**
  - Systém vrátí seznam všech aktivních autorů (`hidden=False`), resp. detail konkrétního aktivního autora.
  - Pokud je požadován detail autora, který má `hidden=True` nebo neexistuje, systém vrátí `404 Not Found`.

### UC-02: Vytvoření nového autora
- **Aktér:** Uživatel
- **Endpoint:** `POST /api/authors`
- **Vstupní JSON (příklad):**
  ```json
  {
    "name": "Karel Čapek",
    "authorCode": "AUT-001",
    "birthDate": "1890-01-09",
    "country": "Czechia",
    "mail": "karel@capek.cz",
    "biography": "Český spisovatel, dramatik a novinář."
  }
  ```
- **Popis a pravidla:**
  - Systém zkontroluje, zda již neexistuje aktivní autor (`hidden=False`) se stejným `authorCode`.
  - Pokud aktivní autor s tímto kódem existuje, systém vrátí `400 Bad Request` s chybovou hláškou u pole `authorCode`.
  - Pokud kód patří pouze skrytému autorovi (`hidden=True`) nebo je zcela nový, systém vytvoří záznam a vrátí `201 Created` včetně vygenerovaného `_id`.

### UC-03: Úprava autora se zachováním historie (Soft-Update)
- **Aktér:** Uživatel
- **Endpoint:** `PUT /api/authors/<id>`
- **Popis:**
  - Při úpravě autora systém nejprve zvaliduje vstupní data.
  - Původní záznam autora označí jako skrytý (`hidden = True`).
  - Vytvoří nový záznam autora s `hidden = False` a novým `_id`.
  - Vrátí `200 OK` s daty nově vytvořeného záznamu.

### UC-04: Odstranění (skrytí) autora (Soft-Delete)
- **Aktér:** Uživatel
- **Endpoint:** `DELETE /api/authors/<id>`
- **Popis:**
  - Systém záznam z databáze fyzicky nemaže (aby zůstaly zachovány vazby na historické knihy), pouze nastaví `hidden = True`.
  - Vrátí stavový kód `204 No Content`.

### UC-05: Kompletní CRUD správa knih
- **Aktér:** Uživatel
- **Endpointy:**
  - `GET /api/books` – seznam všech knih (včetně vnořeného objektu `author`)
  - `POST /api/books` – vytvoření nové knihy
  - `GET /api/books/<id>` – detail knihy
  - `PUT /api/books/<id>` – úprava knihy
  - `DELETE /api/books/<id>` – fyzické smazání knihy z databáze
- **Vstupní JSON pro `POST /api/books` (příklad):**
  ```json
  {
    "isbn": "978-80-204-2000-1",
    "title": "R.U.R.",
    "author": { "_id": 1 },
    "genre": "SCIFI",
    "publishedDate": "1920-01-01",
    "pages": 120,
    "price": "299.00",
    "description": "Kolektivní drama o robotech."
  }
  ```
- **Výstupní JSON (příklad):**
  - V odpovědi (`200 OK` / `201 Created`) musí pole `author` obsahovat kompletní serializovaný objekt autora (nikoliv jen jeho ID).

### UC-06: Filtrování a vyhledávání knih
- **Aktér:** Uživatel
- **Endpoint:** `GET /api/books`
- **Podporované query parametry:**
  - `authorID` – filtrování knih podle ID autora (`?authorID=1`)
  - `genre` – filtrování podle žánru (`?genre=SCIFI`)
  - `title` – vyhledávání podle podřetězce v názvu knihy bez ohledu na velikost písmen (`?title=r.u.r`)
  - `minPrice` – knihy s cenou větší nebo rovnou zadané hodnotě (`?minPrice=200`)
  - `maxPrice` – knihy s cenou menší nebo rovnou zadané hodnotě (`?maxPrice=500`)
  - `limit` – omezení maximálního počtu vrácených záznamů (`?limit=5`)

### UC-07: Výpis všech knih podle registračního kódu autora
- **Aktér:** Uživatel
- **Endpoint:** `GET /api/identification/<authorCode>/books/`
- **Popis:**
  - Vyhledá aktivního autora podle `authorCode` (pokud neexistuje, vrátí `404 Not Found`).
  - Vrátí seznam všech knih, které daný autor napsal.

### UC-08: Souhrnné statistiky knihovny
- **Aktér:** Uživatel / Dashboard
- **Endpoint:** `GET /api/books/statistics`
- **Výstupní JSON (příklad):**
  ```json
  {
    "allTimePriceSum": 1450.00,
    "currentYearPriceSum": 599.00,
    "booksCount": 4
  }
  ```

### UC-09: Statistiky produkce autorů
- **Aktér:** Uživatel / Dashboard
- **Endpoint:** `GET /api/authors/statistics`
- **Výstupní JSON (příklad):**
  ```json
  [
    {
      "authorId": 1,
      "authorName": "Karel Čapek",
      "booksCount": 3,
      "totalBooksValue": 890.00
    },
    {
      "authorId": 2,
      "authorName": "Božena Němcová",
      "booksCount": 0,
      "totalBooksValue": 0
    }
  ]
  ```

---

## 6. Kritéria hodnocení (Definition of Done)

Před odevzdáním si zkontrolujte, že váš projekt splňuje následující body:

1. **Čistá architektura**: Pohledy (`views/`) neobsahují složité agregační dotazy – výpočty statistik jsou vyčleněny v `services/`.
2. **Optimalizace dotazů (N+1)**: Výpis knih používá `select_related('author')`, takže načtení seznamu 50 knih vygeneruje pouze 1 SQL dotaz (s `JOIN`), nikoliv 51 dotazů.
3. **Datová integrita**: Unikátnost `authorCode` pro aktivní autory je vynucena jak na úrovni serializeru (přátelská chybová hláška `400 Bad Request`), tak na úrovni databáze (`UniqueConstraint`).
4. **Integrační testy (`tests.py`)**: Projekt obsahuje alespoň 10 integračních testů (`APITestCase`), které ověřují CRUD operace, soft-delete, filtrování a statistické endpointy.
5. **Dokumentace (`README.md`)**: Projekt obsahuje přehledný návod na instalaci, spuštění a popis API endpointů.
