---
layout: post
title: 'Jak działają łańcuchy certyfikatów?'
date: 2018-09-27 01:32:08
categories: [publications]
tags: [PL, ssl, tls, certificate, chain-of-trust]
comments: false
favorite: false
seo:
  date_modified: 2020-02-21 16:30:58 +0100
---

Kluczową częścią każdego procesu uwierzytelniania opartego na certyfikatach jest walidacja łańcucha certyfikatów, lub inaczej łańcucha zaufania (ang. _chain of trust_).

Łańcuch zaufania to połączona ścieżka weryfikacji od certyfikatu jednostki końcowej do głównego urzędu certyfikacji (CA), który działa jako kotwica zaufania (ang. _trust anchor_), czyli certyfikat, któremu ufasz, ponieważ został Ci dostarczony w drodze pewnej wiarygodnej procedury. Kotwica zaufania to certyfikat urzędu certyfikacji (a ściślej publiczny klucz weryfikacyjny urzędu certyfikacji) używany przez stronę ufającą jako punkt wyjścia do sprawdzania poprawności ścieżki weryfikacji.

Jeśli system nie posiada łańcucha certyfikatów, lub jeśli brakuje certyfikatów pośrednich (przerwany łańcuch), klient nie może sprawdzić, czy certyfikat jest ważny. W związku z tym, certyfikat końcowy traci wszelką użyteczność wraz z tzw. wskaźnikiem zaufania (ang. _metric of trust_).

# Czym właściwie jest łańcuch certyfikatów?

Łańcuch certyfikatów składa się ze wszystkich certyfikatów potrzebnych do weryfikacji podmiotu określonego w certyfikacie końcowym. W praktyce obejmuje to certyfikat serwera, certyfikaty pośrednie urzędów certyfikacji oraz certyfikat głównego urzędu certyfikacji, zaufany przez wszystkie strony w łańcuchu. Każdy pośredni urząd certyfikacji w łańcuchu posiada certyfikat wydany przez urząd certyfikacji jeden poziom nad nim w hierarchii zaufania.

<img src="/assets/img/posts/chain_of_trust.png" align="center" title="chain_of_trust preview">

Przeglądarki oraz systemy operacyjne zawierają listę zaufanych urzędów certyfikacji. Te wstępnie zainstalowane certyfikaty służą jako kotwice zaufania, z których można czerpać dalsze zaufanie (mam nadzieję, że jest to odpowiednie określenie).

Podczas odwiedzania witryny HTTPS przeglądarka sprawdza, czy łańcuch zaufania prezentowany przez serwer podczas uzgadniania TLS kończy się na jednym z lokalnie zaufanych certyfikatów głównych.

# Czym jest certyfikat klucza publicznego?

Certyfikat klucza publicznego to podpisana instrukcja, która służy do ustanowienia powiązania między tożsamością a kluczem publicznym. Zawiera on trzy istotne informacje:

- klucz publiczny podmiotu,
- opis tożsamości podmiotu,
- podpis cyfrowy złożony przez zaufaną trzecią stronę na dwóch powyższych strukturach

Podmiot, który poręczy za to powiązanie i podpisze certyfikat, jest wystawcą certyfikatu, a tożsamość, której klucz publiczny jest potwierdzony, jest przedmiotem certyfikatu. W celu powiązania tożsamości i klucza publicznego wykorzystywany jest właśnie łańcuch certyfikatów.

Certyfikat serwera wraz z łańcuchem nie jest przeznaczony dla serwera. Serwer nie ma zastosowania do własnego certyfikatu. Certyfikaty są zawsze dla innych podmiotów (tutaj klientów). Serwer używa klucza prywatnego (który odpowiada kluczowi publicznemu w certyfikacie). W szczególności, serwer nie musi ufać własnemu certyfikatowi ani żadnemu urzędowi certyfikacji, który go wydał.

## Czy istnieją jakiekolwiek różnice między łańcuchem certyfikatów a łańcuchem zaufania?

Na wstępie stwierdziłem, że oba są tak naprawdę tym samym. I jest to oczywiście prawda, jednak chciałbym zwrócić uwagę na pewien aspekt, dzięki któremu, może istnieć pewna subtelna różnica między nimi.

Łańcuch zaufania traktuję jako uporządkowaną listą certyfikatów, która zawiera certyfikat użytkownika końcowego i certyfikaty pośrednie (które reprezentują pośredni urząd certyfikacji). Dzięki temu, odbiorca ma możliwość sprawdzenie, czy nadawca i wszystkie certyfikaty pośrednie są godne zaufania.

# Dlaczego łańcuch certyfikatów nie powinien zawierać certyfikatu głównego?

Serwer zawsze wysyła łańcuch, ale nigdy nie powinien prezentować łańcuchów certyfikatów zawierających kotwicę zaufania, która jest certyfikatem głównego urzędu certyfikacji, ponieważ katalog główny jest bezużyteczny do celów sprawdzania poprawności. Zgodnie ze standardem TLS łańcuch może zawierać lub nie zawierać samego certyfikatu głównego; klient nie potrzebuje tego certyfikatu głównego, ponieważ już go ma. I rzeczywiście, jeśli klient nie ma jeszcze certyfikatu głównego, wówczas otrzymanie go z serwera nie pomogłoby, ponieważ takiemu certyfikatowi można zaufać tylko dzięki temu, że już tam jest.

Co więcej, obecność kotwicy zaufania na ścieżce certyfikacji może mieć negatywny wpływ na wydajność podczas nawiązywania połączeń za pomocą protokołu SSL/TLS, ponieważ certyfikat główny jest „pobierany” przy każdym uzgadnianiu (zmniejsza zużycie pamięci po stronie serwera dla parametrów sesji TLS).

Zgodnie z zaleceniami, usuń samopodpisany certyfikat główny z serwera. Pakiet certyfikatów powinien zawierać tylko klucz publiczny certyfikatu końcowego i klucz publiczny wszelkich pośrednich urzędów certyfikacji. Przeglądarki będą ufać tylko tym certyfikatom, które przekształcają się w certyfikaty główne, które są już w magazynie zaufanych certyfikatów, zignorują certyfikat główny wysłany w łańcuchu certyfikatów (w przeciwnym razie każdy mógłby wysłać dowolny certyfikat główny).

# Podpisywanie certyfikatów

Spójrz na następujący schemat:

```
ROOT_CERT (isCA=yes)
|
|---INTERMEDIATE_CERT_1 (isCA=yes)
     |
     |---INTERMEDIATE_CERT_2 (isCA=yes)
         |
         |---LEAF_CERT valid for example.com (isCA=no)
```

Gdy urzędy certyfikacji podpisują certyfikaty, podpisują nie tylko klucz publiczny, ale także dodatkowe metadane. Te metadane obejmują m.in. datę wygaśnięcia certyfikatu. Najczęściej jest to zapisywane w formacie danych zdefiniowanym jako **X.509**. Co ważne, główny urząd certyfikacji stwierdza również, czy dany certyfikat może podpisywać inne „(pod) certyfikaty”, a tym samym „certyfikować” je, czy nie.

Zarówno główny urząd certyfikacji, jak i pośrednie urzędy certyfikacji/certyfikaty mają tą samą właściwość. W związku z tym mogą podpisywać inne certyfikaty. Oczywiście, tzw. _leaf certificate_ (certyfikat serwera, końcowy) nie może mieć zgody na podpisywanie innych certyfikatów.

  > Jeśli certyfikat jest podpisany bezpośrednio przez zaufany główny urząd certyfikacji, nie ma potrzeby dodawania żadnych dodatkowych/pośrednich certyfikatów do łańcucha certyfikatów. Główny urząd certyfikacji wydaje dla siebie certyfikat.

# W jaki sposób zbudować poprawnie łańcuch zaufania?

Serwer zawsze wysyła łańcuch, ale nigdy nie powinien prezentować łańcuchów certyfikatów zawierających kotwicę zaufania, która jest certyfikatem głównego urzędu certyfikacji, ponieważ katalog główny jest bezużyteczny do celów sprawdzania poprawności. Zgodnie ze standardem TLS łańcuch może zawierać lub nie zawierać samego certyfikatu głównego; klient nie potrzebuje tego katalogu głównego, ponieważ już go ma. I rzeczywiście, jeśli klient nie ma jeszcze katalogu głównego, wówczas otrzymanie go z serwera nie pomogłoby, ponieważ rootowi można zaufać tylko dzięki temu, że już tam jest.

Co więcej, obecność kotwicy zaufania na ścieżce certyfikacji może mieć negatywny wpływ na wydajność podczas nawiązywania połączeń za pomocą protokołu SSL / TLS, ponieważ katalog główny jest „pobierany” przy każdym uzgadnianiu (pozwala zapisać około 1 kB danych przepustowość na połączenie i zmniejsza zużycie pamięci po stronie serwera dla parametrów sesji TLS).

Aby uzyskać najlepsze praktyki, usuń samopodpisany katalog główny z serwera. Pakiet certyfikatów powinien zawierać tylko klucz publiczny certyfikatu i klucz publiczny wszelkich pośrednich urzędów certyfikacji. Przeglądarki będą ufać tylko tym certyfikatom, które przekształcają się w katalogi główne, które są już w magazynie zaufanych certyfikatów, zignorują certyfikat główny wysłany w pakiecie certyfikatów (w przeciwnym razie każdy mógłby wysłać dowolny katalog główny).

# Co się dzieje, gdy łańcuch jest "przerwany"?

W przypadku przerwania łańcucha nie można zweryfikować, czy serwer, na którym przechowywane są dane i z którym klient nawiązuje połączenie, jest poprawnym (oczekiwanym) serwerem. Dzieje się tak, ponieważ nie ma sposobu, aby upewnić się, że serwer jest faktycznie zaufanym serwerem. Przez to tracimy możliwość sprawdzenia bezpieczeństwa połączenia.

  > Przy niepełnym łańcuchu połączenia są nadal bezpieczne, ponieważ **ruch nadal jest szyfrowany**. Aby rozwiązać ten problem, należy ręcznie rozwiązać niepełny łańcucha certyfikatów, łącząc wszystkie certyfikaty od certyfikatu serwera z zaufanym certyfikatem głównym (wyłącznie, w tej kolejności).

Istnieje kilka możliwości przerwania łańcucha zaufania, w tym między innymi:

- każdy certyfikat w łańcuchu jest samopodpisany, chyba że jest to rootCA
- nie każdy certyfikat pośredni jest sprawdzany, począwszy od oryginalnego certyfikatu aż do certyfikatu głównego
- pośredni certyfikat podpisany przez urząd certyfikacji nie ma oczekiwanych podstawowych ograniczeń ani innych ważnych rozszerzeń
- certyfikat główny został przejęty lub autoryzowany dla niewłaściwej strony

W większości przypadków sam certyfikat serwera jest niewystarczający; do zbudowania pełnego łańcucha zaufania potrzebne są dwa lub więcej certyfikaty. Typowy problem z konfiguracją występuje podczas wdrażania serwera z ważnym certyfikatem, ale bez wszystkich niezbędnych certyfikatów pośrednich. Aby uniknąć tej sytuacji, wystarczy użyć wszystkich certyfikatów dostarczonych przez urząd certyfikacji w tej samej kolejności lub zbudować łańcuch samodzielnie, pobierając wszystkie niezbędne certyfikaty pośrednie.

Niepoprawny łańcuch certyfikatów skutecznie unieważnia certyfikat serwera i powoduje wyświetlanie ostrzeżeń w przeglądarce. W praktyce problem ten jest czasami trudny do zdiagnozowania, ponieważ niektóre przeglądarki mogą odtwarzać niekompletne łańcuchy, a niektóre nie. Wszystkie przeglądarki mają tendencję do buforowania i ponownego wykorzystywania certyfikatów pośrednich.

# Testowanie łańcucha certyfikatów

Aby przetestować poprawność łańcucha certyfikatów, użyj jednego z następujących narzędzi:

- [SSL Checker by sslshopper](https://www.sslshopper.com/ssl-checker.html)
- [SSL Checker by namecheap](https://decoder.link/sslchecker/)
- [SSL Server Test by Qualys](https://www.ssllabs.com/ssltest/analyze.html)

Aby uzyskać więcej informacji, przeczytaj dokument [What is the SSL Certificate Chain?](https://support.dnsimple.com/articles/what-is-ssl-certificate-chain/) oraz [Get your certificate chain right](https://medium.com/@superseb/get-your-certificate-chain-right-4b117a9c0fce).
