Boot A Redshift Cluster From A Snapshot Using R and AWS CLI
================

This script is a quick proof-of-concept to explore connecting to AWS Redshift, identifying a cluster snapshot, restoring that cluster, then shutting it down again. This is a first step in a package which could help automate some database management tasks

-Tim Kiely, 9/9/2016

`tkiely@hodgeswarelliott.com`

SET UP AWS CLI
==============

Note that you must first set up the aws CLI, see: <http://docs.aws.amazon.com/cli/latest/userguide/installing.html>

You will also need to grant access to your user to access your Redshift db through IAM settings, see: <http://docs.aws.amazon.com/redshift/latest/mgmt/redshift-iam-authentication-access-control.html>

Initiating the cluster from a snapshot
======================================

Using `system()` commands, we grab the most recent snapshot from our AWS account, tell the cluster to boot up, then ping the server every few seconds until the cluster is ready to go.

``` r
# issue an AWS CLI command to list all available Redshift snapshots, catpure as JSON
systext_snapshots <- system("aws redshift describe-cluster-snapshots",intern = T)


# convert the snapshot data from JSON to a dataframe
snapshot_frame <- jsonlite::fromJSON(systext_snapshots)

# Grab the most recent snapshot
most_recent_snap <- dplyr::arrange(snapshot_frame$Snapshots, desc(SnapshotCreateTime)) %>% head(n=1)

# create unique snapshot temp identifier
clust_id <- paste0("taxi-cluster-temp-",as.integer(Sys.time()),collapse = "-")

# issue CLI command to create a temp redshift cluster, from the snapshot
restore_command <- paste0("aws redshift restore-from-cluster-snapshot --cluster-identifier ", clust_id," --snapshot-identifier ", most_recent_snap$SnapshotIdentifier)
system(restore_command)


# clusters may take 5-20 minutes to become available. Proceed once the status changes to "available"

status<-"new"

start_time <- Sys.time()

message("Pinging the server every 5 seconds until the cluster is available. \nThis usually takes 5-10 minutes")

while(status != "available"){
  cat("Checking...\n")
  Sys.sleep(5)
  systext_check <- jsonlite::fromJSON(system("aws redshift describe-clusters",intern = T), flatten = T)
  
  # extract the correct cluster based on the unique ID:
  systext_check_temp <- dplyr::filter(as.data.frame(systext_check$Clusters), ClusterIdentifier==clust_id)
  
  # set status. if not "available", repeat loop
  status <- systext_check_temp$ClusterStatus
}; beepr::beep(4)

end_time <- Sys.time()



# grab status again
systext_nodes <- jsonlite::fromJSON(system("aws redshift describe-clusters",intern = T), flatten = T)
# our_nodes <- systext_nodes$Clusters
our_nodes <- if(exists('clust_id')){
  filter(systext_nodes$Clusters, ClusterIdentifier == clust_id)
} else systext_nodes[[1]]



# extract address and port, to be used for querrying later
address <- our_nodes$Endpoint.Address 
port <- our_nodes$Endpoint.Port
```

Shutting the cluster down
=========================

``` r
# a command to shut the cluster down, using the cluster-identifier
shut_down_cmd <- paste0("aws redshift delete-cluster --cluster-identifier "
                        , our_nodes$ClusterIdentifier
                        ," --skip-final-cluster-snapshot")

system(shut_down_cmd)



closeAllConnections()
message("Cluster boot time: ",round(end_time-start_time,2),units(end_time,start_time))
```
