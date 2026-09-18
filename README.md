# AWS Identity and Access Management (IAM)

## Users, Groups, Policies & MFA — Practice Guide

### 1. Introduction to IAM

**AWS Identity and Access Management (IAM)** controls **who can access AWS** and **what actions they are allowed to perform**.

IAM is used for:

* Authentication — **Who are you?**
* Authorization — **What are you allowed to do?**
* Users and groups
* Roles
* Permission policies
* MFA (Multi-Factor Authentication)
* Least-privilege access

### 2. Architecture Diagram

```text
                         AWS ACCOUNT
                             |
             +---------------+---------------+
             |                               |
        ROOT USER                         AWS IAM
        (Account Owner)                      |
        MFA Required                         |
                                             |
                    +------------------------+----------------------+
                    |                        |                      |
                 IAM USERS               IAM GROUPS             IAM ROLES
                    |                        |                      |
              +-----+-----+             +----+-----+          Temporary
              |           |             |          |          Credentials
            Alice        Bob         Developers  Admins
              |           |             |          |
              +-----------+-------------+----------+
                                  |
                              IAM POLICIES
                                  |
                    +-------------+-------------+
                    |                           |
              AWS Managed                  Customer Managed
                 Policies                     Policies
                    |                           |
                    +-------------+-------------+
                                  |
                            AWS RESOURCES
                                  |
                  +---------------+---------------+
                  |               |               |
                 EC2              S3              RDS
```

**Simple flow:**

```text
User
  |
  v
Authentication
Password + MFA
  |
  v
IAM User / Role
  |
  v
IAM Policy Evaluation
  |
  +---- Allow ----> AWS Resource
  |
  +---- Deny -----> Access Denied
```

---

# 3. Types of Users in AWS

For basic IAM learning, understand these identities:

**Root User**

* Created when the AWS account is created.
* Has complete access to the AWS account.
* Should not be used for everyday administrative work.
* Enable MFA on the root user.
* Do not create root access keys unless there is a specific requirement.

**IAM User**

* Identity created inside an AWS account.
* Can represent a person or, for legacy/specific use cases, an application.
* Can have console access and/or access keys.
* Permissions are controlled using policies.

**Federated User**

* User authenticates through an external identity provider.
* Commonly used with AWS IAM Identity Center or enterprise identity providers.
* Usually receives temporary credentials rather than long-lived IAM-user credentials.

> For workforce access, AWS generally recommends federation/IAM Identity Center rather than creating large numbers of long-lived IAM users.

---

# 4. IAM Groups

An **IAM Group** is a collection of IAM users.

Example:

```text
Developers Group
     |
     +-- developer1
     +-- developer2
     +-- developer3
     |
     +-- AmazonEC2ReadOnlyAccess
```

Instead of assigning the same policy individually:

```text
User1 -> Policy
User2 -> Policy
User3 -> Policy
```

assign it to a group:

```text
              Policy
                 |
           Developers
          /     |     \
      User1   User2   User3
```

---

# 5. Creating IAM Users and Groups — Console Steps

### Create a Group

```text
AWS Console
   ↓
IAM
   ↓
User groups
   ↓
Create group
   ↓
Enter Group Name
Example: Developers
   ↓
Attach required policy
   ↓
Create group
```

### Create an IAM User

```text
IAM
   ↓
Users
   ↓
Create user
   ↓
Enter username
Example: developer1
   ↓
Configure required access
   ↓
Add user to Developers group
   ↓
Create user
```

Avoid giving `AdministratorAccess` simply for convenience. Use only the permissions required for the lab or job function.

---

# 6. AWS CLI — IAM Users and Groups

Create a group:

```bash
aws iam create-group \
  --group-name Developers
```

Create a user:

```bash
aws iam create-user \
  --user-name developer1
```

Add the user to the group:

```bash
aws iam add-user-to-group \
  --user-name developer1 \
  --group-name Developers
```

Check users:

```bash
aws iam list-users
```

Check groups:

```bash
aws iam list-groups
```

Check users inside the group:

```bash
aws iam get-group \
  --group-name Developers
```

---

# 7. Types of IAM Policies

For practical learning, focus on:

**AWS Managed Policies**

Created and maintained by AWS.

Examples:

```text
ReadOnlyAccess
AmazonS3ReadOnlyAccess
AmazonEC2ReadOnlyAccess
```

**Customer Managed Policies**

Created and maintained by **you** inside your AWS account.

Example:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListAllMyBuckets",
        "s3:GetBucketLocation"
      ],
      "Resource": "*"
    }
  ]
}
```

**Inline Policies**

Embedded directly into a single IAM user, group, or role.

```text
IAM User
   |
   +---- Inline Policy
```

Managed policies are generally easier to reuse and centrally maintain than inline policies.

---

# 8. Assign Policy to a Group

Example: give the Developers group EC2 read-only access.

Find the policy:

```bash
aws iam list-policies \
  --scope AWS
```

Attach:

```bash
aws iam attach-group-policy \
  --group-name Developers \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ReadOnlyAccess
```

Verify:

```bash
aws iam list-attached-group-policies \
  --group-name Developers
```

Architecture:

```text
AmazonEC2ReadOnlyAccess
          |
          v
     Developers
          |
    +-----+-----+
    |           |
developer1   developer2
    |
    v
EC2 Read Operations
```

---

# 9. Assign Policy Directly to User

Example:

```bash
aws iam attach-user-policy \
  --user-name developer1 \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

Verify:

```bash
aws iam list-attached-user-policies \
  --user-name developer1
```

For multiple users with the same job function, prefer:

```text
Policy -> Group -> Users
```

rather than repeatedly attaching the same policy:

```text
Policy -> User1
Policy -> User2
Policy -> User3
```

---

# 10. Create a Customer Managed Policy

Create:

```bash
nano s3-read-policy.json
```

Add:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListAllMyBuckets",
        "s3:GetBucketLocation"
      ],
      "Resource": "*"
    }
  ]
}
```

Create the policy:

```bash
aws iam create-policy \
  --policy-name CustomS3ListPolicy \
  --policy-document file://s3-read-policy.json
```

Then attach the returned policy ARN to the required user, group, or role.

---

# 11. MFA — Multi-Factor Authentication

MFA adds another authentication factor.

```text
Username
    +
Password
    +
MFA Code
    |
    v
AWS Login
```

Common setup:

```text
IAM User
   |
Password
   +
Authenticator
   |
   v
AWS Console
```

### Enable MFA for IAM User

```text
IAM
 ↓
Users
 ↓
Select User
 ↓
Security credentials
 ↓
Multi-factor authentication (MFA)
 ↓
Assign MFA device
 ↓
Choose MFA method
 ↓
Scan QR / configure device
 ↓
Enter required MFA codes
 ↓
Add MFA
```

### Enable MFA for Root User

```text
Sign in as Root User
 ↓
Account / Security credentials
 ↓
Multi-factor authentication
 ↓
Assign MFA device
 ↓
Configure authenticator/security key
 ↓
Verify
```

Root-user MFA should be treated as a basic AWS account security requirement.

---

# 12. Deleting IAM Users and Groups

Before deleting a user, remove associated resources such as group memberships, policies, access keys, login profile, MFA devices, and other credentials as applicable.

Remove user from group:

```bash
aws iam remove-user-from-group \
  --user-name developer1 \
  --group-name Developers
```

Detach user policy:

```bash
aws iam detach-user-policy \
  --user-name developer1 \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

Delete user:

```bash
aws iam delete-user \
  --user-name developer1
```

Detach group policy:

```bash
aws iam detach-group-policy \
  --group-name Developers \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ReadOnlyAccess
```

Delete group:

```bash
aws iam delete-group \
  --group-name Developers
```

---

# 13. Basic Hands-On Lab

Use this sequence for classroom practice:

```text
STEP 1
Login to AWS

STEP 2
Open IAM

STEP 3
Create Group
Name: Developers

STEP 4
Attach:
AmazonEC2ReadOnlyAccess

STEP 5
Create IAM User
Name: developer1

STEP 6
Add developer1 to Developers

STEP 7
Login/Test as developer1

STEP 8
Verify:
EC2 Read = Allowed
EC2 modification = Not allowed

STEP 9
Create Customer Managed Policy

STEP 10
Attach policy and test permissions

STEP 11
Configure MFA for IAM user

STEP 12
Verify MFA login

STEP 13
Verify MFA on Root User

STEP 14
Remove policies/group memberships

STEP 15
Delete lab IAM user and group
```

---

# 14. Points to Remember

* **IAM is a global AWS service**; IAM resources such as users are not created separately per AWS Region.
* **Root user** has account-level privileges and should be protected carefully.
* Enable **MFA on the root user**.
* Avoid using the root user for daily work.
* Use **least privilege** — grant only permissions that are required.
* **IAM User = individual identity.**
* **IAM Group = collection of IAM users.**
* **IAM Role = assumable identity using temporary credentials.**
* **IAM Policy = permissions document.**
* Groups contain **users**, not other groups.
* A user can belong to multiple groups.
* Policies can contain `Allow` and `Deny`.
* An **explicit Deny overrides an Allow** during policy evaluation.
* Prefer managed policies over inline policies when permissions need to be reused.
* Avoid long-lived access keys where temporary credentials or roles can be used.
* Never put AWS access keys in GitHub repositories, source code, screenshots, or shared documents.
* Rotate/remove unnecessary credentials and permissions.
* For organizations and workforce access, consider **AWS IAM Identity Center** rather than managing many individual IAM users.
* Regularly review permissions and unused access.

### Quick Memory

```text
USER   = WHO
GROUP  = COLLECTION OF USERS
ROLE   = TEMPORARY/ASSUMABLE IDENTITY
POLICY = WHAT IS ALLOWED/DENIED
MFA    = EXTRA AUTHENTICATION FACTOR
```

### Complete Architecture

```text
                       AWS ACCOUNT
                           |
            +--------------+--------------+
            |                             |
        Root User                         IAM
       + MFA                              |
                                          |
                    +---------------------+--------------------+
                    |                     |                    |
                  Users                 Groups               Roles
                    |                     |                    |
                 + MFA               Users grouped       Temporary
                    |                by job function      credentials
                    |                     |                    |
                    +---------------------+--------------------+
                                          |
                                      POLICIES
                                          |
                      +-------------------+-------------------+
                      |                   |                   |
                  AWS Managed       Customer Managed       Inline
                      |                   |                   |
                      +-------------------+-------------------+
                                          |
                                  Policy Evaluation
                                          |
                              +-----------+-----------+
                              |                       |
                            Allow                  Deny
                              |                       |
                              v                       X
                       AWS Resources            Access Denied
                    EC2 / S3 / RDS / etc.
```
