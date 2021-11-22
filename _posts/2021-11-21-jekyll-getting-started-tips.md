---
title                    : "Creating blog site using GitHub Pages and Jekyll."
date                     : 2021-11-22 11:20:00 -0800
last_modified_at         : 2021-11-22 11:20:00 -0800
categories               : Coding Jekyll
permalinks               : /:categories/:year/:month/:day/:title.html
header:
  teaser                 : /assets/images/teaser-default.jpg
---

This post talks about some places I stumbled upon when creating this blog site using GitHub Pages and Jekyll.

This is not a complete guide to set up Jekyll. References about that are the following:
- Jekyll's [Quickstart](https://jekyllrb.com/docs/)
- Jekyll's [Community Tutorials](https://jekyllrb.com/tutorials/home/)
- Minimal Mistakes theme's [Quick-Start Guide](https://mmistakes.github.io/minimal-mistakes/docs/quick-start-guide/)
- GitHub's [Guide](https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll)

## Ruby installation

I faced an issue when I was installing Ruby on Windows. I downloaded the latest Ruby installer without DevKit, and started the installation. When I was at "ridk install" step to set up MSYS2 and development toolchain, as shown in the figure below, I pressed Enter to do action 1 and action 3.

{% include figure image_path="/assets/images/coding-ruby-msys2.jpg" alt="MSYS2 installation and the development toolchain setup." caption="MSYS2 installation and the development toolchain setup." %}

Then I encountered the errors like: "key XXXX is unknown" and "invalid or corrupted database", as shown in the figure below.

{% include figure image_path="/assets/images/coding-ruby-msys2-error.jpg" alt="MSYS2 development toolchain setup error." caption="MSYS2 development toolchain setup error." %}

I found that I need to download and use the Ruby installer "with" DevKit, and check the "MSYS2 and development toolchain" when I select the components to install. With that, the installation and the toolchain setup were successful.

**Watch out!** To uninstall Ruby and MSYS2 in case you encounter issues, use Windows' "Add or remove programs" and make sure that the Ruby and MSYS2 folders are removed.
{: .notice--info}

## Using Minimal Mistakes theme on GitHub Pages

I did not pay attention to the "Remote theme method" section at Minimal Mistakes theme's [Quick-Start Guide](https://mmistakes.github.io/minimal-mistakes/docs/quick-start-guide/). As it turned out, I cannot use the default "theme" method since Minimal Mistakes theme is not inbox-supported by GitHub Pages. Make sure that you use the "remote theme" if you use a Jekyll theme that is not inbox-supported by GitHub Pages.

## Setting up Google Search Console and Bing Webmaster Tools

Once the blog is created and is up on GitHub Pages, I want it to be searchable at Google and Bing. Minimal Mistakes theme's [Configuration](https://mmistakes.github.io/minimal-mistakes/docs/configuration/#google-search-console) and this [post](https://victor2code.github.io/blog/2019/07/04/jekyll-github-pages-appear-on-Google.html) describe the steps. The [jekyll-sitemap](https://github.com/jekyll/jekyll-sitemap) describes how to generate the sitemap file locally.

For my case, downloading the verification HTML file and hosting it on my site's root worked well for me. I downloaded Google's and Bing's files, put them at my site's root folder, and then pushed it to my GitHub Pages' repository. After I successfully viewed those files at my site URL on a web browser, I went back to click "Verify" button at Google Search Console and at Bing Webmaster Tools.

{% include figure image_path="/assets/images/coding-jekyll-googleverification.jpg" alt="Google: Download HTML file and host it on your site's root to verify your site." caption="Google: Download HTML file and host it on your site's root to verify your site." %}

{% include figure image_path="/assets/images/coding-jekyll-bingverification.jpg" alt="Bing: Download HTML file and host it on your site's root to verify your site." caption="Bing: Download HTML file and host it on your site's root to verify your site." %}

Then I followed the posts above to get my sitemap file created, and added my sitemap file to Google Search Console and Bing Webmaster Tools. You can see the figures below that the sitemap checking at Google is pretty slow.

**Watch out!** At this time (after 3 days), the sitemap checking is still not done at Google. I will update later when it is done.
{: .notice--info}

{% include figure image_path="/assets/images/coding-jekyll-googlesitemap.jpg" alt="Google: Sitemap (submitted on Nov 19) checking is still not ready on Nov 22." caption="Google: Sitemap (submitted on Nov 19) checking is still not ready on Nov 22." %}

{% include figure image_path="/assets/images/coding-jekyll-bingsitemap.jpg" alt="Bing: Sitemap (submitted on Nov 21) checking is ready on Nov 21." caption="Bing: Sitemap (submitted on Nov 21) checking is ready on Nov 21." %}

Even if the sitemap checking is done, it does not mean that the site is indexed yet at Google or Bing.
- For Google, I used "URL Inspection" at Google Search Console and found that my site were not indexed yet on Nov 21.
- For Bing, I used "URL Inspection" at Bing Webmaster Tools and found that my site were not indexed yet on Nov 21. The indexing was done on Nov 22. So the total time from adding the site to being searchable is 1 to 2 days.

**Watch out!** I will update later when my site is indexed at Google and is searchable at Google.
{: .notice--info}




