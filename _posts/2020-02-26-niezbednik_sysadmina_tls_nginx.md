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

Napisano wiele bardzo dobrym artykułów dotyczących TLS. W tej serii wpisów, chciałbym przedstawić najważniejsze rzeczy oraz poruszyć niektóre ciekawe tematy z punktu widzenia każdego administratora, a także przedstawić je na przykładzie konfiguracji serwera NGINX.

Prezentowane konfiguracje oraz zasady odnoszą się do najnowszych zaleceń.

# Czym jest TLS?

Protokół TLS (ang. _Transport Layer Security_) jest podstawowym protokołem zapewniającym prywatność i integralność danych między dwoma komunikującymi się aplikacjami. Jest to także najczęściej stosowany protokół bezpieczeństwa, zastępujący dziś SSL (ang. _Secure Socket Layer_), używany w przeglądarkach internetowych i innych aplikacjach wymagających bezpiecznej wymiany danych przez sieć.

TLS zapewnia też, że podczas połączenia ze zdalnym punktem końcowym, jest on tym, za kogo się podaje poprzez szyfrowanie oraz weryfikację jego tożsamości.

# Używaj aktualnej i gotowej do produkcji wersji OpenSSL

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

  > Następną wersją będzie [OpenSSL 3.0.0](https://blog.apnic.net/2019/10/21/openssl-3-0-accelerating-forwards/).

# Używaj kluczy prywatnych min. 2048-bit (RSA) lub min. 256-bit for (ECC)

Certyfikaty SSL najczęściej używają kluczy RSA, a zalecany rozmiar tych kluczy stale rośnie, aby utrzymać wystarczającą siłę kryptograficzną. Alternatywą dla RSA jest ECC. ECC (i ECDSA) jest prawdopodobnie lepszy do większości celów, ale nie do wszystkiego.

Oba typy kluczy mają tę samą ważną właściwość, że są algorytmami asymetrycznymi (jeden klucz do szyfrowania i jeden klucz do deszyfrowania). NGINX obsługuje podwójne certyfikaty, dzięki czemu możesz uzyskać lżejsze, wredniejsze certyfikaty ECC, ale nadal pozwala odwiedzającym przeglądać Twoją witrynę za pomocą standardowych certyfikatów.

Prawda jest taka (jeśli mówimy o RSA), że przemysł/społeczność są podzielone na temat rozmiaru kluczy. Sam jestem w obozie „użyj kluczy RSA 2048-bit, ponieważ 4096-bit nie daje nam prawie nic, a jednocześnie kosztuje nas sporo”.

Zalecane są obecnie klucze 2048-bitowe dla RSA (lub 256-bitowe dla ECC). Eksperci ds. Bezpieczeństwa przewidują, że 2048 bitów będzie wystarczających do użytku komercyjnego do około roku 2030 (zgodnie z NIST). Najnowsza wersja FIPS-186 mówi również, że rząd federalny USA generuje (i używa) podpisy cyfrowe o długości klucza 1024, 2048 lub 3072 bitów. Amerykańska Agencja Bezpieczeństwa Narodowego (NSA) wymaga, aby wszystkie ściśle tajne pliki i dokumenty były szyfrowane przy użyciu 384-bitowych kluczy ECC (7680-bitowy klucz RSA).
