---
layout: post
title:  "vanilla.js document ready event"
date:   2021-09-29 17:47:00 +0100
categories: javascript, js
description: "I often find myself creating simple html webpages for little things. I dont want to use massive javascript libraries, but I sometimes need some functionality from such a library."
author: "@gerritmenzel"
---

An event listener when the webpage is fully loaded could be achieved by simply creating a new file like `util.js` and pasting this simple code inside.

```
document.onreadystatechange = () => {
    if (document.readyState === 'complete') {
        document.dispatchEvent(new Event("ready"));
    }
};
```

including my `util.js` in the html file then in my javascript:

```
document.addEventListener("ready", () => {

    // javascript launched when the website is fully loaded and the DOM is build.
    
}, { once: true });
```

I understand that this is not save in all conditions and not suitable for bigger projects. But for my use cases it works just fine.
