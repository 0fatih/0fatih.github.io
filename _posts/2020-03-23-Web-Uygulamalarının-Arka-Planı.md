---
layout: post
section-type: post
title: TR - Web Uygulamalarının Arka Planı
category: Network
tags: [ 'Network', 'Web' ]
---
Daha önce bir web sitesine girmeye çalışırken arka planda neler gerçekleştiğini hiç düşündünüz mü? Bu yazıda kendi bilgim dahilinde sizlere bu işlerin arka planından bahsedeceğim.

Öncelikle sunucu (server) ve istemci (client) kavramlarını açıklamakta fayda var. İstemci, internetteki son kullanıcı olarak bilinir. Telefonunuz, bilgisayarınız, internete bağlı kameranız birer istemci olarak adlandırılır. Demek oluyor ki, istemciye sıradan kullanıcı diyebiliriz. Sunucu ise, istemcinin isteklerine yanıt veren süper hızlı bilgisayardır. Yani, biz bir istek gönderdiğimizde gelen yanıtlar uzaydan veya buluttan değil, sadece bir başkasının bilgisayarından geliyor. İnternet dediğimiz şey, birbirine kablolarla bağlanmış bilgisayar sürüsüdür. Aşağıda, filmlerde çok gördüğümüz türden bir sunucu çiftliği (birçok sunucunun birleşmesiyle oluşan yapı) fotoğrafını görüyorsunuz.

![Server Farm](https://img.yle.fi/uutiset/ulkomaat/article7467687.ece/ALTERNATES/w960/129%20massadata%20datakeskus%20pilvipalvelu%20serveri "Örnek bir sunucu çiftliği")

Tarayıcınızı açıp URL kısmına "https://www.google.com" yazdınız ve google.com'un ana sayfası ekranınıza geldi. Peki bu nasıl oldu? Aslında bilgisayarınız girdiğiniz URL'in kimi temsil ettiğinin farkında bile değil. Çünkü bilgisayarlar birbirleri ile IP (Internet Protocol) vasıtasıyle haberleşir. URL'in kelimeler ve sayılarla temsil edilmesi tamamen insanın aklında kalması için geliştirilmiş bir yöntemdir. Bilgisayarın hedefe ulaşabilmesi için öncelikle bu URL'in kime ait olduğunu öğrenmesi gerekir. Bu kısımda DNS (Domain Name System) yardımcı olur.

![DNS](https://www.cloudflare.com/img/learning/dns/what-is-dns/dns-lookup-diagram.png "DNS nasıl çalışır")

İlk başta tarayıcınız yazdığınız URL'den www.google.com kısmını alarak bilgisayarınızın yapılandırılmış olduğu DNS'e bu alan adını gönderir. Alan adını alan DNS, öncelikle kendi kayıtlarını inceler; eğer aradığınız alan adı burada varsa direkt size aramanızın sonucunu gönderir, yoksa ilk önce Root Server'a bu domain hakkında sorgu yapar. Root Server, DNS'e başvurması gereken TLD (Top Level Domain) Server'ı gösterir (bu durumda yönlendirileceğimiz TLD Server URL'imizin uzantısı .com olduğu için, COM TLD Server'ı olacaktır). TLD Server ise kayıtlarına bakacak ve DNS Server'ı gerekli Name Server'a yönlendirecek. Name Server da kendi içerisinde gerekli sorgulamayı yapıp DNS Server'a hedef sitenin IP'sini iletecek. Son olarak DNS Server elde ettiği IP'yi bize gönderecek ve bu alan adı için kendisine gelecek bir sonraki isteği daha hızlı yanıtlayabilmek amacıyla, bahsi geçen alan adını kendi kayıtlarına ekleyecektir. Bilgisayarımız kavuştuğu hedef sunucu IP'si ile TCP bağlantısı gerçekleştirecektir.

Bağlantı sağlandıktan sonra; sunucunun güvenlik duvarı, gönderdiğimiz isteği kendi yapılandırmasına göre belirli birkaç filtreleme işlemine maruz bırakır. Eğer isteğimiz bu filtrelemelerden başarılı bir şekilde geçer ise, sunucu isteğimizi işler ve gerekli yanıtı döndürür. Web güvenliği ile uğraşan bizlerin genel amacı; sunucu güvenlik duvarını atlatarak beklenmedik istekler gönderip, beklenmedik yanıtlar almaktır.

![WAF](https://www.cloudflare.com/img/learning/ddos/glossary/waf/waf.png "Güvenlik duvarı gösterimi")

Tabii giden istekler ve gelen yanıtlar sadece bir takım kodlardan oluşmaktadır. Burada da yine tarayıcımız yardımımıza koşarak gelen kodları işler ve bu kodları asıl amaçlanan şekline büründürür. Elbette web güvenliği, sadece bu olaylardan ibaret değil. Bu aradaki iletişimin gizliliği de oldukça önemlidir. Kimse Instagram şifresinin bir başkası tarafından kolayca elde edilmesini istemez değil mi? Bu gibi bilgi sızıntılarından korunmamız için kullanılan yöntemlerden birisi; SSL sertifikasıdır. Eğer bir site SSL sertifikasına sahip ise, istemci ve sunucu arasındaki bağlantı için son derece güvenli diyebiliriz. Girdiğiniz veya gireceğiniz sitenin bu sertifikaya sahip olup olmadığını sadece URL'ine bakarak anlayabilirsiniz. Eğer site "HTTPS" ile başlıyor ise SSL sertifikasına sahip, "HTTP" ile başlıyor ise sahip değil demektir. Yani HTTPS ile başlayan bir siteye girdiğinizde "Bu site güvenli" ibaresi, yalnızca sunucu ile aranızdaki bağlantının güvenli olduğunu belirtir. Mesela hepimizin bildiği Çiftlik Bankası'nın web sitesi SSL sertifikasına sahipti. Hangi siteye güvenip güvenmeyeceğinize daha iyi kararlar vermelisiniz.


Yukarıdaki olaylar ve daha fazlası neredeyse bütün isteklerimizde gerçekleşiyor. İnsanoğlu sonsuz bilgi kaynağına ve o bilgiye ulaşmada inanılmaz bir hıza sahip. Peki biz bu gücü kendimize ne kadar fayda sağlayacak biçimde kullanıyoruz?


**Fatih Furkan Hatipoğlu**

&nbsp;

Bana bu kadar bilgi yetmez, daha fazlasını istiyorum diyenler için birkaç tavsiye kaynak:


[Quora'da "Web uygulamaları nasıl çalışır adlı" başlık.](https://www.quora.com/How-does-a-web-application-work)

[Cloudflare'de TCP/IP hakkında bilgi.](https://www.cloudflare.com/learning/ddos/glossary/tcp-ip/)

[DNS Server'ın nasıl çalıştığını anlatan güzel bir video.](https://www.youtube.com/watch?v=mpQZVYPuDGU)

[IP adresleri ve DNS hakkında yazılmış bir yazı.](https://www.howtogeek.com/341307/how-do-ip-addresses-work/)
