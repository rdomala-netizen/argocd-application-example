# app2 — New Beginnings

A small Helm chart that serves a static landing page with nginx. It's a hands-on sandbox for learning GitOps with Argo CD: change something in Git, push, and watch Argo CD sync it to the cluster.

## Folder structure

```
app2/
├── Chart.yaml               # Chart metadata (name, description, versions)
├── values.yaml              # All configurable settings
├── files/
│   └── index.html           # The landing page (plain HTML, edit this to change the site)
└── templates/
    ├── 01-namespace.yaml    # Creates the app namespace
    ├── 02-configmap.yaml    # Loads files/index.html into a ConfigMap
    ├── 03-service.yaml      # ClusterIP Service in front of the pods
    ├── 04-deployment.yaml   # nginx Deployment serving the ConfigMap
    └── 05-ingress.yaml      # Ingress so the page is reachable without port-forwarding
```

The numeric prefixes are only for readability. Helm sorts resources by kind when installing, not by filename.

## How it fits together

```
Browser ──► Ingress (Traefik) ──► Service ──► nginx Pod ──► /usr/share/nginx/html/index.html
                                                              ▲
                                                   ConfigMap (from files/index.html)
```

1. `files/index.html` is read by `.Files.Get` and placed into the ConfigMap.
2. The Deployment mounts the ConfigMap at nginx's web root.
3. The Service exposes the pods inside the cluster on the named port `http`.
4. The Ingress routes requests for the configured host to the Service.

The Deployment has a `checksum/config` annotation: a hash of the rendered ConfigMap. When the HTML changes, the hash changes, so the pods roll automatically and serve the new page.

## Configuration (`values.yaml`)

| Key | Default | Description |
|-----|---------|-------------|
| `app.namespace` | `app2-ns` | Namespace all resources are created in |
| `image.repository` | `nginx` | Container image |
| `image.tag` | `"1.25"` | Image tag |
| `replicaCount` | `1` | Number of nginx pods |
| `service.type` | `ClusterIP` | Service type |
| `service.port` | `80` | Port the Service listens on (targets the container's `http` port) |
| `ingress.enabled` | `true` | Create the Ingress |
| `ingress.className` | `traefik` | Must match an IngressClass in the cluster |
| `ingress.annotations` | `{}` | Extra Ingress annotations |
| `ingress.host` | `app2.localhost` | Hostname to serve on (`*.localhost` resolves to 127.0.0.1) |
| `ingress.path` | `/` | URL path |
| `ingress.pathType` | `Prefix` | Path matching type |
| `ingress.tls` | _(unset)_ | Optional TLS block, rendered only if set |

## Prerequisites (local kind cluster)

- A kind cluster with port 80 mapped to the host (`extraPortMappings` in the kind config).
- Traefik installed, with `kubectl get ingressclass` showing `traefik`.
- Argo CD installed in the `argocd` namespace.

## Testing locally

Render the chart and check for errors before pushing:

```bash
helm lint ./app2
helm template app2 ./app2
```

## Deploying with Argo CD

Point an Argo CD Application at the chart folder:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app2
  namespace: argocd
spec:
  project: default
  source:
    repoURL: <your-git-repo-url>
    targetRevision: main
    path: app2
  destination:
    server: https://kubernetes.default.svc
    namespace: app2-ns
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

The chart creates its own namespace via `01-namespace.yaml`, so `CreateNamespace=true` is not needed. Use one or the other, not both.

Once synced, open **http://app2.localhost**.

## Making changes (the GitOps loop)

1. Edit `files/index.html`. For example, bump the version chip from `v1.0.0` to `v1.0.1`.
2. Commit and push.
3. Argo CD detects the change and syncs. The checksum annotation triggers a rollout.
4. Refresh the page to see the new version.

To try self-heal, delete the Deployment with `kubectl` and watch Argo CD recreate it.

## Troubleshooting

| Symptom | Check |
|---------|-------|
| `helm template` fails on the ConfigMap | Is `files/index.html` present? Is the `.Files.Get` line indented correctly? |
| "template not found" on the checksum | The `include` path in `04-deployment.yaml` must match `/02-configmap.yaml` |
| Page doesn't load | `kubectl get ingress -n app2-ns`, and confirm Traefik is running and port 80 is mapped in kind |
| Old page still showing | `kubectl get pods -n app2-ns` to confirm new pods rolled, then hard-refresh the browser |
| App OutOfSync in Argo CD | Open the app's diff view to see what differs from Git |