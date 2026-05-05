---
title: "Enhance email security using VPC endpoints with Amazon SES Manager"
url: "https://aws.amazon.com/blogs/messaging-and-targeting/enhance-email-security-using-vpc-endpoints-with-amazon-ses-manager/"
date: "Tue, 30 Dec 2025 15:00:22 +0000"
author: "Gabrielle Zhou"
feed_url: "https://aws.amazon.com/blogs/messaging-and-targeting/feed/"
---
<p>Organizations managing on-premises email infrastructure face a critical challenge: how to modernize email systems while maintaining strict security and compliance standards. For healthcare providers, financial institutions, and government agencies, email messages often contain sensitive data that must remain on private networks throughout processing.</p> 
<p>The virtual private cloud (VPC) endpoint feature of <a href="http://aws.amazon.com/ses" rel="noopener noreferrer" target="_blank">Amazon Simple Email Service</a> (Amazon SES) <a href="https://docs.aws.amazon.com/ses/latest/dg/eb.html" rel="noopener noreferrer" target="_blank">Mail Manager</a> addresses this challenge by enabling SMTP messages to remain on your private network throughout processing, routing, and compliance logging before final delivery. This post walks you through implementing this solution to securely modernize your email infrastructure.</p> 
<p>Consider this scenario: You’re responsible for a healthcare organization’s email infrastructure that processes thousands of patient communications daily. Your on-premises Exchange servers are aging, maintenance costs are climbing, and your organization is moving workloads to AWS. Your security team requires that email processing for sensitive patient communications—including workflow processing, temporary storage, rule-based routing, and compliance logging—remain within private, controlled networks until ready for final delivery. The Amazon SES Mail Manager VPC endpoint feature addresses this requirement by maintaining network-level isolation for email operations from generation through processing, minimizing data exposure, meeting compliance requirements, and providing defense-in-depth security before final message delivery.</p> 
<p>This post demonstrates how to implement VPC endpoints with Amazon SES Mail Manager using exercises from the <a href="https://catalog.workshops.aws/migrate-w-mailmanager" rel="noopener noreferrer" target="_blank">Amazon SES Mail Manager workshop</a>. We show how to configure VPC endpoints, security groups, and ingress endpoints to maintain private network connectivity for your email processing workflows.</p> 
<h2>Solution overview</h2> 
<p>Our approach combines AWS services to create a secure, private email infrastructure:</p> 
<ul> 
 <li><a href="http://aws.amazon.com/vpc" rel="noopener noreferrer" target="_blank">Amazon Virtual Private Cloud</a> (Amazon VPC) <a href="https://docs.aws.amazon.com/whitepapers/latest/aws-privatelink/what-are-vpc-endpoints.html" rel="noopener noreferrer" target="_blank">endpoints</a> powered by <a href="https://aws.amazon.com/privatelink/" rel="noopener noreferrer" target="_blank">AWS PrivateLink</a></li> 
 <li>AWS <a href="https://docs.aws.amazon.com/managedservices/latest/userguide/about-security-groups.html" rel="noopener noreferrer" target="_blank">security groups</a> to further limit SMTP message traffic to approved network subnets</li> 
 <li><a href="https://aws.amazon.com/secrets-manager/" rel="noopener noreferrer" target="_blank">AWS Secrets Manager</a> to manage the creation and rotation of SMTP passwords</li> 
 <li><a href="https://aws.amazon.com/kms/" rel="noopener noreferrer" target="_blank">AWS Key Management Service</a> (AWS KMS) to encrypt and decrypt the SMTP password stored in Secrets Manager</li> 
</ul> 
<p>This solution requires your applications to run within a VPC or have established connectivity between your on-premises network and Amazon VPC through <a href="https://aws.amazon.com/directconnect/" rel="noopener noreferrer" target="_blank">AWS Direct Connect</a> or VPN. For guidance on connecting on-premises networks to AWS, refer to <a href="https://docs.aws.amazon.com/whitepapers/latest/hybrid-connectivity/hybrid-network-connections.html" rel="noopener noreferrer" target="_blank">Hybrid network connections</a>.</p> 
<p>The following diagram illustrates the solution architecture.</p> 
<p><img alt="" class="alignnone size-full wp-image-7274" height="582" src="https://d2908q01vomqb2.cloudfront.net/632667547e7cd3e0466547863e1207a8c0c0c549/2025/12/29/MESSAGING-1364-image-1.jpg" width="842" /></p> 
<p>The workflow consists of the following steps:</p> 
<ol> 
 <li><a href="http://aws.amazon.com/ec2" rel="noopener noreferrer" target="_blank">Amazon Elastic Compute Cloud</a> (Amazon EC2) instances running the sender email application on subnet 10.0.0.0/18 connect to the Amazon SES Mail Manager ingress endpoint through a VPC endpoint.</li> 
 <li>Sender credentials are retrieved securely from Secrets Manager.</li> 
 <li>AWS KMS decrypts credentials using your managed encryption keys.</li> 
 <li>Authenticated email traffic flows securely to SES Amazon SES Mail Manager.</li> 
</ol> 
<h2>Prerequisites</h2> 
<p>Before beginning your migration, ensure you have the following:</p> 
<ul> 
 <li><strong>AWS account </strong>– Use an AWS account with appropriate permissions for creating and managing a VPC, Secrets Manager, AWS KMS, and Amazon SES. Make sure <a href="https://aws.amazon.com/iam/" rel="noopener noreferrer" target="_blank">AWS Identity and Access Management</a> (IAM) policies follow least privilege principles.</li> 
 <li><strong>Existing VPC infrastructure</strong> – Use a VPC that hosts your applications in the same AWS account and AWS Region as Amazon SES. For more information, see <a href="https://docs.aws.amazon.com/vpc/latest/userguide/vpc-getting-started.html" rel="noopener noreferrer" target="_blank">Plan your VPC</a>.</li> 
 <li><strong>Amazon SES configured</strong> – Configure Amazon SES in the same Region and AWS account.</li> 
 <li><strong>Network connectivity</strong> – Deploy application servers either on premises with network connectivity to your VPC using Direct Connect or VPN, or already running within the VPC. 
  <ul> 
   <li>AWS blocks outbound SMTP traffic on port 25 across most AWS services. You can either use an alternative TCP port, such as 587 (recommended), or <a href="https://repost.aws/knowledge-center/ec2-port-25-throttle" rel="noopener noreferrer" target="_blank">request port 25 exemption by submitting a request to AWS Support</a> from your AWS account using the <strong>Request to remove email sending limitations</strong> form. This can take upwards of 7 business days to be reviewed; approval is not guaranteed.</li> 
   <li>Configure DNS resolution for Amazon SES Mail Manager VPC endpoints (for more details, see <a href="https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-overview-DSN-queries-to-vpc.html" rel="noopener noreferrer" target="_blank">Resolving DNS queries between VPCs and your network</a>).</li> 
  </ul> </li> 
</ul> 
<p>For this example, we use Linux SMTP commands from an EC2 instance in the VPC to connect to the Amazon SES Mail Manager ingress endpoint through a VPC endpoint on port 587.</p> 
<h2>Create traffic policy</h2> 
<p>Create an Amazon SES Mail Manager traffic policy to filter incoming messages by a combination of recipient address, sender IP address range, and TLS protocol version (1.2 or 1.3). For more details about Amazon SES Mail Manager traffic policies, refer to <a href="https://docs.aws.amazon.com/ses/latest/dg/eb-filters.html" rel="noopener noreferrer" target="_blank">Traffic policies and policy statements</a>. In this example, we use a traffic policy with minimum TLS version of 1.2.</p> 
<p>Complete the following steps:</p> 
<ol> 
 <li>Open the Amazon SES console in the target Region.</li> 
 <li>In the navigation pane, under <strong>SES Mail Manager</strong>, choose <strong>Traffic policies</strong>.</li> 
 <li>Choose <strong>Create traffic policy</strong>.</li> 
 <li>For <strong>Policy name</strong>, enter a descriptive name, such as <code>first-traffic-policy</code>.</li> 
 <li>For <strong>Default action</strong>, choose <strong>Deny</strong>.</li> 
 <li>Choose <strong>Add new policy statement</strong>.</li> 
 <li>For <strong>Allow or deny properties</strong>, choose <strong>Allow</strong>.</li> 
 <li>For <strong>Properties</strong>, choose <strong>TLS protocol version</strong>.</li> 
 <li>For <strong>Operator</strong>, choose <strong>Minimum version</strong>.</li> 
 <li>For <strong>Value</strong>, choose <strong>TLS 1.2</strong>.</li> 
 <li>Choose <strong>Create traffic policy</strong>.</li> 
</ol> 
<p><img alt="" class="alignnone size-full wp-image-7275" height="608" src="https://d2908q01vomqb2.cloudfront.net/632667547e7cd3e0466547863e1207a8c0c0c549/2025/12/29/MESSAGING-1364-image-2.jpeg" width="1290" /></p> 
<h2>Create rule set</h2> 
<p>Rule sets are containers for an ordered set of rules that determine how the messages are processed. For more information, see <a href="https://docs.aws.amazon.com/ses/latest/dg/eb-rules.html" rel="noopener noreferrer" target="_blank">Rule sets and rules</a>. In this example, we use the archive rule to archive all emails processed by Amazon SES Mail Manager.</p> 
<p>Complete the following steps:</p> 
<ol> 
 <li>On the Amazon SES console, under <strong>Amazon SES Mail Manager</strong> in the navigation pane, choose<strong> Rule sets</strong>.</li> 
 <li>Choose <strong>Create rule set</strong>.</li> 
 <li>Name the rule (for example, <code>first-rule-set</code>) and choose <strong>Create rule set</strong>.</li> 
 <li>Choose <strong>Create new rule</strong>, then choose <strong>Create new rule</strong> again.</li> 
 <li>Under <strong>Rule settings</strong>, name the rule (for example, <code>archive_all</code>).</li> 
 <li>Under <strong>Actions</strong>, choose <strong>Add new action</strong>.</li> 
 <li>Chose <strong>Archive</strong>.</li> 
 <li>Choose <strong>Create archive</strong>.</li> 
 <li>Give the archive a name, such as <code>archive_all</code>.</li> 
 <li>Set a retention period (3 months for testing).</li> 
 <li>Choose <strong>Create</strong> <strong>archive</strong>.</li> 
</ol> 
<p><img alt="" class="alignnone size-full wp-image-7276" height="396" src="https://d2908q01vomqb2.cloudfront.net/632667547e7cd3e0466547863e1207a8c0c0c549/2025/12/29/MESSAGING-1364-image-3.jpg" width="807" /></p> 
<ol start="12"> 
 <li>For <strong>Archive resource name</strong>, choose <code>archive_all</code>.</li> 
 <li>Choose <strong>Save rule set</strong>.</li> 
</ol> 
<p><img alt="" class="alignnone size-full wp-image-7277" height="452" src="https://d2908q01vomqb2.cloudfront.net/632667547e7cd3e0466547863e1207a8c0c0c549/2025/12/29/MESSAGING-1364-image-4.jpeg" width="1288" /></p> 
<h2>Create security group</h2> 
<p>Complete the following steps to create a security group:</p> 
<ol> 
 <li>On the Amazon VPC console, under <strong>Security</strong> in the navigation pane, choose <strong>Security groups</strong>.</li> 
 <li>Choose <strong>Create security group</strong>.</li> 
 <li>For <strong>Security group name</strong>, provide a name that uniquely identifies the security group. For this example, we name the security group <code>my-sg-mail-manager</code>.</li> 
 <li>For <strong>Description</strong>, describe the purpose of this security group.</li> 
 <li>For <strong>VPC</strong>, choose the VPC that hosts your applications.</li> 
 <li>For <strong>Inbound rules</strong>, choose <strong>Add rule</strong>.</li> 
 <li>For <strong>Type</strong>, choose <strong>SMTP</strong>.</li> 
 <li>For <strong>Source</strong>, enter the IP range of your private subnet.</li> 
 <li>Choose <strong>Add rule</strong> again.</li> 
 <li>For <strong>Port range</strong>, enter <code>587</code> and the IP range of your private subnet.</li> 
 <li>Choose <strong>Create security group</strong>.</li> 
</ol> 
<p><img alt="" class="alignnone size-full wp-image-7278" height="1025" src="https://d2908q01vomqb2.cloudfront.net/632667547e7cd3e0466547863e1207a8c0c0c549/2025/12/29/MESSAGING-1364-image-5.jpg" width="1505" /></p> 
<h2>Create VPC endpoint</h2> 
<p>VPC endpoints make it possible to keep your email traffic within your private AWS network. Complete the following steps to create a VPC endpoint:</p> 
<ol> 
 <li>Open the Amazon VPC console in the target Region.</li> 
 <li>Under <strong>PrivateLink and Lattice</strong> in the navigation pane, choose <strong>Endpoints</strong>.</li> 
 <li>Choose <strong>Create endpoint</strong>.</li> 
 <li>For <strong>Name tag</strong>, enter an optional tag, such as <code>mm-vpce-auth-ingress-endpoint</code>.</li> 
 <li>Select <strong>AWS services</strong>.</li> 
 <li>For <strong>Services</strong>, enter <code>mail-manager</code> to search for Amazon SES Mail Manager VPC endpoints.</li> 
 <li>Select <code>com.amazonaws.us-east-1.mail-manager-smtp.auth.fips</code>.</li> 
</ol> 
<p><img alt="" class="alignnone size-full wp-image-7279" height="649" src="https://d2908q01vomqb2.cloudfront.net/632667547e7cd3e0466547863e1207a8c0c0c549/2025/12/29/MESSAGING-1364-image-6.jpeg" width="1290" /></p> 
<ol start="8"> 
 <li>For <strong>VPC</strong>, choose the VPC that hosts your applications.</li> 
 <li>For <strong>DNS name</strong>, select <strong>Enable DNS name</strong></li> 
 <li>For <strong>DNS record IP type</strong>, select <strong>IPv4</strong>.</li> 
 <li>Under <strong>Subnets</strong>, select all <strong>Availability Zones</strong> and choose the subnet ID for each subnet.</li> 
 <li>For <strong>IP type</strong>, select <strong>IPv4</strong>.</li> 
</ol> 
<p><img alt="" class="alignnone size-full wp-image-7280" height="679" src="https://d2908q01vomqb2.cloudfront.net/632667547e7cd3e0466547863e1207a8c0c0c549/2025/12/29/MESSAGING-1364-image-7.jpeg" width="1287" /></p> 
<ol start="13"> 
 <li>For <strong>Security groups</strong>, select the group <code>my-sg-mail-manager</code>.</li> 
 <li>Choose <strong>Create endpoint</strong>.</li> 
</ol> 
<p><img alt="" class="alignnone size-full wp-image-7281" height="376" src="https://d2908q01vomqb2.cloudfront.net/632667547e7cd3e0466547863e1207a8c0c0c549/2025/12/29/MESSAGING-1364-image-8.jpeg" width="1287" /></p> 
<h2>Create ingress endpoint</h2> 
<p>Complete the following steps to create an authenticated ingress endpoint using Secrets Manager and AWS KMS:</p> 
<ol> 
 <li>On the Amazon SES console, under <strong>Amazon SES Mail Manager</strong> in the navigation pane, choose <strong>Ingress endpoints</strong>.</li> 
 <li>Choose <strong>Create ingress endpoint</strong>.</li> 
 <li>For <strong>Ingress endpoint name</strong>, enter a unique name for the ingress endpoint. For this example, we use <code>my-authenticated-ingress-endpoint</code>.</li> 
 <li>For <strong>Type</strong>, choose <strong>Authenticated</strong>.</li> 
 <li>For <strong>Authentication type</strong>, choose <strong>Secret</strong>.</li> 
 <li>Choose <strong>Create new</strong>, which will open a new tab.</li> 
 <li>For <strong>Secret type</strong>, choose <strong>Other type of secret</strong>.</li> 
 <li>Under <strong>Key/value pairs</strong>, enter <code>password</code> as the key (anything else will cause authentication to fail), then enter a password as the value.</li> 
 <li>For <strong>Encryption Key</strong>, choose <strong>Add new key</strong>, which will open a new tab.</li> 
 <li>Choose <strong>Create key</strong>.</li> 
 <li>Keep the default values for <strong>Key type</strong> and <strong>Key usage</strong> and choose <strong>Next</strong>.</li> 
</ol> 
<p><img alt="" class="alignnone wp-image-7282 size-full" height="535" src="https://d2908q01vomqb2.cloudfront.net/632667547e7cd3e0466547863e1207a8c0c0c549/2025/12/29/MESSAGING-1364-image-9.jpg" width="1650" /></p> 
<ol start="12"> 
 <li>For <strong>Alias</strong>, enter a unique name for your custom managed key. For this example, we use <code>my-mail-manager-key</code>.</li> 
 <li>For <strong>Description</strong>, describe the purpose of the key.</li> 
 <li>Choose <strong>Next</strong>.</li> 
</ol> 
<p><img alt="" class="alignnone size-full wp-image-7283" height="563" src="https://d2908q01vomqb2.cloudfront.net/632667547e7cd3e0466547863e1207a8c0c0c549/2025/12/29/MESSAGING-1364-image-10.jpeg" width="1290" /></p> 
<ol start="15"> 
 <li>For <strong>Key administrators</strong>, choose any users (other than yourself) or roles you want to permit to administer the key, then choose <strong>Next</strong>.</li> 
 <li>For <strong>Key users</strong>, choose any users (other than yourself) or roles you want to permit to use the key, then choose <strong>Next</strong>.</li> 
 <li>For <strong>Key policy</strong>, choose <strong>Edit</strong>, then enter the following KMS key policy into the key policy JSON text editor at the <code>"statement"</code> level by adding it as an additional statement separated by a comma. Replace the Region and account number with your own.</li> 
</ol> 
<div class="hide-language"> 
 <pre><code class="lang-css">{
&nbsp;&nbsp; &nbsp;"Effect": "Allow",
&nbsp;&nbsp; &nbsp;"Principal": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;"Service": "ses.amazonaws.com"
&nbsp;&nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp;"Action": "kms:Decrypt",
&nbsp;&nbsp; &nbsp;"Resource": "*",
&nbsp;&nbsp; &nbsp;"Condition": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;"StringEquals": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "kms:ViaService": "secretsmanager.us-east-1.amazonaws.com",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"aws:SourceAccount": "000000000000"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;"ArnLike": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"aws:SourceArn": "arn:aws:ses:us-east-1:000000000000:mailmanager-ingress-point/*"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp;}
}</code></pre> 
</div> 
<ol start="18"> 
 <li>Choose <strong>Next</strong>.</li> 
 <li>Review and choose <strong>Finish</strong>.</li> 
 <li>Switch to the Secrets Manager tab and choose the refresh icon.</li> 
 <li>Choose the KMS key you just created, then choose <strong>Next</strong>.</li> 
</ol> 
<p><img alt="" class="alignnone size-full wp-image-7284" height="554" src="https://d2908q01vomqb2.cloudfront.net/632667547e7cd3e0466547863e1207a8c0c0c549/2025/12/29/MESSAGING-1364-image-11.jpeg" width="1286" /></p> 
<ol start="22"> 
 <li>For <strong>Secret name</strong>, provide a unique name for the secret. For this example, we use <code>my-mail-manager-secret</code>.</li> 
 <li>For <strong>Description</strong>, describe the purpose for the secret.</li> 
 <li>For <strong>Resource permissions</strong>, replace the example JSON code in the editor with the following policy. Replace the Region and the account number with your own.</li> 
</ol> 
<div class="hide-language"> 
 <pre><code class="lang-css">{
&nbsp;&nbsp; &nbsp;"Version": "2012-10-17",
&nbsp;&nbsp; &nbsp;"Id": "Id",
&nbsp;&nbsp; &nbsp;"Statement": [
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;{
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Effect": "Allow",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Principal": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Service": "ses.amazonaws.com"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Action": "secretsmanager:GetSecretValue",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Resource": "*",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Condition": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"StringEquals": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"aws:SourceAccount": "000000000000"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"ArnLike": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"aws:SourceArn": "arn:aws:ses:us-east-1:000000000000:mailmanager-ingress-point/*"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp;]
}</code></pre> 
</div> 
<ol start="25"> 
 <li>Choose <strong>Save</strong>, then choose <strong>Next</strong>.</li> 
 <li>Configuring automatic rotation is optional. We skip this step and choose <strong>Next</strong>.</li> 
 <li>Review and choose <strong>Store</strong>.</li> 
 <li>Switch back to the Amazon SES console tab to finish creating the ingress endpoint.</li> 
 <li>For <strong>Secret ARN</strong>, choose <strong>Refresh list</strong>, then choose the secret you just created.</li> 
 <li>For <strong>Rule set</strong>, choose <code>first-rule-set</code>.</li> 
 <li>For <strong>Traffic policy</strong>, choose <code>first-traffic-policy</code>.</li> 
 <li>For <strong>Network type</strong>, select <strong>Private</strong>.</li> 
 <li>For <strong>VPC endpoint ID</strong>, choose <code>mm-vpce-auth-ingress-endpoint</code>.</li> 
 <li>Choose <strong>Create ingress endpoint</strong>.</li> 
</ol> 
<p><img alt="" class="alignnone size-full wp-image-7285" height="766" src="https://d2908q01vomqb2.cloudfront.net/632667547e7cd3e0466547863e1207a8c0c0c549/2025/12/29/MESSAGING-1364-image-12.jpeg" width="1286" /></p> 
<h2>Test configuration</h2> 
<p>Complete the following steps to test your configuration:</p> 
<ol> 
 <li>Open the Amazon VPC console in the target Region.</li> 
 <li>Under <strong>PrivateLink and Lattice</strong> in the navigation pane, choose <strong>Endpoints</strong>.</li> 
 <li>Choose the VPC endpoint ID of <code>mm-vpce-auth-ingress-endpoint</code> to open the details page.</li> 
 <li>Find the DNS names for the VPC endpoint. The first DNS name on this list is the Regional DNS name of the VPC endpoint; copy this DNS name and save it on a notepad for later use.</li> 
</ol> 
<p><img alt="" class="alignnone size-full wp-image-7286" height="540" src="https://d2908q01vomqb2.cloudfront.net/632667547e7cd3e0466547863e1207a8c0c0c549/2025/12/29/MESSAGING-1364-image-13.jpeg" width="1290" /></p> 
<ol start="5"> 
 <li>Open the Amazon SES console in the target Region.</li> 
 <li>Under <strong>Amazon SES Mail Manager</strong> in the navigation pane, choose <strong>Ingress endpoints</strong>.</li> 
 <li>Choose <code>my-authenticated-ingress-endpoint</code>.</li> 
 <li>In the <strong>Authentication </strong>section, locate the SMTP user name (typically starts with <code>inp-</code>).</li> 
</ol> 
<p><img alt="" class="alignnone size-full wp-image-7287" height="569" src="https://d2908q01vomqb2.cloudfront.net/632667547e7cd3e0466547863e1207a8c0c0c549/2025/12/29/MESSAGING-1364-image-14.jpeg" width="1290" /></p> 
<ol start="9"> 
 <li>Connect to your EC2 instance using SSH.</li> 
 <li><a href="https://docs.aws.amazon.com/ses/latest/dg/send-email-smtp-client-command-line.html#send-email-using-openssl" rel="noopener noreferrer" target="_blank">Use the command line to send an email using the Amazon SES SMTP interface</a> to test the connectivity. Replace the endpoint with the DNS name of the VPC endpoint you copied earlier.</li> 
</ol> 
<p><img alt="" class="alignnone size-full wp-image-7288" height="789" src="https://d2908q01vomqb2.cloudfront.net/632667547e7cd3e0466547863e1207a8c0c0c549/2025/12/29/MESSAGING-1364-image-15.jpeg" width="1289" /></p> 
<p><img alt="" class="alignnone size-full wp-image-7289" height="771" src="https://d2908q01vomqb2.cloudfront.net/632667547e7cd3e0466547863e1207a8c0c0c549/2025/12/29/MESSAGING-1364-image-16.jpeg" width="1289" /></p> 
<p>The message <code>250 OK esllb73q6bd94cnq004ujd544f169sog39bc9ug1</code> indicates the message was successfully accepted by Amazon SES Mail Manager.</p> 
<h2>Clean up</h2> 
<p>When you’re done with this solution, clean up the resources you created including Mail Manager configurations, security groups, VPC endpoints, KMS keys, and Secrets Manager secrets to avoid additional charges.</p> 
<h2>Conclusion</h2> 
<p>In this post, we showed how to enhance email security by implementing Amazon SES Mail Manager with VPC endpoints. This solution can help you modernize your email infrastructure while maintaining network-level isolation and meeting enterprise compliance requirements.</p> 
<p>To learn more about Amazon SES, see the <a href="https://docs.aws.amazon.com/ses/latest/dg/Welcome.html" rel="noopener noreferrer" target="_blank">Amazon SES Developer Guide</a>. For additional security best practices, refer to <a href="https://aws.amazon.com/architecture/security-identity-compliance/" rel="noopener noreferrer" target="_blank">AWS Best Practices for Security, Identity, &amp; Compliance</a>. To get started using Amazon SES Mail Manager, participate in an <a href="https://catalog.workshops.aws/migrate-w-mailmanager" rel="noopener noreferrer" target="_blank">Amazon SES Mail Manager workshop</a> event, explore the advanced workflow features of Amazon SES Mail Manager, and consider integrating with your existing monitoring and alerting systems.</p> 
<hr /> 
<h3>About the authors</h3>
