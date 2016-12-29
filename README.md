HOW TO USE THIS REPO

Sign onto instance having AdministratorAccess.
1. ./cf-kibana.cli <space>
2. Browse at http://gsn-kibana-test.<well-known domain>

Here is the calling sequence:
1-cf-kibana.cli
1.1-elasticsearch.cli
1.2-userdata-kibana
1.2.1-upsert-route53

Here is the flow of parameters:
cf-kibana.cli (input: STACK SPACE GUIPASSWORD) derives: gstype
 calls elasticsearch.cli (input: SPACE) that derives: gstype
 has-hardcoded: instanceType
 looks-up: subnetId securityGroupId
 calls cloudformation (input: instanceType subnetId securityGroupId guiPassword space)
 calls userdata-kibana (input: space guiPassword) that derives: gstype recordsetname
  calls upsert-route53 (input: recordsetname)

There is some duplicated parameter checking code for gstype. It's OK duplicated,
in case the scripts get called individually.
