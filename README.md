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

---------

Any additional/unrecognised arguments are passed to the makepkg process. There are a few flags reserved for script recursive use during dependency builds.


### Examples

#### Linux, official Archlinux package

To build https://gitlab.archlinux.org/archlinux/packaging/packages/linux we need to define a new job in a repository directory.

This could look like: `Archlinux/myOfficialPackageRepo/x86_64/linux`

In the `linux` job, we should set the repository URL to: `https://gitlab.archlinux.org/archlinux/packaging/packages/linux.git`.

Under the `Build Triggers` section we can set `Poll SCM` to something like `H H(0-5) * * *` to poll this git repository daily anywhere from midnight to 5am for building this package.

Personally I delete the worksapce before and after the build to address storage concerns (Cleanbuilding is also a good idea in general).

Notice the repo has a `keys/pgp` directory which contains three keys.

After cloning this repository to the home directory of the jenkins build server user invoke it with an Execute shell Build Step which contains: `$HOME/archPackager/build_package # incomplete`

This would do for most packages but given this repository needs to trust three externally signed pgp public keys we must describe them to the scriptso it can trust them for the duration of the build: `$HOME/archPackager/build_package --key 647F28654894E3BD457199BE38DBBDC86092693E --key 83BC8889351B5DEBBB68416EB8AC08600F108CDF --key ABAF11C65A2970B130ABE3C479BE3E4300411886`

Making sure the directories `/repo/myOfficialPackageRepo/x86_64` exists the job should now be capable of building a package. Make sure the jenkins user has been added to the `docker` group and that the service is running.
