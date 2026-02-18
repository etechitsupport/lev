# Lev Kids Care — Cloudflare Pages Deployment

## Project Structure

```
levkidscare-pages/
├── public/
│   ├── index.html          # Main site (home, services, pricing, scheduling)
│   ├── 404.html            # Custom 404 page
│   ├── privacy.html        # HIPAA & Privacy Policy page
│   ├── _headers            # Security headers (HIPAA-compliant CSP, HSTS, etc.)
│   ├── _redirects           # URL redirects
│   ├── robots.txt          # SEO crawling rules
│   └── sitemap.xml         # SEO sitemap
└── README.md               # This file
```

## Deploy to Cloudflare Pages

### Option 1: Direct Upload (Quickest)

1. Go to [Cloudflare Dashboard](https://dash.cloudflare.com) → **Workers & Pages**
2. Click **Create application** → **Pages** → **Upload assets**
3. Name your project: `levkidscare`
4. Upload the entire `public/` folder
5. Click **Deploy site**
6. Set custom domain: `levkidscare.com`

### Option 2: Git Integration (Recommended for ongoing updates)

1. Push this project to a GitHub/GitLab repo
2. Go to Cloudflare Dashboard → **Workers & Pages** → **Create application**
3. Connect your Git repository
4. Configure build settings:
   - **Build command:** _(leave empty — static site)_
   - **Build output directory:** `public`
   - **Root directory:** `/` (or `levkidscare-pages` if nested)
5. Click **Save and Deploy**

### Option 3: Wrangler CLI

```bash
# Install Wrangler
npm install -g wrangler

# Login
wrangler login

# Deploy
wrangler pages deploy public --project-name=levkidscare
```

## Custom Domain Setup

1. In Cloudflare Pages project → **Custom domains**
2. Add `levkidscare.com` and `www.levkidscare.com`
3. If your domain is already on Cloudflare, DNS records are auto-configured
4. If not, update your domain's nameservers to Cloudflare's

## Post-Deployment Checklist

- [ ] Verify HTTPS is active (automatic with Cloudflare)
- [ ] Test all security headers at [securityheaders.com](https://securityheaders.com)
- [ ] Verify `_headers` and `_redirects` are working
- [ ] Test 404 page by visiting a non-existent URL
- [ ] Submit sitemap to Google Search Console
- [ ] Test mobile responsiveness

## Stripe Integration (Production Setup)

The scheduling form currently logs Stripe payloads to the console. For production:

### 1. Create a Cloudflare Pages Function for the backend

Create `functions/api/create-checkout.js`:

```javascript
export async function onRequestPost(context) {
  const stripe = require('stripe')(context.env.STRIPE_SECRET_KEY);
  const { booking } = await context.request.json();

  // Create or find Stripe customer
  const customer = await stripe.customers.create({
    name: booking.parentName,
    email: booking.parentEmail,
    phone: booking.parentPhone,
    metadata: {
      patient_name: `${booking.childFirst} ${booking.childLast}`,
      patient_dob: booking.childDob,
      hipaa_consent: 'true',
      practice: 'Lev Kids Care'
    }
  });

  // Create Checkout Session
  const session = await stripe.checkout.sessions.create({
    customer: customer.id,
    payment_method_types: ['card'],
    line_items: [{
      price_data: {
        currency: 'usd',
        product_data: {
          name: booking.visitType,
          description: `Appointment: ${booking.date} at ${booking.time}`
        },
        unit_amount: booking.price * 100
      },
      quantity: 1
    }],
    mode: 'payment',
    success_url: 'https://levkidscare.com/?booking=success',
    cancel_url: 'https://levkidscare.com/#schedule',
    metadata: {
      visit_type: booking.visitType,
      appointment_date: booking.date,
      appointment_time: booking.time,
      patient: `${booking.childFirst} ${booking.childLast}`,
      hipaa_compliant: 'true'
    }
  });

  return new Response(JSON.stringify({ url: session.url }), {
    headers: { 'Content-Type': 'application/json' }
  });
}
```

### 2. Set Environment Variables in Cloudflare

Go to Pages project → **Settings** → **Environment variables**:

| Variable | Value |
|---|---|
| `STRIPE_SECRET_KEY` | `sk_live_...` |
| `STRIPE_PUBLISHABLE_KEY` | `pk_live_...` |

### 3. HIPAA Compliance with Stripe

- Sign Stripe's [Business Associate Agreement (BAA)](https://stripe.com/docs/security/guide#hipaa)
- Enable Stripe's HIPAA-eligible environment
- Never log or store full card numbers
- Use Stripe Checkout or Elements (card data never touches your server)

## Security Headers Explained

The `_headers` file includes:

| Header | Purpose |
|---|---|
| `X-Frame-Options: DENY` | Prevents clickjacking |
| `X-Content-Type-Options: nosniff` | Prevents MIME sniffing |
| `Strict-Transport-Security` | Forces HTTPS |
| `Content-Security-Policy` | Controls allowed scripts/styles |
| `Referrer-Policy` | Limits referrer data leakage |
| `Permissions-Policy` | Restricts browser API access |

## SEO

- Schema.org `MedicalBusiness` structured data included
- Open Graph & Twitter Card meta tags
- Canonical URLs set
- Sitemap at `/sitemap.xml`
- `robots.txt` configured

## Tech Stack

- **Hosting:** Cloudflare Pages (global CDN, automatic HTTPS)
- **Frontend:** Vanilla HTML/CSS/JS (zero dependencies, fast load)
- **Fonts:** Playfair Display + Nunito + Quicksand (Google Fonts)
- **Payments:** Stripe Checkout (HIPAA-compliant with BAA)
- **Security:** HSTS, CSP, XSS protection headers
