CREATING CERTIFICATES
You need to set up certificates before running the kibana installation:
1. ./certs dev craigtest
   where dev is a space like dev, int, test, stage, prod
     and craigtest is a test prefix (or you can omit this parameter for non-tests)

INSTALLING KIBANA/NGINX

This is a semi-automated process. 

There is no need at this time to fully automate.

Sign onto instance having AdministratorAccess.
1. ./cf-kibana <stackname> <space> <guipassword>
2. Browse at https://gsn-kibana-<space>.<well-known domain>

VERIFYING RESULTS
You should see on your EC2 instance:
 /etc/letsencrypt/live/craig-gsn-kibana-stage.piazzageo.io
  cert.pem
  chain.pem
  fullchain.pem
  privkey.pem

You should see an S3 bucket for your space:
 craig-gsn-kibana
  letsencrypt
   live
    gsn-kibana-dev.piazzageo.io
    gsn-kibana-int.piazzageo.io
    gsn-kibana-test.piazzageo.io
    gsn-kibana-stage.piazzageo.io

Sometimes the S3 bucket isn't created. This seems to occur intermittently,
and it seems to be related to the security group not successfully opening 443 to 0.0.0.0/0.

Before re-running the script for a space, do this for the affected space:
sudo su -
rm -rf /etc/letsencrypt/live/craig-gsn-kibana-stage.piazzageo.io
exit

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

