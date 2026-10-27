# Simple AD to AWS Managed Microsoft AD WorkSpaces Migration Automation

A set of AWS Systems Manager (SSM) Automation documents that help you migrate
Amazon WorkSpaces users from a legacy **AWS Directory Service Simple AD**
directory to **AWS Managed Microsoft AD**. WorkSpaces cannot be moved between
directories, so the migration recreates the identities and the WorkSpaces under
the new directory.

> **Sample code.** These documents are provided as a starting point. Review
> them, scope the IAM policies to your environment, and test in a non-production
> account before using them on production directories. You run them in your own
> AWS account, against your own directories, on your own management instance.

## What the documents do

| Document | Purpose |
|---|---|
| `AD-Migration-Orchestrator.json` | Exports users, OUs, groups and group memberships from the source Simple AD (via LDAP / `csvde`, because Simple AD is Samba 4 with no AD Web Services) and imports them into the target Managed Microsoft AD. Passwords cannot be migrated (Simple AD has no trust and no Password Export Server), so migrated users get a reset password with change-at-logon. |
| `WorkSpaces-Recreate-Orchestrator.json` | Reads each user's source WorkSpace (read-only) to learn its bundle and properties, then creates a matching WorkSpace under the new Managed AD directory. **Create-only: it never terminates or modifies the source WorkSpaces.** |
| `FSx-GPO-Orchestrator.json` | Optional go-forward data path. Authors a Group Policy Object in Managed AD that maps each user's Amazon FSx for Windows home folder at logon, keyed on username so it survives the SID change. |

The automation does **not** create the Managed AD, the FSx file system, or the
management instance, and does **not** copy user data or terminate anything.

## Run order

1. **`AD-Migration-Orchestrator`** — migrate the identities first.
2. **`WorkSpaces-Recreate-Orchestrator`** — recreate the WorkSpaces (run with
   `dryRun=true` first to review the plan, then `dryRun=false`).
3. **`FSx-GPO-Orchestrator`** (optional) — publish the home-drive GPO (run with
   `dryRun=true` first).

`dryRun` defaults to **true** on `WorkSpaces-Recreate-Orchestrator` and
`FSx-GPO-Orchestrator`. Nothing is created or changed until you set
`dryRun=false`.

## Prerequisites

- A **management EC2 instance** with SSM Agent and the RSAT ActiveDirectory /
  GroupPolicy PowerShell modules. The AD-Migration export can run from a
  source-joined box (`useSourceBind=false`) or, with explicit source
  credentials, from a target-joined box (`useSourceBind=true`). GPO authoring
  must run on a **target-joined** box.
- A **Secrets Manager secret** holding JSON of the form
  `{"source":{"username","password"},"target":{"username","password"}}` (the
  `source` block is only needed for `useSourceBind=true`). Pass its name/ARN as
  `migrationSecretId`.
- **IAM roles** scoped to your environment. Example least-privilege policies are
  in [`iam/`](iam/): `automation-role-ad-migration.json`,
  `automation-role-workspaces.json`, `automation-role-fsx-gpo.json`, and
  `instance-role.json`. Replace the `<account-id>`, `<region>`,
  `<directory-id>`, `<instance-id>` and `<secret-name>` placeholders.
  `automation-trust.json` is the SSM Automation trust policy (fill in
  `aws:SourceAccount` and `aws:SourceArn`).
- A target **AWS Managed Microsoft AD** directory, registered with WorkSpaces.
- **For `FSx-GPO-Orchestrator` (Step 3) only: a group Managed Service Account
  (gMSA)** in the target Managed AD to author the GPO. The document requires
  its `gpoAuthorGmsa` parameter and its Preflight hard-fails a `dryRun=false`
  run if the gMSA is not usable on the box. Before running Step 3:
  1. Create the gMSA in the target Managed AD (the creating identity must be a
     member of **AWS Delegated Managed Service Account Administrators**).
  2. Add the management box to the gMSA's
     `PrincipalsAllowedToRetrieveManagedPassword`.
  3. On the management box, run `Install-ADServiceAccount` and confirm with
     `Test-ADServiceAccount`.
  4. Delegate GPO create and link rights to the gMSA on `targetOuDn` (and write
     access on the domain Policies container).

## Security

These controls are part of the design. If you run the SSM documents without
reading the rest of this section, you can create unsafe outcomes.

- **Verify before you decommission the source (T5).** This automation is
  create-only and never terminates source WorkSpaces. Copy user data to the new
  WorkSpaces and verify the new WorkSpaces work **before** you terminate the old
  ones. Terminating a WorkSpace is irreversible.
- **User data copy is yours, and keep PII off intermediate storage (T10 / T11).**
  The automation does not copy user data. Perform the one-time data seed on the
  new Managed AD WorkSpace using your existing, already-approved data channel.
  Do not stage user data through an unsecured S3 bucket; prefer the native
  session token over cross-domain explicit credentials.
- **Pre-create and ACL the FSx home folders before relying on the mapping
  (T18).** The home-drive GPO maps `\\<fsx>\<share>\%USERNAME%` but does not
  create or ACL those folders. On a share whose root lets Authenticated Users
  create folders, a user can pre-create another user's folder and own its ACL
  (home-folder squatting). Lock down the share root and pre-create each per-user
  folder owned by that user (or use FSx for Windows user home directories).
  `FSx-GPO-Orchestrator` Preflight fails closed on a `dryRun=false` run if the
  share is unreachable or its root ACL is too broad.
- **Rotate and delete the migration secret when you are done (T3).** The secret
  holds domain-administrator-equivalent credentials. Rotate the passwords and
  delete the secret once the migration completes, and decommission the
  management instance (encrypted root volume, no retained snapshots).
- **GPO authoring uses a gMSA, so no password is stored (T3).**
  `FSx-GPO-Orchestrator` authors the GPO from a short-lived scheduled task that
  runs as the group Managed Service Account you supply in `gpoAuthorGmsa` (a
  required parameter). The task principal uses `-LogonType Password`, but for a
  gMSA this stores no credential: Task Scheduler retrieves the gMSA password
  from Active Directory at runtime, so nothing is persisted to the management
  box or as an LSA secret. The document still unregisters the task on exit; if
  you want to confirm cleanup before snapshotting or handing back the box,
  check that `Get-ScheduledTask -TaskName 'FsxGpoAuthor'` returns nothing.

## Passwords

Migrated users receive a randomly generated password and must change it at next
logon. Simple AD cannot export password hashes, so this is unavoidable for the
Simple AD to Managed AD path.

## License

This library is licensed under the MIT-0 License. See the [LICENSE](LICENSE)
file.
