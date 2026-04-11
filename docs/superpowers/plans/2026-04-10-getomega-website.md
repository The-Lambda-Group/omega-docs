# getomega.ai Website Rebuild — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rebuild getomega.ai as a static HTML/CSS sell sheet that positions OmegaAI as an AI automation platform for business workflows. Pure benefits, no code, no CLI. Replaces the current Framer site.

**Architecture:** Two-page static site (index + pricing). Pure HTML/CSS/minimal JS. No framework, no build step. Hosted on Kubernetes.

**Tech Stack:** HTML5, CSS3 (custom properties, grid, flexbox), vanilla JS (scroll animations, mobile nav), Google Fonts.

**Brand:** Primary `#e84bfa` (magenta), secondary `#351c75` (deep purple), dark theme, Omega symbol (Ω) logo.

**Design skill:** Apply `frontend-design plugin SKILL.md` — distinctive typography, bold palette, atmospheric effects, no generic AI aesthetics.

**Visual system:** Wireframe/blueprint-style SVG illustrations of astronomical phenomena — wormholes, black holes, stars, planets, nebulae, gravitational lensing. Drawn in the style of a physicist's sketch: thin stroke lines, grid overlays, orbital paths, coordinate annotations. All rendered in magenta (`#e84bfa`) and purple (`#351c75`) strokes on dark backgrounds. The Omega symbol (Ω) is reimagined as a stargate — a portal shape that echoes the Ω letterform. These illustrations parallax-scroll behind content sections, creating depth and atmosphere. No photorealistic renders, no stock imagery — everything looks hand-drafted by an engineer.

**Voice:** Business-first. Benefits only. No code snippets, no CLI examples, no technical jargon. Sell outcomes.

---

## Site Structure

```
site/
  index.html          <- main sell sheet
  pricing.html        <- dedicated pricing page
  css/
    style.css         <- all styles, CSS custom properties
    animations.css    <- scroll-triggered animations, parallax
  js/
    main.js           <- intersection observer, mobile nav, parallax scroll
  assets/
    omega-stargate.svg    <- Ω as stargate portal (hero centerpiece)
    wormhole.svg          <- wireframe wormhole tunnel (pain/solution bg)
    black-hole.svg        <- wireframe black hole with accretion disk (AI section bg)
    orbital-system.svg    <- planetary orbits / gravitational paths (how-it-works bg)
    star-field.svg        <- scattered wireframe stars and constellations (marketplace bg)
    gravitational-lens.svg <- light-bending grid distortion (built-different bg)
```

## Page Sections — index.html (top to bottom)

1. **Nav** — logo + links (How It Works, Marketplace, Pricing)
2. **Hero** — "AI agents that run your business workflows" + CTA
3. **Pain / Solution** — business automation is broken → OmegaAI fixes it
4. **How It Works** — three steps: Install, AI takes over, see results
5. **AI-First Platform** — 100% operated by AI, designed from the ground up for agents, fully discoverable
6. **Marketplace** — "Automate. Monetize. Scale." — every automation is a distributable plugin
7. **Built Different** — tech credibility (logic database, inspectable, resilient) — positioned last
8. **CTA Banner** — final push to get started
9. **Footer** — links, company, contact

---

## Tasks

### Task 1: Project scaffold and base HTML

**Files:**
- Create: `site/index.html`
- Create: `site/pricing.html`
- Create: `site/css/style.css`
- Create: `site/css/animations.css`
- Create: `site/js/main.js`

- [ ] **Step 1: Create directory structure**

```bash
mkdir -p site/css site/js site/assets
```

- [ ] **Step 2: Create index.html with semantic structure**

All section containers, no content yet. Include:
- `<!DOCTYPE html>`, charset, viewport meta
- Open Graph meta tags (title: "OmegaAI — AI Agents That Run Your Business", description: "Install automation plugins. AI agents operate them 24/7. No code, no configuration, just outcomes.")
- Google Fonts — choose distinctive display + body fonts per frontend-design skill (NOT Inter, Roboto, Space Grotesk, or Arial)
- CSS and JS links
- Semantic sections with IDs for nav anchors

- [ ] **Step 3: Create style.css with brand tokens and reset**

CSS custom properties:
- `--magenta: #e84bfa`
- `--purple-deep: #351c75`
- `--bg-primary: #0a0a0f`
- `--bg-card: #111118`
- `--bg-elevated: #16161f`
- `--text-primary: #e8e6ef`
- `--text-secondary: #8a879c`
- `--text-muted: #5c596e`

Reset, base typography, responsive container (max-width 1200px).

- [ ] **Step 4: Create empty animations.css and main.js placeholders**

- [ ] **Step 5: Create pricing.html skeleton**

Same nav/footer as index, content area for pricing cards. Links back to index.

- [ ] **Step 6: Test and commit**

```bash
git add site/ && git commit -m "feat: scaffold getomega.ai site"
```

---

### Task 2: SVG illustrations — wireframe space phenomena

**Files:**
- Create: `site/assets/omega-stargate.svg`
- Create: `site/assets/wormhole.svg`
- Create: `site/assets/black-hole.svg`
- Create: `site/assets/orbital-system.svg`
- Create: `site/assets/star-field.svg`
- Create: `site/assets/gravitational-lens.svg`

All SVGs share a consistent visual language:

**Style rules:**
- Thin stroke lines (1-2px), no fills
- Primary stroke: `#e84bfa` (magenta) at varying opacities (0.15–0.6)
- Secondary stroke: `#351c75` (deep purple) at low opacity (0.1–0.3)
- Grid/coordinate overlays where appropriate — like graph paper behind the illustration
- Dashed lines for orbital paths and trajectories
- Small annotation marks (tick marks, degree indicators, focal points) like physics diagrams
- Viewbox sized for full-section backgrounds (1440x900 or similar)
- All paths use `vector-effect="non-scaling-stroke"` for consistent line weight

- [ ] **Step 1: Create omega-stargate.svg**

The centerpiece. An Ω shape reimagined as a circular portal/stargate:
- Outer ring with tick marks like a clock or compass
- Inner Ω shape formed by the portal opening
- Radiating lines suggesting energy/light emanating from the gate
- Subtle concentric circles inside suggesting depth/tunnel
- Coordinate grid visible behind it

- [ ] **Step 2: Create wormhole.svg**

A wireframe Einstein-Rosen bridge:
- Two funnel/trumpet shapes connected at narrow throat
- Grid mesh lines following the curved surface (like a 3D wireframe render)
- Coordinate axes showing the spatial distortion
- Dashed trajectory line showing a path through

- [ ] **Step 3: Create black-hole.svg**

Wireframe black hole with accretion disk:
- Central circle (event horizon) with dashed boundary
- Elliptical accretion disk rendered as concentric orbital rings
- Curved light paths showing gravitational lensing (arcing lines bending around center)
- Photon sphere as a dotted circle

- [ ] **Step 4: Create orbital-system.svg**

Planetary orbital diagram:
- Central body (small circle)
- 3-4 elliptical orbits at different inclinations
- Small circles on orbits representing bodies
- Dashed lines showing gravitational connections
- Velocity vectors (small arrows tangent to orbits)

- [ ] **Step 5: Create star-field.svg**

Scattered constellation-style star map:
- Points of varying sizes (small circles, 1-4px)
- Thin lines connecting some stars into constellation patterns
- A few larger stars with subtle cross/diffraction spikes
- Coordinate grid very faintly in background

- [ ] **Step 6: Create gravitational-lens.svg**

Spacetime distortion grid:
- Regular grid that warps/distorts around a central mass
- Grid lines curve inward toward center like a gravity well
- Clean geometric distortion — mathematically plausible
- Faint radial lines from center

- [ ] **Step 7: Commit**

```bash
git add site/assets/ && git commit -m "feat: add wireframe space SVG illustrations"
```

---

### Task 3: Navigation bar

**Files:**
- Modify: `site/index.html`
- Modify: `site/pricing.html`
- Modify: `site/css/style.css`

- [ ] **Step 1: Add nav HTML to both pages**

```html
<nav class="nav" id="nav">
  <div class="nav-container">
    <a href="/" class="nav-logo">
      <span class="nav-logo-symbol">&Omega;</span>
      <span class="nav-logo-text">OmegaAI</span>
    </a>
    <div class="nav-links" id="nav-links">
      <a href="/#how-it-works">How It Works</a>
      <a href="/#marketplace">Marketplace</a>
      <a href="/pricing.html">Pricing</a>
    </div>
    <button class="nav-toggle" id="nav-toggle" aria-label="Toggle navigation">
      <span></span><span></span><span></span>
    </button>
  </div>
</nav>
```

- [ ] **Step 2: Style nav**

Fixed position, backdrop blur, transparent → opaque on scroll. Logo Ω in magenta. Links muted → magenta on hover. Mobile hamburger.

- [ ] **Step 3: Test and commit**

```bash
git add site/ && git commit -m "feat: add navigation bar"
```

---

### Task 3: Hero section

**Files:**
- Modify: `site/index.html`
- Modify: `site/css/style.css`

- [ ] **Step 1: Write hero copy**

```html
<header class="hero" id="hero">
  <div class="hero-container">
    <div class="hero-badge">The Generative Operating System</div>
    <h1 class="hero-title">
      AI agents that run<br>
      your business workflows
    </h1>
    <p class="hero-subtitle">
      Install automation plugins from the marketplace.
      AI agents operate them 24/7 — managing campaigns, syncing data,
      scoring leads, and reporting results.
      No code. No configuration. Just outcomes.
    </p>
    <div class="hero-actions">
      <a href="/pricing.html" class="btn btn-primary">Get Started</a>
      <a href="#how-it-works" class="btn btn-secondary">See How It Works</a>
    </div>
  </div>
</header>
```

- [ ] **Step 2: Style hero**

Full viewport height, centered. Large bold display font for title. Radial magenta glow behind title. Badge: uppercase pill with magenta border. CTAs: primary (magenta fill), secondary (outline). Atmospheric background — noise or mesh gradient. No code blocks.

- [ ] **Step 3: Test and commit**

```bash
git add site/ && git commit -m "feat: add hero section"
```

---

### Task 4: Pain / Solution section

**Files:**
- Modify: `site/index.html`
- Modify: `site/css/style.css`

- [ ] **Step 1: Write pain/solution copy**

```html
<section class="section" id="problem">
  <div class="section-container">
    <h2 class="section-title">Business automation is broken</h2>
    <div class="pain-solution">
      <div class="pain">
        <h3>The old way</h3>
        <div class="pain-item">
          <span class="pain-icon">&times;</span>
          <p>Paying engineers to glue APIs together with fragile scripts that break when anything changes</p>
        </div>
        <div class="pain-item">
          <span class="pain-icon">&times;</span>
          <p>Workflows scattered across Zapier, spreadsheets, custom code, and someone's laptop</p>
        </div>
        <div class="pain-item">
          <span class="pain-icon">&times;</span>
          <p>When something breaks, nobody knows where to look — state is hidden across dozens of services</p>
        </div>
      </div>
      <div class="solution">
        <h3>The OmegaAI way</h3>
        <div class="solution-item">
          <span class="solution-icon">&check;</span>
          <p>Install automations from a marketplace — no engineering required</p>
        </div>
        <div class="solution-item">
          <span class="solution-icon">&check;</span>
          <p>AI agents operate everything 24/7, making decisions and adapting in real time</p>
        </div>
        <div class="solution-item">
          <span class="solution-icon">&check;</span>
          <p>Full transparency — see exactly what's happening, when, and why</p>
        </div>
      </div>
    </div>
  </div>
</section>
```

- [ ] **Step 2: Style pain/solution**

Two-column: pain (muted/red X icons) vs solution (magenta check icons). Visual contrast. Responsive stacking.

- [ ] **Step 3: Test and commit**

```bash
git add site/ && git commit -m "feat: add pain/solution section"
```

---

### Task 5: "How It Works" — three steps

**Files:**
- Modify: `site/index.html`
- Modify: `site/css/style.css`

- [ ] **Step 1: Write three-step walkthrough**

```html
<section class="section section-dark" id="how-it-works">
  <div class="section-container">
    <h2 class="section-title">How it works</h2>
    <p class="section-subtitle">Three steps. No engineering team required.</p>
    <div class="steps">
      <div class="step">
        <div class="step-number">1</div>
        <h3>Install a plugin</h3>
        <p>Browse the marketplace for the automation you need — lead generation, CRM sync, email campaigns, data pipelines. One click to install.</p>
      </div>
      <div class="step">
        <div class="step-number">2</div>
        <h3>AI takes over</h3>
        <p>An AI agent connects to your workspace and runs the plugin autonomously. It manages campaigns, monitors performance, handles errors, and adapts — around the clock.</p>
      </div>
      <div class="step">
        <div class="step-number">3</div>
        <h3>You see results</h3>
        <p>Real-time visibility into everything. Leads scored, contacts synced, emails sent, revenue generated. Full transparency, zero black boxes.</p>
      </div>
    </div>
  </div>
</section>
```

- [ ] **Step 2: Style steps**

Horizontal three-column with connecting lines. Step numbers: large magenta circles. Darker background. Mobile: vertical stack with connector line.

- [ ] **Step 3: Test and commit**

```bash
git add site/ && git commit -m "feat: add how-it-works section"
```

---

### Task 6: AI-First Platform section

**Files:**
- Modify: `site/index.html`
- Modify: `site/css/style.css`

- [ ] **Step 1: Write AI-first section**

Position OmegaAI as the first operating system built for AI. No MCP mentions — pure benefits.

```html
<section class="section" id="ai-first">
  <div class="section-container">
    <h2 class="section-title">The first operating system built for AI</h2>
    <p class="section-subtitle">OmegaAI isn't retrofitted for agents. It was designed from the ground up to be operated 100% by AI.</p>

    <div class="ai-grid">
      <div class="ai-card">
        <h3>100% agentic</h3>
        <p>Every workflow, every decision, every action — handled by AI agents. They don't just assist. They operate. Provision workspaces, run campaigns, monitor systems, and adapt in real time.</p>
      </div>
      <div class="ai-card">
        <h3>Fully navigable</h3>
        <p>The entire system is discoverable by AI from the inside. Every piece of data, every workflow, every configuration — the agent can find it, understand it, and act on it without being told where to look.</p>
      </div>
      <div class="ai-card">
        <h3>Always on, always learning</h3>
        <p>Agents operate around the clock. They don't take breaks, miss emails, or forget to follow up. Every decision is logged. Every outcome feeds back into better performance.</p>
      </div>
      <div class="ai-card">
        <h3>Bring any model</h3>
        <p>Not locked to one AI provider. Use the best model for each task — swap providers without changing a single workflow. Your automations are model-independent.</p>
      </div>
    </div>
  </div>
</section>
```

- [ ] **Step 2: Style AI section**

2x2 card grid. Elevated dark cards, subtle magenta glow on hover. No code anywhere.

- [ ] **Step 3: Test and commit**

```bash
git add site/ && git commit -m "feat: add AI-first platform section"
```

---

### Task 7: Marketplace section — "Automate. Monetize. Scale."

**Files:**
- Modify: `site/index.html`
- Modify: `site/css/style.css`

- [ ] **Step 1: Write marketplace section**

```html
<section class="section section-dark" id="marketplace">
  <div class="section-container">
    <div class="marketplace-header">
      <h2 class="section-title">Automate. Monetize. Scale.</h2>
      <p class="section-subtitle">Every automation you build is a plugin others can install. Turn your workflows into a product.</p>
    </div>

    <div class="marketplace-grid">
      <div class="marketplace-card">
        <h3>Build once, sell everywhere</h3>
        <p>Built a lead scoring pipeline? A CRM sync? An email campaign engine? Package it as a plugin and list it on the marketplace. Other businesses install it with one click.</p>
      </div>
      <div class="marketplace-card">
        <h3>Your automation, your revenue</h3>
        <p>Set your price. Customers subscribe. You earn 85% of every dollar. The marketplace handles distribution, billing, and updates — you focus on building great automations.</p>
      </div>
      <div class="marketplace-card">
        <h3>Composable by design</h3>
        <p>Plugins snap together like building blocks. A data connector feeds a scoring engine feeds a CRM sync. Mix and match — yours, third-party, or built by AI.</p>
      </div>
    </div>
  </div>
</section>
```

- [ ] **Step 2: Style marketplace**

Three-column card grid. Dark background. Cards with subtle border, magenta accent on hover. Focus on the tagline being large and memorable.

- [ ] **Step 3: Test and commit**

```bash
git add site/ && git commit -m "feat: add marketplace section"
```

---

### Task 8: "Built Different" — tech credibility

**Files:**
- Modify: `site/index.html`
- Modify: `site/css/style.css`

- [ ] **Step 1: Write tech credibility section**

Positioned late. Earns trust without leading the pitch. Still benefits-focused, but nods to the technology.

```html
<section class="section" id="built-different">
  <div class="section-container">
    <h2 class="section-title">Enterprise-grade from day one</h2>
    <p class="section-subtitle">The infrastructure behind Fortune 500 workflows — without the Fortune 500 budget.</p>

    <div class="tech-grid">
      <div class="tech-card">
        <h3>Real-time analytics</h3>
        <p>Every workflow generates live, queryable data. Monitor campaign performance, lead conversion, pipeline throughput, and operational health — all in real time, all in one place.</p>
      </div>
      <div class="tech-card">
        <h3>Distributed data pipelines</h3>
        <p>Process hundreds of thousands of records across systems that stay in sync automatically. Your data flows continuously between CRMs, marketing platforms, and analytics — no manual imports, no stale spreadsheets.</p>
      </div>
      <div class="tech-card">
        <h3>Complete audit trail</h3>
        <p>Every action, every decision, every data change is logged and traceable. When a client asks "what happened with that lead?" — you have the answer in seconds, not hours.</p>
      </div>
      <div class="tech-card">
        <h3>Scales with your business</h3>
        <p>Start with one workspace. Add clients, add automations, add channels. The platform grows with you — from solo operator to managing dozens of accounts, without rebuilding anything.</p>
      </div>
    </div>
  </div>
</section>
```

- [ ] **Step 2: Style enterprise section**

2x2 card grid. Professional, confident — conveys reliability and scale without being technical.

- [ ] **Step 3: Test and commit**

```bash
git add site/ && git commit -m "feat: add built-different section"
```

---

### Task 9: CTA banner

**Files:**
- Modify: `site/index.html`
- Modify: `site/css/style.css`

- [ ] **Step 1: Add final CTA section**

```html
<section class="cta-banner">
  <div class="section-container">
    <h2>Ready to put your business on autopilot?</h2>
    <p>Start with a shared workspace for $30/month. Your AI agent is waiting.</p>
    <a href="/pricing.html" class="btn btn-primary btn-large">Get Started</a>
  </div>
</section>
```

- [ ] **Step 2: Style CTA banner**

Full-width, magenta gradient background or deep purple with magenta accent. Large centered text. Prominent button. High contrast — this is the final conversion moment.

- [ ] **Step 3: Test and commit**

```bash
git add site/ && git commit -m "feat: add CTA banner"
```

---

### Task 10: Footer

**Files:**
- Modify: `site/index.html`
- Modify: `site/pricing.html`
- Modify: `site/css/style.css`

- [ ] **Step 1: Add footer HTML to both pages**

```html
<footer class="footer">
  <div class="footer-container">
    <div class="footer-brand">
      <span class="nav-logo-symbol">&Omega;</span>
      <span class="nav-logo-text">OmegaAI</span>
      <p class="footer-tagline">The Generative Operating System</p>
    </div>
    <div class="footer-links">
      <div class="footer-col">
        <h4>Platform</h4>
        <a href="/#how-it-works">How It Works</a>
        <a href="/#marketplace">Marketplace</a>
        <a href="/pricing.html">Pricing</a>
      </div>
      <div class="footer-col">
        <h4>Resources</h4>
        <a href="https://github.com/The-Lambda-Group" target="_blank">GitHub</a>
        <a href="https://github.com/The-Lambda-Group/omega-docs" target="_blank">Docs</a>
      </div>
      <div class="footer-col">
        <h4>Company</h4>
        <p>OmegaDB Inc.</p>
        <a href="mailto:hello@getomega.ai">hello@getomega.ai</a>
      </div>
    </div>
    <div class="footer-bottom">
      <p>&copy; 2026 OmegaDB Inc.</p>
    </div>
  </div>
</footer>
```

- [ ] **Step 2: Style footer and commit**

```bash
git add site/ && git commit -m "feat: add footer"
```

---

### Task 11: Pricing page

**Files:**
- Modify: `site/pricing.html`
- Modify: `site/css/style.css`

- [ ] **Step 1: Build pricing page content**

Adapt from `pricing/omegaai-pricing-sheet.html`. Three pricing cards: Shared Compute ($30), Dedicated Compute (from $60), AI Agent Tokens ($10/1M input). Plus marketplace fee (15%). Business-friendly descriptions — no instance type jargon visible, put technical specs in small print.

```html
<section class="section" id="pricing-content">
  <div class="section-container">
    <h1 class="page-title">Simple, transparent pricing</h1>
    <p class="section-subtitle">Start small. Scale when you're ready.</p>

    <div class="pricing-grid">
      <div class="pricing-card">
        <h3>Starter</h3>
        <div class="pricing-price">
          <span class="pricing-amount">$30</span>
          <span class="pricing-period">/month</span>
        </div>
        <p class="pricing-desc">Shared workspace — perfect for getting started</p>
        <ul class="pricing-features">
          <li>AI agent workspace</li>
          <li>Install marketplace plugins</li>
          <li>Full data visibility</li>
          <li>Platform updates included</li>
        </ul>
        <p class="pricing-specs">1 vCPU &middot; 2 GB RAM &middot; 20 GB storage</p>
        <a href="#" class="btn btn-primary">Get Started</a>
      </div>

      <div class="pricing-card pricing-featured">
        <div class="pricing-badge">Most Popular</div>
        <h3>Dedicated</h3>
        <p class="pricing-from">Starting at</p>
        <div class="pricing-price">
          <span class="pricing-amount">$60</span>
          <span class="pricing-period">/month</span>
        </div>
        <p class="pricing-desc">Isolated resources for production workloads</p>
        <ul class="pricing-features">
          <li>Everything in Starter</li>
          <li>Dedicated compute resources</li>
          <li>Production-grade performance</li>
          <li>Priority support</li>
        </ul>
        <p class="pricing-specs">2–8 vCPU &middot; 4–16 GB RAM &middot; up to 250 GB</p>
        <a href="mailto:hello@getomega.ai" class="btn btn-primary">Contact Us</a>
      </div>

      <div class="pricing-card">
        <h3>AI Agent Usage</h3>
        <div class="pricing-price">
          <span class="pricing-amount">Metered</span>
        </div>
        <p class="pricing-desc">Pay for what your agents use</p>
        <ul class="pricing-features">
          <li>$10 per 1M input tokens</li>
          <li>$50 per 1M output tokens</li>
          <li>Any model supported</li>
          <li>Billed monthly</li>
        </ul>
        <a href="mailto:hello@getomega.ai" class="btn btn-secondary">Learn More</a>
      </div>
    </div>

    <div class="pricing-marketplace">
      <h3>Marketplace</h3>
      <p>Distribute your plugins and earn <strong>85%</strong> of every sale. We handle billing, distribution, and updates.</p>
    </div>
  </div>
</section>
```

- [ ] **Step 2: Style pricing page and commit**

```bash
git add site/ && git commit -m "feat: add pricing page"
```

---

### Task 13: Scroll animations, parallax, and mobile nav

**Files:**
- Modify: `site/js/main.js`
- Modify: `site/css/animations.css`

- [ ] **Step 1: Add intersection observer, parallax, and mobile nav JS**

```javascript
// Scroll reveal
const observer = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) entry.target.classList.add('visible');
  });
}, { threshold: 0.1 });

document.querySelectorAll('.section, .step, .ai-card, .marketplace-card, .tech-card, .pricing-card, .pain-item, .solution-item').forEach(el => {
  observer.observe(el);
});

// Parallax scroll for SVG backgrounds
// Each section with a .parallax-bg child scrolls the SVG at a slower rate
window.addEventListener('scroll', () => {
  const scrollY = window.scrollY;

  document.querySelectorAll('.parallax-bg').forEach(bg => {
    const section = bg.parentElement;
    const sectionTop = section.offsetTop;
    const offset = (scrollY - sectionTop) * 0.3; // 30% scroll speed
    bg.style.transform = `translateY(${offset}px)`;
  });

  // Nav scroll effect
  const nav = document.getElementById('nav');
  nav.classList.toggle('scrolled', scrollY > 50);
});

// Mobile nav
const toggle = document.getElementById('nav-toggle');
const links = document.getElementById('nav-links');
if (toggle) {
  toggle.addEventListener('click', () => {
    links.classList.toggle('open');
    toggle.classList.toggle('open');
  });
}
```

Each content section includes a parallax background element:
```html
<section class="section" id="example">
  <div class="parallax-bg">
    <img src="assets/wormhole.svg" alt="" aria-hidden="true">
  </div>
  <div class="section-container">
    <!-- content -->
  </div>
</section>
```

**SVG placement per section:**
- **Hero**: `omega-stargate.svg` — centered behind hero title, large, prominent
- **Pain/Solution**: `wormhole.svg` — subtle, offset right
- **How It Works**: `orbital-system.svg` — behind the three steps
- **AI-First**: `black-hole.svg` — centered, dramatic
- **Marketplace**: `star-field.svg` — scattered across background
- **Built Different**: `gravitational-lens.svg` — subtle grid distortion
- **CTA**: `omega-stargate.svg` reused — faint, pulling you in

- [ ] **Step 2: Add animation CSS with parallax positioning and prefers-reduced-motion**

```css
.parallax-bg {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  overflow: hidden;
  pointer-events: none;
  z-index: 0;
}

.parallax-bg img {
  width: 100%;
  height: 120%; /* extra height for scroll travel */
  object-fit: contain;
  opacity: 0.15;
}

/* Sections need relative positioning for parallax */
.section, .hero, .cta-banner {
  position: relative;
  overflow: hidden;
}

.section-container {
  position: relative;
  z-index: 1;
}

@media (prefers-reduced-motion: reduce) {
  .parallax-bg { display: none; }
}
```

- [ ] **Step 3: Test and commit**

```bash
git add site/ && git commit -m "feat: add scroll animations and mobile nav"
```

---

### Task 14: Final polish and responsive testing

**Files:**
- Modify: `site/css/style.css`

- [ ] **Step 1: Responsive breakpoints**

Desktop 1200px+, tablet 768-1199px, mobile <768px. All grids collapse gracefully.

- [ ] **Step 2: Atmospheric details**

Noise texture on dark backgrounds, radial magenta glows behind hero and CTA, smooth gradient transitions between sections.

- [ ] **Step 3: Full test both pages**

Open index and pricing in browser. Test all breakpoints. Verify links between pages work. Console clean.

- [ ] **Step 4: Final commit**

```bash
git add site/ && git commit -m "feat: responsive polish and atmospheric details"
```

---

## Copy Principles

- **No code, no CLI, no technical jargon** — this is a sell sheet
- **Benefits over features** — "always up" not "multi-master replication"
- **Three-second test** — every section headline must communicate value instantly
- **Business buyer first** — technical credibility earned late, never led with
- **Action-oriented** — every section ends with a reason to keep scrolling or click a button
