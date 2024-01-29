#####################################################################
Patterns for accessing external resources from Times Square notebooks
#####################################################################

.. abstract::

   Times Square is a Rubin Science Platform application that publishes parameterized Jupyter Notebooks.
   One application is to generate reports that aggregate information from external sources.
   Since Times Square sources Jupyter Notebooks from public GitHub repositories, authors can't embed API tokens.
   For some types of data access, it's necessary to create API proxy services that effectively exchange a Gafaelfawr token for access to an external service.
   This technote discusses a realization of this pattern, the `Jira Data Proxy`_ service.

Introduction
============

Times Square is a Rubin Science Platform application that publishes parameterized Jupyter Notebooks.
The notebooks are computed using the same type of Nublado JupyterLab servers that users work on.
For more information about Times Square's design, see :sqr:`062` :cite:`SQR-062`.

An application of Times Square is to generate and host reports that aggregate information from multiple sources, transform that data, and output customized text, tables, and figures (:sqr:`084`).
An example is a night observing report that aggregates information from the Engineering Facilities Database (EFD), narrativelog, exposurelog, and Jira system issue reports.
These external services generally require authentication.

In a user's JupyterLab, an author can get by with referring to a secrets file located in private storage for API keys.
Times Square notebooks generally can't follow this pattern because they run in Nublado JupyterLab servers under generic bot accounts (see the Noteburst design, :sqr:`065`, :cite:`SQR-065`).
Any storage accessible to Times Square notebooks must be accessible to any Notebook Aspect user in that Science Platform environment.
Running under a generic JupyterLab account simplifies the design of Noteburst, but also improves the accessibility of authoring Times Square notebooks because it guarantees that anyone can open a Times Square notebook in their own Nublado JuptyerLab server, make edits, and execute the notebook.

Another typical pattern for authenticating to APIs from notebooks is to embed the API key directly into the notebook source.
Generally, this is undesirable because it prevents credential rotation and makes credential leaks likely.
Credential leaks would be all but guaranteed since Times Square sources notebooks from public GitHub repositories.

This technote discusses the patterns we have developed for facilitating data access to external sources from Times Square notebooks.

Science Platform APIs
=====================

One type of service that *doesn't* need special treatment is Science Platform APIs.

Sasquatch InfluxDB (EFD) with Segwarides
----------------------------------------

The EFD Client provides a method for creating an authenticated InfluxDB client for Sasquatch.
Behind the scenes, this workflow uses Segwarides_ to get a shared read-only InfluxDB token for the user.
Segwarides is a SQuaRE Roundtable service that provides low-risk (generally read-only only) access tokens through a public API.
Essentially Segwarides provides anonymous read access to InfluxDB databases:

.. code-block:: python

   from lsst_efd_client import EfdClient

   efd_client = EfdClient("usdf_efd")

APIs that require Science Platform tokens
-----------------------------------------

To use APIs within the same Science Platform environment (e.g., narrativelog_, exposurelog_), clients need a Gafaelfawr token for bearer authorization.
The ``lsst.rsp.get_access_token()`` function provides this token for users:

.. code-block:: python

   from lsst.rsp import get_access_token
   import httpx


   token = get_access_token()
   r = httpx.get(
      "https://usdf-rsp.slac.stanford.edu/api/service/endpoint",
      headers={"Authorization": f"Bearer {token}"}
   )

Proxying third-party APIs
=========================

Third-party APIs present special challenges because they require API keys that we generally don't want to make widely available.
Consider Jira.
We cannot include a Jira API key in either the general notebook environment or through a service like Segwarides because these keys represent a specific user and provide read and write access to Jira data.
We need to both provide access to the Jira API only from the Nublado JupyterLab notebook environment, and also ensure that the access is read-only.

The solution we have developed is to create a proxy API that users access with a Gafaelfawr Rubin Science Platform token:

.. code-block:: python

   from lsst.rsp import get_access_token
   import httpx

   url = (
      "https://usdf-rsp.slac.stanford.edu/jira-data-proxy"
      "/rest/api/2/search?jql=project=DM&maxResults=10"
   )
   r = httpx.get(
      url,
      headers={"Authorization": f"Bearer {get_access_token()}"}
   )

The URL for the proxy service is ``https://usdf-rsp.slac.stanford.edu/jira-data-proxy``.
Any URL path and query parameter beyond that base URL is passed through to the Jira API.
With this pattern, a user can send any ``GET`` request to the Jira API, all using a Rubin Science Platform token.

This pattern is implemented in the `Jira Data Proxy`_ service, which we have deployed to the USDF Rubin Science Platform for Times Square notebook users.

Implementation of a proxy service
---------------------------------

A proxy service is simple to implement.
Below is a snippet of a proxy's handler function in FastAPI:

.. code-block:: python
   :caption: Handler module from Jira Data Proxy

   from urllib.parse import urlencode, urljoin
   
   from fastapi import APIRouter, Depends, Request, Response
   from httpx import AsyncClient
   from safir.dependencies.http_client import http_client_dependency
   from safir.dependencies.logger import logger_dependency
   from structlog.stdlib import BoundLogger
   
   from ..config import config
   
   __all__ = ["get_jira", "external_router"]
   
   external_router = APIRouter()
   """FastAPI router for all external handlers."""


   @external_router.get(
       "/{path:path}",
       description="Proxy GET requests to Jira.",
       name="proxy",
       response_model=None,
   )
   async def get_jira(
       path: str,
       request: Request,
       logger: BoundLogger = Depends(logger_dependency),
       http_client: AsyncClient = Depends(http_client_dependency),
   ) -> Response:
       """Proxy GET requests to Jira."""
       # Format the Jira URL. The Configuration model validates that
       # jira_base_url ends with a trailing slash. And path does not
       # start with a slash, so the paths can be concatenated.
       base_url = str(config.jira_base_url)
       if not base_url.endswith("/"):
           base_url += "/"
       url = urljoin(base_url, path, allow_fragments=False)
       if request.query_params:
           qs = urlencode(dict(request.query_params.items()))
           url = f"{url}?{qs}"
   
       r = await http_client.get(
           url,
           auth=(
               config.jira_username,
               config.jira_password.get_secret_value()),
           headers={"Accept": "application/json"},
       )
   
       pass_headers = ["content-type"]
       response_headers = {
           k: v for k, v in r.headers.items() if k.lower() in pass_headers
       }
       return Response(
           r.text, headers=response_headers, status_code=r.status_code
       )

Note that in this example implementation the proxy service receives the full response from the external API (i.e., Jira) before passing that response back to the user.
The performance of the proxy can be improved by streaming the response from the external API back to the user, which is possible with HTTPX and FastAPI.

Accessing APIs with shared read-only tokens
===========================================

Some API services provide fine-grained access control in API keys.
GitHub's `Personal Access Tokens <https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens>`__, for example, can be scoped to read-only access, and even limited to specific API types.
So rather than creating a proxy service for the GitHub API, it's realistic to share an API key from a bot GitHub user or GitHub App with all users in a staff-only Rubin Science Platform environment like the USDF.

A service like Segwarides_ is ideal for sharing such a key with users in a Rubin Science Platform environment.
However, we don't want to share GitHub keys anonymously over the internet, as Segwarides currently does (a malicious user could consume the API key's rate limit).
The solution is to put Segwarides — or a new service like it — behind the Rubin Science Platform's authentication.
Then a user would get a Gafaelfawr token with ``lsst.rsp.get_access_token()`` to then call the Segwarides API to in turn get the GitHub API key.
Once the user has that GitHub API key, they can access the GitHub API directly within the scopes afforded by the key.

Conclusion
==========

This technote has explored four patterns for accessing API resources from Times Square notebooks.
Accessing Sasquatch's InfluxDB (EFD) and Science Platform APIs is straightforward through existing Python APIs available to Notebook Aspect users.
For some third-party APIs, the best approach is to create a proxy service that exchanges a Gafaelfawr token for access to the third-party API.
For Rubin's Jira, we have deployed the `Jira Data Proxy`_ service to the USDF Rubin Science Platform.
Finally, some third-party APIs can be accessed directly with a shared read-only API key.
This pattern is not yet implemented, but could be done by modifying Segwarides_ to run behind the Rubin Science Platform's authentication.

References
==========

.. bibliography::

.. _`Jira Data Proxy`: https://github.com/lsst-sqre/jira-data-proxy
.. _Segwarides: https://github.com/lsst-sqre/segwarides
.. _narrativelog: https://github.com/lsst-sqre/narrativelog
.. _exposurelog: https://github.com/lsst-sqre/exposurelog
