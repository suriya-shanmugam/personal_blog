# Hugo Website – Blowfish Theme Setup Guide

This README provides steps to set up and work with a Hugo website using the Blowfish theme, including prerequisites, project setup, and commonly used commands.

---

# Prerequisites (First-Time Setup)

Ensure the following tools are installed:

## 1. Install Git

```bash
git --version
```

If not installed:

* macOS: `brew install git`
* Ubuntu: `sudo apt install git`

---

## 2. Install Go

```bash
go version
```

Install:

* macOS: `brew install go`
* Ubuntu: `sudo apt install golang`

---

## 3. Install Hugo (Extended Version Recommended)

```bash
hugo version
```

Install:

```bash
brew install hugo
```

---

## 4. Install Dart Sass

```bash
sass --version
```

Install:

```bash
brew install sass/sass/sass
```

---

# Project Setup

## 1. Create a New Hugo Site

```bash
hugo new site my-site
cd my-site
```

---

## 2. Initialize Git

```bash
git init
```

---

## 3. Add Blowfish Theme as Submodule

```bash
t
```

---

## 4. Update Configuration

Edit `hugo.toml`:

```toml
baseURL = "https://example.com"
languageCode = "en-us"
title = "My Hugo Site"
theme = "blowfish"
```

---

## 5. Copy Example Site Configuration

```bash
cp -r themes/blowfish/exampleSite/* .
```

---

# Content Creation

## Create a New Post

```bash
hugo new posts/my-first-post.md
```

Example content:

```markdown
---
title: "My First Post"
date: 2026-03-27
draft: true
---

Welcome to my Hugo site.
```

---

# Run Locally

```bash
hugo server -D
```

Open in browser:

```
http://localhost:1313
```

---

# Build for Production

```bash
hugo
```

Output directory:

```
/public
```

---

# Git Submodule Commands

## Clone repository with submodules

```bash
git clone --recurse-submodules <your-repo-url>
```

## Initialize submodules (if already cloned)

```bash
git submodule update --init --recursive
```

## Update Blowfish theme

```bash
cd themes/blowfish
git pull origin main
cd ../..
```

---

# Useful Hugo Commands

## Run server with drafts

```bash
hugo server -D
```

## Include future posts

```bash
hugo server -D -F
```

## Clean build

```bash
hugo --cleanDestinationDir
```

## Minified production build

```bash
hugo --minify
```

## Watch for changes

```bash
hugo server --watch
```

---

# Project Structure

```
my-site/
├── content/
├── layouts/
├── static/
├── themes/blowfish
├── hugo.toml
```

---

# Quick Start Summary

```bash
hugo new site my-site
cd my-site

git init

git submodule add https://github.com/nunocoracao/blowfish.git themes/blowfish

cp -r themes/blowfish/exampleSite/* .

hugo server -D
```

