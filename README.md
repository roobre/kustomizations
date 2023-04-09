# kustomize PoC

## Kustomize 101

### [Kustomization](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/)

A collection of `resources` (static, plain manifests) that are modified by `patches`, and other modifications that are applied on top of them.

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ingress.yaml
patches:
  - patch: |
      - op: replace
        path: /spec/rules/0/host
        value: testapp.terabox.moe
```

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: 
  name: my-ingress
spec:
  rules:
    - host: foo.bar
    # ...
```

`Kustomizations` can include other `kustomizations`, e.g. A includes B. The included kustomization patches are applied first on top of those resources, then the includer `kustomization`'s patches are applied on top of all resources.

The included patches cannot reference manifests in the includer `kustomization`, or in other included `kustomizations`.

### [Replacements](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/replacements/)

`Replacements` can get fields from an object and paste it into others.

The `Replacement` below gets the name of a `Deployment` and copies it into the name of all `Ingresses` and `Services` in the `Kustomization`.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Replacement
source:
    kind: Deployment
    fieldPath: metadata.name
targets:
  - select:
      kind: Ingress
    fieldPaths: metadata.name
  - select:
      kind: Service
    fieldPaths: metadata.name
```

Like patches, the `fieldPath` fields support descending into arbitrarly deep nodes, and filtering arrays.

`Replacements`

### [Components](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/components/)

`Components` change the rules of the game, by defining sets of both manifests and modifiers (patches, replacements, etc.) that are applied to all manifests belonging to the Kustomization that imports them.

This means a `Component` acts as a reusable patch set that can affect siblings and parents.

By defining a set of `Replacements` and `Patches` into a `Component` we can make, for example, a `Component` that creates a `Service` manifest for a given port, and injects that very same port into the first container of a `StatefulSet` that does _not_ belong to the `Component`.

## This repo

This makes heavy (ab)use of `Components` and `Replacements` to provide a skeleton for standard webapps, which are composed of a `StatefulSet` with a single container and a path to persist data, a `Service`, and an `Ingress`.

### Usage

#### The `kustomization-meta` `ConfigMap`

Base `Components` get _some_ application-specific information using a `Replacement` that copies things from a `ConfigMap` called `kustomization-meta`. This `ConfigMap` needs to be defined by the app and have a `name` and a `port` properties.

Other application-specific behavior is defined as `Patches` or `Replacements`. `name` and `port` are not specified this way because they need to be templated across multiple components, and they need to exist 

#### Components

The `ss-base` component provides a simple `StatefulSet`. `ss-base/hostpath` adds a `hostPath` mount to the `StatefulSet`. `ss-base/service` adds a service and a port to the `StatefulSet` container.

`ingress` adds an ingress to the application. `ingress/external` and `ingress/internal` are convenient wrappers which do the same, but with pre-filles `ingressClassName` and suffixed names.


`_names` is a special component that stamps the application name into every other resource. For this reason, *it must be the last component in the `components` list*.

#### Example

```yaml
# testapp.yaml
resources:
  - meta.yaml

components:
  - ../_resources/ingress/internal
  - ../_resources/ss-base
  - ../_resources/ss-base/hostpath
  - ../_resources/ss-base/service
  - ../_resources/_names

patches:
  - target:
      kind: Ingress
    patch: |
      - op: replace
        path: /spec/rules/0/host
        value: testapp.terabox.moe

images:
  - name: _
    newName: ghcr.io/roobre/testapp:1.2.3
```

```yaml
# meta.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kustomize-meta
data:
  name: testapp
  port: "8080"
```