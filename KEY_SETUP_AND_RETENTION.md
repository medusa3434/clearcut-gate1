# Acceptance signer key setup plan (NOT executed in this preflight)

This file is a plan only. No keystore was generated and no secret exists in this prepared
area. Key generation happens later, at key-setup time, after the repository authorization
from the ready-to-run plan (A/F) is granted.

## What gets created
A dedicated PKCS12 signing key for the acceptance variant only:

```
keytool -genkeypair -v \
  -keystore acceptance.jks \
  -alias acceptancegate1 \
  -keyalg RSA -keysize 4096 \
  -validity 10000 \
  -dname "CN=ClearCut Gate-1 Acceptance, O=Grid1"
```

- Reuses nothing from ClearCut's release identity (`release-signing-identity.json`
  certificate `76cf4c64…` / alias `androiddebugkey`) and nothing from the Demi/Concat
  project's signing or credentials.
- The test keystore properties under `(keystore.properties` (gitignored upstream) are NOT
  involved; the acceptance lane reads ONLY the `ACCEPTANCE_*` environment variables.

## Secrets exposed to the CI run (4 repository secrets in the isolated driver repo)
```
ACCEPTANCE_STORE_B64        base64(upload) of acceptance.jks file body
ACCEPTANCE_STORE_PASSWORD   store password
ACCEPTANCE_KEY_ALIAS        acceptancegate1
ACCEPTANCE_KEY_PASSWORD     key password
```
- Exposed only to the two steps that need them (decode / build) via step-level `env:`; never
  organization-level secrets; never used in other repositories.

## Secure retention (future-update storage)
1. **Local generator**: produce `acceptance.jks` only in the isolation area
   (e.g. a protected subfolder under the driver's vault directory), never in any Git tree.
2. **Retention vault**: immediately after creation move the keystore file + passwords into
   the operator's permanent secret store (vault manager / encrypted archive). The CI repo
   keeps only the four secrets above, never a keystore file.
3. **Update continuity**: store, on the same record, (a) acceptance applicationId
   `com.novacut.editor.acceptance`, (b) the alias, (c) the keystore passwords, (d) the
   certificate SHA-256 recorded from the CI provenance.json. Same package + same certificate
   preserves future acceptance app data across updates; losing the keystore WILL strand
   acceptance installs (same hard rule the pinned upstream identity exists to prevent).
4. **Never delete acceptance data**: the acceptance app keeps project data in its own
   app-private storage; an update that reuses the same keystore + package keeps it intact.
   Any data removal is an operator decision made on-device; the build lane never touches it.
