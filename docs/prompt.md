Create a single-file HTML webpage that replicates the homepage of an organic handmade soap and skincare e-commerce brand called "Triple Diamond Multi Service." 

## Brand Identity
- Tagline: "Discover the Art of Self-Care with Our Luxurious Handmade Soaps – Crafted with Love, Nature, and Quality in Every Bar!"
- Based in the UK and Nigeria
- Products: 100% handmade soaps, organic shea butter, face oils, body butters, luxury skincare sets, organic honey
- Values: vegan-friendly, cruelty-free, chemical-free, eco-friendly, plant-based

## Design Aesthetic
- Natural, organic, earthy luxury feel
- Color palette: deep forest greens (#2d5a27 or similar), warm cream/off-white backgrounds (#faf7f2), gold accents, rich browns
- Typography: Use Google Fonts — a serif or elegant display font for headings (e.g., Playfair Display or Cormorant Garamond) paired with a clean readable body font (e.g., Lato or Nunito)
- Warm, welcoming, artisanal market feel — not sterile or corporate

## Page Sections (in order)

### 1. Navigation Bar
- Logo text "Triple Diamond" on the left with a small leaf/diamond icon
- Nav links: Home | Handmade Soap | Shop by Categories (dropdown: 100% Handmade Soap, Face Oil, Luxury Skincare Set, Combo Set, Body Butter, Organic Shea Butter, Organic Honey) | About Us | Blog | Soap Making
- Cart icon with "£0.00" on the right
- Sticky/fixed on scroll with subtle shadow

### 2. Hero Section
- Full-width hero with a lush nature/botanical background (use a CSS gradient: deep green to warm amber/gold overlay on a dark texture)
- Large centered heading: "Discover the Art of Self-Care with Our Luxurious Handmade Soaps"
- Subtext: "Crafted with Love, Nature, and Quality in Every Bar!"
- A decorative leaf divider element
- Two CTA buttons: "Shop Now" (solid green) and "Our Story" (outlined)
- Subtle fade-in animation on load

### 3. Trending Products Section
- Section title: "Trending Products" with decorative underline
- 4 product cards in a responsive grid (2x2 on mobile, 4 in a row on desktop)
- Each card contains:
  - Product image placeholder (use a soft green/cream gradient box with a leaf emoji 🌿 as placeholder)
  - "40% Off" badge (red/coral badge in top-left corner)
  - "SALE!" label
  - Category tag (e.g., "Organic Shea Butter", "100% Handmade Soap")
  - Product name (bold)
  - Star rating (5 gold stars)
  - Strikethrough original price and current price in green
  - "Add to Basket" button
- Products to include:
  1. 100% ORGANIC RAW SHEA BUTTER 50g — was £2.55, now £1.53 — Organic Shea Butter — 5 stars
  2. 3 thick bars of Lavender and Sunflower Soap — was £19.00, now £11.40 — 100% Handmade Soap
  3. 7 boxes of lovely bars of handmade — was £40.00, now £24.00 — 100% Handmade Soap
  4. A luxurious pack of 7 biggest bar sizes (Gift set) — was £53.00, now £31.80 — 100% Handmade Soap

### 4. Our Story Section
- Two-column layout: text on left, decorative element/signature GIF placeholder on right
- Heading: "Our Story"
- Body text: "At Triple Diamond, we believe nature holds the secret to radiant beauty and wellness. Inspired by the rich traditions of organic skincare, we craft 100% handmade soaps, organic shea butter, and other natural products with love and care. Rooted in the United Kingdom and Nigeria, we blend local expertise with premium, plant-based ingredients to deliver gentle, skin-nourishing solutions that are vegan-friendly and free from chemicals."
- Signature: "Tinu Odunuga — Director / CEO" with a decorative handwritten-style font
- Warm cream background for this section

### 5. Why People Choose Us Section
- Section heading: "Why People Choose Us"
- 3 icon cards side by side:
  1. 🌱 "100% Organic" — pure, chemical-free ingredients, ensuring gentle care for your skin
  2. 🌿 "Nature's Best" — Plant-based oils and nutrients to nourish, protect, and rejuvenate
  3. ♻️ "Eco-Friendly" — vegan-friendly, cruelty-free products that are kind to the planet
- Light green or white card backgrounds with subtle border

### 6. Customer Reviews Section
- Section heading: "Customers Reviews"
- 5 review cards in a horizontal scroll or 3-column grid
- Each card:
  - Italic quote text in quotation marks
  - Reviewer name (bold)
  - Reviewer title/occupation below name
- Reviews:
  1. "I enjoyed using the lavender and sunflower handmade soap... The smell is heaven." — Grace, First Time Customer
  2. "The organic shea butter cleared my stretch marks after giving birth..." — Folly, Shopper
  3. "My sensitive skin allows me to use bars without fragrance..." — Rebecca, Fashion Designer
  4. "I highly recommend shea natural soap for people with sensitive skin..." — Mafoso, Doctor
  5. "My favourite is the lovely green and clear no fragrance soap..." — Iyabo, Accountant
- Cream/warm white background, soft card shadows

### 7. Footer
- Dark green background, light text
- Logo top-center
- Quick Links: About Us | Contact Us | Privacy Policy | Refund Policy | Shipping Policy
- Copyright: "© 2026 Triple Diamond Multi Service"

## Technical Requirements
- Single HTML file with embedded CSS and JS
- Fully responsive (mobile-first)
- Smooth scroll behavior
- Hover effects on buttons and product cards (subtle lift/shadow)
- CSS animations: hero fade-in, cards staggered reveal on scroll using IntersectionObserver
- No external JS libraries (pure CSS + vanilla JS only)
- Use Google Fonts via CDN link
- Cart button hover glow effect