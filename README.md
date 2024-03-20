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

<!-- TODO: ## Example configuration -->

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
To ensure that hybrid packages install correctly, for example, it will be necessary to force the installation with `-fy` or bootstrap `pkg64` beforehand with `pkg64 bootstrap -fy`.

```shell
pkg64 install -fy curl git pot
```

### Pot (jails)

This lab utilises [Pot][pot], a third-party command-line BSD jail manager, to create a set of base, upstream, and downstream containers that provide isolated, "thick" environments for the runners.
Pot itself is a series of shell (sh) scripts that can be used to create, clone, and configure jails, control the extent of the system's isolation, and establish public or private bridges and IPs.

```shell
git -C $HOME clone https://github.com/digicatapult/pot
cd $HOME/pot
pot create-base -r 23.11
```

When a CheriBSD pot is created using this fork, it will attempt to fetch suitable OS manifests from [cheribsd.org][cheribsd.org] and extract the tarballs.
Yet it lacks any explicit reference to the host's libraries for certain ABIs, `/usr/lib64` (hybrid) and `/usr/lib64cb` (benchmark) respectively, hence these need to be copied into the jail via the use of Pot's flavours.
Flavours serve as modular components that Pot executes against each jail when it is first created or subsequently cloned.
Syntatically, a flavour is either a pot subcommand, such as `set-attr`, or itself a shell script that the jail will execute on initialisation.
In our case, we can use a flavour to bootstrap the jail with the requisite libraries, copying from the host first to `/root` and then using a script to copy from within the jail itself to `/usr/lib64`.

Our custom `./flavours/bootstrap` file might look like this:

```
copy-in -s /usr/lib64 -d /root/lib64
copy-in -s /usr/lib64cb -d /root/lib64cb
set-attribute -A no-tmpfs -V ON
set-attribute -A nullfs -V ON
set-attribute -A zfs -V ON
```

Disabling the temporary filesystem (`no-tmpfs`) is necessary to prevent any issues affecting the FS lock order when used in conjunction with OpenZFS, else [Witness][witness] will trigger a panic and the jail creation process will fail.
Assuming the above attributes are set, the libraries can then be copied via the jail's own terminal or by using a start-up script to achieve the same:

```shell
#!/bin/sh

# Install dependencies
cp -fR /root/lib64 /usr
cp -fR /root/lib64cb /usr
rm -R /root/lib64
rm -R /root/lib64cb

# ...
```

We should now be able to create a jail pair that fully supports `pkg64`,
`pkg64c`, and `pkg64cb`, with the following example:

```shell
pot create -p upstream -r 23.11 -t single \
    -f ./flavours/bootstrap
pot start -p upstream
pot exec -p upstream chmod 700 /root/copy.sh
pot exec -p upstream /root/copy.sh
pot exec -p upstream pkg64 -N
pot stop -p upstream
pot snapshot -p upstream
pot create -p downstream -P upstream
```

<!-- TODO: ### Act -->

### Example workflows

Simple user journey [here](docs/user-journey/simple-user-journey.md)
Example workflow of compiling and testing [here](docs/how-to/example-workflow-compiling-and-testing.md)
Example of failed buid and errors [here](docs/how-to/failed-build.md)

<!-- Links -->

[act-pot]: https://github.com/digicatapult/act-pot-cheribsd
[baseimage]: https://github.com/digicatapult/dsbd-github-baseimage
[cheri]: https://www.cl.cam.ac.uk/research/security/ctsrd/cheri
[cheribsd.org]: https://cheribsd.org/
[cheribsd]: https://github.com/CTSRD-CHERI/cheribsd
[cheribuild]: https://github.com/digicatapult/dsbd-cheribuild
[cicd]: /docs/images/cicd.svg
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
