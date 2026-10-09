# Test setup: `bitbake-setup`

Shared by `1-bitbake-setup-override-undeclared-source-ignored.md` and
`1-bitbake-setup-undeclared-bb-layer-accepted.md`. Not a bug report.

## Undeclared layer

`test2.conf.json`, with `meta-mylayer` (any layer directory) next to it:

```json
{
    "description": "undeclared layer repro",
    "sources": {
        "bitbake": {"git-remote": {"uri": "https://git.openembedded.org/bitbake", "branch": "master", "rev": "37b9c56ae3b2294ad170859a638c879910c82c59"}},
        "openembedded-core": {"git-remote": {"uri": "https://git.openembedded.org/openembedded-core", "branch": "master", "rev": "cc848c403aa3832453f1adfaa45411b8b72b9363"}}
    },
    "bitbake-setup": {"configurations": [{"name": "test", "description": "test", "bb-layers": ["openembedded-core/meta", "meta-mylayer"]}]},
    "version": "1.0"
}
```

## Layer outside any source

- List the layer in `bb-layers-file-relative`, relative to the config file
  (`doc/bitbake-user-manual/bitbake-user-manual-environment-setup.rst:1137`).
- Pass the config by path, not from a registry.
- `"../../meta-mylayer"` works (bbappends included), but lands unnormalized in
  `bblayers.conf`.
