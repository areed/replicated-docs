# Referencing license fields

This topic describes how to create references to the license fields that you defined
for a customer in the Replicated vendor portal.

For information about the built-in license fields, see [About built-in license fields](licenses-using-builtin-fields).
For information about creating custom license fields, see [Creating custom license fields](licenses-adding-custom-fields).

## Overview of referencing license fields

After you define built-in or custom license fields for a customer in the vendor
portal, you create references to these license fields that your application can
query.

This ensures that the customer's application instance can enforce the entitlements
that you defined in their license file.

To reference a license field, you can:

* **Write license fields to Kubernetes manifests**: Create references to license
fields in Kubernetes manifest files. This allows you to enforce entitlements when
your customer installs or updates your application. See [Write license fields to Kubernetes manifests](#write-license-fields-to-kubernetes-manifests) below.
* **Query the admin console API**: Query the license field from the application
using the admin console API. This allows you to enforce entitlements during
application runtime. See [Query license fields from the API](#query-license-fields-from-the-api) below.

## Write license fields to Kubernetes manifests

This section describes referencing license fields in Kubernetes manifest files.

### About writing license fields to Kubernetes manifests

To enforce entitlements when your customer installs or updates your application,
you can reference built-in and custom license fields in a Kubernetes manifest.

The Replicated app manager uses the `LicenseFieldValue` template function to read
license fields when a customer installs or updates your application. For more
information, see [LicenseFieldValue](template-functions-license-context#licensefieldvalue).

### Example: Reference a custom license field in a preflight check

For example, a license might limit how many nodes are permitted in a customer's
cluster.
You could define this limit by creating a `node_count` custom license field:

| Name | Key | Type | Description |
|------|-----|------|-------------|
| Node Count | node_count | Integer | The maximum number of nodes permitted |

To enforce the node count when a customer installs or updates your application,
you can create a preflight check that references the `node_count` field:

```
apiVersion: troubleshoot.replicated.com/v1beta1
kind: Preflight
metadata:
  name: example-preflight-checks
spec:
  analyzers:
    - nodeResources:
        checkName: Node Count Check
        outcomes:
          - fail:
              when: 'count() > {{repl LicenseFieldValue "node_count"}}'
              message: The cluster has more nodes than the {{repl LicenseFieldValue "node_count"}} you are licensed for.
          - pass:
              message: The number of nodes matches your license ({{repl LicenseFieldValue "node_count"}})
```

In the example above, the manifest defines a preflight check that uses the `nodeResources`
analyzer and the value of the `node_count` license field to determine if the customer
has exceeded the maximum number of nodes permitted by their license.

This preflight check runs during application installation and update, and prevents
the installation or update from continuing if the maximum number of nodes defined
by the `node_count` license field is exceeded.

//To Do: create new topic from the linked community post//
For more information, see [How Can I Use License Custom Fields Value in a Pre-Flight Check?](https://help.replicated.com/community/t/how-can-i-use-license-custom-fields-value-in-a-pre-flight-check/624) in the Replicated Community.

## Query license fields from the API

The Replicated admin console runs on the customer's cluster and provides entitlement
information during application runtime. You can query the admin console API to
enforce entitlements at runtime.

To reference license fields at runtime, send an HTTP request to the admin console
API `/license/v1/license` endpoint at the following location:

```
http://kotsadm:3000/license/v1/license
```

The admin console API returns a response in YAML format. For example:

```javascript
{"license_id":"WicPRaoCv1pJ57ZMf-iYRxTj25eZalw3",
"installation_id":"a4r1s31mj48qw03b5vwbxvm5x0fqtdl6",
"assignee":"FirstCustomer",
"release_channel":"Unstable",
"license_type":"trial",
"expiration_time":"2026-01-23T00:00:00Z",
"fields":[
  {"field":"Customer ID","title":"Customer ID (Internal)","type":"Integer","value":121,"hide_from_customer":true},
  {"field":"Modules","title":"Enabled Modules","type":"String","value":"Analytics, Integration"}]}
```

To return a license field value, parse the response using the name of the license
field.

For example, the following Javascript parses the API response for the value of a
`seat_count` custom field:

```javascript
import * as rp from "request-promise";

rp({
  uri: "http://kotsadm:3000/license/v1/license",
  json: true
}).then(license => {
  const seatCount = license.fields.find((field) => {
    return field.field === "seat_count";
  });
  console.log(seatCount.value);
}).catch(err => {
  // Handle error response from `kotsadm`
});
```