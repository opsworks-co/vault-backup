# vault-backup
## Pre-requirements
### Prepare Vault user
Create policy:
```
vault login
cat <<EOF | vault policy write vault-backup -
path "sys/storage/raft/snapshot" {
  capabilities = ["read"]
}
EOF
```
Create token:
```
vault token create -period=24h -policy=vault-backup
```

### Prepare AWS credentials
The backuper will upload result snapshot to S3 bucket, so it means the tool is required to have AWS CLI credentials with a policy that allows to LIST, GET, PUT objects in the target bucket.

### Create secret with credentials
```sh
kubectl create secret generic vault-backup-secrets --from-literal=AWS_ACCESS_KEY_ID=<AWS_ACCESS_KEY_ID> --from-literal=AWS_SECRET_ACCESS_KEY=<AWS_SECRET_ACCESS_KEY> --from-literal=VAULT_TOKEN=<VAULT_TOKEN>
```

### Installing chart
```sh
helm upgrade --install vault-backup ./infra/vault/vault-backup
```

## Restoration process

The following prerequisite steps and knowledge are required in order to backup a Vault cluster. All of the following are required to understand or carry out before attempting to backup or restore Vault.
Vault cluster configuration is defined: Vault with raft HA method is in place.
A cluster configuration as defined in either our Vault with Integrated Storage
Vault has been initialised: This SOP assumes you have already initialised Vault, keyholders are available with access to the unseal keys for each, that you have access to tokens with sufficient privileges for both clusters and encrypted data is stored in the storage backend.


### Steps

Bring your Vault cluster back online following the circumstances that required you to restore from backup.
You will need to reinitialise your Vault cluster and log in with the new root token that was generated during its reinitialisation.
Note that these will be temporary- the original unseal keys will be needed following restore.

Copy your Vault Raft Snapshot file onto a Vault cluster member and run the below command, replacing the filename with that of your snapshot file.


**Download snapshot from AWS S3**

```
s3://<BUCKET_NAME>/vault/vault_<CLIENT_NAME>_20230220_120439.snap
```

**Copy file to the fresh clean vault.**
```
kubectl cp vault_<CLIENT_NAME>_20230220_120439.snap vault-0:/tmp/ -n <namespace>
```

**Go to vault shell**
```
kubectl exec -it vault-0 -n <namespace> sh
```

**Login to Vault**
```
vault login <fresh_root_token>
```

**Restore the snapshot**
```
vault operator raft snapshot restore -force /tmp/vault_<CLIENT_NAME>_20230220_120439.snap
```

**Restart containers and tryout to get into vault be re-login with old root token.**

**Note!!!**

Vault-NG was initialized with AWS KMS Autounseal method - all AWS_ environment variables must be in place while backup restoration is in place.

## Decrypt the snapshot
```sh
gpg --batch --yes --passphrase ${ENCRYPTION_PASSWORD} -d --output decrypted.txt  encrypted.txt
```