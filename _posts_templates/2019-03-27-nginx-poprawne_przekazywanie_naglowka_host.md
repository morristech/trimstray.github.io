---
layout: post
title: 'NGINX: Poprawne przekazywanie nagłówka Host'
date: 2019-03-27 00:36:49
categories: [publications]
tags: [PL, http, nginx, best-practices, headers]
comments: false
favorite: false
seo:
  date_modified: 2020-02-23 09:14:19 +0100
---

Nagłówek `Host` jest jednym z najważniejszych nagłówków w komunikacji HTTP. Informuje on serwer, którego wirtualnego hosta ma użyć (pod jaki adres chcemy wysłać zapytanie) oraz określa, która witryna internetowa lub aplikacja internetowa powinna przetwarzać przychodzące żądanie HTTP.

W tym wpisie omówię za pomocą jakich zmiennych możemy zarządzać tym nagłówkiem oraz w jaki sposób przekazać poprawną wartość do warstwy backendu.

# Nagłówek Host i zmienne

W NGINX zmienna `$host` jest równa zmiennej `$http_host`, która zapisuje wartość tego nagłówka małymi literami i bez numeru portu (jeśli jest obecny), z wyjątkiem sytuacji, gdy `HTTP_HOST` jest nieobecny lub jest pustą wartością. W takim przypadku `$host` jest równy wartości dyrektywy `server_name`, czyli serwera, który przetworzył żądanie.

Jednak spójrz na to wyjaśnienie:

  > _An unchanged `Host` request header field can be passed with `$http_host`. However, if this field is not present in a client request header then nothing will be passed. In such a case it is better to use the `$host` variable - its value equals the server name in the `Host` request header field or the primary server name if this field is not present._

Wynika z tego, że jeśli ustawimy nagłówek `Host` na wartość `Host: MASTER:8080`, zmienna `$host` będzie przechowywać wartość `master`, podczas gdy `$http_host` będzie równy `MASTER:8080` (w taki sposób odzwierciedla cały nagłówek).

Zawsze dobrze jest zmodyfikować nagłówek hosta, aby upewnić się, że rozdzielczość hosta wirtualnego na dalszym serwerze działa tak, jak powinna.
