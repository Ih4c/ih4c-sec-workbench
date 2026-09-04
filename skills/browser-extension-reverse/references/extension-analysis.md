# Extension Analysis Key Points

| Field | Risk signal |
|------|----------|
| host_permissions `<all_urls>` | Can read/write any site |
| webRequestBlocking | Man-in-the-middle style rewriting |
| nativeMessaging | Exits the browser to the local machine |
| externally_connectable | Web pages can drive the extension |

MV3: watch the service_worker lifecycle and declarativeNetRequest.
