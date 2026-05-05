---
title: "Establishing finops management: Integrating AWS Budgets with WhatsApp using AWS End User Messaging"
url: "https://aws.amazon.com/blogs/messaging-and-targeting/establishing-finops-management-integrating-aws-budgets-with-whatsapp-using-aws-end-user-messaging/"
date: "Thu, 15 Jan 2026 15:47:36 +0000"
author: "Ruchikka Chaudhary"
feed_url: "https://aws.amazon.com/blogs/messaging-and-targeting/feed/"
---
<p>Managing cloud costs effectively is a critical concern for organizations of all sizes. While <a href="https://aws.amazon.com/aws-cost-management/aws-budgets/" rel="noopener noreferrer" target="_blank">AWS Budgets</a> provides powerful tools to set spending thresholds and receive notifications, these alerts traditionally arrive through email or through AWS Management Console notifications. These traditional notification methods face several challenges when managing cloud costs:</p> 
<ul> 
 <li>Email notifications might not be seen immediately</li> 
 <li>Important budget alerts can get lost in crowded inboxes</li> 
 <li>Team members might not have immediate access to their email or the console</li> 
 <li>Global teams need accessible alerting mechanisms that work across time zones</li> 
</ul> 
<p>Today, we’re sharing a solution that brings <a href="https://aws.amazon.com/aws-cost-management/aws-budgets/" rel="noopener noreferrer" target="_blank">AWS Budgets</a> alerts directly to your WhatsApp using <a href="https://aws.amazon.com/end-user-messaging/" rel="noopener noreferrer" target="_blank">AWS End User Messaging</a>—enabling real-time cost awareness and faster response to budget thresholds wherever you are.</p> 
<h2>Overview of solution</h2> 
<p>Our solution integrates AWS Budgets with WhatsApp messaging using AWS End User Messaging, <a href="https://aws.amazon.com/pm/lambda/" rel="noopener noreferrer" target="_blank">AWS Lambda</a>, and <a href="https://aws.amazon.com/pm/sns/" rel="noopener noreferrer" target="_blank">Amazon Simple Notification Service (Amazon SNS)</a>. When a budget threshold is crossed, the alert is processed and delivered as a formatted WhatsApp message to designated recipients.</p> 
<p>The architecture, shown in the following figure, consists of four main AWS services to deliver budget alerts. AWS Budgets tracks expenses against your defined thresholds. When expenses exceed these thresholds, Amazon SNS receives an alert. An AWS Lambda function processes this alert and sends it through AWS End User Messaging to WhatsApp. Users then receive actionable budget notifications directly on their WhatsApp.</p> 
<p><img alt="" class="alignnone size-full wp-image-7313" height="484" src="https://d2908q01vomqb2.cloudfront.net/632667547e7cd3e0466547863e1207a8c0c0c549/2026/01/09/MESSAGING-1370-image-1.jpeg" width="1381" /></p> 
<p>Billing and Cost Management data, which AWS Budgets uses to monitor resources, is updated at least once per day. Keep in mind that budget information and associated alerts are updated and sent according to this data refresh cadence. In a budget period,</p> 
<p>notifications are triggered every time the notification state goes from <code>OK</code> to <code>Exceeded</code> (when the threshold is exceeded). If the budget stays in <code>Exceeded</code> state in the same budget period, AWS Budgets doesn’t send an additional alert.</p> 
<h2>Prerequisites</h2> 
<p>Implementation requires an AWS account with appropriate permissions for <a href="http://aws.amazon.com/cloudformation" rel="noopener noreferrer" target="_blank">AWS CloudFormation</a>, Lambda, Amazon SNS, and AWS Budgets. You must also have a WhatsApp Business Account integrated with <a href="https://docs.aws.amazon.com/social-messaging/latest/userguide/what-is-service.html" rel="noopener noreferrer" target="_blank">AWS End User Messaging Social</a> and the WhatsApp phone number ID from the AWS End User Messaging console. For instructions to locate this information, see <a href="https://docs.aws.amazon.com/social-messaging/latest/userguide/managing-phone-numbers-id.html" rel="noopener noreferrer" target="_blank">View a phone number’s ID in AWS End User Messaging Social</a>.</p> 
<p>For more information about how to set up WhatsApp using AWS End User Messaging Social, see <a href="https://aws.amazon.com/blogs/messaging-and-targeting/whatsapp-aws-end-user-messaging-social/" rel="noopener noreferrer" target="_blank">Automate workflows with WhatsApp using AWS End User Messaging Social</a>.</p> 
<p>Before you deploy this solution, create an approved utility template in your Meta account named <code>aws_budgets_notification_template</code>(as shown in the following screenshot). Alternatively, use your preferred template name and modify the Lambda function code accordingly.</p> 
<p><img alt="" class="alignnone size-full wp-image-7314" height="1312" src="https://d2908q01vomqb2.cloudfront.net/632667547e7cd3e0466547863e1207a8c0c0c549/2026/01/09/MESSAGING-1370-image-2.png" width="2416" /></p> 
<p><img alt="" class="alignnone size-full wp-image-7315" height="1194" src="https://d2908q01vomqb2.cloudfront.net/632667547e7cd3e0466547863e1207a8c0c0c549/2026/01/09/MESSAGING-1370-image-3.png" width="1734" /></p> 
<p>The preceding figure shows variable samples that can be used while creating a message template. You can also use the following <a href="https://aws.amazon.com/cli/" rel="noopener noreferrer" target="_blank">AWS Command Line Interface (AWS CLI)</a> command to create the messaging template-</p> 
<div class="hide-language"> 
 <pre><code class="lang-css">aws socialmessaging create-whatsapp-message-template \
  --region &lt;region&gt; \
  --id &lt;waba-id&gt; \
  --template-definition "$(echo '{
    "name": "aws_budgets_notification_template",
    "language": "en",
    "category": "UTILITY",
    "parameter_format": "named",
    "components": [
      {
        "type": "HEADER",
        "format": "TEXT",
        "text": "{{emoji}} Budget Alert",
        "example": {
          "header_text_named_params": [
            {"param_name": "emoji", "example": "?"}
          ]
        }
      },
      {
        "type": "BODY",
        "text": "Subject: {{subject}}\\nDetails: {{notification_title}}\\nAWS Account {{account_info}}\\n\\n{{notification_msg}}\\n\\nBudget Name: {{budget_name}}\\nBudget Type: {{budget_type}}\\nBudgeted Amount: {{budgeted_amount}}\\nAlert Type: {{alert_type}}\\nAlert Threshold: {{alert_threshold}}\\nFORECASTED Amount: {{forecasted_amount}}\\n\\nAWS Console: {{console_link}}\\n\\nTime: {{time}}\\n\\nTip: Check your AWS Billing Dashboard for detailed cost breakdown",
        "example": {
          "body_text_named_params": [
            {"param_name": "subject", "example": "AWS Budgets: Budget-Cloudwatch-budget-dev has exceeded your alert threshold"},
            {"param_name": "notification_title", "example": "AWS Budget Notification Oct 19, 2025"},
            {"param_name": "account_info", "example": "AWS Account 12345"},
            {"param_name": "notification_msg", "example": "You requested that we alert you when the FORECASTED Cost associated with your Budget-Cloudwatch-dev Budget is greater"},
            {"param_name": "budget_name", "example": "Budget-Cloudwatch-budget-dev"},
            {"param_name": "budget_type", "example": "Cost"},
            {"param_name": "budgeted_amount", "example": "$1"},
            {"param_name": "alert_type", "example": "Cost"},
            {"param_name": "alert_threshold", "example": "&gt; 1"},
            {"param_name": "forecasted_amount", "example": "$2"},
            {"param_name": "console_link", "example": "https://console.aws.amazon.com/billing/home#/budgets"},
            {"param_name": "time", "example": "2025-10-19 12:38:52 UTC"}
          ]
        }
      }
    ]
  }' | base64)"</code></pre> 
</div> 
<p>You can confirm the template approval status and type in the Meta portal.</p> 
<h2>Solution walkthrough</h2> 
<p>The core component consists of a Python-based Lambda function that processes Budget alerts and formats them for WhatsApp delivery. The function receives SNS events containing budget alerts data, extracts relevant information, formats contextual messages, and delivers notifications through AWS End User Messaging Social.The following function shows an example to parse the SNS message content:</p> 
<div class="hide-language"> 
 <pre><code class="lang-python"> def process_notification(record):
"""Process SNS notification and send WhatsApp message"""
sns_message = record['Sns']
subject = sns_message.get('Subject', 'AWS Notification')
message_body = sns_message.get('Message', '')

logger.info(f"Processing notification - Subject: {subject}")
process_budget_notification(subject, message_body)</code></pre> 
</div> 
<p>The following example demonstrates how to format alert information—including subject, details, and timestamp—and deliver the message to WhatsApp.</p> 
<div class="hide-language"> 
 <pre><code class="lang-css">#Create template message with parsed values
template_name = "aws_budgets_notification_template"
template_message = {
    "name": template_name,
    "language": {
        "code": "en"
    },
    "components": [
        {
            "type": "header",
            "parameters": [{
                "type": "text",
                "parameter_name": "emoji",
                "text": "?"
            }]
        },
        {
            "type": "body",
            "parameters": [
                {
                    "type": "text",
                    "parameter_name": "subject",
                    "text": subject
                },
                {
                    "type": "text",
                    "parameter_name": "notification_title",
                    "text": notification_title
                },
                {
                    "type": "text",
                    "parameter_name": "account_info",
                    "text": f"AWS Account {account_number}"
                },
                {
                    "type": "text",
                    "parameter_name": "notification_msg",
                    "text": notification_msg + "."
                },
                {
                    "type": "text",
                    "parameter_name": "budget_name",
                    "text": budget_details.get('Budget Name', '')
                },
                {
                    "type": "text",
                    "parameter_name": "budget_type",
                    "text": budget_details.get('Budget Type', '')
                },
                {
                    "type": "text",
                    "parameter_name": "budgeted_amount",
                    "text": budget_details.get('Budgeted Amount', '')
                },
                {
                    "type": "text",
                    "parameter_name": "alert_type",
                    "text": budget_details.get('Alert Type', '')
                },
                {
                    "type": "text",
                    "parameter_name": "alert_threshold",
                    "text": budget_details.get('Alert Threshold', '')
                },
                {
                    "type": "text",
                    "parameter_name": "forecasted_amount",
                    "text": budget_details.get('FORECASTED Amount', '')
                },
                {
                    "type": "text",
                    "parameter_name": "console_link",
                    "text": "https://console.aws.amazon.com/billing/home#/budgets"
                },
                {
                    "type": "text",
                    "parameter_name": "time",
                    "text": datetime.now().strftime('%Y-%m-%d %H:%M:%S UTC')
                }
            ]
        }
    ]
}
send_whatsapp_message(template_message)</code></pre> 
</div> 
<p>The <code>send_whatsapp_message</code> function uses AWS End User Messaging Social to deliver formatted messages through the <code>socialmessaging</code> client, as shown in the following example:</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">def send_whatsapp_message(message):
  client = boto3.client('socialmessaging')
  # Get environment variables
  phone_number_id = os.environ.get('WHATSAPP_PHONE_NUMBER_ID')
  recipient = os.environ.get('ALERT_RECIPIENT')
                    
# Prepare message object
  message_object = {
"messaging_product": "whatsapp",
"recipient_type": "individual",
"to": recipient,
"type": "template",
"template": message
}
  # Send message
response = client.send_whatsapp_message(
originationPhoneNumberId=phone_number_id,
metaApiVersion="v20.0",
message=bytes(json.dumps(message_object), "utf-8")  
)</code></pre> 
</div> 
<h2>Deploying the solution</h2> 
<p>The solution uses <a href="https://aws.amazon.com/cloudformation" rel="noopener noreferrer" target="_blank">AWS CloudFormation</a> for infrastructure as code (IaC) deployment. The main template creates an SNS topic for alert notifications, a Lambda function for message processing, and required <a href="https://aws.amazon.com/iam/" rel="noopener noreferrer" target="_blank">AWS Identity and Access Management (IAM)</a> roles with least-privilege permissions.</p> 
<p>The <strong><a href="https://aws-blogs-artifacts-public.s3.us-east-1.amazonaws.com/artifacts/MESS-1370/eum-budget-alerts.yaml" rel="noopener" target="_blank">CloudFormation template</a></strong> requires a recipient number with an active WhatsApp account to receive alert notifications as messages. The template also requires the WhatsApp phone number ID retrieved from the AWS End User Messaging Social console, as noted in the prerequisites. The template must be deployed in the same AWS Region as AWS End User Messaging Social. See the following code:</p> 
<div class="hide-language"> 
 <pre><code class="lang-powershell">aws&nbsp;cloudformation&nbsp;deploy&nbsp;\
&nbsp;&nbsp;--template-file &lt;&gt;&nbsp;\
&nbsp;&nbsp;--stack-name&nbsp;budget-eum-whatsapp-alerts&nbsp;\
&nbsp;&nbsp;--parameter-overrides \
&nbsp;&nbsp; &nbsp;ActualSpendThreshold= \
&nbsp;&nbsp; &nbsp;AlertRecipient=&lt;+1234567890&gt; \
    BudgetAmount=&lt;Budget Amount e.g. 3000&gt; \
&nbsp;&nbsp; &nbsp;Environment=&nbsp;\
    ForecastedSpendThreshold= \
    WhatsAppPhoneNumberId= \
&nbsp;&nbsp;--capabilities CAPABILITY_NAMED_IAM \
&nbsp;&nbsp;--region &lt;EUM-region&gt;</code></pre> 
</div> 
<h2>Testing the solution</h2> 
<p>The solution can be tested and validated using the SNS topic created by the CloudFormation stack. Use the following AWS CLI command to publish a test message to the SNS topic:</p> 
<div class="hide-language"> 
 <pre><code class="lang-powershell">aws sns publish \
  --topic-arn arn:aws:sns:&lt;region&gt;:&lt;account&gt;:&lt;topic-name&gt; \
  --subject "[Test] AWS Budget Alert: Budget-Alert-EUM has exceeded 80% of your budgeted amount" \
  --message "AWS Budget Notification - Your budget has exceeded the alert threshold
AWS Account 123456789012
Budget Name: Budget-Alert-EUM
Budget Type: Cost
Budgeted Amount: $100.00
Alert Type: ACTUAL
Alert Threshold: 80%
FORECASTED Amount: $85.00
You have exceeded 80% of your budget for this period" \
  --region &lt;region&gt;</code></pre> 
</div> 
<p>The SNS message triggers the Lambda function, which sends the alert to your configured WhatsApp recipient, as shown in the following screenshot.</p> 
<p><img alt="" class="alignnone size-full wp-image-7316" height="1076" src="https://d2908q01vomqb2.cloudfront.net/632667547e7cd3e0466547863e1207a8c0c0c549/2026/01/09/MESSAGING-1370-image-4.png" width="1326" /></p> 
<h2>Clean up</h2> 
<p>Use the following steps to clean up your resources when you no longer need this solution:</p> 
<ol> 
 <li>Delete the CloudFormation stack deployed in this solution: <code>budget-eum-whatsapp-alerts</code><em>.&nbsp;</em></li> 
 <li>Delete the template in your Meta account.</li> 
</ol> 
<h2>Conclusion</h2> 
<p>Integrating AWS Budgets alerts with WhatsApp notifications represents a significant step forward in modern cost monitoring. By using AWS End User Messaging Social, you can send teams critical alerts through their preferred communication channel while maintaining the reliability and scalability of AWS services. The solution’s modular architecture, basic yet effective security model, and cost-effective design make it suitable for organizations of various sizes.</p> 
<hr /> 
<h3>About the authors</h3>
