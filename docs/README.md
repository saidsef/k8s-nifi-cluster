# Documentation

Standing it up:

- [Deploying](./deploying.md) - what your cluster needs before you apply anything, and what the first start looks like
- [Ingress and access](./ingress.md) - deploy targets, bringing your own controller, why port-forward fails
- [Configuration](./configuration.md) - every setting in the ConfigMap, and what to change before production

Living with it:

- [Operations](./operations.md) - health checks, and what the errors mean when it will not come up
- [Layout](./layout.md) - how `deployment/` is organised, image pinning, relocating the release
- [Local testing with kind](./local-testing.md) - reproduce the CI cluster on your machine

If you are trying this out for the first time, [local testing with kind](./local-testing.md) gets
you a working cluster in one page. Read [deploying](./deploying.md) before putting it on a real
one, particularly the section on the shared certificate volume.
