# Terraforming Control Plane

This creates an environment for running [Concourse](concourse.io) to then run [Platform Automation](http://docs.pivotal.io/pcf-automation/) to create your PAS or PKS environments.

## Configuring Control Plane

### Prerequisites

1. `om`
1. `pivnet`
1. `texplate`
1. `terraform`
1. `jq`
1. `yq`

### Deploying Concourse on the Control Plane

This assumes that you deployed Ops Manager and properly configured the BOSH Director Tile. (See main [README](../README.md))

1. Download Concourse tile, upload concourse tile to Ops Manager, stage concourse, configure concourse, and then deploy it.
    ```
    # You could copy paste this whole thing, but it's easier to run line-by-line for a better understanding, thus why this isn't a script.
    export OM_TARGET="https://$(terraform output ops_manager_dns)"
    export OM_USERNAME=admin
    export OM_PASSWORD=${OM_PASSWORD} # Change this to your password.

    # If you didn't already do this from before --
    om -k apply-changes -i

    # Grabs the credentials for working with BOSH.
    BOSH_CMD=$(om -k curl --silent --path /api/v0/deployed/director/credentials/bosh_commandline_credentials | jq -c -r ".credential")

    # Downloads all the releases we'll need.
    pivnet dlpf -p p-control-plane-components -r $(pivnet releases -p p-control-plane-components --format json | jq -r -c ".[0].version") -g '*' --accept-eula

    export STEMCELL_VERSION=250.17
    export STEMCELL_RELEASE=$(pivnet releases -p stemcells-ubuntu-xenial --format json | jq -r -c "[.[] | select(.version | startswith(\"$STEMCELL_VERSION\")) | .version][0]")
    pivnet dlpf -p stemcells-ubuntu-xenial -r $STEMCELL_RELEASE -g '*aws*' --accept-eula
    om -k upload-stemcell --stemcell $(ls -1 light-bosh-stemcell-*.tgz)

    # Push that stemcell over to the bosh director.
    om -k apply-changes

    ../scripts/ssh $PWD
    ```
1. Follow directions at https://control-plane-docs.cfapps.io/#login-to-control-plane
1. I applied the following patch to the downloaded yml manifest -- **NOTE: If you're going to continue using self-signed certificates, keep the portion at the end; just make sure to change the `alternative_names` to your `external_url` value.**:
    ```diff
    @@ -2,6 +2,7 @@
    instance_groups:
    - azs: ((azs))
    instances: 1
    +  vm_extensions: [control-plane-lb-cloud-properties]
    jobs:
    - consumes: {}
        name: atc
    @@ -331,15 +332,3 @@
    type: password
    - name: worker_key
    type: ssh
    -- name: control-plane-ca
    -  options:
    -    common_name: controlplaneca
    -    is_ca: true
    -  type: certificate
    -- name: control-plane-tls
    -  options:
    -    alternative_names:
    -    - ((wildcard_domain))
    -    ca: control-plane-ca
    -    common_name: control-plane-tls
    -  type: certificate
    ```
1. Create a `secrets.yml` file that you'll use to keep your SSL certs inside, it looks like below.  In my case, I used [Let's Encrypt](https://letsencrypt.org/) so the contents of `control_plane_internal_ca.certificate` would be [Let’s Encrypt Authority X3 (Signed by ISRG Root X1)](https://letsencrypt.org/certs/letsencryptauthorityx3.pem.txt).  **If you're using self-signed certificates leave this part off.**
    ```
    control-plane-tls:
      certificate: |
        -----BEGIN CERTIFICATE-----
        -----END CERTIFICATE-----
      private_key: |
        -----BEGIN RSA PRIVATE KEY-----
        -----END RSA PRIVATE KEY-----
    control-plane-ca:
      certificate: |
        -----BEGIN CERTIFICATE-----
        -----END CERTIFICATE-----
    ```
1. Also using a different deploy command:
    ```
    bosh deploy -d control-plane control-plane-*.yml \
      -v "external_url=https://plane.control.pivdevops.com" \
      -v "persistent_disk_type=10240" \
      -v "network_name=control-plane" \
      -v "azs=[us-east-1a]" \
      -v "vm_type=r4.large" \
      -l secrets.yml
    ```
