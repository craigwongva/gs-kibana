## Summary
 
## Installing Kibana/nginx

Then use the below instructions to create new Kibana stacks (e.g. gsn-kibana-dev, gsp-kibana-prod) via CloudFormation. The new stack will replace the appropriate Route53 entries. The old stacks (e.g. gsn-qibana-dev, gsp-qibana-dev) can be deleted.
 
## Installing Kibana/nginx
The purpose of this step is to install Kibana plus nginx, connecting to a pre-existing Elasticsearch cluster.

It enables access to a domain and an underlying Kibana instance such as https://gsn-kibana-int.piazzageo.io.

This is a semi-automated process.  There is no requirement at this time to fully automate. But it is largely automated, with scripts that you invoke manually as described below. 

1. Log in to any EC2 instance having the AWS CLI and these IAM privileges (you can use the same EC2 instance that you used earlier for the certificate generation):
  * AdministratorAccess
    * Enables the server to run CloudFormation for an instance with role gsn-iam-KibanaRole (or gsp-iam-KibanaRole).
(TODO: This is likely too liberal a policy.)
2. `yum install -y git`
3. `git clone https://github.com/craigwongva/gs-kibana`
(TODO: Move this repo from craigwongva into venicegeo.)
4. `./cf-kibana <stackname> <space> <guipassword>`

   * stackname: This is the name of the stack that will be permanent in the AWS Console, e.g. gsn-kibana-int.
   * space: This should be `dev`, `int`, `test`, `stage` or `prod`.
   * guipassword: This is the password that our internal staff use to access Kibana.
   
   This bash script will determine some variable values and then launch CloudFormation.

   CloudFormation will create a new EC2 instance, and will use userdata to install Kibana/nginx.

   The new EC2 instance will use a role gsn-iam-KibanaRole (or gsp-iam-KibanaRole) with
   these policies:
     * cloudformation:DescribeStacks * 
       * Enables a script to query for security group info for this instance.

5. Browse to Kibana at https://gsn-kibana-\<space\>.piazzageo.io

### Verifying Installation Results

You should see on your EC2 instance:
```
No more ssl certs to look at...find another way of verifying install
```

### Script Flow
Here is the calling sequence. For example, the `cf-kibana` script calls the `userdata-kibana` script, which calls the `http` script.
```
cf-kibana
  elasticsearch
  userdata-kibana
    upsert-target-group
    http
```

Here is the flow of parameters:
``` 
cf-kibana.cli (input: STACK SPACE GUIPASSWORD) derives: gstype
  calls elasticsearch.cli (input: SPACE) that derives: gstype
  has-hardcoded: instanceType
  looks-up: subnetId securityGroupId
  calls cloudformation (input: instanceType subnetId securityGroupId guiPassword space)
  calls userdata-kibana (input: space guiPassword) that derives: gstype recordsetname
    calls upsert-route53 (input: recordsetname)
    calls https (input: space) that derives: gstype
```
There is some duplicated parameter checking code for gstype. It's OK duplicated,
in case the scripts get called individually.

### Installing Sense
The Sense GUI requires you to enter an Elasticsearch server address.

Rather than requiring our team to look up a long name like
internal-gsp-elast-LoadBala-9OBRHB9UYD999-9999999999.us-east-1.elb.amazonaws.com,
add an alias CNAME record:

1. `git clone https://github.com/craigwongva/gs-kibana`
2. `cd gs-kibana/cnames`
3. `./upsert-cnames`

The above will populate CNAME records for:
* `gsn-es-dev`
* `gsn-es-int`
* `gsn-es-test`
* `gsn-es-stage`
Reminder: You will have already authenticated into Kibana before you need to
add the URL into the Sense GUI.

### Script Flow
Here is the calling sequence.

upsert-cnames
  get-es-url
  upsert-route53-es-cname

