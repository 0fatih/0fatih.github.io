---
layout: post
section-type: post
title: TR - Cache Poisoning
category: Web
tags: [ 'Web', 'Security' ]
---

Merhaba, sizlere bu yazıda açıklayacağım konunun ismi; Web Önbellek Zehirlemesi. Asıl konumuza gelmeden önce bazı temel bilgiler vermeliyim.

## Cache Memory (Önbellek) Nedir?

Her ne kadar birçok çeşidi olsa da (bilgisayar önbelliği, sunucu önbelleği, veritabanı önbelleği vs.), hepsinin amacı aynıdır; bizlere daha hızlı hizmet vermek. Önbelleklerin boyutları, diğer belleklere göre çok daha küçüktür. Çünkü, diğer belleklerden kat kat daha hızlı işlem yaparlar. Örneğin, kişisel bilgisayarlarımızdaki önbelleklerin boyutları MegaByte'larla ölçülür.

Web önbellekleri ise, bizlere hedef sitemizi getirmeye yardımcı olan, [CDN](https://www.cloudflare.com/learning/cdn/what-is-a-cdn/) gibi serverların önbellekleridir. Eğer, girmek istediğiniz bir web sunucusu size çok uzak ise, CDN'ler burada devreye girer. Dünya'da birçok yerde bulunan CDN'ler, belirli aralıklarla sizin ulaşmak istediğiniz web sitesini önbelleğine kaydeder (tabii o web sitesinin herhangi bir CDN sunucu hizmeti alıyorsa). Bunun amacı, tekrar tekrar web sitesi sunucusuna kadar gitmemize engel olmaktır.

![CDN](https://www.cloudflare.com/img/learning/cdn/what-is-a-cdn/what-is-a-cdn.png)

Siz bir siteye girmek isteğinizi gönderdiniz. Ama CDN sunuucusu, sizin istediğiniz siteyi önbelleğine almamış. Bu durumda, CDN web sunucusuna başvururak siteyi bize getirir. Ve bir sonraki istekte daha hızlı davranabilmek için, bu siteyi kendi önbelleğine kaydeder. Bunun sonucunda, bizden sonra herhangi birisi o siteye istek atarsa, çok daha hızlı bir şekilde ulaşması olasıdır.

## Web Önbellekleri Nasıl Çalışır?

Sunucu, kendisine gelen bir isteğin, önbelleğinde bulunup bulunmadığını nasıl anlar? Belki aklınıza, byte-byte karşılaştırmak gelmiş olabilir. Ancak, aynı siteye ulaşmak için, farklı iki bilgisayardan gönderilen HTTP istekleri, temel birkaç şeyin aynı olması ile dışında, muhtemelen farklı olacaktır. Örneğin, [User-Agent](https://www.howtogeek.com/114937/htg-explains-whats-a-browser-user-agent/) başlığı gibi. Bu sorunu aşmak için, anahtarlama sistemi ortaya çıkarılmıştır. Gelen istekte, birkaç adet başlığın, önbellekte bulunan isteğin başlıkları ile uyuşması durumunda, sunucu önbellekteki siteyi, istek yapan kişiye gönderecektir. Mesela şöyle bir istek olsun;


        GET /#blog HTTP/1.1
        Host: fatihfurkan.site
        User-Agent: Mozilla/5.0 … Firefox/57.0
        Accept-Language: tr
        Connection: close

Sunucu yapılandırmasına göre sadece, host başlığı ve istenilen dizin (/#blog) anahtarlandırılmış olsun. Bu durumda, sunucu iki anahtarı birbiri ile uyuşan isteklere, önbelleğindeki yanıtı döndürecektir. İşte sorun burada başlıyor. Peki ya anahtarlanmamış başlıklar ne olacak? Mesela, dili (language) değiştirerek bir istek atalım.

        GET /#blog HTTP/1.1
        Host: fatihfurkan.site
        User-Agent: Mozilla/5.0 … Firefox/57.0
        Accept-Language: tr
        Connection: close

Sunucu, yaptığı anahtar eşleştirmesinin pozitif sonuç üretmesinden ötürü, bize önceki isteğimize gönderdiği yanıtı döndürecektir. Ama biz parametre değiştirdik. Neden aynı yanıt geldi? Çünkü, sunucu dil parametresini algılamak için yapılandırılmadı. İşte Web Cache Poisoning açığının ortaya çıktığı kısım burası. Ayrıca şunu da belirtmeliyim ki, cache poisoning işlemi sadece aşağıda anlattığım şekilde değil, [HTTP Request Smuggling](https://fatihfurkan.site/2020/HTTP-Request-Smuggling/) ve [HTTP Response Splitting](https://fatihfurkan.site/2020/HTTP-Response-Splitting/) ile de yapılabilir.

## Web Önbellek Zehirlemesi

Şimdi, dil parametresi anahtarlanmamış. Bu yüzden sunucu, dil parametresini önemsemeden, diğer anahtarların uyuşması halinde, önbelleğindeki yanıtı gönderecektir. Peki, biz aşağıdaki gibi bir isteği, sunucu önbelleğine yerleştirebilirsek ne olur?

        GET /#blog HTTP/1.1
        Host: fatihfurkan.site
        User-Agent: Mozilla/5.0 … Firefox/57.0
        Accept-Language: <script>alert(document.domain)</script>
        Connection: close

Eğer sunucuya giden bir isteğin anahtarları, bizim isteğimizdeki anahtarlarla aynı ise; sunucu, önbelleğine yerleştirdiğimiz yanıtı döndürecektir. Dönen yanıtın içerisinde, istekte gönderdiğimiz dil parametresi de olacağı için, yazdığımız script, istek sahibinin bilgisayarında çalışacaktır.

İsteğimizi, sunucu önbelliğine nasıl yerleştireceğiz? Önbellekere alınan isteklerin sona erme süresi vardır. Bu süre bittikten sonra, sunucu önbelleğindeki isteği tazelemek için, kendisine gelen ilk istek için web sunucusuna başvuracaktır. Bizim amacımız ise, çok fazla istek atarak veya o süre bitimini yakalayarak isteğimizi, önbelleğe düşürmektir.

Tabii, burdan sonrası sizin hayagücünüze kalmış.

**Fatih Furkan Hatipoğlu**

&nbsp;

Bana bu kadar bilgi yetmez, daha fazlasını istiyorum diyenler için birkaç tavsiye kaynak:

[ PortSwigger - Pratical Web Cache Poisoning (Araştırma)](https://portswigger.net/research/practical-web-cache-poisoning)

[ PortSwigger - Web Cache Poisoning ](https://portswigger.net/web-security/web-cache-poisoning)
