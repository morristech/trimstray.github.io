---
layout: post
title: Atrybut SameSite i Chrome 80
date: 2020-02-24 07:31:05
categories: [news]
tags: [PL, http, security, pentesting, cookie, samesite, chrome]
comments: false
favorite: false
seo:
  date_modified: 2020-02-24 13:10:50 +0100
---

Na początku tego miesiąca (4 lutego 2020r.) Google wydało [nową wersję przeglądarki Chrome](https://developers.google.com/web/updates/2020/02/nic80), w której doszło do wielu ciekawych zmian. Jest ich ponad 50, a większość ma przyspieszyć, jak i zwiększyć bezpieczeństwo samej przeglądarki.

Co więcej, nowa wersja eliminuje kilka wysoko ocenianych luk w systemie CVSS, np. [CVE-2020-6383](https://borncity.com/win/2020/02/22/sicherheitsupdate-edge-80-0-361-57-21-feb-2020/), które mogą zostać wykorzystane przez atakującego w celu przejęcia kontroli nad systemem użytkownika. Podatności były na tyle poważne, że nawet Amerykańska Agencja ds. Ochrony Infrastruktury i Cyberbezpieczeństwa (CISA) wydała [powiadomienie/zalecenie](https://www.us-cert.gov/ncas/current-activity/2020/02/21/google-releases-security-updates-chrome), które „zachęca” użytkowników i administratorów do aktualizacji przeglądarki Google Chrome do najnowszej wersji.

Polecam także zapoznać się z dokumentem [Deprecations and removals in Chrome 80](https://developers.google.com/web/updates/2019/12/chrome-80-deps-rems) oraz [Google Chrome: better cookie protections and controls announced](https://www.ghacks.net/2019/05/08/google-chrome-better-cookie-protections-and-controls-announced/), w których dokładniej opisano niektóre zmiany.

# Cookie i parametr SameSite

Jedną ze zmian, która mnie osobiście bardzo zainteresowała jako administratora, jest zmiana podejścia do plików cookie, które jak wiemy, służą głównie do identyfikacji użytkowników oraz śledzenia ich poczynań w Internecie. Zmiana ta może mieć (i zapewne będzie miała) bardzo duży wpływ na działania aplikacji oraz ich integracji z serwisami/aplikacjami firm trzecich.

Zmiana związana jest z atrybutem `SameSite` i została opisana już wcześniej przez IETF w dokumencie [Incrementally Better Cookies - draft-west-cookie-incrementalism-00](https://tools.ietf.org/html/draft-west-cookie-incrementalism-00). W maju 2019 roku Google ogłosiło, że ciastka, które nie zawierają `SameSite=None` lub `SameSite=Secure` nie będą dostępne dla stron trzecich. Teraz oficjalnie Chrome jako pierwszy implementuje zachowania opisane w drafcie właśnie od wersji 80.

  > W nowej wersji przeglądarki Chrome, jeżeli nie określono atrybutu `SameSite`, cookie będą domyślnie traktowane jako posiadające atrybut `SameSite=Lax`. Przeglądarki Mozilla Firefox oraz Microsoft Edge także zapewniają wprowadzenie tej zmiany.

Przed przejściem do dalszej części, przypomnijmy sobie dwie istotne kwestia związane z ciastkami. Istnieją dwa rodzaje plików cookie — własne (ang. `same-site`) i zewnętrzne (ang. `cross-site`). Oba typy mogą zawierać te same informacje; są one jednak dostępne i tworzone inaczej:

<img src="/assets/img/posts/cookie-comparison.png" align="center" title="cookie-comparison preview">

<sup>Grafika pochodzi z serwisu [Heroku Blog](https://blog.heroku.com/chrome-changes-samesite-cookie).</sup>

Jeżeli chodzi o parametr `SameSite`, to udostępnia trzy różne sposoby kontrolowania swojego zachowania. Można nie określać atrybutu lub można użyć atrybutów `Strict`, lub `Lax`:

- `Strict` - jest to bezwzględna polityka i może rodzić różne dziwne zachowania; cookie będzie wysyłany tylko w kontekście tej samej witryny, co za tym idzie, nie będzie wysyłany w przypadku żadnych żądań między domenami (przeglądarka nie dołączy takiego ciasteczka automatycznie do żądania, które pochodzi z innej domeny; pamiętaj, że przeglądarka decyduje czy dołączyć ciastko bazując na pochodzeniu żądania), nawet jeśli użytkownik po prostu przejdzie do strony docelowej zwykłym linkiem, wtedy także plik cookie nie zostanie wysłany; jest to idealne rozwiązanie dla aplikacji, która nigdy nie musi pobierać wartości plików cookie z kontekstu zewnętrznej domeny

- `Lax` - umożliwia wysłanie (udostępnianie) ciastka podczas nawigacji z zewnętrznej witryny, ale tylko w specyficznych przypadkach — w pasku adresu musi pojawić się witryna docelowa (zmiana domeny w pasku adresu), a zapytanie HTTP musi zostać zrealizowane przez jedną z bezpiecznych metod, np. `GET` (według [RFC 7231](https://tools.ietf.org/html/rfc7231#section-4.2.1) są to dodatkowo `HEAD` oraz `TRACE`); dla żądań między domenami z metodami `POST` oraz `PUT` lub podczas ładowania witryny w ramce pochodzącej z różnych źródeł, nie będą dołączane żadne ciastka

W tej chwili w starszych wersjach Chrome domyślną wartością parametru `SameSite` jest `None`, który umożliwia zewnętrznym ciastkom śledzić użytkowników na różnych stronach. Od lutego 2020 roku wartość tego parametru zmieniona jest na `Lax`, co w skrócie oznacza, że cookie będą ustawiane tylko wtedy, gdy domena w adresie URI odpowiada domenie, z którego pochodzi ciastko.

Atrybut ten ([Same-site Cookies - draft-west-first-party-cookies-07](https://tools.ietf.org/html/draft-west-first-party-cookies-07)) pozwala zadeklarować, czy twoje ciastko powinno być ograniczone do kontekstu pierwszej lub tej samej witryny. Tym samym zapewnia, że dane ciasteczko może być wysyłane wyłącznie z żądaniami zainicjowanymi z domeny, dla której zostało zarejestrowane, a nie z zewnętrznych domen.

  > Na podstawie danych zebranych przez serwis [Can I use](https://caniuse.com/#feat=same-site-cookie-attribute), plik cookie z atrybutem `SameSite` ma już globalną obsługę 86,58% przeglądarek.

Wprowadzona modyfikacja zapewnia też bardzo solidną ochronę przed atakami polegającymi na fałszowaniu żądań między witrynami ([Cross-site request forgery (CSRF)](https://portswigger.net/web-security/csrf)), które defacto, nie są już w pierwszej dziesiątce OWASP Top 10.

Zapoznaj się ze świetnym wyjaśnieniem [Flaga cookies SameSite – jak działa i przed czym zapewnia ochronę?](https://sekurak.pl/flaga-cookies-samesite-jak-dziala-i-przed-czym-zapewnia-ochrone/), [SameSite cookies explained](https://web.dev/samesite-cookies-explained/), a także [Chrome's Changes Could Break Your App: Prepare for SameSite Cookie Updates](https://blog.heroku.com/chrome-changes-samesite-cookie). Każdy szczegółowo wyjaśnia działanie tego parametru. Polecam także podcast [Jak działa flaga SameSite cookie?](https://podtail.com/it/podcast/kacper-szurek/jak-dzia-a-flaga-samesite-cookie/).

# Jakie rodzi to konsekwencje dla aplikacji?

Aktualizacja parametru `SameSite` może wymagać od architektów i developerów wprowadzenia zmian w aplikacji. Jednym z zaleceń jest sprawdzenie wykorzystywanych zapytań między serwisami, które wymagają przesłania cookie. Jeżeli aplikacja nie korzysta z żądań pochodzących z różnych zewnętrznych serwisów, nadal należy podjąć pewne działania (wyeliminować wykorzystanie protokołu HTTP, a także sprawdzić wszelkie niestandardowe integracje oparte na ciastkach).

Jednym z najlepszych dokumentów opisujących ew. problemy i rozwiązania jest [Upcoming Browser Behavior Changes: What Developers Need to Know](https://auth0.com/blog/browser-behavior-changes-what-developers-need-to-know/).

  > W PHP 7.3 dodano obsługę flagi `SameSite` za pomocą dyrektywy `session.cookie_samesite=Lax`. W Django istnieje możliwość ustawienia tego atrybutu od wersji 2.1.x.

Dobrym pomysłem jest także zapoznanie się z [Chrome’s SameSite Cookie Update – What You Need to Do?](https://headerbidding.co/chrome-samesite-cookie-update/), który pokazuje na przykładach zalecenia oraz kroki, jakie należy podjąć, w przypadku wykorzystywania zewnętrznych partnerów takich jak Facebook. Dodatkowo polecam repozytorium [GoogleChromeLabs/samesite-examples](https://github.com/GoogleChromeLabs/samesite-examples), które zawiera przykłady użycia atrybutu `SameSite` w różnych językach, bibliotekach i frameworkach.

Zmiana ta bardzo mocno ograniczy możliwość śledzenia użytkowników przez serwisy zewnętrzne, możliwość wykonania ataku CSRF, a także ewentualnych wycieków danych.

  > Dodatkowe zalecenia dla architektów web-aplikacji opisane zostały w [SameSite cookie recipes](https://web.dev/samesite-cookie-recipes/) oraz na oficjalnym blogu Chromium — [Developers: Get Ready for New SameSite=None; Secure Cookie Settings](https://blog.chromium.org/2019/10/developers-get-ready-for-new.html).

Oczywiście nadal istnieje możliwość użycia `SameSite=None` (będzie działało niemalże identycznie jak te przed opisywanymi zmianami) jednak od teraz musi przyjąć dodatkowo wartość `Secure` (czyli `SameSite=None; Secure`), która oznacza, że ciastko będzie wysyłane do serwera tylko wtedy, gdy żądanie zostanie przesłane za pomocą protokołu HTTPS.

  > Serwisy wykorzystujące protokół HTTP nie mogą ustawiać plików cookie z parametrem `Secure` od wersji Chrome 52+ oraz Firefox 52+).

Zerknij na poniższą ściągę, która z drugiej strony uzmysławia ew. problemy, z jakimi będzie można się zmierzyć:

<img src="/assets/img/posts/chrome_80_samesite_recommendations.png" align="center" title="chrome_80_samesite_recommendations preview">

<sup>Grafika pochodzi z serwisu [adzerk.com](https://adzerk.com/blog/chrome-samesite/).</sup>

Jak widać, parametr `SameSite` wnosi istotny wkład w dziedzinie ochrony przed atakami, których skutkiem może być wyciek danych pomiędzy różnymi domenami. Wprowadzona implementacja po stronie przeglądarek pozwoli zminimalizować ew. pomyłki przez brak jawnej kontroli ciastek.

Na koniec warto przeczytać:

- [CSRF is (really) dead](https://scotthelme.co.uk/csrf-is-really-dead/)
- [Bypass SameSite Cookies Default to Lax and get CSRF](https://medium.com/@renwa/bypass-samesite-cookies-default-to-lax-and-get-csrf-343ba09b9f2b)
- [Cross-Site Request Forgery (CSRF) Prevention Cheat Sheet](https://owasp.org/www-project-cheat-sheets/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
