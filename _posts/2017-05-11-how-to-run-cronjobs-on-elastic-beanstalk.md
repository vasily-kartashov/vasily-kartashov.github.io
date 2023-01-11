---
layout: post
title: How to Run Cronjobs on Elastic Beanstalk
tags: aws php elastic-beanstalk
---

Elastic Beanstalk offers a convenient option for specifying cron tasks and running them on the leader instance, but there can be issues with this approach. For example, if the leader instance is terminated, your cron tasks will be suspended until the next deployment.

To avoid this issue, you can run cron jobs on every instance but have them exit early if the current instance is not the "first" in the autoscaling group. There are different ways to define what "first" means in this context. One option is to sort instance IDs alphabetically.

The following PHP snippet demonstrates how this can be achieved:

```php
$client = new ElasticBeanstalkClient(...);
$description = $client->describeEnvironmentResources([
    'EnvironmentName' => BEANSTALK_ENVIRONMENT;
]);
$currentId = file_get_contents("http://instance-data/latest/meta-data/instance-id");
$availableIds = [];
foreach ($description['EnvironmentResources']['Instances'] as $instance) {
    $availableIds[] = $instance['Id'];
}
sort($availableIds);
if ($currentId != $availableIds[0]) {
    $logger->info('Skipping cron execution on {id}. Leader is {leader}', [
        'id' => $currentId,
        'leader' => availableIds[0]
    ]);
    exit(0);
} else {
    $logger->info('Executing cron on {id}', ['id' => $currentId]);
}

...

```

This snippet first checks which instances are available, sorts the available instances by their ID alphabetically, and then compares the current instance's ID with the leader instance ID. If the current instance is not the leader, the script exits early, otherwise it continues to execute the cron job. This approach ensures that the cron jobs will run on every instance, thus preventing the suspension of cron jobs in case of leader instance termination.
