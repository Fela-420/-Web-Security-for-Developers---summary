Chapter 8 – Cross‑Site Request Forgery (CSRF): Forcing Actions Through Deceit
CSRF forces a victim’s browser to send unwanted requests to a target site where the victim is authenticated. The attacker never sees the response; they just cause the action to happen. It’s a powerful way to change account details, transfer money, or post content—all without the user’s consent.

1. The Core Idea
The attacker crafts a malicious link or auto‑submitting form that sends a request to your site.

Because browsers automatically attach cookies, if the victim is logged in, the request carries their session.

The server sees a legitimate, authenticated request—and executes the action.

Classic attack surface: Any state‑changing action reachable via a simple <a> tag or <form>.

2. Anatomy of a CSRF Exploit
a) GET‑based CSRF (the low‑hanging fruit)

The attacker creates a URL like https://bank.com/transfer?amount=1000&to=attacker

If the bank uses GET for transfers, the victim just has to click the link.

No JavaScript required; an <img> tag can trigger it: <img src="https://bank.com/transfer?amount=1000&to=attacker">.

b) POST‑based CSRF (more work, still doable)

The attacker hosts a page with a hidden form that auto‑submits:

html
<form action="https://site.com/change-email" method="POST">
  <input type="hidden" name="email" value="attacker@evil.com">
</form>
<script>document.forms[0].submit();</script>
When the victim visits this page, their browser sends the POST with the site’s cookies.

If the site lacks anti‑CSRF tokens or SameSite protection, the email change succeeds.

c) PUT / DELETE CSRF

Rare, but can be exploited via JavaScript fetch() or XMLHttpRequest if the server’s CORS policy is permissive.

Usually require a token bypass, but combined with XSS they become trivial.

3. The Four Defenses (from the chapter)
#	Defense	How it stops CSRF
1	RESTful GET usage	Never allow GET requests to change server state. Only use GET for reads. State changes → POST, PUT, DELETE. This kills all <a> and <img>‑based attacks.
2	Anti‑CSRF tokens	A random secret token embedded in the page (hidden field or JS) and checked against a cookie/header on the server. The attacker cannot read the token due to the same‑origin policy.
3	SameSite cookie attribute	Instructs the browser to omit cookies on cross‑site requests. SameSite=Strict strips cookies on all cross‑site requests. SameSite=Lax allows cookies on top‑level GET navigations (safe linking) but blocks POST/other methods.
4	Re‑authentication for sensitive actions	Prompt for password/PIN before critical operations. This breaks CSRF because the attacker cannot know the credentials.
4. The Hunter’s CSRF Checklist
When testing a target, follow this flow to find and verify CSRF bugs.

Step 1: Map state‑changing endpoints

Find any request that modifies data: password change, email update, fund transfer, post creation, API key generation.

Note the HTTP method used. If it’s GET for a sensitive action, that’s a finding by itself.

Step 2: Check for anti‑CSRF tokens

Look at forms and AJAX requests. Do they include a token (e.g., csrf_token, _xsrf, authenticity_token)?

If token is present, test its strength:

Can you remove the token entirely? Does the server still accept the request? → CSRF confirmed.

Can you replace the token with an empty string or a static value (e.g., csrf=1)? → weak token, likely exploitable.

Does the server validate that the token is tied to your session? Try using a token from another user or a different session. If it works, the token isn’t per‑session → CSRF bypass.

Is the token simply a copy of the session cookie? Try duplicating the session value in the token parameter; many older apps do this. → CSRF via token echoing.

Step 3: Check SameSite cookie behavior

If no token is present, look at response headers for Set-Cookie. Is SameSite=Strict or Lax set?

If a session cookie lacks SameSite, and there’s no token, you have CSRF on any POST action.

Even with SameSite=Lax, the site is still vulnerable to top‑level GET CSRF (navigations). That’s why #1 (no GET state changes) is critical.

Bypass note: Some browsers (older Chrome) allowed SameSite=Lax to be bypassed by a window open in a few seconds, or by using SameSite=None with an older incompatible client. Always test manually.

Step 4: Test CSRF exploit delivery

Build a proof‑of‑concept (PoC) page on your own domain or a service like JSFiddle/CodePen.

For GET: <img src="https://target.com/change?email=attacker">

For POST: auto‑submit a form. If you need to include token, try to find token leakage (e.g., via CORS misconfig, JSONP, or a vulnerability like CRLF injection) – but by default, a CSRF finding assumes no token leakage.

Step 5: Check for CSRF via JSON/XML

If an endpoint accepts Content-Type: text/plain or application/json but no token, you can sometimes exploit it by sending a form with enctype="text/plain". Manipulate the formatting to inject JSON structure.

Test with fetch() from your PoC if the server’s CORS allows credentials (Access-Control-Allow-Origin with true). Then it becomes a CRSF‑with‑CORS scenario.

5. Defense‑in‑Depth: Combine Protections
Always use anti‑CSRF tokens and SameSite cookies together. A token guards against a SameSite bypass; SameSite guards against token theft via header injection.

Re‑authentication is the ultimate safety net for money transfers or password changes – it’s immune to CSRF entirely.

Don’t rely on checking Referer/Origin headers alone; they can be spoofed or stripped by privacy tools, but when present they add another layer.

6. Practical Exploit Examples (from history)
Twitter worm (2009): GET request to /share/update?status=... tweeted without user confirmation. No token, no SameSite. A click was enough.

Gmail contact theft: Attackers used a POST CSRF to add a forwarding rule in Gmail settings, copying all emails to an attacker’s address.

Router CSRF: Changing DNS settings on home routers via a single <img> tag – many routers still use GET for configuration changes.

