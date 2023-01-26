---
theme: mokkapps
title: "Building a Polite Popup with Nuxt 3"
# lineNumbers: true
colorSchema: 'light'
exportFilename: 'vuejs-nation-2023-lightning-talk-polite-popup'
# provide a downloadable PDF:
download: true
globalBottomPosition: 'left'
---

# Building a Polite Popup with Nuxt 3

Vue.js Nation 2023 - Lightning Talk

---
layout: about-me
---



<!--
- run a weekly vue newsletter
- very active on Twitter
- follow if you are interested in Vue & Nuxt
-->

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

- <twemoji-warning /> Problem 1: Another annoying full-page popup on the landing page
- <twemoji-warning /> Problem 2: Visitors haven't engaged with the related content
- <twemoji-warning /> Problem 3: Visitors are asked every time they visit the page

</v-clicks>

<!--
We all know the typical website nowadays

1. Ads, right corner
2. Cookies, top center
3. A popup that wants us to subscribe to a newsletter
4. Navigate to Vue page
5. Navigate back home, we are asked again
-->

---
layout: section
---

## What is a "Polite Popup"?

---
layout: iframe-right
url: >-
  https://vuejsnation-2023-polite-popup-nuxt-3.netlify.app/?demoMode=polite-popup
---

# Let's build a polite popup

A polite popup appears to visitors if they

<v-clicks>

- are **visiting a page with Vue-related content** as the newsletter targets Vue developers
- are **actively scrolling** the current page for **6 seconds or more**
- **scroll through at least 35%** of the current page during their visit

</v-clicks>

<v-click>

</v-click>

<!--
- Nuxt Content App
- React page -> no popup
- Visitors will be more likely to sign up because they are asked after they’ve decided they liked the content
-->

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

Let's start by writing a Vue composable for our polite popup:

```ts {1,10|2|4|6-9}
export const usePolitePopup = () => {
  const visible = ref(false);

  const trigger = () => {}

  return {
    visible,
    trigger,
  };
};
```

<!--
* We call it usePolitePopup
* visible: reactive variable, a boolean indicating if the popup should be visible or not.
* The trigger method is exposed and will be filled later. 
-->

---

# Track time spent on page

The visitor must be actively scrolling the current page for 6 seconds or more.

```ts {7|1|9,14|7,10-12|3,13|7,9,17-19} {maxHeight:'350px'}
import { useTimeoutFn } from '@vueuse/core'

const config = { timeoutInMs: 6000 } as const

export const usePolitePopup = () => {
  const visible = ref(false);
  const readTimeElapsed = ref(false)

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

<!--
* readTimeElapsed: reactive variable indicating if the user has spent the defined time on the page.
* VueUse's useTimeoutFn composable: runs a setTimeout function and sets the readTimeElapsed state variable to true after the timer has expired.
* not immediate, we use the returned start method to start the timeout
* the timeout is trigger if the `trigger` method is called
-->


---

# Track scroll progress

The visitor must scroll through at least 35% of the current page during their visit.

```ts {10,11,16|12|1,7,12,13|1,8,14,15|12-15|3,11,16,18-20} {maxHeight:'350px'}
import { useWindowSize, useWindowScroll } from '@vueuse/core'

const config = { timeoutInMs: 6000, contentScrollThresholdInPercentage: 35, } as const

export const usePolitePopup = () => {
    //...
    const { height: windowHeight } = useWindowSize()
    const { y: scrollTop } = useWindowScroll()

    // Returns percentage scrolled (ie: 80 or NaN if trackLength == 0)
    const amountScrolledInPercentage = computed(() => {
      const documentScrollHeight = document.documentElement.scrollHeight
      const trackLength = documentScrollHeight - windowHeight.value
      const scrollPercent = scrollTop.value / trackLength;
      const scrollPercentRounded = Math.floor(scrollPercent * 100);
      return scrollPercentRounded;
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

<v-click at="1">

<img v-if="$slidev.nav.clicks <= 4" src="/scroll-progress.png" class="absolute left-150 top-50 h-50 border rounded shadow" />

</v-click>

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
    const visible = ref(false)
    const readTimeElapsed = ref(false)
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

<!--
Vue Watcher!
-->

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

In `[..slug].vue` we trigger the timer if the route path is equal to `/vue`:

```vue {7,8,12,14,15|7,10,12-15}
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

<!--
- Catch-all route
- check path
- trigger from our composable
-->

---
layout: image-right
image: https://media.giphy.com/media/dkGhBWE3SyzXW/giphy.gif
---

# We're done!

<v-click>

We implemented the main logic for a polite popup in Nuxt 3 💪🏻

</v-click>

<v-click>

Thanks to the amazing people behind

</v-click>

<v-clicks>

- [Vue](https://vuejs.org/)
- [VueUse](https://vueuse.org/)
- [Nuxt 3](https://nuxt.com/)
- [Slidev](https://sli.dev/)

</v-clicks>

<v-click>

</v-click>

---
layout: article
---

# You want more?

For more details read [the corresponding blog post](https://mokkapps.de/blog/building-a-polite-newsletter-popup-with-nuxt-3/)

![Blog Post Image](https://res.cloudinary.com/mokkapps/image/upload/v1673367765/Xnapper-2023-01-10-17.22.06_lkwtbb.png)

<!--
- Missing some functionality
- the popup Vue component
- extended visibility logic
- all details in the blog post
-->

---
layout: outro
---

# Thank you for listening!

Questions?

[Repository](https://github.com/Mokkapps/vuejsnation-2023-lightning-talk-polite-popup-nuxt-3-slides) / [Slides](https://vuejsnation-2023-talk-polite-popup.netlify.app/)
