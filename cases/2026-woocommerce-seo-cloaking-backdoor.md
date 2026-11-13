# SEO cloaking backdoor in a WooCommerce shop

Anonymized incident report. Work done by [AGuard](https://aguard.lt) in autumn 2026.

## Summary

A small EU online shop on WordPress and WooCommerce noticed strange English product pages under its domain in Google results. Visitors and the site owner saw a normal shop.

The cause was a two-file implant in WordPress core. It answered before WordPress produced any output, served foreign store pages to anything that looked like Googlebot, and forwarded request data of ordinary visitors to a remote server. All plugins, the theme and core were up to date, and the plugin layer was clean, which is why earlier checks by other parties found nothing.

The attacker got in through a legitimate administrator account and a trojanized copy of a popular plugin. Cloaking was removed on the first day of cleanup. The full entry path could be reconstructed only partly, because most server logs and backups had already rotated out.

| | |
|---|---|
| Platform | WordPress 7.1, WooCommerce, about 50 plugins, shared hosting with DirectAdmin, Cloudflare in front |
| Time from compromise to detection | 8 days |
| Time from access to removal of cloaking | 1 day |
| Spam URLs injected into search | 1,158 in the attacker's sitemap, index grew to about 42,000 pages |
| Confirmed data loss | None proven. The implant had the capability, see below |

## How it was detected

The first check was done from outside, with no access to the site.

- A request with a Googlebot user agent returned a foreign product page on any URL, including URLs that should return 404.
- The same request with a browser or Bingbot user agent returned the normal shop.
- A separate sitemap listed 1,158 spam URLs.

Then the same test was repeated directly against the origin server, bypassing the CDN. The cloaked answer came from the origin, so the CDN was ruled out. The cloaked response also lacked the headers that the caching plugin adds to every page and used a different charset spelling. That meant the response was produced before WordPress and its plugins ran. The list of candidates shrank to a few early-loaded files: the cache drop-in, must-use plugins, `index.php`, `.htaccess` and core loaders.

## What was found

| Item | Detail |
|---|---|
| Modified core file | `wp-settings.php` with one added `require` line |
| Implant | `wp-includes/class-wp-term-details.php`, about 35 KB, a name that imitates a core class. This file does not exist in WordPress |
| Marker | An empty `products` directory in the web root, created about 90 minutes before the implant |
| Everything else | Core checksums, all plugins against official checksums, database options, cron, users and code snippets were clean |

Behaviour of the implant:

- **Cloaking.** Any user agent containing "Google" received pages fetched from a remote server. Bing and Yandex were not targeted.
- **Data forwarding.** For non-bot visitors the implant sent server variables and POST bodies to the same remote server. On a shop this covers cookies, login forms and checkout forms. The channel was plain HTTP.
- **Persistence.** It lived in core, so plugin and theme updates did not remove it. A core reinstall did.

Whether data was actually collected and used could not be established. The report to the client states this as a capability, not as a proven leak.

## Timeline

Day 0 is the day the implant was written.

| Day | Event | Source |
|---|---|---|
| D-2 to D+1 | Password guessing against an administrator login name, 36 to 108 attempts a day | Login limiter table |
| D-1, night | Successful login to an administrator account of an external contractor from a residential address in South America | Activity log |
| D-1, midday | Login to the same account from a cloud hosting address. The login request went straight to the login endpoint without loading the login page. A plugin archive was uploaded and installed but never activated | Activity log |
| D-1, 3 seconds later | A request to a PHP file inside that plugin's directory. The official release of the plugin does not contain this file | PHP error log |
| D0, morning | A second stray script, `import.php`, runs in the web root | PHP error log |
| D0 | `wp-settings.php` modified, implant file written. No WordPress activity by any administrator at that moment | File change times, activity log |
| D+7 | The activity log plugin and several other plugins are removed from the contractor account | Activity log tables left in the database |
| D+8 | The owner asks for an independent investigation. Cloaking confirmed from outside the same day | |
| D+11 | Access granted. Implant located within hours | |
| D+12 | Cleanup. Cloaking gone | |
| D+13 | Hardening and removal of abandoned neighbouring sites | |
| D+19 | Entry path reconstructed as far as the evidence allows | |

The most useful sources were not the obvious ones:

- **Database tables of a removed plugin.** The activity log plugin had been uninstalled, but its tables stayed in the database with a complete, gapless record.
- **PHP error log timezone.** Entries written by WordPress carried UTC. Entries written by scripts that ran outside WordPress carried the server's local timezone. Filtering by timezone separated foreign scripts from normal site errors.
- **Official plugin archive.** Comparing the file list of the official release with paths in the error log showed that the uploaded archive contained an extra file.

## Entry path and limits of the conclusion

The most probable chain: a contractor's administrator password was obtained elsewhere, the attacker logged in, uploaded a trojanized plugin archive, used the extra file inside it as a dropper, and then wrote the core implant outside WordPress.

What the evidence does not support:

- **Who was behind the logins.** An account being used is not proof of who used it. The report says the account was used by a third party and stops there.
- **How the password was obtained.** No link to a specific breach could be shown.
- **The dropper itself.** The uploaded archive and both stray scripts were gone. The hoster kept web logs for 7 days and file backups for a similar period, the FTP log showed no use in that month, and the site's own backups were failing silently because the destination storage was full.

Being explicit about these limits mattered later, when the client asked for a version of the report for law enforcement.

## Cleanup

Evidence was preserved before any change: copies of the infected files, web logs and a database dump, each with a SHA-256 checksum and an action log with UTC timestamps.

1. Reinstalled WordPress core from the dashboard, which removed the added line.
2. Moved the implant and the marker directory to a quarantine folder outside the web root.
3. Ended all administrator sessions, reset administrator passwords, rotated salts and the database password.
4. Disabled file editing in the dashboard, blocked `xmlrpc.php` and direct access to log and backup files.
5. Moved a 297 MB publicly readable PHP error log out of the web root.
6. Returned 410 for the spam URL pattern and requested recrawl in Search Console.

## Hardening

- **Neighbours.** The hosting account held two abandoned copies of an older shop and an outdated site, about 12 GB, one of them still infected from an incident a year earlier. One orphaned database allowed open registration with the administrator role. All were removed after backup. The main site's `open_basedir` was narrowed.
- **Access.** Two-factor login by email code, a renamed login path, a challenge rule on the login page at the CDN, removal of unused administrator accounts.
- **Transport and mail.** HSTS and basic security headers at the CDN, minimum TLS 1.2, CAA record, DKIM for shop mail, DMARC reporting.
- **Backups.** Off-site backups moved to dedicated storage with a daily file schedule, and a check that they actually complete.
- **Monitoring.** File integrity checks against official checksums, a scheduled self-test that requests the site as Googlebot and compares the answer, alerts on new administrators and on logins from new addresses.

## Indicators of compromise

| Type | Value |
|---|---|
| File | `wp-includes/class-wp-term-details.php` |
| Core change | An extra `require` in `wp-settings.php` pointing to the file above |
| Directory | Empty `products` directory in the web root |
| Stray scripts | `import.php` in the web root. `recommended-modules/ads-toggle/file-response.php` inside the `wp-duplicate-page` plugin directory |
| Remote server | A subdomain of a recently registered domain on US hosting, path ending in `remote_api.php`, plain HTTP. Exact host and file checksum are withheld and available to defenders on request |
| Behaviour | Different page for user agents containing "Google". A sitemap with product URLs that do not belong to the site |

Quick self-check from a terminal:

```
curl -s -A "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)" https://example.com/ | head -c 600
curl -s -A "Mozilla/5.0" https://example.com/ | head -c 600
```

If the two answers show different sites, look at core loaders first, not at plugins.

## Lessons

1. **Up to date does not mean clean.** Everything was current and the implant survived every update. Verify core files against official checksums.
2. **Test as the search engine sees the site.** The owner, the developer and a marketing agency all looked at the site as humans and saw nothing.
3. **Check where the answer is produced.** Missing cache headers showed that the response was generated before WordPress, which cut the search from 50 plugins to a handful of files.
4. **Logs disappear in a week.** On shared hosting, ask the hoster to preserve logs and backups on the first day, before anything else.
5. **A backup that is not checked is not a backup.** The off-site file copies had been failing without anyone noticing.
6. **Old copies on the same account are part of the attack surface.** They are rarely patched and share the same file permissions.
7. **Write down what cannot be proven.** It keeps the report usable for lawyers and police, and it is fair to the people whose accounts were abused.

## Contact

Questions about this case or a similar problem on your site: [aguard.lt](https://aguard.lt)
