---
layout: post
title: 'NGINX: Nieodpowiednie użycie dyrektywy deny'
date: 2016-05-21 23:01:29
categories: [publications]
tags: [PL, http, nginx, best-practices]
comments: false
favorite: false
seo:
  date_modified: 2020-02-21 16:30:58 +0100
---

Dyrektywy `allow` oraz `deny` dostarczane są z modułem `ngx_http_access_module` i umożliwiają zezwolenie na dostęp lub jego ograniczenie do wybranych adresów klientów. Obie nadają się do budowania list kontroli dostępu. Moim zdaniem najlepszym sposobem budowania list ACL jest zacząć od odmowy, a następnie przyznawać dostęp adresom IP tylko do wybranych lokalizacji:

```nginx
location /login {

  allow 192.168.252.10;
  allow 192.168.252.11;
  allow 192.168.252.12;
  deny all;

}
```

  > Pamiętaj, że dyrektywa `deny` zawsze zwróci kod błędu _403 Forbidden_, odnoszący się do klienta uzyskującego dostęp, który nie jest upoważniony do wykonania danego żądania.

Czy stosowanie powyższych dyrektyw może rodzić jakieś negatywne konsekwencje? Zdecydowanie tak: obie dyrektywy mogą działać wbrew oczekiwaniom, zwłaszcza, łącząc je z mechanizmem przepisywania dostarczanym przez serwer NGINX. Spójrz na poniższy przykład:

```nginx
server {

  server_name example.com;

  deny all;

  location = /test {
    return 200 "it's all okay";
    more_set_headers 'Content-Type: text/plain';
  }

}
```

Następnie, wykonując poniższy request:

```bash
curl -i https://example.com/test
HTTP/2 200
date: Wed, 11 Nov 2018 10:02:45 GMT
content-length: 13
server: Unknown
content-type: text/plain

it's all okay
```

Widzisz, że dostaliśmy odpowiedź o treści "_it's all okay_" z kodem 200. Dlaczego, skoro jawnie zablokowaliśmy dostęp do całego zasobu `/test` za pomocą dyrektywy `deny` i która jest jakby nad kontekstem `location = /test` (tj. jej zakres rozchodzi się na cały blok `server`)?

Jest to logiczne i prawidłowe zachowanie i ma związek z całym mechanizmem przetwarzania żądań. Każdy request, jak już dotrze do NGINX'a, zostaje przetwarzany w tzw. fazach. Tych faz jest dokładnie 11:

- `NGX_HTTP_POST_READ_PHASE` - pierwsza faza, w której czytany jest nagłówek żądania
  - przykładowe moduły: `ngx_http_realip_module`

- `NGX_HTTP_SERVER_REWRITE_PHASE` - implementacja dyrektyw przepisywania zdefiniowanych w bloku serwera; w tej fazie m.in. zmieniany jest identyfikator URI żądania za pomocą wyrażeń regularnych (PCRE)
  - przykładowe moduły: `ngx_http_rewrite_module`

- `NGX_HTTP_FIND_CONFIG_PHASE` - zamieniana jest lokalizacja zgodnie z URI (wyszukiwanie lokalizacji)

- `NGX_HTTP_REWRITE_PHASE` - modyfikacja URI na poziomie lokalizacji
  - przykładowe moduły: `ngx_http_rewrite_module`

- `NGX_HTTP_POST_REWRITE_PHASE` - przetwarzanie końcowe URI (żądanie zostaje przekierowane do nowej lokalizacji)
  - przykładowe moduły: `ngx_http_rewrite_module`

- `NGX_HTTP_PREACCESS_PHASE` - wstępne przetwarzanie uwierzytelnienia; sprawdzane są m.in. limity żądań oraz limity połączeń (ograniczenie dostępu)
  - przykładowe moduły: `ngx_http_limit_req_module`, `ngx_http_limit_conn_module`, `ngx_http_realip_module`

- `NGX_HTTP_ACCESS_PHASE` - weryfikacja klienta (proces uwierzytelnienia, ograniczenie dostępu)
  - przykładowe moduły: `ngx_http_access_module`, `ngx_http_auth_basic_module`

- `NGX_HTTP_POST_ACCESS_PHASE` - faza przetwarzania końcowego związana z ograniczaniem dostępu
  - przykładowe moduły: `ngx_http_access_module`, `ngx_http_auth_basic_module`

- `NGX_HTTP_PRECONTENT_PHASE` - generowanie treści (odpowiedzi)
  - przykładowe moduły: `ngx_http_try_files_module`

- `NGX_HTTP_CONTENT_PHASE` - przetwarzanie treści (odpowiedzi)
  - przykładowe moduły: `ngx_http_index_module`, `ngx_http_autoindex_module`, `ngx_http_gzip_module`

- `NGX_HTTP_LOG_PHASE` - mechanizm logowania, tj. zapisywanie informacji do pliku z logami
  - przykładowe moduły: `ngx_http_log_module`

Przygotowałem również prostą grafikę, która pomoże ci zrozumieć, jakie moduły oraz dyrektywy są używane na każdym etapie:

<img src="/assets/img/posts/nginx_phases.png" align="center" title="nginx_phases preview">

Dodatkowo na każdej fazie można zarejestrować dowolną liczbę handlerów oraz każda z nich ma listę powiązanych z nią procedur obsługi.

  > Polecam zapoznać się ze świetnym wyjaśnieniem dotyczącym [faz przetwarzania żądań](http://scm.zoomquiet.top/data/20120312173425/index.html). Dodatkowo, w tym [oficjalnym przewodniku](http://nginx.org/en/docs/dev/development_guide.html) także dość dokładnie opisano cały proces przejścia żądania przez każdą z faz.

Wróćmy teraz do naszego problemu i "dziwnego" zachowania dyrektywy `deny` w połączeniu z wykorzystaniem dyrektywy `return` - co w konsekwencji prowadzi do natychmiastowego przesłania odpowiedzi do klienta a nie zablokowania dostępu do danego zasobu.

Jak już wspomniałem, wynika to z faktu, że przetwarzanie żądania odbywa się w fazach, a faza przepisywania (do której należy dyrektywa `return`) wykonywana jest przed fazą dostępu (w której działa dyrektywa `deny`).

Niestety NGINX nie zgłasza nic niepokojącego (bo i po co) podczas przeładowania, więc odpowiedzialność poprawnego budowania reguł filtrujących wraz z pozostałymi mechanizmami spada na administratora.

Jednym z rozwiązań jest użycie instrukcji `if` w połączeniu z modułami `geo` lub `map`. Na przykład:

```nginx
  server_name example.com;

  location / {

    if ($whitelist.acl) {

      set $pass 1;

    }

    if ($pass = 1) {

      return 200 "it's all okay";
      # lub:
      proxy_pass http://bk_web01;

      more_set_headers 'Content-Type: text/plain';

    }

    if ($pass != 1) {

      return 403;

    }

  }
  ```

  > Nie zaleca się używania instrukcji `if`, chociaż moim zdaniem, użycie takiej konstrukcji może być nieco bardziej elastyczne oraz bezpieczniejsze dzięki wykorzystaniu ww. modułów.

Planując budowanie list kontroli dostępu, rozważ kilka opcji, z których możesz skorzystać. NGINX dostarcza moduły `ngx_http_access_module`, `ngx_http_geo_module`, `ngx_http_map_module` lub `ngx_http_auth_basic_module`, które pozwalają na nadawanie dostępów i zabezpieczanie miejsc w aplikacji.

Zawsze powinieneś przetestować swoje reguły przed ich ostatecznym wdrożeniem:

- sprawdź, na jakich fazach działają wykorzystywane dyrektywy
- wykonaj kilka testowych request'ów w celu potwierdzenia poprawnego działania mechanizmów zezwalających lub blokujących dostęp do chronionych zasobów Twojej aplikacji
- wykonaj kilka testowych request'ów w celu sprawdzenia i weryfikacji kodów odpowiedzi HTTP dla chronionych zasobów Twojej aplikacji
- należy zminimalizować dostęp każdego użytkownika do krytycznych zasobów tylko do wymaganych adresów IP
- przed dodaniem adresu IP klienta zweryfikuj czy jest on faktycznym właścicielem adresu w bazie danych whois
- regularnie poddawaj weryfikacji swoje reguły kontroli dostępu, aby upewnić się, że są aktualne i nie mają słabych punktów
