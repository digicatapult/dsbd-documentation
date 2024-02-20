# DSbD Contributing Guide

Thank you for your interest and willingness to contribute to the wider security ecosystem, and to Digital Catapult's on-going community projects around the Digital Security by Design programme.

This document provides an overview of the workflow, from raising issues to merging pull requests. Please consult our separate [Code of Conduct][conduct] for guidelines on how to engage others appropriately. For an overview of the DSbD Remote Lab, please see the project's [README][readme] file.


## Getting started

This CI/CD project has two build images, for the host and guest respectively, and several forked dependencies, all of which have been modified for the purposes of adapting it to CheriBSD or to our own infrastructure.
- [dsbd-github-baseimage][baseimage]: a HashiCorp Packer build stage for creating AWS and Azure host virtual machines, and then using cheribuild.py to create a guest CheriBSD system with additional configuration bundled via the `./extra-files` subdirectories
- [dsbd-cheribuild][cheribuild]: an all-in-one build script for CHERI-related projects, developed by the University of Cambridge and SRI International
- [pot][pot]: a third-party BSD jail manager
- [act-pot-cheribsd][act-pot]: a third-party set of scripts to install and manage jailed runners

Before this project commenced, we had previously developed a physical CheriBSD demonstrator and prepared documentation around it and the ecosystem as it existed between 2022 and 2023. Some documentation may therefore be outdated, but should be useful still for background context.
- [dsbd-demonstrator][demonstrator]
- [dsbd-getting-started][getting-started]

<!-- Links -->
[act-pot]:         https://github.com/digicatapult/act-pot-cheribsd
[baseimage]:       https://github.com/digicatapult/dsbd-github-baseimage
[cheribuild]:      https://github.com/digicatapult/dsbd-cheribuild
[conduct]:         /CODE_OF_CONDUCT.md
[demonstrator]:    https://github.com/digicatapult/dsbd-demonstrator
[getting-started]: https://github.com/digicatapult/dsbd-getting-started
[pot]:             https://github.com/digicatapult/pot
[readme]:          /README.md
