# Terraforming PKS

## Prerequisites
- install jq and yq (pip)
- install texplate (https://github.com/pivotal-cf/texplate/releases/download/v0.3.0/texplate_linux_amd64)
- PKS CLI (https://network.pivotal.io)
- kubectl (curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl)
- uaac (requires ruby-dev, https://github.com/cloudfoundry/cf-uaac)
- om cli from https://github.com/pivotal-cf/om/releases
- bosh cli from https://bosh.io/docs/cli-v2-install/
- docker

## Configuring PKS

This assumes that you deployed Ops Manager and properly configured the BOSH Director Tile.

1. Download PKS, upload PKS tile to Ops Manager, stage PKS, configure PKS, and login to the `pks` cli.

```
export OM_TARGET="https://$(terraform output ops_manager_dns)"
export OM_USERNAME=admin
export OM_PASSWORD=${OM_PASSWORD} # Change this to your password.
export PIVNET_API_TOKEN=your-token

om -k configure-authentication --decryption-passphrase ${OM_PASSWORD}
om download-product --pivnet-api-token $PIVNET_API_TOKEN --stemcell-iaas aws -p pivotal-container-service -f "*.pivotal" -o . -r "^1..*"

# Note: see which products fail due to EULA and log into pivnet and accept the EULA

om -k upload-product --product $(ls -1 pivotal-container-service*.pivotal)
om -k upload-stemcell --stemcell $(ls -1 light-bosh-stemcell*.tgz)
om -k stage-product --product-name pivotal-container-service --product-version $(unzip -p pivotal-container-service*.pivotal 'metadata/*.yml' | yq -c -r '.product_version') 
om -k create-vm-extension -n pks-api-lb-security-groups -cp '{ "security_groups": ["pks_api_lb_security_group", "vms_security_group"] }'
om -k configure-product -c <(texplate execute ../ci/assets/template/pks-config.yml -f <(jq -e --raw-output '.modules[0].outputs | map_values(.value)' terraform.tfstate) -o yaml)
om -k apply-changes
```

1. Create a new cluster:

```
export PKS_USER=admin
export PKS_PASSWORD=$(om -k credentials --product-name pivotal-container-service -c .properties.uaa_admin_password -f secret)
export PKS_ENDPOINT=$(terraform output pks_api_endpoint)
pks login -a ${PKS_ENDPOINT} -u ${PKS_USER} -p ${PKS_PASSWORD} -k
export CLUSTER_NAME="a"
export CLUSTER_HOST="a.$(terraform output pks_domain)"
pks create-cluster ${CLUSTER_NAME} -e ${CLUSTER_HOST} --plan medium
pks get-credentials a
```

Once the cluster is completed, you will probably want to add a load balancer to access the API with `kubectl`, to do that see: [terraforming-k8s](../terraforming-k8s/README.md).

## Troubleshooting

### SSH into Ops Manager

```
terraform output ops_manager_ssh_private_key > ops_manager_ssh_private_key.pem && \
        chmod 600 ops_manager_ssh_private_key.pem && \
        ssh -oStrictHostKeyChecking=accept-new -oUserKnownHostsFile=/dev/null ubuntu@$(terraform output ops_manager_dns) -i ops_manager_ssh_private_key.pem
```

### Shutting Down an Already Running Environment
Note: assumes all environment variables are set in .envrc

0. Delete PKS clusters: `pks delete-cluster`

### Spinning Up an Already Configured Environment
Note: assumes all environment variables are set in .envrc

0. `pks create-cluster ${CLUSTER_NAME} -e ${CLUSTER_HOST} --plan medium`
