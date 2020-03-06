---
layout: post
title: 'NGINX: Bezpieczeństwo protokołu TLS'
date: 2020-03-06 07:39:05
categories: [publications]
tags: [PL, http, https, nginx, tls, best-practices]
comments: false
favorite: false
seo:
  date_modified: 2020-02-23 09:14:19 +0100
---

Istnieje wiele potencjalnych zagrożeń związanych z wdrażaniem konfiguracji protokołu TLS. Każda konfiguracja wykorzystująca ten protokół powinna spełniać zalecenia poprawnej implementacji i być zgodna ze standardami branżowymi. Wydawać by się mogło, że konfiguracja TLS jest czynnością bardzo prostą i wdrożenie nie powinno być wymagającym procesem. Nic bardziej mylnego.

W tym wpisie chciałbym przedstawić, na przykładzie serwera NGINX, na co powinniśmy zwracać uwagę oraz dlaczego pewne parametry powinny być ustawione tak a nie inaczej.

# Oddzielne dyrektywy nasłuchiwania dla portów 80 i 443

Pierwszą zasadą od której rozpoczniemy będzie sprawa organizacyjna (w kontekście konfiguracji). NGINX umożliwia definiowanie listener'ów za pomocą których procesy nasłuchują na odpowienich adresach IP oraz portach.

Jeśli w twojej konfiguracji wykorzystujesz oba protokoły, tj. HTTP i HTTPS z dokładnie taką samą konfiguracją (pojedynczy serwer, który obsługuje zarówno żądania HTTP, jak i HTTPS), NGINX jest wystarczająco inteligentny, aby zignorować dyrektywy TLS, jeśli zostanie załadowany przez port 80.

Nie lubię powielać reguł, ale osobne dyrektywy `listen` z pewnością pomogą ci utrzymać i modyfikować konfigurację. Jeśli chcę przekierować ruch z HTTP na HTTPS, właściwym dla mnie sposobem jest zdefiniowanie oddzielnego kontekstu `server` w takich przypadkach.

Jest to również przydatne, jeśli przypniesz wiele domen do jednego adresu IP. Pozwala to dołączyć jedną dyrektywę `listen` (np. jeśli zachowasz ją w pliku konfiguracyjnym) do konfiguracji wielu domen.

  > Powinieneś także użyć dyrektywy `return` do przekierowania z HTTP na HTTPS (aby wszystko zakodować na sztywno, a nie używać w ogóle wyrażeń regularnych).

Przykład konfiguracji:

```nginx
# For HTTP:
server {

  listen 10.240.20.2:80;

  # If you need redirect to HTTPS:
  return 301 https://example.com$request_uri;

  ...

}

# For HTTPS:
server {

  listen 10.240.20.2:443 ssl;

  ...

}
```

Zewnętrzne zasoby:

- [Understanding the Nginx Configuration File Structure and Configuration Contexts](https://www.digitalocean.com/community/tutorials/understanding-the-nginx-configuration-file-structure-and-configuration-contexts)
- [Configuring HTTPS servers](http://nginx.org/en/docs/http/configuring_https_servers.html)

# Wspólna konfiguracja TLS dla adresów nasłuchiwania

Reguła ta także dotyczy organizacji konfiguracji. Ułatwia ona debugowanie i utrzymanie, a także zapobiega wykorzystaniu wielu konfiguracji TLS dla tego samego adresu nasłuchiwania (co jest nie do końca prawdą).

Powinieneś użyć jednej konfiguracji TLS do współdzielenia jednego adresu IP między kilkoma konfiguracjami HTTPS (np. protokoły, szyfry, krzywe eliptyczne). Ma to na celu zapobieganie błędom i chaosowi konfiguracji.

Używanie wspólnej konfiguracji TLS (przechowywanej w jednym pliku i dodawanej za pomocą dyrektywy `include`) dla wszystkich kontekstów `server` zapobiega dziwnym zachowaniom. Myślę, że nie ma lepszego lekarstwa na możliwy bałagan konfiguracji.

Pamiętaj, że niezależnie od parametrów TLS możesz używać wielu certyfikatów SSL w tej samej dyrektywie nasłuchiwania (adres IP). Również niektóre parametry TLS mogą się różnić.

Pamiętaj również o konfiguracji domyślnego serwera. Jest to ważne, ponieważ jeśli żadna z dyrektyw nasłuchiwania nie ma parametru `default_server`, to serwerem domyślnym będzie pierwszym serwer w konfiguracji.

Przykład konfiguracji:

```nginx
# Store it in a file, e.g. https.conf:
ssl_protocols TLSv1.2;
ssl_ciphers "ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305";

ssl_prefer_server_ciphers on;

ssl_ecdh_curve secp521r1:secp384r1;

...

# Include this file to the server context (attach domain-a.com for specific listen directive):
server {

  listen 192.168.252.10:443 default_server ssl http2;

  include /etc/nginx/https.conf;

  server_name domain-a.com;

  ssl_certificate domain-a.com.crt;
  ssl_certificate_key domain-a.com.key;

  ...

}

# Include this file to the server context (attach domain-b.com for specific listen directive):
server {

  listen 192.168.252.10:443 ssl;

  include /etc/nginx/https.conf;

  server_name domain-b.com;

  ssl_certificate domain-b.com.crt;
  ssl_certificate_key domain-b.com.key;

  ...

}
```

Zewnętrzne zasoby:

- [Nginx one ip and multiple ssl certificates](https://serverfault.com/questions/766831/nginx-one-ip-and-multiple-ssl-certificates)
- [Configuring HTTPS servers](http://nginx.org/en/docs/http/configuring_https_servers.html)

# Wymuszenie wszystkich połączeń za pomocą TLS

Protokół TLS zapewnia dwie główne usługi. Po pierwsze, sprawdza tożsamość serwera, z którym użytkownik się łączy. Po drugie chroni przesyłanie poufnych informacji od użytkownika do serwera.

Moim zdaniem zawsze powinieneś używać HTTPS zamiast HTTP (użyj HTTP tylko do przekierowania do HTTPS), aby chronić swoją witrynę, nawet jeśli nie obsługuje ona poufnej komunikacji i nie zawiera żadnych mieszanych treści. Aplikacja może mieć wiele wrażliwych miejsc, które powinny być chronione.

Zawsze umieszczaj strony logowania, formularze rejestracyjne, wszystkie kolejne uwierzytelnione strony, formularze kontaktowe i formularze szczegółów płatności w HTTPS, aby zapobiec sniffowaniu i wstrzykiwaniu (atakujący może wstrzyknąć kod do niezaszyfrowanej transmisji HTTP, więc zawsze zwiększa ryzyko zmiany treści, nawet jeśli ktoś czyta tylko treści niekrytyczne. Zobacz atak typu [Man-in-the-browser](https://owasp.org/www-community/attacks/Man-in-the-browser_attack)). Dostęp do nich można uzyskać tylko przez TLS, aby zapewnić bezpieczeństwo ruchu.

Jeśli aplikacja jest dostępna przez TLS, musi składać się całkowicie z treści przesyłanych przez TLS. Żądanie półśrodków przy użyciu niezabezpieczonego protokołu HTTP osłabia bezpieczeństwo całego serwisu jak i samego protokołu HTTPS. Nowoczesne przeglądarki powinny domyślnie blokować lub zgłaszać wszystkie treści mieszane dostarczane przez HTTP.

Pamiętaj także o wdrożeniu polityki [HTTP Strict Transport Security (HSTS)](https://blog.nsbox.int/publications/2020/01/09/jak_poprawnie_wdrozyc_naglowek_hsts.html) i zapewnieniu właściwej konfiguracji TLS (wersja protokołu, zestawy szyfrów, odpowiedni łańcuch certyfikatów i inne).

Przykład przekierowania całego ruchu z HTTP na HTTPS:

```nginx
server {

  listen 10.240.20.2:80;

  server_name example.com;

  return 301 https://$host$request_uri;

}

server {

  listen 10.240.20.2:443 ssl;

  server_name example.com;

  ...

}
```

Przekierowanie tylko strony logowania:

```nginx
server {

  listen 10.240.20.2:80;

  server_name example.com;

  ...

  location ^~ /login {

    return 301 https://example.com$request_uri;

  }

}
```

Zewnętrzne zasoby:

- [Does My Site Need HTTPS?](https://doesmysiteneedhttps.com/)
- [HTTP vs HTTPS Test](https://www.httpvshttps.com/)
- [Let's Encrypt Documentation](https://letsencrypt.org/docs/)
- [Should we force user to HTTPS on website?](https://security.stackexchange.com/questions/23646/should-we-force-user-to-https-on-website)
- [Force a user to HTTPS](https://security.stackexchange.com/questions/137542/force-a-user-to-https)
- [The Security Impact of HTTPS Interception](https://jhalderm.com/pub/papers/interception-ndss17.pdf) <sup>[pdf]</sup>
