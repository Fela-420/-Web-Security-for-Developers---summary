 Chapter 4 – How Web Servers Work: Attacking the Engine Room
A web server isn’t just a box that spits out HTML. For a hunter, it’s a layered attack surface—from how it resolves a URL to which database it queries. This chapter gives you a map of every component you can probe.

1. Static Resources: File‑Serving Can Bleed
Attack surface: The mapping of URLs to files on disk. If the server resolves /images/../etc/passwd or /profile/123/../private.png, you’ve got a path traversal.

Probe: Try URL‑encoded slashes, backslashes, and NULL bytes. Even if the resource is “static,” the resolver might leak internal files.

CDNs: A third party serving content under your target’s certificate. Check if the CDN origin is misconfigured—can you bypass the CDN and hit the origin IP directly? Look for cache poisoning or subdomain takeover if a CDN endpoint is abandoned.

Compression/Caching headers: Cache-Control: public on sensitive data? That’s a PII leak. Burp Suite can spot this.

2. Content Management Systems: The Unpatched Goldmine
The chapter explicitly shows a Google search for WordPress vulnerabilities. Hunters live off unpatched CMS instances.

Quick wins: Identify the CMS (Wappalyzer), check its version, then fire up wpscan or a CVE search. A single outdated plugin is an entry point.

Third‑party integrations: CMS plug‑ins that handle payments, appointments, or support often introduce new vulnerabilities. Map all API endpoints they expose.

3. Dynamic Resources: Where Injection Lives
Dynamic resources execute code based on user input. Every parameter that touches a dynamic resource is a potential injection point.

Templates: Server‑side template injection (SSTI) occurs when user input is inserted into a template before rendering. Test with {{7*7}}, ${7*7}, <%= 7*7 %> depending on the stack.

Databases (SQL): SQL injection is still king. Relational databases use keys and constraints, but a single unescaped ' can dump the whole user table. Focus on error‑based, blind, and out‑of‑band techniques.

NoSQL: NoSQL injection often hides in JSON or URL‑encoded operators. Try {"$gt": ""} in MongoDB, or {$ne: null} in query parameters. Schemaless means you can inject fields that grant admin access.

Distributed Caches (Redis/Memcached): Rarely internet‑facing, but if you find an exposed Redis instance, you can write your own SSH key or exfiltrate session data. Check port 6379.

4. Web Programming Languages: Know Your Target’s Weak Spots
Each language has signature vulnerabilities. Use the stack fingerprint to narrow your hunting:

Language/Framework	Hunter’s Focus
Ruby on Rails	Mass assignment, YAML deserialization, CVE‑heavy history. Check for Rails versions with known RCE.
Python (Django/Flask)	SSTI in Jinja2 templates, debug mode leakage (/console), pickle deserialization.
Node.js (Express)	eval()‑type sinks, child_process.exec() with user input, prototype pollution in merging libraries.
PHP	Loose comparison (== vs ===), type juggling, unserialize() mayhem, file inclusion (LFI/RFI). Legacy code = multiple vulns chained.
Java	Deserialization (ysoserial), JNDI injection (Log4Shell), OGNL injection in Struts.
C#/.NET	ViewState deserialization, path traversal via ..\; on Windows, SQLi via LINQ if not parameterized.
5. Client‑Side JavaScript Frameworks: DOM‑Based XSS
Angular templates can evaluate expressions. If user data ends up in ng-bind-html unsafely, XSS is trivial.

React’s dangerouslySetInnerHTML is a red flag—look for props that pass unsanitized user input.

Modern hunt: Search for sinks like innerHTML, outerHTML, document.write(), or eval() in minified JS. Even a seemingly safe URL in a React app can become javascript:alert(1).

6. Architectural Checklist: Follow the Request
When a browser requests a dynamic page, the server touches many components. Exploit each handoff:

URL resolver – path traversal, forced browsing of hidden endpoints.

Authentication layer – bypass via SQLi, cookie tampering.

Template engine – SSTI.

Database/Cache – injection, exposed interfaces.

Response – HTTP header injection, content type confusion, cache poisoning.
