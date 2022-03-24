Instance assumptions

1. docker
    1. starts on start up
    2. does not require sudo
2. docker-compose
3. golang
4. openssl
5. git
6. make
7. aws-cli
8. CloudWatchAgentServerRole is attached
9. crontab

**How are ami built?**

AMI builder pipeline builds the ami

The pipeline installs required packages and updates ami software

This process generates a new ami we can then use for testing

**How to integration test in your aws account**
1. Create resources and setup local
   1. Install terraform
   2. Set up aws credentials
   3. Set up iam role
      1. Role must include cloud watch server policy and s3 policy
   4. Create s3 bucket
2. Upload binary to s3
   1. make release 
      1. hints
         1. You may want to do this on a linux ec2 instance installing the agent may fail if you build on Mac 
         2. If you want to build faster remove not needed packages and test from release step
   2. aws s3 cp build/bin s3://${your bucket name}/integration-test/binary/${git commit sha} --recursive
3. Start Local Stack
   1. Go to Local Stack directory
      1. cd ../localstack
   2. init terraform
      1. terraform init
   3. Apply terraform 
      1. ```
         terraform apply --auto-approve \
         -var="github_repo=${gh repo you want to use ex https://github.com/aws/amazon-cloudwatch-agent.git}" \
         -var="github_sha=${commit sha you want to use ex fb9229b9eaabb42461a4c049d235567f9c0439f8}" \
         -var='vpc_security_group_ids=["${name of your security group}"]' \
         -var="key_name=${name of key pair your created}" \
         -var="s3_bucket=${name of your s3 bucket created}" \
         -var="iam_instance_profile=${name of your iam role created}" \
         -var="ssh_key=${your key that you downloaded}"
         ```
      2. Write down the dns output that will be important for the next step
   4. Go back to linux directory
      2. cd ../linux
4. Start the test linux test
   1. init terraform
      1. terraform init
   2. Apply terraform
      1. ```
         terraform apply --auto-approve \
         -var="github_repo=${gh repo you want to use ex https://github.com/aws/amazon-cloudwatch-agent.git}" \
         -var="github_sha=${commit sha you want to use ex fb9229b9eaabb42461a4c049d235567f9c0439f8}" \
         -var='vpc_security_group_ids=["${name of your security group}"]' \
         -var="s3_bucket=${name of your s3 bucket created}" \
         -var="iam_instance_profile=${name of your iam role created}" \
         -var="key_name=${name of key pair your created}" \
         -var="ami=${ami for test you want to use ex cloudwatch-agent-integration-test-ubuntu*}" \
         -var="user=${log in for the ec2 instance ex ubuntu}" \
         -var="install_agent=${command to install agent ex dpkg -i -E ./amazon-cloudwatch-agent.deb}" \
         -var="ca_cert_path=${where the default cert on the ec2 instance ex /etc/ssl/certs/ca-certificates.crt}" \
         -var="arc=${what arc to use ex amd64}" \
         -var="binary_name=${binary to install ex amazon-cloudwatch-agent.deb}" \
         -var="local_stack_host_name=${dns value you got from the local stack terraform apply step}" \
         -var="ssh_key=${your key that you downloaded}"
         ```
5. Tear down terraform state
   1. Tear down test state
      1. terraform destroy --auto-approve
   2. Go to local stack directory
      1. cd ../localstack
   3. Tear down localstack state
      1. terraform destroy --auto-approve