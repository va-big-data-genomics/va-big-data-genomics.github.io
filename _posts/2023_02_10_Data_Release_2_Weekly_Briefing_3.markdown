# Data Release 2 Weekly Briefing 3 for 02-10-2023

## Current updates 

I have investigated the issue involving high missingness rates on the X chromosome. While I am still working on visualizing global missingness separated by chromosome, I have created a plot that will show missingness per variant for individual chromosomes. 

Using this methodology, I was able to track missingness on the X chromosome while updating our methods to filter genotypes. I was also able to look at the affect of using Gnomads filtering methods on a subset of our data. After looking at our filter criteria, it appears that high rates of missingness on the X chromosome may be due to low genotype quality scores. 

More detailed analysis can be viewed in the [missingness analysis notebook](https://drive.google.com/file/d/157GIt0LxN9LOdbKmCg4mxhHWAIXoytOk/view?usp=share_link) that I generated. Unfortunately, Bokeh plots do not save when closing the notebook. Because of this, the previous link is to an html version that you can hopefully view in your web browser. 

