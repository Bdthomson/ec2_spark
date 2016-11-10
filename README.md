# Setting Up a Spark Cluster on Amazon EC2 w/ Monitoring

## Table of Contents

[Overview](#overview)

[EC2 Configuration](#configuration)

[SSH](#ssh)

[Hadoop](#hadoop)

[Spark](#spark)

[Ganglia](#ganglia)

## Overview
In this tutorial we will be 

## Initial Configuration
### Creating Instances
AWS Console > Instances > Launch Instance

1. Choose AMI - Ubuntu Server 14.04 LTS (HVM), SSD Volume Type (Might have to scroll down a bit to find this).

2. Choose an Instance Type - t2.micro is Free tier eligible.

If you would like an instance type with more resources, a price list is available here: http://www.ec2instances.info/

3. Configure Instance Details - Number of Instances = 4

4. Add Storage - 8GB/General Purpose SSD (Default)

5. Tag Instance - Create a key/value pair 'Name': 'newNode' (We will change this in a minute).

6. Configure Security Group - 'Select an existing security group'. There should be two options now, choose 'open' ('open to the world') for now.

7. Review Instance Launch - Launch

8. You will be asked to choose a pem-key. If this is your first time with Amazon EC2, you probably have no generated one yet. It will generate one for you, and then you can download it to your machine (my key is named ec2_key.pem). 

(IMPORTANT: You must download this key pair right now. If you do not download at this moment, you will not be able to access your machines, ever.)



### Adjusting Instances
Now in the Instances tab you should see your 4 nodes, all with the name 'newNode'. Change one of them to namenode, and the other three to datanode1, datanode2, and dataenode3. (When you click on a node, the bottom console populates with details about that node. 
    
When spawning multiple nodes, the private ip addresses of my nodes have always been next to eachother (i.e. 60, 61, 62, and 63). It might be convenient if you name the lowest one namenode, and the rest datanode1,2 and 3 by increasing ip).

### Important

It is important to note that at certain points, the installation instructions will split into four categories: `LOCAL`, `ALLNODES`, `NAMENODE`, and `DATANODES`. The `LOCAL` identifier will represent your local machine. 

Things like java need to be installed on all nodes, including the namenode, and will be represented using `ALLNODES` (**This does NOT include your local machine**). However, certain configuration settings only apply to the namenode (`NAMENODE`), and other settings only apply to datanodes (`DATANODES`). 

I will differentiate between them by prepending instructions with identifiers like so:

```bash
ALLNODES$ # Install Java
```

In the above command, we would run the given command on all nodes (the namenode, and 3 datanodes in our setup).

## SSH
Make note of the 'Public DNS' of your 4 machines. It will be helpful to write the names of all your machines with their corresponding DNS address (i.e. namenode - ec2-xx-xx-xx-xx.us-west-2.compute.amazonaws.com).

Once you've downloaded the key, move it to your ~/.ssh/ folder. 

```bash
LOCAL$ mv ~/Downloads/ec2_key.pem ~/.ssh/
```

If you do not already have an ssh config file, create one and edit it with 
```bash
LOCAL$ touch ~/.ssh/config && vim ~/.ssh/config
```

In this tutorial, you will have to make note to replace things in all CAPS with your corresponding value. i.e. if I have `NAMENODE_PUBLIC_DNS` in a file, replace it with your namenode's public dns address.

Here we will place all neccessary info for SSH'ing into our machines. Set up your file like so (replacing the public dns values with your own): 

```bash
Host *
    ServerAliveInterval 120
    
Host namenode
    HostName NAMENODE_PUBLIC_DNS
    User ubuntu
    IdentityFile ~/.ssh/ec2_key.pem

Host datanode1
    HostName DATANODE1_PUBLIC_DNS
    User ubuntu
    IdentityFile ~/.ssh/ec2_key.pem

Host datanode2
    HostName DATANODE2_PUBLIC_DNS
    User ubuntu
    IdentityFile ~/.ssh/ec2_key.pem

Host datanode3
    HostName DATANODE3_PUBLIC_DNS
    User ubuntu
    IdentityFile ~/.ssh/ec2_key.pem
```

Note: The line at the top about ServerAliveInterval will send a pulse every 120 seconds, preventing the client from being disconnected. This is needed to prevent getting disconnected from a server shell because you were idle for too long (i.e. getting the 'Broken Pipe' error.)

Now you have to change the permissions on the key, otherwise it will give you an error.
```bash
LOCAL$ cd ~/.ssh/ && sudo chmod 600 ~/.ssh/ec2_key.pem
```

Now let's ssh into your machines.

```bash
LOCAL$ ssh namenode
```

If this is the first time you are accessing this machine, it will ask you if you want to continue, type 'yes' and hit enter.

Quick Note: If you are idle for a bit after SSH'ing into your machine, you probably will get kicked off after a certain period of time. To prevent this from happening, we can set a variable inside of the namenode's ssh settings.

```bash
NAMENODE$ sudo vim /etc/ssh/sshd_config
```

In here, add the line `ClientAliveInterval 120`, and  When saving the file, you might get a 'READ ONLY' error. This is to prevent certain files from being changed. This change is fine, use `wq!` to quit. It will force the save.

Now exit back out to your local machine. (CTRL-D or type `exit`).

### Passwordless SSH
The namenode has to be able to communicate with the datanodes via ssh. For this, we need to set up passwordless SSH.

Copy your pem and ssh config files from your local machine to the namenode. We will copy over SSH using `scp`. 

```bash
LOCAL$ scp ~/.ssh/ec2_key.pem namenode:~/.ssh/
LOCAL$ scp ~/.ssh/config namenode:~/.ssh/
```
We will only be using this ec2_key.pem file temporarily to get into the datanodes from the namenode.

SSH back into namenode

```bash
LOCAL$ ssh namenode
```

We will create an rsa key pair on namenode. The private key stays on namenode, and the public key will get sent to our datanodes in a minute.

```bash
NAMENODE$ ssh-keygen -f ~/.ssh/id_rsa -t rsa -P ""
```

Note: `ssh-keygen` generates a public/private key pair for authorization. -f is the file option, followed by the name of your desired key file name. -t is the option for specifying the type of key, followed by the rsa type. -P is the desired password for accessing opening the key, which we set to empty using "". This command generates two keys, the private (id_rsa) and the public (id_rsa.pub).

Think of the public key as a lock, and the private key is the key to open that lock. Your public key can go anywhere, and get sent anywhere, as long as your private key is not compromised.

Currently we can access datanodes from the namenode by using the pem key - this is because there currently exists a single lock on the datanodes that accepts our pem file as it's key. We can then place our new public keys (locks) on the datanodes to give us an additional way to access them, without needing the pem key.

Let's first add the the public key to the namenode's list of authorized_keys.

```bash
NAMENODE$ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```

Now we need to add the public key to the `~/.ssh/authorized_keys` of each datanode.

```bash
NAMENODE$ cat ~/.ssh/id_rsa.pub | ssh datanode1 'cat >> ~/.ssh/authorized_keys'
NAMENODE$ cat ~/.ssh/id_rsa.pub | ssh datanode2 'cat >> ~/.ssh/authorized_keys'
NAMENODE$ cat ~/.ssh/id_rsa.pub | ssh datanode3 'cat >> ~/.ssh/authorized_keys'
```

Optional: You can even do the same process from your local machine. Check if you already have an ssh-key in your `~/.ssh/` directory. If not, run the ssh-keygen command from before, and you can run the transfer command from before for each datanode:

```bash
LOCAL$ cat ~/.ssh/id_rsa.pub | ssh namenode 'cat >> ~/.ssh/authorized_keys'
LOCAL$ cat ~/.ssh/id_rsa.pub | ssh datanode1 'cat >> ~/.ssh/authorized_keys'
LOCAL$ cat ~/.ssh/id_rsa.pub | ssh datanode2 'cat >> ~/.ssh/authorized_keys'
LOCAL$ cat ~/.ssh/id_rsa.pub | ssh datanode3 'cat >> ~/.ssh/authorized_keys'
```

Now you no longer rely on having the ec2_key.pem file. It is still important to hold onto this key however, in case your computer crashes, you want to give someone else access to the system.

## Hadoop

### All Nodes (Namenode + Datanodes)

Now we will install hadoop on all four machines.

If using iterm, you can enable `Shell > Broadcast Input > Broadcast Input to All Panes in Current Tab` from the iTerm menu that will send all of your commands to all machines, saving you some time. It is also helpful to turn the background indicator on so that you know which shells are being broadcast too. 

First we will install java.

```bash
ALLNODES$ sudo apt-get install default-jdk
```

You can check that it installed successfully with the command:

```bash
ALLNODES$ java -version
```
Next we will download, unzip, and install hadoop.

```bash
ALLNODES$ wget http://apache.claz.org/hadoop/common/hadoop-2.7.3/hadoop-2.7.3.tar.gz
ALLNODES$ sudo tar -zxf ~/hadoop-* -C /usr/local/ # Extract the tar file to the /usr/local/ directory.
ALLNODES$ sudo mv /usr/local/hadoop-* /usr/local/hadoop # Rename hadoop-version-number to just hadoop.
```

Now we have to set a few environment variables. Open up your ~/.profile.
```bash
ALLNODES$ vim ~/.profile
```
```bash
export HADOOP_BASE=/usr/local/hadoop
export HADOOP_CONF_DIR=$HADOOP_BASE/etc/hadoop
export PATH=$PATH:$HADOOP_BASE/bin

# This might be different for you depending on when you are reading this. 
# Java is installed in /usr/lib/jvm/java-x-openjdk-x. Find it and place the path here.
export JAVA_HOME=/usr/lib/jvm/java-7-openjdk-amd64 
```

The first statement will allow us to run hadoop executables without having to reference their exact location. The second statement makes it easier to edit hadoop config files. The third statement sets the java location that is referenced in one of the hadoop config files.

Make sure you `source` the `~/.profile` for your changes to take effect.
```bash
ALLNODES$ source ~/.profile
```
Before we edit hadoop config files, we need to change owner to `ubuntu`. Otherwise, we will get an error when trying to save the file, because it is currently owned by root.

```bash
ALLNODES$ sudo chown ubuntu -R $HADOOP_BASE
```

Now we will edit the first of a few hadoop config files. Open $HADOOP_CONF_DIR/core-site.xml and place the following config information. Be sure to use the actual public DNS for your namenode.

```bash
ALLNODES$ vim $HADOOP_CONF_DIR/core-site.xml
```

```xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://NAMENODE_PUBLIC_DNS:54310</value>
    </property>
</configuration>
```

Now we will edit the `yarn-site.xml` file.

```bash
ALLNODES$ vim $HADOOP_CONF_DIR/yarn-site.xml
```

```xml
<configuration>
    <property>
            <name>yarn.nodemanager.aux-services</name>
            <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>
        <value>org.apache.hadoop.mapred.ShuffleHandler</value>
    </property>
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>NAMENODE_PUBLIC_DNS</value>
    </property>
</configuration>
```

The last file we are going to edit on all nodes does not exist yet, but a template for it does. Copy over the template with:

```bash
ALLNODES$ cp /usr/local/hadoop/etc/hadoop/mapred-site.xml.template $HADOOP_CONF_DIR/mapred-site.xml
ALLNODES$ vim $HADOOP_CONF_DIR/mapred-site.xml
```

```xml
<configuration>
    <property>
        <name>mapreduce.jobtracker.address</name>
        <value>NAMENODE_PUBLIC_DNS:54311</value>
    </property>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
</configuration>
```
Before you shutoff broadcasting in iterm, run one last command on all nodes:

```bash
ALLNODES$ echo $(hostname)
```

Make note of all hostnames, they will be used in the next step.

### Namenode ONLY

The following commands are specific only to the namenode. Make sure you are not in broadcast mode in iterm.

Open up `/etc/hosts` and add the DNS and hostnames (from the last step). Make sure to place it below localhost, but above all other settings.

```bash
NAMENODE$ sudo vim /etc/hosts
```
```bash
127.0.0.1 localhost
NAMENODE_PUBLIC_DNS NAMENODE_HOSTNAME
DATANODE1_PUBLIC_DNS DATANODE1_HOSTNAME
DATANODE2_PUBLIC_DNS DATANODE2_HOSTNAME
DATANODE3_PUBLIC_DNS DATANODE3_HOSTNAME
...
```

<!-- Now we will create the HDFS data directory to house HDFS data on namenode.

```bash
NAMENODE$ mkdir -P $HADOOP_BASE/hdfs/namenode
``` -->

Now to edit the hdfs hadoop configuration file. Open up hdfs-site.xml and change the config:
```bash
NAMENODE$ vim $HADOOP_CONF_DIR/hdfs-site.xml
```
It is best practice to set the value for `dfs.replication` as the number of datanodes that you have.
```xml
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>3</value>
    </property>
    <!-- <property>
        <name>dfs.namenode.name.dir</name>
        <value>file:///usr/local/hadoop/hdfs/namenode</value>
    </property> -->
</configuration>
```

Now we will remove the old slaves file, then create `masters` and `slaves` files in the hadoop config.

```bash
rm $HADOOP_CONF_DIR/slaves
touch $HADOOP_CONF_DIR/masters $HADOOP_CONF_DIR/slaves
```
Let's edit the `masters` file first.
```bash
vim $HADOOP_CONF_DIR/masters
```
```bash
NAMENODE_HOSTNAME
```
Now the slaves file.
```bash
vim $HADOOP_CONF_DIR/slaves
```
```bash
DATANODE1_HOSTNAME
DATANODE2_HOSTNAME
DATANODE3_HOSTNAME
```

### Datanode ONLY

It will be helpful to turn broadcasting back on in iterm at this point. IMPORTANT: Make sure that you close out of your `namenode` terminal before enabling broadcasting.

<!-- Now we will create the HDFS data directory to house HDFS data on datanodes.

```bash
DATANODES$ mkdir -P $HADOOP_BASE/hdfs/datanode
``` -->

Now to edit the hdfs hadoop configuration file. Open up hdfs-site.xml and change the config:
```bash
DATANODES$ vim $HADOOP_CONF_DIR/hdfs-site.xml
```

```xml
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>3</value>
    </property>
    <!-- <property>
        <name>dfs.datanode.data.dir</name>
        <value>file:///usr/local/hadoop/hdfs/datanode</value>
    </property> -->
</configuration>
```

### Starting Hadoop
Back on namenode, we can now we can start Hadoop. Make sure you've disabled iterm broadcasting.

First format hdfs.

```bash
NAMENODE$ hdfs namenode -format
```
Now start the distributed file system.
```bash
$HADOOP_BASE/sbin/start-dfs.sh
```

If it asks about authenticity, this is because the hadoop system has not accessed other nodes via SSH yet. Just hit yes/enter until the questions stop. 

You can view your Hadoop Web UI using a browser at NAMENODE_PUBLIC_DNS:50070

## Spark

### All Nodes (Namenode + Datanodes)

Now we will install Scala on each machine.

```bash
ALLNODES$ sudo apt-get install scala
```

Download, extract, and install Spark in `/usr/local`

```bash
ALLNODES$ wget http://apache.mirrors.ionfish.org/spark/spark-2.0.1/spark-2.0.1-bin-hadoop2.7.tgz
ALLNODES$ sudo tar -xzf spark-* -C /usr/local
ALLNODES$ sudo mv /usr/local/spark-* /usr/local/spark
```

Add a new environment variable and edit an existing one in `~/.profile`
```bash
ALLNODES$ vim ~/.profile
```
```bash
export SPARK_BASE=/usr/local/spark
export export PATH=$PATH:$HADOOP_BASE/bin:$SPARK_BASE/bin
```

Source the profile, then take ownership of the Spark directory.
```bash
ALLNODES$ source ~/.profile
ALLNODES$ sudo chown -R ubuntu $SPARK_BASE
```
Now we will create a spark configuration file from an existing template and open it.

```bash
cp $SPARK_BASE/conf/spark-env.sh.template $SPARK_BASE/conf/spark-env.sh
vim $SPARK_BASE/conf/spark-env.sh
```
Edit to look like the following.
```bash
export JAVA_HOME=/usr/lib/jvm/YOUR_JAVA_FOLDER
export SPARK_WORKER_CORES=3 # Set this to 3 times the number of cores per machine.

# This environment variable is specific to the node you are on. In namenode, put the NAMENODE_PUBLIC_DNS, and use DATANODE_X_PUBLIC_DNS for the datanodes.
export SPARK_PUBLIC_DNS=THIS_NODE_PUBLIC_DNS
```

### Namenode ONLY

We are editing config files on just the Namenode now, make sure to disable broadcating.

We will be copying over another template that exists.
```bash
NAMENODE$ cp $SPARK_BASE/conf/slaves.template $SPARK_BASE/conf/slaves
```

Open it, go to the bottom, make sure to delete the line with `localhost`. Add the following.
```bash
vim $SPARK_BASE/conf/slaves
```
```bash
DATANODE1_PUBLIC_DNS
DATANODE2_PUBLIC_DNS
DATANODE3_PUBLIC_DNS
```

Now you can start all worker nodes with the following command.
```bash
$SPARK_BASE/sbin/start-all.sh
```

View the SPARK Web UI at NAMENODE_PUBLIC_DNS:8080

## Ganglia
