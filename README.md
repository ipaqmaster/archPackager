## archPackager

This is a script I use to build AUR (And occasionally, official) packages for Archlinux including any AUR dependencies along the way.

It's designed to be used with Jenkins, which exposes some variables regarding a job's name and path for making some assumptions. The script attempts to determine the name of the package, its intended build architecture and repository name from the `JOB_NAME` variable when called by Jenkins or set manually. The script assumes that the repository name comes before the architecture.

An example jenkins job path: `JOB_NAME=Jenkins/myJobs/Repos/myRepo/x86_64/utilities/someProgram` will set the architecture to `x86_64` and repository name to `myRepo`. Without an exposed `PKGDEST` variable the script will assume `/repo` as the base directory. (`/repo/myRepo/x86_64/someProgram.pkg.tar.zst`).

The script can automatically detect when a package has already been built from the `./PKGBUILD` file of a repository and will check to see if it exists already. This check is skipped for `-git` packages, dependency packages (So makepkg can install the package after also determining it has already been built) and when (`-f`/`--f`/`-force`/`--force`) is specified.
