+++
title = "This Blog"
date = 2023-12-18T06:34:30-05:00
draft = false
+++

# Back to Blogging

After a hiatus, I've decided to give blogging another go. My previous blog was set up using Jekyll and hosted on GitHub. Since I blogged infrequently, I didn't feel the need to explore other platforms. However, a recent shift in my attitude towards social media, specifically Twitter, and the ease of writing with GenAI, rekindled my interest in blogging. I yearned for a more aesthetic and user-friendly experience, leading me to transition my blog to Hugo, using the Papermod theme.

## Steps for Setting Up Everything

There are many platofrms that allow you to setup a blog/newsletter. Substack and Wordpress for example. I chose to setup everything myself since I wanted to use Hugo and figured it would be simple enough. Here's a high-level step-by-step breakdown of what I did:

1. **Forking the Papermod Theme:** You don't have to fork it but it makes it easy to modify. I use the fork as a submodule within my Hugo blog repo. 
2. **Domain Registration with CloudFlare:** (Optional) Chose a new domain and registered it through CloudFlare. ($10/year)
3. **Creating a New Blog with Hugo:** Followed their guide, it's very simple. The only difference is that I used my fork of the Papermod repo. 
4. **Deploying with Render:** Followed [Render's guide](https://docs.render.com/docs/configure-cloudflare-dns) to set up deployment. While there are many options, including GitHub Pages, I've heard good things about Render and wanted to try it out.
5.  **Email subscription form:** Many options available. I went with ConvertKit that has a free version for up to 1000 subscribers. I created a form (Grow -> Landing pages & forms) and then added the js code snippet it created to my forked repo of the theme `layouts/_default/baseof.html`. You can check my Github as both the website code and the fork are public. 
6. **Email Redirection and Update on ConvertKit:** Set up a custom email redirect (from contact@<my-domain> to my personal Gmail address) on CloudFlare and updated it on ConvertKit. This is not required but they recommend not using a free email like Gmail to send the subscription confirmation. 

So far I didn't need to pay anything besides acquiring the domain name. I have a new blog. Now I just need to create some interesting content...




