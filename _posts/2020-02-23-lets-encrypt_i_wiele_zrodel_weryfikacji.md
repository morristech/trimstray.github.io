---
layout: post
title: Let’s Encrypt i wiele źródeł weryfikacji
date: 2020-02-23 11:22:46
categories: [news]
tags: [PL, ssl, tls, certificates, lets-encrypt]
comments: false
favorite: false
seo:
  date_modified: 2020-02-23 12:12:11 +0100
---

Od teraz, podczas procesu weryfikacji domeny, wysyłanych będzie wiele żądań HTTP do punktu końcowego, tj. `/.well-known/acme-challenge`, co najważniejsze, **z wielu adresów IP**.

  > Co więcej, min. 3 na 4 muszą zakończyć się sukcesem, zanim certyfikat zostanie wydany!

Organizacja Let's Encrypt nie ujawnia źródłowych adresów IP punktów weryfikacyjnych, potwierdza jedynie, że każde żądanie weryfikacji wykonywane będzie z ich własnych centrów danych.

Cały proces będzie wyglądał mniej więcej tak:

<img src="/assets/img/posts/multiple-perspective-validation.png" align="center" title="multiple-perspective-validation preview">

Skąd taka zmiana?

  > At Let’s Encrypt we’re always looking for ways to improve the security and integrity of the Web PKI. We’re proud to launch multi-perspective domain validation today because we believe it’s an important step forward for the domain validation process. To our knowledge we are the first CA to deploy multi-perspective validation at scale.

I co było jej genezą? Otóż jest ona związana z... protokołem BGP oraz możliwością przejęcia lub przekierowania ruchu sieciowego wykorzystującego ścieżkę weryfikacji poprawności (np. weryfikacja DNS) podmiotu ubiegającego się o certyfikat. Zespół badawczy z Princeton wykazał możliwość przeprowadzenia takiego ataku, który został opisany w dokumencie [Bamboozling Certificate Authorities with BGP](https://www.princeton.edu/~pmittal/publications/bgp-tls-usenix18.pdf).

Dodatkowo stwierdzono, że większość wdrożeń BGP nie jest bezpieczna i może upłynąć wiele czasu, zanim możliwość przechwytywania ruchu BGP'owego będzie niemożliwa. Stąd, zamiast czekać i polegać na zewnętrznych mechanizmach zabezpieczających poczyniono kroki, aby zminimalizować wykonanie ew. ataków na ten protokół.

Więcej informacji znajduje się tutaj: [ACME v1/v2: Validating challenges from multiple network vantage points](https://community.letsencrypt.org/t/acme-v1-v2-validating-challenges-from-multiple-network-vantage-points/112253).
