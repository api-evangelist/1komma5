---
name: Administer 1KOMMA5° Offer Tool tenants, branches and users
description: Switch tenant context, provision and update admin users, assign branches and
  roles, resend invitations, and read the audit trail in the 1KOMMA5° Offer Tool API.
api: openapi/1komma5-offer-tool-openapi-original.json
generated: '2026-08-02'
method: generated
source: openapi/1komma5-offer-tool-openapi-original.json
operations:
- TenantController_getTenants_v1
- TenantController_switchActiveTenant_v1
- GetCurrentAdminUserController_getCurrentAdminUser_v1
- GetUserProfileController_handle_v1
- UpdateUserController_handle_v1
- UserProfilePhotoController_handle_v1
- UserProfilePhotoController_delete_v1
- Auth0UserController_userExistsByEmail_v1
- GetAdminUsersController_getAdminUsers_v1
- CreateAdminUserController_createAdminUser_v1
- GetAdminUserController_getAdminUser_v1
- UpdateAdminUserController_updateAdminUser_v1
- ResendInvitationController_resendInvitation_v1
- GetMyBranchesController_getMyBranches_v1
- GetAssignableBranchesController_getAssignableBranches_v1
- GetAdminBranchesController_getAdminBranches_v1
- CreateAdminBranchController_createAdminBranch_v1
- GetAdminBranchController_getAdminBranch_v1
- UpdateAdminBranchController_updateAdminBranch_v1
- GetAssignableRolesController_getAssignableRoles_v1
- GetAllUsersController_getUsers_v1
- GetSalesUsersController_getSalesUsers_v1
- GetAuditLogsController_getAuditLogs_v1
---

# Administer 1KOMMA5° Offer Tool tenants, branches and users

The Offer Tool is multi-tenant. **Tenant context is a property of the caller, not of the
request** — get it wrong and reads come back empty or 404 rather than 403.

## Steps

1. **See who you are.** `GetCurrentAdminUserController_getCurrentAdminUser_v1`
   (`GET /api/v1/admin/users/me`) for the admin identity, or
   `GetUserProfileController_handle_v1` (`GET /api/v1/users/me`) for the sales-user
   profile. The user record carries `activeTenantId`, `defaultTenantId`, a `tenantIds`
   membership list and `currentTeamId`.
2. **List and switch tenants.** `TenantController_getTenants_v1`
   (`GET /api/v1/tenant`), then `TenantController_switchActiveTenant_v1`
   (`POST /api/v1/tenant/switch`). **Do this before any list or lookup call** — every 404
   in this API is really "not in your active tenant" until proven otherwise.
3. **Branches.** `GetMyBranchesController_getMyBranches_v1`
   (`GET /api/v1/admin/users/me/branches`) for the caller's own;
   `GetAssignableBranchesController_getAssignableBranches_v1` for what may be granted to
   someone else. Manage with `GetAdminBranchesController_getAdminBranches_v1`,
   `CreateAdminBranchController_createAdminBranch_v1`,
   `GetAdminBranchController_getAdminBranch_v1` and
   `UpdateAdminBranchController_updateAdminBranch_v1`. A `404` is "Branch not found".
4. **Check before you create.** `Auth0UserController_userExistsByEmail_v1`
   (`POST /api/v1/auth0/user/exists`). Skipping this earns a `409` "Email already exists"
   from `CreateAdminUserController_createAdminUser_v1` — and because there is **no
   idempotency contract**, a blind retry of the create is not safe.
5. **Provision.** `CreateAdminUserController_createAdminUser_v1`
   (`POST /api/v1/admin/users`), assigning roles from
   `GetAssignableRolesController_getAssignableRoles_v1` (`GET /api/v1/admin/roles`) and
   branches from step 3. Amend with `UpdateAdminUserController_updateAdminUser_v1`
   (`PATCH /api/v1/admin/users/{id}`) — it can also return `409` on an email collision.
6. **Invitations.** `ResendInvitationController_resendInvitation_v1`
   (`POST /api/v1/admin/users/{id}/resend-invitation`). A `400` here means
   "Auth0 not configured or failed to send email" — an upstream identity-provider problem,
   not a bad request from you. `404` is "User not found".
7. **Profile self-service.** `UpdateUserController_handle_v1`
   (`PUT /api/v1/users/me`), `UserProfilePhotoController_handle_v1`
   (`POST /api/v1/users/me/profile-picture`, `multipart/form-data`) and
   `UserProfilePhotoController_delete_v1`.
8. **Directories.** `GetAllUsersController_getUsers_v1` (`GET /api/v1/users`) and
   `GetSalesUsersController_getSalesUsers_v1` (`GET /api/v1/users/sales-persons`) — note
   neither takes paging parameters, so expect a full list.
9. **Audit.** `GetAuditLogsController_getAuditLogs_v1` (`GET /api/v1/admin/audit-logs`)
   filters on `action`, `actorType`, `actorIdentifier`, `subjectType`,
   `subjectIdentifier`, `traceId`, `tenantIds`, `startDate`, `endDate`, with
   `page`/`limit`/`sortBy`/`sortOrder`. `traceId` is the only request-correlation handle
   the Offer Tool exposes — the error envelope does not carry one.

## Do not

- Do not treat `PATCH /api/v1/admin/users/{id}` as safe to repeat: with no idempotency
  key, only the final state is guaranteed, and a concurrent editor will silently win.
- Do not call `BackfillConfigHashesController_backfillConfigHashes_v1` or anything under
  `/api/v1/migration-frozen-state/` — Superadmin-only maintenance jobs, unrelated to user
  administration.
- Do not expect a scope check to protect you. Authorization here is role- and
  tenant-based; the Auth0 tenant advertises only standard OIDC identity scopes
  (`scopes/1komma5-scopes.yml`), no product-resource scopes.
