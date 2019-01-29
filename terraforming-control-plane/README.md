# Terraforming Control Plane

This creates an environment for running [Concourse](concourse.io) to then run [Platform Automation](http://docs.pivotal.io/pcf-automation/) to create your PAS or PKS environments.

## Configuring Control PLane

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

    export STEMCELL_VERSION=170.24
    export STEMCELL_RELEASE=$(pivnet releases -p stemcells-ubuntu-xenial --format json | jq -r -c "[.[] | select(.version | startswith(\"$STEMCELL_VERSION\")) | .version][0]")
    pivnet dlpf -p stemcells-ubuntu-xenial -r $STEMCELL_RELEASE -g '*aws*' --accept-eula

    ../scripts/ssh $PWD
    ```
1. Follow directions at https://control-plane-docs.cfapps.io/#login-to-control-plane
1. I applied the following patch to the downloaded yml manifest:
    ```diff
    @@ -12,7 +12,7 @@
                tls:
                  ca_cert:
                    certificate: ((control_plane_internal_ca.certificate))
    -            url: https://127.0.0.1:8844
    +            url: ((external_url)):8844
              external_url: ((external_url))
              generic_oauth:
                auth_url: ((external_url)):8443/oauth/authorize
    @@ -21,8 +21,8 @@
                client_id: concourse
                client_secret: ((concourse_client_secret))
                display_name: UAA
    -            token_url: https://localhost:8443/oauth/token
    -            userinfo_url: https://localhost:8443/userinfo
    +            token_url: ((external_url)):8443/oauth/token
    +            userinfo_url: ((external_url)):8443/userinfo
              log_level: debug
              main_team:
                auth:
    @@ -60,8 +60,16 @@
                  uaa:
                    ca_certs:
                      - ((control_plane_internal_ca.certificate))
    -                internal_url: https://localhost:8443
    -                url: https://plane.infra:8443
    +                url: ((external_url)):8443
    +                verification_key: ((uaa_jwt.public_key))
    +            authorization:
    +              permissions:
    +                - path: /*
    +                  actors: ["uaa-client:credhub_admin_client"]
    +                  operations: [read, write, delete, read_acl, write_acl]
    +                - path: /concourse/*
    +                  actors: ["uaa-client:concourse_to_credhub"]
    +                  operations: [read]
                ca_certificate: ((control_plane_internal_ca.certificate))
                data_storage:
                  database: credhub
    @@ -130,6 +138,13 @@
                    refresh-token-validity: 3600
                    scope: credhub.read,credhub.write
                    secret: ""
    +              credhub_admin_client:
    +                override: true
    +                authorized-grant-types: client_credentials
    +                scope: uaa.none
    +                authorities: credhub.read,credhub.write
    +                access-token-validity: 3600
    +                secret: ((credhub-admin-client-password))
                jwt:
                  policy:
                    active_key_id: key-1
    @@ -181,6 +196,7 @@
        update:
          max_in_flight: 1
        vm_type: ((vm_type))
    +    vm_extensions: [control-plane-lb-cloud-properties]
      - azs: ((azs))
        instances: 1
        jobs:
    @@ -295,6 +311,8 @@
        options:
          length: 40
        type: password
    +  - name: credhub-admin-client-password
    +    type: password
      - name: postgres_password
        type: password
      - name: token_signing_key
    @@ -317,16 +335,3 @@
        type: password
      - name: worker_key
        type: ssh
    -  - name: control_plane_internal_ca
    -    options:
    -      common_name: internalCA
    -      is_ca: true
    -    type: certificate
    -  - name: control_plane_tls
    -    options:
    -      alternative_names:
    -        - 127.0.0.1
    -        - localhost
    -      ca: control_plane_internal_ca
    -      common_name: 127.0.0.1
    -    type: certificate
    ```
1. Create a `secrets.yml` file that you'll use to keep your SSL certs inside, it looks like below.  In my case, I used [Let's Encrypt](https://letsencrypt.org/) so the contents of `control_plane_internal_ca.certificate` would be [Let’s Encrypt Authority X3 (Signed by ISRG Root X1)](https://letsencrypt.org/certs/letsencryptauthorityx3.pem.txt).
    ```
    control_plane_tls:
      certificate: |
        -----BEGIN CERTIFICATE-----
        -----END CERTIFICATE-----
      private_key: |
        -----BEGIN RSA PRIVATE KEY-----
        -----END RSA PRIVATE KEY-----
    control_plane_internal_ca:
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