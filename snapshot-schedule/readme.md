#   Snapshot CronJob Deployment for KubeVirt Virtual Machine

>   Tested on OpenShift Container Platform (OCP) 4.18

##  Files Overview

1.  **`variables.env`**

    This file contains the key variables needed to configure the cron job, such as the VM name, namespace, retention policy, and schedule for snapshot creation.

2.  **`sa-role-binding.yaml`**

    This file creates the `ServiceAccount`, `Role`, and `RoleBinding` necessary for the cron job to interact with Kubernetes resources.

    ***Important:*** This file needs to be applied only once per namespace.  If your namespace is already configured with a suitable service account and permissions, you can skip this.

3.  **`schedule-snapshot.yaml`**

    This is the CronJob definition file that schedules periodic snapshots for your VM. It is parameterized to be configured using the variables in `variables.env`. You will create a new instance of this CronJob for each VM you want to snapshot.

##  Deployment Flow

The following steps describe how to use these files to create a snapshot schedule for your KubeVirt Virtual Machines in OpenShift.

###  Step 1:  Establish Namespace (If Needed)

1.  **Check Current Namespace**

    * Determine the OpenShift namespace where your KubeVirt VMs are located (e.g., `my-kubevirt-namespace`).

2.  **Create a New Namespace (If Necessary)**

    * If you need to create a new namespace, use the following command:

        ```bash
        oc new-project <your-namespace>
        oc project <your-namespace>  # Set the current namespace
        ```

        * Replace `<your-namespace>` with your desired namespace name (e.g., `vm-snapshots`).

###  Step 2:  Set up Service Account and Permissions (One-Time Setup)

    This step configures the necessary permissions for snapshot creation.  It only needs to be done once per namespace.

1.  **Review `sa-role-binding.yaml`**

    * Verify that the `namespace` in the `sa-role-binding.yaml` file is set to the correct namespace (the one where your VMs are located, or the one you created in Step 1).

    * The file defines a ServiceAccount named `vm-snapshotter`, a Role with the required snapshot permissions, and a RoleBinding that connects them.

        ```yaml
        apiVersion: v1
        kind: ServiceAccount
        metadata:
          name: vm-snapshotter
          namespace: <your-namespace> #  <---  Important:  Set this!

        ---

        apiVersion: rbac.authorization.k8s.io/v1
        kind: Role
        metadata:
          name: vm-snapshot-role
          namespace: <your-namespace> #  <---  Important:  Set this!
        rules:
          - apiGroups:
              - snapshot.kubevirt.io
            resources:
              - virtualmachinesnapshots
            verbs:
              - create
              - delete
              - list
              - get
          - apiGroups:
              - ""
            resources:
              - pods
            verbs:
              - get
              - list
          - apiGroups:
              - kubevirt.io
            resources:
              - virtualmachines
            verbs:
              - get

        ---

        apiVersion: rbac.authorization.k8s.io/v1
        kind: RoleBinding
        metadata:
          name: vm-snapshot-rolebinding
          namespace: <your-namespace> #  <---  Important:  Set this!
        subjects:
          - kind: ServiceAccount
            name: vm-snapshotter
            namespace: <your-namespace> #  <---  Important:  Set this!
        roleRef:
          apiGroup: rbac.authorization.k8s.io
          kind: Role
          name: vm-snapshot-role
        ```

2.  **Apply `sa-role-binding.yaml`**

    * Use `oc` or `kubectl` to apply the YAML:

        ```bash
        oc apply -f sa-role-binding.yaml
        ```

    * This creates the necessary ServiceAccount and RBAC resources in your namespace.

###  Step 3:  Create a Snapshot Schedule for a VM

    Repeat this step for each VM for which you want to create a snapshot schedule.

1.  **Create/Modify `variables.env`**

    * Create a `variables.env` file (or modify an existing one) with the specific settings for the VM you want to snapshot.

        ```
        VM_NAME=<vm-name>           #  Name of the VM (e.g., fedora01)
        NAMESPACE=<your-namespace>  #  Namespace of the VM (e.g., vmexamples-user1)
        RETENTION=<integer>         #  Number of snapshots to keep (e.g., 5)
        SCHEDULE="<cron-schedule>" #  Cron schedule (e.g., "*/15 * * * *" for every 15 minutes)
        ```

    * Replace the placeholders with the actual values for your VM and snapshot schedule.

        * `<vm-name>`:  The name of the KubeVirt Virtual Machine.

        * `<your-namespace>`:  The OpenShift namespace where the VM is located.  This should match the namespace used in `sa-role-binding.yaml`.

        * `<integer>`:  The number of snapshots you want to retain.  Old snapshots will be automatically deleted.

        * `<cron-schedule>`:  A cron expression defining the snapshot schedule.  See [Cron Expressions](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/#schedule) for details.

    * **Example `variables.env`:**

        ```
        VM_NAME=my-vm-001
        NAMESPACE=my-kubevirt-namespace
        RETENTION=3
        SCHEDULE="0 2 * * *"  #  Run daily at 2:00 AM
        ```

2.  **Create ConfigMap from `variables.env`**

    * Use the `oc` command to create a ConfigMap from the `variables.env` file.  It is important to create a unique configmap name for each VM.

        ```bash
        VM_NAME="my-vm-001" #set this to the VM name
        oc create configmap autosnapshot-${VM_NAME}-config --from-env-file=variables.env -n <your-namespace>
        ```

        * Replace  `<your-namespace>` with the namespace where you are working.

        * This command creates a ConfigMap named `autosnapshot-my-vm-001-config` (in this example) containing the variables.

3.  **Apply `schedule-snapshot.yaml`**

    * Apply the `schedule-snapshot.yaml` file to create the CronJob.

        ```bash
        oc apply -f schedule-snapshot.yaml
        ```

    * The `schedule-snapshot.yaml`  uses the  variables from the ConfigMap created in the previous step to define the snapshot schedule.

        ```yaml
        apiVersion: batch/v1
        kind: CronJob
        metadata:
          name: snapshot-$(VM_NAME)
          namespace: $(NAMESPACE)
        spec:
          schedule: $(SCHEDULE)
          jobTemplate:
            spec:
              template:
                spec:
                  serviceAccountName: vm-snapshotter # Use the ServiceAccount created in Step 2
                  containers:
                    - name: snapshot
                      image: quay.io/openshift/origin-cli:4.18
                      env:
                        - name: VM_NAME
                          valueFrom:
                            configMapKeyRef:
                              name: autosnapshot-$(VM_NAME)-config  #  Dynamic name, matches ConfigMap
                              key: VM_NAME
                        - name: NAMESPACE
                          valueFrom:
                            configMapKeyRef:
                              name: autosnapshot-$(VM_NAME)-config  #  Dynamic name, matches ConfigMap
                              key: NAMESPACE
                        - name: SCHEDULE
                          valueFrom:
                            configMapKeyRef:
                              name: autosnapshot-$(VM_NAME)-config  #  Dynamic name, matches ConfigMap
                              key: SCHEDULE
                        - name: RETENTION
                          valueFrom:
                            configMapKeyRef:
                              name: autosnapshot-$(VM_NAME)-config  #  Dynamic name, matches ConfigMap
                              key: RETENTION
                      command:
                        - /bin/sh
                        - -c
                        - |
                          set -e
                          export HOME=/tmp
                          DATE=$(date +%Y%m%d-%H%M%S)
                          SNAPSHOT_NAME="$(VM_NAME)-snapshot-$DATE"
                          echo "Creating snapshot: $SNAPSHOT_NAME"
                          cat <<EOF | oc apply -f -
                          apiVersion: snapshot.kubevirt.io/v1beta1
                          kind: VirtualMachineSnapshot
                          metadata:
                            name: $SNAPSHOT_NAME
                            namespace: $(NAMESPACE)
                            labels:
                              kubevirt.io/snapshot-source-name: $(VM_NAME)
                          spec:
                            source:
                              apiGroup: kubevirt.io
                              kind: VirtualMachine
                              name: $(VM_NAME)
                          EOF
                          echo "Applying retention policy: keep only \$RETENTION latest snapshots"
                          # Get snapshot names sorted by creationTimestamp (descending)
                          SNAPSHOTS=$(oc get vmsnapshot -n $(NAMESPACE) -l kubevirt.io/snapshot-source-name=$(VM_NAME) --sort-by=.metadata.creationTimestamp -o custom-columns=NAME:.metadata.name --no-headers | tac)
                          COUNT=0
                          for SNAP in $SNAPSHOTS; do
                            COUNT=$((COUNT + 1))
                            if [ "$COUNT" -gt "$RETENTION" ]; then
                              echo "Deleting old snapshot: $SNAP"
                              oc delete vmsnapshot "$SNAP" -n $(NAMESPACE) || true
                          fi
                        done
                      resources:
                        requests:
                          memory: "64Mi"
                          cpu: "100m"
                        limits:
                          memory: "128Mi"
                          cpu: "250m"
                      restartPolicy: OnFailure
        ```

###  Step 4:  Verify Snapshot Creation

1.  **Monitor the CronJob**

    * Check the status of the CronJob:

        ```bash
        oc get cronjobs -n <your-namespace>
        ```

    * Check the status of the Jobs created by the CronJob:

        ```bash
        oc get jobs -n <your-namespace>
        ```

2.  **Verify Snapshots**

    * List the created VirtualMachineSnapshots:

        ```bash
        oc get vmsnapshots -n <your-namespace>
        ```

    * Verify that snapshots are being created according to the schedule defined in your `variables.env` file and that the retention policy is being enforced.
