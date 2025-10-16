# kops

## kubebuilder

```sh
kubebuilder init --domain xilia.net --repo xilia.net/guestbook
kubebuilder create api --group webapp --version v1 --kind Guestbook
make manifests

kubebuilder create api --group webapp --version v1 --kind CronJob
make manifests
```

### CRD Size Limitation Fix

When creating CRDs that embed complex Kubernetes types (like `batchv1.JobTemplateSpec`), the generated CRD can exceed Kubernetes' 262KB annotation limit when using `kubectl apply`. 

**Problem**: The CronJob CRD includes a full `JobTemplateSpec` which contains the complete Pod specification, resulting in a very large OpenAPI schema.

**Solutions Applied**:

1. **Added PreserveUnknownFields markers** in `api/v1/cronjob_types.go`:
   ```go
   // +kubebuilder:pruning:PreserveUnknownFields
   // +kubebuilder:validation:XPreserveUnknownFields
   JobTemplate batchv1.JobTemplateSpec `json:"jobTemplate"`
   ```
   This reduces the schema size by not generating full validation for nested fields.

2. **Use server-side apply** in the Makefile:
   ```makefile
   kubectl apply --server-side -f -
   ```
   Server-side apply doesn't create the `kubectl.kubernetes.io/last-applied-configuration` annotation, bypassing the size limit.

## Development on macOS with Nix

### Linker Error Fix

When running tests on macOS ARM64 in a Nix environment, you may encounter a linker error related to `_SecTrustCopyCertificateChain`:

```
Undefined symbols for architecture arm64:
  "_SecTrustCopyCertificateChain", referenced from:
      _crypto/x509/internal/macos.x509_SecTrustCopyCertificateChain_trampoline.abi0
ld: symbol(s) not found for architecture arm64
```

This occurs because Go's `crypto/x509` package on macOS requires access to the Security and CoreFoundation frameworks for TLS certificate handling.

**Solution**: The `flake.nix` is configured to use system frameworks directly via CGO environment variables:

```nix
shellHook = pkgs.lib.optionalString pkgs.stdenv.isDarwin ''
  export CGO_LDFLAGS="-F/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/System/Library/Frameworks -framework CoreFoundation -framework Security"
  export CGO_CFLAGS="-F/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/System/Library/Frameworks"
'';
```

This approach uses the system-provided macOS frameworks instead of Nix-provided framework packages (which have been removed as legacy compatibility stubs in newer nixpkgs versions).

To use the development environment:

```sh
nix develop
make test
```

## Running webhooks locally

If you want to run the webhooks locally, you'll have to generate certificates for serving the webhooks, and place them in the right directory (`/tmp/k8s-webhook-server/serving-certs/tls.{crt,key}`, by default).

If you're not running a local API server, you'll also need to figure out how to proxy traffic from the remote cluster to your local webhook server. For this reason, we generally recommend disabling webhooks when doing your local code-run-test cycle, as we do below.

In a separate terminal, run:

```sh
export ENABLE_WEBHOOKS=false
make run
```

## FAQ

### Too long: must have at most 262144 bytes

If you encounter the error `Too long: must have at most 262144 bytes` when running `make install` or `make deploy`, this occurs due to a size limit enforced by the Kubernetes API.

**Why does this happen?**

When using `kubectl apply`, the Kubernetes API stores the entire previous configuration in a `kubectl.kubernetes.io/last-applied-configuration` annotation. If your CRD is too large, this annotation exceeds the 262KB limit. [More details in the Kubebuilder FAQ](https://book.kubebuilder.io/faq#the-error-too-long-must-have-at-most-262144-bytes-is-faced-when-i-run-make-install-to-apply-the-crd-manifests-how-to-solve-it-why-this-error-is-faced).

**Solutions implemented in this project:**

1. **Server-side apply** (recommended): The `make install` and `make deploy` targets use `kubectl apply --server-side`, which doesn't create the large annotation. This is the approach we've implemented.

2. **PreserveUnknownFields markers**: For complex nested types like `JobTemplateSpec`, we use kubebuilder markers to reduce schema size:
   ```go
   // +kubebuilder:pruning:PreserveUnknownFields
   // +kubebuilder:validation:XPreserveUnknownFields
   JobTemplate batchv1.JobTemplateSpec `json:"jobTemplate"`
   ```

**Alternative approaches** (if server-side apply doesn't work for you):

- **Remove descriptions**: Add `maxDescLen=0` to controller-gen in the Makefile:
  ```makefile
  $(CONTROLLER_GEN) rbac:roleName=manager-role crd:maxDescLen=0 webhook paths="./..." output:crd:artifacts:config=config/crd/bases
  ```

- **Re-design your APIs**: If your CRD is extremely large, consider splitting it into multiple smaller resources.

For more information, see the [CRD Size Limitation Fix](#crd-size-limitation-fix) section above.

