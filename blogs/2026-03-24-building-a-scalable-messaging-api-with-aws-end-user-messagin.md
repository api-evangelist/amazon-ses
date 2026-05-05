---
title: "Building a Scalable Messaging API with AWS End User Messaging and SES"
url: "https://aws.amazon.com/blogs/messaging-and-targeting/building-a-scalable-messaging-api-with-aws-end-user-messaging-and-ses/"
date: "Tue, 24 Mar 2026 16:26:06 +0000"
author: "Bruno Giorgini"
feed_url: "https://aws.amazon.com/blogs/messaging-and-targeting/feed/"
---
<p>Modern applications often need to send notifications across multiple channels either through email and/or SMS. However, building a reliable messaging system that manages templates, handles failures gracefully, scales, and maintains security can be challenging. Following this guide, you’ll learn how to build a template manager and messaging API using API Gateway with JWT authentication for secure access, Amazon SQS for reliable message queuing, AWS Lambda for serverless processing, AWS End User Messaging for SMS, and Amazon Simple Email Service (SES) for email.</p> 
<h2>Architecture overview</h2> 
<p>You deploy a decoupled architecture that separates message ingestion from processing, providing resilience and scalability.</p> 
<div class="wp-caption alignnone" id="attachment_7413" style="width: 1141px;">
 <img alt="Fig. 1 Message Template Manager Architecture" class="wp-image-7413 size-full" height="791" src="https://d2908q01vomqb2.cloudfront.net/632667547e7cd3e0466547863e1207a8c0c0c549/2026/03/23/multichannel-messaging-api-3.png" width="1131" />
 <p class="wp-caption-text" id="caption-attachment-7413">Fig. 1 Message Template Manager Architecture</p>
</div> 
<h3>Architecture flow</h3> 
<ol> 
 <li>Client Application sends authenticated requests with JWT tokens</li> 
 <li>API Gateway validates requests using a Lambda Authorizer</li> 
 <li>Lambda Authorizer retrieves the JWT secret from AWS Secrets Manager and validates the token</li> 
 <li>API Gateway sends validated messages to the SQS Queue</li> 
 <li>Lambda Processor polls messages from SQS in batches</li> 
 <li>Lambda Processor retrieves message templates from DynamoDB (if needed)</li> 
 <li>Lambda Processor sends emails through Amazon SES and SMS through AWS End User Messaging</li> 
 <li>Failed messages (after 3 retries) move to the Dead Letter Queue</li> 
 <li>CloudWatch Alarm triggers when messages arrive in the DLQ</li> 
</ol> 
<p>If a message fails processing after three attempts, it moves to a Dead Letter Queue (DLQ) where it’s preserved for 14 days, and a CloudWatch alarm notifies you of the failure.</p> 
<h2>Key features</h2> 
<p>With this architecture, you get several important capabilities:</p> 
<ul> 
 <li>JWT Authentication: Secure API access using JSON Web Tokens stored in AWS Secrets Manager</li> 
 <li>Automatic Retries: Failed messages retry up to three times before moving to the DLQ</li> 
 <li>Partial Batch Failures: Only failed messages retry, not the entire batch</li> 
 <li>Template Management: Store reusable message templates in Amazon DynamoDB</li> 
 <li>Multi-Channel Support: Send email through Amazon SES and SMS through AWS End User Messaging</li> 
 <li>Configuration Set Support: Track delivery metrics and route events with per-message or deployment-level configuration sets</li> 
 <li>Monitoring: CloudWatch alarms alert you when messages fail</li> 
</ul> 
<h2>Implementation details</h2> 
<h3>1. API Gateway with JWT authorization</h3> 
<p>The API Gateway uses a Lambda authorizer to validate JWT tokens before allowing requests through:</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">def lambda_handler(event, context):
    token = event.get('authorizationToken', '').replace('Bearer ', '')
    try:
        # Retrieve secret from AWS Secrets Manager
        jwt_secret = get_jwt_secret()
        
        # Validate JWT token
        payload = jwt.decode(
            token,
            jwt_secret,
            algorithms=['HS256'],
            issuer='messaging-api'
        )
        
        # Generate IAM policy to allow request
        return generate_policy(payload.get('sub'), 'Allow', event['methodArn'])
    except jwt.ExpiredSignatureError:
        raise Exception('Unauthorized: Token expired')
    except jwt.InvalidTokenError:
        raise Exception('Unauthorized: Invalid token')</code></pre> 
</div> 
<p>The JWT secret is stored securely in AWS Secrets Manager and cached in the Lambda execution environment for performance.</p> 
<h3>2. SQS queue configuration</h3> 
<p>The SAM template defines two queues with appropriate settings:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">MessagesQueue:
  Type: AWS::SQS::Queue
  Properties:
    QueueName: MessagesQueue
    VisibilityTimeout: 300  # 5 minutes
    MessageRetentionPeriod: 345600  # 4 days
    RedrivePolicy:
      deadLetterTargetArn: !GetAtt MessagesDeadLetterQueue.Arn
      maxReceiveCount: 3

MessagesDeadLetterQueue:
  Type: AWS::SQS::Queue
  Properties:
    QueueName: MessagesDeadLetterQueue
    MessageRetentionPeriod: 1209600  # 14 days</code></pre> 
</div> 
<p>The visibility timeout of 5 minutes prevents duplicate processing while giving the Lambda function enough time to complete. Messages that fail three times automatically move to the DLQ.</p> 
<h3>3. Lambda message processor</h3> 
<p>The Lambda function processes messages from SQS and sends them through the appropriate channel:</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">def lambda_handler(event, context):
    failed_messages = []
    
    for record in event['Records']:
        message_id = record['messageId']
        try:
            message = json.loads(record['body'])
            
            # Process email if configured
            if 'EmailMessage' in message:
                send_emails(message)
            
            # Process SMS if configured
            if 'SMSMessage' in message:
                send_sms_messages(message)
                
        except Exception as e:
            print(f"Error processing message {message_id}: {str(e)}")
            failed_messages.append({"itemIdentifier": message_id})
    
    # Return failed messages for automatic retry
    return {"batchItemFailures": failed_messages}</code></pre> 
</div> 
<p><strong>Configuration Sets for tracking and analytics:</strong></p> 
<p>Configuration Sets enable you to track delivery metrics, monitor costs, and route events to analytics pipelines. You can set defaults at deployment time and override them per-message:</p> 
<ul> 
 <li>Deployment-level defaults: Set <code>SMSConfigurationSet</code> parameter during deployment to apply to all messages</li> 
 <li>Per-message override: Include <code>ConfigurationSetName</code> in the <code>SMSMessage</code> payload to use different tracking for specific messages</li> 
</ul> 
<p>This flexibility lets you separate analytics by message type, campaign, or priority without requiring redeployment.</p> 
<h3>4. Template management with DynamoDB</h3> 
<p>Message templates are stored in DynamoDB for reusability:</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">def get_email_template_from_dynamodb(template_name, substitutions):
    response = templates_table.get_item(Key={'TemplateName': template_name})
    
    if 'Item' not in response:
        return build_default_email(), "Account Alert"
    
    template_body = response['Item']['MessageBody']
    subject = response['Item'].get('Subject', 'Notification')
    
    # Replace {variable} placeholders with actual values
    rendered_body = replace_variables(template_body, substitutions)
    rendered_subject = replace_variables(subject, substitutions)
    
    return rendered_body, rendered_subject</code></pre> 
</div> 
<p>You can update message content without redeploying code.</p> 
<p><strong>Template size considerations:</strong></p> 
<p>DynamoDB has a 400 KB limit per item, which includes all attribute names and values. For message templates, this means:</p> 
<ul> 
 <li>Typical email templates (5-20 KB) fit comfortably</li> 
 <li>SMS templates (&lt; 1 KB) have no practical constraints</li> 
</ul> 
<p>If you need to store templates larger than 400 KB, consider storing them in Amazon S3 and referencing the S3 object key in DynamoDB. This hybrid approach provides unlimited template size while maintaining fast lookups.</p> 
<h2>Prerequisites</h2> 
<p>Before deploying this solution, ensure you have the following:</p> 
<ul> 
 <li>AWS End User Messaging SMS configured with a phone pool or origination identity for SMS sending</li> 
 <li>Amazon SES configured with verified email identities (sender and recipient for sandbox mode)</li> 
 <li>IAM permissions to create Lambda functions, API Gateway, SQS queues, DynamoDB tables, and Secrets Manager secrets</li> 
 <li>Python 3.9 or later installed locally</li> 
 <li>AWS SAM CLI installed (version 1.0 or later)</li> 
 <li>AWS CLI installed and configured with your credentials</li> 
 <li>An active AWS account with appropriate permissions</li> 
</ul> 
<h2>Deployment</h2> 
<p>You use AWS SAM for infrastructure as code. Deploy with these commands:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">sam build
sam deploy --guided</code></pre> 
</div> 
<p>During deployment, you’ll set a JWT secret that’s stored in AWS Secrets Manager. Use a strong, random secret for production:</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">python -c "import secrets; print(secrets.token_urlsafe(32))"</code></pre> 
</div> 
<h2>Usage example</h2> 
<p>Once deployed, send messages by making authenticated API requests:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">curl -X POST "https://your-api-endpoint/dev/" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -d '{
    "TraceId": "12345",
    "EmailMessage": {
      "FromAddress": "alerts@example.com",
      "Subject": "Low Balance Alert",
      "ConfigurationSetName": "email-analytics",
      "Substitutions": {
        "productName": "CHEQUING",
        "membershipNumber": "****5493",
        "accountBalance": "100.00"
      }
    },
    "SMSMessage": {
      "MessageType": "TRANSACTIONAL",
      "OriginationNumber": "your-pool-id",
      "TemplateName": "alert-template",
      "ConfigurationSetName": "sms-analytics"
    },
    "Addresses": {
      "user@example.com": {
        "ChannelType": "EMAIL"
      },
      "+16048621234": {
        "ChannelType": "SMS",
        "Substitutions": {
          "productName": "CHEQUING",
          "membershipNumber": "****7303",
          "accountBalance": "100.00"
        }
      }
    }
  }'</code></pre> 
</div> 
<p><strong>Using Configuration Sets:</strong></p> 
<p>The example above shows optional <code>ConfigurationSetName</code> parameters for both email and SMS. These enable:</p> 
<ul> 
 <li>Delivery tracking: Monitor delivery rates, failures, and bounce metrics</li> 
 <li>Cost monitoring: Track spending per campaign or message type</li> 
 <li>Event routing: Send delivery events to CloudWatch, Kinesis, or SNS for analytics</li> 
 <li>Segmented metrics: Separate analytics by use case, priority, or customer segment</li> 
</ul> 
<p>If you don’t specify a <code>ConfigurationSetName</code> in the request, the system uses the deployment-level default (if configured).</p> 
<h2>Monitoring and operations</h2> 
<h3>CloudWatch alarms</h3> 
<p>The solution includes a CloudWatch alarm that triggers when messages arrive in the DLQ:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">DLQAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: !Sub ${AWS::StackName}-DLQ-Messages
    MetricName: ApproximateNumberOfMessagesVisible
    Namespace: AWS/SQS
    Statistic: Sum
    Period: 300
    EvaluationPeriods: 1
    Threshold: 1
    ComparisonOperator: GreaterThanOrEqualToThreshold</code></pre> 
</div> 
<h3>Handling failed messages</h3> 
<p>When messages fail, you can inspect them in the DLQ and redrive them back to the main queue after fixing the issue:</p> 
<div class="hide-language"> 
 <pre><code class="lang-python"># Check DLQ depth
aws sqs get-queue-attributes \
  --queue-url YOUR_DLQ_URL \
  --attribute-names ApproximateNumberOfMessages

# Redrive messages from DLQ to main queue
aws sqs start-message-move-task \
  --source-arn YOUR_DLQ_ARN \
  --destination-arn YOUR_MAIN_QUEUE_ARN</code></pre> 
</div> 
<h3>Viewing logs</h3> 
<p>Lambda automatically logs to CloudWatch Logs:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">aws logs tail /aws/lambda/MessageProcessor --follow</code></pre> 
</div> 
<h2>Cost considerations</h2> 
<p>For 1 million messages per month estimated costs are:</p> 
<table border="1px" cellpadding="10px" class="styled-table"> 
 <tbody> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Service</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Usage</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Cost</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">API Gateway</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">1M requests</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">$3.50</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Amazon SQS</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">1M messages</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">$0.40</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">AWS Lambda</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">1M invocations (128MB, 1s avg)</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">$2.50</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Amazon SES</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">1M emails</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">$100.00</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">AWS End User Messaging SMS</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">1M SMS</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">$ Varies based on destination</td> 
  </tr> 
 </tbody> 
</table> 
<p><strong>Note:</strong> The serverless architecture means you only pay for what you use, with no minimum fees or upfront costs.</p> 
<h2>Security best practices</h2> 
<p>This solution follows several security best practices:</p> 
<ol> 
 <li><strong>JWT Authentication</strong>: All API requests require valid JWT tokens</li> 
 <li><strong>Secrets Manager</strong>: JWT secrets are stored encrypted in AWS Secrets Manager</li> 
 <li><strong>IAM Least Privilege</strong>: Each Lambda function has only the permissions it needs</li> 
 <li><strong>HTTPS Only</strong>: API Gateway enforces HTTPS for all requests</li> 
 <li><strong>Token Caching</strong>: Authorization decisions are cached for 5 minutes to reduce latency</li> 
</ol> 
<h2>Clean up</h2> 
<p>To avoid incurring ongoing charges, delete the resources created by this solution when you no longer need them:</p> 
<ol> 
 <li>If you configured AWS End User Messaging phone pools or origination identities for this solution, remove them from the End User Messaging console</li> 
 <li>If you created Amazon SES email identities specifically for this solution, remove them from the SES console</li> 
 <li>Verify that all resources (Lambda functions, API Gateway, SQS queues, DynamoDB table, Secrets Manager secret) have been removed in the AWS Management Console</li> 
 <li>Delete the CloudFormation stack by running: <code>sam delete --stack-name &lt;your-stack-name&gt;</code></li> 
</ol> 
<h2>Conclusion</h2> 
<p>With this architecture, you can build a production-ready messaging API using AWS serverless services. The decoupled design gives you resilience through automatic retries and dead letter queues, while the serverless approach eliminates infrastructure management and scales automatically.</p> 
<p>The complete solution is deployable through AWS SAM and includes:</p> 
<ul> 
 <li>JWT authentication with AWS Secrets Manager</li> 
 <li>Multi-channel messaging (email and SMS)</li> 
 <li>Template management with DynamoDB</li> 
 <li>Configuration Set support for tracking and analytics</li> 
 <li>Comprehensive monitoring and alerting</li> 
 <li>Automatic retry and failure handling</li> 
</ul> 
<p>You can extend this architecture by adding more channels (push notifications, webhooks), implementing message scheduling, or integrating with Amazon EventBridge for event-driven workflows.</p> 
<h2>Additional resources</h2> 
<ul> 
 <li><a href="https://docs.aws.amazon.com/sqs/" rel="noopener" target="_blank">Amazon SQS Developer Guide</a></li> 
 <li><a href="https://docs.aws.amazon.com/lambda/">AWS Lambda Developer Guide</a></li> 
 <li><a href="https://docs.aws.amazon.com/ses/">Amazon SES Developer Guide</a></li> 
 <li><a href="https://docs.aws.amazon.com/sms-voice/">AWS End User Messaging SMS Documentation</a></li> 
 <li><a href="https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-use-lambda-authorizer.html">API Gateway Lambda Authorizers</a></li> 
 <li>AWS SAM Documentation</li> 
</ul> 
<p>The complete source code for this solution is available in the accompanying <a href="https://github.com/aws-samples/sample-multichannel-messaging-api-with-template" rel="noopener" target="_blank">GitHub repository</a>, including SAM templates, Lambda functions, and deployment scripts.</p> 
<hr style="width: 80%;" /> 
<h2>About the authors</h2>
