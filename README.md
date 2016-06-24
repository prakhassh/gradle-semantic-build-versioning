# Introduction

This is a gradle plugin that provides support for [semantic versioning](http://semver.org) of builds. It is quite easy to use and extremely configurable. The plugin allows you to bump the major, minor, and patch version based on the latest version, which is identified from a git tag. It also allows you to bump pre-release versions based on a scheme that you define. The version can be bumped by using version-component-specific tasks or can be bumped automatically based on the contents of a commit message. If no tasks from the plugin are specifically invoked, the plugin will increment the version-component with the lowest precedence; this is usually the patch version, but can be the pre-release version if the latest version is a pre-release one.

# Usage

Using the plugin is quite simple:

**Gradle version <= 2.1**
```gradle
buildscript {
    repositories {
        maven {
            url "https://plugins.gradle.org/m2/"
        }
    }
    dependencies {
        classpath "gradle.plugin.net.vivin:gradle-semantic-build-versioning:1.0.0"
    }
}

apply plugin: 'net.vivin.gradle-semantic-build-versioning'
```

**Gradle version >= 2.1**
```gradle
plugins {
    id "net.vivin.gradle-semantic-build-versioning' version: "1.0.0"
}
```

This is usually enough to start using the plugin. Assuming that you already have tags that are (or contain) semantic versions, the plugin will find the latest<sup>1</sup> tag and increment the component with the least precedence. This is the default behavior of the plugin.

Additional configuration-options can be specified like so:

```gradle
project.version.with {
    // Options here
}
```

<sup>1</sup>Latest based on ordering-rules defined in the semantic-version specification; **not latest by date**.

# Tasks

The plugin provides tasks that can be used to bump components of the version. Only **one** of the bump tasks can be used at a time.

## `bumpMajor`

This task bumps the major version. Assuming that the current version is `x.y.z`, the new version will be `(x + 1).0.0`. If the current version is a pre-release version, the pre-release version-component is discarded and the new version will still be `(x + 1).0.0`.

## `bumpMinor`

This task bumps the minor version. Assuming that the current version is `x.y.z`, the new version will be `x.(y + 1).0`. If the current version is a pre-release version, the pre-release version-component is discarded and the new version will still be `x.(y + 1).0`.

## `bumpPatch`

This task bumps the patch version. Assuming that the current version is `x.y.z`, the new version will be `x.y.(z + 1)`. If the current version is a pre-release version, the pre-release version-component is discarded and the new version will still be `x.y.(z + 1)`.

## `bumpPreRelease`

This task bumps the pre-release version. Pre-release versions are denoted by appending a hyphen, and a series of dot-separated identifiers that can only consist of alphanumeric characters and hyphens; numeric identifiers cannot contain leading-zeroes. Since pre-release versions are arbitrary, using this task requires some additional configuration (see `preRelease` under **Options**). Bumping the identifier has differing behavior based on whether the latest version is already a pre-release version:

 - **If the current version is not a pre-release version:** Assuming that the latest version is `x.y.z`, the bumped version will be `x.y.(z + 1)-<startingVersion>` (see `startingVersion` in `preRelease` under **Options**).
 - **If the current version is a pre-release version:** Assuming that the latest version is `x.y.z-<identifier>`, the bumped version will be `x.y.z-<identifier++>` where the value of `<identifier++>` is determined based on a scheme defined by you (see `bump` in `preRelease` under **Options**).

## `promoteToRelease`

This task promotes a pre-release version to a release version. This is done by discarding the pre-release version-component. For example, assuming that the current version is `x.y.z-some.identifiers.here`, the new version will be `x.y.z`. **This task can only be used if the latest version is a pre-release version**.

## `autobump`

This task will bump a version-component or promote a pre-release version to a release version based on the latest tag and the contents of the latest commit-message. This task supports additional-configuration (see `autobump` under **Options**).

## `release`

This task specifies that the build is a release build, which means that a snapshot suffix is not attached to the version (see `snapshotSuffix` under **Options**). Some gradle plugins provide a `release` task of their own; in this situation, the build-version plugin will not add its own release task, but the overall impact on the version is the same: the snapshot suffix will not be attached. **You cannot release a build if there are uncommitted changes**.

## `tag`

This task with create a tag corresponding to the current version (with an optional prefix; see `tagPrefix` under **Options**) *and* push all tags. It is recommended to use this tag along with the `release` task when creating a release. **You cannot tag a snapshot release; use pre-release identifiers instead**.

## `printVersion`

Prints out the new version.

# Options

The plugin has a few configuration options that you can use to fine-tune its behavior, or to provide additional options for certain tasks.

## General options

These options control what your versions and tags look like. Using these options, you can set an optional starting-version (used when no tags are found), an optional tag-prefix, and the snapshot-suffix to use for snapshot versions.

### `startingVersion`

This sets the version to use when no previous tags could be found. By default it is set to `0.0.1`. This must be a valid semantic-version string.

### `tagPrefix`

This option defines an optional prefix to use when tagging a release. By default it is blank, which means that the tag corresponds to the version number.

### `snapshotSuffix`

This is the suffix to use for snapshot versions. By default it is `SNAPSHOT`. This suffix is always attached to the version unless the `release` task has been invoked.

## Filtering tags

These options let you restrict the set of tags considered when determining the latest version.

### `tagPattern`

This pattern tells the plugin to only consider those tags matching `tagPattern` when trying to determine the latest version from the tags in your repository. The value for this option is expected to be a regular expression. Its default value is `/.*/`.

**Example:** Only consider tags that start with `foo`
```gradle
project.version.with {
    tagPattern = ~/^foo/
}
```

### `versionsMatching`

This option is similar in function to `tagPattern`, except that it allows you to restrict the set of tags considered, based on the explicitly-specified major, minor, or patch versions. When specifying a version component to match, preceding components (if any) must also be specified. While what `versionsMatching` does can also be accomplished by `tagPattern`, `versionsMatching` provides a friendlier way to restrict the set of considered tags based on versions alone.

**Example:** Only consider tags with major-version `2`:
```gradle
project.version.with {
    versionsMatching {
        major = 2
    }
}
```

**Example:** Only consider tags with major and minor-version `1.2`:
```gradle
project.version.with {
    versionsMatching {
        major = 1
        minor = 2
    }
}
```

**Example:** Only consider tags with major, minor, and patch-version `1.2.0`:
```gradle
project.version.with {
    versionsMatching {
        major = 1
        minor = 2
        patch = 0
    }
}
```

**Note:** Filtering based on `versionsMatching` is performed **after** tags have been filtered based on `tagPattern`.

## Pre-releases

This is how you can define your pre-release versioning strategy. This is a special case because other than defining a basic syntax and ordering rules, the semantic-versioning specification has no other rules about pre-release identifiers. This means that some extra configuration is required if you want to generate pre-release versions.

### `preRelease`

This option allows you to specify how pre-release versions should be generated and bumped. It has the following properties:

 - **`startingVersion`**: This is a required property that describes the starting pre-release version of a new pre-release. This value will be used if `bumpPreRelease` is invoked (either explicitly or via `autobump`) and the latest version is not a pre-release version.

   **Example:** Using a starting version for a pre-release version
   ```gradle
   project.version.with {
       preRelease {
           startingVersion = "alpha.0"
           // ...
       }
   }
   ```
 - **`pattern`**: This property is similar in function to `tagPattern`, except that it allows you to restrict the set of tags considered to those tags with pre-release versions matching `pattern`. The value for this property is expected to be a regular expression. Its default value is `/.*/`. One thing to remember is that starting anchors (`^`) cannot be used, because the actual regular-expression that is used is `-<pattern>`. Hence, if you are trying to filter based on pre-release versions starting with some string, it is simply enough to provide that string in the regular expression without prefixing it with `^`.

   **Example:** Only consider tags whose pre-release version starts with `alpha`:
   ```gradle
   project.version.with {
       preRelease {
           startingVersion = "alpha.0"
           pattern = ~/alpha/
           // ...
       }
   }
   ```

   **Note:** Filtering based on `pattern` is performed **after** tags have been filtered based on `tagPattern` and `versionsMatching`.

 - **`bump`**: This property allows you to specify how pre-release versions should be incremented or bumped. It is a closure that accepts a single string-argument representing the latest version, and it expected to return a string that represents the incremented version.

   **Example:** Defining how the pre-release version should be bumped
   ```gradle
   project.version.with {
        preRelease {
            startingVersion = "alpha.0"

            // The bumping scheme is alpha.0 -> alpha.1 -> ... -> alpha.n
            bump { String version ->
                int num = Integer.parseInt(version.split*/\./[1]) + 1
                return "alpha.${num}"
            }
        }
   }
   ```

## Automatic bumping based on commit messages

Sometimes you might want to automatically bump your version as part of your continuous-integration process. Without this option, you would have to explicitly configure your CI process to use the corresponding bump tasks, if you want to bump the major or minor versions. This is because the default behavior is to bump the component with least precedence. Instead, you can configure the plugin to automatically bump the desired version based on the contents of your commit message.

### `autobump`

This option allows you to specify how the build version should be automatically bumped based on the commit message. Each line of the commit message is checked to see if it matches a specified pattern. If so, the corresponding version is bumped. The option has the following properties:

 - **`majorPattern`**: If any line in the commit message matches `majorPattern`, the major version will be bumped. The value for this property is expected to be a regular expression, and its default value is `/^\[major\]$/`.
 - **`minorPattern`**: If any line in the commit message matches `minorPattern`, the minor version will be bumped. The value for this property is expected to be a regular expression, and its default value is `/^\[minor\]$/`.
 - **`patchPattern`**: If any line in the commit message matches `patchPattern`, the patch version will be bumped. The value for this property is expected to be a regular expression, and its default value is `/^\[patch\]$/`.
 - **`preReleasePattern`**: If any line in the commit message matches `preReleasePattern`, the pre-release version will be bumped (i.e., either a new pre-release version will be created if the latest version is already not one, or the latest pre-release version will be bumped based on `bump` in `preRelease`). The value for this property is expected to be a regular expression, and its default value is `/^\[pre-release\]$/`.
 - **`promoteToReleasePattern`**: If any line in the commit message matches `promoteToReleasePattern`, the version will be promoted to a release version. The same rules that apply to the `promoteToRelease` task, apply here as well. The value for this property is expected to be a regular expression, and its default value is `/^\[promote\]$/`.

**Example:** Defining custom patterns to be used by `autobump`
```gradle
project.version.with {
    autobump {
        majorPattern =~ /^\[bump-major\]$/
        minorPattern =~ /^\[bump-minor\]$/
        patchPattern =~ /^\[bump-patch\]$/
        preReleasePattern =~ /^\[bump-pre-release\]$/
        promoteToReleasePattern =~ /^\[promote-to-release\]$/
    }
}
```