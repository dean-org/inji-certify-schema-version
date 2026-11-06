# QA Handover – Credential Schema Versioning (inji-certify)

**Build status (read first):** The Spring modules have NOT been compiled or run by the author (Maven was unreachable).
Executed by the author: resolver logic (12 JUnit tests), JSON-Schema validator (5 JUnit tests, against a partially stubbed scratch build of networknt 2.0.8),
SQL upgrade/rollback on PostgreSQL 16. NOT executed: service wiring, JPA queries, `SchemaVersionServiceTest`, updated `CertifyIssuanceServiceImplTest`, any end-to-end issuance.
Expect to fix compile/wiring issues on the first build. Nothing below has been run against a live Certify.

## 1. What changed
- **DB (`credential_config`)**: new columns `schema_id`, `schema_version`, `schema_status`, `validation_schema` (jsonb); CHECK constraints; unique indexes now include version; a trigger makes published versioned rows immutable (no delete, no structural edit, forward-only status). Scripts: `db_upgrade_script/mosip_certify/sql/0.14.0_to_0.14.1_{upgrade,rollback}.sql` (file name/version is a guess – rename to the real release).
- **API (no new endpoints)**: `/credential-configurations` accepts `schemaId`, `schemaVersion`, `schemaStatus`, `validationSchema`. Credential request accepts optional `schema_version`.
- **Resolution order**: explicit `schema_version` → configured default (`mosip.certify.schema-versioning.default-versions.<schemaId>`) → legacy (unversioned) row → highest ACTIVE.
- **Status**: ACTIVE → DEPRECATED → DISABLED, forward only. DEPRECATED issuable only if explicit/default (warning logged). DISABLED always rejected and hidden from issuer metadata.
- **Validation**: data provider output is checked against the version's JSON Schema (draft 2020-12) before issuance; errors name location + keyword, never values.
- **Plugin (VCIssuance) mode**: `schema_version` is rejected (`schema_version_unsupported_in_plugin_mode`).
- Legacy (unversioned) configurations behave as before.

## 2. Environment prep
1. Back up DB. 2. Run the upgrade SQL. 3. Build: `./mvnw -pl certify-service -am test` then package/deploy. 4. Note the base URL: `BASE=https://<host><server.servlet.path>`.
5. Have an access token for scopes below (obtain via your normal OIDC/pre-auth flow). 6. Confirm `/credential-configurations` is not publicly exposed (it is in `ignore-auth-urls` in the local profile).
Test data: a data-provider plugin returning name, dateOfBirth (YYYY-MM-DD), gender, address, and (for v1.1+) email, (for v2.0) nationality.

## 3. Test data to register
Same credential type/context for all, `schemaId=IdentityCredential`, `credentialFormat=ldp_vc`:
| Key id | Version | Scope | Required fields |
|---|---|---|---|
| IdentityCredential (existing, unversioned) | none | id_scope | none validated |
| IdentityCredential_v1_0 | 1.0 | id_scope_v1_0 | name, dateOfBirth, gender, address |
| IdentityCredential_v1_1 | 1.1 | id_scope_v1_1 | + email |
| IdentityCredential_v2_0 | 2.0 | id_scope_v2_0 | + nationality |
Note: a versioned 1.0 cannot coexist with the unversioned legacy row for the same selector (legacy counts as 1.0). Run the v1.0 cases in a separate environment/type, or skip row 2.
Each version needs its own VC template and (for ldp_vc) an immutable versioned JSON-LD context URL.

## 4. Manual test cases
Register with `POST $BASE/credential-configurations` (JSON body). Issue with `POST $BASE/issuance/credential` (bearer token + proof; add `"schema_version":"x.y"` as needed). Error codes are the `error` values in the response.

| # | Case | Steps | Expected |
|---|---|---|---|
| 1 | Legacy unchanged | Before upgrade, issue a credential; after upgrade+deploy, issue again (no schema_version) | Output identical in structure/claims/template |
| 2 | Upgrade safety | Compare existing `credential_config` rows before/after upgrade | Rows unchanged; new columns NULL |
| 3 | Register v1.1 | POST v1.1 body with validationSchema | 201; GET returns schemaVersion 1.1, status ACTIVE |
| 4 | Register v2.0 | POST v2.0 | 201 |
| 5 | Duplicate version | POST v1.1 again with a different key id | Rejected (duplicate) |
| 6 | Bad version string | POST with schemaVersion "v1" / "1" | 400 `invalid_schema_version` |
| 7 | validationSchema without version | POST with validationSchema, no schemaVersion | Rejected |
| 8 | Remote $ref | validationSchema `{"$ref":"http://x/s.json"}` | Rejected `invalid_schema_definition` |
| 9 | Dangling local ref | `{"$ref":"#/$defs/missing"}` | Rejected `invalid_schema_definition` |
| 10 | Issue explicit v1.1 | schema_version "1.1", token scope id_scope_v1_1, provider returns email | Credential issued with v1.1 template |
| 11 | Issue explicit v2.0 | schema_version "2.0", scope id_scope_v2_0 | Issued with v2.0 template incl. nationality |
| 12 | Default (no version) | No schema_version | Legacy row used (if present) |
| 13 | Configured default | Set `mosip.certify.schema-versioning.default-versions.IdentityCredential=2.0`, restart, no schema_version | v2.0 used |
| 14 | Bad configured default | Set it to 9.9 | Request fails `schema_default_version_not_found` (no fallback) |
| 15 | Missing mandatory attr | v1.1, provider omits email | `schema_validation_failed`; message has location/keyword, no values; no credential issued, no status-list slot consumed |
| 16 | Wrong type | provider returns name as number | `schema_validation_failed` |
| 17 | Pattern fail | dateOfBirth "bad" | `schema_validation_failed` |
| 18 | Unknown version | schema_version "3.0" | `schema_version_not_found` (lists available versions); never falls back |
| 19 | Invalid version in request | schema_version "abc" | `invalid_schema_version` |
| 20 | Scope mismatch | schema_version "2.0" with token scope id_scope_v1_1 | `schema_version_scope_mismatch` |
| 21 | Deprecate | GET v1.1, set schemaStatus DEPRECATED, PUT full body | 200; issue explicit 1.1 works (warning in logs); default selection no longer picks it |
| 22 | Disable | PUT status DISABLED | 200; issue 1.1 → `schema_version_disabled`; v1.1 absent from `/.well-known/openid-credential-issuer` |
| 23 | Backward status | PUT DISABLED→ACTIVE or DEPRECATED→ACTIVE | Rejected `invalid_schema_status_transition` |
| 24 | Immutability | PUT published v1.1 changing template/mapping/schema | Rejected `schema_version_immutable` |
| 25 | Delete versioned | DELETE v1.1 | Rejected |
| 26 | Delete legacy | DELETE an unversioned test config | Still allowed |
| 27 | DB guard | As DB admin, `UPDATE`/`DELETE` a versioned row directly | Blocked by trigger (SQLSTATE 23001) |
| 28 | Plugin mode | If VCIssuance plugin mode available: send schema_version | `schema_version_unsupported_in_plugin_mode` |
| 29 | Other formats | Repeat 3, 10, 15 for sd-jwt (vct) and mso_mdoc (doctype) | Same behaviour |
| 30 | Concurrency | 50 parallel issuances across versions | Each uses its requested version; no cross-version contamination |
| 31 | Verification | Verify v1.0/1.1/2.0 credentials in your verifier (inji-verify) | Verify succeeds; confirm context URL reachable. Claim 169 QR path NOT verified by author |
| 32 | Rollback | On a scratch DB with versioned rows run rollback SQL | Aborts with message; after removing versioned rows (trigger dropped deliberately) rollback succeeds and original schema restored |
| 33 | Regression | Run the existing automated test suites and API tests | No new failures |

## 5. Known limitations / things to watch
- Validator accepts some malformed schema keywords (e.g. unknown `type` name) at registration.
- PUT on a versioned row must resend structural fields unchanged.
- A configuration with only DISABLED versions yields a generic scope error (metadata hides it).
- Extra indexed DB query per issuance (check latency).
- Concurrent conflicting status updates may surface the DB trigger error.
- Pre-existing, untouched: mdoc cache eviction key mismatch.
- Jackson alignment with networknt 2.0.8 unverified: run `dependency:tree`; override `jackson-bom.version` if `NoSuchMethodError`.
- Admin endpoints may be unauthenticated in some profiles.

## 6. Exit criteria
Cases 1, 2, 10–12, 15, 18, 22–25, 33 must pass before release; any failure in 1 or 2 is a blocker.
