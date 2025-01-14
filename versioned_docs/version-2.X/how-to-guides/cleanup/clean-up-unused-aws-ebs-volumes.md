---
sidebar_label: Clean Up Unused AWS EBS Volumes
sidebar_custom_props:
  tags: [AWS, EBS, EC2]
---

# How to Clean Up Unused AWS EBS Volumes

When EC2 instances are removed, their storage volumes are sometimes left behind. Resoto can find and delete these unused storage volumes.

## Prerequisites

This guide assumes that you have already [installed](../../getting-started/install-resoto/index.md) and configured Resoto to [collect your cloud resources](../../getting-started/configure-cloud-provider-access/index.md).

## Directions

1. Execute the following command in [Resoto Shell](../../concepts/components/shell.md) to open the [Resoto Worker](../../concepts/components/worker.md) configuration for editing:

   ```bash
   > config edit resoto.worker
   ```

2. Enable cleanup by modifying the `resotoworker` section of the configuration as follows:

   ```yaml
   resotoworker:
   # highlight-start
     # Enable cleanup of resources
     cleanup: true
     # Do not actually cleanup resources, just create log messages
     cleanup_dry_run: false
   # highlight-end
     # How many cleanup threads to run in parallel
     cleanup_pool_size: 16
   ```

   When cleanup is enabled, marked resources will be deleted as a part of the `collect_and_cleanup` [workflow](../../concepts/automation/workflow.md), which runs each hour by default.

   :::tip

   Set `cleanup_dry_run` to `true` to simulate cleanup without actually deleting resources.

   :::

3. Execute the following search in [Resoto Shell](../../concepts/components/shell.md) to list all unused EBS volumes:

   ```bash
   > search is(ebs_volume) and not /ancestors.instance
   ```

4. Refine the search criteria to only include unmounted volumes older than 30 days that have not been accessed in the last 7 days, in specific accounts:

   ```bash
   > search is(aws_ec2_volume) and /ancestors.account.reported.name in [eng-jenkins,eng-development] and volume_status = available and age > 30d and last_access > 7d
   ```

5. Now that we've defined the search for unused EBS volumes, simply pipe the result of the search query to the [`clean` command](../../reference/cli/action-commands/clean.md):

   ```bash
   > search is(aws_ec2_volume) and /ancestors.account.reported.name in [eng-jenkins,eng-development] and volume_status = available and age > 30d and last_access > 7d | clean
   ```

   :::note

   The [`clean` command](../../reference/cli/action-commands/clean.md) flags a resource for cleanup. Cleanup is performed whenever the `collect_and_cleanup` [workflow](../../concepts/automation/workflow.md) runs. The workflow runs every hour by default, but can also be manually triggered using the `workflow run cleanup` command.

   :::

6. Finally, let's automate flagging unused EBS volumes for cleanup by creating a [job](../../concepts/automation/job.md):

   ```bash
   > jobs add --id cleanup-unused-volumes --wait-for-event cleanup_plan 'search is(aws_ec2_volume) and /ancestors.account.reported.name in [eng-jenkins,eng-development] and volume_status = available and age > 30d and last_access > 7d | clean'
   ```

The job will now run each time Resoto emits the `cleanup_plan` event. The `cleanup_plan` event is a part of the `collect_and_cleanup` [workflow](../../concepts/automation/workflow.md) and emitted after resource collection is complete but before the cleanup is performed.

## Further Reading

- [Search](../../concepts/search/index.md)
- [Job](../../concepts/automation/job.md)
- [Workflow](../../concepts/automation/job.md)
- [Command-Line Interface](../../reference/cli/index.md)
