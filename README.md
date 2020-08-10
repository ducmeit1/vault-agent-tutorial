# Vault Agent Kubernetes Tutorial

## Introduction

- [Vault](https://www.vaultproject.io) is a tool for securely accessing secrets. A secret is anything that you want to tightly control access to, such as API keys, passwords, or certificates. Vault provides a unified interface to any secret, while providing tight access control and recording a detailed audit log.

- [Kubernetes](https://kubernetes.io) is an open-source system for automating deployment, scaling, and management of containerized applications.

- Kubernetes won't help you store sensitive data even you think the [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) API that Kubernetes provided is a way to store your data. No, it just helps you encode your data with Base64 without any encryption. Therefore, you could uses Vault to improve this problem.

- Vault supports many authentication methods to authorize for the client to get the sensitive data, Kubernetes was supported as well by uses Service Account's Kubernetes. To help you easily connect to Vault Cluster and helps you manage the policy about getting the secrets for your Kubernetes deployment or pod, we would like to uses [Vault Agent](https://www.vaultproject.io/docs/agent) with injection sidecar method. You can read more at [here](https://www.vaultproject.io/docs/platform/k8s/injector).

In this tutorial, I would like to summarize the steps will helps you installing Vault Agent in your Kubernetes Cluster with an external Vault Cluster.

## Scenarios

- How your Kubernetes Cluster do authentication with your external Vault Cluster and getting the secrets information?
  - Firstly, you need to create a service account is **vault-auth** and use its secrets information to do authentication with Vault by SA's JWT Token and SA's CA Certification and Kubernetes cluster master's IP Address.
  - Secondly, you need to assign a service account which is matching with the service account that was registered inside of the role of the authentication method.
  - Finally, assign the service account name just created to the deployment or pod that you want to get vault's secrets information.

## Prerequisites

This tutorial will requires:

- Installed Docker
- Installed Vault Server
- Installed Kubernetes Cluster
- Installed `kubectl` CLI
- Installed `helm` CLI

## TL;DR

### Installing at Local

#### Vault

- [Installing document](https://learn.hashicorp.com/tutorials/vault/getting-started-install) (Quick start with MacOS by run Homebrew with: `brew install vault`)

- Verifying the installing with: `vault` at the Terminal.

- Starting a dev server by: `vault server -dev`

- Export Vault's Server Address: `export VAULT_ADDR="http://127.0.0.1:8200"` **(this is required)**

- Saving the Root Token and Unseal Key are showed up at the Terminal, for example:

  ```
  ➜ vault server -dev
  ==> Vault server configuration:

              Api Address: http://127.0.0.1:8200
                      Cgo: disabled
          Cluster Address: https://127.0.0.1:8201
                Go Version: go1.14.5
                Listener 1: tcp (addr: "127.0.0.1:8200", cluster address: "127.0.0.1:8201", max_request_duration: "1m30s", max_request_size: "33554432", tls: "disabled")
                Log Level: info
                    Mlock: supported: false, enabled: false
            Recovery Mode: false
                  Storage: inmem
                  Version: Vault v1.5.0
              Version Sha: 340cc2fa263f6cbd2861b41518da8a62c153e2e7+CHANGES

  WARNING! dev mode is enabled! In this mode, Vault runs entirely in-memory
  and starts unsealed with a single unseal key. The root token is already
  authenticated to the CLI, so you can immediately begin using Vault.

  You may need to set the following environment variable:

      $ export VAULT_ADDR='http://127.0.0.1:8200'

  The unseal key and root token are displayed below in case you want to
  seal/unseal the Vault or re-authenticate.

  Unseal Key: NNWdeyT8zpNpV6w5nj+ePF8fsdjQ9ZYRYKG4DYhI+Ds=
  Root Token: s.0wQJDeWrheprBSqxMlZtXEAW

  Development mode should NOT be used in production installations!
  ```

- Export Vault's Root Token: `export VAULT_TOKEN="s.0wQJDeWrheprBSqxMlZtXEAW"`

- Login the UI at [http://localhost:8200/ui](http://localhost:8200/ui) with the Root Token.

#### Kubernetes (Minikube)

- Starting Minikube: `minikube start`
- Setting kubectl context config with Minikube `kubectl config set-context minikube`
- Verifying by: `kubectl get namespaces`
- Installing and accessing to the Kubernetes dashboard by `minikube dashboard`

#### Setup a Key Value secret

- Enable a new secrets engine with KV version 2 (KV2):

  ```shell
  vault secrets enable --path=internal kv-v2
  ``` 

- Put secrets data:

  ```shell
  vault kv put internal/database/config username="db-readonly-username" password="db-secret-password"
  ```

  Verifying by:

  ```shell
  vault kv get internal/database/config username
  ```

  For example:

  ```shell
  ====== Metadata ======
  Key              Value
  ---              -----
  created_time     2020-08-10T10:45:26.470948Z
  deletion_time    n/a
  destroyed        false
  version          1

  ====== Data ======
  Key         Value
  ---         -----
  password    db-secret-password
  username    db-readonly-username
  ```

#### Installing Vault Agent

##### Authentication between Kubernetes with Vault

- Create **vault** namespace

  ``` shell
  kubectl create namespace vault
  ```

- Create **vault-auth** service account to authenticate with Vault:

  ```shell
  kubectl apply -f k8s/vault-agent/vault-auth-sa.yaml
  ```

- Enable Kubernetes Authentication Method on Vault Server:
  
  ```shell
  vault auth enable --path=minikube kubernetes
  ```

- Verifying with:
  
  ```shell
  ➜  ~ vault auth list
  Path         Type          Accessor                    Description
  ----         ----          --------                    -----------
  minikube/    kubernetes    auth_kubernetes_7d7034d8    n/a
  token/       token         auth_token_a5a54497         token based credentials
  ```

- Set authentication config for **auth/minikube**:

  - Get JWT Token's **vault-auth** service account:
  
    ```shell
    export VAULT_SA_NAME=$(kubectl get sa vault-auth -n vault -o jsonpath="{.secrets[*]['name']}")
    export SA_JWT_TOKEN=$(kubectl get secret $VAULT_SA_NAME -n vault -o jsonpath="{.data.token}" | base64 --decode; echo)
    ```

  - Get CA Certificate's Kubernetes Cluster:
  
    ```shell
    export SA_CA_CRT=$(kubectl config view --raw --minify --flatten -o jsonpath='{.clusters[].cluster.certificate-authority-data}' | base64 --decode;echo)
    ```

    or

    ```shell
    export SA_CA_CRT=$(kubectl get secret $VAULT_SA_NAME -n vault -o jsonpath="{.data['ca\.crt']}" | base64 --decode; echo)
    ```

  - Get Kubernetes's master host:
  
    ```shell
    export KUBE_HOST=$(kubectl config view --raw --minify --flatten -o jsonpath='{.clusters[].cluster.server}'; echo)
    ``` 

    or
    
    ```shell
    kubectl cluster-info
    ```

  - Write config to authentication method:
  
    ```shell
    vault write auth/minikube/config \
      token_reviewer_jwt="$SA_JWT_TOKEN" \
      kubernetes_host="$KUBE_HOST" \
      kubernetes_ca_cert="$SA_CA_CRT"
    ```

    Verifying with:

    ```shell
    vault read auth/minikube/config
    ```

    Result:

    ```shell
    Key                       Value
    ---                       -----
    disable_iss_validation    false
    issuer                    n/a
    kubernetes_ca_cert        -----BEGIN CERTIFICATE-----
    MIIC5zCCAc+gAwIBAgIBATANBgkqhkiG9w0BAQsFADAVMRMwEQYDVQQDEwptaW5p
    a3ViZUNBMB4XDTE5MTEwNjE3MjE0MFoXDTI5MTEwNDE3MjE0MFowFTETMBEGA1UE
    AxMKbWluaWt1YmVDQTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAL04
    27mxQWGkdEvsfS7I1AYVrDRUxJU4pJfMlYuvOSl8ZiyKxDRqb022mhXXtRGVmjPe
    h8dzADPdm+brFv9DWAm0uPqmyghdpZYb+mT0NzQzNY9Vd+fRnzebHB839qyYK673
    ZrtFeHU5KV1gUuYWq4WLhVnw1g31V+yQxVtl7G6iNbsZxmLWl5qvTFh5nm6welre
    rKeH0UyHpv276ZLlwd6kKIyB4uTXJyJ95iKr0NFUiNF4G/VBuvM6kAj85vRDn6tm
    u9cGWbZoABkDxLy0cYxF0+45A+l0r/Ee45nsBITeAeQtzPFc/6h7I3GBY01JAZf6
    G+CJoFp0F1QKL7X30YsCAwEAAaNCMEAwDgYDVR0PAQH/BAQDAgKkMB0GA1UdJQQW
    MBQGCCsGAQUFBwMCBggrBgEFBQcDATAPBgNVHRMBAf8EBTADAQH/MA0GCSqGSIb3
    DQEBCwUAA4IBAQBtAPeQrWEUyniUjmY6veWTBSFR1QarCeBzJhJTV0OXUgO9KdZN
    iUrlZQ+brCSEX51INoD2pLwVPt32pcWMovi6y3WCwqec8fMfEF9EDLmFxe8bpiLW
    sWY5h3MxumXPyeScVatWcvp+RVlDyVn1jRHrfSAns4YsU9L3LEm3FS+fdrmvhpdN
    Z6YCJE2/HDOFOGnpdh+fpcQEb/Xo+8gwMIQvEJim6wbNNMbuJmMrhUEnK8P5khIV
    AKi2LCLPP7zIE3QYEYs1O/EiF18ytW6SuRNv4qPwyn3SeDuzTHGWnRs69T/lE8Tx
    pheatsHPp7Aee/86RPmAM+cX/OOj60qb6H1F
    -----END CERTIFICATE-----
    kubernetes_host           https://127.0.0.1:32768
    pem_keys                  [
    ```

##### Authentication Kubernetes Pod with Vault

- Create a service account in the namespace that you will deploy your container, for example with `demo` namespace:
  
  ```shell
  kubectl create namespace demo
  kubectl create serviceaccount app
  ```

- Create a new Vault's policy (ACL) to allow read the secrets KV path `internal/database/config` with path `internal/data/base/config` **(*)**:

  ```shell
  vault policy write app - <<EOH
  path "internal/data/database/config" {
    capabilities = ["read"]
  }
  EOH
  ```

  > *In this policy, you are opening access to the data inside of the secret engine `internal` was declared at path `database/config`, therefore, Vault requires you append `/data` to do that.*

- Create a new role `app` for authentication method `minikube`:
  
  ```vault write auth/minikube/role/app \
    bound_service_account_names=app \
    bound_service_account_namespaces=demo \
    policies=app \
    ttl=60s
  ```

  - This role will allow any Kubernetes pod assigned with the service account `app` which is created at `demo` namespace access to the secrets that defined at `app` policy. The access token will be renewed every 60 seconds.

- Verifying:

  - Get JWT Token of `app` service account:

    ```shell
    export APP_SECRET_NAME=$(kubectl get sa app -n demo -o jsonpath="{.secrets[*]['name']}")
    export APP_JWT_TOKEN=$(kubectl get secret $APP_SECRET_NAME -n demo -o jsonpath="{.data.token}" | base64 --decode;echo)
    ```

  - Login to Vault:

    ```shell
    vault write auth/minikube/login \
    role=app \
    jwt="$APP_JWT_TOKEN"
    ```

    Result:

    ```shell
    Key                                       Value
    ---                                       -----
    token                                     s.rJ0sFCCUavPECoqI0IpjWgqY
    token_accessor                            s7yl5gEb90E7yHl539U7e0OL
    token_duration                            1m
    token_renewable                           true
    token_policies                            ["app" "default"]
    identity_policies                         []
    policies                                  ["app" "default"]
    token_meta_role                           app
    token_meta_service_account_name           app
    token_meta_service_account_namespace      demo
    token_meta_service_account_secret_name    app-token-n9ljd
    token_meta_service_account_uid            1956f3ec-b278-4e93-af8e-c9eb7e5a942e
    ```

##### Installing Vault Agent with Helm

- Set kubectl namespace at `vault` with:

  ```shell
  kubectl config set-context --curent --namespace=vault
  ```

- Add Helm repository

  ```shell
  helm repo add hashicorp https://helm.releases.hashicorp.com
  ```

- Get Local IP Address

  ```shell
  export LOCAL_ADDR=$(ipconfig getifaddr en0)
  ```
  

- Installing Vault Agent

  ```shell
  helm install vault hashicorp/vault \
  --set "injector.externalVaultAddr=http://$LOCAL_ADDR:8200" \
  --set "injector.authPath=auth/minikube"
  ```

  Verifying:

  ```shell
  kubectl get pod -n vault
  ```

  Result:

  ```shell
  NAME                                    READY   STATUS    RESTARTS   AGE
  vault-agent-injector-6467f95cc7-jh6jv   1/1     Running   0          48s
  ```

##### Installing Example Deployment

- Create deployment at `demo` namespace with service account `app`

  ```shell
  kubectl apply -f k8s/example/example.yaml --namespace demo
  ```

  Watch pod status every 2s:

  ```shell
  watch kubectl get pod -n demo
  ```

  Result:

  ```shell
  NAME                       READY   STATUS    RESTARTS   AGE
  orgchart-69656bcf6-599jx   1/1     Running   0          33s
  ```

- Patch annotations to inject Vault's secrets to this deployment

  ```shell
  kubectl patch deployment orgchart --patch "$(cat k8s/example/example-path.yaml)" --namespace demo
  ```

- Verifying the secret was injected to primary container
  
  ```shell
  kubectl exec -it orgchart-69656bcf6-599jx -n demo -- cat /vault/secrets/config.txt
  ```

  Result:

  ```shell
  data: map[password:db-secret-password username:db-readonly-username]
  metadata: map[created_time:2020-08-10T15:17:55.200917Z deletion_time: destroyed:false version:1]
  ```

### Installing on GKE (Google Kubernetes Engine)

You can do as the same steps that installing with Minikube on above, however, there are some tips that you should concern.

- You need to open Port TCP:8080 to allow the Mutating Webhook could access to the Vault agent.

- The Vault Server needs to allow access to GKE Master Nodes via TCP:443 as well.

### Cleaning up

```shell
kubectl delete -f k8s/example/example.yaml
kubectl delte -f k8s/vault-agent/vault-auth-sa.yaml
helm delete vault --namespace vault
minikube stop
minikube delete
```