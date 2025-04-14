Snapshot CronJob Deployment for KubeVirt Virtual Machine

Files Overview

1. `variables.env`
   This file contains the key variables needed to configure the cron job, such as the VM name, namespace, service account, retention policy, and schedule for snapshot creation.

2. `sa-role-binding.yaml`
   This file creates the necessary `ServiceAccount`, `Role`, and `RoleBinding` to allow the cron job to interact with Kubernetes resources. If your environment already has the required service account and permissions, this file can be skipped.

3. `schedule-snapshot.yaml`
   The CronJob definition file that will schedule periodic snapshots for your VM. It is parameterized and can be configured using the environment variables provided in `variables.env`.

Deployment Flow

Step 1: Set Up Your Environment (Variables)

1. Create the `variables.env` file
   This file contains important parameters such as:
   - `VM_NAME` - The name of the VM you want to snapshot (e.g., `fedora01`).
   - `NAMESPACE` - The namespace in which your VM and snapshot resources will reside (e.g., `vmexamples-user1`).
   - `SCHEDULE` - The cron job schedule expression for when to create the snapshots (e.g., `"*/15 * * * *"` for every 15 minutes).
   - `SERVICE_ACCOUNT` - The name of the service account to use (e.g., `default`).
   - `RETENTION` - The number of snapshots to keep (e.g., `5`).

   Example `variables.env`:

   VM_NAME=fedora01
   NAMESPACE=vmexamples-user1
   SCHEDULE="*/15 * * * *"
   SERVICE_ACCOUNT=default
   RETENTION=5

   Modify: Modify the `variables.env` file with your specific values. Ensure the `VM_NAME`, `NAMESPACE`, `SERVICE_ACCOUNT`, and `SCHEDULE` align with your environment and needs.

2. Upload the `variables.env` file
   If you're using an automation pipeline (e.g., CI/CD), this file should be uploaded securely. If deploying manually, keep it locally on your system.

Step 2: Service Account and Permissions (Optional)

1. Check for Existing Service Account and Permissions
   If your environment already has a service account (`default` or another) with the necessary permissions for creating snapshots and managing resources in the specified namespace, you can skip this step.

   The necessary permissions include:
   - Creating, deleting, and listing snapshots.
   - Accessing virtual machines and pods.

2. Deploy `sa-role-binding.yaml` (if needed)
   If a new service account or permissions are required, use the `sa-role-binding.yaml` file. This will create the service account, role, and role binding.

   To deploy:
   kubectl apply -f sa-role-binding.yaml

   Modify (if needed): Modify the `sa-role-binding.yaml` file to adjust the namespace or service account name.

   Skip (if not needed): If the `default` service account already has the required permissions, you can skip this deployment.

Step 3: Schedule Snapshot Creation

1. Deploy `schedule-snapshot.yaml`
   This file contains the cron job that will schedule the snapshot creation. It uses the environment variables set in `variables.env` to run at the specified time.

   To deploy the cron job, run:
   kubectl apply -f schedule-snapshot.yaml

   Modify (if needed): Modify the cron job's schedule or other environment variables to fit your requirements.

2. Monitor the CronJob
   Once deployed, the cron job will automatically start creating snapshots as per the schedule you defined. You can monitor the job by running:
   kubectl get cronjobs
   kubectl get jobs

Step 4: Check Snapshot Retention

1. The cron job includes a retention policy. It will keep only the most recent snapshots (as defined by the `RETENTION` variable in `variables.env`) and delete older ones automatically.

2. You can view the current snapshots by running:
   kubectl get vmsnapshots -n <namespace>
