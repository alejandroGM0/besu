# Besu Plugin API Mentorship - Introductory PR Ideas

This document contains a curated list of well-scoped, introductory Pull Request ideas aimed at demonstrating your capability to the LF Decentralized Trust Mentors for Issue #82: "Refactoring and Evolution of the Besu Plugin API".

These ideas are mapped directly to the four core objectives of the mentorship project. They are designed to be small enough to act as an initial contribution, but impactful enough to show you understand the architecture.

---

## 1. Implement Minimum Supported Besu Version for Plugins
**Objective:** API Versioning System
**Difficulty:** Low / Beginner

**Problem:**
Currently, the `BesuPlugin` interface allows plugins to report their own version via `getVersion()`. However, plugins cannot declare which version of the Besu Plugin API they were compiled against or require. This makes backward/forward compatibility tracking impossible when the API changes.

**Proposed Implementation:**
1. Open `plugin-api/src/main/java/org/hyperledger/besu/plugin/BesuPlugin.java`.
2. Add a new default method:
   ```java
   default Optional<String> getMinimumSupportedBesuVersion() {
       return Optional.empty();
   }
   ```
3. Open `app/src/main/java/org/hyperledger/besu/services/BesuPluginContextImpl.java`.
4. In the plugin registration logic (e.g., `registerPlugin()`), check if `getMinimumSupportedBesuVersion()` is present.
5. If the required version is significantly newer than the node's running Besu version, log a clear `WARN` message that the plugin might fail.

**Why it's a good PR:** It demonstrates an understanding of how plugins are registered and sets the foundational interface for the broader API versioning system.

---

## 2. Expose Lifecycle State to Service Manager
**Objective:** Lifecycle Refinement
**Difficulty:** Medium

**Problem:**
The mentorship issue explicitly states: *"This ambiguity can result in critical problems, such as plugins attempting to access a service before it is fully initialized."*
Internally, `BesuPluginContextImpl` manages a strict `Lifecycle` enum (e.g., `REGISTERED`, `BEFORE_EXTERNAL_SERVICES_STARTED`), but plugins using `ServiceManager` have no way to know what phase the node is currently in.

**Proposed Implementation:**
1. Extract a public `BesuPluginLifecycle` enum (or similar) into the `plugin-api` module.
2. Update `plugin-api/src/main/java/org/hyperledger/besu/plugin/ServiceManager.java` to include a method `BesuPluginLifecycle getLifecycleState();`.
3. Implement this method in `app/src/main/java/org/hyperledger/besu/services/BesuPluginContextImpl.java` by exposing its internal `state` field.
4. Add Javadoc explicitly warning developers not to fetch Phase 2 runtime services (like `MetricsSystem`) during the `REGISTERING` phase.

**Why it's a good PR:** It directly touches on the node lifecycle and race conditions, showing the mentors you've read the code related to `BesuPluginContextImpl.java` and understand the initialization order.

---

## 3. Establish Package-Level Design Documentation
**Objective:** Code Reorganization & Documentation
**Difficulty:** Low

**Problem:**
Currently, the `plugin-api/src/main/java/org/hyperledger/besu/plugin/services/` directory is mostly a flat package with numerous unrelated services (`BlockchainService`, `TraceService`, `PicoCLIOptions`). While moving classes is a breaking change (and part of the larger mentorship), setting up the structural documentation is an excellent first step.

**Proposed Implementation:**
1. Create a `package-info.java` file in `plugin-api/src/main/java/org/hyperledger/besu/plugin/services/`.
2. Write comprehensive documentation explaining the design principles of the Plugin API.
3. Categorize the existing services conceptually in the Javadoc (e.g., "Core Blockchain Services", "RPC Extension Services", "Lifecycle Services").
4. Include a paragraph about deprecation and how developers should treat `@Unstable` APIs.

**Why it's a good PR:** The mentors want "renewed API documentation" and "documented clear guidelines on how and where to expose new interfaces". Doing this via `package-info.java` creates living documentation that sits right next to the code.

---

## 4. Add Runtime Deprecation Logging for Plugin Services
**Objective:** API Versioning & Lifecycle Refinement
**Difficulty:** Medium

**Problem:**
There are several methods and interfaces marked with `@Deprecated` or `@Unstable` (like methods in `BesuConfiguration.java` or `TransactionSimulationService.java`), but there is no systemic way to monitor when plugins are relying on them.

**Proposed Implementation:**
1. In `app/src/main/java/org/hyperledger/besu/services/BesuPluginContextImpl.java`, locate the `getService(final Class<T> serviceType)` method.
2. Add a simple reflection check: if the requested `serviceType` has the `@Deprecated` annotation, log a warning:
   `LOG.warn("Plugin requested deprecated service {}. This service will be removed in a future version.", serviceType.getName());`
3. Ensure this check happens only once per plugin/service combo to avoid log spam (e.g., track it in a `Set<Class<?>>`).

**Why it's a good PR:** It directly addresses the "process around deprecation, versioning and backward compatibility" mentioned in the issue. It gives Besu maintainers visibility into which deprecated services are actually still being used by the ecosystem.
