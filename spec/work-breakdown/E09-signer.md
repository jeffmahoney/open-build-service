# E09 — signer

Milestones M2 (sign-svc, rpm signing) / M3 (rest). Inputs:
`services/signer.md`, analysis P § 2, `proto/sign.proto`.

| id | title | scope | acceptance | depends | size |
|---|---|---|---|---|---|
| E09-T01 | sign-svc core: key store (`signkey` refs), `ResolveKey` hierarchy walk + sentinel, `GenerateKey`/`ExtendKey`/`GetPublicKey`/`GetCertificate`, audit log, auth matrix | `cmd/sign-svc`, `internal/sign` | unit tests with software backend | E03 | M |
| E09-T02 | Backends: software (sealed keys), signd adapter (obs-sign TCP protocol; verify against a real signd in CI), KMS/PKCS#11 adapter interface + one implementation | `internal/sign/backend/*` | signatures verify with gpg | T01 | L |
| E09-T03 | `pkg/signing` framing: PGP v3/v4 detached, clearsign, one-pass messages, raw PKCS#1, CMS, X.509 root for TUF, TUF metadata, DSSE PAE, cosign payload/cookie, sigstore bundle, Rekor client, SSHSIG | `pkg/signing` | byte-compat with Perl outputs on fixtures | T01 | L |
| E09-T04 | rpm header signing splice, `.checksums`, delsign/resign with time clamp; deb sidecars; Arch `.sig`; AppImage; apk; helm `.prov`; ISO placeholder patching + PVD retag; kiwi tree placeholders; appx (external tool); rsasign/bulk | `pkg/signing/formats` | signed artefacts verify with native tools (rpm -K, pacman-key, apk verify, …) | T03 | L |
| E09-T05 | sign-worker: queue consumer, job lock, file selection, followup detection, bininfo rebuild, status/event hand-off, error classes | `cmd/sign-worker`, `internal/signworker` | corpus job results signed and integrated | T04, E07-T06 | M |
| E09-T06 | SLSA provenance DSSE signing with `addrefs` pinning | `internal/signworker/slsa` | provenance verifies | T05, E06-T11 | S |
