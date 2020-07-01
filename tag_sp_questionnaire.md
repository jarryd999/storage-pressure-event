# TAG Review: Security and Privacy questionnaire

### 2.1. What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?
This API exposes whether or not a device is below a certain threshold of free disk space. The threshold will be implementation specific, and may be either a fixed or variable threshold.

The event gives applications ample time to adjust caching strategies before running out of disk space, when write errors would start surfacing and potentially, system instability.

### 2.2. Is this specification exposing the minimum amount of information necessary to power the feature?
Yes, we think this is the minimum amount of information needed to address the use cases in the explainer. This feature will not reveal precise information about disk availability.

### 2.3. How does this specification deal with personal information or personally-identifiable information or information derived thereof?
This feature does not deal with personally-identifiable information and thus will not expose any personally-identifiable information.

We don't believe that the amount of available disk space can be used to identify somebody, and thus feel that this feature is not dealing with personal information.

### 2.4. How does this specification deal with sensitive information?
We don’t think that the amount of free disk space on a device is sensitive information.

### 2.5. Does this specification introduce new state for an origin that persists across browsing sessions?
Potentially, in keeping track of when the event was last dispatched so as to debounce events on a per-listener/per-origin basis. This specification will leave it up to implementers whether or not debounce state should persist across browsing sessions. The non-normative section of the explainer recommends against this.

### 2.6. What information from the underlying platform, e.g. configuration data, is exposed by this specification to an origin?
This API exposes whether or not a device is below a certain threshold of free disk space. We recommend that implementers add noise to this information.

### 2.7. Does this specification allow an origin access to sensors on a user’s device
No.

### 2.8. What data does this specification expose to an origin? Please also document what data is identical to data exposed by other features, in the same or different contexts.
This API exposes whether or not a device is below a certain threshold of free disk space.  Said threshold will be implementation specific, but the specification recommends adding noise to further reduce the information exposed.

We believe that most users' devices will be operating above the storage pressure threshold. This means that the information exposed by this feature will be a fraction of a bit of information when viewed across all users.

Currently it is possible to detect when a device is completely out of disk space using storage features such as IndexedDB or Cache Storage. Applications can use these features to write until writes begin to error out, at which point they know the disk is full.

### 2.9. Does this specification enable new script execution/loading mechanisms?
No

### 2.10. Does this specification allow an origin to access other devices?
No.

### 2.11. Does this specification allow an origin some measure of control over a user agent’s native UI?
No.

### 2.12. What temporary identifiers might this specification create or expose to the web?
None.

### 2.13. How does this specification distinguish between behavior in first-party and third-party contexts?
No distinction.

### 2.14. How does this specification work in the context of a user agent’s Private Browsing or "incognito" mode?
Incognito implementations tend to be RAM backed, so the event doesn't make sense in this context and thus, the specification does not recommend dispatching the event for incognito sessions.

### 2.15. Does this specification have a "Security Considerations" and "Privacy Considerations" section?
This is expected to be an addition to the Storage spec, which contains both security and privacy consideration sections, and the explainer contains expected additions to those sections.

### 2.16. Does this specification allow downgrading default security characteristics?
No.

### 2.17. What should this questionnaire have asked?
The questions seem adequate.
