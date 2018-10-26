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
pivnet dlpf -p elastic-runtime -r $(pivnet releases -p elastic-runtime --format json | jq -r -c '[.[] | select(.availability | startswith("All User")) | .version][0]') -g 'srt*.pivotal' --accept-eula
pivnet dlpf -p p-healthwatch -r $(pivnet releases -p p-healthwatch --format json | jq -r -c '[.[] | select(.availability | startswith("All User")) | .version][0]') -g '*.pivotal' --accept-eula
export STEMCELL_VERSION=$(unzip -p p-healthwatch*.pivotal 'metadata/*.yml' | yq -c -r '.stemcell_criteria.version')
export STEMCELL_RELEASE=$(pivnet releases -p stemcells --format json | jq -r -c "[.[] | select(.version | startswith(\"$STEMCELL_VERSION\")) | .version][0]")
pivnet dlpf -p stemcells -r $STEMCELL_RELEASE -g '*aws*' --accept-eula
om -k upload-stemcell --stemcell $(ls -1 *.tgz)
om -k upload-product --product $(ls -1 srt*.pivotal)
om -k upload-product --product $(ls -1 p-healthwatch*.pivotal)
om -k stage-product --product-name cf --product-version $(unzip -p srt*.pivotal 'metadata/*.yml' | yq -c -r '.product_version')
om -k stage-product --product-name p-healthwatch --product-version $(unzip -p p-healthwatch*.pivotal 'metadata/*.yml' | yq -c -r '.product_version') 
texplate execute ../ci/assets/template/srt-config.yml -f <(jq -e --raw-output '.modules[0].outputs | map_values(.value)' terraform.tfstate) -o yaml > srt-config.yml
texplate execute ../ci/assets/template/p-healthwatch-config.yml -f <(jq -e --raw-output '.modules[0].outputs | map_values(.value)' terraform.tfstate) -o yaml > p-healthwatch-config.yml
om -k configure-product -n cf -c srt-config.yml
om -k configure-product -n p-healthwatch -c p-healthwatch-config.yml
om -k apply-changes -i
export PKS_USER=admin
export PKS_PASSWORD=$(om curl --silent --path /api/v0/deployed/products/pivotal-container-service-54233ccfafc7e8775e8f/credentials/.properties.uaa_admin_password | jq -r -c ".credential.value.secret")
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