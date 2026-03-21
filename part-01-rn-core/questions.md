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
