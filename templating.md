As you know, when you apply a manifest file to Kubernetes - the YAML-formatted resource descriptions that Kubernetes
 can understand - you must specify the resource name which must be unique for that type of resource (and within the
  same namespace), otherwise Kubernetes will complain that the resource already exists.

For example, you can only have one Pod named `myapp-1234` within the same namespace, but you can have one Pod and one
 Deployment that are each named `myapp-1234`.

As we want to run multiple Spark jobs simultaneously, and as these Spark jobs are mostly identical except for a few
 runtime parameters, we need to parameterize, or _templatize_, our Kubernetes YAML files.
 
Normally, you don't do that, at least that's not in Kubernetes' philosophy: Kubernetes files should be template-free
 and should only be patched, by the means of [Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/) 
 for instance. [Helm](https://helm.sh/) has also its own templating system.  

But as we want to automate a maximum of things, and we want to operate Kubernetes from Python code (Yes, we do! Oh
, my god, what a plot twist! See the following section), we cannot use such a tool.
Instead, we are going to substitute references to variables of the form `$VAR` or `${VAR}` with the corresponding
 values, exactly like [envsubst](https://www.gnu.org/software/gettext/manual/html_node/envsubst-Invocation.html), but
  programmatically.