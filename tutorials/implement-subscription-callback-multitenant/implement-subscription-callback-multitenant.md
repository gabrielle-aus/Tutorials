---
title: Implement Subscription Callbacks for a Multitenant Application
description: Implement the Subscribe/Unsubscribe callbacks of the SAP SaaS Provisioning Service (saas-registry) and use the information provided in the subscription/unsubscription event for your Multitenant Application in the Kyma Runtime.
auto_validation: true
time: 25
tags: [ tutorial>intermediate, software-product>sap-business-technology-platform]
primary_tag: software-product>sap-btp\, kyma-runtime
---

## Prerequisites
- You have finished the tutorial [Secure a Multitenant Application with the Authorization and Trust Management Service (XSUAA)](secure-multitenant-app-xsuaa-kyma)

## Details
### You will learn
- How to implement the Subscribe/Unsubscribe callbacks for the SAP SaaS Provisioning Service
- How to automatically create `APIRule` for tenant-specific URL in the Kyma runtime


---

[ACCORDION-BEGIN [Step 1: ](Introduction to Subscription Callbacks)]


To perform internal tenant onboarding activities, you must implement the `subscription` and `unsubscription` callbacks of the SAP Software-as-a-Service Provisioning service (saas-registry) and use the information provided in the subscription event. You can also implement the `getDependencies` callback to obtain the dependencies of any SAP reuse services by your application. For more details, please read SAP Help Portal: [Develop the Multitenant Application](https://help.sap.com/products/BTP/65de2977205c403bbc107264b8eccf4b/ff540477f5404e3da2a8ce23dcee602a.html?locale=en-US&q=TENANT_HOST_PATTERN).

When a consumer tenant creates or revokes a subscription to your multitenant application via the cockpit, the SAP SaaS Provisioning service calls the multitenant application with subscription callbacks:

**PUT** – `callback/v1.0/tenants/*`

To inform the application that a new consumer tenant wants to subscribe to the application, implement the callback service with the PUT method. The callback must return a 200 response code and a string in the body, which is the access URL of the tenant to the application subscription (the URL contains the subdomain). This URL is generated by the multitenant application based on the approuter configuration (see [Configure the approuter Application](https://help.sap.com/products/BTP/65de2977205c403bbc107264b8eccf4b/5af9067322214e8dbf354daae44cef08.html??locale=en-US))

For example: `https://customer-cb741ym5-approuter.e6803e4.kyma.shoot.live.k8s-hana.ondemand.com`

> The URL must be in the format: `https://<subaccount-subdomain>-<approuter-application-host>.<domain>`

**DELETE** – `callback/v1.0/tenants/*`

To inform the application that a tenant has revoked its subscription to the multitenant application and is no longer allowed to use it, implement the same callback with the DELETE method.

Payload of subscription PUT and DELETE methods:

```JSON
{
    "subscriptionAppId": "<value>",                     
    # The application ID of the main subscribed application.
    # Generated by Authorization and Trust Management service(XSUAA)-based on the xsappname.
    "subscriptionAppName": "<value>"                    
    # The application name of the main subscribed application.
    "subscribedTenantId": "<value>"                     
    # ID of the subscription tenant
    "subscribedSubdomain": "<value>"                    
    # The subdomain of the subscription tenant (hostname for the identityzone).
    "globalAccountGUID": "<value>"                      
    # ID of the global account
    "subscribedLicenseType": "<value>"                  
    # The license type of the subscription tenant
}
```

**GET** – `callback/v1.0/dependencies`

```JSON
[
   {
      "xsappname" : "<value>"   
      # The xsappname of the reuse service which the application consumes.
      # Can be found in the environment variables of the application:
      # VCAP_SERVICES.<service>.credentials.xsappname
   }
]
```

> The callback must return a 200 response code and a JSON file with the dependent services **`appName`** and **`appId`**, or just the **`xsappname`**.
>
> The JSON response of the callback must be encoded as either UTF8, UTF16, or UTF32, otherwise an error is returned.



[DONE]
[ACCORDION-END]

[ACCORDION-BEGIN [Step 2: ](Implement Subscription Callbacks)]

A tenant-specific URL should be provided to customers in the onboarding process, at the same time, the URL should be exposed to the Internet as well. Otherwise, customers still cannot access the tenant. URL is exposed in the Kyma runtime through `APIRule`, which needs to be created dynamically through the onboarding/offboarding process using [Kubernetes client for NodeJS](https://github.com/kubernetes-client/javascript).

In the `kyma-multitenant-node/routes/index.js` file, implement the `subscription` and `unsubscription` callbacks. Replace the placeholder with your cluster domain.

```JavaScript
//******************************** API Callbacks for multitenancy ********************************

/**
 * Request Method Type - PUT
 * When a consumer subscribes to this application, SaaS Provisioning invokes this API.
 * We return the SaaS application url for the subscribing tenant.
 * This URL is unique per tenant and each tenant can access the application only through it's URL.
 */
router.put('/callback/v1.0/tenants/*', async function(req, res) {
    //1. create tenant unique URL
    var consumerSubdomain = req.body.subscribedSubdomain;
    var tenantAppURL = "https:\/\/" + consumerSubdomain + "-approuter." + "<cluster-domain>";

    //2. create apirules with subdomain,
    const kc = new k8s.KubeConfig();
    kc.loadFromCluster();
    const k8sApi = kc.makeApiClient(k8s.CustomObjectsApi);
    const apiRuleTempl = createApiRule.createApiRule(
        EF_SERVICE_NAME,
        EF_SERVICE_PORT,
        consumerSubdomain + "-approuter",
        kyma_cluster);

    try {
        const result = await k8sApi.getNamespacedCustomObject(KYMA_APIRULE_GROUP,
            KYMA_APIRULE_VERSION,
            EF_APIRULE_DEFAULT_NAMESPACE,
            KYMA_APIRULE_PLURAL,
            apiRuleTempl.metadata.name);
        //console.log(result.response);
        if (result.response.statusCode == 200) {
            console.log(apiRuleTempl.metadata.name + ' already exists.');
            res.status(200).send(tenantAppURL);
        }
    } catch (err) {
        //create apirule if non-exist
        console.warn(apiRuleTempl.metadata.name + ' does not exist, creating one...');
        try {
            const createResult = await k8sApi.createNamespacedCustomObject(KYMA_APIRULE_GROUP,
                KYMA_APIRULE_VERSION,
                EF_APIRULE_DEFAULT_NAMESPACE,
                KYMA_APIRULE_PLURAL,
                apiRuleTempl);
            console.log(createResult.response);

            if (createResult.response.statusCode == 201) {
                console.log("API Rule created!");
                res.status(200).send(tenantAppURL);
            }
        } catch (err) {
            console.log(err);
            console.error("Fail to create APIRule");
            res.status(500).send("create APIRule error");
        }
    }
    console.log("exiting onboarding...");
    res.status(200).send(tenantAppURL)
});

/**
 * Request Method Type - DELETE
 * When a consumer unsubscribes this application, SaaS Provisioning invokes this API.
 * We delete the consumer entry in the SaaS Provisioning service.
 */
router.delete('/callback/v1.0/tenants/*', async function(req, res) {
    console.log(req.body);
    var consumerSubdomain = req.body.subscribedSubdomain;

    //delete apirule with subdomain
    const kc = new k8s.KubeConfig();
    kc.loadFromCluster();

    const k8sApi = kc.makeApiClient(k8s.CustomObjectsApi);

    const apiRuleTempl = createApiRule.createApiRule(
        EF_SERVICE_NAME,
        EF_SERVICE_PORT,
        consumerSubdomain + "-approuter",
        kyma_cluster);

    try {
        const result = await k8sApi.deleteNamespacedCustomObject(
            KYMA_APIRULE_GROUP,
            KYMA_APIRULE_VERSION,
            EF_APIRULE_DEFAULT_NAMESPACE,
            KYMA_APIRULE_PLURAL,
            apiRuleTempl.metadata.name);
        if (result.response.statusCode == 200) {
            console.log("API Rule deleted!");
        }
    } catch (err) {
        console.error(err);
        console.error("API Rule deletion error");
    }

    res.status(200).send("deleted");
});
//************************************************************************************************
```


[DONE]
[ACCORDION-END]


[ACCORDION-BEGIN [Step 3: ](Define Logic for APIRule Creation)]

**1.** Add constant values and variables in the `kyma-multitenant-node/routes/index.js`, replace <namespace> with your own namespace name:

```JavaScript[4-13]
var express = require('express');
var router = express.Router();

const EF_SERVICE_NAME = 'kyma-multitenant-approuter-multitenancy';
const EF_SERVICE_PORT = 8080;
const EF_APIRULE_DEFAULT_NAMESPACE = '<namespace>';
const KYMA_APIRULE_GROUP = 'gateway.kyma-project.io';
const KYMA_APIRULE_VERSION = 'v1alpha1';
const KYMA_APIRULE_PLURAL = 'apirules';

const k8s = require('@kubernetes/client-node');
const createApiRule = require('./createApiRule');
var kyma_cluster = process.env.CLUSTER_DOMAIN || "UNKNOWN";
```


**2.** Create a new file named `createApiRule.js` in the directory `kyma-multitenant-node/routes/` to provide `APIRule` object:

```JavaScript
module.exports = {
    createApiRule: createApiRule
}

function createApiRule(svcName, svcPort, host, clusterName) {

    let forwardUrl = host + '.' + clusterName;
    const supportedMethodsList = [
        'GET',
        'POST',
        'PUT',
        'PATCH',
        'DELETE',
        'HEAD',
    ];
    const access_strategy = {
        path: '/.*',
        methods: supportedMethodsList,
        // mutators: [{
        //     handler: 'header',
        //     config: {
        //         headers: {
        //             "x-forwarded-host": forwardUrl,
        //         }
        //     },
        // }],
        accessStrategies: [{
            handler: 'allow'
        }],
    };

    const apiRuleTemplate = {
        apiVersion: 'gateway.kyma-project.io/v1alpha1',
        kind: 'APIRule',
        metadata: {
            name: host + '-apirule',
        },
        spec: {
            gateway: 'kyma-gateway.kyma-system.svc.cluster.local',
            service: {
                host: host,
                name: svcName,
                port: svcPort,
            },
            rules: [access_strategy],
        },
    };
    return apiRuleTemplate;
}
```

**3.** Add dependency `"@kubernetes/client-node"` in the `kyma-multitenant-node/package.js` file, for example:

```JSON[2]
    "dependencies": {
        "@kubernetes/client-node": "~0.15.0",
        ...
    }
}
```

[DONE]
[ACCORDION-END]

[ACCORDION-BEGIN [Step 4: ](Grant Role to Pod)]

To automatically create `APIRule` from a pod, proper `RoleBinding` should be granted. Add a `RoleBinding` in the `k8s-deployment-backend.yaml` with the following content, replace <namespace> with your own namespace name:

```YAML
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: broker-rolebinding
subjects:
  - kind: ServiceAccount
    name: default
    namespace: <namespace>
roleRef:
  kind: ClusterRole
  name: kyma-namespace-admin
  apiGroup: rbac.authorization.k8s.io  
```


[VALIDATE_1]
[ACCORDION-END]


---
