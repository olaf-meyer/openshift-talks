# What is part of the talk?

First things first! Lets clarify why the OpenShift command line interface is important -> IMHO It allows you to create customized queries that are not possible in the Web-Consol and it can (and will) be used for automation. Which is a big part of DevOps -> Learn tools at hand and how to use them.

FIXME BOT talking about odo!!

## Features in OpenShift CLI (oc tool)

- Get a list of all available resources including CRD with the command `oc api-resources`. Try `oc api-resources -o wide` and `oc api-resources --api-group=image.openshift.io` as well.
- Use `oc explain` to get a description about resources like pod, deployment config build config and other resources. To get details of pods use `oc explain pod`. To dig deep into the resource append a `.` and a append the resource name of which you what to know more `oc explain pod.spec`.
- You can add the parameter `--as` to your command to execute it as defined user. For example to query builds as the service account  Login mit Service Account `oc get builds.build.openshift.io test-1 --as=system:serviceaccount:myproject:builder`. You can login as a service account with the following command `oc login --token=$(oc sa get-token builder)`.
- Usually with the command line tool you fetch a single resource like pods, deployment config and so on. If you concatenate resources with a comma you can query multiple resources at once. This is useful for creating your own templates. An example is `oc get is,bc,dc,svc,route -o yaml`.
- You can change the way resources are displayed in `oc get` command with the parameter `-o`.
  - Printout only resource type and name with `o name` i.e. `oc get po -o name`.
  - It is possible to sort the columns by a custom attribute like i.e. `oc get imagestreamtag -o custom-columns=NAME:.metadata.name,CREATED:.metadata.creationTimestamp --sort-by=.metadata.creationTimestamp -n openshift`
  - For more complicated output you can use either use golang template
 or jsonpath template. This becomes handy if you want to display the used imagestreams for all build and build strategies in an OpenShift cluster:
 `oc get bc -o go-template='{{range .items}}{{.metadata.namespace}}{{"\t"}}{{.metadata.name}}{{"\t"}}{{if .spec.strategy.dockerStrategy}}{.spec.strategy.dockerStrategy.from.name}}{{else if spec.strategy.sourceStrategy}}{{ .spec.strategy.sourceStrategy.from.name}}{ else if .spec.strategy.customStrategy}}{{ .spec.strategy.customStrategy.from.name}}{{end}}{{"\n"}}{{end}}' --all-namespaces`
- To copy files between your computer and a pod Kubernetes offers the `kubectl cp` command which is available on OpenShift as well. In OpenShift there is the command `oc rsync` which offers a syncing of folder between local computer and a pod out of the box. It works on all platforms that support the oc client.
- oc apply -> A word of warning: `kubectl apply` and `oc apply` a like are great command to update a resource. However, use this command only if you are sure that the resource has not been altered manually, because it does a three way comparison between current, previous and next version. Manual modification "usually" not update the previous version as well.

## OC Patch

Similar to `oc patch` allows you to modify resources in OpenShift. IMHO `the oc patch` command can more verbose about the changes that should be performed. I don't what to go into details about every feature of the patch command, I what to highlight in this talk is how easy it is modify resources via JSONPatch

- Adding values in the yaml with `oc patch dc deployment-example --type='json' -p='[{"op": "add", "path": "/metadata/labels/version", "value": "version1" },{"op": "add", "path": "/spec/template/metadata/labels/version", "value": "version1" }]'`
- Replacing values in yaml with `oc patch dc deployment-example --type='json' -p='[{"op": "replace", "path": "/metadata/labels/version", "value": "version2" },{"op": "replace", "path": "/spec/template/metadata/labels/version", "value": "version2" }]'`
- remove

## Erweiterung von oc bzw. kubectl

- stern
- rakkless
- ksniff
- popeye
- k9s
- hey & siege