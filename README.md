## archPackager

### About 

This is a script I use to build AUR (And occasionally, official) packages for Archlinux including any AUR dependencies along the way. It takes advantage of docker for cleanbuilding and will take advantage of existing AUR dependency builds to save resources. By default this script uses all available cores for makepkg (`$(nproc)`)

It's designed to be used with Jenkins, which exposes some variables regarding a job's name and path for making some assumptions. The script attempts to determine the name of the package, its intended build architecture and repository name from the `JOB_NAME` variable when called by Jenkins or set manually. The script assumes that the repository name comes before the architecture.

An example jenkins job path: `JOB_NAME=Jenkins/myJobs/Repos/myRepo/x86_64/utilities/someProgram` will set the architecture to `x86_64` and repository name to `myRepo`. Without an exposed `PKGDEST` variable the script will assume `/repo` as the base directory. (`/repo/myRepo/x86_64/someProgram.pkg.tar.zst`).

The script can automatically detect when a package has already been built from the `./PKGBUILD` file of a repository and will check to see if it exists already. This check is skipped for `-git` packages, dependency packages (So makepkg can install the package after also determining it has already been built) and when (`-f`/`--f`/`-force`/`--force`) is specified.


### Intended usage

This script intends to be called by its full path while in the working directory of an AUR (or official) package with a `./PKGBUILD` file present. The script uses the `archlinux:latest` docker image to get started, which can be updated by running `./docker_update_archlinux` at any time, which is intended to update the latest image to current packaged releases to avoid additional work per container. 

It automatically notifies and exits when it believes an AUR package has been promoted to the official repositories, skipping this check if building official packages.

### Arguments

`-f`/`-force`/`--f`/`--force`

Force a package to build even if a package file exists for it already

`--key`

Add and locally sign a key with `gpg` for trusting sources of a package

`--pacmankey`

Add and locally a key with `pacman-key` for trusting packages.
