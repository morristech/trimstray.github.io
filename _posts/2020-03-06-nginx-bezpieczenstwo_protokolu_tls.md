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

# Używaj tylko TLS w wersji 1.3 oraz 1.2

Zaleca się włączyć TLS 1.2/1.3 oraz całkowicie wyłączyć SSLv2, SSLv3, TLS 1.0 i TLS 1.1, które mają słabości protokołu i używają starszych zestawów szyfrów (nie zapewniają żadnych nowoczesnych trybów szyfrowania), których tak naprawdę nie powinniśmy już używać.

TLS 1.2 jest obecnie najczęściej używaną wersją TLS i wprowadził kilka ulepszeń w zakresie bezpieczeństwa w porównaniu do TLS 1.1. Zdecydowana większość witryn obsługuje TLSv1.2, ale wciąż istnieją takie, które tego nie robią (co więcej, wciąż nie wszyscy klienci są kompatybilni z każdą wersją TLS). Protokół TLS 1.3 jest najnowszą i bardziej niezawodną wersją protokołu TLS i powinien być używany tam, gdzie to możliwe (i tam, gdzie nie jest wymagana kompatybilność wsteczna). Największą zaletą porzucenia TLS 1.0 i 1.1 jest to, że nowoczesne szyfry AEAD są obsługiwane tylko przez TLS 1.2 i nowsze wersje.

<p align="center">
  <img src="/assets/img/posts/qualys_tls_stats.png">
</p>

TLS 1.0 i TLS 1.1 nie powinny być używane (patrz [Deprecating TLSv1.0 and TLSv1.1](https://tools.ietf.org/id/draft-moriarty-tls-oldversions-diediedie-00.html) <sup>[IETF]</sup>) i zostały zastąpione przez TLS 1.2, który sam został zastąpiony przez TLS 1.3 (musi zostać dołączony do 1 stycznia 2024 r.). Te wersje TLS są również aktywnie wycofywane zgodnie z wytycznymi agencji rządowych (np. [NIST Special Publication (SP) 800-52 Revision 2](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-52r2.pdf) <sup>[NIST, pdf]</sup>) i konsorcjów branżowych, takich jak PCI Payment Association Association (PCI) [PCI-TLS - Migrating from SSL and Early TLS (Information Suplement)](https://www.pcisecuritystandards.org/documents/Migrating-from-SSL-Early-TLS-Info-Supp-v1_1.pdf) <sup>[pdf]</sup>.

Obecnie przeglądarki także podchodzą do tematu obsługiwanych wersji TLS dosyć restrykcyjnie. Na przykład w marcu 2020 r. [Mozilla wyłączy obsługę TLS 1.0 i TLS 1.1](https://blog.mozilla.org/security/2018/10/15/removing-old-versions-of-tls/) w najnowszych wersjach przeglądarek Firefox.

Moim zdaniem, trzymanie się TLS 1.0 to bardzo zły i dość niebezpieczny pomysł. Ta wersja może być podatna na ataki POODLE, BEAST, a także padding-Oracle. Nadal obowiązuje wiele innych słabości posiadających identyfikatory CVE, których nie można naprawić, chyba że przez wyłączenie TLS 1.0.

Następnie, trzymanie się TLS 1.1 jest tylko złym kompromisem, chociaż jest w połowie wolne od problemów TLS 1.0. Z drugiej strony czasami ich stosowanie jest nadal wymagane w praktyce (do obsługi starszych klientów). Istnieje wiele innych zagrożeń bezpieczeństwa spowodowanych wykorzystywaniem TLS 1.0 lub 1.1, dlatego zdecydowanie polecam wszystkim aktualizację swoich klientów, usług i urządzeń w celu obsługi min. TLS 1.2.

Usunięcie starszych wersji SSL/TLS jest często jedynym sposobem zapobiegania atakom na obniżenie wersji. Google zaproponowało rozszerzenie protokołu SSL/TLS o nazwie `TLS_FALLBACK_SCSV`, które ma na celu zapobieganie wymuszonym obniżeniom wersji SSL/TLS (rozszerzenie zostało przyjęte jako [RFC 7507](https://tools.ietf.org/html/rfc7507) w kwietniu 2015 r.).

Sama aktualizacja nie jest wystarczająca. Musisz wyłączyć SSLv2 i SSLv3 - więc jeśli twój serwer nie zezwala na połączenia SSLv3 (lub v2), atak typu downgrade nie zadziała. Technicznie `TLS_FALLBACK_SCSV` jest nadal przydatny przy wyłączonym SSL, ponieważ pomaga uniknąć obniżenia połączenia do TLS <1.2. Aby przetestować to rozszerzenie, przeczytaj [ten](https://dwradcliffe.com/2014/10/16/testing-tls-fallback.html) świetny artykuł.

Jeżeli chodzi o najnowsze wersje TLS. Zarówno TLS 1.2, jak i TLS 1.3 nie mają problemów z bezpieczeństwem. Tylko te wersje zapewniają nowoczesne algorytmy kryptograficzne oraz dodają rozszerzenia TLS i zestawy szyfrów. TLS 1.2 poprawia pakiety szyfrów, które zmniejszają zależność od szyfrów blokowych, które zostały wykorzystane przez ataki typu BEAST i wspomnianego POODLE. TLS 1.3 to nowa wersja TLS, która zapewni szybszą i bezpieczniejszą sieć przez kilka następnych lat. Co więcej, TLS 1.3 jest dostarczany bez mnóstwa rzeczy (został usunięty): renegocjacja, kompresja i wiele starszych algorytmów: szyfrów DSA, RC4, SHA1, MD5 i CBC. Ponadto, jak już wspomniano, protokoły TLS 1.0 i TLS 1.1 zostaną usunięte z przeglądarek na początku 2020 r.

TLS 1.2 wymaga starannej konfiguracji, aby zapewnić, że przestarzałe zestawy szyfrów ze zidentyfikowanymi podatnościami nie będą używane w połączeniu z nim. TLS 1.3 eliminuje potrzebę podejmowania tych decyzji i nie wymaga żadnej konkretnej konfiguracji, ponieważ wszystkie szyfry są bezpieczne, a domyślnie OpenSSL włącza tylko GCM i Chacha20 / Poly1305 dla TLSv1.3, bez włączania CCM. Wersja TLS 1.3 poprawia także bezpieczeństwo, prywatność i wydajność TLS 1.2.

Myślę, że najlepszym sposobem na wdrożenie bezpiecznej konfiguracji jest: włączenie TLS 1.2 (jako wersja minimalna; jest wystarczająco bezpieczny) bez szyfrów `CBC` (`ChaCha20` + `Poly1305` lub `AES/GCM` powinny być preferowane nad `CBC` (por. atak BEAST), jednak, używanie szyfrów CBC nie stanowi samo w sobie luki, Zombie POODLE itp. to luki) i / lub TLS 1.3, który jest bezpieczniejszy ze względu na poprawę obsługi i wykluczenie wszystkiego, co stało się przestarzałe od czasu pojawienia się TLS 1.2. Zatem uczynienie TLS 1.2 „minimalnym poziomem protokołu” to solidny wybór i najlepsza praktyka w branży (wszystkie standardy branżowe, takie jak PCI-DSS, HIPAA, NIST, zdecydowanie sugerują stosowanie TLS 1.2 niż TLS 1.1/1.0).

TLS 1.2 jest prawdopodobnie niewystarczający do obsługi starszego klienta. Wytyczne NIST nie mają zastosowania do wszystkich przypadków użycia i zawsze należy przeanalizować bazę użytkowników przed podjęciem decyzji, które protokoły mają być obsługiwane lub upuszczane (na przykład poprzez dodanie zmiennych formatów TLS i szyfrów do formatu dziennika). Należy pamiętać, że nie każdy klient obsługuje najnowszą i najlepszą ofertę TLS.

Jeśli powiedziałeś NGINX, aby używał TLS 1.3, użyje TLS 1.3 tylko tam, gdzie jest dostępny. NGINX obsługuje TLS 1.3 od wersji 1.13.0 (wydanej w kwietniu 2017 r.), Gdy jest zbudowany na OpenSSL 1.1.1 lub nowszym.

  > W przypadku TLS 1.3 zastanów się nad użyciem [ssl_early_data](https://github.com/tlswg/tls13-spec/issues/1001), aby zezwolić na uzgadnianie TLS 1.3 0-RTT.

Przykłady konfiguracji:

LS 1.3 + 1.2:

```nginx
ssl_protocols TLSv1.3 TLSv1.2;
```

TLS 1.2:

```nginx
ssl_protocols TLSv1.2;
```

&nbsp;&nbsp;:arrow_right: ssllabs score: <b>100%</b>

TLS 1.3 + 1.2 + 1.1:

```nginx
ssl_protocols TLSv1.3 TLSv1.2 TLSv1.1;
```

TLS 1.2 + 1.1:

```nginx
ssl_protocols TLSv1.2 TLSv1.1;
```

&nbsp;&nbsp;:arrow_right: ssllabs score: <b>95%</b>

Zewnętrzne zasoby:

- [The Transport Layer Security (TLS) Protocol Version 1.2](https://www.ietf.org/rfc/rfc5246.txt) <sup>[IETF]</sup>
- [The Transport Layer Security (TLS) Protocol Version 1.3](https://tools.ietf.org/html/draft-ietf-tls-tls13-18) <sup>[IETF]</sup>
- [TLS1.2 - Every byte explained and reproduced](https://tls12.ulfheim.net/)
- [TLS1.3 - Every byte explained and reproduced](https://tls13.ulfheim.net/)
- [TLS1.3 - OpenSSLWiki](https://wiki.openssl.org/index.php/TLS1.3)
- [TLS v1.2 handshake overview](https://medium.com/@ethicalevil/tls-handshake-protocol-overview-a39e8eee2cf5)
- [An Overview of TLS 1.3 - Faster and More Secure](https://kinsta.com/blog/tls-1-3/)
- [A Detailed Look at RFC 8446 (a.k.a. TLS 1.3)](https://blog.cloudflare.com/rfc-8446-aka-tls-1-3/)
- [Differences between TLS 1.2 and TLS 1.3](https://www.wolfssl.com/differences-between-tls-1-2-and-tls-1-3/)
- [TLS 1.3 in a nutshell](https://assured.se/2018/08/29/tls-1-3-in-a-nut-shell/)
- [TLS 1.3 is here to stay](https://www.ssl.com/article/tls-1-3-is-here-to-stay/)
- [TLS 1.3: Everything you need to know](https://securityboulevard.com/2019/07/tls-1-3-everything-you-need-to-know/)
- [TLS 1.3: better for individuals - harder for enterprises](https://www.ncsc.gov.uk/blog-post/tls-13-better-individuals-harder-enterprises)
- [How to enable TLS 1.3 on Nginx](https://ma.ttias.be/enable-tls-1-3-nginx/)
- [How to deploy modern TLS in 2019?](https://blog.probely.com/how-to-deploy-modern-tls-in-2018-1b9a9cafc454)
- [Deploying TLS 1.3: the great, the good and the bad](https://media.ccc.de/v/33c3-8348-deploying_tls_1_3_the_great_the_good_and_the_bad)
- [Downgrade Attack on TLS 1.3 and Vulnerabilities in Major TLS Libraries](https://www.nccgroup.trust/us/about-us/newsroom-and-events/blog/2019/february/downgrade-attack-on-tls-1.3-and-vulnerabilities-in-major-tls-libraries/)
- [How does TLS 1.3 protect against downgrade attacks?](https://blog.gypsyengineer.com/en/security/how-does-tls-1-3-protect-against-downgrade-attacks.html)
- [Phase two of our TLS 1.0 and 1.1 deprecation plan](https://www.fastly.com/blog/phase-two-our-tls-10-and-11-deprecation-plan)
- [Deprecating TLSv1.0 and TLSv1.1 (IETF)](https://tools.ietf.org/id/draft-moriarty-tls-oldversions-diediedie-00.html) <sup>[IETF]</sup>
- [Deprecating TLS 1.0 and 1.1 - Enhancing Security for Everyone](https://www.keycdn.com/blog/deprecating-tls-1-0-and-1-1)
- [End of Life for TLS 1.0/1.1](https://support.umbrella.com/hc/en-us/articles/360033350851-End-of-Life-for-TLS-1-0-1-1-)
- [Legacy TLS is on the way out: Start deprecating TLSv1.0 and TLSv1.1 now](https://scotthelme.co.uk/legacy-tls-is-on-the-way-out/)
- [TLS/SSL Explained – Examples of a TLS Vulnerability and Attack, Final Part](https://www.acunetix.com/blog/articles/tls-vulnerabilities-attacks-final-part/)
- [A Challenging but Feasible Blockwise-Adaptive Chosen-Plaintext Attack on SSL](https://eprint.iacr.org/2006/136)
- [TLS/SSL hardening and compatibility Report 2011](http://www.g-sec.lu/sslharden/SSL_comp_report2011.pdf) <sup>[pdf]</sup>
- [This POODLE bites: exploiting the SSL 3.0 fallback](https://security.googleblog.com/2014/10/this-poodle-bites-exploiting-ssl-30.html)
- [New Tricks For Defeating SSL In Practice](https://www.blackhat.com/presentations/bh-dc-09/Marlinspike/BlackHat-DC-09-Marlinspike-Defeating-SSL.pdf) <sup>[pdf]</sup>
- [Are You Ready for 30 June 2018? Saying Goodbye to SSL/early TLS](https://blog.pcisecuritystandards.org/are-you-ready-for-30-june-2018-sayin-goodbye-to-ssl-early-tls)
- [What Happens After 30 June 2018? New Guidance on Use of SSL/Early TLS](https://blog.pcisecuritystandards.org/what-happens-after-30-june-2018-new-guidance-on-use-of-ssl/early-tls-)
- [Mozilla Security Blog - Removing Old Versions of TLS](https://blog.mozilla.org/security/2018/10/15/removing-old-versions-of-tls/)
- [Google - Modernizing Transport Security](https://security.googleblog.com/2018/10/modernizing-transport-security.html)
- [These truly are the end times for TLS 1.0, 1.1](https://www.theregister.co.uk/2020/02/10/tls_10_11_firefox_complete_eradication/)
- [Who's quit TLS 1.0?](https://who-quit-tls10.com/)
- [Recommended Cloudflare SSL configurations for PCI compliance](https://support.cloudflare.com/hc/en-us/articles/205043158-PCI-compliance-and-Cloudflare-SSL#h_8d214b26-c3e5-4632-8056-d2ccd08790dd)
- [Cloudflare SSL cipher, browser, and protocol support](https://support.cloudflare.com/hc/en-us/articles/203041594-Cloudflare-SSL-cipher-browser-and-protocol-support)
- [SSL and TLS Deployment Best Practices](https://github.com/ssllabs/research/wiki/SSL-and-TLS-Deployment-Best-Practices)
- [What level of SSL or TLS is required for HIPAA compliance?](https://luxsci.com/blog/level-ssl-tls-required-hipaa.html)
- [SSL Labs Grade Change for TLS 1.0 and TLS 1.1 Protocols](https://blog.qualys.com/ssllabs/2018/11/19/grade-change-for-tls-1-0-and-tls-1-1-protocols)
- [ImperialViolet - TLS 1.3 and Proxies](https://www.imperialviolet.org/2018/03/10/tls13.html)
