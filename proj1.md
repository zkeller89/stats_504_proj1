
### Setup

```r
setwd("C:/Users/zkeller/Google Drive/stats_504/proj1")
require(readstata13)
require(readxl)
require(dplyr)

# Read in the data for msqc and our data dictionary
msqc <- read.dta13("procedure18.dta")
var.names <- read_excel("DataDictionary.xls", sheet = 1)
```

```
## DEFINEDNAME: 00 00 00 07 0b 00 00 00 00 00 00 00 00 00 00 46 6f 72 6d 61 74 73 3b 01 00 00 00 78 05 00 00 02 00 
## DEFINEDNAME: 00 00 00 09 0b 00 00 00 00 00 00 00 00 00 00 56 61 72 69 61 62 6c 65 73 3b 00 00 00 00 23 03 00 00 05 00 
## DEFINEDNAME: 00 00 00 07 0b 00 00 00 00 00 00 00 00 00 00 46 6f 72 6d 61 74 73 3b 01 00 00 00 78 05 00 00 02 00 
## DEFINEDNAME: 00 00 00 09 0b 00 00 00 00 00 00 00 00 00 00 56 61 72 69 61 62 6c 65 73 3b 00 00 00 00 23 03 00 00 05 00 
## DEFINEDNAME: 00 00 00 07 0b 00 00 00 00 00 00 00 00 00 00 46 6f 72 6d 61 74 73 3b 01 00 00 00 78 05 00 00 02 00 
## DEFINEDNAME: 00 00 00 09 0b 00 00 00 00 00 00 00 00 00 00 56 61 72 69 61 62 6c 65 73 3b 00 00 00 00 23 03 00 00 05 00 
## DEFINEDNAME: 00 00 00 07 0b 00 00 00 00 00 00 00 00 00 00 46 6f 72 6d 61 74 73 3b 01 00 00 00 78 05 00 00 02 00 
## DEFINEDNAME: 00 00 00 09 0b 00 00 00 00 00 00 00 00 00 00 56 61 72 69 61 62 6c 65 73 3b 00 00 00 00 23 03 00 00 05 00
```

### Remove Columns with all NA or same Data

```r
# badcols records column numbers of bad columns
# i.e, columns with all NA or all the same value
badcols <- c()

# for loop to get bad column numbers
for(i in 1:ncol(msqc)){
  if(sum(!is.na(msqc[,i])) == 0){
    badcols <- c(badcols, i)    
  }
  if(sum(!(msqc[1,i] == msqc[-1, i]), na.rm=T) == 0 & !is.na(msqc[1,i])){
    badcols <- c(badcols, i)
  }
}

# remove the bad column numbers and any not followed up for 30 days
msqc <- msqc[msqc$flg_followed_30d == 1, -1*badcols]
```

### Code up our complication variable

```r
### Code up our complication variable

msqc$complication = as.numeric(msqc$death == 1 |msqc$death == 2| msqc$flg_dead30 == 1 |
                            msqc$flg_cmp_sev1 == 1 | msqc$flg_cmp_sev2 == 1 | msqc$flg_cmp_sev3 == 1 | 
                            msqc$flg_cmp_sepsis_severe == 1| msqc$flg_cmp_cardiac_arrest_any == 1 | 
                            msqc$flg_cmp_ssi_any == 1)
```

### Create new variables to be used as covariates

```r
# create our new variables as covariates
msqc$flg_asa <- as.numeric(msqc$asa_class_id >= 3) 

msqc$n_comorbid <- msqc$flg_cmb_cancer + 
  msqc$flg_cmb_bleeding_disorder + 
  msqc$flg_cmb_pvd + #peripheral artery disease
  msqc$flg_cmb_coronary_artery + # coronary artery disease
  msqc$flg_cmb_diabetes + # dieabeetus
  msqc$flg_cmb_copd + #chronic obstructive pulmonary disease
  msqc$flg_cmb_hypertension + #hypertension
  msqc$flg_cmb_dvt +# deep vein thrombosis
  msqc$flg_cmb_preop_sepsis + # preop sepsis
  msqc$flg_cmb_pneumonia + 
  msqc$flg_cmb_smoker + 
  msqc$flg_cmb_preop_transfusion + 
  msqc$flg_cmb_ventilator
```

### Split into open and closed datasets (to split analysis by surgical approach)

```r
msqc_open <- msqc[msqc$e_surgical_approach == 1,]
msqc_closed <- msqc[msqc$e_surgical_approach == 9 | msqc$e_surgical_approach == 8,]
```

### Check Proportion of complication in each subset

```r
msqc_open_cmp_prop <- sum(msqc_open$complication, na.rm = T) / nrow(msqc_open)
msqc_closed_cmp_prop <- sum(msqc_closed$complication, na.rm = T) / nrow(msqc_closed)
```

## BEGIN COUNTERFACTUAL ANALYSIS
**Not Finished**


```r
# Get cids with more than 15 operations

msqc_open <- msqc_open[!is.na(msqc_open$site_cid_160801),]
msqc_closed <- msqc_closed[!is.na(msqc_closed$site_cid_160801),]

msqc_open$site_cid_160801 <- as.factor(msqc_open$site_cid_160801)
msqc_closed$site_cid_160801 <- as.factor(msqc_closed$site_cid_160801)

open_cids <- names(summary(msqc_open$site_cid_160801))[summary(msqc_open$site_cid_160801) >= 25]
closed_cids <- names(summary(msqc_closed$site_cid_160801))[summary(msqc_closed$site_cid_160801) >= 25]

id <- "_006_"
model_obs <- msqc_open[msqc_open$site_cid_160801 == id, ]
loglm <- lm(complication ~ val_age + flg_asa + flg_female + n_comorbid)

for (i in 1:length(open_cids)){
  temp.df <- msqc_open
  temp.df$cid_flg <- temp.df$site_cid_160801 == open_cids[i]
  
  ## Fit model here, including cid_flg as a binary response
  ## record estimate and associated p-value
}
```