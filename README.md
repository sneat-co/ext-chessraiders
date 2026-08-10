# sneat-ext-contract-template

Template repository for creating a public `ext-<id>` contract repository.

`frontend/` is the sole owner and publisher of
`@sneat/extension-template-contract`. The paired implementation template is
[`sneat-ext-template`](../sneat-ext-template); it consumes this package and owns
the runtime/app code.

For a new extension, create `ext-<id>` from this template, rename `template`,
publish the contract, and then point the `<id>` implementation at that release.

## Layout

```text
typespec/   # frozen wire contract
backend/    # contract-facing Go definitions and checks
frontend/   # @sneat/extension-<id>-contract workspace
```
