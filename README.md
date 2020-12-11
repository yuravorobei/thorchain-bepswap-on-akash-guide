# Thorchain BEPSwap interface on Akash DeCloud deployment guide
![title picture](https://github.com/yuravorobei/thorchain-bepswap-on-akash-guide/blob/main/media/main.png)

  In this guide for the "Akashian Challenge: Phase 3 Start Guide" I'll show you how to deploy the [Thorchain BEPSwap interface](https://chaosnet.bepswap.com/) on the [Akash Decentralized Cloud (DeCloud)](https://akash.network/).  
  **BEPSwap** is UniSwap for BinanceChain. It will be the first go-to market product for THORChain and makes some compromises as to infrastructure and trustlessness. It will only swap BNB and BEP2 assets on Binance Chain using a second layer protocol that moves assets around on BNB accounts.  
  **Akash** is a marketplace for cloud compute resources which is designed to reduce waste, thereby cutting costs for consumers and increasing revenue for providers. Akash DeCloud is a faster, better, and lower cost cloud built for DeFi, decentralized projects, and high growth companies, providing unprecedented scale, flexibility, and price performance. 10x lower in cost, Akash serverless computing platform is compatible with all cloud providers and all applications that run on the cloud.

This guide is divided into three main parts: dockerizing, installing and preparing Akash, deploying. All actions I perform on a Ubuntu 18.04 operating system. And so we go.

# 1. Dockerizing
The Akash works with a Docker images, so the first thing we need to do to deploy is to get the Docker image of the Thorchain BEPSwap interface application. If you already have the image or know how to get it, you can skip this step.


### 1.1 Source code
First, you need to get the source code of the Thorchain BEPSwap interface application on your computer. To do this, clone the [BEPSwap interface GitHub repository](https://github.com/thorchain/bepswap-web-ui):
  ```bash
  $ git clone https://github.com/thorchain/bepswap-web-ui.git
  $ cd bepswap-web-ui
  ```

### 1.2 Dockerfile
Docker can build images automatically by reading the instructions from a Dockerfile. A Dockerfile is a text document that contains all the commands a user could call on the command line to assemble an image. 
The developers have provided a ready-made Dockerfile that builds the project and then deploy it to a web server as static resources on the Nginx web server.
But personally, I am gettiing errors when building the image using this Dockerfile. So I edited a few lines in the file a bit and now it looks like this:
  ```dockerfile
  # build environment
  FROM node:12.2.0-alpine as build
  WORKDIR /app
  ENV PATH /app/node_modules/.bin:$PATH

  RUN apk update && apk add git
  COPY package.json /app/package.json
  COPY yarn.lock /app/yarn.lock
  RUN npm config set unsafe-perm true
  RUN yarn install
  COPY . /app
  RUN yarn build

  # docker environment
  FROM nginx:1.16.0-alpine
  COPY --from=build /app/build /usr/share/nginx/html
  RUN rm /etc/nginx/conf.d/default.conf
  COPY etc/nginx/nginx.conf /etc/nginx/conf.d
  EXPOSE 80
  CMD ["nginx", "-g", "daemon off;"]
  ```

In the original Dockerfile, I replaced the command `RUN npm install --silent` with` RUN yarn install` and `RUN npm run build` with` RUN yarn build`. I also removed one command `RUN npm install react-scripts@3.0.1 -g --silent`. Ok, now the image should build without errors.


### 1.3 Building the Docker image
To build the Docker image run the following command:
  ```bash
  $ docker build -t thorchain-bepswap .
  ```
Any other name can be used here instead of `thorchain-bepswap`, most importantly, do not forget to also change this name in the following commands. Waiting for the completion of the function.


### 1.4 Publishing the image
To transfer the created image to the cloud server, you need to share it. To do this I will push the image to the [Docker Hub](https://hub.docker.com/). Log into the Docker Hub from the command line. Enter the following command, replace `<your_hub_username>` with your Docker Hub usesrname:
  ```bash
  $ docker login --username=<your_hub_username>
  ```
And enter your Docker Hub password. 

Then find the created image ID using `docker images` command:
  ```
  $ docker images

  REPOSITORY                      TAG                 IMAGE ID            CREATED             SIZE
  thorchain-bepswap               latest              c8aaeec58faa        12 minutes ago      44.9MB
  ```
The ID of my image is `c8aaeec58faa`.  
Then tag your image with the following command, replace `<image_id>` with your image ID and replace `<your_hub_username>` with your Docker Hub usesrname. Any name can be used instead of `thorchain-bepswap`:
  ```bash
  $ docker tag <image_id> <your_hub_username>/thorchain-bepswap
  ```
Push your image to the Docker Hub repository with the following command:
  ```bash
  $ docker push <your_hub_username>/thorchain-bepswap
  ```
Waiting for loading.
Dockerization done!



# 2. Installing and preparing Akash
To interact with the Akash system, you need to install it and create a wallet. I will follow the official [Akash Documentation guides](https://docs.akash.network/v/master/).

### 2.1 Choosing a Network
At any given time, there are a number of different active Akash networks running, each with a different akash version, chain-id, seed hosts, etc… Three networks  are currently available: mainnet, testnet and edgenet. In this guide for the "Akashian Challenge: Phase 3 Start Guide" i'll use the `edgenet` network.

So let’s define the required shell variables:
  ```bash
  $ AKASH_NET="https://raw.githubusercontent.com/ovrclk/net/master/edgenet"
  $ AKASH_VERSION="$(curl -s "$AKASH_NET/version.txt")"
  $ AKASH_CHAIN_ID="$(curl -s "$AKASH_NET/chain-id.txt")"
  $ AKASH_NODE="$(curl -s "$AKASH_NET/rpc-nodes.txt" | shuf -n 1)"
  ```

### 2.2 Installing the Akash client
There are several ways to install Akash: from the [release page](https://github.com/ovrclk/akash/releases), from the source or via `godownloader`. As for me, the easiest and the most convenient way is to download via `godownloader`:
  ```bash
  $ curl https://raw.githubusercontent.com/ovrclk/akash/master/godownloader.sh | sh -s -- "$AKASH_VERSION"
  ```
And add the akash binaries to your shell PATH. Keep in mind that this method will only work in this terminal instance and after restarting your computer or terminal this PATH addition will be removed. You can read how to set your PATH permanently [on this page](https://stackoverflow.com/questions/14637979/how-to-permanently-set-path-on-linux-unix) if you want:
  ```bash
  $ export PATH="$PATH:./bin"
  ```


### 2.3 Wallet setting
Let's define additional shell variables. `KEY_NAME` value of your choice, I uses a value of “crypto”:
  ```bash
  $ KEY_NAME="crypto"
  $ KEYRING_BACKEND="test"
  ```
Derive a new key locally:
  ```bash
  $ akash --keyring-backend "$KEYRING_BACKEND" keys add "$KEY_NAME"
  ```

You'll see a response similar to below:
  ```yaml
  - name: crypto
    type: local
    address: akash1kfd3adu7sgcu2vxd3ucc3pehmrautp25kxenn5
    pubkey: akashpub1addwnpepqvc7x8r2dyula25ucxtawrt39henydttzddvrw6xz5gvcee4arfp7ppe6md
    mnemonic: ""
    threshold: 0
    pubkeys: []


  **Important** write this mnemonic phrase in a safe place.
  It is the only way to recover your account if you ever forget your password.

  share mammal walnut direct plug drive cruise real pencil random regret chunk use live entire gloom donate require autumn bid brown impact scrap slab
  ```
In the above example, your new Akash address is `akash1kfd3adu7sgcu2vxd3ucc3pehmrautp25kxenn5`, define a new shell variable with it:
  ```bash
  $ ACCOUNT_ADDRESS="akash1kfd3adu7sgcu2vxd3ucc3pehmrautp25kxenn5"
  ```

### 2.4 Funding your account
In this guide I use the edgenet Akash network. Non-mainnet networks will often times have a "faucet" running - a server that will send tokens to your account. You can see the faucet url by running:
  ```bash
  $ curl "$AKASH_NET/faucet-url.txt"

  https://akash.vitwit.com/faucet
  ```
Go to the resulting URL and enter your account address; you should see tokens in your account shortly.
Check your account balance with:
  ```yaml
  $ akash --node "$AKASH_NODE" query bank balances "$ACCOUNT_ADDRESS"

  balances: 
  - amount: "100000000" 
   denom: uakt 
  pagination: 
   next_key: null 
   total: "0"
  ```
As you can see, the balance is non-zero and contains 100000000 uakt.
Okay, now you're ready to deploy the application.



# 3. Deploying
Make sure you have the following set of variables defined on your shell in the previous step "2. Installing and preparing Akash":
  * `AKASH_CHAIN_ID` - Chain ID of the Akash network connecting to
  * `AKASH_NODE` - Akash network configuration base URL
  * `KEY_NAME` - The name of the key you will be deploying from
  * `KEYRING_BACKEND` - Keyring backend to use for local keys
  * `ACCOUNT_ADDRESS` - The address of your account

### 3.1 Creating a deployment configuration
For configuration in Akash uses [Stack Definition Language (SDL)](https://docs.akash.network/v/master/documentation/sdl) similar to Docker Compose files. Deployment services, datacenters, pricing, etc.. are described by a YAML configuration file. These files may end in .yml or .yaml.
Create `deploy.yml` configuration file:
  ```bash
  $ nano deploy.yml
  ```
With following content:  

  ```yaml
  version: "2.0"

  services:
    web:
      image: yuravorobei/thorchain-bepswap
      expose:
        - port: 80
          as: 80
          to:
            - global: true

  profiles:
    compute:
      web:
        resources:
          cpu:
            units: 0.1
          memory:
            size: 512Mi
          storage:
            size: 512Mi
    placement:
      westcoast:    
        pricing:
          web: 
            denom: uakt
            amount: 5000

  deployment:
    web:
      westcoast:
        profile: web
        count: 1
  ```
First, a service named `web` is created in which its settings are specified.
The `services.web.images` field is the name of your published Docker image. In my case it is `yuravorobei/thorchain-bepswap`.
The `services.expose.port` field means the container port to expose. I have specified listening port `80` in my Nginx settings.
The `services.expose.as` field is the port number to expose the container port as specified. Port `80` is the standard port for web servers HTTP protocol.  

Next section `profiles.compute` is map of named compute profiles. Each profile specifies compute resources to be leased for each service instance uses uses the profile. This defines a profile named `web` having resource requirements of 0.1 vCPUs, 512 megabytes of RAM memory, and 512 megabytes of storage space available. `cpu.units` represent a virtual CPU share and can be fractional.

`profiles.placement` is map of named datacenter profiles. Each profile specifies required datacenter attributes and pricing configuration for each compute profile that will be used within the datacenter. This defines a profile named `westcoast` having required attribute `pricing` with a max price for the `web` compute profile of 5000 uakt per block. 

The `deployment` section defines how to deploy the services. It is a mapping of service name to deployment configuration.
Each service to be deployed has an entry in the deployment. This entry is maps datacenter profiles to compute profiles to create a final desired configuration for the resources required for the service.


### 3.2 Creating the deployment
In this step, you post your deployment, the Akash marketplace matches you with a provider via auction. To deploy on Akash, run:
  ```bash
  $ akash tx deployment create deploy.yml --from $KEY_NAME \
    --node $AKASH_NODE --chain-id $AKASH_CHAIN_ID \
    --keyring-backend $KEYRING_BACKEND -y
  ```
Make sure there are no errors in the command response. The error information will be in the `raw_log` responce field.


### 3.3 Wait for your lease
You can check the status of your lease by running:
  ```
  $ akash query market lease list --owner $ACCOUNT_ADDRESS \
    --node $AKASH_NODE --state active
  ```
  You should see a response similar to:
  ```yaml
  leases:
  - lease_id:
      dseq: "200394"
      gseq: 1
      oseq: 1
      owner: akash1kfd3adu7sgcu2vxd3ucc3pehmrautp25kxenn5
      provider: akash1uu8wfvxscqt7ax89hjkxral0r2k73c6ee97dzn
    price:
      amount: "875"
      denom: uakt
    state: active
  pagination:
    next_key: null
    total: "0"
  ```

From this response we can extract some new required for future referencing shell variables:
  ```bash
  $ PROVIDER="akash1uu8wfvxscqt7ax89hjkxral0r2k73c6ee97dzn"
  $ DSEQ="200394"
  $ OSEQ="1"
  $ GSEQ="1"
  ```


### 3.4 Uploading manifest
Upload the manifest using the values from above step:
  ```bash
  $ akash provider send-manifest deploy.yml \
    --node $AKASH_NODE --dseq $DSEQ \
    --oseq $OSEQ --gseq $GSEQ \
    --owner $ACCOUNT_ADDRESS --provider $PROVIDER
  ```
Your image is deployed, once you uploaded the manifest. You can retrieve the access details by running the below:
  ```bash
  $ akash provider lease-status \
    --node $AKASH_NODE --dseq $DSEQ \
    --oseq $OSEQ --gseq $GSEQ \
    --provider $PROVIDER --owner $ACCOUNT_ADDRESS
  ```
You should see a response similar to:
  ```json
  {
    "services": {
      "web": {
        "name": "web",
        "available": 1,
        "total": 1,
        "uris": [
          "ekelgiqhg9dt74ea80mho159gk.provider3.akashdev.net"
        ],
        "observed-generation": 0,
        "replicas": 0,
        "updated-replicas": 0,
        "ready-replicas": 0,
        "available-replicas": 0
      }
    },
    "forwarded-ports": {}
  }
  ```
You can access the application by visiting the hostnames mapped to your deployment. In above example, its http://ekelgiqhg9dt74ea80mho159gk.provider3.akashdev.net. Following the address, make sure that the application works:
![result1](https://github.com/yuravorobei/thorchain-bepswap-on-akash-guide/blob/main/media/deployed.png)


### 3.5 Service Logs
You can view your application logs to debug issues or watch progress using `akash provider service-logs` command, for example:
```
  $ akash provider service-logs --node "$AKASH_NODE" --owner "$ACCOUNT_ADDRESS" \
  --dseq "$DSEQ" --gseq 1 --oseq $OSEQ --provider "$PROVIDER" \
  --service web \
```

You should see a response similar to this:
```
[web-6bf7cd4f76-dwztp] 10.233.90.1 - - [11/Dec/2020:10:27:47 +0000] "GET / HTTP/1.1" 200 5289 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/87.0.4280.66 Safari/537.36" "92.191.40.120"
[web-6bf7cd4f76-dwztp] 10.233.90.1 - - [11/Dec/2020:10:27:48 +0000] "GET /static/css/11.21560ced.chunk.css HTTP/1.1" 200 673824 "http://ekelgiqhg9dt74ea80mho159gk.provider3.akashdev.net/" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/87.0.4280.66 Safari/537.36" "92.191.40.120"
[web-6bf7cd4f76-dwztp] 10.233.90.1 - - [11/Dec/2020:10:27:48 +0000] "GET /static/css/main.a88d6281.chunk.css HTTP/1.1" 200 78 "http://ekelgiqhg9dt74ea80mho159gk.provider3.akashdev.net/" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/87.0.4280.66 Safari/537.36" "92.191.40.120"
[web-6bf7cd4f76-dwztp] 10.233.90.1 - - [11/Dec/2020:10:27:48 +0000] "GET /static/js/main.832f729e.chunk.js HTTP/1.1" 200 97457 "http://ekelgiqhg9dt74ea80mho159gk.provider3.akashdev.net/" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/87.0.4280.66 Safari/537.36" "92.191.40.120"
```

### 3.6 Close your deployment

When you are done with your application, close the deployment. This will deprovision your container and stop the token transfer. Close deployment using deployment by creating a deployment-close transaction:
  ```shell
  $ akash tx deployment close --node $AKASH_NODE \
    --chain-id $AKASH_CHAIN_ID --dseq $DSEQ \
    --owner $ACCOUNT_ADDRESS --from $KEY_NAME \
    --keyring-backend $KEYRING_BACKEND -y
  ```

Additionally, you can also query the market to check if your lease is closed:
  ```bash
  $ akash query market lease list --owner $ACCOUNT_ADDRESS --node $AKASH_NODE
  ```
You should see a response similar to:
  ```yaml
  - lease_id:
      dseq: "200394"
      gseq: 1
      oseq: 1
      owner: akash1kfd3adu7sgcu2vxd3ucc3pehmrautp25kxenn5
      provider: akash1uu8wfvxscqt7ax89hjkxral0r2k73c6ee97dzn
    price:
      amount: "875"
      denom: uakt
    state: closed
  pagination:
    next_key: null
    total: "0"
  ```
As you can notice from the above, you lease will be marked `closed`.

