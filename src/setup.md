# Setup

## Install Quarve CLI
```bash
cargo install quarve
```

## Additional Setup

### Cocoa Backend (macOS)

The default backend on macOS builds is Cocoa.
For Cocoa, Quarve works out of the box and you do not have to perform additional setup.

### Qt Backend (Windows/Linux)

The default backend on Windows and Linux builds is Qt, which you will have to install.

*Windows:*
[Install Qt](https://www.qt.io/download-dev) using msvc2022 (make sure you have visual studio installed).
1. Set the environment variable `QUARVE_BACKEND_PATH` to the root of Qt installation.
E.g. `C:\Qt\6.8.1\msvc2022_64`.
2. Append the binaries folder to your `PATH`. E.g. `C:\Qt\6.8.1\msvc2022_64\bin`.

*Linux:*
[Install Qt](https://www.qt.io/download-dev)
1. Set the environment variable `QUARVE_BACKEND_PATH` to the root of Qt installation.
E.g. `~/Qt/6.8.1/gcc_64`.
2. Append the library folder to to your e`PATH`. E.g. `~/Qt/6.8.1/gcc_64/bin/`

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
have to do to that yourself (and may want to change directory structure as well).
For macOS, you may have to update the `Info.plist` file to
e.g. customize the application icon. Depending on your intended distribution style,
you may have to codesign the application yourself too.

## Debugging

On Linux, if you get the error that `<Gl/gl.h>` is not found (or something similar),
it means that you must install the OpenGL dev kit.
