

We are looking to create a new Databricks environment in Gelsight.  We have already created an Azure Landing Zone and set up all of the networking.
The Azure Databricks service has also been created as has the compute.
We need to accomplish the following:

You can see the ProjectSumary.md for details about the data

1.  Set up a medallion architecture.  This means a bronze, silver and gold data table.  
2.  I would assume that we would want to set up a unity catalog to be the brain of the entire operation.  This would mean that we would not mount file systems, etc.
3.  We need source control and code promotion.
4.  At the end of this will be a machine learning model.  It isn't in scope right now, but I want to keep that in mind as we build.

I want you to walk me through all of this

