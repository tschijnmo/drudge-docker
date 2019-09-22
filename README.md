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


Development workflow
--------------------

In addition to being the platform to build docker images for drudge, this
repository can also serve as the ground for drudge development.  The images can
be easily launched as services, with the source tree and a swap working
directory on the host machine mounted for convenience.  For instance, to do
some drudge development, `docker-compose up drudge` can be followed by a
`docker exec -it` in the launched container to do the development.  Changes to
the source code can immediately be visible.

