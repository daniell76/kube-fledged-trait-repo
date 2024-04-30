# Overview

This repository is used to add Kube Fledge trait to an ACM-enabled Anthos cluster

> :warning: Verify there is NO `<primary-namespace>` namespace on the cluster BEFORE applying this Cluster Trait Repo IF the Primary Root Repo is `structured`.

## How to use

Must apply the following code snippet. This code will create a new `RootSync` object pointing to the Cluster Trait Repo, and create an `ExternalSecret` that allows the `RootSync` to reference a private repository.

> :note: This can be applied via ACM updates IF the Primary Root Repo is `unstructured`, otherwise a manual `kubectl apply -f <file.yaml>` is required to install this CTR.

```yaml
apiVersion: configsync.gke.io/v1beta1
kind: RootSync
metadata:
  name: kubefledged-trait-sync
  namespace: config-management-system
spec:
  git:
    repo: "https://gitlab.com/gcp-solutions-public/retail-edge/available-cluster-traits/kube-fledged-trait-repo.git"
    branch: "main"
    dir: "/config"
    auth: "token"
    secretRef:
      name: kube-fledged-git-creds               # matches the ExternalSecret spec.target.name below

---

apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: kube-fledged-git-creds-es
  namespace: config-management-system
spec:
  refreshInterval: 24h
  secretStoreRef:
    kind: ClusterSecretStore
    name: gcp-secret-store
  target:                                       # K8s secret definition
    name: kube-fledged-git-creds                ############# Matches the secretRef above
    creationPolicy: Owner
  data:
  - secretKey: username                         # K8s secret key name inside secret
    remoteRef:
      key: kube-fledged-access-token-creds      #  GCP Secret Name
      property: username                        # field inside GCP Secret
  - secretKey: token                            # K8s secret key name inside secret
    remoteRef:
      key: kube-fledged-access-token-creds      #  GCP Secret Name
      property: token                           # field inside GCP Secret

```

### Create GCP Secret for git-creds

Create the GCP Secret Manager secret used by `ExternalSecret` to proxy for K8s `Secret`

```
export PROJECT_ID=<your google project id>
export SCM_TOKEN_TOKEN=<your gitlab personal-access token value>
export SCM TOKEN_USER=<your gitlab personal-access token user>

gcloud secrets create kube-fledged-access-token-creds --replication-policy="automatic" --project="${PROJECT_ID}"
echo -n "{\"token\"{{':'}} \"${SCM_TOKEN_TOKEN}\", \"username\"{{':'}} \"${SCM_TOKEN_USER}\"}" | gcloud secrets versions add kube-fledged-access-token-creds --project="${PROJECT_ID}" --data-file=-

```

## Local Validation

Assuming `nomos` is installed (via `gcloud components install nomos`)

```
nomos vet --no-api-server-check --path config/
```

### Docker method

Using this link to find the version of nomos-docker:  https://cloud.google.com/anthos-config-management/docs/how-to/updating-private-registry#expandable-1

```
docker pull gcr.io/config-management-release/nomos:stable
docker run -it -v $(pwd):/code/ gcr.io/config-management-release/nomos:stable nomos vet --no-api-server-check --path /code/config/
```

### ACM Overview

See [our documentation](https://cloud.google.com/anthos-config-management/docs/repo) for how to use each subdirectory.



## Usage

Details are outlined in the project [https://github.com/senthilrch/kube-fledged](https://github.com/senthilrch/kube-fledged)

### Obtain a list of Containers in a Namespace

It will be useful to list out all of the containers used in a Namespace when constructing an `ImageCache` object. This will print out ALL containers used and their counts

```
# Run on the node
kubectl get pods --all-namespaces -o jsonpath="{.items[*].spec['initContainers', 'containers'][*].image}" |\
tr -s '[[:space:]]' '\n' |\
sort |\
uniq -c

```