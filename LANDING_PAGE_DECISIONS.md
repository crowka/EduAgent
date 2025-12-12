# EduAgent Landing Page Decisions

> **Version:** 1.0  
> **Last Updated:** 2024-12-12  
> **Status:** Decided

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Framework Decision](#framework-decision)
3. [Hosting Decision](#hosting-decision)
4. [A/B Testing Strategy](#ab-testing-strategy)
5. [Email Capture Strategy](#email-capture-strategy)
6. [Core Web Vitals Optimization](#core-web-vitals-optimization)
7. [Landing Page Architecture](#landing-page-architecture)
8. [Cost Analysis](#cost-analysis)
9. [Implementation Checklist](#implementation-checklist)

---

## Executive Summary

### Decision: Next.js (Same as App) + Cloudflare Pages

**Rationale:** For a solo developer building one product, the simplicity of one framework outweighs the marginal performance gains of a separate static site generator.

| Aspect | Decision | Alternative Considered |
|--------|----------|------------------------|
| **Framework** | Next.js 14 (App Router) | Astro |
| **Hosting** | Cloudflare Pages | Vercel Pro |
| **A/B Testing** | PostHog (free tier) | None affordable |
| **Email Capture** | GetWaitlist | Mailchimp, MailerLite |
| **Total Cost** | ~$10/year (domain only) | $35/month (Framer) |

---

## Framework Decision

### Options Evaluated

| Framework | Learning Curve | JS Shipped | Best For |
|-----------|---------------|------------|----------|
| **Astro** | Low-Moderate | 0KB default | Pure static landing pages |
| **Next.js** | Moderate (already using) | 20-40KB optimized | Full-stack apps + landing |
| **Framer** | Low (no-code) | N/A | Design-focused, non-developers |

### Decision: Next.js for Everything

**Pros:**
- ✅ One codebase, one deploy
- ✅ Shared components (buttons, forms, design system)
- ✅ No context switching between frameworks
- ✅ Already in the stack (no new learning)
- ✅ Server Components ship zero JS for static content
- ✅ Can achieve 95+ Lighthouse scores with optimization

**Cons:**
- ❌ Requires knowing optimization patterns (not automatic like Astro)
- ❌ Slightly heavier baseline than pure static generators
- ❌ More complexity than needed for simple landing page

### When to Reconsider Astro

Add Astro as a separate landing page framework if:
- Building 5+ different products with separate landing pages
- Page speed becomes a measurable conversion bottleneck
- Want complete separation between marketing site and app
- Team grows and marketing site needs different deploy cycle

---

## Hosting Decision

### Options Evaluated

| Platform | Free Bandwidth | Custom Domains | Commercial Use | Cost |
|----------|---------------|----------------|----------------|------|
| **Cloudflare Pages** | Unlimited | ✅ Free + SSL | ✅ Allowed | $0 |
| **Vercel Hobby** | 100GB/month | ✅ Free + SSL | ❌ Personal only | $0 |
| **Vercel Pro** | 1TB/month | ✅ Free + SSL | ✅ Allowed | $20/month |
| **Netlify Free** | 100GB/month | ✅ Free + SSL | ✅ Allowed | $0 |

### Decision: Cloudflare Pages

**Rationale:** 
- Unlimited bandwidth on free tier
- Commercial use explicitly allowed
- Supports Next.js via adapter
- Free SSL, free custom domains
- Edge network (fast globally)

**Important:** Vercel's Hobby tier prohibits commercial use. For a SaaS product, this means either paying $20/month for Vercel Pro or using Cloudflare Pages for free.

### Implementation

```bash
# Install Cloudflare Pages adapter
npm install @cloudflare/next-on-pages

# Build for Cloudflare
npx @cloudflare/next-on-pages
```

---

## A/B Testing Strategy

### The Budget Problem

| Platform | A/B Testing Tier | Monthly Cost |
|----------|------------------|--------------|
| Unbounce | Starting tier | $74/month |
| Leadpages | Pro tier | $74/month |
| Instapage | Starting tier | $79/month |
| Framer | Scale tier | $100/month |
| **PostHog** | **Free tier** | **$0** |

No affordable no-code platform offers A/B testing under $74/month.

### Decision: PostHog Free Tier

**Included in free tier:**
- 1 million feature flag requests/month
- Built-in A/B testing
- Analytics
- Session replays
- Unlimited experiments

**Implementation:**

```typescript
// lib/posthog.ts
import posthog from 'posthog-js'

export function initPostHog() {
  posthog.init(process.env.NEXT_PUBLIC_POSTHOG_KEY!, {
    api_host: 'https://app.posthog.com',
  })
}

// components/Hero.tsx
'use client'
import { useFeatureFlagVariantKey } from 'posthog-js/react'

export function Hero() {
  const variant = useFeatureFlagVariantKey('hero-headline')
  
  const headlines = {
    control: "Learn anything with AI",
    'variant-a': "AI that teaches how you learn",
    'variant-b': "Master any subject 2x faster",
  }
  
  return <h1>{headlines[variant] || headlines.control}</h1>
}
```

### Priority A/B Tests

Run in this order (highest impact first):

1. **Headlines** — "Learn 2x faster with AI" vs "AI that teaches how you learn"
2. **CTA copy** — "Start Free Trial" vs "Start Learning Now" vs "Try 7 Days Free"
3. **Hero imagery** — Video demo vs static screenshot vs illustration
4. **Form length** — Email only vs Email + "What do you want to learn?"

---

## Email Capture Strategy

### Options Evaluated

| Tool | Free Tier | Best Feature |
|------|-----------|--------------|
| **GetWaitlist** | 100,000 signups | Most generous, referral built-in |
| LaunchList | $79 one-time | No subscription |
| MailerLite | 1,000 subscribers | Landing page builder included |
| Buttondown | 100 subscribers | Simple newsletter |

### Decision: GetWaitlist (Free Tier)

**Rationale:**
- 100,000 signups free (more than enough for validation)
- Built-in referral mechanics
- Simple embed, no backend needed
- Can migrate to Supabase-based system later

**Implementation:**

```typescript
// components/EmailCapture.tsx
'use client'

export function EmailCapture() {
  return (
    <div 
      id="getWaitlistContainer" 
      data-waitlist_id="YOUR_ID"
      data-widget_type="WIDGET_1"
    />
  )
}

// In layout or page, add the script
<Script src="https://prod-waitlist-widget.s3.us-east-2.amazonaws.com/getwaitlist.min.js" />
```

---

## Core Web Vitals Optimization

### Targets

| Metric | Target | What It Measures |
|--------|--------|------------------|
| **LCP** (Largest Contentful Paint) | < 2.5s | How fast main content loads |
| **INP** (Interaction to Next Paint) | < 200ms | How fast page responds to clicks |
| **CLS** (Cumulative Layout Shift) | < 0.1 | How much page jumps around |

### Optimization Checklist

#### 1. Static Generation

```typescript
// app/(marketing)/page.tsx
export const dynamic = 'force-static'
export const revalidate = false
```

#### 2. Image Optimization

```typescript
import Image from 'next/image'

<Image
  src="/hero.png"
  alt="EduAgent hero"
  width={1200}
  height={630}
  priority          // Preloads for LCP
  placeholder="blur"
/>
```

#### 3. Font Optimization

```typescript
// app/layout.tsx
import { Inter } from 'next/font/google'

const inter = Inter({ 
  subsets: ['latin'],
  display: 'swap',
  preload: true,
})
```

#### 4. Server Components (Zero JS)

```typescript
// Landing page structure - all Server Components except interactive parts
export default function LandingPage() {
  return (
    <>
      <Hero />          {/* Server Component - 0KB JS */}
      <Features />      {/* Server Component - 0KB JS */}
      <Testimonials />  {/* Server Component - 0KB JS */}
      <Pricing />       {/* Server Component - 0KB JS */}
      <EmailCapture />  {/* Only this needs 'use client' */}
    </>
  )
}
```

#### 5. Prevent Layout Shift

```css
.hero-image {
  aspect-ratio: 16/9;
}

.testimonial-carousel {
  min-height: 300px;
}
```

### Expected Results

| Metric | Unoptimized | Optimized | Target |
|--------|-------------|-----------|--------|
| **LCP** | 3-5s | 1.2-1.8s | < 2.5s ✅ |
| **INP** | 300-500ms | 50-100ms | < 200ms ✅ |
| **CLS** | 0.2-0.5 | 0.01-0.05 | < 0.1 ✅ |
| **JS Bundle** | 150KB+ | 20-40KB | Minimal ✅ |

---

## Landing Page Architecture

### Route Structure

```
app/
├── (marketing)/              ← Landing pages (static, public)
│   ├── layout.tsx            ← Marketing layout (minimal nav)
│   ├── page.tsx              ← Homepage
│   ├── pricing/
│   │   └── page.tsx
│   ├── about/
│   │   └── page.tsx
│   └── blog/
│       └── page.tsx
│
├── (app)/                    ← Actual app (dynamic, authenticated)
│   ├── layout.tsx            ← App layout (full nav, auth required)
│   ├── learn/
│   ├── progress/
│   └── settings/
│
└── (auth)/                   ← Auth pages
    ├── login/
    └── signup/
```

**Why route groups?**
- `(marketing)` and `(app)` have different layouts
- Visitors to landing page don't download app code
- Clean URL structure (no `/marketing/` prefix)

### Component Structure

```
components/
├── marketing/
│   ├── Hero.tsx              # Server Component
│   ├── Features.tsx          # Server Component
│   ├── Testimonials.tsx      # Server Component
│   ├── Pricing.tsx           # Server Component
│   ├── FAQ.tsx               # Server Component
│   └── EmailCapture.tsx      # Client Component (interactive)
│
└── shared/
    ├── Button.tsx
    ├── Card.tsx
    └── Container.tsx
```

---

## Cost Analysis

### No-Code Approach (Framer)

| Item | Cost |
|------|------|
| Framer Pro (for custom domain) | $30/month |
| Domain (~$12/year) | ~$1/month |
| Email capture (GetWaitlist free) | $0 |
| A/B testing | ❌ Not available under $100/month |
| **Total** | **~$31/month** |

### Our Approach (Next.js + Cloudflare)

| Item | Cost |
|------|------|
| Hosting (Cloudflare Pages) | $0 |
| Domain (~$12/year) | ~$1/month |
| Email capture (GetWaitlist free) | $0 |
| A/B testing (PostHog free) | $0 |
| Analytics (PostHog free) | $0 |
| **Total** | **~$1/month ($12/year)** |

### Savings

- **Monthly:** $30/month saved
- **Yearly:** $360/year saved
- **With A/B testing:** Priceless (no affordable no-code option)

---

## Implementation Checklist

### Phase 1: Setup (Day 1)

- [ ] Create `(marketing)` route group
- [ ] Create marketing layout (minimal nav)
- [ ] Set up `next/font` with Inter or preferred font
- [ ] Configure Cloudflare Pages deployment
- [ ] Set up PostHog account and add tracking script

### Phase 2: Build Landing Page (Day 1-2)

- [ ] Hero section with headline, subheadline, CTA
- [ ] Features section (3-6 key benefits)
- [ ] Social proof (testimonials or "Join X learners")
- [ ] Pricing section (Free trial + $9.99/month)
- [ ] FAQ section
- [ ] Footer with links

### Phase 3: Optimize (Day 2)

- [ ] Add `priority` to hero image
- [ ] Add `placeholder="blur"` to images
- [ ] Verify all marketing components are Server Components
- [ ] Add `aspect-ratio` to prevent CLS
- [ ] Run Lighthouse audit, fix issues
- [ ] Test on mobile

### Phase 4: Email Capture (Day 2)

- [ ] Set up GetWaitlist account
- [ ] Embed waitlist widget
- [ ] Test signup flow
- [ ] Set up confirmation email

### Phase 5: A/B Testing (Week 2+)

- [ ] Set up PostHog feature flags
- [ ] Create headline variants
- [ ] Create CTA variants
- [ ] Monitor conversion rates
- [ ] Iterate based on data

---

## Must-Have Landing Page Elements

High-converting SaaS landing pages (3-15%+ conversion) include:

| Element | Requirement |
|---------|-------------|
| **Hero** | Benefit-driven headline (5-6 words), single prominent CTA, product visual |
| **Social Proof** | Testimonials with photos/names, specific metrics, customer logos |
| **Core Web Vitals** | LCP < 2.5s, INP < 200ms, CLS < 0.1 |
| **Mobile-first** | Responsive design, touch-friendly buttons |
| **Minimal Form** | Email-only converts highest; each field reduces conversion |
| **Clear CTA** | One primary action per screen |
| **Trust Signals** | Security badges, "No credit card required", testimonials |

---

## Related Documents

| Document | Description |
|----------|-------------|
| [PRE_MVP_PROTOTYPE.md](./PRE_MVP_PROTOTYPE.md) | Full prototype specification |
| [PRD.md](./PRD.md) | Product requirements |
| [ARCHITECTURE_DECISIONS.md](./ARCHITECTURE_DECISIONS.md) | Technical decisions |

---

## Decision Log

| Date | Decision | Rationale |
|------|----------|-----------|
| 2024-12-12 | Use Next.js (not Astro) | One codebase, already in stack |
| 2024-12-12 | Use Cloudflare Pages | Free, unlimited, commercial allowed |
| 2024-12-12 | Use PostHog for A/B testing | Only free option with experiments |
| 2024-12-12 | Use GetWaitlist for email | 100K free signups |

