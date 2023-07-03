## BANK-VAULTS 

**Following to the bank-vaults docs:**
https://bank-vaults.dev/docs/installing/#deploy-operator

### Deploy a local Vault operator:

**1. Install the Bank-Vaults operator:**
    
    helm repo add banzaicloud-stable https://kubernetes-charts.banzaicloud.com
    helm upgrade --install vault-operator banzaicloud-stable/vault-operator

**2. Create a Vault instance using the Vault custom resources:**
This will create a Kubernetes CustomResource called vault and a PersistentVolumeClaim for it. apply those to cluster:

    https://github.com/bank-vaults/vault-operator/blob/main/deploy/rbac.yaml
    https://github.com/bank-vaults/vault-operator/blob/main/deploy/cr.yaml
	

**3. Configure your Vault client to access the Vault instance running in the vault-0 pod:**

  **a. exec into the vault pod** (install kubectl)

	kubectl exec -it vault-0 -- sh

  **b. Set the address of the Vault instance:**

	export VAULT_ADDR=https://127.0.0.1:8200

  **c. Import the CA certificate of the Vault instance by running the following commands:**

    kubectl get secret vault-tls -o jsonpath="{.data.ca\.crt}" | base64 -d > $PWD/vault-ca.crt
    export VAULT_CACERT=$PWD/vault-ca.crt

  **d. Check that you can access the vault:**

		vault status

**4. To authenticate to Vault, you can access its root token by running:**

      export VAULT_TOKEN=$(kubectl get secrets vault-unseal-keys -o jsonpath={.data.vault-root} | base64 -d)
    
**5. Now you can interact with Vault. add a secret by running :**

      vault kv put secret/demosecret/mysql-secret MYSQL_ROOT_PASSWORD=test12345

**6. we can port forward the pod: vault-0 and get into the vault ui, using the root token**

    (to reveal the token, run: echo $VAULT_TOKEN).
    
	
 ### Deploy the mutating webhook 

**1. Create a namespace for the webhook and add a label to the namespace, for example, vault-infra:**
    
      kubectl create namespace vault-infra
      kubectl label namespace vault-infra name=vault-infra

**2. Deploy the vault-secrets-webhook chart:** 

	helm repo add banzaicloud-stable https://kubernetes-charts.banzaicloud.com
  	helm upgrade --namespace vault-infra --install vault-secrets-webhook banzaicloud-	stable/vault-secrets-webhook

**3. add the following role and policy for the kubernetes access to vault, Attention to the SA is default.**

    vault write auth/kubernetes/role/default \
      bound_service_account_names=default \
      bound_service_account_namespaces=default \
      policies=default \
      ttl=1h

    -----

    cat <<EOF > /home/vault/app-policy.hcl
    path "secret/data/demosecret/aws" {
      capabilities = ["read"]
    }
    EOF
    vault policy write default /home/vault/app-policy.hcl

**4. deploy mysql-deployment to the cluster.**

	kubectl apply -f mysql-deployment.yaml
