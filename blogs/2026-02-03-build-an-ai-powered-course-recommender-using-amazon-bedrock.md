---
title: "Build an AI-powered course recommender using Amazon Bedrock and AWS End User Messaging"
url: "https://aws.amazon.com/blogs/messaging-and-targeting/build-an-ai-powered-course-recommender-using-amazon-bedrock-and-aws-end-user-messaging/"
date: "Tue, 03 Feb 2026 21:19:02 +0000"
author: "Ruchikka Chaudhary"
feed_url: "https://aws.amazon.com/blogs/messaging-and-targeting/feed/"
---
<p>Educational technology (EdTech) providers face the challenge of maintaining seamless, personalized communication and presenting the right recommendations to their diverse stakeholders. This post explores how combining <a href="https://aws.amazon.com/" rel="noopener noreferrer" target="_blank">Amazon Web Services</a> (AWS) <a href="https://aws.amazon.com/end-user-messaging/" rel="noopener noreferrer" target="_blank">End User Messaging</a> and WhatsApp Business API with the advanced AI capabilities of <a href="https://aws.amazon.com/bedrock/" rel="noopener noreferrer" target="_blank">Amazon Bedrock</a> can transform educational engagement.</p> 
<p>In this post, we explore use cases that are reshaping the EdTech industry. We discover how application automation can streamline admissions and enrollment processes, making them more efficient and user-friendly. We demonstrate how instant student engagement can be achieved through AI-powered, personalized interactions that keep learners motivated and connected. We showcase how real-time course feedback mechanisms can help educators adapt and improve their teaching methods. We also examine how student support can be automated using intelligent assistants that provide continuous, all-day assistance while maintaining a personal touch.</p> 
<p>We show you how to build an AI-powered course recommendation system. We explain how to set up WhatsApp Business API integration with Amazon Bedrock, implement smart search capabilities for course matching, and create a scalable serverless architecture. You’ll learn how to build meaningful analytics dashboards to track engagement and learn best practices for handling errors and maintaining system reliability. Whether you’re an EdTech professional or a cloud architect, this guide gives you practical insights into combining conversational AI with educational services.</p> 
<h2>Use cases</h2> 
<ul> 
 <li>An AI-powered personalized learning pathway generator that automatically recommends customized content based on individual student performance metrics and learning requirements</li> 
 <li>Course improvement suggestions and real-time course feedback</li> 
 <li>A smart communication orchestrator that delivers role-specific, automated notifications and updates across multiple channels to enhance student and parent engagement</li> 
 <li>An early warning system using predictive analytics to identify at-risk students through real-time monitoring of engagement metrics and performance indicators</li> 
 <li>Student support automation with always available AI assistant support, FAQ handling, escalation management, and multilingual support</li> 
</ul> 
<h2>Prerequisites</h2> 
<ul> 
 <li>An AWS account</li> 
 <li>AWS End User Messaging set up with WhatsApp channel enabled</li> 
 <li>A pre-existing WhatsApp Business account</li> 
 <li>Amazon Bedrock setup must be completed with preferred model</li> 
 <li><a href="https://aws.amazon.com/quick/quicksight/" rel="noopener noreferrer" target="_blank">Amazon Quick Sight</a> for the AWS Region must be enabled</li> 
</ul> 
<h2>Solution overview</h2> 
<p>With this solution, users can discover and order educational courses through WhatsApp conversations. Instead of navigating complex websites, the user can send a WhatsApp message saying, “I want to learn Python programming.” They’ll receive personalized course recommendations instantly. The architecture processes WhatsApp messages through AWS End User Messaging, uses Amazon Bedrock for AI-powered conversations, performs semantic search with <a href="https://aws.amazon.com/opensearch-service/features/serverless/" rel="noopener noreferrer" target="_blank">Amazon OpenSearch Serverless</a>, and captures analytics for business insights. (For step-by-step implementation and rollback guidelines, see the <a href="https://github.com/aws-samples/sample-course-recommendation-system" rel="noopener noreferrer" target="_blank">sample course recommendation system</a>.) The following architectural diagram illustrates a modern AI-powered course recommendation system that uses multiple AWS services.</p> 
<p><img alt="" class="alignnone wp-image-7351 size-large" height="576" src="https://d2908q01vomqb2.cloudfront.net/632667547e7cd3e0466547863e1207a8c0c0c549/2026/01/28/messaging-1394-1-1024x576.jpeg" width="1024" /></p> 
<p>Figure 1: AI-powered course recommendation system</p> 
<h3>Message processing</h3> 
<p>When users send WhatsApp messages, AWS End User Messaging captures them and publishes events to an <a href="https://aws.amazon.com/sns/" rel="noopener noreferrer" target="_blank">Amazon Simple Notification Service (Amazon SNS)</a> topic. This creates a decoupled architecture where multiple services can process the same message events independently. <a href="https://aws.amazon.com/lambda/" rel="noopener noreferrer" target="_blank">AWS Lambda</a> functions subscribe to these events, facilitating reliable message processing during high-traffic periods. The decoupled design provides several advantages:</p> 
<ul> 
 <li>If one component fails, others continue operating</li> 
 <li>You can add new message processors without affecting existing ones</li> 
 <li>The system automatically scales based on message volume without manual intervention</li> 
</ul> 
<h3>AI conversation engine</h3> 
<p>Amazon Bedrock with <a href="https://askaichat.app/onboarding/multi-model-tools/claude?utm_source=google&amp;utm_medium=web&amp;utm_campaign=askai_website_go_search_us_purchase_claude_150925&amp;utm_content=askai_website_go_search_us_purchase_claude_150925&amp;utm_term=anthropics+claude&amp;matchtype=b&amp;device=c&amp;GeoLoc=9007903&amp;placement=&amp;network=g&amp;campaign_id=23024247805&amp;adset_id=184701123119&amp;ad_id=780380913348&amp;extension_id=&amp;gad_source=1&amp;gad_campaignid=23024247805&amp;gbraid=0AAAAA9YrWBArKyPgrM1szWz0NJr_1qJGE&amp;gclid=Cj0KCQiAsNPKBhCqARIsACm01fSvYxEBmicAKwCo3iVlFtjeT8aPhn_8uieOsb-yJ4ugZ1IwWhI0pWsaAt7JEALw_wcB" rel="noopener noreferrer" target="_blank">Claude 3 Haiku</a> powers natural language understanding. It is configured specifically for WhatsApp with instructions for short paragraphs, relevant emoji, and mobile-optimized responses.</p> 
<h3>AI agents</h3> 
<p>The agent maintains conversation context and handles structured actions such as course search, detail retrieval, and booking through defined functions. The following workflow is the agent action flow and sample code:</p> 
<h4>Agent flow</h4> 
<ol> 
 <li>Greets user → Understands intent → Searches courses → Provides details → Facilitates booking</li> 
 <li>Maintains context throughout the conversation</li> 
 <li>Can switch between actions based on user responses</li> 
 <li>Handles complex queries by combining multiple actions</li> 
</ol> 
<h4>Sample code</h4> 
<p>The following is sample code to create a Bedrock agent using AWS CDK:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">    agent = bedrock.CfnAgent(foundation_model="anthropic.claude-3-haiku-20240307-v1:0",
     instruction="""
     Format for WhatsApp: short paragraphs,
        focus on technical courses only
       """,
      action_groups=[# Functions for search, details, booking]
)</code></pre> 
</div> 
<h3>Semantic search</h3> 
<p>Traditional keyword search can miss the user’s intent. The application uses <a href="https://aws.amazon.com/blogs/machine-learning/getting-started-with-amazon-titan-text-embeddings/" rel="noopener noreferrer" target="_blank">Amazon Titan Embeddings</a> in Amazon Bedrock to convert courses and queries into vectors, enabling semantic understanding. When users ask for “cloud computing courses,” the system can understand related terms such as “AWS” and “serverless” without exact matches. Amazon OpenSearch Serverless handles vector similarity matching combined with traditional filters for course price, level, and duration.</p> 
<h3>Analytics pipeline</h3> 
<p>Every WhatsApp message interaction generates business intelligence. Messages are stored in <a href="https://aws.amazon.com/s3/" rel="noopener noreferrer" target="_blank">Amazon Simple Storage Service (Amazon S3)</a> with date partitioning, catalogued through <a href="https://aws.amazon.com/glue/" rel="noopener noreferrer" target="_blank">AWS Glue</a>, and made queryable using <a href="https://aws.amazon.com/athena/" rel="noopener noreferrer" target="_blank">Amazon Athena</a>. Teams can analyze user behavior, popular topics, and conversion rates through Quick Sight dashboards. The following dashboard shows example widgets displaying pie-chart breakdown of message delivery status and count of messages per day.</p> 
<p><img alt="" class="alignnone wp-image-7352 size-large" height="367" src="https://d2908q01vomqb2.cloudfront.net/632667547e7cd3e0466547863e1207a8c0c0c549/2026/01/28/messaging-1394-2-1024x367.png" width="1024" /></p> 
<p>Figure 2: Amazon Quick Sight dashboard</p> 
<p>As shown in the following dashboard, Amazon Q in QuickSight enables you to explore and analyze your data using conversational AI capabilities.</p> 
<p><img alt="" class="alignnone size-large wp-image-7353" height="389" src="https://d2908q01vomqb2.cloudfront.net/632667547e7cd3e0466547863e1207a8c0c0c549/2026/01/28/messaging-1394-3-1024x389.png" width="1024" /></p> 
<p>Figure 3: Amazon Quick Sight dashboard showing chat window</p> 
<h3>Error handling and resilience</h3> 
<p>Such highly scalable and distributed solutions require robust error handling. The application has exponential backoff and retries for API calls, meaning the system can gracefully handle rate limits and temporary service unavailability.</p> 
<p>The following is sample code for error handling and resilience:</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">python
def retry_with_backoff(func, max_retries=5):
retries = 0
backoff = 1
while retries &lt; max_retries:
try:
return func()
except ThrottlingException:
sleep_time = backoff + random.uniform(0, 1)
time.sleep(sleep_time)
backoff = min(backoff * 2, 32)
retries += 1
raise Exception("Max retries exceeded")</code></pre> 
</div> 
<h2>Business impact</h2> 
<p>With the global EdTech market expected to reach $165 billion by 2026, educators and institutions are seeking solutions to prevent student dropouts, improve learning outcomes, and maintain their competitive advantage. Poor personalization can lead to decreased student engagement, lower course completion rates, and ultimately revenue loss.</p> 
<p>Implementing AI-driven personalization and communication systems means institutions can significantly improve student retention rates, boost learning outcomes, and create a more engaging educational experience, which directly impacts their bottom line and reputation in an increasingly competitive educational landscape. This solution could transform educational delivery through intelligent personalization and operational excellence. A serverless architecture can help educational institutions focus on content quality rather than infrastructure management while potentially maintaining rapid response times for course searches. The system’s analytics capabilities could offer insights into student behavior and course preferences, helping shape future curriculum development.</p> 
<p>With mobile optimization, institutions can better serve the growing population of digital-first learners. The combination of automated scaling and pay-per-use pricing could create opportunities for cost optimization, and real-time dashboards can be used to facilitate data-informed decision-making. Such improvements in user experience and operational efficiency could lead to enhanced student engagement and institutional growth in the evolving education environment.</p> 
<h2>Sample conversation</h2> 
<p>The following video shows how a user can interact with the generative AI-powered course recommendation system and receive course recommendations.</p> 
<div> 
 <div class="wp-video" style="width: 640px;">
  <video class="wp-video-shortcode" controls="controls" height="360" id="video-7347-1" preload="metadata" width="640">
   <source src="https://d2908q01vomqb2.cloudfront.net/artifacts/DBSBlogs/messaging-1394/EUM_+demo.mp4?_=1" type="video/mp4" />
  </video>
 </div> 
</div> 
<h2>Future enhancements</h2> 
<p>We’re expanding to more messaging platforms, adding voice integration through <a href="https://aws.amazon.com/partners/featured/contact-center/" rel="noopener noreferrer" target="_blank">Amazon Connect</a>, and implementing predictive analytics for personalized recommendations. The serverless architecture makes these additions straightforward without infrastructure changes. Future scenarios could involve:</p> 
<ul> 
 <li><strong>Educator and student support </strong>– This solution can be enhanced for student and educator experiences. For educators, it can automate administrative tasks. For students, it can create personalized engagement campaigns, a communication approach that could be significantly more effective than traditional methods.</li> 
 <li><strong>Digital admission process flow </strong>– The solution integrates AWS Bedrock AI with WhatsApp Business API to streamline digital admissions. It can enable instant document verification, guide secure payments, and provide automated updates, all within the AWS End User Messaging WhatsApp channel. This AI-powered system could transform the complex admission process into an efficient, chat-based experience, benefiting both institutions and applicants.</li> 
 <li><strong>Parental support and study material management </strong>– The system could intelligently distribute learning resources based on student needs, send automated schedule updates, and provide personalized progress reports to parents through WhatsApp. Parents could receive AI-curated study materials and real-time updates about their child’s academic performance, homework assignments, and upcoming assessments through familiar chat interactions. This integration could transform traditional parent-teacher communication into an efficient, automated system while providing timely access to relevant educational resources.</li> 
</ul> 
<h2>Conclusion</h2> 
<p>The WhatsApp course recommender agent demonstrates how modern AWS services can create sophisticated, AI-powered conversational experiences that scale automatically and provide rich business insights. The serverless architecture provides cost-effectiveness while maintaining enterprise-grade reliability. Key architectural principles that make this solution successful include event-driven design for scalability, AI integration for natural interactions, semantic search for superior user experience, customizable analytics for business intelligence, and infrastructure as code (IaC) for reliable deployments.</p> 
<p>For organizations considering similar implementations, we recommend focusing on user experience optimization, robust error handling, comprehensive monitoring, and gradual feature rollout. The conversational AI environment is rapidly evolving, and solutions that prioritize user experience while maintaining technical excellence can drive the most business value. This implementation can serve as a reference architecture for building production-ready conversational AI systems on AWS, demonstrating patterns that can apply across industries and use cases.</p> 
<hr /> 
<h2 style="clear: both;">About the authors</h2>
