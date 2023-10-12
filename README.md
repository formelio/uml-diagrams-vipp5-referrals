# UML diagrams for Zuyderland-MUMC-VieCuri referrals

UML diagrams for the VIPP5 referral project with Zuyderland, MUMC and VieCuri. These sequence diagrams are from the perspective of Zuyderland.

## SMART-on-FHIR based Zuyderland-HINQ SSO

```mermaid
sequenceDiagram
    autonumber

    title Zuyderland HINQ ZNO SSO

    box Zuyderland
        actor zp as Care provider
        participant ze as Zuyderland EHR
        participant zff as Firely FHIR server
        participant zfa as Firely auth server
    end

    box HINQ
        participant ha as ZNO auth server
        participant hz as ZNO
        participant hn as HINQ Nuts node
    end

    zp->>ze: Refer patient x<br/>for use case y
    note over ze: Prerequisite: patient is<br/>loaded into Firely Server
    note over zfa:Precondition<br/>EHR is<br/>registered
    ze->>zfa: Request launch context (authz header + secret + patient/practitioner/encounter id)
    zfa->>ze: Launch context identifier
    note over ze: iframe
    ze->>ha: Redirect to ZNO with SMART EHR Launch request (iss, launch)
    ha->>zff: Obtain CapabilityStatement OR<br/>.well-known/smart-configuration.json
    zff->>ha: authorization_endpoint, token_endpoint
    note over zfa:Precondition<br/>ZNO is registered as client
    ha->>zfa: Redirect to authorization_endpoint: authorization request (client_id, redirect_uri, <br/>launch, scope, state, aud (==iss), code_challenge(_method))
    note over zfa: scope:<br/>patient/[BgZ].s<br/>patient/Task.c
    note over zfa: We need to provide<br/>launch/patient<br/>launch/Encounter<br/>launch/Practitioner
    zfa->>ha: Redirect with authorization code
    ha->>zfa: token_endpoint: request access token
    zfa->>ha: access_token & launch context parameters (Patient ID, Encounter ID and Practitioner ID)
    note over ha: Do we need to make FHIR queries for additional info or<br/>is all necessary account info included in launch context parameters?
    opt We need additional resources for account info
    ha->>zff: request resources (access_token as bearer token)
    zff->>zfa: validate token
    zfa->>zff: valid
    zff->>zff: Obtain BSN from access token
    zff->>zff: Perform search<br/>(SoF enabled)
    zff->>zff: Post process based<br/>on workflow task
    zff->>ha: resources
    end
    ha->>hn: Look up Zuyderland DID using known identifiers
    hn->>ha: Zuyderland DID
    ha->>ha: Add Zuyderland DID to session context
    ha->>hz: Redirect user to ZNO
    hz->>zp: Display referral page
```

## Incoming referral notification

```mermaid
sequenceDiagram
    autonumber

    title Receiving a referral notification

    box MUMC
        actor mp as Referring care provider
        participant me as EPD
        participant mf as Firely Server
        participant mr as Referral
        participant mn as Nuts node
    end

    box HINQ
        participant hn as IOP Nuts node
        participant hf as IOP FHIR interface
        participant ha as IOP audit serivce
    end

    box Zuyderland
        participant zf as Firely Server
        participant za as Firely Auth
        participant zt as ETL
        participant ze as EPD
        actor zp as Care provider
    end

    note over mr: Nuts auth flow
    mp->>me: Start referral flow
    me->>mr: Redirect to referral application
    mr->>mn: Search Nuts network for<br/>organizations supporting<br/>the service
    mn->>mr: Available organizations
    mr->>mp: Present available organizations
    mp->>mr: Select Zuyderland
    mr->>mp: Display data available for referral
    mp->>mr: Select data to be sent
    mr->>mn: Issue workflow Task AuthorizationCredential that grants<br/>access to workflow Task
    mr->>mn: Issue BGZ AuthorizationCredential that grants<br/>access to referral data.
    mn-->hn: Sync new AuthorizationCredentials
    mr->>mn: Look up Zuyderland notification and token endpoints
    mn->>mr: Zuyderland notification and token endpoints
    mr->>mn: Get access token without<br/>any authorization credentials
    mn->>mr: Access token
    mr->>hf: Send Task to notification endpoint (POST Task)
    hf->>hf: Validate access token
    hf->>hf: Validate notification Task
    note right of hf: To check<br/>- owner.identifier matches token authorizer and supports bgz-receiver service (so is Zuyderland)<br/>- requester.onBehalfOf.identifier matches token requester and supports bgz-sender service<br/>- requester.agent.identifier matches the bgz-sender service endpoint in the requester organization's DID document<br/>- input:workflow-task is either not set or true
    hf->>hf: Get necessary info from notifcation Task
    note right of hf: To extract<br/>- input.authorization-base, the ID of the AuthorizationCredential to use to fetch the workflow Task<br/>- requester.onBehalfOf.identifier, the DID of the sending organization<br/>- input:read-available-resources, relative path to the workflow Task (e.g. Task/123)
    hf->>hn: Get workflow Task AuthorizationCredential<br/>by ID from Task.input.authorization-base
    hn->>hf: Workflow Task AuthorizationCredential
    hf->>hf: Validate AuthorizationCredential exists and is valid
    hf->>hn: Get BGZ sender service from DID document of organization<br/> with DID Task.requester.onBehalfOf.identifier
    hn->>hf: BGZ sender token and FHIR endpoints
    hf->>hn: Request access token with AuthorizationCredential
    hn->>hf: Access token
    hf->>hf: Get workflow Task id X<br/>from notification task
    hf->>mf: Get workflow Task from BGZ sender service FHIR endpoint (GET Task/X)
    note over mf: Nuts plugin
    mf->>hf: Task/X
    hf->>hf: Get Patient ID Y from Task resource
    hf->>mf: GET Patient/Y
    note over mf: Nuts plugin
    mf->>hf: Patient/Y
    note over hf:TO DO<br/>SoF backend<br/>services flow
    note over hf:To decide:<br/>level of access needed<br/>depending on whether or <br/>not always launched from <br/>EHR context
    hf->>zf : Obtain .well-known/smart-configuration.json<br/>OR CapabilityStatement
    zf->>hf: authorization_endpoint, token_endpoint
    hf->>hf: Generate and sign one-time-use<br/>authentication JWT
    note over za: Precondition<br/>Zuyderland IOP node is registered<br/>with agreed upon signing keypair
    hf->>za: Token request<br/>scope: system/Patient.c system/Task.c<br/>grant_type: client_credentials<br/>client_assertion: <jwt>
    za->>hf: Access token
    hf->>zf: POST Task & Patient
    note over zf:Consideration<br/>custom operation
    zf->>zf: Process<br/>notification task
    zf->>hf: 202 Accepted or 200 OK
    hf->>mr: 202 Accepted
    mr->>mp: Show notification sent page
    zf->>zt: Create<br/>notification message
    note right of zf: subscription,<br/>polling consumer?
    zt->>ze: Notify
    ze->>ze: Update worklist
    ze->>zp: Notify
```

## Accepting and importing an incoming referral

```mermaid
sequenceDiagram
    autonumber

    title Receiving and importing a referral

    box Zuyderland
        actor zp as Care provider
        participant ze as EPD
        participant za as Firely Auth
        participant zf as Firely Server
    end

    box Zuyderland IOP node
        participant zif as FHIR interface
    end

    box HINQ
        participant hz as ZNO
        participant hn as Nuts node
    end

    box MUMC
        participant mf as Firely Server
    end


    ze->>zp: Notification about incoming referral
    zp->>ze: Open referral application
    ze->>hz: Redirect to ZNO
    note over ze,hz: Insert SMART EHR Launch SSO flow here
    hz->>hn: Search for workflow AuthorizationCredentials<br/>(purposeOfUse = bgz-sender)
    note over hz,hn: Problem<br/>The AuthorizationCredential is a private credential issued to the organization<br/>at the Zuyderland node, not the organization at the HINQ node. The HINQ node therefore<br/>can't find it.
    hn->>hz: Available workflow AuthorizationCredentials
    note over hz: Question<br/>How do we determine which referrals<br/>are pending and not completed yet?<br/><br/>Spec says AuthorizationCredentials should be revoked<br/>when the Task is updated. Does Parsek do that?<br/>Or should we get the status from the workflow Task resource?
    hz->>zp: Present pending referrals and option to start a new referral
    zp->>hz: Select pending referral
    hz->>hz: Get necessary info from workflow AuthorizationCredential
    note over hz: To extract <br/>- issuer, the DID of the sending organization <br/>- resources[0].path, the ID of the workflow Task resource (Task/123)
    hz->>hn: Look up bgz-sender service endpoints of<br/>sending organization using DID
    hn->>hz: Token and FHIR endpoints
    hz->>hn: Request access token for workflow Task
    hn->>hz: Access token
    hz->>mf : Read workflow Task (GET Task/123)
    mf->>hz: Workflow Task
    note over hz: To extract<br/>- input.authorization-base, DID of the AuthorizationCredential needed to<br/>  retrieve the data.<br/>  - input:read-available-resources, all the to be executed FHIR reads to get the data<br/>  - input:query-available-resources, all the to be executed FHIR searches to get the data
    hz->>hz: Check workflow Task still has status requested
    hz->>hn: Get BGZ AuthorizationCredential using input.authorization-base from the workflow Task
    hn->>hz: BGZ AuthorizationCredential
    hz->>hn: Request access token using BGZ AuthorizationCredential
    hn->>hz: Access token
    hz->>mf: Execute all the FHIR requests
    mf->>hz: Responses
    hz->>zp: Present resources
    zp->>hz: Select resources to import
    hz->>hn: Find DID of Zuyderland organization at Zuyderland node
    hn->>hz: Zuyderland DID
    hz->>hn: Request same-organization access token
    note over hz,hn: Question<br/>How do we determine that this access token is meant for a referral import?<br/>Do we use some sort of custom purposeOfUse in the access token request?
    hn->>hz: Access token
    hz->>zif: Send all selected resources<br/>(POST transaction Bundle)
    note over za,zif: Insert SMART Backend Service flow here
    zif->>zif: Validate same-organization<br/>access token
    zif->>zf: Import resources
    zf->>zif: Ok
    zif->>hz: Ok
    hz->>mf: Update workflow Task with status completed (PUT Task/123)
    mf->>hz: Ok
    hz->>zp: Present success page
```
