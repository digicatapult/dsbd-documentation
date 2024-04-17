# DSbD Remote Lab

Digital Security by Design is an initiative funded by the UK government to advance [CHERI][cheri], a capability-based protection model, as the _de facto_ solution to unsafe memory access.
One tranche of funding has been allocated to the development of a remote laboratory, built on existing open-source CI/CD toolchains through the use of self-hosted runners.
These runners are established for GitHub Actions or GitLab CI API calls via a token generated for each repository, to run on [CheriBSD][cheribsd], a fork of FreeBSD.
These examples use a [QEMU][qemu] virtual machine operating in system mode, emulating an experimental 64-bit system-on-a-chip ([Morello][morello]) that implements memory-safe capabilities in silicon.

Please consult the [Code of Conduct][conduct] and [contributing guidelines][contributing] for additional information on how to participate further.

![cicd][cicd]

## Getting started

This CI/CD project has two build images, for the host and guest respectively, and several forked dependencies, all of which have been modified for the purposes of adapting them to CheriBSD or to our own infrastructure.

- [dsbd-github-baseimage][baseimage]: a HashiCorp Packer build stage for creating AWS and Azure host virtual machines, and then using cheribuild.py to create a guest CheriBSD system with additional configuration bundled via the `./extra-files` subdirectories
- [dsbd-cheribuild][cheribuild]: an all-in-one build script for CHERI-related projects, developed by the University of Cambridge and SRI International
- [pot][pot]: a third-party BSD jail manager
- [act-pot-cheribsd][act-pot]: a third-party set of scripts to install and manage jailed runners

Before this project commenced, we had previously developed a physical CheriBSD demonstrator and prepared documentation around it and the ecosystem as it existed between 2022 and 2023. Some documentation may therefore be outdated, but should be useful still for background context.

- [dsbd-demonstrator][demonstrator]
- [dsbd-getting-started][start]

### Example workflows

- Simple user journey [here](./docs/user-journey/simple-user-journey.md)
- Example workflow of compiling and testing [here](./docs/how-to/example-workflow-compiling-and-testing.md)
- Example of failed buid and errors [here](./docs/how-to/failed-build.md)

## Components

### cheribuild.py (QEMU/OS/SDK)

This solution's architecture depends on a third-party build script, [cheribuild.py][cheribuild], to instantiate the minimal required artefacts for CheriBSD, such as QEMU, [LLVM][llvm], and the guest's root BSD filesystem.
A typical build with `./cheribuild.py` will specify individual components, `qemu`, or entire systems with prefixed architectures and kernels: `run-morello-purecap` or `run-riscv64-purecap`.

```bash
./cheribuild.py --build qemu --include-dependencies
./cheribuild.py --build disk-image-morello-purecap \
    --disk-image/path "/output/cheribsd-morello-purecap.zfs.img" \
    --disk-image/rootfs-type zfs \
    --include-dependencies
```

When the above disk image has been created, there will be a QEMU system mode binary residing within the target directory, and various additional parameters can be passed when executing it.
Our example below is specific to the Morello CPU architecture, taking a single thread from the host (`smp`), reserving 4 GB of RAM, and instantiating its own network interface (`netdev`) for SSH from ports 10005 to 22.

```shell
/output/sdk/bin/qemu-system-morello \
    -M virt,gic-version=3 -cpu morello \
    -bios edk2-aarch64-code.fd \
    -smp 1 -m 4096 -nographic \
    -drive if=none,file=/output/cheribsd-morello-purecap.zfs.qcow2,id=drv0,format=qcow2 \
    -device virtio-blk-pci,drive=drv0 \
    -device virtio-net-pci,netdev=net0 \
    -netdev user,id=net0,hostfwd=tcp::10005-:22
```

There are now three ABIs specific to CheriBSD: "hybrid", "purecap" (CheriABI), and "benchmark", each with their own flavour of FreeBSD's package manager and their own libraries.
Bootstrapping either pkg64, pkg64c, or pkg64cb will only be necessary once for the host, and will occur automatically when any of them are used to install packages.

```shell
pkg64 install -y curl git pot
```

### Pot (jails)

This lab utilises [Pot][pot], a third-party command-line BSD jail manager, to create a set of base, upstream, and downstream containers that provide isolated, "thick" environments for the runners.
Pot itself is a series of shell (sh) scripts that can be used to create, clone, and configure jails, control the extent of the system's isolation, and establish public or private bridges and IPs.

```shell
git clone https://github.com/digicatapult/pot
cd pot
for dir in bin etc share; do
    cp -fR ./$dir /usr/local64/
done
pot init
```

When a CheriBSD pot is created using this fork, it will attempt to fetch suitable OS manifests from [cheribsd.org][cheribsd.org] and extract the tarballs. They can also be downloaded manually:

```shell
manifests="/usr/local/share/freebsd/MANIFESTS"
mkdir -p $manifests
releases=$(curl -sS "https://download.cheribsd.org/releases/arm64/aarch64c/" | grep -Eo "\w{1,}\.\w{1,}" | sort -u)
for release in $releases; do
    curl -sS -C - "https://download.cheribsd.org/releases/arm64/aarch64c/$release/ftp/MANIFEST" > "$manifests/arm64-aarch64c-$release-RELEASE"
done
```

Pots lack any explicit reference to the host's libraries for certain ABIs, `/usr/lib64` (hybrid) and `/usr/lib64cb` (benchmark) respectively, hence these need to be copied into the jail via the use of Pot's "flavours".
Flavours serve as modular components that Pot executes against each jail when it is first created or subsequently cloned.
Syntatically, a flavour is either a pot subcommand, such as `set-attr`, or itself a shell script that the jail will execute on initialisation.
In our case, we can use a flavour to bootstrap the jail with the requisite libraries, copying from the host first to `/root` and then using a script to copy from within the jail itself to `/usr/lib64`.

Our custom `./flavours/bootstrap` file might look like this:

```
copy-in -s /usr/lib64 -d /root/lib64
copy-in -s /usr/lib64cb -d /root/lib64cb
set-attribute -A no-tmpfs -V ON
```

Disabling the temporary filesystem (`no-tmpfs`) is necessary to prevent any issues affecting the FS lock order when used in conjunction with OpenZFS, else [Witness][witness] will trigger a panic and the jail creation process will fail.
Assuming the above attributes are set, the libraries can then be copied via the jail's own terminal or by using a start-up script to achieve the same.
We should now be able to create a jail pair that fully supports `pkg64`, `pkg64c`, and `pkg64cb`, with the following example:

```shell
pot create -p sibling -b 23.11 -t single -f bootstrap
pot start -p sibling
pot exec -p sibling cp -R /root/lib64 /usr
pot exec -p sibling cp -R /root/lib64cb /usr
pot exec -p sibling rm -rdf /root/lib64
pot exec -p sibling rm -rdf /root/lib64cb
pot exec -p sibling pkg64 install -y bash curl git node readline
pot stop -p sibling
pot snap -p sibling
```

After bootstrapping an upstream pot, we can move to installing the scripts to execute runners.

### Act (GitHub self-hosted runners)

We next pass configuration variables and download the [github-act-runner][github-act-runner] binary to individual pots using [act-pot-cheribsd][act-pot], a set of wrapper scripts that help to automatically create, rename, and destroy the jails as required.
Installing the scripts to the CheriBSD guest is straightforward:

```shell
git clone https://github.com/digicatapult/act-pot-cheribsd
cd act-pot-cheribsd
./install.sh
```

We are now set to configure and start the first runner.

```shell
./config.sh --url [GITHUB_ORG] --token [GITHUB_TOKEN]
sysrc gh_actions_enable="YES"
./jobs/restart_actions.sh
```

This last srcipt can be run manually or via `crontab` to automate the initialisation of ephemeral pots when the guest has finished booting.
It merely ensures that the system variable (rcvar) `gh_actions_pots` always has the most up-to-date pot names.

At this point, the guest system is now ready to process a GitHub workflow.

<!-- Links -->

[act-pot]: https://github.com/digicatapult/act-pot-cheribsd
[baseimage]: https://github.com/digicatapult/dsbd-github-baseimage
[cheri]: https://www.cl.cam.ac.uk/research/security/ctsrd/cheri
[cheribsd.org]: https://cheribsd.org/
[cheribsd]: https://github.com/CTSRD-CHERI/cheribsd
[cheribuild]: https://github.com/digicatapult/dsbd-cheribuild
[cicd]: ./docs/images/cicd.svg
[conduct]: /CODE_OF_CONDUCT.md
[contributing]: /CONTRIBUTING.md
[demonstrator]: https://github.com/digicatapult/dsbd-demonstrator
[start]: https://github.com/digicatapult/dsbd-getting-started
[llvm]: https://llvm.org/
[morello]: https://www.morello-project.org/
[pot]: https://github.com/digicatapult/pot
[qemu]: https://www.qemu.org/
[readme]: /README.md
[witness]: https://man.freebsd.org/cgi/man.cgi?query=witness
[github-act-runner]: https://github.com/ChristopherHX/github-act-runner
[act-pot]: https://github.com/digicatapult/act-pot-cheribsd
