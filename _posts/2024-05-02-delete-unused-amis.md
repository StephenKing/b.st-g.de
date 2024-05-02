---
layout: post
title:  "Delete unused AMIs using the new 'LastLaunchedTime' attribute"
date:   2024-05-02 15:00:00 +0000
tags: [aws]
description: Reduce your AWS costs by (more) safely deleting unused AMIs.

comments: true
share: true

---

When you as AWS user bake your own AMIs, e.g., using Packer, then you likely end up with having many of them after a while.
So far, there has been no easy way to identify, which AMIs are actually in use, and which could be deleted. While you live in this uncertainty, you pay AWS for the disk space they consume. We did so for a long time...

Yesterday, AWS has added a new attribute to [simplify visibility into active AMIs](https://aws.amazon.com/de/about-aws/whats-new/2024/05/amazon-ec2-simplifies-visibility-active-amis/). While I would still argue that this does _not_ list the AMIs in use by any currently running EC2 instances, it is at least a step in the right direction.
Using the `LastLaunchedTime` attribute, you can now see, when - and if at all - this AMI has been used. Luckily, this works also across accounts.

As some of our teams have automated workflows to build weekly AMIs, but not necessarily use them, I saw a savings potential going into the thousands of dollars.

## Finding unused AMIs

My first reflex seeing this announcement was to delete all AMIs that were a) created before year 2024 and b) never launched. This is what's covered in this blog post.

Unfortunately, I wasn't able to filter for `Last launched time` to be null in the AWS Console.

![AWS Console: filtering options for 'Last launched time'](/images/2024-02-02-aws-ami-deletion/console.png)

Anyway, the AWS CLI is the better choice. This is how you can list your AMIs using the AWS CLI, no matter if they ever have been launched or not:

```bash
$ aws ec2 describe-images --owners self --query 'Images[?CreationDate<`2024-01-01`] | sort_by(@, &CreationDate)[].[ImageId,ImageLocation,LastLaunchedTime]'
[
    [
        "ami-1234567890abcdef0",
        "123456789012/foo/bar/mycorp-base-amd64-server-ubuntu-22.04-20220501...",
        "2022-05-05T01:02:03Z"
    ],
    [
        "ami-1234567890abcdef1",
        "123456789012/foo/bar/mycorp-base-amd64-server-ubuntu-22.04-20220508...",
        null
    ],
...
```

To now only list the images which have never been launched, you can use the following command (adding the beautiful double negation `!not_null(LastLaunchedTime)`):

```bash
$ aws ec2 describe-images --owners self --query 'Images[?CreationDate<`2025-01-01` && !not_null(LastLaunchedTime)] | sort_by(@, &CreationDate)[].[ImageId,ImageLocation,LastLaunchedTime]'
[
    [
        "ami-1234567890abcdef1",
        "123456789012/foo/bar/mycorp-base-amd64-server-ubuntu-22.04-20220508...",
        null
    ],
...
```

For us, this list has been really, really long.

## Deleting unused AMIs - and snapshots

By iterating over the list of unused AMIs, you can now delete them using the AWS CLI:

```bash
for image in $(aws ec2 describe-images --owners self --query 'Images[?CreationDate<`2024-01-01` && !not_null(LastLaunchedTime)] | sort_by(@, &CreationDate)[].[ImageId][]' --output text); do
  echo "# $image";
  echo aws ec2 deregister-image --image-id $image; # remove the echo to execute it 
done
```

⚠️⚠️⚠️️️ As you should be careful with executing some random dude's code against your AWS account, I have added an extra `echo` statement to only output the delete commands.
After you've reviewed the output, either remove the `echo` or  copy & paste it into your shell.  

Deleting the AMI itself doesn't save any money. What actually lowers the bill is finally deleting the EBS snapshots that unterpin the AMIs:

![EC2 Console listing EBS snapshots](/images/2024-02-02-aws-ami-deletion/snapshots.png)

Given that one can't delete a snapshot, which is in use by an AMI, you can just go ahead and try to delete all EBS snapshots - assuming that you have no manually created snapshots in this region that you want to keep.
To be sure, let's limit deletion to EBS snapshots that have a description "Created by CreateImage" - or "Copied for DestinationAmi" (latter one is used in cases when an AMI is copied to a different region).

![EC2 Console listing EBS snapshots, which were created from cross-region AMI copy](/images/2024-02-02-aws-ami-deletion/snapshots-copy.png)

This will list your snapshots:

```bash
$ aws ec2 describe-snapshots --owner self --filters "Name=description,Values='Created by CreateImage*','Copied for DestinationAmi*'"
```

After you've double-checked, you can (what should go wrong - did I mention you do everythign at your own risk?) go ahead and try to delete all of them:


```bash
$ for snap in $(aws ec2 describe-snapshots --owner self --filters "Name=description,Values='Created by CreateImage*','Copied for DestinationAmi*'" --query "Snapshots[*].SnapshotId" --output text); do
  echo "# $snap";
  echo aws ec2 delete-snapshot --snapshot-id $snap; # remove the echo to execute it 
done
```

You will see failures of deleting snapshots that are still associated with an existing AMI:

> An error occurred (InvalidSnapshot.InUse) when calling the DeleteSnapshot operation: The snapshot snap-0bcf7289e9ad65306 is currently in use by ami-0ad9bdfd83929399f


## Cleaning all regions

So far so good, but this was only a single region. If you want to iterate over all (enabled) regions, here is the code (which I find surprisingly hard to find through a Google search):

```
$ aws account list-regions --region-opt-status-contains ENABLED ENABLED_BY_DEFAULT --query "Regions[].RegionName" --output text
```

For the brave ones of you, this is how you can clean up all your regions (usual safeguard included):

```bash
$ for region in $(aws account list-regions --region-opt-status-contains ENABLED ENABLED_BY_DEFAULT --query "Regions[].RegionName" --output text); do
  echo "#####################################";
  echo "# $region";
  echo "#####################################";
  
  for image in $(aws ec2 describe-images --owners self --query 'Images[?CreationDate<`2024-01-01` && !not_null(LastLaunchedTime)] | sort_by(@, &CreationDate)[].[ImageId][]' --output text --region $region); do
    echo "# $image";
    echo aws ec2 deregister-image --image-id $image --region $region; # remove the echo to execute it  
  done
  
  for snap in $(aws ec2 describe-snapshots --owner self --filters "Name=description,Values='Created by CreateImage*','Copied for DestinationAmi*'" --query "Snapshots[*].SnapshotId" --output text --region $region); do
    echo "# $snap";
    echo aws ec2 delete-snapshot --snapshot-id $snap --region $region; # remove the echo to execute it
  done
  
  echo "# Finished $region"
  echo
done

```



## Next Steps

This has cleaned up all AMIs plus underlying EBS snapshots, which were a) created before 2024 and b) never ever used to launch an EC2 instance.
Cleaning up AMIs that have been used to start an EC2 instance before... let's say 2023... is left as an exercise to the reader or an update to this blog post.
However, the devil might be more in the detail here, as you should understand your infrastructure well, if you might ever still need those AMIs.

I'm curious to see our bill tomorrow.