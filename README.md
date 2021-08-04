# Assisted-Installer-API
Using Assisted Installer with API

reference - https://cloudcult.dev/cilium-installation-openshift-assisted-installer/

# Downloading Offline Token

1. Click [here](https://console.redhat.com/openshift/token) to get the token
   ![Openshift Console](https://github.com/rh-telco-tigers/Assisted-Installer-API/blob/main/images/load-token.png)

2. Click on Load Token
   ![Load Token](https://github.com/rh-telco-tigers/Assisted-Installer-API/blob/main/images/copy-token.png)

3. Use copy to clipboard function and provide it as a variable OFFLINE_ACCESS_TOKEN
   ```bash
   OFFLINE_ACCESS_TOKEN="<PASTE_TOKEN_HERE>"

   export TOKEN=$(curl \
   --silent \
   --data-urlencode "grant_type=refresh_token" \
   --data-urlencode "client_id=cloud-services" \
   --data-urlencode "refresh_token=${OFFLINE_ACCESS_TOKEN}" \
   https://sso.redhat.com/auth/realms/redhat-external/protocol/openid-connect/token | \
   jq -r .access_token)
   ```
4. Create a file (cluster-details) with following content. You can modfy this as per your environment. 
    ```bash
    export ASSISTED_SERVICE_API="api.openshift.com"
    export CLUSTER_VERSION="4.8"
    export CLUSTER_IMAGE="quay.io/openshift-release-dev/ocp-release:4.8.2-x86_64"
    export CLUSTER_NAME="test-rgrs"
    export CLUSTER_DOMAIN="home.lab"
    export CLUSTER_NET_TYPE="openshiftSDN"
    export CLUSTER_CIDR_NET="10.128.0.0/14"
    export CLUSTER_CIDR_SVC="172.31.0.0/16"
    export CLUSTER_HOST_NET="172.30.244.0/24"
    export CLUSTER_HOST_PFX="23"
    export CLUSTER_WORKER_HT="Enabled"
    export CLUSTER_WORKER_COUNT="0"
    export CLUSTER_MASTER_HT="Enabled"
    export CLUSTER_MASTER_COUNT="0"
    export CLUSTER_SSHKEY='<PUT-PUBLIC-SSH-KEY-HERE-AND-LEAVE-SINGLE-QUOTES>'
    ```
5. Export the variables in your environment
    ```bash
    #source cluster-details.sh
    ```
6. Verify that you can communicate with API with following curl command
    ```bash
    curl -s -X GET "https://$ASSISTED_SERVICE_API/api/assisted-install/v1/clusters" \
    -H "accept: application/json" \
    -H "Authorization: Bearer $TOKEN" \
    | jq -r
    ```
7. Now download the pull secret from [here](https://cloud.redhat.com/openshift/install/pull-secret) and put it in a file pull-scret.txt
    ![Pull Secret](https://github.com/rh-telco-tigers/Assisted-Installer-API/blob/main/images/pull-secret.png)
    
8. Create a variable with raw content of pull-secret.txt file.
    ```bash
    PULL_SECRET=$(cat pull-secret.txt | jq -R .)
    ```
9. Create an Assisted-Service deployment.json file
    ```bash
    cat << EOF > ./deployment.json
    {
      "kind": "Cluster",
      "name": "$CLUSTER_NAME",
      "openshift_version": "$CLUSTER_VERSION",
      "ocp_release_image": "$CLUSTER_IMAGE",
      "base_dns_domain": "$CLUSTER_DOMAIN",
      "hyperthreading": "all",
      "cluster_network_cidr": "$CLUSTER_CIDR_NET",
      "cluster_network_host_prefix": $CLUSTER_HOST_PFX,
      "service_network_cidr": "$CLUSTER_CIDR_SVC",
      "user_managed_networking": true,
      "vip_dhcp_allocation": false,
      "host_networks": "$CLUSTER_HOST_NET",
      "hosts": [],
      "ssh_public_key": "$CLUSTER_SSHKEY",
      "pull_secret": $PULL_SECRET
    }
    EOF
    ```
10. Create the cluster via Assisted-Servcice API this will generate a "cluster id" whcih will need to be exported for future use.
     ```bash
     curl -s -X POST "https://$ASSISTED_SERVICE_API/api/assisted-install/v1/clusters" \
     -d @./deployment.json \
     --header "Content-Type: application/json" \
     -H "Authorization: Bearer $TOKEN" \
     | jq '.id'
     "6c0d6c3a-7c4e-4a2b-9448-5a00a0195914"

    CLUSTER_ID="6c0d6c3a-7c4e-4a2b-9448-5a00a0195914"
    ```
11. Update the cluster install-config via the Assisted-Service API
     ```bash
     curl \
     --header "Content-Type: application/json" \
     --request PATCH \
     --data '"{\"networking\":{\"networkType\":\"Cilium\"}}"' \
     -H "Authorization: Bearer $TOKEN" \
     "https://$ASSISTED_SERVICE_API/api/assisted-install/v1/clusters/$CLUSTER_ID/install-config"
     ```
12. Review your changes by issuing following curl command
    ```bash
    curl -s -X GET \
    --header "Content-Type: application/json" \
    -H "Authorization: Bearer $TOKEN" \
    "https://$ASSISTED_SERVICE_API/api/assisted-install/v1/clusters/$CLUSTER_ID/install-config" \
    | jq -r
    
    apiVersion: v1
    baseDomain: home.lab
    networking:
      networkType: Cilium
      clusterNetwork:
      - cidr: 10.128.0.0/14
        hostPrefix: 23
      machineNetwork:
      - cidr: 172.30.244.0/24
      serviceNetwork:
      - 172.31.0.0/16
    metadata:
      name: test-rgrs
    compute:
    - hyperthreading: Enabled
      name: worker
      replicas: 0
    controlPlane:
      hyperthreading: Enabled
      name: master
      replicas: 0
    platform:
      none: {}
      vsphere: null
    fips: false
    pullSecret: 'Your-Pull-Secret'
    sshKey: 'Your-SSH_KEY'
    ```
13. Now you will see a cluster created in https://console.redhat.com/openshift/assisted-installer/clusters
    ![AI Console](https://github.com/rh-telco-tigers/Assisted-Installer-API/blob/main/images/ai-console.png)

14. Click on cluster name for details. Review and click on "Next"
    ![cluster details](https://github.com/rh-telco-tigers/Assisted-Installer-API/blob/main/images/cluster-details.png)
 
15. In Host discovery Tab click on Generate Discovery ISO button. Download the ISO and boot you VM/Baremetal with ISO.
    ![discovery host](https://github.com/rh-telco-tigers/Assisted-Installer-API/blob/main/images/discovery-iso.png)

16. In the networking tab review the details of service ip. Note that service IP is the one which we specified in cluster-details file
    ![Service IP](https://github.com/rh-telco-tigers/Assisted-Installer-API/blob/main/images/service-ip.png)

17. Proceed to click on Next and Install the cluster
