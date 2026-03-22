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

**Difficulty:** 🔴 Hard | **Frequency:** Low | **Category:** Animations

**Answer:**
`withDecay` creates a physics-based animation that decelerates from an initial velocity (like a scroll or throw gesture continuing after release).

```js
import Animated, { useSharedValue, useAnimatedStyle, withDecay } from 'react-native-reanimated';
import { Gesture, GestureDetector } from 'react-native-gesture-handler';

const SwipeCard = () => {
  const translateX = useSharedValue(0);

  const panGesture = Gesture.Pan()
    .onUpdate((e) => {
      translateX.value = e.translationX;
    })
    .onEnd((e) => {
      // After release, card continues with velocity then slows to stop
      translateX.value = withDecay({
        velocity: e.velocityX,
        clamp: [-200, 200], // optional bounds
      });
    });

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ translateX: translateX.value }],
  }));

  return (
    <GestureDetector gesture={panGesture}>
      <Animated.View style={[styles.card, animatedStyle]} />
    </GestureDetector>
  );
};
```

---

### Q88. What is `Animated.event` and how does it sync animations with scroll?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Animations

**Answer:**
`Animated.event` maps native events directly to animated values without going through JS — enabling smooth 60fps scroll-driven animations.

```js
const scrollY = useRef(new Animated.Value(0)).current;

// Animated.event maps the scroll event's y value directly to scrollY
const onScroll = Animated.event(
  [{ nativeEvent: { contentOffset: { y: scrollY } } }],
  { useNativeDriver: true } // runs on UI thread
);

// Use scrollY to animate a header
const headerOpacity = scrollY.interpolate({
  inputRange: [0, 100],
  outputRange: [1, 0],   // fade out as you scroll
  extrapolate: 'clamp',  // don't go below 0 or above 1
});

return (
  <View>
    <Animated.View style={{ opacity: headerOpacity }}>
      <Text>Header</Text>
    </Animated.View>
    <Animated.ScrollView onScroll={onScroll} scrollEventThrottle={16}>
      {/* content */}
    </Animated.ScrollView>
  </View>
);
```

---

### Q89. What is `interpolate` in the Animated API?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Animations

**Answer:**
`interpolate` maps an animated value from one range to another, enabling complex animation relationships.

```js
const scrollY = useRef(new Animated.Value(0)).current;

// Map scroll 0-200 to opacity 1-0
const opacity = scrollY.interpolate({
  inputRange: [0, 200],
  outputRange: [1, 0],
  extrapolate: 'clamp', // 'extend' | 'clamp' | 'identity'
});

// Map scroll 0-100 to scale 1-1.5
const scale = scrollY.interpolate({
  inputRange: [0, 100],
  outputRange: [1, 1.5],
  extrapolate: 'clamp',
});

// Multi-step interpolation
const color = scrollY.interpolate({
  inputRange: [0, 50, 100],
  outputRange: ['rgb(255,255,255)', 'rgb(200,200,200)', 'rgb(0,0,0)'],
});
```

**`extrapolate: 'clamp'`** — the value won't go beyond the output range bounds. Without it, the value extrapolates linearly beyond the defined range.

---

### Q90. What is `LayoutAnimation` and when should you use it?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Animations

**Answer:**
`LayoutAnimation` automatically animates layout changes (add/remove elements, size changes) with a single call. It's simpler than the `Animated` API but less controllable.

```js
import { LayoutAnimation, Platform, UIManager } from 'react-native';

// Enable on Android (required — disabled by default)
if (Platform.OS === 'android') {
  UIManager.setLayoutAnimationEnabledExperimental(true);
}

const ExpandableSection = () => {
  const [expanded, setExpanded] = useState(false);

  const toggleExpand = () => {
    // Animate the next layout change
    LayoutAnimation.configureNext(LayoutAnimation.Presets.easeInEaseOut);
    setExpanded(!expanded);
  };

  return (
    <View>
      <Pressable onPress={toggleExpand}>
        <Text>Toggle</Text>
      </Pressable>
      {expanded && (
        <View> {/* This view appears/disappears with animation */}
          <Text>Expanded content here</Text>
        </View>
      )}
    </View>
  );
};
```

**When to use:** Simple show/hide, accordion, height changes. For precise control, use `Animated` or `Reanimated`.

---

## Section 3: Navigation (Q91–Q130)

---

### Q91. What are the navigator types in React Navigation?

**Difficulty:** 🟢 Easy | **Frequency:** High | **Category:** Navigation

**Answer:**

| Navigator | Behaviour | Use case |
|-----------|-----------|----------|
| Stack | Push/pop screens | Most app flows |
| Native Stack | Same as Stack but 100% native | Preferred for performance |
| Tab | Bottom/top tabs | Main app sections |
| Drawer | Side menu | Settings, admin |
| Material Top Tabs | Swipeable tabs | Category filtering |

```js
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';
import { createDrawerNavigator } from '@react-navigation/drawer';

// Nested: Tabs inside a Stack (common ERP pattern)
const Tab = createBottomTabNavigator();
const Stack = createNativeStackNavigator();

const MainTabs = () => (
  <Tab.Navigator>
    <Tab.Screen name="Dashboard" component={DashboardScreen} />
    <Tab.Screen name="Employees" component={EmployeesScreen} />
    <Tab.Screen name="Attendance" component={AttendanceScreen} />
  </Tab.Navigator>
);

const RootNavigator = () => (
  <Stack.Navigator>
    <Stack.Screen name="Login" component={LoginScreen} options={{ headerShown: false }} />
    <Stack.Screen name="Main" component={MainTabs} options={{ headerShown: false }} />
    <Stack.Screen name="Profile" component={ProfileScreen} />
  </Stack.Navigator>
);
```

---

### Q92. How do you protect routes (authentication guard) in React Navigation?

**Difficulty:** 🟡 Medium | **Frequency:** Very High | **Category:** Navigation

**Answer:**
The recommended pattern is conditional rendering of navigators based on auth state, not intercepting navigation.

```js
const App = () => {
  const { isAuthenticated, isLoading } = useAuth();

  if (isLoading) return <SplashScreen />;

  return (
    <NavigationContainer>
      {isAuthenticated ? <AuthenticatedStack /> : <UnauthenticatedStack />}
    </NavigationContainer>
  );
};

const AuthenticatedStack = () => (
  <Stack.Navigator>
    <Stack.Screen name="Dashboard" component={DashboardScreen} />
    <Stack.Screen name="Profile" component={ProfileScreen} />
  </Stack.Navigator>
);

const UnauthenticatedStack = () => (
  <Stack.Navigator screenOptions={{ headerShown: false }}>
    <Stack.Screen name="Login" component={LoginScreen} />
    <Stack.Screen name="Register" component={RegisterScreen} />
  </Stack.Navigator>
);
```

**Why not `navigation.navigate` in useEffect?** → Conditional navigator rendering is React's declarative approach. Imperative navigation for auth can cause race conditions and flickering.

---

### Q93. How do you navigate without access to the `navigation` prop?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Navigation

**Answer:**
Use a navigation ref to access navigation outside the component tree.

```js
// navigationRef.js
import { createNavigationContainerRef } from '@react-navigation/native';
export const navigationRef = createNavigationContainerRef();

export const navigate = (name, params) => {
  if (navigationRef.isReady()) {
    navigationRef.navigate(name, params);
  }
};

// App.js
import { NavigationContainer } from '@react-navigation/native';
import { navigationRef } from './navigationRef';

<NavigationContainer ref={navigationRef}>
  <AppNavigator />
</NavigationContainer>

// Usage anywhere (API interceptor, Redux action, etc.)
import { navigate } from './navigationRef';

// In Redux-Saga or Axios interceptor (outside React tree)
api.interceptors.response.use(null, (error) => {
  if (error.response?.status === 401) {
    navigate('Login'); // navigate from outside React
  }
  return Promise.reject(error);
});
```

---

### Q94. What is the `useFocusEffect` hook in React Navigation?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Navigation

**Answer:**
`useFocusEffect` runs an effect when the screen comes into focus and cleans up when it leaves focus. Unlike `useEffect` which only runs on mount, `useFocusEffect` runs every time you navigate to the screen.

```js
import { useFocusEffect } from '@react-navigation/native';

const ProfileScreen = () => {
  useFocusEffect(
    useCallback(() => {
      // Runs every time this screen is focused
      fetchLatestProfile();
      const subscription = subscribeToUpdates();

      return () => {
        // Runs when screen loses focus (navigated away)
        subscription.unsubscribe();
      };
    }, [])
  );

  return <Profile />;
};
```

**Use case in your ERP app:** Each screen refreshes its data when navigated back to (e.g., after editing an employee, going back to the list shows updated data).

---

### Q95. What is `useNavigationState`?

**Difficulty:** 🔴 Hard | **Frequency:** Low | **Category:** Navigation

**Answer:**
`useNavigationState` gives you access to the navigation state of the navigator. Use when you need to know the current route, history, or index.

```js
import { useNavigationState } from '@react-navigation/native';

const MyComponent = () => {
  const routeCount = useNavigationState(state => state.routes.length);
  const currentRouteName = useNavigationState(state => state.routes[state.index].name);

  return (
    <View>
      <Text>Current: {currentRouteName}</Text>
      <Text>Stack depth: {routeCount}</Text>
    </View>
  );
};
```

---

### Q96. How do you implement a custom header in React Navigation?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Navigation

**Answer:**
```js
// Option 1: headerRight, headerLeft, headerTitle options
<Stack.Screen
  name="Profile"
  component={ProfileScreen}
  options={({ navigation, route }) => ({
    headerTitle: () => <CustomTitle name={route.params.name} />,
    headerRight: () => (
      <Pressable onPress={() => navigation.navigate('Settings')}>
        <Icon name="settings" />
      </Pressable>
    ),
    headerLeft: () => (
      <Pressable onPress={() => navigation.goBack()}>
        <Icon name="back" />
      </Pressable>
    ),
    headerStyle: { backgroundColor: '#6200EE' },
    headerTintColor: '#fff',
    headerShown: true,
  })}
/>

// Option 2: fully custom header (replace the entire header)
options={{ header: ({ navigation }) => <MyFullCustomHeader /> }}

// Option 3: hide header and use your own
options={{ headerShown: false }}
```

---

### Q97. How do you animate between screens in React Navigation?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Navigation

**Answer:**
```js
// Native Stack — built-in animations
<Stack.Screen
  name="Detail"
  component={DetailScreen}
  options={{
    animation: 'slide_from_right',     // iOS-style
    // 'slide_from_bottom' — modal style
    // 'fade' — crossfade
    // 'none' — instant
    // 'flip' — flip transition
    presentation: 'modal',             // presents as modal
  }}
/>

// JS Stack — custom animation via cardStyleInterpolator
import { CardStyleInterpolators } from '@react-navigation/stack';

<Stack.Screen
  options={{
    cardStyleInterpolator: CardStyleInterpolators.forModalPresentationIOS,
  }}
/>

// Custom slide-up animation
const forSlideUp = ({ current, layouts }) => ({
  cardStyle: {
    transform: [{
      translateY: current.progress.interpolate({
        inputRange: [0, 1],
        outputRange: [layouts.screen.height, 0],
      }),
    }],
  },
});
```

---

### Q98. How do you implement shared element transitions in React Navigation?

**Difficulty:** 🔴 Hard | **Frequency:** Low | **Category:** Navigation

**Answer:**
Shared element transitions animate a UI element from one screen to another seamlessly. Use `react-native-shared-element` with React Navigation.

```js
import { createSharedElementStackNavigator } from 'react-navigation-shared-element';

const Stack = createSharedElementStackNavigator();

// Screen 1: List with shared element
const ImageCard = ({ item }) => (
  <SharedElement id={`item.${item.id}.photo`}>
    <Image source={{ uri: item.photoUrl }} style={styles.thumbnail} />
  </SharedElement>
);

// Screen 2: Detail with shared element (same id)
const DetailScreen = ({ route }) => {
  const { item } = route.params;
  return (
    <SharedElement id={`item.${item.id}.photo`}>
      <Image source={{ uri: item.photoUrl }} style={styles.fullImage} />
    </SharedElement>
  );
};

// Navigator config
<Stack.Screen
  name="Detail"
  component={DetailScreen}
  sharedElements={(route) => [`item.${route.params.item.id}.photo`]}
/>
```

---

### Q99. What is the `navigation.reset` method and when do you use it?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Navigation

**Answer:**
`navigation.reset` replaces the entire navigation state — useful for logging out (clearing the stack so back doesn't go to a logged-in screen).

```js
// After logout — reset to Login screen, clear all history
const handleLogout = () => {
  dispatch(clearAuthTokens());
  navigation.reset({
    index: 0,
    routes: [{ name: 'Login' }],
  });
  // User can't press back to get to authenticated screens
};

// After onboarding completion — reset to main app
navigation.reset({
  index: 0,
  routes: [{ name: 'MainTabs' }],
});

// Replace current screen (no back button)
navigation.replace('NewScreen', { params });
```

---

### Q100. What is `linking` configuration in React Navigation for deep links?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Navigation

**Answer:**
```js
// Complete deep link configuration
const linking = {
  prefixes: ['myerp://', 'https://myerp.com'],
  config: {
    screens: {
      Login: 'login',
      MainTabs: {
        screens: {
          Dashboard: 'dashboard',
          Employees: {
            screens: {
              EmployeeList: 'employees',
              EmployeeDetail: 'employees/:id', // dynamic param
            },
          },
        },
      },
      Settings: 'settings',
    },
  },

  // Custom URL parsing
  getStateFromPath: (path, options) => {
    // Custom logic for non-standard URLs
    return getStateFromPath(path, options);
  },

  // Custom URL generation
  getPathFromState: (state, config) => {
    return getPathFromState(state, config);
  },
};

<NavigationContainer linking={linking} fallback={<SplashScreen />}>
  <AppNavigator />
</NavigationContainer>
```

---

---

## Section 4: Hooks Deep Dive — Continued (Q101–Q130)

---

### Q101. What is `useSyncExternalStore` and why was it introduced?

**Difficulty:** 🔴 Hard | **Frequency:** Low | **Category:** React 18 Hooks

**Answer:**
`useSyncExternalStore` is a React 18 hook for subscribing to external stores (non-React state — like Redux, Zustand, or browser APIs) in a way that's safe with concurrent rendering. Before React 18, subscribing to external stores in `useEffect` could cause "tearing" — different components reading different values from the same store within a single render pass.

`useSyncExternalStore` solves this by making React aware of the store subscription and ensuring a consistent snapshot.

**Code:**
```js
import { useSyncExternalStore } from 'react';

// Subscribe to any external store
const useNetworkStatus = () => {
  return useSyncExternalStore(
    // subscribe — called when store changes, returns unsubscribe fn
    (callback) => {
      const listener = NetInfo.addEventListener(callback);
      return () => listener(); // unsubscribe
    },
    // getSnapshot — returns current value (must be stable reference if unchanged)
    () => NetInfo.fetch(),
    // getServerSnapshot — optional, for SSR
    () => ({ isConnected: true })
  );
};

// Custom store example
let store = { count: 0 };
let listeners = new Set();

const counterStore = {
  subscribe: (cb) => { listeners.add(cb); return () => listeners.delete(cb); },
  getSnapshot: () => store,
  increment: () => {
    store = { count: store.count + 1 };
    listeners.forEach(cb => cb());
  },
};

const Counter = () => {
  const { count } = useSyncExternalStore(
    counterStore.subscribe,
    counterStore.getSnapshot
  );
  return <Text>{count}</Text>;
};
```

**Follow-up:** Why can't you just use `useEffect` + `useState` for this? → In concurrent mode, React can render components multiple times without committing. If an external store updates between these renders, components could read inconsistent values — "tearing". `useSyncExternalStore` prevents this.

---

### Q102. What is `useDeferredValue`?

**Difficulty:** 🔴 Hard | **Frequency:** Low | **Category:** React 18 Hooks

**Answer:**
`useDeferredValue` accepts a value and returns a deferred version of it. During urgent updates (like typing), the deferred value lags behind — React re-renders the non-urgent parts with the old value first, then updates them in the background.

It is the value equivalent of `useTransition` (which wraps state updates).

**Code:**
```js
import { useDeferredValue, useMemo } from 'react';

const SearchScreen = ({ query }) => {
  // deferredQuery lags behind query during typing
  const deferredQuery = useDeferredValue(query);

  // This expensive computation uses the deferred (old) value
  // while the input shows the latest value immediately
  const results = useMemo(() => {
    return heavyFilter(allData, deferredQuery);
  }, [deferredQuery]);

  const isStale = query !== deferredQuery; // true while deferred

  return (
    <View style={{ opacity: isStale ? 0.5 : 1 }}>
      <FlatList data={results} renderItem={renderItem} />
    </View>
  );
};
```

**Difference from `useTransition`:**
- `useTransition` — wraps a state *setter* call, marks the update as non-urgent
- `useDeferredValue` — wraps a *value* (often a prop you don't control), defers its propagation

**Follow-up:** When would you use `useDeferredValue` over `debounce`? → `useDeferredValue` is aware of React's scheduler — it defers work during busy renders without a fixed time delay. Debounce uses a fixed timer regardless of system load.

---

### Q103. What is the difference between `useEffect` and `useInsertionEffect`?

**Difficulty:** 🔴 Hard | **Frequency:** Very Low | **Category:** React 18 Hooks

**Answer:**
`useInsertionEffect` fires synchronously before any DOM mutations — even before `useLayoutEffect`. It was added specifically for CSS-in-JS libraries that need to inject styles before the browser measures layout.

**Execution order:**
```
render → useInsertionEffect → DOM mutation → useLayoutEffect → paint → useEffect
```

**Code:**
```js
import { useInsertionEffect } from 'react';

// Only for CSS-in-JS library authors — NOT for app code
const useStyles = (styles) => {
  useInsertionEffect(() => {
    // Inject a <style> tag before layout is measured
    const styleEl = document.createElement('style');
    styleEl.innerHTML = styles;
    document.head.appendChild(styleEl);
    return () => document.head.removeChild(styleEl);
  }, [styles]);
};
```

**In React Native:** Almost never used — there is no CSS injection. It's mainly relevant for styled-components or Emotion on React Native Web.

**Follow-up:** What is the real-world use? → Libraries like styled-components use it to inject styles before `useLayoutEffect` reads layout, preventing a flash of unstyled content.

---

### Q104. How do you create a global state without Redux?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** State Management

**Answer:**
Three common patterns for global state without Redux:

**Pattern 1: Context + useReducer (Redux-lite)**
```js
// store.js
const initialState = { user: null, theme: 'light', cart: [] };

const reducer = (state, action) => {
  switch (action.type) {
    case 'SET_USER': return { ...state, user: action.payload };
    case 'SET_THEME': return { ...state, theme: action.payload };
    default: return state;
  }
};

const StoreContext = createContext(null);
const DispatchContext = createContext(null);

export const StoreProvider = ({ children }) => {
  const [state, dispatch] = useReducer(reducer, initialState);
  return (
    <StoreContext.Provider value={state}>
      <DispatchContext.Provider value={dispatch}>
        {children}
      </DispatchContext.Provider>
    </StoreContext.Provider>
  );
};

// Split contexts so components only re-render for what they use
export const useStore = () => useContext(StoreContext);
export const useDispatch = () => useContext(DispatchContext);
```

**Pattern 2: Zustand (simplest)**
```js
import { create } from 'zustand';

const useStore = create((set) => ({
  user: null,
  count: 0,
  setUser: (user) => set({ user }),
  increment: () => set((state) => ({ count: state.count + 1 })),
}));

// Any component — no Provider needed
const ProfileScreen = () => {
  const { user, setUser } = useStore();
  return <Text>{user?.name}</Text>;
};
```

**Pattern 3: `useSyncExternalStore` with a custom store**
```js
// Covered in Q101 — use for complex subscription scenarios
```

**Follow-up:** When should you reach for Redux vs Zustand vs Context? → Context for small apps or auth/theme. Zustand for medium apps wanting simplicity. Redux for large teams needing DevTools, middleware, and strict conventions.

---

### Q105. How does React batch state updates?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** React Internals

**Answer:**
Batching means React groups multiple `setState` calls into a single re-render. This prevents wasteful intermediate renders.

**React 17 and earlier — only batched inside React event handlers:**
```js
// Inside a React event handler — batched (one re-render)
const handlePress = () => {
  setCount(c => c + 1);
  setName('Devesh');
  // ONE re-render
};

// Inside setTimeout / async — NOT batched (two re-renders)
setTimeout(() => {
  setCount(c => c + 1); // re-render 1
  setName('Devesh');    // re-render 2
}, 0);
```

**React 18 — automatic batching everywhere:**
```js
// ALL of these are batched in React 18 — even in async, setTimeout, promises
setTimeout(() => {
  setCount(c => c + 1);
  setName('Devesh');
  // ONE re-render ✅
}, 0);

fetch('/api').then(() => {
  setData(result);
  setLoading(false);
  // ONE re-render ✅
});
```

**Opt out of batching (rare):**
```js
import { flushSync } from 'react-dom';

flushSync(() => setCount(c => c + 1)); // forces immediate re-render
flushSync(() => setName('Devesh'));     // second immediate re-render
```

**Follow-up:** Does batching cause any issues? → Rarely. If you need to read updated DOM between two state updates (e.g., measure layout after first update), `flushSync` forces a synchronous flush.

---

### Q106. What is the difference between `null`, `undefined`, and `false` in JSX rendering?

**Difficulty:** 🟢 Easy | **Frequency:** Medium | **Category:** React Basics

**Answer:**
In JSX, `null`, `undefined`, `false`, and empty string `''` all render nothing (no output). But there's one common trap:

```js
// These all render nothing — safe
{null}
{undefined}
{false}
{true}

// ⚠️ TRAP — 0 renders as "0" on screen!
const count = 0;
{count && <Text>Has items</Text>}
// Renders "0" not nothing! Because 0 is falsy but not null/false

// ✅ Fix — convert to boolean
{count > 0 && <Text>Has items</Text>}
{!!count && <Text>Has items</Text>}
{Boolean(count) && <Text>Has items</Text>}

// Or use ternary
{count ? <Text>Has items</Text> : null}
```

**This is one of the most common React Native bugs in production** — a `0` unexpectedly showing up in the UI because of `{array.length && <Component />}`.

**Follow-up:** Why does `0 && <Component />` render `0`? → `&&` short-circuits and returns the first falsy value. `0` is falsy, so it returns `0` — which JSX renders as the text "0".

---

### Q107. What is prop drilling and what are its solutions?

**Difficulty:** 🟢 Easy | **Frequency:** High | **Category:** Architecture

**Answer:**
Prop drilling is passing props through multiple component layers just to reach a deeply nested component that needs the data. The intermediate components don't use the data — they just pass it through.

```js
// Prop drilling (bad for deep trees)
const App = () => <Screen user={user} />;
const Screen = ({ user }) => <Header user={user} />;
const Header = ({ user }) => <Avatar user={user} />;
const Avatar = ({ user }) => <Text>{user.name}</Text>; // only this needs it!

// Solutions:

// 1. Context API
const UserContext = createContext(null);
const App = () => (
  <UserContext.Provider value={user}>
    <Screen />
  </UserContext.Provider>
);
const Avatar = () => {
  const user = useContext(UserContext);
  return <Text>{user.name}</Text>;
};

// 2. Component composition (pass JSX as children/props)
const App = () => (
  <Screen header={<Avatar user={user} />} />
);
const Screen = ({ header }) => <View>{header}</View>;

// 3. Redux / Zustand global store
const Avatar = () => {
  const user = useSelector(state => state.auth.user);
  return <Text>{user.name}</Text>;
};
```

**Follow-up:** Is prop drilling always bad? → No. For 1–2 levels deep, prop drilling is fine and explicit. Over-using Context or Redux for simple cases adds complexity. The rule: if you're drilling through 3+ unrelated components, consider an alternative.

---

### Q108. How do you implement a HOC (Higher Order Component) in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Patterns

**Answer:**
A Higher Order Component is a function that takes a component and returns a new enhanced component. It's a pattern for reusing component logic (predates hooks; hooks are usually preferred now).

**Code:**
```js
// HOC: withAuth — redirects to Login if not authenticated
const withAuth = (WrappedComponent) => {
  return (props) => {
    const { isAuthenticated } = useAuth();
    const navigation = useNavigation();

    useEffect(() => {
      if (!isAuthenticated) {
        navigation.replace('Login');
      }
    }, [isAuthenticated]);

    if (!isAuthenticated) return null;
    return <WrappedComponent {...props} />;
  };
};

// Usage
const DashboardScreen = ({ navigation }) => <View>...</View>;
export default withAuth(DashboardScreen);

// HOC: withLoading — shows spinner while loading prop is true
const withLoading = (WrappedComponent) => {
  return ({ isLoading, ...props }) => {
    if (isLoading) return <ActivityIndicator size="large" />;
    return <WrappedComponent {...props} />;
  };
};

// HOC: withErrorBoundary — wraps in error boundary
const withErrorBoundary = (WrappedComponent, FallbackComponent) => {
  return (props) => (
    <ErrorBoundary fallback={<FallbackComponent />}>
      <WrappedComponent {...props} />
    </ErrorBoundary>
  );
};
```

**HOC vs Custom Hook:** HOCs wrap the component (JSX tree). Custom hooks share logic without wrapping. Prefer custom hooks — they're simpler, composable, and easier to test.

---

### Q109. What is the render props pattern?

**Difficulty:** 🟡 Medium | **Frequency:** Low | **Category:** Patterns

**Answer:**
Render props is a pattern where a component's `render` (or `children`) prop is a function that returns JSX. The component calls this function with data it controls, letting the consumer decide what to render.

**Code:**
```js
// DataFetcher — handles loading/error, calls render with data
const DataFetcher = ({ url, render }) => {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    api.get(url).then(setData).finally(() => setLoading(false));
  }, [url]);

  return render({ data, loading });
};

// Consumer decides what to render with the data
<DataFetcher
  url="/users"
  render={({ data, loading }) => {
    if (loading) return <Spinner />;
    return <UserList users={data} />;
  }}
/>

// Children as a function (same pattern, different prop)
const MouseTracker = ({ children }) => {
  const [pos, setPos] = useState({ x: 0, y: 0 });
  return (
    <View onStartShouldSetResponder={() => true}
          onResponderMove={(e) => setPos({ x: e.nativeEvent.pageX, y: e.nativeEvent.pageY })}>
      {children(pos)}
    </View>
  );
};

<MouseTracker>
  {({ x, y }) => <Text>Position: {x}, {y}</Text>}
</MouseTracker>
```

**Note:** Custom hooks have largely replaced render props for logic sharing. Render props are still useful for component-level patterns where you want the consumer to control the rendered output.

---

### Q110. What is memoization and when does it hurt performance?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Performance

**Answer:**
Memoization caches the result of a computation. React provides `useMemo`, `useCallback`, and `React.memo` for memoization. But memoization has a cost — it must store the previous result and compare dependencies on every render.

**When memoization HELPS:**
```js
// Expensive computation — filtering 10,000 items
const filtered = useMemo(() => bigArray.filter(heavyFn), [bigArray]);

// Stable callback passed to a memoized child
const onPress = useCallback(() => navigate('Detail', { id }), [id]);
const Child = React.memo(({ onPress }) => <Button onPress={onPress} />);
```

**When memoization HURTS (or is useless):**
```js
// ❌ Cheap computation — no benefit, adds overhead
const sum = useMemo(() => a + b, [a, b]);
// Just write: const sum = a + b;

// ❌ New object deps — memo never hits because deps always "change"
const options = useMemo(() => getData(), [{ id: 1 }]); // new object every render!

// ❌ Memoizing a component that re-renders anyway because its parent passes a new object prop
const Parent = () => {
  return <MemoizedChild style={{ margin: 10 }} />; // new object → memo useless
};
// Fix: move style to StyleSheet.create or useMemo
```

**Rule:** Profile first. Only add `useMemo`/`useCallback` when you've identified a real performance problem. Premature optimisation adds cognitive overhead and can introduce bugs (stale deps).

---

### Q111. What is `React.Children` and when do you use it?

**Difficulty:** 🟡 Medium | **Frequency:** Low | **Category:** React API

**Answer:**
`React.Children` provides utilities for working with the `children` prop — mapping, counting, iterating, or converting children to an array, regardless of whether `children` is a single element, an array, or `null`.

**Code:**
```js
import React from 'react';

// Custom Tab component that injects active state into each Tab
const TabGroup = ({ children, activeTab }) => {
  return (
    <View style={styles.row}>
      {React.Children.map(children, (child, index) => {
        if (!React.isValidElement(child)) return null;

        // Clone each child and inject extra props
        return React.cloneElement(child, {
          isActive: index === activeTab,
          key: index,
        });
      })}
    </View>
  );
};

// Usage
<TabGroup activeTab={0}>
  <Tab label="Home" />
  <Tab label="Profile" />
  <Tab label="Settings" />
</TabGroup>

// Other React.Children methods
React.Children.count(children);         // total count (handles null/undefined)
React.Children.toArray(children);       // flat array of children
React.Children.forEach(children, fn);   // iterate without returning
React.Children.only(children);          // assert exactly one child (throws otherwise)
```

**Follow-up:** What is `React.cloneElement`? → Creates a copy of an element with merged props. Used in HOCs and compound component patterns to inject props into children.

---

### Q112. What is the compound component pattern?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Patterns

**Answer:**
Compound components are a group of components that work together and share implicit state through Context. The parent manages state; children access it via `useContext` — no prop drilling.

**Code:**
```js
// Accordion compound component
const AccordionContext = createContext(null);

const Accordion = ({ children }) => {
  const [openIndex, setOpenIndex] = useState(null);
  return (
    <AccordionContext.Provider value={{ openIndex, setOpenIndex }}>
      <View>{children}</View>
    </AccordionContext.Provider>
  );
};

const AccordionItem = ({ children, index }) => {
  const { openIndex, setOpenIndex } = useContext(AccordionContext);
  const isOpen = openIndex === index;

  return (
    <View>
      <Pressable onPress={() => setOpenIndex(isOpen ? null : index)}>
        {children[0] /* Header */}
      </Pressable>
      {isOpen && children[1] /* Body */}
    </View>
  );
};

// Attach sub-components to parent
Accordion.Item = AccordionItem;

// Consumer — clean, readable API
<Accordion>
  <Accordion.Item index={0}>
    <Text>Question 1</Text>
    <Text>Answer 1</Text>
  </Accordion.Item>
  <Accordion.Item index={1}>
    <Text>Question 2</Text>
    <Text>Answer 2</Text>
  </Accordion.Item>
</Accordion>
```

**Real-world use in your ERP app:** The HRMS module's form sections used compound components — `<FormSection>`, `<FormSection.Field>`, `<FormSection.Error>` — sharing form state internally without prop drilling.

---

### Q113. How do you handle multiple refs with `useRef`?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Hooks

**Answer:**
```js
// Pattern 1: Ref array for dynamic lists (e.g. form fields)
const fields = ['name', 'email', 'phone', 'address'];
const inputRefs = useRef(fields.map(() => createRef()));

// Focus next field on submit
const focusNext = (index) => {
  if (index < fields.length - 1) {
    inputRefs.current[index + 1].current?.focus();
  }
};

return (
  <View>
    {fields.map((field, i) => (
      <TextInput
        key={field}
        ref={inputRefs.current[i]}
        placeholder={field}
        returnKeyType={i < fields.length - 1 ? 'next' : 'done'}
        onSubmitEditing={() => focusNext(i)}
      />
    ))}
  </View>
);

// Pattern 2: Ref callback (set ref on dynamic items)
const refsMap = useRef({});

const setRef = useCallback((id) => (el) => {
  refsMap.current[id] = el;
}, []);

items.map(item => (
  <TextInput key={item.id} ref={setRef(item.id)} />
));

// Access: refsMap.current[item.id]?.focus();
```

---

### Q114. What is `useEvent` (React RFC) and why was it proposed?

**Difficulty:** 🔴 Hard | **Frequency:** Low | **Category:** React Hooks

**Answer:**
`useEvent` is a proposed React hook (RFC — not yet in stable React) that would return a stable function reference that always reads the latest props/state, without needing to list them as `useCallback` dependencies.

**The problem it solves:**
```js
// Current problem: either stale closure OR re-creating function every render
const handlePress = useCallback(() => {
  doSomethingWith(count); // count must be in deps
}, [count]); // new function every time count changes → defeats React.memo

// useEvent proposal — stable reference, always reads latest state
const handlePress = useEvent(() => {
  doSomethingWith(count); // no deps needed — always latest count
  // reference never changes → React.memo works perfectly
});
```

**Current workaround (before useEvent ships):**
```js
// useStableCallback custom hook
const useStableCallback = (fn) => {
  const ref = useRef(fn);
  useLayoutEffect(() => { ref.current = fn; });
  return useCallback((...args) => ref.current(...args), []);
};
```

**Follow-up:** Why hasn't `useEvent` shipped yet? → The React team is refining the semantics, especially around Concurrent Mode edge cases where calling the function during rendering (not just events) could cause issues.

---

### Q115. How do you implement optimistic UI updates?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** UX Patterns

**Answer:**
Optimistic UI updates apply the change to the UI immediately (assuming success) and roll back if the server returns an error. This makes the app feel instant.

**Code:**
```js
const TodoList = () => {
  const [todos, setTodos] = useState(initialTodos);

  const toggleTodo = async (id) => {
    // 1. Optimistically update UI immediately
    const previous = todos;
    setTodos(todos.map(t =>
      t.id === id ? { ...t, completed: !t.completed } : t
    ));

    try {
      // 2. Send to server in background
      await api.patch(`/todos/${id}/toggle`);
    } catch (error) {
      // 3. Rollback on error
      setTodos(previous);
      showToast('Failed to update. Please try again.');
    }
  };

  return (
    <FlatList
      data={todos}
      keyExtractor={t => t.id.toString()}
      renderItem={({ item }) => (
        <Pressable onPress={() => toggleTodo(item.id)}>
          <Text style={{ textDecorationLine: item.completed ? 'line-through' : 'none' }}>
            {item.title}
          </Text>
        </Pressable>
      )}
    />
  );
};
```

**With React Query (recommended):**
```js
const { mutate } = useMutation(toggleTodo, {
  onMutate: async (id) => {
    await queryClient.cancelQueries(['todos']);
    const previous = queryClient.getQueryData(['todos']);
    queryClient.setQueryData(['todos'], old =>
      old.map(t => t.id === id ? { ...t, completed: !t.completed } : t)
    );
    return { previous };
  },
  onError: (err, id, context) => {
    queryClient.setQueryData(['todos'], context.previous); // rollback
  },
  onSettled: () => queryClient.invalidateQueries(['todos']),
});
```

---

### Q116. What is the `children` prop and how do you type it in TypeScript?

**Difficulty:** 🟢 Easy | **Frequency:** Medium | **Category:** TypeScript + React

**Answer:**
```ts
import React, { ReactNode, ReactElement, FC, PropsWithChildren } from 'react';

// Option 1: ReactNode — most permissive (string, number, JSX, null, array)
interface CardProps {
  children: ReactNode;
  title: string;
}
const Card: FC<CardProps> = ({ children, title }) => (
  <View>
    <Text>{title}</Text>
    {children}
  </View>
);

// Option 2: PropsWithChildren — shortcut
const Card: FC<PropsWithChildren<{ title: string }>> = ({ children, title }) => (
  <View><Text>{title}</Text>{children}</View>
);

// Option 3: ReactElement — only JSX (not strings/numbers)
interface StrictProps {
  children: ReactElement; // exactly one React element
}

// Option 4: React.FC automatically includes children in React 17
// (removed in React 18 — must be explicit)

// Rendering children conditionally
const Wrapper = ({ children }: { children: ReactNode }) => {
  const count = React.Children.count(children);
  if (count === 0) return null;
  return <View>{children}</View>;
};
```

---

### Q117. How do you share logic between a React Native app and a React web app?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Architecture

**Answer:**
The key is separating business logic (platform-agnostic) from UI (platform-specific) using a monorepo structure.

**Strategies:**

**1. Custom hooks (no UI, pure logic):**
```js
// packages/shared/useAuth.ts — works in RN and web
export const useAuth = () => {
  const [user, setUser] = useState(null);
  const login = async (creds) => { /* pure logic, no RN/web APIs */ };
  return { user, login };
};

// React Native app
import { useAuth } from '@myapp/shared';
const LoginScreen = () => { const { login } = useAuth(); ... };

// React web app
import { useAuth } from '@myapp/shared';
const LoginPage = () => { const { login } = useAuth(); ... };
```

**2. Platform-specific file extensions:**
```
Button.tsx         // shared interface/types
Button.native.tsx  // RN implementation (Pressable)
Button.web.tsx     // Web implementation (<button>)
```

**3. React Native for Web:**
```js
// Run the same RN components in a browser
// metro.config.js resolves .web.js before .js
// Works for simple components; complex native components need separate implementations
```

**4. Turbo-repo / Nx monorepo:**
```
apps/
  mobile/    # React Native
  web/       # Next.js
packages/
  ui/        # shared components
  hooks/     # shared custom hooks
  api/       # shared API layer
  utils/     # shared utilities
```

---

### Q118. What is `Animated.parallel` vs `Animated.sequence` vs `Animated.stagger`?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Animations

**Answer:**
These are combination APIs for the `Animated` library:

```js
const opacity = useRef(new Animated.Value(0)).current;
const translateY = useRef(new Animated.Value(50)).current;
const scale = useRef(new Animated.Value(0.8)).current;

// parallel — all animations start at the same time
Animated.parallel([
  Animated.timing(opacity, { toValue: 1, duration: 300, useNativeDriver: true }),
  Animated.timing(translateY, { toValue: 0, duration: 300, useNativeDriver: true }),
  Animated.timing(scale, { toValue: 1, duration: 300, useNativeDriver: true }),
]).start();

// sequence — each animation starts when the previous finishes
Animated.sequence([
  Animated.timing(opacity, { toValue: 1, duration: 200, useNativeDriver: true }),
  Animated.timing(translateY, { toValue: 0, duration: 200, useNativeDriver: true }),
  Animated.timing(scale, { toValue: 1, duration: 200, useNativeDriver: true }),
]).start();

// stagger — like parallel but each starts delay ms after the previous
// Great for list item entrance animations
const items = [val1, val2, val3, val4];
Animated.stagger(
  100, // 100ms between each start
  items.map(val =>
    Animated.timing(val, { toValue: 1, duration: 300, useNativeDriver: true })
  )
).start();
```

**Real use case:** Stagger animation for list items appearing one by one on screen mount — each item fades in 100ms after the previous one.

---

### Q119. What is `Animated.loop` and how do you create a pulsing animation?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Animations

**Answer:**
`Animated.loop` repeats an animation indefinitely (or a set number of times).

**Code:**
```js
const pulseAnim = useRef(new Animated.Value(1)).current;

useEffect(() => {
  const pulse = Animated.loop(
    Animated.sequence([
      Animated.timing(pulseAnim, {
        toValue: 1.2,
        duration: 800,
        easing: Easing.inOut(Easing.ease),
        useNativeDriver: true,
      }),
      Animated.timing(pulseAnim, {
        toValue: 1,
        duration: 800,
        easing: Easing.inOut(Easing.ease),
        useNativeDriver: true,
      }),
    ])
  );

  pulse.start();
  return () => pulse.stop(); // cleanup on unmount
}, []);

<Animated.View style={{ transform: [{ scale: pulseAnim }] }}>
  <View style={styles.notificationDot} />
</Animated.View>

// Rotating loader
const spinAnim = useRef(new Animated.Value(0)).current;

const spin = spinAnim.interpolate({
  inputRange: [0, 1],
  outputRange: ['0deg', '360deg'],
});

Animated.loop(
  Animated.timing(spinAnim, {
    toValue: 1,
    duration: 1000,
    easing: Easing.linear,
    useNativeDriver: true,
  })
).start();

<Animated.View style={{ transform: [{ rotate: spin }] }}>
  <ActivityIndicator />
</Animated.View>
```

---

### Q120. What is the difference between `transform` and layout properties in animations?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Animations

**Answer:**
This is critical for performance:

**Transform properties** (translateX, translateY, scale, rotate, skewX, skewY):
- Run on the **UI thread** when `useNativeDriver: true`
- Do NOT affect layout — other components are not repositioned
- 60fps even when JS thread is busy
- ✅ **Always prefer for animations**

**Layout properties** (width, height, top, left, margin, padding, flex):
- Must run on the **JS thread** (Yoga layout engine)
- Trigger full layout recalculation
- `useNativeDriver: true` NOT supported
- ⚠️ Use sparingly — causes layout passes

```js
// ✅ GPU-accelerated, UI thread, smooth
Animated.timing(translateX, {
  toValue: 100,
  useNativeDriver: true, // supported
}).start();

<Animated.View style={{ transform: [{ translateX }] }} />

// ⚠️ Layout-triggered, JS thread, can jank
const widthAnim = useRef(new Animated.Value(100)).current;
Animated.timing(widthAnim, {
  toValue: 200,
  useNativeDriver: false, // cannot use native driver for layout props!
}).start();

<Animated.View style={{ width: widthAnim }} />
```

**Trick:** To "move" a view without affecting layout, use `transform: translateX/Y`. To change actual layout dimensions (e.g., accordion expand), use `LayoutAnimation` or `useNativeDriver: false`.

---

### Q121. How do you implement skeleton loading screens?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** UX Patterns

**Answer:**
Skeleton screens show placeholder shapes while content loads, giving a perceived performance boost compared to spinners.

**Code:**
```js
// Shimmer animation
const ShimmerPlaceholder = ({ width, height, borderRadius = 4 }) => {
  const shimmerAnim = useRef(new Animated.Value(0)).current;

  useEffect(() => {
    Animated.loop(
      Animated.sequence([
        Animated.timing(shimmerAnim, { toValue: 1, duration: 1000, useNativeDriver: true }),
        Animated.timing(shimmerAnim, { toValue: 0, duration: 1000, useNativeDriver: true }),
      ])
    ).start();
  }, []);

  const opacity = shimmerAnim.interpolate({
    inputRange: [0, 1],
    outputRange: [0.3, 0.7],
  });

  return (
    <Animated.View
      style={{
        width,
        height,
        borderRadius,
        backgroundColor: '#E0E0E0',
        opacity,
      }}
    />
  );
};

// Skeleton screen for a user card
const UserCardSkeleton = () => (
  <View style={styles.card}>
    <ShimmerPlaceholder width={50} height={50} borderRadius={25} /> {/* avatar */}
    <View style={{ marginLeft: 12 }}>
      <ShimmerPlaceholder width={120} height={14} />
      <View style={{ height: 6 }} />
      <ShimmerPlaceholder width={80} height={12} />
    </View>
  </View>
);

// Usage
const UserList = () => {
  const { data, loading } = useApi('/users');

  if (loading) {
    return (
      <FlatList
        data={[1, 2, 3, 4, 5]}
        keyExtractor={i => i.toString()}
        renderItem={() => <UserCardSkeleton />}
      />
    );
  }
  return <FlatList data={data} renderItem={renderUser} />;
};
```

**Library option:** `react-native-skeleton-placeholder` or `react-content-loader` for more complex skeleton patterns.

---

### Q122. What is `TouchableWithoutFeedback` and when should you use it?

**Difficulty:** 🟢 Easy | **Frequency:** Medium | **Category:** Components

**Answer:**
`TouchableWithoutFeedback` is a wrapper that detects taps without any visual feedback (no opacity change, no highlight). It must wrap exactly one child component.

**Use cases:**
- Dismiss keyboard when tapping outside inputs
- Wrap a custom component that handles its own visual feedback
- Accessibility-only interaction areas

```js
import { TouchableWithoutFeedback, Keyboard } from 'react-native';

// Dismiss keyboard on background tap
<TouchableWithoutFeedback onPress={Keyboard.dismiss} accessible={false}>
  <View style={{ flex: 1 }}>
    <TextInput />
    <TextInput />
  </View>
</TouchableWithoutFeedback>

// Close modal on overlay tap
<TouchableWithoutFeedback onPress={onClose}>
  <View style={styles.overlay}>
    {/* Stop propagation for content inside */}
    <TouchableWithoutFeedback>
      <View style={styles.modal}>
        <Text>Modal content</Text>
      </View>
    </TouchableWithoutFeedback>
  </View>
</TouchableWithoutFeedback>
```

**Prefer `Pressable` in modern code** — it's more flexible. Use `TouchableWithoutFeedback` specifically when you need the "no visual feedback" behaviour explicitly.

---

### Q123. How do you implement pull-to-reveal (collapsible header) with scroll?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Animations + UX

**Answer:**
```js
import Animated, {
  useSharedValue,
  useAnimatedScrollHandler,
  useAnimatedStyle,
  interpolate,
  Extrapolation,
} from 'react-native-reanimated';

const HEADER_HEIGHT = 80;

const CollapsibleHeaderScreen = () => {
  const scrollY = useSharedValue(0);

  const scrollHandler = useAnimatedScrollHandler((event) => {
    scrollY.value = event.contentOffset.y;
  });

  const headerStyle = useAnimatedStyle(() => {
    const height = interpolate(
      scrollY.value,
      [0, HEADER_HEIGHT],
      [HEADER_HEIGHT, 0],
      Extrapolation.CLAMP
    );
    const opacity = interpolate(
      scrollY.value,
      [0, HEADER_HEIGHT / 2],
      [1, 0],
      Extrapolation.CLAMP
    );
    return { height, opacity };
  });

  return (
    <View style={{ flex: 1 }}>
      <Animated.View style={[styles.header, headerStyle]}>
        <Text style={styles.headerTitle}>My App</Text>
      </Animated.View>

      <Animated.FlatList
        onScroll={scrollHandler}
        scrollEventThrottle={16}
        data={data}
        renderItem={renderItem}
        keyExtractor={keyExtractor}
      />
    </View>
  );
};
```

---

### Q124. What is the React Native accessibility API?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Accessibility

**Answer:**
React Native has a comprehensive accessibility API that maps to iOS VoiceOver and Android TalkBack.

**Key props:**
```js
<Pressable
  accessible={true}                          // marks as accessible (default: true for interactive)
  accessibilityLabel="Submit button"         // read by screen reader
  accessibilityHint="Double tap to submit"   // additional context
  accessibilityRole="button"                 // semantic role: button, link, header, image, etc.
  accessibilityState={{ disabled: false, selected: false }}
  onAccessibilityTap={handlePress}           // iOS double-tap equivalent
>
  <Text>Submit</Text>
</Pressable>

// Group children under a single accessible element
<View accessible={true} accessibilityLabel="Price: 499 rupees, In stock">
  <Text>₹499</Text>
  <Text>In stock</Text>
</View>

// Announce dynamic changes
import { AccessibilityInfo } from 'react-native';
AccessibilityInfo.announceForAccessibility('Form submitted successfully');

// Check if screen reader is enabled
const isEnabled = await AccessibilityInfo.isScreenReaderEnabled();
```

**Follow-up:** How do you test accessibility in React Native? → iOS: enable VoiceOver in Simulator (Cmd+F5). Android: enable TalkBack in emulator settings. Automated: use `accessibilityLabel` in RNTL test queries.

---

### Q125. How do you prevent memory leaks in React Native?

**Difficulty:** 🔴 Hard | **Frequency:** Very High | **Category:** Performance

**Answer:**
Memory leaks occur when resources are not released after a component unmounts. Common sources and fixes:

**1. State update after unmount:**
```js
// ❌ Leak — setData called after unmount
useEffect(() => {
  api.get('/data').then(data => setData(data));
}, []);

// ✅ Fix — cancel flag or AbortController
useEffect(() => {
  let active = true;
  const controller = new AbortController();

  api.get('/data', { signal: controller.signal })
    .then(data => { if (active) setData(data); })
    .catch(err => { if (!controller.signal.aborted) console.error(err); });

  return () => { active = false; controller.abort(); };
}, []);
```

**2. Event listener not removed:**
```js
// ❌ Leak
useEffect(() => {
  Keyboard.addListener('keyboardDidShow', handler);
}, []);

// ✅ Fix
useEffect(() => {
  const sub = Keyboard.addListener('keyboardDidShow', handler);
  return () => sub.remove();
}, []);
```

**3. Interval/timeout not cleared:**
```js
// ✅ Always clear
useEffect(() => {
  const id = setInterval(tick, 1000);
  return () => clearInterval(id);
}, []);
```

**4. WebSocket not closed:**
```js
useEffect(() => {
  const ws = new WebSocket(url);
  return () => ws.close();
}, []);
```

**5. Animated loops not stopped:**
```js
useEffect(() => {
  const anim = Animated.loop(...);
  anim.start();
  return () => anim.stop();
}, []);
```

**Detection:** Use Flipper Memory plugin or Android Studio's Memory Profiler to detect growing heap that doesn't GC.

---

### Q126. What is `React.createRef` vs `useRef`?

**Difficulty:** 🟢 Easy | **Frequency:** Medium | **Category:** Hooks

**Answer:**

| | `useRef` | `createRef` |
|--|---------|-------------|
| Used in | Function components | Class components (or dynamic arrays) |
| Persistence | Same object across renders | New object on every render |
| Recommended | ✅ Yes (function components) | Only for class components |

```js
// useRef — persists the same {current} object
const ref = useRef(null); // always same object, current is mutable

// createRef — creates a new ref object every time
class MyClass extends React.Component {
  constructor(props) {
    super(props);
    this.inputRef = createRef(); // created once in constructor
  }
}

// Dynamic ref list (where createRef is still useful)
const MyList = ({ items }) => {
  const refs = useRef(items.map(() => createRef()));
  // ...
};
```

---

### Q127. What is reconciliation key strategy for animated lists?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Performance

**Answer:**
When items are added/removed from an animated list, the `key` prop determines whether React reuses or remounts components — directly affecting animation behaviour.

```js
// Scenario: removing an item with a fade-out animation

// ❌ Using index as key — React reuses wrong component, animation breaks
<FlatList keyExtractor={(_, index) => index.toString()} />

// ✅ Using stable ID — React correctly identifies the removed item
<FlatList keyExtractor={(item) => item.id.toString()} />

// Animated list with enter/exit animations using react-native-reanimated
const AnimatedItem = ({ item, onRemove }) => {
  const opacity = useSharedValue(0);
  const height = useSharedValue(0);

  // Fade in on mount
  useEffect(() => {
    opacity.value = withTiming(1, { duration: 300 });
    height.value = withTiming(ITEM_HEIGHT, { duration: 300 });
  }, []);

  const handleRemove = () => {
    // Animate out before removing from data
    opacity.value = withTiming(0, { duration: 200 });
    height.value = withTiming(0, { duration: 200 }, () => {
      runOnJS(onRemove)(item.id); // call on JS thread after animation
    });
  };

  const style = useAnimatedStyle(() => ({
    opacity: opacity.value,
    height: height.value,
    overflow: 'hidden',
  }));

  return (
    <Animated.View style={style}>
      <Pressable onPress={handleRemove}>
        <Text>{item.title}</Text>
      </Pressable>
    </Animated.View>
  );
};
```

---

### Q128. How do you implement infinite carousel / auto-scroll in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** UI Components

**Answer:**
```js
import React, { useRef, useEffect, useState } from 'react';
import { FlatList, Dimensions } from 'react-native';

const { width: SCREEN_WIDTH } = Dimensions.get('window');

const AutoScrollCarousel = ({ items, autoScrollInterval = 3000 }) => {
  const flatListRef = useRef(null);
  const [currentIndex, setCurrentIndex] = useState(0);
  const currentIndexRef = useRef(0); // use ref to avoid stale closure in interval

  useEffect(() => {
    const id = setInterval(() => {
      const nextIndex = (currentIndexRef.current + 1) % items.length;
      flatListRef.current?.scrollToIndex({ index: nextIndex, animated: true });
      currentIndexRef.current = nextIndex;
      setCurrentIndex(nextIndex);
    }, autoScrollInterval);

    return () => clearInterval(id);
  }, [items.length, autoScrollInterval]);

  const onMomentumScrollEnd = (e) => {
    const index = Math.round(e.nativeEvent.contentOffset.x / SCREEN_WIDTH);
    currentIndexRef.current = index;
    setCurrentIndex(index);
  };

  return (
    <View>
      <FlatList
        ref={flatListRef}
        data={items}
        horizontal
        pagingEnabled
        showsHorizontalScrollIndicator={false}
        keyExtractor={(_, i) => i.toString()}
        renderItem={({ item }) => (
          <View style={{ width: SCREEN_WIDTH }}>
            <Image source={{ uri: item.image }} style={{ width: SCREEN_WIDTH, height: 200 }} />
          </View>
        )}
        onMomentumScrollEnd={onMomentumScrollEnd}
        getItemLayout={(_, index) => ({
          length: SCREEN_WIDTH, offset: SCREEN_WIDTH * index, index
        })}
      />
      {/* Dots indicator */}
      <View style={styles.dotsRow}>
        {items.map((_, i) => (
          <View key={i} style={[styles.dot, i === currentIndex && styles.activeDot]} />
        ))}
      </View>
    </View>
  );
};
```

---

### Q129. What is `runOnJS` in Reanimated 2?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Reanimated

**Answer:**
`runOnJS` is a Reanimated utility that calls a JavaScript function from a worklet (UI thread code). Since worklets run on the UI thread, they cannot directly call JS functions — `runOnJS` bridges the gap.

**Code:**
```js
import { runOnJS } from 'react-native-reanimated';

const MyComponent = ({ onSwipeComplete }) => {
  const translateX = useSharedValue(0);

  const gesture = Gesture.Pan()
    .onEnd((event) => {
      // We're on the UI thread here (worklet context)
      if (event.translationX > 150) {
        // ✅ Call JS function from UI thread
        runOnJS(onSwipeComplete)('right');
        // ❌ Cannot call onSwipeComplete directly — it's a JS function
      }
      translateX.value = withSpring(0);
    });

  // Another common pattern: update React state from a worklet
  const [direction, setDirection] = useState('none');

  const panGesture = Gesture.Pan()
    .onUpdate((e) => {
      runOnJS(setDirection)(e.translationX > 0 ? 'right' : 'left');
    });
};

// runOnUI — opposite direction: call a worklet from JS thread
import { runOnUI } from 'react-native-reanimated';

const triggerAnimation = () => {
  runOnUI(() => {
    'worklet';
    sharedValue.value = withSpring(100); // runs on UI thread
  })();
};
```

**Follow-up:** Is `runOnJS` expensive? → It schedules a message from the UI thread back to the JS thread — slightly more expensive than pure worklet calls. Minimise `runOnJS` calls in hot paths (like `onUpdate` — prefer `onEnd`).

---

### Q130. How do you handle biometric authentication in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Security / Native APIs

**Answer:**
Use `react-native-biometrics` or `react-native-keychain` for biometric (Face ID / Fingerprint) authentication.

**Code:**
```js
import ReactNativeBiometrics, { BiometryTypes } from 'react-native-biometrics';

const rnBiometrics = new ReactNativeBiometrics();

// Check what biometrics are available
const checkBiometrics = async () => {
  const { available, biometryType } = await rnBiometrics.isSensorAvailable();

  if (available && biometryType === BiometryTypes.TouchID) {
    console.log('TouchID available');
  } else if (available && biometryType === BiometryTypes.FaceID) {
    console.log('FaceID available');
  } else if (available && biometryType === BiometryTypes.Biometrics) {
    console.log('Android fingerprint available');
  } else {
    console.log('Biometrics not available');
  }
};

// Authenticate
const authenticateWithBiometrics = async () => {
  try {
    const { success, error } = await rnBiometrics.simplePrompt({
      promptMessage: 'Confirm fingerprint',
      cancelButtonText: 'Use PIN instead',
    });

    if (success) {
      // Retrieve stored token from Keychain
      const credentials = await Keychain.getGenericPassword();
      if (credentials) {
        await dispatch(loginWithToken(credentials.password));
      }
    }
  } catch (error) {
    console.log('Biometric auth failed:', error);
    // Fall back to PIN/password
    navigation.navigate('PINLogin');
  }
};

// Store token securely after normal login (to be retrieved with biometrics later)
const storeTokenForBiometrics = async (token) => {
  await Keychain.setGenericPassword('user', token, {
    accessControl: Keychain.ACCESS_CONTROL.BIOMETRY_ANY,
    accessible: Keychain.ACCESSIBLE.WHEN_UNLOCKED,
  });
};
```

**Where this applies to your work:** In the Debt Relief India fintech app, biometric authentication could gate access to sensitive financial data — the token is stored in the Keychain (encrypted, hardware-backed on supported devices) and only retrieved after successful biometric verification.

---

## Section 5: Styling & Layout (Q131–Q175)

---

### Q131. What is the default flex direction in React Native vs CSS?

**Difficulty:** 🟢 Easy | **Frequency:** High | **Category:** Flexbox

**Answer:**
This is one of the most important differences between React Native and web CSS:

| | React Native | Web CSS |
|--|-------------|---------|
| Default `flexDirection` | `column` (vertical) | `row` (horizontal) |
| Default `alignContent` | `flex-start` | `stretch` |
| Default `flexShrink` | `0` | `1` |

In React Native, every `View` is a flex container by default, and items stack **vertically** unless you set `flexDirection: 'row'`.

```js
// React Native — items stack top-to-bottom by default
<View>
  <Text>First</Text>   {/* top */}
  <Text>Second</Text>  {/* below First */}
</View>

// To lay items left-to-right
<View style={{ flexDirection: 'row' }}>
  <Text>Left</Text>
  <Text>Right</Text>
</View>

// Common pattern — horizontal row with space between
<View style={{ flexDirection: 'row', justifyContent: 'space-between', alignItems: 'center' }}>
  <Text>Label</Text>
  <Switch value={isOn} onValueChange={setIsOn} />
</View>
```

**Follow-up:** What does `alignItems` control vs `justifyContent`?
→ `justifyContent` controls alignment along the **main axis** (the direction of `flexDirection`). `alignItems` controls alignment along the **cross axis** (perpendicular). In column direction: `justifyContent` = vertical, `alignItems` = horizontal.

---

### Q132. What are all the `justifyContent` values and what does each do?

**Difficulty:** 🟢 Easy | **Frequency:** High | **Category:** Flexbox

**Answer:**
`justifyContent` distributes children along the **main axis**.

```js
// flex-start (default) — children at the start
<View style={{ justifyContent: 'flex-start' }}>
  {/* items at top (in column) or left (in row) */}
</View>

// flex-end — children at the end
<View style={{ justifyContent: 'flex-end' }}>
  {/* items at bottom or right */}
</View>

// center — children centered
<View style={{ flex: 1, justifyContent: 'center' }}>
  {/* items centered vertically (column) */}
</View>

// space-between — first at start, last at end, equal space between
<View style={{ flexDirection: 'row', justifyContent: 'space-between' }}>
  <Icon name="back" />
  <Text>Title</Text>
  <Icon name="menu" />
</View>

// space-around — equal space around each item (half-space at edges)
<View style={{ flexDirection: 'row', justifyContent: 'space-around' }}>

// space-evenly — equal space between AND at edges
<View style={{ flexDirection: 'row', justifyContent: 'space-evenly' }}>
```

**Visual guide:**
```
flex-start:   [A][B][C]___
flex-end:     ___[A][B][C]
center:       _[A][B][C]__  (centered)
space-between:[A]__[B]__[C]
space-around: _[A]_[B]_[C]_
space-evenly: _[A]__[B]__[C]_
```

---

### Q133. What are all the `alignItems` values?

**Difficulty:** 🟢 Easy | **Frequency:** High | **Category:** Flexbox

**Answer:**
`alignItems` aligns children along the **cross axis** (perpendicular to `flexDirection`).

```js
// stretch (default) — children stretch to fill cross-axis width/height
<View style={{ alignItems: 'stretch' }}>
  {/* each child fills the full width (in column direction) */}
</View>

// flex-start — children at cross-axis start (left edge in column direction)
<View style={{ alignItems: 'flex-start' }}>
  {/* children only as wide as their content */}
</View>

// flex-end — children at cross-axis end (right edge)
<View style={{ alignItems: 'flex-end' }}>

// center — children centered on cross axis
<View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
  {/* perfectly centered on screen */}
</View>

// baseline — children aligned by text baseline (rarely used in RN)
<View style={{ flexDirection: 'row', alignItems: 'baseline' }}>
  <Text style={{ fontSize: 24 }}>Big</Text>
  <Text style={{ fontSize: 12 }}>small</Text>
  {/* baselines aligned despite different sizes */}
</View>
```

**`alignSelf`** — overrides `alignItems` for a single child:
```js
<View style={{ alignItems: 'flex-start' }}>
  <Text>Left aligned</Text>
  <Text style={{ alignSelf: 'flex-end' }}>I'm right aligned!</Text>
  <Text style={{ alignSelf: 'center' }}>I'm centered!</Text>
</View>
```

---

### Q134. What is `flexWrap` and when do you use it?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Flexbox

**Answer:**
`flexWrap` controls whether flex children wrap to a new line when they overflow the container.

```js
// nowrap (default) — children never wrap, overflow clipped or squeezed
<View style={{ flexDirection: 'row', flexWrap: 'nowrap' }}>
  <Tag /><Tag /><Tag /><Tag />  {/* might overflow */}
</View>

// wrap — children wrap to next line
<View style={{ flexDirection: 'row', flexWrap: 'wrap' }}>
  <Tag /><Tag /><Tag /><Tag />  {/* wraps to next row */}
</View>

// wrap-reverse — wraps but in reverse direction
<View style={{ flexDirection: 'row', flexWrap: 'wrap-reverse' }}>

// Real use case: tag/chip cloud
const TagCloud = ({ tags }) => (
  <View style={{ flexDirection: 'row', flexWrap: 'wrap', gap: 8 }}>
    {tags.map(tag => (
      <View key={tag} style={styles.tag}>
        <Text>{tag}</Text>
      </View>
    ))}
  </View>
);
```

**`alignContent`** — works with `flexWrap` to align wrapped rows/columns as a group (same values as `justifyContent` but for the cross axis of multi-line flex containers). Only has effect when wrapping.

---

### Q135. What is `gap`, `rowGap`, and `columnGap` in React Native?

**Difficulty:** 🟢 Easy | **Frequency:** Medium | **Category:** Flexbox

**Answer:**
React Native added `gap` support (from RN 0.71+), matching the CSS gap property. It sets spacing between flex/grid children without needing margin hacks.

```js
// gap — uniform spacing between all children
<View style={{ flexDirection: 'row', flexWrap: 'wrap', gap: 12 }}>
  <Card /><Card /><Card />
</View>

// rowGap — spacing between rows only
<View style={{ flexDirection: 'row', flexWrap: 'wrap', rowGap: 16, columnGap: 8 }}>
  <Card /><Card /><Card />
</View>

// Before gap support — margin hack (still works, but gap is cleaner)
<View style={{ flexDirection: 'row', flexWrap: 'wrap', margin: -6 }}>
  {items.map(item => (
    <View key={item.id} style={{ margin: 6 }}>
      <Card item={item} />
    </View>
  ))}
</View>
```

**Tip:** Check `gap` support in your target RN version. For <0.71, use the margin hack or a library like `react-native-gap`.

---

### Q136. What is `position: 'absolute'` in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Layout

**Answer:**
`position: 'absolute'` removes an element from the normal flex flow and positions it relative to its **nearest ancestor with `position: 'relative'`** (or the root if none). In React Native, the default position is `'relative'`.

```js
// Overlay badge on an icon
const IconWithBadge = ({ count }) => (
  <View style={{ position: 'relative', width: 40, height: 40 }}>
    <Icon name="bell" size={32} />
    {count > 0 && (
      <View style={{
        position: 'absolute',
        top: -4,
        right: -4,
        backgroundColor: 'red',
        borderRadius: 10,
        width: 20,
        height: 20,
        alignItems: 'center',
        justifyContent: 'center',
      }}>
        <Text style={{ color: 'white', fontSize: 11 }}>{count}</Text>
      </View>
    )}
  </View>
);

// Full-screen overlay
<View style={{ flex: 1 }}>
  <MainContent />
  <View style={{
    position: 'absolute',
    top: 0, left: 0, right: 0, bottom: 0,
    backgroundColor: 'rgba(0,0,0,0.5)',
    justifyContent: 'center',
    alignItems: 'center',
  }}>
    <LoadingSpinner />
  </View>
</View>

// Floating Action Button (FAB)
<View style={{ flex: 1 }}>
  <FlatList data={data} renderItem={renderItem} />
  <Pressable style={{
    position: 'absolute',
    bottom: 24,
    right: 24,
    width: 56,
    height: 56,
    borderRadius: 28,
    backgroundColor: '#6200EE',
    justifyContent: 'center',
    alignItems: 'center',
  }}>
    <Icon name="add" color="white" size={28} />
  </Pressable>
</View>
```

**Follow-up:** Does `position: 'absolute'` respect safe area? → No. Absolutely positioned elements don't get pushed by the safe area. Use `useSafeAreaInsets` and add the inset manually to `bottom`/`top` values.

---

### Q137. What is `zIndex` in React Native and how does it work?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Layout

**Answer:**
`zIndex` controls the stacking order of components. Higher `zIndex` renders on top of lower `zIndex`. By default, later siblings in the JSX tree render on top of earlier siblings.

```js
// Without zIndex — later element (B) renders on top of A
<View style={{ position: 'relative' }}>
  <View style={{ position: 'absolute', top: 0, left: 0, width: 100, height: 100, backgroundColor: 'blue' }} />  {/* A — bottom */}
  <View style={{ position: 'absolute', top: 20, left: 20, width: 100, height: 100, backgroundColor: 'red' }} /> {/* B — on top */}
</View>

// With zIndex — A on top despite being first in JSX
<View style={{ position: 'relative' }}>
  <View style={{ position: 'absolute', zIndex: 10, backgroundColor: 'blue', ... }} />  {/* A — on top */}
  <View style={{ position: 'absolute', zIndex: 1,  backgroundColor: 'red', ... }} />   {/* B — behind */}
</View>

// Common use cases
const styles = StyleSheet.create({
  modal:   { zIndex: 1000 },
  tooltip: { zIndex: 500 },
  header:  { zIndex: 100 },
  content: { zIndex: 1 },
});
```

**Android caveat:** On Android, `zIndex` requires `elevation` to work correctly with shadows and in some stacking contexts. If `zIndex` isn't working on Android, add `elevation` to the same component.

```js
// Android — combine zIndex with elevation
<View style={{ zIndex: 10, elevation: 10 }}>
```

---

### Q138. How do you create a responsive layout in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** Very High | **Category:** Responsive Design

**Answer:**
React Native has no CSS media queries. Instead:

**1. `useWindowDimensions` — most reliable, updates on rotation:**
```js
import { useWindowDimensions } from 'react-native';

const ResponsiveGrid = ({ items }) => {
  const { width } = useWindowDimensions();

  const numColumns = width > 768 ? 3 : width > 480 ? 2 : 1;
  const itemWidth = (width - (numColumns + 1) * 16) / numColumns;

  return (
    <FlatList
      data={items}
      numColumns={numColumns}
      key={numColumns} // key change forces FlatList re-render when columns change
      renderItem={({ item }) => (
        <View style={{ width: itemWidth, margin: 8 }}>
          <Card item={item} />
        </View>
      )}
    />
  );
};
```

**2. Percentage-based dimensions (limited support):**
```js
// Supported in RN for width/height as strings
<View style={{ width: '100%', height: '50%' }} />
// Percentages are relative to parent — not viewport
```

**3. `Dimensions` API with event listener:**
```js
const [dimensions, setDimensions] = useState(Dimensions.get('window'));

useEffect(() => {
  const sub = Dimensions.addEventListener('change', ({ window }) => {
    setDimensions(window);
  });
  return () => sub.remove();
}, []);
```

**4. `react-native-responsive-screen` library:**
```js
import { widthPercentageToDP as wp, heightPercentageToDP as hp } from 'react-native-responsive-screen';

<View style={{ width: wp('80%'), height: hp('10%') }} />
```

**Follow-up:** How did you handle responsive design in your 100-screen ERP app? → Used `useWindowDimensions` hook combined with breakpoint constants, and tested on multiple screen sizes including tablets used in warehouse management.

---

### Q139. What is `PixelRatio` and why does it matter?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Responsive Design

**Answer:**
`PixelRatio` provides access to the device's pixel density. React Native uses logical pixels (dp/pt), but physical pixels vary by device. A 1dp border on a 3x density screen is rendered as 3 physical pixels.

```js
import { PixelRatio } from 'react-native';

// Get device pixel ratio (1, 1.5, 2, 3, 3.5)
const ratio = PixelRatio.get(); // e.g., 3 on iPhone 13

// Convert dp to px
const px = PixelRatio.getPixelSizeForLayoutSize(16); // 16dp → 48px on 3x

// Get font scale (user's text size setting)
const fontScale = PixelRatio.getFontScale(); // 1.0 = normal, 1.3 = large text

// Hairline width — thinnest possible line on device
const hairline = StyleSheet.hairlineWidth; // 1 / PixelRatio.get()

// Use hairline for subtle dividers
<View style={{ height: StyleSheet.hairlineWidth, backgroundColor: '#ccc' }} />

// Responsive font size respecting user's font scale
const scaledFontSize = (size) => size / PixelRatio.getFontScale();
// Prevents huge text when user sets system font to "Extra Large"
```

**Real use case:** Loading the right image resolution based on device DPI:
```js
const getImageSource = (uri) => {
  const ratio = PixelRatio.get();
  if (ratio >= 3) return `${uri}@3x.png`;
  if (ratio >= 2) return `${uri}@2x.png`;
  return `${uri}.png`;
};
```

---

### Q140. How do you handle font scaling and accessibility text sizes?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Accessibility + Styling

**Answer:**
Users can set system font size to large/extra-large for accessibility. React Native respects this by default — `fontSize: 16` might render larger depending on the system setting. Use `allowFontScaling` and `maxFontSizeMultiplier` to control this.

```js
// allowFontScaling={false} — disable system font scaling (use sparingly)
<Text allowFontScaling={false} style={{ fontSize: 12 }}>
  Tab label  {/* tabs/navigation labels should stay fixed */}
</Text>

// maxFontSizeMultiplier — cap how large text can scale
<Text maxFontSizeMultiplier={1.5} style={{ fontSize: 16 }}>
  Body text  {/* won't scale beyond 24px even at "Extra Large" system font */}
</Text>

// Global override via Text defaults (React Native 0.72+)
// In your root component:
Text.defaultProps = Text.defaultProps || {};
Text.defaultProps.maxFontSizeMultiplier = 1.3;

// Check current font scale
import { PixelRatio } from 'react-native';
const fontScale = PixelRatio.getFontScale(); // 1.0, 1.15, 1.3, 1.5...

// Dynamic font size that respects user preference but has a ceiling
const responsiveFontSize = (size) => {
  const scale = PixelRatio.getFontScale();
  return Math.min(size * scale, size * 1.3); // max 30% larger
};
```

**Follow-up:** Should you always disable font scaling? → No. Disabling `allowFontScaling` breaks accessibility for visually impaired users who rely on large text. Use `maxFontSizeMultiplier` to allow scaling with a reasonable cap instead.

---

### Q141. What is the difference between `width: '100%'` and `alignSelf: 'stretch'`?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Layout

**Answer:**
Both make a component fill its parent's width, but they work differently:

```js
// width: '100%' — explicitly sets width to 100% of parent
// Works as expected when parent has a defined width
<View style={{ width: 200 }}>
  <Text style={{ width: '100%' }}>Fills 200px</Text>
</View>

// alignSelf: 'stretch' — stretch along the parent's cross axis
// Default behaviour in flex column (alignItems: 'stretch' is the default)
<View style={{ flexDirection: 'column' }}> {/* default */}
  <Text>Full width by default (stretch)</Text>
</View>

// Key difference — percentages are relative to PARENT, not screen
<View style={{ width: 150 }}>
  <View style={{ width: '50%' }}>  {/* 75px, not 50% of screen */}
    <Text>75px wide</Text>
  </View>
</View>

// When width: '100%' breaks:
// If parent has no defined width (flex auto-sized), '100%' may not work
// alignSelf: 'stretch' or flex: 1 is more reliable in those cases

// When to use which:
// - width: '100%' → parent has explicit width, or you're in a ScrollView
// - flex: 1 → fill remaining space in the main axis
// - alignSelf: 'stretch' → fill cross axis (width in column, height in row)
```

---

### Q142. How do you build a two-column grid layout in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Layout

**Answer:**
```js
// Method 1: FlatList numColumns (most common, handles virtualisation)
const ProductGrid = ({ products }) => {
  const { width } = useWindowDimensions();
  const NUM_COLUMNS = 2;
  const PADDING = 16;
  const GAP = 12;
  const itemWidth = (width - PADDING * 2 - GAP) / NUM_COLUMNS;

  const renderItem = useCallback(({ item }) => (
    <View style={{ width: itemWidth, marginBottom: GAP }}>
      <ProductCard product={item} />
    </View>
  ), [itemWidth]);

  return (
    <FlatList
      data={products}
      numColumns={NUM_COLUMNS}
      keyExtractor={item => item.id.toString()}
      renderItem={renderItem}
      columnWrapperStyle={{ gap: GAP, paddingHorizontal: PADDING }}
      contentContainerStyle={{ paddingTop: PADDING }}
    />
  );
};

// Method 2: Manual flexWrap (for small fixed lists)
<View style={{ flexDirection: 'row', flexWrap: 'wrap', padding: 8 }}>
  {products.map(product => (
    <View key={product.id} style={{ width: '50%', padding: 8 }}>
      <ProductCard product={product} />
    </View>
  ))}
</View>

// Method 3: Two FlatLists side by side (for independent scrolling columns)
<View style={{ flex: 1, flexDirection: 'row' }}>
  <FlatList style={{ flex: 1 }} data={leftColumnData} renderItem={renderItem} />
  <FlatList style={{ flex: 1 }} data={rightColumnData} renderItem={renderItem} />
</View>
```

---

### Q143. What is `overflow` in React Native and how does it work?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Layout

**Answer:**
`overflow` controls whether content that extends beyond a component's boundaries is visible or clipped.

```js
// visible (default on iOS) — content extends beyond bounds
<View style={{ width: 100, height: 100, overflow: 'visible' }}>
  <View style={{ width: 200, height: 50, backgroundColor: 'blue' }} />
  {/* blue box extends 100px beyond parent on right */}
</View>

// hidden — clips content to component bounds
<View style={{ width: 100, height: 100, overflow: 'hidden', borderRadius: 50 }}>
  <Image source={avatar} style={{ width: 100, height: 100 }} />
  {/* image is clipped to circle — essential for round avatars */}
</View>

// scroll — makes content scrollable (mostly use ScrollView instead)
<View style={{ height: 200, overflow: 'scroll' }}>

// Important Android behaviour:
// On Android, overflow: 'hidden' is required for borderRadius to clip children
// On iOS, it clips by default
<View style={{
  borderRadius: 16,
  overflow: 'hidden', // required on Android to clip child Image to rounded corners
}}>
  <Image source={thumbnail} style={{ width: '100%', height: 200 }} />
</View>
```

**Follow-up:** Why is `overflow: 'hidden'` required on Android for `borderRadius`? → Android renders each view independently and doesn't automatically clip children to the parent's rounded corners. `overflow: 'hidden'` explicitly clips. iOS composites views differently and clips by default.

---

### Q144. How do you create a circular component in React Native?

**Difficulty:** 🟢 Easy | **Frequency:** High | **Category:** Styling

**Answer:**
Set `borderRadius` to exactly half the `width` and `height`. Both must be equal (square) and `overflow: 'hidden'` for clipping children.

```js
// Circle avatar
const Avatar = ({ uri, size = 48 }) => (
  <View style={{
    width: size,
    height: size,
    borderRadius: size / 2,
    overflow: 'hidden',           // clips image to circle on Android
    backgroundColor: '#E0E0E0',   // fallback colour
  }}>
    <Image source={{ uri }} style={{ width: size, height: size }} />
  </View>
);

// Circle with border
<View style={{
  width: 60,
  height: 60,
  borderRadius: 30,
  borderWidth: 2,
  borderColor: '#6200EE',
  justifyContent: 'center',
  alignItems: 'center',
}}>
  <Text style={{ fontWeight: '600' }}>DK</Text>
</View>

// Online indicator dot
<View style={{
  width: 12,
  height: 12,
  borderRadius: 6,
  backgroundColor: '#4CAF50',
  position: 'absolute',
  bottom: 0,
  right: 0,
  borderWidth: 2,
  borderColor: 'white',
}} />
```

---

### Q145. What is `borderRadius` shorthand and individual corner control?

**Difficulty:** 🟢 Easy | **Frequency:** Medium | **Category:** Styling

**Answer:**
```js
// Uniform border radius
<View style={{ borderRadius: 12 }} />

// Individual corners
<View style={{
  borderTopLeftRadius: 16,
  borderTopRightRadius: 16,
  borderBottomLeftRadius: 4,
  borderBottomRightRadius: 4,
}} />

// Common patterns

// Card with only top corners rounded (for bottom sheets / modals)
<View style={{
  borderTopLeftRadius: 24,
  borderTopRightRadius: 24,
  backgroundColor: 'white',
}}>
  <BottomSheetContent />
</View>

// Pill button
<View style={{
  borderRadius: 999,   // large number → always pill regardless of height
  paddingVertical: 12,
  paddingHorizontal: 24,
  backgroundColor: '#6200EE',
}}>
  <Text style={{ color: 'white' }}>Submit</Text>
</View>

// Chat bubble — one corner flat
const MessageBubble = ({ isOwn, children }) => (
  <View style={{
    borderRadius: 16,
    borderBottomRightRadius: isOwn ? 4 : 16,
    borderBottomLeftRadius: isOwn ? 16 : 4,
    backgroundColor: isOwn ? '#6200EE' : '#F0F0F0',
    padding: 12,
    maxWidth: '75%',
    alignSelf: isOwn ? 'flex-end' : 'flex-start',
  }}>
    {children}
  </View>
);
```

---

### Q146. How do you add shadows in React Native (iOS vs Android)?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Styling

**Answer:**
Shadows are implemented differently on each platform:

```js
// iOS — shadow props
const iosCardStyle = {
  shadowColor: '#000',
  shadowOffset: { width: 0, height: 4 },
  shadowOpacity: 0.15,
  shadowRadius: 8,
  backgroundColor: 'white', // REQUIRED — shadow won't show without bg color
};

// Android — elevation (0–24)
const androidCardStyle = {
  elevation: 4,             // depth of shadow
  backgroundColor: 'white', // REQUIRED
};

// Cross-platform card style
const cardStyle = {
  backgroundColor: 'white',
  borderRadius: 12,
  // iOS
  shadowColor: '#000',
  shadowOffset: { width: 0, height: 2 },
  shadowOpacity: 0.1,
  shadowRadius: 8,
  // Android
  elevation: 4,
};

// Platform.select for finer control
const shadowStyle = Platform.select({
  ios: {
    shadowColor: '#6200EE',
    shadowOffset: { width: 0, height: 6 },
    shadowOpacity: 0.3,
    shadowRadius: 10,
  },
  android: {
    elevation: 8,
  },
});
```

**Important limitations:**
- iOS shadow only works on views with `backgroundColor` set
- Android `elevation` adds shadow but also affects the view's Z-order and can interfere with `zIndex`
- Shadow color is ignored on Android below API 28; use `elevation` only
- Custom shadow shapes are not supported — only rectangular shadows (use `react-native-shadow-2` for custom shadows)

---

### Q147. How do you implement a gradient background in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Styling

**Answer:**
React Native does not support CSS gradients natively. Use `react-native-linear-gradient` (Expo: `expo-linear-gradient`).

```js
import LinearGradient from 'react-native-linear-gradient';

// Basic vertical gradient
<LinearGradient
  colors={['#6200EE', '#3700B3']}
  style={{ flex: 1 }}
>
  <Text style={{ color: 'white' }}>Gradient background</Text>
</LinearGradient>

// Horizontal gradient
<LinearGradient
  colors={['#FF6B6B', '#FFE66D']}
  start={{ x: 0, y: 0 }}
  end={{ x: 1, y: 0 }}
  style={styles.button}
>
  <Text>Gradient Button</Text>
</LinearGradient>

// Diagonal gradient
<LinearGradient
  colors={['#667EEA', '#764BA2']}
  start={{ x: 0, y: 0 }}   // top-left
  end={{ x: 1, y: 1 }}     // bottom-right
  style={styles.card}
/>

// Multi-stop gradient with locations
<LinearGradient
  colors={['#4c669f', '#3b5998', '#192f6a']}
  locations={[0, 0.5, 1]}  // position of each color stop (0-1)
  style={styles.container}
/>

// Gradient text (using MaskedView)
import MaskedView from '@react-native-masked-view/masked-view';

<MaskedView maskElement={<Text style={styles.title}>Gradient Text</Text>}>
  <LinearGradient colors={['#6200EE', '#FF6B6B']} start={{x:0,y:0}} end={{x:1,y:0}}>
    <Text style={[styles.title, { opacity: 0 }]}>Gradient Text</Text>
  </LinearGradient>
</MaskedView>
```

---

### Q148. What is `StyleSheet.flatten` and when do you use it?

**Difficulty:** 🟡 Medium | **Frequency:** Low | **Category:** Styling

**Answer:**
`StyleSheet.flatten` merges an array of styles (or a single style) into a plain JavaScript object. Useful when you need to read individual style values or merge styles programmatically.

```js
import { StyleSheet } from 'react-native';

const base = StyleSheet.create({ text: { fontSize: 16, color: 'black' } });
const override = { color: 'red', fontWeight: '600' };

// Flatten into a plain object
const merged = StyleSheet.flatten([base.text, override]);
// { fontSize: 16, color: 'red', fontWeight: '600' }

console.log(merged.color); // 'red'

// Use case: reading a style value from a ref
const Component = ({ style }) => {
  const flatStyle = StyleSheet.flatten(style);
  const bgColor = flatStyle?.backgroundColor || '#fff';
  // ...
};

// Use case: conditional style merging
const getTextStyle = (variant, isDisabled) =>
  StyleSheet.flatten([
    styles.base,
    variant === 'primary' && styles.primary,
    variant === 'secondary' && styles.secondary,
    isDisabled && styles.disabled,
  ]);
```

---

### Q149. How do you implement a sticky header in a ScrollView/FlatList?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Layout

**Answer:**
```js
// ScrollView — stickyHeaderIndices
<ScrollView
  stickyHeaderIndices={[0, 4]}  // indices of children that stick
>
  <View style={styles.stickyHeader}><Text>Stuck Header</Text></View>
  <Item /><Item /><Item />
  <View style={styles.stickyHeader}><Text>Another Stuck Header</Text></View>
  <Item /><Item />
</ScrollView>

// FlatList — renderSectionHeader on SectionList (stickySectionHeadersEnabled)
<SectionList
  sections={sections}
  stickySectionHeadersEnabled={true}   // true by default on iOS, false on Android
  renderSectionHeader={({ section }) => (
    <View style={styles.sectionHeader}>
      <Text>{section.title}</Text>
    </View>
  )}
  renderItem={({ item }) => <Row item={item} />}
/>

// FlatList — use ListHeaderComponent with scroll-driven animation
const headerY = useRef(new Animated.Value(0)).current;
const stickyTranslate = headerY.interpolate({
  inputRange: [0, HEADER_HEIGHT],
  outputRange: [0, -HEADER_HEIGHT],
  extrapolate: 'clamp',
});

<View style={{ flex: 1 }}>
  <Animated.View style={[styles.header, { transform: [{ translateY: stickyTranslate }] }]}>
    <Text>Sticky Header</Text>
  </Animated.View>
  <Animated.FlatList
    onScroll={Animated.event(
      [{ nativeEvent: { contentOffset: { y: headerY } } }],
      { useNativeDriver: true }
    )}
    scrollEventThrottle={16}
    data={data}
    renderItem={renderItem}
  />
</View>
```

---

### Q150. What is `contentContainerStyle` vs `style` on ScrollView/FlatList?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Layout

**Answer:**
This is a very common source of confusion:

- **`style`** — styles the outer scroll container (the view that clips content)
- **`contentContainerStyle`** — styles the inner content wrapper (the scrollable content area)

```js
// style: affects the scroll view's outer bounds
// contentContainerStyle: affects what's inside (adds padding to scrollable area)

// ✅ Correct — use contentContainerStyle for inner padding
<FlatList
  style={{ flex: 1, backgroundColor: '#f5f5f5' }}           // outer container
  contentContainerStyle={{ padding: 16, paddingBottom: 80 }} // inner content
  data={items}
  renderItem={renderItem}
/>

// ❌ Wrong — padding on style clips content at edges
<FlatList style={{ padding: 16 }}  />  // clips overscroll

// Common use: center content vertically when list is short
<FlatList
  contentContainerStyle={{ flexGrow: 1, justifyContent: 'center' }}
  data={emptyData}
  ListEmptyComponent={<EmptyState />}
/>
// flexGrow: 1 makes the content container fill available space
// justifyContent: 'center' centers the empty state

// Add safe area padding to bottom (for tab bar)
<ScrollView
  contentContainerStyle={{ paddingBottom: insets.bottom + 80 }}
>
```

**Rule:** Use `contentContainerStyle` for padding/alignment of scrollable content. Use `style` for the scroll view's own dimensions and background.

---

### Q151. How do you handle RTL (right-to-left) layouts for Arabic/Hebrew in React Native?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Internationalisation + Layout

**Answer:**
React Native has built-in RTL support that mirrors the layout when the device is in RTL mode.

```js
import { I18nManager } from 'react-native';

// Check if device is in RTL mode
const isRTL = I18nManager.isRTL; // true for Arabic, Hebrew, Farsi, Urdu

// Force RTL for development/testing (requires app restart)
I18nManager.forceRTL(true);

// RTL-aware styles — use 'start'/'end' instead of 'left'/'right'
const styles = StyleSheet.create({
  container: {
    // ❌ Hardcoded — won't flip in RTL
    paddingLeft: 16,
    textAlign: 'left',

    // ✅ RTL-aware — flips automatically
    paddingStart: 16,  // = paddingLeft in LTR, paddingRight in RTL
    paddingEnd: 8,     // = paddingRight in LTR, paddingLeft in RTL
  },
});

// Text alignment
<Text style={{ textAlign: 'left' }}>  {/* always left — doesn't flip */}
<Text style={{ writingDirection: 'rtl' }}> {/* explicit RTL text */}

// Conditional styles
<View style={{
  flexDirection: isRTL ? 'row-reverse' : 'row',
}}>
  <Icon name="arrow-back" style={{ transform: [{ scaleX: isRTL ? -1 : 1 }] }} />
  <Text>Back</Text>
</View>

// react-i18next handles text direction automatically with locale
```

**Properties that flip automatically in RTL:** `flexDirection: 'row'` → becomes right-to-left. `paddingStart/End`, `marginStart/End`, `borderStartWidth/EndWidth`, `left`/`right` positions all flip.

---

### Q152. What is `aspectRatio` in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Layout

**Answer:**
`aspectRatio` maintains a width-to-height ratio. When one dimension is defined (or derived from flex), the other is calculated automatically.

```js
// Maintain 16:9 ratio — useful for video thumbnails
<View style={{ width: '100%', aspectRatio: 16 / 9 }}>
  <Video source={videoSource} style={{ flex: 1 }} />
</View>

// Square image that fills container width
<View style={{ width: '100%', aspectRatio: 1 }}>
  <Image source={photo} style={{ flex: 1 }} />
</View>

// 4:3 card
<View style={{ aspectRatio: 4 / 3, backgroundColor: '#f0f0f0' }}>
  <CardContent />
</View>

// In FlatList grid — dynamic square items
const { width } = useWindowDimensions();
const itemSize = (width - 32) / 3;  // 3-column grid with 16px padding each side

<FlatList
  numColumns={3}
  renderItem={({ item }) => (
    <View style={{ width: itemSize, aspectRatio: 1 }}>
      <Image source={{ uri: item.thumbnail }} style={{ flex: 1 }} />
    </View>
  )}
/>
```

**Follow-up:** What happens if both `width` and `height` are set alongside `aspectRatio`? → `aspectRatio` is ignored. It only kicks in when one dimension is undefined or derived from flex.

---

### Q153. How do you implement a full-screen image with proper aspect ratio?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Layout

**Answer:**
```js
import { Image, useWindowDimensions } from 'react-native';

// Method 1: resizeMode
<Image
  source={{ uri: imageUrl }}
  style={{ width: '100%', height: 300 }}
  resizeMode="cover"    // cover: fills bounds (may crop) — most common
  // resizeMode="contain" — fits inside bounds (letterbox)
  // resizeMode="stretch" — stretches to fill (distorts)
  // resizeMode="center" — centers at natural size (no scale)
/>

// Method 2: aspectRatio for dynamic height
const FullWidthImage = ({ uri, aspectRatio = 16/9 }) => (
  <View style={{ width: '100%', aspectRatio }}>
    <Image source={{ uri }} style={{ flex: 1 }} resizeMode="cover" />
  </View>
);

// Method 3: get image dimensions and calculate
const [imageSize, setImageSize] = useState({ width: 0, height: 0 });
const { width: screenWidth } = useWindowDimensions();

Image.getSize(uri, (w, h) => {
  setImageSize({ width: w, height: h });
});

const displayHeight = (imageSize.height / imageSize.width) * screenWidth;

<Image
  source={{ uri }}
  style={{ width: screenWidth, height: displayHeight }}
/>
```

---

### Q154. What is `minWidth`, `maxWidth`, `minHeight`, `maxHeight` in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Layout

**Answer:**
These constrain a component's size within bounds, useful for responsive layouts that should not get too small or too large.

```js
// Button that doesn't get too narrow or too wide
<Pressable style={{
  minWidth: 120,
  maxWidth: 300,
  paddingHorizontal: 24,
  paddingVertical: 12,
  backgroundColor: '#6200EE',
  borderRadius: 8,
  alignSelf: 'center',
}}>
  <Text style={{ color: 'white' }}>Submit</Text>
</Pressable>

// Text bubble in chat that doesn't exceed 75% of screen
<View style={{ maxWidth: '75%' }}>
  <ChatBubble message={message} />
</View>

// Image that doesn't exceed its natural size
<Image
  source={logo}
  style={{
    maxWidth: 200,
    maxHeight: 80,
    width: '100%',         // tries to be full width
    resizeMode: 'contain', // but constrained by maxWidth/maxHeight
  }}
/>

// Card grid — items never get too large on wide screens
const CARD_MAX_WIDTH = 400;
const { width } = useWindowDimensions();
const cardWidth = Math.min(width - 32, CARD_MAX_WIDTH);

<View style={{ width: cardWidth, alignSelf: 'center' }}>
  <Card />
</View>
```

---

### Q155. How do you create a bottom navigation bar from scratch (without a library)?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Layout + Navigation

**Answer:**
```js
const BottomNav = ({ activeTab, onTabPress }) => {
  const insets = useSafeAreaInsets();
  const tabs = [
    { key: 'home',     icon: 'home',     label: 'Home' },
    { key: 'search',   icon: 'search',   label: 'Search' },
    { key: 'profile',  icon: 'person',   label: 'Profile' },
    { key: 'settings', icon: 'settings', label: 'Settings' },
  ];

  return (
    <View style={{
      flexDirection: 'row',
      backgroundColor: 'white',
      borderTopWidth: StyleSheet.hairlineWidth,
      borderTopColor: '#E0E0E0',
      paddingBottom: insets.bottom,  // safe area for iPhone home indicator
      paddingTop: 8,
    }}>
      {tabs.map(tab => {
        const isActive = activeTab === tab.key;
        return (
          <Pressable
            key={tab.key}
            style={{ flex: 1, alignItems: 'center', gap: 4 }}
            onPress={() => onTabPress(tab.key)}
            accessibilityRole="tab"
            accessibilityState={{ selected: isActive }}
            accessibilityLabel={tab.label}
          >
            <Icon
              name={tab.icon}
              size={24}
              color={isActive ? '#6200EE' : '#9E9E9E'}
            />
            <Text style={{
              fontSize: 11,
              color: isActive ? '#6200EE' : '#9E9E9E',
              fontWeight: isActive ? '600' : '400',
            }}>
              {tab.label}
            </Text>
          </Pressable>
        );
      })}
    </View>
  );
};
```

---

### Q156. How do you implement a tab indicator (underline) that slides between tabs?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Animations + Layout

**Answer:**
```js
import Animated, { useSharedValue, useAnimatedStyle, withSpring } from 'react-native-reanimated';

const SlidingTabs = ({ tabs, onTabPress }) => {
  const [activeIndex, setActiveIndex] = useState(0);
  const [tabWidths, setTabWidths] = useState([]);
  const indicatorX = useSharedValue(0);

  const handleTabPress = (index) => {
    setActiveIndex(index);
    // Calculate X position from cumulative widths
    const xPos = tabWidths.slice(0, index).reduce((sum, w) => sum + w, 0);
    indicatorX.value = withSpring(xPos, { damping: 20 });
    onTabPress(tabs[index]);
  };

  const indicatorStyle = useAnimatedStyle(() => ({
    transform: [{ translateX: indicatorX.value }],
    width: tabWidths[activeIndex] || 0,
  }));

  return (
    <View>
      <View style={{ flexDirection: 'row' }}>
        {tabs.map((tab, i) => (
          <Pressable
            key={tab.key}
            onPress={() => handleTabPress(i)}
            onLayout={(e) => {
              const widths = [...tabWidths];
              widths[i] = e.nativeEvent.layout.width;
              setTabWidths(widths);
            }}
            style={{ paddingHorizontal: 16, paddingVertical: 12 }}
          >
            <Text style={{
              color: activeIndex === i ? '#6200EE' : '#666',
              fontWeight: activeIndex === i ? '600' : '400',
            }}>
              {tab.label}
            </Text>
          </Pressable>
        ))}
      </View>
      {/* Sliding underline indicator */}
      <Animated.View style={[{
        height: 3,
        backgroundColor: '#6200EE',
        borderRadius: 2,
        position: 'absolute',
        bottom: 0,
      }, indicatorStyle]} />
    </View>
  );
};
```

---

### Q157. What is `onLayout` and how do you use it to measure views?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Layout

**Answer:**
`onLayout` fires when a component is laid out (mounted or resized), providing its dimensions and position.

```js
const MyComponent = () => {
  const [layout, setLayout] = useState({ width: 0, height: 0, x: 0, y: 0 });

  return (
    <View
      onLayout={(event) => {
        const { x, y, width, height } = event.nativeEvent.layout;
        setLayout({ x, y, width, height });
        console.log(`Width: ${width}, Height: ${height}`);
      }}
    >
      <Text>Measure me</Text>
    </View>
  );
};

// Measuring with ref (more imperative, after mount)
const ref = useRef(null);

const measure = () => {
  ref.current?.measure((x, y, width, height, pageX, pageY) => {
    // x, y — relative to parent
    // pageX, pageY — relative to screen root
    console.log(`Position on screen: ${pageX}, ${pageY}`);
  });
};

// measureInWindow — position relative to the device window
ref.current?.measureInWindow((x, y, width, height) => {
  // Used for tooltips, popovers positioned relative to a trigger element
});
```

**Use case from your ERP app:** Measuring a table row's height to set `getItemLayout` for FlatList performance optimisation.

---

### Q158. How do you create a card component with a divider line?

**Difficulty:** 🟢 Easy | **Frequency:** Medium | **Category:** Styling

**Answer:**
```js
// Divider component
const Divider = ({ horizontal = true, color = '#E0E0E0', thickness = StyleSheet.hairlineWidth, spacing = 0 }) => (
  <View style={{
    [horizontal ? 'height' : 'width']: thickness,
    backgroundColor: color,
    marginVertical: horizontal ? spacing : 0,
    marginHorizontal: horizontal ? 0 : spacing,
  }} />
);

// Card with sections and dividers
const ProfileCard = ({ user }) => (
  <View style={styles.card}>
    <View style={styles.cardHeader}>
      <Avatar uri={user.avatar} />
      <View style={{ marginLeft: 12 }}>
        <Text style={styles.name}>{user.name}</Text>
        <Text style={styles.role}>{user.role}</Text>
      </View>
    </View>

    <Divider spacing={12} />

    <View style={styles.cardBody}>
      <InfoRow icon="email" label={user.email} />
      <Divider color="#F5F5F5" spacing={8} />
      <InfoRow icon="phone" label={user.phone} />
      <Divider color="#F5F5F5" spacing={8} />
      <InfoRow icon="location" label={user.location} />
    </View>
  </View>
);

const styles = StyleSheet.create({
  card: {
    backgroundColor: 'white',
    borderRadius: 12,
    padding: 16,
    marginHorizontal: 16,
    marginVertical: 8,
    elevation: 2,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 1 },
    shadowOpacity: 0.08,
    shadowRadius: 4,
  },
});
```

---

### Q159. What are `padding` shorthand values in React Native?

**Difficulty:** 🟢 Easy | **Frequency:** Medium | **Category:** Styling

**Answer:**
Unlike CSS, React Native does **not** support 4-value shorthand (`padding: '8px 16px 8px 16px'`). All padding values must be numbers.

```js
// React Native padding properties
padding: 16                   // all sides
paddingTop: 16
paddingBottom: 16
paddingLeft: 16
paddingRight: 16
paddingHorizontal: 16         // left + right (shorthand — unique to RN)
paddingVertical: 8            // top + bottom (shorthand — unique to RN)
paddingStart: 16              // RTL-aware left
paddingEnd: 16                // RTL-aware right

// ❌ CSS-style shorthand — DOES NOT WORK in RN
padding: '8 16'               // invalid
padding: '8px 16px'           // invalid — no units in RN

// ✅ Equivalent in RN
paddingVertical: 8,
paddingHorizontal: 16,

// Priority — more specific overrides:
// padding < paddingVertical/Horizontal < paddingTop/Bottom/Left/Right
{
  padding: 16,
  paddingTop: 8,         // overrides top: 8, others remain 16
  paddingHorizontal: 24, // overrides left+right: 24
}
```

---

### Q160. How do you implement a horizontal scrolling chip/tag filter bar?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** UI Components

**Answer:**
```js
const FilterChips = ({ filters, activeFilter, onSelect }) => {
  const scrollRef = useRef(null);

  return (
    <ScrollView
      ref={scrollRef}
      horizontal
      showsHorizontalScrollIndicator={false}
      contentContainerStyle={{ paddingHorizontal: 16, gap: 8 }}
      style={{ flexGrow: 0 }}         // don't grow vertically
    >
      {filters.map((filter, index) => {
        const isActive = activeFilter === filter.key;
        return (
          <Pressable
            key={filter.key}
            onPress={() => onSelect(filter.key)}
            style={{
              paddingHorizontal: 16,
              paddingVertical: 8,
              borderRadius: 20,
              backgroundColor: isActive ? '#6200EE' : '#F0F0F0',
              borderWidth: 1,
              borderColor: isActive ? '#6200EE' : 'transparent',
            }}
          >
            <Text style={{
              color: isActive ? 'white' : '#333',
              fontSize: 13,
              fontWeight: isActive ? '600' : '400',
            }}>
              {filter.label}
            </Text>
          </Pressable>
        );
      })}
    </ScrollView>
  );
};

// Auto-scroll to active chip when filter changes
useEffect(() => {
  const activeIndex = filters.findIndex(f => f.key === activeFilter);
  scrollRef.current?.scrollTo({ x: activeIndex * 110, animated: true });
}, [activeFilter]);
```

---

### Q161. What is `pointerEvents` in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Layout

**Answer:**
`pointerEvents` controls whether a view can receive touch events.

```js
// 'auto' (default) — view and children receive touches normally
<View pointerEvents="auto">

// 'none' — view and all children are invisible to touches (touches pass through)
<View pointerEvents="none" style={styles.overlay}>
  {/* Overlay that doesn't block touches to content beneath */}
</View>

// 'box-none' — the view itself doesn't receive touches, but children do
// Useful for transparent containers that have interactive children
<View pointerEvents="box-none" style={StyleSheet.absoluteFillObject}>
  {/* The full-screen transparent container passes touches through */}
  {/* But positioned children (buttons, etc.) still receive touches */}
  <Pressable style={styles.floatingButton} onPress={handlePress}>
    <Icon name="add" />
  </Pressable>
</View>

// 'box-only' — only the view receives touches, children do NOT
<View pointerEvents="box-only" onStartShouldSetResponder={() => true}>
  {/* Children can't be tapped — parent intercepts all touches */}
</View>
```

**Common use case — overlay that doesn't block:**
```js
// Loading overlay that shows a spinner but doesn't block touches
// (so user can still interact with content behind it — usually NOT what you want)
<View style={StyleSheet.absoluteFillObject} pointerEvents="none">
  <ActivityIndicator />
</View>
```

---

### Q162. How do you implement a responsive font size that scales with screen width?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Responsive Design

**Answer:**
```js
import { Dimensions, PixelRatio } from 'react-native';

const BASE_WIDTH = 375; // iPhone 11 base design width
const { width: SCREEN_WIDTH } = Dimensions.get('window');
const SCALE = SCREEN_WIDTH / BASE_WIDTH;

// Scale a font size from the base design
const scaledFont = (size) => {
  const newSize = size * SCALE;
  return Math.round(PixelRatio.roundToNearestPixel(newSize));
};

// Usage
const styles = StyleSheet.create({
  title: { fontSize: scaledFont(24) },   // 24dp on 375px screen, larger on wide
  body:  { fontSize: scaledFont(16) },
  small: { fontSize: scaledFont(12) },
});

// More sophisticated — moderate scale (avoids too-large on tablets)
const moderateScale = (size, factor = 0.5) => {
  return size + (scaledFont(size) - size) * factor;
};

// react-native-size-matters library (popular)
import { scale, verticalScale, moderateScale } from 'react-native-size-matters';

<Text style={{ fontSize: moderateScale(16) }}>Scaled text</Text>
<View style={{ padding: scale(16), height: verticalScale(80) }} />
```

**Follow-up:** What is the danger of scaling fonts too aggressively? → On large tablets, fonts can become uncomfortably large. Use `moderateScale` with a factor < 1 to limit growth, or set `maxFontSizeMultiplier`.

---

### Q163. What is `StyleSheet.absoluteFillObject`?

**Difficulty:** 🟢 Easy | **Frequency:** High | **Category:** Styling

**Answer:**
`StyleSheet.absoluteFillObject` is a pre-defined style constant for positioning an element to fill its parent completely.

```js
// StyleSheet.absoluteFillObject is equivalent to:
const absoluteFillObject = {
  position: 'absolute',
  left: 0,
  right: 0,
  top: 0,
  bottom: 0,
};

// Usage — full-screen overlay
<View style={StyleSheet.absoluteFillObject}>
  <BlurView />
</View>

// Combine with other styles
<Image
  source={backgroundImage}
  style={[StyleSheet.absoluteFillObject, { resizeMode: 'cover' }]}
/>

// Or use StyleSheet.absoluteFill (returns the pre-created style, same thing)
<View style={StyleSheet.absoluteFill} />

// Common patterns
// Loading overlay
<View style={[StyleSheet.absoluteFillObject, { backgroundColor: 'rgba(0,0,0,0.5)', justifyContent: 'center', alignItems: 'center' }]}>
  <ActivityIndicator color="white" size="large" />
</View>

// Background image behind content
<View style={{ flex: 1 }}>
  <Image source={bg} style={StyleSheet.absoluteFillObject} resizeMode="cover" />
  <SafeAreaView style={{ flex: 1 }}>
    <Content />
  </SafeAreaView>
</View>
```

---

### Q164. How do you prevent a component from collapsing when it has no content?

**Difficulty:** 🟢 Easy | **Frequency:** Medium | **Category:** Layout

**Answer:**
```js
// Views with no children or content collapse to 0x0
<View />  // renders nothing visible

// Solutions:

// 1. minHeight / minWidth
<View style={{ minHeight: 50, backgroundColor: '#f0f0f0' }}>
  {/* even with no children, takes 50px height */}
</View>

// 2. Explicit dimensions
<View style={{ height: 200, width: '100%', backgroundColor: 'blue' }} />

// 3. flex: 1 (fills parent)
<View style={{ flex: 1, backgroundColor: '#f5f5f5' }} />

// 4. alignSelf: 'stretch' with height (in row direction)
<View style={{ flexDirection: 'row' }}>
  <View style={{ alignSelf: 'stretch', width: 4, backgroundColor: '#6200EE' }} />
  {/* Vertical divider that stretches to match sibling height */}
  <Text style={{ flex: 1, padding: 16 }}>Content</Text>
</View>

// 5. Placeholder content
const EmptyCard = () => (
  <View style={{ height: 120, backgroundColor: '#f9f9f9', borderRadius: 8 }}>
    <Text style={{ color: '#bbb', textAlign: 'center', marginTop: 40 }}>No data</Text>
  </View>
);
```

---

### Q165. What is the difference between `margin: 'auto'` and `alignSelf: 'center'`?

**Difficulty:** 🟡 Medium | **Frequency:** Low | **Category:** Layout

**Answer:**
`margin: 'auto'` was added in React Native 0.74 (matching CSS behaviour). Before that, only `alignSelf` was available for centering.

```js
// alignSelf: 'center' — centers on the cross axis only
// (horizontal center in column direction)
<View style={{ alignSelf: 'center', width: 200 }}>
  <Button />
</View>

// margin: 'auto' — RN 0.74+ — consumes available space equally on both sides
// Works on both axes — like CSS auto margins
<View style={{ marginHorizontal: 'auto', width: 200 }}>
  {/* horizontally centered */}
</View>

<View style={{ marginVertical: 'auto', height: 50 }}>
  {/* vertically centered in available space */}
</View>

// The real power: push one item to the end (classic CSS trick)
<View style={{ flexDirection: 'row' }}>
  <Text>Title</Text>
  <View style={{ marginLeft: 'auto' }}>  {/* pushes button to far right */}
    <Button title="Action" />
  </View>
</View>
```

---

### Q166. How do you implement a badge/notification count on an icon?

**Difficulty:** 🟢 Easy | **Frequency:** High | **Category:** UI Patterns

**Answer:**
```js
const BadgeIcon = ({ iconName, count, size = 28 }) => {
  const displayCount = count > 99 ? '99+' : count.toString();

  return (
    <View style={{ position: 'relative', width: size + 16, height: size + 16 }}>
      {/* Icon */}
      <Icon name={iconName} size={size} color="#333" style={{ margin: 4 }} />

      {/* Badge — only show when count > 0 */}
      {count > 0 && (
        <View style={{
          position: 'absolute',
          top: 0,
          right: 0,
          minWidth: 18,
          height: 18,
          borderRadius: 9,
          backgroundColor: '#F44336',
          alignItems: 'center',
          justifyContent: 'center',
          paddingHorizontal: count > 9 ? 4 : 0,
          borderWidth: 1.5,
          borderColor: 'white',
        }}>
          <Text style={{
            color: 'white',
            fontSize: 10,
            fontWeight: '700',
            lineHeight: 12,
          }}>
            {displayCount}
          </Text>
        </View>
      )}
    </View>
  );
};

// Usage in Tab Bar
<BadgeIcon iconName="notifications" count={notifications.unread} />
```

---

### Q167. How do you implement a progress bar in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** UI Components

**Answer:**
```js
// Simple custom progress bar
const ProgressBar = ({ progress, color = '#6200EE', backgroundColor = '#E0E0E0', height = 8 }) => {
  const clampedProgress = Math.min(Math.max(progress, 0), 1); // 0–1

  return (
    <View style={{
      width: '100%',
      height,
      backgroundColor,
      borderRadius: height / 2,
      overflow: 'hidden',
    }}>
      <View style={{
        width: `${clampedProgress * 100}%`,
        height: '100%',
        backgroundColor: color,
        borderRadius: height / 2,
      }} />
    </View>
  );
};

// Animated progress bar (smooth transitions)
const AnimatedProgressBar = ({ progress, color = '#6200EE' }) => {
  const width = useRef(new Animated.Value(0)).current;

  useEffect(() => {
    Animated.timing(width, {
      toValue: progress,
      duration: 400,
      easing: Easing.out(Easing.ease),
      useNativeDriver: false, // width animation requires false
    }).start();
  }, [progress]);

  const widthInterpolated = width.interpolate({
    inputRange: [0, 1],
    outputRange: ['0%', '100%'],
  });

  return (
    <View style={styles.track}>
      <Animated.View style={[styles.fill, { width: widthInterpolated, backgroundColor: color }]} />
    </View>
  );
};

// File upload progress
<AnimatedProgressBar progress={uploadProgress / 100} color="#4CAF50" />
<Text>{uploadProgress}% uploaded</Text>
```

---

### Q168. What is `lineHeight` and how does it affect text layout in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Typography

**Answer:**
`lineHeight` sets the vertical space allocated per line of text. Unlike CSS, RN `lineHeight` is an absolute pixel value (not a multiplier).

```js
// lineHeight must be >= fontSize to avoid text clipping on Android
<Text style={{ fontSize: 16, lineHeight: 24 }}>  {/* 1.5x line height */}
  Well-spaced paragraph text.
</Text>

// Common ratios
// Body text:    lineHeight = fontSize * 1.5  (e.g., 16 → 24)
// Headings:     lineHeight = fontSize * 1.2  (e.g., 24 → 29)
// Compact UI:   lineHeight = fontSize * 1.3  (e.g., 14 → 18)

// ⚠️ Android bug — text clipping
// On Android, if lineHeight < fontSize, descenders (g, p, y) clip
<Text style={{ fontSize: 16, lineHeight: 14 }}>  {/* Bad — clips descenders */}

// Vertically center single-line text in a fixed-height box
<View style={{ height: 44 }}>
  <Text style={{ lineHeight: 44, fontSize: 16 }}>
    Vertically centered  {/* lineHeight equals container height */}
  </Text>
</View>
```

---

### Q169. How do you implement a swipeable list item (swipe to delete)?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Gestures + UI

**Answer:**
```js
import { Swipeable } from 'react-native-gesture-handler';

const SwipeableRow = ({ item, onDelete, onArchive }) => {
  const swipeableRef = useRef(null);

  const renderRightActions = (progress, dragX) => {
    // Interpolate action button scale as swipe reveals them
    const scale = dragX.interpolate({
      inputRange: [-80, 0],
      outputRange: [1, 0.5],
      extrapolate: 'clamp',
    });

    return (
      <View style={{ width: 160, flexDirection: 'row' }}>
        {/* Archive action */}
        <Pressable
          style={{ width: 80, backgroundColor: '#2196F3', justifyContent: 'center', alignItems: 'center' }}
          onPress={() => { onArchive(item.id); swipeableRef.current?.close(); }}
        >
          <Animated.View style={{ transform: [{ scale }] }}>
            <Icon name="archive" color="white" size={24} />
            <Text style={{ color: 'white', fontSize: 11 }}>Archive</Text>
          </Animated.View>
        </Pressable>
        {/* Delete action */}
        <Pressable
          style={{ width: 80, backgroundColor: '#F44336', justifyContent: 'center', alignItems: 'center' }}
          onPress={() => onDelete(item.id)}
        >
          <Animated.View style={{ transform: [{ scale }] }}>
            <Icon name="delete" color="white" size={24} />
            <Text style={{ color: 'white', fontSize: 11 }}>Delete</Text>
          </Animated.View>
        </Pressable>
      </View>
    );
  };

  return (
    <Swipeable
      ref={swipeableRef}
      renderRightActions={renderRightActions}
      rightThreshold={40}
      friction={2}
      overshootRight={false}
    >
      <View style={styles.row}>
        <Text>{item.title}</Text>
      </View>
    </Swipeable>
  );
};
```

---

### Q170. How do you implement a search bar with clear button?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** UI Components

**Answer:**
```js
const SearchBar = ({ value, onChangeText, placeholder = 'Search...' }) => {
  const inputRef = useRef(null);

  const handleClear = () => {
    onChangeText('');
    inputRef.current?.focus();
  };

  return (
    <View style={{
      flexDirection: 'row',
      alignItems: 'center',
      backgroundColor: '#F5F5F5',
      borderRadius: 10,
      paddingHorizontal: 12,
      height: 44,
      marginHorizontal: 16,
    }}>
      <Icon name="search" size={18} color="#9E9E9E" style={{ marginRight: 8 }} />
      <TextInput
        ref={inputRef}
        value={value}
        onChangeText={onChangeText}
        placeholder={placeholder}
        placeholderTextColor="#9E9E9E"
        style={{ flex: 1, fontSize: 16, color: '#333', padding: 0 }}
        returnKeyType="search"
        clearButtonMode="never"   // disable iOS built-in clear (use custom)
        autoCorrect={false}
        autoCapitalize="none"
      />
      {value.length > 0 && (
        <Pressable
          onPress={handleClear}
          hitSlop={{ top: 8, bottom: 8, left: 8, right: 8 }}
        >
          <View style={{
            width: 18, height: 18, borderRadius: 9,
            backgroundColor: '#BDBDBD',
            alignItems: 'center', justifyContent: 'center',
          }}>
            <Icon name="close" size={12} color="white" />
          </View>
        </Pressable>
      )}
    </View>
  );
};
```

---

### Q171. What is `letterSpacing` and `textTransform` in React Native?

**Difficulty:** 🟢 Easy | **Frequency:** Low | **Category:** Typography

**Answer:**
```js
// letterSpacing — spacing between characters
<Text style={{ letterSpacing: 2 }}>SPACED OUT</Text>
<Text style={{ letterSpacing: -0.5 }}>Tighter text</Text>  // negative = tighter

// textTransform — case transformation
<Text style={{ textTransform: 'uppercase' }}>becomes ALL CAPS</Text>
<Text style={{ textTransform: 'lowercase' }}>BECOMES all lowercase</Text>
<Text style={{ textTransform: 'capitalize' }}>becomes Capitalised Words</Text>
<Text style={{ textTransform: 'none' }}>No transformation</Text>

// Common heading style
const headingStyle = {
  fontSize: 11,
  fontWeight: '600',
  letterSpacing: 1.2,
  textTransform: 'uppercase',
  color: '#9E9E9E',
};

// Section headers in your ERP app
<Text style={headingStyle}>EMPLOYEE DETAILS</Text>
<Text style={headingStyle}>ATTENDANCE SUMMARY</Text>
```

---

### Q172. How do you handle text truncation and ellipsis in React Native?

**Difficulty:** 🟢 Easy | **Frequency:** High | **Category:** Typography

**Answer:**
```js
// numberOfLines — limit lines, add ellipsis at end
<Text numberOfLines={1} ellipsizeMode="tail">
  Very long text that will be truncated with an ellipsis at the end
</Text>

// ellipsizeMode values:
// 'tail' (default) — truncate at end:    "Very long te..."
// 'head'           — truncate at start:  "...ext at start"
// 'middle'         — truncate in middle: "Very l...start"
// 'clip'           — hard clip, no dots: "Very long tex"

// Multi-line truncation
<Text numberOfLines={3} ellipsizeMode="tail">
  This is a paragraph that will show maximum three lines
  and then truncate with an ellipsis if the content
  extends beyond the third line.
</Text>

// Expandable "Read more" text
const ExpandableText = ({ text, maxLines = 3 }) => {
  const [expanded, setExpanded] = useState(false);
  const [isTruncated, setIsTruncated] = useState(false);

  return (
    <View>
      <Text
        numberOfLines={expanded ? undefined : maxLines}
        onTextLayout={(e) => setIsTruncated(e.nativeEvent.lines.length > maxLines)}
      >
        {text}
      </Text>
      {isTruncated && (
        <Pressable onPress={() => setExpanded(!expanded)}>
          <Text style={{ color: '#6200EE' }}>
            {expanded ? 'Read less' : 'Read more'}
          </Text>
        </Pressable>
      )}
    </View>
  );
};
```

---

### Q173. What is `includeFontPadding` and why does it matter on Android?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Typography + Android

**Answer:**
On Android, React Native adds extra padding above and below text by default (`includeFontPadding: true`) to accommodate ascenders and descenders in fonts. This can cause text to appear visually off-center in fixed-height containers.

```js
// Problem: text appears too low in a centered container on Android
<View style={{ height: 44, justifyContent: 'center', alignItems: 'center' }}>
  <Text style={{ fontSize: 16 }}>
    Looks off-center on Android
  </Text>
</View>

// Fix: disable includeFontPadding
<Text style={{
  fontSize: 16,
  includeFontPadding: false,    // Android only — removes extra vertical padding
  textAlignVertical: 'center',  // Android only — vertically centers text
}}>
  Properly centered on Android
</Text>

// Cross-platform text centering
const centeredTextStyle = {
  fontSize: 16,
  lineHeight: 44,               // iOS: set lineHeight to container height
  ...Platform.select({
    android: {
      includeFontPadding: false,
      textAlignVertical: 'center',
    },
  }),
};
```

**Follow-up:** Is `includeFontPadding` going to be removed? → Yes, React Native has been working toward removing this inconsistency between platforms in the New Architecture.

---

### Q174. How do you implement a collapsible/accordion section?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** UI Components

**Answer:**
```js
import Animated, {
  useSharedValue, useAnimatedStyle,
  withTiming, Easing
} from 'react-native-reanimated';

const Accordion = ({ title, children }) => {
  const [open, setOpen] = useState(false);
  const [contentHeight, setContentHeight] = useState(0);
  const animatedHeight = useSharedValue(0);
  const animatedRotation = useSharedValue(0);

  const toggle = () => {
    const isOpening = !open;
    setOpen(isOpening);
    animatedHeight.value = withTiming(isOpening ? contentHeight : 0, {
      duration: 300,
      easing: Easing.inOut(Easing.ease),
    });
    animatedRotation.value = withTiming(isOpening ? 1 : 0, { duration: 300 });
  };

  const bodyStyle = useAnimatedStyle(() => ({
    height: animatedHeight.value,
    overflow: 'hidden',
  }));

  const arrowStyle = useAnimatedStyle(() => ({
    transform: [{
      rotate: `${animatedRotation.value * 180}deg`
    }],
  }));

  return (
    <View style={styles.accordion}>
      {/* Header */}
      <Pressable style={styles.header} onPress={toggle}>
        <Text style={styles.title}>{title}</Text>
        <Animated.View style={arrowStyle}>
          <Icon name="chevron-down" size={20} color="#666" />
        </Animated.View>
      </Pressable>

      {/* Animated body */}
      <Animated.View style={bodyStyle}>
        <View
          onLayout={(e) => setContentHeight(e.nativeEvent.layout.height)}
          style={styles.body}
        >
          {children}
        </View>
      </Animated.View>
    </View>
  );
};
```

---

### Q175. How do you implement a floating label (Material Design) TextInput?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** UI Components

**Answer:**
```js
const FloatingLabelInput = ({ label, value, onChangeText, ...rest }) => {
  const [isFocused, setIsFocused] = useState(false);
  const animatedValue = useRef(new Animated.Value(value ? 1 : 0)).current;

  const animate = (toValue) => {
    Animated.timing(animatedValue, {
      toValue,
      duration: 200,
      easing: Easing.out(Easing.ease),
      useNativeDriver: false, // cannot use native driver for top/fontSize
    }).start();
  };

  const handleFocus = () => { setIsFocused(true); animate(1); };
  const handleBlur = () => {
    setIsFocused(false);
    if (!value) animate(0); // only float down if no value
  };

  const labelStyle = {
    position: 'absolute',
    left: 12,
    top: animatedValue.interpolate({
      inputRange: [0, 1],
      outputRange: [16, -8],  // moves from center to top
    }),
    fontSize: animatedValue.interpolate({
      inputRange: [0, 1],
      outputRange: [16, 12],  // shrinks from 16 to 12
    }),
    color: animatedValue.interpolate({
      inputRange: [0, 1],
      outputRange: ['#9E9E9E', '#6200EE'],
    }),
    backgroundColor: 'white',
    paddingHorizontal: 4,
    zIndex: 1,
  };

  return (
    <View style={{
      borderWidth: 1.5,
      borderColor: isFocused ? '#6200EE' : '#BDBDBD',
      borderRadius: 8,
      paddingHorizontal: 12,
      paddingTop: 16,
      paddingBottom: 8,
      marginVertical: 8,
    }}>
      <Animated.Text style={labelStyle}>{label}</Animated.Text>
      <TextInput
        value={value}
        onChangeText={onChangeText}
        onFocus={handleFocus}
        onBlur={handleBlur}
        style={{ fontSize: 16, color: '#212121', padding: 0 }}
        {...rest}
      />
    </View>
  );
};

// Usage
const [email, setEmail] = useState('');
<FloatingLabelInput label="Email address" value={email} onChangeText={setEmail} keyboardType="email-address" />
```

---

## Sections Overview (Q176–Q500)

| Section | Questions | Topics |
|---------|-----------|--------|
| Animations deep dive | Q176–Q230 | Reanimated 2, gesture combos, shared element |
| Performance | Q231–Q290 | Memory leaks, profiling, bundle optimisation |
| Native Modules | Q291–Q340 | Writing custom native modules (iOS + Android) |
| Expo vs CLI | Q341–Q370 | Managed vs bare, EAS Build, Expo Go |
| Testing | Q371–Q420 | Unit, integration, e2e (Detox), mocking |
| Debugging | Q421–Q450 | Flipper, Hermes debugger, crash reporting |
| Storage & Permissions | Q451–Q480 | AsyncStorage, Keychain, permission flows |
| Miscellaneous | Q481–Q500 | Accessibility, internationalisation, misc APIs |

---

> 💡 **Tip for GitHub:** Add a `## Table of Contents` section at the top with anchor links to each question for easy navigation.

---

*Part 01 of 8 — [← Back to Part README](./README.md) · [← Main README](../README.md)*

## Section 6: Animations Deep Dive (Q176–Q230)

---

### Q176. What are the core primitives of Reanimated 2?

**Difficulty:** 🟡 Medium | **Frequency:** Very High | **Category:** Reanimated

**Answer:**
Reanimated 2 introduced a completely new API built on JSI (runs on the UI thread). The five core primitives:

| Primitive | Purpose |
|-----------|---------|
| `useSharedValue` | Mutable animated value living on the UI thread |
| `useAnimatedStyle` | Worklet that returns styles computed on the UI thread |
| `useAnimatedProps` | Worklet that returns non-style props (e.g., SVG `strokeDashoffset`) |
| `useAnimatedScrollHandler` | Worklet for scroll event handling |
| `useDerivedValue` | Derived shared value computed from other shared values |

```js
import Animated, {
  useSharedValue, useAnimatedStyle, withSpring, withTiming,
} from 'react-native-reanimated';

const MyComponent = () => {
  const scale = useSharedValue(1);   // lives on UI thread

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ scale: scale.value }],
  }));

  return (
    <Pressable
      onPressIn={() => { scale.value = withSpring(0.95); }}
      onPressOut={() => { scale.value = withSpring(1); }}
    >
      <Animated.View style={[styles.box, animatedStyle]}>
        <Text>Press me</Text>
      </Animated.View>
    </Pressable>
  );
};
```

**Key advantage:** Animations never drop frames even when the JS thread is 100% busy — the animation worklet runs entirely on the UI thread via JSI.

---

### Q177. What is a worklet in Reanimated 2?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Reanimated

**Answer:**
A worklet is a JavaScript function compiled by Reanimated's Babel plugin to run on the UI thread. Mark with the `'worklet'` directive, or it's implied inside `useAnimatedStyle` / gesture handlers.

```js
import { runOnUI, runOnJS } from 'react-native-reanimated';

// Manual worklet — explicit directive
const clamp = (value, min, max) => {
  'worklet';
  return Math.min(Math.max(value, min), max);
};

// Reanimated worklet context — 'worklet' is implicit
const animatedStyle = useAnimatedStyle(() => {
  const clamped = clamp(scale.value, 0.5, 2.0); // call our worklet
  return { transform: [{ scale: clamped }] };
});

// runOnUI — execute a worklet from JS thread
const triggerUIAnimation = () => {
  runOnUI(() => {
    'worklet';
    sharedValue.value = withSpring(100);
  })(); // call immediately
};

// runOnJS — call JS function FROM a worklet (UI → JS thread)
const gesture = Gesture.Pan().onUpdate((e) => {
  translateX.value = e.translationX;
  runOnJS(setMyState)(e.translationX); // bridge back to JS
});
```

**Follow-up:** Does `'worklet'` mean it always runs on UI thread? → No — it makes the function *capable* of running on the UI thread. The same function can still be called from the JS thread.

---

### Q178. What is `withSpring`, `withTiming`, `withDecay`, and `withRepeat`?

**Difficulty:** 🟡 Medium | **Frequency:** Very High | **Category:** Reanimated

**Answer:**
```js
import { withSpring, withTiming, withDecay, withRepeat, withSequence, Easing } from 'react-native-reanimated';

// withTiming — linear/eased animation to target value
value.value = withTiming(100, { duration: 300, easing: Easing.out(Easing.ease) });

// withSpring — physics-based spring (natural, bouncy)
value.value = withSpring(100, {
  damping: 15,      // higher = less bounce
  stiffness: 150,   // higher = faster
  mass: 1,
});

// withDecay — velocity-based deceleration (flick gesture)
value.value = withDecay({
  velocity: gestureEvent.velocityX,
  clamp: [-300, 300],
  deceleration: 0.997,
});

// withRepeat — repeat any animation
value.value = withRepeat(
  withTiming(1, { duration: 800 }),
  -1,    // -1 = infinite
  true,  // reverse each cycle
);

// withSequence — chain in order
value.value = withSequence(
  withTiming(1.2, { duration: 100 }),
  withSpring(1, { damping: 10 }),
);

// Callback on completion
value.value = withTiming(100, { duration: 500 }, (finished) => {
  'worklet';
  if (finished) runOnJS(onComplete)();
});
```

**Rule:** `withTiming` for precise duration control; `withSpring` for natural-feeling drag/press interactions.

---

### Q179. What is `useDerivedValue` in Reanimated 2?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Reanimated

**Answer:**
`useDerivedValue` creates a new shared value **automatically computed** from other shared values — runs on the UI thread, no JS involvement on update.

```js
import { useSharedValue, useDerivedValue, useAnimatedStyle, interpolate } from 'react-native-reanimated';

const progress = useSharedValue(0); // 0 → 1

// Derived values — auto-update when progress changes
const opacity = useDerivedValue(() => progress.value);
const translateY = useDerivedValue(() => progress.value * -100);
const rotation = useDerivedValue(() =>
  interpolate(progress.value, [0, 1], [0, 360], Extrapolation.CLAMP)
);

// Reuse across multiple animated components
const barStyle = useAnimatedStyle(() => ({ width: `${progress.value * 100}%` }));
const iconStyle = useAnimatedStyle(() => ({
  transform: [{ rotate: `${rotation.value}deg` }],
  opacity: opacity.value,
}));
```

**`useDerivedValue` vs `useAnimatedStyle`:**
- `useDerivedValue` → creates a reusable shared *value*
- `useAnimatedStyle` → creates *styles* for one specific component

---

### Q180. What is `useAnimatedScrollHandler`?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Reanimated

**Answer:**
Creates a scroll event handler worklet running on the UI thread — no Bridge crossing per scroll event.

```js
import Animated, {
  useSharedValue, useAnimatedScrollHandler, useAnimatedStyle,
  interpolate, Extrapolation,
} from 'react-native-reanimated';

const ParallaxScreen = () => {
  const scrollY = useSharedValue(0);

  const scrollHandler = useAnimatedScrollHandler({
    onScroll: (event) => {
      scrollY.value = event.contentOffset.y;
    },
  });

  const headerStyle = useAnimatedStyle(() => ({
    opacity: interpolate(scrollY.value, [0, 100], [1, 0], Extrapolation.CLAMP),
    transform: [{
      translateY: interpolate(scrollY.value, [0, 100], [0, -50], Extrapolation.CLAMP),
    }],
  }));

  // Parallax — background moves at half scroll speed
  const bgStyle = useAnimatedStyle(() => ({
    transform: [{ translateY: scrollY.value * 0.5 }],
  }));

  return (
    <View style={{ flex: 1 }}>
      <Animated.Image source={heroBg} style={[styles.bg, bgStyle]} />
      <Animated.View style={[styles.header, headerStyle]}>
        <Text style={styles.title}>My App</Text>
      </Animated.View>
      <Animated.ScrollView onScroll={scrollHandler} scrollEventThrottle={16}>
        <Content />
      </Animated.ScrollView>
    </View>
  );
};
```

---

### Q181. What is `useAnimatedProps` and when do you use it?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Reanimated

**Answer:**
`useAnimatedProps` animates **non-style props** — SVG attributes, progress values, text content. Use when the property to animate is not a CSS/style property.

```js
import Animated, { useSharedValue, useAnimatedProps, withTiming } from 'react-native-reanimated';
import Svg, { Circle } from 'react-native-svg';

const AnimatedCircle = Animated.createAnimatedComponent(Circle);

const CircularProgress = ({ progress }) => {
  const RADIUS = 60;
  const CIRCUMFERENCE = 2 * Math.PI * RADIUS;
  const animatedOffset = useSharedValue(CIRCUMFERENCE);

  useEffect(() => {
    animatedOffset.value = withTiming(CIRCUMFERENCE * (1 - progress), { duration: 800 });
  }, [progress]);

  const animatedProps = useAnimatedProps(() => ({
    strokeDashoffset: animatedOffset.value,
  }));

  return (
    <Svg width={140} height={140}>
      <Circle cx={70} cy={70} r={RADIUS} stroke="#E0E0E0" strokeWidth={10} fill="none" />
      <AnimatedCircle
        cx={70} cy={70} r={RADIUS}
        stroke="#6200EE" strokeWidth={10} fill="none"
        strokeDasharray={CIRCUMFERENCE}
        animatedProps={animatedProps}
        strokeLinecap="round" rotation="-90" origin="70, 70"
      />
    </Svg>
  );
};
```

---

### Q182. How do you implement drag-and-drop with Reanimated 2?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Reanimated + Gestures

**Answer:**
```js
import Animated, { useSharedValue, useAnimatedStyle, withSpring } from 'react-native-reanimated';
import { Gesture, GestureDetector } from 'react-native-gesture-handler';

const DraggableCard = ({ onDrop }) => {
  const translateX = useSharedValue(0);
  const translateY = useSharedValue(0);
  const offsetX = useSharedValue(0);
  const offsetY = useSharedValue(0);
  const scale = useSharedValue(1);

  const panGesture = Gesture.Pan()
    .onStart(() => {
      offsetX.value = translateX.value;
      offsetY.value = translateY.value;
      scale.value = withSpring(1.05); // lift effect
    })
    .onUpdate((e) => {
      translateX.value = offsetX.value + e.translationX;
      translateY.value = offsetY.value + e.translationY;
    })
    .onEnd(() => {
      scale.value = withSpring(1);
      const inDropZone =
        translateX.value > 100 && translateX.value < 300 &&
        translateY.value > 400;

      if (inDropZone) {
        runOnJS(onDrop)({ x: translateX.value, y: translateY.value });
      } else {
        translateX.value = withSpring(0);
        translateY.value = withSpring(0);
      }
    });

  const cardStyle = useAnimatedStyle(() => ({
    transform: [
      { translateX: translateX.value },
      { translateY: translateY.value },
      { scale: scale.value },
    ],
    elevation: scale.value > 1 ? 8 : 2,
  }));

  return (
    <GestureDetector gesture={panGesture}>
      <Animated.View style={[styles.card, cardStyle]}>
        <Text>Drag me</Text>
      </Animated.View>
    </GestureDetector>
  );
};
```

---

### Q183. How do you compose gestures in React Native Gesture Handler v2?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Gestures

**Answer:**
```js
import { Gesture, GestureDetector } from 'react-native-gesture-handler';

// Gesture.Simultaneous — both gestures active at same time
const simultaneous = Gesture.Simultaneous(panGesture, pinchGesture);

// Gesture.Exclusive — only first active gesture runs
const exclusive = Gesture.Exclusive(doubleTap, singleTap);

// Gesture.Race — first gesture to activate wins, cancels others
const race = Gesture.Race(doubleTap, panGesture);

// Full pinch-zoom + pan example
const pinchGesture = Gesture.Pinch()
  .onStart(() => { savedScale.value = scale.value; })
  .onUpdate((e) => { scale.value = savedScale.value * e.scale; })
  .onEnd(() => {
    if (scale.value < 0.8) scale.value = withSpring(0.8);
    if (scale.value > 5)   scale.value = withSpring(5);
  });

const panGesture = Gesture.Pan()
  .minPointers(2)
  .onStart(() => { savedX.value = translateX.value; savedY.value = translateY.value; })
  .onUpdate((e) => {
    translateX.value = savedX.value + e.translationX;
    translateY.value = savedY.value + e.translationY;
  });

const doubleTap = Gesture.Tap()
  .numberOfTaps(2)
  .onEnd(() => {
    scale.value = withSpring(scale.value > 1 ? 1 : 2.5);
    if (scale.value > 1) { translateX.value = withSpring(0); translateY.value = withSpring(0); }
  });

const composed = Gesture.Race(doubleTap, Gesture.Simultaneous(pinchGesture, panGesture));

<GestureDetector gesture={composed}>
  <Animated.Image source={source} style={[styles.image, imageStyle]} resizeMode="contain" />
</GestureDetector>
```

---

### Q184. What is `scrollEventThrottle` and what value should you use?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Performance + Animations

**Answer:**
Controls how often `onScroll` fires on iOS (Android fires every frame by default).

```js
// For animations — use 16 (≈60fps)
<Animated.ScrollView onScroll={scrollHandler} scrollEventThrottle={16} />

// Common values:
// 16  — ~60fps — use for scroll-driven animations
// 1   — near-every-frame — very precise tracking
// 100 — ~10fps — for "did user scroll" checks (no animations)

// With Reanimated's useAnimatedScrollHandler — always 16
// Without animations — 100 saves CPU
```

**iOS vs Android:** On iOS, `scrollEventThrottle` defaults to `0` (only start/end). On Android, the JS thread receives events every frame. Always set `scrollEventThrottle={16}` for animations on iOS.

---

### Q185. How do you implement a swipe-to-dismiss modal?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Gestures + Animations

**Answer:**
```js
import Animated, { useSharedValue, useAnimatedStyle, withSpring, withTiming, runOnJS } from 'react-native-reanimated';
import { Gesture, GestureDetector } from 'react-native-gesture-handler';

const SCREEN_HEIGHT = Dimensions.get('window').height;
const DISMISS_THRESHOLD = SCREEN_HEIGHT * 0.3;

const SwipeToDismissModal = ({ visible, onDismiss, children }) => {
  const translateY = useSharedValue(SCREEN_HEIGHT);

  useEffect(() => {
    translateY.value = withSpring(visible ? 0 : SCREEN_HEIGHT, { damping: 20, stiffness: 300 });
  }, [visible]);

  const panGesture = Gesture.Pan()
    .onUpdate((e) => {
      if (e.translationY > 0) translateY.value = e.translationY;
    })
    .onEnd((e) => {
      const shouldDismiss = e.translationY > DISMISS_THRESHOLD || e.velocityY > 1500;
      if (shouldDismiss) {
        translateY.value = withTiming(SCREEN_HEIGHT, { duration: 200 }, () =>
          runOnJS(onDismiss)()
        );
      } else {
        translateY.value = withSpring(0, { damping: 20 });
      }
    });

  const backdropStyle = useAnimatedStyle(() => ({
    opacity: 1 - translateY.value / SCREEN_HEIGHT,
  }));

  const sheetStyle = useAnimatedStyle(() => ({
    transform: [{ translateY: translateY.value }],
  }));

  if (!visible) return null;

  return (
    <View style={StyleSheet.absoluteFillObject}>
      <Animated.View style={[StyleSheet.absoluteFillObject, { backgroundColor: 'rgba(0,0,0,0.5)' }, backdropStyle]}
        onTouchEnd={onDismiss} />
      <GestureDetector gesture={panGesture}>
        <Animated.View style={[styles.sheet, sheetStyle]}>
          <View style={styles.handle} />
          {children}
        </Animated.View>
      </GestureDetector>
    </View>
  );
};
```

---

### Q186. How do you animate a list item entrance (staggered)?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Reanimated + FlatList

**Answer:**
```js
// Method 1: Manual stagger with withDelay
const AnimatedListItem = ({ item, index }) => {
  const opacity = useSharedValue(0);
  const translateY = useSharedValue(30);

  useEffect(() => {
    const delay = index * 80;
    opacity.value = withDelay(delay, withTiming(1, { duration: 400 }));
    translateY.value = withDelay(delay, withSpring(0, { damping: 15 }));
  }, []);

  const style = useAnimatedStyle(() => ({
    opacity: opacity.value,
    transform: [{ translateY: translateY.value }],
  }));

  return <Animated.View style={[styles.item, style]}><Text>{item.title}</Text></Animated.View>;
};

// Method 2: Reanimated Layout Animations (cleanest)
import { FadeInDown, FadeOutLeft, Layout, LinearTransition } from 'react-native-reanimated';

{items.map((item, index) => (
  <Animated.View
    key={item.id}
    entering={FadeInDown.delay(index * 80).springify()}
    exiting={FadeOutLeft.duration(200)}
    layout={LinearTransition.springify()} // animate siblings when item removed
  >
    <ListItem item={item} />
  </Animated.View>
))}
```

---

### Q187. What are Layout Animations in Reanimated 2?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Reanimated

**Answer:**
Declarative animations for component mount/unmount/reorder — no manual `useSharedValue` needed.

```js
import Animated, {
  FadeIn, FadeOut, FadeInDown, SlideInRight, ZoomIn, BounceIn,
  Layout, LinearTransition, CurvedTransition,
  Keyframe,
} from 'react-native-reanimated';

// Basic — entering / exiting
<Animated.View entering={FadeInDown} exiting={FadeOut}>

// With chained modifiers
<Animated.View
  entering={FadeInDown.delay(200).duration(400).springify().damping(12)}
  exiting={FadeOut.duration(200)}
  layout={LinearTransition.springify()} // siblings animate on this view's exit
>

// Custom keyframe animation
const PopIn = new Keyframe({
  0:   { opacity: 0, transform: [{ scale: 0.5 }] },
  60:  { opacity: 1, transform: [{ scale: 1.1 }] },
  100: { opacity: 1, transform: [{ scale: 1 }] },
}).duration(500);

<Animated.View entering={PopIn}>

// Enable on Android (required for Reanimated < 3)
// AndroidManifest.xml — handled by autolinking
```

---

### Q188. How do you implement a photo gallery transition without a library?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Animations

**Answer:**
```js
// Measure source element → animate clone from source bounds to full screen
const openPhoto = (photo, ref) => {
  ref.current.measureInWindow((x, y, width, height) => {
    setSourceLayout({ x, y, width, height });
    setSelectedPhoto(photo);
  });
};

const PhotoModal = ({ photo, sourceLayout, onClose }) => {
  const { width: W, height: H } = useWindowDimensions();
  const x = useSharedValue(sourceLayout.x);
  const y = useSharedValue(sourceLayout.y);
  const w = useSharedValue(sourceLayout.width);
  const h = useSharedValue(sourceLayout.height);
  const bgOpacity = useSharedValue(0);

  useEffect(() => {
    x.value = withSpring(0, { damping: 20 });
    y.value = withSpring(0, { damping: 20 });
    w.value = withSpring(W, { damping: 20 });
    h.value = withSpring(H, { damping: 20 });
    bgOpacity.value = withTiming(1, { duration: 300 });
  }, []);

  const handleClose = () => {
    x.value = withSpring(sourceLayout.x);
    y.value = withSpring(sourceLayout.y);
    w.value = withSpring(sourceLayout.width);
    h.value = withSpring(sourceLayout.height);
    bgOpacity.value = withTiming(0, { duration: 250 }, () => runOnJS(onClose)());
  };

  const imgStyle = useAnimatedStyle(() => ({
    position: 'absolute', left: x.value, top: y.value, width: w.value, height: h.value,
  }));

  return (
    <View style={StyleSheet.absoluteFillObject}>
      <Animated.View style={[StyleSheet.absoluteFillObject, { backgroundColor: 'black' }, { opacity: bgOpacity }]} />
      <Pressable style={StyleSheet.absoluteFillObject} onPress={handleClose} />
      <Animated.Image source={{ uri: photo.uri }} style={imgStyle} resizeMode="cover" />
    </View>
  );
};
```

---

### Q189. What is `cancelAnimation` in Reanimated 2?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Reanimated

**Answer:**
Stops an in-progress animation, freezing the shared value at its current position.

```js
import { cancelAnimation, withRepeat, withTiming } from 'react-native-reanimated';

const LoadingSpinner = ({ isLoading }) => {
  const rotation = useSharedValue(0);

  useEffect(() => {
    if (isLoading) {
      rotation.value = withRepeat(
        withTiming(360, { duration: 1000, easing: Easing.linear }),
        -1
      );
    } else {
      cancelAnimation(rotation);    // freeze at current angle
      rotation.value = withTiming(0, { duration: 200 }); // optionally snap to 0
    }
  }, [isLoading]);

  const style = useAnimatedStyle(() => ({
    transform: [{ rotate: `${rotation.value}deg` }],
  }));

  return <Animated.View style={style}><Icon name="refresh" /></Animated.View>;
};
```

---

### Q190. How do you animate a counter from 0 to a target number?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Animations

**Answer:**
```js
import Animated, { useSharedValue, useAnimatedProps, withTiming, Easing } from 'react-native-reanimated';
import { TextInput } from 'react-native';

const AnimatedTextInput = Animated.createAnimatedComponent(TextInput);

const AnimatedCounter = ({ target, duration = 1000 }) => {
  const value = useSharedValue(0);

  useEffect(() => {
    value.value = withTiming(target, { duration, easing: Easing.out(Easing.ease) });
  }, [target]);

  const animatedProps = useAnimatedProps(() => ({
    text: Math.round(value.value).toLocaleString('en-IN'),
    defaultValue: '0',
  }));

  return (
    <AnimatedTextInput
      animatedProps={animatedProps}
      editable={false}
      style={styles.counterText}
    />
  );
};

// Usage in ERP dashboard
<AnimatedCounter target={totalEmployees} />
<AnimatedCounter target={monthlyRevenue} duration={1500} />
```

---

### Q191. What is `withSequence` and `withDelay` in Reanimated 2?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Reanimated

**Answer:**
```js
import { withSequence, withDelay, withTiming, withSpring } from 'react-native-reanimated';

// withSequence — each starts after previous finishes
value.value = withSequence(
  withTiming(0, { duration: 0 }),       // instant reset
  withTiming(1.2, { duration: 150 }),   // scale up
  withSpring(1, { damping: 8 }),        // spring back
);

// Shake animation
const shake = () => {
  translateX.value = withSequence(
    withTiming(-10, { duration: 50 }),
    withTiming(10, { duration: 50 }),
    withTiming(-8, { duration: 50 }),
    withTiming(8, { duration: 50 }),
    withSpring(0)
  );
};

// withDelay — wait before starting
opacity.value = withDelay(500, withTiming(1)); // wait 500ms, then fade in

// Stagger multiple items
items.forEach((item, i) => {
  item.opacity.value = withDelay(i * 100, withTiming(1, { duration: 400 }));
  item.translateY.value = withDelay(i * 100, withSpring(0, { damping: 14 }));
});

// withDelay inside withSequence
value.value = withSequence(
  withTiming(1, { duration: 300 }),
  withDelay(500, withTiming(0, { duration: 300 })) // pause 500ms then fade out
);
```

---

### Q192. How do you implement a card flip animation?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Animations

**Answer:**
```js
const FlipCard = ({ front, back }) => {
  const rotation = useSharedValue(0);
  const [isFlipped, setIsFlipped] = useState(false);

  const flip = () => {
    rotation.value = withTiming(isFlipped ? 0 : 1, { duration: 500 });
    setIsFlipped(!isFlipped);
  };

  const frontStyle = useAnimatedStyle(() => ({
    transform: [
      { perspective: 1000 },
      { rotateY: `${interpolate(rotation.value, [0, 1], [0, 180])}deg` },
    ],
    backfaceVisibility: 'hidden',
  }));

  const backStyle = useAnimatedStyle(() => ({
    transform: [
      { perspective: 1000 },
      { rotateY: `${interpolate(rotation.value, [0, 1], [180, 360])}deg` },
    ],
    backfaceVisibility: 'hidden',
    position: 'absolute', top: 0, left: 0, right: 0, bottom: 0,
  }));

  return (
    <Pressable onPress={flip}>
      <View style={styles.cardContainer}>
        <Animated.View style={[styles.card, frontStyle]}>{front}</Animated.View>
        <Animated.View style={[styles.card, backStyle]}>{back}</Animated.View>
      </View>
    </Pressable>
  );
};
```

---

### Q193. How do you implement pinch-to-zoom with Reanimated 2?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Gestures + Animations

**Answer:**
```js
const ZoomableImage = ({ source }) => {
  const scale = useSharedValue(1);
  const savedScale = useSharedValue(1);
  const translateX = useSharedValue(0);
  const translateY = useSharedValue(0);
  const savedX = useSharedValue(0);
  const savedY = useSharedValue(0);

  const pinch = Gesture.Pinch()
    .onStart(() => { savedScale.value = scale.value; })
    .onUpdate((e) => { scale.value = savedScale.value * e.scale; })
    .onEnd(() => {
      if (scale.value < 1) { scale.value = withSpring(1); translateX.value = withSpring(0); translateY.value = withSpring(0); }
      if (scale.value > 5) scale.value = withSpring(5);
    });

  const pan = Gesture.Pan()
    .minPointers(2)
    .onStart(() => { savedX.value = translateX.value; savedY.value = translateY.value; })
    .onUpdate((e) => { translateX.value = savedX.value + e.translationX; translateY.value = savedY.value + e.translationY; });

  const doubleTap = Gesture.Tap().numberOfTaps(2).onEnd(() => {
    if (scale.value > 1) { scale.value = withSpring(1); translateX.value = withSpring(0); translateY.value = withSpring(0); }
    else scale.value = withSpring(2.5);
  });

  const composed = Gesture.Race(doubleTap, Gesture.Simultaneous(pinch, pan));

  const imageStyle = useAnimatedStyle(() => ({
    transform: [{ translateX: translateX.value }, { translateY: translateY.value }, { scale: scale.value }],
  }));

  return (
    <GestureDetector gesture={composed}>
      <Animated.Image source={source} style={[styles.image, imageStyle]} resizeMode="contain" />
    </GestureDetector>
  );
};
```

---

### Q194. What is `interpolate` in Reanimated 2?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Reanimated

**Answer:**
```js
import { interpolate, Extrapolation } from 'react-native-reanimated';

const animatedStyle = useAnimatedStyle(() => {
  const opacity = interpolate(
    scrollY.value,      // input value
    [0, 100],           // input range
    [1, 0],             // output range
    Extrapolation.CLAMP // clamp, extend, or identity
  );

  const scale = interpolate(
    scrollY.value,
    [0, 50, 100],       // multi-step
    [1, 1.2, 0.8],
    Extrapolation.CLAMP
  );

  return { opacity, transform: [{ scale }] };
});

// interpolateColor — color interpolation
import { interpolateColor } from 'react-native-reanimated';

const bg = interpolateColor(
  progress.value,
  [0, 0.5, 1],
  ['#FF0000', '#FFFF00', '#00FF00']
);

// Extrapolation.CLAMP  — stays within outputRange bounds (most common)
// Extrapolation.EXTEND — continues linearly beyond bounds
// Extrapolation.IDENTITY — output equals input beyond bounds
```

---

### Q195. How do you implement a tinder-style swipe card?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Gestures + Animations

**Answer:**
```js
const SWIPE_THRESHOLD = 120;

const SwipeCard = ({ card, onSwipeLeft, onSwipeRight }) => {
  const translateX = useSharedValue(0);
  const translateY = useSharedValue(0);

  const gesture = Gesture.Pan()
    .onUpdate((e) => {
      translateX.value = e.translationX;
      translateY.value = e.translationY;
    })
    .onEnd((e) => {
      if (e.translationX > SWIPE_THRESHOLD) {
        translateX.value = withTiming(500, { duration: 200 }, () => runOnJS(onSwipeRight)(card));
      } else if (e.translationX < -SWIPE_THRESHOLD) {
        translateX.value = withTiming(-500, { duration: 200 }, () => runOnJS(onSwipeLeft)(card));
      } else {
        translateX.value = withSpring(0, { damping: 15 });
        translateY.value = withSpring(0, { damping: 15 });
      }
    });

  const cardStyle = useAnimatedStyle(() => ({
    transform: [
      { translateX: translateX.value },
      { translateY: translateY.value },
      { rotate: `${translateX.value * 0.1}deg` },
    ],
  }));

  const likeOpacity = useAnimatedStyle(() => ({
    opacity: Math.max(0, translateX.value / SWIPE_THRESHOLD),
  }));
  const nopeOpacity = useAnimatedStyle(() => ({
    opacity: Math.max(0, -translateX.value / SWIPE_THRESHOLD),
  }));

  return (
    <GestureDetector gesture={gesture}>
      <Animated.View style={[styles.card, cardStyle]}>
        <Image source={{ uri: card.photo }} style={styles.cardImage} />
        <Animated.View style={[styles.likeLabel, likeOpacity]}><Text style={styles.likeText}>LIKE 💚</Text></Animated.View>
        <Animated.View style={[styles.nopeLabel, nopeOpacity]}><Text style={styles.nopeText}>NOPE ❌</Text></Animated.View>
      </Animated.View>
    </GestureDetector>
  );
};
```

---

### Q196. How do you implement a bottom sheet with snap points?

**Difficulty:** 🔴 Hard | **Frequency:** Very High | **Category:** Reanimated + Gestures

**Answer:**
```js
const SCREEN_HEIGHT = Dimensions.get('window').height;
const SNAP = {
  COLLAPSED: SCREEN_HEIGHT * 0.75,
  HALF: SCREEN_HEIGHT * 0.4,
  EXPANDED: SCREEN_HEIGHT * 0.08,
};

const SnapBottomSheet = ({ children }) => {
  const translateY = useSharedValue(SNAP.COLLAPSED);
  const startY = useSharedValue(SNAP.COLLAPSED);

  const findNearestSnap = (y, velocity) => {
    'worklet';
    const pts = Object.values(SNAP);
    if (velocity > 500) {
      const lower = pts.filter(p => p > y);
      return lower.length ? Math.min(...lower) : SNAP.COLLAPSED;
    }
    if (velocity < -500) {
      const higher = pts.filter(p => p < y);
      return higher.length ? Math.max(...higher) : SNAP.EXPANDED;
    }
    return pts.reduce((c, p) => Math.abs(p - y) < Math.abs(c - y) ? p : c);
  };

  const panGesture = Gesture.Pan()
    .onStart(() => { startY.value = translateY.value; })
    .onUpdate((e) => {
      translateY.value = Math.max(SNAP.EXPANDED, Math.min(SNAP.COLLAPSED, startY.value + e.translationY));
    })
    .onEnd((e) => {
      translateY.value = withSpring(findNearestSnap(translateY.value, e.velocityY), { damping: 20, stiffness: 200 });
    });

  const sheetStyle = useAnimatedStyle(() => ({
    transform: [{ translateY: translateY.value }],
    position: 'absolute', left: 0, right: 0, top: 0, height: SCREEN_HEIGHT,
  }));

  return (
    <GestureDetector gesture={panGesture}>
      <Animated.View style={[styles.sheet, sheetStyle]}>
        <View style={styles.handle} />
        {children}
      </Animated.View>
    </GestureDetector>
  );
};
```

---

### Q197. What are the `Easing` functions and when do you use each?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Animations

**Answer:**
```js
import { Easing } from 'react-native-reanimated';

// Linear — constant speed (looping spinners)
withTiming(1, { easing: Easing.linear })

// Ease — slow-fast-slow (default, natural)
withTiming(1, { easing: Easing.ease })

// Ease-in — slow → fast (exits — item accelerates away)
withTiming(1, { easing: Easing.in(Easing.ease) })

// Ease-out — fast → slow (entrances — item decelerates into place)
withTiming(1, { easing: Easing.out(Easing.ease) })

// Ease-in-out — slow → fast → slow (transitions between states)
withTiming(1, { easing: Easing.inOut(Easing.ease) })

// Cubic bezier — custom curve (matches CSS cubic-bezier)
withTiming(1, { easing: Easing.bezier(0.25, 0.1, 0.25, 1.0) })

// Bounce — overshoots and bounces
withTiming(1, { duration: 600, easing: Easing.bounce })

// Elastic — spring-like overshoot
withTiming(1, { duration: 800, easing: Easing.elastic(2) })

// Practical rules:
// Entrance → Easing.out   (item decelerates into place — feels natural)
// Exit     → Easing.in    (item accelerates away)
// Transition → Easing.inOut
// Spinner  → Easing.linear
```

---

### Q198. What is `useAnimatedReaction` in Reanimated 2?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Reanimated

**Answer:**
Watches a shared value and runs a worklet side effect when it changes — like `useEffect` but on the UI thread.

```js
import { useAnimatedReaction, runOnJS } from 'react-native-reanimated';

// React to scrollY crossing a threshold
useAnimatedReaction(
  () => scrollY.value > 500,                        // watched value
  (isPast, wasPast) => {                            // reaction worklet
    if (isPast && !wasPast) {
      runOnJS(ReactNativeHapticFeedback.trigger)('impactLight');
    }
  }
);

// Derive active section from scroll position and sync to React state
useAnimatedReaction(
  () => Math.floor(scrollY.value / SECTION_HEIGHT), // selector
  (current, previous) => {
    if (current !== previous) {
      runOnJS(setActiveSectionIndex)(current);
    }
  }
);

// Trigger callback when animation completes
const scale = useSharedValue(0);
useAnimatedReaction(
  () => scale.value,
  (v) => { if (v >= 0.99) runOnJS(onAnimationComplete)(); }
);
```

---

### Q199. What is `GestureHandlerRootView` and why is it required?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Gestures

**Answer:**
```js
// Required root wrapper for react-native-gesture-handler v2
// Creates the native gesture manager — without it gestures fail silently on Android

import { GestureHandlerRootView } from 'react-native-gesture-handler';

// App.tsx — wrap at the TOP of the component tree
const App = () => (
  <GestureHandlerRootView style={{ flex: 1 }}>
    <SafeAreaProvider>
      <NavigationContainer>
        <AppNavigator />
      </NavigationContainer>
    </SafeAreaProvider>
  </GestureHandlerRootView>
);

// With Expo Router (app/_layout.tsx)
export default function RootLayout() {
  return (
    <GestureHandlerRootView style={{ flex: 1 }}>
      <Stack />
    </GestureHandlerRootView>
  );
}

// ❌ Common mistake — wrapping too deep in the tree
const Screen = () => (
  <GestureHandlerRootView>  {/* Wrong — must be at app root */}
    <DraggableContent />
  </GestureHandlerRootView>
);
```

---

### Q200. How do you implement a page indicator (dots) synced to a carousel?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Animations

**Answer:**
```js
const AnimatedDot = ({ index, scrollX, pageWidth }) => {
  const dotStyle = useAnimatedStyle(() => {
    const inputRange = [(index-1)*pageWidth, index*pageWidth, (index+1)*pageWidth];
    return {
      width:  interpolate(scrollX.value, inputRange, [8, 20, 8], Extrapolation.CLAMP),
      opacity: interpolate(scrollX.value, inputRange, [0.4, 1, 0.4], Extrapolation.CLAMP),
      backgroundColor: interpolateColor(scrollX.value, inputRange, ['#C0C0C0', '#6200EE', '#C0C0C0']),
    };
  });
  return <Animated.View style={[{ height: 8, borderRadius: 4 }, dotStyle]} />;
};

const Carousel = ({ slides }) => {
  const scrollX = useSharedValue(0);
  const { width } = useWindowDimensions();
  const scrollHandler = useAnimatedScrollHandler({ onScroll: (e) => { scrollX.value = e.contentOffset.x; } });

  return (
    <View>
      <Animated.ScrollView horizontal pagingEnabled showsHorizontalScrollIndicator={false}
        onScroll={scrollHandler} scrollEventThrottle={16}>
        {slides.map((slide, i) => (
          <View key={i} style={{ width }}><SlideContent slide={slide} /></View>
        ))}
      </Animated.ScrollView>
      <View style={{ flexDirection: 'row', justifyContent: 'center', gap: 8, paddingVertical: 12 }}>
        {slides.map((_, i) => (
          <AnimatedDot key={i} index={i} scrollX={scrollX} pageWidth={width} />
        ))}
      </View>
    </View>
  );
};
```

---

### Q201. What is the difference between `Gesture.Pan` and `Gesture.Fling`?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Gestures

**Answer:**

| | `Gesture.Pan` | `Gesture.Fling` |
|--|--------------|----------------|
| Activation | Movement distance | Fast velocity in a direction |
| Position updates | Continuous | No |
| Velocity info | At end | At activation |
| Use for | Drag, swipe, bottom sheet | Quick directional flick |

```js
// Pan — continuous position updates (drag)
const pan = Gesture.Pan()
  .onUpdate((e) => { translateX.value = e.translationX; })
  .onEnd((e) => {
    if (Math.abs(e.translationX) > 100) runOnJS(onSwipe)(e.translationX > 0 ? 'right' : 'left');
    else translateX.value = withSpring(0);
  });

// Fling — quick throw gesture (no position updates)
const fling = Gesture.Fling()
  .direction(Directions.RIGHT | Directions.LEFT)
  .onEnd(() => { runOnJS(goToNextCard)(); });
```

---

### Q202. What is `createAnimatedComponent` and when do you need it?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Reanimated

**Answer:**
Wraps any component so it can receive Reanimated animated styles and props. By default only `Animated.View`, `Animated.Text`, `Animated.Image`, `Animated.ScrollView`, `Animated.FlatList` exist.

```js
import Animated from 'react-native-reanimated';
import { Circle } from 'react-native-svg';
import LottieView from 'lottie-react-native';

// Create animated versions of any component
const AnimatedCircle = Animated.createAnimatedComponent(Circle);
const AnimatedLottie = Animated.createAnimatedComponent(LottieView);

// Animate Lottie progress on UI thread
const progress = useSharedValue(0);
const lottieProps = useAnimatedProps(() => ({ progress: progress.value }));

<AnimatedLottie source={require('./success.json')} animatedProps={lottieProps} loop={false} />

// ❌ Never create inside component — recreates every render
const BadScreen = () => {
  const AnimatedCustom = Animated.createAnimatedComponent(Custom); // bad!
};

// ✅ Create once at module level
const AnimatedCustom = Animated.createAnimatedComponent(Custom);
const GoodScreen = () => <AnimatedCustom style={style} />;
```

---

### Q203. How do you implement a notification bell shake animation?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Animations

**Answer:**
```js
const NotificationBell = ({ hasNotification }) => {
  const rotation = useSharedValue(0);
  const scale = useSharedValue(1);

  const ring = () => {
    rotation.value = withSequence(
      withTiming(15, { duration: 100 }),
      withTiming(-15, { duration: 100 }),
      withTiming(10, { duration: 80 }),
      withTiming(-10, { duration: 80 }),
      withTiming(5, { duration: 60 }),
      withSpring(0)
    );
    scale.value = withSequence(
      withTiming(1.2, { duration: 100 }),
      withSpring(1, { damping: 8 })
    );
  };

  useEffect(() => { if (hasNotification) ring(); }, [hasNotification]);

  const bellStyle = useAnimatedStyle(() => ({
    transform: [{ rotate: `${rotation.value}deg` }, { scale: scale.value }],
  }));

  return (
    <Pressable onPress={ring}>
      <Animated.View style={bellStyle}>
        <Icon name="notifications" size={28} color={hasNotification ? '#FF6B6B' : '#666'} />
      </Animated.View>
    </Pressable>
  );
};
```

---

### Q204. How do you animate a conditional component (enter + exit)?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Animations

**Answer:**
```js
// Method 1: Reanimated Layout Animations (simplest)
{show && (
  <Animated.View entering={FadeInDown.duration(300)} exiting={FadeOutDown.duration(200)}>
    <Content />
  </Animated.View>
)}

// Method 2: Keep mounted, animate opacity + pointer events
const ToggleView = ({ visible, children }) => {
  const opacity = useSharedValue(visible ? 1 : 0);
  useEffect(() => { opacity.value = withTiming(visible ? 1 : 0, { duration: 300 }); }, [visible]);

  const style = useAnimatedStyle(() => ({ opacity: opacity.value }));
  return (
    <Animated.View style={style} pointerEvents={visible ? 'auto' : 'none'}>
      {children}
    </Animated.View>
  );
};

// Method 3: Animate out, then unmount
const [mounted, setMounted] = useState(show);
const opacity = useSharedValue(show ? 1 : 0);

useEffect(() => {
  if (show) {
    setMounted(true);
    opacity.value = withTiming(1, { duration: 300 });
  } else {
    opacity.value = withTiming(0, { duration: 200 }, () => runOnJS(setMounted)(false));
  }
}, [show]);

{mounted && <Animated.View style={{ opacity }}><Content /></Animated.View>}
```

---

### Q205. How do you implement a shimmer skeleton with Reanimated 2?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Animations + UX

**Answer:**
```js
const ShimmerBox = ({ width, height, borderRadius = 4 }) => {
  const shimmer = useSharedValue(0);

  useEffect(() => {
    shimmer.value = withRepeat(withTiming(1, { duration: 1200 }), -1, true);
  }, []);

  const style = useAnimatedStyle(() => ({
    opacity: interpolate(shimmer.value, [0, 1], [0.4, 1]),
    backgroundColor: '#E0E0E0',
    width, height, borderRadius,
  }));

  return <Animated.View style={style} />;
};

// Skeleton card
const SkeletonCard = () => (
  <View style={[styles.card, { padding: 16 }]}>
    <View style={{ flexDirection: 'row', alignItems: 'center', marginBottom: 12 }}>
      <ShimmerBox width={44} height={44} borderRadius={22} />
      <View style={{ marginLeft: 10 }}>
        <ShimmerBox width={120} height={12} borderRadius={6} />
        <View style={{ height: 6 }} />
        <ShimmerBox width={80} height={10} borderRadius={5} />
      </View>
    </View>
    <ShimmerBox width="100%" height={180} borderRadius={8} />
    <View style={{ height: 8 }} />
    <ShimmerBox width="100%" height={12} borderRadius={6} />
    <View style={{ height: 6 }} />
    <ShimmerBox width="70%" height={12} borderRadius={6} />
  </View>
);
```

---

### Q206. What is `interpolateColor` in Reanimated 2?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Reanimated

**Answer:**
```js
import { interpolateColor, ColorSpace } from 'react-native-reanimated';

// Background changes with scroll
const animatedStyle = useAnimatedStyle(() => {
  const backgroundColor = interpolateColor(
    scrollY.value,
    [0, 100, 300],
    ['transparent', '#ffffff', '#6200EE']
  );
  return { backgroundColor };
});

// Theme transition
const progress = useSharedValue(0); // 0 = light, 1 = dark

const containerStyle = useAnimatedStyle(() => ({
  backgroundColor: interpolateColor(progress.value, [0, 1], ['#FFFFFF', '#121212']),
}));
const textStyle = useAnimatedStyle(() => ({
  color: interpolateColor(progress.value, [0, 1], ['#000000', '#FFFFFF']),
}));

const toggleTheme = (isDark) => { progress.value = withTiming(isDark ? 1 : 0, { duration: 400 }); };

// HSV color space — more natural transitions
interpolateColor(value, [0, 1], ['#FF0000', '#0000FF'], ColorSpace.HSV);
```

---

### Q207. How do you implement `useAnimatedKeyboard`?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Reanimated

**Answer:**
Tracks keyboard height in real time on the UI thread — frame-perfect keyboard animations.

```js
import { useAnimatedKeyboard, useAnimatedStyle, KeyboardState } from 'react-native-reanimated';

const ChatScreen = () => {
  const keyboard = useAnimatedKeyboard();

  // Input bar rises and falls perfectly in sync with keyboard
  const inputBarStyle = useAnimatedStyle(() => ({
    transform: [{ translateY: -keyboard.height.value }],
  }));

  // Content container shrinks as keyboard rises
  const contentStyle = useAnimatedStyle(() => ({
    paddingBottom: keyboard.height.value,
  }));

  // Check keyboard state
  const overlayStyle = useAnimatedStyle(() => ({
    opacity: keyboard.state.value === KeyboardState.OPEN ? 1 : 0,
  }));

  return (
    <View style={{ flex: 1 }}>
      <Animated.ScrollView style={contentStyle}>
        <MessageList />
      </Animated.ScrollView>
      <Animated.View style={[styles.inputBar, inputBarStyle]}>
        <TextInput placeholder="Type a message..." style={styles.input} />
        <SendButton />
      </Animated.View>
    </View>
  );
};
```

**Why better than `KeyboardAvoidingView`?** → Synchronises frame-by-frame on the UI thread — no jank or platform inconsistencies.

---

### Q208. How do you animate an SVG line chart drawing itself?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Animations + SVG

**Answer:**
```js
import Svg, { Path } from 'react-native-svg';
import Animated, { useSharedValue, useAnimatedProps, withTiming } from 'react-native-reanimated';

const AnimatedPath = Animated.createAnimatedComponent(Path);

const AnimatedLineChart = ({ pathD, totalLength }) => {
  const progress = useSharedValue(0);

  useEffect(() => {
    progress.value = withTiming(1, { duration: 1500 });
  }, []);

  const animatedProps = useAnimatedProps(() => ({
    strokeDashoffset: totalLength * (1 - progress.value),
  }));

  return (
    <Svg width={300} height={200}>
      <AnimatedPath
        d={pathD}
        stroke="#6200EE"
        strokeWidth={2}
        fill="none"
        strokeDasharray={totalLength}  // one full dash = total path length
        animatedProps={animatedProps}   // offset animates from full → 0 = "drawing"
      />
    </Svg>
  );
};

// How strokeDasharray / strokeDashoffset creates the effect:
// strokeDasharray={L} — creates one dash equal to full path length
// strokeDashoffset={L} — shifts dash fully off screen (path hidden)
// strokeDashoffset={0} — dash fully on screen (path visible)
// Animate offset from L → 0 = path "draws" itself
```

---

### Q209. What is `runOnUI` and when do you need it?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Reanimated

**Answer:**
```js
import { runOnUI, runOnJS } from 'react-native-reanimated';

// Execute a worklet on UI thread from JS thread
const triggerAnimation = () => {
  // Currently on JS thread
  runOnUI(() => {
    'worklet';
    // Now on UI thread
    scale.value = withSpring(1.5);
    translateX.value = withTiming(100, { duration: 300 });
  })(); // Note () — runOnUI returns a function, must be called

  // Pass arguments
  runOnUI((targetX, targetY) => {
    'worklet';
    posX.value = withSpring(targetX);
    posY.value = withSpring(targetY);
  })(100, 200);
};

// Reading shared value correctly
// ❌ Wrong — reads from JS thread, may be stale
const current = scale.value;

// ✅ Correct — read inside worklet on UI thread
runOnUI(() => {
  'worklet';
  const current = scale.value; // accurate
  runOnJS(setReactState)(current);
})();

// Common pattern: batch multiple animations from one JS event
const onSuccess = () => {
  runOnUI(() => {
    'worklet';
    checkScale.value = withSpring(1, { damping: 8 });
    confettiOpacity.value = withTiming(1);
  })();
  dispatch(markSuccess()); // JS-side effect
};
```

---

### Q210. What is `withClamp` in Reanimated 2?

**Difficulty:** 🟡 Medium | **Frequency:** Low | **Category:** Reanimated

**Answer:**
```js
import { withClamp, withSpring, withTiming } from 'react-native-reanimated';

// Wrap any animation — output is bounded between min and max
scale.value = withClamp(
  { min: 0.5, max: 2 },
  withSpring(1.5) // tries 1.5 but clamped — result depends on spring
);

// Clamp drag value
translateX.value = withClamp(
  { min: -200, max: 200 },
  e.translationX
);

// Only clamp one side
translateY.value = withClamp(
  { min: 0 },       // no max — only prevent going negative
  withSpring(0)
);
```

---

### Q211. How do you implement a toast notification with Reanimated?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** UI + Animations

**Answer:**
```js
import { FadeOut, SlideInDown } from 'react-native-reanimated';

const Toast = ({ message, type = 'info', onDismiss }) => {
  const colors = { success: '#4CAF50', error: '#F44336', info: '#2196F3', warning: '#FF9800' };
  return (
    <Animated.View
      entering={SlideInDown.springify().damping(15)}
      exiting={FadeOut.duration(200)}
      style={{
        backgroundColor: colors[type],
        borderRadius: 8, padding: 12,
        marginHorizontal: 16, marginBottom: 8,
        flexDirection: 'row', alignItems: 'center',
        elevation: 6,
      }}
    >
      <Text style={{ color: 'white', flex: 1, fontSize: 14 }}>{message}</Text>
      <Pressable onPress={onDismiss} hitSlop={8}>
        <Icon name="close" color="white" size={16} />
      </Pressable>
    </Animated.View>
  );
};

// Auto-dismiss after 3s
const useToast = () => {
  const [toasts, setToasts] = useState([]);
  const showToast = (message, type = 'info', duration = 3000) => {
    const id = Date.now();
    setToasts(prev => [...prev, { id, message, type }]);
    setTimeout(() => setToasts(prev => prev.filter(t => t.id !== id)), duration);
  };
  return { toasts, showToast };
};
```

---

### Q212. How do you implement a wave pulse animation for loading states?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Animations

**Answer:**
```js
const WavePulse = ({ size = 60, color = '#6200EE' }) => {
  const makeWave = (delay) => {
    const scale = useSharedValue(0);
    const opacity = useSharedValue(1);
    useEffect(() => {
      scale.value = withDelay(delay, withRepeat(withTiming(1, { duration: 2000 }), -1, false));
      opacity.value = withDelay(delay, withRepeat(withTiming(0, { duration: 2000 }), -1, false));
    }, []);
    return useAnimatedStyle(() => ({
      transform: [{ scale: scale.value }],
      opacity: opacity.value,
      position: 'absolute',
      width: size, height: size, borderRadius: size / 2,
      backgroundColor: color,
    }));
  };

  const wave1 = makeWave(0);
  const wave2 = makeWave(600);
  const wave3 = makeWave(1200);

  return (
    <View style={{ width: size, height: size, alignItems: 'center', justifyContent: 'center' }}>
      <Animated.View style={wave1} />
      <Animated.View style={wave2} />
      <Animated.View style={wave3} />
      <View style={{ width: size * 0.4, height: size * 0.4, borderRadius: size * 0.2, backgroundColor: color }} />
    </View>
  );
};
```

---

### Q213. What is `reduceMotion` accessibility for animations?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Accessibility

**Answer:**
```js
import { AccessibilityInfo } from 'react-native';
import { ReduceMotion } from 'react-native-reanimated';

// Check system setting
const [reduceMotion, setReduceMotion] = useState(false);
useEffect(() => {
  AccessibilityInfo.isReduceMotionEnabled().then(setReduceMotion);
  const sub = AccessibilityInfo.addEventListener('reduceMotionChanged', setReduceMotion);
  return () => sub.remove();
}, []);

// Respect in animations
const duration = reduceMotion ? 0 : 300;
opacity.value = withTiming(1, { duration });

// Reanimated built-in support
value.value = withTiming(1, {
  duration: 500,
  reduceMotion: ReduceMotion.System, // auto-respects system setting
  // ReduceMotion.Always — always reduce
  // ReduceMotion.Never  — always animate
});
```

---

### Q214. How do you debug animation performance issues?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Performance

**Answer:**
```js
// 1. Dev menu → Show Perf Monitor
// Goal: UI FPS = 60 continuously (JS FPS can dip without issue if using Reanimated)

// 2. Identify JS-thread vs UI-thread animations
// Reanimated animations → UI FPS stays 60 even when JS FPS drops
// Animated API with useNativeDriver: false → drops UI FPS when JS busy

// 3. Why-did-you-render — find re-renders during animation
import './wdyr';

// 4. React DevTools Profiler (Flipper) — find components rendering on every frame

// 5. Common fixes:
// ❌ Anonymous functions in animated component props
<Animated.View style={{ opacity }} onPress={() => doSomething()} />  // re-render
// ✅ useCallback
const handlePress = useCallback(() => doSomething(), []);

// ❌ useNativeDriver: false for transform
// ✅ Always useNativeDriver: true for transform + opacity

// ❌ Heavy computation in useAnimatedStyle
const style = useAnimatedStyle(() => {
  heavyComputation(); // runs every frame on UI thread!
  return { opacity: value.value };
});
// ✅ useDerivedValue for pre-computation
const computed = useDerivedValue(() => heavyComputation());

// 6. 120fps ProMotion — budget = 8.33ms per frame instead of 16.67ms
// Even more reason to keep animations on UI thread
```

---

### Q215. What is `Animated.parallel` vs Reanimated simultaneous values?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Animations

**Answer:**
```js
// Old Animated API — Animated.parallel runs multiple animations simultaneously
import { Animated } from 'react-native';

Animated.parallel([
  Animated.timing(opacity, { toValue: 1, duration: 300, useNativeDriver: true }),
  Animated.timing(translateY, { toValue: 0, duration: 300, useNativeDriver: true }),
  Animated.timing(scale, { toValue: 1, duration: 300, useNativeDriver: true }),
]).start();

// Animated.sequence — one after another
Animated.sequence([
  Animated.timing(opacity, { toValue: 1, duration: 200, useNativeDriver: true }),
  Animated.timing(translateY, { toValue: 0, duration: 200, useNativeDriver: true }),
]).start();

// Animated.stagger — staggered start times
Animated.stagger(100, items.map(item =>
  Animated.timing(item, { toValue: 1, duration: 300, useNativeDriver: true })
)).start();

// Reanimated 2 equivalent — just call multiple simultaneously
// They all run on the UI thread, no composition needed
const startAnimation = () => {
  opacity.value = withTiming(1, { duration: 300 });      // simultaneously
  translateY.value = withSpring(0, { damping: 15 });     // simultaneously
  scale.value = withTiming(1, { duration: 300 });        // simultaneously
};
// Stagger equivalent
items.forEach((item, i) => {
  item.opacity.value = withDelay(i * 100, withTiming(1));
});
```

---

### Q216. How do you implement a progress bar animation?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Animations + UI

**Answer:**
```js
const AnimatedProgressBar = ({ progress, color = '#6200EE', height = 8 }) => {
  const width = useSharedValue(0);

  useEffect(() => {
    width.value = withTiming(progress, { duration: 400, easing: Easing.out(Easing.ease) });
  }, [progress]);

  // Note: width % requires useNativeDriver: false (layout property)
  // Use transform: scaleX instead for native driver
  const barStyle = useAnimatedStyle(() => ({
    transform: [{ scaleX: width.value }],    // useNativeDriver: true compatible!
    transformOrigin: 'left',
  }));

  // Alternative: width animation (useNativeDriver: false)
  const barStyle2 = useAnimatedStyle(() => ({
    width: `${width.value * 100}%`,
  }));

  return (
    <View style={{ width: '100%', height, backgroundColor: '#E0E0E0', borderRadius: height / 2, overflow: 'hidden' }}>
      <Animated.View style={[{ height: '100%', backgroundColor: color, borderRadius: height / 2 }, barStyle2]} />
    </View>
  );
};
```

---

### Q217. How do you implement a swipe-to-delete row (iOS style)?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Gestures + UI

**Answer:**
```js
import { Swipeable } from 'react-native-gesture-handler';

const SwipeableRow = ({ item, onDelete, onArchive }) => {
  const swipeRef = useRef(null);

  const renderRightActions = (progress, dragX) => {
    const scale = dragX.interpolate({ inputRange: [-80, 0], outputRange: [1, 0.5], extrapolate: 'clamp' });
    return (
      <View style={{ width: 160, flexDirection: 'row' }}>
        <Pressable style={{ width: 80, backgroundColor: '#2196F3', justifyContent: 'center', alignItems: 'center' }}
          onPress={() => { onArchive(item.id); swipeRef.current?.close(); }}>
          <Animated.View style={{ transform: [{ scale }] }}>
            <Icon name="archive" color="white" size={24} />
            <Text style={{ color: 'white', fontSize: 11 }}>Archive</Text>
          </Animated.View>
        </Pressable>
        <Pressable style={{ width: 80, backgroundColor: '#F44336', justifyContent: 'center', alignItems: 'center' }}
          onPress={() => onDelete(item.id)}>
          <Animated.View style={{ transform: [{ scale }] }}>
            <Icon name="delete" color="white" size={24} />
            <Text style={{ color: 'white', fontSize: 11 }}>Delete</Text>
          </Animated.View>
        </Pressable>
      </View>
    );
  };

  return (
    <Swipeable ref={swipeRef} renderRightActions={renderRightActions} rightThreshold={40} friction={2}>
      <View style={styles.row}><Text>{item.title}</Text></View>
    </Swipeable>
  );
};
```

---

### Q218. What is the `Keyframe` API in Reanimated 2?

**Difficulty:** 🔴 Hard | **Frequency:** Low | **Category:** Reanimated

**Answer:**
`Keyframe` allows defining multi-step animations (like CSS `@keyframes`) for entering/exiting components.

```js
import { Keyframe } from 'react-native-reanimated';

// Define keyframe animation
const PopIn = new Keyframe({
  0: {
    opacity: 0,
    transform: [{ scale: 0.3 }, { rotate: '-15deg' }],
  },
  60: {
    opacity: 1,
    transform: [{ scale: 1.15 }, { rotate: '5deg' }],
    easing: Easing.out(Easing.ease),
  },
  100: {
    opacity: 1,
    transform: [{ scale: 1 }, { rotate: '0deg' }],
  },
}).duration(600);

const PopOut = new Keyframe({
  0: { opacity: 1, transform: [{ scale: 1 }] },
  100: { opacity: 0, transform: [{ scale: 0.3 }] },
}).duration(300);

// Use like any Layout Animation
<Animated.View entering={PopIn} exiting={PopOut}>
  <SuccessBadge />
</Animated.View>

// Useful for: payment success screens, achievement unlocks,
// notification badges appearing, modal entrances with personality
```

---

### Q219. How do you implement scroll-to-top on tab re-press?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Navigation + Animations

**Answer:**
```js
import { useScrollToTop } from '@react-navigation/native';

// Method 1: useScrollToTop (React Navigation built-in)
const FeedScreen = () => {
  const ref = useRef(null);
  useScrollToTop(ref);  // scrolls to top when tab is pressed while already on the screen

  return (
    <Animated.FlatList
      ref={ref}
      data={feedData}
      renderItem={renderItem}
    />
  );
};

// Method 2: Manual with animation
const FeedScreen = () => {
  const scrollY = useSharedValue(0);
  const flatListRef = useRef(null);
  const navigation = useNavigation();

  useEffect(() => {
    const unsubscribe = navigation.addListener('tabPress', (e) => {
      if (navigation.isFocused()) {
        // Animate scroll to top
        flatListRef.current?.scrollToOffset({ offset: 0, animated: true });
      }
    });
    return unsubscribe;
  }, [navigation]);

  return (
    <Animated.FlatList ref={flatListRef} data={feedData} renderItem={renderItem} />
  );
};
```

---

### Q220. What is `useSharedValue` vs `useRef` for storing values in Reanimated?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Reanimated

**Answer:**

| | `useSharedValue` | `useRef` |
|--|-----------------|---------|
| Thread | Both JS + UI thread | JS thread only |
| Triggers re-render | No | No |
| Animatable | Yes (withSpring, etc.) | No |
| Accessible in worklets | Yes | No |
| Use for | Animated values | DOM refs, timers, flags |

```js
// useSharedValue — for anything that needs to animate or be read in a worklet
const translateX = useSharedValue(0); // animatable
const isGestureActive = useSharedValue(false); // readable in gesture worklets

// useRef — for stable references that don't animate
const flatListRef = useRef(null);      // component reference
const intervalRef = useRef(null);      // timer ID
const isMountedRef = useRef(true);     // flag

// ❌ Wrong: useRef for an animation value
const translateX = useRef(new Animated.Value(0)).current; // old API only

// ❌ Wrong: useSharedValue for a DOM ref
const inputRef = useSharedValue(null); // can't attach to TextInput
```

---

### Q221. How do you implement a confetti burst for a success screen?

**Difficulty:** 🔴 Hard | **Frequency:** Low | **Category:** Animations

**Answer:**
```js
const COLORS = ['#FF6B6B', '#FFE66D', '#4ECDC4', '#45B7D1', '#96CEB4', '#6200EE'];
const PIECE_COUNT = 60;

const ConfettiPiece = ({ containerWidth, containerHeight }) => {
  const startX = Math.random() * containerWidth;
  const translateX = useSharedValue(startX);
  const translateY = useSharedValue(-20);
  const rotate = useSharedValue(0);
  const opacity = useSharedValue(1);
  const color = COLORS[Math.floor(Math.random() * COLORS.length)];
  const delay = Math.random() * 400;
  const duration = 2000 + Math.random() * 1000;
  const drift = (Math.random() - 0.5) * 200;
  const isCircle = Math.random() > 0.5;

  useEffect(() => {
    translateY.value = withDelay(delay, withTiming(containerHeight + 20, { duration, easing: Easing.linear }));
    translateX.value = withDelay(delay, withTiming(startX + drift, { duration }));
    rotate.value = withDelay(delay, withTiming(720, { duration, easing: Easing.linear }));
    opacity.value = withDelay(delay + duration * 0.7, withTiming(0, { duration: duration * 0.3 }));
  }, []);

  const style = useAnimatedStyle(() => ({
    transform: [{ translateX: translateX.value }, { translateY: translateY.value }, { rotate: `${rotate.value}deg` }],
    opacity: opacity.value,
    position: 'absolute',
    width: 8, height: 8,
    borderRadius: isCircle ? 4 : 0,
    backgroundColor: color,
  }));

  return <Animated.View style={style} />;
};

const Confetti = ({ active }) => {
  const { width, height } = useWindowDimensions();
  if (!active) return null;
  return (
    <View style={StyleSheet.absoluteFillObject} pointerEvents="none">
      {Array.from({ length: PIECE_COUNT }).map((_, i) => (
        <ConfettiPiece key={i} containerWidth={width} containerHeight={height} />
      ))}
    </View>
  );
};
```

---

### Q222. How do you animate between two screens using native stack transitions?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Navigation + Animations

**Answer:**
```js
import { createNativeStackNavigator } from '@react-navigation/native-stack';
const Stack = createNativeStackNavigator();

<Stack.Navigator>
  <Stack.Screen name="Home" component={HomeScreen} />

  {/* Standard card push */}
  <Stack.Screen name="Detail" component={DetailScreen}
    options={{ animation: 'slide_from_right' }} />

  {/* Modal slide up */}
  <Stack.Screen name="Modal" component={ModalScreen}
    options={{ presentation: 'modal', animation: 'slide_from_bottom', headerShown: false }} />

  {/* Fade transition */}
  <Stack.Screen name="Splash" component={SplashScreen}
    options={{ animation: 'fade', headerShown: false }} />

  {/* Transparent modal overlay */}
  <Stack.Screen name="ImageViewer" component={ImageViewerScreen}
    options={{ presentation: 'transparentModal', animation: 'fade', headerShown: false }} />

  {/* No animation (instant) */}
  <Stack.Screen name="Login" component={LoginScreen}
    options={{ animation: 'none', headerShown: false }} />
</Stack.Navigator>

// Available animation values:
// 'default', 'fade', 'flip', 'simple_push', 'slide_from_right',
// 'slide_from_left', 'slide_from_bottom', 'fade_from_bottom', 'none'
```

---

### Q223. How do you implement a rubber band overscroll effect?

**Difficulty:** 🔴 Hard | **Frequency:** Low | **Category:** Animations

**Answer:**
```js
const RUBBER_FACTOR = 0.55;

const rubberBand = (offset, dim) => {
  'worklet';
  const c = dim * RUBBER_FACTOR;
  return (offset * c) / (Math.abs(offset) + c);
};

const RubberScrollView = ({ children }) => {
  const scrollY = useSharedValue(0);
  const { height } = useWindowDimensions();

  const scrollHandler = useAnimatedScrollHandler({
    onScroll: (e) => { scrollY.value = e.contentOffset.y; },
  });

  // Apply rubber band resistance to overscroll
  const containerStyle = useAnimatedStyle(() => {
    const overscroll = Math.min(scrollY.value, 0); // negative = over-pulled top
    const resistance = overscroll < 0 ? rubberBand(overscroll, height) - overscroll : 0;
    return { transform: [{ translateY: resistance }] };
  });

  return (
    <Animated.View style={containerStyle}>
      <Animated.ScrollView onScroll={scrollHandler} scrollEventThrottle={16} bounces>
        {children}
      </Animated.ScrollView>
    </Animated.View>
  );
};
```

---

### Q224. What is `Animated.Value` vs Reanimated `useSharedValue` — key differences?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Animations

**Answer:**

| | `Animated.Value` (core RN) | `useSharedValue` (Reanimated 2) |
|--|--------------------------|--------------------------------|
| Thread | JS thread | Both JS + UI thread |
| Access in worklets | No | Yes |
| Frame drops when JS busy | Yes (with `useNativeDriver: false`) | Never (UI thread) |
| API style | Object-based `.interpolate()` | Functional `interpolate()` in worklet |
| Setup | No Babel plugin needed | Requires Reanimated Babel plugin |

```js
// Animated.Value (old API) — still works, still used
const opacity = useRef(new Animated.Value(0)).current;
Animated.timing(opacity, { toValue: 1, duration: 300, useNativeDriver: true }).start();
<Animated.View style={{ opacity }} />

// Reanimated useSharedValue (new API — preferred)
const opacity = useSharedValue(0);
opacity.value = withTiming(1, { duration: 300 });
const style = useAnimatedStyle(() => ({ opacity: opacity.value }));
<Animated.View style={style} />

// When to still use old Animated API:
// - Simple, non-gesture animations (fade in on mount)
// - Team not using Reanimated
// - Libraries that depend on Animated.Value (most are migrating)
```

---

### Q225. How do you implement a stretchy header (image that grows on overscroll)?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Animations

**Answer:**
```js
const HEADER_HEIGHT = 250;

const StretchyHeader = ({ imageUri, children }) => {
  const scrollY = useSharedValue(0);
  const scrollHandler = useAnimatedScrollHandler({
    onScroll: (e) => { scrollY.value = e.contentOffset.y; },
  });

  // Image stretches when user overscrolls at top (negative scroll)
  const imageStyle = useAnimatedStyle(() => {
    const overscroll = Math.max(0, -scrollY.value); // positive when pulled down
    return {
      height: HEADER_HEIGHT + overscroll,
      transform: [{
        // Compensate position so top stays fixed
        translateY: scrollY.value < 0 ? scrollY.value / 2 : 0,
      }],
    };
  });

  // Header image parallax — moves slower than scroll
  const imageTranslateStyle = useAnimatedStyle(() => ({
    transform: [{
      translateY: scrollY.value > 0 ? scrollY.value * 0.5 : 0,
    }],
  }));

  return (
    <View style={{ flex: 1 }}>
      <Animated.View style={[{ overflow: 'hidden' }, imageStyle]}>
        <Animated.Image
          source={{ uri: imageUri }}
          style={[{ width: '100%', height: HEADER_HEIGHT + 100 }, imageTranslateStyle]}
          resizeMode="cover"
        />
      </Animated.View>
      <Animated.ScrollView
        onScroll={scrollHandler}
        scrollEventThrottle={16}
        style={StyleSheet.absoluteFillObject}
        contentContainerStyle={{ paddingTop: HEADER_HEIGHT }}
      >
        {children}
      </Animated.ScrollView>
    </View>
  );
};
```

---

### Q226. What is the `Gesture.LongPress` and how do you combine it with Pan?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Gestures

**Answer:**
```js
import { Gesture, GestureDetector } from 'react-native-gesture-handler';

// Long press to activate, then pan to drag (common for sortable lists)
const SortableItem = ({ item }) => {
  const scale = useSharedValue(1);
  const translateX = useSharedValue(0);
  const translateY = useSharedValue(0);
  const isActive = useSharedValue(false);

  const longPress = Gesture.LongPress()
    .minDuration(400) // activate after 400ms
    .onStart(() => {
      isActive.value = true;
      scale.value = withSpring(1.08);
      runOnJS(ReactNativeHapticFeedback.trigger)('impactMedium');
    });

  const pan = Gesture.Pan()
    .manualActivation(true)   // won't activate on its own — waits for long press
    .onUpdate((e) => {
      if (isActive.value) {
        translateX.value = e.translationX;
        translateY.value = e.translationY;
      }
    })
    .onEnd(() => {
      isActive.value = false;
      scale.value = withSpring(1);
      translateX.value = withSpring(0);
      translateY.value = withSpring(0);
      runOnJS(onSortEnd)();
    });

  // Long press activates, then pan takes over
  const composed = Gesture.Simultaneous(longPress, pan);

  const itemStyle = useAnimatedStyle(() => ({
    transform: [
      { translateX: translateX.value },
      { translateY: translateY.value },
      { scale: scale.value },
    ],
    zIndex: isActive.value ? 10 : 1,
    elevation: isActive.value ? 8 : 1,
  }));

  return (
    <GestureDetector gesture={composed}>
      <Animated.View style={[styles.item, itemStyle]}>
        <Text>{item.title}</Text>
        <Icon name="menu" color="#999" size={20} />
      </Animated.View>
    </GestureDetector>
  );
};
```

---

### Q227. How do you implement a morphing FAB (speed-dial) animation?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Animations + UI

**Answer:**
```js
const SpeedDial = ({ actions }) => {
  const [open, setOpen] = useState(false);
  const rotation = useSharedValue(0);
  const actionValues = actions.map(() => ({
    scale: useSharedValue(0),
    opacity: useSharedValue(0),
    translateY: useSharedValue(20),
  }));

  const toggle = () => {
    const opening = !open;
    setOpen(opening);
    rotation.value = withSpring(opening ? 1 : 0, { damping: 15 });

    actions.forEach((_, i) => {
      const delay = opening ? i * 60 : (actions.length - 1 - i) * 40;
      actionValues[i].scale.value = withDelay(delay, withSpring(opening ? 1 : 0));
      actionValues[i].opacity.value = withDelay(delay, withTiming(opening ? 1 : 0));
      actionValues[i].translateY.value = withDelay(delay, withSpring(opening ? 0 : 20));
    });
  };

  const fabIconStyle = useAnimatedStyle(() => ({
    transform: [{ rotate: `${rotation.value * 45}deg` }],
  }));

  return (
    <View style={{ position: 'absolute', bottom: 24, right: 24, alignItems: 'center' }}>
      {/* Action buttons */}
      {actions.map((action, i) => {
        const style = useAnimatedStyle(() => ({
          transform: [{ scale: actionValues[i].scale.value }, { translateY: actionValues[i].translateY.value }],
          opacity: actionValues[i].opacity.value,
          marginBottom: 12,
        }));
        return (
          <Animated.View key={i} style={style}>
            <Pressable style={[styles.miniButton, { backgroundColor: action.color }]} onPress={() => { toggle(); action.onPress(); }}>
              <Icon name={action.icon} color="white" size={20} />
            </Pressable>
          </Animated.View>
        );
      })}

      {/* Main FAB */}
      <Pressable onPress={toggle} style={styles.fab}>
        <Animated.View style={fabIconStyle}>
          <Icon name="add" color="white" size={28} />
        </Animated.View>
      </Pressable>
    </View>
  );
};
```

---

### Q228. How do you implement a stepper/wizard with animated transitions?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Animations + UI

**Answer:**
```js
const Stepper = ({ steps }) => {
  const [currentStep, setCurrentStep] = useState(0);
  const slideX = useSharedValue(0);
  const { width } = useWindowDimensions();

  const goToStep = (step) => {
    const direction = step > currentStep ? 1 : -1;
    // Slide out
    slideX.value = withTiming(-direction * width, { duration: 200, easing: Easing.in(Easing.ease) }, () => {
      runOnJS(setCurrentStep)(step);
      // Slide in from opposite direction
      slideX.value = direction * width;
      slideX.value = withSpring(0, { damping: 20 });
    });
  };

  const contentStyle = useAnimatedStyle(() => ({
    transform: [{ translateX: slideX.value }],
  }));

  return (
    <View style={{ flex: 1 }}>
      {/* Step indicator */}
      <View style={{ flexDirection: 'row', justifyContent: 'center', paddingVertical: 16, gap: 8 }}>
        {steps.map((_, i) => (
          <Animated.View key={i} style={{
            width: i === currentStep ? 24 : 8, height: 8, borderRadius: 4,
            backgroundColor: i <= currentStep ? '#6200EE' : '#E0E0E0',
          }} entering={FadeIn} layout={LinearTransition.springify()} />
        ))}
      </View>

      {/* Animated step content */}
      <Animated.View style={[{ flex: 1 }, contentStyle]}>
        {steps[currentStep].component}
      </Animated.View>

      {/* Navigation */}
      <View style={{ flexDirection: 'row', justifyContent: 'space-between', padding: 16 }}>
        {currentStep > 0 && <Button title="Back" onPress={() => goToStep(currentStep - 1)} />}
        {currentStep < steps.length - 1
          ? <Button title="Next" onPress={() => goToStep(currentStep + 1)} />
          : <Button title="Submit" onPress={onSubmit} />
        }
      </View>
    </View>
  );
};
```

---

### Q229. What is `withTiming` vs `withSpring` — when to choose each?

**Difficulty:** 🟡 Medium | **Frequency:** Very High | **Category:** Reanimated

**Answer:**

| Scenario | Use | Why |
|----------|-----|-----|
| Modal appear/disappear | `withTiming` | Precise, predictable duration |
| Button press feedback | `withSpring` | Natural, responsive feel |
| Screen transition | `withTiming` | Consistent UX timing |
| Drag release snap | `withSpring` | Natural deceleration |
| Loading/progress bar | `withTiming` | Controlled pacing |
| Bounce on arrive | `withSpring` | Physics-based feel |
| Tab switch | `withTiming` | Snappy and precise |
| Zoom pinch release | `withSpring` | Bouncy snap back |

```js
// withTiming — you control WHEN it finishes
scale.value = withTiming(1, { duration: 300 }); // exactly 300ms

// withSpring — finishes when physics settle
scale.value = withSpring(1, {
  damping: 15,    // low damping = more bouncy
  stiffness: 150, // high stiffness = faster
  // Duration is NOT fixed — spring settles naturally
});

// Hybrid: spring with timing fallback
// Use withSpring with overshootClamping for spring without overshoot
scale.value = withSpring(1, { overshootClamping: true }); // no bounce, but spring velocity curve
```

---

### Q230. How do you implement a loading spinner using Reanimated 2?

**Difficulty:** 🟢 Easy | **Frequency:** Medium | **Category:** Animations

**Answer:**
```js
import Animated, {
  useSharedValue, useAnimatedStyle,
  withRepeat, withTiming, Easing, cancelAnimation,
} from 'react-native-reanimated';

const Spinner = ({ size = 40, color = '#6200EE', strokeWidth = 3 }) => {
  const rotation = useSharedValue(0);

  useEffect(() => {
    rotation.value = withRepeat(
      withTiming(360, { duration: 900, easing: Easing.linear }),
      -1,   // infinite
      false // don't reverse
    );
    return () => cancelAnimation(rotation);
  }, []);

  const spinStyle = useAnimatedStyle(() => ({
    transform: [{ rotate: `${rotation.value}deg` }],
  }));

  return (
    <Animated.View style={spinStyle}>
      <Svg width={size} height={size}>
        {/* Background track */}
        <Circle
          cx={size/2} cy={size/2} r={(size - strokeWidth) / 2}
          stroke={`${color}33`} strokeWidth={strokeWidth} fill="none"
        />
        {/* Spinning arc */}
        <Circle
          cx={size/2} cy={size/2} r={(size - strokeWidth) / 2}
          stroke={color} strokeWidth={strokeWidth} fill="none"
          strokeDasharray={`${Math.PI * (size - strokeWidth) * 0.75} ${Math.PI * (size - strokeWidth) * 0.25}`}
          strokeLinecap="round"
        />
      </Svg>
    </Animated.View>
  );
};

// Usage
<Spinner size={48} color="#6200EE" />
<Spinner size={24} color="white" strokeWidth={2} />
```

---

## Sections Overview (Q231–Q500)

| Section | Questions | Topics |
|---------|-----------|--------|
| Performance | Q231–Q290 | Memory leaks, profiling, bundle optimisation |
| Native Modules | Q291–Q340 | Writing custom native modules (iOS + Android) |
| Expo vs CLI | Q341–Q370 | Managed vs bare, EAS Build, Expo Go |
| Testing | Q371–Q420 | Unit, integration, e2e (Detox), mocking |
| Debugging | Q421–Q450 | Flipper, Hermes debugger, crash reporting |
| Storage & Permissions | Q451–Q480 | AsyncStorage, Keychain, permission flows |
| Miscellaneous | Q481–Q500 | Accessibility, internationalisation, misc APIs |

---

> 💡 **Tip for GitHub:** Add a `## Table of Contents` section at the top with anchor links to each question for easy navigation.

---

*Part 01 of 8 — [← Back to Part README](./README.md) · [← Main README](../README.md)*

## Section 7: Performance (Q231–Q290)

---

### Q231. What is the React Native performance bottleneck model?

**Difficulty:** 🔴 Hard | **Frequency:** Very High | **Category:** Architecture + Performance

**Answer:**
React Native has three threads. Performance problems happen when any of them are starved:

| Thread | Responsibility | Bottleneck symptom |
|--------|---------------|-------------------|
| **JS Thread** | Business logic, React rendering, state updates | Slow UI response, dropped interaction frames |
| **UI Thread** (Main) | Native rendering, layout, touch events | Frozen scrolling, unresponsive gestures |
| **Shadow Thread** | Yoga layout calculations | Slow layout on complex screens |

```
JS Thread → Bridge (old) / JSI (new) → UI Thread
              ↑ bottleneck in old arch
```

**Old Architecture bottleneck:** Every prop update crosses the bridge (async, serialised JSON). The bridge is a shared communication channel — heavy JS work blocks UI updates.

**New Architecture (JSI):** JS talks directly to C++ via JSI — synchronous, no serialisation, no bridge queue. TurboModules call native code directly. Fabric renders synchronously on the UI thread.

```js
// Diagnosing which thread is causing drops
// Dev menu → Show Perf Monitor → watch JS FPS vs UI FPS

// JS FPS drops, UI FPS = 60:
// → JS thread overloaded (heavy computation, excessive re-renders)
// Fix: memoisation, move work off JS, InteractionManager

// UI FPS drops regardless:
// → Layout thrashing, overdraw, animations with useNativeDriver: false
// Fix: Reanimated, optimise layout, reduce overdraw

// Both drop:
// → Heavy work on both threads simultaneously
// Fix: audit both JS logic and native operations
```

**Follow-up:** What causes UI FPS to drop even with Reanimated? → Shadow thread layout thrashing from rapidly changing `width`/`height` (layout properties can't be animated natively), or excessive overdraw from transparency layers.

---

### Q232. What is `useNativeDriver` and when can you use it?

**Difficulty:** 🟡 Medium | **Frequency:** Very High | **Category:** Performance

**Answer:**
`useNativeDriver: true` runs the animation entirely on the UI thread — no JS involvement per frame. This means animations continue smoothly even when the JS thread is busy.

```js
import { Animated } from 'react-native';

// ✅ useNativeDriver: true — supported properties
// ONLY works with: transform, opacity (and some others on specific platforms)
Animated.timing(opacity, {
  toValue: 1,
  duration: 300,
  useNativeDriver: true,  // runs on UI thread
}).start();

Animated.timing(scale, {
  toValue: 1.5,
  duration: 200,
  useNativeDriver: true,  // transform is supported
}).start();

// ❌ useNativeDriver: false — REQUIRED for layout properties
Animated.timing(height, {
  toValue: 200,
  duration: 300,
  useNativeDriver: false, // layout properties can't be animated natively
}).start();

Animated.timing(backgroundColor, {
  toValue: 1,
  duration: 300,
  useNativeDriver: false, // color animations not natively supported
}).start();

// Properties that support useNativeDriver: true
// transform (translateX, translateY, scale, rotate, etc.)
// opacity
// elevation (Android)

// Properties that DON'T support useNativeDriver: true
// width, height, top, left, right, bottom (layout props)
// backgroundColor, color (style props)
// padding, margin, borderWidth, borderRadius

// Reanimated 2 — useNativeDriver is irrelevant
// Everything runs on UI thread via JSI automatically
// No flag needed
```

**Interview tip:** Forgetting `useNativeDriver: true` on transform/opacity animations is the most common cause of janky animations at senior level.

---

### Q233. What are the most common causes of React Native performance issues?

**Difficulty:** 🟡 Medium | **Frequency:** Very High | **Category:** Performance

**Answer:**
```
1. Unnecessary re-renders
2. FlatList not optimised
3. Animations on JS thread (no useNativeDriver)
4. Heavy computation on JS thread during interaction
5. Memory leaks (event listeners, async operations not cleaned up)
6. Large bundle size (slow startup)
7. Images not cached or oversized
8. Excessive console.log in production
9. Deep component trees without memoisation
10. Bridge overload (old arch) — too many native calls
```

```js
// 1. Unnecessary re-renders — use React DevTools Profiler to find
// Fix: React.memo, useMemo, useCallback, proper key strategy

// 2. FlatList — most impactful for list-heavy apps (ERP)
<FlatList
  removeClippedSubviews={true}     // unmount off-screen items
  maxToRenderPerBatch={10}          // items per batch render
  updateCellsBatchingPeriod={50}    // ms between batches
  windowSize={5}                    // viewport windows to keep rendered
  initialNumToRender={10}           // first batch size
  getItemLayout={getItemLayout}     // skip dynamic height measurement
/>

// 3. Animations — always useNativeDriver: true (or Reanimated)

// 4. Heavy computation — defer or move off-thread
InteractionManager.runAfterInteractions(() => {
  loadHeavyData(); // runs after animation finishes
});

// 5. Memory leaks — always clean up
useEffect(() => {
  const sub = someEvent.addListener(handler);
  return () => sub.remove(); // cleanup on unmount
}, []);

// 8. Remove console.log in prod
// babel-plugin-transform-remove-console — removes all console calls
```

---

### Q234. How do you prevent unnecessary re-renders with `React.memo`, `useMemo`, `useCallback`?

**Difficulty:** 🟡 Medium | **Frequency:** Very High | **Category:** Performance

**Answer:**
```js
// React.memo — prevents re-render if props haven't changed (shallow comparison)
const ExpensiveListItem = React.memo(({ item, onPress }) => {
  console.log('Rendered:', item.id); // should only log when item or onPress changes
  return (
    <Pressable onPress={() => onPress(item.id)}>
      <Text>{item.title}</Text>
    </Pressable>
  );
});

// ❌ Problem: new function reference on every parent render breaks React.memo
const ParentList = () => {
  const handlePress = (id) => navigate('Detail', { id }); // new ref each render!
  return <FlatList renderItem={({ item }) => <ExpensiveListItem item={item} onPress={handlePress} />} />;
};

// ✅ Fix: useCallback stabilises the reference
const ParentList = () => {
  const navigation = useNavigation();
  const handlePress = useCallback((id) => {
    navigation.navigate('Detail', { id });
  }, [navigation]); // only changes if navigation changes

  const renderItem = useCallback(({ item }) => (
    <ExpensiveListItem item={item} onPress={handlePress} />
  ), [handlePress]);

  return <FlatList renderItem={renderItem} data={items} />;
};

// useMemo — memoises expensive computed values
const sortedEmployees = useMemo(() => {
  return employees
    .filter(e => e.department === selectedDept)
    .sort((a, b) => a.name.localeCompare(b.name));
}, [employees, selectedDept]); // only recomputes when deps change

// Custom comparison for React.memo
const areEqual = (prevProps, nextProps) => {
  return prevProps.item.id === nextProps.item.id &&
         prevProps.item.updatedAt === nextProps.item.updatedAt;
};
export default React.memo(ListItem, areEqual);

// When NOT to memoize:
// - Simple components with cheap renders
// - Components that always receive new props anyway
// - Memoization itself has a cost (comparison) — don't over-apply
```

---

### Q235. What is `getItemLayout` in FlatList and why does it matter?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** FlatList Performance

**Answer:**
By default FlatList measures each item's height dynamically. `getItemLayout` tells FlatList the exact dimensions without measurement — enabling instant scroll-to-index and dramatically faster rendering.

```js
const ITEM_HEIGHT = 72;
const SEPARATOR_HEIGHT = 1;

// For fixed-height items — precompute layout
<FlatList
  data={employees}
  getItemLayout={(data, index) => ({
    length: ITEM_HEIGHT + SEPARATOR_HEIGHT,
    offset: (ITEM_HEIGHT + SEPARATOR_HEIGHT) * index,
    index,
  })}
  renderItem={renderItem}
/>

// For items with headers/sections
const HEADER_HEIGHT = 48;
<FlatList
  data={data}
  ListHeaderComponent={<SectionHeader />}
  getItemLayout={(data, index) => ({
    length: ITEM_HEIGHT,
    offset: HEADER_HEIGHT + ITEM_HEIGHT * index,
    index,
  })}
/>

// Benefits of getItemLayout:
// 1. FlatList can jump to any index instantly (scrollToIndex)
// 2. No layout measurement overhead on scroll
// 3. Better initial render performance

// Using scrollToIndex only works reliably WITH getItemLayout
flatListRef.current?.scrollToIndex({
  index: targetIndex,
  animated: true,
  viewPosition: 0.5, // 0 = top, 0.5 = center, 1 = bottom of viewport
});

// When items have dynamic height — use onLayout measurement + cache
const itemHeights = useRef({});
const getItemLayout = useCallback((data, index) => {
  const height = itemHeights.current[index] || ESTIMATED_ITEM_HEIGHT;
  const offset = data.slice(0, index).reduce((sum, _, i) =>
    sum + (itemHeights.current[i] || ESTIMATED_ITEM_HEIGHT), 0
  );
  return { length: height, offset, index };
}, []);
```

---

### Q236. What is `removeClippedSubviews` and should you always enable it?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** FlatList Performance

**Answer:**
`removeClippedSubviews={true}` detaches native views of off-screen FlatList items from the view hierarchy, reducing memory usage and improving scroll performance. The items remain in the React tree but their native backing views are detached.

```js
<FlatList
  data={largeDataset}
  removeClippedSubviews={true}    // detach off-screen native views
  renderItem={renderItem}
/>
```

**Caveats — don't always blindly enable:**
```js
// ❌ Known issue 1: content clipping bugs on Android
// Items near the edges may occasionally clip or flash
// If you see this, disable it

// ❌ Known issue 2: breaks some absolute-positioned overlays
// If list items have tooltips/dropdowns that extend beyond the item bounds,
// removeClippedSubviews will clip them

// ❌ Known issue 3: can cause blank flashes on fast scroll
// The item re-attaches as you scroll to it — may show blank briefly

// ✅ Safe to enable for:
// - Simple list items (text + image, no overflow content)
// - Very long lists (1000+ items)
// - Lists with fixed, predictable item height

// Better alternative for extreme cases: windowSize
<FlatList
  windowSize={5}        // render 5 viewports worth (2 above, 2 below current)
  initialNumToRender={10} // first render batch
  maxToRenderPerBatch={5} // items rendered per batch during scroll
/>
```

---

### Q237. How do you profile a React Native app to find performance issues?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Profiling

**Answer:**
```js
// Tool 1: React DevTools Profiler (via Flipper or browser)
// Steps:
// 1. Open Flipper → React DevTools → Profiler tab
// 2. Click Record
// 3. Interact with the slow part of your app
// 4. Stop recording
// 5. Flamegraph shows: which components rendered, how long each took
// Look for: unexpected re-renders (components that lit up when they shouldn't)

// Tool 2: Native Systrace (Android)
// adb shell am start -n com.android.traceur/.MainActivity
// Captures JS thread, UI thread, GPU frames

// Tool 3: Xcode Instruments (iOS)
// Product → Profile → Time Profiler
// Shows CPU usage per function on each thread

// Tool 4: In-app FPS monitor
// Dev menu → Show Perf Monitor
// Watch JS FPS and UI FPS simultaneously

// Tool 5: Hermes sampling profiler (production-realistic)
// In __DEV__ = false mode, Hermes records JS CPU samples
// View in Chrome DevTools

// Code-level profiling
import { Profiler } from 'react';

<Profiler
  id="EmployeeList"
  onRender={(id, phase, actualDuration, baseDuration) => {
    if (actualDuration > 16) {
      console.warn(`Slow render in ${id}: ${actualDuration.toFixed(1)}ms`);
    }
  }}
>
  <EmployeeList employees={employees} />
</Profiler>
// actualDuration: time for this render
// baseDuration: estimated time without memoisation
// If baseDuration >> actualDuration: memo is working well

// Performance marks (custom timestamps)
console.time('loadDashboard');
await loadDashboardData();
console.timeEnd('loadDashboard'); // logs: loadDashboard: 342.5ms
```

---

### Q238. What are memory leaks in React Native and how do you find them?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Memory

**Answer:**
A memory leak occurs when objects are retained in memory after they're no longer needed — growing heap until the app crashes or becomes sluggish.

**Common sources in React Native:**

```js
// 1. Event listeners not cleaned up
useEffect(() => {
  const sub = AppState.addEventListener('change', handleAppState);
  // ❌ Missing cleanup:
  // return () => sub.remove();
}, []);

// 2. Async operations that setState after unmount
useEffect(() => {
  let isMounted = true;
  fetchData().then(data => {
    if (isMounted) setData(data); // ✅ guard
  });
  return () => { isMounted = false; }; // cleanup
}, []);

// 3. Interval/timeout not cleared
useEffect(() => {
  const interval = setInterval(() => {
    fetchNewMessages();
  }, 5000);
  return () => clearInterval(interval); // ✅ always clear
}, []);

// 4. Large closures captured by event handlers
const [bigData, setBigData] = useState(new Array(10000).fill('data'));

const handlePress = useCallback(() => {
  // bigData is captured in closure — keeps 10k items in memory
  // even if component re-renders with new data
  console.log(bigData.length);
}, [bigData]); // correct dependency, but be aware of closure size

// 5. Image/video not released (rare in RN but possible with custom native modules)

// Detecting leaks:
// Flipper → Memory tab → take heap snapshot before and after navigating
// If heap grows and doesn't shrink → leak
// Android: Android Studio → Memory Profiler
// iOS: Instruments → Leaks / Allocations
```

---

### Q239. How do you fix the most common memory leak patterns?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Memory

**Answer:**
```js
// Pattern 1: Subscription cleanup
useEffect(() => {
  const keyboardSub = Keyboard.addListener('keyboardDidShow', handler);
  const appStateSub = AppState.addEventListener('change', handler2);
  const dimensionSub = Dimensions.addEventListener('change', handler3);
  const netInfoSub = NetInfo.addEventListener(handler4);

  return () => {
    keyboardSub.remove();
    appStateSub.remove();
    dimensionSub.remove();
    netInfoSub();  // NetInfo returns unsubscribe function directly
  };
}, []);

// Pattern 2: Cancellable async with AbortController
useEffect(() => {
  const controller = new AbortController();

  fetch('/api/employees', { signal: controller.signal })
    .then(res => res.json())
    .then(data => setEmployees(data))
    .catch(err => {
      if (err.name !== 'AbortError') console.error(err);
    });

  return () => controller.abort(); // cancel fetch on unmount
}, []);

// Pattern 3: Axios cancel token
useEffect(() => {
  const source = axios.CancelToken.source();

  axios.get('/api/data', { cancelToken: source.token })
    .then(res => setData(res.data))
    .catch(err => { if (!axios.isCancel(err)) console.error(err); });

  return () => source.cancel('Component unmounted');
}, []);

// Pattern 4: useRef isMounted guard (older pattern, still valid)
const isMountedRef = useRef(true);
useEffect(() => {
  return () => { isMountedRef.current = false; };
}, []);

const safeFetch = async () => {
  const data = await fetchData();
  if (isMountedRef.current) setData(data);
};

// Pattern 5: Reanimated shared values — no cleanup needed
// Shared values are automatically garbage collected when no longer referenced
```

---

### Q240. What is `InteractionManager` and when should you use it?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Performance

**Answer:**
`InteractionManager` lets you defer heavy work until all JS animations and interactions have completed — keeps the UI smooth during transitions and gesture feedback.

```js
import { InteractionManager } from 'react-native';

// Run after all animations complete
useEffect(() => {
  const task = InteractionManager.runAfterInteractions(() => {
    // Heavy work here — runs after navigation animation settles
    loadHeavyCharts();
    processLargeDataset();
    initializeWebSocket();
  });

  return () => task.cancel(); // cancel if component unmounts before task runs
}, []);

// Navigation + data loading pattern
// ❌ Bad — starts heavy fetch during screen transition animation
const EmployeeScreen = () => {
  useEffect(() => {
    fetchAllEmployees(); // runs during nav animation → janky transition
  }, []);
};

// ✅ Good — waits for animation to finish
const EmployeeScreen = () => {
  useEffect(() => {
    const task = InteractionManager.runAfterInteractions(() => {
      fetchAllEmployees();
    });
    return () => task.cancel();
  }, []);
};

// Show loading while waiting
const [ready, setReady] = useState(false);
useEffect(() => {
  const task = InteractionManager.runAfterInteractions(() => setReady(true));
  return () => task.cancel();
}, []);

if (!ready) return <SkeletonScreen />;
return <FullContent />;

// Create your own interaction handle
const handle = InteractionManager.createInteractionHandle();
// ... run animation ...
InteractionManager.clearInteractionHandle(handle); // signals animation done
```

---

### Q241. How do you optimise a FlatList rendering 1,000+ items?

**Difficulty:** 🔴 Hard | **Frequency:** Very High | **Category:** FlatList Performance

**Answer:**
```js
const ITEM_HEIGHT = 72;
const SEPARATOR_HEIGHT = StyleSheet.hairlineWidth;
const ITEM_TOTAL = ITEM_HEIGHT + SEPARATOR_HEIGHT;

const OptimisedFlatList = ({ data }) => {
  // 1. Stable keyExtractor — avoid index as key
  const keyExtractor = useCallback((item) => item.id.toString(), []);

  // 2. Memoised renderItem — prevents recreation on parent re-render
  const renderItem = useCallback(({ item }) => (
    <EmployeeRow employee={item} />
  ), []);

  // 3. getItemLayout — skip dynamic height measurement
  const getItemLayout = useCallback((_, index) => ({
    length: ITEM_TOTAL,
    offset: ITEM_TOTAL * index,
    index,
  }), []);

  // 4. ItemSeparatorComponent — more efficient than margin in items
  const ItemSeparator = useCallback(() => (
    <View style={{ height: SEPARATOR_HEIGHT, backgroundColor: '#E0E0E0' }} />
  ), []);

  return (
    <FlatList
      data={data}
      keyExtractor={keyExtractor}
      renderItem={renderItem}
      getItemLayout={getItemLayout}
      ItemSeparatorComponent={ItemSeparator}

      // 5. Virtualisation tuning
      initialNumToRender={12}         // first visible screen worth
      maxToRenderPerBatch={10}        // items per scroll batch
      updateCellsBatchingPeriod={50}  // ms between batches
      windowSize={7}                  // 3 screens above + 3 below current
      removeClippedSubviews={true}    // detach off-screen native views (Android safe)

      // 6. Content container style instead of wrapper View
      contentContainerStyle={{ paddingBottom: 16 }}

      // 7. Avoid inline functions / objects in props
      // ❌ style={{ flex: 1 }} on FlatList itself creates new obj each render
      style={styles.list}  // ✅ from StyleSheet

      // 8. Disable scroll indicator if not needed
      showsVerticalScrollIndicator={false}
    />
  );
};

// 9. Keep list item components lean
const EmployeeRow = React.memo(({ employee }) => (
  <View style={styles.row}>
    <FastImage source={{ uri: employee.avatar }} style={styles.avatar} />
    <View style={styles.info}>
      <Text style={styles.name}>{employee.name}</Text>
      <Text style={styles.role}>{employee.role}</Text>
    </View>
  </View>
));
```

---

### Q242. What is `windowSize` on FlatList and how do you choose the right value?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** FlatList Performance

**Answer:**
`windowSize` determines how many **viewport-heights** worth of items FlatList keeps rendered. The default is 21 (10 screens above + current + 10 screens below).

```js
// windowSize={21} (default) — keeps 21 screens worth rendered
// windowSize={5}  — keeps 5 screens (2 above + current + 2 below)
// windowSize={1}  — only current viewport (most memory-efficient, most blank flash risk)

<FlatList
  windowSize={7}  // recommended sweet spot for most apps
  data={data}
  renderItem={renderItem}
/>

// Tradeoffs:
// Higher windowSize (21):
//   ✅ No blank flash on scroll
//   ❌ More items in memory
//   ❌ More initial render time
//
// Lower windowSize (3-5):
//   ✅ Less memory usage
//   ✅ Faster initial render
//   ❌ Blank flashes on very fast scroll

// For your ERP employee list (300-500 employees):
// windowSize={7} — 3 screens buffer each direction — good balance

// For photo gallery (heavy images):
// windowSize={3} — memory is more important

// For chat messages (small, fast scroll):
// windowSize={10} — smooth scroll experience more important

// windowSize works with maxToRenderPerBatch and updateCellsBatchingPeriod:
<FlatList
  windowSize={7}
  maxToRenderPerBatch={8}         // items rendered per "tick" during scroll
  updateCellsBatchingPeriod={50}  // 50ms between render batches
  initialNumToRender={10}         // first synchronous render batch
/>
```

---

### Q243. How do you measure and reduce the JavaScript bundle size?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Bundle Optimisation

**Answer:**
```bash
# Step 1: Generate bundle and sourcemap
npx react-native bundle \
  --platform ios \
  --dev false \
  --entry-file index.js \
  --bundle-output ./bundle/main.jsbundle \
  --sourcemap-output ./bundle/main.jsbundle.map

# Step 2: Analyse with source-map-explorer
npm install -g source-map-explorer
source-map-explorer ./bundle/main.jsbundle ./bundle/main.jsbundle.map

# Or use bundle-buddy for a different visualization
npx bundle-buddy ./bundle/main.jsbundle.map
```

```js
// Common bundle size culprits and fixes:

// 1. moment.js — huge locale files (67KB gzipped)
// ❌ import moment from 'moment';
// ✅ import dayjs from 'dayjs'; // 2KB gzipped
// ✅ Or: import { format } from 'date-fns'; // tree-shakeable

// 2. lodash — importing the whole library
// ❌ import _ from 'lodash';
// ✅ import debounce from 'lodash/debounce'; // only import what you need
// ✅ import { debounce } from 'lodash-es'; // tree-shakeable version

// 3. Icons — loading all icons
// ❌ import Icon from 'react-native-vector-icons/MaterialIcons'; // all icons
// ✅ Only the icon sets you use (configure in gradle/xcode)

// 4. Unused dependencies — audit package.json regularly
// depcheck — finds unused packages
npx depcheck

// 5. Large assets bundled in JS
// ❌ require('./big-image.png') in JS — bundled into JS bundle
// ✅ Store in Assets folder, reference via path

// 6. babel-plugin-transform-remove-console
// Removes all console.* calls from production bundle
// metro.config.js:
module.exports = {
  transformer: {
    minifierConfig: {
      compress: { drop_console: true },
    },
  },
};
```

---

### Q244. What is Hermes and how does it improve performance?

**Difficulty:** 🟡 Medium | **Frequency:** Very High | **Category:** JS Engine Performance

**Answer:**
Hermes is a JavaScript engine built by Meta specifically for React Native. It replaces JavaScriptCore (JSC) as the default engine from RN 0.70+.

**Key improvements over JSC:**

| | JavaScriptCore | Hermes |
|--|---------------|--------|
| Bundle type | JS source → JIT at runtime | Pre-compiled bytecode |
| TTI (Time to Interactive) | Slower (compiles on device) | Faster (already compiled) |
| Memory usage | Higher | Lower (smaller heap) |
| APK/IPA size | Larger | Smaller (bytecode is smaller) |
| Debugging | Chrome DevTools | Hermes debugger |

```js
// Enable Hermes (android/app/build.gradle)
project.ext.react = [
  enableHermes: true,
]

// Enable Hermes (iOS — Podfile)
use_react_native!(
  :hermes_enabled => true
)

// Check if Hermes is running
const isHermes = () => !!global.HermesInternal;
console.log('Hermes:', isHermes()); // true in Hermes, false in JSC

// Hermes-specific optimisations:
// 1. Bytecode pre-compilation — JS compiled to bytecode at build time
//    Metro bundler outputs .hbc file instead of .jsbundle
// 2. Lazy loading — functions compiled lazily, not all upfront
// 3. Static analysis — better dead code elimination at compile time
// 4. GC improvements — generational garbage collector, lower pause times

// Profiling with Hermes:
// 1. Shake device → Open Debugger → Enable Sampling Profiler
// 2. Interact with app
// 3. Stop profiler → download .cpuprofile
// 4. Open in Chrome DevTools → Performance tab → Load profile
```

---

### Q245. How do you implement code splitting and lazy loading in React Native?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Bundle Optimisation

**Answer:**
React Native's Metro bundler doesn't support dynamic `import()` for code splitting out of the box like webpack. You need RAM bundles or re-pack.

```js
// Method 1: React.lazy + Suspense (React 18, requires Hermes/JSC support)
import React, { Suspense, lazy } from 'react';

// Lazily loaded screen
const HeavyAnalyticsScreen = lazy(() => import('./HeavyAnalyticsScreen'));

const App = () => (
  <Suspense fallback={<LoadingScreen />}>
    <HeavyAnalyticsScreen />
  </Suspense>
);

// Method 2: RAM Bundles (Metro feature)
// Each module is loaded only when first required
// Configure in metro.config.js:
module.exports = {
  serializer: {
    processModuleFilter: module => true,
  },
};
// Add to index.js:
// @format
// RAM bundle — each require() call loads the module on demand

// Method 3: Re.Pack (webpack for RN — enables true code splitting)
// https://re-pack.dev
// Creates separate chunks loaded dynamically

// Method 4: Manual lazy loading (simplest, always works)
const LazyScreen = ({ navigation }) => {
  const [Component, setComponent] = useState(null);

  useEffect(() => {
    // Load component only when screen first mounts
    import('./HeavyComponent').then(module => {
      setComponent(() => module.default);
    });
  }, []);

  if (!Component) return <LoadingSpinner />;
  return <Component navigation={navigation} />;
};

// Method 5: Defer non-critical modules
// Put heavy requires inside functions instead of at module top level
// ❌ import HeavyLib from 'heavy-lib'; // loads at app start
// ✅ const getHeavyLib = () => require('heavy-lib'); // loads on first call
```

---

### Q246. What is `console.log` doing to your production performance?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Performance

**Answer:**
`console.log` in production causes measurable performance degradation — each call:
1. Serialises its arguments to strings (can be expensive for large objects)
2. Crosses the bridge to native (in old architecture) to output to system logs
3. Accumulates in memory if the remote debugger is connected

```js
// Remove console.log from production builds

// Method 1: babel-plugin-transform-remove-console (recommended)
// .babelrc or babel.config.js:
module.exports = {
  plugins: [
    ...(__DEV__ ? [] : [['transform-remove-console', { exclude: ['error', 'warn'] }]]),
  ],
};

// Method 2: Override console in production
if (!__DEV__) {
  const emptyFn = () => {};
  console.log = emptyFn;
  console.debug = emptyFn;
  console.info = emptyFn;
  // Keep console.error and console.warn for Sentry/crash reporting
}

// Method 3: Custom logger utility
const logger = {
  log: (...args) => { if (__DEV__) console.log(...args); },
  warn: (...args) => console.warn(...args),           // always warn
  error: (...args) => console.error(...args),         // always error
  perf: (...args) => { if (__DEV__) console.log('[PERF]', ...args); },
};

export default logger;

// Real performance impact:
// console.log(largeArray) — serialises potentially MBs of data
// In a FlatList with 1000 items, a console.log inside renderItem
// = 1000 serialisations per render pass
```

---

### Q247. How do you optimise image loading and caching in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** Very High | **Category:** Images + Performance

**Answer:**
```js
// Problem: React Native's built-in Image component
// - No disk caching by default on Android
// - Each mount may re-download the image
// - No progressive loading

// Solution: react-native-fast-image (Glorified SDWebImage/Glide)
import FastImage from 'react-native-fast-image';

// Basic usage with priority and cache strategy
<FastImage
  source={{
    uri: 'https://example.com/employee-avatar.jpg',
    priority: FastImage.priority.normal,   // low, normal, high
    cache: FastImage.cacheControl.immutable, // immutable, web, cacheOnly
  }}
  style={{ width: 60, height: 60, borderRadius: 30 }}
  resizeMode={FastImage.resizeMode.cover}
  onLoadStart={() => setLoading(true)}
  onLoadEnd={() => setLoading(false)}
/>

// Preload images before they're needed
FastImage.preload([
  { uri: 'https://example.com/image1.jpg', priority: FastImage.priority.high },
  { uri: 'https://example.com/image2.jpg' },
]);

// Clear cache
FastImage.clearMemoryCache();
FastImage.clearDiskCache();

// Image optimisation tips:
// 1. Serve correctly-sized images (don't serve 2000px for 60px avatar)
// 2. Use CDN with image transformation (Cloudinary, imgix)
//    https://res.cloudinary.com/demo/image/upload/w_120,h_120,c_fill/avatar.jpg

// 3. Use WebP format (30% smaller than JPEG, lossless)
<FastImage source={{ uri: imageUrl.replace('.jpg', '.webp') }} />

// 4. Placeholder while loading
const [loaded, setLoaded] = useState(false);
<View>
  {!loaded && <ShimmerPlaceholder style={styles.image} />}
  <FastImage
    source={{ uri }}
    style={[styles.image, !loaded && styles.hidden]}
    onLoad={() => setLoaded(true)}
  />
</View>

// 5. Progressive JPEG — show blurry version first
<Image source={{ uri }} blurRadius={loaded ? 0 : 10} />
```

---

### Q248. What is `shouldComponentUpdate` equivalent in functional React Native components?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Performance

**Answer:**
```js
// Class component equivalent
class EmployeeRow extends PureComponent {
  // PureComponent = shouldComponentUpdate with shallow prop comparison
  render() {
    return <View><Text>{this.props.employee.name}</Text></View>;
  }
}

// Functional equivalent — React.memo
const EmployeeRow = React.memo(({ employee }) => {
  return <View><Text>{employee.name}</Text></View>;
});
// React.memo does shallow comparison by default

// Deep custom comparison
const EmployeeRow = React.memo(({ employee, onPress }) => {
  return <Pressable onPress={() => onPress(employee.id)}><Text>{employee.name}</Text></Pressable>;
}, (prevProps, nextProps) => {
  // Return true = same = skip re-render
  // Return false = different = re-render
  return (
    prevProps.employee.id === nextProps.employee.id &&
    prevProps.employee.name === nextProps.employee.name &&
    prevProps.employee.avatarUrl === nextProps.employee.avatarUrl &&
    prevProps.onPress === nextProps.onPress // ← requires useCallback in parent
  );
});

// useMemo for expensive derived data
const processedData = useMemo(() => {
  return rawData
    .filter(item => item.active)
    .map(item => ({ ...item, displayName: `${item.firstName} ${item.lastName}` }))
    .sort((a, b) => a.displayName.localeCompare(b.displayName));
}, [rawData]); // only reprocess when rawData changes

// Common mistake: wrapping everything in useMemo/memo
// Memoisation has overhead — only use when:
// 1. The computation is expensive (>1ms)
// 2. The component renders frequently with same props
// 3. The component is in a list (FlatList)
```

---

### Q249. How do you reduce the app startup time (Time to Interactive)?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Startup Performance

**Answer:**
```js
// Time to Interactive (TTI) = time from app launch to first interactive screen

// 1. Enable Hermes (biggest single win)
// Bytecode pre-compilation → no JIT startup cost
// 40-50% TTI improvement typical

// 2. Inline Requires (Metro)
// metro.config.js
module.exports = {
  transformer: {
    getTransformOptions: async () => ({
      transform: {
        inlineRequires: true, // defer requires until first use
      },
    }),
  },
};

// 3. Reduce bundle size (fewer modules to parse)
// See Q243 for bundle analysis

// 4. Splash screen — hide transition between native and RN
import SplashScreen from 'react-native-splash-screen';

// App.tsx
useEffect(() => {
  // Wait for initial data to load, then hide splash
  const init = async () => {
    await loadCriticalData(); // auth state, user prefs, theme
    SplashScreen.hide();
  };
  init();
}, []);

// 5. Defer non-critical initialisation
// ❌ Load everything before first render
useEffect(() => {
  initAnalytics();     // can wait
  initPushNotifications(); // can wait
  loadUserData();      // CRITICAL — load now
  setupCrashReporting(); // can wait
}, []);

// ✅ Defer non-critical
useEffect(() => {
  loadUserData().then(() => {
    // Defer these until after first render
    InteractionManager.runAfterInteractions(() => {
      initAnalytics();
      initPushNotifications();
      setupCrashReporting();
    });
  });
}, []);

// 6. Lazy-load heavy screens
// Render AuthStack first, load AppStack only when authenticated
// Don't pre-load screens the user may never visit

// 7. Pre-warm React Native bridge (cold start only)
// iOS: [ReactNativeFactory preWarm] in AppDelegate
// Android: preload ReactInstanceManager in Application class

// Measure TTI:
// iOS: Instruments → App Launch template
// Android: adb shell am start -W com.yourapp/.MainActivity
// Flipper → Performance plugin
```

---

### Q250. What is the `Hermes` bytecode compilation pipeline?

**Difficulty:** 🔴 Hard | **Frequency:** Low | **Category:** JS Engine

**Answer:**
```
Source Code (.js)
      ↓
Metro Bundler (dev: transpile only, prod: bundle + minify)
      ↓
Hermes Compiler (hermes -emit-binary) — at BUILD TIME
      ↓
Hermes Bytecode (.hbc) — pre-compiled, no parsing at runtime
      ↓
Device: Hermes VM loads .hbc → execute directly
```

```bash
# Manual compilation (for understanding)
# hermes -emit-binary -out output.hbc input.js

# In production builds:
# Android: gradle builds .hbc automatically when Hermes enabled
# iOS: react-native-xcode.sh handles it via Xcode build phase
```

```js
// Development mode:
// Hermes still uses JS source (for better debugging / sourcemaps)
// No bytecode compilation in dev

// Production mode:
// Metro bundles → Hermes compiler produces .hbc
// .hbc is shipped in the APK/IPA instead of .js

// Benefits of .hbc:
// - Parsing: zero cost (already parsed at build time)
// - Startup: only execute, don't parse
// - Size: .hbc is typically smaller than minified .js
//   (binary format + dead code elimination at compile time)

// Checking Hermes is active in a production build:
// adb shell run-as com.yourapp cat files/ReactNativeResources/index.android.bundle | file -
// Should show: data (binary) not: ASCII text
```

---

### Q251. How do you measure and improve the Time to First Frame?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Startup Performance

**Answer:**
Time to First Frame (TTFF) = time from app launch until the first React Native frame renders.

```js
// Measure TTFF on Android
// adb logcat | grep "ReactNative"
// Look for: "ReactNativeJS: Running application" timestamp
// Subtract from: ActivityManager: Displayed com.app/.MainActivity

// Measure on iOS — Instruments → App Launch

// Improve TTFF

// 1. Minimise root component complexity
// ❌ Heavy root component
const App = () => {
  const [config, setConfig] = useState(null);
  useEffect(() => { loadLargeConfig().then(setConfig); }, []);

  if (!config) return <SplashScreen />; // renders native splash
  return <AppContent config={config} />;
};

// ✅ Show native splash (faster than React Native view) while loading
// 2. Move heavy initialisation out of synchronous render

// 3. Reduce synchronous work in root render
// Every ms in synchronous render = ms delay to first frame
// Move data loading to async, render skeleton immediately

// 4. Reduce number of JS modules loaded synchronously
// Check metro.config.js inlineRequires (defers require to first use)

// 5. Profile with React Profiler
<Profiler id="AppRoot" onRender={(id, phase, duration) => {
  if (phase === 'mount') {
    console.log(`First render: ${duration.toFixed(1)}ms`);
  }
}}>
  <App />
</Profiler>
```

---

### Q252. What is `VirtualizedList` and how is it different from FlatList?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Lists + Performance

**Answer:**
`VirtualizedList` is the **base component** that `FlatList` and `SectionList` are built on. It implements virtualisation — only rendering items in and near the viewport.

```js
// FlatList = VirtualizedList with a simple array data prop
// SectionList = VirtualizedList with sections support
// VirtualizedList = lower-level, most flexible

import { VirtualizedList } from 'react-native';

// VirtualizedList requires: getItem, getItemCount, keyExtractor
<VirtualizedList
  data={myCustomData}  // can be any data structure (not just array)
  getItemCount={(data) => data.size}          // how many items total
  getItem={(data, index) => data.get(index)}  // get item at index
  keyExtractor={(item) => item.id}
  renderItem={({ item }) => <Row item={item} />}
  getItemLayout={(data, index) => ({ length: 72, offset: 72 * index, index })}
/>

// Use VirtualizedList directly when:
// 1. Your data source is not a plain array (Map, custom object, etc.)
// 2. You need extreme control over the virtualisation behaviour
// 3. Building a custom list component

// FlatList is fine for 99% of cases
// SectionList when you need grouped sections with sticky headers

// All three share the same performance props:
// initialNumToRender, maxToRenderPerBatch, windowSize, removeClippedSubviews
// getItemLayout, keyExtractor (same interface)
```

---

### Q253. What are `PureComponent` and `React.memo` and their caveats?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Performance

**Answer:**
Both prevent re-renders when props haven't changed, but have important caveats:

```js
// PureComponent — class component, shallow prop comparison
class MyRow extends React.PureComponent {
  render() { return <View><Text>{this.props.title}</Text></View>; }
}

// React.memo — functional component equivalent
const MyRow = React.memo(({ title }) => <View><Text>{title}</Text></View>);

// CAVEAT 1: shallow comparison — new objects/arrays always trigger re-render
const Parent = () => {
  // ❌ New array on every render — breaks memo!
  return <MyList data={[1, 2, 3]} />;
};
// ✅ Stable reference
const DATA = [1, 2, 3];
const Parent = () => <MyList data={DATA} />;

// Or with useMemo:
const data = useMemo(() => [1, 2, 3], []);

// CAVEAT 2: inline functions break memo
const Parent = () => {
  // ❌ New function reference on every render
  return <Button onPress={() => doSomething()} />;
};
// ✅ useCallback for stable reference
const handlePress = useCallback(() => doSomething(), []);

// CAVEAT 3: Context changes propagate through memo
// If a memoised component consumes Context, it re-renders on context change
// regardless of memo

// CAVEAT 4: Forwardref components need memo wrapping explicitly
const ForwardedInput = React.memo(React.forwardRef((props, ref) => (
  <TextInput ref={ref} {...props} />
)));

// When memo HURTS (anti-pattern):
// - Very simple, cheap-to-render components
// - Components that always receive new props anyway
// - Costs: extra comparison function call, closure allocation
```

---

### Q254. How do you find and fix re-render waterfalls?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Performance

**Answer:**
A re-render waterfall happens when a state update in a parent causes a chain of re-renders through children, even when children's own data hasn't changed.

```js
// Diagnosing: react-addons-perf (old) or React DevTools Profiler (new)
// Also: why-did-you-render library

// Setup why-did-you-render
// wdyr.js
import React from 'react';
if (__DEV__) {
  const whyDidYouRender = require('@welldone-software/why-did-you-render');
  whyDidYouRender(React, {
    trackAllPureComponents: true,
    logOnDifferentValues: true, // logs what changed
  });
}

// In component to track:
MyComponent.whyDidYouRender = true;

// Common waterfall patterns and fixes:

// 1. Context causing all consumers to re-render
// ❌ One context for everything
const AppContext = createContext({ user, theme, notifications, cart });

// ✅ Split contexts
const UserContext = createContext(user);
const ThemeContext = createContext(theme);
// Each component only consumes what it needs

// 2. Redux mapStateToProps selecting too much
// ❌ Selecting whole state slice
const mapState = (state) => ({ everything: state.employees });

// ✅ Select minimal data with reselect
const selectSortedEmployees = createSelector(
  state => state.employees.list,
  state => state.employees.sortBy,
  (list, sortBy) => [...list].sort(...)
);

// 3. Parent re-renders because its own state changes
// but children don't depend on that state
const Parent = () => {
  const [inputValue, setInputValue] = useState(''); // changes on every keystroke
  const expensiveData = computeExpensiveData(); // recalculates on every keystroke!

  return (
    <View>
      <TextInput value={inputValue} onChangeText={setInputValue} />
      <ExpensiveChild data={expensiveData} /> {/* re-renders on every keystroke! */}
    </View>
  );
};

// ✅ Separate state and memoize
const Parent = () => {
  const [inputValue, setInputValue] = useState('');
  const expensiveData = useMemo(() => computeExpensiveData(), []); // stable
  const handleChange = useCallback((text) => setInputValue(text), []);

  return (
    <View>
      <TextInput value={inputValue} onChangeText={handleChange} />
      <MemoizedExpensiveChild data={expensiveData} /> {/* won't re-render */}
    </View>
  );
};
```

---

### Q255. How do you optimise Redux performance in a large React Native app?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** State + Performance

**Answer:**
```js
// 1. Memoised selectors with reselect
import { createSelector } from '@reduxjs/toolkit';

const selectEmployees = state => state.employees.list;
const selectDept = state => state.filters.department;

// Memoised — only recomputes when employees or dept changes
const selectFilteredEmployees = createSelector(
  [selectEmployees, selectDept],
  (employees, dept) => dept === 'all'
    ? employees
    : employees.filter(e => e.department === dept)
);

// 2. useSelector with equality function
import { shallowEqual, useSelector } from 'react-redux';

// ❌ New object on every select — triggers re-render every dispatch
const { name, role } = useSelector(state => ({
  name: state.user.name,
  role: state.user.role,
}));

// ✅ shallowEqual — only re-render if name or role actually changes
const { name, role } = useSelector(
  state => ({ name: state.user.name, role: state.user.role }),
  shallowEqual
);

// 3. Split large slices — components only subscribe to relevant parts
// One slice per domain: userSlice, employeesSlice, attendanceSlice

// 4. Normalise state with @reduxjs/toolkit createEntityAdapter
import { createEntityAdapter } from '@reduxjs/toolkit';

const employeesAdapter = createEntityAdapter();
// Stores: { ids: ['1','2'], entities: {'1': emp1, '2': emp2} }
// O(1) lookups, no array scanning

// 5. Batch dispatches — RTK handles this automatically in RN 18
// For older RN: unstable_batchedUpdates
import { unstable_batchedUpdates } from 'react-native';

unstable_batchedUpdates(() => {
  dispatch(setEmployees(data));
  dispatch(setLoading(false));
  dispatch(setLastFetched(Date.now()));
  // All three batched into one React render pass
});

// 6. RTK Query — automatic caching, deduplication, background refresh
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

const api = createApi({
  baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
  endpoints: builder => ({
    getEmployees: builder.query({ query: () => '/employees' }),
  }),
});
// Auto-caches, doesn't re-fetch if data is fresh, deduplicates concurrent requests
```

---

### Q256. What is excessive re-rendering and how do you audit for it?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Performance

**Answer:**
```js
// Quick audit in development — add render counter to suspect components
const renderCount = useRef(0);
renderCount.current++;
console.log(`MyComponent rendered ${renderCount.current} times`);

// More detailed — log what changed
import { useEffect, useRef } from 'react';
const usePrevious = (value) => {
  const ref = useRef();
  useEffect(() => { ref.current = value; });
  return ref.current;
};

const MyComponent = ({ propA, propB, propC }) => {
  const prevA = usePrevious(propA);
  const prevB = usePrevious(propB);
  const prevC = usePrevious(propC);

  if (__DEV__) {
    if (prevA !== propA) console.log('propA changed:', prevA, '->', propA);
    if (prevB !== propB) console.log('propB changed:', prevB, '->', propB);
    if (prevC !== propC) console.log('propC changed:', prevC, '->', propC);
  }
};

// React DevTools Profiler — visual audit
// 1. Open React DevTools → Profiler
// 2. Enable "Highlight updates when components render" in settings
//    (painted rectangles flash on components when they re-render)
// 3. Interact with the app
// 4. Components that flash too frequently are candidates for memoisation

// Why-did-you-render — logs re-renders with reason to console
import './wdyr'; // setup (see Q254)
MyList.whyDidYouRender = true;
// Console: MyList re-rendered. prevProps.data !== nextProps.data
// {a: 1, b: 2} vs {a: 1, b: 2} (different reference, same value)
// → Need useMemo or stable reference
```

---

### Q257. How do you profile startup performance with Flipper?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Profiling Tools

**Answer:**
```js
// Flipper is the official RN debugging platform (desktop app)
// Install: https://fbflipper.com

// Startup performance plugins in Flipper:

// 1. Hermes Debugger — JS profiling (replaces Chrome DevTools for Hermes)
// Flipper → Hermes Debugger → Profiler → Record
// Shows: function call tree, CPU time per function

// 2. React DevTools → Profiler
// Shows: component render tree, which components rendered, render duration

// 3. Performance Plugin (community)
// Shows: JS/UI FPS, memory usage over time, network timing

// 4. Network Inspector
// Shows: all HTTP/HTTPS requests, request/response timing
// Identify slow API calls on startup

// Manual startup timing
// index.js (very first file)
global.__appStartTime = Date.now();

// Your first screen component
useEffect(() => {
  const startupTime = Date.now() - global.__appStartTime;
  console.log(`App startup: ${startupTime}ms`);
  // Track in analytics
  analytics.track('app_startup', { duration: startupTime });
}, []);

// Mark specific milestones
performance.mark('authCheckStart');
await checkAuthState();
performance.mark('authCheckEnd');
performance.measure('authCheck', 'authCheckStart', 'authCheckEnd');
```

---

### Q258. How does React Native handle overdraw and how do you reduce it?

**Difficulty:** 🔴 Hard | **Frequency:** Low | **Category:** Rendering Performance

**Answer:**
Overdraw occurs when a pixel is drawn multiple times in a single frame — every transparent or overlapping layer multiplies GPU work.

```
Level 0 (no overdraw):   white
Level 1 (1x overdraw):   blue
Level 2 (2x overdraw):   green
Level 3 (3x overdraw):   pink
Level 4+ (4x overdraw):  red  ← problem
```

```js
// Detect on Android:
// Developer Options → Debug GPU Overdraw → Show overdraw areas
// Red areas = 4x+ overdraw → optimize

// Common sources of overdraw:

// 1. Stacked Views with backgrounds
<View style={{ backgroundColor: 'white' }}>  {/* layer 1 */}
  <View style={{ backgroundColor: 'white' }}> {/* layer 2 — redundant! */}
    <View style={{ backgroundColor: 'white' }}> {/* layer 3 — redundant! */}
      <Text>Content</Text>
    </View>
  </View>
</View>

// ✅ Fix: remove redundant background colors
<View style={{ backgroundColor: 'white' }}>
  <View> {/* no background — transparent */}
    <Text>Content</Text>
  </View>
</View>

// 2. Modal with semi-transparent backdrop + content background
// Can't avoid all overdraw here — minimize layers

// 3. Images with background color AND image overlapping
<View style={{ backgroundColor: '#f0f0f0', borderRadius: 8 }}>
  {/* If image covers entire view, the backgroundColor never shows */}
  <Image source={...} style={{ width: '100%', height: '100%' }} />
</View>
// ✅ Fix: no background if fully covered by image

// 4. Shadow layers (iOS) — each shadow is a composited layer
// Minimize shadow usage on frequently-rendered list items

// 5. Opacity < 1 creates a composited layer
// Only use opacity: 0/1 (binary) or opacity with Reanimated for animations
```

---

### Q259. How do you handle large datasets without pagination in React Native?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Lists + Performance

**Answer:**
```js
// Scenario: offline ERP app, 5,000 employees, no server pagination

// Strategy 1: Client-side virtualised FlatList (already covered Q241)
// FlatList virtualises rendering — only renders ~15-20 items at a time
// 5,000 items in FlatList: only render 20, virtualise 4,980
// ✅ Works well up to ~50,000 items with proper getItemLayout

// Strategy 2: Chunked loading into state
const CHUNK_SIZE = 200;

const EfficientEmployeeList = () => {
  const [visibleData, setVisibleData] = useState([]);
  const allData = useRef([]);

  useEffect(() => {
    const loadChunks = async () => {
      const fullList = await getAllEmployees();
      allData.current = fullList;

      // Load first chunk immediately
      setVisibleData(fullList.slice(0, CHUNK_SIZE));

      // Load remaining chunks after interaction
      InteractionManager.runAfterInteractions(() => {
        // Load in chunks to avoid UI freeze
        for (let i = CHUNK_SIZE; i < fullList.length; i += CHUNK_SIZE) {
          setTimeout(() => {
            setVisibleData(prev => [...prev, ...fullList.slice(i, i + CHUNK_SIZE)]);
          }, (i / CHUNK_SIZE) * 100); // stagger by 100ms per chunk
        }
      });
    };
    loadChunks();
  }, []);

  return <FlatList data={visibleData} renderItem={renderItem} getItemLayout={getItemLayout} />;
};

// Strategy 3: Search-filtered view (most practical for ERP)
const [searchQuery, setSearchQuery] = useState('');
const [page, setPage] = useState(50); // start with 50, expand on scroll

const filteredData = useMemo(() => {
  const filtered = searchQuery
    ? allEmployees.filter(e => e.name.toLowerCase().includes(searchQuery.toLowerCase()))
    : allEmployees.slice(0, page);
  return filtered;
}, [allEmployees, searchQuery, page]);

const loadMore = useCallback(() => {
  setPage(prev => Math.min(prev + 50, allEmployees.length));
}, [allEmployees.length]);
```

---

### Q260. What is `useMemo` for style objects and when is it worth it?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Performance

**Answer:**
```js
// Inline style objects create new references on every render
// This breaks React.memo for child components that receive the style as a prop

// ❌ New object every render — breaks memo for children
const MyComponent = ({ isActive, size }) => (
  <View style={{ width: size, backgroundColor: isActive ? '#6200EE' : '#ccc' }}>
    <Text>Content</Text>
  </View>
);

// ✅ useMemo for dynamic styles passed to memoised children
const MyComponent = ({ isActive, size }) => {
  const containerStyle = useMemo(() => ({
    width: size,
    backgroundColor: isActive ? '#6200EE' : '#ccc',
  }), [isActive, size]);

  return (
    <View style={containerStyle}>
      <Text>Content</Text>
    </View>
  );
};

// ✅ Better: StyleSheet.create for static styles (computed once at load time)
const styles = StyleSheet.create({
  active: { backgroundColor: '#6200EE' },
  inactive: { backgroundColor: '#ccc' },
});

// Compose dynamic + static
<View style={[styles.container, { width: size }, isActive ? styles.active : styles.inactive]}>

// When useMemo for styles IS worth it:
// - Style is complex to compute (involves math, multiple conditions)
// - Style is passed as prop to a React.memo child
// - Style changes infrequently relative to component renders

// When it's NOT worth it:
// - Style is passed only to native elements (View, Text) — they don't use React.memo
// - Style computation is trivially cheap
// - Component is already memoised for other reasons
```

---

### Q261. How do you batch state updates in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Performance

**Answer:**
```js
// React 18+ (React Native 0.71+) — automatic batching everywhere
// Multiple state updates in async functions, timeouts, and event handlers
// are automatically batched into one re-render

// React 18 — automatic batching
const handleSubmit = async () => {
  await saveEmployee(data);
  setLoading(false);     // batched
  setEmployee(null);     // batched
  setSuccess(true);      // batched
  // Single re-render for all three — automatic in React 18
};

// React 17 and earlier — only event handlers were batched
// async/timeout updates caused multiple re-renders
// Fix: unstable_batchedUpdates
import { unstable_batchedUpdates } from 'react-native';

const handleOldReactSubmit = async () => {
  await saveEmployee(data);
  unstable_batchedUpdates(() => {
    setLoading(false);     // batched
    setEmployee(null);     // batched
    setSuccess(true);      // batched
    // Single re-render — React 17 compatible
  });
};

// Opt OUT of batching (rare need):
import { flushSync } from 'react-dom'; // React 18
const handleEdgeCase = () => {
  flushSync(() => { setA(1); }); // forces immediate re-render for A
  flushSync(() => { setB(2); }); // then immediate re-render for B
};

// Redux RTK — batches multiple dispatches automatically (React 18)
// In React 17: use unstable_batchedUpdates around dispatch calls
```

---

### Q262. How do you optimise heavy computation that runs on the JS thread?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** JS Thread Performance

**Answer:**
```js
// Option 1: Web Workers via react-native-workers or react-native-threads
import { DataWorker } from 'react-native-workers';

const worker = new DataWorker('./dataProcessingWorker.js');
worker.postMessage({ data: rawPayrollData });
worker.onmessage = (event) => {
  const { processedData } = event.data;
  setPayrollData(processedData);
};

// worker.js (runs on a separate JS thread)
self.onmessage = (event) => {
  const { data } = event.data;
  const result = data.map(processPayrollItem).reduce(aggregatePayroll, {});
  self.postMessage({ processedData: result });
};

// Option 2: Native module for CPU-intensive work
// Write the computation in Swift/Kotlin, call via TurboModule
// Best for: image processing, encryption, large data transformations

// Option 3: Defer and chunk on JS thread
const processInChunks = async (data, chunkSize = 100) => {
  const results = [];
  for (let i = 0; i < data.length; i += chunkSize) {
    const chunk = data.slice(i, i + chunkSize);
    const processed = chunk.map(heavyProcessItem);
    results.push(...processed);
    // Yield to event loop between chunks — allows UI to update
    await new Promise(resolve => setTimeout(resolve, 0));
  }
  return results;
};

// Option 4: Memoize expensive computations
const memoized = useMemo(() => expensiveComputation(input), [input]);

// Option 5: Use a library with native backing
// React Native Reanimated 2 — runs worklets on UI thread
// React Native Vision Camera — ML/CV on dedicated thread
// MMKV — synchronous storage on native thread
```

---

### Q263. What is `setNativeProps` and when should you use it?

**Difficulty:** 🔴 Hard | **Frequency:** Low | **Category:** Performance

**Answer:**
`setNativeProps` updates native component properties directly without going through the React reconciler — useful for highly interactive animations that can't use Reanimated.

```js
const inputRef = useRef(null);

// Direct native update — bypasses React render cycle
inputRef.current.setNativeProps({
  text: 'Hello',
  style: { color: '#6200EE' },
});

// Use case 1: Updating text input value during rapid typing
// without setState (avoids re-render on each keystroke)
const TextCounter = () => {
  const counterRef = useRef(null);
  let count = 0;

  return (
    <View>
      <TextInput onChangeText={() => {
        count++;
        counterRef.current?.setNativeProps({ text: `${count} chars` });
        // No setState → no re-render → no jank
      }} />
      <TextInput ref={counterRef} editable={false} value="0 chars" />
    </View>
  );
};

// Modern alternative: Reanimated's useAnimatedProps is better
// setNativeProps has these limitations:
// 1. Only updates specific properties, not all
// 2. Not tracked by React reconciler — state and view can diverge
// 3. Not type-safe
// 4. Deprecated in New Architecture (Fabric uses a different mechanism)

// Prefer: Reanimated useAnimatedProps for performance-critical prop updates
const animatedProps = useAnimatedProps(() => ({
  text: String(counter.value),
}));
```

---

### Q264. How do you reduce bridge traffic in React Native (old architecture)?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Architecture + Performance

**Answer:**
In the old architecture, every JS↔Native communication crosses the async bridge. High bridge traffic is a significant performance bottleneck.

```js
// Measure bridge traffic:
// Systrace → look for JS→UI bridge calls
// Chrome DevTools → check frequency of native module calls

// Strategy 1: Batch multiple native calls
// ❌ Three bridge crossings
AsyncStorage.setItem('key1', value1);
AsyncStorage.setItem('key2', value2);
AsyncStorage.setItem('key3', value3);

// ✅ One bridge crossing
AsyncStorage.multiSet([['key1', value1], ['key2', value2], ['key3', value3]]);

// Strategy 2: useNativeDriver for animations (zero bridge per frame)
Animated.timing(value, { toValue: 1, useNativeDriver: true }).start();

// Strategy 3: Send large data in one call
// ❌ Multiple small bridge calls
employees.forEach(emp => {
  NativeModule.addEmployee(emp);
});

// ✅ One call with array
NativeModule.addEmployees(employees);

// Strategy 4: Debounce/throttle native calls
const debouncedUpdateNative = useMemo(
  () => debounce((value) => NativeModule.updateValue(value), 100),
  []
);

// Strategy 5: Move computation to native (new arch: TurboModules do this synchronously)
// Instead of: JS fetches data → processes → sends to native
// Do: native fetches data → processes → sends result to JS once

// Strategy 6: Use JSI (New Architecture) — eliminates bridge entirely
// TurboModules: synchronous JS↔Native calls
// Fabric: React component tree rendered directly in C++
```

---

### Q265. What is the New Architecture and how does it improve performance?

**Difficulty:** 🔴 Hard | **Frequency:** Very High | **Category:** Architecture

**Answer:**
```
Old Architecture:
React → Serialise to JSON → Async Bridge → Deserialise → Native

New Architecture (3 main pillars):
1. JSI (JavaScript Interface) — C++ layer, JS calls native synchronously
2. TurboModules — native modules loaded lazily, called via JSI
3. Fabric — new rendering system, tree diffing in C++ (not JS)
```

```js
// Enabling New Architecture
// android/gradle.properties
newArchEnabled=true

// ios/Podfile
ENV['RCT_NEW_ARCH_ENABLED'] = '1'

// What changes for developers:

// 1. TurboModules — no more bridge, synchronous or async
// Old: NativeModules.SomeModule.someMethod(callback)
// New: SomeModule.someMethod() — direct JSI call

// 2. Fabric — concurrent rendering support
// ✅ Suspense, startTransition, useDeferredValue work properly in RN
// ✅ Priority-based rendering

// 3. Codegen — TypeScript specs generate native type-safe interfaces
// nativeSpec.ts → generates Swift/Kotlin bindings at build time

// 4. React Native Web compatibility improves
// Fabric shares rendering infrastructure with React DOM

// Migration considerations:
// - Some third-party libs not yet compatible with New Arch
// - Must enable or disable per package: fabric: true/false in pod
// - gradual migration possible (interop layer available)

// Check if New Arch is active
import { TurboModuleRegistry } from 'react-native';
const isNewArch = TurboModuleRegistry.getEnforcing !== undefined;
```

---

### Q266. How do you optimise navigation performance between screens?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Navigation Performance

**Answer:**
```js
// Problem: Heavy screens cause janky navigation transitions

// 1. Defer data loading until after transition (InteractionManager)
const ScreenB = () => {
  const [ready, setReady] = useState(false);

  useEffect(() => {
    const task = InteractionManager.runAfterInteractions(() => setReady(true));
    return () => task.cancel();
  }, []);

  if (!ready) return <SkeletonScreen />;
  return <HeavyContent />;
};

// 2. Use Native Stack (hardware-accelerated transitions)
import { createNativeStackNavigator } from '@react-navigation/native-stack';
// NativeStack uses native UIViewController/Fragment transitions
// vs Stack which animates in JS

// 3. Prevent heavy screens from re-rendering on focus
// Navigation.setOptions forces a re-render — move to useFocusEffect
useFocusEffect(
  useCallback(() => {
    loadUpdatedData(); // only on focus, not on every navigation state change
  }, [])
);

// 4. Lazy-load tab screens
<Tab.Navigator
  screenOptions={{ lazy: true }} // don't render until first visit
>

// 5. Pre-load likely-next screen data
// When user is on Product List, pre-fetch the first product detail
const prefetchProductDetail = useCallback(async (productId) => {
  await queryClient.prefetchQuery(['product', productId], fetchProduct);
}, []);

// 6. Minimise transition listener overhead
// Don't add listeners in every screen — use navigation state at root
```

---

### Q267. What is `useCallback` and when does it NOT help?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Performance

**Answer:**
```js
// useCallback memoises a function reference across renders
// ONLY helps when passing the function to a memoised child or as a dependency

// ✅ useCallback HELPS here — passed to React.memo child
const handlePress = useCallback((id) => {
  navigation.navigate('Detail', { id });
}, [navigation]);

// MemoChild only re-renders if handlePress reference changes
<MemoChild onPress={handlePress} />

// ❌ useCallback DOESN'T HELP here — passed to native element
const handleTextChange = useCallback((text) => {
  setQuery(text);
}, []); // no benefit — TextInput doesn't do reference equality

<TextInput onChangeText={handleTextChange} /> // React never checks function ref on native elements

// ❌ useCallback DOESN'T HELP — child is not memoised
const handle = useCallback(() => doSomething(), []);
<NonMemoChild onClick={handle} /> // always re-renders anyway

// ❌ Premature optimisation — cheap components
const handlePress = useCallback(() => setCount(c => c + 1), []);
// Counter costs: closure allocation + comparison work
// vs Benefit: prevents re-render of simple Text component
// NOT worth it

// When useCallback IS worth it:
// 1. Function passed to React.memo child
// 2. Function used as dependency in useEffect
// 3. Function passed to FlatList renderItem or keyExtractor
// 4. Function passed to useCallback or useMemo dependencies

// Checking if useCallback helped:
// Before: ChildComponent.whyDidYouRender = true → logs "function changed"
// After: no log → reference is stable
```

---

### Q268. How do you profile native module calls?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Profiling

**Answer:**
```js
// Method 1: Systrace (Android profiler)
// 1. Enable in dev menu → Start Systrace
// 2. Perform the action
// 3. Stop → open trace in chrome://tracing
// Look for: callNativeModule events, their duration and frequency

// Method 2: Custom timing wrapper
const profiledModule = new Proxy(NativeModule, {
  get: (target, methodName) => {
    const original = target[methodName];
    if (typeof original !== 'function') return original;
    return (...args) => {
      const start = Date.now();
      const result = original.apply(target, args);
      if (result && typeof result.then === 'function') {
        return result.then(res => {
          console.log(`[Native] ${methodName}: ${Date.now() - start}ms`);
          return res;
        });
      }
      console.log(`[Native] ${methodName}: ${Date.now() - start}ms`);
      return result;
    };
  }
});

// Method 3: Flipper → Network Plugin
// Shows all native HTTP calls, their timing
// Shows AsyncStorage operations in the Storage plugin

// Method 4: Android Studio Profiler
// Connect physical Android device
// Android Studio → Profile → CPU → Record method trace
// Shows: each native method call, thread, duration

// Method 5: Xcode Instruments
// Instruments → Time Profiler
// Filter by thread name "com.yourapp.RCTModuleThread"
// Shows: native method call tree

// Key metrics to monitor:
// - AsyncStorage reads > 10ms = consider MMKV
// - Network calls > 200ms = add loading state
// - Native module calls > 16ms = optimise or move to TurboModule
```

---

### Q269. How does React Native's garbage collection work and affect performance?

**Difficulty:** 🔴 Hard | **Frequency:** Low | **Category:** Memory + JS Engine

**Answer:**
```js
// Hermes GC (default in RN 0.70+):
// Uses a generational garbage collector
// - Young generation: short-lived objects (most JS objects)
// - Old generation: long-lived objects (closures, large arrays)
// GC runs automatically but can cause "GC pauses" — brief JS thread freezes

// Signs of GC pressure:
// - Occasional frame drops every few seconds
// - Memory usage grows steadily then drops suddenly (sawtooth pattern)
// - Chrome/Hermes profiler shows "Minor GC" / "Major GC" events

// Reducing GC pressure:

// 1. Avoid creating many short-lived objects in hot paths
// ❌ Inside renderItem (called every frame during scroll)
const renderItem = ({ item }) => {
  const style = { fontSize: 16, color: '#333' }; // new object EVERY render
  return <Text style={style}>{item.name}</Text>;
};

// ✅ Static styles
const styles = StyleSheet.create({ text: { fontSize: 16, color: '#333' } });
const renderItem = ({ item }) => <Text style={styles.text}>{item.name}</Text>;

// 2. Avoid large array operations in render
// ❌ Spreads create new arrays
const allItems = [...list1, ...list2, ...list3]; // in component body

// ✅ useMemo
const allItems = useMemo(() => [...list1, ...list2, ...list3], [list1, list2, list3]);

// 3. Object pooling for frequently created/destroyed objects
const messagePool = [];
const getMessageObject = () => messagePool.pop() || {};
const releaseMessageObject = (msg) => {
  Object.keys(msg).forEach(k => delete msg[k]); // clear
  messagePool.push(msg); // return to pool
};

// 4. WeakMap/WeakRef for caches — allows GC to collect entries
const cache = new WeakMap(); // entries collected when key is GC'd
```

---

### Q270. How do you implement infinite scroll with optimal performance?

**Difficulty:** 🟡 Medium | **Frequency:** Very High | **Category:** FlatList + Performance

**Answer:**
```js
const PAGE_SIZE = 20;

const InfiniteScrollList = () => {
  const [data, setData] = useState([]);
  const [page, setPage] = useState(1);
  const [loading, setLoading] = useState(false);
  const [hasMore, setHasMore] = useState(true);
  const isLoadingRef = useRef(false); // prevent duplicate requests

  const loadMore = useCallback(async () => {
    if (isLoadingRef.current || !hasMore) return;
    isLoadingRef.current = true;
    setLoading(true);

    try {
      const newItems = await fetchEmployees({ page, limit: PAGE_SIZE });
      setData(prev => [...prev, ...newItems]);
      setPage(prev => prev + 1);
      setHasMore(newItems.length === PAGE_SIZE);
    } finally {
      setLoading(false);
      isLoadingRef.current = false;
    }
  }, [page, hasMore]);

  useEffect(() => { loadMore(); }, []); // initial load

  const renderItem = useCallback(({ item }) => (
    <EmployeeRow employee={item} />
  ), []);

  const keyExtractor = useCallback((item) => item.id.toString(), []);

  const ListFooter = useCallback(() => (
    loading ? <ActivityIndicator style={{ padding: 16 }} /> :
    !hasMore ? <Text style={styles.endText}>No more employees</Text> : null
  ), [loading, hasMore]);

  return (
    <FlatList
      data={data}
      keyExtractor={keyExtractor}
      renderItem={renderItem}
      ListFooterComponent={ListFooter}

      onEndReached={loadMore}
      onEndReachedThreshold={0.5} // trigger when 50% from bottom

      // Performance props
      getItemLayout={(_, index) => ({ length: 72, offset: 72 * index, index })}
      initialNumToRender={PAGE_SIZE}
      maxToRenderPerBatch={10}
      windowSize={7}
      removeClippedSubviews={true}
    />
  );
};
```

---

### Q271. What is `onEndReachedThreshold` and how do you tune it?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** FlatList

**Answer:**
`onEndReachedThreshold` controls how far from the bottom of the list (as a fraction of the visible length) `onEndReached` fires.

```js
// 0.5 = trigger when 50% of visible list remains below the current position
// 0.1 = trigger when 10% remains (very close to bottom)
// 2   = trigger 2x viewport before end (pre-fetch aggressively)

// For most apps: 0.5 is the sweet spot
<FlatList
  onEndReached={loadNextPage}
  onEndReachedThreshold={0.5}
/>

// Problem: onEndReached fires multiple times
// This is a known FlatList bug — guard with isLoading ref
const onEndReached = useCallback(() => {
  if (!isLoadingRef.current && hasMore) {
    loadNextPage();
  }
}, [hasMore]);

// Network latency tuning:
// Fast API (< 200ms): threshold 0.3 — start loading close to end
// Slow API (> 500ms): threshold 1.0 or 2.0 — start loading early

// For grids (numColumns > 1):
// Threshold behaves the same way but items-per-row means you need fewer rows
<FlatList
  numColumns={3}
  onEndReachedThreshold={0.5} // still 50% of visible area
  onEndReached={loadMore}
/>
```

---

### Q272. How do you handle performance on older Android devices (low-end)?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Performance + Android

**Answer:**
```js
// Target: devices with 1-2GB RAM, Android 5-8, older CPUs
// These are common in India (your target market for 10-12 LPA jobs)

// 1. Detect low-end device and adjust behaviour
import { Platform } from 'react-native';
import DeviceInfo from 'react-native-device-info';

const [isLowEnd, setIsLowEnd] = useState(false);
useEffect(() => {
  DeviceInfo.getTotalMemory().then(bytes => {
    setIsLowEnd(bytes < 2 * 1024 * 1024 * 1024); // < 2GB
  });
}, []);

// 2. Reduce animation complexity on low-end
const animationDuration = isLowEnd ? 0 : 300;
const useAnimations = !isLowEnd;

// 3. Smaller windowSize for FlatList
<FlatList windowSize={isLowEnd ? 3 : 7} />

// 4. Lower image quality
const imageQuality = isLowEnd ? 50 : 80;
const imageUrl = `https://cdn.example.com/image.jpg?q=${imageQuality}`;

// 5. Disable heavy effects
{!isLowEnd && <BlurView />}

// 6. Reduce bundle size
// Hermes helps significantly — smaller bytecode, better GC
// Remove polyfills not needed for modern Android (API 21+)

// 7. Avoid large inline styles in FlatList (StyleSheet.create)

// 8. Throttle expensive operations more aggressively
const throttleMs = isLowEnd ? 500 : 200;
const debouncedSearch = useMemo(() => debounce(search, throttleMs), [throttleMs]);

// 9. Use MMKV instead of AsyncStorage
// MMKV: synchronous, C++ backed, ~10x faster than AsyncStorage
import { MMKV } from 'react-native-mmkv';
const storage = new MMKV();
storage.set('key', 'value');
const value = storage.getString('key'); // synchronous!
```

---

### Q273. What is `react-native-mmkv` and why is it faster than `AsyncStorage`?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Storage + Performance

**Answer:**
MMKV (Mobile Mapped Key-Value) is a C++ key-value storage library developed by WeChat. It's 10-100x faster than AsyncStorage.

| | AsyncStorage | MMKV |
|--|-------------|------|
| Implementation | JS + Native (slow) | C++ with mmap |
| Read API | Async (Promise) | Synchronous |
| Write API | Async (Promise) | Synchronous |
| Encryption | No built-in | AES-256 |
| Multi-process | No | Yes |
| Typical read speed | 5-20ms | 0.01-0.1ms |

```js
import { MMKV } from 'react-native-mmkv';

// Default storage
const storage = new MMKV();

// With encryption
const encryptedStorage = new MMKV({ id: 'secure', encryptionKey: 'my-secret-key' });

// Read/write (synchronous — no await needed!)
storage.set('userId', '12345');
storage.set('theme', 'dark');
storage.set('count', 42);
storage.set('preferences', JSON.stringify({ fontSize: 16, language: 'en' }));

const userId = storage.getString('userId');    // '12345'
const theme = storage.getString('theme');      // 'dark'
const count = storage.getNumber('count');      // 42
const prefs = JSON.parse(storage.getString('preferences') ?? '{}');

// Delete
storage.delete('userId');
storage.clearAll();

// Check existence
const hasKey = storage.contains('theme');

// Integrate with React state
const useMMKVString = (key, defaultValue = '') => {
  const [value, setValue] = useState(() => storage.getString(key) ?? defaultValue);

  const setStoredValue = useCallback((newValue) => {
    setValue(newValue);
    storage.set(key, newValue);
  }, [key]);

  return [value, setStoredValue];
};

// Use Zustand with MMKV persistence
import { persist, createJSONStorage } from 'zustand/middleware';

const useStore = create(persist(
  (set) => ({ theme: 'light', setTheme: (t) => set({ theme: t }) }),
  { name: 'app-storage', storage: createJSONStorage(() => storage) }
));
```

---

### Q274. How do you detect and fix slow renders with `React Profiler`?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Profiling

**Answer:**
```js
import { Profiler } from 'react';

// Wrap suspicious component trees
<Profiler
  id="EmployeeDashboard"
  onRender={(
    id,           // component tree identifier
    phase,        // 'mount' or 'update'
    actualDuration, // ms this render took
    baseDuration,   // estimated time without memoisation
    startTime,      // when React started rendering
    commitTime,     // when React committed the render
  ) => {
    // Log slow renders
    if (actualDuration > 16) {
      console.warn(`SLOW RENDER [${id}] phase=${phase}: ${actualDuration.toFixed(1)}ms`);
    }

    // Track in analytics
    if (!__DEV__) {
      analytics.track('slow_render', { screen: id, duration: actualDuration });
    }
  }}
>
  <EmployeeDashboard />
</Profiler>

// Analyse baseDuration vs actualDuration:
// actualDuration << baseDuration → memoisation is working well
// actualDuration ≈ baseDuration → memoisation not helping
// actualDuration > 16ms → need to optimise

// Find the slow component in the Profiler output:
// 1. Record a session in React DevTools Profiler
// 2. Click the slowest commit (red bar in timeline)
// 3. Flamegraph → hover bars to see render time per component
// 4. Ranked chart → sorted by self time — top = slowest

// After fixing:
// Re-run profiler → baseDuration should drop
// If actualDuration drops but baseDuration stays same → memo helped
// If baseDuration drops → you made the render itself cheaper
```

---

### Q275. What is the `Performance` API in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** Low | **Category:** Profiling

**Answer:**
React Native (Hermes) supports a subset of the Web Performance API for precise timing measurements.

```js
// performance.now() — high-resolution timestamp (fractional ms)
const start = performance.now();
expensiveOperation();
const duration = performance.now() - start;
console.log(`Operation took: ${duration.toFixed(3)}ms`);

// performance.mark() — named timestamp
performance.mark('dataFetchStart');
const data = await fetchDashboardData();
performance.mark('dataFetchEnd');

// performance.measure() — duration between two marks
performance.measure('dataFetch', 'dataFetchStart', 'dataFetchEnd');
const [measure] = performance.getEntriesByName('dataFetch');
console.log(`Data fetch: ${measure.duration.toFixed(1)}ms`);

// performance.getEntries() — all recorded entries
const entries = performance.getEntries();
entries.forEach(entry => {
  console.log(`${entry.name}: ${entry.duration?.toFixed(1)}ms`);
});

// Clear entries
performance.clearMarks();
performance.clearMeasures();

// Custom performance monitoring utility
const perfMonitor = {
  marks: {},
  start: (label) => { perfMonitor.marks[label] = performance.now(); },
  end: (label) => {
    const duration = performance.now() - (perfMonitor.marks[label] || 0);
    delete perfMonitor.marks[label];
    if (__DEV__) console.log(`⏱ ${label}: ${duration.toFixed(1)}ms`);
    return duration;
  },
};

perfMonitor.start('screenLoad');
await loadScreenData();
perfMonitor.end('screenLoad'); // ⏱ screenLoad: 342.1ms
```

---

### Q276. How do you lazy-load images in a FlatList?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Images + Performance

**Answer:**
```js
// FlatList already virtualises rendering — images outside the window
// are unmounted, cancelling any pending loads.
// Additional optimisations:

// 1. FastImage with priority
import FastImage from 'react-native-fast-image';

const EmployeeAvatar = React.memo(({ uri, size }) => (
  <FastImage
    source={{
      uri,
      priority: FastImage.priority.normal,
      cache: FastImage.cacheControl.immutable,
    }}
    style={{ width: size, height: size, borderRadius: size / 2 }}
    resizeMode={FastImage.resizeMode.cover}
  />
));

// 2. Progressive loading with blur placeholder
const ProgressiveImage = React.memo(({ uri, thumbUri, style }) => {
  const [fullLoaded, setFullLoaded] = useState(false);

  return (
    <View style={style}>
      {/* Low-res thumbnail loads first */}
      <FastImage
        source={{ uri: thumbUri }}
        style={[StyleSheet.absoluteFillObject, { opacity: fullLoaded ? 0 : 1 }]}
        blurRadius={2}
      />
      {/* Full image fades in */}
      <FastImage
        source={{ uri }}
        style={[StyleSheet.absoluteFillObject, { opacity: fullLoaded ? 1 : 0 }]}
        onLoad={() => setFullLoaded(true)}
      />
    </View>
  );
});

// 3. Preload next page images
const preloadNextPage = (items) => {
  FastImage.preload(items.map(item => ({
    uri: item.avatarUrl,
    priority: FastImage.priority.low,
  })));
};

// 4. FlatList viewability callback — preload when item enters viewport
const onViewableItemsChanged = useCallback(({ viewableItems }) => {
  const nextItems = getNextItems(viewableItems);
  FastImage.preload(nextItems.map(i => ({ uri: i.item.avatarUrl })));
}, []);

<FlatList
  onViewableItemsChanged={onViewableItemsChanged}
  viewabilityConfig={{ itemVisiblePercentThreshold: 50 }}
/>
```

---

### Q277. What is `shouldRasterizeIOS` and `renderToHardwareTextureAndroid`?

**Difficulty:** 🔴 Hard | **Frequency:** Low | **Category:** Rendering Performance

**Answer:**
Both offload component rendering to the GPU as a texture — the layer is rendered once and composited, avoiding re-rendering on every frame. Use for static complex components inside scroll views or animations.

```js
// shouldRasterizeIOS — iOS: rasterize to a composited layer
<View
  shouldRasterizeIOS={true}       // iOS only
  style={styles.complexCard}
>
  <ComplexCardContent />           {/* rendered once to texture */}
</View>

// renderToHardwareTextureAndroid — Android: render to GPU texture
<View
  renderToHardwareTextureAndroid={true}  // Android only
  style={styles.complexCard}
>
  <ComplexCardContent />
</View>

// Cross-platform wrapper
const GPU_LAYER_PROPS = Platform.select({
  ios: { shouldRasterizeIOS: true },
  android: { renderToHardwareTextureAndroid: true },
});

<View {...GPU_LAYER_PROPS} style={styles.card}>
  <ComplexContent />
</View>

// When to use:
// ✅ Complex static UI inside a scrolling list (shadows, gradients, many children)
// ✅ Animating a view that doesn't change its contents (move/scale/opacity only)
// ✅ Bottom tabs, headers that animate but don't redraw

// When NOT to use:
// ❌ Content that changes frequently — GPU texture must be regenerated each time
// ❌ Views with changing text — re-rasterisation is expensive
// ❌ Simple views (overhead > benefit)

// Note: consumes GPU memory for the texture
// Too many rasterised layers → GPU memory pressure → slower
```

---

### Q278. How do you implement background task performance optimisation?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Background + Performance

**Answer:**
```js
// Background tasks in React Native have strict time limits
// iOS: ~30 seconds of background execution via BackgroundFetch
// Android: WorkManager for deferred tasks

// 1. react-native-background-fetch (cross-platform)
import BackgroundFetch from 'react-native-background-fetch';

BackgroundFetch.configure({
  minimumFetchInterval: 15,  // minutes (minimum 15 on iOS)
  stopOnTerminate: false,
  startOnBoot: true,
  enableHeadless: true,
}, async (taskId) => {
  // Perform quick, essential work only
  await syncAttendanceData();
  BackgroundFetch.finish(taskId); // MUST call within time limit
}, (taskId) => {
  console.log('[BackgroundFetch] TIMEOUT:', taskId);
  BackgroundFetch.finish(taskId);
});

// 2. Prioritise background work
const syncInBackground = async () => {
  // Order by priority:
  await syncCriticalData();    // attendance, payroll = high priority
  if (backgroundTimeLeft() > 5000) {
    await syncSecondaryData(); // reports, analytics = lower priority
  }
};

// 3. Incremental sync — don't sync everything, only changed data
const lastSyncTime = await MMKV.getNumber('lastSync') ?? 0;
const changedData = await api.getChangedSince(lastSyncTime);
await db.upsertAll(changedData);
await MMKV.set('lastSync', Date.now());

// 4. react-native-background-actions for long-running background service (Android)
import BackgroundService from 'react-native-background-actions';

const uploadTask = async (taskData) => {
  const { files } = taskData;
  for (let i = 0; i < files.length; i++) {
    await uploadFile(files[i]);
    await BackgroundService.updateNotification({
      taskDesc: `Uploading ${i+1}/${files.length}`,
      progressBar: { max: files.length, value: i + 1 },
    });
  }
};
```

---

### Q279. How do you handle performance for real-time data (WebSocket updates)?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Real-time + Performance

**Answer:**
```js
// Problem: Frequent WebSocket messages → many setState calls → many re-renders

// Strategy 1: Throttle/batch state updates
const pendingUpdates = useRef([]);
const flushTimeout = useRef(null);

const handleWebSocketMessage = useCallback((message) => {
  pendingUpdates.current.push(message);

  // Flush at most every 100ms (10fps for data updates)
  if (!flushTimeout.current) {
    flushTimeout.current = setTimeout(() => {
      const updates = pendingUpdates.current;
      pendingUpdates.current = [];
      flushTimeout.current = null;

      // Batch all updates into single state update
      setMessages(prev => [...prev, ...updates]);
    }, 100);
  }
}, []);

// Strategy 2: Use a state manager that supports bulk updates
// Zustand batches automatically
const useStore = create((set) => ({
  messages: [],
  addMessages: (msgs) => set(state => ({ messages: [...state.messages, ...msgs] })),
}));

// Strategy 3: Normalised state — update by ID, not push to array
// Prevents duplicates and makes updates O(1) instead of O(n) scan
const handleUpdate = (update) => {
  setEmployees(prev => ({
    ...prev,
    [update.id]: { ...prev[update.id], ...update.data },
  }));
};

// Strategy 4: Virtualised list handles new items gracefully
// Adding to head with prepend strategy
const [data, setData] = useState([]);
const handleNewMessage = useCallback((msg) => {
  setData(prev => [msg, ...prev.slice(0, MAX_MESSAGES)]); // cap at MAX_MESSAGES
}, []);

// Strategy 5: Memoize expensive derivations
const unreadCount = useMemo(
  () => messages.filter(m => !m.read).length,
  [messages]
);
```

---

### Q280. What is the `initialNumToRender` prop and how do you choose it?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** FlatList

**Answer:**
`initialNumToRender` is the number of items rendered **synchronously** in the first render. These are displayed immediately without any async batch work.

```js
// Rule of thumb: set to slightly more than what fills the screen
// If each item is 72px and screen is 800px: 800/72 ≈ 11 → use 12-15

<FlatList
  initialNumToRender={12}    // renders 12 items synchronously
  data={employees}
  renderItem={renderItem}
/>

// Too LOW (e.g., 1):
// - First render shows only 1 item (blank space below)
// - Items "pop in" asynchronously — bad UX
// - Good for: TTI improvement when list is below the fold

// Too HIGH (e.g., 100):
// - First render is slow (100 items × render time)
// - Worse TTI even though list looks full immediately
// - Wastes time rendering items that won't be visible

// Calibration:
// Measure screen height: const { height } = useWindowDimensions()
// Measure item height: getItemLayout or onLayout
// Compute: Math.ceil(screenHeight / itemHeight) + 2 (buffer)

const { height: screenHeight } = useWindowDimensions();
const ITEM_HEIGHT = 72;
const INITIAL_ITEMS = Math.ceil(screenHeight / ITEM_HEIGHT) + 2;

<FlatList initialNumToRender={INITIAL_ITEMS} />

// For screens where the list is NOT immediately visible (below header):
// Can use smaller value since list won't be seen immediately
// For screens where the list IS immediately visible:
// Use exact screen-fill value or slightly more
```

---

### Q281. How do you handle font loading performance?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Startup Performance

**Answer:**
```js
// Problem: Custom fonts not loading before first render → FOUT (Flash of Unstyled Text)

// Solution 1: expo-font (Expo)
import * as Font from 'expo-font';
import { useFonts } from 'expo-font';
import AppLoading from 'expo-app-loading';

const App = () => {
  const [fontsLoaded] = useFonts({
    'Inter-Regular': require('./assets/fonts/Inter-Regular.ttf'),
    'Inter-Medium': require('./assets/fonts/Inter-Medium.ttf'),
    'Inter-Bold': require('./assets/fonts/Inter-Bold.ttf'),
  });

  if (!fontsLoaded) return <AppLoading />; // show native splash until fonts load

  return <AppContent />;
};

// Solution 2: react-native-vector-icons (CLI)
// Fonts are linked natively via autolinking or Info.plist/build.gradle
// Available immediately — no loading needed

// Solution 3: System fonts (no loading, best performance)
const fontFamily = Platform.select({
  ios: 'SF Pro Display',     // system font on iOS
  android: 'Roboto',         // system font on Android
});
// Zero loading time — fonts are built into the OS

// Font loading performance tips:
// 1. Subset fonts — only include characters you need
//    fonttools + pyftsubset can reduce font size by 80%
// 2. Preload at app start — block first render until fonts ready
// 3. Use variable fonts — one file instead of 6 weights
// 4. woff2 format for web, but RN uses .ttf — keep .ttf small
```

---

### Q282. What is tree shaking and does it work in React Native?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Bundle Optimisation

**Answer:**
Tree shaking = static analysis to eliminate unused exports from a bundle. Webpack supports it for ES modules. Metro (RN's bundler) has **limited** tree shaking support.

```js
// Metro tree shaking limitations:
// - Metro uses CommonJS (require/module.exports) internally
// - CommonJS is not statically analysable → full modules included
// - ES modules (import/export) can be tree-shaken by Metro in some cases

// What DOES work in Metro:
// Import specific named exports when library uses ES modules
// ✅ lodash-es (ES module version)
import { debounce, throttle } from 'lodash-es';
// Only debounce and throttle included in bundle

// ❌ Regular lodash (CommonJS)
import { debounce } from 'lodash';
// Entire lodash included (despite named import!)

// Libraries that are tree-shakeable in Metro:
// date-fns, lodash-es, @reduxjs/toolkit, zustand, react-query

// Manual tree shaking strategies:
// 1. Direct file imports
import debounce from 'lodash/debounce'; // only debounce module
import format from 'date-fns/format';   // only format module

// 2. babel-plugin-import (automatic direct imports)
// babel.config.js:
plugins: [
  ['import', { libraryName: 'lodash', libraryDirectory: '', camel2DashComponentName: false }]
]
// import { debounce } from 'lodash' → import debounce from 'lodash/debounce'

// 3. Async imports for large optional features
const loadPDFViewer = () => import('./PDFViewer'); // loaded on demand
```

---

### Q283. How do you detect memory usage programmatically in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** Low | **Category:** Memory Monitoring

**Answer:**
```js
// Method 1: react-native-device-info
import DeviceInfo from 'react-native-device-info';

const checkMemory = async () => {
  const totalMemory = await DeviceInfo.getTotalMemory();      // bytes
  const usedMemory = await DeviceInfo.getUsedMemory();        // bytes (Android only)
  const freeMemory = totalMemory - usedMemory;

  console.log(`Total: ${(totalMemory / 1e9).toFixed(1)}GB`);
  console.log(`Used: ${(usedMemory / 1e6).toFixed(0)}MB`);

  // Alert if app using too much memory
  if (usedMemory > 500 * 1e6) { // > 500MB
    console.warn('High memory usage! Consider clearing caches.');
  }
};

// Method 2: Performance memory (limited support)
if (performance.memory) {
  console.log('Heap size:', performance.memory.usedJSHeapSize / 1e6, 'MB');
}

// Method 3: Flipper Memory tab (development only)
// Connect Flipper → Memory → Take heap snapshot
// Allocations view shows live memory growth
// Heap snapshot shows what's consuming memory

// Method 4: Android Studio Memory Profiler
// Profile → Memory → record allocations
// Find largest retained objects

// Method 5: Platform-specific native calls
import { NativeModules } from 'react-native';
const { MemoryModule } = NativeModules;
// Requires custom native module to expose Android's ActivityManager

// Proactive memory management:
useEffect(() => {
  // Clear image caches when app goes to background
  const sub = AppState.addEventListener('change', (state) => {
    if (state === 'background') {
      FastImage.clearMemoryCache();
      queryClient.clear(); // React Query cache
    }
  });
  return () => sub.remove();
}, []);
```

---

### Q284. What is `keyExtractor` and why does a stable key matter for performance?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** FlatList Performance

**Answer:**
```js
// keyExtractor tells React which rendered item corresponds to which data item
// Used for reconciliation — React reuses components when data changes
// Stable, unique keys = React reuses existing components (fast)
// Unstable keys = React unmounts + remounts components (slow)

// ✅ Stable unique key — React reuses components on sort/filter
const keyExtractor = useCallback((item) => item.id.toString(), []);

// ❌ Using index — unstable on sort/filter/insert
const keyExtractor = (item, index) => index.toString();
// If you insert an item at position 0:
// All items after shift — new keys → all remounted!

// ❌ Math.random() — always remounts everything
const keyExtractor = () => Math.random().toString();

// ✅ Composite key for items that might have duplicate IDs across types
const keyExtractor = (item) => `${item.type}-${item.id}`;

// ✅ For nested lists
const keyExtractor = (item) => `section-${item.sectionId}-item-${item.id}`;

// What happens with wrong keys:
// 1. Re-renders all items (performance)
// 2. Loses focused state (TextInput focus lost on filter)
// 3. Loses scroll position
// 4. Animations reset on every data change

// FlatList logs a warning in dev if key is missing or not unique
// Always see: "Each child in a list should have a unique 'key' prop"

// keyExtractor vs key prop:
// FlatList: use keyExtractor prop on FlatList component
// Manual map: use key prop directly on the element
{items.map(item => <Row key={item.id.toString()} item={item} />)}
```

---

### Q285. How do you implement pagination with a cursor in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Pagination + Performance

**Answer:**
```js
// Cursor-based pagination: more efficient than offset for large datasets
// Uses a cursor (usually last item ID) instead of page number
// Consistent results even when data is inserted/deleted

const useCursorPagination = (fetcher) => {
  const [items, setItems] = useState([]);
  const [cursor, setCursor] = useState(null);
  const [hasMore, setHasMore] = useState(true);
  const [loading, setLoading] = useState(false);
  const isLoadingRef = useRef(false);

  const loadMore = useCallback(async () => {
    if (isLoadingRef.current || !hasMore) return;
    isLoadingRef.current = true;
    setLoading(true);

    try {
      // API: GET /employees?cursor=<lastId>&limit=20
      const { data, nextCursor, hasMore: more } = await fetcher(cursor);

      setItems(prev => {
        // Deduplicate using Set of IDs
        const existingIds = new Set(prev.map(i => i.id));
        const newItems = data.filter(i => !existingIds.has(i.id));
        return [...prev, ...newItems];
      });
      setCursor(nextCursor);
      setHasMore(more);
    } catch (err) {
      console.error('Pagination error:', err);
    } finally {
      setLoading(false);
      isLoadingRef.current = false;
    }
  }, [cursor, hasMore, fetcher]);

  const refresh = useCallback(async () => {
    setItems([]);
    setCursor(null);
    setHasMore(true);
    isLoadingRef.current = false;
    // loadMore will run on next call
  }, []);

  return { items, loadMore, loading, hasMore, refresh };
};

// Usage
const { items, loadMore, loading, hasMore } = useCursorPagination(
  (cursor) => fetchEmployees({ cursor, limit: 20 })
);

<FlatList
  data={items}
  renderItem={renderItem}
  onEndReached={loadMore}
  onEndReachedThreshold={0.5}
  refreshing={loading && items.length === 0}
  onRefresh={refresh}
/>
```

---

### Q286. How do you reduce the APK/IPA size in React Native?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Bundle + App Size

**Answer:**
```bash
# Android APK size reduction:

# 1. Enable app bundles (AAB) instead of APK (recommended for Play Store)
# android/app/build.gradle
android {
  bundle { language { enableSplit = true } density { enableSplit = true } resources { enableSplit = true } }
}
# Play Store delivers only the APK for user's device config → smaller download

# 2. Enable Proguard / R8 (minifies and obfuscates Java code)
# android/app/build.gradle
buildTypes {
  release {
    minifyEnabled true
    shrinkResources true
    proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
  }
}

# 3. Split APKs by ABI (removes code for other CPU architectures)
android { splits { abi { enable true; reset(); include 'armeabi-v7a', 'arm64-v8a'; universalApk false } } }

# 4. Hermes — bytecode smaller than JS source
# (already enabled in RN 0.70+ by default)

# iOS IPA size reduction:
# 1. Enable bitcode (deprecated in Xcode 14 but still reduces size for older)
# 2. App Thinning (App Store slicing) — automatic
# 3. Strip debug symbols from release build (automatic with release config)
```

```js
// JS bundle size reduction (both platforms):

// 1. Remove unused icon sets
// android/app/build.gradle / ios/Podfile
// Only include fonts you use from react-native-vector-icons

// 2. Optimize images
// Use WebP instead of PNG/JPEG
// Resize to actual display size before bundling

// 3. Remove development-only packages from production
// Check package.json: move dev tools to devDependencies
// "react-native-flipper" should be devDependency

// 4. Analyse what's in your bundle (Q243)
// source-map-explorer to find large dependencies

// 5. Lazy require large optional modules
const getPDF = () => require('react-native-pdf'); // loaded on demand
```

---

### Q287. How do you handle slow startup on Android specifically?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Android + Startup

**Answer:**
```java
// Android cold start is typically slower than iOS due to JVM startup + Dalvik

// 1. Application class — defer non-critical initialisation
// android/app/src/main/java/com/yourapp/MainApplication.kt
class MainApplication : Application(), ReactApplication {
  override fun onCreate() {
    super.onCreate()
    // ✅ Only initialise what's needed for first screen
    SoLoader.init(this, false)
    // ❌ Don't do this here — defer it
    // initHeavyLibrary() — move to after first render
  }
}
```

```js
// 2. ReactNativeFlipper.initializeFlipper only in DEBUG
// Already handled by default React Native template

// 3. Preload React Native bridge in Application (for fastest launch)
// android/app/src/main/.../MainActivity.kt
// ReactActivityDelegate.fabricEnabled = true (for New Arch)

// 4. Splash screen — native Android splash (not RN)
// react-native-splash-screen or expo-splash-screen
// Show IMMEDIATELY on app launch, before RN bridge initialises
// activity_splash.xml → immediate display, no wait for React

// 5. Application.onCreate profiling
// TraceCompat.beginSection("YourApp.initLibrary")
// initLibrary()
// TraceCompat.endSection()
// Then: adb shell am start-activity -W com.yourapp/...
// View in Android Studio → Profiler

// 6. Multi-dex (Android < 5.0 / large apps)
// android/app/build.gradle
// defaultConfig { multiDexEnabled true }
// implementation 'androidx.multidex:multidex:2.0.1'
// Prevents "64K reference limit" crash but slightly slower startup
// Not needed for most apps targeting API 21+

// 7. Baseline Profiles (Android 9+ / Google Play)
// Pre-compiles hot code paths at install time
// 20-40% startup improvement on first launch after install
```

---

### Q288. What is `Fabric` (New Architecture renderer) and how does it improve performance?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Architecture

**Answer:**
Fabric is the new rendering system in React Native's New Architecture that replaces the old UIManager.

**Old rendering (UIManager):**
```
React reconciler (JS) → Async Bridge → UIManager (native) → Native views
    3 threads, async, serialised JSON
```

**Fabric rendering:**
```
React reconciler (JS) → JSI (synchronous) → C++ Shadow Tree → Native views
    Shared C++ layer, synchronous, no serialisation
```

```js
// Benefits of Fabric:

// 1. Synchronous layout measurements
// Old: measure() is async → callback-based
// Fabric: measurements can be synchronous (same frame)

// 2. Concurrent rendering support
// React 18 concurrent features (Suspense, startTransition) work in Fabric
// Old arch: not supported
import { startTransition } from 'react';

const handleSearch = (query) => {
  startTransition(() => {
    setSearchResults(filterEmployees(query)); // low-priority update
  });
  setSearchInput(query); // high-priority update (immediate)
};

// 3. Better error handling
// Errors in Fabric are caught at render boundary (ErrorBoundary works better)

// 4. Native components can be written in C++
// Shared code between iOS and Android

// 5. React Native Web interoperability
// Fabric and React DOM share component model

// Enable:
// android/gradle.properties: newArchEnabled=true
// ios/Podfile: ENV['RCT_NEW_ARCH_ENABLED'] = '1'
```

---

### Q289. How do you implement pagination cache to avoid re-fetching?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Performance + Caching

**Answer:**
```js
// React Query (TanStack Query) — best in class for server state caching
import { useInfiniteQuery } from '@tanstack/react-query';

const useEmployees = () => {
  return useInfiniteQuery({
    queryKey: ['employees'],
    queryFn: ({ pageParam = null }) =>
      fetchEmployees({ cursor: pageParam, limit: 20 }),
    getNextPageParam: (lastPage) => lastPage.nextCursor ?? undefined,
    staleTime: 5 * 60 * 1000,      // data considered fresh for 5 minutes
    cacheTime: 10 * 60 * 1000,     // keep in cache for 10 minutes
    refetchOnWindowFocus: false,   // don't refetch on app foreground (use staleTime)
    refetchOnMount: false,         // use cached data on re-mount if fresh
  });
};

const EmployeeList = () => {
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useEmployees();

  const flatData = useMemo(
    () => data?.pages.flatMap(page => page.employees) ?? [],
    [data]
  );

  return (
    <FlatList
      data={flatData}
      renderItem={renderItem}
      keyExtractor={keyExtractor}
      onEndReached={() => hasNextPage && fetchNextPage()}
      onEndReachedThreshold={0.5}
      ListFooterComponent={isFetchingNextPage ? <ActivityIndicator /> : null}
    />
  );
};

// Manual caching with MMKV
const CACHE_TTL = 5 * 60 * 1000; // 5 minutes

const getCachedEmployees = async (cursor) => {
  const cacheKey = `employees-${cursor ?? 'start'}`;
  const cached = storage.getString(cacheKey);

  if (cached) {
    const { data, timestamp } = JSON.parse(cached);
    if (Date.now() - timestamp < CACHE_TTL) return data; // fresh
  }

  const fresh = await fetchEmployees({ cursor });
  storage.set(cacheKey, JSON.stringify({ data: fresh, timestamp: Date.now() }));
  return fresh;
};
```

---

### Q290. How do you set up a performance monitoring dashboard for a production React Native app?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Monitoring + Production

**Answer:**
```js
// Production performance monitoring stack:

// 1. Crash reporting + performance: Sentry
import * as Sentry from '@sentry/react-native';

Sentry.init({
  dsn: 'https://your-dsn@sentry.io/project-id',
  tracesSampleRate: 0.2,        // 20% of sessions get performance traces
  enableAutoSessionTracking: true,
  sessionTrackingIntervalMillis: 30000,
});

// Track custom performance transactions
const transaction = Sentry.startTransaction({ name: 'loadDashboard' });
await loadDashboardData();
transaction.finish();

// Sentry automatically captures:
// - JS exceptions + native crashes
// - Slow renders (> 16ms)
// - Network requests + timing
// - App startup time

// 2. Firebase Performance Monitoring
import perf from '@react-native-firebase/perf';

const httpMetric = perf().newHttpMetric('https://api.yourapp.com/employees', 'GET');
await httpMetric.start();
const data = await fetchEmployees();
httpMetric.httpResponseCode = 200;
await httpMetric.stop();

// Custom trace
const trace = await perf().startTrace('process_payroll');
processPayroll(data);
await trace.stop();

// 3. Custom analytics events
const trackPerformance = (event, data) => {
  analytics.logEvent('performance', {
    event_name: event,
    screen: currentScreen,
    ...data,
  });
};

// Track in critical paths:
const loadStart = performance.now();
await loadScreen();
trackPerformance('screen_load', {
  screen: 'EmployeeList',
  duration: performance.now() - loadStart,
  employee_count: employees.length,
});

// 4. Alerting thresholds
// Sentry: alert if p95 screen load > 3s
// Firebase: alert if crash-free sessions < 99.5%
// Custom: alert if API error rate > 1%
```

---

## Sections Overview (Q291–Q500)

| Section | Questions | Topics |
|---------|-----------|--------|
| Native Modules | Q291–Q340 | Writing custom native modules (iOS + Android) |
| Expo vs CLI | Q341–Q370 | Managed vs bare, EAS Build, Expo Go |
| Testing | Q371–Q420 | Unit, integration, e2e (Detox), mocking |
| Debugging | Q421–Q450 | Flipper, Hermes debugger, crash reporting |
| Storage & Permissions | Q451–Q480 | AsyncStorage, Keychain, permission flows |
| Miscellaneous | Q481–Q500 | Accessibility, internationalisation, misc APIs |

---

> 💡 **Tip for GitHub:** Add a `## Table of Contents` section at the top with anchor links to each question for easy navigation.

---

*Part 01 of 8 — [← Back to Part README](./README.md) · [← Main README](../README.md)*

## Section 8: Native Modules (Q291–Q340)

---

### Q291. What is a Native Module in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** Very High | **Category:** Native Modules

**Answer:**
A Native Module is a bridge between JavaScript and platform-specific native code (Swift/Objective-C on iOS, Kotlin/Java on Android). Use them when React Native's built-in APIs don't expose the functionality you need.

**When you need a native module:**
- Access platform APIs with no JS equivalent (biometrics, Bluetooth, camera hardware features)
- Use existing native SDKs (Razorpay, Sentry native, Firebase)
- Perform CPU-intensive work in native (encryption, image processing)
- Synchronous native operations (MMKV, SQLite)

```
JavaScript                   Native
─────────────────            ─────────────────
import { NativeModules }     // iOS: @objc class BiometricModule: NSObject
const { BiometricModule }    // Android: class BiometricModule : ReactContextBaseJavaModule
  = NativeModules;
BiometricModule
  .authenticate()
  .then(success => ...)      → native SDK call → result back to JS
```

**Old Architecture:** Modules communicate over the async Bridge (JSON serialisation).
**New Architecture (TurboModules):** Modules are called synchronously via JSI with TypeScript-generated type-safe interfaces.

```js
// Consuming a native module in JS
import { NativeModules } from 'react-native';
const { DeviceSecurityModule } = NativeModules;

// Call native method
const isSecure = await DeviceSecurityModule.isDeviceSecure();

// Or wrap in a JS module for a clean API
export const DeviceSecurity = {
  isSecure: () => DeviceSecurityModule.isDeviceSecure(),
  hasBiometrics: () => DeviceSecurityModule.hasBiometrics(),
};
```

**Follow-up:** What's the difference between a Native Module and a Native Component? → Native Module exposes methods/functions to JS. Native Component (View Manager) exposes a native UI component (like `<MapView>` or `<WebView>`) as a React component.

---

### Q292. How do you create a Native Module on Android (Kotlin)?

**Difficulty:** 🔴 Hard | **Frequency:** Very High | **Category:** Native Modules — Android

**Answer:**
Three files needed: the module class, a package class, and registration in `MainApplication`.

```kotlin
// Step 1: Create the module
// android/app/src/main/java/com/yourapp/DeviceInfoModule.kt

package com.yourapp

import com.facebook.react.bridge.*
import android.os.Build

class DeviceInfoModule(reactContext: ReactApplicationContext)
    : ReactContextBaseJavaModule(reactContext) {

    // Name exposed to JavaScript: NativeModules.DeviceInfo
    override fun getName() = "DeviceInfo"

    // Synchronous method — returns value directly (no callback/promise)
    @ReactMethod(isBlockingSynchronousMethod = true)
    fun getDeviceName(): String {
        return "${Build.MANUFACTURER} ${Build.MODEL}"
    }

    // Async method with Promise
    @ReactMethod
    fun getBatteryLevel(promise: Promise) {
        try {
            val batteryManager = reactApplicationContext
                .getSystemService(android.content.Context.BATTERY_SERVICE)
                as android.os.BatteryManager
            val level = batteryManager.getIntProperty(
                android.os.BatteryManager.BATTERY_PROPERTY_CAPACITY
            )
            promise.resolve(level)
        } catch (e: Exception) {
            promise.reject("BATTERY_ERROR", e.message, e)
        }
    }

    // Method with callback
    @ReactMethod
    fun isRooted(callback: Callback) {
        val isRooted = checkIfRooted()
        callback.invoke(null, isRooted) // (error, result)
    }

    // Method that returns a Map (JS object)
    @ReactMethod
    fun getDeviceInfo(promise: Promise) {
        val map = Arguments.createMap().apply {
            putString("manufacturer", Build.MANUFACTURER)
            putString("model", Build.MODEL)
            putInt("sdkVersion", Build.VERSION.SDK_INT)
            putString("androidVersion", Build.VERSION.RELEASE)
        }
        promise.resolve(map)
    }

    // Expose constants to JS (available synchronously without method call)
    override fun getConstants(): Map<String, Any> = mapOf(
        "DEVICE_NAME" to "${Build.MANUFACTURER} ${Build.MODEL}",
        "OS_VERSION" to Build.VERSION.RELEASE,
        "SDK_INT" to Build.VERSION.SDK_INT,
    )
}
```

```kotlin
// Step 2: Create the package
// android/app/src/main/java/com/yourapp/DeviceInfoPackage.kt

package com.yourapp

import com.facebook.react.ReactPackage
import com.facebook.react.bridge.NativeModule
import com.facebook.react.bridge.ReactApplicationContext
import com.facebook.react.uimanager.ViewManager

class DeviceInfoPackage : ReactPackage {
    override fun createNativeModules(context: ReactApplicationContext):
        List<NativeModule> = listOf(DeviceInfoModule(context))

    override fun createViewManagers(context: ReactApplicationContext):
        List<ViewManager<*, *>> = emptyList()
}
```

```kotlin
// Step 3: Register in MainApplication.kt
override fun getPackages(): List<ReactPackage> =
    PackageList(this).packages.apply {
        add(DeviceInfoPackage())  // ← add your package
    }
```

---

### Q293. How do you create a Native Module on iOS (Swift)?

**Difficulty:** 🔴 Hard | **Frequency:** Very High | **Category:** Native Modules — iOS

**Answer:**
```swift
// Step 1: Create the Swift module
// ios/DeviceInfoModule.swift

import Foundation

@objc(DeviceInfoModule)
class DeviceInfoModule: NSObject {

  // requiresMainQueueSetup — return true if module uses UI APIs
  @objc static func requiresMainQueueSetup() -> Bool { return false }

  // Async method with Promise
  @objc func getBatteryLevel(
    _ resolve: @escaping RCTPromiseResolveBlock,
    rejecter reject: @escaping RCTPromiseRejectBlock
  ) {
    UIDevice.current.isBatteryMonitoringEnabled = true
    let level = UIDevice.current.batteryLevel
    if level < 0 {
      reject("BATTERY_ERROR", "Could not get battery level", nil)
    } else {
      resolve(Int(level * 100))
    }
  }

  // Sync method (use sparingly — blocks JS thread)
  @objc func getDeviceName() -> String {
    return UIDevice.current.name
  }

  // Returns a NSDictionary (JS object)
  @objc func getDeviceInfo(
    _ resolve: @escaping RCTPromiseResolveBlock,
    rejecter reject: @escaping RCTPromiseRejectBlock
  ) {
    let info: [String: Any] = [
      "name": UIDevice.current.name,
      "model": UIDevice.current.model,
      "systemVersion": UIDevice.current.systemVersion,
      "identifierForVendor": UIDevice.current.identifierForVendor?.uuidString ?? "",
    ]
    resolve(info)
  }
}
```

```objc
// Step 2: Expose to React Native via Objective-C bridge header
// ios/DeviceInfoModule.m

#import <React/RCTBridgeModule.h>

@interface RCT_EXTERN_MODULE(DeviceInfoModule, NSObject)

RCT_EXTERN_METHOD(
  getBatteryLevel:(RCTPromiseResolveBlock)resolve
  rejecter:(RCTPromiseRejectBlock)reject
)

RCT_EXTERN_METHOD(getDeviceName)

RCT_EXTERN_METHOD(
  getDeviceInfo:(RCTPromiseResolveBlock)resolve
  rejecter:(RCTPromiseRejectBlock)reject
)

@end
```

```swift
// Step 3: Bridging header (if not already present)
// ios/YourApp-Bridging-Header.h
#import <React/RCTBridgeModule.h>
#import <React/RCTEventEmitter.h>
```

---

### Q294. How do you send events from Native to JavaScript?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Native Events

**Answer:**
Use `RCTEventEmitter` (iOS) or `RCTDeviceEventEmitter` (Android) to push events from native code to JavaScript.

```kotlin
// Android — emit events from native to JS
// DeviceEventModule.kt

class SensorModule(private val reactContext: ReactApplicationContext)
    : ReactContextBaseJavaModule(reactContext) {

    override fun getName() = "SensorModule"

    // Send event to JS
    private fun sendEvent(eventName: String, params: WritableMap?) {
        reactContext
            .getJSModule(DeviceEventManagerModule.RCTDeviceEventEmitter::class.java)
            .emit(eventName, params)
    }

    @ReactMethod
    fun startListening() {
        // Start native sensor (e.g., accelerometer)
        sensorManager.registerListener(object : SensorEventListener {
            override fun onSensorChanged(event: SensorEvent) {
                val params = Arguments.createMap().apply {
                    putDouble("x", event.values[0].toDouble())
                    putDouble("y", event.values[1].toDouble())
                    putDouble("z", event.values[2].toDouble())
                    putDouble("timestamp", System.currentTimeMillis().toDouble())
                }
                sendEvent("onAccelerometerUpdate", params)
            }
            override fun onAccuracyChanged(sensor: Sensor, accuracy: Int) {}
        }, accelerometer, SensorManager.SENSOR_DELAY_UI)
    }

    @ReactMethod
    fun stopListening() { sensorManager.unregisterListener(listener) }

    // Required for addListener/removeListeners (new arch compatibility)
    @ReactMethod fun addListener(eventName: String) {}
    @ReactMethod fun removeListeners(count: Int) {}
}
```

```swift
// iOS — RCTEventEmitter
// SensorModule.swift

@objc(SensorModule)
class SensorModule: RCTEventEmitter {

    // Declare all event names
    override func supportedEvents() -> [String]! {
        return ["onAccelerometerUpdate", "onGyroUpdate"]
    }

    override static func requiresMainQueueSetup() -> Bool { return false }

    @objc func startListening() {
        motionManager.startAccelerometerUpdates(to: .main) { [weak self] data, error in
            guard let data = data else { return }
            self?.sendEvent(withName: "onAccelerometerUpdate", body: [
                "x": data.acceleration.x,
                "y": data.acceleration.y,
                "z": data.acceleration.z,
            ])
        }
    }

    @objc func stopListening() { motionManager.stopAccelerometerUpdates() }
}
```

```js
// JavaScript — listen for native events
import { NativeEventEmitter, NativeModules } from 'react-native';
const { SensorModule } = NativeModules;
const emitter = new NativeEventEmitter(SensorModule);

const useSensor = () => {
  const [reading, setReading] = useState({ x: 0, y: 0, z: 0 });

  useEffect(() => {
    SensorModule.startListening();

    const subscription = emitter.addListener('onAccelerometerUpdate', (data) => {
      setReading(data);
    });

    return () => {
      SensorModule.stopListening();
      subscription.remove(); // ALWAYS clean up
    };
  }, []);

  return reading;
};
```

---

### Q295. What are the data types you can pass between JS and Native?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Native Modules

**Answer:**
```
JavaScript Type    →    Android (Kotlin/Java)    →    iOS (Swift/ObjC)
─────────────────────────────────────────────────────────────────────
null/undefined          null                         nil
boolean                 Boolean                      Bool / NSNumber
number (int)            Int                          Int / NSInteger
number (double)         Double                       Double
string                  String                       String / NSString
array                   ReadableArray                NSArray
object                  ReadableMap                  NSDictionary
function/callback       Callback                     RCTResponseSenderBlock
Promise                 Promise                      RCTPromiseResolveBlock
```

```kotlin
// Android: Creating nested structures to return to JS
@ReactMethod
fun getEmployeeData(id: Int, promise: Promise) {
    val employee = Arguments.createMap().apply {
        putInt("id", id)
        putString("name", "Devesh Kumar")
        putDouble("salary", 850000.0)
        putBoolean("active", true)
        putNull("terminationDate")

        // Nested object
        val address = Arguments.createMap().apply {
            putString("city", "New Delhi")
            putString("state", "Delhi")
        }
        putMap("address", address)

        // Array
        val skills = Arguments.createArray().apply {
            pushString("React Native")
            pushString("Kotlin")
            pushString("Swift")
        }
        putArray("skills", skills)
    }
    promise.resolve(employee)
}
```

```swift
// iOS: Equivalent structure
@objc func getEmployeeData(_ id: Int,
  resolver resolve: @escaping RCTPromiseResolveBlock,
  rejecter reject: @escaping RCTPromiseRejectBlock) {
    let employee: [String: Any] = [
        "id": id,
        "name": "Devesh Kumar",
        "salary": 850000.0,
        "active": true,
        "terminationDate": NSNull(),
        "address": ["city": "New Delhi", "state": "Delhi"],
        "skills": ["React Native", "Kotlin", "Swift"],
    ]
    resolve(employee)
}
```

---

### Q296. What is the difference between Callbacks and Promises in Native Modules?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Native Modules

**Answer:**
```kotlin
// Callbacks — older API, fire once, can send multiple values
// Android
@ReactMethod
fun checkPermission(callback: Callback) {
    val granted = hasPermission()
    callback.invoke(null, granted)  // (error, result)
    // or on error: callback.invoke("PERMISSION_ERROR", null)
}

// Promises — modern API, cleaner, chainable in JS
@ReactMethod
fun checkPermissionAsync(promise: Promise) {
    try {
        val granted = hasPermission()
        promise.resolve(granted)
    } catch (e: SecurityException) {
        promise.reject("PERMISSION_ERROR", "Permission check failed", e)
    }
}

// Calling from JS:
// Callback style (messy):
DeviceModule.checkPermission((error, granted) => {
    if (error) { handleError(error); return; }
    setGranted(granted);
});

// Promise style (clean async/await):
try {
    const granted = await DeviceModule.checkPermissionAsync();
    setGranted(granted);
} catch (error) {
    handleError(error);
}
```

**Rules:**
- A Promise can only be resolved or rejected **once** — calling resolve/reject again crashes on Android
- A Callback can technically be called multiple times but should only be called once (use events for recurring data)
- For streaming/repeated data → use events (Q294)
- **Never** pass a callback across the bridge and store it in native — it creates a memory leak

---

### Q297. How do you handle threading in Native Modules?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Native Modules

**Answer:**
```kotlin
// Android — by default @ReactMethod runs on a background JS thread
// For UI work: must dispatch to main thread

@ReactMethod
fun showNativeDialog(message: String, promise: Promise) {
    // ❌ Crashes — can't update UI from background thread
    AlertDialog.Builder(currentActivity).show()

    // ✅ Run on main thread
    currentActivity?.runOnUiThread {
        AlertDialog.Builder(currentActivity)
            .setMessage(message)
            .setPositiveButton("OK") { _, _ -> promise.resolve(true) }
            .setNegativeButton("Cancel") { _, _ -> promise.resolve(false) }
            .show()
    }
}

// Override getMethodCallHandler to specify thread
override fun getName() = "MyModule"
// Default: all @ReactMethod calls are dispatched to the JS/module thread
// To run on main thread: @UiThread annotation (use sparingly)

@UiThread
@ReactMethod
fun updateNativeUI(color: String) {
    // Runs on UI thread automatically
    currentActivity?.window?.statusBarColor = Color.parseColor(color)
}
```

```swift
// iOS — default thread depends on requiresMainQueueSetup
// requiresMainQueueSetup = false → calls on background module queue
// requiresMainQueueSetup = true  → calls on main thread

// For UI work from background module:
@objc func presentCamera(_ resolve: @escaping RCTPromiseResolveBlock,
                          rejecter reject: @escaping RCTPromiseRejectBlock) {
    DispatchQueue.main.async { [weak self] in
        let picker = UIImagePickerController()
        picker.sourceType = .camera
        self?.bridge.viewControllerForPresentingModal?.present(picker, animated: true) {
            resolve(true)
        }
    }
}

// methodQueue — override to run all methods on a specific queue
@objc override var methodQueue: DispatchQueue {
    return DispatchQueue.main  // all methods run on main thread
}
// Use for modules that are primarily UI-related
```

---

### Q298. What is a TurboModule and how does it differ from a classic Native Module?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** New Architecture

**Answer:**
TurboModules are the New Architecture replacement for classic Native Modules. They use JSI (JavaScript Interface) instead of the Bridge.

| | Classic Native Module | TurboModule |
|--|----------------------|------------|
| Communication | Async Bridge (JSON) | Synchronous JSI |
| Loading | All modules at startup | Lazy (on first use) |
| Type safety | No (stringly-typed) | Yes (Codegen from TypeScript spec) |
| Performance | Bridge overhead | Near-zero overhead |
| Threading | Always async to JS | Can be synchronous |

```typescript
// Step 1: Define the TypeScript spec (Codegen input)
// js/NativeDeviceInfo.ts

import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  // Codegen generates native bindings from these signatures
  getDeviceName(): string;
  getBatteryLevel(): Promise<number>;
  getDeviceInfo(): Promise<{
    name: string;
    model: string;
    systemVersion: string;
  }>;
  addListener(eventName: string): void;
  removeListeners(count: number): void;
}

export default TurboModuleRegistry.getEnforcing<Spec>('DeviceInfo');
```

```kotlin
// Step 2: Implement on Android (Kotlin)
// Codegen generates NativeDeviceInfoSpec.kt — implement it
class DeviceInfoModule(context: ReactApplicationContext)
    : NativeDeviceInfoSpec(context) {

    override fun getName() = NAME

    override fun getDeviceName() = "${Build.MANUFACTURER} ${Build.MODEL}"

    override fun getBatteryLevel(promise: Promise) {
        // ... implementation
    }

    companion object {
        const val NAME = "DeviceInfo"
    }
}
```

```swift
// iOS: Codegen generates RCTNativeDeviceInfoSpec — implement it
@objc class DeviceInfoModule: NSObject, RCTNativeDeviceInfoSpec {
    func getDeviceName() -> String { UIDevice.current.name }
    func getBatteryLevel(_ resolve: @escaping RCTPromiseResolveBlock,
                         rejecter reject: @escaping RCTPromiseRejectBlock) { ... }
}
```

---

### Q299. What is Codegen in the New Architecture?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** New Architecture

**Answer:**
Codegen is the build-time code generation tool that reads TypeScript/Flow specifications and generates native type-safe C++/Kotlin/Swift interfaces. It eliminates runtime type errors in native modules.

```typescript
// You write this (TypeScript spec):
// NativeMyModule.ts
import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  multiply(a: number, b: number): number;
  fetchData(url: string): Promise<string>;
}

export default TurboModuleRegistry.getEnforcing<Spec>('MyModule');
```

```
Codegen at build time generates:
─────────────────────────────────────
iOS:      RCTNativeMyModuleSpec.h/.mm  (Objective-C protocol)
Android:  NativeMyModuleSpec.kt         (abstract Kotlin class)
C++:      NativeMyModule-generated.cpp  (JSI binding)
```

```kotlin
// You implement the generated spec:
class MyModule(context: ReactApplicationContext) : NativeMyModuleSpec(context) {
    override fun getName() = "MyModule"

    // Type-checked at compile time — must match spec
    override fun multiply(a: Double, b: Double): Double = a * b

    override fun fetchData(url: String, promise: Promise) {
        // ... implementation
    }
}
```

**Benefits:**
- Compile-time type safety — wrong types fail at build, not runtime
- No `NativeModules.MyModule.someMethod()` stringly-typed calls
- Better IDE support — autocomplete on native methods from TS types
- Faster builds — generated code is optimised

---

### Q300. How do you handle errors in Native Modules?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Native Modules

**Answer:**
```kotlin
// Android — error handling patterns

// 1. Promise.reject with error code, message, and throwable
@ReactMethod
fun readSecureStorage(key: String, promise: Promise) {
    try {
        val value = secureStorage.get(key)
            ?: return promise.reject("KEY_NOT_FOUND", "No value for key: $key")
        promise.resolve(value)
    } catch (e: SecurityException) {
        promise.reject("SECURITY_ERROR", "Access denied: ${e.message}", e)
    } catch (e: Exception) {
        promise.reject("UNKNOWN_ERROR", e.message, e)
    }
}

// 2. Consistent error codes (define as constants)
companion object {
    const val E_KEY_NOT_FOUND = "KEY_NOT_FOUND"
    const val E_SECURITY = "SECURITY_ERROR"
    const val E_NETWORK = "NETWORK_ERROR"
}
```

```swift
// iOS — error handling
@objc func readSecureStorage(_ key: String,
    resolver resolve: @escaping RCTPromiseResolveBlock,
    rejecter reject: @escaping RCTPromiseRejectBlock) {

    guard !key.isEmpty else {
        reject("INVALID_KEY", "Key cannot be empty", nil)
        return
    }

    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrAccount as String: key,
        kSecReturnData as String: true,
    ]

    var result: AnyObject?
    let status = SecItemCopyMatching(query as CFDictionary, &result)

    switch status {
    case errSecSuccess:
        guard let data = result as? Data,
              let value = String(data: data, encoding: .utf8) else {
            reject("DECODE_ERROR", "Failed to decode stored value", nil)
            return
        }
        resolve(value)
    case errSecItemNotFound:
        reject("KEY_NOT_FOUND", "No value for key: \(key)", nil)
    default:
        reject("KEYCHAIN_ERROR", "Keychain error: \(status)", nil)
    }
}
```

```js
// JavaScript — catching native errors
const readSecure = async (key) => {
    try {
        const value = await SecureStorageModule.readSecureStorage(key);
        return value;
    } catch (error) {
        switch (error.code) {
            case 'KEY_NOT_FOUND':
                return null; // expected — key doesn't exist
            case 'SECURITY_ERROR':
                throw new Error('Biometric authentication required');
            default:
                Sentry.captureException(error);
                throw error;
        }
    }
};
```

---

### Q301. How do you write a Native Module for biometric authentication?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Native Modules — Real World

**Answer:**
```kotlin
// Android — BiometricModule.kt
package com.yourapp

import androidx.biometric.BiometricManager
import androidx.biometric.BiometricPrompt
import androidx.core.content.ContextCompat
import com.facebook.react.bridge.*

class BiometricModule(reactContext: ReactApplicationContext)
    : ReactContextBaseJavaModule(reactContext) {

    override fun getName() = "BiometricModule"

    @ReactMethod
    fun canAuthenticate(promise: Promise) {
        val manager = BiometricManager.from(reactApplicationContext)
        val result = manager.canAuthenticate(
            BiometricManager.Authenticators.BIOMETRIC_STRONG or
            BiometricManager.Authenticators.DEVICE_CREDENTIAL
        )
        when (result) {
            BiometricManager.BIOMETRIC_SUCCESS -> promise.resolve("BIOMETRIC_AVAILABLE")
            BiometricManager.BIOMETRIC_ERROR_NO_HARDWARE -> promise.resolve("NO_HARDWARE")
            BiometricManager.BIOMETRIC_ERROR_HW_UNAVAILABLE -> promise.resolve("HW_UNAVAILABLE")
            BiometricManager.BIOMETRIC_ERROR_NONE_ENROLLED -> promise.resolve("NONE_ENROLLED")
            else -> promise.resolve("UNKNOWN")
        }
    }

    @ReactMethod
    fun authenticate(title: String, subtitle: String, promise: Promise) {
        val activity = currentActivity ?: return promise.reject("NO_ACTIVITY", "No activity")
        val executor = ContextCompat.getMainExecutor(reactApplicationContext)

        activity.runOnUiThread {
            val callback = object : BiometricPrompt.AuthenticationCallback() {
                override fun onAuthenticationSucceeded(result: BiometricPrompt.AuthenticationResult) {
                    promise.resolve(true)
                }
                override fun onAuthenticationError(errorCode: Int, errString: CharSequence) {
                    when (errorCode) {
                        BiometricPrompt.ERROR_USER_CANCELED,
                        BiometricPrompt.ERROR_NEGATIVE_BUTTON -> promise.resolve(false)
                        else -> promise.reject("AUTH_ERROR", errString.toString())
                    }
                }
                override fun onAuthenticationFailed() {
                    // Called when biometric doesn't match — NOT a rejection
                    // The prompt continues showing
                }
            }

            val promptInfo = BiometricPrompt.PromptInfo.Builder()
                .setTitle(title)
                .setSubtitle(subtitle)
                .setAllowedAuthenticators(
                    BiometricManager.Authenticators.BIOMETRIC_STRONG or
                    BiometricManager.Authenticators.DEVICE_CREDENTIAL
                )
                .build()

            BiometricPrompt(activity as androidx.fragment.app.FragmentActivity, executor, callback)
                .authenticate(promptInfo)
        }
    }
}
```

```swift
// iOS — BiometricModule.swift
import LocalAuthentication

@objc(BiometricModule)
class BiometricModule: NSObject {

    @objc static func requiresMainQueueSetup() -> Bool { false }

    @objc func canAuthenticate(_ resolve: @escaping RCTPromiseResolveBlock,
                                rejecter reject: @escaping RCTPromiseRejectBlock) {
        let context = LAContext()
        var error: NSError?
        let canEval = context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics,
                                                error: &error)
        if canEval {
            resolve(context.biometryType == .faceID ? "FACE_ID" : "TOUCH_ID")
        } else {
            resolve("NONE")
        }
    }

    @objc func authenticate(_ reason: String,
                             resolver resolve: @escaping RCTPromiseResolveBlock,
                             rejecter reject: @escaping RCTPromiseRejectBlock) {
        let context = LAContext()
        context.evaluatePolicy(
            .deviceOwnerAuthenticationWithBiometrics,
            localizedReason: reason
        ) { success, error in
            if success {
                resolve(true)
            } else if let err = error as? LAError {
                switch err.code {
                case .userCancel, .userFallback, .systemCancel:
                    resolve(false)
                default:
                    reject("AUTH_ERROR", err.localizedDescription, err)
                }
            }
        }
    }
}
```

```js
// JavaScript wrapper
import { NativeModules, Platform } from 'react-native';
const { BiometricModule } = NativeModules;

export const Biometric = {
    canAuthenticate: () => BiometricModule.canAuthenticate(),
    authenticate: (reason = 'Verify your identity') =>
        BiometricModule.authenticate(reason, Platform.OS === 'android' ? reason : undefined),
};
```

---

### Q302. How do you implement a Native View Component?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Native View Managers

**Answer:**
A Native View Component exposes a native UI widget as a React component — like `<MapView>`, `<VideoPlayer>`, or `<SignaturePad>`.

```kotlin
// Android — Custom NativeView example: a native SignatureView

// 1. Custom Android View
class SignatureView(context: Context) : View(context) {
    private val path = Path()
    private val paint = Paint().apply {
        color = Color.BLACK; strokeWidth = 4f; style = Paint.Style.STROKE
    }
    var onSignatureChange: ((String) -> Unit)? = null

    override fun onTouchEvent(event: MotionEvent): Boolean {
        when (event.action) {
            MotionEvent.ACTION_DOWN -> path.moveTo(event.x, event.y)
            MotionEvent.ACTION_MOVE -> {
                path.lineTo(event.x, event.y)
                onSignatureChange?.invoke(exportSignatureAsBase64())
                invalidate()
            }
        }
        return true
    }
    override fun onDraw(canvas: Canvas) { canvas.drawPath(path, paint) }
    fun clear() { path.reset(); invalidate() }
    private fun exportSignatureAsBase64(): String { /* ... */ return "" }
}

// 2. ViewManager
class SignatureViewManager : SimpleViewManager<SignatureView>() {
    override fun getName() = "SignatureView"
    override fun createViewInstance(context: ThemedReactContext) = SignatureView(context)

    // Expose prop to JS: strokeColor
    @ReactProp(name = "strokeColor")
    fun setStrokeColor(view: SignatureView, color: Int) {
        view.paint.color = color
    }

    // Fire JS event: onSignatureChange
    override fun getExportedCustomBubblingEventTypeConstants() = mapOf(
        "onSignatureChange" to mapOf(
            "phasedRegistrationNames" to mapOf(
                "bubbled" to "onSignatureChange",
                "captured" to "onSignatureChangeCapture"
            )
        )
    )

    // Expose commands (callable from JS ref)
    override fun getCommandsMap() = mapOf("clear" to 0)

    override fun receiveCommand(view: SignatureView, commandId: String, args: ReadableArray?) {
        if (commandId == "clear") view.clear()
    }
}
```

```js
// JavaScript wrapper
import { requireNativeComponent, UIManager, findNodeHandle } from 'react-native';

const NativeSignatureView = requireNativeComponent('SignatureView');

const SignaturePad = React.forwardRef(({ strokeColor = '#000', onSignatureChange, style }, ref) => {
    const nativeRef = useRef(null);

    // Expose clear() to parent via ref
    useImperativeHandle(ref, () => ({
        clear: () => {
            UIManager.dispatchViewManagerCommand(
                findNodeHandle(nativeRef.current),
                UIManager.SignatureView.Commands.clear,
                []
            );
        },
    }));

    return (
        <NativeSignatureView
            ref={nativeRef}
            strokeColor={processColor(strokeColor)} // processColor converts JS color to native int
            onSignatureChange={(e) => onSignatureChange?.(e.nativeEvent.signature)}
            style={style}
        />
    );
});

// Usage
const sig = useRef(null);
<SignaturePad
    ref={sig}
    strokeColor="#1a1a1a"
    onSignatureChange={(base64) => console.log('Signed:', base64)}
    style={{ height: 200, backgroundColor: '#f5f5f5' }}
/>
<Button title="Clear" onPress={() => sig.current?.clear()} />
```

---

### Q303. How do you write a native module that uses the device camera?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Native Modules — Camera

**Answer:**
```swift
// iOS — CameraModule.swift
import UIKit
import AVFoundation

@objc(CameraModule)
class CameraModule: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {

    var resolve: RCTPromiseResolveBlock?
    var reject: RCTPromiseRejectBlock?

    @objc static func requiresMainQueueSetup() -> Bool { true }

    @objc func openCamera(_ options: NSDictionary,
                           resolver resolve: @escaping RCTPromiseResolveBlock,
                           rejecter reject: @escaping RCTPromiseRejectBlock) {

        self.resolve = resolve
        self.reject = reject

        DispatchQueue.main.async { [weak self] in
            guard AVCaptureDevice.authorizationStatus(for: .video) == .authorized ||
                  AVCaptureDevice.authorizationStatus(for: .video) == .notDetermined else {
                reject("PERMISSION_DENIED", "Camera permission denied", nil)
                return
            }

            let picker = UIImagePickerController()
            picker.sourceType = .camera
            picker.allowsEditing = options["allowsEditing"] as? Bool ?? false
            picker.delegate = self

            guard let vc = RCTSharedApplication()?.keyWindow?.rootViewController else {
                reject("NO_VC", "Could not find view controller", nil)
                return
            }
            vc.present(picker, animated: true)
        }
    }

    func imagePickerController(_ picker: UIImagePickerController,
                                didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey: Any]) {
        picker.dismiss(animated: true)
        guard let image = info[.editedImage] as? UIImage ?? info[.originalImage] as? UIImage else {
            reject?("NO_IMAGE", "Could not get image", nil)
            return
        }
        guard let data = image.jpegData(compressionQuality: 0.8) else {
            reject?("ENCODE_ERROR", "Could not encode image", nil)
            return
        }
        let base64 = data.base64EncodedString()
        resolve?(["base64": base64, "width": image.size.width, "height": image.size.height])
    }

    func imagePickerControllerDidCancel(_ picker: UIImagePickerController) {
        picker.dismiss(animated: true)
        resolve?(NSNull()) // user cancelled — resolve with null
    }
}
```

---

### Q304. How do you test Native Modules?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Testing + Native Modules

**Answer:**
```js
// 1. Mock the native module in Jest
// __mocks__/react-native.js (or in your Jest setup file)
jest.mock('react-native', () => {
    const RN = jest.requireActual('react-native');
    RN.NativeModules.BiometricModule = {
        canAuthenticate: jest.fn().mockResolvedValue('FACE_ID'),
        authenticate: jest.fn().mockResolvedValue(true),
    };
    return RN;
});

// 2. Test your JS wrapper
import { Biometric } from '../src/modules/Biometric';
import { NativeModules } from 'react-native';

describe('Biometric', () => {
    it('returns face id type', async () => {
        const type = await Biometric.canAuthenticate();
        expect(type).toBe('FACE_ID');
        expect(NativeModules.BiometricModule.canAuthenticate).toHaveBeenCalledTimes(1);
    });

    it('resolves false on user cancel', async () => {
        NativeModules.BiometricModule.authenticate.mockResolvedValueOnce(false);
        const result = await Biometric.authenticate('Confirm payment');
        expect(result).toBe(false);
    });

    it('rejects on auth error', async () => {
        NativeModules.BiometricModule.authenticate.mockRejectedValueOnce({
            code: 'AUTH_ERROR',
            message: 'Biometric sensor unavailable',
        });
        await expect(Biometric.authenticate()).rejects.toMatchObject({
            code: 'AUTH_ERROR',
        });
    });
});

// 3. Test native code (Android — JUnit)
// BiometricModuleTest.kt
@RunWith(RobolectricTestRunner::class)
class BiometricModuleTest {
    private lateinit var module: BiometricModule
    private val context = ApplicationProvider.getApplicationContext<Application>()

    @Before
    fun setUp() {
        val reactContext = mock(ReactApplicationContext::class.java)
        module = BiometricModule(reactContext)
    }

    @Test
    fun `getName returns BiometricModule`() {
        assertEquals("BiometricModule", module.name)
    }
}
```

---

### Q305. How do you create a native module that persists data to the Keychain (iOS) / Keystore (Android)?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Security + Native Modules

**Answer:**
```swift
// iOS — Keychain module
@objc(KeychainModule)
class KeychainModule: NSObject {

    @objc static func requiresMainQueueSetup() -> Bool { false }

    @objc func setItem(_ key: String,
                        value: String,
                        resolver resolve: @escaping RCTPromiseResolveBlock,
                        rejecter reject: @escaping RCTPromiseRejectBlock) {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecValueData as String: value.data(using: .utf8)!,
            kSecAttrAccessible as String: kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
        ]
        SecItemDelete(query as CFDictionary) // delete if exists
        let status = SecItemAdd(query as CFDictionary, nil)
        status == errSecSuccess ? resolve(true) : reject("KEYCHAIN_ERROR", "Save failed: \(status)", nil)
    }

    @objc func getItem(_ key: String,
                        resolver resolve: @escaping RCTPromiseResolveBlock,
                        rejecter reject: @escaping RCTPromiseRejectBlock) {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne,
        ]
        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)

        switch status {
        case errSecSuccess:
            guard let data = result as? Data, let str = String(data: data, encoding: .utf8) else {
                reject("DECODE_ERROR", "Failed to decode value", nil); return
            }
            resolve(str)
        case errSecItemNotFound:
            resolve(nil)
        default:
            reject("KEYCHAIN_ERROR", "Read failed: \(status)", nil)
        }
    }

    @objc func removeItem(_ key: String,
                           resolver resolve: @escaping RCTPromiseResolveBlock,
                           rejecter reject: @escaping RCTPromiseRejectBlock) {
        let query: [String: Any] = [kSecClass as String: kSecClassGenericPassword,
                                     kSecAttrAccount as String: key]
        let status = SecItemDelete(query as CFDictionary)
        resolve(status == errSecSuccess || status == errSecItemNotFound)
    }
}
```

```kotlin
// Android — EncryptedSharedPreferences (API 23+)
import androidx.security.crypto.EncryptedSharedPreferences
import androidx.security.crypto.MasterKeys

class KeystoreModule(reactContext: ReactApplicationContext)
    : ReactContextBaseJavaModule(reactContext) {

    override fun getName() = "KeychainModule"

    private val sharedPreferences by lazy {
        EncryptedSharedPreferences.create(
            "secure_prefs",
            MasterKeys.getOrCreate(MasterKeys.AES256_GCM_SPEC),
            reactApplicationContext,
            EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
            EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
        )
    }

    @ReactMethod
    fun setItem(key: String, value: String, promise: Promise) {
        try {
            sharedPreferences.edit().putString(key, value).apply()
            promise.resolve(true)
        } catch (e: Exception) { promise.reject("KEYSTORE_ERROR", e.message, e) }
    }

    @ReactMethod
    fun getItem(key: String, promise: Promise) {
        try {
            promise.resolve(sharedPreferences.getString(key, null))
        } catch (e: Exception) { promise.reject("KEYSTORE_ERROR", e.message, e) }
    }

    @ReactMethod
    fun removeItem(key: String, promise: Promise) {
        try {
            sharedPreferences.edit().remove(key).apply()
            promise.resolve(true)
        } catch (e: Exception) { promise.reject("KEYSTORE_ERROR", e.message, e) }
    }
}
```

---

### Q306. How do you handle permissions in a native module?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Permissions + Native Modules

**Answer:**
```kotlin
// Android — requesting permissions from within a native module

class CameraPermissionModule(reactContext: ReactApplicationContext)
    : ReactContextBaseJavaModule(reactContext), ActivityEventListener {

    private var permissionPromise: Promise? = null
    private val CAMERA_PERMISSION_CODE = 1001

    init { reactContext.addActivityEventListener(this) }

    override fun getName() = "CameraPermissionModule"

    @ReactMethod
    fun requestCameraPermission(promise: Promise) {
        val activity = currentActivity
            ?: return promise.reject("NO_ACTIVITY", "No activity found")

        if (ContextCompat.checkSelfPermission(activity, Manifest.permission.CAMERA)
            == PackageManager.PERMISSION_GRANTED) {
            promise.resolve("granted")
            return
        }

        // Store promise — will be resolved in onActivityResult/onRequestPermissionsResult
        permissionPromise = promise
        ActivityCompat.requestPermissions(
            activity,
            arrayOf(Manifest.permission.CAMERA),
            CAMERA_PERMISSION_CODE
        )
    }

    // Called when user responds to permission dialog
    override fun onRequestPermissionsResult(
        requestCode: Int, permissions: Array<String>, grantResults: IntArray
    ) {
        if (requestCode == CAMERA_PERMISSION_CODE) {
            if (grantResults.isNotEmpty() && grantResults[0] == PackageManager.PERMISSION_GRANTED) {
                permissionPromise?.resolve("granted")
            } else {
                val neverAskAgain = !ActivityCompat.shouldShowRequestPermissionRationale(
                    currentActivity!!, Manifest.permission.CAMERA
                )
                permissionPromise?.resolve(if (neverAskAgain) "never_ask_again" else "denied")
            }
            permissionPromise = null
        }
    }

    override fun onActivityResult(activity: Activity?, requestCode: Int, resultCode: Int, data: Intent?) {}
    override fun onNewIntent(intent: Intent?) {}
}
```

```swift
// iOS — requesting permissions from Swift module
@objc func requestCameraPermission(_ resolve: @escaping RCTPromiseResolveBlock,
                                    rejecter reject: @escaping RCTPromiseRejectBlock) {
    let status = AVCaptureDevice.authorizationStatus(for: .video)

    switch status {
    case .authorized:
        resolve("granted")
    case .notDetermined:
        AVCaptureDevice.requestAccess(for: .video) { granted in
            resolve(granted ? "granted" : "denied")
        }
    case .denied:
        resolve("denied")
    case .restricted:
        resolve("restricted")
    @unknown default:
        resolve("unknown")
    }
}
```

---

### Q307. How do you implement a native module for network requests with custom certificate pinning?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Security + Networking

**Answer:**
```kotlin
// Android — OkHttp with certificate pinning
class SecureNetworkModule(reactContext: ReactApplicationContext)
    : ReactContextBaseJavaModule(reactContext) {

    override fun getName() = "SecureNetworkModule"

    private val client: OkHttpClient by lazy {
        val certificatePinner = CertificatePinner.Builder()
            .add("api.yourbank.com", "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=")
            .add("api.yourbank.com", "sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=") // backup
            .build()

        OkHttpClient.Builder()
            .certificatePinner(certificatePinner)
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .build()
    }

    @ReactMethod
    fun securePost(url: String, body: String, headers: ReadableMap, promise: Promise) {
        val requestBody = body.toRequestBody("application/json".toMediaType())

        val requestBuilder = Request.Builder().url(url).post(requestBody)
        headers.entryIterator.forEach { entry ->
            requestBuilder.addHeader(entry.key, entry.value as String)
        }

        Thread {
            try {
                val response = client.newCall(requestBuilder.build()).execute()
                val responseBody = response.body?.string() ?: ""
                if (response.isSuccessful) {
                    promise.resolve(responseBody)
                } else {
                    promise.reject("HTTP_ERROR", "Status: ${response.code}", null)
                }
            } catch (e: javax.net.ssl.SSLPeerUnverifiedException) {
                promise.reject("PINNING_FAILED", "Certificate pinning verification failed", e)
            } catch (e: Exception) {
                promise.reject("NETWORK_ERROR", e.message, e)
            }
        }.start()
    }
}
```

---

### Q308. What is `requiresMainQueueSetup` in iOS native modules?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** iOS Native Modules

**Answer:**
`requiresMainQueueSetup` tells React Native whether the module needs to be initialised on the main (UI) thread. Introduced to fix warnings and improve performance.

```swift
// Return true if:
// - Module accesses UIKit APIs during init
// - Module registers observers for UI events during init
// - Module accesses window, rootViewController, etc. at startup

@objc(MyUIModule)
class MyUIModule: NSObject {
    @objc static func requiresMainQueueSetup() -> Bool {
        return true  // needs main thread for UIKit
    }
    // ...
}

// Return false if:
// - Module is purely data/computation (no UIKit during init)
// - Module only accesses UI APIs when methods are called (dispatch manually)

@objc(DataProcessingModule)
class DataProcessingModule: NSObject {
    @objc static func requiresMainQueueSetup() -> Bool {
        return false  // pure data — no main thread needed at init
    }
    // ...
}

// Best practice: return false by default
// Only return true when you get the warning:
// "Module MyModule requires main queue setup since it overrides..."
// Or when you see crashes related to UI access off main thread

// If false but method needs main thread — dispatch manually:
@objc func showAlert(_ message: String, resolver resolve: @escaping RCTPromiseResolveBlock, ...) {
    DispatchQueue.main.async {
        // UIKit code here
    }
}
```

---

### Q309. How do you create a native module that communicates with Bluetooth?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Native Modules — BLE

**Answer:**
```kotlin
// Android — BLE Scanner Module (simplified)
import android.bluetooth.*
import android.bluetooth.le.*

class BLEModule(reactContext: ReactApplicationContext)
    : ReactContextBaseJavaModule(reactContext) {

    override fun getName() = "BLEModule"

    private val bluetoothAdapter: BluetoothAdapter? by lazy {
        (reactApplicationContext.getSystemService(Context.BLUETOOTH_SERVICE) as? BluetoothManager)
            ?.adapter
    }

    private val discoveredDevices = mutableMapOf<String, BluetoothDevice>()

    @ReactMethod
    fun startScan(promise: Promise) {
        val adapter = bluetoothAdapter
        if (adapter == null || !adapter.isEnabled) {
            promise.reject("BT_DISABLED", "Bluetooth is disabled")
            return
        }

        val scanner = adapter.bluetoothLeScanner
        val callback = object : ScanCallback() {
            override fun onScanResult(callbackType: Int, result: ScanResult) {
                val device = result.device
                discoveredDevices[device.address] = device

                val params = Arguments.createMap().apply {
                    putString("id", device.address)
                    putString("name", device.name ?: "Unknown")
                    putInt("rssi", result.rssi)
                }
                sendEvent("onDeviceDiscovered", params)
            }
            override fun onScanFailed(errorCode: Int) {
                promise.reject("SCAN_FAILED", "Scan failed with code: $errorCode")
            }
        }

        scanner.startScan(callback)
        promise.resolve(true)
    }

    @ReactMethod
    fun connectToDevice(deviceId: String, promise: Promise) {
        val device = discoveredDevices[deviceId]
            ?: return promise.reject("DEVICE_NOT_FOUND", "Device not found: $deviceId")

        device.connectGatt(reactApplicationContext, false, object : BluetoothGattCallback() {
            override fun onConnectionStateChange(gatt: BluetoothGatt, status: Int, newState: Int) {
                when (newState) {
                    BluetoothProfile.STATE_CONNECTED -> {
                        gatt.discoverServices()
                        promise.resolve(mapOf("id" to deviceId, "connected" to true))
                    }
                    BluetoothProfile.STATE_DISCONNECTED -> {
                        promise.resolve(mapOf("id" to deviceId, "connected" to false))
                    }
                }
            }
        })
    }

    private fun sendEvent(name: String, params: WritableMap) {
        reactApplicationContext
            .getJSModule(DeviceEventManagerModule.RCTDeviceEventEmitter::class.java)
            .emit(name, params)
    }

    @ReactMethod fun addListener(name: String) {}
    @ReactMethod fun removeListeners(count: Int) {}
}
```

---

### Q310. How do you implement in-app purchase via a native module?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Native Modules — IAP

**Answer:**
```kotlin
// Android — Google Play Billing (simplified)
import com.android.billingclient.api.*

class IAPModule(reactContext: ReactApplicationContext)
    : ReactContextBaseJavaModule(reactContext), PurchasesUpdatedListener {

    override fun getName() = "IAPModule"
    private var billingClient: BillingClient? = null
    private var purchasePromise: Promise? = null

    @ReactMethod
    fun initBilling(promise: Promise) {
        billingClient = BillingClient.newBuilder(reactApplicationContext)
            .setListener(this)
            .enablePendingPurchases()
            .build()

        billingClient?.startConnection(object : BillingClientStateListener {
            override fun onBillingSetupFinished(result: BillingResult) {
                if (result.responseCode == BillingClient.BillingResponseCode.OK) {
                    promise.resolve(true)
                } else {
                    promise.reject("BILLING_INIT_ERROR", result.debugMessage)
                }
            }
            override fun onBillingServiceDisconnected() {
                sendEvent("onBillingDisconnected", null)
            }
        })
    }

    @ReactMethod
    fun purchaseProduct(productId: String, promise: Promise) {
        purchasePromise = promise
        val queryParams = QueryProductDetailsParams.newBuilder()
            .setProductList(listOf(
                QueryProductDetailsParams.Product.newBuilder()
                    .setProductId(productId)
                    .setProductType(BillingClient.ProductType.INAPP)
                    .build()
            ))
            .build()

        billingClient?.queryProductDetailsAsync(queryParams) { result, productList ->
            if (result.responseCode != BillingClient.BillingResponseCode.OK || productList.isEmpty()) {
                promise.reject("PRODUCT_NOT_FOUND", "Product $productId not found")
                purchasePromise = null
                return@queryProductDetailsAsync
            }

            val flowParams = BillingFlowParams.newBuilder()
                .setProductDetailsParamsList(listOf(
                    BillingFlowParams.ProductDetailsParams.newBuilder()
                        .setProductDetails(productList[0])
                        .build()
                ))
                .build()

            currentActivity?.let {
                billingClient?.launchBillingFlow(it, flowParams)
            }
        }
    }

    override fun onPurchasesUpdated(result: BillingResult, purchases: List<Purchase>?) {
        when (result.responseCode) {
            BillingClient.BillingResponseCode.OK -> {
                purchases?.firstOrNull()?.let { purchase ->
                    val params = Arguments.createMap().apply {
                        putString("purchaseToken", purchase.purchaseToken)
                        putString("orderId", purchase.orderId)
                        putString("productId", purchase.products.firstOrNull())
                    }
                    purchasePromise?.resolve(params)
                }
            }
            BillingClient.BillingResponseCode.USER_CANCELED ->
                purchasePromise?.resolve(null)
            else ->
                purchasePromise?.reject("PURCHASE_ERROR", result.debugMessage)
        }
        purchasePromise = null
    }

    @ReactMethod fun addListener(name: String) {}
    @ReactMethod fun removeListeners(count: Int) {}
}
```

---

### Q311. How do you write a module that accesses device contacts?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Native Modules

**Answer:**
```kotlin
// Android — ContactsModule.kt
class ContactsModule(reactContext: ReactApplicationContext)
    : ReactContextBaseJavaModule(reactContext) {

    override fun getName() = "ContactsModule"

    @ReactMethod
    fun getContacts(promise: Promise) {
        val cursor = reactApplicationContext.contentResolver.query(
            ContactsContract.CommonDataKinds.Phone.CONTENT_URI,
            arrayOf(
                ContactsContract.CommonDataKinds.Phone.DISPLAY_NAME,
                ContactsContract.CommonDataKinds.Phone.NUMBER,
                ContactsContract.CommonDataKinds.Phone.CONTACT_ID,
            ),
            null, null,
            ContactsContract.CommonDataKinds.Phone.DISPLAY_NAME + " ASC"
        ) ?: return promise.reject("CURSOR_ERROR", "Failed to query contacts")

        val contacts = Arguments.createArray()
        val seen = mutableSetOf<String>()

        cursor.use {
            val nameIdx = it.getColumnIndex(ContactsContract.CommonDataKinds.Phone.DISPLAY_NAME)
            val numIdx = it.getColumnIndex(ContactsContract.CommonDataKinds.Phone.NUMBER)
            val idIdx = it.getColumnIndex(ContactsContract.CommonDataKinds.Phone.CONTACT_ID)

            while (it.moveToNext()) {
                val id = it.getString(idIdx)
                if (seen.contains(id)) continue
                seen.add(id)

                val contact = Arguments.createMap().apply {
                    putString("id", id)
                    putString("name", it.getString(nameIdx) ?: "")
                    putString("phone", it.getString(numIdx) ?: "")
                }
                contacts.pushMap(contact)
            }
        }
        promise.resolve(contacts)
    }

    @ReactMethod
    fun searchContacts(query: String, promise: Promise) {
        val cursor = reactApplicationContext.contentResolver.query(
            ContactsContract.CommonDataKinds.Phone.CONTENT_URI,
            arrayOf(
                ContactsContract.CommonDataKinds.Phone.DISPLAY_NAME,
                ContactsContract.CommonDataKinds.Phone.NUMBER,
            ),
            "${ContactsContract.CommonDataKinds.Phone.DISPLAY_NAME} LIKE ?",
            arrayOf("%$query%"),
            ContactsContract.CommonDataKinds.Phone.DISPLAY_NAME + " ASC"
        ) ?: return promise.reject("CURSOR_ERROR", "Search failed")

        val contacts = Arguments.createArray()
        cursor.use {
            val nameIdx = it.getColumnIndex(ContactsContract.CommonDataKinds.Phone.DISPLAY_NAME)
            val numIdx = it.getColumnIndex(ContactsContract.CommonDataKinds.Phone.NUMBER)
            while (it.moveToNext()) {
                val contact = Arguments.createMap().apply {
                    putString("name", it.getString(nameIdx) ?: "")
                    putString("phone", it.getString(numIdx) ?: "")
                }
                contacts.pushMap(contact)
            }
        }
        promise.resolve(contacts)
    }
}
```

---

### Q312. How do you share data between native modules and React Native state?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Native Modules

**Answer:**
```js
// Pattern 1: Events (native → JS push) — most common
import { NativeEventEmitter, NativeModules } from 'react-native';
const { LocationModule } = NativeModules;
const locationEmitter = new NativeEventEmitter(LocationModule);

const useNativeLocation = () => {
    const [location, setLocation] = useState(null);
    const [error, setError] = useState(null);

    useEffect(() => {
        LocationModule.startTracking();

        const locSub = locationEmitter.addListener('onLocationUpdate', setLocation);
        const errSub = locationEmitter.addListener('onLocationError', (e) => setError(e.message));

        return () => {
            LocationModule.stopTracking();
            locSub.remove();
            errSub.remove();
        };
    }, []);

    return { location, error };
};

// Pattern 2: Promises (JS → native → JS response)
const useBiometric = () => {
    const [status, setStatus] = useState('idle');

    const authenticate = useCallback(async () => {
        setStatus('authenticating');
        try {
            const success = await NativeModules.BiometricModule.authenticate('Confirm identity');
            setStatus(success ? 'authenticated' : 'cancelled');
            return success;
        } catch (error) {
            setStatus('error');
            throw error;
        }
    }, []);

    return { status, authenticate };
};

// Pattern 3: Zustand store updated from native events
import { create } from 'zustand';

const useDeviceStore = create((set) => ({
    batteryLevel: 100,
    isCharging: false,
    updateBattery: (level, charging) => set({ batteryLevel: level, isCharging: charging }),
}));

// Setup once at app root
const setupNativeListeners = () => {
    const emitter = new NativeEventEmitter(NativeModules.BatteryModule);
    emitter.addListener('onBatteryUpdate', ({ level, isCharging }) => {
        useDeviceStore.getState().updateBattery(level, isCharging);
    });
};
```

---

### Q313. What is the Native Module Lazy Loading in New Architecture?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** New Architecture

**Answer:**
In the old architecture, **all** native modules are loaded eagerly at app startup regardless of whether they're ever used. TurboModules in the New Architecture are loaded **lazily** — only when first accessed.

```typescript
// Old Architecture — all modules loaded on startup
// Even if you never use BiometricModule, it's loaded, initialised, and
// its getConstants() is called on every app startup

// New Architecture (TurboModules) — lazy loading
import { TurboModuleRegistry } from 'react-native';

// ❌ Old API — always eagerly loaded
const { BiometricModule } = NativeModules;

// ✅ New API — loaded only when this line executes (first use)
const BiometricModule = TurboModuleRegistry.getEnforcing<Spec>('BiometricModule');
// OR: null if not found
const BiometricModule = TurboModuleRegistry.get<Spec>('BiometricModule');

// Benefits:
// - Faster startup (fewer modules to initialise)
// - Lower memory (unused modules not in memory)
// - Platform-specific modules only on their platform

// Platform-specific module handling:
const module = Platform.select({
    ios: () => TurboModuleRegistry.getEnforcing('IOSOnlyModule'),
    android: () => TurboModuleRegistry.getEnforcing('AndroidOnlyModule'),
})?.();

// Conditional loading
const getModule = () => {
    if (!isTurboModuleEnabled()) {
        return NativeModules.LegacyModule; // fallback
    }
    return TurboModuleRegistry.get('LegacyModule');
};
```

---

### Q314. How do you implement a file picker native module?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Native Modules

**Answer:**
```kotlin
// Android — FilePickerModule.kt
class FilePickerModule(reactContext: ReactApplicationContext)
    : ReactContextBaseJavaModule(reactContext), ActivityEventListener {

    override fun getName() = "FilePicker"
    private val PICK_FILE_REQUEST = 1234
    private var filePromise: Promise? = null

    init { reactContext.addActivityEventListener(this) }

    @ReactMethod
    fun pickFile(mimeTypes: ReadableArray, promise: Promise) {
        val activity = currentActivity ?: return promise.reject("NO_ACTIVITY", "No activity")
        filePromise = promise

        val intent = Intent(Intent.ACTION_GET_CONTENT).apply {
            type = "*/*"
            if (mimeTypes.size() > 0) {
                val types = Array(mimeTypes.size()) { mimeTypes.getString(it) ?: "*/*" }
                putExtra(Intent.EXTRA_MIME_TYPES, types)
            }
            addCategory(Intent.CATEGORY_OPENABLE)
        }
        activity.startActivityForResult(Intent.createChooser(intent, "Select File"), PICK_FILE_REQUEST)
    }

    override fun onActivityResult(activity: Activity?, requestCode: Int, resultCode: Int, data: Intent?) {
        if (requestCode != PICK_FILE_REQUEST) return

        if (resultCode == Activity.RESULT_CANCELED) {
            filePromise?.resolve(null)
            filePromise = null
            return
        }

        val uri = data?.data ?: run {
            filePromise?.reject("NO_URI", "No file selected")
            filePromise = null
            return
        }

        try {
            val contentResolver = reactApplicationContext.contentResolver
            val cursor = contentResolver.query(uri, null, null, null, null)
            cursor?.use {
                if (it.moveToFirst()) {
                    val nameIdx = it.getColumnIndex(OpenableColumns.DISPLAY_NAME)
                    val sizeIdx = it.getColumnIndex(OpenableColumns.SIZE)
                    val result = Arguments.createMap().apply {
                        putString("uri", uri.toString())
                        putString("name", it.getString(nameIdx) ?: "file")
                        putDouble("size", if (sizeIdx >= 0) it.getLong(sizeIdx).toDouble() else 0.0)
                        putString("type", contentResolver.getType(uri) ?: "application/octet-stream")
                    }
                    filePromise?.resolve(result)
                }
            }
        } catch (e: Exception) {
            filePromise?.reject("FILE_ERROR", e.message, e)
        } finally {
            filePromise = null
        }
    }

    override fun onNewIntent(intent: Intent?) {}
}
```

---

### Q315. How do you implement background geolocation in a native module?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Native Modules — Location

**Answer:**
```kotlin
// Android — Background location with ForegroundService
class LocationModule(reactContext: ReactApplicationContext)
    : ReactContextBaseJavaModule(reactContext) {

    override fun getName() = "LocationModule"

    @ReactMethod
    fun startBackgroundTracking(config: ReadableMap, promise: Promise) {
        val interval = config.getInt("interval") * 1000L  // convert to ms
        val distanceFilter = config.getDouble("distanceFilter").toFloat()

        val serviceIntent = Intent(reactApplicationContext, LocationForegroundService::class.java)
            .putExtra("interval", interval)
            .putExtra("distanceFilter", distanceFilter)

        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            reactApplicationContext.startForegroundService(serviceIntent)
        } else {
            reactApplicationContext.startService(serviceIntent)
        }
        promise.resolve(true)
    }

    @ReactMethod
    fun stopBackgroundTracking(promise: Promise) {
        reactApplicationContext.stopService(
            Intent(reactApplicationContext, LocationForegroundService::class.java)
        )
        promise.resolve(true)
    }

    @ReactMethod
    fun getCurrentLocation(promise: Promise) {
        val fusedClient = LocationServices.getFusedLocationProviderClient(reactApplicationContext)
        if (ActivityCompat.checkSelfPermission(
                reactApplicationContext, Manifest.permission.ACCESS_FINE_LOCATION
            ) != PackageManager.PERMISSION_GRANTED) {
            promise.reject("PERMISSION_DENIED", "Location permission not granted")
            return
        }
        fusedClient.lastLocation.addOnSuccessListener { loc ->
            if (loc != null) {
                promise.resolve(Arguments.createMap().apply {
                    putDouble("latitude", loc.latitude)
                    putDouble("longitude", loc.longitude)
                    putDouble("accuracy", loc.accuracy.toDouble())
                    putDouble("altitude", loc.altitude)
                    putDouble("timestamp", loc.time.toDouble())
                })
            } else {
                promise.reject("NO_LOCATION", "Could not get current location")
            }
        }.addOnFailureListener { e ->
            promise.reject("LOCATION_ERROR", e.message, e)
        }
    }

    @ReactMethod fun addListener(name: String) {}
    @ReactMethod fun removeListeners(count: Int) {}
}
```

---

### Q316. How do you implement push notification handling in a native module?

**Difficulty:** 🔴 Hard | **Frequency:** Very High | **Category:** Push Notifications + Native

**Answer:**
```kotlin
// Android — FCM notification handling module
import com.google.firebase.messaging.FirebaseMessaging
import com.google.firebase.messaging.RemoteMessage

class PushNotificationModule(reactContext: ReactApplicationContext)
    : ReactContextBaseJavaModule(reactContext) {

    override fun getName() = "PushNotificationModule"

    @ReactMethod
    fun getFCMToken(promise: Promise) {
        FirebaseMessaging.getInstance().token.addOnCompleteListener { task ->
            if (task.isSuccessful) {
                promise.resolve(task.result)
            } else {
                promise.reject("TOKEN_ERROR", task.exception?.message)
            }
        }
    }

    @ReactMethod
    fun requestPermission(promise: Promise) {
        // Android 13+ requires POST_NOTIFICATIONS permission
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            if (ContextCompat.checkSelfPermission(
                    reactApplicationContext, Manifest.permission.POST_NOTIFICATIONS
                ) == PackageManager.PERMISSION_GRANTED) {
                promise.resolve("granted")
            } else {
                // Request permission (handled via ActivityEventListener)
                promise.resolve("not_determined")
            }
        } else {
            promise.resolve("granted") // Pre-Android 13 — no runtime permission needed
        }
    }

    @ReactMethod
    fun subscribeToTopic(topic: String, promise: Promise) {
        FirebaseMessaging.getInstance().subscribeToTopic(topic)
            .addOnCompleteListener { task ->
                if (task.isSuccessful) promise.resolve(true)
                else promise.reject("SUBSCRIBE_ERROR", task.exception?.message)
            }
    }

    @ReactMethod
    fun setBadgeCount(count: Int, promise: Promise) {
        // Android badge count via ShortcutBadger
        try {
            ShortcutBadger.applyCount(reactApplicationContext, count)
            promise.resolve(true)
        } catch (e: Exception) {
            promise.reject("BADGE_ERROR", e.message)
        }
    }
}

// Handle incoming notifications (in your FirebaseMessagingService)
class MyFirebaseMessagingService : FirebaseMessagingService() {
    override fun onMessageReceived(message: RemoteMessage) {
        // Forward to React Native if app is in foreground
        val intent = Intent("onRemoteNotification")
        intent.putExtra("data", Gson().toJson(message.data))
        LocalBroadcastManager.getInstance(this).sendBroadcast(intent)
    }

    override fun onNewToken(token: String) {
        // Forward new token to JS
        val params = Arguments.createMap().apply { putString("token", token) }
        // emit via ReactInstanceManager
    }
}
```

---

### Q317. How do you implement a native PDF viewer module?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Native View Managers

**Answer:**
```swift
// iOS — PDFView as React Native component
import PDFKit

// ViewManager (exposes native view to React Native)
@objc(PDFViewManager)
class PDFViewManager: RCTViewManager {

    override func view() -> UIView! {
        return PDFReactView()
    }

    @objc override static func requiresMainQueueSetup() -> Bool { true }
}

// Native view
class PDFReactView: UIView {
    private let pdfView = PDFView()
    var onPageChange: RCTBubblingEventBlock?
    var onLoadComplete: RCTBubblingEventBlock?

    override init(frame: CGRect) {
        super.init(frame: frame)
        pdfView.autoScales = true
        pdfView.displayMode = .singlePageContinuous
        addSubview(pdfView)
        pdfView.translatesAutoresizingMaskIntoConstraints = false
        NSLayoutConstraint.activate([
            pdfView.topAnchor.constraint(equalTo: topAnchor),
            pdfView.leadingAnchor.constraint(equalTo: leadingAnchor),
            pdfView.trailingAnchor.constraint(equalTo: trailingAnchor),
            pdfView.bottomAnchor.constraint(equalTo: bottomAnchor),
        ])
    }
    required init?(coder: NSCoder) { fatalError() }

    @objc var url: String = "" {
        didSet {
            guard let pdfURL = URL(string: url),
                  let document = PDFDocument(url: pdfURL) else { return }
            pdfView.document = document
            onLoadComplete?(["pageCount": document.pageCount])

            NotificationCenter.default.addObserver(
                self,
                selector: #selector(pageChanged),
                name: .PDFViewPageChanged,
                object: pdfView
            )
        }
    }

    @objc func pageChanged() {
        guard let page = pdfView.currentPage,
              let doc = pdfView.document else { return }
        onPageChange?(["page": doc.index(for: page) + 1])
    }
}
```

```objc
// Bridge header
@interface RCT_EXTERN_MODULE(PDFViewManager, RCTViewManager)
RCT_EXPORT_VIEW_PROPERTY(url, NSString)
RCT_EXPORT_VIEW_PROPERTY(onPageChange, RCTBubblingEventBlock)
RCT_EXPORT_VIEW_PROPERTY(onLoadComplete, RCTBubblingEventBlock)
@end
```

```js
// JavaScript component
const PDFView = requireNativeComponent('PDFView');

export const PDFViewer = ({ url, onPageChange, style }) => (
    <PDFView
        url={url}
        onPageChange={(e) => onPageChange?.(e.nativeEvent.page)}
        style={style}
    />
);
```

---

### Q318. How do you debug Native Modules?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Debugging + Native Modules

**Answer:**
```kotlin
// Android debugging strategies:

// 1. Log from native code (visible in Logcat)
Log.d("MyModule", "getBatteryLevel called")
Log.e("MyModule", "Error: ${e.message}", e)
// Filter in Logcat: tag:MyModule

// 2. Flipper — Native Logs plugin
// Flipper → Logs → filter by your module tag
// Shows both Java and React Native logs side by side

// 3. Android Studio Debugger
// Set breakpoints in Kotlin/Java native module code
// Run app → Attach to process → your app
// Breakpoints hit when JS calls your @ReactMethod

// 4. Chrome DevTools for JS side
// Shake device → Debug JS Remotely → Chrome DevTools
// Network tab shows calls, Console shows errors from JS wrapper
```

```swift
// iOS debugging strategies:

// 1. print() from Swift (visible in Xcode console)
print("[MyModule] authenticate called with reason: \(reason)")

// 2. Xcode Debugger — set breakpoints in Swift module
// Run from Xcode → breakpoints hit on native method calls

// 3. NSLog (visible in system console and Xcode)
NSLog("[MyModule] Error: %@", error?.localizedDescription ?? "unknown")

// 4. Instruments — for performance profiling of native code
// Product → Profile → Time Profiler
```

```js
// JS-side debugging:
// 1. Verify module is registered
import { NativeModules } from 'react-native';
console.log('Available modules:', Object.keys(NativeModules));
// If your module name is missing → registration error

// 2. Check for null module
const { MyModule } = NativeModules;
if (!MyModule) {
    console.error('MyModule not found. Check native registration.');
}

// 3. React Native Error Overlay
// Unhandled promise rejections from native modules show as red screen
// Error includes native error code and message

// 4. Error boundaries for native errors
try {
    await MyModule.doSomething();
} catch (error) {
    console.log('Error code:', error.code);
    console.log('Error message:', error.message);
    // error.nativeStackAndroid or error.nativeStackIOS for stack trace
}
```

---

### Q319. What is `processColor` and why do you need it for native modules?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Native Modules

**Answer:**
React Native's JS color system uses strings (`'#FF0000'`, `'red'`, `'rgba(255,0,0,1)'`). Native code needs colors as integers. `processColor` converts JS color strings to native integers.

```js
import { processColor } from 'react-native';

// processColor converts any valid React Native color to a native integer
processColor('#FF0000')         // -65536 (ARGB int on Android)
processColor('red')             // -65536
processColor('rgba(255,0,0,1)') // -65536
processColor('#FF000080')       // semi-transparent red

// Use when passing colors as props to native views
<NativeColorPickerView
    selectedColor={processColor('#6200EE')}  // pass as integer to native
    onColorSelected={handleColorChange}
/>
```

```kotlin
// Android — receive color as Int
@ReactProp(name = "selectedColor", customType = "Color")
fun setSelectedColor(view: ColorPickerView, color: Int) {
    view.setColor(color)
    // color is already an ARGB int — no conversion needed
}
```

```swift
// iOS — receive color from processed value
// The @ReactProp equivalent in iOS receives UIColor
// RCTConvert handles the conversion automatically

// In ViewManager:
@objc func setSelectedColor(_ view: ColorPickerView, color: UIColor) {
    view.selectedColor = color
}
```

```js
// In the Objective-C bridge header:
RCT_EXPORT_VIEW_PROPERTY(selectedColor, UIColor) // RCTConvert handles UIColor conversion
```

---

### Q320. How do you implement a Native Module for device info?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Native Modules — Practical

**Answer:**
```kotlin
// Android — DeviceInfoModule.kt (production-quality)
class DeviceInfoModule(reactContext: ReactApplicationContext)
    : ReactContextBaseJavaModule(reactContext) {

    override fun getName() = "DeviceInfo"

    override fun getConstants() = mapOf(
        "deviceId" to Settings.Secure.getString(
            reactApplicationContext.contentResolver, Settings.Secure.ANDROID_ID
        ),
        "brand" to Build.BRAND,
        "manufacturer" to Build.MANUFACTURER,
        "model" to Build.MODEL,
        "osVersion" to Build.VERSION.RELEASE,
        "sdkVersion" to Build.VERSION.SDK_INT,
        "isEmulator" to isEmulator(),
    )

    @ReactMethod
    fun getStorageInfo(promise: Promise) {
        val stat = StatFs(Environment.getDataDirectory().path)
        val total = stat.totalBytes
        val available = stat.availableBytes

        promise.resolve(Arguments.createMap().apply {
            putDouble("totalStorage", total.toDouble())
            putDouble("freeStorage", available.toDouble())
            putDouble("usedStorage", (total - available).toDouble())
        })
    }

    @ReactMethod
    fun getNetworkInfo(promise: Promise) {
        val cm = reactApplicationContext
            .getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
        val network = cm.activeNetwork
        val caps = cm.getNetworkCapabilities(network)

        val type = when {
            caps == null -> "NONE"
            caps.hasTransport(NetworkCapabilities.TRANSPORT_WIFI) -> "WIFI"
            caps.hasTransport(NetworkCapabilities.TRANSPORT_CELLULAR) -> "CELLULAR"
            caps.hasTransport(NetworkCapabilities.TRANSPORT_ETHERNET) -> "ETHERNET"
            else -> "UNKNOWN"
        }
        promise.resolve(type)
    }

    private fun isEmulator(): Boolean = (
        Build.FINGERPRINT.startsWith("generic") ||
        Build.MODEL.contains("Emulator") ||
        Build.MODEL.contains("Android SDK")
    )
}
```

---

### Q321. How do you implement a native module for Razorpay payment gateway?

**Difficulty:** 🔴 Hard | **Frequency:** Very High | **Category:** Native Modules — Payments (India)

**Answer:**
```kotlin
// Android — RazorpayModule.kt
// Uses Razorpay Android SDK: implementation 'com.razorpay:checkout:1.6.26'

class RazorpayModule(reactContext: ReactApplicationContext)
    : ReactContextBaseJavaModule(reactContext),
      ActivityEventListener,
      PaymentResultWithDataListener {

    override fun getName() = "RazorpayModule"
    private var paymentPromise: Promise? = null

    init { reactContext.addActivityEventListener(this) }

    @ReactMethod
    fun startPayment(options: ReadableMap, promise: Promise) {
        val activity = currentActivity
            ?: return promise.reject("NO_ACTIVITY", "No activity")
        paymentPromise = promise

        activity.runOnUiThread {
            val checkout = Checkout()
            checkout.setKeyID(options.getString("key") ?: "")
            checkout.setImage(R.mipmap.ic_launcher)

            try {
                val jsonOptions = JSONObject().apply {
                    put("name", options.getString("name") ?: "")
                    put("description", options.getString("description") ?: "")
                    put("currency", options.getString("currency") ?: "INR")
                    put("amount", options.getInt("amount"))       // in paise
                    put("order_id", options.getString("order_id") ?: "")
                    put("prefill", JSONObject().apply {
                        put("name", options.getString("prefill_name") ?: "")
                        put("email", options.getString("prefill_email") ?: "")
                        put("contact", options.getString("prefill_contact") ?: "")
                    })
                    put("theme", JSONObject().apply {
                        put("color", options.getString("theme_color") ?: "#6200EE")
                    })
                }
                checkout.open(activity as AppCompatActivity, jsonOptions)
            } catch (e: Exception) {
                promise.reject("RAZORPAY_ERROR", e.message, e)
                paymentPromise = null
            }
        }
    }

    // Called on payment success
    override fun onPaymentSuccess(razorpayPaymentId: String?, paymentData: PaymentData?) {
        val result = Arguments.createMap().apply {
            putString("razorpay_payment_id", razorpayPaymentId)
            putString("razorpay_order_id", paymentData?.orderId)
            putString("razorpay_signature", paymentData?.signature)
        }
        paymentPromise?.resolve(result)
        paymentPromise = null
    }

    // Called on payment failure
    override fun onPaymentError(code: Int, description: String?, paymentData: PaymentData?) {
        paymentPromise?.reject(code.toString(), description)
        paymentPromise = null
    }

    override fun onActivityResult(activity: Activity?, requestCode: Int, resultCode: Int, data: Intent?) {}
    override fun onNewIntent(intent: Intent?) {}
}
```

```js
// JavaScript wrapper
import { NativeModules } from 'react-native';
const { RazorpayModule } = NativeModules;

export const Razorpay = {
    openPaymentCheckout: async (options) => {
        try {
            const result = await RazorpayModule.startPayment(options);
            return result; // { razorpay_payment_id, razorpay_order_id, razorpay_signature }
        } catch (error) {
            throw {
                code: parseInt(error.code),
                description: error.message,
            };
        }
    }
};

// Usage
const handlePayment = async () => {
    try {
        const result = await Razorpay.openPaymentCheckout({
            key: 'rzp_test_xxxxxxxx',
            name: 'Your Company',
            description: 'ERP Subscription',
            currency: 'INR',
            amount: 99900, // ₹999.00 in paise
            order_id: 'order_xxxxxxxx',
            prefill_name: 'Devesh Kumar',
            prefill_email: 'devesh@example.com',
            prefill_contact: '9876543210',
            theme_color: '#6200EE',
        });
        await verifyPayment(result);
    } catch (error) {
        if (error.code === 0) {
            Alert.alert('Payment cancelled');
        } else {
            Alert.alert('Payment failed', error.description);
        }
    }
};
```

---

### Q322. How do you implement deep linking handling in a native module?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Deep Linking + Native

**Answer:**
```kotlin
// Android — handle deep links in MainActivity
// android/app/src/main/java/com/yourapp/MainActivity.kt

class MainActivity : ReactActivity(), ActivityEventListener {

    override fun getMainComponentName() = "YourApp"

    // Handle deep link when app is launched from a link (cold start)
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // Deep link data available in intent
        val uri = intent?.data
        // React Navigation reads this via Linking.getInitialURL()
    }

    // Handle deep link when app is already open (warm start)
    override fun onNewIntent(intent: Intent?) {
        super.onNewIntent(intent)
        setIntent(intent)
        // ReactApplication processes this via Linking API
    }
}
```

```kotlin
// Custom deep link module with validation
class DeepLinkModule(reactContext: ReactApplicationContext)
    : ReactContextBaseJavaModule(reactContext), ActivityEventListener {

    override fun getName() = "DeepLinkModule"

    init { reactContext.addActivityEventListener(this) }

    @ReactMethod
    fun getInitialURL(promise: Promise) {
        val activity = currentActivity
        val uri = activity?.intent?.data
        promise.resolve(uri?.toString())
    }

    override fun onNewIntent(intent: Intent?) {
        val uri = intent?.data?.toString() ?: return
        val params = Arguments.createMap().apply { putString("url", uri) }
        sendEvent("onDeepLink", params)
    }

    private fun sendEvent(name: String, params: WritableMap) {
        reactApplicationContext
            .getJSModule(DeviceEventManagerModule.RCTDeviceEventEmitter::class.java)
            .emit(name, params)
    }

    override fun onActivityResult(a: Activity?, r: Int, c: Int, d: Intent?) {}
    @ReactMethod fun addListener(name: String) {}
    @ReactMethod fun removeListeners(count: Int) {}
}
```

```js
// React Navigation + Linking configuration
const config = {
    screens: {
        Home: 'home',
        Profile: 'profile/:userId',
        Product: { path: 'product/:productId', parse: { productId: Number } },
        Payment: 'payment/:orderId',
    },
};

const linking = {
    prefixes: ['yourapp://', 'https://yourapp.com'],
    config,
    // Handle custom validation
    getStateFromPath: (path, options) => {
        // Optional: validate and transform the path
        if (path.startsWith('/admin')) return null; // reject admin paths
        return getStateFromPath(path, options);
    },
};

export default function App() {
    return (
        <NavigationContainer linking={linking}>
            <AppStack />
        </NavigationContainer>
    );
}
```

---

### Q323. How do you implement a custom splash screen with animations via native module?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Native Modules — UI

**Answer:**
```kotlin
// Android — Animated Splash Module
class SplashModule(reactContext: ReactApplicationContext)
    : ReactContextBaseJavaModule(reactContext) {

    override fun getName() = "SplashModule"

    @ReactMethod
    fun hide(duration: Int, promise: Promise) {
        val activity = currentActivity ?: return promise.resolve(false)

        activity.runOnUiThread {
            val splashView = activity.findViewById<View>(R.id.splash_container)
                ?: return@runOnUiThread

            // Fade out animation
            val anim = android.view.animation.AnimationUtils.loadAnimation(
                activity, android.R.anim.fade_out
            )
            anim.duration = duration.toLong()
            anim.setAnimationListener(object : android.view.animation.Animation.AnimationListener {
                override fun onAnimationStart(a: android.view.animation.Animation?) {}
                override fun onAnimationRepeat(a: android.view.animation.Animation?) {}
                override fun onAnimationEnd(a: android.view.animation.Animation?) {
                    splashView.visibility = View.GONE
                    promise.resolve(true)
                }
            })
            splashView.startAnimation(anim)
        }
    }

    @ReactMethod
    fun show(promise: Promise) {
        val activity = currentActivity ?: return promise.reject("NO_ACTIVITY", "")
        activity.runOnUiThread {
            val splashView = activity.findViewById<View>(R.id.splash_container)
            splashView?.visibility = View.VISIBLE
            promise.resolve(true)
        }
    }
}
```

```js
// JS — coordinated splash + RN animation
const useSplashScreen = () => {
    const { SplashModule } = NativeModules;
    const contentOpacity = useRef(new Animated.Value(0)).current;

    const hideSplashAndReveal = useCallback(async () => {
        // Fade out native splash
        await SplashModule.hide(300);

        // Fade in React Native content
        Animated.timing(contentOpacity, {
            toValue: 1,
            duration: 400,
            useNativeDriver: true,
        }).start();
    }, []);

    return { contentOpacity, hideSplashAndReveal };
};

// App.tsx
const App = () => {
    const { contentOpacity, hideSplashAndReveal } = useSplashScreen();

    useEffect(() => {
        const init = async () => {
            await loadCriticalData();       // auth, user prefs, theme
            await hideSplashAndReveal();    // reveal app smoothly
        };
        init();
    }, []);

    return (
        <Animated.View style={{ flex: 1, opacity: contentOpacity }}>
            <AppContent />
        </Animated.View>
    );
};
```

---

### Q324. What is `RCTConvert` and how is it used in iOS native modules?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** iOS Native Modules

**Answer:**
`RCTConvert` is an Objective-C utility class that converts JavaScript values to native iOS types. React Native uses it internally for all prop conversions.

```objc
// Auto-used by RCT_EXPORT_VIEW_PROPERTY
// Explicit use in complex prop handling:

#import <React/RCTConvert.h>

// In your ViewManager:
RCT_CUSTOM_VIEW_PROPERTY(style, NSDictionary, MyCustomView) {
    if (json) {
        CGFloat cornerRadius = [RCTConvert CGFloat:json[@"borderRadius"]];
        UIColor *bgColor = [RCTConvert UIColor:json[@"backgroundColor"]];
        view.layer.cornerRadius = cornerRadius;
        view.backgroundColor = bgColor;
    }
}
```

```swift
// Swift equivalent — access through Objective-C interop
@objc func configureView(_ view: MyView, json: NSDictionary) {
    if let colorValue = json["backgroundColor"] {
        // RCTConvert converts JS color int to UIColor
        let color = RCTConvert.uiColor(colorValue)
        view.backgroundColor = color
    }
}

// Common RCTConvert methods:
// RCTConvert.uiColor(_ json) → UIColor
// RCTConvert.cgFloat(_ json) → CGFloat
// RCTConvert.nsString(_ json) → NSString
// RCTConvert.nsDictionary(_ json) → NSDictionary
// RCTConvert.nsArray(_ json) → NSArray
// RCTConvert.bool(_ json) → Bool
// RCTConvert.nsInteger(_ json) → NSInteger
// RCTConvert.cgRect(_ json) → CGRect
// RCTConvert.uiFont(_ json) → UIFont
```

---

### Q325. How do you implement a native status bar module?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Native Modules — UI

**Answer:**
```kotlin
// Android — StatusBarModule.kt
class StatusBarModule(reactContext: ReactApplicationContext)
    : ReactContextBaseJavaModule(reactContext) {

    override fun getName() = "StatusBarModule"

    @ReactMethod
    fun setBarStyle(style: String, animated: Boolean) {
        currentActivity?.runOnUiThread {
            val window = currentActivity?.window ?: return@runOnUiThread

            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
                val controller = window.insetsController ?: return@runOnUiThread
                when (style) {
                    "dark-content" ->
                        controller.setSystemBarsAppearance(
                            WindowInsetsController.APPEARANCE_LIGHT_STATUS_BARS,
                            WindowInsetsController.APPEARANCE_LIGHT_STATUS_BARS
                        )
                    "light-content" ->
                        controller.setSystemBarsAppearance(
                            0, WindowInsetsController.APPEARANCE_LIGHT_STATUS_BARS
                        )
                }
            } else {
                @Suppress("DEPRECATION")
                var flags = window.decorView.systemUiVisibility
                flags = when (style) {
                    "dark-content" -> flags or View.SYSTEM_UI_FLAG_LIGHT_STATUS_BAR
                    else -> flags and View.SYSTEM_UI_FLAG_LIGHT_STATUS_BAR.inv()
                }
                window.decorView.systemUiVisibility = flags
            }
        }
    }

    @ReactMethod
    fun setBackgroundColor(color: Int, animated: Boolean) {
        currentActivity?.runOnUiThread {
            currentActivity?.window?.statusBarColor = color
        }
    }

    @ReactMethod
    fun setHidden(hidden: Boolean, animated: Boolean) {
        currentActivity?.runOnUiThread {
            if (hidden) {
                currentActivity?.window?.addFlags(WindowManager.LayoutParams.FLAG_FULLSCREEN)
            } else {
                currentActivity?.window?.clearFlags(WindowManager.LayoutParams.FLAG_FULLSCREEN)
            }
        }
    }
}
```

---

### Q326. How do you implement a native network interceptor module?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Networking + Native Modules

**Answer:**
```kotlin
// Android — OkHttp Interceptor Module
// Intercepts all HTTP traffic from React Native's networking layer

class NetworkInterceptorModule(reactContext: ReactApplicationContext)
    : ReactContextBaseJavaModule(reactContext) {

    override fun getName() = "NetworkInterceptor"

    // OkHttp interceptor — logs all requests/responses
    class LoggingInterceptor(private val emitter: DeviceEventManagerModule.RCTDeviceEventEmitter)
        : Interceptor {

        override fun intercept(chain: Interceptor.Chain): Response {
            val request = chain.request()
            val requestId = System.nanoTime().toString()
            val startTime = System.currentTimeMillis()

            // Emit request start event
            emitter.emit("onNetworkRequest", Arguments.createMap().apply {
                putString("requestId", requestId)
                putString("url", request.url.toString())
                putString("method", request.method)
                putDouble("timestamp", startTime.toDouble())
            })

            val response = try {
                chain.proceed(request)
            } catch (e: Exception) {
                emitter.emit("onNetworkError", Arguments.createMap().apply {
                    putString("requestId", requestId)
                    putString("error", e.message)
                })
                throw e
            }

            val duration = System.currentTimeMillis() - startTime

            // Emit response event
            emitter.emit("onNetworkResponse", Arguments.createMap().apply {
                putString("requestId", requestId)
                putInt("statusCode", response.code)
                putDouble("duration", duration.toDouble())
                putBoolean("success", response.isSuccessful)
            })

            return response
        }
    }

    // Install interceptor into RN's OkHttp client
    @ReactMethod
    fun install(promise: Promise) {
        try {
            val emitter = reactApplicationContext
                .getJSModule(DeviceEventManagerModule.RCTDeviceEventEmitter::class.java)

            OkHttpClientProvider.addOkHttpClientFactory { builder ->
                builder.addInterceptor(LoggingInterceptor(emitter))
            }
            promise.resolve(true)
        } catch (e: Exception) {
            promise.reject("INSTALL_ERROR", e.message, e)
        }
    }

    @ReactMethod fun addListener(name: String) {}
    @ReactMethod fun removeListeners(count: Int) {}
}
```

---

### Q327. What are `WritableMap` and `ReadableMap` in Android native modules?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Android Native Modules

**Answer:**
```kotlin
// ReadableMap — JS object received by native (read-only from native's perspective)
// WritableMap — JS object created by native to send back to JS

@ReactMethod
fun processConfig(config: ReadableMap, promise: Promise) {
    // Reading from ReadableMap (received from JS)
    val apiKey = config.getString("apiKey") ?: ""
    val timeout = if (config.hasKey("timeout")) config.getInt("timeout") else 30
    val isDebug = if (config.hasKey("debug")) config.getBoolean("debug") else false

    // Nested object
    if (config.hasKey("headers")) {
        val headers = config.getMap("headers")
        val authHeader = headers?.getString("Authorization") ?: ""
    }

    // Array
    if (config.hasKey("allowedDomains")) {
        val domains = config.getArray("allowedDomains")
        for (i in 0 until (domains?.size() ?: 0)) {
            val domain = domains?.getString(i)
        }
    }

    // Creating WritableMap to return to JS
    val result = Arguments.createMap().apply {
        putString("status", "initialized")
        putBoolean("success", true)
        putInt("timeout", timeout)

        // Nested WritableMap
        val metadata = Arguments.createMap().apply {
            putString("version", "1.0.0")
            putDouble("timestamp", System.currentTimeMillis().toDouble())
        }
        putMap("metadata", metadata)

        // WritableArray inside WritableMap
        val features = Arguments.createArray().apply {
            pushString("biometrics")
            pushString("offline")
        }
        putArray("features", features)
    }
    promise.resolve(result)
}

// ReadableMap type checking:
// config.getType("key") returns ReadableType: Null, Boolean, Number, String, Map, Array
when (config.getType("value")) {
    ReadableType.String -> config.getString("value")
    ReadableType.Number -> config.getDouble("value")
    ReadableType.Boolean -> config.getBoolean("value")
    ReadableType.Map -> config.getMap("value")
    ReadableType.Array -> config.getArray("value")
    ReadableType.Null -> null
    else -> null
}
```

---

### Q328. How do you implement a native module that accesses the camera roll / photo library?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Native Modules — Photos

**Answer:**
```swift
// iOS — PhotoLibraryModule.swift
import Photos

@objc(PhotoLibraryModule)
class PhotoLibraryModule: NSObject {

    @objc static func requiresMainQueueSetup() -> Bool { false }

    @objc func requestPermission(_ resolve: @escaping RCTPromiseResolveBlock,
                                  rejecter reject: @escaping RCTPromiseRejectBlock) {
        PHPhotoLibrary.requestAuthorization { status in
            switch status {
            case .authorized, .limited: resolve("authorized")
            case .denied: resolve("denied")
            case .restricted: resolve("restricted")
            case .notDetermined: resolve("notDetermined")
            @unknown default: resolve("unknown")
            }
        }
    }

    @objc func getPhotos(_ options: NSDictionary,
                          resolver resolve: @escaping RCTPromiseResolveBlock,
                          rejecter reject: @escaping RCTPromiseRejectBlock) {
        let limit = options["limit"] as? Int ?? 20
        let offset = options["offset"] as? Int ?? 0

        let fetchOptions = PHFetchOptions()
        fetchOptions.fetchLimit = limit + offset
        fetchOptions.sortDescriptors = [NSSortDescriptor(key: "creationDate", ascending: false)]

        let assets = PHAsset.fetchAssets(with: .image, options: fetchOptions)
        var photos: [[String: Any]] = []
        let manager = PHImageManager.default()
        let requestOptions = PHImageRequestOptions()
        requestOptions.isSynchronous = true
        requestOptions.deliveryMode = .opportunistic

        let start = min(offset, assets.count)
        let end = min(offset + limit, assets.count)

        for i in start..<end {
            let asset = assets.object(at: i)
            manager.requestImage(
                for: asset,
                targetSize: CGSize(width: 200, height: 200),
                contentMode: .aspectFill,
                options: requestOptions
            ) { image, _ in
                guard let image = image,
                      let data = image.jpegData(compressionQuality: 0.7) else { return }
                photos.append([
                    "id": asset.localIdentifier,
                    "width": asset.pixelWidth,
                    "height": asset.pixelHeight,
                    "thumbnail": "data:image/jpeg;base64,\(data.base64EncodedString())",
                    "creationDate": asset.creationDate?.timeIntervalSince1970 ?? 0,
                ])
            }
        }
        resolve(["photos": photos, "hasMore": end < assets.count])
    }
}
```

---

### Q329. How do you create a cross-platform JS wrapper for a native module?

**Difficulty:** 🟡 Medium | **Frequency:** Very High | **Category:** Native Modules — Architecture

**Answer:**
```js
// src/modules/Keychain.ts
// Clean cross-platform wrapper with types, fallbacks, and error handling

import { NativeModules, Platform } from 'react-native';

const { KeychainModule } = NativeModules;

export interface KeychainResult {
    service: string;
    username: string;
    password: string;
}

export const Keychain = {
    // setGenericPassword — stores credentials with service name
    async setGenericPassword(
        username: string,
        password: string,
        service: string = 'default'
    ): Promise<boolean> {
        if (!KeychainModule) {
            console.warn('Keychain: native module not available, using fallback');
            // Fallback for simulators or test environments
            if (__DEV__) {
                await AsyncStorage.setItem(`keychain_${service}`, JSON.stringify({ username, password }));
                return true;
            }
            throw new Error('Keychain not available');
        }
        return KeychainModule.setItem(`${service}:${username}`, password);
    },

    async getGenericPassword(service: string = 'default'): Promise<KeychainResult | null> {
        if (!KeychainModule) {
            if (__DEV__) {
                const stored = await AsyncStorage.getItem(`keychain_${service}`);
                if (!stored) return null;
                const { username, password } = JSON.parse(stored);
                return { service, username, password };
            }
            throw new Error('Keychain not available');
        }

        // Different key format per platform
        const key = Platform.OS === 'ios'
            ? `${service}:credentials`
            : `${service}:username`;

        const value = await KeychainModule.getItem(key);
        if (!value) return null;

        const [username, password] = value.split(':');
        return { service, username, password };
    },

    async resetGenericPassword(service: string = 'default'): Promise<boolean> {
        if (!KeychainModule) return false;
        return KeychainModule.removeItem(`${service}:credentials`);
    },

    // Check if keychain is available
    isAvailable(): boolean {
        return !!KeychainModule;
    },
};
```

---

### Q330. What happens when a Native Module is not available on a platform?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Native Modules

**Answer:**
```js
import { NativeModules, Platform } from 'react-native';

// NativeModules.SomeModule will be undefined if not registered on the current platform
const { IOSOnlyModule } = NativeModules;

// ❌ Will crash on Android
const handlePress = async () => {
    const result = await IOSOnlyModule.doSomething(); // TypeError: null is not an object
};

// ✅ Guard with null check
const handlePress = async () => {
    if (!IOSOnlyModule) {
        console.warn('IOSOnlyModule not available on this platform');
        return;
    }
    const result = await IOSOnlyModule.doSomething();
};

// ✅ Platform.select for platform-specific modules
const getModule = () => Platform.select({
    ios: () => NativeModules.IOSOnlyModule,
    android: () => NativeModules.AndroidOnlyModule,
    default: () => null,
})?.();

// ✅ Graceful degradation with feature flags
const BiometricAuth = {
    isAvailable: async () => {
        if (!NativeModules.BiometricModule) return false;
        try {
            const type = await NativeModules.BiometricModule.canAuthenticate();
            return type !== 'NONE' && type !== 'NO_HARDWARE';
        } catch {
            return false;
        }
    },
    authenticate: async (reason: string) => {
        if (!NativeModules.BiometricModule) {
            // Fallback: show PIN/password prompt instead
            return showPasswordPrompt(reason);
        }
        return NativeModules.BiometricModule.authenticate(reason);
    }
};

// ✅ TurboModuleRegistry.get (returns null instead of throwing)
import { TurboModuleRegistry } from 'react-native';
const module = TurboModuleRegistry.get('OptionalModule');
// module is null if not registered — no crash
```

---

### Q331. How do you expose constants from a native module?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Native Modules

**Answer:**
Constants are synchronously available in JS immediately — no async call needed.

```kotlin
// Android — getConstants()
class AppConfigModule(reactContext: ReactApplicationContext)
    : ReactContextBaseJavaModule(reactContext) {

    override fun getName() = "AppConfig"

    // getConstants() is called once at startup — keep it fast
    override fun getConstants(): Map<String, Any> {
        val packageInfo = reactApplicationContext.packageManager
            .getPackageInfo(reactApplicationContext.packageName, 0)

        return mapOf(
            "APP_VERSION" to packageInfo.versionName,
            "BUILD_NUMBER" to if (Build.VERSION.SDK_INT >= 28)
                packageInfo.longVersionCode.toString()
                else packageInfo.versionCode.toString(),
            "BUNDLE_ID" to reactApplicationContext.packageName,
            "IS_EMULATOR" to isEmulator(),
            "DEVICE_ID" to Settings.Secure.getString(
                reactApplicationContext.contentResolver,
                Settings.Secure.ANDROID_ID
            ),
            "ENVIRONMENT" to BuildConfig.BUILD_TYPE,  // "debug", "release"
        )
    }
}
```

```swift
// iOS — constantsToExport()
@objc(AppConfigModule)
class AppConfigModule: NSObject, RCTBridgeModule {

    static func moduleName() -> String! { "AppConfig" }

    func constantsToExport() -> [AnyHashable: Any]! {
        let bundle = Bundle.main
        return [
            "APP_VERSION": bundle.object(forInfoDictionaryKey: "CFBundleShortVersionString") ?? "",
            "BUILD_NUMBER": bundle.object(forInfoDictionaryKey: "CFBundleVersion") ?? "",
            "BUNDLE_ID": bundle.bundleIdentifier ?? "",
            "IS_SIMULATOR": TARGET_OS_SIMULATOR != 0,
            "DEVICE_ID": UIDevice.current.identifierForVendor?.uuidString ?? "",
            "ENVIRONMENT": Bundle.main.object(forInfoDictionaryKey: "ENV") as? String ?? "production",
        ]
    }
}
```

```js
// JavaScript — access synchronously
import { NativeModules } from 'react-native';
const { APP_VERSION, BUILD_NUMBER, IS_EMULATOR } = NativeModules.AppConfig;

console.log(`v${APP_VERSION} (${BUILD_NUMBER})`);
if (IS_EMULATOR) console.log('Running on emulator');

// Or access via the module directly
const config = NativeModules.AppConfig;
// No await needed — constants are available synchronously
```

---

### Q332. How do you implement a native module for crash reporting?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Native Modules — Analytics

**Answer:**
```kotlin
// Android — Custom crash reporter + Sentry wrapper
class CrashReportingModule(reactContext: ReactApplicationContext)
    : ReactContextBaseJavaModule(reactContext) {

    override fun getName() = "CrashReporting"

    @ReactMethod
    fun initialize(dsn: String, config: ReadableMap, promise: Promise) {
        try {
            SentryAndroid.init(reactApplicationContext) { options ->
                options.dsn = dsn
                options.tracesSampleRate = if (config.hasKey("tracesSampleRate"))
                    config.getDouble("tracesSampleRate") else 0.1
                options.environment = if (config.hasKey("environment"))
                    config.getString("environment") else "production"
                options.release = if (config.hasKey("release"))
                    config.getString("release") else BuildConfig.VERSION_NAME
            }
            promise.resolve(true)
        } catch (e: Exception) {
            promise.reject("INIT_ERROR", e.message, e)
        }
    }

    @ReactMethod
    fun captureException(message: String, extras: ReadableMap) {
        Sentry.captureException(Exception(message)) { scope ->
            extras.entryIterator.forEach { entry ->
                scope.setExtra(entry.key, entry.value?.toString() ?: "null")
            }
        }
    }

    @ReactMethod
    fun captureMessage(message: String, level: String) {
        val sentryLevel = when (level.lowercase()) {
            "fatal" -> SentryLevel.FATAL
            "error" -> SentryLevel.ERROR
            "warning" -> SentryLevel.WARNING
            "info" -> SentryLevel.INFO
            else -> SentryLevel.DEBUG
        }
        Sentry.captureMessage(message, sentryLevel)
    }

    @ReactMethod
    fun setUser(userId: String, email: String, username: String) {
        Sentry.setUser(User().apply {
            this.id = userId
            this.email = email
            this.username = username
        })
    }

    @ReactMethod
    fun addBreadcrumb(message: String, category: String, level: String) {
        Sentry.addBreadcrumb(Breadcrumb().apply {
            this.message = message
            this.category = category
            this.level = SentryLevel.valueOf(level.uppercase())
        })
    }

    @ReactMethod
    fun clearUser() { Sentry.setUser(null) }
}
```

---

### Q333. How do you handle native module version incompatibilities?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Native Modules

**Answer:**
```js
// Problem: native module API changes between library versions
// Your JS code calls .oldMethodName() but newer native has .newMethodName()

// Strategy 1: Version checking
import { NativeModules } from 'react-native';
const { MyModule } = NativeModules;

const getVersion = () => MyModule?.version ?? 0;

const compatibleAPI = {
    doSomething: () => {
        if (getVersion() >= 2) {
            return MyModule.doSomethingV2(); // new API
        }
        return MyModule.doSomething();       // old API
    }
};

// Strategy 2: Try/catch fallback
const robustCall = async () => {
    try {
        return await MyModule.newMethod();
    } catch (error) {
        if (error.code === 'NOT_FOUND' || error.message?.includes('not a function')) {
            return MyModule.oldMethod(); // fallback to old API
        }
        throw error;
    }
};

// Strategy 3: Constants version flag
// Native module exposes: getConstants() → { MODULE_VERSION: 2 }
const { MODULE_VERSION } = NativeModules.MyModule;

// Strategy 4: checkForUpdate (semver)
// If library version >= '2.0.0', use new API
import { version as rnVersion } from 'react-native/package.json';
import semver from 'semver';

if (semver.gte(myLibVersion, '2.0.0')) {
    // use new API
}
```

---

### Q334. How do you implement a native speech recognition module?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Native Modules

**Answer:**
```swift
// iOS — SpeechRecognitionModule.swift
import Speech

@objc(SpeechModule)
class SpeechModule: RCTEventEmitter {

    private var recognitionRequest: SFSpeechAudioBufferRecognitionRequest?
    private var recognitionTask: SFSpeechRecognitionTask?
    private let audioEngine = AVAudioEngine()
    private var recognizer: SFSpeechRecognizer?

    override func supportedEvents() -> [String]! {
        ["onSpeechStart", "onSpeechPartialResult", "onSpeechFinalResult", "onSpeechError", "onSpeechEnd"]
    }

    override static func requiresMainQueueSetup() -> Bool { false }

    @objc func requestPermission(_ resolve: @escaping RCTPromiseResolveBlock,
                                  rejecter reject: @escaping RCTPromiseRejectBlock) {
        SFSpeechRecognizer.requestAuthorization { status in
            AVAudioSession.sharedInstance().requestRecordPermission { micGranted in
                let speechGranted = status == .authorized
                resolve(speechGranted && micGranted ? "authorized" : "denied")
            }
        }
    }

    @objc func startRecognition(_ locale: String,
                                 resolver resolve: @escaping RCTPromiseResolveBlock,
                                 rejecter reject: @escaping RCTPromiseRejectBlock) {
        recognizer = SFSpeechRecognizer(locale: Locale(identifier: locale))
        guard let recognizer = recognizer, recognizer.isAvailable else {
            reject("NOT_AVAILABLE", "Speech recognition not available for locale: \(locale)", nil)
            return
        }

        do {
            let audioSession = AVAudioSession.sharedInstance()
            try audioSession.setCategory(.record, mode: .measurement, options: .duckOthers)
            try audioSession.setActive(true, options: .notifyOthersOnDeactivation)

            recognitionRequest = SFSpeechAudioBufferRecognitionRequest()
            recognitionRequest?.shouldReportPartialResults = true

            let inputNode = audioEngine.inputNode
            recognitionTask = recognizer.recognitionTask(with: recognitionRequest!) { [weak self] result, error in
                if let result = result {
                    let text = result.bestTranscription.formattedString
                    if result.isFinal {
                        self?.sendEvent(withName: "onSpeechFinalResult", body: ["value": text])
                    } else {
                        self?.sendEvent(withName: "onSpeechPartialResult", body: ["value": text])
                    }
                }
                if let error = error {
                    self?.sendEvent(withName: "onSpeechError", body: ["error": error.localizedDescription])
                }
            }

            let format = inputNode.outputFormat(forBus: 0)
            inputNode.installTap(onBus: 0, bufferSize: 1024, format: format) { [weak self] buffer, _ in
                self?.recognitionRequest?.append(buffer)
            }

            audioEngine.prepare()
            try audioEngine.start()
            sendEvent(withName: "onSpeechStart", body: nil)
            resolve(true)
        } catch {
            reject("START_ERROR", error.localizedDescription, error)
        }
    }

    @objc func stopRecognition(_ resolve: @escaping RCTPromiseResolveBlock,
                                rejecter reject: @escaping RCTPromiseRejectBlock) {
        audioEngine.stop()
        audioEngine.inputNode.removeTap(onBus: 0)
        recognitionRequest?.endAudio()
        recognitionTask?.cancel()
        sendEvent(withName: "onSpeechEnd", body: nil)
        resolve(true)
    }
}
```

---

### Q335. What is `ActivityEventListener` in Android native modules?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Android Native Modules

**Answer:**
`ActivityEventListener` is an interface that allows a native module to receive callbacks from the Android Activity lifecycle — specifically `onActivityResult` and `onNewIntent`.

```kotlin
class DocumentPickerModule(reactContext: ReactApplicationContext)
    : ReactContextBaseJavaModule(reactContext), ActivityEventListener {

    override fun getName() = "DocumentPicker"
    private var pickerPromise: Promise? = null
    private val PICK_DOC = 5678

    // MUST register listener in init block
    init { reactContext.addActivityEventListener(this) }

    @ReactMethod
    fun pickDocument(promise: Promise) {
        val activity = currentActivity ?: return promise.reject("NO_ACTIVITY", "")
        pickerPromise = promise

        val intent = Intent(Intent.ACTION_OPEN_DOCUMENT).apply {
            addCategory(Intent.CATEGORY_OPENABLE)
            type = "application/pdf"
        }
        activity.startActivityForResult(intent, PICK_DOC)
    }

    // Called after startActivityForResult completes (user picks or cancels)
    override fun onActivityResult(activity: Activity?, requestCode: Int, resultCode: Int, data: Intent?) {
        if (requestCode != PICK_DOC) return

        when (resultCode) {
            Activity.RESULT_OK -> {
                val uri = data?.data
                if (uri != null) {
                    pickerPromise?.resolve(Arguments.createMap().apply {
                        putString("uri", uri.toString())
                    })
                } else {
                    pickerPromise?.reject("NO_URI", "No document selected")
                }
            }
            Activity.RESULT_CANCELED -> pickerPromise?.resolve(null)
        }
        pickerPromise = null
    }

    // Called when a new Intent is received (deep link, notification tap)
    override fun onNewIntent(intent: Intent?) {
        val url = intent?.data?.toString() ?: return
        sendEvent("onDeepLink", Arguments.createMap().apply { putString("url", url) })
    }
}
```

---

### Q336. How do you implement a native module for reading device sensors?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Native Modules — Sensors

**Answer:**
```kotlin
// Android — SensorsModule.kt
class SensorsModule(reactContext: ReactApplicationContext)
    : ReactContextBaseJavaModule(reactContext) {

    override fun getName() = "SensorsModule"

    private val sensorManager = reactApplicationContext
        .getSystemService(Context.SENSOR_SERVICE) as SensorManager
    private var sensorListener: SensorEventListener? = null

    @ReactMethod
    fun startAccelerometer(intervalMs: Int, promise: Promise) {
        val sensor = sensorManager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER)
            ?: return promise.reject("NO_SENSOR", "Accelerometer not available")

        val delay = when {
            intervalMs <= 20  -> SensorManager.SENSOR_DELAY_FASTEST
            intervalMs <= 60  -> SensorManager.SENSOR_DELAY_GAME
            intervalMs <= 200 -> SensorManager.SENSOR_DELAY_UI
            else              -> SensorManager.SENSOR_DELAY_NORMAL
        }

        sensorListener = object : SensorEventListener {
            override fun onSensorChanged(event: SensorEvent) {
                val params = Arguments.createMap().apply {
                    putDouble("x", event.values[0].toDouble())
                    putDouble("y", event.values[1].toDouble())
                    putDouble("z", event.values[2].toDouble())
                    putDouble("timestamp", System.currentTimeMillis().toDouble())
                }
                sendEvent("onAccelerometer", params)
            }
            override fun onAccuracyChanged(sensor: Sensor, accuracy: Int) {}
        }

        sensorManager.registerListener(sensorListener, sensor, delay)
        promise.resolve(true)
    }

    @ReactMethod
    fun startGyroscope(intervalMs: Int, promise: Promise) {
        val sensor = sensorManager.getDefaultSensor(Sensor.TYPE_GYROSCOPE)
            ?: return promise.reject("NO_SENSOR", "Gyroscope not available")
        // Similar implementation...
    }

    @ReactMethod
    fun stopSensor(type: String, promise: Promise) {
        sensorListener?.let { sensorManager.unregisterListener(it) }
        sensorListener = null
        promise.resolve(true)
    }

    @ReactMethod
    fun isSensorAvailable(type: String, promise: Promise) {
        val sensorType = when (type) {
            "accelerometer" -> Sensor.TYPE_ACCELEROMETER
            "gyroscope" -> Sensor.TYPE_GYROSCOPE
            "magnetometer" -> Sensor.TYPE_MAGNETIC_FIELD
            "barometer" -> Sensor.TYPE_PRESSURE
            else -> return promise.reject("UNKNOWN_SENSOR", "Unknown type: $type")
        }
        promise.resolve(sensorManager.getDefaultSensor(sensorType) != null)
    }

    private fun sendEvent(name: String, params: WritableMap) {
        reactApplicationContext
            .getJSModule(DeviceEventManagerModule.RCTDeviceEventEmitter::class.java)
            .emit(name, params)
    }

    @ReactMethod fun addListener(name: String) {}
    @ReactMethod fun removeListeners(count: Int) {}
}
```

---

### Q337. How do you pass large data efficiently between JS and Native?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Performance + Native Modules

**Answer:**
Passing large data across the bridge (old arch) or JSI (new arch) has overhead. Strategy depends on data type and size.

```kotlin
// Strategy 1: File path instead of base64 for binary data
// ❌ Passing 5MB image as base64 string
@ReactMethod
fun captureScreenBad(promise: Promise) {
    val bitmap = captureScreen()
    val bytes = bitmapToBytes(bitmap)
    promise.resolve(Base64.encodeToString(bytes, Base64.DEFAULT)) // 7MB base64 string!
}

// ✅ Write to file, return path
@ReactMethod
fun captureScreenGood(promise: Promise) {
    val bitmap = captureScreen()
    val file = File(reactApplicationContext.cacheDir, "screenshot_${System.currentTimeMillis()}.jpg")
    FileOutputStream(file).use { out ->
        bitmap.compress(Bitmap.CompressFormat.JPEG, 85, out)
    }
    promise.resolve(file.absolutePath) // small string!
}

// Strategy 2: Streaming large arrays in chunks
@ReactMethod
fun getLargeDataset(chunkSize: Int, promise: Promise) {
    // Return first chunk immediately + token for more
    val firstChunk = getLargeDataset().take(chunkSize)
    val token = generatePaginationToken()

    promise.resolve(Arguments.createMap().apply {
        val arr = Arguments.createArray()
        firstChunk.forEach { item ->
            arr.pushMap(Arguments.createMap().apply { putString("id", item.id) })
        }
        putArray("data", arr)
        putString("nextToken", token)
        putBoolean("hasMore", true)
    })
}

// Strategy 3: JSI SharedArrayBuffer (New Architecture)
// Direct memory sharing between JS and native — zero copy
// Supported in Hermes + New Architecture
// Use for: audio buffers, image data, large numeric arrays
```

---

### Q338. What is `LifecycleEventListener` in Android native modules?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Android Native Modules

**Answer:**
`LifecycleEventListener` lets a native module react to Android activity lifecycle events — resume, pause, destroy.

```kotlin
class BackgroundSyncModule(reactContext: ReactApplicationContext)
    : ReactContextBaseJavaModule(reactContext), LifecycleEventListener {

    override fun getName() = "BackgroundSync"
    private var syncJob: Job? = null

    // Register in init
    init { reactContext.addLifecycleEventListener(this) }

    // App comes to foreground (Activity.onResume)
    override fun onHostResume() {
        // Restart any paused operations
        startRealTimeSync()
        // Refresh data that may be stale from background
        syncPendingData()
    }

    // App goes to background (Activity.onPause)
    override fun onHostPause() {
        // Stop expensive operations
        stopRealTimeSync()
        // Flush any pending writes
        flushPendingWrites()
    }

    // App is destroyed (Activity.onDestroy — not guaranteed to be called)
    override fun onHostDestroy() {
        syncJob?.cancel()
        cleanupResources()
    }

    private fun startRealTimeSync() {
        syncJob = CoroutineScope(Dispatchers.IO).launch {
            while (isActive) {
                try { syncData() } catch (e: Exception) { /* log */ }
                delay(30_000) // sync every 30s while in foreground
            }
        }
    }

    private fun stopRealTimeSync() { syncJob?.cancel() }

    @ReactMethod
    fun manualSync(promise: Promise) {
        CoroutineScope(Dispatchers.IO).launch {
            try {
                syncData()
                promise.resolve(true)
            } catch (e: Exception) {
                promise.reject("SYNC_ERROR", e.message, e)
            }
        }
    }
}
```

---

### Q339. How do you implement a native module that uses Android WorkManager?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Background Tasks + Android

**Answer:**
```kotlin
// WorkManager provides guaranteed background execution even after app restart

// 1. Define the work
class DataSyncWorker(context: Context, params: WorkerParameters)
    : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        return try {
            val employees = fetchEmployeesFromServer()
            saveToLocalDB(employees)

            // Report progress
            setProgress(workDataOf("processed" to employees.size))

            Result.success(workDataOf("syncedCount" to employees.size))
        } catch (e: Exception) {
            if (runAttemptCount < 3) Result.retry()
            else Result.failure(workDataOf("error" to e.message))
        }
    }
}

// 2. Native module to schedule/cancel work
class WorkManagerModule(reactContext: ReactApplicationContext)
    : ReactContextBaseJavaModule(reactContext) {

    override fun getName() = "WorkManager"
    private val workManager = WorkManager.getInstance(reactApplicationContext)

    @ReactMethod
    fun scheduleSync(config: ReadableMap, promise: Promise) {
        val intervalHours = if (config.hasKey("intervalHours"))
            config.getInt("intervalHours").toLong() else 1L

        val constraints = Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)
            .setRequiresBatteryNotLow(true)
            .build()

        val syncRequest = PeriodicWorkRequestBuilder<DataSyncWorker>(
            intervalHours, TimeUnit.HOURS,
            15, TimeUnit.MINUTES  // flex interval
        )
            .setConstraints(constraints)
            .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 1, TimeUnit.HOURS)
            .setInputData(workDataOf("syncType" to "full"))
            .addTag("data_sync")
            .build()

        workManager.enqueueUniquePeriodicWork(
            "DataSync",
            ExistingPeriodicWorkPolicy.UPDATE,
            syncRequest
        )
        promise.resolve(syncRequest.id.toString())
    }

    @ReactMethod
    fun cancelSync(promise: Promise) {
        workManager.cancelUniqueWork("DataSync")
        promise.resolve(true)
    }

    @ReactMethod
    fun getSyncStatus(workId: String, promise: Promise) {
        workManager.getWorkInfoByIdLiveData(UUID.fromString(workId))
            .observeForever { info ->
                promise.resolve(Arguments.createMap().apply {
                    putString("state", info.state.name)
                    putString("id", workId)
                    info.progress.getInt("processed", -1).let { if (it >= 0) putInt("processed", it) }
                })
            }
    }
}
```

---

### Q340. How do you write a production-ready native module with full error handling?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Native Modules — Best Practices

**Answer:**
```kotlin
// Android — Production-grade native module pattern
class ProductionModule(reactContext: ReactApplicationContext)
    : ReactContextBaseJavaModule(reactContext), LifecycleEventListener {

    override fun getName() = MODULE_NAME

    // Error codes as constants — consistent with iOS and JS
    companion object {
        const val MODULE_NAME = "ProductionModule"
        const val E_NOT_INITIALIZED = "E_NOT_INITIALIZED"
        const val E_PERMISSION_DENIED = "E_PERMISSION_DENIED"
        const val E_NETWORK_ERROR = "E_NETWORK_ERROR"
        const val E_INVALID_ARGUMENT = "E_INVALID_ARGUMENT"
        const val E_OPERATION_FAILED = "E_OPERATION_FAILED"
    }

    private var isInitialized = false
    private val coroutineScope = CoroutineScope(Dispatchers.IO + SupervisorJob())

    init { reactContext.addLifecycleEventListener(this) }

    override fun getConstants() = mapOf(
        "MODULE_VERSION" to 2,
        "E_NOT_INITIALIZED" to E_NOT_INITIALIZED,
        "E_PERMISSION_DENIED" to E_PERMISSION_DENIED,
    )

    @ReactMethod
    fun initialize(config: ReadableMap, promise: Promise) {
        // Validate required config fields
        val apiKey = config.getString("apiKey")
        if (apiKey.isNullOrBlank()) {
            promise.reject(E_INVALID_ARGUMENT, "apiKey is required")
            return
        }

        coroutineScope.launch {
            try {
                initSDK(apiKey)
                isInitialized = true
                withContext(Dispatchers.Main) { promise.resolve(true) }
            } catch (e: SecurityException) {
                withContext(Dispatchers.Main) {
                    promise.reject(E_PERMISSION_DENIED, e.message, e)
                }
            } catch (e: Exception) {
                withContext(Dispatchers.Main) {
                    promise.reject(E_OPERATION_FAILED, "Initialization failed: ${e.message}", e)
                }
            }
        }
    }

    @ReactMethod
    fun performOperation(input: String, promise: Promise) {
        // Guard: check initialization
        if (!isInitialized) {
            promise.reject(E_NOT_INITIALIZED, "Module not initialized. Call initialize() first.")
            return
        }

        // Guard: validate input
        if (input.isBlank()) {
            promise.reject(E_INVALID_ARGUMENT, "Input cannot be empty")
            return
        }

        coroutineScope.launch {
            try {
                val result = doOperation(input)
                withContext(Dispatchers.Main) {
                    promise.resolve(Arguments.createMap().apply {
                        putString("result", result)
                        putDouble("timestamp", System.currentTimeMillis().toDouble())
                    })
                }
            } catch (e: IOException) {
                withContext(Dispatchers.Main) {
                    promise.reject(E_NETWORK_ERROR, "Network error: ${e.message}", e)
                }
            } catch (e: Exception) {
                withContext(Dispatchers.Main) {
                    promise.reject(E_OPERATION_FAILED, e.message, e)
                }
            }
        }
    }

    override fun onHostResume() {}
    override fun onHostPause() {}
    override fun onHostDestroy() {
        // Cancel all in-flight coroutines on destroy
        coroutineScope.cancel()
    }

    @ReactMethod fun addListener(name: String) {}
    @ReactMethod fun removeListeners(count: Int) {}
}
```

```js
// JavaScript — matching error handling
import { NativeModules } from 'react-native';
const { ProductionModule } = NativeModules;
const { E_NOT_INITIALIZED, E_PERMISSION_DENIED, E_NETWORK_ERROR } = ProductionModule;

export const ProductionAPI = {
    initialize: async (config) => {
        try {
            return await ProductionModule.initialize(config);
        } catch (error) {
            if (error.code === E_PERMISSION_DENIED) {
                throw new Error('Permission denied. Please grant the required permissions.');
            }
            throw error;
        }
    },

    performOperation: async (input) => {
        try {
            return await ProductionModule.performOperation(input);
        } catch (error) {
            switch (error.code) {
                case E_NOT_INITIALIZED:
                    throw new Error('Please call initialize() first');
                case E_NETWORK_ERROR:
                    throw new Error('Network unavailable. Please check your connection.');
                default:
                    // Log unexpected errors to Sentry
                    Sentry.captureException(error);
                    throw error;
            }
        }
    }
};
```

---

## Sections Overview (Q341–Q500)

| Section | Questions | Topics |
|---------|-----------|--------|
| Expo vs CLI | Q341–Q370 | Managed vs bare, EAS Build, Expo Go |
| Testing | Q371–Q420 | Unit, integration, e2e (Detox), mocking |
| Debugging | Q421–Q450 | Flipper, Hermes debugger, crash reporting |
| Storage & Permissions | Q451–Q480 | AsyncStorage, Keychain, permission flows |
| Miscellaneous | Q481–Q500 | Accessibility, internationalisation, misc APIs |

---

> 💡 **Tip for GitHub:** Add a `## Table of Contents` section at the top with anchor links to each question for easy navigation.

---

*Part 01 of 8 — [← Back to Part README](./README.md) · [← Main README](../README.md)*

## Section 9: Expo vs CLI (Q341–Q370)

---

### Q341. What is the difference between Expo Managed, Expo Bare, and React Native CLI?

**Difficulty:** 🟡 Medium | **Frequency:** Very High | **Category:** Expo vs CLI

**Answer:**

| | Expo Managed | Expo Bare | React Native CLI |
|--|-------------|-----------|-----------------|
| Native code visible | ❌ Hidden | ✅ Exposed | ✅ Exposed |
| Custom native modules | ❌ Not allowed | ✅ Allowed | ✅ Allowed |
| Expo SDK | ✅ Full access | ✅ Full access | ⚠️ Most work |
| OTA updates | ✅ Built-in | ✅ via EAS Update | ⚠️ Manual |
| Build service | ✅ EAS Build | ✅ EAS Build | ⚠️ Manual / CI |
| Flexibility | Low | High | Highest |
| Setup time | Minutes | Hours | Hours |

```bash
# Expo Managed — no ios/ or android/ folders
npx create-expo-app MyApp
# app.json configures native behaviour

# Expo Bare — has ios/ and android/ but uses Expo SDK
npx create-expo-app MyApp --template bare-minimum
# OR eject from managed:
npx expo prebuild

# React Native CLI — full native control
npx @react-native-community/cli@latest init MyApp
```

**When to use each:**
- **Managed:** Prototypes, demos, apps with standard features only, solo developer
- **Bare:** Production apps that need custom native code + Expo SDK convenience
- **CLI:** Existing native codebases, specific native SDK integrations, max control

**Follow-up:** Can you add a custom native module to a Managed Expo app? → No, managed workflow apps run inside the Expo Go sandbox which doesn't support arbitrary native code. You must eject to bare (run `npx expo prebuild`) to add custom native modules.

---

### Q342. What is `expo prebuild` and what does it do?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Expo

**Answer:**
`expo prebuild` generates the native `ios/` and `android/` directories from your `app.json` / `app.config.js` configuration and the Expo SDK. It's the bridge between managed and bare workflow.

```bash
# Generate native projects from app.json config
npx expo prebuild

# Clean and regenerate (discards manual native changes)
npx expo prebuild --clean

# Platform-specific
npx expo prebuild --platform ios
npx expo prebuild --platform android
```

```json
// app.json — configures what prebuild generates
{
  "expo": {
    "name": "My ERP App",
    "slug": "my-erp-app",
    "version": "1.0.0",
    "ios": {
      "bundleIdentifier": "com.yourcompany.erp",
      "buildNumber": "1",
      "infoPlist": {
        "NSCameraUsageDescription": "Scan QR codes for attendance",
        "NSFaceIDUsageDescription": "Authenticate with Face ID"
      },
      "entitlements": {
        "com.apple.developer.associated-domains": ["applinks:yourapp.com"]
      }
    },
    "android": {
      "package": "com.yourcompany.erp",
      "versionCode": 1,
      "permissions": ["CAMERA", "ACCESS_FINE_LOCATION"],
      "googleServicesFile": "./google-services.json"
    },
    "plugins": [
      "expo-camera",
      "expo-location",
      ["expo-notifications", { "icon": "./assets/notification-icon.png" }]
    ]
  }
}
```

**What prebuild generates:**
- `ios/` — Xcode project, Podfile, Info.plist, entitlements
- `android/` — Gradle files, AndroidManifest.xml, build.gradle
- Installs CocoaPods automatically
- Applies config plugins from `app.json`

**Important:** Running `prebuild --clean` deletes and regenerates native folders — **do not** keep manual native changes that aren't in a config plugin or they'll be lost.

---

### Q343. What are Expo Config Plugins?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Expo Config Plugins

**Answer:**
Config plugins are JavaScript functions that modify native project files during `expo prebuild`. They're the Expo way to add native configuration without manually editing Xcode/Android Studio files.

```js
// app.config.js — using existing plugins
export default {
  expo: {
    plugins: [
      // Simple plugin (no config)
      "expo-camera",

      // Plugin with options
      ["expo-notifications", {
        icon: "./assets/notif-icon.png",
        color: "#ffffff",
        sounds: ["./assets/notification.wav"],
      }],

      // Custom inline config plugin
      (config) => {
        // Modify iOS Info.plist
        config.modResults.ios.infoPlist = {
          ...config.modResults.ios.infoPlist,
          NSBluetoothAlwaysUsageDescription: "Used for BLE attendance scanning",
        };
        return config;
      },
    ],
  },
};
```

```js
// Writing a custom config plugin (plugins/withCustomAndroidPermission.js)
const { withAndroidManifest } = require('@expo/config-plugins');

const withBluetoothPermissions = (config) => {
  return withAndroidManifest(config, async (config) => {
    const androidManifest = config.modResults;
    const mainApplication = androidManifest.manifest.application[0];

    // Add uses-permission to AndroidManifest.xml
    const permissions = androidManifest.manifest['uses-permission'] || [];
    const needed = [
      'android.permission.BLUETOOTH',
      'android.permission.BLUETOOTH_SCAN',
      'android.permission.BLUETOOTH_CONNECT',
    ];

    needed.forEach(perm => {
      if (!permissions.some(p => p.$['android:name'] === perm)) {
        permissions.push({ $: { 'android:name': perm } });
      }
    });

    androidManifest.manifest['uses-permission'] = permissions;
    return config;
  });
};

module.exports = withBluetoothPermissions;
```

```js
// app.config.js — use custom plugin
const withBluetoothPermissions = require('./plugins/withCustomAndroidPermission');

export default {
  expo: {
    plugins: [withBluetoothPermissions],
  },
};
```

**Available mod types:**
- `withInfoPlist` — iOS Info.plist
- `withAndroidManifest` — AndroidManifest.xml
- `withAppBuildGradle` — app/build.gradle
- `withProjectBuildGradle` — root build.gradle
- `withPodfile` — iOS Podfile
- `withMainApplication` — MainApplication.java/kt
- `withMainActivity` — MainActivity.java/kt

---

### Q344. What is EAS Build and how does it work?

**Difficulty:** 🟡 Medium | **Frequency:** Very High | **Category:** EAS Build

**Answer:**
EAS Build (Expo Application Services Build) is Expo's cloud build service. It builds iOS and Android apps on Expo's servers — no need for a Mac to build iOS.

```bash
# Install EAS CLI
npm install -g eas-cli

# Login
eas login

# Configure (creates eas.json)
eas build:configure

# Build for development (debug build with dev client)
eas build --platform android --profile development
eas build --platform ios --profile development

# Build for TestFlight / internal testing
eas build --platform ios --profile preview

# Build for production (Play Store / App Store)
eas build --platform all --profile production

# Check build status
eas build:list
```

```json
// eas.json — build profiles
{
  "cli": { "version": ">= 5.0.0" },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "android": { "buildType": "apk" },
      "ios": { "simulator": false }
    },
    "preview": {
      "distribution": "internal",
      "android": { "buildType": "apk" },
      "ios": {
        "distribution": "internal",
        "enterpriseProvisioning": "adhoc"
      }
    },
    "production": {
      "android": {
        "buildType": "app-bundle"  // .aab for Play Store
      },
      "ios": {
        "distribution": "store"   // .ipa for App Store
      },
      "env": {
        "APP_ENV": "production",
        "API_URL": "https://api.yourapp.com"
      }
    }
  },
  "submit": {
    "production": {
      "android": {
        "serviceAccountKeyPath": "./pc-api-key.json",
        "track": "internal"
      },
      "ios": {
        "appleId": "developer@yourcompany.com",
        "ascAppId": "1234567890",
        "appleTeamId": "XXXXXXXXXX"
      }
    }
  }
}
```

**How it works:**
1. EAS CLI uploads your source code + secrets to Expo's servers
2. A build worker clones your repo in an isolated environment
3. iOS: macOS runner with Xcode → produces `.ipa`
4. Android: Ubuntu runner with Gradle → produces `.apk` or `.aab`
5. Signed artifact available for download or auto-submitted to stores

---

### Q345. What is EAS Update and how does it differ from EAS Build?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** EAS Update / OTA

**Answer:**
| | EAS Build | EAS Update |
|--|-----------|-----------|
| What it produces | Native binary (.ipa/.apk/.aab) | JS bundle + assets |
| When it runs | When native code changes | When JS/assets change |
| Delivery | App Store / Play Store | Over-the-air (OTA) |
| User action | Full app update (install) | Silent background update |
| Time to users | Days (store review) | Minutes |
| What it can change | Everything | JS, assets only — NOT native |

```bash
# Install EAS Update
npx expo install expo-updates

# Configure
eas update:configure

# Publish an update
eas update --branch production --message "Fix login bug"

# Publish to specific branch
eas update --branch staging --message "New feature for QA"
```

```js
// app.json — configure updates
{
  "expo": {
    "updates": {
      "url": "https://u.expo.dev/your-project-id",
      "checkAutomatically": "ON_LOAD",  // ON_LOAD, ON_ERROR_RECOVERY, WIFI_ONLY
      "fallbackToCacheTimeout": 3000    // ms to wait before using cached bundle
    },
    "runtimeVersion": {
      "policy": "sdkVersion"  // fingerprint, sdkVersion, nativeVersion, appVersion
    }
  }
}
```

```js
// Manual update check in app
import * as Updates from 'expo-updates';

const checkForUpdate = async () => {
  if (__DEV__) return; // no OTA in development

  try {
    const update = await Updates.checkForUpdateAsync();
    if (update.isAvailable) {
      await Updates.fetchUpdateAsync();
      await Updates.reloadAsync(); // restart with new bundle
    }
  } catch (error) {
    console.error('OTA update failed:', error);
  }
};

// Soft update — notify user
const handleUpdate = async () => {
  const update = await Updates.checkForUpdateAsync();
  if (update.isAvailable) {
    Alert.alert(
      'Update available',
      'A new version is ready. Reload now?',
      [
        { text: 'Later' },
        { text: 'Reload', onPress: async () => {
          await Updates.fetchUpdateAsync();
          Updates.reloadAsync();
        }},
      ]
    );
  }
};
```

---

### Q346. What is Expo Go and what are its limitations?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Expo Go

**Answer:**
Expo Go is a free app on the App Store and Play Store that lets you run your Expo project on a device without building a binary. You scan a QR code and the app loads instantly.

```bash
# Start dev server for Expo Go
npx expo start
# Scan QR code with Expo Go app on device
```

**Limitations of Expo Go:**

```
1. No custom native modules
   → Only Expo SDK modules included in Expo Go are available
   → react-native-razorpay, custom native modules WON'T work

2. Fixed Expo SDK version
   → Expo Go ships with a specific SDK version
   → If you use SDK 51, you need Expo Go that supports SDK 51

3. No custom native build configuration
   → Can't test custom Info.plist keys, custom permissions, etc.

4. Can't simulate push notifications
   → Expo Go uses its own push token, not your app's

5. Not suitable for production debugging
   → Production builds behave differently (Hermes enabled, no debugger)

6. Can't test in-app purchases
   → IAP requires a signed app build

7. Limited background execution
   → Background tasks not fully supported

8. No custom URL schemes
   → Deep linking works differently in Expo Go vs standalone
```

**Alternative: Expo Dev Client (Development Build)**
```bash
# Create a development build — your own app + Expo dev tools
eas build --profile development --platform android
# Install on device → opens your custom dev client
# Supports ALL your native modules + Expo dev experience
```

---

### Q347. What is a Development Build in Expo?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Expo Dev Client

**Answer:**
A Development Build is a custom version of Expo Go that includes your app's native code. It replaces Expo Go for development — giving you the Expo dev experience (hot reload, DevTools) while supporting your custom native modules.

```bash
# Install expo-dev-client
npx expo install expo-dev-client

# Build dev client (once, when native deps change)
eas build --profile development --platform android
# OR locally:
npx expo run:android  # builds and installs debug APK

# Start dev server (dev client connects to it)
npx expo start --dev-client
# OR
npx expo start  # auto-detects dev client

# Install the APK on device/emulator, then scan QR
```

```json
// eas.json — development profile
{
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "android": {
        "buildType": "apk",
        "gradleCommand": ":app:assembleDebug"
      },
      "ios": {
        "buildConfiguration": "Debug",
        "simulator": true   // for simulator testing
      }
    }
  }
}
```

**Workflow:**
```
1. Add native dependency → npx expo install some-native-lib
2. Build new dev client → eas build --profile development
3. Install on device (once per native change)
4. Develop JS freely → fast refresh, no rebuild needed
5. Add another native dep? → rebuild dev client
```

**Key difference from Expo Go:**
- Expo Go = Expo's generic sandbox (fixed modules)
- Dev Client = YOUR app's sandbox (your custom native modules)

---

### Q348. How do you handle environment variables in Expo?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Expo Configuration

**Answer:**
```bash
# Method 1: .env files (Expo SDK 49+)
# .env
EXPO_PUBLIC_API_URL=https://api.dev.yourapp.com
EXPO_PUBLIC_RAZORPAY_KEY=rzp_test_xxxxxxxx

# .env.production
EXPO_PUBLIC_API_URL=https://api.yourapp.com
EXPO_PUBLIC_RAZORPAY_KEY=rzp_live_xxxxxxxx
```

```js
// Access with EXPO_PUBLIC_ prefix (bundled into JS — visible to users)
const apiUrl = process.env.EXPO_PUBLIC_API_URL;
const razorpayKey = process.env.EXPO_PUBLIC_RAZORPAY_KEY;

// ⚠️ NEVER put secrets in EXPO_PUBLIC_ variables
// They're embedded in the JS bundle — users can extract them
```

```js
// Method 2: app.config.js with dynamic config (recommended for secrets)
// app.config.js
export default ({ config }) => ({
  ...config,
  extra: {
    apiUrl: process.env.API_URL || 'http://localhost:3000',
    environment: process.env.APP_ENV || 'development',
    // Server-side secret — NOT included in JS bundle
    // sentryDsn: process.env.SENTRY_DSN  ← use EAS Secrets instead
  },
  hooks: {
    postPublish: [
      {
        file: 'sentry-expo/upload-sourcemaps',
        config: {
          organization: 'your-org',
          project: 'your-project',
          authToken: process.env.SENTRY_AUTH_TOKEN, // build-time only
        },
      },
    ],
  },
});

// Read in app:
import Constants from 'expo-constants';
const { apiUrl } = Constants.expoConfig.extra;
```

```bash
# Method 3: EAS Secrets (truly secret — injected at build time)
eas secret:create --scope project --name SENTRY_DSN --value "https://xxx@sentry.io/xxx"
eas secret:create --scope project --name API_KEY --value "secret-key-value"

# Secrets are available as process.env.SENTRY_DSN in eas.json env block
# NOT in the JS bundle — only available during build
```

```json
// eas.json — reference secrets in build env
{
  "build": {
    "production": {
      "env": {
        "SENTRY_DSN": "$SENTRY_DSN",      // from EAS Secrets
        "APP_ENV": "production",
        "API_URL": "https://api.yourapp.com"
      }
    }
  }
}
```

---

### Q349. What is `app.json` vs `app.config.js` in Expo?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Expo Configuration

**Answer:**
```json
// app.json — static configuration (JSON, no logic)
{
  "expo": {
    "name": "My App",
    "version": "1.0.0",
    "icon": "./assets/icon.png",
    "platforms": ["ios", "android"],
    "ios": {
      "bundleIdentifier": "com.company.app",
      "buildNumber": "1"
    },
    "android": {
      "package": "com.company.app",
      "versionCode": 1
    }
  }
}
```

```js
// app.config.js — dynamic configuration (JavaScript, supports logic)
// Replaces app.json — more powerful
import 'dotenv/config';

export default ({ config }) => {
  // Spread the base config from app.json if it exists
  const isProduction = process.env.APP_ENV === 'production';

  return {
    ...config,
    name: isProduction ? 'My App' : 'My App (Dev)',
    ios: {
      ...config.ios,
      bundleIdentifier: isProduction
        ? 'com.company.app'
        : 'com.company.app.dev',
      buildNumber: String(process.env.BUILD_NUMBER || '1'),
    },
    android: {
      ...config.android,
      package: isProduction ? 'com.company.app' : 'com.company.app.dev',
      versionCode: parseInt(process.env.VERSION_CODE || '1'),
      googleServicesFile: isProduction
        ? './google-services-prod.json'
        : './google-services-dev.json',
    },
    extra: {
      apiUrl: process.env.API_URL,
      environment: process.env.APP_ENV || 'development',
      eas: { projectId: 'your-eas-project-id' },
    },
    updates: {
      url: 'https://u.expo.dev/your-project-id',
    },
    runtimeVersion: {
      policy: process.env.APP_ENV === 'production' ? 'fingerprint' : 'sdkVersion',
    },
  };
};
```

**Key differences:**
- `app.json` = static, committed to git, no secrets
- `app.config.js` = dynamic, can read env vars, can have logic, can merge from multiple sources

---

### Q350. What is the Expo SDK and what modules does it include?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Expo SDK

**Answer:**
The Expo SDK is a collection of well-maintained React Native libraries that cover common app functionality. Each package is prefixed with `expo-`.

```bash
# Install Expo SDK packages (use npx expo install — ensures compatible version)
npx expo install expo-camera        # Camera access
npx expo install expo-location      # GPS + geolocation
npx expo install expo-notifications # Push + local notifications
npx expo install expo-auth-session  # OAuth (Google, Facebook, Apple)
npx expo install expo-secure-store  # Keychain / Keystore wrapper
npx expo install expo-file-system   # File I/O
npx expo install expo-image-picker  # Photo library + camera picker
npx expo install expo-av            # Audio + video playback
npx expo install expo-font          # Custom font loading
npx expo install expo-splash-screen # Splash screen control
npx expo install expo-updates       # OTA updates
npx expo install expo-constants     # App constants (version, etc.)
npx expo install expo-device        # Device info
npx expo install expo-haptics       # Haptic feedback
npx expo install expo-clipboard     # Copy/paste
npx expo install expo-share         # Native share sheet
npx expo install expo-web-browser   # In-app browser (OAuth)
npx expo install expo-linear-gradient # Gradient views
npx expo install expo-blur          # Blur view
npx expo install expo-contacts      # Contacts access
npx expo install expo-calendar      # Calendar events
npx expo install expo-sensors       # Accelerometer, gyroscope
npx expo install expo-barcode-scanner # QR code scanner
```

```js
// Example: expo-secure-store (Keychain/Keystore wrapper)
import * as SecureStore from 'expo-secure-store';

await SecureStore.setItemAsync('authToken', 'jwt-token-here');
const token = await SecureStore.getItemAsync('authToken');
await SecureStore.deleteItemAsync('authToken');

// Example: expo-notifications (local)
import * as Notifications from 'expo-notifications';

await Notifications.scheduleNotificationAsync({
    content: {
        title: "Attendance Reminder",
        body: "Don't forget to mark your attendance!",
        data: { screen: 'attendance' },
    },
    trigger: { hour: 9, minute: 0, repeats: true }, // every day at 9 AM
});
```

**Why use `npx expo install` not `npm install`?**
→ `npx expo install` picks the exact version compatible with your Expo SDK version. `npm install` may pick an incompatible version causing hard-to-debug issues.

---

### Q351. How do you migrate from Expo Managed to Bare workflow?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Expo Migration

**Answer:**
```bash
# The official migration command
npx expo prebuild

# This generates ios/ and android/ directories
# Your app.json config is translated to native config files

# After prebuild:
# Install pods (iOS)
cd ios && pod install && cd ..

# Verify the build works
npx expo run:ios
npx expo run:android

# Clean prebuild (regenerate from scratch)
npx expo prebuild --clean
```

**Post-migration checklist:**
```bash
# 1. Add ios/ and android/ to .gitignore? No — commit them!
#    (They're now part of your project, not generated artifacts)
#    Unless you use CNG (Continuous Native Generation) with prebuild --clean

# 2. Update CI/CD pipeline
#    Old: EAS Build handles everything
#    New: May need fastlane, or keep using EAS Build (still works with bare)

# 3. Test all Expo SDK modules still work
#    npx expo install doesn't change — same commands
#    But now native linking is handled by autolinking + pod install

# 4. Add custom native module
#    Now you can add NativeModules in ios/ and android/
#    No more "this module requires native code" errors

# 5. Update README with new setup steps
#    cd ios && pod install (iOS devs)
#    No changes for Android (gradle handles it)
```

**When NOT to migrate:**
- You don't need custom native code
- Team has no iOS/Android native experience
- You rely on Expo Go for quick client demos

---

### Q352. What is `expo-dev-client` vs `expo-go`?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Expo Dev Tools

**Answer:**

| | Expo Go | expo-dev-client |
|--|---------|----------------|
| What it is | Expo's generic sandbox app | Your custom dev sandbox |
| Custom native modules | ❌ Not supported | ✅ Fully supported |
| Native dependencies | Fixed (Expo SDK only) | Whatever you install |
| Rebuild needed | Never (it's a store app) | When native deps change |
| Distribution | App Store / Play Store | Internal (side-load) |
| Dev menu | ✅ Expo dev menu | ✅ Expo dev menu |
| Fast refresh | ✅ | ✅ |
| Remote debugging | ✅ | ✅ |

```bash
# Setup expo-dev-client
npx expo install expo-dev-client

# Build dev client (once)
# Local build (faster, requires Xcode/Android Studio)
npx expo run:ios --device
npx expo run:android --device

# Cloud build (no Xcode/Android Studio required)
eas build --profile development --platform ios

# Start dev server (dev client connects automatically)
npx expo start --dev-client
```

```js
// expo-dev-client adds a custom dev menu with:
// - Reload JavaScript
// - Open JS Debugger
// - Toggle Performance Monitor
// - Toggle Element Inspector
// - Your own custom dev menu items

// Add custom dev menu items
import { registerDevMenuItems } from 'expo-dev-client';

registerDevMenuItems([
    {
        name: 'Clear AsyncStorage',
        callback: async () => {
            await AsyncStorage.clear();
            alert('AsyncStorage cleared!');
        },
    },
    {
        name: 'Reset to Onboarding',
        callback: () => navigationRef.current?.reset({ index: 0, routes: [{ name: 'Onboarding' }] }),
    },
]);
```

---

### Q353. What is EAS Submit and how do you automate app store uploads?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** EAS Submit

**Answer:**
EAS Submit automates uploading your built `.ipa` / `.aab` to App Store Connect and Google Play Console.

```bash
# Submit latest build
eas submit --platform ios
eas submit --platform android

# Submit a specific build by ID
eas submit --platform ios --id <build-id>

# Submit production build (auto-submit after build)
eas build --platform all --profile production --auto-submit
```

```json
// eas.json — submit configuration
{
  "submit": {
    "production": {
      "ios": {
        "appleId": "developer@yourcompany.com",
        "ascAppId": "1234567890",        // App Store Connect App ID
        "appleTeamId": "XXXXXXXXXX",
        "bundleIdentifier": "com.company.app"
      },
      "android": {
        "serviceAccountKeyPath": "./pc-api-key.json",  // Google Play API key
        "track": "internal",            // internal, alpha, beta, production
        "releaseStatus": "completed"    // draft, completed
      }
    }
  }
}
```

```bash
# Apple credentials setup
eas credentials --platform ios
# EAS manages provisioning profiles and certificates automatically

# Google Play setup:
# 1. Create service account in Google Play Console
# 2. Download JSON key
# 3. Grant service account access in Play Console
# 4. Path it in eas.json serviceAccountKeyPath

# Full automated pipeline:
# 1. git push to main
# 2. CI/CD detects push
# 3. eas build --profile production --auto-submit
# 4. Build → sign → upload to store → review
```

---

### Q354. What is the `runtimeVersion` in EAS Update?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** EAS Update

**Answer:**
`runtimeVersion` is a string that identifies the **native binary compatibility** of an OTA update. An update is only delivered to devices whose native binary has the matching `runtimeVersion`. This prevents delivering a JS update that requires a newer native module to an old binary.

```json
// app.json — runtimeVersion policies
{
  "expo": {
    "runtimeVersion": {
      "policy": "sdkVersion"
      // "sdkVersion"     — uses Expo SDK version (e.g., "50.0.0")
      //                    update delivered to all devices with same SDK
      // "appVersion"     — uses app.json version (e.g., "1.2.0")
      //                    change version to break compatibility
      // "nativeVersion"  — concatenates iOS buildNumber + Android versionCode
      // "fingerprint"    — hash of all native dependencies (most precise)
      //                    automatically changes when any native dep changes
    }
  }
}
```

```bash
# fingerprint policy — recommended for production
# Computes a hash of:
# - package.json native dependencies
# - android/ and ios/ native files
# - expo-modules configuration
# - Any native code change → new fingerprint → old binaries won't get update

# Set explicit runtime version
# (useful for coordinating updates across a staged rollout)
eas update --branch production --runtime-version "1.2.0"
```

```
Timeline example with fingerprint policy:

Build 1.0 (fingerprint: abc123)
    ↓
OTA Update A → targets abc123 → delivered to Build 1.0 ✅
OTA Update B → targets abc123 → delivered to Build 1.0 ✅

New native dep added → rebuild
Build 1.1 (fingerprint: xyz789)
    ↓
OTA Update C → targets xyz789 → NOT delivered to Build 1.0 ✅ (correct)
                               → delivered to Build 1.1 ✅
```

---

### Q355. How do you handle multiple environments (dev/staging/prod) with Expo?

**Difficulty:** 🔴 Hard | **Frequency:** Very High | **Category:** Expo Configuration

**Answer:**
```js
// app.config.js — environment-aware config
const ENV = process.env.APP_ENV || 'development';

const envConfig = {
  development: {
    name: 'My App (Dev)',
    bundleId: 'com.company.app.dev',
    package: 'com.company.app.dev',
    apiUrl: 'http://localhost:3000',
    icon: './assets/icon-dev.png',
  },
  staging: {
    name: 'My App (Staging)',
    bundleId: 'com.company.app.staging',
    package: 'com.company.app.staging',
    apiUrl: 'https://api.staging.yourapp.com',
    icon: './assets/icon-staging.png',
  },
  production: {
    name: 'My App',
    bundleId: 'com.company.app',
    package: 'com.company.app',
    apiUrl: 'https://api.yourapp.com',
    icon: './assets/icon.png',
  },
};

const env = envConfig[ENV] || envConfig.development;

export default {
  expo: {
    name: env.name,
    slug: 'my-app',
    version: '1.0.0',
    icon: env.icon,
    ios: {
      bundleIdentifier: env.bundleId,
    },
    android: {
      package: env.package,
    },
    extra: {
      apiUrl: env.apiUrl,
      environment: ENV,
    },
    updates: {
      url: 'https://u.expo.dev/your-project-id',
    },
  },
};
```

```json
// eas.json — build profiles per environment
{
  "build": {
    "development": {
      "developmentClient": true,
      "env": { "APP_ENV": "development" }
    },
    "staging": {
      "distribution": "internal",
      "env": {
        "APP_ENV": "staging",
        "API_URL": "https://api.staging.yourapp.com"
      }
    },
    "production": {
      "env": {
        "APP_ENV": "production",
        "API_URL": "https://api.yourapp.com"
      }
    }
  }
}
```

```bash
# Build for specific environment
APP_ENV=staging eas build --profile staging --platform android
eas build --profile production --platform all

# Local development with environment
APP_ENV=staging npx expo start
```

---

### Q356. What is Expo's Continuous Native Generation (CNG)?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Expo Architecture

**Answer:**
CNG is the practice of never committing `ios/` and `android/` folders — instead regenerating them via `expo prebuild` whenever needed. The native directories are treated as build artifacts, not source.

```bash
# .gitignore — CNG setup
/ios
/android
# These are generated — not committed

# Regenerate native projects:
npx expo prebuild --clean   # always clean for CNG
cd ios && pod install && cd ..

# Typical CNG workflow:
# 1. Developer changes app.config.js or adds npm package
# 2. CI/CD runs: npx expo prebuild --clean
# 3. CI/CD runs: eas build (uses freshly generated native)
```

**Benefits of CNG:**
```
✅ No merge conflicts in Xcode/Android Studio files
✅ Native config is always derived from app.config.js (single source of truth)
✅ Config plugins handle all native changes declaratively
✅ Easy to upgrade Expo SDK (just change version, regenerate)
✅ Less knowledge of native required — everything in JS config
```

**Drawbacks:**
```
❌ All native config MUST be expressible as config plugins
❌ Complex native changes harder to implement without writing plugins
❌ Prebuild takes time in CI
❌ Can't make one-off manual native tweaks (they'd be overwritten)
```

```js
// When CNG breaks down — config plugin needed for:
// 1. Custom build.gradle changes
// 2. Custom Podfile changes
// 3. Custom AndroidManifest entries
// 4. Custom Info.plist entries
// These MUST be expressed as config plugins, not manual edits
```

---

### Q357. How do you add a native module (non-Expo) to an Expo bare app?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Expo Bare + Native Modules

**Answer:**
```bash
# 1. Install the package
npm install react-native-razorpay

# 2. Auto-linking handles most packages
# Android: nothing needed — Gradle auto-links
# iOS: run pod install
cd ios && pod install && cd ..

# 3. Rebuild (auto-linking changes native code)
# Dev client rebuild needed:
eas build --profile development --platform all
# OR locally:
npx expo run:ios
npx expo run:android
```

```js
// Verify the module is linked
// Android: android/settings.gradle should include the module
// iOS: ios/Podfile.lock should list the pod

// If auto-linking doesn't work (older libraries):
// react-native link react-native-old-module (deprecated but sometimes needed)
```

```bash
# Packages that need config plugins (app.json entry):
# expo-camera, expo-notifications, etc. → add to plugins array in app.json
# Community packages with config plugins: react-native-permissions, etc.

# Manual permissions (without config plugin):
# Android: edit android/app/src/main/AndroidManifest.xml directly
# iOS: edit ios/YourApp/Info.plist directly

# Example: add permission manually (bare workflow)
# ios/YourApp/Info.plist:
# <key>NSLocationWhenInUseUsageDescription</key>
# <string>Track attendance location</string>
```

```js
// Verify native module is accessible
import { NativeModules } from 'react-native';
console.log('Razorpay linked:', !!NativeModules.RazorpayModule);

// If undefined → check pod install / gradle sync ran after install
```

---

### Q358. What is `npx expo install` vs `npm install`?

**Difficulty:** 🟢 Easy | **Frequency:** High | **Category:** Expo

**Answer:**
```bash
# npm install — picks latest version (may be incompatible)
npm install expo-camera
# Installs latest expo-camera — may require newer Expo SDK than you have

# npx expo install — picks version compatible with YOUR Expo SDK
npx expo install expo-camera
# Installs expo-camera@14.0.3 (or whatever matches your SDK 50, for example)

# npx expo install can also fix mismatched versions:
npx expo install --fix
# Checks all expo-* packages, downgrades/upgrades to compatible versions
```

**Why it matters:**
```
Expo SDK 50 → requires expo-camera@14.x
If you npm install → gets expo-camera@15.x (requires SDK 51)
Result: runtime crash or build error about incompatible modules

npx expo install → gets expo-camera@14.0.x automatically ✅
```

```bash
# Check for version mismatches
npx expo-doctor
# Reports: incompatible package versions, deprecated config, etc.

# Audit all dependencies
npx expo install --check
# Lists packages that need updating to be compatible

# Fix all at once
npx expo install --fix
```

---

### Q359. How do you set up CI/CD with EAS Build?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** EAS Build + CI/CD

**Answer:**
```yaml
# GitHub Actions workflow: .github/workflows/build.yml

name: EAS Build & Submit
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-android:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Setup EAS
        uses: expo/expo-github-action@v8
        with:
          eas-version: latest
          token: ${{ secrets.EXPO_TOKEN }}

      - name: Build (PR — development APK for testing)
        if: github.event_name == 'pull_request'
        run: eas build --platform android --profile development --non-interactive

      - name: Build & Submit (main — production)
        if: github.ref == 'refs/heads/main'
        run: |
          eas build --platform android --profile production --non-interactive
          eas submit --platform android --profile production --non-interactive
        env:
          APP_ENV: production
          GOOGLE_SERVICES_JSON: ${{ secrets.GOOGLE_SERVICES_JSON }}

  build-ios:
    runs-on: ubuntu-latest  # EAS Build handles iOS on its own macOS runner
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - uses: expo/expo-github-action@v8
        with:
          eas-version: latest
          token: ${{ secrets.EXPO_TOKEN }}
      - name: Build iOS production
        if: github.ref == 'refs/heads/main'
        run: eas build --platform ios --profile production --non-interactive --auto-submit
```

```bash
# Required secrets in GitHub repo settings:
# EXPO_TOKEN — from expo.dev → Account Settings → Access Tokens
# GOOGLE_SERVICES_JSON — base64 encoded google-services.json
# (iOS credentials managed by EAS automatically)
```

---

### Q360. What is the difference between `expo start`, `expo run:android`, and `eas build`?

**Difficulty:** 🟢 Easy | **Frequency:** High | **Category:** Expo Commands

**Answer:**

| Command | What it does | When to use |
|---------|-------------|-------------|
| `npx expo start` | Starts Metro bundler, serves JS bundle | Daily development (with Expo Go or dev client) |
| `npx expo run:android` | Local Gradle build + install debug APK on device | Test native changes locally |
| `npx expo run:ios` | Local Xcode build + install on simulator/device | Test native changes locally on Mac |
| `eas build` | Cloud build on Expo's servers | Production builds, iOS builds without Mac |
| `eas build --profile development` | Cloud debug build | Share dev build with team |

```bash
# expo start — pure JS dev, no native build
npx expo start          # Expo Go or dev client
npx expo start --tunnel # Use ngrok tunnel (for physical device on different network)
npx expo start --offline # Disable manifest network checks

# expo run — local native build (needs Xcode/Android Studio)
npx expo run:android
npx expo run:android --device        # specific physical device
npx expo run:ios --simulator         # any simulator
npx expo run:ios --device "iPhone 15" # specific simulator

# eas build — cloud build
eas build --platform android --profile production
eas build --platform ios --profile preview --non-interactive
eas build --platform all --profile production --auto-submit

# Key: expo start/run are for DEVELOPMENT
#      eas build is for DISTRIBUTION (internal/store)
```

---

### Q361. How do you manage app signing in Expo?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** App Signing + Expo

**Answer:**
```bash
# EAS Credentials — manages all signing automatically
eas credentials --platform ios
eas credentials --platform android

# iOS credentials EAS manages:
# - Distribution certificate (for App Store / AdHoc)
# - Push notification certificate
# - Provisioning profiles (development, adhoc, distribution)
# - App Store Connect API key

# Android credentials EAS manages:
# - Keystore (upload and app signing key)
# - SHA-1 fingerprints (for Firebase, Google APIs)
```

```json
// eas.json — credential management
{
  "build": {
    "production": {
      "ios": {
        "credentialsSource": "remote",  // EAS manages credentials
        // OR:
        // "credentialsSource": "local"   // you manage credentials
      },
      "android": {
        "credentialsSource": "remote"
      }
    }
  }
}
```

```bash
# iOS — letting EAS manage signing
eas build --platform ios --profile production
# EAS asks: "Generate new certificate?" → Yes (first time)
# EAS creates + stores certificate in its credentials store
# Future builds automatically use stored certificate

# Android — EAS generates keystore on first build
eas build --platform android --profile production
# Keystore stored in EAS — download backup!
eas credentials --platform android
# → Download Keystore → save backup in secure location

# CRITICAL: Back up your Android keystore
# If lost, you CANNOT update your app on Play Store
# Store in: 1Password / AWS Secrets Manager / company vault
```

```bash
# Bring your own credentials (if you already have them)
# iOS: provide .p12 certificate + provisioning profile
# Android: provide existing .jks keystore

eas credentials --platform android
# → Import existing Keystore
# → Provide path to .jks file + alias + passwords
```

---

### Q362. What is Expo Router and how does it compare to React Navigation?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Expo Router

**Answer:**
Expo Router is a file-based routing system for Expo/React Native apps, inspired by Next.js. It uses the file system as the source of truth for navigation structure.

```
app/
├── _layout.tsx          ← Root layout (NavigationContainer equivalent)
├── index.tsx            ← Route: /
├── (auth)/
│   ├── _layout.tsx      ← Auth stack layout
│   ├── login.tsx        ← Route: /login
│   └── register.tsx     ← Route: /register
├── (tabs)/
│   ├── _layout.tsx      ← Tab navigator layout
│   ├── home.tsx         ← Route: /home (Tab 1)
│   ├── employees.tsx    ← Route: /employees (Tab 2)
│   └── profile.tsx      ← Route: /profile (Tab 3)
├── employee/
│   └── [id].tsx         ← Route: /employee/:id (dynamic)
└── +not-found.tsx       ← 404 handler
```

```tsx
// app/(tabs)/_layout.tsx — tab navigator
import { Tabs } from 'expo-router';

export default function TabLayout() {
    return (
        <Tabs screenOptions={{ tabBarActiveTintColor: '#6200EE' }}>
            <Tabs.Screen name="home" options={{ title: 'Home' }} />
            <Tabs.Screen name="employees" options={{ title: 'Employees' }} />
            <Tabs.Screen name="profile" options={{ title: 'Profile' }} />
        </Tabs>
    );
}

// app/employee/[id].tsx — dynamic route
import { useLocalSearchParams } from 'expo-router';
export default function EmployeeDetail() {
    const { id } = useLocalSearchParams<{ id: string }>();
    return <EmployeeDetailScreen employeeId={id} />;
}

// Navigation (like React Navigation navigate)
import { router, Link } from 'expo-router';
router.push('/employee/123');
router.replace('/login');
router.back();

<Link href="/employee/123">View Employee</Link>
<Link href={{ pathname: '/employee/[id]', params: { id: '123' } }}>View Employee</Link>
```

**Expo Router vs React Navigation:**
| | Expo Router | React Navigation |
|--|------------|-----------------|
| Config style | File-based (like Next.js) | Code-based (explicit) |
| Deep linking | Automatic (from file structure) | Manual config |
| Web support | ✅ Built-in | ⚠️ Extra setup |
| Type safety | ✅ TypeScript paths | Partial |
| Flexibility | Less | More |
| Learning curve | Easier (if familiar with Next.js) | Steeper |

---

### Q363. How do you handle push notifications in Expo?

**Difficulty:** 🟡 Medium | **Frequency:** Very High | **Category:** Push Notifications + Expo

**Answer:**
```bash
npx expo install expo-notifications expo-device expo-constants
```

```js
import * as Notifications from 'expo-notifications';
import * as Device from 'expo-device';
import Constants from 'expo-constants';

// Configure notification behaviour
Notifications.setNotificationHandler({
    handleNotification: async () => ({
        shouldShowAlert: true,
        shouldPlaySound: true,
        shouldSetBadge: true,
    }),
});

// Get Expo push token (for Expo's push service)
const registerForPushNotifications = async () => {
    if (!Device.isDevice) {
        console.log('Must use physical device for push notifications');
        return null;
    }

    const { status: existingStatus } = await Notifications.getPermissionsAsync();
    let finalStatus = existingStatus;

    if (existingStatus !== 'granted') {
        const { status } = await Notifications.requestPermissionsAsync();
        finalStatus = status;
    }

    if (finalStatus !== 'granted') {
        alert('Failed to get push token — permission denied');
        return null;
    }

    const projectId = Constants.expoConfig?.extra?.eas?.projectId;
    const token = (await Notifications.getExpoPushTokenAsync({ projectId })).data;
    console.log('Expo push token:', token);

    // Send token to your backend
    await api.updatePushToken(token);
    return token;
};

// Listen for incoming notifications
const usePushNotifications = () => {
    useEffect(() => {
        registerForPushNotifications();

        // Notification received while app is foregrounded
        const foregroundSub = Notifications.addNotificationReceivedListener(notification => {
            console.log('Received:', notification);
        });

        // User tapped on notification
        const responseSub = Notifications.addNotificationResponseReceivedListener(response => {
            const { screen, params } = response.notification.request.content.data;
            if (screen) navigation.navigate(screen, params);
        });

        return () => {
            foregroundSub.remove();
            responseSub.remove();
        };
    }, []);
};
```

```js
// Schedule local notification
await Notifications.scheduleNotificationAsync({
    content: {
        title: 'Shift starting',
        body: 'Your shift starts in 15 minutes',
        data: { screen: 'Attendance', action: 'checkIn' },
        sound: 'default',
        badge: 1,
    },
    trigger: {
        seconds: 900, // 15 minutes from now
    },
});

// Cancel all scheduled
await Notifications.cancelAllScheduledNotificationsAsync();
```

---

### Q364. What is the difference between `expo-updates` manual vs automatic updates?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** EAS Update

**Answer:**
```json
// app.json — update check strategy
{
  "expo": {
    "updates": {
      "checkAutomatically": "ON_LOAD"
      // Options:
      // "ON_LOAD"              — check on every app launch (default)
      // "ON_ERROR_RECOVERY"    — only check when previous bundle fails
      // "WIFI_ONLY"            — check only on Wi-Fi
      // "NEVER"                — disable automatic checks (manual only)
    }
  }
}
```

```js
// Automatic update (ON_LOAD behaviour):
// 1. App starts → check Expo servers for update
// 2. If update available → download in background
// 3. NEXT launch → uses new bundle
// ⚠️ User won't see update until NEXT launch

// Manual update check:
import * as Updates from 'expo-updates';

const ForceUpdatePrompt = () => {
    const [hasUpdate, setHasUpdate] = useState(false);

    useEffect(() => {
        checkUpdate();
    }, []);

    const checkUpdate = async () => {
        if (__DEV__) return;
        try {
            const { isAvailable } = await Updates.checkForUpdateAsync();
            if (isAvailable) setHasUpdate(true);
        } catch (e) {
            console.error('Update check failed:', e);
        }
    };

    const applyUpdate = async () => {
        try {
            await Updates.fetchUpdateAsync();
            await Updates.reloadAsync(); // immediate restart
        } catch (e) {
            Alert.alert('Update failed', 'Please restart the app manually');
        }
    };

    if (!hasUpdate) return null;
    return (
        <Banner
            message="New version available"
            action={{ label: 'Update Now', onPress: applyUpdate }}
        />
    );
};

// Critical update (force restart immediately)
const applyCriticalUpdate = async () => {
    const { isAvailable } = await Updates.checkForUpdateAsync();
    if (isAvailable) {
        await Updates.fetchUpdateAsync();
        await Updates.reloadAsync(); // user sees immediate restart
    }
};
```

---

### Q365. How do you profile and reduce Expo build times?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** EAS Build Optimisation

**Answer:**
```bash
# Check build duration in EAS dashboard
# https://expo.dev/accounts/[account]/projects/[project]/builds

# Common slow build causes and fixes:

# 1. Pod install taking too long (iOS)
# Fix: cache pods between builds
# eas.json:
# "cache": { "paths": ["~/.cocoapods"] }

# 2. npm install slow
# Fix: use npm ci + package-lock.json
# OR switch to yarn + yarn.lock
# EAS caches node_modules if lock file unchanged

# 3. Too many cocoapods
# Audit and remove unused ios pods
# pod deintegrate && pod install (clean install)

# 4. Gradle build slow (Android)
# Enable gradle caching in eas.json
```

```json
// eas.json — build caching
{
  "build": {
    "production": {
      "cache": {
        "key": "production-v1",
        "paths": [
          "node_modules",
          "~/.gradle/caches",
          "~/.cocoapods",
          "ios/Pods"
        ]
      }
    }
  }
}
```

```bash
# 5. Reduce JS bundle size (faster Metro bundling)
# Remove unused dependencies: npx depcheck
# Use tree-shakeable imports

# 6. Use EAS Build worker type
# "resourceClass": "medium" (default) vs "large" (faster, paid)
# eas.json:
# "resourceClass": "large"  # for faster builds

# 7. Split iOS and Android builds
# Don't use --platform all if only one changed
eas build --platform android --profile production

# 8. Parallel builds
eas build --platform all --profile production
# EAS automatically builds iOS and Android in parallel

# Monitor build logs for slow steps:
eas build --platform android --profile production --json | jq '.buildJob.logs'
```

---

### Q366. What is Expo's `app.json` `plugins` array?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Expo Config Plugins

**Answer:**
The `plugins` array in `app.json` / `app.config.js` specifies **config plugins** that run during `expo prebuild` to modify native project files.

```json
{
  "expo": {
    "plugins": [
      // String — plugin with no configuration
      "expo-router",

      // Array — plugin with options [pluginName, options]
      ["expo-camera", {
        "cameraPermission": "Allow $(PRODUCT_NAME) to access your camera to scan attendance QR codes."
      }],

      ["expo-location", {
        "locationWhenInUsePermission": "Allow $(PRODUCT_NAME) to use your location for attendance tracking.",
        "locationAlwaysPermission": "Allow $(PRODUCT_NAME) to track location in background."
      }],

      ["expo-notifications", {
        "icon": "./assets/notification-icon.png",
        "color": "#6200EE",
        "sounds": ["./assets/notification-sound.wav"],
        "androidMode": "default",
        "androidCollapsedTitle": "%(count)s notifications"
      }],

      // Path to custom plugin
      "./plugins/withCustomAndroidPermissions",
      "./plugins/withSentryNative"
    ]
  }
}
```

```js
// Writing your own plugin
// plugins/withSentryNative.js
const { withAppBuildGradle } = require('@expo/config-plugins');

const withSentryNative = (config) => {
    return withAppBuildGradle(config, (config) => {
        const buildGradle = config.modResults.contents;

        // Add Sentry plugin to build.gradle
        if (!buildGradle.includes('io.sentry.android.gradle')) {
            config.modResults.contents = buildGradle.replace(
                `apply plugin: "com.facebook.react"`,
                `apply plugin: "com.facebook.react"\napply plugin: "io.sentry.android.gradle"`
            );
        }
        return config;
    });
};

module.exports = withSentryNative;
```

---

### Q367. How do you implement OTA updates safely in production?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** EAS Update + Production

**Answer:**
```bash
# EAS Update channels — environment separation
# Each build is linked to a channel
# Updates are published to channels, not directly to builds

# Branch ≠ git branch — it's an EAS Update distribution group
eas update --branch production --message "Fix login crash"
eas update --branch staging --message "New dashboard feature"
eas update --branch dev --message "Experimental feature"
```

```json
// eas.json — link build profiles to update channels
{
  "build": {
    "production": {
      "channel": "production"   // builds receive updates from "production" branch
    },
    "staging": {
      "channel": "staging"
    },
    "development": {
      "channel": "development"
    }
  }
}
```

```bash
# Staged rollout — publish to subset of users
eas update --branch production --rollout-percentage 10
# 10% of users get the update → monitor for issues
# Increase rollout:
eas update:rollout --branch production --percentage 50
eas update:rollout --branch production --percentage 100

# Rollback — re-publish previous update
eas update:list --branch production
# Get ID of last good update
eas update:republish --branch production --group <update-group-id>
```

```js
// Client-side: graceful update with retry
const safeUpdate = async () => {
    try {
        const { isAvailable } = await Updates.checkForUpdateAsync();
        if (!isAvailable) return;

        await Updates.fetchUpdateAsync();
        // Only reload if app is idle (not mid-transaction)
        if (!isProcessingPayment && !isFormDirty) {
            await Updates.reloadAsync();
        } else {
            // Notify user to restart manually after completing action
            setPendingUpdate(true);
        }
    } catch (error) {
        // Don't crash the app if update fails
        Sentry.captureException(error);
    }
};
```

---

### Q368. What is `expo-constants` and what information does it expose?

**Difficulty:** 🟢 Easy | **Frequency:** High | **Category:** Expo SDK

**Answer:**
`expo-constants` exposes app configuration and device information that's available at JavaScript runtime.

```js
import Constants from 'expo-constants';

// App configuration
const appVersion = Constants.expoConfig?.version;           // "1.2.0"
const appName = Constants.expoConfig?.name;                 // "My App"
const slug = Constants.expoConfig?.slug;                    // "my-app"
const extra = Constants.expoConfig?.extra;                  // custom config from app.json

// Runtime info
const isExpoGo = Constants.appOwnership === 'expo';        // true in Expo Go
const sessionId = Constants.sessionId;                      // unique ID for this session
const executionEnvironment = Constants.executionEnvironment; // 'storeClient', 'standalone', 'bare'

// Device info (limited in Expo, use expo-device for more)
const deviceName = Constants.deviceName;                    // "Devesh's iPhone"

// Build info
const nativeAppVersion = Constants.nativeAppVersion;        // "1.2.0" from native build
const nativeBuildVersion = Constants.nativeBuildVersion;    // "23" (build number)

// Detect environment
const isStandaloneApp = Constants.executionEnvironment === 'standalone';
const isDevelopment = __DEV__;
const isExpo = Constants.appOwnership === 'expo';

// Access custom extra config (from app.config.js)
const { apiUrl, environment } = Constants.expoConfig?.extra || {};

// EAS Update info
const updateId = Updates.updateId;      // null if running embedded bundle
const channel = Updates.channel;        // "production", "staging"

// Usage in environment detection
export const getEnvironment = () => {
    const extra = Constants.expoConfig?.extra;
    return extra?.environment || 'development';
};

export const getApiUrl = () => {
    const extra = Constants.expoConfig?.extra;
    return extra?.apiUrl || 'http://localhost:3000';
};
```

---

### Q369. How do you debug EAS Build failures?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** EAS Build + Debugging

**Answer:**
```bash
# 1. Read build logs (most important step)
eas build:list          # list recent builds with IDs
eas build:view <build-id>  # open build details in browser
# OR open https://expo.dev → your project → Builds

# Common Android build failures:

# Error: "Task :app:compileDebugKotlin FAILED"
# Cause: Kotlin compilation error in native module
# Fix: Check the specific Kotlin error in logs, usually wrong API usage

# Error: "Execution failed for task ':app:mergeDebugAssets'"
# Cause: Duplicate asset files
# Fix: Check for duplicate files in android/app/src/main/assets

# Error: "Could not resolve com.example:library:1.0.0"
# Cause: Missing Maven dependency
# Fix: Add repository to android/build.gradle:
# maven { url 'https://jitpack.io' }

# Error: "Duplicate class kotlin.collections.jdk8"
# Cause: Kotlin version conflict
# Fix: android/build.gradle:
# configurations.all { resolutionStrategy.force 'org.jetbrains.kotlin:kotlin-stdlib:1.8.0' }
```

```bash
# Common iOS build failures:

# Error: "No profiles for 'com.company.app' were found"
# Fix: eas credentials --platform ios → regenerate provisioning profile

# Error: "CocoaPods could not find compatible versions for pod 'FirebaseAnalytics'"
# Fix: Update Podfile.lock: cd ios && pod update FirebaseAnalytics && cd ..
#      Commit updated Podfile.lock

# Error: "Undefined symbol: _RCTRegisterModule"
# Fix: Missing React Native linkage — check all pods are installed

# Error: "Multiple commands produce '/path/to/Info.plist'"
# Fix: Remove duplicate targets in Xcode project
```

```bash
# Local reproduction of EAS Build failure:
npx expo run:ios --no-install  # skip pod install (use cached pods)
npx expo run:android           # local Gradle build

# Verbose EAS build logs
eas build --platform android --profile production --local
# Runs build locally with same commands as EAS
# Faster debugging — see full output without waiting for CI
```

---

### Q370. What are the key differences between Expo SDK versions and how do you upgrade?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Expo Upgrades

**Answer:**
```bash
# Expo releases major SDK versions ~3x per year
# Each SDK version targets a specific React Native version:
# SDK 50 → React Native 0.73
# SDK 51 → React Native 0.74
# SDK 52 → React Native 0.76 (New Architecture default)

# Check your current SDK version
cat package.json | grep '"expo"'

# Upgrade to latest SDK
npx expo install expo@latest    # upgrade expo core

# Upgrade all Expo SDK packages to match new SDK
npx expo install --fix

# Full upgrade process:
# 1. Read the migration guide: docs.expo.dev/workflow/upgrading-expo-sdk
# 2. Update expo in package.json
npm install expo@51  # or whatever version

# 3. Update all expo-* packages
npx expo install --fix

# 4. Update React Native (if SDK requires it)
npx expo install react-native@0.74 react@18.2.0

# 5. Regenerate native (if bare/CNG)
npx expo prebuild --clean
cd ios && pod install && cd ..

# 6. Test thoroughly
npx expo run:android
npx expo run:ios
```

```bash
# Breaking changes to watch for:

# SDK 50: expo-router v3, New Architecture improvements
# SDK 51: Expo DOM Components, expo-video (replaces expo-av for video)
# SDK 52: New Architecture default, React 19

# expo-doctor — checks for issues before upgrading
npx expo-doctor

# Check changelog before upgrading:
# https://expo.dev/changelog
# https://github.com/expo/expo/releases

# Canary versions — test upcoming SDK before release
npm install expo@canary

# Support window:
# Expo supports last 3 SDK versions with security patches
# Older than 3 versions = no support, must upgrade
```

---

## Sections Overview (Q371–Q500)

| Section | Questions | Topics |
|---------|-----------|--------|
| Testing | Q371–Q420 | Unit, integration, e2e (Detox), mocking |
| Debugging | Q421–Q450 | Flipper, Hermes debugger, crash reporting |
| Storage & Permissions | Q451–Q480 | AsyncStorage, Keychain, permission flows |
| Miscellaneous | Q481–Q500 | Accessibility, internationalisation, misc APIs |

---

> 💡 **Tip for GitHub:** Add a `## Table of Contents` section at the top with anchor links to each question for easy navigation.

---

*Part 01 of 8 — [← Back to Part README](./README.md) · [← Main README](../README.md)*

## Section 10: Testing (Q371–Q420)

---

### Q371. What is the React Native testing pyramid?

**Difficulty:** 🟡 Medium | **Frequency:** Very High | **Category:** Testing Strategy

**Answer:**
The testing pyramid defines three layers of tests, each with different speed, cost, and coverage tradeoffs:

```
        ▲
       /E\       E2E Tests (Detox / Maestro)
      /   \      — Slowest, most realistic, fewest
     /-----\
    / Integ  \   Integration Tests (React Native Testing Library)
   /    ration \  — Medium speed, test component trees
  /─────────────\
 /  Unit Tests   \ Unit Tests (Jest)
/─────────────────\ — Fastest, most isolated, most numerous
```

```
Layer      | Tool              | What to test                     | Count
───────────────────────────────────────────────────────────────────────
Unit       | Jest              | Utilities, hooks, reducers        | Many (70%)
Integration| RNTL              | Component + child interactions    | Some (20%)
E2E        | Detox / Maestro   | Critical user journeys            | Few (10%)
```

```js
// Unit test — isolated, fast, no rendering
test('formatCurrency formats INR correctly', () => {
    expect(formatCurrency(85000, 'INR')).toBe('₹85,000');
    expect(formatCurrency(0, 'INR')).toBe('₹0');
});

// Integration test — renders component tree, tests behaviour
test('LoginForm submits credentials', async () => {
    const onLogin = jest.fn();
    render(<LoginForm onLogin={onLogin} />);
    fireEvent.changeText(screen.getByPlaceholderText('Email'), 'dev@test.com');
    fireEvent.changeText(screen.getByPlaceholderText('Password'), 'password123');
    fireEvent.press(screen.getByText('Login'));
    await waitFor(() => expect(onLogin).toHaveBeenCalledWith('dev@test.com', 'password123'));
});

// E2E test — real app, real device
describe('Login flow', () => {
    it('should log in and land on dashboard', async () => {
        await element(by.id('email-input')).typeText('dev@test.com');
        await element(by.id('password-input')).typeText('password123');
        await element(by.id('login-button')).tap();
        await expect(element(by.id('dashboard-screen'))).toBeVisible();
    });
});
```

**Follow-up:** What percentage of your ERP app was covered by tests? → We maintained ~65% unit + integration coverage with Jest/RNTL for business-critical flows (payroll, attendance, leave management). E2E with Detox covered the 5 core user journeys.

---

### Q372. How do you set up Jest for a React Native project?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Jest Setup

**Answer:**
```bash
# React Native CLI — Jest is pre-configured
# package.json already has jest config after init

# Expo — also pre-configured
npx expo install jest-expo --save-dev

# Install testing utilities
npm install --save-dev @testing-library/react-native
npm install --save-dev @testing-library/jest-native  # custom matchers
npm install --save-dev @testing-library/user-event    # user interactions
```

```json
// package.json — Jest configuration
{
  "jest": {
    "preset": "react-native",
    // OR for Expo:
    // "preset": "jest-expo",

    "setupFilesAfterFramework": [
      "@testing-library/jest-native/extend-expect"
    ],
    "setupFiles": [
      "./jest.setup.js"
    ],
    "transformIgnorePatterns": [
      "node_modules/(?!(react-native|@react-native|@react-navigation|react-native-reanimated|react-native-gesture-handler)/)"
    ],
    "moduleNameMapper": {
      "\\.(png|jpg|jpeg|gif|svg)$": "<rootDir>/__mocks__/fileMock.js",
      "\\.(mp4|mp3|wav)$": "<rootDir>/__mocks__/fileMock.js"
    },
    "collectCoverageFrom": [
      "src/**/*.{ts,tsx}",
      "!src/**/*.d.ts",
      "!src/**/*.stories.tsx"
    ],
    "coverageThreshold": {
      "global": {
        "branches": 60,
        "functions": 70,
        "lines": 70
      }
    }
  }
}
```

```js
// jest.setup.js — global setup
import 'react-native-gesture-handler/jestSetup';

// Mock Reanimated
jest.mock('react-native-reanimated', () => {
    const Reanimated = require('react-native-reanimated/mock');
    Reanimated.default.call = jest.fn();
    return Reanimated;
});

// Silence yellow box warnings in tests
jest.mock('react-native/Libraries/Animated/NativeAnimatedHelper');

// Mock AsyncStorage
jest.mock('@react-native-async-storage/async-storage',
    () => require('@react-native-async-storage/async-storage/jest/async-storage-mock')
);
```

```js
// __mocks__/fileMock.js
module.exports = 'test-file-stub';
```

---

### Q373. What is React Native Testing Library (RNTL) and how does it differ from Enzyme?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** RNTL

**Answer:**
React Native Testing Library (RNTL) is the recommended testing library for React Native. It renders components and lets you query/interact with them in a way that mirrors user behaviour.

**RNTL vs Enzyme:**

| | RNTL | Enzyme |
|--|------|--------|
| Philosophy | Test behaviour (what user sees) | Test implementation (component internals) |
| Query style | By text, label, role, testID | By component name, props, state |
| Maintenance | ✅ Actively maintained | ❌ Deprecated |
| Resilience | Refactor-friendly | Brittle (breaks on rename/restructure) |

```js
import { render, screen, fireEvent, waitFor } from '@testing-library/react-native';

// Queries — multiple ways to find elements
const el = screen.getByText('Submit');              // exact text match
const el = screen.getByRole('button', { name: 'Submit' }); // by accessibility role
const el = screen.getByTestId('submit-button');     // by testID prop
const el = screen.getByPlaceholderText('Email');    // by placeholder
const el = screen.getByLabelText('Email address'); // by accessibilityLabel
const el = screen.getByDisplayValue('devesh@...');  // current input value

// queryBy — returns null if not found (no throw)
const el = screen.queryByText('Error message');     // null if not present
expect(el).toBeNull();

// findBy — async, waits for element to appear
const el = await screen.findByText('Loading complete'); // waits

// getAllBy / queryAllBy / findAllBy — multiple elements
const buttons = screen.getAllByRole('button');

// Custom matchers (@testing-library/jest-native)
expect(el).toBeVisible();
expect(el).toBeDisabled();
expect(el).toHaveTextContent('Hello');
expect(el).toHaveProp('style', expect.objectContaining({ color: 'red' }));
```

---

### Q374. How do you write a unit test for a React hook?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Hook Testing

**Answer:**
```js
// src/hooks/useCounter.ts
export const useCounter = (initial = 0) => {
    const [count, setCount] = useState(initial);
    const increment = useCallback(() => setCount(c => c + 1), []);
    const decrement = useCallback(() => setCount(c => c - 1), []);
    const reset = useCallback(() => setCount(initial), [initial]);
    return { count, increment, decrement, reset };
};

// useCounter.test.ts
import { renderHook, act } from '@testing-library/react-native';
import { useCounter } from '../hooks/useCounter';

describe('useCounter', () => {
    it('initialises with default value 0', () => {
        const { result } = renderHook(() => useCounter());
        expect(result.current.count).toBe(0);
    });

    it('initialises with custom value', () => {
        const { result } = renderHook(() => useCounter(10));
        expect(result.current.count).toBe(10);
    });

    it('increments count', () => {
        const { result } = renderHook(() => useCounter());
        act(() => { result.current.increment(); });
        expect(result.current.count).toBe(1);
    });

    it('decrements count', () => {
        const { result } = renderHook(() => useCounter(5));
        act(() => { result.current.decrement(); });
        expect(result.current.count).toBe(4);
    });

    it('resets to initial value', () => {
        const { result } = renderHook(() => useCounter(3));
        act(() => { result.current.increment(); result.current.increment(); });
        expect(result.current.count).toBe(5);
        act(() => { result.current.reset(); });
        expect(result.current.count).toBe(3);
    });
});
```

```js
// Testing async hooks
import { renderHook, waitFor } from '@testing-library/react-native';

const useEmployees = () => {
    const [employees, setEmployees] = useState([]);
    const [loading, setLoading] = useState(true);

    useEffect(() => {
        fetchEmployees().then(data => {
            setEmployees(data);
            setLoading(false);
        });
    }, []);

    return { employees, loading };
};

// Test with API mock
jest.mock('../api/employees', () => ({
    fetchEmployees: jest.fn().mockResolvedValue([
        { id: 1, name: 'Devesh Kumar' },
        { id: 2, name: 'Priya Singh' },
    ]),
}));

test('loads employees', async () => {
    const { result } = renderHook(() => useEmployees());

    // Initially loading
    expect(result.current.loading).toBe(true);
    expect(result.current.employees).toHaveLength(0);

    // After fetch resolves
    await waitFor(() => {
        expect(result.current.loading).toBe(false);
    });
    expect(result.current.employees).toHaveLength(2);
    expect(result.current.employees[0].name).toBe('Devesh Kumar');
});
```

---

### Q375. How do you mock modules in Jest?

**Difficulty:** 🟡 Medium | **Frequency:** Very High | **Category:** Jest Mocking

**Answer:**
```js
// 1. Auto-mock an entire module
jest.mock('../api/userApi');
// All exports become jest.fn() — return undefined by default

// 2. Manual mock with implementation
jest.mock('../api/userApi', () => ({
    fetchUser: jest.fn().mockResolvedValue({ id: 1, name: 'Devesh' }),
    updateUser: jest.fn().mockResolvedValue({ success: true }),
    deleteUser: jest.fn().mockRejectedValue(new Error('Forbidden')),
}));

// 3. Mock a React Native module
jest.mock('react-native', () => {
    const RN = jest.requireActual('react-native');
    return {
        ...RN,
        Alert: { alert: jest.fn() },
        Linking: { openURL: jest.fn().mockResolvedValue(true) },
        Platform: { OS: 'ios', select: jest.fn(obj => obj.ios) },
    };
});

// 4. Mock per-test (restore between tests)
import { fetchUser } from '../api/userApi';
const mockFetchUser = fetchUser as jest.MockedFunction<typeof fetchUser>;

beforeEach(() => {
    mockFetchUser.mockResolvedValue({ id: 1, name: 'Devesh' });
});

test('handles API error', async () => {
    mockFetchUser.mockRejectedValueOnce(new Error('Network error'));
    // test error handling
});

// 5. Spy on module method (observe without replacing)
import * as Analytics from '../utils/analytics';
const trackSpy = jest.spyOn(Analytics, 'trackEvent');

test('tracks login event', () => {
    fireEvent.press(screen.getByText('Login'));
    expect(trackSpy).toHaveBeenCalledWith('login', { method: 'email' });
});

afterEach(() => { trackSpy.mockRestore(); });

// 6. Mock with different values per test
test('shows error when fetch fails', async () => {
    jest.mocked(fetchUser).mockRejectedValueOnce(new Error('Server error'));
    render(<UserProfile userId="1" />);
    await screen.findByText('Failed to load user');
});
```

---

### Q376. How do you test a React Native component with RNTL?

**Difficulty:** 🟡 Medium | **Frequency:** Very High | **Category:** RNTL Component Testing

**Answer:**
```js
// src/components/EmployeeCard.tsx
const EmployeeCard = ({ employee, onPress, onDelete }) => (
    <Pressable testID="employee-card" onPress={() => onPress(employee.id)}>
        <View>
            <Text testID="employee-name">{employee.name}</Text>
            <Text testID="employee-role">{employee.role}</Text>
            <Text testID="employee-dept">{employee.department}</Text>
            {employee.isActive
                ? <View testID="active-badge" accessibilityLabel="Active" />
                : <View testID="inactive-badge" accessibilityLabel="Inactive" />
            }
            <Pressable
                testID="delete-button"
                accessibilityLabel={`Delete ${employee.name}`}
                onPress={() => onDelete(employee.id)}
            >
                <Text>Delete</Text>
            </Pressable>
        </View>
    </Pressable>
);

// EmployeeCard.test.tsx
import { render, screen, fireEvent } from '@testing-library/react-native';
import { EmployeeCard } from '../EmployeeCard';

const mockEmployee = {
    id: '42',
    name: 'Devesh Kumar Singh',
    role: 'React Native Developer',
    department: 'Engineering',
    isActive: true,
};

describe('EmployeeCard', () => {
    it('renders employee information', () => {
        render(<EmployeeCard employee={mockEmployee} onPress={jest.fn()} onDelete={jest.fn()} />);

        expect(screen.getByTestId('employee-name')).toHaveTextContent('Devesh Kumar Singh');
        expect(screen.getByTestId('employee-role')).toHaveTextContent('React Native Developer');
        expect(screen.getByTestId('employee-dept')).toHaveTextContent('Engineering');
    });

    it('shows active badge for active employee', () => {
        render(<EmployeeCard employee={mockEmployee} onPress={jest.fn()} onDelete={jest.fn()} />);
        expect(screen.getByLabelText('Active')).toBeTruthy();
        expect(screen.queryByLabelText('Inactive')).toBeNull();
    });

    it('shows inactive badge for inactive employee', () => {
        render(<EmployeeCard employee={{ ...mockEmployee, isActive: false }}
            onPress={jest.fn()} onDelete={jest.fn()} />);
        expect(screen.getByLabelText('Inactive')).toBeTruthy();
    });

    it('calls onPress with employee id when card is tapped', () => {
        const onPress = jest.fn();
        render(<EmployeeCard employee={mockEmployee} onPress={onPress} onDelete={jest.fn()} />);

        fireEvent.press(screen.getByTestId('employee-card'));
        expect(onPress).toHaveBeenCalledWith('42');
        expect(onPress).toHaveBeenCalledTimes(1);
    });

    it('calls onDelete with employee id when delete is tapped', () => {
        const onDelete = jest.fn();
        render(<EmployeeCard employee={mockEmployee} onPress={jest.fn()} onDelete={onDelete} />);

        fireEvent.press(screen.getByLabelText('Delete Devesh Kumar Singh'));
        expect(onDelete).toHaveBeenCalledWith('42');
    });
});
```

---

### Q377. How do you test asynchronous behaviour (API calls) in RNTL?

**Difficulty:** 🔴 Hard | **Frequency:** Very High | **Category:** Async Testing

**Answer:**
```js
// Component under test
const EmployeeList = () => {
    const [employees, setEmployees] = useState([]);
    const [loading, setLoading] = useState(false);
    const [error, setError] = useState(null);

    const load = async () => {
        setLoading(true);
        setError(null);
        try {
            const data = await fetchEmployees();
            setEmployees(data);
        } catch (e) {
            setError(e.message);
        } finally {
            setLoading(false);
        }
    };

    useEffect(() => { load(); }, []);

    if (loading) return <ActivityIndicator testID="loader" />;
    if (error) return <Text testID="error-message">{error}</Text>;
    return (
        <FlatList
            testID="employee-list"
            data={employees}
            keyExtractor={e => e.id}
            renderItem={({ item }) => <Text testID={`employee-${item.id}`}>{item.name}</Text>}
        />
    );
};

// Test file
import { render, screen, waitFor, act } from '@testing-library/react-native';
import { fetchEmployees } from '../api';
jest.mock('../api');

describe('EmployeeList', () => {
    // Happy path
    it('shows employees after loading', async () => {
        (fetchEmployees as jest.Mock).mockResolvedValue([
            { id: '1', name: 'Devesh Kumar' },
            { id: '2', name: 'Priya Singh' },
        ]);

        render(<EmployeeList />);

        // Shows loader initially
        expect(screen.getByTestId('loader')).toBeTruthy();

        // Waits for async update
        await waitFor(() => {
            expect(screen.queryByTestId('loader')).toBeNull();
        });

        // Shows employees
        expect(screen.getByTestId('employee-1')).toHaveTextContent('Devesh Kumar');
        expect(screen.getByTestId('employee-2')).toHaveTextContent('Priya Singh');
    });

    // Error path
    it('shows error message on API failure', async () => {
        (fetchEmployees as jest.Mock).mockRejectedValue(new Error('Network error'));

        render(<EmployeeList />);

        await waitFor(() => {
            expect(screen.getByTestId('error-message')).toHaveTextContent('Network error');
        });
        expect(screen.queryByTestId('loader')).toBeNull();
    });

    // Empty state
    it('shows empty list when no employees', async () => {
        (fetchEmployees as jest.Mock).mockResolvedValue([]);

        render(<EmployeeList />);

        await waitFor(() => {
            expect(screen.getByTestId('employee-list')).toBeTruthy();
        });
        expect(screen.queryByTestId('employee-1')).toBeNull();
    });
});
```

---

### Q378. How do you test navigation in React Native?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Navigation Testing

**Answer:**
```js
// Option 1: Mock navigation object (unit test style)
const createMockNavigation = () => ({
    navigate: jest.fn(),
    goBack: jest.fn(),
    replace: jest.fn(),
    push: jest.fn(),
    reset: jest.fn(),
    setOptions: jest.fn(),
    addListener: jest.fn(() => jest.fn()), // returns unsubscribe fn
    isFocused: jest.fn().mockReturnValue(true),
    getParam: jest.fn(),
    dispatch: jest.fn(),
});

test('navigates to detail on employee press', () => {
    const navigation = createMockNavigation();
    const route = { params: { departmentId: 'eng' } };

    render(<EmployeeList navigation={navigation} route={route} />);

    fireEvent.press(screen.getByText('Devesh Kumar'));

    expect(navigation.navigate).toHaveBeenCalledWith('EmployeeDetail', { employeeId: '1' });
});

// Option 2: Render with full NavigationContainer (integration test)
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';

const Stack = createNativeStackNavigator();

const TestNavigator = ({ initialRouteName = 'EmployeeList' }) => (
    <NavigationContainer>
        <Stack.Navigator initialRouteName={initialRouteName}>
            <Stack.Screen name="EmployeeList" component={EmployeeList} />
            <Stack.Screen name="EmployeeDetail" component={EmployeeDetail} />
        </Stack.Navigator>
    </NavigationContainer>
);

test('navigates to detail and shows employee info', async () => {
    render(<TestNavigator />);

    // EmployeeList is showing
    await screen.findByText('Devesh Kumar');

    // Tap to navigate
    fireEvent.press(screen.getByText('Devesh Kumar'));

    // EmployeeDetail loads
    await screen.findByText('React Native Developer'); // role shown in detail
});

// Option 3: useNavigation mock (for components that use the hook)
jest.mock('@react-navigation/native', () => ({
    ...jest.requireActual('@react-navigation/native'),
    useNavigation: () => ({ navigate: jest.fn(), goBack: jest.fn() }),
    useRoute: () => ({ params: { employeeId: '42' } }),
    useFocusEffect: jest.fn(cb => cb()),
}));
```

---

### Q379. How do you test Redux (or Zustand) connected components?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** State Management Testing

**Answer:**
```js
// Redux — wrap with Provider in tests
import { Provider } from 'react-redux';
import { configureStore } from '@reduxjs/toolkit';
import { employeesSlice } from '../store/employeesSlice';

// Create a test store factory
const createTestStore = (preloadedState = {}) =>
    configureStore({
        reducer: { employees: employeesSlice.reducer },
        preloadedState,
    });

// Custom render helper
const renderWithRedux = (ui, { preloadedState = {}, store = createTestStore(preloadedState) } = {}) => {
    const Wrapper = ({ children }) => <Provider store={store}>{children}</Provider>;
    return { store, ...render(ui, { wrapper: Wrapper }) };
};

// Test
test('displays employees from Redux store', async () => {
    const { store } = renderWithRedux(<EmployeeListScreen />, {
        preloadedState: {
            employees: {
                list: [{ id: '1', name: 'Devesh Kumar', role: 'Developer' }],
                loading: false,
                error: null,
            },
        },
    });

    expect(screen.getByText('Devesh Kumar')).toBeTruthy();
});

test('dispatches fetchEmployees on mount', async () => {
    const { store } = renderWithRedux(<EmployeeListScreen />);
    const dispatchSpy = jest.spyOn(store, 'dispatch');
    // wait for mount effect
    await waitFor(() => {
        expect(dispatchSpy).toHaveBeenCalled();
    });
});
```

```js
// Zustand — reset store between tests
import { create } from 'zustand';
import { useEmployeeStore } from '../store/employeeStore';

// Reset Zustand store between tests
beforeEach(() => {
    useEmployeeStore.setState({
        employees: [],
        loading: false,
        error: null,
    });
});

test('add employee updates store and rerenders', async () => {
    render(<EmployeeForm />);

    fireEvent.changeText(screen.getByPlaceholderText('Employee name'), 'Priya Singh');
    fireEvent.press(screen.getByText('Add Employee'));

    await waitFor(() => {
        const { employees } = useEmployeeStore.getState();
        expect(employees).toHaveLength(1);
        expect(employees[0].name).toBe('Priya Singh');
    });
});
```

---

### Q380. What is `act()` and when do you need it explicitly?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Testing Internals

**Answer:**
`act()` ensures that all pending state updates, effects, and re-renders are processed before you make assertions. React Testing Library wraps most interactions in `act()` automatically.

```js
import { act, render } from '@testing-library/react-native';

// ✅ RNTL automatically wraps these in act():
fireEvent.press(button);          // automatic act()
fireEvent.changeText(input, 'x'); // automatic act()
await waitFor(() => ...);         // automatic act()
await findByText('...');          // automatic act()

// ❌ You need explicit act() when:
// 1. Triggering state updates outside RNTL helpers

test('timer updates count', async () => {
    jest.useFakeTimers();
    render(<CountdownTimer seconds={5} />);

    // Fast-forward timer — must wrap in act() because it triggers state update
    act(() => {
        jest.advanceTimersByTime(3000);
    });

    expect(screen.getByText('2')).toBeTruthy();
    jest.useRealTimers();
});

// 2. Resolving promises that update state
test('async state update', async () => {
    const { rerender } = render(<MyComponent />);

    await act(async () => {
        await someAsyncOperation(); // triggers state update
    });

    expect(screen.getByText('Updated')).toBeTruthy();
});

// Common warning: "Warning: An update to Component inside a test was not wrapped in act(...)"
// Fix: wrap the trigger in act() or use waitFor()
```

---

### Q381. How do you test a form with validation?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** RNTL Forms

**Answer:**
```js
// LoginForm.tsx
const LoginForm = ({ onSubmit }) => {
    const { control, handleSubmit, formState: { errors } } = useForm({
        resolver: zodResolver(loginSchema),
    });

    return (
        <View>
            <Controller
                control={control}
                name="email"
                render={({ field: { onChange, value } }) => (
                    <TextInput
                        testID="email-input"
                        placeholder="Email"
                        value={value}
                        onChangeText={onChange}
                        accessibilityLabel="Email address"
                    />
                )}
            />
            {errors.email && (
                <Text testID="email-error">{errors.email.message}</Text>
            )}

            <Controller
                control={control}
                name="password"
                render={({ field: { onChange, value } }) => (
                    <TextInput
                        testID="password-input"
                        placeholder="Password"
                        secureTextEntry
                        value={value}
                        onChangeText={onChange}
                    />
                )}
            />
            {errors.password && (
                <Text testID="password-error">{errors.password.message}</Text>
            )}

            <Pressable testID="submit-button" onPress={handleSubmit(onSubmit)}>
                <Text>Login</Text>
            </Pressable>
        </View>
    );
};

// LoginForm.test.tsx
describe('LoginForm', () => {
    it('shows validation errors for empty submission', async () => {
        const onSubmit = jest.fn();
        render(<LoginForm onSubmit={onSubmit} />);

        fireEvent.press(screen.getByTestId('submit-button'));

        await waitFor(() => {
            expect(screen.getByTestId('email-error')).toHaveTextContent('Email is required');
            expect(screen.getByTestId('password-error')).toHaveTextContent('Password is required');
        });
        expect(onSubmit).not.toHaveBeenCalled();
    });

    it('shows invalid email error', async () => {
        render(<LoginForm onSubmit={jest.fn()} />);
        fireEvent.changeText(screen.getByTestId('email-input'), 'not-an-email');
        fireEvent.press(screen.getByTestId('submit-button'));

        await waitFor(() => {
            expect(screen.getByTestId('email-error')).toHaveTextContent('Invalid email');
        });
    });

    it('submits with valid credentials', async () => {
        const onSubmit = jest.fn();
        render(<LoginForm onSubmit={onSubmit} />);

        fireEvent.changeText(screen.getByTestId('email-input'), 'devesh@example.com');
        fireEvent.changeText(screen.getByTestId('password-input'), 'password123');
        fireEvent.press(screen.getByTestId('submit-button'));

        await waitFor(() => {
            expect(onSubmit).toHaveBeenCalledWith({
                email: 'devesh@example.com',
                password: 'password123',
            });
        });
    });
});
```

---

### Q382. How do you mock `AsyncStorage` in Jest?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Mocking Native Modules

**Answer:**
```js
// Option 1: Use the official mock package
// jest.setup.js
jest.mock('@react-native-async-storage/async-storage',
    () => require('@react-native-async-storage/async-storage/jest/async-storage-mock')
);

// The mock is an in-memory store that resets between test files
// (but NOT between tests in the same file unless you clear it)

// Test using the mock
import AsyncStorage from '@react-native-async-storage/async-storage';

test('saves and retrieves auth token', async () => {
    // Save
    await AsyncStorage.setItem('authToken', 'jwt-token-123');

    // Retrieve
    const token = await AsyncStorage.getItem('authToken');
    expect(token).toBe('jwt-token-123');
});

// Clear between tests
afterEach(async () => {
    await AsyncStorage.clear();
});

// Option 2: Manual mock (more control)
jest.mock('@react-native-async-storage/async-storage', () => {
    let store: Record<string, string> = {};
    return {
        setItem: jest.fn((key, value) => { store[key] = value; return Promise.resolve(); }),
        getItem: jest.fn((key) => Promise.resolve(store[key] ?? null)),
        removeItem: jest.fn((key) => { delete store[key]; return Promise.resolve(); }),
        clear: jest.fn(() => { store = {}; return Promise.resolve(); }),
        getAllKeys: jest.fn(() => Promise.resolve(Object.keys(store))),
        multiSet: jest.fn((pairs) => { pairs.forEach(([k, v]) => { store[k] = v; }); return Promise.resolve(); }),
        multiGet: jest.fn((keys) => Promise.resolve(keys.map(k => [k, store[k] ?? null]))),
    };
});

// Verify calls
test('token is stored on login', async () => {
    await loginUser('devesh@test.com', 'pass123');

    expect(AsyncStorage.setItem).toHaveBeenCalledWith(
        'authToken',
        expect.any(String)
    );
});
```

---

### Q383. How do you test React Navigation screens?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Navigation Testing

**Answer:**
```js
// Create a navigation test utility
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';

const Stack = createNativeStackNavigator();

// Test wrapper with navigation
const renderScreen = (Screen, params = {}, additionalScreens = []) => {
    const TestApp = () => (
        <NavigationContainer>
            <Stack.Navigator>
                <Stack.Screen
                    name="Screen"
                    component={Screen}
                    initialParams={params}
                />
                {additionalScreens.map(({ name, component }) => (
                    <Stack.Screen key={name} name={name} component={component} />
                ))}
            </Stack.Navigator>
        </NavigationContainer>
    );
    return render(<TestApp />);
};

// Test a screen
test('ProfileScreen displays user data', async () => {
    const mockUser = { id: '1', name: 'Devesh Kumar', email: 'dev@test.com' };
    jest.mocked(fetchUser).mockResolvedValue(mockUser);

    renderScreen(ProfileScreen, { userId: '1' });

    await screen.findByText('Devesh Kumar');
    expect(screen.getByText('dev@test.com')).toBeTruthy();
});

test('EmployeeDetail back button goes to list', async () => {
    const mockEmployee = { id: '1', name: 'Devesh' };
    jest.mocked(fetchEmployee).mockResolvedValue(mockEmployee);

    renderScreen(
        EmployeeDetailScreen,
        { employeeId: '1' },
        [{ name: 'EmployeeList', component: EmployeeListScreen }]
    );

    await screen.findByText('Devesh');
    fireEvent.press(screen.getByLabelText('Go back'));

    // Should be on list now
    await screen.findByTestId('employee-list');
});

// Test header buttons
test('save button in header triggers save', async () => {
    const onSave = jest.fn();
    renderScreen(EditEmployeeScreen, { employeeId: '1', onSave });

    await screen.findByTestId('employee-form');
    fireEvent.press(screen.getByText('Save'));

    await waitFor(() => expect(onSave).toHaveBeenCalled());
});
```

---

### Q384. How do you test custom React hooks with side effects?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Hook Testing

**Answer:**
```js
// Complex hook with side effects
const useAttendance = (employeeId: string) => {
    const [attendance, setAttendance] = useState<Attendance[]>([]);
    const [loading, setLoading] = useState(false);
    const [error, setError] = useState<string | null>(null);

    const checkIn = useCallback(async () => {
        try {
            const record = await api.checkIn(employeeId);
            setAttendance(prev => [...prev, record]);
            return record;
        } catch (e) {
            setError(e.message);
            throw e;
        }
    }, [employeeId]);

    const checkOut = useCallback(async (recordId: string) => {
        const updated = await api.checkOut(recordId);
        setAttendance(prev => prev.map(r => r.id === recordId ? updated : r));
    }, []);

    useEffect(() => {
        setLoading(true);
        api.getAttendance(employeeId)
            .then(setAttendance)
            .catch(e => setError(e.message))
            .finally(() => setLoading(false));
    }, [employeeId]);

    return { attendance, loading, error, checkIn, checkOut };
};

// Test file
import { renderHook, act, waitFor } from '@testing-library/react-native';
import { useAttendance } from '../hooks/useAttendance';
import * as api from '../api';
jest.mock('../api');

describe('useAttendance', () => {
    const mockRecord = { id: 'r1', employeeId: 'e1', checkIn: '09:00', checkOut: null };

    beforeEach(() => {
        jest.mocked(api.getAttendance).mockResolvedValue([mockRecord]);
        jest.mocked(api.checkIn).mockResolvedValue({ ...mockRecord, id: 'r2' });
        jest.mocked(api.checkOut).mockResolvedValue({ ...mockRecord, checkOut: '18:00' });
    });

    it('loads attendance on mount', async () => {
        const { result } = renderHook(() => useAttendance('e1'));

        expect(result.current.loading).toBe(true);

        await waitFor(() => {
            expect(result.current.loading).toBe(false);
        });

        expect(result.current.attendance).toHaveLength(1);
        expect(result.current.attendance[0].id).toBe('r1');
    });

    it('adds record on checkIn', async () => {
        const { result } = renderHook(() => useAttendance('e1'));
        await waitFor(() => expect(result.current.loading).toBe(false));

        await act(async () => {
            await result.current.checkIn();
        });

        expect(result.current.attendance).toHaveLength(2);
    });

    it('updates record on checkOut', async () => {
        const { result } = renderHook(() => useAttendance('e1'));
        await waitFor(() => expect(result.current.loading).toBe(false));

        await act(async () => {
            await result.current.checkOut('r1');
        });

        expect(result.current.attendance[0].checkOut).toBe('18:00');
    });

    it('sets error on API failure', async () => {
        jest.mocked(api.getAttendance).mockRejectedValue(new Error('Unauthorized'));
        const { result } = renderHook(() => useAttendance('e1'));

        await waitFor(() => {
            expect(result.current.error).toBe('Unauthorized');
        });
    });
});
```

---

### Q385. What is Detox and how does it work?

**Difficulty:** 🟡 Medium | **Frequency:** Very High | **Category:** E2E Testing

**Answer:**
Detox is a grey-box E2E testing framework for React Native. It runs your actual app on a real device/simulator and simulates user interactions.

**Grey-box testing:** Detox has access to the React Native runtime internals — it knows when the JS thread is idle, so it can synchronise test actions without arbitrary sleeps.

```bash
# Install
npm install --save-dev detox jest-circus

# Configure Detox
npx detox init --runner jest

# Build test app (once, or when native changes)
npx detox build --configuration ios.sim.debug
npx detox build --configuration android.emu.debug

# Run tests
npx detox test --configuration ios.sim.debug
npx detox test --configuration ios.sim.debug --testNamePattern "Login"
```

```js
// .detoxrc.js
module.exports = {
    testRunner: {
        $0: 'jest',
        args: { config: 'e2e/jest.config.js' },
    },
    apps: {
        'ios.debug': {
            type: 'ios.app',
            binaryPath: 'ios/build/Build/Products/Debug-iphonesimulator/MyApp.app',
            build: 'xcodebuild -workspace ios/MyApp.xcworkspace -scheme MyApp -configuration Debug -sdk iphonesimulator -derivedDataPath ios/build',
        },
        'android.debug': {
            type: 'android.apk',
            binaryPath: 'android/app/build/outputs/apk/debug/app-debug.apk',
            build: 'cd android && ./gradlew assembleDebug assembleAndroidTest -DtestBuildType=debug',
        },
    },
    devices: {
        simulator: {
            type: 'ios.simulator',
            device: { type: 'iPhone 15' },
        },
        emulator: {
            type: 'android.emulator',
            device: { avdName: 'Pixel_6_API_33' },
        },
    },
    configurations: {
        'ios.sim.debug': { device: 'simulator', app: 'ios.debug' },
        'android.emu.debug': { device: 'emulator', app: 'android.debug' },
    },
};
```

---

### Q386. How do you write a Detox E2E test?

**Difficulty:** 🟡 Medium | **Frequency:** Very High | **Category:** Detox

**Answer:**
```js
// e2e/login.e2e.js
describe('Login Flow', () => {
    beforeAll(async () => {
        await device.launchApp({ newInstance: true });
    });

    beforeEach(async () => {
        await device.reloadReactNative(); // fresh state each test
    });

    it('should show login screen on launch', async () => {
        await expect(element(by.id('login-screen'))).toBeVisible();
        await expect(element(by.id('email-input'))).toBeVisible();
        await expect(element(by.id('password-input'))).toBeVisible();
    });

    it('should show error for invalid credentials', async () => {
        await element(by.id('email-input')).typeText('wrong@test.com');
        await element(by.id('password-input')).typeText('wrongpass');
        await element(by.id('login-button')).tap();

        await expect(element(by.text('Invalid credentials'))).toBeVisible();
    });

    it('should navigate to dashboard after login', async () => {
        await element(by.id('email-input')).typeText('devesh@erp.com');
        await element(by.id('password-input')).typeText('ValidPass123');
        await element(by.id('login-button')).tap();

        // Detox waits for JS idle automatically — no arbitrary sleep needed
        await expect(element(by.id('dashboard-screen'))).toBeVisible();
        await expect(element(by.text('Good morning, Devesh'))).toBeVisible();
    });

    it('should persist login across app restart', async () => {
        // Login
        await element(by.id('email-input')).typeText('devesh@erp.com');
        await element(by.id('password-input')).typeText('ValidPass123');
        await element(by.id('login-button')).tap();
        await expect(element(by.id('dashboard-screen'))).toBeVisible();

        // Restart app (simulates kill + reopen)
        await device.launchApp({ newInstance: true });

        // Should skip login and go straight to dashboard
        await expect(element(by.id('dashboard-screen'))).toBeVisible();
    });
});
```

---

### Q387. What are Detox matchers and how do you use them?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Detox

**Answer:**
```js
// Finders (by.* — locate elements)
element(by.id('testID-value'))          // testID prop
element(by.text('Exact text'))          // exact text content
element(by.label('accessibility label'))// accessibilityLabel
element(by.type('RCTTextInput'))        // native component type
element(by.traits(['button']))          // accessibilityTraits (iOS)
element(by.hint('What this does'))      // accessibilityHint

// Combining matchers
element(by.id('list').withAncestor(by.id('modal')))
element(by.id('item-1').withDescendant(by.text('Active')))
element(by.id('section').not().withDescendant(by.text('Empty')))

// Index (when multiple matching elements)
element(by.text('Delete')).atIndex(2)  // third "Delete" button

// Actions
await element(by.id('input')).typeText('Hello World');
await element(by.id('input')).clearText();
await element(by.id('input')).replaceText('New text');
await element(by.id('button')).tap();
await element(by.id('button')).longPress();
await element(by.id('button')).longPress(2000); // 2 seconds
await element(by.id('list')).scroll(200, 'down');
await element(by.id('list')).scrollTo('bottom');
await element(by.id('switch')).tap(); // toggle switch
await element(by.id('slider')).adjustSliderToPosition(0.5); // 50%

// Swipe
await element(by.id('card')).swipe('left', 'fast', 0.75);
await element(by.id('screen')).swipe('down', 'slow', 0.5);

// Assertions (expect.*)
await expect(element(by.id('header'))).toBeVisible();
await expect(element(by.id('header'))).toBeNotVisible();
await expect(element(by.id('modal'))).toExist();
await expect(element(by.id('modal'))).toNotExist();
await expect(element(by.text('Hello'))).toHaveText('Hello');
await expect(element(by.id('badge'))).toHaveLabel('3 notifications');
await expect(element(by.id('checkbox'))).toHaveToggleValue(true);
await expect(element(by.id('image'))).toHaveId('profile-photo');
```

---

### Q388. How do you handle Detox test setup and teardown?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Detox

**Answer:**
```js
// e2e/setup/globalSetup.js
module.exports = async () => {
    const { DetoxCircusEnvironment } = require('detox/runners/jest');
    // Global setup runs ONCE before all test suites
};

// e2e/setup/globalTeardown.js
module.exports = async () => {
    // Global teardown runs ONCE after all test suites
};

// e2e/jest.config.js
module.exports = {
    maxWorkers: 1,  // Detox tests run serially (one device)
    testEnvironment: './setup/detoxEnvironment.js',
    globalSetup: './setup/globalSetup.js',
    globalTeardown: './setup/globalTeardown.js',
    testTimeout: 120000,
    verbose: true,
};

// Inside test file — lifecycle hooks
describe('Employee Management', () => {
    beforeAll(async () => {
        // Runs once before all tests in this describe
        await device.launchApp({
            newInstance: true,
            userNotification: null,
            permissions: {
                notifications: 'YES',
                location: 'inuse',
                camera: 'YES',
            },
        });

        // Login once for all tests in this suite
        await loginAsAdmin();
    });

    afterAll(async () => {
        // Cleanup after all tests
        await logoutUser();
    });

    beforeEach(async () => {
        // Reset to known state before each test
        await device.reloadReactNative();
        await navigateTo('EmployeeList');
    });

    afterEach(async () => {
        // Take screenshot on failure (auto in Detox, but manual here)
        if (jasmine.currentTest.failedExpectations.length > 0) {
            await device.takeScreenshot('failure-' + jasmine.currentTest.fullName);
        }
    });

    it('can add a new employee', async () => {
        await element(by.id('add-employee-btn')).tap();
        // ...
    });
});
```

---

### Q389. How do you mock the network in Detox tests?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Detox Network Mocking

**Answer:**
```js
// Option 1: Use a test server (most reliable)
// Run a real mock server during E2E tests

// jest.globalSetup.js
const { createServer } = require('./testServer');
module.exports = async () => {
    global.testServer = createServer();
    await global.testServer.listen(3001);
    process.env.API_URL = 'http://localhost:3001';
};

// jest.globalTeardown.js
module.exports = async () => {
    await global.testServer.close();
};

// testServer.js — using express + JSON fixtures
const express = require('express');
const app = express();

app.get('/api/employees', (req, res) => {
    res.json([
        { id: '1', name: 'Devesh Kumar', department: 'Engineering' },
        { id: '2', name: 'Priya Singh', department: 'HR' },
    ]);
});

app.post('/api/auth/login', (req, res) => {
    const { email, password } = req.body;
    if (email === 'admin@test.com' && password === 'TestPass123') {
        res.json({ token: 'test-jwt-token', user: { name: 'Admin' } });
    } else {
        res.status(401).json({ error: 'Invalid credentials' });
    }
});

// Option 2: Detox Network Mocking (Detox 20+)
it('shows error when server is down', async () => {
    await device.launchApp({
        newInstance: true,
        launchArgs: { mockNetwork: 'error' }, // custom launch arg
    });

    // App reads launch args and uses mock network layer
    await expect(element(by.text('No internet connection'))).toBeVisible();
});

// Option 3: Intercept in app code using launch params
// index.js / App.tsx
if (DevSettings.isSimulator && NativeModules.LaunchArgs?.useMockApi) {
    // Use mock API instead of real API
}
```

---

### Q390. How do you test push notification handling in Detox?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Detox Advanced

**Answer:**
```js
// Detox supports simulating push notifications

describe('Push Notification Handling', () => {
    beforeAll(async () => {
        await device.launchApp({ newInstance: true });
    });

    it('opens attendance screen from notification tap', async () => {
        // Login first
        await loginAsEmployee();

        // Simulate receiving and tapping a push notification
        await device.sendUserNotification({
            // iOS format
            trigger: { type: 'push' },
            title: 'Attendance Reminder',
            body: 'Please mark your attendance',
            badge: 1,
            payload: {
                screen: 'Attendance',
                action: 'checkIn',
            },
        });

        // Should navigate to attendance screen
        await expect(element(by.id('attendance-screen'))).toBeVisible();
    });

    it('shows badge on notification received while app is open', async () => {
        await loginAsEmployee();

        // App is in foreground — notification received
        await device.sendUserNotification({
            trigger: { type: 'push' },
            title: 'New Leave Request',
            body: 'Priya Singh requested 2 days leave',
        });

        // Should show in-app notification banner
        await expect(element(by.id('notification-banner'))).toBeVisible();
        await expect(element(by.text('New Leave Request'))).toBeVisible();
    });

    it('handles notification when app is in background', async () => {
        await loginAsEmployee();

        // Send app to background
        await device.sendToHome();

        // Simulate push + tap (launches app with notification data)
        await device.launchApp({
            newInstance: false,
            userNotification: {
                title: 'Payroll Processed',
                body: 'Your salary has been credited',
                payload: { screen: 'Payslip', month: '2024-01' },
            },
        });

        await expect(element(by.id('payslip-screen'))).toBeVisible();
    });
});
```

---

### Q391. What is Maestro and how does it compare to Detox?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** E2E Testing Tools

**Answer:**
Maestro is a newer E2E testing framework that uses YAML-based test flows. Simpler to set up than Detox, doesn't require building a test binary.

```yaml
# e2e/flows/login.yaml — Maestro test
appId: com.yourcompany.erp
---
- launchApp
- assertVisible: "Login"
- tapOn:
    id: "email-input"
- inputText: "devesh@erp.com"
- tapOn:
    id: "password-input"
- inputText: "ValidPass123"
- tapOn:
    id: "login-button"
- assertVisible: "Good morning, Devesh"
- assertVisible:
    id: "dashboard-screen"
```

```bash
# Run Maestro tests
maestro test e2e/flows/login.yaml
maestro test e2e/flows/  # run all flows in directory
```

**Detox vs Maestro:**

| | Detox | Maestro |
|--|-------|---------|
| Test language | JavaScript | YAML |
| Setup complexity | High (build config) | Low (no build needed) |
| Reliability | Very high (grey-box) | High |
| CI support | ✅ Full | ✅ Maestro Cloud |
| Debugging | JS debugger | maestro studio |
| Custom logic | Full JS | Limited |
| Build required | Yes | No |
| Learning curve | High | Low |

**When to use Maestro:**
- Quick QA smoke tests written by non-developers
- Simple happy-path testing
- Teams new to E2E testing

**When to use Detox:**
- Complex test logic (conditionals, loops)
- Need grey-box synchronisation
- Existing Detox CI infrastructure

---

### Q392. How do you test context and providers?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** RNTL Advanced

**Answer:**
```js
// ThemeContext
const ThemeContext = createContext<{ theme: 'light' | 'dark'; toggle: () => void } | null>(null);

const ThemeProvider = ({ children }) => {
    const [theme, setTheme] = useState<'light' | 'dark'>('light');
    const toggle = useCallback(() => setTheme(t => t === 'light' ? 'dark' : 'light'), []);
    return <ThemeContext.Provider value={{ theme, toggle }}>{children}</ThemeContext.Provider>;
};

// Component using context
const ThemedHeader = () => {
    const { theme, toggle } = useContext(ThemeContext)!;
    return (
        <View testID="header" style={{ backgroundColor: theme === 'light' ? '#fff' : '#111' }}>
            <Text testID="theme-label">{theme}</Text>
            <Pressable testID="toggle-button" onPress={toggle}><Text>Toggle</Text></Pressable>
        </View>
    );
};

// Tests
describe('ThemedHeader', () => {
    // Render with real provider
    const renderWithTheme = (ui: React.ReactNode, initialTheme?: 'light' | 'dark') => {
        const Wrapper = ({ children }) => <ThemeProvider>{children}</ThemeProvider>;
        return render(ui, { wrapper: Wrapper });
    };

    it('shows light theme by default', () => {
        renderWithTheme(<ThemedHeader />);
        expect(screen.getByTestId('theme-label')).toHaveTextContent('light');
    });

    it('toggles to dark theme', async () => {
        renderWithTheme(<ThemedHeader />);
        fireEvent.press(screen.getByTestId('toggle-button'));
        await waitFor(() => {
            expect(screen.getByTestId('theme-label')).toHaveTextContent('dark');
        });
    });

    // Test with mock context (when you want to control context value)
    it('renders with injected theme value', () => {
        const mockContext = { theme: 'dark' as const, toggle: jest.fn() };
        render(
            <ThemeContext.Provider value={mockContext}>
                <ThemedHeader />
            </ThemeContext.Provider>
        );
        expect(screen.getByTestId('theme-label')).toHaveTextContent('dark');
    });
});
```

---

### Q393. How do you test animations in React Native?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Animation Testing

**Answer:**
```js
// Animations are hard to test fully — focus on:
// 1. Component renders correctly before/after animation
// 2. Animated values reach target state
// 3. Callbacks fire (onAnimationComplete, etc.)

// Mock Animated API
jest.mock('react-native', () => {
    const RN = jest.requireActual('react-native');
    // Disable native animations in tests
    RN.Animated.timing = (value, config) => ({
        start: (callback) => {
            value.setValue(config.toValue); // immediately set final value
            callback?.({ finished: true });
        },
    });
    return RN;
});

// Mock Reanimated (jest.setup.js)
jest.mock('react-native-reanimated', () => {
    const Reanimated = require('react-native-reanimated/mock');
    return Reanimated;
});

// Test component that animates in
test('AnimatedModal shows content after animation', async () => {
    const { rerender } = render(<AnimatedModal visible={false} />);
    expect(screen.queryByText('Modal content')).toBeNull();

    rerender(<AnimatedModal visible={true} />);

    // With mocked animation, state update is synchronous
    await waitFor(() => {
        expect(screen.getByText('Modal content')).toBeTruthy();
    });
});

// Test with fake timers
test('loading spinner appears during animation delay', async () => {
    jest.useFakeTimers();
    render(<DelayedLoader delay={500} />);

    expect(screen.queryByTestId('spinner')).toBeNull();

    act(() => { jest.advanceTimersByTime(500); });

    await waitFor(() => {
        expect(screen.getByTestId('spinner')).toBeTruthy();
    });

    jest.useRealTimers();
});
```

---

### Q394. How do you write integration tests for a full screen component?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Integration Testing

**Answer:**
```js
// Full screen integration test — renders the screen with all its dependencies
import { render, screen, fireEvent, waitFor } from '@testing-library/react-native';
import { NavigationContainer } from '@react-navigation/native';
import { Provider } from 'react-redux';
import { AttendanceScreen } from '../screens/AttendanceScreen';
import * as api from '../api';

jest.mock('../api');

const renderAttendanceScreen = (overrides = {}) => {
    const store = createTestStore({
        auth: { user: { id: 'e1', name: 'Devesh Kumar' } },
        ...overrides,
    });

    const navigation = { navigate: jest.fn(), setOptions: jest.fn() };

    return render(
        <Provider store={store}>
            <NavigationContainer>
                <AttendanceScreen navigation={navigation} route={{ params: {} }} />
            </NavigationContainer>
        </Provider>
    );
};

describe('AttendanceScreen Integration', () => {
    beforeEach(() => {
        jest.mocked(api.getAttendanceRecords).mockResolvedValue([
            { id: 'r1', date: '2024-01-15', checkIn: '09:00', checkOut: null, status: 'present' },
        ]);
        jest.mocked(api.checkIn).mockResolvedValue(
            { id: 'r2', date: '2024-01-16', checkIn: '09:15', checkOut: null }
        );
    });

    it('loads and displays attendance records', async () => {
        renderAttendanceScreen();

        // Shows skeleton while loading
        expect(screen.getAllByTestId('skeleton-row')).toHaveLength(3);

        // Wait for data
        await waitFor(() => {
            expect(screen.getByText('Jan 15, 2024')).toBeTruthy();
            expect(screen.getByText('09:00')).toBeTruthy();
        });

        // Skeleton gone
        expect(screen.queryByTestId('skeleton-row')).toBeNull();
    });

    it('completes full check-in flow', async () => {
        renderAttendanceScreen();
        await waitFor(() => expect(screen.queryByTestId('skeleton-row')).toBeNull());

        // Tap check-in
        fireEvent.press(screen.getByTestId('check-in-button'));

        // Confirmation modal appears
        await screen.findByText('Confirm check-in');
        fireEvent.press(screen.getByText('Confirm'));

        // API called
        expect(api.checkIn).toHaveBeenCalledWith('e1');

        // Success feedback shown
        await screen.findByText('Checked in at 09:15');

        // Button changes to check-out
        expect(screen.getByTestId('check-out-button')).toBeTruthy();
        expect(screen.queryByTestId('check-in-button')).toBeNull();
    });

    it('shows error when check-in fails', async () => {
        jest.mocked(api.checkIn).mockRejectedValueOnce(new Error('Already checked in'));
        renderAttendanceScreen();
        await waitFor(() => expect(screen.queryByTestId('skeleton-row')).toBeNull());

        fireEvent.press(screen.getByTestId('check-in-button'));
        await screen.findByText('Confirm check-in');
        fireEvent.press(screen.getByText('Confirm'));

        await screen.findByText('Already checked in');
    });
});
```

---

### Q395. How do you snapshot test in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Snapshot Testing

**Answer:**
```js
// Basic snapshot test
import { render } from '@testing-library/react-native';
import { EmployeeCard } from '../EmployeeCard';

const mockEmployee = {
    id: '1', name: 'Devesh Kumar', role: 'RN Developer', isActive: true,
};

test('EmployeeCard matches snapshot', () => {
    const { toJSON } = render(
        <EmployeeCard employee={mockEmployee} onPress={jest.fn()} onDelete={jest.fn()} />
    );
    expect(toJSON()).toMatchSnapshot();
});
// Creates __snapshots__/EmployeeCard.test.tsx.snap

// Update snapshot when intentional change made:
// jest --updateSnapshot (or -u flag)

// Inline snapshot (snapshot stored in test file)
test('button renders correctly', () => {
    const { toJSON } = render(<Button title="Submit" variant="primary" />);
    expect(toJSON()).toMatchInlineSnapshot(`
        <View
          accessibilityRole="button"
          style={{"backgroundColor": "#6200EE", "borderRadius": 8}}
        >
          <Text style={{"color": "white"}}>Submit</Text>
        </View>
    `);
});

// Snapshot testing best practices:
// ✅ Use for UI regression testing (catch unintended style/structure changes)
// ✅ Snapshot small, focused components
// ✅ Review snapshot diffs carefully in PR — don't blindly update
// ❌ Don't snapshot entire screens (too large, too many false positives)
// ❌ Don't snapshot components with dynamic content (timestamps, random IDs)
```

---

### Q396. How do you test error boundaries?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Error Handling Testing

**Answer:**
```js
// ErrorBoundary component
class ErrorBoundary extends Component {
    state = { hasError: false, error: null };

    static getDerivedStateFromError(error) {
        return { hasError: true, error };
    }

    componentDidCatch(error, info) {
        Sentry.captureException(error, { extra: info });
    }

    render() {
        if (this.state.hasError) {
            return (
                <View testID="error-boundary">
                    <Text>Something went wrong</Text>
                    <Text testID="error-message">{this.state.error?.message}</Text>
                    <Pressable testID="retry-button" onPress={() => this.setState({ hasError: false })}>
                        <Text>Try Again</Text>
                    </Pressable>
                </View>
            );
        }
        return this.props.children;
    }
}

// Component that can throw
const BrokenComponent = ({ shouldThrow }) => {
    if (shouldThrow) throw new Error('Component crashed!');
    return <Text>Working fine</Text>;
};

// Tests
describe('ErrorBoundary', () => {
    // Suppress error output in test console
    const spy = jest.spyOn(console, 'error').mockImplementation(() => {});
    afterAll(() => spy.mockRestore());

    it('renders children normally when no error', () => {
        render(
            <ErrorBoundary>
                <BrokenComponent shouldThrow={false} />
            </ErrorBoundary>
        );
        expect(screen.getByText('Working fine')).toBeTruthy();
        expect(screen.queryByTestId('error-boundary')).toBeNull();
    });

    it('shows error UI when child throws', () => {
        render(
            <ErrorBoundary>
                <BrokenComponent shouldThrow={true} />
            </ErrorBoundary>
        );
        expect(screen.getByTestId('error-boundary')).toBeTruthy();
        expect(screen.getByTestId('error-message')).toHaveTextContent('Component crashed!');
    });

    it('recovers when retry is pressed', async () => {
        const { rerender } = render(
            <ErrorBoundary>
                <BrokenComponent shouldThrow={true} />
            </ErrorBoundary>
        );
        expect(screen.getByTestId('error-boundary')).toBeTruthy();

        fireEvent.press(screen.getByTestId('retry-button'));

        rerender(
            <ErrorBoundary>
                <BrokenComponent shouldThrow={false} />
            </ErrorBoundary>
        );

        await waitFor(() => {
            expect(screen.queryByTestId('error-boundary')).toBeNull();
            expect(screen.getByText('Working fine')).toBeTruthy();
        });
    });
});
```

---

### Q397. How do you run tests in parallel and on CI?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** CI Testing

**Answer:**
```json
// package.json — Jest scripts
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "test:ci": "jest --ci --coverage --runInBand",
    "test:unit": "jest --testPathPattern='__tests__/unit'",
    "test:integration": "jest --testPathPattern='__tests__/integration'"
  }
}
```

```yaml
# GitHub Actions — CI testing
name: Test
on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - name: Run unit + integration tests
        run: npx jest --ci --coverage --maxWorkers=2
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage/lcov.info

  e2e-tests:
    runs-on: macos-latest  # Detox requires macOS for iOS
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - name: Install Detox CLI
        run: npm install -g detox-cli
      - name: Build E2E app
        run: detox build --configuration ios.sim.release
      - name: Run E2E tests
        run: detox test --configuration ios.sim.release --headless --ci
      - name: Upload E2E screenshots (on failure)
        if: failure()
        uses: actions/upload-artifact@v3
        with:
          name: detox-screenshots
          path: artifacts/
```

```js
// jest.config.js — parallel test configuration
module.exports = {
    preset: 'react-native',
    maxWorkers: '50%',     // use half available CPUs
    // OR:
    // maxWorkers: 4,      // fixed number
    // --runInBand         // single process (for CI with limited resources)

    // Test timeout
    testTimeout: 10000,    // 10 seconds per test

    // Fail fast on CI
    bail: process.env.CI ? 1 : 0,  // stop after 1st failure in CI
};
```

---

### Q398. How do you test native module integrations?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Mocking Native Modules

**Answer:**
```js
// Mock the entire native module
jest.mock('react-native', () => {
    const RN = jest.requireActual('react-native');
    return {
        ...RN,
        NativeModules: {
            ...RN.NativeModules,
            BiometricModule: {
                canAuthenticate: jest.fn().mockResolvedValue('FACE_ID'),
                authenticate: jest.fn().mockResolvedValue(true),
            },
            RazorpayModule: {
                startPayment: jest.fn().mockResolvedValue({
                    razorpay_payment_id: 'pay_test_123',
                    razorpay_order_id: 'order_test_456',
                }),
            },
        },
    };
});

// Test component using biometrics
import { NativeModules } from 'react-native';

describe('BiometricLogin', () => {
    it('shows Face ID button when available', async () => {
        NativeModules.BiometricModule.canAuthenticate.mockResolvedValueOnce('FACE_ID');

        render(<BiometricLogin onSuccess={jest.fn()} />);

        await screen.findByText('Login with Face ID');
    });

    it('authenticates successfully', async () => {
        NativeModules.BiometricModule.authenticate.mockResolvedValueOnce(true);
        const onSuccess = jest.fn();

        render(<BiometricLogin onSuccess={onSuccess} />);
        await screen.findByText('Login with Face ID');
        fireEvent.press(screen.getByText('Login with Face ID'));

        await waitFor(() => {
            expect(onSuccess).toHaveBeenCalled();
        });
    });

    it('handles biometric failure gracefully', async () => {
        NativeModules.BiometricModule.authenticate.mockRejectedValueOnce({
            code: 'AUTH_ERROR',
            message: 'Biometric sensor unavailable',
        });

        render(<BiometricLogin onSuccess={jest.fn()} />);
        await screen.findByText('Login with Face ID');
        fireEvent.press(screen.getByText('Login with Face ID'));

        await screen.findByText('Authentication failed. Please try PIN.');
    });
});
```

---

### Q399. What is code coverage and how do you interpret it?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Code Coverage

**Answer:**
```bash
# Generate coverage report
npx jest --coverage

# Output:
# --------------------------------|---------|----------|---------|---------|
# File                            | % Stmts | % Branch | % Funcs | % Lines |
# --------------------------------|---------|----------|---------|---------|
# src/components/EmployeeCard.tsx |   92.31 |    83.33 |     100 |   91.67 |
# src/hooks/useAttendance.ts      |   78.57 |    66.67 |   85.71 |   78.57 |
# src/api/employeeApi.ts          |   100   |    100   |     100 |     100 |
```

```
Coverage types:
─────────────────────────────────────────────────────
Statement coverage  — every executable line run
Branch coverage     — every if/else/switch path taken
Function coverage   — every function called
Line coverage       — every line executed (similar to statement)
```

```js
// package.json — enforce minimum coverage
{
  "jest": {
    "coverageThreshold": {
      "global": {
        "branches": 60,
        "functions": 70,
        "lines": 70,
        "statements": 70
      },
      // Per-file thresholds for critical paths
      "./src/utils/payrollCalculator.ts": {
        "branches": 90,
        "functions": 100,
        "lines": 90,
        "statements": 90
      }
    }
  }
}

// Exclude files from coverage
// jest.config.js
coveragePathIgnorePatterns: [
    '/node_modules/',
    '__mocks__',
    '.stories.tsx',
    'index.ts',       // re-export files — no logic to test
],

// Add istanbul comment to exclude specific lines
const value = isDev
    ? 'development'
    /* istanbul ignore next */
    : 'production';  // line excluded from coverage
```

---

### Q400. How do you test React Native with TypeScript?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** TypeScript + Testing

**Answer:**
```bash
# Additional packages for TypeScript testing
npm install --save-dev @types/jest ts-jest
```

```json
// tsconfig.json — include test types
{
  "compilerOptions": {
    "types": ["jest", "@testing-library/jest-native"],
    "jsx": "react-native"
  },
  "include": ["src", "__tests__", "e2e"]
}
```

```typescript
// Typed mocks
import { fetchEmployee } from '../api/employeeApi';

// jest.mocked() provides full TypeScript type inference
const mockFetchEmployee = jest.mocked(fetchEmployee);
mockFetchEmployee.mockResolvedValue({
    id: '1',
    name: 'Devesh Kumar',
    // TypeScript will error if required fields are missing
});

// Typed render helpers
import { RenderOptions } from '@testing-library/react-native';

const renderWithProviders = (
    ui: React.ReactElement,
    options?: Omit<RenderOptions, 'wrapper'> & {
        preloadedState?: Partial<RootState>;
    }
) => {
    const { preloadedState = {}, ...renderOptions } = options ?? {};
    const store = createTestStore(preloadedState);
    const Wrapper: React.FC<{ children: React.ReactNode }> = ({ children }) => (
        <Provider store={store}>{children}</Provider>
    );
    return { store, ...render(ui, { wrapper: Wrapper, ...renderOptions }) };
};

// Type-safe screen queries
const emailInput = screen.getByTestId<React.ElementRef<typeof TextInput>>('email-input');

// Typed hook test results
const { result } = renderHook<ReturnType<typeof useCounter>, Parameters<typeof useCounter>>(() => useCounter(0));
expect(result.current.count).toBe(0);
```

---

### Q401. How do you test FlatList rendering?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** List Testing

**Answer:**
```js
// FlatList virtualises rendering — only renders visible items
// RNTL renders all items by default (no virtualisation in tests)

test('FlatList renders all employees', async () => {
    const employees = Array.from({ length: 5 }, (_, i) => ({
        id: String(i + 1),
        name: `Employee ${i + 1}`,
    }));

    render(<EmployeeFlatList employees={employees} />);

    // All items rendered in test environment (no virtualisation)
    await waitFor(() => {
        employees.forEach(emp => {
            expect(screen.getByText(emp.name)).toBeTruthy();
        });
    });
});

// Test ListEmptyComponent
test('shows empty state when no data', () => {
    render(<EmployeeFlatList employees={[]} />);
    expect(screen.getByText('No employees found')).toBeTruthy();
    expect(screen.queryByTestId('employee-item')).toBeNull();
});

// Test onEndReached (infinite scroll trigger)
test('calls loadMore when end is reached', async () => {
    const loadMore = jest.fn();
    const employees = Array.from({ length: 10 }, (_, i) => ({
        id: String(i + 1), name: `Employee ${i + 1}`,
    }));

    render(
        <FlatList
            testID="employee-list"
            data={employees}
            renderItem={({ item }) => <Text>{item.name}</Text>}
            keyExtractor={item => item.id}
            onEndReached={loadMore}
            onEndReachedThreshold={0.5}
        />
    );

    // Simulate scroll to end
    fireEvent.scroll(screen.getByTestId('employee-list'), {
        nativeEvent: {
            contentOffset: { y: 500 },
            contentSize: { height: 600, width: 300 },
            layoutMeasurement: { height: 200, width: 300 },
        },
    });

    await waitFor(() => expect(loadMore).toHaveBeenCalled());
});

// Test pull-to-refresh
test('triggers refresh on pull down', async () => {
    const onRefresh = jest.fn();
    render(
        <FlatList
            testID="list"
            data={[{ id: '1', name: 'Item' }]}
            renderItem={({ item }) => <Text>{item.name}</Text>}
            keyExtractor={item => item.id}
            onRefresh={onRefresh}
            refreshing={false}
        />
    );

    fireEvent(screen.getByTestId('list'), 'refresh');
    expect(onRefresh).toHaveBeenCalled();
});
```

---

### Q402. How do you test with fake timers in Jest?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Jest Timers

**Answer:**
```js
// Fake timers control setTimeout, setInterval, Date, etc.

describe('Debounced search', () => {
    beforeEach(() => { jest.useFakeTimers(); });
    afterEach(() => { jest.useRealTimers(); });

    it('debounces API call', async () => {
        const mockSearch = jest.fn().mockResolvedValue([]);
        render(<SearchInput onSearch={mockSearch} debounceMs={300} />);

        // Type rapidly
        fireEvent.changeText(screen.getByTestId('search-input'), 'D');
        fireEvent.changeText(screen.getByTestId('search-input'), 'De');
        fireEvent.changeText(screen.getByTestId('search-input'), 'Dev');

        // API not called yet
        expect(mockSearch).not.toHaveBeenCalled();

        // Advance past debounce window
        act(() => { jest.advanceTimersByTime(300); });

        await waitFor(() => {
            expect(mockSearch).toHaveBeenCalledWith('Dev');
            expect(mockSearch).toHaveBeenCalledTimes(1); // only once
        });
    });
});

// Test setTimeout-based loading state
test('shows skeleton for 500ms minimum', async () => {
    jest.useFakeTimers();
    jest.mocked(fetchData).mockResolvedValue([]);

    render(<DataScreen />);
    expect(screen.getAllByTestId('skeleton')).toHaveLength(3);

    // Data resolves but skeleton should still show
    act(() => { jest.advanceTimersByTime(200); });
    expect(screen.getAllByTestId('skeleton')).toHaveLength(3); // still showing

    // After minimum time passes
    act(() => { jest.advanceTimersByTime(300); }); // total 500ms
    await waitFor(() => {
        expect(screen.queryByTestId('skeleton')).toBeNull();
    });

    jest.useRealTimers();
});

// Test interval
test('updates counter every second', () => {
    jest.useFakeTimers();
    render(<CountdownTimer seconds={5} />);

    expect(screen.getByText('5')).toBeTruthy();

    act(() => { jest.advanceTimersByTime(1000); });
    expect(screen.getByText('4')).toBeTruthy();

    act(() => { jest.advanceTimersByTime(4000); });
    expect(screen.getByText('0')).toBeTruthy();

    jest.useRealTimers();
});
```

---

### Q403. How do you test accessibility in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Accessibility Testing

**Answer:**
```js
// Test accessibility props
test('EmployeeCard has correct accessibility', () => {
    const employee = { id: '1', name: 'Devesh Kumar', role: 'Developer', isActive: true };
    render(<EmployeeCard employee={employee} onPress={jest.fn()} />);

    const card = screen.getByRole('button');
    expect(card).toHaveAccessibleName('Devesh Kumar, Developer, Active');

    // accessibilityHint
    expect(card).toHaveAccessibilityHint('Double tap to view details');
});

// Test screen reader announcements
test('shows accessible state for disabled button', () => {
    render(<Button title="Submit" disabled={true} />);
    const button = screen.getByRole('button', { name: 'Submit' });
    expect(button).toBeDisabled();
    expect(button).toHaveAccessibilityState({ disabled: true });
});

// Test that all interactive elements have labels
test('all buttons have accessible labels', () => {
    render(<ActionBar />);
    const buttons = screen.getAllByRole('button');
    buttons.forEach(button => {
        // Every button should have either a label or a name
        const hasLabel = button.props.accessibilityLabel ||
                         button.props.accessibilityHint ||
                         button.props['aria-label'];
        expect(hasLabel).toBeTruthy();
    });
});

// Test keyboard focus order (for external keyboard users)
test('form fields are in logical tab order', () => {
    render(<LoginForm onSubmit={jest.fn()} />);

    const fields = screen.getAllByRole('textbox');
    // Email should come before password
    expect(fields[0]).toHaveAccessibilityHint('Enter your email address');
    expect(fields[1]).toHaveAccessibilityHint('Enter your password');
});

// axe-core for automated accessibility checks (web-focused but some RN support)
// jest-axe for RN
import { toHaveNoViolations } from 'jest-axe';
expect.extend(toHaveNoViolations);

test('AttendanceCard has no accessibility violations', async () => {
    const { container } = render(<AttendanceCard />);
    const results = await axe(container);
    expect(results).toHaveNoViolations();
});
```

---

### Q404. How do you test deep links?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Deep Link Testing

**Answer:**
```js
// Unit test: URL parsing
import { getStateFromPath } from '@react-navigation/native';

const linkingConfig = {
    screens: {
        Employee: 'employee/:id',
        Payslip: 'payslip/:month',
    },
};

test('parses employee deep link correctly', () => {
    const state = getStateFromPath('/employee/42', linkingConfig);
    expect(state).toMatchObject({
        routes: [{
            name: 'Employee',
            params: { id: '42' },
        }],
    });
});

test('parses payslip deep link', () => {
    const state = getStateFromPath('/payslip/2024-01', linkingConfig);
    expect(state?.routes[0].name).toBe('Payslip');
    expect(state?.routes[0].params).toMatchObject({ month: '2024-01' });
});

// Integration test: Linking.getInitialURL
import { Linking } from 'react-native';

jest.mock('react-native', () => ({
    ...jest.requireActual('react-native'),
    Linking: {
        getInitialURL: jest.fn().mockResolvedValue('yourapp://employee/42'),
        addEventListener: jest.fn(),
    },
}));

test('app opens employee detail from deep link', async () => {
    render(<AppWithNavigation />);
    // NavigationContainer reads Linking.getInitialURL on mount
    await screen.findByTestId('employee-detail-screen');
    // Route should have id: 42
});

// Detox E2E deep link test
it('handles deep link while app is closed', async () => {
    await device.openURL({ url: 'yourapp://employee/42' });
    await expect(element(by.id('employee-detail-screen'))).toBeVisible();
    await expect(element(by.text('Devesh Kumar'))).toBeVisible();
});
```

---

### Q405. What is `userEvent` in RNTL and how does it differ from `fireEvent`?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** RNTL Advanced

**Answer:**
`userEvent` simulates user interactions more realistically than `fireEvent` — it fires multiple events in sequence (like a real user interaction), including pointerDown, pointerUp, press, focus, blur, etc.

```js
import userEvent from '@testing-library/user-event';
// OR from RNTL directly (v12+):
import { userEvent } from '@testing-library/react-native';

// userEvent.setup() — creates a user event instance
const user = userEvent.setup();

// More realistic interactions
test('typing text fires intermediate events', async () => {
    render(<TextInput testID="input" />);

    // fireEvent.changeText — fires onChangeText directly (no intermediate events)
    fireEvent.changeText(screen.getByTestId('input'), 'Hello');

    // userEvent.type — fires keyDown, keyUp, input, change for each character
    await user.type(screen.getByTestId('input'), 'Hello');
    // Also fires: focus on first character, blur check, etc.
});

// Example where difference matters:
test('keyboard submit fires correctly', async () => {
    const onSubmit = jest.fn();
    render(<TextInput testID="input" onSubmitEditing={onSubmit} returnKeyType="done" />);

    // userEvent simulates pressing the "Done" key on keyboard
    await user.type(screen.getByTestId('input'), 'search term{Enter}');

    expect(onSubmit).toHaveBeenCalled();
    // fireEvent.changeText would NOT trigger onSubmitEditing
});

// Press interactions
test('long press triggers menu', async () => {
    const onLongPress = jest.fn();
    render(<Pressable testID="btn" onLongPress={onLongPress}><Text>Hold</Text></Pressable>);

    await user.longPress(screen.getByTestId('btn'));
    expect(onLongPress).toHaveBeenCalled();
});
```

---

### Q406. How do you test a component that uses `useRef`?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Ref Testing

**Answer:**
```js
// Testing refs imperatively
const PasswordInput = React.forwardRef(({ label }, ref) => {
    const [show, setShow] = useState(false);
    const inputRef = useRef<TextInput>(null);

    useImperativeHandle(ref, () => ({
        focus: () => inputRef.current?.focus(),
        clear: () => inputRef.current?.clear(),
        getValue: () => inputRef.current?.props.value,
    }));

    return (
        <View>
            <Text>{label}</Text>
            <TextInput
                ref={inputRef}
                testID="password-input"
                secureTextEntry={!show}
            />
            <Pressable testID="toggle-visibility" onPress={() => setShow(!show)}>
                <Text>{show ? 'Hide' : 'Show'}</Text>
            </Pressable>
        </View>
    );
});

// Test component with ref
test('ref exposes focus and clear methods', () => {
    const ref = React.createRef<{ focus: () => void; clear: () => void }>();
    render(<PasswordInput label="Password" ref={ref} />);

    // Verify ref methods exist
    expect(ref.current?.focus).toBeInstanceOf(Function);
    expect(ref.current?.clear).toBeInstanceOf(Function);
});

test('toggle shows and hides password', async () => {
    render(<PasswordInput label="Password" ref={null} />);

    const input = screen.getByTestId('password-input');
    expect(input.props.secureTextEntry).toBe(true);

    fireEvent.press(screen.getByTestId('toggle-visibility'));
    await waitFor(() => {
        expect(screen.getByTestId('password-input').props.secureTextEntry).toBe(false);
    });

    expect(screen.getByText('Hide')).toBeTruthy();
});
```

---

### Q407. How do you test network requests with `msw` (Mock Service Worker)?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Network Mocking

**Answer:**
```bash
npm install --save-dev msw
```

```js
// src/mocks/handlers.ts — API mock definitions
import { rest } from 'msw';

export const handlers = [
    rest.get('/api/employees', (req, res, ctx) => {
        return res(ctx.json([
            { id: '1', name: 'Devesh Kumar', department: 'Engineering' },
            { id: '2', name: 'Priya Singh', department: 'HR' },
        ]));
    }),

    rest.post('/api/auth/login', async (req, res, ctx) => {
        const { email, password } = await req.json();
        if (email === 'admin@test.com' && password === 'TestPass123') {
            return res(ctx.json({ token: 'test-jwt', user: { name: 'Admin' } }));
        }
        return res(ctx.status(401), ctx.json({ error: 'Invalid credentials' }));
    }),

    rest.get('/api/employees/:id', (req, res, ctx) => {
        const { id } = req.params;
        return res(ctx.json({ id, name: `Employee ${id}`, role: 'Developer' }));
    }),
];
```

```js
// src/mocks/server.ts — test server setup
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

export const server = setupServer(...handlers);
```

```js
// jest.setup.js
import { server } from './src/mocks/server';

beforeAll(() => server.listen({ onUnhandledRequest: 'warn' }));
afterEach(() => server.resetHandlers()); // reset per-test overrides
afterAll(() => server.close());
```

```js
// In tests — use handlers automatically
test('loads employees from API', async () => {
    render(<EmployeeList />);
    await screen.findByText('Devesh Kumar');
    await screen.findByText('Priya Singh');
});

// Override for specific test
test('shows error on API failure', async () => {
    server.use(
        rest.get('/api/employees', (req, res, ctx) =>
            res(ctx.status(500), ctx.json({ error: 'Internal server error' }))
        )
    );

    render(<EmployeeList />);
    await screen.findByText('Failed to load employees');
});
```

---

### Q408. How do you test Reanimated animations?

**Difficulty:** 🔴 Hard | **Frequency:** Medium | **Category:** Reanimated Testing

**Answer:**
```js
// jest.setup.js — mock Reanimated
jest.mock('react-native-reanimated', () => {
    const Reanimated = require('react-native-reanimated/mock');
    // Reanimated mock makes all animations instant
    return Reanimated;
});

// What the Reanimated mock does:
// - useSharedValue returns a plain object { value: initialValue }
// - withTiming, withSpring, etc. immediately set the value
// - useAnimatedStyle returns a plain style object
// - Animated.View renders as View

// Test component with Reanimated
const FadeInView = ({ children, visible }) => {
    const opacity = useSharedValue(0);

    useEffect(() => {
        opacity.value = withTiming(visible ? 1 : 0, { duration: 300 });
    }, [visible]);

    const style = useAnimatedStyle(() => ({ opacity: opacity.value }));

    return <Animated.View style={style}>{children}</Animated.View>;
};

test('FadeInView is opaque when visible', async () => {
    const { rerender } = render(<FadeInView visible={false}><Text>Content</Text></FadeInView>);

    // With mock, animation is instant — check final state
    rerender(<FadeInView visible={true}><Text>Content</Text></FadeInView>);

    await waitFor(() => {
        const view = screen.getByText('Content').parent;
        expect(view.props.style.opacity).toBe(1);
    });
});

// Testing layout animations (entering/exiting)
jest.mock('react-native-reanimated', () => {
    const Reanimated = require('react-native-reanimated/mock');
    // Make entering/exiting animations transparent in tests
    Reanimated.FadeIn = { duration: jest.fn().mockReturnThis(), springify: jest.fn().mockReturnThis() };
    return Reanimated;
});

test('item enters with animation', async () => {
    render(
        <Animated.View entering={FadeIn}>
            <Text>New item</Text>
        </Animated.View>
    );
    // Just test it renders — animation is mocked out
    expect(screen.getByText('New item')).toBeTruthy();
});
```

---

### Q409. How do you write a test for a Redux Saga?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Redux Saga Testing

**Answer:**
```js
// saga under test
function* fetchEmployeesSaga(action) {
    try {
        yield put(setLoading(true));
        const employees = yield call(api.fetchEmployees, action.payload.deptId);
        yield put(setEmployees(employees));
        yield put(setLoading(false));
    } catch (error) {
        yield put(setError(error.message));
        yield put(setLoading(false));
    }
}

// Method 1: Effect-by-effect testing (unit test)
import { call, put } from 'redux-saga/effects';

test('fetchEmployeesSaga happy path', () => {
    const action = { payload: { deptId: 'eng' } };
    const gen = fetchEmployeesSaga(action);

    // Step through each yield
    expect(gen.next().value).toEqual(put(setLoading(true)));
    expect(gen.next().value).toEqual(call(api.fetchEmployees, 'eng'));

    const mockEmployees = [{ id: '1', name: 'Devesh' }];
    expect(gen.next(mockEmployees).value).toEqual(put(setEmployees(mockEmployees)));
    expect(gen.next().value).toEqual(put(setLoading(false)));
    expect(gen.next().done).toBe(true);
});

test('fetchEmployeesSaga error path', () => {
    const action = { payload: { deptId: 'eng' } };
    const gen = fetchEmployeesSaga(action);

    gen.next(); // setLoading(true)
    gen.next(); // call fetchEmployees

    // Simulate error thrown by call effect
    expect(gen.throw(new Error('Network error')).value).toEqual(
        put(setError('Network error'))
    );
    expect(gen.next().value).toEqual(put(setLoading(false)));
});

// Method 2: Integration with runSaga (end-to-end)
import { runSaga } from 'redux-saga';

test('fetchEmployeesSaga dispatches correct actions', async () => {
    const dispatched = [];
    jest.mocked(api.fetchEmployees).mockResolvedValue([{ id: '1', name: 'Devesh' }]);

    await runSaga(
        { dispatch: (action) => dispatched.push(action) },
        fetchEmployeesSaga,
        { payload: { deptId: 'eng' } }
    ).toPromise();

    expect(dispatched).toEqual([
        setLoading(true),
        setEmployees([{ id: '1', name: 'Devesh' }]),
        setLoading(false),
    ]);
});
```

---

### Q410. How do you test a component that uses `useCallback` and `useMemo`?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Performance Hooks Testing

**Answer:**
```js
// You don't test useCallback/useMemo directly —
// you test the BEHAVIOUR they optimise

// Component under test
const SearchableList = ({ items }) => {
    const [query, setQuery] = useState('');

    // useMemo — filtered results
    const filtered = useMemo(
        () => items.filter(i => i.name.toLowerCase().includes(query.toLowerCase())),
        [items, query]
    );

    // useCallback — stable handler for memoised child
    const handleSelect = useCallback((id) => {
        console.log('Selected:', id);
    }, []);

    return (
        <View>
            <TextInput
                testID="search-input"
                value={query}
                onChangeText={setQuery}
                placeholder="Search..."
            />
            <Text testID="result-count">{filtered.length} results</Text>
            {filtered.map(item => (
                <Pressable key={item.id} testID={`item-${item.id}`} onPress={() => handleSelect(item.id)}>
                    <Text>{item.name}</Text>
                </Pressable>
            ))}
        </View>
    );
};

test('filters items based on search query', async () => {
    const items = [
        { id: '1', name: 'Devesh Kumar' },
        { id: '2', name: 'Priya Singh' },
        { id: '3', name: 'Deepak Verma' },
    ];
    render(<SearchableList items={items} />);

    // All shown initially
    expect(screen.getByTestId('result-count')).toHaveTextContent('3 results');

    // Filter by 'Dev'
    fireEvent.changeText(screen.getByTestId('search-input'), 'Dev');
    await waitFor(() => {
        expect(screen.getByTestId('result-count')).toHaveTextContent('2 results');
        expect(screen.getByText('Devesh Kumar')).toBeTruthy();
        expect(screen.getByText('Deepak Verma')).toBeTruthy();
        expect(screen.queryByText('Priya Singh')).toBeNull();
    });

    // Clear filter
    fireEvent.changeText(screen.getByTestId('search-input'), '');
    await waitFor(() => {
        expect(screen.getByTestId('result-count')).toHaveTextContent('3 results');
    });
});
```

---

### Q411. How do you mock `NativeEventEmitter` in tests?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Native Events Testing

**Answer:**
```js
// Mock NativeEventEmitter
jest.mock('react-native', () => {
    const RN = jest.requireActual('react-native');

    // Create a simple mock emitter
    class MockNativeEventEmitter {
        listeners: Record<string, Function[]> = {};

        addListener(event: string, handler: Function) {
            this.listeners[event] = this.listeners[event] || [];
            this.listeners[event].push(handler);
            return { remove: () => this.removeListener(event, handler) };
        }

        removeListener(event: string, handler: Function) {
            this.listeners[event] = (this.listeners[event] || []).filter(h => h !== handler);
        }

        emit(event: string, data: any) {
            (this.listeners[event] || []).forEach(h => h(data));
        }
    }

    return {
        ...RN,
        NativeEventEmitter: MockNativeEventEmitter,
    };
});

// Test component that listens to native events
test('LocationTracker updates on position change', async () => {
    const { NativeEventEmitter } = require('react-native');
    const { NativeModules } = require('react-native');

    render(<LocationTracker />);

    // Get reference to the emitter instance
    const emitter = new NativeEventEmitter(NativeModules.LocationModule);

    // Simulate a native event
    act(() => {
        emitter.emit('onLocationUpdate', {
            latitude: 28.6139,
            longitude: 77.2090,
            accuracy: 10,
        });
    });

    await waitFor(() => {
        expect(screen.getByTestId('lat')).toHaveTextContent('28.6139');
        expect(screen.getByTestId('lng')).toHaveTextContent('77.2090');
    });

    // Simulate another event
    act(() => {
        emitter.emit('onLocationError', { message: 'GPS unavailable' });
    });

    await screen.findByText('GPS unavailable');
});
```

---

### Q412. How do you test components with `Platform.OS`?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Platform Testing

**Answer:**
```js
// Component that has platform-specific behaviour
const ShadowCard = ({ children }) => {
    return (
        <View
            testID="shadow-card"
            style={[
                styles.card,
                Platform.OS === 'ios'
                    ? styles.iosShadow
                    : styles.androidShadow,
            ]}
        >
            {children}
        </View>
    );
};

// Test iOS behaviour
describe('ShadowCard on iOS', () => {
    beforeAll(() => {
        jest.mock('react-native/Libraries/Utilities/Platform', () => ({
            OS: 'ios',
            select: (obj) => obj.ios,
        }));
    });

    it('applies iOS shadow styles', () => {
        render(<ShadowCard><Text>Content</Text></ShadowCard>);
        const card = screen.getByTestId('shadow-card');
        expect(card.props.style).toContainEqual(
            expect.objectContaining({ shadowColor: '#000' })
        );
    });
});

// Better: use Platform mock per-test
describe('ShadowCard', () => {
    it('applies iOS styles on iOS', () => {
        jest.replaceProperty(require('react-native').Platform, 'OS', 'ios');
        render(<ShadowCard><Text>Content</Text></ShadowCard>);
        const card = screen.getByTestId('shadow-card');
        // assert ios styles
    });

    it('applies Android styles on Android', () => {
        jest.replaceProperty(require('react-native').Platform, 'OS', 'android');
        render(<ShadowCard><Text>Content</Text></ShadowCard>);
        const card = screen.getByTestId('shadow-card');
        // assert android styles (elevation instead of shadow)
    });
});

// Even better: use jest.config.js testEnvironmentOptions
// For platform-specific test files:
// Component.ios.test.tsx — jest uses 'ios' preset
// Component.android.test.tsx — jest uses 'android' preset
```

---

### Q413. What is the `debug()` function in RNTL and when do you use it?

**Difficulty:** 🟢 Easy | **Frequency:** Medium | **Category:** RNTL Debugging

**Answer:**
```js
import { render, screen } from '@testing-library/react-native';

// debug() — prints the rendered component tree to console
test('debugging a test', () => {
    render(<LoginForm />);

    // Print entire tree
    screen.debug();

    // Print specific element
    screen.debug(screen.getByTestId('login-button'));

    // Set max depth (default: 6)
    screen.debug({ maxDepth: 3 });
});

// Output looks like:
// <View>
//   <TextInput
//     accessibilityLabel="Email address"
//     placeholder="Email"
//     testID="email-input"
//   />
//   <Pressable
//     accessibilityRole="button"
//     testID="login-button"
//   >
//     <Text>Login</Text>
//   </Pressable>
// </View>

// Common debugging scenarios:
// 1. "Unable to find element" — debug() to see what's actually rendered
test('submit button found', () => {
    render(<Form />);
    screen.debug(); // find the actual testID/text in output
    const button = screen.getByTestId('submit-btn'); // use exact testID from debug output
});

// 2. Accessibility queries failing — debug to check accessible props
test('debug accessibility', () => {
    render(<IconButton icon="delete" accessibilityLabel="Delete item" />);
    screen.debug();
    // See: accessibilityLabel="Delete item" in output
    screen.getByLabelText('Delete item'); // now works
});

// 3. Testing async — debug after waitFor to see final state
await waitFor(() => {
    screen.debug(); // shows what's rendered after async update
    expect(screen.getByText('Loaded')).toBeTruthy();
});
```

---

### Q414. How do you test `useEffect` cleanup?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** Hook Testing

**Answer:**
```js
// Component with cleanup
const WebSocketConnection = ({ url, onMessage }) => {
    useEffect(() => {
        const ws = new WebSocket(url);
        ws.onmessage = (event) => onMessage(event.data);

        return () => {
            ws.close(); // CLEANUP
        };
    }, [url, onMessage]);

    return <View testID="ws-component" />;
};

// Mock WebSocket
const mockWsClose = jest.fn();
const mockWsInstance = {
    close: mockWsClose,
    onmessage: null,
    send: jest.fn(),
};
global.WebSocket = jest.fn().mockImplementation(() => mockWsInstance);

describe('WebSocketConnection', () => {
    beforeEach(() => {
        mockWsClose.mockClear();
        global.WebSocket.mockClear();
    });

    it('opens WebSocket on mount', () => {
        render(<WebSocketConnection url="ws://test.com" onMessage={jest.fn()} />);
        expect(global.WebSocket).toHaveBeenCalledWith('ws://test.com');
    });

    it('closes WebSocket on unmount (cleanup)', () => {
        const { unmount } = render(
            <WebSocketConnection url="ws://test.com" onMessage={jest.fn()} />
        );

        expect(mockWsClose).not.toHaveBeenCalled();

        unmount(); // triggers useEffect cleanup

        expect(mockWsClose).toHaveBeenCalledTimes(1);
    });

    it('reopens WebSocket when URL changes', () => {
        const { rerender } = render(
            <WebSocketConnection url="ws://server1.com" onMessage={jest.fn()} />
        );
        expect(global.WebSocket).toHaveBeenCalledTimes(1);

        rerender(<WebSocketConnection url="ws://server2.com" onMessage={jest.fn()} />);

        // Cleanup of old connection + new connection
        expect(mockWsClose).toHaveBeenCalledTimes(1);
        expect(global.WebSocket).toHaveBeenCalledTimes(2);
        expect(global.WebSocket).toHaveBeenLastCalledWith('ws://server2.com');
    });
});
```

---

### Q415. How do you achieve 80%+ test coverage in a React Native app?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** Testing Strategy

**Answer:**
```
Coverage strategy — what to test at each level:

1. Utility functions (100% coverage goal)
   - formatCurrency, validateEmail, calculateLeaveBalance
   - Pure functions — easy to test, high value

2. Custom hooks (90%+ coverage)
   - useAuth, useAttendance, usePayroll
   - Test all states: loading, success, error

3. Redux/Zustand (90%+ coverage)
   - Reducers, selectors, sagas/thunks
   - All action types + error cases

4. Business-critical components (80%+ coverage)
   - Payment forms, attendance marking, leave requests
   - All user paths including error states

5. UI components (60%+ coverage)
   - Buttons, inputs, cards
   - Focus on behaviour, not visual details

6. Screens (50%+ coverage)
   - Integration tests for main user flows
   - Unit test individual sections
```

```js
// What NOT to test (low value):
// - Inline styles (verify in Storybook)
// - Third-party component props (they test themselves)
// - Static constants
// - Simple re-export files

// High-value tests to add first:
// 1. Authentication flow (login, logout, session expiry)
// 2. Form validation (all error cases)
// 3. API error handling (network failures, 401, 500)
// 4. Navigation guards (unauthenticated redirect)
// 5. Data formatting utilities (currency, dates, names)

// Coverage improvement workflow:
// 1. npm run test:coverage
// 2. Open coverage/lcov-report/index.html
// 3. Find uncovered branches (yellow/red)
// 4. Write tests for uncovered branches
// 5. Repeat

// Quick wins for branch coverage:
// Test both sides of every conditional:
// if (isActive) ... → test isActive=true AND isActive=false
// error && <ErrorText> → test with and without error
// item ?? defaultValue → test with null AND with value
```

---

### Q416. How do you test a component that uses `react-query` / TanStack Query?

**Difficulty:** 🔴 Hard | **Frequency:** High | **Category:** React Query Testing

**Answer:**
```js
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

// Create a fresh QueryClient for each test (prevent cache sharing)
const createTestQueryClient = () =>
    new QueryClient({
        defaultOptions: {
            queries: {
                retry: false, // don't retry failed queries in tests
                gcTime: 0,    // don't cache between tests
            },
        },
    });

// Render wrapper
const renderWithQuery = (ui: React.ReactElement) => {
    const queryClient = createTestQueryClient();
    const Wrapper = ({ children }) => (
        <QueryClientProvider client={queryClient}>
            {children}
        </QueryClientProvider>
    );
    return render(ui, { wrapper: Wrapper });
};

// Component under test
const EmployeeProfile = ({ id }) => {
    const { data, isLoading, isError } = useQuery({
        queryKey: ['employee', id],
        queryFn: () => api.fetchEmployee(id),
    });

    if (isLoading) return <ActivityIndicator testID="loader" />;
    if (isError) return <Text testID="error">Failed to load</Text>;
    return <Text testID="name">{data?.name}</Text>;
};

// Tests
describe('EmployeeProfile', () => {
    it('shows loading state', async () => {
        jest.mocked(api.fetchEmployee).mockReturnValue(new Promise(() => {})); // never resolves
        renderWithQuery(<EmployeeProfile id="1" />);
        expect(screen.getByTestId('loader')).toBeTruthy();
    });

    it('shows employee name on success', async () => {
        jest.mocked(api.fetchEmployee).mockResolvedValue({ id: '1', name: 'Devesh Kumar' });
        renderWithQuery(<EmployeeProfile id="1" />);
        await screen.findByTestId('name');
        expect(screen.getByTestId('name')).toHaveTextContent('Devesh Kumar');
    });

    it('shows error on failure', async () => {
        jest.mocked(api.fetchEmployee).mockRejectedValue(new Error('Not found'));
        renderWithQuery(<EmployeeProfile id="1" />);
        await screen.findByTestId('error');
    });
});
```

---

### Q417. How do you write tests that are resilient to refactoring?

**Difficulty:** 🟡 Medium | **Frequency:** High | **Category:** Testing Best Practices

**Answer:**
```js
// The "testing trophy" principle: test what users care about, not implementation

// ❌ Fragile — tests implementation details
test('sets isLoading to true then false', () => {
    const wrapper = enzyme.mount(<LoginForm />);
    expect(wrapper.state('isLoading')).toBe(false);

    wrapper.find('button').simulate('click');
    expect(wrapper.state('isLoading')).toBe(true);
    // BREAKS if you rename isLoading or move it to Context
});

// ✅ Resilient — tests what user experiences
test('shows loading indicator during login', async () => {
    jest.mocked(api.login).mockReturnValue(new Promise(resolve => setTimeout(resolve, 100)));
    render(<LoginForm />);

    fireEvent.press(screen.getByText('Login'));

    // User sees loading spinner
    expect(screen.getByTestId('loading-spinner')).toBeTruthy();

    await waitFor(() => expect(screen.queryByTestId('loading-spinner')).toBeNull());
    // SURVIVES refactoring state management
});

// Resilience checklist:
// ✅ Query by accessibility role/label (not internal IDs)
screen.getByRole('button', { name: 'Submit' })  // resilient
screen.getByTestId('submit-btn')                 // OK, but change testID = test breaks

// ✅ Test outcomes, not implementation
expect(onSubmit).toHaveBeenCalledWith(formData)  // resilient
expect(component.state.submitted).toBe(true)     // fragile

// ✅ Avoid querying by component class name
screen.getByRole('textbox')                  // resilient
screen.getByType(TextInput)                  // fragile (Enzyme-style)

// ✅ Don't test prop drilling
// Don't test that ChildComponent receives 'onPress' prop
// Test that pressing the button calls the expected action
```

---

### Q418. How do you set up test coverage gates in CI/CD?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** CI Testing

**Answer:**
```json
// package.json — coverage thresholds (fails CI if below)
{
  "jest": {
    "coverageThreshold": {
      "global": {
        "branches": 60,
        "functions": 70,
        "lines": 70,
        "statements": 70
      },
      "./src/utils/": {
        "branches": 90,
        "functions": 100
      },
      "./src/api/": {
        "branches": 80,
        "functions": 90
      }
    }
  }
}
```

```yaml
# GitHub Actions — coverage gate + reporting
- name: Run tests with coverage
  run: npx jest --ci --coverage --coverageReporters=json --coverageReporters=lcov

- name: Check coverage thresholds
  run: |
    COVERAGE=$(cat coverage/coverage-summary.json | jq '.total.lines.pct')
    echo "Line coverage: $COVERAGE%"
    if (( $(echo "$COVERAGE < 70" | bc -l) )); then
      echo "Coverage below threshold!"
      exit 1
    fi

- name: Upload to Codecov
  uses: codecov/codecov-action@v3
  with:
    file: ./coverage/lcov.info
    fail_ci_if_error: true
    threshold: 70   # Codecov also enforces threshold

- name: Comment coverage on PR
  uses: romeovs/lcov-reporter-action@v0.3.1
  with:
    lcov-file: ./coverage/lcov.info
    github-token: ${{ secrets.GITHUB_TOKEN }}
```

```bash
# Local coverage check before push
npm run test:coverage -- --passWithNoTests
# Exit code 1 if below threshold → CI will catch it
```

---

### Q419. How do you test internationalization (i18n) in React Native?

**Difficulty:** 🟡 Medium | **Frequency:** Medium | **Category:** i18n Testing

**Answer:**
```js
// Mock i18n library (react-i18next)
jest.mock('react-i18next', () => ({
    useTranslation: () => ({
        t: (key: string, params?: object) => {
            // Simple mock — return key or formatted string
            if (params) {
                return `${key}(${JSON.stringify(params)})`;
            }
            return key;
        },
        i18n: { language: 'en', changeLanguage: jest.fn() },
    }),
    Trans: ({ children }) => children,
    withTranslation: () => (Component) => Component,
}));

// Test with actual translations (integration test)
import i18n from '../src/i18n';
import { I18nextProvider } from 'react-i18next';

const renderWithI18n = (ui: React.ReactElement, locale = 'en') => {
    const testI18n = i18n.cloneInstance();
    testI18n.changeLanguage(locale);

    return render(
        <I18nextProvider i18n={testI18n}>{ui}</I18nextProvider>
    );
};

test('renders in English', () => {
    renderWithI18n(<LoginScreen />, 'en');
    expect(screen.getByText('Login')).toBeTruthy();
    expect(screen.getByPlaceholderText('Email address')).toBeTruthy();
});

test('renders in Hindi', () => {
    renderWithI18n(<LoginScreen />, 'hi');
    expect(screen.getByText('लॉगिन')).toBeTruthy();
});

// Test locale-specific formatting
test('formatDate uses correct locale', () => {
    renderWithI18n(<AttendanceDate date={new Date('2024-01-15')} />, 'en-IN');
    expect(screen.getByText('15 Jan 2024')).toBeTruthy(); // Indian date format
});

// Test RTL layout switching (for Arabic/Hindi)
test('layout mirrors for RTL locale', () => {
    renderWithI18n(<NavigationHeader />, 'ar');
    const header = screen.getByTestId('header');
    expect(header.props.style).toContainEqual({ flexDirection: 'row-reverse' });
});
```

---

### Q420. What is the best testing strategy for a production React Native app?

**Difficulty:** 🔴 Hard | **Frequency:** Very High | **Category:** Testing Strategy

**Answer:**
```
Production testing strategy for a 10-12 LPA React Native role:

Layer 1 — Static Analysis (fast, free, always on)
├── TypeScript — type errors caught at compile time
├── ESLint — code quality, common bugs
└── Prettier — consistent formatting

Layer 2 — Unit Tests (fast, ~70% of tests)
├── All utility functions (100% coverage)
├── All Redux reducers + selectors (100% coverage)
├── All custom hooks (90%+ coverage)
├── All API service functions (90%+ coverage)
└── Run on every commit — ~30 seconds

Layer 3 — Integration Tests (medium, ~20% of tests)
├── Complete screens with mocked API
├── Full user flows (login, checkout, form submission)
├── All error states and edge cases
└── Run on every PR — ~3 minutes

Layer 4 — E2E Tests (slow, ~10% of tests)
├── Login and authentication
├── Core user journeys (5-8 flows max)
├── Regression suite for critical paths
└── Run nightly / pre-release — ~30 minutes

Layer 5 — Manual QA (exploratory)
├── New features on real devices
├── Edge cases on different OS versions
└── Performance testing on low-end devices
```

```js
// Practical implementation tips for your ERP app:

// 1. Critical paths to always test:
const criticalPaths = [
    'Employee login/logout and session management',
    'Attendance check-in/check-out',
    'Leave request submission and approval',
    'Payslip generation and download',
    'Biometric authentication flow',
];

// 2. Test file co-location (preferred)
// src/components/EmployeeCard/
//   EmployeeCard.tsx
//   EmployeeCard.test.tsx  ← co-located
//   EmployeeCard.stories.tsx

// 3. CI gates
// PR → unit + integration must pass (< 5 min)
// Merge to main → full suite including E2E (< 30 min)
// Release → manual QA + performance testing on devices

// 4. Coverage targets (realistic for 10-12 LPA level)
// utils/: 95%
// hooks/: 85%
// store/: 80%
// components/: 70%
// screens/: 60%
// Overall: 70%+

// 5. What to say in interviews about testing:
// "We had 68% coverage with RNTL + Jest.
//  Detox covered our 5 critical user journeys.
//  We ran tests in CI with GitHub Actions —
//  unit tests on every PR, E2E on every release.
//  This helped us catch a payroll calculation bug
//  that would have affected salary disbursement."
```

---

## Sections Overview (Q421–Q500)

| Section | Questions | Topics |
|---------|-----------|--------|
| Debugging | Q421–Q450 | Flipper, Hermes debugger, crash reporting |
| Storage & Permissions | Q451–Q480 | AsyncStorage, Keychain, permission flows |
| Miscellaneous | Q481–Q500 | Accessibility, internationalisation, misc APIs |

---

> 💡 **Tip for GitHub:** Add a `## Table of Contents` section at the top with anchor links to each question for easy navigation.

---

*Part 01 of 8 — [← Back to Part README](./README.md) · [← Main README](../README.md)*
