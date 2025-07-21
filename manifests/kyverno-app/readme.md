# Kyverno setup

I had to use ``kubectl apply -f template.yaml --server-side`` to get the Kyverno app installed.

Something about the policy and clusterPolicy crd was causing the following error without server-side apply:

```log
Error from server (Invalid): error when creating ".\\template.yaml": CustomResourceDefinition.apiextensions.k8s.io "policies.kyverno.io" is invalid: metadata.annotations: Too long: may not be more than 262144 bytes
```