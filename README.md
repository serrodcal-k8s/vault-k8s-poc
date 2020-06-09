# vault-k8s-poc

Proof of concept about Hashicorp Vault solution for K8s using agent injection

## Vault + Kubernetes (vault-k8s)

The `vault-k8s` binary includes first-class integrations between Vault and
Kubernetes.  Currently the only integration in this repository is the
Vault Agent Sidecar Injector (`agent-inject`).  In the future more integrations
will be found here.

The Kubernetes integrations with Vault are
[documented directly on the Vault website](https://www.vaultproject.io/docs/platform/k8s/index.html).
This README will present a basic overview of each use case, but for full
documentation please reference the Vault website.

This project is versioned separately from Vault. Supported Vault versions
for each feature will be noted below. By versioning this project separately,
we can iterate on Kubernetes integrations more quickly and release new versions
without forcing Vault users to do a full Vault upgrade.

## Features

  * [**Agent Inject**](https://www.vaultproject.io/docs/platform/k8s/injector/index.html):
    Agent Inject is a mutation webhook controller that injects Vault Agent containers
    into pods meeting specific annotation criteria.
    _(Requires Vault 1.3.1+)_

## Installation

`vault-k8s` is distributed in multiple forms:

  * The recommended installation method is the official
    [Vault Helm chart](https://github.com/hashicorp/vault-helm). This will
    automatically configure the Vault and Kubernetes integration to run within
    an existing Kubernetes cluster.

  * A Docker image [`hashicorp/vault-k8s`](https://hub.docker.com/r/hashicorp/vault-k8s) is available. This can be used to manually run `vault-k8s` within a scheduled environment.

  * Raw binaries are available in the [HashiCorp releases directory](https://releases.hashicorp.com/vault-k8s/). These can be used to run vault-k8s directly or build custom packages.

## Demo

In this demo we are going to test the new Kubernetes integration that enables applications
with no native HashiCorp Vault logic built-in to leverage static and dynamic secrets
sourced from Vault. This is powered by a new tool called [vault-k8s](https://www.vaultproject.io/docs/platform/k8s/injector/index.html)
, which leverages the Kubernetes Mutating Admission Webhook to intercept and augment
specifically annotated pod configuration for secrets injection using Init and Sidecar
containers.

Applications need only concern themselves with finding a secret at a filesystem
path, rather than managing tokens, connecting to an external API, or other mechanisms
for direct interaction with Vault.

Here are a few supported use-cases:

* _Init_ only container to pre-populate secrets before an application starts. For example, a backup job that runs on a regular schedule and only needs an initial secret at start time.
* _Init_ and _Sidecar_. _Init_ container to fetch secrets before an application starts, and a _Sidecar_ container that starts alongside your application for keeping secrets fresh (sidecar periodically checks to ensure secrets are current). For example, a web application that is using dynamic secrets to connect to a database with an expiring lease.
* Pod authentication through Kubernetes Service Account for Vault Policy enforcement. For example, you likely want to restrict a Pod to only access the secrets they need to function correctly. This should also assist in auditing secret usage of each application.
* Flexible output formatting options using the Vault Agent template functionality which was incorporated from consul-template. For example, fetching secret data from Vault to creating a database connection string, or adapting your output to match pre-existing configuration file formats, etc.

In this demo, we are going to use [Kind](https://kind.sigs.k8s.io/), which is a
local Kubernetes environment. Please, install [Docker](https://www.docker.com/) and
Kind in your system.

Once you have Docker and Kind, first start the Docker daemon and, then, start
the Kubernetes cluster with:

```sh
kind create cluster
```

Let’s configure a demo namespace with:

```sh
kubectl create namespace demo
```

Next, let's set the current context to it:

```sh
kubectl config set-context --current --namespace=demo
```

Finally, let's install Vault using the Helm Chart (Install [Helm](https://helm.sh/)
in your system):

```sh
helm install vault --set='server.dev.enabled=true' ./vault-helm
```

Note that, Vault Helm Chart is provided here which is the 0.4.0 version. In this
directory, we provide the configuration to enable the injector in the `values.yaml`:

```yaml
injector:
  enabled: true
```

Next, connect to Vault and configure a policy named “app” for the demo. Once, your
vault server and the vault agent injector is up and running, get in the `vault-0`
pod with:


```sh
kubectl exec -ti vault-0 /bin/sh
```

Go to `/home/vault` directory with:

```sh
cd /home/vault
```

And, create a `app-policy.hcl` file

```sh
vi app-policy.hcl
```

Put the following configuration in place and save the file:

```
path "secret*" {
  capabilities = ["read"]
}
```

Finally, let's create the policy with:

```sh
vault policy write app /home/vault/app-policy.hcl
```

Next, we want to configure the Vault Kubernetes Auth method and attach our newly
recreated policy to our applications service account:

```sh
vault auth enable kubernetes
```

Let's configure the properties to be able to authenticate against the Kubernetes with:

```sh
vault write auth/kubernetes/config \
   token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
   kubernetes_host=https://${KUBERNETES_PORT_443_TCP_ADDR}:443 \
   kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

Finally, let's link the policies created before with the -namespaces and the Service
Account:

```sh
vault write auth/kubernetes/role/myapp \
   bound_service_account_names=app \
   bound_service_account_namespaces=demo \
   policies=app \
   ttl=1h
```

Note that, we haven't configured any Service Account yet. Let's create it just in
a minute.

Now, let's create an example username and password in Vault using the KV Secrets Engine.
The end goal here, is for this username and password to be injecting into our target
pod's filesystem, which knows nothing about Vault:

```sh
vault kv put secret/helloworld username=foobaruser password=foobarbazpass
```

In this point, we can exit from the `vault-0` pod. For that, just type `exit`.

The `demo/app.yaml` is demo application which spawns a simple web service container
useful for our testing purposes. We are also defining a Service Account which we
can then tie back to the Vault Policy we created earlier. This allows you to specify
each secret an application is allowed to access.

Next, lets launch our example application and create the service account.

```sh
kubectl create -f demo/app.yaml
```

We can also verify there are no secrets mounted at `/vault/secrets`:

```sh
kubectl exec -ti app-XXXXXXXXX -c app -- ls -l /vault/secrets
```

This returns a `No such file or directory` message.

Next, let’s apply our annotations patch to enable the secret injection:

```sh
kubectl patch deployment app --patch "$(cat demo/patch-basic-annotations.yaml)"
```

We can verify the secret has been mounted at `/vault/secrets` with:

```sh
kubectl exec -ti app-XXXXXXXXX -c app -- cat /vault/secrets/helloworld
```

This returns the following output:

```
data: map[password:foobarbazpass username:foobaruser]
metadata: map[created_time:2020-06-09T05:14:45.431541445Z deletion_time: destroyed:false version:1]
```

What happened here, is that when we applied the patch, our vault-k8s webhook intercepted
and changed the pod definition, to include an Init container to pre-populate our secret,
and a Vault Agent Sidecar to keep that secret data in sync throughout our applications
lifecycle.

However, if you were to actually run this, you would notice that the data in `/vault/secrets/helloworld`
is not formatted, and is just a Go struct export of our secret. This can be problematic,
since you almost certainly want the output to be formatted in a particular way, for
consumption by your application. So, what is the solution?

Well, you can format your secret data using by leveraging Vault Agent Templates,
which is very useful for dealing with your various output formatting needs. In the next example here,
we are going to parse our secrets data into a postgresql connection string. Templates
can come in extremely handy and should fit a variety of your use cases, where you need
to conform to existing configuration formats, construct connection stings (as seen below), etc.

```sh
kubectl patch deployment app --patch "$(cat demo/patch-template-annotations.yaml)"
```

We can verify the output with:

```sh
kubectl exec -ti app-XXXXXXXXX -c app -- cat /vault/secrets/helloworld
```

With the output looking like this:

```
postgresql://foobaruser:foobarbazpass@postgres:5432/wizard
```

Taking a look all the service deployed in `demo` namespace, we can see there is
a sidecar container running together our application:

```sh
kubectl get pods                                                                        
NAME                                    READY   STATUS    RESTARTS   AGE
app-847d44cd7f-kqr9z                    2/2     Running   0          79s
vault-0                                 1/1     Running   0          9m38s
vault-agent-injector-66446d56f4-kp2ws   1/1     Running   0          9m49s
```

We can see as `app-847d44cd7f-kqr9z` has `2/2` containers in place.

Finally, let's describe the `app-847d44cd7f-kqr9z` pod with:

```sh
kubectl describe pod app-847d44cd7f-kqr9z
```

We can see the annotations we set before:

```
Annotations:  vault.hashicorp.com/agent-inject: true
              vault.hashicorp.com/agent-inject-secret-helloworld: secret/helloworld
              vault.hashicorp.com/agent-inject-status: injected
              vault.hashicorp.com/agent-inject-template-helloworld:
                {{- with secret "secret/helloworld" -}}
                postgresql://{{ .Data.data.username }}:{{ .Data.data.password }}@postgres:5432/wizard
                {{- end }}
              vault.hashicorp.com/role: myapp
```

And, finally, the events that happen:

```
Events:
  Type    Reason     Age   From                         Message
  ----    ------     ----  ----                         -------
  Normal  Scheduled  34m   default-scheduler            Successfully assigned demo/app-847d44cd7f-kqr9z to kind-control-plane
  Normal  Pulled     34m   kubelet, kind-control-plane  Container image "vault:1.4.2" already present on machine
  Normal  Created    34m   kubelet, kind-control-plane  Created container vault-agent-init
  Normal  Started    34m   kubelet, kind-control-plane  Started container vault-agent-init
  Normal  Pulled     34m   kubelet, kind-control-plane  Container image "jweissig/app:0.0.1" already present on machine
  Normal  Created    34m   kubelet, kind-control-plane  Created container app
  Normal  Started    34m   kubelet, kind-control-plane  Started container app
  Normal  Pulled     34m   kubelet, kind-control-plane  Container image "vault:1.4.2" already present on machine
  Normal  Created    34m   kubelet, kind-control-plane  Created container vault-agent
  Normal  Started    34m   kubelet, kind-control-plane  Started container vault-agent
```

The `vault-agent-init` was created and started to inject our secret in our application.
