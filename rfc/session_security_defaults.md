====== PHP RFC: Secure Session Configuration Defaults ======
  * Date: 2026-03-28
  * Target version: PHP 8.6
  * Author: Jorg Sowa
  * Status: Draft
  * First Published at: http://wiki.php.net/rfc/session_security_defaults

===== Introduction =====

Three ini settings in PHP's session extension have security implications but ship with defaults that leave applications unnecessarily exposed. This RFC proposes changing those defaults so that new PHP installations are secure out of the box, without requiring application code changes for the common case.

The affected settings are:

  - ''session.use_strict_mode'' — reject unrecognised session IDs supplied by the client
  - ''session.cookie_httponly'' — exclude the session cookie from the JavaScript cookie API
  - ''session.cookie_samesite'' — restrict cross-site delivery of the session cookie

Each change is a single-line edit to ''ext/session/session.c''. No new ini entries, functions, or constants are introduced.

===== Background =====

==== Session Fixation (use_strict_mode) ====

Session fixation allows an attacker to pre-plant a known session ID and wait for a victim to authenticate against it. With ''use_strict_mode=0'' (the current default), ''session_start()'' accepts any well-formed ID supplied in a cookie, ''$_GET'', or ''$_POST'' and begins a session under that ID without verifying that the ID already exists in storage.

With ''use_strict_mode=1'', ''php_session_initialize()'' calls ''s_validate_sid()'' on the proposed ID before accepting it. The built-in files handler checks whether a session file for that ID exists on disk; because an attacker-planted ID has no corresponding file, a fresh random ID is generated instead.

''use_strict_mode'' was added in PHP 5.5.2. The [[https://www.php.net/manual/en/session.security.ini.php|PHP manual]] has recommended enabling it since PHP 7.1.

==== JavaScript Cookie Access (cookie_httponly) ====

The ''[[https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie#httponly|HttpOnly]]'' cookie attribute, defined in [[https://www.rfc-editor.org/rfc/rfc6265#section-4.1.2.6|RFC 6265 §4.1.2.6]] and supported by all browsers since IE 6, instructs the browser to exclude the cookie from the ''document.cookie'' API. A script running in the page — whether first-party or injected by an XSS attack — cannot read or exfiltrate an HttpOnly cookie.

PHP has supported ''session.cookie_httponly'' since PHP 5.2.0 (2006) and the flag has been universally recommended by security guidance ([[https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html|OWASP]], [[https://infosec.mozilla.org/guidelines/web_security.html|Mozilla]], [[https://pages.nist.gov/800-63-4/sp800-63b/session/|NIST]]) ever since. There is no legitimate reason to default it to off.

==== CSRF via Cross-Site Requests (cookie_samesite) ====

The ''[[https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie#samesitesamesite-value|SameSite]]'' attribute controls whether the browser attaches a cookie to cross-site requests. Without it, a forged form or link on an attacker-controlled site can trigger authenticated requests using the victim's session.

PHP's ''session.cookie_samesite'' defaults to an empty string, which emits ''Set-Cookie'' with no ''SameSite'' attribute. Browsers have progressively tightened their handling of such cookies: Chrome applied ''SameSite=Lax'' as the default starting with [[https://www.chromium.org/updates/same-site/|Chrome 80 (February 2020)]], and [[https://groups.google.com/g/mozilla.dev.platform/c/nx2uP0CzA9k|Firefox followed with version 103 (August 2022)]]. Setting the attribute explicitly in PHP aligns the ''Set-Cookie'' header with browser behaviour and removes reliance on per-browser inference.

===== Comparison with Other Platforms =====

Python, Node.js, and Go do not provide built-in session handling at the standard-library level; their defaults are determined entirely by third-party libraries. Java, .NET, and Ruby each provide session handling at the platform or interface layer and offer a direct comparison:

^ Platform                  ^ Session cookie          ^ HttpOnly default ^ SameSite default ^
| PHP                       | ''PHPSESSID''           | ''0'' (off)      | not set          |
| Java (Tomcat 10)          | ''JSESSIONID''          | ''true''         | not set          |
| ASP.NET Core (3.1+)       | ''.AspNetCore.Session'' | ''true''         | ''Lax''          |
| Ruby (Rack)               | ''rack.session''        | ''true''         | not set          |

The Servlet 3.0 specification (December 2009) introduced the ''SessionCookieConfig'' API but left the HttpOnly default implementation-defined. Tomcat, the dominant reference container, chose ''useHttpOnly=true'' as its default. ASP.NET Core added stable ''SameSite=Lax'' defaults for session and authentication cookies in version 3.1 (December 2019), following earlier attempts in 2.0 that were partially rolled back due to browser compatibility issues. Rack, the universal Ruby web interface layer, has defaulted to ''httponly: true'' since the option was introduced but does not set a SameSite attribute by default.

PHP is the only platform in this group where ''HttpOnly'' is off by default. For SameSite, PHP and Rack are both without a default, while ASP.NET Core explicitly sets ''Lax''.

It is also worth noting that the two most widely used PHP frameworks — Laravel and Symfony — both override these PHP-level defaults and ship with ''HttpOnly=true'' and ''SameSite=lax''. This indicates broad consensus within the PHP ecosystem that the built-in defaults are insufficient, and that application developers are currently required to compensate for them.

===== Proposal =====

Change the following three default values in ''ext/session/session.c'':

^ INI setting                 ^ Old default ^ New default ^
| ''session.use_strict_mode'' | ''0''       | ''1''       |
| ''session.cookie_httponly'' | ''0''       | ''1''       |
| ''session.cookie_samesite'' | ''''        | ''Lax''     |

===== Backward Incompatible Changes =====

==== session.use_strict_mode = 1 ====

Applications that deliberately supply an externally controlled session ID — for example, a shared-secret hand-off between subdomains using the files save handler — will have the ID rejected because no matching session file exists at the time of the first request. The correct approach is to write and close the session on the originating side (''session_write_close()'') before presenting the ID to the receiving side.

Applications using a custom session handler whose ''validateId()'' method returns ''true'' unconditionally are unaffected. This includes common backends such as Redis and Memcached when using the default ''php-memcached'' or ''phpredis'' session handlers, which do not implement ''validateId()'' and therefore fall back to returning ''true''.

==== session.cookie_httponly = 1 ====

The ''HttpOnly'' flag is enforced by the browser. It prevents the session cookie value from being read via ''document.cookie''; it has no effect on whether the browser sends the cookie with requests.

Applications that read the session cookie value from ''document.cookie'' in JavaScript (for example, to embed the ID in a custom request header) will no longer be able to do so. The session ID is a server-side credential and should not be consumed by client-side code. Applications that need a client-accessible token for request correlation should use a separate, explicitly non-HttpOnly CSRF token rather than the session cookie itself.

==== session.cookie_samesite = Lax ====

With ''SameSite=Lax'', the browser sends the session cookie on same-site requests and on top-level cross-site navigations using safe HTTP methods (GET, HEAD). It does not send the cookie on cross-site sub-resource requests or cross-site POST requests.

Applications that rely on cross-site POST carrying the session cookie — for example, SP-initiated SAML SSO flows or legacy cross-origin form submissions — must either set ''SameSite=None; Secure'' explicitly for those endpoints or migrate to a token-based flow.

As noted above, Chrome and Firefox already apply ''Lax'' as the implicit default for cookies sent without a ''SameSite'' attribute. Applications running on those browsers are already subject to this behaviour; the change makes it explicit and consistent across all browsers and PHP versions.

===== Proposed PHP Versions =====

PHP 8.6.

===== Unaffected Functionality =====

  * ''session.cookie_secure'' remains ''0''. Defaulting to ''1'' would prevent session cookies from being sent over HTTP, breaking local development environments. This should be set to ''1'' in production php.ini configuration.
  * ''session.use_only_cookies'' remains ''1'' (already the secure default since PHP 5.3.0).
  * No changes to session serialisation, save handlers, or the session module API.

===== Vote =====

As per the [[https://wiki.php.net/rfc/voting|voting RFC]], a Yes/No vote with a 2/3 majority is required for each item. Each change is voted on separately so that the internals list can accept a subset.

  - Change ''session.use_strict_mode'' default to ''1'': Yes / No
  - Change ''session.cookie_httponly'' default to ''1'': Yes / No
  - Change ''session.cookie_samesite'' default to ''Lax'': Yes / No

===== Implementation =====

Patch to the ''PHP_INI_BEGIN()'' block in ''ext/session/session.c''. No PR exists yet.

===== References =====

  * [[https://www.php.net/manual/en/session.security.ini.php|PHP Manual: Session security-related ini settings]]
  * [[https://www.rfc-editor.org/rfc/rfc6265#section-4.1.2.6|RFC 6265 §4.1.2.6: HttpOnly attribute]]
  * [[https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie#httponly|MDN: HttpOnly attribute]]
  * [[https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie#samesitesamesite-value|MDN: SameSite attribute]]
  * [[https://caniuse.com/same-site-cookie-attribute|Can I Use: SameSite]]
  * [[https://www.chromium.org/updates/same-site/|Chromium: SameSite cookie changes]]
  * [[https://groups.google.com/g/mozilla.dev.platform/c/nx2uP0CzA9k|Firefox: SameSite=Lax by default]]
  * [[https://web.dev/articles/samesite-cookies-explained|web.dev: SameSite cookies explained]]
