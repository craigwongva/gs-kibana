## Creating Certificates
The purpose of this step is to set up certificates that will be used by the Kibana/nginx installation.

That is, this step generates certificates appropriate for a domain like gsn-kibana-int.piazzageo.io,
then it stores them in S3 for later use by the Kibana/nginx installer.

You can specify a prefix like 'craig' for testing purposes. The prefix is added to:
* the certificate identity (e.g. craig-gsn-kibana-dev.piazzageo.io),
* the Route53 A record (e.g. craig-gsn-kibana-dev .piazzageo.io.),
* the S3 bucket name (e.g. craig-dsn-kibana)
but not to the  
* the instance /etc/letsencrypt/live folder.

This process is not ready to be run automatically. It is, however, largely automated via a bash script.

1. Log in to any EC2 instance having the AWS CLI and these IAM privileges:
  * ec2:DescribeInstances 
    * Enables a script to query for security group info for this instance.
  * ec2:AuthorizeSecurityGroupIngress
    * Enables a script to open port 443 to letsencrypt challenger daemon.
  * s3:CreateBucket 
    * Enables a script to create a bucket that will hold the generated certificates.
  * route53:ChangeResourceRecordsets Z3CJCO7XTRTAHX 
    * Enables a script to associate this EC2 instance's IP to the desired domain name, e.g. gsn-kibana-dev. 
  * s3:PutObject 
    * Enables a script to upload the generated certificates into the S3 bucket.

2. Run on the command line where 
   * `dev` is a space like dev, int, test, stage, or prod, and   
   * `craig` is a test prefix (or you can omit this parameter for non-tests)
```
./certs dev craigtest
```

## Verifying Results
You should see an S3 bucket with folders (these don't have a `craig` test prefix):
```
 gsn-kibana
  letsencrypt
   live
    gsn-kibana-dev.piazzageo.io
    gsn-kibana-int.piazzageo.io
    gsn-kibana-test.piazzageo.io
    gsn-kibana-stage.piazzageo.io
```
### Re-running
If it is ever necessary to regenerate certificates, do this first:
```
sudo su -
rm -rf /etc/letsencrypt
exit
```
You should do re-runs sparingly, because letsencrypt has a certificate generation rate limit of 20
certificates per week.

TODO: Refine the above /etc/letsencrypt path. Which directories actually need to be deleted to enable
certificate re-generation (as opposed to automatic certificate renewal)?
 
## Installing Kibana/nginx
The purpose of this step is to install Kibana plus nginx, connecting to a pre-existing Elasticsearch cluster.

It first enables access to a domain and an underlying Kibana instance such as http://gsn-kibana-int.piazzageo.io.

Then it replaces http with https, using the certificates from S3.

This is a semi-automated process.  There is no requirement at this time to fully automate.

1. Log in to any EC2 instance having the AWS CLI and these IAM privileges:
  * AdministratorAccess
    * Enables the server to run CloudFormation for an instance with role gsn-iam-KibanaRole (or gsp-iam-KibanaRole).
(TODO: This is likely too liberal a policy.)
2. `yum install -y git`
3. `git clone https://github.com/craigwongva/gs-kibana`
(TODO: Move this into venicegeo.)
4. `./cf-kibana <stackname> <space> <guipassword>`

   This bash script will determine some variable values and then launch CloudFormation.

   CloudFormation will create a new EC2 instance, and will use userdata to install Kibana/nginx.

   The new EC2 instance will use a role gsn-iam-KibanaRole (or gsp-iam-KibanaRole) with
   these policies:
     * cloudformation:DescribeStacks * 
       * Enables querying for security group info for this instance.
     * route53:ChangeResourceRecordSets Z3CJCO7XTRTAHX
       * Once a Kibana instance is created and Kibana/nginx are installed, update Route53 so that the instance is findable.
     * s3:GetObject gsn-kibana/* 
       * Gets the letsencrypt certificates for nginx from S3. (TODO: Allow test prefixes like craig-gsn-kibana.)

5. Browse to Kibana at https://gsn-kibana-<space>.piazzageo.io

## Verifying Results

You should see on your EC2 instance:
 /etc/letsencrypt/live/gsn-kibana-<space>.piazzageo.io
  cert.pem
  chain.pem
  fullchain.pem
  privkey.pem

PROGRAM FLOW
Here is the calling sequence:
1-cf-kibana
1.1-elasticsearch
1.2-userdata-kibana
1.2.1-upsert-route53
1.2.2-https

Here is the flow of parameters:
cf-kibana.cli (input: STACK SPACE GUIPASSWORD) derives: gstype
 calls elasticsearch.cli (input: SPACE) that derives: gstype
 has-hardcoded: instanceType
 looks-up: subnetId securityGroupId
 calls cloudformation (input: instanceType subnetId securityGroupId guiPassword space)
 calls userdata-kibana (input: space guiPassword) that derives: gstype recordsetname
  calls upsert-route53 (input: recordsetname)
  calls https (input: space) that derives: gstype

There is some duplicated parameter checking code for gstype. It's OK duplicated,
in case the scripts get called individually.

