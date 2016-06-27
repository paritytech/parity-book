# Getting Started

There are a few ways of proceeding here. You can build Parity from the sources; you can install Parity from our binary releases for Ubuntu and Mac/Homebrew or, if you're on an Ubuntu Snappy platform, just use our Snappy App. Other Unix-like environments should work (assuming you have the latex *nix installed); we're not going to expend much effort supporting them, though build PRs are welcome.

## Installers

Head to the [latest release page](https://github.com/ethcore/parity/releases/latest) to find the latest release of Parity with the Windows, Mac and Ubuntu installer binaries.

## One-line Installer

This method works on Ubuntu and Mac with Homebrew installed. To use the script just run:

```
bash <(curl get.parity.io/ -Lk)
```

## Building Manually

You can build & install Parity manually; this means you'll have the codebase around for development but is otherwise slower to do.

### Grab Rust

NOTE: If you already have Rust in your environment, you don't need to bother with this. 

This will download and install Rust on Linux and OS X:

```
curl https://sh.rustup.rs -sSf | sh
```

If you are using Windows make sure you have Visual Studio 2015 with C++ support installed, download and run [rustup](https://static.rust-lang.org/rustup/dist/x86_64-pc-windows-msvc/rustup-init.exe) and use the following command to setup msvc toolchain:

```
rustup default stable-x86_64-pc-windows-msvc
```

### Install and Build Parity

Next, grab the Parity repository:
```
git clone https://github.com/ethcore/parity && cd parity
```

To install Parity in Mac OS or Linux, just build it and copy it to `/usr/local/bin`:
```
cargo build --release && cp target/release/parity /usr/local/bin
```

Or for Windows:
```
cargo build --release && copy target/release/parity C:/Windows
```
