# Zadání projektu: REST API pro databázi knih v Django Ninja (`books_ninja_server`)

## 1. Úvod: Co je Django Ninja a proč ho používat?

V tomto projektu vytvoříte stejnou aplikaci **elektronického katalogu knih a autorů** jako v předchozím zadání, ale místo *Django REST Frameworku (DRF)* použijete moderní framework **Django Ninja** (`django-ninja`).

Pokud jste s **Django Ninja** ještě nikdy nepracovali, vůbec to nevadí – toto zadání je napsané jako **podrobný průvodce krok za krokem**, který vám vysvětlí všechny principy od nuly.

### V čem se Django Ninja liší od Django REST Frameworku (DRF)?
Django Ninja je silně inspirovaný populárním frameworkem **FastAPI**, ale běží přímo nad **Django ORM** a celým ekosystémem Djanga:
1. **Typové anotace (Python Type Hints)**: Místo složitého skládání tříd v DRF napíšete běžnou Python funkci a pomocí typů (`author_id: int`, `payload: AuthorIn`) řeknete frameworku, jaká data očekáváte. Django Ninja vstup automaticky zvaliduje a převede.
2. **Pydantic Schémata místo Serializerů**: V DRF se používá `ModelSerializer`, kde se často míchá čtení i zápis v jedné třídě. V Django Ninja se používají třídy dědící ze `Schema` (postavené na knihovně **Pydantic**), přičemž **důsledně oddělujeme vstupní schéma (`In`) od výstupního schématu (`Out`)**.
3. **Routery a funkce místo `ViewSet`ů**: Místo”magických” tříd jako `ModelViewSet`, kde je těžké dohledat, která metoda se kdy volá, píšete v Django Ninja přehledné funkce s dekorátory `@router.get(...)`, `@router.post(...)`, `@router.put(...)` a `@router.delete(...)`.
4. **Automatická interaktivní dokumentace (Swagger UI)**: Ihned po spuštění serveru najdete na adrese `http://127.0.0.1:8000/api/docs` kompletní interaktivní webovou dokumentaci, kde můžete všechny své endpointy přímo klikáním testovat.

### Slovníček pojmů: Přechod z DRF na Django Ninja

| Koncept | Django REST Framework (DRF) | Django Ninja |
| :--- | :--- | :--- |
| **Serializace a validace** | `serializers.ModelSerializer` | `ninja.Schema` / `ninja.ModelSchema` (Pydantic) |
| **Skupina endpointů** | `viewsets.ModelViewSet` / `APIView` | `ninja.Router` + dekorátory `@router.get`, `@router.post` |
| **Filtrování z URL** | `django_filters.FilterSet` | `ninja.FilterSchema` + `Query(...)` |
| **Chybová odpověď** | `raise ValidationError(...)` | `raise HttpError(400, "Zpráva")` nebo stavový tuple `return 400, {...}` |
| **Testovací klient** | `rest_framework.test.APITestCase` | `django.test.TestCase` + `ninja.testing.TestClient` |

---

## 2. Doporučená architektura a struktura projektu

Stejně jako v dobře navrženém DRF projektu oddělíme **prezentační API vrstvu (`routers/`)**, **datové kontrakty (`schemas.py`)**, **doménovou logiku (`services/`)** a **databázové modely (`models.py`)**:

```text
books_ninja_server/
├── library/                        # Hlavní aplikace databáze knih
│   ├── migrations/                 # Databázové migrace
│   ├── routers/                    # Prezentační vrstva (Django Ninja Routery = obdoba views/)
│   │   ├── __init__.py
│   │   ├── author_router.py        # Endpointy pro autory (/api/authors/...)
│   │   └── book_router.py          # Endpointy pro knihy (/api/books/...)
│   ├── services/                   # Servisní vrstva (business logika, DB transakce, statistiky)
│   │   ├── __init__.py
│   │   ├── author_service.py       # Operace nad autory, soft-delete, statistiky autorů
│   │   └── book_service.py         # Operace nad knihami, souhrnné statistiky knihovny
│   ├── api.py                      # Hlavní instance NinjaAPI, která spojuje routery
│   ├── models.py                   # Django ORM modely Author, Book a výčet Genres
│   ├── schemas.py                  # Pydantic schémata (vstupní, výstupní a filtrační)
│   └── tests.py                    # Integrační testy pomocí TestClient
├── books_ninja_server/             # Konfigurace Django projektu
│   ├── settings.py
│   └── urls.py                     # Napojení api.urls na cestu /api/
├── manage.py
├── requirements.txt
└── README.md
```

---

## 3. Podrobný postup tvorby aplikace (Krok za krokem)

### Krok 1: Inicializace projektu a instalace Django Ninja

1. **Vytvořte složku projektu, virtuální prostředí a aktivujte jej:**
   ```powershell
   mkdir books_ninja_server
   cd books_ninja_server
   python -m venv .venv
   .\.venv\Scripts\Activate.ps1
   ```

2. **Nainstalujte potřebné balíčky:**
   Všimněte si, že nepotřebujete `djangorestframework` ani `django-filter` – obojí zastane `django-ninja`.
   ```powershell
   pip install django django-ninja django-cors-headers
   pip freeze > requirements.txt
   ```

3. **Vytvořte Django projekt a aplikaci `library`:**
   ```powershell
   django-admin startproject books_ninja_server .
   python manage.py startapp library
   ```

4. **Upravte `books_ninja_server/settings.py`:**
   Do `INSTALLED_APPS` přidejte `'library'` a `'corsheaders'`. Do `MIDDLEWARE` přidejte `'corsheaders.middleware.CorsMiddleware'` a povolte `CORS_ALLOW_ALL_ORIGINS = True`.

---

### Krok 2: Datové modely (`library/models.py`)

**Dobrá zpráva:** Modely v Django Ninja jsou **100% standardní Django ORM modely**. Nic speciálního se zde nemění.

#### Specifikace modelů:
1. **`Genres(models.TextChoices)`** – výčet žánrů: `FANTASY`, `SCIFI`, `DETECTIVE`, `HISTORY`, `EDUCATIONAL`.
2. **`Author(models.Model)`**:
   - `name = models.CharField(max_length=100, db_index=True)`
   - `authorCode = models.CharField(max_length=50)` – registrační kód autora
   - `birthDate = models.DateField()`
   - `country = models.CharField(max_length=50)`
   - `mail = models.EmailField(blank=True, null=True)`
   - `biography = models.TextField(blank=True, null=True)`
   - `hidden = models.BooleanField(default=False, db_index=True)` – příznak pro **soft-delete**
   - V `class Meta:` přidejte podmíněný unikátní index, aby `authorCode` musel být unikátní pouze mezi **aktivními** (`hidden=False`) autory:
     ```python
     class Meta:
         constraints = [
             models.UniqueConstraint(
                 fields=['authorCode'],
                 condition=models.Q(hidden=False),
                 name='unique_active_author_code',
             )
         ]
     ```
3. **`Book(models.Model)`**:
   - `isbn = models.CharField(max_length=20, unique=True)`
   - `title = models.CharField(max_length=255, db_index=True)`
   - `author = models.ForeignKey(Author, related_name="books", on_delete=models.PROTECT)`
   - `genre = models.CharField(max_length=20, choices=Genres.choices, default=Genres.FANTASY, db_index=True)`
   - `publishedDate = models.DateField(db_index=True)`
   - `pages = models.PositiveIntegerField()`
   - `price = models.DecimalField(max_digits=10, decimal_places=2, db_index=True)`
   - `description = models.TextField(blank=True, null=True)`

Po dopsání modelů nezapomeňte vytvořit a spustit migrace:
```powershell
python manage.py makemigrations
python manage.py migrate
```

---

### Krok 3: Tvorba Pydantic schémat (`library/schemas.py`)

Zde se poprvé setkáte s **Django Ninja**. Místo `serializer.py` vytvoříte soubor `library/schemas.py`.

#### Jak fungují schémata v Django Ninja?
Každé schéma je třída dědící z `ninja.Schema`.
- **Vstupní schéma (`...In`)** definuje, jaký JSON musí poslat klient v těle `POST` nebo `PUT` požadavku.
- **Výstupní schéma (`...Out`)** definuje, jaký JSON vrátí náš server klientovi (včetně `_id` nebo vnořených objektů).

#### 1. Jak mapovat databázové `id` na `_id` ve výstupním JSONu?
V našem API chceme, aby se primární klíč `id` v JSONu jmenoval `_id` (kvůli kompatibilitě s frontendem).
V Django Ninja k tomu slouží parametr `alias` u `Field`:
- `id: int = Field(..., alias="_id")` říká: *"Přečti z Django modelu atribut `obj.id`, ale při serializaci do JSONu (nebo čtení z JSONu) použij klíč `_id`"*.
- Aby Pydantic dovolil plnit pole `id` z atributu `obj.id` modelu i tehdy, když má nastavený `alias="_id"`, přidáme do schématu konfiguraci `model_config = ConfigDict(populate_by_name=True, from_attributes=True)`.

```python
from datetime import date
from decimal import Decimal
from typing import Optional
from ninja import Schema, FilterSchema, Field
from pydantic import ConfigDict
from .models import Genres


class ErrorOut(Schema):
    """Univerzální schéma pro vrácení chybové zprávy (např. HTTP 400 nebo 404)."""
    detail: str


# ==========================================
# SCHÉMATA PRO AUTORY (Author)
# ==========================================

class AuthorIn(Schema):
    """Vstupní schéma pro vytvoření nebo úpravu autora (POST / PUT)."""
    name: str
    authorCode: str
    birthDate: date
    country: str
    mail: Optional[str] = None
    biography: Optional[str] = None


class AuthorOut(Schema):
    """Výstupní schéma autora vracené klientovi."""
    model_config = ConfigDict(populate_by_name=True, from_attributes=True)

    id: int = Field(..., alias="_id")
    name: str
    authorCode: str
    birthDate: date
    country: str
    mail: Optional[str] = None
    biography: Optional[str] = None
    hidden: bool
```

#### 2. Jak elegantně vyřešit vnořeného autora u knihy (`{"author": {"_id": 1}}`)?
Vzpomeňte si, jak jste v DRF museli přepisovat metody `to_internal_value` a `to_representation`. V Django Ninja to vyřešíte čistě pomocí schémat:
- Pro **vstup (`BookIn`)** vytvoříme malé pomocné schéma `AuthorIdSchema`, které očekává objekt `{"_id": 1}`.
- Pro **výstup (`BookOut`)** uvedeme `author: AuthorOut` – Django Ninja automaticky vezme `book.author` a převede ho přes `AuthorOut`!

```python
# ==========================================
# SCHÉMATA PRO KNIHY (Book)
# ==========================================

class AuthorIdSchema(Schema):
    """Pomocné schéma pro zápis vazby na autora ve tvaru {"_id": 1}."""
    model_config = ConfigDict(populate_by_name=True)
    id: int = Field(..., alias="_id")


class BookIn(Schema):
    """Vstupní schéma pro vytvoření nebo úpravu knihy."""
    isbn: str
    title: str
    author: AuthorIdSchema
    genre: Genres = Genres.FANTASY
    publishedDate: date
    pages: int
    price: Decimal
    description: Optional[str] = None


class BookOut(Schema):
    """Výstupní schéma knihy s kompletně vnořeným objektem autora."""
    model_config = ConfigDict(populate_by_name=True, from_attributes=True)

    id: int = Field(..., alias="_id")
    isbn: str
    title: str
    author: AuthorOut
    genre: str
    publishedDate: date
    pages: int
    price: Decimal
    description: Optional[str] = None
```

#### 3. Jak funguje filtrování přes `FilterSchema`?
Místo instalace `django-filter` má Django Ninja vestavěnou třídu `FilterSchema`. Pomocí parametru `q="..."` přímo určíte, jaký Django ORM lookup se má použít, pokud uživatel daný query parametr v URL pošle (parametry, které mají hodnotu `None`, Django Ninja při filtrování automaticky přeskočí):

```python
class BookFilterSchema(FilterSchema):
    """Filtrační schéma pro query parametry GET /api/books."""
    authorID: Optional[int] = Field(None, q="author_id")
    genre: Optional[str] = Field(None, q="genre")
    title: Optional[str] = Field(None, q="title__icontains")
    minPrice: Optional[Decimal] = Field(None, q="price__gte")
    maxPrice: Optional[Decimal] = Field(None, q="price__lte")
```

#### 4. Schémata pro statistiky
V Django Ninja je dobrou praxí definovat výstupní schéma i pro statistické endpointy – zajistí to typovou kontrolu i automatické vygenerování dokumentace ve Swaggeru:

```python
class BookStatisticsOut(Schema):
    allTimePriceSum: Decimal
    currentYearPriceSum: Decimal
    booksCount: int


class AuthorStatisticsItemOut(Schema):
    authorId: int
    authorName: str
    booksCount: int
    totalBooksValue: Decimal
```

---

### Krok 4: Servisní vrstva (`library/services/`)

Do složky `library/services/` umístíme veškerou aplikační logiku a práci s databází. Routery pak budou pouze přijímat HTTP požadavky, volat metody ze servisní vrstvy a vracet odpověď.

#### A. `library/services/author_service.py`
Zde naimplementujte třídu `AuthorService`:
1. **`validate_unique_active_code(author_code: str, exclude_id: Optional[int] = None)`**:
   - Ověří, zda v DB neexistuje aktivní autor (`hidden=False`) se stejným `authorCode`.
   - Pokud ano, vyhodí výjimku `from ninja.errors import HttpError`:
     ```python
     raise HttpError(400, "Autor s tímto registračním kódem již existuje.")
     ```
2. **`create_author(payload: AuthorIn) -> Author`**:
   - Zavolá `validate_unique_active_code(payload.authorCode)`.
   - Vytvoří záznam pomocí `Author.objects.create(**payload.dict(), hidden=False)`. *(Pozn.: v Pydantic v2 lze použít i `payload.model_dump()`).*
3. **`update_author(author_id: int, payload: AuthorIn) -> Author`**:
   - Najde aktivního autora pomocí `get_object_or_404(Author, id=author_id, hidden=False)`.
   - Zavolá `validate_unique_active_code(payload.authorCode, exclude_id=author.id)` (důležité je `exclude_id=author.id`, aby bylo možné upravit autora i tehdy, když se jeho `authorCode` nemění!).
   - Uvnitř databázové transakce (`with transaction.atomic():`) nastaví původnímu záznamu `author.hidden = True`, uloží ho (`author.save(update_fields=["hidden"])`) a vytvoří nový záznam `Author.objects.create(**payload.dict(), hidden=False)`.
4. **`soft_delete_author(author_id: int) -> None`**:
   - Najde aktivního autora pomocí `get_object_or_404(Author, id=author_id, hidden=False)`.
   - Nastaví `author.hidden = True` a uloží.
5. **`get_author_statistics() -> list[dict]`**:
   - Pomocí `Author.objects.filter(hidden=False).annotate(...)` spočítá pro každého aktivního autora počet jeho knih (`Count('books')`) a celkovou hodnotu knih (`Sum('books__price')`).

#### B. `library/services/book_service.py`
Zde naimplementujte třídu `BookService`:
1. **`create_book(payload: BookIn) -> Book`**:
   - Ověří, že autor s `id=payload.author.id` existuje, případně vyhledá instanci `author = get_object_or_404(Author, id=payload.author.id)`.
   - Vytvoří knihu v DB.
2. **`update_book(book_id: int, payload: BookIn) -> Book`**:
   - Najde knihu (`get_object_or_404(Book, id=book_id)`), aktualizujte její atributy podle `payload` a uloží ji.
3. **`calculate_statistics() -> dict`**:
   - Pomocí podmíněné agregace v jediném SQL dotazu spočítá `allTimePriceSum`, `currentYearPriceSum` a `booksCount`.

---

### Krok 5: Tvorba Routerů (`library/routers/`)

Nyní přichází nejzajímavější část Django Ninja – tvorba samotných endpointů.

#### Jak fungují dekorátory a parametry funkcí v Django Ninja?
Podívejte se na následující pravidla, podle kterých Django Ninja pozná, odkud má vzít hodnotu argumentu funkce:
1. **`request`** (první argument každé view funkce): standardní Django `HttpRequest`.
2. **Parametr uvedený ve složených závorkách v URL** (např. `"/{author_id}"` -> `author_id: int`): Django Ninja ho automaticky přečte z cesty URL a převede na `int`.
3. **Parametr, jehož typem je třída dědící ze `Schema`** (např. `payload: AuthorIn`): Django Ninja ho automaticky přečte z **JSON těla (body)** požadavku a zvaliduje.
4. **Jednoduchý parametr s výchozí hodnotou** (např. `limit: Optional[int] = None`) nebo **parametr označený `= Query(...)`**: Django Ninja ho přečte z **URL query stringu** (např. `?limit=5`).

#### A. Ukázka `library/routers/author_router.py`

```python
from ninja import Router
from django.shortcuts import get_object_or_404
from ..models import Author, Book
from ..schemas import AuthorIn, AuthorOut, AuthorStatisticsItemOut
from ..services.author_service import AuthorService

author_router = Router(tags=["Authors"])


@author_router.get("", response=list[AuthorOut])
def list_authors(request):
    """Vrátí seznam všech aktivních autorů (hidden=False)."""
    return Author.objects.filter(hidden=False)


@author_router.get("/statistics", response=list[AuthorStatisticsItemOut])
def get_author_statistics(request):
    """Vrátí statistiky počtu a celkové ceny knih pro jednotlivé aktivní autory."""
    return AuthorService.get_author_statistics()


@author_router.get("/{author_id}", response=AuthorOut)
def get_author(request, author_id: int):
    """Vrátí detail konkrétního aktivního autora podle ID."""
    return get_object_or_404(Author, id=author_id, hidden=False)


@author_router.post("", response={201: AuthorOut})
def create_author(request, payload: AuthorIn):
    """Vytvoří nového autora a vrátí stavový kód 201 Created."""
    author = AuthorService.create_author(payload)
    return 201, author


@author_router.put("/{author_id}", response=AuthorOut)
def update_author(request, author_id: int, payload: AuthorIn):
    """Archivuje původní záznam autora (hidden=True) a vytvoří nový aktivní záznam."""
    return AuthorService.update_author(author_id, payload)


@author_router.delete("/{author_id}", response={204: None})
def delete_author(request, author_id: int):
    """Provede soft-delete autora (nastaví hidden=True) a vrátí 204 No Content."""
    AuthorService.soft_delete_author(author_id)
    return 204, None
```

> **Pozor na pořadí rout v souboru!**
> Všimněte si, že endpoint `@author_router.get("/statistics")` je definován **před** `@author_router.get("/{author_id}")`. Kdyby byl až za ním, Django by se pokusilo slovo `"statistics"` naparsovat jako `author_id: int` a vrátilo by chybu 422! Statické cesty vždy pište před dynamické parametry `{...}`.

#### B. Ukázka filtrování v `library/routers/book_router.py`

V `book_router.py` použijeme `BookFilterSchema` spolu s `Query(...)` a `select_related('author')`:

```python
from typing import Optional
from ninja import Router, Query
from django.shortcuts import get_object_or_404
from ..models import Book
from ..schemas import BookIn, BookOut, BookFilterSchema, BookStatisticsOut
from ..services.book_service import BookService

book_router = Router(tags=["Books"])


@book_router.get("", response=list[BookOut])
def list_books(
    request,
    filters: BookFilterSchema = Query(...),
    limit: Optional[int] = None,
):
    """
    Vrátí seznam knih s podporou filtrování (authorID, genre, title, minPrice, maxPrice)
    a volitelným omezením počtu záznamů (limit).
    """
    # select_related('author') vyřeší N+1 problém - načte knihu i autora jedním SQL JOIN dotazem
    queryset = Book.objects.select_related("author").all()
    queryset = filters.filter(queryset)

    if limit is not None and limit > 0:
        queryset = queryset[:limit]

    return queryset
```
*(Doplňte do `book_router.py` i zbývající endpointy: `GET /statistics`, `GET /{book_id}`, `POST ""`, `PUT /{book_id}` a `DELETE /{book_id}`).*

---

### Krok 6: Propojení Routerů v `library/api.py` a `books_ninja_server/urls.py`

1. **Vytvořte soubor `library/api.py`**:
   Zde vytvoříte hlavní instanci `NinjaAPI` a připojíte k ní oba routery i endpoint pro vyhledání knih podle registračního kódu autora:

   ```python
   from ninja import NinjaAPI
   from django.shortcuts import get_object_or_404
   from .models import Author, Book
   from .schemas import BookOut
   from .routers.author_router import author_router
   from .routers.book_router import book_router

   api = NinjaAPI(
       title="Book Database API",
       version="1.0.0",
       description="REST API pro správu knih a autorů postavené na Django Ninja",
   )

   api.add_router("/authors", author_router)
   api.add_router("/books", book_router)


   @api.get(
       "/identification/{authorCode}/books/",
       response=list[BookOut],
       tags=["Identification"],
   )
   def get_books_by_author_code(request, authorCode: str):
       """Vrátí všechny knihy autora se zadaným registračním kódem (authorCode)."""
       # Ověříme, že aktivní autor s tímto kódem existuje
       get_object_or_404(Author, authorCode=authorCode, hidden=False)
       # Vrátíme všechny knihy navázané na tento authorCode (včetně historických verzí autora)
       return Book.objects.select_related("author").filter(
           author__authorCode=authorCode
       )
   ```

2. **Zaregistrujte `api.urls` v `books_ninja_server/urls.py`**:
   ```python
   from django.urls import path
   from library.api import api

   urlpatterns = [
       path("api/", api.urls),
   ]
   ```

3. **Vyzkoušejte interaktivní Swagger dokumentaci!**
   Spusťte server:
   ```powershell
   python manage.py runserver
   ```
   Otevřete v prohlížeči adresu **`http://127.0.0.1:8000/api/docs`**. Uvidíte přehledně zdokumentované všechny své endpointy i schémata!

---

## 4. Přehled všech požadovaných Use Cases (Případů užití)

Vaše aplikace musí splňovat následujících 9 případů užití:

| ID | Název Use Case | HTTP Metoda a URL | Očekávaný HTTP kód | Popis chování |
| :--- | :--- | :--- | :--- | :--- |
| **UC-01** | Výpis a detail aktivních autorů | `GET /api/authors`<br>`GET /api/authors/{id}` | `200 OK`<br>`404 Not Found` | Vrací pouze autory s `hidden=False`. Pro skrytého autora vrátí `404`. |
| **UC-02** | Vytvoření nového autora | `POST /api/authors` | `201 Created`<br>`400 Bad Request` | Zvaliduje unikátnost `authorCode` mezi aktivními autory (`hidden=False`). Pokud kód patří skrytému autorovi, vytvoření projde. |
| **UC-03** | Úprava autora (Soft-Update) | `PUT /api/authors/{id}` | `200 OK`<br>`400` / `404` | V transakci skryje původní záznam (`hidden=True`) a vytvoří nový záznam s novým `_id`. Musí fungovat i tehdy, když se `authorCode` nemění! |
| **UC-04** | Skrytí autora (Soft-Delete) | `DELETE /api/authors/{id}` | `204 No Content` | Fyzicky záznam nemaže, pouze nastaví `hidden=True`. |
| **UC-05** | CRUD správa knih | `GET /api/books`<br>`POST /api/books`<br>`GET /api/books/{id}`<br>`PUT /api/books/{id}`<br>`DELETE /api/books/{id}` | `200 OK`<br>`201 Created`<br>`204 No Content` | Při `POST` a `PUT` přijímá autora jako `{"author": {"_id": 1}}`. Při čtení vrací v poli `author` plný objekt `AuthorOut`. |
| **UC-06** | Filtrování knih | `GET /api/books?...` | `200 OK` | Podporuje query parametry `authorID`, `genre`, `title` (podřetězec, case-insensitive), `minPrice`, `maxPrice` a `limit`. |
| **UC-07** | Knihy autora podle kódu | `GET /api/identification/{authorCode}/books/` | `200 OK`<br>`404 Not Found` | Vrátí všechny knihy autora s daným `authorCode` (i ty, které byly vytvořeny před úpravou autora v UC-03). |
| **UC-08** | Statistiky knihovny | `GET /api/books/statistics` | `200 OK` | Vrátí `allTimePriceSum`, `currentYearPriceSum` a `booksCount`. |
| **UC-09** | Statistiky autorů | `GET /api/authors/statistics` | `200 OK` | Vrátí pole objektů (`authorId`, `authorName`, `booksCount`, `totalBooksValue`) pro aktivní autory. |

---

## 5. Jak psát integrační testy v Django Ninja (`library/tests.py`)

V Django Ninja nepoužíváme `APITestCase` z DRF, ale standardní `django.test.TestCase` v kombinaci s vestavěným **`TestClient`** z modulu `ninja.testing`.

### Jak funguje `TestClient`?
- Vytvoříte jej předáním vaší instance `api` (nebo konkrétního routeru): `self.client = TestClient(api)`.
- Cesty pak píšete relativně k `api` (tedy např. `"/authors"` místo `"/api/authors"`).
- Při volání `post` nebo `put` předáváte slovník do parametru **`json={...}`** (nikoliv `data={...}` jako v DRF!).
- Odpověď převedenou z JSONu na Python slovník/seznam získáte přes **`response.json()`**.

### Ukázka testu v `library/tests.py`:

```python
from datetime import date
from decimal import Decimal
from django.test import TestCase
from ninja.testing import TestClient

from .api import api
from .models import Author, Book, Genres


class LibraryNinjaAPITestCase(TestCase):
    """Integrační testy pro Book Database API v Django Ninja."""

    def setUp(self):
        self.client = TestClient(api)

        self.author_active = Author.objects.create(
            name="Karel Čapek",
            authorCode="AUT-001",
            birthDate=date(1890, 1, 9),
            country="Czechia",
            mail="karel@capek.cz",
            biography="Spisovatel a dramatik",
            hidden=False,
        )

    def test_list_authors_returns_alias_id(self):
        """GET /authors vrátí seznam aktivních autorů s klíčem _id."""
        response = self.client.get("/authors")

        self.assertEqual(response.status_code, 200)
        data = response.json()
        self.assertEqual(len(data), 1)
        self.assertEqual(data[0]["_id"], self.author_active.id)
        self.assertEqual(data[0]["authorCode"], "AUT-001")

    def test_create_author_and_prevent_duplicate_active_code(self):
        """POST /authors vytvoří autora, ale odmítne duplicitní aktivní authorCode."""
        payload = {
            "name": "Josef Čapek",
            "authorCode": "AUT-001",  # Duplicitní kód!
            "birthDate": "1887-03-23",
            "country": "Czechia",
        }
        response = self.client.post("/authors", json=payload)
        self.assertEqual(response.status_code, 400)

    def test_update_author_with_same_code_succeeds(self):
        """PUT /authors/{id} umožní upravit autora i při zachování stejného authorCode."""
        payload = {
            "name": "Karel Čapek (Upraveno)",
            "authorCode": "AUT-001",  # Stejný kód jako měl původní záznam
            "birthDate": "1890-01-09",
            "country": "Czechia",
            "biography": "Doplněný životopis",
        }
        response = self.client.put(f"/authors/{self.author_active.id}", json=payload)

        self.assertEqual(response.status_code, 200)
        self.author_active.refresh_from_db()
        self.assertTrue(self.author_active.hidden)
        self.assertNotEqual(response.json()["_id"], self.author_active.id)
```

---

## 6. Časté chyby začátečníků v Django Ninja (Na co si dát pozor)

1. **Chybné pořadí endpointů v routeru**:
   Pokud napíšete `@router.get("/{book_id}")` před `@router.get("/statistics")`, endpoint `/statistics` nikdy nepůjde zavolat. Statické URL cesty mějte vždy nahoře!
2. **Použití `data=` místo `json=` v `TestClient`**:
   V `ninja.testing.TestClient` se JSON tělo posílá přes `self.client.post("/books", json=payload)`.
3. **N+1 problém u vnořených schémat (`BookOut`)**:
   Když má `BookOut` atribut `author: AuthorOut`, Django Ninja při serializaci přistoupí na `book.author`. Pokud v routeru zapomenete na `Book.objects.select_related("author")`, způsobíte N+1 SQL dotazů.
4. **Chybějící `populate_by_name=True` při použití `alias="_id"`**:
   Pokud v Pydantic schématu přejmenujete pole `id` pomocí `Field(..., alias="_id")`, nezapomeňte přidat `model_config = ConfigDict(populate_by_name=True, from_attributes=True)`, aby schéma umělo načíst atribut `id` z instance Django modelu.

---

## 7. Kritéria hodnocení (Definition of Done)

- [ ] Aplikace běží na **Django 5+** a **Django Ninja (`django-ninja`)** a na `/api/docs` zobrazuje funkční Swagger dokumentaci.
- [ ] Všechna vstupní a výstupní data jsou typově zvalidována přes Pydantic schémata v `library/schemas.py`.
- [ ] Veškerá business logika, transakce a agregace jsou odděleny v `library/services/`.
- [ ] Při výpisu knih je použit `select_related("author")` pro zamezení N+1 dotazů.
- [ ] V `library/tests.py` je alespoň **12 integračních testů** využívajících `ninja.testing.TestClient` s českou dokumentací (docstringy) a všechny testy procházejí (`python manage.py test`).
- [ ] Projekt obsahuje přehledný `README.md`.
