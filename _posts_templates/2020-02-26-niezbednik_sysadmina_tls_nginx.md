---
layout: post
title: 'TLS: Jaki rozmiar powinien mieć klucz prywatny?'
date: 2020-02-27 12:29:39
categories: [publications]
tags: [PL, http, https, security, ssl, tls, private-key]
comments: false
favorite: false
seo:
  date_modified: 2020-02-25 08:59:14 +0100
---

Napisano wiele bardzo dobrych artykułów dotyczących TLS. Podczas moich badań na tym protokołem sporządziłem wiele notatek, którymi chciałbym się podzielić. W tym wpisie, chciałbym przedstawić najważniejsze rzeczy oraz poruszyć niektóre ciekawe tematy z punktu widzenia każdego administratora, a także przedstawić je na przykładzie konfiguracji serwera NGINX.

Prezentowane konfiguracje oraz zasady odnoszą się do najnowszych zaleceń. Oczywiście musimy mieć świadomość, że niektóre z opisywanych tutaj rzeczy mogą ulec przedawnieniu szybciej niż inne dlatego zawsze powinieneś weryfikować dane zagadnienie oraz konfiguracje, którą wprowadzasz. W tym dokumencie zawarłem także odnośniki do zewnętrznych zasobów w celu pogłebienia wiedzy na dany temat.

# Czym jest TLS?

Protokół TLS (ang. _Transport Layer Security_) jest podstawowym protokołem zapewniającym prywatność i integralność danych między dwoma komunikującymi się urządzeniami lub aplikacjami. Jest to także najczęściej stosowany protokół bezpieczeństwa używany w przeglądarkach internetowych i innych aplikacjach wymagających bezpiecznej wymiany danych przez sieć. TLS zastępuje także starszą wersję protokołu, tj. SSL (ang. _Secure Socket Layer_).

Jedną z ważniejszych funkcji protokołu TLS jest gwarancja, że podczas połączenia ze zdalnym punktem końcowym, jest on tym, za kogo się podaje. Wynika to z dwóch kluczowych funkcji:

- szyfrowania komunikacji między punktem źródłowym a punktem docelowym
- weryfikacji tożsamości punktu docelowego

# Biblioteka OpenSSL

Rekomendacja: <font color="#e5282d"><b>Używaj aktualnej i gotowej do uruchomienia produkcyjnie wersji OpenSSL</b></font>

Biblioteka [OpenSSL](https://www.openssl.org/) jest najpopularniejszą implementacją protokołu SSL/TLS. Zawiera ona podstawowe funkcje kryptograficzne wykorzystywane przez aplikacje oraz dostarcza szereg narzędzi administracyjnych.

  > OpenSSL jest dostępny dla większości systemów w tym GNU/Linux oraz BSD.

Moim zdaniem, jedyny bezpieczny sposób wykorzystania tej biblioteki opiera się na aktualnej, wciąż wspieranej i gotowej produkcyjnie wersji. Co więcej, polecam trzymać się najnowszych wersji (np. 1.1.1 lub 1.1.1d w tym momencie).

Upewnij się więc, że twoja biblioteka OpenSSL jest zaktualizowana do najnowszej dostępnej wersji i zachęcaj swoich klientów do korzystania ze zaktualizowanego OpenSSL i współpracującego z nim oprogramowania.

Dobrym pomysłem jest także śledzenie [Release Strategy Policies](https://www.openssl.org/policies/releasestrat.html) oraz oficjalnej [listy zmian](https://www.openssl.org/news/changelog.html).

Oczywiście kryteria wyboru odpowiedniej wersji mogą się różnić i wszystko zależy od zastosowania. Uważam jednak, że niezależnie od zastosowań powinniśmy wybrać jedną z wersji, dla której wciąż świadczone jest wsparcie:

- wersja 1.1.1 wciąż wspierana do 2023-09-11 (LTS)
  - ostatnia: 1.1.1d (10 września 2019 r.)
- wersja 1.1.0 wciąż wspierana do 2019-09-11
  - ostatnia: 1.1.0k (28 maja 2018 r.)
- wersja 1.0.2 ciąż wspierana do 2019-12-31 (LTS)
  - ostatnia: 1.0.2s (28 maja 2018 r.)
- wszelkie inne wersje nie są już obsługiwane

  > Następną wersją biblioteki OpenSSL będzie [OpenSSL 3.0.0](https://blog.apnic.net/2019/10/21/openssl-3-0-accelerating-forwards/).

# Klucze prywatne

Rekomendacja: <font color="#e5282d"><b>Używaj kluczy prywatnych RSA min. 2048-bit lub ECC min. 256-bit</b></font>

Certyfikaty SSL najczęściej używają kluczy RSA, zaś zalecany rozmiar tych kluczy rośnie co jakiś czas, aby utrzymać wystarczającą siłę kryptograficzną.

Alternatywą dla RSA jest ECC. ECC (i ECDSA) jest prawdopodobnie lepszy do większości celów, ale nie do wszystkiego. Oba typy kluczy mają tę samą ważną właściwość, mianowicie są algorytmami asymetrycznymi (jeden klucz do szyfrowania i jeden klucz do deszyfrowania).

  > NGINX obsługuje podwójne certyfikaty, dzięki czemu możesz używać lżejszych i szybszych certyfikatów ECC równocześnie nadal zezwalać odwiedzającym przeglądać Twoją witrynę za pomocą standardowych certyfikatów.

Prawda jest taka (zwłaszcza jeśli mówimy o RSA), że przemysł/społeczność są podzielone na temat rozmiaru kluczy. Sam jestem w obozie „używaj kluczy RSA 2048-bit, ponieważ 4096-bit nie daje nam prawie nic, a jednocześnie sporo kosztuje”.

Eksperci ds. Bezpieczeństwa przewidują, że 2048 bitów będzie wystarczające do użytku komercyjnego do około 2030 roku (zgodnie z normą [NIST](https://www.keylength.com/en/4/)). Amerykańska Agencja Bezpieczeństwa Narodowego (NSA) wymaga, aby wszystkie ściśle tajne pliki i dokumenty były szyfrowane przy użyciu 384-bitowych kluczy ECC (7680-bitowy klucz RSA). Z drugiej strony, najnowsza wersja [FIPS-186](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.186-4.pdf) <sup>[pdf]</sup> mówi, że rząd federalny USA generuje (i używa) podpisy cyfrowe o długości klucza 1024, 2048 lub 3072 bitów.

Co więcej, zalecenia Europejskiej Rady ds. Płatności ([EPC342-08 v8.0](https://www.europeanpaymentscouncil.eu/sites/default/files/kb/file/2019-01/EPC342-08%20v8.0%20Guidelines%20on%20cryptographic%20algorithms%20usage%20and%20key%20management.pdf) <sup>[pdf]</sup>) mówią, że należy unikać używania 1024-bitowych kluczy RSA i 160-bitowych kluczy ECC w nowych aplikacjach, z wyjątkiem krótkoterminowej ochrony niekrytycznych aplikacji. EPC zaleca stosowanie co najmniej 2048-bitowego RSA lub 224-bitowego ECC do ochrony średnioterminowej (np. 10-letniej). Klasyfikują także SHA-1, moduły RSA 1024-bit, klucze ECC 160-bit jako odpowiednie do użycia w starszych wersjach (moim zdaniem SHA-1 nie nadaje się do tych zastosowań).

Książka "SSL/TLS Deployment Best Practices" także opisuje problem rozmiaru klucza w interesujący sposób:

  > The cryptographic handshake, which is used to establish secure connections, is an operation whose cost is highly influenced by private key size. Using a key that is too short is insecure, but using a key that is too long will result in "too much" security and slow operation. For most web sites, using RSA keys stronger than 2048 bits and ECDSA keys stronger than 256 bits is a waste of CPU power and might impair user experience. Similarly, there is little benefit to increasing the strength of the ephemeral key exchange beyond 2048 bits for DHE and 256 bits for ECDHE.

Dłuższe klucze RSA generują więcej czasu i wymagają więcej procesora i energii, gdy są używane do szyfrowania i deszyfrowania, również uzgadnianie SSL na początku każdego połączenia będzie wolniejsze. Ma również niewielki wpływ na stronę klienta (np. przeglądarki). Podczas korzystania z krzywej `curve25519`, ECC jest uważane za bardziej bezpieczne. Z założenia jest szybki i odporny na różne ataki z boku kanału. RSA jest jednak nie mniej bezpieczny, ale pod względem praktycznym jest również uważany za niezniszczalny.

Zasadniczo nie ma istotnego powodu, aby wybierać klucze 4096-bitowe. Nie spotkałem się z dodatkowymi wytycznymi co do tego, jednak moim zdaniem istnieje jeden ważny warunek, na który powinieneś zwrócić uwagę. Stosowanie kluczy 2048-bit powinno iść w parze z rozsądnymi interwałami ważności (np. nie więcej niż 6-12 miesięcy dla 2048-bitowego klucza i certyfikatu), aby dać atakującemu mniej czasu na złamanie klucza i zminimalizować prawdopodobieństwo, że ktoś wykorzysta wszelkie luki, które mogą wystąpić w przypadku naruszenia jego bezpieczeństwa.

  > Moim zdaniem powinniśmy bardziej martwić się, że nasze klucze prywatne zostaną skradzione w wyniku naruszenia bezpieczeństwa serwera. Pamiętaj, że ciągły postęp technologiczny naraża nasz klucz na ataki.

Podsumowując. Myślę, że jeśli kiedykolwiek znajdziemy się w świecie, w którym 2048-bitowe klucze nie będą już wystarczająco dobre, nie będzie to w cale spowodowane możliwością ich z brute-forcowania, tylko dlatego, że RSA stanie się po prostu przestarzałe jako technologia w opozycji do rewolucyjnych osiągnięć komputerowych. Jeśli tak się stanie, 3072 lub 4096 bitów i tak nie zrobi dużej różnicy. Właśnie dlatego wszystko powyżej 2048 bitów jest ogólnie uważane za rodzaj mocnego przerysowania jeśli chodzi o bezpieczeństwo.

## RSA vs ECC

Wróćmy jeszcze do porównania obu typów kluczy. 256-bitowy klucz ECC może być silniejszy niż 2048-bitowy klucz klasyczny. Jeśli używasz ECDSA, zalecany rozmiar klucza zmienia się w zależności od użycia, patrz [NIST 800-57 (page 12, table 2-1)](https://nvlpubs.nist.gov/nistpubs/specialpublications/nist.sp.800-57pt3r1.pdf) <sup>[pdf]</sup>.

Główny problem to wdrożenie. Chociaż prawdą jest, że dłuższy klucz zapewnia lepsze bezpieczeństwo, podwajając długość klucza RSA z 2048 do 4096, wzrost bitów bezpieczeństwa wynosi tylko 18, czyli zaledwie 16% (czas na podpisanie wiadomości wzrasta 7 razy, a w niektórych przypadkach czas weryfikacji podpisu zwiększa się ponad 3-krotnie). Ponadto, poza wymaganiem większej przestrzeni dyskowej (jest to co prawda minimalny impakt ich stosowania), dłuższe klucze przekładają się również na zwiększone użycie procesora.

Klucze ECDSA (zawierające klucze publiczne ECC) są zalecane w porównaniu z RSA, ponieważ oferują ten sam poziom bezpieczeństwa z mniejszymi kluczami w przeciwieństwie do kryptografii spoza ECC. Klucze ECC są lepsze niż klucze RSA i DSA, ponieważ algorytm ECC jest trudniejszy do złamania (mniej wrażliwy). Moim zdaniem ECC nadaje się do środowisk, w których występują ograniczone zasoby pamięci lub przetwarzania danych, np. telefony komórkowe, urządzenia PDA i ogólnie systemy wbudowane. Oczywiście klucze RSA są bardzo szybkie, zapewniają bardzo proste szyfrowanie i weryfikację oraz są łatwiejsze do wdrożenia niż ECC.

Tak naprawdę prawdziwą zaletą używania klucza 4096-bitowego jest zabezpieczenie na przyszłość. Jeśli chcesz uzyskać A + ze 100% s na SSL Lab (dla Key Exchange), zdecydowanie powinieneś użyć 4096 bitowych kluczy prywatnych. To główny (i jedyny dla mnie) powód, dla którego powinieneś ich używać.

## Dodatkowe źródła

- [Key Management Guidelines by NIST](https://csrc.nist.gov/Projects/Key-Management/Key-Management-Guidelines) <sup>[NIST]</sup>
- [Recommendation for Transitioning the Use of Cryptographic Algorithms and Key Lengths](https://csrc.nist.gov/publications/detail/sp/800-131a/archive/2011-01-13) <sup>[NIST]</sup>
- [NIST SP 800-57 Part 1 Rev. 3 - Recommendation for Key Management](https://csrc.nist.gov/publications/detail/sp/800-57-part-1/rev-3/archive/2012-07-10) <sup>[NIST]</sup>
- [FIPS PUB 186-4 - Digital Signature Standard (DSS)](http://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.186-4.pdf) <sup>[NIST, pdf]</sup>
- [Cryptographic Key Length Recommendations](https://www.keylength.com/)
- [Key Lengths - Contribution to The Handbook of Information Security](https://infoscience.epfl.ch/record/164539/files/NPDF-32.pdf) <sup>[pdf]</sup>
- [NIST - Key Management](https://csrc.nist.gov/Projects/Key-Management/publications) <sup>[NIST]</sup>
- [So you're making an RSA key for an HTTPS certificate. What key size do you use?](https://certsimple.com/blog/measuring-ssl-rsa-keys)
- [RSA Key Sizes: 2048 or 4096 bits?](https://danielpocock.com/rsa-key-sizes-2048-or-4096-bits/)
- [Create a self-signed ECC certificate](https://msol.io/blog/tech/create-a-self-signed-ecc-certificate/)
- [ECDSA: Elliptic Curve Signatures](https://cryptobook.nakov.com/digital-signatures/ecdsa-sign-verify-messages)
- [Elliptic Curve Cryptography Explained](https://fangpenlin.com/posts/2019/10/07/elliptic-curve-cryptography-explained/)
- [You should be using ECC for your SSL/TLS certificates](https://www.thesslstore.com/blog/you-should-be-using-ecc-for-your-ssl-tls-certificates/)
- [Comparing ECC vs RSA](https://www.linkedin.com/pulse/comparing-ecc-vs-rsa-ott-sarv)
- [Comparison And Evaluation Of Digital Signature Schemes Employed In Ndn Network](https://arxiv.org/pdf/1508.00184.pdf) <sup>[pdf]</sup>
- [HTTPS Performance, 2048-bit vs 4096-bit](https://blog.nytsoi.net/2015/11/02/nginx-https-performance)
- [RSA and ECDSA hybrid Nginx setup with LetsEncrypt certificates](https://hackernoon.com/rsa-and-ecdsa-hybrid-nginx-setup-with-letsencrypt-certificates-ee422695d7d3)
- [Why ninety-day lifetimes for certificates?](https://letsencrypt.org/2015/11/09/why-90-days.html)
- [SSL Certificate Validity Will Be Limited to One Year by Apple’s Safari Browser](https://www.thesslstore.com/blog/ssl-certificate-validity-will-be-limited-to-one-year-by-apples-safari-browser/)
- [Certificate lifetime capped to 1 year from Sep 2020](https://scotthelme.co.uk/certificate-lifetime-capped-to-1-year-from-sep-2020/)

# Certyfikaty

Certyfikat SSL zawiera klucz publiczny i informacje o jego właścicielu, uwierzytelnia tożsamość strony internetowej i umożliwia bezpieczne połączenia z serwera internetowego do przeglądarki poprzez szyfrowanie informacji i ochronę poufnych danych (dane logowania, rejestracje, adresy i płatności) przesyłane z i na twojej stronie internetowej.

Autentyczność i integralność certyfikatu można sprawdzić metodami kryptograficznymi. Certyfikat cyfrowy zawiera dane wymagane do jego weryfikacji.

  > Bez certyfikatu SSL wszelkie dane gromadzone za pośrednictwem witryny są podatne na przechwycenie przez osoby trzecie.

Certyfikaty pozwalają zabezpieczyć domenę główną i wszystkie jej poddomeny (np. `example.com` i `api.example.com`) za pomocą jednego certyfikatu SSL.

## Łańcuch certyfikatów

Walidacja łańcucha certyfikatów jest kluczową częścią każdego procesu uwierzytelniania opartego na certyfikatach. Jeśli system nie przestrzega łańcucha zaufania certyfikatu do serwera głównego, certyfikat traci wszelką użyteczność jako wskaźnik zaufania.

Łańcuch certyfikatów składa się ze wszystkich certyfikatów potrzebnych do certyfikacji podmiotu określonego w certyfikacie końcowym. W praktyce obejmuje to certyfikat końcowy, certyfikaty pośrednich urzędów certyfikacji oraz certyfikat głównego urzędu certyfikacji zaufany przez wszystkie strony w łańcuchu. Każdy pośredni urząd certyfikacji w łańcuchu posiada certyfikat wydany przez urząd certyfikacji jeden poziom nad nim w hierarchii zaufania.

Certyfikat serwera wraz z łańcuchem nie jest przeznaczony dla serwera. Serwer nie ma zastosowania do własnego certyfikatu. Certyfikaty są zawsze dla innych osób (tutaj klienta). Serwer używa klucza prywatnego (który odpowiada kluczowi publicznemu w certyfikacie). W szczególności serwer nie musi ufać własnemu certyfikatowi ani żadnemu urzędowi certyfikacji, który go wydał.

