# Istio workshop at consol (Part 2)

The workshop is based on this RedHat tutorial for Istio. [Java (Spring Boot, Vert.x and Micro profile) + Istio on Kubernetes/OpenShift](https://redhat-developer-demos.github.io/istio-tutorial/istio-tutorial/1.1.x/index.html). A couples of days ago this Istio tutorial has been updated to version 1.3. I didn't have the time update this page, so we will use the version 1.1.9 of the tutorial for the time being.

We will use the minishift v1.34 to run Istio on OpenShift 3.11. Please follow the instructions how to install Istio on Minishift based on the example above. Furthermore deploy the application like described the section [2. Deploy Microservices](https://redhat-developer-demos.github.io/istio-tutorial/istio-tutorial/1.1.x/2deploy-microservices.html). You can use the images that I uploaded to <https://quay.io/repository/omeyer/istio-tutorial> instead of the building the image locally. The deployments yaml and yaml for the exercises can be found here at my GitHub repo <https://github.com/olaf-meyer/openshift-talks>. Please clone it, because we will use it in the workshop.

Use this command to setup the demo application

``` bash
oc apply -f <(istioctl kube-inject -f ../openshift-talks/istio_example_application.yaml) -n tutorial
```

**Hint:** You might need to change the path to the yaml files to your environment.

We will skip the section with the routing of requests, because this was part of the first Istio workshop.

In the workshop we are not going to compile change in the source code instead we are going to use example images that I uploaded on quay.io.

## Service Resiliency

In this section you will use dynamic routing based on certain error cases.  If i.e an timeout happens for a certain pod than istio will retry the request with alternatives (if possible). In case not alternative is present the error 503 is returned.

### Retry

Lets start with this exercise:
[https://redhat-developer-demos.github.io/istio-tutorial/istio-tutorial/1.1.x/5circuit-breaker.html](https://redhat-developer-demos.github.io/istio-tutorial/istio-tutorial/1.1.x/5circuit-breaker.html)

``` bash
oc exec -it -n tutorial $(oc get pods -n tutorial|grep recommendation-v2|awk '{ print $1 }'|head -1) -c recommendation /bin/bash
curl localhost:8080/misbehave
exit
```

The exercise explains that pod which return 503s is not called. However it is not explaining why this is happening. To get to it we need to get the envoy configuration from the preference pod with the following commands:

``` bash
oc port-forward preference-v1-b56964f5c-fhczd -n istio-system 15000
```

and in a different terminal

``` bash
curl http://localhost:15000/config_dump > ../openshift-talks/envoy_config_preference.json
```

If you search in the envoy_config_preference.json file for the following string `outbound|8080||preference.tutorial.svc.cluster.local` you should find a similar configuration:

``` json
        "routes": [
         {
          "match": {
           "prefix": "/"
          },
          "route": {
           "cluster": "outbound|8080||preference.tutorial.svc.cluster.local",
           "timeout": "0s",
           "retry_policy": {
            "retry_on": "connect-failure,refused-stream,unavailable,cancelled,resource-exhausted,retriable-status-codes",
            "num_retries": 2,
            "retry_host_predicate": [
             {
              "name": "envoy.retry_host_predicates.previous_hosts"
             }
            ],
            "host_selection_retry_max_attempts": "3",
            "retriable_status_codes": [
             503
            ]
           },
           "max_grpc_timeout": "0s"
          },
          "decorator": {
           "operation": "preference.tutorial.svc.cluster.local:8080/*"
          },
```

You can see that by default the envoy proxy is configured to do a retry from the status code 503. Details on the configuration can be found here [route.RetryPolicy](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/route/route.proto#route-retrypolicy)

### Timeout

In the next exercise we are going to simulate slow responses of a service/pod.  Please use this patch command for this instead of the provided command.

``` bash
oc patch deployment recommendation-v2 --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value": "quay.io/omeyer/istio-tutorial:recommendationv3" }]'
```

Run the rest of the exercise like described here [Timeout](https://redhat-developer-demos.github.io/istio-tutorial/istio-tutorial/1.1.x/5circuit-breaker.html#timeout)

Definition of the timeout setting can be found here <https://istio.io/docs/reference/config/networking/v1alpha3/virtual-service/#HTTPRoute> and here <https://istio.io/docs/concepts/traffic-management/#timeouts-and-retries>

Reset the behavior of recommendation-v2

``` bash
oc patch deployment recommendation-v2 --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value": "quay.io/omeyer/istio-tutorial:recommendationv2" }]'
```

### Fail fast with circuit breaker (connection pool)

In the first part we will use the attribute `http1MaxPendingRequests` and `maxRequestsPerConnection` in the DestinationRule to limit the number of concurrent requests.

Please delete all destination rules and virtual services with the script ./clean.sh of the tutorial.

In order that we can see an effect of the connection pool, the requests shouldn't reply immediately but should use a bit of processing time. We can do this by using the image with the time out:

``` bash
oc patch deployment recommendation-v2 --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value": "quay.io/omeyer/istio-tutorial:recommendationv3" }]'
```

Now lets create the virtual service and der destination rule with these commands:

``` bash
oc create -f istio-workshop-part2/circuitbreaker/connection\ pool/destinationrule.yaml
oc create -f istio-workshop-part2/circuitbreaker/connection\ pool/virtualservice.yaml
```

Lets put our application a bit under load (5 requests concurrent) with this command:

``` bash
siege -r 10 -c 5 -v http://istio-ingressgateway-istio-system.$(minishift ip).nip.io/customer

```

The result should be that all requests are return the status 200

If we increase the load we should see that for a lot of cases the status 503 is return:

``` bash
siege -r 10 -c 20 -v http://istio-ingressgateway-istio-system.$(minishift ip).nip.io/customer

```

**Question** Do you think that the connection pool defined per calling pod or per called pod or a globally?

### Handle fast failed errors with outlier detection

The connection pool protects backend application from to much traffic. However the drawback is that the client application gets a lot of error messages which might cause problems as well. Lets check if the outlier setting can help us.

First you need to delete the previous virtual service and destination route and create them new by executing these commands:

``` bash
oc patch deployment recommendation-v2 --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value": "quay.io/omeyer/istio-tutorial:recommendationv2" }]'
oc create -f istio-workshop-part2/circuitbreaker/outlier/destinationrule.yaml
oc create -f istio-workshop-part2/circuitbreaker/outlier/virtualservice.yaml
```

In the next step scale up the number of pod to 4 with this command:

``` bash
oc scale deployment --replicas=4 recommendation-v2
```

After a moment you should have 4 pods up and  running. Next lets test if they all return a value by running this command (location of script my vary):

``` bash
istio-tutorial/scripts/run.sh istio-ingressgateway-istio-system.$(minishift ip).nip.io/customer
```

The output should contain responses from all pods. Now lets convince one pod to misbehave by executing this command:

``` bash
oc exec recommendation-v2-6bd4d5775c-7plwf -- curl -v localhost:8080/misbehave
```

If you now execute the command:

``` bash
istio-tutorial/scripts/run.sh istio-ingressgateway-istio-system.$(minishift ip).nip.io/customer
```

You should not the a response of the misbehave pod. If you however look in the istio-proxy logs of the misbehaved pod you should see **one** log statement for a response with the status 503.
Here you can find details how to setup the outlier detection: [Outlier detection](https://istio.io/docs/reference/config/networking/v1alpha3/destination-rule/#OutlierDetection)

Hint:
With the version 1.1.9 I was not able to setup an outlier dection that is using two services in the destination rule. The misbehaving pod is called more often than configured.

You can play around how to mix outlier detection and the connection pool.

Lets clean up the mess and scale down the application.

## Chaos Testing

In the previous section we had a look how to configure your application to use the proper routing and connection pool (better term would be connection limit). One problem remains however: How can you be sure the the configuration is correct? By testing is of course ;-)

### Injecting errors

The first part would be to inject random errors. See exercise here: [Fault Inject in tutorial](
https://redhat-developer-demos.github.io/istio-tutorial/istio-tutorial/1.0.x/6fault-injection.html#503error). However because of the above mentioned reasons with retries we are going to use a modified version of the exercise that returns a 404 in case a fault. So clean up the settings from before and execute the following commands:

``` bash
oc create -f istio-workshop-part2/faultinjection/error/destination-rule-recommendation.yml
oc create -f istio-workshop-part2/faultinjection/error/virtual-service-recommendation-404.yml
```

The documentation for the configuration can be found here: <https://istio.io/docs/reference/config/networking/v1alpha3/virtual-service/#HTTPFaultInjection-Abort>

To verify that an error occurred, please check the log files of the preference pod or have a look at Jaeger.

**Test** Is it possible to return two different error code? Is it possible to combine the fault setting with other settings?

### Injecting delays

The next test is much easier to verify that it was successful or faulty (??), because the response from the recommendation service will be delayed.

Like always clean up the previous exercise and execute the following commands:

``` bash
oc create -f istio-workshop-part2/faultinjection/timeout/destination-rule-recommendation.yml
oc create -f istio-workshop-part2/faultinjection/timeout/virtual-service-recommendation-timeout.yml
```

To test is the fault injection works we can use this siege command:

``` bash
siege -r 10 -c 2 -v http://istio-ingressgateway-istio-system.$(minishift ip).nip.io/customer
```

The result should look like this. As you can see that every request took about 5 seconds, so our fault injection works:

``` bash
** SIEGE 4.0.4
** Preparing 2 concurrent users for battle.
The server is now under siege...
HTTP/1.1 200     5.10 secs:      71 bytes ==> GET  /customer
HTTP/1.1 200     5.12 secs:      71 bytes ==> GET  /customer
HTTP/1.1 200     5.06 secs:      71 bytes ==> GET  /customer
HTTP/1.1 200     5.06 secs:      71 bytes ==> GET  /customer
HTTP/1.1 200     5.04 secs:      71 bytes ==> GET  /customer
HTTP/1.1 200     5.08 secs:      71 bytes ==> GET  /customer
HTTP/1.1 200     5.03 secs:      71 bytes ==> GET  /customer
HTTP/1.1 200     5.06 secs:      71 bytes ==> GET  /customer
HTTP/1.1 200     5.54 secs:      72 bytes ==> GET  /customer
HTTP/1.1 200     5.47 secs:      72 bytes ==> GET  /customer
HTTP/1.1 200     5.14 secs:      72 bytes ==> GET  /customer
HTTP/1.1 200     5.15 secs:      72 bytes ==> GET  /customer
HTTP/1.1 200     5.15 secs:      72 bytes ==> GET  /customer
HTTP/1.1 200     5.18 secs:      72 bytes ==> GET  /customer
HTTP/1.1 200     5.07 secs:      72 bytes ==> GET  /customer
HTTP/1.1 200     5.10 secs:      72 bytes ==> GET  /customer
HTTP/1.1 200     5.05 secs:      72 bytes ==> GET  /customer
HTTP/1.1 200     5.16 secs:      72 bytes ==> GET  /customer
HTTP/1.1 200     5.11 secs:      72 bytes ==> GET  /customer
HTTP/1.1 200     5.06 secs:      72 bytes ==> GET  /customer

Transactions:                     20 hits
Availability:                     100.00 %
Elapsed time:                     51.40 secs
Data transferred:                 0.00 MB
Response time:                    5.14 secs
Transaction rate:                 0.39 trans/sec
Throughput:                       0.00 MB/sec
Concurrency:                      2.00
Successful transactions:          20
Failed transactions:              0
Longest transaction:              5.54
Shortest transaction:             5.03
```

The configuration of the delay can be found here: <https://istio.io/docs/reference/config/networking/v1alpha3/virtual-service/#HTTPFaultInjection-Delay>

## Policy (& telemetry)

Policies und telemetry are handled by the Mixer in Istio. Details can be found here: [Mixer configuration model](https://istio.io/docs/reference/config/policy-and-telemetry/mixer-overview/)

![Mixer topology](https://istio.io/docs/reference/config/policy-and-telemetry/mixer-overview/topology-without-cache.svg)

This is a quote from Istio documentation regarding the communication between the mixer and the Envoy sidecar container
> The Envoy sidecar logically calls Mixer before each request to perform precondition checks, and after each request to report telemetry. The sidecar has local caching such that a large percentage of precondition checks can be performed from cache. Additionally, the sidecar buffers outgoing telemetry such that it only calls Mixer infrequently.

The functionality of the mixer is extended by adapters. Which can be configured and extend at runtime.

Lets have a look at the components and resources that are used for telemetry and policies.

Adapters are used to abstracts backend functionality from Istio. List of supported adapaters can be found here: [Supported adapter](https://istio.io/docs/reference/config/policy-and-telemetry/adapters/).

An attribute is a small bit of data that describes a single property of a specific service request or the environment for the request. For example, an attribute can specify the size of a specific request, the response code for an operation, the IP address where a request came from, etc.

Attribute expression allows the modification of attributes. For a list of possible attributes see here: [Attribute expression](https://istio.io/docs/reference/config/policy-and-telemetry/expression-language/)

Policies are telemetry are configured by three resources:

1. Handler determines which adapaters are used and how.
2. Instances describe how to map request attributes into adapter input.
3. Rules describe describes under which condition a particular adapter is called and which instances it is using. So binding the adapter with the instance.

A template defines a schema how to map attributes from the request to adapter input. An adapter can support multiple templates. From what I understand a template can be use as in a Instances.

To clarify the terms lets look at some examples.

### Black listing

To do a blacklisting we need to define an handler (adapter), an instance and a rule.

For our application it looks like this:

``` yaml
apiVersion: "config.istio.io/v1alpha2"
kind: denier
metadata:
  name: denycustomerhandler
spec:
  status:
    code: 7
    message: Not allowed
---
apiVersion: "config.istio.io/v1alpha2"
kind: checknothing
metadata:
  name: denycustomerrequests
spec:
---
apiVersion: "config.istio.io/v1alpha2"
kind: rule
metadata:
  name: denycustomer
spec:
  match: destination.labels["app"] == "preference" && source.labels["app"]=="customer"
  actions:
  - handler: denycustomerhandler.denier
    instances: [ denycustomerrequests.checknothing ]

```

What this policy is doing, is to call the handler denier for every request from the source with the label app equals customer to the destination with the label app equals preference with the template (or instance) checknothing.

The resource `denycustomerhandler` is an adapter. You can find a description of it here: [Denier](https://istio.io/docs/reference/config/policy-and-telemetry/adapters/denier/). The template/instance is `denycustomerrequests`. Details of this template can be found here [checknothing](https://istio.io/docs/reference/config/policy-and-telemetry/templates/checknothing/).

BTW a more readable form of the yaml would be this, which is introduced in a later version of Istio.

``` yaml
apiVersion: "config.istio.io/v1alpha2"
kind: handler
metadata:
  name: denycustomerhandler
spec:
  compiledAdapter: denier
  status:
    code: 7
    message: Not allowed
---
apiVersion: "config.istio.io/v1alpha2"
kind: instance
metadata:
  name: denycustomerrequests
spec:
  compiledTemplate: checknothing
---
apiVersion: "config.istio.io/v1alpha2"
kind: rule
metadata:
  name: denycustomer
spec:
  match: destination.labels["app"] == "preference" && source.labels["app"]=="customer"
  actions:
  - handler: denycustomerhandler
    instances: [ denycustomerrequests ]

```

Any how lets create the black listing with the command:

```bash
oc create -f istio-workshop-part2/policy/blacklisting/blacklisting1.yaml
```

To test it run this command:

```bash
istio-tutorial/scripts/run.sh istio-ingressgateway-istio-system.192.168.99.107.nip.io/customer
```

The result should look like this:

```bash
customer => Error: 403 - PERMISSION_DENIED:denycustomerhandler.denier.tutorial:Not allowed
customer => Error: 403 - PERMISSION_DENIED:denycustomerhandler.denier.tutorial:Not allowed
customer => Error: 403 - PERMISSION_DENIED:denycustomerhandler.denier.tutorial:Not allowed
customer => Error: 403 - PERMISSION_DENIED:denycustomerhandler.denier.tutorial:Not allowed
customer => Error: 403 - PERMISSION_DENIED:denycustomerhandler.denier.tutorial:Not allowed
```

### White listing

The whitelisting is using a listchecker as an adapter. A listentry as template ro instance.

``` yaml
apiVersion: "config.istio.io/v1alpha2"
kind: listchecker
metadata:
  name: preferencewhitelist
spec:
  overrides: ["customer"]
  blacklist: false
---
apiVersion: "config.istio.io/v1alpha2"
kind: listentry
metadata:
  name: preferencesource
spec:
  value: source.labels["app"]
---
apiVersion: "config.istio.io/v1alpha2"
kind: rule
metadata:
  name: checkfromcustomer
spec:
  match: destination.labels["app"] == "preference"
  actions:
  - handler: preferencewhitelist.listchecker
    instances:
    - preferencesource.listentry
```

Lets test the white listing:

```bash
oc create -f istio-workshop-part2/policy/whitelisting/whitelisting.yaml
```

To test it run this command:

```bash
istio-tutorial/scripts/run.sh istio-ingressgateway-istio-system.192.168.99.107.nip.io/customer
```

The result should look like this:

```bash
customer => preference => recommendation v2 from '6bd4d5775c-8ltf8': 60
customer => preference => recommendation v1 from '999958457-8rdg8': 40
customer => preference => recommendation v2 from '6bd4d5775c-8ltf8': 61
customer => preference => recommendation v1 from '999958457-8rdg8': 41
```

If you try to call the preference service from with the recommendation pod you get an error:

```bash
 $ oc rsh recommendation-v1-999958457-8rdg8
Defaulting container name to recommendation.
Use 'oc describe pod/recommendation-v1-999958457-8rdg8 -n tutorial' to see all of the containers in this pod.
sh-4.2$ curl customer:8080
customer => preference => recommendation v2 from '6bd4d5775c-8ltf8': 62
sh-4.2$ curl preference:8080
PERMISSION_DENIED:preferencewhitelist.listchecker.tutorial:recommendation is not whitelisted
sh-4.2$
```

Lets clean up the created resources with the command:

```bash
oc delete -f istio-workshop-part2/policy/whitelisting/whitelisting.yaml
```

### Rate limit

The rate limit is a bit more complex. For this we need to define a policy with instance, handler and rule in the mixer and QuotaSpec and QuotaSpecBinding on the client side in the side car container (envoy proxy).

1. The resource quota is the instance of the policy and it defines the dimensions on which the quota should be checked. The documentation for the resource quota can be found here: [Quota doc](https://istio.io/docs/reference/config/policy-and-telemetry/templates/quota/)
1. With memquota we defined the default values of 5000 requests every second and for the recommendation service called by the preference service maximum number of 1 request every second. Documentation for the memquota can be found here [Memory quota](https://istio.io/docs/reference/config/policy-and-telemetry/adapters/memquota/)
1. With the quota resource we bind the memoryquota to the quota.
1. The resource QuotaSpec defines the quotas with name and how much a request to be charged. Documentation can be found here [QuotaSpec](https://istio.io/docs/reference/config/policy-and-telemetry/istio.mixer.v1.config.client/#QuotaSpec)
1. The resource QuotaSpecBinding defines the binding between QuotaSpecs and one or more IstioService. The documentation ca be found here [QuotaSpecBinding](https://istio.io/docs/reference/config/policy-and-telemetry/istio.mixer.v1.config.client/#QuotaSpecBinding)

Lets create the ratelimit and see if it works. The rate limit can be created with the command:

```bash
oc create -f istio-workshop-part2/policy/ratelimit/ratelimit.yaml
```

If we now run a the commands `istio-tutorial/scripts/run.sh istio-ingressgateway-istio-system.192.168.99.107.nip.io/customer` or `siege -r 10 -c 10 -v http://istio-ingressgateway-istio-system.$(minishift ip).nip.io/customer` you will not get an error message. This is because of the default retries of the envoy proxy in the preference pod. Our rate limit works only for the recommendation v2 and not for recommendation v1. In the log if the istio-proxy container of recommendation-v2-... pod, you should find e statement like this:

``` bash
[2019-09-16T11:02:44.946Z] "GET / HTTP/1.1" 429 UAEX ""RESOURCE_EXHAUSTED:Quota is exhausted for: requestcount"" 0 55 0 - "-" "Java/1.8.0_191" "ef79ef60-ebdc-913e-9f5e-8fd6156a0ef5" "recommendation:8080" "-" - - 172.17.0.27:8080 172.17.0.28:50832 -
```

**Hint:** It might take some time till the ratelimits are applied.
**Exercise** What do you need to set in order to get an error immediately?

The last part of the exercise would be to delete the ratelimit with the command

```bash
oc delete -f istio-workshop-part2/policy/ratelimit/ratelimit.yaml
```

## Security

A detailed description about the enhanced security that Istio provides can be found here: [Istio documentation security](https://istio.io/docs/concepts/security/). In the workshop we are going to focus on for aspects of it.

- How to access external systems
- How to turn on secure communication between pods
- How to implement authentication
- How Istios RBAC works

We were using policies in the previous section. One important part of the security in Istio is the definition of authentication policies (which have the kind Policy). The authentication policies can be defined global (Istio wide), namespace wide or service specific. The detailed documentation for this policy can be found here: [Authentication Policy](https://istio.io/docs/reference/config/istio.authentication.v1alpha1/).

### Egress or ServiceEntries

There are to points I'd like to point out at the beginning:

1. Egress are put in security because it allow to control how external resources are accessed.
1. Egress are not offering the same functionality as an OpenShift egress pod. It has no unique IP address assigned to it.

The egress functionality is done by creating ServiceEntries.

![Request flow Istio](https://istio.io/docs/concepts/traffic-management/gateways-1.svg)

The reference documentation for the ServiceEntry can be found here: [ServiceEntry](https://istio.io/docs/reference/config/networking/v1alpha3/service-entry/)

Let's play with the egress functionality with the exercise [ServiceEntry](https://redhat-developer-demos.github.io/istio-tutorial/istio-tutorial/1.1.x/8egress.html)

### Transport security with mutual TLS

Let me quote the Istio documentation on this subject:

> - Transport authentication, also known as service-to-service authentication: verifies the direct client making the connection. Istio offers mutual TLS as a full stack solution for transport authentication. You can easily turn on this feature without requiring service code changes. This solution:
>
> - Provides each service with a strong identity representing its role to enable interoperability across clusters and clouds.
> - Secures service-to-service communication and end-user-to-service communication.
> - Provides a key management system to automate key and certificate generation, distribution, and rotation.

You can get the complete documentation of this topic here: [Istio Authentication](https://istio.io/docs/concepts/security/#authentication). Also have a look at the [Best practices for deployments in respect of security and certificates](https://istio.io/docs/concepts/security/#best-practices).

The default installation of Istio on Minishift set the system wide mTLS (mutual TLS) mode to permissive, which means that the sidecar container accept both plaintext and mutual Traffic at the same time. For the workshop we are going to change the mTLS mode for one service.

In the first exercise we are going to use mTLS for the requests between the preference service and recommendation pods. To do that execute the following commands:

``` bash
oc create -f istio-workshop-part2/security/mtls/virtualservice.yaml
oc create -f istio-workshop-part2/security/mtls/destinationrule.yaml
```

To verify that the connections are using TLS, you can either use tcpdump in one of the pods or use the graph overview in Kiali. To see the mTLS badge enabled you need to set the checkbox `Security` in the Dropdown `Display`.

Lets look down our project with this policy, so that every request in the project needs to use mTLS:

``` yaml
apiVersion: "authentication.istio.io/v1alpha1"
kind: "Policy"
metadata:
  name: "default"
  namespace: "tutorial"
spec:
  peers:
  - mtls: {}
```

Please execute  this command:

``` bash
oc create -f istio-workshop-part2/security/mtls/policy.yaml
```

After we wait a bit you should see something like this if you execute a curl to the demo application:

``` bash
$ istio-tutorial/scripts/run.sh istio-ingressgateway-istio-system.$(minishift ip).nip.io/customer
upstream connect error or disconnect/reset before headers. reset reason: connection terminationupstream connect error or disconnect/reset before headers. reset reason: connection terminationupstream connect error or disconnect/reset before headers. reset reason: connection terminationupstream connect error or disconnect/reset before headers. reset reason: connection terminationupstream connect error or disconnect/reset before headers. reset reason: connection terminationupstream connect error or disconnect/reset before headers. reset reason: connection terminationupstream connect error or disconnect/reset before headers. reset reason: connection terminationupstream connect error or disconnect/reset before headers. reset reason: connection terminationupstream connect error or disconnect/reset before headers.
```

To fix that we need to create destination rules with mTLS enabled. You can do this with the following commands:

``` bash
oc create -f istio-workshop-part2/security/mtls/destinationrule_all.yaml
oc create -f istio-workshop-part2/security/mtls/virtualservice_all.yaml
```

After this the demo application should work again and you should see a padlock on all services in Kiali

### End user authentication with JWT

Before we start with the exercise lets go back to the Authentication Policy because not only can you define that the services must use mTLS but also you can define which authentication method should be used. In our case we will use a Keycloak server that we are going to install on our Minishift VM. The documentation of the authentication method can be found here [Origin Authentication Method](https://istio.io/docs/reference/config/istio.authentication.v1alpha1/#OriginAuthenticationMethod).

In the previous version of the tutorial a Keycloak container was deployed and a user defined in it. If you what to use the Keycloak server instead of hard coded value you can find the example here: [Authentication preparation](https://redhat-developer-demos.github.io/istio-tutorial/istio-tutorial/1.0.x/8jwt.html#preparation)

After this we should be able to to the exercise: [End-user authentication with JWT](https://redhat-developer-demos.github.io/istio-tutorial/istio-tutorial/1.1.x/8jwt.html#enablingauthentication)

### Istio Role Based Access Control

There is one more thing that I like to meantion before we do the exercise. That is authorization. Istio has a role base access model at namespace-level, service-level and method-level. IMHO the documentation of Istio is describing best how it works and what the purpose of ServiceRole and ServiceRoleBinding is. The documentation can be found here [Istio Authorization](https://istio.io/docs/concepts/security/#authorization)

Now lets to the exercise: [Istio Role Based Access Control (RBAC)](https://redhat-developer-demos.github.io/istio-tutorial/istio-tutorial/1.1.x/8rbac.html)

**Exercise** Lets add ServiceRoles and ServiceRoleBinding that only the service customer is allowed to call the service preference. How can we validate that?
