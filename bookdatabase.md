# Zadání projektu: Кnižní databáze (`SpringBookDatabase`)

## 1. Úvod a cíl projektu

Vaším úkolem je vytvořit backendovou REST API aplikaci v **Java Spring Boot** pro správu **knih a autorů (spisovatelů a překladatelů)**. Aplikace bude sloužit jako katalog knihovny nebo knihkupectví s podporou CRUD operací, historizace autorů, dynamického filtrování knih, výpočtu statistik a automaticky generované OpenAPI dokumentace.

Projekt vychází z architektury referenční aplikace `SpringInvoiceDatabase`.

### Požadované technologie
* **Java 17+** a **Maven**
* **Spring Boot 3.x** (`spring-boot-starter-web`, `spring-boot-starter-data-jpa`, `spring-boot-starter-validation`, `spring-boot-starter-test`)
* **MySQL** databáze (`mysql-connector-j`)
* **MapStruct** (mapování mezi entitami a DTO)
* **Hibernate JPA Metamodel Generator** (typově bezpečné Criteria API dotazy)
* **Lombok** (redukce boilerplate kódu)
* **Springdoc OpenAPI / Swagger UI** (dokumentace a testování REST API)
* **JUnit 5, Mockito, AssertJ, MockMvc** (unit testy)

---

## 2. Architektura a datový model

Aplikaci rozdělte do standardní vícevrstvé architektury:
* `cz.itnetwork.controller` (a `controller.advice`) – REST controllery a globální zpracování výjimek
* `cz.itnetwork.service` – rozhraní a implementace aplikační (business) logiky
* `cz.itnetwork.entity` (a `entity.repository`, `entity.repository.specification`, `entity.filter`) – JPA entity, Spring Data repozitáře a filtrační specifikace
* `cz.itnetwork.dto` (a `dto.mapper`) – přenosové objekty (DTO) a MapStruct mappery
* `cz.itnetwork.constant` – výčtové typy (`enum`)
* `cz.itnetwork.configuration` – konfigurace CORS

### Výčtové typy (`constant`)
* **`Countries`** (země původu autora): např. `CZECHIA`, `SLOVAKIA`, `USA`, `UK`, `OTHER`
* **`Genres`** (žánr knihy): např. `FANTASY`, `SCI_FI`, `DETECTIVE`, `BIOGRAPHY`, `EDUCATIONAL`

### Databázové entity (`entity`)

#### 1. `AuthorEntity` (tabulka `author`)
Reprezentuje osobu, která může být **autorem** nebo **překladatelem** knihy.
* `id` (`Long`, primární klíč, auto-increment)
* `name` (`String`, povinné) – celé jméno autora/překladatele
* `registrationNumber` (`String`, povinné) – evidenční číslo autora (obdoba IČO / ISNI)
* `birthDate` (`LocalDate`, povinné) – datum narození
* `country` (`Countries`, povinné, uložené jako `EnumType.STRING`) – země původu
* `mail` (`String`, povinné) – kontaktní e-mail
* `biography` (`String`, volitelné) – krátký životopis nebo poznámka
* `hidden` (`boolean`, výchozí `false`) – příznak logického smazání / historické verze záznamu
* `writtenBooks` (`List<BookEntity>`) – vazba `@OneToMany(fetch = FetchType.LAZY, mappedBy = "author")`
* `translatedBooks` (`List<BookEntity>`) – vazba `@OneToMany(fetch = FetchType.LAZY, mappedBy = "translator")`

#### 2. `BookEntity` (tabulka `book`)
Reprezentuje konkrétní knihu v databázi.
* `id` (`Long`, primární klíč, auto-increment)
* `isbn` (`String`, povinné) – ISBN kód knihy
* `title` (`String`, povinné) – název knihy
* `author` (`AuthorEntity`, povinné) – vazba `@ManyToOne(fetch = FetchType.LAZY)` na hlavního autora
* `translator` (`AuthorEntity`, volitelné) – vazba `@ManyToOne(fetch = FetchType.LAZY)` na překladatele
* `publishedDate` (`LocalDate`, povinné) – datum vydání (formát `yyyy-MM-dd`)
* `genre` (`Genres`, povinné, `EnumType.STRING`) – žánr knihy
* `pages` (`Integer`, povinné) – počet stran
* `price` (`BigDecimal`, povinné) – prodejní cena knihy
* `note` (`String`, volitelné) – poznámka ke knize

### Přenosové objekty (`dto`)
* **`AuthorDTO`** – obsahuje všechna veřejná pole autora (bez příznaku `hidden` a bez kolekcí knih). Pole `id` je v JSONu serializováno jako `_id` (`@JsonProperty("_id")`).
* **`BookDTO`** – obsahuje všechna pole knihy, přičemž `author` a `translator` jsou vnořené objekty typu `AuthorDTO`. Pole `id` je mapováno jako `_id`, datum `publishedDate` ve formátu `yyyy-MM-dd`.
* **`BookFilter`** – objekt pro query parametry filtrování knih:
  * `authorID` (`Long`)
  * `translatorID` (`Long`)
  * `title` (`String`)
  * `genre` (`Genres`)
  * `minPrice` (`BigDecimal`)
  * `maxPrice` (`BigDecimal`)
  * `limit` (`Integer`, výchozí `10`)
* **`BookStatisticsDTO`** – `currentYearPriceSum` (`BigDecimal`), `allTimePriceSum` (`BigDecimal`), `booksCount` (`long`).
* **`AuthorStatisticsDTO`** – `authorId` (`long`), `authorName` (`String`), `writtenBooksCount` (`long`), `totalBooksValue` (`BigDecimal`).

---

## 3. Přehled Use Cases (funkční požadavky a REST API)

### UC 1: Správa autorů a překladatelů (CRUD + historizace)
Protože na autora mohou být navázané již vydané knihy, autoři se z databáze fyzicky nemažou (tzv. *soft delete*).

| Metoda | Endpoint | Popis chování |
| --- | --- | --- |
| `POST` | `/api/authors` | Vytvoří nového autora a vrátí `201 Created` včetně vytvořeného `AuthorDTO`. |
| `GET` | `/api/authors` | Vrátí seznam všech aktivních autorů (`hidden = false`). |
| `GET` | `/api/authors/{authorId}` | Vrátí detail autora podle ID (vrací i skryté záznamy). Pokud neexistuje, vrátí `404 Not Found`. |
| `PUT` | `/api/authors/{authorId}` | **Historizace při úpravě:** Původní záznam autora nastaví na `hidden = true` a ze vstupního `AuthorDTO` vytvoří nový záznam v DB. Pokud původní ID neexistuje, vrátí `404 Not Found`. |
| `DELETE` | `/api/authors/{authorId}` | Nastaví autorovi `hidden = true`. Pokud záznam neexistuje, metoda tiše skončí bez chyby. |

### UC 2: Přehled knih podle evidenčního čísla autora (`registrationNumber`)
Při historizaci může v databázi existovat více záznamů téhož autora (se stejným `registrationNumber`). Endpointy musí vrátit knihy napříč všemi záznamy daného evidenčního čísla.

| Metoda | Endpoint | Popis chování |
| --- | --- | --- |
| `GET` | `/api/registration/{registrationNumber}/written` | Vrátí seznam všech knih (`List<BookDTO>`), které daná osoba napsala jako `author`. |
| `GET` | `/api/registration/{registrationNumber}/translated` | Vrátí seznam všech knih (`List<BookDTO>`), které daná osoba přeložila jako `translator`. |

### UC 3: Správa knih (CRUD)

| Metoda | Endpoint | Popis chování |
| --- | --- | --- |
| `POST` | `/api/books` | Vytvoří novou knihu, propojí ji s existujícím autorem (a případně překladatelem) podle jejich ID a vrátí uložené `BookDTO`. |
| `GET` | `/api/books/{id}` | Vrátí detail knihy podle ID. Pokud kniha neexistuje, vrátí `404 Not Found`. |
| `PUT` | `/api/books/{id}` | Aktualizuje existující knihu podle ID. Pokud kniha neexistuje, vrátí `404 Not Found`. |
| `DELETE` | `/api/books/{id}` | Fyzicky odstraní knihu z databáze podle ID. |

### UC 4: Dynamické filtrování knih
* **`GET /api/books`** – přijímá volitelné query parametry mapované do `BookFilter` (`authorID`, `translatorID`, `title`, `genre`, `minPrice`, `maxPrice`, `limit`).
* Filtrování implementujte pomocí `Specification<BookEntity>` (`JpaSpecificationExecutor`) a výsledky omezte přes `PageRequest.of(0, filter.getLimit())`.
* Pokud není zadán žádný filtr, vrátí prvních `limit` (výchozích 10) knih.

### UC 5: Statistiky knih a autorů
* **`GET /api/books/statistics`** – vrátí `BookStatisticsDTO`:
  * součet cen knih vydaných v aktuálním kalendářním roce (`currentYearPriceSum`),
  * součet cen všech knih v databázi (`allTimePriceSum`),
  * celkový počet knih v databázi (`booksCount`).
* **`GET /api/authors/statistics`** – vrátí `List<AuthorStatisticsDTO>`:
  * pro každého autora v databázi spočítá pomocí JPQL dotazu (`LEFT JOIN` + `GROUP BY`) počet napsaných knih a celkovou hodnotu (součet cen) jím napsaných knih (pokud žádné nenapsal, je součet `0`).

---

## 4. Doporučený postup tvorby aplikace (krok za krokem)

### Krok 1: Založení projektu a konfigurace
1. Vytvořte Maven projekt se Spring Boot 3 a přidejte do `pom.xml` potřebné závislosti (`web`, `data-jpa`, `validation`, `mysql`, `mapstruct`, `hibernate-jpamodelgen`, `lombok`, `springdoc-openapi-starter-webmvc-ui`, `test`).
2. Nastavte `annotationProcessorPaths` v `maven-compiler-plugin` pro Lombok, Hibernate Metamodel Generator a MapStruct.
3. V `src/main/resources/application.yaml` nastavte připojení k MySQL databázi `BookDatabase`, port `8080` a cestu pro OpenAPI dokumentaci.
4. Přidejte třídu `WebConfiguration` implementující `WebMvcConfigurer` pro povolení CORS (pozor: nepoužívejte `@EnableWebMvc`, abyste nevypnuli automatickou konfiguraci Spring Bootu).

### Krok 2: Vytvoření výčtových typů a JPA entit
1. Vytvořte enumy `Countries` a `Genres`.
2. Vytvořte `AuthorEntity` a `BookEntity` s příslušnými JPA anotacemi (`@Entity`, `@Id`, `@GeneratedValue`, `@Column`, `@Enumerated(EnumType.STRING)`, `@ManyToOne`, `@OneToMany`).
3. Všechny vazby `@ManyToOne` i `@OneToMany` nastavte jako `fetch = FetchType.LAZY`.
4. Přeložte projekt (`mvn compile`), aby Hibernate Metamodel Generator vygeneroval třídy `AuthorEntity_` a `BookEntity_`.

### Krok 3: DTO třídy a MapStruct mappery
1. Vytvořte `AuthorDTO`, `BookDTO`, `BookFilter`, `BookStatisticsDTO` a `AuthorStatisticsDTO`.
2. Vytvořte rozhraní `AuthorMapper` a `BookMapper` s anotací `@Mapper(componentModel = "spring")`.
3. V `BookMapper` využijte `uses = {AuthorMapper.class}` a ošetřete mapování metody `updateBookEntity(BookDTO source, @MappingTarget BookEntity target)`.

### Krok 4: Repozitáře a JPA Specification
1. Vytvořte `AuthorRepository extends JpaRepository<AuthorEntity, Long>` s metodami:
   * `List<AuthorEntity> findByHidden(boolean hidden);`
   * `List<AuthorEntity> findByRegistrationNumber(String registrationNumber);`
   * `@Query(...)` pro výpočet statistik autorů.
2. Vytvořte `BookRepository extends JpaRepository<BookEntity, Long>, JpaSpecificationExecutor<BookEntity>`.
3. Pro zabránění **N+1 problému** při načítání knih přepište v `BookRepository` metodu `findAll(Specification<BookEntity> spec, Pageable pageable)` a opatřete ji anotací `@EntityGraph(attributePaths = {"author", "translator"})`.
4. Implementujte třídu `BookSpecification implements Specification<BookEntity>`, která dynamicky skládá `Predicate` podmínky podle vyplněných polí v `BookFilter`.

### Krok 5: Servisní vrstva a ošetření výjimek
1. Vytvořte rozhraní `AuthorService` a `BookService` a jejich implementace `AuthorServiceImpl` a `BookServiceImpl`.
2. Pro injektáž závislostí použijte konstruktorovou injektáž (`private final` pole + `@RequiredArgsConstructor`).
3. Doplňte transakční anotace `@Transactional` na zapisovací metody a `@Transactional(readOnly = true)` na čtecí metody.
4. Vytvořte globální handler výjimek (např. `EntityNotFoundExceptionAdvice` s `@RestControllerAdvice`), který přeloží výjimku při nenalezení záznamu na HTTP status `404 Not Found`.

### Krok 6: REST Controllery
1. Vytvořte `AuthorController` a `BookController` s anotacemi `@RestController` a `@RequestMapping("/api")`.
2. Implementujte všechny endpointy podle tabulek v sekci **Use Cases**.
3. Spusťte aplikaci a otestujte endpointy přes Swagger UI (`http://localhost:8080/swagger-ui.html`) nebo Postman.

### Krok 7: Unit testy
1. V `src/test/java/cz/itnetwork/service/` vytvořte unit testy `AuthorServiceImplTest` a `BookServiceImplTest` pomocí `@ExtendWith(MockitoExtension.class)`.
2. Otestujte vytvoření záznamu, historizaci při `editAuthor`, tiché selhání při `removeAuthor`, vyhození výjimky při nenalezení ID, filtrování i výpočet statistik.
3. V `src/test/java/cz/itnetwork/controller/` vytvořte `AuthorControllerTest` a `BookControllerTest` s využitím `MockMvcBuilders.standaloneSetup(...)` a ověřte HTTP stavové kódy (`200`, `201`, `404`) i strukturu vráceného JSONu.

---

## 5. Bonusové úkoly

1. **Validace vstupů (`Bean Validation`):**
   * Doplňte do `AuthorDTO` a `BookDTO` validační anotace (`@NotBlank`, `@NotNull`, `@Email`, `@Positive`, `@PastOrPresent`) a do controllerů anotaci `@Valid`.
   * V `@RestControllerAdvice` přidejte handler pro `MethodArgumentNotValidException`, který vrátí `400 Bad Request` s přehledem chybných polí.
2. **Statistiky knih přímo v databázi:**
   * Místo načítání všech knih do paměti Javy a jejich sčítání v cyklu napište v `BookRepository` agregační `@Query` dotaz, který vrátí `BookStatisticsDTO` jediným SQL dotazem.
3. **Optimalizace filtrování podle cizího klíče:**
   * V `BookSpecification` při filtrování podle `authorID` a `translatorID` nepoužívejte `root.join(...)`, ale přistupte přímo k ID cizího klíče přes `root.get(BookEntity_.AUTHOR).get(AuthorEntity_.ID)`.
4. **Porovnání Lazy vs. Eager Loadingu:**
   * Zapněte v `application.yaml` výpis SQL dotazů (`show-sql: true`) a porovnejte počet vykonaných SQL dotazů při volání `GET /api/books` s použitím `@EntityGraph` a bez něj.
