Docker for the drudge stack
===========================

This repository has all that is needed to build docker images for the
development and execution of the entire drudge stack.  Note that the images are
not (yet) optimized to be super lean for deployment, but rather for the
convenience of development.

We start with the `base` image, which contains all the relatively stable
dependencies.  Then the major drudge projects are introduced as git submodules.
Docker-compose in the root directory can be used to build all the images.  Note
that to build from scratch, it needs to be done in an order topologically
sorted according to the dependency.

Note that inside the compose file, all the images are given tags in the
`drudge` repository in the root namespace without an explicit user.  For
publishing into a remote registry, proper tagging is needed.  Different stages
in the stack are tagged with plain name, with `-latest` omitted for
convenience.  Specific versions will be marked when needed.

