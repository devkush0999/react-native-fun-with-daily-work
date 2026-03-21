## Q1. What is the difference between the old bridge and JSI?
**Difficulty:** Medium | **Frequency:** Very High

**Answer:** JSI (JavaScript Interface) replaces the old async bridge...

**Code:**
```js
// Example here
```

---
How to proceed — click any part card above and I'll generate the full 500-question markdown file for that section, ready to paste into GitHub. I suggest starting with Part 01 (RN Core) since it has the highest question frequency in interviews at this level.

Which part do you want me to generate first?

Generate the folder structure and README for Part 1: React Native Core for my GitHub interview prep repo

21:35

Check available skills
Check available skills
Now let me create the full folder structure and all files:


Created 3 files, ran a command
Created 3 files, ran a command
Now the big one — all 500 questions. I'll write them in sections:


Part 01 questions.md - Section 1: Architecture (Q1-Q60)
Part 01 questions.md - Section 1: Architecture (Q1-Q60)

Part 01 questions.md - Section 1: Architecture (Q1-Q60)
text

# Part 01 — React Native Core: 500 Questions

> **Format:** Q → Answer → Code (where applicable) → Follow-up
> **Difficulty:** 🟢 Easy · 🟡 Medium · 🔴 Hard

---

## Section 1: Architecture — Bridge, JSI & New Architecture (Q1–Q60)

---

### Q1. What is the React Native Bridge?

**Difficulty:** 🟢 Easy | **Frequency:** Very High | **Category:** Architecture

**Answer:**
The Bridge is the communication layer between the JavaScript thread and the Native (iOS/Android) thread in the old React Native architecture. JS and Native code cannot call each other directly, so the Bridge serializes data to JSON, sends it asynchronously across, and deserializes it on the other side. This causes a performance bottleneck because every interaction crosses this async boundary.

**Diagram (mental model):**
```
JS Thread  ──[JSON serialize]──▶  Bridge  ──[JSON deserialize]──▶  Native Thread
           ◀─[JSON serialize]──  Bridge  ◀─[JSON deserialize]──
```

**Follow-up:** Why is the Bridge considered a bottleneck? → Because serialization + async dispatch adds latency, especially for high-frequency events like scroll or gestures.

---

### Q2. What is JSI (JavaScript Interface)?

**Difficulty:** 🟡 Medium | **Frequency:** Very High | **Category:** Architecture

**Answer:**
JSI is the replacement for the Bridge in the New React Native Architecture. Instead of passing JSON across an async bridge, JSI allows JavaScript to hold a direct reference to C++ objects and call native methods **synchronously** — no serialization needed. This eliminates the bottleneck and enables features like synchronous native calls and shared memory.

**Key differences from Bridge:**

| Feature | Bridge | JSI |
|---------|--------|-----|
| Communication | Async JSON | Synchronous C++ |
| Serialization | Required | Not required |
| JS → Native calls | Batched, async | Direct, synchronous |
| Shared memory | No | Yes |

**Follow-up:** What is Fabric? → Fabric is the new rendering system built on top of JSI — it allows the UI layer to be driven synchronously from JS.

---

### Q3. What is the New React Native Architecture?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Architecture

**Answer:**
The New Architecture has three main pillars:

1. **JSI** — replaces the Bridge for JS-to-Native communication
2. **Fabric** — new concurrent rendering system (replaces the old UIManager)
3. **TurboModules** — lazy-loaded native modules (replaces NativeModules which were all loaded at startup)

Together, these remove the async bottleneck, reduce startup time (lazy loading), enable concurrent rendering, and allow true synchronous JS-Native calls.

**Code:**
```js
// Old architecture: NativeModules loaded eagerly at startup
import { NativeModules } from 'react-native';
const { MyModule } = NativeModules; // all modules loaded

// New architecture: TurboModules loaded lazily
import NativeMyModule from './NativeMyModule'; // loaded only when used
```

**Follow-up:** Is the New Architecture production-ready? → Yes, enabled by default from RN 0.74+.

---

### Q4. What are the two threads in React Native and what does each do?

**Difficulty:** 🟢 Easy | **Frequency:** Very High | **Category:** Architecture

**Answer:**
React Native runs on multiple threads:

1. **JS Thread** — runs your JavaScript code (business logic, state updates, React rendering decisions). Only one JS thread exists.
2. **UI Thread (Main Thread)** — the native iOS/Android main thread. Responsible for rendering views, handling gestures, and all native UI operations.
3. **Shadow Thread** — computes layout using Yoga (Facebook's Flexbox engine) before passing layout info to the UI thread.
4. **Native Modules Thread** — handles async native module calls (camera, file system, etc.)

**Why it matters:** If you run heavy computation on the JS thread, it blocks the UI — causing dropped frames. This is why offloading work (InteractionManager, setTimeout 0, workers) is important.

**Follow-up:** How do you move work off the JS thread? → Use `InteractionManager.runAfterInteractions`, `requestAnimationFrame`, or move logic to a native module.

---

### Q5. What is Hermes and why should you enable it?

**Difficulty:** 🟢 Easy | **Frequency:** Very High | **Category:** Architecture

**Answer:**
Hermes is a JavaScript engine built by Facebook specifically for React Native, optimized for mobile performance. Unlike V8 or JavaScriptCore, Hermes pre-compiles JS to bytecode at build time, so the app doesn't need to parse JS at runtime.

**Benefits:**
- Faster Time To Interactive (TTI) — no JS parsing on startup
- Lower memory usage
- Smaller APK/IPA size
- Improved garbage collection for mobile

**Code (enabling Hermes in android/app/build.gradle):**
```gradle
project.ext.react = [
    enableHermes: true  // add this line
]
```

**From RN 0.70+**, Hermes is enabled by default.

**Follow-up:** Can you use Chrome DevTools with Hermes? → Yes, Hermes supports the Chrome debugger protocol since RN 0.64.

---

### Q6. What is Metro Bundler?

**Difficulty:** 🟢 Easy | **Frequency:** Medium | **Category:** Architecture

**Answer:**
Metro is the JavaScript bundler used by React Native. It watches your source files, bundles them into a single JS file, and serves it to the app (in development) or produces a bundle file (for production). Metro handles:
- Module resolution
- Hot Module Replacement (HMR) in dev
- Source maps
- Asset loading (images, fonts)

**Code (custom Metro config):**
```js
// metro.config.js
const { getDefaultConfig } = require('metro-config');

module.exports = (async () => {
  const {
    resolver: { sourceExts, assetExts },
  } = await getDefaultConfig();

  return {
    transformer: {
      babelTransformerPath: require.resolve('react-native-svg-transformer'),
    },
    resolver: {
      assetExts: assetExts.filter(ext => ext !== 'svg'),
      sourceExts: [...sourceExts, 'svg'],
    },
  };
})();
```

**Follow-up:** What is the difference between Metro and Webpack? → Metro is optimised for React Native's specific needs (fast refresh, native asset loading). Webpack is for web.

---

### Q7. What is Fabric in the New Architecture?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Architecture

**Answer:**
Fabric is the new rendering system in the React Native New Architecture. Key improvements over the old renderer:

1. **Synchronous rendering** — can render synchronously on the UI thread via JSI, enabling true concurrent mode
2. **Shared ownership** — the shadow tree can be accessed from both JS and Native threads safely
3. **Concurrent features** — enables React 18's concurrent features (Suspense, transitions) in RN

In the old architecture, rendering was always async (JS → Bridge → Native). Fabric allows rendering decisions to happen without crossing a thread boundary.

**Follow-up:** What is the Shadow Tree? → A lightweight representation of the UI component tree computed in C++ (using Yoga for layout) before the actual native views are created.

---

### Q8. What are TurboModules?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Architecture

**Answer:**
TurboModules are the replacement for NativeModules in the New Architecture. Key improvements:

1. **Lazy loading** — modules load only when first called (old NativeModules loaded ALL modules at startup, increasing start time)
2. **Type safety** — defined via a typed spec (TypeScript/Flow codegen generates the bindings automatically)
3. **JSI-based** — synchronous communication via JSI, not the async Bridge
4. **No JSON serialization** — data passes as native types

**Code (TurboModule spec):**
```ts
// NativeMyModule.ts (spec file — codegen reads this)
import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  getDeviceName(): Promise<string>;
  multiply(a: number, b: number): number; // synchronous!
}

export default TurboModuleRegistry.getEnforcing<Spec>('MyModule');
```

**Follow-up:** What is Codegen? → Codegen reads your TypeScript/Flow specs and auto-generates C++ bindings at build time, removing manual boilerplate.

---

### Q9. What is the difference between `runAfterInteractions` and `setTimeout`?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Architecture

**Answer:**
Both defer work, but for different reasons:

- **`InteractionManager.runAfterInteractions`** — waits until all animations and interactions have completed. Best for deferring expensive work (heavy computation, data fetching) until the screen transition animation finishes.
- **`setTimeout(fn, 0)`** — yields to the event loop for one tick. Much shorter delay. Use for minor deferrals or breaking up sync work.

**Code:**
```js
import { InteractionManager } from 'react-native';

useEffect(() => {
  // This runs AFTER the screen transition animation completes
  const task = InteractionManager.runAfterInteractions(() => {
    fetchHeavyData();
  });

  return () => task.cancel(); // cleanup
}, []);

// vs setTimeout
useEffect(() => {
  const id = setTimeout(() => {
    doMinorWork();
  }, 0);
  return () => clearTimeout(id);
}, []);
```

**Follow-up:** Where would you use this in your ERP app? → After navigating to a screen with 200+ API calls, defer non-critical data fetching until the transition completes.

---

### Q10. What is the Yoga layout engine?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Architecture

**Answer:**
Yoga is an open-source cross-platform layout engine built by Facebook that implements the Flexbox specification. React Native uses Yoga to compute layout on the Shadow Thread before sending layout information to the native UI thread. It's written in C++ so it runs natively on both iOS and Android without a JS dependency.

**Why it matters:** All the Flexbox properties you write in React Native (`flex`, `flexDirection`, `alignItems`, etc.) are processed by Yoga — not by the browser's CSS engine.

**Follow-up:** What Flexbox properties are NOT supported in RN compared to CSS? → `display: grid`, `float`, `position: sticky`, percentage-based padding/margin relative to parent width (partial support).

---

### Q11. What happens when you call `setState` in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Architecture

**Answer:**
When `setState` (or a state updater from `useState`) is called:

1. React schedules a re-render (doesn't re-render immediately)
2. React batches multiple state updates (in React 18, all updates are batched by default)
3. The component function re-runs with the new state
4. React compares the new virtual DOM with the old one (reconciliation / diffing)
5. Only the changed elements are sent to the native side (via Bridge/JSI) for actual view updates

**Code:**
```js
// In React 18 — both updates are batched into ONE re-render
const handlePress = () => {
  setCount(c => c + 1);  // batched
  setName('Devesh');      // batched → only one re-render
};

// Before React 18 — inside async functions, each caused a separate re-render
setTimeout(() => {
  setCount(c => c + 1);  // re-render 1
  setName('Devesh');      // re-render 2 (two renders)
}, 0);
```

**Follow-up:** What is the reconciliation algorithm? → The Fiber reconciler — it breaks rendering into units of work that can be paused, resumed, and prioritised.

---

### Q12. What is the difference between `React.memo`, `useMemo`, and `useCallback`?

**Difficulty:** 🟡 Medium | **Frequency:** Very High | **Category:** Performance/Architecture

**Answer:**

| Hook/API | What it memoizes | When to use |
|----------|-----------------|-------------|
| `React.memo` | An entire component | When a child component re-renders with the same props |
| `useMemo` | A computed value | When a calculation is expensive and inputs rarely change |
| `useCallback` | A function reference | When passing callbacks to memoized children |

**Code:**
```js
// React.memo — memoize a component
const UserCard = React.memo(({ name, age }) => {
  return <Text>{name} - {age}</Text>;
});

// useMemo — memoize a value
const expensiveValue = useMemo(() => {
  return heavyComputation(data);
}, [data]); // recomputes only when data changes

// useCallback — memoize a function reference
const handlePress = useCallback(() => {
  doSomething(id);
}, [id]); // new function reference only when id changes

// All three together in a real scenario:
const ParentScreen = ({ users }) => {
  const sortedUsers = useMemo(() => [...users].sort(), [users]);
  const handleDelete = useCallback((id) => deleteUser(id), []);
  return <UserCard users={sortedUsers} onDelete={handleDelete} />;
};
```

**Follow-up:** Can overusing `useMemo` hurt performance? → Yes. Memoization itself has a cost (storing old values, comparing deps). Only use when the computation is genuinely expensive.

---

### Q13. What is the React Native component lifecycle?

**Difficulty:** 🟢 Easy | **Frequency:** High | **Category:** Components

**Answer:**

**Functional components (with hooks — the modern standard):**
```
Mount:   Component function runs → useLayoutEffect → useEffect (with [])
Update:  State/prop changes → re-render → useLayoutEffect cleanup+run → useEffect cleanup+run
Unmount: useEffect cleanup runs → useLayoutEffect cleanup runs
```

**Class components (legacy — know for interviews, but don't use):**
```
Mount:   constructor → render → componentDidMount
Update:  shouldComponentUpdate → render → componentDidUpdate
Unmount: componentWillUnmount
```

**Code (functional lifecycle with hooks):**
```js
const MyScreen = ({ userId }) => {
  // Runs after every render (mount + update)
  useEffect(() => {
    console.log('rendered');
    return () => console.log('cleanup');
  });

  // Runs once on mount
  useEffect(() => {
    fetchUser(userId);
  }, []);

  // Runs when userId changes
  useEffect(() => {
    refetchUser(userId);
  }, [userId]);

  // Runs synchronously after DOM mutations (before paint)
  useLayoutEffect(() => {
    measureLayout();
  }, []);
};
```

**Follow-up:** What is the difference between `useEffect` and `useLayoutEffect`? → `useLayoutEffect` runs synchronously after DOM mutations but before the screen is painted. Use for measurements. `useEffect` is async after paint.

---

### Q14. How does React Native handle gestures?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Architecture

**Answer:**
React Native has two approaches:

**Old approach — PanResponder (JS thread):**
Gesture events cross the Bridge from Native → JS → Back to Native. This adds latency, causing janky gestures.

**New approach — React Native Gesture Handler:**
Runs gesture recognition on the UI thread entirely, with no Bridge crossing. The result is 60fps smooth gestures.

**Code:**
```js
// React Native Gesture Handler (recommended)
import { GestureDetector, Gesture } from 'react-native-gesture-handler';
import Animated, { useSharedValue, useAnimatedStyle, withSpring } from 'react-native-reanimated';

const DraggableBox = () => {
  const offsetX = useSharedValue(0);
  const offsetY = useSharedValue(0);

  const panGesture = Gesture.Pan()
    .onUpdate((event) => {
      offsetX.value = event.translationX;
      offsetY.value = event.translationY;
    })
    .onEnd(() => {
      offsetX.value = withSpring(0);
      offsetY.value = withSpring(0);
    });

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [
      { translateX: offsetX.value },
      { translateY: offsetY.value },
    ],
  }));

  return (
    <GestureDetector gesture={panGesture}>
      <Animated.View style={[styles.box, animatedStyle]} />
    </GestureDetector>
  );
};
```

**Follow-up:** Why is Reanimated 2 better than the Animated API? → Reanimated 2 runs animations on the UI thread via JSI, not the JS thread — so even if JS is busy, animations stay smooth.

---

### Q15. What is Fast Refresh in React Native?

**Difficulty:** 🟢 Easy | **Frequency:** Medium | **Category:** Developer Experience

**Answer:**
Fast Refresh is the hot-reload mechanism in React Native (replaced Hot Reloading and Live Reloading). When you save a file, Fast Refresh:
1. Re-runs only the changed module
2. Re-renders only the affected components
3. **Preserves component state** (unless you edit a class component or state initializer)

It's smarter than a full reload because it patches the running app without restarting the JS runtime.

**When does it do a full reload?** → When a non-component module changes (e.g., a utility file) or when there's a runtime error.

**Follow-up:** How do you disable Fast Refresh? → In the in-app dev menu (shake device) → Disable Fast Refresh.

---

### Q16. What are the differences between `View`, `ScrollView`, and `FlatList`?

**Difficulty:** 🟢 Easy | **Frequency:** Very High | **Category:** Components

**Answer:**

| Component | Renders all children? | Scrollable? | Best for |
|-----------|----------------------|-------------|----------|
| `View` | Yes | No | Layout containers |
| `ScrollView` | Yes (all at once) | Yes | Small, fixed-size lists (<20 items) |
| `FlatList` | No (lazy renders) | Yes | Large, dynamic lists |
| `SectionList` | No (lazy renders) | Yes | Grouped lists with headers |

**The critical difference:** `ScrollView` renders ALL children at once (memory expensive). `FlatList` only renders items visible on screen plus a small buffer (`windowSize`). For large lists, always use `FlatList`.

**Code:**
```js
// DO NOT use ScrollView for large lists
<ScrollView>
  {thousandItems.map(item => <Item key={item.id} {...item} />)}
</ScrollView>

// USE FlatList — only renders visible items
<FlatList
  data={thousandItems}
  keyExtractor={item => item.id.toString()}
  renderItem={({ item }) => <Item {...item} />}
  initialNumToRender={10}
  windowSize={5}
/>
```

**Follow-up:** Can you use `ScrollView` inside `FlatList`? → Avoid it — nested scrollable components cause layout issues. Use `FlatList`'s `ListHeaderComponent` instead.

---

### Q17. What is the difference between `key` and `keyExtractor` in FlatList?

**Difficulty:** 🟢 Easy | **Frequency:** High | **Category:** Components

**Answer:**
- **`key` prop** — used in plain `Array.map()` renders to help React identify which items changed
- **`keyExtractor`** — the FlatList equivalent. A function that receives each item and returns a unique string key for that item

Both serve the same purpose: allow React's reconciler to track items for efficient re-renders. Missing or non-unique keys cause incorrect rendering and performance issues.

**Code:**
```js
// keyExtractor — always return a unique STRING
<FlatList
  data={users}
  keyExtractor={(item) => item.id.toString()} // must be string
  renderItem={({ item }) => <UserRow user={item} />}
/>

// Common mistake — using index as key (avoid for dynamic lists)
keyExtractor={(item, index) => index.toString()} // ❌ causes bugs on add/delete

// Correct
keyExtractor={(item) => item.userId.toString()} // ✅ stable, unique
```

**Follow-up:** What happens if you use array index as a key when items can be deleted? → React will associate the wrong component instance with the wrong item, causing state bugs.

---

### Q18. How do you pass data between screens in React Navigation?

**Difficulty:** 🟢 Easy | **Frequency:** Very High | **Category:** Navigation

**Answer:**
There are three main ways:

1. **Route params** — for simple data passed at navigation time
2. **Context / global state (Redux)** — for data shared across many screens
3. **Navigation state** — via `setParams` to update current screen params

**Code:**
```js
// Sending params
navigation.navigate('UserProfile', {
  userId: 42,
  userName: 'Devesh',
});

// Receiving params
const UserProfileScreen = ({ route }) => {
  const { userId, userName } = route.params;
  return <Text>{userName}</Text>;
};

// Updating params from within the screen
navigation.setParams({ userName: 'Updated Name' });

// Going back with data (use React Navigation's goBack or navigate back)
navigation.navigate('Home', { refreshed: true });
```

**Follow-up:** What is the issue with passing complex objects as params? → Params are serialized (for deep linking). Avoid large objects — pass IDs and fetch data on the target screen.

---

### Q19. What is the difference between Stack Navigator and Native Stack Navigator?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Navigation

**Answer:**

| Feature | Stack Navigator (`@react-navigation/stack`) | Native Stack (`@react-navigation/native-stack`) |
|---------|--------------------------------------------|-------------------------------------------------|
| Implementation | JS-based (uses Reanimated under the hood) | 100% native UINavigationController / Fragment |
| Performance | Slightly slower | 60fps native transitions |
| Customisation | Highly customisable | Less customisable |
| Gesture | JS gesture handler | Native swipe-back gesture |
| Recommended | Complex custom headers | Standard apps |

**Always prefer Native Stack** for standard navigation — it uses the platform's native navigation primitives and performs better.

**Code:**
```js
import { createNativeStackNavigator } from '@react-navigation/native-stack';
const Stack = createNativeStackNavigator();

<Stack.Navigator>
  <Stack.Screen name="Home" component={HomeScreen} />
  <Stack.Screen
    name="Detail"
    component={DetailScreen}
    options={{ animation: 'slide_from_right' }} // native animation
  />
</Stack.Navigator>
```

**Follow-up:** When would you choose the JS Stack over Native Stack? → When you need full control over header animations, custom transition animations (like a shared element transition).

---

### Q20. How does deep linking work in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Navigation

**Answer:**
Deep linking lets a URL (from a browser, notification, email, or another app) open a specific screen inside your app. There are two types:

1. **Custom URL schemes** — `myapp://profile/42` — only work if the app is installed
2. **Universal Links (iOS) / App Links (Android)** — `https://myapp.com/profile/42` — work even if the app isn't installed (falls back to website)

**Code:**
```js
// App.js — configure linking
const linking = {
  prefixes: ['myapp://', 'https://myapp.com'],
  config: {
    screens: {
      Home: 'home',
      Profile: 'profile/:userId',
      Settings: 'settings',
    },
  },
};

<NavigationContainer linking={linking}>
  {/* ... navigators */}
</NavigationContainer>

// Profile screen receives userId from the URL
const ProfileScreen = ({ route }) => {
  const { userId } = route.params; // from myapp://profile/42
};
```

**Follow-up:** How do you test deep links in development? → Android: `adb shell am start -W -a android.intent.action.VIEW -d "myapp://profile/42"`. iOS: `xcrun simctl openurl booted "myapp://profile/42"`.

---

### Q21. What is `useRef` and how is it different from `useState`?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Hooks

**Answer:**

| | `useState` | `useRef` |
|--|------------|---------|
| Triggers re-render on change | Yes | No |
| Persists across renders | Yes | Yes |
| Primary use | UI state | DOM/component references, mutable values |

`useRef` returns a mutable object `{ current: value }` that persists for the full lifetime of the component. Changing `ref.current` does NOT cause a re-render.

**Code:**
```js
// Common uses of useRef in React Native

// 1. Reference to a native component (e.g., TextInput)
const inputRef = useRef(null);
<TextInput ref={inputRef} />
<Button onPress={() => inputRef.current.focus()} title="Focus" />

// 2. Store a mutable value that shouldn't trigger re-renders
const intervalRef = useRef(null);
useEffect(() => {
  intervalRef.current = setInterval(() => tick(), 1000);
  return () => clearInterval(intervalRef.current);
}, []);

// 3. Track previous value
const prevCount = useRef(count);
useEffect(() => {
  prevCount.current = count;
}, [count]);
console.log('previous:', prevCount.current, 'current:', count);
```

**Follow-up:** How would you focus a TextInput after screen mount? → `useRef` + `useEffect` with `inputRef.current?.focus()` inside `InteractionManager.runAfterInteractions`.

---

### Q22. What is `useContext` and when should you use it vs Redux?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Hooks

**Answer:**
`useContext` reads the nearest value from a React Context. It's the built-in solution for sharing state without prop drilling.

**When to use Context:**
- Small/medium apps
- Simple global state (theme, locale, auth user)
- State that doesn't update frequently (context updates re-render ALL consumers)

**When to use Redux:**
- Large apps with complex state interactions
- High-frequency state updates (Redux uses selector memoization to prevent unnecessary re-renders)
- When you need middleware (logging, sagas, thunks)
- Dev tools requirement (Redux DevTools)

**Code:**
```js
// Creating a context
const AuthContext = createContext(null);

const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  return (
    <AuthContext.Provider value={{ user, setUser }}>
      {children}
    </AuthContext.Provider>
  );
};

// Consuming with useContext
const ProfileScreen = () => {
  const { user } = useContext(AuthContext);
  return <Text>Hello {user?.name}</Text>;
};
```

**Follow-up:** What is the performance problem with Context? → Every `Context.Provider` value change triggers a re-render for ALL components that called `useContext` with that context, even if the specific value they care about didn't change. Split contexts by concern to mitigate.

---

### Q23. What is the purpose of `useLayoutEffect`?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Hooks

**Answer:**
`useLayoutEffect` has the same signature as `useEffect` but fires **synchronously after all DOM mutations** and **before the browser paints**. Use it when you need to measure layout or mutate the DOM before the user sees the result (to avoid a visual flash).

**Code:**
```js
// Use case: measure a view's size and adjust layout
const MyComponent = () => {
  const viewRef = useRef(null);
  const [height, setHeight] = useState(0);

  useLayoutEffect(() => {
    // This runs before paint — no flash
    viewRef.current.measure((x, y, width, measuredHeight) => {
      setHeight(measuredHeight);
    });
  }, []);

  return (
    <View ref={viewRef} style={{ height }}>
      <Text>Content</Text>
    </View>
  );
};
```

**Rule of thumb:** Start with `useEffect`. Only switch to `useLayoutEffect` when you see a visual flicker.

**Follow-up:** Can `useLayoutEffect` cause performance problems? → Yes — it blocks painting. Keep it fast. Heavy computation inside `useLayoutEffect` delays the first render.

---

### Q24. What are custom hooks and when should you create one?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Hooks

**Answer:**
A custom hook is a JavaScript function whose name starts with `use` and can call other hooks. They allow you to extract and reuse stateful logic across multiple components without duplicating code.

**Create a custom hook when:**
- The same `useEffect + useState + useCallback` pattern appears in multiple components
- You want to encapsulate a specific concern (auth, form, API call, device sensors)
- The logic is complex enough to deserve its own file

**Code:**
```js
// Custom hook: useApi — reusable data fetching
const useApi = (endpoint) => {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    let cancelled = false;

    const fetch = async () => {
      try {
        setLoading(true);
        const result = await apiClient.get(endpoint);
        if (!cancelled) setData(result.data);
      } catch (err) {
        if (!cancelled) setError(err);
      } finally {
        if (!cancelled) setLoading(false);
      }
    };

    fetch();
    return () => { cancelled = true; }; // cleanup to prevent state updates on unmount
  }, [endpoint]);

  return { data, loading, error };
};

// Usage — clean, no duplication
const UsersScreen = () => {
  const { data, loading, error } = useApi('/users');
  if (loading) return <Spinner />;
  if (error) return <ErrorView />;
  return <UserList users={data} />;
};
```

**Follow-up:** What rules must custom hooks follow? → The Rules of Hooks: only call hooks at the top level (not inside conditions/loops), and only call hooks from React function components or other custom hooks.

---

### Q25. What is the `useReducer` hook and when should you prefer it over `useState`?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Hooks

**Answer:**
`useReducer` is an alternative to `useState` for managing complex state logic. It takes a reducer function (same pattern as Redux) and returns the current state + a dispatch function.

**Prefer `useReducer` when:**
- State has multiple sub-values that update together
- Next state depends on the previous state
- State transitions have complex logic (multiple `setState` calls that must happen atomically)
- You want to lift complex state logic out of the component

**Code:**
```js
const initialState = { count: 0, loading: false, error: null };

const reducer = (state, action) => {
  switch (action.type) {
    case 'INCREMENT':
      return { ...state, count: state.count + 1 };
    case 'FETCH_START':
      return { ...state, loading: true, error: null };
    case 'FETCH_SUCCESS':
      return { ...state, loading: false, data: action.payload };
    case 'FETCH_ERROR':
      return { ...state, loading: false, error: action.error };
    default:
      return state;
  }
};

const MyComponent = () => {
  const [state, dispatch] = useReducer(reducer, initialState);

  const fetchData = async () => {
    dispatch({ type: 'FETCH_START' });
    try {
      const data = await api.get('/data');
      dispatch({ type: 'FETCH_SUCCESS', payload: data });
    } catch (error) {
      dispatch({ type: 'FETCH_ERROR', error });
    }
  };

  return (
    <View>
      <Text>Count: {state.count}</Text>
      <Button onPress={() => dispatch({ type: 'INCREMENT' })} title="+" />
    </View>
  );
};
```

**Follow-up:** Is `useReducer` the same as Redux? → Same pattern (reducer + dispatch) but `useReducer` is local to a component (no global store, no middleware, no DevTools). Combine with Context to approximate Redux at small scale.

---

### Q26. What is the Animated API in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Animations

**Answer:**
The `Animated` API is React Native's built-in animation system. It creates animated values that can drive native view properties (opacity, transform, etc.) without triggering JS re-renders for every frame.

**Two modes:**
1. **JS-driven (default)** — animation values computed on JS thread, sent to native via Bridge each frame. Can drop frames if JS is busy.
2. **`useNativeDriver: true`** — moves animation computation to the UI thread entirely. Only works with transform and opacity (not width, height, backgroundColor).

**Code:**
```js
const fadeAnim = useRef(new Animated.Value(0)).current;

// Fade in
const fadeIn = () => {
  Animated.timing(fadeAnim, {
    toValue: 1,
    duration: 500,
    useNativeDriver: true, // always use when possible
  }).start();
};

// Sequence animation
Animated.sequence([
  Animated.timing(fadeAnim, { toValue: 1, duration: 300, useNativeDriver: true }),
  Animated.delay(1000),
  Animated.timing(fadeAnim, { toValue: 0, duration: 300, useNativeDriver: true }),
]).start();

<Animated.View style={{ opacity: fadeAnim }}>
  <Text>Hello</Text>
</Animated.View>
```

**Follow-up:** Why can't `useNativeDriver` work with width/height? → Native driver runs on the UI thread and can only manipulate properties that the UI thread has direct access to. Layout properties (width, height) are computed by the JS/Yoga layer.

---

### Q27. What is Reanimated 2 and how is it better than the Animated API?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Animations

**Answer:**
Reanimated 2 (react-native-reanimated) is a third-party animation library that runs **all animation logic on the UI thread via JSI** — not on the JS thread. This means:

- Animations never drop frames even if JS is doing heavy work
- Supports gesture-driven animations (with react-native-gesture-handler)
- Uses worklets — JS functions that run on the UI thread

**Core concepts:**
- `useSharedValue` — like `useRef` but lives on the UI thread
- `useAnimatedStyle` — a worklet that computes styles on the UI thread
- `withSpring`, `withTiming`, `withDecay` — animation functions
- Worklets — functions marked with `'worklet'` directive, compiled to run on UI thread

**Code:**
```js
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  withTiming,
} from 'react-native-reanimated';

const MyComponent = () => {
  const scale = useSharedValue(1);

  const animatedStyle = useAnimatedStyle(() => {
    // This runs on the UI thread — 'worklet' is added by Reanimated's Babel plugin
    return {
      transform: [{ scale: scale.value }],
    };
  });

  return (
    <Animated.View style={[styles.box, animatedStyle]}>
      <Pressable onPressIn={() => { scale.value = withSpring(0.9); }}
                 onPressOut={() => { scale.value = withSpring(1); }}>
        <Text>Press me</Text>
      </Pressable>
    </Animated.View>
  );
};
```

**Follow-up:** What is a worklet? → A function that Reanimated's Babel plugin transforms to run on the UI thread. Mark with `'worklet'` directive for functions called from `useAnimatedStyle` or gesture handlers.

---

### Q28. How do you optimise a FlatList for a large dataset?

**Difficulty:** 🔴 Hard | **Frequency:** Very High | **Category:** Performance

**Answer:**
This is the most common performance question. Multiple optimisations work together:

**Code:**
```js
<FlatList
  data={data}
  keyExtractor={(item) => item.id.toString()}
  renderItem={renderItem}

  // 1. Control initial render count (default: 10)
  initialNumToRender={8}

  // 2. Window size — renders items in (windowSize * screen height) range
  // windowSize={5} means 2.5 screens above + current + 2.5 below
  windowSize={5}

  // 3. Max items to render per batch (scroll batch)
  maxToRenderPerBatch={8}

  // 4. Remove off-screen items from memory (at cost of blank flash on fast scroll)
  removeClippedSubviews={true}

  // 5. Provide item height to avoid measurement (huge perf gain)
  getItemLayout={(data, index) => ({
    length: ITEM_HEIGHT,
    offset: ITEM_HEIGHT * index,
    index,
  })}

  // 6. Avoid anonymous functions for renderItem
  renderItem={renderItem} // defined outside, memoized
/>

// 7. Memoize the renderItem component
const renderItem = useCallback(({ item }) => (
  <MemoizedItem item={item} />
), []);

const MemoizedItem = React.memo(({ item }) => {
  return <View><Text>{item.title}</Text></View>;
});
```

**Summary of optimisations:**
1. `getItemLayout` — skip height measurement
2. `React.memo` on renderItem component
3. `useCallback` for renderItem function
4. `keyExtractor` with stable unique keys
5. `initialNumToRender`, `windowSize`, `maxToRenderPerBatch`
6. `removeClippedSubviews` on Android
7. `useNativeDriver: true` on any item animations

**Follow-up:** What is `getItemLayout` and when can't you use it? → It's a function that tells FlatList the exact height of each item without measuring it. You can't use it when items have dynamic/variable heights.

---

### Q29. What causes unnecessary re-renders in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** Very High | **Category:** Performance

**Answer:**
The most common causes:

1. **New object/array reference on every render** — `style={{ color: 'red' }}` creates a new object every render
2. **Inline arrow functions as props** — `onPress={() => handlePress(id)}` creates a new function every render
3. **Parent re-renders** — any parent re-render re-renders all children unless they are wrapped in `React.memo`
4. **Context updates** — any change in a Context value re-renders all consumers
5. **Missing `useCallback`/`useMemo`** on callbacks/values passed as props

**Code (fixing each):**
```js
// ❌ Bad — new object on every render
<View style={{ padding: 16 }} />

// ✅ Good — StyleSheet creates stable references
const styles = StyleSheet.create({ container: { padding: 16 } });
<View style={styles.container} />

// ❌ Bad — new function on every render → MemoizedChild always re-renders
<MemoizedChild onPress={() => handlePress(id)} />

// ✅ Good — stable function reference
const handlePress = useCallback((id) => doSomething(id), []);
<MemoizedChild onPress={handlePress} />

// Debugging re-renders with why-did-you-render library
import './wdyr'; // setup file that patches React
```

**Follow-up:** How do you find which component is causing unnecessary re-renders? → React DevTools Profiler (for Expo) or Flipper's React DevTools plugin for CLI projects.

---

### Q30. What is `StyleSheet.create` and why is it better than plain objects?

**Difficulty:** 🟢 Easy | **Frequency:** High | **Category:** Styling

**Answer:**
`StyleSheet.create` validates styles in development (catches typos like `paddingHorizontal` vs `paddingHorizantal`) and produces an integer ID for each style instead of the full object. On the native side, these IDs are resolved to native style objects only once, rather than on every render.

Benefits:
1. Style validation in dev mode
2. Stable object references (prevents unnecessary re-renders)
3. Slight performance improvement — style IDs sent over Bridge/JSI are lighter than full objects

**Code:**
```js
// ✅ Preferred
const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#fff',
    padding: 16,
  },
  title: {
    fontSize: 24,
    fontWeight: '600',
    color: '#000',
  },
});

// Dynamic styles — combine static + dynamic
<View style={[styles.container, isActive && styles.active, { marginTop: dynamicValue }]} />
```

**Follow-up:** Can you use `StyleSheet.create` with TypeScript? → Yes, the types are built in. `StyleSheet.create` is fully typed.

---

### Q31. What is the difference between `padding` and `margin` in React Native's Flexbox?

**Difficulty:** 🟢 Easy | **Frequency:** Medium | **Category:** Styling

**Answer:**
- **`padding`** — space INSIDE the component, between its border and its children
- **`margin`** — space OUTSIDE the component, between its border and surrounding siblings

Same as CSS, but RN does NOT support percentage-based margins relative to parent width (limited support added in later versions).

**Code:**
```js
// Useful shorthand props in React Native
paddingHorizontal  // = paddingLeft + paddingRight
paddingVertical    // = paddingTop + paddingBottom
marginHorizontal   // = marginLeft + marginRight
marginVertical     // = marginTop + marginBottom
```

**Follow-up:** How do you center a component in React Native? → `alignItems: 'center'` (cross-axis) + `justifyContent: 'center'` (main-axis) on the parent View, or `alignSelf: 'center'` on the child.

---

### Q32. What is `flex: 1` and what does it do?

**Difficulty:** 🟢 Easy | **Frequency:** High | **Category:** Styling

**Answer:**
`flex: 1` tells the component to grow and fill all available space in its parent. When multiple siblings have `flex` values, space is divided proportionally.

**Code:**
```js
// Parent has 300px height. Children split it by their flex values.
<View style={{ flex: 1 }}>          {/* takes full screen height */}
  <View style={{ flex: 2 }} />       {/* gets 200px (2/3 of 300) */}
  <View style={{ flex: 1 }} />       {/* gets 100px (1/3 of 300) */}
</View>

// flex: 0 + fixed dimensions = no growing
<View style={{ flex: 0, height: 50 }} />

// Common pattern: header + scrollable body + footer
<View style={{ flex: 1 }}>
  <Header />                          {/* height: 60 fixed */}
  <ScrollView style={{ flex: 1 }} />  {/* takes remaining space */}
  <Footer />                          {/* height: 50 fixed */}
</View>
```

**Follow-up:** What happens if you forget `flex: 1` on a root View? → The root view collapses to zero height (nothing renders visibly).

---

### Q33. How do you handle safe area on modern iPhones (notch, Dynamic Island)?

**Difficulty:** 🟢 Easy | **Frequency:** High | **Category:** Styling

**Answer:**
Use `react-native-safe-area-context` — it provides insets (top, bottom, left, right) for the safe area on any device.

**Code:**
```js
import { SafeAreaProvider, SafeAreaView, useSafeAreaInsets } from 'react-native-safe-area-context';

// Option 1: SafeAreaView wrapper (easiest)
const App = () => (
  <SafeAreaProvider>
    <SafeAreaView style={{ flex: 1 }}>
      <HomeScreen />
    </SafeAreaView>
  </SafeAreaProvider>
);

// Option 2: useSafeAreaInsets (more control)
const Header = () => {
  const insets = useSafeAreaInsets();
  return (
    <View style={{ paddingTop: insets.top, backgroundColor: '#000' }}>
      <Text>My App</Text>
    </View>
  );
};
```

**Follow-up:** Is `SafeAreaView` from core RN enough? → The core `SafeAreaView` only handles iOS. `react-native-safe-area-context` handles both iOS and Android (including android:windowLayoutInDisplayCutoutMode).

---

### Q34. What is the difference between `Platform.OS` and `Platform.select`?

**Difficulty:** 🟢 Easy | **Frequency:** Medium | **Category:** Platform APIs

**Answer:**

```js
import { Platform } from 'react-native';

// Platform.OS — returns 'ios', 'android', or 'web'
if (Platform.OS === 'ios') {
  // iOS-specific logic
}

// Platform.select — returns the value for the current platform
const styles = StyleSheet.create({
  container: {
    paddingTop: Platform.select({
      ios: 20,
      android: 10,
      default: 0, // web or other
    }),
  },
});

// Platform.Version — OS version
if (Platform.OS === 'android' && Platform.Version >= 33) {
  // Android 13+
}
```

**File-level platform splits (for large differences):**
```
Button.ios.js     // used on iOS
Button.android.js // used on Android
// import Button from './Button' — Metro picks the right file
```

---

### Q35. How do you handle keyboard avoiding in React Native forms?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Components

**Answer:**
When the keyboard appears, it can cover form inputs. The solutions:

**Code:**
```js
import { KeyboardAvoidingView, Platform, ScrollView } from 'react-native';

// Option 1: KeyboardAvoidingView (built-in)
<KeyboardAvoidingView
  style={{ flex: 1 }}
  behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
  keyboardVerticalOffset={Platform.OS === 'ios' ? 64 : 0}
>
  <ScrollView>
    <TextInput placeholder="Name" />
    <TextInput placeholder="Email" />
    <Button title="Submit" />
  </ScrollView>
</KeyboardAvoidingView>

// Option 2: react-native-keyboard-aware-scroll-view (more reliable)
import { KeyboardAwareScrollView } from 'react-native-keyboard-aware-scroll-view';

<KeyboardAwareScrollView extraScrollHeight={20}>
  <TextInput placeholder="Name" />
</KeyboardAwareScrollView>
```

**Follow-up:** Why does `KeyboardAvoidingView` sometimes not work perfectly on Android? → Android has its own keyboard-avoidance behavior (`windowSoftInputMode` in AndroidManifest). Set `android:windowSoftInputMode="adjustResize"` for most reliable behavior.

---

### Q36. What is the difference between `TouchableOpacity`, `TouchableHighlight`, `Pressable`, and `TouchableNativeFeedback`?

**Difficulty:** 🟢 Easy | **Frequency:** High | **Category:** Components

**Answer:**

| Component | Feedback style | Recommended |
|-----------|---------------|-------------|
| `TouchableOpacity` | Dims (reduces opacity) | ✅ Cross-platform standard |
| `TouchableHighlight` | Changes background color | Specific use cases |
| `TouchableNativeFeedback` | Native ripple effect (Android only) | Android-only |
| `Pressable` | Fully customisable via style function | ✅ Modern replacement for all |

**`Pressable` is the modern standard** — it provides pressed state directly in the style prop and is more capable than `TouchableOpacity`.

**Code:**
```js
// TouchableOpacity (classic)
<TouchableOpacity activeOpacity={0.7} onPress={handlePress}>
  <Text>Click me</Text>
</TouchableOpacity>

// Pressable (modern — preferred)
<Pressable
  onPress={handlePress}
  style={({ pressed }) => [
    styles.button,
    pressed && styles.buttonPressed, // style changes while pressed
  ]}
>
  {({ pressed }) => (
    <Text style={pressed ? styles.textPressed : styles.text}>
      {pressed ? 'Pressing...' : 'Click me'}
    </Text>
  )}
</Pressable>
```

**Follow-up:** What is `hitSlop`? → A prop that extends the touchable area beyond the visible bounds of the component. Useful for small buttons.

---

### Q37. What is the difference between `Image` and `FastImage`?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Components

**Answer:**

| Feature | `Image` (core RN) | `react-native-fast-image` |
|---------|------------------|--------------------------|
| Caching | Basic, inconsistent | Aggressive, disk + memory |
| Priority | Not supported | Supported (low, normal, high) |
| Performance | Average | High |
| Resizing | Limited | Controlled |
| GIF support | iOS only | Full |

`FastImage` wraps SDWebImage (iOS) and Glide (Android) — the industry-standard image loading libraries for each platform.

**Code:**
```js
import FastImage from 'react-native-fast-image';

<FastImage
  style={{ width: 100, height: 100 }}
  source={{
    uri: 'https://example.com/photo.jpg',
    headers: { Authorization: 'Bearer token' },
    priority: FastImage.priority.high,
    cache: FastImage.cacheControl.immutable,
  }}
  resizeMode={FastImage.resizeMode.contain}
/>

// Preload images before they're needed
FastImage.preload([
  { uri: 'https://example.com/photo1.jpg' },
  { uri: 'https://example.com/photo2.jpg' },
]);
```

**Follow-up:** How do you handle image loading errors? → `onError` prop on both `Image` and `FastImage`. Show a placeholder or fallback image.

---

### Q38. What is AsyncStorage and what are its limitations?

**Difficulty:** 🟢 Easy | **Frequency:** High | **Category:** Storage

**Answer:**
`AsyncStorage` is the key-value storage system for React Native (from `@react-native-async-storage/async-storage`). It stores strings asynchronously, persisted across app restarts.

**Limitations:**
- Stores only strings (must JSON.stringify/parse objects)
- Not encrypted (do NOT store sensitive data like tokens without encryption)
- No query capability — you can't search by value
- Max storage limit varies by platform (~6MB on Android)
- Slow for large amounts of data

**Code:**
```js
import AsyncStorage from '@react-native-async-storage/async-storage';

// Store
await AsyncStorage.setItem('user', JSON.stringify({ id: 1, name: 'Devesh' }));

// Retrieve
const raw = await AsyncStorage.getItem('user');
const user = raw ? JSON.parse(raw) : null;

// Remove
await AsyncStorage.removeItem('user');

// Multi-set (batch)
await AsyncStorage.multiSet([
  ['key1', 'value1'],
  ['key2', 'value2'],
]);
```

**For sensitive data:** Use `react-native-keychain` (stores in iOS Keychain / Android Keystore) or `react-native-encrypted-storage`.

**Follow-up:** Where did you use encrypted storage in your Debt Relief app? → JWT tokens and KYC data were stored using `react-native-encrypted-storage` to comply with security requirements.

---

### Q39. How do you implement push notifications in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Native APIs

**Answer:**
Two main libraries:

1. **`@notifee/react-native`** — local notifications + rich notification display
2. **`@react-native-firebase/messaging`** — Firebase Cloud Messaging (FCM) for push notifications from server

**Flow:**
1. User grants permission
2. App gets a device token from FCM
3. App sends token to your backend
4. Backend sends push via FCM
5. FCM delivers to device → app receives and displays

**Code:**
```js
import messaging from '@react-native-firebase/messaging';
import notifee from '@notifee/react-native';

// Request permission (iOS)
const requestPermission = async () => {
  const authStatus = await messaging().requestPermission();
  const enabled =
    authStatus === messaging.AuthorizationStatus.AUTHORIZED ||
    authStatus === messaging.AuthorizationStatus.PROVISIONAL;
  return enabled;
};

// Get FCM token
const getToken = async () => {
  const token = await messaging().getToken();
  await api.post('/devices', { token }); // send to backend
};

// Handle foreground notification
messaging().onMessage(async (remoteMessage) => {
  await notifee.displayNotification({
    title: remoteMessage.notification.title,
    body: remoteMessage.notification.body,
    android: {
      channelId: 'default',
      importance: AndroidImportance.HIGH,
    },
  });
});

// Handle background/quit notification tap
messaging().onNotificationOpenedApp((remoteMessage) => {
  navigation.navigate(remoteMessage.data.screen);
});
```

**Follow-up:** How do you handle silent push notifications? → Use `data-only` messages (no `notification` key) from FCM. These trigger `setBackgroundMessageHandler` without displaying a visible notification.

---

### Q40. What is the difference between development and production builds?

**Difficulty:** 🟢 Easy | **Frequency:** Medium | **Category:** Build System

**Answer:**

| Feature | Development Build | Production Build |
|---------|------------------|-----------------|
| JS bundle | Unminified, source maps | Minified, obfuscated |
| Dev menu | Available (shake) | Not available |
| Hermes | Interpreted | AOT bytecode compiled |
| Flipper | Connected | Not connected |
| Performance | Slower (dev checks) | Optimised |
| Error messages | Verbose, with stack traces | Minimal |
| Bundle loading | From Metro dev server | Embedded in app |

```js
// Conditional code based on environment
if (__DEV__) {
  console.log('Development mode');
  // Enable logging, dev tools, etc.
}
```

**Follow-up:** How do you create a release build on Android? → `cd android && ./gradlew assembleRelease` (APK) or `./gradlew bundleRelease` (AAB for Play Store).

---

### Q41. What is `useImperativeHandle` and when do you need it?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Hooks

**Answer:**
`useImperativeHandle` lets you control what is exposed on a `ref` when a parent component uses `ref` on your component. Used with `forwardRef`.

**Code:**
```js
// Custom TextInput that exposes focus() and clear()
const CustomInput = forwardRef((props, ref) => {
  const inputRef = useRef(null);

  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current?.focus(),
    clear: () => inputRef.current?.clear(),
    getValue: () => inputRef.current?.value,
  }));

  return <TextInput ref={inputRef} {...props} />;
});

// Parent
const FormScreen = () => {
  const inputRef = useRef(null);

  return (
    <View>
      <CustomInput ref={inputRef} />
      <Button onPress={() => inputRef.current.focus()} title="Focus" />
      <Button onPress={() => inputRef.current.clear()} title="Clear" />
    </View>
  );
};
```

**Follow-up:** When would you NOT use `useImperativeHandle`? → Most of the time. Prefer passing callbacks as props (declarative). Use `useImperativeHandle` only when you genuinely need imperative access (focus, scroll, play/pause).

---

### Q42. What is `React.lazy` and code splitting in React Native?

**Difficulty:** 🔴 Hard | **Frequency:** Low | **Category:** Performance

**Answer:**
`React.lazy` enables lazy loading of components — they load only when needed instead of in the initial bundle. In React Native, this is less common than web (Metro bundles everything), but useful for:
- Heavy screens loaded infrequently
- Conditional features (admin panel, debug screens)

**Code:**
```js
import React, { Suspense, lazy } from 'react';

// Lazy load a heavy screen
const HeavyReportScreen = lazy(() => import('./HeavyReportScreen'));

const App = () => (
  <Suspense fallback={<ActivityIndicator />}>
    <HeavyReportScreen />
  </Suspense>
);
```

**Note:** For true code splitting in RN, the community uses techniques like RAM bundles (for large apps) or dynamic requires.

**Follow-up:** What is a RAM bundle? → A React Native feature that splits the JS bundle into modules loaded on demand, reducing startup time for large apps.

---

### Q43. How do you handle orientation changes in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Device APIs

**Answer:**
```js
import { Dimensions, useWindowDimensions } from 'react-native';

// Option 1: useWindowDimensions hook (reactive — updates on orientation change)
const MyComponent = () => {
  const { width, height } = useWindowDimensions();
  const isLandscape = width > height;

  return (
    <View style={{ flexDirection: isLandscape ? 'row' : 'column' }}>
      <Panel />
      <Panel />
    </View>
  );
};

// Option 2: react-native-orientation-locker (lock orientation)
import Orientation from 'react-native-orientation-locker';

useEffect(() => {
  Orientation.lockToPortrait();
  return () => Orientation.unlockAllOrientations();
}, []);
```

**Follow-up:** When would you lock orientation? → For games (landscape) or video players (landscape). For standard enterprise/fintech apps, portrait lock is common.

---

### Q44. What is the `InteractionManager` API?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Performance

**Answer:**
`InteractionManager` allows deferring work until all animations and interactions (like screen transitions) have completed. This keeps the UI smooth during transitions.

**Code:**
```js
import { InteractionManager } from 'react-native';

const DashboardScreen = () => {
  const [data, setData] = useState(null);

  useEffect(() => {
    // Defer heavy data fetch until screen transition is done
    const task = InteractionManager.runAfterInteractions(async () => {
      const result = await fetchDashboardData();
      setData(result);
    });

    return () => task.cancel();
  }, []);

  if (!data) return <SkeletonLoader />;
  return <Dashboard data={data} />;
};
```

**Where you used it:** In your 100-screen ERP app, each screen deferred its initial API calls with `InteractionManager` to keep navigation transitions at 60fps.

---

### Q45. What is `Linking` API and how do you use it?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Native APIs

**Answer:**
The `Linking` API handles opening URLs, making calls, sending emails, and responding to deep links.

**Code:**
```js
import { Linking } from 'react-native';

// Open a URL
await Linking.openURL('https://google.com');

// Open maps
await Linking.openURL('maps:0,0?q=Connaught+Place+New+Delhi');

// Make a phone call
await Linking.openURL('tel:+918299206978');

// Send email
await Linking.openURL('mailto:devesh7664@gmail.com?subject=Hello');

// Handle incoming deep links (runtime)
useEffect(() => {
  const handleURL = ({ url }) => {
    const { path, queryParams } = Linking.parse(url);
    console.log(path, queryParams);
  };

  const sub = Linking.addEventListener('url', handleURL);
  return () => sub.remove();
}, []);

// Check if a URL can be opened
const canOpen = await Linking.canOpenURL('whatsapp://send?text=Hello');
```

---

### Q46. What is `useWindowDimensions` vs `Dimensions.get`?

**Difficulty:** 🟢 Easy | **Frequency:** Medium | **Category:** Device APIs

**Answer:**

| | `Dimensions.get` | `useWindowDimensions` |
|--|-----------------|----------------------|
| Type | Static call | React hook |
| Updates on orientation? | Only with event listener | Automatically |
| Usage | Outside components | Inside components |

```js
// Dimensions.get — use outside components or when hook isn't available
import { Dimensions } from 'react-native';
const { width, height } = Dimensions.get('window'); // 'window' or 'screen'

// Dimensions.get('window') — visible app area (excludes status bar on Android)
// Dimensions.get('screen') — full device screen

// useWindowDimensions — use inside components (reactive)
import { useWindowDimensions } from 'react-native';
const { width, height, fontScale, scale } = useWindowDimensions();
```

---

### Q47. How do you implement a splash screen in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Native APIs

**Answer:**
Use `react-native-splash-screen` or the newer `expo-splash-screen` (Expo).

**Flow:**
1. Configure native splash screen (shows immediately on app open, before JS loads)
2. JS loads in background
3. Once app is ready (fonts loaded, auth checked), hide the splash screen

**Code:**
```js
import SplashScreen from 'react-native-splash-screen';

const App = () => {
  useEffect(() => {
    const prepare = async () => {
      try {
        await loadFonts();
        await checkAuthStatus();
      } catch (e) {
        console.warn(e);
      } finally {
        SplashScreen.hide(); // hide when ready
      }
    };

    prepare();
  }, []);

  return <NavigationContainer />;
};
```

**Follow-up:** What is the difference between a native splash screen and a JS splash screen? → Native splash screen shows before JS loads (no white flash). JS splash screen is just a component — shown after JS loads, so there's a white flash during JS boot.

---

### Q48. What is `TextInput` `onChangeText` vs `onChange`?

**Difficulty:** 🟢 Easy | **Frequency:** Medium | **Category:** Components

**Answer:**

```js
// onChangeText — receives the text string directly (simpler, use this)
<TextInput onChangeText={(text) => setValue(text)} />

// onChange — receives the full native event object
<TextInput onChange={(event) => setValue(event.nativeEvent.text)} />
```

Always prefer `onChangeText` unless you need the full event (e.g. for cursor position, key pressed, etc.).

**Controlled vs uncontrolled TextInput:**
```js
// Controlled (recommended)
const [value, setValue] = useState('');
<TextInput value={value} onChangeText={setValue} />

// Uncontrolled (use ref to get value)
const inputRef = useRef(null);
<TextInput ref={inputRef} defaultValue="initial" />
// get value: inputRef.current.value — but this is unreliable in RN
```

---

### Q49. What is a Modal in React Native and what are its properties?

**Difficulty:** 🟢 Easy | **Frequency:** Medium | **Category:** Components

**Answer:**
```js
import { Modal } from 'react-native';

const MyModal = ({ visible, onClose }) => (
  <Modal
    visible={visible}
    transparent={true}          // if false, covers full screen with white
    animationType="slide"       // 'none' | 'slide' | 'fade'
    onRequestClose={onClose}    // required on Android (hardware back button)
    statusBarTranslucent={true} // content extends under status bar (Android)
  >
    <View style={styles.overlay}>
      <View style={styles.content}>
        <Text>Modal content</Text>
        <Button onPress={onClose} title="Close" />
      </View>
    </View>
  </Modal>
);
```

**Common issue:** Missing `onRequestClose` causes Android back button to not close the modal.

---

### Q50. How do you implement infinite scroll / pagination in FlatList?

**Difficulty:** 🟡 Medium | **Frequency:** Very High | **Category:** Components

**Answer:**
```js
const UserListScreen = () => {
  const [data, setData] = useState([]);
  const [page, setPage] = useState(1);
  const [loading, setLoading] = useState(false);
  const [hasMore, setHasMore] = useState(true);

  const fetchData = async (pageNum) => {
    if (loading || !hasMore) return;
    setLoading(true);
    try {
      const result = await api.get(`/users?page=${pageNum}&limit=20`);
      if (result.data.length < 20) setHasMore(false);
      setData(prev => [...prev, ...result.data]);
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => { fetchData(1); }, []);

  const handleEndReached = () => {
    if (!loading && hasMore) {
      const nextPage = page + 1;
      setPage(nextPage);
      fetchData(nextPage);
    }
  };

  return (
    <FlatList
      data={data}
      keyExtractor={(item) => item.id.toString()}
      renderItem={({ item }) => <UserRow user={item} />}
      onEndReached={handleEndReached}
      onEndReachedThreshold={0.5}  // trigger when 50% from bottom
      ListFooterComponent={loading ? <ActivityIndicator /> : null}
    />
  );
};
```

**Follow-up:** What is `onEndReachedThreshold`? → A value 0–1 that determines how far from the end to trigger `onEndReached`. `0.5` means trigger when 50% of remaining content is visible.

---

### Q51. What is the difference between `SectionList` and `FlatList`?

**Difficulty:** 🟢 Easy | **Frequency:** Medium | **Category:** Components

**Answer:**
`SectionList` renders data grouped into sections, each with its own header. `FlatList` renders a flat list without sections.

**Code:**
```js
const sections = [
  { title: 'January', data: ['Item 1', 'Item 2'] },
  { title: 'February', data: ['Item 3', 'Item 4'] },
];

<SectionList
  sections={sections}
  keyExtractor={(item, index) => item + index}
  renderItem={({ item }) => <Text>{item}</Text>}
  renderSectionHeader={({ section: { title } }) => (
    <Text style={styles.sectionHeader}>{title}</Text>
  )}
  stickySectionHeadersEnabled={true} // headers stick while scrolling
/>
```

**Use SectionList for:** Contacts lists, grouped transactions (by date), settings screens (grouped options).

---

### Q52. How do you handle refresh-to-reload in FlatList?

**Difficulty:** 🟢 Easy | **Frequency:** High | **Category:** Components

**Answer:**
```js
const [refreshing, setRefreshing] = useState(false);

const onRefresh = useCallback(async () => {
  setRefreshing(true);
  try {
    await fetchData();
  } finally {
    setRefreshing(false);
  }
}, []);

<FlatList
  data={data}
  renderItem={renderItem}
  refreshControl={
    <RefreshControl
      refreshing={refreshing}
      onRefresh={onRefresh}
      colors={['#FF6347']}     // Android spinner color
      tintColor="#FF6347"       // iOS spinner color
    />
  }
/>
```

---

### Q53. What is `VirtualizedList` and when do you use it directly?

**Difficulty:** 🔴 Hard | **Frequency:** Low | **Category:** Components

**Answer:**
`FlatList` and `SectionList` are both built on top of `VirtualizedList`. You'd use `VirtualizedList` directly when:
- Your data is not a simple array (e.g., immutable data structures, custom iterables)
- You need fine-grained control over the virtualisation window

It requires a `getItem(data, index)` and `getItemCount(data)` function to access items.

**Code:**
```js
<VirtualizedList
  data={myCustomDataStructure}
  getItemCount={(data) => data.size()}
  getItem={(data, index) => data.get(index)}
  keyExtractor={(item) => item.id}
  renderItem={({ item }) => <Row item={item} />}
/>
```

**In practice:** Almost always use `FlatList`. Only drop down to `VirtualizedList` for unusual data structures.

---

### Q54. How do you test React Native components with Jest?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Testing

**Answer:**
React Native uses Jest as the test runner. Use `@testing-library/react-native` for component tests (replaces Enzyme).

**Code:**
```js
// UserCard.test.tsx
import React from 'react';
import { render, fireEvent, screen } from '@testing-library/react-native';
import UserCard from '../UserCard';

describe('UserCard', () => {
  it('renders the user name', () => {
    render(<UserCard name="Devesh" age={24} />);
    expect(screen.getByText('Devesh')).toBeTruthy();
  });

  it('calls onPress when tapped', () => {
    const onPressMock = jest.fn();
    render(<UserCard name="Devesh" onPress={onPressMock} />);
    fireEvent.press(screen.getByText('Devesh'));
    expect(onPressMock).toHaveBeenCalledTimes(1);
  });

  it('shows loading state', () => {
    render(<UserCard name="Devesh" loading={true} />);
    expect(screen.getByTestId('loading-indicator')).toBeTruthy();
  });
});

// Snapshot test
it('matches snapshot', () => {
  const tree = render(<UserCard name="Devesh" />).toJSON();
  expect(tree).toMatchSnapshot();
});
```

**Follow-up:** What is the difference between snapshot tests and unit tests? → Snapshot tests catch unintended UI changes. Unit tests verify specific behaviour/logic. Prefer unit tests for business logic.

---

### Q55. What is error boundary in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Error Handling

**Answer:**
Error boundaries catch JavaScript errors in the component tree and render a fallback UI instead of crashing the app. They must be class components (no hooks equivalent yet, but third-party packages like `react-error-boundary` provide a hook wrapper).

**Code:**
```js
// Class-based error boundary
class ErrorBoundary extends React.Component {
  state = { hasError: false, error: null };

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, errorInfo) {
    // Log to crash reporting service
    crashlytics().recordError(error);
    console.log('Error caught:', error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return (
        <View style={styles.errorContainer}>
          <Text>Something went wrong.</Text>
          <Button onPress={() => this.setState({ hasError: false })} title="Try again" />
        </View>
      );
    }
    return this.props.children;
  }
}

// Usage
<ErrorBoundary>
  <CrashProneComponent />
</ErrorBoundary>

// With react-error-boundary (hook-friendly)
import { ErrorBoundary } from 'react-error-boundary';

<ErrorBoundary
  fallbackRender={({ error, resetErrorBoundary }) => (
    <View>
      <Text>{error.message}</Text>
      <Button onPress={resetErrorBoundary} title="Retry" />
    </View>
  )}
  onError={(error) => crashlytics().recordError(error)}
>
  <MyScreen />
</ErrorBoundary>
```

---

### Q56. What is Flipper and how do you use it for debugging?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Developer Tools

**Answer:**
Flipper is the desktop debugging tool for React Native. It provides:
- **React DevTools plugin** — inspect component tree, props, state
- **Network plugin** — inspect all HTTP requests/responses
- **Layout Inspector** — inspect native view hierarchy
- **Logs** — device log viewer
- **Databases** — inspect AsyncStorage, SQLite
- **Hermes Debugger** — JS debugging with breakpoints

**Common use cases in your work:**
- Debugging API calls in the ERP app (Network tab)
- Inspecting component re-renders (React DevTools)
- Checking AsyncStorage values during development

**Follow-up:** What replaced Flipper for newer RN versions? → From RN 0.73+, Flipper is no longer bundled by default. The recommended alternative is the built-in Chrome DevTools via `react-native-debugger`.

---

### Q57. What is the difference between `console.log` and `console.warn` and `console.error` in React Native?

**Difficulty:** 🟢 Easy | **Frequency:** Low | **Category:** Developer Tools

**Answer:**

| Method | Display | Use for |
|--------|---------|---------|
| `console.log` | White text in Metro | General debugging |
| `console.warn` | Yellow warning box in app | Non-critical issues |
| `console.error` | Red error box in app | Errors (don't use in prod) |
| `console.info` | Same as log | Informational |

**Remove all console logs in production:**
```js
// babel.config.js
module.exports = {
  presets: ['module:metro-react-native-babel-preset'],
  plugins: ['transform-remove-console'], // removes all console.* in prod
};
```

---

### Q58. How do you measure and profile React Native app performance?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Performance

**Answer:**

**Tools:**
1. **Flipper Performance plugin** — frame rate, JS thread usage
2. **React DevTools Profiler** — which components take longest to render
3. **Systrace** — Android system-level profiling (tracing JS + UI + render thread)
4. **Perf Monitor** — in-app FPS and RAM counter (shake menu → Show Perf Monitor)
5. **why-did-you-render** — logs unnecessary re-renders

**Key metrics to monitor:**
- **FPS** — should be 60fps (or 120fps on high refresh rate devices). Drops = jank.
- **JS thread frame time** — each frame must complete in 16ms. Over 16ms = dropped frame.
- **RAM** — growing RAM with no release = memory leak
- **TTI (Time To Interactive)** — time from app open to first user interaction

**Code (profiling a slow render):**
```js
// Mark a performance event
import { performance } from 'react-native';

const start = performance.now();
heavyOperation();
const duration = performance.now() - start;
console.log(`Heavy op took ${duration}ms`);
```

---

### Q59. What is `NativeEventEmitter` and when do you use it?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Native Modules

**Answer:**
`NativeEventEmitter` allows JavaScript to listen to events emitted from native modules. When a native module needs to send unsolicited data to JS (e.g., sensor readings, Bluetooth events, payment callbacks), it emits an event that JS listens to.

**Code:**
```js
import { NativeEventEmitter, NativeModules } from 'react-native';

const { BluetoothModule } = NativeModules;
const bluetoothEmitter = new NativeEventEmitter(BluetoothModule);

useEffect(() => {
  const subscription = bluetoothEmitter.addListener(
    'bluetoothDeviceFound',
    (device) => {
      console.log('Found device:', device.name);
      setDevices(prev => [...prev, device]);
    }
  );

  BluetoothModule.startScan();

  return () => {
    subscription.remove(); // always clean up!
    BluetoothModule.stopScan();
  };
}, []);
```

**Follow-up:** How does this differ from TurboModules? → TurboModules use JSI for synchronous calls. Events from native to JS still use an event emitter pattern even with TurboModules — the mechanism is similar but more efficient (no JSON serialization).

---

### Q60. What is the difference between `require` and `import` in React Native?

**Difficulty:** 🟢 Easy | **Frequency:** Low | **Category:** JavaScript/Build

**Answer:**

| | `import` (ES Modules) | `require` (CommonJS) |
|--|----------------------|---------------------|
| Syntax | `import X from 'y'` | `const X = require('y')` |
| Loading | Static (resolved at build) | Dynamic (can be inside functions) |
| Tree shaking | Supported | Limited |
| Standard | ES6+ | Node.js legacy |

**In React Native:** Both work (Metro understands both). Prefer `import` for components and modules. Use `require` for:
- Dynamic image paths: `require('./assets/logo.png')`
- Conditional imports

```js
// Static import (preferred for modules)
import React from 'react';
import { View } from 'react-native';

// Dynamic require (for images — Metro needs to know all images at build time)
const logo = require('./assets/logo.png'); // ✅
const logo = require(`./assets/${name}.png`); // ❌ doesn't work — Metro can't resolve dynamic paths

// Dynamic import (for code splitting — limited RN support)
const LazyModule = await import('./HeavyModule');
```

---

## Section 2: Hooks Deep Dive (Q61–Q120)

---

### Q61. What are the Rules of Hooks?

**Difficulty:** 🟢 Easy | **Frequency:** High | **Category:** Hooks

**Answer:**
React enforces two rules for hooks:

1. **Only call hooks at the top level** — never inside loops, conditions, or nested functions. This ensures hooks are called in the same order on every render (React relies on call order to associate state with hooks).

2. **Only call hooks from React functions** — from React function components or custom hooks. Never from regular JavaScript functions, class components, or event handlers directly.

**Code:**
```js
// ❌ WRONG — hook inside a condition
const MyComponent = ({ isLoggedIn }) => {
  if (isLoggedIn) {
    const [user, setUser] = useState(null); // BREAKS RULES
  }
};

// ✅ CORRECT — always call hooks unconditionally
const MyComponent = ({ isLoggedIn }) => {
  const [user, setUser] = useState(null); // always called
  // then conditionally USE the state
  if (!isLoggedIn) return <LoginScreen />;
  return <UserProfile user={user} />;
};

// ❌ WRONG — hook inside a loop
for (let i = 0; i < 3; i++) {
  const [val, setVal] = useState(i); // breaks rules — call count varies
}
```

**ESLint plugin:** `eslint-plugin-react-hooks` enforces these rules automatically.

---

### Q62. What is the difference between `useState` and `useRef` for tracking values?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Hooks

**Answer:**
Both persist values across renders, but:
- `useState` — triggers re-render when changed. For values that should update the UI.
- `useRef` — does NOT trigger re-render. For values that shouldn't affect the UI (timers, flags, previous values, DOM refs).

```js
// Use useState — want UI to update when count changes
const [count, setCount] = useState(0);

// Use useRef — isMounted flag, doesn't need to show in UI
const isMounted = useRef(true);
useEffect(() => {
  return () => { isMounted.current = false; };
}, []);

// Anti-pattern: using useState for values that never appear in JSX
// Every setState causes a re-render — wasteful if value isn't rendered
const [timer, setTimer] = useState(null); // ❌ timer ID doesn't need to render
const timerRef = useRef(null); // ✅ use ref for timer IDs
```

---

### Q63. How does the dependency array in `useEffect` work?

**Difficulty:** 🟡 Medium | **Frequency:** Very High | **Category:** Hooks

**Answer:**
The dependency array controls when `useEffect` re-runs:

```js
// No array — runs after EVERY render
useEffect(() => {
  console.log('every render');
});

// Empty array [] — runs ONCE after first render (componentDidMount)
useEffect(() => {
  fetchInitialData();
}, []);

// With dependencies — runs when any dep changes
useEffect(() => {
  fetchUserData(userId);
}, [userId]);

// Multiple deps — runs when ANY of them changes
useEffect(() => {
  updateDisplay(width, theme);
}, [width, theme]);
```

**Common mistake — stale closure:**
```js
// ❌ Bug — count is captured from the render when effect first ran
useEffect(() => {
  const id = setInterval(() => {
    console.log(count); // always prints initial count (stale closure)
  }, 1000);
  return () => clearInterval(id);
}, []); // missing count in deps

// ✅ Fix 1 — add count to deps (re-creates interval on each count change)
}, [count]);

// ✅ Fix 2 — use ref
const countRef = useRef(count);
countRef.current = count;
useEffect(() => {
  const id = setInterval(() => {
    console.log(countRef.current); // always fresh
  }, 1000);
  return () => clearInterval(id);
}, []);
```

---

### Q64. What is a stale closure and how do you fix it?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Hooks/JavaScript

**Answer:**
A stale closure is when a function captured variables from an older render and holds onto those old values, even after the state has updated.

```js
// Classic stale closure bug
const Counter = () => {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1); // ❌ always uses count=0 (from first render)
    }, 1000);
    return () => clearInterval(id);
  }, []); // deps are empty, so effect only runs once and captures count=0

  return <Text>{count}</Text>;
};

// Fix 1: Functional update form — doesn't need current count
useEffect(() => {
  const id = setInterval(() => {
    setCount(prev => prev + 1); // ✅ always uses latest count
  }, 1000);
  return () => clearInterval(id);
}, []);

// Fix 2: useRef
const countRef = useRef(count);
countRef.current = count;
```

---

### Q65. What is `useCallback` and what problem does it solve?

**Difficulty:** 🟡 Medium | **Frequency:** Very High | **Category:** Hooks

**Answer:**
`useCallback` memoizes a function — it returns the same function reference between renders (unless its deps change). This solves the problem of `React.memo` children re-rendering because their callback prop is a new function on every parent render.

```js
const ParentScreen = ({ userId }) => {
  // ❌ New function every render → MemoizedChild always re-renders
  const handlePress = () => navigateToProfile(userId);

  // ✅ Stable function reference → MemoizedChild only re-renders when userId changes
  const handlePress = useCallback(() => {
    navigateToProfile(userId);
  }, [userId]);

  return <MemoizedChild onPress={handlePress} />;
};

const MemoizedChild = React.memo(({ onPress }) => {
  console.log('Child rendered'); // only when onPress reference changes
  return <Button onPress={onPress} title="Go to profile" />;
});
```

**When NOT to use `useCallback`:** When the function is only used inside the component (not passed as a prop) — memoizing it offers no benefit and adds overhead.

---

### Q66. What is the `useMemo` hook?

**Difficulty:** 🟡 Medium | **Frequency:** Very High | **Category:** Hooks

**Answer:**
`useMemo` memoizes the result of a computation. The function runs only when its dependencies change.

```js
const ProductListScreen = ({ products, searchQuery }) => {
  // ❌ Filters on every render
  const filtered = products.filter(p => p.name.includes(searchQuery));

  // ✅ Only filters when products or searchQuery changes
  const filtered = useMemo(() => {
    return products.filter(p => p.name.toLowerCase().includes(searchQuery.toLowerCase()));
  }, [products, searchQuery]);

  return <FlatList data={filtered} renderItem={renderItem} />;
};
```

**Real cost example — when useMemo is worth it:**
```js
// Expensive: sorting 10,000 items
const sortedData = useMemo(() => {
  return [...data].sort((a, b) => a.name.localeCompare(b.name));
}, [data]);

// Not worth it: simple arithmetic
const total = useMemo(() => a + b, [a, b]); // overkill — just write: const total = a + b;
```

---

### Q67. How do you fetch data on component mount?

**Difficulty:** 🟢 Easy | **Frequency:** Very High | **Category:** Hooks

**Answer:**
```js
const UserScreen = ({ userId }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    let cancelled = false; // prevent state update on unmount

    const fetchUser = async () => {
      try {
        const data = await api.get(`/users/${userId}`);
        if (!cancelled) {
          setUser(data);
          setLoading(false);
        }
      } catch (err) {
        if (!cancelled) {
          setError(err.message);
          setLoading(false);
        }
      }
    };

    fetchUser();

    return () => {
      cancelled = true; // cleanup on unmount or userId change
    };
  }, [userId]);

  if (loading) return <ActivityIndicator />;
  if (error) return <ErrorView message={error} />;
  return <UserProfile user={user} />;
};
```

**Follow-up:** What happens without the cleanup function? → If the component unmounts before the fetch completes, `setUser` is called on an unmounted component — React shows a warning and the state update is lost.

---

### Q68. How do you debounce a search input?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Hooks

**Answer:**
Debouncing delays the API call until the user has stopped typing for a specified duration.

```js
const SearchScreen = () => {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  // Method 1: useEffect with setTimeout
  useEffect(() => {
    if (!query.trim()) return;

    const timeoutId = setTimeout(async () => {
      const data = await api.search(query);
      setResults(data);
    }, 500); // wait 500ms after typing stops

    return () => clearTimeout(timeoutId); // cancel on next keystroke
  }, [query]);

  // Method 2: custom useDebounce hook
  const debouncedQuery = useDebounce(query, 500);
  useEffect(() => {
    if (debouncedQuery) fetchResults(debouncedQuery);
  }, [debouncedQuery]);

  return (
    <View>
      <TextInput
        value={query}
        onChangeText={setQuery}
        placeholder="Search..."
      />
      <FlatList data={results} renderItem={renderItem} />
    </View>
  );
};

// Custom useDebounce hook
const useDebounce = (value, delay) => {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id);
  }, [value, delay]);

  return debounced;
};
```

---

### Q69. How do you handle component cleanup in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Hooks

**Answer:**
Cleanup prevents memory leaks when components unmount. Return a cleanup function from `useEffect`.

```js
const LiveScreen = () => {
  useEffect(() => {
    // Setup
    const subscription = eventEmitter.addListener('update', handleUpdate);
    const interval = setInterval(pollData, 5000);
    const wsConnection = openWebSocket();

    // Cleanup — runs on unmount or before next effect run
    return () => {
      subscription.remove();
      clearInterval(interval);
      wsConnection.close();
    };
  }, []);

  // Cleanup for async operations (prevent state update after unmount)
  useEffect(() => {
    let active = true;

    fetchData().then(data => {
      if (active) setData(data);
    });

    return () => { active = false; };
  }, []);
};
```

---

### Q70. What is `React.memo` and how does it differ from `PureComponent`?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Performance

**Answer:**

| | `React.memo` | `PureComponent` |
|--|-------------|----------------|
| For | Functional components | Class components |
| Comparison | Shallow prop comparison | Shallow prop + state comparison |
| Custom comparison | Yes (`areEqual` function) | No (manual `shouldComponentUpdate`) |

```js
// React.memo — default shallow comparison
const UserCard = React.memo(({ name, age }) => {
  return <Text>{name} ({age})</Text>;
});

// React.memo with custom comparison
const UserCard = React.memo(
  ({ user }) => <Text>{user.name}</Text>,
  (prevProps, nextProps) => {
    // Return true if should NOT re-render (same as shouldComponentUpdate returning false)
    return prevProps.user.id === nextProps.user.id;
  }
);

// PureComponent (class — legacy)
class UserCard extends React.PureComponent {
  render() {
    return <Text>{this.props.name}</Text>;
  }
}
```

---

### Q71. How do you share state between sibling components in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** State

**Answer:**
Three approaches depending on scale:

```js
// 1. Lift state up to common parent
const Parent = () => {
  const [shared, setShared] = useState('');
  return (
    <View>
      <SiblingA value={shared} onChange={setShared} />
      <SiblingB value={shared} />
    </View>
  );
};

// 2. Context API (for deeper sharing without prop drilling)
const SharedContext = createContext(null);
const Provider = ({ children }) => {
  const [value, setValue] = useState('');
  return (
    <SharedContext.Provider value={{ value, setValue }}>
      {children}
    </SharedContext.Provider>
  );
};
const SiblingA = () => {
  const { setValue } = useContext(SharedContext);
  return <Button onPress={() => setValue('hello')} title="Set" />;
};
const SiblingB = () => {
  const { value } = useContext(SharedContext);
  return <Text>{value}</Text>;
};

// 3. Redux (for app-wide state)
```

---

### Q72. What is `forwardRef` in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Hooks

**Answer:**
`forwardRef` allows a parent to pass a `ref` through to a child component's internal element. By default, refs cannot be attached to function components.

```js
// Without forwardRef — parent can't ref CustomInput's TextInput
// With forwardRef — parent gets direct ref to the TextInput
const CustomInput = forwardRef((props, ref) => {
  return (
    <View style={styles.wrapper}>
      <TextInput ref={ref} {...props} style={styles.input} />
    </View>
  );
});

// Parent usage
const Form = () => {
  const nameRef = useRef(null);
  const emailRef = useRef(null);

  const focusEmail = () => emailRef.current?.focus();

  return (
    <View>
      <CustomInput ref={nameRef} placeholder="Name" returnKeyType="next" onSubmitEditing={focusEmail} />
      <CustomInput ref={emailRef} placeholder="Email" returnKeyType="done" />
    </View>
  );
};
```

---

### Q73. What is the `useId` hook?

**Difficulty:** 🟡 Medium | **Frequency:** Low | **Category:** Hooks

**Answer:**
`useId` generates a stable, unique ID that is consistent between server and client renders (relevant for React Native with SSR setups or web). Useful for accessibility (linking labels to inputs).

```js
const LabeledInput = ({ label }) => {
  const id = useId();
  return (
    <View>
      <Text nativeID={id}>{label}</Text>
      <TextInput accessibilityLabelledBy={id} />
    </View>
  );
};
```

---

### Q74. What are controlled and uncontrolled components?

**Difficulty:** 🟢 Easy | **Frequency:** High | **Category:** Components

**Answer:**
- **Controlled** — React state is the single source of truth. Every change goes through state.
- **Uncontrolled** — the component maintains its own internal state. You read it via a ref.

```js
// Controlled TextInput
const [value, setValue] = useState('');
<TextInput value={value} onChangeText={setValue} />

// Uncontrolled — defaultValue + ref
const inputRef = useRef(null);
<TextInput defaultValue="initial" ref={inputRef} />
// Get current value: inputRef.current?.getValue() — but RN doesn't expose this well
// Controlled components are strongly preferred in React Native
```

**In React Native, always use controlled components** — uncontrolled inputs in RN are unreliable compared to web.

---

### Q75. What is React's reconciliation and the diffing algorithm?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Architecture

**Answer:**
Reconciliation is React's process of comparing the new virtual DOM with the previous one to determine the minimum number of native view changes needed.

**The diffing algorithm (O(n) not O(n³)):**
1. Elements of different types → tear down the old tree, build new one from scratch
2. Same type DOM elements → update only changed attributes
3. Same type component elements → update props, recurse into children
4. Lists → use `key` to match old and new items (stable keys = minimal moves)

**React Fiber** — the reconciler implementation that breaks work into units (fibers) that can be paused and resumed, enabling prioritised and concurrent rendering.

```js
// Why keys matter for reconciliation
// Without keys, React re-renders all items on any change
// With stable keys, React only re-renders changed items:
<FlatList
  data={items}
  keyExtractor={item => item.id.toString()} // stable key = efficient diffing
  renderItem={({ item }) => <Row item={item} />}
/>
```

---

### Q76. How do you implement a theme (dark/light mode) in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Styling

**Answer:**
```js
// 1. Use Appearance API to detect system theme
import { useColorScheme } from 'react-native';

const MyComponent = () => {
  const colorScheme = useColorScheme(); // 'light' | 'dark' | null
  const isDark = colorScheme === 'dark';

  return (
    <View style={{ backgroundColor: isDark ? '#000' : '#fff' }}>
      <Text style={{ color: isDark ? '#fff' : '#000' }}>Hello</Text>
    </View>
  );
};

// 2. Theme Context (recommended for app-wide theming)
const themes = {
  light: { bg: '#fff', text: '#000', primary: '#6200EE' },
  dark:  { bg: '#121212', text: '#fff', primary: '#BB86FC' },
};

const ThemeContext = createContext(themes.light);

const ThemeProvider = ({ children }) => {
  const scheme = useColorScheme();
  const theme = scheme === 'dark' ? themes.dark : themes.light;
  return (
    <ThemeContext.Provider value={theme}>
      {children}
    </ThemeContext.Provider>
  );
};

const useTheme = () => useContext(ThemeContext);

// Usage
const Button = () => {
  const theme = useTheme();
  return (
    <Pressable style={{ backgroundColor: theme.primary }}>
      <Text style={{ color: theme.text }}>Press me</Text>
    </Pressable>
  );
};
```

---

### Q77. What is `Suspense` in React Native?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Architecture

**Answer:**
`Suspense` lets components "wait" for something (data, lazy imports) and show a fallback UI while waiting. In React Native, it works with:
- `React.lazy` (code splitting)
- Data fetching frameworks that support Suspense (React Query, Relay)

```js
import { Suspense, lazy } from 'react';

const HeavyChart = lazy(() => import('./HeavyChart'));

const DashboardScreen = () => (
  <View>
    <Text>Dashboard</Text>
    <Suspense fallback={<ActivityIndicator />}>
      <HeavyChart />
    </Suspense>
  </View>
);

// With React Query's useSuspenseQuery
const UserProfile = () => {
  const { data } = useSuspenseQuery({ queryKey: ['user'], queryFn: fetchUser });
  return <Text>{data.name}</Text>;
};

// Wrap in Suspense above
<Suspense fallback={<Skeleton />}>
  <UserProfile />
</Suspense>
```

---

### Q78. What is `startTransition` and `useTransition`?

**Difficulty:** 🔴 Hard | **Frequency:** Low | **Category:** React 18

**Answer:**
From React 18. Marks state updates as "non-urgent transitions" — React can interrupt them to handle more urgent updates (like user input), keeping the UI responsive.

```js
import { startTransition, useTransition } from 'react';

// useTransition — returns isPending flag + startTransition
const SearchScreen = () => {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();

  const handleSearch = (text) => {
    setQuery(text); // urgent — update input immediately

    startTransition(() => {
      // non-urgent — can be interrupted if user types again
      const filtered = heavyFilter(data, text);
      setResults(filtered);
    });
  };

  return (
    <View>
      <TextInput value={query} onChangeText={handleSearch} />
      {isPending && <ActivityIndicator />}
      <FlatList data={results} renderItem={renderItem} />
    </View>
  );
};
```

---

### Q79. How do you implement a bottom sheet in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** UI Components

**Answer:**
Use `@gorhom/bottom-sheet` — the standard library for performant bottom sheets running on the UI thread via Reanimated.

```js
import BottomSheet, { BottomSheetView } from '@gorhom/bottom-sheet';

const MyScreen = () => {
  const bottomSheetRef = useRef(null);

  // Define snap points
  const snapPoints = useMemo(() => ['25%', '50%', '90%'], []);

  const handleOpen = () => bottomSheetRef.current?.expand();
  const handleClose = () => bottomSheetRef.current?.close();

  return (
    <View style={{ flex: 1 }}>
      <Button onPress={handleOpen} title="Open sheet" />

      <BottomSheet
        ref={bottomSheetRef}
        index={-1} // -1 = hidden initially
        snapPoints={snapPoints}
        enablePanDownToClose={true}
        backgroundStyle={{ backgroundColor: '#fff' }}
      >
        <BottomSheetView>
          <Text>Bottom sheet content</Text>
          <Button onPress={handleClose} title="Close" />
        </BottomSheetView>
      </BottomSheet>
    </View>
  );
};
```

---

### Q80. What is `React.StrictMode` and does it affect production?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** React

**Answer:**
`React.StrictMode` is a development tool that:
- Double-invokes render functions and hooks (to detect side effects)
- Warns about deprecated APIs
- Detects unexpected side effects
- Warns about legacy string refs

**It has NO effect in production** — all StrictMode checks are stripped.

```js
// In development: component renders twice on mount
// (to detect impure renders — should be idempotent)
const App = () => (
  <React.StrictMode>
    <NavigationContainer>
      <AppNavigator />
    </NavigationContainer>
  </React.StrictMode>
);
```

**Common confusion:** "My `useEffect` runs twice on mount!" → This only happens in development with StrictMode. React mounts → unmounts → remounts to verify cleanup. In production, it runs once.

---

### Q81. How do you handle multi-language support (i18n) in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Internationalisation

**Answer:**
Use `react-i18next` or `i18n-js` with `react-native-localize` to detect the device locale.

```js
// Setup i18n
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';
import * as RNLocalize from 'react-native-localize';

const resources = {
  en: { translation: { welcome: 'Welcome', submit: 'Submit' } },
  hi: { translation: { welcome: 'स्वागत है', submit: 'जमा करें' } },
};

i18n.use(initReactI18next).init({
  resources,
  lng: RNLocalize.getLocales()[0].languageCode, // device language
  fallbackLng: 'en',
});

// Usage
import { useTranslation } from 'react-i18next';

const HomeScreen = () => {
  const { t } = useTranslation();
  return <Text>{t('welcome')}</Text>;
};
```

---

### Q82. What is `Keyboard.dismiss` and when do you use it?

**Difficulty:** 🟢 Easy | **Frequency:** Medium | **Category:** Components

**Answer:**
```js
import { Keyboard, TouchableWithoutFeedback } from 'react-native';

// Dismiss keyboard when tapping outside an input
<TouchableWithoutFeedback onPress={Keyboard.dismiss} accessible={false}>
  <View style={{ flex: 1 }}>
    <TextInput placeholder="Search..." />
    <Text>Tap outside to dismiss keyboard</Text>
  </View>
</TouchableWithoutFeedback>

// Dismiss programmatically
const handleSubmit = () => {
  Keyboard.dismiss();
  submitForm();
};

// Listen to keyboard events
useEffect(() => {
  const showSub = Keyboard.addListener('keyboardDidShow', (e) => {
    setKeyboardHeight(e.endCoordinates.height);
  });
  const hideSub = Keyboard.addListener('keyboardDidHide', () => {
    setKeyboardHeight(0);
  });
  return () => { showSub.remove(); hideSub.remove(); };
}, []);
```

---

### Q83. How do you implement local notifications (alarm/reminder)?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Native APIs

**Answer:**
```js
import notifee, { TriggerType, AndroidImportance } from '@notifee/react-native';

// Create a channel (Android)
const createChannel = async () => {
  return await notifee.createChannel({
    id: 'reminders',
    name: 'Reminders',
    importance: AndroidImportance.HIGH,
    sound: 'default',
  });
};

// Schedule a notification
const scheduleReminder = async (title, date) => {
  const channelId = await createChannel();

  await notifee.createTriggerNotification(
    {
      title,
      body: 'You have a payment due!',
      android: { channelId },
    },
    {
      type: TriggerType.TIMESTAMP,
      timestamp: date.getTime(), // Unix timestamp
      repeatFrequency: RepeatFrequency.DAILY, // optional repeat
    }
  );
};

// In your Debt Relief app — EMI reminder
scheduleReminder('EMI Due', new Date(dueDate));
```

---

### Q84. What is the difference between `activityIndicator` and a custom loading spinner?

**Difficulty:** 🟢 Easy | **Frequency:** Low | **Category:** Components

**Answer:**
```js
// ActivityIndicator — native spinner (iOS: wheel, Android: circle)
<ActivityIndicator
  size="large"         // 'small' or 'large'
  color="#6200EE"
  animating={true}     // false = hidden
/>

// Custom spinner with Reanimated
const Spinner = () => {
  const rotation = useSharedValue(0);

  useEffect(() => {
    rotation.value = withRepeat(
      withTiming(360, { duration: 1000, easing: Easing.linear }),
      -1 // -1 = infinite
    );
  }, []);

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ rotate: `${rotation.value}deg` }],
  }));

  return (
    <Animated.View style={animatedStyle}>
      <Image source={require('./spinner.png')} />
    </Animated.View>
  );
};
```

---

### Q85. What is `BackHandler` in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Native APIs

**Answer:**
`BackHandler` handles the Android hardware back button press.

```js
import { BackHandler } from 'react-native';

// In a screen — override back button to show confirmation
useEffect(() => {
  const backAction = () => {
    Alert.alert(
      'Discard changes?',
      'Are you sure you want to go back?',
      [
        { text: 'Cancel', style: 'cancel', onPress: () => null },
        { text: 'Discard', onPress: () => navigation.goBack() },
      ]
    );
    return true; // returning true prevents default back action
  };

  const sub = BackHandler.addEventListener('hardwareBackPress', backAction);
  return () => sub.remove();
}, [navigation]);
```

**Note:** React Navigation automatically handles the back button for navigators. Only use `BackHandler` when you need custom behaviour (confirmation dialogs, preventing exit).

---

### Q86. How do you implement haptic feedback?

**Difficulty:** 🟢 Easy | **Frequency:** Medium | **Category:** Native APIs

**Answer:**
```js
import ReactNativeHapticFeedback from 'react-native-haptic-feedback';

const options = { enableVibrateFallback: true, ignoreAndroidSystemSettings: false };

// Different haptic types
ReactNativeHapticFeedback.trigger('impactLight', options);   // subtle tap
ReactNativeHapticFeedback.trigger('impactMedium', options);  // medium tap
ReactNativeHapticFeedback.trigger('impactHeavy', options);   // heavy tap
ReactNativeHapticFeedback.trigger('notificationSuccess', options);
ReactNativeHapticFeedback.trigger('notificationWarning', options);
ReactNativeHapticFeedback.trigger('notificationError', options);

// Usage in payment success
const handlePaymentSuccess = () => {
  ReactNativeHapticFeedback.trigger('notificationSuccess', options);
  navigation.navigate('PaymentSuccess');
};
```

---

### Q87. What is `react-native-reanimated` `withDecay`?

**Difficulty:** 🔴 Hard | **Frequency:** Low | **Cat



