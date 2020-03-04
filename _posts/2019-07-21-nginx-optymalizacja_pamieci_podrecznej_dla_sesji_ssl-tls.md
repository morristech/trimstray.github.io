---
layout: post
title: 'NGINX: Optymalizacja pamięci podręcznej dla sesji SSL/TLS'
date: 2019-07-21 23:04:51
categories: [publications]
tags: [PL, http, nginx, best-practices]
comments: false
favorite: false
seo:
  date_modified: 2020-02-23 09:14:19 +0100
---

Domyślnie wbudowana pamięć podręczna sesji SSL/TLS nie jest optymalna, ponieważ może być używana tylko przez jeden proces roboczy, co może powodować fragmentację pamięci. O wiele lepiej jest używać współdzielonej pamięci podręcznej, a także zoptymalizować dodatkowe parametry.

## ssl_session_cache

Pierwszy z parametrów zwiększa ogólną wydajność połączeń (zwłaszcza połączeń typu `Keep-Alive`). Wartość 10MB jest dobrym punktem wyjścia (1 MB współdzielonej pamięci podręcznej może pomieścić około 4000 sesji). Dzięki parametrowi `shared` pamięć dla połączeń SSL jest współdzielona przez wszystkie procesy robocze (co więcej pamięć podręczna o tej samej nazwie może być używana na kilku serwerach wirtualnych).

Oczywiście nie ma róży bez kolców. Jednym z powodów, dla których nie należy używać bardzo dużej pamięci podręcznej, jest to, że większość implementacji nie usuwa z niej żadnych rekordów. Nawet wygasłe sesje mogą nadal znajdować się w pamięci podręcznej i można je odzyskać!

Włączenie buforowania sesji pomaga zmniejszyć obciążenie procesora. Zwiększa także wydajność z punktu widzenia klientów, ponieważ eliminuje potrzebę przeprowadzania nowego (i czasochłonnego) uzgadniania SSL/TLS przy każdym żądaniu.

Przykład:

```nginx
ssl_session_cache shared:NGX_SSL_CACHE:10m;
```

# ssl_session_timeout

Zgodnie z [RFC 5077 - Ticket Lifetime](https://tools.ietf.org/html/rfc5077#section-5.6), sesje nie powinny być utrzymywane dłużej niż 24 godziny (jest to maksymalny czas dla sesji SSL). Jednak jakiś czas temu znalazłem rekomendację, aby dyrektywa ta miała mniejszą wartość (ok. 15 minut). Ma to zapobiegać nadużyciom przez reklamodawców (trackerów) takich jak Google i Facebook. Nigdy nie stosowałem tak niskich wartości, jednak myślę, że w jakiś sposób może to mieć sens.

W tym miejscu chciałbym zacytować wypowiedź twórcy serwisu [Hardenize](https://www.hardenize.com/), a także autora świetnej książki [Bulletproof SSL and TLS: Understanding and deploying SSL/TLS and PKI to secure servers and web applications.](https://www.feistyduck.com/books/bulletproof-ssl-and-tls/):

  > _These days I'd probably reduce the maximum session duration to 4 hours, down from 24 hours currently in my book. But that's largely based on a gut feeling that 4 hours is enough for you to reap the performance benefits, and using a shorter lifetime is always better._

Myślę, że wartość 4h jest racjonalną i jedną z optymalnych wartości.

Przykład:

```nginx
ssl_session_timeout 4h;
```

# ssl_session_tickets

Kolejna rzecz do poprawy to klucze sesji lub tzw. bilety sesji. Zawierają one pełny stan sesji (w tym klucz wynegocjowany między klientem a serwerem oraz wykorzystywany zestaw szyfrów). Wszystkie informacje wymagane do kontynuowania sesji są tam zawarte, więc serwer może wznowić sesję wykorzystując wcześniejsze parametry. Cała dodatkowa obsługa odbywa się po stronie klienta.

Niestety większość serwerów nie usuwa kluczy sesji ani biletów, zwiększając w ten sposób ryzyko wycieku danych z poprzednich (i przyszłych) połączeń. Co więcej, takie zachowanie "niszczy" tajemnicę przekazywania (ang. `Forward Secrecy`). Moim zdaniem jedynym sposobem na prawdziwe usunięcie danych sesyjnych jest zastąpienie ich nową sesją.

  > [Vincent Bernat](https://vincent.bernat.ch/en) napisał świetne [narzędzie](https://github.com/vincentbernat/rfc5077/blob/master/rfc5077-client.c) do testowania mechanizmu wznawiania sesji z wykorzystaniem ticket'ów.

Przykład:

```nginx
ssl_session_tickets off;
```

# ssl_buffer_size

Parametr ten odpowiada za ustawienie bufora przesyłanych danych za pomocą protokołu TLS. Jest to jeden z tych parametrów, dla którego spotkać można różne wartości. Spowodowane jest to pewną niejednoznacznością. Aby dostosować wartość tego parametru, należy pamiętać m.in. o rezerwacji miejsca na różne opcje TCP (znaczniki czasu czy SACK), które mogą zajmowac do 40 bajtów. Uwzględnić należy także rekordy TLS (średnio od 20 do 60 bajtów) w zależności od wynegocjowanego szyfru między klientem a serwerem.

Tym samym można przyjąć: 1500 bajtów (MTU) - 40 bajtów (IP) - 20 bajtów (TCP) - 60-100 bajtów (narzut TLS) ~= 1300 bajtów.

  > Jeśli sprawdzisz rekordy emitowane przez serwery Google, zobaczysz, że zawierają one ok. 1300 bajtów danych z powodu powyższej matematyki.

Spakowanie każdego rekordu TLS do dedykowanego pakietu powoduje dodatkowe obciążenie związane z tworzeniem ramek i prawdopodobnie zajdzie potrzeba ustawienia większych rozmiarów rekordów, jeśli przesyłasz strumieniowo większe (i mniej wrażliwe na opóźnienia) dane.

Myślę, że optymalną wartością jest wartość 1400 bajtów (lub bardzo zbliżona). 1400 bajtów (tak naprawdę powinno być nawet nieco niższe zgodnie z równaniem) jest zalecanym ustawieniem dla ruchu interaktywnego, w którym chcesz uniknąć niepotrzebnych opóźnień spowodowanych utratą/fluktuacją fragmentów rekordu TLS.

Spójrzmy także na poniższą rekomendację (autorzy: Leif Hedstrom, Thomas Jackson, Brian Geffon):

- mniejszy rozmiar rekordu TLS; MTU/MSS (1500) - TCP (20 bytes) - IP (40 bytes): 1500 - 40 - 20 = 1440 bytes
- większy rozmiar rekordu TLS; maksymalny rozmiar wynosi 16,383 (2^14 - 1)

Przykład:

```nginx
ssl_buffer_size 1400;
```

# Podsumowanie

Sam widzisz, że nie ma jednoznacznych odpowiedzi dotyczących parametrów sesji SSL/TLS, które można ustawić z poziomu serwera NGINX. Musisz więc zrównoważyć wydajność (nie chcesz, aby użytkownicy używali pełnego uzgadniania przy każdym połączeniu) i bezpieczeństwo (nie chcesz zbytnio narażać komunikacji TLS na szwank). Co więcej, nie ma jednego standardu i różne projekty dyktują różne ustawienia.
