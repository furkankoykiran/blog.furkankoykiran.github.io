---
title: "awesome-trending-repos: Auto-Tracking GitHub Trending"
description: "Automated daily tracking of trending repositories with GitHub Actions"
date: 2026-03-17 17:00:00 +0300
categories: [GitHub, Automation]
tags: [github-actions, automation, nodejs, trending]
image:
  path: /assets/img/posts/2026-03-18-awesome-trending-repos/banner.jpg
  alt: "Awesome Trending Repos"
---

Every day, hundreds of new projects appear on GitHub's trending list. However, tracking these projects and determining which ones are truly important can be time-consuming. To solve this problem, I developed the [awesome-trending-repos](https://github.com/furkankoykiran/awesome-trending-repos) project.

![GitHub Actions](/assets/img/posts/2026-03-18-awesome-trending-repos/github-actions.png)
*Daily automated updates with GitHub Actions*

---

## What is the Project?

awesome-trending-repos is an automatically updated list that tracks projects from GitHub's trending page daily. The project runs every day at midnight UTC, analyzes trending repos, and writes the results to the README.md file.

**Key features:**
- 🔄 Daily automatic updates
- 📊 Historical comparisons and ranking changes
- 📈 ASCII charts and trend visualizations
- 🏆 Fastest growing repositories (rising stars)
- 💾 7-day historical data storage

---

## How Does It Work?

The project runs automatically using GitHub Actions. Every day at midnight (UTC), a workflow is triggered, scrapes the GitHub trending page, analyzes the data, and writes the results to the README.md file.

**Data collection process:**
1. Scrape GitHub trending page (with Cheerio)
2. Get additional data from GitHub Search API (as fallback)
3. Categorize projects by programming language
4. Compare with historical data
5. Automatically update README.md

![GitHub Actions](/assets/img/posts/2026-03-18-awesome-trending-repos/github-actions.png)
*Automatic workflow management with CI/CD pipeline*

---

## Features

### Daily Automation

It runs automatically every day using GitHub Actions. No manual operation required. The system pulls trending data every day at midnight UTC and updates README.md.

### Historical Tracking

The project stores the last 7 days of data. This way you can see how a project's position in the trending list has changed. It provides detailed information about new entries, rising projects, and declining ones.

### Language-Based Ranking

It categorizes trending projects by programming language. You can track projects in popular languages like JavaScript, Python, TypeScript, Go, Rust separately.

### Visualization

With ASCII charts, you can visually track the rise and fall of projects. You can see from the charts how many stars projects have received and how fast they're growing.

---

## Technical Details

The project is written in Node.js and uses ES Modules. It was automated with GitHub Actions and integrated with GitHub API using the Octokit library.

**Technologies used:**
- Node.js 20+
- GitHub Actions (cron-based scheduling)
- Octokit (GitHub API client)
- Cheerio (web scraping)
- Axios (HTTP requests)

---

## Conclusion

With the awesome-trending-repos project, you can automatically track trending projects on GitHub. With the daily updated list, you won't miss the newest and most popular projects.

The project creates a valuable resource both for personal use and for the developer community. It saves time through automation and makes tracking trends easier.

![Furkan Köykıran](/assets/img/avatar.png)
*Furkan Köykıran - Senior Software Engineer*

---

## Other Blog Posts

- [My Open Source Contributions to MCP Ecosystem and AI Platforms](/posts/mcp-ai-contributions/)
- [Dev.to MCP Server Development and Publishing](/posts/devto-mcp-server-gelistirme-ve-yayinlama/)
