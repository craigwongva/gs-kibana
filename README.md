## Summary

To generate new certificates:
git clone https://github.com/craigwongva/gs-kibana
cd gs-kibana
./certs dev
./certs int
./certs test
./certs stage

## Creating Certificates
The purpose of this step is to set up certificates that will be used by the Kibana/nginx installation.

That is, this step generates certificates appropriate for a domain like gsn-kibana-int.piazzageo.io,
then it stores them in S3 for later use by the Kibana/nginx installer.

You can specify a prefix like 'craig' for testing purposes. The prefix is added to:
* the certificate identity (e.g. craig-gsn-kibana-dev.piazzageo.io),
* the Route53 A record (e.g. craig-gsn-kibana-dev .piazzageo.io.),
* the S3 bucket name (e.g. craig-dsn-kibana)

but the testing prefix is not added to:   
* the /etc/letsencrypt/live folder on the EC2 instance you're using to generate the certificates.

This process is not ready to be run automatically. It is, however, largely automated via a bash script `certs` (that you invoke manually as described below).

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
./certs dev craig
```

### Verifying Certificate Results
You should see an S3 bucket with folders (the following example doesn't mention a `craig` test prefix):
```
 gsn-kibana
  letsencrypt
   live
    gsn-kibana-dev.piazzageo.io
    gsn-kibana-int.piazzageo.io
    gsn-kibana-test.piazzageo.io
    gsn-kibana-stage.piazzageo.io
```

If you create certificates for prod, then you would see a gsp-kibana S3 folder:
```
 gsp-kibana
  letsencrypt
   live
    gsp-kibana-prod.piazzageo.io
 
```

Within each of the lowest level folders (e.g. gsn-kibana-stage.piazzageo) you would see files:
```
cert.pem
chain.pem
fullchain.pem
privkey.pem
```

The timestamps should show that the files are newly created. If they are old, then check /var/log/letsencrypt for a message like "Client lacks sufficient authorization." This message actually seems to indicate a letsencrypt server problem. That is, in June 2017 I ran successfully "./certs dev" and "./certs int" but then "./certs test" and "./certs stage" failed and "./certs prod" succeeded. I then tried "./certs test" and "./certs stage" again and they succeeded.

## Renewing Certificates
Note: This section is redundant if you read and acted upon the above section.

The process of 'renewing' certificates is a little misleading, because renewing actually involves generating new certificates.

Do this manually (this could be scripted but hasn't been yet):
```
sudo su -
rm -rf /etc/letsencrypt
rm -rf /home/ec2-user/.local/share/letsencrypt
unset PYTHON_INSTALL_LAYOUT
exit
```

To verify that everything will succeed, do this manually:
```
certbot-auto certonly --standalone -d gsn-kibana-dev.piazzageo.io --email myemail@radiantblue.com --agree-tos --quiet --debug
```

Then run on the command line where 
   * `dev` is a space like dev, int, test, stage, or prod, and   
   * `craig` is a test prefix (or you can omit this parameter for non-tests)
```
./certs dev
```

You should do runs and re-runs sparingly, because letsencrypt has a certificate generation rate limit of 20
certificates per week.

The `certs` process will upload .pem files up to S3 (either `gsn-kibana` or `gsp-kibana`).

Then use the below instructions to create new Kibana stacks (e.g. gsn-kibana-dev, gsp-kibana-prod) via CloudFormation. The new stack will replace the appropriate Route53 entries. The old stacks (e.g. gsn-qibana-dev, gsp-qibana-dev) can be deleted.
 
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
     * route53:ChangeResourceRecordSets Z3CJCO7XTRTAHX
       * Enables a script to update the Route53 hosting zone 'Z3CJCO7XTRTAHX' to know about a new Kibana instance.
     * s3:GetObject gsn-kibana/\*
       * Enables a script to download the letsencrypt certificates for nginx from S3. (TODO: Allow test prefixes like craig-gsn-kibana.)

5. Browse to Kibana at https://gsn-kibana-\<space\>.piazzageo.io

### Verifying Installation Results

You should see on your EC2 instance:
```
/etc/letsencrypt/live/gsn-kibana-<space>.piazzageo.io
  cert.pem
  chain.pem
  fullchain.pem
  privkey.pem
```

### Script Flow
Here is the calling sequence. For example, the `cf-kibana` script calls the `userdata-kibana` script, which calls the `https` script.
```
cf-kibana
  elasticsearch
  userdata-kibana
    upsert-route53
    https
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

