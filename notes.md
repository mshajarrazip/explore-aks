# Notes

## Helm

### Important: Kubernetes Dashboard is retired

The original Dashboard is no longer maintained. Kubernetes now recommends Headlamp for new installations.&#x20;

You can install Headlamp instead:

```
helm repo add headlamp https://kubernetes-sigs.github.io/headlamp/

helm repo update

helm install my-headlamp headlamp/headlamp \
  --namespace kube-system
```

This is the installation method from the [official Headlamp documentation](https://headlamp.dev/?utm_source=chatgpt.com).

My recommendation: For an existing Dashboard setup, the corrected repository URL works. For a fresh installation, use Headlamp to avoid deploying software that no longer receives maintenance or security updates.
