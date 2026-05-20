# 🎵 AI 음악 비주얼 생성 서비스 — 랜딩 페이지 Cursor 프롬프트

아래 프롬프트를 Cursor의 Composer(또는 Chat)에 그대로 붙여넣으세요.

---

## ✅ CURSOR PROMPT (복사해서 사용)

```
Create a single-page landing page (index.html or Next.js/React component) for an AI-powered music visual generator service targeting indie musicians and content creators. The service analyzes uploaded music (MP3/WAV or SoundCloud/YouTube links) and generates high-quality looping video artwork optimized for Spotify Canvas (9:16 vertical), TikTok Reels, and YouTube Shorts.

---

## TECH STACK
- Use Next.js (App Router) + Tailwind CSS + Framer Motion
- If Next.js is not set up, use a single self-contained index.html with vanilla JS and CSS animations
- No backend required — all interactions are UI/demo only

---

## DESIGN DIRECTION

### Overall Tone
- Dark mode ONLY: deep black (#0a0a0a) or very dark charcoal (#0d0d0f) base
- Point colors: minimal white/silver text with neon/glitch accents appearing ONLY on hover or interaction
- Aesthetic: Industrial-cyber meets underground music culture — raw, textured, not clean corporate SaaS
- Background: subtle animated noise texture or grain overlay layered behind all content
- Interactive detail: mouse movement subtly shifts background graphic noise or waveform particles (parallax)

### Typography
- Display font: "Bebas Neue" or "Anton" (Google Fonts) for headings — tall, condensed, bold
- Body font: "DM Mono" or "Space Mono" for subtext — monospace techy feel
- NO Inter, NO Roboto, NO system-ui

### Motion
- Page load: staggered fade-in + slight upward translate for each section (Framer Motion or CSS animation-delay)
- Genre tag clicks: smooth crossfade transition on the showcase card
- CTA button: glitch flicker effect on hover (CSS clip-path animation)
- Scroll-triggered reveals for each section

---

## PAGE SECTIONS (in order)

---

### SECTION 1 — Hero

**Layout:**
- Full viewport height (100vh)
- Behind text: a high-quality looping video or CSS-animated background showing glitchy industrial visuals (dark concrete texture, neon scan lines, or geometric loop). If video is not feasible, use a CSS animated gradient mesh with noise overlay.
- Centered text overlay with slight backdrop blur

**Content:**
- Headline (H1): "음악의 장르를 해석하다, 비주얼을 완성하다."
  - English sub-headline below in smaller monospace: "Your sound. Interpreted. Visualized."
- Sub-copy (small, muted): "음악을 업로드하면 AI가 장르와 무드를 분석하여, 스포티파이 캔버스와 숏폼에 최적화된 아티스틱 영상을 단 1초 만에 생성합니다."
- CTA Buttons (two, side by side):
  - Primary: [ 내 음원 업로드하기 (무료) ] — dark button with neon border glow on hover + glitch flicker
  - Secondary: [ 10초 만에 비주얼 뽑기 ] — ghost/outline button
- Below buttons: drag-and-drop zone with dashed border — label: "MP3 / WAV / SoundCloud / YouTube 링크"

---

### SECTION 2 — Genre Aesthetics Interactive Showcase

**Layout:**
- Section title: "장르별 미학 쇼케이스" (left-aligned, large display font)
- Center: a large card UI resembling a "floating player card" or "digital album cover" (roughly 400x600px or 16:9 wide format). This card shows a genre-specific animated visual.
- Above/beside the card: horizontal genre tag pills

**Genre Tags (clickable, mutually exclusive selection):**
| Tag | Visual Aesthetic | Description |
|-----|-----------------|-------------|
| #Industrial_Rock | Dark concrete texture, halftone noise, monochrome graphics | 묵직하고 거친 디스토션 신스 사운드 |
| #Electronica | Neon blue/cyan color waves, cyberpunk laser, geometric loop | 정박의 테크노 비트와 청량한 패드 사운드 |
| #Vocaloid_Subculture | Neon pink/purple, glitch animation, kitsch typography | 빠르고 통통 튀는 디지털 신스 사운드 |
| #Ambient | Slow aurora gradients, soft blur, minimal motion | 공간감 있는 드론 사운드 |
| #Lo-Fi_Hip-Hop | Warm grain, VHS overlay, muted earth tones | 느슨한 붐뱁 비트와 vinyl 크래클 |

**Interaction:**
- Clicking a genre tag:
  1. Crossfades the card's background CSS animation/gradient to match that genre's aesthetic
  2. Updates the description text below the card
  3. Plays a short (2–3 second) CSS animation simulating the visual style
- Active tag gets a bright highlight/underline; others are muted

**Below card:** small italic copy — "💡 실제 업로드 시, AI가 당신의 곡 BPM·악기·감성을 분석해 이 비주얼을 자동 생성합니다."

---

### SECTION 3 — How It Works (3-Step Process)

**Layout:**
- Three horizontal steps with icons, short title, and one-line description
- Connecting dashed line or arrow between steps
- Section background: slightly lighter dark (e.g. #111114) to differentiate

**Steps:**
1. 🎵 **Drop Your Sound** — "MP3/WAV 파일이나 사운드클라우드/유튜브 링크를 넣습니다."
2. 🤖 **AI Aesthetic Analysis** — "AI가 곡의 BPM, 악기 구성, 장르적 감성을 매핑합니다."
3. 🎬 **Get Your Artwork** — "단 몇 초 만에 내 곡에 완벽히 녹아드는 고화질 루프 영상 완성."

**Motion:** Each step animates in on scroll (slide up + fade in, staggered by 200ms)

---

### SECTION 4 — Social Proof & Deliverables

**Layout:**
- Two-column: left = deliverable specs, right = social proof quote

**Left — Deliverable Specs:**
- ✅ Spotify Canvas 규격 (9:16 vertical) 지원
- ✅ TikTok / Reels / Shorts 최적화
- ✅ 고화질 루프 비디오 (MP4 export)
- ✅ 숏폼 바이럴을 위한 최적의 치트키

**Right — Social Proof:**
- Quote card: "인디 뮤지션 OO명이 이미 앨범 홍보에 사용 중입니다" (styled as a testimonial chip)
- Add 2 more short fictional testimonials styled as review cards

---

### SECTION 5 — Final CTA

**Full-width dark section:**
- Large headline: "지금 바로 당신의 사운드를 비주얼로."
- Sub: "무료로 시작하세요. 카드 없이."
- Single large CTA button: [ 무료로 비주얼 생성하기 ] with glitch hover animation

---

## IMPLEMENTATION NOTES

1. **Noise/grain overlay:** Use an SVG feTurbulence filter or a CSS pseudo-element with a semi-transparent PNG grain texture over the entire page
2. **Genre card animations:** Use CSS @keyframes for each genre (e.g. `industrial` = slow horizontal scan line; `electronica` = pulsing neon ring; `vocaloid` = rapid color flicker)
3. **Glitch button effect:** CSS clip-path animation that briefly slices the button text on hover
4. **Mouse parallax:** On mousemove over hero, shift a background layer by (event.clientX * 0.02)px — subtle depth effect
5. **All text in Korean** except section labels that are naturally English (Spotify Canvas, TikTok, etc.)
6. **Responsive:** Must look good on mobile (375px) and desktop (1440px). On mobile, the genre showcase stacks vertically.
7. **Performance:** Prefer CSS animations over JS where possible. No heavy 3rd-party libraries beyond Framer Motion (if React).

---

## FILE OUTPUT
- If React/Next.js: create `app/page.tsx` (or `pages/index.tsx`) + any needed component files
- If HTML: single `index.html` with embedded `<style>` and `<script>`
- Include Google Fonts import for Bebas Neue + DM Mono
```

---

## 💡 추가 팁

- Cursor에서 **Composer 모드**(`Cmd+I` / `Ctrl+I`)에 붙여넣으면 전체 파일 구조까지 한 번에 생성됩니다.
- 생성 후 "섹션 2의 장르 카드를 실제로 Framer Motion으로 애니메이션 추가해줘" 처럼 후속 프롬프트로 디테일을 다듬으세요.
- 배경 노이즈 텍스처는 [grainy-gradient.com](https://grainy-gradient.vercel.app/) 에서 CSS를 바로 복사할 수 있습니다.
