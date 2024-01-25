.. image:: https://img.shields.io/badge/sqr--083-lsst.io-brightgreen.svg
   :target: https://sqr-083.lsst.io
.. image:: https://github.com/lsst-sqre/sqr-083/workflows/CI/badge.svg
   :target: https://github.com/lsst-sqre/sqr-083/actions/

#####################################################################
Patterns for accessing external resources from Times Square notebooks
#####################################################################

SQR-083
=======

Times Square is a Rubin Science Platform service that publishes parameterized rendered Jupyter Notebooks. One application for these notebooks is to generate reports that aggregate information from external sources. Since these Jupyter Notebooks are sourced from public GitHub repositories, authors can't embed API tokens. For some types of data access, it's necessary to create API proxy services that  effectively exchange a Gafaelfawr token in the notebook environment for access to the external resource. This technote discusses one instance of this pattern, the Jira Data Proxy service.

**Links:**

- Publication URL: https://sqr-083.lsst.io
- Alternative editions: https://sqr-083.lsst.io/v
- GitHub repository: https://github.com/lsst-sqre/sqr-083
- Build system: https://github.com/lsst-sqre/sqr-083/actions/


Build this technical note
=========================

You can clone this repository and build the technote locally if your system has Python 3.11 or later:

.. code-block:: bash

   git clone https://github.com/lsst-sqre/sqr-083
   cd sqr-083
   make init
   make html

Repeat the ``make html`` command to rebuild the technote after making changes.
If you need to delete any intermediate files for a clean build, run ``make clean``.

The built technote is located at ``_build/html/index.html``.

Publishing changes to the web
=============================

This technote is published to https://sqr-083.lsst.io whenever you push changes to the ``main`` branch on GitHub.
When you push changes to a another branch, a preview of the technote is published to https://sqr-083.lsst.io/v.

Editing this technical note
===========================

The main content of this technote is in ``index.rst`` (a reStructuredText file).
Metadata and configuration is in the ``technote.toml`` file.
For guidance on creating content and information about specifying metadata and configuration, see the Documenteer documentation: https://documenteer.lsst.io/technotes.
