---
title: Writing a v2 CloudFoundry Service
---

_Notice: This document is a work-in-progress, as is this version of the API.  As long as this message is here,
this <strong>API is subject to change at any time</strong>._

There are 2 versions of the SERVICES API, this document represents version 2 of that API.  To see the older version,
visit [Writing a v1 Cloud Foundry Service](writing-service.html).  What's different in v2:

* Unidirectional RESTful HTTP messages (Brokers can run standalone)
* Easier catalog management and orphan detection (No logic in the broker)
* API endpoints and fields are consistently named
* Automatic cleanup/prevention of orphans (Inconsistent instances between broker and CC)

The v2 broker API (previously called a service `gateway` in v1) defines the contract between CloudController (CC) and the
service broker.  The broker is expected to implement several HTTP (or HTTPS) endpoints underneath a URI prefix.
A single service can only be provided by a single broker URL,
but horizontal scalability and load balancing allows for multiple redundant brokers.
A broker can also offer multiple services.

## <a id="registering"></a>Authentication ##

CC authenticates to the Broker using HTTP basic authentication (the `Authentication:` header) on every request,
and it will reject any broker registrations that do not contain a username and password.
The *broker is responsible for checking the username and password* and returning `403 Forbidden` if they are invalid.
CC supports connecting to a broker using SSL if additional security is desired.

## <a id="registering"></a>Registering a Broker ##

Before a broker can be using with Cloud Foundry, the operator must register the broker with CloudController.

You can use the `cf add-service-broker` command, which takes the place of the old cf create-service-auth-token.  Or
you can make an API request to the CC API as follows:

`POST /v2/service_brokers`

<table>
<thead>
<tr>
  <th>Request field</th>
  <th>Type</th>
  <th>Description</th>
</tr>
</thead>
<tbody>
<tr>
  <td>name*</td>
  <td>string</td>
  <td>A user-friendly name for the service broker that you are adding</td>
</tr>
<tr>
  <td>broker_url*</td>
  <td>string</td>
  <td>The URL prefix that a broker can be reached from CC.  Usually something like `http://host:port/`, but it can use
  SSL and a path. Basic-auth credentials cannot be supplied in this URL.</td>
</tr>
<tr>
  <td>username*</td>
  <td>string</td>
  <td>The basic-auth username to connect to the broker with</td>
</tr>
<tr>
  <td>password*</td>
  <td>string</td>
  <td>The basic-auth password to connect to the broker with</td>
</tr>
</tbody>
</table>

When you register a broker, CC will synchronously download your service catalog (explained next).  If it is unable
to download the catalog, or the catalog is invalid, CC will reject the API request with an error describing what
went wrong.

## <a id="catalog"></a>Catalog Management ##

The first endpoint that a broker must implement is the service catalog.  CC will initially (and periodically) fetch
this endpoint from all its brokers and make adjustments to the user-facing service catalog stored in CCDB. If the catalog
fails to initially load or validate, CC will not allow the operator to add the new broker, and should give
a meaningful error message.

When the catalog is fetched, if CC sees a service/plan that it does not recognize, it will add it to the catalog.
If it sees one that it does recognize from a previous fetch (using IDs to correlate), it
will update it in the catalog.  Currently it will not delete services/plans.

`GET /v2/catalog`

<table>
<thead>
<tr>
  <th>Request field</th>
  <th>Type</th>
  <th>Description</th>
</tr>
</thead>
<tbody>
<tr>
  <td>services*</td>
  <td>array-of-objects</td>
  <td>Schema of service objects defined below:</td>
</tr>
<tr>
  <td>&nbsp;&nbsp;&nbsp;&nbsp;id*</td>
  <td>string</td>
  <td>A unique identifier used to correlate this service in future requests to the catalog</td>
</tr>
<tr>
  <td>&nbsp;&nbsp;&nbsp;&nbsp;name*</td>
  <td>string</td>
  <td>The name of the service that will appear in the catalog</td>
</tr>
<tr>
  <td>&nbsp;&nbsp;&nbsp;&nbsp;description*</td>
  <td>string</td>
  <td>The description of the service that will appear in the catalog</td>
</tr>
<tr>
  <td>&nbsp;&nbsp;&nbsp;&nbsp;bindable</td>
  <td>boolean</td>
  <td>Whether the service can be bound to applications.  Defaults to true.</td>
</tr>
<tr>
  <td>&nbsp;&nbsp;&nbsp;&nbsp;requires</td>
  <td>array-of-strings</td>
  <td>A list of permissions that the user would have to give the service, if they provision it.
  Currently the only permission with any meaning is <code>syslog_drain</code>.</td>
</tr>
<tr>
  <td>&nbsp;&nbsp;&nbsp;&nbsp;plans*</td>
  <td>array-of-objects</td>
  <td>A list of plans for this service, schema defined below:</td>
</tr>
<tr>
  <td>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;id*</td>
  <td>string</td>
  <td>A unique identifier used to correlate this plan in future requests to the catalog</td>
</tr>
<tr>
  <td>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;name*</td>
  <td>string</td>
  <td>The name of the plan that will appear in the catalog</td>
</tr>
<tr>
  <td>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;description*</td>
  <td>string</td>
  <td>The description of the service that will appear in the catalog</td>
</tr>
</tbody>
</table>

Example response

    {
      "services": [{
        "id": "service-guid-here",
        "name": "A nice user-friendly name",
        "description": "Something users see next to the name",
        "plans": [{
          "id": "plan-guid-here",
          "name": "User friendly plan name",
          "description": "Something users see next to the name"
        }]
      }]
    }

## <a id="provision"></a>Provisioning ##

When the broker receives a provision request from CC, it should take whatever action is necessary to create a new service
resource for the developer.  What a provision means for different services can vary dramatically, but there are a few common
actions that work for many services.  Examples of what provision could mean, when applied to a MySQL service:

* The developer gets an empty dedicated `mysqld` process running on its own VM.
* The developer gets an empty dedicated `mysqld` process running in lightweight container
* The developer gets an empty dedicated database, on an existing shared running `mysqld`
* The developer gets a database with business schema already there
* The developer gets a copy of a full database (such as provisioning a QA database which is a copy of production)

For non-data services, provisioning could just mean getting an account on an existing system.

`PUT /v2/service_instances/:id`

<strong>In this API, the `:id` of a service instance is provided by the CC.</strong>
This ID will be used for future requests (bind and unprovision), so the broker must use it to correlate the resource it creates.

<table>
<thead>
<tr>
  <th>Request field</th>
  <th>Type</th>
  <th>Description</th>
</tr>
</thead>
<tbody>
<tr>
  <td>service_id*</td>
  <td>string</td>
  <td>The ID of the service (from the catalog endpoint) that the user would like provisioned.</td>
</tr>
<tr>
  <td>plan_id*</td>
  <td>string</td>
  <td>The ID of the plan within the above service (from the catalog endpoint) that the user would like provisioned.</td>
</tr>
</tbody>
</table>

Brokers are expected to respond with `200 OK` or `201 Created` with the response body from the table below.
They can also return `409 Conflict` if there is already a provisioned resource at this URL.
Since this endpoint cannot be used to update resources, brokers must return 409 if a non-identical request is made.
If they respond with any other code, CC will assume that the provision request failed and inform the user.

<table>
<thead>
<tr>
  <th>Response field</th>
  <th>Type</th>
  <th>Description</th>
</tr>
</thead>
<tbody>
<tr>
  <td>dashboard_url</td>
  <td>string</td>
  <td>The URL a web-based management UI for the service instance. This optional URL allows space developers to
  visit a friendly management console for this instance.  The URL should contain enough information to identify
  the resource being accessed (9189kdfsk0vfnku in the example below), as well as potentially credentials to access that resource (mongouser:pass below).
  CC will sign this URL so that the management console can trust that a user should be able to see the details of this instance.
  <em>Signing algorithm TBD.</em></td>
</tr>
</tbody>
</table>

Example response

    {
      "dashboard_url": "http://mongouser:pass@mongomgmthost/databases/9189kdfsk0vfnku"
    }


## <a id="bind"></a>Binding ##

When the broker receives a bind request from CC, it should take whatever action is necessary to generate a way for
an application to utilize the already-provisioned resource.  Not all services have to be bindable, some derive all their
value just from being provisioned.  In order for "unbind" to be the most meaningful, applications should be issued different
credentials when possible, allowing one bound application can have its access actually revoked without affecting the others.
Bind can mean different things depending on the service, here are some examples for Mysql:

* New random user credentials are generated for the existing database
* A single set of credentials are returned for every bind to the same database

`PUT /v2/service_bindings/:id`

<strong>In this API, the `:id` of a service binding is provided by the CC.</strong>
This ID will be used for future unbind requests, so the broker must use it to correlate the resource it creates.

<table>
<thead>
<tr>
  <th>Request field</th>
  <th>Type</th>
  <th>Description</th>
</tr>
</thead>
<tbody>
<tr>
  <td>service_instance_id*</td>
  <td>string</td>
  <td>The ID of the service instance to act on, as passed to the broker in previous previous request.</td>
</tr>
</tbody>
</table>

Brokers are expected to respond with `200 OK` or `201 Created` with the response body from the table below.
They can also return `409 Conflict` if there is already a binding at this URL.
Since this endpoint cannot be used to update resources, brokers must return 409 if a non-identical request is made.
If they respond with any other code, CC will assume that the binding request failed and inform the user.

<table>
<thead>
<tr>
  <th>Response field</th>
  <th>Type</th>
  <th>Description</th>
</tr>
</thead>
<tbody>
<tr>
  <td>credentials*</td>
  <td>object</td>
  <td>A free-form hash of credentials that the bound application can use to gain access to the service. We encourage
  the use of a <code>uri</code> field when possible (such as <code>mysql://user:password@host:port/dbname</code>),
  but it's often useful to provide
  individual credentials pieces such as hostname, port, username, and password.</td>
</tr>
<tr>
  <td>syslog_drain_url</td>
  <td>string</td>
  <td>A URL that CF should drain logs to for the bound application.  Requires the `syslog_drain` permission to work.</td>
</tr>
</tbody>
</table>

Example response

    {
      "credentials": {
        "uri": "mysql://mysqluser:pass@mysqlhost:3306/dbname",
        "username": "mysqluser",
        "password": "pass",
        "host": "mysqlhost",
        "port": 3306,
        "database": "dbname"
      }
    }


## <a id="unbind"></a>Unbinding ##

When a broker receives an unbind request from CC, it should delete any resources it created in bind.  Usually this means
that an application immediately cannot access the resource.

`DELETE /v2/service_bindings/:id`

The `:id` in the URL is the identifier returned in by the broker during a bind response.
The request body will be `{}`.

A response code of `200 OK` with a body of `{}` indicates that the resource was deleted.
A response code of `404 Not Found` with a body of `{}` indicates to CC that the resource was not there in the first place, and is likely gone.
*Any other* response code indicates to CC that the the binding could not be deleted,
the user will be informed (and the binding will remain in the database).

## <a id="unprovision"></a>Unprovisioning ##

When a broker receives an unprovision request from CC, it should delete any resources it created during the provision.
Usually this means that all resources are immediately reclaimed for future provisions.

`DELETE /v2/service_instances/:id`

The `:id` in the URL is the identifier returned in by the broker during a provision response.
The request body will be `{}`.

A response code of `200 OK` with a body of `{}` indicates that the resource was deleted.
A response code of `404 Not Found` with a body of `{}` indicates to CC that the resource was not there in the first place, and is likely gone.
*Any other* response code indicates to CC that the the service instance could not be deleted,
the user will be informed (and the instance will remain in the database).

## <a id="errors"></a>Broker Errors ##

When a broker fails to perform any requested operation, unless it fails with a well-defined HTTP response code listed above (like 404 on delete),
it should return a JSON-encoded error payload.
This payload allows the broker to expose a message to end-user or operator.  Any responses not matching this error format will
result in the user seeing a generic internal error message.

* Use HTTP status code `401 Unauthorized` when authentication is missing
* Use HTTP status code `403 Forbidden` when authentication is incorrect
* Use HTTP status code `422 Unprocessable Entity` when the input is invalid
* Use HTTP status code `502 Bad Gateway` when there was an issue with a downstream API
* Use HTTP status code `500 Internal Server Error` for a generic issue

<strong>Error response format:</strong>

<table>
<thead>
<tr>
  <th>Response field</th>
  <th>Type</th>
  <th>Description</th>
</tr>
</thead>
<tbody>
<tr>
  <td>message*</td>
  <td>string</td>
  <td>An error message explaining why the request failed.  This message will likely be forwarded to the person initiating the request.</td>
</tr>
</tbody>
</table>

## <a id="orphans"></a>Orphans ##

Because CCDB and a Broker supposedly store identical copies of which instances and bindings exist,
there is a possibility that the two lists will be inconsistent.  For example, if a Broker times out during a delete request,
CC will be unsure of whether that resource still exists on the broker or not.
CC will implement the following orphan prevention techniques:

* If a Broker fails to provision/bind, the CC will immediately issue a unprovision/unbind request to ensure that any provisioned
resources have been cleaned up.
* If a Broker fails to unbind/unprovision, the CC will periodically retry that DELETE request until it succeeds (or generates a 404).
Eventually it will give up, but this technique helps clean up resources that are left around when a broker fails to delete.
