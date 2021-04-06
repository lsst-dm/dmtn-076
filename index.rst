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

The Vera C. Rubin Observatory's Rubin Science Platform (RSP) presents three "Aspects" to its users: API, Portal, and Notebook.
The API Aspect provides a set of VO service endpoints, augmented by Rubin-specific endpoints as needed, representing the user-facing science data services of the RSP.
The Portal Aspect appears as a Web application (or applications) to its users, but also provides service endpoints for the interaction with or customization of Portal components.
The Notebook Aspect presents a JupyterHub login and session-launching Web application, which brings the user to an instance of the JupyterLab Web application.
Each of these Aspects is made available externally through Web endpoints.

The Science Platform is or will be deployed in several distinct "instances" in LSST:

- the Data Access Center at the U.S. Data Facility;
- the Chilean Data Access Center;
- the Interim Data Facility, hosted in the Google Cloud, in which three separate instances are deployed: production, integration, and development;
- a partial instance on the Summit, which will likely not have most, or any, of the VO data services;
- an integration instance for preparing updates to the Summit instance (currently hosted at NCSA);
- the Commissioning Cluster; and
- the inwardly-facing instances to support DM development at NCSA: the "lsp-stable" and "lst-int" instances.

Each instance has, in addition to the pages/URLs for its Aspects and services, a main landing page which welcomes the user to the RSP as a whole and directs them to the Aspects and to documentation on each.
This is the only page which is generally available to users without authentication.

Distinct endpoints are required for each Aspect's services in each instance.
In some cases, as for IVOA services, the pattern of endpoints for an individual service may be constrained by external standards.

Within an instance, the Aspects need to be able to communicate with each other, and therefore must have knowledge of each other's endpoints within that deployment.
Even within a single Aspect, there are internal components that need to communicate with each other.

User code running in the Notebook Aspect, or in Python plugins in the Portal Aspect,
or in the envisioned next-to-the-data processing component of the Data Access system,
also needs to be able to communicate with RSP endpoints.
Notable use cases include references to API Aspect services - e.g., TAP - from user code in a notebook,
as well as references to Portal Aspect visualization services such as pushing an image to Firefly for display.
We wish to provide as transparent a user interface for those references as possible,
ideally providing instance-sensitive defaults.
For example, we wish a user to be able to use ``firefly_client`` or ``afw.display`` from within the Notebook Aspect without being concerned about knowing the correct URL for the instance-specific Firefly server.

For these reasons, as well as for general considerations of ease of deployment and development, it is highly desirable for these inter-Aspect and intra-Aspect connections to all be "relocatable",
in the sense that all these connections can be redirected via a single instance-wide configuration parameter.

Note that the division of services into "Aspects" is primarily of pedagogical value and need not be directly reflected in the implementation.
However, reflection of the existence of the Aspects in *visible* parts of the implementation, such as URLs displayed in a browser or in code,
does reinforce the user-facing documentation and is preferable, all other things being equal.

In addition to the nominal public-facing services of the RSP, the implementation of the RSP uses additional services, some of which are also exposed externally but are not ones we highlight in the description for users of the RSP.
These also need to be accommodated in any scheme.
A key example is the central authentication and authorization service, which is discussed further below.

By default all RSP services will be made available via HTTPS.
Exceptions may be granted after change-control review including ISO feedback if there are strong performance or other technical grounds for them.

URL Structure Alternatives
==========================

With the following substitutions:

*instance_base*
    Base DNS name for an entire instance of the RSP

*instance_stem*
    A stem for a hostname component of a URL, shared by all services in a single instance of the RSP

*aspect*
    Aspect name: ``api``, ``portal``, or ``nb``; optionally extensible to provide endpoints for "implementation" services such as authorization

*service*
    URL pathname component(s) for Aspect-specific services, e.g., ``tap``, ``firefly``

where *aspect* and *service* are meant to be standardized names across all instances,
we have considered the following alternative URL structures, with the URLs for two instances (arbitrarily called ``data`` and ``data-int``) given for clarity.

#. https://\ *instance_base*\ /\ *aspect*\ /\ *service*, e.g., ``https://data.lsst.org/api/tap``, ``https://data-int.lsst.org/api/tap``
#. https://\ *aspect*\ .\ *instance_base*\ /\ *service*, e.g., ``https://api.data.lsst.org/tap``, ``https://api.data-int.lsst.org/tap``
#. https://\ *service*\ .\ *aspect*\ .\ *instance_base* , e.g., ``https://tap.api.data.lsst.org``, ``https://tap.api.data-int.lsst.org``
#. https://\ *instance_stem*\ -\ *aspect*\ .\ *instance_base*\ /\ *service*, e.g., ``https://data-api.lsst.org/tap``, ``https://data-int-api.lsst.org/tap``
#. https://\ *instance_stem*\ -\ *aspect*\ -\ *service*\ .\ *instance_base*\ , e.g., ``https://data-api-tap.lsst.org``, ``https://data-int-api-tap.lsst.org``

(NB: The ``data.lsst.org`` and ``data-int.lsst.org`` URLs, etc., are purely hypothetical,
chosen simply to avoid using any existing concrete instance as an example.)

Note that there are additional options in which the "api" is dropped and the individual API Aspect services are simply promoted to the same level in the hierarchy as ``portal`` and ``nb``.
We do not discuss those further at this time, as we consider them somewhat less desirable, for the pedagogical reasons mentioned above.

Evaluation
----------

Option 1 requires the least number of DNS names (only one per instance), and corresponding host certificates.
Option 1 also has the simplest recipe for the relocatability of RSP URLs from one instance to another.
The mapping is very clear: hostname = instance name; instance-specific services are defined in the following full pathname.
Option 1 has the substantive disadvantage (see `SQR-051 <https://sqr-051.lsst.io/>`__) that placing all the services under a common hostname allows cookies to "leak" across Aspect and service boundaries, enabling a class of attacks that could be mitigated with a diversity of hostnames.
This is a point of particular sensitivity for the authorization service itself, which, if its cookies are compromised, permits significant escalation of the severity of the consequences of a breach.
The Notebook Aspect service is also more sensitive, because of the ability of a Notebook Aspect user to run arbitrary commands within the user's RSP computing environment.

Option 1 also provides a natural home for the main RSP landing page, simply by not specifying an Aspect or service name: https://\ *instance_base*, e.g., ``https://data.lsst.org/``.
Options 2-5 would require adding to the specification a URL for this landing page, but, with some care with the management of DNS names, ``https://data.lsst.org/`` might be usable in all cases.
Note again that this is the only significant page in the RSP that is accessible pre-authentication.

Options 2-5 mitigate the severity of the cookie-based attacks enabled in Option 1, limiting a cookie's scope to a single Aspect or even a single service.
In these options, we recommend the identification of the primary authentication and authorization service as a separate entity at the *aspect* level of the hierarchy, i.e., ``auth``.
Separation of the Notebook and A&A services into their own hostnames provides the greatest marginal security benefit.
Therefore, Options 2 and 4, with separate hostnames only per-Aspect (plus ``auth``) may be sufficient for this purpose,
without the need to go to a hostname-per-service model.

Options 2 and 3 would permit the use of delegated subdomains for the Science Platform instances,
as in this case *instance_base* serves as a domain name, not a full hostname.
While more DNS names are required - it is not difficult to imagine substantially more than a hundred for Option 3 - the delegated-subdomain approach may make them easier to manage.

Options 4 and 5 require all the additional names (again, more in Option 5) to be included in the parent domain.
Depending on the DNS name management tools being used, this may or may not be more trouble than using a delegated subdomain.
These options also seem to be the least well suited to the desire for "relocatability":
"instance", "aspect", and potentially "service" are all mapped to the same hostname element.
In this model, the parallel construction of instances is manifest only in a convention for writing the initial hostname component.
There are therefore only "administrative controls", not "engineering controls", on the pathname hierarchy, and it is easy to imagine a messier outcome as individual implementers devise arguments for why they do not need to follow the pattern.

Some of the disadvantages of the Option 4/5 patterns can be mitigated by the provision of standard, common code to enable applications to do the work of reasoning out a partner service's URL endpoint from their own, so that the pattern need only be implemented once.
However, this would have to be done once per implementation language, certainly at least Python and JavaScript, and quite possibly also Java.

Variations for Testing
----------------------

The relocatability of the configuration of the RSP, if properly implemented, should facilitate the creation of transient instances of the entire RSP for testing purposes (in some cases this might be with supporting services, like Qserv, dummied out).

We have discussed the notion of permitting parallel test deployments of individual Aspects or even services to be brought up *within a single running instance.*
The idea discussed was permitting an Aspect name, or a single service name, to be postfixed with, e.g., ``-test``, to allow it to coexist with the standard version on a single instance of the RSP.
This was not implemented at the time of the initial "PDAC" deployment, and the development of the deployment tooling since then has made it even less desirable.
The baseline is that all testing of this nature will be done on a full test instance of the RSP, e.g., the integration instances ``lsst-lsp-int`` at NCSA or ``data-int.lsst.cloud`` at the IDF.


Usage To Date
=============

Option 1 was used for all the initial deployments at NCSA (e.g., ``https://lsst-lsp-stable.ncsa.illinois.edu/(aspect)``), the IDF (e.g., ``https://data.lsst.cloud`), and the Summit.
The security issues mentioned above were raised before these deployments began and were not considered in the original decision.

The pattern has worked well, absent that concern, and the transformation rule based on the hostname=instance mapping is embedded in a number of places in existing code.
A change to Option 2, or any other one, would require identifying and modifying those rules.
This should be done in a backwardly-compatible way, so that service code capable of running with an Option 2 (for example) pattern can also still be run in RSP instances using the existing Option 1 pattern.
This might require some special-casing of the recognition of legacy RSP URLs, but this seems preferable to a "hard fork" of the services.

The security argument for changing the existing design and segregating the authorization service to its own hostnames appears very strong.

If we wish to make a minimal change, we could keep all other services where they are, in an Option 1 pattern, and only add a single new hostname per instance, just for the auth service, e.g., ``(hostname)-auth.(domain)``.

Instance Naming
---------------

The existing instance bases, currently all deployed Option-1-style, are:

- ``lsst-lsp-stable.ncsa.illinois.edu`` - staff developer and Stack Club usage
- ``lsst-lsp-int.ncsa.illinois.edu`` - integration instance for the NCSA RSP
- ``data.lsst.cloud`` - IDF production instance, to be used for Data Previews
- ``data-int.lsst.cloud`` - integration instance for the IDF RSP
- ``data-dev.lsst.cloud`` - reserved for development work on low-level services
- ``lsst-nts-k8s.ncsa.illinois.edu`` - NCSA-based test stand for pre-deployment tests for the Summit instance
- ``summit-lsp.lsst.codes`` - Summit production instance (very limited access)
- any others?

Note that for the future public Data Access Centers' RSP instances, to be hosted at the USDF, the DNS naming patterns for the instance base hostnames/domains have not yet been determined and involve issues such as the branding of the project that are outside the scope of this note.


Path Forward
============

This is a matter of active discussion.
The present version of this note is only intended to lay out alternative more explicitly, to facilitate a decision.
The decision will be recorded here in a later release of this note.


Intra-Aspect Service Naming
===========================

In order to supply additional context, the following subsections, to be fleshed out over time, will set out the basic plans from each Aspect for the use of the pathname space below their main entry points.

API Aspect
----------

Existing services:

- ``/api/tap`` - IVOA TAP service parent endpoint (e.g., ``/api/tap/async`` for asynchronous queries)
- ``/api/obstap`` - independent, experimental ObsTAP service

Portal Aspect
-------------

- ``/portal`` - reserved for future use as a user-friendly welcome page, with documentation and referring the user to a variety of possible starting points for using the Portal application
- ``/portal/app`` - the actual Portal React Web application, on RSP instances having functional TAP and other IVOA services
- ``/portal/firefly`` - the "vanilla Firefly" application, used on limited-functionality RSP instances having no data services of their own

Notebook Aspect
---------------

- ``/nb`` - main landing page for the Notebook Aspect
- ``/nb/hub`` and descendents - JupyterHub components associated with the startup and lifetime control of a user session
- ``/nb/user/(username)`` and descendents - JupyterLab components associated with a user's specific session


.. Add content here.
.. Do not include the document title (it's automatically added from metadata.yaml).

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :encoding: latex+latin
..    :style: lsst_aa
