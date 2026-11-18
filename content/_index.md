---
title: Open Gensokyo Network
layout: hextra-home
---

<div class="hx:mt-6 hx:mb-6">
{{< hextra/hero-headline >}}
  Open Gensokyo Network
{{< /hextra/hero-headline >}}
</div>

<div class="hx:mb-12">
{{< hextra/hero-subtitle >}}
  Open, compatible and private connectivity&nbsp;<br class="hx:sm:block hx:hidden" />for services on the existing Internet.
{{< /hextra/hero-subtitle >}}
</div>

<div class="hx:mb-12">
{{< hextra/hero-button text="About the project" link="about" >}}
</div>

{{< hextra/feature-grid >}}
  {{< hextra/feature-card
    title="Open"
    subtitle="The protocols and implementations are open; anyone can implement them and run the services and infrastructure themselves. A service's identity can be self-certifying, so trusting it does not depend on any certificate authority."
  >}}
  {{< hextra/feature-card
    title="Compatible"
    subtitle="Built on QUIC, HTTP/3 and TLS. New capabilities are added through negotiated extensions; with peers that do not support them, connections work like ordinary ones."
  >}}
  {{< hextra/feature-card
    title="Private"
    subtitle="Connections reveal as little as possible to the network and to the infrastructure they pass through, down to which service a client is reaching."
  >}}
{{< /hextra/feature-grid >}}

<div class="content hx:mt-12 hx:w-full">

{{< callout type="info" >}}
**Status: early implementation.** [sakuya](https://github.com/opengensokyonet/sakuya), the project's QUIC implementation in Rust, completes handshakes and exchanges data with other QUIC implementations; its APIs are experimental, and there is no release yet. The protocols built on it are still being designed. Code, including sakuya and [nitori](https://github.com/opengensokyonet/nitori), is developed in the open on GitHub.
{{< /callout >}}

## Latest from the blog

{{< recent-posts count="5" >}}

</div>
