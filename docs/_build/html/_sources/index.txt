AWS CloudFormation Challenge for MiniClip
=========================================

Hello, i thought it would be a nice idea to use the excelent readthedocs and
`sphinx <http://sphinx-doc.org/>`_ to write a small README
to go along with the challenge.

I hope this way, you'll have some fun reading it, and at the same time have no problems testing the solution
and be well informed about the choices i've made.


Introduction
------------

Every requirements have been met, so:

1) The template must be able to bring up the service in Frankfurt region without any modifications
   to it.

- Yes, you can even launch the template on diferent regions, i have used mappings for EC2 AMI instances
  for every regions.

2) Output of the template is one endpoint what can be used to access the service.

- Yes, the output is the AWS LB URL for the service, it listens to 8080 port, you can use it directly
  to access the service.

3) Autoscaling group must span across multiple AZ's.

- Yes, ASG supports multiple AZ's.

4) Service binary is deployed with the help of instructions in instance "UserData".

- Yes, I have given priority to AWS::CloudFormation::Init config keys, i personally think it's cleaner to
  use it this way. UserData embedded scripts are not well suited for changes (configuration management).
  I have used an install script inside a zip file on S3, were everything that can't be done using config keys is done.

5) Service binary is stored in S3 bucket.

- Yes, service binary is on miniclip.zip, previously downloaded to S3 service.

6) Service logs are rotated and pushed to CloudWatch.

- Yes, rotation of miniclip log is done with linux logrotate config. Logs are sent to Cloudwatch using awslogs service.

7) Secure the access. Only way to access the "service" from outside is via ELB.

- Yes, the access it secured, and extra parameters were added, so that access to LB service and SSH to the
  instances can be easily configured when deploying the stack withoud any modification. I just exposed the CIDRIP
  access. 

8) Autoscaling group (ASG) should launch new instance if inbound network traffic exceeds a X
   amount of bandwidth.

- Yes, used NetworkIn builtin CW metric, and exposed two thresholds parameters for ASG upscale and downscale.

9) A instance should be replaced in ASG if a service binary fails to serve or crashes.

- Yes, uses ELB Alarm for UnHealthyHostCount to span one more instance.

Extra Features
--------------

Some extra features were added for various reasons. The extra features are the following:

1 - Miniclip binary service is installed as a service on SysVInit, so an extra script was made to daemonize
the binary. This script is placed on **init.d** and installed as a service. This is a best practice and
as several benefits like:

a) The instance reboots, and the service starts without problems.
b) Start and Stop the service in a compliance and easy way.
c) Easy to monitor the service *service miniclip status*.

2 - Extra user is created named **miniclip**, service binaries are installed on the users homedir and run on it's name.

3 - An extra service and daemon script was devoloped to monitor the Miniclip service binary. This is also installed
on SysVInit. This is called *miniclip_monitor* and writes to a log file the state of the miniclip service binary.
This log file is sent to CloudWatch to a specific stream.

4 - The messages system log is sent to CloudWatch also. This seemed like an easy and useful feature to include.

5 - The stack supports rolling updates of stack changes or even for new code deploy. This made with two extra
parameters:

a) DEPLOY_VERSION - The deploiment version
b) DEPLOY_CONTEXT - The context: Production, Staging, Development.

The parameters values are set on a created file name env.sh (on ec2-user home),
this changes the stack UserData, and forces the instances to perform a rolling update and download
the service from S3 again. These could be used by the application to perform extra config.

Special Concerns
----------------

It was taken into concern the following aspects:

1 - **Security**:

a) No passwords, RSA keys or tokens are hardcoded on the stack or even created to be included on the
instances. The would have been easier to include an cloudFormation admin key on the instance to fullfill requirement
number 9.

b) Used IAM Roles to access CloudWatch and S3 on stack init.

c) Service binary runs with a non privileged user **miniclip**.

2 - **Simple dev best practices**:

a) The bash scripts do not have any hardcoded values on the code, this values are all set
as global vars at the top of the script.

b) The scripts are commented.

c) The JSON template follows the same principle.

d) The use of scripts inside UserData was set to a minimum, i personally think this is *cleaner* and better to
support future changes.

3 - **Organization, compliance, best practices and infrastructure management**.

Instalation
-----------

1 - **Upload miniclip.zip**:

To test the stack creation you must first upload the supplied miniclip.zip to an S3 Bucket. Do it on Frankfurt region,
you can use different regions as long as the URL for it complies with s3.<REGION>.amazonaws.com/<BUCKET>/<FILE>.
US regions do not comply with it (this could be overcomed using mapping on the template).

**The region for S3 must be the same as the region for the stack**.

**Take note of the bucket name and file. It will be used further for the stack parameters**.

So, the propused default S3 upload would be miniclip-deploy/miniclip.zip that would produce the following URL:
https://s3.eu-central-1.amazonaws.com/miniclip-deploy/miniclip.zip


2 - **Create the Stack**

Use the supplied miniclip.json to create the stack.

a) On AWS CloudFormation use Create Stack.
b) Choose 'Upload a template to Amazon S3' and select miniclip.json.
c) click 'Next'.


.. image:: ./images/create_stack.png
    :width: 50%
    :align: center


3 - **Set Stack Parameters**

a) Fill the stack parameters, use special care with DeployS3Bucket and DeployS3Pkg (use the values from step 1),
and suply your region KeyName (to access the instances). The rest of the parameters can be set to default. If you
have any problems/questions with it please send me an email.

b) click next.


.. image:: ./images/create_stack_parameters.png
    :width: 50%
    :align: center


4 - **Options**

a) click next.

5 - **Review**

a) Check the 'I acknowledge that this template might cause AWS CloudFormation to create IAM resources.'

b) click Create.


.. image:: ./images/create_stack_review.png
    :width: 50%
    :align: center


.. note:: The Stack was eavily tested on Frankfurt, and i do not expect any problems with it. If it rollsback please
   email me.


Package description
-------------------

Miniclip.zip file contains the following files:

- **miniclip_service.sh** : This is the binary service suplied by Miniclip.
- **miniclip** : The init.d script that starts, stops, and checks the status of the binary service.
- **miniclip_monitor.sh** : This script monitors miniclip service, writes it's state on /opt/miniclip/miniclip_monitor.log
- **miniclip_monitor** : The init.d script that starts, stops, and checks the status of the monitoring service.
- **install.sh** : This script installs all previous scripts.

Final Notes and Critics
-----------------------

There are some self critics i would like to point out:

1 - I'm not happy with the solution for requirement 9, using ELB UnHealthyHostCount metric is ok to span another
instance, but does not terminate the faulty instance by itself, this is done by the NetworkInAlarmLow.
This could be done with two possible approaches:

a) Using the service miniclip_monitor on the instance, to shutdown, or recover the instance. (shutdown is trivial,
but to trigger an instance recovery would need to setup Admin keys on the instance environment, i did not want to
do this for security reasons, although it's a common approach).

b) Using an extra Alarm from miniclip_monitor logs. (problem to associate the alarm with the instanceID and not the
ASG.



Indices and tables
==================

* :ref:`genindex`
* :ref:`search`

