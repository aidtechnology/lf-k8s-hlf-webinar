# Webinar deployment

## Create

Before running this tutorial you will need:

1) A Kubernetes (K8S) cluster (you can get free credits to deploy a managed K8S cluster on AWS, GCP, Azure, etc)
2) Helm (and Tiller) installed on K8S
3) An nginx-ingress installation (using the Helm chart)
4) A cert-manager installation (using the Helm chart)
5) A domain name for your components (e.g. the Certificate Authority), connected to your nginx-ingress IP address - you can obtain one for free or $1.00
6) The fabric-tools (particularly `configtxgen`) installed on your local computer

### Fabric CA

Install the PostgeSQL chart

    helm install stable/postgresql -n ca-pg --namespace blockchain -f ./helm_values/ca-pg_values.yaml

Install the Fabric CA chart (it automatically gets the login information from the PostgreSQL chart secret)

    helm install stable/hlf-ca -n ca --namespace blockchain -f ./helm_values/ca_values.yaml

Get pod for CA release

    CA_POD=$(kubectl get pods -n blockchain -l "app=hlf-ca,release=ca" -o jsonpath="{.items[0].metadata.name}")

Check if server is ready

    kubectl logs -n blockchain $CA_POD | grep "Listening on"

Check that we don't have a certificate

    kubectl exec $CA_POD -n blockchain  -- cat /var/hyperledger/fabric-ca/msp/signcerts/cert.pem

    kubectl exec $CA_POD -n blockchain  -- bash -c 'fabric-ca-client enroll -d -u http://$CA_ADMIN:$CA_PASSWORD@$SERVICE_DNS:7054'

    CA_INGRESS=$(kubectl get ingress -n blockchain -l "app=hlf-ca,release=ca" -o jsonpath="{.items[0].spec.rules[0].host}")

Check that ingress works correctly

    curl https://$CA_INGRESS/cainfo

Get CA certificate [TODO: This should be using https and Fabric CA client 1.2]

    FABRIC_CA_CLIENT_HOME=./config fabric-ca-client getcacert -u http://$CA_INGRESS -M AidTechMSP

Get identity of org-admin

    kubectl exec $CA_POD -n blockchain  -- fabric-ca-client identity list --id org-admin

Register Organisation admin if the previous command did not work

    kubectl exec $CA_POD -n blockchain  -- fabric-ca-client register --id.name org-admin --id.secret OrgAdm1nPW --id.attrs 'admin=true:ecert'

Enroll the Organisation Admin identity

    FABRIC_CA_CLIENT_HOME=./config fabric-ca-client enroll -u http://org-admin:OrgAdm1nPW@$CA_INGRESS -M AidTechMSP

Copy the signcerts to admincerts

    cp ./config/AidTechMSP/signcerts ./config/AidTechMSP/admincerts

Create a secret to hold the admincert

    ORG_CERT=$(ls ./config/AidTechMSP/admincert/cert.pem)

    kubectl -n blockchain create secret generic hlf--org-admincert --from-file=cert.pem=$ORG_CERT

Find the adminkey and create a secret to hold it

    ORG_KEY=$(ls ./config/AidTechMSP/keystore/*_sk)

    kubectl -n blockchain create secret generic hlf--org-adminkey --from-file=key.pem=$ORG_KEY

### Crypto material

Change directory to the config folder, having the cryptographic material and configuration

    cd ./config

Create Genesis block and Channel

    configtxgen -profile OrdererGenesis -outputBlock genesis.block

    configtxgen -profile ComposerChannel -channelID composerchannel -outputCreateChannelTx composerchannel.tx

Save them as secrets

    kubectl -n blockchain create secret generic hlf--genesis --from-file=genesis.block

    kubectl -n blockchain create secret generic hlf--channel --from-file=composerchannel.tx

Go back to the root directory

    cd ..


### Kafka (for Ordering service)

Install Kafka chart (use special values to ensure 4 Kafka brokers and that Kafka messages don't disappear)

    helm install incubator/kafka -n kafka-hlf -n blockchain -f ./values/kafka-hlf_values.yam

### Fabric Orderer

Install orderers

    helm install stable/hlf-ord -n ord1 --namespace blockchain -f ./values/ord1_values.yaml

Register orderer with CA

    ORD1_SECRET=$(kubectl -n blockchain get secret ord1-hlf-ord -o jsonpath="{.data.CA_PASSWORD}" | base64 --decode)

    kubectl exec $CA_POD -n blockchain  -- fabric-ca-client register --id.name ord1 --id.secret $ORD1_SECRET --id.type orderer

Get logs from orderer to check it's actually started

    ORD_1=$(kubectl get pods -n blockchain -l "app=hlf-ord,release=ord1" -o jsonpath="{.items[0].metadata.name}")

    kubectl logs $ORD1 -n blockchain | grep "completeInitialization"

> Repeat all above for Orderer 2, etc.

### Fabric Peer

Install CouchDB chart

    helm install stable/hlf-couchdb -n cdb-peer1 --namespace blockchain -f ./values/cdb-peer1_values.yaml

Check that CouchDB is running

    CDB_1=$(kubectl get pods --namespace blockchain -l "app=hlf-couchdb,release=cdb-peer1" -o jsonpath="{.items[*].metadata.name}")

    kubectl logs $CDB1 -n blockchain | grep "Apache CouchDB has started on"

    helm install aidtech/hlf-peer -n peer1 --namespace blockchain -f /Users/sasha/Aid_Tech/deployer/trace_donate/lf_hlf_webinar/values/hlf-peer/peer1.yaml

Register peer with CA

    PEER1_SECRET=$(kubectl -n blockchain get secret peer1-hlf-ord -o jsonpath="{.data.CA_PASSWORD}" | base64 --decode)

    kubectl exec $CA_POD -n blockchain  -- fabric-ca-client register --id.name peer1 --id.secret $PEER1_SECRET --id.type orderer

Check that Peer is running

    PEER_1=$(kubectl get pods -n blockchain -l "app=hlf-ord,release=peer1" -o jsonpath="{.items[0].metadata.name}")

    kubectl logs $PEER_1 -n blockchain | grep "Starting peer"

> Repeat all above commands for Peer 2, etc.

Create channel (do this only once in Peer 1)

    kubectl exec $PEER_1 -n blockchain -- peer channel create -o ord1-hlf-ord.blockchain.svc.cluster.local:7050 -c mychannel -f /hl_config/channel/composerchannel.tx

Fetch channel for peer 1

    kubectl exec $PEER_1 --namespace blockchain -- peer channel fetch config /var/hyperledger/composerchannel.block -c mychannel -o ord1-hlf-ord.blockchain.svc.cluster.local:7050

Join channel (peer 1 may give an error, as they auto-joined when creating)

    kubectl exec $PEER_1 --namespace blockchain -- bash -c 'CORE_PEER_MSPCONFIGPATH=$ADMIN_MSP_PATH peer channel join -b /var/hyperledger/composerchannel.block'

> Repeat 2 commands above for Peer 2, etc.

### Delete deployment

    helm delete --purge peer1 peer2 cdb-peer1 cdb-peer2 kafka-hlf ord1 ord2 ca ca-pg

    rm -rf ./config/*MSP ./config/genesis.block ./config/composerchannel.tx

    kubectl delete statefulset -n blockchain kafka-log kafka-hlf-zookeeper kafka-hlf
    kubectl delete pvc -n blockchain ca-pg-postgresql data-kafka-hlf-zookeeper-0 data-kafka-hlf-zookeeper-1 data-kafka-hlf-zookeeper-2 datadir-kafka-hlf-0 datadir-kafka-hlf-1 datadir-kafka-hlf-2 datadir-kafka-hlf-3
    kubectl delete secret -n blockchain hlf--org-admincert  hlf--org-adminkey hlf--channel hlf--genesis
