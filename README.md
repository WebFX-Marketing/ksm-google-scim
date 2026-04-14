![Keeper Secret Manager Google SCIM Push Header](https://github.com/user-attachments/assets/856e2170-d1ce-4262-a425-869e10fd04fc)

# Keeper Secrets Manager : Google SCIM Push

This repository contains the source code that synchronizes Google Workspace Users/Groups and Keeper Enterprise Users/Teams. This is necessary because Google Workspace does not adequately support Team SCIM provisioning.

## Configuration Providers
This tool supports fetching its required configuration via two methods:
1.  **Keeper Secrets Manager (KSM)**: The default and recommended approach.
2.  **Google Cloud Secret Manager (GCP SM)**: For strictly native GCP environments.

## Step by Step Instructions
Read this document: [Google Workspace User and Group Provisioning with Cloud Function](https://docs.keeper.io/en/sso-connect-cloud/identity-provider-setup/g-suite-keeper/google-workspace-user-and-group-provisioning-with-cloud-function)

> This project replicates the `keeper scim push --source=google` [Commander CLI command](https://docs.keeper.io/en/keeperpam/commander-cli/command-reference/enterprise-management-commands/scim-push-configuration) and shares configuration settings with this command.

### Prerequisites
* Keeper Secret Manager enterprise subscription (if using KSM)
* Google Cloud project with billing enabled

### Option 1: Prepare KSM Application
  * Create KSM application or reuse the existing one
  * Share the SCIM configuration record with this KSM application
  * `Add Device` and make sure method is `Configuration File` Base64 encoding.

### Option 2: Prepare GCP Secret Manager
  * Format a single JSON payload containing the required parameters (derived from what KSM would normally provide):
    ```json
    {
      "scimUrl": "https://keepersecurity.com/api/rest/scim/v2/...",
      "scimToken": "your-scim-token",
      "googleAdminAccount": "admin@yourdomain.com",
      "scimGroups": ["Keeper-SCIM-Users"],
      "googleCredentials": {
         // Full Google Service Account JSON content here
      }
    }
    ```
  * Save this JSON as a new secret in GCP Secret Manager.
  * Ensure the Service Account executing the Cloud Function has the `Secret Manager Secret Accessor` role.

### Configuration with `gcloud`
1. Clone this repository locally
2. Copy `.env.yaml.sample` to `.env.yaml`
3. Edit `.env.yaml`
   * **If using KSM:**
     * Set `KSM_CONFIG_BASE64` to the content of the KSM configuration file generated at the previous step
     * Set `KSM_RECORD_UID` to configuration record UID created for Commander's `scim push` command
   * **If using GCP SM:**
     * Set `GCP_SECRET_NAME` to the full path of your secret (e.g., `projects/YOUR_PROJECT_ID/secrets/YOUR_SECRET_NAME/versions/latest`)
4. Create Google Cloud function. Replace `<REGION>` placeholder with the GCP region. 
```shell
gcloud functions deploy <PickUniqueFunctionName> \
--gen2 \
--runtime=go125 \
--max-instances=1 \
--memory=512M \
--env-vars-file .env.yaml \
--region=<REGION> \
--timeout=120s \
--source=. \
--entry-point=GcpScimSyncHttp \
--trigger-http \
--no-allow-unauthenticated
```

### Configuration with `Google Console`
1. Clone this repository locally
2. Create `source.zip` file that contains "*.go" and "go.*" matches
```shell
zip source.zip `find . -name "*.go"`
zip source.zip `find . -name "go.*"`
```
3. Login to Google Console
4. Create a new function ![Create New Function](./images/create_new_function.png)
![Create Step 1](./images/create_step1.png)
![Create Step 2](./images/create_step2.png)
![Create Step 3](./images/create_step3.png)
   * **If using KSM:**
     * Set `KSM_CONFIG_BASE64` to the content of the KSM configuration file generated at the previous step
     * Set `KSM_RECORD_UID` to configuration record UID created for Commander's `scim push` command
   * **If using GCP SM:**
     * Set `GCP_SECRET_NAME` to the full path of your secret (e.g., `projects/YOUR_PROJECT_ID/secrets/YOUR_SECRET_NAME/versions/latest`)
5. Click `NEXT`
6. Set "Entry point" to `GcpScimSyncHttp`
7. Upload the source code using `source.zip`. "Destination bucket" can be any.
![Create Step 4](./images/create_step4.png)
8. Click `DEPLOY`

### Create Cloud Scheduler with `Google Console`
1. Find the created function and copy function URL to the clipboard
   ![Copy URL](./images/copy_url.png)

2. Search for `scheduler` and select `Cloud Scheduler`
3. Click `CREATE JOB`. `15 * * * *` means every hour at 15th minute

   ![Scheduler Step 1](./images/scheduler_step1.png)
4. Grant the scheduler access to SCIM function 

   ![Scheduler Access](./images/scheduler_access.png)
5. Create Scheduler and check it works by clicking `FORCE RUN`

   ![Scheduler Run](./images/scheduler_run.png)
