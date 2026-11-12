# Compilation du binaire `xai-grok-pager` en CI (GitHub Actions)

Ce dépôt est une copie de [`xai-org/grok-build`](https://github.com/xai-org/grok-build)
(Apache-2.0) à laquelle on a ajouté le workflow
[`.github/workflows/build.yml`](.github/workflows/build.yml) qui compile le binaire
en **x86_64-unknown-linux-gnu**.

> Remplace simplement les sources par ta version adaptée (NemApi, etc.) :
> copie `.github/workflows/build.yml` dans ton projet local, puis
> `git push --force` — le workflow reste valable tel quel.

## Ce que fait le workflow

| Étape | Détail |
|---|---|
| Déclencheurs | `push`, `workflow_dispatch` (bouton *Run workflow*) |
| Runner | `ubuntu-latest` (x86_64) |
| Toolchain | Rust **1.94.0** via `dtolnay/rust-toolchain@stable`, target `x86_64-unknown-linux-gnu` |
| Système | `pkg-config libssl-dev protobuf-compiler cmake build-essential perl unzip` |
| protoc | **29.3** téléchargé depuis les releases protobuf, installé dans `/opt/protoc`, `PROTOC=/opt/protoc/bin/protoc` (celui de l'OS est ignoré) |
| Mémoire | swap 8 G, `CARGO_INCREMENTAL=0`, `CARGO_BUILD_JOBS=4`, `RUSTFLAGS="-C codegen-units=1 -C force-unwind-tables=yes"` |
| Commande | `cargo build --release -p xai-grok-pager-bin --target x86_64-unknown-linux-gnu --no-default-features --features stock,sandbox-enforce` |
| Sortie | artefact `xai-grok-pager` (= `target/x86_64-unknown-linux-gnu/release/xai-grok-pager`) |

## Pourquoi x86_64 et pas i686 ?

La cible d'origine du dépôt est `i686-unknown-linux-gnu` (32-bit), d'où :

- `fastant` ne compile pas en 32-bit (`core::arch::x86_64` est absent) ;
- les runners GitHub Actions sont des x86_64 (le build 32-bit exigerait un toolchain
  multilib) ;
- le 64-bit est plus rapide et nettement moins gourmand.

Le workflow **surcharge** donc la cible de deux façons (la CLI gagne sur le fichier de
config) :

1. variable `CARGO_BUILD_TARGET: x86_64-unknown-linux-gnu` ;
2. option `--target x86_64-unknown-linux-gnu` sur la ligne de commande.

Si ton `.cargo/config.toml` contient `[build] target = "i686-unknown-linux-gnu"`, rien
à changer : c'est écrasé.

## Récupérer le binaire

1. Onglet **Actions** → run **Build xai-grok-pager (x86_64)**.
2. Section **Artifacts** → télécharger `xai-grok-pager`.

```bash
unzip xai-grok-pager.zip
chmod +x xai-grok-pager
./xai-grok-pager --version
```

## Rappels

- `bin/protoc` du dépôt est un wrapper **DotSlash** (protoc 29.3). Le workflow n'installe
  pas `dotslash` : il fournit directement `PROTOC`, qui est prioritaire (cf.
  `crates/build/xai-proto-build/src/find_protoc.rs`).
- Le workspace de travail reste sur ta machine (NemApi en local) : la CI ne fait que
  produire le binaire.
- Si le job meurt en OOM : passe `CARGO_BUILD_JOBS` à `2` (voir `env:` du workflow).
