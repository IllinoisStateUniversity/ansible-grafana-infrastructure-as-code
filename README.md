# Grafana - Infrastructure as Code

This Ansible playbook automates the deployment and configuration of Grafana on OpenShift, leveraging Infrastructure as Code principles. It is designed to run within Ansible Automation Platform (AAP)/AWX and handles the following tasks:

- Retrieval of secrets from Keeper Secrets Manager
- Renewal and management of SSL certificates using a custom private ACME Ansible collection
- Deployment of the Grafana Operator and Grafana instance
- Configuration of Grafana with OIDC authentication
- Deployment of Grafana datasources and dashboards
- Creation of OpenShift routes for Grafana with SSL termination

## Table of Contents

- [Prerequisites](#prerequisites)
- [Folder Structure](#folder-structure)
- [Variables](#variables)
- [Secrets Management](#secrets-management)
- [SSL Certificate Management](#ssl-certificate-management)
- [Usage Instructions](#usage-instructions)
- [Playbook Breakdown](#playbook-breakdown)
  - [Tasks Overview](#tasks-overview)
- [Custom Ansible Collection](#custom-ansible-collection)
- [License](#license)

## Prerequisites

- **Ansible Automation Platform (AAP)/AWX**: The playbook is designed to run in AAP/AWX.
- **Collections**: The following Ansible collections are required:
  - `redhat.openshift`
  - `community.crypto`
  - `keepersecurity.keeper_secrets_manager`
  - `illinoisstate.acme` (Custom private collection for ACME certificate management, maybe released in the future)
- **OpenShift Cluster**: Access to an OpenShift cluster where Grafana will be deployed.
- **Keeper Secrets Manager**: Used for securely retrieving secrets (accounts, passwords, certificates).
- **ACME Account**: For SSL certificate renewal.

## Folder Structure

```bash
├── site.yml
├── vars/
│   ├── prod.yml
│   └── test.yml
├── templates/
│   └── grafana-dashboards.yml.j2
├── dashboards/
│   └── (Place your Grafana dashboard JSON files here)
├── collections/
│   └── requirements.yml
```

## Variables

The playbook uses environment-specific variables located in the `vars/` directory:

- `prod.yml`: Variables for the production environment.
- `test.yml`: Variables for the test environment.

Variables include OpenShift API URLs, namespaces, route URLs, and OIDC configurations. Modify these files according to your environment settings.

## Secrets Management

The playbook retrieves secrets from Keeper Secrets Manager. Ensure that the following records exist:

- **OpenShift Local Admin Account**: `openshift-<environment>-svc-ansible`
- **Grafana Local Admin Account**: `grafana-<environment>-admin`
- **Grafana OIDC Account**: `grafana-<environment>-oidc`
- **SSL Certificate Records**:
  - For production: `ssl-certificate-grafana-example-com`
  - For test: `ssl-certificate-grafana-test-example-com`

## SSL Certificate Management

The playbook attempts to renew the Grafana SSL certificate using a custom private Ansible collection `illinoisstate.acme`. Ensure that:

- The ACME account is set up and the credentials are stored in Keeper Secrets Manager under the title `acme-login`.
- The private key field in Keeper is named `keyPair`.

## Usage Instructions

### 1. Place Grafana Dashboards

Place your Grafana dashboard JSON files in the `dashboards/` directory. These will be deployed to Grafana.

### 2. Configure Variables

Modify the variables in `vars/prod.yml` and `vars/test.yml` according to your environment.

### 3. Configure Secrets

Ensure that the Keeper Secrets Manager is set up with the required secrets as mentioned in [Secrets Management](#secrets-management).

### 4. Run the Playbook in AAP/AWX

#### a. Create a New Project

In AAP/AWX, create a new project and point it to the repository containing this playbook.

#### b. Create a New Job Template

- **Name**: Grafana Deployment
- **Job Type**: Run
- **Inventory**: Use `localhost` or an inventory that includes `localhost`.
- **Project**: Select the project you created.
- **Playbook**: `site.yml`
- **Credentials**: Add any necessary credentials, including:
  - **Keeper Secrets Manager** credentials
  - **OpenShift Namespace Admin Service Account** credentials
- **Extra Variables**:
  - Set `grafana_environment` to either `prod` or `test` depending on your target environment.

#### c. Run the Job Template

Execute the job template to run the playbook.

## Playbook Breakdown

### Tasks Overview

1. **Get OpenShift Local Admin Account from Keeper**

   Retrieves OpenShift admin credentials from Keeper Secrets Manager.

2. **Get Grafana Local Admin Account from Keeper**

   Retrieves Grafana admin credentials from Keeper Secrets Manager.

3. **Get Grafana OIDC Account from Keeper**

   Retrieves Grafana OIDC client credentials from Keeper Secrets Manager.

4. **Attempt to Renew the Grafana SSL Certificate**

   Invokes the `illinoisstate.acme` role to renew the SSL certificate.

5. **Get Grafana Certificate Record from Keeper**

   Retrieves the renewed SSL certificate from Keeper Secrets Manager.

6. **Decrypt SSL Private Key**

   Decrypts the SSL private key and saves it to a temporary file on the Execution Environment.

7. **Login to OpenShift**

   Logs into OpenShift using the admin credentials to obtain an API token.

8. **Get API Token for OpenShift**

   Retrieves the API token for subsequent OpenShift API calls.

9. **Deploy the Grafana Operator**

   Deploys the Grafana Operator in the specified OpenShift namespace using `OperatorGroup` and `Subscription` resources.

10. **Deploy Grafana Instance**

    Deploys and configures the Grafana instance, including OIDC authentication and persistent storage.

11. **Deploy Grafana Data Source - Mimir**

    Deploys the Grafana datasource for Mimir, configuring it with the necessary parameters.

12. **Deploy Grafana Dashboards**

    Deploys Grafana dashboards from JSON files in the `dashboards/` directory using a Jinja2 template.

13. **Deploy OpenShift Route for Grafana**

    Creates a route in OpenShift with SSL termination for Grafana, using the certificate and key retrieved earlier.
