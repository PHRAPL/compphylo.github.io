---
layout: default
title: Tutorial 2 - Subsets and power analysis
parent: Tutorials
nav_order: 2
---





setwd("/working_path/sensitivityAnalyses")


########################
### 1. Subsampling  ####
########################


## Load libraries and functions
library(ape)
library(partitions)
library(phrapl)


## Define arguments

## Name of files(without path)
inputAssignment<-"Pleth_align.txt"
inputTrees<-"Pleth_bestTree.tre"       ##Tree file in NEWICK format
subsamplesPerGene<-10 


nloci<-5
popAssignments<-list(c(3,3))

currentAssign<-read.table(paste(getwd(),"/input/",inputAssignment,sep=""), header=TRUE)
currentTrees<-read.tree(paste(getwd(),"/input/",inputTrees,sep=""))
#currentAssign<-read.table(paste(path.package("phrapl"),"/extdata/Pleth_align.txt.txt",sep=""), header=TRUE,stringsAsFactors=FALSE)
#currentTrees<-read.tree(paste(path.package("phrapl"),"/extdata/Pleth_bestTree.tre",sep=""))

    
## Do subsampling
observedTrees<-PrepSubsampling(assignmentsGlobal=currentAssign,observedTrees=currentTrees,
popAssignments=popAssignments,subsamplesPerGene=subsamplesPerGene,outgroup=FALSE,outgroupPrune=FALSE)

## Save subsampled Trees
save(list="observedTrees",file=paste(getwd(),"/input/phraplInput_Pleth.rda",sep=""))


## Get subsample weights for phrapl
subsampleWeights.df<-GetPermutationWeightsAcrossSubsamples(popAssignments=popAssignments,observedTrees=observedTrees)

load(paste(getwd(),"/input/MigrationArray_2pop_3K.rda",sep=""))
migrationArrayMap<-GenerateMigrationArrayMap(migrationArray)

save(list=c("observedTrees","subsampleWeights.df","migrationArrayMap"),file=paste(getwd(),"/input/phraplInput_Pleth.rda",sep=""))


######################
### 2. GridSearch  ###
######################
#######################Subsampling was done before

## Load libraries
library(ape)
library(partitions)
library(lattice)
library(polynom)
library(gmp)
library(rgenoud)
library(parallel)
library(optimx)
library(igraph)
library(numDeriv)
library(nloptr)
library(Matrix)
library(rgl)
library(RColorBrewer)
library(base)
library(phrapl)

## Load input files (from subsampling) if the run is starting here
load(paste(getwd(),"/input/phraplInput_Pleth.rda",sep=""))                 #load the input file 
load(paste(getwd(),"/input/MigrationArray_2pop_3K.rda",sep=""))
#load(paste(path.package("phrapl"),"/extdata/phraplInput_Pleth.rda",sep=""), header=TRUE,stringsAsFactors=FALSE)
#load(paste(path.package("phrapl"),"/extdata/MigrationArray_2pop_3K.rda",sep=""))


## Specify search details

#modelRange<-c(1:length(migrationArray))        
modelRange<-c(1:5)
popAssignments<-list(c(3,3))  
nTrees<-100      
subsamplesPerGene<-10
totalPopVector<-list(c(4,4))     ## total number of indvs per pop

## Run search and keep track of the time
startTime<-as.numeric(Sys.time())

result<-GridSearch(modelRange=modelRange,
	migrationArray=migrationArray,
	migrationArrayMap=migrationArrayMap,           # do not required in the new version CHECK
	popAssignments=popAssignments,
	nTrees=nTrees,
	#	msPath="/Users/ariadnamoralesgarcia/msdir/ms",
	#	comparePath="/Library/Frameworks/R.framework/Versions/3.1/Resources/library/phrapl/extdata/comparecladespipe.pl",
	observedTree=observedTrees,
	subsampleWeights.df=subsampleWeights.df,
	print.ms.string=TRUE,
	print.results=TRUE,
	debug=TRUE,return.all=TRUE,
	collapseStarts=c(0.30,0.58,1.11,2.12,4.07),
	migrationStarts=c(0.10,0.22,0.46,1.00,2.15),
	subsamplesPerGene=subsamplesPerGene,
	totalPopVector=totalPopVector,
	popScaling=c(0.25, 1, 1, 1, 1),
	print.matches=TRUE)

#Print summary results to end of Rout file
print(result[[1]])

#Make dedicated grid list
gridList<-result[[1]]

#Get elapsed time
stopTime<-as.numeric(Sys.time()) #stop system timer
elapsedSecs<-stopTime - startTime #elapsed time in hours
elapsedHrs<-(elapsedSecs / 60) / 60 #convert to hours
elapsedDays<-elapsedHrs / 24 #convert to days

#Save the workspace from first grid analysis and create output folders
system(paste("mkdir ", getwd(),  "/results", sep=""))
save(list=ls(), file=paste(paste(getwd(),"/results/Pleth_",min(modelRange),"_",max(modelRange),".rda",sep="")))
system(paste("mkdir ", getwd(),  "/results/RoutFiles", sep=""))
system(paste("mv ", getwd(), "/scripts/1.Subsampling_GridSearch_Post.Rout ", getwd(), "/results/RoutFiles/1.Subsampling_GridSearch_Post.Rout", sep=""))
system(paste("rm ", getwd(), "/scripts/1.Subsampling_GridSearch_Post.R.out", sep=""))


########################
### 3. Post-process  ###
########################

## If different models (from the same migrationArray file) were run separatly, the output can be concatenated.
totalData<-ConcatenateResults(rdaFilesPath=paste(getwd(),"/results/",sep=""),rdaFiles=NULL,addAICweights=TRUE,addTime.elapsed=FALSE)
modelAverages<-CalculateModelAverages(totalData, parmStartCol=9)

## Save output to a txt file
write.table(totalData, file=paste(getwd(),"/results/totalData.txt",sep=""), sep="\t", row.names=FALSE, quote=FALSE)

## Plot the best model
#PlotModel(migrationArray[[1]], taxonNames=c("S","N"))
#PlotModel(migrationArray[[6]], taxonNames=c("S","N"))
#PlotModel(migrationArray[[3]], taxonNames=c("S","N"))


setwd("/working_path/sensitivityAnalyses")

################################
### 4. Sensitivity analyses  ###
################################

## Load libraries
library(phrapl)
library(gplots)

## Load modified function
source(paste(getwd(), "/scripts/GenerateSetLoci_cum8_v3.R", sep=""))

## Load phrapl input (subsampled dataset)
load(paste(getwd(),"/input/phraplInput_Pleth.rda", sep=""))

## Load file with models and store them in an object named "migrationArray"
load(paste(getwd(),"/input/MigrationArray_2pop_3K.rda",sep=""))

## Create output folders
system(paste("mkdir ", getwd(), "/results/subsets",sep=""), ignore.stderr=TRUE)

#########################################################
### Define arguments (some of these were already defined)
nloci<-5
NintervalLoci<-1
modelRange<-c(1:5)
lociRange<-c(1:5)
subsamplesPerGene<-10
popAssignments<-list(c(3,3))
collapseStarts<-c(0.3, 0.58, 1.11, 2.12, 4.07)   
migrationStarts<-c(0.1, 0.22, 0.46, 1, 2.15) 
n0multiplierStarts<-NULL
setCollapseZero<-NULL
nTrees<-100
dAIC.cutoff<-2
nEq<-100
subsampleWeightsVec=subsampleWeights.df[[1]][,1]
migrationArrayShort<-migrationArray[modelRange]

#####################################
### Load files with match vectors ###
rdaFilename<-paste(getwd(),'/results/Pleth_1_5.rda', sep="")

system(paste("grep matches -A1 ",getwd(),"/results/RoutFiles/1.Subsampling_GridSearch_Post.Rout | grep -v matches | grep 0 > ",getwd(),"/results/RoutFiles/matches_2.RunGridSearch.Rout.txt", sep=""))
RoutFilename<-read.table(file=paste(getwd(),'/results/RoutFiles/matches_2.RunGridSearch.Rout.txt', sep="") ,skip=1)

#####################################################
### Recalculate model averages in subsets of loci ###

# Command line to run function GenerateSetLoci
GenerateSetLoci(lociRange=lociRange,NintervalLoci=NintervalLoci,RoutFilename,rdaFilename,migrationArray=migrationArray,modelRange=modelRange,subsamplesPerGene=subsamplesPerGene,collapseStarts=collapseStarts,migrationStarts=migrationStarts,n0multiplierStarts=n0multiplierStarts,setCollapseZero=setCollapseZero,cumulative=TRUE,nTrees=nTrees,dAIC.cutoff=2,nEq=nEq, subsampleWeightsVec=subsampleWeightsVec)


#########################################
### Postprocess RDA files per subset  ###

# Set paths to input and output files
pathSubsets<-paste(getwd(), "/results/subsets/", sep="")              # Files created by GenerateSetLoci. Do not forget "/" at the end
pathtotalData_subsets<-paste(getwd(), "/results/subsets/", sep="") # Do not forget "/" at the end

# Loop to analyze output files ----> Will generate a totalData file per subset per model
# You may need to create one folder per subset and move files to each folder
for(i in 1:5){                    ## ------> Be careful to specify the total number of subsets (in this case 101)
    totalData<-list()
    totalData<-ConcatenateResults(rdaFilesPath=paste(getwd(), "/results/subsets/subset",i,"/",sep=""),rdaFiles=NULL, migrationArray, rm.n0=TRUE, longNames = TRUE, outFile=NULL,addAICweights=TRUE,addTime.elapsed=FALSE, nonparmCols = 4)
	#modelAverages<-CalculateModelAverages(totalData,parmStartCol=9)
    write.table(totalData, file=paste(pathtotalData_subsets,"totalData_subset",i,".txt",sep=""), sep="\t", row.names=FALSE)
}

########################
### 5. Plot subsets  ###
########################

pathInputFiles<-paste(getwd(), "/results/subsets/",sep="")

totalData_list<-list.files(pathInputFiles, pattern="*.txt", full.names=FALSE)

totalData_files<-c()
for(x in 1:length(totalData_list)){
    totalData_files<-append(totalData_files,list(read.table(paste(pathInputFiles,totalData_list[x],sep=""),header=TRUE, fill=TRUE, sep="\t")))
}

#Sort each dataframe by model
totalData_filesSorted<-c()
for(x in 1:length(totalData_list)){
	totalData_filesSorted<-append(totalData_filesSorted,list(totalData_files[[x]][order(totalData_files[[x]]$models),]))
}

#Create a dataframe with all models and their wAIC
model_wAIC<-data.frame("models"=c(1:5),row.names=NULL)
for(x in 1:length(totalData_list)){
    model_wAIC[,paste("Subset ",x,sep="")]<-totalData_filesSorted[[x]][,"wAIC"]
}

########################
## Define plot settings

## Row Labels
row.label<-c()
setLoci<-0
for(i in 1:length(totalData_list)){
    setLoci<-setLoci+8      ##loci interval
    row.label<-append(row.label, c(paste(setLoci, " Loci",sep="")))
}

## Row labels ---> 
row.labels <- rep("", 5)

## Col Labels
col.label<-seq(1:5)

## Color scale
bks<-c(seq(0,0.99,by=0.05))
lm.palette <- colorRampPalette(c("light yellow","orange", "tomato"), space = "rgb")

## Possition of the plot and legend
lmat <- rbind(c(0,3,3,3),c(2,1,1,1),c(0,0,0,4))
lwid <- c(0.2,1,1,1)
lhei <- c(0.3,2.1,0.5)

pdf(file=paste(getwd(),"/results/wAICinSubsets.pdf", sep=""), width=9, height=7)
heatmap.2(as.matrix(t(model_wAIC[2:length(model_wAIC)])), lmat = lmat, lhei=lhei,lwid=lwid,
	Rowv=FALSE,Colv=FALSE, dendrogram="none", trace="none",
	margins=c(2,12),
	col=lm.palette(length(bks)-1), breaks=bks,
	#colsep=1:5,
	rowsep=1:(length(model_wAIC)-1), sepcolor="white",
	#labRow=row.labels,
	#labCol=col.label,
	key = TRUE,
	#keysize =1.0,
	density.info=c("density"),
	denscol=tracecol,
	key.xlab="wAIC",
	key.par=list(mar=c(4,4,4,4)),
	main="wAIC for each model per locus")
dev.off()


R CMD BATCH 1.Subsampling_GridSearch_Post.R > 1.Subsampling_GridSearch_Post.R.out
R CMD BATCH 2.Subsets.R > 2.Subsets.out
rm *.Rout
rm *.out