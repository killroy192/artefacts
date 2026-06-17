
URL: https://github.com/calcom/cal.diy/issues/28923

## Issue Summary

There is a potential data exposure vulnerability in packages/app-store/_utils/getConnectedApps.ts.

The current implementation uses object destructuring with a deny-list approach:

({ credentials: _, credential, key: _2, ...app })

While specific sensitive fields are removed, the remaining properties are blindly spread using app into the API response.

This approach is unsafe because:

- It implicitly exposes all non-excluded fields
- Any new fields added in the future (e.g., internal configs, metadata)
  will be automatically leaked
- It violates the principle of explicit data control in API design

Additionally, there is an internal developer note highlighting this issue:

// TODO: Refactor this to pick up only needed fields and prevent more leaking

---

## Steps to Reproduce

1. Open:
   packages/app-store/_utils/getConnectedApps.ts

2. Navigate to lines 149–151

3. Observe the mapping logic:
   - Sensitive fields are excluded using destructuring
   - Remaining object is spread into the response

4. Call getConnectedApps API -> All fields will be exposed to the frontend unintentionally

---

## Actual Result

- The API uses a deny-list pattern
- All unspecified properties are automatically included in the response
- This can lead to unintentional leakage of internal or sensitive metadata

---

## Expected Result

The API should follow a strict allow-list approach, where only explicitly defined safe fields are returned.

Example:

return {
  slug: app.slug,
  name: app.name,
  logo: app.logo,
  categories: app.categories,
  variant: app.variant,
};

This ensures:

- No accidental exposure of new/internal fields
- Safer and predictable API behavior
- Better long-term maintainability

---

## Technical Details

- File: packages/app-store/_utils/getConnectedApps.ts
- Lines: 149–151
- Component: Backend utilities / tRPC layer
- Environment: Backend logic issue (browser independent)
