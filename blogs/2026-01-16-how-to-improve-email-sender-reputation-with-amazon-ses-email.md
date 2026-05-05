---
title: "How to improve email sender reputation with Amazon SES Email Validation"
url: "https://aws.amazon.com/blogs/messaging-and-targeting/how-to-improve-email-sender-reputation-with-amazon-ses-email-validation/"
date: "Fri, 16 Jan 2026 15:27:23 +0000"
author: "Zip Zieper"
feed_url: "https://aws.amazon.com/blogs/messaging-and-targeting/feed/"
---
<p>If you’re sending emails at scale with <a href="https://aws.amazon.com/ses" rel="noopener noreferrer" target="_blank">Amazon Simple Email Service (Amazon SES)</a>, maintaining high deliverability depends on more than the content you send. It’s about who receives those emails. Mailbox providers like Gmail, Yahoo, and Outlook assign reputation scores based on your sending practices, domain and IP authentication records, message quality, and recipient engagement. These providers use their own algorithms to decide whether your emails reach the inbox, are filtered as spam, or aren’t delivered at all. For more information about managing your email reputation, see&nbsp;<a href="https://aws.amazon.com/blogs/messaging-and-targeting/the-four-pillars-of-email-reputation/" rel="noopener noreferrer" target="_blank">The Four Pillars of Managing Email Reputation</a>. In this post, we show you how the Amazon SES Email Validation feature can help you to protect your sender reputation.</p> 
<p>The email bounce rate is the percentage of emails that fail to deliver and is one of the most critical factors affecting your sender reputation. Every bounce damages your sender reputation. Mailbox providers like Gmail and Outlook closely monitor bounce rates, and accounts that bounce over 5% trigger warnings. If your account bounce rate exceeds 10%, the email services providers might throttle, or completely block sending. For customers sending email at scale with Amazon SES, a high bounce rate may trigger immediate consequences: damaged sender reputation, blocked deliverability, and ISP penalties that can throttle or suspend your entire email program. Traditional approaches to email quality are reactive, because you will only discover problems after bounces have damaged your reputation. While account suppression lists protect against known problematic addresses, they can’t protect you from the normal decay of email address quality that occurs because of job changes, abandoned mailboxes, domain expirations, or bots and bad actors looking to damage your email reputation.</p> 
<h2>Use Amazon SES Email Validation to help you protect your sender reputation</h2> 
<p><a href="https://docs.aws.amazon.com/ses/latest/dg/email-validation.html" rel="noopener noreferrer" target="_blank">Amazon SES Email Validation</a> shifts bounce management from reactive to proactive, helping you detect problems before they damage your sender reputation. The feature provides two validation approaches: the <a href="https://docs.aws.amazon.com/ses/latest/dg/email-validation-api.html" rel="noopener noreferrer" target="_blank">Email Validation API</a> for&nbsp;timely checks during registration and <a href="https://docs.aws.amazon.com/ses/latest/dg/email-validation-auto.html" rel="noopener noreferrer" target="_blank">Auto Validation</a> to automatically review all outbound email addresses before sending&nbsp;and only deliver messages to recipients that meet your selected validation threshold. Both methods are intended to catch problem addresses before they become bounces, helping to protect your sender reputation.</p> 
<p>In this post, we guide you through implementing both validation approaches using AnyCompany—a fictitious ecommerce website—as our example. You will see how AnyCompany might use the Email Validation API for timely registration checks on address acquisition and Auto Validation at time of sending. You’ll learn how to protect your sender reputation proactively and integrate validation into existing workflows with minimal disruption. We also show you how to use <a href="https://aws.amazon.com/cloudwatch" rel="noopener noreferrer" target="_blank">Amazon CloudWatch</a> metrics to improve email list health over time. After you’re done reading and experimenting, you’ll understand how Amazon SES Email Validation can help transform your email operations from reactive bounce management to proactive quality assurance.</p> 
<h2>Solution overview – how to use the Email Validation API to avoid ingesting invalid email addresses</h2> 
<p>You can use the Email Validation API&nbsp;to validate email addresses through synchronous API calls to check addresses at the point of collection. This method gives you immediate feedback about address validity and helps prevent invalid addresses from entering your database. You control when validation occurs and how to handle the results. The Email Validation API costs $0.01 per validation using the API or the AWS Management Console for Amazon SES. See&nbsp;<a href="https://aws.amazon.com/ses/pricing/" rel="noopener noreferrer" target="_blank">Amazon SES pricing</a> for details.</p> 
<p>The Amazon SES console&nbsp;uses the&nbsp;Email Validation API to manually validate up to 10 email addresses at a time. The results are shown in the console—shown in the following screenshot—and you can export the results to a CSV file.</p> 
<p><img alt="" class="alignnone size-full wp-image-7331" height="580" src="https://d2908q01vomqb2.cloudfront.net/632667547e7cd3e0466547863e1207a8c0c0c549/2026/01/15/mess-1393-image-1.jpg" width="1308" /></p> 
<p>The&nbsp;Email Validate API can be used in your code or using the <a href="https://aws.amazon.com/cli" rel="noopener noreferrer" target="_blank">AWS Command Line Interface (AWS CLI)</a> to validate individual email addresses through synchronous API calls. This method is&nbsp;well-suited for validating addresses at the point of collection—during user registration, subscription form submission, or during an email list import to help prevent invalid addresses from entering your database. The following is an example using the AWS CLI.</p> 
<div class="hide-language"> 
 <pre><code class="lang-powershell">aws sesv2 get-email-address-insights \
&nbsp; &nbsp;&nbsp;--email-address user@example.com \
&nbsp;&nbsp;&nbsp;&nbsp;--region us-east-1</code></pre> 
</div> 
<p>The API returns a response structure similar to the following example:</p> 
<div class="hide-language"> 
 <pre><code class="lang-css">{
&nbsp;&nbsp;"MailboxValidation": {
&nbsp;&nbsp; &nbsp;"IsValid": {
&nbsp;&nbsp; &nbsp; &nbsp;"ConfidenceVerdict": "HIGH"
&nbsp;&nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp;"Evaluations": {
&nbsp;&nbsp; &nbsp; &nbsp;"HasValidSyntax": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;"ConfidenceVerdict": "HIGH"
&nbsp;&nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp;"HasValidDnsRecords": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;"ConfidenceVerdict": "MEDIUM"
&nbsp;&nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp;"MailboxExists": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;"ConfidenceVerdict": "MEDIUM"
&nbsp;&nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp;"IsRoleAddress": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;"ConfidenceVerdict": "LOW"
&nbsp;&nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp;"IsDisposable": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;"ConfidenceVerdict": "LOW"
&nbsp;&nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp;"IsRandomInput": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;"ConfidenceVerdict": "LOW"
&nbsp;&nbsp; &nbsp; &nbsp;}
&nbsp;&nbsp; &nbsp;}
&nbsp;&nbsp;}
}</code></pre> 
</div> 
<h4>Understanding Email Validation API verdicts</h4> 
<p>For each email, the&nbsp;Email Validation API returns an <em>overall validity confidence</em> with&nbsp;three possible aggregate <em>verdicts</em>:</p> 
<ul> 
 <li><strong>HIGH</strong> – The email address passed all critical validation checks and is highly likely to be deliverable. These addresses can be accepted without additional scrutiny.</li> 
 <li><strong>MEDIUM</strong> – The email address passed basic validation but has characteristics that might affect deliverability (such as being a role address or having uncertain mailbox existence). Your use case and bounce risk tolerance should be used to determine whether to accept these addresses.</li> 
 <li><strong>LOW</strong> – The email address failed one or more critical validation checks and is unlikely to be deliverable.&nbsp;Your use case and bounce risk tolerance will most likely cause you to reject these addresses.</li> 
</ul> 
<p>To reach the <em>overall validity confidence</em>, the Email Validation API performs six detailed checks on each email address:</p> 
<ul> 
 <li><strong>Syntax validation (</strong><code>HasValidSyntax</code><strong>)</strong> – Confirms the address follows RFC 5321 and RFC 5322 standards for email address formatting. This catches obvious errors such as missing @ symbols or invalid characters.</li> 
 <li><strong>DNS verification (</strong><code>HasValidDnsRecords</code><strong>)</strong> – Validates that the domain exists and has proper mail exchange (MX) records and corresponding A records configured. This helps confirm that the domain can receive email.</li> 
 <li><strong>Mailbox existence</strong>&nbsp;(<code>MailboxExists</code>) – Predicts whether the specific mailbox exists and can receive messages.</li> 
 <li><strong>Role address detection</strong>&nbsp;(<code>IsRoleAddress</code>) – Identifies generic addresses like&nbsp;<em>sysadmin@anycompany.com</em>&nbsp;or&nbsp;<em>support@anycompany.com</em> that typically represent shared mailboxes rather than individual recipients.</li> 
 <li><strong>Disposable email detection</strong>&nbsp;(<code>IsDisposable</code>) – Checks temporary email services like <a href="http://mailinator.com" rel="noopener noreferrer" target="_blank">mailinator.com</a> or <a href="http://guerrillamail.com" rel="noopener noreferrer" target="_blank">guerrillamail.com</a> that users often employ to avoid providing real contact information.</li> 
 <li><strong>Random input detection</strong>&nbsp;(<code>IsRandomInput</code>) –&nbsp;Checks randomly generated patterns.</li> 
</ul> 
<p>For more information about response values and data types, see the <a href="https://alpha.www.docs.aws.a2z.com/ses/latest/APIReference-V2/API_MailboxValidation.html" rel="noopener noreferrer" target="_blank"><code>MailboxValidation</code></a> data type in the <a href="https://docs.aws.amazon.com/ses/latest/APIReference-V2/Welcome.html" rel="noopener noreferrer" target="_blank">Amazon SES API v2 reference</a>.</p> 
<p>The&nbsp;Email Validation API provides a dashboard in the Amazon SES console that you can use to view email address verification results over time, with the ability to look back for up to one month, as shown in the following screenshot.</p> 
<p><img alt="" class="alignnone size-full wp-image-7332" height="870" src="https://d2908q01vomqb2.cloudfront.net/632667547e7cd3e0466547863e1207a8c0c0c549/2026/01/15/mess-1393-image-2.png" width="1428" /></p> 
<h3>Use auto validation to help prevent bounces when sending from Amazon SES</h3> 
<p>Amazon SES Auto Validation automatically performs comprehensive address validation through multiple checks such as syntax validation, DNS records, and others before each message is sent. When auto validation is enabled, Amazon SES will only deliver messages to recipients that meet your selected validation threshold. This helps you protect your sender reputation by preventing sends to addresses that have a high probability of being invalid or risky without requiring manual intervention or API integration. Auto Validation must be enabled separately for each AWS region in your account. For example, if you enable it in us-east-1, it will not be active in us-west-2 unless you explicitly enable it there as well. You can enable it at the account level for an entire region, or selectively within <a href="https://docs.aws.amazon.com/ses/latest/dg/managing-configuration-sets.html" rel="noopener noreferrer" target="_blank">configuration sets</a>. Auto validation costs $0.01 per 1,000 validations.&nbsp;Be aware that sends suppressed by Auto Validation count towards your daily send quota, and you will be charged the standard outgoing message fee for suppressed sends (in addition to the fee for auto validation). See <a href="https://aws.amazon.com/ses/pricing/" rel="noopener noreferrer" target="_blank">Amazon SES pricing</a> for more information.</p> 
<p>When enabled at the AWS account level, you set the&nbsp;<em>Validation threshold</em> to&nbsp;determine which email addresses to suppress based on their validity confidence, as shown in the following screenshot.</p> 
<p><img alt="" class="alignnone size-full wp-image-7333" height="424" src="https://d2908q01vomqb2.cloudfront.net/632667547e7cd3e0466547863e1207a8c0c0c549/2026/01/15/mess-1393-image-3.jpg" width="1076" /></p> 
<ul> 
 <li><strong>Amazon SES managed threshold (recommended)</strong> –&nbsp;Amazon SES automatically manages the threshold to suppress invalid addresses based on your sending patterns and reputation. This option allows Amazon SES to optimize the validation threshold dynamically. Use this threshold when you want AWS to handle validation decisions based on your account’s specific characteristics.</li> 
 <li><strong>Custom threshold – </strong> 
  <ul> 
   <li><strong>High</strong> – Delivers emails only to addresses with high delivery likelihood. This provides maximum protection for your sender reputation but might suppress some legitimate addresses with medium delivery confidence. Use this threshold for critical transactional emails or when protecting sender reputation is your top priority.</li> 
   <li><strong>Medium</strong> – Delivers emails to addresses with medium or high delivery likelihood. This balances reputation protection with delivery reach by allowing addresses with moderate deliverability scores. Use this threshold for marketing campaigns where you want to maximize reach while still filtering obviously invalid addresses.</li> 
  </ul> </li> 
</ul> 
<p>You will usually find that using the recommended Amazon SES managed threshold works best for the bulk of your sending, however for certain use cases&nbsp;you might want to override the account setting and use a custom threshold in your configuration set.&nbsp;If you choose <strong>High</strong> or <strong>Medium</strong> thresholds instead of Amazon SES managed (as shown in the following screenshot), it’s important that you monitor your delivery metrics and validation results regularly.</p> 
<p><img alt="" class="alignnone size-full wp-image-7334" height="789" src="https://d2908q01vomqb2.cloudfront.net/632667547e7cd3e0466547863e1207a8c0c0c549/2026/01/15/mess-1393-image-4.jpg" width="1064" /></p> 
<p>Auto Validation applies to all outbound emails sent through your account. Addresses that don’t meet your threshold will be suppressed with the <code>bounceSubType</code> of <code>EmailValidationSuppressed</code>. Suppressed sends count towards your daily send quota, and you will be charged the standard outgoing message fee for suppressed sends in addition to the fee for auto validation.</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">{
  "Type": "Notification",
  "MessageId": "0ded6fd6-4e59-5ae0-9782-0e68faa886e7",
  "TopicArn": "arn:aws:sns:us-east-1:252640393490:ses-auto-validate",
  "Subject": "Amazon SES Email Event Notification",
  "Message": "{\"**eventType**\":\"**Bounce**\",\"bounce\":{\"feedbackId\":\"0100019b345a05a0-95e3062a-9594-499f-aafc-dc2dc9647cb6-000000\",\"**bounceType**\":\"**Permanent**\",\"**bounceSubType**\":\"**EmailValidationSuppressed**\",\"bouncedRecipients\":"
}</code></pre> 
</div> 
<h2>How AnyCompany might use the Email Validation API and auto validation</h2> 
<p>AnyCompany runs an ecommerce platform for both business and consumer office supplies. The company’s website hosts various web-forms for customers to create accounts and sign up to receive newsletters and discount offers. When they place orders through the platform, AnyCompany’s system sends order confirmations and delivery tracking emails.&nbsp;Today, when a new user registers through one of the web forms, AnyCompany sends a verification email to confirm the user’s email address, contact details, and opt-in to the company’s emails. If a user misspells their email address, they will never receive this verification email. Frustrated, they might&nbsp;move on to another provider. Similarly, if a bot or bad actor deliberately submits invalid addresses to the web form, verification emails will bounce. Both scenarios cost AnyCompany money with no return; at scale, a high bounce rate might cause email service providers to throttle or block future sends.&nbsp;AnyCompany previously investigated various third-party email validation services, but the engineering work, security reviews, and costs outweighed the expected benefits. This necessitated the cloud team’s constant and careful vigilance over the company’s bounce rate and reputation, diverting resources that the company would prefer to deploy elsewhere.&nbsp;As an&nbsp;ecommerce company, AnyCompany needs to be highly protective of its sender reputation. With email validation now built directly into Amazon SES, AnyCompany can bypass the complexity and cost of third-party tools and directly benefit from proactive bounce prevention across all email use cases. In the following section, we guide you through the simple steps AnyCompany might take to implement Amazon SES Email Validation.</p> 
<h2>Prerequisites</h2> 
<p>Before implementing Email Validation, you’ll need:</p> 
<p><strong>AWS account :</strong></p> 
<ul> 
 <li>An AWS account with Amazon SES enabled in your desired AWS Region</li> 
 <li>AWS CLI version 2.0 or later installed and configured</li> 
</ul> 
<p><strong>Required IAM permissions</strong>:</p> 
<p>Your IAM user or role needs the following permissions to configure Email Validation:</p> 
<ul> 
 <li><code>ses:PutAccountSuppressionAttributes</code>&nbsp;– To enable and configure Email Validation at the account level</li> 
 <li><code>ses:GetAccount</code> to verify Email Validation configuration</li> 
 <li><code>ses:CreateConfigurationSet</code>&nbsp;when creating a new configuration set</li> 
 <li><code>ses:PutConfigurationSetSuppressionOptions</code>to override validation settings for specific configuration sets</li> 
 <li><code>ses:GetEmailAddressInsights</code>&nbsp;to call the Email Validation API</li> 
 <li><code>iam:CreateServiceLinkedRole</code>&nbsp;&nbsp;creates an <a href="https://aws.amazon.com/blogs/security/introducing-an-easier-way-to-delegate-permissions-to-aws-services-service-linked-roles/" rel="noopener noreferrer" target="_blank">IAM service-linked role</a> that is used by Amazon SES to publish CloudWatch metrics</li> 
 <li><code>cloudwatch:GetMetricStatistics</code> – To retrieve validation metrics</li> 
</ul> 
<p><strong>Development environment:</strong></p> 
<ul> 
 <li>Familiarity with AWS CLI commands and JSON configuration files</li> 
 <li>For API integration, SDK support for Amazon SES API v2 in your preferred programming language</li> 
 <li>Access to your application’s user registration code</li> 
</ul> 
<p><strong>Existing Amazon SES configuration:</strong></p> 
<ul> 
 <li>At least one verified email address or domain in Amazon SES</li> 
</ul> 
<p>While the Email Validation API works with an AWS account that is in the Amazon SES sandbox, auto validation is best demonstrated after your AWS account has been granted production access.</p> 
<ul> 
 <li>(Optional) Configuration sets created for different email types (transactional, marketing, and so on)</li> 
</ul> 
<h2>Validating email addresses at acquisition with the Email Validation API</h2> 
<p>To prevent bad addresses from entering their customer database at time of acquisition, AnyCompany will integrate the Email Validation API directly into their registration form. When a user submits their contact details, including email address, the&nbsp;Email Validation API is used. Results arrive within 100&nbsp;milliseconds, with the overall validity confidence and<strong>&nbsp;</strong>six detailed checks of the email address.&nbsp;AnyCompany will then use the results and custom business logic they designed for their different use cases. For example, for their B2B business, they might allow registrations with high overall confidences and a role address, such as <code>billing@anycompany.example</code>. For the consumer business, they might accept registrations with medium overall confidences, but&nbsp;always reject emails that the API identifies as disposable, such as&nbsp;<code>vw92h7+575f4mhhy5wk@example.com</code>&nbsp;or random, such as <code>sfdoihjofsdi@example.com</code>.</p> 
<h3>Sample code snippets</h3> 
<p>The code snippets in this section are examples only and are not intended for production use.</p> 
<p><strong>Step 1: Validate email address on form submission</strong></p> 
<p>When a user submits the registration form, AnyCompany’s application calls the Email Validation API before creating the account:</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">import boto3

ses_client = boto3.client('sesv2', region_name='us-east-1')

def validate_registration_email(email_address):
&nbsp;&nbsp; &nbsp;try:
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;response = ses_client.get_email_address_insights(
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;EmailAddress=email_address
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;)
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;return response
&nbsp;&nbsp; &nbsp;except Exception as e:
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;# Handle API errors gracefully
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;print(f"Validation error: {e}")
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;return None</code></pre> 
</div> 
<p><strong>Step 2: Apply business rules based on verdict</strong></p> 
<p>AnyCompany’s business logic handles different validation outcomes:</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">def should_accept_email(validation_response, registration_type):
&nbsp;&nbsp; &nbsp;if not validation_response:
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;# API error - accept email but flag for manual review
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;return True, "accepted_with_warning"

&nbsp; &nbsp; overall_verdict = validation_response['MailboxValidation']['IsValid']['ConfidenceVerdict']
&nbsp;&nbsp; &nbsp;checks = validation_response['MailboxValidation']['Evaluations']

&nbsp;&nbsp; &nbsp;# Always reject FAIL verdicts
&nbsp;&nbsp; &nbsp;if&nbsp;overall_verdict == 'LOW':
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;return False, "rejected_invalid"

&nbsp;&nbsp; &nbsp;# Always accept PASS verdicts
&nbsp;&nbsp; &nbsp;if&nbsp;overall_verdict == 'HIGH':
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;return True, "accepted"

&nbsp;&nbsp; &nbsp;# Handle NEUTRAL verdicts based on registration type
&nbsp;&nbsp; &nbsp;if&nbsp;overall_verdict == 'MEDIUM':
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;# Reject disposable emails for all registration types
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;if checks['IsDisposable']['ConfidenceVerdict'] == 'HIGH':
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;return False, "rejected_disposable"

&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;# Accept role addresses for B2B, reject for consumer
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;if checks['IsRoleAddress']['ConfidenceVerdict'] == 'HIGH':
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;if registration_type == 'b2b':
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;return True, "accepted_role_address"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;else:
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;return False, "rejected_role_address"

&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;# Accept other NEUTRAL cases with warning
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;return True, "accepted_with_warning"</code></pre> 
</div> 
<p><strong>Step 3: Provide user-friendly error messages</strong></p> 
<p>When validation fails, AnyCompany provides specific, actionable feedback (you can add more conditions based on your requirements):</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">def get_user_error_message(validation_response):
&nbsp;&nbsp; &nbsp;checks = validation_response['Evaluations']

&nbsp;&nbsp; &nbsp;if checks['HasValidSyntax']['ConfidenceVerdict'] == 'LOW':
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;return "Please check your email address for typos. It appears to have formatting errors."

&nbsp;&nbsp; &nbsp;if checks['HasValidDnsRecords']['ConfidenceVerdict'] == 'LOW':
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;return "The domain in your email address doesn't appear to exist. Please verify you entered it correctly."

&nbsp;&nbsp; &nbsp;if checks['IsDisposable']['ConfidenceVerdict'] == 'HIGH':
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;return "Temporary email addresses are not accepted. Please use a different&nbsp;email address."

&nbsp;&nbsp; &nbsp;if checks['IsRoleAddress']['ConfidenceVerdict'] == 'HIGH':
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;return "Please use a personal email address rather than a shared mailbox like support@ or admin@."

&nbsp;&nbsp; &nbsp;return "We couldn't verify this email address. Please check for typos and try again."</code></pre> 
</div> 
<p><strong>Step 4: Suggest corrections for common mistakes</strong></p> 
<p>For addresses that fail DNS validation, AnyCompany suggests common corrections to popular domain typos.</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">def suggest_email_correction(email_address):
&nbsp;&nbsp; &nbsp;common_domains = {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;'gmial.com': 'gmail.com',
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;'gmai.com': 'gmail.com',
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;'yahooo.com': 'yahoo.com',
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;'hotmial.com': 'hotmail.com',
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;'outlok.com': 'outlook.com'
&nbsp;&nbsp; &nbsp;}

&nbsp;&nbsp; &nbsp;# Extract domain from email
&nbsp;&nbsp; &nbsp;if '@' in email_address:
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;local, domain = email_address.split('@', 1)

&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;# Check for common misspellings
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;if domain.lower() in common_domains:
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;suggested_domain = common_domains[domain.lower()]
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;return f"{local}@{suggested_domain}"

&nbsp;&nbsp; &nbsp;return None</code></pre> 
</div> 
<h3>Key benefits of the Email Validation API</h3> 
<p>The Email Validation API provide proactive quality control by preventing invalid addresses from entering AnyCompany’s database, preventing the reputation damage that occurs when they send to addresses that bounce.</p> 
<ul> 
 <li><strong>Immediate user feedback</strong> – Because validation results return within milliseconds, AnyCompany can provide real-time feedback during registration without impacting user experience.</li> 
 <li><strong>Flexible policy enforcement</strong> – AnyCompany can use individual check results to define custom validation policies that match their various business requirements, accepting or rejecting addresses based on use case-specific risk tolerance.</li> 
 <li><strong>Cost-effective validation</strong> –&nbsp;AnyCompany pays only for the addresses they validate, with no infrastructure to provision or manage and no license fees. Preventing a single bounce might save more than the cost of validation.</li> 
</ul> 
<p>By integrating the Email Validation API into their registration workflow, AnyCompany can transform their approach from reactive bounce management to proactive quality assurance. Invalid addresses are prevented from entering their database, legitimate customers receive verification emails reliably, and their sender reputation remains protected with little to no ongoing effort.</p> 
<h3>Set up a CloudWatch alarm for high rates of LOW verdicts</h3> 
<p>You can configure CloudWatch alarms to notify you when validation patterns indicate a consistently high rate of LOW verdicts. This might indicate malicious bots attempting to sign up through a web-form or other mechanism.</p> 
<p>The following example creates a CloudWatch alarm that fires when the rate of LOW verdicts exceeds 20%.</p> 
<div class="hide-language"> 
 <pre><code class="lang-css">aws cloudwatch put-metric-alarm \
&nbsp;&nbsp;--region us-east-1 \
&nbsp;&nbsp;--alarm-name "EmailInsights-LOW-Rate-Above-20-Percent" \
&nbsp;&nbsp;--alarm-description "Alarm when LOW confidence verdict rate exceeds 20%" \
&nbsp;&nbsp;--comparison-operator GreaterThanThreshold \
&nbsp;&nbsp;--threshold 20 \
&nbsp;&nbsp;--evaluation-periods 2 \
&nbsp;&nbsp;--treat-missing-data notBreaching \
&nbsp;&nbsp;--metrics '[
&nbsp;&nbsp; &nbsp;{
&nbsp;&nbsp; &nbsp; &nbsp;"Id": "low",
&nbsp;&nbsp; &nbsp; &nbsp;"MetricStat": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;"Metric": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"MetricName": "EmailAddressInsights.ConfidenceVerdict.LOW",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Namespace": "AWS/SES"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;"Period": 300,
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;"Stat": "Sum"
&nbsp;&nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp;"ReturnData": false
&nbsp;&nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp;{
&nbsp;&nbsp; &nbsp; &nbsp;"Id": "medium",
&nbsp;&nbsp; &nbsp; &nbsp;"MetricStat": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;"Metric": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"MetricName": "EmailAddressInsights.ConfidenceVerdict.MEDIUM",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Namespace": "AWS/SES"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;"Period": 300,
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;"Stat": "Sum"
&nbsp;&nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp;"ReturnData": false
&nbsp;&nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp;{
&nbsp;&nbsp; &nbsp; &nbsp;"Id": "high",
&nbsp;&nbsp; &nbsp; &nbsp;"MetricStat": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;"Metric": {
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"MetricName": "EmailAddressInsights.ConfidenceVerdict.HIGH",
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;"Namespace": "AWS/SES"
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;"Period": 300,
&nbsp;&nbsp; &nbsp; &nbsp; &nbsp;"Stat": "Sum"
&nbsp;&nbsp; &nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp; &nbsp;"ReturnData": false
&nbsp;&nbsp; &nbsp;},
&nbsp;&nbsp; &nbsp;{
&nbsp;&nbsp; &nbsp; &nbsp;"Id": "e1",
&nbsp;&nbsp; &nbsp; &nbsp;"Expression": "IF((low+medium+high)&gt;0, low/(low+medium+high)*100, 0)",
&nbsp;&nbsp; &nbsp; &nbsp;"Label": "LOW Rate Percentage",
&nbsp;&nbsp; &nbsp; &nbsp;"ReturnData": true
&nbsp;&nbsp; &nbsp;}
&nbsp;&nbsp;]'</code></pre> 
</div> 
<h3>How AnyCompany uses validation metrics</h3> 
<p>AnyCompany monitors their Email Validation dashboard in the Amazon SES console to track list quality trends. For example, if they notice an increase in disposable email failures, they can add additional client-side validation to their registration forms to discourage this behavior. When Auto&nbsp;Validation blocks a spike of invalid addresses from a specific partner marketing campaign, they avoid the problems associated with a spike in bounces while being better informed when investigating the list source and removing or cleaning it for future campaigns.</p> 
<h2>Validating email addresses at send time with Auto Validation</h2> 
<p>AnyCompany has been operating its online platform for many years without a way to validate email addresses. The company also makes frequent acquisitions and partnerships that regularly introduce new email addresses into their sending. This means that no matter how well the new registration for with the Email Validation API performs, they will always have some invalid email addresses in their outbound sends.</p> 
<p>This is&nbsp;one of the scenarios that can be addressed with no code or process changes by using Auto Validation. When enabled at the AWS account level, Auto Validation checks each address before sending, automatically suppressing the send of invalid addresses, adding those addresses to the account suppression list, and generating bounce notification events. These bounce events appear in <a href="https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html" rel="noopener noreferrer" target="_blank">Amazon SES event publishing</a> and can be monitored using <a href="https://aws.amazon.com/cloudwatch/" rel="noopener noreferrer" target="_blank">Amazon CloudWatch</a>, <a href="https://aws.amazon.com/sns/" rel="noopener noreferrer" target="_blank">Amazon Simple Notification Service (Amazon SNS)</a>, or <a href="https://aws.amazon.com/eventbridge/" rel="noopener noreferrer" target="_blank">Amazon EventBridge</a> or written to an <a href="https://aws.amazon.com/s3/" rel="noopener noreferrer" target="_blank">Amazon Simple Storage Service (Amazon S3)</a> bucket. Auto Validation bounce events appear as:</p> 
<ul> 
 <li><strong>Bounce type</strong>: <code>Permanent</code> for addresses that will never be deliverable</li> 
 <li><strong>Bounce subtype</strong>:&nbsp;<code>EmailValidationSuppressed</code> indicating Auto&nbsp;Validation blocked the send</li> 
</ul> 
<p>Because it’s implemented in Amazon SES events, AnyCompany can handle address validation failures the same way they currently handle actual bounces from mailbox providers, maintaining consistency in their email processing workflows.</p> 
<h2>Conclusion</h2> 
<p>Amazon SES Email Validation addresses critical needs for organizations sending email at scale: preventing invalid addresses at registration and automatically filtering risky recipients before sending. The feature’s two complementary approaches—the Email Validation API for real-time checks and Auto&nbsp;Validation for automatic send-time filtering—give you flexibility to implement validation where it makes the most sense for your workflows.</p> 
<p><strong>Use the Email Validation API to:</strong></p> 
<ul> 
 <li>Validate at point of collection (registration, imports)</li> 
 <li>Receive immediate user feedback</li> 
 <li>Build custom validation workflows</li> 
 <li>Validate before database entry</li> 
 <li>Validate up to 10 addresses</li> 
</ul> 
<p><strong>Use Auto&nbsp;Validation to:</strong></p> 
<ul> 
 <li>Automatically protect ongoing campaigns automatically</li> 
 <li>Avoid code changes to sending logic</li> 
 <li>Provide consistent quality across all sends</li> 
 <li>Set organization-wide quality standards</li> 
</ul> 
<p>By implementing both features of Amazon SES Email Validation, you can better protect your sender reputation by proactively preventing bounces, reducing the possibility of high bounce rates that can damage your deliverability.</p> 
<h3>Next steps</h3> 
<p>Start improving your email deliverability today:</p> 
<ol> 
 <li><strong>Enable Email Validation</strong> in your AWS account using the Amazon SES console or the AWS CLI</li> 
 <li><strong>Implement API validation</strong> at your registration points to improve data quality from the start</li> 
 <li><strong>Configure Auto&nbsp;Validation</strong> policies to protect your sender reputation across all campaigns</li> 
 <li><strong>Set up CloudWatch dashboards</strong> to track validation performance and identify list quality trends</li> 
 <li><strong>Review validation metrics weekly</strong> to refine your validation policies based on actual patterns</li> 
</ol> 
<p>For more information about Amazon SES Email Validation, see the <a href="https://docs.aws.amazon.com/ses/" rel="noopener noreferrer" target="_blank">Amazon SES Developer Guide</a>.</p> 
<hr /> 
<h3>About the authors</h3>
