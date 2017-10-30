---
layout: docs
title: 'The StorageManager API'
---

<img src="/assets/images/disc.jpg" style="float:left;margin:24px;padding-right:16px;" />

Even though IndexedDB is a fully functional client-side database for the web, it cannot be used as a relyable, persistent storage by default. IndexedDB without [StorageManager](https://developer.mozilla.org/en-US/docs/Web/API/StorageManager), is like a very advanced cookie store and nothing more. A user can delete your data at any time, and the browser can delete it without noticing the user in case it believes that your database is taking up too much disk space on a device.

If you are syncing your data towards a server, these limitations are ok to live with, as a resync would restore your data. If not syncing, or in case your app requires a large quota, you should consider using the [StorageManager API](https://developer.mozilla.org/en-US/docs/Web/API/StorageManager) to control how your database is stored and protected.

## Controlling Persistence

If you need to prohibit the user from accidentally getting the data deleted, you should try putting the following code in your app's bootstrap somewhere:

```javascript

async function persist() {
  await navigator.storage && navigator.storage.persist && navigator.storage.persist();
}

```

This does not guarantee you be allowed to persist the database. The browser may pop up a dialog to the user, asking for the permission to persist the storage. On older browsers without the StorageManager API the function will not do anything, as it initially checks for the existance of the StorageManager API and it's persist method.

To check whether your IndexedDB database is successfully persisted, inspect the returned promise returned by persist(), or use the following function to query it without trying to persist:

```javascript

async function isStoragePersisted() {
  return await navigator.storage && navigator.storage.persisted && navigator.storage.persisted();
}

```

Example of use:

```javascript

isStoragePersisted().then(async isPersisted => {
  if (isPersisted) {
    console.log(":) Storage is successfully persisted.");
  } else {
    console.log(":( Storage is not persisted.");
    console.log("Trying to persist..:");
    if (await persist()) {
      console.log(":) We successfully turned the storage to be persisted.");
    } else {
      console.log(":( Failed to make storage persisted");
    }
  }
})
```

## What is "storage" and how does it apply to Dexie?

Dexie is just a wrapper for IndexedDB and enables the creation of (and access to) client databases in your browser. These databases are bound to a certain origin - the part of the URL before its path starts (for example, http://dexie.org is the origin of this page).
Each origin can contain various storages, for example localStorage, indexedDB databases, etc. All of these APIs put their data in a common [box](https://storage.spec.whatwg.org/#introduction) that is bound to the origin of the page. This box has a  mode that can be either "persistent" or "best-effort".

## Prohibit Unwanted Dialogs?

Using navigator.storage.persist() may prompt the current user for permission. Personally, I would not like a webapp to ask for a permission the first thing it did. I would rather provide my own GUI to advertise that based on my app's criterias. For example, when the user seems to get more involved with the application, I could advertise and explain that the app might need to ensure that the data will not be accidentially cleared without user's notice.

Luckily, this has been thought of in the specification and can be accomplished, as you may convert existing databases (or rather origins containing all kinds of storages, including IndexedDB databases) to become persisted. There is also a [permissions](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/permissions) API that lets you ask whether a certain permission needs to be prompted for or not. User may have configured the browser to allow this already, or your app is run in installed mode and get the permissions implicitely. See [Summary](#summary) for a sample on how to control user dialogs.


## How Much Data Can Be Stored?

If you successfully made your storage persistent, the quota it is allowed to use may vary depending on device. Typically, persisted storages will be allowed to use a larger quota than non-persisted storages.
You might want to show your user how much storage is available, or you might want to take actions when storage reaches a certain percent of available storage. This can be accomplished using [StorageManager.estimate()](https://developer.mozilla.org/en-US/docs/Web/API/StorageManager/estimate) as shown in the sample below:

```javascript

async function showEstimatedQuota() {
  if (navigator.storage && navigator.storage.estimate) {
    const estimation = await navigator.storage.estimate();
    console.log(`Quota: ${estimation.quota}`);
    console.log(`Usage: ${estimation.usage}`);
  } else {
    console.error("StorageManager not found");
  }
}

```

## Caveats

Some things to consider:

* Your app must be served over HTTPS, as the StorageManager() is only available in secure contexts.
* StorageManager API is still considered an experimental technology but is already available on Chrome, Firefox and Opera (as of October 30, 2017).

## Workers

Web Workers and Service Workers access the storage API the same way as web pages do - through `navigator.storage`.

## Summary

If you are storing critical data with Dexie (or in IndexedDB generally), you might consider using [StorageManager](https://developer.mozilla.org/en-US/docs/Web/API/StorageManager) to ensure the data can be persistently stored, and not just "best-effort". For user experience, some apps may want to wait with enabling the persistend mode until the user seems to be repeatingly using the application, or maybe using certain parts of the application where persisted storage is critical.

In summary, here are some handy functions to use:

```javascript

/** Check if storage is persisted already.
  @returns {Promise<boolean>} Promise resolved with true if current origin is using persistent
  storage, false if not, and undefined if the API is not present.
*/
async function isStoragePersisted() {
  return await navigator.storage && navigator.storage.persisted ?
    navigator.storage.persisted() :
    undefined;
}

/** Tries to convert to persisted storage.
  @returns {Promise<boolean>} Promise resolved with true if successfully persisted the storage,
  false if not, and undefined if the API is not present.
*/
async function persist() {
  return await navigator.storage && navigator.storage.persist ?
    navigator.storage.persist() :
    undefined;
}

/** Queries available disk quota.
  @see https://developer.mozilla.org/en-US/docs/Web/API/StorageEstimate
  @returns {Promise<{quota: number, usage: number}>} Promise resolved with
  {quota: number, usage: number} or undefined.
*/
async function showEstimatedQuota() {
  return await navigator.storage && navigator.storage.estimate ?
    navigator.storage.estimate() :
    undefined;
}

/** Tries to persist storage without ever prompting user.
  @returns {Promise<string>}
    "never" In case persisting is not ever possible. Caller don't bother asking user for
    permission.
    "prompt" In case persisting would be possible if prompting user first.
    "persisted" In case this call successfully silently persisted the storage, or if it
    was already persisted.
*/
async function tryPersistWithoutPromtingUser() {
  if (!navigator.storage || !navigator.storage.persisted) {
    return "never";
  }
  let persisted = await navigator.storage.persisted();
  if (persisted) {
    return "persisted";
  }
  if (!navigator.permissions || !navigator.permissions.query) {
    return "prompt"; // It MAY be successful to prompt. Don't know.
  }
  const permission = await navigator.permissions.query({name: "persistent-storage"});
  if (permission.status === "granted") {
    persisted = await navigator.storage.persist();
    if (persisted) {
      return "persisted";
    } else {
      throw new Error("Failed to persist");
    }
  }
  if (permission.status === "prompt") {
    return "prompt";
  }
  return "never";
}


```

And to use it from your app:

```javascript

async function initStoragePersistence() {
  const persist = await tryPersistWithoutPromtingUser();
  switch (persist) {
    case "never":
      console.log("Not possible to persist storage");
      break;
    case "persisted":
      console.log("Successfully persisted storage silently");
      break;
    case "prompt":
      console.log("Not persisted, but we may prompt user when we want to.");
      break;
  }
}

```

If the result was "prompt" you could show your own view where you explain the reason for persistence along with a button to enable it.
When user presses the button, you call `navigator.storage.persist()`.
