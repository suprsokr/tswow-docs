# TSWoW macOS Setup Guide

Quick setup for building TSWoW on macOS.

## Prerequisites

- **macOS** (Apple Silicon or Intel)
- **Xcode Command Line Tools**: `xcode-select --install`
- **Homebrew**: https://brew.sh

## Installation

### Step 1: Install Homebrew Dependencies

First, ensure Homebrew is installed. If not, install it from https://brew.sh

Then install the required dependencies:

```bash
brew install mysql@8.4 openssl@3 p7zip icu4c
```

### Step 2: Install CMake 3.25.0

Install CMake 3.25.0 to `/opt/cmake-3.25.0`:

```bash
if [ ! -d "/opt/cmake-3.25.0" ]; then
    cd /tmp
    curl -L -O https://github.com/Kitware/CMake/releases/download/v3.25.0/cmake-3.25.0-macos-universal.tar.gz
    tar xzf cmake-3.25.0-macos-universal.tar.gz
    sudo mv cmake-3.25.0-macos-universal /opt/cmake-3.25.0
    rm cmake-3.25.0-macos-universal.tar.gz
    cd -
fi
```

### Step 3: Install Boost 1.82.0

Install Boost 1.82.0 to `/opt/boost_1_82_0` (this takes ~10 minutes):

```bash
if [ ! -d "/opt/boost_1_82_0" ]; then
    cd /tmp
    curl -L -O https://archives.boost.io/release/1.82.0/source/boost_1_82_0.tar.bz2
    tar xjf boost_1_82_0.tar.bz2
    cd boost_1_82_0
    
    ICU_PATH=$(brew --prefix icu4c)
    ./bootstrap.sh --prefix=/opt/boost_1_82_0 --with-icu=$ICU_PATH
    
    sudo ./b2 --prefix=/opt/boost_1_82_0 \
      --with-system \
      --with-filesystem \
      --with-program_options \
      --with-iostreams \
      --with-regex \
      --with-locale \
      cxxflags="-std=c++11" \
      -j$(sysctl -n hw.ncpu) install
    
    cd -
    rm -rf /tmp/boost_1_82_0 /tmp/boost_1_82_0.tar.bz2
fi
```

### Step 4: Install Node.js 20.18.0 via nvm

Install nvm (Node Version Manager) if you don't have it:

```bash
if [ ! -d "$HOME/.nvm" ]; then
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
fi
```

Load nvm and install Node.js 20.18.0:

```bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && source "$NVM_DIR/nvm.sh"
nvm install 20.18.0
nvm use 20.18.0
```

### Step 5: Install direnv

Install direnv if not already present:

```bash
brew install direnv
```

### Step 6: Setup MySQL

#### Initial Setup (First Time Only)

If MySQL hasn't been initialized yet, you need to initialize the data directory first. This creates the MySQL data directory and sets up the initial database structure with a root user that has no password:

```bash
MYSQL_DATA_DIR="/opt/homebrew/var/mysql@8.4"
MYSQL_BIN="/opt/homebrew/opt/mysql@8.4/bin"

if [ ! -d "$MYSQL_DATA_DIR" ] || [ -z "$(ls -A $MYSQL_DATA_DIR 2>/dev/null)" ]; then
    echo "Initializing MySQL data directory..."
    $MYSQL_BIN/mysqld --initialize-insecure \
        --datadir="$MYSQL_DATA_DIR" \
        --user=$(whoami)
    echo "MySQL initialization complete. Root user has no password."
fi
```

**Note:** The `--initialize-insecure` flag creates a root user with no password, which is fine for local development. The initialization only needs to be done once - if the data directory already exists and contains files, this step will be skipped.

#### Start MySQL Service

Start MySQL using Homebrew services (recommended):

```bash
brew services start mysql@8.4
```

#### Create the TSWoW MySQL User

TSWoW requires a MySQL user named `tswow` with password `password` that has access to create and manage databases. Connect to MySQL as root (no password if initialized with `--initialize-insecure`) and create the user:

```bash
MYSQL_BIN="/opt/homebrew/opt/mysql@8.4/bin"
$MYSQL_BIN/mysql -u root -h 127.0.0.1 -P 3306 <<EOF
CREATE USER IF NOT EXISTS 'tswow'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON *.* TO 'tswow'@'localhost' WITH GRANT OPTION;
FLUSH PRIVILEGES;
EOF
```

Leave the mysql server running. When doing a full build of tswow, it will download and import a TrinityCore database.

## Setup direnv

### Step 1: Configure direnv in Your Shell

Add to your `~/.zshrc` (or `~/.bashrc` if using bash):

```bash
eval "$(direnv hook zsh)"  # or: eval "$(direnv hook bash)"
```

Reload your shell:

```bash
source ~/.zshrc  # or source ~/.bashrc
```

### Step 2: Create .envrc File

Create a `.envrc` file in the root of your TSWoW project directory (the same directory where you cloned the repository). This file will configure the development environment automatically when you enter the directory.

Create the file:

```bash
cd /path/to/tswow-root  # Navigate to your TSWoW root directory
touch .envrc
```

Then add the following contents to `.envrc`:

```bash
# TSWoW macOS Development Environment

# Homebrew
HOMEBREW_PREFIX="${HOMEBREW_PREFIX:-/opt/homebrew}"
[ ! -d "$HOMEBREW_PREFIX" ] && HOMEBREW_PREFIX="/usr/local"
export PATH="$HOMEBREW_PREFIX/bin:$HOMEBREW_PREFIX/sbin:$PATH"

# Node.js 20.18.0 via nvm
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && source "$NVM_DIR/nvm.sh"
nvm use 20.18.0 2>/dev/null

# CMake 3.25.0
export PATH="/opt/cmake-3.25.0/CMake.app/Contents/bin:$PATH"

# Compilers
export CC="clang"
export CXX="clang++"

# Compiler flags for MSVC compatibility
# Disable PNG NEON optimizations to avoid linker errors on macOS ARM
# Use C++17 for Node.js v20 native modules (deasync) and general compatibility
export CXXFLAGS="-DSOL_NO_NIL=0 -w -fms-extensions -std=c++17 -DPNG_ARM_NEON_OPT=0"
export CPPFLAGS="-DSOL_NO_NIL=0 -DPNG_ARM_NEON_OPT=0"
export CMAKE_CXX_FLAGS="-DSOL_NO_NIL=0 -w -fms-extensions -std=c++17 -DPNG_ARM_NEON_OPT=0"

# C flags - use C99 standard (needed for BLPConverter) but allow implicit function declarations
# This is needed because LibTomMalloc/LibTomFree are used before being declared in StormLib
# Disable PNG NEON optimizations to avoid linker errors on macOS ARM
export CFLAGS="-std=c99 -Wno-error=implicit-function-declaration -Wno-implicit-function-declaration -Wno-error=int-conversion -Wno-int-conversion -DPNG_ARM_NEON_OPT=0"
export CMAKE_C_FLAGS="-std=c99 -Wno-error=implicit-function-declaration -Wno-implicit-function-declaration -Wno-error=int-conversion -Wno-int-conversion -DPNG_ARM_NEON_OPT=0"

# MySQL 8.4
export MYSQL_DIR="$HOMEBREW_PREFIX/opt/mysql@8.4"
export MYSQL_ROOT_DIR="$MYSQL_DIR"
export PATH="$MYSQL_DIR/bin:$PATH"
export MYSQL_INCLUDE_DIR="$MYSQL_DIR/include/mysql"
export MYSQL_LIBRARY="$MYSQL_DIR/lib/libmysqlclient.dylib"

# OpenSSL
export OPENSSL_DIR="$HOMEBREW_PREFIX/opt/openssl@3"
export PATH="$OPENSSL_DIR/bin:$PATH"
export OPENSSL_ROOT_DIR="$OPENSSL_DIR"
export OPENSSL_INCLUDE_DIR="$OPENSSL_DIR/include"
export PKG_CONFIG_PATH="$OPENSSL_DIR/lib/pkgconfig:$PKG_CONFIG_PATH"

# Boost 1.82.0
export BOOST_ROOT="/opt/boost_1_82_0"
export BOOST_INCLUDE_DIR="$BOOST_ROOT/include"
export BOOST_LIBRARY_DIR="$BOOST_ROOT/lib"
export CMAKE_PREFIX_PATH="$BOOST_ROOT:$MYSQL_DIR:${CMAKE_PREFIX_PATH:-}"
export Boost_NO_BOOST_CMAKE=ON

# Library paths
export LDFLAGS="-L$HOMEBREW_PREFIX/lib -L$MYSQL_DIR/lib"
export CPPFLAGS="$CPPFLAGS -I$HOMEBREW_PREFIX/include"
export DYLD_LIBRARY_PATH="$HOMEBREW_PREFIX/lib:$MYSQL_DIR/lib:${DYLD_LIBRARY_PATH:-}"
export LIBRARY_PATH="$HOMEBREW_PREFIX/lib:$MYSQL_DIR/lib:${LIBRARY_PATH:-}"
export PKG_CONFIG_PATH="$HOMEBREW_PREFIX/lib/pkgconfig:$PKG_CONFIG_PATH"
```

### Step 3: Allow direnv to Load the Configuration

After creating the `.envrc` file, you need to allow direnv to load it. This is a security feature that prevents direnv from automatically executing untrusted scripts.

Run the following command in your TSWoW root directory:

```bash
direnv allow
```

**Note:** You only need to run `direnv allow` once per `.envrc` file. If you modify the `.envrc` file later, direnv will prompt you to run `direnv allow` again to approve the changes.

After running `direnv allow`, direnv will automatically load the environment variables whenever you enter the directory, and unload them when you leave. You should see a message indicating that direnv is loading the `.envrc` file.

## Building TSWoW

When building TSWoW from source, we are concerned about three directories:

- **The source directory** is the root directory containing all source code used to build TSWoW.
- **The build directory** contains all intermediate files and build configurations.
- **The install directory** is where we install TSWoW.

The source, build and install directories should all be separate. Do not place any of them inside any of the others. The recommended setup is to have a `tswow-root` containing all three folders.

### Step 1: Clone the Repository

Run the following command (optionally in a new empty folder):

```bash
git clone https://github.com/tswow/tswow.git --recurse-submodules
```

This will create the source directory, called `tswow`.

**Note:** This download is expected to take some time. It is recommended to start developing on the latest release rather than the bleeding edge, as macOS is often only tested for new releases.

### Step 2: Configure Build Directories

Copy `source/build.default.yaml` to `source/build.yaml` and open it. Here you can configure where tswow should place build and install directories.

**Important:** Do not set install to point at your normal TSWoW installation unless you know what you're doing, it will frequently flush out all your settings and modules!

### Step 3: Install Dependencies

In the source directory, run:

```bash
cd tswow
npm install
```

### Step 4: Build TSWoW

In the source directory, run:

```bash
# For interactive build (recommended for first-time setup)
npm run build-interactive

# OR for building everything with a single command
npm run build
```

You should now have entered the main TSWoW build program. You can now build any components you want.

**Note:** All TypeScript for TSWoW and the transpiler is compiled automatically as long as the build program is running.

**Known Issue:** We currently have a bug where the prompt doesn't allow you to enter anything. Restarting the build script seems to fix this for now.

### Common Build Commands

- **Build everything:** `build full` - This will compile TrinityCore and all other components necessary for a fully working TSWoW installation.
- **Build only TrinityCore:** `build trinitycore-relwithdebinfo`

## Running TSWoW

### Step 1: Configure Client Path

Navigate to your install directory (typically `tswow-install` or `tswow/install`):

```bash
cd tswow-install  # or wherever your install directory is
```

Open `node.conf` and set the `Default.Client` path to point to your WoW 3.3.5a client:

```conf
Default.Client = "/path/to/your/wotlk/client"
```

**Important:** The path cannot contain spaces!

### Step 2: Start TSWoW

Make sure MySQL is running (see above), then:

```bash
npm start
```

This will start the TSWoW server. The first run may take some time as it sets up databases and applies client patches.

To create a gm account, type `create account myuser mypassword 3` (this commands requires that at least one worldserver is currently running)