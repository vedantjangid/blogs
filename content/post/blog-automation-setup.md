---
title: Setting Up a CI/CD Blog with Hugo, GitHub, Netlify, and a Custom Subdomain
date: 2024-12-02
hero: "/images/blog-setup-hero.jpg"
excerpt: A step-by-step guide on building a seamless blogging workflow using Hugo, GitHub, Netlify, and GoDaddy subdomains.
authors:
  - Vedant
---

Creating a blog can be a straightforward process, but automating it with a CI/CD pipeline elevates its efficiency and professionalism. This post describes how I set up a blogging workflow using Hugo, integrated it with GitHub and Netlify, and configured a custom subdomain using GoDaddy.

<!--more-->

## Overview

I built a CI/CD pipeline for my blog using **Hugo**, a fast and flexible static site generator. Here’s what I achieved:

1. Automated deployment and hosting via **GitHub** and **Netlify**.
2. A custom blog theme created and managed seamlessly.
3. A subdomain (`blog.vedant.cloud`) configured with **GoDaddy** to point to the Netlify-hosted site using a CNAME record.

This setup ensures that every update to my blog repository automatically triggers a rebuild and redeployment, delivering an efficient, streamlined workflow.

## Step-by-Step Guide

### 1. Installing Hugo

The first step was installing Hugo. On macOS, this can be done conveniently using Homebrew:

\`\`\`bash
brew install hugo
\`\`\`

After installation, I verified it using:

\`\`\`bash
hugo version
\`\`\`

### 2. Setting Up a New Blog

Once Hugo was installed, I created a new Hugo project:

\`\`\`bash
hugo new site blogs
\`\`\`

Next, I explored Hugo themes and downloaded a base theme that I customized to align with my style. I used Git for version control throughout the process:

\`\`\`bash
git submodule add https://github.com/<theme-repo-url> themes/hugo-theme
\`\`\`

### 3. Customizing the Blog Theme

Using Hugo's templates, I modified the blog's design. I customized layouts, color schemes, and content structures to give my blog a unique and personalized appearance. These changes were committed to a separate repository to ensure version control and reusability.

### 4. CI/CD Pipeline with GitHub and Netlify

I created a GitHub repository to store my blog's code and connected it to Netlify for hosting. Netlify automatically builds and deploys the site every time I push updates to the repository.

Netlify was configured as follows:

1. Link the GitHub repository to Netlify.
2. Define the build command (`hugo`) and specify the output directory (`public`).
3. Ensure Netlify’s build environment includes all necessary dependencies.

### 5. Configuring the Subdomain

Finally, I set up a subdomain (`blog.vedant.cloud`) using GoDaddy:

1. Create a CNAME record in GoDaddy's DNS settings.
2. Point the subdomain to Netlify’s domain provided during setup.
3. Verify the connection to ensure the subdomain properly redirects to the Netlify-hosted site.

### 6. Testing and Finalizing

To test the setup, I made a few changes to the blog content and pushed them to GitHub. Each update triggered a build and redeployment on Netlify, confirming the CI/CD pipeline worked seamlessly.

## Conclusion

This project highlights the power of automation in blogging. Using Hugo, GitHub, Netlify, and GoDaddy together created an efficient workflow for deploying and maintaining a professional blog. By automating repetitive tasks, I can now focus on creating quality content while the pipeline handles the rest.

---

If you’re interested in setting up a similar workflow or have questions, feel free to reach out!
