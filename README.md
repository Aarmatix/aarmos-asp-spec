# Agent Security Policy (ASP)

Public specification for the Aarmos Security Policy (ASP) YAML authoring surface.

ASP is a declarative, YAML-authored format for the policies the Aarmos runtime enforces on every agent turn. It compiles deterministically to a canonical, signed JSON policy bundle. The runtime does not need to change to adopt ASP.

- **Normative spec:** [SPEC.md](./SPEC.md)
- **Aarmos docs:** https://www.aarmos.io/docs/asp-spec
- **Reference compiler:** [`@aarmos/cli`](https://www.npmjs.com/package/@aarmos/cli)

## Quickstart

```bash
# Generate a signing keypair
aarmos policy keygen

# Author asp.yaml, then lint and test offline
aarmos lint --strict
aarmos test

# Sign for shipment
export AARMOS_POLICY_PRIVATE_KEY=...
aarmos policy sign asp.yaml --out policy.signed.json
```

See [SPEC.md](./SPEC.md) for the full field reference, canonicalization rules, signing algorithm, and conformance criteria.

## License

Specification text is licensed under [CC BY 4.0](./LICENSE).
