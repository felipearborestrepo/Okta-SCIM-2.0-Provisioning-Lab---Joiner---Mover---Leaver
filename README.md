# Okta SCIM 2.0 Provisioning Lab: Joiner, Mover, Leaver

A hands-on lab that uses **Okta Lifecycle Management** to automatically provision users to an application over **SCIM 2.0**. I built a small SCIM 2.0 server in Python, exposed it to Okta through an ngrok HTTPS tunnel, and tested the full user lifecycle: creating a user (joiner), updating their profile (mover), deactivating them (leaver), and pushing a group.

Because I control the app, I could see the **actual SCIM requests Okta sends** for each lifecycle event, not just the result in the Okta console.

## Project Goal

- Connect Okta to a SCIM 2.0 application using a bearer token
- Automatically **create** a user in the app when they're assigned in Okta
- Automatically **update** the app when the user's Okta profile changes
- Automatically **deactivate** the user in the app when access is removed
- **Push** an Okta group to the app
- Capture the SCIM requests behind each event as evidence

## Architecture

```
Okta (Lifecycle Management)
        │  SCIM 2.0 over HTTPS
        │  Authorization: Bearer <token>
        ▼
ngrok tunnel (public HTTPS URL)
        │
        ▼
scim_server.py on my Mac (localhost:5000)
   ├── /scim/v2/Users    (create, read, update, deactivate)
   ├── /scim/v2/Groups   (group push)
   └── /                 (dashboard showing users and groups)
```

| Component | Role |
|---|---|
| Okta (Integrator Free Plan) | Identity source; sends SCIM provisioning requests |
| `scim_server.py` (Python/Flask) | The target application; implements the SCIM 2.0 Users and Groups endpoints |
| ngrok | Gives the local server a public HTTPS URL that Okta's cloud can reach |
| Bearer token | Authenticates Okta to the SCIM app |

## Key Concepts

| Term | Meaning |
|---|---|
| **SCIM 2.0** | An open standard (RFC 7643/7644) for automatically creating, updating, and deactivating user accounts across systems using a REST API |
| **Provisioning** | Creating or updating a user's account in an app automatically |
| **Deprovisioning** | Removing or deactivating a user's access automatically |
| **Joiner / Mover / Leaver** | The three lifecycle events: a new hire, a role change, and a departure |
| **Bearer token** | A secret sent in the `Authorization` header that proves the caller is allowed to use the API |

---

## Step 1: Start the SCIM Server

I started the SCIM app on my Mac with a fixed bearer token, so the token stays the same across restarts:

```bash
SCIM_TOKEN=<your-token> python3 scim_server.py
```

The server listens on port 5000 and exposes the SCIM API at `/scim/v2` and a dashboard at `/`.

<img width="789" height="188" alt="Screenshot 2026-10-09 at 18 11 30" src="https://github.com/user-attachments/assets/76ec5f1e-6d7b-4b35-a864-5b42a3a83286" />

## Step 2: Expose the Server with ngrok

Okta runs in the cloud and can't reach `localhost`, so I used **ngrok** to create a public HTTPS tunnel to port 5000. I authenticated the ngrok agent with my account's authtoken, then started the tunnel:

```bash
ngrok config add-authtoken <ngrok-authtoken>
ngrok http 5000
```

<img width="1407" height="471" alt="Screenshot 2026-10-09 at 18 44 23" src="https://github.com/user-attachments/assets/0c2601a3-bd30-42d8-b1fa-b0f31cd665d6" />

ngrok returned a public **Forwarding** URL. The SCIM base URL for Okta is that URL plus `/scim/v2`.

<img width="558" height="262" alt="Screenshot 2026-10-09 at 18 13 05" src="https://github.com/user-attachments/assets/de3a13fa-87f3-4b79-b1ca-ed5ba14b4f2f" />

## Step 3: Seed a Test User

Okta's connector test checks that the app can return users when importing is enabled. Since the app started empty, I created one seed user directly through the SCIM API with `curl`. The server responded with the new user's SCIM record, including the `id` it generated and its `meta` data.

```bash
curl -s -X POST http://localhost:5000/scim/v2/Users \
  -H "Authorization: Bearer <your-token>" \
  -H "Content-Type: application/json" \
  -d '{"userName":"seed.user@scimlab.test","name":{"givenName":"Seed","familyName":"User"},"active":true}'
```
<img width="787" height="300" alt="Screenshot 2026-10-09 at 18 14 18" src="https://github.com/user-attachments/assets/c0025bea-fde8-4c9a-83a3-e98691077746" />

This also confirmed the API and token worked locally before involving Okta.

## Step 4: Create the App Integration in Okta

Okta Admin Console → **Applications** → **Create App Integration** → **SAML 2.0**. A custom app integration is needed to attach SCIM provisioning to it.

<img width="965" height="549" alt="Screenshot 2026-10-09 at 18 19 02" src="https://github.com/user-attachments/assets/6730dfbc-9f9d-40d7-8d76-52f1191b9568" />

I named the app **SCIM Lab App** and entered the ngrok URL as the **Single sign-on URL** and **Audience URI**. SSO isn't tested in this lab; Okta requires these values to create the app.

<img width="899" height="613" alt="Screenshot 2026-10-09 at 18 20 40" src="https://github.com/user-attachments/assets/7cbe33c3-fbaa-4645-b9d1-e8a11b9b8407" />

## Step 5: Enable SCIM Provisioning

On the app's **General** tab → **App Settings** → **Edit**, I set **Provisioning** to **SCIM**. This adds the **Provisioning** tab.

<img width="751" height="555" alt="Screenshot 2026-10-09 at 18 21 53" src="https://github.com/user-attachments/assets/254e97c7-6cd6-4f42-88e0-9e4a84da85f0" />

## Step 6: Connect Okta to the SCIM Server

**Provisioning** → **Integration** → **Edit**:

| Field | Value |
|---|---|
| SCIM connector base URL | `https://<ngrok-url>/scim/v2` |
| Unique identifier field for users | `userName` |
| Supported provisioning actions | Import New Users and Profile Updates, Push New Users, Push Profile Updates, Push Groups, Import Groups |
| Authentication Mode | HTTP Header |
| Authorization | `Bearer <your-token>` |

<img width="795" height="592" alt="Screenshot 2026-10-09 at 18 24 02" src="https://github.com/user-attachments/assets/fda34a33-04b9-4284-b7a1-678a5074c667" />

I clicked **Test Connector Configuration**. Okta successfully authenticated and detected the supported features: User Import, Import Profile Updates, Create Users, Update User Attributes, Push Groups, and Import Groups.

<img width="557" height="591" alt="Screenshot 2026-10-09 at 18 24 54" src="https://github.com/user-attachments/assets/3f91ef64-f9ed-418d-b4ad-7653568fab73" />

## Step 7: Enable Lifecycle Actions

**Provisioning** → **To App** → **Edit**, I enabled:

- **Create Users**: creates the user in the app when the app is assigned in Okta
- **Update User Attributes**: pushes Okta profile changes to the app
- **Deactivate Users**: deactivates the app account when the user is unassigned or deactivated in Okta

<img width="1042" height="594" alt="Screenshot 2026-10-09 at 18 26 13" src="https://github.com/user-attachments/assets/8869ec9d-995d-49da-83c5-9826edaf62be" />

---

## Step 8: Joiner Test (Create User)

I assigned the app to a test user, **Alex Eng** (`alex.eng@yourlabdomain.com`), on the **Assignments** tab.

<img width="1127" height="410" alt="Screenshot 2026-10-09 at 18 34 07" src="https://github.com/user-attachments/assets/77632384-8659-4d58-9f4f-802870ae28ad" />

Okta immediately sent a `POST /scim/v2/Users` request containing the user's profile: `userName`, name, email, display name, locale, department (`Engineering`, via the SCIM enterprise extension), and `externalId` (the user's Okta ID). The server responded **201 Created**.

<img width="788" height="115" alt="Screenshot 2026-10-09 at 18 43 41" src="https://github.com/user-attachments/assets/1c9f7abe-2969-4fce-bb28-8dbea0703917" />

Alex Eng appeared on the app's dashboard as **Active**.

<img width="1106" height="500" alt="Screenshot 2026-10-09 at 18 35 38" src="https://github.com/user-attachments/assets/6053654b-d680-4321-9462-d9d4ae2fd781" />

**What happened:** Assigning the app in Okta triggered automatic account creation in the target app, with no manual work in the app itself.

## Step 9: Mover Test (Update User)

I changed Alex Eng's **Title** in Okta to `Engineer`.

<img width="671" height="317" alt="Screenshot 2026-10-09 at 18 47 06" src="https://github.com/user-attachments/assets/dad6727e-d098-491a-9a11-b76208d3c35a" />

Okta sent a `PUT /scim/v2/Users/{id}` request with the full updated user record, including `"title": "Engineer"`. The server responded **200 OK**.

<img width="779" height="65" alt="Screenshot 2026-10-09 at 18 47 37" src="https://github.com/user-attachments/assets/e1c0bb51-b9be-48c6-bcbc-797d2029937b" />

The dashboard showed the new title.

<img width="1325" height="309" alt="Screenshot 2026-10-09 at 18 48 02" src="https://github.com/user-attachments/assets/6bc91ba3-f513-4387-a316-83cef741297b" />

**What happened:** A profile change in Okta flowed to the app automatically, keeping the two systems in sync. Okta uses `PUT`, which replaces the whole user record with the current Okta values.

## Step 10: Leaver Test (Deactivate User)

I removed Alex Eng's assignment to the app by clicking the **X** on the **Assignments** tab.

<img width="803" height="490" alt="Screenshot 2026-10-09 at 18 48 53" src="https://github.com/user-attachments/assets/d23e7488-9d66-4e10-bc68-28c851ca5bc3" />

Okta first sent a `GET` for the user's current record, then a `PUT /scim/v2/Users/{id}` with **`"active": false`**. The server responded **200 OK**.

<img width="794" height="166" alt="Screenshot 2026-10-09 at 18 49 27" src="https://github.com/user-attachments/assets/c74fe770-0a96-4ded-9c63-5b0f4b25065f" />

The dashboard showed Alex Eng as **Deactivated**.

<img width="1400" height="314" alt="Screenshot 2026-10-09 at 18 49 57" src="https://github.com/user-attachments/assets/16720989-a962-4d07-8123-e9eaa8e740cc" />

**What happened:** Removing access in Okta automatically disabled the user's account in the app. Okta **deactivates** instead of deleting, which keeps the account and its history for auditing and allows reactivation if the user is reassigned. This prevents orphaned accounts when someone leaves.

## Step 11: Group Push

On the **Push Groups** tab, I pushed the Okta group **Dept-Engineering** to the app. Okta shows the push status as **Active**.

<img width="1038" height="556" alt="Screenshot 2026-10-09 at 18 54 04" src="https://github.com/user-attachments/assets/f97a849e-cdb6-4b81-be36-c053ce0d7d97" />

Okta sent a `POST /scim/v2/Groups` to create the group (**201 Created**), then a `GET` and a `PUT` to sync its membership.

<img width="777" height="128" alt="Screenshot 2026-10-09 at 18 54 41" src="https://github.com/user-attachments/assets/b2543062-c98a-446e-999d-36912831d86f" />

The group appeared in the app's dashboard.

<img width="1364" height="434" alt="Screenshot 2026-10-09 at 18 55 18" src="https://github.com/user-attachments/assets/e8ff4689-4296-42a8-8749-1d4f713d018a" />

**Note:** The group was pushed with **0 members**. Group push only syncs members who are also assigned to the app, and Alex Eng had already been unassigned in the leaver test. To show membership sync, assign the app to the group's members (or to the group itself) and push again.

---

## Results Summary

| Lifecycle event | Okta action | SCIM request | Result in app |
|---|---|---|---|
| Connector test | Test Connector Configuration | `GET /Users` | Connection verified |
| **Joiner** | Assign app to Alex Eng | `POST /Users` → 201 | User created, Active |
| **Mover** | Change Title to Engineer | `PUT /Users/{id}` → 200 | Title updated |
| **Leaver** | Unassign Alex Eng | `GET` then `PUT /Users/{id}` with `active:false` → 200 | User Deactivated |
| **Group push** | Push Dept-Engineering | `POST /Groups` → 201, then `PUT` | Group created |

## Troubleshooting

Getting the connection working took several fixes. Each one is a common real-world SCIM issue.

| Symptom | Root cause | Fix |
|---|---|---|
| Connector test: **"Error authenticating: Forbidden"** | The SCIM server wasn't running, and on macOS the **AirPlay Receiver** listens on port 5000 and answers with 403 | Turned off AirPlay Receiver (System Settings → General → AirDrop & Handoff) and started the server |
| Server log showed **404** on `GET /Users` | The SCIM base URL in Okta was missing `/scim/v2` | Set the base URL to `https://<ngrok-url>/scim/v2` |
| Server log showed **401 Unauthorized** | The bearer token in the request didn't match the server's token (a randomly generated token was misread) | Restarted the server with a fixed token set by `SCIM_TOKEN` and used the same value in Okta |
| Connector test: **"No results for users returned"** | The app had no users, and Okta's test expects at least one when importing is enabled | Seeded one user through the SCIM API (Step 3) |
| Server: **"Address already in use"** | An earlier server instance was still running on port 5000 | Stopped the old process before starting a new one |

**Lesson:** Reading the server's request log made each failure easy to diagnose: no request at all meant a network or tunnel problem, a 404 meant the wrong URL path, and a 401 meant a token mismatch.
