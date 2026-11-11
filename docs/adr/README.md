# Architecture decision records

Why flock is built the way it is. The format, the naming and the superseding rule are recorded
in [0001](0001-record-architecture-decisions.md).

| Record                                                                                                                                                                                                    | Status   |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- |
| [0001. Record architecture decisions](0001-record-architecture-decisions.md)                                                                                                                              | Accepted |
| [0002. An Account bundles receiving and sending; Identities belong to one Account](0002-an-account-bundles-receiving-and-sending.md)                                                                      | Proposed |
| [0003. One binary, two runtime modes: Local mode serves only the desktop app, Server mode serves browsers](0003-one-binary-two-runtime-modes.md)                                                          | Proposed |
| [0004. A public, stable REST API, code-first OpenAPI via utoipa, in Server mode only](0004-a-public-stable-rest-api-code-first-openapi-in-server-mode.md)                                                 | Proposed |
| [0005. One Rust crate with feature and platform modules; Cargo and Bun workspaces side by side](0005-one-rust-crate-with-feature-and-platform-modules.md)                                                 | Proposed |
| [0006. Credentials under envelope encryption with an operator-held master key; an optional Master password in Local mode](0006-credentials-under-envelope-encryption-with-an-operator-held-master-key.md) | Proposed |
