# Terraforming PAS

## Configuring PAS

### Prerequisites

1. `om`
1. `pivnet`
1. `texplate`
1. `terraform`
1. `jq`
1. `yq`

### Deploying PAS and Healthwatch

This assumes that you deployed Ops Manager and properly configured the BOSH Director Tile.

1. Download PAS and Healthwatch, upload PAS and Healthwatch tile to Ops Manager, stage PAS and Healthwatch, configure PAS and Healthwatch, and login to the `cf` cli.

```
# You could copy paste this whole thing, but it's easier to run line-by-line for a better understanding, thus why this isn't a script.
export OM_TARGET="https://$(terraform output ops_manager_dns)"
export OM_USERNAME=admin
export OM_PASSWORD=${OM_PASSWORD} # Change this to your password.

# Download your products -- Small Footprint, Healthwatch, and SSO
pivnet dlpf -p elastic-runtime -r $(pivnet releases -p elastic-runtime --format json | jq -r -c '[.[] | select(.availability | startswith("All User")) | .version][0]') -g 'srt*.pivotal' --accept-eula
pivnet dlpf -p p-healthwatch -r $(pivnet releases -p p-healthwatch --format json | jq -r -c '[.[] | select(.availability | startswith("All User")) | .version][0]') -g '*.pivotal' --accept-eula
pivnet dlpf -p pivotal_single_sign-on_service -r $(pivnet releases -p pivotal_single_sign-on_service --format json | jq -r -c '[.[] | select(.availability | startswith("All User")) | .version][0]') -g '*.pivotal' --accept-eula


# Check to make sure we have the right stemcells for those products.
export STEMCELL_VERSION=$(unzip -p p-healthwatch*.pivotal 'metadata/*.yml' | yq -c -r '.stemcell_criteria.version')
export STEMCELL_RELEASE=$(pivnet releases -p stemcells-ubuntu-xenial --format json | jq -r -c "[.[] | select(.version | startswith(\"$STEMCELL_VERSION\")) | .version][0]")
pivnet dlpf -p stemcells-ubuntu-xenial -r $STEMCELL_RELEASE -g '*aws*' --accept-eula
om -k upload-stemcell --stemcell $(ls -1 *${STEMCELL_RELEASE}*.tgz)

export STEMCELL_VERSION=$(unzip -p Pivotal_Single_Sign-On_Service*.pivotal 'metadata/*.yml' | yq -c -r '.stemcell_criteria.version')
export STEMCELL_RELEASE=$(pivnet releases -p stemcells-ubuntu-xenial --format json | jq -r -c "[.[] | select(.version | startswith(\"$STEMCELL_VERSION\")) | .version][0]")
pivnet dlpf -p stemcells-ubuntu-xenial -r $STEMCELL_RELEASE -g '*aws*' --accept-eula
om -k upload-stemcell --stemcell $(ls -1 *${STEMCELL_RELEASE}*.tgz)

# Upload products
om -k upload-product --product $(ls -1 srt*.pivotal)
om -k upload-product --product $(ls -1 p-healthwatch*.pivotal)
om -k upload-product --product $(ls -1 Pivotal_Single_Sign-On_Service*.pivotal)

# Add custom VM extensions for Network Load Balancers
om -k curl --path /api/v0/staged/vm_extensions/web-lb-security-group -x PUT -d '{"name": "web-lb-security-group", "cloud_properties": { "security_groups": ["web_lb_security_group"] }}'
om -k curl --path /api/v0/staged/vm_extensions/ssh-lb-security-group -x PUT -d '{"name": "ssh-lb-security-group", "cloud_properties": { "security_groups": ["ssh_lb_security_group"] }}'
om -k curl --path /api/v0/staged/vm_extensions/tcp-lb-security-group -x PUT -d '{"name": "tcp-lb-security-group", "cloud_properties": { "security_groups": ["tcp_lb_security_group"] }}'
om -k curl --path /api/v0/staged/vm_extensions/vms -x PUT -d '{"name": "vms", "cloud_properties": { "security_groups": ["vms_security_group"] }}'

# Stage products
om -k stage-product --product-name cf --product-version $(unzip -p srt*.pivotal 'metadata/*.yml' | yq -c -r '.product_version')
om -k stage-product --product-name p-healthwatch --product-version $(unzip -p p-healthwatch*.pivotal 'metadata/*.yml' | yq -c -r '.product_version') 
om -k stage-product --product-name Pivotal_Single_Sign-On_Service --product-version $(unzip -p Pivotal_Single_Sign-On_Service*.pivotal 'metadata/*.yml' | yq -c -r '.product_version') 

# Configure products
om -k configure-product -n cf -c <(texplate execute ../ci/assets/template/srt-config.yml -f <(jq -e --raw-output '.modules[0].outputs | map_values(.value)' terraform.tfstate) -o yaml)

om -k configure-product -n p-healthwatch -c <(texplate execute ../ci/assets/template/p-healthwatch-config.yml -f <(jq -e --raw-output '.modules[0].outputs | map_values(.value)' terraform.tfstate) -o yaml)

om -k configure-product -n Pivotal_Single_Sign-On_Service -c <(texplate execute ../ci/assets/template/Pivotal_Single_Sign-On_Service.yml -f <(jq -e --raw-output '.modules[0].outputs | map_values(.value)' terraform.tfstate) -o yaml)

# Lastly, apply changes.
om -k apply-changes -i

export PAS_USER=admin
export PKS_PASSWORD=$(om curl --silent --path /api/v0/deployed/products/cf-e7239061b8b89610b55b/credentials/.uaa.admin_credentials | jq -r -c ".credential.value.password")
export PKS_ENDPOINT=$(terraform output pks_api_endpoint)
cf login -a ${PKS_ENDPOINT} -u ${PKS_USER} -p ${PKS_PASSWORD} -k
```

## Troubleshooting

### SSH into Ops Manager

```
terraform output ops_manager_ssh_private_key > ops_manager_ssh_private_key.pem && \
        chmod 600 ops_manager_ssh_private_key.pem && \
        ssh -oStrictHostKeyChecking=accept-new -oUserKnownHostsFile=/dev/null ubuntu@$(terraform output ops_manager_dns) -i ops_manager_ssh_private_key.pem
```