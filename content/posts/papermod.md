---
title: 'Step by step to build a blog with PaperMod like mine'
tags:
- hugo
- papermod
cover:
    image: /images/papermod.png
---

# Installation
Follow [Installation doc](https://github.com/adityatelange/hugo-PaperMod/wiki/Installation) to install Hugo and PaperMod.

**Git Submodule Method is recommended**, which keeps your repo *independent and tidy* by seperating your configs and posts from the theme's source code.

**Notice:** Don't forget to add a `.gitignore` file in your repo, like this:
```
/public
.DS_Store
.hugo_build.lock
resources/_gen/
```

Following are the tutorials of configuring PaperMod. You can reproduce the features by **simply copy the corresponding code snippets** or just **clone [my github page repo](https://github.com/figuremout/figuremout.github.io)** and replace my posts with yours.

Check [PaperMod doc](https://adityatelange.github.io/hugo-PaperMod/archives/) and [Hugo quick start](https://gohugo.io/getting-started/quick-start/) for more details. [PaperMod exampleSite source](https://github.com/adityatelange/hugo-PaperMod/tree/exampleSite) is also a good start.

# Configuration
## Home Info
Show welcome message and social links in home page like the cover image of this blog, follow the [regular mode](https://adityatelange.github.io/hugo-PaperMod/posts/papermod/papermod-features/#regular-mode-default-mode) doc.

Add following to your `hugo.yaml`, you can modify as you like:
```yaml
params:
  homeInfoParams:
    Title: "👋 Welcome to Z'Blog"
    Content: >
      - Hi, A **Software Engineer** here! 🤗

      - Passionate about exploring Linux 🐧, AI 🤖 and software development 💻.

  socialIcons:
    - name: github
      title: Visit me on GitHub
      url: "https://github.com/figuremout"
```

More social icons are available in [papermod icons](https://adityatelange.github.io/hugo-PaperMod/posts/papermod/papermod-icons/).

**Code Snippets**:
- [hugo.yaml](https://github.com/figuremout/figuremout.github.io/blob/604ce26d8fb019780aaa8682a024a6cef3467ee2/hugo.yaml#L42-L64)

## Menus
[Construct main menu](https://adityatelange.github.io/hugo-PaperMod/posts/papermod/papermod-faq/#add-menu-to-site) in the top-right corner.

First, you need to specify menu entries and their URL routes in `hugo.yaml`:
```yaml
menu:
  main:
    - name: Archive
      url: archives/
      weight: 10
    - name: Search
      url: search/
      weight: 20
    - name: Tags
      url: tags/
      weight: 30
    - name: Github
      url: https://github.com/figuremout
      weight: 40
```

`weight` is used to control the order of entries.

**Code Snippets**:
- [hugo.yaml](https://github.com/figuremout/figuremout.github.io/blob/604ce26d8fb019780aaa8682a024a6cef3467ee2/hugo.yaml#L9-L22)

*Tags* page will be automatically generated (see [hugo defualt taxonomies](https://gohugo.io/content-management/taxonomies/#default-taxonomies)), while additional works are needed to generate [*Archive*](https://adityatelange.github.io/hugo-PaperMod/posts/papermod/papermod-features/#archives-layout) and [*Search*](https://adityatelange.github.io/hugo-PaperMod/posts/papermod/papermod-features/#search-page) pages.

### *Archive* Page
Create *Archive* page `content/archives.md`:
```md
---
title: "Archive"
layout: "archives"
url: "/archives/"
summary: archives
---
```

**Code Snippets**:
- [content/archives.md](https://github.com/figuremout/figuremout.github.io/blob/604ce26d8fb019780aaa8682a024a6cef3467ee2/content/archives.md?plain=1#L1-L6)

### *Search* Page
Add following to your `hugo.yaml`:
```yaml
outputs:
  home:
    - HTML
    - RSS
    - JSON # necessary for search
```

Create *Search* page `content/search.md`:
```md
---
title: "Search" # in any language you want
layout: "search" # necessary for search
summary: "search"
placeholder: "placeholder text in search input box"
---
```

**Code Snippets**:
- [hugo.yaml](https://github.com/figuremout/figuremout.github.io/blob/604ce26d8fb019780aaa8682a024a6cef3467ee2/hugo.yaml#L70-L74)
- [content/search.md](https://github.com/figuremout/figuremout.github.io/blob/604ce26d8fb019780aaa8682a024a6cef3467ee2/content/search.md?plain=1#L1-L8)

## Zoom in Image on Click
The [issue comment](https://github.com/adityatelange/hugo-PaperMod/issues/384#issuecomment-899219940) provides a solution using medium-zoom, which works perfectly.

Add this to `layouts/partials/extend_footer.html` to zoom an image and dim the background on click:
```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/medium-zoom/1.0.6/medium-zoom.min.js" integrity="sha512-N9IJRoc3LaP3NDoiGkcPa4gG94kapGpaA5Zq9/Dr04uf5TbLFU5q0o8AbRhLKUUlp8QFS2u7S+Yti0U7QtuZvQ==" crossorigin="anonymous" referrerpolicy="no-referrer"></script>

<script>
const images = Array.from(document.querySelectorAll(".post-content img"));
images.forEach(img => {
  mediumZoom(img, {
    margin: 0, /* The space outside the zoomed image */
    scrollOffset: 40, /* The number of pixels to scroll to close the zoom */
    container: null, /* The viewport to render the zoom in */
    template: null, /* The template element to display on zoom */
    background: 'rgba(0, 0, 0, 0.8)'
  });
});
</script>
```

**Code Snippets**:
- [layouts/partials/extend_footer.html](https://github.com/figuremout/figuremout.github.io/blob/604ce26d8fb019780aaa8682a024a6cef3467ee2/layouts/partials/extend_footer.html#L1-L16)

**Demostration**:
![test](/images/papermod.png#center)

## Latex Support
Papermod has a [guide](https://adityatelange.github.io/hugo-PaperMod/posts/math-typesetting/) to setup KaTeX.

Create a partial `layouts/partials/math.html`.
**Notice**: In order to support `$` as delimiter, you need to call the `renderMathInElement` to [config Auto-render Extention](https://katex.org/docs/autorender.html) as following:
```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/katex@0.16.10/dist/katex.min.css" integrity="sha384-wcIxkf4k558AjM3Yz3BBFQUbk/zgIYC2R0QpeeYb+TwlBVMrlgLqwRjRtGZiK7ww" crossorigin="anonymous">
<script defer src="https://cdn.jsdelivr.net/npm/katex@0.16.10/dist/katex.min.js" integrity="sha384-hIoBPJpTUs74ddyc4bFZSM1TVlQDA60VBbJS0oA934VSz82sBx1X7kSx2ATBDIyd" crossorigin="anonymous"></script>
<script defer src="https://cdn.jsdelivr.net/npm/katex@0.16.10/dist/contrib/auto-render.min.js" integrity="sha384-43gviWU0YVjaDtb/GhzOouOXtZMP/7XUzwPTstBeZFe/+rCMvRwr4yROQP43s0Xk" crossorigin="anonymous"></script>
<script>
    document.addEventListener("DOMContentLoaded", function() {
        renderMathInElement(document.body, {

          // customised options
          // • auto-render specific keys, e.g.:
          delimiters: [

              {left: '$$', right: '$$', display: true},
              {left: '$', right: '$', display: false},
              {left: '\\(', right: '\\)', display: false},
              {left: '\\[', right: '\\]', display: true}
          ],
          // • rendering keys, e.g.:
          throwOnError : false
        });
    });
</script>
```

Include the partial in `extend_head.html`:
```html
{{ if or .Params.math .Site.Params.math }}
{{ partial "math.html" . }}
{{ end }}
```

Then you can globally enable KaTex in `hugo.yaml` as below or manually add `math: true` in posts' front matter:
```yaml
params:
  math: true
```

**Code Snippets**:
- [layouts/partials/math.html](https://github.com/figuremout/figuremout.github.io/blob/604ce26d8fb019780aaa8682a024a6cef3467ee2/layouts/partials/math.html#L1-L21)
- [layouts/partials/extend_head.html](https://github.com/figuremout/figuremout.github.io/blob/604ce26d8fb019780aaa8682a024a6cef3467ee2/layouts/partials/extend_head.html#L1-L3)
- [hugo.yaml](https://github.com/figuremout/figuremout.github.io/blob/604ce26d8fb019780aaa8682a024a6cef3467ee2/hugo.yaml#L38)

**Demostration**:
```markdown
Inline math: $E=mc^2$

Block math:
$$
E=mc^2
$$
```
Inline math: $E=mc^2$

Block math:
$$
E=mc^2
$$

## Show Lastmod Time as Date
Set date of every post in frontmatter is tooooo troublesome and hard to maintain. Luckily, we can [configure dates](https://gohugo.io/getting-started/configuration/#configure-dates) to show file’s last modification timestamp.

Add this to `hugo.yaml`:
```yaml
frontmatter:
  date:
  - :fileModTime # Fetches the date from the content file’s last modification timestamp.
```

**Code Snippets**:
- [hugo.yaml](https://github.com/figuremout/figuremout.github.io/blob/604ce26d8fb019780aaa8682a024a6cef3467ee2/hugo.yaml#L66-L68)


## Edit Link
[Edit link for posts](https://adityatelange.github.io/hugo-PaperMod/posts/papermod/papermod-features/#edit-link-for-posts) is the *"Suggest Changes"* button on this page, through which you can access the source file easily.

Add following to `hugo.yaml`, you should change the `URL` to your repo that contains the post’s source file:
```yaml
params:
  editPost:
    URL: "https://github.com/figuremout/figuremout.github.io/tree/main/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
```

It will link to `$URL/posts/post-name.md`.

**Code Snippets**:
- [hugo.yaml](https://github.com/figuremout/figuremout.github.io/blob/604ce26d8fb019780aaa8682a024a6cef3467ee2/hugo.yaml#L34-L37)

## Syntax Highlighter
[Using Hugo's syntax highlighter "chroma"](https://adityatelange.github.io/hugo-PaperMod/posts/papermod/papermod-faq/#using-hugos-syntax-highlighter-chroma) to config the code block's line number, style and so on.

**Code Snippets**:
- [hugo.yaml](https://github.com/figuremout/figuremout.github.io/blob/e9c1f32d82b34dc7efa90afe47214911da13afb7/hugo.yaml#L77-L84)
- [hugo.yaml](https://github.com/figuremout/figuremout.github.io/blob/e9c1f32d82b34dc7efa90afe47214911da13afb7/hugo.yaml#L41)

**Demostration**:
```python
print("hello")
print("world")
```

## Host on Github Pages
Host your site on Github Pages and setup **Github Actions**, so that Github will rebuild and deploy your site automatically whenever you push a change. Hugo provides a [guide](https://gohugo.io/hosting-and-deployment/hosting-on-github/) to build the workflow.

**Code Snippets**:
- [.github/workflows/hugo.yaml](https://github.com/figuremout/figuremout.github.io/blob/e9c1f32d82b34dc7efa90afe47214911da13afb7/.github/workflows/hugo.yaml#L1-L89)

## Render Raw HTML in Markdown
[Hugo does not render raw HTML by default](https://discourse.gohugo.io/t/do-you-set-unsafe-true-in-markup-goldmark-renderer/37555). As a result, you cannot embed vedios using `<iframe>`, superscript using `<sup>` or insert collapsible sections using `<details>`, etc. Though you can achieve the same by [*Hugo Shortcodes*](https://gohugo.io/content-management/shortcodes/), you may prefer the **more direct way: Insert pure HTML in Markdown**.

Just enable the `unsafe` option of Goldmark Markdown renderer in `hugo.yaml` to make it work.
```yaml
markup:
  goldmark:
    renderer:
      unsafe: true
```

**Code Snippets**:
- [hugo.yaml](https://github.com/figuremout/figuremout.github.io/blob/6e1f81d2542c39e54cf36c0771532283c8030835/hugo.yaml#L85-L87)

<details open>
<summary><b>Demostration</b>:</summary>

<iframe src="//player.bilibili.com/player.html?isOutside=true&aid=841786906&bvid=BV1x54y1e7zf&cid=226204073&p=1&autoplay=0&danmaku=0" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true" width="560" height="315" style="border:none;overflow:hidden;display:block;margin:0 auto;"></iframe>
</details>

## Change Content Width
This [issue comment](https://github.com/adityatelange/hugo-PaperMod/discussions/442#discussioncomment-5662173) provides a nice solution.

Add following to `assets/css/extended/theme-vars.css`:
```css
:root {
    --main-width: 750px;
}
```

**Code Snippets**:
- [assets/css/extended/theme-vars.css](https://github.com/figuremout/figuremout.github.io/blob/c0eb8b8157a19994ae231c96b8886374610c96a2/assets/css/extended/theme-vars.css#L1-L3)

## Change Fonts
You can use **fonts from [Google Fonts](https://fonts.google.com)**.
1. Find a font on Google Fonts.
2. Get embed code for Web (several lines of html and css).
3. Copy the html parts to `layouts/partials/extend_head.html`.
4. Copy the css parts to `assets/css/extended/blank.css` (adjust css selector according to your needs, e.g. `body`, `code`, `.post-content`...).

**Code Snippets**:
- [layouts/partials/extend_head.html](https://github.com/figuremout/figuremout.github.io/blob/c0eb8b8157a19994ae231c96b8886374610c96a2/layouts/partials/extend_head.html#L5-L7)
- [assets/css/extended/blank.css](https://github.com/figuremout/figuremout.github.io/blob/c0eb8b8157a19994ae231c96b8886374610c96a2/assets/css/extended/blank.css#L1-L9)

**Or download fonts** like *[DroidSansM Nerd Font](https://www.nerdfonts.com/font-downloads)* into `static/fonts/`, then create `assets/css/extended/fonts.css` with the following code:
```css
@font-face {
    font-family: "DroidSansMNerdFontMono-Regular";
    src: url("/fonts/DroidSansMono/DroidSansMNerdFontMono-Regular.otf");
}

@font-face {
    font-family: "DroidSansMNerdFontPropo-Regular";
    src: url("/fonts/DroidSansMono/DroidSansMNerdFontPropo-Regular.otf");
}

@font-face {
    font-family: "DroidSansMNerdFont-Regular";
    src: url("/fonts/DroidSansMono/DroidSansMNerdFont-Regular.otf");
}
```

And specify `font-family` in `assets/css/extended/blank.css` with the following code:
```css
.post-content {
    font-family: "DroidSansMNerdFontPropo-Regular";
}
```

## View Count
This [blog](https://www.elegantcrazy.com/posts/blog/build-blog-with-github-pages-hugo-and-papermod/#%E6%B7%BB%E5%8A%A0%E8%AE%BF%E9%97%AE%E9%87%8F%E7%BB%9F%E8%AE%A1%E5%8A%9F%E8%83%BD) is a great tutorial to count page views and site visits using [busuanzi API](https://busuanzi.ibruce.info/).

**Code Snippets**:
- [hugo.yaml](https://github.com/figuremout/figuremout.github.io/blob/8b47dfa14ecd99fc9f6a06d5bb51b5cf44acdda7/hugo.yaml#L72-L73)
- [layouts/partials/extend_head.html](https://github.com/figuremout/figuremout.github.io/blob/8b47dfa14ecd99fc9f6a06d5bb51b5cf44acdda7/layouts/partials/extend_head.html#L10-L13)
- [layouts/partials/footer.html](https://github.com/figuremout/figuremout.github.io/blob/8b47dfa14ecd99fc9f6a06d5bb51b5cf44acdda7/layouts/partials/footer.html#L1-L157)
- [layouts/_default/single.html](https://github.com/figuremout/figuremout.github.io/blob/8b47dfa14ecd99fc9f6a06d5bb51b5cf44acdda7/layouts/_default/single.html#L1-L72)

Copying and editing the theme code could make things less elegant than before, but it's a necessary compromise.

# QA
## How to specify image path?
Check [this](https://stackoverflow.com/questions/71501256/how-to-insert-an-image-in-my-post-on-hugo), there are two ways to reference a image in markdown.

My preferred way is:
- Put all images in `static/images`.
- And reference an image by `![img](/images/test.png)`.

BTW, add `#center` after image link to center it, see [doc](https://adityatelange.github.io/hugo-PaperMod/posts/papermod/papermod-faq/#centering-image-in-markdown).
