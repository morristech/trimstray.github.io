---
layout: post
title: 'Niezbędnik SysAdmina: TLS + NGINX'
date: 2020-02-27 12:29:39
categories: [publications]
tags: [PL, http, https, security, ssl, tls, ciphers]
comments: false
favorite: false
seo:
  date_modified: 2020-02-25 08:59:14 +0100
---

Napisano wiele bardzo dobrym artykułów dotyczących TLS. W tym wpisie, chciałbym przedstawić najważniejsze rzeczy oraz poruszyć niektóre ciekawe tematy z punktu widzenia każdego administratora, a także przedstawić je na przykładzie konfiguracji serwera NGINX.

Prezentowane konfiguracje oraz zasady odnoszą się do najnowszych zaleceń. Oczywiście musimy mieć świadomość, że niektóre z opisywanych tutaj opcji mogą ulec przedawnieniu szybciej niż inne. W tym dokumencie zawarłem także odnośniki do zewnętrznych zasobów w celu pogłebienia wiedzy na dany temat.

# Czym jest TLS?

Protokół TLS (ang. _Transport Layer Security_) jest podstawowym protokołem zapewniającym prywatność i integralność danych między dwoma komunikującymi się urządzeniami lub aplikacjami. Jest to także najczęściej stosowany protokół bezpieczeństwa używany w przeglądarkach internetowych i innych aplikacjach wymagających bezpiecznej wymiany danych przez sieć. TLS zastępuje także starszą wersję protokołu, tj. SSL (ang. _Secure Socket Layer_).

Jedną z ważniejszych funkcji protokołu TLS jest gwarancja, że podczas połączenia ze zdalnym punktem końcowym, jest on tym, za kogo się podaje. Wynika to z dwóch kluczowych funkcji:

- szyfrowania komunikacji między punktem źródłowym a punktem docelowym
- weryfikacji tożsamości punktu docelowego

# Biblioteka OpenSSL

Zalecenie: <font color="#e5282d"><b>Używaj aktualnej i gotowej do uruchomienia produkcyjnie wersji OpenSSL</b></font>

Biblioteka [OpenSSL](https://www.openssl.org/) jest najpopularniejszą implementacją protokołu SSL/TLS. Zawiera ona podstawowe funkcje kryptograficzne oraz dostarcza szereg narzędzi administratyjnych.

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

Zalecenie: <font color="#e5282d"><b>Używaj kluczy prywatnych RSA o min. długości 2048-bit lub ECC o min. długości 256-bit</b></font>

Certyfikaty SSL najczęściej używają kluczy RSA, zaś zalecany rozmiar tych kluczy rośnie co jakiś czas, aby utrzymać wystarczającą siłę kryptograficzną.

Alternatywą dla RSA jest ECC. ECC (i ECDSA) jest prawdopodobnie lepszy do większości celów, ale nie do wszystkiego. Oba typy kluczy mają tę samą ważną właściwość, mianowicie są algorytmami asymetrycznymi (jeden klucz do szyfrowania i jeden klucz do deszyfrowania).

  > NGINX obsługuje podwójne certyfikaty, dzięki czemu możesz używać lżejszych i szybszych certyfikatów ECC równocześnie nadal zezwalać odwiedzającym przeglądać Twoją witrynę za pomocą standardowych certyfikatów.

Prawda jest taka (zwłaszcza jeśli mówimy o RSA), że przemysł/społeczność są podzielone na temat rozmiaru kluczy. Sam jestem w obozie „używaj kluczy RSA 2048-bit, ponieważ 4096-bit nie daje nam prawie nic, a jednocześnie sporo kosztuje”.

Eksperci ds. Bezpieczeństwa przewidują, że 2048 bitów będzie wystarczające do użytku komercyjnego do około 2030 roku (zgodnie z normą [NIST](https://www.keylength.com/en/4/)). Amerykańska Agencja Bezpieczeństwa Narodowego (NSA) wymaga, aby wszystkie ściśle tajne pliki i dokumenty były szyfrowane przy użyciu 384-bitowych kluczy ECC (7680-bitowy klucz RSA). Z drugiej strony, najnowsza wersja [FIPS-186](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.186-4.pdf) <sup>[pdf]</sup> mówi, że rząd federalny USA generuje (i używa) podpisy cyfrowe o długości klucza 1024, 2048 lub 3072 bitów.

Co więcej, zalecenia Europejskiej Rady ds. Płatności ([EPC342-08 v8.0](https://www.europeanpaymentscouncil.eu/sites/default/files/kb/file/2019-01/EPC342-08%20v8.0%20Guidelines%20on%20cryptographic%20algorithms%20usage%20and%20key%20management.pdf) <sup>[pdf]</sup>) mówią, że należy unikać używania 1024-bitowych kluczy RSA i 160-bitowych kluczy ECC w nowych aplikacjach, chyba że krótkoterminowej ochrony nie krytycznych aplikacji. EPC zaleca stosowanie co najmniej 2048-bitowego RSA lub 224-bitowego ECC do ochrony średnioterminowej (np. 10-letniej). Klasyfikują także SHA-1, moduły RSA 1024-bit, klucze ECC 160-bit jako odpowiednie do użycia w starszych wersjach (moim zdaniem SHA-1 nie nadaje się do tych zastosowań).

Zasadniczo nie ma istotnego powodu, aby wybierać klucze 4096-bitowe dla RSA. Nie spotkałem się z dodatkowymi wytycznymi co do tego. Jednak moim zdaniem istnieje jeden ważny warunek, na który powinieneś zwrócić uwagę. Stosowanie kluczy 2048-bit powinno iść w parze z rozsądnymi interwałami ważności (np. nie więcej niż 6-12 miesięcy dla 2048-bitowego klucza i certyfikatu), aby dać atakującemu mniej czasu na złamanie klucza i zminimalizować prawdopodobieństwo, że ktoś wykorzysta wszelkie luki, które mogą wystąpić w przypadku naruszenia jego bezpieczeństwa.

Co więcej, moim zdaniem powinniśmy bardziej się martwić, że nasze klucze prywatne zostaną skradzione w wyniku naruszenia bezpieczeństwa serwera, a postęp technologiczny naraża nasz klucz na ataki.

256-bitowy klucz ECC może być silniejszy niż 2048-bitowy klucz klasyczny. Jeśli używasz ECDSA, zalecany rozmiar klucza zmienia się w zależności od użycia, patrz [NIST 800-57 (page 12, table 2-1)](https://nvlpubs.nist.gov/nistpubs/specialpublications/nist.sp.800-57pt3r1.pdf) <sup>[pdf]</sup>. Chociaż prawdą jest, że dłuższy klucz zapewnia lepsze bezpieczeństwo, podwajając długość klucza RSA z 2048 do 4096, wzrost bitów bezpieczeństwa wynosi tylko 18, zaledwie 16% (czas na podpisanie wiadomości wzrasta 7 razy, a w niektórych przypadkach czas weryfikacji podpisu zwiększa się ponad 3-krotnie). Ponadto, poza wymaganiem większej przestrzeni dyskowej, dłuższe klucze przekładają się również na zwiększone użycie procesora.

ECC jest lepsze niż RSA pod względem długości klucza. Ale główne problemy to wdrożenie. Myślę, że RSA jest łatwiejszy do wdrożenia niż ECC. Klucze ECDSA (zawierające klucze publiczne ECC) są zalecane w porównaniu z RSA, ponieważ oferują ten sam poziom bezpieczeństwa z mniejszymi kluczami w przeciwieństwie do kryptografii spoza ECC. Klucze ECC są lepsze niż klucze RSA i DSA, ponieważ algorytm ECC jest trudniejszy do złamania (mniej wrażliwy). Moim zdaniem ECC nadaje się do środowisk o dużej ilości ograniczeń (ograniczone zasoby pamięci lub przetwarzania danych), np. telefony komórkowe, urządzenia PDA i ogólnie systemy wbudowane. Oczywiście klucze RSA są bardzo szybkie, zapewniają bardzo proste szyfrowanie i weryfikację oraz są łatwiejsze do wdrożenia niż ECC.

Dłuższe klucze RSA generują więcej czasu i wymagają więcej procesora i energii, gdy są używane do szyfrowania i deszyfrowania, również uzgadnianie SSL na początku każdego połączenia będzie wolniejsze. Ma również niewielki wpływ na stronę klienta (np. Przeglądarki). Podczas korzystania z curve25519, ECC jest uważane za bardziej bezpieczne. Z założenia jest szybki i odporny na różne ataki z boku kanału. RSA jest jednak nie mniej bezpieczny, ale pod względem praktycznym jest również uważany za niezniszczalny.

Tak naprawdę prawdziwą zaletą używania klucza 4096-bitowego jest zabezpieczenie na przyszłość. Jeśli chcesz uzyskać A + ze 100% s na SSL Lab (dla Key Exchange), zdecydowanie powinieneś użyć 4096 bitowych kluczy prywatnych. To główny (i jedyny dla mnie) powód, dla którego powinieneś ich używać.
