# Basic APIs for Imaging Access

## Actors, Roles, and Responsibilities

* SMART on FHIR EHR
  * Authorization Endpoint
    * Provide SMART authorization services
  * Token Introspection Endpoint
    * Support SMART on FHIR token introspection
  * Clinical FHIR Server (EHR)
    * Primary clinical information store, including Patient records, etc.
* (New!) SMART on FHIR Imaging Subsystem
  * ImagingStudy FHIR Endpoint
    * FHIR store either containing ImagingStudy records or having the ability to fetch / generate them
    * Can be hosted within the EHR's FHIR endpoint or separately
  * DICOM WADO Endpoint(s)
    * Imaging data store supporting (at a minimum) WADO access to fetch a study by UID
* (New!) Imaging Client Application

Note that some of actors are separated by responsibility for clarity, but this project makes no assumptions about implementation.  For example, FHIR Imaging records may be stored in the Clinical FHIR Server, the DICOM server endpoint may be a proxy that implements the necessary WADO interface, etc..

## Assumptions and Pre-conditions

* EHR supports SMART on FHIR authorization
* EHR supports SMART on FHIR token introspection
* App knows the EHR's imaging endpoint either through:
  * out-of-band config, or
  * EHR's `.well-known/smart-configuration` includes an `imaging` FHIR base URL
* EHR authorization screen includes support for `patient/*.read` (or `patient/ImagingStudy.read`)
* User agrees to share imaging data with App, during the SMART App "Standalone Launch" flow

## Scaling Considerations

* Passing every "Get Studies for Patient" query through to an underlying PACS might overload these systems. The ImagingStudy FHIR Endpoint can avoid this with approaches like:
  * Caching FHIR ImagingStudies (e.g., with some heuristic for invalidation)
  * Establishing change feeds from the underlying PACS to improve invalidation accuracy
  
## Workflow

* App authorized via SMART on FHIR
* Optional: App queries EHR clinical FHIR server for data
* App queries EHR imaging FHIR server for `ImagingStudy` resource
* App following links from ImagingStudy to fetch DICOM data

```mermaid
sequenceDiagram
  autonumber
  participant AS as Authorization<br/>Server
  participant CFS as Clinical<br/>FHIR Server
  participant IFS as Imaging<br/>FHIR Server
  participant D as DICOM Server
  participant C as Client
  C->>CFS: Get SMART configuration<br/>GET .well-known/smart-configuration
  CFS-->>C: SMART Configuration
  C->>AS: Redirect to {smart-configuration.authorization_endpoint}
  Note over AS: Authorization workflow<br/>with user<br/>(assumes approval)
  AS-->>C: Authorization Code
  C->>AS: Request for Token
  AS-->>C: Token: abc
  C->>CFS: GET Patient/123
  CFS-->>C: Resource: Patient/123
  C->>IFS: GET ImagingStudy/?patient=Patient/123
  IFS->>AS: Introspect Token: abc
  AS-->>IFS: Token valid for Patient/123
  IFS-->>C: Bundle of ImagingStudy resources
  C->>D: GET studies/example-study-uid
  D->>AS: Introspect Token: abc
  AS-->>D: Token valid for Patient/123
  D-->>C: DICOM Imaging Data...
```

Notes:
* Steps `1` through `6` in the diagram represent basic functionality in the SMART App Launch workflow.
* Steps `7` and `8` represent basic FHIR functionality (getting patient information), which should be available in any FHIR implementation.
* Steps `9` and `10` use standard FHIR functionality around the [ImagingStudy](http://hl7.org/fhir/imagingstudy.html) resource.
* In Step `11`, the client is requesting DICOM data via WADO, using the token received during SMART App Launch.
* In Steps `12` and `13`, the DICOM server is performing token introspection to validate the patient context against the request.
* In Step `14`, DICOM data is returned to the client.

## API Calls

### App obtains an access token 

App redirects to `/authorize` and receives an authorization code. Then App calls `POST /token` to obtain:

```
   {
     "expires_in": 3600,
     "access_token": "access-token-value-unguessable",
     "refresh_token": "refresh-token-value-unguessable-and-long-lasting",
     "patient": "123" # FHIR Patient ID ==> https://ehr.example.org/Patient/123
   }
```

Now app has an access token that it can use for clinical + imaging APIs

### App requests imaging studies for patient

    GET https://imaging-api.example.org/ImagingStudy?patient=123&_include=ImagingStudy.endpoint[&_lastUpdated=gt2021-04-17T04:00:00]
    Authorization: Bearer access-token-value-unguessable

If the Imaging FHIR server is distinct from the Clinical FHIR server, it makes an access control decision informed by the EHR's Token Introspection API:

```
   POST https://introspection.internal.example.org/introspect
   token=access-token-value-unguessable
```

The Imaging FHIR server makes sure that 
* the Patient ID from `ImagingStudy?patient={}` matches the patient ID value returned in the introspection response
* the access token is `active`
* the access token includes `patient/ImagingStudy.read` or `patient/*.read` scopes

The Imaging FHIR server MAY looks up additional information about this Patient ID as needed from the EHR (e.g., leveraging Backend Services API access to `GET https://clinical-api.example.org/Patient/123`.) This returns a list of patient Identifiers that the Imaging FHIR server may need to cross-map with its own data.

### Imaging FHIR server responds with data or a "wait up!" status

If the Imaging FHIR server needs to fetch data from sources that will take some time (e.g., issuing a DICOM C-FIND query under the hood), it MAY respond with a `503` status and a `Retry-After` header indicating how many seconds the app should wait before re-trying its query.

If Imaging data is already available (e.g., URLs point to the DICOM server, data has already been cached, etc.), it responds with a FHIR Bundle of ImagingStudy resources, filtering by `Meta.lastUpdated` or `ImagingStudy.started` if client has supplied one of these search parameters. Each resource in the Bundle should populate at least the following elements:

```
  {
    "resourceType": "ImagingStudy",
    "id": "",
    "identifier":{
        "system":"urn:dicom:uid",
	"value":"<Example Study UID>"
    },
    "status": "available",
    "patient": "Patient/123",
    "started": <StudyDate + StudyTime + Timezone Offset>,
    "modality": [<Modalities in Study>],
    "endpoint": {"reference": "#e"} // May be "contained" or external
                 // connectionType = dicom-wado-rs
  }
```

### App requests instance data from wado-rs

The app can construct a series of requests from the $wado-rs URL by appending `/studies/<Example Study UID>`:

```
   GET https://imaging-api.example.org/Patient/123/$wado-rs/studies/example-study-uid
   Accept: multipart/related; type=application/dicom; transfer-syntax=*
   Authorization: Bearer access-token-value-unguessable
```

At a minimum the server SHALL support retrieving a full study when the client requests 

    multipart/related; type=application/dicom; transfer-syntax=*
    
This minimum subset is designed to ensure that even a static server can participate in hosting images. However, additional app features become possible with more capable servers. Servers MAY support additional WADO-RS functionality and are encouraged to support:

* series-level retrieval
* instance-level retrieval
* thumbnail generation

### WADO endpoint responds with instance data

As above, the server validates the access token via the Token Introspection API, ensuring that the token:
* is active
* matches the patient ID for the requested study
* includes scopes for `patient/ImagingStudy.read` or `patient/*.read` access

As above, if the server needs to fetch data from sources that will take some time (e.g., issuing a DICOM C-MOVE query under the hood), it MAY respond with a `503` status and a `Retry-After` header indicating how many seconds the app should wait before re-trying its query.

If response data are available directly, the server returns

 ```
200
 Content-type: multipart/related; type=application/dicom
 [[DICOM DATA BINARY PAYLOAD]]
```
