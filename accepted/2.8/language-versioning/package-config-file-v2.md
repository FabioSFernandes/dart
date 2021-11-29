# Dart Package Configuration File v2.0

Author: lrn@google.com
Version: 1.1

This document specifies the location, format and semantics of a new package resolution configuration file. A new file format is needed because the Dart "language versioning" feature requires adding data that are not supported by the existing file format. For backwards compatibility with third party tools, we retain the existing file for a while. Tools currently using `.packages` should migrate to using the new file when it becomes available, and tools creating `.package` should start doing so as soon as possible.

## Background

Dart currently uses a `.packages` file to configure *package URI resolution* (converting a package URI to a file name). The file uses a `.ini` file-like format with package names as keys and URI references as values. The URI references generated by Pub are always absolute or relative paths, so third-party users of the file tend to see them as paths, not URI.

For the [*language versioning*](feature-specification.md) feature, we have proposed extending the format by adding fragments to the URI references and an entry with the empty string as key. This is a breaking change (the empty key was not previously accepted) and even more breaking for tools which assume the values are plain paths.

Instead of breaking the existing file users, we will instead introduce a *new* file containing the information that we intended to store in the `.packages` file. This file will use a structured format which allows extending it with further information in the future. We will still generate the `.packages` file for now, allowing users of the file to continue working until they can migrate to using the new file.

## New File

### Syntax

The new file will be structured as JSON text.

The JSON format is extensible and efficiently parsable in languages other than Dart. It should be possible to check the file into a source versioning system, so a text based format is preferred. We want something structured, text-based and widely supported, and JSON satisfies those constraints.

JSON is easy to read, but not easy to *write* by hand (although doable), and it does not allow comments. This is a machine generated file that is not intended to be edited by hand, and comments can be added as object properties with names like `"toolComment"`.

The primary way for users to create the file is by running the Pub tool. That tool is free to overwrite an existing file, and Pub may be invoked automatically by editors, so any manually added changes are likely lost. There is no support for manual edits surviving across generation of a new file.

Some users may use other tools than Pub to control their package resolution. Those tools need to also support the new format.

### Location

The new file will be stored by Pub in the `.dart_tool` sub-directory of the current pub package's root directory. It is a file used and shared by various Dart tools, so this location is sensible and consistent. It avoids clutter at the top-level of a package, and the `.dart_tool` directory is already in `.gitignore` files.

Tools which needs the file will automatically look for it in `.dart_tool` subdirectories of the directories where they would currently look for `.package` files.

### Name

The new file is named `package_config.json`.

The file's name should signal three things:

- It is used by the Dart language tools to understand Dart and Pub packages source correctly, hence the "package".
- It is a generated file, not intended for manual editing. The "config" is intended to give this association. It is a "scare word" by being abbreviated and vague, while still correctly describing the content.
- It's JSON, so in case you do want to view or edit the file, the editor can treat it properly.

### Contents

The file will contain information specifying:

- A number of *packages*, each with:
  - A *package name*.
  - A *location URI* specifying where to find the files of the Pub package.
  - A sub-directory (relative URI reference) specifying the files accessible through `package:` URIs (the Dart package).
  - A *language version* (major/minor version number).

On top of that, we also add extra metadata which allows us to version the format and attach information about when and how the file was created.

The file JSON text defines a JSON object (the "top level object").

That object has, at least, the following properties:

- `configVersion`: A single integer marking the version of this format, currently `2` (counting `.packages` as the first). Tools should refuse to work on files with a larger version number than the ones they are aware of. Incremental changes will not require increasing the version, so new versions are not expected any time soon, but code should still prepare for it.
- `packages`: An array of packages, each with the following properties:
  - `name`: A string containing a package name.
  - `rootUri`: A URI reference with no query or fragment parts. It can be, for example, a full URI, an absolute path or a relative path. The URI reference is resolved relative to the (likely `file:`) URI identifying the `package_config.json` file. If the resolved URI does not end with a `/` (signaling that it is a directory), a `/` is appended. The resulting URI specifies the *root directory* of the package. All files inside this directory, including any subdirectories, are considered to belong to the package, execpt if they belong to a nested package. The value is a URI reference, and tools must support URI references with schemes and/or `%`-escapes properly.
  - `packageUri`: Optional. A URI reference consisting of just a relative URI path. It should be resolved against the root directory to specify the *package URI directory*, which must be a sub-directory of the root directory. If the result of this resolution does not end with a `/`, a `/` is appended. This is the directory containing files available to Dart programs using `package:packageName/...` URIs. If omitted, the root directory is also the package URI directory. The value is a URI path, and tools must support URI references with `%`-escapes and `..` segments properly, and validate that the result of resolving it against the root directory stays inside the root directory (which a leading `..` can violate).
  - `languageVersion`: Optional. A string containing a Dart language version consisting of a decimal numeral, a `.` character and another decimal numeral, where a decimal numeral must not have a leading `0` unless the numeral is exactly `0`. That is, a string matched in its entirety by the regular expression `(0|[1-9]\d*)\.(0|[1-9]\d*)`. *This is the same format accepted by `// @dart = x.y` comments in source files.*

It is an error if the package URI of a package is not inside the same package's root URI.
It is an error if the package URI of a package is inside a nested package's root URI.
It is an error if a nested package's root URI is insde the package URI of a package .
It is an error if two different packages have the same directory as root URI.
If any of these happen, the configuration is invalid and must be rejected.

If one package's root URI is inside another package's root URI, any file which is inside both is considered to belong to the inner root's package. *That is, package roots can be nested inside each other, and a file belongs only to the package with the nearest package root. This ensures that all files belong to at most one package.*

All files inside a package's package URI must belong to that package, so it must not overlap with any other nested package's root.

The following optional properties of the top-level object allows the generator to specify metadata about the file. They can all be omitted, but if they are present, they should have the expected format.

- `generated`: A string containing an ISO-8601 UTC timestamp for when the file was generated, using the format: `YYYY-MM-DDTHH:mm:ss.sssZ`, for example `"generated": "2019-09-24T12:34:56.000Z"`.
- `generator`:  A string with a name/identifier of the generator which created the file, typically `"pub"`.
- `generatorVersion`: A string with the version of the generator, if the generator wants to remember that information. The version must be a [Semantic Version](https://semver.org/spec/v2.0.0.html). Pub can use the SDK version. Example: `"generatorVersion": "2.5.0"`.

The JSON properties may occur in any order, but where convenient, the `configVersion` *should* be the first entry in the file. The Dart JSON serializer uses iteration order of the incoming map, so controlling the order is possible. The file *should* be "pretty printed" with newlines and indentation, to make human inspection easier, but tools consuming the file must accept JSON with or without any optional whitespace.

#### Stability and Versioning

The `"configVersion"` property allows versioning of the format.

There is no "minor version". Either the format is backwards compatible, or it is not. If we add more properties in a future release, in a backwards compatible way, tools should look for the presence those properties, not predict them based on the API version.

Tools should ignore properties that they don't recognize. This will allow us to add more properties in the future without increasing the `configVersion` number, but not to remove properties or change the meaning of existing properties. Any change which allows existing tools to continue working without doing something decidedly *wrong*, is considered no-breaking.

If we remove a property that tools rely on, change the meaning of an existing property, or add more information that is *required* for resolving and understanding Dart source files, it is probably a breaking change&mdash;tools expecting the previously available data will no longer understand the whole picture. Such changes will require increasing the `configVersion` number.

#### Extra Information in the File

If a tool wants to store extra information in the `package_config.json` file, they can do so under a key which corresponds to or contains the name or identifier of the tool. Authors should ensure that the name is available in the ecosystem before claiming it for their tool. If Pub wanted to store more information on a package, say the package version, it could do so as `"pubPkgVersion": "1.16.0"` in the package entry.

This feature can only really be used by the tool generating the file (as a generated file, it may be overwritten at any time), and it should be used sparingly since unknown properties are just overhead for all other users of the file. Still, if a tool can cooperate with Pub and provide the information in `pubspec.yaml`, Pub might be able to put it into the configuration file as well.

Any tool which modifies the file piecemeal by changing or adding information, rather than writing it from scratch like Pub, should retain all properties that it does not recognize. This allows multiple tools to all cooperate about adding information to the file.

Again, Pub is not required to retain any information, or read the file at all, it can overwrite the file when invoked.

#### Package Names

Some properties are specified as being "package names".

A package name is used as both a URI path segment and a directory name. To ensure this is possible, we define a package name as a string containing:

- Only [URI path characters](https://www.ietf.org/rfc/rfc3986.txt) (RFC 3986 "pchar"),
- but no `:` or `%`,
- and containing at least one non-`.` character.

This allows the characters: `a`-`z`, `A`-`Z`, `0`-`9`, `-`, `.`, `_`, `~`, `!`, `$`, `&`, `'`, `(`, `)`, `*`, `+`, `,`, `;`, `=`, `@`.
It avoids the characters `/`, `\` and `:` which are meaningful in paths on various operating systems, as well as the strings `.` and `..`, and it ensures that the package name is a valid non-empty URI path segment.

Pub has stronger restrictions on its accepted package names, but this file will also be used in settings where packages are not supplied by Pub.

### Finding the File

The *default transitional layout*, generated by the Pub tool, is to generate both a `.packages` file in the Pub package root directory, and a `package_config.json` file in the `.dart_tool` directory which is itself in the Pub package root directory. Tools should prefer the `package_config.json` file because it contains information that is essential to parsing and understanding Dart programs, but old tools which do not yet use that information, can still use the information of the `.packages` file to resolve `package:` URIs. 
Eventually we will stop generating the `.packages` file.

When compiling a Dart script file, the Dart tool will look for the necessary information starting in the directory of the script file. It will first check for a `.dart_tool/package_config.json` file, then for a `.packages` file, then it will repeat this through the parent directories until it finds one such file or it reaches the file system root.

The analyzer needs to cleverly determine which configuration file applies to which file, based on the location of scripts, which scripts may transitively import a file, and understanding of the structure of Pub packages. It should be able to use the same algorithm as now, just that where it currently looks for a `.packages` in a directory, it will have to for `.dart_tool/package_config.json` first.

### Specifying the File

Tools currently allow you to specify the `.packages` file with a command line argument:

```
--packages=someFile
-p someFile
```

This approach will work with the JSON configuration file as well. The tool must scan the file to detect whether it is a JSON file or an original `.packages` file. It is considered a JSON file if the first non-whitespace (space, tab, carriage return, linefeed) character is a `{`. The JSON file must start this way, and the existing format cannot do so; it must start with either a comment, `#`, or a non-empty package name, and package names cannot contain a `{` character.

In order to allow other tools which currently specify a `.packages` file using a `--packages` or `-p` parameter to a Dart tool, to continue working, even after those Dart tools starts to expect a `package_config.json` file, we will automatically redirect to the `package_config.json` file if we expect the file to be part of the default transitional layout.

To that end, if a parameter like the above is provided, the *file name* of the provided parameter is `.packages`, *and* a `.dart_tool/package_config.json` file exists next to that `.packages` file, then the `.dart_tool/package_config.json` file is read instead.
The `package_config.json` file must be a valid JSON configuration file. Tools should emit a warning if that `package_config.json` file turns out to not be a JSON file.

## Possible Extensions

We can consider already adding more features to the format. These features should be ones that are of relevance to compilers reading source files, since that is the goal of the file.

Any such extensions would need to be added by the generator, so in practice they must come from `pubspec.yaml` to begin with. If the Pub tool wants to support these, it is possible.

### Experiments

Maybe you can enable experiments for a specific package by adding, for example:

```json
"experiments": ["non-nullable"]
```

to a single package entry, or add it in the outer object to enable it for all compatible packages.

### Environment Variables

Maybe you can hard-code some environment variables by adding, for example:

```json
"environment": {
  "myFlag": "true",
  "logLevel": "verbose"
}
```

in the outer object. These would be available for `String.fromEnvironment("logLevel", defaultValue: "default")` or `bool.fromEnvironment("myFlag")`

## Summary

A file named `package_config.json`, stored in `.dart_tool`, will coexist with `.packages` until `.packages` can be phased out.

The format will look like:

```json
{
  "configVersion": 2,
  "packages": [
    {
      "name": "myPackage",
      "rootUri": "../",
      "packageUri": "lib/",
      "languageVersion": "2.6"
    },
    {
      "name": "myHelperPackage",
      "rootUri": "../../myHelperPackage/",
      "packageUri": "lib/",
      "languageVersion": "2.5"
    },
    {
      "name": "test",
      "rootUri": "/users/myself/.pubcache/test-1.16.0/lib/",
      "languageVersion": "2.5"
    },
    ...
  ],
  "generated": "2019-09-12T12:13:14Z",
  "generator": "pub",
  "generatorVersion": "2.6.0-dev.0.2"
}
```

## Revisions

1.0: Initial version.

1.1: Allow overlapping package roots, but not package URI directories.