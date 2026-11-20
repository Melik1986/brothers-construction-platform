# UX-001 — USER EXPERIENCE SPECIFICATION

## БРАТЬЯ / եղբայրներ

**Version:** 0.1  
**Date:** 17.11.2026  
**Status:** Draft  
**Related Documents:** BRD-001, SRS-001, SAD-001, IA-001

---

# 1. UX OBJECTIVE

Design a premium architectural digital experience that communicates construction expertise, showcases completed projects and converts visitors into qualified leads.

## 1.1. UX Principles

1. Visual First.
2. Clarity.
3. Trust.
4. Conversion.
5. Consistency.
6. Performance.

# 2. USER PERSONAS

### Persona A — Private Homeowner

Needs:

- House construction.
- Apartment renovation.
- Interior finishing.
- Design consultation.

Primary action: Request a consultation.

### Persona B — Armenian Diaspora

Needs:

- Remote construction management.
- Transparent communication.
- Project documentation.
- Visual progress reports.

Primary action: Arrange an online consultation.

### Persona C — Commercial Client

Needs:

- Commercial construction.
- Renovation.
- Interior finishing.
- Individual proposals.

Primary action: Submit a commercial project inquiry.

# 3. USER JOURNEYS

### Journey A — Organic Search

Google Search → Service Page → Projects → Case Study → CTA → Lead.

### Journey B — Portfolio Discovery

Homepage → Projects → Project Detail → Before/After → Related Service → Contact.

### Journey C — Diaspora

Search / Social Media → Localized Website → Portfolio → Remote Collaboration Information → WhatsApp / Telegram → Lead.

### Journey D — Commercial

Search → Commercial Services → Relevant Projects → Company Information → Consultation → Lead.

# 4. USER FLOW REQUIREMENTS

- Every major page must expose a relevant conversion action.
- Contact channels must be accessible without mandatory form submission.
- Language switching must preserve the corresponding page when a translation exists.
- Form validation must be clear.
- Form success and error states must be designed.
- Project navigation must expose related projects and services.
- No forced account registration.

# 5. WIREFRAME SPECIFICATIONS

## 5.1 Homepage

1. Header.
2. Hero Image / Video.
3. Introduction.
4. Services Overview.
5. Featured Projects.
6. Before / After.
7. Work Process.
8. Design & 3D.
9. Company Advantages.
10. Testimonials.
11. FAQ.
12. Final CTA.
13. Footer.

## 5.2 Service Detail

1. Breadcrumbs.
2. Service Hero.
3. Description.
4. Target Customer.
5. Included Work.
6. Process.
7. Related Projects.
8. Materials and Technologies.
9. FAQ.
10. CTA.

## 5.3 Project Detail

1. Project Hero.
2. Project Metadata.
3. Initial Challenge.
4. Design Concept.
5. 3D Visualization.
6. Construction Process.
7. Before / After.
8. Final Gallery.
9. Related Services.
10. Related Projects.
11. CTA.

Optional blocks must not render when content is unavailable.

## 5.4 Contact Page

- Contact details.
- WhatsApp.
- Telegram.
- Phone.
- Email.
- Contact form.
- Service selection.
- Location information.

# 6. LEAD FORM UX

Required fields:

- Name.
- Phone.
- Service.
- Preferred Contact Method.

Optional fields:

- Email.
- Project Description.

Technical states:

- Default.
- Focus.
- Filled.
- Validation Error.
- Loading.
- Success.
- Server Error.
- Anti-spam Verification.

A successful lead submission must be confirmed by the backend.

Direct messaging actions must be tracked separately from successfully registered leads.

# 7. INTERACTION DESIGN

### Navigation

- Desktop navigation.
- Mobile navigation.
- Language selector.
- CTA.
- Accessible keyboard controls.

### Media

- Responsive gallery.
- Lightbox.
- Before / After slider.
- Lazy-loaded video.
- Optional 3D viewer.

### Animation

- CSS transitions.
- GSAP ScrollTrigger where justified.
- Reduced-motion support.
- No mandatory smooth scrolling.
- No animation blocking initial rendering.

# 8. FIGMA FILE STRUCTURE

Main file: BRATYA — Digital Platform.

Pages:

00 — Cover & Documentation  
01 — Brand Identity  
02 — Design Tokens  
03 — Foundations  
04 — Components  
05 — Desktop 1440  
06 — Tablet 768  
07 — Mobile 390  
08 — Interactions & Prototypes  
09 — Developer Handoff  
10 — Archive

## 8.1 Design Foundations

Color:

- Warm Ivory.
- Charcoal.
- Stone.
- Muted Olive.
- Semantic Feedback Colors.

Typography:

- Display.
- Heading.
- Body.
- Caption.
- Label.

Armenian font coverage must be verified.

Spacing:
4 / 8 / 12 / 16 / 24 / 32 / 48 / 64 / 96.

Grid:

- Desktop: 12 columns.
- Tablet: 8 columns.
- Mobile: 4 columns.

## 8.2 Component Library

Navigation:

- Header.
- Footer.
- Desktop Menu.
- Mobile Menu.
- Language Switcher.

Buttons:

- Primary.
- Secondary.
- Outline.
- Text Link.
- Icon Button.

Cards:

- Service.
- Project.
- Article.
- Testimonial.

Media:

- Gallery.
- Lightbox.
- BeforeAfter.
- Video.
- 3D Viewer.

Forms:

- Input.
- Phone Input.
- Select.
- Textarea.
- Validation.
- Success.
- Error.

## 8.3 Naming Convention

```text
Button/Primary
Button/Secondary

Card/Service
Card/Project
Card/Article

Navigation/Header/Desktop
Navigation/Header/Mobile

Media/BeforeAfter
Media/Gallery
Media/Video

Form/Input/Default
Form/Input/Error
Form/Input/Success
```

# 9. RESPONSIVE DESIGN

Primary design frames:

- Desktop: 1440 px.
- Tablet: 768 px.
- Mobile: 390 px.

Additional validation:

- 360 px.
- 375 px.
- 430 px.
- 1024 px.
- 1920 px.

Mobile layouts must be independently optimized.

# 10. ACCESSIBILITY

Target: WCAG 2.2 AA.

Requirements:

- Semantic structure.
- Keyboard navigation.
- Visible focus states.
- Contrast compliance.
- Accessible controls.
- Alternative text.
- Reduced motion.
- Form error announcements.

# 11. ASTRO IMPLEMENTATION MAPPING

```text
Header                 → Header.astro
Footer                 → Footer.astro
Hero                   → Hero.astro
Service Card           → ServiceCard.astro
Project Card           → ProjectCard.astro
Before/After           → BeforeAfter.tsx
Project Gallery        → ProjectGallery.tsx
Lead Form              → LeadForm.tsx
Services Section       → ServicesSection.astro
```

Use React Islands only where interactive complexity justifies them.

# 12. DESIGN HANDOFF

Every approved component must include:

- Figma component reference.
- Component states.
- Responsive behavior.
- Typography tokens.
- Color tokens.
- Spacing tokens.
- Interaction specification.
- Accessibility requirements.
- Astro implementation reference.

# 13. ACCEPTANCE CRITERIA

UX-001 is accepted when:

1. Three main personas are defined.
2. Primary user journeys are approved.
3. Main conversion flows are documented.
4. Homepage wireframe is approved.
5. Service page wireframe is approved.
6. Project case study wireframe is approved.
7. Contact form states are defined.
8. Figma file structure is established.
9. Responsive requirements are defined.
10. Accessibility requirements are documented.
11. Developer handoff requirements are established.

**Document status:** UX-001 v0.1 — Draft.
