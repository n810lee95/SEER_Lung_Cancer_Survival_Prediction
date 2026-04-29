# SEER-Surveillance-Epidemiology-and-End-Results-Program-Analysis
Analysis of Lung Cancer Tumors Using SEER Database Extract


Citation: Surveillance, Epidemiology, and End Results (SEER) Program (www.seer.cancer.gov) SEER*Stat Database: Incidence - SEER Research Data, 17 Registries, Nov 2024 Sub (2000-2022) - Linked To County Attributes - Time Dependent (1990-2023) Income/Rurality, 1969-2023 Counties, National Cancer Institute, DCCPS, Surveillance Research Program, released April 2025, based on the November 2024 submission.

Filename:  
SEER*Stat Version:  9.0.43.0
Date:  April 14, 2026

Session Type:                                         Case Listing

SUGGESTED CITATION
Software:                                             Surveillance Research Program, National Cancer Institute SEER*Stat software (www.seer.cancer.gov/seerstat) version 9.0.43.0.
Data:                                                 Surveillance, Epidemiology, and End Results (SEER) Program (www.seer.cancer.gov) SEER*Stat Database: Incidence - SEER Research Data, 17 Registries, Nov 2024 Sub (2000-2022) - Linked To County Attributes - Time Dependent (1990-2023) Income/Rurality, 1969-2023 Counties, National Cancer Institute, DCCPS, Surveillance Research Program, released April 2025, based on the November 2024 submission.

DATA
Database:                                             Incidence - SEER Research Data, 17 Registries, Nov 2024 Sub (2000-2022) - Linked To County Attributes - Time Dependent (1990-2023) Income/Rurality, 1969-2023 Counties
Database ID:                                          2536
Link Database ID:                                     2593
Use Summary Files:                                    No

SELECTION
Select Only:                                          Malignant Behavior, Known Age
Case:                                                  {Race, Sex, Year Dx.Year of diagnosis} = '2014','2015','2016'

TABLE
Column:                                               Patient ID
                                                      Age recode with <1 year olds and 90+
                                                      Sex
                                                      Marital status at diagnosis
                                                      Year of diagnosis
                                                      Race and origin (recommended by SEER) [Race and origin recode (NHW, NHB, NHAIAN, NHAPI, Hispanic); PRCDA 2020]
                                                      Site recode ICD-O-3 2023 Revision Expanded
                                                      SEER historic stage A (1973-2015)
                                                      Derived Summary Grade 2018 (2018+)
                                                      RX Summ--Surg Prim Site (1998+)
                                                      Ulceration Recode (2010+)
                                                      Time from diagnosis to treatment in days recode
                                                      Tumor Size Over Time Recode (1988+)
                                                      Survival months
                                                      Survival months flag
                                                      COD to site recode ICD-O-3 2023 Revision Expanded (1999+)

USER DEFINITIONS
                                                      Race and origin (recommended by SEER) [Race and origin recode (NHW, NHB, NHAIAN, NHAPI, Hispanic), PRCDA 2020]
                                                        Description: These are the groupings typcailly used by SEER for reporting by race and ethnicity for analyses starting in 1990 or later.  For more details see:  https://seer.cancer.gov/seerstat/variables/seer/race_ethnicity
                                                        All races/ethnicities:  {Race, Sex, Year Dx.Race and origin recode (NHW, NHB, NHAIAN, NHAPI, Hispanic)} = 'Non-Hispanic White','Non-Hispanic Black','Non-Hispanic American Indian/Alaska Native','Non-Hispanic Asian or Pacific Islander','Hispanic (All Races)','Non-Hispanic Unknown Race'
                                                        Non-Hispanic White:  {Race, Sex, Year Dx.Race and origin recode (NHW, NHB, NHAIAN, NHAPI, Hispanic)} = 'Non-Hispanic White'
                                                        Non-Hispanic Black:  {Race, Sex, Year Dx.Race and origin recode (NHW, NHB, NHAIAN, NHAPI, Hispanic)} = 'Non-Hispanic Black'
                                                        Non-Hispanic American Indian/Alaska Native (PRCDA counties only):  {Race, Sex, Year Dx.Race and origin recode (NHW, NHB, NHAIAN, NHAPI, Hispanic)} = 'Non-Hispanic American Indian/Alaska Native'
                                                            AND {Race, Sex, Year Dx.PRCDA 2020} = 'PRCDA'
                                                        Non-Hispanic Asian or Pacific Islander:  {Race, Sex, Year Dx.Race and origin recode (NHW, NHB, NHAIAN, NHAPI, Hispanic)} = 'Non-Hispanic Asian or Pacific Islander'
                                                        Hispanic (All Races):  {Race, Sex, Year Dx.Race and origin recode (NHW, NHB, NHAIAN, NHAPI, Hispanic)} = 'Hispanic (All Races)'

PROPERTIES
Database                                              Incidence - SEER Research Data, 17 Registries, Nov 2024 Sub (2000-2022)
Statistic                                             Case Listing
Row Count                                             1,266,926
Creation Time                                         53.00 seconds
Submitted                                             Tuesday, April 14, 2026 03:22 PM
Information                                           
    Number of Input Records Read                      10,334,116
                                                      
                                                      Number         Number         
                                                      Selected       Excluded       Statement      
                                                      ----------     ----------     ------------------------------------------
                                                                                    User Selection (Case)
                                                      1,447,465      8,886,651       {Year of diagnosis} = '2014','2015','2016'
                                                                                                   
                                                                                    Standard Exclusions
                                                      1,266,941      180,524        Select only malignant cancers
                                                      1,266,926      15             Select only known age
                                                      
    Number of Case Records Used For Analysis          1,266,926
                                                      
                                                      
                                                      Column:        Patient ID     
                                                                     Age recode with <1 year olds and 90+
                                                                     Sex            
                                                                     Marital status at diagnosis
                                                                     Year of diagnosis
                                                                     Race and origin (recommended by SEER)
                                                                     Site recode ICD-O-3 2023 Revision Expanded
                                                                     SEER historic stage A (1973-2015)
                                                                     Derived Summary Grade 2018 (2018+)
                                                                     RX Summ--Surg Prim Site (1998+)
                                                                     Ulceration Recode (2010+)
                                                                     Time from diagnosis to treatment in days recode
                                                                     Tumor Size Over Time Recode (1988+)
                                                                     Survival months
                                                                     Survival months flag
                                                                     COD to site recode ICD-O-3 2023 Revision Expanded (1999+)
                                                      
                                                      Sort:          Not Sorted     
                                                      
                                                      
                                                      
