---
layout: post
title: 'NGINX: Usprawnienie zamykania procesów roboczych'
date: 2017-01-02 11:35:12
categories: [publications]
tags: [PL, http, nginx, best-practices]
comments: false
favorite: false
seo:
  date_modified: 2020-02-21 16:30:58 +0100
---

Mechanizm ten powinien się przydać, jeśli chcesz poprawić czas zamykania procesów roboczych serwera NGINX. Dyrektywa `worker_shutdown_timeout` określa limit czasu, który będzie używany podczas płynnego zamykania procesów roboczych. Po upływie tego czasu, NGINX spróbuje zamknąć wszystkie aktualnie otwarte połączenia, aby ułatwić zamknięcie worker'ów.

Maxim Dounin wyjaśnia to w ten oto sposób:

  > The `worker_shutdown_timeout` directive is not expected to delay shutdown if there are no active connections. It was introduced to limit possible time spent in shutdown, that is, to ensure fast enough shutdown even if there are active connections.

Zobacz co się dzieje, gdy proces roboczy wchodzi w stan „wychodzenia”. Wykonywanych jest wtedy kilka czynności:

- proces jest oznaczany jako "do zamknięcia"
- ustawiany jest licznik czasu wyłączania, jeśli zdefiniowano czas za pomocą `worker_shutdown_timeout`
- zamykane jest gniazdo nasłuchiwania (`listen`)
- zamykane są bezczynne połączenia

Następnie, jeśli został ustawiony licznik czasu wyłączania, po upływie tego czasu wszystkie połączenia zostaną zamknięte.

Domyślnie NGINX musi czekać i przetwarzać dodatkowe dane od klienta przed całkowitym zamknięciem połączenia, ale tylko jeśli klient może wysyłać więcej danych.

Czasami możesz zobaczyć ostrzeżenie: `nginx: worker process is shutting down` w pliku dziennika. Problem pojawia się podczas ponownego ładowania konfiguracji gdzie NGINX zwykle całkiem efektywnie zamyka istniejące procesy robocze. Jednak czasami zamknięcie tych procesów może zająć bardzo dużo czasu, a każde przeładowanie konfiguracji może spowodować ciągłe działanie "starych" worker'ów, które mogą trwale pochłaniać dostępną pamięć systemu. W takim przypadku rozwiązaniem może być jawne zdefiniowanie czasu ich zamknięcia.

W celu ustawienia tego parametru:

```nginx
worker_shutdown_timeout 60s;
```

Moim zdaniem, wartość tej dyrektywy powinna być dostosowana do limitów czasu połączenia oraz czasu przetwarzania żądania przez serwer. 60 sekund to wartość z naprawdę bardzo solidnym zapasem (zdecydowanie żaden request nie powinien trwać dłużej).

Z mojego doświadczenia wynika, że ​​jeśli masz wiele workerów w stanie zamknięcia, być może powinieneś w pierwszej kolejności spojrzeć na dodatkowe moduły, które mogą powodować problemy z powolnym zamykaniem procesów roboczych.
