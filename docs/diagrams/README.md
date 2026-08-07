# Architecture diagrams

Self-contained HTML pages — open them directly in a browser, no build step or
server needed. They render in light and dark mode and the topology diagrams
are zoomable (Ctrl/Cmd + wheel, drag to pan, ⛶ for full size).

| Page | What it shows |
|------|---------------|
| [`platform-factory-overall-architecture.html`](platform-factory-overall-architecture.html) | The whole pattern: seven-repo topology, the flow of a change (PR → CODEOWNERS tier → merge → Argo CD → Kyverno → Crossplane → cloud), three approval tiers, the autonomy ladder, build-milestone status. |
| [`gcp-platform-factory-architecture.html`](gcp-platform-factory-architecture.html) | The layer-0 deep dive: `platform-bootstrap`'s four Terraform layers, persist-vs-teardown boundaries, the session teardown rhythm (claim C-02), the Artifact Registry image plane and its single egress pinhole (ADR-0010, claim C-23), the HA VPN to a peer network, and where Terraform's job ends (claim C-01). |

The two pages cross-link: the overall page is *what the factory is*; the
layer-0 page is *the floor it stands on*. Diagrams are point-in-time snapshots
of the design — the source of truth remains `docs/design/` and the ADRs.

Note: GitHub renders HTML files as source. To view them from a clone, open the
files locally; they are fully self-contained except for Google Fonts and the
Mermaid CDN import.

For GitHub browsing, static PNG snapshots of both topology diagrams sit next
to the HTML (`overall-architecture.png`, `gcp-layer0-architecture.png`) and
are embedded in the repo's root README. The HTML pages are canonical; the
PNGs are regenerated from the same Mermaid sources when the topology changes.
