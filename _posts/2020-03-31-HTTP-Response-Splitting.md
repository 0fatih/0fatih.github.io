---
layout: post
section-type: post
title: TR - HTTP Response Splitting
category: Web
tags: [ 'Web', 'Security' ]
---

Bu yazıda işleyeceğimiz konu: HTTP İstek Bölme saldırısı. Sıradan basit bir HTTP isteği, aşağıdaki şekilde olacaktır.


          GET / HTTP/1.1
          Host: fatihfurkan.site

Burada odaklanmamız gereken kısım, satırların birbirinden nasıl ayrıldığı. Bize görünmeyen kısımda CR/LF (Carriage Return/Line Feed) karakterleri vardır. Bu karakterler HTTP'de, o satırın bitip, yeni bir satırın başladığını belirtir. Eğer, bu karakterlerden iki tane varsa (%0d%0a%0d%0a), headerın bitip, bodynin başladığını anlarız.  Normal gösterimleri "\r\n" dir. Ancak HTTP istek ve yanıtlarında URL Encode biçimi kullanıldığı için, bu karakterleri daha çok %0d%0a şeklinde göreceğiz.


Birkaç temel bilgiden sonra, asıl konumuza dönebiliriz. HTTP Response Splitting açığından faydalanabilmemiz için, siteye verdiğimiz herhangi bir girdinin, headerda gönderilmesi gerekir. Örneğin; ismimizi isteyen bir siteye ismimizi girdiğimizde, X-For-Name adlı bir başlık ile göndersin. Yani isteğimiz,

          GET / HTTP/1.1
          Host: fatihfurkan.site
          X-For-Name: Ceren

şeklinde gözüksün. İsteğimizde, Ceren'in yanına, %0d%0aContent- Length:%200%0d%0aHTTP/1.1%20200%20OK%0d%0aContent-Type:%20text/html%0d%0aContent-Length:%2019%0d%0a%0d%0a<html>Attack</html>  ekleyerek bir saldırı gerçekleştirmiş oluruz. Tabii burada hedef sitenin bahsi geçen açığa karşı zaaflı olduğunu varsayıyoruz.

Yazdığımız şeyi parça parça ele alalım. Öncelikle isteğimizdeki her %0d%0a'yı \r\n'e dönüştürerek, okunabilirliği arttıralım: \r\nContent-Length:%200\r\n\r\nHTTP/1.1%20200%20OK\r\nContent-Type:%20text/html\r\nContent-Length:%2019\r\n\r\n<html>Attack</html>

Daha güzel oldu. Bir de \r\n'lerin görevi olan, satır atlamayı ve URL Encode'da %20'nin karşılığı olan boşluk ifadesini de yerlerine koyarak aşağıdaki sonucu elde edelim.

          GET / HTTP/1.1
          Host: fatihfurkan.site
          X-For-Name: Ceren
          Content-Length: 0

          HTTP/1.1 200 OK
          Content-Type: text/html
          Content-Length: 19

          <html>Attack</html>

O kadar da karmaşık bir şey değilmiş değil mi? Şimdi, bu isteğin ne yaptığını satır satır inceleyelim.

1) GET ile kök dizinini istediğimizi ve bu isteği HTTP protokolünün 1.1 versiyonu üstünden istediğimizi,

2) Bu isteğin, fatihfurkan.site adlı siteye gönderildiğini,

3) İsmimizi Ceren olarak girdiğimizi,

4) İçerik uzunluğunun 0 olduğunu (çünkü isteğimizde, headerdaki ismimiz hariç hiçbir içerik göndermiyoruz),

5) Header kısmının bitip, body kısmının başladığını belirtiyoruz.

6) Burada, içerik uzunluğunu 0 girmemize rağmen bir şeyler yazıyoruz. Bunun sebebi, sunucunun bu kısmı, isteğimizin bir parçası olarak değil de başka bir istek/yanıt olarak görmesini sağlamak. Eğer amacımıza ulaşmışsak, sunucu, metnin yapısından ötürü bunu bir yanıt olarak algılayacaktır. Yanıtımızın ilk satırında bu yanıtın, HTTP 1.1 versiyonu ile gönderildiğini ve başarılı bir istek olduğunu görebiliriz.

7) Yanıtımızın içerik türünün, text/html olduğunu,

8) Yanıtımızın içerik uzunluğunun 19 olduğunu,

9) Header kısmının bitip, body kısmının başladığını belirtiyoruz.

10) Buraya ne istersek yazabiliriz. Çünkü bu kısım, saldırı başarılı olduğunda çalışacak kısımdır.

İsteğimizi açıkladığımıza göre, bunun çalışma mantığına geçelim.


Biz, içerik uzunluğu 0 olan bir istek gönderiyoruz ve yanıtımızı alıyoruz. Bir istek - bir sonuç değil mi? Peki ya isteğimizle beraber gönderdiğimiz yanıta ne oldu? Sunucu, bir istek - bir sonuç politikasına göre, o yanıtı bir isteğe göndermeli. Sunucu bu durumda ilginçtir ki, kendisine gelen ilk isteğe bu yanıtı döndürecektir. Bunun sonucunda, bizden sonraki isteği gönderen kişi, asıl istediği yanıtı değil de bizim gönderdiğimizi alacaktır. Biraz daha anlaşılır olması için şöyle açıklayayım.


a) Saldırgan, içerisinde CRLF karakterleri ile ayrılmış bir istek ve bir yanıt bulunan, isteği gönderir.


          GET / HTTP/1.1
          Host: fatihfurkan.site
          X-For-Name: Ceren
          Content-Length: 0

          HTTP/1.1 200 OK
          Content-Type: text/html
          Content-Length: 19

          <html>Attack</html>


b) Bu istek fatihfurkan.site aldı siteye gidecektir. Ama önemli olan; bunu bir proxy serverdan geçerek yapmasıdır. İlk istek proxy server tarafından, aradığı yanıtı alır. Ancak göndermiş olduğumuz yanıt, herhangi bir istek ile eşleşmediği için askıda kalır.


c) İlk isteği gönderdikten hemen sonra, saldırganın yeni bir istek gönderdiğini varsayalım.

          GET /blog HTTP/1.1
          Host: fatihfurkan.site

d) Proxy server /blog için gelen isteği gördüğünde, bu isteğe karşılık, bizim gönderdiğimiz yanıtı verir. Peki bu sadece saldırgan için mi geçerli? Elbette hayır! Yanıtımızı sunucuda askı durumuna düşürdükten sonra, sunucu kendisine gelen ilk isteğe, kimin gönderdiğine bakmaksızın, bizim yanıtımızı döndürecektir. Ve eğer biz bu isteğimizi proxy server'ın ön belleğine girdirebilirsek, ön bellek süresi bitene kadar o siteye girmek için gelen her isteğe bizim yanıtımız dönecektir.





















Her ne kadar heyecan verici bir saldırı olsa da, sanırım kendimizi kurban durumunda düşündüğümüzde, o heyecanın yerini endişe alacaktır.




**Fatih Furkan Hatipoğlu**

&nbsp;

Bana bu kadar bilgi yetmez, daha fazlasını istiyorum diyenler için birkaç tavsiye kaynak:

[ Netsparker - CRLF Injection ve HTTP Response Splitting Zafiyeti ](https://www.netsparker.com.tr/blog/web-guvenligi/crlf-injection-ve-http-response-splitting-zafiyeti/)

[ Montana Üniversitesi - HTTP Response Splitting (slayt) ](https://www.cs.montana.edu/courses/csci476/topics/http_response_splitting.pdf)

[ InfosecInstitute - HTTP Response Splitting Attack ](https://resources.infosecinstitute.com/http-response-splitting-attack/)
