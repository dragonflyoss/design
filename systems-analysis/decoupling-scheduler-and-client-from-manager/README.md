# Decoupling Scheduler and Client from Manager

## Introduction

The Manager has hard dependencies on PostgreSQL and Redis, which makes it impossible to deploy
Dragonfly in a lightweight manner. In environments where users only need P2P distribution
capabilities (e.g., small Kubernetes clusters, edge environments, or CI systems), requiring a
Manager along with its PostgreSQL and Redis backends significantly increases operational cost
and deployment complexity.

This document proposes decoupling the Scheduler and Client (dfdaemon) from the Manager, so that
both components can run without a Manager. When the manager address is not configured, dynamic
configuration is loaded from a local configuration file (typically mounted as a Kubernetes
ConfigMap) instead of being fetched from the Manager.

## Goals

- Allow the Client (dfdaemon) to run without a Manager by loading dynamic configuration from a
  local file.
- Allow the Scheduler to run without a Manager by loading dynamic configuration from a local
  file and disabling Manager-dependent features (e.g., Announcer).
- Keep full backward compatibility: when the manager address is configured, the existing
  behavior remains unchanged.

## Design

### Client

#### Dynconfig

Dynconfig currently depends on the Manager to query configuration information. The code is
restructured to support two backends — remote (Manager) and local (file) — with the following
directory layout:

```text
dragonfly-client/
  - src/
    - dynconfig/
      - mod.rs      # Backend selection and periodic refresh loop.
      - remote.rs   # Existing logic: fetch dynamic configuration from Manager.
      - local.rs    # New logic: load dynamic configuration from local dynconfig.yaml.
```

#### Configuration

The manager configuration is always present, but its address is optional. When
`manager.addr` is not configured, the Client uses the `local.rs` logic; otherwise it uses the
existing `remote.rs` logic.

```rust
/// Config is the configuration for dfdaemon.
#[derive(Debug, Clone, Default, Validate, Deserialize)]
#[serde(default, rename_all = "camelCase")]
pub struct Config {
    /// The manager configuration for dfdaemon.
    #[validate]
    pub manager: Manager,
    ...
}

/// Manager is the manager configuration for dfdaemon.
#[derive(Debug, Clone, Default, Validate, Deserialize)]
#[serde(default, rename_all = "camelCase")]
pub struct Manager {
    /// The manager address. If not configured, the dynamic configuration is
    /// loaded from the local dynconfig file instead of being fetched from the
    /// manager.
    pub addr: Option<String>,
    ...
}
```

#### Local Dynconfig (`local.rs`)

When the `local.rs` logic is used:

- The Client loads a `dynconfig.yaml` configuration file (typically provided via a Kubernetes
  ConfigMap). The default path is `dynconfig.yaml` in the same directory as `dfdaemon.yaml`,
  and can be overridden with the `--dynconfig` flag or the `DFDAEMON_DYNCONFIG` environment
  variable. If the file does not exist, it is generated with default values on startup.
- The configuration is refreshed periodically, reusing the existing
  `dynconfig.refreshInterval` from `dfdaemon.yaml` (default `60s`). The dynconfig file itself
  does not carry a refresh interval.
- Scheduler discovery supports two modes: if the static address list `scheduler.addrs` (e.g.,
  `192.168.1.10:8002`) is non-empty, it takes precedence; otherwise the Client resolves the
  Scheduler headless service address `scheduler.addr` (e.g.,
  `scheduler-headless.default.svc:8002`) via DNS to obtain the list of Scheduler IPs. The
  discovered addresses are sorted and deduplicated to keep scheduler selection stable across
  refreshes, and the Client then builds a consistent hash ring (hashring) from the list for
  scheduler selection.
- Each discovered scheduler is health-checked, and unhealthy schedulers are filtered out. The
  refresh fails if no healthy scheduler is available.

Example `dynconfig.yaml`:

```yaml
scheduler:
  addr: 'scheduler-headless.default.svc:8002'
  # addrs takes precedence over addr when non-empty.
  # addrs: ['192.168.1.10:8002', '192.168.1.11:8002']
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

The manager configuration is always present, but its address is optional. When `manager.addr`
is not configured, the manager client is not initialized and the Announcer feature is disabled,
since announcing only makes sense when a Manager is present.

```go
type Config struct {
	// Manager configuration. If the manager address is not configured, the
	// scheduler runs without a manager, and the dynamic configuration is
	// loaded from the local file.
	Manager ManagerConfig `yaml:"manager" mapstructure:"manager"`
	...
}

type ManagerConfig struct {
	// Addr is manager address.
	*Addr string `yaml:"addr" mapstructure:"addr"`
	...
}
```

#### Dynconfig

Dynconfig currently depends on the Manager to query configuration information. The code is
restructured with the following directory layout:

```text
scheduler/
  - config/
    - config.go            # Scheduler configuration definitions.
    - dynconfig.go         # Dynconfig interface and backend selection logic.
    - dynconfig_remote.go  # Existing logic: fetch dynamic configuration from Manager.
    - dynconfig_local.go   # New logic: load dynamic configuration from local dynconfig.yaml.
```

When `manager.addr` is not configured, the Scheduler uses the `dynconfig_local.go` logic;
otherwise it uses the existing `dynconfig_remote.go` logic.

#### Local Dynconfig (`dynconfig_local.go`)

When the `dynconfig_local.go` logic is used:

- The Scheduler loads a `dynconfig.yaml` configuration file (typically provided via a
  Kubernetes ConfigMap). The default path is `dynconfig.yaml` in the same directory as
  `scheduler.yaml`, and can be overridden with the `--dynconfig` flag. If the file does not
  exist, it is generated with default values on startup.
- The configuration is refreshed periodically, reusing the existing
  `dynConfig.refreshInterval` from `scheduler.yaml` (default `1m`). The dynconfig file itself
  does not carry a refresh interval.
- The file provides the applications, seed peer cluster, scheduler cluster, and client
  configurations that would otherwise be fetched from the Manager.

Example `dynconfig.yaml`:

```yaml
applications: []
seedPeerClusterConfig:
  loadLimit: 2000
schedulerClusterConfig:
  candidateParentLimit: 3
  filterParentLimit: 15
schedulerClusterClientConfig:
  loadLimit: 200
```

## Deployment

In a Manager-less deployment on Kubernetes:

- The Scheduler is deployed with a headless service (e.g.,
  `scheduler-headless.default.svc:8002`) so that Clients can discover all Scheduler instances
  via DNS, or a static scheduler address list is set via `scheduler.addrs`.
- The `dynconfig.yaml` for both the Client and the Scheduler is delivered as a ConfigMap and
  mounted into the same directory as the corresponding `config.yaml`.
- Updating the ConfigMap propagates the new configuration to Clients and Schedulers within one
  refresh interval (`dynconfig.refreshInterval` for the Client and `dynConfig.refreshInterval`
  for the Scheduler, default `60s`) after kubelet syncs the mounted file.
