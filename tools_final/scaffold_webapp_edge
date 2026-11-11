#!/usr/bin/env sh
#Copyright (C) 2026 by Theron Kauk and John Morris Beck <hdconfigure.direct@gmail.com> THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR ANY SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR PROFITS, WHETHER IN AN ACTION OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS ACTION, ARISING OUT OF OR IN CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE.

# scaffold_webapp_edge.sh
#
# Makes sure Node.js and npm are available (installing them from Debian's
# apt repositories if they are missing), checks that Node is new enough, then
# installs the monorepo's workspace dependencies.
#
# Design notes:
#   - This script never changes directory. The project is targeted with
#     "npm --prefix <dir>", which makes npm treat <dir> as the project root.
#   - All messages (errors and status) go to stderr, and so does the output of
#     the tools it runs (apt-get, npm), so stdout is left empty.
#   - Any failure exits with a non-zero status (set -eu plus explicit checks).
#   - Node comes from ONE source: Debian's own packages. The script does not
#     run "npm install -g npm@latest": that rewrites files dpkg owns, can break
#     on a later "apt upgrade", and makes installs non-reproducible. If Debian's
#     Node is too old for the project, the script stops and says so instead of
#     mixing in a second Node source (NodeSource, nvm).
#   - The minimum Node version is read from the project's .nvmrc (the single
#     source of truth, kept in sync with "engines" in package.json). If .nvmrc
#     is missing or unparsable, the defaults below are used.
#   - This script writes no files of its own. npm creates package-lock.json
#     (machine-generated JSON, not hand-edited) and node_modules/, so there is
#     nothing to annotate and no companion files are needed.
#
# Usage: scaffold_webapp_edge.sh [target_directory]   (default: current directory)

set -eu

# Fallback minimum Node version, used only when .nvmrc cannot be read.
DEFAULT_NODE_MAJOR=20
DEFAULT_NODE_MINOR=19

TARGET_DIR="${1:-.}"

# --- Helpers -----------------------------------------------------------------

# say <message>: status message to stderr
say() {
  echo "$@" >&2
}

# die <message>: error message to stderr, then exit with failure
die() {
  echo "Error: $*" >&2
  exit 1
}

# as_root <command...>: run a command as root, using sudo when not already root
as_root() {
  if [ "$(id -u)" -eq 0 ]; then
    "$@"
  else
    sudo "$@"
  fi
}

# is_number <string>: succeeds if the string is a non-empty run of digits
is_number() {
  case "$1" in
    '' | *[!0-9]*) return 1 ;;
    *) return 0 ;;
  esac
}

# --- Preconditions -----------------------------------------------------------

[ -d "$TARGET_DIR" ] || die "Target directory '$TARGET_DIR' does not exist."

[ -f "$TARGET_DIR/package.json" ] ||
  die "'$TARGET_DIR' has no package.json. Run create_webapp_repo.sh first, or pass the project root."

# Minimum Node version: from .nvmrc ("20.19", "v20.19.2" or "20"), else defaults
min_major="$DEFAULT_NODE_MAJOR"
min_minor="$DEFAULT_NODE_MINOR"
if [ -f "$TARGET_DIR/.nvmrc" ]; then
  nvmrc_line=""
  IFS= read -r nvmrc_line < "$TARGET_DIR/.nvmrc" || true
  nvmrc_version="${nvmrc_line#v}"
  nvmrc_major="${nvmrc_version%%.*}"
  case "$nvmrc_version" in
    *.*) nvmrc_rest="${nvmrc_version#*.}"; nvmrc_minor="${nvmrc_rest%%.*}" ;;
    *) nvmrc_minor=0 ;;
  esac
  if is_number "$nvmrc_major" && is_number "$nvmrc_minor"; then
    min_major="$nvmrc_major"
    min_minor="$nvmrc_minor"
  else
    say "Warning: could not parse '$TARGET_DIR/.nvmrc'; requiring Node >= $min_major.$min_minor."
  fi
fi

# --- 1. Node.js and npm ------------------------------------------------------

if ! command -v node >/dev/null 2>&1 || ! command -v npm >/dev/null 2>&1; then
  say "Node.js and/or npm not found. Installing from Debian packages..."

  command -v apt-get >/dev/null 2>&1 ||
    die "apt-get not found. This script supports Debian-based systems; install Node.js >= $min_major.$min_minor and npm manually."

  if [ "$(id -u)" -ne 0 ]; then
    command -v sudo >/dev/null 2>&1 ||
      die "Root privileges are required to install packages, but sudo was not found. Re-run as root."
    say "Root privileges required for package installation. Using sudo..."
  fi

  as_root apt-get update >&2 || die "apt-get update failed."
  as_root apt-get install -y --no-install-recommends nodejs npm >&2 ||
    die "Failed to install nodejs and npm via apt-get."

  command -v node >/dev/null 2>&1 || die "node is still missing after installation."
  command -v npm >/dev/null 2>&1 || die "npm is still missing after installation."
else
  say "Node.js and npm are already installed."
fi

# --- 2. Verify the Node version ----------------------------------------------

node_version="$(node --version)" || die "Could not run 'node --version'."
version_digits="${node_version#v}"
node_major="${version_digits%%.*}"
version_rest="${version_digits#*.}"
node_minor="${version_rest%%.*}"

if ! is_number "$node_major" || ! is_number "$node_minor"; then
  die "Could not parse Node version '$node_version'."
fi

if [ "$node_major" -lt "$min_major" ] ||
   { [ "$node_major" -eq "$min_major" ] && [ "$node_minor" -lt "$min_minor" ]; }; then
  die "Node $node_version is older than the required $min_major.$min_minor. Use a newer Debian release, or install a newer Node from a single upstream source (NodeSource or nvm) instead of Debian's, then re-run."
fi

npm_version="$(npm --version)" || die "Could not run 'npm --version'."
say "Using Node $node_version and npm $npm_version."

# --- 3. Workspace dependencies -----------------------------------------------

# With a lockfile and no node_modules, "npm ci" gives a reproducible install and
# fails loudly if package.json and the lockfile disagree. Otherwise "npm install"
# creates the lockfile on first run, or verifies and refreshes workspace links.
if [ -f "$TARGET_DIR/package-lock.json" ] && [ ! -d "$TARGET_DIR/node_modules" ]; then
  say "Installing workspace dependencies from the lockfile in '$TARGET_DIR'..."
  npm ci --prefix "$TARGET_DIR" >&2 || die "Failed to install workspace dependencies (npm ci)."
elif [ -d "$TARGET_DIR/node_modules" ]; then
  say "node_modules already exists in '$TARGET_DIR'."
  say "Verifying and updating workspace links..."
  npm install --prefix "$TARGET_DIR" >&2 || die "Failed to update workspace dependencies."
else
  say "Installing workspace dependencies in '$TARGET_DIR'..."
  npm install --prefix "$TARGET_DIR" >&2 || die "Failed to install workspace dependencies."
fi

[ -f "$TARGET_DIR/package-lock.json" ] ||
  say "Note: no package-lock.json was produced; commit one for reproducible installs."

say "Dependency setup and verification complete."
say "Next, from inside '$TARGET_DIR': npm run check"
