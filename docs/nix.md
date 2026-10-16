# Building and installing with Nix

The flake provides Penguin Mail and a development shell for `x86_64-linux`
and `aarch64-linux`. It pins Nixpkgs in `flake.lock`, including Rust and the
GTK, libadwaita, and WebKitGTK libraries. Nix needs its `nix-command` and
`flakes` experimental features enabled.

## Build and run

From a checkout:

```sh
nix build
nix run -- --demo
nix run .#cli -- --help
```

The package contains the app, command-line client, translations, icons,
desktop launcher, and AppStream metadata. `result/bin/penguin-mail` starts
the built app. To install it in your user profile:

```sh
nix profile install .#penguin-mail
```

Without a checkout, use `github:c9dev/penguin-mail` in place of `.`:

```sh
nix run github:c9dev/penguin-mail -- --demo
nix profile install github:c9dev/penguin-mail#penguin-mail
```

On NixOS, add this repository as an input to your system flake and put
`inputs.penguin-mail.packages.${pkgs.stdenv.hostPlatform.system}.penguin-mail`
in `environment.systemPackages`, or in Home Manager's `home.packages`.
The app uses the desktop's Secret Service for passwords and its portals
for desktop integration. Enable those services through your desktop's
NixOS or Home Manager configuration.

Updates come through your Nix profile, NixOS configuration, or Home
Manager configuration. The Nix build does not offer the app's own updater.

Assistant skill scripts currently expect `/usr` and `/bin` in their
bubblewrap sandbox. On NixOS that check may fail and the app will report
that scripts cannot run. The rest of the assistant does not need that
sandbox.

## Account sign-in

The default package supports IMAP, POP3, and JMAP accounts. Google and
Microsoft browser sign-in need client registration values supplied when
building; the flake does not import `packaging/secrets.env`. See
[Building your own copy](setup.md#building-your-own-copy) for registration
instructions.

For a package with your client values, override the package from your
own flake:

```nix
inputs.penguin-mail.packages.${pkgs.stdenv.hostPlatform.system}.penguin-mail.override {
  googleClientId = "your-client.apps.googleusercontent.com";
  googleClientSecret = "your-desktop-client-secret";
  microsoftClientId = "your-application-id";
}
```

Set only the providers you use. These are desktop application registration
values, not an account password. Nix stores build arguments in its store,
where other users can read them; do not pass account passwords, refresh
tokens, or a confidential server application's secret this way.

## Development

```sh
nix develop
cargo build -p mailrs -p mailrs-cli
cargo run -p mailrs -- --demo
```

The shell supplies Rust, Clippy, rustfmt, rust-analyzer, native library
headers, gettext, GnuPG, bubblewrap, D-Bus, and Xvfb. It also sets
the GTK schema, icon, and GIO module paths needed by a build-tree app.
For Google or Microsoft sign-in in that build, set the variables listed
in [setup](setup.md#building-your-own-copy) before running Cargo.

For translation work, install the Rust string extractor once with
`cargo install --locked xtr --version 0.1.11`, as the regular build does.

`nix flake check` builds the package and checks both executables' version
commands. The Nix workflow also evaluates both Linux architectures and
checks the development shell's compiler and native libraries.
Run the workspace test suite and
Clippy separately as described in [Contributing](../CONTRIBUTING.md#checks).
The suite needs a display and session bus, and its server integration
tests need Docker and the images prepared by `scripts/test-images.sh`.
The Nix package builder does not run that suite.

After a Cargo dependency change, the package vendors the new `Cargo.lock`
without a separate Cargo dependency hash. To update the native
libraries and compiler, run `nix flake update`, build the package, and
commit the updated lock file.
