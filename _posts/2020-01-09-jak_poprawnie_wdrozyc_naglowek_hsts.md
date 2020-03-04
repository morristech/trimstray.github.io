---
layout: post
title: 'Jak poprawnie wdrożyć nagłówek HSTS?'
date: 2020-01-09 19:01:56
categories: [publications]
tags: [PL, http, https, security, headers, hsts]
comments: false
favorite: false
seo:
  date_modified: 2020-02-23 09:14:19 +0100
---

Nagłówek HSTS (opisany w [RFC 6797](https://tools.ietf.org/html/rfc6797)) jest jednym z najważniejszych nagłówków bezpieczeństwa. W dużym skrócie zapobiega on korzystaniu z niezabezpieczonych połączeń HTTP i wymusza użycie protokołu TLS. W tym wpisie po krótce omówię sam nagłówek oraz skupię się mocno na samej procedurze jego poprawnej implementacji.

Zasadniczo HSTS (_HTTP Strict Transport Security_) pozwala stronom internetowym (aplikacjom) informować przeglądarki, że połączenie powinno być zawsze szyfrowane (wykorzystujące protokół HTTPS). Pozwala on zapobiegać atakom MITM, atakom typu downgrade a także wysyłaniu plików cookie i identyfikatorów sesji niezaszyfrowanym kanałem. Prawidłowe wdrożenie HSTS to dodatkowy mechanizm bezpieczeństwa zgodny z zasadą bezpieczeństwa wielowarstwowego (ang. _defense in depth_).

Co ciekawe, nagłówek ten jest świetny pod względem wydajności, ponieważ instruuje przeglądarkę, aby po stronie klienta przeprowadzała przekierowanie HTTP na HTTPS bez dotykania warstwy serwera.

  > Jedną z ważniejszych informacji o tym nagłówku jest to, że wskazuje on, jak długo przeglądarka powinna bezwarunkowo odmawiać udziału w niezabezpieczonym połączeniu HTTP dla określonej domeny.

Gdy przeglądarka wie, że domena włączyła HSTS, robi dwie rzeczy:

  - zawsze używa połączenia `https://`, nawet po kliknięciu linku `http://` lub po wpisaniu domeny w pasku lokalizacji bez określania protokołu
  - usuwa możliwość zatwierdzania ostrzeżeń o nieważnych certyfikatach

Jeżeli chcemy ustawić ten nagłówek z poziomu serwera NGINX, należy pamiętać o ustawieniu go w w bloku `http` z opcją `ssl` dla danej konfiguracji nasłuchiwania — w przeciwnym razie ryzykujesz wysłanie nagłówka `Strict-Transport-Security` przez połączenie HTTP, które również mogłeś skonfigurować w innym bloku konfiguracji. Dodatkowo powinieneś użyć przekierowania 301 za pomocą `return 301`, aby blok serwera HTTP został przekierowany do HTTPS.

Nagłówek ten powinien być zawsze ustawiony z parametrem `includeSubdomains`. Zapewni to solidne bezpieczeństwo zarówno dla głównej domeny, jak i wszystkich subdomen. Problem polega na tym, że (bez `includeSubdomains`) atakujący, który przeprowadza atak man-in-the-middle może stworzyć dowolne subdomeny i używać ich do wstrzykiwania plików cookie do aplikacji.

Wadą tego parametru jest oczywiście to, że będziesz musiał wdrożyć wszystkie subdomeny za pośrednictwem protokołu SSL (jednak obecnie powinno to być standardem!).

# Na co uważać przy wdrażaniu nagłówka HSTS?

Wdrożenie nagłówka HSTS powinno być obowiązkowym krokiem jednak musi zostać zrobione z głową. Niestety wiele artykułów pomija dobre praktyki związane z przeprowadzeniem jego prawidłowej implementacji i skupia się na samych zaleceniach jego włączenia.

Myślę, że najlepszym howto jak to zrobić są zalecenia firmy Qualys z dokumentu [The Importance of a Proper HTTP Strict Transport Security Implementation on Your Web Server](https://blog.qualys.com/securitylabs/2016/03/28/the-importance-of-a-proper-http-strict-transport-security-implementation-on-your-web-server). Jest to świetne wyjaśnienie dlatego pozwolę je sobie zacytować:

- The strongest protection is to ensure that all requested resources use only TLS with a well-formed HSTS header. Qualys recommends providing an HSTS header on all HTTPS resources in the target domain

- It is advisable to assign the `max-age` directive’s value to be greater than `10368000` seconds (120 days) and ideally to `31536000` (one year). Websites should aim to ramp up the `max-age` value to ensure heightened security for a long duration for the current domain and/or subdomains

- [RFC 6797 - The Need for includeSubDomains](https://tools.ietf.org/html/rfc6797) <sup>[IETF]</sup>, advocates that a web application must aim to add the `includeSubDomain` directive in the policy definition whenever possible. The directive’s presence ensures the HSTS policy is applied to the domain of the issuing host and all of its subdomains, e.g. `example.com` and `www.example.com`

- The application should never send an HSTS header over a plaintext HTTP header, as doing so makes the connection vulnerable to SSL stripping attacks

- It is not recommended to provide an HSTS policy via the `http-equiv` attribute of a meta tag. According to [RFC 6797](https://tools.ietf.org/html/rfc6797) <sup>[IETF]</sup>, user agents don’t heed `http-equiv="Strict-Transport-Security"` attribute on `<meta>` elements on the received content

Co ważne, przed wdrożeniem tego nagłówka koniecznie zapoznaj się z zaleceniami firmy Google, która zaleca włączenie HSTS zgodnie z poniższą procedurą, ponieważ nieprzemyślane włączenie zwiększa znacznie strategię wycofywania:

- bądź pewny, że twoja witryna rzeczywiście w całości działa z wykorzystaniem protokołu HTTPS
- opublikuj najpierw swoją aplikację z wykorzystaniem protokołu HTTPS bez nagłówka HSTS
- zacznij wysyłać nagłówki HSTS z niską wartością parametru `max-age`. Monitoruj ruch zarówno użytkowników, jak i innych klientów, a także wydajność
- powoli zwiększaj wartość parametru `max-age`
- jeśli HSTS nie wpływa negatywnie na użytkowników i wyszukiwarki, możesz poprosić o dodanie Twojej witryny do tzw. listy wstępnego ładowania HSTS używanej przez większość głównych przeglądarek
