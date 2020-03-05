---
layout: post
title: 'NGINX: Blokowanie niebezpiecznych metod'
date: 2018-11-19 06:28:11
categories: [publications]
tags: [PL, http, nginx, best-practices, methods]
comments: false
favorite: false
seo:
  date_modified: 2020-02-23 09:14:19 +0100
---

Zwykły serwer HTTP obsługuje metody `GET`, `HEAD` i `POST` w celu pobierania zawartości statycznej i dynamicznej. Inne metody (np. `OPTIONS`, `TRACE`) nie powinny być obsługiwane na publicznych serwerach HTTP, ponieważ zwiększają powierzchnię ewentualnego ataku.

Niektóre z tych metod są zazwyczaj niebezpieczne, a inne są po prostu niepotrzebne w środowisku produkcyjnym. Według mnie warto je wyłączyć, ponieważ prawdopodobnie nie będziesz ich potrzebować. Listę dostępnych metod znajdziesz w dokumencie [Hypertext Transfer Protocol (HTTP) Method Registry](https://www.iana.org/assignments/http-methods/http-methods.xhtml) <sup>[IANA]</sup>.

  > Możesz także spróbować ominąć mechanizmy filtrujące aplikacji lub serwera HTTP, wysyłając dowolne ciągi znaków, takie jak `gEt` lub nawet `AsdF`, jako metody HTTP przekazane w żądaniu.

Dodatkowo polecam zaznajomić się z poniższymi zasobami:

- [Vulnerability name: Unsafe HTTP methods](https://www.onwebsecurity.com/security/unsafe-http-methods.html)
- [Penetration Testing Of A Web Application Using Dangerous HTTP Methods](https://www.sans.org/reading-room/whitepapers/testing/penetration-testing-web-application-dangerous-http-methods-33945) <sup>[pdf]</sup>

Obsługa metody `TRACE` może umożliwić atak typu Cross-Site Tracing, który może ułatwić przechwycenie identyfikatora sesji innego użytkownika aplikacji. Ponadto, tej metody można użyć do próby zidentyfikowania dodatkowych informacji o środowisku, w którym działa aplikacja (np. istnienie serwerów cache'ujących na ścieżce do aplikacji).

Moim zdaniem, obsługa metody `OPTIONS` nie jest bezpośrednim zagrożeniem, ale może być źródłem dodatkowych informacji dla atakującego, które mogą ułatwić przeprowadzenie skutecznego ataku.

  > Niektóre interfejsy API (np. RESTful API) korzystają również z innych metod. Oprócz ochrony po stronie serwera architekci aplikacji powinni również weryfikować przychodzące żądania.

Co ciekawe, obsługa metody `HEAD` jest również ryzykowna (naprawdę!), ponieważ może przyspieszyć proces ataku, ograniczając ilość danych wysyłanych z serwera. Co wiecej, może być wykorzystana do zaatakowania aplikacji poprzez tzw. Mimicking Attack, polegający na naśladowaniu metody `GET`, dzięki czemu można pominąć mechanizmu uwierzytelniania. Wiele artykułów uznaje metodę `HEAD` za w pełni bezpieczną jednak należy mieć świadomość ewentualnych zagrożeń.

  > Jeśli mechanizmy autoryzacji są oparte na `GET` i `POST`, metoda `HEAD` może także pozwolić na ominięcie tych zabezpieczeń.

Użycie metody `HEAD` jest jednak bardzo przydatne do wyszukiwania meta informacji zapisanych w nagłówkach odpowiedzi, bez konieczności przenoszenia jej całej zawartości. Na przykład pozwala na skuteczne określenie, czy dany zasób uległ zmianie bez zwiększania kosztu (ruchu w sieci), jaki wykonany by został przy pomocy metody `GET` (z drugiej strony atakujący mogą uzyskać te same informacje także za pomocą właśnie metody `GET`).

  > Bardzo prawdopodobne jest, że przeglądarki lub wyszukiwarki używają żądań `HEAD`, aby sprawdzić, czy ich wersje stron w pamięci podręcznej są nadal aktualne. Jeśli nie, to logicznym byłoby takie wykorzystanie tej metody.

Zanim zaczniesz blokować potencjalnie niebezpieczne metody, chciałbym zwrócić uwagę na jeszcze jedną pomijaną rzecz. Otóż jaki kod błędu powinien zostać zwrócony w przypadku żądania z wykorzystaniem jednej z niedozwolonych przez twój serwer metod?

Odpowiedź nie jest wcale taka oczywista (z jednej strony, ponieważ mam nadzieję, że każdy pomyślał o kodzie 405) i jest związana z kolejnością kodów odpowiedzi, które powinny zostać zwrócone przez poprawnie napisany i skonfigurowany serwer HTTP — czyli w pełni zgodny z [RFC7230](https://tools.ietf.org/html/rfc7230), [RFC7231](https://tools.ietf.org/html/rfc7231), [RFC7232](https://tools.ietf.org/html/rfc7232), [RFC7233](https://tools.ietf.org/html/rfc7233), [RFC7234](https://tools.ietf.org/html/rfc7234) a także [RFC7235](https://tools.ietf.org/html/rfc7235).

Spójrz na poniższe wyjaśnie opisujące różnice oraz pierwszeństwo typowej implementacji:

- 0: Przychodzi żądanie...
- 1: `405 Method Not Allowed` oznacza, że serwer nie zezwala na tę metodę dla tego identyfikatora URI
- 2: `401 Unauthorized` oznacza, że użytkownik nie jest uwierzytelniony
- 3: `403 Forbidden` oznacza, że klient uzyskujący dostęp nie jest upoważniony do wykonania tego żądania

Zgodnie z tym, odpowiednim kodem odpowiedzi jest `405 Method Not Allowed`. Moim zdaniem, jeśli zasób HTTP nie jest w stanie obsłużyć żądania przy użyciu danej metody HTTP, powinien wysłać nagłówek `Allow`, aby przekazać klientowi listę dozwolonych metod HTTP. W tym celu możesz użyć dyrektywy `add_header`, ale pamiętaj o potencjalnych problemach.

Na koniec przykłady konfiguracji. Pierwsza rekomendowana (mimo wykorzystania `if`):

```nginx
# If we are in server context, it’s good to use construction like this:
add_header Allow "GET, HEAD, POST" always;

if ($request_method !~ ^(GET|HEAD|POST)$) {

  # You can also use 'add_header' inside 'if' context instead of outside 'if':
  # add_header Allow "GET, HEAD, POST" always;
  return 405;

}
```

Oraz druga, będąca alternatywą:

```nginx
# Note: allowing the GET method makes the HEAD method also allowed.
location /api {

  limit_except GET POST {

    allow 192.168.1.0/24;
    deny  all;  # always return 403 error code

    # or:
    # auth_basic "Restricted access";
    # auth_basic_user_file /etc/nginx/htpasswd;

    ...

  }

}
```
