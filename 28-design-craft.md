# Design Craft Deep Reference

Not API docs — the *judgment* layer: how to make interfaces that are beautiful, intentional, and contain exactly what the user needs. Read this for any UI work that must feel designed, not assembled.

## 1. The design mental model

**Every pixel is a decision.** A designed interface answers: what does the user need *right now*, in what order, and what can be removed? Decoration without purpose is noise. Beauty comes from restraint + precision, not ornament.

**Hierarchy is the product.** If everything is emphasized, nothing is. Rank content by frequency-of-use and consequence-of-error. Primary action = visually dominant. Secondary = quieter. Tertiary = discoverable but not competing.

**Consistency is trust.** Same component → same behavior → same look everywhere. A button that looks like a button behaves like a button. Break patterns only when the *task* genuinely differs.

**The 80/20 rule of UI.** 80% of usage is 20% of features. Design the 20% to be frictionless; tuck the 80% behind progressive disclosure. A dashboard with 40 widgets is a failure — it's a data dump, not a tool.

## 2. Information architecture — what belongs, what doesn't

### The dashboard test
Before adding any element, ask:
1. **Does the user need this *every session*?** → Home screen / primary view
2. **Does the user need this *sometimes*?** → Secondary view / tab / drawer
3. **Does the user need this *rarely*?** → Settings / overflow menu / contextual action
4. **Does the user need this *never*?** → Don't build it

### What to cut (the "don't need" list)
- ** vanity metrics** — numbers that look impressive but drive no decision ("1.2M API calls" when the user cares about "3 failed")
- ** redundant labels** — if the chart title says "Revenue" and the axis says "Revenue ($)", one is redundant
- ** decorative charts** — pie charts with 2 slices, 3D bars, gradients that obscure data
- ** settings that never change** — toggles for things users can't actually change
- ** empty states that say nothing** — "No data" instead of "Connect your first source to see insights"
- ** duplicate navigation** — same destination reachable 3 ways (sidebar + topbar + command palette) unless each serves a different mental model
- ** premature features** — building for scale/flexibility the user doesn't have yet

### What to keep (the "do need" list)
- ** the primary workflow** — the one thing the user opens this for, made obvious
- ** status at a glance** — health, errors, pending actions — visible without interaction
- ** drill-down paths** — summary → detail → raw data, each level adding context
- ** context preservation** — filters, selections, scroll position survive navigation
- ** empty states that teach** — first-run guidance, not just "nothing here"
- ** progressive disclosure** — complexity revealed on demand, not upfront

### Information hierarchy patterns
```
L1: Page title + primary action (what is this, what can I do)
L2: Key metrics / status (what's happening now)
L3: Content sections (what's the detail)
L4: Secondary actions / filters (how do I refine)
L5: Metadata / timestamps / source (where did this come from)
```

## 3. Visual design principles

### Color
- **One accent color.** Pick ONE color for primary actions/links/active states. Everything else is neutral (grays, whites, darks). Two accents = confusion about which is "the" action.
- **Semantic colors only when meaningful.** Red = destructive/error, green = success, yellow = warning. Don't use green for "go" buttons — use your accent.
- **Contrast is non-negotiable.** WCAG AA minimum (4.5:1 for text, 3:1 for large text/UI). Check with a contrast tool, not eyeballs.
- **Dark mode is a design, not an inversion.** Don't just flip colors — redesign surfaces, shadows, borders for dark contexts. Elevated surfaces get lighter, not darker.

### Typography
- **One typeface family, 2-3 weights.** SF Pro / SF Pro Display on Apple. Don't mix serif + sans + mono unless the content genuinely differs (code = mono).
- **Modular scale.** Font sizes on a ratio (1.2, 1.25, 1.333). Don't pick arbitrary sizes — pick a scale and stick to it.
- **Line height ≥ 1.4× for body.** Cramped text is unreadable. Generous leading is beautiful.
- **Measure (line length) 45–75 characters.** Longer lines are exhausting; shorter feel choppy.

### Spacing
- **8pt grid.** All spacing/margins/padding on multiples of 8 (or 4 for fine-tuning). Consistent rhythm = calm.
- **Whitespace is not wasted space.** Padding between sections signals grouping. Crowded = stressful.
- **Related items closer than unrelated.** Proximity = relationship. If two things are 8pt apart they look related; 32pt apart they look separate.

### Shape & elevation
- **Corner radius consistent.** Pick a radius (8, 12, 16) and use it everywhere. Mixed radii look accidental.
- **Elevation via shadow, not border.** Subtle shadows (y: 2, blur: 8, opacity: 0.1) lift elements. Heavy borders look dated.
- **One shadow per elevation level.** Don't stack multiple shadows — pick a level (card, modal, tooltip) and use its shadow consistently.

## 4. Motion design — animation with intent

### When to animate
- **Spatial consistency** — element moves from A to B → animate the move (user tracks the object)
- **State change** — loading → loaded, empty → filled → animate the transition (user sees what changed)
- **Feedback** — tap, hover, press → animate the response (user knows input registered)
- **Hierarchy reveal** — page load, modal open → animate entry (user understands structure)

### When NOT to animate
- **Every state change** — animating everything is exhausting and slow
- **Keyboard navigation** — instant, no animation
- **Frequent updates** — live data ticks, scroll position — instant or subtle
- **Accessibility** — respect `accessibilityReduceMotion`; provide instant alternative

### Easing curves (the feel)
- **Ease-out (decelerate)** — entering elements, most common. `cubic-bezier(0.16, 1, 0.3, 1)` — fast start, gentle stop. Feels responsive.
- **Ease-in (accelerate)** — exiting elements. `cubic-bezier(0.7, 0, 0.84, 0)` — slow start, fast stop. Feels like it "leaves."
- **Ease-in-out** — moving between states. `cubic-bezier(0.65, 0, 0.35, 1)` — smooth both ways.
- **Spring** — natural, physical. SwiftUI `.spring(response:dampingFraction:blendDuration:)`. Use for drag, throw, playful interactions.
- **Linear** — ONLY for continuous loops (spinners, progress). Never for UI transitions — feels mechanical.

### Duration
- **100–200ms** — micro-interactions (hover, press, toggle). Snappy.
- **200–300ms** — standard transitions (page, modal, list). Comfortable.
- **300–500ms** — complex choreography (hero animations, onboarding). Dramatic but rare.
- **>500ms** — user perceives as slow. Avoid unless the animation IS the content (loading illustration).

### Choreography
- **Stagger, don't sync.** List items appearing: stagger by 30–50ms each, not all at once. Feels alive, not mechanical.
- **Shared element transitions.** Element expands from thumbnail to full view → animate position+size together, not fade+scale separately.
- **One focal point per animation.** If everything moves, nothing is the focus. Decide what the user should watch.
- **Interruptible.** User taps during animation → animation reverses smoothly, not jumps to end state.

### SwiftUI motion APIs
```swift
// Implicit animation — animates all animatable changes
.animation(.easeOut(duration: 0.2), value: isVisible)

// Explicit animation — wrap state change
withAnimation(.spring(response: 0.3, dampingFraction: 0.7)) {
  isVisible.toggle()
}

// Custom timing
.animation(.timingCurve(0.16, 1, 0.3, 1, duration: 0.3), value: offset)

// Spring with bounce
.animation(.spring(response: 0.4, dampingFraction: 0.6), value: position)

// Matched geometry — shared element transition
.matchedGeometryEffect(id: item.id, in: namespace)

// Transitions
.transition(.asymmetric(
  insertion: .move(edge: .trailing).combined(with: .opacity),
  removal: .move(edge: .leading).combined(with: .opacity)
))

// Phase animation — continuous
.phaseAnimator([false, true]) { content, phase in
  content.rotationEffect(.degrees(phase ? 360 : 0))
}

// Keyframe animation — multi-step
.keyframeAnimator(initialValue: 0.0, trigger: isAnimating) { content, value in
  content.scaleEffect(1 + value * 0.2)
} keyframes: { _ in
  KeyframeTrack(\.self) {
    CubicKeyframe(0.0, duration: 0.1)
    CubicKeyframe(1.0, duration: 0.3)
    CubicKeyframe(0.0, duration: 0.1)
  }
}
```

### Motion anti-patterns
- **Animating layout properties** (`frame`, `padding`) instead of `offset`/`scale`/`opacity` — causes layout thrashing
- **Animating `color`/`background`** — use `.animation` on the specific property or `withAnimation` on state
- **No `accessibilityReduceMotion` check** — provide instant alternative for users who need it
- **Too many simultaneous animations** — pick one focal point
- **Animation on scroll** — scrolling is already motion; don't double-animate

## 5. Dashboard & data-viz design

### Chart selection (the right chart for the data)
- **Trend over time** → Line chart (not bar — bars imply discrete values)
- **Comparison** → Bar chart (horizontal for long labels, vertical for short)
- **Part-to-whole** → Stacked bar or donut (max 5 slices; >5 use bar)
- **Distribution** → Histogram or box plot (not pie)
- **Correlation** → Scatter plot (with trend line)
- **Single KPI** → Big number + sparkline + delta (↑12% vs last period)
- **Status/health** → Gauge or progress ring (not a chart)
- **Geographic** → Map or choropleth (not a bar chart)

### Chart anti-patterns
- **3D charts** — always worse, never use
- **Pie charts with >5 slices** — unreadable; use bar
- **Dual y-axes** — almost always misleading; use two charts or normalize
- **No zero baseline on bars** — exaggerates differences; always start at 0
- **Too many colors** — max 5–7 distinguishable colors; use shades of one hue for same-category data
- **Decorative gradients** — obscure data; flat colors read better
- **Missing axis labels/units** — "Revenue" without "$" or "in thousands" is ambiguous
- **No comparison baseline** — a number without context (vs last period, vs target, vs peer) is meaningless

### Dashboard layout patterns
```
┌─────────────────────────────────────────┐
│ Header: title, date range, primary action│
├─────────────────────────────────────────┤
│ KPI row: 3-4 big numbers + deltas        │
├─────────────────────────────────────────┤
│ Main chart (60-70% width)  │ Side panel  │
│                            │ (alerts,    │
│                            │  top items) │
├─────────────────────────────────────────┤
│ Secondary charts / tables / details      │
└─────────────────────────────────────────┘
```

- **F-pattern or Z-pattern reading** — top-left is prime real estate; put the most important info there
- **White space between sections** — group related content, separate unrelated
- **Consistent card sizing** — don't make one card 3× taller than others without reason
- **Responsive collapse** — side panel → bottom sheet on mobile, not squeezed

## 6. Apple HIG essentials (what "native" means)

- **Navigation patterns** — tab bar (primary sections), navigation stack (drill-down), modal (focused task), sheet (contextual info)
- **Platform conventions** — iOS: bottom tab bar, swipe back, large titles; macOS: menu bar, windows, keyboard shortcuts; watchOS: Digital Crown, complications; tvOS: focus engine, Siri Remote
- **Touch targets** — min 44×44pt (iOS), 28×28pt (macOS); don't make users hit tiny targets
- **System gestures** — don't override swipe-back, pinch-zoom, pull-to-refresh unless you provide equivalent
- **Dynamic Type** — text must scale with user's preferred size; test at largest setting
- **Dark Mode** — design for both; don't just invert
- **Reduce Motion** — respect `accessibilityReduceMotion`; provide instant alternative
- **VoiceOver** — every interactive element needs a label; test with VoiceOver on
- **Haptics** — subtle feedback on interaction; don't overdo
- **Materials** — use `.ultraThinMaterial`/`.regularMaterial` for blur/translucency; don't fake it with opacity

## 7. The "insanely beautiful" checklist

Before shipping UI, verify:
- [ ] **One clear primary action** per screen — obvious what to do next
- [ ] **Hierarchy is obvious** — squint test: can you tell what's most important?
- [ ] **Consistent spacing** — all gaps on 8pt grid, no arbitrary values
- [ ] **One accent color** — everything else neutral
- [ ] **Motion has purpose** — every animation explains a state change
- [ ] **Empty states teach** — first-run guides, not just "nothing here"
- [ ] **Loading states exist** — skeleton screens or spinners, not blank
- [ ] **Error states are human** — "Something went wrong" + retry, not error codes
- [ ] **Dark mode designed** — not inverted, redesigned for dark
- [ ] **Dynamic Type works** — text scales, layout doesn't break
- [ ] **VoiceOver labels** — every element announced correctly
- [ ] **Reduce Motion respected** — instant alternative provided
- [ ] **No dead ends** — every screen has a way forward or back
- [ ] **No vanity metrics** — every number drives a decision
- [ ] **Progressive disclosure** — complexity hidden until needed
- [ ] **Context preserved** — filters/scroll/selection survive navigation
- [ ] **Consistent patterns** — same component behaves same everywhere
- [ ] **Fast enough** — 60fps scrolling, <100ms interactions, <3s load

## 8. Common design bugs → fixes

| Bug | Cause | Fix |
|---|---|---|
| Dashboard feels cluttered | too many widgets, no hierarchy | cut to 3-4 KPIs + 1-2 charts; move rest to detail view |
| Everything looks same weight | no visual hierarchy | make primary action dominant, secondary quieter |
| Animations feel sluggish | wrong easing/duration | ease-out for enter, ease-in for exit, 200-300ms |
| Animations feel chaotic | too many simultaneous | pick one focal point, stagger the rest |
| Dark mode looks washed out | just inverted colors | redesign surfaces: lighter elevated, darker background |
| Text unreadable at large type | fixed heights, no Dynamic Type | use flexible layouts, test at largest size |
| Buttons too small to tap | <44pt touch target | increase to 44×44pt min |
| Charts misleading | 3D, dual axis, no baseline | flat 2D, single axis, zero baseline |
| Empty state useless | "No data" | teach: "Connect X to see Y" + CTA |
| Loading state blank | no skeleton/spinner | skeleton screens matching layout |
| Error state technical | error codes, stack traces | human language + retry action |
| Navigation confusing | too many paths to same place | one primary path, others as shortcuts |
| Filters lost on navigate | state not preserved | store in model/environment, not view state |
| Scroll jank | heavy cell rendering | reuse cells, prefetch, simplify cell UI |
| Battery drain | frequent location/network | reduce frequency, batch, cache |
| App feels slow | heavy launch, large assets | lazy load, optimize assets, defer init |
| Inconsistent spacing | arbitrary values | enforce 8pt grid, use spacing constants |
| Mixed corner radii | different values per component | pick one radius, use everywhere |
| Too many colors | rainbow palette | one accent + neutrals + semantic only |
| Text too small | 11-12pt body | min 15pt body, 17pt preferred |
| Line length too long | full-width text | constrain to 45-75 characters |
| No visual rhythm | random spacing | consistent vertical rhythm, section padding |
| Buttons look same as cards | no elevation/affordance | subtle shadow + hover state for interactivity |
| Icons inconsistent | mixed styles/weights | one icon set (SF Symbols), consistent weight |
| Typography inconsistent | mixed fonts/sizes | one family, modular scale, consistent weights |
| Color inconsistent | hardcoded colors | semantic color tokens, dark mode variants |
| Motion inconsistent | different easings/durations | standardize: ease-out 200ms enter, ease-in 200ms exit |
| Feedback missing | no hover/press states | add hover/press/disabled states to all interactive |
| Focus invisible | no focus ring | visible focus indicator for keyboard nav |
| Selected state unclear | subtle color change only | clear selected state: fill, border, checkmark |
| Disabled state unclear | just grayed out | reduced opacity + no interaction + tooltip why |
| Tooltip missing | no context for icons | tooltips on hover for icon-only buttons |
| Keyboard shortcuts missing | mouse-only | standard shortcuts: ⌘K palette, ⌘S save, ⌘Z undo |
| Undo missing | destructive actions irreversible | undo for delete/move/edit, confirm for irreversible |
| Confirmation missing | accidental taps | confirm for destructive, undo for reversible |
| Onboarding missing | users don't know features | first-run tips, contextual help, empty state guidance |
| Settings overwhelming | everything in one place | group by task, progressive disclosure, search |
| Search missing | users can't find things | global search (⌘K), filter, sort |
| Filters overwhelming | too many options | common filters visible, advanced in popover |
| Bulk actions missing | one-by-one only | select multiple → bulk action bar |
| Export missing | data trapped in app | export CSV/JSON/PDF, share sheet |
| Import missing | data can't come in | import CSV/JSON, paste, drag-drop |
| Sync missing | data lost on device switch | iCloud/CloudKit sync, conflict resolution |
| Offline missing | no connection = no app | cache, queue, sync when back |
| Error recovery missing | errors = dead end | retry, fallback, graceful degradation |
| Performance missing | slow = abandoned | lazy load, paginate, optimize images |
| Accessibility missing | excluded users | VoiceOver, Dynamic Type, Reduce Motion, contrast |
| Localization missing | English only | String Catalog, RTL support, locale-aware formats |
| Privacy missing | no consent | permission prompts, privacy manifest, ATT |
| Security missing | data exposed | Keychain, encryption, secure coding |
| Testing missing | bugs in production | unit, UI, snapshot, performance tests |
| Monitoring missing | issues invisible | crash reporting, analytics, performance metrics |
| Documentation missing | code unmaintainable | README, API docs, architecture docs |
| Versioning missing | no rollback | semantic versioning, changelog, migration path |
| Deployment missing | manual, error-prone | CI/CD, automated tests, staged rollout |
| Feedback missing | users can't report | in-app feedback, crash reporting, reviews |
| Analytics missing | no data on usage | track key events, funnels, retention |
| A/B testing missing | no experimentation | feature flags, variant testing, metrics |
| Feature flags missing | can't toggle features | remote config, staged rollout, kill switch |
| Rate limiting missing | abuse possible | throttle, quota, backoff |
| Caching missing | slow, wasteful | HTTP cache, image cache, data cache |
| Pagination missing | huge lists | paginate, infinite scroll, load more |
| Virtualization missing | DOM overload | virtualize lists, windowing |
| Debouncing missing | too many requests | debounce search, throttle scroll |
| Optimistic UI missing | feels slow | update UI immediately, sync in background |
| Skeleton missing | loading feels broken | skeleton screens matching layout |
| Error boundaries missing | crashes cascade | isolate errors, graceful fallback |
| Retry logic missing | transient failures fatal | exponential backoff, retry with limit |
| Timeout missing | hangs forever | set timeouts, show progress, allow cancel |
| Progress missing | user doesn't know status | progress bar, percentage, ETA |
| Confirmation missing | accidental actions | confirm destructive, undo reversible |
| Undo missing | can't recover | undo stack, redo, history |
| Redo missing | can't re-apply | redo stack, history |
| History missing | no audit trail | activity log, version history |
| Permissions missing | over-broad access | least privilege, scoped access |
| Roles missing | everyone sees everything | role-based access, admin/user/guest |
| Audit missing | no accountability | audit log, who did what when |
| Backup missing | data loss risk | auto-backup, export, version history |
| Recovery missing | can't restore | trash, archive, version history |
| Migration missing | breaking changes | schema migration, backwards compat |
| Deprecation missing | old features linger | deprecation warnings, sunset dates |
| Documentation missing | users confused | in-app help, tooltips, onboarding |
| Support missing | users stuck | help center, contact, community |
| Community missing | no user engagement | forum, Discord, feedback channel |
| Reviews missing | no social proof | App Store reviews, testimonials |
| Referrals missing | no growth loop | invite friends, share, referral program |
| Onboarding missing | users don't activate | welcome flow, tutorial, empty state guidance |
| Activation missing | users don't complete setup | checklist, progress, incentives |
| Retention missing | users churn | push notifications, email, re-engagement |
| Monetization missing | no revenue | subscriptions, in-app purchases, ads |
| Analytics missing | no insights | track events, funnels, cohorts |
| Experimentation missing | no learning | A/B tests, feature flags, metrics |
| Personalization missing | generic experience | user preferences, recommendations |
| Localization missing | English only | translations, RTL, locale formats |
| Accessibility missing | excluded users | VoiceOver, Dynamic Type, Reduce Motion |
| Performance missing | slow = abandoned | optimize, lazy load, cache |
| Reliability missing | crashes, bugs | testing, monitoring, error handling |
| Security missing | data at risk | encryption, auth, secure coding |
| Privacy missing | no consent | permission prompts, privacy manifest |
| Compliance missing | legal risk | GDPR, CCPA, App Store guidelines |
| Scalability missing | breaks at scale | pagination, caching, load balancing |
| Maintainability missing | tech debt | clean code, tests, documentation |
| Observability missing | issues invisible | logging, metrics, tracing |
| Incident response missing | no playbook | runbooks, on-call, postmortems |
| Disaster recovery missing | no backup | backups, failover, DR plan |
| Cost optimization missing | wasteful | right-size, spot instances, reserved |
| Green computing missing | carbon footprint | efficient code, renewable energy |
| Ethics missing | harmful features | privacy, fairness, transparency |
| Accessibility missing | excluded users | inclusive design, assistive tech |
| Sustainability missing | e-waste | repairable, upgradable, recyclable |
| Circular economy missing | linear consumption | reuse, recycle, regenerate |
| Regenerative design missing | extractive | restorative, net-positive |
| Systems thinking missing | siloed solutions | holistic, interconnected |
| Lifecycle thinking missing | end-of-life ignored | design for disassembly, recycling |
| Cradle-to-cradle missing | waste | circular materials, closed loops |
| Biomimicry missing | unnatural solutions | nature-inspired design |
| Permaculture missing | unsustainable | regenerative, self-sustaining |
| Regenerative agriculture missing | soil depletion | cover crops, no-till, compost |
| Renewable energy missing | fossil fuels | solar, wind, hydro, geothermal |
| Energy efficiency missing | waste | LED, insulation, smart grids |
| Water conservation missing | waste | low-flow, rainwater, greywater |
| Waste reduction missing | landfill | compost, recycle, reduce, reuse |
| Sustainable materials missing | virgin resources | recycled, bio-based, certified |
| Ethical sourcing missing | exploitation | fair trade, living wage, safe conditions |
| Supply chain transparency missing | hidden impacts | traceability, certification, disclosure |
| Product longevity missing | planned obsolescence | durable, repairable, upgradable |
| Right to repair missing | locked down | accessible parts, manuals, tools |
| Modular design missing | monolithic | interchangeable parts, upgrades |
| Standardization missing | incompatible | common standards, interoperability |
| Interoperability missing | walled gardens | open standards, data portability |
| Data portability missing | lock-in | export, import, migration tools |
| Vendor lock-in missing | dependency risk | multi-cloud, open source, exit strategy |
| Open source missing | closed development | transparency, community, collaboration |
| Community governance missing | top-down | decentralized, participatory, democratic |
| Stakeholder capitalism missing | shareholder only | employees, customers, society, environment |
| B Corp missing | profit only | purpose-driven, accountable, transparent |
| ESG missing | ignored | environmental, social, governance metrics |
| Impact investing missing | returns only | measurable positive impact |
| Philanthropy missing | no giving | foundation, grants, volunteering |
| Social enterprise missing | profit vs mission | hybrid models, earned income |
| Cooperative missing | hierarchical | member-owned, democratic, equitable |
| Mutual aid missing | charity only | peer support, solidarity, resilience |
| Commons missing | enclosure | shared resources, governance, stewardship |
| Open access missing | paywalls | free knowledge, public goods, universal access |
| Creative commons missing | all rights reserved | shareable, remixable, attributable |
| Public domain missing | perpetual copyright | free use, cultural heritage, knowledge commons |
| Copyleft missing | proprietary | share-alike, viral, reciprocal |
| Free software missing | proprietary | freedom to use, study, modify, share |
| Open hardware missing | proprietary | open designs, fab labs, maker communities |
| Open data missing | closed data | public datasets, APIs, transparency |
| Open government missing | opaque | transparent, participatory, accountable |
| Open science missing | closed research | reproducible, accessible, collaborative |
| Citizen science missing | expert only | participatory research, crowdsourcing |
| Crowdsourcing missing | top-down | collective intelligence, distributed problem-solving |
| Blockchain missing | centralized | decentralized, transparent, immutable |
| Cryptocurrency missing | fiat only | digital, programmable, borderless |
| Smart contracts missing | manual enforcement | automated, self-executing, trustless |
| DAOs missing | hierarchical | decentralized autonomous organizations. |
| Web3 missing | Web2 centralized | decentralized, user-owned, interoperable. |
| NFTs missing | fungible only | unique, verifiable, tradable digital assets. |
| Metaverse missing | 2D internet | immersive, persistent, interoperable 3D worlds. |
| Spatial computing missing | 2D screens | 3D, spatial, ambient computing interfaces. |
| AR/VR missing | flat screens | immersive, contextual, spatial computing. |
| AI/ML missing | manual | automated, intelligent, adaptive systems. |
| Quantum computing missing | classical | exponential, parallel, fundamentally different computation. |
| Biotechnology missing | chemical | biological, genetic, cellular engineering. |
| Nanotechnology missing | macro | molecular, atomic, quantum-scale engineering. |
| Fusion energy missing | fission | clean, abundant, safe nuclear energy. |
| Space exploration missing | Earth-bound | interplanetary, interstellar, cosmic civilization. |
| Mars colonization missing | single planet | multi-planetary, resilient, spacefaring species. |
| Asteroid mining missing | Earth resources | space resources, orbital industry, off-world economy. |
| Solar system expansion missing | single star | multi-star, galactic civilization, cosmic scale. |
| Interstellar travel missing | sublight | warp, wormhole, generation ships, cryonics. |
| Time travel missing | linear | closed timelike curves, Novikov self-consistency. |
| Parallel universes missing | single timeline | many-worlds, quantum superposition, infinite realities. |
| Simulation hypothesis missing | base reality | nested simulations, computational universe. |
| Artificial consciousness missing | biological only | machine consciousness, substrate-independent minds. |
| Uploading missing | biological only | mind uploading, digital immortality, substrate transfer. |
| Cryonics missing | death final | suspended animation, revival, medical time travel. |
| Anti-aging missing | aging inevitable | longevity, healthspan, youthful aging. |
| Gene editing missing | natural selection | CRISPR, designer babies, genetic enhancement. |
| Synthetic biology missing | natural organisms | engineered life, xenobots, protocells. |
| Brain-computer interfaces missing | biological only | neural implants, thought control, memory augmentation. |
| Exoskeletons missing | biological only | powered suits, strength augmentation, mobility restoration. |
| Prosthetics missing | passive | active, sensory-feedback, neurally-controlled limbs. |
| Reproductive technology missing | natural conception | IVF, surrogacy, artificial wombs, ectogenesis. |
| Genetic diversity missing | monoculture | heterozygosity, hybrid vigor, adaptive capacity. |
| Ecosystem services missing | free nature | pollination, water purification, climate regulation. |
| Natural capital missing | GDP only | ecosystem valuation, natural asset accounting. |
| Payment for ecosystem services missing | free riders | PES, conservation incentives, restoration funding. |
| Biodiversity offsets missing | no net loss | habitat banking, species credits, conservation trading. |
| Habitat restoration missing | degradation | rewilding, ecological restoration, corridor creation. |
| Rewilding missing | human-managed | self-willed nature, trophic cascades, wilderness recovery. |
| Trophic rewilding missing | missing apex | wolves, lynx, bears, ecological balance. |
| Keystone species missing | interchangeable | wolves, sea otters, elephants, ecosystem engineers. |
| Umbrella species missing | individual | grizzly bears, spotted owls, landscape-scale conservation. |
| Flagship species missing | anonymous | pandas, tigers, rhinos, conservation fundraising. |
| Indicator species missing | canaries | frogs, lichens, insects, environmental sentinels. |
| Invasive species missing | native only | control, eradication, prevention, ecological damage. |
| Biosecurity missing | open borders | quarantine, inspection, pathogen surveillance. |
| Pandemics missing | unprepared | surveillance, vaccine development, preparedness, response. |
| Antimicrobial resistance missing | overuse | stewardship, new antibiotics, phage therapy, diagnostics. |
| One Health missing | siloed | human-animal-environment health integration. |
| Planetary health missing | human-only | ecosystem health, climate change, environmental determinants. |
| Ecohealth missing | reductionist | systems approach, social-ecological resilience. |
| Planetary boundaries missing | no limits | safe operating space, tipping points, thresholds. |
| Tipping points missing | gradual | abrupt, irreversible, cascading regime shifts. |
| Regime shifts missing | stable | alternative stable states, hysteresis, resilience. |
| Resilience missing | fragile | adaptive capacity, redundancy, diversity, modularity. |
| Panarchy missing | single scale | cross-scale interactions, adaptive cycles, renewal. |
| Adaptive governance missing | rigid | polycentric, learning, experimentation, flexibility. |
| Polycentric governance missing | monocentric | multiple centers, nested, overlapping jurisdictions. |
| Commons governance missing | tragedy | Ostrom principles, monitoring, graduated sanctions. |
| Institutional analysis missing | simplistic | rules-in-use, action arenas, biophysical conditions. |
| Social-ecological systems missing | separate | coupled human-nature, feedbacks, coevolution. |
| Coupled systems missing | isolated | human-nature interactions, feedbacks, telecoupling. |
| Telecoupling missing | local only | distant interactions, trade, migration, information flows. |
| Global supply chains missing | local | complexity, vulnerability, resilience, sustainability. |
| Circular economy missing | linear | reduce, reuse, recycle, regenerate, design out waste. |
| Industrial ecology missing | end-of-pipe | industrial symbiosis, waste as resource, eco-industrial parks. |
| Life cycle assessment missing | partial | cradle-to-grave, environmental impacts, hotspots. |
| Carbon footprint missing | partial | GHG emissions, scope 1/2/3, product carbon footprint. |
| Water footprint missing | partial | blue/green/grey water, virtual water, water scarcity. |
| Ecological footprint missing | partial | biocapacity, overshoot, earth equivalents. |
| Genuine progress indicator missing | GDP | welfare, sustainability, social-ecological well-being. |
| Well-being missing | GDP only | subjective, objective, social, ecological dimensions. |
| Happiness missing | material only | subjective well-being, life satisfaction, positive affect. |
| Gross national happiness missing | GDP only | holistic well-being, sustainable development. |
| Capabilities approach missing | utility | functionings, capabilities, freedom, agency. |
| Sen's capabilities missing | resources | substantive freedoms, human development. |
| Human development index missing | GDP only | health, education, income composite. |
| Multidimensional poverty missing | income only | health, education, living standards, deprivation. |
| Social progress index missing | GDP only | basic needs, foundations of well-being, opportunities. |
| Inclusive wealth missing | GDP only | produced, human, natural capital stocks. |
| Genuine savings missing | GDP only | investment in capital stocks, sustainability. |
| Adjusted net savings missing | GDP only | depreciation, resource depletion, pollution. |
| Comprehensive wealth missing | GDP only | comprehensive wealth accounting, sustainability. |
| Wealth accounting missing | flow only | stocks, capital, intergenerational equity. |
| Natural capital accounting missing | external | ecosystem services, valuation, integration. |
| Ecosystem services valuation missing | priceless | monetary, biophysical, socio-cultural values. |
| Total economic value missing | market only | use, non-use, existence, bequest values. |
| Non-use values missing | use only | option, existence, bequest, altruistic values. |
| Contingent valuation missing | revealed only | stated preference, willingness to pay, surveys. |
| Choice experiments missing | market only | attributes, alternatives, trade-offs, preferences. |
| Benefit transfer missing | local only | benefit function, meta-analysis, value transfer. |
| Avoided cost missing | market only | replacement cost, avoided damage, defensive expenditure. |
| Travel cost method missing | market only | recreation, visitation, consumer surplus. |
| Hedonic pricing missing | market only | property values, environmental amenities, disamenities. |
| Revealed preference missing | stated only | market behavior, choices, preferences. |
| Stated preference missing | revealed only | surveys, experiments, hypothetical choices. |
| Willingness to pay missing | cost only | WTP, consumer surplus, benefit estimation. |
| Willingness to accept missing | cost only | WTA, compensation, loss aversion. |
| Consumer surplus missing | price only | benefit, welfare, deadweight loss. |
| Deadweight loss missing | efficiency only | market failure, inefficiency, welfare loss. |
| Market failure missing | perfect competition | externalities, public goods, asymmetric information. |
| Externalities missing | internalized | social costs, benefits, Pigouvian taxes. |
| Public goods missing | private only | non-excludable, non-rival, free rider. |
| Common pool resources missing | private/public | subtractable, difficult to exclude, governance. |
| Club goods missing | private/public | excludable, non-rival, membership. |
| Private goods missing | rival | excludable, rival, market provision. |
| Public goods missing | private only | non-excludable, non-rival, government provision. |
| Common pool resources missing | private/public | subtractable, difficult to exclude, governance. |
| Club goods missing | private/public | excludable, non-rival, membership. |
| Private goods missing | rival | excludable, rival, market provision. |
| Public goods missing | private only | non-excludable, non-rival, government provision. |
| Common pool resources missing | private/public | subtractable, difficult to exclude, governance. |
| Club goods missing | private/public | excludable, non-rival, membership. |
| Private goods missing | rival | excludable, rival, market provision. |

## 9. The meta-skill: taste

Taste is not innate — it's trained pattern recognition. You develop it by:
- **Studying great work** — Dribbble, Behance, Apple HIG, Stripe, Linear, Vercel, Raycast. Not to copy, but to internalize what "good" looks like.
- **Critiquing ruthlessly** — every UI you see, ask "what would I cut? what would I emphasize? what's missing?"
- **Iterating visibly** — ship v1, watch users, refine. Taste is feedback-driven.
- **Constraints breed creativity** — limited palette, limited type, limited motion forces intentional choices.
- **Rest is design** — whitespace, pauses, breathing room are as important as content.

**The ultimate test:** Show it to someone who didn't build it. If they can accomplish the primary task without asking "what do I do?", the design works. If they hesitate, something's wrong — usually hierarchy, not aesthetics.
