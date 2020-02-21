---
layout: post
title: 'NGINX: Przetwarzanie żądań a niezdefiniowane nazwy serwerów'
date: 2018-04-19 12:42:51
categories: [publications]
tags: [PL, http, nginx, best-practices]
comments: false
favorite: false
seo:
  date_modified: 2020-02-18 22:13:20 +0100
---

Jak wiemy, nagłówek `Host` informuje serwer, którego wirtualnego hosta ma użyć. Nagłówek ten można również modyfikować, co może pozwolić na ominięcie filtrów lub przekazanie żądań do nieodpowiednich backend'ów. Ze względów bezpieczeństwa dobrą praktyką jest odrzucanie żądań bez hosta lub z hostami nieskonfigurowanymi po stronie serwera HTTP.

Zgodnie z tym NGINX powinien zapobiegać przetwarzaniu żądań (także na adres IP) z nieokreślonymi nazwami serwerów. Rozwiązaniem problemu jest ustawienie dyrektywy `listen` z jawnym wskazaniem parametru `default_server`. Jeśli żadna z dyrektyw `listen` nie ma parametru `default_server`, wówczas pierwszy blok `server {...}` z parą `listen adres:port` będzie serwerem domyślnym (oznacza to, że NGINX zawsze ma domyślny serwer).

W rzeczywistości `default_server` nie potrzebuje instrukcji `server_name`, ponieważ pasuje do wszystkiego, do czego inne bloki serwera nie pasują jawnie.

Jeśli nie można znaleźć serwera z pasującym parametrem `listen` i `server_name`, NGINX użyje serwera domyślnego. Jeśli konfiguracje są rozłożone na wiele plików, kolejność oceny będzie niejednoznaczna, dlatego należy wyraźnie zaznaczyć, który z bloków obsługuje żądania nie zdefiniowane nigdzie indziej.

Przykład:

```nginx
# Umieść na początku konfiguracji:
server {

  # Dla obsługi SSL pamiętaj o odpowiedniej konfiguracji;
  # Dodając default_server to dyrektywy listen w kontekście server mówisz,
  # żeby NGINX traktował ten blok jako domyślny:
  listen 10.240.20.2:443 default_server ssl;

  # Za pomocą poniższej dyrektywy obsługujemy:
  #   - niepoprawne domeny (nieobsługiwane przez NGINX)
  #   - requesty bez nagłówka "Host"
  # Pamiętaj, że wartość default_server w dyrektywie server_name nie jest wymagany,
  # co więcej dyrektywy server_name może nie być w ogóle (a jeśli jest
  # może zawierać cokolwiek).
  server_name _ "" default_server;

  # Dodatkowo ustawiamy limitowanie:
  limit_req zone=per_ip_5r_s;

  ...

  # Zamykamy połączenie wewnętrznie (bez zwracania odpowiedzi do klienta):
  return 444;

  # Można także zaserwować klientowi stronę statyczną lub przekierować go
  # w inny miejsce:
  # location / {
  #
  #   static file (error page):
  #     root /etc/nginx/error-pages/404;
  #   or redirect:
  #     return 301 https://badssl.com;
  #
  # }

  # Pamiętaj o logowaniu takich akcji:
  access_log /var/log/nginx/default-access.log main;
  error_log /var/log/nginx/default-error.log warn;

}

server {

  listen 10.240.20.2:443 ssl;

  server_name example.com;

  ...

}

server {

  listen 10.240.20.2:443 ssl;

  server_name domain.org;

  ...

}
```
