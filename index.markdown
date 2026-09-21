---
layout: default
title: Kiln
---

<section class="hero">
  <div class="hero-copy">
    <div class="eyebrow"><span></span> Kubernetes-native Bitcoin + Lightning</div>
    <h1>Run Bitcoin infrastructure like infrastructure.</h1>
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
  <div><strong>Seed</strong><span>Secret-backed wallet material</span></div>
</section>

<section class="section">
  <div class="section-kicker">What Kiln does</div>
  <div class="section-heading">
    <h2>A control plane for protocol infrastructure.</h2>
    <p>Bitcoin and Lightning nodes are stateful, identity-bearing infrastructure. Kiln gives Kubernetes enough domain knowledge to operate them intentionally instead of treating them as generic containers.</p>
  </div>
  <div class="feature-grid">
    <article class="feature-card">
      <span class="feature-index">01</span>
      <h3>Reconcile node state</h3>
      <p>Model Bitcoin and Lightning nodes as first-class Kubernetes resources. Desired state lives in the API, while the operator handles the work of converging the cluster toward it.</p>
    </article>
    <article class="feature-card">
      <span class="feature-index">02</span>
      <h3>Protect Lightning identity</h3>
      <p>Lightning state is not disposable. Kiln retains node storage, uses controlled StatefulSet updates, and reuses persisted wallet state across pod and controller restarts.</p>
    </article>
    <article class="feature-card">
      <span class="feature-index">03</span>
      <h3>Publish safer RPC access</h3>
      <p>Managed client credentials expose restricted read-only and invoice macaroons with TLS material, while deliberately keeping the admin macaroon private.</p>
    </article>
    <article class="feature-card">
      <span class="feature-index">04</span>
      <h3>Compose node dependencies</h3>
      <p>A LightningNode can reference a BitcoinNode directly. Kiln waits for the Bitcoin backend to become ready, then derives the in-cluster connection details and credentials.</p>
    </article>
  </div>
</section>

<section class="architecture">
  <div class="section-kicker">The shape</div>
  <div class="arch-grid">
    <div class="arch-copy">
      <h2>Small API surface. Stateful consequences.</h2>
      <p>The operator keeps the user-facing model compact while handling the details that matter underneath: service discovery, storage fencing, wallet initialization, TLS trust, RPC credentials, readiness, and graceful shutdown.</p>
      <a class="text-link" href="https://github.com/kiln-fired/kiln-operator/blob/main/README.md">Read the current architecture notes <span>→</span></a>
    </div>
    <div class="stack-diagram">
      <div class="stack-node orange">LightningNode <small>LND</small></div>
      <div class="stack-connector"><i></i><span>references</span><i></i></div>
      <div class="stack-node">BitcoinNode <small>btcd</small></div>
      <div class="stack-connector"><i></i><span>reconciled by</span><i></i></div>
      <div class="stack-node muted">Kiln Operator <small>Kubernetes</small></div>
    </div>
  </div>
</section>

<section class="section history" id="field-notes">
  <div class="section-kicker">Field notes</div>
  <div class="section-heading">
    <h2>Built in public, then fired again.</h2>
    <p>These notes capture the original 2022 experiments that shaped Kiln: simulated Bitcoin networks, operator reconciliation, node APIs, mining behavior, and the path toward self-contained Lightning testing.</p>
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
    <h2>See what’s burning now.</h2>
    <p>Kiln is actively being modernized around a current Kubernetes operator stack and current Bitcoin/Lightning dependencies.</p>
  </div>
  <a class="button primary" href="https://github.com/kiln-fired/kiln-operator">github.com/kiln-fired/kiln-operator <span>↗</span></a>
</section>
