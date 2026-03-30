# Solution: Engineering Test (r-nep-kz/eng-test)

## 1. Authentication (JWT & Alternatives)

**Experience & Strategy:**
JWT has long been the "golden standard" for stateless architectures. However, in large-scale systems or those with high security requirements (e.g., fintech), its conceptual flaws become critical:
- **Lack of Instant Invalidation:** Revoking a token before TTL expiration requires stateful mechanisms (Blacklists in Redis), which negates the stateless benefit.
- **Traffic Overhead:** Base64-encoded JSON in every header inflates request size, especially with many roles/permissions.

**Alternative Approach (Used in my practice):**
In high-load microservice environments behind an API Gateway, I've successfully implemented **Opaque Tokens (Reference Tokens)**.
- **Mechanism:** The client receives a short, cryptographically secure random string. The API Gateway exchanges this for user context (or signs a short-lived internal JWT) via an ultra-fast cache (Redis).
- **Benefits:** Instant session invalidation (just delete from Redis), minimized external traffic, and internal payload obfuscation.

**When to choose JWT?**
I would choose JWT for public APIs with massive, independent consumers where the cost of checking a database/cache for every request outweighs the risks of delayed invalidation.

## 2. Identification (UUID)

**Problems with UUID v4 in scale:**
In my experience, random UUID v4 often becomes a performance bottleneck in RDBMS (PostgreSQL/MySQL) due to B-Tree index structure.
- **Index Fragmentation:** Since UUID v4 is random, new records are inserted at arbitrary points in the tree, causing frequent page splits and degrading INSERT performance.
- **Cache Efficiency:** At 16 bytes (vs 4/8 bytes for INT/BIGINT), large tables with multiple secondary indexes suffer from index bloat, reducing the efficiency of the database memory buffer (cache misses).

**Solution:**
For decentralized generation with high-speed indexing, I prefer **UUID v7** (time-ordered). It maintains lexicographical sorting by time, allowing B-Tree indexes to append records sequentially, achieving performance similar to auto-increment keys while preserving global uniqueness.

## 3. Code Organization (DI & Frameworks)

**The Value of Dependency Injection (DI):**
The primary benefit of DI is **Testability**. Decoupling logic through IoC (Inversion of Control) allows us to write robust unit tests by easily swapping real database or API repositories with mocks. This leads to 90%+ test coverage without requiring a heavy Docker environment.

**Challenges with NestJS/Decorators:**
- **Magic & Implicit Flow:** Heavy use of decorators (eflect-metadata) can hide the actual execution flow. Understanding the order of Interceptors, Guards, and Pipes becomes complex in large projects.
- **Runtime DI Errors:** Circular dependencies or "Unknown provider" errors often only appear at startup, making debugging tedious in large monoliths compared to compile-time DI.

## 4. Reactivity (React)

**When Reactivity is a Hindrance:**
The main challenge in React's reactivity is **Implicit Cascade Re-renders**.
In high-performance UIs (interactive maps, real-time charts, Canvas/WebGL), binding cursor coordinates or 1000s of object positions to useState triggers heavy Virtual DOM reconciliation (60fps), leading to frame drops.

**Alternative Rendering Principle:**
In such cases, I bypass React's reactive cycle for specific components. I use useRef for mutable state and update the DOM/Canvas imperatively via equestAnimationFrame. React is used only as a mounter for the root element.

**React Implementation Pain Points:**
- **Lack of Granularity:** Unlike Solid.js or Svelte, React doesn't know *which* property changed. It re-renders the whole component tree, forcing developers to use "optimization mines" like useMemo, useCallback, and React.memo.
- **useEffect Complexity:** The paradigm of synchronizing effects with state is prone to race conditions, infinite loops, and stale closures if the dependency array is slightly off.

---

## Roadmap: Enterprise System Architecture

### Phase 1: Foundation (Security & Identity)
- [ ] Implement UUID v7 for all write-intensive tables.
- [ ] Deploy API Gateway with Opaque Token support + Redis session management.
- [ ] Define Docker-base images (Node.js slim) and resource limits (IaC).

### Phase 2: Core Services & DI
- [ ] Model domain entities with Strict Typing.
- [ ] Configure IoC Container (registration of services, repositories, adapters).
- [ ] Implement Event-Driven messaging (e.g., RabbitMQ) with idempotency.

### Phase 3: UI & Reactivity
- [ ] Build Atomic Component library with TypeScript.
- [ ] Establish State Management strategy (minimizing useEffect).
- [ ] Optimize performance-critical renders (Canvas/Web Workers).

### Phase 4: Validation & Hardening
- [ ] Achieve 90%+ Test Coverage (Unit/Integration/E2E).
- [ ] Perform Load Testing on B-Tree indexes with massive data volumes.

---

## Test Cases (Validation Scenarios)

### Case 1: Session Invalidation (Auth)
- **Scenario:** User clicks "Logout from all devices" after suspicious activity.
- **Expected:** Redis key is deleted; all subsequent requests return 401 Unauthorized in <50ms (Zero Propagation Delay).

### Case 2: Write Performance at Scale (Identity)
- **Scenario:** Load testing Events table with 10M+ records using UUID v4 vs UUID v7.
- **Expected:** UUID v7 shows stable insert speed with minimal page splits compared to UUID v4.

### Case 3: Dependency Isolation (DI)
- **Scenario:** External Billing API is down.
- **Expected:** Unit test with Mock Billing verifies that the system handles failure gracefully (Retry/Circuit Breaker) without logic changes.

### Case 4: FPS Stability (React Performance)
- **Scenario:** Dynamically updating 1000+ items at 30fps.
- **Expected:** React Profiler shows no cascade re-renders in unrelated branches; Main Thread idle > 50%.
