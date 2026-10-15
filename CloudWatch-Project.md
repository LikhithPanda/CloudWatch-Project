### **CloudWatch Monitoring for Linux and Windows VMs**



**Goal:** monitor two EC2 instances (one Linux, one Windows) with Amazon CloudWatch. We’ll collect default metrics, add memory and disk metrics plus logs with the CloudWatch agent, build a dashboard, and set up an alarm with email notification.



###### **Part A: Prepare**

* Use one region (e.g. Mumbai ap-south-1) for everything.
* Create an IAM role for the instances:

  * IAM → Roles → Create role → AWS service → EC2 (Select EC2 under use case).
  * Attach CloudWatchAgentServerPolicy (and AmazonSSMManagedInstanceCore if you want Systems Manager access).
  * Name it CloudWatchAgentRole.
* Create an SNS (Simple Notification Service) topic for alerts: SNS → Topics → Create (Standard) → cw-alerts → Create subscription → Email → your address → confirm the email you receive.



###### **Part B: Launch both VMs**

|**Setting**|**Linux VM**|**Windows VM**|
|-|-|-|
|Name|linux-monitor|windows-monitor|
|AMI|Amazon Linux 2023|Microsoft Windows Server 2025 Base|
|Instance type|t2.micro|t3.micro|
|Key pair|.ppk file|.ppk file|
|Security group|SSH 22 from My IP|RDP 3389 from My IP|
|IAM instance profile|CloudWatchAgentRole|CloudWatchAgentRole|
|Monitoring|Enable detailed monitoring (Advanced details)|Enable detailed monitoring|



Wait for 2/2 status checks passed on linux instance.

Wait for 3/3 status checks passed on windows instance.





###### **Part C: Check the default (basic) monitoring**

* EC2 → Instances → select an instance → Monitoring tab.
* Screenshot the graphs: CPU utilization, Network in/out, Disk read/write ops, Status check failed.
* Open CloudWatch → Metrics → All metrics → EC2 → Per-Instance Metrics and compare both machines.



Note : EC2 does not report memory or disk-space usage by default. That is why the agent is installed next.



###### **Part D: Install the CloudWatch agent**



**Linux (connect via PuTTY):**



bash

\# Amazon Linux

sudo yum install -y amazon-cloudwatch-agent



\# Ubuntu

wget https://amazoncloudwatch-agent.s3.amazonaws.com/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb

sudo dpkg -i -E ./amazon-cloudwatch-agent.deb



Create a minimal config at /opt/aws/amazon-cloudwatch-agent/bin/config.json (or run the wizard sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard):



json

{

&#x20; "metrics": {

&#x20;   "metrics\_collected": {

&#x20;     "mem":  { "measurement": \["mem\_used\_percent"] },

&#x20;     "disk": { "measurement": \["used\_percent"], "resources": \["/"] }

&#x20;   },

&#x20;   "append\_dimensions": { "InstanceId": "${aws:InstanceId}" }

&#x20; },

&#x20; "logs": {

&#x20;   "logs\_collected": {

&#x20;     "files": {

&#x20;       "collect\_list": \[

&#x20;         { "file\_path": "/var/log/syslog", "log\_group\_name": "linux-syslog", "log\_stream\_name": "{instance\_id}" }

&#x20;       ]

&#x20;     }

&#x20;   }

&#x20; }

}



(On Amazon Linux use /var/log/messages instead of syslog.) Start the agent:



bash

sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \\

&#x20; -a fetch-config -m ec2 \\

&#x20; -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json -s

sudo systemctl status amazon-cloudwatch-agent





**Windows** (connect via RDP: EC2 → Connect → RDP client → decrypt the password with your .ppk; log in as Administrator):



* Download and run the MSI in PowerShell:

&#x09;Invoke-WebRequest https://amazoncloudwatch-agent.s3.amazonaws.com/windows/amd64/latest/amazon-cloudwatch-agent.msi -OutFile agent.msi

&#x09;msiexec /i agent.msi /qn

* Run the config wizard from C:\\Program Files\\Amazon\\AmazonCloudWatchAgent\\ (amazon-cloudwatch-agent-config-wizard.exe), choose Windows, enable memory, disk and Windows Event Logs (System, Application), and save the config.
* Start the agent:

&#x09;\& "C:\\Program Files\\Amazon\\AmazonCloudWatchAgent\\amazon-cloudwatch-agent-ctl.ps1" -a fetch-config -m ec2 -c file:"C:\\Program Files\\Amazon\\AmazonCloudWatchAgent\\config.json" -s

* Check: Get-Service AmazonCloudWatchAgent should show Running.





###### **Part E: Verify the new metrics and logs**

* CloudWatch → Metrics → All metrics → CWAgent: you should see memory and disk metrics for both instances (a few minutes after starting).
* CloudWatch → Logs → Log groups: open the Linux syslog group and the Windows event log groups, and view the log streams.





###### **Part F: Create a dashboard**

* CloudWatch → Dashboards → Create dashboard → vm-monitoring.
* Add line widgets for: CPUUtilization (both instances), mem\_used\_percent, disk used\_percent, NetworkIn/Out.
* Add a Logs table widget for recent log entries. 





###### **Part G: Create alarms**

* CloudWatch → Alarms → Create alarm → Select metric → EC2 → CPUUtilization → choose the Linux instance.
* Condition: Greater than 70%, 1 datapoint for 1 period (1 or 5 min).
* Action: In alarm → send to SNS topic cw-alerts. Name it linux-high-cpu.
* Repeat for the Windows instance (windows-high-cpu). Optionally add alarms for memory and for StatusCheckFailed.





###### **Part H: Test the monitoring**

**Linux**: generate CPU load.

bash

&#x09;sudo yum install -y stress-ng        # for Ubuntu sudo apt install -y stress-ng

&#x09;stress-ng --cpu 2 --timeout 300s



**Windows**: run a CPU loop in PowerShell for a few minutes.

&#x09;1..2 | ForEach-Object { Start-Job { while($true){ \[math]::Sqrt(12345) } } }

&#x09;# stop later: Get-Job | Stop-Job

* Watch the dashboard spike, wait for the alarm to go In alarm, and confirm the email arrives. Screenshot the alarm history and the email.





###### **Part I: Cleanup**

Stop the load, delete the alarms and dashboard, delete the log groups, terminate both instances, and delete the SNS topic and IAM role if no longer needed.

