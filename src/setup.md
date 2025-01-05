# Setup

## Install Quarve CLI
```bash
cargo install quarve
```

## Additional Setup

### Cocoa Backend (macOS)

The default backend on macOS builds is Cocoa.
For Cocoa, Quarve works out of the box and you do not have to perform additional setup.

### Qt Backend (Windows/Linux/macOS)

The default backend on Windows and Linux builds is Qt, which you will have to install.
Optionally, for macOS builds you can use Qt as the backend as well, but we notice
worse performance compared to Cocoa.

*Windows:*
[Install Qt](https://www.qt.io/download-dev) using msvc2022 (make sure you have visual studio installed).
1. Set the environment variable `QUARVE_BACKEND_PATH` to the root of Qt installation.
E.g. `C:\Qt\6.8.1\msvc2022_64`.
2. Append the binaries folder to your `PATH`. E.g. `C:\Qt\6.8.1\msvc2022_64\bin`.

*macOS:*
[Install Qt](https://www.qt.io/download-dev)
1. SET the environment variable `QUARVE_BACKEND_PATH` to the root of Qt installation.
E.g. `~/Applications/Qt/6.8.1/macos`.
2. Add the following to your `build.rs`
```rust
#[cfg(target_os = "macos")]
{
    let mut qt_path = std::env::var("QUARVE_BACKEND_PATH").unwrap();
    qt_path += "lib/";

    println!("cargo:rustc-link-arg=-Wl,-rpath,{}", qt_path);
}
```

## Create new project
```bash
quarve new <project_name>
```

It may be helpful for beginners to include the quarve prelude to access commonly
used constants, modules, functions, and traits without having to explicitly import them.
This is optional, though.
```rust
use quarve::prelude::*
```

## Run project
You can run the project just like any other rust binary, or use the following command:
```bash
quarve run
# for workspaces, you may have to explicitly specify the name of the crate
quarve run -n <crate_name>
```

## Deploy Project
```bash
quarve deploy
# for workspaces, you may have to explicitly specify the name of the crate
quarve deploy -n <crate_name>
```
After deploying, a target application will be created in the `quarve_target` directory.
As of now, we do not know do executable packing so for Windows and Linux you will
have to do to that yourself. For macOS, you may have to update the `Info.plist` file to
e.g. customize the application icon. Depending on your intended distribution style,
you may have to codesign the application yourself too.
