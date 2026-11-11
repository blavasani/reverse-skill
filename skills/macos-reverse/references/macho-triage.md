# Mach-O Triage

```bash
file ./app
otool -hv ./app
otool -l ./app | head
codesign -d --entitlements :- ./app
```

Focus on `com.apple.security.*` entitlements, Library Validation, and flags that disable library injection.
