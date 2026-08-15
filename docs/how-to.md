# How-to guides

> **Status: the library is designed, not implemented.** `graph.aql`
> exports an empty `Graph` namespace, so there is no API to document
> yet. See **[DESIGN.md](../DESIGN.md)** for the argued plan.

Task recipes will appear here as words land.

## Install and run boru

Build the interpreter from source:

```bash
git clone https://github.com/boru-lang/boru
cd boru/cmd/go && go build -o ~/.local/bin/boru ./boru
```

Then, from this repository's root:

```bash
boru test/graph_smoke_test.aql
```

## Run the tests

```bash
for f in test/graph_*.aql; do boru "$f"; done
```

Every assertion-bearing suite ends by printing `all green`; the smoke
suite prints `--- ok ---`.
