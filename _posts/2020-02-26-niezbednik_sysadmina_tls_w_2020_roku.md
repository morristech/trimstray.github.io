---
layout: post
title: 'Niezbędnik SysAdmina: TLS w 2020 roku'
date: 2020-02-27 12:29:39
categories: [publications]
tags: [PL, http, https, security, ssl, tls, ciphers]
comments: false
favorite: false
seo:
  date_modified: 2020-02-25 08:59:14 +0100
---

Napisano wiele bardzo dobrym artykułów dotyczących TLS. W tej serii wpisów, chciałbym przedstawić najważniejsze rzeczy oraz poruszyć niektóre ciekawe tematy z punktu widzenia każdego administratora, a także przedstawić je na przykładzie konfiguracji serwera NGINX.

Prezentowane konfiguracje oraz zalecenia określają stan na 2020 rok.

# Czym jest TLS?

Protokół TLS jest podstawowym protokołem zapewniającym prywatność i integralność danych między dwoma komunikującymi się aplikacjami. Jest to także najczęściej stosowany protokół bezpieczeństwa, zastępujący dziś SSL (ang. _Secure Socket Layer_), używany w przeglądarkach internetowych i innych aplikacjach wymagających bezpiecznej wymiany danych przez sieć.

TLS zapewnia też, że podczas połączenia ze zdalnym punktem końcowym, jest on tym, za kogo się podaje poprzez szyfrowanie oraz weryfikację jego tożsamości.

## OpenSSL

Biblioteka [OpenSSL](https://www.openssl.org/) jest najpopularniejszą implementacją protokołu SSL/TLS i została napisana w języku C. Zawiera ona podstawowe funkcje kryptograficzne oraz dostarcza szereg narzędzi administratyjnych.

OpenSSL jest dostępny dla większości systemów w tym GNU/Linux oraz BSD.
