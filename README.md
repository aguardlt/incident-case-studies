# WordPress incident case studies

Anonymized write-ups of real incident response work on WordPress and WooCommerce sites, by [AGuard](https://aguard.lt), a small website security practice in Lithuania.

Each case covers how the compromise was detected, how the timeline was reconstructed from whatever evidence survived, what was done to clean up and harden, and what could not be proven. Client names, domains, people and vendors are removed. Dates are given relative to the day of compromise. Behavioural indicators and file names are published so that others can check their own sites.

No malware source code is published here.

## Cases

| Case | Platform | Summary |
|---|---|---|
| [SEO cloaking backdoor in a WooCommerce shop](cases/2026-woocommerce-seo-cloaking-backdoor.md) | WordPress, WooCommerce, shared hosting | A core-level implant served spam store pages to Googlebot only and forwarded visitor request data to a remote server. Entry through a stolen administrator account and a trojanized plugin archive. |

## About

Aleksandras Izmalkovas, [aguard.lt](https://aguard.lt). Monitoring, incident investigation and cleanup for websites of small businesses and public institutions.

Text is licensed under CC BY 4.0.
