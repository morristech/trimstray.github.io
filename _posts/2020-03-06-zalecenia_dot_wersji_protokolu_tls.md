---
layout: post
title: 'Zalecenia dot. wersji protokołu TLS'
date: 2020-03-06 07:39:05
categories: [publications]
tags: [PL, http, https, nginx, tls, best-practices]
comments: false
favorite: false
seo:
  date_modified: 2020-02-23 09:14:19 +0100
---

Istnieje wiele potencjalnych zagrożeń związanych z wdrażaniem protokołu TLS. Każda konfiguracja wykorzystująca ten protokół powinna spełniać zalecenia poprawnej implementacji i być zgodna ze standardami branżowymi. Wydawać by się mogło, że konfiguracja TLS jest czynnością bardzo prostą i wdrożenie nie powinno być wymagającym procesem. Nic bardziej mylnego.

W tym wpisie chciałbym poruszyć kwestię obsługiwanych wersji TLS, na przykładzie serwera NGINX. Przedstawię także na co powinniśmy zwracać uwagę oraz dlaczego określenie zalecanych wersji tego protokołu jest tak ważne.

Poniżej znajduje się tabela z wszystkimi dostępnymi wersjami SSL/TLS:

| <b>PROTOCOL</b> | <b>RFC</b> | <b>PUBLISHED</b> | <b>STATUS</b> |
| :---:        | :---:        | :---:        | :---         |
| SSL 1.0 | | Unpublished | Unpublished |
| SSL 2.0 | | 1995 | Depracated in 2011 ([RFC 6176](https://tools.ietf.org/html/rfc6176)) <sup>[IETF]</sup> |
| SSL 3.0 | | 1996 | Depracated in 2015 ([RFC 7568](https://tools.ietf.org/html/rfc7568)) <sup>[IETF]</sup> |
| TLS 1.0 | [RFC 2246](https://tools.ietf.org/html/rfc2246) <sup>[IETF]</sup> | 1999 | Deprecation in 2020 |
| TLS 1.1 | [RFC 4346](https://tools.ietf.org/html/rfc4346) <sup>[IETF]</sup> | 2006 | Deprecation in 2020 |
| TLS 1.2 | [RFC 5246](https://tools.ietf.org/html/rfc5246) <sup>[IETF]</sup> | 2008 | Still secure |
| TLS 1.3 | [RFC 8446](https://tools.ietf.org/html/rfc8446) <sup>[IETF]</sup> | 2018 | Still secure |

# TLS w wersji 1.2 i 1.3

Zaleca się włączyć TLSv1.2 oraz TLSv1.3, oraz całkowicie wyłączyć SSLv2, SSLv3, TLSv1.0 i TLSv1.1, które mają słabości protokołu i używają starszych zestawów szyfrów (nie zapewniają żadnych nowoczesnych trybów szyfrowania), których tak naprawdę nie powinniśmy obecnie używać.

Największą zaletą porzucenia TLSv1.0/v1.1 jest to, że nowoczesne szyfry `AEAD` są obsługiwane tylko przez TLSv1.2 i nowsze wersje.

TLSv1.2 jest obecnie najczęściej używaną wersją TLS i wprowadza kilka ulepszeń w zakresie bezpieczeństwa w porównaniu do starszych wersji. Zdecydowana większość witryn obsługuje TLSv1.2, ale wciąż istnieją takie, które tego nie robią (co więcej, wciąż nie wszyscy klienci są kompatybilni z każdą wersją TLS).

<p align="center">
  <img src="/assets/img/posts/qualys_tls_stats.png">
</p>

Protokół TLSv1.3 jest najnowszą i znacznie bezpieczniejszą wersją. Powinien być używany tam, gdzie to możliwe (i tam, gdzie nie jest wymagana kompatybilność wsteczna).

## Szyfry AEAD

Szyfry `AEAD` dostarczają wyspecjalizowane tryby działania szyfru blokowego zwane szyfrowaniem uwierzytelnionym z powiązanymi/dodatkowymi danymi (ang. _Authenticated Encryption with Associated Data_). Łączą one funkcje trybów gwarantujących poufność, a także zapewniają silne uwierzytelnanie oraz wymianę kluczy z funkcją utajniania z wyprzedzeniem (ang. _forward secrecy_).

Potrzeba ich użycia wynika ze słabości wcześniejszych schematów szyfrowania. Szyfry te są jedynymi obsługiwanymi szyframi w TLSv1.3. Powinniśmy z nich korzystać także w przypadku TLSv1.2, włączając tylko te szyfry wykorzystujące algorytmy `AES-GCM` i `ChaCha20-Poly1305`.

Jeżeli chodzi o ten typ szyfrów, to pozwolę sobie zacytować pewną wypowiedź znalezioną na Stack Exchange:

  > _AEAD stands for "Authenticated Encryption with Additional Data" meaning there is a built-in message authentication code for integrity checking both the ciphertext and optionally additional authenticated (but unencrypted) data, and the only AEAD cipher suites in TLS are those using the AES-GCM and ChaCha20-Poly1305 algorithms, and they are indeed only supported in TLS 1.2. This means that if you have any clients trying to connect to this system that don't support either TLS 1.2, or even those that do support TLS 1.2 but not those specific cipher suites (and they're not mandatory... Only TLS_RSA_WITH_AES_128_CBC_SHA is mandatory, and it isn't an AEAD cipher suite) then those clients will not be able to connect at all._ - [Xander](https://security.stackexchange.com/a/136181)

Polecam także [TLS 1.3 (with AEAD) and TLS 1.2 cipher suites demystified: how to pick your ciphers wisely](https://www.cloudinsidr.com/content/tls-1-3-and-tls-1-2-cipher-suites-demystified-how-to-pick-your-ciphers-wisely/).

# Dlaczego powinniśmy wyłączyć starsze wersje SSL/TLS?

TLSv1.0 i TLSv1.1 nie powinny być używane (patrz [Deprecating TLSv1.0 and TLSv1.1](https://tools.ietf.org/id/draft-moriarty-tls-oldversions-diediedie-00.html) <sup>[IETF]</sup>) i zostały zastąpione przez TLSv1.2, który sam został zastąpiony przez TLSv1.3 (powinien zostać dołączony do każdej konfiguracji do 1 stycznia 2024 r.). Te wersje TLS są również aktywnie wycofywane zgodnie z wytycznymi agencji rządowych (np. [NIST Special Publication (SP) 800-52 Revision 2](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-52r2.pdf) <sup>[NIST, pdf]</sup>) i konsorcjów branżowych, takich jak PCI Payment Association Association ([PCI-TLS - Migrating from SSL and Early TLS (Information Suplement)](https://www.pcisecuritystandards.org/documents/Migrating-from-SSL-Early-TLS-Info-Supp-v1_1.pdf) <sup>[pdf]</sup>).

Obecnie przeglądarki także podchodzą do tematu obsługiwanych wersji TLS dosyć restrykcyjnie. Na przykład w marcu 2020 r. [Mozilla wyłączy obsługę TLSv1.0 i TLSv1.1](https://blog.mozilla.org/security/2018/10/15/removing-old-versions-of-tls/) w najnowszych wersjach przeglądarek Firefox.

Moim zdaniem, trzymanie się TLSv1.0 to bardzo zły i dość niebezpieczny pomysł. Ta wersja może być podatna na ataki [POODLE](https://en.wikipedia.org/wiki/POODLE), [BEAST](https://en.wikipedia.org/wiki/Transport_Layer_Security#BEAST_attack), a także [padding-Oracle](https://en.wikipedia.org/wiki/Padding_oracle_attack). Nadal obowiązuje wiele innych słabości posiadających identyfikatory CVE (niektóre zostały opisane w dokumencie [TLS Security 6: Examples of TLS Vulnerabilities and Attacks](https://www.acunetix.com/blog/articles/tls-vulnerabilities-attacks-final-part/), których nie można naprawić, chyba że przez wyłączenie TLSv1.0.

Obsługa wersji TLSv1.1 jest tylko złym kompromisem, chociaż ta wersja jest w połowie wolna od problemów TLSv1.0. Z drugiej strony czasami jej stosowanie jest nadal wymagane w praktyce (do obsługi starszych klientów). Istnieje wiele innych zagrożeń bezpieczeństwa spowodowanych wykorzystywaniem TLSv1.0 lub TLSv1.1, dlatego zdecydowanie zalecam aktualizację oprogramowania, usług i urządzeń w celu obsługi min. TLSv1.2.

Usunięcie starszych wersji SSL/TLS jest często jedynym sposobem zapobiegania atakom na obniżenie wersji. Google zaproponowało rozszerzenie protokołu SSL/TLS o nazwie `TLS_FALLBACK_SCSV`, które ma na celu zapobieganie wymuszonym obniżeniom wersji SSL/TLS (rozszerzenie zostało przyjęte jako [RFC 7507](https://tools.ietf.org/html/rfc7507) w kwietniu 2015 r.).

Sama aktualizacja nie jest wystarczająca. Musisz wyłączyć SSLv2 i SSLv3 - więc jeśli twój serwer nie zezwala na połączenia z tymi wersjami protokołu, atak typu downgrade nie zadziała. Technicznie `TLS_FALLBACK_SCSV` jest nadal przydatny przy wyłączonych wersjach SSL, ponieważ pomaga uniknąć obniżenia połączenia do TLS <1.2. Aby przetestować to rozszerzenie, przeczytaj [ten](https://dwradcliffe.com/2014/10/16/testing-tls-fallback.html) świetny artykuł.

# Czy TLSv1.2 jest w pełni bezpieczny?

Jeżeli chodzi o najnowsze wersje TLS, zarówno TLSv1.2, jak i TLSv1.3 nie mają problemów z bezpieczeństwem (TLSv1.2 tak naprawdę dopiero po spełnieniu określonych warunków, np. wyłączenie szyfrów `CBC`). Tylko te wersje zapewniają nowoczesne algorytmy kryptograficzne oraz dodają rozszerzenia TLS i zestawy szyfrów. TLSv1.2 poprawia zestawy szyfrów, które zmniejszają zależność od szyfrów blokowych, które zostały wykorzystane przez ataki typu BEAST i wspomniany wcześniej POODLE.

Co ciekawe, Craig Young, badacz bezpieczeństwa w zespole firmy Tripwire, znalazł luki w TLSv1.2, które pozwalają na ataki podobne do POODLE ze względu na ciągłe wsparcie TLSv1.2 dla dawno przestarzałych metod kryptograficznych, tj. szyfrów blokowych `CBC`. Znalezione słabości umożliwiają ataki typu man-in-the-middle (MitM) na zaszyfrowane sesje użytkownika.

  > Używanie szyfrów `CBC` nie stanowi samo w sobie luki, którymi de fakto są luki w zabezpieczeniach, takie jak Zombie POODLE, GOLDENDOODLE, 0-Length OpenSSL i Sleeping POODLE, które zostały opublikowane dla serwisów korzystających z trybów szyfrowania blokowego `CBC` (ang. _Cipher Block Chaining_). Luki te mają zastosowanie tylko wtedy, gdy serwer używa TLSv1.0, TLSv1.1 lub TLSv1.2 z trybami szyfrowania `CBC`. Spójrz na [Zombie POODLE, GOLDENDOODLE, & How TLSv1.3 Can Save Us All](https://i.blackhat.com/asia-19/Fri-March-29/bh-asia-Young-Zombie-Poodle-Goldendoodle-and-How-TLSv13-Can-Save-Us-All.pdf) <sup>[pdf]</sup>  z Black Hat Asia 2019. Na TLSv1.0 i TLSv1.1 mogą mieć wpływ luki, takie jak [FREAK, POODLE, BEAST i CRIME](https://www.acunetix.com/blog/articles/tls-vulnerabilities-attacks-final-part/).

Oprócz tego TLSv1.2 wymaga starannej konfiguracji, aby zapewnić, że przestarzałe zestawy szyfrów ze zidentyfikowanymi podatnościami nie będą używane. TLSv1.3 eliminuje potrzebę podejmowania tych decyzji i nie wymaga żadnej konkretnej konfiguracji, ponieważ wszystkie szyfry są bezpieczne, a domyślnie OpenSSL włącza tylko tryby `GCM` i `Chacha20/Poly1305` dla TLSv1.3, bez włączania `CCM`.

# Włączenie TLSv1.3

TLSv1.3 to nowa wersja TLS, która zapewnia szybszą i bezpieczniejszą komunikację przez kilka następnych lat (poprawia także bezpieczeństwo, prywatność i wydajność TLSv1.2). Co więcej, TLSv1.3 jest dostarczany bez wielu rzeczy (zostały usunięte): renegocjacja, kompresja i wiele starych i słabych szyfrów, tj. `DSA`, `RC4`, `SHA1`, `MD5` i `CBC`.

Niestety nie jest jeszcze w pełni wspierany przez wszystkich klientów:

<p align="center">
  <img src="/assets/img/posts/tlsv1.3_support.png">
</p>

NGINX wspiera TLSv1.3 od wersji 1.13.0 wydanej w kwietniu 2017 r., pod warunkiem, że obsługiwaną wersją biblioteki OpenSSL jest min. 1.1.1 (lub nowszy).

# Zalecenia

Myślę, że najlepszym sposobem na wdrożenie bezpiecznej konfiguracji jest włączenie TLSv1.2 (jako minimalna obsługiwana wersja) bez szyfrów `CBC` z jawnym wskazaniem szyfrów `ChaCha20` + `Poly1305` lub `AES/GCM`, które powinny być preferowane nad `CBC` (por. atak BEAST) lub TLSv1.3, który jest bezpieczniejszy ze względu na poprawę obsługi i wykluczenie wszystkiego, co stało się przestarzałe od czasu pojawienia się TLSv1.2.

Zatem uczynienie TLSv1.2 „minimalnym poziomem protokołu” to solidny wybór i najlepsza praktyka w branży (wszystkie standardy branżowe, takie jak PCI-DSS, HIPAA, NIST, zdecydowanie sugerują stosowanie TLSv1.2 niż TLSv1.1/v1.0).

TLSv1.2 jest prawdopodobnie niewystarczający do obsługi starszego klienta. Wytyczne NIST nie mają zastosowania do wszystkich przypadków użycia i zawsze należy przeanalizować bazę użytkowników przed podjęciem decyzji, które protokoły mają być obsługiwane lub nie (na przykład poprzez dodanie zmiennych formatów TLS i szyfrów do formatu dziennika). Należy pamiętać, że nie każdy klient obsługuje najnowszą i najlepszą ofertę TLS.

  > W przypadku TLSv1.3 zastanów się nad użyciem [ssl_early_data](https://github.com/tlswg/tls13-spec/issues/1001), aby zezwolić na uzgadnianie TLSv1.3 0-RTT.

Jeżeli masz wątpliwości związane z wyłączeniem starszych wersji TLS, np. jak wspomiałem wyłączenie TLSv1.1 może uniemożliwić komunikację starszym klientom, zastanów się jaki jest sens stosowania protokołów nie zapewniających odpowiedniego poziomu bezpieczeństwa, jeżeli dostępne są znacznie lepsze (pod każdym względem) wersje? Skoro mamy możliwość skonfigurowania naszych serwerów do obsługi protokołów technicznie przewyższających ich starsze odmiany, i to bardzo niskim kosztem, nie powinniśmy zastanawiać się ani chwili. Myślę, że jest to całkiem sensowny argument, mimo, że nie kluczowy.

Oczywiście jest wiele opinii na ten temat. Powinniśmy mieć także świadomość tego, że ew. podatność nie zawsze jest prosta do wykorzystania i bardzo często musi zostać spełnionych kilka dodatkowych warunków aby możliwe było jej wykorzystanie.

# Przykłady konfiguracji

TLSv1.3 + 1.2:

```nginx
ssl_protocols TLSv1.3 TLSv1.2;
```

TLSv1.2:

```nginx
ssl_protocols TLSv1.2;
```

&nbsp;&nbsp; ssllabs score: <b>100%</b>

TLSv1.3 + 1.2 + 1.1:

```nginx
ssl_protocols TLSv1.3 TLSv1.2 TLSv1.1;
```

TLSv1.2 + 1.1:

```nginx
ssl_protocols TLSv1.2 TLSv1.1;
```

&nbsp;&nbsp; ssllabs score: <b>95%</b>

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
- [AEAD Ciphers - shadowsocks](https://shadowsocks.org/en/spec/AEAD-Ciphers.html)
- [SSL Labs Grade Change for TLS 1.0 and TLS 1.1 Protocols](https://blog.qualys.com/ssllabs/2018/11/19/grade-change-for-tls-1-0-and-tls-1-1-protocols)
- [ImperialViolet - TLS 1.3 and Proxies](https://www.imperialviolet.org/2018/03/10/tls13.html)
