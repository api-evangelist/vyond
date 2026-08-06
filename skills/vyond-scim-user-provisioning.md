---
name: Provision Vyond users over SCIM 2.0
description: Create, read, update, patch and deactivate Vyond user accounts through the SCIM 2.0 API, using the RFC 7644 filter and PATCH grammar and Vyond's own constraints.
api: openapi/vyond-openapi-original.json
generated: '2026-08-05'
method: generated
source: https://api.vyond.com/doc/#tag/SCIM
operations:
  - ScimController.getUsers
  - ScimController.createUser
  - ScimController.getUser
  - ScimController.updateUser
  - ScimController.patchUser
  - ScimController.getSchemas
  - ScimController.getSchema
---

# Provision Vyond users over SCIM 2.0

Base URL `https://api.vyond.com/scim/v2/`. This is a genuine RFC 7643/7644
implementation — `application/scim+json`, SCIM schema URNs, index pagination, the
SCIM filter grammar and `scimType` error discriminators. Verified live: an
unauthenticated `GET /scim/v2/ServiceProviderConfig` returns
`401 application/scim+json` with a
`urn:ietf:params:scim:api:messages:2.0:Error` body.

## Prerequisites

- **Enterprise plan with SSO enabled.** Without it, `createUser` returns
  `403 SSO not enabled`.
- The token is the **SCIM API token**, generated in the Vyond app under
  *Security > SCIM Provisioning > Authorization token*. It is not the REST
  Personal Access Token. Send it as `Authorization: Bearer <token>` — in Okta this
  is the "OAuth Bearer Token" field of the API integration.

## Discover the schemas first

`ScimController.getSchemas` — `GET /scim/v2/Schemas`
`ScimController.getSchema` — `GET /scim/v2/Schemas/{id}`

Vyond extends the core user with a Vyond-specific schema (carrying a subscription
object) and the standard enterprise-user extension. Read the schemas rather than
assuming the attribute set.

## List and filter users

`ScimController.getUsers` — `GET /scim/v2/Users`

- `startIndex` — 1-based index of the first result
- `count` — page size
- `filter` — SCIM 2.0 filter grammar (RFC 7644 §3.4.2.2)

The response is a `ScimUserList` with `totalResults`, `itemsPerPage`, `startIndex`
and `Resources[]`. Page by incrementing `startIndex` by `count` until you have
`totalResults`.

`400` means an invalid filter or pagination parameter — `reason` carries the
message. `403` means the token lacks permission.

## Read one user

`ScimController.getUser` — `GET /scim/v2/Users/{userId}`.
`404` means the user does not exist in the account.

## Create a user

`ScimController.createUser` — `POST /scim/v2/Users`

**Vyond's hard constraint: `userName` must equal the email address.** Violating it
returns `400` with the validation message in `reason`.

Failures to handle distinctly:

- `403 SSO not enabled, unsupported country, or insufficient team seats` — three
  very different causes behind one status; read `reason`.
  *Insufficient team seats* is a commercial condition, not a bug: stop and escalate.
- `409 Conflict` with `scimType: uniqueness` — the email or username is already
  registered. Reconcile against the existing user rather than retrying.

## Update a user

`ScimController.updateUser` — `PUT /scim/v2/Users/{userId}` (full replacement)
`ScimController.patchUser` — `PATCH /scim/v2/Users/{userId}` (`ScimPatchOp`)

Prefer PATCH. Use it for attribute changes and for **deactivation** — set `active`
to false rather than attempting a delete; there is no SCIM DELETE operation in the
Vyond contract.

`400` on PATCH means an invalid operation or attribute value, and may carry a
`details[]` array of `ValidationDetail` entries (`property`, `message[]`) when
validation fails. `409` with `scimType: uniqueness` applies here too.

## What is not supported

- **Groups.** `ScimGroup`, `ScimGroupList` and `ScimGroupMember` exist in the
  OpenAPI components, but no `/scim/v2/Groups` operation is defined and Vyond's own
  SCIM Provisioning article states Groups is unsupported in the current version.
  Do not build group-based provisioning against this API.
- **DELETE.** No user-deletion operation exists. Deactivate with PATCH.
- **`/scim/v2/ServiceProviderConfig` and `/ResourceTypes`** respond on the host but
  are not declared in the OpenAPI. Treat them as undocumented.

## Errors

Same envelope as REST but served as `application/scim+json`:

```json
{"err": "...", "reason": "...", "scimType": "uniqueness", "details": []}
```

Every SCIM operation also declares `401` and `429`. Full catalog:
`errors/vyond-problem-types.yml`.
