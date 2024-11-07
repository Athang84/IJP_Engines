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

1. userRotationEligibleCheck
This function checks if a user is eligible for job rotation.

* Parameters: userData (user details), nestedConfig (configuration with job rotation eligibility settings).
* Logic: Checks if the rotationEligibleCheck is enabled in nestedConfig. If it is, it verifies the user's eligibility status; otherwise, eligibility defaults to true.

2. designationLevelCheck
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

