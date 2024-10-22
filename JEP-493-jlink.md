Summary
-------

Reduce the size of the JDK by approximately 25% by enabling the `jlink` tool to
create custom run-time images without using the JDK's JMOD files. This feature
must be enabled when the JDK is built; it will not be enabled by default, and
some JDK vendors may choose not to enable it.


Motivation
----------

The installed size of the JDK on the filesystem is important in cloud
environments, where container images that include an installed JDK are
automatically and frequently copied over the network from container registries.
Reducing the size of the JDK would improve the efficiency of these operations.

A complete, installed JDK has two main components: A _run-time image_, which is
the executable Java run-time system, and a set of _packaged modules_, in the
[JMOD format](https://openjdk.org/jeps/261#Packaging:-JMOD-files), for each
module in the run-time image.

The JMOD files are used by the `jlink` tool when creating [custom run-time images](https://dev.java/learn/jlink/).
The run-time image in a complete JDK is
itself such an image, created from these JMOD files via `jlink`. Thus every
class file, native library, configuration file, and other resource in the
run-time image is also present in one of these JMOD files — arguably a massive
waste of space.

In fact, the JMOD files in a complete JDK account for about 25% of the JDK's
total size. If we could enhance the `jlink` tool to extract class files, native
libraries, configuration files, and other resources from the run-time image
itself then we could dramatically reduce the size of the installed JDK by
omitting the JMOD files.


Description
----------

The new JDK [build-time](https://openjdk.org/groups/build/doc/building.html)
configuration option `--enable-linkable-runtime` builds a JDK whose `jlink` tool
can create run-time images without using the JDK's JMOD files, and which does
not include those files, i.e., there is no `jmods` directory.

    $ configure [ ... other options ... ] --enable-linkable-runtime
    $ make images

The resulting JDK is approximately 25% smaller than a JDK built with the default
configuration, and it contains exactly the same modules.

The `jlink` tool in the resulting JDK works exactly the same way as the `jlink`
tool in a JDK built with the default configuration. For example, to create a
run-time image containing only the `java.xml` and `java.base` modules, the
`jlink` invocation is the same:

    $ jlink --add-modules java.xml --output image
    $ image/bin/java --list-modules
    java.base@24
    java.xml@24
    $ 

The invocations for more complex cases are also the same. For example, suppose
we want to create a run-time image containing an application module `app` which
requires a library `lib`. These modules are packaged as [modular JAR files](https://docs.oracle.com/en/java/javase/23/docs/specs/jar/jar.html#modular-jar-files) 
in an `mlib` directory. We specify them to `jlink` via the `--module-path`
option, as usual:

    $ ls mlib
    app.jar	lib.jar
    $ jlink --module-path mlib --add-modules app --output app
    $ app/bin/java --list-modules
    app
    lib
    java.base@24
    $ 

The `jlink` tool copies the class files and resources for the `app` and `lib`
modules from the modular JAR files `app.jar` and `lib.jar`. It extracts the
class files, native libraries, configuration files, and other resources for the
JDK's modules from the JDK's run-time image.


### Not enabled by default

The default build configuration will remain as it is today: The resulting JDK
will contain JMOD files and its `jlink` tool will not be able to operate without
them. Whether the JDK build that you get from your preferred vendor contains
this feature is up to that vendor.

We may propose to enable this feature by default in a future release.


### Restrictions

The `jlink` tool in a JDK built with `--enable-linkable-runtime` has a few
limitations compared to that in a JDK built with the default configuration:

- `jlink` cannot be used to create a run-time image that itself contains the
  `jlink` tool. The `jlink` tool is in the `jdk.jlink` module, so this fails:

      $ jlink --add-modules jdk.jlink --output image
      Error: This JDK does not contain packaged modules and cannot be used \
      to create another run-time image that includes the jdk.jlink module.

  We may revisit this restriction in the future if it proves problematic.

- `jlink` fails if any user-editable configuration files are modified.

  The JDK's `conf` directory contains various files that developers may edit to
  configure the JDK. The `conf/security/java.security` file, in particular,
  configures security providers, cryptographic algorithms, and so forth. In a
  default build, `jlink` copies the user-editable configuration files for the
  JDK's modules from the JDK's JMOD files. Without JMOD files, `jlink` copies
  the configuration files from the run-time image, and fails if any of the files
  differ from the original files:

      $ jlink --add-modules java.xml --output image
      Error: [...]/bin/conf/security/java.security has been modified.

  This restriction prevents `jlink` from creating a run-time image with an
  ad-hoc or insecure configuration. If the security configuration was changed
  to, e.g., enable an obsolete message-digest algorithm that is disabled by
  default, then it would be inappropriate to copy that configuration to the new
  run-time image.

- Cross-linking, e.g., running `jlink` on Linux/x64 to create a run-time image
  for Windows/x64, is not possible.


Alternatives
----------

A JDK vendor could provide a JDK's JMOD files as a separate download. Some Linux
distributions already do essentially this, by providing one installation package
for the JDK run-time image and another for the corresponding JMOD files.

This approach is brittle, since if the second package is not installed then the
`jlink` tool will not work. This approach is, furthermore, not well-suited to
cloud environments in which a JDK run-time image and its JMOD files might wind
up in different and conflicting container-image layers.

