
# Part 03 — State Management (Redux / Saga / Zustand / Context)
> **React Native Interview Prep · 10–12 LPA**  
> 500 Questions · Answers · Code Snippets  
> Covers: Redux, Redux-Saga, Redux-Thunk, Zustand, Context API, Selectors, Middleware, Effects Patterns

---

## 📁 Folder Structure
```
part-03-state-management/
├── README.md                  ← You are here
├── 01-redux-core.md
├── 02-redux-saga.md
├── 03-redux-thunk.md
├── 04-context-api.md
├── 05-zustand.md
├── 06-selectors-reselect.md
├── 07-middleware.md
├── 08-effects-patterns.md
└── 09-interview-scenarios.md
```

---

## 📊 Question Distribution

| Topic | Questions |
|---|---|
| Redux Core | 80 |
| Redux-Saga | 100 |
| Redux-Thunk | 40 |
| Context API | 50 |
| Zustand | 40 |
| Selectors & Reselect | 40 |
| Middleware | 50 |
| Effects Patterns | 50 |
| Interview Scenarios (Real-world) | 50 |
| **Total** | **500** |

---

## 🔴 SECTION 1 — Redux Core (80 Questions)

---

### Q1. What is Redux and why is it used in React Native?
**Answer:** Redux is a predictable state container for JavaScript applications. It stores the entire app state in a single immutable object (store), making debugging, testing, and state prediction much easier — especially critical in large RN apps like ERPs with 100+ screens.

---

### Q2. What are the three core principles of Redux?
**Answer:**
1. **Single source of truth** — entire state in one store
2. **State is read-only** — change via actions only
3. **Changes made with pure functions** — reducers

---

### Q3. What is a Redux store?
**Answer:** The store holds the complete state tree of the app. Created via `configureStore` (Redux Toolkit) or `createStore` (legacy).

```js
import { configureStore } from '@reduxjs/toolkit';
import rootReducer from './rootReducer';

const store = configureStore({ reducer: rootReducer });
export default store;
```

---

### Q4. What is an action in Redux?
**Answer:** A plain JS object describing what happened. Must have a `type` property.

```js
const loginAction = {
  type: 'auth/login',
  payload: { userId: 'u123', token: 'abc' }
};
```

---

### Q5. What is a reducer?
**Answer:** A pure function that takes current state + action and returns new state. Never mutates state directly.

```js
const authReducer = (state = initialState, action) => {
  switch (action.type) {
    case 'auth/login':
      return { ...state, user: action.payload, isLoggedIn: true };
    default:
      return state;
  }
};
```

---

### Q6. What is the difference between `createStore` and `configureStore`?
**Answer:** `configureStore` (Redux Toolkit) automatically sets up Redux DevTools, applies thunk middleware, and uses Immer for immutable state updates. `createStore` is the legacy approach requiring manual setup.

---

### Q7. What is `combineReducers`?
**Answer:** Merges multiple slice reducers into a single root reducer.

```js
import { combineReducers } from 'redux';
import authReducer from './authSlice';
import erpReducer from './erpSlice';

const rootReducer = combineReducers({
  auth: authReducer,
  erp: erpReducer,
});
```

---

### Q8. What is Redux Toolkit (RTK)?
**Answer:** The official, opinionated toolset for Redux. Includes `createSlice`, `createAsyncThunk`, `configureStore`, and `createEntityAdapter`. Eliminates boilerplate from classic Redux.

---

### Q9. What is `createSlice`?
**Answer:** Automatically generates action creators and action types from reducer functions.

```js
import { createSlice } from '@reduxjs/toolkit';

const authSlice = createSlice({
  name: 'auth',
  initialState: { user: null, token: null },
  reducers: {
    setUser: (state, action) => { state.user = action.payload; },
    logout: (state) => { state.user = null; state.token = null; },
  },
});

export const { setUser, logout } = authSlice.actions;
export default authSlice.reducer;
```

---

### Q10. Why must reducers be pure functions?
**Answer:** Pure functions have no side effects — same input always returns same output. This guarantees predictable state updates, enables time-travel debugging, and makes testing easy.

---

### Q11. What is the Redux data flow?
**Answer:**
```
UI dispatches Action
  → Middleware processes it
    → Reducer creates new state
      → Store updates
        → Connected components re-render
```

---

### Q12. What is `dispatch`?
**Answer:** A function that sends an action to the Redux store to trigger a state change.

```js
import { useDispatch } from 'react-redux';
const dispatch = useDispatch();
dispatch(setUser({ name: 'Devesh' }));
```

---

### Q13. What is `useSelector`?
**Answer:** A React-Redux hook that reads state from the Redux store. Re-renders the component only when the selected value changes.

```js
const user = useSelector((state) => state.auth.user);
```

---

### Q14. What is `connect()` and how does it differ from hooks?
**Answer:** `connect()` is the legacy HOC to link a component to the Redux store via `mapStateToProps` and `mapDispatchToProps`. Hooks (`useSelector`, `useDispatch`) are cleaner and the modern standard.

---

### Q15. What is shallow equality in `useSelector`?
**Answer:** By default `useSelector` uses strict (`===`) equality. When selecting objects, use `shallowEqual` to avoid unnecessary re-renders.

```js
import { shallowEqual, useSelector } from 'react-redux';
const { user, token } = useSelector((state) => state.auth, shallowEqual);
```

---

### Q16. What is `initialState` in a reducer?
**Answer:** The default value of the state slice if the reducer has never been called. Always define it explicitly.

---

### Q17. How do you handle async actions in Redux without Saga?
**Answer:** Use `createAsyncThunk` (RTK) or manual thunks. Saga is preferred for complex async flows.

```js
export const fetchUser = createAsyncThunk('user/fetch', async (id) => {
  const res = await api.getUser(id);
  return res.data;
});
```

---

### Q18. What are `extraReducers` in RTK?
**Answer:** Handle actions from outside the slice — like async thunk lifecycle actions.

```js
extraReducers: (builder) => {
  builder
    .addCase(fetchUser.pending, (state) => { state.loading = true; })
    .addCase(fetchUser.fulfilled, (state, action) => {
      state.loading = false;
      state.user = action.payload;
    })
    .addCase(fetchUser.rejected, (state, action) => {
      state.loading = false;
      state.error = action.error.message;
    });
}
```

---

### Q19. What is the purpose of `Provider` in React-Redux?
**Answer:** Makes the Redux store available to all nested components via React context.

```jsx
import { Provider } from 'react-redux';

```

---

### Q20. What causes unnecessary re-renders in Redux-connected components?
**Answer:** Selecting entire state objects, not using `shallowEqual`, or creating new object/array references inside selectors. Fix with memoized selectors (Reselect).

---

### Q21. What is action creator? Why use it?
**Answer:** A function that returns an action object. Reduces typos and centralizes action shape.

```js
const login = (user) => ({ type: 'auth/login', payload: user });
```

---

### Q22. What is `bindActionCreators`?
**Answer:** Wraps action creators so dispatching happens automatically — used in `mapDispatchToProps`.

---

### Q23. How does Redux DevTools work?
**Answer:** Intercepts dispatched actions and state snapshots. Supports time-travel debugging — you can step forward/backward through state changes. Enabled automatically via `configureStore`.

---

### Q24. What is `normalize` state shape?
**Answer:** Storing data by ID in a flat object instead of nested arrays, similar to a database. Prevents duplication and simplifies updates.

```js
// Bad
state.users = [{ id: 1, name: 'Dev' }, { id: 2, name: 'Ram' }]

// Good (normalized)
state.users = { 1: { id: 1, name: 'Dev' }, 2: { id: 2, name: 'Ram' } }
state.userIds = [1, 2]
```

---

### Q25. What is `createEntityAdapter`?
**Answer:** RTK utility that generates normalized state + CRUD reducers automatically.

```js
const usersAdapter = createEntityAdapter();
const initialState = usersAdapter.getInitialState();
```

---

### Q26. How do you reset Redux state on logout?
**Answer:** Intercept the logout action in the root reducer and return `undefined` to reset all slices.

```js
const rootReducer = (state, action) => {
  if (action.type === 'auth/logout') state = undefined;
  return appReducer(state, action);
};
```

---

### Q27. What is an immutable state update? Give an example.
**Answer:** Never mutate state directly. Return a new object. RTK uses Immer under the hood, so you can write "mutating" syntax safely.

```js
// RTK slice (Immer allows this safely)
setUser: (state, action) => { state.user = action.payload; }

// Classic Redux (must spread)
case 'SET_USER': return { ...state, user: action.payload };
```

---

### Q28. What is `Immer` and how does RTK use it?
**Answer:** Immer creates a draft copy of state, applies mutations, then produces a new immutable object. RTK wraps reducers in Immer automatically, so you can write `state.user = x` instead of spreading.

---

### Q29. Difference between local state and Redux state?
**Answer:**

| Local State | Redux State |
|---|---|
| Component-specific | Global / shared |
| `useState` | Redux store |
| Ephemeral UI state | Persistent business data |
| Fast, no boilerplate | Requires setup |

Use local state for form inputs, modals. Use Redux for auth, API data, app-wide config.

---

### Q30. When should you NOT use Redux?
**Answer:** For purely local UI state (modal open/close, tab index, input text). Overusing Redux adds unnecessary boilerplate. Use `useState` or `useReducer` for component-scoped state.

---

### Q31. What is the Flux architecture and how does Redux relate to it?
**Answer:** Flux is Facebook's pattern: Action → Dispatcher → Store → View. Redux simplifies it: single store, no dispatcher — just pure reducers.

---

### Q32. What is `applyMiddleware`?
**Answer:** Extends Redux dispatch functionality. Used to add Saga, Thunk, logger, etc.

```js
const store = createStore(rootReducer, applyMiddleware(sagaMiddleware, logger));
```

---

### Q33. What is a Redux selector?
**Answer:** A function that extracts a specific piece of state.

```js
const selectUser = (state) => state.auth.user;
const selectToken = (state) => state.auth.token;
```

---

### Q34. What is memoization in selectors?
**Answer:** Caching selector results so they recompute only when inputs change. Implemented via Reselect's `createSelector`.

---

### Q35. What is a "duck" pattern in Redux?
**Answer:** Bundling actions, action types, and reducers for one feature in a single file — similar to what RTK's `createSlice` achieves.

---

### Q36. What is `store.getState()`?
**Answer:** Returns the current state snapshot. Rarely used in components (use `useSelector`), but useful in Saga channels or utility functions.

---

### Q37. What is `store.subscribe()`?
**Answer:** Registers a callback that fires after every state change. Used for persistence (e.g., saving to AsyncStorage).

---

### Q38. How do you persist Redux state in React Native?
**Answer:** Use `redux-persist` with AsyncStorage.

```js
import AsyncStorage from '@react-native-async-storage/async-storage';
import { persistStore, persistReducer } from 'redux-persist';

const persistConfig = { key: 'root', storage: AsyncStorage };
const persistedReducer = persistReducer(persistConfig, rootReducer);
```

---

### Q39. What is the difference between `action.type` and `action.payload`?
**Answer:** `type` identifies what happened (string). `payload` carries the data needed to perform the state update.

---

### Q40. How do you structure Redux for a large ERP app?
**Answer:** Feature-based slices: each domain (HRMS, CRM, Payroll) has its own slice file. Root reducer combines all.

```
store/
  auth/authSlice.js
  hrms/hrmsSlice.js
  crm/crmSlice.js
  payroll/payrollSlice.js
  rootReducer.js
  store.js
```

---

### Q41. What is Redux middleware?
**Answer:** Code that sits between dispatch and the reducer, enabling side effects, logging, async handling.

---

### Q42. What is the difference between Redux and MobX?
**Answer:** Redux uses immutable state and explicit actions. MobX uses observable mutable state and automatic reactions. Redux is more predictable; MobX has less boilerplate.

---

### Q43. What is RTK Query?
**Answer:** A data fetching and caching tool built into Redux Toolkit. Auto-generates hooks, manages loading/error states, caches responses, and invalidates cache. Alternative to Saga for simple CRUD.

---

### Q44. When would you use RTK Query over Redux-Saga?
**Answer:** RTK Query for straightforward CRUD API calls. Saga for complex async workflows — parallel calls, race conditions, cancellable tasks, and long-running background processes like your ERP sync flows.

---

### Q45. What is `prepareCallback` in `createSlice`?
**Answer:** Customizes the action creator to shape the payload before it reaches the reducer.

```js
addItem: {
  reducer: (state, action) => { state.items.push(action.payload); },
  prepare: (name, qty) => ({ payload: { id: Date.now(), name, qty } }),
}
```

---

### Q46. How do you type Redux with TypeScript?
**Answer:**

```ts
import { RootState, AppDispatch } from './store';
import { TypedUseSelectorHook, useSelector, useDispatch } from 'react-redux';

export const useAppDispatch = () => useDispatch();
export const useAppSelector: TypedUseSelectorHook = useSelector;
```

---

### Q47. What is `RootState` type?
**Answer:** TypeScript type inferred from the store's root reducer. Used to type `useSelector`.

```ts
export type RootState = ReturnType;
```

---

### Q48. What is `AppDispatch` type?
**Answer:** TypeScript type for dispatch, including middleware types (like Thunk/Saga dispatch).

```ts
export type AppDispatch = typeof store.dispatch;
```

---

### Q49. How do you test Redux reducers?
**Answer:** Pure functions — just call them with state + action and assert the output.

```js
it('sets user on login', () => {
  const newState = authReducer(undefined, setUser({ name: 'Dev' }));
  expect(newState.user.name).toBe('Dev');
});
```

---

### Q50. What is the `initialState` pattern for API data in Redux?

```js
const initialState = {
  data: null,
  loading: false,
  error: null,
};
```

---

### Q51–80. Additional Redux Core Questions

**Q51.** What is the difference between `useSelector` and `mapStateToProps`?  
**A:** Both read from the store. `useSelector` is hook-based and simpler. `mapStateToProps` is legacy HOC pattern.

**Q52.** Can you have multiple Redux stores in one app?  
**A:** Technically yes, but strongly discouraged. Use one store with multiple slices instead.

**Q53.** What is action batching in Redux?  
**A:** React 18+ batches multiple dispatches into one render. In older versions, use `unstable_batchedUpdates`.

**Q54.** What is `redux-logger` middleware?  
**A:** Logs every action and state diff to console. Remove in production builds.

**Q55.** How do you conditionally apply middleware?  
**A:** Check `process.env.NODE_ENV !== 'production'` before adding logger/devtools.

**Q56.** What is an action creator factory?  
**A:** A function returning action creators dynamically — useful for paginated endpoints.

**Q57.** How does RTK handle serializable check?  
**A:** RTK warns if non-serializable values (Dates, functions) are put in state. Disable selectively with `serializableCheck` config.

**Q58.** What is `createListenerMiddleware` in RTK?  
**A:** Allows responding to dispatched actions with side-effects, similar to a lightweight Saga.

**Q59.** What happens if a reducer throws an error?  
**A:** Redux catches it, state remains unchanged, and error propagates. Always guard with try/catch in sagas or thunks.

**Q60.** What is `store enhancer`?  
**A:** Higher-order function that augments the store — DevTools, persistence, and middleware use this pattern.

**Q61.** How do you handle optimistic updates in Redux?  
**A:** Update state immediately on action, then revert on API failure.

**Q62.** What is `createReducer` in RTK?  
**A:** A utility to write reducers with `builder` notation, without `createSlice`.

**Q63.** How do you avoid selector recalculation?  
**A:** Use `createSelector` to memoize derived values.

**Q64.** What is `HYDRATE` action in Next.js + Redux?  
**A:** Used with `next-redux-wrapper` to sync server-side state to client Redux store.

**Q65.** How do you handle pagination state in Redux?  
**A:** Store `page`, `hasMore`, `loading`, `data[]` per feature slice.

**Q66.** What is action creator middleware?  
**A:** Middleware that intercepts action creators (not plain objects) — used by Thunk.

**Q67.** Difference between `state.entities` and `state.ids` in `createEntityAdapter`?  
**A:** `entities` is a map `{id: item}`, `ids` is ordered array of IDs.

**Q68.** What are adapter selectors?  
**A:** Pre-built selectors from `createEntityAdapter`: `selectAll`, `selectById`, `selectTotal`.

**Q69.** How do you handle loading spinners with Redux?  
**A:** Each slice has a `loading` boolean. Components read it with `useSelector`.

**Q70.** How do you prevent double API calls on mount?  
**A:** Check if data already exists in Redux state before dispatching a fetch action.

**Q71.** What is `Rehydration` in redux-persist?  
**A:** Restoring persisted state from AsyncStorage into Redux on app launch.

**Q72.** What is `PURGE` in redux-persist?  
**A:** Clears persisted state (used on logout).

**Q73.** What is `PAUSE`/`PERSIST`/`FLUSH`/`REHYDRATE` in redux-persist?  
**A:** Lifecycle action types to control persistence flow.

**Q74.** How do you migrate persisted state schema changes?  
**A:** Use `migrations` config in `redux-persist` to transform old state shapes.

**Q75.** What is `storeRef` pattern?  
**A:** Storing a reference to the Redux store outside React for use in non-component utilities (e.g., Axios interceptors, as used in your ERP).

**Q76.** How do you dispatch from outside a component in RN?  
**A:** Export the store and call `store.dispatch(action)` directly.

**Q77.** What is `onRehydrationComplete` in redux-persist?  
**A:** Callback fired after persisted state is fully loaded — use to show splash screen or navigate.

**Q78.** Difference between `state` and `draft` in Immer?  
**A:** `draft` is the mutable copy Immer creates. After the reducer runs, Immer produces a new immutable `state` from the draft.

**Q79.** What is action deduplication?  
**A:** Preventing the same action from being dispatched multiple times in quick succession — use debounce in sagas or UI handlers.

**Q80.** What is the most common Redux anti-pattern?  
**A:** Storing derived/computed values (like totals, filtered lists) in Redux state instead of computing them via selectors.

---

## 🟡 SECTION 2 — Redux-Saga (100 Questions)

---

### Q81. What is Redux-Saga?
**Answer:** A Redux middleware library using ES6 Generators to handle side effects (API calls, timers, background tasks) in a declarative, testable way.

---

### Q82. What is a Generator function?
**Answer:** A function that can pause execution and resume later, using `yield`.

```js
function* mySaga() {
  const data = yield call(fetchUser, userId);
  yield put(setUser(data));
}
```

---

### Q83. What is `call` effect in Saga?
**Answer:** Calls a function (usually async) and waits for it to resolve. Blocking.

```js
const response = yield call(api.getEmployee, empId);
```

---

### Q84. What is `put` effect?
**Answer:** Dispatches a Redux action from inside a saga.

```js
yield put(setEmployees(response.data));
```

---

### Q85. What is `take` effect?
**Answer:** Blocks the saga until a specific action is dispatched.

```js
yield take('auth/loginRequest');
```

---

### Q86. What is `takeEvery`?
**Answer:** Forks a new saga for every matching action. Does not cancel previous runs.

```js
yield takeEvery('FETCH_EMPLOYEE', fetchEmployeeSaga);
```

---

### Q87. What is `takeLatest`?
**Answer:** Cancels previous saga if a new matching action arrives. Best for search, fetch-on-tab-change.

```js
yield takeLatest('SEARCH_EMPLOYEES', searchSaga);
```

---

### Q88. What is `takeLeading`?
**Answer:** Ignores new actions until the current saga finishes. Best for login/form submit — prevents double submissions.

```js
yield takeLeading('auth/login', loginSaga);
```

---

### Q89. Difference between `takeEvery`, `takeLatest`, `takeLeading`?

| | New action while running |
|---|---|
| `takeEvery` | Runs both (parallel) |
| `takeLatest` | Cancels old, runs new |
| `takeLeading` | Ignores new, keeps old |

---

### Q90. What is `fork` effect?
**Answer:** Starts a non-blocking child saga. Parent continues immediately.

```js
yield fork(syncInventorySaga);
yield fork(syncAttendanceSaga);
```

---

### Q91. What is `spawn` effect?
**Answer:** Like `fork` but fully detached — parent's cancellation does not affect spawned saga.

---

### Q92. What is `all` effect?
**Answer:** Runs multiple effects in parallel, waits for all to complete.

```js
const [employees, payroll] = yield all([
  call(api.getEmployees),
  call(api.getPayroll),
]);
```

---

### Q93. What is `race` effect?
**Answer:** Runs multiple effects in parallel, resolves with the first to finish and cancels the rest.

```js
const { data, timeout } = yield race({
  data: call(api.fetchDashboard),
  timeout: delay(5000),
});
if (timeout) yield put(showTimeoutError());
```

---

### Q94. What is `call` vs `apply`?
**Answer:** `call(fn, args)` calls `fn` with args. `apply(context, fn, args)` calls with a specific `this` context.

---

### Q95. What is `select` effect?
**Answer:** Reads current Redux state from inside a saga.

```js
const token = yield select((state) => state.auth.token);
```

---

### Q96. What is `delay` effect?
**Answer:** Pauses saga execution for a given duration.

```js
yield delay(2000); // wait 2 seconds
```

---

### Q97. How do you cancel a saga task?
**Answer:** Use `cancel` effect with a forked task reference.

```js
const task = yield fork(longRunningSaga);
yield take('CANCEL_TASK');
yield cancel(task);
```

---

### Q98. What is `cancelled` effect?
**Answer:** Returns `true` if the saga was cancelled — used in `finally` blocks for cleanup.

```js
try {
  yield call(api.fetch);
} finally {
  if (yield cancelled()) yield put(resetLoading());
}
```

---

### Q99. What is a watcher saga?
**Answer:** A saga that listens for actions and spawns worker sagas.

```js
function* watchAuth() {
  yield takeLatest('auth/loginRequest', loginWorkerSaga);
  yield takeLatest('auth/logout', logoutSaga);
}
```

---

### Q100. What is a worker saga?
**Answer:** Does the actual async work — API calls, dispatches, error handling.

```js
function* loginWorkerSaga(action) {
  try {
    const data = yield call(api.login, action.payload);
    yield put(loginSuccess(data));
  } catch (e) {
    yield put(loginFailure(e.message));
  }
}
```

---

### Q101. How do you structure sagas in a large RN app like ERP?

```
sagas/
  auth/authSaga.js
  hrms/hrmsSaga.js
  crm/crmSaga.js
  payroll/payrollSaga.js
  rootSaga.js
```

```js
// rootSaga.js
export default function* rootSaga() {
  yield all([
    fork(authSaga),
    fork(hrmsSaga),
    fork(crmSaga),
  ]);
}
```

---

### Q102. How do you test a saga?
**Answer:** Test by asserting effects step by step — no actual API calls needed.

```js
const gen = loginWorkerSaga({ payload: { email, pass } });
expect(gen.next().value).toEqual(call(api.login, { email, pass }));
expect(gen.next(mockData).value).toEqual(put(loginSuccess(mockData)));
```

---

### Q103. What is `eventChannel` in Saga?
**Answer:** Connects Saga to event sources like WebSockets, push notifications, timers.

```js
function createWebSocketChannel(socket) {
  return eventChannel((emit) => {
    socket.on('message', emit);
    return () => socket.close();
  });
}
```

---

### Q104. How do you listen to a channel in Saga?

```js
function* watchSocket() {
  const channel = yield call(createWebSocketChannel, socket);
  while (true) {
    const message = yield take(channel);
    yield put(receiveMessage(message));
  }
}
```

---

### Q105. What is `actionChannel`?
**Answer:** Buffers incoming actions in a queue so they're processed one at a time.

```js
const requestChan = yield actionChannel('API_REQUEST');
while (true) {
  const action = yield take(requestChan);
  yield call(handleRequest, action);
}
```

---

### Q106. What is the difference between `call` (blocking) and `fork` (non-blocking)?

| | Blocks parent? | Use case |
|---|---|---|
| `call` | Yes | Ordered async (login flow) |
| `fork` | No | Parallel background tasks |

---

### Q107. How do you handle errors in sagas?

```js
function* fetchDataSaga() {
  try {
    const data = yield call(api.getData);
    yield put(fetchSuccess(data));
  } catch (error) {
    yield put(fetchFailure(error.message));
    yield put(showAlert('Something went wrong'));
  }
}
```

---

### Q108. How do you implement a retry mechanism in Saga?

```js
function* fetchWithRetry(fn, args, retries = 3) {
  for (let i = 0; i < retries; i++) {
    try {
      return yield call(fn, ...args);
    } catch (e) {
      if (i === retries - 1) throw e;
      yield delay(1000 * (i + 1));
    }
  }
}
```

---

### Q109. How do you implement a polling mechanism with Saga?

```js
function* pollDashboard() {
  while (true) {
    yield call(fetchDashboardSaga);
    yield delay(30000); // poll every 30s
  }
}

function* watchPolling() {
  while (true) {
    yield take('START_POLLING');
    const task = yield fork(pollDashboard);
    yield take('STOP_POLLING');
    yield cancel(task);
  }
}
```

---

### Q110. How do you implement debounce in Saga?

```js
function* handleSearch(action) {
  yield delay(400);
  yield call(searchSaga, action.payload);
}

function* watchSearch() {
  yield takeLatest('SEARCH_INPUT', handleSearch);
}
```

---

### Q111. How do you implement throttle in Saga?

```js
import { throttle } from 'redux-saga/effects';
yield throttle(1000, 'BUTTON_CLICK', handleClick);
```

---

### Q112. How did you use Saga in your ERP app for 200+ REST APIs?
**Answer (Your Answer):** Each module (HRMS, CRM, Payroll) had its own saga file. A root saga forked all of them. `takeLatest` was used for GET requests, `takeLeading` for form submissions to prevent double-posts. Error boundaries dispatched global alert actions. Background tasks like attendance sync used channels.

---

### Q113. How do you pass parameters from a component to a saga?
**Answer:** Via action payload.

```js
// Component
dispatch(fetchEmployee({ empId: '123' }));

// Saga
function* fetchEmployeeSaga(action) {
  const data = yield call(api.getEmployee, action.payload.empId);
}
```

---

### Q114. What is `buffers` in Saga channels?
**Answer:** Controls how many messages to buffer. `buffers.fixed(5)` keeps last 5; `buffers.dropping()` drops new ones when full; `buffers.sliding()` drops old ones.

---

### Q115. How do you handle JWT token refresh with Saga?

```js
function* refreshTokenSaga() {
  const refreshToken = yield select((s) => s.auth.refreshToken);
  const data = yield call(api.refreshToken, refreshToken);
  yield put(setToken(data.token));
}

function* apiRequestSaga(action) {
  try {
    const result = yield call(api.request, action.payload);
    yield put(apiSuccess(result));
  } catch (e) {
    if (e.status === 401) {
      yield call(refreshTokenSaga);
      // retry
      const result = yield call(api.request, action.payload);
      yield put(apiSuccess(result));
    }
  }
}
```

---

### Q116. What is `put.resolve` in Saga?
**Answer:** Like `put` but waits for the dispatched action's effects to complete before continuing — rarely needed.

---

### Q117. How do you share data between sagas?
**Answer:** Via Redux state (`select`), or by passing values as saga parameters, or via channels.

---

### Q118. What is the difference between Saga and Thunk?

| | Saga | Thunk |
|---|---|---|
| Complexity | High (Generators) | Low |
| Testability | Excellent | Moderate |
| Async patterns | Complex (race, cancel, poll) | Simple |
| Use case | Enterprise-scale async | Simple CRUD |

---

### Q119. How do you implement login + auto-logout (session timeout) with Saga?

```js
function* sessionManager() {
  while (true) {
    yield take('auth/loginSuccess');
    const { expired } = yield race({
      expired: delay(30 * 60 * 1000), // 30 min
      logout: take('auth/logout'),
    });
    if (expired) yield put(autoLogout());
  }
}
```

---

### Q120. How do you run initialization sagas on app launch?

```js
function* appInitSaga() {
  yield call(checkAuthSaga);
  yield call(loadSettingsSaga);
  yield call(syncOfflineDataSaga);
  yield put(appReady());
}
```

---

### Q121–180. Additional Redux-Saga Questions

**Q121.** What is `sagaMiddleware.run()`?  
**A:** Starts the root saga. Called once after store creation.

**Q122.** What happens if a saga throws an uncaught error?  
**A:** The saga terminates. Use global `onError` handler in `createSagaMiddleware({ onError })`.

**Q123.** What is the `monitor` option in `createSagaMiddleware`?  
**A:** Allows observing saga lifecycle events for debugging.

**Q124.** Can you `yield` a Promise in a saga?  
**A:** Yes. Saga automatically wraps Promises and waits for resolution.

**Q125.** What is `takeEvery` implemented with internally?  
**A:** `fork` + infinite `while(true)` loop + `take`.

**Q126.** What is `call` effect's advantage over calling async directly?  
**A:** Returns a plain object (`{type: 'CALL', fn, args}`), making it fully testable without actually calling the function.

**Q127.** How do you mock `call` in saga tests?  
**A:** `gen.next(mockReturnValue)` — inject mock data at each yield step.

**Q128.** What is a "saga channel"?  
**A:** A message queue that decouples event sources from saga logic.

**Q129.** How do you handle WebSocket reconnection in Saga?  
**A:** Use a while loop with try/catch and delay-based backoff.

**Q130.** What is `END` in Saga channels?  
**A:** Signals that no more messages will come — closes the channel.

**Q131.** How do you implement sequential API calls?  
**A:** Use `call` sequentially — each line waits for the previous.

**Q132.** How do you implement parallel API calls?  
**A:** Use `all([call(a), call(b)])`.

**Q133.** How do you abort an API call on navigation away?  
**A:** Fork the API call task, listen for navigation action, then `cancel` the task.

**Q134.** What is `detach` in Saga fork?  
**A:** `spawn` creates a detached fork — errors don't bubble to parent.

**Q135.** What is a "long-lived saga"?  
**A:** A saga that runs for the app's lifetime 
