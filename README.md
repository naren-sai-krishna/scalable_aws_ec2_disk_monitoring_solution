# scalable_aws_ec2_disk_monitoring_solution
Ansible-based solution to monitor EC2 disk usage across multiple AWS accounts using AWS SSM and centralized reporting.

## Background
The enterprise operates in a multi-cloud environment (AWS, Azure, GCP) due to organic growth and acquisitions. Each cloud provider has multiple accounts/subscriptions/projects with hundreds of virtual machines (VMs). Leadership has asked for a monitoring solution to proactively detect low disk space, **leveraging Ansible** (already in use) wherever possible and considering cloud-native options **only if they provide substantial benefits**.

This document proposes a **secure scalable disk usage monitoring solution** for **AWS**.

## Problem Statement
1. Multiple AWS accounts, each hosting EC2 instances.
2. Risk of downtime from disk exhaustion.
3. Need for **centralized monitoring** and early detection.
4. Leadership prefers **existing Ansible stack** unless cloud-native options provide significant benefits.

## Solution Approaches Considered

### **1. SSH-based Monitoring**

* **Approach**: Use Ansible from a central control node to SSH into managed EC2 instances across multiple AWS accounts to collect disk usage metrics.

* **Connectivity**:
  * Public Instances:
    * Can be accessed directly over the internet if SSH (port 22) is exposed, though this increases attack surface.
    * Requires strong key management and network restrictions (e.g., Security Group whitelisting, Bastion).

  * Private Instances:
    * **Not directly reachable** without additional networking.
    * Requires **VPC peering, Transit Gateway, or VPN** connectivity to establish SSH sessions → **adds cost and operational overhead**.
    * Scaling this across accounts and VPCs is complex.
 
* **Advantages**:
  * Straightforward to implement if SSH access already exist to public EC2 instances.

* **Challenges**:
  * **Cannot scale efficiently** for hundreds of instances across multiple accounts and regions.
  * **Cross-account bastion hosts not natively supported**, requiring complex networking setup for private instance monitoring.
  * Requires **inbound SSH (port 22) access**, increasing the attack surface.
  * Involves SSH key distribution, rotation and management challenges at scale.

### **2. Systems manager (SSM) based Monitoring**

* **Approach**: Use **AWS Systems Manager** to remotely execute disk usage commands on instances and collect disk metrics — **no SSH access required**.

* **Connectivity**:
  * Public Instances:
    * Instances with internet access can communicate with SSM APIs directly.
    * No inbound ports required (unlike SSH).

  * Private Instances:
    * Communicate with SSM through **VPC Interface Endpoints** (`ssm`, `ssmmessages`, `ec2messages`).
    * Secure, does not require public IPs, NAT, peering, or TGW.

* **Advantages**:
  * Secure (no inbound SSH).
  * Scales across accounts and regions.
  * Lightweight — directly executes `df -h`.
  * Easier and secure IAM-based access control compared to key-based SSH.

* **Challenges**:
  * Requires **SSM Agent** on instances and an **IAM instance role** with SSM permissions.
  * However, **most modern EC2 AMIs ship with SSM Agent pre-installed**, so this is rarely a blocker.

### **3. CloudWatch Cross-Account Observability**

* **Approach**: Leverage **Amazon CloudWatch Cross-Account Observability** to centralize disk usage metrics from multiple AWS accounts into a monitoring account. CloudWatch collects metrics (via CloudWatch Agent or custom scripts) and surfaces them in real time with dashboards and alarms.

* **Connectivity**:
  * Fully managed and push-based — no inbound access, agents push metrics securely to CloudWatch.
  * Works seamlessly across AWS accounts and regions once the monitoring account is designated.
  * No need for bastions, VPC endpoints, or open ports.

* **Advantages**:
  * **Entirely managed** on the cloud side → minimal operational overhead.
  * **Push-based and real-time** → metrics flow continuously into CloudWatch.
  * **Secure** → uses IAM and service-to-service communication, no inbound ports or SSH keys.
  * **Scales automatically** across 100s of instances and accounts.
  * Provides **dashboards, alarms and logs natively** in CloudWatch.

* **Challenges**:
  * Requires installation/configuration of **CloudWatch Agent** or equivalent to publish disk metrics.
  * If the enterprise chooses **monitoring via Ansible** for cloud-agnostic operations, CloudWatch’s **AWS-native nature does not align** with that strategy.
  * Limited flexibility if the same workflow needs to extend to **non-AWS environments**.

## Chosen Design:
After evaluating the three approaches, I recommend to adopt the Ansible + SSM-based monitoring design. Below are the reasons:

1. **Security**: Unlike SSH, SSM does not require inbound ports or key distribution. Private instances communicate with SSM through secure VPC Interface Endpoints, eliminating the need for NAT, VPC peering, or Transit Gateway.
2. **Scalability**: SSM scales seamlessly across hundreds of instances, multiple accounts and regions, while SSH requires complex networking and is difficult to scale.
3. **Operational Simplicity**: SSM allows Ansible playbooks to execute disk usage commands (`df -h`) directly without installing CloudWatch Agent or managing cross-account CloudWatch dashboards. This keeps the workflow aligned with the enterprise’s existing **Ansible-driven strategy**.
4. **Compatibility**: Most modern AMIs already include the SSM Agent, minimizing additional setup effort. IAM-based access control provides a more secure and manageable alternative to SSH keys.
5. **Cost Efficiency**: Avoids the additional infrastructure and cost overhead of VPC peering, Transit Gateway or CloudWatch agent management, while still centralizing execution and reporting.
6. **Dynamic Instance Discovery:** The central control node leverages the Ansible aws_ec2 dynamic inventory plugin to automatically discover EC2 instances across accounts/regions based on filters or tags (e.g., Environment=Prod, Monitor=True). This ensures that newly launched or terminated instances are automatically included or excluded in monitoring, reducing manual effort.

## **Workflow Overview**
1. **Cross-Account Access**
   * The central control node assumes **IAM roles** into target accounts to obtain inventory and execute commands.

2. **Instance Discovery**
   * Within each account, the **Ansible `aws_ec2` dynamic inventory plugin** discovers EC2 instances dynamically using tags/filters.
   * This avoids static host files and ensures the monitoring fleet reflects the live AWS environment.

3. **Execution via SSM**
   * For **public instances**, SSM communicates with AWS APIs over the internet.
   * For **private instances**, SSM communicates securely via **VPC Interface Endpoints** (`ssm`, `ssmmessages`, `ec2messages`).

4. **Command Execution**
   * Ansible triggers SSM `send-command` or SSM documents to run `df -h` on each discovered EC2 instance.

5. **Metrics Collection**
   * Ansible collects outputs centrally, aggregates disk usage reports and can push results to dashboards or alerting systems.

## High level architecture diagram
![Diagram](https://github.com/naren-sai-krishna/scalable_aws_ec2_disk_monitoring_solution/blob/main/architecture_diagram/disk_monitor.png)

## AWS Setup:

### 1. Management Account (Control Node Plane)

1. **Provision Control Node**

   * Launch an **EC2 instance** in the Management Account that will serve as the **central Ansible control node**.

2. **Create IAM Instance Profile Role for Control Node**

   * Role Name: `ControlNodeRole`
   * **Trust Policy** (allow EC2 to assume this role):

     ```json
     {
       "Version": "2012-10-17",
       "Statement": [
         {
           "Effect": "Allow",
           "Principal": { "Service": "ec2.amazonaws.com" },
           "Action": "sts:AssumeRole"
         }
       ]
     }
     ```
   * **Permissions Policy** (allow assuming member account roles):

     ```json
     {
       "Version": "2012-10-17",
       "Statement": [
         {
           "Sid": "AllowAssumeMemberRoles",
           "Effect": "Allow",
           "Action": "sts:AssumeRole",
           "Resource": "arn:aws:iam::*:role/ManagedNodeRole"
         }
       ]
     }
     ```

3. **Attach the Role to Control Node**

   * Associate `ControlNodeRole` as the instance profile for the control node EC2.

4. **Configure Control Node**

   * SSH into the instance using your private key.
   * Install Ansible, boto3 and other required packages.
   * Clone the Ansible repository that contains the **inventory and playbook files**. Create or update the inventory files with your account information.

### 2. Member Accounts (Managed Nodes & Inventory Discovery)

Each member account requires **two roles**:

#### a. Managed Node Role (EC2 → SSM)

1. Create IAM role `RoleForSSM`.

   * **Trust Policy**:

     ```json
     {
       "Version": "2012-10-17",
       "Statement": [
         {
           "Effect": "Allow",
           "Principal": { "Service": "ec2.amazonaws.com" },
           "Action": "sts:AssumeRole"
         }
       ]
     }
     ```
   * **Permissions Policy**: Attach AWS managed policy `AmazonSSMManagedInstanceCore`.

2. **Associate Role with Managed Nodes**

   * While launching EC2 instances, assign the **IAM instance profile** `RoleForSSM`.
   * For existing EC2 instances, update the IAM role via **EC2 → Actions → Security → Modify IAM role**.
   * This ensures the EC2 instances can:

     * Register with **AWS Systems Manager**.
     * Respond to SSM SendCommand from the control node.

---

#### b. Discovery & Remote Execution Role (for Control Node Access)

1. Create IAM role `ManagedNodeRole`.

   * **Trust Policy** (allow the control node in Management Account to assume this role):

     ```json
     {
       "Version": "2012-10-17",
       "Statement": [
         {
           "Effect": "Allow",
           "Principal": { "AWS": "arn:aws:iam::Account-A:root" },
           "Action": "sts:AssumeRole"
         }
       ]
     }
     ```
   * **Permissions Policy** (allow SSM command execution and output retrieval):

     ```json
     {
       "Version": "2012-10-17",
       "Statement": [
         {
           "Sid": "AllowSSMCommands",
           "Effect": "Allow",
           "Action": [
             "ssm:SendCommand",
             "ssm:GetCommandInvocation"
           ],
           "Resource": "*"
         }
       ]
     }
     ```
   * **Permissions Policy** (allow EC2 inventory discovery):

     ```json
     {
       "Version": "2012-10-17",
       "Statement": [
         {
           "Effect": "Allow",
           "Action": [
             "ec2:DescribeInstances",
             "iam:ListInstanceProfiles",
             "iam:GetInstanceProfile"
           ],
           "Resource": "*"
         }
       ]
     }
     ```

---
## Ansible Setup

### 1. Inventory Configuration

For each member account, create a dedicated inventory file under the `/inventory` folder (e.g., `/inventory/account-b.yml`) using the **`amazon.aws.aws_ec2`** dynamic inventory plugin:

```yaml
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1
assume_role_arn: arn:aws:iam::<Account-B>:role/ManagedNodeRole
filters:
  instance-state-name: running
  tag: Environment: Dev
keyed_groups:
  - prefix: account
    key: owner_id
compose:
  ansible_aws_ssm_instance_id: instance_id
  account_id: owner_id
  ansible_aws_ssm_role_arn: "'arn:aws:iam::<Account-B>:role/ManagedNodeRole'"
  ansible_aws_ssm_region: placement.region
```

> **Note:** In this example, only instances tagged with `Environment: Dev` are monitored.
> Replace `<Account-B>` with the actual AWS Account ID.

---

### 2. Ansible Configuration

Create or update `ansible.cfg` in the project root to point to the `/inventory` directory:

```ini
[inventory]
enable_plugins = amazon.aws.aws_ec2

[defaults]
inventory = ./inventory/       # All account inventories should be placed here
host_key_checking = False
remote_user = ssm-user
gathering = smart
interpreter_python = auto
```

---

### 3. Running the Playbook

Execute the disk monitoring playbook:

```bash
ansible-playbook disk_report.yml
```

* Results will be generated in `disk_usage_report.html`.
* This report provides a consolidated view of disk utilization across all monitored instances.

---

### 4. (Optional) Email Integration

To automatically send the report via email, extend the playbook with the Ansible `mail` module:

```yaml
- name: Send disk usage report via email
  ansible.builtin.mail:
    host: smtp.example.com
    port: 587
    username: your-username
    password: your-password
    to: ops-team@example.com
    subject: "Disk Usage Report"
    body: "Please find the attached disk usage report."
    attach:
      - disk_usage_report.html
```

This ensures the **operations team receives the disk usage report in their inbox** after every run.

## End-to-End Flow

1. Control Node (Management Account EC2 with `ControlNodeRole`) uses **STS AssumeRole** into `ManagedNodeRole` in each member account.
2. Control Node fetches **EC2 inventory dynamically** (via `aws_ec2` plugin in Ansible).
3. Control Node sends **SSM SendCommand** (e.g., `df -h`) to all targeted EC2s.
4. EC2 instances (with `RoleForSSM` attached) execute the command via SSM Agent and return the output.
5. Ansible collects and aggregates disk usage results.


## Conclusion

This solution ensures **secure, scalable and automated disk usage monitoring** across multiple AWS accounts, while staying aligned with the enterprise’s Ansible-first strategy.

* **Security**: By leveraging AWS Systems Manager (SSM) instead of SSH, no inbound ports or key distribution are required. All communication happens over secure, IAM-controlled channels. EC2 instances only need the `AmazonSSMManagedInstanceCore` policy, minimizing the attack surface while ensuring least-privilege access.

* **Scalability**: Dynamic inventory discovery using the Ansible `aws_ec2` plugin ensures that instances are automatically included or excluded based on tags and filters. This eliminates manual host file management and scales seamlessly across hundreds of instances, multiple accounts and regions.

* **Automation of AWS Setup**: While the IAM roles and trust relationships can be configured manually, they can also be fully **automated using AWS CloudFormation StackSets**. With StackSets, the required IAM roles (`RoleForSSM` on EC2 instances and `ManagedNodeRole` for cross-account execution) can be rolled out to all member accounts consistently, reducing manual overhead and configuration drift.

* **Extensibility**: Beyond generating `disk_usage_report.html`, the workflow can be extended to send automated emails, feed into dashboards, or trigger alerts when thresholds are breached — ensuring proactive monitoring and faster response.

In summary, this design balances **security (IAM + SSM)**, **scalability (dynamic inventory discovery)** and **operational efficiency (CloudFormation StackSets for automation)**, making it a future-proof solution for enterprise-scale disk monitoring on AWS.

---
## Results:

### Dynamic Inventory Discovery across multiple accounts:
```
[ec2-user@ip-172-31-XX-XX test-ansible]$ ansible-inventory --graph
@all:
  |--@ungrouped:
  |--@aws_ec2:
  |  |--ec2-35-173-XX-XX.compute-1.amazonaws.com
  |  |--ec2-54-234-XX-XX.compute-1.amazonaws.com
  |  |--ip-10-1-XX-XX.ec2.internal
  |--@account_0129XXXXXXXX:
  |  |--ec2-35-173-XX-XX.compute-1.amazonaws.com
  |  |--ec2-54-234-XX-XX.compute-1.amazonaws.com
  |--@account_2001XXXXXX:
  |  |--ip-10-1-XX-XX.ec2.internal
```

### Collecting metrics from instances in multiple accounts and report generation
```
[ec2-user@ip-172-31-XX-XX test-ansible]$ ansible-playbook disk_report.yml
PLAY [all] **********************************************************************************************************************************************************************************************
TASK [Ensure metrics directory exists] ******************************************************************************************************************************************************************
changed: [ec2-35-173-XX-XX.compute-1.amazonaws.com -> localhost]
TASK [Set AWS account ID] *******************************************************************************************************************************************************************************
ok: [ec2-35-173-XX-XX.compute-1.amazonaws.com]
ok: [ec2-54-234-XX-XX.compute-1.amazonaws.com]
ok: [ip-10-1-XX-XX.ec2.internal]
TASK [Assume target account role] ***********************************************************************************************************************************************************************
ok: [ec2-54-234-XX-XX.compute-1.amazonaws.com]
ok: [ec2-35-173-XX-XX.compute-1.amazonaws.com]
ok: [ip-10-1-XX-XX.ec2.internal]
TASK [Execute disk usage check] *************************************************************************************************************************************************************************
ok: [ec2-54-234-XX-XX.compute-1.amazonaws.com]
ok: [ec2-35-173-XX-XX.compute-1.amazonaws.com]
ok: [ip-10-1-XX-XX.ec2.internal]
TASK [Retrieve command output] **************************************************************************************************************************************************************************
ok: [ec2-35-173-XX-XX.compute-1.amazonaws.com]
ok: [ec2-54-234-XX-XX.compute-1.amazonaws.com]
ok: [ip-10-1-XX-XX.ec2.internal]
TASK [Set default disk metrics] *************************************************************************************************************************************************************************
ok: [ec2-35-173-XX-XX.compute-1.amazonaws.com]
ok: [ec2-54-234-XX-XX.compute-1.amazonaws.com]
ok: [ip-10-1-XX-XX.ec2.internal]
TASK [Parse disk usage data] ****************************************************************************************************************************************************************************
ok: [ec2-35-173-XX-XX.compute-1.amazonaws.com]
ok: [ec2-54-234-XX-XX.compute-1.amazonaws.com]
ok: [ip-10-1-XX-XX.ec2.internal]
TASK [Save instance metrics] ****************************************************************************************************************************************************************************
changed: [ip-10-1-XX-XXec2.internal -> localhost]
changed: [ec2-35-173-XX-XX.compute-1.amazonaws.com -> localhost]
changed: [ec2-54-234-XX-XX.compute-1.amazonaws.com -> localhost]
TASK [Aggregate all metrics] ****************************************************************************************************************************************************************************
ok: [ec2-35-173-XX-XX.compute-1.amazonaws.com -> localhost]
TASK [Generate HTML report] *****************************************************************************************************************************************************************************
changed: [ec2-35-173-XX-XX.compute-1.amazonaws.com -> localhost]
TASK [Cleanup old metrics] ******************************************************************************************************************************************************************************
changed: [ec2-35-173-XX-XX.compute-1.amazonaws.com -> localhost]
TASK [Display execution summary] ************************************************************************************************************************************************************************
ok: [ec2-35-173-XX-XX.compute-1.amazonaws.com] => {
    "msg": "Disk Usage Check Summary:\n- Account: 0129XXXXXX\n- Instance: i-00184639ea30XXXX\n- Status: Success\n- Usage: 30%\n"
}
ok: [ec2-54-234-XX-XX.compute-1.amazonaws.com] => {
    "msg": "Disk Usage Check Summary:\n- Account: 01294XXXXXX\n- Instance: i-0d16aa3bd5faXXXX\n- Status: Success\n- Usage: 30%\n"
}
ok: [ip-10-1-XX-XX.ec2.internal] => {
    "msg": "Disk Usage Check Summary:\n- Account: 20011XXXXX\n- Instance: i-0b912fe4d782XXXX\n- Status: Success\n- Usage: 19%\n"
}
PLAY RECAP **********************************************************************************************************************************************************************************************
ec2-35-173XX-XX.compute-1.amazonaws.com : ok=12   changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
ec2-54-234-XX-XX.compute-1.amazonaws.com : ok=8    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
ip-10-1-XX-XX.ec2.internal : ok=8    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0 
```
### Report:
![Report](https://github.com/naren-sai-krishna/scalable_aws_ec2_disk_monitoring_solution/blob/main/misc/Report.png)
