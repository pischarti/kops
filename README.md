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
