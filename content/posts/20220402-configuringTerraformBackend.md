---
title: Terraform for ricebucket.life
description: Where to store State Files and Secrets?
date: 2022-04-02
toc: true
authors:
  - konri
categories:
  - homelab
draft: false
---

I'll be using Terraform to automate my deployment of cloud services in different cloud providers. I'll need to decide on the following three things:

1. ~~A centralized location to store Terraform code.~~ I already decided as [my code repositories should be in Github](https://www.ricebucket.life/posts/20230101-helloworld#existing-services).
1. A centralized storage for remote Terraform state files.
1. A centralized secrets management.

Hashicorp also offers [free Cloud Terraform service](https://www.hashicorp.com/products/terraform/pricing). I don't know how this works yet in terms of what is free and limitations, so I'm going to use the CLI to start.

## Storage for Terraform state files

The remote storage for Terraform state files need to be secure. I can't store the state file locally since I don't always use the same development box and I can't just push the state files into a git repository as that might break the compliance of encryption at rest. Why not just store it on a cloud provider's storage since all of the cloud providers have encryption-at-rest?

Here are some possible solutions:

- [AWS provides 5GB of S3 for up to 12 months](https://aws.amazon.com/free/?all-free-tier.sort-by=item.additionalFields.SortRank&all-free-tier.sort-order=asc&awsf.Free%20Tier%20Types=tier%23always-free%7Ctier%2312monthsfree&awsf.Free%20Tier%20Categories=categories%23storage). I don't want to pay for it after 1 year.
- [GCP provides 5GB of Cloud Storage as part of their Always Free program](https://cloud.google.com/free/docs/gcp-free-tier/#storage). Free is nice.
- [Oracle Cloud offers 20GB of Storage in their Always Free plan](https://www.oracle.com/cloud/free/#always-free). It's the combination of Object Storage (10GB) and Archive Storage (10GB). Storage buckets (like AWS S3) falls under Object Storage which would give me 20GB of storage.

## Secrets for Terraform

At my job, we use Hashicorp Vault for secrets management. I would love self-host the Vault service but since this is the start of a fresh new homelab, I don't have any available servers to install Hashicorp Vault.

Here are some other potential solutions:

### AWS Secret Manager

<https://aws.amazon.com/secrets-manager/pricing/>

It's not free. It's cheap but it's not free.

### GCP Secret Manager

<https://cloud.google.com/free/docs/gcp-free-tier/#secret-manager>

![GCP Secret Manager Always Free](/images/posts/gcp_secretManager.png)

This is how I create secrets in GCP.

  1. Enable Secret Manager API

     ```bash
     gcloud services enable secretmanager.googleapis.com
     ```

  1. Add the secret manager view role to a service account

     ```bash
     gcloud projects add-iam-policy-binding <project> --member=serviceAccount:terraform@<project>.iam.gserviceaccount.com --role=roles/secretmanager.viewer --condition=None
     ```

     There's no way to limit specific set of secrets to the service account. For example, I only want to allow terraform service account to read secrets with the name starting with *terraform_*.
     The following does not work:

     ```bash
     gcloud projects add-iam-policy-binding <project> --member=serviceAccount:terraform@<project>.iam.gserviceaccount.com --role=roles/secretmanager.secretAccessor --condition='expression=resource.name.startsWith("projects/_/secrets/terraform"),title=terraform-secretAccessor'
     ```

     Maybe I am doing this wrong?

  1. Create secrets

     ```bash
     echo -n "passwordisthis" | gcloud secrets create testpassword --data-file=-
     ```

  1. To view the secrets

     ```bash
     gcloud secrets versions access latest --secret=testpassword
     ```

My assumption to make this service always free is to

- Not have 10,000 access operations per month. It's very unlikely that I would request more than 10,000 operations per month read to all my secrets.

    > **Update**: When I attached the Terraform code to a CI/CD pipeline (Github Actions) in which I would do a `terraform plan` on every commit, I ate up the 10,000 operations in few days.

- Not have 6 active secret versions. That means per secret right? As long as I have 1 active version per secret, I shouldn't be charged, right?
- Not using secret rotation. I'm using the Secrets Manager like a "*password manager*"

After adding 16 secrets, and not using over 10,000 operations, GCP is charging me.

![GCP Secret Manager is not free](/images/posts/gcp_secretManagerBill.png)

I don't understand why I'm getting charged. Is it for replication? [Is that the default setting? Can I turn it off? It doesn't seem like there is a way.](https://cloud.google.com/secret-manager/docs/choosing-replication)
Well, this sucks. Secret Manager is not really free?

> **Update** I think I am misunderstanding the `6 active secrets versions`. If each secret is consider a version, then it really means I can only store 6 secrets for free. #lame

### Oracle Cloud Vault

<https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm>

![Oracle Cloud Always Free Vault](/images/posts/oci-freeTierVault.png)

Oracle Cloud Vault provides 150 total secrets across any number of Vaults. I don't know if this is a lot for my homelab but 150 secrets should be sufficient to start.

#### Create and View Secrets

Using the Python's OCI command line tool, create the secret manually.

  ```bash
  oci vault secret create-base64 \
    --compartment-id ocid1.compartment.oc1..<REDACTED> \
    --secret-name testpassword \
    --vault-id ocid1.vault.oc1.iad.<REDACTED> \
    --description "Test Password" \
    --key-id ocid1.key.oc1.iad.<REDACTED> \
    --secret-content-content $(echo "passwordisthis" | base64)
    --secret-content-stage CURRENT
  ```

Secret can be viewed:

  ```bash
  oci secrets secret-bundle get \
    --secret-id ocid1.vaultsecret.oc1.iad.<REDACTED> \
    --raw-output \
    --query "data.\"secret-bundle-content\".content" | base64 -d
  Private key passphrase:
  passwordisthis
  ```

This is how to get the secrets using Terraform:

```bash
❯ cat test.tf
provider "oci" {
  tenancy_ocid         = var.tenancy_ocid
  user_ocid            = var.user_ocid
  private_key_path     = var.private_key_path
  fingerprint          = var.fingerprint
  region               = var.region
  private_key_password = var.private_key_password
}

data "oci_vault_secrets" "secrets" {
    compartment_id = var.compartment_ocid
}

data "oci_vault_secrets" "secrets" {
    compartment_id = var.compartment_ocid
}

data "oci_secrets_secretbundle" "testpassword" {
    secret_id = data.oci_vault_secrets.secrets.secrets[index(data.oci_vault_secrets.secrets.secrets.*.secret_name, "testpassword")].id
}

output "test" {
    # value = data.oci_vault_secrets.test_secrets.secrets[index(data.oci_vault_secrets.test_secrets.secrets.*.secret_name, "testpassword")]
    value = base64decode(data.oci_secrets_secretbundle.testpassword.secret_bundle_content[0].content)
}
```

This is the output of the Terraform plan.

```bash
❯ terraform plan

Changes to Outputs:
  + secrets = {
      + compartment_id = "ocid1.compartment.oc1..<REDACTED>"
      + filter         = null
      + id             = "VaultSecretsDataSource-<REDACTED>"
      + name           = null
      + secrets        = [
          + {
              + compartment_id                 = "ocid1.compartment.oc1..<REDACTED>"
              + current_version_number         = ""
              + defined_tags                   = {}
              + description                    = ""
              + freeform_tags                  = {}
              + id                             = "ocid1.vaultsecret.oc1.iad.<REDACTED>"
              + key_id                         = "ocid1.key.oc1.iad.<REDACTED>"
              + lifecycle_details              = ""
              + metadata                       = {}
              + secret_content                 = []
              + secret_name                    = "testpassword"
              + secret_rules                   = []
              + state                          = "ACTIVE"
              + time_created                   = "2022-04-03 00:05:37.275 +0000 UTC"
              + time_of_current_version_expiry = ""
              + time_of_deletion               = ""
              + vault_id                       = "ocid1.vault.oc1.iad.<REDACTED>"
            },
        ]
      + state          = null
      + vault_id       = null
    }
  + test    = "passwordisthis"
```

### Unix Passwordstore

I [read an article](https://blog.gruntwork.io/a-comprehensive-guide-to-managing-secrets-in-your-terraform-code-1d586955ace1) on how some users manage their secrets. Unix Pass is completely free app in which I can securely save and access the secrets via GPG key. It satifies encryption at rest compliance and I can even use a GIT respository as secret revisions. There is even a community provider, [camptocamp/terraform-provider-pass](https://github.com/camptocamp/terraform-provider-pass), that I can include in my Terraform code to access the secrets.

Secrets will be GPG encrypted and stored locally. This is the easiest solution as there is a Terraform provider that can read and write the GPG encrypted data.
The GPG key can be stored in OCI Vault for security.

This is probability the easiest solution and thinking about the future where I'll be using ansible as configuration manager, [ansible also has `pass` lookup to read secrets](https://docs.ansible.com/ansible/latest/collections/community/general/passwordstore_lookup.html).

**Cons**:

- This is not scalable for users; even though you can add multiple GPG keys in the `.gpg-id` file to for encryption, it's not convenient to re-encrypt with multiple keys when new users are boarded. It's only me for now so, it's not a problem but it is something to think about.
- Lost of GPG key means the secret can't be decrypted again. Where should I store the GPG key?
- Compromised key means the old secrets in the GIT versioning will be exposed and it's not easy/convenient to update each secret.

**Pros**:

- This solution is not relient on any services. All the descryption and encryption is done locally.
- GIT versioning control

## Conclusion

Since this is a learning environment which would replicate a production infrastructure, I'm going to pick a Cloud solution: Oracle Cloud for storage and Oracle Cloud for secrets management.
