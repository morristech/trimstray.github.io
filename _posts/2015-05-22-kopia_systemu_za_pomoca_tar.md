---
layout: post
title: 'Kopia systemu za pomocą narzędzia TAR'
date: 2015-05-22 12:05:09
categories: [publications]
tags: [PL, backup, system, tar, pigz]
comments: false
favorite: false
seo:
  date_modified: 2020-02-22 09:47:41 +0100
---

Wykonanie kopii systemu za pomocą narzędzia `tar` w parze z `gzip/pigz` ma kilka zalet, a jedną z nich jest niewątpliwie prostota, ponieważ nie potrzebujemy żadnych zewnętrznych narzędzi do tego celu.

Sposób na wykonanie kopii całego systemu z wykorzystaniem narzędzia `tar`, włączając w to kompresję `gzip`:

```bash
tar czvpf /mnt/system-$(date +%d%m%Y%s).tgz --directory=/ --exclude=proc \
--exclude=sys --exclude=dev --exclude=mnt --exclude=tmp.
```

Jeżeli serwer posiada więcej niż jeden rdzeń, zamiast kompresji programem `gzip`, można użyć polecenia `pigz`, które działa wielowątkowo (pamiętajmy o usunięciu opcji `-z`), dzięki czemu znacznie przyspiesza cały proces tworzenia kopii:

```bash
tar cvpf /backup/snapshot-$(date +%d%m%Y%s).tgz --directory=/mnt/system \
--exclude=proc/* --exclude=sys/* --exclude=dev/* \
--exclude=mnt/* --exclude=tmp/* --use-compress-program=pigz .
```
