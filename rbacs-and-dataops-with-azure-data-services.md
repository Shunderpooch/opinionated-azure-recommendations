# Role Based Access Control and DataOps for Azure Data Services

Author: [Arthur Dooner](@mailto:arthurdooner@gmail.com)

Version 1.0, Revised 2021-03-30

## Table of Contents

- [Overview/Guiding Principles](#overview/guiding-principles)
- [Azure Data Lake Storage Gen 2](#azure-data-lake-storage-gen-2)
    - [User Flow Recommendations](#user-flow-recommendations)
    - [System/Integration Runtime Flow Recommendations](#system/integration-runtime-flow-recommendations)
- [Azure Data Factory Security Paradigms](#azure-data-factory-security-paradigms)
    - [Data Factory 1: Production Data Factory](#data-factory-1-production-data-factory)
    - [Data Factory 2: Sandbox Data Factory](#data-factory-2-sandbox-data-factory)
- [On-Premises Databases and Data Services](#on-premises-databases-and-data-services)
- [Azure ML Security Paradigms](#azure-ml-security-paradigms)
    - [Securing Storage Through Dataset Credentials in Azure ML](#securing-storage-through-dataset-credentials-in-azure-ml)
    - [Securing Azure Data Lake Storage Gen 2 with Identity Based Access (Preview)](#securing-azure-data-lake-storage-gen-2-with-identity-based-access-preview)
- [Azure Databricks Security Paradigms](#azure-databricks-security-paradigms)
    - [Working with Azure Data Lake Storage](#working-with-azure-data-lake-storage)
    - [Additional Notes](#additional-notes)
- [Snowflake on Azure Security Paradigms](#snowflake-on-azure-security-paradigms-work-in-progress)

## Overview/Guiding Principles

When discussing Role-Based Access Control (RBAC), we try to follow a _least priviledge_ model of exposing information. What this means is that every user has the _minimum_ amount of access to the systems they need to perform their roles appropriately. You will see the application of this paradigm applys consistently: Systems and automated services will have write-access to the places where they may need to save data, and interactive user-facing systems will not have write access. Users will be able to read relevant data, but not write back to any resource except an isolated, "Sandbox-like" environment.

## Azure Data Lake Storage Gen 2

When reviewing Azure Data Lake Storage Gen 2 (ADLS2), there are some roles that can be assigned to users and systems that are custom to the Storage Account. The roles that should be used primarily are:

- Storage Blob Data Reader
- Storage Blob Data Writer

There are two other roles, focused for admins (Storage Blob Data Owner and Storage Blob Data Contributor), and one for delegating credentials through SAS (Storage Blob Delegator)- [Read more about Storage Account Roles here](https://docs.microsoft.com/en-us/azure/storage/common/storage-auth-aad#assign-azure-roles-for-access-rights).

With ADLS2, the recommendation for the most governable Data Lake follows this structure:

- ``<container tier (bronze, silver, gold)>/<data-source>/<dataset>/<dataset-with-timestamp>``

For example, with a SEC 10K dataset pulled in as raw filings:

- ``bronze/sec/10k/<company-id>/2021-03-30-<company-id>-10k.txt``

That way, access can be restricted per _data tier_ and _dataset source_.

For deeper views on how ACLs work on ADLS2, [Review this Documentation](https://docs.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-access-control)

For more on Data Lake Structure/Data Lake Design, review Databricks' Recommendations on [Data Lake Structure](https://databricks.com/blog/2019/08/14/productionizing-machine-learning-with-delta-lake.html)

### User Flow Recommendations

Users should never have write access to production data. Production here is defined by some resource depending on the Dataset being there in the Data Lake on a consistent basis. They should, however, have access to write to a Sandbox, a non-prod space where they can save their transformations and analyses.

For Production Datasets:

- Group all similar users with the same access rules into one shared Azure Active Directory group.
- For the Containers & Folders in the Data Lake, grant "Read" and "Execute" permissions to each data source the Group should have access to. The Group needs "Execute" access to all leading folders so that the folders can be traversed properly. 
- If they should automatically have access as new datasets get created under a data source, consider also enabling the "Default" permission to also be "Read" and "Execute".

For Sandbox Datasets, ACLs should be configured similarly, since users sharing a Sandbox Data Lake may be able to write data derived from these protected, production datasets. Generally, different user groups that do not have the _exact_ same permissions should not have overlapping access in sandboxes:

- Group all similar users with the same access rules into one shared Azure Active Directory group.
- For the Containers & Folders in the Data Lake, grant "Read", "Write", and "Execute" permissions to each data source the Group should have access to. The Group needs "Execute" access to all leading folders so that the folders can be traversed properly. 
- If they should automatically have access as new datasets get created under a data source, consider also enabling the "Default" permission to also be "Read", "Write", and "Execute".

### System/Integration Runtime Flow Recommendations

Systems/Automated processes should have write access to production data, but _only_ the production data they directly affect. The Data Factory, for example, moving numerous data streams into production daily, will have near-admin priviledges to the Data Lake, but a Service Principal that Azure Databricks uses can be limited to specific places where Databricks/that tool is needed.

For Production Datasets:

- Group all similar processes with the same access rules into one shared Azure Active Directory group.
- For the Containers & Folders in the Data Lake the system needs to Read from, grant "Read" and "Execute" permissions to each data source the Group should have Read access to. The Group needs "Execute" access to all leading folders so that the folders can be traversed properly. 
- For the Containers & Folders in the Data Lake the system needs to Write to, grant "Write" and "Execute" permissions to each data source the Group should have Write access to. The Group needs "Execute" access to all leading folders so that the folders can be traversed properly.
- If they should automatically have access as new datasets get created under a data source, consider also enabling the "Default" permission to also be "Read" and "Execute".
- If they should automatically have write access to those folders, also set the "Default" permission to include "Write".

For Sandbox Datasets, these data pipelines will follow the same structure as production, but will be ad-hoc since users will be configuring these services together  

## Azure Data Factory Security Paradigms

With Data Factory, the tool is designed to be a _central place of orchestration_ for data pipelines across an organization. This does not mean it is the easiest to configure for "Citizen Integrators", however, as Data Factory has a concept of __Linked Services__ that are permanently linked to the resource, these Citizen Integrators would be able to use potentially _production resources with write access_ in a shared Data Factory. Thus, to resolve these security concerns, I recommend two different Data Factory Deployments:

### Data Factory 1: Production Data Factory

The production data factory should be a standalone configuration. Since Data Factory supports Managed Identity, the recommendation is grant Data Factory access to Azure Resources like Data Lake Storage through that managed identity, following the [System/Integration Runtime Flow](#system-integration-runtime-flow-recommendation) for ADLS2. It should be granted Read access to sources it needs to read from, Write for sources it needs to write to. 

As an additional note, services that Data Factory may trigger (Azure Databricks, Azure ML) have independent Service Principals or Managed Identities that will need to be independently configured so that those resources can access the datasets and potentially read or modify data in the cloud. Make sure to configure those appropriately per the requirements of the pipelines you are building.

### Data Factory 2: Sandbox Data Factory

As a way of protecting production data, the "Sandbox" Data Factory can be built as a space for Citizen Integrators to step in, have __Linked Services__ with read-only access to Production resources, and pull those Data resources in as necessary to build their analyses and pipelines. That Data Factory can then be granted write access to a Sandbox Data Lake, so that the users can only save their custom pipelines' outputs to an isolated space, not interfering with any production workflow. 

Citizen Data Integrators can be set as "Data Factory Contributor" role in the resource group of the Sandbox Data Factory. For more info on this configuration, [follow the docs here](https://docs.microsoft.com/en-us/azure/data-factory/concepts-roles-permissions).

Should a Citizen Integrator build a set of Pipelines in Data Factory that _would be integrated to the Production environment_, the configuration, managed in Git, would be pushed to the production Data Factory through a change management process, by a service adminstrator. Variables could then be changed to point at the Production resources, and thus be integrated as a new Production Dataset.

## On-Premises Databases and Data Services

With On-Premises Databases, especially in the context of Self-Hosted Integration Runtimes, the recommendation is to work with each Database team to build out Read-Only permissions on the Database or Data Service for the Data Factory, generally a "Data Factory User". In that way, the Data Factory Integration Runtime can pull all of the data necessary from that source, land it in a Data Lake to be integrated, and configure a schedule to update it from those source systems and transform/integrate it in the Data Lake.

## Azure ML Security Paradigms

With Azure Machine Learning, there are powerful and customizable scopes that can be configured per ML workspace. Users of the same group should be granted access to a set of Compute Instances, Clusters, and Inference Clusters, through customized roles designed [following this guide](https://docs.microsoft.com/en-us/azure/machine-learning/how-to-assign-roles). 

### Securing Storage Through Dataset Credentials in Azure ML

To create Datastores in [Azure ML, follow this guide](https://docs.microsoft.com/en-us/azure/machine-learning/how-to-connect-data-ui#storage-access-and-permissions). However, all of this is through varying Authentication types. Review these [Supported Data Storage Service Types](https://docs.microsoft.com/en-us/azure/machine-learning/how-to-access-data). None of these are individual-based, and are more focused workspace-wide. Thus, the separation of a "Production/Sandbox" environment would have to be setup to keep write priviledges properly protected from end users. Unless the next section is followed, with [Securing Azure Data Lake Storage Gen 2 with Identity Based Access](#securing-azure-data-lake-storage-gen-2-with-identity-based-access-preview)

### Securing Azure Data Lake Storage Gen 2 with Identity Based Access (Preview)

Moving forward and in the future state environment, I would recommend using Identity-Based Datasets so that users can access data in Azure ML, through the ACLs applied to the Data Lake. Follow [this documentation to enable the identity-based functionality](https://docs.microsoft.com/en-us/azure/machine-learning/how-to-identity-based-data-access).

## Azure Databricks Security Paradigms

With Azure Databricks, there are Robust ACLs that can be configured per notebook, job, user, MLFlow experiment, etc. Permissions are managed through Azure Databricks, and similar "Groups" can be designed per security perimeter/people with shared permissions, and they can be granted access to the same artifacts, clusters, etc on a group level. 

_Note_: This robust security paradigm only exists in Azure Databricks Premium.

### Working with Azure Data Lake Storage

The primary recommendation when relying on the Identity Layer of Security in Azure Databricks is to lean on __Azure Data Lake Storage Gen 2__ for the main mounted datastore when working with Databricks. Then, create High-Concurrency clusters with __Credential Passthrough__ enabled. To enable this workflow, follow the [Mounting Azure Data Lake Storage with Credential Passthrough Guide Here](https://docs.microsoft.com/en-us/azure/databricks/security/credential-passthrough/adls-passthrough#--mount-azure-data-lake-storage-to-dbfs-using-credential-passthrough). Follow documentation for ADLS Gen 2.

### Additional Notes

If you want to ensure your notebooks are as secure as possible, enable [Customer Managed Keys for Notebooks](https://docs.microsoft.com/en-us/azure/databricks/security/keys/customer-managed-key-notebook)

If you want to ensure your dbfs artifacts are as secure as possible, enable [Customer Managed Keys for DBFS](https://docs.microsoft.com/en-us/azure/databricks/security/keys/customer-managed-keys-dbfs/)

Finally, if it's necessary, and cluster-worker-node communication should be encrypted in this isolated VNet, turn on [Encrypt Traffic Between Cluster Worker Nodes](https://docs.microsoft.com/en-us/azure/databricks/security/encryption/encrypt-otw)

If you need IP Access Controls over who can access your Databricks workspace on the web, use the [IP Access List API](https://docs.microsoft.com/en-us/azure/databricks/dev-tools/api/latest/ip-access-list) to govern network level access.

Use the [Permissions API](https://docs.microsoft.com/en-us/azure/databricks/dev-tools/api/latest/permissions) to delegate permissions to specific resources programmatically.

## Snowflake on Azure Security Paradigms (Work-in-Progress)

When interfacing with Snowflake to the rest of the Azure platform, I would recommend that roles are built for teams with similarly grouped permissions. Then, permissions are delegated that can be then used in a team's Data Factory, Databricks instance, etc so that each team has auditability as to which of their services are pulling from Snowflake.