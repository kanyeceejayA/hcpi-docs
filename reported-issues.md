# System Issues
Dashboard issues For all:
1. A drop-down option to select the month and year to show the figures (both index and inflation), Filter by month (Latest)
2. Show National CPI Value at the top, historical trend comes higher, 
3. See how to show all Divisions on the Screen 
4. Fix Issues of No national figures shown on dashboards (uganda-specific bug)

5. See How to setup Dashboard for public access

 
KNBS;
1. Systems shows 100% during data collection but later reduces to 67%
2. Standard and non-standard of weights*
3. Provision of in-built calculator - Explanation of how inbuilt calculations are being done 
4. [data collection] System slow and more so when more users are added and data increases. 
5. Dashboard Issue: Figures from every sector are showing 0
 
UBOS:
1. Dashboard Issue: Figures from every sector are showing 0
 
NBS:
1. GPS not working during data collection. 
2. Clarify if server is offline whether the tablet offline works [Test to see]

OCGS:
Unable to update EA Indices - Resolved

EAC:
1. Review Syncing of Data to Secretariat Dashboard

# Computations:
1. Review of Computations Process 

# Suggestions:
1. Data Collection Analytics: Show counts by state of collection qtnires by outlet, centre, national, etc.
2. Calculation of Provisional Index / with details of items. methodology: copy over prev. month values.
3. Review outliers to ensure we have 3 levels, extreme outlier=24, outlier=12, mild Outlier =6 months
4. Review Inliers to ensure we have it checking 12 previous consecutive months and also if change in the price is the same for the same period.
5. Report of outliers and inliers
6. Monitor and flag changes in Centrally Administrative Prices
7. Document XML/RPC endpoints for system.
8. Transfer Mobile App from Kola to EAC-Controlled Google Play Account

Annual % change
core and non-core to indicator cards


- Item codes should be standardized
- Deleting update history and jobs does not mark as failed. a lock remains preventing restarting an update on the same items
- Collected Data needs to be able to go through all validation steps and make it to observations so that it is used in calculations.
- Add Period and division filters to index update functions in HCPI_Index. Iwe should be able to run the update on a subset of the data.
- Large log tables leading to slow queries and large database size. review what is being logged and see howto balance what is needed and what isn't.
- After 3 months of temporarily missing, the system should indicate how long it has been
- A report on items that have been temporarily missing for 3 months or more.
- Code on replacement of items not sufficient.

Now, some updates:
swap blue and red, and add blue to key for matrix on dashboard. and have the key explain them
operational dashboard should show which consumption segments are so far behind in terms of low coverage of outlet-items or something like that which is relevant. 

We need to review the computations that happen, and compare current process to how it should be. 
1. there should be no zero prices. if price has not been captured for an item, it should have been imputed, or should be imputed. Those imputations need to follow one of the proper imputation methods
2. there should be no 0s or missing values are w ecompute the heigher levels. any item that has a weight should have a value, ensuring that these figures cascade up to the final view
3. If a figure is missing, it needs to be highlighted on operational dashboard for each of the levels.
4. For the provisional CPI, carry-forward method can be used, with operational dashboard highlighting that
There should be no zeros at all i nthe computation at all the levels
all active items should be included in ccalculations, impute where none
all EA indexes should be included, no gaps. if item has weight, it needs to have a figure. highlight any on operational dashboard, calc
impute prices, not later fields 
carry-forward can be used at start of calculations, but is temporary. Proper method should be proper imputation using one of imputation methods. 
Add a link to report for validations on the provisional
a repor tshowing when collections are happening vs days
Link the matrix of events from 

# Data Collection App
- When Questionnaires synced to app, they shouldcome with the questionnaire content, not coming afterwards. so that once sync happens they can work offline.
- reorder questionnaire so that collection code is selected after price is entered, and we clear the price if temporarily missing or permanently missing is selected as the code. 

- Pulish a version of the APK on the servers so that it can be used to ensure data collectors are always on the right version
- Publish to play store to replace the existing entry 
- 

ssh ocgs_ubuntu@hcpi.ocgs.go.tz -p 7575

