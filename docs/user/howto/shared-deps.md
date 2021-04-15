# Working with shared dependencies

Hypper charts understand the concept of shared dependency charts.

A chart declared as a shared dependency is a chart that more than one chart may
depend on. Charts deployed as shared dependencies are the analogous of system
libraries in an OS: dependencies that are used by several consumers. One
deployment may satisfy the shared dependencies of several charts.

## Creating a Hypper chart with shared dependencies

Let's create the simplest chart possible, and add charts as shared dependencies
to it.

First, we create a simple empty chart:

```terminal
$ helm create our-app
```

Now, we can edit `./our-app/Chart.yaml`, and add some shared dependencies to it:

```diff
apiVersion: v2
name: our-app
description: A Helm chart for Kubernetes
type: application
version: 0.1.0
appVersion: 1.16.0
annotations:
+ hypper.cattle.io/shared-dependencies: |
+   - name: postgresql
+     version: "~8.9.4"
+     repository: "https://charts.bitnami.com/bitnami"
+   - name: prometheus
+     version: "~13.7.0"
+     repository: "https://prometheus-community.github.io/helm-charts"
```

Shared dependencies are just normal Helm
[Dependencies](https://helm.sh/docs/topics/charts/#chart-dependencies). As
such, they are defined with:
- The name field is the name of the chart you want.
- The version field is the version of the chart you want.
- The repository field is the full URL to the chart repository. Note that you
  must also use helm repo add to add that repo locally. You might use the name
  of the repo instead of URL.

We can also add a default release name and namespace, where our-app and its
shared-dependencies will get installed:

```diff
apiVersion: v2
name: our-app
description: A Helm chart for Kubernetes
type: application
version: 0.1.0
appVersion: 1.16.0
annotations:
+ hypper.cattle.io/namespace: hypper
+ hypper.cattle.io/release-name: our-app-name
  hypper.cattle.io/shared-dependencies: |
    - name: postgresql
      version: "~8.9.4"
      repository: "https://charts.bitnami.com/bitnami"
    - name: prometheus
      version: "~13.7.0"
      repository: "https://prometheus-community.github.io/helm-charts"
```

To verify that we did create the correctly, let's lint it:

```terminal
$ hypper lint our-app
==> Linting our-app
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed
```

## Listing shared dependencies

Hypper's `shared-dep list` command will list the shared dependencies, its status, and other information:

```
$ hypper shared-deps list our-app
NAME            VERSION REPOSITORY                                              STATUS
postgresql      ~8.9.4  https://charts.bitnami.com/bitnami                      not-installed
prometheus      ~13.7.0 https://prometheus-community.github.io/helm-charts      not-installed
```


## Deploying shared dependencies

First, add the repos of the shared dependencies, so they are found:

```terminal
$ hypper repo add bitnami 'https://charts.bitnami.com/bitnami'
"bitnami" has been added to your repositories
$ hypper repo add prometheus-community 'https://prometheus-community.github.io/helm-charts'
"prometheus-community" has been added to your repositories
$ hypper repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "stable" chart repository
...Successfully got an update from the "bitnami" chart repository
...Successfully got an update from the "prometheus-community" chart repository
🛳  Update Complete.
```


Now, let's pretend that we had `prometheus` already installed, so let's install
it by hand:

```
$ hypper install prometheus-community/prometheus -n hypper --create-namespace
Installing chart "prometheus" in namespace "hypper"…
Done! 👏
```

That satisfies one shared dependency of `our-app`:

```terminal
$ hypper shared-deps list our-app
NAME            VERSION REPOSITORY                                              STATUS
postgresql      ~8.9.4  https://charts.bitnami.com/bitnami                      not-installed
prometheus      ~13.7.0 https://prometheus-community.github.io/helm-charts      deployed
```

Then, we can install `our-app`, and any of its missing shared dependencies:

```terminal
$ hypper install our-app --create-namespace
Installing chart "postgresql" in namespace "hypper"…
Installing chart "prometheus" in namespace "hypper"…
Installing chart "hypper-synapse" in namespace "hypper"…
Done! 👏
```

What has happened?
1. Hypper has made sure that all declared shared dependencies of `our-app` are
   satisfied, installing those are missing. Since the chart of the shared
   dependency didn't have annotations for default namespace, it will install it
   on the dependent namespace.
2. Since we haven't specified the release name or namespace in the command,
   Hypper has installed our-app in the default release-name (`our-app-name`) and
   namespace (`hypper`) we specified in the hypper annotations.

If you want, you can always install `our-app` without the shared-dependencies, by
passing the flag `--no-shared-deps`.
