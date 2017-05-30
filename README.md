
# Lab 9 - Writing Cucumber JVM Tests
This lab covers creating a basic test using cucumber-jvm for testing REST APIs.

Cucumber JVM is a BDD style of writing tests. BDD or Behaviour Driven Development is a way to bridge the gap between developers, who can read code, and people who are less fluent in reading code.

This lab focuses on writing tests for [Lab1](/Lab1-BasicRestApi) for chassis-producer service. In the example below we have placed the cucumber test under chassis-labs-web module. This lab demonstrates how to have cucumber tests executed as integration tests because Cucumber JVM uses RestAssured libraries to invoke APIs and they can be seen as a substitute to current ITs. 

One more advantage is that these tests can be executed using maven's devint-test profile. The [devint-test](/Lab9-AddCucumberJVMTests/chassis-labs-web/pom.xml) profile uses maven-failsafe-plugin which helps to run the integration tests. Using this approach you are not required to spin up your cargo container explicitly. The devint-test profile take cares of it. This profile spins up a cargo container, runs your test against localhost and than tears down the server.

#### Prerequisites

This lab builds on top of a completed [Lab 1](/Lab1-BasicRestApi) project. You can use your own finished Lab 1 project or build on top of Lab 1 from this repository, see these instructions - [Building Individual Labs](/README.md#building-individual-labs-from-this-repo).

### Lab Steps

1. [Create a Feature File](#create-a-feature-file)
2. [Define Cucumber Runner](#define-cucumber-runner)
3. [Add Step Definitions](#add-step-definitions)
4. [Enable Jenkins Integration](#enable-jenkins-integration)

## Create a Feature File

This is the first place to start writing your cucumber test.

The following Gherkins file shows a scenario for invoking a POST API by providing Header, Post, Body, and End Point information.

```Gherkin
Feature: I want to test chassis-lab01 to validate customer and accounts api

  Scenario Outline: Validate success post response for accounts api
    Given the Chassis the header information
      | Api-Key      | RTM                   |
      | User-Id      | TEST                  |
      | Content-Type | application/json; v=3 |
      | Accept       | application/json; v=3 |
    And post message body "<PostBody>"
    When I make a rest call to the service "<ServiceURL>" using POST
    Then response code be"<ResponseCode>"

    Examples:
      | ServiceURL                                   | ResponseCode | PostBody                   |
      | chassis-refapp-producer-web/refapp/accounts/ | 201          | postBody/post-message.json |
```

#### Terms

- **Feature** indicates to Cucumber that this is a feature file
- **Scenario Outline** start a new scenario
- **Given, When, Then** scenario steps which are executed

## Define Cucumber Runner
In order to run the feature we create a runner class CucumberRunnerIt which contains Cucumber execution parameters.

```Java
package com.capitalone.chassis.api.labs.cucumber;

import cucumber.api.CucumberOptions;
import cucumber.api.junit.Cucumber;
import org.junit.runner.RunWith;

@RunWith(Cucumber.class)
@CucumberOptions(
        format = {"html:target/TestResults/html/cucumber-html-report", "json:target/TestResults/json/TestRunner-reports.json"},
        features = {"src/test/resources/feature/test.feature"},
        glue = "com.capitalone.chassis.api.labs.cucumber")
public class CucumberRunnerIt {
}

```

#### Terms

- **@RunWith(Cucumber.class)** indicates that we want Cucumber to run this test
- **@CucumberOptions** indicates the output formatting and location of the feature file

Also update the pom.xml of chassis-labs-web module to add required dependencies for cucumber.

```xml
    <dependency>
      <groupId>info.cukes</groupId>
      <artifactId>cucumber-junit</artifactId>
    </dependency>
    <dependency>
      <groupId>info.cukes</groupId>
      <artifactId>cucumber-java</artifactId>
    </dependency>
```

You might have noticed that version numbers are not specified above. It's because those are getting pulled from chassis-dep-bom from chassis-framework.

So in next step let's write down our StepDefinitions class to have the implementation for our test cases.

## Add Step Definitions

Below is the code for StepDefinitions to handle POST scenario. Check out the implementation of other methods in [StepDefinitions.java](/Lab9-AddCucumberJVMTests/chassis-labs-web/src/test/java/com/capitalone/chassis/api/labs/cucumber/StepDefinitions.java)

```Java
package com.capitalone.chassis.steps;

import com.capitalone.chassis.engine.model.exception.ChassisSystemException;
import com.capitalone.chassis.it.test.ApiRestClassRule;
import com.jayway.restassured.response.Response;
import cucumber.api.java.en.Given;
import cucumber.api.java.en.Then;
import cucumber.api.java.en.When;
import org.junit.ClassRule;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.Map;

import static com.jayway.restassured.RestAssured.given;
import static org.hamcrest.CoreMatchers.is;
import static org.hamcrest.MatcherAssert.assertThat;

public class StepDefinitions {

        @ClassRule
        private static ApiRestClassRule apiRule = new ApiRestClassRule("/chassis-labs-web");
    
        private Map<String, String> headerMap = null;
        private String postBdy = null;
    
        private Response resp=null;
    
        private String jsonText;

        @Given("^the header information$")
        public void the_header_information(Map<String, String> tblHeader) throws Throwable {
            this.headerMap = tblHeader;
        }
    
        @Given("^post message body \"(.*?)\"$")
        public void post_message_body(String postBody) throws Throwable {
            this.postBdy = postBody;
        }
    
        @When("^I make a rest call to the service \"(.*?)\" using POST$")
        public void i_make_a_rest_call_to_the_service_using_POST(String serviceUrl) throws Throwable {
            Path postBodyPath = Paths.get(ClassLoader.getSystemResource(postBdy).toURI());
            jsonText = readFile(postBodyPath);
            resp =  given().urlEncodingEnabled(false).log().all().
                    headers(headerMap).body(jsonText).
                    when().
                    post(serviceUrl).prettyPeek();
        }
        
        @Then("^response code be same as \"(.*?)\"$")
           public void response_code_be_same_as(String respCode) throws Throwable {
           assertThat(respCode, is(String.valueOf(resp.getStatusCode())));
        }
        
        private String readFile(Path filename) {
        
                String content;
                try {
                    content = new String(Files.readAllBytes(filename));
                } catch (IOException e) {
                    throw new ChassisSystemException("There was an error ", e);
                }
                return content;
        
            }

}
```

Under the resources folder, add files to hold postmessage body.

### postBody/post-message.json

```json
{
  "bankTypeCode": "MOBI",
  "bankTypeDescription": "BANK",
  "name": "Banking for good"
}
```

Run <code>mvn clean install -Pdevint-test</code> and you should see the following in the console output for the test execution.

```
-------------------------------------------------------
 T E S T S
-------------------------------------------------------
Running com.capitalone.chassis.cucumber.RunCucumberTest
ERROR StatusLogger No log4j2 configuration file found. Using default configuration: logging only errors to the console.
Request method: POST
Request path:   http://localhost:9090/chassis-labs-web/labs/accounts
Proxy:                  <none>
Request params: <none>
Query params:   <none>
Form params:    <none>
Path params:    <none>
Multiparts:             <none>
Headers:                Accept=application/json; v=3
                                Api-Key=RTM
                                User-Id=TEST
                                Content-Type=application/json; v=3; charset=ISO-8859-1
Cookies:                <none>
Body:
{
    "bankTypeCode": "MOBI",
    "bankTypeDescription": "BANK",
    "name": "Banking for good"
}

1 Scenarios (1 passed)
4 Steps (4 passed)
0m1.079s

Tests run: 5, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 1.354 sec - in com.capitalone.chassis.cucumber.RunCucumberTest

Results :

Tests run: 5, Failures: 0, Errors: 0, Skipped: 0

[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 6.509s
[INFO] Finished at: Thu Mar 10 11:12:42 EST 2016
[INFO] Final Memory: 33M/981M
[INFO] ------------------------------------------------------------------------
```

The console output describes that test went successful.


To see more scenarios, clone the framework-labs repositories with following commands:

```
git clone https://github.kdc.capitalone.com/chassis-labs/framework-labs.git
cd framework-labs
git checkout Lab9-AddCucumberJVMTests
mvn clean test -Pdevint-test
```

## Enable Jenkins Integration

Cucumber tests can be integrated with Jenkins and can run against an AWS instance.

[Enterprise Jenkins](https://digitaljenkins.kdc.capitalone.com/digitaljenkins/) is configured with a Cucumber JVM plugin.

Add a Post-build Actions in your job and Select Publish Cucumber results as report.

<kbd>![screenshot](/Lab9-AddCucumberJVMTests/images/post-build.png)</kbd>

After you add this step, specify the path to the Cucumber json file, which generates in the target folder.

You can see the cucumber icon as part of your job as shown in the screenshot below.

<kbd>![screenshot](/Lab9-AddCucumberJVMTests/images/reports-icon.png)</kbd>

Click on Cucumber reports and view the results.

<kbd>![screenshot](/Lab9-AddCucumberJVMTests/images/results.png)</kbd>

<HR>

Good Job! Please see the Chassis Lab [Table of Contents](https://github.kdc.capitalone.com/chassis-labs/framework-labs) for more labs.

## Contribute a Lab
Have an idea for a lab that's not listed here? You can contribute your own lab to this repo via a pull request. Here are some technical guidelines on how to [contribute a lab](../HowToPortANewLab.md).
