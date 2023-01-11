---
theme: mokkapps
title: "Building a Polite Popup with Nuxt 3"
# lineNumbers: true
colorSchema: 'light'
---

# Building a Polite Popup with Nuxt 3

Vue.js Nation 2023 - Lightning Talk

<!--
The last comment block of each slide will be treated as slide notes. It will be visible and editable in Presenter Mode along with the slide. [Read more in the docs](https://sli.dev/guide/syntax.html#notes)
-->

---
layout: about-me
---

---
layout: section
---

## What is a "Impolite Popup"?

---
layout: iframe-right
url: "https://vuejsnation-2023-polite-popup-nuxt-3.netlify.app/?demoMode=impolite-popup"
---

# Demo Time

[Source Code](https://stackblitz.com/edit/vuejsnation-2023-lightning-talk-polite-popup-nuxt-3)

Let's take a look at an example of an "impolite popup"

<v-clicks>

- <twemoji-warning /> Problem 1: Another annoying popup on the landing page
- <twemoji-warning /> Problem 2: Visitor haven't engaged with the related content
- <twemoji-warning /> Problem 3: Visitor is asked every time he visits the page

</v-clicks>


---
layout: section
---

## What is a "Polite Popup"?

---
layout: image-right
image: https://sd.keepcalms.com/i/keep-calm-and-please-be-polite-thank-you.png
---

# Polite Popup

In the following slides, we'll build a popup that

<v-clicks>

- waits for a visitor to browse the website
- makes sure visitors are interested in the website
- appears off to the side in a non-intrusive way
- is easy to dismiss or ignore
- asks for permission first
- waits a bit before it appears again

</v-clicks>

<v-click>

This means they’ll be **more likely to sign up** by the time we ask them because it’ll be **after they’ve decided they liked our content**.

</v-click>

---
layout: image
image: https://media.giphy.com/media/13GIgrGdslD9oQ/giphy.gif
---

# Coding Time

<style>
h1 {
  color: white !important;
  @apply !text-shadow-lg;
  @apply !text-center;
  @apply !text-8xl
}
</style>

---

# Vue Composable

Let's start by writing a Vue composable for our polite popup. The goal is that the visitors have to:

<v-clicks>

- be **visiting a page with Vue-related content** as the newsletter targets Vue developers
- be **actively scrolling** the current page for **6 seconds or more**
- **scroll through at least 35%** of the current page during their visit

</v-clicks>

<v-click>

before they get asked to sign up.

</v-click>

<br/>

<v-click>

```ts {1,10|2|4|6-9} {maxHeight:'170px'}
export const usePolitePopup = () => {
  const visible = useState("visible", () => false);

  const trigger = () => {}

  return {
    visible,
    trigger,
  };
};
```

</v-click>

<!--
We defined two state variables:
* visible: a boolean indicating if the popup should be visible or not.
* readTimeElapsed: a boolean indicating if the user has spent the defined time on the page.

The trigger method is exposed and triggers the a timer which is used to check if the visitor has spent a predefined amount of time on the page. 

A Vue watcher is used to set visible to true if the timer has expired and the scroll threshold is exceeded. 

For the timer, we use VueUse's useTimeoutFn composable which runs a setTimeout function and sets the readTimeElapsed state variable to true after the timer has expired.
-->

---

# Track time spent on page

The visitor must be actively scrolling the current page for 6 seconds or more.

```ts {7|1,3,9-15|10-12|3,13|14|7,9,17-19} {maxHeight:'350px'}
import { useTimeoutFn } from '@vueuse/core'

const config = { timeoutInMs: 3000 }

export const usePolitePopup = () => {
  const visible = useState("visible", () => false);
  const readTimeElapsed = useState('read-time-elapsed', () => false)

  const { start } = useTimeoutFn(
    () => {
      readTimeElapsed.value = true
    },
    config.timeoutInMs,
    { immediate: false }
  )

  const trigger = () => {
    readTimeElapsed.value = false
    start()
  }

  return {
    visible,
    trigger,
  };
};
```

---

# Track scroll progress

The visitor must scroll through at least 35% of the current page during their visit.

```ts {1,7-16|3,18-20} {maxHeight:'350px'}
import { useWindowSize, useWindowScroll } from '@vueuse/core'

const config = { timeoutInMs: 3000, contentScrollThresholdInPercentage: 35, }

export const usePolitePopup = () => {
    //...
    const { height: windowHeight } = useWindowSize()
    const { y: scrollYInPx } = useWindowScroll()

    // Returns percentage scrolled (ie: 80 or NaN if trackLength == 0)
    const amountScrolledInPercentage = computed(() => {
      const documentScrollHeight = document.documentElement.scrollHeight
      const trackLength = documentScrollHeight - windowHeight.value
      const percentageScrolled = Math.floor((scrollYInPx.value / trackLength) * 100)
      return percentageScrolled
    })

    const scrolledContent = computed(() => {
      return amountScrolledInPercentage.value >= config.contentScrollThresholdInPercentage
    )}

    return {
        visible,
        trigger,
    }
}
```

<!--
To get the total scrollable area of a document, we need to retrieve the following two measurements of the page:
* The height of the browser window: We use VueUse's useWindowSize composable to get reactive variable of the browser window height.
* The height of the entire document: We use document.documentElement.scrollHeight to get the height of the document, including content not visible on the screen due to overflow.

By subtracting 2 from 1, we get the total scrollable area of the document. VueUse's useWindowScroll composable is used to access the number of pixels the document is currently scrolled along the vertical axis.

Move your eyes down to the trackLength variable, which gets the total available scroll length of the document. The variable will contain 0 if the page is not scrollable. The percentageScrolled variable then divides the scrollYInPx variable (amount the user has scrolled) with trackLength to derive how much the user has scrolled percentage wise.
-->

---

# Trigger visible

We have now all information available to update the `visible` reactive variable: 

```ts {2,3,6,8-12} {maxHeight:'350px'}
export const usePolitePopup = () => {
    const visible = useState('visible', () => false)
    const readTimeElapsed = useState('read-time-elapsed', () => false)
    //...

    const scrolledContent = computed(() => amountScrolledInPercentage.value >= config.contentScrollThresholdInPercentage)

    watch([readTimeElapsed, scrolledContent], ([newReadTimeElapsed, newScrolledContent]) => {
        if (newReadTimeElapsed && newScrolledContent) {
            visible.value = true
        }
    })

    return {
        visible,
        trigger,
    }
}
```

---

# Wait Before the Popup Appears Again

```ts {3-7|1,11-16|20-24} {maxHeight:'350px'}
import { useLocalStorage } from '@vueuse/core'

interface PolitePopupStorageDTO {
  status: 'unsubscribed' | 'subscribed'
  seenCount: number
  lastSeenAt: number
}

export const usePolitePopup = () => {
  //...
  const storedData: Ref<PolitePopupStorageDTO> = useLocalStorage('polite-popup', {
    status: 'unsubscribed',
    seenCount: 0,
    lastSeenAt: 0,
  })
  //...
  watch(
    [readTimeElapsed, scrolledContent],
    ([newReadTimeElapsed, newScrolledContent]) => {
      if (newReadTimeElapsed && newScrolledContent) {
        visible.value = true;
        storedData.value.seenCount += 1;
        storedData.value.lastSeenAt = new Date().getTime();
      }
    }
  );
  //...
  return {
    visible,
    trigger
  }
}
```

<!-- 
* status is per default unsubscribed and is set to subscribed if a visitor subscribes to the newsletter.
* seenCount tracks how often the user has seen the popup.
* lastSeenAt tracks the timestamp when the visitor has seen the popup. 
-->

---

## Trigger timer

In `[..slug].vue` we trigger the timer if the route path equal `/vue`:

```vue {7-15}
<template>
  <main>
    <ContentDoc />
  </main>
</template>

<script setup lang="ts">
const route = useRoute();

const { trigger } = usePolitePopup();

if (route.path === "/vue") {
  trigger();
}
</script>
```

---
layout: iframe-demo
url: "https://vuejsnation-2023-polite-popup-nuxt-3.netlify.app/?demoMode=inpolite-popup"
---

# Demo Time

[Source Code](https://stackblitz.com/edit/vuejsnation-2023-lightning-talk-polite-popup-nuxt-3)

---
layout: article
---

# Blog Post

For more details read [the corresponding blog post](https://mokkapps.de/blog/building-a-polite-newsletter-popup-with-nuxt-3/)

![Blog Post Image](https://res.cloudinary.com/mokkapps/image/upload/v1673367765/Xnapper-2023-01-10-17.22.06_lkwtbb.png)

---
layout: image-right
image: https://media.giphy.com/media/l3vR4yk0X20KimqJ2/giphy.gif
---

# Thanks to the amazing people behind

- [Vue](https://vuejs.org/)
- [VueUse](https://vueuse.org/)
- [Nuxt 3](https://nuxt.com/)
- [Slidev](https://sli.dev/)


---
layout: outro
---

# Thank you for listening!

Questions?

[Repository](https://github.com/Mokkapps/vuejsnation-2023-lightning-talk-polite-popup-nuxt-3-slides) / [Slides](https://vuejsnation-2023-talk-polite-popup.netlify.app/)
