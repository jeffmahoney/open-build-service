# Service: signer

Replaces: `bs_signer`, `BSSrcServer/Signkey.pm` (key operations), the
`sign`/`signd` client usage in `bs_publish`, `bs_regpush`, `bs_notar`,
`BSPublisher/Registry.pm`, `BSSSHSign.pm`, `BSPGP.pm` (packet building),
`BSX509.pm`/`BSASN1.pm` (cert generation), `BSTUF.pm`, `BSConSign.pm`,
`BSRekor.pm` (as libraries used by callers).

Reference analysis: `01-legacy-system-analysis.md` § P.2.
Contract: `proto/sign.proto`.

## 1. Two parts

1. **sign-svc** (gRPC, small fixed replica count, hardened): key management
   and digest signing. Never exposes private key material. Backends:
   - `signd` adapter: speaks the obs-sign TCP protocol to a legacy `signd`
     (migration path; the wrapped `_signkey` blobs keep working).
   - KMS/HSM adapter (PKCS#11, AWS KMS, GCP KMS, Vault Transit): keys
     referenced by id; PGP packets are built around raw signatures.
   - Software adapter (keys sealed with an instance master key in
     PostgreSQL `signkey.privkey_ref`): for development and small instances.
2. **sign-worker** (KEDA on `obs.sign.*`): the job-result signing stage
   between `built` and `finished`, plus a library (`pkg/signing`) used by
   publisher/registry for format splicing.

## 2. sign-svc API

```
service Sign {
  rpc GenerateKey(GenerateKeyRequest) returns (KeyInfo);      // algo (rsa@4096 default), expiry days (800), name, email; returns pubkey + key ref (wrapped blob or KMS id)
  rpc ExtendKey(ExtendKeyRequest) returns (KeyInfo);          // days
  rpc GetPublicKey(KeySelector) returns (PublicKey);          // armored PGP
  rpc GetCertificate(CertRequest) returns (Certificate);      // X.509 PEM for signtype cms|appx|notary; cached per project in signkey.sslcert
  rpc SignDigest(SignDigestRequest) returns (Signature);      // mode: PGP_V3 | PGP_V4_DETACHED | PGP_CLEARSIGN | RAW_PKCS1 | CMS | AUTHENTICODE; hash algo; signing time
  rpc SignRpm(stream SignRpmRequest) returns (stream SignRpmResponse); // header+payload digests in, signature header packets out; -T time, -S checksums
  rpc BulkSign(stream Digest) returns (stream Signature);     // rsasign cpio
  rpc ResolveKey(ResolveKeyRequest) returns (KeySelector);    // project hierarchy walk (findsignkey), signtype, signflavor, package; default key fallback
}
KeySelector { string project; string package; string signtype; string signflavor; string key_ref; }
```

Rules:
- `ResolveKey` walks the project namespace upward (`home:a:b` → `home:a` →
  `home`) and returns the first project with a key; the sentinel "key lives in
  signd" (`_signkey` length ≤ 2) maps to a selector without `key_ref`.
- Auto-extend: if the pubkey expires within 14 days and the caller passed
  `autoextend`, extend and persist.
- `forceprojectkeys`: `source-svc` calls `GenerateKey` on project creation;
  deleting the last key in a hierarchy is refused.
- Audit log of every signing operation (who, selector, mode, digest).
- mTLS; only `sign-worker`, `publisher`, `registry-svc`, `source-svc`,
  `compat-edge` (for `_keyinfo`/`_pubkey`/`getsslcert`) may call it.

## 3. sign-worker

Implements `signjob` (P § 2.3):

1. Consume `obs.sign.job` (flavours `default`, `iso`, `rsasign`).
2. Lock the job (`job.state = signing`), download the result bundle to a
   scratch volume, select sign files by the legacy suffix list.
3. Obtain key material/cert via `ResolveKey`/`GetPublicKey`/`GetCertificate`.
4. Per artefact, splice signatures using `pkg/signing`:
   rpm/drpm header signature (`go-rpmutils`-style header rewrite, `-T
   buildtime` for drpms, `--delsign` and re-sign for aggregates with
   signtime clamping), Arch `.sig`, AppImage embedded, `.sha256`/`.sha512`
   detached `.asc` with iso checksum fixups, ISO placeholder patching
   (`sIGnMe!`, `sIGnMeP`, `.p7s` CMS, PVD tag re-tag and `;signature=`
   block), kiwi tree placeholders, `.appx` Authenticode (via sign-svc
   `AUTHENTICODE` mode or external `osslsigncode` in the sandbox), `.apk`
   RSA signature, helm `.prov` clearsign, `.rsasign`/`.cpio.rsasign` raw
   signatures (followup builds, max 3 steps), SLSA provenance DSSE with
   `addrefs` pinning through `artifact-svc`.
5. Rebuild the job's `bininfo` (hdrmd5/leadsigmd5/md5sum change after
   signing) and the `.checksums` sidecar (`-S` equivalent) for createrepo.
6. Upload the signed bundle (new object), set `job.state = finished`, publish
   `obs.build.job.finished`.
7. Errors: `fatal:` → `result=failed`; corrupt rpm or `addrefs` 404 →
   rebuild (purge job, scheduler re-creates); else retry in 3 min.

## 4. Library `pkg/signing`

PGP packet parsing/building (`go-crypto/openpgp` plus a thin packet layer
for v3/v4 detached and one-pass messages), X.509 self-signed root for
TUF, TUF metadata (`go-tuf`), DSSE PAE, cosign simplesigning payload and
cookie (`sha256("OBS/<fingerprint>/<gun>")`), sigstore bundle v0.3, Rekor
client (`hashedrekord`, `rekord`, `intoto`, `dsse`, v2 tiles), atomic
container signatures, SSHSIG framing, PKCS#7/CMS detached signatures. All
signatures come from `sign-svc.SignDigest`; the library only frames.

## 5. Tasks

`work-breakdown/epics.md` E9.
