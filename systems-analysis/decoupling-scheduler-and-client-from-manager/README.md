# Decoupling Scheduler and Client from Manager

## Introduction

The Manager has hard dependencies on PostgreSQL and Redis, which makes it impossible to deploy
Dragonfly in a lightweight manner. In environments where users only need P2P distribution
capabilities (e.g., small Kubernetes clusters, edge environments, or CI systems), requiring a
Manager along with its PostgreSQL and Redis backends significantly increases operational cost
and deployment complexity.

This document proposes decoupling the Scheduler and Client (dfdaemon) from the Manager, so that
both components can run without a Manager. When the Manager is not configured, dynamic
configuration is loaded from a local configuration file (typically mounted as a Kubernetes
ConfigMap) instead of being fetched from the Manager.

## Goals

- Allow the Client (dfdaemon) to run without a Manager by loading dynamic configuration from a
  local file.
- Allow the Scheduler to run without a Manager by loading dynamic configuration from a local
  file and disabling Manager-dependent features (e.g., Announcer).
- Keep full backward compatibility: when the Manager is configured, the existing behavior
  remains unchanged.

## Details

## Design

### Client

#### Dynconfig

Dynconfig currently depends on the Manager to query configuration information. The code will be
restructured to support two sources — remote (Manager) and local (file) — with the following
directory layout:

```text
dragonfly-client/
  - src/
    - dynconfig/
      - mod.rs      # Dynconfig trait and source selection logic.
      - remote.rs   # Existing logic: fetch dynamic configuration from Manager.
      - local.rs    # New logic: load dynamic configuration from local dynconfig.yaml.
```

#### Configuration

By default, the Manager configuration is `None`. When it is `None`, the Client uses the
`local.rs` logic; otherwise it uses the existing `remote.rs` logic.

```rust
/// Config is the configuration for dfdaemon.
#[derive(Debug, Clone, Default, Validate, Deserialize)]
#[serde(default, rename_all = "camelCase")]
pub struct Config {
    /// Manager is the manager configuration for dfdaemon.
    #[validate]
    pub manager: Option<Manager>,
    ...
}
```

#### Local Dynconfig (`local.rs`)

When the `local.rs` logic is used:

- The Client loads a `dynconfig.yaml` configuration file (typically provided via a Kubernetes
  ConfigMap). The file is located in the same directory as `config.yaml`.
- The configuration is refreshed periodically. The default refresh interval is `5s` and is
  configurable via `refreshInterval`.
- Scheduler discovery is performed via DNS: the Client resolves the Scheduler headless service
  address (e.g., `scheduler-headless.default.svc:8002`) to obtain the list of Scheduler IPs, and
  then builds a consistent hash ring (hashring) from the resolved IP list for scheduler
  selection.

Example `dynconfig.yaml`:

```yaml
refreshInterval: 5s
scheduler:
  addr: 'scheduler-headless.default.svc:8002'
clientConfig:
  blockList:
    task:
      download:
        applications: []
        urls: []
        tags: []
        priorities: []
    persistentTask:
      upload:
        applications: []
        urls: []
        tags: []
      download:
        applications: []
        urls: []
        tags: []
        priorities: []
    persistentCacheTask:
      upload:
        applications: []
        urls: []
        tags: []
      download:
        applications: []
        urls: []
        tags: []
        priorities: []
seedClientConfig:
  blockList:
    task:
      download:
        applications: []
        urls: []
        tags: []
        priorities: []
    persistentTask:
      upload:
        applications: []
        urls: []
        tags: []
      download:
        applications: []
        urls: []
        tags: []
        priorities: []
    persistentCacheTask:
      upload:
        applications: []
        urls: []
        tags: []
      download:
        applications: []
        urls: []
        tags: []
        priorities: []
```

### Scheduler

#### Announcer

By default, the Manager configuration is `nil`. When it is `nil`, the Announcer feature is
disabled, since announcing only makes sense when a Manager is present.

```go
type Config struct {
  // Manager configuration.
  Manager *ManagerConfig `yaml:"manager" mapstructure:"manager"`
}
```

#### Dynconfig

Dynconfig currently depends on the Manager to query configuration information. The code will be
restructured with the following directory layout:

```text
scheduler/
  - config/
    - config.go           # Scheduler configuration definitions.
    - dynconfig.go        # Existing logic: fetch dynamic configuration from Manager.
    - dynconfig_local.go  # New logic: load dynamic configuration from local dynconfig.yaml.
```

When the Manager configuration is `nil`, the Scheduler uses the `dynconfig_local.go` logic;
otherwise it uses the existing `dynconfig.go` logic.

#### Local Dynconfig (`dynconfig_local.go`)

When the `dynconfig_local.go` logic is used:

- The Scheduler loads a `dynconfig.yaml` configuration file (typically provided via a Kubernetes
  ConfigMap). The file is located in the same directory as `config.yaml`.
- The configuration is refreshed periodically. The default refresh interval is `5s` and is
  configurable via `refreshInterval`.

Example `dynconfig.yaml`:

```yaml
refreshInterval: 5s
schedulerClusterConfig:
  candidateParentLimit: 5
  filterParentLimit: 10
  jobRateLimit: 10
```

## Deployment

In a Manager-less deployment on Kubernetes:

- The Scheduler is deployed with a headless service (e.g.,
  `scheduler-headless.default.svc:8002`) so that Clients can discover all Scheduler instances
  via DNS.
- The `dynconfig.yaml` for both the Client and the Scheduler is delivered as a ConfigMap and
  mounted into the same directory as the corresponding `config.yaml`.
- Updating the ConfigMap propagates the new configuration to Clients and Schedulers within one
  refresh interval (default `5s`) after kubelet syncs the mounted file.
