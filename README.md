# Digester

Digester resolves tags to
[digests](https://cloud.google.com/solutions/using-container-images) for
container and initContainer images in Kubernetes
[Pod](https://kubernetes.io/docs/concepts/workloads/pods/) and
[Pod template](https://kubernetes.io/docs/concepts/workloads/pods/#pod-templates)
specs.

It replaces container image references that use tags:

```yaml
spec:
  containers:
  - image: gcr.io/google-samples/hello-app:1.0
```

With references that use the image digest:

```yaml
spec:
  containers:
  - image: gcr.io/google-samples/hello-app:1.0@sha256:95214fdf834ae96b1969e38c9768f5180366fdf430e5e31b39f7defb584698fb
```

Digester can run either as a
[mutating admission webhook](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)
in a Kubernetes cluster, or as a client-side
[Kubernetes Resource Model (KRM) function](https://kpt.dev/book/02-concepts/03-functions)
with the [kpt](https://kpt.dev/) or
[kustomize](https://kubectl.docs.kubernetes.io/guides/introduction/kustomize/)
command-line tools.

If a tag points to an
[image index](https://github.com/opencontainers/image-spec/blob/master/image-index.md#oci-image-index-specification)
or
[manifest list](https://docs.docker.com/registry/spec/manifest-v2-2/#manifest-list),
digester resolves the tag to the digest of the image index or manifest list.

The webhook is opt-in at the namespace level by label, see
[Deploying the webhook](#deploying-the-webhook).

If you use
[Binary Authorization](https://cloud.google.com/binary-authorization/docs),
digester can help to ensure that only verified container images can be deployed
to your clusters. A Binary Authorization
[attestation](https://cloud.google.com/binary-authorization/docs/key-concepts#attestations)
is valid for a particular container image digest. You must deploy container
images by digest so that Binary Authorization can verify the attestations for
the container image. You can use digester to deploy container images by digest.

## Running the KRM function

1.  Download the digester binary for your platform from the
    [Releases page](../../releases).

    Alternatively, you can download the latest version using these commands:

    ```sh
    VERSION=v0.1.4

    curl -Lo digester "https://github.com/google/k8s-digester/releases/download/$VERSION/digester_$(uname -s)_$(uname -m)"

    chmod +x digester
    ```

2.  [Install kpt](https://kpt.dev/installation/) v1.0.0-beta.1 or later, and/or
    [install kustomize](https://kubectl.docs.kubernetes.io/installation/kustomize/)
    v4.2.0 or later.

3.  Run the digester KRM function:

    -   Using kpt:

        ```sh
        kpt fn source [manifest directory] \
          | kpt fn eval - --exec ./digester
        ```

    -  Using kustomize:

        ```sh
        kustomize fn source [manifest files or directory] \
          | kustomize fn run --enable-exec --exec-path ./digester
        ```

    By running as an executable, the KRM function has access to container
    image registry credentials in the current environment, such as the current
    user's
    [Docker config file](https://github.com/google/go-containerregistry/blob/main/pkg/authn/README.md#the-config-file)
    and
    [credential helpers](https://docs.docker.com/engine/reference/commandline/login/#credential-helper-protocol).
    For more information, see the digester documentation on
    [Authenticating to container image registries](docs/authentication.md).

## Deploying the webhook

You need a Kubernetes cluster version 1.16 or later.

1.  Grant yourself the `cluster-admin` Kubernetes
    [cluster role](https://kubernetes.io/docs/reference/access-authn-authz/rbac/):

    ```sh
    kubectl create clusterrolebinding cluster-admin-binding \
      --clusterrole cluster-admin \
      --user "$(gcloud config get-value core/account)"
    ```

2.  [Install kpt](https://kpt.dev/installation/).

3.  Get the digester webhook kpt package and store the files in a directory
    called `manifests`:

    ```sh
    VERSION=v0.1.4

    kpt pkg get https://github.com/google/k8s-digester.git/manifests@$VERSION manifests
    ```

4.  Deploy the webhook:

    ```sh
    kpt live apply manifests --reconcile-timeout=3m --output=table
    ```

5.  Add the `digest-resolution: enabled` label to namespaces where you want the
    webhook to resolve tags to digests:

    ```sh
    kubectl label namespace [NAMESPACE] digest-resolution=enabled
    ```

To configure how the webhook authenticates to your container image registries,
see the documentation on
[Authenticating to container image registries](docs/authentication.md).

### Private clusters

If you install the webhook in a
[private Google Kubernetes Engine (GKE) cluster](https://cloud.google.com/kubernetes-engine/docs/how-to/private-clusters),
you must add a firewall rule. In a private cluster, the nodes only have
[internal IP addresses](https://cloud.google.com/vpc/docs/ip-addresses).
The firewall rule allows the API server to access the webhook running on port
8443 on the cluster nodes.

1.  Create an environment variable called `CLUSTER`. The value is the name of
    your cluster that you see when running `gcloud container clusters list`:

    ```sh
    CLUSTER=[your private GKE cluster name]
    ```

2.  Look up the IP address range for the cluster API server and store it in an
    environment variable:

    ```sh
    API_SERVER_CIDR=$(gcloud container clusters describe $CLUSTER \
      --format 'value(privateClusterConfig.masterIpv4CidrBlock)')
    ```

3.  Look up the
    [network tags](https://cloud.google.com/vpc/docs/add-remove-network-tags)
    for your cluster nodes and store them comma-separated in an environment
    variable:

    ```sh
    TARGET_TAGS=$(gcloud compute firewall-rules list \
      --filter "name~^gke-$CLUSTER" \
      --format 'value(targetTags)' | uniq | paste -d, -s -)
    ```

4.  Create a firewall rule that allow traffic from the API server to cluster
    nodes on TCP port 8443:

    ```sh
    gcloud compute firewall-rules create allow-api-server-to-digester-webhook \
      --action ALLOW \
      --direction INGRESS \
      --source-ranges "$API_SERVER_CIDR" \
      --rules tcp:8443 \
      --target-tags "$TARGET_TAGS"
    ```

You can read more about private cluster firewall rules in the
[GKE private cluster documentation](https://cloud.google.com/kubernetes-engine/docs/how-to/private-clusters#add_firewall_rules).

### Deploying using kubectl

We recommend deploying the webhook using kpt as described above. If you are
unable to use kpt, you can deploy the digester using kubectl:

```sh
git clone https://github.com/google/k8s-digester.git digester
cd digester
VERSION=v0.1.4
git checkout $VERSION
kubectl apply -f manifests/namespace.yaml
kubectl apply -f manifests/
```

## Documentation

-   [Motivation](docs/motivation.md)

-   [Recommendations](docs/recommendations.md)

-   [Authenticating to container image registries](docs/authentication.md)

-   [Configuring GKE Workload Identity for authenticating to Container Registry and Artifact Registry](docs/workload-identity.md)

-   [Resolving common issues](docs/common-issues.md)

-   [Troubleshooting](docs/troubleshooting.md)

-   [Building digester](docs/build.md)

-   [Developing digester](docs/development.md)

-   [Releasing digester](docs/release.md)

## Disclaimer

This is not an officially supported Google product.
