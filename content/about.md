---
title: About
toc: true
---

Open Gensokyo Network is a set of protocols and infrastructure that lets services be reached across the existing Internet while staying open, compatible and private.

## Principles

- **Open.** The protocols and implementations are open; anyone can implement them and run the services and infrastructure themselves. A service's identity can be self-certifying, so trusting it does not depend on any certificate authority.
- **Compatible.** Built on QUIC, HTTP/3 and TLS. New capabilities are added through negotiated extensions; with peers that do not support them, connections work like ordinary ones.
- **Private.** Connections reveal as little as possible to the network and to the infrastructure they pass through, down to which service a client is reaching.

## Self-certifying identity

A service can hold an identity that does not depend on certificate authorities. Its name is derived from the identity's own origin, and every later change, such as a new key, is a signed update that clients verify themselves. No certificate authority or domain registrar sits in the chain of trust. As a result:

- A misissued certificate or a tampered DNS answer cannot impersonate the service.
- The service rotates its keys without asking anyone, and a key it stops endorsing is no longer accepted within a bounded time.
- The identity stays the same when the service's domain, host or hosting changes.

Services identified only by a domain name keep using ordinary certificates.

## Status

The project is in its design phase, and there is no usable release yet. Design write-ups and development notes are published on the [blog](/blog/). Experimental code, such as [nitori](https://github.com/opengensokyonet/nitori), is developed in the open on GitHub.

## Support

[Starspun Works](https://starspunworks.com/) provides administrative support for the project.

## License

Unless stated otherwise, text on this site is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Code snippets are dual-licensed under the MIT and Apache 2.0 licenses. The site source is on [GitHub](https://github.com/opengensokyonet/website).

## Privacy

This site sets no cookies and runs no analytics. Your light/dark theme choice is kept in your browser's local storage. The site is hosted on Cloudflare, which processes request data such as IP addresses to serve and protect the site.

## Contact

Questions, corrections and design feedback are welcome on [GitHub](https://github.com/opengensokyonet).
