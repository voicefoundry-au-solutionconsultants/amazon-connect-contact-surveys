# Contact Surveys for Amazon Connect

Surveys are important as a diagnostic tool to fine-tune the experience and service delivered. They not only assess perceptions of experiences, but also help an organization understand customer motivations and intentions following the experience.

This project is aimed at delivering an end-to-end solution that will enable users to create and manage surveys, use these surveys with their Amazon Connect contact centre, and visualise their results.
Once deployed, users can access a secure web application to define surveys, consult / edit existing surveys, and visualise aggregated results per survey.

The solution is composed of:
- a web application for management
- a data-store to hold survey configuration
- a data-store to hold results
- a Contact Flow Module that will dynamically play the right survey for to a contact

Thanks to the flexible nature of this solution, surveys can be played in these scenario:
- reactively, after a contact (inbound voice, outbound voice) with a customer
- proactively, in combination with the startOutboundContact API

To find more information about how to implement these scenario, see [Example usage](#example-usage) 

## Architecture

![Architecture](/img/Architecture.png)

The solution deploys the required resources and follow the pattern:

1. Amazon S3 stores and serves the frontend static content through Amazon Cloudfront and restricted by Amazon Cognito for user management
2. The required Contact Flow Module is deployed in the Amazon Connect instance
3. Administrators use the web application to define contact surveys according to their needs
4. The configuration of the surveys is stored in Amazon DynamoDB
5. When required, the Contact Flow Module is invoked for a contact, with a contact attribute set to identify the survey to be offered
6. The Contact Flow Module retrieves the configuration of the survey for the contact
7. The contact is offered the survey, and the survey is answered
8. Results are stored in an Amazon DynamoDB table for the individual contact and if required, an Amazon Connect Task is created

## Deployment

To deploy this solution, you will need to have the following permissions in your AWS account:
- Create and manage S3 buckets
- Create and manage Amazon DynamoDB resources
- Create and manage AWS API Gateway resources
- Create and manage Amazon Cloudfront resources
- Create and manage Amazon Cognito resources
- Create and manage AWS Lambda resources
- Create and manage Amazon Connect resources

Typically, this solution should be deployed by a user with full access to your AWS environment.

1. Download this repository. If you choose to download the repository, make sure you unzip the archive on your disk before proceeding.

2. Log in to your AWS account.

3. Create a new S3 bucket that will contain the resources to be deployed.

4. In this new bucket, upload the content of the *deployment* folder obtained when cloning / downloading this repository.

![Files to upload](/img/files-to-upload.png)

5. Create a new Cloudformation stack, using the *contact-surveys-amazon-connect.yaml* template provided as part of the deployment resources.

6. Provide the required parameters:
- an email address for the initial user of the solution
- the ARN of the Amazon Connect instance you want to use with this solution
- the alias of the Amazon Connect instance you want to use with this solution
- the id of the Contact Flow to which task created by this solution will be sent to
- the name of the S3 bucket where you uploaded the resources at step 4

**Note:** If you are unsure about which Contact Flow to choose to process the tasks generated by the solution, use the id of the *Sample inbound flow (first contact experience)* available by default in your Amazon Connect instance.

7. Proceed with the the stack creation steps.

**Note:** It will take approximately 5 minutes for the stack to complete the deployment. You will receive an email containing a temporary password.

8. Once the stack is deployed, in the **Output** tab, note the value of the **AdminUser** and the URL of the application.

9. Navigate the the URL noted at step 8. Log in to the application with the *Username* collected at step 8, and the *password* received during the stack deployment.

**Note:** You will be required to change your password at first login.

10. If you see the following screen, the solution has been successfully deployed and you are good to go.

![Application Deployed Successfully](/img/application-deploy-success.png)

## Example usage

The following examples will help you understand how you can use this solution to cater for typical use cases. Make sure you have deployed the solution before going through each of the examples.

### Simple post-contact survey, with low-score flagged for review

In this example, you will learn to implement a simple post-contact survey that will alert a supervisor through an Amazon Connect Task every time a customer replies to a given question with a low score.

1. Create a new survey using the Contact Surveys for Amazon Connect application deployed with the solution:

![Simple survey example definition](/img/simple-survey-example-definition.png)

2. Add a couple of questions to your survey:

![Simple survey example questions](/img/simple-survey-example-questions.png)

3. For one of these questions, select **Additional settings**, tick the **Flag for review**, and define the threshold:

![Simple survey example flag](/img/simple-survey-example-flag.png)

Any contact that inputs a score lower than the defined threshold to that question will trigger a task in Amazon Connect. This task will be routed through the Contact Flow defined when the solution was deployed.

4. Save, refresh the list, and note the **Id** of your new survey.

![Simple survey example id](/img/simple-survey-example-id.png)

**Note:** the id of your survey will be different than the one on the screenshot.

5. In you Amazon Connect instance, create a new Contact Flow and import the *Survey Example Disconnect* flow. Before publishing, make sure that the **Invoke module** block is pointing to the *Contact Survey* module available in your Amazon Connect instance.

![Simple survey example disconnect flow](/img/simple-survey-example-disconnect-flow.png)

6. Repeat step 5 to import the *Simple Survey with flag* flow. Don't publish it just yet, you will need to make some configuration adjustments.

7. Locate the *Set Disconnect Flow* block. Configure this block to set the disconnect flow to the *Survey Example Disconnect* flow imported at step 5.

![Simple survey example set disconnect flow](/img/simple-survey-example-inbound-flow-1.png)

8. Locate the **Set Contact Attributes** block. This block sets a single *surveyId* contact attribute. Paste the *id* of your survey (noted at step 4) in the *Value* field of the *surveyId* contact attribute.

![Simple survey example set contact attribute](/img/simple-survey-example-inbound-flow-2.png)

**Note:** It is important to understand that what determines the survey that will be played to a contact is based on the value of the *surveyId* contact attribute. Here defining the survey to play for a contact in a static fashion. You could of course set this attribute dynamically based on, for example, choices made in an IVR, or other contact attributes.

9. Save and publish the flow.

**Note:** contacts processed by this flow will be queued on the *BasicQueue* which is available in your Amazon Connect instance. If you want to execute the test with a different queue, adjust the **Set working queue** block accordingly.

10. Associate the *Simple Survey Example* contact flow to any DID available to you for testing.

11. Place a call to the number you have selected for testing. Since we are intending to run this survey at the end of the customer interaction, make sure you have an agent available on the queue where the call is going to be directed.

12. While the customer stays on the line, end the call for the agent's end. The customer will be directed to the survey.

13. The customer hears the first question, and answer *2* (i.e. a score lower than the threshold defined at step 3). For the second question, the customer enters any digit between 0 and 5.

14. Since the customer has given a low score to our first question, a task is created for a supervisor to review. This task is processed by the Contact Flow defined during the deployment of the solution.

**Note:** for the purpose of this example, we have decided to use the *Sample inbound flow (first contact experience)* that is available to all Amazon Connect instances. This flow will direct the task to the *BasicQueue*. In a real world scenario, create a specific contact flow to handle the distribution logic of these tasks according to your needs.

15. The task is received by a supervisor and contains the details of the interaction that was poorly rated.

![Simple survey example task](/img/simple-survey-example-task-description.png)

16. You can visualise the aggregated results for your survey using the Contact Surveys for Amazon Connect application. Simply select the survey, and navigate to the **Results** tab. You can also export the individual results by clicking on the **Export** button.

![Simple survey example results](/img/simple-survey-example-results.png)