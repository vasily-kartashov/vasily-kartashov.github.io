---
layout: post
title: How to Run Cronjobs on Elastic Beanstalk
---

Although Elastic Beanstalk has a nice option of specifiying a cron tasks and make them run on leader instance it doesn't always work. For example the leader might be terminated which is generally not an issue within an autoscaling environment, but with a teminated leader all your clon tasks are effectively gone till next redeployment.

One way to address it is to run cronjobs on every instance but leave early if the current instance is not the "first" in the autoscaling group. There are many possible ways to define what "first" should mean in this case. One option for example is to sort instance IDs alphabetically and take the first one.

The following PHP code would do exactly that

```
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