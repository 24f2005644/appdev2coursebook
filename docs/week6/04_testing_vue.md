# 4. Testing Vue Applications



## Learning objectives

By the end of this topic, you should be able to:

- Explain the main ideas covered in **4. Testing Vue Applications**.
- Connect these ideas to modern web application development.
- Recognize the patterns, terminology, and trade-offs used in practice.

---

## 4.1 Testing Levels & Strategies

### Why Test?

Testing is the practice of **systematically verifying that your application behaves as expected** — both in isolation and as a whole. Without tests:
- Bugs discovered late (in production) are far more expensive to fix.
- Refactoring becomes risky — you can't be sure you haven't broken something.
- New developers have no safety net when modifying existing code.

There are **three primary levels** of testing for Vue applications, each with a different scope and purpose:

---

### 🔬 Level 1: Unit Testing

**Scope**: Individual components or functions **in isolation**.

- Tests a single unit of code — typically a single Vue component, a `computed` property, a `method`, or a utility function.
- The component is **mounted into a mock / virtual DOM** (not a real browser), so tests can run headlessly (e.g., in Node.js via CI pipelines).
- External dependencies (API calls, child components) are **mocked** — the unit under test is isolated from the rest of the system.

**What to unit test in Vue:**

| Target | Example |
|---|---|
| `data()` initialization | Does a counter start at 0? |
| `computed` properties | Does `fullName` return `"John Doe"` given `firstName` and `lastName`? |
| `methods` | Does `increment()` increase `count` by 1? |
| Conditional rendering | Is `.error-msg` present in the DOM when `isError` is `true`? |
| Event emission | Does a button click emit the `submit` event? |
| Props rendering | Does the component display the correct `title` prop? |

**Advantages:**
- ✅ Very fast to run (no browser, no network).
- ✅ Pinpoints exactly which unit is broken.
- ✅ Easiest to write and maintain.

**Disadvantages:**
- ❌ Doesn't test how components integrate with each other.
- ❌ Mocked dependencies may not reflect real behaviour.

---

### 🌐 Level 2: End-to-End (E2E) Testing

**Scope**: The **entire application stack** — frontend UI + backend APIs + database — as a real user would experience it.

- A real browser is automated (via tools like **Cypress** or **Playwright**) to simulate user actions: clicking buttons, filling forms, navigating pages.
- Verifies complete **user workflows** (user stories) end-to-end.
- Example workflow: *"User opens the login page → enters credentials → clicks Submit → is redirected to the dashboard → sees their username in the header."*

**What E2E tests verify:**

- Navigation and routing work correctly.
- API calls are made and responses are handled properly.
- Data persists correctly (e.g., a form submission saves to the database).
- Authentication and authorization flows work.

**Advantages:**
- ✅ Tests the system exactly as a user experiences it.
- ✅ Catches integration bugs that unit tests miss.
- ✅ High confidence that the whole system works together.

**Disadvantages:**
- ❌ Slow — can take minutes to run a full suite.
- ❌ Brittle — minor UI changes (e.g., renaming a button) break tests.
- ❌ Requires a running backend and test database.
- ❌ Harder to debug when they fail.

---

### 🖥️ Level 3: Cross-Browser Testing

**Scope**: Verifying that the application functions correctly and looks right across **different browsers and versions** (Chrome, Firefox, Safari, Edge, and legacy versions like IE11).

- Different browsers implement web standards differently — CSS rendering, JavaScript APIs, and HTML parsing can vary.
- Testing in every browser version is extremely resource-intensive.

**Cost vs. Benefit / Diminishing Returns:**

This is a key exam concept — there's a diminishing returns curve:

| Browser Coverage | Value | Cost |
|---|---|---|
| Testing in Chrome + Firefox | ✅ High — covers ~75%+ of users | Low |
| Adding Safari | ✅ Good — adds iOS/macOS users | Moderate |
| Adding Edge | ✅ Reasonable | Low |
| Adding IE11 | ⚠️ Very low — tiny user base today | Very High (major compatibility work) |
| Testing every version of every browser | ❌ Effectively zero marginal gain | Enormous |

> **Practical rule**: Focus cross-browser testing on browsers that represent a meaningful share of your user base. Use analytics data to make this decision. Don't spend engineering time supporting browsers used by <1% of your users.

**Tools for cross-browser testing**: BrowserStack, Sauce Labs (cloud-based; lets you run tests on real browsers without owning every device).

---

### The Testing Pyramid

A well-tested application has more low-level tests (fast, cheap) and fewer high-level tests (slow, expensive):

```
          /\
         /  \          ← E2E Tests (few, slow, expensive, high confidence)
        /----\
       /      \        ← Integration Tests (moderate number)
      /--------\
     /          \      ← Unit Tests (many, fast, cheap)
    /____________\
```

- Write **many unit tests** (they run in milliseconds).
- Write **some integration/component tests**.
- Write **a few critical E2E tests** for core user flows.

---

## 4.2 Test Mechanisms & Concepts

### Test Fixtures

A **test fixture** is the **set of preconditions and data** needed to run a test reliably. It defines the known state of the world *before* the test executes.

- Fixtures ensure tests are **deterministic** — they always produce the same result given the same inputs.
- Prevents tests from depending on each other (order independence).

```js
// Example: A fixture providing mock user data for component tests
const userFixture = {
  id: 1,
  name: 'Vinay Maurya',
  email: 'vinay@iitm.ac.in',
  role: 'student'
};

// Used in a test:
it('displays user name in the header', () => {
  const wrapper = mount(UserCard, {
    props: { user: userFixture }   // inject the fixture as a prop
  });
  expect(wrapper.find('h2').text()).toBe('Vinay Maurya');
});
```

**Types of fixtures:**
- **Data fixtures**: Pre-built mock objects (like the example above).
- **State fixtures**: A specific application state to test from (e.g., "user is logged in").
- **DOM fixtures**: A specific HTML structure to test against.

---

### Test Suites

A **test suite** is a **logical grouping of related tests** — typically all tests for a single component or module. Test suites help organise tests and make output readable.

In most testing frameworks, `describe()` creates a suite and `it()` (or `test()`) defines individual test cases:

```js
// Test suite for a LoginForm component
describe('LoginForm.vue', () => {

  // Sub-suite: initial state
  describe('on initial render', () => {
    it('renders an email input', () => { /* ... */ });
    it('renders a password input', () => { /* ... */ });
    it('has the submit button disabled', () => { /* ... */ });
  });

  // Sub-suite: validation
  describe('validation', () => {
    it('shows an error when email is invalid', () => { /* ... */ });
    it('shows an error when password is too short', () => { /* ... */ });
    it('enables submit button when form is valid', () => { /* ... */ });
  });

  // Sub-suite: submission
  describe('on form submit', () => {
    it('calls the login API with correct credentials', () => { /* ... */ });
    it('shows an error message on failed login', () => { /* ... */ });
    it('redirects to dashboard on success', () => { /* ... */ });
  });

});
```

**Benefits of suites:**
- Organised, readable test output.
- Shared `beforeEach` / `afterEach` setup and teardown per suite.
- Easy to run just one suite when debugging a specific component.

---

### Assertions

An **assertion** is a statement that **declares what you expect to be true** at a specific point in a test. If the assertion fails, the test fails.

Assertions answer the question: *"Did the component produce the output I expected?"*

```js
// Asserting DOM presence/absence
expect(wrapper.find('.error-message').exists()).toBe(true);   // Element IS in DOM
expect(wrapper.find('.success-banner').exists()).toBe(false); // Element NOT in DOM

// Asserting text content
expect(wrapper.find('h1').text()).toBe('Welcome back, Vinay');

// Asserting CSS classes
expect(wrapper.find('button').classes()).toContain('btn-primary');

// Asserting a data property value
expect(wrapper.vm.count).toBe(3);

// Asserting an emitted event
await wrapper.find('button').trigger('click');
expect(wrapper.emitted('increment')).toBeTruthy();
expect(wrapper.emitted('increment')[0]).toEqual([1]); // emitted with value 1
```

---

### Simulated Events

Testing doesn't mean clicking in a real browser — you **simulate** events programmatically:

```js
// Simulate user typing into an input
await wrapper.find('input[type="email"]').setValue('test@iitm.ac.in');

// Simulate a button click
await wrapper.find('button[type="submit"]').trigger('click');

// Simulate a form submission
await wrapper.find('form').trigger('submit');

// After simulating, assert the expected outcome
expect(wrapper.find('.success').text()).toBe('Form submitted!');
```

> **Why `await`?** Vue updates the DOM **asynchronously** after state changes. `await` waits for Vue's reactivity to flush (DOM update) before your assertion runs.

---

## 4.3 Tooling & Ecosystem

### The Testing Stack

A typical Vue testing setup combines **multiple tools**, each with a specific role:

```
Test Runner     →  Finds & runs test files, reports results
    +
Assertion Lib   →  Provides expect(...).toBe(...) syntax
    +
Vue Test Utils  →  Mounts Vue components, provides DOM querying helpers
    +
Mock Library    →  Fakes API calls, timers, or modules
```

---

### 🏃 Test Runners

| Tool | Description |
|---|---|
| **Mocha** | Lightweight, flexible test runner. Provides `describe()` / `it()` structure. Needs a separate assertion library (e.g., Chai). |
| **Jest** | All-in-one framework (runner + assertions + mocking). Created by Facebook. Currently most popular for Vue 3 projects. |
| **Vitest** | Modern, Vite-native test runner. API-compatible with Jest. Extremely fast (re-uses Vite's pipeline). **Recommended for new Vue projects using Vite.** |

---

### ✅ Assertion Libraries

| Tool | Description |
|---|---|
| **Chai** | Used alongside Mocha. Offers multiple assertion styles: `assert`, `expect`, and `should`. |
| **Jest / Vitest built-in** | Both include their own `expect()` assertion API — no separate library needed. |

**Chai assertion styles:**

```js
// 'assert' style (procedural)
assert.equal(wrapper.find('h1').text(), 'Hello World');

// 'expect' style (chainable — most common)
expect(wrapper.find('h1').text()).to.equal('Hello World');

// 'should' style (extends Object prototype — avoid in production)
wrapper.find('h1').text().should.equal('Hello World');
```

---

### 🔧 Vue Test Utils (`@vue/test-utils`)

The **official Vue testing library** — provides everything needed to mount and interact with Vue components in tests:

```bash
npm install --save-dev @vue/test-utils
```

**Key API:**

```js
import { mount, shallowMount } from '@vue/test-utils';
import MyComponent from '@/components/MyComponent.vue';

// mount: Renders component AND all child components
const wrapper = mount(MyComponent, {
  props: { title: 'Hello' },
  global: {
    stubs: { 'router-link': true }  // Stub out Vue Router
  }
});

// shallowMount: Renders component but STUBS all child components
// Useful for true unit isolation
const wrapper = shallowMount(MyComponent);

// Querying the DOM
wrapper.find('h1')           // Find first matching element
wrapper.findAll('li')        // Find all matching elements
wrapper.find('.btn').exists() // Check if element exists

// Interacting
await wrapper.find('button').trigger('click');
await wrapper.find('input').setValue('hello');

// Accessing component internals
wrapper.vm.myMethod()        // Call a component method
wrapper.vm.myData            // Access reactive data
wrapper.emitted('myEvent')   // Check emitted events
```

**`mount` vs `shallowMount`:**

| | `mount` | `shallowMount` |
|---|---|---|
| **Child components** | Fully rendered | Replaced with stubs |
| **Test scope** | Component + its children (integration) | Component in isolation (unit) |
| **Use when** | Testing interactions between parent and child | Testing a component's own logic only |

---

### Complete Unit Test Example

```js
// tests/unit/Counter.spec.js
import { mount } from '@vue/test-utils';
import Counter from '@/components/Counter.vue';

describe('Counter.vue', () => {

  it('renders the initial count as 0', () => {
    const wrapper = mount(Counter);
    expect(wrapper.find('[data-testid="count"]').text()).toBe('0');
  });

  it('increments count when + button is clicked', async () => {
    const wrapper = mount(Counter);
    await wrapper.find('[data-testid="increment-btn"]').trigger('click');
    expect(wrapper.find('[data-testid="count"]').text()).toBe('1');
  });

  it('does not go below 0 when decrement is clicked at 0', async () => {
    const wrapper = mount(Counter);
    await wrapper.find('[data-testid="decrement-btn"]').trigger('click');
    expect(wrapper.find('[data-testid="count"]').text()).toBe('0');
  });

  it('emits a "countChanged" event on increment', async () => {
    const wrapper = mount(Counter);
    await wrapper.find('[data-testid="increment-btn"]').trigger('click');
    expect(wrapper.emitted('countChanged')).toBeTruthy();
    expect(wrapper.emitted('countChanged')[0]).toEqual([1]);
  });

});
```

---

## Summary

```
Testing Vue Applications
├── 4.1 Testing Levels
│   ├── Unit Testing        → Single component/function in isolation; mock DOM; fast
│   ├── E2E Testing         → Full stack, real browser automation; slow but high confidence
│   └── Cross-browser       → Diminishing returns; focus on browser market share
│
├── 4.2 Core Concepts
│   ├── Fixtures            → Pre-built mock data/state for deterministic tests
│   ├── Test Suites         → describe() groupings of related it() test cases
│   ├── Assertions          → expect(actual).toBe(expected) — did we get what we wanted?
│   └── Simulated Events    → trigger('click'), setValue() — no real browser needed
│
└── 4.3 Tools
    ├── Mocha               → Flexible test runner (needs Chai for assertions)
    ├── Chai                → Assertion library (assert / expect / should styles)
    ├── Jest                → All-in-one runner + assertions + mocking (popular)
    ├── Vitest              → Vite-native, Jest-compatible, fastest (recommended new)
    └── Vue Test Utils      → Mount components, query DOM, simulate events
        ├── mount()         → Full render including child components
        └── shallowMount()  → Render component; stub children (true unit isolation)
```

## Further practice

- Revisit the examples in this topic and explain each step in your own words.
- Identify one place where the concept could be used in a web application.
- Test a small variation and note how the behavior changes.
