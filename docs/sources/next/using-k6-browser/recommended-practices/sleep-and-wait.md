---
title: 'Sleep and Wait*'
description: 'A guide on how to work with sleep and wait*.'
weight: 04
---

We will discuss the differences between `sleep` in `k6` and the various `wait*` prepended methods that are available in `k6/browser`. By the end of this page, you should be able to successfully use the correct API where necessary.

{{< admonition type="note" >}}

To avoid flaky tests, don't work with `sleep` or `waitForTimeout` to wait for things to appear/disappear/occur (such as waiting for elements to appear/disappear or navigating to a different page). They're useful for mimicking a real user by slowing the test down between API calls, or to debug your test.

{{< /admonition >}}

# What is `sleep`?

[sleep](https://grafana.com/docs/k6/<K6_VERSION>/javascript-api/k6/sleep) is a first class function built into k6. It's main use is to _"suspend VU execution for the specified duration"_ which is most useful when you want to replicate user behavior, such as:

- Navigating to a page
- sleeping for a 1 second to mimic a user looking for a specific element
- clicking on the element
- etc.

{{< admonition type="warning" >}}

`sleep` is a synchronous function that blocks the JS event loop, which means that all asynchronous work will also be suspended until the `sleep` completes. The browser module predominantly provides asynchronous APIs, and so it's best to avoid working with `sleep`. **We recommend you use [waitForTimeout](<#waitForTimeout(timeout)>) instead of `sleep`**.

{{< /admonition >}}

# What is `wait*`?

In the browser modules there are various asynchronous APIs that can be used to wait for certain states:

| Method | Description |
| [waitForFunction(pageFunction, arg[, options])](#waitForFunction(pageFunction, arg[, options])) | waits for the given function to return a truthy value. |
| [waitForLoadState(state[, options])](#waitForLoadState(state[, options])) | waits for the specified page life cycle event. |
| [waitForNavigation([options])](<#waitForNavigation([options])>) | waits for the navigation to complete after one starts. |
| [waitForSelector(selector[, options])](#waitForSelector(selector[, options])) | waits for the given selector to match the waiting criteria. |
| [waitForTimeout(timeout)](<#waitForTimeout(timeout)>) | waits the given time. |

## waitForFunction(pageFunction, arg[, options])

[waitForFunction](https://grafana.com/docs/k6/<K6_VERSION>/javascript-api/k6-browser/page/waitForFunction) is useful when you want more control over when a test progresses with a javascript function that returns true when a condition (or many conditions) is met. It can be used to poll for changes in the dom or non dom elements and variables.

{{< code >}}

<!-- eslint-skip-->

```javascript
import { browser } from 'k6/browser';
import { check } from 'k6';

export const options = {
  scenarios: {
    browser: {
      executor: 'shared-iterations',
      options: {
        browser: {
          type: 'chromium',
        },
      },
    },
  },
};

export default async function () {
  const page = await browser.newPage();

  try {
    // Setting up the example that will mutate the h1 element by setting the
    // h1 elements text value to "Hello".
    await page.evaluate(() => {
      setTimeout(() => {
        const el = document.createElement('h1');
        el.innerHTML = 'Hello';
        document.body.appendChild(el);
      }, 1000);
    });

    // Waiting until the h1 element has mutated.
    const ok = await page.waitForFunction("document.querySelector('h1')", {
      polling: 'mutation',
      timeout: 2000,
    });

    const innerHTML = await ok.innerHTML();
    check(ok, { 'waitForFunction successfully resolved': innerHTML == 'Hello' });
  } finally {
    await page.close();
  }
}
```

{{< /code >}}

## waitForLoadState(state[, options])

[waitForLoadState](https://grafana.com/docs/k6/<K6_VERSION>/javascript-api/k6-browser/page/waitForLoadState) is useful when there’s no explicit navigation, but you need to wait for the page or network to settle. This is mainly used when working with single page applications or when no full page reloads happen.

{{< code >}}

```javascript
import { check } from 'k6';
import { browser } from 'k6/browser';

export const options = {
  scenarios: {
    browser: {
      executor: 'shared-iterations',
      options: {
        browser: {
          type: 'chromium',
        },
      },
    },
  },
};

export default async function () {
  const page = await browser.newPage();

  try {
    // Goto a SPA
    await page.goto('<url>');

    // ... perform some actions that reload part of the page.

    // waits for the default `load` event.
    await page.waitForLoadState();
  } finally {
    await page.close();
  }
}
```

{{< /code >}}

## waitForNavigation([options])

[waitForNavigation](https://grafana.com/docs/k6/<K6_VERSION>/javascript-api/k6-browser/page/waitForNavigation) is a very useful API when performing other actions that could start a page navigation, and they don't automatically wait for a navigation to end. Usually you'll find it in our examples with a `click` API call. [goto](https://grafana.com/docs/k6/<K6_VERSION>/javascript-api/k6-browser/page/goto) is an example of an API that **doesn't** require `waitForNavigation` since it will automatically wait for the navigation to complete before returning.

It's important to call this in a [Promise.all](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all) along with the API that will cause the navigation to start.

{{< code >}}

```javascript
import { check } from 'k6';
import { browser } from 'k6/browser';

export const options = {
  scenarios: {
    browser: {
      executor: 'shared-iterations',
      options: {
        browser: {
          type: 'chromium',
        },
      },
    },
  },
};

export default async function () {
  const page = await browser.newPage();

  try {
    await page.goto('https://test.k6.io/my_messages.php');

    await page.locator('input[name="login"]').type('admin');
    await page.locator('input[name="password"]').type('123');

    const submitButton = page.locator('input[type="submit"]');

    // The click action will start a navigation, and the waitForNavigation
    // will help the test wait until the navigation completes.
    await Promise.all([page.waitForNavigation(), submitButton.click()]);
  } finally {
    await page.close();
  }
}
```

{{< /code >}}

## waitForSelector(selector[, options])

[waitForSelector](https://grafana.com/docs/k6/<K6_VERSION>/javascript-api/k6-browser/page/waitForSelector) will wait until the given selector meets the waiting criteria. It's useful when dealing with dynamic websites where elements may take time to appear or change state (they might load after some delay due to async calls, JavaScript execution, etc.).

{{< code >}}

```js
import { browser } from 'k6/browser';

export const options = {
  scenarios: {
    browser: {
      executor: 'shared-iterations',
      options: {
        browser: {
          type: 'chromium',
        },
      },
    },
  },
};

export default async function () {
  const page = await browser.newPage();
  await page.goto('https://test.k6.io/browser.php');
  await page.waitForSelector('#text1');
}
```

{{< /code >}}

## waitForTimeout(timeout)

[waitForTimeout](https://grafana.com/docs/k6/<K6_VERSION>/javascript-api/k6-browser/page/waitForTimeout) will wait the given amount of time. It's functionally the same as k6's [sleep](#What-is-`sleep`) but it is asynchronous which means it will not block the event loop, thus allowing the background tasks to continue to be worked on. We're also planning on instruementing it with tracing to then allow us visualize it in the timeline in grafana cloud k6.

{{< code >}}

```js
import { browser } from 'k6/browser';

export const options = {
  scenarios: {
    browser: {
      executor: 'shared-iterations',
      options: {
        browser: {
          type: 'chromium',
        },
      },
    },
  },
};

export default async function () {
  const page = await browser.newPage();

  try {
    await page.goto('https://test.k6.io');

    // Slow the test down to mimic a user looking for the element on the page.
    await page.waitForTimeout(1000);

    // ... perform the next action
  } finally {
    await page.close();
  }
}
```

{{< /code >}}
