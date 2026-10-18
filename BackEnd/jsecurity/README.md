# jsecurity

*J2OS Platform 2.4*

Almost every application has to answer the same questions. Who is calling? Is the caller allowed to do this? Is the text that arrived safe to show to other people? Is the uploaded file really a picture, and is it free of viruses? How are passwords and secrets stored so that a stolen database is not a disaster?

**jsecurity** (`org.j2os.platform.jsecurity`) answers these questions with small, direct tools: hashing and encryption, signed tokens, a request filter that checks them, rules about which fields a client may send and see, text cleaning, safe file upload, and sign-in with Google.

```java
String hash = Cryptography.hashByArgon2(password);                  // store this, never the password
String token = tokens.createToken("alice", Map.of("roles", List.of("USER")), 60L);
Profile safe = RequestAccessControl.apply("shop", fromClient, stored, "update");
String text = XssProtector.toRichText(comment);
FileSaveResult saved = uploads.save(stream, fileName, sizeBytes);
```

The module has eight parts:

- `Cryptography` offers hashes, password hashing, encryption for text and files, RSA keys and random values. All its methods are `static`.
- `TokenProcessor` creates, reads, changes and renews *JWT* tokens, the signed tokens that prove who the caller is.
- `AuthorizationFilter` checks the token on every request and decides, from the path and the caller's roles, whether the request may continue.
- `RequestAccessControl` removes the fields a client may not set and restores their stored values.
- `ResponseAccessControl` removes or blanks the fields a client may not see.
- `XssProtector` cleans text so that it cannot run as a script in someone's browser.
- `FileSecurity` accepts an uploaded file only if its size, real type, image content and virus scan are acceptable.
- `GoogleOAuthClient` lets a user sign in with a Google account.

Roughly, the guide goes like this: each chapter has a short program and the output it prints. The parts that only make sense inside a web application are shown as Spring snippets. The appendices are meant for quick lookups.

## Contents

1. [Hashes and Base64](#1-hashes-and-base64)
2. [Storing passwords](#2-storing-passwords)
3. [Encrypting text](#3-encrypting-text)
4. [RSA keys](#4-rsa-keys)
5. [Encrypting files](#5-encrypting-files)
6. [Random values](#6-random-values)
7. [Tokens](#7-tokens)
8. [Changing and renewing tokens](#8-changing-and-renewing-tokens)
9. [The authorization filter](#9-the-authorization-filter)
10. [How a request is judged](#10-how-a-request-is-judged)
11. [Paths: matching and safety](#11-paths-matching-and-safety)
12. [Limiting what a client may send](#12-limiting-what-a-client-may-send)
13. [Limiting what a client may see](#13-limiting-what-a-client-may-see)
14. [Cleaning text](#14-cleaning-text)
15. [Uploading files safely](#15-uploading-files-safely)
16. [Sign in with Google](#16-sign-in-with-google)
17. [The final example: accounts for a small shop](#17-the-final-example-accounts-for-a-small-shop)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The module needs these libraries on the classpath: BouncyCastle (hashing and encryption), the `java-jwt` library from Auth0 (tokens), the Jakarta Servlet API (the filter), Apache Commons Lang and SLF4J (field control), the OWASP Java HTML Sanitizer and Apache Commons Text (text cleaning), Apache Tika (file type detection) and Feign with the J2OS `jclient` module (Google). `ResponseAccessControl` uses the J2OS `page2` module. A program needs only the libraries of the parts it uses. The upload programs in chapter 15 also need an antivirus scanner installed on the machine.

---

## 1. Hashes and Base64

A *hash* is a fixed-length fingerprint of some data. The same input always gives the same fingerprint, a different input gives a different one, and the original cannot be recovered from the fingerprint. Hashes are used to compare data without keeping it, for example to check that a downloaded file was not changed.

*Base64* is something else: a way to write arbitrary bytes as ordinary letters and digits so that they can travel in text, such as a URL or a JSON document. It is not secret. Anyone can turn it back into the bytes.

```java
import org.j2os.platform.jsecurity.cryptography.Cryptography;

import java.nio.file.Files;
import java.nio.file.Path;

public class HashBasics {

    public static void main(String[] args) throws Exception {
        String text = "hello";

        String encoded = Cryptography.encodeBase64(text.getBytes());
        System.out.println("base64      : " + encoded);
        System.out.println("decoded     : " + new String(Cryptography.decodeBase64(encoded)));
        System.out.println("url-safe    : " + Cryptography.encodeBase64URL(new byte[]{(byte) 0xfb, (byte) 0xff, (byte) 0xfe}));

        System.out.println("sha2-256    : " + Cryptography.hashBySHA2_256(text));
        System.out.println("sha3-256    : " + Cryptography.hashBySHA3_256(text));
        System.out.println("sha2-512 len: " + Cryptography.hashBySHA2_512(text).length());

        Path file = Files.createTempFile("report", ".txt");
        Files.writeString(file, text);
        System.out.println("file sha2-256 equals text hash: "
                + Cryptography.hashFileBySHA2_256(file.toString()).equals(Cryptography.hashBySHA2_256(text)));

        try {
            Cryptography.encodeBase64((byte[]) null);
        } catch (IllegalArgumentException e) {
            System.out.println(e.getMessage());
        }
    }
}
```

```text
base64      : aGVsbG8=
decoded     : hello
url-safe    : -__-
sha2-256    : 2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824
sha3-256    : 3338be694f50c5f338814986cdf0686453a888b84f424d792af4b9202398f392
sha2-512 len: 128
file sha2-256 equals text hash: true
data must not be null
```

- `encodeBase64` and `decodeBase64` convert between bytes and text. The `URL` variants use a form that is safe inside a web address, with `-` and `_` instead of `+` and `/`. In the output, three bytes become `-__-`.
- `hashBySHA2_256`, `hashBySHA2_512`, `hashBySHA3_256` and `hashBySHA3_512` return the fingerprint of the UTF-8 bytes of a text as lowercase hexadecimal. SHA-256 gives 64 hexadecimal characters, SHA-512 gives 128.
- `hashByMD5` also exists. MD5 is broken and must not be used for anything that protects data. It is there only to meet old formats that require it.
- `hashFileBySHA2_256`, `hashFileBySHA2_512` and `hashFileByMD5` take the path of a file and read it as a stream, so a large file does not have to fit in memory. The output shows that the hash of the file equals the hash of the same text.
- A `null` argument is rejected with `IllegalArgumentException`, and the message names the parameter.

Do not use these plain hashes for passwords. They are made to be fast, and a fast hash lets an attacker try billions of guesses per second. Chapter 2 shows the right tools.

## 2. Storing passwords

A password must never be stored. Instead the application stores a *password hash* and compares what a user types against it. A password hash is deliberately slow and *salted*: a random value, the *salt*, is mixed in so that two users with the same password get different hashes, and so that tables of precomputed answers are useless.

```java
import org.j2os.platform.jsecurity.cryptography.Cryptography;

public class PasswordHashing {

    public static void main(String[] args) {
        String password = "correct horse battery";

        String bcrypt = Cryptography.hashByBCrypt(password);
        System.out.println("bcrypt  : " + bcrypt.substring(0, 7) + "... (" + bcrypt.length() + " chars)");
        System.out.println("  right : " + Cryptography.checkByBCrypt(password, bcrypt));
        System.out.println("  wrong : " + Cryptography.checkByBCrypt("guess", bcrypt));

        String argon2 = Cryptography.hashByArgon2(password);
        System.out.println("argon2  : " + argon2.substring(0, 31) + "...");
        System.out.println("  right : " + Cryptography.checkByArgon2(password, argon2));

        String pbkdf2 = Cryptography.hashByPBKDF2(password);
        System.out.println("pbkdf2  : " + pbkdf2.substring(0, 24) + "...");
        System.out.println("  right : " + Cryptography.checkByPBKDF2(password, pbkdf2));

        System.out.println("same password twice gives the same hash: "
                + Cryptography.hashByArgon2(password).equals(argon2));

        try {
            Cryptography.checkByArgon2(password, "not-a-hash");
        } catch (IllegalArgumentException e) {
            System.out.println(e.getMessage());
        }
    }
}
```

```text
bcrypt  : $2y$12$... (60 chars)
  right : true
  wrong : false
argon2  : $argon2id$v=19$m=65536,t=3,p=1$...
  right : true
pbkdf2  : $pbkdf2-sha256$i=210000$...
  right : true
same password twice gives the same hash: false
Invalid Argon2 hash format
```

jsecurity offers three algorithms. Each has a `hashBy...` method that returns a text containing everything needed to check the password later, and a `checkBy...` method that takes the typed password and the stored text and returns `true` or `false`.

| Algorithm | Methods | What the stored text looks like |
|-----------|---------|---------------------------------|
| Argon2id | `hashByArgon2`, `checkByArgon2` | `$argon2id$v=19$m=65536,t=3,p=1$<salt>$<hash>` |
| BCrypt | `hashByBCrypt`, `checkByBCrypt` | `$2y$12$...`, 60 characters |
| PBKDF2 | `hashByPBKDF2`, `checkByPBKDF2` | `$pbkdf2-sha256$i=210000$<salt>$<hash>` |

- **Argon2id** is the best default. It is slow and also needs a lot of memory (64 MB per hash), which makes attacks with special hardware expensive. The stored text records the settings, so a hash made today can still be checked after the defaults change.
- **BCrypt** uses cost 12. The password is first reduced to a 43-character fingerprint with SHA-256, because BCrypt silently ignores everything after 72 bytes. For this reason the hashes are **not** interchangeable with BCrypt hashes made by other software.
- **PBKDF2** uses HMAC-SHA-256 with 210000 rounds. It is the choice when a standard approved by a regulation is required.
- Hashing the same password twice gives different texts, because the salt is random. The output shows `false` for that comparison. Always use `check...` to compare.
- Inside `checkByArgon2` and `checkByPBKDF2` the two hashes are compared in a way that takes the same time wherever they differ, so the time of the check gives no hint about the hash.
- `checkByArgon2` and `checkByPBKDF2` read the settings from the stored text. A stored text with absurd settings, which would let someone force the server to spend minutes or gigabytes on one check, is refused. The limits are in Appendix C.
- A stored text that is not in the expected format gives `IllegalArgumentException`, for example `Invalid Argon2 hash format`.

## 3. Encrypting text

*Encryption* turns readable data into unreadable data that only the holder of a *key* can turn back. The module uses AES in GCM mode, which also detects any change to the encrypted data: if someone alters a single character, decryption fails instead of returning wrong text.

```java
import org.j2os.platform.jsecurity.cryptography.Cryptography;

import javax.crypto.SecretKey;

public class AesText {

    public static void main(String[] args) {
        SecretKey key = Cryptography.generateAESKey(256);
        String secret = "card ending 4242";

        String sealed = Cryptography.encryptStringByAES(secret, key);
        System.out.println("sealed      : " + sealed.length() + " characters of Base64");
        System.out.println("opened      : " + Cryptography.decryptStringByAES(sealed, key));
        System.out.println("sealed again is different: "
                + !Cryptography.encryptStringByAES(secret, key).equals(sealed));

        String withPassword = Cryptography.encryptStringByAES(secret, "a long passphrase");
        System.out.println("by password : " + Cryptography.decryptStringByAES(withPassword, "a long passphrase"));

        try {
            Cryptography.decryptStringByAES(sealed, Cryptography.generateAESKey(256));
        } catch (IllegalStateException e) {
            System.out.println(e.getMessage());
        }
        try {
            Cryptography.decryptStringByAES("AAAA", key);
        } catch (IllegalArgumentException e) {
            System.out.println(e.getMessage());
        }
        try {
            Cryptography.generateAESKey(100);
        } catch (IllegalArgumentException e) {
            System.out.println(e.getMessage());
        }
    }
}
```

```text
sealed      : 60 characters of Base64
opened      : card ending 4242
sealed again is different: true
by password : card ending 4242
AES decryption failed - wrong key/password or corrupted data
Invalid AES payload: too short to be valid
AES key size must be 128, 192, or 256 bits
```

- `generateAESKey(bits)` makes a random key of 128, 192 or 256 bits. Any other size is refused.
- `encryptStringByAES(text, key)` returns Base64 text that contains a random starting value (the *IV*, 12 bytes) followed by the encrypted data. Encrypting the same text twice gives different results, as the output shows.
- `decryptStringByAES(sealed, key)` returns the original text.
- With a **password** instead of a key, the module derives a key from the password with PBKDF2 and puts a random salt in front of the result. Use this form when the secret is something a person types.
- A wrong key, a wrong password and damaged data all give the same `IllegalStateException`, with the message `AES decryption failed - wrong key/password or corrupted data`. This is on purpose: the message does not say which of them it was.
- A text that is too short to be valid gives `IllegalArgumentException`.

The key is the secret. Keep it outside the code, for example in a secrets store or an environment variable, and keep a copy somewhere safe: data encrypted with a lost key is lost.

## 4. RSA keys

RSA works with a *pair* of keys. What is encrypted with the public key can only be decrypted with the private key. The public key can therefore be given to anyone.

```java
import org.j2os.platform.jsecurity.cryptography.Cryptography;

import java.security.KeyPair;

public class RsaKeys {

    public static void main(String[] args) {
        KeyPair pair = Cryptography.generateRSAKeys(2048);

        String publicText = Cryptography.encodeRSAPublicKey(pair.getPublic());
        String privateText = Cryptography.encodeRSAPrivateKey(pair.getPrivate());
        System.out.println("public key text : " + publicText.length() + " characters");
        System.out.println("private key text: " + privateText.length() + " characters");

        byte[] sealed = Cryptography.encryptStringByRSA("session-key-123",
                Cryptography.decodeRSAPublicKey(publicText));
        System.out.println("sealed bytes    : " + sealed.length);
        System.out.println("opened          : " + Cryptography.decryptStringByRSA(sealed,
                Cryptography.decodeRSAPrivateKey(privateText)));

        try {
            Cryptography.encryptStringByRSA("x".repeat(500), pair.getPublic());
        } catch (IllegalArgumentException e) {
            System.out.println(e.getMessage());
        }
        try {
            Cryptography.generateRSAKeys(1024);
        } catch (IllegalArgumentException e) {
            System.out.println(e.getMessage());
        }
    }
}
```

```text
public key text : 392 characters
private key text: 1624 characters
sealed bytes    : 256
opened          : session-key-123
Plain text is too large for RSA encryption with this key size. RSA is only suitable for small payloads (e.g. an AES key) — use AES for larger data.
RSA key size must be at least 2048 bits for adequate security.
```

- `generateRSAKeys(size)` needs at least 2048 bits.
- `encodeRSAPublicKey` and `encodeRSAPrivateKey` turn a key into Base64 text for storage or transport. `decodeRSAPublicKey` and `decodeRSAPrivateKey` turn it back. The public key is in the X.509 format and the private key in PKCS#8, without a password, so the private text must be protected like any secret.
- `encryptStringByRSA` returns raw bytes. `decryptStringByRSA` takes those bytes and the private key.
- RSA can only encrypt a small amount of data, a little less than the key size. It is meant for things such as an AES key. For anything larger, encrypt the data with AES and send the AES key with RSA. The output shows the error for a text of 500 characters.
- The module has no digital signatures. Signed tokens are covered in chapter 7.

## 5. Encrypting files

`encryptFileByAES` and `decryptFileByAES` work on a file **in place**: after the call, the file at the same path holds the encrypted (or decrypted) content.

```java
import org.j2os.platform.jsecurity.cryptography.Cryptography;

import javax.crypto.SecretKey;
import java.nio.file.Files;
import java.nio.file.Path;

public class AesFile {

    public static void main(String[] args) throws Exception {
        Path file = Files.createTempFile("statement", ".csv");
        Files.writeString(file, "id,amount\n1,250\n2,990\n");
        String path = file.toString();
        long plainSize = Files.size(file);

        SecretKey key = Cryptography.generateAESKey(256);
        Cryptography.encryptFileByAES(path, key);
        System.out.println("size grew by " + (Files.size(file) - plainSize) + " bytes");
        System.out.println("starts with the text? " + Files.readString(file, java.nio.charset.StandardCharsets.ISO_8859_1).startsWith("id,"));

        try {
            Cryptography.decryptFileByAES(path, Cryptography.generateAESKey(256));
        } catch (IllegalStateException e) {
            System.out.println(e.getMessage().replace(path, "<file>"));
        }
        try {
            Cryptography.decryptFileByAES(path, "some password");
        } catch (IllegalArgumentException e) {
            System.out.println(e.getMessage());
        }

        Cryptography.decryptFileByAES(path, key);
        System.out.print(Files.readString(file));

        Cryptography.encryptFileByAES(path, "a long passphrase");
        try {
            Cryptography.decryptFileByAES(path, key);
        } catch (IllegalArgumentException e) {
            System.out.println(e.getMessage());
        }
        Cryptography.decryptFileByAES(path, "a long passphrase");
        System.out.println("back to " + Files.size(file) + " bytes");
    }
}
```

```text
size grew by 29 bytes
starts with the text? false
AES file decryption failed (wrong key/password or corrupted file): <file>
File was encrypted with a SecretKey; that same key is required to decrypt it.
id,amount
1,250
2,990
File was encrypted with a password; a password is required to decrypt it.
back to 22 bytes
```

- The file grows by 29 bytes: one byte that says whether a key or a password was used, the 12-byte IV and the 16-byte check value that detects changes. With a password, 16 bytes of salt are added as well.
- Each method has two forms, one with a `SecretKey` and one with a password.
- The file is read and written as a stream, so its size does not matter for memory.
- The new content is first written to a temporary file next to the original, and the original is replaced only when that succeeded. When decrypting, the original is replaced only after the content has been verified. A failure in the middle leaves the original untouched and removes the temporary file.
- Opening a file with the wrong kind of secret gives a clear message, as the output shows: a file made with a key asks for that key, a file made with a password asks for a password.
- The optional last argument `durable` (`true`) also forces the data to disk before the replacement, for the case that a power failure must not lose the file. It is slower.

## 6. Random values

For anything that must not be guessable, such as codes, tokens and identifiers, a normal `Random` is not enough. The `Cryptography` helpers use `SecureRandom`.

```java
import org.j2os.platform.jsecurity.cryptography.Cryptography;

public class RandomValues {

    public static void main(String[] args) {
        System.out.println("uuid length      : " + Cryptography.uuid().length());
        System.out.println("code             : " + Cryptography.randomAlphaNumericString(8).length() + " characters");
        System.out.println("letters only     : " + Cryptography.randomAlphaString(6).matches("[A-Za-z]{6}"));

        boolean inRange = true;
        for (int i = 0; i < 1000; i++) {
            int n = Cryptography.randomInt(10, 20);
            inRange &= n >= 10 && n < 20;
        }
        System.out.println("randomInt(10,20) stayed in [10, 20): " + inRange);

        try {
            Cryptography.randomInt(5, 5);
        } catch (IllegalArgumentException e) {
            System.out.println(e.getMessage());
        }
        try {
            Cryptography.randomAlphaString(0);
        } catch (IllegalArgumentException e) {
            System.out.println(e.getMessage());
        }
    }
}
```

```text
uuid length      : 36
code             : 8 characters
letters only     : true
randomInt(10,20) stayed in [10, 20): true
minimum must be less than maximum
length must be positive
```

| Method | Result |
|--------|--------|
| `uuid()` | a random identifier such as `3f2b8c1e-...`, 36 characters |
| `randomInt(max)` | an integer from 0 up to but not including `max` |
| `randomInt(min, max)` | an integer from `min` up to but not including `max` |
| `randomDouble()`, `randomDouble(min, max)` | a decimal number in the range |
| `randomBoolean()` | `true` or `false` |
| `randomAlphaString(length)` | letters only |
| `randomAlphaNumericString(length)` | letters and digits |

A range that is empty or backwards, and a length of zero or less, are refused with `IllegalArgumentException`.

## 7. Tokens

When a user signs in, the server must recognize the same user on the next request, and the next hundred. The usual answer is a *token*: a piece of text that the server gives to the client after sign-in, and that the client sends back with every request. jsecurity uses *JWT* (JSON Web Token). A JWT contains a few facts about the user, called *claims*, and a *signature* made with a secret that only the server knows. If anyone changes the token, the signature no longer fits and the server rejects it.

```java
import org.j2os.platform.jsecurity.token.TokenProcessor;

import java.util.List;
import java.util.Map;

public class TokenBasics {

    public static void main(String[] args) {
        String secret = "change-me-this-secret-must-be-at-least-sixty-four-bytes-long-0123456789";
        TokenProcessor tokens = new TokenProcessor("shop-api", secret);

        String token = tokens.createToken("alice", Map.of(
                "roles", List.of("USER", "EDITOR"),
                "department", "sales"), 30L);

        System.out.println("three parts     : " + token.split("\\.").length);
        System.out.println("subject         : " + tokens.getSubject(token));
        System.out.println("department      : " + tokens.getClaimAsString(token, "department"));
        System.out.println("roles           : " + tokens.getClaimAsList(token, "roles"));
        System.out.println("expired         : " + tokens.isTokenExpired(token));
        System.out.println("valid           : " + tokens.isTokenValid(token));

        String forged = token.substring(0, token.length() - 4) + "AAAA";
        try {
            tokens.getSubject(forged);
        } catch (RuntimeException e) {
            System.out.println("forged token    : " + e.getClass().getSimpleName());
        }

        TokenProcessor other = new TokenProcessor("other-api", secret);
        try {
            other.getSubject(token);
        } catch (RuntimeException e) {
            System.out.println("other issuer    : " + e.getClass().getSimpleName());
        }

        try {
            new TokenProcessor("shop-api", "too short");
        } catch (IllegalArgumentException e) {
            System.out.println(e.getMessage());
        }
        try {
            tokens.createToken("bob", Map.of("level", 3), 5L);
        } catch (IllegalArgumentException e) {
            System.out.println(e.getMessage());
        }
        try {
            tokens.createToken("bob", Map.of("exp", "1"), 5L);
        } catch (IllegalArgumentException e) {
            System.out.println(e.getMessage());
        }
    }
}
```

```text
three parts     : 3
subject         : alice
department      : sales
roles           : [USER, EDITOR]
expired         : false
valid           : true
forged token    : SignatureVerificationException
other issuer    : InvalidClaimException
HMAC512 secret key must contain at least 64 UTF-8 bytes
Unsupported claim value type: java.lang.Integer
Registered JWT claim cannot be used: exp
```

- `new TokenProcessor(issuer, secretKey)` makes the object. The *issuer* is a name for your service, written into every token and required when the token is read. The secret must be at least 64 bytes long, and a blank issuer or secret is refused.
- `createToken(subject, claims, minutes)` makes a token. The *subject* is the user. The claims are extra facts. The last argument is the lifetime in minutes. `null` or `0` means that the token never expires, and a negative number is refused.
- A claim value must be a `String`, a `List<String>` or a `Map` with `String` keys and `String` or `List<String>` values. Anything else, such as the `Integer` in the example, is refused. Write numbers as text.
- The names `iss`, `sub`, `aud`, `exp`, `nbf`, `iat` and `jti` belong to the JWT standard. The module sets some of them itself, so a claim with one of these names is refused.
- `getSubject`, `getClaimAsString`, `getClaimAsList` and `getClaimAsMap` read a token. They first check the signature, the issuer and the expiry, and they throw if any of them fails.
- `isTokenValid` and `isTokenExpired` answer with `true` or `false`. A token that is only expired is a normal answer (`isTokenExpired` is `true`, `isTokenValid` is `false`). A token that is forged or has a different issuer is not: the exception passes through, as the `forged token` and `other issuer` lines show.
- The exceptions are the ones of the Auth0 library: `TokenExpiredException`, `SignatureVerificationException`, `InvalidClaimException`, `AlgorithmMismatchException` and `JWTDecodeException`. All of them extend `JWTVerificationException`.

A token is signed, not encrypted. Anyone who holds it can read its claims. Never put a password or other secret into a claim.

## 8. Changing and renewing tokens

A token cannot be edited, because the signature would break. `TokenProcessor` offers methods that read a token, change it and sign a new one.

```java
import org.j2os.platform.jsecurity.token.TokenProcessor;

import java.util.Map;

public class TokenChange {

    public static void main(String[] args) throws Exception {
        String secret = "change-me-this-secret-must-be-at-least-sixty-four-bytes-long-0123456789";
        TokenProcessor tokens = new TokenProcessor("shop-api", secret);

        String token = tokens.createToken("alice", Map.of("plan", "free"), 30L);
        Thread.sleep(1100);

        String upgraded = tokens.setClaim(token, "plan", "pro");
        System.out.println("plan after setClaim : " + tokens.getClaimAsString(upgraded, "plan"));

        String trimmed = tokens.removeClaim(upgraded, "plan");
        System.out.println("plan after remove   : " + tokens.getClaimAsString(trimmed, "plan"));

        String renewed = tokens.renewToken(token);
        System.out.println("renewed is a new token: " + !renewed.equals(token));
        System.out.println("renewed keeps claims  : " + tokens.getClaimAsString(renewed, "plan"));

        String forever = tokens.createToken("service", Map.of(), null);
        System.out.println("no expiry stays so    : " + tokens.isTokenExpired(tokens.renewToken(forever)));

        try {
            tokens.createToken("alice", Map.of(), -5L);
        } catch (IllegalArgumentException e) {
            System.out.println(e.getMessage());
        }
    }
}
```

```text
plan after setClaim : pro
plan after remove   : null
renewed is a new token: true
renewed keeps claims  : free
no expiry stays so    : false
Expiration duration cannot be negative
```

- `setClaim(token, name, value)` and `removeClaim(token, name)` return a new token. The subject, the issue time and the expiry time stay exactly as they were. Changing a claim does **not** extend the life of a token.
- `renewToken(token)` returns a new token with the same claims and a fresh issue time. The new lifetime equals the old one. A token that never expired keeps never expiring.
- `renewToken(token, minutes)` does the same with a new lifetime.
- Every one of these methods checks the old token first, and refuses an expired one. Each has a twin with `IgnoringExpiration` in the name. The twin accepts an expired token but still checks the signature, the algorithm and the issuer. Use it with care, for example for a "refresh within a grace period" feature.
- Only claims of the supported kinds (text, list of text, map) survive a rebuild. A token made by other software with, say, a number claim loses that claim when renewed.

## 9. The authorization filter

`AuthorizationFilter` stands in front of the application. For each request it finds the token, checks it, finds the caller's roles and compares them with the rules for the requested path. The request goes on only if everything fits. Otherwise the filter writes an error response and the application never sees the request.

It has two kinds of rules:

- **Open paths** need no token at all: the login page, the health check, static files.
- **Role rules** map a path pattern to the roles allowed there. A *role* is a label such as `ADMIN` or `USER`, kept in a claim of the token (by default the claim named `roles`).

Every path that is not open must match a role rule. A path that matches nothing is refused. This is a safe default: a new endpoint is closed until someone opens it.

The filter is configured once, usually when the application starts, and the settings are shared by the whole application.

```java
@Configuration
public class SecurityConfig {

    @Bean
    public FilterRegistrationBean<AuthorizationFilter> authorizationFilter(
            @Value("${app.jwt.secret}") String secret) {

        Map<String, Set<String>> rules = new LinkedHashMap<>();   // the order matters
        rules.put("/api/admin/**", Set.of("ADMIN"));
        rules.put("/api/orders/*", Set.of("USER", "ADMIN"));
        rules.put("/api/profile/**", Set.of("USER", "ADMIN"));

        AuthorizationFilter
                .issuer("shop-api")
                .secretKey(secret)
                .openPermits(List.of("/api/login", "/api/login/google/**", "/health", "/static/**"))
                .rolePermits(rules)
                .apply();

        FilterRegistrationBean<AuthorizationFilter> registration =
                new FilterRegistrationBean<>(new AuthorizationFilter());
        registration.addUrlPatterns("/*");
        return registration;
    }

    @Bean
    public TokenProcessor tokenProcessor(@Value("${app.jwt.secret}") String secret) {
        return new TokenProcessor("shop-api", secret);
    }
}
```

- The static methods `issuer(...)`, `secretKey(...)`, `openPermits(...)`, `rolePermits(...)` and the others each start a *builder*. Call more methods on the builder and finish with `apply()`.
- `apply()` merges the new settings into the current ones. Settings you do not mention keep their previous value, so a later call can change only one thing. The issuer and the secret are required for the first `apply()`; without them it throws `IllegalStateException`. A secret under 64 bytes is refused as in chapter 7.
- The roles map must be a `LinkedHashMap`. The filter tries the rules **in order and the first pattern that matches decides**. A map without a defined order, such as `Map.of`, gives an undefined result.
- A blank pattern, a `null` list and a rule without roles are refused at once.

The filter reads the token from the header `Authorization: Bearer <token>` by default. For an application with a browser front end the token can instead travel in a cookie; the other settings are in the following snippet.

```java
AuthorizationFilter
        .tokenSource(TokenSource.COOKIE)
        .cookieName("session")
        .roleClaimName("roles")
        .before((request, response) -> {
            // runs for every request; return false after writing the response yourself
            return true;
        })
        .after((request, response) -> {
            // runs only after the request was allowed
            return true;
        })
        .onError((request, response, error) -> {
            if (error instanceof MissingTokenException) {
                response.setStatus(401);
                response.setContentType("application/json; charset=UTF-8");
                response.getWriter().write("{\"message\":\"Please sign in\"}");
            } else {
                response.setStatus(403);
                response.setContentType("application/json; charset=UTF-8");
                response.getWriter().write("{\"message\":\"Not allowed\"}");
            }
        })
        .apply();
```

| Setting | Default | Meaning |
|---------|---------|---------|
| `roleClaimName` | `roles` | the claim that holds the list of roles |
| `tokenSource` | `AUTHORIZATION_HEADER_BEARER` | `COOKIE` reads the token from a cookie instead |
| `cookieName` | `token` | the cookie to read when `tokenSource` is `COOKIE` |
| `bypassPreflightOptions` | `true` | lets `OPTIONS` requests pass without a token |
| `before` | does nothing | a function that runs first, for every request |
| `after` | does nothing | a function that runs after the request is allowed |
| `onError` | JSON error with the right status | writes the response for a refused request |

*Preflight* requests are the `OPTIONS` requests that a browser sends before a cross-site call (CORS). They carry no token, so they are let through by default.

Instead of registering the filter, a servlet can call the check itself:

```java
@WebServlet("/api/reports/daily")
public class DailyReportServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response) throws IOException {
        if (!AuthorizationFilter.authorize(request, response)) {
            return;                       // the error response has already been written
        }
        String user = (String) request.getAttribute("authorization.subject");
        response.getWriter().write("report for " + user);
    }
}
```

`authorize` returns `true` when the request may continue and `false` when it was refused. In the second case the error response has already been written, so the servlet just returns.

On success the filter stores two attributes on the request. `authorization.subject` holds the user name from the token, and `authorization.roles` holds the roles as a `Set<String>`. A controller reads them with `request.getAttribute(...)`.

## 10. How a request is judged

For every request, the filter takes these steps in order:

1. It clears the two result attributes, so that nothing left over from earlier processing can be mistaken for a result.
2. It runs the `before` function. If that returns `false`, the filter stops.
3. If the method is `OPTIONS` and `bypassPreflightOptions` is on, the request passes.
4. It cleans the path (chapter 11).
5. If the path matches an open pattern, the request passes. Nothing is stored in the attributes and `after` does not run.
6. It reads the token and takes the user name and the roles from it.
7. It walks the role rules in order. The **first** rule whose pattern matches the path decides: the caller needs at least one of its roles.
8. It stores the attributes, runs `after` and lets the request go on. If `after` returns `false`, the filter stops and expects `after` to have written the answer.

A request that is stopped at any of these steps gets one of six errors. Each has a default status and a JSON body of the form `{"error":"<message>"}`.

| Error | Status | Cause |
|-------|--------|-------|
| `MissingTokenException` | 401 | no token in the expected place |
| `InvalidTokenException` | 401 | the token is damaged, forged, from another issuer, or the cookie was sent twice |
| `ExpiredTokenException` | 401 | the token has expired |
| `InsufficientRoleException` | 403 | the caller has none of the roles the rule needs |
| `UnmappedPathException` | 403 | no rule covers the path |
| `MalformedRequestPathException` | 400 | the path looks like an attack (chapter 11) |

Any other exception, for example one thrown by your own `before` function, gives status 500 with the fixed text `Internal server error`. The real message is hidden from the client on purpose.

The following table comes from real requests against a filter with the open paths `/`, `/health`, `/api/login` and `/static/**`, and the rules `/api/admin/**` for `ADMIN`, `/api/orders/*` for `USER` and `ADMIN`, and `/api/reports/**` for `ADMIN` and `AUDITOR`. The user token belongs to `alice` with the role `USER`, and the admin token to `root` with the role `ADMIN`.

| Request | Result |
|---------|--------|
| `GET /health` without a token | passes |
| `GET /api/orders/7` without a token | 401 `No Authorization header in the form "Bearer <token>" was found` |
| the same with the header `Basic abc` | 401, the same message |
| the same with the user token | passes; `authorization.subject` is `alice`, `authorization.roles` is `[USER]` |
| the same with the scheme written as `bearer` | passes (the word `Bearer` is not case sensitive) |
| the same with the last characters of the token changed | 401 `Token is invalid` |
| the same with a token from another issuer | 401 `Token is invalid` |
| the same with a token that has no `roles` claim | 403 `User does not have a sufficient role to access /api/orders/7` |
| `GET /api/admin/users` with the user token | 403 `User does not have a sufficient role to access /api/admin/users` |
| `GET /api/admin/users` with the admin token | passes |
| `GET /api/orders/7/items` with the user token | 403 `Path /api/orders/7/items is not covered by any configured access rule` |
| `OPTIONS /api/admin/users` without a token | passes |

A token that has expired gives 401 with `Token has expired`.

When the token comes from a cookie, a request without the cookie gives 401 `No cookie named "session" was found`, and a request that carries two cookies of that name gives 401 `Multiple cookies named "session" were supplied`. The second rule exists because some servers pick one of them silently, which an attacker could use.

The functions passed to `before`, `after` and `onError` may do anything, including writing the response themselves. If a function throws, the exception goes to `onError`. If `onError` itself throws and the response is not yet sent, the filter writes status 500 with `{"error":"Internal server error"}`, so the request is never allowed by accident.

Errors in the application itself are none of the filter's business. The filter calls the next step in the chain outside its own error handling, so an exception from a controller travels up unchanged to the usual error handling of the application.

If the filter is used before `apply()` has ever succeeded, every request fails with an `IllegalStateException` (wrapped in a `ServletException` when it runs as a filter). The message says `AuthorizationFilter is not configured yet`.

## 11. Paths: matching and safety

A rule's pattern is written in the *Ant style*, which is also used by Spring:

| Pattern | Matches | Does not match |
|---------|---------|----------------|
| `/a/b` | `/a/b` | `/a/b/`, `/a/b/c`, `/a/B` |
| `/a/b/*` | `/a/b/c` | `/a/b`, `/a/b/c/d` |
| `/a/b/**` | `/a/b`, `/a/b/`, `/a/b/c`, `/a/b/c/d` | `/a/bc` |
| `/a/**/z` | `/a/z`, `/a/x/z`, `/a/x/y/z` | `/a/zz` |
| `/img/*.png` | `/img/a.png` | `/img/x/a.png`, `/img/a.jpg` |
| `/f?le` | `/file` | `/fle`, `/fiile` |
| `/` | `/` | `/a` |
| `/**` | every path | |

- `*` stands for any characters inside one segment, never a `/`. `**` stands for any number of segments, including none. `?` stands for exactly one character.
- A pattern without a wildcard matches only itself. A trailing slash counts: `/a/b/` and `/a/b` are different.
- Matching is case sensitive, and it uses the whole `getRequestURI()`. **The context path of the application is not removed.** An application deployed under `/shop` must write `/shop/api/...` in its patterns.

Many attacks hide in the way a path is written. A client can write `/api/x/../admin`, or `%2e%2e`, or `;` parameters, hoping that the filter and the application read the path differently. The filter therefore *normalizes* the path before it compares: it writes the path in one canonical form, and refuses paths that cannot be put in that form safely.

| Path as received | Result |
|------------------|--------|
| `/api/orders/./7` | read as `/api/orders/7` |
| `/api/x/../orders/7` | read as `/api/orders/7` |
| `/api/orders/7;jsessionid=1` | read as `/api/orders/7` (the `;` parameters are removed) |
| `/api/orders/%37` | read as `/api/orders/7` (`%37` is the character `7`) |
| `/api/orders/7/` | stays as written; no rule matches, 403 |
| `/API/orders/7` | stays as written; no rule matches, 403 |
| `/api/orders%2F7` | 400 `Request path contains a disallowed encoded or suspicious sequence` |
| `/api/orders/%2e%2e/admin/users` | 400, the same message |
| `/api//orders/7` | 400, the same message |
| `/api/orders\7` | 400, the same message |
| `/api/orders/%zz` | 400 `Request path could not be decoded` |
| `/api/orders/7%00` | 400 `Request path contains a disallowed control character` |
| `/../etc` | 400 `Request path contains an invalid traversal sequence` |

An encoded slash (`%2f`), an encoded backslash (`%5c`) and an encoded dot (`%2e`), written in any letter case, are refused. So are a backslash, a doubled slash, a control character, a broken percent sequence and bytes that are not valid UTF-8. The filter does not treat `%3B` as a `;`.

## 12. Limiting what a client may send

A client sends a JSON body with a user profile. The body contains a field `loyaltyPoints`. Nothing stops a malicious client from sending `1000000` there. The problem is called *mass assignment*: the framework copies every field of the request into the object, including those that the client must never set.

`RequestAccessControl` solves it with a list of rules. For a *scope* (a name for an area of the application), a class and an *action* (a name such as `update`), you register the fields the client may **not** set. The method `apply` then returns a copy of the object in which those fields are `null`, or restored from the stored object.

```java
import org.j2os.platform.jsecurity.access.RequestAccessControl;

public class FieldLimits {

    public static void main(String[] args) {
        RequestAccessControl.registerFieldLimitation("staff", Employee.class, "salary", "update");
        RequestAccessControl.registerFieldLimitation("staff", Employee.class, "address.street", "update");
        RequestAccessControl.registerActionDenial("staff", Employee.class, "delete");

        Address address = new Address();
        address.city = "Oslo";
        address.street = "Karl Johans gate 1";
        Employee fromClient = new Employee(7L, "Alice Berg", 999999, address);

        Employee cleaned = RequestAccessControl.apply("staff", fromClient, "update");
        System.out.println("received : " + fromClient + ", salary field: " + fromClient.salary);
        System.out.println("cleaned  : " + cleaned + ", street: " + cleaned.address.street);

        Employee stored = new Employee(7L, "Alice B.", 52000, null);
        Employee merged = RequestAccessControl.apply("staff", fromClient, stored, "update");
        System.out.println("merged   : " + merged);

        System.out.println("other action untouched: " + RequestAccessControl.apply("staff", fromClient, "create").salary);

        try {
            RequestAccessControl.apply("staff", fromClient, "delete");
        } catch (RequestAccessControl.DeniedException e) {
            System.out.println("denied   : " + e.getMessage());
        }
        try {
            RequestAccessControl.registerFieldLimitation("staff", Employee.class, "nickname", "update");
        } catch (IllegalArgumentException e) {
            System.out.println(e.getMessage());
        }
    }
}

class Employee {
    Long id;
    String name;
    Integer salary;
    Address address;

    Employee() {
    }

    Employee(Long id, String name, Integer salary, Address address) {
        this.id = id;
        this.name = name;
        this.salary = salary;
        this.address = address;
    }

    public String toString() {
        return "Employee{id=" + id + ", name=" + name + ", salary=" + salary + ", city=" + (address == null ? null : address.city) + "}";
    }
}

class Address {
    String city;
    String street;

    Address() {
    }
}
```

```text
received : Employee{id=7, name=Alice Berg, salary=999999, city=Oslo}, salary field: 999999
cleaned  : Employee{id=7, name=Alice Berg, salary=null, city=Oslo}, street: null
merged   : Employee{id=7, name=Alice Berg, salary=52000, city=Oslo}
other action untouched: 999999
denied   : delete
Field 'nickname' not found on Employee while validating path 'nickname'
```

- `registerFieldLimitation(scope, Class, field, action)` forbids one field for one action. The field can be a dotted path such as `address.street`. The class is checked at once: a field that does not exist is refused with `IllegalArgumentException`, as the last line of the output shows.
- `apply(scope, target, action)` returns a **copy** with the forbidden fields set to `null`. The original object is not changed. The `cleaned` line shows `salary=null` while the `received` object still holds the value.
- `apply(scope, target, oldTarget, action)` also takes the object that is stored now, and puts the stored values back into the forbidden fields. This is the form to use for updates: the result can be saved as it is. In the output, `salary` returns to `52000`.
- A rule belongs to one action. Other actions are not touched (`other action untouched`).
- `registerActionDenial(scope, Class, action)` forbids a whole action. `apply` then throws `RequestAccessControl.DeniedException`, and its message is the name of the action. The check is made before any copying. The same rule in code form:

```java
RequestAccessControl.registerActionDenial("shop", Order.class, "delete");

// later, in the code that handles the request
Order order = RequestAccessControl.apply("shop", fromClient, "delete");   // throws DeniedException
```

- `unregisterFieldLimitation` and `unregisterActionDenial` remove a rule again.
- The registrations are kept for the whole JVM. Register them once, when the application starts.

The classes you use must follow a few rules. Each needs a no-argument constructor (it may be non-public). A restricted field must have a *wrapper* type such as `Integer`, not `int`, because a primitive cannot be set to `null`. Records are not supported. Nested objects on a restricted path are copied, other nested objects are shared with the original.

There is a second form of `registerFieldLimitation` that takes the class name as text. It only logs a warning when the class cannot be found or the path is wrong, so that a module can be configured before all its classes are loaded.

## 13. Limiting what a client may see

The same idea applies to responses. A list of customers goes to a screen where the support staff must not see the phone number, while the accounting staff may. `ResponseAccessControl` removes or blanks fields on the way out.

```java
import org.j2os.platform.jsecurity.access.ResponseAccessControl;

import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.TreeMap;

public class HideColumns {

    public static class Customer {
        public String name = "Alice Berg";
        public String phone = "555-0100";
        public Map<String, Object> card = new LinkedHashMap<>(Map.of("last4", "4242", "brand", "visa"));
    }

    public static void main(String[] args) {
        List<Map<String, Object>> rows = new ArrayList<>();
        rows.add(new LinkedHashMap<>(Map.of("name", "Alice Berg", "phone", "555-0100", "note", "vip")));
        rows.add(new LinkedHashMap<>(Map.of("name", "Omar Haddad", "phone", "555-0101", "note", "new")));

        List<Map<String, Object>> removed = ResponseAccessControl.apply(rows, List.of("phone"), "REMOVE");
        System.out.println("removed : " + new TreeMap<>(removed.get(0)));

        List<Map<String, Object>> emptied = ResponseAccessControl.apply(rows, List.of("phone"), ResponseAccessControl.EMPTY);
        System.out.println("emptied : " + new TreeMap<>(emptied.get(0)));

        List<Map<String, Object>> anyWord = ResponseAccessControl.apply(rows, List.of("phone"), "hide");
        System.out.println("any word: " + new TreeMap<>(anyWord.get(1)));

        Map<String, Object> page = new LinkedHashMap<>();
        page.put("rows", new ArrayList<>(List.of(new LinkedHashMap<>(Map.of("name", "Alice Berg", "note", "vip")))));
        page.put("total", 1);
        Map<String, Object> filtered = ResponseAccessControl.apply(page, List.of("note"), "REMOVE");
        System.out.println("page    : total=" + filtered.get("total") + ", rows=" + filtered.get("rows"));

        Map<String, Object> one = ResponseAccessControl.apply(new Customer(), List.of("phone", "card.last4"), "REMOVE");
        System.out.println("object  : name=" + one.get("name") + ", phone=" + one.containsKey("phone")
                + ", card=" + new TreeMap<>((Map<String, Object>) one.get("card")));
    }
}
```

```text
removed : {name=Alice Berg, note=vip}
emptied : {name=Alice Berg, note=vip, phone=}
any word: {name=Omar Haddad, note=new}
page    : total=1, rows=[{name=Alice Berg}]
object  : name=Alice Berg, phone=false, card={brand=visa}
```

- `apply(rows, restrictedFields, action)` works on a list of rows, on a single object, or on a result map that has a `rows` entry (the shape that `page2` returns). The other entries of the map, such as `total`, stay as they were.
- The action `EMPTY` replaces the value of the field with an empty text, so the column still exists. **Any other** text removes the field. `REMOVE` is a constant that documents this, but any word has the same effect.
- A field name can be a dotted path such as `card.last4`.
- Rows are converted to `Map<String, Object>`. The result holds maps, not the original class, so it is meant to be sent to the client, not used further as an entity.
- A `null` input gives `null`.

## 14. Cleaning text

*Cross-site scripting* (XSS) happens when text written by one user is shown on a page and the browser treats part of it as code. A comment that contains `<script>` is the classic case. The remedy is to clean the text, and `XssProtector` has three ways, for three different purposes.

```java
import org.j2os.platform.jsecurity.protection.XssProtector;

public class SafeText {

    public static void main(String[] args) {
        String comment = "<b>Hello</b><script>alert(1)</script>";

        System.out.println("rich text   : " + XssProtector.toRichText(comment));
        System.out.println("plain text  : " + XssProtector.toPlainText("<b>Hello</b>"));
        System.out.println("display html: " + XssProtector.toDisplayHtml("<b>Hello</b>"));
        System.out.println("null stays  : " + XssProtector.toRichText(null));
    }
}
```

```text
rich text   : <b>Hello</b>
plain text  : Hello
display html: &lt;b&gt;Hello&lt;/b&gt;
null stays  : null
```

| Method | Use it when | Effect |
|--------|-------------|--------|
| `toRichText` | users may format their text (bold, lists, links, tables, images) | keeps a safe set of tags and removes scripts and everything dangerous |
| `toPlainText` | only the words matter | removes all tags and keeps the text |
| `toDisplayHtml` | the text must be shown exactly as typed | turns `<` and `>` into `&lt;` and `&gt;`, so the browser shows the characters |

```java
String comment = request.getParameter("comment");

String forPage    = XssProtector.toDisplayHtml(comment);   // shown literally, as typed
String forEditor  = XssProtector.toRichText(comment);      // formatting kept, scripts removed
String forPlain   = XssProtector.toPlainText(comment);     // no markup at all
```

A `null` stays `null` in all three. Choose the method by where the text will be used, and apply it when the text is shown, or at the latest when it is stored, but do not forget it anywhere on the way.

## 15. Uploading files safely

An uploaded file is untrusted data. Its name can be a lie, its declared type can be a lie, and its content can be an attack. `FileSecurity` checks all of this in a fixed order before it puts the file in its place.

A `FileSecurity` object is configured once and kept for the life of the application, then called as often as needed, from many threads at the same time.

```java
import org.j2os.platform.jsecurity.file.FileSaveResult;
import org.j2os.platform.jsecurity.file.FileSecurity;

import javax.imageio.ImageIO;
import java.awt.image.BufferedImage;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.nio.file.Files;
import java.nio.file.Path;

public class UploadImage {

    public static void main(String[] args) throws Exception {
        Path uploads = Files.createTempDirectory("uploads");

        ByteArrayOutputStream png = new ByteArrayOutputStream();
        ImageIO.write(new BufferedImage(120, 80, BufferedImage.TYPE_INT_RGB), "png", png);
        byte[] bytes = png.toByteArray();

        FileSecurity avatars = FileSecurity.builder()
                .allowedMimeTypes("image/jpeg", "image/png")
                .maxSizeBytes(500 * 1024)
                .scannerType(FileSecurity.LINUX_CLAM_AV_SCANNER)
                .scannerPath("/usr/bin/clamscan")
                .storageDirectory(uploads)
                .overrideFile(false)
                .minImageWidth(100)
                .minImageHeight(100);

        try {
            avatars.save(new ByteArrayInputStream(bytes), "me.png", bytes.length);
        } catch (RuntimeException e) {
            System.out.println(e.getClass().getSimpleName() + ": " + e.getMessage());
        }

        FileSecurity photos = FileSecurity.builder()
                .allowedMimeTypes("image/jpeg", "image/png")
                .maxSizeBytes(500 * 1024)
                .scannerType(FileSecurity.LINUX_CLAM_AV_SCANNER)
                .scannerPath("/usr/bin/clamscan")
                .storageDirectory(uploads)
                .overrideFile(false);

        FileSaveResult saved = photos.save(new ByteArrayInputStream(bytes), "../Holiday Photo (final).PNG", bytes.length);
        System.out.println("original name : " + saved.getOriginalFileName());
        System.out.println("stored name   : " + saved.getFileName().replaceAll("[0-9a-f-]{36}_\\d{14}", "<uuid>_<time>"));
        System.out.println("detected type : " + saved.getDetectedMimeType());
        System.out.println("inside folder : " + saved.getFilePath().startsWith(uploads));
        System.out.println("size is real  : " + (saved.getSizeBytes() == Files.size(saved.getFilePath())));
    }
}
```

```text
ImageTooSmallException: Image dimensions (120x80) are below the configured minimum (100x100)
original name : ../Holiday Photo (final).PNG
stored name   : Holiday-Photo-final_<uuid>_<time>.png
detected type : image/png
inside folder : true
size is real  : true
```

The configuration methods are:

| Method | Meaning |
|--------|---------|
| `allowedMimeTypes(...)` | the allowed *MIME types* (names of file kinds such as `image/png` or `application/pdf`) |
| `maxSizeBytes(n)` | the largest allowed file |
| `scannerType(...)` | `FileSecurity.LINUX_CLAM_AV_SCANNER` or `FileSecurity.WINDOWS_DEFENDER_SCANNER` |
| `scannerPath(path)` | the full path of the antivirus program |
| `storageDirectory(path)` | the folder where accepted files are kept; it is created when needed |
| `overrideFile(boolean)` | `false` stores each file under a unique name; `true` overwrites a file of the same name |
| `quarantineDirectory(path)` | optional; where infected files are moved instead of being deleted |
| `scanTimeout(duration)` | optional; the longest time a scan may take, 30 seconds by default |
| `minImageWidth`, `minImageHeight`, `maxImageWidth`, `maxImageHeight` | optional limits for images |

`save(stream, originalFileName, declaredSizeBytes)` takes these steps:

1. It refuses a missing stream, and a declared size above the limit.
2. It copies the stream to a temporary file and counts the real bytes. The real count decides, so a lie in the declared size does not help.
3. It finds the **real type** of the file from its content, not from its name. A text file named `notes.png` is a text file.
4. It refuses a type that is not on the allowed list.
5. For a JPEG or PNG image, it checks the width and height against your limits and against a fixed ceiling of 40,000,000 pixels, which protects the server's memory. Then it *decodes the whole picture and writes it again*. This removes hidden data such as camera information and bytes appended after the image, and it proves that the file really is a picture.
6. It runs the antivirus scan.
7. It chooses the final name and moves the file into the storage folder.

The stored name is built from the original name, cleaned: only the last part is kept (so `../Holiday Photo (final).PNG` cannot point out of the folder), the extension is dropped, and characters other than letters, digits, dot, underscore and hyphen are replaced by `-`. The name is cut to 80 characters. The extension is then added from the **detected** type. With `overrideFile(false)` the name also gets a unique part and a time stamp, `Holiday-Photo-final_<uuid>_<time>.png` in the output; with `true` the name is just `Holiday-Photo-final.png` and an existing file is replaced.

`save` returns a `FileSaveResult` with `getFileName()`, `getFilePath()`, `getOriginalFileName()`, `getDetectedMimeType()` and `getSizeBytes()`.

The first program also shows a refusal: an image of 120 by 80 pixels does not reach the minimum of 100 by 100 that its configuration asks for in height. The second program shows the other refusals.

```java
import org.j2os.platform.jsecurity.file.FileSecurity;

import java.io.ByteArrayInputStream;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.concurrent.Callable;

public class UploadRejects {

    public static void main(String[] args) throws Exception {
        Path uploads = Files.createTempDirectory("uploads");
        Path quarantine = Files.createTempDirectory("quarantine");

        FileSecurity documents = FileSecurity.builder()
                .allowedMimeTypes("image/png", "application/pdf")
                .maxSizeBytes(1000)
                .scannerType(FileSecurity.LINUX_CLAM_AV_SCANNER)
                .scannerPath("/usr/bin/clamscan")
                .storageDirectory(uploads)
                .quarantineDirectory(quarantine)
                .overrideFile(false);

        byte[] text = "just some notes".getBytes(StandardCharsets.UTF_8);
        byte[] fakePng = new byte[]{(byte) 0x89, 'P', 'N', 'G', 0x0D, 0x0A, 0x1A, 0x0A, 1, 2, 3, 4};
        byte[] virus = "%PDF-1.4 TEST-VIRUS-MARKER".getBytes(StandardCharsets.UTF_8);

        String[] names = {"text sent as a document", "declared size too big", "real size too big",
                "damaged image", "infected pdf", "no stream"};
        Callable<?>[] attempts = {
                () -> documents.save(new ByteArrayInputStream(text), "notes.png", text.length),
                () -> documents.save(new ByteArrayInputStream(text), "notes.pdf", 5000),
                () -> documents.save(new ByteArrayInputStream(new byte[5000]), "big.pdf", 10),
                () -> documents.save(new ByteArrayInputStream(fakePng), "logo.png", fakePng.length),
                () -> documents.save(new ByteArrayInputStream(virus), "invoice.pdf", virus.length),
                () -> documents.save(null, "none.pdf", 0)
        };

        for (int i = 0; i < attempts.length; i++) {
            try {
                attempts[i].call();
                System.out.println(names[i] + " -> saved");
            } catch (RuntimeException e) {
                System.out.println(names[i] + " -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
            }
        }

        try (var left = Files.list(uploads); var held = Files.list(quarantine)) {
            System.out.println("files in uploads: " + left.count() + ", in quarantine: " + held.count());
        }

        try {
            documents.maxSizeBytes(5_000_000);
        } catch (IllegalStateException e) {
            System.out.println(e.getMessage());
        }
    }
}
```

```text
text sent as a document -> UnsupportedMimeTypeException: File type is not allowed; detected MIME: text/plain
declared size too big -> FileTooLargeException: File size exceeds the allowed limit (1000 bytes)
real size too big -> FileTooLargeException: File size exceeds the allowed limit (1000 bytes)
damaged image -> InvalidImageContentException: File content does not match the declared image type (image/png) or is corrupt
infected pdf -> MalwareDetectedException: File was detected as infected with malware
no stream -> NoFileProvidedException: No file was provided to save
files in uploads: 0, in quarantine: 1
This FileSecurity instance can no longer be configured after the first save() call
```

Each refusal has its own exception type, all of them extending `FileSecurityException` (which extends `RuntimeException`), so one `catch` can handle them together while a more specific one can answer differently. The last two lines of the output show two more facts:

- The infected file is not stored. It is moved to the quarantine folder (`in quarantine: 1`). Without a quarantine folder it is deleted. If it were impossible to delete or move, the problem is attached to the `MalwareDetectedException` as a *suppressed* exception.
- After the first call to `save`, the object cannot be configured any more, even if that first call failed. A configuration mistake must be fixed in the code and not patched on a running application.

About the scanners:

- **ClamAV** (`clamscan`): exit code 0 means clean, 1 means a virus was found, anything else is a scan error.
- **Windows Defender** (`MpCmdRun.exe`): exit code 0 means clean, and **any** other exit code is reported as a scan error (`ScanExecutionException`), never as a virus.
- If the scan takes longer than `scanTimeout`, the scanner and its child processes are stopped and `ScanTimeoutException` is thrown.
- The path of the scanner is checked on every `save`. A missing or wrong path gives `InvalidScannerPathException`, so a broken installation is noticed at once.
- A failed scan other than a virus deletes the temporary file. Nothing is stored.

An upload endpoint in Spring:

```java
@Configuration
public class UploadConfig {

    @Bean
    public FileSecurity avatarUploads(@Value("${app.upload.dir}") Path directory) {
        return FileSecurity.builder()
                .allowedMimeTypes("image/jpeg", "image/png")
                .maxSizeBytes(500 * 1024)
                .scannerType(FileSecurity.LINUX_CLAM_AV_SCANNER)
                .scannerPath("/usr/bin/clamscan")
                .storageDirectory(directory.resolve("avatars"))
                .quarantineDirectory(directory.resolve("quarantine"))
                .overrideFile(false)
                .minImageWidth(100)
                .minImageHeight(100);
    }
}

@RestController
public class AvatarController {

    private final FileSecurity avatarUploads;

    public AvatarController(FileSecurity avatarUploads) {
        this.avatarUploads = avatarUploads;
    }

    @PostMapping("/api/profile/avatar")
    public Map<String, String> upload(@RequestParam("file") MultipartFile file) throws IOException {
        try (InputStream in = file.getInputStream()) {
            FileSaveResult saved = avatarUploads.save(in, file.getOriginalFilename(), file.getSize());
            return Map.of("file", saved.getFileName());
        } catch (FileSecurityException e) {
            throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "The file was not accepted");
        }
    }
}
```

The message sent to the client says only that the file was not accepted. The exception message may contain details (such as the detected type) that are useful in a log, not for the user.

## 16. Sign in with Google

*OAuth* is the protocol that lets a user sign in with an account on another site, here Google, without giving your application the Google password. The steps are:

1. Your application sends the user's browser to Google with a link that names your application.
2. The user signs in at Google and agrees.
3. Google sends the browser back to your application, with a one-time *code*.
4. Your application exchanges the code for the user's profile.

`GoogleOAuthClient` does steps 1 and 4. It needs a *client id*, a *client secret* and a *redirect address* that you register in the Google Cloud console for your application.

```java
import org.j2os.platform.jsecurity.oauth.google.GoogleAuthorizationRequest;
import org.j2os.platform.jsecurity.oauth.google.GoogleOAuthClient;

public class GoogleSignIn {

    public static void main(String[] args) {
        GoogleOAuthClient google = new GoogleOAuthClient(
                "1234567890-demo.apps.googleusercontent.com",
                "demo-secret-not-real",
                "https://shop.example.com/login/google/callback");

        GoogleAuthorizationRequest request = google.authorizationUrl();

        String url = request.url();
        System.out.println(url.substring(0, url.indexOf("&state=")));
        System.out.println("state length : " + request.state().length());
        System.out.println("state in url : " + url.endsWith("&state=" + request.state()));
        System.out.println("fresh each time: " + !google.authorizationUrl().state().equals(request.state()));

        try {
            google.getProfile(" ");
        } catch (IllegalArgumentException e) {
            System.out.println(e.getMessage());
        }
        try {
            new GoogleOAuthClient("", "secret", "https://shop.example.com/cb");
        } catch (IllegalArgumentException e) {
            System.out.println(e.getMessage());
        }
    }
}
```

```text
https://accounts.google.com/o/oauth2/v2/auth?scope=openid+profile+email&response_type=code&access_type=offline&redirect_uri=https%3A%2F%2Fshop.example.com%2Flogin%2Fgoogle%2Fcallback&client_id=1234567890-demo.apps.googleusercontent.com
state length : 43
state in url : true
fresh each time: true
authorizationCode cannot be blank
clientId cannot be blank
```

- `authorizationUrl()` returns a `GoogleAuthorizationRequest` with the `url` to send the browser to, and a `state`. The state is a random value of 43 characters, new for every call.
- The state protects against a forged callback. **Your application must store it** (for example in the session) and compare it with the `state` that comes back in the callback. The client does not do this comparison for you.
- `getProfile(code)` exchanges the code with Google and returns a `GoogleProfile` with `name`, `givenName`, `familyName`, `picture`, `email` and `emailVerified`. A missing verification flag counts as `false`.
- A blank code is refused with `IllegalArgumentException`. If Google answers with an error status, the call throws `feign.FeignException`.
- The client may be used by many threads at the same time. Create one and keep it.
- A second constructor takes the connect and read timeouts in milliseconds.

```java
@RestController
public class GoogleLoginController {

    private final GoogleOAuthClient google = new GoogleOAuthClient(
            "1234567890-demo.apps.googleusercontent.com", System.getenv("GOOGLE_SECRET"),
            "https://shop.example.com/api/login/google/callback");
    private final TokenProcessor tokens;

    public GoogleLoginController(TokenProcessor tokens) {
        this.tokens = tokens;
    }

    @GetMapping("/api/login/google")
    public void start(HttpSession session, HttpServletResponse response) throws IOException {
        GoogleAuthorizationRequest request = google.authorizationUrl();
        session.setAttribute("oauth-state", request.state());
        response.sendRedirect(request.url());
    }

    @GetMapping("/api/login/google/callback")
    public Map<String, String> callback(@RequestParam String code, @RequestParam String state, HttpSession session) {
        String expected = (String) session.getAttribute("oauth-state");
        session.removeAttribute("oauth-state");
        if (expected == null || !expected.equals(state)) {
            throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "Unknown sign-in attempt");
        }
        GoogleProfile profile = google.getProfile(code);
        if (!profile.emailVerified()) {
            throw new ResponseStatusException(HttpStatus.FORBIDDEN, "The e-mail address is not verified");
        }
        return Map.of("token", tokens.createToken(profile.email(), Map.of("roles", List.of("USER")), 60L));
    }
}
```

The profile is trustworthy only as far as Google says it is. Accept an e-mail address as the user's identity only when `emailVerified()` is `true`.

## 17. The final example: accounts for a small shop

A shop needs sign-in, a profile that the customer may edit, and a list of customers for the staff. The rules are:

- a password is never stored, only its Argon2 hash;
- a signed-in user holds a token with the role `USER`;
- the customer may change the display name and the e-mail, but not the user name, the password hash or the loyalty points;
- other people never see the e-mail or the password hash.

The following program runs this logic, without the web layer.

```java
import org.j2os.platform.jsecurity.access.RequestAccessControl;
import org.j2os.platform.jsecurity.access.ResponseAccessControl;
import org.j2os.platform.jsecurity.cryptography.Cryptography;
import org.j2os.platform.jsecurity.token.TokenProcessor;

import java.util.List;
import java.util.Map;
import java.util.TreeMap;

public class ShopAccounts {

    public static void main(String[] args) {
        TokenProcessor tokens = new TokenProcessor("shop-api",
                "change-me-this-secret-must-be-at-least-sixty-four-bytes-long-0123456789");

        // 1. registration: only the hash is stored
        Profile stored = new Profile();
        stored.username = "alice";
        stored.passwordHash = Cryptography.hashByArgon2("S3cret-pass");
        stored.displayName = "Alice";
        stored.email = "alice@example.com";
        stored.loyaltyPoints = 120;

        // 2. login: check the password, then issue a token that carries the roles
        String attempt = "S3cret-pass";
        String token = null;
        if (Cryptography.checkByArgon2(attempt, stored.passwordHash)) {
            token = tokens.createToken(stored.username, Map.of("roles", List.of("USER")), 60L);
        }
        System.out.println("login ok    : " + (token != null));
        System.out.println("who is it   : " + tokens.getSubject(token) + " " + tokens.getClaimAsList(token, "roles"));

        // 3. profile update: the client may only change the display name and the e-mail
        RequestAccessControl.registerFieldLimitation("shop", Profile.class, "loyaltyPoints", "update");
        RequestAccessControl.registerFieldLimitation("shop", Profile.class, "passwordHash", "update");
        RequestAccessControl.registerFieldLimitation("shop", Profile.class, "username", "update");

        Profile fromClient = new Profile();
        fromClient.displayName = "Alice B.";
        fromClient.email = "alice@example.org";
        fromClient.loyaltyPoints = 1_000_000;
        fromClient.passwordHash = "something-else";

        Profile accepted = RequestAccessControl.apply("shop", fromClient, stored, "update");
        System.out.println("after update: " + accepted.displayName + ", " + accepted.email
                + ", points " + accepted.loyaltyPoints + ", same hash " + accepted.passwordHash.equals(stored.passwordHash));

        // 4. response: ordinary users never see the e-mail or the hash of others
        Map<String, Object> forOthers = ResponseAccessControl.apply(accepted,
                List.of("email", "passwordHash"), ResponseAccessControl.REMOVE);
        System.out.println("shown to others: " + new TreeMap<>(forOthers));
    }
}

class Profile {
    String username;
    String passwordHash;
    String displayName;
    String email;
    Integer loyaltyPoints;

    public String getUsername() { return username; }
    public String getPasswordHash() { return passwordHash; }
    public String getDisplayName() { return displayName; }
    public String getEmail() { return email; }
    public Integer getLoyaltyPoints() { return loyaltyPoints; }
}
```

```text
login ok    : true
who is it   : alice [USER]
after update: Alice B., alice@example.org, points 120, same hash true
shown to others: {displayName=Alice B., loyaltyPoints=120, username=alice}
```

The steps of the program, and the module that does each:

1. **Registration.** `hashByArgon2` turns the password into the text that is stored. The password itself is forgotten.
2. **Sign-in.** `checkByArgon2` compares the typed password with the stored text. On success, `createToken` makes a token for `alice` with the role `USER`, valid for 60 minutes. Reading it back gives the user and the roles.
3. **Profile update.** Three fields are registered as forbidden for the action `update`. The client sends a new display name and a new e-mail, and also tries to set `loyaltyPoints` and `passwordHash`. `apply` with the stored profile keeps the two allowed changes and restores the rest: the points are still 120 and the hash is unchanged.
4. **Response.** `ResponseAccessControl` removes `email` and `passwordHash` from the result. In the output, the user name is still `alice` because it is restored from the stored object.

In the web application, the same pieces are connected like this. First the filter and the token processor from chapter 9. Then the login:

```java
@RestController
public class LoginController {

    private final AccountRepository accounts;
    private final TokenProcessor tokens;

    public LoginController(AccountRepository accounts, TokenProcessor tokens) {
        this.accounts = accounts;
        this.tokens = tokens;
    }

    @PostMapping("/api/login")
    public Map<String, String> login(@RequestBody LoginForm form) {
        Account account = accounts.findByUsername(form.username());
        if (account == null || !Cryptography.checkByArgon2(form.password(), account.getPasswordHash())) {
            throw new ResponseStatusException(HttpStatus.UNAUTHORIZED, "Wrong user name or password");
        }
        String token = tokens.createToken(account.getUsername(), Map.of("roles", account.getRoles()), 60L);
        return Map.of("token", token);
    }
}
```

The login path is in the open list of the filter, because the caller has no token yet. A wrong user name and a wrong password give the same answer, so the response does not tell an attacker which one was wrong.

The profile update uses the user name from the token, never one sent by the client:

```java
@RestController
public class ProfileController {

    private final ProfileRepository profiles;

    public ProfileController(ProfileRepository profiles) {
        this.profiles = profiles;
    }

    @PutMapping("/api/profile")
    public Map<String, Object> update(@RequestBody Profile fromClient, HttpServletRequest request) {
        String user = (String) request.getAttribute("authorization.subject");
        Profile stored = profiles.findByUsername(user);

        Profile accepted = RequestAccessControl.apply("shop", fromClient, stored, "update");
        profiles.save(accepted);

        return ResponseAccessControl.apply(accepted, List.of("passwordHash"), ResponseAccessControl.REMOVE);
    }
}
```

With the avatar upload from chapter 15 and the Google sign-in from chapter 16, the application has sign-in, role checks, safe input and output, and safe files, and each of them is a few lines of code.

---

## Appendix A. Quick reference

### A.1 Cryptography

| Method | Description |
|--------|-------------|
| `encodeBase64(bytes)`, `decodeBase64(text)` | Base64 text and bytes |
| `encodeBase64URL(bytes)`, `decodeBase64URL(text)` | the URL-safe form |
| `hashByMD5(text)` | MD5 as hexadecimal; for old formats only |
| `hashBySHA2_256(text)`, `hashBySHA2_512(text)` | SHA-2 as hexadecimal |
| `hashBySHA3_256(text)`, `hashBySHA3_512(text)` | SHA-3 as hexadecimal |
| `hashFileByMD5(path)`, `hashFileBySHA2_256(path)`, `hashFileBySHA2_512(path)` | the hash of a file, read as a stream |
| `hashByBCrypt(password)`, `checkByBCrypt(password, hash)` | BCrypt with cost 12 |
| `hashByArgon2(password)`, `checkByArgon2(password, hash)` | Argon2id, 64 MB, 3 passes, 1 lane |
| `hashByPBKDF2(password)`, `checkByPBKDF2(password, hash)` | PBKDF2 with HMAC-SHA-256 and 210000 rounds |
| `generateAESKey(bits)` | a random AES key of 128, 192 or 256 bits |
| `encryptStringByAES(text, key)`, `decryptStringByAES(sealed, key)` | AES-GCM with a key |
| `encryptStringByAES(text, password)`, `decryptStringByAES(sealed, password)` | AES-GCM with a password |
| `encryptFileByAES(path, key)`, `decryptFileByAES(path, key)` | in place, with a key |
| `encryptFileByAES(path, password)`, `decryptFileByAES(path, password)` | in place, with a password |
| the same four with a last argument `durable` | also forces the data to disk |
| `generateRSAKeys(size)` | an RSA key pair, at least 2048 bits |
| `encodeRSAPublicKey`, `decodeRSAPublicKey` | public key as Base64 (X.509) and back |
| `encodeRSAPrivateKey`, `decodeRSAPrivateKey` | private key as Base64 (PKCS#8) and back |
| `encryptStringByRSA(text, publicKey)`, `decryptStringByRSA(bytes, privateKey)` | RSA-OAEP with SHA-256 |
| `uuid()`, `randomInt(...)`, `randomDouble(...)`, `randomBoolean()` | random values |
| `randomAlphaString(n)`, `randomAlphaNumericString(n)` | random texts |

### A.2 TokenProcessor

| Member | Description |
|--------|-------------|
| `new TokenProcessor(issuer, secretKey)` | the secret is at least 64 bytes |
| `createToken(subject, claims, minutes)` | a new token; `null` or `0` minutes means no expiry |
| `getSubject(token)` | the user |
| `getClaimAsString(token, name)`, `getClaimAsList(token, name)`, `getClaimAsMap(token, name)` | a claim |
| `isTokenValid(token)`, `isTokenExpired(token)` | boolean answers |
| `setClaim(token, name, value)`, `removeClaim(token, name)` | a changed token with the same times |
| `renewToken(token)`, `renewToken(token, minutes)` | a token with a fresh issue time |
| each of the above with `IgnoringExpiration` | accepts an expired token, still checks the signature and the issuer |
| `isTokenValidIgnoringExpiration(token)` | validity without the expiry check |

### A.3 AuthorizationFilter

| Member | Description |
|--------|-------------|
| `issuer`, `secretKey` | required; the token issuer and its secret |
| `openPermits(list)` | patterns that need no token |
| `rolePermits(map)` | pattern to roles; a `LinkedHashMap`, first match wins |
| `roleClaimName`, `tokenSource`, `cookieName` | where the roles and the token are |
| `bypassPreflightOptions(boolean)` | let `OPTIONS` pass |
| `before(hook)`, `after(hook)`, `onError(handler)` | the three customization points |
| `Builder.apply()` | merges the settings; ends every chain |
| `new AuthorizationFilter()` | the object to register as a `Filter` |
| `AuthorizationFilter.authorize(request, response)` | the check for use inside a servlet |
| attributes `authorization.subject`, `authorization.roles` | set on success |
| `TokenSource` | `AUTHORIZATION_HEADER_BEARER`, `COOKIE` |

### A.4 RequestAccessControl

| Member | Description |
|--------|-------------|
| `registerFieldLimitation(scope, Class, field, action)` | forbids a field for an action; validated at once |
| `registerFieldLimitation(scope, className, field, action)` | the same by class name; problems are only logged |
| `registerActionDenial(scope, Class, action)`, the className form | forbids a whole action |
| `unregisterFieldLimitation(...)`, `unregisterActionDenial(...)` | remove a rule |
| `apply(scope, target, action)` | a copy with the forbidden fields set to `null` |
| `apply(scope, target, oldTarget, action)` | a copy with the forbidden fields taken from `oldTarget` |
| `DeniedException` | thrown for a denied action |

### A.5 ResponseAccessControl

| Member | Description |
|--------|-------------|
| `apply(Map result, fields, action)` | restricts every row of a result map with a `rows` entry |
| `apply(List rows, fields, action)` | restricts a list; returns a list of maps |
| `apply(Object object, fields, action)` | restricts one object; returns a map |
| `EMPTY` | the action that blanks a field; any other action removes it |
| `REMOVE` | documents the default behavior |

### A.6 XssProtector

| Method | Description |
|--------|-------------|
| `toRichText(text)` | safe HTML with formatting kept |
| `toPlainText(text)` | the text without any markup |
| `toDisplayHtml(text)` | HTML-escaped text |

### A.7 FileSecurity

| Member | Description |
|--------|-------------|
| `FileSecurity.builder()` | a new, unconfigured object |
| `allowedMimeTypes`, `maxSizeBytes`, `scannerType`, `scannerPath`, `storageDirectory`, `overrideFile` | required settings |
| `quarantineDirectory`, `scanTimeout`, `minImageWidth`, `minImageHeight`, `maxImageWidth`, `maxImageHeight` | optional settings |
| `save(stream, originalFileName, declaredSizeBytes)` | checks and stores a file; returns a `FileSaveResult` |
| `FileSaveResult` | `getFileName`, `getFilePath`, `getOriginalFileName`, `getDetectedMimeType`, `getSizeBytes` |
| `ScannerType` | `LINUX_CLAM_AV`, `WINDOWS_DEFENDER` |

### A.8 GoogleOAuthClient

| Member | Description |
|--------|-------------|
| `new GoogleOAuthClient(clientId, clientSecret, redirectUri)` | all three required |
| `new GoogleOAuthClient(..., connectTimeoutMillis, readTimeoutMillis)` | with timeouts |
| `authorizationUrl()` | a `GoogleAuthorizationRequest(url, state)` |
| `getProfile(code)` | a `GoogleProfile(name, givenName, familyName, picture, email, emailVerified)` |

## Appendix B. Failure handling

| Situation | Result |
|-----------|--------|
| a `null` argument to a Base64 or hash method | `IllegalArgumentException` |
| an unreadable file for a file hash | `UncheckedIOException` |
| a wrong format of a stored password hash | `IllegalArgumentException` |
| a stored hash with settings above the safety limits | `IllegalArgumentException` |
| a wrong AES key or password, or damaged data | `IllegalStateException` |
| an AES text that is too short, or an AES key size not allowed | `IllegalArgumentException` |
| an RSA key under 2048 bits, or data too large for RSA | `IllegalArgumentException` |
| an RSA key text that cannot be decoded | `IllegalArgumentException` |
| a file that is empty, not in the module's format, or opened with the wrong kind of secret | `IllegalArgumentException` |
| a file encryption or decryption that fails | `IllegalStateException`; the original file is kept |
| a blank issuer, subject or claim name, a secret under 64 bytes | `IllegalArgumentException` |
| an unsupported claim value, a registered claim name, a negative lifetime | `IllegalArgumentException` |
| an expired, forged or foreign token | an Auth0 exception (`TokenExpiredException`, `SignatureVerificationException`, `InvalidClaimException` and others) |
| `apply()` of the filter without an issuer or secret | `IllegalStateException` |
| a request to the filter before it is configured | `IllegalStateException` (`ServletException` when used as a filter) |
| a refused request | the error response of chapter 10; the application is not called |
| an exception in the application behind the filter | passes through unchanged |
| a forbidden field that does not exist, or has a primitive type | `IllegalArgumentException` at registration |
| a class without a no-argument constructor, or a record | `RuntimeException` at `apply` |
| a denied action | `RequestAccessControl.DeniedException` |
| a file that is refused | a subclass of `FileSecurityException` (Appendix D) |
| a configuration method after the first `save` | `IllegalStateException` |
| a blank client id, client secret, redirect address or code | `IllegalArgumentException` |
| an error status from Google | `feign.FeignException` |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Never store or log a password.** Store the hash of chapter 2. Do not use SHA-256 or MD5 for passwords.
- **Never keep keys and secrets in the code.** Read them from the environment or a secrets store. Anyone who has the token secret can create tokens for any user.
- **A token is signed, not encrypted.** Do not put secrets in claims.
- **A token cannot be taken back.** It is valid until it expires. Keep lifetimes short, and renew tokens instead of making them long.
- **Roles are copied into the token at sign-in.** If an administrator removes a role, the old token still has it until it expires.
- **The order of the role rules decides.** Put the specific patterns first and the general ones later, in a `LinkedHashMap`.
- **The context path counts.** An application under `/shop` writes `/shop/...` in every pattern.
- **A trailing slash and the letter case matter.** `/api/orders/7/` is not `/api/orders/7`.
- **The filter settings are global.** All code in the application shares one configuration, and a later `apply()` changes it for everyone.
- **Open paths set no attributes.** A controller behind an open path must not expect `authorization.subject`.
- **Do not trust a client field that you did not check.** Use `RequestAccessControl` for every object that is filled from a request, with a rule for each field the client must not set.
- **A restricted field needs a wrapper type.** Use `Integer`, not `int`.
- **The result of `ResponseAccessControl` is a map.** It is for sending, not for further use as an object.
- **A hidden field is not a secret field.** Removing a field from a response protects that response only. Keep secrets out of the objects that are returned in the first place.
- **Match the cleaning method to the use.** Escaped text is not rich text, and cleaned rich text is not safe to put inside a script or an attribute you build yourself.
- **Do not trust the file name or the declared type.** The module checks the content, but a file that it accepts is still a user's file: never run it, and serve it with the stored name.
- **`overrideFile(true)` overwrites without asking.** Use it only for names that you control.
- **Only JPEG and PNG are re-encoded.** For every other type, the virus scan is the only content check.
- **Check `emailVerified`.** An e-mail address without verification is a claim, not an identity.
- **Store the OAuth state.** If you do not compare it, the sign-in is open to a forged callback.
- **An unknown user answers faster than a wrong password**, because no hash is computed. If that difference matters, compute a hash for unknown users too.

### C.2 Behavior worth knowing

- All cryptographic parameters (costs, key sizes, rounds) are fixed in the module. They cannot be changed by configuration.
- Every hash and every encrypted text is different each time, because salts and IVs are random.
- A BCrypt hash is made from a fingerprint of the password and is not interchangeable with BCrypt hashes from other software.
- The filter tries the role rules in order. The first match decides, even if a later rule would have allowed the caller.
- The filter runs `before` for every request, including open ones. It runs `after` only after a full success.
- `setClaim` and `removeClaim` keep the old issue and expiry times. `renewToken` makes new ones.
- `RequestAccessControl.apply` makes a shallow copy. Only containers on a restricted path are copied.
- A restricted field in `apply(scope, target, oldTarget, action)` is taken from `oldTarget`, which must be of the same class as `target`.
- `FileSecurity` takes a snapshot of its settings at the start of every `save`, so concurrent calls do not influence each other.
- The scan output is read up to 64 KB. More is discarded and the scanner is not blocked.
- A scan problem that is not a virus removes the temporary file and keeps nothing.

### C.3 Limits

- **Argon2:** a stored hash is refused above 262144 KB of memory, 10 passes, 8 lanes or a 128-byte hash.
- **PBKDF2:** a stored hash is refused above 1000000 rounds or a 64-byte hash.
- **AES:** key sizes of 128, 192 and 256 bits. RSA: at least 2048 bits, and only small data.
- **Secret for tokens:** at least 64 bytes. The algorithm is HMAC-SHA-512.
- **Claims:** text, lists of text and maps only.
- **Images:** at most 40,000,000 pixels, a fixed ceiling. JPEG is written again with quality 0.92.
- **File names:** at most 80 characters in the base name.
- **Scan time:** at least 1 millisecond; 30 seconds by default.
- **Digital signatures, key storage and certificate handling** are not part of the module.

### C.4 Tuning

- Use Argon2id for new passwords. Remember that each check needs 64 MB. Many sign-ins at the same moment need many times that, so size the memory of the server for the busiest minute, or limit the number of simultaneous sign-ins.
- Make tokens short-lived, from a few minutes to an hour, and renew them while the user is active.
- Keep the list of open paths as short as possible, and keep the role rules readable, with one line per area of the application.
- Write a custom `onError` when the front end needs a different JSON shape, but keep the status codes.
- Register request and response rules in one place, at startup, so that the whole policy can be read in a single file.
- Set `maxSizeBytes` to what the feature needs, and set the size limit of the web server a little higher, so that large uploads are stopped early.
- Install and update the antivirus program on the server and test the scanner path after every deployment.
- Use `quarantineDirectory` when infected files must be examined later, and clean it regularly.
- Use `durable` for file encryption only when losing the file in a power failure is not acceptable.

## Appendix D. Message reference

### Cryptography

**IllegalArgumentException**

| Message | Meaning |
|---------|---------|
| `<name> must not be null` | a `null` argument; `<name>` is the parameter |
| `Invalid BCrypt hash format` | the stored text is not a BCrypt hash |
| `Invalid Argon2 hash format` | the stored text is not in the Argon2 format |
| `Argon2 hash cost parameters exceed the allowed safety ceilings (memory<=262144KB, iterations<=10, parallelism<=8, hash length<=128 bytes)` | the settings in the stored text are too high |
| `Invalid PBKDF2 hash format` | the stored text is not in the PBKDF2 format |
| `PBKDF2 hash parameters exceed the allowed safety ceilings (iterations<=1000000, hash length<=64 bytes)` | the settings in the stored text are too high |
| `AES key size must be 128, 192, or 256 bits` | a key size that is not allowed |
| `Invalid AES payload: too short to be valid` | an encrypted text that is too short |
| `File is empty or corrupted: <path>` | the file to decrypt has no content |
| `Unrecognized AES file format (unknown leading flag byte: N); file is corrupted or was not produced by encryptFileByAES: <path>` | the file does not start like an encrypted file |
| `File was encrypted with a password; a password is required to decrypt it.` | the file was made with a password but is opened with a key |
| `File was encrypted with a SecretKey; that same key is required to decrypt it.` | the file was made with a key but is opened with a password |
| `Invalid AES file: truncated salt: <path>`, `Invalid AES file: truncated IV: <path>` | the file ends too early |
| `RSA key size must be at least 2048 bits for adequate security.` | the key size is too small |
| `Plain text is too large for RSA encryption with this key size. RSA is only suitable for small payloads (e.g. an AES key) — use AES for larger data.` | the text does not fit |
| `Invalid RSA public key encoding`, `Invalid RSA private key encoding` | the key text cannot be decoded |
| `maximum must be positive` | `randomInt(max)` with `max` of zero or less |
| `minimum must be less than maximum` | an empty or backwards range |
| `length must be positive` | a random text of length zero or less |

**IllegalStateException**

| Message | Meaning |
|---------|---------|
| `AES decryption failed - wrong key/password or corrupted data` | text decryption fails |
| `AES encryption failed` | text encryption fails |
| `AES file decryption failed (wrong key/password or corrupted file): <path>` | file decryption fails; the file is unchanged |
| `AES file encryption failed: <path>` | file encryption failed; the file is unchanged |
| `RSA decryption failed - wrong key or corrupted data` | RSA decryption fails |
| `RSA encryption failed` | RSA encryption fails |

**UncheckedIOException**: `Failed reading file: <path>` for a file hash.

### TokenProcessor

**IllegalArgumentException**

| Message | Meaning |
|---------|---------|
| `Issuer cannot be null or blank` | no issuer |
| `Secret key cannot be null or blank` | no secret |
| `HMAC512 secret key must contain at least 64 UTF-8 bytes` | the secret is too short |
| `Subject cannot be null or blank` | no subject |
| `Expiration duration cannot be negative` | a negative lifetime |
| `Expiration duration is too large` | a lifetime that overflows |
| `Claim name cannot be null or blank` | no claim name |
| `Registered JWT claim cannot be used: <name>` | `iss`, `sub`, `aud`, `exp`, `nbf`, `iat` or `jti` |
| `Claim value cannot be null` | a `null` value |
| `Unsupported claim value type: <class>` | a value of another type |
| `Claim list must contain only String values` | a list with other elements |
| `Claim map keys must be String` | a map with other keys |
| `Claim map values must be String or List<String>` | a map with other values |

The `IgnoringExpiration` methods can also throw `JWTDecodeException` (`Token cannot be null or blank`), `AlgorithmMismatchException` and `InvalidClaimException` (`Invalid token issuer`).

### AuthorizationFilter

| Exception | Message | Meaning |
|-----------|---------|---------|
| `IllegalStateException` | `issuer and secretKey must be set before apply()` | the first `apply()` lacks one |
| `IllegalStateException` | `AuthorizationFilter is not configured yet. Call AuthorizationFilter.issuer(...).secretKey(...)....apply() before use.` | the filter is used before `apply()` |
| `IllegalArgumentException` | `Path pattern must not be blank` | a blank open pattern |
| `IllegalArgumentException` | `Path pattern must not be null or blank` | a blank role pattern |
| `IllegalArgumentException` | `Allowed roles for path "<path>" must not be null` | a rule without roles |
| `IllegalArgumentException` | `roleClaimName must not be null or blank`, `cookieName must not be null or blank` | a blank name |
| `NullPointerException` | `openPathPatterns must not be null`, `tokenSource must not be null` and similar | a `null` setting |
| `MissingTokenException` | `No Authorization header in the form "Bearer <token>" was found` | no usable header |
| `MissingTokenException` | `No cookie named "<name>" was found` | no cookie |
| `InvalidTokenException` | `Multiple cookies named "<name>" were supplied` | the cookie is sent twice |
| `InvalidTokenException` | `Token is invalid` | the token could not be verified |
| `InvalidTokenException` | `The roles claim in the token is invalid` | the roles claim cannot be read |
| `ExpiredTokenException` | `Token has expired` | the token is expired |
| `InsufficientRoleException` | `User does not have a sufficient role to access <path>` | no allowed role |
| `UnmappedPathException` | `Path <path> is not covered by any configured access rule` | no rule matches |
| `MalformedRequestPathException` | `Request path contains a disallowed encoded or suspicious sequence` | `%2f`, `%5c`, `%2e`, a backslash or `//` |
| `MalformedRequestPathException` | `Request path could not be decoded` | a broken percent sequence or invalid UTF-8 |
| `MalformedRequestPathException` | `Request path contains a disallowed control character` | a control character |
| `MalformedRequestPathException` | `Request path contains an invalid traversal sequence` | `..` above the root |

All six exceptions in the table extend `AuthorizationFilterException`, which extends `RuntimeException`. They live in the package `org.j2os.platform.jsecurity.authorization.exception`.

### RequestAccessControl and ResponseAccessControl

| Exception | Message | Meaning |
|-----------|---------|---------|
| `IllegalArgumentException` | `field path must not be blank` | a blank field path |
| `NullPointerException` | `field must not be null` | no field |
| `IllegalArgumentException` | `Field '<segment>' not found on <class> while validating path '<path>'` | the field does not exist |
| `IllegalArgumentException` | `Field '<field>' on <class> must be a Wrapper type (e.g. Integer, not int) to be used with RequestAccessControl - apply() nulls out restricted fields` | a primitive field |
| `IllegalArgumentException` | `target (<A>) and oldTarget (<B>) must be the same runtime class` | the two objects differ in class |
| `RuntimeException` | `<class> must have a no-arg constructor to be used with RequestAccessControl` | no no-argument constructor |
| `RuntimeException` | `<class> is a record and cannot be used with RequestAccessControl - ...` | a record |
| `RuntimeException` | `Field '<field>' not found on runtime class <class> - ...` | the path does not fit the real object |
| `RequestAccessControl.DeniedException` | the action name | the action is denied |

### FileSecurity

All exceptions extend `FileSecurityException`, which extends `RuntimeException`, and live in `org.j2os.platform.jsecurity.file.exception`.

| Exception | Message | Meaning |
|-----------|---------|---------|
| `MissingConfigurationException` | `allowedMimeTypes must have at least one value` | no allowed type |
| `MissingConfigurationException` | `maxSizeBytes must be set and positive` | no or invalid size limit |
| `MissingConfigurationException` | `scannerType must be set`, `storageDirectory must be set` | a missing setting |
| `MissingConfigurationException` | `overrideFile must be explicitly set to true or false` | a missing choice |
| `MissingConfigurationException` | `scanTimeout must be at least 1 millisecond`, `scanTimeout is too large to be represented in milliseconds` | an invalid time |
| `MissingConfigurationException` | `minImageWidth must be positive` and the same for the other three limits | an invalid limit |
| `MissingConfigurationException` | `minImageWidth cannot be greater than maxImageWidth`, and the same for the height | a minimum above its maximum |
| `InvalidScannerPathException` | `scannerPath is not set` | no scanner path |
| `InvalidScannerPathException` | `scannerPath is not a valid path: <path>` | a path that cannot be read |
| `InvalidScannerPathException` | `Scanner executable not found at the given path: <path>` | no program there |
| `IllegalStateException` | `This FileSecurity instance can no longer be configured after the first save() call` | configuration after `save` |
| `NoFileProvidedException` | `No file was provided to save` | a `null` stream |
| `FileTooLargeException` | `File size exceeds the allowed limit (N bytes)` | the declared or real size is above the limit |
| `UnsupportedMimeTypeException` | `File type is not allowed; detected MIME: <type>` | the detected type is not allowed; `getDetectedMimeType()` returns it |
| `InvalidImageContentException` | `File content does not match the declared image type (<type>) or is corrupt` | a damaged image |
| `ImageTooSmallException` | `Image dimensions (WxH) are below the configured minimum (WxH)` | an image below the minimum |
| `ImageTooLargeException` | `Image dimensions (WxH) exceed the configured maximum (WxH)` | an image above your maximum |
| `ImageTooLargeException` | `Image dimensions (WxH = N pixels) exceed the allowed limit of 40000000 pixels` | an image above the fixed ceiling |
| `MalwareDetectedException` | `File was detected as infected with malware` | the scanner finds a virus |
| `ScanExecutionException` | `Scanner execution ended with unexpected exit code N; raw output: ...` | the scan fails |
| `ScanExecutionException` | `Could not start the scanner process: <command>`, `Scan execution was interrupted` | the scanner does not start or is interrupted |
| `ScanTimeoutException` | `Antivirus scan did not finish within N seconds` | the scan takes too long |
| `FileStorageException` | `Could not create directory storageDirectory: ...`, `Could not create temp file in ...`, `Error reading/writing file content`, `Could not detect the file's MIME type`, `Could not save the final file to ...`, `Could not read the final file size`, `Error while re-encoding the image file`, `Could not create a temp file for image re-encoding` | a problem with the disk |

### GoogleOAuthClient

| Exception | Message | Meaning |
|-----------|---------|---------|
| `IllegalArgumentException` | `clientId cannot be blank`, `clientSecret cannot be blank`, `redirectUri cannot be blank` | a missing constructor value |
| `IllegalArgumentException` | `authorizationCode cannot be blank` | an empty code |
| `feign.FeignException` | status of Google's answer | Google refused the request |
