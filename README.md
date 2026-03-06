# mattcarp12.github.io

## Quickstart Guide

This blog is built with [Jekyll](https://jekyllrb.com/) and configured to run in an isolated environment using VS Code Dev Containers.

### Prerequisites

To run this project locally without managing Ruby versions on your host machine, ensure you have the following installed:

* [Docker Engine](https://docs.docker.com/engine/install/) (running via Docker Desktop or natively in WSL2)
* [Visual Studio Code](https://code.visualstudio.com/)
* The [Dev Containers extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) for VS Code

### 1. Launch the Development Environment

1. Clone this repository and open the project folder in VS Code.

   ```bash
   git clone https://github.com/mattcarp12/mattcarp12.github.io.git
   cd mattcarp12.github.io
   code .
   ```

2. When prompted by VSCode, click **Reopen in Container**.
3. Wait for the container to build. All necessary dependencies, including Ruby and Jekyll, are handled automatically by the container configuration.

### 2. Run the Local Server

```Bash
bundle exec jekyll serve --livereload
```

You can now preview the site at <http://localhost:4000>. The `--livereload` flag ensures the page refreshes automatically whenever you save a file.

### 3. Create a New Post
Blog posts are written in Markdown and live in the _posts directory.

Create a new file in _posts using the strict naming convention: YYYY-MM-DD-title-of-post.md (e.g., 2026-03-06-hello-world.md).

At the very top of your new file, include the following YAML front matter:

```YAML
---
layout: post
title: "Your Post Title Here"
date: 2026-03-06 12:00:00 -0700
categories: [general]
---
```

Write your article content directly below the second ---.

### 4. Deploy to Production

This site is hosted on GitHub Pages. Deployment is fully automated. Simply commit and push your changes to the main branch, and GitHub Actions will build and deploy your new content to the live site.

```Bash
git add .
git commit -m "Add new blog post"
git push origin main
```
