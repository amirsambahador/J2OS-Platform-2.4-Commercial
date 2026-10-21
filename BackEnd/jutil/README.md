# jutil

*J2OS Platform 2.4*

Almost every application needs the same small helpers: a date shown in the Persian calendar, an amount written in words on an invoice, a JSON text read into an object, the distance between two cities. They are easy to describe and surprisingly easy to get wrong, so they are best written once and shared.

**jutil** (`org.j2os.platform.jutil`) is that shared collection. It has five main classes:

- `JDate` holds one calendar date and converts it between the Persian (Jalali) and Gregorian calendars.
- `JNumber` writes numbers in words (English or Persian) and formats them with Persian or Latin digits and thousands separators.
- `JSon` turns objects into JSON text and back, and reads single fields out of JSON text.
- `Geo` calculates distances, midpoints and search areas for geographic coordinates. `BoundingBox` and `Coordinate` are the small classes that carry its results.
- `SpringContext` gives code that is not a Spring bean access to beans, to the current request and to cookies.

```java
String due = new JDate(2025, 3, 21).getPersianDate();         // a date in the Persian calendar
String words = JNumber.getPersianWords("1250000");            // a number written in words
String json = JSon.write(order);                              // an object as JSON text
double meters = Geo.distanceMeters(35.69, 51.39, 32.65, 51.67);   // a distance on the Earth
```

Here is how the guide is arranged: each chapter has a short program followed by the output it prints.

## Contents

1. [What jutil contains](#1-what-jutil-contains)
2. [A first look at JDate](#2-a-first-look-at-jdate)
3. [Converting between the two calendars](#3-converting-between-the-two-calendars)
4. [Moving a date forward and back](#4-moving-a-date-forward-and-back)
5. [Weekdays and leap years](#5-weekdays-and-leap-years)
6. [Dates as text and as timestamps](#6-dates-as-text-and-as-timestamps)
7. [Dates that do not exist](#7-dates-that-do-not-exist)
8. [Numbers in words](#8-numbers-in-words)
9. [Formatting numbers](#9-formatting-numbers)
10. [Numbers typed by users](#10-numbers-typed-by-users)
11. [JSON: objects to text and back](#11-json-objects-to-text-and-back)
12. [JSON: reading single fields](#12-json-reading-single-fields)
13. [Distance, midpoint and valid coordinates](#13-distance-midpoint-and-valid-coordinates)
14. [Searching near a point](#14-searching-near-a-point)
15. [Reaching Spring from anywhere](#15-reaching-spring-from-anywhere)
16. [From idea to application: a purchase receipt](#16-from-idea-to-application-a-purchase-receipt)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

`JDate`, `JNumber` and `Geo` need nothing but the JDK. `JSon` needs the Jackson library (version 3) on the classpath, and `SpringContext` needs Spring Web and the Jakarta Servlet API. `JNumber`, `JSon` and `Geo` have only static methods and are never instantiated. `JDate` is the class that you create objects of.

---

## 1. What jutil contains

The five classes do not depend on each other, so after this chapter you can read the rest in any order. They do share three habits, and knowing them up front saves a lot of repetition.

**Text in, text out.** Numbers, dates and coordinates are mostly passed as text, in the form a person or a web form would give them: `"1,234,567"`, `"1403/01/01"`, `"35.6892"`. The classes are strict about the shape of that text and say exactly what is wrong when it does not fit.

**Persian and Latin side by side.** Wherever digits or calendars appear, both versions are supported. `JNumber` reads Persian digits as readily as Latin ones, and `JDate` converts in both directions.

**Plain failures.** A value that cannot be handled throws a standard exception, mostly `IllegalArgumentException`, with a message that names the problem. Nothing is silently corrected, except where a chapter says otherwise. Appendix B lists every case.

## 2. A first look at JDate

A `JDate` represents one calendar day. It knows that day in both calendars at the same time, so there is no "Persian mode" or "Gregorian mode" to switch between.

```java
import org.j2os.platform.jutil.date.JDate;

public class DateBasics {

    public static void main(String[] args) {
        JDate nowruz = new JDate(2025, 3, 21);  // Gregorian year, month, day
        System.out.println(nowruz);
        System.out.println("Persian:   " + nowruz.getPersianDate());
        System.out.println("Gregorian: " + nowruz.getGregorianDate());

        JDate today = new JDate();  // no arguments: today
        System.out.println("Persian year is 1404 or later: " + (today.getPersianYear() >= 1404));
    }
}
```

```text
Friday, Gregorian:[2025/03/21], Persian:[1404/01/01]
Persian:   1404/01/01
Gregorian: 2025/03/21
Persian year is 1404 or later: true
```

- `new JDate(2025, 3, 21)` takes a **Gregorian** year, month and day. March 21, 2025 is the first day of the Persian year 1404 (Nowruz).
- `toString()` prints the weekday and both versions of the date, which is handy for logging.
- `getPersianDate()` and `getGregorianDate()` return the date as `yyyy/MM/dd`, with the month and the day padded to two digits.
- `new JDate()` with no arguments is today, read from the system clock and the default time zone of the Java process.

A `JDate` has no time of day. It is a date and nothing more.

## 3. Converting between the two calendars

The constructor takes a Gregorian date. To start from a Persian date, create any `JDate` and set it:

```java
import org.j2os.platform.jutil.date.JDate;

public class PersianInput {

    public static void main(String[] args) {
        JDate date = new JDate();

        date.setPersianDate(1403, 12, 30);  // the last day of a leap year
        System.out.println("Gregorian: " + date.getGregorianDate());
        System.out.println("Parts: " + date.getGregorianYear() + ", "
                + date.getGregorianMonth() + ", " + date.getGregorianDay());

        date.setGregorianDate(2024, 3, 20);  // the same object, set from the other side
        System.out.println("Persian:  " + date.getPersianDate());
        System.out.println("Parts: " + date.getPersianYear() + ", "
                + date.getPersianMonth() + ", " + date.getPersianDay());
    }
}
```

```text
Gregorian: 2025/03/20
Parts: 2025, 3, 20
Persian:  1403/01/01
Parts: 1403, 1, 1
```

- `setPersianDate(year, month, day)` and `setGregorianDate(year, month, day)` replace the date held by the object. Both calendars are updated, so you can read the result in either.
- 1403 is a leap year, so Esfand (month 12) has 30 days and 1403/12/30 exists. It is the day before Nowruz 1404, which fell on March 21, 2025.
- Six getters return the parts as numbers: `getPersianYear()`, `getPersianMonth()`, `getPersianDay()` and the three Gregorian ones. Use them when you need a part and not a text.

A `JDate` is **mutable**: the setters change the object. If one part of a program passes a `JDate` to another, both see the same object. Chapter 4 shows how to keep an original.

## 4. Moving a date forward and back

A few methods move the date by whole days, in either direction:

```java
import org.j2os.platform.jutil.date.JDate;

public class MovingDates {

    public static void main(String[] args) {
        JDate date = new JDate(2025, 3, 21);
        System.out.println("start:          " + date.getPersianDate());

        date.nextDay();
        System.out.println("nextDay():      " + date.getPersianDate());

        date.nextDay(30);
        System.out.println("nextDay(30):    " + date.getPersianDate());

        date.previousDay();
        System.out.println("previousDay():  " + date.getPersianDate());

        date.addDays(-365);
        System.out.println("addDays(-365):  " + date.getPersianDate());
    }
}
```

```text
start:          1404/01/01
nextDay():      1404/01/02
nextDay(30):    1404/02/01
previousDay():  1404/01/31
addDays(-365):  1403/02/01
```

- `nextDay()` and `previousDay()` move by one day, `nextDay(n)` and `previousDay(n)` by `n` days.
- `addDays(n)` is the general form. A negative number goes back, so `addDays(-365)` and `previousDay(365)` are the same.
- The month lengths and the leap days of both calendars are handled for you: 1404/01/31 is one step back from 1404/02/01.

Because the object changes in place, make a copy before you move a date that you still need. A copy is a new `JDate` built from the Gregorian parts of the original:

```java
JDate original = new JDate(2025, 3, 21);
JDate due = new JDate(original.getGregorianYear(), original.getGregorianMonth(), original.getGregorianDay());
due.nextDay(30);        // original is unchanged
```

## 5. Weekdays and leap years

```java
import org.j2os.platform.jutil.date.JDate;

public class WeekAndLeap {

    public static void main(String[] args) {
        JDate date = new JDate(2025, 3, 21);
        System.out.println(date.getWeekDayName());
        System.out.println(date.getPersianWeekDayName());
        System.out.println("getDayOfWeek(): " + date.getDayOfWeek());

        System.out.println("1404 is a leap year: " + date.isLeap());  // the year of this date
        for (int year : new int[]{1399, 1400, 1403, 1404, 1408}) {
            System.out.println(year + " -> " + date.isLeap(year));
        }
    }
}
```

```text
Friday
جمعه
getDayOfWeek(): 4
1404 is a leap year: false
1399 -> true
1400 -> false
1403 -> true
1404 -> false
1408 -> true
```

- `getWeekDayName()` gives the English name and `getPersianWeekDayName()` the Persian one. In the Persian names the number and the word are written apart, for example «سه شنبه».
- `getDayOfWeek()` is the weekday as a number from **0 (Monday) to 6 (Sunday)**. This is not the numbering of `java.util.Calendar`, where Sunday is 1.
- `isLeap()` tells whether the Persian year of the date is a leap year. `isLeap(year)` asks the same about any Persian year. Both are ordinary methods, so call them on an object; the object's own date does not matter for `isLeap(year)`.
- The Persian leap years follow the real calendar rule, not a simple "every four years". The years 1399, 1403 and 1408 are leap years; the years between them are not.

## 6. Dates as text and as timestamps

Databases and web forms rarely hand over a `JDate`. They hand over a `java.sql.Timestamp` or a piece of text like `"1403/01/01"`. Two methods convert between those and the Persian calendar:

```java
import org.j2os.platform.jutil.date.JDate;

import java.sql.Timestamp;

public class Timestamps {

    public static void main(String[] args) {
        JDate converter = new JDate();

        Timestamp start = converter.getGregorianDateTimestamp("1403/01/01");
        System.out.println(start);

        System.out.println(converter.getPersianDateString(Timestamp.valueOf("2024-03-20 23:59:59")));
        System.out.println(converter.getGregorianDateTimestamp(" 1403 / 1 / 1 "));
    }
}
```

```text
2024-03-20 00:00:00.0
1403/01/01
2024-03-20 00:00:00.0
```

- `getGregorianDateTimestamp(persianText)` reads a Persian date given as `yyyy/MM/dd` and returns the matching Gregorian day as a `Timestamp` at midnight. The parts are separated by `/`. Spaces around a part are ignored, and the month and day need not have two digits.
- `getPersianDateString(timestamp)` does the reverse and returns the Persian date as `yyyy/MM/dd`. Only the date of the timestamp is used. The time `23:59:59` stays on the same day.
- Both methods work on a temporary date inside, so the object you call them on is not changed. Any `JDate` will do as the "converter".

The `Timestamp` is created in the default time zone of the Java process. A `Timestamp` at midnight is a different instant in a different time zone.

To get a `JDate` from Persian text, convert it to a timestamp first, as chapter 16 does.

## 7. Dates that do not exist

Both calendars have months of different lengths, and a day that is out of range for its month is an error, not a date in the next month.

```java
import org.j2os.platform.jutil.date.JDate;

public class DateErrors {

    public static void main(String[] args) {
        JDate converter = new JDate();
        String[] texts = {"1403-01-01", "1403/13/01", "1403/01/32", "1404/12/30", "0/01/01"};
        for (String text : texts) {
            try {
                converter.getGregorianDateTimestamp(text);
            } catch (IllegalArgumentException e) {
                System.out.println(text + " -> " + e.getMessage());
            }
        }

        try {
            new JDate(2023, 2, 29);
        } catch (IllegalArgumentException e) {
            System.out.println("2023/2/29 -> " + e.getMessage());
        }
    }
}
```

```text
1403-01-01 -> Expected format yyyy/MM/dd but got: "1403-01-01"
1403/13/01 -> Month must be between 1 and 12, got: 13
1403/01/32 -> Day must be between 1 and 31, got: 32
1404/12/30 -> Day must be between 1 and 29 for Persian 1404/12, got: 30
0/01/01 -> Year must be positive, got: 0
2023/2/29 -> Day must be between 1 and 28 for Gregorian 2023/2, got: 29
```

- Text that is not three numbers separated by `/` fails with the message `Expected format yyyy/MM/dd but got: ...`.
- A month outside 1 to 12, a day outside 1 to 31 and a year that is not positive are rejected first with a short message. `null` is rejected as well.
- Then the real calendar is applied. In the Persian calendar months 1 to 6 have 31 days, months 7 to 11 have 30 days, and Esfand has 29 days, or 30 in a leap year. This is why 1404/12/30 fails and 1403/12/30 does not. In the Gregorian calendar, February has 28 or 29 days.
- The constructor and the two setters check in the same way. A `JDate` can never hold an impossible date.

All of these are `IllegalArgumentException`s, so a single `catch` handles them. The exact messages are listed in Appendix D.

## 8. Numbers in words

Invoices and checks need the amount twice: in digits and in words. `getEnglishWords` and `getPersianWords` take the number as text and return the words.

```java
import org.j2os.platform.jutil.number.JNumber;

public class Words {

    public static void main(String[] args) {
        String[] numbers = {"0", "21", "110", "1001", "1,234,567", "-45.6", "0.05"};

        for (String number : numbers) {
            System.out.println(number + " -> " + JNumber.getEnglishWords(number));
        }
        System.out.println();
        for (String number : numbers) {
            System.out.println(number + " -> " + JNumber.getPersianWords(number));
        }
    }
}
```

```text
0 -> Zero
21 -> Twenty One
110 -> One Hundred Ten
1001 -> One Thousand One
1,234,567 -> One Million Two Hundred Thirty Four Thousand Five Hundred Sixty Seven
-45.6 -> Negative Forty Five Point Six
0.05 -> Zero Point Zero Five

0 -> صفر
21 -> بیست و یک
110 -> صد و ده
1001 -> یک هزار و یک
1,234,567 -> یک میلیون و دویست و سی و چهار هزار و پانصد و شصت و هفت
-45.6 -> منفی چهل و پنج ممیز شش
0.05 -> صفر ممیز صفر پنج
```

- The number is passed as a `String`, so `int` and `long` impose no limit. The only limit is 36 integer digits (section 8.1).
- Thousands separators are ignored (`1,234,567`). A sign is allowed only as a first `-`, which becomes «Negative» or «منفی».
- A decimal part is read digit by digit after «Point» or «ممیز». The number `0.05` is «Zero Point Zero Five», not «five hundredths».
- A Persian thousand is written «یک هزار» and a million «یک میلیون», with the «یک» (one), as in the output. In English the words are joined with spaces, and the Persian ones with «و» (and).

### 8.1 Unusual inputs

```java
import org.j2os.platform.jutil.number.JNumber;

public class WordsEdges {

    public static void main(String[] args) {
        System.out.println(JNumber.getEnglishWords("۱۲۳"));  // Persian digits are accepted
        System.out.println(JNumber.getEnglishWords("-0.00"));  // a negative zero is just zero
        System.out.println("[" + JNumber.getEnglishWords(null) + "]");  // nothing in, nothing out

        String[] bad = {"12a", "1.2.3", "+5", "1 000"};
        for (String text : bad) {
            try {
                JNumber.getEnglishWords(text);
            } catch (IllegalArgumentException e) {
                System.out.println(text + " -> " + e.getMessage());
            }
        }

        try {
            JNumber.getEnglishWords("1" + "0".repeat(36));  // 37 digits
        } catch (ArithmeticException e) {
            System.out.println("too large -> " + e.getMessage());
        }
    }
}
```

```text
One Hundred Twenty Three
Zero Point Zero Zero
[]
12a -> Invalid numeric string: contains non-digit characters.
1.2.3 -> Invalid numeric string: multiple decimal points.
+5 -> Invalid numeric string: contains non-digit characters.
1 000 -> Invalid numeric string: contains non-digit characters.
too large -> Number is too large to convert to words (exceeds supported magnitude).
```

- Persian digits and the Persian decimal separator `٫` are accepted and understood as the Latin ones.
- `-0.00` is zero, so no «Negative» is added. `null` and an empty text give an empty text and no exception, which makes the methods safe to call on an optional value.
- Anything that is not a digit fails. That includes `+5`, spaces inside the number and letters. A second decimal point fails as well.
- The integer part can have up to 36 digits (twelve groups of three). A 37th digit throws an `ArithmeticException`.

## 9. Formatting numbers

For display, a long number needs separators. `getPersianNumber` writes it with Persian digits and `getEnglishNumber` with Latin digits. Both group the integer part in threes with a comma and put the Persian decimal separator `٫` before the fraction.

```java
import org.j2os.platform.jutil.number.JNumber;

public class Formatting {

    public static void main(String[] args) {
        String price = "1234567.5";

        String persian = JNumber.getPersianNumber(price);
        String latin = JNumber.getEnglishNumber(price);
        System.out.println(persian);
        System.out.println(latin);
        System.out.println(JNumber.getEnglishNumberWithoutCommas(latin));  // back to a plain number

        System.out.println(JNumber.getPersianNumber("-1234"));
        System.out.println(JNumber.getEnglishNumber("۱۲۳۴۵۶۷"));
        System.out.println(JNumber.getEnglishNumber(" 12 345 "));
    }
}
```

```text
۱,۲۳۴,۵۶۷٫۵
1,234,567٫5
1234567.5
-۱,۲۳۴
1,234,567
12,345
```

- The first two lines show the same text in the two scripts. In both, the thousands separator is the ordinary comma and the decimal separator is `٫` (U+066B), which is different from `.`.
- `getEnglishNumberWithoutCommas` goes the other way: it removes the commas, turns Persian digits into Latin ones and turns `٫` into `.`. The result, `1234567.5`, can be given to `Long.parseLong`, `Double.parseDouble` or `new BigDecimal(...)`.
- Persian digits in the input are fine, and a leading `-` is kept. Spaces are dropped, which is why `" 12 345 "` becomes `12,345`.
- The methods format, they do not round. `12.50` keeps both decimals, and leading zeros are removed.

## 10. Numbers typed by users

People type numbers in every form: with Persian digits, with the Arabic thousands mark `٬`, with a currency word at the end. The formatting methods remove everything that is not a digit or a decimal separator, so they also work as a cleaner:

```java
import org.j2os.platform.jutil.number.JNumber;

public class MoneyInput {

    public static void main(String[] args) {
        String typed = "۱٬۲۵۰٬۰۰۰ ریال";  // what a user might type in a form

        String clean = JNumber.getEnglishNumberWithoutCommas(JNumber.getEnglishNumber(typed));
        long amount = Long.parseLong(clean);

        System.out.println("clean:  " + clean);
        System.out.println("amount: " + (amount + 250_000));
        System.out.println("shown:  " + JNumber.getPersianNumber(String.valueOf(amount + 250_000)));
    }
}
```

```text
clean:  1250000
amount: 1500000
shown:  ۱,۵۰۰,۰۰۰
```

- `getEnglishNumber` removes the thousands mark `٬`, the unit «ریال» and the spaces, and converts the digits. `getEnglishNumberWithoutCommas` then leaves a plain number.
- After this step, `Long.parseLong` works and ordinary arithmetic is possible. At the end `getPersianNumber` formats the new amount for the screen.

This tolerance has a price. The cleaner does not judge what it removes: `1e5` becomes `15`, and a second `.` is silently dropped. When the field must contain a number and nothing else, check the text first, for example with the number rules of the `jvalidation` module, and clean it afterwards.

## 11. JSON: objects to text and back

JSON is the format in which almost everything travels between a browser, a server and other services. `JSon.write` turns an object into JSON text and `JSon.read` turns text back into an object of the class you name.

```java
import org.j2os.platform.jutil.json.JSon;

public class JsonObjects {

    public record Address(String city, String street) {
    }

    public record Person(Address address, int age, String name) {
    }

    public static void main(String[] args) {
        Person sara = new Person(new Address("Shiraz", "Zand Street"), 29, "Sara");

        String json = JSon.write(sara);
        System.out.println(json);

        Person copy = JSon.read(json, Person.class);
        System.out.println(copy.name() + " lives in " + copy.address().city());
    }
}
```

```text
{"address":{"city":"Shiraz","street":"Zand Street"},"age":29,"name":"Sara"}
Sara lives in Shiraz
```

- `write(object)` accepts any object that Jackson can read: records, classes with getters, maps and lists. The nested `Address` becomes a nested JSON object.
- `read(text, Type.class)` builds a new object. The class needs a form that Jackson can fill in: a record, or a class with a default constructor and setters or public fields.
- All the methods share one configured Jackson mapper, created once and safe to use from many threads. It uses Jackson's default settings.

## 12. JSON: reading single fields

Often you need one value from a JSON text and a full class would be too much. Two methods read a field directly, following a path of names:

```java
import org.j2os.platform.jutil.json.JSon;

public class JsonFields {

    public static void main(String[] args) {
        String json = "{\"name\":\"Sara\",\"age\":29,\"address\":{\"city\":\"Shiraz\"},\"tags\":[\"a\",\"b\"]}";

        System.out.println(JSon.readFieldAsText(json, "name"));
        System.out.println(JSon.readFieldAsText(json, "address", "city"));  // a path of names
        System.out.println(JSon.readFieldAsText(json, "age"));
        System.out.println("[" + JSon.readFieldAsText(json, "email") + "]");  // missing: empty text

        System.out.println(JSon.readFieldAsString(json, "name"));  // keeps the quotes
        System.out.println(JSon.readFieldAsString(json, "address"));  // a whole object
        System.out.println(JSon.readFieldAsString(json, "tags"));  // a whole array
    }
}
```

```text
Sara
Shiraz
29
[]
"Sara"
{"city":"Shiraz"}
["a","b"]
```

- `readFieldAsText(json, "address", "city")` follows the path `address`, then `city`, and returns the value as plain text. A string comes back without quotes, and numbers and booleans come back as text.
- A field that does not exist does not cause an exception. `readFieldAsText` returns an empty text, so an absent field and an empty value look the same. If the difference matters, check the field with `readFieldAsString`: an existing empty string comes back as `""`, with its quotes.
- `readFieldAsString` returns the field as JSON text: a string with its quotes, and an object or an array in full.

Two more methods start from a JSON array and take the position of the element:

```java
import org.j2os.platform.jutil.json.JSon;

public class JsonArrays {

    public static void main(String[] args) {
        String json = "[{\"id\":1,\"title\":\"Tea\"},{\"id\":2,\"title\":\"Coffee\"}]";

        System.out.println(JSon.readFieldArrayAsText(json, 1, "title"));
        System.out.println(JSon.readFieldArrayAsString(json, 0, "title"));
        System.out.println(JSon.readFieldArrayAsString(json, 0, "id"));
        System.out.println("[" + JSon.readFieldArrayAsText(json, 5, "title") + "]");  // no such element
    }
}
```

```text
Coffee
"Tea"
1
[]
```

- `readFieldArrayAsText(json, index, field)` and `readFieldArrayAsString(json, index, field)` read one field of the element at `index`. The first element has index 0.
- An index beyond the end of the array does not fail. The text version returns an empty text.

Invalid JSON is the one case that does throw. Appendix B shows it.

## 13. Distance, midpoint and valid coordinates

`Geo` works on latitude and longitude in decimal degrees: latitude from -90 (the South Pole) to 90 (the North Pole), and longitude from -180 to 180. The Earth is treated as a sphere of radius 6,371,000 meters.

```java
import org.j2os.platform.jutil.geo.Coordinate;
import org.j2os.platform.jutil.geo.Geo;

public class GeoBasics {

    public static void main(String[] args) {
        double tehranLat = 35.6892, tehranLon = 51.3890;
        double isfahanLat = 32.6546, isfahanLon = 51.6680;

        double meters = Geo.distanceMeters(tehranLat, tehranLon, isfahanLat, isfahanLon);
        System.out.printf("Tehran to Isfahan: %.0f m (%.1f km)%n", meters, meters / 1000);

        Coordinate middle = Geo.midpointBetween(tehranLat, tehranLon, isfahanLat, isfahanLon);
        System.out.printf("Midpoint: %.4f, %.4f%n", middle.latitude, middle.longitude);

        System.out.println(Geo.isValidCoordinate("35.7", "51.4"));  // true
        System.out.println(Geo.isValidCoordinate("91", "51.4"));  // latitude out of range
        System.out.println(Geo.isValidCoordinate("abc", "51.4"));  // not a number
        System.out.println(Geo.isValidCoordinate(null, "51.4"));  // missing
    }
}
```

```text
Tehran to Isfahan: 338406 m (338.4 km)
Midpoint: 34.1720, 51.5310
true
false
false
false
```

- `distanceMeters(lat1, lon1, lat2, lon2)` is the length of the shortest path over the surface of the Earth, in meters. It uses the *haversine* formula, the standard way to measure the distance between two points on a sphere.
- `midpointBetween(...)` returns a `Coordinate` with `latitude` and `longitude` fields: the point halfway along that shortest path. It is not the average of the two numbers.
- `isValidCoordinate(latitudeText, longitudeText)` answers `true` only if both texts are numbers in range. It never throws. `null`, text that is not a number, `NaN` and infinite values all give `false`. Use it to check what a user or a device sent, before the other methods.

## 14. Searching near a point

"Which branches are within 5 km of this customer?" is easy to ask and expensive to answer: measuring the distance to every row of a table is slow. The usual way is two steps. First a cheap test keeps only the rows inside a rectangle around the point, and the database can answer it with an index. Then the exact distance is calculated for those few rows.

`boundingBoxAround(latitudeText, longitudeText, radiusMeters)` builds that rectangle:

```java
import org.j2os.platform.jutil.geo.BoundingBox;
import org.j2os.platform.jutil.geo.Geo;

public class GeoBox {

    public static void main(String[] args) {
        String[][] searches = {
                {"Around the centre of Tehran", "35.6892", "51.3890"},
                {"Near the 180 degree meridian", "0", "179.99"},
                {"Near the North Pole", "89.99", "10"}};

        for (String[] search : searches) {
            System.out.println(search[0] + ", 5 km:");
            for (BoundingBox box : Geo.boundingBoxAround(search[1], search[2], 5000)) {
                System.out.printf("  latitude %.4f to %.4f, longitude %.4f to %.4f%n",
                        box.minLatitude, box.maxLatitude, box.minLongitude, box.maxLongitude);
            }
        }
    }
}
```

```text
Around the centre of Tehran, 5 km:
  latitude 35.6442 to 35.7342, longitude 51.3336 to 51.4444
Near the 180 degree meridian, 5 km:
  latitude -0.0450 to 0.0450, longitude 179.9450 to 180.0000
  latitude -0.0450 to 0.0450, longitude -180.0000 to -179.9650
Near the North Pole, 5 km:
  latitude 89.9450 to 90.0000, longitude -180.0000 to 180.0000
```

- Each `BoundingBox` has four fields: `minLatitude`, `maxLatitude`, `minLongitude` and `maxLongitude`.
- In the normal case you get one box.
- Near the 180° meridian, longitude jumps from 180 to -180, and a single rectangle cannot cover both sides. The method then returns **two** boxes, one on each side, and you combine them with `OR`.
- Near a pole, every longitude is inside the circle, so the box spans all longitudes.

A query that works for every case combines all the boxes that are returned:

```sql
SELECT id, name, latitude, longitude
FROM branch
WHERE (latitude BETWEEN :minLat1 AND :maxLat1 AND longitude BETWEEN :minLon1 AND :maxLon1)
   OR (latitude BETWEEN :minLat2 AND :maxLat2 AND longitude BETWEEN :minLon2 AND :maxLon2)
```

The rectangle is larger than the circle, because it includes the corners. The rows it returns must still be filtered with `Geo.distanceMeters`, which is what chapter 16 does.

## 15. Reaching Spring from anywhere

In a Spring application the framework creates objects and passes them to each other (*dependency injection*). Code outside that structure, such as a static helper method, has nothing to receive them with. `SpringContext` is a bridge for that case. It is a Spring component that remembers the application context when the application starts, and offers static methods:

| Method | Returns |
|--------|---------|
| `getBean(Class<T>)` | the bean of that type |
| `getBean(String)` | the bean with that name, as an `Object` |
| `getHttpServletRequest()` | the request being handled by the current thread |
| `getHttpServletResponse()` | the response of the current request |
| `getCookieValue(name)` | the value of a cookie of the current request, or `null` |

The class has to be found by the component scan, so the application includes its package:

```java
@SpringBootApplication(scanBasePackages = {"com.example.shop", "org.j2os.platform.jutil.spring"})
public class ShopApplication {

    public static void main(String[] args) {
        SpringApplication.run(ShopApplication.class, args);
    }
}
```

Here is a static helper that writes an audit record. It is called from anywhere, with no parameters for the request or the service:

```java
import jakarta.servlet.http.HttpServletRequest;
import org.j2os.platform.jutil.spring.SpringContext;

public final class AuditTrail {

    private AuditTrail() {
    }

    public static void record(String action) {
        HttpServletRequest request = SpringContext.getHttpServletRequest();
        String session = SpringContext.getCookieValue("session");   // null when the cookie is absent
        AuditService service = SpringContext.getBean(AuditService.class);

        service.save(action, request.getRemoteAddr(), session);
    }
}
```

And the way a controller uses it:

```java
@Service
public class AuditService {

    public void save(String action, String address, String session) {
        // write one row to the audit table
    }
}

@RestController
public class OrderController {

    @PostMapping("/api/orders/{id}/cancel")
    public void cancel(@PathVariable long id) {
        // ... cancel the order ...
        AuditTrail.record("order " + id + " cancelled");   // a plain static call, no injection
    }
}
```

Three rules apply:

- The bean lookups work after the application has started. Before that they throw `IllegalStateException`.
- The request and response exist only on the thread that is handling a request. A thread that you start yourself has none, and the methods throw `IllegalStateException`.
- Injection through the constructor is still the better choice wherever it is possible. `SpringContext` is meant for the places where it is not.

## 16. From idea to application: a purchase receipt

A shop receives an order as JSON from a mobile application. The customer types the amount, the order date is in the Persian calendar, and the position comes from the phone. The program reads the order, finds the nearest branch within 100 km, calculates the payment date 30 days after the order, and answers with a receipt in JSON.

```java
import org.j2os.platform.jutil.date.JDate;
import org.j2os.platform.jutil.geo.BoundingBox;
import org.j2os.platform.jutil.geo.Geo;
import org.j2os.platform.jutil.json.JSon;
import org.j2os.platform.jutil.number.JNumber;

import java.time.LocalDate;
import java.util.List;
import java.util.Locale;

public class Receipt {

    record Branch(String name, double latitude, double longitude) {
    }

    public record Answer(String branch, String customer, String distance,
                         String dueDate, String total, String totalInWords) {
    }

    public static void main(String[] args) {
        String request = "{\"customer\":\"Sara\",\"amount\":\"۱٬۲۵۰٬۰۰۰\",\"orderDate\":\"1404/01/15\","
                + "\"latitude\":\"29.61\",\"longitude\":\"52.53\"}";

        // 1. Read what the client sent.
        String customer = JSon.readFieldAsText(request, "customer");
        String latitude = JSon.readFieldAsText(request, "latitude");
        String longitude = JSon.readFieldAsText(request, "longitude");
        if (!Geo.isValidCoordinate(latitude, longitude)) {
            throw new IllegalArgumentException("The location is not valid");
        }

        // 2. Turn the typed amount into a plain number.
        String typed = JSon.readFieldAsText(request, "amount");
        String amount = JNumber.getEnglishNumberWithoutCommas(JNumber.getEnglishNumber(typed));

        // 3. The payment is due 30 days after the order date (given in the Persian calendar).
        LocalDate order = new JDate().getGregorianDateTimestamp(JSon.readFieldAsText(request, "orderDate"))
                .toLocalDateTime().toLocalDate();
        JDate due = new JDate(order.getYear(), order.getMonthValue(), order.getDayOfMonth());
        due.nextDay(30);

        // 4. The nearest branch within 100 km: a cheap box test first, the exact distance second.
        List<Branch> branches = List.of(
                new Branch("Tehran", 35.6892, 51.3890),
                new Branch("Isfahan", 32.6546, 51.6680),
                new Branch("Shiraz", 29.5918, 52.5837));
        double lat = Double.parseDouble(latitude), lon = Double.parseDouble(longitude);
        List<BoundingBox> boxes = Geo.boundingBoxAround(latitude, longitude, 100_000);

        Branch nearest = null;
        double nearestMeters = Double.MAX_VALUE;
        for (Branch branch : branches) {
            boolean inside = false;
            for (BoundingBox box : boxes) {
                inside |= branch.latitude() >= box.minLatitude && branch.latitude() <= box.maxLatitude
                        && branch.longitude() >= box.minLongitude && branch.longitude() <= box.maxLongitude;
            }
            if (!inside) continue;
            double meters = Geo.distanceMeters(lat, lon, branch.latitude(), branch.longitude());
            if (meters < nearestMeters) {
                nearest = branch;
                nearestMeters = meters;
            }
        }

        // 5. Build the answer.
        Answer answer = new Answer(nearest.name(), customer,
                String.format(Locale.ROOT, "%.1f km", nearestMeters / 1000),
                due.getPersianDate(), JNumber.getPersianNumber(amount), JNumber.getPersianWords(amount));
        System.out.println(JSon.write(answer));
    }
}
```

```text
{"branch":"Shiraz","customer":"Sara","distance":"5.6 km","dueDate":"1404/02/14","total":"۱,۲۵۰,۰۰۰","totalInWords":"یک میلیون و دویست و پنجاه هزار"}
```

The program uses every part of the module:

1. `JSon.readFieldAsText` reads the fields, and `Geo.isValidCoordinate` rejects a position that makes no sense.
2. The amount `۱٬۲۵۰٬۰۰۰` is cleaned with `getEnglishNumber` and `getEnglishNumberWithoutCommas`, as in chapter 10.
3. `getGregorianDateTimestamp` turns the Persian order date into a date that Java understands, and `JDate.nextDay(30)` moves it forward. The answer is read back with `getPersianDate()`.
4. `boundingBoxAround` removes the branches that are far away, and `distanceMeters` measures the rest exactly. Isfahan and Tehran are outside the box.
5. `getPersianNumber` and `getPersianWords` write the amount in digits and in words, and `JSon.write` produces the answer.

The same steps work in a Spring controller. The request body and the answer are the same JSON, and the middle of the method does not change.

---

## Appendix A. Quick reference

### A.1 JDate

| Member | Description |
|--------|-------------|
| `new JDate()` | today |
| `new JDate(year, month, day)` | a Gregorian date |
| `setGregorianDate(y, m, d)` / `setPersianDate(y, m, d)` | replaces the date; validates it |
| `getGregorianDate()` / `getPersianDate()` | the date as `yyyy/MM/dd` |
| `getGregorianYear()`, `getGregorianMonth()`, `getGregorianDay()` | Gregorian parts |
| `getPersianYear()`, `getPersianMonth()`, `getPersianDay()` | Persian parts |
| `addDays(n)`, `nextDay()`, `nextDay(n)`, `previousDay()`, `previousDay(n)` | moves the date |
| `getDayOfWeek()` | 0 (Monday) to 6 (Sunday) |
| `getWeekDayName()` / `getPersianWeekDayName()` | name of the weekday |
| `isLeap()` / `isLeap(year)` | Persian leap year of the date / of any Persian year |
| `getGregorianDateTimestamp(persianText)` | Persian `yyyy/MM/dd` to `Timestamp` |
| `getPersianDateString(timestamp)` | `Timestamp` to Persian `yyyy/MM/dd` |
| `toString()` | weekday and both dates |

### A.2 JNumber

| Method | Description |
|--------|-------------|
| `getEnglishWords(text)` | the number in English words |
| `getPersianWords(text)` | the number in Persian words |
| `getPersianNumber(text)` | Persian digits, comma-grouped, `٫` before the fraction |
| `getEnglishNumber(text)` | Latin digits, comma-grouped, `٫` before the fraction |
| `getEnglishNumberWithoutCommas(text)` | plain Latin number with `.` as the decimal point |

### A.3 JSon

| Method | Description |
|--------|-------------|
| `write(object)` | object to JSON text |
| `read(json, Type.class)` | JSON text to object |
| `readFieldAsText(json, names...)` | field value as plain text; empty text if missing |
| `readFieldAsString(json, names...)` | field as JSON text |
| `readFieldArrayAsText(json, index, field)` | field of an array element, as plain text |
| `readFieldArrayAsString(json, index, field)` | field of an array element, as JSON text |

### A.4 Geo

| Method | Description |
|--------|-------------|
| `distanceMeters(lat1, lon1, lat2, lon2)` | distance in meters |
| `midpointBetween(lat1, lon1, lat2, lon2)` | the middle point, as a `Coordinate` |
| `isValidCoordinate(latitudeText, longitudeText)` | `true` if both are numbers in range |
| `boundingBoxAround(latitudeText, longitudeText, radiusMeters)` | one or two `BoundingBox` objects |

`Coordinate` has the fields `latitude` and `longitude`. `BoundingBox` has `minLatitude`, `maxLatitude`, `minLongitude` and `maxLongitude`.

### A.5 SpringContext

| Method | Description |
|--------|-------------|
| `getBean(Class)` / `getBean(String)` | looks up a bean |
| `getHttpServletRequest()` / `getHttpServletResponse()` | the current request / response |
| `getCookieValue(name)` | a cookie of the current request or `null` |

## Appendix B. Failure handling

The classes of jutil do not hide problems. A value they cannot handle throws a standard exception. The table shows which one.

| Situation | Result |
|-----------|--------|
| date text is `null`, has the wrong shape, or has a number out of range | `IllegalArgumentException` |
| a day that does not exist in its month (`1404/12/30`, `2023/2/29`) | `IllegalArgumentException` |
| `JNumber` words: text with a character that is not a digit, or two decimal points | `IllegalArgumentException` |
| `JNumber` words: more than 36 integer digits | `ArithmeticException` |
| `JNumber` words or formatting with `null` or empty text | empty text, no exception |
| `Geo`: latitude or longitude out of range, `NaN` or infinite | `IllegalArgumentException` |
| `Geo.boundingBoxAround`: negative radius | `IllegalArgumentException` |
| `Geo.boundingBoxAround`: `null` coordinate | `NullPointerException` |
| `Geo.boundingBoxAround`: coordinate that is not a number | `NumberFormatException` |
| `Geo.midpointBetween`: antipodal points | `IllegalArgumentException` |
| `Geo.isValidCoordinate` with any input | never throws |
| `JSon`: invalid JSON, or a text that does not fit the class | `JacksonException` (unchecked) |
| `JSon.readField...`: missing field or array index | no exception; empty text or the text of an absent node |
| `SpringContext`: used before the application started, or outside a request | `IllegalStateException` |
| `SpringContext.getBean`: no such bean, or more than one candidate | Spring's `BeansException` |
| `SpringContext.getCookieValue`: `null` or blank name | `IllegalArgumentException` |

Out-of-range `Geo` arguments and the other `Geo` failures look like this:

```java
import org.j2os.platform.jutil.geo.Geo;

public class GeoErrors {

    public static void main(String[] args) {
        try {
            Geo.distanceMeters(95, 0, 0, 0);
        } catch (IllegalArgumentException e) {
            System.out.println("distanceMeters -> " + e.getMessage());
        }
        try {
            Geo.midpointBetween(0, 0, 0, 180);
        } catch (IllegalArgumentException e) {
            System.out.println("midpointBetween -> " + e.getMessage());
        }
        try {
            Geo.boundingBoxAround("35.7", "51.4", -1);
        } catch (IllegalArgumentException e) {
            System.out.println("boundingBoxAround -> " + e.getMessage());
        }
        try {
            Geo.boundingBoxAround("north", "51.4", 100);
        } catch (NumberFormatException e) {
            System.out.println("boundingBoxAround -> " + e.getMessage());
        }
        try {
            Geo.boundingBoxAround(null, "51.4", 100);
        } catch (NullPointerException e) {
            System.out.println("boundingBoxAround -> " + e.getMessage());
        }
    }
}
```

```text
distanceMeters -> Invalid latitude: 95.0
midpointBetween -> Midpoint is undefined for antipodal points
boundingBoxAround -> radiusMeters must not be negative
boundingBoxAround -> For input string: "north"
boundingBoxAround -> latitudeText must not be null
```

And invalid JSON:

```java
import org.j2os.platform.jutil.json.JSon;
import tools.jackson.core.JacksonException;

public class JsonErrors {

    public static void main(String[] args) {
        try {
            JSon.read("{not json}", Object.class);
        } catch (JacksonException e) {
            System.out.println("read(): invalid JSON rejected");
        }

        try {
            JSon.readFieldAsText("{\"name\":", "name");
        } catch (JacksonException e) {
            System.out.println("readFieldAsText(): invalid JSON rejected");
        }
    }
}
```

```text
read(): invalid JSON rejected
readFieldAsText(): invalid JSON rejected
```

`JacksonException` belongs to Jackson 3 and is unchecked, so no `throws` clause is needed. Catch it where you can answer the sender with a clear "the request is not valid JSON".

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **`JDate` is mutable.** `nextDay()` and the setters change the object. Copy it first when you need the original (chapter 4).
- **Day numbering.** `getDayOfWeek()` starts at Monday = 0, which differs from `java.util.Calendar` and from `java.time.DayOfWeek` (Monday = 1).
- **Date texts are strict.** Only `/` separates the parts, and the order is year, month, day. A text such as `1403-01-01` or `01/01/1403` fails.
- **Words need clean numbers.** `getEnglishWords("+5")` and `getEnglishWords("1 000")` fail. Clean user input first (chapter 10).
- **The cleaner is generous.** `getPersianNumber("1e5")` returns `۱۵`. Validate the text before you format it if it must be a number.
- **Fractions are not rounded or grouped.** The digits after the decimal separator are left as they are.
- **`getEnglishNumber` still uses `٫`.** The name says "English" for the digits only. Pass the result through `getEnglishNumberWithoutCommas` before parsing it.
- **A missing JSON field is an empty text.** `readFieldAsText` cannot tell a missing field from an empty string.
- **The bounding box is not a circle.** Filter its results with the exact distance.

### C.2 Behavior worth knowing

- `JDate` keeps one internal day number and derives both calendars from it, so the two never disagree.
- `getGregorianDateTimestamp` and `getPersianDateString` use a temporary date and leave the called object unchanged.
- The Persian names of large numbers: «هزار» (10³), «میلیون» (10⁶), «میلیارد» (10⁹), «بیلیون» (10¹²), «بیلیارد» (10¹⁵), «تریلیون» (10¹⁸), and so on. Beyond «میلیارد» the Persian names do not line up with the English ones, so 10¹² is «یک بیلیون» in Persian and «One Trillion» in English. The number is the same; only the name differs.
- `midpointBetween` returns a longitude in the range -180 (included) to 180 (excluded), so the middle of 170° and -170° is -180°.
- `isValidCoordinate` accepts spaces around a number and the exponent form (`1e1`).

### C.3 Limits

- `JDate` holds a day, not a time. For times use `java.time`.
- `JDate` is not thread-safe. Do not share one object between threads that change it.
- The Persian leap years come from a table of break points that ends at the Persian year 3178. Dates of practical interest are far inside it.
- Words are produced for numbers of up to 36 integer digits. The formatting methods have no such limit.
- `Geo` uses a spherical Earth. Distances can differ from the exact ellipsoid distance by up to about half a percent, which is acceptable for "nearby" searches and not for surveying.
- `JSon` uses one mapper with default settings. When you need other settings, for example a special date format, build your own Jackson mapper for that purpose.

### C.4 Tuning

- Validate first, then convert. Use `isValidCoordinate` for coordinates and the rules of `jvalidation` for numbers and dates.
- Keep numbers as text until you do arithmetic. Parse them with `BigDecimal` for money and keep `getPersianNumber` for display only.
- Store dates in the database as Gregorian values and show the Persian date only when a person reads it.
- Do the bounding-box test in the query and the exact distance in Java.
- Use `SpringContext` in static helpers and leave constructor injection for everything else.

## Appendix D. Message reference

The messages below are the texts of the exceptions that jutil throws. A value in angle brackets is replaced by the actual value.

**JDate**

| Message | Meaning |
|---------|---------|
| `Date string must not be null` | the date text is `null` |
| `Expected format yyyy/MM/dd but got: "<text>"` | the text does not have three numbers separated by `/` |
| `Year must be positive, got: <year>` | the year is zero or negative |
| `Month must be between 1 and 12, got: <month>` | the month is out of range |
| `Day must be between 1 and 31, got: <day>` | the day in a date text is out of range |
| `Day must be between 1 and <max> for Persian <year>/<month>, got: <day>` | the day does not exist in that Persian month |
| `Day must be between 1 and <max> for Gregorian <year>/<month>, got: <day>` | the day does not exist in that Gregorian month |

**JNumber**

| Message | Meaning |
|---------|---------|
| `Invalid numeric string: contains non-digit characters.` | the text has a character that is not a digit |
| `Invalid numeric string: multiple decimal points.` | the text has more than one decimal point |
| `Number is too large to convert to words (exceeds supported magnitude).` | more than 36 integer digits (`getEnglishWords`) |
| `عدد برای تبدیل به حروف بسیار بزرگ است (خارج از محدوده‌ی پشتیبانی‌شده).` | the same, for `getPersianWords` |

**Geo**

| Message | Meaning |
|---------|---------|
| `Invalid latitude: <value>` | the latitude is outside -90 to 90, or is `NaN` or infinite |
| `Invalid longitude: <value>` | the longitude is outside -180 to 180, or is `NaN` or infinite |
| `radiusMeters must not be negative` | the radius of `boundingBoxAround` is negative |
| `latitudeText must not be null` / `longitudeText must not be null` | a coordinate of `boundingBoxAround` is `null` |
| `Midpoint is undefined for antipodal points` | the two points are exact opposites on the Earth |

**SpringContext**

| Message | Meaning |
|---------|---------|
| `ApplicationContext has not been initialized yet!` | a bean was requested before Spring started the application |
| `There is no current HTTP request (outside of request scope)` | the request or response was requested on a thread without a request |
| `Cookie name must not be null or blank` | `getCookieValue` received an empty name |
