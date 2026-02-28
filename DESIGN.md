# Chain Snap — Design Document
Round 17

---

## 1. Identity

**Name:** Chain Snap
**Tagline:** *Pull. Hold. Let go.*

**What is the player (specific):** You are a kinetic disruptor — a force that passes through a network of hostile nodes like a finger dragged across a taut wire. You have no body, no health bar. You are the gesture.

**World feel:**
A bioluminescent web stretched across a midnight forest canopy — living tissue under tension, each link a ligament waiting to be severed. The nodes pulse like fireflies trapped in amber, aware of your approach, tightening against you.

**Emotional experience:** *Tense precision releasing into chaos.*

**Reference games:**
- **R-Type** — DNA: reading the screen as a system, the satisfaction of a perfectly-timed input in a hostile environment, punishment that teaches rather than frustrates
- **Thumper** — DNA: the physicality of audio-visual feedback locked to rhythm, the sense that you are *hitting* the world with your input, escalating pattern density
- **Inside** — DNA: environmental storytelling through object behavior, a world that reacts with more intelligence than you expect, dread and beauty coexisting

---

## 2. Visual Spec

**Background:** `#3A6B3A` — forest green, full opacity, flat fill. Never darken toward black.

**Color Palette:**

| Role | Hex | Usage |
|------|-----|-------|
| Background | `#3A6B3A` | Canvas fill |
| Node fill | `#C8F0A0` | Enemy cluster bodies |
| Node stroke | `#8FD460` | Node border ring |
| Link (resting) | `#FFFDE0` | Elastic connection lines |
| Link (tensioned) | `#FFD700` | Link under active drag stretch |
| Link (critical) | `#FF6B1A` | Link at >80% snap threshold |
| Recoil flash | `#FFFFFF` | Full-screen additive flash on snap |
| Score/UI | `#EAFFCC` | HUD text |
| Chain reaction | `#FF3366` | Secondary node burst ring |
| Background grain | `#4A7B4A` | Subtle noise overlay, 8% opacity |

**Bloom:**
- Strength: `1.4`
- Threshold: `0.55`
- Radius: `18px`
- Applied to: node bodies, tensioned links, recoil flash, chain reaction rings

**Camera:**
- Type: Fixed orthographic 2D
- Position: Dead center of canvas, no scroll
- Zoom: 1:1 pixel ratio
- Canvas size: 540×960px (portrait mobile), letterboxed on desktop with `#2E5A2E` bars

**Player silhouette:** *Invisible force, visible consequence*

---

## 3. Sound Spec

**Music: 108 BPM**
Polyrhythmic forest percussion — a 4/4 backbone of deep wooden drum hits (think taiko processed through reverb, not acoustic) layered with a 3-against-4 marimba pattern. No melodic line in early levels. A single synthesized "breath" tone (sine wave, C2, slow LFO vibrato at 0.8Hz) pulses under everything. At level 3+, a 16th-note hi-hat grid locks in, and a bass drone (Bb1) enters on the 1. The music never pauses — it reacts.

**Music state changes (6 triggers):**

1. **Game start / level load** → Filter opens: highpass sweeps from 800Hz to 80Hz over 1.8 seconds. Feels like a door opening into the forest.
2. **Link enters tension (drag begins)** → Pitch-shift whole track up by +2 semitones over 0.3s. Stays shifted while drag is held.
3. **Snap occurs** → Transient hit layer: single pitched rim-shot at root note + major 3rd (C+E), 50ms decay. Track pitch snaps back to normal in 0.1s.
4. **Chain reaction (2+ nodes)** → Marimba arp fires: ascending minor pentatonic run, 16th notes, starting from C4, 4 notes. One per reaction level.
5. **Level complete** → All drums drop, breath tone swells to +6dB over 0.8s, single bell tone (C5 sine, 1.2s decay).
6. **Level failed** → Track pitch-shifts down -3 semitones over 0.6s, then hard cut to silence. 200ms later: single low drum hit.

**6 SFX (Tone.js):**

1. **link_stretch** — `Tone.Synth({ oscillator: { type: "sawtooth" }, envelope: { attack: 0.01, decay: 0.1, sustain: 0.8, release: 0.2 } })` — pitch rises from 220Hz to `220 + (stretchRatio * 440)`Hz in real-time as drag increases. Volume: -18dB base.

2. **link_snap** — `Tone.MetalSynth({ frequency: 400, envelope: { attack: 0.001, decay: 0.08, release: 0.05 }, harmonicity: 5.1, modulationIndex: 32, resonance: 4000, octaves: 1.5 })` — sharp metallic crack, panned slightly left or right based on snap position on screen.

3. **node_recoil** — `Tone.MembraneSynth({ pitchDecay: 0.08, octaves: 6, envelope: { attack: 0.001, decay: 0.3, sustain: 0, release: 0.1 } })` — deep thud at 80Hz, triggered when a recoiled node hits screen boundary or another node.

4. **chain_reaction** — `Tone.Synth({ oscillator: { type: "triangle" }, envelope: { attack: 0.005, decay: 0.15, sustain: 0, release: 0.1 } })` — tone at 330Hz (E4), pitched up +2 semitones per chained snap, max 4 steps. Each fires 80ms after previous.

5. **danger_pulse** — `Tone.LFO({ frequency: 3, type: "square" })` modulating `Tone.Gain` on an ambient drone — activates when <3 links remain unsnapped. Creates rhythmic volume swell at 3Hz, barely audible but felt.

6. **level_clear_chime** — `Tone.PolySynth` playing C4-E4-G4-C5 (major chord) as arpeggiated sine waves, 120ms between notes, each with `{ attack: 0.02, decay: 0.8, sustain: 0, release: 0.5 }`. Volume: -10dB.

---

## 4. Mechanic Spec

**Core loop:** Tap and drag elastic links between enemy nodes to stretch them past their snap threshold, triggering recoil that sends nodes flying and chains reactions through the network.

---

### Input Behavior

**pointerdown:**
- Hit detection radius: `28px` from link midpoint (generous — link is thin, finger is fat)
- If pointer lands within 28px of any link midpoint: *attach* to that link. Link highlight activates (`#FFD700`). `link_stretch` SFX begins.
- If no link within 28px: no action, no feedback. Dead input.

**pointermove:**
- Track delta from pointerdown origin.
- Stretch ratio = `clampedDistance / snapThreshold` where `clampedDistance = min(dragDistance, snapThreshold * 1.2)`
- Link visually deforms: midpoint follows pointer, endpoints stay anchored to node centers. Bézier curve with control point at pointer position.
- Link color interpolates: `#FFFDE0` → `#FFD700` → `#FF6B1A` across 0%→50%→80% of snap threshold.
- When stretch ratio ≥ 1.0: snap triggers on next frame (do not wait for pointerup).
- Minimum drag threshold to register stretch: `20px` (prevents accidental triggers on tap).

**pointerup (before snap threshold):**
- Link snaps back to resting position with spring rebound.
- Rebound animation: damped spring, `stiffness: 180`, `damping: 12`, `mass: 0.8`.
- `link_stretch` SFX fades out over 0.15s.
- No damage, no penalty.

---

### Physics Values

**Link elasticity:**
- `snapThreshold`: distance in pixels that triggers snap (level-dependent, see below)
- `springStiffness`: `180` (Newton/meter equivalent, normalized to canvas units)
- `springDamping`: `12`
- `restLength`: distance between two connected nodes at spawn (varies per level layout)

**Node recoil on snap:**
- Snap releases stored elastic energy as impulse.
- `recoilForce = (stretchDistance / snapThreshold) * baseRecoilMagnitude`
- `baseRecoilMagnitude`: `1400` canvas-units/s² (level-dependent scalar, see below)
- Direction: away from snap midpoint, along the link axis, equal and opposite for both connected nodes.
- Node mass: `1.0` (all nodes identical)
- Velocity cap: `900` canvas-units/s
- Friction (drag coefficient): `0.04` per frame (60fps) — nodes coast, they don't stop fast.
- Boundary behavior: elastic bounce off canvas edges, restitution coefficient `0.6`.

**Chain reaction rules:**
- When a flying node collides with another node: check if that node has remaining links.
- If the collision force exceeds `chainTriggerThreshold` (`380` canvas-units/s at impact): snap ALL links on the struck node simultaneously.
- Chain reaction delay between snaps: `80ms` per level (visual separation, not gameplay pause).
- Maximum chain depth: `6` nodes.
- Chain reaction score multiplier: `1.5×` per additional node beyond first (see Score System).
- A node with no links cannot chain-trigger; it just bounces.

---

### Difficulty Curve (5 Levels)

| Parameter | L1 | L2 | L3 | L4 | L5 |
|-----------|----|----|----|----|-----|
| `snapThreshold` (px) | 140 | 120 | 100 | 85 | 70 |
| `baseRecoilMagnitude` | 1000 | 1200 | 1400 | 1600 | 1900 |
| Node count | 6 | 8 | 10 | 12 | 15 |
| Links per node (avg) | 1.5 | 2.0 | 2.5 | 3.0 | 3.5 |
| `chainTriggerThreshold` (u/s) | 280 | 330 | 380 | 380 | 380 |
| Time limit (seconds) | 60 | 55 | 50 | 45 | 40 |
| Minimum snaps to win | 4 | 6 | 8 | 10 | 14 |
| Node respawn | No | No | Yes (1×) | Yes (2×) | Yes (3×) |
| Moving nodes | No | No | No | Yes (1 node, 60px/s orbit) | Yes (3 nodes, 80px/s) |

---

### Win / Lose Conditions

**Win:** All required links snapped (see `minimumSnaps` per level) within time limit. Win triggers level-complete sequence.

**Lose:** Timer reaches 0:00 with insufficient snaps completed. No lives — instant retry offered.

**Perfect:** All links on all nodes snapped AND at least one chain reaction of depth ≥3 achieved. Unlocks visual skin for next run (golden link color `#FFD700` → `#FF4488` hot-pink glow).

---

### Score System

- **Base snap score:** `100` points per link snapped
- **Speed bonus:** `+50` points if snap completed within 1.5 seconds of touching the link
- **Chain multiplier:** Score for the triggering snap × `1.5^(chainDepth-1)` — so depth-3 chain on a 100pt snap = `100 × 1.5² = 225`
- **Recoil boundary hit:** `+25` points when a recoiling node hits a canvas edge at >500 u/s
- **Time remaining bonus:** At level complete: `timeRemaining × 10` points added
- **Combo:** Consecutive snaps within 2.0 seconds of each other add a ×1.2 multiplier, stacking up to ×3.0
- Score displayed in top-right: `#EAFFCC`, monospaced font, 28px, updates live

---

## 5. Level Design

### Level 1 — "First Fiber"
**What's new:** The mechanic itself. One cluster, few links, generous thresholds.
**Layout:** 6 nodes arranged in a loose hexagon, ~200px diameter. 9 links (1-2 per node). Static.
**Parameter changes from defaults:** `snapThreshold: 140`, `baseRecoilMagnitude: 1000`
**Teaching moment:** First link is dead-center screen, horizontal, at comfortable thumb reach. The game *shows you the link glowing* before you touch it (idle animation pulses it).
**Win condition:** 4 snaps

---

### Level 2 — "Crossing Wires"
**What's new:** Links that cross each other — you must choose which to snap first. Two clusters, no connection between them.
**Layout:** Two groups of 4 nodes each. Left cluster: diamond shape. Right cluster: square. 6 links in left cluster (crossing pair in center), 4 in right. 160px between clusters.
**Parameter changes:** `snapThreshold: 120`, node count: 8
**Teaching moment:** Snapping a crossing-link pair sends nodes from both toward each other — first near-collision moment.
**Win condition:** 6 snaps

---

### Level 3 — "The Web"
**What's new:** Chain reactions become mandatory to win. First node respawn.
**Layout:** 10 nodes in irregular web. 3 high-tension "spine" links run diagonally across canvas. Snapping spine link sends node flying into side cluster — chain reaction is the *only* way to hit the required snap count in time.
**Parameter changes:** `snapThreshold: 100`, `baseRecoilMagnitude: 1400`, respawn: 1×
**Teaching moment:** Player discovers that aiming recoil at a cluster is more efficient than hunting individual links.
**Win condition:** 8 snaps — impossible without 1+ chain reaction

---

### Level 4 — "Orbit"
**What's new:** One moving node. Link to it stretches and compresses dynamically as it orbits. Snapping its link sends it off its orbit in a new direction.
**Layout:** 12 nodes. One "satellite" node orbits a central anchor at 60px/s, orbit radius 120px, clockwise. 4 links connect satellite to inner cluster at rest. Moving node's links change rest length dynamically — player must time their grab to when the link is *stretched* (faster snap) not compressed.
**Parameter changes:** `snapThreshold: 85`, `chainTriggerThreshold: 380`, moving node speed: 60px/s
**Teaching moment:** Timing + tension awareness. The moving node teaches the player that link *state* matters, not just position.
**Win condition:** 10 snaps

---

### Level 5 — "Tangle"
**What's new:** 3 moving nodes, short time limit, 3 respawns. Maximum density.
**Layout:** 15 nodes, 3 orbiting at different speeds (60/75/80 px/s), orbit radii (90/130/160px). Dense link web — avg 3.5 links per node. Several "hub" nodes with 5+ connections. Snapping a hub creates cascade.
**Parameter changes:** `snapThreshold: 70`, `baseRecoilMagnitude: 1900`, time: 40s, respawn: 3×
**The challenge:** Chain reactions from hubs can trigger counter-chains that reform connections (respawned nodes re-link with 50% probability at 60px restLength) — player must sequence hub snaps before respawns connect.
**Win condition:** 14 snaps (all non-respawned links)

---

## 6. The Moment

Level 3. You've been methodically snapping links one at a time, getting the rhythm. You spot the diagonal spine link — it's glowing orange (`#FF6B1A`), stretched taut between two distant nodes. You drag it. It bends deep. You feel the link-stretch SFX pitch rising. At the threshold: snap.

The two nodes rocket outward in opposite directions. The left node blows through a 4-node cluster at 840 u/s. Four links detonate in sequence — 80ms apart — each with its own metallic crack, each adding a note to the ascending pentatonic run. The marimba fires four notes. The whole canvas flashes white four times, staggered. `+100 → +150 → +225 → +338`. Score climbs `+813` in under half a second.

Then silence. The remaining nodes drift. You blink.

---

## 7. Emotional Arc

**First 30 seconds:**
Uncertainty → discovery. The player taps near a node, nothing happens. Taps a link: it stretches, glows, the SFX rises — and they let go. The link rebounds. *Oh. I have to go further.* Second attempt: they push past the threshold. Snap. Node flies. The drum hit lands exactly on the beat. "Wait — did the game do that on purpose?"

**After 2 minutes:**
Flow state. The player is no longer reading the screen analytically — they're feeling it. They're pulling links with confidence, scanning for chain opportunities, tracking moving nodes with peripheral awareness. The 108 BPM has entered their hands. They snap early in the beat window automatically.

**Near win (final 3 links):**
The music is at full density. The `danger_pulse` is trembling under everything. The player counts the remaining links. Two are easy. One is a moving node, fast, in the densest part of the tangle. They wait. They watch the orbit. They grab. Pull. Threshold incoming — and snap. The chain reaction fires. Level clear chime. Silence. Then the swell.

---

## 8. Identity Line

**This is the game where you pull a living wire until it breaks, and the world flies apart in perfect rhythm.**

---

## 9. Start Screen

### Idle Animation (game-world specific)

The start screen shows a live simulation of 8 nodes arranged in a rough circle, connected by 12 links — the same visual grammar as gameplay. The simulation runs continuously:

- **Nodes:** Slowly breathe (scale pulse from 1.0→1.08→1.0 over 2.4s, staggered per node by `nodeIndex × 0.3s` offset). Color: `#C8F0A0`. Radius: 14px. Glow enabled.
- **Links:** All at rest. They sway gently: each link midpoint drifts ±8px in a slow Lissajous pattern (x-frequency 0.11Hz, y-frequency 0.17Hz, phase offset per link). Links are drawn as quadratic Bézier curves between node centers, control point at the drifting midpoint.
- **Autonomous snap event (loops every 4.5 seconds):** One link is chosen at random. Over 0.8 seconds, its midpoint is pulled outward by 90px (simulating a drag). At 0.8s: snap fires. Both connected nodes launch with recoil (baseRecoilMagnitude: 600 for idle — gentler than gameplay). The nodes coast and softly decelerate (friction 0.03), then drift back to their start positions over 2.0 seconds. The snapped link regenerates with a thin alpha-fade-in over 0.6s.
- **Background:** `#3A6B3A` flat, no gradient. Very faint grain texture overlay (8% opacity, `#4A7B4A`).
- **After 8 seconds without input:** A secondary idle snap fires a chain reaction (2 linked nodes snap together). Plays the chain reaction SFX at -8dB.

---

### SVG Overlay

#### Option A — Glow Title "CHAIN SNAP" (required)

```svg
<svg width="540" height="200" viewBox="0 0 540 200" xmlns="http://www.w3.org/2000/svg">
  <defs>
    <!-- Outer glow: wide, soft, forest-offset -->
    <filter id="glow-outer" x="-40%" y="-40%" width="180%" height="180%">
      <feGaussianBlur stdDeviation="14" result="blur1"/>
      <feColorMatrix type="matrix"
        values="0 0 0 0 0.55
                0 0 0 0 0.95
                0 0 0 0 0.30
                0 0 0 2.5 -0.5" result="green-glow"/>
      <feMerge>
        <feMergeNode in="green-glow"/>
        <feMergeNode in="SourceGraphic"/>
      </feMerge>
    </filter>
    <!-- Inner tight glow: white core -->
    <filter id="glow-inner" x="-10%" y="-10%" width="120%" height="120%">
      <feGaussianBlur stdDeviation="3" result="blur2"/>
      <feMerge>
        <feMergeNode in="blur2"/>
        <feMergeNode in="SourceGraphic"/>
      </feMerge>
    </filter>
    <!-- Animated opacity for breathing effect -->
    <animate id="breathe" attributeName="opacity"
      values="0.88;1.0;0.88" dur="2.4s" repeatCount="indefinite"/>
  </defs>

  <!-- Drop shadow layer -->
  <text x="270" y="105"
    font-family="'Courier New', monospace"
    font-size="72" font-weight="900"
    letter-spacing="8"
    text-anchor="middle"
    fill="#1A3A1A"
    opacity="0.6"
    transform="translate(3,4)">CHAIN SNAP</text>

  <!-- Outer glow layer -->
  <text x="270" y="105"
    font-family="'Courier New', monospace"
    font-size="72" font-weight="900"
    letter-spacing="8"
    text-anchor="middle"
    fill="#8FD460"
    filter="url(#glow-outer)"
    opacity="0.9">
    CHAIN SNAP
    <animate attributeName="opacity"
      values="0.7;0.95;0.7" dur="2.4s" repeatCount="indefinite"/>
  </text>

  <!-- Core text: crisp white with tight glow -->
  <text x="270" y="105"
    font-family="'Courier New', monospace"
    font-size="72" font-weight="900"
    letter-spacing="8"
    text-anchor="middle"
    fill="#EAFFCC"
    filter="url(#glow-inner)">
    CHAIN SNAP
    <animate attributeName="opacity"
      values="0.95;1.0;0.95" dur="2.4s" repeatCount="indefinite"/>
  </text>

  <!-- Subtitle -->
  <text x="270" y="148"
    font-family="'Courier New', monospace"
    font-size="16" font-weight="400"
    letter-spacing="6"
    text-anchor="middle"
    fill="#8FD460"
    opacity="0.75">PULL · HOLD · LET GO</text>
</svg>
```

**Filter values summary:**
- `glow-outer`: `feGaussianBlur stdDeviation="14"` — wide bloom halo in `#8FD460`-toned green
- `glow-inner`: `feGaussianBlur stdDeviation="3"` — tight white core bleed
- Breathing animation: 2.4s cycle, opacity 0.88→1.0→0.88

---

#### Option B — Tap-to-Start Button with Pulse Ring

```svg
<svg width="540" height="120" viewBox="0 0 540 120" xmlns="http://www.w3.org/2000/svg">
  <defs>
    <filter id="btn-glow" x="-30%" y="-30%" width="160%" height="160%">
      <feGaussianBlur stdDeviation="8" result="blur"/>
      <feMerge>
        <feMergeNode in="blur"/>
        <feMergeNode in="SourceGraphic"/>
      </feMerge>
    </filter>
  </defs>

  <!-- Expanding pulse ring (3 rings, staggered) -->
  <circle cx="270" cy="60" r="42" fill="none" stroke="#8FD460" stroke-width="1.5" opacity="0">
    <animate attributeName="r" values="42;72" dur="1.8s" begin="0s" repeatCount="indefinite"/>
    <animate attributeName="opacity" values="0.7;0" dur="1.8s" begin="0s" repeatCount="indefinite"/>
  </circle>
  <circle cx="270" cy="60" r="42" fill="none" stroke="#8FD460" stroke-width="1.5" opacity="0">
    <animate attributeName="r" values="42;72" dur="1.8s" begin="0.6s" repeatCount="indefinite"/>
    <animate attributeName="opacity" values="0.7;0" dur="1.8s" begin="0.6s" repeatCount="indefinite"/>
  </circle>
  <circle cx="270" cy="60" r="42" fill="none" stroke="#8FD460" stroke-width="1.5" opacity="0">
    <animate attributeName="r" values="42;72" dur="1.8s" begin="1.2s" repeatCount="indefinite"/>
    <animate attributeName="opacity" values="0.7;0" dur="1.8s" begin="1.2s" repeatCount="indefinite"/>
  </circle>

  <!-- Button pill -->
  <rect x="170" y="34" width="200" height="52" rx="26"
    fill="#3A6B3A" stroke="#8FD460" stroke-width="2"
    filter="url(#btn-glow)"/>

  <!-- Button text -->
  <text x="270" y="68"
    font-family="'Courier New', monospace"
    font-size="20" font-weight="700"
    letter-spacing="4"
    text-anchor="middle"
    fill="#EAFFCC">TAP TO SNAP</text>
</svg>
```

**Timing:** Pulse rings staggered at 0s / 0.6s / 1.2s offsets, each expanding from r=42 to r=72 over 1.8s with fade to opacity=0. Creates continuous ripple effect.

---

### Start Screen Composition (layered, top to bottom)

1. `#3A6B3A` canvas fill
2. Live idle simulation (canvas/WebGL layer) — nodes + links running physics
3. Option A SVG title — centered horizontally, `y` offset: 22% from top
4. Option B tap button — centered, `y` offset: 72% from top
5. Bottom metadata: `Round 17 · 108 BPM` in `#8FD460`, 11px, letter-spacing 3, opacity 0.5

---

*End of Design Document — Chain Snap, Round 17*
