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

