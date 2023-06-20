---
title: Updating logical ID of an AWS CF
tags: [aws]
created: 2022-07-13T00:43:47.773Z
modified: 2022-07-13T01:22:49.580Z
---

Recently I had been refactoring some older code and encountered an error
with AWS CloudFormation failing to update an existing stack.

```
<resource_name> already exists in stack
```

It turned out some CF resources' logical names (AKA logical ID)
had been changed in the new CF template however their custom physical names remained unchanged.

The problem had been [explained at SO]
(https://stackoverflow.com/questions/58268397/how-to-re-deploy-stack-when-getting-resource-already-exists-in-stack-error-wi)
and I will leave it here (slighly updated).

When CloudFormation creates a resource, it associates a logical name
specified in CF template with this resource. For example:


```
...
SomeLogicalName:
  Type: AWS::IAM::ManagedPolicy
...
```

In addition to the logical ID, certain resources also have a physical name (AKA physical ID),
which is the actual assigned name for that resource, such as an EC2 instance ID or an S3 bucket name.

If the physical name is not specified, by default, AWS CloudFormation generates a unique physical ID to name a
resource (with a string of random characters as a suffix). For example, CloudFormation might name an Amazon S3 bucket with
the following physical ID stack123123123123-s3bucket-abcdefghijk1.

If you want to use a custom physical name, specify a name property for that resource
in your CloudFormation template. Each resource that supports custom names
has its own property that you specify. For example, to name an DynamoDB
table, you use the TableName property. For ManagedPolicy, ou use ManagedPolicyName property.

```
...
SomeLogicalName:
  Type: AWS::IAM::ManagedPolicy
  Properties:
    PolicyDocument:
      ManagedPolicyName: SomePhysicalName
...
```

Above we create a resource with a logical name SomeLogicalName.
As long as the logical ID (exampleLogicalId) remains unchanged,
CloudFormation will recognise that the resource has already been created
and will update it as necessary.

However if you change the logical ID (as happened to me when using CDK,
because it auto-generates these logical IDs) then CloudFormation thinks it
needs to create a new resource. But because the name already exists, and
for this resource type the names must be unique, there is a conflict and
the creation fails. This seems to be because CloudFormation creates all new
resources before deleting any removed ones.

The solution is to either

1) Put the logical ID back to what it originally was so CloudFormation
recognises it as an 'update' rather than a 'create' or

2) Change the name (the 'SomePhysicalName' in this example) to be something else.

Changing the name (or whatever the unique field is for the resource in
question) can be a convenient option, because it will create the new
resource and associate it with the new logical ID, then delete the old
resource if that logical ID no longer exists in your CloudFormation
template. You can then rename the resource back to what you wanted
originally (keeping the same logical ID) and deploy a second time, and
CloudFormation will recognise it as an update operation, and rename the
resource back to what you wanted the first time.

Just be aware that doing this will not perform an update, but rather a
delete-then-create. So if your resource has data (e.g. DynamoDB table with
data, IAM policy already attached to roles, Parameter Store entry with a
value entered) you may need to recreate this data for the new resource.
