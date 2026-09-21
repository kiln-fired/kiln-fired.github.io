---
layout: default
title: About
permalink: /about/
---

<section class="page-hero narrow">
  <div class="eyebrow"><span></span> About Kiln</div>
  <h1>Bitcoin infrastructure, expressed as desired state.</h1>
  <p class="lede">Kiln is an open source Kubernetes operator for managing Bitcoin and Lightning node resources. It began as an experiment in 2022 and is now being modernized with a sharper focus on lifecycle, recovery, identity, and safe RPC access.</p>
</section>

<section class="prose-shell">
  <h2>Why “Kiln”?</h2>
  <p>A kiln takes raw material, applies controlled heat, and turns it into something durable. The project name fits the operator model: continuously apply a controlled process until infrastructure reaches the state you intended.</p>

  <h2>What exists today</h2>
  <p>The project provides <code>BitcoinNode</code>, <code>LightningNode</code>, and <code>Seed</code> APIs. The current Lightning baseline uses upstream LND with <code>lndinit</code> for idempotent wallet initialization. Node storage and credentials are treated as durable infrastructure rather than disposable pod state.</p>

  <h2>Project history</h2>
  <p>The first Kiln experiments explored whether Kubernetes reconciliation could make private Bitcoin and Lightning infrastructure easier to operate. The original field notes were published on OpenPersuasion in 2022 and are preserved here as the project’s technical origin story.</p>

  <p><a class="text-link" href="https://github.com/kiln-fired/kiln-operator">Explore the source on GitHub <span>→</span></a></p>
</section>
