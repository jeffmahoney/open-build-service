# Work Breakdown

The specification decomposed into epics and agent-sized tasks.

- [epics.md](epics.md): epics, milestones, dependency graph, staffing
  guidance.
- `E01` … `E17` files: tasks per epic.

## Task format

Every task has:

| Field | Meaning |
|---|---|
| id | `E<nn>-T<nn>` |
| title | one line |
| scope | what to build; which packages/dirs it may touch |
| inputs | spec sections and analysis sections to read first |
| deliverables | code, tests, docs |
| acceptance | objective checks (tests that must pass, parity subsets) |
| depends | task ids that must be done first |
| size | S (≤ 1 agent-day), M (2-3), L (4-6), XL (> 6, must be split before start) |

## Rules for agents taking a task

1. Read the task's inputs completely before writing code; cite the analysis
   section in the PR description.
2. Touch only the directories listed in `scope`. Shared code goes to
   `pkg/` only when the task says so.
3. A task is done when `acceptance` is met in CI, not when code compiles.
4. If the spec and the Perl behaviour disagree and the spec does not say
   "deviation intended", the Perl behaviour wins; open a spec issue.
5. Never change a `.proto` outside a task that lists `proto/` in scope.
6. Record open questions in the task file under "Notes" on completion.

## Repository layout the tasks assume (`obs-ng`)

```
api/proto/obs/v1/      generated code + the .proto sources copied from spec/proto
cmd/<service>/         main packages
internal/<service>/    service implementation
pkg/bsxml              XML::Structured-compatible marshalling + all schemas
pkg/bsverify           name/format validation rules
pkg/bsxpath            OBS XPath dialect
pkg/prjconf            project config parser (scheduler/publisher subset)
pkg/recipe             recipe parsers (spec/dsc/kiwi/Dockerfile/PKGBUILD/...)
pkg/solv               libsolv binding (cgo) + expander + depsort + meta algorithms
pkg/meta               gen_meta / add_meta / diffdepth (pure Go reference)
pkg/obscpio            cpio/obscpio/tar helpers
pkg/rpm, pkg/deb, pkg/arch, pkg/apk  package header queries (hdrmd5, evra, provides)
pkg/contar             container tar normalisation, containerinfo
pkg/signing            PGP/X.509/TUF/DSSE/cosign/Rekor framing
pkg/store              PostgreSQL access layer, migrations
pkg/objstore           S3 client, presign, GC helpers
pkg/events             NATS JetStream helpers, envelope codec
pkg/arch               cando table, known archs
pkg/legacyerr          legacy error catalogue
deploy/charts, deploy/operator
test/corpus            golden corpus loader; test/parity; test/e2e
```
