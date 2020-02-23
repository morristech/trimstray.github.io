---
layout: post
title: 'Let’s Encrypt i wiele źródeł weryfikacji'
date: 2020-02-23 11:22:46
categories: [news]
tags: [PL, ssl, tls, certificates, lets-encrypt]
comments: false
favorite: false
seo:
  date_modified: 2020-02-19 08:43:17 +0100
---

Od teraz, podczas procesu weryfikacji domeny, wysyłanych będzie wiele żądań HTTP do punktu końcowego `/.well-known/acme-challenge`, co kluczowe, **z różnych adresów IP**.

  > Co więcej, min. 3 na 4 muszą zakończyć się sukcesem, zanim certyfikat zostanie wydany!

Wyglądało będzie to tak:

<img src="/assets/img/posts/multiple-perspective-validation.png" align="center" title="multiple-perspective-validation preview">

Skąd taka zmiana?

  > At Let’s Encrypt we’re always looking for ways to improve the security and integrity of the Web PKI. We’re proud to launch multi-perspective domain validation today because we believe it’s an important step forward for the domain validation process. To our knowledge we are the first CA to deploy multi-perspective validation at scale.

Więcej informacji o tej zmianie można poczytać tutaj: [ACME v1/v2: Validating challenges from multiple network vantage points](https://community.letsencrypt.org/t/acme-v1-v2-validating-challenges-from-multiple-network-vantage-points/112253).
