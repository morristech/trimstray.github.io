---
layout: post
title: 'NGINX: Ujawnianie wersji i sygnatur serwera'
date: 2017-07-19 22:01:35
categories: [publications]
tags: [PL, http, nginx, best-practices]
comments: false
favorite: false
seo:
  date_modified: 2020-02-21 16:30:58 +0100
---

Ujawnienie wersji oraz sygnatur serwera NGINX może być niepożądane, szczególnie w środowiskach wrażliwych na ujawnianie informacji (tj. przetwarzających dane krytyczne). NGINX domyślnie wyświetla numer wersji na stronach błędów i w nagłówkach odpowiedzi HTTP.

Informacje te mogą być wykorzystane jako punkt wyjścia dla atakujących, którzy znają określone luki związane z określonymi wersjami i mogą pomóc w lepszym zrozumieniu używanych systemów, a także potencjalnie rozwinąć dalsze ataki ukierunkowane na określoną wersję usługi. Na przykład Shodan zapewnia powszechnie używaną bazę danych tych informacji. O wiele bardziej wydajne jest po prostu wypróbowanie luki na wszystkich losowych serwerach niż ich zapytanie.

Zlekceważenie tak ważnego czynnika związanego z bezpieczeństwem jest moim zdaniem elementarnym błędem. Oczywiście bezpieczeństwo poprzez zaciemnienie nie ma tak naprawdę żadnego wpływu na bezpieczeństwo serwera czy infrastruktury, jednak jest pewne, że ochroni przed najprostrzymi wektorami ataku.

Moim zdaniem, bezpieczeństwo poprzez zaciemnienie jest koniecznym krokiem i powinno być podstawową zasadą związaną z hardeningiem serwera. Całkowite pominięcie tego kroku to bardzo zły pomysł, ponieważ nawet najbezpieczniejsze serwery HTTP mogą zostać złamane, jeśli znany jest wektor ataku specyficzny dla wersji.

Jeżeli masz jakiekolwiek dylematy co do takiej konfiguracji, [RFC 2616 - Personal Information](https://tools.ietf.org/html/rfc2616#section-15.1) będzie tutaj bardzo pomocne w podjęciu decyzji:

  > _History shows that errors in this area often create serious security and/or privacy problems and generate highly adverse publicity for the implementor's company. [...] Like any generic data transfer protocol, HTTP cannot regulate the content of the data that is transferred, nor is there any a priori method of determining the sensitivity of any particular piece of information within the context of any given request. Therefore, applications SHOULD supply as much control over this information as possible to the provider of that information. Four header fields are worth special mention in this context: `Server`, `Via`, `Referer` and `From`._

W ramach ciekawostki, spójrz co na ten temat mówi dokumentacja serwera Apache:

  > _Setting ServerTokens to less than minimal is not recommended because it makes it more difficult to debug interoperational problems. Also note that disabling the Server: header does nothing at all to make your server more secure. The idea of "security through obscurity" is a myth and leads to a false sense of safety._

Polecam także:

- [Shhh... don’t let your response headers talk too loudly](https://www.troyhunt.com/shhh-dont-let-your-response-headers/)
- [Configuring Your Web Server to Not Disclose Its Identity](https://www.acunetix.com/blog/articles/configure-web-server-disclose-identity/)
- [Reduce or remove server headers](https://www.tunetheweb.com/security/http-security-headers/server-header/)
- [Fingerprint Web Server (OTG-INFO-002)](https://www.owasp.org/index.php/Fingerprint_Web_Server_(OTG-INFO-002)).

# Ujawnianie wersji serwera

Ukrywanie informacji o wersji nie powstrzyma ataku, ale sprawi, że będziesz mniejszym celem, jeśli atakujący szukają określonej wersji sprzętu lub oprogramowania. Według mnie, dane transmitowane przez serwer HTTP należy traktować jako dane osobowe (bynajmniej nie jest to stwierdzenie ani trochę na wyrost).

Bezpieczeństwo przez zaciemnienie nie oznacza, że ​​jesteś bezpieczny, ale czasami spowalnia atakującego, i to jest dokładnie to, co jest potrzebne w przypadku luk w zerowym dniu.

Aby wyłączyć rozgłaszanie wersji na stronach błedów oraz w polu nagłówka `Server`:

```nginx
server_tokens off;
```

# Ujawnianie sygnatur serwera

Nagłówek `Server` zawiera informacje o oprogramowaniu używanym przez serwer obsługujący żądania. Wartość tego nagłówka jest używana przez takie serwisy jak Alexa i Netcraft do zbierania statystyk o serwerach HTTP. Jednym z najłatwiejszych kroków zabezpieczenia serwera HTTP jest wyłączenie wyświetlania informacji o używanym oprogramowaniu i technologii za pośrednictwem tego nagłówka serwera.

Istnieje kilka powodów, dla których rozgłaszanie wersji jest bardzo nieporządane. Osoba atakująca zbiera wszystkie dostępne informacje o aplikacji i jej środowisku. Informacje o zastosowanych technologiach i wersjach oprogramowania są niezwykle cennymi informacjami.

Moim zdaniem nie ma żadnego racjonalnego powodu ani potrzeby pokazywania tak wielu informacji o twoim serwerze. Po wykryciu numeru wersji łatwo jest wyszukać określone luki w zabezpieczeniach. Co więcej, nie są to informacje kluczowe i niezbędne do poprawnego działania serwera lub aplikacji (w tym aplikacji zewnętrznych), więc zasadniczo jestem za ich usunięciem, jeśli można to osiągnąć przy minimalnym wysiłku.

Wyłączenie wersji serwera można wykonać na kilka sposobów. Najbardziej wskazanym sposobem jest usunięcie tego nagłówka za pomocą modułu [headers-more-nginx-module](https://github.com/openresty/headers-more-nginx-module):

```nginx
http {

  more_clear_headers 'Server';

  ...
```

Innym, także wykorzystującym ten moduł, jest ustawienie własnej wartości tego nagłówka:

```nginx
http {

  more_set_headers "Server: Unknown";

  ...
```

Do tego celu możesz wykorzystać moduł [lua-nginx-module](https://github.com/openresty/lua-nginx-module):

```nginx
http {

  header_filter_by_lua_block {
    ngx.header["Server"] = nil
  }

  ...
```
