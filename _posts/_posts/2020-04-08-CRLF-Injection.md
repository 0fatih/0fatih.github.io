---
layout: post
section-type: post
title: TR - CRLF Injection
category: Web
tags: [ 'Web', 'Security' ]
---

CRLF, birçok protokolde kullanılan bir yöntemdir. Kullanım amacı genellikle; bir satırı bitirip yeni bir satırın başladığını belirtmektir. Web Uygulamalarında ise, HTTP isteklerinde kullanılır. Bir isteğin header kısmı dediğimiz bölüm ile HTML kodlarını ayırmada ve satır sonlarını belirtmekte kullanırız. Encode edilmiş hali "%0d%0a" iken, encode edilmemiş hali "\r\n" dir. CRLF'lerin kullanımına örnek bir istek şu şekilde olabilir:

        GET / HTTP/1.1%0d%0a
        Host: fatihfurkan.site%0d%0a

CRLF'den bahsettiğimize göre, CRLF Injection saldırısına geçebiliriz.

CRLF karakterlerin, bölümleri ve satırları ayırma işlemlerinde kullanıldığını söyledik. Biraz kurnazlık yaparak gönderdiğimiz isteklerdeki bu karakterleri amacına yönelik olmayan bir şekilde bölümleri ayırabiliriz.

CRLF Injection kullanarak; [HTTP Request Smuggling](https://fatihfurkan.site/2020/HTTP-Request-Smuggling/), [HTTP Response Splitting](https://fatihfurkan.site/2020/HTTP-Response-Splitting/) gibi açıklardan faydalanabiliriz.

Bu konu pek uzun olmadı biliyorum ancak uzatılması gereksiz bir konu. Mantığını anlayıp sebep olacağı açıklara yönelmenin daha mantıklı olacağını düşünüyorum.

**Fatih Furkan Hatipoğlu**

&nbsp;

Bana bu kadar bilgi yetmez, daha fazlasını istiyorum diyenler için birkaç tavsiye kaynak:

[Netsparket - CRLF Injection](https://www.netsparker.com.tr/blog/web-guvenligi/crlf-injection-ve-http-response-splitting-zafiyeti/)

[Acunetix - CRLF Injection](https://www.acunetix.com/websitesecurity/crlf-injection/)
