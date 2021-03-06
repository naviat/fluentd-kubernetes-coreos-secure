# fluentd-kubernetes-coreos-secure

[![Build Status](https://travis-ci.org/cmachler/fluentd-kubernetes-coreos-secure.svg?branch=master)](https://travis-ci.org/cmachler/fluentd-kubernetes-coreos-secure)
[![](https://images.microbadger.com/badges/version/evergreenitco/fluentd-kubernetes-coreos-secure.svg)](http://microbadger.com/images/evergreenitco/fluentd-kubernetes-coreos-secure "Get your own version badge on microbadger.com") [![](https://images.microbadger.com/badges/image/evergreenitco/fluentd-kubernetes-coreos-secure.svg)](http://microbadger.com/images/evergreenitco/fluentd-kubernetes-coreos-secure "Get your own image badge on microbadger.com")

Docker Container with Fluentd that will capture logs from containers running on Kubernetes, and collect CoreOS logs from systemd. This image is also using the secure forward plugin to send logs to td-agent running on Elasticsearch (EFK) stack.

## Tags:

* `latest`     - using fluentd 0.14.x just basic ingesting of CoreOS and Kubernetes logs forwarding to EFK stack.
* `nginx`      - same basic functions as latest but parses nginx logs from pods whose name's begin with "nginx". You also need to flatten the JSON hash after the burrow parse for Elasticsearch to ingest, see td-agent.conf on nginx branch for reference.
* `dev`        - development environment for latest image.
* `nginx-dev`  - development environment for nginx image.


Environment variables used by the image:

* `EFK_HOST`   - the IP address for the td-agent running on the EFK server.
* `EFK_PORT`   - the port for the td-agent is listening on the EFK server.
* `EFK_SECRET` - the shared secret between the fluentd agent and server (in Kubernetes setting it as a secret and obtaining as environment variable. Make sure to base64 encode the string.).
* `SELF_HOSTNAME` - put whatever value you like (didn't seem to matter but required by the fluentd secure forward plugin).


### Required to mount the ca_cert.pem file from the server to "/etc/fluent_cert/ca-cert.pem".
I am making this file in Kubernetes a secret and then mounting it to the pod as "/etc/fluent_cert/ca-cert.pem" Make sure to base64 encode the pem file contents, then you can post to the Kubernetes API the pem file as a secret. See below for posting secret to Kubernetes API.

### Corresponding td-agent.conf running on EFK server attached to the GitHub repository.

### How to POST secret ca-cert.pem file to Kubernetes:

1. [base64 encode contents of pem file](https://linux.die.net/man/1/base64)
2. Create Kubernetes secret template file to be called by cURL:

    ```
    {"kind":"Secret","apiVersion":"v1","metadata":{"name":"fluentd-ca-cert-secret","creationTimestamp":null},"data":{"ca-cert.pem":"base64 output"}}
    ```
3. POST to Kubernetes API the template JSON file created above:

    ```
    curl -H "Content-Type: application/json" -XPOST -d"$(cat /srv/kubernetes/manifests/fluentd-cloud-logging-ca-cert-secret.json)" "http://127.0.0.1:8080/api/v1/namespaces/kube-system/secrets"
    ```

### The following should be added to your worker(s) cloud-config:

```
Environment="RKT_OPTS=--volume var-log,kind=host,source=/var/log --mount volume=var-log,target=/var/log"
ExecStartPre=/usr/bin/mkdir -p /var/log/containers
```

### DaemonSet JSON file Mounting "/var/log" for CoreOS systemd logs, and "/var/lib/docker/containers" for Kubernetes Container logs for fluentd to capture. Also mounting secret volume and creating secret environment variable. This should be used in your cloud-config for your controller(s) [Full example can be found here](https://github.com/cmachler/coreos-kubernetes/blob/master/multi-node/generic/controller-install.sh):

```
{
    "apiVersion": "extensions/v1beta1",
          "kind": "DaemonSet",
          "metadata": {
            "name": "fluentd-kube-coreos",
            "namespace": "kube-system",
            "labels": {
              "k8s-app": "fluentd-logging"
            }
          },
          "spec": {
            "template": {
              "metadata": {
                "name": "fluentd-kube-coreos",
                "namespace": "kube-system",
                "labels": {
                  "k8s-app": "fluentd-logging"
                }
              },
              "spec": {
                "containers": [
                  {
                    "name": "fluentd-kube-coreos",
                    "image": "evergreenitco/fluentd-kubernetes-coreos-secure:latest",
                    "env": [
                      {
                        "name": "EFK_HOST",
                        "value": ""
                      },
                      {
                        "name": "EFK_PORT",
                        "value": "24284"
                      },
                      {
                        "name": "EFK_SECRET",
                        "valueFrom": {
                            "secretKeyRef": {
                                "name": "efk-shared-key",
                                "key": "shared-key"
                            }
                        }
                      },
                      {
                        "name": "SELF_HOSTNAME",
                        "value": ""
                      }
                    ],
                    "resources": {
                      "limits": {
                        "memory": "200Mi"
                      },
                      "requests": {
                        "cpu": "100m",
                        "memory": "200Mi"
                      }
                    },
                    "volumeMounts": [
                      {
                        "name": "varlog",
                        "mountPath": "/var/log"
                      },
                      {
                        "name": "varlibdockercontainers",
                        "mountPath": "/var/lib/docker/containers",
                        "readOnly": true
                      },
                      {
                        "name": "secret-volume",
                        "readOnly": true,
                        "mountPath": "/etc/fluent_cert"
                      }
                    ]
                  }
                ],
                "terminationGracePeriodSeconds": 30,
                "volumes": [
                  {
                    "name": "varlog",
                    "hostPath": {
                      "path": "/var/log"
                    }
                  },
                  {
                    "name": "varlibdockercontainers",
                    "hostPath": {
                      "path": "/var/lib/docker/containers"
                    }
                  },
                  {
                    "name": "secret-volume",
                    "secret": {
                      "secretName": "fluentd-ca-cert-secret"
                    }
                  }
                ]
              }
            }
          }
}
```
