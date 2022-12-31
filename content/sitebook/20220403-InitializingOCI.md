---
title: 20220403 - Initializing Oracle Cloud for Tenant ricebucket11
description: Manually setting up the Oracle Cloud Tenant ricebucket1
date: 2022-04-03
toc: true
authors:
  - konri
categories:
  - homelab
draft: false
---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Create a Terraform Service Account](#create-a-terraform-service-account)
- [Setting up Storage for Terraform Remote State Files](#setting-up-storage-for-terraform-remote-state-files)
- [Setting up Vault to store Terraform Secrets](#setting-up-vault-to-store-terraform-secrets)
- [Set up policies for sa-terraform](#set-up-policies-for-sa-terraform)
- [Automation](#automation)

## Prerequisites

After signing up for a free OCI account, I need to manually configure Oracle Cloud accounts. It's best to use a python virtual environment to install `oci-cli`.

```bash
~$ python3 -m venv oci
~$ source oci/bin/activate
(oci) ~$ pip install --upgrade pip
(oci) ~$ pip install oci-cli
```

We need to secure the root account.

  1. Authenticate a session. Login in with the root account's credentials.

     ```bash
     (oci) ~$ oci session authenticate \
                    --region us-ashburn-1 \
                    --profile-name ricebucket1 \
                    --tenancy-name ricebucket1
     ```

     This will generate a profile in `~/.oci/config`

     ```bash
     (oci) ~$ cat ~/.oci/config 
     [ricebucket1]
     fingerprint=<REDACTED>
     key_file=/Users/konri/.oci/sessions/ricebucket1/oci_api_key.pem
     tenancy=<REDACTED>
     region=us-ashburn-1
     security_token_file=/Users/konri/.oci/sessions/ricebucket1/token
     ```

  1. Get the root Compartment ID

     ```bash
     (oci) ~$ export OCI_TENANCY_OCID=$(
                oci --config-file $HOME/.oci/config \
                    --profile ricebucket1 \
                    --auth security_token \
                  iam compartment list --all \
                                       --compartment-id-in-subtree true \
                                       --access-level ACCESSIBLE \
                                       --include-root \
                                       --raw-output \
                                       --query "data[?contains(\"id\",'tenancy')].id | [0]")
     ```

  1. Get the account owner's ID

     ```bash
     (oci) ~$ export OCI_OWNER_OCID=$(
                oci --config-file $HOME/.oci/config \
                    --profile ricebucket1 \
                    --auth security_token \
                  iam user list --raw-output \
                                --query "data[0].id")
     ```

  1. Secure the owner's account with 2 Factor Authentication using Time-based One-time Password (TOTP)

     1. Create the TOTP device

        ```bash
        (oci) ~$ oci --config-file $HOME/.oci/config \
                     --profile ricebucket1 \
                     --auth security_token \
                  iam mfa-totp-device create --user-id $OCI_OWNER_OCID
        ```

        ```bash
        (oci) ~$ export OWNER_TOTP_DEVICE_ID=$(
                   oci --config-file $HOME/.oci/config \
                       --profile ricebucket1 \
                       --auth security_token \
                    iam mfa-totp-device list --user-id $OCI_OWNER_OCID \
                                             --raw-output \
                                             --query 'data[0].id’)
        ```

     1. Generate the TOTP seed

        ```bash
        (oci) ~$ oci --config-file $HOME/.oci/config \
                     --profile ricebucket1 \
                     --auth security_token \
                  iam mfa-totp-device generate-totp-seed \
                     --user-id $OCI_OWNER_OCID \
                     --mfa-totp-device-id $OWNER_TOTP_DEVICE_ID
        ```

        This seed should be stored somewhere safe; losing it will lock the root account.

     1. Activate the TOTP device

        ```bash
        (oci) ~$ oci --config-file $HOME/.oci/config \
                     --profile ricebucket1 \
                     --auth security_token \
                  iam mfa-totp-device activate \
                      --user-id $OCI_OWNER_OCID \
                      --mfa-totp-device-id $OWNER_TOTP_DEVICE_ID
                      --totp-token <token>
        ```

        > :warning: Note-to-self: Need to automatically generate the token via `oathtool`

        Now, logging into the root account will prompt for 2FA token.

## Create a Terraform Service Account

It's not a good idea to use root account to deploy infrastructure, so it's to create a service that has proper permission to deploy in selected compartments. This service account will be called `sa-terraform`.

  1. Create the Service Account

     ```bash
     (oci) ~$ oci --config-file $HOME/.oci/config \
                  --profile ricebucket1 \
                  --auth security_token \
                iam user create --name sa-terraform \
                                --description "Service Account - terraform"
    
     (oci) ~$ export OCI_SA_TERRAFORM_ID=$(
                oci --config-file $HOME/.oci/config \
                    --profile ricebucket1 \
                    --auth security_token \
                  iam user list --raw-output \
                                --query "data[?name == 'sa-terraform'].id | [0]")
     ```

  1. Disable some capabilities for the Service Account

     ```bash
     (oci) ~$ oci --config-file $HOME/.oci/config \
                  --profile ricebucket1 \
                  --auth security_token \
                iam user update-user-capabilities \
                    --user-id $OCI_SA_TERRAFORM_ID \
                    --can-use-console-password False \
                    --can-use-customer-secret-keys False \
                    --can-use-smtp-credentials False
     ```

  1. Generate a RSA key for the Service Account.

     1. Generate the RSA key and save it to a file

        ```bash
        (oci) ~$ openssl genrsa -out ~/.oci/oci_terraform_api.key -aes128 4096
        ```

     1. Generate the public key from the private key

        ```bash
        (oci) ~$ openssl rsa -pubout \
                             -in ~/.oci/oci_ricebucket1_terraform_api.key \
                             -out ~/.oci/oci_ricebucket1_terraform_api.pem
        ```

     1. Upload the public key to Terraform Service Account to allow key Authentication

        ```bash
        (oci) ~$ oci --config-file $HOME/.oci/config \
                     --profile ricebucket1 \
                     --auth security_token \
                  iam user api-key upload \
                      --user-id $OCI_SA_TERRAFORM_ID \
                      --key-file ~/.oci/oci_ricebucket1_terraform_api.pem
        ```

  1. Create a Terraform IAM Group to manage policies for the Terraform Service Account.

     ```bash
     (oci) ~$ oci --config-file $HOME/.oci/config \
                  --profile ricebucket1 \
                  --auth security_token \
                iam group create --name "group-terraform" \
                                 --description "Group - Terraform"

     (oci) ~$ export OCI_GROUP_TERRAFORM_ID=$(
                oci --config-file $HOME/.oci/config \
                    --profile ricebucket1 \
                    --auth security_token \
                  iam group list --raw-output \
                                 --query "data[?name == 'group-terraform'].id | [0]")
     ```

     I don't think it's possible to manage policies on per user bases. If it's possible, then I have no idea how to do it.

  1. Add Terraform Service Account to Terraform Group

     ```bash
     (oci) ~$ oci --config-file $HOME/.oci/config \
                  --profile ricebucket1 \
                  --auth security_token \
                iam group add-user --user-id $OCI_SA_TERRAFORM_ID \
                                   --group-id $OCI_GROUP_TERRAFORM_ID
     ```

We can now use Terraform service account to access Oracle Cloud's resources via `oci-cli` by updating the `~/.oci/config` file.

```bash
(oci) ~$ cat ~/.oci/config
[sa-terraform]
user = ocid1.user.oc1..<REDACTED>
fingerprint = f3:b7:<REDACTED>
tenancy = ocid1.tenancy.oc1..<REDACTED>
region = us-ashburn-1
key_file = /Users/konri/.oci/oci_ricebucket1_terraform_api.key

[ricebucket1]
fingerprint=22:09:<REDACTED>
key_file=/Users/konri/.oci/sessions/ricebucket1/oci_api_key.pem
tenancy=ocid1.tenancy.oc1..<REDACTED>
region=us-ashburn-1
security_token_file=/Users/konri/.oci/sessions/ricebucket1/token
```

- user = `$OCI_SA_TERRAFORM_ID`
- tenancy = `$OCI_TENANCY_OCID`
- fingerprint = `openssl rsa -pubout -outform DER -in ~/.oci/oci_ricebucket1_terraform_api.key | openssl md5 -c`

Instead of `--profile ricebucket1`, `--profile sa-terraform` can be used to get anything the service-account has access to.

## Setting up Storage for Terraform Remote State Files

This Oracle tenant account will be used to store Terraform state files.

  1. Enable `--can-use-customer-secret-keys` for Terraform Service Account

     ```bash
     (oci) ~$ oci --config-file $HOME/.oci/config \
                  --profile ricebucket1 \
                  --auth security_token \
                iam user update-user-capabilities --user-id $OCI_SA_TERRAFORM_ID \
                                                  --can-use-customer-secret-keys True
     ```

     This allows the Service Account to generate Customer Secret Keys to access storage buckets.

  1. Create Customer Secret Key in Terraform Service Account

     ```bash
     (oci) ~$ oci --config-file $HOME/.oci/config \
                  --profile ricebucket1 \
                  --auth security_token \
                iam customer-secret-key create \
                    --user-id $OCI_SA_TERRAFORM_ID \
                    --display-name "terraform-backend"
     ```

     Copy the value of `key` that is displayed as it will not be shown again:

     ```bash
     {
      "data": {
        "display-name": "terraform-backend",
        "id": "<REDACTED>",
        "inactive-status": null,
        "key": "<REDACTED>",
        "lifecycle-state": "ACTIVE",
        "time-created": "2022-04-20T20:04:27.086000+00:00",
        "time-expires": null,
        "user-id": "ocid1.user.oc1..<REDACTED>"
      }
     }
     ```

     Create the following credential file to be used in terraform backend

     ```bash
     (oci) ~$ cat ~/.aws/credentials
     [oci-ricebucket1]
     aws_access_key_id=<REDACTED>              # The ID from data
     aws_secret_access_key=<REDACTED>          # The Key from data
     ```

  1. Create a compartment to store resources that will be used for automation.

     This is not the same compartment that holds all the resources created by Terraform. This compartment will only store the resources used for automation: Vault for secrets, storage for Terraform state files, etc.

     ```bash
     (oci) ~$ oci --config-file $HOME/.oci/config \
                  --profile ricebucket1 \
                  --auth security_token \
                iam compartment create \
                    --compartment-id  $OCI_TENANCY_OCID \
                    --description 'Stores automation resources' \
                    --name cpm-automation

     (oci) ~$ export OCI_CPM_AUTOMATION_ID=$(
                oci --config-file $HOME/.oci/config \
                    --profile ricebucket1 \
                    --auth security_token \
                  iam compartment list \
                      --raw-output \
                      --query "data[?name == 'cpm-automation'].id | [0]")
     ```

  1. Create a bucket to store Terraform state files

     Get the namespace.

     ```bash
     (oci) ~$ export OCI_NAMESPACE=$(
                oci --config-file $HOME/.oci/config \
                    --profile ricebucket1 \
                    --auth security_token \
                  os ns get --raw-output --query "data")
     ```

     Create the bucket in the namespace with versioning enabled

     ```bash
     (oci) ~$ oci --config-file $HOME/.oci/config \
                  --profile ricebucket1 \
                  --auth security_token \
                os bucket create --namespace $OCI_NAMESPACE \
                                 --name bucket-terraform \
                                 --compartment-id $OCI_CPM_AUTOMATION_ID \
                                 --versioning Enabled
     ```

## Setting up Vault to store Terraform Secrets

  1. Create a Vault in the CPM Automation compartment

     ```bash
     (oci) ~$ oci --config-file $HOME/.oci/config \
                  --profile ricebucket1 \
                  --auth security_token \
                kms management vault create --compartment-id $OCI_CPM_AUTOMATION_ID \
                                            --display-name vault-terraform \
                                            --vault-type DEFAULT
     ```

     This usually takes a minute or so.
     We need to save the Vault endpoint and Vault OCID.

     ```bash
     (oci) ~$ export OCI_VAULT_TERRAFORM_ENDPOINT=$(
                oci --config-file $HOME/.oci/config \
                    --profile ricebucket1 \
                    --auth security_token \
                  kms management vault list \
                      --compartment-id $OCI_CPM_AUTOMATION_ID \
                      --raw-output \
                      --query "data[?\"display-name\" == 'vault-terraform'] | [0].\"management-endpoint\"")
    
     (oci) ~$ export OCI_VAULT_ID=$(
                oci --config-file $HOME/.oci/config \
                    --profile ricebucket1 \
                    --auth security_token \
                  kms management vault list \
                      --compartment-id $OCI_CPM_AUTOMATION_ID \
                      --raw-output \
                      --query "data[?\"display-name\" == 'vault-terraform'] | [0].id")
     ```

  1. Generate a key for the Vault

     A key is needed to create secrets.

     ```bash
     (oci) ~$ oci --config-file $HOME/.oci/config \
                  --profile ricebucket1 \
                  --auth security_token \
                kms management key create --compartment-id $OCI_CPM_AUTOMATION_ID \
                                          --display-name vault-key \
                                          --key-shape '{"algorithm":"AES","length":"32"}' \
                                          --endpoint $OCI_VAULT_TERRAFORM_ENDPOINT
      ```

     Save the Vault Key's ID

     ```bash
     (oci) ~$ export OCI_VAULT_KEY_ID=$(
               oci --config-file $HOME/.oci/config \
                   --profile ricebucket1 \
                   --auth security_token \
                 kms management key list \
                   --compartment-id $OCI_CPM_AUTOMATION_ID \
                   --endpoint $OCI_VAULT_TERRAFORM_ENDPOINT \
                   --all \
                   --raw-output \
                   --query "data[?\"display-name\" == 'vault-key'] | [0].id")
     ```

     Secrets can now be created with the following:

     ```bash
     (oci) ~$ oci --config-file $HOME/.oci/config \
                  --profile ricebucket1 \
                  --auth security_token \
                vault secret create-base64 --compartment-id $OCI_CPM_AUTOMATION_ID \
                                           --vault-id $OCI_VAULT_ID \
                                           --key-id $OCI_VAULT_KEY_ID \
                                           --secret-content-stage CURRENT \
                                           --secret-name <secret_name> --description <secret_name> --secret-content-content $(echo "<plain text secret>" | base64)
     ```

## Set up policies for sa-terraform

`sa-teraform` should be limited to the following things:

- manage objects in `bucket-terraform`
- read secrets in `cpm-automation` compartment

The policy needs to be in JSON format:

```bash
(oci) ~$ cat ociPolicy-groupTerraform-backend.json
[
    "Allow group group-terraform to manage objects in compartment cpm-automation where target.bucket.name='bucket-terraform'",
    "Allow group group-terraform to read secret-family in compartment cpm-automation"
]
```

To apply the policy, run the following:

```bash
(oci) ~$ oci --config-file $HOME/.oci/config \
             --profile ricebucket1 \
             --auth security_token \
            iam policy create \
              --compartment-id $OCI_ROOT_COMPARTMENT_OCID \
              --name policy-groupTerraform-backend \
              --description "Group Terraform - Backend" \
              --statements file://ociPolicy-groupTerraform-backend.json
```

## Automation

Yea. I am not doing all of that manually. I wrote two scripts to automate the process.

Here's the commit for the script: <https://github.com/initialgyw/sitebook-ricebucket.life/commit/e626c9e9a555256e7988686e9c372c439d9b66ec>

Running the `./script/oci_secure_account.sh` looks like this:

```bash
.venv ❯ ./scripts/oci_secure_account.sh -r us-ashburn-1 -t ricebucket1 --config-file ~/.oci/config-ricebucket1 -v
{"timestamp":"20221230T190903","log_level":"DEBUG","message":"App detected in PATH: oci"}
{"timestamp":"20221230T190904","log_level":"DEBUG","message":"App detected in PATH: oathtool"}
{"timestamp":"20221230T190904","log_level":"DEBUG","message":"VAR: tenancy = ricebucket1"}
{"timestamp":"20221230T190904","log_level":"DEBUG","message":"VAR: region = us-ashburn-1"}
{"timestamp":"20221230T190904","log_level":"DEBUG","message":"VAR: profile = ricebucket1"}
{"timestamp":"20221230T190904","log_level":"DEBUG","message":"App detected in PATH: jq"}
{"timestamp":"20221230T190904","log_level":"DEBUG","message":"App detected in PATH: openssl"}
{"timestamp":"20221230T190904","log_level":"DEBUG","message":"RUNNING -- oci session validate --profile ricebucket1 --config-file ~/.oci/config-ricebucket1 --region us-ashburn-1 --local"}
{"timestamp":"20221230T190906","log_level":"CRITICAL","message":"Running oci session validate --profile ricebucket1 --config-file ~/.oci/config-ricebucket1 --region us-ashburn-1 --local returned exit code of 0: "}
{"timestamp":"20221230T190906","log_level":"DEBUG","message":"RUNNING -- oci session authenticate --config-location ~/.oci/config-ricebucket1 --profile-name ricebucket1 --region us-ashburn-1 --tenancy-name ricebucket1"}
{"timestamp":"20221230T190918","log_level":"DEBUG","message":"RUNNING -- oci session validate --profile ricebucket1 --config-file ~/.oci/config-ricebucket1 --region us-ashburn-1 --local"}
{"timestamp":"20221230T190919","log_level":"DEBUG","message":"Current session is valid."}
{"timestamp":"20221230T190919","log_level":"DEBUG","message":"RUNNING -- oci --config-file ~/.oci/config-ricebucket1 --profile ricebucket1 --auth security_token iam user list"}
{"timestamp":"20221230T190920","log_level":"DEBUG","message":"VAR ocid_tenancy = cid1.tenancy.oc1..<REDACTED>"}
{"timestamp":"20221230T190920","log_level":"DEBUG","message":"VAR user_account_name = ricebucket1@ricebucket.life"}
{"timestamp":"20221230T190920","log_level":"DEBUG","message":"VAR ocid_user_account = ocid1.user.oc1..<REDACTED>"}
{"timestamp":"20221230T190920","log_level":"DEBUG","message":"VAR user_account_mfa_status = false"}
{"timestamp":"20221230T190920","log_level":"INFO","message":"ricebucket1@ricebucket.life MFA is not enable. Will enable it."}
{"timestamp":"20221230T190920","log_level":"DEBUG","message":"RUNNING -- oci --config-file ~/.oci/config-ricebucket1 --profile ricebucket1 --auth security_token iam mfa-totp-device list --user-id ocid1.user.oc1..<REDACTED>"}
{"timestamp":"20221230T190921","log_level":"INFO","message":"No TOTP device found in ricebucket1@ricebucket.life account. Will create."}
{"timestamp":"20221230T190921","log_level":"DEBUG","message":"RUNNING -- oci --config-file ~/.oci/config-ricebucket1 --profile ricebucket1 --auth security_token iam mfa-totp-device create --user-id ocid1.user.oc1..<REDACTED>"}
{"timestamp":"20221230T190923","log_level":"SUCCESS","message":"Successfully created TOTP Device for ricebucket1@ricebucket.life: ocid1.credential.oc1..<REDACTED>"}
{"timestamp":"20221230T190923","log_level":"DEBUG","message":"VAR ocid_user_totp_device = ocid1.credential.oc1..<REDACTED>"}
{"timestamp":"20221230T190923","log_level":"DEBUG","message":"VAR ocid_user_totp_device_status = false"}
{"timestamp":"20221230T190923","log_level":"DEBUG","message":"VAR ocid_user_totp_device_seed = 26WRTBTWNHWEPPMIR3HNANTZESDQLEDS"}
{"timestamp":"20221230T190923","log_level":"INFO","message":"TOTP device is not active. Will activate."}
{"timestamp":"20221230T190923","log_level":"DEBUG","message":"RUNNING -- oci --config-file ~/.oci/config-ricebucket1 --profile ricebucket1 --auth security_token iam mfa-totp-device generate-totp-seed --user-id ocid1.user.oc1..<REDACTED> --mfa-totp-device-id ocid1.credential.oc1..<REDACTED>"}
*******************************************************
SAVE THIS SECRET SEED: S77UOB5WJ6FQSZMIMRMYBCEQXI7OOYQ3
*******************************************************
{"timestamp":"20221230T190927","log_level":"DEBUG","message":"RUNNING -- oci --config-file ~/.oci/config-ricebucket1 --profile ricebucket1 --auth security_token iam mfa-totp-device activate --user-id ocid1.user.oc1..<REDACTED> --mfa-totp-device-id ocid1.credential.oc1..<REDACTED> --totp-token 020952"}
{"timestamp":"20221230T190928","log_level":"SUCCESS","message":"Successfully activated TOTP device for ocid1.user.oc1..<REDACTED>"}
{"timestamp":"20221230T190928","log_level":"INFO","message":"DONE."}
```

And running the `./script/oci_setup_terraform.sh` looks like this:

```bash
{"timestamp":"20221230T191129","log_level":"DEBUG","message":"App detected in PATH: oci"}
{"timestamp":"20221230T191129","log_level":"DEBUG","message":"VAR: tenancy = ricebucket1"}
{"timestamp":"20221230T191129","log_level":"DEBUG","message":"VAR: region = us-ashburn-1"}
{"timestamp":"20221230T191129","log_level":"DEBUG","message":"VAR: profile = ricebucket1"}
{"timestamp":"20221230T191129","log_level":"DEBUG","message":"VAR private_rsa_key_path = ~/.oci/ricebucket1-sa-terraform.key"}
{"timestamp":"20221230T191129","log_level":"DEBUG","message":"VAR public_rsa_key_path = ~/.oci/ricebucket1-sa-terraform.pub"}
{"timestamp":"20221230T191130","log_level":"DEBUG","message":"VAR tmp_file = /tmp/oci_setup_terraform.sh.tmp"}
{"timestamp":"20221230T191130","log_level":"DEBUG","message":"App detected in PATH: jq"}
{"timestamp":"20221230T191130","log_level":"DEBUG","message":"App detected in PATH: openssl"}
{"timestamp":"20221230T191130","log_level":"DEBUG","message":"RUNNING -- oci session validate --profile ricebucket1 --config-file ~/.oci/config-ricebucket1 --region us-ashburn-1 --local"}
{"timestamp":"20221230T191130","log_level":"DEBUG","message":"Current session is valid."}
{"timestamp":"20221230T191130","log_level":"DEBUG","message":"RUNNING -- oci --config-file ~/.oci/config-ricebucket1 --profile ricebucket1 --auth security_token iam user list"}
{"timestamp":"20221230T191132","log_level":"INFO","message":"sa-terraform service account does not exist. Will create."}
{"timestamp":"20221230T191132","log_level":"DEBUG","message":"RUNNING -- oci --config-file ~/.oci/config-ricebucket1 --profile ricebucket1 --auth security_token iam user create --name sa-terraform --description service_account-tf"}
{"timestamp":"20221230T191133","log_level":"DEBUG","message":"VAR ocid_sa_tf = ocid1.user.oc1..<REDACTED>"}
{"timestamp":"20221230T191133","log_level":"DEBUG","message":"Capabilities to update: --can-use-o-auth2-client-credentials false --can-use-db-credentials false --can-use-console-password false --can-use-auth-tokens false --can-use-customer-secret-keys false --can-use-smtp-credentials false"}
{"timestamp":"20221230T191133","log_level":"DEBUG","message":"RUNNING -- oci --config-file ~/.oci/config-ricebucket1 --profile ricebucket1 --auth security_token iam user update-user-capabilities --user-id ocid1.user.oc1..<REDACTED> --can-use-o-auth2-client-credentials false --can-use-db-credentials false --can-use-console-password false --can-use-auth-tokens false --can-use-customer-secret-keys false --can-use-smtp-credentials false"}
{"timestamp":"20221230T191134","log_level":"SUCCESS","message":"Successfully updated capabilities for ocid1.user.oc1..<REDACTED>"}
{"timestamp":"20221230T191135","log_level":"DEBUG","message":"ocid1.user.oc1..<REDACTED> already have required capabilities"}
{"timestamp":"20221230T191135","log_level":"DEBUG","message":"RUNNING -- oci --config-file ~/.oci/config-ricebucket1 --profile ricebucket1 --auth security_token iam user api-key list --user-id ocid1.user.oc1..<REDACTED>"}
{"timestamp":"20221230T191136","log_level":"DEBUG","message":"RUNNING -- openssl genrsa -out ~/.oci/ricebucket1-sa-terraform.key 4096"}
{"timestamp":"20221230T191136","log_level":"SUCCESS","message":"~/.oci/ricebucket1-sa-terraform.key generated."}
{"timestamp":"20221230T191136","log_level":"INFO","message":"DEBUG -- openssl rsa -pubout -in ~/.oci/ricebucket1-sa-terraform.key -out ~/.oci/ricebucket1-sa-terraform.pub"}
{"timestamp":"20221230T191136","log_level":"SUCCESS","message":"~/.oci/ricebucket1-sa-terraform.pub generated from ~/.oci/ricebucket1-sa-terraform.key."}
{"timestamp":"20221230T191136","log_level":"DEBUG","message":"RUNNING -- oci --config-file ~/.oci/config-ricebucket1 --profile ricebucket1 --auth security_token iam user api-key upload --user-id ocid1.user.oc1..<REDACTED> --key-file ~/.oci/ricebucket1-sa-terraform.pub"}
{"timestamp":"20221230T191137","log_level":"SUCCESS","message":"Successfully uploaded ~/.oci/ricebucket1-sa-terraform.pub to sa-terraform."}
{"timestamp":"20221230T191138","log_level":"DEBUG","message":"~/.oci/ricebucket1-sa-terraform.key permission update."}
1,2d0
<
<
8a7,12
> [sa-terraform]
> fingerprint=80:40:67:64:70:ca:9d:c1:d5:47:cd:c3:24:c6:cc:cd
> key_file=~/.oci/ricebucket1-sa-terraform.key
> region=us-ashburn-1
> tenancy=ocid1.tenancy.oc1..<REDACTED>
> user=ocid1.user.oc1..<REDACTED>
{"timestamp":"20221230T191139","log_level":"DEBUG","message":"~/.oci/config-ricebucket1 permission update."}
{"timestamp":"20221230T191139","log_level":"DEBUG","message":"RUNNING -- oci --config-file ~/.oci/config-ricebucket1 --profile ricebucket1 --auth security_token iam group list"}
{"timestamp":"20221230T191140","log_level":"DEBUG","message":"VAR group_tf = group-terraform"}
{"timestamp":"20221230T191140","log_level":"WARN","message":"group-tf does not exist. Will create."}
{"timestamp":"20221230T191140","log_level":"DEBUG","message":"RUNNING -- oci --config-file ~/.oci/config-ricebucket1 --profile ricebucket1 --auth security_token iam group create --name group-terraform --description group_tf"}
{"timestamp":"20221230T191141","log_level":"SUCCESS","message":"Created group-terraform: ocid1.group.oc1..<REDACTED>"}
{"timestamp":"20221230T191141","log_level":"DEBUG","message":"RUNNING -- oci --config-file ~/.oci/config-ricebucket1 --profile ricebucket1 --auth security_token iam group list-users --group-id ocid1.group.oc1..<REDACTED>"}
{"timestamp":"20221230T191142","log_level":"INFO","message":"Adding sa-terraform to group-terraform"}
{"timestamp":"20221230T191142","log_level":"DEBUG","message":"RUNNING -- oci --config-file ~/.oci/config-ricebucket1 --profile ricebucket1 --auth security_token iam group add-user --user-id ocid1.user.oc1..<REDACTED> --group-id ocid1.group.oc1..<REDACTED>"}
{"timestamp":"20221230T191144","log_level":"SUCCESS","message":"Successfully added sa-terraform to group-terraform"}
{"timestamp":"20221230T191144","log_level":"DEBUG","message":"RUNNING -- oci --config-file ~/.oci/config-ricebucket1 --profile ricebucket1 --auth security_token iam compartment list"}
{"timestamp":"20221230T191145","log_level":"INFO","message":"cpm-terraform does not exist. Will create."}
{"timestamp":"20221230T191145","log_level":"DEBUG","message":"RUNNING -- oci --config-file ~/.oci/config-ricebucket1 --profile ricebucket1 --auth security_token iam compartment create --name cpm-terraform --compartment-id ocid1.tenancy.oc1..<REDACTED> --description stores_terraform_created_resources"}
{"timestamp":"20221230T191146","log_level":"SUCCESS","message":"Successfully created cpm-terraform: ocid1.compartment.oc1..<REDACTED>"}
{"timestamp":"20221230T191146","log_level":"DEBUG","message":"VAR ocid_compartment_tf = ocid1.compartment.oc1..<REDACTED>"}
{"timestamp":"20221230T191147","log_level":"DEBUG","message":"RUNNING -- oci --config-file ~/.oci/config-ricebucket1 --profile ricebucket1 --auth security_token iam policy list --compartment-id ocid1.tenancy.oc1..<REDACTED>"}
{"timestamp":"20221230T191148","log_level":"DEBUG","message":"RUNNING -- oci --config-file ~/.oci/config-ricebucket1 --profile ricebucket1 --auth security_token iam policy create --compartment-id ocid1.tenancy.oc1..<REDACTED> --name policy-group-terraform --description policy_for_group-terraform --statements file:///tmp/oci_setup_terraform.sh.tmp.json"}
{"timestamp":"20221230T191149","log_level":"SUCCESS","message":"Successfully set policies "}
{"timestamp":"20221230T191149","log_level":"SUCCESS","message":"Done!"}
```
