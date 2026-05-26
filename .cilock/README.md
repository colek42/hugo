# cilock secure supply chain (NIST SP 800-204D / SSDF)

This fork carries a `secure-supply-chain` GitHub Actions pipeline
(`.github/workflows/secure-supply-chain.yml`) that attests Hugo's build
end-to-end with [cilock](https://github.com/aflock-ai/rookery) and gates the
result against a signed [Witness](https://github.com/in-toto/witness) policy.

## Pipeline stages → standards mapping

| Stage | What it attests | SSDF (SP 800-218) | 800-204D |
|-------|-----------------|-------------------|----------|
| source | git commit, author, remote | PS.1, PO.3 | Source integrity |
| sbom | CycloneDX SBOM of dependencies (syft) | PW.4 | Component inventory |
| vuln-scan | govulncheck call-graph results | RV.1, PW.8 | Vulnerability scanning |
| build | go build + binary SBOM, **eBPF-traced** materials/products, SLSA provenance | PW.6, PS.1 | Build provenance / hermeticity evidence |
| container | OCI image digest bound to base images + external downloads | PW.6 | Packaging provenance |
| verify | policy verification summary (the gate) | PS.2 | Admission / release decision |

## Why eBPF for the build

Walk-mode capture diffs the workspace before/after the build and can only see
files whose *content digest* changed. The build stage instead uses
`--capture-mode trace:ebpf`, which observes the actual `open`/`write`/`rename`
syscalls the compiler makes — capturing the precise set of source files read
and artifacts produced, including deterministic rebuilds that re-emit
byte-identical output. The flag **fails loudly** if eBPF can't load (no silent
fallback to ptrace or walk), so a green build is proof the trace was eBPF.

cilock runs under `sudo` on the hosted runner to obtain `CAP_BPF` /
`CAP_SYS_ADMIN`, then drops the traced build back to the unprivileged runner
uid before exec.

## Trust roots

- `cilock-signing.pub` — the public key every stage's signature is verified
  against (keyid `d188282e…`).
- `policy.signed.json` — the signed policy enumerating required steps, their
  required attestation types, and the trusted functionary key.

This demo uses a file-based signing key (stored as the `CILOCK_SIGNING_KEY`
repo secret). The production path is keyless: Fulcio short-lived certs from
GitHub OIDC, transparency logging, and Archivista evidence storage via the
TestifySec platform.
