# How to Hack Like a Ghost — Table of Contents

*Breaching the Cloud — By Sparc Flow (No Starch Press, 2021)*

## Introduction

- How the Book Works
- The Vague Plan

---

## Part I: Catch Me If You Can

### Chapter 1: Becoming Anonymous Online

- VPNs and Their Failings
- Location, Location, Location
- The Operation Laptop
- Bouncing Servers
- The Attack Infrastructure
- Resources

### Chapter 2: Return of Command and Control

- Command and Control Legacy
- The Search for a New C2
- Merlin
- Koadic
- SILENTTRINITY
- Resources

### Chapter 3: Let There Be Infrastructure

- Legacy Method
- Containers and Virtualization
  - Namespaces
  - Union Filesystem
  - Cgroups
- IP Masquerading
- Automating the Server Setup
- Tuning the Server
- Pushing to Production
- Resources

---

## Part II: Try Harder

### Chapter 4: Healthy Stalking

- Understanding Gretsch Politico
- Finding Hidden Relationships
- Scouring GitHub
- Pulling Web Domains
  - From Certificates
  - By Harvesting the Internet
- Discovering the Web Infrastructure Used
- Resources

### Chapter 5: Vulnerability Seeking

- Practice Makes Perfect
- Revealing Hidden Domains
- Investigating the S3 URLs
- S3 Bucket Security
- Examining the Buckets
- Inspecting the Web-Facing Application
- Interception with WebSocket
- Server-Side Request Forgery
- Exploring the Metadata
- The Dirty Secret of the Metadata API
- AWS IAM
- Examining the Key List
- Resources

---

## Part III: Total Immersion

### Chapter 6: Fracture

- Server-Side Template Injection
- Fingerprinting the Framework
- Arbitrary Code Execution
- Confirming the Owner
- Smuggling Buckets
- Quality Backdoor Using S3
  - Creating the Agent
  - Creating the Operator
- Trying to Break Free
  - Checking for Privileged Mode
  - Linux Capabilities
  - Docker Socket
- Resources

### Chapter 7: Behind the Curtain

- Kubernetes Overview
- Introducing Pods
- Balancing Traffic
- Opening the App to the World
- Kube Under the Hood
- Resources

### Chapter 8: Shawshank Redemption: Breaking Out

- RBAC in Kube
- Recon 2.0
- Breaking Into Datastores
- API Exploration
- Abusing the IAM Role Privileges
- Abusing the Service Account Privileges
- Infiltrating the Database
- Redis and Real-Time Bidding
- Deserialization
- Cache Poisoning
- Kube Privilege Escalation
- Resources

### Chapter 9: Sticky Shell

- Stable Access
- The Stealthy Backdoor
- Resources

---

## Part IV: The Enemy Inside

### Chapter 10: The Enemy Inside

- The Path to Apotheosis
- Automation Tool Takeover
- Jenkins Almighty
- Hell's Kitchen
- Taking Over Lambda
- Resources

### Chapter 11: Nevertheless, We Persisted

- The AWS Sentries
- Persisting in the Utmost Secrecy
- The Program to Execute
- Building the Lambda
- Setting Up the Trigger Event
- Covering Our Tracks
- Recovering Access
- Alternative (Worse) Methods
- Resources

### Chapter 12: Apotheosis

- Persisting the Access
- Understanding Spark
- Malicious Spark
- Spark Takeover
- Finding Raw Data
- Stealing Processed Data
- Privilege Escalation
- Infiltrating Redshift
- Resources

### Chapter 13: Final Cut

- Hacking Google Workspace
- Abusing CloudTrail
- Creating a Google Workspace Super Admin Account
- Sneaking a Peek
- Closing Thoughts
- Resources

---

## Appendix: Tools and Resources

## Index
