# Plan: Add AKS Identity Bindings

Add opt-in AKS identity binding support to `WorkloadIdentityCredential`, matching the cross-language Azure Identity contract. The credential will read the AKS proxy configuration only when `WorkloadIdentityCredentialOptions::enable_azure_proxy` is true. Normal workload identity remains unchanged, and no aggregate/default credential is added or changed because Rust intentionally removed `DefaultAzureCredential` and AKS does not support identity bindings through aggregate credentials.

## Phase 1: Proxy Transport and Configuration

1. Add the HTTP/TLS dependencies and crate feature wiring needed by `azure_identity` to construct a private proxy-capable transport. Keep the dependency aligned with the repository's existing reqwest 0.13/rustls stack, include the feature in the crate's defaults, and ensure proxy-disabled builds and behavior remain unchanged.
2. Create a private `custom_token_proxy` module in `azure_identity` that owns the environment-variable contract: `AZURE_KUBERNETES_TOKEN_PROXY`, `AZURE_KUBERNETES_SNI_NAME`, `AZURE_KUBERNETES_CA_FILE`, and `AZURE_KUBERNETES_CA_DATA`.
3. In that module, parse and validate proxy configuration at credential construction time: the proxy URL must be HTTPS and contain no user info, query, or fragment; auxiliary SNI/CA variables without a proxy are invalid when proxy mode is enabled; CA file and inline CA data are mutually exclusive; no CA override means system roots. Return credential/data-conversion errors that identify the invalid variable without exposing certificate contents.
4. Implement a private `HttpClient`/transport wrapper that receives requests after existing pipeline policies, appends each original request path to the validated proxy base path, preserves the original query/body/headers, and sends through the proxy TLS client. For custom SNI, resolve the proxy endpoint host, rewrite the outbound TLS URL host to the configured SNI name, pin that name to the resolved proxy socket addresses with reqwest's resolver override, and set the HTTP `Host` header back to the proxy authority. This keeps TCP routing on the Kubernetes service endpoint while presenting the cluster-specific TLS server name.
5. Load and validate inline PEM CA data once. For file-based CA configuration, cache the parsed CA/client and re-read on the same 10-minute cadence used by workload token files; serialize refreshes for concurrent requests, retain or cleanly replace the cached client, and surface read/parse failures. Build with system roots plus the supplied CA override, matching the Go behavior, and do not mutate the caller's original `ClientOptions` value.

## Phase 2: Credential Integration

6. Add public `enable_azure_proxy: bool` to `WorkloadIdentityCredentialOptions`, defaulting to false and documented with the AKS identity bindings link and explicit opt-in semantics. Include the flag, but no secret proxy configuration, in `Debug` only if consistent with nearby option debugging.
7. In `WorkloadIdentityCredential::new`, clone or move the nested `ClientAssertionCredentialOptions`, and only when the flag is true, configure its `ClientOptions.transport` with the private proxy transport before constructing `ClientAssertionCredential`. When false, ignore all AKS proxy environment variables so existing direct-Entra behavior and custom caller transports remain unchanged.
8. Register the private module in `lib.rs`; keep all proxy implementation types private so the only new public API is `WorkloadIdentityCredentialOptions::enable_azure_proxy`.

## Phase 3: Tests and Documentation

9. Add configuration unit tests for no proxy settings, minimal proxy settings, invalid URL schemes/components, auxiliary settings without a proxy, mutually exclusive CA sources, missing/invalid CA files, invalid inline PEM, and successful CA file/data parsing. Use the existing test-only `Env` injection instead of mutating process-wide environment where possible.
10. Add request-flow tests proving opt-in redirects the token POST, joins proxy and Entra token paths correctly, including proxy base paths and escaped path cases, preserves query/body/headers and caller policies, uses the service-account assertion, and does not mutate the caller's `ClientOptions`.
11. Add TLS-focused tests with a local HTTPS endpoint and test CA covering inline CA, CA file, custom SNI, system-root/no-override configuration where testable, concurrent requests, CA cache reuse, and CA file rotation after forced cache expiry. Keep cache timing injectable or directly controllable in tests rather than sleeping ten minutes.
12. Add regression coverage that `enable_azure_proxy == false` ignores even malformed proxy environment variables and continues using the explicitly supplied mock Entra transport. This is the Rust equivalent of the Go `DefaultAzureCredential` protection, without adding a removed aggregate credential.
13. Add a concise `Features Added` entry to the current unreleased `azure_identity` changelog and update `WorkloadIdentityCredential` rustdocs, and troubleshooting text if new configuration errors need mitigation guidance, with the exact option and environment variables. Do not claim `DefaultAzureCredential` support.

## Relevant Files

- `Cargo.toml`: reuse the workspace reqwest version and add any TLS helper dependencies centrally if required.
- `sdk/identity/azure_identity/Cargo.toml`: wire proxy transport dependencies/features without disturbing existing `client_certificate` or `tokio` features.
- `sdk/identity/azure_identity/src/custom_token_proxy.rs`: add private environment parsing, validation, CA cache, URL rewriting, TLS transport, and focused unit tests.
- `sdk/identity/azure_identity/src/workload_identity_credential.rs`: add `enable_azure_proxy`, configure the nested client options, and add credential-level regression/flow tests; reuse the existing `Token`/`FileCache` locking pattern.
- `sdk/identity/azure_identity/src/lib.rs`: register the private module only.
- `sdk/identity/azure_identity/CHANGELOG.md`: document the public feature under the existing unreleased `Features Added` heading.
- `sdk/identity/azure_identity/TROUBLESHOOTING.md`: add actionable proxy configuration errors only if the implementation introduces user-facing cases not adequately covered by constructor messages.

## Verification

1. Run `cargo fmt -p azure_identity --check` after formatting.
2. Run `cargo test -p azure_identity --all-features`, including proxy configuration, URL rewrite, TLS/SNI, concurrency, rotation, and opt-out regression tests.
3. Run `cargo clippy -p azure_identity --all-features --all-targets` and resolve all warnings.
4. Run `cargo check -p azure_identity --no-default-features` to ensure the existing runtime-independent configuration still compiles. If proxy support is feature-gated, also check the minimal documented feature combination that enables it.
5. Run `cargo test -p azure_identity --doc --all-features` for the updated public option documentation.
6. Manually inspect the public API diff to confirm only `WorkloadIdentityCredentialOptions::enable_azure_proxy` is added and no `DefaultAzureCredential`/aggregate chain is introduced.

## Decisions

- Identity bindings are supported only through direct `WorkloadIdentityCredential` usage, per user confirmation and AKS documentation.
- Proxy mode is explicitly opt-in and defaults to false; proxy-related environment variables are ignored when disabled.
- Use the final cross-language environment variable name `AZURE_KUBERNETES_TOKEN_PROXY`, not the early issue's `AZURE_KUBERNETES_TOKEN_ENDPOINT` wording.
- Support both `AZURE_KUBERNETES_CA_FILE` and `AZURE_KUBERNETES_CA_DATA`, mutually exclusively; allow neither to use system roots.
- Preserve caller pipeline policies but replace or wrap the final transport only inside the credential's cloned options.
- Exclude restoration of `DefaultAzureCredential`, changes to `DeveloperToolsCredential`, managed identity behavior, live AKS resource provisioning, and generated code.
