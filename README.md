CREATING CERTIFICATES
The purpose of this step is to set up certificates that will be used by the kibana/nginx installation.

That is, this step generates certificates appropraite for a domain like gsn-kibana-int.piazzageo.io,
then it stores them in S3 for later use by the Kibana/nginx installer.

You can specify a prefix like 'craigtest' for testing purposes. The prefix is added to:
  the certificate name,
  the Route53 A record,
  the S3 bucket name
but not to the  
  the instance /etc/letsencrypt/live folder.

This step is not ready to be run automatically. It is, however, largely automated via a bash script.

1. Log in to any EC2 instance having the AWS CLI and these IAM privileges:

     ec2:DescribeInstances 
       Enables querying for security group info for this instance.

     ec2:AuthorizeSecurityGroupIngress
       Enables opening of port 443 to letsencrypt challenger daemon.

     s3:CreateBucket 
       Enables creation of a bucket that will hold the generated certificates.

     route53:ChangeResourceRecordsets Z3CJCO7XTRTAHX 
       Enables association of this EC2 instance's IP to the desired domain name, e.g. gsn-kibana-dev. 

     s3:PutObject 
       Write the generated certificates to S3.

2. Run on the command line like this:

     ./certs dev craigtest

       where dev is a space like dev, int, test, stage, prod
         and craigtest is a test prefix (or you can omit this parameter for non-tests)

VERIFYING RESULTS
You should see an S3 bucket for your space:
 gsn-kibana
  letsencrypt
   live
    gsn-kibana-dev.piazzageo.io
    gsn-kibana-int.piazzageo.io
    gsn-kibana-test.piazzageo.io
    gsn-kibana-stage.piazzageo.io

If it is ever necessary to re-run a script for a space or spaces, do this (it seems harsh but it's OK):
  sudo su -
  rm -rf /etc/letsencrypt
  exit

You should do re-runs sparingly, because letsencrypt has a certificate generation rate limit of 20
certificates per week.

TODO: Refine the above /etc/letsencrypt path. Which directories actually need to be deleted to enable
certificate re-generation (as opposed to automatic certificate renewal)?
 

INSTALLING KIBANA/NGINX
The purpose of this step is to install Kibana plus nginx, connecting to a pre-existing Elasticsearch cluster.

It first enables access to a domain and an underlying Kibana instance such as http://gsn-kibana-int.piazzageo.io.

Then it replaces http with https, using the certificates from S3.

This is a semi-automated process.  There is no requirement at this time to fully automate.

1. Log in to any EC2 instance having the AWS CLI and these IAM privileges:

     AdministratorAccess
       Enables the server to run CloudFormation for an instance with role gsn-iam-KibanaRole (or gsp-iam-KibanaRole).
       (TODO: This is likely too strong a policy.)

2. yum install -y git
3. git clone https://github.com/craigwongva/gs-kibana
4. ./cf-kibana <stackname> <space> <guipassword>

   This bash script will determine some variable values and then launch CloudFormation.

   CloudFormation will create a new EC2 instance, and will use userdata to install Kibana/nginx.

   The new EC2 instance will use a role gsn-iam-KibanaRole (or gsp-iam-KibanaRole) with
   these policies:

     cloudformation:DescribeStacks * 
       Enables querying for security group info for this instance.

     route53:ChangeResourceRecordSets Z3CJCO7XTRTAHX
       Once a Kibana instance is created and Kibana/nginx are installed, update Route53 so the instance is findable.

     s3:GetObject gsn-kibana/asterisk 
       Gets the letsencrypt certificates for nginx from S3. Needs to be generalized to allow test prefixes like craig-gsn-kibana.

5. Browse to Kibana at https://gsn-kibana-<space>.piazzageo.io

VERIFYING RESULTS
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

INSTALLING SENSE

The Sense GUI requires you to enter an Elasticsearch server address.

Rather than requiring our team to look up a long name like
internal-gsp-elast-LoadBala-9OBRHB9UYD999-9999999999.us-east-1.elb.amazonaws.com,
add an alias CNAME record:

1. git clone https://github.com/craigwongva/gs-kibana
2. cd gs-kibana/cnames
3. ./upsert-cnames

The above will populate CNAME records for:
* gsn-es-dev
* gsn-es-int
* gsn-es-test
* gsn-es-stage
* gsp-es-prod

In the Sense GUI, enter a URL like 'http://gsn-es-dev.piazzageo.io:9200'.

Reminder: You will have already authenticated into Kibana before you need to
add the URL into the Sense GUI.

