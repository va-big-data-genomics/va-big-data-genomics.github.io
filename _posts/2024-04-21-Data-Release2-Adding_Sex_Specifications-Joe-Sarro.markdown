# #16: Adding sex determinatation data from Hare to our dataset and visualizing missingness
##### Joe Sarro 04/21/2023

##### In today's meeting I will be presenting a [notebook](https://github.com/va-big-data-genomics/mvp-wgs-snp-indel-release/blob/main/SNPs-Indels/data_release_2023/notebooks/Filtering_Variants_With_Hare_Blog.pdf) stored as a PDF. This noteboks contains methods for loading Hare data, pulling out sex specification, and incorporating this information into our matrix table.

##### This new information can now be used to set sex specific paramaters for filtering variants. In addition to adding sex specification, the notebook shown today includes methods for filtering the X and Y chromosomes based on gender and a plot to visualize variant missingness per sample in chromosomes.

##### In a seperate notbook, I am currently comparing our previous filtering method to this new method that incorporates sex specific filtering. Preliminary observations suggest there is no change. I am currently taking steps to validate that both the method is working correctly and that adding a sex filter has no affect.
