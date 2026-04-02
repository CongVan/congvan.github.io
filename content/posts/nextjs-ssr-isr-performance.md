---
title: "How We Improved PageSpeed from 56 to 80+ with Next.js SSR/ISR"
date: 2026-04-02T02:00:00+07:00
draft: false
tags: ["Frontend", "DevOps"]
categories: ["Engineering"]
summary: "Practical strategies for improving Core Web Vitals on a high-traffic education portal using Next.js rendering strategies."
ShowToc: true
---

At Galaxy Education, I led the frontend architecture for icantech.vn -- a high-traffic education portal. When I joined, the Google PageSpeed score was **56**. We got it above **80** using a combination of Next.js rendering strategies.

## The Problem

The existing implementation was doing too much client-side rendering. Large JavaScript bundles, layout shifts, and slow initial paints made the site feel sluggish, especially on mobile.

## The Strategy

### 1. Server-Side Rendering for Dynamic Pages

Course listings, tutor profiles, and search results needed fresh data. We used `getServerSideProps` with careful caching headers:

```typescript
export const getServerSideProps: GetServerSideProps = async (ctx) => {
  ctx.res.setHeader(
    'Cache-Control',
    'public, s-maxage=60, stale-while-revalidate=300'
  );
  
  const courses = await fetchCourses(ctx.query);
  return { props: { courses } };
};
```

The `stale-while-revalidate` pattern was key -- users get a cached response instantly while the server refreshes data in the background.

### 2. Incremental Static Regeneration for Content Pages

Blog posts, landing pages, and course detail pages changed infrequently. ISR was perfect:

```typescript
export const getStaticProps: GetStaticProps = async ({ params }) => {
  const course = await getCourseBySlug(params.slug);
  return {
    props: { course },
    revalidate: 300, // Regenerate every 5 minutes
  };
};
```

### 3. Image Optimization

Switching to `next/image` with proper `sizes` attributes and lazy loading had the single biggest impact on LCP (Largest Contentful Paint).

## Results

- PageSpeed: **56 -> 80+**
- LCP: **4.2s -> 1.8s**
- CLS: Virtually eliminated with proper image dimensions

The lesson: rendering strategy choice should be driven by data freshness requirements, not convention.
