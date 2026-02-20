-----  
  
## name: lovefrom-design  
description: Create frontend interfaces guided by the Jony Ive / LoveFrom design philosophy — craft-forward, materially honest, physics-alive, and quietly extraordinary. Use this skill when building dashboards, data tools, mobile-first components, or immersive fullscreen experiences that should feel like they were made by hand, with intention, and with deep respect for the person using them.  
  
This skill guides the creation of frontend interfaces rooted in the Jony Ive / LoveFrom design philosophy. Every decision — from spacing to spring curves to the weight of a shadow — should feel considered, not generated. These are interfaces that earn trust through material honesty, restraint of color, and motion that behaves like the physical world.  
  
The user provides a component, dashboard, tool, or immersive experience to build. They may specify purpose, audience, or technical constraints.  
  
## The LoveFrom Philosophy  
  
Before writing a single line of code, internalize this: **Jony Ive and LoveFrom believe that the best design is invisible in its effort but unmistakable in its quality.** Every surface, every transition, every typographic choice should feel as though it could not have been any other way.  
  
Ask yourself before committing to any design decision:  
  
- **Does this feel made, or generated?** LoveFrom work feels hand-considered. Avoid anything that could have come from a template.  
- **What is the material?** Frosted glass, warm linen, brushed aluminum — every UI has a material. Know yours and honor it consistently.  
- **Does this deserve to exist?** Remove anything that doesn’t earn its place. Restraint is the most demanding form of confidence.  
- **What does the object weigh?** Motion must reflect mass. A modal has weight. A tooltip is light. A full-screen transition is monumental.  
  
## Design Thinking Process  
  
**Step 1 — Name the material.** Choose from: frosted glass & translucency (depth without heaviness), warm paper & linen (tactile, organic, almost printlike), or polished stone & metal (cool precision, weight, permanence). Most interfaces blend two. Name the primary and secondary material explicitly in your reasoning before coding.  
  
**Step 2 — Establish the palette.** The LoveFrom palette for this skill is built on cool whites and silvers — platinum `#E8E8ED`, moonstone `#F2F2F7`, brushed aluminum `#C8C8D0`, warm white `#FAFAF8` — anchored by deep neutrals like obsidian `#1C1C1E` or warm slate `#2C2C2E`. Then choose **one** precious accent and use it sparingly: gold `#C9A84C`, terracotta `#C1654A`, aged brass `#A89060`, or sage `#7A9E7E`. The accent should appear no more than twice in a composition — on the single most important interactive element, and perhaps as a hover state or data highlight.  
  
**Step 3 — Set the typographic scale.** Always pair an expressive display serif with a clean functional sans. Recommended pairings:  
  
- *Freight Display* or *Canela* (display) + *DM Sans* or *Söhne* (body/UI)  
- *Cormorant Garamond* (display) + *Neue Haas Grotesk* or *ABC Diatype* (functional)  
- *Playfair Display* (editorial moments) + *Instrument Sans* (data, labels, metadata)  
  
Headlines carry weight and personality. Body text and data labels are quiet, precise, and never compete. Letter-spacing in data labels: `0.04em`. Headline tracking: tight, almost touching (`-0.02em`). Never use Inter, SF Pro, Roboto, or system-ui as primary fonts.  
  
**Step 4 — Define the physics.** Every animated element must have a defined mass. Use spring-based easing, not linear or cubic-bezier approximations. In CSS, use `transition` with `cubic-bezier(0.34, 1.56, 0.64, 1)` for light objects (tooltips, chips, small toggles), `cubic-bezier(0.22, 1, 0.36, 1)` for medium objects (cards, drawers, modals), and `cubic-bezier(0.16, 1, 0.3, 1)` for heavy objects (full-screen transitions, page-level reveals). In React, prefer the Motion library with `type: "spring"`, tuned `stiffness` and `damping` per element weight. Duration is a consequence of physics, not a design choice.  
  
## Material Implementation  
  
### Frosted Glass  
  
```css  
background: rgba(248, 248, 252, 0.72);  
backdrop-filter: blur(24px) saturate(180%);  
-webkit-backdrop-filter: blur(24px) saturate(180%);  
border: 1px solid rgba(255, 255, 255, 0.45);  
box-shadow: 0 8px 32px rgba(0, 0, 0, 0.06), 0 1px 2px rgba(0, 0, 0, 0.04);  
```  
  
Use for: floating panels, overlaid dashboards, mobile sheets, navigation bars over content.  
  
### Warm Linen / Paper  
  
```css  
background: #FAFAF5;  
background-image: url("data:image/svg+xml,..."); /* subtle noise texture at 3% opacity */  
border: 1px solid rgba(180, 170, 155, 0.3);  
box-shadow: inset 0 1px 0 rgba(255,255,255,0.8), 0 2px 8px rgba(100,90,70,0.06);  
```  
  
Use for: cards that hold rich content, sidebars, editorial panels, data summaries.  
  
### Polished Stone / Brushed Metal  
  
```css  
background: linear-gradient(145deg, #D8D8DC 0%, #C4C4C8 50%, #CACACE 100%);  
border: 1px solid rgba(255, 255, 255, 0.5);  
box-shadow: 0 1px 0 rgba(255,255,255,0.6) inset, 0 -1px 0 rgba(0,0,0,0.08) inset, 0 4px 16px rgba(0,0,0,0.10);  
```  
  
Use for: primary action elements, hardware-like controls, anchoring components.  
  
## Spatial Composition  
  
LoveFrom compositions are not grids — they are *arrangements*. Think about how physical objects are placed on a surface: with breathing room, with hierarchy, with deliberate relationships between elements.  
  
- **Generous negative space is not emptiness — it is confidence.** Padding inside cards: never less than `28px`. Section breathing room: `64px–120px` vertically.  
- **Asymmetry signals craft.** Don’t center everything. Anchor key content left or slightly off-center. Let data breathe to the right.  
- **Layer depth, not decoration.** Use `z-index` and `box-shadow` to create a physical sense of stack. Foreground elements feel closer. Background feels recessed.  
- **Data tools**: Use a clear hierarchy of three zones — navigation/context (far left or top), primary data canvas (center), metadata/detail (right or bottom sheet). Let the data be the hero; chrome is minimal.  
- **Mobile-first components**: Full-bleed surfaces. Bottom sheet patterns with spring-based reveal. Thumb-zone awareness — primary actions live in the bottom 40% of the screen.  
- **Immersive fullscreen**: One idea per screen. Transitions between screens use full-frame morphing or crossfade with depth shifts (scale from 0.96 → 1.0). Nothing slides in from the side unless it is a physical metaphor.  
  
## Motion Principles  
  
Motion in this skill is **alive and tactile**. It behaves like the physical world because it is *modeled* on it, not imposed on it.  
  
- **Every interactive element has a resting state, an active state, and a return.** The return is always spring-based — it overshoots slightly and settles, like a physical object.  
- **Press states matter.** Buttons scale to `0.97` on press with `transform: scale(0.97)` and a subtle shadow reduction. They feel like they have give.  
- **Cards and panels respond to cursor proximity** on desktop — subtle `box-shadow` bloom, a `1-2px` `translateY(-1px)` lift. The world notices your attention.  
- **Data reveals are staggered**, never simultaneous. Rows, bars, and values enter in sequence with `animation-delay` increments of `40–80ms`. The user’s eye is guided, not overwhelmed.  
- **Loading is material.** Use shimmer animations built from the palette’s silver and platinum tones, not grey gradients. A loading skeleton should look like the thing it’s about to become.  
- **Destructive actions pause.** Deletions, irreversible changes — these feel heavier. A confirmation appears with a slower entrance (`cubic-bezier(0.22, 1, 0.36, 1)` at `320ms`), and the confirm button itself uses a deep anchor color, never the accent.  
  
## What This Skill Explicitly Rejects  
  
As Jony Ive himself has said, the best design requires saying no to a thousand things. This skill says no to:  
  
- Purple-to-indigo gradients on white backgrounds — the lingua franca of generic AI UI  
- Rounded rectangles with `border-radius: 12px` as a default — radius should be *chosen*, not defaulted  
- Icon libraries used at face value — if you use icons, customize stroke width and size to feel intentional  
- Drop shadows as decoration — shadows define physical position, nothing else  
- Micro-animations on every element — motion is precious; spend it wisely  
- Inter, Roboto, SF Pro, or system fonts as the typographic voice  
- Color used for brand expression rather than function — color communicates state, hierarchy, and one precious moment of delight  
  
## Code Quality Standards  
  
Every artifact produced with this skill must be:  
  
- **Production-grade and complete** — no placeholder content, no `// TODO`, no stub styles  
- **Cohesive** — CSS variables defined at `:root`, used everywhere, never hardcoded inline  
- **Accessible** — `prefers-reduced-motion` respected; all motion wrapped in `@media (prefers-reduced-motion: no-preference)` or equivalent  
- **Materially consistent** — the chosen material appears in every surface decision, never broken by a foreign component style  
  
```css  
:root {  
  /* LoveFrom Palette */  
  --platinum: #E8E8ED;  
  --moonstone: #F2F2F7;  
  --aluminum: #C8C8D0;  
  --warm-white: #FAFAF8;  
  --obsidian: #1C1C1E;  
  --warm-slate: #2C2C2E;  
  --ghost: rgba(248, 248, 252, 0.72);  
  
  /* One precious accent — choose one per project */  
  --accent: #C9A84C; /* gold */  
  /* --accent: #C1654A; */ /* terracotta */  
  /* --accent: #A89060; */ /* aged brass */  
  
  /* Physics curves */  
  --spring-light: cubic-bezier(0.34, 1.56, 0.64, 1);  
  --spring-medium: cubic-bezier(0.22, 1, 0.36, 1);  
  --spring-heavy: cubic-bezier(0.16, 1, 0.3, 1);  
  
  /* Spacing — generous by default */  
  --space-xs: 8px;  
  --space-sm: 16px;  
  --space-md: 28px;  
  --space-lg: 48px;  
  --space-xl: 80px;  
  --space-2xl: 120px;  
}  
```  
  
## The Final Question  
  
Before you ship, ask what Jony Ive would ask: *“Does this feel inevitable? Could it have been made any other way?”* If the answer is yes — if any element could be swapped for something else without loss — remove it or replace it with something that earns its place. The goal is not beauty for its own sake. The goal is **rightness**. When a thing is right, beauty follows without effort.  
