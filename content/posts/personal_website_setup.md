---
title: Creating a personal website with Hugo, S3, CloudFront, Forestry with a fully
  functioning CI/CD pipeline
date: 2021-12-01T23:21:00.000+11:00
description: Inspired by the JAMstack, I created a fully functioning blog and personal
  website that's static, builds on the main branch and  gets deployed worldwide through
  a CDN - all without needing to host a webserver.
tldr: Hugo + Forestry + CircleCI + Terraform + AWS = Awesome

---
Creating a personal website for developers is nowadays often mandatory - showing your skill and expertise to fellow developers and prospective employers. But oftentimes creating a dynamic server side rendered website is overkill and requires maintenance and upkeep.

Inspired by the JAMstack, I created a fully functioning blog and personal website that's static, builds on the main branch and  gets deployed worldwide through a CDN - all without needing to host a webserver.

This website was created with the following technologies:

* Hugo
* Forestry
* AWS
  * S3
  * CloudFront
* Terraform
* CircleCI

## Front End Setup

I used [Hugo](https://gohugo.io/ "Hugo") as my static site generator because it was fast, easy and the way it structured markdown seemed intuitive to me. Additionally I used the [Archie](https://themes.gohugo.io/themes/archie/ "Archie Hugo Theme") theme as  a simple markdown-esque inspired theme which had no frills.

I then used [Forestry](https://forestry.io/ "Forestry.io") as a git based headless CMS so when I create pages and posts I can focus on the content and not the technology. Any changes made are pushed to the git repo which auto builds and deploys a new version of the website.

## CI/CD Pipeline Setup

When a commit is placed on the main branch of my git repo a job is run by the CI tool [CircleCI](https://circleci.com/ "CircleCI"). There are currently two stages within my pipeline: build and deploy.

The build stage checks out the git repo, builds the static site with Hugo, and checks if there are any HTML errors with HTML proofer before passing on to the deploy stage. This is done using the Hugo orb using the `hugo/build` job

![Hugo Build  Stage](/uploads/circleci_build.png "Hugo Build Stage")

The deploy stage simply uploads the generated static files to an S3 bucket using the `aws-s3` orb. Once uploaded I invalidate the CloudFront distribution cache so my changes are retrieved directly from the S3 bucket before being cached.

    aws cloudfront create-invalidation --distribution-id $CLOUDFRONT_DISTRIBUTION_ID --paths "/*"

For more information you can check out my CircleCI config [here](https://github.com/sammaritan12/personal-website/blob/main/.circleci/config.yml "My CircleCI config").

## Infrastructure Setup

Essentially any objects that are uploaded to the S3 bucket will be retrieved by a CloudFront distribution and placed around the world via their CDNs. An alias record is then created on the apex domain within a public Route53 zone. This is how you can access this website from `markpatricio.com`

One caveat I found was that Hugo does not play well with REST API endpoints on CloudFront origins. Pages must be linked to directly and it does not automatically assume `index.html`, e.g. we must use `/about/index.html` instead of `/about`. Therefore we cannot restrict access to the bucket via an OAI (Origin Access Identity).

To fix this we must use a website endpoint on the CloudFront origin. This involves allowing public access to the bucket policy, using a custom origin in the CloudFront distribution origin and allowing the bucket to host websites.

This means that the bucket is publicly accessible and any requests from CloudFront to the S3 bucket is done via HTTP connection. This is obviously not secure. Two alternatives exist:

1. Update the bucket policy to only allow public read access if there exists a header within the request that CloudFront adds in
2. Use Lambda@Edge to change requests to automatically add the `index.html`

At the time of writing this has not been implemented but if you wish to learn more about it you can check out [AWS' support documentation](https://aws.amazon.com/premiumsupport/knowledge-center/cloudfront-serve-static-website/ "Hosting static website with S3 and CloudFront") regarding this.

![Diagram of infrastucture used to create website](/uploads/untitled-diagram-page-2-drawio.png "Infrastructure Diagram")

Additionally we add ACM certificates so we can use HTTPS on a custom alias. Once we've connected the alias record to the CloudFront distribution we're almost done.

We must then connect our domain registrar with the public Route53 zone we've created. This is done by changing the DNS of our domain to use AWS provided nameservers. Once propagated our website should be available for the world to see.

The entire infrastructure was created using [Terraform](https://www.terraform.io/ "Terraform"), my preferred IaC (Infrastructure as Code) tool. You can check out how I created my modules, organised my workspace and how the infrastructure is set up [here.](https://github.com/sammaritan12/terraform-personal-website "Terraform Personal Website")