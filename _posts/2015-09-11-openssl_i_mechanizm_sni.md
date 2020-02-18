---
layout: post
title: OpenSSL i mechanizm SNI
date: 2015-09-11 07:51:13
categories: [publications]
tags: [PL, security, openssl]
comments: false
favorite: false
seo:
  date_modified: 2020-02-18 21:56:52 +0100
---

Za pomocą biblioteki `openssl` można testować praktycznie **każdą** usługę opartą na protokołach SSL/TLS. Po nawiązaniu połączenia można nimi sterować, stosując komendy/wiadomości specyficzne dla każdego protokołu warstwy aplikacji. W tym poście przedstawię koncepcję mechanizmu **SNI** oraz w jaki sposób nawiązywać połączenia z wykorzystaniem tego rozszerzenia lub bez niego.

# Czym jest rozszerzenie SNI?

**SNI** (ang. _Server Name Indication_) jest rozszerzeniem protokołu TLS, które umożliwia serwerom używanie wielu certyfikatów na jednym adresie IP.

Ponieważ liczba dostępnych adresów IP stale maleje, pozostałe mogą być alokowane bardziej efektywnie. W większości przypadków można uruchomić aplikację z obsługą protokołu SSL/TLS bez konieczności zakupu określonego adresu IP.

Zgodnie z [RFC 6066 - Server Name Indication](https://tools.ietf.org/html/rfc6066#page-6) rozszerzenie to pozwala klientowi na wskazanie nazwy hosta, z którym klient stara się nawiązać połączenie na początku procesu uzgadniania. Jak zostało powiedziane wyżej — pozwala to serwerowi na przedstawienie wielu certyfikatów na tym samym adresie IP i numerze portu, a tym samym umożliwia korzystanie z tego samego adresu IP przez wiele witryn wykorzystujących protokół HTTPS.

  > Żądana nazwa hosta (domeny), którą ustala klient podczas połączenia, nie jest szyfrowana. Dzięki temu podsłuchując można zobaczyć, z którą witryną nawiązywane będzie połączenie.

## Proces nawiązywania połączenia

Bez wdawania się w szczegóły, podczas nawiązywania połączenia TLS klient wysyła żądanie z prośbą o certyfikat serwera. Gdy serwer wysyła certyfikat, klient sprawdza go i porównuje nazwę z nazwami zawartymi w certyfikacie (pola **CN** oraz **SAN**).

Jeżeli domena zostanie znaleziona, połączenie odbywa się w normalny sposób (standardowa sesja SSL/TLS). Jeżeli domena nie zostanie znaleziona, oprogramowanie klienta powinno wyświetlić ostrzeżenie zaś połączenie powinno zostać przerwane.

  > Niedopasowanie nazw może oznaczać próbę ataku typu **MITM**. Niektóre z aplikacji (np. przeglądarki internetowe) pozwalają na ominięcie ostrzeżenia w celu kontynuowania połączenia — przerzucają tym samym odpowiedzialność na użytkownika, który często jest nieświadomy czyhających zagrożeń.

## Szczegóły połączenia

Gdy klient (np. przeglądarka) nawiązuje połączenie, ustawia specjalny nagłówek HTTP (nagłówek `Host`) określający, do której witryny klient próbuje uzyskać dostęp.

Serwer dopasowuje podaną zawartość nagłówka do domeny i odpowiada klientowi np. wyświetlając odpowiednią zawartość lub kierując ruch dalej. Podanej techniki nie można zastosować do protokołu HTTPS ponieważ nagłówek ten jest wysyłany dopiero po zakończeniu uzgadniania sesji TLS.

Tym samym powstaje następujący problem:

- serwer potrzebuje nagłówków HTTP w celu określenia, która witryna (domena) powinna być dostarczona do klienta
- nie może jednak uzyskać tych nagłówków bez wcześniejszego uzgodnienia sesji TLS, ponieważ wcześniej wymagane jest dostarczenie samych certyfikatów

Dlatego do tej pory (przed wprowadzeniem rozszerzenia SNI) jedynym sposobem dostarczania różnych certyfikatów było hostowanie jednej domeny na jednym adresie IP.

Na podstawie adresu IP (dla którego doszło żądanie o zaserwowanie treści) oraz przypisanej do niego domeny serwer wybierał odpowiedni certyfikat. Pierwszym rozwiązaniem tego problemu w przypadku ruchu HTTPS jest przejście na protokół IPv6.

Rozwiązaniem tymczasowym jest właśnie wykorzystanie mechanizmu **SNI**, który wstawia żądaną nazwę hosta (domeny, adresu internetowego) w ramach uzgadniania ruchu TLS — przeglądarka wysyła tą nazwę w komunikacie `Client Hello` pozwalając serwerowi określenie najbardziej odpowiedniego certyfikatu.

Dzięki rozszerzeniu **SNI** serwer może bezpiecznie "trzymać" wiele certyfikatów używając pojedynczego adresu IP.

## SNI a Subject Alternative Name (SAN)

**SAN** (ang. _Subject Alternative Name_) jest częścią specyfikacji X509, w której certyfikat zawiera pole z listą nazw alternatywnych (np. dodatkowych domen), które są tak samo ważne jak standardowe pole `CN`, które notabene jest ignorowane przez większość klientów, jeżeli wskazano pole `SAN`.

Rozszerzenie to wykorzystuje się głównie, aby rozwiązać problem stosowania jednego certyfikatu dla jednej domeny, co jest oczywiście bardzo niepraktyczne i zwiększa znacząco koszty utrzymania. Dzięki temu serwer nie musi przedstawiać innego certyfikatu dla każdej domeny, tylko za pomocą jednego certyfikatu, w którym zawarte jest jedno pole `CN` (dla domeny głównej) oraz rozszerzenie `SAN` (w którym określamy domenę główną oraz dodatkowe domeny) umożliwia przechowywanie wielu domen w jednym certyfikacie.

Oto przykład:

```bash
ssl: on, version(TLSv1.2), cipher(ECDHE-RSA-AES128-GCM-SHA256), temp_key(ECDH,P-256,256bits)
public-key(2048 bit), signature(sha256WithRSAEncryption)
date: Mar 18 00:00:00 2017 GMT / Mar 25 12:00:00 2020 GMT (37 days to expired)
issuer: DigiCert SHA2 Secure Server CA (DigiCert Inc)
owner: Lucas Garron
cn: *.badssl.com
san: *.badssl.com badssl.com
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

Jak już powiedziałem na wstępie, `SNI` to rozszerzenie protokołu TLS, które jest swego rodzaju protokołem TLS równoważnym nagłówkowi hosta HTTP, czyli pozwala serwerowi wybrać odpowiedni certyfikat, który ma zostać przedstawiony klientowi, bez ograniczenia korzystania z oddzielnych adresów IP po stronie serwera (upraszcza to posiadanie wielu certyfikatów).

**SAN** i **SNI** to dwie całkowicie różne rzeczy, których jedyną cechą wspólną jest to, że wymagają obsługi po stronie klienta.

## SNI a klient

Poprawne działanie rozszerzenia **SNI** zależy od:

- poprawnej obsługi po stronie serwera (w większości przypadków każdy serwer obsługuje ten mechanizm poprawnie)
- poprawnej obsługi po stronie klienta (w większości oprogramowania funkcja ta jest zaimplementowana)

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
