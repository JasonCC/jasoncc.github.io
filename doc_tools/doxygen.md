---
layout: default
---

# How to Generate Documentation from Source Code in Linux

To install doxygen on Ubuntu, Mint or Debian:

```
$ sudo apt-get install doxygen
$ sudo apt-get install graphviz
```
To install doxygen on CentOS, Fedora or RHEL:

```
$ sudo yum/dnf install doxygen
$ sudo yum/dnf install graphviz
```

# Generate Documentation from Source Code with Doxygen

To generate documentation of source code, proceed as follows.

First, generate a project specific doxygen configuration file:

```
$ doxygen -g qat.doxygen
```

The above command will generate a template configuration file for a particular project.
Among others, you can edit the following options in the configuration file.

```
# document all entities in the project.
EXTRACT_ALL            = YES

# document all static members of a file.
EXTRACT_STATIC         = YES

# specify the root directory that contains the project's source files.
INPUT                  = /home/xmodulo/source

# search sub-directories for all source files.
RECURSIVE              = YES

# include the body of functions and classes in the documentation.
INLINE_SOURCES         = YES

# generate visualization graph by using dot program (part of graphviz package).
HAVE_DOT               = YES
```

Now go ahead and run doxygen with the configuration file.

```
$ doxygen qat.doxygen
```

Documentations are generated in both HTML and Latex formats, and store in `./html` and
`./latex` directories respectively.

To browse the HTML-formatted documentation, run the following.

```
$ cd html
$ firefox index.html
```

# References

- [How to generate documentation from source code in Linux](http://xmodulo.com/how-to-generate-documentation-from-source-code-in-linux.html)
- [How to browse and search API documentation offline on Linux](http://xmodulo.com/browse-search-api-documentation-offline-linux.html)
- [Doxygen Manual](http://www.stack.nl/~dimitri/doxygen/manual/index.html)


[Back](../)

