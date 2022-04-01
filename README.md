# Monitoring-On-Chain-Full-Node-Data-With-Splunk
This guide covers directly monitoring on-chain block data, transactional data, node information and metrics from your full nodes in Splunk, as well as how to visualize that data leveraging prebuilt Ethereum dashboards.  This guide builds off of the previous repo [here](https://github.com/maverick705/Splunk-Setup-For-Chainlink) which covers Splunk basic configuration and setup, and also leverages the [Ethlogger](https://github.com/splunk/splunk-connect-for-ethereum) custom connector built by Splunk for this purpose.

**Disclaimer**: This is not an official guide by Splunk, but a passionate Chainlink community member.  Splunk is a complex and sophisticated monitoring solution, and it is recommended to have at least some pre-existing knowledge before attempting to deploy this in your own environment.  This guide however will provide the necessary information to get the Ethlogger connector up and running.

## Initial deployment
Please review the pre-existing Ethlogger docs on Github for more information on the connector. Ethlogger can be deployed using Docker with Docker run commands or docker-compose. This guide will leverage the latter. Requirements include:
* A linux server with Splunk already installed and configured utilizing the previous [Splunk setup guide](https://github.com/maverick705/Splunk-Setup-For-Chainlink).
* Docker and Docker-Compose installed
* Root level privileges

#### Example Rinkeby Node Gas Analytics Dashboard
![Rinkeby_Gas_Analytics](Rinkeby_Gas_Analytics.png)

#### Example Polygon Address Explorer
![Polygon_Address_Explorer](Polygon_Address_Explorer.png)

## Create Splunk indexes
You will need to create three different indexes in Splunk. Two event indexes and one metric index for your full node metrics.

### Create event indexes
Modify the config file at /opt/splunk/etc/system/local/indexes.conf and add the two event indexes.
By default these indexes store 500GB of ingested data and have no time based retention settings. Therefore once an index fills up, data will start to be deleted off the back end be default. View Splunk docs on [indexes.conf](https://docs.splunk.com/Documentation/Splunk/latest/admin/Indexesconf) for more information on data retention settings.
```
vi /opt/splunk/etc/system/local/indexes.conf

## Add these to the file.
[ethereum]
coldPath = /indexes/ethereum/colddb
homePath = /indexes/ethereum/db
thawedPath = /indexes/ethereum/thaweddb

[ethlogger]
coldPath = /indexes/ethlogger/colddb
homePath = /indexes/ethlogger/db
thawedPath = /indexes/ethlogger/thaweddb
```

### Create a metric index
This index is for storing your full node metrics data. Modify the same file above.
Metric data is utilized differently than event data in Splunk.
```
[ethereum_metrics]
coldPath = /indexes/ethereum_metrics/colddb
homePath = /indexes/ethereum_metrics/db
thawedPath = /indexes/ethereum_metrics/thaweddb
datatype=metric
```
### Restart Splunk for changes to take effect. 
In general it is good practice to restart Splunk after any OS level config change.
```
cd /opt/splunk/bin
./splunk restart
```

## Create HEC tokens
You will need to create four new HEC tokens which are used to send the Docker and Ethlogger data over http to your Splunk instance.
Complete the below step four times, and give each HEC token a different name i.e. "ethlogger docker", "ethereum events", "ethereum metrics", and "ethereum internal metrics".
```
1. Navigate to Settings
2. Navigate to Data Inputs
3. Click "HTTP Event Collector"
4. Click "New Token"
5. Specify a name and hit next
6. Select "Review"
7. Select "Submit"
8. Copy the HEC token for use in the next step
```
## Ethlogger deployment and setup
For simplicity, we will use the same server that Splunk is deployed on for deploying the Ethlogger connector.
```
cd ~
git clone "https://github.com/splunk/splunk-connect-for-ethereum.git"
cd splunk-connect-for-ethereum
```


### Create the docker-compose.yaml file
This will be the main config file used for deploying Ethlogger with docker-compose.
```
cd cd ~/splunk-connect-for-ethereum/
touch docker-compose.yaml
vi docker-compose.yaml
```

### Update the docker-compose.yaml file
* There are many different configuration settings here which can be viewed in the Ethlogger docs, but see below for an example of how I built the yaml file.
* The following docker-compose file is pointed towards two full Rinkeby GETH nodes but feel free to scale back if only monitoring one node.  Block data is only being collected from the first full node as the block data is a considerable amount of ingest, and ingesting block data from both nodes leads to duplicate data in Splunk.
* Pay careful attention and ensure you update all 8 HEC token locations as well as all four IPs for your Splunk instance and full node(s).
* x-logging, x-log-opts, and logging at the bottom are optional for collecting the docker log data from the ethlogger containers.
```
version: "3.6"
x-logging:
  &default-logging
  driver: "splunk"
x-log-opts:
  &log-opts
  splunk-token: <HEC token for "ethlogger docker" index>
  splunk-url: https://<splunk IP>:8088
  splunk-insecureskipverify: "true"
  splunk-verify-connection: "false"
  splunk-format: "json"
  tag: "{{.Name}}-{{.ID}}"

services:
  ethlogger:
    image: ghcr.io/splunkdlt/ethlogger:3.5.0
    container_name: ethlogger
    environment:
      - ETH_RPC_URL=http://<full node 1 IP>:8545
      - START_AT_BLOCK=latest
      - SPLUNK_HEC_URL=https://<Splunk IP>:8088
      - SPLUNK_HEC_TOKEN=<HEC token for ethereum index>
      - SPLUNK_EVENTS_HEC_TOKEN=<HEC token for ethereum index>
      - SPLUNK_METRICS_HEC_TOKEN=<HEC token for ethereum_metrics index>
      - SPLUNK_INTERNAL_HEC_TOKEN=<HEC token for ethereum_metrics index>
      - SPLUNK_EVENTS_INDEX=ethereum
      - SPLUNK_METRICS_INDEX=ethereum_metrics
      - SPLUNK_INTERNAL_INDEX=ethereum_metrics
      - SPLUNK_HEC_REJECT_INVALID_CERTS=false
      - ABI_DIR=/app/abis
      - COLLECT_NODE_METRICS=true
      - COLLECT_NODE_INFO=true
      - NETWORK_NAME=testnet
      - CHAIN_NAME=rinkeby
    volumes:
      - ./abis:/app/abis
      - ./:/app
      - ./checkpoint.json:/app/checkpoint.json
    restart: always
    logging:
      <<: *default-logging
      options:
        <<: *log-opts
        splunk-sourcetype: "docker:ethlogger"
        splunk-source: "ethlogger-primary"
        splunk-index: "ethlogger"
  ethlogger-secondary:
    image: ghcr.io/splunkdlt/ethlogger:3.5.0
    container_name: ethlogger-secondary
    environment:
      - ETH_RPC_URL=http://<Full Node 2 IP>:8545
      - START_AT_BLOCK=latest
      - SPLUNK_HEC_URL=https://<Splunk IP>:8088
      - SPLUNK_HEC_TOKEN=<HEC token for ethereum index>
      - SPLUNK_EVENTS_HEC_TOKEN=<HEC token for ethereum index>
      - SPLUNK_METRICS_HEC_TOKEN=<HEC token for ethereum_metrics index>
      - SPLUNK_INTERNAL_HEC_TOKEN=<HEC token for ethereum_metrics index>
      - SPLUNK_EVENTS_INDEX=ethereum
      - SPLUNK_METRICS_INDEX=ethereum_metrics
      - SPLUNK_INTERNAL_INDEX=ethereum_metrics
      - SPLUNK_HEC_REJECT_INVALID_CERTS=false
      - ABI_DIR=/app/abis
      - COLLECT_BLOCKS=false
      - COLLECT_NODE_METRICS=true
      - COLLECT_NODE_INFO=true
      - NETWORK_NAME=testnet
      - CHAIN_NAME=rinkeby
    volumes:
      - ./abis1:/app/abis
      - ./:/app
      - ./checkpoint1.json:/app/checkpoint.json
    restart: always
    logging:
      <<: *default-logging
      options:
        <<: *log-opts
        splunk-sourcetype: "docker:ethlogger"
        splunk-source: "ethlogger-secondary"
        splunk-index: "ethlogger"
```
### Deploy the docker-compose yaml file
Once the yaml file is finished, simply run the below command and if all goes well, your node metrics and log data should be sending to Splunk.
```
docker-compose up -d
```

### Verify that there are no errors in the docker logs
At this point you should have successfully deployed two docker containers, one for each full node.
Run the commands below to verify:
```
docker logs ethlogger
docker logs ethlogger-secondary
```
Based on the docker logs it should be fairly clear whether the ethlogger containers are successfully pulling in data or not.

## Confirm data delivery to Splunk
This is done by running searches on the indexes created above in the Splunk GUI within the search and reporting app.
Run the below searches to confirm data flow:
```
index=ethereum
index=ethlogger
```
If you see data, you are most of the way there but still need to do a few steps.
If you don't, run the below search to verify if data is coming in but with the wrong index or sourcetype.
```
| tstats values(sourcetype) where index=* by index
```

## Install the Ethereum Basics app 
This can be found from the Ethlogger repo [here](https://github.com/splunk/ethereum-basics/releases/tag/1.0.13)
* Download the ethereum-basics_1.0.13.tgz file.
* In the Splunk GUI, navigate to "Apps" in the top left, then click on "Manage Apps"
* Click on "Install app from file" in the top right.
* Select the ethereum-basics app downloaded previously.
* Navigate back to "Apps" to confirm that Ethereum Basics is visible. If not, restart Splunk.

## Update the macro in the search GUI
The macro we need to potentially modify points to your ethereum index and is required for the dashboards to work.
* Navigate to settings in the top right of the GUI, click on "advanced search" and then "search macros".
* Change the filter settings to the ethereum basics app and filter on the ethereum index macro.
* Ensure the macro is pointed towards "index=ethereum" and if it isn't, update it.

## Verify app functionality
Navigate to "Apps" and click on Ethereum Basics. If everything was deployed correctly, you should now see data being populated in your app similar to below.
![Ethereum_Basics_Setup.png](Ethereum_Basics_Setup.png)
Browse through the different tabs at the top of the app for access to the address explorer and gas analytics dashboards.

### Confirm node health
In the Ethereum Basics app, browse to the "node monitoring tab".
Data should automatically populate and look similar to the below image:
![Node_Metrics](Node_Metrics.png)

## Modifying dashboards
If you are only pulling in full node data from one network i.e. Rinkeby, then no changes need to be made to the dashboards. However, if you are pulling in data from multiple networks such as Polygon as well, and both the Rinkeby and Polygon logs are going to the same index with the same sourcetypes, you will need to modify the searches in the dashboards to point to the respective networks.
* Before modifying the default dashboards, first create clones of the dashboards by clicking on the "..." in the top right and clicking clone.

In the cloned dashboard, click edit in the top right, then click source in the top left.
For any line that starts with <query> change from:
  `ethereum_index` sourcetype="ethereum:block"
to
  `ethereum_index` network::rinkeby sourcetype="ethereum:block"


