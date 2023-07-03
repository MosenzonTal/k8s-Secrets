# K8s Secrets 

#### Project Description:

When you start the MySQL image, you can adjust the configuration of the MySQL instance by passing one or more environment variables:

- MYSQL_ROOT_PASSWORD
- MYSQL_USER
- MYSQL_DATABASE

The idea is to make the mySQL image to use k8s secrets in order to initialize the environment variables which includes senssitive data as user, password etc.

By Default the secrets are stored in ETCD and are not encrypted at rest. In order to achieve that, there are different approaches. in this project ill use the following solutions:

1. Sops and Age
2. Sealed Secrets
3. Hashicorp Vault 

In Addition, in order to place the Secrets in a Repository, we might find a way to store them Encrypted, or find a mechanism to provision secrets to their applications without compromising their confidentiality.

 Also, we would like to implement GipOps approach in which the secrets that are stored in Git would be deployed automatically to the cluster, if there is any change to them's value. We will use ArgoCD for that.

**GitOps** - Storing your YAML manifests in Git and using GitOps tools to sync the manifests to your clusters (For example Flux and ArgoCD!)

In this Project we would have main repo consists 3 Branches, 1 for each solution:

1. Sops
2. Sealed-Secrets
3. Hashicorp-Vault 

each branch would include the relevant resorces to the solution to be implemented.
and we will see how each approach solves the problems mentioned above.

#### Getting Started:

**Connection to mySQL Database:**

    in mysql pod shell run: mysql -p<Password>

#### First Run a container to work in: 

in order to not install the relevant tools on local machine, we can run and use it from inside of a container:

    docker run -it --rm -v ${HOME}:/root/ -v ${PWD}:/work -w /work --net host alpine sh
    
####  Solution 1: Sops & Age

* the configuration file of sops includes only the public key and can be published in git Repo.

**run this command from the root folder of .sops.yaml configuration file:**

    sops -e mysql-secret.yaml > mysql-secret-encrypted.yaml 
    #sops -e --in-place mysql-secret.yaml --> (encrypts the exsiting file)

* To be able to decrypt it, you'll have to have sops-key.txt on your server and the environment variable: SOPS_AGE_KEY_FILE point to it.

**For Manual Decryption:**

    export SOPS_AGE_KEY_FILE=~/Desktop/k8s-secrets/sops/sops-key.txt
    sops -d --in-place /path/to/file


#### Solution 2: Sealed-Secrets

**1. install kubectl + kubeseal.** <br>

***kubectl:*** 

    apk add --no-cache curl
    curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
    chmod +x ./kubectl
    mv ./kubectl /usr/local/bin/kubectl

***kubeseal:***

    curl -L -o /tmp/kubeseal.tar.gz \
    https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.19.1/kubeseal-0.19.1-linux-amd64.tar.gz
    tar -xzf /tmp/kubeseal.tar.gz -C /tmp/
    chmod +x /tmp/kubeseal
    mv /tmp/kubeseal /usr/local/bin/

***Helm:***

    curl -o /tmp/helm.tar.gz -LO https://get.helm.sh/helm-v3.10.1-linux-amd64.tar.gz
    tar -C /tmp/ -zxvf /tmp/helm.tar.gz
    mv /tmp/linux-amd64/helm /usr/local/bin/helm
    chmod +x /usr/local/bin/helm

    * add helm hashicorp repositoriy to our helm reposotries
    helm repo add hashicorp https://helm.releases.hashicorp.com

**2. Install Sealed Secret Controller**


    curl -L -o controller-v0.19.1.yaml  https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.19.1/controller.yaml
    kubectl apply -f controller-v0.19.1.yaml



**3. Create a sealed secret using a YAML file: creating new encrypted sealed secret file of the k8s secret with kubeseal:**

    kubeseal -f ~/Desktop/k8s-secrets/sealed-secrets/mysql-secret.yaml > ~/Desktop/k8s-secrets/sealed-secrets/mysql-sealed-secret.yaml

* when we pass the secret to kubeseal, kubeseal would pass is to the cluster and the sealedSecretController will use its encryption key to encrypt the file.

**4. Deploy the sealed secret** 

    kubectl apply -f mysql-sealed-secret.yaml
	
* then the sealedSecretController will take that sealedSecret and decrypt it and would create a k8s regular secret

**Notes:**
* we can define retention of the contoller to create his new encryption key.
* old keys are saved as secrets in the cluster.
* migrating to new cluster is by migrating the old encryption keys secrets, and then applying the old sealed secrets.
* also we can Re-encrypting secrets with the latest key
<<<<<<< HEAD



#### Solution 3: Hashicorp-Vault

Vault has several different types of storage backends. because Vault has to store Secrets somewhere - it can be either store onto disk using a kubernets volume, or the default backend of vault which is Console Server.

### Storage: Consul
**1. Deploy the Vault Console (UI) by helm:**    

**a. search for the latest version available (1.1.2):**

	helm search repo hashicorp/consul --versions

**b. created consul-values.yaml and pulled the manifests to local:**	
	
    helm template consul hashicorp/consul \
    --namespace vault \
    --version 1.1.2 \
    -f consul-values.yaml \
    > ./manifests/consul.yaml

  **c. create new ns and apply the manifests:
  
 	 kubectl create ns vault
  	 kubectl -n vault apply -f ./manifests/consul.yaml


### TLS End to End Encryption
   Vault communication have to be fully encrypted ened to end.

**a. I created free tls Certificate using cloudflare**

**b. Create the TLS secret:**

    kubectl -n vault create secret tls tls-ca \
    --cert ./tls/ca.pem  \
    --key ./tls/ca-key.pem

    kubectl -n vault create secret tls tls-server \
    --cert ./tls/vault.pem \
    --key ./tls/vault-key.pem

### Valut Deployment

**1. Deploy Vault by helm:**    

**a. search for the latest version available (0.24.1):** 
	
	helm search repo hashicorp/vault --versions 

**  b. created consul-values.yaml with basic vault secret injection configuration: 				(customize the vault as your needs)**  
*		tls enabled, ui on, storage backend (consul) etc..*

**c. grab the manifests:**

		helm template vault hashicorp/vault \
        --namespace vault \
        --version 0.24.1 \
        -f vault-values.yaml \
        > ./manifests/vault.yaml
    
**d. deploy:**

	kubectl -n vault apply -f ./manifests/vault.yaml

### Web UI

Now we can access the web UI 

	kubectl -n vault port-forward svc/vault-ui 443:8200

### Enable Kubernetes Authentication

In order to be able to automatically inject secret from the vault into our k8s pods, we need to enable the kuberenets authentication. This is so that our k8s injector can access the vault.

**1. go to one of out vault pods:**

	kubectl -n vault exec -it vault-0 -- sh 
	vault login
	vault auth enable kubernetes

**creates auth file for kubernetes to authenticate with the vault:**

	vault write auth/kubernetes/config \
	token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
	kubernetes_host=https://${KUBERNETES_PORT_443_TCP_ADDR}:443 \
	kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
	issuer="https://kubernetes.default.svc.cluster.local"
	exit

**So we have a vault, an injector, TLS end to end, stateful storage. The injector can now inject secrets for pods from the vault.**

Now we are ready to use the platform for different types of secrets:

### Inject secrets into pods using the vault injector

In order for us to start using secrets in vault, we need to setup a policy.

**1. Create a role in vault for our app : binds a specific serviceAccount to this role. when we will deploy our pod we will create a service account for that pod which will be allowed to get secrets.**

	kubectl -n vault exec -it vault-0 -- sh 
	
	vault write auth/kubernetes/role/mysql-secret-role \
	   bound_service_account_names=mysql-sa \
	   bound_service_account_namespaces=vault-mysql-app \
	   policies=basic-secret-policy \
	   ttl=1h

**Now we can deploy a secret to the vault:**

	vault kv put secret/basic-secret/mysql-secret root-password=test12345

