
HOWTO: Access your AWS:RDS database from localhost.
===============================================

As my first foray into the world of Amazon Web Services, we had some data that it was convenient to host
on an RDS database, as it needed to be accessed from many people in many places.

Naturally, we did not want our database open to the world, so we decided the secure thing to do would be
to set up an ssh tunnel to the database instance.  Unfortunately, Amazon did not really set up RDS for this sort
of thing, so we had to go round-about, tunneling through an EC2 instance.

# Overview:
1. [Set up security](#security)
2. [Set up your EC2 instance](#ec2)
3. [Set up your RDS instance](#rds)
4. [SSH tunnel through EC2 to RDS](#ssh)
5. [Log in to your database](#database)

<a name="security"/>
## Set up security

First, we need to set up the framework to allow our EC2 instance and our RDS instance to communicate.  The AWS process for this
is to put them on the same VPC.  

Log in to AWS, go to the VPC panel, and create a new VPC.  Name it something like "database_access" and give it any CIDR you like.
I think the pop-up suggests 10.0.0.0/16.  It doesn't really matter as long as it doesn't overlap with any other VPC you have set up.
[CIDR explained](http://phpfunk.com/uncategorized/cidr-notation-explained-simply/)

Now, we have a VPC that we can assign our EC2 instance to and our RDS instance to, and they'll be able to communicate.  AWS makes this
easy by creating a default security group for the VPC, which basically allows any communication between members of the VPC.  We'll see
how this works soon...

Second, however, we'll need one more security group.  Navigate now to the AWS EC2 panel, and click on the tab to edit your security groups.
You should see one for the VPC you just created, and now we're going to create one more so that we can access our EC2 instance, that we'll be
creating soon.  So... Create a new security group, assign it to your new VPC and give it access to SSH port 22.  You can assign a range of
IP addresses that you'll be accessing it from, but I don't have a static IP, so I didn't do this.

Third, you'll need a ssl key to access your EC2 instance.  In the EC2 panel, find the section to create Key Pairs.  Create a new key pair,
and save the pem file somewhere safe.  Now, to keep people who aren't you from reading your key file:

```
chmod 400 mykey.pem
```

We'll need this key file later, so let's call the path to it \[keyFilePath\]

Now we have our security all set up.

<a name="ec2"/>
## Set up your EC2 instance

Go back to the EC2 panel, and create a new EC2 instance.  Put it in the SSH security group you created earlier, and also put it in your database
access VPC.  Inspect your instance by clicking on it vigorously, and somewhere you should see a Public DNS entry.  This is how you will access
your EC2 instance.  Copy this down, because we'll use it in a later section.  Call this thing \[ec2Domain\]

Also, you'll need to determine your EC2 login name.  It's actually not what you would think.  It's different based on the operating system you
used for your EC2 instance.  Look [here](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AccessingInstancesLinux.html) and do a find for 
"ec2-user" for an explanation.  Call this thing \[ec2User\]

Now our EC2 instance is all set up.

<a name="rds"/>
## Set up your RDS instance
 
Go to the RDS panel, and create a new RDS instance.  Make a note of the database name (NOT the instance name) and the user 
(hereafter \[dbName\],  \[dbUser\])

Put it in the database access VPC.  Inspect the instance by clicking around, and find the database endpoint.  Note that this includes
a domain followed by a port.  Copy these down, separately.  (\[dbEndpoint\],  \[dbPort\])

The port will depend on the type of database you set up.  I set mine up using postgres, but it really shouldn't matter.

<a name="ssh"/>
## SSH tunnel through EC2 to RDS

Pick some unused local port to tunnel through.  I'm fond of just tacking a zero to the end of standard ports.  For example, the port for postgres
is usually 5432, so instead, I use local port 54320 to attach to non-local servers.  Make up a local port for yourself, and call it \[localPort\]

Now, do this:

```
ssh -N -L [localPort]:[dbEndpoint]:[dbPort] [ec2User]@[ec2Domain] -i [keyFilePath] 
```

This creates an ssh tunnel from our local port, through the ec2 instance, to the database port specified on the database instance.

<a name="database"/>
## Log in to your database

This will depend on what type of database you ran, so you may have to do some research here, but for postgres, we use:

```
psql -h localhost -p 54320 -U [dbUser] [dbName]
```

This connects to a postgres server on localhost at port 54320, with the appropriate user name and database name.  Note that we don't actually
have a database on host localhost, but the ssh tunnel essentially routes port 5432 on the actual database instance back and forth to this port
on localhost, so as far as psql is concerned, it can't tell the difference.

Now, open a beer, and enjoy your ultra-secure database access.

