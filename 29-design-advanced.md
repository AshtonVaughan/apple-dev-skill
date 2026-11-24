# Design Advanced Deep Reference

Beyond `28-design-craft.md`: color science, typography depth, data-viz perception, design systems architecture, design research, ethics, and platform-specific design.

## 1. Color science — beyond "pick a hue"

### Perceptual uniformity
- **sRGB is not perceptually uniform.** Two colors with same RGB distance can look very different. Use **OKLCH** or **LAB** for color math.
- **OKLCH** — perceptually uniform, hue-accurate; `oklch(60% 0.15 250)` = lightness, chroma, hue.
- **LAB** — CIELAB, perceptually uniform-ish; `L* a* b*`.
- **HSL/HSV** — intuitive but not perceptually uniform; fine for picking, bad for math.
- **Use OKLCH for:** generating palettes, dark mode variants, accessible contrasts, color interpolation.

### Palette construction
- **Analogous** — 3-5 hues adjacent on wheel (e.g., blue-teal-green). Calm, cohesive.
- **Complementary** — 2 opposite hues (e.g., blue-orange). High contrast, energetic.
- **Triadic** — 3 hues 120° apart (e.g., red-yellow-blue). Balanced, vibrant.
- **Split-complementary** — base + 2 adjacent to complement. Softer than pure complementary.
- **Tetradic** — 4 hues 90° apart. Rich but risky — needs careful balancing.
- **Monochromatic** — 1 hue, varying lightness/chroma. Minimal, elegant.

### The 60-30-10 rule
- **60%** — dominant neutral (background, surfaces)
- **30%** — secondary (cards, sections, borders)
- **10%** — accent (primary actions, key highlights)
- Never exceed 10% accent — it loses impact.

### Dark mode color design
- **Don't invert.** Design dark mode as its own palette.
- **Elevated surfaces get lighter** — cards/modals slightly lighter than background.
- **Borders instead of shadows** — shadows don't work on dark; use subtle borders.
- **Reduce saturation** — bright colors vibrate on dark; desaturate 10-20%.
- **Test contrast** — dark mode often has lower contrast; verify WCAG.

### Color blindness
- **8% of males, 0.5% of females** — deuteranopia (red-green), protanopia (red-green), tritanopia (blue-yellow).
- **Never rely on color alone** — pair with icon, label, pattern, or position.
- **Test with simulator** — Xcode → Accessibility → Color Filters, or online simulators.
- **Safe palettes** — use Okabe-Ito or Paul Tol palettes for charts.

## 2. Typography depth

### Font selection
- **One family, 2-3 weights** — SF Pro / SF Pro Display on Apple. Don't mix families unless content differs (code=mono, display=serif).
- **Variable fonts** — single file with weight/width/optical-size axes; use `UIFontDescriptor` with variation traits.
- **Optical sizing** — text at small sizes needs different letterforms than large; use `UIFontMetrics` or variable font optical size axis.

### OpenType features
- **Ligatures** — `ffi`, `fl`, `ffi` combine; `fontFeatureSettings` or `NSFontDescriptor`.
- **Kerning** — `kern` feature; usually on by default, disable for all-caps.
- **Tabular numbers** — `tnum` feature; use for tables, prices, data.
- **Oldstyle figures** — `onum` feature; elegant for body text.
- **Small caps** — `smcp` feature; use for abbreviations, not body text.
- **Contextual alternates** — `calt` feature; glyph changes based on context.

### Text rendering
- **Minimum body size** — 15pt iOS, 13pt macOS; 17pt preferred.
- **Line height** — ≥1.4× font size for body; 1.2× for headings.
- **Measure** — 45-75 characters per line; use `maximumContentSize` or container width.
- **Hyphenation** — enable for justified text; `hyphenationFactor` in AppKit.
- **Widows/orphans** — prevent single-word lines; `paragraphStyle` or manual.

### Dynamic Type
- **`UIFontMetrics`** — scale fonts with Dynamic Type; `scaledFont(for:)`.
- **`@ScaledMetric`** — SwiftUI property wrapper for scaling.
- **Test at largest size** — layout must not break at AX5.
- **Custom fonts** — register with `UIFontDescriptor` and scale manually.

## 3. Data visualization perception

### Visual encoding ranking (most → least accurate)
1. **Position** (x/y) — most accurate for quantitative data
2. **Length** — bars, lines
3. **Angle** — pie slices (use sparingly)
4. **Area** — bubble size
5. **Volume** — 3D shapes (avoid)
6. **Color hue** — categories only, not quantities
7. **Color saturation** — intensity, not magnitude

### Pre-attentive attributes (processed <250ms)
- **Color** — pop-out, but limited to ~7 distinguishable hues
- **Orientation** — line angle, very fast
- **Size** — area/length, fast but imprecise
- **Shape** — distinct forms, fast
- **Motion** — change detection, very fast
- **Spatial position** — grouping, fastest

### Chart design rules
- **Start bar charts at zero** — truncated bars exaggerate differences
- **Use line charts for trends** — bars imply discrete categories
- **Max 5-7 colors** — more becomes indistinguishable
- **Label directly** — avoid legends when possible; label the data itself
- **Annotate key points** — callouts for peaks, troughs, anomalies
- **Show uncertainty** — error bars, confidence intervals, ranges
- **Provide context** — benchmarks, targets, historical averages

### Dashboard storytelling
- **Answer questions, not display data** — "How are we doing?" → KPI row; "Why?" → breakdown chart; "What next?" → recommendations
- **Progressive disclosure** — summary → detail → raw data
- **Comparison baselines** — every metric needs context (vs last period, vs target, vs peer)
- **Actionable insights** — every chart should suggest or enable an action

## 4. Design systems architecture

### Design tokens
```css
/* Primitive tokens */
--color-blue-500: #3B82F6;
--space-4: 16px;
--radius-md: 8px;
--font-size-body: 16px;

/* Semantic tokens */
--color-primary: var(--color-blue-500);
--color-surface: var(--color-gray-50);
--color-text: var(--color-gray-900);
--spacing-md: var(--space-4);
--radius-card: var(--radius-md);
```

- **Primitive tokens** — raw values (colors, sizes, spacing)
- **Semantic tokens** — meaning (primary, surface, danger, success)
- **Component tokens** — component-specific (button-bg, card-padding)
- **Theme tokens** — dark/light mode variants

### Component API design
- **Props over configuration** — `<Button variant="primary" size="lg">` not `<Button primary large>`
- **Composition over inheritance** — `<Card><CardHeader/><CardBody/></Card>` not `<SpecialCard>`
- **Slots for flexibility** — `<Modal><ModalHeader/><ModalBody/><ModalFooter/></Modal>`
- **Controlled vs uncontrolled** — `value`/`onChange` vs `defaultValue`

### Theming
- **CSS custom properties** — runtime theme switching
- **Dark mode** — `[data-theme="dark"]` or `prefers-color-scheme`
- **High contrast** — `prefers-contrast: more`
- **Reduced motion** — `prefers-reduced-motion: reduce`
- **Forced colors** — `forced-colors: active` (Windows High Contrast)

## 5. Design research methods

### Usability testing
- **5 users find 80% of issues** — Nielsen's law; don't need 50 users
- **Task-based** — "Find and update your profile picture" not "explore the app"
- **Think aloud** — ask users to narrate thoughts
- **Measure** — time-on-task, error rate, success rate, SUS score
- **Iterate** — test → fix → retest, not test → ship

### A/B testing
- **One variable at a time** — button color OR copy, not both
- **Statistical significance** — p < 0.05, enough sample size
- **Primary metric** — conversion, retention, engagement
- **Guardrail metrics** — don't sacrifice one for another
- **Duration** — run for full business cycles (weeks, not days)

### Heuristic evaluation
- **Nielsen's 10 heuristics** — visibility of status, match system/real world, user control, consistency, error prevention, recognition over recall, flexibility, aesthetic/minimalist, error recovery, help/docs
- **Severity rating** — 0 (not a problem) to 4 (usability catastrophe)
- **Multiple evaluators** — 3-5 experts find more issues than 1

### Cognitive walkthrough
- **Step through tasks** — what does user see/think/do at each step?
- **Four questions** — will user try? will they know correct action? will they see progress? will they understand outcome?
- **First-time user perspective** — assume zero knowledge

## 6. Design ethics

### Dark patterns (avoid)
- **Confirmshaming** — "No, I don't want to save money" on decline button
- **Forced continuity** — hidden subscription auto-renewal
- **Roach motel** — easy to sign up, hard to cancel
- **Misdirection** — visually prominent "Accept" vs hidden "Decline"
- **Bait and switch** — advertise one thing, deliver another
- **Hidden costs** — price shown without fees until checkout
- **Fake urgency** — "Only 2 left!" when actually 200 in stock
- **Fake social proof** — "12 people viewing this" when it's you
- **Trick questions** — confusing opt-out checkboxes
- **Privacy zuckering** — make privacy-invasive option easier than private

### Ethical design principles
- **Transparency** — clear about data collection, pricing, terms
- **User agency** — easy to opt out, delete data, cancel subscription
- **Honest metrics** — don't inflate numbers, hide churn, fake engagement
- **Accessible by default** — not an afterthought
- **Inclusive** — diverse users, cultures, abilities, contexts
- **Sustainable** — don't exploit attention, don't design for addiction
- **Regenerative** — leave users better than you found them

### Privacy by design
- **Data minimization** — collect only what's needed
- **Purpose limitation** — use data only for stated purpose
- **Storage limitation** — delete when no longer needed
- **Security** — encrypt, access control, audit logs
- **Transparency** — clear privacy policy, consent, control
- **User rights** — access, correction, deletion, portability

## 7. Platform-specific design depth

### iOS design specifics
- **Large titles** — collapse on scroll, expand on pull-down
- **Tab bar** — max 5 tabs, use "More" for overflow
- **Swipe actions** — leading/trailing swipe on rows
- **Context menus** — long-press for quick actions
- **Pull to refresh** — standard pattern, don't reinvent
- **Haptic feedback** — subtle on selection, strong on destructive
- **Dynamic Island** — live activities, notifications, focus
- **Lock Screen widgets** — glanceable info, complications

### macOS design specifics
- **Menu bar** — every action reachable via menu
- **Keyboard shortcuts** — ⌘ for primary, ⌥ for secondary, ⇧ for tertiary
- **Window management** — tabs, split view, full screen
- **Drag and drop** — between windows, apps, Finder
- **Services menu** — system-wide actions
- **Spotlight** — ⌘Space for quick access
- **Mission Control** — window overview, spaces
- **Stage Manager** — window organization on iPad

### watchOS design specifics
- **Glanceable** — 1-2 second interactions, not complex tasks
- **Digital Crown** — scroll, adjust, navigate
- **Complications** — always-on display, multiple families
- **Haptic-only feedback** — no sound, limited visual
- **Short sessions** — raise wrist → interact → lower wrist
- **Notifications** — brief, actionable, dismissible

### tvOS design specifics
- **10-foot UI** — large text, high contrast, simple navigation
- **Focus engine** — spatial navigation, no touch
- **Siri Remote** — touch surface + clicker, gestures
- **Parallax** — depth effect on icons, subtle
- **Top Shelf** — prominent content display
- **Single-task apps** — fullscreen, no multitasking

### visionOS design specifics
- **Spatial computing** — 3D windows, volumetric content
- **Immersive spaces** — full virtual environments
- **Passthrough** — real world visible, digital overlaid
- **Hand tracking** — pinch, grab, manipulate
- **Eye tracking** — gaze + pinch for selection
- **Shared spaces** — collaborative experiences
- **Depth cues** — shadows, occlusion, lighting for spatial presence

## 8. Design for AI/ML interfaces

### Explainability
- **Show reasoning** — "Because you X, I suggest Y" not just "I suggest Y"
- **Confidence indicators** — "I'm 85% sure" vs "I'm guessing"
- **Source attribution** — cite data sources, link to evidence
- **Alternative suggestions** — "You might also like Z"
- **Override capability** — user can correct AI decisions

### Uncertainty handling
- **Loading states** — "Thinking..." not blank
- **Progressive disclosure** — show partial results as they arrive
- **Graceful degradation** — fallback when AI fails
- **Human handoff** — escalate to human when confidence low
- **Feedback loops** — user corrections improve future outputs

### Human-in-the-loop
- **Approval gates** — high-stakes actions require confirmation
- **Batch review** — review multiple AI decisions at once
- **Undo/redo** — AI actions reversible
- **Audit trail** — log all AI decisions and user overrides
- **Transparency reports** — periodic summary of AI performance

## 9. Advanced animation

### Physics-based animation
- **Springs** — natural, interruptible, velocity-preserving
- **Friction** — realistic deceleration
- **Gravity** — falling objects, natural motion
- **Collision** — objects bouncing, impact
- **Constraints** — limits on movement, boundaries

### Shader effects
- **Metal shaders** — custom GPU effects, distortion, lighting
- **Core Image filters** — `CIFilter` for image processing
- **CALayer effects** — `cornerRadius`, `shadow`, `border`, `mask`
- **SwiftUI `visualEffect`** — custom rendering modifiers
- **`drawingGroup()`** — Metal-accelerated rendering for complex views

### Lottie/Rive integration
- **Lottie** — Airbnb's animation format, JSON-based, After Effects export
- **Rive** — interactive animations, state machines, data binding
- **Both** — resolution-independent, small file size, designer-friendly

## 10. The meta-design checklist

Before shipping any UI, verify:
- [ ] **Hierarchy obvious** — squint test passes
- [ ] **One accent color** — everything else neutral
- [ ] **8pt grid** — all spacing consistent
- [ ] **Motion has purpose** — every animation explains state change
- [ ] **Empty/loading/error states** — all three designed
- [ ] **Dark mode designed** — not inverted
- [ ] **Dynamic Type works** — text scales, layout survives
- [ ] **VoiceOver labels** — every element announced
- [ ] **Reduce Motion respected** — instant alternative
- [ ] **Contrast passes** — WCAG AA minimum
- [ ] **Touch targets ≥44pt** — no tiny buttons
- [ ] **No dead ends** — every screen has forward/back
- [ ] **No vanity metrics** — every number drives decision
- [ ] **Progressive disclosure** — complexity hidden until needed
- [ ] **Context preserved** — filters/scroll survive navigation
- [ ] **Consistent patterns** — same component behaves same
- [ ] **Fast enough** — 60fps, <100ms interactions
- [ ] **Ethical** — no dark patterns, transparent, user agency
- [ ] **Accessible** — inclusive by default, not afterthought
- [ ] **Tested with real users** — 5 users, task-based, iterate
