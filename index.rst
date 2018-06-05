..
  Technote content.

  See https://developer.lsst.io/docs/rst_styleguide.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote is not yet published.**

   Design note on the possibilities for the Internet-facing endpoints of the LSST Science Platform deployments.

Introduction
============

The LSST Science Platform (LSP) presents three "Aspects" to its users: API, Portal, and Notebook.
The API Aspect provides a set of VO service endpoints, augmented by LSST-specific endpoints as needed.
The Portal Aspect appears as a Web application (or applications) to its users, but also provides service endpoints for the interaction with or customization of Portal components.
The Notebook Aspect presents a JupyterHub login and session-launching Web application, which brings the user to an instance of the JupyterLab Web application.
Each of these require Web endpoints.

The Science Platform will be deployed in several distinct "instances" in LSST:

- the U.S. Data Access Center;
- the Chilean Data Access Center;
- the Commissioning Cluster;
- the Science Validation (SV) instance;
- the pipeline developer environment; and
- the Prototype Data Access Center / integration environment.

Distinct endpoints are required for each Aspect's services in each instance.
(There is a possibility that the SV and pipeline developer instances could share some services or even be combined.)

Within an instance, the Aspects need to be able to communicate with each other, and therefore must have knowledge of each other's endpoints within that deployment.
Even within a single Aspect, there are internal components that need to communicate with each other.

User code running in the Notebook Aspect, or in Python plugins in the Portal Aspect,
or in the envisioned next-to-database processing component of the Data Access system,
also needs to be able to communicate with LSP endpoints.
Notable use cases include references to API Aspect services from user code in a notebook,
as well as references to Portal Aspect visualization services such as pushing an image to Firefly for display.
We wish to provide as transparent a user interface for those references as possible,
ideally providing instance-sensitive defaults.
For example, we wish a user to be able to use ``firefly_client`` or ``afw.display`` without being concerned about knowing the correct URL for the instance-specific Firefly server.

For these reasons, as well as for general considerations of ease of deployment and development, it is highly desirable for these inter-Aspect and intra-Aspect connections to all be "relocatable",
in the sense that all these connections can be redirected via a single instance-wide configuration parameter.

By default all LSP services will be made available via HTTPS.
Exceptions may be granted after change-control review including ISO feedback if there are strong performance or other technical grounds for them.

URL Structure
=============

Alternatives Considered
-----------------------

With the following substitutions:

*instance_base*
    Base DNS name for an entire instance of the LSP

*stem*
    A stem for a hostname component of a URL

*aspect*
    Aspect name: ``api``, ``portal``, or ``nb``

*service*
    URL pathname component(s) for Aspect-specific services, e.g., ``tap``, ``firefly``

where *aspect* and *service* are meant to be standardized names across all instances,
we considered the following alternative URL structures:

#. https://\ *instance_base*\ /\ *aspect*\ /\ *service*, e.g., ``https://data.lsst.org/api/tap``
#. https://\ *aspect*\ .\ *instance_base*\ /\ *service*, e.g., ``https://api.data.lsst.org/tap``
#. https://\ *stem*\ -\ *aspect*\ .\ *instance_base*\ /\ *service*, e.g., ``https://data-api.lsst.org/tap``

(The ``data.lsst.org`` URL is purely hypothetical,
chosen simply to avoid using any existing concrete instance as an example.)

In the context of Option 1, we also briefly considered, and rejected, promoting each API Aspect service to be a peer of the top-level URLs for the other Aspects.
This would have yielded, e.g., ``https://data.lsst.org/tap`` and ``https://data.lsst.org/sia`` for the various IVOA services provided.

Considerations
^^^^^^^^^^^^^^

Option 1 requires the least number of DNS names, and corresponding host certificates.

Option 2 would have permitted the use of delegated subdomains for the Science Platform instances,
as in this case *instance_base* serves as a domain name, not a full hostname.
Thus, in this model, one could imagine delegated subdomains ``lsst-pdac.ncsa.illinois.edu``, 
``lsst-sv.ncsa.illinois.edu``, ``lsst-lspdev.ncsa.illinois.edu``,
with the name resolution handled intra-instance.

By giving each Aspect for each instance its own effective hostname,
both Options 2 and 3 slightly simplify the mapping of external URLs to internal redirection services such as Kubernetes ingress controllers.
However, there are other ways to achieve this end result in Option 1.

Experience with the LSP deployments to date has suggested that it is likely to be very useful to make it easy to set up "pop-up" deployments in addition to the standard instances mentioned above.
The "relocatability" of instances appears to be simplest to arrange in Option 1,
where all Aspects share a common URL stem.
It is next simplest in Option 2, where all Aspects share a common subdomain;
the benefits of this would be strongest if delegated subdomains were used.
Option 3 seems the least well suited to this, with no instance-specific common core to the URL, but only a common name-construction pattern.

Proposal
--------

We have chosen Option 1 and thus are imagining URLs of the form:

- ``https://lsst-pdac.ncsa.illinois.edu/api``

  - ``https://lsst-pdac.ncsa.illinois.edu/api/tap``, ``.../api/sia``, etc.
- ``https://lsst-pdac.ncsa.illinois.edu/portal``

  - ``https://lsst-pdac.ncsa.illinois.edu/portal/firefly``, etc.
- ``https://lsst-pdac.ncsa.illinois.edu/nb``

  - ``https://lsst-pdac.ncsa.illinois.edu/nb/hub``, etc.

Variations for Testing
^^^^^^^^^^^^^^^^^^^^^^

The relocatability of the configuration of the LSP, if properly implemented, should facilitate the creation of transient instances of the entire LSP for testing purposes (in some cases this might be with supporting services, like Qserv, dummied out).

We also envision permitting parallel test instances of individual Aspects and services to be brought up within a running Aspect.
That is, it is possible to imagine putting a test instance of Firefly up underneath a ``https://lsst-pdac.ncsa.illinois.edu/portal-test`` URL.

It may be valuable to standardize the ``-test`` convention across Aspects to allow a common switching mechanism to be set up.

It is also possible to imagine a ``-test`` convention applied at the service level, e.g.,
``https://lsst-pdac.ncsa.illinois.edu/portal/firefly-test``.


Instance Naming
---------------

At the moment the only explicitly decided Option-1-style *instance_base* is ``lsst-lspdev.ncsa.illinois.edu`` for the pipeline developer LSP instance.

The *instance_base* values for the other instances mentioned in the Introduction above are not yet decided.
The PDAC, Science Validation, and Commissioning Cluster names should be decided soon.
For the public Data Access Centers, the DNS names to be used will likely depend on the outcome of current deliberations regarding the final name of the Project in the operations era.


Aspect-Specific Service Naming
------------------------------

The following subsections, to be written, will set out the basic plans from each aspect for the use of the pathname space below their main entry points.

API Aspect
^^^^^^^^^^

Portal Aspect
^^^^^^^^^^^^^

Notebook Aspect
^^^^^^^^^^^^^^^

.. Add content here.
.. Do not include the document title (it's automatically added from metadata.yaml).

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :encoding: latex+latin
..    :style: lsst_aa
