---
name: kumospace-onboard-members
description: Invite people into a Kumospace space and assign their roles — single and bulk invitations, expiring invite links, resends, and per-space and per-floor role assignment.
api: kumospace:api
generated: '2026-07-19'
method: generated
source: openapi/kumospace-api-openapi.json
base_url: https://api.kumospace.com
operations:
  - GetInvitationsForSpace
  - CreateUserInvitation
  - CreateUserInvitationsBulk
  - ResendInviteEmail
  - UpdateUserInvitation
  - DeleteUserInvitation
  - CreateExpiringInvitation
  - GetExpiringInviteInfo
  - GetUserRoles
  - SetUserRole
  - SetUserRoleBulk
  - SetFloorRoles
---

# Onboard members into a Kumospace space

Get people into a space and give them the right level of access.

## Before you start

- All operations here require `Authorization: Bearer <firebase-id-token>`.
- Spaces are keyed by their `name` slug. Invitations are keyed by `invitationId`.
- There is **no idempotency key**, so an ambiguous `CreateUserInvitation` retry can send a second invite
  email. Always reconcile with `GetInvitationsForSpace` before retrying.
- The spec declares **no error responses**, so you cannot distinguish "already invited" from "not
  permitted" from the contract. Read the outstanding invitation list to disambiguate.

## Steps

1. **List what is outstanding.** `GetInvitationsForSpace`
   (`GET /v1/userInvitations/space/{spaceName}`) returns pending invitations for the space. Run this
   first — it is your only guard against duplicate invites.

2. **Invite people.**
   - One at a time: `CreateUserInvitation` (`POST /v1/userInvitations`).
   - Many at once: `CreateUserInvitationsBulk` (`POST /v1/userInvitations/bulk`). Prefer this for
     onboarding a team; it is one request instead of N unguarded creates.

3. **Or hand out a link.** `CreateExpiringInvitation` (`POST /v1/invitations`) mints a time-limited
   invite link — better for events and open sessions than per-address invitations. Recipients (and you)
   can inspect it with `GetExpiringInviteInfo` (`GET /v1/invitations/{id}`). A space may also carry a
   standing add-member link, readable via `GetAddMemberLink`
   (`GET /v1/spaces/{name}/addMemberLink/{addMemberLinkId}`).

4. **Chase or clean up.**
   - `ResendInviteEmail` (`POST /v1/userInvitations/{invitationId}/resend`)
   - `UpdateUserInvitation` (`PUT /v1/userInvitations/{invitationId}`)
   - `DeleteUserInvitation` (`DELETE /v1/userInvitations/{invitationId}`)

5. **Assign roles.** Roles are per space, held in `ApiUserRole` (`userId`, `email`, `displayName`,
   `role`, `avatarUrl`).
   - Read current state: `GetUserRoles` (`GET /v1/spaces/{name}/userRoles`), or a single user with
     `GetUserRoleById` (`GET /v1/spaces/{name}/userRole`).
   - Set one: `SetUserRole` (`PUT /v1/spaces/{name}/userRoles`).
   - Set many: `SetUserRoleBulk` (`PUT /v1/spaces/{name}/userRoles/bulk`) — use this when promoting a
     cohort after a bulk invite.
   - Scope access to a floor: `SetFloorRoles` (`PUT /v1/spaces/{name}/floorRoles`).

6. **Confirm.** Re-read `GetUserRoles` and reconcile against the list you intended to onboard.

## Notes and gotchas

- Auto-join by email domain is a space-level setting, not an invitation: `emailDomain` and
  `emailDomains` on `ApiSpace`, changed via `UpdateSpace`. For a whole company, setting the domain can
  replace per-address invitations entirely.
- `memberLimit` and `memberCount` on `ApiSpace` bound how many people you can onboard on the current
  tier. Check them before a bulk invite — the API declares no specific error for exceeding the limit.
- Individual rooms can restrict access independently of space roles, via `isAccessRestricted` and
  `accessAllowList` on `ApiRoom`.
- Onboarding forms are separate: `GetUserIntakeProfileFormStatus`
  (`GET /v1/users/me/profile/form/{spaceName}/response-required`) tells you whether a new member still
  owes an intake profile, submitted with `SaveUserIntakeProfileForm` (`POST /v1/users/me/profile/form`).
