---
layout: post
title: 'Czy certyfikat wildcard obsługuje domenę główną?'
date: 2019-11-29 08:27:09
categories: [publications]
tags: [PL, ssl, tls, certificates, wildcard]
comments: false
favorite: false
seo:
  date_modified: 2020-02-23 09:14:19 +0100
---

Nie, nie jest możliwa obsługa takiej domeny. Przeanalizujmy ten problem na przykładzie domeny `example.com` oraz certyfikatu wildcard wystawionego tylko dla `*.example.com`.

Symbol wieloznaczny w nazwie odzwierciedla tylko jedną etykietę i można pozostawić go tylko od lewej strony. W takim przypadku (domyślnie), symbol wieloznaczny jest ważny tylko dla `sub.example.com` ale już nie dla `www.subdomain.example.com` ani `example.com`. Zgodnie z tym, `*.*.example.com` lub `www.*.example.com` także nie będą chronione certyfikatem wildcard wystawionym dla `*.example.com`.

  > Aby zabezpieczyć samą nazwę domeny i hosty w domenie, musisz uzyskać certyfikat z nazwami w rozszerzeniu SAN.

Z technicznego punktu widzenia certyfikaty typu wildcard wydawane są na podstawie „nieznanych potomków” poddomeny. Większość certyfikatów wieloznacznych wydawanych jest dla domen 3-częściowych (`*.example.com`), ale można je spotkać w przypadku domen 4-częściowych (np. `*.example.co.uk`).

Dokładna odpowiedź na zadane pytanie znajduje się w [RFC 2818 - Server Identity](https://tools.ietf.org/html/rfc2818#section-3.1) <sup>[IETF]</sup>:

  > _Matching is performed using the matching rules specified by RFC 2459. If more than one identity of a given type is present in the certificate (e.g., more than one dNSName name, a match in any one of the set is considered acceptable.) Names may contain the wildcard character `*` which is considered to match any single domain name component or component fragment. E.g., `*.a.com` matches `foo.a.com` but not `bar.foo.a.com`. `f*.com` matches `foo.com` but not `bar.com`._

Co więcej, [RFC 2459 - Server Identity Check](https://tools.ietf.org/html/rfc2595#section-2.4) <sup>[IETF]</sup> mówi:

  > _A "`*`" wildcard character MAY be used as **the left-most name component** in the certificate.  For example, `*.example.com` would match `a.example.com`, `foo.example.com`, etc. but **would not match** `example.com`._

Jak widzisz, standardy mówią, że `*` powinien pasować do minimum jednego znaku bez kropek. Dlatego domena główna musi być alternatywną nazwą, aby mogła być chroniona tym samym certyfikatem.

W przypadku certyfikatu dla `*.example.com`:

- `a.example.com` będzie obsługiwany
- `www.example.com` będzie obsługiwany
- `example.com` nie będzie obsługiwany
- `a.b.example.com` nie będzie obsługiwany

Czasami niektórzy dostawcy automatycznie dodają domenę główną jako alternatywną nazwę podmiotu (pole `SAN`) do wieloznacznego certyfikatu SSL, np.:

```bash
issuer: RapidSSL RSA CA 2018 (DigiCert Inc)
cn: example.com
san: *.example.com example.com
```

Inną interesującą rzeczą jest to, że możesz mieć wiele nazw z symbolami wieloznacznymi w tym samym certyfikacie, tzn. możesz mieć `*.example.com` i `*.subdomain.example.com` obsługiwane z poziomu tego samego certyfikatu. Powinieneś nie mieć problemu ze znalezieniem urzędu certyfikacji, który wyda taki certyfikat, a większość klientów powinna go zaakceptować.
