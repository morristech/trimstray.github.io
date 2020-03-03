---
layout: post
title: 'Kopia systemu za pomocą TAR + GZIP lub PIGZ'
date: 2015-05-22 12:05:09
categories: [publications]
tags: [PL, backup, system, tar, pigz]
comments: false
favorite: false
seo:
  date_modified: 2020-02-22 09:47:41 +0100
---

Sposób na wykonanie kopii całego systemu z wykorzystaniem narzędzi `tar` oraz `gzip`:

```bash
tar czvpf /mnt/system-$(date +%d%m%Y%s).tgz --directory=/ --exclude=proc \
--exclude=sys --exclude=dev --exclude=mnt --exclude=tmp.
```

Jeżeli maszyna posiada więcej niż jeden rdzeń, zamiast kompresji programem `gzip`, można użyć polecenia `pigz`, które działa wielowątkowo (pamiętajmy o usunięciu opcji `-z`), dzięki czemu znacznie przyspiesza cały proces:

```bash
tar cvpf /backup/snapshot-$(date +%d%m%Y%s).tgz --directory=/mnt/system \
--exclude=proc/* --exclude=sys/* --exclude=dev/* \
--exclude=mnt/* --exclude=tmp/* --use-compress-program=pigz .
```
