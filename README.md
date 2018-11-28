# CloudFormation Bringup

Most of Confirmed's backend can be brought up with CloudFormation. However, because CloudFormation doesn't support specific actions yet, some steps have to be done manually. To see how each piece fits together in the overall infrastructure, see the Architecture diagram in the Shared repository.

# Bringup Instructions

Start with `0-Global.yml` and end in `5-VPN.yml`. Files with the same number prefix can be run concurrently, otherwise, wait for previous number files to complete first. Files ending in `-StackSet` should be brought up as StackSets instead of regular Stacks.

Use either the web CloudFormation console to do the bringup, or use the CLI. The console is more versatile because the CLI has a size limit of ~52KB per YAML file.

## Prerequisites

1. __AWS Simple Email Service__ - Configure AWS Simple Email Service and get out of the sandbox so sending email is allowed. Also, verify the `team@[domain]` and `admin@[domain]` addresses. __[Instructions](https://docs.aws.amazon.com/ses/latest/DeveloperGuide/request-production-access.html)__

2. __AWS Route 53__ - Create a Route 53 Public Hosted Zone for the service's domain (e.g, confirmedvpn.com) and point the nameservers to it. __[Route 53](https://console.aws.amazon.com/route53/home#hosted-zones:)__

3. __Google Play Keys__ - Create OAuth credentials and other keys for the Google Play Store, so the server can validate Android In-App Purchase receipts. __[Instructions](https://gist.github.com/derrek/763ef95c71491833ae3c1e4acad6785e)__
	- The keys are used in `2-Base.yml` as `GoogleRefreshToken`, `GoogleClientId`, `GoogleClientSecret`, `GoogleLicenseKey`, and `GooglePackageName`.

4. __App Store Subscription Secret__ - Generate a secret that has access to subscription information for the app on Apple's App Store. This key is used in `2-Base.yml` as `IosSubscriptionSecret`.

5. __Stripe__ - Create a Stripe account. Get API keys and enable Webhooks to send to `https://webhook.[DOMAIN]/stripe` with the following events enabled:
	- invoice.created
	- invoice.payment_failed
	- invoice.payment_succeeded
	- customer.source_expiring
	- customer.subscription.trial_will_end

6. __Stripe Apple Pay__ - Add the `www.[domain]` and `[domain]` domains to Stripe Apple Pay. __[Instructions](https://dashboard.stripe.com/account/apple_pay)__

7. __Domain Email Access__ - Must have access to the following email addresses: `hostmaster@[domain]`, `team@[domain]`, and `admin@[domain]`. This is for SSL certificate validation, support emails, and admin/alert emails. 

## 0-Global.yml

Run once per AWS account. This configures:

1. CloudTrail
2. CloudWatch alerts for CloudTrail
3. Audit IAM group and user
4. Allows StackSets to be created/managed

Parameter | Description
--- | --- | ---
`AdministratorAccountId` | AWS Account Id of the administrator account (the account in which StackSets will be created).
`AlertEmail` | Email address to send CloudWatch Alerts to.
`TrailName` | Name of the CloudTrail to create. Defaults to `CloudTrail`.

### After Completion
The Audit User's access id and key are visible in "Outputs" of the CloudFormation stack.

## 1-StackSet-Shared.yml

Run this in every region as a StackSet. This configures shared roles, Lambda functions, etc that are used by both the `Base` servers and `VPN` servers.

Parameter | Description
--- | --- | ---
`Environment` | Name of environment to bring up (e.g, PROD).
`GitBranch` | Which Git branch's code to deploy.
`Domain` | Domain to deploy the VPN service to.

## 2-Base-Shared.yml

Run in `Base` region. This bring up the database and other shared basics for `Main` and other servers. Also brings up CodeCommit (git) repositories, creates a S3 Bucket for clients to do speed tests on, and creates a CloudWatch Logs Log Group called `ENVIRONMENT-AdminAudit`, which has streams PostgresQueries and Actions, so they can be audited. It also creates VPC Peering Connections so all regions can communicate with the Base region.

The DependsOn attributes on parameters are purely for API throttling -- e.g, CloudFormation doesn't allow creation of too many Parameter Store parameters simultaneously, so we stagger them.

### After Completion

__Configure Git and Push to CodeCommit__

`git clone` each of `Shared`, `Support`, `Admin`, `Main`, `Webhook`, `Renewer`, and `Helper` to a local machine. Then, run the following:

```
REGION=us-east-1
ENV=PROD
BRANCH=prod
REPOS=(
  Shared
  Support
  Admin
  Main
  Webhook
  Renewer
  Helper
)
	
for REPO in "${REPOS[@]}"
do
  cd $REPO
  git remote add $ENV https://git-codecommit.$REGION.amazonaws.com/v1/repos/$ENV-$REPO
  git remote set-url --add --push $ENV https://git-codecommit.$REGION.amazonaws.com/v1/repos/$ENV-$REPO
  git push $ENV $BRANCH 
  cd ..
done
```

__Speed Test Files__
Upload speed test files to the S3 `SpeedTestBucket`, and be sure to grant public read permission to each file. Files will be accessible by the following format:
`https://<bucketname>.s3-accelerate.amazonaws.com/<filename>`
  
## 3-Base-Admin.yml

Run in `Base` region. This is the Admin server that only a whitelisted IP has access to. It initializes the database schema, has a dashboard for Administrators, amongst other things.

### Initialize Database

Go to `https://admin.[domain]/?initialize=true`, which initializes the database schema and redirects to sign in page.

### Create Admin User

Go to `https://admin.[domain]/signup` to create new admin account. The admin account must have the same domain as the domain of the environment. Confirm the account with email, then sign in.

### Generate and Configure Source

A "Source" is the root and server certificates of a new certificate chain.

Once logged into the Admin Dashboard:

1. Click `Source`.
2. Create a new source (certificate chain) with the current date in MMDDYY format.
3. Click `Set Current Source` on the newly created Source to make it the current Source.
4. Enter a number of Client Certificates to generate, and `Generate` them.
5. Click `Toggle Secret` to allow VPN servers to retrieve the Source.

### Suricata Setup

Suricata is used to detect and stop malicious network activity that might be worms, exploits, or other bad activity. Its rules are set and updated by a tool called `Suricata-Update`, and that tool is configured from Admin.

1. Click `Suricata`.
2. Set rules for each configuration file and click `Save`.

Because Confirmed VPN is openly operated, Suricata rules are in a publicly readable S3 bucket at: `https://s3-<region>.amazonaws.com/<suricata_bucket>/<filename>`.

### Upload Windows/Mac Client Apps

1. Click `Clients`
2. Upload Mac and Windows clients and their update files.
3. Set the percentage distribution of these client files. Percentages must add up to 100%.

## 4-Base-Helper.yml

Run in `Base` region, and name the stacks `Helper-1`, `Helper-2`, etc. This is the Helper server that VPN servers talk to on client connection and disconnection.

On client connection to the VPN, the VPN server's `Bandwidth` script checks with Helper to see if the client should be throttled for having an expired subscription or abusive behavior.

On client disconnection from VPN, the VPN server's RADIUS Accounting configuration sends the client bandwidth usage in bytes to Helper. Excessive bandwidth usage can negatively affect other users.

## 4-Base-Main.yml

Run in `Base` region. This is the Main publicly facing server that hosts the website and API for clients.

## 4-Base-Renewer.yml

Run in `Base` region. This is the Renewer server, which updates user subscriptions every day against Stripe, iTunes, and Google Play to ensure subscriptions are correctly set to active or inactive.

## 4-Base-Support.yml

Run in `Base` region. Similar to the Admin server, the Support server is only accessible via a whitelisted IP. It's for the Support team to troubleshoot and resolve customer issues. All requests for customer data send a notification to the customer to let them know exactly what's being accessed, when, and why it's being accessed.

## 4-Base-Webhook.yml
Run in `Base` region. Webhook for Stripe that mostly sends emails to users when something needs to be corrected, like when their payment method is about to expire.

## 4-StackSet-VPN.yml
Run as a StackSet and apply to all regions. This completes VPC Peering and sets up the Bandwidth CodePipeline in all regions.

### After Completion

__Configure Git and Push to CodeCommit__

The `Bandwidth` repository should be created as a remote in to all regions, because it's deployed on every VPN server. To do that:

```
cd Bandwidth
  
REPO=Bandwidth
ENV=PROD
BRANCH=prod
REGIONS=(
  us-east-1
  us-east-2
  us-west-1
  us-west-2
  ap-south-1
  ap-northeast-2
  ap-southeast-1
  ap-southeast-2
  ap-northeast-1
  ca-central-1
  eu-central-1
  eu-west-1
  eu-west-2
  eu-west-3
  sa-east-1
)
for REGION in "${REGIONS[@]}"
do
  git remote add $ENV https://git-codecommit.$REGION.amazonaws.com/v1/repos/$ENV-$REPO
  git remote set-url --add --push $ENV https://git-codecommit.$REGION.amazonaws.com/v1/repos/$ENV-$REPO
done
git push $ENV $BRANCH
``` 

## 5-VPN.yml

Run multiple times in each region, once for each VPN instance. Stack name is format: `VPN-[SourceId]-[DateOfBringup]-0`

### After Completion
Ensure there is a CNAME record pointing to the VPN server batch that's currently active in every region.

Check CodePipeline and ensure that Bandwidth is pushed to all instances.