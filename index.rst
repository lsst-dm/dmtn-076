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

URL Structure
=============

Alternatives Considered
-----------------------

.. Add content here.
.. Do not include the document title (it's automatically added from metadata.yaml).

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :encoding: latex+latin
..    :style: lsst_aa
