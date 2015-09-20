## Ripple RCM - Remote Chemotherapy Technical guide

This document describes the series of [Code4Health Ehrscape API](https://ehrscape.code-4-health.org/api-explorer.html) calls required for the LCR Demo Summary app and assumes a base level of understanding of that API and the use of openEHR - further details can be found at [Overview of openEHR and Ehrscape](/docs/training/openehr_intro.md).

The steps covered are...  

* Open the Ehrscape Session

* Retrieve Patient's ehrId from Ehrscape, based on their `subjectId` (Hospital Number)

* Retrieve the compositionID of Patient's most recent RCM Patient Monitoring composition
* Retrieve the most recent RCM Patient Monitoring composition
* Persist a new  RCM Patient Monitoring composition as a POST i.e. create a new instance of the document.
* Run an AQL query which returns a 'flat list' of key datapoints from the 3 most recent Full blood counts for read-only display.

* Close the Ehrscape Session

* Other API services
 * Access the Indizen SNOMED CT Terminology browser service  

The baseURL for all Ehrscape API calls is https://ehrscape.code-4-health.org

### API parameter convenstions

In the descriptions below we use the convention

`{{Password}}`

to indicate that a local variable or parameter should be substituted

The baseURL for all Ehrscape API calls is https://ehrscape.code-4-health.org

i.e. the call below, described as
`POST /rest/v1/session?username={{userName}}&password={{password}}`

where
```
login=myylogin
password = mypassword
```
should be resolved to
```
https://ehrscape.code-4-health.org/rest/v1/session?username=mylogin&password=mypassword
```

### A. Open the Ehrscape session

The first step in working with Ehrscape is to open a Session and retrieve the ``sessionId`` token. This allows subsequent API calls to be made without needing to login on each occasion.  
The session should be formally closed when you are finished.

##### Call: Create a new openEHR session:
 ````
 POST /rest/v1/session?username={{userName}}&password={{password}}
 ````
##### Returns:
````json
{
"sessionId": "fc234d24-7b59-49f5-a724-8d37072e832b"
}

````

### B. Retrieve Patient’s ehrId from Ehrscape based on their subjectId

We now need to retrieve the patient’s internal `ehrID` associated with their subjectId. The ehrID is a unique string which, for security reasons, cannot be associated with the patient, if for instance their openEHR records were leaked.

##### Call: Returns the EHR for the specified subject ID and namespace.
````
GET /rest/v1/ehr/?subjectId={{subjectId}}&subjectNamespace={{subjectNamespace}}
Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId
````
##### Return:
````son
{
  “ehrId”: “0da489ee-c0ae-4653-9074-57b7f63c9f16”
}
````

### C. Retrieve the compositionId of Patient’s most recent Remote Chemotherapy Monitoring Report composition

Now that we have the patient’s `ehrId` we can use it to locate their existing records.
We use an Archetype Query Language (AQL) call to retrieve a list of the identifiers and dates of existing Remote Chemotherapy Monitoring Report ``composition`` records. Compositions are document-level records which act as the container for all openEHR patient data.

The `name/value` of the Composition is the root name of the templates composition archetype (case-sensitive). In a real-world example we would query on other factors to ensure we had the ‘correct’ list.

##### AQL statement

````
select
    a/uid/value as compositionId,
    a/context/start_time/value as start_time
from EHR e[ehr_id/value=‘{{ehrId}}’]
contains COMPOSITION a[openEHR-EHR-COMPOSITION.report.v1]
where a/name/value=‘Patient Remote Chemo monitoring’
order by a/context/start_time/value desc
offset 0 limit 1
````
The query API call returns a `results` which is a nested set of name/value pairs whose format is determined by the AQL query.

The `compositionId` element in the response is the unique identifier for the composition and `start_time` is the time that the document was authored.

##### Call: Run AQL query and return a Resultset
````
GET /rest/v1/query?aql=select a/uid/value as compositionId, a/context/start_time/value as start_time from EHR e[ehr_id/value=‘7dfbbeab-d49e-42bf-9f6e-a23d5b55e812’] contains COMPOSITION a[openEHR-EHR-COMPOSITION.report.v1] where a/name/value=‘Patient Remote Chemo monitoring’ order by a/context/start_time/value desc offset 0 limit 1
Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId
````

##### Return:
````json
{
  “meta”: {
    “href”: “https://ehrscape.code-4-health.org/rest/v1/query/?aql=select%20%20%20%20%20a/uid/value%20as%20compositionId,%20%20%20%20%20a/context/start_time/value%20as%20start_time%20from%20EHR%20e%5Behr_id/value%3D'5d685333-147f-4aa0-bf1d-e746a5d846b1’%5D%20contains%20COMPOSITION%20a%5BopenEHR-EHR-COMPOSITION.report.v1%5D%20where%20a/name/value%3D’Patient%20Remote%20Chemo%20monitoring’%20order%20by%20a/context/start_time/value%20desc%20offset%200%20limit%201”
  },
  “all”: “select     a/uid/value as compositionId,     a/context/start_time/value as start_time from EHR e[ehr_id/value=‘5d685333-147f-4aa0-bf1d-e746a5d846b1’] contains COMPOSITION a[openEHR-EHR-COMPOSITION.report.v1] where a/name/value=‘Patient Remote Chemo monitoring’ order by a/context/start_time/value desc offset 0 limit 1”,
  “executedAql”: “select     a/uid/value as compositionId,     a/context/start_time/value as start_time from EHR e[ehr_id/value=‘5d685333-147f-4aa0-bf1d-e746a5d846b1’] contains COMPOSITION a[openEHR-EHR-COMPOSITION.report.v1] where a/name/value=‘Patient Remote Chemo monitoring’ order by a/context/start_time/value desc offset 0 limit 1”,
  “resultSet”: [
    {
      “start_time”: “2015-09-08T22:02:07.706+02:00”,
      “compositionId”: “8263bdaf-a9de-4141-98e5-ae912078b2d4::c4h_ripple_rcm.ehrscape.com::1”
    }
  ]
}
````

### C. Retrieve the Patient’s most recent ‘Patient Remote Chemo monitoring’ composition

We will use the results of the previous query to retrieve one of the compositions via its ‘’compositionId’’.

##### Call: Returns the specified Composition in FLAT JSON format.
````
GET /rest/v1/composition/435e6808-9a7e-4805-bef6-a20c28ba48cb::c4h_rcm.ehrscape.com::2?format=FLAT
Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId
 Content-Type: application/json
````
##### Return:
````json
{
    “meta”: {
        “href”: “https://ehrscape.code-4-health.org/rest/v1/composition/8263bdaf-a9de-4141-98e5-ae912078b2d4::c4h_ripple_rcm.ehrscape.com::1”
    },
    “format”: “FLAT”,
    “templateId”: “Ripple RCM - Chemo monitoring Report”,
    “composition”: {
        “patient_remote_chemo_monitoring/_uid”: “8263bdaf-a9de-4141-98e5-ae912078b2d4::c4h_ripple_rcm.ehrscape.com::1”,
        “patient_remote_chemo_monitoring/context/start_time”: “2015-09-08T20:02:07.706Z”,
        “patient_remote_chemo_monitoring/context/setting|238”: true,
        “patient_remote_chemo_monitoring/context/setting|code”: “238”,
        “patient_remote_chemo_monitoring/context/setting|value”: “other care”,
        “patient_remote_chemo_monitoring/context/setting|terminology”: “opener”,
        “patient_remote_chemo_monitoring/pulse_heart_beat/any_event:0/heart_rate|magnitude”: 9,
        “patient_remote_chemo_monitoring/pulse_heart_beat/any_event:0/heart_rate|unit”: “/min”,
        “patient_remote_chemo_monitoring/pulse_heart_beat/any_event:0/time”: “2015-09-08T20:02:07.706Z”,
        “patient_remote_chemo_monitoring/indirect_oximetry/any_event:0/spo2|numerator”: 92,
        “patient_remote_chemo_monitoring/indirect_oximetry/any_event:0/spo2|denominator”: 100,
        “patient_remote_chemo_monitoring/indirect_oximetry/any_event:0/spo2”: 0.92,
        “patient_remote_chemo_monitoring/indirect_oximetry/any_event:0/time”: “2015-09-08T20:02:07.706Z”,
        “patient_remote_chemo_monitoring/respirations/any_event:0/rate|magnitude”: 25,
        “patient_remote_chemo_monitoring/respirations/any_event:0/rate|unit”: “/min”,
        “patient_remote_chemo_monitoring/respirations/any_event:0/time”: “2015-09-08T20:02:07.706Z”,
        “patient_remote_chemo_monitoring/body_temperature/any_event:0/temperature|magnitude”: 37.4,
        “patient_remote_chemo_monitoring/body_temperature/any_event:0/temperature|unit”: “°C”,
        “patient_remote_chemo_monitoring/body_temperature/any_event:0/time”: “2015-09-08T20:02:07.706Z”,
        “patient_remote_chemo_monitoring/blood_pressure/any_event:0/systolic|magnitude”: 120,
        “patient_remote_chemo_monitoring/blood_pressure/any_event:0/systolic|unit”: “mm[Hg]”,
        “patient_remote_chemo_monitoring/blood_pressure/any_event:0/diastolic|magnitude”: 76,
        “patient_remote_chemo_monitoring/blood_pressure/any_event:0/diastolic|unit”: “mm[Hg]”,
        “patient_remote_chemo_monitoring/blood_pressure/any_event:0/time”: “2015-09-08T20:02:07.706Z”,
        “patient_remote_chemo_monitoring/symptoms/comments”: “My joints ache and I feel sick”,
        “patient_remote_chemo_monitoring/symptoms/symptom:0/symptom_name|22253000”: true,
        “patient_remote_chemo_monitoring/symptoms/symptom:0/symptom_name|code”: “22253000”,
        “patient_remote_chemo_monitoring/symptoms/symptom:0/symptom_name|value”: “Pain”,
        “patient_remote_chemo_monitoring/symptoms/symptom:0/symptom_name|terminology”: “snomed-ct”,
        “patient_remote_chemo_monitoring/symptoms/symptom:1/symptom_name|422587007”: true,
        “patient_remote_chemo_monitoring/symptoms/symptom:1/symptom_name|code”: “422587007”,
        “patient_remote_chemo_monitoring/symptoms/symptom:1/symptom_name|value”: “Nausea”,
        “patient_remote_chemo_monitoring/symptoms/symptom:1/symptom_name|terminology”: “snomed-ct”,
        “patient_remote_chemo_monitoring/symptoms/time”: “2015-09-08T20:02:07.706Z”,
        “patient_remote_chemo_monitoring/howru_score/pain_or_discomfort|code”: “at0042”,
        “patient_remote_chemo_monitoring/howru_score/pain_or_discomfort|value”: “quite a lot”,
        “patient_remote_chemo_monitoring/howru_score/pain_or_discomfort|ordinal”: 2,
        “patient_remote_chemo_monitoring/howru_score/feeling_low_or_worried|code”: “at0043”,
        “patient_remote_chemo_monitoring/howru_score/feeling_low_or_worried|value”: “extreme”,
        “patient_remote_chemo_monitoring/howru_score/feeling_low_or_worried|ordinal”: 3,
        “patient_remote_chemo_monitoring/howru_score/limited_in_what_i_can_do|code”: “at0043”,
        “patient_remote_chemo_monitoring/howru_score/limited_in_what_i_can_do|value”: “extreme”,
        “patient_remote_chemo_monitoring/howru_score/limited_in_what_i_can_do|ordinal”: 3,
        “patient_remote_chemo_monitoring/howru_score/dependent_on_others|code”: “at0042”,
        “patient_remote_chemo_monitoring/howru_score/dependent_on_others|value”: “quite a lot”,
        “patient_remote_chemo_monitoring/howru_score/dependent_on_others|ordinal”: 2,
        “patient_remote_chemo_monitoring/howru_score/summary_score”: 10,
        “patient_remote_chemo_monitoring/howru_score/time”: “2015-09-08T20:02:07.706Z”,
        “patient_remote_chemo_monitoring/composer|name”: “Hazel Smith”
    },
    “deleted”: false,
    “lastVersion”: true
}
````

### D. Persist an new instance of the Patient Remote Chemo monitoring Composition as a POST

All openEHR data is persisted as a COMPOSITION (document) class. openEHR data can be highly structured and potentially complex. To simplify the challenge of persisting openEHR data, examples of  ‘target composition’ data instances have been provided in the Ehrscape ``FLAT JSON`` format.

Once the data is assembled in the correct format, the actual service call is very simple requiring only the setting of simple parameters and headers.

[Example JSON Composition](technical/instances/c4h_ripple_rcm/Ripple%20RCM%20-%20Chemo%20monitoring%20Report%20-%20FLAT_0.json)  

In this scenario every Encounter merits a separate instance of the document and a POST is used, with PUT updates only being used to correct errors or to perform ‘sign-off’.

In other scenarios we might want to, for example, **maintain only a single instance of an Allergies List** and so will use a PUT call to update the previous version. The previous version remains available for audit purposes but would not normally be found by routine querying.
**When a PUT call is used the compositionId that is passed in must include the domain and version suffix  of the last version**

e.g. `798e27b1-f2e8-48c3-8ced-42d4d27d1db3::answer.hopd.com::2`

##### Call: Creates a new openEHR composition and returns the new CompositionId
````
POST /rest/v1/composition?ehrId=7dfbbeab-d49e-42bf-9f6e-a23d5b55e812&templateId=Ripple RCM - Chemo monitoring Report&committerName=handi&format=FLAT HTTP/1.1

Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId
 Content-Type: application/json
````
````json
{
    “ctx/language”: “en”,
    “ctx/territory”: “GB”,
    “ctx/composer_name”: “Hazel Smith”,
    “ctx/time”: “2015-09-08T22:02:07.706+02:00”,
    “ctx/id_namespace”: “NHSE”,
    “ctx/id_scheme”: “NHSE”,
    “patient_remote_chemo_monitoring/pulse_heart_beat:0/any_event:0/heart_rate|magnitude”: 9,
    “patient_remote_chemo_monitoring/pulse_heart_beat:0/any_event:0/heart_rate|unit”: “/min”,
    “patient_remote_chemo_monitoring/indirect_oximetry:0/any_event:0/spo2|numerator”: 92,
    “patient_remote_chemo_monitoring/indirect_oximetry:0/any_event:0/spo2|denominator”: 100,
    “patient_remote_chemo_monitoring/respirations:0/any_event:0/rate|magnitude”: 25,
    “patient_remote_chemo_monitoring/respirations:0/any_event:0/rate|unit”: “/min”,
    “patient_remote_chemo_monitoring/body_temperature:0/any_event:0/temperature|magnitude”: 37.4,
    “patient_remote_chemo_monitoring/body_temperature:0/any_event:0/temperature|unit”: “°C”,
    “patient_remote_chemo_monitoring/blood_pressure:0/any_event:0/systolic|magnitude”: 120,
    “patient_remote_chemo_monitoring/blood_pressure:0/any_event:0/systolic|unit”: “mm[Hg]”,
    “patient_remote_chemo_monitoring/blood_pressure:0/any_event:0/diastolic|magnitude”: 76,
    “patient_remote_chemo_monitoring/blood_pressure:0/any_event:0/diastolic|unit”: “mm[Hg]”,
    “patient_remote_chemo_monitoring/symptoms/comments”: “My joints ache and I feel sick”,
    “patient_remote_chemo_monitoring/symptoms/symptom:0/symptom_name|code”: “22253000”,
    “patient_remote_chemo_monitoring/symptoms/symptom:0/symptom_name|value”: “Pain”,
    “patient_remote_chemo_monitoring/symptoms/symptom:1/symptom_name|code”: “422587007”,
    “patient_remote_chemo_monitoring/symptoms/symptom:1/symptom_name|value”: “Nausea”,
	  “patient_remote_chemo_monitoring/howru_score/pain_or_discomfort|code”: “at0042”,
	   “patient_remote_chemo_monitoring/howru_score/pain_or_discomfort|value”: “quite a lot”,
	   “patient_remote_chemo_monitoring/howru_score/pain_or_discomfort|ordinal”: 2,
	   “patient_remote_chemo_monitoring/howru_score/feeling_low_or_worried|code”: “at0043”,
	   “patient_remote_chemo_monitoring/howru_score/feeling_low_or_worried|value”: “extreme”,
	   “patient_remote_chemo_monitoring/howru_score/feeling_low_or_worried|ordinal”: 3,
	   “patient_remote_chemo_monitoring/howru_score/limited_in_what_i_can_do|code”: “at0043”,
	   “patient_remote_chemo_monitoring/howru_score/limited_in_what_i_can_do|value”: “extreme”,
	   “patient_remote_chemo_monitoring/howru_score/limited_in_what_i_can_do|ordinal”: 3,
	   “patient_remote_chemo_monitoring/howru_score/dependent_on_others|code”: “at0042”,
	   “patient_remote_chemo_monitoring/howru_score/dependent_on_others|value”: “quite a lot”,
	   “patient_remote_chemo_monitoring/howru_score/dependent_on_others|ordinal”: 2,
	   “patient_remote_chemo_monitoring/howru_score/summary_score”: 10
}
````

  ##### Return:
  ````json
  {
      “meta”: {
          “href”: “https://ehrscape.code-4-health.org/rest/v1/composition/c811104e-1f69-4405-afe0-182654063c9c::c4h_rcm.ehrscape.com::1”
      },
      “action”: “CREATE”,
      “compositionUid”: “c811104e-1f69-4405-afe0-182654063c9c::c4h_rcm.ehrscape.com::1”
  }
````

### D. Run an AQL query with key values

This AQL query eturns a flattened Resultset with key values from recent compositions.  

##### Call: Run AQL query and return a Resultset

This AQL call returns a set of key data points from recent Patient Remote Chemo monitoring Compositions. The `results’ is a simple nested name/value pair structure whose exact format is determined by the query itself. Either single data points or objects can be  returned.

````json
GET /rest/v1/query?aql=select     b_a/data[at0002]/events[at0003]/data[at0001]/items[at0004]/value/magnitude as Heart_Rate_magnitude,     b_b/data[at0001]/events[at0002]/data[at0003]/items[at0006]/value/numerator as spO2_numerator,     b_c/data[at0002]/events[at0003]/data[at0001]/items[at0004]/value/magnitude as Temperature_magnitude,     b_d/data[at0001]/events[at0006]/data[at0003]/items[at0004]/value/magnitude as Systolic_magnitude,     b_d/data[at0001]/events[at0006]/data[at0003]/items[at0005]/value/magnitude as Diastolic_magnitude,     b_f as Symptom,     b_g/data[at0001]/events[at0002]/data[at0003]/items[at0004, ‘Comments’]/value as Comments,     a/uid/value as did,     a/context/start_time/value as Date from EHR e[ehr_id/value=‘7dfbbeab-d49e-42bf-9f6e-a23d5b55e812’] contains COMPOSITION a[openEHR-EHR-COMPOSITION.report.v1] contains (     OBSERVATION b_a[openEHR-EHR-OBSERVATION.pulse.v0] or     OBSERVATION b_b[openEHR-EHR-OBSERVATION.indirect_oximetry.v1] or     OBSERVATION b_c[openEHR-EHR-OBSERVATION.body_temperature.v1] or     OBSERVATION b_d[openEHR-EHR-OBSERVATION.blood_pressure.v1] or     CLUSTER b_f[openEHR-EHR-CLUSTER.symptom.v0] or     OBSERVATION b_g[openEHR-EHR-OBSERVATION.story.v1]) where a/name/value=‘Patient Remote Chemo monitoring’ order by a/context/start_time/value desc offset 0 limit 100

Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId
````

##### Return:
````json

“resultSet”: [
  {
    “did”: “435e6808-9a7e-4805-bef6-a20c28ba48cb::c4h_rcm.ehrscape.com::2”,
    “spO2_numerator”: 92,
    “Comments”: {
      “@class”: “DV_TEXT”,
      “value”: “My joints ache and I feel sick”
    },
    “Diastolic_magnitude”: 76,
    “Heart_Rate_magnitude”: 9,
    “Systolic_magnitude”: 120,
    “Temperature_magnitude”: 37.4,
    “Symptom”: {
      “@class”: “CLUSTER”,
      “name”: {
        “@class”: “DV_TEXT”,
        “value”: “Symptom”
      },
      “archetype_details”: {
        “@class”: “ARCHETYPED”,
        “archetype_id”: {
          “@class”: “ARCHETYPE_ID”,
          “value”: “openEHR-EHR-CLUSTER.symptom.v0”
        },
        “rm_version”: “1.0.1”
      },
      “archetype_node_id”: “openEHR-EHR-CLUSTER.symptom.v0”,
      “items”: [
        {
          “@class”: “ELEMENT”,
          “name”: {
            “@class”: “DV_TEXT”,
            “value”: “Symptom name”
          },
          “archetype_node_id”: “at0001”,
          “value”: {
            “@class”: “DV_CODED_TEXT”,
            “value”: “Pain”,
            “defining_code”: {
              “@class”: “CODE_PHRASE”,
              “terminology_id”: {
                “@class”: “TERMINOLOGY_ID”,
                “value”: “snomed-ct”
              },
              “code_string”: “22253000”
            }
          }
        }
      ]
    },
    “Date”: “2015-09-08T20:02:07.706Z”
  },
  {
    “did”: “435e6808-9a7e-4805-bef6-a20c28ba48cb::c4h_rcm.ehrscape.com::2”,
    “spO2_numerator”: 92,
    “Comments”: {
      “@class”: “DV_TEXT”,
      “value”: “My joints ache and I feel sick”
    },
    “Diastolic_magnitude”: 76,
    “Heart_Rate_magnitude”: 9,
    “Systolic_magnitude”: 120,
    “Temperature_magnitude”: 37.4,
    “Symptom”: {
      “@class”: “CLUSTER”,
      “name”: {
        “@class”: “DV_TEXT”,
        “value”: “Symptom #2”
      },
      “archetype_details”: {
        “@class”: “ARCHETYPED”,
        “archetype_id”: {
          “@class”: “ARCHETYPE_ID”,
          “value”: “openEHR-EHR-CLUSTER.symptom.v0”
        },
        “rm_version”: “1.0.1”
      },
      “archetype_node_id”: “openEHR-EHR-CLUSTER.symptom.v0”,
      “items”: [
        {
          “@class”: “ELEMENT”,
          “name”: {
            “@class”: “DV_TEXT”,
            “value”: “Symptom name”
          },
          “archetype_node_id”: “at0001”,
          “value”: {
            “@class”: “DV_CODED_TEXT”,
            “value”: “Nausea”,
            “defining_code”: {
              “@class”: “CODE_PHRASE”,
              “terminology_id”: {
                “@class”: “TERMINOLOGY_ID”,
                “value”: “snomed-ct”
              },
              “code_string”: “422587007”
            }
          }
        }
      ]
    },
    “Date”: “2015-09-08T20:02:07.706Z”
  },
  {
    “did”: “72899599-672b-4115-b34d-10698fe53fb7::c4h_rcm.ehrscape.com::1”,
    “spO2_numerator”: 92,
    “Comments”: {
      “@class”: “DV_TEXT”,
      “value”: “My joints ache a bit”
    },
    “Diastolic_magnitude”: 76,
    “Heart_Rate_magnitude”: 91,
    “Systolic_magnitude”: 120,
    “Temperature_magnitude”: 37.4,
    “Symptom”: {
      “@class”: “CLUSTER”,
      “name”: {
        “@class”: “DV_TEXT”,
        “value”: “Symptom”
      },
      “archetype_details”: {
        “@class”: “ARCHETYPED”,
        “archetype_id”: {
          “@class”: “ARCHETYPE_ID”,
          “value”: “openEHR-EHR-CLUSTER.symptom.v0”
        },
        “rm_version”: “1.0.1”
      },
      “archetype_node_id”: “openEHR-EHR-CLUSTER.symptom.v0”,
      “items”: [
        {
          “@class”: “ELEMENT”,
          “name”: {
            “@class”: “DV_TEXT”,
            “value”: “Symptom name”
          },
          “archetype_node_id”: “at0001”,
          “value”: {
            “@class”: “DV_CODED_TEXT”,
            “value”: “Pain”,
            “defining_code”: {
              “@class”: “CODE_PHRASE”,
              “terminology_id”: {
                “@class”: “TERMINOLOGY_ID”,
                “value”: “snomed-ct”
              },
              “code_string”: “22253000”
            }
          }
        }
      ]
    },
    “Date”: “2015-09-08T17:52:07.706Z”
  },
  {
    “did”: “c811104e-1f69-4405-afe0-182654063c9c::c4h_rcm.ehrscape.com::1”,
    “spO2_numerator”: 98,
    “Comments”: {
      “@class”: “DV_TEXT”,
      “value”: “Feeling fine”
    },
    “Diastolic_magnitude”: 80,
    “Heart_Rate_magnitude”: 80,
    “Systolic_magnitude”: 122,
    “Temperature_magnitude”: 37,
    “Symptom”: null,
    “Date”: “2015-09-01T21:52:07.706Z”
  },
  {
    “did”: “c811104e-1f69-4405-afe0-182654063c9c::c4h_rcm.ehrscape.com::1”,
    “spO2_numerator”: 98,
    “Comments”: {
      “@class”: “DV_TEXT”,
      “value”: “Feeling fine”
    },
    “Diastolic_magnitude”: 72,
    “Heart_Rate_magnitude”: 80,
    “Systolic_magnitude”: 118,
    “Temperature_magnitude”: 37,
    “Symptom”: null,
    “Date”: “2015-09-01T21:52:07.706Z”
  },
  {
    “did”: “c811104e-1f69-4405-afe0-182654063c9c::c4h_rcm.ehrscape.com::1”,
    “spO2_numerator”: 98,
    “Comments”: {
      “@class”: “DV_TEXT”,
      “value”: “Feeling fine”
    },
    “Diastolic_magnitude”: 76,
    “Heart_Rate_magnitude”: 80,
    “Systolic_magnitude”: 124,
    “Temperature_magnitude”: 37,
    “Symptom”: null,
    “Date”: “2015-09-01T21:52:07.706Z”
  },
  {
    “did”: “4e161f95-ae20-4006-98a3-25d0eb5a4f41::c4h_rcm.ehrscape.com::1”,
    “spO2_numerator”: 98,
    “Comments”: {
      “@class”: “DV_TEXT”,
      “value”: “Feeling fine”
    },
    “Diastolic_magnitude”: 80,
    “Heart_Rate_magnitude”: 80,
    “Systolic_magnitude”: 122,
    “Temperature_magnitude”: 37,
    “Symptom”: null,
    “Date”: “2015-09-01T17:52:07.706Z”
  },
  {
    “did”: “4e161f95-ae20-4006-98a3-25d0eb5a4f41::c4h_rcm.ehrscape.com::1”,
    “spO2_numerator”: 98,
    “Comments”: {
      “@class”: “DV_TEXT”,
      “value”: “Feeling fine”
    },
    “Diastolic_magnitude”: 72,
    “Heart_Rate_magnitude”: 80,
    “Systolic_magnitude”: 118,
    “Temperature_magnitude”: 37,
    “Symptom”: null,
    “Date”: “2015-09-01T17:52:07.706Z”
  },
  {
    “did”: “4e161f95-ae20-4006-98a3-25d0eb5a4f41::c4h_rcm.ehrscape.com::1”,
    “spO2_numerator”: 98,
    “Comments”: {
      “@class”: “DV_TEXT”,
      “value”: “Feeling fine”
    },
    “Diastolic_magnitude”: 76,
    “Heart_Rate_magnitude”: 80,
    “Systolic_magnitude”: 124,
    “Temperature_magnitude”: 37,
    “Symptom”: null,
    “Date”: “2015-09-01T17:52:07.706Z”
  }
]
````

### E. Close the Ehrscape session

The last step in working with Ehrscape is to close the session.

##### Call: Close the openEHR session:
 ````
 DELETE /rest/v1/session?sessionId={{sessionId}}
 ````
##### Returns:
````json
{
  "sessionId": "2dcd6528-0471-4950-82fa-a018272f1339"
}
````
