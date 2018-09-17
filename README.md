# Webinar deployment

## Create

Before running this tutorial you will need:

1) A Kubernetes (K8S) cluster (you can get free credits to deploy a managed K8S cluster on AWS, GCP, Azure, etc)
2) Helm (and Tiller) installed on K8S
3) An nginx-ingress installation (using the Helm chart)
4) A cert-manager installation (using the Helm chart)
5) A domain name for your components (e.g. the Certificate Authority), connected to your nginx-ingress IP address - you can obtain one for free or $1.00

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

    mkdir -p ./config/AidTechMSP/admincerts

    cp ./config/AidTechMSP/signcerts/* ./config/AidTechMSP/admincerts

Create a secret to hold the admincert

    ORG_CERT=$(ls ./config/AidTechMSP/admincerts/cert.pem)

    kubectl -n blockchain create secret generic hlf--org-admincert --from-file=cert.pem=$ORG_CERT

Find the adminkey and create a secret to hold it

    ORG_KEY=$(ls ./config/AidTechMSP/keystore/*_sk)

    kubectl -n blockchain create secret generic hlf--org-adminkey --from-file=key.pem=$ORG_KEY

### Crypto material

    cd ./config

Create Genesis block and Channel

    configtxgen -profile OrdererGenesis -outputBlock genesis.block

    configtxgen -profile ComposerChannel -channelID mychannel -outputCreateChannelTx mychannel.tx

Save them as secrets

    kubectl -n blockchain create secret generic hlf--genesis --from-file=genesis.block

    kubectl -n blockchain create secret generic hlf--channel --from-file=mychannel.tx

    cd ..

### Kafka for Ordering service

Install Kafka chart (use special values to ensure 4 Kafka brokers and that Kafka messages don't disappear)

    helm install incubator/kafka -n kafka-hlf --namespace blockchain -f ./helm_values/kafka-hlf_values.yaml

### Fabric Orderer

Install orderers

    helm install stable/hlf-ord -n ord1 --namespace blockchain -f ./helm_values/ord1_values.yaml

Register orderer with CA

    ORD_1_SECRET=$(kubectl -n blockchain get secret ord1-hlf-ord -o jsonpath="{.data.CA_PASSWORD}" | base64 --decode)

    kubectl exec $CA_POD -n blockchain  -- fabric-ca-client register --id.name ord1 --id.secret $ORD_1_SECRET --id.type orderer

Get logs from orderer to check it's actually started

    ORD_1=$(kubectl get pods -n blockchain -l "app=hlf-ord,release=ord1" -o jsonpath="{.items[0].metadata.name}")

    kubectl logs $ORD_1 -n blockchain | grep "completeInitialization"

> Repeat all above steps for Orderer 2, etc.

### Fabric Peer

Install CouchDB chart

    helm install stable/hlf-couchdb -n cdb-peer1 --namespace blockchain -f ./helm_values/cdb-peer1_values.yaml

Check that CouchDB is running

    CDB_1=$(kubectl get pods --namespace blockchain -l "app=hlf-couchdb,release=cdb-peer1" -o jsonpath="{.items[*].metadata.name}")

    kubectl logs $CDB_1 -n blockchain | grep "Apache CouchDB has started on"

    helm install aidtech/hlf-peer -n peer1 --namespace blockchain -f /Users/sasha/Aid_Tech/deployer/trace_donate/lf_hlf_webinar/values/hlf-peer/peer1.yaml

Register peer with CA

    PEER_1_SECRET=$(kubectl -n blockchain get secret peer1-hlf-peer -o jsonpath="{.data.CA_PASSWORD}" | base64 --decode)

    kubectl exec $CA_POD -n blockchain  -- fabric-ca-client register --id.name peer1 --id.secret $PEER_1_SECRET --id.type orderer

Check that Peer is running

    PEER_1=$(kubectl get pods -n blockchain -l "app=hlf-peer,release=peer1" -o jsonpath="{.items[0].metadata.name}")

    kubectl logs $PEER_1 -n blockchain | grep "Starting peer"

> Repeat all above steps for Peer 2, etc.

Create channel (do this only once in Peer 1)

    kubectl exec $PEER_1 -n blockchain -- peer channel create -o ord1-hlf-ord.blockchain.svc.cluster.local:7050 -c mychannel -f /hl_config/channel/mychannel.tx

Fetch and join channel for Peer 2, etc.

    kubectl exec $PEER_2 --namespace blockchain -- peer channel fetch config /var/hyperledger/mychannel.block -c mychannel -o ord1-hlf-ord.blockchain.svc.cluster.local:7050

    kubectl exec $PEER_2 --namespace blockchain -- bash -c 'CORE_PEER_MSPCONFIGPATH=$ADMIN_MSP_PATH peer channel join -b /var/hyperledger/mychannel.block'

[bis for Peer 2, etc.]

### Delete deployment

    helm delete --purge peer1 peer2 cdb-peer1 cdb-peer2 kafka-hlf ord1 ord2 ca ca-pg

    rm -rf ./config/*MSP ./config/genesis.block ./config/mychannel.tx

    kubectl delete statefulset -n blockchain kafka-log kafka-hlf-zookeeper kafka-hlf

    kubectl delete pvc -n blockchain ca-pg-postgresql data-kafka-hlf-zookeeper-0 data-kafka-hlf-zookeeper-1 data-kafka-hlf-zookeeper-2 datadir-kafka-hlf-0 datadir-kafka-hlf-1 datadir-kafka-hlf-2 datadir-kafka-hlf-3

    kubectl delete secret -n blockchain hlf--org-admincert  hlf--org-adminkey hlf--channel hlf--genesis
