---
layout: post
title: 'HTTPS z samopodpisanym certyfikatem kontra HTTP'
date: 2019-05-19 15:19:22
categories: [publications]
tags: [PL, http, https, certificates, self-signed]
comments: false
favorite: false
seo:
  date_modified: 2020-02-23 09:14:19 +0100
---

W tym wpisie postaram się odpowiedzieć na wcale nie łatwie pytanie: co jest lepsze? HTTPS z samopodpisanym certyfikatem czy transmisja wykorzystująca czysty protokół HTTP?

Odpowiedź nie jest wcale taka oczywista. Z jednej strony szyfrowanie ruchu znacznie zwiększa bezpieczeństwo komunikacji, z drugiej strony, jaką mamy pewność, że wystawca certyfikatu jest tym, za kogo się podaje? Następna sporna kwestia to wydajność. Wydawać by się mogło, że wykorzystanie protokołu HTTP jest znacznie szybsze, ponieważ pomijana zostaje cała obsługa szyfrowania (zestawianie sesji itd.).

Moim zdaniem, certyfikaty samopodpisane nie są gorsze niż certyfikaty podpisane przez renomowany urząd certyfikacji i pod każdym względem technicznym są lepsze niż zwykły HTTP. Z punktu widzenia podpisywania i szyfrowania są one identyczne. Oba mogą podpisywać i szyfrować ruch, więc nie jest możliwe, aby inni szpiegowali lub wprowadzali modyfikacje.

Spójrz na to proste porównanie:

| <b>FUNKCJA</b> | <b>HTTP</b> | <b>HTTPS + SELF-SIGNED</b> |
| :---         | :---         | :---         |
| szyfrowanie | nie | **tak** |
| autoryzacja | nie | nie |
| prywatność | nie | nie (lub **tak**, jeśli ufasz wystawcy w sposób domniemany) |
| wydajność | **szybki** | **szybszy niż HTTP** (przy spełnieniu pewnych warunków) |

# Bezpieczeństwo

Jeżeli chodzi o ten ważny aspekt, to według mnie, certyfikaty samopodpisane nadają się tylko i wyłącznie do celów testowych i usług wewnętrznych, pod warunkiem, że możesz zaufać wystawcy certyfikatu (którym najczęściej jest dział IT w Twojej firmie). W przeciwnym razie, certyfikaty takie, tworzą iluzję bezpieczeństwa (zapewniają tylko szyfrowanie), nic więcej.

Zaufanie jest tutaj kluczowe, ponieważ nadal niejawnie autoryzujesz wystawcę (domniemamy, że serwer urzędu certyfikacji jest bezpieczny), weryfikując go ręcznie. W przypadku certyfikatu self-signed nie sposób dowiedzieć się, kto podpisał certyfikat i czy należy ufać takiemu podmiotowi.

  > Certyfikaty samopodpisane powinny zawsze budzić wątpliwości i być używane tylko w kontrolowanych środowiskach.

## Słów kilka o certyfikacie self-signed

Różnica polega na sposobie oznaczenia certyfikatu jako zaufanego. Z certyfikatem podpisanym przez urząd certyfikacji użytkownik ufa zestawowi zaufanym urzędom certyfikacji zainstalowanym w przeglądarce/systemie operacyjnym. Jeśli przeglądarka zobaczy certyfikat podpisany przez jednego z nich, akceptuje go i wszystko jest w porządku. Jeśli tak nie jest, otrzymasz duże przerażające ostrzeżenie.

To ostrzeżenie jest wyświetlane w przypadku certyfikatów samopodpisanych, ponieważ przeglądarka nie ma pojęcia, kto kontroluje certyfikat. Urzędy certyfikacji, którym ufa, są znane z tego, że weryfikują/podpisują tylko certyfikaty właściciela strony internetowej. Dlatego przeglądarka, poprzez domniemanie, ufa, że ​​odpowiedni klucz prywatny certyfikatu jest kontrolowany przez operatora strony internetowej (i ma nadzieję, że tak jest).

Dzięki samopodpisanemu certyfikatowi przeglądarka nie ma możliwości dowiedzenia się, czy certyfikat został wygenerowany przez właściciela strony internetowej lub stronę trzecią, która może chcieć odczytać ruch. Aby zachować bezpieczeństwo, przeglądarka odrzuca taki certyfikat.

Jeśli nie zweryfikujesz certyfikatu, nic nie zyskasz dzięki niezaszyfrowanemu HTTP, ponieważ ktokolwiek między tobą a serwerem może po prostu wygenerować własny certyfikat i nie będziesz wcale bezpieczniejszy.

# Wydajność

Ważną rzeczą, o której należy pamiętać, jest wydajność. HTTP jest wolniejszy niż HTTPS wykorzystujący HTTP/2 (tj. jedno połączenie TCP, multipleksowanie, kompresja nagłówków HPACK), HSTS, OCSP Stapling i kilka innych ulepszeń, z wyjątkiem początkowego uzgadniania TLS.

Jak wiemy, HTTPS wymaga wstępnego „uścisku dłoni”. Proces ten może być bardzo wolny mimo tego, że rzeczywista ilość danych przesyłanych w ramach uzgadniania nie jest ogromna (zwykle poniżej 5 kB). Jednak w przypadku bardzo małych żądań może to być dość duże obciążenie.

Podczas uzgadniania TLS używane jest szyfrowanie asymetryczne, następnie po ustanowieniu wspólnego klucza (w dużym skrócie, po zakończeniu uzgadniania) wykorzystywane jest szyfrowanie symetryczne, czyli bardzo szybka forma szyfrowania, więc narzut jest minimalny.

Moim zdaniem, wysyłanie wielu krótkich żądań za pomocą protokołu HTTPS będzie nieco wolniejsze niż za pomocą HTTP, jednak jeśli przesyłasz dużo danych w jednym żądaniu, różnica będzie nieznaczna (pamiętajmy także o pamięci podręcznej dla sesji TLS, w NGINX będzie to dyrektywa `ssl_session_cache`) — stąd koszt wydajności nie jest już tak istotny, jak kiedyś.

Oczywiście ciężko jest udzielić sensownej odpowiedzi bez informacji na temat charakteru aplikacji (np. stosunek treści dynamicznej do statycznej), sprzętu, oprogramowania i konfiguracji sieci (np. odległość klienta do serwera).

Polecam zerknąć na [HTTP vs HTTPS Test](http://www.httpvshttps.com/) oraz na [TLS has exactly one performance problem: it is not used widely enough](https://istlsfastyet.com/).
