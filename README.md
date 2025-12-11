# Tech Blog

A Hugo-powered technical blog covering AI, cloud architecture, and modern software development.

## Structure

```
tech-blog/
├── .github/
│   └── workflows/
│       └── deploy.yml      # GitHub Actions for S3 deployment
├── site/
│   ├── hugo.toml           # Hugo configuration
│   ├── content/
│   │   ├── posts/          # Blog posts
│   │   ├── about.md        # About page
│   │   └── search.md       # Search page
│   ├── static/             # Static assets
│   └── themes/
│       └── PaperMod/       # Theme (git submodule)
├── AWS_SETUP.md            # AWS infrastructure guide
└── README.md
```

## Local Development

### Prerequisites

- [Hugo Extended](https://gohugo.io/installation/) (v0.139.0+)

### Running Locally

```bash
cd site
hugo server -D
```

Visit `http://localhost:1313` to preview the site.

### Creating New Posts

```bash
cd site
hugo new posts/my-new-post.md
```

## Deployment

The blog automatically deploys to AWS S3 + CloudFront when changes are pushed to the `main` branch.

### Setup Required

1. Follow the instructions in [AWS_SETUP.md](AWS_SETUP.md)
2. Configure GitHub secrets:
   - `AWS_ROLE_ARN` - IAM role for OIDC authentication
   - `AWS_S3_BUCKET` - S3 bucket name
   - `AWS_CLOUDFRONT_DISTRIBUTION_ID` - CloudFront distribution ID

## Current Content

### HR Helper Series

A 5-part series on building AI-powered recruitment technology:

1. Why HR Departments Are Drowning in CVs
2. From Azure to AWS: A Practical Cloud Migration Story
3. Beyond Keywords: Using LLMs to Actually Understand Resumes
4. The ROI of AI-Powered Recruitment
5. What We Learned Migrating to AI-First HR Tech

## Tech Stack

- **Static Site Generator**: Hugo
- **Theme**: PaperMod
- **Hosting**: AWS S3 + CloudFront
- **CI/CD**: GitHub Actions

