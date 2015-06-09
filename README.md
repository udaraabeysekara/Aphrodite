# Aphrodite

Automated PHenotype Routine for Observational Definition, Identification, Training and Evaluation

Requirementes
===================
- CDM v5
- R Packages:
	- caret
	- dplyr
	- pROC
	- devtools
	- SqlRender
	- DatabaseConnector

Before you start
===================

You need to udpate the settings.R file with your CDM connection information and Phenotyping settings.

Full Example
===================

```r

folder = "/home/jmbanda/OHDSI/Aphrodite/R/" # Folder containing the R files and outputs, use forward slashes
setwd(folder)

source("settings.R")

###########################################################
# All parameters connection info and extras are in        #
# the file settings.R, please make any appropiate changes #
###########################################################

#Initiate connection
connectionDetails <- createConnectionDetails(dbms=dbms, server=server, user=user, password=pw, schema=cdmSchema, port=port)
conn <- connect(connectionDetails)

# STEP 1 - Generate Keywords
status <- buildKeywordList(conn, aphrodite_concept_name, cdmSchema)
message(status)

# Load Keyword list after editing
keywordList_FF <- read.table('keywordlist.tsv', sep="\t", header=FALSE)
ignoreList_FF <- read.table('ignorelist.tsv', sep="\t", header=FALSE)


# STEP 2 - Get cases, controls

casesANDcontrolspatient_ids_df<- getdPatientCohort(conn,as.character(keywordList_FF$V3),as.character(ignoreList_FF$V3), cdmSchema,nCases,nControls)
if (nCases > nrow(casesANDcontrolspatient_ids_df[[1]])) {
    message("Not enough patients to get the number of cases specified")
    stop
} else {
    if (nCases > nrow(casesANDcontrolspatient_ids_df[[2]])) {
        message("Not enough patients to get the number of controls specified")
        stop
    }
}

cases<- casesANDcontrolspatient_ids_df[[1]][sample(nrow(casesANDcontrolspatient_ids_df[[1]]), nCases), ]
controls<- casesANDcontrolspatient_ids_df[[2]][sample(nrow(casesANDcontrolspatient_ids_df[[2]]), nControls), ]

if (saveALLresults) {
    write.table(cases, file=paste('cases.tsv',sep=''), quote=FALSE, sep='\t', row.names = FALSE, col.names = FALSE)
    write.table(controls, file=paste('controls.tsv',sep=''), quote=FALSE, sep='\t', row.names = FALSE, col.names = FALSE)
}

# STEP 3 - Get Patient Data

#Cases needs its own function
dataFcases <-getPatientDataCases(conn, cases, as.character(keywordList_FF$V3),as.character(ignoreList_FF$V3), flag , cdmSchema)
if (saveALLresults) {
    save(dataFcases,file=paste(studyName,"-RAW_FV_CASES_",as.character(nCases),".Rda",sep=''))
}

#Get Controls
dataFcontrols <- getPatientData(conn, controls, flag , cdmSchema)
if (saveALLresults) {
    save(dataFcontrols,file=paste(studyName,"-RAW_FV_CONTROLS_",as.character(nControls),".Rda",sep=''))
}

# Build feature vectors
fv_all<-buildFeatureVector(flag, dataFcases,dataFcontrols)
if (saveALLresults) {
    save(fv_all,file=paste(studyName,"-FULL_FV_CASES_",as.character(nCases),"_CONTROLS_",as.character(nControls),".Rda",sep=''))
}

# Step 4 - Build Model

model_predictors <- buildModel(flag, cases, controls, fv_all, outcomeName)

###### Save Model to file #############
model<-model_predictors$model
predictors<-model_predictors$predictors

save(model, file=paste(flag$model[1], " MODEL FILE FOR ",outcomeName,".Rda",sep=''))
#Save Predictors for model
save(predictors, file=paste(flag$model[1], " PREDICTORS FOR ",outcomeName,".Rda",sep=''))



#Close connection
dbDisconnect(conn)

```

License
=======
Aphrodite is licensed under Apache License 2.0

Development
============
Aphrodite is being developed in R Studio.
