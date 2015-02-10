
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






Paragraphs are separated by a blank line.

2nd paragraph. *Italic*, **bold**, and `monospace`. Itemized lists
look like:

  * this one
  * that one
  * the other one

Note that --- not considering the asterisk --- the actual text
content starts at 4-columns in.

> Block quotes are
> written like so.
>
> They can span multiple paragraphs,
> if you like.

Use 3 dashes for an em-dash. Use 2 dashes for ranges (ex., "it's all
in chapters 12--14"). Three dots ... will be converted to an ellipsis.
Unicode is supported. ☺



An h2 header
------------

Here's a numbered list:

 1. first item
 2. second item
 3. third item

Note again how the actual text starts at 4 columns in (4 characters
from the left side). Here's a code sample:

    # Let me re-iterate ...
    for i in 1 .. 10 { do-something(i) }

As you probably guessed, indented 4 spaces. By the way, instead of
indenting the block, you can use delimited blocks, if you like:

~~~
define foobar() {
    print "Welcome to flavor country!";
}
~~~

(which makes copying & pasting easier). You can optionally mark the
delimited block for Pandoc to syntax highlight it:

~~~python
import time
# Quick, count to ten!
for i in range(10):
    # (but not *too* quick)
    time.sleep(0.5)
    print i
~~~



### An h3 header ###

Now a nested list:

 1. First, get these ingredients:

      * carrots
      * celery
      * lentils

 2. Boil some water.

 3. Dump everything in the pot and follow
    this algorithm:

        find wooden spoon
        uncover pot
        stir
        cover pot
        balance wooden spoon precariously on pot handle
        wait 10 minutes
        goto first step (or shut off burner when done)

    Do not bump wooden spoon or it will fall.

Notice again how text always lines up on 4-space indents (including
that last line which continues item 3 above).

Here's a link to [a website](http://foo.bar), to a [local
doc](local-doc.html), and to a [section heading in the current
doc](#an-h2-header). Here's a footnote [^1].

[^1]: Footnote text goes here.

Tables can look like this:

size  material      color
----  ------------  ------------
9     leather       brown
10    hemp canvas   natural
11    glass         transparent

Table: Shoes, their sizes, and what they're made of

(The above is the caption for the table.) Pandoc also supports
multi-line tables:

--------  -----------------------
keyword   text
--------  -----------------------
red       Sunsets, apples, and
          other red or reddish
          things.

green     Leaves, grass, frogs
          and other things it's
          not easy being.
--------  -----------------------

A horizontal rule follows.

***

Here's a definition list:

apples
  : Good for making applesauce.
oranges
  : Citrus!
tomatoes
  : There's no "e" in tomatoe.

Again, text is indented 4 spaces. (Put a blank line between each
term/definition pair to spread things out more.)

Here's a "line block":

| Line one
|   Line too
| Line tree

and images can be specified like so:

![example image](example-image.jpg "An exemplary image")

Inline math equations go in like so: $\omega = d\phi / dt$. Display
math should get its own line and be put in in double-dollarsigns:

$$I = \int \rho R^{2} dV$$

And note that you can backslash-escape any punctuation characters
which you wish to be displayed literally, ex.: \`foo\`, \*bar\*, etc.
