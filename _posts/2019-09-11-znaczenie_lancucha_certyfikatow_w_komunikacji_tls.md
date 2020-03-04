---
layout: post
title: Znaczenie łańcucha certyfikatów w komunikacji TLS
date: 2019-09-11 01:32:08
categories: [publications]
tags: [PL, ssl, tls, certificate, chain-of-trust]
comments: false
favorite: false
seo:
  date_modified: 2020-02-23 11:01:19 +0100
---

Kluczową częścią każdego procesu uwierzytelniania opartego na certyfikatach jest walidacja łańcucha certyfikatów lub inaczej łańcucha zaufania (ang. _chain of trust_).

Łańcuch zaufania to połączona ścieżka certyfikacji (uporządkowana lista certyfikatów), która zawiera certyfikat użytkownika końcowego i certyfikaty pośrednie (które reprezentują pośrednie urządy certyfikacji). W skład łańcucha certyfikatów wchodzi także certyfikat głównego urzędu certyfikacji (CA), który działa jako kotwica zaufania (ang. _trust anchor_).

Kotwica zaufania to certyfikat (a ściślej publiczny klucz weryfikacyjny urzędu certyfikacji), któremu ufasz, ponieważ został Ci dostarczony w drodze pewnej wiarygodnej procedury. Jest on używany przez stronę ufającą jako punkt wyjścia do sprawdzania poprawności łańcucha.

  > W [RFC 5280](https://tools.ietf.org/html/rfc5280) łańcuch certyfikatów lub łańcuch zaufania jest zdefiniowany jako tzw. ścieżka certyfikacji (ang. _certification path_).

Jeśli system nie posiada łańcucha certyfikatów lub jeśli łańcuch jest przerwany (np. brakuje certyfikatów pośrednich), klient nie może sprawdzić, czy certyfikat końcowy jest ważny. W związku z tym certyfikat taki traci wszelką użyteczność wraz z tzw. wskaźnikiem zaufania (ang. _metric of trust_).

# Czym jest certyfikat klucza publicznego?

Certyfikat klucza publicznego to podpisana instrukcja, która służy do ustanowienia powiązania między tożsamością a kluczem publicznym. Pozwala on „udowodnić”, że jego posiadacz ma prawo dostępu do danych zasobów (udowadnia własności do klucza publicznego). Dzięki takiemu certyfikatowi można jednoznacznie zidentyfikować pewną jednostkę oraz stwierdzić, czy jest ona rzeczywiście tą, za którą się podaje.

Certyfikat klucza publicznego zawiera cztery istotne informacje:

- klucz publiczny podmiotu,
- opis tożsamości podmiotu,
- podpis cyfrowy złożony przez zaufaną trzecią stronę na dwóch powyższych strukturach
- zdefiniowany czas życia, określony w treści certyfikatu

Podmiot, który poręczy za to powiązanie i podpisze certyfikat, jest wystawcą certyfikatu, a tożsamość, za pomocą której klucz publiczny jest potwierdzony, jest przedmiotem certyfikatu. W celu powiązania tożsamości i klucza publicznego wykorzystywany jest właśnie łańcuch certyfikatów.

Certyfikat serwera wraz z łańcuchem nie jest przeznaczony dla serwera. Serwer nie ma zastosowania do własnego certyfikatu. Certyfikaty są zawsze dla innych podmiotów (tutaj klientów). Serwer używa klucza prywatnego (który odpowiada kluczowi publicznemu w certyfikacie) do deszyfrowania wiadomości zaszyfrowanych przez klienta kluczem publicznym. W szczególności, serwer nie musi ufać własnemu certyfikatowi ani żadnemu urzędowi certyfikacji, który go wydał.

# Co wchodzi w skład poprawnego łańcucha certyfikatów?

Łańcuch certyfikatów składa się ze wszystkich certyfikatów potrzebnych do weryfikacji podmiotu określonego w certyfikacie końcowym. W praktyce obejmuje to certyfikat serwera, certyfikaty pośrednie urzędów certyfikacji oraz certyfikat głównego urzędu certyfikacji — zaufany przez wszystkie strony w łańcuchu. Każdy pośredni urząd certyfikacji posiada certyfikat wydany przez urząd certyfikacji jeden poziom nad nim w hierarchii zaufania.

<p align="center">
  <img src="/assets/img/posts/chain_of_trust.png">
</p>

Przeglądarki oraz systemy operacyjne zawierają listę zaufanych urzędów certyfikacji. Te wstępnie zainstalowane certyfikaty służą jako kotwice zaufania, z których można czerpać dalsze zaufanie (mam nadzieję, że jest to odpowiednie określenie).

Podczas odwiedzania witryny HTTPS przeglądarka sprawdza, czy łańcuch zaufania prezentowany przez serwer podczas uzgadniania TLS kończy się na jednym z lokalnie zaufanych certyfikatów głównych.

## Dlaczego łańcuch certyfikatów nie powinien zawierać certyfikatu głównego?

Zgodnie ze standardem TLS łańcuch może zawierać certyfikat główny lub nie — jednak klient nie potrzebuje tego certyfikatu, ponieważ już go ma.

Serwer zawsze wysyła łańcuch, ale nigdy nie powinien prezentować łańcuchów certyfikatów zawierających kotwicę zaufania, która jest certyfikatem głównego urzędu certyfikacji, ponieważ certyfikat główny jest bezużyteczny do celów sprawdzania poprawności.

I rzeczywiście, jeśli klient nie ma jeszcze certyfikatu głównego, wówczas otrzymanie go z serwera nie pomogłoby, ponieważ takiemu certyfikatowi można zaufać tylko wtedy, jeśli zostanie dostarczony z zaufanego źródła (tj. lokalnego magazynu certyfikatów).

Co więcej, obecność kotwicy zaufania na ścieżce certyfikacji może mieć negatywny wpływ na wydajność podczas nawiązywania połączeń za pomocą protokołu SSL/TLS, ponieważ certyfikat główny jest „pobierany” przy każdym uzgadnianiu połączenia między klientem a serwerem. Jego brak, zmniejsza również zużycie pamięci po stronie serwera dla parametrów sesji TLS.

Zgodnie z zaleceniami, łańcuch certyfikatów powinien zawierać tylko klucz publiczny certyfikatu końcowego i klucze publiczne wszelkich pośrednich urzędów certyfikacji. Przeglądarki będą ufać tylko tym certyfikatom, które przekształcają się w certyfikaty główne, które są już w magazynie zaufanych certyfikatów, ignorując tym samym certyfikat główny wysłany w łańcuchu certyfikatów (w przeciwnym razie każdy mógłby wysłać dowolny certyfikat główny).

## Jaki jest cel stosowania certyfikatów pośrednich?

Ciekawe pytanie, ponieważ większość dzisiejszych certyfikatów użytkowników końcowych jest wydawana przez pośrednie urzędy certyfikacji, a nie przez urząd główny.

Jak to zwykle bywa, wszystko związane jest z bezpieczeństwem. Korzystanie z certyfikatów pośredniego urzędu certyfikacji jest bezpieczniejsze (dodatkowy poziom bezpieczeństwa), ponieważ w ten sposób główny urząd certyfikacji działa w trybie offline (poświęca wygodę w celu uzyskania bezpieczeństwa). Tak więc, jeśli certyfikat pośredni jest zagrożony, nie wpływa to na główny urząd certyfikacji.

Jeśli serwer nie wyśle ​​certyfikatów pośrednich wraz z głównym certyfikatem domeny, przeglądarki zaczną zgłaszać błąd z informacją `NET: ERR_CERT_AUTHORITY_INVALID` (w Chrome), ponieważ oczekiwały certyfikatu pośredniego, który podpisał certyfikat domeny, ale w odpowiedzi otrzymały tylko certyfikat domeny.

  > Nigdy nie należy ignorować takiego błędu, jeśli nie ufasz wystawcy certyfikatu!

W celu uzyskania dodatkowych informacji polecam dwie świetne odpowiedzi:

  > _Getting a new root certificates deployed due to compromised root is massively more difficult than replacing the certificates whose intermediates are compromised. [...] This is extremely hard to do in a short time. People don't upgrade their browser often enough. Some softwares like browsers have mechanism to quickly broadcasts revoked root certificates, and some software vendors have processes to rush release when a critical security vulnerability is found in their product, however you could be almost sure that they would not necessarily consider adding a new Root to warrant a rush update. Nor would people rush to update their software to get the new Root._ - [Lie Ryan](https://security.stackexchange.com/questions/128779/why-is-it-more-secure-to-use-intermediate-ca-certificates/128800#128800)

  > _The Root CA is offline for slow, awkward, but more secure servicing of requests. The use of multiple Intermediate CAs allows the "risk" of having the authority online and accessible to be divided into different sets of certificates; the eggs are spread into different baskets._ - [gowenfawr](https://security.stackexchange.com/questions/128779/why-is-it-more-secure-to-use-intermediate-ca-certificates/128791#128791)

# Podpisywanie certyfikatów

Spójrz na następujący schemat:

```
ROOT_CERT (isCA=yes)
|
|---INTERMEDIATE_CERT_1 (isCA=yes)
     |
     |---INTERMEDIATE_CERT_2 (isCA=yes)
         |
         |---LEAF_CERT valid for example.com (isCA=no)
```

Gdy urzędy certyfikacji podpisują certyfikaty, podpisują nie tylko klucz publiczny, ale także dodatkowe metadane. Te metadane obejmują m.in. datę wygaśnięcia certyfikatu. Najczęściej jest to zapisywane w formacie danych zdefiniowanym jako **X.509**. Co ważne, główny urząd certyfikacji stwierdza również, czy dany certyfikat może podpisywać inne „(pod) certyfikaty”, a tym samym „certyfikować” je, czy nie.

Zarówno główny urząd certyfikacji, jak i pośrednie urzędy certyfikacji mają tę samą właściwość. W związku z tym mogą podpisywać inne certyfikaty. Oczywiście, tzw. _leaf certificate_ (certyfikat serwera, końcowy) nie może mieć zgody na podpisywanie innych certyfikatów.

  > Jeśli certyfikat jest podpisany bezpośrednio przez zaufany główny urząd certyfikacji, nie ma potrzeby dodawania żadnych dodatkowych/pośrednich certyfikatów do łańcucha certyfikatów. Pamiętaj także, że główny urząd certyfikacji wydaje dla siebie certyfikat.

# Co się dzieje, gdy łańcuch jest „przerwany”?

W przypadku przerwania łańcucha nie można zweryfikować, czy serwer, na którym przechowywane są dane i z którym klient nawiązuje połączenie, jest poprawnym (oczekiwanym) serwerem. Dzieje się tak, ponieważ nie ma sposobu, aby upewnić się, że serwer jest faktycznie zaufanym serwerem. Przez to tracimy możliwość zweryfikowania bezpieczeństwa połączenia.

  > Przy niepełnym łańcuchu połączenia są nadal bezpieczne, ponieważ **ruch nadal jest szyfrowany**. Aby rozwiązać ten problem, należy ręcznie rozwiązać niepełny łańcucha certyfikatów, łącząc wszystkie certyfikaty od certyfikatu serwera z zaufanym certyfikatem głównym (wyłącznie, w tej kolejności).

Istnieje kilka możliwości przerwania łańcucha zaufania, w tym między innymi:

- każdy certyfikat w łańcuchu jest samopodpisany, chyba że jest to rootCA
- nie każdy certyfikat pośredni jest sprawdzany, począwszy od oryginalnego certyfikatu aż do certyfikatu głównego
- pośredni certyfikat podpisany przez urząd certyfikacji nie ma oczekiwanych podstawowych ograniczeń ani innych ważnych rozszerzeń
- certyfikat główny został przejęty lub autoryzowany dla niewłaściwej strony

W większości przypadków sam certyfikat serwera jest niewystarczający — do zbudowania pełnego łańcucha zaufania potrzebne są dwa lub więcej certyfikaty. Typowy problem z konfiguracją występuje podczas wdrażania serwera z ważnym certyfikatem, ale bez wszystkich niezbędnych certyfikatów pośrednich. Aby uniknąć tej sytuacji, wystarczy użyć wszystkich certyfikatów dostarczonych przez urząd certyfikacji w tej samej kolejności lub zbudować łańcuch samodzielnie, pobierając wszystkie niezbędne certyfikaty pośrednie.

Niepoprawny łańcuch certyfikatów skutecznie unieważnia certyfikat serwera i powoduje wyświetlanie ostrzeżeń w przeglądarce. W praktyce problem ten jest czasami trudny do zdiagnozowania, ponieważ niektóre przeglądarki mogą odtwarzać niekompletne łańcuchy, a niektóre nie. Wszystkie przeglądarki mają tendencję do buforowania i ponownego wykorzystywania certyfikatów pośrednich.

Jeżeli chcesz sprawdzić, jak zachowują się przeglądarki w przypadku napotkania niekompletnego łańcucha certyfikatów, zerknij na [incomplete-chain.badssl.com](https://incomplete-chain.badssl.com/).

# Przykład złożenia cerytyfikatów w poprawny łańcuch

Przyjmijmy, że dostaliśmy poniższy zestaw certyfikatów. Przed złożeniem ich w łańcuch, należy zweryfikować pola wystawcy oraz podmiotu, dla którego wystawiono certyfikat, aby mieć pewność, że są to wszystkie certyfikaty, za pomocą których utworzymy poprawny łańuch zaufania.

W tym przykładzie mamy następujący zestaw certyfikatów:

```bash
$ ls
root_ca.crt inter_ca.crt example.com.crt
```

Możemy zbudować łańcuch ręcznie, pamiętając, że pierwszym musi być certyfikat serwera (końcowy):

```bash
$ cat example.com.crt inter_ca.crt > certs/example.com/example.com-chain.crt
```

Dawno temu napisałem proste [narzędzie](https://github.com/trimstray/mkchain), które wykonuje całą pracę:

```bash
# Jeśli masz wszystkie certyfikaty:
$ ls /data/certs/example.com
root.crt inter01.crt inter02.crt certificate.crt
$ mkchain -i /data/certs/example.com -o /data/certs/example.com-chain.crt

# Jeśli masz tylko certyfikat końcowy, pozostałe wymagane są automatycznie pobierane:
$ ls /data/certs/example.com
certificate.crt
$ mkchain -i certificate.crt -o /data/certs/example.com-chain.crt

# Możesz także pobrać cały łańcuch dla danej domeny:
$ mkchain -i https://incomplete-chain.badssl.com/ -o /data/certs/example.com-chain.crt
```

Razem z nim dostarczone są przykładowe certyfikaty do testowego utworzenia poprawnego łańcucha. Na przykład:

```bash
cd example
ls
github.com  google.com  mozilla.com  ssllabs.com  vultr.com
cd ssllabs.com/all
ls
Intermediate1.crt  Intermediate2.crt  RootCertificate.crt  ServerCertificate.crt
mkchain -i . -o ../ssllabs-chain.crt

  SSL certificate chain:

             (ServerCertificate.crt)
             (Identity Certificate)
    S:(18319780):(ssllabs.com)
    I:(2835d715):(EntrustCertificationAuthority-L1K)
             (Intermediate1.crt)
             (Intermediate Certificate)
    S:(2835d715):(EntrustCertificationAuthority-L1K)
    I:(02265526):(EntrustRootCertificationAuthority-G2)
             (Intermediate2.crt)
             (Intermediate Certificate)
    S:(02265526):(EntrustRootCertificationAuthority-G2)
    I:(6b99d060):(EntrustRootCertificationAuthority)
             (RootCertificate.crt)
             (Root Certificate)
    S:(6b99d060):(EntrustRootCertificationAuthority)
    I:(6b99d060):(EntrustRootCertificationAuthority)

  Comments:

    * found correct identity (end-user, server) certificate
    * found 2 correct intermediate certificate(s)
    * found correct root certificate

  Result: chain generated correctly

  Chain file: ../ssllabs-chain.crt
```

# Testowanie łańcucha certyfikatów

Aby przetestować poprawność łańcucha certyfikatów, użyj jednego z następujących narzędzi:

- [SSL Checker by sslshopper](https://www.sslshopper.com/ssl-checker.html)
- [SSL Checker by namecheap](https://decoder.link/sslchecker/)
- [SSL Server Test by Qualys](https://www.ssllabs.com/ssltest/analyze.html)

Aby uzyskać więcej informacji, przeczytaj dokument [What is the SSL Certificate Chain?](https://support.dnsimple.com/articles/what-is-ssl-certificate-chain/) oraz [Get your certificate chain right](https://medium.com/@superseb/get-your-certificate-chain-right-4b117a9c0fce).
