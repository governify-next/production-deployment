# Governify production deployment

Private configuration for a second Governify installation at `*.next.governify.io`, in namespace `governify-production`.

This repository prepares the deployment. Creating or cloning it does not deploy anything. Applying the Argo CD Application enables automatic synchronization.

## What is pinned

`production/kustomization.yaml` imports the public infrastructure base at commit
`3dc95dac1400ac96b7f4a3a312a9fe2e58927eb9`.

The following services use the published `v1.1.0` images, with immutable SHA-256 digests: authenticator, scope-manager, registry, computer, fetcher, director, join-backend, join-frontend, and frontend.

Reporter has no tagged release. Its `develop` image is fixed to digest
`sha256:286348ad6efb29486ebb713d6b0df8297e93c8085303d5d50cf16b83b53658a2`,
resolved on 2026-09-15. This image will not follow later pushes to `develop`.

MongoDB 8.0, Redis 8.2, InfluxDB 3.9-core, and Grafana 12.3.0 are also pinned by digest. The complete image references live in one place: `production/kustomization.yaml`.

Application source is the released code, including the feature set present in `develop` when v1.1.0 was prepared. Later feature-branch changes are not included.

## Production addresses

| Service | URL |
| --- | --- |
| frontend | https://frontend.next.governify.io |
| authenticator | https://authenticator.next.governify.io |
| scope-manager | https://scope-manager.next.governify.io |
| registry | https://registry.next.governify.io |
| computer | https://computer.next.governify.io |
| fetcher | https://fetcher.next.governify.io |
| reporter | https://reporter.next.governify.io |
| director | https://director.next.governify.io |
| join-backend | https://join-backend.next.governify.io |
| join-frontend | https://join.next.governify.io |
| grafana | https://grafana.next.governify.io |

Create DNS records for these names pointing to the existing ingress controller. A wildcard DNS record for `*.next.governify.io` is sufficient unless a more specific record overrides it.

The ingress lists each hostname explicitly. It uses the existing Traefik `websecure` entrypoint and ACME resolver `default`, which obtains certificates for the listed hosts. It does not request a wildcard TLS certificate.

## Cluster requirements

- The existing k3s Traefik controller and its `default` certificate resolver must be configured.
- The published service images support Linux amd64. The overlay selects Linux amd64 nodes.
- A default StorageClass and 85 GiB of additional PVC capacity are required: MongoDB 20 GiB, Redis 5 GiB, InfluxDB 50 GiB, and Grafana 10 GiB.
- The namespace has its own application services, data services, persistent volumes through PVCs, and runtime Secret.
- Headlamp resources and their cluster-admin binding are omitted. The existing installation continues to own the shared dashboard and Traefik configuration.

## Preview without deploying

From the repository root, with kubectl installed and GitHub access:

```sh
kubectl kustomize production
```

The build fetches the pinned public base. It renders one namespace, 14 Services, 14 Deployments, four PVCs, and one Ingress.

## Prepare runtime configuration when ready

Copy `secrets.example.yaml` to `secrets.yaml` and fill every placeholder. The real file is ignored by Git; never commit credentials.

Use independent production JWT, service-client, database-token, Grafana, and administrator secrets. The two Influx token fields must contain exactly the same token. Store the filled file securely outside Git.

Register these OIDC redirect URIs with the Google OIDC application used by the production credentials:

- `https://authenticator.next.governify.io/api/v1/users/oidc/callback`
- `https://scope-manager.next.governify.io/api/v1/users/oidc/callback`

Configure the GitHub App for:
`https://join-backend.next.governify.io/api/v1/integrations/github/callback`.

The inherited GitHub App slug is `governify-next`. If production uses a separate GitHub App, add its `GITHUB_APP_SLUG` to the Join Backend patch in `production/public-urls.yaml`, and supply that app's credentials in the Secret. Ensure provider configuration also continues to support the existing development installation.

## Deploy with Argo CD when ready

1. Connect this private repository in Argo CD using a read-only deploy key or read-only GitHub credential. Keep that credential in Argo CD, outside this repository.
2. Select the actual production cluster context explicitly.
3. Create the production namespace and runtime Secret before applying the Application:

```sh
export PRODUCTION_CONTEXT=your-production-context
kubectl --context "$PRODUCTION_CONTEXT" create namespace governify-production --dry-run=client -o yaml | kubectl --context "$PRODUCTION_CONTEXT" apply -f -
kubectl --context "$PRODUCTION_CONTEXT" apply -f secrets.yaml
kubectl --context "$PRODUCTION_CONTEXT" apply -f argocd/governify-production.yaml
```

The Application tracks this repository's `main` branch and `production` directory. Automatic sync, pruning, and self-healing start as soon as the Application is applied. Shared-resource detection helps catch accidental overlap with another Argo CD Application.

The existing Image Updater matches only the development Application `governify-next`; keep the new Application `governify-production` excluded.

Alternatively, after creating the namespace and Secret, deploy directly with
`kubectl --context "$PRODUCTION_CONTEXT" apply -k production` and do not apply the Argo CD Application.

## Release updates

Update each selected service's `newTag` and corresponding `digest` together in `production/kustomization.yaml`, render the manifests, and commit the change. Argo CD then applies it once the Application is installed.

Update the public base commit only deliberately. Reverting an image-pin commit restores the previous image selection; database migrations may require separate rollback work.
