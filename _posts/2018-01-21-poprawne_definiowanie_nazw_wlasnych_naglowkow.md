---
layout: post
title: 'Poprawne definiowanie nazw własnych nagłówków'
date: 2018-01-21 07:50:11
categories: [publications]
tags: [PL, http, headers]
comments: false
favorite: false
seo:
  date_modified: 2020-02-18 22:13:20 +0100
---

Początki konwencji `X-` można znaleźć w sugestii Briana Harveya z 1975 r. w odniesieniu do parametrów protokołu FTP [RFC691](https://tools.ietf.org/html/rfc691). Konwencja ta jest kontynuowana z różnymi specyfikacjami w tym dla nagłówków protokołu HTTP. W tym krótkim wpisie chciałbym omówić prawidłowy sposób definiowania nazw nowych nagłówków oraz dlaczego nie powinno się stosować nazewnictwa z prefiksem `X`.

Używanie niestandardowych nagłówków z prefiksem `X` nie jest zabronione, ale odradzane. Innymi słowy, możesz nadal używać nagłówków rozpoczynających się tym prefiksem, jednak nie jest to zalecane i nie możesz ich traktować tak, jakby były ogólnym standardem.

<p align="center">
  <img src="/assets/img/posts/http_headers_x_prefix.png">
</p>

`X` przed nazwą nagłówka zwyczajowo oznaczało, że jest on eksperymentalny/niestandardowy dla danego dostawcy lub produktu. Gdy nagłówek taki stanie się standardową częścią protokołu HTTP, powinien utracić prefiks zawarty w swojej nazwie.

Oczywiście nie zawsze tak się dzieje i w wielu przypadkach stosowane nagłówki posiadają w swojej nazwie ten prefiks, np. `X-Forwarded-For` lub `X-Requested-With` (jednak są one nadal traktowane jako niestandardowe).

  > Jeśli możliwe jest ujednolicenie nowego niestandardowego nagłówka, użyj nieużywanej i znaczącej nazwy nagłówka.

Dokładne wyjaśnienie znajduje się w [RFC 6648](https://tools.ietf.org/html/rfc6648):

  > _[...] application protocols have often distinguished between standardized and unstandardized parameters by prefixing the names of unstandardized parameters with the string "X-" or similar constructs (e.g., "x."), where the "X" is commonly understood to stand for "eXperimental" or "eXtension"._

A także:

- 3. Recommendations for Creators of New Parameters:

  > _[...] SHOULD NOT prefix their parameter names with "X-" or similar constructs._

- 4. Recommendations for Protocol Designers:

  > _[...] SHOULD NOT prohibit parameters with an "X-" prefix or similar constructs from being registered. [...] MUST NOT stipulate that a parameter with an "X-" prefix or similar constructs needs to be understood as unstandardized. [...] MUST NOT stipulate that a parameter without an "X-" prefix or similar constructs needs to be understood as standardized._

Jednak czy takie zalecenia nie wprowadzają lekkiego zamieszania? Moim zdaniem nie ma tragedii w stosowaniu obu sposobów nazewnictwa. Co więcej, niekiedy nagłówki z prefiksem `X` są łatwiejsze do interpretacji a ewentualne usunięcie początkowego `X` z nazwy może mieć negatywny wpływ na aplikację. Więc by zachować zgodność wstecz, w takim wypadku, należy je zachować.

Jednym z zaleceń stosowania niestandardowych nagłówków jest dodanie, zamiast omawianego prefiksu, początkowej nazwy organizacji.

Przykład implementacji własnego nagłówka z poziomu serwera NGINX:

- konfiguracja niezalecana:

```nginx
add_header X-Backend-Server $hostname;
```

- konfiguracja zalecana:

```nginx
add_header Backend-Server $hostname;
```
