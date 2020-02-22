---
layout: post
title: 'Protokół HTTP: Połączenie kontra żądanie'
date: 2019-09-17 21:15:31
categories: [publications]
tags: [PL, http, connections, requests]
comments: false
favorite: false
seo:
  date_modified: 2020-02-22 09:50:08 +0100
---

Jak zdefiniować połączenie a jak żądanie? Czym jest pierwsze a czym drugie? Może tym samym?

Zasadniczo, nawiązujemy połączenia w celu wysyłania za jego pomocą żądań. Można mieć wiele żądań na połączenie.

Połączenie jest niezawodnym potokiem opartym na protokole TCP między dwoma punktami końcowymi. Każde połączenie wymaga śledzenia zarówno adresów/portów punktów końcowych, numerów sekwencyjnych, jak i pakietów, których nie potwierdzono lub których może brakować. Żądanie zaś, to "prośba" o dany zasób za pomocą określonej metody, wykorzystujące połączenie podczas komunikacji z serwerem.

Spójrz na poniższy zrzut (dodatkowo jest to porównanie HTTP/1.1 oraz H2):

<img src="/assets/img/posts/http_conn_requests.png" align="center" title="http_conn_requests preview">

Większość współczesnych przeglądarek otwiera jednocześnie kilka połączeń i jednocześnie pobiera różne pliki (obrazy, css, js), aby przyspieszyć ładowanie strony. Stąd jak widać, każde połączenie może obsługiwać wiele żądań.

- Połączenia (ang. _connection_) HTTP - klient i serwer przedstawiają się; nawiązanie połączenia z serwerem wymaga uzgadniania protokołu TCP i zasadniczo polega na utworzeniu połączenia z gniazdem serwera

- Żądanie (ang. _request_) HTTP - klient pyta serwer o dany zasób; aby złożyć żądanie HTTP, należy już ustanowić połączenie z serwerem. Jeśli nawiązano takie połączenie, można złożyć wiele żądań przy użyciu tego samego połączenia (HTTP/1.0 domyślnie jedno żądanie na połączenie, HTTP/1.1 domyślnie od 4-6 połączeń z wykorzystaniem mechanizmu podtrzymywania połączeń)

Zerknij na to proste porównanie:

- 25 połączeń, jedno po drugim i pobieranie 1 pliku przez każde połączenie (najwolniej)
- 1 połączenia i pobieranie przez niego 25 plików (wolne)
- 5 połączeń i pobieranie 5 plików przez każde połączenie (szybkie)
- 25 połączeń i pobieranie 1 pliku przez każde połączenie (marnotrawstwo zasobów)

Jeśli więc nadmiernie ograniczysz liczbę połączeń lub liczbę żądań, spowolnisz szybkość ładowania serwisu.
