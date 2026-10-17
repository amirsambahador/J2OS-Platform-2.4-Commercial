# jreport

*J2OS Platform 2.4*

Sooner or later every business application is asked for a *report*: a list of customers as a PDF, the sales of last month as an Excel sheet, an invoice as a Word document. The data is already in a database or in a list in memory. What is missing is the work around it: laying the data out on pages, producing the file format, sending it to the browser as a download or saving it on disk.

**jreport** (`org.j2os.platform.jreport`) does this work. It is built on *JasperReports*, a widely used Java reporting engine, and on *DynamicReports*, a library that describes a report in code. You give it the data and the format, and it gives you a finished file.

```java
// rows from a SQL query, columns chosen in code
DynamicReport.generateToResponse(response, connection, "customers", ReportType.PDF, "Customers",
        "select name, city from customer", List.of("name", "city"), List.of("Name", "City"),
        null, HorizontalTextAlignment.LEFT, null);

// a hand-designed template, filled from a query or from a list
TemplateReport.generateToFile("out/sales.xlsx", connection, "/reports/sales.jrxml", ReportType.XLSX, params);
EntityTemplateReport.generateToFile("out/list.pdf", "/reports/list.jrxml", ReportType.PDF, params, rows);
```

The module has four types:

- `ReportType` names the three output formats, PDF, DOCX and XLSX, and writes a finished report either to an HTTP response or to a file.
- `DynamicReport` builds a simple table from a SQL query and a list of columns. No template is needed.
- `TemplateReport` fills a template that you designed by hand (a `.jrxml` file) with the result of a query.
- `EntityTemplateReport` fills a hand-designed template with a list of records that are already in memory, with no database at all.

This guide goes step by step: each chapter has a short program and the output it prints. The parts that only make sense inside a web application are shown as Spring snippets. The appendices are for quick reference.

## Contents

1. [Three ways to make a report](#1-three-ways-to-make-a-report)
2. [Output formats](#2-output-formats)
3. [Saving a report as a file](#3-saving-a-report-as-a-file)
4. [Sending a report as a download](#4-sending-a-report-as-a-download)
5. [A table from a query: DynamicReport](#5-a-table-from-a-query-dynamicreport)
6. [Your own design: TemplateReport](#6-your-own-design-templatereport)
7. [A template filled from a list: EntityTemplateReport](#7-a-template-filled-from-a-list-entitytemplatereport)
8. [Putting it together: customer reports every month](#8-putting-it-together-customer-reports-every-month)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The module needs JasperReports, DynamicReports (for `DynamicReport` only), the Jakarta Servlet API and SLF4J on the classpath. The programs in this guide use `EntityTemplateReport` and the checks of `DynamicReport`, so they need no database.

---

## 1. Three ways to make a report

The three engines differ in where the *layout* comes from and where the *data* comes from.

| Engine | Layout | Data | Use it when |
|--------|--------|------|-------------|
| `DynamicReport` | built by the module: a styled table | a SQL query run on a JDBC connection | you need a plain list and do not want to design anything |
| `TemplateReport` | your `.jrxml` template | the query inside the template, run on a JDBC connection | the report has a special design and its data comes from the database |
| `EntityTemplateReport` | your `.jrxml` template | a `List` of `Map` objects in memory | the data is already in your program, or does not come from a database |

All three finish in the same way. Each has two methods with the same shape: `generateToResponse(...)` sends the report to an HTTP response as a download, and `generateToFile(...)` saves it on disk. In both, you choose the format with a `ReportType`. The first parameters differ between the engines, as the following chapters show.

A *`.jrxml` template* is an XML file that describes the page: where each piece of text goes, which fields come from the data, and which parameters the caller may pass. It is designed once, usually with a visual tool such as Jaspersoft Studio, and kept in the application as a resource. jreport does not look inside it. It only compiles it and fills it.

## 2. Output formats

`ReportType` is an enum with three values.

```java
import org.j2os.platform.jreport.report.ReportType;

public class ReportFormats {

    public static void main(String[] args) {
        for (String code : new String[]{"pdf", " PDF ", "doc", "docx", "xls", "XLSX"}) {
            ReportType type = ReportType.fromCode(code);
            System.out.println("'" + code + "' -> " + type + ", ." + type.getExtension() + ", " + type.getMimeType());
        }

        try {
            ReportType.fromCode("csv");
        } catch (IllegalArgumentException e) {
            System.out.println(e.getMessage());
        }
        try {
            ReportType.fromCode(null);
        } catch (IllegalArgumentException e) {
            System.out.println(e.getMessage());
        }

        System.out.println(ReportType.deriveDisplayName("/reports/customer-list.jrxml"));
        System.out.println(ReportType.deriveDisplayName("C:\\templates\\invoice.final.jrxml"));
        System.out.println(ReportType.deriveDisplayName("invoice"));
    }
}
```

```text
'pdf' -> PDF, .pdf, application/pdf
' PDF ' -> PDF, .pdf, application/pdf
'doc' -> DOCX, .docx, application/vnd.openxmlformats-officedocument.wordprocessingml.document
'docx' -> DOCX, .docx, application/vnd.openxmlformats-officedocument.wordprocessingml.document
'xls' -> XLSX, .xlsx, application/vnd.openxmlformats-officedocument.spreadsheetml.sheet
'XLSX' -> XLSX, .xlsx, application/vnd.openxmlformats-officedocument.spreadsheetml.sheet
Type not supported: csv
Type not supported: null
customer-list
invoice.final
invoice
```

| Value | File extension | MIME type |
|-------|----------------|-----------|
| `PDF` | `pdf` | `application/pdf` |
| `DOCX` | `docx` | the Word type of the Open XML family |
| `XLSX` | `xlsx` | the Excel type of the Open XML family |

A *MIME type* is the name that a web browser uses to know what kind of file it receives.

- `fromCode(code)` turns a short text into a `ReportType`. It is meant for a request parameter such as `?format=pdf`. Capital letters and spaces around the text do not matter. `doc` and `docx` both give `DOCX`, and `xls` and `xlsx` both give `XLSX`.
- Any other text, and `null`, gives `IllegalArgumentException` with the message `Type not supported: <code>`. In a web application this is the place to answer with status 400.
- `getExtension()` and `getMimeType()` return the values of the table.
- `deriveDisplayName(path)` removes the folder and the extension from a path and keeps the rest. The engines use it to name a download after its template. Only the last dot counts, so `invoice.final.jrxml` gives `invoice.final`, and a name without a dot stays as it is. A `null` path gives `IllegalArgumentException`.

## 3. Saving a report as a file

`generateToFile` writes a report on disk. This is the right choice for reports that a scheduled job prepares for later, or that another program picks up.

```java
import org.j2os.platform.jreport.jasper.EntityTemplateReport;
import org.j2os.platform.jreport.jasper.TemplateReport;
import org.j2os.platform.jreport.report.ReportType;

import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;
import java.util.Map;

public class SaveToFile {

    public static void main(String[] args) throws Exception {
        List<Map<String, Object>> customers = List.of(
                Map.of("name", "Alice Berg", "city", "Oslo"),
                Map.of("name", "Omar Haddad", "city", "Cairo"),
                Map.of("name", "Mei Tanaka", "city", "Kyoto"));

        Path folder = Path.of("reports-out", "2026", "october");
        Path target = folder.resolve("customers.pdf");

        EntityTemplateReport.generateToFile(target.toString(), "/reports/customer-list.jrxml",
                ReportType.PDF, Map.of("title", "Customers"), customers);
        System.out.println("folder created : " + Files.isDirectory(folder));
        System.out.println("file written   : " + (Files.size(target) > 0));

        EntityTemplateReport.generateToFile(target.toString(), "/reports/customer-list.jrxml",
                ReportType.PDF, null, customers.subList(0, 1));
        try (var files = Files.list(folder)) {
            System.out.println("files in folder: " + files.map(f -> f.getFileName().toString()).toList());
        }

        try {
            EntityTemplateReport.generateToFile(target.toString(), "/reports/missing.jrxml",
                    ReportType.PDF, null, customers);
        } catch (IllegalArgumentException e) {
            System.out.println(e.getMessage());
        }
        try {
            EntityTemplateReport.generateToFile(target.toString(), "reports/customer-list.jrxml",
                    ReportType.PDF, null, customers);
        } catch (IllegalArgumentException e) {
            System.out.println(e.getMessage());
        }
        try {
            TemplateReport.generateToFile(target.toString(), null, "/reports/missing.jrxml",
                    ReportType.XLSX, null);
        } catch (IllegalArgumentException e) {
            System.out.println(e.getMessage());
        }
    }
}
```

```text
folder created : true
file written   : true
files in folder: [customers.pdf]
No .jrxml found on the classpath at /reports/missing.jrxml
No .jrxml found on the classpath at reports/customer-list.jrxml
No .jrxml found on the classpath at /reports/missing.jrxml
```

- The folders of the path are created when they do not exist. The first call creates `reports-out/2026/october`.
- The report is first written to a temporary file in the **same folder**, and the temporary file is moved to the final name only when the export has finished. If the export fails, the temporary file is deleted and a file that was already at the path stays as it was. A reader therefore never meets a half-written report. The second call replaces the first report, and the folder holds one file.
- The move is done in one step where the file system allows it, and as an ordinary replace where it does not.
- The parameters may be `null` when the template needs none.
- A template that cannot be found gives `IllegalArgumentException` with the message `No .jrxml found on the classpath at <path>`. The message repeats the path that you gave. This is the point to remember: **the template path must begin with `/`**. Without it, Java looks for the file in the package of the module itself, and does not find it.

## 4. Sending a report as a download

`generateToResponse` writes the report to an `HttpServletResponse`. The browser receives it as an attachment and offers to save it.

```java
import jakarta.servlet.ServletOutputStream;
import jakarta.servlet.http.HttpServletResponse;
import org.j2os.platform.jreport.jasper.EntityTemplateReport;
import org.j2os.platform.jreport.report.ReportType;

import java.io.ByteArrayOutputStream;
import java.lang.reflect.Proxy;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

public class DownloadHeaders {

    public static void main(String[] args) throws Exception {
        Map<String, String> seen = new LinkedHashMap<>();
        ByteArrayOutputStream body = new ByteArrayOutputStream();
        ServletOutputStream stream = new ServletOutputStream() {
            public void write(int b) {
                body.write(b);
            }
        };

        HttpServletResponse response = (HttpServletResponse) Proxy.newProxyInstance(
                DownloadHeaders.class.getClassLoader(), new Class<?>[]{HttpServletResponse.class},
                (proxy, method, a) -> {
                    switch (method.getName()) {
                        case "setContentType" -> seen.put("Content-Type", (String) a[0]);
                        case "setContentLength" -> seen.put("Content-Length", String.valueOf(a[0]));
                        case "setHeader" -> seen.put((String) a[0], (String) a[1]);
                        case "getOutputStream" -> {
                            return stream;
                        }
                        default -> {
                        }
                    }
                    return null;
                });

        EntityTemplateReport.generateToResponse(response, "/reports/customer-list.jrxml", ReportType.XLSX,
                Map.of("title", "Customers"), List.of(Map.of("name", "Alice Berg", "city", "Oslo")));
        System.out.println("Content-Type: " + seen.get("Content-Type"));
        System.out.println("Content-Disposition: " + seen.get("Content-Disposition"));
        System.out.println("Content-Length equals the body size: " + seen.get("Content-Length").equals(String.valueOf(body.size())));

        seen.clear();
        EntityTemplateReport.generateToResponse(response, "/reports/لیست-مشتریان.jrxml", ReportType.PDF,
                null, List.of());
        System.out.println(seen.get("Content-Disposition"));
    }
}
```

```text
Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet
Content-Disposition: attachment; filename*=UTF-8''customer-list.xlsx
Content-Length equals the body size: true
attachment; filename*=UTF-8''%D9%84%DB%8C%D8%B3%D8%AA-%D9%85%D8%B4%D8%AA%D8%B1%DB%8C%D8%A7%D9%86.pdf
```

The module sets three things on the response and then writes the bytes:

- `Content-Type` is the MIME type of the format.
- `Content-Length` is the exact size of the report, so the browser can show the progress of the download. The module needs the size, so it builds the whole report in memory first and sends it afterwards.
- `Content-Disposition` is `attachment; filename*=UTF-8''<name>.<extension>`. The name is the display name of the template (chapter 2) with the extension of the format. It is encoded in UTF-8, so a name in any script is delivered correctly. The second program has a template with a Persian name, and the last line shows how the browser receives it.

The response must be fresh. Call the method before anything else has been written to it. The stream of the response is closed by the module when the report is written.

A request handler that lets the user choose the format uses `fromCode` for the parameter and answers a wrong value with a clear message:

```java
@ExceptionHandler(IllegalArgumentException.class)
public ResponseEntity<String> badRequest(IllegalArgumentException e) {
    return ResponseEntity.badRequest().body(e.getMessage());   // for example "Type not supported: csv"
}
```

## 5. A table from a query: DynamicReport

`DynamicReport` needs no template. You give it a SQL query, the names of the columns that the query returns, and the titles to show above them. The module lays out an A4 page in portrait orientation with a header row, striped body rows and a page number at the bottom.

```java
@RestController
public class CustomerReportController {

    private final DataSource dataSource;

    public CustomerReportController(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @GetMapping("/reports/customers")
    public void customers(@RequestParam String city,
                          @RequestParam(defaultValue = "pdf") String format,
                          HttpServletResponse response) throws Exception {
        try (Connection connection = dataSource.getConnection()) {
            DynamicReport.generateToResponse(
                    response,
                    connection,
                    "customers-" + city,
                    ReportType.fromCode(format),
                    "Customers in " + city,
                    "select name, city from customer where city = $P{city} order by name",
                    List.of("name", "city"),
                    List.of("Name", "City"),
                    "/images/logo.png",
                    HorizontalTextAlignment.LEFT,
                    Map.of("city", city));
        }
    }
}
```

The method has the following parameters, in this order:

| Parameter | Meaning |
|-----------|---------|
| `response` or `filePath` | where the report goes (the first parameter of `generateToResponse` or `generateToFile`) |
| `connection` | an open JDBC connection |
| `reportName` | a name for the report; it names the download and appears in log lines and error messages |
| `reportType` | the format |
| `title` | an optional title at the top of the first page; `null` for none |
| `sql` | the query that supplies the data |
| `columns` | the names of the result columns, as the query returns them |
| `columnTitles` | the text to show in the header, one per column |
| `logoPath` | an optional image next to the title, as a classpath path that begins with `/`; `null` for none |
| `tableTextAlignment` | a `HorizontalTextAlignment` such as `LEFT`, `CENTER` or `RIGHT`, for the text of the table |
| `parameters` | the values of the parameters named in the query, or `null` |

- In the query, a parameter is written `$P{name}` and receives the value of the entry with that name in `parameters`. The values are never pasted into the text of the query. The type of each parameter is taken from its value, so a value of `null` is refused.
- `columns` and `columnTitles` must be lists of the same size, and not empty.
- The logo is shown only together with a title. With no title, `logoPath` is ignored.
- Every column is treated as text. Format dates and numbers in the query when a special format is wanted.
- The style is fixed: a dark header row with white bold text, light borders, a pale background on every second row, a page number at the bottom, and the font *Vazirmatn* at 11 points. Vazirmatn supports Persian and Arabic as well as Latin letters. The font must be available to JasperReports on the server, for example installed on the machine.
- The module does not close the connection. The caller opens and closes it, and `try`-with-resources as in the snippet does that.

The checks on the arguments happen before the query is run:

```java
import net.sf.dynamicreports.report.constant.HorizontalTextAlignment;
import org.j2os.platform.jreport.dynamic.DynamicReport;
import org.j2os.platform.jreport.report.ReportType;

import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.concurrent.Callable;

public class DynamicChecks {

    public static void main(String[] args) throws Exception {
        String sql = "select name, city from customer where city = $P{city}";
        List<String> columns = List.of("name", "city");
        List<String> titles = List.of("Name", "City");
        Map<String, Object> noValue = new HashMap<>();
        noValue.put("city", null);

        String[] names = {"no columns", "empty columns", "titles do not match", "parameter without a value", "logo not found"};
        Callable<?>[] attempts = {
                () -> run(sql, null, titles, null, Map.of("city", "Oslo")),
                () -> run(sql, List.of(), List.of(), null, Map.of("city", "Oslo")),
                () -> run(sql, columns, List.of("Name"), null, Map.of("city", "Oslo")),
                () -> run(sql, columns, titles, null, noValue),
                () -> run(sql, columns, titles, "/images/nothing.png", Map.of("city", "Oslo"))
        };

        for (int i = 0; i < attempts.length; i++) {
            try {
                attempts[i].call();
            } catch (IllegalArgumentException e) {
                System.out.println(names[i] + " -> " + e.getMessage());
            }
        }
    }

    private static Object run(String sql, List<String> columns, List<String> titles, String logo, Map<String, Object> parameters) throws Exception {
        DynamicReport.generateToFile("reports-out/dynamic.pdf", null, "customers", ReportType.PDF, "Customers",
                sql, columns, titles, logo, HorizontalTextAlignment.LEFT, parameters);
        return null;
    }
}
```

```text
no columns -> columns and columnTitles must not be null
empty columns -> columns must not be empty
titles do not match -> columns and columnTitles must have the same size
parameter without a value -> Parameter 'city' has a null value; its parameter type cannot be inferred
logo not found -> No logo image found on the classpath at /images/nothing.png
```

Each of these is an `IllegalArgumentException` with a message that says what to correct. In a web application, they are a signal for status 400 or 500, depending on where the wrong value came from.

## 6. Your own design: TemplateReport

When the report must look in a special way, you design it as a `.jrxml` template and let `TemplateReport` fill it. The data comes from a query that lives inside the template, so the module needs only the connection.

A template for a customer list, which fills its rows from the data and takes a `title` parameter:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jasperReport xmlns="http://jasperreports.sourceforge.net/jasperreports"
              xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xsi:schemaLocation="http://jasperreports.sourceforge.net/jasperreports http://jasperreports.sourceforge.net/xsd/jasperreport.xsd"
              name="customer-list" pageWidth="595" pageHeight="842" columnWidth="555"
              leftMargin="20" rightMargin="20" topMargin="20" bottomMargin="20">

    <parameter name="title" class="java.lang.String"/>

    <field name="name" class="java.lang.String"/>
    <field name="city" class="java.lang.String"/>

    <title>
        <band height="40">
            <textField>
                <reportElement x="0" y="0" width="555" height="30"/>
                <textElement textAlignment="Center"/>
                <textFieldExpression><![CDATA[$P{title}]]></textFieldExpression>
            </textField>
        </band>
    </title>
    <detail>
        <band height="24">
            <textField>
                <reportElement x="0" y="0" width="300" height="24"/>
                <textFieldExpression><![CDATA[$F{name}]]></textFieldExpression>
            </textField>
            <textField>
                <reportElement x="300" y="0" width="255" height="24"/>
                <textFieldExpression><![CDATA[$F{city}]]></textFieldExpression>
            </textField>
        </band>
    </detail>
</jasperReport>
```

For `TemplateReport` the template also contains the query, and a field for each column of its result:

```xml
<queryString><![CDATA[select name, city from customer where city = $P{city} order by name]]></queryString>

<field name="name" class="java.lang.String"/>
<field name="city" class="java.lang.String"/>
```

The call:

```java
@GetMapping("/reports/sales")
public void sales(@RequestParam int year, HttpServletResponse response) throws Exception {
    try (Connection connection = dataSource.getConnection()) {
        TemplateReport.generateToResponse(
                response,
                connection,
                "/reports/sales.jrxml",
                ReportType.PDF,
                Map.of("year", year, "title", "Sales " + year));
    }
}
```

- `generateToResponse(response, connection, jrxmlPath, reportType, parameters)` and `generateToFile(filePath, connection, jrxmlPath, reportType, parameters)` have the same shape as in the other engines.
- `jrxmlPath` is a classpath path that begins with `/`, such as `/reports/sales.jrxml`. The template lives in the resources of the application.
- `parameters` is a `Map`. Each entry is available in the template as `$P{key}`, so a template can take a title, a year or a city. A `null` map means no parameters. The module works on a copy of the map, so your map is not changed.
- A template is **compiled once**. The compiled form is kept by its path, and the following calls use it at once. A change to the `.jrxml` file therefore takes effect after the application restarts.
- The module does not close the connection.
- A problem in the template or in its query gives a `RuntimeException`. The message names the template, and the original problem is its *cause*: `Failed to compile template <path>` or `Failed to fill template report <path>`.

## 7. A template filled from a list: EntityTemplateReport

`EntityTemplateReport` uses the same kind of template, but the rows come from a list in your program. Nothing here needs a database. Use it when the data comes from a service, a file or a calculation, or when you want to report on exactly the rows that the user sees on screen.

```java
@GetMapping("/reports/customers/list")
public void list(HttpServletResponse response) throws Exception {
    List<Map<String, Object>> rows = customerService.findAll().stream()
            .map(c -> Map.<String, Object>of("name", c.getName(), "city", c.getCity()))
            .toList();

    EntityTemplateReport.generateToResponse(
            response,
            "/reports/customer-list.jrxml",
            ReportType.XLSX,
            Map.of("title", "Customers"),
            rows);
}
```

The rows are a `List<Map<String, Object>>`, one map per row. The *key* of each entry is the name of a field of the template: with the template of chapter 6, the keys are `name` and `city`. The programs in chapters 3, 4 and 8 use this engine.

- The template has no `queryString`, because the data does not come from a query.
- Its compiled form is cached separately from the cache of `TemplateReport`, so the same path may name different templates in the two engines.
- A problem when the report is filled gives `Failed to fill bean report <path>`, with the original problem as the cause.

## 8. Putting it together: customer reports every month

A shop wants two things from its customer data. Every month, a copy of the list in all three formats must be kept in a folder for the accountants. During the day, a manager must be able to download the same list from a page, in the format that she likes.

The first part is a program that exports the list in every format:

```java
import org.j2os.platform.jreport.jasper.EntityTemplateReport;
import org.j2os.platform.jreport.report.ReportType;

import java.nio.file.Files;
import java.nio.file.Path;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

public class ExportAll {

    public static void main(String[] args) throws Exception {
        List<Map<String, Object>> customers = new ArrayList<>();
        for (int i = 1; i <= 40; i++) {
            customers.add(Map.of("name", "Customer " + i, "city", i % 2 == 0 ? "Oslo" : "Kyoto"));
        }

        Path folder = Path.of("monthly");
        for (ReportType type : ReportType.values()) {
            Path file = folder.resolve("customers." + type.getExtension());
            EntityTemplateReport.generateToFile(file.toString(), "/reports/customer-list.jrxml",
                    type, Map.of("title", "Customers, October"), customers);
            System.out.println(file + " -> " + (Files.size(file) > 0 ? "written" : "empty"));
        }

        try (var files = Files.list(folder)) {
            System.out.println(files.count() + " files, no temporary files left");
        }
    }
}
```

```text
monthly/customers.pdf -> written
monthly/customers.docx -> written
monthly/customers.xlsx -> written
3 files, no temporary files left
```

The program uses the whole module in a few lines:

1. `ReportType.values()` gives the three formats. The program writes one file per format and builds the file name from `getExtension()`.
2. `EntityTemplateReport.generateToFile` fills the template of chapter 6 with the 40 customers and writes the file. The folder is created by the module.
3. The title reaches the template as the parameter `title`.
4. The folder holds exactly three files. The temporary files of the export are gone.

In the same way, the module is used by a job that runs once a month. With a query inside the template, the job takes a connection and no list:

```java
@Scheduled(cron = "0 0 2 1 * *")          // 02:00 on the first day of each month
public void monthlyReport() throws Exception {
    String month = YearMonth.now().minusMonths(1).toString();
    try (Connection connection = dataSource.getConnection()) {
        TemplateReport.generateToFile(
                "/var/reports/sales-" + month + ".pdf",
                connection,
                "/reports/sales.jrxml",
                ReportType.PDF,
                Map.of("month", month));
    }
}
```

For the manager, a controller sends the same list as a download, and the format comes from the request:

```java
@GetMapping("/reports/customers/download")
public void download(@RequestParam(defaultValue = "pdf") String format, HttpServletResponse response) throws Exception {
    List<Map<String, Object>> rows = customerService.findAll().stream()
            .map(c -> Map.<String, Object>of("name", c.getName(), "city", c.getCity()))
            .toList();

    EntityTemplateReport.generateToResponse(
            response,
            "/reports/customer-list.jrxml",
            ReportType.fromCode(format),
            Map.of("title", "Customers"),
            rows);
}
```

A wrong `format` ends in the `IllegalArgumentException` that chapter 4 turns into status 400.

When the data is a plain table and no special design is needed, `DynamicReport` replaces the template with a query, as in chapter 5.

---

## Appendix A. Quick reference

### A.1 ReportType

| Member | Description |
|--------|-------------|
| `PDF`, `DOCX`, `XLSX` | the three formats |
| `fromCode(code)` | `pdf`, `doc`, `docx`, `xls` or `xlsx`, in any letter case and with spaces around; otherwise `IllegalArgumentException` |
| `deriveDisplayName(path)` | the file name of a path without folder and extension |
| `getMimeType()` | the MIME type of the format |
| `getExtension()` | the extension without a dot |
| `newExporter()` | a new JasperReports exporter for the format |
| `writeAsAttachment(response, displayName, jasperPrint)` | writes a filled report to a response as a download |
| `writeToFile(filePath, displayName, jasperPrint)` | writes a filled report to a file, safely |

### A.2 DynamicReport

| Member | Description |
|--------|-------------|
| `generateToResponse(response, connection, reportName, reportType, title, sql, columns, columnTitles, logoPath, tableTextAlignment, parameters)` | builds the table and sends it as a download |
| `generateToFile(filePath, connection, reportName, reportType, title, sql, columns, columnTitles, logoPath, tableTextAlignment, parameters)` | builds the table and saves it |

### A.3 TemplateReport

| Member | Description |
|--------|-------------|
| `generateToResponse(response, connection, jrxmlClasspathPath, reportType, parameters)` | fills the template with the result of its query and sends it as a download |
| `generateToFile(filePath, connection, jrxmlClasspathPath, reportType, parameters)` | the same, saved to a file |

### A.4 EntityTemplateReport

| Member | Description |
|--------|-------------|
| `generateToResponse(response, jrxmlClasspathPath, reportType, parameters, entities)` | fills the template with the list and sends it as a download |
| `generateToFile(filePath, jrxmlClasspathPath, reportType, parameters, entities)` | the same, saved to a file |

All methods are `static` and declare `IOException`.

## Appendix B. Failure handling

| Situation | Result |
|-----------|--------|
| a format code that is not known, or `null` | `IllegalArgumentException` |
| a `null` path for `deriveDisplayName` | `IllegalArgumentException` |
| a template that is not on the classpath | `IllegalArgumentException` |
| a template that cannot be loaded or compiled | `RuntimeException`, the cause is attached |
| a template that cannot be filled (a bad query, a missing parameter) | `RuntimeException`, the cause is attached |
| `DynamicReport` with `null` or empty columns, or with different sizes of columns and titles | `IllegalArgumentException` |
| `DynamicReport` with a parameter whose value is `null` | `IllegalArgumentException` |
| `DynamicReport` with a logo that is not found | `IllegalArgumentException` |
| a report that cannot be built by `DynamicReport` | `RuntimeException`, the cause is attached |
| an export that fails | `RuntimeException`, the cause is attached |
| a file that cannot be written | `IOException` |
| a failed export to a file | the temporary file is deleted, the old file stays |
| a failed write to the response | `IOException` |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Begin the template path with `/`.** Without it the template is looked for in the wrong place.
- **A template is compiled once.** Changing the `.jrxml` takes effect only after a restart.
- **The connection is yours.** The module neither opens nor closes it. Close it after the call, for example with `try`-with-resources.
- **Do not paste user input into the SQL text.** Pass it as a parameter with `$P{name}` and the `parameters` map.
- **A parameter of `DynamicReport` cannot be `null`.** Its type is taken from the value.
- **Everything in `DynamicReport` is text.** Format dates and numbers in the query.
- **The font must exist.** `DynamicReport` uses Vazirmatn. Install it on the server.
- **Call `generateToResponse` on a fresh response.** Nothing else may have been written to it.
- **The response is sent after the report is complete.** The whole report is in memory at that time, so a very large report needs memory.
- **The format comes from `fromCode`.** Check what the user sends and answer a wrong format with status 400.
- **The keys of the maps are the field names.** A key that the template does not use is ignored, and the template must declare every field that it shows.

### C.2 Behaviour worth knowing

- Every export uses a new exporter, so concurrent exports do not influence each other.
- `generateToFile` writes to a temporary file in the folder of the target. The target and the temporary file are therefore on the same disk.
- A template that is cached under one path stays cached for the life of the application.
- `TemplateReport` and `EntityTemplateReport` have separate caches.
- The parameters map is copied before it is used.
- The module logs at the `info` level how long each report took, and `DynamicReport` also logs the number of pages.
- The name of a download comes from the template path in `TemplateReport` and `EntityTemplateReport`, and from `reportName` in `DynamicReport`.

### C.3 Limits

- **Three formats:** PDF, DOCX and XLSX. There is no CSV and no HTML.
- **A fixed layout in `DynamicReport`:** A4, portrait, a margin of 20 points, all columns as text, no totals and no groups. Anything else needs a template.
- **Templates come from the classpath.** A template from a file on the disk or from a database is not supported.
- **One data source per report.** A report cannot combine a query and a list.

### C.4 Tuning

- Use `DynamicReport` for plain lists and a template for anything with a design. Do not build a layout in `DynamicReport` that a template does better.
- For reports that take long, use `generateToFile` in a background job and let the user download the file later, so that a request does not wait.
- Limit what a user may ask for, for example the number of rows or the range of dates, before the query runs.
- Keep the templates in one folder of the resources, such as `/reports/`, and name them after the report, because the name appears in the download.
- Add a restart step to the release process when a template changes.
- Make the size of the report visible to the user, for example with a note on the page, when a query can return many rows.

## Appendix D. Message reference

**IllegalArgumentException**

| Message | Meaning |
|---------|---------|
| `Type not supported: <code>` | the format code is not known; it is `null` when no code is given |
| `path must not be null` | `deriveDisplayName` receives `null` |
| `No .jrxml found on the classpath at <path>` | there is no template at that path |
| `columns and columnTitles must not be null` | one of the two lists is `null` |
| `columns must not be empty` | no column is given |
| `columns and columnTitles must have the same size` | the lists differ in size |
| `Parameter '<name>' has a null value; its parameter type cannot be inferred` | a parameter of `DynamicReport` has no value |
| `No logo image found on the classpath at <path>` | the logo is not on the classpath |

**RuntimeException** (the original problem is the cause)

| Message | Meaning |
|---------|---------|
| `Failed to compile template <path>` | the template cannot be loaded or compiled |
| `Failed to fill template report <path>` | `TemplateReport` cannot fill the template |
| `Failed to fill bean report <path>` | `EntityTemplateReport` cannot fill the template |
| `Failed to build report <name>` | `DynamicReport` cannot build the report |
| `Failed to export report <name> as <FORMAT>` | the export fails; `<FORMAT>` is `PDF`, `DOCX` or `XLSX` |
