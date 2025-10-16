# kops

## kubebuilder

```sh
kubebuilder init --domain xilia.net --repo xilia.net/guestbook
kubebuilder create api --group webapp --version v1 --kind Guestbook
make manifests
```
