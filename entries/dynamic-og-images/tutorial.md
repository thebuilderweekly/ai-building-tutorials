## Opening thesis

You will build dynamic Open Graph image generation for a Next.js site using Vercel's `@vercel/og` library. Every page on your site will produce a unique 1200x630 image at build time, branded consistently, containing the page's title, category, and date. An agent generates a unique OG image per page in milliseconds, which a human designing them by hand would need a graphic design tool and an hour per piece. By the end, every link you or anyone else shares from your site will display a card that looks like it was made for that specific page, because it was.

## Before

You publish a new article. You share it on Twitter. The card shows your site logo, your site name, and your site tagline. The same card showed up last week on a different article. The same card will show up next week. People scroll past because the card tells them nothing about the specific piece. Click-through rate from social hovers around 0.8%. You know good cards drive clicks. You looked at how Stratechery, The Verge, and Substack handle social cards and they all use unique-per-page imagery with the article title rendered at large size. You opened Figma. You designed one card. It took 35 minutes. You have 80 articles. You closed Figma. You went back to publishing with the generic card.

## Architecture

The system has three components: a shared OG image template that defines the visual style, a per-route OG image function that fills the template with page-specific data, and Next.js App Router's built-in `opengraph-image.tsx` convention that automatically wires generated images to each route's metadata. The image is generated at build time (or on-demand for dynamic routes) and cached at the edge.

```text
DIAGRAM: Dynamic OG image generation pipeline
Caption: Per-route OG image functions render unique cards using a shared template
Nodes:
1. Page route - Each page in the Next.js app, e.g. /articles/[slug]
2. Page metadata - Title, category, date, author from the page's data layer
3. Shared template function - Returns JSX rendering the visual layout
4. Per-route opengraph-image.tsx - Reads page data, calls shared template, returns ImageResponse
5. Vercel @vercel/og library - Renders JSX to PNG using Satori
6. Edge cache - Caches generated images at Vercel's CDN
7. Social platform crawler - Twitter, LinkedIn, Slack scrapers fetch the cached image
8. Final card display - Shown to users in social feeds and chat previews
Flow:
- Page route exists with metadata in its data layer
- Per-route opengraph-image.tsx reads the page metadata
- Per-route function calls Shared template function with page-specific data
- Shared template returns JSX layout
- Vercel @vercel/og renders the JSX to a 1200x630 PNG
- Edge cache stores the PNG at the route's opengraph-image URL
- Social platform crawler fetches the PNG when someone shares the page URL
- Final card displays the unique branded image
```

## Step-by-step implementation

### Step 1: Verify your Next.js version

You need Next.js 13.3 or later for the App Router's automatic OG image convention. Check your version.

```bash
cat package.json | grep '"next"'
# Should show "next": "^13.3.0" or higher. If older, upgrade:
npm install next@latest
```

### Step 2: Install the OG image library

The `@vercel/og` library handles JSX-to-PNG rendering. It works in the Edge runtime, which is required for Next.js's `opengraph-image.tsx` convention.

```bash
npm install @vercel/og
```

### Step 3: Build the shared OG image template

A shared template ensures every card across the site looks consistent. It accepts the variables that change per page (title, content type, date) and returns the JSX layout. Save this at `lib/og/template.tsx`.

```tsx
// lib/og/template.tsx
import { ImageResponse } from "@vercel/og";

interface OGImageProps {
  title: string;
  contentType: "Article" | "Tutorial" | "Weekly" | "Monthly";
  subLabel?: string; // e.g. date, category, or volume
}

export function renderOGImage({ title, contentType, subLabel }: OGImageProps) {
  return new ImageResponse(
    (
      <div style={{ display: "flex", flexDirection: "column", justifyContent: "space-between", width: "100%", height: "100%", padding: "72px", background: "#F8F9F7" }}>
        <div style={{ display: "flex", fontSize: "18px", fontWeight: 500, textTransform: "uppercase", letterSpacing: "0.18em", color: "#000" }}>
          {contentType}
        </div>

        <div style={{ display: "flex", fontSize: "72px", fontWeight: 500, lineHeight: 1.02, letterSpacing: "-0.02em", color: "#000", maxWidth: "1056px" }}>
          {title}
        </div>

        <div style={{ display: "flex", justifyContent: "space-between", fontSize: "20px", fontWeight: 500, textTransform: "uppercase", letterSpacing: "0.14em", color: "#000" }}>
          <div style={{ display: "flex" }}>
            {subLabel || ""}
          </div>
          <div style={{ display: "flex" }}>
            yoursite.com
          </div>
        </div>
      </div>
    ),
    {
      width: 1200,
      height: 630,
    }
  );
}
```

### Step 4: Create per-route OG image functions

Next.js App Router automatically uses any `opengraph-image.tsx` file inside a route as that route's OG image generator. Create one for articles. Save at `app/articles/[slug]/opengraph-image.tsx`.

```tsx
// app/articles/[slug]/opengraph-image.tsx
import { renderOGImage } from "@/lib/og/template";
import { getArticleBySlug } from "@/lib/content/articles";

export const runtime = "edge";
export const alt = "Article social card";
export const size = { width: 1200, height: 630 };
export const contentType = "image/png";

export default async function ArticleOGImage({
  params,
}: {
  params: { slug: string };
}) {
  const article = await getArticleBySlug(params.slug);

  return renderOGImage({
    title: article.title,
    contentType: "Article",
    subLabel: new Date(article.publishedAt).toLocaleDateString("en-US", {
      month: "long",
      day: "numeric",
      year: "numeric",
    }),
  });
}
```

### Step 5: Repeat for other content types

Each content type gets its own `opengraph-image.tsx` file inside the route directory. The template is shared; only the data fetching differs.

```tsx
// app/tutorials/[slug]/opengraph-image.tsx
import { renderOGImage } from "@/lib/og/template";
import { getTutorialBySlug } from "@/lib/content/tutorials";

export const runtime = "edge";
export const alt = "Tutorial social card";
export const size = { width: 1200, height: 630 };
export const contentType = "image/png";

export default async function TutorialOGImage({
  params,
}: {
  params: { slug: string };
}) {
  const tutorial = await getTutorialBySlug(params.slug);

  return renderOGImage({
    title: tutorial.title,
    contentType: "Tutorial",
    subLabel: tutorial.pillar.toUpperCase(),
  });
}
```

```tsx
// app/weekly/[slug]/opengraph-image.tsx
import { renderOGImage } from "@/lib/og/template";
import { getWeeklyBySlug } from "@/lib/content/weekly";

export const runtime = "edge";
export const alt = "Weekly social card";
export const size = { width: 1200, height: 630 };
export const contentType = "image/png";

export default async function WeeklyOGImage({
  params,
}: {
  params: { slug: string };
}) {
  const weekly = await getWeeklyBySlug(params.slug);

  return renderOGImage({
    title: weekly.title,
    contentType: "Weekly",
    subLabel: `VOL ${weekly.volumeNumber}`,
  });
}
```

### Step 6: Add a homepage OG image

The root `app/opengraph-image.tsx` handles the homepage card. This one uses static content rather than per-route data.

```tsx
// app/opengraph-image.tsx
import { renderOGImage } from "@/lib/og/template";

export const runtime = "edge";
export const alt = "Site homepage card";
export const size = { width: 1200, height: 630 };
export const contentType = "image/png";

export default async function HomepageOGImage() {
  return renderOGImage({
    title: "Your site tagline goes here.",
    contentType: "Article",
    subLabel: "yoursite.com",
  });
}
```

### Step 7: Verify the OG meta tags are emitted

Next.js automatically generates the `<meta property="og:image">` tag when an `opengraph-image.tsx` file exists. Verify by inspecting the page source.

```bash
npm run build
npm run start

# In another terminal
curl -s http://localhost:3000/articles/some-slug | grep "og:image"
# Expected: <meta property="og:image" content="...opengraph-image" />
```

Visit the OG image URL directly in a browser to confirm it renders.

```bash
open http://localhost:3000/articles/some-slug/opengraph-image
```

### Step 8: Test the cards in real social platforms

After deploying, use these official validators to see how each platform will render your card.

```bash
# Twitter Card Validator
echo "https://cards-dev.twitter.com/validator"

# Facebook Sharing Debugger
echo "https://developers.facebook.com/tools/debug/"

# LinkedIn Post Inspector
echo "https://www.linkedin.com/post-inspector/"
```

Paste a URL from your live site into each. Confirm the card displays correctly. If the card shows the old generic image, click "Scrape Again" or equivalent to force a re-fetch.

## Breakage

Skip the per-route image functions. Set a single static image as the site-wide OG image in your root layout. Every page now shares the same card. The card looks fine for the homepage. It looks generic for every article. People scrolling Twitter see your link and skip it because there is nothing specific to draw them in. Your share rate is identical to having no card. Worse, you cannot tell from the share preview whether the link is to an article, a tutorial, or a podcast. The card carries no information about what the link is.

```text
DIAGRAM: Static OG image failure
Caption: One static image used across all pages produces flat social engagement
Nodes:
1. Page routes - Many different pages on the site
2. Root layout - Sets a single static og:image
3. Social platforms - Show the same card for every link
4. Users - Scroll past because nothing distinguishes one share from another
Flow:
- Page routes all reference the same root layout
- Root layout sets a fixed og:image URL
- Social platforms fetch the static image for every link
- Every share looks identical regardless of content
- Users scroll past because the card carries no specific information
```

## The fix

The fix is the per-route `opengraph-image.tsx` file from Steps 4 through 6. Each route's file fetches the page-specific data and calls the shared template with that data. The Next.js App Router convention automatically routes the generated image to the page's metadata. The critical pattern is that the shared template stays the same while the per-route data varies, isolated below.

```tsx
// The pattern that makes every page unique while keeping the visual consistent
import { renderOGImage } from "@/lib/og/template";
import { getArticleBySlug } from "@/lib/content/articles";

export const runtime = "edge";
export const alt = "Article social card";
export const size = { width: 1200, height: 630 };
export const contentType = "image/png";

export default async function ArticleOGImage({ params }) {
  // Per-page data fetch
  const article = await getArticleBySlug(params.slug);

  // Shared template, page-specific inputs
  return renderOGImage({
    title: article.title,
    contentType: "Article",
    subLabel: new Date(article.publishedAt).toLocaleDateString("en-US", {
      month: "long",
      day: "numeric",
      year: "numeric",
    }),
  });
}
```

Two pieces matter. First, the template is one function used everywhere, so the brand stays consistent. Second, the per-route file fetches that route's specific data, so the title and sub-label match the page being shared. Add a new content type, write one new `opengraph-image.tsx` file with the relevant data fetcher, and the system covers it automatically.

## Fixed state

```text
DIAGRAM: Per-route OG image system with shared template
Caption: Every route generates a unique card using consistent visual branding
Nodes:
1. Articles route - Has its own opengraph-image.tsx
2. Tutorials route - Has its own opengraph-image.tsx
3. Weekly route - Has its own opengraph-image.tsx
4. Monthly route - Has its own opengraph-image.tsx
5. Homepage - Has its own opengraph-image.tsx
6. Shared template (lib/og/template.tsx) - One source of visual truth
7. @vercel/og library - Renders each route's JSX to PNG
8. Edge cache - Stores PNGs at each route's URL
9. Social platforms - Fetch unique cards per shared URL
10. Users - See specific, branded cards in their feeds
Flow:
- Each route directory contains an opengraph-image.tsx file
- Each file fetches that route's specific page data
- Each file calls the Shared template with page-specific inputs
- Shared template returns JSX with consistent visual style
- @vercel/og renders the JSX to PNG
- Edge cache stores the PNG at the route's URL
- Social platforms fetch the unique PNG when a URL is shared
- Users see a branded card specific to the shared content
```

## After

You publish a new article. You share it on Twitter. The card shows the article title rendered at 72px in your serif font, the word "ARTICLE" in small caps above it, the publication date below, and your site name in the corner. Someone scrolling sees the title and stops. They click. Your social click-through rate over the next month climbs from 0.8% to 2.3%. You add a new tutorial. The card automatically generates with "TUTORIAL" as the content type and the pillar name as the sub-label. You did not design the card. The same template you wrote in 30 minutes for articles handles every new piece you publish, in any content type, forever. You add a new content type called "research." You write one `opengraph-image.tsx` file. The cards work from the first publication.

## Takeaway

The pattern is one shared template plus per-route data fetching. The visual identity stays consistent across the site because one function controls the layout. The specificity comes from each route's data. This applies anywhere you need consistent visual output that varies by content: email signatures, document headers, certificate generators, social cards, even printable PDFs. Centralize the design, distribute the data, render at the edge. Every share becomes an asset rather than a generic placeholder.
