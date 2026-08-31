# Documentation

## Design

- [Architecture](./architecture.md) - how the components fit together, with a diagram

## Deployment

- [Deploying](./deploying.md) - cluster requirements, and what the first start looks like
- [Ingress and access](./ingress.md) - deploy targets, supplying your own controller, why port-forward fails
- [Configuration](./configuration.md) - every setting in the ConfigMap, and what to change before production

## Operation

- [Operations](./operations.md) - health checks, and what the common failures mean
- [Layout](./layout.md) - how `deployment/` is organised, image pinning, relocating the release
- [Local testing with kind](./local-testing.md) - reproduce the CI cluster locally

For a first deployment, [local testing with kind](./local-testing.md) produces a working cluster in
a single page. Read [deploying](./deploying.md) before deploying to a production cluster,
particularly the section on the shared certificate volume.
