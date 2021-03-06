---
title: "Kubernetes Deployment"
linkTitle: "kubernetes"
weight: 15
description: >
  Deploy and configure the IOP on your cluster.
---

## Prerequisites

- You must have the `sciencemesh` helm repo in your client sources:

```bash
helm repo add sciencemesh https://sciencemesh.github.io/charts/
```

- We'll be deploying a reva daemon using the [standalone example config](https://github.com/cs3org/reva/blob/master/examples/standalone/standalone.toml). We just need to tweak a couple of keys in this config for reva to use the right values:

```bash
wget -q https://raw.githubusercontent.com/cs3org/reva/master/examples/standalone/standalone.toml

# Example edits for the CERN deployment:
$ sed -i '/^\[grpc.services.gateway\]/a datagateway = "https://sciencemesh.cernbox.cern.ch/iop/datagateway"' standalone.toml
$ sed -i '/^\[grpc.services.storageprovider\]/a data_server_url = "https://sciencemesh.cernbox.cern.ch/iop/data"' standalone.toml
```

| key                                                                                                                             | value                                                          |
|---------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------|
| [`grpc.services.gateway.datagateway`](https://reva.link/docs/config/grpc/services/gateway/#datagateway)                         | Set to our externally-accessible Data Gateway (`/datagateway`) |
| [`grpc.services.storageprovider.data_server_url`](https://reva.link/docs/config/grpc/services/storageprovider/#data_server_url) | Points to the external endpoint for the Data Server (`/data`)  |

- All the mesh providers and some dummy users per-provider have been specified in the [ocm-partners example](https://github.com/cs3org/reva/tree/master/examples/ocm-partners). We can fetch these files to later pass them to helm by running:

```bash
wget -q https://raw.githubusercontent.com/cs3org/reva/master/examples/ocm-partners/providers.demo.json
# Get the CERN users, for instance:
wget -q https://raw.githubusercontent.com/cs3org/reva/master/examples/ocm-partners/users-cern.json
```

- To simplify things, we will rely on a pre-deployed [nginx-ingress](https://kubernetes.github.io/ingress-nginx/deploy/) controller. The `nginx.ingress.kubernetes.io/backend-protocol: "GRPC"` annotation can be supplied to expose GRPC services in a very easy way.

## Configuring an IOP deployment

To configure the two ingress resources that expose the IOP endpoints (GRPC and HTTP), we'll just need to pass a few values into a `custom-ingress.yaml` file:

```bash
cat << EOF > custom-ingress.yaml
ingress:
  enabled: true
  services:
    grpc:
      hostname: <hostname>
      path: /
      annotations:
        kubernetes.io/ingress.class: nginx
        nginx.ingress.kubernetes.io/ssl-redirect: "true"
        nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
      tls:
        - secretName: <keypair>
          hosts:
            - <hostname>
    http:
      hostname: <hostname>
      path: /iop(/|$)(.*)
      annotations:
        kubernetes.io/ingress.class: nginx
        nginx.ingress.kubernetes.io/ssl-redirect: "true"
        nginx.ingress.kubernetes.io/rewrite-target: /$2
      tls:
        - secretName: <keypair>
          hosts:
            - <hostname>
EOF
```

- `<hostname>` is the domain you'll use to expose the IOP publicly.
- (Optional) The `<keypair>` is a Kubernetes tls secret created from the `.key` and `.crt` files. It can be omitted together with both `tls` sections to expose the services without TLS-termination. Also note that the secret must be present in the cluster before deploying the IOP:

```bash
kubectl create secret tls <keypair> --key=tls.key --cert=tls.crt
```

Once all this is done, we can carry on with the deployment by running:

```bash
helm install iop sciencemesh/iop \
  --set-file revad.configFiles.revad\\.toml=standalone.toml \
  --set-file revad.configFiles.users\\.json=users-cern.json \
  --set-file revad.configFiles.ocm-providers\\.json=providers.demo.json \
  -f custom-ingress.yaml
```

## Testing the deployment

You can easily test your deployment is reachable outside the cluster by running the `reva` cli and `curl` against the exposed services:

```bash
# Configure the REVA cli client to connect to your GRPC service:
reva configure
host: <hostname>:443
config saved in /.reva.config

# Log-in using any of the users provided in revad.configFiles.users.json
reva login -list
Available login methods:
- basic

reva login basic
username: ishank
password: ishankpass
OK

# HTTP: Query the Prometheus metrics endpoint:
curl https://<hostname>/iop/metrics
```

