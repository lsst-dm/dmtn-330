[![Website](https://img.shields.io/badge/dmtn--330-lsst.io-brightgreen.svg)](https://dmtn-330.lsst.io)
[![CI](https://github.com/lsst-dm/dmtn-330/actions/workflows/ci.yaml/badge.svg)](https://github.com/lsst-dm/dmtn-330/actions/workflows/ci.yaml)

# Publication flow for Butler prompt data products

## DMTN-330

Describes the infrastructure supporting the end-user prompt data products Butler repository and the process of making data available in it.

**Links:**

- Publication URL: https://dmtn-330.lsst.io
- Alternative editions: https://dmtn-330.lsst.io/v
- GitHub repository: https://github.com/lsst-dm/dmtn-330
- Build system: https://github.com/lsst-dm/dmtn-330/actions/


## Build this technical note

You can clone this repository and build the technote locally if your system has Python 3.11 or later:

```sh
git clone https://github.com/lsst-dm/dmtn-330
cd dmtn-330
make init
make html
```

Repeat the `make html` command to rebuild the technote after making changes.
If you need to delete any intermediate files for a clean build, run `make clean`.

The built technote is located at `_build/html/index.html`.

## Publishing changes to the web

This technote is published to https://dmtn-330.lsst.io whenever you push changes to the `main` branch on GitHub.
When you push changes to a another branch, a preview of the technote is published to https://dmtn-330.lsst.io/v.

## Editing this technical note

The main content of this technote is in `index.md` (a Markdown file parsed as [CommonMark/MyST](https://myst-parser.readthedocs.io/en/latest/index.html)).
Metadata and configuration is in the `technote.toml` file.
For guidance on creating content and information about specifying metadata and configuration, see the Documenteer documentation: https://documenteer.lsst.io/technotes.
