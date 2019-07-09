# What is part of the talk?

First things first! Lets clarify why the OpenShift command line interface is important -> IMHO it allows you to create customized queries that are not possible in the Web-Console and it can (and will) be used for automation. Automation is a big part of DevOps -> Learn tools at hand and how to use them. Furthermore it can be extended with plugins.

For the snippets below I was using oc cli with version 4.1. Plugins should work with older versions as well. All commands are only tested on Linux.

## Features in OpenShift CLI (oc tool)

- Get a list of all available resources including CRD with the command `oc api-resources`. Try out `oc api-resources -o wide` and `oc api-resources --api-group=image.openshift.io` as well.
- Use `oc explain` to get a description about resources like pods, deployment configs, build configs and other resources. To get details of pods use `oc explain pod`. To dig deeper into the resource append a `.` and the sub resource name of which you what to know more `oc explain pod.spec`.
- You can add the parameter `--as` to your command to execute it as defined user. For example to query builds as the service account  Login with Service Account `oc get builds.build.openshift.io test-1 --as=system:serviceaccount:myproject:builder`. You can login as a service account with the following command `oc login --token=$(oc sa get-token builder)`.
- Usually with the command line tool you fetch a single resource like pods, deployment config and so on. If you concatenate resources with a comma you can query multiple resources at once. This is useful for creating your own templates. An example is `oc get is,bc,dc,svc,route -o yaml`.
- You can change the way resources are displayed in `oc get` command with the parameter `-o`.
  - Printout only resource type and name with `-o name` i.e. `oc get po -o name`.
  - It is possible to sort the columns by a custom attribute like i.e. `oc get imagestreamtag -o custom-columns=NAME:.metadata.name,CREATED:.metadata.creationTimestamp --sort-by=.metadata.creationTimestamp -n openshift`
  - For more complicated output you can either use golang template
 or jsonpath template. This becomes handy if you want to display the used imagestreams for all build and build strategies in an OpenShift cluster:

     ```bash
    {% raw%}
    oc get bc -o go-template='{{range .items}}{{.metadata.namespace}}{{"\t"}}{{.metadata.name}}{{"\t"}}{{if .spec.strategy.dockerStrategy}}{{.spec.strategy.dockerStrategy.from.name}}{{else if .spec.strategy.sourceStrategy}}{{ .spec.strategy.sourceStrategy.from.name}}{{else if .spec.strategy.customStrategy}}{{ .spec.strategy.customStrategy.from.name}}{{end}}{{"\n"}}{{end}}' --all-namespaces
    {% endraw %}
    ```

- To copy files between your computer and a pod Kubernetes offers the `kubectl cp` command which is available on OpenShift as well. In OpenShift there is the command `oc rsync` which offers a syncing of a folder between your local computer and a pod out of the box. It works on all platforms that support the oc client.
- oc apply -> A word of warning: `kubectl apply` and `oc apply` a like are great commands to update a resource. However, use this command only if you are sure that the resource has not been altered manually, because it does a three way comparison between current, previous and next version. Manual modification "usually" not update the previous version as well.

## OC Patch

Similar to `oc apply` `oc patch` allows you to modify resources in OpenShift. IMHO the `oc patch` command is verbose about the changes that should be performed than `oc apply`. I don't want to go into details about every feature of the patch command, I just want to highlight in this talk how easy it is modify resources via JSONPatch.

- Adding values in the yaml with `oc patch dc deployment-example --type='json' -p='[{"op": "add", "path": "/metadata/labels/version", "value": "version1" },{"op": "add", "path": "/spec/template/metadata/labels/version", "value": "version1" }]'`
- Replacing values in yaml with `oc patch dc deployment-example --type='json' -p='[{"op": "replace", "path": "/metadata/labels/version", "value": "version2" },{"op": "replace", "path": "/spec/template/metadata/labels/version", "value": "version2" }]'`
- Removing values with `oc patch dc deployment-example --type json -p '[{ "op": "remove", "path": "/spec/template/metadata/labels/version" },{ "op": "remove", "path": "/metadata/labels/version" }]'`

## Extensions or plugins for oc or kubectl

- [stern](https://github.com/wercker/stern) --> Log-Output of multiple Pods and allows filtering of Pods via regular expressions.
- [popeye](https://github.com/derailed/popeye) --> Shows potential issues of deployments and configuration intended for Kubernetes but works well with OpenShift
- [k9s](https://github.com/derailed/k9s) --> Text based "graphical" user for Kubernetes but works partly for OpenShift as well.
- [rakkless](https://github.com/corneliusweig/rakkess) --> Shows matrix of permissions for a user, since `oc policy can-i --list --user=<user>` is deprecated with OpenShift 3.11.
- [ksniff](https://github.com/eldadru/ksniff) --> Allows tcp dump of pods
- [hey](https://github.com/rakyll/hey) & [siege](https://github.com/JoeDog/siege) --> Simple load test tools
- [select](https://github.com/brendandburns/kubectl-select) --> Plugin to select resource for further processing i.e. `oc rsh $(oc select po)`
