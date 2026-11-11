# Extension analysis points

| Field | Risk signal |
|------|----------|
| host_permissions `<all_urls>` | Can read and write any site |
| webRequestBlocking | Man-in-the-middle-style rewriting |
| nativeMessaging | Exits the browser to the local machine |
| externally_connectable | Web pages can drive the extension |

MV3:Focus on service_worker lifecycle and declarativeNetRequest.
