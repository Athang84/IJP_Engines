# IJP Engines

## Overview
This is the documentation of the IJP engines, there working and functionalities

## Contents
- [Search Engine](#search-engine)
- [Apply Engine](#apply-engine)
- [Post Engine](#post-engine)
- [How to Trigger](#how-to-trigger)

## Search Engine
Search Engine filters and retrieves job postings based on various criteria (e.g., user eligibility, designation, and specific configurations set by the company).
Break down of search engine.

### Helper Functions

1. *userRotationEligibleCheck*
   
This function checks if a user is eligible for job rotation.

* Parameters: userData (user details), nestedConfig (configuration with job rotation eligibility settings).
* Logic: Checks if the rotationEligibleCheck is enabled in nestedConfig. If it is, it verifies the user's eligibility status; otherwise, eligibility defaults to true.

2. *designationLevelCheck*
   
This function applies a tolerance filter for designation levels.

* Parameters:
  * userObject (contains user data).
  * nestedConfig (includes designation filters and tolerances).
  * designationLevel (user's current designation level).
  * rmsSearchConfig (contains designation-related configurations).

* Logic:
  * Attempts to parse the user's designationLevel.
  * Checks if there is a tolerance range configured (e.g., if a user with designation level 5 can view jobs for levels 4–6).
  * Based on the calculated range, filters out designation IDs from the job postings and returns a condition that filters job postings based on designation levels.

### Main function : **getIJPSearchFilters**

This asynchronous function builds a SQL query to retrieve job postings based on various criteria and user attributes.
This code is divided into 4 major parts
1. *Parameter* :
   
   * clientId: The unique identifier for the client organization.
   * userId: The unique identifier for the user requesting job postings.
   * designationLevel: The user's designation level, used to filter postings by job level.
   * rmsSearchConfig: Contains various configuration options for search criteria related to the RMS system.
   * expectedMatchingScore: (optional) A threshold for a "matching score" that determines how well a user's profile matches job criteria.
     
2. *Retrieve Configurations* : Gets the configuration for the specific client.
   ```javascript
   const clientUUID = clientId.replace(/\-/g, '');
   const rmsIjpConfig = await getIjpClientConfiguration(clientId);
   ```
   * Converts clientId into a clientUUID by removing hyphens, as it may be used as part of a table name in queries.
   * Fetches the IJP configuration (rmsIjpConfig) for the client using getIjpClientConfiguration. This configuration dictates the search filters applicable to this client.

   ```javascript
   if (!rmsIjpConfig || Object.keys(rmsIjpConfig).length === 0) {
     Logger.error(`IJP client: ${clientId} not found`);
     return '';
   }
   ```
   * If no configuration is found for the client, an error is logged, and an empty string is returned, as there’s nothing to filter on.
     
3. *Build Base Query* : Starts with a general SQL query for retrieving eligible job postings.
   ```sql
   const baseQuery = `select id, rrp.createdon from g.rms_request_profiles_${clientUUID} rrp where isijp = true and  profile_status in ('IDFT','RMGA', 'PLAV', 'SLCT') and (jsonb_extract_path_text(profile_criteria, 'deploymentInfo','confidentialRole') is null or 'false')`;
   ```
   * Constructs a basic SQL query that retrieves job IDs and creation dates from the rms_request_profiles table for that client. Only postings with specific statuses ('IDFT', 'RMGA', 'PLAV', 'SLCT') are retrieved.
   * Ensures that the job is not marked as a “confidential role” in the criteria JSON.
   
4. *Apply Filters* :
   * Retrieve user data
      ```javascript
      const userData = await db.readSequelize.query(`select data from g.users_${clientUUID} c where id = '${userId}'`);
      const userObject = userData[0][0].data;
      ```
      * Fetches user data based on userId from the users table and extracts relevant user information (userObject) from the result.
   * Check for rotation eligibility
      ```javascript
      if (!userRotationEligibleCheck(userObject, nestedConfig)) return '';
      ```
      * Uses userRotationEligibleCheck to determine if the user is eligible for job rotation based on their eligibility status and client configuration. If not eligible, returns an empty string.
        
   * Apply nested configuration filters if available
      ```javascript
      const filterVal = nestedConfig?.filters;
      if (!filterVal) {
        return baseQuery;
      }
      if (filterVal?.onNotice && userObject?.onNotice?.toUpperCase() === 'YES') {
        return '';
      }
      ```
      * Checks for various filters in filterVal (obtained from nestedConfig.filters). If the user is on notice and the configuration excludes such users, returns an empty string.
        
   * Iterate through the filters
     * allocationPool:
      Checks if the user belongs to the required allocation pool based on allocationPoolAr array and includes only those who match the configured pool.

     * profileStatus:
      Updates profileStatusList based on the specified statuses in the configuration.

     * bu:
      Adds a condition to the query based on the user's business unit (BU) if specified.

     * subBU:
      Filters based on the user's sub-business unit.

     * employeePractice:
      Adds a condition to filter postings by the user’s practice area.

     * employeeSubPractice: 
      Adds a condition for filtering postings by the user’s sub-practice area.

     * designationLevel:
      Uses the designationLevelCheck function to adjust the query for designation level tolerances, if applicable.

     * onsiteVsOffshore:
      Filters postings based on the user’s onsite vs. offshore preference.

5. *Return the Final Query* : 
Adds a filter to the query based on profileStatusList, ensuring only postings with the specified statuses are included.
```javascript
baseQueryFilters += ` and profile_status in ('${profileStatusList.join("', '")}')`;
```

## Apply Engine
Apply Engine is responsible for various eligibility criteria are checked before a user can apply for a position. Let's break down the key parts of this code.

### Helper Functions
* errorResponseHandler and successResponseHandler:
   * errorResponseHandler: Returns an error response object with status: false and an error message.
   * successResponseHandler: Returns a successful response with status: true.
     
* designationLevelCheck:
   * Checks if a user's designation level falls within a range (or tolerance) around the job profile's designation level.
   * If the tolerance is defined in the configuration, it adjusts the min and max levels for eligible designations.
   * It then checks if the profile’s designation ID is within the acceptable range.
  
* userRotationEligibleCheck:
   * Checks if the user is eligible for rotation based on the isRotationEligible property in the user data and the nested configuration.

* checkForApplyRateLimiter and checkForApplyThreshold:
   * checkForApplyRateLimiter: Prevents users from applying more frequently than allowed by checking if their application count exceeds a set limit within a specified time frame.
   * checkForApplyThreshold: Checks if a user has exceeded the ongoing application threshold, ensuring they do not have too many active applications at the same time.

* getElementResult:
   * A utility to verify that a user’s property (e.g., business unit, practice area) matches the profile’s required property, or that both are unspecified, making them compatible.


### Main Function: *checkApplyEngine*

This is the core function, performing the eligibility checks. It accepts profileId, userId, clientId, designationLevel, and rmsSearchConfig as inputs, representing the job profile, user, client, and job-related configurations.

1. Initial Setup and Cofiguration setup
   
   * It removes dashes from clientId to create clientUUID, used to query client-specific tables.
   * rmsIjpConfig is fetched via getIjpClientConfiguration, which contains configurations related to application eligibility criteria.
     
2. Early Exit Checks
   * If rmsIjpConfig is missing or empty, it logs the error and returns an error response.
   * If applyconfigurations is empty or set to allow all requests, it immediately returns a success response.
     
3. Filter Configuration
   * The function retrieves filters from nestedConfig. If no filters are configured, it returns success.

4. Data Fetching
   * Queries are created to fetch:
      * User data: from users_<clientUUID> table.
      * Profile data: from rms_request_profiles_<clientUUID> table joined with rms_request_tbl_<clientUUID>, retrieving the candidate's profile and request data.

5. Eligibility Checks
Each filter has a specific check to ensure compliance with configured conditions. If any check fails, an error response is returned.

Key Filters and Their Checks:
* Apply Threshold:
   Uses checkForApplyThreshold to see if the user has exceeded the limit on ongoing applications.

* Apply Rate Limiter:
   Uses checkForApplyRateLimiter to check if the user has applied too frequently within a specified timeframe.
  
* Allocation Pool:
   Ensures the user belongs to an eligible allocation pool by querying timeseriesdata based on the allocationPool filter.
  
* BU (Business Unit):
   Checks if the user’s BU matches the BU in the profile data.

* Sub-BU, Employee Practice, and Employee Sub-Practice:
   Verifies the user’s sub-BU, practice, and sub-practice align with profile requirements.

* Designation Level:
   Calls designationLevelCheck to ensure the user's designation level is within a permissible range.

* Onsite vs. Offshore:
   Validates if the user’s location (onsite/offshore) matches the job’s preference.

* Location Country and City:
   Compares the user’s country and city with the profile requirements, including delivery center and preferred office location.
  
* Profile Status:
   Ensures the user’s profile status matches one of the acceptable statuses in the configuration.
  
* Match Score:
   Uses getMatchScore to check if the user’s match score meets the minimum required score.
  
* On Notice:
   Checks if the user is marked as on notice, which may disqualify them from applying.


## Post Engine

### *mapDesignationName* Function

```javascript
const mapDesignationName = async (clientid, designationCodes) => { ... };
```

This function takes in a clientid and designationCodes to return a unique list of designation IDs matching the given codes.

* Fetch Designations: Retrieves configuration data for the client from RMS.
* Filter and Collect IDs: Filters designations based on matching designationCode and stores the relevant IDs in an array.
* Return Unique IDs: Returns the array as a Set to ensure uniqueness.

### *markJobsForIJPByClient* Function

```javascript
const markJobsForIJPByClient = async (client, shouldBeAddIjIjpFilter = true) => { ... };
```
This function builds SQL queries to identify and update job profiles for a client based on their IJP settings.

* Initial Setup:
   * It sets response with empty updateQuery and getQuery strings.
   * Retrieves client-specific configurations from client.data.
     
* Profile Status Check:
   * Sets a default profile status (profileStatus) list or uses a status list from the client configuration. This determines which profiles to update.

* Configuration Check:
   * Checks if the client has IJP posting enabled (postToIjp).
   * If not enabled, it exits and returns an empty response.
     
* Build Query Based on All Requests (allRequests):
   * If allRequests is enabled, constructs SQL queries to update job profiles matching status and confidentiality criteria, then fetches profile IDs needing IJP updates.

* Conditional Query Components:
   * The code also handles cases where allRequests is disabled by filtering on finer criteria:
      * requestIJP: Adds a filter if the request requires IJP (requestIJP flag).
      * Business Units and Sub-Business Units: Filters profiles based on project-related attributes (projectBU, projectSubBU).
      * Designations: Uses mapDesignationName to map and filter profiles by designation.
      * Location: Filters based on locationOnsiteOffsite preference.
      * Project Type and Sub-Demand Type: Filters on the type of project (projectType) or demand type for the resource (subDemandTypeNewAddition).

* Finalize SQL Query:
   * Each filter component dynamically builds parts of the SQL query.
   * Once filters are applied, the complete SQL statement is logged and returned in response.

### *markJobsforIJP* Function
```javascript
const markJobsforIJP = async (loggerObject) => { ... };
```

This is the main function that executes the IJP posting process for all clients with IJP enabled.

* Fetch Clients with IJP Configurations: Gets a list of clients where isIJPEnabled is set to true.
* Iterate Through Clients: For each client:
   * Determine Query Engine: Uses either EngineV2 or markJobsForIJPByClient based on isV2PostToIjpEngineEnabled.
   * Execute Queries: Executes getQuery to retrieve profiles that need IJP updates and updateQuery to mark these profiles as IJP-enabled in the database.
   * Profile Updates and Logging:
      * Updates IJP status for relevant profiles.
      * Logs the operation’s start, success, or failure.
    
### *addAutoPostToIjpHistory* Function
```javascript
const addAutoPostToIjpHistory = (clientid) => (configurations, profilesDataList) => async (action, newStatus, category) => { ... };
```
This function adds IJP actions to the profile’s activity history.

* Prepare History Entry: Adds an action (e.g., POST_TO_IJP) to the profile’s IJP activity history.
* Update Database: Updates the ijp_activity field in the database with the new history.
* Activity Tracking: If activity tracking is enabled in the client’s configuration, initiates a new IJP activity tracker event.

### Helper Function *getMappedArray*
```javascript
const getMappedArray = (array) => { ... };
```

This helper function converts each item in an array to SQL-compatible single-quoted strings for direct inclusion in SQL statements.

## How to Trigger

### Triggering *markJobsForIJPByClient*

* Purpose
  
   *markJobsForIJPByClient* processes job profiles for a single client based on their configurations and generates SQL queries to update and retrieve profiles eligible for IJP.
  
* How to Trigger

   You need to pass in a client object, which includes clientid and configuration data. This function is generally called internally for each client but can be triggered directly if needed.
   ```javascript
      const client = {
     clientid: 'client-uuid-1234',
     data: {
       postingconfiguration: {
         postToIjp: true,
         profileStatus: ['IDFT', 'RMGA'],
         allRequests: true,
         requestIJP: false,
         designations: ['Design1', 'Design2'],
         // Additional nested configuration data if needed
       },
     },
   };
   
   markJobsForIJPByClient(client)
     .then((response) => {
       console.log('Generated Update Query:', response.updateQuery);
       console.log('Generated Get Query:', response.getQuery);
     })
     .catch((error) => {
       console.error('Error while processing markJobsForIJPByClient:', error);
     });
  ```
   
* Response

When you call markJobsForIJPByClient, it will return an object with two properties: updateQuery and getQuery.

Example Response
```javascript
   {
  updateQuery: `update g.rms_request_profiles_clientuuid1234 rrp1 set "isijp" = true, modifiedon=now(), profile_criteria = jsonb_set(rrp1.profile_criteria, '{deploymentInfo, otherDetails, ijpEnable}', '"true"') where profile_status in ('IDFT', 'RMGA') and (jsonb_extract_path_text(profile_criteria, 'deploymentInfo','confidentialRole') is null or 'false') and isijp = false;`,
  getQuery: `select id, requestid, ijp_activity, isijp, profile_criteria->'deploymentInfo'->'otherDetails'->>'ijpEnable' as "ijpEnable" from g.rms_request_profiles_clientuuid1234 where profile_status in ('IDFT', 'RMGA') and (jsonb_extract_path_text(profile_criteria, 'deploymentInfo','confidentialRole') is null or 'false') and isijp = false;`
}
```
* updateQuery: The SQL query to update job profiles for the client, setting isijp to true for profiles matching the client’s configuration.
* getQuery: The SQL query to retrieve job profiles for this client that meet specific conditions, so you can verify which profiles are affected by the update.

You can then execute these queries on the database to perform the updates and retrieve the relevant profiles.

### Triggering *markJobsforIJP*
* Purpose

   markJobsforIJP is the batch-processing function that applies the IJP posting configuration for all clients with IJP enabled. This function fetches each client’s configuration and then processes them, calling markJobsForIJPByClient or an alternative engine (EngineV2) as configured.
  
* How to Trigger

   To call markJobsforIJP, provide a loggerObject for tracking logging information. This function runs without needing direct input parameters for each client.
   ```javascript
      const loggerObject = {
     logStart: (clientid) => console.log(`Started processing for client: ${clientid}`),
     logSuccess: (clientid) => console.log(`Successfully processed for client: ${clientid}`),
     logFailure: (error, clientid) => console.error(`Failed for client ${clientid} with error:`, error),
   };
   ```

markJobsforIJP(loggerObject);
* Response

markJobsforIJP doesn’t return a direct response object because it processes all IJP-enabled clients sequentially, updating each client's profiles based on configuration.

Example Logs
Instead of a response, you’ll see logs (if logging is enabled) that detail the processing status for each client. Here’s a typical log output example:

```text
markJobsforIJP is started
Started processing for client: client-uuid-1234
IJP enabled for client: client-uuid-1234
Successfully processed for client: client-uuid-1234
Started processing for client: client-uuid-5678
update postToIJP from IJPJobEngine
Successfully processed for client: client-uuid-5678
markJobsforIJP is finished
```
Each log entry indicates:

* Client Start: Logs the start of processing for each client.
* Client Success: Logs a success message if all updates are applied correctly for a client.
* Client Failure (if applicable): Logs an error if processing for a specific client fails.


### Triggering *addAutoPostToIjpHistory*     
* Purpose

   The addAutoPostToIjpHistory function adds an activity history entry for each job profile, marking actions like posting to IJP or enabling IJP, and logs this activity in the database.

* How to Trigger

   Call addAutoPostToIjpHistory with clientid, configuration details, and a list of profiles for which you want to update the activity history.
   ```javascript
   const clientid = 'client-uuid-1234';
   const configurations = {
     isActivityRequired: true, // Set to true if activity tracking is needed
   };
   
   const profilesDataList = [
     {
       id: 'profile-id-1',
       ijp_activity: { history: [] },
       requestid: 'request-id-1',
     },
     {
       id: 'profile-id-2',
       ijp_activity: { history: [] },
       requestid: 'request-id-2',
     },
   ];
   
   const action = IJP_ACTION_ENUM.POST_TO_IJP;
   const newStatus = IJP_ACTION_ENUM.POST_TO_IJP_AUTOMATIC;
   const category = ACTIVITY_TRACKER_CATEGORY.POST_TO_IJP_AUTOMATIC;
   
   addAutoPostToIjpHistory(clientid)(configurations, profilesDataList)(action, newStatus, category)
     .then(() => {
       console.log('Auto-post history updated successfully.');
     })
     .catch((error) => {
       console.error('Error while updating auto-post history:', error);
     });
   ```
* Response

addAutoPostToIjpHistory also doesn’t return a direct response, but it updates each profile’s activity history with the specified action. You’ll typically observe updates in the database and see logs or console messages if the activity is tracked.

Example Activity History in Database
Each profile's ijp_activity field is updated with a history entry. Here’s what a database entry might look like after the update:

```json
{
  "ijp_activity": {
    "history": [
      {
        "action": "POST_TO_IJP",
        "date": "2024-11-07T10:23:45.678Z"
      },
      {
        "action": "ENABLE_IJP",
        "date": "2024-11-07T11:15:00.123Z"
      }
    ]
  }
}
```

In this example:

* ijp_activity.history: An array that lists actions taken on this profile, each with a timestamp.
If isActivityRequired is enabled in configurations, additional logs or messages might confirm the activity.


