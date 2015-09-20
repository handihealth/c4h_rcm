# NHS Code4Health: Ripple Remote Chemotherapy Monitoring Challenge

## The RCM Remote Chemotherapy Monitoring scenario 

This scenario represents the key clinical information typically recorded and updated as part of routine monitoring of patients who are on chemotherapy.

[Detailed description of scenario](/docs/scenarios/RemoteChemotherapyMonitoringScenario.pdf)

[Workflow diagram](/docs/scenarios/RemoteChemotherapyMonitoring.pdf)

### What has been pre-prepared

1. A Code4Health [Ehrscape](https://ehrscape.code-4-health.org) domain with access to the Marand [EhrExplorer](https://ehrscape.code-4-health.org/explorer) view of the c4h_ripple_rcm domain. This will allow templates to be viewed and AQL queries to be constructed.

2. A [Workspace.md](/workspace.md) file containing details of key domain identifiers and assets e.g. dummy patient identifiers, template names, login and password details for the c4h_rcm domain.

3. A [local repository](/models) of openEHR archetypes and templates. These archetypes and templates have assembled partly from remote repositories such as the [openEHR Foundation CKM](http://openehr.org/ckm) repository, or [UK Clinical Models](http://clinicmodels.org.uk) repository. Other artefacts, in particular the openEHR templates, have been authored locally and are maintained along with copies of the required archetypes in this Git repository. The two key templates are ..

	``Ripple RCM Patient Monitoring Report`` - the regular report sent by the patient to the monitoring unit.

	``FBC Laboratory panel report (SNOMED CT)`` - the blood test results sent to the patient.

	The ‘operational templates’ (effectively the compiled version 	of each template) have been uploaded to the RCM domain along with some dummy data, with the [instance examples](/technical/instance/ripple_rcm) being available.

The templates are most easily viewed via the Marand [EhrExplorer](https://ehrscape.code-4-health.org/explorer) tool.

4. A [Postman](https://www.getpostman.com/) [Collections](/technical/postman/NHS%20Code4Health%20Ehrscape%20Master.json.postman_collection) file which can be imported to provide easy access to the Code4Health Ehrscape API.

5. A [Postman](https://www.getpostman.com/) [Environment](/technical/postman/C4H%20Ripple%20RCM.postman_environment) file for the `c4h_ripple_rcm domain`, containing a number of preset variables.

6. A [Technical tasks guide](/docs/scenarios/ripple_rcm_tech_tasks.md), describing the key tasks involved in getting up and running with Ehrscape for this clinical scenario

7. An [Overview of openEHR and Ehrscape guide](/docs/openehr/openehr_intro.md) for those new to openEHR concepts.
