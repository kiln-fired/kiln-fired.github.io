---
layout: default
title: Kiln
---

<section class="hero">
  <div class="hero-copy">
    <div class="eyebrow"><span></span> Kubernetes-native Bitcoin + Lightning</div>
    <h1>Run Bitcoin infrastructure on Kubernetes</h1>
    <p class="lede">Kiln extends Kubernetes with purpose-built APIs for operating Bitcoin and Lightning nodes, preserving node identity, managing credentials, and reconciling the lifecycle of stateful protocol infrastructure.</p>
    <div class="hero-actions">
      <a class="button primary" href="https://github.com/kiln-fired/kiln-operator">View the operator <span>↗</span></a>
      <a class="button ghost" href="#field-notes">Read the field notes</a>
    </div>
  </div>
  <div class="hero-mark">
    <img class="hero-logo" src="{{ '/assets/kiln-logo.svg' | relative_url }}" alt="Kiln logo">
  </div>
</section>

<section class="proof-strip">
  <div><strong>BitcoinNode</strong><span>Declarative btcd-backed nodes</span></div>
  <div><strong>LightningNode</strong><span>Persistent LND lifecycle</span></div>
  <div><strong>LightningPeer</strong><span>Desired peer connectivity</span></div>
  <div><strong>LightningChannel</strong><span>Funded channel lifecycle</span></div>
</section>

<section class="section">
  <div class="section-kicker">What Kiln does</div>
  <div class="section-heading">
    <h2>Protocol-aware lifecycle, expressed through Kubernetes.</h2>
    <p>Kiln models Bitcoin and Lightning as a graph of Kubernetes resources, then reconciles both cluster state and live protocol state. References, ownership, persistence, recovery, credentials, and safety are explicit parts of the API rather than deployment conventions.</p>
  </div>
  <div class="feature-grid">
    <article class="feature-card">
      <span class="feature-index">01</span>
      <h3>Model the protocol graph</h3>
      <p>BitcoinNode, LightningNode, LightningPeer, LightningChannel, and Seed form a reference-based API. Resources declare relationships without turning the custom resources themselves into a Kubernetes ownership tree.</p>
    </article>
    <article class="feature-card">
      <span class="feature-index">02</span>
      <h3>Preserve identity and recovery</h3>
      <p>Kiln fences stateful volumes, retains node storage, publishes retained Static Channel Backups, and keeps Seed output Secrets independent of the resources that produced them.</p>
    </article>
    <article class="feature-card">
      <span class="feature-index">03</span>
      <h3>Reconcile live protocol state</h3>
      <p>Readiness comes from authenticated runtime checks, not just healthy Pods. Kiln also reconciles Bitcoin peer sets, Lightning peer connectivity, and channel lifecycle against the nodes themselves.</p>
    </article>
    <article class="feature-card">
      <span class="feature-index">04</span>
      <h3>Make safety boundaries explicit</h3>
      <p>Restricted LND client credentials exclude admin access, managed dependencies resolve by reference, external Bitcoin backends are supported explicitly, and mainnet requires deliberate opt-in.</p>
    </article>
  </div>
</section>

<section class="section history" id="field-notes">
  <div class="section-kicker">Field notes</div>
  <div class="section-heading">
    <h2>Notes from active development.</h2>
    <p>These notes capture both the original experiments and the current design work: simulated networks, lifecycle recovery, API boundaries, reconciliation semantics, and the path toward a trustworthy Bitcoin and Lightning control plane.</p>
  </div>
  <div class="post-grid">
    {% for post in site.posts %}
    <a class="post-card" href="{{ post.url | relative_url }}">
      <div class="post-meta"><span>{{ post.date | date: "%d %b %Y" }}</span><span>{{ post.topic }}</span></div>
      <h3>{{ post.title }}</h3>
      <p>{{ post.excerpt | strip_html | truncatewords: 26 }}</p>
      <span class="post-arrow">Read note →</span>
    </a>
    {% endfor %}
  </div>
</section>

<section class="cta">
  <div>
    <div class="section-kicker">Open source</div>
    <h2>Explore Kiln on GitHub.</h2>
    <p>Kiln is under active development. The operator, API definitions, tests, examples, and recovery work are all available in the project repository.</p>
  </div>
  <a class="button inverse" href="https://github.com/kiln-fired/kiln-operator">github.com/kiln-fired/kiln-operator <span>↗</span></a>
</section>
