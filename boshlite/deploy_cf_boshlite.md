---
title: Deploying Cloud Foundry on vSphere using BOSH
---

This topic describes the process for deploying Cloud Foundry to a BOSH-Lite environment.

##<a id="prerequisites"></a>Prerequisites ##

You must have your BOSH-Lite VM up and running, and have your BOSH CLI targetted at the BOSH-Lite director.  For instructions, consult the [BOSH-Lite repository](https://github.com/cloudfoundry/bosh-lite)

##<a id="stemcell"></a>Upload a Stemcell ##

1. Open [https://bosh.io/stemcells](https://bosh.io/stemcells) in a web browser to view a list of publicly available BOSH stemcells. The list displays the most recent build numbers of BOSH stemcells, organized by operating system, target IaaS, and hypervisor.

1. Choose a BOSH stemcell for "BOSH Lite Warden" and click the build number to download.

1. In a terminal window, run `bosh upload stemcell STEMCELL-NAME` to upload the
stemcell to the BOSH Director.

    <pre class="terminal">
    $ bosh upload stemcell bosh-stemcell-2776-warden-boshlite-ubuntu-trusty-go_agent.tgz
    </pre>

##<a id="create-stub"></a>Create a Deployment Manifest Stub ##

For BOSH-Lite, the minimum required for your stub is just the BOSH Director uuid.  Create a manifest stub file named `cf-stub.yml` with the following contents:

    <pre class="terminal">
    ---
    director_uuid: DIRECTOR_UUID
    </pre>

You can determine the Director UUID with the `bosh status --uuid` command:

    <pre class="terminal">
    $ bosh status --uuid
    b02f8ab5-b63c-4f08-b144-89f96810a022
    </pre>

##<a id="deploy-cf"></a>Deploy Cloud Foundry##

1. Clone the `cf-release` GitHub repository.

    <pre class="terminal">
    $ git clone https://github.com/cloudfoundry/cf-release.git
    </pre>

1. Use the `update` helper script to update the `cf-release` submodules.

    <pre class="terminal">
    $ cd cf-release
    $ ./update
    </pre>

1. Install [spiff](https://github.com/cloudfoundry-incubator/spiff).

1. Run the following command from the `cf-release` directory to create a deployment manifest named `cf-deployment.yml`:

    `./generate_deployment_manifest INFRASTRUCTURE MANIFEST-STUB > cf-deployment.yml`

    Replace INFRASTRUCTURE with `warden` and replace MANIFEST-STUB with the name and location of your `cf-stub.yml file`. For example:

    <pre class="terminal">
	$ ./generate_deployment_manifest warden cf-stub.yml > cf-deployment.yml
    </pre>

1. Use `bosh target` to target the BOSH Director.

    <pre class="terminal">
    $ bosh target
	Current target is https://192.168.50.4:25555 (Bosh Lite Director)
    </pre>

1. Set your deployment to the generated manifest.

    <pre class="terminal">
    $ bosh deployment cf-deployment.yml
    </pre>

1. Use `bosh create release` to create a Cloud Foundry release.
This command prompts you for a development release name.

    <pre class="terminal">
    $ bosh create release
    </pre>

1. Use `bosh upload release` to upload the generated release to the BOSH
Director.

    <pre class="terminal">
    $ bosh upload release
    </pre>

1. Deploy the uploaded Cloud Foundry release.

    <pre class="terminal">
    $ bosh deploy
    </pre>

## <a id="verify"></a>Verify the Deployment ##

Install the [Cloud Foundry CLI](https://github.com/cloudfoundry/cli) and run the following:

<pre class="terminal">
    $ # If using the AWS provider for your BOSH-Lite VM, use https://api.BOSH_LITE_PUBLIC_IP.xip.io
    $ #   instead of https://api.10.244.0.3.xip.io
    $ # If not using AWS, but behind a proxy, make sure to run 'export no_proxy=192.168.50.4,xip.io'
    $ cf api --skip-ssl-validation https://api.10.244.0.34.xip.io
    $ cf auth admin admin
    $ cf create-org test-org
    $ cf target -o test-org
    $ cf create-space test-space
    $ cf target -s test-space
</pre>

Now you are ready to run commands such as `cf push`. If your Cloud Foundry deployment needs to go through an HTTP proxy to reach the Internet, specify `http_proxy`, `https_proxy` and `no_proxy` environment variables using `cf set-env` or add them to the `env:` section of your application's `manifest.yml`. This ensures the buildpacks can download required libraries, gems, etc. during application staging and running.

##<a id="update-cf"></a>Update Cloud Foundry##

* If you make change to your manifest, run `bosh deploy` to update your Cloud Foundry deployment with these changes.

* If you make changes to the `cf-release` directory, run `bosh create release && bosh upload release && bosh deploy` to update your Cloud Foundry deployment with
these changes.