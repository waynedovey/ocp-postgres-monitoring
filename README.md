# Monitoring PostgreSQL on OpenShift with IBM Guardium

## Overview
This guide outlines the steps to configure IBM Guardium to monitor database activity and data changes in a PostgreSQL database deployed on OpenShift. We’ll use EDB PostgreSQL as an example, leveraging Guardium’s External S-TAP for Kubernetes to capture database traffic in a containerized environment.

---

## Step 1: Deploy PostgreSQL on OpenShift Using EDB
To begin, you need a running PostgreSQL instance on OpenShift. EDB provides tools and operators to simplify this deployment.

1. **Prerequisites**:
   - An OpenShift cluster is set up and accessible.
   - You have administrative access to deploy applications.

2. **Deploy EDB PostgreSQL**:
   - EDB offers a Kubernetes Operator for PostgreSQL, which can be deployed on OpenShift. Follow EDB’s official documentation (typically available on their website or GitHub repository) to install the operator.
   - Example steps:
     - Install the EDB operator using OpenShift’s OperatorHub or by applying the operator’s YAML manifests.
     - Create a PostgreSQL cluster using a custom resource definition (CRD). For example:
```yaml
     apiVersion: postgresql.k8s.enterprisedb.io/v1
     kind: Cluster
     metadata:
       name: edb-postgres
       namespace: default
     spec:
       instances: 1
       imageName: "containers.enterprisedb.io/postgres/pe:14.5"
       imagePullSecrets:
         - name: edb-pull-secret
       postgresql:
         parameters:
           shared_buffers: "256MB"
       storage:
         size: 1Gi
       ```
     - Apply this manifest using `oc apply -f <filename>.yaml`.
   - Verify the deployment:
     ```bash
     oc get pods -n default
     ```
     Ensure the PostgreSQL pod is running.

3. **Expose the Database**:
   - Ensure the PostgreSQL instance is accessible via a Kubernetes service. The operator typically creates a service automatically (e.g., `edb-postgres-pods`). Check with:
     ```bash
     oc get svc -n default
     ```
   - Note the service name, namespace, and port (default is 5432).

---

## Step 2: Set Up the IBM Guardium Collector
The Guardium collector is the central component that receives and processes monitoring data.

1. **Deploy the Collector**:
   - Install an IBM Guardium collector appliance, which can be on-premises or a virtual appliance in your cloud environment. Follow IBM’s installation guide for your specific setup.
   - Ensure the collector is accessible from your OpenShift cluster (network connectivity is required).

2. **Configure the Collector**:
   - Log in to the Guardium management console.
   - Verify that the collector is running and ready to accept data from monitoring agents.

---

## Step 3: Deploy External S-TAP on OpenShift
Since PostgreSQL is running in a containerized environment on OpenShift, use Guardium’s External S-TAP for Kubernetes to monitor database traffic without installing an agent inside the database container.

1. **Obtain External S-TAP**:
   - Download the External S-TAP for Kubernetes package from IBM’s official resources (e.g., IBM Fix Central or the Guardium documentation portal).

2. **Deploy External S-TAP**:
   - IBM provides deployment manifests or an operator for External S-TAP. If using the Guardium operator:
     - Install the Guardium operator via OpenShift OperatorHub or by applying its YAML manifests.
     - Configure a custom resource to deploy External S-TAP, specifying the PostgreSQL service to monitor.
   - Manual deployment example:
     - Create a deployment YAML for External S-TAP, adjusting the configuration to target your PostgreSQL service:
       ```yaml
       apiVersion: apps/v1
       kind: Deployment
       metadata:
         name: guardium-external-stap
         namespace: default
       spec:
         replicas: 1
         selector:
           matchLabels:
             app: guardium-external-stap
         template:
           metadata:
             labels:
               app: guardium-external-stap
           spec:
             containers:
             - name: external-stap
               image: ibmcom/guardium-external-stap:latest
               env:
               - name: DB_TYPE
                 value: "POSTGRESQL"
               - name: DB_SERVICE_NAME
                 value: "edb-postgres-pods.default.svc.cluster.local"
               - name: DB_PORT
                 value: "5432"
               - name: GUARDIUM_COLLECTOR_HOST
                 value: "<collector-ip-or-hostname>"
       ```
     - Apply with `oc apply -f <filename>.yaml`.
   - Verify the deployment:
     ```bash
     oc get pods -n default
     ```
     Ensure the External S-TAP pod is running.

3. **Configure External S-TAP**:
   - Ensure environment variables or a configuration file specifies the PostgreSQL service details (host, port, etc.) and the Guardium collector’s address.

---

## Step 4: Configure Guardium to Monitor the Database
Set up Guardium to recognize the PostgreSQL database and define monitoring policies.

1. **Add a Datasource**:
   - In the Guardium console, navigate to **Manage > Data Sources**.
   - Add a new datasource:
     - **Type**: PostgreSQL
     - **Host**: Use the service name (e.g., `edb-postgres-pods.default.svc.cluster.local`).
     - **Port**: 5432
     - **Monitoring Method**: External S-TAP
   - Test the connection to ensure Guardium can communicate with the database via External S-TAP.

2. **Define Monitoring Policies**:
   - Go to **Protect > Security Policies**.
   - Create a new policy:
     - **Name**: PostgreSQL Monitoring
     - **Rules**: Log SQL queries (e.g., SELECT, INSERT, UPDATE, DELETE) and data changes (DML operations).
     - Example rule: Log all `UPDATE` statements affecting the `appdb` database.
   - Save and apply the policy.

---

## Step 5: Verify and Test the Monitoring
Ensure Guardium is capturing database activity and data changes as expected.

1. **Generate Database Activity**:
   - Connect to the PostgreSQL database (e.g., using `psql` or a client tool):
     ```bash
     oc exec -it <postgresql-pod-name> -- psql -U appuser -d appdb
     ```
   - Run sample queries:
     ```sql
     INSERT INTO my_table (column1) VALUES ('test');
     UPDATE my_table SET column1 = 'updated' WHERE column1 = 'test';
     SELECT * FROM my_table;
     ```

2. **Check Guardium Reports**:
   - In the Guardium console, go to **Monitor > Reports**.
   - Run a report (e.g., “Database Activity Report”) to verify that the SQL statements and data changes are logged.

---

## Additional Notes
- **Scalability**: If your PostgreSQL deployment scales across multiple pods, ensure External S-TAP is configured to monitor all relevant services or use a daemonset for broader coverage.
- **EDB Specifics**: Since EDB PostgreSQL is an enhanced version of PostgreSQL, the monitoring setup remains the same unless EDB provides custom integrations (check EDB documentation).
- **Security**: Secure communication between External S-TAP and the Guardium collector using TLS if required.

---

This guide provides a complete setup to monitor database activity and data changes in EDB PostgreSQL on OpenShift using IBM Guardium. Adjust configurations as needed based on your specific environment.
