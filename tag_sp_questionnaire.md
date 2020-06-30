# TAG Review: Security and Privacy questionnaire

### 2.1. What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?
This API exposes whether or not a device is below a certain threshold of free disk space. This gives applications ample time to adjust caching strategies before running out of disk space, when write errors would start surfacing and potentially, system instability.

### 2.2. Is this specification exposing the minimum amount of information necessary to power the feature?
Yes. This feature will not reveal precise information about disk availability.

### 2.3. How does this specification deal with personal information or personally-identifiable information or information derived thereof?
No extra personal information or personally-identifiable information is exposed by this API.

### 2.4. How does this specification deal with sensitive information?
No extra sensitive information is exposed by this API.

### 2.5. Does this specification introduce new state for an origin that persists across browsing sessions?
Potentially, in keeping track of when the event was last dispatched so as to rate limit events on a per-listener/per-origin basis.

### 2.6. What information from the underlying platform, e.g. configuration data, is exposed by this specification to an origin?
None.

### 2.7. Does this specification allow an origin access to sensors on a user’s device
No.

### 2.8. What data does this specification expose to an origin? Please also document what data is identical to data exposed by other features, in the same or different contexts.
This will expose, in boolean form, whether a device is below a certain threshold of free disk space. Depending on the device capacity, the threshold could be either an absolute value, or a percentage of the total device capacity. This is not currently exposed by any other features.

### 2.9. Does this specification enable new script execution/loading mechanisms?
No

### 2.10. Does this specification allow an origin to access other devices?
No.

### 2.11. Does this specification allow an origin some measure of control over a user agent’s native UI?
No.

### 2.12. What temporary identifiers might this specification create or expose to the web?
It exposes WebRTC synchronization sources and contributing sources (already exposed through other APIs).

### 2.13. How does this specification distinguish between behavior in first-party and third-party contexts?
No distinction.

### 2.14. How does this specification work in the context of a user agent’s Private Browsing or "incognito" mode?
The incognito storage pool size is smaller than the threshold considered for this event, so the event will not fire in incognito mode.

### 2.15. Does this specification have a "Security Considerations" and "Privacy Considerations" section?
Yes.

### 2.16. Does this specification allow downgrading default security characteristics?
No.

### 2.17. What should this questionnaire have asked?
The questions seem adequate.
