# resource.hashiui

This repository contains the source code necessary to build Ubuntu Hyper-V hard-drives containing the
[Consul native UI](https://www.consul.io/docs/agent/options.html#_ui).

## Image

The image is created by using the [Linux base image](https://github.com/Calvinverse/base.vm.linux)
and amending it using a [Chef](https://www.chef.io/chef/) cookbook which updates the Consul
configuration to enable the UI.

There are two different images that can be created. One for use on a Hyper-V server and one for use
in Azure. Which image is created depends on the build command line used.

### Contents

In addition to the default applications installed in the template image the following items are
also installed and configured:

* The Consul native UI is enabled

The image is configured to add the Consul UI as service to Consul under the `dashboard`
service name with the `consul` tag. Additionally the services is added to the reverse
proxy so that the UI is available from outside the environment.

### Configuration

The configuration comes from a set of [Consul-Template](https://github.com/hashicorp/consul-template)
template files which replaces some of the template parameters with values from the Consul Key-Value store.

### Provisioning

No additional configuration is applied other than the default one for the base image.

### Logs

No additional configuration is applied other than the default one for the base image.

### Metrics

No additional configuration is applied other than the default one for the base image.

## Build, test and release

The build process follows the standard procedure for
[building Calvinverse images](https://www.calvinverse.net/documentation/how-to-build).

### Hyper-V

For building Hyper-V images use the following command line

    msbuild entrypoint.msbuild /t:build /P:ShouldCreateHypervImage=true /P:RepositoryArchive=PATH_TO_ARTIFACTLOCATION

where `PATH_TO_ARTIFACTLOCATION` is the full path to the directory where the base image artifact
file is stored.

In order to run the smoke tests on the generated image run the following command line

    msbuild entrypoint.msbuild /t:test /P:ShouldCreateHypervImage=true


### Azure

For building Azure images use the following command line

    msbuild entrypoint.msbuild /t:build
        /P:ShouldCreateAzureImage=true
        /P:AzureLocation=LOCATION
        /P:AzureClientId=CLIENT_ID
        /P:AzureClientCertPath=CLIENT_CERT_PATH
        /P:AzureSubscriptionId=SUBSCRIPTION_ID
        /P:AzureImageResourceGroup=IMAGE_RESOURCE_GROUP

where:

* `LOCATION` - The azure data center in which the image should be created. Note that this needs to be the same
  region as the location of the base image. If you want to create the image in a different location then you need to
  copy the base image to that region first.
* `CLIENT_ID` - The client ID of the user that [Packer](https://packer.io) will use to
  [authenticate with Azure](https://www.packer.io/docs/builders/azure#azure-active-directory-service-principal).
* `CLIENT_CERT_PATH` - The client certificate which Packer will use to authenticate with Azure
* `SUBSCRIPTION_ID` - The subscription ID in which the image should be created.
* `IMAGE_RESOURCE_GROUP` - The resource group from which the base image will be pulled and in which the new image
  will be placed once the build completes.

For running the smoke tests on the Azure image

    msbuild entrypoint.msbuild /t:test
        /P:ShouldCreateAzureImage=true
        /P:AzureLocation=LOCATION
        /P:AzureClientId=CLIENT_ID
        /P:AzureClientCertPath=CLIENT_CERT_PATH
        /P:AzureSubscriptionId=SUBSCRIPTION_ID
        /P:AzureImageResourceGroup=IMAGE_RESOURCE_GROUP
        /P:AzureTestImageResourceGroup=TEST_RESOURCE_GROUP

where all the arguments are similar to the build arguments and `TEST_RESOURCE_GROUP` points to an Azure resource
group in which the test images are placed. Note that this resource group needs to be cleaned out after successful
tests have been run because Packer will in that case create a new image.

## Deploy

### Hyper-V

* Download the new image to one of your Hyper-V hosts.
* Create a directory for the image and copy the image VHDX file there.
* Create a VM that points to the image VHDX file with the following settings
  * Generation: 2
  * RAM: at least 1024 Mb
  * Hard disk: Use existing. Copy the path to the VHDX file
  * Attach the VM to a suitable network
* Update the VM settings:
  * Enable secure boot. Use the Microsoft UEFI Certificate Authority
  * Attach a DVD image that points to an ISO file containing the settings for the environment. These
    are normally found in the output of the [Calvinverse.Infrastructure](https://github.com/Calvinverse/calvinverse.infrastructure)
    repository. Pick the correct ISO for the task, in this case the `Linux Consul Client` image
  * Disable checkpoints
  * Set the VM to always start
  * Set the VM to shut down on stop
* Start the VM, it should automatically connect to the correct environment once it has provisioned
* Remove the old VM
  * SSH into the host
  * Issue the `consul leave` command
  * Shut the machine down with the `sudo shutdown now` command
  * Once the machine has stopped, delete it

### Azure

The easiest way to deploy the Azure images into a cluster on Azure is to use the terraform scripts
provided by the [Azure service discovery](https://github.com/Calvinverse/infrastructure.azure.core.servicediscovery)
repository. Those scripts will create a Consul cluster of the suitable size and add a single instance
of a node with the Consul UI enabled.

## Usage

The Consul UI webpage will be made available from the proxy at the `/dashboards/consul` sub-address.
