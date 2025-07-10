## archPackager

### About 

This is a collection of scripts I use to build AUR (And occasionally, official) packages for Archlinux including any AUR dependencies along the way. The main script takes advantage of docker for cleanbuilding and existing AUR dependency builds to save on resources. By default it uses all available cores for makepkg (`$(nproc)`)

The main script is designed to be used with Jenkins which exposes some variables regarding a job's name and path for making some assumptions. It attempts to figure out the package name, its intended build architecture and repository name from the `JOB_NAME` variable when called by Jenkins (or set manually). It also assumes that the repository name comes right before the architecture.

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


### Example usage of these scripts

#### Linux, official Archlinux package

To build https://gitlab.archlinux.org/archlinux/packaging/packages/linux we need to define a new job in a repository directory.

This could look like: `Archlinux/myOfficialPackageRepo/x86_64/linux`

In the `linux` job, we should set the repository URL to: `https://gitlab.archlinux.org/archlinux/packaging/packages/linux.git`.

Under the `Build Triggers` section we can set `Poll SCM` to something like `H H(0-5) * * *` to poll this git repository daily anywhere from midnight to 5am for building this package.

Personally I delete the worksapce before and after the build to address storage concerns (Cleanbuilding is also a good idea in general).

Notice the repo has a `keys/pgp` directory which contains three keys.

After cloning this repository to the home directory of the jenkins build server user invoke it with an `Execute shell` option under `Build Steps` containing: `$HOME/archPackager/build_package # incomplete`

This would do for most packages but given this repository needs to trust three externally signed pgp public keys we must describe them to the script so the build process can trust them when verifying key files of the build: `$HOME/archPackager/build_package --key 647F28654894E3BD457199BE38DBBDC86092693E --key 83BC8889351B5DEBBB68416EB8AC08600F108CDF --key ABAF11C65A2970B130ABE3C479BE3E4300411886`

Making sure the directories `/repo/myOfficialPackageRepo/x86_64` exists the job should now be capable of building a package. Make sure the jenkins user has been added to the `docker` group and that the service is running.


#### Nightly archlinux:latest docker image preparation

This script can be invoked either manually at will or through a new Jenkins job. If you build a lot of packages I highly recommend using this method to reduce network consumption and preserve local resources working from this base image rather than reinstalling packages for each build container.

1. Create a new job in Docker of any name, preferably: `docker_update_archlinux`
2. Schedule it to "Build periodically" at midnight using value: `0 0 * * *` (Midnightly)
3. After cloning this repo to the homedir of the `jenkins` user, create a new `Execute shell` option under `Build Steps` with this content: `~/archPackager/docker_update_archlinux`
4. Save and run the job.

#### repo_prune

The ./repo_prune script is used for culling older packages as time goes on to save on disk space. A new job can be created which just calls out to ./repo_prune and it will assume the top level directory `/repo` by default. Otherwise a repo top level directory can be given as an argument. This script serves to cull old package files as new ones come in.


#### repo_sign

This script is intended to be called after building a package so that a .sig can be generated with your designated package signing key.

Create a Jenkins job (anywhere) to call this script and make sure you define a secure `gpg_passphrase` passphrase variable for the script to use for signing. Having this script separate ensures build tasks have no access to the signing process.

By default it signs packages without a signature. With `--verify` it verifies and re-signs packages in the event of a forced build but ignores packages over 100MB in size. With both `--verify` and `--force` it will verify all packages including larger sized ones.

Ideally you should never have to run `--verify` and/or `--force` however some packages such as python-based packages may need to be re-built as newer versions of python roll out for Archlinux. I am working on automatic python package detection so a epoch value can be set based on the python version - to avoid overwriting older pkg files in-place.

#### Building and using an aarch64 container

`CARCH=aarch64 ./docker_update_archlinux` needs to be in your system's `docker` group and requires the following sudoers configuration to prepare the host system to pacstrap archlinuxarm:

```
jenkins ALL=(ALL) NOPASSWD: /usr/bin/pacman-key --add archlinuxarm.gpg
jenkins ALL=(ALL) NOPASSWD: /usr/bin/pacman-key --lsign-key 9D22B7BB678DC056B1F7723CB55C5315DCD9EE1A
jenkins ALL=(ALL) NOPASSWD: /usr/bin/pacman-key --lsign-key 02922214DE8981D14DC2ACABBC704E86B823CD25
jenkins ALL=(ALL) NOPASSWD: /usr/bin/pacman-key --lsign-key 69DD6C8FD314223E14362848BF7EEF7A9C6B5765
jenkins ALL=(ALL) NOPASSWD: /usr/bin/pacman-key --lsign-key 68B3537F39A313B3E574D06777193F152BDBE6A6

jenkins ALL=(ALL) NOPASSWD: /sbin/mkdir -p /var/cache/pacman/pkg/aarch64
jenkins ALL=(ALL) NOPASSWD: /sbin/mount --bind /var/cache/pacman/pkg/aarch64 /tmp/aarch64_build/var/cache/pacman/pkg
jenkins ALL=(ALL) NOPASSWD: /sbin/pacman-key --populate archlinuxarm
jenkins ALL=(ALL) NOPASSWD: /sbin/pacman-key --lsign-key archlinuxarm
jenkins ALL=(ALL) NOPASSWD: /sbin/pacstrap -M -K -C * /tmp/aarch64_build archlinuxarm-keyring base git sudo
jenkins ALL=(ALL) NOPASSWD: /sbin/tar -czf *arch-rootfs-aarch64.tar.gz .
jenkins ALL=(ALL) NOPASSWD: /sbin/umount /tmp/aarch64_build/var/cache/pacman/pkg
jenkins ALL=(ALL) NOPASSWD: /sbin/pacman --needed --noconfirm -S qemu-user-static-binfmt qemu-user-static
jenkins ALL=(ALL) NOPASSWD: /sbin/cp -nv /usr/lib/binfmt.d/qemu-aarch64-static.conf /etc/binfmt.d/
jenkins ALL=(ALL) NOPASSWD: /sbin/rm -rf /tmp/aarch64_build
```

#### SUID

The build user inside the containers needs sudo to grab build dependencies with pacman. By default `/proc/sys/fs/binfmt_misc/qemu-aarch64` will usually have the `F` flag set which is `nosetuid`. This disallows SUID usage and causes problems with the current design of the build_package script. I will probably fix this later down the line.

In the meantime suid in qemu-aarch64-static can be enabled by editing /etc/binfmt.d/qemu-aarch64-static.conf and appending the flags `O` and `C`. Refresh the host's aarch64 binfmt definition by dropping it with `echo -1 | sudo tee /proc/sys/fs/binfmt_misc/qemu-aarch64` and re-registering it with the `OC` flags appended: `cat /etc/binfmt.d/qemu-aarch64-static.conf | sudo tee /proc/sys/fs/binfmt_misc/register`
