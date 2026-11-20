# IA-001 — INFORMATION ARCHITECTURE

## БРАТЬЯ / եղբայրներ

**Version:** 0.2  
**Date:** 17.11.2026  
**Status:** Draft — Updated Business & UX Baseline

### 1. Brand Identity

- Brand name: Братья
- Armenian name: եղբայրներ
- Positioning: Premium Construction & Renovation
- Visual direction: Premium Architecture
- Working tagline: From Vision to Reality
- Typography: Editorial / Large Typography
- Photography: Real architectural photography

### 2. Target Geography

Primary market:

- Yerevan.
- Regions of Armenia.

Target audiences:

- Private homeowners.
- Property investors.
- Commercial customers.
- Armenian diaspora.

### 3. Languages

- Armenian (hy) — primary.
- Russian (ru).
- English (en).

### 4. Main Navigation

1. Home
2. Services
3. Projects
4. About
5. Design & 3D
6. Journal
7. Contact

Primary CTA: Discuss Your Project.

Priority contact channels:

1. WhatsApp.
2. Telegram.
3. Phone.
4. Email.

### 5. Sitemap

```text
/
├── /hy/
│   ├── /services/
│   │   ├── /apartment-renovation/
│   │   ├── /house-construction/
│   │   ├── /interior-finishing/
│   │   ├── /interior-design/
│   │   ├── /facade-roofing/
│   │   └── /commercial-construction/
│   ├── /projects/
│   │   └── /[project-slug]/
│   ├── /about/
│   ├── /design/
│   ├── /journal/
│   │   └── /[article-slug]/
│   ├── /locations/
│   ├── /contact/
│   ├── /privacy-policy/
│   └── /terms/
│
├── /ru/
│   └── Same localized structure
│
└── /en/
    └── Same localized structure
```

### 6. Main Page Templates

- Homepage.
- Services Overview.
- Service Detail.
- Projects Overview.
- Project Case Study.
- About.
- Design & 3D.
- Journal.
- Article Detail.
- Contact.
- Legal Pages.

### 7. Content Entities

- Services.
- Projects.
- Articles.
- Testimonials.
- Team Members.
- FAQ.
- Locations.
- Global Settings.

### 8. Localization Architecture

Every localized entity must have:

- Stable translationKey.
- Locale.
- Localized slug.
- Localized content.
- SEO metadata.

SEO requirements:

- Canonical.
- hreflang.
- Localized sitemap.
- Localized metadata.
- Correct language routing.

### 9. User Journey

Discovery → Exploration → Evaluation → Trust → Conversion.

Main conversion actions:

- WhatsApp.
- Telegram.
- Phone.
- Lead Form.

### 10. SEO Strategy

- Service-specific landing pages.
- Real project case studies.
- Local search optimization.
- Regional content where justified.
- Internal linking.
- Structured data.
- Expert journal content.

### 11. UX Priorities

P0:

- Homepage.
- Services.
- Projects.
- Project Details.
- Contact.
- Multilingual navigation.

P1:

- Design & 3D.
- Journal.
- FAQ.
- Testimonials.

P2:

- Advanced interactive 3D.
- Extended regional landing pages.
- Advanced calculators.

### 12. Acceptance Criteria

- Main navigation approved.
- Sitemap approved.
- Three-language structure defined.
- Content entities defined.
- User journeys defined.
- Conversion points identified.
- MVP page templates identified.

**Document status:** IA-001 v0.2 — Draft.
