# FINDIFY × WSO2 API Manager

Securing a Spring Boot microservice with an enterprise API gateway — OAuth2 authentication and rate limiting, with **zero changes to the application code**.

---

## The problem

FINDIFY is an AI-integrated Lost and Found portal I built as a university project. It runs as five Spring Boot microservices:

| Service | Port | Responsibility |
|---|---|---|
| user-service | 8080 | Registration, login, user records |
| item-service | 8082 | Lost and found item listings |
| email-service | 8083 | Notifications |
| chat-service | 8084 | Real-time messaging between users |
| image-similarity-service | — | ML-based item matching |

Each service handled its own security. That meant:

- A hand-written JWT filter duplicated in every service
- CORS configured separately per service
- Token validation happening redundantly at every hop
- No rate limiting anywhere
- No central way to revoke a compromised token
- No single place to see what traffic was arriving

Every new service meant copying the same security code again. That is the definition of a cross-cutting concern living in the wrong place.

## The solution

Put **WSO2 API Manager** in front of the services as a gateway. Authentication, throttling, routing and logging move to the gateway; the services go back to doing only their own job.

```
                    ┌─────────────────────────┐
  Client  ───────►  │  WSO2 API Manager       │
                    │  gateway :8243          │
                    │                         │
                    │  · OAuth2 validation    │
                    │  · Rate limiting        │
                    │  · Routing              │
                    │  · Logging              │
                    └───────────┬─────────────┘
                                │
                                ▼
                    ┌─────────────────────────┐
                    │  user-service :8080     │
                    │  (Spring Boot)          │
                    │  no security code added │
                    └─────────────────────────┘
```

## What was built

WSO2 API Manager 4.7.0 running locally, with FINDIFY's user-service exposed through it as a secured, throttled API.

**Endpoint used for the demo:** `GET /api/users/count` — a public endpoint returning `{"count": 2}`.

| | Before | After |
|---|---|---|
| Authentication | Hand-written JWT filter in each service | OAuth2 validated at the gateway |
| Rate limiting | None | 5 requests/minute, enforced before the backend |
| Developer docs | None | Auto-generated Developer Portal |
| Versioning | Manual | Built-in lifecycle |
| Code changes required | — | **Zero** |

---

## Setup

### Prerequisites

- JDK 21 (API Manager 4.7.0 requires it)
- MySQL running
- The FINDIFY user-service on port 8080

> **Check `JAVA_HOME`, not `java -version`.** WSO2 reads `JAVA_HOME`. On my machine `java -version` reported 21 while `JAVA_HOME` still pointed at a JDK 8 install — WSO2 would have failed to start with no obvious cause.
>
> ```
> echo %JAVA_HOME%
> "%JAVA_HOME%\bin\java" -version
> ```
> Both must report 21.

### 1. Start API Manager

Download the all-in-one distribution from [wso2.com/api-manager](https://wso2.com/api-manager/), unzip it, then:

```
cd <wso2am-home>\bin
api-manager.bat
```

Wait for `WSO2 Carbon started`. First startup takes a few minutes.

> The zip may extract as a folder inside a folder (`wso2am-4.7.0.17\wso2am-4.7.0\bin`). Use whichever directory actually contains `bin`.

### 2. Create the API

Publisher Portal: `https://localhost:9443/publisher` (`admin` / `admin`)

Chrome will warn about the certificate — it is self-signed on localhost. Proceed.

**REST API** → fill in:

| Field | Value |
|---|---|
| Name | `FindifyUserAPI` |
| Context | `/findify-users` |
| Version | `1.0.0` |
| Endpoint | `http://localhost:8080` |

The endpoint is the host and port only. WSO2 appends the rest of the path.

![Creating the API](images/02-create-api.png)

### 3. Deploy and publish

These are two separate actions and both are required:

- **Deployments → Deploy** puts the API onto the gateway runtime and creates a revision
- **Lifecycle → Publish** makes it discoverable in the Developer Portal

![Published](images/04-published.png)

### 4. Subscribe and get a token

Developer Portal: `https://localhost:9443/devportal`

**Sign in first.** The Dev Portal displays APIs to anonymous visitors but hides every action, so an unauthenticated page looks broken rather than locked.

1. Subscriptions → `DefaultApplication`
2. **PROD KEYS** → Generate Keys
3. Copy the Consumer Secret immediately — it is shown exactly once
4. Generate Access Token

### 5. Call it through the gateway

```bash
curl -k -H "Authorization: Bearer <TOKEN>" \
  https://localhost:8243/findify-users/1.0.0/api/users/count
```

```json
{"count":2}
```

Same response as calling port 8080 directly, but the request now travels:

`curl → gateway :8243 → OAuth2 validation → user-service :8080 → back`

![Working through the gateway](images/06-gateway-success.png)

---

## Rate limiting

The built-in business plans start at Bronze (1000 requests/minute), which is sensible for production and impossible to test by hand. A custom plan is needed.

**Admin Portal** (`https://localhost:9443/admin`) → Rate Limiting Policies → Subscription Policies → Add Policy:

| Setting | Value |
|---|---|
| Name | `TestFivePerMinute` |
| Request Count | 5 |
| Unit Time | 1 Minute |
| Stop On Quota Reach | enabled |

`Stop On Quota Reach` is what makes excess requests get rejected rather than merely counted.

Then in the Publisher: **Portal Configurations → Subscriptions** → tick the new plan, untick Unlimited, save, and deploy a new revision.

![Business plans](images/07-business-plans.png)

> **The step that is easy to miss:** attaching a plan to an API does **not** move an existing subscriber onto it. My `DefaultApplication` had subscribed while Unlimited was the only option and stayed there, so ten test requests all sailed through.
>
> Check the **Tier** column under *Manage Subscriptions*. To fix: unsubscribe, resubscribe on the new plan, generate a fresh token, redeploy.

Testing the limit:

```bash
for /L %i in (1,1,10) do curl -s -k -H "Authorization: Bearer %TOKEN%" ^
  https://localhost:8243/findify-users/1.0.0/api/users/count
```

Five successes, then:

```json
{
  "code": "900804",
  "message": "Message throttled out",
  "description": "You have exceeded your quota. You can access API after 2026-Sep-26 14:01:00+0000 UTC",
  "nextAccessTime": "2026-Sep-26 14:01:00+0000 UTC"
}
```

The gateway rejects the request before it reaches Spring Boot, and `nextAccessTime` tells a well-behaved client exactly when to retry.

![Throttled](images/08-throttled.png)

---

## Problems hit along the way

| Symptom | Cause | Fix |
|---|---|---|
| WSO2 won't start | `JAVA_HOME` pointed at JDK 8 while `java` resolved to 21 | Set `JAVA_HOME` to the JDK 21 directory, open a fresh terminal |
| `cd ...\bin` — path not found | Zip extracted a nested folder with a slightly different name | Use the directory that actually contains `bin` |
| Dev Portal shows no Subscribe button | Not signed in | Sign in — anonymous visitors see a read-only view |
| `900901 Invalid Credentials` in Try Out | Pasted the OAuth2 token into a field expecting an Internal Key | Use the *Generate Key* button on the Try Out page |
| `303001 ... State : SUSPENDED` | Circuit breaker tripped after failed requests to an unmapped root path | Confirm the backend is healthy, wait for cooldown or redeploy |
| Rate limit never triggers | Subscription still on the old tier | Resubscribe on the new plan, new token, redeploy |

---

## What this replaces

Building the same thing by hand in Spring Boot would have meant choosing a counter store, picking a windowing strategy, writing a filter, handling the rejection response, deciding the throttling key — and then repeating some version of that across five services.

At the gateway it is a form, a checkbox and a redeploy. And because limits are enforced per subscriber, two applications can call the same API under different quotas without the service knowing anything about it.

The Spring Boot code in this project is unchanged throughout.

---

## Write-ups

1. [I Built Microservices the Hard Way and Then I Found WSO2 API Manager](https://medium.com/@okitha.20240578/i-built-microservices-the-hard-way-and-then-i-found-wso2-api-manager-040f9576d464)
2. So I Actually Did It: Putting My Spring Boot Service Behind WSO2 API Manager — PASTE_MEDIUM_LINK_2
3. Rate Limiting in WSO2 API Manager: What the Docs Don't Tell You — PASTE_MEDIUM_LINK_3

**Video walkthrough:** PASTE_YOUTUBE_LINK

## Next

- Put item-service behind the gateway with a different quota
- Try the AI Gateway with the Gemini integration from another project
- Replace the remaining hand-written JWT auth with WSO2 Identity Server

---

**Okitha Dintharu** — Second-year BSc (Hons) Computer Science, IIT / University of Westminster
[LinkedIn](https://www.linkedin.com/in/okitha-dintharu-bb679a334/) · [Medium](https://medium.com/@okitha.20240578)
