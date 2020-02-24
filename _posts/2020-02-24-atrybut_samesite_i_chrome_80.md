---
layout: post
title: Atrybut SameSite i Chrome 80
date: 2020-02-24 07:31:05
categories: [news]
tags: [PL, http, security, pentesting, cookie, samesite, chrome]
comments: false
favorite: false
seo:
  date_modified: 2020-02-23 12:30:23 +0100
---

Na początku tego miesiąca (4 lutego 2020r.) Google wydało [nową wersję przeglądarki Chrome](https://developers.google.com/web/updates/2020/02/nic80), w której doszło do wielu ciekawych zmian. Jest ich ponad 50, a większość ma przyspieszyć, jak i zwiększyć bezpieczeństwo samej przeglądarki.

Co więcej, nowa wersja eliminuje kilka wysoko ocenianych luk w systemie CVSS, np. [CVE-2020-6383](https://borncity.com/win/2020/02/22/sicherheitsupdate-edge-80-0-361-57-21-feb-2020/), które mogą zostać wykorzystane przez atakującego w celu przejęcia kontroli nad systemem użytkownika. Podatności były na tyle poważne, że nawet Amerykańska Agencja ds. Ochrony Infrastruktury i Cyberbezpieczeństwa (CISA) wydała [powiadomienie/zalecenie](https://www.us-cert.gov/ncas/current-activity/2020/02/21/google-releases-security-updates-chrome), które „zachęca” użytkowników i administratorów do aktualizacji przeglądarki Google Chrome do najnowszej wersji.

Polecam także zapoznać się z dokumentem [Deprecations and removals in Chrome 80](https://developers.google.com/web/updates/2019/12/chrome-80-deps-rems), w którym dokładniej opisano, co zostało dodane a co usunięte.

# Cookie i parametr SameSite

Jedną ze zmian, która mnie osobiście bardzo zainteresowała jako administratora, jest zmiana podejścia do plików cookie, które jak wiemy, służą głównie do identyfikacji użytkowników oraz śledzenia ich poczynań w Internecie.

Zmiana związana jest z atrybutem `SameSite` i została opisana już wcześniej przez IETF w dokumencie [Incrementally Better Cookies - draft-west-cookie-incrementalism-00](https://tools.ietf.org/html/draft-west-cookie-incrementalism-00). W maju 2019 roku Google ogłosiło, że ciastka, które nie zawierają `SameSite=None` lub `SameSite=Secure` nie będą dostępne dla stron trzecich. Teraz oficjalnie Chrome jako pierwszy implementuje zachowania opisane w drafcie właśnie od wersji 80.

  > W nowej wersji przeglądarki Chrome, jeżeli nie określono atrybutu `SameSite`, cookie będą domyślnie traktowane jako posiadające atrybut `SameSite=Lax`. Przeglądarki Mozilla Firefox oraz Microsoft Edge także zapewniają wprowadzenie tej zmiany.

Przed przejście do dalszej części, przypomnijmy sobie dwie istotne kwestia związane z ciastkami. Istnieją dwa rodzaje plików cookie — własne (`same-site`) i zewnętrzne (`cross-site`). Oba typy mogą zawierać te same informacje; są one jednak dostępne i tworzone inaczej:

<img src="/assets/img/posts/cookie-comparison.png" align="center" title="cookie-comparison preview">

Jeżeli chodzi o zmianę w przeglądarce Chrome. Parametr `SameSite` udostępnia trzy różne sposoby kontrolowania swojego zachowania. Można nie określać atrybutu lub można użyć atrybutów `Strict`, lub `Lax`:

- `Strict` - jest to bezwzględna polityka, cookie będzie wysyłany tylko w konktekście tej samej witryny co za tym idzie, nie będzie wysyłany w przypadku żadnych żądań między domenami (przeglądarka nie dołączy takiego ciasteczka automatycznie do żądania, które pochodzi z innej domeny; pamiętaj, że przeglądarka decyduje czy dołączyć ciastko bazując na pochodzeniu żądania), nawet jeśli użytkownik po prostu przejdzie do strony docelowej zwykłym linkiem, plik cookie nie zostanie wysłany (wartość ta może rodzić dziwne zachowania)

- `Lax` - umożliwia wysłanie (udostępnianie) ciastka podczas nawigacji z zewnętrznej witryny, ale tylko w specyficznych przypadkach — w pasku adresu musi pojawić się witryna docelowa (zmiana domeny w pasku adresu), a zapytanie HTTP musi zostać zrealizowane przez jedną z bezpiecznych metod, np. `GET` (według [RFC 7231](https://tools.ietf.org/html/rfc7231#section-4.2.1) są to dodatkowo `HEAD` oraz `TRACE`); dla żądań między domenami z metodami `POST` oraz `PUT` lub podczas ładowania witryny w ramce pochodzącej z różnych źródeł, nie będą dołączane żadne ciastka

W tej chwili w starszych wersjach Chrome domyślną wartością parametru `SameSite` jest `None`, który umożliwia zewnętrznym ciastkom śledzić użytkowników na różnych stronach. Od lutego 2020 roku wartość tego parametru zmieniona jest na `Lax`, co w skrócie oznacza, że cookie będą ustawiane tylko wtedy, gdy domena w adresie URI odpowiada domenie, z którego pochodzi ciastko.

Atrybut ten ([Same-site Cookies - draft-west-first-party-cookies-07](https://tools.ietf.org/html/draft-west-first-party-cookies-07)) pozwala zadeklarować, czy twoje ciastko powinno być ograniczone do kontekstu pierwszej lub tej samej witryny. Tym samym zapewnia, że dane ciasteczko może być wysyłane wyłącznie z żądaniami zainicjowanymi z domeny, dla której zostało zarejestrowane, a nie z zewnętrznych domen.

  > Na podstawie danych zebranych przez serwis [Can I use](https://caniuse.com/#feat=same-site-cookie-attribute), plik cookie z atrybutem `SameSite` ma już globalną obsługę 86,58% przeglądarek.

Wprowadzona modyfikacja zapewnia też bardzo solidną ochronę przed atakami polegającymi na fałszowaniu żądań między witrynami ([Cross-site request forgery (CSRF)](https://portswigger.net/web-security/csrf)), które defacto, nie są już w pierwszej dziesiątce OWASP Top 10.

Zapoznaj się ze świetnym wyjaśnieniem [Flaga cookies SameSite – jak działa i przed czym zapewnia ochronę?](https://sekurak.pl/flaga-cookies-samesite-jak-dziala-i-przed-czym-zapewnia-ochrone/) oraz [SameSite cookies explained](https://web.dev/samesite-cookies-explained/). Oba szczegółowo wyjaśniają działanie tego parametru. Polecam także podcast [Jak działa flaga SameSite cookie?](https://podtail.com/it/podcast/kacper-szurek/jak-dzia-a-flaga-samesite-cookie/).

# Jakie rodzi to konsekwencje dla architektów aplikacji?

Aktualizacja `SameSite` będzie wymagać od architektów i developerów **wyraźnego oznaczenia wszystkich plików cookie stron trzecich**, które mogą być używane. Pliki cookie bez odpowiedniego parametru nie będą działać w nowej wersji przeglądarki Chrome.

  > W PHP 7.3 dodano obsługę flagi `SameSite` za pomocą dyrektywy `session.cookie_samesite=Lax`. W Django istnieje możliwość ustawienia tego atrybutu od wersji 2.1.x.

Zmiana ta bardzo mocno ograniczy możliwość śledzenia użytkowników przez serwisy zewnętrzne, możliwość wykonania ataku CSRF, a także ewentualnych wycieków danych.

  > Dodatkowe zalecenia dla architektów web-aplikacji opisane zostały [tutaj](https://web.dev/samesite-cookie-recipes/) oraz [tutaj](https://blog.chromium.org/2019/10/developers-get-ready-for-new.html).

Oczywiście nadal istnieje możliwość użycia `SameSite=None` (będzie działało niemalże identycznie jak te przed opisywanymi zmianami) jednak od teraz musi przyjąć dodatkowo wartość `Secure` (czyli `SameSite=None; Secure`), która oznacza, że ciastko będzie wysyłane do serwera tylko wtedy, gdy żądanie zostanie przesłane za pomocą protokołu HTTPS.

  > Serwisy wykorzystujące protokół HTTP nie mogą ustawiać plików cookie z parametrem `Secure` od wersji Chrome 52+ oraz Firefox 52+).

Jak widać, parametr `SameSite` wnosi istotny wkład w dziedzinie ochrony przed atakami, których skutkiem może być wyciek danych pomiędzy różnymi domenami. Wprowadzona implementacja po stronie przeglądarek pozwoli zminimalizować ew. pomyłki przez brak jawnej kontroli ciastek. Na koniec warto zapoznać się z artykułem [Bypass SameSite Cookies Default to Lax and get CSRF](https://medium.com/@renwa/bypass-samesite-cookies-default-to-lax-and-get-csrf-343ba09b9f2b) a także [CSRF is (really) dead](https://scotthelme.co.uk/csrf-is-really-dead/).
