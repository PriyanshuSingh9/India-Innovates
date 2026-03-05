# Design System & UX

> Covers: F38 BIE Design Language · F39 Mobile Optimization

---

## F38 — BIE Design Language

### What Is Being Built

A cohesive visual design system that defines BIE's look and feel across every component: color palette, typography, spacing, component styles, iconography, animation patterns, and dark-mode theming. This ensures every panel, button, badge, and chart looks like it belongs to the same product.

### Why It Is Needed

BIE has 6+ UI components (map, news feed, query bar, KG panel, posture panel, IOR dashboard) built by two frontend engineers. Without a shared design language:
- Each panel looks like a different product
- Color meanings are inconsistent (is red = danger or red = military?)
- Typography varies between components
- The demo looks amateurish instead of polished

A cohesive design language is what separates a hackathon prototype from a product that makes judges say "this looks like it costs $50M."

### What Use It Provides

- **Consistent visual identity** across all components
- **Semantic color system** — colors mean specific things everywhere
- **Dark-mode first** — designed for the dark map background
- **Professional polish** — the demo looks world-class
- **Developer speed** — engineers use design tokens instead of picking colors per component

### How It Is Built

1. **Color palette** (CSS custom properties):
   ```css
   :root {
     /* Core */
     --bie-bg-primary: #0a0a1a;         /* Deep navy, almost black */
     --bie-bg-secondary: #111128;        /* Panel backgrounds */
     --bie-bg-tertiary: #1a1a3e;         /* Card backgrounds */
     --bie-text-primary: #e8e8f0;        /* Main text */
     --bie-text-secondary: #8888aa;      /* Muted text */
     --bie-accent: #00d4ff;              /* Cyan accent — primary action */
     --bie-accent-hover: #33ddff;

     /* Threat levels */
     --bie-threat-low: #2ecc71;          /* Green */
     --bie-threat-medium: #f1c40f;       /* Yellow */
     --bie-threat-high: #e74c3c;         /* Red */
     --bie-threat-critical: #c0392b;     /* Dark red */

     /* Categories (7-category system) */
     --bie-cat-military: #e74c3c;        /* Red */
     --bie-cat-diplomatic: #3498db;      /* Blue */
     --bie-cat-economic: #2ecc71;        /* Green */
     --bie-cat-internal: #e67e22;        /* Orange */
     --bie-cat-maritime: #1abc9c;        /* Teal */
     --bie-cat-cyber: #9b59b6;           /* Purple */
     --bie-cat-terrorism: #8b1a1a;       /* Dark Red */

     /* Posture */
     --bie-posture-normal: #2ecc71;
     --bie-posture-elevated: #f1c40f;
     --bie-posture-high: #e74c3c;
     --bie-posture-critical: #c0392b;

     /* Borders and surfaces */
     --bie-border: rgba(255, 255, 255, 0.08);
     --bie-glass: rgba(17, 17, 40, 0.85);
     --bie-shadow: 0 4px 30px rgba(0, 0, 0, 0.5);
   }
   ```

2. **Typography**:
   - Primary font: `Inter` (Google Fonts) — clean, readable, tech-forward
   - Monospace: `JetBrains Mono` — for data, IDs, timestamps
   - Scale: 12px (caption), 14px (body), 16px (subtitle), 20px (title), 28px (header)

3. **Component patterns**:
   - **Panels**: glassmorphism (`backdrop-filter: blur(10px)`) with dark glass background
   - **Cards**: `--bie-bg-tertiary` with subtle border, 8px radius
   - **Badges**: pill-shaped, color-coded by category/threat
   - **Buttons**: minimal, accent-colored for primary actions
   - **Animations**: 200ms ease-out for transitions, pulse animation for CRITICAL alerts

4. **Glassmorphism panels**:
   ```css
   .bie-panel {
     background: var(--bie-glass);
     backdrop-filter: blur(12px);
     border: 1px solid var(--bie-border);
     border-radius: 12px;
     box-shadow: var(--bie-shadow);
   }
   ```

5. **Implementation**: design tokens live in `frontend/app/globals.css` as CSS custom properties. All components reference tokens, never hardcoded colors.

---

## F39 — Mobile Optimization

### What Is Being Built

Responsive layout adaptations that make BIE usable on tablet and mobile devices. While the primary experience is desktop (analysts at workstations), the mobile version ensures the demo can be shown on any screen and field analysts can access BIE on tablets.

### Why It Is Needed

- Demo hardware unpredictability — the venue might provide a small screen
- Judges may pull up BIE on their phones during evaluation
- Field analysts may need tablet access
- A responsive design signals product maturity

### What Use It Provides

- **Tablet (768px–1024px)**: side panels collapse to bottom sheets; map stays full-width
- **Mobile (< 768px)**: panels become full-screen overlays; map fills the background
- **Touch interactions**: pinch-zoom on map, swipe to open/close panels
- **Performance**: disable heatmap and limit markers on mobile to maintain framerates

### How It Is Built

1. **Responsive breakpoints**:
   ```css
   /* Mobile first */
   .bie-layout { display: flex; flex-direction: column; }

   @media (min-width: 768px) {
     .bie-layout { flex-direction: row; }
     .bie-news-panel { width: 380px; }
   }

   @media (min-width: 1200px) {
     .bie-right-panel { width: 400px; }
   }
   ```

2. **Panel behavior**:
   - Desktop: fixed side panels
   - Tablet: collapsible bottom sheets (swipe up to expand)
   - Mobile: full-screen overlays with back button

3. **Map adaptation**: on mobile, map fills 100% viewport. Controls moved to bottom for thumb reach.

4. **Performance**: detect `navigator.hardwareConcurrency < 4` and disable HeatmapLayer, cap markers at 200.

---

← Back to [prd.md](./prd.md)
