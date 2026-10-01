# 02 – Keycloak: admin account, realm and MFA

*Date: 30 September – 1 October 2026*

In this part I turn the freshly installed Keycloak into a usable identity provider: a permanent administrator account, a dedicated realm for my lab, a regular user, and mandatory multi-factor authentication (MFA).

---

## 1. Replacing the bootstrap admin

On first start, Keycloak creates a **temporary admin account** from the environment variables `KC_BOOTSTRAP_ADMIN_USERNAME` and `KC_BOOTSTRAP_ADMIN_PASSWORD`. Keycloak itself flags this account as temporary: it is meant only to get in, not to keep using.

**Steps (in the `master` realm):**
1. Created a new user `kcadmin`.
2. Set a strong password with *Temporary* switched off.
3. Assigned the realm role **admin** via *Role mapping → Assign role → Realm roles*.
4. Signed out, signed in as `kcadmin` to verify it works.
5. Only then deleted the bootstrap user `admin`.

**Why this order:** delete the bootstrap account before confirming the new one works, and you lock yourself out.

**Why at all:** the bootstrap password lives in the deployment configuration. A permanent, personal admin account separates "how the system was installed" from "who administers it".

> Note: the bootstrap account is only created on the very first start of an empty database. The `KC_BOOTSTRAP_ADMIN_*` variables can be removed from the compose file afterwards.

---

## 2. A dedicated realm: `homelab`

Created a new realm `homelab` via *Manage realms → Create realm*.

**Design decision:** the `master` realm is used **only** to administer Keycloak itself. All users and applications live in `homelab`.

- Separation of administration and regular use: a regular user can never end up with rights over Keycloak itself.
- Each realm has its own users, clients, roles and authentication settings, fully isolated from other realms.
- This mirrors how Keycloak is used in production, where `master` is typically locked down.

---

## 3. A regular user

Created user `jarich` in realm `homelab`:
- Username, email, first and last name.
- Password set with *Temporary* off.
- Required user action: **Configure OTP**.

Note: Keycloak stores usernames in lowercase, so `Jarich` became `jarich`.

The same email address also exists on `kcadmin` in `master`. That is not a conflict: realms have completely separate user stores.

---

## 4. Mandatory MFA (OTP)

Two settings:
1. **For the existing user:** *Users → jarich → Required user actions → Configure OTP*.
2. **For all future users:** *Authentication → Required actions → Configure OTP → Set as default action*. New users must set up OTP on their first login.

Note: a default action only applies to users created **after** it is enabled. Existing users need the required action set individually, which is why step 1 is needed.

### Test

1. Opened a private browser window (so the admin session stays intact) and went to:
   `http://192.168.178.11:8080/realms/homelab/account`
2. Signed in as `jarich` with username and password.
3. Keycloak showed a QR code. Scanned it with an authenticator app, entered the 6-digit code and named the device.
4. Signed out and in again: Keycloak now asks for a one-time code after the password. ✅

---

## Result

| Item | Status |
|---|---|
| Bootstrap admin replaced by permanent account `kcadmin` | ✅ |
| Realm `homelab` for users and applications | ✅ |
| User `jarich` | ✅ |
| MFA (TOTP) mandatory, tested end to end | ✅ |

## What I learned

- **Order matters in access changes.** Create and verify the new admin before removing the old one.
- **Separate administration from usage.** The `master`/`homelab` split is the same least-privilege idea as separate admin accounts in Entra ID.
- **Defaults are not retroactive.** A default required action does not affect existing users.
- **Test in a private window.** It keeps the admin session alive and shows exactly what a real user sees.

## Next steps

- First SSO integration via **OIDC**: let Portainer sign in through Keycloak.
- Groups and roles in `homelab`, mapped to permissions in the application.
- Later: SAML, federation with Microsoft Entra ID, and HTTPS behind a reverse proxy.
