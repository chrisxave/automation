# **Snapshot CronJob Deployment for KubeVirt Virtual Machine**

## **Files Overview**

1. **`variables.env`**  
   This file contains the key variables needed to configure the cron job, such as the VM name, namespace, service account, retention policy, and schedule for snapshot creation.

2. **`sa-role-binding.yaml`**  
   This file creates the necessary `ServiceAccount`, `Role`, and `RoleBinding` to allow the cron job to interact with Kubernetes resources.  
   *If your environment already has the required service account and permissions, this file can be skipped.*

3. **`schedule-snapshot.yaml`**  
   The CronJob definition file that will schedule periodic snapshots for your VM.  
   It is parameterized and can be configured using the environment variables provided in `variables.env`.

---

## **Deployment Flow**

### **Step 1: Set Up Your Environment (Variables)**

1. **Create the `variables.env` file**  
   This file contains important parameters:

   - `VM_NAME` — The name of the VM to snapshot (e.g., `fedora01`)
   - `NAMESPACE` — The namespace containing the VM (e.g., `vmexamples-user1`)
   - `SCHEDULE` — The cron job schedule (e.g., `"*/15 * * * *"` for every 15 minutes)
   - `SERVICE_ACCOUNT` — The service account to use (e.g., `default`)
   - `RETENTION` — Number of snapshots to retain (e.g., `5`)

   **Example `variables.env`:**

   ```env
   VM_NAME=fedora01
   NAMESPACE=vmexamples-user1
   SCHEDULE="*/15 * * * *"
   SERVICE_ACCOUNT=default
   RETENTION=5
   ```

2. **Modify:**  
   Edit the values in `variables.env` to match your environment.

3. **Upload:**  
   If using automation, upload this file securely.  
   If deploying manually, keep it on your local machine.

---

### **Step 2: Service Account and Permissions (Optional)**

1. **Check existing service account and permissions**  
   If your environment already has a service account (e.g., `default`) with the following permissions, skip this step:

   - Create, delete, and list `VirtualMachineSnapshot`
   - Access `pods` and `virtualmachines`

2. **Deploy `sa-role-binding.yaml` (if needed)**  
   If a service account or permissions are missing, use the provided file to create them:

   ```bash
   kubectl apply -f sa-role-binding.yaml
   ```

   **Modify (if needed):**  
   Adjust the namespace or service account name in the YAML if necessary.

   **Skip (if not needed):**  
   If the `default` service account already has the required permissions, skip this step.

---

### **Step 3: Schedule Snapshot Creation**

1. **Deploy the CronJob**

   ```bash
   kubectl apply -f schedule-snapshot.yaml
   ```

   This uses the values in `variables.env` and will start the periodic snapshot process.

2. **Modify (if needed):**  
   You can change the schedule or resource settings in `schedule-snapshot.yaml` if needed.

3. **Monitor the CronJob**

   ```bash
   kubectl get cronjobs -n <namespace>
   kubectl get jobs -n <namespace>
   ```

---

### **Step 4: Check Snapshot Retention**

1. The cron job automatically applies retention, keeping only the latest N snapshots as defined in `RETENTION`.

2. You can check existing snapshots with:

   ```bash
   kubectl get vmsnapshots -n <namespace>
   ```

---
