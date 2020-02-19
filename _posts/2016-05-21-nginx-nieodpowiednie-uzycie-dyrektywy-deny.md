---
layout: post
title: 'NGINX: Nieodpowiednie użycie dyrektywy deny'
date: 2016-05-21 23:01:29
categories: [publications]
tags: [PL, http, nginx, best-practices]
comments: false
favorite: false
seo:
  date_modified: 2020-02-19 14:42:54 +0100
---

Dyrektywy `allow` oraz `deny` dostarczane są z modułem `ngx_http_access_module` i umożliwiają zezwolenie na dostęp lub jego ograniczenie do wybranych adresów klientów. Obie nadają się do budowania list kontroli dostępu. Moim zdaniem najlepszym sposobem budowania za ich pomocą list ACL jest zacząć od odmowy, a następnie przyznawać dostęp tylko do tych lokalizacji, które chcesz.

  > Pamiętaj, że dyrektywa `deny` zawsze zwróci kod błędu 403.

Czy stosowanie obu dyrektyw może rodzić jakieś negatywne konsekwencje? Zdecydowanie tak: obie dyrektywy mogą działać nieoczekiwanie! Spójrz na następujący przykład:

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

Widzisz, że dostaliśmy odpowiedź o treści "it's all okay" z kodem 200. Dlaczego, skoro jawnie zablokowaliśmy dostęp do całego `/test` za pomocą dyrektywy `deny`?

Jest to logiczne i prawidłowe zachowanie i ma związek z całym mechanizmem przetwarzania żądań przez serwer NGINX. Każdy request, jak już dotrze do NGINX'a, zostaje przetwarzany w tzw. fazach. Jest ich dokładnie 11:

- `NGX_HTTP_POST_READ_PHASE` - pierwsza faza, w której czytany jest nagłówek żądania
  - przykładowe moduły: `ngx_http_realip_module`

- `NGX_HTTP_SERVER_REWRITE_PHASE` - implementacja dyrektyw przepisywania zdefiniowanych w bloku serwera; w tej fazie m.in. zmieniany jest identyfikator URI żądania za pomocą wyrażeń regularnych PCRE
  - przykładowe moduły: `ngx_http_rewrite_module`

- `NGX_HTTP_FIND_CONFIG_PHASE` - zamieniana jest lokalizacja zgodnie z URI (wyszukiwanie lokalizacji)

- `NGX_HTTP_REWRITE_PHASE` - modyfikacja URI na poziomie lokalizacji
  - przykładowe moduły: `ngx_http_rewrite_module`

- `NGX_HTTP_POST_REWRITE_PHASE` - przetwarzanie końcowe URI (żądanie zostaje przekierowane do nowej lokalizacji)
  - przykładowe moduły: `ngx_http_rewrite_module`

- `NGX_HTTP_PREACCESS_PHASE` - wstępne przetwarzanie uwierzytelnienia; sprawdzane są m.in. limity żądań oraz limit połączeń (ograniczenie dostępu)
  - przykładowe moduły: `ngx_http_limit_req_module`, `ngx_http_limit_conn_module`, `ngx_http_realip_module`

- `NGX_HTTP_ACCESS_PHASE` - weryfikacja klienta (proces uwierzytelnienia, ograniczenie dostępu)
  - przykładowe moduły: `ngx_http_access_module`, `ngx_http_auth_basic_module`

- `NGX_HTTP_POST_ACCESS_PHASE` - faza przetwarzania końcowego związanego z ograniczaniem dostępu
  - przykładowe moduły: `ngx_http_access_module`, `ngx_http_auth_basic_module`

- `NGX_HTTP_PRECONTENT_PHASE` - generowanie treści (odpowiedzi)
  - przykładowe moduły: `ngx_http_try_files_module`

- `NGX_HTTP_CONTENT_PHASE` - przetwarzanie treści (odpowiedzi)
  - przykładowe moduły: `ngx_http_index_module`, `ngx_http_autoindex_module`, `ngx_http_gzip_module`

- `NGX_HTTP_LOG_PHASE` - przetwarzanie logowania, tj. zapisywanie do pliku z logami
  - przykładowe moduły: `ngx_http_log_module`

Przygotowałem również prosty schemat, który pomoże ci zrozumieć, jakie moduły są używane w każdej z faz:

<img src="/assets/img/posts/nginx_phases.png" align="center" title="nginx_phases.png preview">

Na każdej fazie można zarejestrować dowolną liczbę handlerów. Każda faza ma listę powiązanych z nią procedur obsługi.

Polecam przeczytać świetne wyjaśnienie dotyczące [faz przetwarzania żądań](http://scm.zoomquiet.top/data/20120312173425/index.html) i, oczywiście, oficjalny przewodnik [Development guide](http://nginx.org/en/docs/dev/development_guide.html).

Wróćmy teraz do naszego problemu i "dziwnego" zachowania dyrektywy `deny` w połączeniu z wykorzystaniem dyrektywy `return` w celu natychmiastowego przesłania odpowiedzi do klienta. Jak już wspomniałem, zynika to z faktu, że przetwarzanie żądania odbywa się w fazach, a faza przepisywania (do której należy dyrektywa `return`) wykonywana jest przed fazą dostępu (gdzie działa dyrektywa `deny`).

Niestety NGINX nie zgłasza nic niepokojącego (bo i po co) więc odpowiedzialność poprawnego budowania reguł filtrujących wraz z pozostałymi mechanizmami spada na administratora.

Jednym z rozwiązań jest użycie instrukcji `if` w połączeniu z modułami `geo` lub `map`.

  > Nie zaleca się używania instrukcji `if`, chociaż moim zdaniem, użycie takiej konstrukcji może być nieco bardziej elastyczne oraz bezpieczniejsze dzięki wykorzystaniu ww. modułów.

Planując budowanie list kontroli dostępu, rozważ kilka opcji, z których możesz skorzystać. NGINX dostarcza moduły `ngx_http_access_module`, `ngx_http_geo_module`, `ngx_http_map_module` lub `ngx_http_auth_basic_module`, które pozwalają na nadawanie dostępów i zabezpieczanie miejsc w aplikacji.

Zawsze powinieneś przetestować swoje reguły przed ich ostatecznym wdrożeniem:

- sprawdź wszystkie wykorzystane dyrektywy i ich występowanie/priorytety na wszystkich fazach
- wykonaj kilka testowych request'ów w celu potwierdzenia poprawnego działania mechnizmów zezwalających lub blokujących dostęp do zasobów Twojej aplikacji
- wykonaj kilka testowych request'ów w celu sprawdzenia i weryfikacji kodów odpowiedzi HTTP dla wszystkich chronionych zasobów
- mniej znaczy więcej; należy zminimalizować dostęp każdego użytkownika do krytycznych zasobów
- dodaj tylko naprawdę wymagane adresy IP i sprawdź ich właściciela w bazie danych whois
- regularnie poddawaj weryfikacji swoje reguły kontroli dostępu, aby upewnić się, że są aktualne
