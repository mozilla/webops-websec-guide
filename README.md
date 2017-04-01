Securing WebOps websites using Web Security Headers

# First, everything but CSP.

CSP is difficult to design, test, and deploy. This section addresses non-CSP issues first, so that we can gain valuable security improvements prior to tackling CSP.

## SSO — Single Sign On (Okta, Auth0)

SSO gateways do not use the same web security headers as the site they intercept and authenticate for. While this is not always to our liking, it presents a complication. Observatory will follow redirects to a final endpoint site, whatever that may be. We’re nearly always interested in securing *our* site, post-SSO, rather than the *SSO* site, which gets a known score.

To scan a site beyond SSO, you must sign in to the SSO site in your browser, locate whatever cookies they use for SSO, and include those in your Observatory scans. For a Webops site hosted with mod_auth_mellon for SAML SSO, these are the X-Mapping and mellon cookies. You put these cookies into a JSON hash of { "cookie-name": “cookie-value” } and pass that JSON hash on the command line to the Observatory local scanner. See the heading ‘[Observatory local scanner](#heading=h.6iq4oajdpqb7)’ later in this document.

## HTTPS — SSL/TLS certificates

The majority of the sites we secure are HTTPS, with an HTTP-to-HTTPS redirect. Occasionally we come across a site that is HTTP-only. Most of those sites should be set up to redirect to HTTP-to-HTTPS, but it varies occasionally.

First, ensure that the site has HTTPS-capable clients, and that they’ll be able to follow the HTTP-to-HTTPS redirect. This is usually the case for human beings operating a browser, but may not be true for features built-in to a browser and other automated clients. In rare cases, we’ll set up the HTTPS site and then alter clients one-by-one to use HTTPS instead, with an eventual redirect-or-shutdown once that work has concluded.

Second, set up an HTTPS endpoint for that site, using a trusted SSL certificate. This is usually easy as we host most sites ourselves, but sometimes they’re offsite at a SaaS provider. We usually issue our SSL certificates from Digicert for Zeus-hosted sites and from Amazon for ELB-hosted sites, with occasional use of Let’s Encrypt and Thawte. In very rare cases certificate pinning dictates the SSL issuer; see heading '[HPKP](#heading=h.ehvm3s3awelk)' for more.

Third, redirect HTTP to HTTPS. There are a set of rules associated with doing this — http://xyz must always redirect to https://xyz, even if the eventual target is https://bcd – HSTS should be set on https://xyz, even if the eventual target is https://bcd — that the observatory scanner will call out for you if any repairs are required. See heading '[curl](#heading=h.dtr0iaswt1tn)' for more.

## HSTS — Strict-Transport-Security

The majority of the sites we secure are HTTPS, with an HTTP-to-HTTPS redirect. All HTTPS endpoints associated with a site should have an HSTS header set. This is the most likely issue we’ll encounter on an unsecured site, it’s extremely important to the websec team, and an easy win for most sites.

Occasionally a site will not be HTTPS-only — for instance, Relengweb or HG — and in those cases, HSTS may take more time to evaluate and ship. HSTS only affects web browsers, not automated and command-line clients. HSTS does not require an HTTP-to-HTTPS redirect, but it will be more effective with the redirect in place. HSTS is ignored by ancient browsers. HSTS can be activated for a given domain by loading any resource from that domain with the HSTS response header present, even from another website.

Most sites should protect both the domain and all subdomains thereof. We prefer to use a 1-year time interval when a site has been operating under SSL with HTTP-to-HTTPS redirects for some period of time but occasionally proceed with the more careful 1 minute / 1 hour / 1 day intervals to handle any issues. The following example will work for most domains. includeSubdomains MUST NOT be used on any top-level domain where we do not control the subdomains, such as mozilla.com and mozilla.org.

Strict-Transport-Security: max-age: 31536000; includeSubDomains

## XFO — X-Frame-Options, CSP ‘frame-ancestors’

The simplest defense to clickjacking is to simply prohibit a site from being loaded in a frame by undesirable sites. We typically choose to prohibit all framing of a site, but several options exist. Depending on the option chosen, this can require either an HTTP header and/or a CSP clause. Testing is mandatory here, as with all things involving CSP, but the testing required is less than is required for other CSP clauses.

If framing should be prohibited entirely, then we use both the HTTP header and the CSP clause, as shown below. XFO: DENY is supported by older clients and CSP: frame-ancestors is supported by newer clients. No other CSP clause interacts with the frame-ancestors directive, and default-src doesn’t affect it. Defining a CSP with only the frame-ancestors clause will not activate any other CSP protections. Neither default-src nor any other CSP clauses interact with frame-ancestors, so it is generally safe to prepend to any existing policy.

X-Frame-Options: DENY
Content-Security-Policy: …; frame-ancestors: 'none' ; …

If the site needs to frame itself — that is, if a page https://xyz/one expects to frame https://xyz/two — then we use the XFO: SAMEORIGIN policy and accompanying CSP clause. This does not permit other sites to frame it, but is effective for sites that expect framing.

X-Frame-Options: SAMEORIGIN
Content-Security-Policy: …; frame-ancestors: 'self' ; …

If the site needs to permit specific other sites besides itself to frame it, then those sites can permitted either instead of or in addition to the site itself. XFO cannot be used in this case — ALLOW-FROM is not supported — and must removed. This clause supports only domain names and URI protocols, with one possible wildcard. Single-quotes MUST be used with 'self' and 'none', and MUST NOT be used with domains and protocols. Trailing slashes MUST NOT be used with domains. The trailing colon MUST be present on protocols. Several example clauses follow:

Content-Security-Policy: frame-ancestors: …

'self' https://xyz.com ;
https://*.bcd.com ;
https://*.efg.com:4443 http://bcd.com ;

Very rarely, a site needs to permit all other sites to frame it. This is uncommon but can occur. Evaluate carefully whether it’s safe for a site to be framed. Sites with a ‘login’ feature should not permit framing by arbitrary sources. If necessary, framing can be permitted either by HTTPS sites only – or otherwise it can be permitted by all sites – using a CSP clause. XFO cannot be used in this case and must be removed. It is important to declare a framing policy for the site, both for the Observatory score and to indicate that we evaluated the site to need this policy.

Content-Security-Policy: frame-ancestors: …

https: ;
* ;

## XXP — X-XSS-Protection

This header instructs certain browsers (MSIE 8+) to automatically defend against XSS attacks. We generally enable it on all sites as it only rarely causes false positives, and only on MSIE. We have not run into any instances where this header had to be removed. It is not a complete defense against XSS attacks, and should be combined with other protections. It’s an easy win.

X-XSS-Protection: 1; mode=block

## XCTO — X-Content-Type-Options

This header prevents browsers from using ‘magic’ file detection routines to override the Content-Type provided by the server. This can, for instance, be abused by malicious actors to serve Windows executable files with a filename of .png that ends up being executed somehow by the client. Any site depending on this behavior is at risk of the site breaking if/when the browser’s ‘magic’ routines change. We haven’t found any sites that break when this is applied.

X-Content-Type-Options: nosniff

# Second, CSP.

## Always be in the developer console.

If you're working with CSP, you need to have your browsers' developer consoles open. CSP errors are logged to the error console. You also have to test with multiple browsers. Firefox, Chrome, Safari. I develop in one, then test in all.

## I don’t have time for this right now.

The below CSP header is generally safe when applied to most sites, without any further tuning. It blocks <object> tags from activating browser plugins like Flash, permits only HTTPS resources, and makes no assertions about framing (see XFO, above). It is less secure than a carefully-tuned CSP header, but it’s also better than none at all. It will not a substitute for a proper CSP evaluation, and will only gain a few points on the observatory, but at least it’s there.

Content-Security-Policy:

default-src: https: 'unsafe-inline';
object-src: 'none';

Always be in the developer console, even if you don't have time to write a CSP header, or you'll miss important errors.

## Understanding the header syntax.

CSP’s header syntax is somewhat tricky, and is easier to convey using examples. These clauses will be explained later on. Each of these examples is a valid CSP header, though they may not be particularly useful when copy-pasted into a given site.

Content-Security-Policy: *<example>*

### Valid examples

frame-ancestors: 'self' https: http://insecure.site.com:8080;

This example contains one CSP attribute, ‘frame-ancestors’, with three parameters. The ‘self’ parameter is one of the special parameters, and MUST always be quoted. The protocol scheme and domain parameters MUST NOT be quoted.

default-src: 'none'; script-src: 'self' https://jquery.com; object-src: *; img-src: data:;

Three attributes, terminated by semicolons. ‘default-src’ sets the default for all -src directives (‘frame-ancestors’, for instance, is not included in its defaults). ‘*’ here means "any source except data: URIs", which is usually interpreted to mean “all sources”. Images may be loaded from data: URIs and nowhere else. This is not a very practical header.

### Invalid examples

default-src: none;

The unquoted none here is invalid. For whatever reason, CSP explicitly quotes specific values. Those values vary per attribute, but are all single-quoted.

default-src: 'https:' 'http://example.com' '*';

The quoted clauses here are invalid. CSP only permits quoting of the special values. Protocols, URIs, and the wildcard aren’t special values.

script-src: 'unsafe-inline' 'hash-8f7as9d8f7as9d7f';

Trick question. The syntax is correct, but the spec for this attribute prohibits mixing these two values.

## Building a header from scratch.

Begin with the below CSP header. It disables most things by default and will almost certainly break every site it’s applied to at first. Newlines are only for the purpose of explanation; all CSP clauses should be one header with a single, one-line, value.

Content-Security-Policy:

frame-ancestors: 'none';
default-src: 'none';
img-src: 'self';
script-src: 'self';
style-src: 'self';

If you can ship your site with this policy and you don’t encounter any CSP errors in your developer tools, congratulations. If not, you’ll need to start adding exceptions. Each time a page resource violates the CSP policy, it will log an error to the developer console. Reviewing those errors is the only known way of testing a CSP policy at this time.

### Framing permissions

Framing is prohibited initially by ‘none’, but can be permitted by setting various framing options. See the "XFO — X-Frame-Options" section earlier in this document for more guidance here.

### Defaults none and self

The default ‘none’ prohibits the resources, and then the various ‘self’ permit resources that are loaded from the site itself. This is often sufficient for some sites, but will break for offsite resources. For example, to fix Google Fonts, the following policy attributes would be needed:

font-src: … https://fonts.gstatic.com …;
style-src: … https://fonts.googleapis.com …;

### On * and data: URIs in -src policies

When writing -src policies such as img-src: *;, it's essential to know that the wildcard does not match data: URIs. There's a weird browser spec where you can encode a file as base64 and then say data:base64goeshere, and the browser will actually load up the base64 from the URI rather than from some file on disk or remote URL or whatever. CSP does not include data: URIs in the wildcard because they're unsafe in certain ways, and because they're rare. If your site uses them, add them as you would any other URL. Valid example:

style-src: 'self' data: 'unsafe-inline' https://styles.example.com;

### Inline scripts and styles

Many sites write <script> and <style> tags directly into their page source, containing raw JavaScript and CSS rather than using <script src> and <style src>. This is prohibited by the above policy, as it permits attacks where the page content is manipulated (on-server, in-transit, or in-browser). There are three approaches to fixing this. They are applicable to both <script> and <style> tags, which will be treated interchangeably in this explanation.

#### hash-src: SHA256 content hashes

Content-Security-Policy:

style-src: 'sha256-18234717fdsa7f' 'sha256-77vdnqnejru';

<style> P.DIV { font: bold; } </style>
<style src="77vdnqnejru.css"></style>

Each time the page is generated and served to the user, the SHA256 hashes of the authorized content are included in the CSP header. The browser will accept any inline content loaded with either <style> or <style src> with a matching SHA256 checksum. Mediawiki may implement CSP using this approach, to permit per-user configurable resources in a CSP-compliant manner. It is more common than nonces, and is a viable inline content solution when <style src> migration is not an option.

#### nonce-src: Random single-use nonces

Content-Security-Policy:

script-src: 'nonce-1284sdfa8a7fa';

<script nonce="1284sdfa8a7fa"> window.alert(1); </script>

Each time the page is generated and served to the user, a new random nonce hash is generated and inserted into the CSP header in the appropriate attribute. That same one-time nonce is injected into the <script> tag by the server. The browser accepts all inline content in the page that has matches one of the specified one-time nonces. This is an uncommon approach, and it’s usually more effective to migrate inline content to the <style src> method instead.

#### unsafe-inline: Permit inline content

Content-Security-Policy:

script-src: 'unsafe-inline';

<script> window.alert(1); </script>

This disables CSP protections against inline content. As noted in the attribute name, it is considered unsafe, and permits in-page manipulation using the above-described attacks. It is unfortunately also the most common approach to solving the inline content problem, as it does not require making any changes to the content.

## Reporting and ‘report-only’ mode.

CSP reporting is not supported by Firefox, which presents complications for developing and testing CSP in report-only mode in production. It'd still be nice if we could collect CSP reports from clients, since that would let us know if we'd failed to implement the CSP policy correctly.

There are public CSP reporting sites that let you submit them your site's reports. This is fine for personal use only. We treat user visits as PII, so offsite reporting isn't an option, and we have no authorized endpoint today for this purpose.

# Third, everything else.

If you’re motivated to go the extra mile, here’s some of the ways you can do so. Some of these may require modifying the application. These may be difficult or impossible to implement, and must be evaluated on a case-by-case basis.

## HSTS Preload

We encourage submitting new domain names to the HSTS Preload list. Once accepted, and after some time has passed, all modern browsers will force an implied policy of "max-age: forever; includeSubDomains" for all requests to all hostnames at your domain. Approval is automatic once you add the preload clause. Removal is not: there is generally no way to remove a site from the list once it's added. To submit a domain name, make sure your site's HSTS header already had includeSubDomains, add preload, and then [submit it to the preload list](https://hstspreload.org/). (If it doesn't already have includeSubDomains, adding it will enforce the declared HSTS policy across all sites at your domain.)

Strict-Transport-Security: max-age: 31536000; includeSubDomains; preload

## SRI — Subresource integrity hashes

SRI adds checksums to your <script src> and <style src> elements, and enforces them on the loaded sources. If the checksum doesn't match, the content isn't loaded. This protects against content injection attacks. It's best to have some automated way of calculating and updating checksums in your site repository. Firefox and Chrome support SRI and can be used to develop and test it. SRI decays gracefully on older browsers, which ignore the 'integrity' attribute entirely and simply load the unchecked content as usual.

### Offsite only

If your site loads <script src> or <style src> from some other site (CSP -src policy more than 'self'), and that content will never change (e.g. jquery-1.11.2.min.js), then you can calculate the SRI checksum for that content and include it in the page. If that site ever gets hacked, your site will break *without* exposing your visitors to hacked JS.

<link rel="stylesheet" type="text/css" href="css-1.2.3.min.css" integrity="sha384-X7L1bh.....">

<script type="text/javascript" src="js-1.2.3.min.js" integrity="sha384-UMMEM1.....">

You can calculate the SRI checksum for a single filename using this command:

echo -n 'sha384-'; openssl dgst -sha384 -binary < *filename* | openssl enc -base64 -A; echo;

And you can download a single remote source (to calculate a checksum) using this command:

curl -o *filename* https://remote/js/css

### Everything

This form of SRI is less common for sites that co-host their HTML and script/style resources from the same servers, as it provides only a limited defense against a determined attacker in that scenario. If it's easy to implement, it's worth it.

You can write a build process for your site that calculates SRI hashes for every <script src> and <style src> target in the site, assuming that they're all loaded as static files from the server, and include those hashes into your page content for every resource. Avoid writing SRI hashes into your site by hand if possible. One of our sites uses [a Makefile](https://github.com/mozilla/phonebook/blob/master/Makefile) to regenerate [the SRI hashes](https://github.com/mozilla/phonebook/blob/master/config-srihashes.php) after changes are made by developers.

### Enforcement

A draft CSP extension 'require-sri-for' permits you to tell browsers to refuse to load sources that do not have SRI integrity attributes. This extension is available today, off-by-default, in release channel Firefox and Chrome, and the Observatory [will likely add support](https://github.com/mozilla/http-observatory/issues/216) for it soon. If you enable this policy, please also enable browser support and test.

Content-Security-Policy: …; require-sri-for: script style; …

## HPKP — HTTP Public Key Pinning

You can ask browsers to not only require HTTPS to your site, but also to require a specific set of certificate authorities to have signed the certificate in use at your site. This is not appropriate for everyone, and is a significant commitment to get right. Managing certificate authority hashes (and planning ahead for future authority changes) is a difficult task. This protects users against CAs other than your approved list being used to issue illegitimate certificates for your site. Configuring HPKP is beyond the scope of this guide.

## RP — Referrer policy

Referrer-Policy is a header that lets you specify how capable browsers should treat the Referrer header when visiting your site. Support varies for the options available. It's important to plan for what happens when browsers ignore your first choice of policy. Consider starting with this header:

Referrer-Policy: strict-origin-when-cross-origin

This is a safer 'default' policy that still provides you referrer headers for requests from your site to *your* site, but strips the path information when sending referrer headers to *other* sites. It's not supported in all the major browsers, and "fails open" to the default referrer behavior we're accustomed to today.

If you're confident your site doesn't need to be listed in referrer headers, you can disable them and save bandwidth:

Referrer-Policy: no-referrers

Be wary of any of [the other options](https://www.w3.org/TR/referrer-policy/) unless you're prepared to compromise a set of very well-defined referrer needs.

# References

## Mozilla Web Security Guidelines

[https://wiki.mozilla.org/Security/Guidelines/Web_Security](https://wiki.mozilla.org/Security/Guidelines/Web_Security)

This is a reference manual, covering all of the headers we expect to be using for this work. I recommend setting aside an hour and reading the entire document top to bottom once, and then using it as a reference manual when evaluating how to secure sites. Clicking on the ‘Order’ column heading on the table at the top will sort it from "most important" to “least important”, as evaluated by the web security team, but our ordering varies somewhat from theirs.

## The 26-step CSP generator

[https://report-uri.io/home/generate](https://report-uri.io/home/generate)

Once you’re familiar with how to build a CSP header, you could theoretically use this tool to ensure that whatever you do build is, at the very least, valid syntax. There are 26 steps in the generator, which is significantly more than we configure ourselves below. Most of these steps are beyond what we can evaluate on our own and should be ignored unless listed below.

## Developer tools

### Scanning

#### Observatory by Mozilla

[Observatory by Mozilla](https://observatory.mozilla.org/) works on any public site where the policies expressed at the root of the site match those of the rest of the site. If your site is protected by authentication or SSO, or is very complex and has multiple policies at multiple endpoints, install and run the local scanner.

#### curl

curl has no local caching between runs, making it a perfect tool for testing things that browsers would normally cache. It's an efficient way to inspect and confirm that a chain of site x -> site y redirects are done correctly to ensure that the site x HSTS header is sent to the client before sending them off to site-y.

$ curl -v http://site-x
Location: https://site-x

$ curl -v https://site-x
Location: https://site-y
Strict-Transport-Security: ...

$ curl -v http://site-y
Location: https://site-y

$ curl -v https://site-y
Content-Type: text/html
Strict-Transport-Security: ...

#### Observatory local scanner

Clone [gh:mozilla/http-observatory](https://github.com/mozilla/http-observatory) and open README.md. The heading "Running a scan from the local codebase" provides the necessary steps to get it running on any Python3 capable system. OS X users can access Python3 safely through homebrew using ‘brew install python3’.

If the site you’re accessing has SSO session cookies that are necessary to scan it correctly, use your favorite browser’s developer tools console to find the cookie and copy it into the JSON.

If the site you’re accessing has basic authorization, you can capture your password *as plaintext* by copying the value of the request header ‘Authorization’ and specifying it using JSON.

In both the cookie and header cases, the JSON structure is { "name": “value”, “name”: “value” }. When you provide the correct cookies and/or headers, the site should permit you to request and access the root at https://. Take special care to protect these JSON blobs, as those session cookies could be reused to hack whatever site you’re trying to secure.

httpobs-local-scan --format report --cookies '{"foo": "bar"}' example.com   # --headers

This command outputs a clear report that can be copy-pasted, diff’d, and so forth.

### Browsers

It’s often necessary to ensure that headers work correctly in multiple browsers, and so I assume you have at least Firefox and Chrome available (release channel is fine).

Chrome has offered me more clear guidance about CSP failures than Firefox, but that could always have changed. In both cases, you’ll need to open the developer console. (Tools or View -> Developer -> Show Developer Console.) Once you’re there, it’ll usually show warnings.

Safari works as well, with varying levels of stability on developer releases of Apple platforms.

### Caching

Browsers disable caching entirely by default when, in the developer tools, you navigate to the Network Timeline pane. You can reenable it yourself manually, but it’s generally more predictable to study uncached queries.

This has specific value for CSP testing: if you’re modifying CSP headers frequently, you can force the page load to ignore the caches by reloading at the Network pane. As a bonus, this also lets you check the syntax of the returned CSP header in the inspector, and has various useful benefits to debugging.

### Developing

#### CSP: The Easy Way

Install [Laboratory by Mozilla](https://addons.mozilla.org/en-US/firefox/addon/laboratory-by-mozilla/) extension into Firefox release. Watch the [17 minute instructional video](https://www.youtube.com/watch?v=scKTqjNof20) and try it out on a site. It’ll produce something that looks like a good CSP policy. Ask someone to vet the policy and explain what parts of the site you visited so they can review it.

This extension trivially automates the vast majority of time spent developing CSP. By collecting information your browser already knows about each page, it can calculate the minimum CSP necessary to serve those pages. This is invaluable when securing a wide variety of sites over time.

#### CSP: The Hard Way

If you need to *alter* CSP on your local client *only*, to experiment with CSP changes in a client-side manner that protects the server from your risky choices, then you need to intercept and modify your own traffic to the remote site. You’d think this would be the domain of browser extensions, but as of right now, it isn’t.

It is extremely difficult to develop CSP from scratch right now, due to an absence of simple client-side request-header alteration tools. There are extensions, but they need to be forked and patched by an experienced developer to strip out all of the advertising and account gunk. So I personally use a solution that requires a boring OS X setup. If you’re running super complex firewalls and things, you’re on your own to trap and route the traffic.

Mitmproxy is available for OS X users on Homebrew. Proxifier costs a few dollars, and serves as a GUI over the underlying network cruft with useful routing features. I wrote a small Python fragment that acts as an Mitmproxy filter, modifying the CSP header of all traffic routed through an mitmproxy instance:

![image alt text](image_0.png)

And then you use Proxifier to route non-python traffic through mitmproxy, which now alters CSP. Proxifier can be difficult to configure, because you have to only intercept requests from your browser. My [Proxifier configuration screenshots](https://dl.dropboxusercontent.com/u/7533709/Proxifier%20mitmproxy%20setup.zip) may be helpful. It’s tricky business to get Proxifier working correctly, but the simplicity of editing headers live is worth it. There are other solutions on OS X and other platforms as well. I no longer use flow.request.host() as shown in the screenshot above, because Proxifier's got that covered for me, and I'm only testing one CSP header at a time. My app includes in Proxifier are currently *web*;*safari*;*curl*. It's all fragile and takes a lot of fiddling to work.

This space is ripe for improvement. This should just be cross-platform and in-the-browser. Safely developing and testing security headers locally for a live production site shouldn't require knowing Python, Mitmproxy, and Proxifier.

### Testing

CSP headers are often a static webserver header, in which case they’re relatively easy to deploy in a staging environment and then to production. Some applications have more complex requirements for how CSP headers are sent, and will need to be tested where possible to do so.

Verify that the website is not only free of unexpected errors but also actually looks right to someone who’s familiar with it. Web security headers can produce some very slight failures that are easy to overlook as an outsider ot any given site.

