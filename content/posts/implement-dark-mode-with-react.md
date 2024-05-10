---
title: "Implement Dark Mode With React"
date: 2024-05-03T11:10:08-07:00
draft: false
categories:
  - Development
tags:
  - frontend
---

Being able to switch between light and dark mode has always been a fancy feature in my mind when it comes to the UI part of an app. Recently I've done it myself and found it actually quite straightforward to implement.

# Rationale

Aside from any frameworks, let's think about how color mode switch should work. Apparently we need 2 sets of colors defined with CSS, as well as an easy way to change between them for our HTML elements.

To conveniently reuse pre-defined colors, we can certainly leverage on [CSS variables](https://www.w3schools.com/css/css3_variables.asp):

```CSS
:root {
  --background-color: white;
  --text-color: black;
}

[data-theme="dark"] {
  --background-color: black;
  --text-color: white;
}

div {
  background-color: var(--background-color);
}

.title {
  color: var(--text-color);
}
```

With the above setup, it becomes effortless to select either of the 2 color sets. Remeber the variable names must begin with two dashes to be used by the `var()` function.

Noticeably I also use the attribute selector in CSS to define the colors for dark mode, which is for HTML elements to pick up with a custom data attribute. The `data-*` format is according to naming recommendations, but not mandatory.

The last piece would be a toggler. With vanilla JS we only have to manipulate the `data-theme` attribute on the target HTML elements. Of course in React it could take a state variable instead.

Now we have all the foundamental knowledge to implement dark mode.

# Remeber the choice

Sometimes, users want to keep their choice of the theme mode, even after closing the window or browser. In order to achieve this, we can store the decision info in `localStorage`.

Simply put, the [Web Storage API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Storage_API) provides us with 2 ways of saving key/value pairs. Compared with `sessionStorage`, `localStorage` persists data even when the browser is closed and reopened.

There's a NPM package called [use-local-storage](https://www.npmjs.com/package/use-local-storage) if you want to save the hassle. Essentially it offers a thin layer of abstraction of the original API, which also makes serialization a bit easier.

# Follow system preference

Some users would like the app to follow their theme settings of the operating system, which might be light, dark or automatic following time of the day.

For this, we're going to [use CSS media queries](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_media_queries/Using_media_queries), which allow us to get the device's media type or other configs. In JS there's an API to get such media info:

```javascript
// Here isDarkMode would be either true or false
const isDarkMode = window.matchMedia("(perfers-color-scheme: dark)").matches;
```

Furthermore, if you need to test it out, Chrome offers a simulating setting in its [Rendering tab](https://developer.chrome.com/docs/devtools/rendering).

# With frameworks

TailwindCSS is undoubtedly a powerful CSS framework and one of my favorite. It provides a very simple way to customize colors for dark mode:

```HTML
<div class="bg-white dark:bg-slate-800">
  <p class="text-slate-900 dark:text-white">Hello world!</p>
</div>
```

Besides, I also love using Shadcn/ui for delicate UI components. It offers copy-and-paste code for things to work out-of-the-box, such as [this tutorial](https://ui.shadcn.com/docs/dark-mode/vite) for adding dark mode to a vite app.

By default, Shadcn/ui uses a basic white/black color scheme, which is very similar to how Vercel looks like. However, it's fairly easy to customize as well, with either TailwindCSS or CSS variables which gives us the chance to decorate for both light and dark mode in a flexible way.

It has pre-defined CSS variables for the colors of its various components, such as `--background` or `--primary`. You can also add new colors as new variables. HSL colors are supported as well as hex and RGB, making it extremely handy to forge all the details to be exactly how you want them to be. Check out its [tutorial](https://ui.shadcn.com/docs/theming) if you would like to give a shot.

# A little demo

In case you want to take a look, here's a simple form with a mode toggle button that I built in CodeSandbox, you can switch to light or dark mode, or let it follow the system preference:

<iframe src="https://codesandbox.io/p/devbox/dark-mode-demo-tdkdkg?embed=1&file=%2Fsrc%2FApp.tsx"
     style="width:100%; height: 800px; border:0; border-radius: 4px; overflow:hidden;"
     title="dark-mode-demo"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>

---

### References

- [React Dark Mode Toggle/Theme - Complete Guide](https://www.youtube.com/watch?v=sy-rRtT84CQ&ab_channel=FullstackSimplified)
- [HTML custom data attributes](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/data-*)
- [Dark mode with TailwindCSS](https://tailwindcss.com/docs/dark-mode)
