HOW TO USE THIS REPO

Sign onto AdministratorAccess instance.
#Update NACL and security groups:
./elasticsearch.cli space

#Create a Kibana instance:
./cf-kibana space

Sign onto instance.
sudo yum install -y git
git clone https://github.com/craigwongva/gs-kibana
cd gs-kibana
#Install Kibana:
./userdata-kibana space username


NEXT STEPS

Run scripts from CloudFormation.
