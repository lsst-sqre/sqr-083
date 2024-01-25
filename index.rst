#####################################################################
Patterns for accessing external resources from Times Square notebooks
#####################################################################

Abstract
========

.. abstract::

   Times Square is a Rubin Science Platform service that publishes parameterized rendered Jupyter Notebooks. One application for these notebooks is to generate reports that aggregate information from external sources. Since these Jupyter Notebooks are sourced from public GitHub repositories, authors can't embed API tokens. For some types of data access, it's necessary to create API proxy services that  effectively exchange a Gafaelfawr token in the notebook environment for access to the external resource. This technote discusses one instance of this pattern, the Jira Data Proxy service.

Add content here
================

See the `Documenteer documentation <https://documenteer.lsst.io/technotes/index.html>`_ for tips on how to write and configure your new technote.
