# Storage Pressure Event


## Authors:



*   jarrydg@chromium.org


## Participate



*   [Issue tracker]
*   [Discussion forum]


## Table of Contents [if the explainer is longer than one printed page]

[You can generate a Table of Contents for markdown documents using a tool like [doctoc](https://github.com/thlorenz/doctoc).]


## Introduction

Add an DOM event that will inform applications about storage pressure (low disk space availability).


## Background

Quota is the system which sets and enforces limits on disk usage by the browser and web applications. Up until recently (M80), the quota system would factor in free disk space when determining how much quota each origin is allowed. This allowed applications to rely on the quota system to know exactly how much disk space they had to write to. **Without information about what will and will not fit on disk, applications that depend on storage usage cannot guarantee a working experience to their users.**

In the current design, free disk is no longer incorporated into the calculation of an application’s storage quota due to a security issue. This leaves applications and developers in a place where they have no guarantees around how much space they can write to, and must wait for write errors before realizing they need to scale back their offline caching.


## Goals [or Motivating Use Cases, or Scenarios]

The primary objective is to provide applications with a signal indicating a limited amount of storage resources, which will allow them ample time to adjust their caching strategies and ensure a smooth and uninterrupted experience for their users.


## Non-goals

Expose precise information about free disk space.


## Proposed API

Add a storage pressure event listener


```
navigator.storage.addEventListener('storagepressure', event => {
        // clear all IndexedDB databases to make room for new offline
        // content
        const dbs = await window.indexedDB.databases();
dbs.forEach(db => { window.indexedDB.deleteDatabase(db.name) });
});
```



## Key scenarios

[If there are a suite of interacting APIs, show how they work together to solve the key scenarios described.]


### Scenario 1

Consider an email client that uses IndexedDB to store text and Cache Storage to store attachments. With offline mode enabled, the client will sync the last 30 days of emails to disk by default. Without any indication of free disk space, the client can run out of space after partial caching and end up in a wedged state. 

Consider a new user that enables offline mode for the first time. The mail client will begin caching emails, starting with the most recent. In the middle of caching for offline, a storage pressure event is dispatched. The application recognizes that there will not be enough space to cache all email text and assets from the last month, so it stops syncing attachments, and maybe it even starts removing some attachments to ensure there is enough space to sync all email text from the past month.


```
let emailAttachmentMaxAgeInDays = 30;
let emailTextMaxAgeInDays = 30;

navigator.storage.addEventListener('onstoragepressure', event => {
	emailAttachmentMaxAgeInDays = 7;
});

function cacheResourcesByAge() {
	for (let email of emailList) {
		let ageInDays = moment.duration(
			moment(email.sentAt)
			.diff(moment().startOf('day'))
		).asDays()

		if (age < emailTextMaxAgeInDays) {
			// indexedDB work to store email text
		}
		if (age < emailAttachmentMaxAgeInDays) {
			// cache storage work for attachments
		}
	}
}
```



## Considered alternatives


### Trigger

There are two flows which can be hooked into to trigger said event:



1. When QuotaManager is queried to make sure there is enough quota to write to disk (QuotaManager::UsageAndQuotaInfoGatherer::Completed())
    *   has most recent numbers on disk availability
    *   this also runs when DevTools polls for quota (every second when Clear Storage section is open)
    *   may not be temporally relevant if nothing is written anyway
2. After writes happen (QuotaManager::NotifyStorageModified())
    *   currently does not poll for disk availability, and adding this is another expensive operation
    *   if we skip polling for disk stats and used usage_delta + cached_usage, cached_usage may not be up-to-date

The proposal is to move forward with option number one because it has fresh information regarding disk availability. Rate limiting calls into the storage pressure flow can help mitigate the problems associated with DevTools updates. It’s unclear right now whether or not the temporal relevance is an issue.


### Context

StorageManager is currently exposed in all workers, including service workers. This means that, in the current design, this API will be exposed to service workers. Storage work is oftentimes done from service workers, and work done from there is subject to the same problems described in the beginning of this document. It rarely makes sense to dispatch the same event in service workers as you would in windows, but given the amount of storage work done from within service workers, this API should support them. The event will not, however, start a service worker.


### Params / Levers

We can adjust parameters in order to tweak



*   How often events are fired
*   The delay between crossing the moment the disk availability threshold and the moment the event is fired
*   The threshold randomization


## Stakeholder Feedback / Opposition

[Implementors and other stakeholders may already have publicly stated positions on this work. If you can, list them here with links to evidence as appropriate.]



*   [Implementor A] : Positive
*   [Stakeholder B] : No signals
*   [Implementor C] : Negative

[If appropriate, explain the reasons given by other implementors for their concerns.]


## References & acknowledgements

Many thanks for valuable feedback and advice from:



*   jsbell@chromium.org
*   mek@chromium.org
*   [pwnall@chromium.or](mailto:pwnall@chromium.org)g
