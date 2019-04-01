# GitHub Action for Deploying Pull Requests via Weave Flux

**Table of Contents**
<!-- toc -->

- [Background](#background)
- [Usage](#usage)
  * [Secrets](#secrets)
  * [Environment Variables](#environment-variables)
- [Notes](#notes)
  * [ADDITIONAL_CONFIG/ADDITIONAL_CONFIG_SUBSTITUTION](#additional_configadditional_config_substitution)
  * [Helm Release Deletion](#helm-release-deletion)
  
<!-- tocstop -->

## Background

[Weaveworks Flux](https://github.com/weaveworks/flux) is a great tool for GitOps-style deployments to Kubernetes. That said, it assumes you have a static set of environments you'd like to deploy your applications to.

Sometimes you want to be able to preview what changes in a pull request will actually do when deployed. This is particularly useful for frontend applications that often require real-world testing. When a pull request is opened, this action will take an existing `HelmRelease` file in a Flux config repo, copy it, make some modifications to avoid naming conflicts, and push the changes back to the repo. When the pull request is closed (or merged), it will delete the pull request `HelmRelease` and commit the change. Effectively, new releases are created and destroyed based on the status of the pull request.



## Usage

```workflow

workflow "Flux Pull Request" {
  on = "pull_request"
  resolves = ["flux-pull-request"]
}

action "flux-pull-request" {
  uses = "stevenpall/github-actions/flux-pull-request@master"
  secrets = ["FLUX_GITHUB_TOKEN"]
  env = {
    FLUX_REPO = "<Name of Flux config repo>" 
    FLUX_BRANCH = "<Flux config repo branch>"
    FLUX_PATH = "<Path to HelmRelease files in Flux config repo>"
    FLUX_SERVICE = "<Name of the service>"
    BASE_BRANCH = "<The application repo branch to be considered the "base" (e.g. master)>"
    COMMIT_USER = "<Username to make commits as>"
    COMMIT_EMAIL = "<Email address to make commits as>"
    ORG = "<Flux config repo user/org>"
    USERNAME = "<Username to use for cloning Flux config repo>"
    ADDITIONAL_CONFIG = "<A JSON representation of any additional configs>"
    ADDITIONAL_CONFIG_SUBSTITUTION = "<true/false> Substitute the service name for branch-name-service-name"
  }
}

```



### Secrets

- `FLUX_GITHUB_TOKEN` - **Required** A separate GitHub token must be created that has access to the Flux config repo. `GITHUB_TOKEN` that gets supplied to the action by default is only scoped to the repo being built. `USERNAME` is the user that this token corresponds to.



### Environment Variables

- FLUX_REPO – **Required**
- FLUX_BRANCH – **Required**
- FLUX_PATH – **Required**
- FLUX_SERVICE – **Required**
- BASE_BRANCH – **Required**
- COMMIT_USER – **Required**
- COMMIT_EMAIL – **Required**
- ORG – **Required**
- USERNAME – **Required**
- ADDITIONAL_CONFIG – **Optional**
- ADDITIONAL_CONFIG_SUBSTITUTION – **Optional**



## Notes



### ADDITIONAL_CONFIG/ADDITIONAL_CONFIG_SUBSTITUTION

Use the `ADDITIONAL_CONFIG` field to specify any additional sections in the generated `HelmRelease` that should added and/or modified. This is necessary in certain circumstances because the YAML structure of the `HelmRelease` is not rigidly defined, and this script cannot account for those variations. This is mainly useful if there are fields that must be modified that correspond to namespaced resources. For example, if you specify an ingress hostname in your `HelmRelease`, this will need to be modified to avoid DNS conflicts.

A simple way to create the JSON string is to run a YAML file through [yq](https://mikefarah.github.io/yq/) and Python `json.dumps`. Given the file:

```
spec:
  values:
    service:
      name: test
    ingress:
      enabled: true
      hosts:
      - name: test.website.com
        paths:
        - name: /*
          serviceName: test
          servicePort: 80
    env:
      APP_URL: "https://test.website.com"
```

Running the command `yq r -j test.yaml | python -c 'import json,sys; print(json.dumps(sys.stdin.read()))'` would create the following JSON representation:

```
"{\"spec\":{\"values\":{\"env\":{\"APP_URL\":\"https://test.website.com\"},\"ingress\":{\"enabled\":true,\"hosts\":[{\"name\":\"test.website.com\",\"paths\":[{\"name\":\"/*\",\"serviceName\":\"test\",\"servicePort\":80}]}]},\"service\":{\"name\":\"test\"}}}}\n"
```

If `ADDITIONAL_CONFIG_SUBSTITUTION` is specified, once this JSON representation is rendered back into YAML and merged with the copied `HelmRelease`, the application branch name will be appended to any instances of `FLUX_SERVICE`. For example, given the above YAML file, `FLUX_SERVICE` set to `test`, and the branch name of `new-feature`, the generated file will look like this:

```
spec:
  values:
    service:
      name: new-feature-test
    ingress:
      enabled: true
      hosts:
      - name: new-feature-test.website.com
        paths:
        - name: /*
          serviceName: new-feature-test
          servicePort: 80
    env:
      APP_URL: "https://new-feature-test.website.com"
```



### Helm Release Deletion

To have Flux clean up resources it created, you must be running version `1.11.0` and include the flag `--sync-garbage-collection`. More information on this feature can be found [here](https://github.com/weaveworks/flux/blob/master/site/garbagecollection.md).

Unfortunately, it appears that there is currently a bug with this feature (https://github.com/weaveworks/flux/issues/1873), so Helm releases may not be properly cleaned up even though their `HelmRelease` has been deleted from the config repo. For now, this may require a manual `helm del --purge <release name>`.
