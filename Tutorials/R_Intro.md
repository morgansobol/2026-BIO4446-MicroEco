# Intro to R Studio

R is a free, open-source programming language designed for statistical computing and data visualization, while RStudio is an Integrated Development Environment that provides a user-friendly interface for writing and managing R code. 
 
<img width="1280" height="720" alt="image" src="https://github.com/user-attachments/assets/a5f776e1-55a7-47f2-b541-104df7262122" />

## Commonly used R scripts 

Where am I?
```R
getwd()
```
Change my working directory to the desired location
```R
setwd("[insert path to directory you want to be in]")
```
 List the contents of the directory
```R
dir()
```
Import a csv file
```R
filename_df <-read.csv("path/to/a/dataset/ending/in/filename.csv",
                   header = TRUE, sep=",", strip.white=TRUE, stringsAsFactors=FALSE)
```
## Variables
Variables are containers for storing data values.

Write a new variable for R to use. A variable is a name that stores a value or some data.  
```bash
new_df = filename_df
new_df <- filename_df
filename_df -> new_df
```
All three mean the same thing: you are creating a new variable called `new_df` that stores whatever is in `filename_df` (for example, a dataframe).

### Other commands
Use `print()` to see what is inside a variable
```R
print(filename_df)
```

List column and row names of the data
```R
colnames(filename_df)
rownames(filename_df)
```
See the structure of the data
```R
str(filename_df)
```
View the data 
```R
view(filename_df)
```
Get basic stats
```R
summary(filename_df)
```

Write a table/data matrix as a tab-delimited file
```R
write.table(object, "filename.tsv", sep="\t")
```

Ok, let's begin!
