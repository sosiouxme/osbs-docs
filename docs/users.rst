Building Container Images
=========================

Building images using fedpkg
----------------------------

Building images using koji
--------------------------

Streamed build logs
~~~~~~~~~~~~~~~~~~~

When atomic-reactor in the orchestrator build runs its
`orchestrate_build` plugin and watches the builds, it will stream in
the logs from those builds and emit them as logs itself, with the
platform name as one of the fields. The extra fields for these worker
logs will be: platform, level.

Note that there will be a single Koji task with a single log output,
which will contain logs from multiple builds. When watching this using
``koji watch-logs <task id>`` the log output from each worker build
will be interleaved. To watch logs from a particular worker build
image owners can use ``koji watch-logs <task id> | grep -w x86_64``.


Configuring koji-containerbuild for koji CLI
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Building images using osbs-client
---------------------------------

Accessing built images
----------------------

Building a buildroot using atomic-reactor
-----------------------------------------

Using atomic-reactor plugins
----------------------------

Creating a build JSON
---------------------

Writing a Dockerfile
--------------------

A Dockerfile is required for building container images in OSBS. It must be
placed at the root of git repository. There can only be a single Dockerfile per
git repository branch.

Some labels are required to be defined:

- ``com.redhat.component``: the value of this label is used when importing a
  build into Koji via content generator API.
- ``name``: value is used to define the repository name in container registry to
  push built image. Limit this to lowercase alphanumerical values with the
  possibility to use dash as a word separator. A single ``/`` is also allowed.
  ``.`` is not allowed in the first section. For instance, **fed/rsys.log** and
  **rsyslog** are allowed, but **fe.d/rsyslog** and **rsys.log** aren't.
- ``version``: used as the version portion of Koji build NVR, as well as, for
  the version tag in container repository.

For example::

    LABEL com.redhat.component=rsyslog-docker \
          name=fedora/rsyslog \
          version=32

When OSBS builds a container image that defines the above labels, a Koji build
will be created in the format rsyslog-docker-32-X. Where X is the release value.
The container image will be available in container registry at:
``my-container-registry.example.com/fedora/rsyslog:32``.

The ``release`` label can also be used to specify the release value use for Koji
build. When omitted, the release value will be automatically determined by
querying Koji's getNextRelease API method.

.. _image-configuration:

Image configuration
-------------------

Some aspects of the container image build process are controlled by a
file in the git repository named ``container.yaml``. This file need
not be present, but if it is it must adhere to the `container.yaml
schema`_.

.. _`container.yaml schema`: https://github.com/projectatomic/atomic-reactor/blob/master/atomic_reactor/schemas/container.json

An example::

  ---
  platforms:
    # all these keys are optional

    only:
    - x86_64   # can be a list (as here) or a string (as below)
    - ppc64le
    - armhfp
    not: armhfp

  compose:
    # used for requesting ODCS compose of type "tag"
    packages:
    - nss_wrapper  # package name, not an NVR.
    - httpd
    - httpd-devel
    # used for requesting ODCS compose of type "pulp"
    pulp_repos: true
    # used for requesting ODCS compose of type "module"
    modules:
    - "module_name1:stream1"
    - "module_name2:stream1"
    # Possible values, and default, are configured in OSBS environment.
    signing_intent: release

platforms
~~~~~~~~~

Keys in this map relate to multi-platform builds. The full set of
platforms for which builds may be required will come initially from
the Koji build tag associated with the build target, or from the
``platforms`` parameter provided to the ``create_orchestrator_build``
API method when Koji is not used.

only
  list of platform names (or a single platform name as a string); this
  restricts the platforms to build for using set intersection

not
  list of platform names (or a single platform name as a string);
  this restricts the platforms to build for using set difference

.. _container.yaml-compose:

compose
~~~~~~~

This section is used for requesting yum repositories at build time. When this
section is defined, a compose will be requested by using ODCS.

packages
  list of package names to be included in ODCS compose. Package in this case
  refers to the "name" portion of the NVR (name-version-release) of an RPM, not
  the Koji package name. Packages will be selected based on the Koji build tag
  of the Koji build target used. The following command is useful in determining
  which packages are available in a given Koji build tag:
  ``koji list-tagged --inherit --latest TAG``


pulp_repos
  boolean to control whether or not an ODCS compose of type "pulp" should be
  requested. If set to true, ``content_sets.yml`` must also be provided. A
  compose will be requested for each architecture in ``content_sets.yml``.
  See :ref:`content_sets.yml`.

modules
  list of modules for requesting ODCS compose of type "module".

signing_intent
  used for verifying packages in yum repositories are signed with expected
  signing keys. The possible values for signing intent are defined in OSBS
  environment. See :ref:`config.yaml-odcs` section for environment configuration
  details, and full explanation of :ref:`signing-intent`.

**If "compose" section is defined, non-empty list of "modules" or "packages" must
be provided, or "pulp_repos" set to true. Otherwise, build will fail.**

**The "packages" and "modules" keys are mutually exclusive. If both are
provided, "modules" will be ignored. "pulp_repos" can be used by itself, or with
either "packages" or "modules".**


.. _content_sets.yml:

Content Sets
------------

The file ``content_sets.yml`` is used to define the content sets relevant to the
container image.  This is relevant if RPM packages in container image are in
pulp repositories. See ``pulp_repos`` in :ref:`container.yaml-compose` for how
this file is used during build time.

An example::

  ---
  x86_64:
  - server-rpms
  - server-extras-rpms

  ppx64le:
  - server-for-power-le-rpms
  - server-extras-for-power-le-rpms

Using Artifacts from Koji
-------------------------

During a container build, it might be desireable to fetch some artifacts
from an existing Koji build. For instance, when building a Java-based
container, JAR archives from a Koji build are required to be added to
the resulting container image.


The atomic-reactor pre-build plugin, fetch_maven_artifacts, can be used
for including non-RPM content in a container image during build time.
This plugin will look for the existence of two files in the git repository
in the same directory as the Dockerfile:
fetch-artifacts-koji.yaml and fetch-artifacts-url.yaml.

The first is meant to fetch artifacts from an existing Koji build.
The second allows specific URLs to be used for fetching artifacts.
fetch-artifacts-koji.yaml will be processed first.

fetch-artifacts-koji.yaml
~~~~~~~~~~~~~~~~~~~~~~~~~

::

  - nvr: foobar # All archives will be downloaded

  - nvr: com.sun.xml.bind.mvn-jaxb-parent-2.2.11.redhat_4-1
    archives:
    # pull a specific archive
    - filename: jaxb-core-2.2.11.redhat-4.jar
      group_id: org.glassfish.jaxb

    # group_id omitted - multiple archives may be downloaded
    - filename: jaxb-jxc-2.2.11.redhat-4.jar

    # glob support
    - filename: txw2-2.2.11.redhat-4-*.jar

    # pull all archives for a specific group
    - group_id: org.glassfish.jaxb

    # glob support with group_id restriction
    - filename: txw2-2.2.11.redhat-4-*.jar
      group_id: org.glassfish.jaxb

    # causes build failure due to unmatched archive
    - filename: archive-filename-with-a-typo.jar

Each archive will be downloaded to artifacts/<mavenfile_path> at the root
of git repository. It can be used from Dockerfile via ADD/COPY instruction:

::

  COPY \
    artifacts/org/glassfish/jaxb/jaxb-core/2.2.11.redhat-4/jaxb-core-2.2.11.redhat-4.jar /jars

The directory structure under ``artifacts`` directory is determined
by ``koji.PathInfo.mavenfile`` method. It’s essentially the end of
the URL after ``/maven/`` when downloading archive from Koji Web UI.

Upon downloading each file, the plugin will verify the file checksum by
leveraging the checksum value in the archive info stored in Koji. If
checksum fails, container build fails immediately. The checksum algorithm
used is dictated Koji via the `checksum_type` vaue of archive info.

If build specified in nvr attribute does not exist, the container
build will fail.

If any of the archives does not produce a match, the container build will fail.
In other words, every item in the archives list is expected to match at least
one archive from specified Koji build. However, the build will not fail if it
matches multiple archives.

*Note that only archives of maven type are supported.* If in the nvr
supplied an archive item references a non maven artifact, the container
build will fail due to no archives matching request.


fetch-artifacts-url.yaml
~~~~~~~~~~~~~~~~~~~~~~~~

::

  - url: http://download.example.com/JBossDV/6.3.0/jboss-dv-6.3.0-teiid-jdbc.jar
    md5: e85807e42460b3bc22276e6808839013
  - url: http://download.example.com/JBossDV/6.3.0/jboss-dv-6.3.0-teiid-javadoc.jar
    # Use different hashing algorithm
    sha256: 3ba8a145a3b1381d668203cd73ed62d53ba8a145a3b1381d668203cd73ed62d5
    # Optionally, overwrite target name
    target: custom-dir/custom-name.jar

Each archive will be downloaded to artifacts/<target_path> at the root
of git repository. It can be used from Dockerfile via ADD/COPY instruction:

::

  COPY artifacts/jboss-dv-6.3.0-teiid-jdbc.jar /jars/
  COPY artifacts/custom-dir/custom-name.jar /jars/

By default, target_path is set to the filename from provided url. It can
be customized by providing a target. The target value can be either a
filename, archive.jar, or also include a path, my/path/archive.jar, for
easier archive management.

The md5, sha1, sha256 attributes specify the corresponding hash to be used
when verifying artifact was downloaded properly. At least one of them is
required. If more than one is defined, multiple hashes will be computed
and verified.


Koji Build Metadata Integration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In the future, a reference of each artifact fetched by OSBS will be
added to the koji build metadata once imported via content generator API.
The list of components for the container image in output list will
include the fetched artifacts in addition to the installed RPMs.

.. _image-tags:

Image tags
----------

The output from atomic-reactor includes container images tagged into a
registry (or Pulp, if Pulp integration is enabled). In addition, when
multi-platform builds are enabled each set of images will be grouped
into a manifest list, which itself is tagged.

While the repository name is specified by the ``name`` label in the
Dockerfile, the tags used within the repository are:

- a unique tag including the timestamp (this tag is the only tag
  applied for scratch builds)
- ``latest``
- ``{version}`` (the ``version`` label)
- ``{version}-{release}`` (the ``version`` and ``release`` labels together)
- any additional tags configured in the git repository (named in the
  ``additional-tags`` file)

These tags are applied to the manifest list or, if multi-platform
image builds are not enabled (see :ref:`group_manifests
<group-manifests>`), to the sole image manifest resulting from the
build.

Override Parent Image
----------------------

The parent image used for building a layered image is determined by the ``FROM``
instruction in the Dockerfile by default. Users can override this behavior
by specifying a koji parent build via the ``koji_parent_build`` API parameter.
When given, the image reference in the provided koji parent build will be used as
the value of the FROM instruction. The same source registry restrictions
apply.

Additionally, the koji parent build must use the same container image repository
as the value of the FROM instruction in Dockerfile. For instance, if the
Dockerfile states ``FROM fedora:27``, the koji parent build has to be of a
container image that pushed to the ``fedora`` repository. The koji parent build
may refer to a ``fedora:26`` image, but using a koji parent build for an image
that was pushed to ``rsyslog`` will cause a build failure.

This behavior requires koji integration to be enabled in the OSBS environment.

Koji NVR
--------

When koji integration is enabled, every container image build requires a unique
Name-Version-Release, NVR. The Name and Version are extracted from the **name**
and **version** labels in Dockerfile. Users can also use the **release** label
to hard code the release value, although this requires a git commit for every
build to change the value. A better alternative is to leave off the **release**
label which causes OSBS to query koji for what the next release value should be.
This is done via koji's ``getNextRelease`` API method. In either case, the
release value can also be overridden by using the ``release`` API parameter.

Isolated Builds
---------------

When a build is created via OSBS, the built container image is pushed to the
container registry updating various tag references in the container registry.
Some of these tag references are unique, while others are meant to transition
over time to reference a newer image, for instance ``latest`` and ``{version}``
tags. In other words, building a container image usually updates these
transient, non-unique, tags in container registry. In some cases, this is not
desired.

Consider the case of a container image that includes one, or more, packages that
have recently been identified as containing security vulnerabilities. To address
this issue, a new container image must be built. The difference for this build
is that only changes related to the security fix must be applied. Any unrelated
development that has occurred should be ignored. It would not be correct to
update the ``latest`` tag reference with this build.  To achieve this, the
concept of isolated builds was introduced.

As an example, let's use the image ``rsyslog`` again. At some point the
container image 7.4-2 is released (version 7.4, release 2). Soon after, minor
bug fixes are addressed in 7.4-3, a new feature is added to 7.4-4, and so on. A
security vulnerability is then discovered in the released image 7.4-2. To
minimize disruption to users, you may want to build a patched version of 7.4-2,
say 7.4-2.1. The packages installed in this new container image will differ from
the former only when needed to address the security vulnerability. It will not
include the minor bug fixes from 7.4-3, nor the new features added in 7.4-4. For
this reason, updating the ``latest`` tag is considered incorrect.

::

    7.4 version
    |
    |____
    |   |1 release
    |
    |__________________
    |   |2 release    |2.1 release
    |
    |____
    |   |3 release
    |
    |____
    |   |4 release
    |

To start an isolated build, use the ``isolated`` boolean parameter. Due to the
nature of isolated builds, the release value must be set via the ``release``
parameter which must match the format ``^\d+\.\d+(\..+)?$``

Isolated builds will only update the ``{version}-{release}`` unique tag and the
primary tag in target container registry.

Yum repositories
----------------

In most cases, part of the process of building container images is to install
RPM packages. These packages must come from yum repositories. There are various
methods for making a yum repository available for your container build.

.. _yum-repositories-odcs-compose:

ODCS compose
~~~~~~~~~~~~

The preferred method for injecting yum repositories in container builds is by
enabling ODCS integration via the "compose" key in ``container.yaml``. See
:ref:`image-configuration` and :ref:`signing-intent` for details.

RHEL subscription
~~~~~~~~~~~~~~~~~

If the underlying host is Red Hat Enterprise Linux (RHEL), its subscriptions
will be made available during container builds. Note that changes in the
underlying host to enable/disable yum repositories is not reflected in container
builds. ``Dockerfile`` must explicitly enable/disable yum repositories as
needed. Although this is desirable in most cases, in an OSBS deployment it can
cause unexpected behavior. It's recommended to disable subscription for RHEL
hosts when they are being used by OSBS.

Yum repository URL
~~~~~~~~~~~~~~~~~~

As part of a build request, you may provide the ``repo-url`` parameter with the
URL to a yum repository file. This file is injected into the container build.
Note that "ODCS compose" method is disabled if this parameter is given.

Koji tag
~~~~~~~~

When Koji integration is enabled, a Koji build target parameter is provided. The
yum repository for the build tag of target is automatically injected in
container build. This behavior is disabled if either "ODCS compose" or "Yum
repository URL" are used.

.. _signing-intent:

Signing intent
--------------

When the "compose" section in ``container.yaml`` is defined, ODCS composes will
be requested at build time. ODCS is aware of RPM package signatures and can be
used to ensure that only signed packages are added to the generated yum
repositories. Ultimately, this can be used to ensure a container image only
contains packages signed by known signing keys.

Signing intents are an abstraction for signing keys. It allows the OSBS
environment administrator to define which signing keys are valid for different
types of releases. See :ref:`config.yaml-odcs` section for details.

For instance, an environment may provide the following signing intents:
``release``, ``beta``, and ``unsigned``. Each one of those intents is then
mapped to a list of signing keys. These signing keys are then used during ODCS
compose creation. The packages to be included must have been signed by any of
the signing keys listed. In the example above, the intents could be mapped to
the following keys::

    # Only include packages that have been signed by "my-release-key"
    release -> my-release-key
    # Include packages that have been signed by either "my-beta-key" or
    # "my-release-key"
    beta -> my-beta-key, my-release-key
    # Do not check signature of packages - may include unsigned packages
    unsigned -> <empty>

The signing intents are also defined by their restrictive order, which will be
enforced when building layered images. For instance, consider the case of two
images, X and Y. Y uses X as its parent image (FROM X). If image X was built
with "beta" intent, image Y's intent can only be "beta" or "unsigned". If the
dist-git repo for image Y has it configured to use "release" intent, this value
will be downgraded to "beta" at build time.

Automatically downgrading the signing intent, instead of failing the build, is
important for allowing a hierarchy of layered images to be built automatically
by ``ImageChangeTriggers``. For instance, with Continuous Integration in mind, a
user may want to perform daily builds without necessarily requiring signed
packages, while periodically also producing builds with signed content. In this
case, the ``signing_intent`` in ``container.yaml`` can be set to ``release`` for
all the images in hierarchy. Whether or not the layered images in the hierarchy
use signed packages can be controlled by simply overriding the signing intent of
the top most ancestor image. The signing intent of the layered images would then
be automatically adjusted as needed.

In the case where multiple composes are used, the least restrictive intent is
used. Continuing with our previous signing intent example, let's say a container
image build request uses two composes. Compose 1 was generated with no signing
keys provided, and compose 2 was generated with "my-release-key". In this case,
the intent is "unsigned".

Compose IDs can be passed in to OSBS in a build request. If one or more compose
IDs are provided, OSBS will classify the intent of the existing compose. This
is done by inspecting the signing keys used for generating the compose and
performing a reverse mapping to determine the signing intent. If a match cannot
be determined, the build will fail. Note that if given compose is expired or
soon to be expired, OSBS will automatically renew it.

The ``signing_intent`` specified in ``container.yaml`` can be overridden with
the build parameter of same name. This particular parameter will be ignored for
autorebuilds. The value in ``container.yaml`` should always be used in that
case. Note that the signing intent used by the compose of parent image is still
taken into account which may lead to downgrading signing intent for the layered
image.

The Koji build metadata will contain a new key,
``build.extra.image.odcs.signing_intent_overridden``, to indicate whether or not
the ``signing_intent`` was overridden (CLI parameter, automatically downgraded,
etc). This value will only be ``true`` if
``build.extra.image.odcs.signing_intent`` does not match the ``signing_intent``
in ``container.yaml``.
