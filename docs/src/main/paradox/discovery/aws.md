## AWS API

@@@ warning

This module is community maintained and the Lightbend subscription doesn't cover support for this module.

@@@

If you're using EC2 directly _or_ you're using ECS with host mode networking
_and_ you're deploying one container per cluster member, continue to
@ref:[Discovery Method: AWS API - EC2 Tag-Based Discovery](aws.md#discovery-method-aws-api-ec2-tag-based-discovery).

If you're using ECS with
[awsvpcs](https://aws.amazon.com/blogs/compute/introducing-cloud-native-networking-for-ecs-containers/)
mode networking (whether on EC2 or with
[Fargate](https://aws.amazon.com/fargate/)), continue to
@ref:[Discovery Method: AWS API - ECS Discovery](aws.md#discovery-method-aws-api-ecs-discovery).

ECS with bridge mode networking is not supported.

If you're using EKS, then you may want to use the
@ref:['Kubernetes API'-based discovery method](kubernetes.md)
instead.


## Project Info

@@project-info{ projectId="discovery-aws-api" }

### Discovery Method: AWS API - EC2 Tag-Based Discovery

You can use tags to simply mark the instances that belong to the same cluster. Use a tag that
has "service" as the key and set the value equal to the name of your service (same value as `pekko.management.cluster.bootstrap.contact-point-discovery.service-name`
defined in `application.conf`, if you're using this module for bootstrapping your Pekko cluster).

Screenshot of two tagged EC2 instances:

![EC2 instances](../images/discovery-aws-ec2-tagged-instances.png)

Note the tag **service** -> *products-api*. It is set on both instances.

Note that this implementation is adequate for users running service clusters on *vanilla* EC2 instances. These
instances can be created and tagged manually, or created via an auto-scaling group (ASG). If they are created via an ASG,
they can be tagged automatically on creation. Simply add the tag to the auto-scaling group configuration and
ensure the "Tag New Instances" option is checked.


#### Dependencies and usage (EC2 Tag-Based Discovery)

This is a separate JAR file:

@@dependency[sbt,Gradle,Maven] {
  symbol1=PekkoManagementVersion
  value1=$project.version$
  group="org.apache.pekko"
  artifact="pekko-discovery-aws-api_$scala.binary.version$"
  version=PekkoManagementVersion
}

`pekko-discovery-aws-api` can be used with Pekko $pekko.version$ or later.
You have to override the following Pekko dependencies by defining them explicitly in your build and
define the Pekko version to the one that you are using. Latest patch version of Pekko is recommended and
a later version than $pekko.version$ can be used.

@@dependency[sbt,Gradle,Maven] {
  symbol=PekkoVersion
  value=$pekko.version$
  group=org.apache.pekko
  artifact=pekko-cluster_$scala.binary.version$
  version=PekkoVersion
  group2=org.apache.pekko
  artifact2=pekko-discovery_$scala.binary.version$
  version2=PekkoVersion
}

And in your `application.conf`:

```
pekko.discovery {
  method = aws-api-ec2-tag-based
}
```

Notes:

* You will need to make sure that the proper privileges are in place for the discovery implementation to access
the Amazon EC2 API. The simplest way to do this is by creating a IAM role that, at a minimum, allows the _DescribeInstances_ action.
Attach this IAM role to the EC2 instances that need to access the discovery implementation. See the docs for
[IAM Roles for Amazon EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html).


* In general, for the EC2 instances to "talk to each other" (necessary for forming a cluster), they need to be in the
same security group and [the proper rules have to be set](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/security-group-rules-reference.html#sg-rules-other-instances).

* You can set additional filters (by instance type, region, other tags etc.) in your application.conf file, in the
`pekko.discovery.aws-api-ec2-tag-based.filters` key. The filters have to be key=value pairs separated by the semicolon
character. For example:
    ```
    pekko {
      discovery {
        aws-api-ec2-tag-based {
          filters = "instance-type=m1.small;tag:purpose=production"
        }
      }
    }
    ```

* By default, this module is configured for clusters with one Pekko node per EC2 instance: it
  separates cluster members solely by their EC2 IP address. However, we can change the default configuration to indicate multiple ports per discovered EC2 IP, and achieve
a setup with multiple Pekko nodes (i.e. multiple JVMs) per EC2 instance.
    ```
    pekko {
      discovery {
        aws-api-ec2-tag-based {
          ports = [7625, 7626, 7627] # 3 Pekko nodes per EC2 instance
          # note that the above need to be the ports associated with the *Pekko Management* extension
        }
      }
    }
    ```
    Note: this comes with the limitation that each EC2 instance has to have the same number of Pekko nodes.

* You can change the default tag key from "service" to something else. This can be done via `application.conf`, by
setting `pekko.discovery.aws-api-ec2-tag-based.tag-key` to something else.
    ```
    pekko.discovery.aws-api-ec2-tag-based.tag-key = "pekko-cluster"
    ```

* If your service is running in a docker container, you will need to configure Pekko Management with separate
  IPs for binding and discovery. This is because Pekko Management needs to _bind_ to the internal docker IP,
  but will _discover_ the "host" IP (the EC2 private IP) on the AWS API. See @ref:[Basic
  Configuration](../pekko-management.md) on how to separate the bind IP from the discovery IP.

Demo:

* A working demo app is available in the [integration-test](https://github.com/apache/incubator-pekko-management/tree/master/integration-test/aws-api-ec2)
folder.


### Discovery Method: AWS API - ECS Discovery

If you're using ECS with
[awsvpc](https://aws.amazon.com/blogs/compute/introducing-cloud-native-networking-for-ecs-containers/)
mode networking, you can have all task instances of a given ECS service discover
each other. If you're using this module for bootstrapping your Pekko cluster that
you'll do so by setting the value of
`pekko.management.cluster.bootstrap.contact-point-discovery.service-name` to that of the
ECS service itself.

Screenshot of two ECS task instances (the service name is
`liquidity-application`):

![ECS task instances](../images/discovery-aws-ecs-task-instances.png)


#### Dependencies and usage (ECS Discovery)

There are two "flavours" of the ECS Discovery module.

##### pekko-discovery-aws-api

This uses the mainstream AWS SDK. The advantage here is that if you've already
got the mainstream AWS SDK as a dependency you're not now also bringing in the
preview SDK. The disadvantage is that the mainstream SDK does blocking IO.

@@dependency[sbt,Gradle,Maven] {
  symbol1=PekkoManagementVersion
  value1=$project.version$
  group="org.apache.pekko"
  artifact="pekko-discovery-aws-api_$scala.binary.version$"
  version=PekkoManagementVersion
}

And in your `application.conf`:

```
pekko.discovery {
  method = aws-api-ecs
  aws-api-ecs {
    # Defaults to "default" to match the AWS default cluster name if not overridden
    cluster = "your-ecs-cluster-name"
  }
}
```


##### pekko-discovery-aws-api-async

This uses the preview AWS SDK. The advantage here is that the SDK does
non-blocking IO, which you probably want. You might need to think carefully
before using this though if you've already got the mainstream AWS SDK as a
dependency.

Once the async AWS SDK is out of preview it is likely that the
`pekko-discovery-aws-api` module will be discontinued in favour of
`pekko-discovery-aws-api-async`.

@@dependency[sbt,Gradle,Maven] {
  symbol1=PekkoManagementVersion
  value1=$project.version$
  group="org.apache.pekko"
  artifact="pekko-discovery-aws-api-async_$scala.binary.version$"
  version=PekkoManagementVersion
}

We have 2 approaches in ECS: `aws-api-ecs-async` and `aws-api-ecs-task-set-async`.

###### aws-api-ecs-async

In your `application.conf`:

```
pekko.discovery {
  method = aws-api-ecs-async
  aws-api-ecs-async {
    # Defaults to "default" to match the AWS default cluster name if not overridden
    cluster = "your-ecs-cluster-name"
    # Defaults to an empty list
    tags = [
      { key = "deployment-side", value = "blue" }
    ]
  }
}
```

This will query the AWS API to retrieve all running tasks of the ESC service specified at
`pekko.management.cluster.bootstrap.contact-point-discovery.service-name`.

###### aws-api-ecs-task-set-async

If you use AWS CodeDeploy, you probably want to use this method of discovery.

In your `application.conf`:

```
pekko.discovery {
  method = aws-api-ecs-task-set-async
  aws-api-ecs-task-set-async {
    # Defaults to "default" to match the AWS default cluster name if not overridden
    cluster = "your-ecs-cluster-name"
  }
}
```

The service-discovery works in 3 steps:
1. Query the internal ECS metadata API to retrieve the TaskARN of itself 
(See [AWS docs](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-metadata-endpoint-v3.html))
2. Retrieve the TaskSet to which the TaskARN belongs
3. Retrieve all Tasks belonging to that TaskSet


Notes:

* Since the implementation uses the AWS ECS API, you'll need to make sure that
  AWS credentials are provided. The simplest way to do this is to create an IAM
  role that includes appropriate permissions for AWS ECS API access. Attach
  this IAM role to the task definition of the ECS Service. See the docs for
  [IAM Roles for Tasks](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-iam-roles.html).

* In general, for the ECS task instances to "talk to each other" (necessary for
  forming a cluster), they need to be in the same security group and the proper
  rules have to be set. See the docs for
  [Task Networking with the `awsvpc` Network Mode](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-networking.html).

* pekko-remote by default sets `pekko.remote.netty.tcp.hostname` to the result of
  `InetAddress.getLocalHost.getHostAddress`, and pekko-management does the same
  for `pekko.management.http.hostname`. However,
  `InetAddress.getLocalHost.getHostAddress` throws an exception when running in
  awsvpc mode (because the container name cannot be resolved), so you will need
  to set this explicitly. An alternative host address discovery method is
  provided by both modules. The methods are
  `EcsSimpleServiceDiscovery.getContainerAddress` and
  `AsyncEcsDiscovery.getContainerAddress` respectively, which you
  should use to programmatically set both config hostnames.

* Because ECS service discovery is only able to discover IP addresses (not ports
  too) you'll need to set
  `pekko.management.cluster.bootstrap.contact-point.fallback-port = 7626`, where
  7626 is whatever port you choose to bind pekko-management to.

* You can set additional filters to only discover nodes with specific tag values in 
  your application.conf file, in the `pekko.discovery.aws-api-ecs-async.tags` key. 
  An empty list of tags will not filter any nodes out.

  For example:
    ```
    pekko {
      discovery {
        aws-api-ecs-async {
          tags = [
                { key = "environment", value = "staging" },
                { key = "deployment-side", value = "blue" }
              ]
        }
      }
    }
    ```

* The current implementation only supports discovery of service task instances
  within the same region.

Demo:

* A working demo app is available in the
  [integration-test](https://github.com/apache/incubator-pekko-management/tree/master/integration-test/aws-api-ecs)
  folder. It includes CloudFormation templates with minimal permissions w.r.t to
  IAM policies and security group ingress, and so is a good starting point for
  any deployment that integrates the
  [principle of least privilege](https://en.wikipedia.org/wiki/Principle_of_least_privilege).


