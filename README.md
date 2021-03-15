# Terraform provider registry to GitHub pages

A GitHub Action to generate Terraform Provider registry metadata and publish it as GitHub pages static website. So GitHub pages site becomes you private Terraform provider registry.

Action requires similar workflow as terraform-provider-scaffolding [uses](https://github.com/hashicorp/terraform-provider-scaffolding/blob/main/.github/workflows/release.yml), with notable excpetions:
* We require KeyID and ASCII Armored Public key, you can use [this](https://github.com/AbsaOSS/ghaction-import-gpg) action.
* Drop `--rm-dist` goreleaser's flag (action depend on binaries goreleaser builds)

# Terraform Provider release Workflow

* GPG key import
* goreleaser release binaries into Github Release
* terraform-registry-generator walks through `artifacts_dir` to collect info about release, such as provider name and version. Then generate required metadata and merge with existing (if any) versions file. As the last step it commits and pushes changes into GitHub pages branch of given repository.

# Usage

Inputs:
* `token` The GitHub token with write access to the target repository (required)
* `artifacts_dir` Directory which holds provider artifacts, defaults to `dist`
* `repository` The GitHub repository to store provider registry metadata, defaults to `<GITHUB_REPOSITORY>`
* `branch` The branch to publish artifacts, defaults to `gh-pages`
* `username` The user name used as the commit user, defaults to `<GITHUB_ACTOR>`
* `email` The user's email used as the commit email, defaults to `<GITHUB_ACTOR>@users.noreply.github.com`
* `namespace` Terraform provider namespace, defaults to `<GITHUB_REPOSITORY_OWNER>` 
* `gpg_keyid` Key id used to sign provider (required)
* `gpg_ascii_armor`ASCII Armored Public key to verify provider shasums (required)
* `generator_version` Version of terraform metadata generator to use, defaults to 0.1
    default: 0.1

## Example

Generates provider repository metadata for provider binaries in `./dist`:

```yaml
name: release

on:
  push:
    tags: '*'

jobs:
  goreleaser:
    runs-on: ubuntu-20.04
    steps:
      - name: Checkout
        uses: actions/checkout@v2
      - name: Set up Go
        uses: actions/setup-go@v2
        with:
          go-version: 1.16
      - name: Import GPG key
        id: import_gpg
        uses: absaoss/ghaction-import-gpg@main
        env:
          GPG_PRIVATE_KEY: ${{ secrets.GPG_PRIVATE_KEY }}
          PASSPHRASE: ${{ secrets.PASSPHRASE }}
      - name: Run GoReleaser
        uses: goreleaser/goreleaser-action@v2
        with:
          version: latest
          args: release
        env:
          GPG_FINGERPRINT: ${{ steps.import_gpg.outputs.keyid }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      - uses: AbsaOSS/ghpages-to-tf-provider-registry@v0.1
        with:
          token: ${{ secrets.GH_REG_TOKEN }}
          artifacts_dir: dist
          gpg_keyid: ${{ steps.import_gpg.outputs.keyid }}
          gpg_ascii_armor: ${{ steps.import_gpg.outputs.pub_ascii_armor }}
```
