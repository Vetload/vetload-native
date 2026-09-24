## What and why

<!-- One concern per pull request. Link the issue. -->

Closes #

**Component:** Cnn

## How verified

<!-- Commands run and their results. For every new test: confirm you saw it fail before the fix. -->

- [ ] New or changed tests were seen failing, then passing
- [ ] Checks that report "nothing found" have a positive control

## Checklists

**Security**
- [ ] Untrusted bytes are parsed only in isolated child processes or bounded memory-safe code
- [ ] Every stored or cached record is scoped to its organisation and project
- [ ] Outbound requests follow the URL policy in `docs/architecture/security.md`
- [ ] No secrets, signed URLs, keys or file contents in code, fixtures or logs

**Cost**
- [ ] No new always-on resource, AWS service or paid dependency, or an ADR is linked
- [ ] Logging stays within the per-request budget

**Contracts and docs**
- [ ] Shared shapes changed only through `contracts/`
- [ ] User-facing change documented, or a docs issue is open on C20
