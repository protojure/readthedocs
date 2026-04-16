## Migrating to Protojure 3.x (Pedestal 0.8.x)

Protojure 3.0.0 upgrades the Pedestal dependency from 0.7.x to 0.8.x.  Pedestal 0.8
replaces the service-map / chain-provider extension model with a new connector-based API.
This is a **breaking change**: the two protojure-specific Pedestal extension functions
(`provider` and `config`) have been removed, and the server lifecycle functions have
changed.

This guide walks through every change you need to make to adopt protojure 3.x.

---

### 1. Dependency Updates

Update your `project.clj` or `deps.edn` to reference protojure 3.x and Pedestal 0.8.1:

**project.clj (Leiningen)**

```clojure
;; Before (protojure 2.x)
[io.github.protojure/grpc-server  "2.x.x"]
[io.pedestal/pedestal.service      "0.7.2"]
[io.pedestal/pedestal.log          "0.7.2"]
[io.pedestal/pedestal.error        "0.7.2"]

;; After (protojure 3.x)
[io.github.protojure/grpc-server  "3.0.0"]
[io.pedestal/pedestal.service      "0.8.1"]
[io.pedestal/pedestal.log          "0.8.1"]
[io.pedestal/pedestal.error        "0.8.1"]
[cheshire "5.13.0"]                           ;; now required explicitly for JSON body handling
```

**deps.edn (Clojure CLI)**

```clojure
;; Before
io.github.protojure/grpc-server  {:mvn/version "2.x.x"}
io.pedestal/pedestal.service     {:mvn/version "0.7.2"}
io.pedestal/pedestal.log         {:mvn/version "0.7.2"}
io.pedestal/pedestal.error       {:mvn/version "0.7.2"}

;; After
io.github.protojure/grpc-server  {:mvn/version "3.0.0"}
io.pedestal/pedestal.service     {:mvn/version "0.8.1"}
io.pedestal/pedestal.log         {:mvn/version "0.8.1"}
io.pedestal/pedestal.error       {:mvn/version "0.8.1"}
cheshire/cheshire                {:mvn/version "5.13.0"}
```

---

### 2. Namespace Changes

Several Pedestal namespaces were renamed or removed in 0.8.x.

| Old namespace (Pedestal 0.7.x) | New namespace (Pedestal 0.8.x) |
|---|---|
| `io.pedestal.http` | `io.pedestal.connector` |
| `io.pedestal.interceptor.helpers` | **Removed** — use `io.pedestal.interceptor/interceptor` directly |
| `io.pedestal.test` | `io.pedestal.connector.test` |

Common interceptors have moved:

| Old var | New var |
|---|---|
| `io.pedestal.http/html-body` | `io.pedestal.service.interceptors/html-body` |
| `io.pedestal.http/json-body` | `io.pedestal.service.interceptors/json-body` |
| `io.pedestal.http/not-found` _(implicit)_ | `io.pedestal.service.interceptors/not-found` _(must be explicit)_ |

---

### 3. Removed protojure Functions

The following functions have been **removed** from `protojure.pedestal.core`:

- `provider` — previously used as the `::http/chain-provider` value in the service map
- `config` — previously used as the `::http/type` value in the service map

They are replaced by a single function: `protojure.pedestal.core/create-connector`.

---

### 4. Server Setup Migration

This is the most significant change.  The old Pedestal 0.7.x "service map" pattern is
completely replaced by the new "connector map" pattern.

**Before (Pedestal 0.7.x)**

```clojure
(ns demo-server.service
  (:require [io.pedestal.http :as http]
            [protojure.pedestal.core :as protojure.pedestal]
            [protojure.pedestal.routes :as proutes]
            [com.example.addressbook.Greeter.server :as greeter]))

(def common-interceptors [http/html-body])

(def routes
  (into #{}
    (proutes/->tablesyntax {:rpc-metadata     greeter/rpc-metadata
                            :interceptors     common-interceptors
                            :callback-context (Greeter.)})))

(def service-map
  {:env                  :prod
   ::http/routes         routes
   ::http/port           8080
   ::http/type           protojure.pedestal/config     ;; <-- removed
   ::http/chain-provider protojure.pedestal/provider   ;; <-- removed
   ::http/container-options {:ssl-port    8443
                             :keystore    "keystore.jks"
                             :key-password "changeit"}})

(defn start-server []
  (-> service-map
      http/create-server
      http/start))

(defn stop-server [server]
  (http/stop server))
```

**After (Pedestal 0.8.x)**

```clojure
(ns demo-server.service
  (:require [io.pedestal.connector :as conn]
            [io.pedestal.service.interceptors :as svc.interceptors]
            [protojure.pedestal.core :as protojure.pedestal]
            [protojure.pedestal.routes :as proutes]
            [com.example.addressbook.Greeter.server :as greeter]))

(def common-interceptors [svc.interceptors/html-body])

(def routes
  (into #{}
    (proutes/->tablesyntax {:rpc-metadata     greeter/rpc-metadata
                            :interceptors     common-interceptors
                            :callback-context (Greeter.)})))

(defn start-server [port]
  (let [connector-map (-> (conn/default-connector-map port)
                          ;; previously-implicit interceptors must now be explicit
                          (conn/with-interceptors [svc.interceptors/not-found])
                          (conn/with-routes routes))]
    (conn/start! (protojure.pedestal/create-connector connector-map))))

(defn stop-server [connector]
  (conn/stop! connector))
```

**With SSL**

```clojure
(defn start-server-ssl [port ssl-port]
  (let [connector-map (-> (conn/default-connector-map port)
                          (conn/with-interceptors [svc.interceptors/not-found])
                          (conn/with-routes routes)
                          (assoc :container-options {:ssl-port    ssl-port
                                                     :keystore    "keystore.jks"
                                                     :key-password "changeit"}))]
    (conn/start! (protojure.pedestal/create-connector connector-map))))
```

---

### 5. Interceptors Previously Added Automatically

Pedestal 0.7.x automatically included several interceptors when calling `http/create-server`.
In Pedestal 0.8.x you must add them explicitly via `conn/with-interceptors`:

| Interceptor | Notes |
|---|---|
| `svc.interceptors/not-found` | Returns 404 for unmatched routes — add this first |
| `route/query-params` | Parses query string parameters — add if your routes use query params |

Example:

```clojure
(conn/with-interceptors [svc.interceptors/not-found
                         route/query-params])
```

---

### 6. Server Lifecycle Changes

| Old (Pedestal 0.7.x) | New (Pedestal 0.8.x) |
|---|---|
| `(http/create-server service-map)` | `(protojure.pedestal/create-connector connector-map)` |
| `(http/start server)` | `(conn/start! connector)` |
| `(http/stop server)` | `(conn/stop! connector)` |

`create-connector` returns an `UndertowConnector` record that satisfies the
`io.pedestal.service.protocols/PedestalConnector` protocol.  It holds all server state
internally, so the same value is threaded through start and stop.

---

### 7. Custom Thread Pool

If you configure a custom thread pool, use the `::protojure.pedestal.core/thread-pool` key
on the connector map instead of the old `::http/container-options` approach:

```clojure
(require '[promesa.exec :as p.exec]
         '[protojure.pedestal.core :as protojure.pedestal])

(let [thread-pool (p.exec/fixed-executor {:parallelism 64})
      connector-map (-> (conn/default-connector-map port)
                        (assoc ::protojure.pedestal/thread-pool thread-pool)
                        (conn/with-interceptors [...])
                        (conn/with-routes routes))]
  (conn/start! (protojure.pedestal/create-connector connector-map)))
```

---

### 8. Test Infrastructure Changes

If you use Pedestal's in-process test utilities, the API has changed.

**Before**

```clojure
(require '[io.pedestal.test :refer [response-for]]
         '[io.pedestal.http :as http])

;; Build a servlet-backed service function
(def service-fn
  (::http/service-fn (http/create-servlet service-map)))

;; Execute a test request
(response-for service-fn :get "/health"
              :headers {"Content-Type" "application/json"}
              :body "{}")
```

**After**

```clojure
(require '[io.pedestal.connector.test :as conn.test]
         '[protojure.pedestal.core :as protojure.pedestal])

;; Build a connector with port 0 — no real server started for tests
(def connector
  (protojure.pedestal/create-connector
    (-> (conn/default-connector-map 0)
        (conn/with-interceptors [...])
        (conn/with-routes routes))))

;; Execute a test request — first arg is connector, not service-fn
(conn.test/response-for connector :get "/health"
                         :headers {"content-type" "application/json"}  ;; note: lowercase keys
                         :body "{}")
```

Key differences:
- First argument is a `connector` (not a service function)
- Use `conn.test/response-for` instead of `io.pedestal.test/response-for`
- Header keys should be lowercase (`"content-type"` not `"Content-Type"`)

---

### 9. Creating Interceptors Manually

If your code used `io.pedestal.interceptor.helpers/on-response` (or other helpers from
that removed namespace), replace them with `io.pedestal.interceptor/interceptor`:

```clojure
;; Before
(require '[io.pedestal.interceptor.helpers :as helpers])
(helpers/on-response ::my-interceptor (fn [response] ...))

;; After
(require '[io.pedestal.interceptor :as interceptor])
(interceptor/interceptor
  {:name  ::my-interceptor
   :leave (fn [context]
            ;; operate on (:response context)
            context)})
```

Pedestal 0.8.x also emits a warning for unnamed interceptors.  Ensure any interceptors
you create or wrap have a `:name` key set.

---

### Summary Checklist

- [ ] Bump `io.github.protojure/grpc-server` to `3.0.0`
- [ ] Bump all `io.pedestal/*` deps to `0.8.1`
- [ ] Add `cheshire "5.13.0"` as an explicit dependency
- [ ] Replace `[io.pedestal.http :as http]` with `[io.pedestal.connector :as conn]`
- [ ] Replace `http/html-body` / `http/json-body` with `svc.interceptors/` equivalents
- [ ] Remove `::http/type` and `::http/chain-provider` keys from your service map
- [ ] Replace service-map pattern with `conn/default-connector-map` + `conn/with-routes`
- [ ] Add `svc.interceptors/not-found` explicitly to `conn/with-interceptors`
- [ ] Replace `http/create-server` + `http/start` with `create-connector` + `conn/start!`
- [ ] Replace `http/stop` with `conn/stop!`
- [ ] Replace `io.pedestal.test/response-for` with `conn.test/response-for`
- [ ] Ensure all interceptors have a `:name` key (Pedestal 0.8.x warns on unnamed interceptors)
