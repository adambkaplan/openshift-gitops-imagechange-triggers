<!-- 
Copyright Red Hat, Inc.

SPDX-License-Identifier: Apache-2.0
-->
# OpenShift GitOps with ImageChange Triggers

An example of how to apply GitOps practices with OpenShift ImageChange triggers.

## The Problem

Most Kubernetes GitOps practices/patterns assume that developers maintain a source of truth in git.
That source of truth is then reconciled with the state of the cluster, usually by running a
`kubectl apply -f ` command against a directory of YAML manifests.
For most core Kubernetes objects and even newer Custom Resources, this works even if the apiserver
or a mutating admission webhook alters the applied data as the resource is being created or
updated.

Unfortunately this is not the case for OpenShift objects which utilize ImageChange triggers.
Naively running `kubectl apply -f` against these manifests will result in extraneous image change
trigger events for the following reasons:

1. When a trigger is fired, it records the image reference sha in the object's `spec: triggers:`
   array. Triggered image values are not recorded in the `status` subresource. See
   [openshift/origin#23696](https://github.com/openshift/origin/issues/23696) for further details.
2. Due to the implementation of the triggers API, changes to the `spec: triggers:` array cannot be
   strategically merged with any existing content. When `triggers` is included in an object's YAML
   manifest, the entire contents of the `triggers` array is replaced. In practice, this removes the
   injected image reference and causes the trigger to be re-fired.


## Initialize/Patch solution

The solution presented here provides a pattern for using GitOps to deploy a `BuildConfig` with
ImageChange triggers. A similar approach can also work for `DeploymentConfigs`.

Manifests for the GitOps-based deployment are placed in two directories:

1. `init` - these are the initial manifests created in a new namespace. The image change triggers
   for the `BuildConfig` or `DeploymentConfig` should be set here. File are applied in alphabetical
   order. Only one resource should be defined in each manifest file.
2. `patch` - these are [JSON Patch](http://jsonpatch.com/) files applied on top of the resources in
    the `init` directory.

Files need to adopt the following naming conventions:

1. Files in `init` **should** use numeric prefixes to ensure resources are applied in proper order.
   YAML is the recommended format - JSON is also acceptable.
2. Files in `patch` **must** have a prefix that matches a filename in the `init` directory. The
   suffix **should** be of the form `patch.yaml` or `patch.json`. Patch files can be in either YAML
   or JSON format.

Reconciliation of manifests via GitOps is accomplished via the `sync` script.
Without arguments, it applies the patch files in `patch` based on the resources specified in `init`
with the same filename.

To initialize resources, run `$ ./sync --init`.

## Drawbacks

To effectively use this solution, teams need to ensure that changes to their manifests are applied
in two places - the full manifest in `init`, and the patched manifest in `patch`. Ensuring that the
patch YAML/JSON is properly formatted and is idempotent can be challenging.

## Alternatives

Instead of `ImageChange` triggers, teams can utilize the following features to ensure `Builds` are
run with regularly updated images:

1. Enable [scheduled import](https://docs.openshift.com/container-platform/4.5/openshift_images/image-streams-manage.html#images-imagestreams-import_image-streams-managing)
   for images in imagestreams.
2. Create a `CronJob` which runs all builds in the namespace nightly. This `CronJob` can be added
   to the cluster [project template](https://docs.openshift.com/container-platform/4.5/applications/projects/configuring-project-creation.html),
   ensuring all projects run nightly builds.

## License

Licensed under [Apache-2.0](LICENSE).
