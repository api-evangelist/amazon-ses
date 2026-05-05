---
title: "Implement Tenants in your Amazon SES environment, Part 3: Implementation guide"
url: "https://aws.amazon.com/blogs/messaging-and-targeting/implement-tenants-in-your-amazon-ses-environment-part-3-implementation-guide/"
date: "Wed, 26 Nov 2025 19:18:56 +0000"
author: "Rommel Sunga"
feed_url: "https://aws.amazon.com/blogs/messaging-and-targeting/feed/"
---
<p>This is part 3 in a series covering <a href="https://docs.aws.amazon.com/ses/latest/dg/tenants.html" rel="noopener noreferrer" target="_blank">the new tenants feature</a> in <a href="https://aws.amazon.com/ses/" rel="noopener noreferrer" target="_blank">Amazon Simple Email Service (SES)</a>. The first post in this series discussed how users can <a href="https://aws.amazon.com/blogs/messaging-and-targeting/improve-email-deliverability-with-tenant-management-in-amazon-ses/" rel="noopener noreferrer" target="_blank">improve email deliverability with tenant management in Amazon SES</a>. <a href="https://aws.amazon.com/blogs/messaging-and-targeting/implement-tenants-in-your-amazon-ses-environment-part-2-assessment-and-planning/" rel="noopener" target="_blank">Part 2</a> covered key aspects involved in planning the migration of existing Amazon SES infrastructure to use tenant-based reputation isolation.</p> 
<p>With Amazon SES tenants, users can:</p> 
<ul> 
 <li>Manage individual tenant onboardings and their reputations in isolation</li> 
 <li>Provision isolated tenants within a single SES account</li> 
 <li>Apply automated reputation policies to manage email sending</li> 
 <li>Detect and isolate deliverability issues within isolated email streams</li> 
 <li>Preserve sender reputation and improve inbox placement with mailbox providers</li> 
</ul> 
<p>This post provides a step-by-step migration guide to Tenants for key AWS components like <a href="https://aws.amazon.com/iam/" rel="noopener noreferrer" target="_blank">AWS Identity and Access Management (IAM)</a> permissions, <a href="https://aws.amazon.com/cloudwatch/" rel="noopener noreferrer" target="_blank">Amazon CloudWatch</a> logging and <a href="https://aws.amazon.com/eventbridge/" rel="noopener noreferrer" target="_blank">Amazon EventBridge</a> monitoring. Additionally, we provide several code examples that show how to programmatically provision tenants in real time as customers are onboarded. The goal is to demonstrate how to use the Tenants feature to achieve reputation isolation between customers or business units (BUs), get more control over sending policies, and enable the automatic pause mechanism to limit the damage from problematic senders.</p> 
<h2>Step-by-step migration guide</h2> 
<p>Having completed the inventory and configuration planning prescribed in <a href="https://aws.amazon.com/blogs/messaging-and-targeting/implement-tenants-in-your-amazon-ses-environment-part-2-assessment-and-planning/" rel="noopener" target="_blank">part 2</a> of this series, we are ready to start the 4-step migration (or implementation) of the tenants feature in the AWS SES account.</p> 
<p><img alt="" class="alignnone size-full wp-image-7239" height="280" src="https://d2908q01vomqb2.cloudfront.net/632667547e7cd3e0466547863e1207a8c0c0c549/2025/11/26/image-1-22.png" width="951" /></p> 
<p>The Amazon SES Tenants Migration process is as follows:</p> 
<ol> 
 <li>Preparation 
  <ol type="a"> 
   <li>Verify SES V2 API usage</li> 
   <li>Update SDK versions</li> 
  </ol> </li> 
 <li>Create Your First Tenant 
  <ol type="a"> 
   <li>Create Tenant API calls</li> 
   <li>Implement in onboarding workflow</li> 
   <li>Verify tenant creation</li> 
  </ol> </li> 
 <li>Associating Resources 
  <ol type="a"> 
   <li>Link verified domains to tenants</li> 
   <li>Associate configuration sets</li> 
   <li>Connect IP pools</li> 
  </ol> </li> 
 <li>Updating Your Sending Code 
  <ol type="a"> 
   <li>Add Tenant/Name parameter to API calls</li> 
   <li>Add X-SES-TENANT header for SMTP</li> 
   <li>Update application sending code</li> 
   <li>Test email sending functionality</li> 
  </ol> </li> 
</ol> 
<h3>Preparation</h3> 
<p>When sending email through a tenant, be sure to specify the tenant in the API calls or SMTP headers and ensure that all resources used are associated with that tenant. Before getting started, keep in mind that there’s an additional charge per tenant per month based on the number of emails. For more detailed information, see <a href="https://aws.amazon.com/ses/pricing/" rel="noopener noreferrer" target="_blank">the SES Pricing page</a>.</p> 
<p>For applications that use SMTP, see the <strong>SMTP Implementation </strong>section later in this document.</p> 
<p>For applications that use the SES API, confirm that the latest Amazon SES V2 API is being used, as tenant management capabilities are only available in this version (see <a href="https://aws.amazon.com/blogs/messaging-and-targeting/upgrade-your-email-tech-stack-with-amazon-sesv2-api/" rel="noopener noreferrer" target="_blank">SES V2 API migration guide</a>). We also recommend verifying that the <a href="https://aws.amazon.com/tools/" rel="noopener noreferrer" target="_blank">AWS SDK</a> version being used supports these operations, otherwise you may need to update to a version that supports them.</p> 
<p>The SES V2 API includes the seven essential operations for managing the tenant architecture that the application will need to leverage throughout the migration (or implementation) and tenant lifecycle:</p> 
<ol> 
 <li><a href="https://docs.aws.amazon.com/ses/latest/APIReference-V2/API_CreateTenant.html" rel="noopener noreferrer" target="_blank">CreateTenant</a> for establishing new tenant containers</li> 
 <li><a href="https://docs.aws.amazon.com/ses/latest/APIReference-V2/API_CreateTenantResourceAssociation.html" rel="noopener noreferrer" target="_blank">CreateTenantResourceAssociation</a> for linking resources like domains and configuration sets to tenants</li> 
 <li><a href="https://docs.aws.amazon.com/ses/latest/APIReference-V2/API_DeleteTenant.html" rel="noopener noreferrer" target="_blank">DeleteTenant</a> for removing tenants when workloads offboard</li> 
 <li><a href="https://docs.aws.amazon.com/ses/latest/APIReference-V2/API_DeleteTenantResourceAssociation.html" rel="noopener noreferrer" target="_blank">DeleteTenantResourceAssociation</a> to unlink resources from tenants</li> 
 <li><a href="https://docs.aws.amazon.com/ses/latest/APIReference-V2/API_GetTenant.html" rel="noopener noreferrer" target="_blank">GetTenant</a> retrieves specific tenant details</li> 
 <li><a href="https://docs.aws.amazon.com/ses/latest/APIReference-V2/API_ListTenants.html" rel="noopener noreferrer" target="_blank">ListTenants</a> provides an overview of all tenants in an account</li> 
 <li><a href="https://docs.aws.amazon.com/ses/latest/APIReference-V2/API_ListTenantResources.html" rel="noopener noreferrer" target="_blank">ListTenantResources</a> shows which resources are associated with each tenant</li> 
</ol> 
<h3>Implementation Steps</h3> 
<h4>Creating the tenant(s)</h4> 
<p>The following Python code example uses the <strong>AWS SDK for Python (Boto3)</strong>&nbsp;and <a href="https://docs.aws.amazon.com/ses/latest/APIReference-V2/API_CreateTenant.html" rel="noopener noreferrer" target="_blank">CreateTenant</a> to demonstrate tenant creation. We’ve added optional tags to better organize the tenant resource for billing or logging purposes.</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">import boto3
from botocore.exceptions import ClientError

def setup_ses_tenant():
&nbsp;&nbsp; &nbsp;ses_client = boto3.client('sesv2')
&nbsp;&nbsp; &nbsp;
&nbsp;&nbsp; &nbsp;# Create a new tenant with descriptive tags
&nbsp;&nbsp; &nbsp;tenant_response = ses_client.create_tenant(
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;TenantName='MyTenant',
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;Tags=[
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;{
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;'Key': 'Environment',
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;'Value': 'Production'
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;{
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;'Key': 'CustomerID',
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;'Value': '[customer_id]'
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;]
&nbsp;&nbsp; &nbsp;)
&nbsp;&nbsp; &nbsp;
&nbsp;&nbsp; &nbsp;# Verify tenant creation
&nbsp;&nbsp; &nbsp;tenants = ses_client.list_tenants()
&nbsp;&nbsp; &nbsp;print(f"Total tenants in account: {len(tenants['Tenants'])}")
&nbsp;&nbsp; &nbsp;
&nbsp;&nbsp; &nbsp;return tenant_response['TenantName']


if __name__ == "__main__":
&nbsp;&nbsp; &nbsp;try:
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;tenant_name = setup_ses_tenant()
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;print(f"Successfully created tenant: {tenant_name}")
&nbsp;&nbsp; &nbsp;except ClientError as e:
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;print(f"AWS error: {e}")
&nbsp;&nbsp; &nbsp;except Exception as e:
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;print(f"Unexpected error: {e}")</code></pre> 
</div> 
<p>This example code can be used when a customer first onboards onto the platform to send email. By creating a tenant for this customer, resources under that tenant will be associated together whenever the tenant is used as explained in the next steps.</p> 
<h4>Associating resources with the tenant</h4> 
<p>Each tenant needs appropriate resources—configuration sets, sending identities, and potentially dedicated IP pools—to begin sending email. The association process should align with the resource sharing strategy as established during the migration planning phase.</p> 
<p>The following Python code example uses the <strong>AWS SDK for Python (Boto3)</strong>&nbsp;and <a href="https://docs.aws.amazon.com/ses/latest/APIReference-V2/API_CreateTenantResourceAssociation.html" rel="noopener noreferrer" target="_blank">CreateTenantResourceAssociation</a> to associate resources to the tenant that was created in the previous step.</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">import boto3
from botocore.exceptions import ClientError

def associate_resources_with_tenant():
&nbsp;&nbsp; &nbsp;ses_client = boto3.client('sesv2')
&nbsp;&nbsp; &nbsp;
&nbsp;&nbsp; &nbsp;try:
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;# Associate verified domain identity
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;ses_client.create_tenant_resource_association(
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;TenantName='MyTenant',
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;ResourceArn='arn:aws:ses:[aws_region]:[account_id]:identity/[domain_name]'
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;)
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;print("Successfully associated email identity with tenant")

&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;# Associate configuration set for tracking
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;ses_client.create_tenant_resource_association(
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;TenantName='MyTenant',
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;ResourceArn='arn:aws:ses:[aws_region]:[account_id]:configuration-set/MyTenantConfigurationSet'
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;)
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;print("Successfully associated configuration set with tenant")

&nbsp;&nbsp; &nbsp;except ClientError as e:
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;print(f"Error: {e.response['Error']['Message']}")
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;raise

if __name__ == "__main__":
&nbsp;&nbsp; &nbsp;associate_resources_with_tenant()</code></pre> 
</div> 
<p>Consider implementing batch association for <em>email streams</em>&nbsp;with multiple domains or configuration sets. This approach reduces API calls and improves provisioning efficiency. Remember that resources can be shared across multiple tenants if the architecture requires it, allowing flexible resource allocation strategies.</p> 
<h3>Update the applications</h3> 
<p>The transition to tenant-based sending requires minimal code changes in the apps, namely adding the <code>TenantName</code> and <code>ConfigurationSetName</code> to the sending process.</p> 
<p><strong>API Implementations</strong></p> 
<p>For applications that use the SES V2 API, add the tenant and configuration set parameters to the send calls as demonstrated below using the <strong>AWS SDK for Python (Boto3)</strong>&nbsp;:</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">import boto3

def send_email_from_tenant():
&nbsp;&nbsp; &nbsp;ses_client = boto3.client('sesv2')
&nbsp;&nbsp; &nbsp;
&nbsp;&nbsp; &nbsp;response = ses_client.send_email(
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;FromEmailAddress='sender@[domain_name]',
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;Destination={
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;'ToAddresses': ['[recipient_email]']
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;Content={
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;'Simple': {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;'Subject': {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;'Data': 'Test email from SES tenant'
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;'Body': {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;'Text': {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;'Data': 'This is a test email sent from an SES tenant'
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;ConfigurationSetName='MyTenantConfigurationSet',
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;TenantName='MyTenant' &nbsp;# Critical addition for tenant routing
&nbsp;&nbsp; &nbsp;)
&nbsp;&nbsp; &nbsp;
&nbsp;&nbsp; &nbsp;print(f"Message sent! Message ID: {response['MessageId']}")
&nbsp;&nbsp; &nbsp;return response

if __name__ == "__main__":
&nbsp;&nbsp; &nbsp;send_email_from_tenant()</code></pre> 
</div> 
<p><strong>SMTP Implementations</strong></p> 
<p>For sending applications that use SMTP, add the <code>X-SES-TENANT</code> and the <code>ConfigurationSetName</code> header parameters to every message. In the code block that follows, we demonstrate the proper header configuration for SMTP sending using Python:</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
import smtplib

def send_smtp_email_with_tenant():
&nbsp;&nbsp; &nbsp;sender = 'smtp-sender@[domain_name]'
&nbsp;&nbsp; &nbsp;sender_name = 'Sender Name'
&nbsp;&nbsp; &nbsp;recipient = '[recipient_email]'
&nbsp;&nbsp; &nbsp;username_smtp = '[smtp_username]'
&nbsp;&nbsp; &nbsp;configuration_set = 'MyTenantConfigurationSet'
&nbsp;&nbsp; &nbsp;host = 'email-smtp.[aws_region].amazonaws.com'
&nbsp;&nbsp; &nbsp;port = 587
&nbsp;&nbsp; &nbsp;
&nbsp;&nbsp; &nbsp;subject = 'Amazon SES test (SMTP interface accessed using Python)'
&nbsp;&nbsp; &nbsp;body_text = """Email Test
&nbsp;&nbsp; &nbsp;This email was sent through the Amazon SES SMTP interface using Python."""
&nbsp;&nbsp; &nbsp;body_html = """&lt;h1&gt;Email Test&lt;/h1&gt;
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;&lt;p&gt;This email was sent through the 
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;&lt;a href="https://aws.amazon.com/ses"&gt;Amazon SES&lt;/a&gt; SMTP
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;interface using Python."&lt;/p&gt;"""
&nbsp;&nbsp; &nbsp;
&nbsp;&nbsp; &nbsp;msg = MIMEMultipart('alternative')
&nbsp;&nbsp; &nbsp;msg['Subject'] = subject
&nbsp;&nbsp; &nbsp;msg['From'] = f"{sender_name} &lt;{sender}&gt;"
&nbsp;&nbsp; &nbsp;msg['To'] = recipient
&nbsp;&nbsp; &nbsp;msg['X-SES-CONFIGURATION-SET'] = configuration_set
&nbsp;&nbsp; &nbsp;
&nbsp;&nbsp; &nbsp;# Critical: Specify tenant for SMTP sending
&nbsp;&nbsp; &nbsp;msg['X-SES-TENANT'] = 'MyTenant'
&nbsp;&nbsp; &nbsp;
&nbsp;&nbsp; &nbsp;part1 = MIMEText(body_text, 'plain')
&nbsp;&nbsp; &nbsp;part2 = MIMEText(body_html, 'html')
&nbsp;&nbsp; &nbsp;msg.attach(part1)
&nbsp;&nbsp; &nbsp;msg.attach(part2)
&nbsp;&nbsp; &nbsp;
&nbsp;&nbsp; &nbsp;try:
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;server = smtplib.SMTP(host, port)
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;server.ehlo()
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;server.starttls()
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;server.ehlo()
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;server.login(username_smtp, fetch_smtp_password_from_secure_storage())
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;server.sendmail(sender, recipient, msg.as_string())
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;print("Email sent successfully!")
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;
&nbsp;&nbsp; &nbsp;except Exception as e:
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;print(f"Error: {str(e)}")
&nbsp;&nbsp; &nbsp;
&nbsp;&nbsp; &nbsp;finally:
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;server.quit()

def fetch_smtp_password_from_secure_storage():
&nbsp;&nbsp; &nbsp;# Implement secure password retrieval from AWS Secrets Manager
&nbsp;&nbsp; &nbsp;# or your preferred secret storage solution
&nbsp;&nbsp; &nbsp;return '[smtp_password]'


if __name__ == "__main__":
&nbsp;&nbsp; &nbsp;send_smtp_email_with_tenant()</code></pre> 
</div> 
<h2>Configuring IAM policies and permissions for tenants</h2> 
<p>This section explains how to configure IAM permissions for SES tenants, including how to set up different permission levels for tenant management, email sending, and monitoring while following security best practices to control access based on organizational roles. Remember that IAM policies for tenants follow the principle of least privilege. Start with minimal permissions and expand as needed, regularly reviewing and removing unused permissions to maintain security.</p> 
<h3>Tenant management permissions</h3> 
<p>SES Tenants can be controlled through specific IAM permissions that determine who can create, modify, and use specific tenant(s) in the organization. The tenant management system’s core API actions discussed previously can be granted with granular access through IAM policies for administrative operations. The <a href="https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazonsimpleemailservicev2.html" rel="noopener noreferrer" target="_blank">Service Authorization Reference for Amazon Simple Email Service v2</a> page contains the latest documentation for the service-specific actions used below for Amazon Simple Email Service.</p> 
<p>We recommend limiting each IAM role’s permission based on the minimum capabilities required by that role. This helps mitigate the potential for negative effects if an SMTP credential is misused. What follows is a basic IAM policy that demonstrates full tenant management capabilities; this is NOT demonstrating the principle of least privilege (yet):</p> 
<div class="hide-language"> 
 <pre><code class="lang-css">{
&nbsp;&nbsp; &nbsp;"Version": "2012-10-17",
&nbsp;&nbsp; &nbsp;"Statement": [
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;{
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Effect": "Allow",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Action": [
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"ses:CreateTenant",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"ses:DeleteTenant",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"ses:GetTenant",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"ses:ListTenants",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"ses:CreateTenantResourceAssociation",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"ses:DeleteTenantResourceAssociation",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"ses:ListTenantResources",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"ses:ListResourceTenants"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;],
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Resource": "*"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp;]
}</code></pre> 
</div> 
<h3>Configuring sending permissions with tenants</h3> 
<p>Applications that send emails through tenants need different permissions than those managing tenants. The key distinction is using the <code>ses:TenantName</code> condition to restrict which tenants an application can use for sending.</p> 
<p>The IAM policy below allows sending emails only through the specified <code>CustomerA-Tenant</code> tenant, ensuring applications can’t accidentally or maliciously send through other customers’ tenants.</p> 
<div class="hide-language"> 
 <pre><code class="lang-css">{
&nbsp;&nbsp; &nbsp;"Version": "2012-10-17",
&nbsp;&nbsp; &nbsp;"Statement": [
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;{
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Effect": "Allow",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Action": [
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"ses:SendEmail",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"ses:SendBulkEmail"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;],
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Resource": [
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"arn:aws:ses:[aws_region]:[account_id]:identity/*",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"arn:aws:ses:[aws_region]:[account_id]:configuration-set/*"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;],
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Condition": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"StringEquals": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"ses:TenantName": "CustomerA-Tenant"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp;]
}</code></pre> 
</div> 
<h3>Separating administrative and operational access</h3> 
<p>Production environments implement role separation between tenant management and email sending operations. Administrative roles handle tenant creation and resource association during customer onboarding, while application roles can only send emails through assigned tenants. The IAM policy below is an example of an administrative role; it allows creating and configuring tenants for use during customer onboarding but does not allow the tenant deletion action to prevent accidental deletions.</p> 
<div class="hide-language"> 
 <pre><code class="lang-css">{
&nbsp;&nbsp; &nbsp;"Version": "2012-10-17",
&nbsp;&nbsp; &nbsp;"Statement": [
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;{
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Effect": "Allow",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Action": [
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"ses:CreateTenant",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"ses:ListTenants"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;],
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Resource": "*"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;{
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Effect": "Allow",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Action": [
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"ses:CreateTenantResourceAssociation",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"ses:GetTenant"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;],
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Resource": "arn:aws:ses:[aws_region]:[account_id]:tenant/*/tn-*"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;{
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Effect": "Deny",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Action": "ses:DeleteTenant",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Resource": "*"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp;]
}</code></pre> 
</div> 
<h3>Resource-level permissions</h3> 
<p>Tenants support resource-level permissions using Amazon Resource Names (ARNs), enabling fine-grained access control. Grant access to specific tenants by specifying the tenant name and tenant id (ex. <code>CustomerA-Tenant/tn-1a2b3c4d5e6f7890abcdef1234567890</code>) rather than granting blanket permissions using the <code>“*/tn-*”</code> wildcard as above. To obtain the tenant id you can use the <a href="https://docs.aws.amazon.com/cli/latest/reference/sesv2/list-tenants.html" rel="noopener noreferrer" target="_blank">list-tenants</a> command of the AWS SESv2 CLI.</p> 
<p>The IAM policy below grants access only to tenants <code>CustomerA-Tenant</code> and <code>CustomerB-Tenant</code> where the following tenant id is a placeholder that should be replaced by the correct tenant id that you obtained from list-tenants.</p> 
<div class="hide-language"> 
 <pre><code class="lang-css">{
&nbsp;&nbsp; &nbsp;"Version": "2012-10-17",
&nbsp;&nbsp; &nbsp;"Statement": [
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;{
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Effect": "Allow",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Action": "ses:GetTenant",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Resource": [
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"arn:aws:ses:[aws_region]:[account_id]:tenant/CustomerA-Tenant/tn-1a2b3c4d5e6f7890abcdef1234567890",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"arn:aws:ses:[aws_region]:[account_id]:tenant/CustomerB-Tenant/tn-9876543210fedcba0987654321abcdef"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;]
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp;]
}</code></pre> 
</div> 
<h3>Monitoring and compliance access</h3> 
<p>Security and compliance teams often need read-only access to monitor tenant usage. The following IAM policy grants read-only access to all tenants, but does not permit modifications or deletions of any tenants:</p> 
<div class="hide-language"> 
 <pre><code class="lang-css">{
&nbsp;&nbsp; &nbsp;"Version": "2012-10-17",
&nbsp;&nbsp; &nbsp;"Statement": [
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;{
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Effect": "Allow",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Action": [
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"ses:GetTenant",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"ses:ListTenants",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"ses:ListTenantResources"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;],
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Resource": "*"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp;]
}</code></pre> 
</div> 
<h2>Monitoring Tenants with EventBridge and CloudWatch</h2> 
<p>Amazon SES integrates with <a href="https://aws.amazon.com/eventbridge/" rel="noopener noreferrer" target="_blank">Amazon EventBridge</a> to deliver comprehensive monitoring capabilities for tenant management, providing real-time visibility into reputation changes and enabling automated response workflows. EventBridge is a serverless service that uses JSON-formatted events to connect application components, making it straightforward to build scalable event-driven applications. Amazon SES’s tenant feature’s integration with EventBridge enables organizations to track tenant-specific metrics and other metrics, such as reputation findings and reputation status changes, and tenant status changes.</p> 
<h3>Understanding EventBridge Integration with SES</h3> 
<p>EventBridge operates as a router that receives events from SES and delivers them to one or many destinations (aka targets). When SES features experience state changes or status updates, they automatically send events to the EventBridge default event bus. Rules associated with the event bus evaluate events as they arrive, checking whether each event matches the rule’s pattern before routing to specified targets. For SES tenant management, organizations can receive real-time alerts through Amazon EventBridge when tenant reputation findings are detected or when tenant status changes occur. These events are delivered on a best-effort basis; they might be delivered out of order, requiring users to deploy processing logic to handle such scenarios gracefully.</p> 
<p>The code block that follows can guide users through the basic EventBridge integration with the SES tenant feature. For more extensive documentation on integrating with EventBridge please consult <a href="https://docs.aws.amazon.com/ses/latest/dg/tenants.html" rel="noopener noreferrer" target="_blank">AWS documentation</a>.</p> 
<h3>Setting up EventBridge integration</h3> 
<p>Create an EventBridge rule using the <strong>AWS SDK for Python (Boto3)</strong>&nbsp;to capture tenant status changes and reputation findings:</p> 
<div class="hide-language"> 
 <pre><code class="lang-css"># Create rule for monitoring tenant status changes
aws events put-rule \
&nbsp;&nbsp; &nbsp;--name "SESTenantStatusMonitor" \
&nbsp;&nbsp; &nbsp;--description "Monitor SES tenant status changes" \
&nbsp;&nbsp; &nbsp;--event-pattern '{
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;"source": ["aws.ses"],
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;"detail-type": [
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Sending Status Enabled",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Sending Status Disabled",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Advisor Recommendation Status Open",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Advisor Recommendation Status Closed"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;]
&nbsp;&nbsp; &nbsp;}'</code></pre> 
</div> 
<h3>Routing events from EventBridge to CloudWatch Logs</h3> 
<p>CloudWatch provides the capability to collect raw data and process it into readable, near real-time metrics. Follow these steps to set up a CloudWatch log group for SES tenant events and configure the appropriate IAM permissions:</p> 
<ol> 
 <li>Create a CloudWatch log group for tenant events:</li> 
</ol> 
<p><code>aws logs create-log-group --log-group-name "/aws/ses/tenants"</code></p> 
<ol start="2"> 
 <li>Add resource-based policy to CloudWatch Logs:</li> 
</ol> 
<div class="hide-language"> 
 <pre><code class="lang-css">aws logs put-resource-policy \
&nbsp;&nbsp; &nbsp;--policy-name EventBridgeToCloudWatchLogsPolicy \
&nbsp;&nbsp; &nbsp;--policy-document '{
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;"Version": "2012-10-17",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;"Statement": [{
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Sid": "TrustEventsToStoreLogEvent",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Effect": "Allow",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Principal": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Service": ["events.amazonaws.com", "delivery.logs.amazonaws.com"]
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Action": ["logs:CreateLogStream", "logs:PutLogEvents"],
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Resource": "arn:aws:logs:[aws_region]: [account_id]:log-group:/aws/ses/tenants:*"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;}]
&nbsp;&nbsp; &nbsp;}'</code></pre> 
</div> 
<ol start="3"> 
 <li>Add the EventBridge target:</li> 
</ol> 
<div class="hide-language"> 
 <pre><code class="lang-css">
aws events put-targets \
&nbsp;&nbsp; &nbsp;--rule "SESTenantStatusMonitor" \
&nbsp;&nbsp; &nbsp;--targets '[{
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;"Id": "SendToCloudWatchLogs",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;"Arn": "arn:aws:logs:[aws_region]:[account_id]:log-group:/aws/ses/tenants"
&nbsp;&nbsp; &nbsp;}]'</code></pre> 
</div> 
<p>An example event of a paused (“disabled”) tenant is shown below for reference:</p> 
<div class="hide-language"> 
 <pre><code class="lang-css">{
&nbsp;&nbsp; &nbsp;"version": "0",
&nbsp;&nbsp; &nbsp;"id": "3cc76530-9842-03a9-fdef-e4e4f667cf4e",
&nbsp;&nbsp; &nbsp;"detail-type": "Sending Status Disabled",
&nbsp;&nbsp; &nbsp;"source": "aws.ses",
&nbsp;&nbsp; &nbsp;"account": "[account_id]",
&nbsp;&nbsp; &nbsp;"time": "2025-10-01T03:37:17Z",
&nbsp;&nbsp; &nbsp;"region": "[aws_region]",
&nbsp;&nbsp; &nbsp;"resources": [
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;"arn:aws:ses:[aws_region]::tenant/CustomerA-Tenant/tn-2a8c678ec0000fdaf76cc1f127b40"
&nbsp;&nbsp; &nbsp;],
&nbsp;&nbsp; &nbsp;"detail": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;"version": "1.0.0",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;"data": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"origin": "CUSTOMER_MANAGED",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"record": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"status": "DISABLED",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"cause": "Status manually updated.",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"lastUpdatedTimestamp": [
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;2025,
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;10,
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;1,
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;3,
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;37,
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;17,
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;671000000
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;]
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp;}
}</code></pre> 
</div> 
<h2>Managing tenant reputation with key Tenants features</h2> 
<p>Reputation management for individual tenants is one of the core benefits of using Amazon SES’s tenant feature, providing automated protection against deliverability issues that could damage overall account reputation. This section demonstrates how to configure reputation policies that can automatically pause tenants when they experience high bounce rates or reputation issues, as well as how to manually control tenant sending status for custom workflows.</p> 
<h3>Setting reputation policies</h3> 
<p>Amazon SES provides three reputation policy enforcement levels that determine how the system responds to reputation findings.</p> 
<ul> 
 <li>The <strong>Standard</strong> policy (recommended) pauses sending only for high-impact findings, providing a balance between protection and operational flexibility.</li> 
 <li>The <strong>Strict</strong> policy pauses sending for any reputation finding, offering maximum protection for sensitive environments.</li> 
 <li>The <strong>None</strong> option disables automated pausing while continuing to track findings, useful for manual monitoring scenarios. 
  <ul> 
   <li><strong>Note:</strong> Even with the <strong>None</strong> policy enabled, Amazon SES reserves the right to apply sending pauses if a tenant exhibits severe reputation issues to protect the overall sending infrastructure and deliverability for all customers.</li> 
  </ul> </li> 
</ul> 
<p>Reputation findings are generated in two severity levels—<strong>low</strong> and <strong>high</strong>—based on metrics like bounce rates and complaint rates. When these metrics indicate a potential deliverability issue, SES creates findings that can trigger automatic pausing based on a chosen policy.</p> 
<p>In the following code block, we demonstrate how to set a reputation policy on the <code>CustomerA-Tenant</code></p> 
<div class="hide-language"> 
 <pre><code class="lang-python">import boto3
from botocore.exceptions import ClientError

def update_tenant_reputation_policy(tenant_arn, policy_arn):
&nbsp;&nbsp; &nbsp;ses_client = boto3.client('sesv2')
&nbsp;&nbsp; &nbsp;
&nbsp;&nbsp; &nbsp;ses_client.update_reputation_entity_policy(
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;ReputationEntityType='RESOURCE',
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;ReputationEntityReference=tenant_arn,
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;ReputationEntityPolicy=policy_arn
&nbsp;&nbsp; &nbsp;)
&nbsp;&nbsp; &nbsp;print(f"Tenant policy_arn updated to: {policy_arn}")

# Example usage
if __name__ == "__main__":
&nbsp;&nbsp; &nbsp;tenant_arn = "arn:aws:ses:[aws_region]:[account_id]:tenant/CustomerA-Tenant/tn-2a8c678ec0000fdaf76cc1f127b40"
&nbsp;&nbsp; &nbsp;
&nbsp;&nbsp; &nbsp;# Enable normal sending
&nbsp;&nbsp; &nbsp;update_tenant_reputation_policy(tenant_arn, 'arn:aws:ses:[aws_region]:aws:reputation-policy/standard')
&nbsp;&nbsp; &nbsp;
&nbsp;&nbsp; &nbsp;# Disable/pause sending
&nbsp;&nbsp; &nbsp;update_tenant_reputation_policy(tenant_arn, 'arn:aws:ses:[aws_region]:aws:reputation-policy/strict')
&nbsp;&nbsp; &nbsp;
&nbsp;&nbsp; &nbsp;# Reinstate with monitoring
&nbsp;&nbsp; &nbsp;update_tenant_reputation_policy(tenant_arn, 'arn:aws:ses:[aws_region]:aws:reputation-policy/none')</code></pre> 
</div> 
<p>Our recommendation is to choose the <strong>Standard</strong> policy for most production tenants, as this policy provides automatic protection against severe reputation issues while avoiding unnecessary disruptions. Reserve the <strong>Strict</strong> policy for new or untrusted tenants where maximum caution is warranted. Use the <strong>None</strong> option during initial monitoring periods or when implementing custom reputation management logic.</p> 
<h3>Handling paused tenants</h3> 
<p>When a tenant is paused, either automatically through reputation policies or manually, the sending status prevents any emails from being sent through that tenant. The system derives this aggregate status from both customer-managed and AWS-managed statuses; if either is set to <code>DISABLED</code>, the tenant cannot send emails.</p> 
<p>Amazon SES publishes notifications to EventBridge when tenant status changes occur or new reputation findings are detected, enabling real-time response to reputation events. After investigating and resolving the underlying issues, the tenant’s sending capabilities can be reinstated. During reinstatement (REINSTATED status), the tenant can continue sending while metrics are monitored to verify improvement.</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">def update_tenant_status(tenant_arn, status):
&nbsp;&nbsp; &nbsp;ses_client = boto3.client('sesv2')
&nbsp;&nbsp; &nbsp;
&nbsp;&nbsp; &nbsp;ses_client.update_reputation_entity_customer_managed_status(
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;ReputationEntityType='RESOURCE',
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;ReputationEntityReference=tenant_arn,
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;SendingStatus=status
&nbsp;&nbsp; &nbsp;)
&nbsp;&nbsp; &nbsp;print(f"Tenant status updated to: {status}")

# Example usage
if __name__ == "__main__":
&nbsp;&nbsp; &nbsp;tenant_arn = "arn:aws:ses:[aws_region]:[account_id]:tenant/CustomerA-Tenant/tn-2a8c678ec0000fdaf76cc1f127b40"
&nbsp;&nbsp; &nbsp;
&nbsp;&nbsp; &nbsp;# Enable normal sending
&nbsp;&nbsp; &nbsp;update_tenant_status(tenant_arn, 'ENABLED')
&nbsp;&nbsp; &nbsp;
&nbsp;&nbsp; &nbsp;# Disable/pause sending
&nbsp;&nbsp; &nbsp;update_tenant_status(tenant_arn, 'DISABLED')
&nbsp;&nbsp; &nbsp;
&nbsp;&nbsp; &nbsp;# Reinstate with monitoring
&nbsp;&nbsp; &nbsp;update_tenant_status(tenant_arn, 'REINSTATED')</code></pre> 
</div> 
<p>The <code>REINSTATED</code> status allows the tenant to continue sending even with active reputation findings.&nbsp;Once metrics return to healthy levels, the tenant automatically transitions back to <code>ENABLED</code> status. This approach ensures minimal disruption while protecting the overall account reputation from problematic email streams.</p> 
<h2>Managing tenant lifecycle</h2> 
<p>When customers modify a service tier or leave the platform, proper cleanup ensures resource efficiency and maintains account organization.</p> 
<h3>Removing resource associations</h3> 
<p>Before deleting a tenant, remove all resource associations to prevent orphaned configurations:</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">import boto3
from botocore.exceptions import ClientError

def remove_resource_from_tenant():
&nbsp;&nbsp; &nbsp;ses_client = boto3.client('sesv2')
&nbsp;&nbsp; &nbsp;
&nbsp;&nbsp; &nbsp;try:
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;# Remove identity association
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;ses_client.delete_tenant_resource_association(
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;TenantName='MyTenant',
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;ResourceArn='arn:aws:ses:[aws_region]:[account_id]:identity/[domain_name]'
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;)
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;print("Successfully removed identity association")
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;# Remove configuration set association
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;ses_client.delete_tenant_resource_association(
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;TenantName='MyTenant',
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;ResourceArn='arn:aws:ses:[aws_region]:[account_id]:configuration-set/MyTenantConfigurationSet'
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;)
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;print("Successfully removed configuration set association")
&nbsp;&nbsp; &nbsp;
&nbsp;&nbsp; &nbsp;except ClientError as e:
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;print(f"Error: {e.response['Error']['Message']}")
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;raise


if __name__ == "__main__":
&nbsp;&nbsp; &nbsp;remove_resource_from_tenant()</code></pre> 
</div> 
<h3>Deleting tenants</h3> 
<p>Once all resources are disassociated, remove the tenant entirely:</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">import boto3
from botocore.exceptions import ClientError

def delete_tenant(tenant_name):
&nbsp;&nbsp; &nbsp;ses_client = boto3.client('sesv2')
&nbsp;&nbsp; &nbsp;
&nbsp;&nbsp; &nbsp;try:
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;# Delete the tenant
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;ses_client.delete_tenant(TenantName=tenant_name)
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;print(f"Successfully deleted tenant: {tenant_name}")
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;return True
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;
&nbsp;&nbsp; &nbsp;except ClientError as e:
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;print(f"Error deleting tenant: {e.response['Error']['Message']}")
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;raise

# Example usage with error handling
if __name__ == "__main__":
&nbsp;&nbsp; &nbsp;try:
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;delete_tenant("MyTenant")
&nbsp;&nbsp; &nbsp;except ClientError as e:
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;print(f"Cleanup failed: {e}")</code></pre> 
</div> 
<p>Tenant lifecycle management ensures clean transitions when customers change service tiers or leave the platform. Implement these operations in customer offboarding workflows to maintain optimal account organization and resource utilization.</p> 
<h3>Resource Management</h3> 
<h4>Resource Sharing Capabilities</h4> 
<p>Resources can be assigned to multiple tenants simultaneously. This enables sharing common resources between tenants while maintaining separate reputation tracking. For example, the reputation for marketing and transactional email could be tracked separately across independent tenants while using the same sending domain. SES validates tenant-resource associations at send time, rejecting requests if the specified tenant lacks access to the requested resources.</p> 
<h4>Resource Migration Between Tenants</h4> 
<p>Resource migration involves two API calls. First, remove the association from the current tenant using <a href="https://docs.aws.amazon.com/ses/latest/APIReference-V2/API_DeleteTenantResourceAssociation.html" rel="noopener noreferrer" target="_blank">DeleteTenantResourceAssociation</a>, then create a new association with the target tenant using <a href="https://docs.aws.amazon.com/ses/latest/APIReference-V2/API_CreateTenantResourceAssociation.html" rel="noopener noreferrer" target="_blank">CreateTenantResourceAssociation</a>. This process can be automated for bulk migrations during reorganizations.</p> 
<h3>Reputation Management</h3> 
<h4>Tenant Isolation Protection</h4> 
<p>Each tenant maintains independent reputation metrics and sending status. When one tenant experiences deliverability issues, it can be automatically paused without affecting other tenants’ ability to send, protecting both shared resources and account-level reputation.</p> 
<h4>Tenant Pausing Triggers</h4> 
<p>Tenants can either be paused manually using the <a href="https://docs.aws.amazon.com/ses/latest/APIReference-V2/API_UpdateReputationEntityCustomerManagedStatus.html" rel="noopener noreferrer" target="_blank">UpdateReputationEntityCustomerManagedStatus</a> API or paused automatically based on the reputation policy assigned to the tenant. Reputation policies pause tenants based on reputation findings generated from bounce rates, complaint rates, and third-party feedback reports. The Standard policy (recommended) pauses only for high-severity findings (bounce rate &gt; 15%, complaint rate &gt; 1%), while Strict pauses even for low severity findings (bounce rate &gt; 10%, complaint rate &gt; 0.5%).</p> 
<h4>Tenant Reactivation Process</h4> 
<p>For tenants paused by automated reputation policies, use the <a href="https://docs.aws.amazon.com/ses/latest/APIReference-V2/API_UpdateReputationEntityCustomerManagedStatus.html" rel="noopener noreferrer" target="_blank">UpdateReputationEntityCustomerManagedStatus</a> API to reinstate sending after addressing root causes. Tenants paused by AWS Trust &amp; Safety require case resolution through <a href="https://aws.amazon.com/premiumsupport/" rel="noopener noreferrer" target="_blank">AWS Support.</a></p> 
<h4>Migrating Existing Customers</h4> 
<p>Creating tenants for existing email streams can be completed with no disruption to email sending. Start by creating tenants for each customer or business unit, then associate existing resources like email identities, configuration sets, and templates using the tenant association APIs. Once those steps are complete, update the application, or inform the customer or BU they now need to specify the tenant name and configuration set in their SES <a href="https://docs.aws.amazon.com/ses/latest/APIReference-V2/API_SendEmail.html" rel="noopener noreferrer" target="_blank">SendEmail</a> API calls or SMTP headers which enables SES to route emails through the appropriate tenant.</p> 
<h4>Reputation Metrics Transition</h4> 
<p>New reputation metrics will be tracked separately for each tenant from the point of creation and use. Historical metrics from before tenant implementation are not available. Tenant metrics contribute to the overall account-level reputation. For example, if tenant-a, tenant-b, and tenant-c each send 1,000 emails and:</p> 
<ul> 
 <li>Tenant-a receives 150 bounce notifications over 1,000 emails (for a bounce rate of 15%), tenant reputation protection will pause this tenant before the issue escalates.</li> 
 <li>Tenant-b receives 0 bounces over 1,000 emails, for a bounce rate of 0%</li> 
 <li>Tenant-c receives 0 bounces over 1,000 emails, for a bounce rate of 0%</li> 
</ul> 
<p>The SES Account Level Bounce Rate (all tenants) is 150 out of 3,000, or 5%</p> 
<h4>Account Activation Requirements</h4> 
<p>No account-level activation is required to configure and use the Tenant feature, it is immediately available through the SES V2 API or the AWS SES Console. Users can also start using the tenant management APIs (<a href="https://docs.aws.amazon.com/ses/latest/APIReference-V2/API_CreateTenant.html" rel="noopener noreferrer" target="_blank">CreateTenant</a>, <a href="https://docs.aws.amazon.com/ses/latest/APIReference-V2/API_CreateTenantResourceAssociation.html" rel="noopener noreferrer" target="_blank">CreateTenantResourceAssociation</a>, <a href="https://docs.aws.amazon.com/ses/latest/APIReference-V2/API_DeleteTenant.html" rel="noopener noreferrer" target="_blank">DeleteTenant</a>, etc.) without any account modifications or support requests.</p> 
<h2>Conclusion</h2> 
<p>This post covered detailed migration steps, monitoring setup, practical implementation examples and troubleshooting steps. From running a SaaS platform, to managing multiple brands, to operating separate business units, tenant-based reputation isolation ensures Amazon SES email infrastructure scales reliably as an organization grows.</p> 
<h2>Additional resources</h2> 
<ul> 
 <li><a href="https://aws.amazon.com/blogs/messaging-and-targeting/improve-email-deliverability-with-tenant-management-in-amazon-ses/" rel="noopener noreferrer" target="_blank">Blog:&nbsp;Improve email deliverability with tenant management in Amazon SES</a></li> 
 <li><a href="https://docs.aws.amazon.com/ses/latest/dg/tenants.html" rel="noopener noreferrer" target="_blank">Tenants in Developer Guide</a></li> 
 <li><a href="https://docs.aws.amazon.com/ses/latest/APIReference-V2/API_CreateTenant.html" rel="noopener noreferrer" target="_blank">CreateTenant API SES V2 Documentation</a></li> 
</ul> 
<p style="clear: both;"></p> 
<hr style="width: 100%;" /> 
<h3>About the authors</h3>
