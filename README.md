# SCD2_In_Hive

1.	Build source specific Temporary tables(Tmp) from Tier 1 applying Trim on all the columns for the delta records   

INSERT OVERWRITE TABLE <TABLENAME_TEMP> 
SELECT <TRIM(COLUMNNAME1),TRIM(COLUMNNAME2)……… 
FROM TIER 1 <TABLENAME> A 
LEFTJOIN (SEL MAX(PARTITION) LST_PARTATION FROM TIER 1 <TABLENAME> GROUP BY ORDER DATE )B 
ON A.ODERDATE = B. LST_PARTATION 

2.	Build Prestage tables(SRC) using logical transformation on above Temporary tables  
INSERT OVERWRITE TABLE <TABLENAME_PRESTAGE> 
SELECT <A.COLUMNNAME1, B.COLUMNNAME2 ……… 
FROM TIER1.<TABLENAME> A 
LEFTJOIN TIER1.<TABLENAME> B 
ON A.KEY_COLUMN = B.KEY_COLUMN

3.	Build exception tables(_EXP) adding Reject_Type and Reject_Status on Prestage tables with partition on business date with checking for source primary keys duplicates and NULL values, Here Exception records can be accessed in case of mismatch with Source and Target 
INSERT OVERWRITE TABLE <TABLENAME_EXP>  PARTITION (BUSINESS_DATE='${HIVECONF:SHELLPARAMETER}')
SELECT <COLUMNNAME1,COLUMNNAME2,CASE WHEN TGT.COL IS NOT NULL THEN 'DUMMY_CODE' ELSE 'DUMMY_CODE' END AS REJECT_TYPE
,'OPEN' AS REJECT_STATUS
FROM TABLENAME_PRESTAGE SRC
LEFT OUTER JOIN
(SELECT ‘1’ AS COL, KEY_COLUMN1,KEY_COLUMN2 FROM TABLENAME_PRESTAGE GROUP BY KEY_COLUMN1,KEY_COLUMN2 HAVING COUNT(*) > 1 ) TGT
ON SRC.KEY_COLUMN1 = TGT. KEY_COLUMN1
   SRC.KEY_COLUMN2 = TGT. KEY_COLUMN2
WHERE
TGT.COL IS NOT NULL
OR SRC.KEY_COLUMN1 = ''
OR SRC.KEY_COLUMN_DATE IS NULL
OR SRC.KEY_COLUMN2 = ''
;

4.	Build Stage_01 (_NEW) adding target Primary key and MD5_Checksum fields excluding all those records which are present in the _EXP tables from Prestage tables(SRC) to ensure clean data in target
INSERT OVERWRITE TABLE <tablename_NEW>  
SELECT <COLUMN1,COLUMN2,CONCAT(<KEY_COLUMN1>,<KEY_COLUMN2>) AS PARTY_KEY,reflect('org.apache.commons.codec.digest.DigestUtils', 'sha256Hex',CONCAT(<NON_KEY_COLUMN1>,< NON_KEY_COLUMN2>)) AS MD5_CHECKSUM
FROM tablename_Prestage SRC
LEFT OUTER JOIN
tablename_Exp EXP
ON  <SRC.KEY_COLUMN1> = <EXP.KEY_COLUMN1>
AND <SRC.KEY_COLUMN2> = <EXP.KEY_COLUMN2>
AND EXP.BUSINESS_DATE = '${hiveconf:ShellParameter}'
WHERE 
EXP.BUSINESS_DATE IS NULL;

5.	Build Stage2Work with dynamic partition on Record_Ind comparing stage_01(NEW) with target HIST for identify the “I” and “U” records joining condition on target primary keys and Non keys comparison.
INSERT OVERWRITE TABLE <TABLENAME_SCD2_WORK> PARTITION (RECORD_ACTION_IND)
SELECT <CURR.COLUMN1>,CURR.COLUMN2>….. 0 AS REC_DEL_FLAG,
CASE WHEN PREV.MD5_CHECKSUM = CURR.MD5_CHECKSUM THEN PREV.START_DT ELSE CURR.BUSINESS_DATE END START_DT,'9999-12-31' AS END_DT,
CASE WHEN <PREV.PARTY_KEY> IS NULL THEN 'I' WHEN PREV.MD5_CHECKSUM = CURR.MD5_CHECKSUM THEN 'NC' ELSE 'U'END RECORD_ACTION_IND
FROM 
TABLENAME_NEW CURR
LEFT OUTER JOIN 
TABLENAME_HIST PREV
ON <CURR.PARTY_KEY> = <PREV.PARTY_KEY>
AND PREV.END_DT = '9999-12-31'

6.	Build Stage2Work with dynamic partition on Record_ind  comparing stage_01(NEW) with target HIST for identify the “D” records joining condition on target primary keys and Non keys comparisons.
INSERT OVERWRITE TABLE <TABLENAME_SCD2_WORK> PARTITION (RECORD_ACTION_IND)
SELECT <PREV.COLUMN1>,PREV.COLUMN2>….. CASE WHEN <CURR.PARTY_KEY> IS NULL THEN 1 ELSE 0 END REC_DEL_FLAG,
PREV.START_DT, '${hiveconf:ShellParameter}' AS END_DT,
CASE WHEN <CURR.PARTY_KEY> IS NULL THEN 'D' ELSE 'UO' END RECORD_ACTION_IND
FROM 
TABLENAME_HIST PREV
LEFT OUTER JOIN 
TABLENAME_SCD2_WORK CURR
ON <CURR.PARTY_KEY> = <PREV.PARTY_KEY>
AND PREV.END_DT = '9999-12-31'
AND (<CURR.PARTY_KEY> IS NULL OR CURR.RECORD_ACTION_IND='U');

7.	Final merge Work2Target with static partition on End_dt using stage2Work.
  INSERT OVERWRITE TABLE TABLENAME_HIST PARTITION (END_DT)
   SELECT COLUMN1,COLUMN2,…… 
FROM TABLENAME_SCD2_WORK;
