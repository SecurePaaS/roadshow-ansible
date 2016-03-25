# OpenShift Roadshow Environment Ansible Scripts

This installation can be used to standup an OpenShift Enterprise roadshow environment on AWS.  

It can also be used to stand up your own environment running the demo from the
[2015 JBoss Middleware Keynote](https://www.youtube.com/watch?v=wWNVpFibayA) at Red Hat Summit. 
 

## AWS Prerequisites

- An AWS account with permissions to do the following:
  - Create and modify a VPC (A VPC is created for each cluster-id)
  - Create and modify Security Groups (2 security groups are created for each
    cluster-id, one for masters and one for nodes)
  - Create and modify route53 entries (Route 53 entries are added to the
    hosted zone specified by r53-zone for each ec2 instance created as well as
    a wildcard dns entry for router)
  - Create and modify EC2 instances
- AWS credentials may be specified either through the `AWS_ACCESS_KEY_ID` and
    `AWS_SECRET_ACCESS_KEY` env variables or using any of the environment
    variables/configs supported by
    [boto](http://boto.readthedocs.org/en/latest/boto_config_tut.html)
- A pre-created route53
    [public hosted zone](http://docs.aws.amazon.com/Route53/latest/DeveloperGuide/CreatingHostedZone.html)
- A pre-created ec2 keypair
  - You will need to specify the name of this keypair when running the
      environment creation script
  - You will also need to add the private key to your ssh agent: `ssh-add <path to key file>`

**Other Amazon AWS Notes**

- If you do not have your `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`, you can create a new user account under the Identity & Access Management (IAM).  You can use your root keys, but this is not recommended by Amazon.
- Identity & Access Management (IAM) Policies, Users / Permissions - If you are not sure which detailed permissions based on the above requirements, the below will be more than sufficient.
  - AmazonEC2FullAccess
  - AmazonRoute53FullAccess
- Create the AWS keypair in your preferred region 
  - Keypair creation is specific to the AWS region 
- The default AMI/Region referenced by the `run.py` script is `us-east-1`
  - `us-east-1` is `ami-2051294a`
  - `us-west-1` is `ami-d1315fb1`
  - `us-west-2` is `ami-775e4f16`
  - You can find the AMI for your specific region with AMI search based on the this description.  EC2 -> AMIs -> "Public images" - `RHEL-7.2_HVM_GA-20151112-x86_64-1-Hourly2-GP2`  
- Route53 must be setup
- Environment size and capacity
  - Small: 8 Users @ 95% CPU, 9-10 Users (Untested)
  - Medium: 15 Users
  - Large: 20 Users @ 70% CPU
- Logging in to your amazon instance
  ```
  ssh -i keypair.pem openshift@ec2-xxxx.compute-1.amazonaws.com
  ```

## System Requirements
- Use RHEL 7, other linux instances didn't work.
- Ensure your machine is entitled with a standard subscription, if centos install EPEL
- Install python 2.7 (should be the default in the yum repo)
  ```
  [root@rhel7 ~]# yum install python
  ```
- Install pip (python should be installed OOTB)
  ```
  [root@rhel7 ~]# wget https://pypi.python.org/packages/source/s/setuptools/setuptools-7.0.tar.gz --no-check-certificate
  [root@rhel7 ~]# tar xzf setuptools-7.0.tar.gz
  [root@rhel7 ~]# cd setuptools-7.0
  [root@rhel7 ~]# python setup.py install
  ...
  Installed /usr/lib/python2.7/site-packages/setuptools-7.0-py2.7.egg
  Processing dependencies for setuptools==7.0
  Finished processing dependencies for setuptools==7.0  
  
  [root@rhel7 ~]# wget https://bootstrap.pypa.io/get-pip.py
  [root@rhel7 ~]# python get-pip.py
  Downloading/unpacking pip
    Downloading pip-1.5.6-py2.py3-none-any.whl (1.0MB): 1.0MB downloaded
  Installing collected packages: pip
  Successfully installed pip
  Cleaning up...
  ```
- [Ansible](https://github.com/ansible/ansible) version 1.9.4
  - v2.0+ may not work, errors may occur upon initial run
  ```
  [root@rhel7 ~]# pip install ansible==1.9.4
  ```
- [Click](https://github.com/mitsuhiko/click) version 3.0 or greater

  ```
  [root@rhel7 ~]# pip install click==3.0
  ```
- [Boto](https://github.com/boto/boto) 
  ```
  [root@rhel7 ~]# pip install boto
  
  ```
- The master branch of
    [openshift/openshift-ansible](https://github.com/openshift/openshift-ansible)
    is expected to be a sibling repo to the demo-ansible repo

  ```
  git clone https://github.com/securepaas/roadshow-ansible.git -b roadshow
  git clone https://github.com/openshift/openshift-ansible.git
  ```
- Before running run.py, associate your private AWS pem key keypair
  - Ensure your keypair has the proper permissions
```
chmod 400 keypair.pem
```
  - **OPTION 1** Use IdentityFile reference in `~/.ssh/config`, below is a sample config file.  The host may be different depending on the EC2 region.
```
Host *.compute-1.amazonaws.com
  IdentityFile ~/.ssh/keypair-east.pem
```
```
Host *.us-west-1.compute.amazonaws.com
  IdentityFile ~/.ssh/keypair-west.pem
```
  - **OPTION 2** Load into `ssh-agent` process via `ssh-add`
```
ssh-add keypair.pem
```

## OpenShift notes
- If you need to change the OpenShift Enterprise version to deploy, you must edit the `roadshow-ansible/playbooks/openshift_setup.yml`  It's yaml so be conscious of spacing.
  - Under the section: `Add the created hosts to groups for configuration` modify two lines with the variable named `openshift_pkg_version`.  This was added to the original `openshift_setup.yml`.
  - Commenting out the two variable lines will tell the ansible installer to pull the latest version
  
    ```
    - hosts: localhost
    ...
      tasks:
      - add_host: 
      ...
          osm_default_node_selector: "region={{ os_defaults.regions.1.name }}"
          openshift_pkg_version: -3.1.1.6-1.git.0.b57e8bd.el7aos
        with_items:master_host
      - add_host:
      ...
          openshift_node_sdn_mtu: 1450
          openshift_pkg_version: -3.1.1.6-1.git.0.b57e8bd.el7aos
        with_items: node_hosts    
      ...
    ```
    
  - Known versions
  ```
  -3.1.0.4-1.git.15.5e061c3.el7aos
  -3.1.1.6-1.git.0.b57e8bd.el7aos
  ```
- The script `roadshow_aws.sh` is available to test if your environment is sufficient for the amount of roadshow attendees.
    - This script runs through the scenarios outlined in the Roadshow documentation
    - Specify the following variables in the script or pass as parameters `NUM_USERS`, `CLUSTER_ID` and `DOMAIN_NAME`
    ```
    roadshow_aws.sh <NUM_USERS> <CLUSTER_ID> <DOMAIN_NAME>
    ```
   
- Each roadshow user consumes about **1.8-2.6GB** memory after completion of the roadshow materials.

## Stopping / Starting OpenShift instances
- If you plan on stopping and starting the AWS instances, start the instances in the following order:
  - `openshift-master` instance (wait ~1 minute)
  - `openshift-node-infra` instance (wait ~1 minute)
  - `openshift-node-demo` instances 
- The Route53 DNS IP settings need to be updated after a stop/start.  This is not an automatic process.  The external IPs are available once the instance is started.  Replace the old IP with the new one.  The following DNS entries are required for proper external access and roadshow functionality.  
  - `*.apps.<cluster_id>.<domain_name/r53 zone>`
  - `openshift-master.<cluster_id>.<domain_name/r53 zone>`
- The `docker-registry` in this roadshow-ansible has been modified to use a persistent store for the registry so that the built projects are able to survive stop/start scenarios.


## Standing up a new Environment
List the options for run.py:
```
cd roadshow-ansible
./run.py --help
```

Stand up an environment using the defaults. run.py will prompt for rhsm user, rhsm password and route53 hosted zone
```
./run.py
```

Stand up an environment without being prompted for confirmation and overriding
the cluster id, environment size, and keypair:
```
./run.py --no-confirm --cluster-id my-cluster --env-size medium --keypair my_keypair \
--r53-zone my.hosted.domain --rhsm-user my_redhat_user --rhsm-pass my_redhat_pass
```

After the run has completed the openshift web console will be available at
`https://openshift-master.<cluster id>.<r53 zone>` and routes created for
applications will default to `<app name>.<project name>.<cluster id>.<r53 zone>`

**Other run.py notes**
- Cluster ID for `run.py` cannot include a "\_", the OpenShift subdomain regex does not support this character. 
    ``` 
    --cluster-id my-cluster
    ```  

  - Details from the SUBDOMAIN_REGEXP from [origin git](https://github.com/openshift/origin/blob/master/assets/app/scripts/directives/oscKeyValues.js)
 
- Keypair is the name of your AWS private keypair without the  `.pem`  If your keypair is `my_keypair.pem`
    ```
    --keypair my_keypair
    ```

  - If you recall this was loaded into memory with `ssh-add`  

- Enable smoke tests users by adding additional parameters to the run.py
    ```
    --run-smoke-tests
    --num-smoke-test-users 5
    ```
  - Based on this [Roadshow](http://training.runcloudrun.com/roadshow/)


## Demo Notes (default)
- Demo User
    ```
    demo/openshift3
    ```

- Roadshow Users
    ```
    user00-userXX/openshift3
    ```
