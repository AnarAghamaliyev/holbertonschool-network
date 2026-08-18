# What Happens When You Type "[https://www.google.com](https://www.google.com)" and Press Enter?

It's one of the oldest interview questions in software engineering, and for good reason: it doesn't test whether you've memorized a framework or a syntax rule. It tests whether you understand what your code actually runs on top of. Millions of engineers ship applications every day without ever thinking about the eight or nine layers of infrastructure that carry a single click from a browser tab to a server rack and back — in under 200 milliseconds.

If you're prepping for an interview and someone hands you this question, here's the first thing to do: **ask which layer they want you to go deep on.** A front-end interviewer might want you to linger on DOM construction and rendering. An SRE or infrastructure interviewer might want the load-balancing and failover story. A backend interviewer might want the database query path. The question is broad by design — the best candidates narrow it down before diving in, rather than reciting a memorized script.

For this post, I'm going to walk through the full path end-to-end, spending real time on the piece that trips up the most candidates: DNS.

Let's start at the very beginning.

## The Setup: You Type, You Press Enter

Before any network activity happens, your browser does a small amount of work. It parses what you typed and decides: is this a URL, or a search query? Since `https://www.google.com` has a valid scheme (`https://`) and a recognizable domain structure, the browser treats it as a URL rather than shipping it off to a search engine.

It then breaks the URL into its components:

-   **Scheme:** `https` — use HTTPS, not plain HTTP
-   **Host:** `www.google.com` — the domain we need to resolve
-   **Path:** `/` — the resource being requested (in this case, the root)

With those pieces in hand, the browser's very first job is to figure out *where* `www.google.com` actually lives on the internet. Domain names are for humans; machines route traffic using IP addresses. That translation is DNS.

## Step 1: The DNS Request

This is the step most candidates gloss over, and it's the one that separates a good answer from a great one.

Every device that connects to the internet is assigned an IP address, and every server serving a website is too. `www.google.com` is a human-friendly alias for something like `142.250....` under the hood. DNS (the Domain Name System) is the distributed, hierarchical system that performs that lookup.

Here's the resolution chain, from fastest to slowest:

1.  **Browser cache** — Chrome, Firefox, and Safari all keep a short-lived cache of recently resolved domains. If you visited Google five minutes ago, the answer might already be sitting there.
2.  **OS cache** — If the browser doesn't have it, it asks the operating system, which keeps its own resolver cache.
3.  **Router / local network cache** — Your home or office router often caches recent lookups for every device on the network.
4.  **ISP's recursive resolver** — If nobody local has the answer, the query goes out to your Internet Service Provider's DNS resolver (or a public one like `8.8.8.8` or `1.1.1.1`, if you've configured your device to use it). This resolver does the heavy lifting on your behalf.

If none of those caches have a fresh answer, the recursive resolver kicks off the full lookup:

1.  It asks a **root name server** — there are 13 logical root server clusters worldwide — "who handles `.com`?"
2.  The root server doesn't know Google's IP, but it knows the **TLD (Top-Level Domain) server** for `.com`, and points the resolver there.
3.  The `.com` TLD server doesn't know Google's IP either, but it knows which **authoritative name server** is responsible for the `google.com` zone, and points the resolver there.
4.  The authoritative name server for `google.com` *does* have the answer, and returns the IP address for `www.google.com`.

You can watch this chain yourself from a terminal:

```
dig www.google.com

;; ANSWER SECTION:
www.google.com.    248    IN    A    142.250.72.196
```

Once the recursive resolver has the answer, it caches it (respecting a TTL, or time-to-live, set by Google) and hands the IP address back to your browser. This entire multi-hop process typically takes single-digit to double-digit milliseconds, and most of the time it doesn't even happen — cached answers make repeat visits nearly instant.

**Interview tip:** if you can explain the difference between a *recursive* resolver and an *authoritative* server, and describe what a TTL does and why it matters for caching, you're already ahead of most candidates.

## Step 2: TCP/IP — Establishing a Connection

Now that the browser has an IP address, it needs to actually open a connection to that machine. This happens over **TCP/IP**, the foundational protocol suite of the internet.

-   **IP (Internet Protocol)** handles addressing and routing — getting packets of data from your machine to the destination machine, hopping across routers along the way.
-   **TCP (Transmission Control Protocol)** sits on top of IP and adds reliability: it guarantees packets arrive in order, retransmits anything lost, and manages flow control so a fast sender doesn't overwhelm a slow receiver.

Before any data is exchanged, TCP performs what's called the **three-way handshake**:

1.  **SYN** — your browser sends a packet to the server saying "I'd like to start a connection."
2.  **SYN-ACK** — the server responds, "Got it, I'm ready too."
3.  **ACK** — your browser confirms, "Great, let's go."

Only after this handshake completes do the two machines have an open, reliable communication channel. This typically happens on **port 443**, since we're connecting over HTTPS (port 80 would be used for plain HTTP).

## Step 3: The Firewall

Somewhere along this path — and often at multiple points — your traffic passes through a firewall. This could be the firewall on your own machine, one built into your router, your company's network firewall, or a firewall sitting in front of Google's infrastructure.

A firewall's job is to inspect traffic and decide whether to allow or block it, based on rules like:

-   Which **ports** are allowed (443 for HTTPS is almost always open; more obscure ports are usually blocked by default)
-   Which **IP addresses or ranges** are permitted
-   Whether the traffic pattern looks like part of a known attack (port scanning, SYN floods, and so on)

On the client side, this is usually invisible — your normal web browsing sails through without a hitch. On the server side, companies like Google run much more sophisticated firewall and DDoS-mitigation infrastructure, filtering out malicious traffic before it ever reaches a web server, so that only legitimate requests make it deeper into the system.

## Step 4: HTTPS / SSL (TLS Handshake)

Because we typed `https://`, not `http://`, the connection needs to be encrypted before any actual web content is exchanged. This is where **TLS** (Transport Layer Security — the modern successor to SSL, though people still say "SSL" out of habit) comes in.

Once the TCP three-way handshake is done, a second handshake happens on top of it — the **TLS handshake**:

1.  **ClientHello** — your browser tells the server which TLS versions and cipher suites (encryption algorithms) it supports.
2.  **ServerHello + Certificate** — the server responds with its choice of cipher suite and presents its **SSL/TLS certificate**, which contains its public key and is signed by a trusted Certificate Authority (CA).
3.  **Certificate validation** — your browser checks that the certificate is valid, unexpired, matches the domain, and was signed by a CA it trusts.
4.  **Key exchange** — the browser and server use asymmetric encryption (the public/private key pair) to securely agree on a **symmetric session key** — a shared secret that's much faster to encrypt and decrypt with than the original public/private key pair.
5.  From this point on, all communication is encrypted using that symmetric session key.

This is why the padlock icon shows up in your address bar: it means the connection is encrypted, and you can verify you're actually talking to Google and not an impersonator sitting in the middle of the network.

## Step 5: The Load Balancer

Google doesn't serve billions of daily searches from a single machine — that would be a single point of failure and would fall over instantly under real-world traffic. Instead, your encrypted request arrives at a **load balancer**, a system that sits in front of a fleet of servers and distributes incoming traffic across them.

Load balancers make decisions based on strategies like:

-   **Round robin** — cycle through servers in order
-   **Least connections** — send traffic to whichever server currently has the fewest active connections
-   **Geographic / latency-based routing** — send you to the data center physically closest to you, which is a big part of why Google feels fast no matter where in the world you are

Load balancers also handle health checks, quietly routing traffic away from any server that's unresponsive or degraded, and they're often where TLS termination actually happens in large infrastructures — decrypting the request before passing it internally, so the servers behind it don't each have to manage certificates themselves.

## Step 6: The Web Server

Once routed to a specific machine, the request hits a **web server** — software like Nginx or Apache (Google runs its own custom infrastructure, but the role is the same). The web server's job is relatively narrow:

-   Serve static assets directly (HTML, CSS, JavaScript, images) when possible
-   Terminate HTTP-level concerns like request headers, cookies, and compression
-   Decide whether the request needs dynamic processing, and if so, hand it off to an **application server**

Think of the web server as the front desk: it can answer simple requests itself, but anything requiring real "thinking" gets routed further back.

## Step 7: The Application Server

This is where your request actually gets *handled*. The application server runs the business logic — the actual code that knows what "searching Google" means. For a request to `www.google.com`, this layer:

-   Parses the request and figures out what's being asked for
-   Applies whatever logic is relevant (in Google's case, kicking off a search, personalizing results based on your account, checking feature flags, and so on)
-   Talks to one or more **databases** or backend services to gather the data it needs
-   Assembles a response, typically as HTML for a full page load or JSON for an API call

This is usually where most of an engineering team's actual application code lives — it's the layer you're probably writing and debugging day to day.

## Step 8: The Database

To build a meaningful response, the application server almost always needs data it doesn't have in memory — user account details, search indexes, cached results, personalization signals. That's the **database's** job.

At Google's scale, "the database" isn't a single machine running PostgreSQL — it's a sprawling ecosystem of distributed data stores, purpose-built for different needs: some optimized for massive read throughput, some for strong consistency, some for the kind of near-instant full-text search that powers the actual Google Search product. But the underlying pattern is the same one you'd see in a much smaller app: the application server sends a query, the database returns the relevant rows or documents, and that data gets folded into the response.

## The Trip Back

Once the application server has what it needs, the response flows back the way it came: application server → web server → load balancer → back across the encrypted TLS connection → through your firewall → to your browser.

Your browser then takes over the final act: parsing the HTML into the **DOM**, parsing CSS into the **CSSOM**, combining them into a render tree, running any JavaScript, and painting pixels to your screen. All of that — DNS lookup, TCP handshake, TLS handshake, load balancing, server processing, database queries, and rendering — usually happens in well under a second.

## Why This Question Matters

It's tempting to treat "what happens when you type a URL" as trivia. It isn't. Every layer in this chain is a place where real systems fail in production: DNS misconfigurations that take down entire sites, TLS certificates that expire silently, load balancers that route traffic to unhealthy nodes, database queries that don't scale. Understanding the full path — even at a high level — is what lets you reason about where things break, and it's exactly why this question has stuck around as an interview staple for so long.

If you're prepping for your own interview, the best way to get comfortable here isn't memorizing this post — it's opening a terminal, running `dig` and `curl -v` against a few real sites, and watching each of these steps happen for yourself.
