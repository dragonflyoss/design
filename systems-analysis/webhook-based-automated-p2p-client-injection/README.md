# Mutating Admission Webhook for Automatic P2P Injection

## Overview

This document proposes the design of a Kubernetes Mutating Admission Webhook integrated into the official Dragonfly Helm Chart. The primary function of this webhook is to automate the injection of P2P capabilities into user Pods. It intercepts Pod creation requests and, based on specific annotations, automatically modifies the Pod's YAML definition to include necessary configurations like volumes, volume mounts, and environment variables. This approach will reduce the need for manual modifications to application manifests, significantly simplifying the use of Dragonfly in a Kubernetes environment.

## Motivation

- **Simplification**: Manually configuring Pods to use Dragonfly's P2P client (`dfget`) is complex, requiring specific knowledge of volumes, mounts, and container commands.

- **Error Reduction**: Automated injection prevents common manual configuration errors, improving reliability.

- **User Experience**: Lowers the barrier to entry for developers, making the integration of Dragonfly's P2P capabilities seamless and non-intrusive.

- **Consistency**: Ensures that all designated Pods receive a standardized and correct P2P configuration.

## Goals

1. Develop a core Mutating Admission Webhook service to automate the injection of P2P proxy settings, `dfdaemon` socket mounts, and binaries such as `dfget`.

2. Provide CI support, including `Dockerfiles` for building both the webhook service container image and its associated initContainer image.

3. Integrate the webhook deployment (Deployment, Service, RBAC, MutatingWebhookConfiguration) into the official Dragonfly Helm Chart.

4. Implement comprehensive unit and E2E tests for the core injection logic.

5. Produce clear user documentation detailing how to enable and configure the webhook.

## Architecture

```mermaid
graph LR
    A[Pod Creation] --> B[Pod Webhook]
    B --> C{Check Conditions}
    C -->|Meet| D[ConfigManager]
    C -->|Not Meet| E[Skip]
    D --> F[Injector Chain]
    F --> G[Environment Variables]
    F --> H[Unix Socket]
    F --> I[CLI Tools]
    G --> J[Modify Pod]
    H --> J
    I --> J
    J --> K[Return Pod]
```

## Modules

```bash
dragonfly-injector/
├── cmd/
│   └── main.go                    # Entry point with manager and webhook server setup
├── internal/webhook/v1
│   ├── injector
│   │   ├── config.go     # Configuration management and constants
│   │   ├── proxy_env.go  # P2P proxy environment variable injection
│   │   ├── tools_initcontainer.go  # CLI tools init container injection
│   │   └── unix_socket.go  # dfdaemon Unix socket volume mounting
│   └── pod_webhook.go  # Pod webhook implementation with injection logic
├── config/   # Kubernetes manifests and configurations
├── dist/
└── test/     # Test files
```

## Implementation

### Config

```go
type InjectConf struct {
	Enable          bool   `yaml:"enable" json:"enable"`         // Whether to enable dragonfly injection
	ProxyPort       int    `yaml:"proxy_port" json:"proxy_port"` // Proxy port of dragonfly proxy(dfdaemon proxy port)
	CliToolsImage   string `yaml:"cli_tools_image" json:"cli_tools_image"`
	CliToolsDirPath string `yaml:"cli_tools_dir_path" json:"cli_tools_dir_path"`
}
```

### Server

```go
type PodCustomDefaulter struct {
	configManager *injector.ConfigManager
	kubeClient    client.Client
	injectors     []Injector
}

func (d *PodCustomDefaulter) applyDefaults(ctx context.Context, pod *corev1.Pod) {
	config := d.configManager.GetConfig()
	if config == nil || !config.Enable {
		podlog.Info("Config disabled, skip inject", "name", pod.GetName())
		return
	}
	// check if need inject
	if !d.injectRequired(ctx, pod) {
		podlog.Info("Pod not inject", "name", pod.GetName())
		return
	}
	podlog.Info("Pod inject ")
	for _, ij := range d.injectors {
		ij.Inject(pod, config)
	}
}
```

### Injector

```go
type Injector interface {
	Inject(pod *corev1.Pod, config *injector.InjectConf)
}

type ProxyEnvInjector struct {
    // proxyInjector config
    // such as proxy env ...
}

func (pei *ProxyEnvInjector) Inject(pod *corev1.Pod, config *injector.InjectConf) {
    // inject proxy env
    // ...
}

// SocketInjector
// ...

// BinaryInjector
// ...
```

## Webhook Details

1. **Annotation(label)-based Injection Scope**:
   The webhook supports injecting P2P configurations based on annotations(labels) at both the namespace and pod levels. By adding a specific label to a namespace, all pods within that namespace will have the P2P capabilities automatically injected. Additionally, pods can be annotated to enable or customize the injection. The priority of annotations is as follows: `pod-level annotations` > `namespace-level labels`.

   - Namespace injection:

     ```yaml
     apiVersion: v1
     kind: Namespace
     metadata:
       labels:
         dragonfly.io/inject: "true"
       name: test-namespace
     ```

   - Pod injection:

     ```yaml
     apiVersion: v1
     kind: Pod
     metadata:
       name: test-pod
       namespace: test-namespace
       annotations:
         dragonfly.io/inject: "true"
     spec:
       containers:
         - image: test-pod-image
           name: test-container
     ```

2. **P2P Proxy Environment Variable Injection**:
   To enable application traffic within the Pod to pass through the Dragonfly P2P network proxy, the Webhook will inject environment variables such as `DRAGONFLY_INJECT_PROXY` into the application container of the target Pod. The proxy address will be dynamically constructed, where the node name or IP can be obtained via the Downward API (`spec.nodeName` or `status.hostIP`), and the proxy port is retrieved from the Webhook configuration or Helm Chart, forming a proxy address in the form of `http://$(NODE_NAME_OR_IP):$(DRAGONFLY_PROXY_PORT)`. A sample yaml is as follows:

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: test-pod
     annotations:
       dragonfly.io/inject: "true" # webhook listens for this annotation
   spec:
     containers:
       - name: test-pod-cotainer
         image: test-pod-image:latest
         env:
           - name: NODE_NAME # Obtain the scheduled node name via Downward API
             valueFrom:
               fieldRef:
                 fieldPath: spec.nodeName
           - name: DRAGONFLY_PROXY_PORT # Port value obtained from Helm Chart
             value: "8001" # Assume the Helm Chart sets the port to 8001
           - name: DRAGONFLY_INJECT_PROXY # Concatenated proxy address
             value: "http://$(NODE_NAME):$(DRAGONFLY_PROXY_PORT)"
   ```

3. **dfdaemon Socket Volume Mounting**:
   `dfget` or other clients need to communicate with the dfdaemon daemon on the node via a Unix Domain Socket. The Webhook will automatically add a hostPath Volume to the Pod to expose the Socket file based on the configuration (default is `/var/run/dfdaemon.sock`) and add the corresponding VolumeMount in the target container to ensure the client can access the Socket. A sample yaml is as follows:

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: test-app-with-dfdaemon-socket
     annotations:
       dragonfly.io/inject: "true" # Annotation to trigger the Webhook
   spec:
     containers:
       - name: test-app-container
         image: test-app-image:latest
         volumeMounts:
           - name: dfdaemon-socket
             mountPath: /var/run/dfdaemon.sock # Path to dfdaemon socket inside the container
     volumes:
       - name: dfdaemon-socket
         hostPath:
           path: /var/run/dfdaemon.sock # Actual path to dfdaemon socket on the node
           type: Socket
   ```

4. **Cli Tool Injection**:
   Considering that many base container images do not include the cli tool (such as `dfget`), and manual installation is inconvenient, this project will solve this problem using an Init Container. The Webhook will automatically add an initContainer to the target Pod. This initContainer is a custom lightweight image available in both amd64 and arm64 architectures, each containing the corresponding architecture's cli tool. The Webhook will also copy the cli tool from this initContainer to a shared volume. Subsequently, the Webhook add the `DRAGONFLY_TOOLS_PATH` environment variable of the application container to add the shared volume directory where cli is located, allowing the application container to execute cli commands directly from the command line without additional user installation or specifying the full path.

   The InitContainer uses Docker's manifest list to achieve the function of automatically importing the corresponding architecture initContainer, and its build commands are as follows:

   ```bash
   # Create manifest
   docker manifest create dragonflyoss/toolkits:latest \
   dragonflyoss/toolkits-amd64-linux:latest \
   dragonflyoss/toolkits-arm64-linux:latest

   docker manifest annotate dragonflyoss/toolkits-amd64-linux:latest --arch amd64 --os linux
   docker manifest annotate dragonflyoss/toolkits-arm64-linux:latest --arch arm64 --os linux
   docker manifest push dragonflyoss/toolkits:latest
   ```

   Sample yaml for the injected pod:

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: test-app-with-cli-tools-image
     annotations:
       dragonfly.io/inject: "true" # Annotation to trigger the Webhook
       # The image and version fields only need to be added if you want to specify non-default values.
       dragonfly.io/cli-tools-image: "dragonflyoss/toolkits:v0.0.1"
   spec:
    containers:
    - command:
      - sh
      - -c
      - sleep 3600
      env:
      - name: NODE_NAME
        valueFrom:
          fieldRef:
            apiVersion: v1
            fieldPath: spec.nodeName
      - name: DRAGONFLY_PROXY_PORT
        value: "4001"
      - name: DRAGONFLY_INJECT_PROXY
        value: http://$(NODE_NAME):$(DRAGONFLY_PROXY_PORT)
      - name: DRAGONFLY_TOOLS_PATH
        value: /dragonfly-tools-mount
      image: busybox:latest
    initContainers:
    - command:
      - cp
      - -rf
      - /dragonfly-tools/.
      - /dragonfly-tools-mount/
      image: dragonflyoss/toolkits:latest
      imagePullPolicy: IfNotPresent
      name: d7y-cli-tools
      volumeMounts:
      - mountPath: /dragonfly-tools-mount
        name: d7y-cli-tools-volume
       containers:
         - name: test-app-container
           image: test-app-image:latest
           env:
             - name: DRAGONFLY_TOOLS_PATH
               value: "/dragonfly-tools-mount"
           volumeMounts:
             - name: dragonfly-tools-volume
               mountPath: /dragonfly-tools-mount
     volumes:
       - name: dragonfly-tools-volume
         emptyDir: {}
   ```

## Testing

1. **Unit Tests**: Core injection logic, annotation parsing, and patch generation will be covered by unit tests to ensure correctness.

2. **Integration & E2E Testing**: End-to-end test scripts will be developed to verify the complete workflow, from Pod creation with annotations to the successful mutation and deployment of the Pod with all injected configurations.
3. **Stability Testing**: Evaluate the webhook's resilience and stability under concurrent Pod creation requests to ensure consistent behavior during peak workloads.

## Compatibility

- **Backward Compatibility**: The webhook is disabled by default and only acts on Pods that are explicitly annotated. It will have no impact on existing Dragonfly installations or other workloads.

- **Opt-In Model**: Functionality is strictly opt-in via Pod annotations, giving users full control over which workloads are affected.

- **Configurability**: Key parameters, such as socket paths and proxy ports, will be configurable through the Helm Chart to adapt to different environments.

## Future

- **Extended Binary Injection**: Expand the injection mechanism to include other Dragonfly tools, such as `dfcache`, to provide a richer set of P2P capabilities.

- **Sidecar Container Investigation**: Evaluate using a sidecar container as an alternative to the Init Container for more advanced tool lifecycle management.

- **Granular Annotation Control**: Enhance annotations to allow for more fine-grained control over the injected configurations, such as resource limits or specific command-line flags for the tools.
