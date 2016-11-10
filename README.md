# Setting Up a Spark Cluster on Amazon EC2 w/ Monitoring

## Table of Contents

[Overview](#overview)

[EC2 Configuration](#configuration)

[SSH](#ssh)

[Hadoop](#hadoop)

[Spark](#spark)

[Ganglia](#ganglia)

[Errors](#errors)

[WebUIs](#Webuis)

## Overview
In this tutorial we will be 

## Initial Configuration
### Creating Instances
AWS Console > Instances > Launch Instance

1. Choose AMI - Ubuntu Server 14.04 LTS (HVM), SSD Volume Type (Might have to scroll down a bit to find this).

2. Choose an Instance Type

    t2.micro (1GB RAM) is Free Tier eligible.
    
    t2.small (2GB RAM) is not Free Tier eligible, but does provide a boost in ram that is needed for your main machine (the one that will control the datanodes).
    
    IMPORTANT: Here, we will create all nodes as micro, and then switch one of them to small afterwards. Creating 4 at a time all at the micro level helps to keep the DNS names together.

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

IMPORTANT: When you get to end of this guide and have a working Spark environment, the Scala shell that runs on the machine you've now designated as `NAMENODE` will take up a hefty chunk of memory. It is important that you upgrade the namenode to a `t2.small` machine, so that it has 2GB of RAM minimum.

To do this, first select the namenode in the AWS console. With ONLY the namenode selected, go to `Actions > Instante Settings > Change Instance Type`. This will bring up a new window, choose `t2.small` and hit apply. 

Now select all 4 of your machines and select `Actions > Instance State > Start`. You now have 4 working EC2 Instances.

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

Now exit back out to your local machine. (CTRL-D or type `exit`).

### Passwordless SSH
The namenode has to be able to communicate with the datanodes via ssh, but it does not have access to the pem key. For this, we need to enable passwordless SSH.

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

Now you no longer rely on having the ec2_key.pem file. It is still important to backup this key somewhere, in case you want to give someone else access to the system, or your computer crashes.

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
        <name>fs.default.name</name>
        <value>hdfs://NAMENODE_PUBLIC_DNS</value>
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
NAMENODE$ rm $HADOOP_CONF_DIR/slaves
NAMENODE$ touch $HADOOP_CONF_DIR/masters $HADOOP_CONF_DIR/slaves
```
Let's edit the `masters` file first.
```bash
NAMENODE$ vim $HADOOP_CONF_DIR/masters
```
```bash
NAMENODE_HOSTNAME
```
Now the slaves file.
```bash
NAMENODE$ vim $HADOOP_CONF_DIR/slaves
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
NAMENODE$ $HADOOP_BASE/sbin/start-dfs.sh
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
ALLNODES$ cp $SPARK_BASE/conf/spark-env.sh.template $SPARK_BASE/conf/spark-env.sh
ALLNODES$ vim $SPARK_BASE/conf/spark-env.sh
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
NAMENODE$ vim $SPARK_BASE/conf/slaves
```
```bash
DATANODE1_PUBLIC_DNS
DATANODE2_PUBLIC_DNS
DATANODE3_PUBLIC_DNS
```

Now you can start all worker nodes with the following command.
```bash
NAMENODE$ $SPARK_BASE/sbin/start-all.sh
```

View the SPARK Web UI at NAMENODE_PUBLIC_DNS:8080

### Scala Shell
Now to test that everything works.

We will be using GeoSpark, a cluster computing system that processes large-scale spatial data. We can get a precompiled GeoSpark JAR file from the [GeoSpark](github.com/DataSystemsLab/GeoSpark/) repo.

```bash
wget https://github.com/DataSystemsLab/GeoSpark/releases/download/0.3.2/geospark-0.3.2-spark-2.x.jar
```

Create a directory for third-party jar files in the $SPARK_BASE directory, and place the geospark jar inside of it.

```bash
NAMENODE$ mkdir $SPARK_BASE/user_jars
NAMENODE$ mv geospark-* $SPARK_BASE/user_jars/geospark.jar
```

The command to start the shell involves calling `$SPARK_BASE/bin/spark-shell` with addtional parameters pointing it to both the master node, and giving it access to the geospark jar. Let's create an alias for it to save space.

```bash
vim ~/.profile
```
Go to the bottom of the file and add the alias `start-shell`.
```bash
alias start-shell='$SPARK_BASE/bin/spark-shell --jars $SPARK_BASE/user_jars/geospark.jar --master spark://NAMENODE_PUBLIC_DNS:7077'
```

Let's break down the pieces of this alias.
`$SPARK_BASE/bin/spark-shell` is calling the Scala-Spark REPL shell.

`--jars` is letting the shell know that we want to start with additional third party jar files, and then we point to the location of `geospark.jar`.

`--master` is letting the shell know the address of a master node so that it can communicate with the node that controls all of the datanodes. In our case, it's the public DNS of our namenode (NAMENODE_PUBLIC_DNS)

Save and exit vim, then source your `~/.profile` so that your new alias is created.

```bash
source ~/.profile
```

Now when we run the alias, a scala shell will start up.
```bash
ALLNODES$ start-shell
```

When a scala shell gets created, a new WebUI is available that let's you see the individual jobs running inside of the shell. That WebUI is available at `NAMENODE_PUBLIC_DNS:4040`.

You can load a script using `:load path/to/file_name.scala`


If your shell crashes during this part, it is most likely because you ran out of memory on your `NAMENODE`, and picked the 1GB machine at the beginning. Please see [Errors](#errors) for more information.

You can load a script using `:load FILE_NAME`

You can safely exit the shell using `:q`

## Ganglia

## Errors
Insufficient Memory
```bash
# There is insufficient memory for the Java Runtime Environment to continue.
# Native memory allocation (malloc) failed to allocate 36831232 bytes for committing reserved memory.
```

This error occurs when you have too many things running on the `NAMENODE` machine. It can typically be fixed by killing all java processes on this machine. (`sudo killall java`). This does however kill hadoop as well, and you will have to restart hadoop.

I have provided scripts that start hadoop from scratch, and loads example csv files into the Hadoop File System in case you wish to stay with a 1GB RAM machine. Your workflow if you wish to stay with a 1GB machine would be to kill all java processes on the `NAMENODE` and then restart hadoop everytime your shell crashes.

This can be cumbersome, and it is probably best to switch to at least a 2GB machine for your `NAMENODE`.

## WebUIs

Now that everything is up and running, there are quite a bit of Web Interfaces that let you monitor the tasks and services you have running, You might want to bookmark them: 

Hadoop - `NAMENODE_PUBLIC_DNS:50070`

Spark Master - `NAMENODE_PUBLIC_DNS:8080`

Spark Jobs - `NAMENODE_PUBLIC_DNS:4040`

Ganglia `NAMENODE_PUBLIC_DNS/ganglia`
