---
layout: post
title: 'OpenSSL: mechanizm SNI oraz rozszerzenie SAN'
date: 2015-09-11 07:51:13
categories: [publications]
tags: [PL, security, openssl, sni]
comments: false
favorite: false
seo:
  date_modified: 2020-02-22 09:47:41 +0100
---

Za pomocą biblioteki `openssl` można testować praktycznie **każdą** usługę opartą na protokołach SSL/TLS. Po nawiązaniu połączenia można nimi sterować, stosując komendy/wiadomości specyficzne dla każdego protokołu warstwy aplikacji.

W tym poście przedstawię, czym jest mechanizm **SNI** a także do czego potrzebne jest rozszerzenie **SAN**. Dodatkowo zaprezentuję, w jaki sposób nawiązywać połączenia z wykorzystaniem tego pierwszego.

# Czym jest rozszerzenie SNI?

**SNI** (ang. _Server Name Indication_) jest rozszerzeniem protokołu TLS, które umożliwia serwerom używanie wielu certyfikatów na jednym adresie IP.

Ponieważ liczba dostępnych adresów IPv4 stale maleje, pozostałe mogą być alokowane bardziej efektywnie. W większości przypadków można uruchomić aplikację z obsługą protokołu SSL/TLS bez konieczności zakupu dodatkowego adresu IP.

Zgodnie z [RFC 6066 - Server Name Indication](https://tools.ietf.org/html/rfc6066#page-6) rozszerzenie to pozwala klientowi na wskazanie nazwy hosta, z którym stara się nawiązać połączenie na początku procesu uzgadniania sesji SSL/TLS. Jak zostało powiedziane wyżej — pozwala to serwerowi na przedstawienie wielu certyfikatów na tym samym gnieździe (adresie IP i numerze portu), dzięki czemu możliwe jest korzystanie z tego samego adresu IP przez wiele witryn wykorzystujących protokół HTTPS.

  > Żądana nazwa hosta (domeny), którą ustala klient podczas połączenia, nie jest szyfrowana. W związku z tym, podsłuchując ruch można zobaczyć, z którą witryną nawiązywane będzie połączenie.

## Proces nawiązywania połączenia

Podczas nawiązywania połączenia TLS klient wysyła żądanie z prośbą o certyfikat serwera. Gdy serwer odsyła certyfikat do klienta, ten sprawdza go i porównuje nazwę z nazwami zawartymi w certyfikacie (pola **CN** oraz **SAN**).

Jeżeli domena zostanie znaleziona, połączenie odbywa się w normalny sposób (standardowa sesja SSL/TLS). Jeżeli domena nie zostanie znaleziona, oprogramowanie klienta powinno wyświetlić ostrzeżenie zaś połączenie powinno zostać przerwane.

  > Niedopasowanie nazw może oznaczać próbę ataku typu **MITM**. Niektóre z aplikacji (np. przeglądarki internetowe) pozwalają na ominięcie ostrzeżenia w celu kontynuowania połączenia — przerzucając tym samym odpowiedzialność na użytkownika, który często jest nieświadomy czyhających zagrożeń.

## Szczegóły połączenia

Gdy klient (np. przeglądarka) nawiązuje połączenie, ustawia specjalny nagłówek HTTP (nagłówek `Host`) określający, do której witryny klient próbuje uzyskać dostęp.

Serwer dopasowuje podaną zawartość nagłówka do domeny w swojej konfiguracji i odpowiada klientowi np. wyświetlając odpowiednią zawartość lub kierując ruch dalej i w konsekwencji także serwując odpowiednią treść.

Podanej techniki nie można zastosować do protokołu HTTPS ponieważ nagłówek ten jest wysyłany dopiero po zakończeniu uzgadniania sesji TLS.

Tym samym powstaje następujący problem:

- serwer potrzebuje nagłówków HTTP w celu określenia, która witryna (domena) powinna być dostarczona do klienta
- nie może jednak uzyskać tych nagłówków bez wcześniejszego uzgodnienia sesji TLS, ponieważ wcześniej wymagane jest dostarczenie samych certyfikatów

Dlatego do tej pory (przed wprowadzeniem rozszerzenia **SNI**) jedynym sposobem dostarczania różnych certyfikatów było hostowanie jednej domeny na jednym adresie IP. Na podstawie adresu IP (do którego doszło żądanie o zaserwowanie treści) oraz przypisanej do niego domeny serwer wybierał odpowiedni certyfikat.

Pierwszym rozwiązaniem tego problemu w przypadku ruchu HTTPS jest przejście na protokół IPv6.

  > Nie stanowi to oczywiście problemu w przypadku protokołu HTTP, ponieważ jak tylko połączenie TCP zostanie otwarte, klient wskaże, do której strony internetowej próbuje dotrzeć w żądaniu.

Rozwiązaniem tymczasowym jest właśnie wykorzystanie mechanizmu **SNI**, który wstawia żądaną nazwę hosta (domeny, adresu internetowego) w ramach uzgadniania ruchu TLS — przeglądarka wysyła tę nazwę w komunikacie `Client Hello` pozwalając serwerowi na określenie najbardziej odpowiedniego certyfikatu.

**SNI** dodaje nazwę domeny do procesu uzgadniania TLS, dzięki czemu klient dotrze do właściwej domeny i otrzyma prawidłowy certyfikat SSL, tym samym będzie możliwe normalne kontynuowanie sesji TLS oraz przejście poziom wyżej, do wymiany danych na poziomie protokołu HTTP z pełnym i bezpiecznym wykorzystaniem TLS.

Spójrz na poniższy obrazek:

<img src="/assets/img/posts/sni_tls.jpg" align="center" title="sni_tls preview">

Dzięki rozszerzeniu **SNI** serwer może bezpiecznie "trzymać" wiele certyfikatów używając pojedynczego adresu IP.

## SNI a Subject Alternative Name (SAN)

**SAN** i **SNI** to dwie całkowicie różne rzeczy, których jedyną cechą wspólną jest to, że wymagają obsługi po stronie klienta.

**SAN** (ang. _Subject Alternative Name_) jest częścią specyfikacji X509, w której certyfikat zawiera pole z listą nazw alternatywnych (np. dodatkowych domen). Są one tak samo ważne jak standardowe pole `CN`, które notabene jest ignorowane przez większość klientów, jeżeli wskazano pole `SAN`.

Rozszerzenie to wykorzystuje się głównie, aby rozwiązać problem stosowania jednego certyfikatu dla jednej domeny, co jest oczywiście bardzo niepraktyczne i zwiększa znacząco koszty utrzymania. Dzięki temu serwer nie musi przedstawiać innego certyfikatu dla każdej domeny, tylko za pomocą jednego certyfikatu, w którym zawarte jest jedno pole `CN` (dla domeny głównej) oraz rozszerzenie `SAN` (w którym określamy domenę główną oraz dodatkowe domeny) umożliwia obsługę wielu domen w jednym certyfikacie.

Oto przykład:

```bash
issuer: DigiCert SHA2 Secure Server CA (DigiCert Inc)
owner: Lucas Garron
cn: *.badssl.com  <= POLE CN
san: *.badssl.com badssl.com  <= POLE SAN
sni: not match
validity: match
chain of trust:
 └─0:*.badssl.com ★ ✓
   ├   DigiCert SHA2 Secure Server CA
   └─1:DigiCert SHA2 Secure Server CA ✓
     ├   DigiCert Global Root CA
     └─2:DigiCert Global Root CA ✓ ⊙
       └ DigiCert Global Root CA
verification: ok
```

## SNI a klient

Jak już powiedziałem na wstępie, `SNI` to rozszerzenie protokołu TLS, które jest swego rodzaju odpowiednikiem nagłówka `Host` protokołu HTTP. Pozwala serwerowi wybrać odpowiedni certyfikat, który ma zostać przedstawiony klientowi, bez ograniczenia korzystania z oddzielnych adresów IP po stronie serwera (upraszcza to posiadanie wielu certyfikatów).

Poprawne działanie tego rozszerzenia zależy od:

- poprawnej obsługi po stronie serwera (w większości przypadków każdy serwer obsługuje ten mechanizm poprawnie)
- poprawnej obsługi po stronie klienta (w większości oprogramowania funkcja ta jest zaimplementowana)

Zdecydowana większość przeglądarek i systemów operacyjnych obsługuje **SNI**. W przypadku nieaktualnych klientów, którzy nie wspierają tego rozszerzenia, użytkownik prawdopodobnie nie będzie mógł uzyskać dostępu do niektórych witryn, a przeglądarka zwróci komunikat "_Połączenie nie jest prywatne_".

# Testowanie połączenia

## OpenSSL

Połączenie do zdalnej usługi z ustaloną nazwą domeny (rozszerzenie **SNI**):

```bash
echo | openssl s_client -showcerts -servername www.example.com -connect example.com:443
```

Jeżeli chcemy połączyć się bez włączonego **SNI**:

```bash
echo | openssl s_client -showcerts -connect example.com:443
```

## gnutls-cli

Wykorzystujemy rozszerzenie **SNI** (domyślnie):

```bash
gnutls-cli -p 443 www.example.com
```

Bez wykorzystania rozszerzenia **SNI**:

```bash
gnutls-cli --disable-sni -p 443 www.example.com
```
