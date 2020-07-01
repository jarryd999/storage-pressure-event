# Storage Pressure Event


## Authors:

*   jarrydg@chromium.org
*   pwnall@chromium.org


## Participate

*   [Issue tracker](https://github.com/jarryd999/Storage-Pressure-Event/issues)


## Table of Contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [Introduction](#introduction)
- [Background](#background)
- [Motivating use cases](#motivating-use-cases)
- [Non-goals](#non-goals)
- [Proposed API](#proposed-api)
- [Considered alternatives](#considered-alternatives)
  - [Event naming](#event-naming)
  - [Event target](#event-target)
  - [Service worker availability](#service-worker-availability)
  - [Event dispatch logic](#event-dispatch-logic)
  - [Storage space reservation](#storage-space-reservation)
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)
- [References & acknowledgements](#references--acknowledgements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


## Introduction


This proposal aims to provide an early warning when the storage device backing
the user's data is running low on free space. This warning is provided to
currently running applications, which can pause or abandon in-progress
operations that would otherwise fill up the user's storage device.


## Background

The Storage standard
[recommends](https://storage.spec.whatwg.org/#storage-quota) that
"[the storage quota of a storage shelf] should be less than the total
available storage space on the device to give users some wiggle room." In the
past, Chrome treated this recommendation as a requirement, and used the free
disk space as a hard cap for quota. In other words, for certain users, a
site's quota would match the available disk space.

Chrome's approach became [a security problem](https://crbug.com/960305) when
combined with the
[API exposed in the Storage Standard](https://storage.spec.whatwg.org/#api),
which includes
[a quota estimate method](https://storage.spec.whatwg.org/#dom-storagemanager-estimate).
The method allows sites to obtain their available quota which, for some Chrome
users, used to reflect their free disk space. The ability to precisely
measure free disk space is a security problem, because it can be used to measure
the size of cross-origin resources. The same ability may also be used for
fingerprinting.

In order to fix the security problem described above, Chrome switched to
granting sites a fixed amount of quota, which does not account for the amount of
free disk space.

While this change fixed the security problem, it was a regression for web
applications that relied on the details of Chrome's implementation of the
Storage API to provide useful functionality to their users.

One class of user experience regressions is that applications are no longer able
to avoid downloading a large corpus of data when the free disk space is low.
For example, some applications would not enable offline sync for users with
low free disk space. This was an attempt to avoid a bad user experience.
Enabling offline sync would lead to a large amount of data being downloaded
and stored locally, which would lead to running out of disk space, which
could cause system instability such as OS crashes. **This proposal aims to
address this class of regressions.**

Another class of user experience regressions is that applications lost the
ability to surface the amount of free disk space inside their UIs. Applications
did this to facilitate storage management decisions. For example, some
applications show the available quota (which used to reflect free disk space)
when asking their users if they wish to enable offline. Another (currently
theoretical) example is a management UI that allows a user to decide which
media items (movies, song playlists) to sync for offline access, and shows
the amount of free disk space on the side. **This proposal does not tackle
this class of regressions.**


## Motivating use cases

This proposal aims to enhance applications that model the user's data as a
collection of discrete items, and work offline by storing a copy of some of
these items on the user's device (locally), via storage APIs such as
[IndexedDB](https://w3c.github.io/IndexedDB/) and the
[CacheStorage API](https://w3c.github.io/ServiceWorker/#cachestorage). When the
user is online, the local copy of the data is kept in sync with the data on
the server. When the user is offline, the local copy is presented to the user.
This data model applies to the following classes of applications.

* Email clients, such as [Gmail](https://www.google.com/gmail/about/) and
  [Microsoft Outlook](https://www.microsoft.com/en-us/microsoft-365/outlook/)
* Document editors, such as [Google Docs](https://www.google.com/docs/about/)
  and [Microsoft Office 365](https://www.microsoft.com/en-us/microsoft-365/products-apps-services)
* Video services, such as [YouTube](https://www.youtube.com/) and
  [Netflix](https://www.netflix.com/)
* Music services, such as [Google Play Music](https://play.google.com/music/) and
  [Spotify](https://www.spotify.com/)
* Cloud ebook readers, such as [Kindle Cloud Reader](https://read.amazon.com/)
  and [Google Play Books](https://play.google.com/books)

A key usage moment for these applications is the initial setup for offline
support. This initialization process may entail copying a large amount of data
to the user's device. For example, the last 90 days of email in a busy Gmail
inbox take up multiple gigabytes. This initialization process can trigger a bad
experience for users who are low on free disk space.

Given the security concerns around exposing the amount of free disk space, the
Web platform does not offer applications a good way to avoid triggering this bad
experience. The only indication that the user may be running out of disk space
is failed storage API calls, which may indicate write errors from the underlying
file system.

Applications can (and should) pause or give up on storing data locally if a
significant number of write errors occur. However, by that point, the free
disk space has been completely exhausted, and all other software running on
the device is suffering from similar errors. Furthermore, on some operating
systems, running out of free disk space results in complete system
instability.

This proposal aims to provide an early warning as free disk space is running
low, which can be used to pause or abandon currently running heavy I/O
operations. The warning should support the following actions.

1. Change the criteria used to select the data that stays offline. For example,
   an email client can choose to keep the very most recent messages, and discard
   everything else.

2. Pause the sync process and prompt the user for action.



## Non-goals

This proposal does not aim to provide a generic hook into the user agent's
eviction process. In other words, we're not introducing a method for all
applications on the user's device to delete data in response to storage
pressure. The event introduced by this proposal is only delivered to
applications with active execution contexts.

We are not currently proposing to expose the amount of free disk space on the
user's device, either exactly or as an approximation.

This explainer does not introduce a storage space reservation system, where
an application states an intent to store some amount of data, and the user
agent either rejects the intent, or makes some effort to ensure that the
application can store the reserved amount of data.


## Proposed API

We are proposing adding a `quotachange` event on the StorageManager interface.
An email client could use this signal to set a global flag that stops any
ongoing sync operation.

```javascript
let underStoragePressure = false;
navigator.storage.addEventListener('quotachange', event => {
  underStorarePressure = true;
});

async function syncEmail() {
  let pageIndex = 0;
  for (let pageIndex = 0; !underStoragePressure; ++pageIndex) {
    const emailIds = await fetchEmailIds(pageIndex);
    if (emailIds.length === 0 || underStoragePressure) break;

    for (const emailId of emailIds) {
      const email = await fetchEmail(emailId);
      if (underStoragePressure) break;

      await storeEmail(email);
    }
  }
}
```


## Considered alternatives

### Event naming

The newly introduced event is named `quotachange`. Given the intended use, a
name like `storagepressure` would have been more intuitive.

The `quotachange` name was chosen because Firefox intends to signal low disk
space to the application by reducing the quota for the entire web
application, or for some of its [storage
buckets](https://github.com/whatwg/storage/issues/2).

Chrome would also like to have quota adapt to free space, provided we can do so
while respecting the user's security and privacy. For this reason, we're happy
to align on an event name that opens up this future possibility.


### Event target

The event will be dispatched on
[the StorageManager object](https://storage.spec.whatwg.org/#storagemanager).
We have also considered dispatching on
[the Navigator object](https://html.spec.whatwg.org/multipage/system-state.html#the-navigator-object).

We settled for the StorageManager object because it also has the quota-related
methods
[estimate](https://storage.spec.whatwg.org/#dom-storagemanager-estimate) and
[persist](https://storage.spec.whatwg.org/#dom-storagemanager-persist). In a
future with
[multiple storage buckets](https://github.com/whatwg/storage/issues/2), the
StorageManager methods will operate on the default bucket. The same methods will
be available on objects representing individual storage buckets.


### Service worker availability

Code running in service workers will be able to listen for the storage pressure
event. The event dispatched to service workers will not be an
[ExtendableEvent](https://w3c.github.io/ServiceWorker/#extendableevent).

We considered the following alternatives.

1. The storage pressure event is not exposed on StorageManager in service
   workers. We rejected this option because it adds significant difficulties
   to sharin code for offline availability between all execution contexts
   (windows, dedicated workers, service workers).

2. The storage pressure event behaves like the
   [fetch event](https://w3c.github.io/ServiceWorker/#fetchevent). Service
   workers can register event handlers and be woken up to handle storage
   pressure events. We rejected this option due to concerns that it would lead
   to a bad user experience. Having all service workers wake up at once and
   attempt to delete non-essential storage would cause I/O contention on the
   user's storage device, slowing down the entire system. Waking up service
   workers for unused applications may also have privacy implications.

3. The storage event does not wake up service workers, but is an
   [ExtendableEvent](https://w3c.github.io/ServiceWorker/#extendableevent), so
   it can be used to keep service workers alive. We rejected this option because
   it adds complexity to the platform (no such event currently exists), and the
   use cases above do not require the ability to extend service worker lifetime.


### Event dispatch logic

While we aim to specify the semantics of the storage pressure event, we
propose designating the specific conditions for dispatching the event up as
user agent implementation details. Based on implementation wisdom, we will
make the following non-normative recommendations.

1. The user agent should consider itself to be under storage pressure when
   the amount of free space on the user's storage device drops below a fixed
   amount, such as 2 GB, or below a fixed proportion, such as 1%, of the
   device's total capacity.

2. When the user agent is under storage pressure, it should dispatch storage
   pressure events to all registered listeners, subject to the constraints
   below.

3. Event dispatch for each listener should be subject to a random delay designed
   to minimize the ability to link the user's identity across origins. For
   example, the user agent can group all listeners by origins, and delay the
   event dispatch for each group by a sample from a uniform distribution
   between 0 and 5 seconds.

4. Events should be de-bounced per-listener and per-origin. For example, a user
   agent may choose to dispatch at most one storage pressure event per listener
   per day.

5. De-bounce state need not be persisted across browser sessions and can live 
   exclusively in memory.
   
6. Most implementations of Incognito Mode tend to be RAM-backed. In this case,
   the event should not be dispatched as it does not make sense to dispatch an
   event related to disk availability.

We chose not to turn any of the recommendations above into normative
specification text, in order to allow implementations to experiment with
dispatch logic, and eventually adopt the best practices that emerge into the
storage pressure event specification.


### Storage space reservation

A space reservation system is an entirely different approach that appears to
address the use cases in this explainer. However, it turns out that the
reservation system still requires a storage pressure signal.

In a reservation system, the application indicates an intent to write a
certain amount of data, with the understanding that the amount is an
approximation. The user agent replies with a grant for some amount of storage,
which may be bigger or smaller than the amount requested. The application may
assume that it can use up the amount of space granted without pushing the system
into storage pressure. After using up the space granted, the application can
make a new request.

The main drawback of the reservation system is that the user agent cannot easily
guarantee space availability. At the time of this writing, on most devices, the
user agent runs at the same time as other software that may use up storage
space. So, while a web application uses up the space it was granted, other
applications may fill up the disk. In conclusion, we still need a storage
pressure signal to let the web application know it needs to back off.

Given the storage pressure event proposed here, a space reservation system
becomes a promising avenue for future work. We would like to draw attention to
the following delicate design issues.

1. The reservation system has the potential of reducing fingerprinting around
   the amount of available disk space. This is because an application may only
   request space reservation after using up any previously granted space, and
   the user agent has flexibility in the amount of space granted. At the same
   time, this setup may create an incentive for applications to learn about
   the amount of free space by filling all of it up, which will also cause
   I/O contention and increased power usage. The reservation system needs to
   deter this class of abuse.

2. The reservation system must be able to prevent a set of collaborating origins
   from figuring out the amount of free space. For example, consider a
   seemingly conservative reservation system that grants 1GB at a time as
   long as the sum of grants does not exceed the free space. This system can
   be defeated by an evil.com site that reserves 1GB of space at a time via
   iframes hosted on suborigin{1...n}.evil.com, until the reservations are
   rejected. Specifically, evil.com gets to learn the amount of free space,
   with 1GB granularity.

3. The reservation system must mitigate the ability to link a user's identity
   across multiple origins under storage pressure. This means the reservation
   system may need to overcommit free space, and use the storage pressure
   signal to prevent well-behaving applications from filling up the user's disk.
  


## Stakeholder Feedback / Opposition

*   Chrome : Positive, authoring this explainer
*   [Firefox](https://github.com/whatwg/storage/issues/73#issuecomment-535541438) :
    Positive, agreed at TPAC 2019
*   Safari : No signals
*   Edge : No signals


## References & acknowledgements

Many thanks for valuable feedback and advice from:

*   Andrew Sutherland
*   Alexandra (Lexi) Stavrakos
*   Anne van Kesteren
*   Joshua Bell
*   Marijn Kruisselbrink
