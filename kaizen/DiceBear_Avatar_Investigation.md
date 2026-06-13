# Investigation: DiceBear Avatars for Local-First Expo Implementation

## Executive Summary
DiceBear is an ideal solution for the E7 ecosystem because it separates the **avatar logic** (the library) from the **avatar delivery** (the API). For our Expo/React Native apps, we can completely bypass the network by using the JavaScript library to generate SVG strings locally, which are then rendered via `react-native-svg`. This ensures zero network overhead and complete control over styling.

---

## 1. Local Rendering (Offline Mode)
To avoid burdening URLs and eliminate network dependency, we should use the **DiceBear JS Library** instead of the HTTP API.

### Technical Workflow
1.  **Library Installation**: Install `@dicebear/core` and the specific style collections needed (e.g., `@dicebear/collection`).
2.  **Local Generation**: Use the `createAvatar` function. This function takes a style and a seed, then returns an avatar object.
3.  **SVG Conversion**: Call `.toString()` on the resulting avatar object to get the raw SVG XML string.
4.  **Expo Rendering**: Use the `SvgXml` component from the `react-native-svg` library to render that string directly in the UI.

### Implementation Pattern (Conceptual)
```javascript
import { createAvatar } from '@dicebear/core';
import { lorelei } from '@dicebear/collection';
import { SvgXml } from 'react-native-svg';

const LocalAvatar = ({ seed }) => {
  // Generate SVG string locally based on seed
  const svgString = createAvatar(lorelei, {
    seed: seed,
    // options like colors, size, etc.
  }).toString();

  return <SvgXml xml={svgString} width="100" height="100" />;
};
```

**Key Benefits:**
*   **Zero Latency**: Avatars render instantly without waiting for an HTTP response.
*   **Offline Capability**: Works in airplane mode or poor network conditions.
*   **Privacy**: User seeds never leave the device.
*   **Deterministic**: The same seed always produces the same avatar locally.

---

## 2. Creating Custom Avatar Styles
To match our specific app styling and context, we can move beyond the provided collections and create **Custom Styles**.

### How Custom Styles Work
A DiceBear style is essentially a TypeScript object that implements the `AvatarStyle` interface. The core is the `create` method:

```typescript
const myCustomStyle = {
  create: ({ prng, options }) => {
    // 1. Define the SVG viewport
    const attributes = { viewBox: '0 0 100 100' };

    // 2. Use the PRNG (Pseudo-Random Number Generator) to pick attributes
    // This ensures the avatar is deterministic based on the seed
    const color = prng.pick(['#FF0000', '#00FF00', '#0000FF']);
    const shape = prng.bool() ? 'circle' : 'rect';

    // 3. Construct the SVG body
    const body = shape === 'circle' 
      ? `<circle cx="50" cy="50" r="50" fill="${color}" />`
      : `<rect x="0" y="0" width="100" height="100" fill="${color}" />`;

    return { attributes, body };
  },
};
```

### Strategy for Tailoring to E7 Apps
*   **Contextual Styling**: We can create different style objects for different app contexts (e.g., a "Professional/Minimal" style for productivity apps and a "Playful/Abstract" style for creative apps).
*   **Design System Integration**: We can hardcode our app's specific color palette into the `prng.pick()` arrays within the custom style, ensuring avatars always match the brand's theme.
*   **Layered Complexity**: We can build complex avatars by creating multiple "fragment" sets (eyes, mouths, hats) and using the `prng` to combine them.

### Tooling for Custom Styles
1.  **Figma Plugin**: DiceBear provides a Figma plugin that allows designers to visually assemble the fragments. This is the recommended path for the design phase.
2.  **TypeScript Implementation**: Once the design is finalized, the fragments are implemented as SVG strings within the `AvatarStyle` object.

---

## 3. Final Recommendations for Wxpi/E7 Apps

| Requirement | Solution |
| :--- | :--- |
| **No Network** | Use `@dicebear/core` $\rightarrow$ `toString()` $\rightarrow$ `SvgXml`. |
| **No URL Burden** | Store only the `seed` (string) in the database; generate the image on the fly. |
| **Custom Styling** | Implement a custom `AvatarStyle` object using our design system colors. |
| **Performance** | Wrap the avatar generation in `useMemo` to prevent re-generating the SVG on every render. |

## 4. Licensing & Commercial Use (Monetization)

Since E7 apps are intended to be monetizable, it is critical to distinguish between the **library license** and the **style license**.

### The Core Library (MIT License)
The `@dicebear/core` and other helper libraries are released under the **MIT License**. This is extremely permissive and allows for full commercial use, modification, and distribution within your proprietary apps.

### The Avatar Styles (Variable Licenses)
The actual visual designs (the "styles") are created by different artists and carry their own licenses. You must check the license for the specific style you choose:

1.  **CC0 1.0 (Public Domain)**: These are the safest for commercial apps. No attribution is required, and you can use them however you like (e.g., *Identicon, Notionists*).
2.  **CC BY 4.0 (Attribution)**: You can use these commercially, but you **must** provide attribution to the artist. For an E7 app, this would mean adding a line in your "About" or "Credits" screen (e.g., *"Avatars powered by [Artist Name] under CC BY 4.0"*).
3.  **Custom Commercial Licenses**: Some styles are explicitly labeled "Free for personal and commercial use" (e.g., *Bottts*). These are safe for your startup.

### Strategic Advice for E7
*   **Safe Bet**: Prioritize **CC0** styles or **Custom Styles** (created by E7) to avoid the administrative overhead of tracking attributions across multiple apps.
*   **Custom Styles = Full Ownership**: If you create your own `AvatarStyle` object using your own SVG fragments and design system, you own the intellectual property entirely, eliminating all licensing risk.
*   **Avoid the API**: Some community discussions suggest that using the *HTTP API* for high-volume commercial purposes may be frowned upon or restricted. By using the **JS Library locally**, you avoid this entirely and only deal with the asset licenses.

**Verdict**: DiceBear is perfectly aligned with our "local-first" and "design-pattern-driven" philosophy. We can achieve high-quality, branded avatars with zero external dependencies at runtime and full legal compliance by choosing the right styles or building our own.

