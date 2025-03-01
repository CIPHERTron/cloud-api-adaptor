# Introduction

This directory contain the framework to run complete end-to-end (e2e) tests. It was built upon the [kubernetes-sigs e2e framework](https://github.com/kubernetes-sigs/e2e-framework).

# Running end-to-end tests

As long as the cloud provider support is implemented on this framework, you can run the tests
as shown below for *libvirt*:

```
$ CLOUD_PROVIDER=libvirt make test-e2e
```

The above command run tests on an existing cluster. It will look for the kubeconf file exported on the
`KUBECONFIG` variable, and then in `$HOME/.kube/config` if not found. 

You can instruct the tool to provision a test environment though, as shown below:

```
$ TEST_E2E_PROVISION=yes CLOUD_PROVIDER=libvirt make test-e2e
```

The `TEST_E2E_PODVM_IMAGE` is an optional variable which specifies the path to the podvm qcow2 image. If it is set then the image should be uploaded to the VPC storage. The following command, as an example, instructs the tool to upload `path/to/podvm-base.qcow2` after the provisioning of the test environment:

```
$ TEST_E2E_PROVISION=yes TEST_E2E_PODVM_IMAGE="path/to/podvm-base.qcow2" CLOUD_PROVIDER=libvirt make test-e2e
```

# Adding support for a new cloud provider

In order to add a test pipeline for a new cloud provider, you will need to implement some
Go interfaces and create a test suite. You will find the reference implementation on the files
for the *libvirt* provider.

## Create the provision implementation

Create a new Go file (.go) named `provision_<CLOUD_PROVIDER>`.go (e.g., `provision_libvirt.go`)
that should be tagged with `//go:build <CLOUD_PROVIDER>`. That file should have the implementation
of the `CloudProvision` interface (see its definition in [provision.go](./provision.go)).

Apart from that, it should implement the `func GetCloudProvisioner() (CloudProvision, error)` factory function.

## Create the test suite

Create another Go file named `<CLOUD_PROVIDER>_test.go` to host the test suite and provider-specific assertions. It is interpreted as any [Go standard testing](https://pkg.go.dev/testing) framework test file, where functions with `func TestXxx(*testing.T)` pattern are tests to be executed.

Likewise the provision file, you should tag the test file with `//go:build <CLOUD_PROVIDER>`.

You can have tests specific for the cloud provider or re-use the existing suite found in
[common_suite.go]() (or mix both). In the later cases, you must first implement the `CloudAssert` interface (see its definition in [common.go](./common.go)) because some tests will need to do assertions on the cloud side, so there should provider-specific asserts implementations.   

Once you got the assertions done, create the test function which wrap the common suite function. For example, suppose there is a re-usable `doTestCreateSimplePod` test then you can wrap it in test function like shown below:  

```go
func TestCloudProviderCreateSimplePod(t *testing.T) {
	assert := MyAssert{}
	doTestCreateSimplePod(t, assert)
}
```