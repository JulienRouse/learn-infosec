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
