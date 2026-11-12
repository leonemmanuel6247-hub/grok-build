# Compilation 32-bit (i686) du binaire `xai-grok-pager` — Debian 12

Cible : machine **32-bit** (HP Pavilion dv1000, 2 Go RAM) sous **Debian 12 (i386, glibc 2.36)**.

Le workflow [`.github/workflows/build.yml`](.github/workflows/build.yml) produit un binaire
`xai-grok-pager` **ELF 32-bit i686**, directement exécutable sur cette machine.

## Choix techniques

| Point | Décision |
|---|---|
| Cible | `i686-unknown-linux-gnu` (un binaire x86_64 ne démarrerait pas sur ce CPU) |
| Runner | `ubuntu-latest` + `container: debian:12` → le binaire est lié à **glibc 2.36**, exactement celle de la machine cible |
| Multilib | `gcc-multilib` + `libc6-dev-i386` (le runner est x86_64), contrôlé par un `gcc -m32` |
| `fastant` | **0.1.10 ne compile pas en 32-bit** ; `Cargo.lock` est épinglé sur **0.1.11** (correctif amont) |
| protoc | **29.3** téléchargé depuis protobuf/releases, `PROTOC=/opt/protoc/bin/protoc` (le wrapper DotSlash du dépôt n'est pas installé) |
| Mémoire build | `CARGO_INCREMENTAL=0`, `CARGO_BUILD_JOBS=2`, `RUSTFLAGS="-C codegen-units=1 -C force-unwind-tables=yes"` |
| Commande | `cargo build --release -p xai-grok-pager-bin --target i686-unknown-linux-gnu --no-default-features --features stock,sandbox-enforce` |
| Sortie | artefact `xai-grok-pager-i686-debian12` (binaire strippé) |

La cible du dépôt (`.cargo/config.toml`) est écrasée deux fois : `CARGO_BUILD_TARGET` (env)
et `--target` (CLI, prioritaire).

## Récupérer et lancer le binaire

```bash
unzip xai-grok-pager-i686-debian12.zip
chmod +x xai-grok-pager
./xai-grok-pager --version
grep -m1 -o sse2 /proc/cpuinfo   # vérifier SSE2 (cible i686 par défaut = pentium4)
```

Avec 2 Go de RAM, prévoir un swap :

```bash
sudo fallocate -l 2G /swapfile && sudo chmod 600 /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile
```

## Dépannage

| Symptôme | Cause | Solution |
|---|---|---|
| `error[E0432]: could not find x86_64 in arch` | `fastant` < 0.1.11 | déjà corrigé dans `Cargo.lock` |
| `aws-lc-sys` ne compile pas | AWS-LC est mal supporté en x86 32-bit | remplacer `features = ["aws-lc-rs"]` par `["ring"]` (rustls) et `tls-aws-lc` par `tls-ring` (tonic) |
| `openssl-sys` / `-lssl` introuvable | OpenSSL requis en 32-bit | `dpkg --add-architecture i386 && apt-get install libssl-dev:i386` |
| `GLIBC_2.xx not found` | build hors Debian 12 | garder `container: debian:12` |
| `Illegal instruction` | CPU sans SSE2 | ajouter `-C target-cpu=pentium3` aux `RUSTFLAGS` |
| Job tué (OOM) | trop de jobs | `CARGO_BUILD_JOBS: '1'` |
