# Ommega

**Ommega** (`id=ommega`) is an A-side relay module for remote TEE attestation via the real hardware keystore. It intercepts Android KeyMint/KeyStore calls inside `keystore2` and relays attestation through the device's actual hardware TEE, with a local keybox fallback. Version 1.2.0, KernelSU / APatch module format.

> A-side relay: remote TEE attestation via real hardware TEE.

## Features

- **Hardware TEE attestation** — attestation chains are produced by the device's real hardware keystore (KeyMint), not a purely software spoof
- **Remote relay mode** — optional connection to a remote relay server (server URL, device ID, token) with automatic local-keybox fallback
- **Per-package targeting** — `injector.toml` decides which UIDs/packages are intercepted and which KeyMint binder calls are hooked (default scoop list covers `keyattestation`, `gms`, `gsf`, `vending`, ...)
- **WebUI manager** — multilingual in-app panel (KernelSU / APatch) for the app list, keybox, security patch, boot hash, and remote config
- **Self-verifying install** — every payload file is SHA-256 verified at install time (`verify.sh`)

## How it works

```
keystore2 ──KeyMint binder──► injected hook (daemon-injector + libs/arm64-v8a/keymint)
                                    │ selects per-pkg targets from injector.toml
                                    ▼
                     real hardware TEE (KeyMint TA) ──► attestation chain
                                    │  (fallback: bundled keybox.xml)
                     remote relay (optional) ◄── device reports
```

1. `post-fs-data.sh` mounts/verifies, `service.sh` starts the daemon pair; `sepolicy.rule` grants the required SELinux domains.
2. `daemon-injector` injects into `keystore2`; the injected `keymint` binary intercepts configured binder calls.
3. Attestation requests are re-routed through the hardware TEE; the response chain (or keybox fallback) is returned to the caller.

## Install

Flash the zip in KernelSU / APatch manager. `customize.sh` verifies SHA-256 of every payload, installs binaries to the module dir, and applies sepolicy. Targets `arm64-v8a`.

## Source map

| Path | Role |
|---|---|
| `daemon` | supervisor keeping the injection alive across keystore2 restarts |
| `daemon-injector` | ptrace-based injector |
| `libs/arm64-v8a/keymint` | KeyMint interception layer |
| `libs/arm64-v8a/inject` | injection core |
| `injector.toml` | package/UID filter + intercept config |
| `webroot/` | WebUI (Vue build: app list, keybox, patch, boot hash, remote config) |
| `verify.sh` / `customize.sh` | integrity verification + installer |
