# Code Review: best-resume-ever

## Overview
This repository provides a Vue 2 based application to generate resumes from a `data.yml` file and export them to PDF using Puppeteer. Overall, the idea is excellent and easy to use, but there are significant security, maintainability, and architecture issues—largely stemming from outdated dependencies and unsafe data parsing.

---

## 🔴 CRITICAL

### 1. Remote Code Execution (RCE) via `js-yaml`
- **File:** `src/resumes/options.js` (Line 13)
- **Impact:** The code uses `yaml.load(PERSON)` to parse `resume/data.yml`. The `js-yaml` version used is `3.13.1`. In this version, `yaml.load()` is unsafe by default and can execute arbitrary JavaScript (e.g., via `!!js/function` tags). Since `data.yml` is user-provided, if someone renders a resume from an untrusted YAML source, it leads to immediate RCE on their machine.
- **Fix Suggestion:** Change `yaml.load(PERSON)` to `yaml.safeLoad(PERSON)`. Additionally, upgrade `js-yaml` to v4+ where `load()` is safe by default.

### 2. Vue 2 End-of-Life (EOL) & Massively Outdated Dependencies
- **File:** `package.json`
- **Impact:** The project relies on Vue 2.5.17, which is EOL and receives no security updates. Furthermore, `npm audit` reports over 200 vulnerabilities (dozens critical/high) from obsolete Webpack, Babel, and loader dependencies.
- **Fix Suggestion:** Migrate the project to Vue 3 and Vite (replacing Webpack 4). This will eliminate nearly all devDependency vulnerabilities, vastly improve build performance, and future-proof the codebase.

---

## 🟠 HIGH

### 1. Puppeteer Sandbox Disabled
- **File:** `scripts/export.js` (Lines 58-59)
- **Impact:** Puppeteer is launched with `args: ['--no-sandbox']`. Disabling the sandbox on a headless browser is highly dangerous, particularly if the rendered HTML contains user-provided data from the YAML file that might include unescaped HTML/XSS.
- **Fix Suggestion:** Remove the `--no-sandbox` argument. If running in a containerized environment (like Docker) that requires it, handle it via environment variables while keeping the sandbox enabled by default.

### 2. Lack of Unit Testing & Component Tests
- **File:** Entire repository (`test/`)
- **Impact:** The repository currently relies solely on TestCafe for superficial E2E rendering checks (`test/e2e/resumes.js`). There are zero unit tests for data normalization in `src/resumes/options.js` or the individual Vue components. If a user has a missing field (e.g., `person.contact.github`), components often fail silently or display broken UI.
- **Fix Suggestion:** Introduce a test runner like Vitest or Jest. Write unit tests for `getVueOptions` to ensure missing data fallbacks work, and add basic component tests for the layouts.

---

## 🟡 MEDIUM

### 1. Poor Error Handling in Export Scripts
- **File:** `scripts/export.js` & `scripts/preview.js`
- **Impact:** The `convert()` functions use `forEach(async (dir) => { ... })` and a naive `try/catch` around the block. Because `forEach` doesn't await inner promises, errors in page generation are swallowed or lead to unhandled promise rejections, leaving zombie browser processes running.
- **Fix Suggestion:** Use a `for...of` loop with `await` or `Promise.all()` to map over directories, and ensure proper `try/catch/finally` blocks to explicitly call `await browser.close()` even if rendering fails.

### 2. Hardcoded Ports and Network Polling
- **File:** `scripts/export.js` (Lines 18, 30, 62)
- **Impact:** The script polls `http://localhost:${config.dev.port}/#/` with a custom RxJS polling loop to wait for the dev server. If the port is in use or the Dev Server fails, the RxJS loop might hang indefinitely or fail obscurely.
- **Fix Suggestion:** Use an established tool like `wait-on` to block until the server is ready. Also, allow configuring the host/port via environment variables.

### 3. High Bundle Size due to `@import` of Global Fonts/Icons
- **File:** `src/App.vue` (Lines 11-18)
- **Impact:** The application pulls in raw `.css` files for multiple font families (Roboto, Source Sans Pro, Open Sans, Raleway) and heavy icon packs (FontAwesome, Material Icons). This heavily slows down the initial bundle compilation and browser load time.
- **Fix Suggestion:** Only bundle the specific fonts required by the chosen template, or lazy-load the fonts within the specific resume components that need them.

---

## 🟢 LOW

### 1. Code Duplication in Resume Templates
- **File:** `src/resumes/*.vue`
- **Impact:** Every single resume layout duplicates HTML for the `contact`, `education`, and `experience` sections. Any new field added to `data.yml` requires modifying 15+ separate Vue components.
- **Fix Suggestion:** Extract reusable layout primitives (e.g., `<ContactSection />`, `<ExperienceSection />`) into shared components that accept `person` data as props, drastically reducing template boilerplate.

### 2. Developer Experience (DX) for Images
- **File:** `README.md` / `resume/` folder
- **Impact:** The developer must manually copy an image and explicitly name it `id.jpg`. If the image is a `.png` or named differently, it requires modifying the code.
- **Fix Suggestion:** Update `options.js` to look for `profile-pic.*` using a glob, or allow configuring the profile picture path inside `data.yml`.
