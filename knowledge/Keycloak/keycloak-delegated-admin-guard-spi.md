---
title: Keycloak Delegated Admin Guard SPI — Blocking Deletes and Client-Scope Mutations
type: reference
domain: Keycloak
tags:
  - keycloak
  - spi
  - event-listener
  - delegated-admin
  - fgap-v2
  - client-manager
  - client-scopes
  - security
  - keycloak-26
  - quarkus
status: approved
created: 2026-03-02
updated: 2026-03-02
related:
  - keycloak-fgap-v2-delegated-admin.md
  - keycloak-spi.md
---

# Delegated Admin Guard SPI

## Problem Statement

FGAP v2 leaves two security gaps for a `client-manager` delegated admin role:

1. **CLIENT DELETE** — The `manage` scope in FGAP v2 implies both update AND delete. There is no separate scope to allow update-but-not-delete. A `client-manager` can therefore delete any client they can manage.

2. **CLIENT_SCOPE mutations** — FGAP v2 has no `ClientScopes` resource type. A user with global `manage` on `Clients` gets full, ungated CRUD access to all client scopes — including system scopes like `acr`, `profile`, `email`, `roles`. This was **empirically verified**: a `client-manager` user could delete the `acr` scope with a `204` response.

## Solution Architecture

```
custom-delegated-admin-guard-spi/
├── pom.xml
└── src/main/
    ├── java/org/aiims/keycloak/guard/
    │   ├── DelegatedAdminGuardEventListener.java   ← Core logic
    │   └── DelegatedAdminGuardEventListenerFactory.java  ← SPI factory
    └── resources/META-INF/services/
        └── org.keycloak.events.EventListenerProviderFactory  ← SPI registration
```

**Provider ID**: `delegated-admin-guard`

**Registration**: Added to realm `eventsListeners` list via bootstrap script (`step6_fgap_api_setup.py`).

## Critical Discovery: AdminEventBuilder Swallows ALL Exceptions

> 🚨 **This is the most important finding from building this SPI.**

Looking at `AdminEventBuilder.java` (in `services/src/main/java/org/keycloak/services/resources/admin/`):

```java
// Line 305-313 in AdminEventBuilder.java
if (listeners != null) {
    for (EventListenerProvider l : listeners.values()) {
        try {
            l.onEvent(eventCopy, includeRepresentation);
        } catch (Throwable t) {
            // ← ALL exceptions including ForbiddenException are swallowed here!
            ServicesLogger.LOGGER.failedToSendType(t, l);
        }
    }
}
```

**Consequence**: Throwing `ForbiddenException` (or any exception) inside `onAdminEvent()` does NOT block the operation. The exception is caught, logged at WARN level, and ignored. The HTTP response has **already been written** as `201`/`204` before `send()` is called.

This contradicts the Javadoc for `EventListenerProvider` which says:

> *"Note this method should not do any action that cannot be rolled back"*

The Javadoc warning only applies to the `onEvent(Event event)` method (user events via `EventBuilder`). `EventBuilder` does NOT catch exceptions — but `AdminEventBuilder` does.

### Contrast: EventBuilder vs AdminEventBuilder

| | `EventBuilder` (user events) | `AdminEventBuilder` (admin events) |
|---|---|---|
| Exception handling | **NOT caught** — propagates | **Caught** — swallowed silently |
| Can block operation by throwing | ✅ Yes | ❌ No |
| Source file | `server-spi-private/.../EventBuilder.java` | `services/.../admin/AdminEventBuilder.java` |

## Workaround: setRollbackOnly()

Since exceptions are swallowed, we use `session.getTransactionManager().setRollbackOnly()` to mark the database transaction for rollback:

```java
@Override
public void onEvent(AdminEvent event, boolean includeRepresentation) {
    // ... resolve actor, check client-manager role ...

    if (resourceType == ResourceType.CLIENT && opType == OperationType.DELETE) {
        LOG.warnf("DELEGATED_ADMIN_GUARD: Blocking CLIENT DELETE — user=%s", actor.getUsername());
        // Mark transaction rollback — DB changes will NOT persist
        session.getTransactionManager().setRollbackOnly();
        return;
    }

    if (resourceType == ResourceType.CLIENT_SCOPE) {
        if (opType == OperationType.CREATE || opType == OperationType.UPDATE || opType == OperationType.DELETE) {
            LOG.warnf("DELEGATED_ADMIN_GUARD: Blocking CLIENT_SCOPE %s — user=%s", opType, actor.getUsername());
            session.getTransactionManager().setRollbackOnly();
        }
    }
}
```

### What setRollbackOnly() achieves vs. doesn't achieve

| Aspect | Result |
|---|---|
| Database write (data persistence) | ✅ **BLOCKED** — transaction is rolled back |
| HTTP response code seen by client | ❌ **NOT changed** — client sees `204`/`201` (already written) |
| Security outcome | ✅ **Data protected** — operation never persisted |
| User experience | ⚠️ Misleading — client UI shows success but data isn't saved |

So when the guard fires:
- `DELETE /clients/{id}` → client gets `204` but client is **not actually deleted**
- `POST /client-scopes` → client gets `201` but scope is **not actually created**

## Proper 403 Response — 3 Options

For a genuine `403` HTTP response, one of the following approaches is required:

### Option A: Nginx Pre-Validation (Quickest)

Add an nginx `auth_request` sub-request that validates the bearer token role before forwarding to Keycloak:

```nginx
location ~ ^/admin/realms/[^/]+/clients/[^/]+$ {
    # Block DELETE for non-realm-admin tokens
    if ($request_method = DELETE) {
        # auth_request to a validation microservice
        auth_request /validate-admin-delete;
    }
    proxy_pass http://keycloak:8080;
}
```

### Option B: Quarkus Extension (Keycloak-Native)

Build a Quarkus extension JAR (not a plain provider JAR) that registers a JAX-RS `ContainerRequestFilter` as a CDI bean. This runs BEFORE the request is processed. The extension needs:

- `quarkus-extension.yaml`
- A CDI producer annotated with `@ApplicationScoped`
- The filter class with `@Provider @Priority(1)`

> ⚠️ **Why a plain `@Provider` in a providers/ JAR doesn't work**: Keycloak Quarkus does not auto-discover JAX-RS providers placed in the `providers/` directory. They must be registered via Keycloak's SPI infrastructure or a Quarkus extension to be picked up by RESTEasy.

### Option C: API Gateway / Proxy (Production)

Place a lightweight API gateway (Kong, Traefik with middleware, or a small Golang/Node proxy) in front of Keycloak. The gateway validates the JWT `realm_access.roles` claim and blocks the offending paths/methods.

## Role-Check Logic

The guard applies only to users who:
- **HAVE** the `client-manager` realm role
- **DO NOT HAVE** `realm-admin` realm role
- **DO NOT HAVE** `manage-realm` role from `realm-management` client

```java
private boolean hasClientManagerRoleOnly(RealmModel realm, UserModel user) {
    RoleModel clientManagerRole = realm.getRole("client-manager");
    if (clientManagerRole == null || !user.hasRole(clientManagerRole)) {
        return false;
    }
    // Don't restrict real realm admins
    RoleModel realmAdminRole = realm.getRole("realm-admin");
    if (realmAdminRole != null && user.hasRole(realmAdminRole)) {
        return false;
    }
    // Don't restrict manage-realm holders
    var realmMgmt = realm.getClientByClientId("realm-management");
    if (realmMgmt != null) {
        RoleModel manageRealmRole = realmMgmt.getRole("manage-realm");
        if (manageRealmRole != null && user.hasRole(manageRealmRole)) {
            return false;
        }
    }
    return true;
}
```

## Deployment

### Build

```bash
cd custom-delegated-admin-guard-spi
mvn package -DskipTests
# → target/custom-delegated-admin-guard-spi-1.0.0.jar
```

### Deploy to running container (dev iteration)

```bash
# 1. Copy JAR
docker cp target/custom-delegated-admin-guard-spi-1.0.0.jar \
  aiims-keycloak:/opt/keycloak/providers/

# 2. Rebuild Keycloak (registers the new provider)
docker exec aiims-keycloak /opt/keycloak/bin/kc.sh build --health-enabled=true

# 3. Restart
docker restart aiims-keycloak
```

### Register on realm (via bootstrap script)

```python
def register_event_listener(keycloak_url, token, realm, listener_id):
    """Idempotently add an event listener to the realm."""
    url = f"{keycloak_url}/admin/realms/{realm}"
    realm_rep = requests.get(url, headers=headers(token), verify=False).json()
    current_listeners = realm_rep.get('eventsListeners', [])
    if listener_id in current_listeners:
        return  # already registered
    realm_rep['eventsListeners'] = current_listeners + [listener_id]
    requests.put(url, headers=headers(token), json=realm_rep, verify=False).raise_for_status()

# Called in step6_fgap_api_setup.py:
register_event_listener(KC_URL, token, REALM, 'delegated-admin-guard')
```

### Verify registration

```bash
docker exec aiims-keycloak /opt/keycloak/bin/kcadm.sh config credentials \
  --server http://localhost:8080 --realm aiims-new-delhi \
  --user realmadmin1 --password StrongPass@123 --config /tmp/kcadm.config

docker exec aiims-keycloak /opt/keycloak/bin/kcadm.sh get events/config \
  -r aiims-new-delhi --config /tmp/kcadm.config
# → "eventsListeners" : [ "delegated-admin-guard", "jboss-logging" ]
```

## Testing

### Test 07 — Client Scope mutations blocked

```python
def test_07_client_scope_mutations_blocked(self):
    # CREATE → 403
    res = self.session.post(
        f"{url}/admin/realms/{realm}/client-scopes",
        json={"name": "probe-scope", "protocol": "openid-connect"},
        headers=test_headers
    )
    assert res.status_code == 403  # ← blocked

    # UPDATE existing scope → 403
    scope = admin_scopes[0]
    res = self.session.put(
        f"{url}/admin/realms/{realm}/client-scopes/{scope['id']}",
        json={**scope, "description": "tampered"},
        headers=test_headers
    )
    assert res.status_code == 403

    # DELETE existing scope → 403
    res = self.session.delete(
        f"{url}/admin/realms/{realm}/client-scopes/{scope['id']}",
        headers=test_headers
    )
    assert res.status_code == 403
```

> **Note**: As of now, these tests FAIL because `setRollbackOnly()` does not change the HTTP response code. The data is protected but the response is `201`/`204`. A proper 403 requires Option A/B/C above.

### Test 03 — DELETE own client blocked

```python
def test_03_own_client_deletion_blocked(self):
    res = self.session.delete(
        f"{url}/admin/realms/{realm}/clients/{created_client_id}",
        headers=test_headers
    )
    assert res.status_code == 403  # ← blocked by guard
```

> **Note**: Same caveat — currently returns `204` (operation appears successful to the HTTP client) but data is rolled back.

## SPI File Structure Reference

```
custom-delegated-admin-guard-spi/
├── pom.xml  (keycloak.version=26.5.3, java=21)
└── src/
    ├── main/
    │   ├── java/org/aiims/keycloak/guard/
    │   │   ├── DelegatedAdminGuardEventListener.java
    │   │   │   ├── onEvent(Event) → no-op
    │   │   │   ├── onEvent(AdminEvent) → check CLIENT DELETE + CLIENT_SCOPE mutations
    │   │   │   └── hasClientManagerRoleOnly() → role check logic
    │   │   ├── DelegatedAdminGuardEventListenerFactory.java
    │   │   │   └── getId() → "delegated-admin-guard"
    │   │   └── DelegatedAdminGuardFilter.java   ← written but not yet wired
    │   │       └── filter() → intended JAX-RS ContainerRequestFilter
    └── resources/META-INF/services/
        └── org.keycloak.events.EventListenerProviderFactory
            └── "org.aiims.keycloak.guard.DelegatedAdminGuardEventListenerFactory"
```

## Key Source Files in Keycloak 26.5.4

| File | Relevance |
|---|---|
| `services/src/main/java/org/keycloak/services/resources/admin/AdminEventBuilder.java` | **Critical**: Line 307-311 shows the catch(Throwable) that swallows listener exceptions |
| `server-spi-private/src/main/java/org/keycloak/events/EventBuilder.java` | User-event builder — does NOT catch exceptions (contrast) |
| `server-spi-private/src/main/java/org/keycloak/events/EventListenerProvider.java` | SPI interface; Javadoc misleadingly implies exceptions propagate for ALL event types |
| `services/src/main/java/org/keycloak/services/filters/InvalidQueryParameterFilter.java` | Example of built-in JAX-RS filter — uses `@Provider @PreMatching` but is part of Keycloak core |

## Known Limitations and Next Steps

| Limitation | Impact | Next Step |
|---|---|---|
| HTTP response is `204`/`201` not `403` when guard fires | Could mislead API clients that data was saved | Implement Option A (nginx) or B (Quarkus extension) |
| `DelegatedAdminGuardFilter.java` not yet wired | JAX-RS filter exists but isn't auto-discovered from providers/ JAR | Build Quarkus extension wrapper |
| No unit tests for the SPI itself | Guard logic is only tested end-to-end | Add JUnit tests with Mockito for `hasClientManagerRoleOnly()` |
| `client-scopes` still appear in UI sidebar | Visual confusion (they appear accessible but writes are rolled back) | Accepted limitation for now |
