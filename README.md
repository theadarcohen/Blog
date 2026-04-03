# Portfolio & Blog — Jekyll + GitHub Pages

A clean, minimal portfolio site built with Jekyll and designed to work out of the box with GitHub Pages.

---

## Quick Start

### Deploy to GitHub Pages

1. **Create a repository** on GitHub named `yourusername.github.io` (replace `yourusername` with your actual GitHub username).
2. **Push this code** to the repository:
   ```bash
   git init
   git add .
   git commit -m "Initial commit"
   git branch -M main
   git remote add origin https://github.com/yourusername/yourusername.github.io.git
   git push -u origin main
   ```
3. **Enable GitHub Pages:** Go to your repository → Settings → Pages → Under "Source," select **Deploy from a branch** and choose `main` / `/ (root)`. Click Save.
4. **Wait 1-2 minutes**, then visit `https://yourusername.github.io` — your site is live!

### Run Locally (optional)

```bash
# Install dependencies (requires Ruby & Bundler)
bundle install

# Start the local server
bundle exec jekyll serve

# Open http://localhost:4000 in your browser
```

---

## Creating a New Project Post

Create a new Markdown file in the `_posts/` folder using this naming convention:

```
_posts/YYYY-MM-DD-your-post-title.md
```

### Front Matter Template

Copy and paste this into your new file:

```yaml
---
title: "Your Project Title"
description: "A one-sentence summary of the project."
date: 2026-01-15
thumbnail: "https://placehold.co/800x500/cccccc/333333?text=Project+Image"
tags: [Design, Development]
---

Your project write-up goes here in Markdown.
```

### Front Matter Fields

| Field         | Required | Description                                                |
|---------------|----------|------------------------------------------------------------|
| `title`       | Yes      | The project title (shown on cards and the post page)       |
| `description` | Yes      | Short summary (shown on home page cards)                   |
| `date`        | Yes      | Publication date in `YYYY-MM-DD` format                    |
| `thumbnail`   | Yes      | Path or URL to the card thumbnail image                    |
| `tags`        | No       | List of tags, e.g. `[Design, React, Case Study]`          |

---

## Adding Images

### Option 1: Local images

1. Place your image files in `assets/images/` (create the folder if needed).
2. Reference them in your post:
   ```markdown
   ![Alt text](/assets/images/my-image.png)
   ```
3. For thumbnails in front matter:
   ```yaml
   thumbnail: "/assets/images/my-thumbnail.png"
   ```

### Option 2: External images

Use any image URL directly:
```markdown
![Alt text](https://example.com/image.png)
```

---

## Customization

### Site Identity

Edit `_config.yml` to update your name, description, and social links:

```yaml
title: "Jane Smith"
description: "Product designer & frontend developer."
author:
  name: "Jane Smith"
  email: "jane@example.com"
  github: "janesmith"
  linkedin: "janesmith"
```

### Monogram / Logo

The header shows a two-letter monogram by default. To change the letters, edit `_includes/header.html` and update the text inside `<span class="monogram">`.

To replace it with an image, swap the `<span>` for an `<img>` tag.

### Accent Color

Change the accent color in `assets/css/style.css` by updating the CSS variable:

```css
--color-accent: #4A6CF7;       /* Your new accent color */
--color-accent-hover: #3b5de7; /* Slightly darker for hover states */
```

---

## File Structure

```
.
├── _config.yml          # Site configuration
├── _layouts/
│   ├── default.html     # Base HTML shell (head, header, footer)
│   ├── page.html        # Layout for standalone pages (About)
│   └── post.html        # Layout for project posts
├── _includes/
│   ├── header.html      # Site header with nav
│   └── footer.html      # Site footer with social links
├── _posts/              # Your project posts (Markdown)
│   ├── 2026-03-15-mobile-app-redesign.md
│   ├── 2026-03-25-api-dashboard.md
│   └── 2026-04-01-brand-identity-solara.md
├── assets/
│   └── css/
│       └── style.css    # All styles
├── about.md             # About page
├── index.html           # Home page
├── Gemfile              # Ruby dependencies
├── .gitignore
└── README.md
```

---

## License

This project is open source. Feel free to use it as a starting point for your own portfolio.
