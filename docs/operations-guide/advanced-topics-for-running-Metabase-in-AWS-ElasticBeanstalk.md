- [Logging](#logging)
  - [Network Access log](#network-access-log)
  - [Application Logs](#application-logs)
  - [Using Papertrail for logging on AWS](#using-papertrail-for-logging-on-aws)
- [Fault-tolerance/High-Availability in Metabase](#fault-tolerancehigh-availability-in-metabase)
  - [Horizontal scaling with Elastic Beanstalk](#horizontal-scaling-with-elastic-beanstalk)
  - [Upgrading Metabase with Rolling Update deployment](#upgrading-metabase-with-rolling-update-deployment)
- [Running Metabase over HTTPS](#running-metabase-over-https)
    - [Upload a Server Certificate](#upload-a-server-certificate)
    - [Setup DNS CNAME (using AWS)](#setup-dns-cname-using-aws)
    - [Modify Metabase to enforce HTTPS](#modify-metabase-to-enforce-https)

# Logging
## Network Access log

If you need a log of all the IP addresses and URLs that were accesed during a specific period, you can make the Load Balancer to send those logs to S3. This is useful for analyzing the traffic to your Metabase instance.

To enable this logging, you have to go to the settings of the Load Balancer and enable "Store Logs" in the Access Log Files section. You will need to choose an S3 bucket to dump all the logs and a prefix that will identify the logs coming from this load balancer.

## Application Logs

If you want to retain the Metabase application logs you can do so by publishing then to an S3 bucket of your choice. Here's how:

- On your Metabase Elastic Beanstalk environment, click on the `Configuration` link in the navigation bar on the left side. You will be taken to a page with a number of boxes containing different configuration options for your environment. Click on the `Edit` button next to `Software`
- Scroll down and then check the box under S3 log storage
![EB Enable Log Rotation](images/EBEnableS3LogRotatoin.png)
- Click `Save` in the bottom right corner.

After you click save your Environment will begin updating with your new change. You will have to wait a minute for this to complete and then you're good to go. Elastic Beanstalk will now periodically publish the application log files to S3 for you and you can download them and analyze them at your leisure.

## Using Papertrail for logging on AWS

This provides a simple way to use the Papertrail logging service for collecting the logs for you Metabase instance in an easy to read location.

- Click on `Configuration` on the left hand sidebar.
- Scroll down to `Software Configuration` under the _Web Tier_ section and click the gear icon to edit those settings.
- Under `Environment Properties` add the following entries:
  - `PAPERTRAIL_HOST` - provided by Papertrail
  - `PAPERTRAIL_PORT` - provided by Papertrail
  - `PAPERTRAIL_HOSTNAME` - the name you want to see showing up in Papertrail for this server
- Scroll to the bottom of the page and click `Apply` in the lower right, then wait for your application to update.

_NOTE: Sometimes these settings will not apply until you restart your application server, which you can do by either choosing `Restart App Server(s)` from the Actions dropdown or by deploying the same version again._

# Fault-tolerance/High-Availability in Metabase

## Horizontal scaling with Elastic Beanstalk
If you followed our guide for deploying Metabase with Elastic Beanstalk, you would have noticed that we used a Load Balancer for sending traffic to only one instance. This same load balancer can be used for sending traffic to multiple intances just by increasing the max instance number from 1 to the number you might need. so you can end up with more complex architecture like this one:
![AWS Horizontal Scaling](images/Metabase-AWS-MI.png)

Remember that horizontal scaling needs a single database to persist all the data, so please make sure to follow our guide for [creating an RDS database and connecting it to your instance first](creating-RDS-database-on-RDS.html)

If you would like a reliable, scalable and fully managed Metabase, please consider our [Metabase Cloud](https://www.metabase.com/start/hosted/).

## Upgrading Metabase with Rolling Update deployment

Thanks to ElasticBeanstalk, when you have multiple instances you can update them one by one without downtime. This is useful if you have users in many timezones at once and you don't have a time period to upgrade the instances without leaving your users without Metabase.

Rolling update deployment means that the Load Balancer will send traffic to a healthy instance while others are being upgraded, and when the first ones finish the update process, then the Load Balancer will switch the traffic to the upgraded instances while it upgrades the old ones.

To enable this feature, you need to go to Rolling Updates and deployments section and click on the `Edit` button.

![EB Edit button in Software Configuration](images/EBRollingUpdates.png)

Once inside, you need to change the Deployment Policy to `Rolling` and the batch size to a fixed number (or percentage, depending on your needs). This number or percentage is the amount of instances that will be upgraded while leaving others receiving traffic. Click on `Save` at the end of the page once you are done with the configs.

![EB Edit button in Software Configuration](images/EBRollingUpdatesConfig.png)

# Running Metabase over HTTPS

### Upload a Server Certificate

This is only relevant if you plan to use HTTPS (recommended) for your Metabase instance on AWS. There is no requirement to do this, but we are sticklers for security and believe you should always be careful with your data.

First, you need to open a new tab in your browser and open AWS certificate manager. Once inside, you have the options for provisioning certificates or become a private certificate authority. We will choose `Provision certificates` and we will click on `Get Started`.

![EB Edit button in Software Configuration](images/CertManagerHome.png)

A blue button will appear on the top of the page with the feature to import certificates (you can also ask AWS for a new certificate if needed and they will provision one for you)
![EB Edit button in Software Configuration](images/CertManagerImport.png)

On the following screens, you will have to input all the details and data about the certificate you want to import. After having followed the steps, you will now see the certificate in other tools of AWS (like HTTPS on your load balancer)

### Setup DNS CNAME (using AWS)

- Open up AWS **Route 53** by navigating to **Services > Networking > Route 53** in the AWS Console header.
- Click on **Hosted Zones** then click on the domain name you want to use for Metabase.
- Click on the blue button **Create Record** (a new panel will open up).
  - Enter in a **Record name**: for your application. This should be the exact URL you plan to access Metabase with (e.g. `metabase.mycompany.com`).
  - Under the dropdown for **Record type**: select _A – Routes traffic to an IPv4 address and some AWS resources_.
  - Enable the **Alias** switch and on the box that is titled **Route traffic to:** select __Alias to Application and Classic Load Balancer__, region __US East (Ohio) [us-east-2]__ or the one that you deployed your instance to (e.g. `mycompany-metabase.elasticbeanstalk.com`).
  - Choose the load balancer that corresponds to your instance (if this is a new AWS account there should be only one)
  - Leave all other settings in their default values and click the **Create Record** button at the bottom of the page.
  - _NOTE: After the record is created you must wait for your change to propagate on the internet. This can take 5-10 minutes, sometimes longer._

### Modify Metabase to enforce HTTPS

Before trying to enable HTTPS support you must upload a server certificate to your AWS account. Instructions above.

- Go to Elastic Beanstalk and select your `Metabase` application.
- Click on Environment that you would like to update.
- Click on `Configuration` on the left hand sidebar.
- Scroll down to `Load Balancer` and click the Edit button on the right of the screen.
- On Listeners section, click on "Add Listener" and change the Protocol to HTTPS on the modal window that opens.
- Set the value for `Port` to _443_.
- Then, a little bit lower on the dropdown for `SSL certificate ID`, choose the name of the certificate that you uploaded to your account.
  - _NOTE: The certificate MUST match the domain you plan to use for your Metabase install._
- In SSL Policy select `ELBSecurityPolicy-TLS-1-2-2017-01`
- Scroll to the bottom of the page and click `Save` in the lower right.
  - _NOTE: Your Environment will begin updating with your new change. You will have to wait for this to complete before making additional updates._
  - _IMPORTANT: Once this change is made you will no longer be able to access your Metabase instance at the `*.elasticbeanstalk.com` URL provided by Amazon because it will result in a certificate mismatch. To continue accessing your secure Metabase instance you must [Setup a DNS CNAME](#setup-dns-cname)._

Once your application is working properly over HTTPS we recommend setting an additional property to force non-HTTPS clients to use the HTTPS endpoint.

- Click on `Configuration` on the left hand sidebar.
- Scroll down to `Software Configuration` under the _Web Tier_ section and click the gear icon to edit those settings.
- Under `Environment Properties` add an entry for `NGINX_FORCE_SSL` with a value of `1`.
- Scroll to the bottom of the page and click `Apply` in the lower right, then wait for your application to update.