---
layout: post
title: 'TLS: Jaki rozmiar powinien mieć klucz prywatny?'
date: 2020-02-29 12:29:39
categories: [publications]
tags: [PL, http, https, security, ssl, tls, private-key]
comments: false
favorite: false
seo:
  date_modified: 2020-02-25 08:59:14 +0100
---

Kryptografia klucza publicznego, zwana także kryptografią asymetryczną zakłada, że każda transmisja danych używa dwóch kluczy (w przeciwieństwie do kryptografii symetrycznej). Jeden z nich, klucz publiczny, wykorzystywany jest do szyfrowania wiadomości, zaś drugi, klucz prywatny, do jej deszyfrowania.

W tym wpisie chciałbym omówić kwestię długości klucza prywatnego oraz przedstawić czym jest i jaką pełni rolę w komunikacji.

# Czym jest klucz prywatny?

Klucz prywatny jest kluczem tajnym (co do zasady powinien być traktowany jako tajny) używanym do odszyfrowywania wiadomości, które są zaszyfrowane kluczem publicznych (dostarczanym razem z certyfikatem i przesyłanym za każdym razem do klienta). Jego użycie pozwala uniknąć słabości szyfrowania symetrycznego, w którym klucz tajny jest współdzielony przez obie strony komunikacji.

  > Serwer NGINX dostarcza dyrektywę `ssl_certificate_key` za pomocą której można ustawić ścieżkę do klucza prywatnego. Plik z kluczem prywatnym powinien być przechowywany w pliku z ograniczonym dostępem, co więcej, musi być możliwy do odczytania przez główny proces NGINX.

Spójrzmy na przykładzie komunikacji klienta z serwerem HTTP:

<p align="center">
  <img src="/assets/img/posts/tls_priv_pub.png">
</p>

# Związek między długością klucza a wydajnością

Klucz publiczny jest w pewien sposób powiązany z kluczem prywatnym co oznacza, że między nimi musi istnieć jakaś unikalna (matematyczna) zależność. W związku z tym, może to być słaby punkt, który przy jego złamaniu, może doprowadzić do kompromitacji szyfrowania. Stąd klucze asymetryczne muszą być znacznie dłuższe, aby móc zapewnić równoważny poziom bezpieczeństwa.

Narzut obliczeniowy jest wtedy dość oczywisty, ponieważ klucz publiczny jest dostępny dla każdego, stąd musi być wraz z kluczem prywatnym wystarczająco długi by zapewnić silniejszy poziom szyfrowania oraz by nie można było go złamać. Rezultatem jest oczywiście znacznie silniejszy poziom szyfrowania. Koniec końców, 128-bitowy klucz symetryczny i 2048-bitowy klucz asymetryczny oferują mniej więcej podobny poziom bezpieczeństwa.

Na podstawie znajomości klucza publicznego, nie powinno być możliwe odtworzenie klucza prywatnego, i na odwrót. Co więcej, każda asymetryczna para kluczy jest unikatowa, dzięki czemu wiadomość zaszyfrowana przy użyciu klucza publicznego może zostać odczytana tylko przez osobę posiadającą odpowiedni klucz prywatny.

  > Jeżeli klucz prywatny zostanie udostępniony lub w jakikolwiek sposób ujawniony, bezpieczeństwo wszystkich wiadomości, które zostały zaszyfrowane za pomocą odpowiadającego mu klucza publicznego, zostanie naruszona.

Jednym z najpopularniejszych algorytmów asymetrycznych jest RSA. Niestety, ze względu na złożone operacje matematyczne związane z szyfrowaniem i deszyfrowywaniem, algorytmy asymetryczne okazują się dosyć powolne (zwłaszcza sam proces deszyfrowania) w przypadku zetknięcia ich z dużymi zestawami danych.

Dzieje się tak, ponieważ bezpieczeństwo szyfrowania opiera się na trudności faktoryzacji (złożoności obliczeniowej) dużych liczb pierwszych (tzw. `p` i `q`). Alternatywą jest szyfrowania oparte na krzywych eliptycznych, które wymaga znacznie mniejszych kluczy.

Klucze ECDSA (zawierające klucze publiczne ECC) są zalecane w porównaniu z RSA, ponieważ oferują ten sam lub większy poziom bezpieczeństwa — klucze oparte na krzywych eliptycznych są mniej wrażliwe i trudniejsze do złamania. Oczywiście klucze RSA są również bardzo szybkie, zapewniając bardzo proste szyfrowanie i weryfikację oraz są łatwiejsze do wdrożenia niż ECC.

Oczywiście istnieje wiele innych dodatkowych czynników, które mogą wpływać na „szybkość” infrastruktury klucza publicznego. Ponieważ jednym z problemów związanych z tym systemem jest zaufanie, większość implementacji dotyczy urzędu certyfikacji (CA), które są podmiotami ufającymi w delegowaniu par kluczy i sprawdzaniu „tożsamości” kluczy.

# Długość klucza prywatnego - fakty i mity

Rekomendacja: <font color="#e5282d"><b>Używaj kluczy prywatnych RSA min. 2048-bit lub ECC min. 256-bit</b></font>

Certyfikaty SSL najczęściej używają kluczy RSA, zaś zalecany rozmiar tych kluczy rośnie co jakiś czas, aby utrzymać wystarczającą siłę kryptograficzną. Prawda jest taka (zwłaszcza jeśli mówimy o RSA), że przemysł/społeczność są podzielone na temat rozmiaru kluczy. Sam jestem w obozie „używaj kluczy RSA 2048-bit, ponieważ 4096-bit nie daje nam prawie nic, a jednocześnie sporo kosztuje”.

Alternatywą dla RSA jest ECC (ECDSA). Oba typy kluczy mają tę samą ważną właściwość, mianowicie są algorytmami asymetrycznymi (jeden klucz do szyfrowania i jeden klucz do deszyfrowania).

  > NGINX obsługuje podwójne certyfikaty, dzięki czemu możesz używać lżejszych i szybszych certyfikatów ECC równocześnie nadal zezwalać odwiedzającym przeglądać Twoją witrynę za pomocą standardowych certyfikatów.

Eksperci ds. Bezpieczeństwa przewidują, że 2048 bitów będzie wystarczające do użytku komercyjnego do około 2030 roku (zgodnie z normą [NIST](https://www.keylength.com/en/4/)). Amerykańska Agencja Bezpieczeństwa Narodowego (NSA) wymaga, aby wszystkie ściśle tajne pliki i dokumenty były szyfrowane przy użyciu 384-bitowych kluczy ECC (7680-bitowy klucz RSA). Ponadto, ze względów bezpieczeństwa, [CA/Browser forum - Baseline Requirements](https://cabforum.org/wp-content/uploads/CA-Browser-Forum-BR-1.6.7.pdf) <sup>[pdf]</sup> i IST zaleca użycie 2048-bitowego klucza RSA do certyfikatów/kluczy subskrybentów.

Z drugiej strony, najnowsza wersja [FIPS-186-5 (Draft)](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.186-5-draft.pdf) <sup>[pdf]</sup> określa zastosowanie modułu, którego długość bitu jest liczbą całkowitą równą i większą lub równą 2048 bitów (starsza wersja, tj. FIPS-186-4 z 2013 roku, mówiła, że rząd federalny USA generuje (i używa) podpisy cyfrowe o długości klucza 1024, 2048 lub 3072 bity).

Co więcej, zalecenia Europejskiej Rady ds. Płatności ([EPC342-08 v8.0](https://www.europeanpaymentscouncil.eu/sites/default/files/kb/file/2019-01/EPC342-08%20v8.0%20Guidelines%20on%20cryptographic%20algorithms%20usage%20and%20key%20management.pdf) <sup>[pdf]</sup>) mówią, że należy unikać używania 1024-bitowych kluczy RSA i 160-bitowych kluczy ECC w nowych aplikacjach, z wyjątkiem krótkoterminowej ochrony niekrytycznych aplikacji. EPC zaleca stosowanie co najmniej 2048-bitowego RSA lub 224-bitowego ECC do ochrony średnioterminowej (np. 10-letniej). Klasyfikują także SHA-1, moduły RSA 1024-bit, klucze ECC 160-bit jako odpowiednie do użycia w starszych wersjach (moim zdaniem SHA-1 nie nadaje się do tych zastosowań).

Książka „SSL/TLS Deployment Best Practices” także opisuje problem rozmiaru klucza w interesujący sposób:

  > The cryptographic handshake, which is used to establish secure connections, is an operation whose cost is highly influenced by private key size. Using a key that is too short is insecure, but using a key that is too long will result in "too much" security and slow operation. For most web sites, using RSA keys stronger than 2048 bits and ECDSA keys stronger than 256 bits is a waste of CPU power and might impair user experience. Similarly, there is little benefit to increasing the strength of the ephemeral key exchange beyond 2048 bits for DHE and 256 bits for ECDHE.

Dłuższe klucze RSA zajmują więcej czasu procesora, gdy są używane do szyfrowania i deszyfrowania. Również uzgadnianie sesji SSL/TLS na początku każdego połączenia będzie wolniejsze. Ten typ kryptografii ma również niewielki wpływ na stronę klienta (np. przeglądarki). Podczas korzystania z krzywej `curve25519`, ECC jest uważane za bardziej bezpieczne. Z założenia jest szybki i odporny na różne ataki. Oczywiście, RSA nie jest mniej bezpieczny, co więcej, pod względem praktycznym jest również uważany za „niezniszczalny”.

Zasadniczo nie ma istotnego powodu, aby wybierać klucze 4096-bitowe. Nie spotkałem się z dodatkowymi wytycznymi co do tego, jednak moim zdaniem istnieje jeden ważny warunek, na który powinieneś zwrócić uwagę. Stosowanie kluczy 2048-bit powinno iść w parze z rozsądnymi interwałami ważności (np. nie więcej niż 6-12 miesięcy dla 2048-bitowego klucza i certyfikatu), aby dać atakującemu mniej czasu na złamanie klucza i zminimalizować prawdopodobieństwo, że ktoś wykorzysta wszelkie luki, które mogą wystąpić w przypadku naruszenia jego bezpieczeństwa.

  > Moim zdaniem powinniśmy bardziej martwić się, że nasze klucze prywatne zostaną skradzione w wyniku naruszenia bezpieczeństwa serwera. Pamiętaj, że ciągły postęp technologiczny naraża nasz klucz na ataki. Polecam przeczytanie dokumentu [Secure Distribution of SSL Private Keys with NGINX](https://www.nginx.com/blog/secure-distribution-ssl-private-keys-nginx/) w celu zminimalizowania ew. ataku na klucze prywatne obsługiwane z poziomu serwera NGINX.

Podsumowując. Myślę, że jeśli kiedykolwiek znajdziemy się w świecie, w którym 2048-bitowe klucze nie będą już wystarczająco dobre, nie będzie to w cale spowodowane możliwością ich z brute-forcowania, tylko dlatego, że RSA stanie się po prostu przestarzałe jako technologia w opozycji do rewolucyjnych osiągnięć komputerowych. Jeśli tak się stanie, 3072 lub 4096 bitów i tak nie zrobi dużej różnicy. Właśnie dlatego wszystko powyżej 2048 bitów jest ogólnie uważane za rodzaj mocnego przerysowania jeśli chodzi o bezpieczeństwo.

## RSA vs ECC

Wróćmy jeszcze do porównania obu typów kluczy. 256-bitowy klucz ECC może być silniejszy niż 2048-bitowy klucz klasyczny. Jeśli używasz ECDSA, zalecany rozmiar klucza zmienia się w zależności od użycia, patrz [NIST 800-57 (page 12, table 2-1)](https://nvlpubs.nist.gov/nistpubs/specialpublications/nist.sp.800-57pt3r1.pdf) <sup>[pdf]</sup>.

Główny problem to wdrożenie. Chociaż prawdą jest, że dłuższy klucz zapewnia lepsze bezpieczeństwo, podwajając długość klucza RSA z 2048 do 4096, wzrost bitów bezpieczeństwa wynosi tylko 18, czyli zaledwie 16% (czas na podpisanie wiadomości wzrasta 7 razy, a w niektórych przypadkach czas weryfikacji podpisu zwiększa się ponad 3-krotnie). Ponadto, poza wymaganiem większej przestrzeni dyskowej (jest to co prawda minimalny skutek uboczny ich stosowania), dłuższe klucze przekładają się również na zwiększone użycie procesora.

Tak naprawdę prawdziwą zaletą używania klucza 4096-bitowego jest zabezpieczenie na przyszłość. Jeśli chcesz uzyskać ocenę A+ oraz 100%s dla Key Exchange skanera SSLLabs, zdecydowanie powinieneś użyć 4096 bitowych kluczy prywatnych. To główny (i jedyny dla mnie) powód, dla którego powinieneś ich używać.

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
