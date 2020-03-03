---
layout: post
title: 'OpenSSH: Wskazanie metod autoryzacji'
date: 2015-07-19 22:39:18
categories: [publications]
tags: [PL, system, openssh]
comments: false
favorite: false
seo:
  date_modified: 2020-02-20 19:32:56 +0100
---

Domyślną metodą autoryzacji z poziomu klienta SSH jest logowanie za pomocą loginu i hasła, jednak oczywiście istnieje możliwość autoryzacji za pomocą klucza.

Podczas nawiązywania połączenia klient sprawdza czy istnieje klucz i na jego podstawie ustala sposób logowania. Jeżeli taka metoda się nie powiedzie podejmowana jest próba autoryzacji za pomocą loginu i hasła.

Jeżeli zależy Nam na wymuszeniu jednej z metod możemy przekazać parametry wywołania z poziomu klienta.

# Wymuszenie logowania za pomocą hasła

Jeżeli mamy problem z logowaniem za pomocą klucza można spróbować wymusić logowanie za pomocą hasła (standardowy sposób). Powiedzie się on jedynie w przypadku pozostawienia takiej możliwości autoryzacji po stronie serwera:

```bash
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no user@remote_host
```

# Wymuszenie logowania za pomocą klucza

Jeżeli chcielibyśmy wykonać sytuację odwrotną czyli wymusić na kliencie logowanie za pomocą klucza:

```bash
ssh -o PreferredAuthentications=publickey -o PubkeyAuthentication=yes -i id_rsa user@remote_host
```
