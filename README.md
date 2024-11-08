# IJP Engines

## Overview
This is the documentation of the IJP engines, there working and functionalities

## Contents
- [Search Engine](#search-engine)
- [Apply Engine](#apply-engine)
- [Post Engine](#post-engine)

## Search Engine
Search Engine filters and retrieves job postings based on various criteria (e.g., user eligibility, designation, and specific configurations set by the company).
Break down of search engine.

### Modules imported

1. db: Interface with the database.

2. Logger: Provides logging capabilities to log errors or important information.

3. getIjpClientConfiguration: A helper function to retrieve the IJP configuration for a specific client, which is used to determine which filters should be applied for that client.

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

1. *Retrieve Configurations* : Gets the configuration for the specific client.
2. *Build Base Query* : Starts with a general SQL query for retrieving eligible job postings.
   ```sql
   const baseQuery = `select id, rrp.createdon from g.rms_request_profiles_${clientUUID} rrp where isijp = true and  profile_status in ('IDFT','RMGA', 'PLAV', 'SLCT') and (jsonb_extract_path_text(profile_criteria, 'deploymentInfo','confidentialRole') is null or 'false')`;
   ```
   
3. *Apply Filters* :
   * Retrieve user data
   * Check for rotation eligibility
   * Apply nested configuration filters if available
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

4. *Return the Final Query* : 
Adds a filter to the query based on profileStatusList, ensuring only postings with the specified statuses are included.
```javascript
baseQueryFilters += ` and profile_status in ('${profileStatusList.join("', '")}')`;
```

## Apply Engine
Apply Engine is responsible for various eligibility criteria are checked before a user can apply for a position. Let's break down the key parts of this code.

### Modules Imported
* *sequelize* for database operations.
* *db* and *Logger* modules for database connections and logging.
* *RMSConfiguration*, *getMatchScore*, and other utility functions for applying specific configurations and criteria checks.
* *DateUtils* for date manipulation and *getIjpClientConfiguration* for fetching specific client configurations.

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

Configuration Fetching:

Converts clientId to clientUUID and retrieves the IJP configuration (rmsIjpConfig) for that client.
If the configuration is not found, it logs an error and returns a failure response.
Filter and Profile Data Setup:

Checks if there are specific applyconfigurations for this client. If not, it allows the application by returning a success response.
If filters exist, they are processed as criteria to validate against the user and job profile.
Fetch Profile and User Data:

Queries for profile and user data based on profileId and userId.
Retrieves the user’s eligibility for rotation using userRotationEligibleCheck, returning a failure response if ineligible.
Filter-based Checks:

Iterates through each element in filters and applies various checks based on job and user data:
applyThreshold and applyRateLimiter: Verifies application limits using checkForApplyThreshold and checkForApplyRateLimiter.
allocationPool: Checks if the user is in a required allocation pool, based on project allocation and user data.
bu, subBU, employeePractice, employeeSubPractice: Verifies that user-specific details match job requirements using getElementResult.
designationLevel: Uses designationLevelCheck to ensure the user’s designation level meets job requirements.
onsiteVsOffshore and locationCountry, locationCity: Verifies user location preferences align with the job profile.
profileStatus and matchScore: Checks that the user's profile status and match score meet the minimum requirements for application.
onNotice: Restricts applications if the user is on notice, per configuration.
Return Result:

If all filters pass, checkApplyEngine returns a success response.
If any check fails, it returns an error response with a relevant message logged.


## Post Engine

### Modules imported

1. db: Interface with the database.

2. Logger: Provides logging capabilities to log errors or important information.

3. getIjpClientConfiguration: A helper function to retrieve the IJP configuration for a specific client, which is used to determine which filters should be applied for that client.

### Helper Functions

1. *userRotationEligibleCheck*
   
This function checks if a user is eligible for job rotation.

* Parameters: userData (user details), nestedConfig (configuration with job rotation eligibility settings).
* Logic: Checks if the rotationEligibleCheck is enabled in nestedConfig. If it is, it verifies the user's eligibility status; otherwise, eligibility defaults to true.


