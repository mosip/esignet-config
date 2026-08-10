# AGENTS.md

## Repository Overview

This repository holds the default configuration files for the eSignet family
of MOSIP services: the eSignet authorization server itself, the eSignet
Signup service, and the Mock Identity System used for local/non-production
testing. It is a flat data/config repository, not application source code —
there is no build toolchain, no source directories, and no tests here. The
files are served to running services through a Spring Cloud Config Server
(the same pattern used across MOSIP's `*-config` repositories): each service
reads its `${spring_config_label_env}` branch of this repo at startup to
pick up its `.properties` file.

## Technology Stack

- Java `.properties` files (Spring Boot property syntax, including
  `${...}` placeholders resolved by the config server or by the consuming
  service at runtime).
- JSON files for structured configuration (auth-method mappings, mock
  identity-verification story scripts, identity-verifier metadata, and a
  JSON Schema).
- No compiled code, no package manager, no build tool. There is nothing to
  install and nothing to compile in this repository.

## Build & Test Commands

None. This is config only — there is no build step and no test suite.
The only meaningful validation is:

- JSON files must parse as valid JSON (check with any JSON validator/linter
  before committing, e.g. `python -m json.tool <file>` or a Node-based
  linter).
- `.properties` files must keep valid Spring property syntax — in
  particular, multi-line values use a trailing `\` to continue onto the
  next line, and nested curly-brace map/list literals must stay balanced.
- Changes only take effect once the consuming service (eSignet, eSignet
  Signup, or Mock Identity System) is restarted or reloads the config
  server on the branch/label it is pointed at. There is no CI in this repo
  today (no `.github/workflows` directory exists on `develop`).

## Configuration

Root-level files on `develop` (verified via `git ls-tree`):

```text
.gitignore
LICENSE
README.md
amr-acr-mapping.json
esignet-default.properties
mock-identity-system-default.properties
mock-idv-user-story.json
signup-default.properties
signup-identity-verifier-details.json
signup-idv_mock-identity-verifier.json
verified_claims_request_schema.json
```

Naming convention: `<service>-default.properties` holds the base Spring
Boot configuration for one service. JSON files are named after the data
they describe (`signup-idv_mock-identity-verifier.json`,
`mock-idv-user-story.json`, etc.) rather than following a single fixed
pattern — match an existing file's naming style when adding a sibling file
for the same service.

**Secrets are never stored as literal values in this repository.** Both
`esignet-default.properties` and `mock-identity-system-default.properties`
carry a header comment listing the properties that must be supplied as
environment variables on the config-server Helm deployment instead of
being hardcoded here, including:

- `db.dbuser.password`
- `keycloak.external.url`, `keycloak.internal.host`, `keycloak.internal.url`, `keycloak.admin.password`
- `mosip.ida.client.secret`, `mosip.admin.client.secret`, `mosip.reg.client.secret`, `mosip.prereg.client.secret`
- `softhsm.kernel.pin`, `softhsm-security-pin`
- `email.smtp.host`, `email.smtp.username`, `email.smtp.secret`
- `mosip.kernel.tokenid.uin.salt`, `mosip.kernel.tokenid.partnercode.salt`
- `mosip.api.internal.url`, `mosip.api.public.url`

In the properties files themselves this shows up as a placeholder, e.g.
`spring.datasource.password=${db.dbuser.password}` or
`mosip.kernel.keymanager.hsm.keystore-pass=${softhsm.esignet.security.pin}`
— never a literal password or key. When adding a new property that needs a
secret, follow the same placeholder pattern and add the variable name to
the relevant header comment block.

## Project Structure Notes

Files are grouped by which service consumes them:

- **eSignet (authorization server)** — `esignet-default.properties`
  (OAuth/OIDC endpoints, auth-challenge formats, cache, Kafka, key-manager,
  UI config map), `amr-acr-mapping.json` (AMR/ACR authentication-method
  mapping consumed via
  `mosip.esignet.amr-acr-mapping-file-url`), `verified_claims_request_schema.json`
  (JSON Schema for OIDC verified-claims requests).
- **Mock Identity System** — `mock-identity-system-default.properties`
  (a stand-in IDA used for local/demo deployments instead of the real
  MOSIP ID Authentication service).
- **eSignet Signup** — `signup-default.properties` (challenge/OTP,
  identity-verification, notification templates, UI config map),
  `signup-identity-verifier-details.json` (list of available identity
  verifiers, e.g. the mock verifier), `signup-idv_mock-identity-verifier.json`
  (mock verifier's terms/consent content), `mock-idv-user-story.json`
  (scripted frame-by-frame liveness/IDV story used by the mock verifier).

`README.md` documents this same grouping with direct links; keep it in
sync when files are added, renamed, or removed.

## Development Workflow

1. Fork and clone this repository; add `upstream` pointing at
   `https://github.com/mosip/esignet-config.git`.
2. Branch from `upstream/develop` — `develop` is the active integration
   branch for this repo (not `master`, which is the reported GitHub
   default branch).
3. Edit the relevant `.properties`/`.json` file directly. Keep changes
   scoped to one service/concern per PR where possible.
4. Validate JSON syntax and re-check any Spring property placeholders you
   touched still resolve to values defined either elsewhere in the same
   file or via the environment-variable list described above.
5. Commit with sign-off (`git commit -s`) and push to your fork, then open
   a PR against `mosip/esignet-config`'s `develop` branch.

## Pull Request Guidelines

- Target `develop`, not `master`.
- Reference the tracking issue in the PR description.
- Sign off commits (`git commit -s`) per MOSIP's contribution requirements.
- Never introduce a literal secret, password, or private key value — use a
  `${...}` placeholder and document the required environment variable in
  the file's header comment if it's new.
- Keep the change minimal and targeted; this repo has no automated CI
  checks today, so careful manual review of property syntax and JSON
  validity before opening the PR matters more than usual.

## Repository-Specific Considerations

- This repo is one of several MOSIP `*-config` repositories (compare with
  `mosip/mosip-config`); the same "config server serves this branch to
  running services" model applies here.
- Property files reference other MOSIP services by convention
  (`${mosip.esignet.host}`, `${mosip.signup.host}`, `${mosip.api.internal.url}`,
  etc.) — these are resolved by the config server's environment, not by
  anything in this repo, so a value that looks "unset" here is often
  intentional.
- `signup-default.properties` currently ships Khmer (`khm`) as the default
  language and a Cambodia-specific phone-number regex
  (`mosip.signup.identifier.regex=^\+855[1-9]\d{7,8}$`) — this is a
  deployment-specific default, not a hardcoded platform constraint. Treat
  it as an example to override per-deployment rather than a value to
  "correct."
- There are no Helm charts, deploy scripts, or CI workflow files in this
  repository (confirmed via `git ls-tree -r develop` and absence of
  `.github/workflows`); deployment plumbing (config-server Helm chart,
  environment variable injection) lives in other MOSIP repositories.

## Agent rules — Do / Do not

### Do

1. Verify the actual file list and file contents in this repo before
   describing or changing them — do not assume based on other MOSIP
   `*-config` repos.
2. Use `${...}` placeholders for any secret, password, or key value, and
   document new required environment variables in the appropriate header
   comment.
3. Branch from and target `develop` for all changes and PRs.
4. Validate JSON files for syntax correctness before committing.
5. Keep new files consistent with the existing per-service naming and
   grouping convention, and update `README.md` when adding/removing files.

### Do not

1. Do not commit literal secrets, passwords, private keys, or tokens into
   any `.properties` or `.json` file.
2. Do not target `master` for branches or PRs — it is not the active
   integration branch for this repo.
3. Do not invent build/test commands — this repo has none; do not add a
   build toolchain unless explicitly requested.
4. Do not remove or alter the environment-variable-override header
   comments in `esignet-default.properties` or
   `mock-identity-system-default.properties` without also updating every
   property that relies on them.
5. Do not assume CI validates your change — there is no
   `.github/workflows` directory on `develop` today, so manual review of
   syntax and placeholders is required before merging.
