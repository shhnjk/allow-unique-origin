# Sandbox `allow-unique-origin`

A Web Platform API proposal to improve sandboxing and Blob URL

David Dworken and Jun Kokatsu, Aug 2023 (©2023, Google)

## TL;DR: Add a `allow-unique-origin` sandbox keyword and response headers to Blob URLs

### `allow-unique-origin` sandbox keyword

Add a new [sandbox](https://html.spec.whatwg.org/multipage/browsers.html#sandboxing) keyword, `allow-unique-origin`, that causes the rendered content to execute in a unique non-`null` origin. 

The origin of a document sandboxed in this way will be `sandbox:["$RANDOM_UUID","$PRECURSOR_ORIGIN"]`. For example, if `https://example.com` set a `CSP: sandbox allow-unique-origin` header, then the origin of the document would be `sandbox:["9138ee47-c4f7-4e30-8751-acf51834e3f6","https://example.com"]`. Having a unique origin that contains the precursor origin, makes it possible to implement a number of useful product features like checking the precursor origin when responding to CORS requests.

These sandboxed pages will have access to isolated new storage partitions with a lifetime scoped to the current page. This includes `document.cookie`, `window.localStorage`, `window.caches`, and more.

Since unique-origin documents are considered cross-origin (and cross-site) from the precursor origin (and any other origins), it should be process-isolated whenever possible by User Agents.

### Blob URL Response Headers

Expose a `headers` option to [BlobPropertyBag](https://w3c.github.io/FileAPI/#dfn-BlobPropertyBag) in the [Blob constructor](https://developer.mozilla.org/en-US/docs/Web/API/Blob/Blob). The `headers` option will take in an object containing response headers to associate with the Blob. Currently, supported headers are `Content-Security-Policy` and `Content-Disposition`.

This will enable a number of new uses for Blobs including creating Blob URLs that are guaranteed to not lead to XSS vulnerabilities:

```
const blob = new Blob([untrustedHTML], {
      type: 'text/html',
      headers: {
        'Content-Security-Policy': 'sandbox allow-scripts allow-unique-origin'
      },
    });
const safeBlobUrl = URL.createObjectURL(blob);
iframe.src = safeBlobUrl;
window.open(safeBlobUrl);
```

In this case, `safeBlobUrl` is guaranteed to render in a unique origin no matter how it is used, and thus it cannot lead to an XSS vulnerability in the creating application.

When a Blob sets a CSP policy that includes the `allow-unique-origin` sandbox keyword, it does not inherit the CSP of the creator. This matches the web's model where loading an iframe with a distinct origin also does not inherit the CSP of the parent page. 

## Why do we need this?

It aims to solve three problems.

### 1. XSS through Blob URLs

Blob URL is useful for loading locally available resources. However it also leads to XSS bugs when Blob URLs are created with untrusted content.

1. [XSS on WhatsApp Web](https://blog.checkpoint.com/2017/03/15/check-point-discloses-vulnerability-whatsapp-telegram/).
2. [XSS on Shopify](https://hackerone.com/reports/1276742).
3. [XSS on chat.mozilla.org](https://gccybermonks.com/posts/xss-mozilla/).

This proposal makes it possible to create sandboxed Blob URLs that are guaranteed not to cause an XSS (because any JS will execute in a unique origin). 

### 2. A web-platform alternative to sandbox domains

Many Web apps require a place to host user contents (e.g. `usercontent.goog`, `dropboxusercontent.com`, etc) to safely render them. In order to do so securely (e.g. to avoid exploitable XSS, [cookie bombing](https://speakerdeck.com/filedescriptor/the-cookie-monster-in-your-browsers?slide=26), and [Spectre attacks](https://security.googleblog.com/2021/03/a-spectre-proof-of-concept-for-spectre.html)), many sites rely on [sandbox domains](https://security.googleblog.com/2012/08/content-hosting-for-modern-web.html) to get unique origins. Sandbox domains entail significant complexity for authentication (especially if one also adds it to the [public suffix list](https://publicsuffix.org/)) and are [very hard to do right](https://security.googleblog.com/2023/04/securely-hosting-user-data-in-modern.html#:~:text=Classical%20Solutions%20for%20Isolating%20Untrusted%20Content). Adding the ability to create Blob URLs, iframes, and top level pages with unique origins can replace this complexity with a native Web platform API. 

### 3. Increasing the applicability of sandboxing

CSP sandbox is widely used for isolating content, but running in a `null` origin significantly limits many uses:

* Sandboxed content can't access any storage APIs. Many third-party scripts (e.g. analytics scripts) rely on being able to access these APIs, so it becomes impossible to sandbox these pages. 
* Sandboxed content sends requests with a `Origin: null` header, so there is no way for services to make fine grained decisions based on exactly what content is executing in a given sandboxed page. 

Supporting a unique origin, fixes both of these limitations and will enable more services to use sandboxing to isolate untrusted content. 

## FAQ

### Why not [Safe Blob URL proposal](https://github.com/shhnjk/Safe-Blob-URL)?

Compared to that proposal, this doc proposes a more generic solution for creating unique origins that can be used for more than just Blob URLs. This makes it easier to use this feature to refactor existing web applications. 

However, there is a downside to this proposal:

1. It does not fully address XSS issues stemming from Blob URLs. SVG use elements could point to a same-origin sandboxed Blob URL and thus lead to unsandboxed code execution. 
    * This seems acceptable since this is a very rarely used feature. 

### The format of cross-origin Blob URLs look weird. Why did you choose that?

Having origins like `sandbox:[$RANDOM_UUID][$ORIGINAL_ORIGIN]` make it possible for endpoints receiving CORS requests to ascertain both the original origin, and which unique origin a request is coming from. I'm not attached to this format, but I believe including this information is useful. 

### Why is CSP not inherited from the creator to Sandboxed Blob URLs?

A creator of cross-origin Blob URLs is often a sensitive origin. And therefore it is likely to have a [strict CSP](https://csp.withgoogle.com/docs/strict-csp.html) to defend against XSS attacks.
However, the creator likely is using a sandboxed Blob URL in order to safely run arbitrary HTML/JS (because it acts like a sandbox domain), which is not possible if CSP is automatically inherited.

Conceptually, this isn't a CSP bypass since it is treating sandboxed Blob URLs identically to cross-origin iframes. 

However, some websites use CSP for containment purposes. In order to ensure that this doesn't bypass those CSPs we'll add a `'allow-unique-blob'` CSP keyword. This means that a CSP like `frame-src blob:` won't allow loading sandboxed blobs unless it is changed to `frame-src blob: 'allow-unique-blob'`. 

## Acknowledgements

Thank you to the following folks who provided insightful feedback which shaped this proposal.

[Damien Engels](https://github.com/engelsdamien), [terjanq](https://github.com/terjanq), [Artur Janc](https://github.com/arturjanc), and [Mike West](https://github.com/mikewest).
