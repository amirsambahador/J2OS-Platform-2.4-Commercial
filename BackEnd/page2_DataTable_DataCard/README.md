# page2 + DataTable + DataCard

*J2OS Platform 2.4*

Lists are in every application: books, orders, users, invoices. Each one needs paging, sorting and a search box, and each one tends to be rewritten from scratch. J2OS solves this once, on both sides of the wire:

- **page2** (`org.j2os.platform.page2`) is the Java side. It turns a JPA entity, a JPQL query, a SQL query or a plain `List` into one page of rows plus the total count.
- **DataTable** and **DataCard** are the Nuxt side. They show those pages as a table with pagination, or as a grid of cards with infinite scrolling.

The three pieces share a small, fixed protocol, so a DataTable can be pointed at any page2 endpoint without extra glue. The guide is organized like this: the first chapters get a list on screen and the later ones cover every option. The appendices are meant for quick lookups.

## Contents

1. [The protocol between browser and server](#1-the-protocol-between-browser-and-server)
2. [A working list in five minutes](#2-a-working-list-in-five-minutes)
3. [Choosing the right page2 class](#3-choosing-the-right-page2-class)
4. [PageDataEntity: lists straight from an entity](#4-pagedataentity-lists-straight-from-an-entity)
5. [How the search box behaves](#5-how-the-search-box-behaves)
6. [PageDataJPQL: when the query is hand-written](#6-pagedatajpql-when-the-query-is-hand-written)
7. [PageDataSQL: plain SQL over JDBC](#7-pagedatasql-plain-sql-over-jdbc)
8. [PageDataList: filtering a list in memory](#8-pagedatalist-filtering-a-list-in-memory)
9. [PageDataResultFilter: reshaping rows before they leave](#9-pagedataresultfilter-reshaping-rows-before-they-leave)
10. [DataTable](#10-datatable)
11. [DataCard](#11-datacard)
12. [In practice: a book catalog](#12-in-practice-a-book-catalog)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

All Java examples in chapters 5, 8 and 9 are complete programs that can be run as they are; the output printed below each of them is what it prints. Examples that need a database or a running Nuxt application are shown as they would sit in a project.

---

## 1. The protocol between browser and server

Everything in this guide rests on one request and one response.

**The request.** Whenever the user opens a page, types in the search box, clicks a column header or scrolls, the component sends a `GET` request with five query parameters:

```text
GET /api/books?page=2&rows=10&q=martin&sort=title&order=DESC
```

| Parameter | Meaning | Default |
|-----------|---------|---------|
| `page` | 1-based page number | `1` |
| `rows` | rows per page | `10` |
| `sort` | field to sort by. **Required.** It must have been registered with `searchAndSortOn` | none |
| `order` | `ASC` or `DESC`; anything else means `ASC` | `ASC` |
| `q` | text from the search box | empty (no search) |

**The response.** page2 answers with a map, which Spring writes as JSON:

```json
{
  "total": 37,
  "page": 2,
  "size": 10,
  "search": "martin",
  "rows": [
    { "bookId": 14, "title": "Refactoring", "author": "Martin Fowler", "price": 50,
      "publisher": { "publisherId": 2, "name": "Addison-Wesley" } }
  ]
}
```

| Key | Meaning |
|-----|---------|
| `total` | number of matching rows across all pages |
| `rows` | rows of the requested page |
| `page`, `size` | the page number and size that were actually applied |
| `search` | the trimmed, lower-cased search text |

Three rules are worth remembering from the start:

- `sort` is mandatory. A request without it fails with an `IllegalArgumentException`; the components always send it.
- A bad `page` or `rows` value is repaired, not rejected: a page below 1 becomes 1, and a size below 1 is replaced (by 10 in `PageDataEntity` and `PageDataList`, by 1 in `PageDataJPQL` and `PageDataSQL`). The `page` and `size` in the response always show what was applied. Text that is not a number is rejected.
- `q` may be empty or missing; both mean "no search".

## 2. A working list in five minutes

The smallest useful setup is one entity, one endpoint and one component.

**What is needed.** Java 17 or newer with Jakarta Persistence (Spring Boot 3 or later) and the page2 module on the classpath. `PageDataResultFilter` additionally uses Jackson 3 (`tools.jackson`) and SLF4J. On the front end: a Nuxt project with Nuxt UI, and `DataTable.vue` and `DataCard.vue` copied into `components/`.

**Step 1 – the entities.** A book that belongs to a publisher:

```java
import jakarta.persistence.*;
import lombok.Data;

@Entity
@Data
public class Publisher {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long publisherId;

    private String name;
}
```

```java
import jakarta.persistence.*;
import lombok.Data;

@Entity
@Data
public class Book {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long bookId;

    private String title;
    private String author;
    private Integer price;
    private boolean active = true;

    @ManyToOne
    private Publisher publisher;
}
```

**Step 2 – the endpoint.** `@RequestParam Map<String, Object>` hands the whole query string to page2, which reads the five parameters it knows and ignores the rest:

```java
import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import org.j2os.platform.page2.PageDataEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import java.util.Map;

@RestController
@RequestMapping("/api/books")
public class BookAPI {

    @PersistenceContext
    private EntityManager entityManager;

    @GetMapping
    public Map<String, Object> list(@RequestParam Map<String, Object> params) {
        return new PageDataEntity(entityManager)
                .searchAndSortOn("bookId", "title", "author", "price", "publisher.name")
                .getResult(Book.class, params);
    }
}
```

`searchAndSortOn` does two things at once: it lists the fields that the search box looks into, and it lists the fields the user may sort by. `publisher.name` reaches into the related publisher.

**Step 3 – the page.** Nuxt registers everything in `components/` automatically. The `url` must end with `?` (or `&` when it already carries parameters) because the component appends the query string directly:

```vue
<script setup>
const config = useRuntimeConfig()
const url = `${config.public.API_BASE}/api/books?`

const columns = [
  { field: 'bookId', label: 'ID', sortable: true },
  { field: 'title', label: 'Title', sortable: true },
  { field: 'author', label: 'Author', sortable: true },
  { field: 'price', label: 'Price', sortable: true }
]
</script>

<template>
  <DataTable
    :url="url"
    :columns="columns"
    default-sort="bookId"
    row-key="bookId"
  />
</template>
```

The component fetches the first page on mount, renders the table, shows a pagination bar, and re-fetches whenever the user searches, sorts or changes the page. Nothing else needs to be written.

The component reads the server address from `useRuntimeConfig().public.API_BASE`, so the project needs this once:

```ts
export default defineNuxtConfig({
  runtimeConfig: {
    public: {
      API_BASE: 'http://localhost:8080'
    }
  }
})
```

## 3. Choosing the right page2 class

page2 has four query classes. All of them share the same grid parameters, the same result map and the same `searchAndSortOn` method, so switching between them later is easy.

| Class | Data source | Use it when |
|-------|-------------|-------------|
| `PageDataEntity` | JPA entity class | the list is one entity, optionally filtered. No query to write. **First choice for database data.** |
| `PageDataJPQL` | JPQL text | the list needs a custom query: sub-queries, `group by`, hand-picked joins |
| `PageDataSQL` | SQL text and a JDBC `Connection` | the list is a report, spans tables that are not entities, or uses database-specific features |
| `PageDataList` | `java.util.List` | the data already sits in memory: a cache, a service result, a small configuration list |

A fifth class, `PageDataResultFilter`, post-processes the result of any of the four (chapter 9).

Two rules apply to all of them:

1. **Create a new instance for every request.** An instance collects its filters and search fields while it is configured, so sharing one across requests (for example as a singleton Spring bean) mixes up the requests.
2. **Register every field that may be sorted or searched.** Fields that are not registered cannot be sorted on and are never searched. This is deliberate: it keeps the browser from asking for a sort on a column that has no index, or a search through a column that holds secrets.

## 4. PageDataEntity: lists straight from an entity

`PageDataEntity` generates the JPQL itself from the entity class, so the list endpoint stays a few lines long.

### 4.1 The pieces

```java
new PageDataEntity(entityManager)            // 1. bind to an EntityManager
    .searchAndSortOn("bookId", "title")       // 2. which fields can be sorted and searched
    .where(...).and(...).or(...)              // 3. optional filters
    .restrict(...)                            // 4. optional rules that can never be bypassed
    .getResult(Book.class, params);           // 5. run
```

Each call to `getResult` runs two queries: one for the page and one `COUNT` for the total.

### 4.2 Nested fields

A dotted path such as `publisher.name` works for single-valued relations (`@ManyToOne`, `@OneToOne`) and `@Embedded` fields. Every relation on the way is reached with a `LEFT JOIN`, so books without a publisher stay in the list. For a request with `sort=publisher.name&order=DESC` the generated query looks like this:

```text
SELECT o FROM Book o LEFT JOIN o.publisher j0
WHERE 1=1
ORDER BY j0.name DESC, o.bookId ASC
```

Collections (`@OneToMany`, `@ManyToMany`) are not supported in a path. For those, write the query by hand (chapter 6).

When several rows have the same value in the sort column, the entity identifier is added as a last sort key, so no row appears twice or goes missing between pages.

### 4.3 Filters: where, and, or

Filters narrow the list before the search box applies. Each takes a field, an operator and a value:

```java
.where("price", "<=", 50)
```

The supported operators are `=`, `!=`, `<>`, `>`, `>=`, `<`, `<=`, `LIKE`, `NOT LIKE`, `IS NULL` and `IS NOT NULL`. Any other operator is rejected immediately with an `IllegalArgumentException`, and so is a field name that is not a plain identifier (letters, digits, underscore and dots), which closes the door on injection.

`where` and `and` are the same thing; `or` joins with `OR`. Conditions are evaluated in the order written, with `AND` binding tighter than `OR` and no way to add parentheses:

```java
new PageDataEntity(entityManager)
        .searchAndSortOn("bookId", "title")
        .where("price", "<", 30)
        .or("title", "LIKE", "%Java%")
        .and("price", "<", 60)
        .restrict("active", "=", true)
        .getResult(Book.class, params);
```

becomes

```text
SELECT o FROM Book o
WHERE 1=1
  AND (o.price < :p1 OR o.title LIKE :p2 AND o.price < :p3)
  AND o.active = :p4
ORDER BY o.bookId ASC
```

Values are always sent as query parameters, never pasted into the text (the names above are shortened).

**Converting values.** The value may come from the URL, and everything in a URL is a string. A fourth argument converts it just before it is used:

```java
@GetMapping("/affordable")
public Map<String, Object> affordable(@RequestParam Map<String, Object> params) {
    PageDataEntity page = new PageDataEntity(entityManager)
            .searchAndSortOn("bookId", "title", "author", "price", "publisher.name")
            .restrict("active", "=", true);

    Object maxPrice = params.get("maxPrice");
    if (maxPrice != null) {
        page.where("price", "<=", maxPrice, value -> Integer.valueOf(value.toString()));
    }
    return page.getResult(Book.class, params);
}
```

`IS NULL` and `IS NOT NULL` take no value; pass `null`.

`LIKE` follows the database: it is case-sensitive in PostgreSQL, and `%` must be written explicitly (`"%Java%"`).

### 4.4 restrict: rules that cannot be bypassed

Because of the `AND`/`OR` precedence, a security condition added with `where` could be canceled by a later `or`. `restrict` avoids that: it is always combined with `AND` to the whole query, whatever else is configured.

```java
.restrict("organizationId", "=", currentOrganizationId)
.restrict("active", "=", true)
```

Fields used only in `where` or `restrict` are **not** sortable or searchable. That makes it safe to filter on a sensitive field without exposing it to the browser.

Master–detail screens are a natural fit for `restrict` on a nested path. This endpoint serves the books of one publisher:

```java
@GetMapping("/by-publisher")
public Map<String, Object> byPublisher(@RequestParam Map<String, Object> params) {
    Long publisherId = Long.valueOf(params.get("publisherId").toString());

    return new PageDataEntity(entityManager)
            .searchAndSortOn("bookId", "title", "price")
            .restrict("publisher.publisherId", "=", publisherId)
            .getResult(Book.class, params);
}
```

## 5. How the search box behaves

The search rules are identical in `PageDataEntity`, `PageDataJPQL`, `PageDataSQL` and `PageDataList`. The four classes share one implementation of the search, so the matches are the same. `PageDataList` can run without a database, so the examples here use it.

### 5.1 Paging, sorting and the result map

The first program lists books, two per page, sorted by title:

```java
import org.j2os.platform.page2.PageDataList;

import java.util.List;
import java.util.Map;

public class BookPagingDemo {

    record Publisher(String name) {
    }

    record Book(int id, String title, String author, int price, Publisher publisher) {
    }

    public static void main(String[] args) {
        List<Book> books = List.of(
                new Book(1, "Clean Code", "Robert Martin", 40, new Publisher("Prentice Hall")),
                new Book(2, "Effective Java", "Joshua Bloch", 45, new Publisher("Addison-Wesley")),
                new Book(3, "Refactoring", "Martin Fowler", 50, new Publisher("Addison-Wesley")),
                new Book(4, "Java Concurrency", "Brian Goetz", 55, new Publisher("Addison-Wesley")),
                new Book(5, "Head First Java", "Kathy Sierra", 35, new Publisher("O'Reilly")));

        Map<String, Object> result = new PageDataList()
                .searchAndSortOn("id", "title", "author", "publisher.name")
                .getResult(books, Map.of("sort", "title", "order", "ASC", "page", "1", "rows", "2"));

        System.out.println("total  = " + result.get("total"));
        System.out.println("page   = " + result.get("page"));
        System.out.println("size   = " + result.get("size"));
        System.out.println("search = '" + result.get("search") + "'");
        for (Object row : (List<?>) result.get("rows")) {
            System.out.println(row);
        }
    }
}
```

Output:

```text
total  = 5
page   = 1
size   = 2
search = ''
Book[id=1, title=Clean Code, author=Robert Martin, price=40, publisher=Publisher[name=Prentice Hall]]
Book[id=2, title=Effective Java, author=Joshua Bloch, price=45, publisher=Publisher[name=Addison-Wesley]]
```

Two details are visible here. `rows` contains the original objects (entities in `PageDataEntity` and `PageDataJPQL`, maps in `PageDataSQL`). And the four registered fields, including the nested `publisher.name`, are the only ones usable for sorting.

Sorting descending by a nested field, on the second page:

```java
import org.j2os.platform.page2.PageDataList;

import java.util.List;
import java.util.Map;

public class NestedSortDemo {

    record Publisher(String name) {
    }

    record Book(int id, String title, Publisher publisher) {
    }

    public static void main(String[] args) {
        List<Book> books = List.of(
                new Book(1, "Clean Code", new Publisher("Prentice Hall")),
                new Book(2, "Effective Java", new Publisher("Addison-Wesley")),
                new Book(3, "Refactoring", new Publisher("Addison-Wesley")),
                new Book(4, "Java Concurrency", new Publisher("Manning")),
                new Book(5, "Head First Java", new Publisher("O'Reilly")));

        Map<String, Object> result = new PageDataList()
                .searchAndSortOn("id", "title", "publisher.name")
                .getResult(books, Map.of("sort", "publisher.name", "order", "DESC", "page", "2", "rows", "2"));

        System.out.println("total=" + result.get("total") + " page=" + result.get("page") + " size=" + result.get("size"));
        for (Object row : (List<?>) result.get("rows")) {
            Book book = (Book) row;
            System.out.println(book.publisher().name() + " / " + book.title());
        }
    }
}
```

```text
total=5 page=2 size=2
Manning / Java Concurrency
Addison-Wesley / Effective Java
```

### 5.2 What a query string does

The program below tries six search texts:

```java
import org.j2os.platform.page2.PageDataList;

import java.util.List;
import java.util.Map;

public class SearchDemo {

    record Publisher(String name) {
    }

    record Book(int id, String title, String author, int price, Publisher publisher) {
    }

    public static void main(String[] args) {
        List<Book> books = List.of(
                new Book(1, "Clean Code", "Robert Martin", 40, new Publisher("Prentice Hall")),
                new Book(2, "Effective Java", "Joshua Bloch", 45, new Publisher("Addison-Wesley")),
                new Book(3, "Refactoring", "Martin Fowler", 50, new Publisher("Addison-Wesley")),
                new Book(4, "Java Concurrency", "Brian Goetz", 55, new Publisher("Addison-Wesley")),
                new Book(5, "Head First Java", "Kathy Sierra", 35, new Publisher("O'Reilly")));

        String[] queries = {"martin", "java wesley", "\"Clean Code\"", "\"clean\"", "eff_ctive", "%java"};
        for (String q : queries) {
            Map<String, Object> result = new PageDataList()
                    .searchAndSortOn("id", "title", "author", "publisher.name")
                    .getResult(books, Map.of("sort", "id", "q", q));
            List<?> rows = (List<?>) result.get("rows");
            System.out.println("q=" + q + "  ->  total=" + result.get("total"));
            for (Object row : rows) {
                System.out.println("    " + ((Book) row).title());
            }
        }
    }
}
```

```text
q=martin  ->  total=2
    Clean Code
    Refactoring
q=java wesley  ->  total=4
    Effective Java
    Refactoring
    Java Concurrency
    Head First Java
q="Clean Code"  ->  total=1
    Clean Code
q="clean"  ->  total=0
q=eff_ctive  ->  total=1
    Effective Java
q=%java  ->  total=3
    Effective Java
    Java Concurrency
    Head First Java
```

Reading the results:

- **Case does not matter.** `martin` finds *Robert Martin* and *Martin Fowler*.
- **Several words widen the search.** `java wesley` returns rows that contain *either* word, in *any* registered field. Words are combined with OR, not AND.
- **Quotes mean an exact match.** `"Clean Code"` matches only a field that is exactly `Clean Code`. `"clean"` finds nothing, because no field is exactly `clean`. Several quoted phrases can be combined: `"Clean Code" "Refactoring"`.
- **`_` and `%` are wildcards** in an unquoted search: `eff_ctive` matches *Effective*, and `%java` matches anything that contains *java*. Inside quotes they are plain characters, so `"50%"` finds the literal text `50%`.
- A field value of `null` is treated as an empty text.

### 5.3 Persian and Arabic letters

Persian and Arabic keyboards produce different characters for the letters *ye* (ی / ي / ى) and *ke* (ک / ك). The search treats them as the same letter, so data entered with one keyboard is found by the other. In this program the data is stored with the Persian *ke* and searched with the Arabic *kaf*:

```java
import org.j2os.platform.page2.PageDataList;

import java.util.List;
import java.util.Map;

public class PersianSearchDemo {

    record City(int id, String name) {
    }

    public static void main(String[] args) {
        // "ی" and "ک" below are the Persian letters; the search is typed with the Arabic ones.
        List<City> cities = List.of(
                new City(1, "کرج"),
                new City(2, "یزد"),
                new City(3, "تهران"));

        String arabicYehKaf = "كرج";
        Map<String, Object> result = new PageDataList()
                .searchAndSortOn("id", "name")
                .getResult(cities, Map.of("sort", "id", "q", arabicYehKaf));

        System.out.println("total = " + result.get("total"));
        for (Object row : (List<?>) result.get("rows")) {
            System.out.println(row);
        }
    }
}
```

```text
total = 1
City[id=1, name=کرج]
```

This works the same way in the three database classes; the search text is expanded to every spelling before the query is built.

## 6. PageDataJPQL: when the query is hand-written

Some lists cannot be described as "one entity plus filters": a sub-query, a `group by`, a join that is not a relation. For these, write the JPQL and let page2 add the search, the sorting and the paging.

```java
@GetMapping("/above-average")
public Map<String, Object> aboveAverage(@RequestParam Map<String, Object> params) {
    String jpql = "select b from Book b "
            + "where b.price > (select avg(x.price) from Book x where x.active = true)";
    String countJpql = "select count(b) from Book b "
            + "where b.price > (select avg(x.price) from Book x where x.active = true)";

    return new PageDataJPQL(entityManager)
            .searchAndSortOn("bookId", "title", "price", "publisher.name")
            .getResult(jpql, countJpql, "b", Book.class, null, params);
}
```

`getResult` takes six arguments:

| Argument | Meaning |
|----------|---------|
| `jpql` | the query that returns the rows |
| `countJpql` | the matching query that counts them |
| `"b"` | the alias that the entity has in both queries |
| `Book.class` | what the query returns |
| `queryParams` | named parameters used by the two queries, or `null` |
| `params` | the grid parameters from the browser |

Named parameters are given in a map:

```java
@GetMapping("/cheaper-than")
public Map<String, Object> cheaperThan(@RequestParam Map<String, Object> params) {
    Map<String, Object> queryParams = new HashMap<>();
    queryParams.put("limit", Integer.valueOf(params.get("limit").toString()));

    return new PageDataJPQL(entityManager)
            .searchAndSortOn("bookId", "title")
            .getResult(
                    "select b from Book b where b.price < :limit",
                    "select count(b) from Book b where b.price < :limit",
                    "b", Book.class, queryParams, params);
}
```

When the user searches, sorts and goes to the next page, page2 changes the text like this (formatting simplified):

```text
select b from Book b where  (b.price < :limit
) AND (LOWER(COALESCE(CONCAT(b.bookId, ''), '')) LIKE :search_0 ESCAPE '\'
       OR LOWER(COALESCE(CONCAT(b.title, ''), '')) LIKE :search_1 ESCAPE '\')
ORDER BY b.title ASC, b.bookId ASC
```

What happens to the hand-written query:

- The search condition is added with `AND` to the existing `WHERE`, or a `WHERE` is added if there is none. An existing condition containing `OR` is wrapped in parentheses first, so it keeps its meaning.
- Sub-queries are left alone: `GROUP BY`, `ORDER BY` and `WHERE` inside parentheses are never mistaken for the main query's.
- The query should not contain its own `ORDER BY`; the sort comes from the browser. If it has one, the requested sort is appended as an additional key.
- The two queries are trusted text written by the developer. Never build them from request values; pass those as `queryParams`.
- Names `search_0`, `search_1`, … are used internally and must not appear in `queryParams`.
- Registered fields may be nested (`publisher.name`); the needed `LEFT JOIN`s are added after the entity alias in the `FROM` clause.

## 7. PageDataSQL: plain SQL over JDBC

`PageDataSQL` takes a JDBC `Connection`. It targets PostgreSQL (H2 also accepts the generated SQL) and uses `LIMIT`/`OFFSET`.

```java
@RestController
@RequestMapping("/api/report")
public class ReportAPI {

    @Resource
    private DataSource dataSource;

    @GetMapping
    @Transactional(readOnly = true)
    public Map<String, Object> report(@RequestParam Map<String, Object> params) {
        Connection connection = DataSourceUtils.getConnection(dataSource);
        try {
            return new PageDataSQL(connection)
                    .searchAndSortOn("b.book_id", "b.title", "p.name")
                    .getResult(
                            "SELECT b.book_id, b.title, p.name AS publisher_name "
                                    + "FROM book b JOIN publisher p ON p.publisher_id = b.publisher_id "
                                    + "WHERE b.active = true",
                            "b.book_id", null, params);
        } finally {
            DataSourceUtils.releaseConnection(connection, dataSource);
        }
    }
}
```

Things to notice:

- **Use the connection that Spring manages.** `DataSourceUtils.getConnection` and `releaseConnection` keep the call inside the current transaction.
- **Registered fields are SQL expressions.** They are placed in `WHERE` and `ORDER BY`, where a column alias such as `publisher_name` is not allowed. Register `p.name`, not `publisher_name`.
- **The second argument, `"b.book_id"`, is the primary key.** It is used as a last sort key so that equal sort values never reorder between pages. It is optional: the three-argument overload `getResult(sql, sqlParams, params)` leaves it out.
- **Rows are maps** keyed by the result column label. Labels that would clash get a numeric suffix (`id`, `id_2`).
- In the browser, the table must send the registered name when sorting, while reading the label from the row. That is what `sortName` is for:

```vue
const columns = [
  { field: 'book_id', label: 'ID', sortable: true, sortName: 'b.book_id' },
  { field: 'title', label: 'Title', sortable: true, sortName: 'b.title' },
  { field: 'publisher_name', label: 'Publisher', sortable: true, sortName: 'p.name' }
]
```

**Named parameters** are written `:name` in the SQL (PostgreSQL's `::type` casts, quoted text and comments are skipped correctly):

```java
Map<String, Object> sqlParams = new HashMap<>();
sqlParams.put("minPrice", 40);

new PageDataSQL(connection)
        .searchAndSortOn("b.book_id", "b.title")
        .getResult("SELECT b.book_id, b.title FROM book b WHERE b.price >= :minPrice",
                "b.book_id", sqlParams, params);
```

A missing parameter fails with `PageDataSQL: missing value for named parameter ':minPrice'`.

**Combined queries.** A query that uses `UNION`, `INTERSECT` or `EXCEPT` is wrapped as a table and searched as a whole. The registered fields must then be unqualified result column names:

```java
new PageDataSQL(connection)
        .searchAndSortOn("code", "label")
        .getResult(
                "SELECT 'B' || book_id AS code, title AS label FROM book "
                        + "UNION ALL "
                        + "SELECT 'P' || publisher_id, name FROM publisher",
                null, params);
```

## 8. PageDataList: filtering a list in memory

`PageDataList` applies the same protocol to a `java.util.List`. It reads fields by reflection (including nested paths and inherited fields) and sorts, filters and pages in memory. It is a good fit for small lists, and a poor one for tables that hold thousands of rows. The whole list is processed on each request.

```java
@GetMapping("/countries")
public Map<String, Object> countries(@RequestParam Map<String, Object> params) {
    List<Country> all = countryService.findAll();

    return new PageDataList()
            .searchAndSortOn("code", "name", "region.name")
            .getResult(all, params);
}
```

### 8.1 Filters

The same `where`, `and`, `or`, `restrict` and `searchAndSortOn` methods exist. The next program tries four filters on a list in which one book has no price:

```java
import org.j2os.platform.page2.PageDataList;

import java.util.List;
import java.util.Map;

public class FilterDemo {

    record Publisher(String name) {
    }

    record Book(int id, String title, String author, Integer price, Publisher publisher) {
    }

    @SuppressWarnings("unchecked")
    public static void main(String[] args) {
        List<Book> books = List.of(
                new Book(1, "Clean Code", "Robert Martin", 40, new Publisher("Prentice Hall")),
                new Book(2, "Effective Java", "Joshua Bloch", 45, new Publisher("Addison-Wesley")),
                new Book(3, "Refactoring", "Martin Fowler", 50, new Publisher("Addison-Wesley")),
                new Book(4, "Java Concurrency", "Brian Goetz", null, new Publisher("Addison-Wesley")),
                new Book(5, "Head First Java", "Kathy Sierra", 35, new Publisher("O'Reilly")));
        Map<String, Object> grid = Map.of("sort", "id");

        // AND binds tighter than OR:  price < 45  OR  (author LIKE '%Martin%' AND price >= 50)
        // In a list, a missing price counts as smaller than any value, so "Java Concurrency" matches price < 45.
        Map<String, Object> a = new PageDataList()
                .searchAndSortOn("id", "title")
                .where("price", "<", 45)
                .or("author", "LIKE", "%Martin%")
                .and("price", ">=", 50)
                .getResult(books, grid);
        System.out.println("A: total=" + a.get("total") + " " + ((List<Book>) a.get("rows")).stream().map(Book::title).toList());

        // restrict() is always ANDed with the whole filter:  (price < 45 OR price > 49) AND publisher = Addison-Wesley
        Map<String, Object> b = new PageDataList()
                .searchAndSortOn("id", "title")
                .where("price", "<", 45)
                .or("price", ">", 49)
                .restrict("publisher.name", "=", "Addison-Wesley")
                .getResult(books, grid);
        System.out.println("B: total=" + b.get("total") + " " + ((List<Book>) b.get("rows")).stream().map(Book::title).toList());

        // Use IS NULL / IS NOT NULL to test for a missing value; the value argument is ignored.
        Map<String, Object> c = new PageDataList()
                .searchAndSortOn("id", "title")
                .where("price", "IS NULL", null)
                .getResult(books, grid);
        System.out.println("C: total=" + c.get("total") + " " + ((List<Book>) c.get("rows")).stream().map(Book::title).toList());

        // The transformer converts the incoming value (usually a String from the URL) before comparing.
        Map<String, Object> d = new PageDataList()
                .searchAndSortOn("id", "title")
                .where("price", ">=", "50", value -> Integer.valueOf(value.toString()))
                .getResult(books, grid);
        System.out.println("D: total=" + d.get("total") + " " + ((List<Book>) d.get("rows")).stream().map(Book::title).toList());
    }
}
```

```text
A: total=4 [Clean Code, Refactoring, Java Concurrency, Head First Java]
B: total=2 [Refactoring, Java Concurrency]
C: total=1 [Java Concurrency]
D: total=1 [Refactoring]
```

- **A.** Reads as `price < 45 OR (author LIKE '%Martin%' AND price >= 50)`.
- **B.** `restrict` applies to the result of the whole `where`/`or` group.
- **C.** `IS NULL` is the way to find missing values.
- **D.** The transformer turns the text `"50"` into a number before comparing.
- **Missing values behave differently here.** In a `List`, `null` counts as smaller than any value, which is why *Java Concurrency* appears in A and B. The database classes treat `null` as unknown and leave such rows out of `<`, `>` and `=`.
- Numbers of different types compare by value (`Long` against `Integer` works). A `LIKE` pattern is case-insensitive and must match the whole value.

### 8.2 Mistakes are reported

Each wrong call gets a precise message:

```java
import org.j2os.platform.page2.PageDataList;

import java.util.List;
import java.util.Map;

public class ErrorDemo {

    record Book(int id, String title, int price) {
    }

    public static void main(String[] args) {
        List<Book> books = List.of(new Book(1, "Clean Code", 40), new Book(2, "Refactoring", 50));

        try {
            new PageDataList().searchAndSortOn("id", "title")
                    .getResult(books, Map.of("sort", "price"));
        } catch (RuntimeException e) {
            System.out.println("1) " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }

        try {
            new PageDataList().searchAndSortOn("id", "title")
                    .getResult(books, Map.of("page", "1"));
        } catch (RuntimeException e) {
            System.out.println("2) " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }

        try {
            new PageDataList().searchAndSortOn("id", "title")
                    .getResult(books, Map.of("sort", "id", "page", "abc"));
        } catch (RuntimeException e) {
            System.out.println("3) " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }

        try {
            new PageDataList().searchAndSortOn("id; drop table book");
        } catch (RuntimeException e) {
            System.out.println("4) " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }

        try {
            new PageDataList().searchAndSortOn("id").where("price", "CONTAINS", 3);
        } catch (RuntimeException e) {
            System.out.println("5) " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }

        try {
            new PageDataList().searchAndSortOn("id", "titel")
                    .getResult(books, Map.of("sort", "id"));
        } catch (RuntimeException e) {
            System.out.println("6) " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }

        Map<String, Object> clamped = new PageDataList().searchAndSortOn("id")
                .getResult(books, Map.of("sort", "id", "page", "-4", "rows", "0"));
        System.out.println("7) page=" + clamped.get("page") + " size=" + clamped.get("size"));
    }
}
```

```text
1) RuntimeException: price is not sortable
2) IllegalArgumentException: gridParams must include a 'sort' field (sort is mandatory)
3) IllegalArgumentException: gridParams['page'] must be a valid integer, got: 'abc'
4) IllegalArgumentException: Invalid/unsafe field name: id; drop table book
5) IllegalArgumentException: Unsupported operator: CONTAINS
6) IllegalArgumentException: PageDataList: 'titel' is not a field of the items
7) page=1 size=10
```

Line 1 is a `RuntimeException`, the rest are `IllegalArgumentException`. Line 7 shows the repair of page and size values.

## 9. PageDataResultFilter: reshaping rows before they leave

Rows from page2 are entities or maps, and sometimes they carry more than the browser should see. `PageDataResultFilter` converts each row to a map and applies rules in order:

| Method | Effect |
|--------|--------|
| `remove(path)` | deletes the field |
| `empty(path)` | sets it to an empty string |
| `mask(path)` | sets it to `********` |
| `put(path, row -> value)` | adds or replaces a field with a computed value |
| `put(path, Type.class, (row, current) -> value)` | changes the current value; `current` is converted to `Type` first |

Paths can be nested (`publisher.name`). Rules run in the order written, for each row.

```java
import org.j2os.platform.page2.PageDataList;
import org.j2os.platform.page2.PageDataResultFilter;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;

public class ResultFilterDemo {

    record Publisher(String name) {
    }

    record Employee(int id, String name, String email, String password, Publisher publisher) {
    }

    @SuppressWarnings("unchecked")
    public static void main(String[] args) {
        List<Employee> employees = List.of(
                new Employee(1, "Sara Ahmadi", "sara@example.com", "s3cret", new Publisher("Addison-Wesley")),
                new Employee(2, "Omid Karimi", "omid@example.com", "p4ss", new Publisher("O'Reilly")));

        Map<String, Object> raw = new PageDataList()
                .searchAndSortOn("id", "name")
                .getResult(employees, Map.of("sort", "id"));

        Map<String, Object> shaped = new PageDataResultFilter<Employee>(raw)
                .remove("email")
                .mask("password")
                .empty("publisher.name")
                .put("displayName", e -> e.name().toUpperCase())
                .put("tags", e -> new ArrayList<>(List.of("staff")))
                .put("tags", List.class, (e, tags) -> {
                    tags.add("id-" + e.id());
                    return tags;
                })
                .getResult();

        for (Object row : (List<?>) shaped.get("rows")) {
            System.out.println(row);
        }

        PageDataResultFilter<Employee> once = new PageDataResultFilter<>(
                new PageDataList().searchAndSortOn("id").getResult(employees, Map.of("sort", "id")));
        once.getResult();
        try {
            once.getResult();
        } catch (IllegalStateException e) {
            System.out.println(e.getMessage());
        }
    }
}
```

```text
{id=1, name=Sara Ahmadi, password=********, publisher={name=}, displayName=SARA AHMADI, tags=[staff, id-1]}
{id=2, name=Omid Karimi, password=********, publisher={name=}, displayName=OMID KARIMI, tags=[staff, id-2]}
PageDataResultFilter.getResult() has already been called on this instance; create a new instance for each result/request instead of reusing this one
```

What the output shows:

- `email` is gone, `password` is masked, `publisher.name` is emptied.
- `displayName` is a new field that exists only in the response.
- `tags` is created by the first `put` and extended by the second, which works on the existing list.
- The second `getResult()` on the same filter fails. A filter is **single-use**; create a new one per response.

A typical use inside an endpoint:

```java
@GetMapping("/public")
public Map<String, Object> publicList(@RequestParam Map<String, Object> params) {
    Map<String, Object> result = new PageDataEntity(entityManager)
            .searchAndSortOn("bookId", "title", "author", "price")
            .restrict("active", "=", true)
            .getResult(Book.class, params);

    return new PageDataResultFilter<Book>(result)
            .remove("active")
            .put("priceLabel", book -> book.getPrice() + " USD")
            .getResult();
}
```

A few more facts: the original rows are copied and never modified; a failing transformer sets the field to `null` and writes a warning to the log instead of failing the whole request; `empty` and `mask` skip a path if something on the way is `null` or not an object.

## 10. DataTable

`DataTable` is a Nuxt UI based table with a search box, sortable headers, selection and pagination. It speaks the protocol of chapter 1 and needs only a `url`, the `columns`, a default sort and a row key.

### 10.1 Props

| Prop | Type | Default | Meaning |
|------|------|---------|---------|
| `url` | String, required | – | Endpoint, ending in `?` or `&`. An empty string clears the table. |
| `columns` | Array, required | – | Column definitions (10.2) |
| `defaultSort` | String, required | – | Field sent as `sort` before the user clicks any header. Must be registered on the server. |
| `defaultOrder` | `'ASC'` or `'DESC'` | `'ASC'` | Initial direction |
| `rowKey` | String | `'id'` | Field that is unique in each row (the id) |
| `pageSize` | Number | `10` | Sent as `rows` |
| `emptyText` | String | `'داده‌ای وجود ندارد'` | Text when there are no rows |
| `rowStyle` | Function | none | `row => cssString`, applied to each cell of the row |
| `multiSelectable` | Boolean | `false` | Checkboxes instead of radio buttons |

### 10.2 Columns

A column is an object:

| Key | Meaning |
|-----|---------|
| `field` | name of the field in the row |
| `label` | header text |
| `sortable` | `true` lets the user click the header |
| `sortName` | name sent as `sort` instead of `field`. Needed when the displayed value is nested or computed |
| `processor` | `row => value` computes what the cell shows |
| `render` | `true` inserts the processor's result as **HTML** |

```vue
const xss = useXssProtector()

const columns = [
  { field: 'bookId', label: 'ID', sortable: true },
  { field: 'title', label: 'Title', sortable: true },

  // The cell shows publisher.name; the server sorts by "publisher.name".
  {
    field: 'publisher',
    label: 'Publisher',
    sortable: true,
    sortName: 'publisher.name',
    processor: row => row.publisher?.name ?? ''
  },

  // processor + render: the returned string is inserted as HTML, so it must be sanitized.
  {
    field: 'price',
    label: 'Price',
    sortable: true,
    processor: row => `<b>${xss.toPlainText(row.price)}</b> USD`,
    render: true
  },

  // No field: the column is always visible. It cannot be sorted by the server, so sortable stays off.
  {
    label: 'Code',
    processor: row => `BK-${row.bookId}`
  }
]
```

Rules that are easy to miss:

- **A column whose `field` is missing from the rows is hidden.** The check uses the rows of the current page. This lets one column list serve endpoints that return different fields. A column without any `field` is always shown.
- A header shows `⇅` when sortable, `↑` or `↓` on the active column, and the active header is bold. The first click on a column sorts ascending, the next one descending, then it alternates. Sorting returns to page 1.
- `sortName` must be registered with `searchAndSortOn`. A sortable column that sends an unregistered name makes the request fail.
- **`render: true` writes HTML into the page.** Always pass user data through `useXssProtector().toPlainText(...)` (or `toDisplayHtml` for allowed markup) first. The composable uses a global `DOMPurify`, which the application has to provide.

### 10.3 Events, methods and slots

| Name | Kind | Meaning |
|------|------|---------|
| `selected` | event | The chosen row (a single row object, or `null` when cleared). With `multiSelectable`, an array of rows. |
| `loaded` | event | The whole response object after each successful load; `null` when `url` is empty |
| `error` | event | An `Error` with an optional `status` (HTTP status) when loading fails |
| `reload()` | method | Reloads the current state and clears the selection |
| `focus()` / `blur()` | method | Moves keyboard focus to / away from the table |
| `before-search` | slot | Content before the search box (typically an "add" button) |
| `after-search` | slot | Content after it (reload, view, edit, delete buttons) |

A table with toolbar, selection and reload:

```vue
<script setup>
const config = useRuntimeConfig()
const url = `${config.public.API_BASE}/api/books?`
const tableRef = ref()
const selected = ref(null)

const columns = [
  { field: 'bookId', label: 'ID', sortable: true },
  { field: 'title', label: 'Title', sortable: true },
  { field: 'author', label: 'Author', sortable: true }
]

function reload() {
  selected.value = null
  tableRef.value?.reload()
}

function edit() {
  navigateTo(`/books/${selected.value.bookId}`)
}

onMounted(() => tableRef.value?.focus())
</script>

<template>
  <DataTable
    ref="tableRef"
    :url="url"
    :columns="columns"
    default-sort="bookId"
    default-order="DESC"
    row-key="bookId"
    :page-size="15"
    empty-text="No books found"
    @selected="selected = $event"
  >
    <template #before-search>
      <UButton icon="i-lucide-plus" to="/books/new" />
    </template>

    <template #after-search>
      <UButton variant="outline" icon="i-lucide-refresh-ccw" @click="reload" />
      <UButton icon="i-lucide-pencil" :disabled="!selected" @click="edit" />
    </template>
  </DataTable>
</template>
```

Reading the events:

```vue
<script setup>
function onLoaded(data) {
  // data is the whole response; it is null when the url is empty
  console.log('total rows:', data?.total)
}

function onError(error) {
  // error.status holds the HTTP status when the server answered
  console.error('loading failed', error.status)
}
</script>

<template>
  <DataTable
    :url="url"
    :columns="columns"
    default-sort="bookId"
    row-key="bookId"
    @loaded="onLoaded"
    @error="onError"
    @selected="selected = $event"
  />
</template>
```

**Selection.** Single selection is a radio button in the first column; clicking anywhere on a row selects it as well. The selection survives page changes, sorting and searching, and is cleared by `reload()` and when `url` or `pageSize` changes. Call `reload()` after every action that changes data, and set the selected variable back to `null` so that buttons become disabled again.

**Multiple selection.** With `multiSelectable`, rows are checkboxes and `selected` delivers an array that keeps rows from other pages as well:

```vue
<script setup>
const selectedRows = ref([])
const tableRef = ref()

const ids = computed(() => selectedRows.value.map(row => row.bookId))

async function removeSelected() {
  await $fetch('/api/books/delete', { method: 'POST', body: ids.value })
  tableRef.value?.reload()
}
</script>

<template>
  <DataTable
    ref="tableRef"
    :url="url"
    :columns="columns"
    multi-selectable
    default-sort="bookId"
    row-key="bookId"
    @selected="selectedRows = $event"
  >
    <template #after-search>
      <UButton color="error" icon="i-lucide-trash" :disabled="!ids.length" @click="removeSelected">
        Delete selected ({{ ids.length }})
      </UButton>
    </template>
  </DataTable>
</template>
```

**Keyboard.** Once the table has focus (click it, or call `focus()` after mount), `↑`/`↓` move between rows and `Space` selects the highlighted one. Because the application is right-to-left, `←` goes to the **next** page and `→` to the previous one.

**Row colors.** `rowStyle` returns inline CSS. Selected and highlighted rows keep their own colors on top:

```vue
<script setup>
// The returned string is applied as the inline style of every cell of the row.
function rowStyle(row) {
  return row.price > 50 ? 'background-color: #fff4d6; color: #5a4300;' : ''
}
</script>

<template>
  <DataTable :url="url" :columns="columns" :row-style="rowStyle" default-sort="bookId" row-key="bookId" />
</template>
```

### 10.4 Two linked tables

Setting `url` to an empty string empties a table without an error, and changing `url` loads it again from page 1. That makes master–detail screens simple: the detail's address is computed from the master's selection.

```vue
<script setup>
const config = useRuntimeConfig()
const base = config.public.API_BASE

const publisher = ref(null)
const booksUrl = computed(() =>
  publisher.value ? `${base}/api/books/by-publisher?publisherId=${publisher.value.publisherId}&` : ''
)

const publisherColumns = [
  { field: 'publisherId', label: 'ID', sortable: true },
  { field: 'name', label: 'Name', sortable: true }
]

const bookColumns = [
  { field: 'bookId', label: 'ID', sortable: true },
  { field: 'title', label: 'Title', sortable: true },
  { field: 'price', label: 'Price', sortable: true }
]
</script>

<template>
  <h2>Publishers</h2>
  <DataTable
    :url="`${base}/api/publishers?`"
    :columns="publisherColumns"
    default-sort="publisherId"
    row-key="publisherId"
    @selected="publisher = $event"
  />

  <h2>Books of the selected publisher</h2>
  <DataTable
    :url="booksUrl"
    :columns="bookColumns"
    default-sort="bookId"
    row-key="bookId"
  />
</template>
```

The detail endpoint is the one from section 4.4. Note that its URL ends with `&`, because it already has a parameter.

### 10.5 A table on PageDataSQL

Register the column expressions on the server and use `sortName` on the client (section 7). Set `defaultSort` to a registered expression such as `b.book_id`, and `row-key` to the label that the rows carry, `book_id`.

## 11. DataCard

`DataCard` shows the same data as a grid of cards. Instead of pages, it loads the next batch when the user scrolls to the bottom (an `IntersectionObserver` watches a marker under the last card). It has no columns and no selection; the card design is a slot.

### 11.1 Props

| Prop | Type | Default | Meaning |
|------|------|---------|---------|
| `url` | String, required | – | Endpoint, ending in `?` or `&` |
| `pageSize` | Number | `10` | Cards per load, sent as `rows` |
| `itemKey` | String | `'id'` | Unique field of each item, used as the list key |
| `cols` | Object | `{}` | Number of columns by screen width (11.2) |
| `emptyText` | String | `'داده‌ای وجود ندارد'` | Text when there are no items |
| `v-model:sort` | String | none | Field to sort by. **Provide it.** |
| `v-model:order` | String | none | `'ASC'` or `'DESC'` |

> **Unlike DataTable, DataCard has no default sort.** If `sort` is not bound, the request goes out without it and the server rejects it ("sort is mandatory"). Always bind both models to refs with starting values.

### 11.2 Columns by screen width

`cols` maps breakpoints to a column count. `sm` is the base (below 768 px); the others apply from their minimum width upwards. A missing breakpoint inherits the nearest smaller one, and everything defaults to 1.

| Key | From | 
|-----|------|
| `sm` | 0 px (base) |
| `md` | 768 px |
| `lg` | 1024 px |
| `xl` | 1280 px |
| `2xl` | 1536 px |

```vue
<DataCard
  v-model:sort="sort"
  v-model:order="order"
  :url="url"
  item-key="bookId"
  :cols="{ sm: 1, lg: 3, '2xl': 6 }"
/>
```

That example gives 1 column below 1024 px (`md` inherits the value of `sm`), 3 from 1024 px (`xl` inherits the value of `lg`) and 6 from 1536 px.

### 11.3 A basic grid

Two selects change `sort` and `order`; changing either one reloads from the beginning:

```vue
<script setup>
const config = useRuntimeConfig()
const url = `${config.public.API_BASE}/api/books?`

const sort = ref('title')
const order = ref('ASC')

const sortOptions = [
  { label: 'Title', value: 'title' },
  { label: 'Price', value: 'price' }
]
const orderOptions = [
  { label: 'Ascending', value: 'ASC' },
  { label: 'Descending', value: 'DESC' }
]
</script>

<template>
  <DataCard
    v-model:sort="sort"
    v-model:order="order"
    :url="url"
    item-key="bookId"
    :page-size="12"
    :cols="{ sm: 1, md: 2, lg: 4 }"
    empty-text="No books found"
  >
    <template #after-search>
      <USelect v-model="sort" :items="sortOptions" />
      <USelect v-model="order" :items="orderOptions" />
    </template>

    <template #default="{ item }">
      <UCard>
        <h3 class="font-bold">{{ item.title }}</h3>
        <p class="text-sm text-gray-500">{{ item.author }}</p>
        <p>{{ item.price }} USD</p>
      </UCard>
    </template>
  </DataCard>
</template>
```

### 11.4 Behavior

- Typing in the search box waits 500 ms, then reloads from the beginning.
- Changing `url`, `pageSize`, `sort` or `order` reloads from the beginning.
- Loading ends when the number of loaded items reaches `total`.
- If the first load fails, an error message is shown. If a later batch fails, a **Retry** button appears under the cards, and the failed page is requested again.
- If the first batch is too short to fill the screen, the next one is requested automatically.

### 11.5 Slots, events and methods

| Name | Kind | Meaning |
|------|------|---------|
| default | slot | Receives `{ item }` and renders one card |
| `before-search`, `after-search` | slot | Content around the search box |
| `error` | event | An `Error` when loading fails |
| `reload()` | method | Reloads from the beginning |
| `focus()` | method | Focuses the search box |

After an action that changes data (delete, edit) call `reload()`:

```vue
<script setup>
const cardsRef = ref()

async function remove(book) {
  await $fetch(`/api/books/${book.bookId}`, { method: 'DELETE' })
  cardsRef.value?.reload()
}
</script>

<template>
  <DataCard ref="cardsRef" v-model:sort="sort" v-model:order="order" :url="url" item-key="bookId">
    <template #default="{ item }">
      <UCard>
        <h3>{{ item.title }}</h3>
        <UButton color="error" block @click="remove(item)">Delete</UButton>
      </UCard>
    </template>
  </DataCard>
</template>
```

Images and other heavy content belong inside the slot; native lazy loading keeps the scroll smooth:

```vue
<template #default="{ item }">
  <UCard>
    <img :src="`/covers/${item.bookId}.jpg`" :alt="item.title" loading="lazy" class="h-40 w-full object-cover">
    <h3 class="font-bold">{{ item.title }}</h3>
  </UCard>
</template>
```

## 12. In practice: a book catalog

This chapter builds one complete feature: a catalog with a table page for staff and a card page for visitors, both served by a single endpoint.

**1. Entities.** `Publisher` and `Book` from chapter 2.

**2. The endpoint.** It combines everything from the earlier chapters: it limits the page size, registers sortable fields, hides inactive books for good, accepts an optional price limit, and shapes the rows.

```java
import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import org.j2os.platform.page2.PageDataEntity;
import org.j2os.platform.page2.PageDataResultFilter;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import java.util.HashMap;
import java.util.Map;

@RestController
@RequestMapping("/api/books")
public class BookAPI {

    @PersistenceContext
    private EntityManager entityManager;

    @GetMapping
    public Map<String, Object> list(@RequestParam Map<String, Object> params) {
        // 1. Never trust the page size that comes from the browser.
        Map<String, Object> grid = new HashMap<>(params);
        int rows = Integer.parseInt(String.valueOf(grid.getOrDefault("rows", "10")));
        grid.put("rows", Math.min(Math.max(rows, 1), 100));

        // 2. Build a new PageDataEntity for every request.
        PageDataEntity page = new PageDataEntity(entityManager)
                .searchAndSortOn("bookId", "title", "author", "price", "publisher.name")
                .restrict("active", "=", true);

        // 3. Optional filter that comes from the URL.
        Object maxPrice = grid.get("maxPrice");
        if (maxPrice != null) {
            page.where("price", "<=", maxPrice, value -> Integer.valueOf(value.toString()));
        }

        // 4. Shape the rows before they leave the server.
        return new PageDataResultFilter<Book>(page.getResult(Book.class, grid))
                .remove("active")
                .put("priceLabel", book -> book.getPrice() + " USD")
                .getResult();
    }
}
```

The comments mark the four decisions:

1. The browser decides `rows`, so the server caps it. Without a cap, `rows=1000000` is answered.
2. A fresh `PageDataEntity` per request.
3. `restrict` for the rule that must always hold; `where` with a transformer for the optional one.
4. The filter hides the `active` flag and adds `priceLabel`.

**3. The table page.** `priceLabel` is displayed, but sorting must use the real `price` column, hence `sortName`:

```vue
<script setup>
const config = useRuntimeConfig()
const url = `${config.public.API_BASE}/api/books?`
const tableRef = ref()
const selected = ref(null)

const columns = [
  { field: 'bookId', label: 'ID', sortable: true },
  { field: 'title', label: 'Title', sortable: true },
  { field: 'author', label: 'Author', sortable: true },
  { field: 'priceLabel', label: 'Price', sortable: true, sortName: 'price' },
  {
    field: 'publisher',
    label: 'Publisher',
    sortable: true,
    sortName: 'publisher.name',
    processor: row => row.publisher?.name ?? ''
  }
]

function reload() {
  selected.value = null
  tableRef.value?.reload()
}
</script>

<template>
  <h1>Books</h1>

  <DataTable
    ref="tableRef"
    :url="url"
    :columns="columns"
    default-sort="bookId"
    row-key="bookId"
    @selected="selected = $event"
  >
    <template #after-search>
      <UButton variant="outline" icon="i-lucide-refresh-ccw" @click="reload" />
      <UButton icon="i-lucide-eye" :disabled="!selected" :to="selected ? `/books/${selected.bookId}` : undefined" />
    </template>
  </DataTable>
</template>
```

**4. The card page.** Sorting by publisher uses the nested path directly:

```vue
<script setup>
const config = useRuntimeConfig()
const url = `${config.public.API_BASE}/api/books?`

const sort = ref('title')
const order = ref('ASC')

const sortOptions = [
  { label: 'Title', value: 'title' },
  { label: 'Price', value: 'price' },
  { label: 'Publisher', value: 'publisher.name' }
]
const orderOptions = [
  { label: 'Ascending', value: 'ASC' },
  { label: 'Descending', value: 'DESC' }
]
</script>

<template>
  <h1>Book catalog</h1>

  <DataCard
    v-model:sort="sort"
    v-model:order="order"
    :url="url"
    item-key="bookId"
    :page-size="12"
    :cols="{ sm: 1, md: 2, lg: 4 }"
  >
    <template #after-search>
      <USelect v-model="sort" :items="sortOptions" />
      <USelect v-model="order" :items="orderOptions" />
    </template>

    <template #default="{ item }">
      <UCard>
        <h3 class="font-bold">{{ item.title }}</h3>
        <p class="text-sm text-gray-500">{{ item.author }} · {{ item.publisher?.name }}</p>
        <p class="mt-2">{{ item.priceLabel }}</p>
      </UCard>
    </template>
  </DataCard>
</template>
```

**5. Check it.** Open the table page and verify, in this order: the first page loads; typing in the search box returns to page 1 after a short pause; clicking *Price* sorts ascending, then descending; the pagination moves between pages; selecting a row enables the toolbar buttons. Then open the card page and scroll to the bottom to load the next batch.

---

## Appendix A. Quick reference

### A.1 Common to PageDataEntity, PageDataJPQL, PageDataSQL, PageDataList

| Method | Description |
|--------|-------------|
| `searchAndSortOn(String... fields)` | Registers the searchable and sortable fields. Names are validated (letters, digits, `_`, dots). Repeating a field has no effect. Returns `this`. |

Result map keys: `total` (Long), `rows` (List), `page` (Integer), `size` (Integer), `search` (String).

### A.2 PageDataEntity

| Method | Description |
|--------|-------------|
| `PageDataEntity(EntityManager)` | Creates the instance |
| `where(field, operator, value)` | Adds a condition (joined with AND) |
| `where(field, operator, value, transformer)` | Same, with `Function<Object,Object>` applied to the value before it is used |
| `and(...)` | Same as `where` (both overloads) |
| `or(...)` | Adds a condition joined with OR (both overloads) |
| `restrict(field, operator, value)` | Adds a condition that is always ANDed with the whole query |
| `getResult(Class<T> entityClass, Map<String,Object> gridParams)` | Runs the page and count queries |

Operators: `=`, `!=`, `<>`, `>`, `>=`, `<`, `<=`, `LIKE`, `NOT LIKE`, `IS NULL`, `IS NOT NULL`.

### A.3 PageDataJPQL

| Method | Description |
|--------|-------------|
| `PageDataJPQL(EntityManager)` | Creates the instance |
| `getResult(jpql, countJpql, alias, entityClass, queryParams, gridParams)` | Runs the two queries with search, sort and paging added. `queryParams` may be `null`. |

### A.4 PageDataSQL

| Method | Description |
|--------|-------------|
| `PageDataSQL(Connection)` | Creates the instance (the connection must not be `null`) |
| `getResult(sql, sqlParams, pageParams)` | Runs the SQL; rows are maps |
| `getResult(sql, primaryKey, sqlParams, pageParams)` | Same, and orders equal sort values by `primaryKey` |

### A.5 PageDataList

| Method | Description |
|--------|-------------|
| `PageDataList()` | Creates the instance |
| `where`, `and`, `or`, `restrict` | As above |
| `getResult(List<T> data, Map<String,Object> gridParams)` | Filters, searches, sorts and pages the list |

### A.6 PageDataResultFilter&lt;T&gt;

| Method | Description |
|--------|-------------|
| `PageDataResultFilter(Map<String,Object> result)` | Wraps a result map (must not be `null`) |
| `remove(path)` | Removes a field |
| `empty(path)` | Sets it to `""` |
| `mask(path)` | Sets it to `********` |
| `put(path, Function<T,Object>)` | Adds or replaces a field |
| `put(path, Class<V>, BiFunction<T,V,V>)` | Changes a field using its current value converted to `V` |
| `getResult()` | Applies the rules and returns the result map. Callable once. |

### A.7 DataTable

Props: `url`, `columns`, `defaultSort`, `defaultOrder`, `rowKey`, `pageSize`, `emptyText`, `rowStyle`, `multiSelectable`.
Events: `selected`, `loaded`, `error`.
Methods: `reload()`, `focus()`, `blur()`.
Slots: `before-search`, `after-search`.
Column keys: `field`, `label`, `sortable`, `sortName`, `processor`, `render`.

### A.8 DataCard

Props: `url`, `pageSize`, `itemKey`, `cols`, `emptyText`.
Models: `sort`, `order`.
Events: `error`.
Methods: `reload()`, `focus()`.
Slots: `before-search`, `after-search`, default (`{ item }`).

---

## Appendix B. Failure handling

**On the server.** page2 throws; the application decides what the browser sees.

| Situation | What is thrown |
|-----------|----------------|
| `gridParams` is `null`, or has no `sort` | `IllegalArgumentException` |
| `sort` is not registered | `RuntimeException` ("… is not sortable") |
| `page` or `rows` is not a number | `IllegalArgumentException` |
| Field name fails validation | `IllegalArgumentException` |
| Operator is not supported | `IllegalArgumentException` |
| Field of a path does not exist (PageDataList), or is not a relation (PageDataEntity) | `IllegalArgumentException` |
| Class is not a managed entity | `IllegalArgumentException` |
| Reserved parameter name used / SQL parameter missing | `IllegalArgumentException` |
| SQL execution fails | `RuntimeException` wrapping the `SQLException` |
| `PageDataResultFilter.getResult()` called twice | `IllegalStateException` |

Without special handling, Spring answers such errors with an HTTP 500 and the components show their error state. A `@RestControllerAdvice` that maps `IllegalArgumentException` to HTTP 400 makes the difference visible in logs and monitoring.

**In the browser.**

| Situation | DataTable | DataCard |
|-----------|-----------|----------|
| Loading | spinner and "loading" text | same |
| Request fails or HTTP status is not 2xx | warning message, empty table, `error` event | warning if nothing is loaded yet; if a later batch fails, the cards stay and a retry button appears; `error` event |
| No rows | `emptyText` | `emptyText` |
| A newer request starts | the older one is canceled | same |
| `url` is empty | table is cleared, `loaded` receives `null` | not handled; always pass an address |

Requests are sent with `credentials: 'include'`, so cookie-based sessions work out of the box. Both components contain a commented block showing how to send a bearer token instead.

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Forgetting `?` at the end of `url`.** The query string is appended without a separator: `…/api/books` + `page=1…` becomes `…/api/bookspage=1`. Use `…/api/books?` or `…?publisherId=3&`.
- **Sorting by a field that is not registered.** Every `field` or `sortName` of a sortable column must appear in `searchAndSortOn`.
- **Not binding `sort` on DataCard.** See the note in 11.1.
- **Sharing a page2 instance** between requests. Always create it inside the handler.
- **Sending text to a numeric comparison.** `where("price", "<", "50")` compares a number with text. Use a transformer.
- **A `rowKey` that is not in the row.** Selection works on this field; with a missing or duplicate key, rows are selected together.
- **`render: true` with raw data.** This is an XSS hole. Sanitize.
- **Aliases in PageDataSQL.** Register real expressions (`p.name`), not select-list aliases.

### C.2 Limits

- `PageDataEntity` joins only single-valued relations. Lists need `PageDataJPQL`.
- A search with many registered fields and many words generates one condition per field per word; keep the registered list short.
- Every request runs two queries (page and count).
- The generated SQL targets PostgreSQL. `LIKE` is case-sensitive in `where` conditions there, while the free-text search lower-cases both sides.
- `PageDataList` handles the entire list on every request.
- `DataTable` and `DataCard` have fixed Persian texts for the search box and status messages (Appendix D).

### C.3 Tuning

- Cap `rows` on the server (see the endpoint in chapter 12).
- Register few fields, and prefer indexed ones. A search on a long text column cannot use an index.
- Put row-visibility rules into `restrict`; they also shrink the count query.
- Prefer `PageDataEntity` over `PageDataList(repository.findAll())` for database data: the database does the paging and the full table is not loaded.
- Remove fields the screen does not show with `PageDataResultFilter.remove`, so less data crosses the network.
- Keep the card slot light. Big images should load lazily.

## Appendix D. Message reference

**Server messages** (English; exception type in brackets)

| Message | When |
|---------|------|
| `gridParams must not be null (sort is mandatory)` [IllegalArgumentException] | `PageDataEntity`, `PageDataList` got a `null` grid map |
| `gridParams must include a 'sort' field (sort is mandatory)` [IllegalArgumentException] | no `sort` in the request |
| `<field> is not sortable` [RuntimeException] | `sort` names a field that is not registered |
| `gridParams['page'] must be a valid integer, got: '<value>'` [IllegalArgumentException] | `page` or `rows` is not a number (the key name changes accordingly) |
| `Field name is empty` [IllegalArgumentException] | a registered or filtered field is `null` or empty |
| `Invalid/unsafe field name: <field>` [IllegalArgumentException] | a field name has characters other than letters, digits, `_` and dots |
| `Unsupported operator: <operator>` [IllegalArgumentException] | operator is not in the supported list |
| `data must not be null` [IllegalArgumentException] | `PageDataList.getResult` got `null` data |
| `PageDataList: '<name>' is not a field of the items` [IllegalArgumentException] | a registered or filtered field does not exist on any item |
| `PageDataList: the values of '<field>' cannot be compared with each other` [IllegalArgumentException] | the sort field holds values that cannot be ordered |
| `PageDataEntity: '<part>' in field path '<path>' is not an attribute of <class>` [IllegalArgumentException] | nested path names an unknown attribute |
| `PageDataEntity: '<part>' in field path '<path>' is not a single-valued association or embedded attribute` [IllegalArgumentException] | nested path goes through a collection or a simple field |
| `PageDataEntity: '<class>' is not a JPA entity managed by this EntityManager's persistence unit` [IllegalArgumentException] | wrong class passed to `getResult` |
| `queryParams must not contain the reserved parameter name '<name>' (used internally by PageDataJPQL for search)` [IllegalArgumentException] | `search_N` used in `queryParams` |
| `sqlParams must not contain the reserved parameter name '<name>' (used internally by PageDataSQL for paging/search)` [IllegalArgumentException] | `search_N`, `__limit` or `__offset` used in `sqlParams` |
| `PageDataSQL: missing value for named parameter ':<name>'` [IllegalArgumentException] | the SQL uses a parameter that is not in the map |
| `PageDataSQL: query execution failed` [RuntimeException] | the database rejected the SQL; the cause is attached |
| `PageDataResultFilter.getResult() has already been called on this instance; create a new instance for each result/request instead of reusing this one` [IllegalStateException] | the filter is used twice |
| `jpql must not be null`, `countJpql must not be null`, `sql must not be null`, `connection must not be null` [NullPointerException] | a required argument is `null` |

**Messages shown by the components** (Persian)

| Text | Meaning | Where |
|------|---------|-------|
| `جستجو...` | "Search…", placeholder of the search box | DataTable, DataCard |
| `در حال بارگذاری...` | "Loading…" | DataTable, DataCard |
| `خطا در دریافت اطلاعات وجود دارد` | "An error occurred while fetching data" | DataTable, DataCard |
| `داده‌ای وجود ندارد` | "No data" (default `emptyText`) | DataTable, DataCard |
| `گزینه` | "Option": header of the selection column | DataTable |
| `در حال بارگذاری موارد بیشتر...` | "Loading more…" | DataCard |
| `خطا در دریافت موارد بیشتر` | "Error while fetching more" | DataCard |
| `تلاش مجدد` | "Try again", retry button | DataCard |
