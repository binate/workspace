# Binate Workspace

Container repo with git submodules for all Binate project repos.

## Submodules

| Repo | Description |
|------|-------------|
| [explorations](https://github.com/binate/explorations) | Design docs, notes, grammar, plans |
| [bootstrap](https://github.com/binate/bootstrap) | Bootstrap interpreter (Go) |
| [binate](https://github.com/binate/binate) | Self-hosted interpreter and compiler (Binate) |
| [website](https://github.com/binate/website) | Project website source |

## Setup

```sh
git clone --recurse-submodules https://github.com/binate/workspace.git
```

Or if already cloned:

```sh
git submodule update --init --recursive
```

## License

[MIT](LICENSE)
