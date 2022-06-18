---
title: "How to manage all my K8s secrets in git securely with Bitnami Sealed Secrets"
date: 2022-03-27T08:06:25+06:00
description: I can manage all my K8s config in git, except Secrets.
menu:
  sidebar:
    name: Bitnami Sealed Secrets
    identifier: bitnami-sealed-secrets
    weight: 10
tags: ["Basic", "Multi-lingual"]
categories: ["Basic"]
---


# I can manage all my K8s config in git, except Secrets.

### Introduction

The problem is rather clear, I cannot save any secret securely in any Git platform as Github or Gitlab. If the repository is public, it is obvious that anyone could read these secrets. And, if the repository is private, we are storing key valuable information that could compromise our software, platform or company as plain text in a code repository. What if someone gets access to these repositories full of secrets ? What if some “non-ethical group of cybersecurity experts” “accidentally” leaks the content of these repositories to the Internet? The solution to this problem is water clear, do not store any kind of secret in Git,  but where then ?

There are many options that are free to use that can help us to solve this problem. Some are more complex to use, for example Hashicorp Vault. Using vault involves modifying the way you deploy your applications as it needs you to embody a sidecar pod on your Kubernetes deployments, and give this third component extra permissions by the use of a Service Account. 

Another option could be modify the way the application gets the information of those secrets. We could use a password manager, like Onepassword or Bitwarden, where we could save securely all our passwords and confidential information. Then, from the application we could access these secrets by making use of the API that expose the functionality of saving, modifying or retrieving secrets from the password manager. The problem again here is how to store the API key needed for accessing the password manager through its API.

A much simpler option could be hiring a third party platform that does this for us. Every big cloud company like Google Cloud or Amazon Web Services offers a service for storing securely secrets and then retrieving them from a Kubernetes cluster. That is the case of AWS Secrets, for example. 

In this post I’d like to talk a little bit about Bitnami Sealed Secrets, a way of storing secrets in a Git repository securely that I liked since the first time I read about it. 

### What is Bitnami Sealed Secrets ?

This software offers a simple solution for the very same problem stated above: storing securely kubernetes secrets in Git.

Basically there are two main actors in this process: the local client and the cluster controller or operator. The local client is a tool named “kubeseal” and it will be the tool in charge of creating the manifests ready to be stored safely in the repository. 

The other  component, the controller/operator is a tool installed in the cluster and in charge of “transforming” the sealed secret coming from the repository and encrypted into a Kubernetes secret capable of being consumed by an application. 

![Untitled](static/files/diagram.png)

The first thing to know about this tool is how crytographically works, it uses PKI (public key infrastructure) to cipher the content of the local secrets, but how this is used ?

The local client, kubeseal, retrieves the public certificate of the controller installed on the cluster. This may be done manually if you have some curiosity:

```bash
$ kubeseal --fetch-cert > controller-certificate.pem
$ kubeseal --cert controller-certificate.pem
```

This certificate is used then for encrypting the normal secret which is in clear obtaining as result a SealedSecret type Kubernetes manifest with the data encrypted. This encrypted data can only be deciphered with the private key of the controller/operator installed on the target cluster. 

One point to take into account about the encryption process is which parameters are part of it. In a first instance you could think that only the secret is taken into account but no, also the namespace and the name of the secret are taken into account. This way, a sealed secret is scoped to only one namespace of a certain cluster, so it cannot be deciphered from another one. 

This namespace scope if the default, but there are two more options to define the scope of the sealed secret:

- namespace-wide: this mode allows to change the name of the secret after its creating without ending in a decryption error, as the name is not part of the encryption process.
- cluster-wide: in this mode neither the name nor the namespace are taken into account when encrypting the secret, this way the secret can be unsealed in any namespace of the cluster and given any name.

To describe the scope of the secret are two different options: 

- In the CLI tool using scope flag.
    
    ```bash
    $ kubeseal --scope namespace-wide 
    ```
    
- In the original secret with a annotation:
    
    ```bash
    sealedsecrets.bitnami.com/namespace-wide: "true"
    ```
    

Once this has been said, we have a new kubernetes manifest that contains an object of type SealedSecrets containing the plain secret data encrypted with the public certificate of the controller. This manifest is ready to be applied into the cluster to generate a new object. 

When the controller detects a new object of this type, SealedSecret, it uses its private key to decipher the information on it and creates a regular secret that will contain the plain data that we originally had at the beginning ready to use by any application deployed on the cluster. 

This way we have obtained a manifest ready to be securely stored in the Git repository as it cannot be decipher without the private key of the controller, which is the solution to our problem of where or how to store kubernetes secrets. 

### Installation

Let’s see how we can put this into practice. First we need some basic requirements to make this work:

- A Kubernetes cluster. You can use a cloud one or a local distribution like k3s, rancher or Docker desktop.
- 10 minutes at the most.

The first step is installing the kubeseal tool. For this, get the latest release that fits your environment and download it, uncompress it and move it into your $PATH directory :

```bash
$ wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.17.3/kubeseal-0.17.3-darwin-amd64.tar.gz
$ tar -xvf kubeseal-0.17.3-darwin-amd64.tar.gz
$ cp kubeseal /usr/local/bin/
$ kubeseal --version
```

Now we can create SealedSecrets objects, but for this we need the public certificate of the controller so let’s install the controller on the cluster:

The easiest option is to use the Helm chart for the sealed-secrets controller, but this is like a blackbox for a first installation and in my opinion is better to take a look to the manifests we want to install. The needed manifests are in the release also:

```bash
$ kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.17.3/controller.yaml
```

![Captura de pantalla 2022-03-19 a las 11.24.28.png](bitnami-images/Captura_de_pantalla_2022-03-19_a_las_11.24.28.png)

This step will install a lot of thing in the cluster, and if it is a shared cluster these thing may be a problem due to the permissions they require, like creating a ClusterRole and a ClusterRoleBinding. This permissions are necessary for the controller to work properly, it needs to watch new SealedSecrets objects and it needs to create new Secrets objects based on them. 

All the objects have been created on the “kube-system” namespace, but they can be created on another namespace by just modifying the namespace field in the manifest file. 

Once this step is completed you should have something like this:

![Captura de pantalla 2022-03-19 a las 11.30.34.png](bitnami-images/Captura_de_pantalla_2022-03-19_a_las_11.30.34.png)

If you observe the logs you can see a couple of interesting things:

- It search for the private key and finds it in a secret. For security reasons, you could rotate this private key after a period of time without having to reinstall all again the controller.
- The public certificate is directly shown in the logs, so you could retrieve it from there and use it with the kubeseal tool offline or in a automation workflow.
- The controller is serving on the port 8080.

The controller also exposes a REST API that contains the following routes:

| Route | Description |
| --- | --- |
| /healthz | Health check route useful for the readiness and liveness probes and for creating an external probe, for example with blackbox exporter.  |
| /metrics | Endpoint for the Prometheus to retrieve the controller’s metrics.  |
| /v1/verify | Validates a secret. |
| /v1/rotate | Rotates the secret. |
| /v1/cert.pem | Retrieves the public certificate.  |

You can access this endpoint by just doing a port-forward from the pod’s port 8080 to your [localhost:8080](http://localhost:8080) 

```bash
$ kubectl -n kube-system port-forward svc/sealed-secrets-controller 8080:8080
$ curl localhost:8080/healthz
```

Now the controller is fully installed on the cluster and ready to work!

### Creating a secure secret

For this we will use a simple secret example manifest:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-example
data:
  secret: bXlzdXBlcnNlY3JldAo=
```

The secret data is the string “*mysupersecret*” encoded in base64. To convert this into a sealed secret object we use the kubeseal tool like this:

```bash
$ kubeseal --secret-file secret-example.yaml --sealed-secret-file sealed-secret-example.yaml
```

This command generates a SealedSecret object with the secret from above encrypted with the public certificate of the controller:

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  creationTimestamp: null
  name: secret-example
  namespace: default
spec:
  encryptedData:
    secret: AgBiBleS4QuvmycylFBT1n1kj+XICsCe+QJCftDHfW0RPCWn7eDqV2Xo06Tjv//SUyiTUmkiOvUlQSNmV0uAAIGBqSs/nnQDM5o7chdpVZLIyL+0cScec3NsEA5385xjgXjUIOITomYOLSvogO2ybqmi57zB7/yjnL+aaGaaC9HunEo7taQ5nAEquhe3f6C9uU92M8Iue2LFN9ZjDeTb3Ygos9at600NqBe0bb97zB1ygSrHqpJnB/wnyjk7L0P3dzhT3I9UBW+VTtknfVAAVpUV9N+eWGu5wnwqL6zpPeMHAo4P1R1rnCXGVVo0O/yy4kQvflHEoGiCMdA/mVNW5KbGF4MW6nRysGXMEapD9upQdLN9PLz6tjRtA0wyEESEPBk9so3k66mfxpknT+bTHAIKpdmL5CL3Htd4u6BX0zyi3a9YGrStAuI9+27GYwZFfECvAjxZehpVXW5c7q3U2ncvWG89Er0trFvBmWgenBIyp0m3YD9ahqvyxbMYruF2BrqCtbs+yohYfFManN/QOg0tDpH5ZOzE3kNIoqEzZcLntIfnoY9DwbRNIDrAsvwvA9dSiYc/saP8hW9OG9SQpjI9cm2lzzOLnSPSSqaXXWu0mygBjT/tAb/MNCHI9VznyHXPOXg9HdUS4UOWY6mmZFCLDsA88uuvIAGdOIBfOmo8JCgdHj0OAWXBbL3uZagdvBGJEB3OhRvxWmRdjiXf9A==
  template:
    data: null
    metadata:
      creationTimestamp: null
      name: secret-example
      namespace: default
```

And then it can by applied into the cluster:

```bash
$ kubectl apply -f sealed-secret-example.yaml
```

This creates the object in the cluster and will trigger the controller to create an unsealed secret deciphering it with its private key and creating a normal base64 encoded secret.

![Captura de pantalla 2022-03-19 a las 13.27.05.png](bitnami-images/Captura_de_pantalla_2022-03-19_a_las_13.27.05.png)

![Captura de pantalla 2022-03-19 a las 13.29.50.png](bitnami-images/Captura_de_pantalla_2022-03-19_a_las_13.29.50.png)

This way, we have a sealed secret ready to be stored in the Git repository and the possibility of applying this secret to a specific cluster to create a clear secret ready to be consumed by a pod. 

### Monitoring

As many other tools that are Kubernetes native, Bitnami Sealed Secrets offers the option to scrap some metrics from Prometheus to give the administrator an idea about how the controller is performing. 

This metrics are exposed in the route /metrics of the port 8080, so it is as simple as configuring a Prometheus to scrape the metrics from there and the use a Grafana for creating a dashboard that will inform us about the controller status. 

In the Prometheus configuration add the following job to scrape the controller metrics. In my case I am running the controller in a local Kubernetes cluster whereas the Prometheus server is running as a Docker container, that is why I use the “*host.docker.internal:8080*” address, so adapt that parameter to your own deployment configuration. 

```bash
scrape_configs:
  - job_name: 'bitnami-sealed-secrets'
    scrape_interval: 5s
    static_configs:
      - targets:
          - "host.docker.internal:8080"
```

After that, we can go to the Prometheus dashboard on port 9090 (if running on Docker) and see that the target referring to the sealed secrets controller is up

![Captura de pantalla 2022-03-19 a las 14.09.40.png](bitnami-images/Captura_de_pantalla_2022-03-19_a_las_14.09.40.png)

This will allow us to obtain the metrics directly from the Prometheus server, so we can go to Grafana and configure a new dashboard the will give us a quick overview about the status of the controller.

There is already a dashboard created that Bitnami Sealed Secrets offers ([https://github.com/bitnami-labs/sealed-secrets/blob/main/contrib/prometheus-mixin/dashboards/sealed-secrets-controller.json](https://github.com/bitnami-labs/sealed-secrets/blob/main/contrib/prometheus-mixin/dashboards/sealed-secrets-controller.json)) so is as simple as import this JSON into our Grafana and adjust some configurations:

![Captura de pantalla 2022-03-19 a las 14.14.49.png](bitnami-images/Captura_de_pantalla_2022-03-19_a_las_14.14.49.png)

This dashboard is quite simple and only has to panels, one for the unsealed secrets rate and another one for the errors rate:

![Captura de pantalla 2022-03-19 a las 14.18.50.png](bitnami-images/Captura_de_pantalla_2022-03-19_a_las_14.18.50.png)

If we want to be aware of the possible error that may happen when unsealing a secret we can create an alert that can send us a message to Telegram, Slack, Teams, etc, reporting us about the error. To achieve this add the following expression to the Alert Manager of Prometheus or create another Grafana panel that executes the expression:

```yaml
sum by (reason, namespace) (rate(sealed_secrets_controller_unseal_errors_total{}[5m])) > 0
```

 

### Conclusions

- This tool only helps to store the secret securely in the Git repository, but once it is deployed to the Kubernetes cluster the secret will be in plain text encoded in base 64.
- The Sealed Secrets stored in the repository are only usable for the cluster they were originally targeted for in the moment of their creation. This is because the sealed secrets are created using the public key of the controller installed on the targeted cluster.
- It can be helpful to generate automatically the secrets and convert them into sealed secrets ready to be stored in a repository.
- Installing the controller in a shared cluster may be a bit problematic due to the ClusterRole permissions needed. A customize installation would be needed in this case if the cluster admin does not want to install it at a cluster scope.