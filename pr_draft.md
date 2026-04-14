# Draft Pull Request Message

**Title:** Refactor: Introduce pluggable secret provider architecture (Add GCP Secret Manager support)

**Description:**

Hi Keeper Team,

This PR introduces a pluggable configuration architecture to `ksm-scim`, allowing the tool to fetch its configuration from different secret management backends. 

While Keeper Secret Manager (KSM) remains the primary and default method, this update adds support for Google Cloud Secret Manager (GCP SM) to facilitate deployments in strictly GCP-native environments where external secret managers might face compliance or architectural hurdles.

**Key Changes:**
1. **Zero Breaking Changes:** 100% backward compatibility for existing users. The CLI prioritizes the `config.base64` file, and the Cloud Function prioritizes the `KSM_CONFIG_BASE64` environment variable.
2. **Provider Pattern:** Abstracted the configuration loading phase behind a new `SecretProvider` interface. The `cmd/main.go` and `gcp_function.go` entry points now route to the appropriate provider based on configuration presence.
3. **Symmetry & Cleanup:** Extracted KSM loading logic from the entry points into `scim/provider_ksm.go` for cleaner separation of concerns.
4. **GCP Implementation:** Added `scim/provider_gcp.go` which allows fetching configuration from a unified JSON secret if `GCP_SECRET_NAME` is set. Catch-blocks ensure clear error messaging for missing IAM permissions.
5. **Struct Annotations:** Added standard `json` tags to the structs in `scim_data.go` to facilitate native mapping from GCP. Utilized `json.RawMessage` for credentials to prevent Base64 deserialization issues, requiring zero changes to the core SCIM synchronization engine.
6. **Tooling:** Ensured code is correctly formatted via `gofmt` and passes standard `go build` / `go test` checks.

**Testing:**
* Verified GCP flow works as expected
* Clean build and format check

We understand Keeper's core focus is on KSM, so we ensured this addition is entirely non-intrusive. Happy to discuss any structural changes you’d prefer to see to align with your roadmap!

Best regards,
Chris
