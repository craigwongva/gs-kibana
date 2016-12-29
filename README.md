HOW TO USE THIS REPO

Sign onto AdministratorAccess instance.
#Update NACL and security groups:
./elasticsearch.cli <space>

#Create a Kibana instance (this calls CloudFormation with userdata-kibana and upsert-route53):
./cf-kibana.cli <space>

Browse at http://95.95.95.95

Here is the calling sequence:
1-cf-kibana.cli
1.1-elasticsearch.cli
1.2-userdata-kibana
1.2.1-upsert-route53
