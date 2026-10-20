# ConfigMaps and Secrets

## What Are ConfigMaps and Secrets?

Applications often need configuration values such as environment names, application settings, URLs, and credentials.

Kubernetes provides two resources for managing this information:

- **ConfigMap:** Stores non-confidential configuration data.
- **Secret:** Stores sensitive data, such as passwords, tokens, and credentials.

These resources help separate application configuration from the container image and Pod specification.

For example, an application image can remain unchanged across development, testing, and production while each environment supplies different configuration values.

--------------------------------------------------------------------

## Why Do We Need Them?

Without ConfigMaps and Secrets, configuration values may be hard-coded into application images or repeated directly in Pod manifests.

This makes configuration harder to maintain and can expose sensitive information unnecessarily.

ConfigMaps and Secrets help with:

- Separating configuration from application code.
- Reusing configuration across multiple workloads.
- Supplying configuration through environment variables.
- Mounting configuration as files inside containers.
- Managing sensitive values separately from ordinary settings.

**Important:** A ConfigMap is intended for non-sensitive configuration. A Secret is designed for sensitive values, but a Secret is not automatically secure merely because it is stored as a Kubernetes Secret.

--------------------------------------------------------------------

## What Is a ConfigMap?

A **ConfigMap** stores non-confidential configuration data as key-value pairs.

Examples include:

- Application environment names.
- Feature flags.
- Logging levels.
- Non-sensitive application settings.
- Configuration files.

### Example ConfigMap YAML

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "development"
  LOG_LEVEL: "info"
  API_ENDPOINT: "http://backend-service"
```

### Explanation

- `apiVersion: v1` identifies the core Kubernetes API version.
- `kind: ConfigMap` declares the resource type.
- `metadata.name` identifies the ConfigMap.
- `data` contains non-confidential configuration values.

In this example, `APP_ENV`, `LOG_LEVEL`, and `API_ENDPOINT` are configuration keys.

Do not put passwords, API tokens, or other confidential values in a ConfigMap.

--------------------------------------------------------------------

## Creating and Inspecting ConfigMaps

### Create a ConfigMap from a Manifest

```bash
kubectl apply -f configmap.yaml
```

### List ConfigMaps

```bash
kubectl get configmaps
```

The shorter form is:

```bash
kubectl get cm
```

### Inspect a ConfigMap

```bash
kubectl describe configmap app-config
```

### Display Its YAML

```bash
kubectl get configmap app-config -o yaml
```

These commands help verify that the ConfigMap exists and contains the expected configuration values.

--------------------------------------------------------------------

## What Is a Secret?

A **Secret** is a Kubernetes resource designed to hold sensitive information, such as:

- Passwords.
- API tokens.
- Authentication credentials.
- Private keys and certificates.

Secrets allow sensitive values to be supplied to workloads without embedding them directly in the application image.

### Example Secret YAML

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
stringData:
  DB_USERNAME: "appuser"
  DB_PASSWORD: "replace-with-a-real-secret"
```

### Explanation

- `apiVersion: v1` identifies the core Kubernetes API version.
- `kind: Secret` declares the resource type.
- `metadata.name` identifies the Secret.
- `type: Opaque` identifies a general-purpose Secret.
- `stringData` allows plain-text input that Kubernetes converts into the Secret's stored `data` representation.

The password in this example is only a placeholder. Do not commit real credentials to Git or paste them into shared documentation.

**Important:** `stringData` is convenient for creating a Secret, but the manifest itself contains plain-text values. Protect the file and avoid storing real credentials in source control.

--------------------------------------------------------------------

## Base64 Encoding Is Not Encryption

Secret values displayed in the `data` field are Base64-encoded.

Base64 is an encoding format, not an encryption mechanism. Anyone who obtains the encoded value can decode it.

For example:

```bash
echo -n 'example' | base64
```

To decode it:

```bash
echo -n 'ZXhhbXBsZQ==' | base64 --decode
```

These commands use harmless example text. Do not use real passwords in terminal commands that may be retained in shell history.

Kubernetes Secret security depends on access controls and cluster configuration. For stronger protection, use appropriate RBAC permissions, enable encryption at rest where required, restrict access to etcd and backups, and consider an external secrets manager.

--------------------------------------------------------------------

## Creating and Inspecting Secrets

### Create a Secret from a Manifest

```bash
kubectl apply -f secret.yaml
```

### List Secrets

```bash
kubectl get secrets
```

### Inspect Secret Metadata

```bash
kubectl describe secret app-secret
```

### Display a Secret's YAML

```bash
kubectl get secret app-secret -o yaml
```

The YAML output can expose encoded sensitive values. Avoid sharing it or printing it unnecessarily.

To inspect one value, Base64-decode the relevant field only when authorized and necessary. Remember that decoding does not decrypt the value.

--------------------------------------------------------------------

## Using a ConfigMap as Environment Variables

A Pod can load all key-value pairs from a ConfigMap into its container environment using `envFrom`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-demo
spec:
  containers:
    - name: app
      image: busybox:stable
      command: ["sh", "-c", "env; sleep 3600"]
      envFrom:
        - configMapRef:
            name: app-config
```

In this example:

- `envFrom` imports configuration values as environment variables.
- `configMapRef.name` identifies the ConfigMap.
- The container can read these values through its environment.

This example assumes the `app-config` ConfigMap already exists in the same namespace as the Pod.

--------------------------------------------------------------------

## Using Individual ConfigMap Values

You can inject a specific key using `configMapKeyRef`.

```yaml
env:
  - name: APP_ENV
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: APP_ENV
```

This makes the ConfigMap key `APP_ENV` available to the container under the environment variable name `APP_ENV`.

Use this method when an application needs only selected configuration values.

--------------------------------------------------------------------

## Using a Secret as Environment Variables

A Pod can obtain a specific environment variable from a Secret using `secretKeyRef`.

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: app-secret
        key: DB_PASSWORD
```

Here:

- `secretKeyRef.name` identifies the Secret.
- `secretKeyRef.key` identifies the key containing the sensitive value.
- The resulting environment variable is named `DB_PASSWORD`.

The Secret must exist in the same namespace as the Pod.

Avoid printing secret environment variables in application logs or diagnostic output.

--------------------------------------------------------------------

## Mounting ConfigMaps and Secrets as Files

Configuration can also be mounted into a container as files.

### Mount a ConfigMap

```yaml
volumes:
  - name: config-volume
    configMap:
      name: app-config

containers:
  - name: app
    image: nginx:stable
    volumeMounts:
      - name: config-volume
        mountPath: /etc/app-config
        readOnly: true
```

Each ConfigMap key is made available as a file in the mounted directory.

For example, the `APP_ENV` key can appear as a file named `APP_ENV`.

### Mount a Secret

```yaml
volumes:
  - name: secret-volume
    secret:
      secretName: app-secret

containers:
  - name: app
    image: nginx:stable
    volumeMounts:
      - name: secret-volume
        mountPath: /etc/app-secret
        readOnly: true
```

Secret keys can be made available as files under `/etc/app-secret`.

These are illustrative Pod-specification fragments. In a complete manifest, `volumes` belongs under the Pod specification and `volumeMounts` belongs inside the relevant container specification.

Restrict file permissions and access according to the application's security requirements.

--------------------------------------------------------------------

## Environment Variables Versus Mounted Files

| Method | How configuration is delivered | Important consideration |
|---|---|---|
| Environment variables | Values are injected into the container environment. | Existing environment variables do not automatically update when the ConfigMap or Secret changes. |
| Mounted files | Keys are exposed as files in a mounted directory. | Updates may become visible after a delay; the application may need to reload the files. |

Environment variables are useful for simple configuration. Mounted files can be useful for configuration files, certificates, and applications that read settings from disk.

A `subPath` volume mount does not receive automated updates from ConfigMap or Secret changes.

--------------------------------------------------------------------

## Updating ConfigMaps and Secrets

When a ConfigMap or Secret changes, the effect on a running application depends on how it consumes the data.

- **Environment variables:** Existing processes do not automatically receive updated environment values. A restart or replacement Pod is commonly needed.
- **Mounted volumes:** Kubernetes can update projected files eventually, but the application may need to reload them.
- **`subPath` mounts:** These do not receive automated updates.

After changing configuration, verify that the workload has picked up the new values.

For Deployments, a controlled restart can be triggered with:

```bash
kubectl rollout restart deployment/<deployment-name>
```

This command affects the specified Deployment. Replace the placeholder with the intended name and use it only when a restart is appropriate.

--------------------------------------------------------------------

## Security Best Practices

- Never commit real passwords, API tokens, or private keys to Git.
- Use ConfigMaps only for non-sensitive configuration.
- Restrict Secret access using Kubernetes RBAC.
- Enable encryption at rest for Secrets where required.
- Limit access to etcd storage and backups.
- Avoid displaying Secret contents in logs, screenshots, and shared terminal output.
- Use dedicated service accounts and least-privilege permissions.
- Consider an external secrets manager for production workloads.
- Rotate credentials when exposure is suspected or when required by policy.
- Use separate configuration and credentials for different environments.

A Secret is a resource for handling sensitive data, not a replacement for a complete security strategy.

--------------------------------------------------------------------

## Common Troubleshooting

If a Pod cannot load configuration, inspect the Pod status, events, and resource references.

```bash
# List ConfigMaps
kubectl get configmaps

# List Secrets
kubectl get secrets

# Inspect a ConfigMap
kubectl describe configmap app-config

# Inspect Secret metadata
kubectl describe secret app-secret

# Inspect the Pod
kubectl describe pod <pod-name>

# Check application logs
kubectl logs <pod-name>
```

Common causes include:

- The ConfigMap or Secret does not exist.
- The resource is in a different namespace.
- The referenced key name is incorrect.
- The Pod specification contains invalid configuration.
- The application does not read the injected environment variable or mounted file.
- The application requires a restart or reload after a configuration change.
- Access controls prevent the workload or user from accessing the resource.

Troubleshoot by checking resource names, namespaces, keys, and Pod events. Avoid exposing secret values while diagnosing the problem.

--------------------------------------------------------------------

## Key Takeaways

- ConfigMaps store non-sensitive configuration.
- Secrets are intended for sensitive data such as credentials and tokens.
- ConfigMaps and Secrets can be consumed as environment variables or mounted files.
- `envFrom` can import multiple key-value pairs from a ConfigMap or Secret.
- `configMapKeyRef` and `secretKeyRef` select individual keys.
- Base64 encoding does not provide confidentiality.
- Secret security requires suitable access controls and storage protections.
- Configuration updates affect environment variables and mounted files differently.
- A running application may require a restart or reload to use changed configuration.
- Never commit real credentials to a public or shared Git repository.

--------------------------------------------------------------------

## Revision Questions

1. What is the difference between a ConfigMap and a Secret?
2. Why should configuration be separated from container images?
3. What is the purpose of the `data` field in a ConfigMap?
4. What do `stringData` and `type: Opaque` mean in a Secret?
5. Why is Base64 encoding not encryption?
6. How can a Pod consume a ConfigMap as environment variables?
7. What is the difference between `configMapKeyRef` and `envFrom`?
8. How can a Pod consume a Secret using `secretKeyRef`?
9. What is the difference between environment variables and mounted configuration files?
10. Why should real credentials never be committed to Git?

--------------------------------------------------------------------