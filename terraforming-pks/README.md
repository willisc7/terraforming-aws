# Terraforming PKS

## Configuring PKS

This assumes that you deployed Ops Manager and properly configured the BOSH Director Tile.

1. Download PKS, upload PKS tile to Ops Manager, stage PKS, configure PKS, and login to the `pks` cli.

```

export OM_TARGET="https://$(terraform output ops_manager_dns)"
export OM_USERNAME=admin
export OM_PASSWORD=${OM_PASSWORD} # Change this to your password.

pivnet dlpf -p pivotal-container-service -r $(pivnet releases -p pivotal-container-service --format json | jq -r -c ".[0].version") -g '*.pivotal'
om -k upload-product --product $(ls -1 *.pivotal)
om -k stage-product --product-name pivotal-container-service --product-version $(unzip -p pivotal-container-service*.pivotal 'metadata/*.yml' | yq -c -r '.product_version') 
om -k curl --path /api/v0/staged/vm_extensions/pks-api-lb-security-group -x PUT -d '{"name": "pks-api-lb-security-group", "cloud_properties": { "security_groups": ["pks_api_lb_security_group"] }}'
om -k configure-product -n pivotal-container-service -c <(texplate execute ../ci/assets/template/pks-config.yml -f <(jq -e --raw-output '.modules[0].outputs | map_values(.value)' terraform.tfstate) -o yaml)
om -k apply-changes
export PKS_USER=admin
export PKS_PASSWORD=$(om curl --silent --path /api/v0/deployed/products/$(om curl --silent --path /api/v0/deployed/products | jq -r -c ".[1].guid")/credentials/.properties.uaa_admin_password | jq -r -c ".credential.value.secret")
export PKS_ENDPOINT=$(terraform output pks_api_endpoint)
pks login -a ${PKS_ENDPOINT} -u ${PKS_USER} -p ${PKS_PASSWORD} -k
```

1. Create a new cluster:

```
export CLUSTER_NAME=dev
export CLUSTER_HOST=dev.example.com
pks create-cluster ${CLUSTER_NAME} -e ${CLUSTER_HOST} --plan medium
pks get-credentials dev
```

Once the cluster is completed, you will probably want to add a load balancer to access the API with `kubectl`, to do that see: [terraforming-k8s](../terraforming-k8s/README.md).

## Troubleshooting

### SSH into Ops Manager

```
terraform output ops_manager_ssh_private_key > ops_manager_ssh_private_key.pem && \
        chmod 600 ops_manager_ssh_private_key.pem && \
        ssh -oStrictHostKeyChecking=accept-new -oUserKnownHostsFile=/dev/null ubuntu@$(terraform output ops_manager_dns) -i ops_manager_ssh_private_key.pem
```