# RFC 0004: Time-based release schedule for Qrisp

- Feature Name: `release_schedule`
- Start Date: 2026-11-20
- RFC PR:
- Qrisp Issue:

## Summary
[summary]: #summary

This RFC proposes a time-based release schedule for Qrisp: a new minor version every quarter (last Friday of January, April, July and October, starting with 0.10.0 on 2027-01-29), a one-week feature freeze before each release, bug-fix releases for the two previous minor versions, a `release/<major>.<minor>` branching model, and a rotating release manager role backed by a written release document.

## Motivation
[motivation]: #motivation

A predictable release cadence gives users regular access to new features and bug fixes, and helps contributors and downstream projects plan their work. Rather than relying only on feature-driven releases, we propose a time-based schedule combined with a limited maintenance policy for previous releases.

## Background and current architecture
[background-and-current-architecture]: #background-and-current-architecture

Qrisp releases are currently feature-driven: a release happens when a set of features is considered ready. There is no fixed cadence, no defined maintenance window for older versions, and no documented process for who performs a release and which steps it involves.

## Proposed design
[proposed-design]: #proposed-design

### Release cadence

- **Quarterly releases:** releases happen on the last Friday of January, April, July and October, starting with 0.10.0 on 2027-01-29.
- **Feature freeze:** one week before the release date, the release branch is created. From then on it only accepts bug fixes, while `main` stays open for the next release. That week is reserved for stabilization, testing, documentation and release notes.
- **Fixed dates:** releases happen on schedule. Features that are not ready by the freeze move to the next release.

### Versioning

- Minor versions may contain breaking changes. They must be listed in the release notes.

### Bug-fix releases

- Bug-fix releases contain fixes only, no new features.
- Each scheduled release publishes one new minor version plus bug-fix releases for the two previous minor versions. Older versions are no longer maintained.
- If a maintained version has received no fixes, no bug-fix release is published for it.
- Only in very critical cases (e.g. severe correctness or security issues) is a bug-fix release published outside the schedule, applied to all maintained versions. The maintainers decide this in the Qrisp community meeting.

### Branching

- All fixes are merged into `main` first, then cherry-picked to the maintained release branches.
- Each minor version has a release branch named `release/<major>.<minor>`, e.g. `release/0.11`.
- Releases are tagged on their release branch as `v<major>.<minor>.<patch>`, e.g. `v0.11.0`, `v0.11.1`.

### Example: 2027

| Freeze     | Release    | New version | Bug-fix releases |
|------------|------------|-------------|------------------|
| 2027-01-22 | 2027-01-29 | 0.10.0      | –                |
| 2027-04-23 | 2027-04-30 | 0.11.0      | 0.10.1           |
| 2027-07-23 | 2027-07-30 | 0.12.0      | 0.11.1, 0.10.2   |
| 2027-10-22 | 2027-10-29 | 0.13.0      | 0.12.1, 0.11.2   |

After October 2027, 0.10 is no longer maintained.

### Release managers

- Releases are done by a rotating release manager, chosen from a list of release managers. Each person handles two consecutive releases, then the role passes to the next person on the list.
- The other release managers help when needed, for example if the current release manager is unavailable.
- A release document describes every step of the process and keeps a record of previous releases, including who managed them.
- The list of release managers is kept in the release document. Maintainers add new release managers in the Qrisp community meeting.

### Release checklist

The release document contains the full steps. At minimum, each release includes:

- Integration and regression testing.
- Documentation updates.
- Release notes covering new features, improvements, breaking changes, deprecations and bug fixes.
- Publication of packages and release artifacts.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

- Who is on the initial list of release managers, and who manages 0.10.0 and 0.11.0?
- Where does the release document live (e.g. in the Qrisp repository or its documentation)?

## Future possibilities
[future-possibilities]: #future-possibilities

- A formal deprecation policy, e.g. how many minor versions a deprecated API is kept before removal.
- Automating parts of the release checklist (cherry-picks to release branches, tagging, package publication) in CI.

## Public API impact
[public-api-impact]: #public-api-impact

None. This RFC changes the release process, not the Qrisp API. Breaking API changes remain possible in minor versions but must be listed in the release notes.
