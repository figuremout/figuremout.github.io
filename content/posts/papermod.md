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

**Git Submodule Method is recommended**, which keeps your repo independent and tidy by seperating your configs and posts from the theme's source code.

**Notice:** Don't forget to add a `.gitignore` file in your repo, like this:
```
/public
.DS_Store
.hugo_build.lock
resources/_gen/
```

Following are the tutorials of configuring PaperMod.

You can reproduce the features by **simply copy the corresponding code snippets**.

Check [PaperMod doc](https://adityatelange.github.io/hugo-PaperMod/archives/) and [Hugo doc](https://gohugo.io/getting-started/quick-start/) for more configuration. [PaperMod exampleSite source](https://github.com/adityatelange/hugo-PaperMod/tree/exampleSite) is also a good start.

# Configuration
## Home Info
Show welcome message and social links in home page, follow the doc [here](https://adityatelange.github.io/hugo-PaperMod/posts/papermod/papermod-features/#regular-mode-default-mode).

**Code Snippets**:
- [hugo.yaml](https://github.com/figuremout/figuremout.github.io/blob/604ce26d8fb019780aaa8682a024a6cef3467ee2/hugo.yaml#L42-L64)

## Menus
Construct main menu in the top-right corner, follow the doc [here](https://adityatelange.github.io/hugo-PaperMod/posts/papermod/papermod-faq/#add-menu-to-site).

**Code Snippets**:
- [hugo.yaml](https://github.com/figuremout/figuremout.github.io/blob/604ce26d8fb019780aaa8682a024a6cef3467ee2/hugo.yaml#L9-L22)

**[Archive menu](https://adityatelange.github.io/hugo-PaperMod/posts/papermod/papermod-features/#archives-layout) Code Snippets**:
- [content/archives.md](https://github.com/figuremout/figuremout.github.io/blob/604ce26d8fb019780aaa8682a024a6cef3467ee2/content/archives.md?plain=1#L1-L6)

**[Search menu](https://adityatelange.github.io/hugo-PaperMod/posts/papermod/papermod-features/#search-page) Code Snippets**:
- [content/search.md](https://github.com/figuremout/figuremout.github.io/blob/604ce26d8fb019780aaa8682a024a6cef3467ee2/content/search.md?plain=1#L1-L8)
- [hugo.yaml](https://github.com/figuremout/figuremout.github.io/blob/604ce26d8fb019780aaa8682a024a6cef3467ee2/hugo.yaml#L70-L74)

## Zoom in Image on Click
Works perfectly with medium-zoom, follow the issue [here](https://github.com/adityatelange/hugo-PaperMod/issues/384#issuecomment-899871056).

**Code Snippets**:
- [layouts/partials/extend_footer.html](https://github.com/figuremout/figuremout.github.io/blob/604ce26d8fb019780aaa8682a024a6cef3467ee2/layouts/partials/extend_footer.html#L1-L16)

**Demostration**:
![test](/images/papermod.png#center)

## Latex Support
Follow the [doc](https://adityatelange.github.io/hugo-PaperMod/posts/math-typesetting/) to setup KaTeX.

**Notice**: In order to support `$` as delimiter, you need to call the `renderMathInElement` to [config Auto-render Extention](https://katex.org/docs/autorender.html).

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
Set date of every post in frontmatter is tooooo troublesome and hard to maintain.

Luckily, we can configure it to show file’s last modification timestamp as date, see [doc](https://gohugo.io/getting-started/configuration/#configure-dates).

**Code Snippets**:
- [hugo.yaml](https://github.com/figuremout/figuremout.github.io/blob/604ce26d8fb019780aaa8682a024a6cef3467ee2/hugo.yaml#L66-L68)


## Edit Link
Add a button (like the "Suggest Changes" button on this page) to link the post to a edit destination (a github repo typically). See [doc](https://adityatelange.github.io/hugo-PaperMod/posts/papermod/papermod-features/#edit-link-for-posts).

**Code Snippets**:
- [hugo.yaml](https://github.com/figuremout/figuremout.github.io/blob/604ce26d8fb019780aaa8682a024a6cef3467ee2/hugo.yaml#L34-L37)

## Syntax Highlighter
Config the code block's line number, style and so on, check the doc [here](https://adityatelange.github.io/hugo-PaperMod/posts/papermod/papermod-faq/#using-hugos-syntax-highlighter-chroma).

**Code Snippets**:
- [hugo.yaml](https://github.com/figuremout/figuremout.github.io/blob/e9c1f32d82b34dc7efa90afe47214911da13afb7/hugo.yaml#L77-L84)
- [hugo.yaml](https://github.com/figuremout/figuremout.github.io/blob/e9c1f32d82b34dc7efa90afe47214911da13afb7/hugo.yaml#L41)

**Demostration**:
```python
print("hello")
print("world")
```

## Host on Github Pages
Host your site on Github Pages and setup **Github Actions**, so that Github will rebuild and deploy your site automatically whenever you push a change, see [doc](https://gohugo.io/hosting-and-deployment/hosting-on-github/).

**Code Snippets**:
- [.github/workflows/hugo.yaml](https://github.com/figuremout/figuremout.github.io/blob/e9c1f32d82b34dc7efa90afe47214911da13afb7/.github/workflows/hugo.yaml#L1-L89)

# QA
## How to specify the image path?
Check [this](https://stackoverflow.com/questions/71501256/how-to-insert-an-image-in-my-post-on-hugo), there are two ways to reference a image in markdown.

My preferred way is:
- Put all images in `static/images`.
- And reference an image by `![img](/images/test.png)`.

BTW, add `#center` after image link to center it, see [doc](https://adityatelange.github.io/hugo-PaperMod/posts/papermod/papermod-faq/#centering-image-in-markdown).
