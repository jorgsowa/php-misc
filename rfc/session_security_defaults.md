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

==== JavaScript Cookie Access (cookie_httponly) ====

The ''[[https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie#httponly|HttpOnly]]'' cookie attribute, defined in [[https://www.rfc-editor.org/rfc/rfc6265#section-4.1.2.6|RFC 6265 §4.1.2.6]] and supported by all browsers since IE 6, instructs the browser to exclude the cookie from the ''document.cookie'' API. A script running in the page — whether first-party or injected by an XSS attack — cannot read or exfiltrate an HttpOnly cookie.

PHP has supported ''session.cookie_httponly'' since PHP 5.2.0 (2006) and the flag has been universally recommended by security guidance ([[https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html#httponly-attribute|OWASP]], [[https://infosec.mozilla.org/guidelines/web_security.html#cookies|Mozilla]], [[https://pages.nist.gov/800-63-4/sp800-63b/session/#sesscookies|NIST]]) ever since. There is no legitimate reason to default it to off.

==== CSRF via Cross-Site Requests (cookie_samesite) ====

The ''[[https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie#samesitesamesite-value|SameSite]]'' attribute controls whether the browser attaches a cookie to cross-site requests. Without it, a forged form or link on an attacker-controlled site can trigger authenticated requests using the victim's session.

PHP's ''session.cookie_samesite'' defaults to an empty string, which emits ''Set-Cookie'' with no ''SameSite'' attribute. Browsers have progressively tightened their handling of such cookies: Chrome applied ''SameSite=Lax'' as the default starting with [[https://www.chromium.org/updates/same-site/|Chrome 80 (February 2020)]], and [[https://groups.google.com/g/mozilla.dev.platform/c/nx2uP0CzA9k|Firefox followed with version 103 (August 2022)]]. Setting the attribute explicitly in PHP aligns the ''Set-Cookie'' header with browser behaviour and removes reliance on per-browser inference.

Setting ''SameSite=Lax'' on session cookies is recommended by ([[https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html#samesite-cookie-attribute|OWASP]], [[https://infosec.mozilla.org/guidelines/web_security.html#cookies|Mozilla]], [[https://pages.nist.gov/800-63-4/sp800-63b/session/#sesscookies|NIST]]).

===== Comparison with Other Platforms =====

^ Platform                                                                                                    ^ HttpOnly default                                                                                                                                                       ^ SameSite default                                                                                                                    ^
| [[https://www.php.net/manual/en/session.security.ini.php|PHP]]                                             | [[https://www.php.net/manual/en/session.configuration.php#ini.session.cookie-httponly|''0'' (off)]]                                                                    | [[https://www.php.net/manual/en/session.configuration.php#ini.session.cookie-samesite|not set]]                                     |
| [[https://tomcat.apache.org/tomcat-11.0-doc/config/context.html|Java (Tomcat 11)]]                         | [[https://tomcat.apache.org/tomcat-11.0-doc/config/context.html#Common_Attributes|''true'']]                                                                           | not set                                                                                                                             |
| [[https://learn.microsoft.com/en-us/aspnet/core/fundamentals/app-state?view=aspnetcore-10.0|ASP.NET Core]] | [[https://learn.microsoft.com/en-us/aspnet/core/fundamentals/app-state?view=aspnetcore-10.0#configure-session-options|''true'']]                                       | [[https://learn.microsoft.com/en-us/aspnet/core/fundamentals/app-state?view=aspnetcore-10.0#configure-session-options|''Lax'']]     |
| [[https://guides.rubyonrails.org/security.html#sessions|Ruby (Rails)]]                                      | [[https://github.com/rails/rails/blob/main/actionpack/lib/action_dispatch/middleware/cookies.rb#L167|''false'' (off)]]                                                 | [[https://github.com/rails/rails/blob/main/actionpack/lib/action_dispatch/middleware/cookies.rb#L254|''Lax'']]          |
| [[https://docs.djangoproject.com/en/stable/topics/http/sessions/|Python (Django)]]                         | [[https://docs.djangoproject.com/en/stable/ref/settings/#session-cookie-httponly|''true'']]                                                                            | [[https://docs.djangoproject.com/en/stable/ref/settings/#session-cookie-samesite|''Lax'']]                                          |
| [[https://github.com/expressjs/session|Node.js (express-session)]]                                          | [[https://github.com/expressjs/session/blob/408229ea3373097732875315d6f63c45e39fd3b6/session/cookie.js#L28|''true'']]                                                  | not set                                                                                                                             |
| [[https://pkg.go.dev/github.com/gin-contrib/sessions|Go (gin-contrib/sessions)]]                           | [[https://github.com/gorilla/sessions/blob/main/store.go#L49-L53|''false'' (off)]]                                                                  | not set                                                                                                                             |
| [[https://laravel.com/docs/session|PHP (Laravel)]]                                                          | [[https://github.com/laravel/laravel/blob/master/config/session.php#L172|''true'']]                                                                                    | [[https://github.com/laravel/laravel/blob/master/config/session.php#L187|''Lax'']]                                                  |
| [[https://symfony.com/doc/current/session.html|PHP (Symfony)]]                                              | [[https://github.com/symfony/symfony/blob/master/src/Symfony/Component/HttpKernel/EventListener/AbstractSessionListener.php#L143|''true'']]                            | [[https://github.com/symfony/symfony/blob/master/src/Symfony/Component/HttpKernel/EventListener/AbstractSessionListener.php#L144|''Lax'']]  |

===== Proposal =====

Change the following three default values in ''ext/session/session.c'':

^ INI setting                 ^ Old default ^ New default ^
| ''session.use_strict_mode'' | ''0''       | ''1''       |
| ''session.cookie_httponly'' | ''0''       | ''1''       |
| ''session.cookie_samesite'' | ''''        | ''Lax''     |

===== Backward Incompatible Changes =====

==== session.use_strict_mode = 1 ====

Applications that deliberately supply an externally controlled session ID — for example, a shared-secret hand-off between subdomains using the files save handler — will have the ID rejected because no matching session file exists at the time of the first request. The correct approach is to write and close the session on the originating side (''[[https://www.php.net/manual/en/function.session-write-close.php|session_write_close()]]'') before presenting the ID to the receiving side.

Applications using a custom save handler that implements [[https://www.php.net/manual/en/sessionupdatetimestamphandlerinterface.validateid.php|''SessionUpdateTimestampHandlerInterface::validateId()'']] are governed by that method's return value. Built-in extension handlers such as [[https://github.com/phpredis/phpredis/blob/develop/redis_session.c|phpredis]] and [[https://github.com/php-memcached-dev/php-memcached/blob/master/php_memcached_session.c|php-memcached]] implement the equivalent C-level ''PS_VALIDATE_SID_FUNC'' hook and participate fully in strict mode validation. Userland handlers registered via ''session_set_save_handler(object)'' that do not implement [[https://www.php.net/manual/en/class.sessionupdatetimestamphandlerinterface.php|''SessionUpdateTimestampHandlerInterface'']] fall back to accepting any ID and are therefore unaffected by this change.

==== session.cookie_httponly = 1 ====

The ''HttpOnly'' flag is enforced by the browser. It prevents the session cookie value from being read via ''document.cookie''; it has no effect on whether the browser sends the cookie with requests.

Applications that read the session cookie value from ''document.cookie'' in JavaScript (for example, to embed the ID in a custom request header) will no longer be able to do so. The session ID is a server-side credential and should not be consumed by client-side code. Applications that need a client-accessible token for request correlation should use a separate, explicitly non-HttpOnly CSRF token rather than the session cookie itself.

==== session.cookie_samesite = Lax ====

With ''SameSite=Lax'', the browser sends the session cookie on same-site requests and on top-level cross-site navigations using safe HTTP methods (GET, HEAD). It does not send the cookie on cross-site sub-resource requests or cross-site POST requests.

Applications that rely on cross-site POST carrying the session cookie are affected. Two common cases:

  * **[[https://docs.oasis-open.org/security/saml/v2.0/saml-profiles-2.0-os.pdf|SP-initiated SAML SSO]]** (section 4.1 of the SAML 2.0 Profiles standard) — the identity provider returns a ''SAMLResponse'' as an auto-submitted HTML form POSTed to the service provider's Assertion Consumer Service URL, a cross-site POST. The session cookie will not be sent on that request. [[https://learn.microsoft.com/en-us/entra/identity-platform/howto-handle-samesite-cookie-changes-chrome-browser|Microsoft]], [[https://support.okta.com/help/s/article/FAQ-How-Chrome-80-Update-for-SameSite-by-default-Potentially-Impacts-Your-Okta-Environment|Okta]], and the [[https://shibboleth.atlassian.net/wiki/spaces/DEV/pages/1181253974/IdP+SameSite+Testing|Shibboleth project]] all recommend setting ''SameSite=None; Secure'' on the SP-side state and correlation cookies for the duration of the SSO flow, or switching to the HTTP Redirect binding (which is not subject to SameSite restrictions).
  * **[[https://openid.net/specs/oauth-v2-form-post-response-mode-1_0.html#FormPostResponseMode|OAuth 2.0 form_post response mode]]** — the authorisation server POSTs response parameters back to the client's redirect URI. Applications that verify state against the session on that POST will lose the session cookie. Use ''response_mode=query'' or handle state via a separate cookie.

Applications switching affected endpoints to ''SameSite=None'' must also set the ''[[https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie#none|Secure]]'' attribute; browsers silently ignore ''SameSite=None'' without ''Secure''.

As noted above, Chrome and Firefox already apply ''Lax'' as the implicit default for cookies sent without a ''SameSite'' attribute. Applications running on those browsers are already subject to this behaviour; the change makes it explicit and consistent across all browsers and PHP versions.

==== Browser support ====

The ''SameSite'' cookie attribute is supported across all modern browsers:

^ Browser   ^ SameSite support since ^ Lax-by-default since ^ Reference ^
| Chrome    | 51 (May 2016)          | [[https://www.chromium.org/updates/same-site/|80 (February 2020)]] | [[https://developer.chrome.com/blog/samesite-cookies-explained|Chrome blog]] |
| Firefox   | 60 (May 2018)          | [[https://groups.google.com/g/mozilla.dev.platform/c/nx2uP0CzA9k|103 (August 2022)]] | [[https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie#samesitesamesite-value|MDN]] |
| Safari    | 12.1 (March 2019)      | not applied            | [[https://webkit.org/blog/10218/full-third-party-cookie-blocking-and-more/|WebKit blog]] |
| Edge      | 18 (October 2018)      | [[https://learn.microsoft.com/en-us/microsoft-edge/web-platform/site-impacting-changes|85 (August 2020)]] | [[https://learn.microsoft.com/en-us/microsoft-edge/web-platform/site-impacting-changes|Edge docs]] |

Safari supports the attribute and respects an explicit ''SameSite=Lax'' value but does not apply Lax as an implicit default for cookies lacking the attribute. Setting the attribute explicitly in PHP therefore closes the cross-browser gap and removes reliance on per-browser inference.

===== Proposed PHP Versions =====

PHP 8.6.

===== Vote =====

As per the [[https://wiki.php.net/rfc/voting|voting RFC]], each change requires a 2/3 majority to pass. Each ini setting is voted on independently so that the internals list can accept a subset of the proposal.

**Change ''session.use_strict_mode'' default from ''0'' to ''1''**

<doodle title="Change session.use_strict_mode default to 1?" auth="jorgsowa" voteType="single" closed="false">
   * Yes
   * No
</doodle>

**Change ''session.cookie_httponly'' default from ''0'' to ''1''**

<doodle title="Change session.cookie_httponly default to 1?" auth="jorgsowa" voteType="single" closed="false">
   * Yes
   * No
</doodle>

**Change ''session.cookie_samesite'' default from '''' to ''Lax''**

<doodle title="Change session.cookie_samesite default to Lax?" auth="jorgsowa" voteType="single" closed="false">
   * Yes
   * No
</doodle>

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
