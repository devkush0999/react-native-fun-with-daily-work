
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



# 01 — Redux Fundamentals
**Part 03 · State Management | React Native Interview Prep **
> Topics: Store · Actions · Reducers · Dispatch · combineReducers · Immutability · Pure Functions · State Shape Design
> Total: 60 Questions | Easy: 20 · Medium: 25 · Hard: 15

---

## 📑 Table of Contents

| Section | Topic | Questions |
|---------|-------|-----------|
| [A](#a-store--setup) | Store & Setup | Q1–Q10 |
| [B](#b-actions--action-creators) | Actions & Action Creators | Q11–Q20 |
| [C](#c-reducers--pure-functions) | Reducers & Pure Functions | Q21–Q32 |
| [D](#d-dispatch--data-flow) | Dispatch & Data Flow | Q33–Q40 |
| [E](#e-combinereducers) | combineReducers | Q41–Q48 |
| [F](#f-immutability) | Immutability | Q49–Q54 |
| [G](#g-state-shape-design) | State Shape Design | Q55–Q60 |

---

## A. Store & Setup

---

### Q1 🟢 Easy
**What is the Redux store and what are its three core responsibilities?**

**Answer:**
The Redux store is a single JavaScript object that holds the entire application state tree. Its three responsibilities are:
1. **Holds state** — `store.getState()` returns the current state
2. **Allows updates** — `store.dispatch(action)` is the only way to trigger state changes
3. **Notifies listeners** — `store.subscribe(listener)` registers callbacks that run after every dispatch

```typescript
import { createStore } from 'redux';
import rootReducer from './reducers';

const store = createStore(rootReducer);

// 1. Read state
console.log(store.getState());

// 2. Dispatch an action
store.dispatch({ type: 'INCREMENT' });

// 3. Subscribe to changes
const unsubscribe = store.subscribe(() => {
  console.log('State updated:', store.getState());
});

// Cleanup
unsubscribe();
```

---

### Q2 🟢 Easy
**What is the single source of truth principle in Redux?**

**Answer:**
The entire application state lives in one single store object. This means:
- There is **no local component state split across files** for shared data
- Debugging is easier — you can log/inspect the whole app state at any point
- Time-travel debugging (Redux DevTools) is possible because state is centralized

```typescript
// ✅ Entire app state in one place
const state = store.getState();
/*
{
  auth: { user: null, token: null },
  employees: { list: [], loading: false },
  attendance: { records: [], selectedDate: null },
  ui: { theme: 'light', sidebarOpen: false }
}
*/

// ❌ Anti-pattern — don't split shared state across components
// Component A: useState([]) for employee list
// Component B: useState([]) for the same employee list
```

---

### Q3 🟢 Easy
**How do you create a Redux store? What is the signature of `createStore`?**

**Answer:**
`createStore(reducer, [preloadedState], [enhancer])` takes:
- `reducer` — the root reducer function (required)
- `preloadedState` — initial state to hydrate from (e.g. AsyncStorage)
- `enhancer` — middleware/devtools enhancer (optional)

```typescript
import { createStore, applyMiddleware, compose } from 'redux';
import createSagaMiddleware from 'redux-saga';
import rootReducer from './rootReducer';
import rootSaga from './rootSaga';

const sagaMiddleware = createSagaMiddleware();

// With middleware + DevTools
const composeEnhancers =
  (window as any).__REDUX_DEVTOOLS_EXTENSION_COMPOSE__ || compose;

const store = createStore(
  rootReducer,
  composeEnhancers(applyMiddleware(sagaMiddleware))
);

sagaMiddleware.run(rootSaga);

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
export default store;
```

---

### Q4 🟢 Easy
**How do you connect a Redux store to a React Native app?**

**Answer:**
Wrap the root component with `<Provider store={store}>`. The Provider uses React Context internally to make the store available to all nested components via `useSelector` and `useDispatch`.

```tsx
// App.tsx
import React from 'react';
import { Provider } from 'react-redux';
import store from './store';
import AppNavigator from './navigation/AppNavigator';

const App = () => {
  return (
    <Provider store={store}>
      <AppNavigator />
    </Provider>
  );
};

export default App;
```

```tsx
// Any child component
import { useSelector, useDispatch } from 'react-redux';
import { RootState } from '../store';

const Dashboard = () => {
  const user = useSelector((state: RootState) => state.auth.user);
  const dispatch = useDispatch();

  return <Text>Welcome {user?.name}</Text>;
};
```

---

### Q5 🟢 Easy
**What does `store.getState()` return and when should you avoid calling it directly in components?**

**Answer:**
`store.getState()` returns the current state snapshot. You should avoid calling it directly in components because:
- It **does not trigger re-renders** when state changes
- It bypasses React's rendering cycle
- Use `useSelector` instead — it subscribes to the store and re-renders on relevant state changes

```typescript
// ❌ Wrong — component won't re-render on state change
const MyComponent = () => {
  const user = store.getState().auth.user; // stale after first render
  return <Text>{user?.name}</Text>;
};

// ✅ Correct — re-renders when auth.user changes
const MyComponent = () => {
  const user = useSelector((state: RootState) => state.auth.user);
  return <Text>{user?.name}</Text>;
};

// ✅ store.getState() is fine OUTSIDE components
// e.g., in sagas, utility functions, API interceptors
const token = store.getState().auth.token;
```

---

### Q6 🟡 Medium
**What is `applyMiddleware` and how does it enhance the store?**

**Answer:**
`applyMiddleware` is a store enhancer that wraps `store.dispatch` to add custom behaviour — like logging, async operations, crash reporting, or routing. Each middleware has access to `getState` and `dispatch` and calls `next(action)` to pass the action down the chain.

```typescript
// Custom logger middleware
const loggerMiddleware = (store: any) => (next: any) => (action: any) => {
  console.group(action.type);
  console.log('Prev State:', store.getState());
  console.log('Action:', action);
  const result = next(action); // Pass to next middleware / reducer
  console.log('Next State:', store.getState());
  console.groupEnd();
  return result;
};

// Middleware chain: dispatch → logger → sagaMiddleware → reducer
const store = createStore(
  rootReducer,
  applyMiddleware(loggerMiddleware, sagaMiddleware)
);
```

---

### Q7 🟡 Medium
**What is the difference between `createStore` (Redux core) and `configureStore` (Redux Toolkit)?**

**Answer:**
| Feature | `createStore` | `configureStore` (RTK) |
|---------|--------------|----------------------|
| Setup | Manual enhancer composition | Auto-configured |
| DevTools | Manual setup | Enabled by default |
| Immer | Not included | Built-in (immutable updates) |
| Thunk | Must add manually | Pre-installed |
| Boilerplate | High | Minimal |

```typescript
// Old way — createStore
const store = createStore(
  rootReducer,
  composeWithDevTools(applyMiddleware(thunk, sagaMiddleware))
);

// Modern way — configureStore (RTK)
import { configureStore } from '@reduxjs/toolkit';

const store = configureStore({
  reducer: rootReducer,
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware().concat(sagaMiddleware),
  devTools: process.env.NODE_ENV !== 'production',
});
```

---

### Q8 🟡 Medium
**How do you preload/hydrate Redux store from AsyncStorage in React Native?**

**Answer:**
Pass `preloadedState` as the second argument to `createStore`. Fetch data from AsyncStorage first, then initialize the store with it.

```typescript
import AsyncStorage from '@react-native-async-storage/async-storage';
import { createStore, applyMiddleware } from 'redux';
import rootReducer from './rootReducer';

const loadState = async () => {
  try {
    const serialized = await AsyncStorage.getItem('reduxState');
    return serialized ? JSON.parse(serialized) : undefined;
  } catch {
    return undefined;
  }
};

const saveState = (state: any) => {
  AsyncStorage.setItem('reduxState', JSON.stringify(state));
};

// In your app entry point:
const initStore = async () => {
  const preloadedState = await loadState();
  const store = createStore(rootReducer, preloadedState, applyMiddleware(sagaMiddleware));

  // Persist on every change (debounce in production)
  store.subscribe(() => saveState(store.getState()));

  return store;
};
```

---

### Q9 🟡 Medium
**What is Redux DevTools and how does it help during development?**

**Answer:**
Redux DevTools is a browser/RN debugger extension that lets you:
- **Inspect** every action dispatched and the resulting state diff
- **Time-travel** — replay or rewind state changes
- **Import/export** state for bug reproduction
- **Skip actions** to isolate bugs

```typescript
import { createStore, applyMiddleware, compose } from 'redux';

// Enable DevTools only in development
const composeEnhancers =
  process.env.NODE_ENV === 'development' &&
  (global as any).__REDUX_DEVTOOLS_EXTENSION_COMPOSE__
    ? (global as any).__REDUX_DEVTOOLS_EXTENSION_COMPOSE__
    : compose;

const store = createStore(
  rootReducer,
  composeEnhancers(applyMiddleware(sagaMiddleware))
);

// For React Native — use Flipper + redux-flipper plugin
// npm install redux-flipper
import { createFlipperMiddleware } from 'redux-flipper';
const flipperMiddleware = createFlipperMiddleware();

const store = createStore(
  rootReducer,
  applyMiddleware(sagaMiddleware, flipperMiddleware)
);
```

---

### Q10 🔴 Hard
**Explain the full Redux data flow with a real ERP example. What happens from button press to UI update?**

**Answer:**
Redux follows a strict **unidirectional data flow**: Action → Reducer → Store → View

Step-by-step for "Load Employee List" in an ERP app:
1. User navigates to Employees screen
2. Component dispatches `FETCH_EMPLOYEES_REQUEST`
3. Saga intercepts, calls API
4. On success, dispatches `FETCH_EMPLOYEES_SUCCESS` with payload
5. Reducer updates `employees.list` in state
6. `useSelector` in component detects change, triggers re-render
7. FlatList displays updated employee list

```typescript
// 1. Action Types
const FETCH_EMPLOYEES_REQUEST = 'FETCH_EMPLOYEES_REQUEST';
const FETCH_EMPLOYEES_SUCCESS = 'FETCH_EMPLOYEES_SUCCESS';
const FETCH_EMPLOYEES_FAILURE = 'FETCH_EMPLOYEES_FAILURE';

// 2. Component dispatches action
const EmployeeScreen = () => {
  const dispatch = useDispatch();
  const { list, loading, error } = useSelector((s: RootState) => s.employees);

  useEffect(() => {
    dispatch({ type: FETCH_EMPLOYEES_REQUEST });
  }, []);

  if (loading) return <ActivityIndicator />;
  return <FlatList data={list} renderItem={({ item }) => <Text>{item.name}</Text>} />;
};

// 3. Saga intercepts (Part 02 topic, shown for flow context)
function* fetchEmployeesSaga() {
  try {
    const data: Employee[] = yield call(api.getEmployees);
    yield put({ type: FETCH_EMPLOYEES_SUCCESS, payload: data });
  } catch (e: any) {
    yield put({ type: FETCH_EMPLOYEES_FAILURE, payload: e.message });
  }
}

// 4. Reducer updates state
const employeesReducer = (state = initialState, action: AnyAction) => {
  switch (action.type) {
    case FETCH_EMPLOYEES_REQUEST:
      return { ...state, loading: true, error: null };
    case FETCH_EMPLOYEES_SUCCESS:
      return { ...state, loading: false, list: action.payload };
    case FETCH_EMPLOYEES_FAILURE:
      return { ...state, loading: false, error: action.payload };
    default:
      return state;
  }
};
```

---

## B. Actions & Action Creators

---

### Q11 🟢 Easy
**What is a Redux action? What are the required and optional fields?**

**Answer:**
An action is a plain JavaScript object that describes **what happened**. The only required field is `type` (a string constant). An optional `payload` carries data; `error: true` flags error actions; `meta` carries extra info.

```typescript
// Minimal action
{ type: 'INCREMENT' }

// With payload (FSA — Flux Standard Action convention)
{ type: 'ADD_EMPLOYEE', payload: { id: '1', name: 'Devesh', dept: 'Engineering' } }

// Error action
{ type: 'FETCH_FAILED', payload: new Error('Network Error'), error: true }

// With meta
{ type: 'LOG_EVENT', payload: { screen: 'Dashboard' }, meta: { timestamp: Date.now() } }

// TypeScript typed action
interface AddEmployeeAction {
  type: 'ADD_EMPLOYEE';
  payload: {
    id: string;
    name: string;
    dept: string;
  };
}
```

---

### Q12 🟢 Easy
**What is an action creator and why use one instead of dispatching plain objects?**

**Answer:**
An action creator is a function that returns an action object. Benefits:
- **Reusable** — define once, call everywhere
- **Type-safe** — TypeScript can infer payload shape
- **Testable** — easy to unit test in isolation
- **Less typo-prone** — action type string is defined once

```typescript
// ❌ Inline — type string can be mistyped anywhere
dispatch({ type: 'FETCH_EMPLOYYEES_REQUEST' }); // bug: typo

// ✅ Action creator + constant
const FETCH_EMPLOYEES_REQUEST = 'FETCH_EMPLOYEES_REQUEST' as const;

const fetchEmployeesRequest = () => ({
  type: FETCH_EMPLOYEES_REQUEST,
});

const addEmployee = (employee: Employee) => ({
  type: 'ADD_EMPLOYEE' as const,
  payload: employee,
});

// Dispatch
dispatch(fetchEmployeesRequest());
dispatch(addEmployee({ id: '2', name: 'Rahul', dept: 'HR' }));
```

---

### Q13 🟢 Easy
**What is the Flux Standard Action (FSA) convention?**

**Answer:**
FSA is a community convention for action structure:
- `type` — string, required
- `payload` — any value, optional. If `error` is true, payload should be an Error object
- `error` — boolean, optional. `true` signals a failure action
- `meta` — any extra info not part of payload (timestamps, request IDs)
- **No other top-level keys allowed**

```typescript
// ✅ FSA-compliant actions
const successAction = {
  type: 'FETCH_PAYROLL_SUCCESS',
  payload: { employees: [], totalAmount: 50000 },
};

const errorAction = {
  type: 'FETCH_PAYROLL_FAILURE',
  payload: new Error('Server Error'),
  error: true,
};

const metaAction = {
  type: 'TRACK_SCREEN_VIEW',
  payload: { screen: 'Attendance' },
  meta: { timestamp: Date.now(), userId: 'u123' },
};

// ❌ Non-FSA — avoid custom top-level keys
const badAction = {
  type: 'FETCH_DONE',
  data: [],        // ❌ should be payload
  isError: false,  // ❌ should be error
};
```

---

### Q14 🟢 Easy
**How do you define action types in TypeScript to get full type safety?**

**Answer:**
Use `as const` assertions or string literal union types to make action types narrowly typed. This enables TypeScript to discriminate between action types in switch statements.

```typescript
// Method 1: as const on string
const INCREMENT = 'INCREMENT' as const;
const DECREMENT = 'DECREMENT' as const;

// Method 2: enum (avoid — poor tree shaking)
enum ActionTypes { INCREMENT = 'INCREMENT' }

// Method 3: Union type with interfaces (recommended)
interface IncrementAction { type: 'INCREMENT' }
interface DecrementAction { type: 'DECREMENT'; payload: number }
interface ResetAction    { type: 'RESET' }

type CounterAction = IncrementAction | DecrementAction | ResetAction;

// In reducer — TypeScript knows payload exists only for DECREMENT
const reducer = (state = 0, action: CounterAction): number => {
  switch (action.type) {
    case 'INCREMENT': return state + 1;
    case 'DECREMENT': return state - action.payload; // ✅ TS knows payload: number
    case 'RESET':     return 0;
    default:          return state;
  }
};
```

---

### Q15 🟢 Easy
**What is the difference between synchronous and asynchronous actions in Redux?**

**Answer:**
- **Synchronous actions** — plain objects, handled directly by reducers
- **Asynchronous actions** — functions (thunks) or effects (sagas) that perform async work then dispatch sync actions

```typescript
// Synchronous action — reducer handles immediately
const setUser = (user: User) => ({ type: 'SET_USER' as const, payload: user });

// Asynchronous — Thunk (returns a function)
const loginThunk = (credentials: Credentials) =>
  async (dispatch: AppDispatch) => {
    dispatch({ type: 'LOGIN_REQUEST' });
    try {
      const user = await authApi.login(credentials);
      dispatch({ type: 'LOGIN_SUCCESS', payload: user });
    } catch (e: any) {
      dispatch({ type: 'LOGIN_FAILURE', payload: e.message });
    }
  };

// Asynchronous — Saga (watcher/worker pattern, Part 02)
// dispatch({ type: 'LOGIN_REQUEST', payload: credentials })
// → saga intercepts → calls API → dispatches LOGIN_SUCCESS/FAILURE
```

---

### Q16 🟡 Medium
**Why should action type strings be unique across the entire app? How do you enforce this in large apps?**

**Answer:**
If two reducers have the same action type string, both will respond to the same dispatch — often causing unexpected state changes. In large apps enforce uniqueness with:
1. **Namespace prefixes** — `auth/LOGIN_REQUEST`, `employee/FETCH_REQUEST`
2. **Feature-based modules** — each module owns its type strings
3. **Redux Toolkit** — auto-namespaces via `createSlice`

```typescript
// ❌ Naming collision
// auth/reducer.ts
const FETCH_REQUEST = 'FETCH_REQUEST'; // same string!
// employee/reducer.ts
const FETCH_REQUEST = 'FETCH_REQUEST'; // both respond to same dispatch!

// ✅ Namespaced constants
// auth/actionTypes.ts
export const AUTH_FETCH_REQUEST   = 'auth/FETCH_REQUEST';
export const AUTH_FETCH_SUCCESS   = 'auth/FETCH_SUCCESS';
export const AUTH_FETCH_FAILURE   = 'auth/FETCH_FAILURE';

// employee/actionTypes.ts
export const EMPLOYEE_FETCH_REQUEST = 'employee/FETCH_REQUEST';
export const EMPLOYEE_FETCH_SUCCESS = 'employee/FETCH_SUCCESS';

// RTK auto-namespaces (createSlice)
const authSlice = createSlice({
  name: 'auth',
  // generates 'auth/loginRequest', 'auth/loginSuccess' automatically
});
```

---

### Q17 🟡 Medium
**What is a "bound action creator"? How do you use `bindActionCreators`?**

**Answer:**
A bound action creator automatically calls `dispatch` — so you call `add()` instead of `dispatch(add())`. `bindActionCreators` wraps each action creator with dispatch.

```typescript
import { bindActionCreators } from 'redux';
import { useDispatch } from 'react-redux';
import * as employeeActions from '../store/employee/actions';

// Manual binding
const BoundExample = () => {
  const dispatch = useDispatch();

  // Without binding
  const handleAdd = (emp: Employee) => dispatch(employeeActions.addEmployee(emp));

  // With bindActionCreators
  const actions = bindActionCreators(employeeActions, dispatch);

  // Now call directly
  const handleFetch = () => actions.fetchEmployees();
  const handleDelete = (id: string) => actions.deleteEmployee(id);

  return <Button title="Fetch" onPress={handleFetch} />;
};

// Note: In modern React-Redux with hooks, bindActionCreators
// is rarely needed — useDispatch + direct calls is cleaner
```

---

### Q18 🟡 Medium
**How would you handle multiple async states (idle, loading, success, error) for an action?**

**Answer:**
Use a status enum in state to represent all four async states. This is cleaner than multiple booleans (`isLoading`, `isError`, `isSuccess`).

```typescript
// Status type
type AsyncStatus = 'idle' | 'loading' | 'succeeded' | 'failed';

interface EmployeeState {
  list: Employee[];
  status: AsyncStatus;
  error: string | null;
}

const initialState: EmployeeState = {
  list: [],
  status: 'idle',
  error: null,
};

const employeeReducer = (state = initialState, action: AnyAction): EmployeeState => {
  switch (action.type) {
    case 'FETCH_EMPLOYEES_REQUEST':
      return { ...state, status: 'loading', error: null };
    case 'FETCH_EMPLOYEES_SUCCESS':
      return { ...state, status: 'succeeded', list: action.payload };
    case 'FETCH_EMPLOYEES_FAILURE':
      return { ...state, status: 'failed', error: action.payload };
    default:
      return state;
  }
};

// Component usage
const EmployeeScreen = () => {
  const { list, status, error } = useSelector((s: RootState) => s.employees);

  if (status === 'idle')      return <Button title="Load" onPress={() => dispatch(fetchEmployees())} />;
  if (status === 'loading')   return <ActivityIndicator />;
  if (status === 'failed')    return <Text>Error: {error}</Text>;
  return <FlatList data={list} />;
};
```

---

### Q19 🟡 Medium
**What is action batching and why does it matter for React Native performance?**

**Answer:**
Action batching means grouping multiple dispatches so React only re-renders once. By default in React 18+, updates inside event handlers are batched. For async operations (sagas, timeouts), manual batching may be needed.

```typescript
import { batch } from 'react-redux';

// Without batching — 3 re-renders
dispatch({ type: 'SET_USER', payload: user });
dispatch({ type: 'SET_TOKEN', payload: token });
dispatch({ type: 'SET_LOADING', payload: false });

// With batch — 1 re-render
batch(() => {
  dispatch({ type: 'SET_USER', payload: user });
  dispatch({ type: 'SET_TOKEN', payload: token });
  dispatch({ type: 'SET_LOADING', payload: false });
});

// In Redux-Saga — use channels or dispatch a single combined action
// instead of multiple puts when possible
function* loginSaga(action: LoginAction) {
  const { user, token } = yield call(api.login, action.payload);

  // One dispatch = one re-render
  yield put({
    type: 'LOGIN_COMPLETE',
    payload: { user, token, loading: false }
  });
}
```

---

### Q20 🔴 Hard
**Design a complete action architecture for a Fintech app (like Debt Relief India). How do you handle KYC, payment, and loan state actions without conflicts?**

**Answer:**
Use feature-based namespacing with a centralized action types registry. Group related actions into domain modules.

```typescript
// store/actionTypes.ts — central registry
export const ActionTypes = {
  // KYC Module
  KYC: {
    INIT:           'kyc/INIT',
    UPLOAD_DOC:     'kyc/UPLOAD_DOC',
    UPLOAD_SUCCESS: 'kyc/UPLOAD_SUCCESS',
    UPLOAD_FAILURE: 'kyc/UPLOAD_FAILURE',
    VERIFY:         'kyc/VERIFY',
    VERIFIED:       'kyc/VERIFIED',
  },
  // Payment Module
  PAYMENT: {
    INITIATE:       'payment/INITIATE',
    RAZORPAY_OPEN:  'payment/RAZORPAY_OPEN',
    SUCCESS:        'payment/SUCCESS',
    FAILURE:        'payment/FAILURE',
    REFUND_REQUEST: 'payment/REFUND_REQUEST',
  },
  // Loan Module
  LOAN: {
    FETCH:          'loan/FETCH',
    FETCH_SUCCESS:  'loan/FETCH_SUCCESS',
    EMI_CALCULATE:  'loan/EMI_CALCULATE',
    REPAYMENT_DUE:  'loan/REPAYMENT_DUE',
    CLOSE:          'loan/CLOSE',
  },
} as const;

// Action creators — strongly typed
interface PaymentInitiateAction {
  type: typeof ActionTypes.PAYMENT.INITIATE;
  payload: { amount: number; loanId: string; method: 'upi' | 'card' };
}

const initiatePayment = (
  amount: number, loanId: string, method: 'upi' | 'card'
): PaymentInitiateAction => ({
  type: ActionTypes.PAYMENT.INITIATE,
  payload: { amount, loanId, method },
});

// Safe dispatch — no type string conflicts across modules
dispatch(initiatePayment(5000, 'loan_001', 'upi'));
```

---

## C. Reducers & Pure Functions

---

### Q21 🟢 Easy
**What is a reducer? What are the rules it must follow?**

**Answer:**
A reducer is a pure function `(state, action) => newState` that describes how state changes in response to an action. Rules:
1. **Pure** — no side effects, no API calls, no random values
2. **Immutable** — never mutate state directly, return new objects
3. **Deterministic** — same inputs always produce same output
4. **Handle default** — always return current state for unknown actions
5. **Initialize state** — provide default value for state parameter

```typescript
interface CounterState { count: number }

const initialState: CounterState = { count: 0 };

const counterReducer = (
  state: CounterState = initialState,
  action: AnyAction
): CounterState => {
  switch (action.type) {
    case 'INCREMENT':
      return { ...state, count: state.count + 1 }; // ✅ new object

    case 'DECREMENT':
      return { ...state, count: state.count - 1 };

    case 'RESET':
      return initialState;

    default:
      return state; // ✅ always return state for unknown actions
  }
};

// ❌ Violations
const badReducer = (state = initialState, action: AnyAction) => {
  state.count++;             // ❌ mutating state
  fetch('/api/log');         // ❌ side effect
  return Math.random() > 0.5 // ❌ non-deterministic
    ? state
    : initialState;
};
```

---

### Q22 🟢 Easy
**What is a pure function? Give an example of a pure vs impure reducer.**

**Answer:**
A pure function: given the same inputs, always returns the same output, and has no side effects (no mutations, no I/O, no randomness).

```typescript
// ✅ Pure reducer
const pureReducer = (state = { count: 0 }, action: AnyAction) => {
  switch (action.type) {
    case 'ADD':
      return { count: state.count + action.payload }; // same inputs → same output
    default:
      return state;
  }
};

// ❌ Impure — mutation
const impure1 = (state = { items: [] }, action: AnyAction) => {
  if (action.type === 'ADD_ITEM') {
    state.items.push(action.payload); // ❌ mutates original array
    return state;                     // ❌ returns same reference
  }
  return state;
};

// ❌ Impure — side effect
const impure2 = (state = {}, action: AnyAction) => {
  if (action.type === 'SAVE') {
    localStorage.setItem('key', JSON.stringify(state)); // ❌ side effect
  }
  return state;
};

// ❌ Impure — non-deterministic
const impure3 = (state = { id: '' }, action: AnyAction) => {
  if (action.type === 'CREATE') {
    return { id: Math.random().toString() }; // ❌ different result each time
  }
  return state;
};
```

---

### Q23 🟢 Easy
**Why must reducers return the existing state for unrecognized actions?**

**Answer:**
When an app initializes, Redux dispatches an internal `@@INIT` action to populate the store. If a reducer doesn't return `state` for unknown actions, it returns `undefined`, breaking the entire state tree. Also, third-party libraries may dispatch their own actions — reducers must silently ignore them.

```typescript
// ❌ Bug — returns undefined for @@INIT and unknown actions
const brokenReducer = (state = { count: 0 }, action: AnyAction) => {
  if (action.type === 'INCREMENT') {
    return { count: state.count + 1 };
  }
  // Returns undefined for all other actions including @@redux/INIT
};

// ✅ Correct — always return state
const correctReducer = (state = { count: 0 }, action: AnyAction) => {
  switch (action.type) {
    case 'INCREMENT': return { count: state.count + 1 };
    default:          return state; // ✅ handles @@INIT, unknown actions
  }
};
```

---

### Q24 🟢 Easy
**How do you handle arrays in a reducer without mutation?**

**Answer:**
Use spread operator, `map`, `filter`, and `concat` — these return new arrays. Never use `push`, `pop`, `splice`, or direct index assignment.

```typescript
interface Item { id: string; name: string }
interface ListState { items: Item[] }

const listReducer = (state: ListState = { items: [] }, action: AnyAction): ListState => {
  switch (action.type) {

    case 'ADD_ITEM':
      return { ...state, items: [...state.items, action.payload] }; // ✅

    case 'REMOVE_ITEM':
      return {
        ...state,
        items: state.items.filter(item => item.id !== action.payload),
      };

    case 'UPDATE_ITEM':
      return {
        ...state,
        items: state.items.map(item =>
          item.id === action.payload.id ? { ...item, ...action.payload } : item
        ),
      };

    default:
      return state;
  }
};

// ❌ Never do this
case 'ADD_ITEM':
  state.items.push(action.payload); // ❌ mutation!
  return state;
```

---

### Q25 🟢 Easy
**How do you handle nested objects in a reducer without mutation?**

**Answer:**
Spread each level of nesting that changes. For deeply nested state, consider `immer` (used internally by RTK).

```typescript
interface UserProfile {
  id: string;
  personal: { name: string; email: string };
  address: { city: string; pincode: string };
}

const profileReducer = (
  state: UserProfile = initialProfile,
  action: AnyAction
): UserProfile => {
  switch (action.type) {

    case 'UPDATE_EMAIL':
      return {
        ...state,
        personal: {
          ...state.personal,          // ✅ spread nested level
          email: action.payload,
        },
      };

    case 'UPDATE_CITY':
      return {
        ...state,
        address: {
          ...state.address,
          city: action.payload,
        },
      };

    default:
      return state;
  }
};

// ✅ With immer (cleaner for deep nesting)
import produce from 'immer';

const profileReducerImmer = produce((draft, action) => {
  if (action.type === 'UPDATE_EMAIL') {
    draft.personal.email = action.payload; // looks like mutation, but isn't
  }
}, initialProfile);
```

---

### Q26 🟡 Medium
**What is reducer composition and why is it important for scalable apps?**

**Answer:**
Reducer composition means splitting the root reducer into smaller, focused reducers — each managing its own slice of state. This keeps reducers small, testable, and independently maintainable.

```typescript
// auth reducer — only manages auth slice
const authReducer = (state = authInitial, action: AnyAction) => {
  switch (action.type) {
    case 'LOGIN_SUCCESS': return { ...state, user: action.payload };
    case 'LOGOUT':        return authInitial;
    default:              return state;
  }
};

// employee reducer — only manages employee slice
const employeeReducer = (state = empInitial, action: AnyAction) => {
  switch (action.type) {
    case 'FETCH_EMPLOYEES_SUCCESS': return { ...state, list: action.payload };
    default:                        return state;
  }
};

// Root reducer — composed from slices
const rootReducer = combineReducers({
  auth:      authReducer,
  employees: employeeReducer,
  // Each key maps to state.auth, state.employees, etc.
});

// state shape output:
// { auth: { user, token }, employees: { list, loading } }
```

---

### Q27 🟡 Medium
**How do you reset the entire Redux state on logout?**

**Answer:**
Wrap the root reducer and return `undefined` state when a LOGOUT action is dispatched. Each sub-reducer will then re-initialize to its `initialState`.

```typescript
import { combineReducers, AnyAction } from 'redux';

const appReducer = combineReducers({
  auth:      authReducer,
  employees: employeeReducer,
  payroll:   payrollReducer,
  ui:        uiReducer,
});

// Root reducer wrapper
const rootReducer = (state: ReturnType<typeof appReducer> | undefined, action: AnyAction) => {
  if (action.type === 'LOGOUT') {
    // Pass undefined → each slice returns its initialState
    state = undefined;
  }
  return appReducer(state, action);
};

// Optionally preserve some state after logout (e.g. ui preferences)
const rootReducerWithPersist = (
  state: ReturnType<typeof appReducer> | undefined,
  action: AnyAction
) => {
  if (action.type === 'LOGOUT') {
    const { ui } = state!; // preserve ui settings
    state = { ui } as any;
  }
  return appReducer(state, action);
};
```

---

### Q28 🟡 Medium
**How do you handle multiple items in state using a map/dictionary pattern instead of an array?**

**Answer:**
For large lists (employees, products), use a normalized map `{ byId: {}, allIds: [] }` instead of a plain array. O(1) lookups vs O(n) array searches.

```typescript
interface Employee { id: string; name: string; dept: string }

interface NormalizedState {
  byId: Record<string, Employee>;
  allIds: string[];
  loading: boolean;
}

const initial: NormalizedState = { byId: {}, allIds: [], loading: false };

const employeeReducer = (state = initial, action: AnyAction): NormalizedState => {
  switch (action.type) {

    case 'FETCH_EMPLOYEES_SUCCESS': {
      const byId: Record<string, Employee> = {};
      const allIds: string[] = [];
      (action.payload as Employee[]).forEach(emp => {
        byId[emp.id] = emp;
        allIds.push(emp.id);
      });
      return { ...state, byId, allIds, loading: false };
    }

    case 'UPDATE_EMPLOYEE':
      return {
        ...state,
        byId: {
          ...state.byId,
          [action.payload.id]: { ...state.byId[action.payload.id], ...action.payload },
        },
      };

    case 'DELETE_EMPLOYEE': {
      const { [action.payload]: removed, ...rest } = state.byId;
      return {
        ...state,
        byId: rest,
        allIds: state.allIds.filter(id => id !== action.payload),
      };
    }

    default:
      return state;
  }
};

// O(1) lookup by id — no .find() needed
const emp = state.employees.byId['emp_001'];
```

---

### Q29 🟡 Medium
**How do you write unit tests for a reducer?**

**Answer:**
Reducers are pure functions — easy to test. Pass a state + action, assert the output. No mocking needed.

```typescript
import { employeeReducer } from '../employee/reducer';
import { initialState } from '../employee/initialState';

describe('employeeReducer', () => {

  it('returns initialState for unknown actions', () => {
    expect(employeeReducer(undefined, { type: '@@INIT' })).toEqual(initialState);
  });

  it('sets loading true on FETCH_REQUEST', () => {
    const state = employeeReducer(initialState, { type: 'FETCH_EMPLOYEES_REQUEST' });
    expect(state.loading).toBe(true);
    expect(state.error).toBeNull();
  });

  it('populates list on FETCH_SUCCESS', () => {
    const employees = [{ id: '1', name: 'Devesh', dept: 'Eng' }];
    const state = employeeReducer(initialState, {
      type: 'FETCH_EMPLOYEES_SUCCESS',
      payload: employees,
    });
    expect(state.list).toEqual(employees);
    expect(state.loading).toBe(false);
  });

  it('does not mutate original state', () => {
    const frozen = Object.freeze({ ...initialState });
    expect(() => employeeReducer(frozen, { type: 'FETCH_EMPLOYEES_REQUEST' })).not.toThrow();
  });
});
```

---

### Q30 🟡 Medium
**What happens if a reducer throws an error? How do you handle it?**

**Answer:**
If a reducer throws, Redux propagates the error and the dispatch call throws — potentially crashing the app. Wrap reducers in try-catch for resilience, or use error boundaries + logging.

```typescript
// Safe reducer wrapper
const safeReducer = <S>(
  reducer: (state: S | undefined, action: AnyAction) => S,
  fallback: S
) => (state: S | undefined, action: AnyAction): S => {
  try {
    return reducer(state, action);
  } catch (e) {
    console.error('Reducer error:', e, '\nAction:', action);
    // Report to Crashlytics / Sentry
    crashlytics().recordError(e as Error);
    return state ?? fallback; // ✅ return last known good state
  }
};

const safeEmployeeReducer = safeReducer(employeeReducer, initialState);

const rootReducer = combineReducers({
  employees: safeEmployeeReducer,
  // ...
});
```

---

### Q31 🔴 Hard
**How does Redux know which reducer to call when an action is dispatched?**

**Answer:**
When you dispatch an action, Redux calls the **root reducer** with the current state and action. `combineReducers` then calls **every** sub-reducer, passing each its own state slice. Each reducer checks `action.type` and either handles it (returns new state) or returns the current state unchanged.

```typescript
// This is what combineReducers does internally (simplified):
function myCombineReducers(reducers: Record<string, Reducer>) {
  return function rootReducer(state: any = {}, action: AnyAction) {
    let hasChanged = false;
    const nextState: any = {};

    Object.keys(reducers).forEach(key => {
      const reducer = reducers[key];
      const previousStateForKey = state[key];
      const nextStateForKey = reducer(previousStateForKey, action);

      nextState[key] = nextStateForKey;
      hasChanged = hasChanged || nextStateForKey !== previousStateForKey;
    });

    // If nothing changed, return same reference (no re-render)
    return hasChanged ? nextState : state;
  };
}

// All reducers are called for every action dispatch!
// This is why action types must be unique — multiple reducers
// could accidentally handle the same action type
```

---

### Q32 🔴 Hard
**How do you handle cross-slice state updates — when one action needs to update multiple reducer slices?**

**Answer:**
**Option 1:** Multiple reducers listen to the same action type (recommended for simple cases).
**Option 2:** Use a saga that dispatches multiple targeted actions.
**Option 3:** Use a root-level reducer that handles cross-slice coordination.

```typescript
// Option 1 — Multiple slices respond to same action
// auth reducer
const authReducer = (state = authInit, action: AnyAction) => {
  switch (action.type) {
    case 'LOGOUT':
      return authInit; // clears auth
    default: return state;
  }
};

// cart reducer — also responds to LOGOUT
const cartReducer = (state = cartInit, action: AnyAction) => {
  switch (action.type) {
    case 'LOGOUT':
      return cartInit; // clears cart on logout
    default: return state;
  }
};

// Option 2 — Saga dispatches multiple actions
function* logoutSaga() {
  yield put({ type: 'auth/CLEAR' });
  yield put({ type: 'cart/CLEAR' });
  yield put({ type: 'notifications/CLEAR' });
  yield call(AsyncStorage.clear);
}

// Option 3 — Root-level reducer for complex cross-slice logic
const rootReducer = (state: RootState | undefined, action: AnyAction) => {
  if (action.type === 'USER_DEACTIVATED') {
    // Complex cross-slice transformation
    return {
      ...appReducer(state, action),
      auth:     authInit,
      employee: empInit,
      audit: { ...state?.audit, deactivatedAt: Date.now() },
    };
  }
  return appReducer(state, action);
};
```

---

## D. Dispatch & Data Flow

---

### Q33 🟢 Easy
**What is `dispatch` and what happens when you call it?**

**Answer:**
`dispatch(action)` is the only way to trigger a state change in Redux. When called:
1. The action is sent to the root reducer
2. The reducer computes new state
3. Store saves the new state
4. All subscribers (including `useSelector` hooks) are notified
5. React re-renders components with changed state

```typescript
import { useDispatch } from 'react-redux';

const AttendanceButton = () => {
  const dispatch = useDispatch();

  const handleCheckIn = () => {
    // Step 1: dispatch sends action to store
    dispatch({
      type: 'ATTENDANCE_CHECK_IN',
      payload: { employeeId: 'emp_001', timestamp: Date.now() },
    });
    // Steps 2–5 happen synchronously inside Redux
  };

  return <Button title="Check In" onPress={handleCheckIn} />;
};
```

---

### Q34 🟢 Easy
**What is the difference between `useDispatch` and accessing `store.dispatch` directly?**

**Answer:**
| | `useDispatch` | `store.dispatch` |
|--|--------------|-----------------|
| Location | Inside React components | Anywhere (sagas, utils, interceptors) |
| Stable ref | Yes, memoized by React-Redux | Direct reference |
| Typing | Returns `AppDispatch` when typed correctly | Same |
| Best for | Component event handlers | Outside React tree |

```typescript
// ✅ In components — use useDispatch
const MyComponent = () => {
  const dispatch = useDispatch<AppDispatch>();
  return <Button onPress={() => dispatch(fetchEmployees())} title="Load" />;
};

// ✅ Outside React — use store.dispatch directly
// In axios interceptor (for token refresh):
axiosInstance.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      store.dispatch({ type: 'AUTH_TOKEN_EXPIRED' });
    }
    return Promise.reject(error);
  }
);
```

---

### Q35 🟢 Easy
**Can you dispatch multiple actions in sequence? What is the execution order?**

**Answer:**
Yes. Each `dispatch` call is processed synchronously and sequentially. Reducers run, state updates, and subscribers are notified after each dispatch before the next one begins.

```typescript
const handleLogin = async () => {
  // Each dispatch is processed fully before the next line runs
  dispatch({ type: 'AUTH_SET_LOADING', payload: true });  // State: loading=true
  dispatch({ type: 'AUTH_SET_USER', payload: user });      // State: user=...
  dispatch({ type: 'AUTH_SET_TOKEN', payload: token });    // State: token=...
  dispatch({ type: 'AUTH_SET_LOADING', payload: false }); // State: loading=false
  // 4 separate renders (unless batched)

  // ✅ Better — batch or combine into one action
  batch(() => {
    dispatch({ type: 'AUTH_SET_LOADING', payload: true });
    dispatch({ type: 'AUTH_SET_USER', payload: user });
    dispatch({ type: 'AUTH_SET_TOKEN', payload: token });
    dispatch({ type: 'AUTH_SET_LOADING', payload: false });
  }); // 1 render
};
```

---

### Q36 🟡 Medium
**How does dispatch work differently with Redux-Thunk vs Redux-Saga middleware?**

**Answer:**
- **Without middleware:** `dispatch` only accepts plain objects
- **Thunk:** `dispatch` accepts functions; thunk middleware intercepts and calls the function
- **Saga:** `dispatch` still only accepts plain objects, but Saga middleware watches for specific actions and handles async side effects externally

```typescript
// Plain Redux — only objects
dispatch({ type: 'INCREMENT' }); // ✅
dispatch(() => {});               // ❌ throws without middleware

// With Thunk middleware
const thunkAction = () => async (dispatch: AppDispatch) => {
  await api.call();
  dispatch({ type: 'DONE' });
};
dispatch(thunkAction()); // ✅ thunk middleware intercepts the function

// With Saga middleware
// Saga watches for 'FETCH_REQUEST', so just dispatch the trigger action
dispatch({ type: 'FETCH_REQUEST', payload: { id: '1' } }); // ✅
// Saga intercepts, calls API, then dispatches FETCH_SUCCESS
// Your component code stays clean — no async logic here
```

---

### Q37 🟡 Medium
**What happens if you dispatch inside a reducer?**

**Answer:**
Dispatching inside a reducer is a **critical error**. It causes infinite dispatch loops, state corruption, and Redux will throw a warning ("Reducers may not dispatch actions"). Reducers must be pure and synchronous.

```typescript
// ❌ NEVER dispatch inside a reducer
const badReducer = (state = initialState, action: AnyAction) => {
  if (action.type === 'SOME_ACTION') {
    store.dispatch({ type: 'ANOTHER_ACTION' }); // ❌ infinite loop!
    return { ...state };
  }
  return state;
};

// ✅ If you need chained actions, use middleware
// Thunk approach
const chainedAction = () => (dispatch: AppDispatch) => {
  dispatch({ type: 'FIRST_ACTION' });
  dispatch({ type: 'SECOND_ACTION' });
};

// Saga approach
function* watchFirstAction() {
  yield takeEvery('FIRST_ACTION', function* () {
    yield put({ type: 'SECOND_ACTION' }); // ✅ safe, outside reducer
  });
}
```

---

### Q38 🟡 Medium
**How do you type `useDispatch` properly in TypeScript for thunks and sagas?**

**Answer:**
Create a typed `AppDispatch` from the store type and use it everywhere instead of the default `Dispatch`.

```typescript
// store/index.ts
import { createStore, applyMiddleware } from 'redux';
import createSagaMiddleware from 'redux-saga';
import rootReducer from './rootReducer';

const sagaMiddleware = createSagaMiddleware();
const store = createStore(rootReducer, applyMiddleware(sagaMiddleware));

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;

// hooks/useAppDispatch.ts — typed hook
import { useDispatch } from 'react-redux';
export const useAppDispatch = () => useDispatch<AppDispatch>();

// hooks/useAppSelector.ts — typed selector hook
import { useSelector, TypedUseSelectorHook } from 'react-redux';
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;

// In component — no more type assertions
const MyComponent = () => {
  const dispatch = useAppDispatch(); // ✅ fully typed
  const user = useAppSelector(s => s.auth.user); // ✅ RootState inferred
};
```

---

### Q39 🔴 Hard
**What is "dispatch storm" and how did you handle it in your ERP app?**

**Answer:**
A dispatch storm is when many rapid dispatches (e.g. from a real-time data stream or scroll event) cause too many re-renders, degrading performance. Solutions: debounce dispatch calls, batch updates, or consolidate actions.

```typescript
import { useCallback, useRef } from 'react';
import { useAppDispatch } from '../hooks/useAppDispatch';

// Solution 1: Debounce dispatch
const useDebounceDispatch = (delay = 300) => {
  const dispatch = useAppDispatch();
  const timerRef = useRef<NodeJS.Timeout | null>(null);

  return useCallback((action: any) => {
    if (timerRef.current) clearTimeout(timerRef.current);
    timerRef.current = setTimeout(() => {
      dispatch(action);
    }, delay);
  }, [dispatch, delay]);
};

// Solution 2: Throttle real-time data updates
// In saga — throttle effect
function* watchLocationUpdates() {
  yield throttle(500, 'LOCATION_UPDATE', handleLocationUpdate);
}

// Solution 3: Batch from real-time WebSocket stream
const handleWebSocketMessage = (messages: any[]) => {
  batch(() => {
    messages.forEach(msg => dispatch({ type: msg.type, payload: msg.data }));
  });
};
```

---

### Q40 🔴 Hard
**Explain how `useSelector` subscribes to the store. When does it trigger a re-render?**

**Answer:**
`useSelector` subscribes to the Redux store. After every dispatch, it runs the selector function and compares the result with the previous result using **strict equality (`===`)** by default. If the reference changed, the component re-renders.

```typescript
import { useSelector, shallowEqual } from 'react-redux';

// ❌ New array reference every render — always re-renders
const items = useSelector((s: RootState) => s.employees.list.filter(e => e.active));

// ✅ Option 1: Memoized selector with reselect (see Part 06)
const selectActiveEmployees = createSelector(
  (s: RootState) => s.employees.list,
  list => list.filter(e => e.active) // recomputed only if list changes
);
const items = useSelector(selectActiveEmployees);

// ✅ Option 2: shallowEqual for object comparison
const { name, email } = useSelector(
  (s: RootState) => ({ name: s.auth.user?.name, email: s.auth.user?.email }),
  shallowEqual // compares object fields, not reference
);

// ✅ Option 3: Select primitives where possible — never re-creates
const userName = useSelector((s: RootState) => s.auth.user?.name); // string comparison
const isLoading = useSelector((s: RootState) => s.employees.loading); // boolean comparison
```

---

## E. combineReducers

---

### Q41 🟢 Easy
**What is `combineReducers` and what state shape does it create?**

**Answer:**
`combineReducers` merges multiple reducers into one root reducer. The key names in the passed object become the top-level keys of the state tree.

```typescript
import { combineReducers } from 'redux';

const rootReducer = combineReducers({
  auth:       authReducer,      // → state.auth
  employees:  employeeReducer,  // → state.employees
  attendance: attendanceReducer,// → state.attendance
  payroll:    payrollReducer,   // → state.payroll
  ui:         uiReducer,        // → state.ui
});

// Resulting state shape:
/*
{
  auth: { user: null, token: null, loading: false },
  employees: { list: [], loading: false, error: null },
  attendance: { records: [], selectedDate: null },
  payroll: { data: [], processed: false },
  ui: { theme: 'light', activeTab: 'dashboard' }
}
*/

export type RootState = ReturnType<typeof rootReducer>;
```

---

### Q42 🟢 Easy
**Does `combineReducers` create a deep merge or shallow merge of slices?**

**Answer:**
Shallow merge only. Each key in the object you pass to `combineReducers` maps to a top-level state slice. Each reducer is 100% responsible for its own slice — `combineReducers` never merges nested state across reducers.

```typescript
// Each reducer manages ONLY its own slice
const rootReducer = combineReducers({
  auth: authReducer,        // auth reducer CANNOT touch state.employees
  employees: employeeReducer, // employee reducer CANNOT touch state.auth
});

// ✅ auth reducer
const authReducer = (state = authInit, action: AnyAction) => {
  // `state` here is ONLY state.auth, not the full state
  switch (action.type) {
    case 'LOGIN_SUCCESS': return { ...state, user: action.payload };
    default: return state;
  }
};

// To share state between slices, use:
// 1. Selectors (read only)
// 2. Sagas that dispatch multiple actions
// 3. Root reducer wrapper
```

---

### Q43 🟢 Easy
**What warning does Redux log when a slice returns `undefined`?**

**Answer:**
Redux logs: *"Reducer [key] returned undefined during initialization."* This happens when a reducer doesn't have a default value for its state parameter. Always provide `initialState` as the default parameter.

```typescript
// ❌ Causes warning — no default state
const badReducer = (state: any, action: AnyAction) => {
  return state; // undefined on first call → warning
};

// ✅ Always provide initialState
interface AuthState { user: null | User; token: null | string }
const initialState: AuthState = { user: null, token: null };

const authReducer = (state: AuthState = initialState, action: AnyAction): AuthState => {
  return state;
};
```

---

### Q44 🟡 Medium
**How do you nest `combineReducers` for a large feature-based app?**

**Answer:**
You can nest `combineReducers` calls. Each feature can have its own combined reducer, and these are composed into the root.

```typescript
// features/hrms/reducer.ts
const hrmsReducer = combineReducers({
  employees:  employeeReducer,
  attendance: attendanceReducer,
  leaves:     leaveReducer,
  payroll:    payrollReducer,
});

// features/crm/reducer.ts
const crmReducer = combineReducers({
  leads:    leadReducer,
  contacts: contactReducer,
  deals:    dealReducer,
});

// features/inventory/reducer.ts
const inventoryReducer = combineReducers({
  items:    itemReducer,
  stock:    stockReducer,
  orders:   orderReducer,
});

// rootReducer.ts
const rootReducer = combineReducers({
  hrms:      hrmsReducer,      // → state.hrms.employees, state.hrms.attendance
  crm:       crmReducer,       // → state.crm.leads
  inventory: inventoryReducer, // → state.inventory.items
  auth:      authReducer,
  ui:        uiReducer,
});

// Access nested state
const employees = useAppSelector(s => s.hrms.employees.list);
const leads = useAppSelector(s => s.crm.leads.list);
```

---

### Q45 🟡 Medium
**How do you replace a slice reducer dynamically (code splitting in React Native)?**

**Answer:**
Use `store.replaceReducer`. This is useful for lazy-loading feature reducers only when needed — reducing initial bundle size.

```typescript
// store/index.ts
const staticReducers = {
  auth: authReducer,
  ui:   uiReducer,
};

const createRootReducer = (asyncReducers = {}) =>
  combineReducers({ ...staticReducers, ...asyncReducers });

const store = createStore(createRootReducer());

// Attach new reducer dynamically when a feature loads
export const injectReducer = (key: string, reducer: Reducer) => {
  if ((store as any).asyncReducers[key]) return; // already injected
  (store as any).asyncReducers[key] = reducer;
  store.replaceReducer(createRootReducer((store as any).asyncReducers));
};

// Usage — in lazy-loaded Inventory screen
import { injectReducer } from '../../store';
import inventoryReducer from './reducer';

const InventoryModule = () => {
  useEffect(() => {
    injectReducer('inventory', inventoryReducer); // loaded on demand
  }, []);
  // ...
};
```

---

### Q46 🟡 Medium
**Can two slices from `combineReducers` communicate with each other?**

**Answer:**
Reducers cannot directly access sibling slices. They only receive their own slice as `state`. To coordinate between slices use: selectors (read), sagas (write side effects), or a root reducer wrapper.

```typescript
// ❌ IMPOSSIBLE inside reducer — employee reducer cannot see auth state
const employeeReducer = (state = empInit, action: AnyAction) => {
  // `state` is ONLY state.employees, not RootState
  // Cannot do: if (rootState.auth.user.role === 'admin') ...
};

// ✅ Saga can access full state
function* fetchEmployeesSaga() {
  const userRole: string = yield select((s: RootState) => s.auth.user?.role);

  if (userRole === 'admin') {
    yield call(api.getAllEmployees);
  } else {
    yield call(api.getMyTeam);
  }
}

// ✅ Selector composes across slices (read-only)
const selectCanEditPayroll = createSelector(
  (s: RootState) => s.auth.user?.role,
  (s: RootState) => s.payroll.isLocked,
  (role, isLocked) => role === 'admin' && !isLocked
);
```

---

### Q47 🔴 Hard
**What is the performance implication of `combineReducers` calling all reducers on every dispatch?**

**Answer:**
Every dispatch calls ALL reducers, even if they don't handle that action type. For 20+ slices, this means 20 function calls per dispatch. This is generally fast (pure functions, no I/O), but can be a concern for:
- Very high-frequency dispatches (real-time, scroll events)
- Reducers with expensive computations in their default case

```typescript
// ✅ Default case must be O(1) — just return state
const goodReducer = (state = init, action: AnyAction) => {
  switch (action.type) {
    case 'MY_ACTION': return { ...state, value: action.payload };
    default: return state; // ✅ O(1) reference return — free
  }
};

// ❌ Expensive default case — runs on EVERY dispatch in the app
const badReducer = (state = init, action: AnyAction) => {
  switch (action.type) {
    case 'MY_ACTION': return { ...state };
    default:
      // ❌ This expensive logic runs on every dispatch everywhere!
      return { ...state, computed: heavyComputation(state.items) };
  }
};

// Move heavy computation to selectors (memoized) — see Part 06
const selectComputed = createSelector(
  (s: RootState) => s.mySlice.items,
  items => heavyComputation(items) // runs only when items change
);
```

---

### Q48 🔴 Hard
**How do you add TypeScript type safety to `combineReducers` so `RootState` is always accurate?**

**Answer:**
Derive `RootState` from `ReturnType<typeof rootReducer>` — TypeScript infers it automatically. Never write the type manually.

```typescript
// rootReducer.ts
import { combineReducers } from 'redux';

const rootReducer = combineReducers({
  auth:       authReducer,
  employees:  employeeReducer,
  attendance: attendanceReducer,
  payroll:    payrollReducer,
  ui:         uiReducer,
});

// ✅ Auto-inferred — always in sync with actual reducers
export type RootState = ReturnType<typeof rootReducer>;

// RootState is:
// {
//   auth: AuthState,
//   employees: EmployeeState,
//   attendance: AttendanceState,
//   payroll: PayrollState,
//   ui: UiState
// }

// Typed selector — full autocomplete in VS Code
const selectUserRole = (state: RootState) => state.auth.user?.role;

// Typed useSelector hook
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;

// Usage — no type assertions needed
const role = useAppSelector(s => s.auth.user?.role); // TS knows type
```

---

## F. Immutability

---

### Q49 🟢 Easy
**Why is immutability important in Redux?**

**Answer:**
Redux relies on reference equality checks to detect state changes. If you mutate state in-place, the reference stays the same, Redux thinks nothing changed, and `useSelector` won't trigger re-renders.

```typescript
// ❌ Mutation — React WILL NOT re-render
const badReducer = (state = { count: 0 }, action: AnyAction) => {
  if (action.type === 'INCREMENT') {
    state.count++; // ❌ same reference
    return state;  // === previous state → no re-render
  }
  return state;
};

// ✅ New reference — React WILL re-render
const goodReducer = (state = { count: 0 }, action: AnyAction) => {
  if (action.type === 'INCREMENT') {
    return { ...state, count: state.count + 1 }; // new object
  }
  return state;
};
```

---

### Q50 🟢 Easy
**What are the common immutable update patterns in JavaScript?**

**Answer:**

```typescript
const state = {
  user: { name: 'Devesh', age: 25 },
  skills: ['React Native', 'Redux'],
  meta: { active: true }
};

// Update top-level property
const s1 = { ...state, meta: { ...state.meta, active: false } };

// Add to array
const s2 = { ...state, skills: [...state.skills, 'TypeScript'] };

// Remove from array
const s3 = { ...state, skills: state.skills.filter(s => s !== 'Redux') };

// Update item in array
const s4 = {
  ...state,
  skills: state.skills.map(s => s === 'Redux' ? 'Redux Saga' : s)
};

// Update nested object in array
interface Employee { id: string; salary: number }
const employees: Employee[] = [{ id: '1', salary: 50000 }];

const updatedEmployees = employees.map(emp =>
  emp.id === '1' ? { ...emp, salary: 60000 } : emp
);

// Delete key from object
const { user, ...withoutUser } = state; // removes user key
```

---

### Q51 🟡 Medium
**What is `immer` and how does it help with deeply nested immutable updates?**

**Answer:**
`immer` lets you write "mutating" code that actually produces a new immutable state via a structural clone (Proxy). Redux Toolkit uses immer by default inside `createSlice` reducers.

```typescript
import produce from 'immer';

interface AppState {
  employees: {
    byId: Record<string, { name: string; address: { city: string } }>;
  };
}

const state: AppState = {
  employees: {
    byId: { e1: { name: 'Devesh', address: { city: 'Delhi' } } },
  },
};

// ❌ Without immer — verbose for deep nesting
const newState = {
  ...state,
  employees: {
    ...state.employees,
    byId: {
      ...state.employees.byId,
      e1: {
        ...state.employees.byId.e1,
        address: {
          ...state.employees.byId.e1.address,
          city: 'Noida',
        },
      },
    },
  },
};

// ✅ With immer — reads like mutation, produces immutable state
const newStateImmer = produce(state, draft => {
  draft.employees.byId.e1.address.city = 'Noida'; // safe
});

// In RTK createSlice (immer built-in)
const employeeSlice = createSlice({
  name: 'employees',
  initialState,
  reducers: {
    updateCity: (state, action) => {
      state.byId[action.payload.id].address.city = action.payload.city; // ✅ immer
    },
  },
});
```

---

### Q52 🟡 Medium
**How do you detect accidental mutations in Redux during development?**

**Answer:**
Use `redux-immutable-state-invariant` middleware in development. It freezes state objects using `Object.freeze` and throws if any mutation is detected.

```typescript
import { createStore, applyMiddleware } from 'redux';

const getMiddleware = () => {
  if (__DEV__) {
    // Only in development — adds overhead
    const { default: immutableStateInvariant } = require('redux-immutable-state-invariant');
    return applyMiddleware(immutableStateInvariant(), sagaMiddleware);
  }
  return applyMiddleware(sagaMiddleware);
};

const store = createStore(rootReducer, getMiddleware());

// Now if you accidentally mutate state in a reducer:
// ❌ state.items.push(item)
// → Redux throws: "A state mutation was detected..."
// → Points to exact action type that caused the mutation
```

---

### Q53 🔴 Hard
**What is structural sharing in immutable state updates and why does it matter for memory?**

**Answer:**
When you spread state (`{ ...state, x: newX }`), unchanged parts still reference the same objects in memory — they aren't copied. Only the changed subtrees are new. This is structural sharing. It makes immutable updates memory-efficient.

```typescript
const state = {
  auth:      { user: 'Devesh' },    // ref: 0x001
  employees: { list: [] },           // ref: 0x002
  ui:        { theme: 'dark' },      // ref: 0x003
};

// Dispatch: update only auth.user
const newState = {
  ...state,
  auth: { ...state.auth, user: 'Rahul' }, // new ref: 0x004
};

// Memory layout after update:
// newState.auth      → 0x004 (new object — changed)
// newState.employees → 0x002 (SAME reference — not copied!)
// newState.ui        → 0x003 (SAME reference — not copied!)

// useSelector benefits:
// Components subscribed to state.employees will NOT re-render
// because the reference is identical (0x002 === 0x002)
console.log(newState.employees === state.employees); // true ✅
console.log(newState.auth === state.auth);           // false ✅
```

---

### Q54 🔴 Hard
**When does spreading state (`{ ...state }`) fail to create proper immutability? Give a real example.**

**Answer:**
The spread operator is **shallow**. It only copies top-level keys. Nested arrays and objects still share the same reference. Mutating a nested property mutates both the old and new state.

```typescript
const state = {
  employees: {
    list: [{ id: '1', skills: ['React', 'Redux'] }]
  }
};

// ❌ Shallow spread — appears immutable but isn't
const newState = { ...state };
newState.employees.list[0].skills.push('TypeScript'); // ❌ mutates state AND newState!

// Because:
console.log(newState.employees === state.employees); // true — SAME reference
console.log(newState.employees.list === state.employees.list); // true — SAME reference

// ✅ Must spread every modified level
const correctState = {
  ...state,
  employees: {
    ...state.employees,
    list: state.employees.list.map(emp =>
      emp.id === '1'
        ? { ...emp, skills: [...emp.skills, 'TypeScript'] } // new array
        : emp
    ),
  },
};

// Or use immer for deep nesting (see Q51)
```

---

## G. State Shape Design

---

### Q55 🟡 Medium
**What are the principles of good Redux state shape design?**

**Answer:**
1. **Flat, not nested** — avoid deeply nested objects
2. **Normalize collections** — use `byId` + `allIds` for lists
3. **Separate UI state from domain data** — don't mix server data with loading flags in the same object
4. **Avoid redundant derived state** — compute from existing state via selectors
5. **Mirror API response shape minimally** — transform to what UI needs

```typescript
// ❌ Poor shape — deeply nested, redundant derived data
{
  data: {
    employees: {
      allEmployees: {
        employeesList: Employee[],
        totalCount: number, // ❌ derived — count of list
        isLoading: boolean,
        hasError: boolean,
        errorMessage: string
      }
    }
  }
}

// ✅ Good shape — flat, normalized, clean
{
  employees: {
    byId: Record<string, Employee>,
    allIds: string[],
    status: 'idle' | 'loading' | 'succeeded' | 'failed',
    error: string | null
  }
}
// totalCount → employees.allIds.length (derived via selector)
```

---

### Q56 🟡 Medium
**How do you design Redux state for a multi-module ERP app (HRMS + CRM + Inventory)?**

**Answer:**

```typescript
// state shape for Qurilo-style ERP
interface RootState {
  // Auth — small, always loaded
  auth: {
    user: User | null;
    token: string | null;
    permissions: string[];
    status: AsyncStatus;
  };

  // HRMS module
  employees: {
    byId: Record<string, Employee>;
    allIds: string[];
    status: AsyncStatus;
    selectedId: string | null;
  };
  attendance: {
    records: AttendanceRecord[];
    selectedDate: string;
    status: AsyncStatus;
  };
  payroll: {
    byEmployeeId: Record<string, PayrollEntry>;
    processedMonth: string | null;
    status: AsyncStatus;
  };

  // CRM module
  leads: {
    byId: Record<string, Lead>;
    allIds: string[];
    filters: LeadFilters;
    status: AsyncStatus;
  };

  // Inventory
  inventory: {
    items: Record<string, Item>;
    lowStockAlerts: string[]; // item ids
    status: AsyncStatus;
  };

  // UI state — NOT persisted
  ui: {
    activeModule: 'hrms' | 'crm' | 'inventory';
    sidebarOpen: boolean;
    theme: 'light' | 'dark';
    toasts: Toast[];
  };
}
```

---

### Q57 🟡 Medium
**How do you handle pagination state in Redux?**

**Answer:**

```typescript
interface PaginatedState<T> {
  byId: Record<string, T>;
  allIds: string[];
  pagination: {
    currentPage: number;
    totalPages: number;
    pageSize: number;
    totalCount: number;
    hasNextPage: boolean;
  };
  status: AsyncStatus;
  error: string | null;
}

const initialEmployeeState: PaginatedState<Employee> = {
  byId: {},
  allIds: [],
  pagination: {
    currentPage: 1,
    totalPages: 0,
    pageSize: 20,
    totalCount: 0,
    hasNextPage: false,
  },
  status: 'idle',
  error: null,
};

// Reducer
case 'FETCH_EMPLOYEES_PAGE_SUCCESS':
  const { items, totalCount, currentPage, pageSize } = action.payload;
  const newById = { ...state.byId };
  const newAllIds = [...state.allIds];

  items.forEach((emp: Employee) => {
    if (!newById[emp.id]) {
      newById[emp.id] = emp;
      newAllIds.push(emp.id);
    }
  });

  return {
    ...state,
    byId: newById,
    allIds: newAllIds,
    status: 'succeeded',
    pagination: {
      currentPage,
      pageSize,
      totalCount,
      totalPages: Math.ceil(totalCount / pageSize),
      hasNextPage: currentPage * pageSize < totalCount,
    },
  };
```

---

### Q58 🔴 Hard
**How do you design state for an offline-first React Native app with sync capabilities?**

**Answer:**

```typescript
interface SyncableEntity<T> {
  data: T;
  syncStatus: 'synced' | 'pending' | 'conflict' | 'error';
  localUpdatedAt: number;
  serverUpdatedAt: number | null;
  pendingOperation: 'create' | 'update' | 'delete' | null;
}

interface OfflineState {
  attendance: {
    byId: Record<string, SyncableEntity<AttendanceRecord>>;
    allIds: string[];
    pendingSyncIds: string[];     // IDs waiting to sync
    lastSyncedAt: number | null;
    isOnline: boolean;
  };
}

// Actions for offline-first
const offlineReducer = (state = offlineInit, action: AnyAction) => {
  switch (action.type) {
    // Local operation — save immediately, mark pending
    case 'CHECK_IN_LOCAL':
      return {
        ...state,
        byId: {
          ...state.byId,
          [action.payload.id]: {
            data: action.payload,
            syncStatus: 'pending',
            localUpdatedAt: Date.now(),
            serverUpdatedAt: null,
            pendingOperation: 'create',
          },
        },
        allIds: [...state.allIds, action.payload.id],
        pendingSyncIds: [...state.pendingSyncIds, action.payload.id],
      };

    // After successful sync
    case 'SYNC_SUCCESS':
      return {
        ...state,
        byId: {
          ...state.byId,
          [action.payload.id]: {
            ...state.byId[action.payload.id],
            syncStatus: 'synced',
            serverUpdatedAt: action.payload.serverTimestamp,
            pendingOperation: null,
          },
        },
        pendingSyncIds: state.pendingSyncIds.filter(id => id !== action.payload.id),
        lastSyncedAt: Date.now(),
      };

    default:
      return state;
  }
};
```

---

### Q59 🔴 Hard
**What data should NOT go into Redux state? Give a decision framework.**

**Answer:**
Not all state belongs in Redux. Use this decision framework:

```typescript
// Decision: Is this data shared across multiple unrelated components?
// Decision: Does this data need to persist across navigation?
// Decision: Does this data need time-travel/DevTools debugging?

// ❌ Keep OUT of Redux:
// 1. Form state (local until submit)
const FormExample = () => {
  const [name, setName] = useState(''); // ✅ local state
};

// 2. UI-only transient state
const [isDropdownOpen, setDropdownOpen] = useState(false); // ✅ component state

// 3. Server cache (use React Query / SWR instead)
// const { data } = useQuery('employees', fetchEmployees); // ✅ react-query

// 4. Refs / DOM state
const scrollRef = useRef(null); // ✅ useRef

// ✅ PUT in Redux:
// 1. Auth state — needed everywhere
// 2. User permissions — drives feature visibility app-wide
// 3. Feature data shared by 3+ unrelated screens (Employee list)
// 4. State that needs to survive screen unmount (wizard multi-step)
// 5. Real-time data updates from WebSocket
// 6. Offline queue / sync state

// Rule of thumb:
// Component state → useState
// Cross-component in same tree → Context or prop drill
// Cross-screen / global → Redux
// Server data with caching → React Query
```

---

### Q60 🔴 Hard
**How did you design the Redux state for your Debt Relief India fintech app? Walk through key decisions.**

**Answer:**
This is a behavioural + technical hybrid. Connect your real experience with state design principles.

```typescript
// Debt Relief India — state architecture
interface DebtReliefRootState {
  // Auth & KYC — security-sensitive, encrypted in storage
  auth: {
    user: User | null;
    token: string | null;         // never log, never display
    kycStatus: 'pending' | 'submitted' | 'verified' | 'rejected';
    sessionExpiresAt: number | null;
    status: AsyncStatus;
  };

  // Loan data — core domain
  loans: {
    byId: Record<string, Loan>;
    allIds: string[];
    activeLoanId: string | null;  // currently viewed loan
    status: AsyncStatus;
  };

  // EMI/Repayment — financial data, needs accuracy
  repayment: {
    schedule: EMIEntry[];
    nextDueDate: string | null;
    overdueAmount: number;
    totalOutstanding: number;     // derived on server, not recomputed client-side
    status: AsyncStatus;
  };

  // Payment flow — transient, cleared after success/failure
  payment: {
    currentTransaction: {
      amount: number;
      loanId: string;
      razorpayOrderId: string | null;
      method: 'upi' | 'card' | 'netbanking' | null;
    } | null;
    history: PaymentRecord[];
    status: AsyncStatus;
    error: string | null;
  };

  // Documents — KYC uploads
  documents: {
    byType: Record<'aadhaar' | 'pan' | 'bankStatement', DocumentEntry | null>;
    uploadStatus: Record<string, AsyncStatus>;
  };

  // UI — not persisted
  ui: {
    activeTab: 'home' | 'emi' | 'documents' | 'profile';
    biometricEnabled: boolean;
    notificationBadge: number;
  };
}

// Key design decisions:
// 1. payment.currentTransaction → cleared on success/failure (no stale payment state)
// 2. totalOutstanding → trust server value, don't derive client-side (compliance)
// 3. token → stored encrypted in AsyncStorage, NOT plain in Redux persist
// 4. kycStatus → drives entire app navigation flow (onboarding vs main app)
```

---

## ✅ Progress Checklist

- [ ] Section A: Store & Setup (Q1–Q10)
- [ ] Section B: Actions & Action Creators (Q11–Q20)
- [ ] Section C: Reducers & Pure Functions (Q21–Q32)
- [ ] Section D: Dispatch & Data Flow (Q33–Q40)
- [ ] Section E: combineReducers (Q41–Q48)
- [ ] Section F: Immutability (Q49–Q54)
- [ ] Section G: State Shape Design (Q55–Q60)

---

## 🔗 Next File

👉 [`02-redux-saga.md`](./02-redux-saga.md) — 100 Questions on Effects, Workers, Watchers, Channels & Error Handling

---

*Part 03 · Redux Fundamentals · 60/500 Questions · Devesh Kumar Singh*
