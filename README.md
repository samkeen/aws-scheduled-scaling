This is mostly an experiment I undertook to practice working with AWS [Cloud Formation](http://aws.amazon.com/cloudformation/)
and the AWS [Scheduled Scaling](http://docs.aws.amazon.com/AutoScaling/latest/DeveloperGuide/schedule_time.html) feature.  It may not prove useful as a whole, but I am still learning quite a bit exploring the technology.

It is **not** meant to be something you would use as is in your business although
some of the ideas and techniques leveraged here could certainly be utilized in a Continuous Integration 
and or QA system for your company.

The script here will launch an auto-scale group which consists of a single EC2 instance with a running Postgres Db Server.
It uses the stock Ubuntu AMI and via a cloud-init script configures itself as a functional Postgres Db server on first boot.
The created Auto Scale Group used the AWS [Scheduled Scaling](http://docs.aws.amazon.com/AutoScaling/latest/DeveloperGuide/schedule_time.html)
feature.  As a cost saving measure the scaling group will scale to zero instances in the evening the then back to one in 
the early morning.

The thought is that this could be your QA and/or CI Testing db which in not used in the off hours.

The Cloud Formation Template is fully self contained.  It creates its own VPC and public subnet to contain everything it
creates so it is safe to launch on your own AWS account.

You would run the template to create the stack only once, then allow the ASG scheduling to scale up and down the Db instance.
Note if you create the stack in the *Off Hours* time span, you will not see an EC2 instance start until the time enters the 
*On Hours* time span.

Here is the Template in its entirety is [here](https://github.com/samkeen/aws-scheduled-scaling/blob/master/cf-templates/db-scheduled-scaling.json)

Scheduled Scaling is not exposed all that much in the AWS console so it can be difficult to determine if it is
active and what it is set to in the while in the Console.  You can list all your Scheduled Scaling actions using the AWS CLI 
as shown below.

```json
> aws autoscaling describe-scheduled-actions
{
    "ScheduledUpdateGroupActions": [
        {
            "MinSize": 1,
            "DesiredCapacity": 1,
            "AutoScalingGroupName": "non-prod-db_ASG",
            "MaxSize": 1,
            "Recurrence": "0 21 * * *",
            "ScheduledActionARN": "arn:aws:autoscaling:us-west-2:<ACCOUNT NUMBER>:scheduledUpdateGroupAction:2f69b178-7dbf-4244-96cd-fd15b5c15338:autoScalingGroupName/non-prod-db_ASG:scheduledActionName/lights-ON",
            "ScheduledActionName": "lights-ON",
            "StartTime": "2015-05-10T21:00:00Z",
            "Time": "2015-05-10T21:00:00Z"
        },
        {
            "MinSize": 0,
            "DesiredCapacity": 0,
            "AutoScalingGroupName": "non-prod-db_ASG",
            "MaxSize": 0,
            "Recurrence": "0 13 * * *",
            "ScheduledActionARN": "arn:aws:autoscaling:us-west-2:<ACCOUNT NUMBER>:scheduledUpdateGroupAction:54cc4c39-a000-4d3a-9def-0a7a09f93a08:autoScalingGroupName/non-prod-db_ASG:scheduledActionName/lights-OUT",
            "ScheduledActionName": "lights-OUT",
            "StartTime": "2015-05-11T13:00:00Z",
            "Time": "2015-05-11T13:00:00Z"
        }
    ]
}

```

Also from the CLI, you can delete and create them as shown here:

Add the ***Light Out*** scheduled action for every day at 9pm Pacific time (9pm + 8 hours = **05:00 UTC**) 

```bash
> aws autoscaling put-scheduled-update-group-action\
 --auto-scaling-group-name db_scale_ASG \
 --scheduled-action-name lights-OUT \
 --recurrence "0 5 * * *" \
 --min-size 0 \
 --max-size 0 \
 --desired-capacity 0
```

Add the ***Lights On*** action for every day at 5am  Pacific time (5am + 8 hrs = **13:00 UTC**)

```bash
> aws autoscaling put-scheduled-update-group-action\
 --auto-scaling-group-name db_scale_ASG \
 --scheduled-action-name lights-ON \
 --recurrence "0 13 * * *" \
 --min-size 1 \
 --max-size 1 \
 --desired-capacity 1
```

To Delete a Scheduling action

```bash
> aws autoscaling delete-scheduled-action \
--auto-scaling-group-name non-prod-db_ASG  \
--scheduled-action-name lights-OUT
```

# Troubleshooting

If you cannot connect to the launched instance via Postgres, ssh to the instance and examine `/var/log/cloud-init-output.log`.
This is the STDOUT for the run of the cloud init script.  Any issues with the initial configuration should be apparent there.
