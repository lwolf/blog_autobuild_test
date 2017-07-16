+++
author = "Sergey Nuzhdin"
categories = ["letsencrypt", "python", "ssl"]
date = "2016-06-14"
description = "Debugging lets encrypt renewal script after backup"
featured = "letsencrypt.jpg"
featuredalt = ""
featuredpath = "date"
linktitle = ""
title = "Debugging lets encrypt renewal script after backup"

+++

I started to use letsencrypt everywhere ... bla-bla-bla...

After migrating one of configurations from one machine to another, I was unable to renew domain.
I was getting a weird error and adding `--debug` did not make it more helpful.

```
root@router-vm:/home/lwolf# certbot-auto renew --no-self-upgrade --debug

Processing /etc/letsencrypt/renewal/domain1.conf
Processing /etc/letsencrypt/renewal/domain2.conf
-------------------------------------------------------------------------------
2016-06-13 13:15:31,123:WARNING:certbot.renewal:Renewal configuration file /etc/letsencrypt/renewal/domain2.conf is broken. Skipping.
-------------------------------------------------------------------------------
Processing /etc/letsencrypt/renewal/domain3.conf

The following certs are not due for renewal yet:
  /etc/letsencrypt/live/domain1/fullchain.pem (skipped)
  /etc/letsencrypt/live/domain3/fullchain.pem (skipped)
No renewals were attempted.

Additionally, the following renewal configuration files were invalid:
  /etc/letsencrypt/renewal/domain2.conf (parsefail)
Traceback (most recent call last):
  File "/root/.local/share/letsencrypt/bin/letsencrypt", line 11, in <module>
    sys.exit(main())
  File "/root/.local/share/letsencrypt/local/lib/python2.7/site-packages/certbot/main.py", line 735, in main
    return config.func(config, plugins)
  File "/root/.local/share/letsencrypt/local/lib/python2.7/site-packages/certbot/main.py", line 583, in renew
    renewal.renew_all_lineages(config)
  File "/root/.local/share/letsencrypt/local/lib/python2.7/site-packages/certbot/renewal.py", line 363, in renew_all_lineages
    len(renew_failures), len(parse_failures)))
Error: 0 renew failure(s), 1 parse failure(s)

```
Error says that I have broken config. I compared it with new working one, and it was identical except for the path.
I spent some time googling, and almost agreed to recreate certificates for this domain from scratch, as was suggested in some github issue.

But before recreating I decided to try to debug what caused the actual problem. It was not hard since certbot is written in Python.

I was very surprised when during debug I found out that the actual error was because of missing/broken symlinks in letsencrypt `live` folder.
My problem was in wrong names of symlinks for certificates.
When I moved my configs and recreated symlinks, I didn't notice that symlink name should be without `1` e.g `fullchain.pem` instead of `fullchain1.pem`.

