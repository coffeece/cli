# coffeece

CLI for the [Coffeece](https://coffeece.com) PaaS — deploys apps and AI agents in one command.

```sh
$ coffeece deploy
→ criando app "meu-agente" (python)
→ aplicando 3 variável(is) privada(s)
→ deploy de "meu-agente"
✓ no ar: https://meu-agente.coffeece.com
```

## Install

```sh
go install github.com/coffeece/cli/cmd/coffeece@latest
```

Pre-built binaries: [Releases](https://github.com/coffeece/cli/releases).

## Usage

```sh
coffeece login          # autenticar
coffeece deploy         # create + env-set + deploy num comando só
coffeece app list       # gerenciar apps
coffeece env set …      # variáveis de ambiente
coffeece service …      # bancos, caches, recursos gerenciados
```

`coffeece` reuses the Tsuru API and command engine from
[`tsuru/tsuru-client`](https://github.com/tsuru/tsuru-client), pinned to the
Coffeece API endpoint and curated for the user-facing surface (no admin/operator
commands). The `coffeece deploy` composite is original to this CLI.

## License

[Apache-2.0](LICENSE). Bundled dependencies retain their own licenses
(`tsuru-client` is BSD-3-Clause).
