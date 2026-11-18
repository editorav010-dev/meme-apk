# Card Stack Swiping & Dynamic Aspect-Ratio Architecture

### SYSTEM ROLE & OBJECTIVE
You are a Principal Frontend Motion Engineer specializing in React, gesture physics, and mobile WebView architectures. Your objective is to implement, document, or extend the physics-driven, dynamic-aspect-ratio card stack swiping engine used in Meme Capsule APK (`com.meme.capsule`).

---

### 1. HIGH-LEVEL ARCHITECTURAL STACK
The swiping mechanism is built around a decoupled gesture-physics pipeline designed to eliminate React re-render lag during high-frequency touch interactions:
- **Physics Core**: `@react-spring/web` (`useSprings`, `to` multi-value interpolators). Animates entirely on the compositor thread without triggering React component re-renders during active drag frames.
- **Gesture Listener**: `@use-gesture/react` (`useDrag`). Captures raw pointer/touch vectors (`movement`, `velocity`, `direction`) with rubberband resistance and intentional propagation filtering.
- **Dynamic Dimension Engine**: Standalone dimension sniffing, caching, and viewport constraint math (`computeCanvasDimensions` in [dimensions.ts](file:///c:/Users/avspn/Desktop/projects/meme-apk/src/utils/dimensions.ts)).
- **Hybrid CSS/Spring Deck**: An evolving outer container with cubic-bezier morph transitions housing a 4-card 3D perspective stack.

---

### 2. THE 4-CARD 3D PERSPECTIVE DECK MECHANISM
The stack maintains exactly 4 rendered cards at any time to guarantee constant $O(1)$ memory consumption and 60/120 FPS performance:

#### Deck Coordinate System (`POS_CONFIG`):
- **Position 0 (Active Front)**: `scale: 1.0`, `x: 0`, `y: 0`, `z: 0`, `rotateZ: 0°`, `opacity: 1.0`, `zIndex: 40`, Neo-Brutalist shadow `#f4c300`.
- **Position 1 (Understudy)**: `scale: 0.95`, `y: 10px`, `z: -40px`, `opacity: 0.9`, `zIndex: 30`, Neo-Brutalist shadow `#9b30ff`.
- **Position 2 (Queue #3)**: `scale: 0.90`, `y: 18px`, `z: -80px`, `opacity: 0.75`, `zIndex: 20`, Neo-Brutalist shadow `#dd0061`.
- **Position 3 (Queue #4)**: `scale: 0.85`, `y: 24px`, `z: -120px`, `opacity: 0.5`, `zIndex: 10`, transparent shadow.

#### Gesture & Live Reactive Choreography:
1. **Interactive Element Bypassing**: When touch originates on nested action buttons (Share, Download, Favorite, Report), `useDrag` aborts immediately (`target.closest('button')`), ensuring UI buttons never trigger accidental swipes.
2. **Real-Time Pointer Tracking**: On active drag, card 0 translates directly with `[mx, my]`. Rotation is coupled to horizontal delta:
   $$\text{rotateZ} = \left(\frac{mx}{80}\right) \times 15^\circ$$
3. **Simultaneous Understudy Reveal**: As card 0 is dragged outward, card 1 does not sit idle. Its progress $\in [0, 1]$ is calculated from displacement:
   $$\text{progress} = \min\left(1, \frac{\max(|mx|, \max(0, -my))}{80}\right)$$
   Card 1 dynamically scales up ($0.95 \to 1.0$), shifts up ($y: 10\text{px} \to 0\text{px}$), and reaches full opacity ($0.9 \to 1.0$) in direct lockstep with user finger position.
4. **Dismissal & Threshold Physics**:
   - **Keep (Swipe Right)**: $mx > 80\text{px}$ or ($vx > 0.5$ and $mx > 25\text{px}$). Flings out to $x = +(window.innerWidth + 200\text{px})$, $\text{rotateZ} = +15^\circ$.
   - **Skip (Swipe Left)**: $mx < -80\text{px}$ or ($vx > 0.5$ and $mx < -25\text{px}$). Flings out to $x = -(window.innerWidth + 200\text{px})$, $\text{rotateZ} = -15^\circ$.
   - **Vault / Save (Swipe Up)**: $my < -60\text{px}$ or ($vy > 0.5$ and $my < -20\text{px}$). Flings upward to $y = -(window.innerHeight + 200\text{px})$, scaling down to $0.5$.
   - **Below Threshold**: Snaps back instantly with spring configuration `tension: 280, friction: 26, mass: 1.2`.
5. **Synchronous Queue Shift**: When card 0 executes its exit animation, cards 1, 2, and 3 simultaneously spring forward into slots 0, 1, and 2. After 260ms, the dismissed meme is popped from state, and the spring values are reset without visual flicker.

---

### 3. DYNAMIC ASPECT-RATIO EVOLUTION (HOW IT ACQUIRES ORIGINAL IMAGE RATIOS)
Memes come in every imaginable proportion: 9:16 vertical reels, 1:1 squares, 16:9 landscape screenshots, and odd crop factors. Traditional card swipers either letterbox with ugly black bars or crop critical punchlines. Meme Capsule solves this with a 4-part **Dynamic Canvas Morphing Architecture**:

#### Part A: Zero-Jank Dimension Sniffing & In-Memory LRU Cache
- Image natural pixel dimensions (`naturalWidth`, `naturalHeight`) are extracted directly from the native `HTMLImageElement` via `onLoad` or cached DOM instances.
- Cached in `memeDimensionCache = new Map<string, {width: number, height: number}>()`.
- **Eager Prefetching**: Cards at index 0 and index 1 are rendered with `loading="eager"`. By the time card 0 is swiped away, card 1's image has already downloaded into browser memory, its natural dimensions are already cached, and zero layout delay occurs.

#### Part B: Viewport-Constrained Optimal Canvas Math
The algorithm ([dimensions.ts: computeCanvasDimensions](file:///c:/Users/avspn/Desktop/projects/meme-apk/src/utils/dimensions.ts#L13-L59)) balances native aspect ratio against mobile screen real estate:
```typescript
const maxW = Math.min(viewportWidth * 0.88, 380);  // Max 88% screen width (capped at 380px)
const maxH = Math.min(viewportHeight * 0.52, 460); // Max 52% screen height (capped at 460px)
const minW = Math.min(230, maxW);
const minH = 190;

const ratio = naturalWidth / naturalHeight;

if (ratio >= maxW / maxH) {
  // Landscape / Wide Memes: Constrained by Max Width
  width = maxW;
  height = maxW / ratio;
  if (height < minH) {
    height = minH;
    width = Math.min(minH * ratio, viewportWidth * 0.92);
  }
} else {
  // Portrait / Tall Memes: Constrained by Max Height
  height = maxH;
  width = maxH * ratio;
  if (width < minW) {
    width = minW;
    height = Math.min(minW / ratio, maxH);
  }
}
```

#### Part C: Liquid Canvas Morphing via Smooth Bezier Interpolation
- Instead of resizing cards individually, the outer deck container ([MemeStack.tsx](file:///c:/Users/avspn/Desktop/projects/meme-apk/src/components/MemeStack.tsx#L360-L369)) binds its explicit pixel `width` and `height` to the active meme's computed dimensions.
- The deck applies a custom hardware-accelerated spring-like cubic bezier transition:
  ```css
  transition: width 0.22s cubic-bezier(0.2, 0.9, 0.3, 1), 
              height 0.22s cubic-bezier(0.2, 0.9, 0.3, 1);
  ```
- Because all 4 stacked cards inside use `width: 100%; height: 100%` and `object-contain`, **the entire physical card deck expands, contracts, and morphs its physical shape in 220ms** when transitioning from a landscape meme to a portrait meme.
- No punchlines are clipped, no black letterbox padding is needed, and the neo-brutalist border frame hugs the actual meme geometry seamlessly.

---

### 4. MOBILE-HARDENED EDGE-CASE RESILIENCE
1. **Background Resume & Sleep Recovery**:
   When the app is backgrounded (e.g. system share sheet or lock screen), mobile WebViews frequently terminate network sockets. A `visibilitychange` + `focus` listener detects app resumption, validates `img.naturalWidth > 0`, and re-triggers dimension calculation or retries image hydration automatically.
2. **Decryption Safety Watchdog**:
   If a remote meme CDN is sluggish, a 2.5-second fallback timer disarms the `DECRYPTING...` shimmer overlay, preventing the user interface from locking up even on partial network failures.

---

### 5. SOURCE CODE REFERENCES
- [src/components/MemeStack.tsx](file:///c:/Users/avspn/Desktop/projects/meme-apk/src/components/MemeStack.tsx) — Main gesture handling, `@use-gesture/react` drag bindings, understudy scale interpolation, and `@react-spring/web` 3D positioning.
- [src/components/MemeCard.tsx](file:///c:/Users/avspn/Desktop/projects/meme-apk/src/components/MemeCard.tsx) — Natural dimension sniffing, eager prefetching, resume recovery, and loading shimmers.
- [src/utils/dimensions.ts](file:///c:/Users/avspn/Desktop/projects/meme-apk/src/utils/dimensions.ts) — Mathematical bounding algorithm and aspect-ratio cache.
