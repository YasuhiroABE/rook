# Rook Test Framework

The Rook Test Framework is used to run end to end and integration tests on Rook. The framework depends on a running instance of Kubernetes.
The framework also provides scripts for starting Kubernetes using `kubeadm` or `minikube` so users can
quickly spin up a Kubernetes cluster.The Test framework is designed to install Rook, run tests, and uninstall Rook.

## Requirements
1. Docker version => 1.2 && < 17.0
2. Ubuntu 16 (the framework has only been tested on this version)
3. Kubernetes with kubectl configured
4. Rook

## Instructions

## Setup

### Install Kubernetes
You can choose any Kubernetes flavor of your choice.  The test framework only depends on kubectl being configured.
The framework also provides scripts to install Kubernetes. There are two scripts to start the cluster:
1. **Minikube** (recommended for MacOS): Run [minikube.sh](/tests/scripts/minikube.sh) to setup a single-node Minikube Kubernetes.
    1. Minikube v0.21.0 and higher is supported. Older minikube versions do not have cephfs or rbd tools installed.
    1. If using Kubernetes v1.8 or higher, then Minikube v0.23.0 or higher is required.
1. **Kubeadm** (recommended for Ubuntu): run [kubeadm.sh](/tests/scripts/kubeadm.sh) to setup a single-node K8s cluster using kubeadm

#### Minikube (recommended for MacOS)
Starting the cluster on Minikube is as simple as running:
```console
tests/scripts/minikube.sh up
```

To copy Rook images generated from your local build into the Minikube VM, run the following commands after `minikube.sh up` succeeded:
```
tests/scripts/minikube.sh update
tests/scripts/minikube.sh helm
```

Stopping the cluster and destroying the Minikube VM can be done with:
```console
tests/scripts/minikube.sh clean
```

#### Kubeadm (recommended for Ubuntu)
Starting the cluster using `kubeadm` is as simple as running:
```console
tests/scripts/kubeadm.sh up
```

`kubeadm.sh` starts Kubernetes on the host, so all the images generated by your local build should be available to the cluster without any additional commands.

Stopping the cluster can be done with:
```console
tests/scripts/kubeadm.sh clean
```

#### Alternate Kubernetes Versions
These two scripts can install any version of Kubernetes you wish based on the `KUBE_VERSION` environment variable.
To use an alternate version, simply set this variable before running the relevant `up` command from above.
For example, if you wanted to use `v1.9.6`, you would run `export KUBE_VERSION=v1.9.6` first before running `up`.

### Install Helm
Use [helm.sh](/tests/scripts/helm.sh) to install Helm and set up Rook charts defined under `_output/charts` (generated by build):

- To install and set up Helm charts for Rook run `tests/scripts/helm.sh up`.
- To clean up `tests/scripts/helm.sh clean`.

**NOTE:** `*kubeadm.sh`, `minikube.sh` and `helm.sh` scripts depend on some artifacts under the `_output/` directory generated during build time,
these scripts should be run from project root. e.g., `tests/script/kubeadm.sh up`.

**NOTE**: If Helm is not available in your `PATH`, Helm will be downloaded to a temporary directory (`/tmp/rook-tests-scripts-helm`) and used from that directory.
The temporary directory the Helm binary is in needs to be given to the `integration` binary calls. The flag for that is `--helm=HELM_BINARY`, e.g., `--helm=/tmp/rook-tests-scripts-helm/linux-amd64/helm`.

## Run Tests
From the project root do the following:
#### 1. Build rook:
Run `make build`

#### 2. Start Kubernetes
Using one of the following:

- Using Kubeadm
```
tests/scripts/kubeadm.sh up
tests/scripts/helm.sh up
```
- Using minikube
```
tests/scripts/minikube.sh up
tests/scripts/minikube.sh helm
tests/scripts/helm.sh up
```

#### 3. Run integration tests:
Integration tests can be run using tests binary `_output/tests/${platform}/integration` that is generated during build time e.g.:
```
~/integration -test.v
```

### Test parameters
In addition to standard go tests parameters, the following custom parameters are available while running tests:

| Parameter         | Description                                  | Possible values  | Default           |
| ----------------- | -------------------------------------------- | ---------------- | ----------------- |
| rook_platform     | platform Rook needs to be installed on       | kubernetes       | kubernetes        |
| k8s_version       | version of Kubernetes to be installed        | v1.8+            | v1.8.5            |
| rook_image        | Rook image name to be installed              | valid image name | rook/ceph         |
| skip_install_rook | skips installing Rook (if already installed) | true or false    | false             |

### Running Tests with parameters.
#### To run all integration tests run
```
go test -v -timeout 1800s github.com/rook/rook/tests/integration
```

#### To run a specific suite (uses regex)
```
go test -v -timeout 1800s -run SmokeSuite github.com/rook/rook/tests/integration
```

#### To run specific tests inside a suite:
```
go test -v -timeout 1800s -run SmokeSuite github.com/rook/rook/tests/integration -testify.m TestRookClusterInstallation_smokeTest
```

##### To run specific without installing rook
```
go test -v -timeout 1800s -run SmokeSuite github.com/rook/rook/tests/integration --skip_install_rook
```
If the `skip_install_rook` flag is set to true, then Rook is not uninstalled either.

#### Run Longhaul Tests
[Longhaul](/tests/block/k8s/longhaul) tests are integration tests that run for extended period of time. A load profile can be configured
using the following load test flags

| Parameter          | Description                                       | Possible values         | Default |
| ------------------ | ------------------------------------------------- | ----------------------- | ------- |
| load_parallel_runs | performs concurrent operations                    | any number              | 20      |
| load_volumes       | number of volumes                                 | >1                      | 1       |
| load_time          | number of seconds to run                          | >1                      | 1800    |
| load_size          | size of load profile (3M, 10M, or 50M per thread) | small, medium, or large | medium  |
| enable_chaos       | kill random pods in Rook cluster                  | true or false           | false   |

e.g.
```
go test -run TestObjectLongHaul github.com/rook/rook/tests/longhaul --load_parallel_runs=20 --load_time 1800 --load_size small --load_volumes 3
```
The longhaul test just like other test is going to install Rook if it's not already installed, but it is not going to clean up test data or uninstall Rook after the run.
Longhaul test is designed to run multiple times on the same setup and installation of Rook to tests its stability. Test data and Rook should be cleaned up manually after the test.


You can measure memory, CPU, IOPS, throughput, and other settings on a cluster using Prometheus. The metrics collected during load test can be visualize using Grafana.
Here a couple of helpful links to get prometheus and grafana started and collect metrics:
[kube-prometheus](https://github.com/coreos/prometheus-operator/tree/master/contrib/kube-prometheus) and [cluster-deploy-script](https://github.com/coreos/prometheus-operator/blob/master/contrib/kube-prometheus/hack/cluster-monitoring/deploy)

**Prerequisites**:
* Go installed and GO_PATH set
* Dep installed
* When running tests locally, make sure `kubectl` is accessible globally in your `PATH` as the test framework uses `kubectl`
