## Building Vanilla Image

To build `macos-sonoma-vanilla`:

```bash
packer build templates/vanilla-sonoma.pkr.hcl
```

Optionally, SIP can be disabled for each image by running the following commands:

```bash
packer build -var vm_name=sonoma-vanilla templates/disable-sip.pkr.hcl
```

## Building Base Image

```bash
packer build -var vm_name=sonoma-vanilla templates/base.pkr.hcl
```

## Building Xcode Image

```bash
packer build -var macos_version=sonoma -var xcode_version="[15.4]" templates/xcode.pkr.hcl
```

## Building Xcode 27.1 RC locally

Use an Apple Silicon Mac with Homebrew and enough RAM for the template's 8 GB
guest. Allow space for the base image, Xcode archive, simulator downloads, and a
240 GB virtual disk (the disk grows as it is used). The Tahoe guest must run
macOS 26.6 or later, as required by
[Xcode 27.1 RC](https://developer.apple.com/documentation/xcode-release-notes/xcode-27_1-release-notes).

Run these commands from the repository root:

```bash
brew install cirruslabs/cli/tart
brew tap hashicorp/tap
brew install hashicorp/tap/packer
mkdir -p "$HOME/XcodesCache"
```

Download Xcode 27.1 RC (build 27A9275) from
[Apple Developer Downloads](https://developer.apple.com/download/all/?q=Xcode%2027.1)
after signing in with your Apple ID. Save or copy the `.xip` archive to exactly
`~/XcodesCache/Xcode_27.1-rc.xip`. Packer copies this host archive into the VM;
installing Xcode on the host is unnecessary.

```bash
test -s "$HOME/XcodesCache/Xcode_27.1-rc.xip"
tart pull ghcr.io/cirruslabs/macos-tahoe-base:latest
packer init templates/xcode.pkr.hcl
packer build \
  -var macos_version=tahoe \
  -var 'xcode_version=["27.1-rc"]' \
  -var 'xcode_components=["MetalToolchain"]' \
  -var disk_size=240 \
  templates/xcode.pkr.hcl

packer init templates/exex-script.pkr.hcl
packer build \
  -var 'vm_name=tahoe-xcode:27.1-rc' \
  -var script_path=scripts/finalize-tahoe.sh \
  templates/exex-script.pkr.hcl

tart run tahoe-xcode:27.1-rc
```

The result is a local Tart VM named `tahoe-xcode:27.1-rc`, with guest credentials
`admin` / `admin`. Provisioning needs internet access for Homebrew packages and
Xcode components. This builds the single-Xcode image; the CI runner image also
includes the older toolchains and runtimes.

To reproduce the Tahoe CI runner, first cache **every** archive in the list
below as `~/XcodesCache/Xcode_<version>.xip`, including the RC archive. Then run:

```bash
packer build \
  -var macos_version=tahoe \
  -var tag=runner \
  -var 'xcode_version=["27.1-rc","27","26.6","27-beta-6","26.5","27-beta","26.4.1","26.3","26.2","26.1.1","26.0.1"]' \
  -var 'additional_ios_builds=["18.6"]' \
  -var 'xcode_components=["MetalToolchain"]' \
  -var disk_size=520 \
  -var disk_free_mb=100000 \
  -var expected_runtimes_file=data/expected.tahoe.runtimes.txt \
  templates/xcode.pkr.hcl

packer build \
  -var 'vm_name=tahoe-xcode:runner' \
  -var script_path=scripts/finalize-tahoe.sh \
  templates/exex-script.pkr.hcl

tart run tahoe-xcode:runner
```

This version needs substantially more host storage and enforces CI's 100 GB
free-space check inside the guest. Both builds keep the image locally; publishing
requires a separate `tart push` command.
