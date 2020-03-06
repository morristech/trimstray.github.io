---
layout: post
title: 'NGINX: Bezpieczeństwo protokołu TLS'
date: 2020-03-06 07:39:05
categories: [publications]
tags: [PL, http, https, nginx, tls, best-practices]
comments: false
favorite: false
seo:
  date_modified: 2020-02-23 09:14:19 +0100
---

Istnieje wiele potencjalnych zagrożeń związanych z wdrażaniem konfiguracji protokołu TLS. Każda konfiguracja wykorzystująca ten protokół powinna spełniać zalecenia poprawnej implementacji i być zgodna ze standardami branżowymi. Wydawać by się mogło, że konfiguracja TLS jest czynnością bardzo prostą i wdrożenie nie powinno być wymagającym procesem. Nic bardziej mylnego.

W tym wpisie chciałbym przedstawić, na przykładzie serwera NGINX, na co powinniśmy zwracać uwagę oraz dlaczego pewne parametry powinny być ustawione tak a nie inaczej.

# Oddzielne dyrektywy nasłuchiwania dla portów 80 i 443

Pierwszą zasadą od której rozpoczniemy będzie sprawa organizacyjna (w kontekście konfiguracji). NGINX umożliwia definiowanie listener'ów za pomocą których procesy nasłuchują na odpowienich adresach IP oraz portach.

Jeśli w twojej konfiguracji wykorzystujesz oba protokoły, tj. HTTP i HTTPS z dokładnie taką samą konfiguracją (pojedynczy serwer, który obsługuje zarówno żądania HTTP, jak i HTTPS), NGINX jest wystarczająco inteligentny, aby zignorować dyrektywy TLS, jeśli zostanie załadowany przez port 80.

Nie lubię powielać reguł, ale osobne dyrektywy `listen` z pewnością pomogą ci utrzymać i modyfikować konfigurację. Jeśli chcę przekierować ruch z HTTP na HTTPS, właściwym dla mnie sposobem jest zdefiniowanie oddzielnego kontekstu `server` w takich przypadkach.

Jest to również przydatne, jeśli przypniesz wiele domen do jednego adresu IP. Pozwala to dołączyć jedną dyrektywę `listen` (np. jeśli zachowasz ją w pliku konfiguracyjnym) do konfiguracji wielu domen.

  > Powinieneś także użyć dyrektywy `return` do przekierowania z HTTP na HTTPS (aby wszystko zakodować na sztywno, a nie używać w ogóle wyrażeń regularnych).

Przykład konfiguracji:

```nginx
# For HTTP:
server {

  listen 10.240.20.2:80;

  # If you need redirect to HTTPS:
  return 301 https://example.com$request_uri;

  ...

}

# For HTTPS:
server {

  listen 10.240.20.2:443 ssl;

  ...

}
```
