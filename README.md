# rhel10

Experimenting with RHEL 10 for the EX200 certification exam.

`rhel-10.yml` creates an x86_64 EC2 instance with a 50 GiB root volume,
an Elastic IP, and Systems Manager access. It installs Git, the OpenJDK 21
JDK, Python pip, and SSM Agent.

The template automatically selects the latest official RHEL 10 x86_64 HVM
pay-as-you-go AMI in the region you deploy to. Red Hat does not publish RHEL
AMI IDs as public SSM parameters, so a small Lambda-backed custom resource
calls `ec2:DescribeImages` against Red Hat's publisher account
(`309956199498`, or `219670896067` in GovCloud) and returns the newest image
matching `AmiNamePattern`. The AMI's own root device name is used for the
volume mapping. The selected AMI is shown in the `AmiId` and `AmiName`
outputs.

The lookup runs when the stack is created. It does not re-run on later
updates unless you change `AmiLookupVersion`; doing so may select a newer
AMI, which replaces the instance.

SSH access uses an existing EC2 key pair (`KeyName`) and is limited to the
CIDR range in `SshCidr`. `SshCidr` defaults to `0.0.0.0/0`, which allows SSH
from anywhere; set it to your own address (for example `203.0.113.10/32`) to
restrict access. The login user is `ec2-user`.

Example deployment:

```sh
aws cloudformation deploy \
  --template-file rhel-10.yml \
  --stack-name rhel-10-lab \
  --parameter-overrides KeyName=my-key \
  --capabilities CAPABILITY_IAM \
  --region us-east-1
```

The template uses the region's default VPC and default subnet selection.
The instance needs outbound access to RHEL repositories, PyPI, S3,
CloudFormation, and Systems Manager endpoints. The security group permits
inbound HTTP, but the template does not install a web server or configure
an OS firewall for HTTP.

After deployment, use the `SshCommand`, `TerminalURL`, or `SsmCommand` stack
output to connect. Bootstrap logs are in `/var/log/user-data.log` and
`/var/log/cfn-init.log`. Failures before the CloudFormation helper scripts
are installed will expire the 15-minute creation timeout.

Validate locally with `cfn-lint rhel-10.yml`.
