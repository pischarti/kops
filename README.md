# kops

## kubebuilder

```sh
kubebuilder init --domain xilia.net --repo xilia.net/guestbook
kubebuilder create api --group webapp --version v1 --kind Guestbook
make manifests
```

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
