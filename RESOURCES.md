# Argo CD learning resources

## Knowledge

- [Argo CD getting started](https://argo-cd.readthedocs.io/en/stable/getting_started/)
  Bootstrap, initial admin login and password rotation. Use for checkpoint 1.
- [Argo CD v3.5.4 release](https://github.com/argoproj/argo-cd/releases/tag/v3.5.4)
  Security fixes and release changes behind the installation pin.
- [Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)
  Resources, namespaces and patches. Use when explaining our installation files.
- [Kubernetes custom resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
  How CRDs extend the API and pair with controllers.
- [Argo Ingress](https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/)
  HTTP/TLS and proxy configuration; use to understand the existing Traefik route.

- [Sealed Secrets v0.40.0 documentation](https://github.com/bitnami/sealed-secrets/tree/v0.40.0#readme)
  Strict scope, full key backup, 30-day renewal and offline recovery. Use for checkpoint 2.
- [Sealed Secrets v0.40.0 release](https://github.com/bitnami/sealed-secrets/releases/tag/v0.40.0)
  Includes the published controller endpoint security fixes.
- [ApplicationSet Go templates](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/GoTemplate/)
  Directory path segments, missing-key handling and generated Application fields.
- [Application deletion](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Application-Deletion/)
  Resource preservation when a generated Application disappears.
- [age](https://github.com/FiloSottile/age)
  Passphrase encryption and decryption for independent key backups.

## Wisdom (Communities)

- [Argo CD discussions](https://github.com/argoproj/argo-cd/discussions)
  Project discussions for operational questions that benefit from practitioner experience.
