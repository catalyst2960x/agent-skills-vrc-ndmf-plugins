# VPM Packaging Reference

What a plugin package must satisfy to be installed and managed as a
VRChat Package Manager (VPM) package through the VRChat Creator
Companion (VCC). Facts are those of the VCC documentation and the
Unity Manual; URLs per section. VPM behaviour changes with VCC
releases, so re-verify against the same pages before relying on a
field's exact semantics.

Contents:

- Package format and placement
- Manifest: required and recommended fields
- Project templates are not packages
- Dependencies and version ranges
- Migrating from an Assets-based distribution
- Testing before publishing: user packages

## Package format and placement

- The VPM format is the Unity Package Manager format with additions.
  A package is a folder containing a `package.json` manifest, and
  the folder carries the package's technical name: the `name` in
  the manifest, in reverse domain notation.
- The VCC installs packages into the project's `Packages` folder and
  manages them from the Manage Project screen: select a version in
  the dropdown, or use the + icon to install the newest version.
- An installed package sits under the package manager's control: a
  package installed from a registry is immutable, and the VCC
  replaces a package's contents when its version changes. A plugin
  never writes generated files into its own package folder. Files
  a plugin generates into a project belong to the project: write
  them under a single root folder in `Assets` that the plugin
  owns, and confine the plugin's writes and cleanup to that root.
- A project must be in VPM format before its packages can be managed;
  legacy SDK3 projects are migrated by the VCC first.
- Package sources: official packages, curated packages, community
  repositories added by the user, and user packages (below).

Source: https://vcc.docs.vrchat.com/vpm/
Source: https://vcc.docs.vrchat.com/vpm/packages
Source: https://docs.unity3d.com/Components/cus-naming.html
Source: https://docs.unity3d.com/2021.3/Documentation/Manual/upm-embed.html

## Manifest: required and recommended fields

Required fields in `package.json`:

| Field | Meaning |
| --- | --- |
| `name` | The package ID |
| `displayName` | Human-readable name shown in the VCC |
| `version` | Semantic version (below) |
| `url` | Direct-download link to a zip file of the package |
| `author` | Object with `name` and `email` |

- The remaining fields are optional, but the documentation recommends
  filling them out as much as possible.
- `license`: highly recommended in the manifest, in SPDX format,
  even when a LICENSE file is present in the package.
- `changelogUrl`: optional; points to a changelog file. Not currently
  used by the VCC.
- `zipSHA256` does NOT belong in the package's own `package.json`; it
  goes in the repository listing's manifest, where it is currently
  used for cache invalidation.

Source: https://vcc.docs.vrchat.com/vpm/packages

## Project templates are not packages

- VCC templates create new projects; their manifest is a different
  format whose only required fields are `name` and `displayName`.
  Do not model a package's `package.json` on the template example.

Source: https://vcc.docs.vrchat.com/vpm/templates

## Dependencies and version ranges

- `vpmDependencies` lists the VPM packages this package requires.
  The VCC resolves dependencies of dependencies, and treats every
  dependency version as a range (SemanticVersioning library;
  Semantic Versioning 2.0.0; prerelease versions supported).
- A package's own versioning follows semantic versioning, starting
  at 1.0.0.
- For a dependency on the VRChat SDK, the documentation recommends
  declaring against the SDK's current Breaking version with `x` for
  the Bump version, e.g. `"com.vrchat.avatars": "3.5.x"` means
  >=3.5.0 <3.6.0. The package must then be retested and re-released
  when the SDK moves to a new Breaking version.
- Set every framework dependency's lower bound from the APIs the
  package actually calls: for NDMF, the version floors in Framework;
  for Modular Avatar, the release that introduced each feature used.
  A bound below the floor of a used API admits versions in which the
  package does not compile.

Source: https://vcc.docs.vrchat.com/vpm/packages

## Migrating from an Assets-based distribution

- `legacyFolders`: an object mapping an Assets-relative path to a
  folder GUID, or to an empty string. When the package is installed,
  a folder found at a matching path is removed. With a GUID, the
  folder is found even if it was moved from its original location;
  with an empty string, only the path matches. Paths use backslashes,
  e.g. `"Assets\\FolderName": ""`.
- `legacyFiles`: the same mechanism for individual files.
- `legacyPackages`: a list of package IDs to remove from the project
  and its manifest when this package is added, for a package that has
  been integrated into another package.
- A plugin that was previously distributed (or verified) as an
  Assets folder declares that folder in `legacyFolders`, so
  installing the VPM package replaces the Assets copy instead of
  leaving both in the project.

Source: https://vcc.docs.vrchat.com/vpm/packages

## Testing before publishing: user packages

- A package under development can be registered as a user package:
  in the VCC, open the Settings screen, press Add under User
  Packages, and choose a folder that contains a package (or several
  packages).
- Valid user packages appear in the listing and on the project's
  Manage Project screen, and can be installed into projects from
  there. The documentation presents this as the route to test a
  package's structure and manifest before publishing.
- Distribution beyond the local machine is through a repository
  listing (community repository) or the curated listing, not through
  the user-package route.

Source: https://vcc.docs.vrchat.com/vpm/packages

## See Also

- `references/framework.md` — the package's internal layout and
  assembly shape.
- `references/vrchat-avatar-platform.md` — the VRChat-side facts the
  generated avatar output must satisfy.
- `references/procedure-index.md` — P-25 runs this reference as a
  procedure.
