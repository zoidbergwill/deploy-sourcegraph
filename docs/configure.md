# Configure

Common customizations:

- [Configure network access](#Configure-network-access)
- [Update site configuration](#Update-site-configuration)
- [Configure TLS/SSL](#Configure-TLS%2FSSL)
- [Configure repository cloning via SSH](#Configure-repository-cloning-via-SSH)
- [Configure language servers](#Configure-language-servers)

Other customizations:

- [Install without RBAC](#Install-without-RBAC)
- [Configure a storage class](#Configure-a-storage-class)
- [Configure gitserver replica count](#Configure-gitserver-replica-count)
- [Configure Lightstep tracing](#Configure-lightstep-tracing)
- [Configure custom redis](#Configure-custom-redis)
- [Configure SSDs to boost performance](#Configure-SSDs-to-boost-performance)
- [Assign resource-hungry pods to larger nodes](#Assign-resource-hungry-pods-to-larger-nodes)

## Configure network access

You need to make the main web server accessible over the network to external users.

There are a few approaches, but using a load balancer is recommended.

### Load balancer (recommended)

For production environments, we recommend using a [load balancer](https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/).

- HTTP
  ```
  kubectl expose deployment sourcegraph-frontend --type=LoadBalancer --name=sourcegraph-frontend-loadbalancer --port=80 --target-port=3080
  ```
- HTTPS (requires you to [configure TLS/SSL](#Configure-TLS%2FSSL))
  ```
  kubectl expose deployment sourcegraph-frontend --type=LoadBalancer --name=sourcegraph-frontend-loadbalancer --port=443 --target-port=3443
  ```

Once the load balancer has acquired an external IP address, you should be able to access Sourcegraph using that. You can check the external IP address by running the following command:

```bash
kubectl get service sourcegraph-frontend-loadbalancer -o=custom-columns=EXTERNAL-IP:.status.loadBalancer.ingress[*].ip
```

### Ingress controller

You can also potentially use an [Ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress/). We haven't tested this.

### Network rule

You can expose Kubernetes nodes directly to avoid provisioning/paying for a load balancer (but honestly you probably want a load balancer).

Add a network rule that allows ingress traffic to port 30080 (HTTP) and/or 30081 (HTTPS) on at least one node.

- [Google Cloud Platform Firewall rules](https://cloud.google.com/compute/docs/vpc/using-firewalls).

  1. Expose the necessary ports.

     ```bash
     gcloud compute --project=$PROJECT firewall-rules create sourcegraph-frontend-http --direction=INGRESS --priority=1000 --network=default --action=ALLOW --rules=tcp:30080
     gcloud compute --project=$PROJECT firewall-rules create sourcegraph-frontend-https --direction=INGRESS --priority=1000 --network=default --action=ALLOW --rules=tcp:30081
     ```

  2. Find a node name.

     ```bash
     kubectl get pods -l app=sourcegraph-frontend -o=custom-columns=NODE:.spec.nodeName
     ```

  3. Get the EXTERNAL-IP address (will be ephemeral unless you [make it static](https://cloud.google.com/compute/docs/ip-addresses/reserve-static-external-ip-address#promote_ephemeral_ip)).
     ```bash
     kubectl get node $NODE -o wide
     ```

* [AWS Security Group rules](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_SecurityGroups.html).

Sourcegraph should then be accessible at `$EXTERNAL_ADDR:30080` and/or `$EXTERNAL_ADDR:30081`, where `$EXTERNAL_ADDR` is the address of _any_ node in the cluster.

## Update site configuration

The site configuration is stored inside a [ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#add-configmap-data-to-a-volume), which is mounted inside every deployment that needs it.

Updates to the site configuration are [eventually propogated](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#mounted-configmaps-are-updated-automatically) to all services, but it take on the order of 1 minute. [Future Kubernetes versions may improve this behavior](https://github.com/kubernetes/kubernetes/pull/64752).

To have site configuration changes take effect immediately, you need to restart the relevant pods.

Recommended steps:

1. Change the name of the ConfigMap in all deployments.

   ```bash
   #!/bin/bash

   # e.g. 2018-08-15t23-42-08z
   CONFIG_DATE=$(date -u +"%Y-%m-%dt%H-%M-%Sz")

   # update all references to the site config's ConfigMap
   # from: 'config-file.*' , to:' config-file-$CONFIG_DATE'
   find . -name "*yaml" -exec sed -i.sedibak -e "s/name: config-file.*/name: config-file-$CONFIG_DATE/g" {} +

   # delete sed's backup files
   find . -name "*.sedibak" -delete
   ```

2. Apply the new configuration to your Kubernetes cluster.

   ```bash
   kubectl apply --prune -l deploy=sourcegraph -f base --recursive

   # The Go language server also reads from the site configuration.
   # If you have it enabled, you'll also need to apply the changes to its deployment:
   # kubectl apply -f configure/xlang/go/ --recursive
   ```

## Configure TLS/SSL

If you intend to make your Sourcegraph instance accessible on the Internet or another untrusted network, you should use TLS so that all traffic will be served over HTTPS. You can configure Sourcegraph to use TLS by providing the `TLS_CERT` and `TLS_KEY` environment variables variables to the `sourcegraph-frontend` deployment.

### Steps

1. Create a [secret](https://kubernetes.io/docs/concepts/configuration/secret/#using-secrets-as-environment-variables) that contains your base64 encoded contents of your certificate and private key.

   ```yaml
   # tls.Secret.yaml
   apiVersion: v1
   data:
     cert: "" # base64 certificate
     key: "" # base64 private key
   kind: Secret
   metadata:
     name: tls
   type: Opaque
   ```

2. Add `TLS_CERT` and `TLS_KEY` environment variables to `sourcegraph-frontend.Deployment.yaml`.

   ```yaml
   # base/frontend/sourcegraph-frontend.Deployment.yaml
   env:
     - name: TLS_CERT
       valueFrom:
         secretKeyRef:
           key: cert
           name: tls
     - name: TLS_KEY
       valueFrom:
         secretKeyRef:
           key: cert
           name: tls
   ```

   ```bash
   FE=base/frontend/sourcegraph-frontend.Deployment.yaml
   cat $FE | yj | jq '(.spec.template.spec.containers[] | select(.name == "frontend") | .env) += [{name: "TLS_CERT", valueFrom: {secretKeyRef: {key: "cert", name: "tls"}}}, {name: "TLS_KEY", valueFrom: {secretKeyRef: {key: "key", name: "tls"}}}]' | jy -o $FE
   ```

3. Change your `appURL` in the site configuration stored in `base/config-file.ConfigMap.yaml`.

   ```json
   {
     "appURL": "https://example.com:3443" // Must begin with "https"; replace with the public IP or hostname of your machine
   }
   ```

_You'll need to [update the site configuration](#update-site-configuration) so that all your deployments will see the updated site configuration._

## Configure repository cloning via SSH

Sourcegraph will clone repositories using SSH credentials if they are mounted at `/root/.ssh` in the `gitserver` deployment.

### Steps

1. [Create a secrets object](https://kubernetes.io/docs/concepts/configuration/secret/#using-secrets-as-environment-variables) that contains your SSH credentials.

   ```yaml
   # gitserver-ssh.Secret.yaml
   apiVersion: v1
   data:
     id_rsa: "-----BEGIN RSA PRIVATE KEY-----\nMII..."
     known_hosts: "github.com,192.30.255.113 ssh-rsa AAAA..."
   kind: Secret
   metadata:
     name: gitserver-ssh
   type: Opaque
   ```

2. Refer to the secret inside your `gitserver` deployment by adding a `volume` and `volumeMount`.

   ```yaml
   # base/gitserver/gitserver.StatefulSet.yaml
   spec:
     containers:
       volumeMounts:
         - mountPath: /root/.ssh
           name: ssh
     volumes:
       - name: ssh
         secret:
           defaultMode: 384
           secretName: gitserver-ssh
   ```

## Configure language servers

> Code intelligence is a paid upgrade on top of the Data Center deployment option. After following these instructions to confirm it works, [buy code intelligence](https://about.sourcegraph.com/pricing).

Code intelligence provides advanced code navigation and cross-references for your code on Sourcegraph.

After setting up the base Sourcegraph deployment, refer to the following docs for each language for instructions about how to deploy / configure each language server:

- [Go](../configure/xlang/go/README.md)
- [Java](../configure/xlang/java/README.md)
- [PHP](../configure/xlang/php/README.md)
- [Python](../configure/xlang/python/README.md)
- [Javascript / Typescript](../configure/xlang/typescript/README.md)

## Install without RBAC

Sourcegraph Data Center communicates with the Kubernetes API for service discovery. It also has some janitor DaemonSets that clean up temporary cache data. To do that we need to create RBAC resources.

If using RBAC is not an option, then you will not want to apply `*.Role.yaml` and `*.RoleBinding.yaml` files.

## Configure a storage class

Sourcegraph relies on the default storage class of your cluster. If your cluster does not have a default storage class or if you wish to use a different storage class for Sourcegraph, then you need to update all PersistentVolumeClaims with the name of the desired storage class.

```bash
find . -name "*PersistentVolumeClaim.yaml" -exec sh -c "cat {} | yj | jq '.spec.storageClassName = \"$STORAGE_CLASS_NAME\"' | jy -o {}" \;
```

## Configure gitserver replica count

Increasing the `replica` count of the `gitserver` StatefulSet increases the scalability of your deployment. Repository clones are consistently striped across all `giterver` replicas, so other services need to be aware of how many `gitserver` replicas have been specified in order to know how to a resolve an individual repo.

Services that talk to `gitserver` are passed a list of `gitserver` addresses via the `SRC_GIT_SERVERS` environment variable. You'll need to update this environment variable for each deployment if you change `gitserver`'s `replica` count.

1. Get all the deployments which use `SRC_GIT_SERVERS`.

   ```bash
   grep SRC_GIT_SERVERVS -l
   ```

2. For each one of those files, change the value of `SRC_GIT_SERVERS` to be a space separated list of addresses like the following:

   ```bash
   # $REPLICA_COUNT = 1
   gitserver-0.gitserver:3178

   # $REPLICA_COUNT = 2
   gitserver-0.gitserver:3178 gitserver-1.gitserver:3178

   # ...

   # $REPLICA_COUNT = n
   gitserver-0.gitserver:3178 gitserver-1.gitserver:3178 ... gitserver-${n-1}:3178
   ```

## Configure Lightstep tracing

Lightstep is a closed-source distributed tracing and performance monitoring tool created by some of the authors of Dapper. Every Sourcegraph deployment supports Lightstep, and it can be configured via the following environment variables (with example values):

```yaml
env:
  # https://about.sourcegraph.com/docs/config/site/#lightstepproject-string
  - name: LIGHTSTEP_PROJECT
    value: my_project

  # https://about.sourcegraph.com/docs/config/site/#lightstepaccesstoken-string
  - name: LIGHTSTEP_ACCESS_TOKEN
    value: M3WKBuqsCnRYz1c

  # If false, any logs (https://github.com/opentracing/specification/blob/master/specification.md#log-structured-data)
  # from spans will be omitted from the spans sent to Lightstep.
  - name: LIGHTSTEP_INCLUDE_SENSITIVE
    value: true
```

To enable this, you must first purchase Lightstep and create a project corresponding to the Sourcegraph instance. Then, add the above environment to each deployment.

## Configure custom Redis

Sourcegraph supports specifying a custom Redis server for:

- caching information (specified via the `REDIS_CACHE_ENDPOINT` environment variable)
- storing information (session data) (specified via the `REDIS_STORE_ENDPOINT` environment variable)

If you want to specify a custom Redis server, you'll need specify the corresponding environment variable for each of the following deployments:

- `sourcegraph-frontend`
- `indexer`
- `lsp-proxy`
- `repo-updater`
- `xlang-go`
- `xlang-go-bg`

## Configure SSDs to boost performance

See [ssd/README.md](../configure/ssd/README.md).

## Assign resource-hungry pods to larger nodes

If you have a heterogeneous cluster where you need to ensure certain more resource-hungry pods are assigned to more powerful nodes (e.g. `indexedSearch`), you can [specify node constraints](https://kubernetes.io/docs/concepts/configuration/assign-pod-node) (such as `nodeSelector`, etc.).