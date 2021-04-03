.. note::

   **This technote is not yet published.**

Abstract
========

We present an exploration of the possible roles of the IVOA UWS (Universal Worker Service) in Rubin Observatory systems, including both near-real-time processing in the OCS-Controlled Pipeline System (OCPS) and in data services in the Rubin Science Platform (RSP).
We also briefly look at some implementation issues and options.

Motivation
==========

This note covers the possibility of applying a common service implementation pattern, based
on the IVOA UWS standard, to a number of applications in the Rubin Observatory computing
systems where asynchronously-triggered, potentially long-running computational tasks need
to be made available behind a Web API.

The use cases all envision the actual computation as being defined, or at least limited in
scope, in advance.
The use of this pattern to solve the general "user batch service" problem (which itself is
still very much an open problem) is not contemplated at this time.

A number of use cases arise from existing elements of the Rubin design, most of long standing:

1. The OCS-Controlled Pipeline System (OCPS), also previously called "OCS-controlled batch";
2. The need for additional IVOA-compatible services as part of the image service architecture,
   including services for the re-creation of virtual data products on demand, and services
   for obtaining "film-strip cutouts" of a target in all LSST epochs; and
3. On-demand computational services such as forced photometry.

We describe these in turn below.

Additional relevant context is that we already use the UWS pattern and standard in the
context of the TAP service, which is mandated by the IVOA TAP standard to provide 
asynchronous query services under a UWS-based framework.
The existing TAP server implementation, obtained from CADC, already implements this.
It is therefore not a candidate for reimplementation over the generic UWS framework
discussed elsewhere in this note, but it does strengthen the case for a wider application
of UWS, particularly because it opens up the possibility of the use of generic UWS clients
for accessing multiple services and monitoring job status.

OCS-Controlled Pipeline System
------------------------------

An element of the interface design between DM and the OCS has long been the provision of a
computational service by DM with an interface allowing it to be controlled like one of the
real-time Observatory components.
This means that it would appear as a "CSC" (Commandable SAL Component) to the OCS, obeying
the OCS command protocol.
The intent of the design is that it provide a flexibly configurable mechanism for executing
DM-style computational tasks in response to a command received through this interface.
Ideally this would allow the OCPS to be configured with a number of selectable tasks,
either by different SAL commands or selected via a parameter of a single generic "run" SAL
command.

The OCPS would have the ability to return a result directly to the OCS via its SAL interface,
and/or to deposit a larger result, or a set of results, in an appropriate location.

The OCPS has been envisioned as providing the framework for executing tasks such as:

- computing daily synthetic calibration data products upon completion of afternoon
  calibration data acquisition;
- performing on-demand evaluations of whole-camera focus-stepping datasets, analyzing
  them for "donuts" and a whole-camera wavefront solution; and
- (additional examples would be useful).

Many of the "payloads" envisioned for the OCPS would likely be implementable as 
Gen3 Pipelines, with the data on which the payload would act accessible via a Butler
with access to near-real-time data, presumably via the Observatory Operations Data
Service (OODS).

A framework implementation allowing the OCPS to run any suitable Gen3 pipeline would be
very helpful in this regard, but it does not appear acceptable to strictly limit the
OCPS to running *only* pipelines of this nature.
Most likely the OCPS will have to support some payloads that are implemented at a lower
layer than a Gen3 Pipeline; but the UWS pattern could still be applicable.


IVOA-standard and IVOA-style services in the image service architecture
-----------------------------------------------------------------------

As part of the implementation of the image service architecture (to be described in DMTN-139),
we envision the provision of a number of potentially computationally-intensive services,
notably image cutouts (based on the IVOA SODA standard), especially multi-cutout services,
virtual-data-product-recreation services, and mosaicing services.

Invocation of image services
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This is discussed in greater detail in DMTN-139, but in summary: the architecture envisions
user (and client software) being directed to the invocation of cutout and image-recreation
services via DataLink service descriptors.
In order to enable such a metadata-driven, generic architecture to succeed, the service
invocations should follow a common pattern.
We intend to achieve this with widespread use of the DALI, VOSI, and UWS patterns, even for
services that are not fully standardized (e.g., virtual-data-product-recreation services
do not fit the SODA standard perfectly, but all the above subsidiary standards are still
applicable).

A common UWS interface to such services allows the creation of a common monitoring 
interface for users to watch their jobs - either through a UI or a (Python) client library.
A common interface for the provision of results of jobs allows a single client to provide
users access to the results of jobs of a variety of types.

Image cutouts
^^^^^^^^^^^^^

For "classical" image cutouts we will follow the SODA standard.
The existing version of the standard is quite narrow in scope and, taken literally,
covers only the basic case of cutouts without scalable reprojection, but can be
extended in a natural way to cover more complex cases without significant change to
the overall service pattern.
(We will also be involved in future efforts to extend the standard itself.)

The SODA standard envisions the provision of asynchronous cutout service for
potentially long-running jobs, and these would follow the UWS pattern.

No existing SODA service implementation seems suitable for use in the RSP, because
of the need for interaction with the Rubin Science Pipelines software stack to
perform optimal reprojections.

We envision (again, see DMTN-139) an implementation based on a layered stack of
Tasks to do the cutout mathematics, PipelineTasks to encapsulate the I/O, and a
framework for executing a PipelineTask Pipeline as a UWS job.

Rubin-specific image services
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

More general calculational services
-----------------------------------

A forced-photometry-on-demand service has been envisioned as part of the user-facing
Rubin Observatory services from early on, and is called for in the requirement
DMS-REQ-0341.
At times it has been proposed that the availability of the Nublado environment
can meet this need, but the requirement appears to the author to call for a
scriptable *Web API*, not just a Python interface usable in Nublado.

The use case of an automated service to analyze data in the vicinity of GWB alerts,
obtaining targets from galaxy catalogs (in the spatial and redshift region indicated
by the alert) seems a natural one to satisfy via external API, allowing the analysis
of Rubin data to be embedded in a more complex service built by others.

A Gen3 Pipeline-based service that takes as input a table of coordinates and
optional shape parameters, and returns forced photometry results, at the user's
choice, from single-epoch images (satisfying DMS-REQ-0341) or coadds (a natural
extension), is a good candidate for implementation as a UWS service.


Existing UWS-based services
---------------------------


Concerns with the use of UWS
----------------------------

The UWS standard is relatively complex.
It envisions a non-trivial state diagram for job execution, and it has a complex
invocation interface based on HTTP(S) ``POST`` operations.
It has options for both polling and blocking monitoring of job status,
and it has an results interface that supports a variable number of results
from any given job.

There is little doubt that UWS is more complex than might be appropriate for the
implementation of any one of the above services.
This affects both the complexity of implementation of the server and of client software.

However, it is now a community standard, and both server and client (UI and Python) implementations
are already available that have tackled much of this complexity.
If additional tool development is needed, the Rubin team can contribute back to these
community tools.

The author believes that, in preference to inventing potentially simpler, but unique, APIs for the
services discussed herein, the use of a common UWS interface facilitates the development of an
ecosystem of tools.
If all long-running user jobs are in the UWS framework, then a user can monitor all their
in-flight jobs, whether cutouts, forced photometry, or image re-creations, in a single
UI and/or through a single Python API.
If all results are returned through a common API, then the UI or Python client can retrieve
any such result and visualize it or instantiate it locally in memory.

Common use of UWS between the OCPS and the user-facing RSP services will facilitate the
use of common tools across the two domains.
This seems useful since many Rubin staff and community scientists will be involved both
in working with the RSP services and with participating in commissioning.

Overview of UWS
===============


Conceptual Design
=================


Implementation Options
======================


Related Documents
=================

DMTN-139: Image Service Architecture


..
  Technote content.

  See https://developer.lsst.io/restructuredtext/style.html
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


.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
