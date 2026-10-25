# Fullstack Blog

**One-line summary:** An English-language collection of practical engineering articles.

## Table of contents

1. [About](#1-about)
2. [Browse posts](#2-browse-posts)
3. [Categories](#3-categories)
4. [Add a post](#4-add-a-post)

## 1. About

This repository keeps English editions of selected technical articles for reading and reference.

## 2. Browse posts

- [Post index](posts/README.md)
- [Social login with a backend callback, PKCE, and an httpOnly cookie session](posts/2026-10-20-social-login-backend-callback-pkce-httponly-cookie-session.md)
- [One verification script instead of dozens of `curl` commands](posts/2026-10-19-verify-api-script.md)
- [Axios interceptors and sliding sessions for refresh tokens](posts/2026-10-18-axios-interceptors-sliding-session-refresh-token.md)

## 3. Categories

Posts are grouped by their main topic in the [post index](posts/README.md). The category structure is intentionally small and can grow when more articles are added:

- Backend
  - API testing and verification
- Full Stack
  - Authentication
    - Social login and OAuth
    - Token refresh and sessions

## 4. Add a post

1. Start from [`templates/post-template.md`](templates/post-template.md).
2. Save the article under `posts/` with a date and a short, descriptive kebab-case slug in the filename.
3. Update the [post index](posts/README.md) with the title and category.
4. Check that section links work and that every fenced code block has a language tag.
