# Request

While I use Burpsuite to learn about infosec and complete CTF, I found request that I want to keep for later inspection.

## Mozilla Firefox update?

```
POST /downloads?client=navclient-auto-ffox&appver=77.0&pver=2.2 HTTP/1.1
Host: shavar.services.mozilla.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:77.0) Gecko/20100101 Firefox/77.0
Accept: */*
Accept-Language: en,fr;q=0.8,fr-FR;q=0.5,en-US;q=0.3
Accept-Encoding: gzip, deflate
Content-Type: text/plain
Content-Length: 773
Connection: close
DNT: 1
Pragma: no-cache
Cache-Control: no-cache

ads-track-digest256;a:1591202430
allow-flashallow-digest256;a:1490633678
analytics-track-digest256;a:1591202430
base-cryptomining-track-digest256;a:1591202430
base-fingerprinting-track-digest256;a:1591202430
block-flash-digest256;a:1496263270
block-flashsubdoc-digest256;a:1512160865
content-track-digest256;a:1591202430
except-flash-digest256;a:1494877265
except-flashallow-digest256;a:1490633678
except-flashsubdoc-digest256;a:1517935265
google-trackwhite-digest256;a:1591202430
mozplugin-block-digest256;a:1471849627
mozstd-trackwhite-digest256;a:1591202430
social-track-digest256;a:1591202430
social-tracking-protection-facebook-digest256;a:1591202430
social-tracking-protection-linkedin-digest256;a:1591202430
social-tracking-protection-twitter-digest256;a:1591202430
```

followed by

```
GET /update/6/Firefox/77.0.1/20200602222727/WINNT_x86_64-msvc-x64/fr/release/Windows_NT%2010.0.0.0.19041.329%20(x64)/ISET:SSE4_2,MEM:16241/default/default/update.xml?force=1 HTTP/1.1
Host: aus5.mozilla.org
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:77.0) Gecko/20100101 Firefox/77.0
Accept: */*
Accept-Language: en,fr;q=0.8,fr-FR;q=0.5,en-US;q=0.3
Accept-Encoding: gzip, deflate
Cache-Control: no-cache
Pragma: no-cache
DNT: 1
Connection: close
```

Looks like FF is trying to download an update.

What is it about all the body fields?
