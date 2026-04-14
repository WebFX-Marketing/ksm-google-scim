# Dual Support Migration Plan: Pluggable Secret Providers (KSM & GCP SM)

## Overview
This document outlines the strategy for refactoring the `ksm-scim` application to support multiple secret management backends. It preserves the existing Keeper Secret Manager (KSM) functionality (maintaining 100% backward compatibility) while introducing a new provider for Google Cloud Secret Manager (GCP SM). 

This is achieved by implementing a formal "Provider" interface pattern, deciding which backend to use dynamically based on the provided environment variables or configuration files.

## Technical Strategy

### 1. Formalize the Provider Interface (`scim/provider.go`)
Define a standard interface to make the configuration loading extensible and easily testable.

```go
package scim

type SecretProvider interface {
    Load() (*ScimEndpointParameters, *GoogleEndpointParameters, error)
}
```

### 2. Update Data Structures for JSON Parsing (`scim/scim_data.go`)
Update the existing structs to include `json` tags. This allows GCP Secret Manager to inject its payload directly into the standard application structs.

Use `json.RawMessage` for the `Credentials` field to avoid the default behavior where `encoding/json` attempts to parse `[]byte` as a Base64 string. `json.RawMessage` cleanly captures the raw JSON structure required by `google.CredentialsFromJSONWithParams`.

```go
import "encoding/json"

type ScimEndpointParameters struct {
    Url         string `json:"scimUrl"`
    Token       string `json:"scimToken"`
    Verbose     bool   `json:"verbose"`
    Destructive int32  `json:"destructive"`
}

type GoogleEndpointParameters struct {
    AdminAccount string          `json:"googleAdminAccount"`
    ScimGroups   []string        `json:"scimGroups"`
    Credentials  json.RawMessage `json:"googleCredentials"` // Captures raw JSON correctly
}
```

### 3. Implement the GCP Provider (`scim/provider_gcp.go`)
Create a new file containing the GCP Secret Manager logic, implementing `SecretProvider`.

*   Read `GCP_SECRET_NAME` from the environment. Expect the full path format: `projects/PROJECT_ID/secrets/SECRET_ID/versions/VERSION`.
*   Instantiate `secretmanager.NewClient`.
*   Fetch the secret via `client.AccessSecretVersion`. Catch and wrap permission errors cleanly (e.g., "Ensure service account has Secret Accessor role").
*   Unmarshal the JSON payload into the parameter structs.
*   Return standard `*ScimEndpointParameters` and `*GoogleEndpointParameters`.

### 4. Refactor KSM Logic (`scim/provider_ksm.go`)
Move the KSM initialization (reading config base64, querying the secret, parsing records via `LoadScimParametersFromRecord`) from the entry points into a single cohesive provider that implements `SecretProvider`.

This ensures symmetry with the GCP provider and cleans up the main entry points.

### 5. Update Entry Points (`cmd/main.go` and `gcp_function.go`)
Both entry points will replace their hardcoded KSM flow with a router that selects the correct provider.

*   **`cmd/main.go` Routing**: 
    *   Check if `config.base64` exists locally or in the home directory. If so, use KSM.
    *   Else if `GCP_SECRET_NAME` is present, use GCP SM.
    *   Else throw a configuration error.
*   **`gcp_function.go` Routing**:
    *   If `KSM_CONFIG_BASE64` is present, use KSM.
    *   Else if `GCP_SECRET_NAME` is present, use GCP SM.
    *   Else throw a configuration error.

Both files will then call `provider.Load()` and continue with the standard SCIM logic.

### 6. Dependency Updates
Update `go.mod` to include the GCP Secret Manager SDK.
*   `go get cloud.google.com/go/secretmanager/apiv1`
*   Run `go mod tidy`.

### 7. Formatting and Testing
Ensure all code aligns with standard Go conventions.
*   Format the codebase using `gofmt -w .`
*   Verify everything builds cleanly with `go build ./...`
*   Run tests (if any are introduced or existing ones need to pass) using `go test ./...`

### 8. Doc Updates
* Update this file if the plan needs to change
* Update README.md as needed for setup instructions
* Update pr_draft.md as needed for accuracy

## Pre-Requisites for GCP Users
Users leveraging GCP will need to:
1.  Store their configuration as a single JSON payload in GCP Secret Manager.
2.  Grant the application Service Account the `Secret Manager Secret Accessor` role.
3.  Provide the `GCP_SECRET_NAME` environment variable formatted as `projects/*/secrets/*/versions/*`.