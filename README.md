# Scalable Jitsi Helm Chart

[Jitsi](https://jitsi.org/jitsi-meet/) on Kubernetes made easy. Scalable jitsi helm chart based on [this architecture](https://github.com/hpi-schul-cloud/jitsi-deployment). 

## TL;DR;

```bash
git clone https://github.com/minhpq331/jitsi-scalable-helm.git

# Make custom values file
cp jitsi-scalable-helm/values.yaml values-custom.yaml

# Seed default password
./jitsi-scalable-helm/scripts/generate_password.sh values-custom.yaml

helm upgrade --install -f values-custom.yaml easyjitsi ./jitsi-scalable-helm
```

## Introduction

This chart bootstraps a scalable [jitsi-meet](https://jitsi.org/jitsi-meet/) deployment with multiple shards, multiple JVB on Kubernetes.

## Presequites

- [Helm v3](https://helm.sh)
- A Kubernetes cluster (>= 1.17 - dont know if it work before this version) with public-accessible nodes for JVB
- Opened firewall rules for JVB's port ranges (for ex: 30000-30xxx + UDP)
  
> JVB NodePort is calculated based on `shard-[n].jvbBasePort` and pod index in statefulset. You can predict those ports like this: with `shard-0.jvbBasePort = 30000`, `shard-0-jvb-0` will expose 30000, `shard-0-jvb-1` will expose 30001,... So in order to make jvb work, you need to open 30000-300xx port on worker nodes with UDP protocol

## Architecture

This architecture is from this repo [https://github.com/hpi-schul-cloud/jitsi-deployment](https://github.com/hpi-schul-cloud/jitsi-deployment). I only made some modifications to fit more general cases.

![Architecture Jitsi Meet](https://github.com/hpi-schul-cloud/jitsi-deployment/blob/master/docs/architecture/build/jitsi_meet_one_shard.png)

This image show only a single shard. A more detailed explanation of the system architecture with multiple shards can be found in [original architecture](https://github.com/hpi-schul-cloud/jitsi-deployment/blob/master/docs/architecture/architecture.md).

This chart is tested with Google GKE and DigitalOcean DOKS cluster version 1.20, 1.21.

## Modifications

The original deployment uses kustomize to deploy all components and requires some manual `copy-paste` stuffs which I don't really like. Their approach is relying on [metacontroller](https://github.com/metacontroller/metacontroller) to provision a Service per JVB's pod and expose that service with NodePort. I think their architecture is awesome but they're lack of documentation on metacontroller's part so a lot of people stuck with that part.

My approach is simpler: **Use Helm to provision (if you disable autoscaling) or pre-provision (with autoscaling enabled) all JVB's services and ingresses** (for media traffic and colibri websockets). Of course you can bring your own implementation and disable mine. Setting some `enabled` flags will do the trick.

Apart from that, there are a few things to note about my chart:

- Use **official [jitsi docker image](https://github.com/jitsi/docker-jitsi-meet)** and **pass configurations through env** (I didn't make helm-specific values except some xmpp configurations to communicate between components). You can use your familiar `docker-compose .env` and it will work.
- Built-in ability to **inject custom files to prosody, web and jvb components**. I uses this a lot to inject custom plugins, custom webpages, modify default configs,...
- **Separate deployment between controller components (prosody, jicofo, haproxy, web) and data components (jvb, jibri) is supported**. 
For example: Separated cluster for jvb, jibri,... You can choose components to install on each cluster. Or JVB on VMs with docker can join this cluster with prosody 5222 port exposed.
- **Easiest and fastest way** to deploy a scalable jitsi cluster on cloud environments.

Checkout default [values.yaml](values.yaml) for more parameter details.

## Attentions

- Currently this chart don't support some jitsi components like jigasi or jibri. I'm working on making jibri deployment without alsa loopback. (But if you have any idea to make it work, feel free to drop a PR).
- Don't set different `deploymentInfo.shard` or you will be forced to disconnect after a minute because haproxy only mark requests with `?room=` sticky.
