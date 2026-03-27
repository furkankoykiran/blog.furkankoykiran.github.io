---
title: "awesome-trending-repos: Modern Web Interface for GitHub Trending"
description: "A daily-updated trending repos website with React, Vite and GitHub Pages"
date: 2026-03-27 18:00:00 +0300
categories: [Web Development, GitHub, React]
tags: [react, vite, github-pages, github-actions, tailwindcss]
image:
  path: /assets/img/posts/2026-03-27-awesome-trending-repos/og-image.png
  alt: "Awesome Trending Repos"
---

In my previous blog post, I introduced the [awesome-trending-repos](https://github.com/furkankoykiran/awesome-trending-repos) project. Back then, the project only wrote data to a README.md file. Things have changed. The project is now a fully functional modern web application.

![CI/CD Pipeline](/assets/img/posts/github-actions-cicd-pipeline-diagram.png)
*Automated build and deployment process with GitHub Actions*

---

## Why a Web Interface?

A README.md file works, but it's static. To track trending projects effectively, I wanted a more interactive experience. Users needed to be able to filter by programming language, search repositories, and see data visualized with charts.

So I built a modern single-page application (SPA) with React + Vite.

---

## Tech Stack

**Frontend:**
- React 19 - Latest version
- Vite - Fast dev server and build tool
- Tailwind CSS v4 - Utility-first styling
- Framer Motion - Animations
- Recharts - Data visualization

**Backend/Infrastructure:**
- GitHub Actions - Automation
- GitHub Pages - Hosting
- Custom Domain - furkankoykiran.com.tr

---

## Project Structure

The core project structure:

```text
awesome-trending-repos/
├── src/
│   ├── App.jsx          # Main React component
│   ├── main.jsx         # Entry point
│   └── index.css        # Tailwind + custom styles
├── scripts/
│   └── update.js        # Trending data scrape script
├── public/
│   ├── data/
│   │   └── trending.json # Frontend data source
│   └── logo.png
├── .github/workflows/
│   └── update-trends.yml # CI/CD pipeline
└── vite.config.js       # GitHub Pages config
```

---

## GitHub Pages Integration

When using Vite with GitHub Pages, the most important configuration is the `base` path. If your project runs in a subdirectory (e.g., `github.io/repo-name`), you must specify this in `vite.config.js`:

```javascript
// vite.config.js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import tailwindcss from '@tailwindcss/vite';

export default defineConfig({
  plugins: [react(), tailwindcss()],
  base: '/awesome-trending-repos/', // GitHub Pages subdirectory
});
```

Without this setting, asset paths will break in production and you'll get 404 errors.

---

## GitHub Actions Workflow

The workflow runs every day at midnight UTC:

1. **Fetch trending data** - Scrape GitHub Trending page
2. **Build frontend** - Create dist/ folder with `npm run build`
3. **Commit data** - Commit README and data files if changed
4. **Deploy** - Distribute to GitHub Pages

```yaml
# .github/workflows/update-trends.yml
name: Update and Deploy Trends

on:
  schedule:
    - cron: '0 0 * * *'  # Daily at midnight
  workflow_dispatch:

permissions:
  contents: write
  pages: write
  id-token: write

jobs:
  update-and-build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Run update script
        run: npm run update
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Build frontend
        run: npm run build

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: './dist'

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: update-and-build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

![Deployment Flow](/assets/img/posts/cicd-deployment-automation-flow.png)
*Automated deployment flow*

---

## Frontend Features

### 1. Language Filtering

Users can filter by programming language:

```javascript
const languages = ['All', ...new Set(repos.map(r => r.language).filter(Boolean))];

const filteredRepos = repos.filter(repo => {
  const matchesLang = filter === 'All' || repo.language === filter;
  const matchesSearch = repo.name.toLowerCase().includes(searchQuery.toLowerCase());
  return matchesLang && matchesSearch;
});
```

### 2. Search Function

Real-time search across repo name, owner, and description.

### 3. Visual Statistics

Recharts library for language distribution bar chart and growth leaders list:

```javascript
<ResponsiveContainer width="100%" height="100%">
  <BarChart data={data?.insights?.topLanguages || []}>
    <CartesianGrid strokeDasharray="8 8" stroke="#ffffff03" />
    <XAxis dataKey="language" stroke="#475569" />
    <YAxis stroke="#475569" />
    <Tooltip contentStyle={{ backgroundColor: 'rgba(15, 23, 42, 0.9)' }} />
    <Bar dataKey="count" radius={[12, 12, 0, 0]}>
      {data?.insights?.topLanguages?.map((entry, index) => (
        <Cell key={index} fill={getLangColor(entry.language)} />
      ))}
    </Bar>
  </BarChart>
</ResponsiveContainer>
```

### 4. Deployment Feed

A feed entry is created for each update. Users can see the deployment history.

---

## Custom Domain Setup

To use a custom domain with GitHub Pages:

1. **Create CNAME file** - Add your domain to `public/CNAME`:
   ```
   furkankoykiran.com.tr
   ```

2. **DNS setting** - At your domain provider:
   ```
   CNAME furkankoykiran.github.io
   ```

3. **HTTPS** - GitHub Pages automatically provides SSL with Let's Encrypt.

---

## Performance Tips

### 1. Asset Optimization

Vite production build automatically does minification and tree-shaking. For large images, consider using optimized images instead of putting everything in `public/`.

### 2. Lazy Loading

For large lists, consider virtual scrolling. Not needed for this project with 25 repos, but good to know if you scale.

### 3. Caching

GitHub Pages automatically adds cache headers. For `index.html`, use cache-busting:

```javascript
// vite.config.js
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        entryFileNames: `assets/[name].[hash].js`,
        chunkFileNames: `assets/[name].[hash].js`,
        assetFileNames: `assets/[name].[hash].[ext]`
      }
    }
  }
});
```

---

## Live Demo

The project is currently live at [furkankoykiran.com.tr/awesome-trending-repos](https://furkankoykiran.com.tr/awesome-trending-repos/). It updates automatically every day.

![Project Logo](/assets/img/posts/2026-03-27-awesome-trending-repos/logo.png)
*Project logo*

![OG Image](/assets/img/posts/2026-03-27-awesome-trending-repos/og-image.png)
*Open Graph preview image*

---

## Conclusion

Moving from a README.md to a modern web application significantly improved the project's usability. Users now have an interactive experience instead of just a static list.

The GitHub Pages + Vite combination is an excellent solution for static sites. Free, fast, and fully automated with CI/CD.

The project is fully open source. If you want to contribute, send a pull request to the [GitHub repo](https://github.com/furkankoykiran/awesome-trending-repos).

![Furkan Köykıran](/assets/img/avatar.png)
*Furkan Köykıran - Senior Software Engineer*

---

## Other Blog Posts

- [Python Project Management: Poetry, uv and Modern Standards](/posts/python-proje-yonetimi-poetry-uv/)
- [Traefik v3: Reverse Proxy Solution for Docker Era](/posts/traefik-reverse-proxy-ssl-yonetimi/)
- [My Open Source Contributions to MCP Ecosystem and AI Platforms](/posts/mcp-ai-contributions/)
