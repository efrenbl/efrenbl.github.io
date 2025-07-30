# efrenbl.github.io

My personal website built with Jekyll and hosted on GitHub Pages.

## 🚀 Quick Start

### Prerequisites

- Ruby 2.7+ (recommended: 3.1+)
- Bundler
- Git

### Local Development

1. Clone the repository:
   ```bash
   git clone https://github.com/efrenbl/efrenbl.github.io.git
   cd efrenbl.github.io
   ```

2. Install dependencies:
   ```bash
   bundle install
   ```

3. Start the development server:
   ```bash
   bundle exec jekyll serve
   ```

4. Open your browser and navigate to `http://localhost:4000`

### Creating New Posts

1. Create a new file in the `_posts` directory with the format: `YYYY-MM-DD-title.md`
2. Add the required front matter:
   ```yaml
   ---
   layout: post
   title: "Your Post Title"
   date: YYYY-MM-DD HH:MM:SS -0000
   categories: [category1, category2]
   tags: [tag1, tag2, tag3]
   ---
   ```
3. Write your content in Markdown below the front matter

### Adding Pages

Create new `.md` files in the root directory or organize them in folders. Make sure to include the proper front matter:

```yaml
---
layout: page
title: "Page Title"
permalink: /page-url/
---
```

## 🏗️ Project Structure

```
.
├── _config.yml          # Jekyll configuration
├── _posts/              # Blog posts
├── _layouts/            # Page layouts (if custom)
├── _includes/           # Reusable components (if custom)
├── _sass/               # Sass stylesheets (if custom)
├── assets/              # Images, CSS, JS files
├── .github/workflows/   # GitHub Actions
├── index.md             # Homepage
├── about.md             # About page
├── blog.md              # Blog listing page
├── Gemfile              # Ruby dependencies
└── README.md            # This file
```

## 🎨 Customization

### Theme

This site uses the `minima` theme. You can:

- Customize colors and styles by overriding the theme's Sass variables
- Create custom layouts in the `_layouts` directory
- Add custom includes in the `_includes` directory

### Configuration

Edit `_config.yml` to customize:

- Site title and description
- Social media links
- Navigation menu
- SEO settings
- And much more!

## 🚀 Deployment

This site is automatically deployed to GitHub Pages using GitHub Actions. Every push to the `main` branch triggers a new build and deployment.

### Manual Deployment

If you prefer to deploy manually:

1. Build the site: `bundle exec jekyll build`
2. The generated site will be in the `_site` directory

## 📝 Content Guidelines

### Writing Posts

- Use clear, descriptive titles
- Add relevant tags and categories
- Include a brief excerpt or description
- Use proper Markdown formatting
- Add images to the `assets/images/` directory

### Markdown Tips

- Use `#` for headings
- Use `**text**` for bold, `*text*` for italic
- Create links with `[text](url)`
- Add images with `![alt text](image-url)`
- Use code blocks with triple backticks

## 🔧 Troubleshooting

### Common Issues

1. **Bundle install fails**: Make sure you have Ruby 2.7+ installed
2. **Site not updating**: Check that you're pushing to the `main` branch
3. **Images not showing**: Verify the image paths are correct and files exist

### Getting Help

- Check the [Jekyll documentation](https://jekyllrb.com/docs/)
- Review [GitHub Pages documentation](https://docs.github.com/en/pages)
- Open an issue in this repository

## 📄 License

This project is open source and available under the [MIT License](LICENSE).

## 🤝 Contributing

Feel free to open issues or submit pull requests if you have suggestions for improvements!

---

Built with ❤️ using Jekyll and GitHub Pages
