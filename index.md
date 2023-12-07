---
title: "Is Perl a write only language? "
date: 2023-12-06
---

I started using Perl in the early 21st century during the late phase of my PhD and postdoc to glue together 
various components in C and do some bioinformatics work in gene expression analysis. In the late 2000s switched
to R as my research interests became more aligned with clinical dataset analysis and biostatistical applications.
In the last couple of years, as my research on discovering RNA biomarkers from biofluids (blood, urine, and other
yucky stuff), I started using a combination of the (excellent) Bioconductor R facilities and considered Python for
the construction of applications regarding the analysis of my own (and other people's! biological sequencing data).
But something did not seem right ... so I looked back to my past. Could Perl be useful again? Despite many saying
it is a dead, read only language, the Perl of the 2020s is beautiful, feature rich and useful. 
Judge for your self if the following code (which downloads a bunch of sequence files from the ENSEMBL genome database
project, computes some basic statistics about the length of the RNA molecules of the Homo sapiens (that would be you!) 
is a throw away piece of code that you would not be able to decipher 30 minutes after you had written it. 
