# exmikupro/input

Mio CIL INPUT Bunko

Version: **0.1.0**  
License: **MIT** (see [LICENSE](LICENSE))  
Supported publishing targets: **macos/arm64**

## Develop and test

```text
package.mio          Public package source (hello is an initial example)
package_test.mio     Tests using kiso/testing
bunko.toml           Exact local dependency declarations
bunko.package.toml   Local package metadata and selected platforms
README.md            Development and publishing guide
LICENSE              Redistribution terms
.gitignore           Development output exclusions
```

```sh
mio check .
mio test .
```

Add meaningful tests in `*_test.mio` files. Each `test_` function must take no
parameters and return `unit`. Include `kiso/testing` assertions. Bunko requires
at least one passed test and no failures; an empty test run is insufficient.
Keep tests outside hidden directories and `build`, `dist`, `node_modules`,
`vendor`, `cache` and `tmp`. Declare exact direct dependencies in `bunko.toml`.
Scaffolding does not install them. Use a compiler that supports explicit Bunko
installation when dependencies are needed; native installation availability
is separate from local package creation.

## Publish to the official Bunko Index

This is a local development project, **not a published package**. Creation does
not log in to GitHub, initialize Git, create a remote repository, push code,
create a release or submit an index PR. Perform the following steps yourself.
There is no Bunko publishing command in this guide.

### 1. Prepare the upstream repository

Create a public GitHub repository owned by the lowercase GitHub account or
organization `exmikupro` (or obtain verifiable publisher delegation).
Reserved namespaces require the index's separate authorization policy.
Review your source and license, commit the intended release source, and push
it to that repository using your normal Git workflow. Record the exact commit:

```sh
git rev-parse HEAD
```

Use the real 40-character lowercase commit SHA below. The source archive must
match this commit, including its tests and license. Verify the chosen version
`0.1.0` has never been published before. Existing index records are permanent.
Test with the compiler pinned by the current official platform policy (currently
Mio 2.1.0) on every declared target. Compiler compatibility and platform support
are separate claims; every dependency must also support every selected target.

### 2. Create a source-only archive

`bunko.toml` and `bunko.package.toml` are **local development metadata**.
Do not put either file in a Bunko V1 release archive. Exclude `.git`, `.github`,
`.gitignore`, development caches, binaries, installation scripts, executable
scripts and all symbolic or hard links. Only regular non-executable UTF-8 text
files are allowed: `.mio`, `.md`, `.txt`, and `LICENSE`, `COPYING`, `NOTICE`,
`AUTHORS` or `README` (optionally with a `-suffix`). Use portable names containing
only letters, digits, `.`, `_` and `-`; no Windows reserved names, traversal,
absolute paths or case-insensitive collisions. Limits: 2000 files, 8 MiB per
file, 64 MiB total. Paths start at the package root, **without a wrapper directory**.

For the initial flat scaffold, run this Python 3 command from the package root.
It archives an explicit allowlist, validates text and file types, normalizes
modes to `0644`, and creates a real root-relative `.tar.gz` outside the project.
Extend the explicit list for additional source/tests/docs, applying the same
rules to every file and directory. Do not replace it with `tar .` or a GitHub
automatically generated source archive.

```sh
python3 - <<'PY'
import io
import os
import stat
import tarfile
from pathlib import Path

files = ['package.mio', 'package_test.mio', 'README.md', 'LICENSE']
output = Path('../exmikupro-input-0.1.0.tar.gz')
contents = []
for name in files:
    with open(name, 'rb') as stream:
        info = os.fstat(stream.fileno())
        assert not Path(name).is_symlink(), 'symlinks are prohibited'
        assert stat.S_ISREG(info.st_mode) and info.st_nlink == 1, 'regular files only; no hard links'
        assert not info.st_mode & 0o7111, 'executable/setuid/setgid/sticky modes are prohibited'
        data = stream.read(8 * 1024 * 1024 + 1)
    assert len(data) <= 8 * 1024 * 1024, 'file too large'
    data.decode('utf-8')
    assert b'\0' not in data, 'binary content is prohibited'
    contents.append((name, data))
assert len(contents) <= 2000 and sum(len(data) for _, data in contents) <= 64 * 1024 * 1024
with output.open('xb') as stream:
    with tarfile.open(fileobj=stream, mode='w:gz', format=tarfile.USTAR_FORMAT) as archive:
        for name, data in contents:
            entry = tarfile.TarInfo(name)
            entry.size = len(data)
            entry.mode = 0o644
            archive.addfile(entry, io.BytesIO(data))
print(output)
PY
```

Extract into a fresh directory and run `mio check .` and `mio test .` there too.
For dependencies, reproduce the official test layout from the testing guide.
Confirm all release files match the selected source commit before uploading.

### 3. Publish an Immutable GitHub Release

In the upstream repository's settings, enable immutable releases following
[GitHub's instructions](https://docs.github.com/en/code-security/concepts/supply-chain-security/immutable-releases).
Create a **draft** release, choose a tag that resolves to your exact source
commit, upload the `.tar.gz` as a release asset, then publish the draft.
Do not use GitHub's automatic source-code download as the Bunko asset.
After publication, verify GitHub reports `immutable: true`. If it does not,
resolve the repository settings and publish a new immutable release/version.
Do not claim publication or immutability based on the local scaffold.

### 4. Retrieve real IDs and verify the downloaded bytes

The following are read-only GitHub CLI examples. Replace every uppercase
placeholder with your actual repository/tag/asset; install and authenticate
`gh` yourself if needed. These commands do not create a release or log you in.

```sh
gh api repos/OWNER/REPOSITORY/releases/tags/TAG --jq '{release_id: .id, immutable: .immutable, draft: .draft, assets: [.assets[] | {name, id, url}]}'
gh api repos/OWNER/REPOSITORY/releases/RELEASE_ID
```

Select the ID of the uploaded `.tar.gz` from that release's `assets` list.
Confirm it belongs to the real published immutable release, and resolve the
tag to the full commit SHA (peel annotated tags to their commit).
Download the exact asset bytes and compute their SHA-256:

```sh
gh api -H 'Accept: application/octet-stream' repos/OWNER/REPOSITORY/releases/assets/ASSET_ID > ../downloaded-release.tar.gz
python3 - <<'PY'
import hashlib
from pathlib import Path
print(hashlib.sha256(Path('../downloaded-release.tar.gz').read_bytes()).hexdigest())
PY
```

Compare the downloaded archive with the one you uploaded. Record the real
positive decimal release and asset IDs and 64-character lowercase SHA-256.
Never copy fixture IDs, invented hashes or the placeholders below into a submission.

### 5. Add a version record to the index

Fork [mio-kobo/bunko-index](https://github.com/mio-kobo/bunko-index), clone your
fork, and add the official repository as `upstream`. Fetch its latest `main`
and create a submission branch from `upstream/main`. Add this new file:

`packages/exmikupro/input/0.1.0.toml`

The following is a **template, not a valid submission until all placeholders
are replaced**. Set the compiler range only after testing it; it must include
the compiler currently pinned by index policy. Copy the exact dependency table
from your local `bunko.toml`, ensuring each dependency version is in the index.

```toml
name = "exmikupro/input"
version = "0.1.0"
publisher = "github:exmikupro"
description = "Mio CIL INPUT Bunko"
repository = "REPLACE_WITH_CANONICAL_PUBLIC_GITHUB_REPOSITORY_URL"
source_commit = "REPLACE_WITH_40_CHARACTER_LOWERCASE_COMMIT_SHA"
release_id = "REPLACE_WITH_REAL_POSITIVE_DECIMAL_RELEASE_ID"
archive = "https://api.github.com/repos/OWNER/REPOSITORY/releases/assets/ASSET_ID"
sha256 = "REPLACE_WITH_64_CHARACTER_LOWERCASE_SHA256"
license = "MIT"

[mio]
min = "REPLACE_WITH_TESTED_INCLUSIVE_MINIMUM"
before = "REPLACE_WITH_TESTED_EXCLUSIVE_UPPER_BOUND"

[platforms]
macos = ["arm64"]

[dependencies]

```

The selected platforms above are explicit; omitted targets are unsupported.
Check the latest [platform policy](https://github.com/mio-kobo/bunko-index/blob/main/policies/platform-targets.toml)
before submitting. Do not add targets merely because a compiler supports them.

### 6. Validate and submit a Pull Request

From your index checkout, use Python 3.11 or newer and Git:

```sh
git fetch upstream main
python3 tools/validate.py --base upstream/main
python3 -m unittest discover -s tests -v
```

The official guide uses `--base origin/main` when `origin` points to the official
repository. In a fork checkout, `upstream/main` must refer to the official base.
Commit the new TOML file, push your submission branch to your fork, then open a
PR against `mio-kobo/bunko-index:main`. Link the immutable release and source
commit. Include publisher-controlled delegation evidence if submitting from a
different account. Submission and passing tests do not grant automatic approval.

### 7. Follow CI and review

Open the PR's **Checks** tab. Wait for **Validate Bunko index** and
**Package tests**, inspect each declared platform job and its step summary,
and respond to maintainer feedback. CI runs source/tests with a pinned compiler
on GitHub-hosted runners. It never approves or merges a PR automatically.
Published records cannot be edited or deleted. Correct release problems by
publishing a new release/version and adding its new record; coordinate with
reviewers about replacing an unmerged proposed record.

## Troubleshooting

- No tests: include a discoverable `*_test.mio` with at least one valid `test_` function.
- Archive rejected: remove forbidden files/links, check names, text encoding and modes.
- Source verification failed: verify immutability, tag commit, IDs and the downloaded asset digest.
- Platform dependency error: every dependency must support all declared targets.
- Compiler error: reproduce using the pinned compiler and the official dependency layout.
- Validator error: replace every placeholder, use exact versions, and preserve accepted records.

## Official documentation

- [Bunko Index](https://github.com/mio-kobo/bunko-index)
- [Contribution guide](https://github.com/mio-kobo/bunko-index/blob/main/CONTRIBUTING.md)
- [Package format and archive rules](https://github.com/mio-kobo/bunko-index/blob/main/docs/package-format.md)
- [Testing and troubleshooting](https://github.com/mio-kobo/bunko-index/blob/main/docs/testing.md)
- [Mio test syntax](https://github.com/mio-kobo/mio/blob/v2.1.0/docs/testing.md)
