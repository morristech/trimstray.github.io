---
layout: post
title: 'NGINX: Poprawne przekazywanie nagłówka Host'
date: 2019-03-27 00:36:49
categories: [publications]
tags: [PL, http, nginx, best-practices, headers]
comments: false
favorite: false
seo:
  date_modified: 2020-02-23 09:14:19 +0100
---

Nagłówek `Host` jest jednym z najważniejszych nagłówków w komunikacji HTTP. Informuje on serwer, którego wirtualnego hosta ma użyć, pod jaki adres chcemy wysłać zapytanie oraz określa, która witryna internetowa lub aplikacja internetowa powinna przetwarzać przychodzące żądanie HTTP.

Nagłówek ten, wprowadzony w HTTP/1.1, to trzecia z najważniejszych informacji której można użyć, oprócz adresu IP i numeru portu, w celu jednoznacznego zidentyfikowania serwera aplikacji lub domeny internetowej. Możesz sobie wyobrazić, że jest on pewnego rodzaju mechanizmem routingu na poziomie aplikacji — na jego podstawie serwery aplikacyjne decydują o dalszym sposobie przetwarzania żądania a także umożliwiają obsługę wielu serwisów na jednym adresie IP.

W tym wpisie omówię za pomocą jakich zmiennych możemy zarządzać tym nagłówkiem oraz w jaki sposób przekazać poprawną wartość do warstwy backendu.

# Nagłówek Host a aplikacja

Bardzo często, po otrzymaniu żądania, aplikacja wykorzystuje przesłany nagłówek `Host` w celu określenia sposobu obsługi żądania. Moim zdaniem poleganie na wartościach ustawionych w tym nagłówku jest złym pomysłem, ponieważ może umożliwić przekierowanie klienta do spreparowanych zasobów (poprzez wstrzyknięcie sfałszowanej wartości nagłówka do pamięci podręcznej), do miejsc w aplikacji, które nie powinny być dostępne z zewnątrz lub całkowicie do innego (niezamierzonego) serwera.

Jeśli serwer aplikacji nie jest właściwie skonfigurowany, napastnik będzie mógł wykorzystać technikę polegającą na sfałszowaniu wartości tego nagłówka, w celu uzyskania dostępu np. do funkcji administracyjnych serwera aplikacji lub zmylenia serwerów odpowiedzialnych za load-balancing, którym zdarza się podejmować decyzję o kierowaniu żądań na podstawie wartości tego nagłówka.

Jednak jeśli obsługa nagłówka `Host` jest wymagana po stronie aplikacji, jako administratorzy powinniśmy zagwarantować jego poprawną wartość (tyle na ile jest to możliwe) aby upewnić się, że zachowanie hosta wirtualnego na dalszym serwerze działa tak, jak powinna.

Zgodnie z [RFC 7230 - Host](https://tools.ietf.org/html/rfc7230#section-5.4), gdy serwer proxy (który jest szczególnie wrażliwy na fałszowanie tego nagłówka) odbierze żądanie w formie bezwzględnej ([absolute-form](https://tools.ietf.org/html/rfc7230#section-5.3.2), przykład: `GET https://example.com/index.html HTTP/1.1` zamiast „standardowej” postaci tj. `GET /index.html HTTP/1.1 Host: example.com`) docelowego żądania, musi zignorować otrzymane pole nagłówka `Host` (jeśli istnieje) i zamiast tego zastąpić je informacją o hoście będącym celem żądania.

Serwer proxy, który przekazuje takie żądanie musi wygenerować nową wartość nagłówka na podstawie otrzymanego celu żądania, a nie przekazywać odebraną wartość pola `Host`. W takiej sytuacja serwer musi odpowiedzieć kodem 400 (Bad Request) na każdy komunikat żądania HTTP, który nie ma pola nagłówka `Host`, i na każdy komunikat żądania zawierający więcej niż jedno pole tego nagłówka lub zawierające niepoprawną wartością (według mnie, także adres IP).

Oczywiście najważniejszą linią obrony jest odpowiednia implementacja mechanizmów weryfikujących po stronie aplikacji, np. wykorzystanie listy dozwolonych wartości nagłówka `Host`. Twoja aplikacja powinna być w pełni zgodna z [RFC 7230](https://tools.ietf.org/html/rfc7230), aby uniknąć problemów spowodowanych niespójną interpretacją hosta w celu powiązania go z daną transakcją HTTP. Zgodnie z RFC 7230 poprawnym rozwiązaniem jest traktowanie wielu nagłówków `Host` i białych znaków wokół nazw pól jako błędów.

# Nagłówek Host a NGINX

NGINX udostępnia zmienne, które mogą przechowywać nagłówek `Host` dostarczony w żądaniu. Jedną ze zmiennych jest zmienna `$host`, która zapisuje wartość tego nagłówka z małych liter i z pominięciem numeru portu (jeśli był obecny).

Wyjątkiem jest, gdy `HTTP_HOST` jest nieobecny lub jest pustą wartością. W takim przypadku `$host` jest równy wartości dyrektywy `server_name`, czyli serwera, który przetworzył żądanie.

Jednak spójrz na to wyjaśnienie:

  > _An unchanged `Host` request header field can be passed with `$http_host`. However, if this field is not present in a client request header then nothing will be passed. In such a case it is better to use the `$host` variable - its value equals the server name in the `Host` request header field or the primary server name if this field is not present._

Wynika z tego, że jeśli ustawimy nagłówek `Host` w żądaniu na wartość `Host: MASTER:8080`, zmienna `$host` będzie przechowywać wartość `master`, podczas gdy `$http_host` będzie równy `MASTER:8080` (w taki sposób odzwierciedla ona cały nagłówek).

Zgodnie z tym `$host` to po prostu `$http_host` z pewnymi modyfikacjami (zostaje usunięty numeru portu oraz wykonana jest konwersja na małe litery) i wartością domyślną (`server_name`).

  > Zmienna `$host` to nazwa hosta z wiersza żądania lub nagłówka HTTP. Zmienna `$server_name` to nazwa bloku serwera, w którym przetwarzane jest żądanie.

Różnicę wyjaśniono w dokumentacji NGINX:

- `$host` zawiera w następującej kolejności: nazwa hosta z wiersza żądania, nazwa hosta z pola nagłówka żądania lub nazwa serwera pasująca do żądania
- `$http_host` zawiera zawartość pola nagłówka `Host`, jeśli była obecna w żądaniu (zawsze równa się nagłówkowi żądania `HTTP_HOST`)
- `$server_name` zawiera nazwę serwera wirtualnego hosta, który przetworzył żądanie, tak jak zostało zdefiniowane w konfiguracji NGINX. Jeśli serwer zawiera wiele nazw serwerów, tylko pierwszy z nich będzie obecny w tej zmiennej

`$http_host` ponadto jest lepszy niż konstrukcja `$host:$server_port`, ponieważ używa portu obecnego w adresie URL, w przeciwieństwie do `$server_port`, który używa portu, na którym nasłuchuje NGINX.

W związku z tym aby poprawnie przekazać wartość nagłówka `Host` do aplikacji należy:

```nginx
proxy_set_header Host $host;
```

Takie ustawienie pozwala używać przeparsowanego nazwy host z wierdza żądania lub nagłówka `Host` oraz gwarantuje, że nagłówek hosta jest ustawiony tak, aby serwer nadrzędny mógł odwzorować żądanie na serwer wirtualny lub w inny sposób wykorzystać część hosta adresu URL wprowadzonego przez użytkownika.

Z drugiej strony, użycie zmiennej `$host` ma swoją własną podatność; musisz poradzić sobie z sytuacją, gdy pole nagłówka `Host` jest nieobecne, definiując domyślne bloki serwera, aby wychwycić takie żądania. Kluczową kwestią jest jednak to, że dyrektywa `proxy_set_header Host $host` w ogóle nie zmieni tego zachowania, ponieważ wartość zawarta w zmiennej `$host` będzie równa wartości `$http_host`, gdy obecne będzie pole nagłówka w żądaniu HTTP.

  > Jeśli wymagane jest użycie origynalnej nazwy wirtualnego hosta z pierwotnego żądania, możesz użyć zmiennej `$http_host` zamiast `$host`.

Aby temu zapobiec, należy wykorzystać wirtualne hosty typu catch-all. Są to bloki, do których odwołuje się serwer NGINX, jeśli w żądaniu klienta pojawia się nierozpoznany/niezdefiniowany nagłówek `Host`. Dobrym pomysłem jest także podanie dokładnej (nie wieloznacznej) wartości w dyrektywie `server_name`.

# Alterntywy dla nagłówka Host

Spójrz co mówi na ten temat [RFC 7540 - Request Pseudo-Header Fields](https://tools.ietf.org/html/rfc7540#section-8.1.2.3):

  > _To ensure that the HTTP/1.1 request line can be reproduced accurately, this pseudo-header field MUST be omitted when translating from an HTTP/1.1 request that has a request target in origin or asterisk form. Clients that generate HTTP/2 requests directly SHOULD use the ":authority" pseudo-header field instead of the Host header field. An intermediary that converts an HTTP/2 request to HTTP/1.1 MUST create a Host header field if one is not present in a request by copying the value of the ":authority" pseudo-header field._

Oczywiście odnosi się to do protokołu HTTP/2, który dostarcza pseudonagłówek `:authority` będący alternatywą dla nagłówka `Host` w HTTP/1.1.
