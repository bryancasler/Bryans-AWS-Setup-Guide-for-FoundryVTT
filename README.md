<!-- + Bryan's AWS Setup Guide for FoundryVTT + -->

<img src="https://raw.githubusercontent.com/bryancasler/Bryans-AWS-Setup-Guide-for-FoundryVTT/master/assets/Bryan's%20AWS%20Setup%20Guide%20for%20Foundry%20VTT%20-%20Full%20-%20Social%20-%20On%20Dark.png" width="0" height="0">

![Bryan's AWS Set Up Guide for FoundryVTT](https://raw.githubusercontent.com/bryancasler/Bryans-AWS-Setup-Guide-for-FoundryVTT/master/assets/Bryan's%20AWS%20Setup%20Guide%20for%20Foundry%20VTT%20-%20Title%20-%20On%20White.png)

My personal setup guide for getting FoundryVTT running on a free instance of AWS.

## Overview
This step-by-step guide provides instructions for creating free AWS EC2 and AWS S3 instances for running FoundryVTT in the cloud. The intruction also include details for installing FoundryVTT on the Unbuntu server running on your AWS EC2 instance.  All instructions are for Mac users. The end result will be FoundryVTT running for free in the cloud using a custom domain (e.g. https://mygame.mydomain.com) you own.

## Introduction
While the official KB provides some useful information for setting up Foundry VTT on AWS, a complete guide to doing so is outside its scope. However, a guide to doing so that also outlines some best practices for the use of Amazon Web Services seems like it would be handy to help others along and make their setup process a little smoother. This guide is intended as an outline of the basic infrastructure needed and how to set it up on AWS. It is not intended as a guide for setting up and configuring the server itself. The excellent [Ubuntu setup guide](https://github.com/foundry-vtt-community/wiki/wiki/Ubuntu-VM) more than adequately documents how to set up the actual webserver and Foundry server. I’ll relink it again at the appropriate point in this guide, but you might want to keep it open in a separate tab. Also, while this guide is intended to be approachable by people with little to no experience with the AWS platform, it is not intended as a general introduction to it.

I personally selected AWS in particular because it offers a very robust cloud hosting platform with a deep list of options and services at a competitive price. Additionally, for those who haven’t used AWS before, they offer new accounts [a number of services free for the first year](https://aws.amazon.com/free/?all-free-tier.sort-by=item.additionalFields.SortRank&all-free-tier.sort-order=asc), which covers most of the expense of self-hosting Foundry in the cloud, and the fees for hosting are nominal thereafter. AWS is by default pay as you go for all services, and only a certain level of usage is covered under the free tier. If you are concerned about the potential charges involved, I highly suggest setting up a [billing alert](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/monitor_estimated_charges_with_cloudwatch.html) using AWS Cloudwatch after setting everything up. Amazon will automatically notify you when it projects you’ll go over your limits. Do note that as the author of this guide, I do not take responsibility for any overage charges accidentally incurred. I will do my best to keep everything within the free tier, however.

Requirements for this guide are a web browser, a notepad app, and possibly some patience. The Ubuntu server setup guide requires an SSH client and some familiarity with the Linux command line.

## Create an AWS Account
If you already have an AWS account, please feel free to skip this section. Everyone else, the first thing you’ll need is an AWS account – go to [aws.amazon.com](https://aws.amazon.com/) and click the Create an AWS Account button in the top right. Fill out the forms as listed, setting up a Personal account. Do note that when it asks you for a payment method, you must provide a valid payment method. Even on the free tier, they will test the card with a nominal $1 charge to test to see if it’s valid. This charge will be credited back to your account after 3-5 days. When they ask you to select a support plan, pick Basic, and you’re done. After you receive confirmation via email that the account is set up, you can log into the console and proceed.\

## Set up AWS Identity and Access Management (IAM)
Now that we have created an AWS account, we’re going to create a couple of sub-accounts for the actual setup and management. When you set up your AWS account, you created what they call a “root account”, which has the keys to the entire castle. Ideally, you’d use a less privileged account for the actual day to day tasks.

We’re going to set up two accounts:

An admin account to set up and manage the services we’ll be using.
A system account to let Foundry VTT access an S3 bucket for static hosting and object storage.
Locate the Services menu at the top of the AWS console.
<img src="https://raw.githubusercontent.com/foundry-vtt-community/wiki/master/images/Getting%20Started/AWS%20Self%20Hosting/2-1.PNG">

Then on the menu that drops down, select IAM
<img src="https://raw.githubusercontent.com/foundry-vtt-community/wiki/master/images/Getting%20Started/AWS%20Self%20Hosting/2-11.PNG">

You should see the main page for AWS Identity and Access Management (IAM) come up.
<img src="https://raw.githubusercontent.com/foundry-vtt-community/wiki/master/images/Getting%20Started/AWS%20Self%20Hosting/2-2.PNG">

Right now we’re most concerned about the Users link on the left. Click it and an empty user list will come up. First, we’re going to set up the admin user. Click Add User at the top, and fill out the form that comes up.
<img src="https://raw.githubusercontent.com/foundry-vtt-community/wiki/master/images/Getting%20Started/AWS%20Self%20Hosting/2-3.PNG">

You can set the user name to whatever you’d like – I put admin here for simplicity’s sake. This user doesn’t need a programmatic access key, so select AWS Management Console access only, and set them up with a custom password. Uncheck Require password reset as you don’t need to reset a password only you should know. Click Next to get the set permissions dialog.
<img src="https://raw.githubusercontent.com/foundry-vtt-community/wiki/master/images/Getting%20Started/AWS%20Self%20Hosting/2-4.PNG">

I’ve selected Attach existing policies directly and selected AdministratorAccess. This will allow your admin account to manage all AWS services. In a real world environment, we would want to be more picky on permissions, but this will do for this guide. Hit Next to go to the Tags page, then Next to review the account, then Create User. Hit Close in the bottom right after you’re done.
<img src="https://raw.githubusercontent.com/foundry-vtt-community/wiki/master/images/Getting%20Started/AWS%20Self%20Hosting/2-5.PNG">

Now that you have an admin account set up, let’s create a user that Foundry will use to access assets on S3. Follow the same Add User process again, but this time give the user a name like “foundry-s3-user”. Instead of giving them Console access, give them Programmatic Access. Don’t attach any policies, and click through to the final screen. On step 5, you’ll have the option to download a CSV file – this contains the credentials that Foundry will use. Download it, but keep it somewhere safe.
<img src="https://raw.githubusercontent.com/foundry-vtt-community/wiki/master/images/Getting%20Started/AWS%20Self%20Hosting/2-6.PNG">


Our users are now set up. We’ll revisit the foundry-s3-user in a later step to grant them actual access to S3, but we need to set up the S3 bucket before we can do that. Before we leave the first part of IAM setup, there’s a couple more things we need to do. Click the Dashboard link on the left to go back to the main dashboard.
<img src="https://raw.githubusercontent.com/foundry-vtt-community/wiki/master/images/Getting%20Started/AWS%20Self%20Hosting/2-7.PNG">

At the top, I’d recommend clicking Customize by the IAM users sign-in link and giving the account an easy to remember name. This will make it easier to log in as the admin user. The name has to be globally unique, but once you’ve set a friendlier name, you’ll get a link you can copy and paste into a bookmark. Additionally, the checklist under Security Status is AWS’ recommended ways to secure your account. At the very least, I suggest activating multi-factor authentication (MFA) on the root account. Google Authenticator and Authy are both good MFA apps available for all major smartphones. Once you log in with the admin account, you can come back to IAM to set up MFA for that user as well, and I recommend doing so.

Now that you have a customized signin link and a separate admin user, open up a new tab, paste in your signin link, and log in under the admin account.

## Create an AWS Elastic Cloud Compute (EC2) Instance
On the same services menu you used to get to IAM, you’ll note that EC2 is in the top left corner. This is because it is far and away Amazon Web Services’ most heavily used service. It provides access to the setup and maintenance interface for the virtual private servers that put AWS on the map. This setup guide will be using screenshots from the New EC2 Experience control panel. If you toggle it off, things won’t look exactly the same, but the same options should still be roughly available. However, as this is probably the most complex part of the infrastructure setup, I advise toggling on the New EC2 Experience for clarity’s sake.

Before we get started, I need to introduce a basic concept: AWS regions. All AWS resources launch in a given region, usually indicated by a locale code, then what side of the locale it’s in, and then the number of the region on that side of the country. For instance, AWS’ Oregon data centers fall under US-West-2, while their Dublin data centers are EU-West-1. You can see what region you’re in at the top right of any AWS dashboard. I suggest using one that’s close to all of your players, to minimize latency.

On the left hand toolbar, click on Instances. You should see the following screen.
<img src="https://raw.githubusercontent.com/foundry-vtt-community/wiki/master/images/Getting%20Started/AWS%20Self%20Hosting/3-1%20.PNG">


Click on either of the Launch Instances button. That’ll take you to the page for selecting the AMI you want to launch. AMIs are base templates for instances – they provide the OS install and a preselected software package. In this case, the Ubuntu 18.04 image will just come with Ubuntu and an assortment of must-have utilities.
<img src="https://raw.githubusercontent.com/foundry-vtt-community/wiki/master/images/Getting%20Started/AWS%20Self%20Hosting/3-2.PNG">

For this guide, we’ll be using the Ubuntu Server 18.04 LTS (HVM), SSD Volume Type AMI. Leave it on the default of 64-bit (x86), and hit Select. The next screen will let you choose the type of instance you want to use. It should default to t2.micro, which is reasonable for a Foundry server. Also, as the little label by the type will mention, it’s eligible for free tier, which no other instance type is. Hit Next: Configure Instance Details.
<img src="https://raw.githubusercontent.com/foundry-vtt-community/wiki/master/images/Getting%20Started/AWS%20Self%20Hosting/3-3.PNG">

I have not included the full list as you can ignore most of them. Just make sure Number of instances is set to 1, and Auto-Assign Public IP is set to Use Subnet Setting (Enable) or Enable. Your instance will need a public IP to be accessible from the wider internet. The rest is a fairly reasonable set of defaults. Click Next: Add Storage.
<img src="https://raw.githubusercontent.com/foundry-vtt-community/wiki/master/images/Getting%20Started/AWS%20Self%20Hosting/3-4.PNG">

This set of defaults is perfectly reasonable, as we will not be storing the game assets on the instance itself. Ubuntu 18.04 Server and Foundry, along with all the required dependencies, take up about 2.2gb installed. Storage-heavy data like images will get pushed to S3, which is generally cheaper for storage and serving them up after the free tier benefits end. If you’d prefer on-instance asset storage, you can provision this with as much space as you think you’ll need and use the standard local storage options in Foundry. However, do note that the charge for EBS space is higher, per gigabyte-month, than S3.

Once you’ve settled on how to handle your storage, click Next: Add Tags. You can add any tags you might want in key-value pairs here, but none are required. Tags are generally handy if you’re running a number of different servers tasked with different things, so you might want to at least set a Name tag for your instance, but again, it’s not required. Click Next: Configure Security Group.
<img src="https://raw.githubusercontent.com/foundry-vtt-community/wiki/master/images/Getting%20Started/AWS%20Self%20Hosting/3-5.PNG">

This is where you’re going to set up the traffic allowed through to your instance. By default, your instance will permit SSH traffic for management from anywhere. You can make this more secure by clicking on the dropdown that says Anywhere, and changing it to My IP. AWS will automatically pull the public source IP for your traffic and restrict SSH traffic to that IP only. Please note that most residential ISPs do dynamic public IP addressing, so you are not guaranteed to have the same address day to day – if SSH login works one day, but not the next, you’ll need to edit the security group later.

If you intend on using nginx as a reverse proxy, like the Ubuntu VM guide lays out, you’ll want to hit Add Rule twice and add rules to let traffic in on TCP ports 80 and 443. If you are just going to run Foundry on the instance by itself, you’ll want to let in TCP port 30000 (or whatever port you intend to put Foundry on). You may also need to allow additional ports through for voice and video traffic – I’d advise referring to the guides for those to get the appropriate port ranges.

You can edit security groups at any time via the EC2 control panel, and changes are instant, so don’t feel you have to get things perfect right now. SSH, HTTP, and HTTPS are the minimums required to setup Foundry and verify it’s working. Click Review and Launch when you’re ready to move on. This will bring you to the last page, which lets you review your instance setup. There is one final task to do before the instance is launched. Click Launch, and this window should pop up.
<img src="https://raw.githubusercontent.com/foundry-vtt-community/wiki/master/images/Getting%20Started/AWS%20Self%20Hosting/3-6.PNG">

By default, EC2 uses key pairs to regulate access. Select Create a new key pair, give it a name, and hit download. Save it somewhere safe where you will remember it. I cannot emphasize this strongly enough: if you lose this file, you will not be able to log into your instance. The instance will need to be recreated – no one, not even AWS Support, can recover a lost private key.

Hit Launch, wait while the spinner spins, then hit View Instances once you get the Launch Status page. A new instance can take a few minutes to go from zero to hero, so go get a glass of water and come back. Refresh the page, and it should show as running and passing the status checks. Highlight it if it isn’t already, and get the IPv4 Public IP out of the information at the bottom. Copy-paste that to a notepad, we’ll need that later.

## Create an AWS Simple Storage Service (S3) Instance
If EC2 is what runs everything on AWS, S3 is what stores it (mostly). The service mostly lives up to its name, as it provides an easy way to store large amounts of objects. These objects can be, so far as we’re concerned for this, of basically unlimited size – sending more than 5gb in a single file requires special transfer methods, but other than that, as long as you’re willing to pay per gigabyte-month for it to get stored, they’ll host it. Open the Services menu and click on S3 near the top left.
<img src="https://raw.githubusercontent.com/foundry-vtt-community/wiki/master/images/Getting%20Started/AWS%20Self%20Hosting/4-1.PNG">


This is the basic S3 console. Hit Create Bucket.
<img src="https://raw.githubusercontent.com/foundry-vtt-community/wiki/master/images/Getting%20Started/AWS%20Self%20Hosting/4-2.PNG">

The bucket name needs to follow a certain set of conventions. First, it can only contain lowercase letters, numbers, and dashes. Second it must be globally unique across all AWS regions and accounts – for example, if I create a bucket called foundry-vtt-assets, you cannot create a bucket with the same name, even though we’re using separate accounts. Leave the region on the default. Under Bucket settings for Block Public Access, uncheck Block all public access and acknowledge the risk.

Normally, I would not use this setting, but Foundry requires that the bucket be made public as it links players directly to assets in the bucket. If the bucket isn’t public, the links won’t work. Hit Create Bucket. Using the public permission for your S3 bucket is a security risk!

When you go back to the dashboard, you should see the new bucket there. Click on the little circle to the left of the name and click Copy ARN above it. Paste that into a notepad – we’ll need it in the next section. Click on the bucket name after you’ve got the ARN.
<img src="https://raw.githubusercontent.com/foundry-vtt-community/wiki/master/images/Getting%20Started/AWS%20Self%20Hosting/4-3.PNG">

This is the bucket itself. Click on the Permissions Tab, then click on CORS configuration. The Foundry defaults for bucket setup can be found [here](https://foundryvtt.com/article/aws-s3/). I suggest just copy-pasting the CORS configuration on that page in and hitting save.

Additionally, while we have made the bucket public, objects in it are not public by default. Both the bucket and the objects in it need to be public in order for Foundry to use S3 for static content. Click on Bucket Policy by the CORS configuration, and paste in the following bucket policy. Insert your bucket name in the Resource line.

```
{     
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Action": "s3:GetObject",
            "Effect": "Allow",
            "Resource": "arn:aws:s3:::<your bucket>/*",
            "Principal": "*"
        }
    ]
}
```
AWS will warn you that the bucket has public access. Again, this is generally not best practices, but it is required for Foundry to use S3. Please do not put anything in this bucket that you are not comfortable sharing with the world.

That’s it for bucket setup. Now all we have to do is set up our system account to have permissions to access only this bucket.

## Connect AWS Identity and Access Management (IAM) Policy with AWS Simple Storage Service (S3) Instance
We need to set up a custom policy that locks down the user we set up for Foundry to access S3 and allows them to only access Foundry-related assets. If this is the first time you’ve used AWS, it may not seem like it’s necessary at this point, but if you start using it for more and more tasks, you’ll come to appreciate restricting accounts access to anything they don’t need. First we need to set up a custom permissions policy. Click Policies on the left, then click Create Policy. This will bring up the Policy Editor.
<img src="https://raw.githubusercontent.com/foundry-vtt-community/wiki/master/images/Getting%20Started/AWS%20Self%20Hosting/5-1.PNG">


Click the JSON tab, delete the text already in the field, and paste in the following policy. Under Resource in the part of the statement with SID “VisualEditor0”, put in your S3 bucket ARN that you copied down in the prior section.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:ListBucket",
                "s3:DeleteObject",
                "s3:PutObjectAcl"
            ],
            "Resource": [
                "<Insert bucket ARN>/*",
                "<Insert bucket ARN>"
            ]
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": "s3:ListAllMyBuckets",
            "Resource": "*"
        }
    ]
}
```
Hit Review and give it a name that’s easy to find – like foundry-s3-access-policy. Next, click Users on the left hand side, then click on the name of your Foundry S3 user. You’ll see a large button asking you to add permissions to the account, as it does not currently have any. This will bring up the Grant permissions dialog.
<img src="https://raw.githubusercontent.com/foundry-vtt-community/wiki/master/images/Getting%20Started/AWS%20Self%20Hosting/5-2.PNG">

Click the Attach existing policies directly button. This will bring up a long list of Amazon-made permissions policies, but you can use the search bar at the top to bring up the policy we just made by searching for its name. Click the checkbox by it, and hit Next: Review, then Add Permissions.

## Install FoundryVTT on your AWS Elastic Cloud Compute (EC2) Instance
You’re now ready to set up your new AWS-hosted Ubuntu server with Foundry. Again, you can find the generic guide [here](https://github.com/foundry-vtt-community/wiki/wiki/Ubuntu-VM).


### Update permisions on your ".pem" file
Go to your ".pem" file and right click on it and select "Get Info". In the pop-up, in the bottom right corner, click the padlock. Then under the "Sharing & Permissions" header select any row that is not your user account (e.g. has "me" in it) or "everyone" and delete them by clicking the "minus". Then ensure "everyone" has a privlege of "No Access". Setting these permission on this file will meet the prerequesited required for the next step.

### Log into your EC2 instance with your ".pem" file
Go back to the EC2 Dashboard and in the sidebar click "Instances". Check the checkbox next to the instance you previously created and click "Connect". This will give you a pop-up with a "Connection method" already selected "A standalone SSH client". On Mac's, Terminal is your SSH client. Simplely hit CMD+Spacebar to bring up Mac's Spotlight search and type in "Terminal". Run the "Terminal" application.

With terminal running, navigate to the directly with ".pem" key file you created earlier. If you've never used Terminal before, you can type "ls" without the quotes and hit enter. This will show all the folders and files in the directory you're currently located. To dive deeper into a directory type "cd {directory name}" for example I might type "cd desktop" to navigate from my user folder into its desktop folder. If you need to go up a directory you can type "cd.." and then would move me from my "desktop" folder back up to its parent "user" folder.

I have put my ".pem" file in a foulder in my desktop titled "Foundry" to navigate from a fresh terminal instance I type "cd desktop" and hit enter. Then "cd foundry" and hit enter. While in the "Foundry" folder if I type "ls" and hit enter it will list all the files in the folder. If I see my ".pem" file I am in the right place. Now I can copy/paste from the "Example" shown on the S3 popup. Copy and past it into your terminal and hit enter.

e.g. `ssh -i "foundry-key-pair.pem" ubuntu@ec2-3-93-212-97.compute-1.amazonaws.com`

This will log you into your EC2 instance using your ".pem" file which acts like a password. The first time you log in it may display something similar to the following. If so, type "yes" and hit enter.

```
ECDSA key fingerprint is SHA256:Vw6TSx3EmsjzOWJaKoYsC9vIxxPDDQG1j7INx1zIIds.
Are you sure you want to continue connecting (yes/no/[fingerprint])? 
```

### Install Node and NPM on your EC2 instance
You are now logged into your Ubuntu server running on AWS EC2. You will now want to install Node and NPM (Node Package Manager) which are required for foundry to run. To do just that, copy and paste the following into your terminal and hit enter. When you do this, you may get a warning that Node is out of date. Don't worry, we'll update it next. Also this process of downloading and installing may take a few miuntes. Wait until the text stops scrolling and you see your cursor after the "$" sign.

```
curl -sL https://deb.nodesource.com/setup_14.x | sudo -E bash -
sudo apt-get install -y nodejs
```
### Install a Reverse Proxy and the Unzip Utility on your EC2 instance
Copy and paste the following and hit enter. Confirm "Do you want to continue?" by pressing "y"

`sudo apt-get install nginx unzip`

### Install the Process Manager pm2 on your EC2 instance
Copy and paste the following and hit enter. 

`sudo npm install pm2 -g`

### Create a directory for Foundry to be installed in
Copy and paste the following and hit enter. 

```
# switch to the home directory
mkdir ~/foundry
cd ~/foundry
```

### Purchase FoundryVTT
Now is the time to buy foundry if you haven't already. You are going to need a license key in the next step. There are two ways you can buy FoundryVTT. The first is through the Patreon and the second is through the website. Each requires different steps to get it downloaded and installed on your EC2 instance.

@TODO: The following two sections need to be cleaned up. What I ended up doing was downloaded the Linux version from FoundryVTT and uploaded it to our S3 bucket. Then used wget like recommended in the Patreon instance to wget the file on our S3 instance.

### Download FoundryVTT and Unpack It
#### If purchased through FoundryVTT's Patreon
Go to your patreon page and grab the link for the Linux version of foundry, it should look like this:

`https://foundryvtt.s3-us-west-2.amazonaws.com/releases/[AccessKey]/FoundryVirtualTabletop-linux-x64.zip`

Enter the following into the terminal to download Foundry from Patreon:

`wget https://foundryvtt.s3-us-west-2.amazonaws.com/releases/[AccessKey]/FoundryVirtualTabletop-linux-x64.zip`

#### If purchased through FoundryVTT.com
Download the node.js zip file from Foundry's website by going logging into [https://foundryvtt.com](FoundryVTT.com) your profile and clicking on Purchased Licenses. Download the version next to "Linux".

From here, you'll need to use a secure file transfer protocal (SFTP) utility to upload the file to your EC2 virtual machine instance. Examples of this are FileZilla, Cyberduck, or Transmit.

You'll use the same credentials you used to log in via (Terminal) SSH. Provide your SFTP client of choice with the server address, username, and password or SSH private key, and you should be able to log in and navigate to the Foundry directory you just set up. Drag and drop the .zip file to there.

### Install the Unzip module
Copy and paste the following and hit enter.
`sudo apt install unzip`

### Unzip the FoundryVTT file you just uploaded and then delete the ZIP
Replace the file names below with your file names (e.g. "foundryvtt-0.7.0")  and then copy and paste the following and hit enter. 

```
unzip foundryvtt-0.6.6.zip
rm foundryvtt-0.6.6.zip
```

### Start up FoundryVTT
Check if Foundry starts up without any errors by running it from the command line

`node /home/ubuntu/foundry/resources/app/main.js --port=8080`

If you get the error `Error: The fallback data path /home/ubuntu/.local/share/FoundryVTT does not exist.` then do the following. Type "cd" and hit enter. This will put you in the parent directory. Now type `mkdir -p .local/share/FoundryVTT` and hit enter. Now type `foundry` and hit enter and then `node /home/ubuntu/foundry/resources/app/main.js --port=8080`.

### Your instance is running!
If you see something similar to the following, your FoundryVTT instance is running!

```
FoundryVTT | 2020-08-31 02:46:52 | [warn] Software license requires signature.
FoundryVTT | 2020-08-31 02:46:52 | [info] Requesting UPnP port forwarding to destination 8080
FoundryVTT | 2020-08-31 02:46:52 | [info] Server started and listening on port 8080
```

Type **Control+C** to terminate this process so we can get back to work and finalize our setup

### Set up Process Manager pm2 to startup FoundryVTT automatically on server boot
First, we need to configure the process manager to startup automatically on server boot. Go back into terminal and type in `pm2 startup` and hit enter. You should see something like

```
[PM2] Init System found: systemd
[PM2] To setup the Startup Script, copy/paste the following command:
sudo env PATH=$PATH:/usr/local/bin /usr/local/lib/node_modules/pm2/bin/pm2 startup systemd -u ubuntu --hp /home/ubuntu
```

Copy and paste that last line listed back into the terminal and hit enter. You should see the following

```
[PM2] [v] Command successfully executed.
+---------------------------------------+
[PM2] Freeze a process list on reboot via:
$ pm2 save

[PM2] Remove init script via:
$ pm2 unstartup systemd
```

### Start FoundryVTT using pm2
Copy and paste the following, and hit enter.

`pm2 start "node /home/ubuntu/foundry/resources/app/main.js" --name "foundry" -- --port=8080`

You should see something similar to the following.

```
[PM2] Starting /home/ubuntu/foundry/resources/app/main.js in fork_mode (1 instance)
[PM2] Done.
┌──────────┬────┬─────────┬──────┬───────┬────────┬─────────┬────────┬─────┬───────────┬────────┬──────────┐
│ App name │ id │ version │ mode │ pid   │ status │ restart │ uptime │ cpu │ mem       │ user   │ watching │
├──────────┼────┼─────────┼──────┼───────┼────────┼─────────┼────────┼─────┼───────────┼────────┼──────────┤
│ foundry  │ 0  │ 0.3.2   │ fork │ 31130 │ online │ 0       │ 0s     │ 0%  │ 8.9 MB    │ ubuntu │ disabled │
└──────────┴────┴─────────┴──────┴───────┴────────┴─────────┴────────┴─────┴───────────┴────────┴──────────┘
 ```
 
`pm2 start foundry`, `pm2 stop foundry`, `pm2 restart foundry` can control the running status of Foundry and `pm2 delete foundry` will remove this process from the pm2 config.

### Save pm2 settings
Once everything is settled and running, you can type `pm2 save` and hit enter. This will save the current settings so they persist after the next server reboot.
 
 
### Setting Up the Reverse Proxy
nginx likes log messages an so do you (even if you don't know it yet). Let's create a directory to store those messages in a structured manner: `sudo mkdir /var/log/nginx/foundry`.

The configuration files for nginx are stored at /etc/nginx and split up in `/etc/nginx/sites-available` for configuration thats are already setup, and `/etc/nginx/sites-enabled` with links to configurations that should be enabled. So let's create a reverse proxy configuration for our Foundry instance running on `127.0.0.1:8080`. The `127.0.0.1` is an IP alias (to simplyfy it) for your own host, so whenever you see that or `localhost`, it means: "This is running locally, on my host, don't search the internet for it, it's right here."

We need to allow larger upload sizes (scene backgrounds are sometimes rather large). 10 MB should be quite okay, but feel free to adjust this value.

Copy and paste the following and hit enter

`sudo nano /etc/nginx/nginx.conf`

The result will be the content of nginx.conf which you can edit and append the following line into the section `http { ... }` right before the closing bracket, this is rather important (recording: https://d.pr/v/M30ufg).

Copy the code between the brackets below and scroll down the corresponding section in Terminal. Then paste it in and hit "Control + X" to exit. When existing it will ask if you want to save your changes so hit "y". Then it will ask for a directory and name, prepopulated with its current directly and name, simply hit "enter" to confirm this and it will save over the existing file.

```
http {
	# Other settings, puth this right at the end of the http-section (this is important!)
        client_max_body_size 10m;
}
```

Afterwards, we can create a new configuration for our reverse proxy for Foundry.

Copy and paste the following and hit enter.
`sudo nano /etc/nginx/sites-available/foundry`.

You've created a opened a text editor at the path specified. Copy the following contents and paste it into the nano editor, then adjust every occurance of the port number :8080 and your fully qualified domain name (FQDN) to your configuration.

Once you're ready to save hit "Control + X" to exit. When existing it will ask if you want to save your changes so hit "y". Then it will ask for a directory and name, prepopulated with its current directly and name, simply hit "enter" to confirm this and it will save over the existing file.


```
server {
    listen 80;

    # Adjust this to your the FQDN you chose!
    server_name                 foundry.myhost.com;

    access_log                  /var/log/nginx/foundry/access.log;
    error_log                   /var/log/nginx/foundry/error.log;

    location / {
        proxy_set_header        Host $host;
        proxy_set_header        X-Real-IP $remote_addr;
        proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header        X-Forwarded-Proto $scheme;

        # Adjust the port number you chose!
        proxy_pass              http://127.0.0.1:8080;

        proxy_http_version      1.1;
        proxy_set_header        Upgrade $http_upgrade;
        proxy_set_header        Connection "Upgrade";
        proxy_read_timeout      90;

        # Again, adjust both your FQDN and your port number here!
        proxy_redirect          http://127.0.0.1:8080 http://foundry.myhost.com;
    }
}
```

Let's create a link from this configuration to enable it: `sudo ln -s /etc/nginx/sites-available/foundry /etc/nginx/sites-enabled/foundry`. Restart the server to reflect the changes: `sudo service nginx restart`. Check the browser, not pointing at port 80 (we are not yet installing the certificate) to see if the reverse proxy is working correctly: `http://foundry.myhost.com` should now be showing the foundry instance served by the process manager pm2. Looks great already! 
 
### Set up SSL (HTTPS) Encryption
The Let's encrypt!-Homepage has information about installing the Certbot on different systems, the one for [Ubuntu Linux and nginx](https://certbot.eff.org/lets-encrypt/ubuntubionic-nginx) should be the right one. Let's just follow the instructions on their website (from 07/08/2019):

```
sudo apt-get install software-properties-common
sudo add-apt-repository universe
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update
sudo apt-get install certbot python-certbot-nginx
```

installs the software itself and `sudo certbot --nginx` runs the installer for nginx.

Insert an e-mail address that will be contacted regarding critical information, agree to the terms of service after reading them thouroughly, choose if you want to share your e-mail address with the EFF and then choose your FQDN you want to secure (foundry.myhost.com) and enable automatic redirection of someone insists of using non-encrypted connections.

Everything else, including the automatic renewal after three months is now setup and you are done with a sparkling new installation of Foundry using a subdomain of your choice and an encrypted connection of you and your players to the gaming instance. Have fun!

```
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator nginx, Installer nginx
Enter email address (used for urgent renewal and security notices) (Enter 'c' to
cancel): info@myhost.com

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please read the Terms of Service at
https://letsencrypt.org/documents/LE-SA-v1.2-November-15-2017.pdf. You must
agree in order to register with the ACME server at
https://acme-v02.api.letsencrypt.org/directory
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(A)gree/(C)ancel: A

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Would you be willing to share your email address with the Electronic Frontier
Foundation, a founding partner of the Let's Encrypt project and the non-profit
organization that develops Certbot? We'd like to send you email about our work
encrypting the web, EFF news, campaigns, and ways to support digital freedom.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: n

Which names would you like to activate HTTPS for?

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1: foundry.myhost.com
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Select the appropriate numbers separated by commas and/or spaces, or leave input
blank to select all options shown (Enter 'c' to cancel): 1
Obtaining a new certificate
Performing the following challenges:
http-01 challenge for foundry.myhost.com
Waiting for verification...
Cleaning up challenges
Deploying Certificate to VirtualHost /etc/nginx/sites-enabled/foundry

Please choose whether or not to redirect HTTP traffic to HTTPS, removing HTTP access.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1: No redirect - Make no further changes to the webserver configuration.
2: Redirect - Make all requests redirect to secure HTTPS access. Choose this for
new sites, or if you're confident your site works on HTTPS. You can undo this
change by editing your web server's configuration.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Select the appropriate number [1-2] then [enter] (press 'c' to cancel): 2
Redirecting all traffic on port 80 to ssl in /etc/nginx/sites-enabled/foundry

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Congratulations! You have successfully enabled
https://foundry.myhost.com

You should test your configuration at:
https://www.ssllabs.com/ssltest/analyze.html?d=foundry.myhost.com
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/foundry.myhost.com/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/foundry.myhost.com/privkey.pem
   Your cert will expire on 2019-10-06. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot again
   with the "certonly" option. To non-interactively renew *all* of
   your certificates, run "certbot renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```

#### Adding Auto-Refreshing of the SSL Certificate

First, create a new directory that is used by Certbot to auto-refresh your new certificate: `sudo mkdir /var/www/letsencrypt` and reference to this location for a very specific http answer that letsencrypt wants to verify. Open up your Nginx configuration for foundry and add some more configuration items: `sudo nano /etc/nginx/sites-available/foundry`.

Right above the section `location / { .... }` you prepend the following block:

```
include conf.d/drop;

location ^~ /.well-known/acme-challenge {
    allow all;
    root /var/www/letsencrypt;
    auth_basic off;
}
```
At the end, the whole configuration file should look like this:

```
server {
    # Adjust this to your the FQDN you chose!
    server_name                 foundry.myhost.com;

    access_log                  /var/log/nginx/foundry/access.log;
    error_log                   /var/log/nginx/foundry/error.log;

    include conf.d/drop;

    location ^~ /.well-known/acme-challenge {
        allow all;
        root /var/www/letsencrypt;
        auth_basic off;
    }

    location / {
        proxy_set_header        Host $host;
        proxy_set_header        X-Real-IP $remote_addr;
        proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header        X-Forwarded-Proto $scheme;

        # Adjust the port number you chose!
        proxy_pass              http://127.0.0.1:8080;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_read_timeout      90;

        # Again, adjust both your FQDN and your port number here!
        proxy_redirect          https://127.0.0.1:8080 http://foundry.myhost.com;
    }

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/foundry.myhost.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/foundry.myhost.com/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
}

server {
    if ($host = foundry.myhost.com) {
        return 301 https://$host$request_uri;
    } # managed by Certbot

    listen 80;
    server_name                 foundry.myhost.com;
    return 404; # managed by Certbot
}
```

Now create a new file `sudo nano /etc/nginx/conf.d/drop`, add the following contents into it and restart nginx with `sudo service nginx restart`:

```
# Most sites won't have configured favicon or robots.txt
# and since its always grabbed, turn it off in access log
# and turn off it's not-found error in the error log
location = /favicon.ico { access_log off; log_not_found off; }
location = /robots.txt { access_log off; log_not_found off; }
location = /apple-touch-icon.png { access_log off; log_not_found off; }
location = /apple-touch-icon-precomposed.png { access_log off; log_not_found off; }

# Rather than just denying .ht* in the config, why not deny
# access to all .invisible files
location ~ /\. { deny  all; access_log off; log_not_found off; } 
```

## Something Went Wrong
Make sure that your port 443 is accessible. UFW (Universal FireWall) is pretty common, so you can check `ufw status` and see if that is enabled. `sudo ufw allow https` and `sudo ufw allow http` should add ports 80 and 443 to your firewall, thus allowing access to your ssl encrypted foundry instance and the Certbot for it's automatic certification renewal

Check the `pm2 log foundry` for errors on the configuration there. Perhaps foundry starts up, fails and pm2 wants to restart it all the time, resulting in an endless loop of frustration for everybody? Stop the process by `pm2 stop foundry` and run it manually `node /home/ubuntu/foundry/resources/app/main.js` and check for errors (directory permissions are alright? Perhaps you changed anything there and it's just not working?).

## Final Steps
Open up your web browser at http://foundry.myhost.com and it should automatically redirect to the encrypted site at https://foundry.myhost.com. Everything else should work as you are used to.
 
## Connect FoundryVTT with S3

After you’ve set up Foundry, you’ll want to add in the information that Foundry needs to connect to S3. First, you need to create a .json file to contain the access key ID and secret access key, as well as your preferred region. You can place this anywhere you like, but for simplicity’s sake, I like to put it alongside my options.json file, like so.

`nano ~/.local/share/FoundryVTT/Config/s3.json`

The precise path will vary based on how you installed it. The keys you need are in the .csv file you made all the way back in IAM, part 1. The region is the AWS-compliant name of the region your bucket is located in – if you hit the Regions dropdown in the top right, it’ll be in all lower case next to the friendlier name. Copy and paste the following into your s3.json file, replace the accessKeyId, secretAccessKey, and region with the appropriate selections:

```
{
    "accessKeyId": "<your access key id>",
    "secretAccessKey": "<your secret access key",
    "region": "<your region>"
}
```
  
Save the file, then go into the Configuration tab of the Foundry setup page. Put the file path for the .json file into the AWS Configuration Path box, and hit Save Changes. Note that is is from my personal setup – yours may look different.
<img src="https://raw.githubusercontent.com/foundry-vtt-community/wiki/master/images/Getting%20Started/AWS%20Self%20Hosting/7-1.PNG">

Once you’ve done that, you should be able to launch a game table and open the image browser. You should see a tab for Amazon S3 that will let you choose objects uploaded to S3 as well as upload objects to S3. Note that if you like organization, you cannot create folders or delete in S3 from this tab at this time – you’ll need to use the S3 console to create folders for organization, or remove media you no longer need.

## OPTIONAL: Set up NGINX to Sign In prompt for the domain
Should you want to restrict access to the domain and its subdomains via a browser Sign In prompt simply to stop bots from ever crawling you FoundryVTT instance copy and paste the following and hit enter.

@TODO

## OPTIONAL: Set up Landing Page for the main domain

@TODO

## TODO's
Add info about Elastic EC2 IP Addresses
Add better info about domain and sub-domain setup for Hover.com
Add info about Vouch-Proxy SSO login (https://github.com/vouch/vouch-proxy) and (https://www.reddit.com/r/FoundryVTT/comments/hw14rr/anyone_else_using_container/)
Add info about Resource Monitoring for EC2 / S3
Add info about connecting FoudnryVTT storage with S3
  - Finally got it working, but not sure how. Was probably due in part to this tutorial (https://www.youtube.com/watch?v=AQtTG2N_QCg) which recommended this for port changes (https://d.pr/i/vp0opr) and I also learned about "Account Users ID"

## Contributions
- [Foundry VTT Community - Self Hosting on AWS](https://foundry-vtt-community.github.io/wiki/Self-Hosting-on-AWS/)
- [Foundry VTT Community - Unbuntu VW](https://github.com/foundry-vtt-community/wiki/wiki/Ubuntu-VM) - @ShadowMorph from Discord provided additions to the nginx/SSl configuration which Saif Addin partly added to the above config. Without ShadowMorph's support, the auto-refreshing would not have worked and the SSL configuration would be quite a bit less robust.

## Bryan's Repositories
- [Bryan's AWS Setup Guide for FoundryVTT](https://github.com/bryancasler/Bryans-AWS-Setup-Guide-for-FoundryVTT)
- [Bryan's FoundryVTT Assets and Resources](https://github.com/bryancasler/Bryans-FoundryVTT-Assets-And-Resources)
- [Bryan's Preferred Modules for FoundryVTT](https://github.com/bryancasler/Bryans-Preferred-Modules-for-FoundryVTT)
