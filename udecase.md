https://systemdesignschool.io/problems/leetcode/solution?utm_source=neetcode

https://systemdesignschool.io/problems/url-shortener/solution

https://systemdesignschool.io/problems/webhook/solution?utm_source=neetcode

https://youtu.be/-eMtcFqj8vI  -  also see googledocs.md in same repo


A URL shortener generates a unique short URL by combining **ID generation + encoding + collision handling + database mapping**.

Typical flow:

```
Original URL:
https://example.com/blog/how-url-shorteners-work

        ↓

Generate unique ID
(e.g., 125673)

        ↓

Encode ID to short form
(Base62)

125673 → "WJf"

        ↓

Store mapping

WJf → https://example.com/blog/how-url-shorteners-work

        ↓

Short URL

https://myapp.com/WJf
```

## Common approaches

### 1. Auto-increment ID + Base62 (most common)

Database:

```sql
id     original_url
1      google.com
2      youtube.com
3      openai.com
```

Convert `id` to Base62:

Characters:

```text
0-9 = 10 chars
a-z = 26 chars
A-Z = 26 chars

Total = 62
```

Example:

```java
125 → "cb"
1000000 → "4C92"
```

Java example:

```java
String chars = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";

String encode(long num){
    StringBuilder sb = new StringBuilder();

    while(num > 0){
        sb.append(chars.charAt((int)(num % 62)));
        num /= 62;
    }

    return sb.reverse().toString();
}
```

Advantages:

* Small URLs
* No collisions
* Fast lookup
* Easy to scale

Problem:

* Predictable sequence:

```text
abc
abd
abe
```

Someone can guess future URLs.

---

### 2. Hash the original URL

Generate:

```java
MD5("google.com")
→ 8ffdefbdec...

Take first 7 chars:

8ffdefb
```

Store:

```text
8ffdefb → google.com
```

Problem:

Two URLs might produce same first 7 characters:

```text
hash1 → abc123x
hash2 → abc123x
```

Need collision handling:

```java
if(codeExists){
     take next 8 chars
}
```

Advantages:

* Deterministic
* Same URL can return same short URL

Disadvantages:

* Collision management required

---

### 3. Random string generation

Generate:

```java
A7xP9m
```

Check database:

```java
if(codeExists())
     regenerate();
```

Pseudo:

```java
while(true){
    code=randomString(6);

    if(!exists(code))
        return code;
}
```

Problem:

As database grows:

```text
6-char Base62:

62^6 ≈ 56 billion combinations
```

Collisions become more frequent.

---

### 4. Distributed ID generation (large systems)

For systems like Bitly/TinyURL with multiple servers:

Using database auto-increment becomes difficult:

```text
Server1 → ID=100
Server2 → ID=100
```

Use distributed ID generation:

* Snowflake algorithm
* UUID
* Redis INCR
* Zookeeper-based sequence

Snowflake example:

```text
Timestamp + MachineId + Sequence
```

Result:

```text
7823472938472
```

Encode to Base62:

```text
gX8Ab2
```

---

## Real system design (TinyURL style)

```
User
   ↓
API Gateway
   ↓
URL Service
   ↓
Generate ID (Snowflake)
   ↓
Base62 encode
   ↓
Store in DB

shortCode → originalURL
```

For redirect:

```
User hits:

myurl.com/gX8Ab2

        ↓

Lookup DB/Cache

gX8Ab2 → original URL

        ↓

HTTP 301/302 redirect
```

For performance:

* Cache popular URLs in Redis
* Store mappings in DB
* CDN for global access

---

For system-design interviews, the expected answer is usually:

**"Generate a unique numeric ID (auto-increment/Snowflake), encode using Base62, store mapping in DB, and use cache for reads."**

Snowflake already generates a **unique ID**, so Base62 is **not needed for uniqueness**. It is used mainly to make the URL **short and user-friendly**.

Example Snowflake ID:

```text
78234729384729472
```

If you directly use it:

```text
https://tiny.com/78234729384729472
```

Problems:

* Long URL
* Less readable
* Harder to share/type
* Takes more characters in messages/social posts

Now encode the same number into Base62:

```text
78234729384729472
        ↓
Base62
        ↓
gX8Ab2
```

URL becomes:

```text
https://tiny.com/gX8Ab2
```

Much shorter.

Why does this happen?

Base62 uses:

```text
0-9    → 10 chars
a-z    → 26 chars
A-Z    → 26 chars

Total = 62 symbols
```

Each character stores more information than a decimal digit.

Quick comparison:

| Number system | Symbols | Characters needed for same value |
| ------------- | ------: | -------------------------------: |
| Decimal       |      10 |                             More |
| Base36        |      36 |                            Fewer |
| Base62        |      62 |                       Even fewer |

For example:

```text
1000000

Decimal → "1000000"   (7 chars)
Base62  → "4C92"      (4 chars)
```

The uniqueness still comes entirely from Snowflake.

Flow:

```text
Snowflake
(Timestamp + MachineId + Sequence)

       ↓

Unique numeric ID
78234729384729472

       ↓

Base62 encode

       ↓

gX8Ab2

       ↓

Store

gX8Ab2 → original_url
```

Think of it this way:

* **Snowflake → creates uniqueness**
* **Base62 → compresses for readability**

Without Base62 the system still works; the URLs just become unnecessarily long.
To convert a decimal number to Base62, you repeatedly:

1. Divide the number by `62`
2. Store the remainder
3. Map the remainder to a character
4. Continue with the quotient
5. Reverse the collected characters

Base62 character set:

```text
0-9  -> values 0-9
a-z  -> values 10-35
A-Z  -> values 36-61
```

Index table:

```text
0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ
```

Example: convert `125673`

| Number | Divide by 62 | Quotient | Remainder | Character |
| ------ | -----------: | -------: | --------: | --------- |
| 125673 |    125673/62 |     2026 |        61 | Z         |
| 2026   |      2026/62 |       32 |        42 | G         |
| 32     |        32/62 |        0 |        32 | w         |

Collected:

```text
ZGw
```

Reverse it:

```text
wGZ
```

So:

```text
125673(decimal) = wGZ(Base62)
```

Java implementation:

```java
public class Base62 {

    private static final String CHARS =
        "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";

    public static String encode(long num) {

        if (num == 0)
            return "0";

        StringBuilder sb = new StringBuilder();

        while (num > 0) {
            int remainder = (int)(num % 62);

            sb.append(CHARS.charAt(remainder));

            num = num / 62;
        }

        return sb.reverse().toString();
    }

    public static void main(String[] args) {
        System.out.println(encode(125673));
    }
}
```

Output:

```text
wGZ
```

To decode Base62 back to decimal, multiply each digit by powers of 62:

For `wGZ`:

```text
w = 32
G = 42
Z = 61
```

Calculation:

```text
32×62² + 42×62¹ + 61×62⁰
=
32×3844 + 42×62 + 61
=
123008 + 2604 + 61
=
125673
```

The algorithm is the same idea as decimal → binary conversion:

```text
Decimal → Binary : divide by 2
Decimal → Hex    : divide by 16
Decimal → Base62 : divide by 62
```

The only difference is the set of symbols used.

That avoids collisions and scales well.

