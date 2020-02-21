---
layout: post
title: 'NGINX: Maksymalna ilość równoczesnych połączeń'
date: 2020-02-17 12:04:25
categories: [publications]
tags: [PL, http, nginx, best-practices]
comments: false
favorite: false
seo:
  date_modified: 2020-02-21 16:30:58 +0100
---

Zastanawiałeś się kiedyś ile maksymalnie równoczesnych połączeń jest w stanie obsłużyć serwer NGINX? Oraz jaki wpływ na ich ilość mają dyrektywy `worker_processes`, `worker_connections` i `worker_rlimit_nofile`?

Spójrz na poniższe równanie:

```
worker_processes * worker_connections = max connections
```

Zgodnie z tym: jeśli uruchomisz 4 procesy robocze z ustawioną wartością 4096 połączeń na proces roboczy (worker), będziesz w stanie obsłużyć maksymalnie 16 384 połączeń. Oczywiście ustawienia te są ograniczone przez jądro (maksymalna liczba połączeń, maksymalna liczba otwartych plików lub maksymalna liczba procesów).

  > W tym miejscu polecam przeczytanie świetnego artykułu: [Understanding socket and port in TCP](https://medium.com/fantageek/understanding-socket-and-port-in-tcp-2213dc2e9b0c), w celu pełniejszego zrozumienia gniazd i portów TCP. Warto także zaznajomić się z dokumentem opisującym [maksymalną liczbę otwartych połączeń TCP](https://stackoverflow.com/questions/2332741/what-is-the-theoretical-maximum-number-of-open-tcp-connections-that-a-modern-lin), jakie może utworzyć system GNU/Linux.

Jednak czy powyższe równanie w sposób definitywny określa maksymalną liczbę połączeń? Tak, jednak jest pewna rzecz warta wyjaśnienia. W wielu artykułach dostępnych w Internecie widziałem, jak niektórzy administratorzy tłumaczą sumę wartości dyrektyw `worker_processes` oraz `worker_connections` bezpośrednio na maksymalną liczbę klientów, którzy mogą być obsługiwani jednocześnie.

Moim zdaniem jest to duży błąd, ponieważ niektórzy klienci (np. przeglądarki) otwierają szereg równoległych połączeń (na potwierdzenie moich słów, zerknij na [to krótkie wyjaśnienie](https://stackoverflow.com/questions/985431/max-parallel-http-connections-in-a-browser)). Klienci zwykle ustanawiają od 4 do 8 połączeń TCP, aby równolegle pobierać zasoby (aby pobierać różne komponenty strony internetowej, na przykład obrazy, skrypty itp.). Takie zachowanie zwiększa efektywną przepustowość i zmniejsza opóźnienia.

  > Jest to limit dla protokołu HTTP/1.1, który zgodnie z RFC wynosi 6-8 równoczesnych połączeń. Najlepszym rozwiązaniem w celu poprawy wydajności (bez aktualizacji sprzętu i użycia pamięci podręcznej między klientem a aplikacją, tj. CDN, Varnish) jest użycie protokołu w wersji HTTP/2 ([RFC 7540](https://tools.ietf.org/html/rfc7540) <sup>[IETF]</sup>) zamiast HTTP/1.1. Jak wiemy, protokół HTTP/2 multipleksuje wiele żądań w jednym połączeniu i nie ma on standardowego limitu ilości połączeń, jednak definiuje ich ilość w następujący sposób: "_It is recommended that this value (SETTINGS_MAX_CONCURRENT_STREAMS) be no smaller than 100_" (zgodnie z RFC 7540).

Ponadto należy wiedzieć, że dyrektywa `worker_connections` obejmuje **wszystkie połączenia na proces roboczy**, tj. połączenia do gniazd nasłuchiwania czy połączenia wewnętrznej komunikacji między procesami NGINX, połączenia z serwerami proxy i połączenia z serwerami warstwy backend'u), a nie tylko połączenia przychodzące od klientów.

  > Ciekawostka: Każde połączenie obsługiwane przez proces roboczy, który jest w stanie uśpienia, potrzebuje 256 bajtów pamięci.

Liczba połączeń jest szczególnie ograniczona przez maksymalną liczbę otwartych plików (`RLIMIT_NOFILE` w systemach GNU/Linux).

  > O deskryptorach plików oraz uchwytach plików możesz poczytaj [tutaj](https://pl.wikipedia.org/wiki/Deskryptor_pliku) oraz [tutaj](https://stackoverflow.com/questions/2423628/whats-the-difference-between-a-file-descriptor-and-file-pointer). W tym artykule będę stosował zamiennie oba terminy.

Powodem takiego zachowania jest to, że system operacyjny potrzebuje pamięci do zarządzania każdym otwartym deskryptorem pliku, a jak wiemy, pamięć jest zasobem, który można bardzo szybko wysycić. Oczywiście powyższe ograniczenie wpływa tylko na limity dla bieżącego procesu. Granice bieżącego procesu są również przekazywane procesom potomnym, ale każdy proces ma wartość niezależną od procesu głównego.

Aby zmienić limit maksymalnych deskryptorów plików (które mogą być otwarte przez pojedynczy proces roboczy), możesz również edytować dyrektywę `worker_rlimit_nofile`. Dzięki temu NGINX zapewnia bardzo potężne możliwości dynamicznej konfiguracji bez ponownego uruchamiania serwera.

  > Maksymalna ilość otwartych deskryptorów plików nie jest jedynym ograniczeniem liczby połączeń — pamiętaj także o parametrach sieci jądra (stosu TCP/IP) i maksymalnej liczbie procesów.

Jeżeli chodzi o jasne wskazanie maksymalnej ilości otwartych deskryptorów plików, oficjalna dokumentacja jest tutaj bardzo powściągliwa. Mówi ona jedynie, że `worker_rlimit_nofile` jest ograniczeniem maksymalnej liczby otwartych plików dla procesów roboczych. Uważam, że jest to związane z jednym procesem roboczym, a nie ze wszystkimi.

Jeśli ustawisz `RLIMIT_NOFILE` na 25 000, a `worker_rlimit_nofile` na 12 000, NGINX ustawia (tylko dla procesów roboczych) maksymalny limit otwartych plików jako wartość dyrektywy `worker_rlimit_nofile`. Jednak proces główny będzie miał ustawioną wartość określoną za pomocą `RLIMIT_NOFILE`. Domyślnie `worker_rlimit_nofile` nie jest ustawiona, więc NGINX ustawia wartość początkową maksymalnej liczby otwartych plików na podstawie limitów systemowych.

Przykład:

```
# Dla GNU/Linux (lub /usr/lib/systemd/system/nginx.service):
grep "LimitNOFILE" /lib/systemd/system/nginx.service
LimitNOFILE=5000

grep "worker_rlimit_nofile" /etc/nginx/nginx.conf
worker_rlimit_nofile 256;

   PID       SOFT HARD
 24430       5000 5000
 24431        256 256
 24432        256 256
 24433        256 256
 24434        256 256
```

Moim zdaniem poleganie na wartości `RLIMIT_NOFILE` (i alternatywach w innych systemach) jest bardziej zrozumiałe i przewidywalne, ponieważ jest to jakby limit graniczny. Szczerze mówiąc, tak naprawdę nie ma znaczenia, który sposób wybierzesz, jednak należy zawsze pamiętać o tym, jaki priorytet ma każde z rozwiązań i gdzie leżą ograniczenia każdego z nich.

  > Jeśli nie ustawisz dyrektywy `worker_rlimit_nofile`, ilość deskryptorów plików używanych przez NGINX będzie określona ustawieniami systemu operacyjnego.

Swoją drogą, prawdopodobieństwo wyczerpania się deskryptorów plików jest minimalne, jednak może być dużym problemem przy obsłudze naprawdę sporego ruchu. Zależy także od rodzaju i ilości procesów działających na serwerze.

Podsumowując, ile maksymalnie deskryptorów plików może otworzyć NGINX?

- jedna uchwyt pliku dla aktywnego połączenia klienta
- jeden uchwyt pliku dla połączeń proxy (który otwiera gniazdo obsługujące dane żądania do zdalnego lub lokalnego hosta/procesu)
- jeden uchwyt pliku dla czytania plików (np. plik statyczny)
- inne uchwyty plików dla połączeń wewnętrznych, bibliotek współdzielonych, plików dziennika i gniazd

  > Ważne: NGINX może używać do dwóch deskryptorów plików na pełne połączenie.

Spójrz na poniższe przykłady:

1) Jeden uchwyt do połączenia z klientem i jeden uchwyt do obsługi plików, w tym wypadku dla pliku statycznego serwowanego przez NGINX:

```
  # 1 połączenie, 2 uchwyty plików

                     +-----------------+
+----------+         |                 |
|          |    1    |                 |
|  CLIENT <---------------> NGINX      |
|          |         |        ^        |
+----------+         |        |        |
                     |      2 |        |
                     |        |        |
                     |        |        |
                     | +------v------+ |
                     | | STATIC FILE | |
                     | +-------------+ |
                     +-----------------+
```

2) Jeden uchwyt do obsługi połączenia z klientem i jeden uchwyt dla otwartego gniazda zdalnego lub lokalnego hosta/procesu:

```
# 2 połączenia, 2 uchwyty plików

                     +-----------------+
+----------+         |                 |         +-----------+
|          |    1    |                 |    2    |           |
|  CLIENT <---------------> NGINX <---------------> BACKEND  |
|          |         |                 |         |           |
+----------+         |                 |         +-----------+
                     +-----------------+
```

3) Dwa uchwyty dla dwóch jednoczesnych połączeń od tego samego klienta (1, 4), jeden uchwyt dla połączenia z innym klientem (3), dwa uchwyty dla plików statycznych (2, 5) i jeden uchwyt dla otwartego gniazda do zdalnego lub lokalnego hosta/procesu:

```
# 4 połączenia, 6 uchwytów plików

                  4
      +-----------------------+
      |              +--------|--------+
+-----v----+         |        |        |
|          |    1    |        v        |  6
|  CLIENT <-----+---------> NGINX <---------------+
|          |    |    |        ^        |    +-----v-----+
+----------+    |    |        |        |    |           |
              3 |    |      2 | 5      |    |  BACKEND  |
+----------+    |    |        |        |    |           |
|          |    |    |        |        |    +-----------+
|  CLIENT  <----+    | +------v------+ |
|          |         | | STATIC FILE | |
+----------+         | +-------------+ |
                     +-----------------+
```

W dwóch pierwszych przykładach: możemy przyjąć, że NGINX potrzebuje dwa deskryptorów plików do obsługi pełnego połączenia (i dla każdego używa dwoch połączeń (`worker_connections`)). W trzecim przykładzie NGINX nadal potrzebuje dwóch uchwytów dla każdego pełnego połączenia (także, jeśli klient korzysta z połączeń równoległych).

Zgodnie z powyższym, uważam, że poprawna wartość dyrektywy `worker_rlimit_nofile` powinna być większa niż wartość dyrektywy `worker_connections`.

Moim zdaniem bezpieczna wartość `worker_rlimit_nofile` (i limitów systemowych) to:

```
# Jeden uchwyt dla jednego połączenia:
worker_connections + (shared libs, log files, event pool, etc.) = worker_rlimit_nofile

# Dwa uchwyty dla jednego połączenia:
(worker_connections * 2) + (shared libs, log files, event pool, etc.) = worker_rlimit_nofile
```

Prawdopodobnie tyle plików może otworzyć każdy pracownik i maksymalna ilość deskryptorów plików, które jest w stanie otworzyć NGINX, powinna mieć wartość większą niż liczba połączeń na proces roboczy (zgodnie z powyższą formułą).

W większości artykułów i samouczków widzimy, że ten parametr ma wartość podobną do maksymalnej liczby (lub nawet więcej) wszystkich otwartych plików jakie może otworzyć serwer NGINX. Jeśli założymy, że dotyczy on każdego procesu roboczego, wartości te są całkowicie zawyżone.

Jednak po głębszej refleksji uważam, że są one w miarę racjonalne, ponieważ pozwalają jednemu workerowi na użycie wszystkich deskryptorów plików, tak, aby nie ograniczały się do innych procesów roboczych, jeśli zostaną np. zamknięte lub stanie się z nimi cokolwiek niedobrego. Pamiętaj jednak, że nadal jesteśmy ograniczeni przez liczbę połączeń przypadających na proces roboczy.

Tak więc, przechodząc dalej, maksymalna liczba otwartych plików przez NGINX powinna wynosić:

```
(worker_processes * worker_connections * 2) + (shared libs, log files, event pool, etc.)
                                            =
                                      max open files
```

Dzięki czemu, aby obsłużyć 16 384 połączeń (4096 połączeń na każdy worker), mając na uwadze inne deskryptory plików używane przez NGINX, rozsądna wartość maksymalnej liczby deskryptorów plików w tym przypadku może wynosić 35 000. Myślę, że taka wartość jest wystarczająca.

Pamiętaj także o następujących zasadach:

- `worker_rlimit_nofile` służy do dynamicznej zmiany maksymalnej ilości deskryptorów plików obsługiwanych przez procesy robocze NGINX, które są zazwyczaj definiowane przez miękki limit systemu (`ulimit -Sn`)
- `worker_rlimit_nofile` działa tylko na poziomie procesu, jest ograniczony do twardego limitu systemu (`ulimit -Hn`)
- jeśli masz włączony SELinux, będziesz musiał uruchomić `setsebool -P httpd_setrlimit 1`, aby NGINX miał uprawnienia do ustawienia swojego limitu

Na koniec przykład:

```
nginx: master process         = LimitNOFILE (35,000)
  \_ nginx: worker process    = LimitNOFILE (35,000), worker_rlimit_nofile (10,000)
  \_ nginx: worker process    = LimitNOFILE (35,000), worker_rlimit_nofile (10,000)
  \_ nginx: worker process    = LimitNOFILE (35,000), worker_rlimit_nofile (10,000)
  \_ nginx: worker process    = LimitNOFILE (35,000), worker_rlimit_nofile (10,000)

                              = master (35,000), all workers:
                                                 - 140,000 by LimitNOFILE
                                                 - 40,000 by worker_rlimit_nofile
```

- każdy z procesów NGINX (master + worker) ma możliwość utworzenia do 35 000 plików
- dla wszystkich worker'ów maksymalna liczba deskryptorów plików wynosi 140 000 (`LimitNOFILE` na worker)
- dla każdego procesu roboczego początkowa/bieżąca liczba deskryptorów plików wynosi 10 000 (`worker_rlimit_nofile`)

W celu dodatkowego zgłębienia wiedzy polecam [Optimizing Nginx for High Traffic Loads](https://blog.martinfjordvald.com/2011/04/optimizing-nginx-for-high-traffic-loads/).
