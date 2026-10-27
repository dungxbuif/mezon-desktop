# Clan administration: roles, permissions, community, audit

**Group:** #7 — Clan administration
**Significance:** Critical
**Commits:** `3c0b500` (clan settings), `f196e7e` (roles restructure), `41a07a5` (permissions), `9a70598` (audit log), `85c5854` (community), `c1224a0` (members page), `2877dcb` (i18n)
**Areas:** `crates/mezon-ui/src/clan/settings/*`, `chat/clan_members_page.rs`, `clan_channels_page.rs`, `chat/channel_settings/*`, `crates/mezon-store/src/roles.rs`, `channel_role_permissions.rs`, `audit_log.rs`

## Change

Clan administration was overhauled: a **role-based permission system** (replacing flat admin/member lists), per-channel permission overrides, a clan audit log, a community settings page, and expanded member/channel management UI.

## Before

At `5a161c5`, `ClanRoles` was a flat struct (`owner, admins: Vec, members: Vec`). Permissions were implicit (admins can do everything, members can do a subset). There was no per-channel permission override, no audit log, no community settings page. The clan members page was a simple list.

## After

### Role-based permissions (`mezon-store/src/roles.rs`)

- `ClanRoles { owner_id, roles: HashMap<RoleId, Role> }` (breaking change — see store report).
- `Role { id, name, color, permissions: PermissionSet, position: i32 }`.
- `PermissionSet` is a bitfield of `Permission` flags: `ManageChannels`, `ManageRoles`, `ManageWebhooks`, `KickMembers`, `BanMembers`, `ViewAuditLog`, `SendMessages`, `ManageMessages`, `MentionEveryone`, `ConnectVoice`, `Speak`, `ShareScreen`, etc.
- Roles have a `position` for hierarchy (higher position = more authority).

### Per-channel permission overrides (`channel_role_permissions.rs`)

- `ChannelRolePermissions { channel_id, overrides: HashMap<RoleId, PermissionOverride> }`.
- `PermissionOverride { allowed: PermissionSet, denied: PermissionSet }`.
- A role's effective permissions in a channel = base `PermissionSet` ∪ `allowed` − `denied`.

### Audit log (`audit_log.rs`)

- `AuditLogEntry { id, clan_id, actor_id, action: AuditAction, target_id, timestamp, reason }`.
- `AuditAction`: `MemberKicked`, `MemberBanned`, `RoleAssigned`, `RoleRemoved`, `ChannelCreated`, `ChannelDeleted`, `WebhookCreated`, `WebhookDeleted`, `MessageDeleted`.
- UI: `clan/settings/audit_log.rs` — paginated, filterable by action type and actor.

### Clan settings UI (`crates/mezon-ui/src/clan/settings/`)

- `roles.rs` — role list, create/edit/delete roles, assign permissions via checkboxes.
- `permissions.rs` — matrix view of roles × permissions.
- `audit_log.rs` — audit log viewer.
- `community.rs` — community settings (welcome screen, rules, description).
- `members.rs` — member list with role assignment, kick/ban actions.
- `channels.rs` — channel list with per-channel permission overrides.

### Member & channel pages

- `chat/clan_members_page.rs` — expanded to show member roles, online status, and inline role assignment.
- `clan_channels_page.rs` — channel list with permission summary per channel.
- `chat/channel_settings/` — per-channel settings: permissions, webhooks, slowmode.

## Mechanism

### Permission check flow (Observed)

```
User attempts action (e.g., kick member)
   │
   ↓
mezon-ui ──> check_permission(clan_id, user_id, Permission::KickMembers)
   │
   ↓
mezon-store::roles::effective_permissions(clan_id, user_id)
   │
   ├─ find user's roles in ClanRoles
   ├─ union their base PermissionSets
   └─ apply channel overrides (if action is channel-scoped)
   │
   ↓
return (effective_permissions & Permission::KickMembers) != 0
```

### Audit log write flow (Observed)

```
Action executed (e.g., MemberKicked)
   │
   ↓
mezon-store::audit_log::append(clan_id, actor_id, MemberKicked, target_id, reason)
   │
   ↓
StoreEvent::AuditLogAppended ──> UI audit log viewer refreshes
```

## Motivation

- **Observed:** The permission flags mirror Discord's model — a familiar UX for clan admins.
- **Strong inference:** Per-channel overrides are needed because a single clan-wide permission set is too coarse (e.g., "mods can manage #announcements but not #off-topic").
- **Strong inference:** The audit log is a trust feature for large clans — admins need to see who did what.
- **Hypothesis:** The `ClanRoles` breaking change was forced because the old flat-list model couldn't express per-role permissions; the migration cost was accepted to unlock the feature.

## Impact

- **Admins:** can now create custom roles with fine-grained permissions, override per channel, and audit actions.
- **Members:** see roles in the member list; UI gates actions based on permissions.
- **Store:** `ClanRoles` schema change (breaking — see store report); new `audit_log`, `channel_role_permissions` modules.
- **UI:** 6 new settings pages; member/channel pages expanded.
- **Security:** permission checks are now enforced in the UI; server-side enforcement is assumed (not in this diff's scope).

## Trade-offs

- **Benefit:** Discord-level admin capabilities; fine-grained control; auditability.
- **Cost:** significant UI complexity (6 settings pages); `ClanRoles` breaking change; permission check overhead on every gated action.
- **Risk:** UI-enforced permissions without server-side enforcement are bypassable (a modified client can skip the check). Server enforcement is assumed but not verified in this diff.
- **Rollback:** reverting `roles.rs` to flat-list + removing the settings pages restores the old model, but persisted `ClanRoles` data from the new format would be unreadable.

## Verification

- `crates/mezon-store/src/roles.rs` defines `ClanRoles`, `Role`, `PermissionSet`, `Permission` (Observed).
- `crates/mezon-store/src/channel_role_permissions.rs` defines `ChannelRolePermissions`, `PermissionOverride` (Observed).
- `crates/mezon-store/src/audit_log.rs` defines `AuditLogEntry`, `AuditAction` (Observed).
- `crates/mezon-ui/src/clan/settings/` contains `roles.rs`, `permissions.rs`, `audit_log.rs`, `community.rs`, `members.rs`, `channels.rs` (Observed).
- `crates/mezon-ui/src/chat/clan_members_page.rs` expanded with role display (Observed).
- **Not verified:** server-side permission enforcement; migration for `ClanRoles`; permission check performance under large role sets.

## Source-reading guide

1. `crates/mezon-store/src/roles.rs` — `ClanRoles`, `Role`, `PermissionSet`, `Permission` enum.
2. `crates/mezon-store/src/channel_role_permissions.rs` — `PermissionOverride`, effective-permission calculation.
3. `crates/mezon-store/src/audit_log.rs` — `AuditLogEntry`, `AuditAction`.
4. `crates/mezon-ui/src/clan/settings/roles.rs` — role management UI.
5. `crates/mezon-ui/src/clan/settings/permissions.rs` — permission matrix.
6. `crates/mezon-ui/src/clan/settings/audit_log.rs` — audit log viewer.
7. `crates/mezon-ui/src/chat/clan_members_page.rs` — member list with roles.
8. `crates/mezon-ui/src/chat/channel_settings/` — per-channel settings.

## Learning

- **Bitfield permission sets** (`PermissionSet`) are the standard for fine-grained perms — fast to check, easy to union/intersect, but hard to debug (a single wrong bit = wrong permission).
- **Per-channel overrides** (allowed/denied) are more expressive than a single mask but require careful ordering: base ∪ allowed − denied. Getting the order wrong grants or denies incorrectly.
- **UI-enforced permissions are a UX gate, not a security boundary** — the server must independently enforce. Always check both sides.

## Unknowns

- **Does the server enforce permissions, or only the UI?** (Critical. Not verifiable from this client-side diff. Hypothesis: server enforces, based on `audit_log` being server-side.)
- Is there a migration for `ClanRoles` from the old flat-list format? (Not found; see store report.)
- What's the max number of roles per clan? (Performance concern for `effective_permissions` with many roles.)
- Can a role's `position` be edited, and does it enforce hierarchy (can't edit a role above yours)? (Hypothesis: yes, but not confirmed.)
