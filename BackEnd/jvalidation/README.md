# jvalidation

*J2OS Platform 2.4*

Every application that accepts data has to check it: a registration form, a payment request, a record imported from a file. The checks are simple, but there are many of them, and they tend to end up as long chains of `if` statements that stop at the first problem. The user fixes one mistake, submits again, and meets the next one.

**jvalidation** (`org.j2os.platform.jvalidation`) replaces those chains with one readable description of the rules. Every rule is checked, and all the problems are returned together, each with the name of the field, a code that a program can recognize, and a message that a person can read.

```java
Validator.of(form).field(Form::email).required().regex(Patterns.EMAIL).validate()
```

The library has four public classes:

- `Validator` builds the chain of rules and runs them.
- `ValidationResult` holds the outcome: valid or not, and the list of errors.
- `Patterns` is a collection of ready-made regular expressions for common formats, including several Iranian ones.
- `ValidationException` carries a failed result when you prefer exceptions.

Here is the route through the guide: each chapter has a short program followed by the output it prints. The appendices are meant for quick lookups.

## Contents

1. [The idea behind a Validator](#1-the-idea-behind-a-validator)
2. [Your first validation](#2-your-first-validation)
3. [Reading the result](#3-reading-the-result)
4. [Rules for text](#4-rules-for-text)
5. [Patterns and regular expressions](#5-patterns-and-regular-expressions)
6. [Rules for numbers](#6-rules-for-numbers)
7. [Rules for true and false](#7-rules-for-true-and-false)
8. [Rules for dates](#8-rules-for-dates)
9. [Rules for collections](#9-rules-for-collections)
10. [Rules that look at more than one field](#10-rules-that-look-at-more-than-one-field)
11. [Missing values and optional fields](#11-missing-values-and-optional-fields)
12. [Messages and message keys](#12-messages-and-message-keys)
13. [How field names are found](#13-how-field-names-are-found)
14. [In a web application](#14-in-a-web-application)
15. [In practice: a registration form](#15-in-practice-a-registration-form)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The programs use Java records (simple, immutable classes that hold data) to keep the sample objects short. An ordinary class with getter methods works in exactly the same way (chapter 13). Persian calendar rules need the `jutil` module on the classpath, because the library uses its `JDate` class for calendar arithmetic.

---

## 1. The idea behind a Validator

A validation is written as one chain, from left to right.

```java
ValidationResult result = Validator.of(form)          // the object to check
        .field(Form::username).required().minLength(5)   // rules for one field
        .field(Form::email).regex(Patterns.EMAIL)        // rules for the next field
        .validate();                                     // all errors, collected
```

Read aloud, it says: the username is required and has at least 5 characters, the email matches the email pattern. Four ideas explain everything else.

**The object and its fields.** `Validator.of(object)` starts a chain for one object. `.field(Form::username)` selects one field of it by a *method reference*, the Java notation `ClassName::methodName` that points to a method without calling it. The rules that follow apply to that field until the next `.field(...)`.

**Errors are collected, not thrown.** A rule that fails does not stop anything. It records an error, and the chain continues with the next rule. At the end the caller receives every error at once.

**Rules run immediately.** This is called *eager* evaluation. Each rule is checked at the moment it appears in the chain, not later when `validate()` is called. `validate()` only hands over the errors recorded so far. For this reason all the rules must be attached before `validate()`, and `Validator.of(x).validate()` with no rules is valid.

```java
Validator.of(user)
        .field(User::getAge).min(18)       // runs now, error recorded now
        .validate();                       // only collects what has already been recorded
```

**A Validator is used once.** Create a new one for each object that you validate. It keeps a list of errors inside and is not meant to be shared between threads.

## 2. Your first validation

A sign-up form with three fields, filled in with three mistakes:

```java
import org.j2os.platform.jvalidation.ValidationResult;
import org.j2os.platform.jvalidation.Validator;

public class FirstValidation {

    record Signup(String username, String email, String password) {
    }

    public static void main(String[] args) {
        Signup form = new Signup("ali", "not-an-email", "secret");

        ValidationResult result = Validator.of(form)
                .field(Signup::username).required().minLength(5)
                .field(Signup::email).required().regex("^[^\\s@]+@[^\\s@]+\\.[^\\s@]+$")
                .field(Signup::password).minLength(8)
                .validate();

        System.out.println("valid: " + result.isValid() + ", errors: " + result.errorCount());
        result.errors().forEach(error ->
                System.out.println(error.getField() + " | " + error.getCode() + " | " + error.getMessage()));
    }
}
```

```text
valid: false, errors: 3
username | MIN_LENGTH | username must be at least 5 characters
email | REGEX | email does not match pattern ^[^\s@]+@[^\s@]+\.[^\s@]+$
password | MIN_LENGTH | password must be at least 8 characters
```

- `required()` and `minLength(5)` belong to the username. `"ali"` has 3 characters, so `MIN_LENGTH` is recorded.
- `regex(...)` takes a regular expression, a compact description of a text shape (chapter 5). The email has no dot after the `@`, so `REGEX` is recorded.
- The password rule fails too, and the result contains all three errors, in the order in which the rules were checked.

Each error says which **field** failed, which **code** names the rule, and a **message** in plain English. The next chapter looks at these in detail.

## 3. Reading the result

`validate()` returns a `ValidationResult`. It has three members:

| Member | Meaning |
|--------|---------|
| `isValid()` | `true` when there are no errors |
| `errorCount()` | the number of errors |
| `errors()` | the list of errors, in the order of the rules; it cannot be modified |

Every element of the list is a `ValidationResult.Error` with these getters:

| Getter | Meaning |
|--------|---------|
| `getField()` | the field name, derived from the method reference (`username`) |
| `getCode()` | a stable, machine-readable code of the rule (`MIN_LENGTH`) |
| `getMessage()` | a readable message; it can be replaced (chapter 12) |
| `getMessageKey()` | an optional key for translations; `null` unless you set one |
| `getInvalidValue()` | the value that failed |

```java
import org.j2os.platform.jvalidation.ValidationException;
import org.j2os.platform.jvalidation.ValidationResult;
import org.j2os.platform.jvalidation.Validator;

public class ResultDemo {

    record Login(String username, String password) {
    }

    public static void main(String[] args) {
        Login login = new Login("ali", "123");

        ValidationResult result = Validator.of(login)
                .field(Login::password).minLength(8)
                .validate();

        ValidationResult.Error error = result.errors().get(0);
        System.out.println("field: " + error.getField());
        System.out.println("code: " + error.getCode());
        System.out.println("invalid value: " + error.getInvalidValue());
        System.out.println("toString: " + error);

        // validateOrThrow: the same check, but a failure becomes an exception
        try {
            Validator.of(login)
                    .field(Login::username).minLength(5)
                    .field(Login::password).minLength(8)
                    .validateOrThrow();
        } catch (ValidationException e) {
            System.out.println("errors in the exception: " + e.getResult().errorCount());
        }
    }
}
```

```text
field: password
code: MIN_LENGTH
invalid value: 123
toString: Error{field='password', code='MIN_LENGTH', message='password must be at least 8 characters'}
errors in the exception: 2
```

- The code is what a program should compare. Messages may be reworded, codes do not change.
- `toString()` of an error leaves the invalid value out on purpose. The value is often a password, and error texts end up in logs. Read `getInvalidValue()` explicitly when you really need it.
- `validateOrThrow()` works like `validate()` but throws a `ValidationException` when at least one error exists. The exception carries the whole result: `e.getResult()` returns the same list of errors. Use it where a failed validation should stop the current operation, for example at the top of a service method.

## 4. Rules for text

Rules for text look at the string form of the value, so they also work on numbers and other objects. A `null` value fails every one of them.

| Rule | Passes when |
|------|-------------|
| `required()` | the value is not `null` and, for a string, not empty |
| `notBlank()` | the string has at least one character that is not whitespace |
| `length(n)` | the length is exactly `n` |
| `minLength(n)`, `maxLength(n)` | the length is at least / at most `n` |
| `lengthBetween(min, max)` | the length is between the two, both included |
| `contains(x)`, `startsWith(x)`, `endsWith(x)` | the text contains / starts with / ends with `x` |

```java
import org.j2os.platform.jvalidation.ValidationResult;
import org.j2os.platform.jvalidation.Validator;

public class TextRules {

    record Profile(String nickname, String code, String bio, String note) {
    }

    public static void main(String[] args) {
        Profile profile = new Profile("   ", "AB1", "engineer at IR", "");

        ValidationResult result = Validator.of(profile)
                .field(Profile::nickname).required()  // "   " is not empty: passes
                .notBlank()  // but it is blank: fails
                .field(Profile::code).length(4)  // 3 characters: fails
                .field(Profile::bio).lengthBetween(5, 40)  // passes
                .contains("developer")  // fails
                .startsWith("engineer")  // passes
                .endsWith("IR")  // passes
                .field(Profile::note).required()  // "" counts as missing: fails
                .validate();

        result.errors().forEach(error -> System.out.println(error.getField() + " -> " + error.getCode()));
    }
}
```

```text
nickname -> NOT_BLANK
code -> LENGTH
bio -> CONTAINS
note -> REQUIRED
```

- A string of spaces passes `required()` because it is not empty, and fails `notBlank()`. For text typed by people, use `notBlank()`.
- The empty string `""` fails `required()`.
- Only the failed rules appear in the output. Every rule that passes leaves no trace.

## 5. Patterns and regular expressions

A *regular expression* (regex) describes the shape of a text: `[a-z]+` means "one or more lowercase letters". `regex(...)` accepts either a string or a ready `java.util.regex.Pattern`, and the **whole** value must match, not just a part of it.

```java
import org.j2os.platform.jvalidation.Patterns;
import org.j2os.platform.jvalidation.ValidationResult;
import org.j2os.platform.jvalidation.Validator;

public class PatternRules {

    record Contact(String mobile, String nationalCode, String email, String code) {
    }

    public static void main(String[] args) {
        Contact contact = new Contact("09121234567", "12345", "ali@example.com", "abc123");

        ValidationResult result = Validator.of(contact)
                .field(Contact::mobile).regex(Patterns.MOBILE)  // passes
                .field(Contact::nationalCode).regex(Patterns.NATIONAL_CODE)  // 5 digits instead of 10: fails
                .field(Contact::email).regex(Patterns.EMAIL)  // passes
                .field(Contact::code).regex("[a-z]+")  // the whole value must match: fails
                .field(Contact::code).regex("[a-z]+[(")  // not a valid expression: INVALID_PATTERN
                .validate();

        result.errors().forEach(error -> System.out.println(error.getField() + " -> " + error.getCode()));
        System.out.println(result.errors().get(0).getMessage());
    }
}
```

```text
nationalCode -> REGEX
code -> REGEX
code -> INVALID_PATTERN
nationalCode has an invalid format
```

- `regex(Patterns.MOBILE)` uses a ready-made pattern, so the code does not repeat the expression.
- The national code has 5 digits instead of 10, so `Patterns.NATIONAL_CODE` rejects it. The message for a `Pattern` is "has an invalid format"; the message for a string pattern names the pattern.
- `"abc123"` does not fully match `[a-z]+`, so it fails.
- An expression that is not valid, like `[a-z]+[(`, does not throw an exception. It records an `INVALID_PATTERN` error, so a typo in a rule cannot crash a request.

### 5.1 The Patterns class

| Constant | Accepts |
|----------|---------|
| `MOBILE` | Iranian mobile number: `09` and 9 more digits |
| `NATIONAL_CODE` | Iranian national code: 10 digits |
| `LEGAL_ENTITY_ID` | Iranian legal entity id: 11 digits |
| `POSTAL_CODE` | Iranian postal code: 10 digits |
| `LANDLINE_IR` | Iranian landline: `0`, area code, number |
| `SHEBA_NUMBER` | Iranian IBAN (Sheba): `IR` and 24 digits |
| `CARD_NUMBER` | bank card: 16 digits |
| `LICENSE_PLATE_IR` | Iranian plate: `NN<letter>NNN-NN` |
| `EMAIL`, `URL` | basic email shape; `http` or `https` URL with a dot in the domain |
| `USERNAME` | 3 to 36 letters, digits or `_` |
| `STRONG_PASSWORD` | at least 8 characters with a lowercase letter, an uppercase letter, a digit and one of `@ $ ! % * ? &` |
| `IPV4`, `IPV6`, `MAC_ADDRESS` | network addresses |
| `SLUG`, `UUID`, `BASE64`, `JWT` | URL slug, UUID, Base64 data, JSON Web Token shape |
| `IBAN`, `CREDIT_CARD_GENERIC` | international IBAN shape; 13 to 19 digits |
| `HEX_COLOR`, `TIME_24H`, `DATE_ISO` | `#RGB` or `#RRGGBB`; `HH:mm[:ss]`; `yyyy-MM-dd` |
| `DOMAIN_NAME`, `IMEI` | domain name; 15 digits |
| `PERSIAN_TEXT`, `ENGLISH_TEXT`, `TEXT_ANY_LANGUAGE` | letters of that script and spaces, without digits |
| `PERSIAN_DIGITS`, `ENGLISH_DIGITS` | only `۰-۹`; only `0-9` |

These patterns check the *shape* of a value, not its meaning. `NATIONAL_CODE` accepts any 10 digits, and `CREDIT_CARD_GENERIC` does not run a checksum. If a value needs a deeper check, add it with `withMethod` (chapter 10).

## 6. Rules for numbers

Number rules accept real numbers (`int`, `double`, `BigDecimal`, ...) and also strings that contain a number. A value that is not a number does not throw an exception: the rule fails.

| Rule | Passes when |
|------|-------------|
| `number()` | the string looks like a number: optional `-`, digits, optional `.digits` |
| `min(x)`, `max(x)` | the value is at least / at most `x` |
| `between(min, max)` | the value is in the range, both ends included |
| `positive()` | the value is greater than zero |
| `digits(i, f)` | at most `i` integer digits and `f` fraction digits |

```java
import org.j2os.platform.jvalidation.ValidationResult;
import org.j2os.platform.jvalidation.Validator;

public class NumberRules {

    record Product(double price, String quantity, String discount, double shipping) {
    }

    public static void main(String[] args) {
        Product product = new Product(12345.678, "42", "abc", 0);

        ValidationResult result = Validator.of(product)
                .field(Product::price).min(100).max(5000)  // above 5000: max fails
                .digits(4, 2)  // 5 integer digits, 3 fraction digits: fails
                .field(Product::quantity).number().between(1, 100)  // "42" is a number inside the range: passes
                .field(Product::discount).number()  // "abc": NUMBER error
                .field(Product::shipping).positive()  // 0 is not positive: fails
                .validate();

        result.errors().forEach(error -> System.out.println(error.getField() + " -> " + error.getCode()));
    }
}
```

```text
price -> MAX
price -> DIGITS
discount -> NUMBER
shipping -> POSITIVE
```

- `price` is above 5000, so `MAX` fails. It also has five integer digits and three fraction digits, so `DIGITS(4, 2)` fails.
- `"42"` is a string, but it is a number and lies in the range, so it passes. `number()` placed first makes the shape of a string explicit.
- `"abc"` records `NUMBER`.
- Zero is not positive.

`digits` ignores trailing zeros: `1200.00` has 4 integer digits and no fraction digits, so it passes `digits(4, 2)`.

## 7. Rules for true and false

| Rule | Passes when |
|------|-------------|
| `isTrue()` | the value is `true` (a `Boolean`, or the text `"true"` in any letter case) |
| `isFalse()` | the value is `false` (a `Boolean`, or the text `"false"`) |
| `bool()` | the text is exactly `true` or `false`, in any letter case |

```java
import org.j2os.platform.jvalidation.ValidationResult;
import org.j2os.platform.jvalidation.Validator;

public class BooleanRules {

    record Consent(boolean termsAccepted, boolean newsletter, String marketing) {
    }

    public static void main(String[] args) {
        Consent consent = new Consent(false, false, "yep");

        ValidationResult result = Validator.of(consent)
                .field(Consent::termsAccepted).isTrue()  // must be accepted: fails
                .field(Consent::newsletter).isFalse()  // passes
                .field(Consent::marketing).bool()  // "yep" is neither true nor false: fails
                .validate();

        result.errors().forEach(error -> System.out.println(error.getField() + " -> " + error.getCode()));
    }
}
```

```text
termsAccepted -> IS_TRUE
marketing -> BOOLEAN
```

`isTrue()` is the usual way to require that a checkbox such as "I accept the terms" was ticked. Use `bool()` when the value arrives as text and you want to be sure it is a boolean at all.

## 8. Rules for dates

Dates are given as text in the form `yyyy/MM/dd` with slashes, in either the Gregorian or the Persian (Jalali) calendar.

| Rule | Meaning |
|------|---------|
| `date()` | a real Gregorian date (`2020/02/30` is rejected; leap years are respected). Selects the Gregorian calendar for the rules after it |
| `persianDate()` | a real Persian date, with leap years. Selects the Persian calendar |
| `past()`, `future()` | strictly before / after today, in the selected calendar |
| `before(d)`, `after(d)` | strictly before / after the date `d` |
| `between(start, end)` | between the two dates, both included |
| `minimumAge(n)` | treating the value as a birth date, the person has completed at least `n` years |

```java
import org.j2os.platform.jvalidation.ValidationResult;
import org.j2os.platform.jvalidation.Validator;

public class DateRules {

    record Booking(String birth, String checkIn, String checkOut, String persianBirth, String broken) {
    }

    public static void main(String[] args) {
        Booking booking = new Booking("2025/03/20", "2040/06/10", "2040/06/05", "1370/01/15", "2020/13/40");

        ValidationResult result = Validator.of(booking)
                .field(Booking::birth).date().past().minimumAge(18)  // a child: minimumAge fails
                .field(Booking::checkIn).date().future()  // passes
                .between("2040/06/01", "2040/06/30")  // passes
                .field(Booking::checkOut).date().after("2040/06/10")  // before check-in: fails
                .field(Booking::persianBirth).persianDate().past().minimumAge(18)  // passes
                .field(Booking::broken).date().past()  // not a date: only DATE is reported
                .validate();

        result.errors().forEach(error -> System.out.println(error.getField() + " -> " + error.getCode()));
    }
}
```

```text
birth -> MINIMUM_AGE
checkOut -> AFTER
broken -> DATE
```

- The date rules after `date()` or `persianDate()` use the calendar that it selected. The two calendars are never mixed in one field.
- The person born in 2025 is not 18, so `MINIMUM_AGE` fails. The check counts completed years, as of today.
- The check-out date is before the check-in date, so `AFTER` fails.
- `"2020/13/40"` is not a date. Only `DATE` is reported: the rules after a failed `date()` stay silent, so one wrong value does not produce a pile of errors.
- A birth date in the future gives `FUTURE_BIRTH_DATE` from `minimumAge`.

Two mistakes get their own codes:

```java
import org.j2os.platform.jvalidation.ValidationResult;
import org.j2os.platform.jvalidation.Validator;

public class DateContext {

    record Event(String day, String other) {
    }

    public static void main(String[] args) {
        Event event = new Event("2040/01/01", "2040/01/01");

        ValidationResult result = Validator.of(event)
                .field(Event::day).future()  // date() was forgotten
                .field(Event::other).date().before("2040-01-05")  // the comparison date has the wrong shape
                .validate();

        result.errors().forEach(error -> System.out.println(error.getField() + " -> " + error.getCode()));
        System.out.println(result.errors().get(1).getMessage());
    }
}
```

```text
day -> DATE_CONTEXT_MISSING
other -> INVALID_COMPARISON_DATE
other: otherDate ('2040-01-05') is not a valid Gregorian date (yyyy/MM/dd)
```

- `future()` without a `date()` before it cannot know the calendar, so it reports `DATE_CONTEXT_MISSING`.
- The comparison date of `before`, `after` or `between` must itself be a valid date of the selected calendar. `"2040-01-05"` uses dashes, so `INVALID_COMPARISON_DATE` is recorded and the message names the offending date.

`between(min, max)` with numbers and `between(start, end)` with text dates are two different rules with the same name; the argument types choose the right one.

## 9. Rules for collections

These rules work on collections (`List`, `Set`), on maps (counted by their entries) and on arrays.

| Rule | Passes when |
|------|-------------|
| `notEmpty()` | there is at least one element |
| `size(n)`, `minSize(n)`, `maxSize(n)` | the number of elements is exactly / at least / at most `n` |
| `unique()` | no element appears twice (compared with `equals`) |
| `contains(x)` | the collection contains `x`; for a map, one of its values equals `x` |

```java
import org.j2os.platform.jvalidation.ValidationResult;
import org.j2os.platform.jvalidation.Validator;

import java.util.List;
import java.util.Map;

public class CollectionRules {

    record Order(List<String> items, String[] tags, Map<String, Integer> quantities) {
    }

    public static void main(String[] args) {
        Order order = new Order(
                List.of("pen", "pen"),
                new String[]{"gift", "fragile", "urgent", "extra"},
                Map.of());

        ValidationResult result = Validator.of(order)
                .field(Order::items).notEmpty().minSize(1).unique()  // duplicate "pen": unique fails
                .contains("book")  // no "book": fails
                .field(Order::tags).maxSize(3)  // arrays work too: 4 elements, fails
                .field(Order::quantities).notEmpty()  // an empty map: fails
                .validate();

        result.errors().forEach(error -> System.out.println(error.getField() + " -> " + error.getCode()));
    }
}
```

```text
items -> UNIQUE
items -> CONTAINS
tags -> MAX_SIZE
quantities -> NOT_EMPTY
```

A value that is not a collection, map or array fails these rules. Note that `required()` accepts an empty collection, because it is not `null`. To demand at least one element, use `notEmpty()` or `minSize(1)`.

## 10. Rules that look at more than one field

Some rules need the whole object: two passwords that must match, a field that is required only in some cases, a check that no built-in rule can express.

```java
import org.j2os.platform.jvalidation.ValidationResult;
import org.j2os.platform.jvalidation.Validator;

public class CrossFieldRules {

    record Account(String password, String confirm, boolean company, String companyName, String coupon) {
    }

    public static void main(String[] args) {
        Account account = new Account("Secret#123", "Secret#124", true, "", "WINTER10");

        ValidationResult result = Validator.of(account)
                .field(Account::password).equalTo(Account::confirm)  // the two passwords differ: fails
                .when(Account::company)  // the next rules run only for companies
                .field(Account::companyName).required()  // fails
                .endWhen()  // back to rules that always run
                .field(Account::coupon).withMethod(
                        (acc, value) -> value != null && value.startsWith("SUMMER"),
                        "COUPON_PREFIX", "coupon must start with SUMMER")  // custom rule: fails
                .validate();

        result.errors().forEach(error -> System.out.println(error.getField() + " -> " + error.getCode()));
    }
}
```

```text
password -> EQUAL_TO
companyName -> REQUIRED
coupon -> COUPON_PREFIX
```

- **`equalTo(getter)`** compares the current field with another getter of the same object. It is the usual "confirm password" rule.
- **`when(condition)` ... `endWhen()`** form a conditional block. The condition receives the whole object, and the rules between the two calls run only when it is true. Here the company name is required only when `company` is true. After `endWhen()`, rules run always again. If you forget `endWhen()`, every later rule stays conditional.
- **`withMethod(rule, code, message)`** adds your own rule. The rule receives the whole object and the field value, and returns `true` when the value is acceptable. If it fails, the error gets the code and message that you supply. If the rule itself throws an exception, it is recorded as a failure too (Appendix B).

## 11. Missing values and optional fields

A `null` value fails every rule, not only `required()`. This is intentional: a rule such as `minLength(5)` cannot say that nothing is long enough, so it reports a problem.

For a field that may be absent, check it only when it is present, with a conditional block.

```java
import org.j2os.platform.jvalidation.ValidationResult;
import org.j2os.platform.jvalidation.Validator;

public class NullRules {

    record Form(String phone, String nickname) {
    }

    public static void main(String[] args) {
        Form form = new Form(null, null);

        // A missing value fails every rule, not only required()
        ValidationResult strict = Validator.of(form)
                .field(Form::phone).minLength(5)
                .validate();
        System.out.println("strict: " + strict.errors().get(0).getCode());

        // An optional value: check it only when it is present
        ValidationResult optional = Validator.of(form)
                .when(f -> f.nickname() != null)
                .field(Form::nickname).minLength(3)
                .endWhen()
                .validate();
        System.out.println("optional: " + optional.isValid());
    }
}
```

```text
strict: MIN_LENGTH
optional: true
```

The first validation reports `MIN_LENGTH` for the missing phone number. In the second, the nickname rules run only when the nickname exists; it does not, so the form is valid.

## 12. Messages and message keys

Default messages are in English and mention the field, for example `username must be at least 5 characters`. Two methods change the error of the rule just before them:

- `message(text)` replaces the message text.
- `messageKey(key)` adds a key, so that the application can look up a translation or a company-approved wording instead of showing the default text.

```java
import org.j2os.platform.jvalidation.ValidationResult;
import org.j2os.platform.jvalidation.Validator;

public class MessageRules {

    record Signup(String username) {
    }

    public static void main(String[] args) {
        ValidationResult result = Validator.of(new Signup("ali"))
                .field(Signup::username).minLength(5)
                .message("The username is too short")
                .messageKey("signup.username.tooShort")
                .maxLength(20)  // passes, so message() below has nothing to change
                .message("never used")
                .validate();

        ValidationResult.Error error = result.errors().get(0);
        System.out.println(error.getCode() + " | " + error.getMessage() + " | " + error.getMessageKey());
        System.out.println("errors: " + result.errorCount());
    }
}
```

```text
MIN_LENGTH | The username is too short | signup.username.tooShort
errors: 1
```

Both methods affect only the **last rule on that field, and only if it failed**. After the passing `maxLength(20)` there is no error to change, so the second `message` is ignored. To translate every rule of a field, place `messageKey` after each rule.

## 13. How field names are found

The library does not need field names typed as strings. It reads the name from the method reference: `getFirstName` becomes `firstName`, and `isActive` becomes `active`. The prefix `get` or `is` is removed and the first letter is lowered, except for names that start with two capitals, so `getURL` stays `URL`. For a record, the accessor name is used as it is.

```java
import org.j2os.platform.jvalidation.ValidationResult;
import org.j2os.platform.jvalidation.Validator;

public class NameRules {

    public static class Member {
        public String getFirstName() { return ""; }
        public boolean isActive() { return false; }
        public String getURL() { return ""; }
    }

    public static void main(String[] args) {
        ValidationResult result = Validator.of(new Member())
                .field(Member::getFirstName).required()
                .field(Member::isActive).isTrue()
                .field(Member::getURL).required()
                .field(member -> member.getFirstName()).required()  // a lambda instead of a method reference
                .validate();

        result.errors().forEach(error -> System.out.println(
                error.getField().startsWith("lambda$") ? "lambda$..." : error.getField()));
    }
}
```

```text
firstName
active
URL
lambda$...
```

The last field is selected with a lambda, `member -> member.getFirstName()`. A lambda has no name to read, so its error gets a made-up name that starts with `lambda$`. Always use method references, so that errors carry useful field names.

## 14. In a web application

A common arrangement is to validate the request body in a controller and answer with HTTP status 400 and the list of errors. `validateOrThrow()` and one handler for `ValidationException` keep the controller free of `if` statements:

```java
import org.j2os.platform.jvalidation.Patterns;
import org.j2os.platform.jvalidation.ValidationException;
import org.j2os.platform.jvalidation.Validator;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.List;
import java.util.Map;

@RestController
public class RegistrationController {

    public record Registration(String username, String email, String password) {
    }

    @PostMapping("/api/register")
    public Map<String, String> register(@RequestBody Registration form) {
        Validator.of(form)
                .field(Registration::username).regex(Patterns.USERNAME)
                .field(Registration::email).regex(Patterns.EMAIL)
                .field(Registration::password).regex(Patterns.STRONG_PASSWORD)
                .validateOrThrow();                  // a problem leaves the method here

        return Map.of("status", "registered");       // only valid forms get this far
    }
}

@RestControllerAdvice
class ValidationErrorHandler {

    @ExceptionHandler(ValidationException.class)
    public ResponseEntity<Map<String, Object>> invalid(ValidationException e) {
        List<Map<String, String>> errors = e.getResult().errors().stream()
                .map(error -> Map.of(
                        "field", error.getField(),
                        "code", error.getCode(),
                        "message", error.getMessage()))
                .toList();
        return ResponseEntity.badRequest().body(Map.of("errors", errors));
    }
}
```

An invalid request now receives a response like this, which a front end can show next to each field:

```json
{
  "errors": [
    { "field": "username", "code": "REGEX", "message": "username has an invalid format" },
    { "field": "password", "code": "REGEX", "message": "password has an invalid format" }
  ]
}
```

The handler goes in its own file, is written once and is enough for every controller. The field names in the response are the names derived from the method references (chapter 13).

## 15. In practice: a registration form

A registration form has a username, an email, a mobile number, a national code, a Persian birth date, a password with confirmation, a terms checkbox and a list of interests. The program validates two submissions and turns each error into a message for the user, taking the text from a table of message keys, as an application with translations would do.

```java
import org.j2os.platform.jvalidation.Patterns;
import org.j2os.platform.jvalidation.ValidationResult;
import org.j2os.platform.jvalidation.Validator;

import java.util.List;
import java.util.Map;
import java.util.TreeMap;

public class RegistrationForm {

    record Registration(String username, String email, String mobile, String nationalCode,
                        String birthDate, String password, String confirmPassword,
                        boolean acceptTerms, List<String> interests) {
    }

    static Map<String, String> validate(Registration form) {
        // Message texts by key: in a real application they come from a resource bundle
        Map<String, String> texts = Map.of(
                "username.length", "The username must have 3 to 36 characters",
                "email.format", "The email address is not valid",
                "mobile.format", "The mobile number must look like 09121234567",
                "nationalCode.format", "The national code must have 10 digits",
                "birth.format", "The birth date must be a Persian date like 1370/05/10",
                "birth.age", "You must be at least 18 years old",
                "password.weak", "The password needs upper and lower case letters, a digit and one of @ $ ! % * ? &",
                "password.confirm", "The two passwords are different",
                "terms.required", "The terms must be accepted",
                "interests.size", "Choose between 1 and 3 interests");

        ValidationResult result = Validator.of(form)
                .field(Registration::username).regex(Patterns.USERNAME).messageKey("username.length")
                .field(Registration::email).regex(Patterns.EMAIL).messageKey("email.format")
                .field(Registration::mobile).regex(Patterns.MOBILE).messageKey("mobile.format")
                .field(Registration::nationalCode).regex(Patterns.NATIONAL_CODE).messageKey("nationalCode.format")
                .field(Registration::birthDate).persianDate().messageKey("birth.format")
                .minimumAge(18).messageKey("birth.age")
                .field(Registration::password).regex(Patterns.STRONG_PASSWORD).messageKey("password.weak")
                .field(Registration::confirmPassword).equalTo(Registration::password).messageKey("password.confirm")
                .field(Registration::acceptTerms).isTrue().messageKey("terms.required")
                .field(Registration::interests).minSize(1).messageKey("interests.size")
                .maxSize(3).messageKey("interests.size")
                .validate();

        // One text per field: the first error of the field, translated through its message key
        Map<String, String> problems = new TreeMap<>();
        result.errors().forEach(error -> problems.putIfAbsent(error.getField(),
                error.getMessageKey() == null ? error.getMessage() : texts.get(error.getMessageKey())));
        return problems;
    }

    public static void main(String[] args) {
        Registration bad = new Registration("al", "ali@example", "0912", "123456789", "1404/05/10",
                "secret", "Secret", false, List.of());
        Registration good = new Registration("ali_2000", "ali@example.com", "09121234567", "0123456789", "1370/05/10",
                "Secret@123", "Secret@123", true, List.of("java", "nuxt"));

        System.out.println("Bad form:");
        validate(bad).forEach((field, text) -> System.out.println("  " + field + ": " + text));
        System.out.println("Good form: " + validate(good).size() + " problems");
    }
}
```

```text
Bad form:
  acceptTerms: The terms must be accepted
  birthDate: You must be at least 18 years old
  confirmPassword: The two passwords are different
  email: The email address is not valid
  interests: Choose between 1 and 3 interests
  mobile: The mobile number must look like 09121234567
  nationalCode: The national code must have 10 digits
  password: The password needs upper and lower case letters, a digit and one of @ $ ! % * ? &
  username: The username must have 3 to 36 characters
Good form: 0 problems
```

What to notice:

- Every rule that can fail for a different reason has its own `messageKey`, so each reason maps to a precise text.
- The program keeps only the first error per field, which is what most forms display.
- `Patterns` supplies the formats, so the rules read like a description of the form.
- The valid submission produces no problems at all.

---

## Appendix A. Quick reference

### A.1 Validator

| Member | Description |
|--------|-------------|
| `Validator.of(target)` | starts a chain for one object |
| `field(getter)` | selects a field by a method reference; also available on a field to continue the chain |
| `when(predicate)` / `endWhen()` | starts / ends a conditional block of rules |
| `validate()` | returns the `ValidationResult` with the errors recorded so far |
| `validateOrThrow()` | the same, but throws `ValidationException` when there is an error |

### A.2 Rules of a field

| Group | Rules |
|-------|-------|
| presence | `required()`, `notBlank()` |
| text length | `length(n)`, `minLength(n)`, `maxLength(n)`, `lengthBetween(min, max)` |
| text content | `contains(x)`, `startsWith(x)`, `endsWith(x)`, `regex(String)`, `regex(Pattern)` |
| numbers | `number()`, `min(x)`, `max(x)`, `between(min, max)`, `positive()`, `digits(i, f)` |
| booleans | `bool()`, `isTrue()`, `isFalse()` |
| dates | `date()`, `persianDate()`, `past()`, `future()`, `before(d)`, `after(d)`, `between(start, end)`, `minimumAge(n)` |
| collections | `notEmpty()`, `size(n)`, `minSize(n)`, `maxSize(n)`, `unique()` |
| other fields and custom | `equalTo(getter)`, `withMethod(rule, code, message)` |
| messages | `message(text)`, `messageKey(key)` |

### A.3 ValidationResult and Error

| Member | Description |
|--------|-------------|
| `isValid()`, `errorCount()`, `errors()` | outcome, number of errors, unmodifiable list of errors |
| `Error.getField()`, `getCode()`, `getMessage()`, `getMessageKey()`, `getInvalidValue()` | the parts of one error |

### A.4 ValidationException

`ValidationException` extends `RuntimeException`. `getResult()` returns the `ValidationResult`. The exception message is `Validation failed with N error(s): [...]`.

## Appendix B. Failure handling

jvalidation avoids exceptions during validation. Problems with the data, and many mistakes in the rules, become errors in the result.

| Situation | What is recorded |
|-----------|------------------|
| a value is `null` | the rule fails with its own code (`REQUIRED`, `MIN_LENGTH`, ...) |
| a getter throws an exception | `FIELD_ACCESS_ERROR`; the rules after it on that field are skipped |
| a `withMethod` rule throws | the error of that rule, with the exception text added to the message |
| the `equalTo` getter throws | `EQUAL_TO`, with the exception text |
| a regular expression is invalid or `null` | `INVALID_PATTERN` |
| a number rule meets text that is not a number | the rule fails (`MIN`, `MAX`, ...); no exception |
| a collection rule meets a value that is not a collection | the rule fails |
| a date rule is used without a valid `date()` or `persianDate()` | `DATE_CONTEXT_MISSING`, or nothing extra if the date check itself already failed |
| the comparison date is not a valid date | `INVALID_COMPARISON_DATE` |
| `when(null)` | the condition counts as false; the rules in the block are skipped |

```java
import org.j2os.platform.jvalidation.ValidationResult;
import org.j2os.platform.jvalidation.Validator;

public class FailureDemo {

    record Box(String name, int[] sizes) {
        String broken() {
            throw new IllegalStateException("database is down");
        }
    }

    public static void main(String[] args) {
        Box box = new Box("crate", new int[]{1, 2});

        ValidationResult result = Validator.of(box)
                .field(Box::broken).required()  // the getter throws
                .field(Box::name).withMethod((b, v) -> {
                    throw new IllegalArgumentException("scanner offline");
                }, "SCAN", "scan failed")  // a custom rule throws
                .validate();

        result.errors().forEach(error -> System.out.println(error.getCode() + ": " + error.getMessage()));
    }
}
```

```text
FIELD_ACCESS_ERROR: broken: failed to evaluate field (IllegalStateException: database is down)
SCAN: scan failed (rule threw IllegalArgumentException: scanner offline)
```

The only exception that the library throws on purpose is `ValidationException` from `validateOrThrow()`.

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Lambda instead of method reference.** The field name becomes `lambda$...` (chapter 13).
- **A rule on a missing value.** Every rule fails on `null`. Guard optional fields with `when(...)` (chapter 11).
- **`required()` and whitespace.** A string of spaces passes `required()`. Use `notBlank()` for typed text.
- **`required()` and empty collections.** An empty list passes `required()`. Use `notEmpty()`.
- **`message(...)` on a passed rule.** It changes nothing, because the error belongs to the last failed rule only (chapter 12).
- **Forgotten `endWhen()`.** Every later field stays conditional.
- **Validating before all rules are attached.** Rules run as they are written; `validate()` does not run anything.
- **The `#` in a password.** `Patterns.STRONG_PASSWORD` accepts only `@ $ ! % * ? &` as the special character.
- **Dates with dashes.** The format is `yyyy/MM/dd` with slashes; `2040-01-05` is rejected.
- **Logging invalid values.** `getInvalidValue()` can be a password. Log the field and the code instead.

### C.2 Behavior worth knowing

- Errors are in the order in which the rules were written.
- Several errors can exist for one field: one for each failed rule.
- `validate()` returns a copy of the errors recorded at that moment. Calling it again after more rules returns the larger list.
- The list of errors in a result cannot be changed.
- Number comparisons use `BigDecimal`, so large and precise values are compared exactly.
- Field names are resolved once per method reference and cached.

### C.3 Limits

- A `Validator` is not thread-safe. Create one per validation and use it from one thread.
- Dates must have the shape `yyyy/MM/dd`. Other date types are not interpreted by the date rules.
- The `Patterns` check shape only. They do not verify checksums (national code, card number) or that an address exists.
- Persian calendar rules depend on the `jutil` module.

### C.4 Tuning

1. Keep the rules of one form in one method, next to the form class.
2. Use `Patterns` for formats and your own `regex` only for rules that are specific to your application.
3. Give every rule a `messageKey` if the application is translated.
4. Return codes and field names to the front end; let it decide how to show them.
5. Use `validateOrThrow()` with one shared handler in web applications.

## Appendix D. Message reference

**Default messages.** `<field>` is the field name, and the other placeholders are the values given to the rule.

| Code | Default message |
|------|-----------------|
| `REQUIRED` | `<field> is required` |
| `NOT_BLANK` | `<field> must not be blank` |
| `LENGTH` | `<field> must have length <n>` |
| `MIN_LENGTH` | `<field> must be at least <n> characters` |
| `MAX_LENGTH` | `<field> must be at most <n> characters` |
| `LENGTH_BETWEEN` | `<field> length must be between <min> and <max>` |
| `REGEX` | `<field> has an invalid format` (with a `Pattern`); `<field> does not match pattern <pattern>` (with a string) |
| `INVALID_PATTERN` | `<field>: pattern must not be null`; `<field>: '<pattern>' is not a valid regular expression (<description>)` |
| `CONTAINS` | `<field> must contain <x>` |
| `STARTS_WITH` | `<field> must start with <x>` |
| `ENDS_WITH` | `<field> must end with <x>` |
| `NUMBER` | `<field> must be a valid number` |
| `MIN` | `<field> must be >= <x>` |
| `MAX` | `<field> must be <= <x>` |
| `POSITIVE` | `<field> must be positive` |
| `BETWEEN` | `<field> must be between <min> and <max>` |
| `DIGITS` | `<field> must have at most <i> integer digit(s) and <f> fraction digit(s)` |
| `BOOLEAN` | `<field> must be true or false` |
| `IS_TRUE` | `<field> must be true` |
| `IS_FALSE` | `<field> must be false` |
| `DATE` | `<field> must be a valid date (yyyy/MM/dd)` |
| `PERSIAN_DATE` | `<field> must be a valid Persian date (yyyy/MM/dd)` |
| `DATE_CONTEXT_MISSING` | `<field>: date()/persianDate() must produce a valid date before this rule applies` |
| `INVALID_COMPARISON_DATE` | `<field>: <label> ('<date>') is not a valid Gregorian date (yyyy/MM/dd)` (or `Persian`) |
| `PAST` | `<field> must be in the past` |
| `FUTURE` | `<field> must be in the future` |
| `BEFORE` | `<field> must be before <date>` |
| `AFTER` | `<field> must be after <date>` |
| `DATE_BETWEEN` | `<field> must be between <start> and <end>` |
| `MINIMUM_AGE` | `<field> must represent an age of at least <n>` |
| `FUTURE_BIRTH_DATE` | `<field> must not be a date in the future` |
| `NOT_EMPTY` | `<field> must not be empty` |
| `SIZE` | `<field> must have size <n>` |
| `MIN_SIZE` | `<field> must have at least <n> elements` |
| `MAX_SIZE` | `<field> must have at most <n> elements` |
| `UNIQUE` | `<field> must contain unique elements` |
| `EQUAL_TO` | `<field> must be equal to the compared field`; `<field>: failed to read compared field (<exception>)` |
| `FIELD_ACCESS_ERROR` | `<field>: failed to evaluate field (<exception>)` |
| your own code | the message that you pass to `withMethod`; `(rule threw <exception>)` is appended when the rule throws |

Numbers given to `min`, `max` and `between` are written as decimals in the message (`must be >= 100.0`).

**Exception message.** `ValidationException`: `Validation failed with <n> error(s): [Error{field='...', code='...', message='...'}]`. The invalid value is not part of the text.
