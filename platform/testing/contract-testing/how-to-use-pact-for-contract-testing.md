# How to use Pact for contract testing (in progress)

#### Table of contents

- [Introduction](#introduction)
  * [Terminology](#terminology)
  * [Requirements](#requirements-draft)
  * [Process](#process)
  * [Implementation details](#implementation-details)
    + [Naming Conventions](#naming-conventions)
	+ [Naming Collisions](#naming-collisions)
  * [FE/BE Communications](#fe-be-communications)
    + [Example Workflow for Search](#example-workflow-for-search)
	+ [Communication and Interaction](#communication-and-interaction)
    + [Response Types and Matching](#response-types-and-matching)
- [Configuring the `vets-website` consumer codebase](#configuring-the-vets-website-consumer-codebase)
  * [Running contract tests](#running-contract-tests)
  * [Writing a contract test](#writing-a-contract-test)
      - [example-app.pact.spec.js](#example-apppactspecjs)
  * [Setting up interactions](#setting-up-interactions)
    + [Provider states](#provider-states)
    + [Matching](#matching)
    + [Responses](#responses)
    + [Optional attributes](#optional-attributes)
    + [Optional arrays](#optional-arrays)
  * [Troubleshooting](#troubleshooting)
    + [Problems connecting to port 3000](#problems-connecting-to-port-3000)
    + [Expected interactions did not match actual](#expected-interactions-did-not-match-actual)
- [Configuring the `vets-api` provider codebase](#configuring-the-vets-api-provider-codebase)
  * [1. Set up a provider state](#1-set-up-a-provider-state)
    + [Using a local file if blocked by frontend](#using-a-local-file-if-blocked-by-frontend)
    + [Expected responses](#expected-responses)
    + [Naming Guidelines](#naming-guidelines)
    + [Authorization](#authorization)
    + [VCR](#vcr)
	+ [Flipper](#flipper)
  * [2. Adjust developer configurations](#2-adjust-developer-configurations)
    + [Configure the `pact_uri/broker_url`](#configure-the--pact-uri-broker-url-)
  * [3. Run the verification task](#3-run-the-verification-task)
    + [Important: Docker workflow settings](#important--docker-workflow-settings)
  * [4. Verify your results](#4-verify-your-results)
  * [5. Remove developer configurations](#5-remove-developer-configurations)
  *  [Broker matrix and tagging](#broker-matrix-and-tagging)
  *  [CircleCI](#circleci)
  *  [CircleCI and the Build Process](#circleci-and-the-build-process)
     + [Deploy Strategy and Tagging](#deploy-strategy-and-tagging)
	 + [Frontend](#frontend)
	 + [Backend](#backend)
  *  [Pact FAQ](#pact-faq)

## Introduction

Pact is a tool that enables consumer driven contract testing (CDCT) by defining a contract between service consumers and providers (e.g.  `vets-website`  and  `vets-api`). As explained by Pact, “Contract testing is a technique for testing an integration point by checking each application in isolation to ensure the messages it sends or receives conform to a shared understanding that is documented in a 'contract'.“

Pact is most valuable for designing and testing integrations where your organization controls the development of both the consumer (`vets-website`) and the provider (`vets-api`), and the requirements of the consumer are going to be used to drive the features of the provider.

Pact enables VFS teams to test integration points with vets-api. This gives VFS teams the ability to ensure that their app is production-ready, and helps VSP ensure that platform-wide changes are non-breaking. Pact allows VFS and VSP to catch issues with integrations before they reach production, and satisfies the need for end-to-end integration tests between  `vets-website`  and  `vets-api`  as part of VA.gov's automated testing processes. 

[Learn more about Pact](https://docs.pact.io/)

### Terminology

* **Consumer** -- The consumer `vets-website` generates pacts.
* **Provider** -- The provider `vets-api` verifies pacts.
* **Pact** -- A contract between a consumer and provider is called a _pact_. Each pact is a collection of _interactions_.
* **Interaction** -- A request and response pair.
* **Broker** -- Central location where pacts are hosted. The Pact broker is currently hosted on [Heroku](https://vagov-pact-broker.herokuapp.com/). You can view the interactions per endpoint and the verification matrix from the broker index.

### Requirements (draft)

VFS teams are required to use Pact for contract testing in the following scenarios: 

* Any new or updated product or service that integrates with vets-api endpoints is required to use Pact
   * Any new UI on VA.gov that integrates with vets-api (whether the endpoint you are integrating with is net new or not)
      * If your team is updating an existing UI on VA.gov, Pact contract testing is required if the changes entail calling an endpoint that the app did not previously call (new endpoints or changes to endpoints)
   * Any new or updated external product or service that lives outside of the VA.gov ecosystem and integrates with vets-api endpoints (e.g. Lighthouse)

To meet this requirement, your team will need to either create a Pact test or update an existing Pact test for the vets-api endpoints used. Integrations that only use vets-api as a proxy to another service are exempt from this requirement.

VSP does not currently actively enforce this requirement (last updated 9/2020). 

PRs related to Pacts will go through the standard [code review process](https://github.com/department-of-veterans-affairs/va.gov-team/blob/master/platform/engineering/code_review_guidelines.md). 

### Process

1. `vets-website`  runs unit tests to validate its request and response interactions with `vets-api` endpoints.
2. `vets-website` generates contracts (referred to as **pacts** in the Pact framework) 
3. Pacts publish to a central location/broker ([Heroku](https://vagov-pact-broker.herokuapp.com/)) to be versioned and shared with the provider (`vets-api`).
4. The provider (`vets-api`) runs a rake task to verify the pacts. This verification task replays the requests defined in the pacts against the provider (`vets-api`) and validates that the actual response matches the expected response.
5. Results of the verification are published back to the broker.

Builds depend on successful verification results to deploy.

*Note: In future iterations of the VSP Pact Process, we plan to integrate the rake verification task into CircleCI as part of the build step.* 


![](https://i.imgur.com/zQMyDS0.png)

### Implementation details

Pact is language agnostic and has packages for both Node.js and Ruby, so both `vets-website` and `vets-api` have access to their language-specific implementations of the Pact spec.
- `vets-website` will install the `@pact-foundation/pact` package.
- `vets-api` will install the `pact` gem.

**Provider states** -- Allows the consumer to define a state in which the provider should be in when making making a request for response codes, data, etc. Provider states define the state of the provider and how it will handle a response given it's current state and the data that should exist.


**Pact helper** - Pacts verified by the `pact:verify` task and configured in the `pact_helper.rb` file in your provider codebase. Currently, the `pact_helper` implements rspec and pact configurations as well as `git_sh`a and `git_branch tagging` for the consumer and provider in the broker. 

#### Naming Conventions

The purpose of contract testing is to ensure that the consumer and provider have a shared understanding of the messages that will pass between them. **To that end, we follow a policy of explicit or descriptive naming when defining interactions and provider states.** The goal is for developers to understand any interaction in a pact. 

Pacts involve a consumer and provider pair that are named from the pact tests written on the consumer side (in vets-website). When writing pact tests, follow these naming conventions:
- The consumers are the apps in vets-website and should be named accordingly. Every app's `manifest.json` has an `appName` value that may be referenced as the consumer name, whether in full or in shorthand (as long as it's clear).
- The provider is vets-api and should be named as "VA.gov API" in the pacts.
- For example, [this pact in the broker](https://vagov-pact-broker.herokuapp.com/pacts/provider/VA.gov%20API/consumer/Search/latest) describes the interactions between the Search app and VA.gov API.

Provider states are namespaced also by pairings of consumer and provider. When defining states in vets-api, the provider in this context is always "VA.gov API", so sets of states are effectively namespaced per consumer using provider state files. A provider state file defines all the provider states used in the pact for a given consumer.
- In vets-api, provider state files are located in `spec/services_consumers/provider_states` and named after the vets-website apps that are consumers vets-api.
- For example, [the provider state file for search](https://github.com/department-of-veterans-affairs/vets-api/blob/master/spec/service_consumers/provider_states_for/search.rb) has a state described as “at least one matching result exists”, which sets up the expected preconditions for any interaction that uses this state in the search pact.


#### Naming Collisions

Interaction/provider state naming collisions will NOT exist outside of a consumer namespace, but could exist within the same consumer namespace. Ex: If a Search contract has an interaction defined as “test interaction” and the User contract has defined interaction of “test interaction” a naming collision will not exist because the two interactions exist within separate consumer namespaces. 

The above described collision COULD exist within the same consumer namespace, so developers need to be explicit as to the state of the interaction when developing interactions within a consumer namespace. E.g. Search

### FE/BE Communications

When in the development process, FE and BE engineers may need to organize communication efforts. The pact workflow is a collaborative effort that developers will need to iterate on. As a first step in the process, the FE engineer will push a pact to the broker. The BE engineer will then use the pact from the broker to set up a matching provider state(s). Provider states follow strict naming conventions for the given consumer and its respective interactions. Often times, there will need to be adjustments to the expected response/requests defined in the pact following verification on the backend. 

#### Example Workflow for Search
In the search example, an interaction exists for the search contract with the name of “at least one matching result exists”
On the [backend](https://github.com/department-of-veterans-affairs/vets-api/blob/master/spec/service_consumers/provider_states_for/search.rb), a provider state matching the search consumer will need to be set up as an exact match the interaction defined in the pact
If an adjustment needs to be made to a request/response defined in a pact, then the communication efforts will need to be coordinated between FE and BE engineers. 

#### Communication and Interaction
Once the BE engineer runs the pact verification task, possible adjustments to the expected requests and responses defined in the pact may be needed. BE and FE engineers will need to communicate pact request/response adjustments as necessary during the development process. Pact again is a collaborative effort and by no means does it amount to the development process being one sided. The development iterations should have equal efforts from the respective backend and frontend team members. 


#### Response Types and Matching
Pact contracts test response types and not all response permutations. For example, if a response attribute is expected to be a string, you don’t necessarily care about the exact value, but rather that the value is present and it’s particular type. Ex: “Test string 1”, vs “Test String 2”. Testing response types ensures that the provider actually does provide the response the consumer expects. Often times the request and response values are difficult to determine beforehand( ex: timestamps or ids) and this is where regular expressions come into play when determining types. See [pact matching documentation](https://docs.pact.io/implementation_guides/ruby/matching/)

It’s important to remember that pact contract testing is not functional testing.  Contract tests focus on the messages that flow between a consumer and provider, while functional tests ensure that the correct side effects have occured as well. When writing a test for an interaction, ask yourself what you are trying to cover. The goal here is not to create unnecessarily tight contracts and dig into the business logic, but rather verify that Consumers and Providers have a shared understanding of what requests and responses should be.


## Configuring the `vets-website` consumer codebase 

### Running contract tests

To run all contract tests locally:

```sh
yarn test:contract
```

To run a specific contract test:

```sh
yarn test:unit src/applications/my-app/tests/example.pact.spec.js
```

### Writing a contract test

1. Create a test file with the suffix `.pact.spec.js`.

2. Use the `contractTest` helper function from `src/platform/testing/contract`
   ```js
   contractTest('Example App', 'VA.gov API', mockApi => { ... });
   ```
   - `Example App` is the name of your app. This must match the app name that the API uses to set up provider states during pact verification for this app.
   - `VA.gov API`  is the name of the provider.
   - `mockApi => { ... }`  is a callback function where the argument `mockApi` is a function that returns the Pact mock provider.
     - In the callback function, write your tests in a Mocha unit test format using `describe()` and `it()` blocks, [hooks](https://mochajs.org/#hooks), and such.
     - `describe` blocks are generally useful for describing pairings of HTTP verbs and endpoints being consumed.
     - `context` blocks can further organize the `describe` blocks by provider states.
     - `it` blocks represent tests for specific interactions and will almost certainly always have asynchronous (async) callbacks.
       - This is because `fetch` requests as well as the [Pact consumer API](https://github.com/pact-foundation/pact-js#api) are async operations.
       - Refer to [Mocha's documentation on asynchronous code](https://mochajs.org/#asynchronous-code) for examples of writing async tests in Mocha.
         We prefer the `async`/`await` approach, and the following examples will use that.

3. For any relevant endpoints, set up the mock API by adding the expected interactions.

   See the next section on interactions for an example of defining an interaction.

   ```js
   await mockApi().addInteraction(interaction);
   ```

   - The easiest place to do this is within the `it()` blocks, which are meant to focus on specific interactions or endpoints.
   - **The test will fail if that interaction is not fulfilled in the scope where it's declared**. So ensure that the request made in the test matches the request defined in the interaction.

The following code is an example of how you might structure your test.

##### example-app.pact.spec.js

```js
import contractTest from 'platform/testing/contract';

import { getStuff, sendThings } from  '../actions';

contractTest('Example App', 'VA.gov API', mockApi => {
  describe('GET /example_endpoint', () => {
    context('when user is authenticated', () => {
      it('responds with status 200', async () => {
        mockApi().addInteraction({
          // code 200 interaction for GET /example_endpoint
        });
        await getStuff();
      });

      it('responds with status 500', async () => {
         mockApi().addInteraction({
          // code 500 interaction for GET /example_endpoint
         });
         await getStuff();
      });
    });
  });

  describe('POST /example_endpoint', () => {
    it('responds with status 200', async () => {
      mockApi().addInteraction({
        // code 200 interaction for POST /example_endpoint
      });
      await sendThings();
    });
  });
});
```

### Setting up interactions

An interaction describes the request you expect to send to the API and the expected response for that request. It's formatted as an object.

The [expected response](#responses) typically uses [matchers](#matching) to allow more flexible matching against what the API actually returns, reducing the brittleness of the provider verification.

```js
import { Matchers } from '@pact-foundation/pact';
const { boolean, string } = Matchers;

const interaction = {
  state: 'user is authenticated',
  uponReceiving: 'a request to get stuff',
  withRequest: {
    method: 'POST',
    path: '/v0/example_endpoint',
    headers: {
      'Content-Type': 'application/json',
      'X-Key-Inflection': 'camel',
    },
    body: {
      foo: 12345,
      bar: 'request data',
    },
  },
  willRespondWith: {
    status: 200,
    body: {
      data: {
        baz: string('response data'),
        quux: boolean(false),
      },
    },
  },
}
```

#### Provider states

The `state` key in an interaction specifies the state of the backend, or the *provider state*, for that interaction. Use provider states to test different responses to the same request.

The state provides a hook for the backend to do any set up before running the verification on its end. The backend often depends on an external services, so provider states are helpful indicators to stub out such services accordingly.

Examples:
- `state: 'external service is up'`
- `state: 'user is authenticated'`

The same state can be used across multiple interactions.
- Avoid naming collisions (within your app) if there are any differences in the setup on the backend.
- Describe the states appropriately if there are any minor differences.
- Don't bloat the contract with a bunch of edge cases if it's not valuable to test or could be tested elsewhere.

The backend needs to handle all states in a pact in order for the verification to fully proceed.

#### Matching

[Use matchers](https://github.com/pact-foundation/pact-js#matching) in your interactions to prevent brittle tests. They allow tests to match the types or formats of values in the requests or responses instead of expecting the exact values.

- "As a rule of thumb, [you generally want to use exact matching when you're setting up the expectations for a request](https://docs.pact.io/getting_started/matching/#request-matching) because you're under control of the data at this stage, and according to Postel's Law, we want to be "strict" with what we send out."
- "[You want to be as loose as possible with the matching for the response though](https://docs.pact.io/getting_started/matching/#response-matching). This stops the tests being brittle on the provider side."

#### Responses

The expected response is defined with `willRespondWith`. Define the expected `status` and use [matchers](#matching) to define the expected `body`. You can also define the expected `headers`.

**If the interaction involves a request that's made with the `apiRequest` helper, you will need to include an expectation for the `Content-Type` header as follows.**
- This is because the helper only parses the response as JSON if it detects the content type as such, so the test needs to be explicit about that.
- Note the use of the `term` matcher to match with the actual `Content-Type` header value that the API returns.

```js
import { Matchers } from '@pact-foundation/pact';
const { term } = Matchers;

...

willRespondWith: {
  status: 200,
  headers: {
    'Content-Type': term({
      matcher: '^application/json',
      generate: 'application/json',
    }),
  },
  body: {
    ...
  },
},
```

#### Optional attributes

[Pact does not support the verification of optional attributes](https://docs.pact.io/faq/#why-is-there-no-support-for-specifying-optional-attributes).

Any attributes in the expected response must be included in the actual response. If any attributes are missing from the actual response during provider verification, the verification will fail.

If an attribute is optional, the way to test it would be to use provider states to force the inclusion of that attribute. But only include it in your app's contract if that attribute is actually used by the app.

Without support for verifying optional attributes, that also means there is no matcher that is flexible enough to allow arrays that could either be empty or contain elements.

#### Optional arrays

If you want to vary the data within an array for your test case, but that array could be empty in a valid response, [best practice dictates the following approach](https://stackoverflow.com/a/61786715/1070621):

1. Decide on what is valuable to test - empty arrays, non-empty or both.
2. Use provider states to specify any variations on the response (consumer test).
3. Implement the state for the provider test to be able to control the response.

Following that convention, there could potentially be multiple interactions, with the responses resembling these examples:

**Empty arrays**

```js
willRespondWith: {
  status: 200,
  body: {
    facilities: [],
    services: [],
  }
}
```

**Non-empty arrays**

```js
willRespondWith: {
  status: 200,
  body: {
    facilities: eachLike({
      id: string('12345'),
      name: string('Central VA Office'),
    }),
    services: eachLike('user-profile'),
  }
}
```

**Specific arrays**

If you need the generated array to contain specific elements, you would set the value to the **exact array** you want and use a provider state to test this particular interaction.

This generated response is useful if you wanted to use the generated pact as a source of mock data, such as for a stub server in local development or end-to-end tests.

*Warning: Do not bloat the pact with interactions like this if they are not going to be used in any meaningful way.*

```js
const interaction = {
  state: 'mock data',
  ...
  willRespondWith: {
    status: 200,
    body: {
      facilities: [
        {
          id: string('12345'),
          name: string('Central VA Office')
        },
        {
          id: string('67890'),
          name: string('Department of Veterans Services'),
        },
      ],
      services: ['user-profile', 'edu-benefits'],
    },
  },
};
```

In this case, using a matcher, like `eachLike`, would not be appropriate.
- You can't do `eachLike(['user-profile', 'edu-benefits'])` because the matcher doesn't work that way.  [However, there is some discussion for adding this feature to v4](https://github.com/pact-foundation/pact-js/issues/402) (as of now, we are on the v2 spec).
- If you passed the option to increase the minimum length of the array, like `eachLike('user-profile', { min: 2 })`, the generated array would only repeat the given element, like `['user-profile', 'user-profile']`. This wouldn't be useful if you're trying to vary the data.
- You could still use matchers if you don't care about the exact values of certain attributes, like how `string()` is used in the above example.

### Troubleshooting

In general, after a test failure, it may be helpful to refer to `logs/pact.log` for more detailed information.

#### Problems connecting to port 3000

If there are any processes or previously interrupted tests running on port 3000, stop them before running your contract tests.

```sh
kill $(lsof -ti :3000)
```

The port is reserved for running the Pact mock server as a substitute for a locally running API.

Problems starting the mock server can also manifest in warnings that look like this:

```sh
UnhandledPromiseRejectionWarning: <!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<title>Error</title>
</head>
<body>
<pre>Cannot POST /interactions</pre>
</body>
</html>
```

#### Expected interactions did not match actual

```sh
"after each" hook: verify interactions for "[your test case name here]":
Error: Pact verification failed - expected interactions did not match actual.
```

There are some possibilities to double-check with the request or response.
1. The request might not match the URL from the interaction.
2. The contents (headers or boddy) of the request or response might not match what you defined in the interaction.

When debugging, it might help to inspect the properties of the request and response at different steps in the function making the request.

## Configuring the `vets-api` provider codebase

### 1. Set up a provider state

Once frontend has pushed a pact to the broker, a corresponding provider state will need to be defined (even if not necessary) on the backend.

*Important: If a provider state is not necessary, please define no_op inside of a wrapping provider state block.*

A custom generator is in place to create a boiler plate provider state file for a given consumer. Just run ```rails pact new your_consumer_name_here```. See the [provider state generator USAGE file](https://github.com/department-of-veterans-affairs/vets-api/blob/master/lib/generators/provider_state/USAGE) for further instructions.

Please see the [provider state documentation](https://github.com/pact-foundation/pact-ruby/wiki/Provider-states#provider-codebase) for provider state instructions. Additionally, see the [search example](https://github.com/department-of-veterans-affairs/vets-api/blob/master/spec/service_consumers/provider_states_for/search.rb#L3).

[Base states](https://docs.pact.io/implementation_guides/ruby/provider_states/#base-state) can be used to DRY up provider state files if the interaction requires code that should run before/after each interaction. For additional provider state settings see the [pact documentation](https://docs.pact.io/implementation_guides/ruby/provider_states/)

#### Using a local file if blocked by frontend 

If you're waiting on frontend to generate the contract and push to the broker, you can use a temporary (local) contract. You can verify a pact at any arbitrary local or remote URL using the `pact:verify:at` task. 

Example local file path:

```
rake pact:verify:at[tmp/hca-va.gov_api.json]
```

#### Expected responses

To determine an endpoints expected response, look at the defined response in the broker. If you're having trouble identifying expected responses, look at `vets-api` request specs to determine provider state implementation details and expected responses.

#### Naming Guidelines

* Provider states -- Provider states are defined in the `service_consumers/provider_states_for/*.rb` directory. Provider States must follow a strict naming protocol and are categorized by consumer name per pact. (Example: Search, Users, HCA, etc.) See the [Consumer column of the Heroku Pact broker](https://vagov-pact-broker.herokuapp.com/) for examples. 

* Consumer name -- The consumer name in your defined `provider_state_for` block must match the name of the consumer configured in your consumer project for the verification task to correctly find the provider states. See [Search Example](https://github.com/department-of-veterans-affairs/vets-api/blob/master/spec/service_consumers/provider_states_for/search.rb#L3)

* Provider state block -- The [provider state block](https://github.com/department-of-veterans-affairs/vets-api/blob/master/spec/service_consumers/provider_states_for/search.rb#L4) must match the corresponding definition defined in the pact as well. In the [search example](https://vagov-pact-broker.herokuapp.com/pacts/provider/VA.gov%20API/consumer/Search/latest) a "multiple matching results exist" interaction is defined and a provider state matching this defintion will need to be defined on the [backend](https://github.com/department-of-veterans-affairs/vets-api/pull/4612/files#diff-102a9104474b45510528e3e28a8071c0R4). 

#### Authorization

If an authorized `vets-api` user is needed for a request to an endpoint, a helper method can be used to stub the session. Call the `build_user_and_stub_session` [method](https://github.com/department-of-veterans-affairs/vets-api/blob/master/spec/support/stub_session.rb#L3) if your endpoint requires an authorized (signed in) user. See this [forms example](https://github.com/department-of-veterans-affairs/vets-api/blob/master/spec/service_consumers/provider_states_for/hca.rb).

#### VCR

Many of the `vets-api` endpoints call out to external services. To mock external service calls, the pact helper has VCR configured for usage. VCR cassettes can be used to mock external service responses for many of the `vets-api` third party service calls. The defined provider state in the [search example](https://vagov-pact-broker.herokuapp.com/pacts/provider/VA.gov%20API/consumer/Search/latest) makes use of a VCR cassette.

***Important***:
If using cassettes across provider states in the same file, you may want to explicily define the VCR cassette ejection. ```VCR.eject_cassette('search/success_utf8')``` There was a bug noted that the provider state tear downs were ejecting all cassettes and not just the one they load, resulting in odd VCR behavior.

#### Flipper
In the case that you don't have VCR tapes to cover multiple states for various interactions, you may need to utilize flipper feature toggle functionality. 

### 2. Adjust developer configurations

#### Configure the `pact_uri/broker_url`

To work with only one pact in the broker, you can verify a pact at any remote URL using the `pact:verify:at task`. Otherwise, the rake task will run all the pacts pushed to the [heroku broker](https://vagov-pact-broker.herokuapp.com/).

```
rake pact:verify:at[https://vagov-pact-broker.herokuapp.com/pacts/provider/VA.gov%20API/consumer/Search/latest]

OR via docker flow

make pact PACT_URI=https://vagov-pact-broker.herokuapp.com/pacts/provider/VA.gov%20API/consumer/Search/latest
```

*Note: If you are blocked by the frontend, you can [point to a local file path](#using-a-local-file-if%20blocked-by-frontend)*

### 3. Run the verification task

By running the pact rake task (`bundle exec rake pact:verify or make pact`), dynamic rspec tests will spin up to validate expected responses defined per pact.

Run:
```
bundle exec rake pact:verify 

OR

make pact

OR 

# if following the docker workflow and working with only one pact, 
# you can pass in a pact uri (broker url or local path) to the docker pact makefile target

make pact PACT_URI=https://vagov-pact-broker.herokuapp.com/pacts/provider/VA.gov%20API/consumer/Search/latest
# broker url example

make pact PACT_URI=tmp/hca-va.gov_api.json
# local path example
```

#### Important: Docker workflow settings

If following the docker workflow, make the following additions/changes to `config/test.yml`. Do not commit these changes.

```
test_database_url: postgis://postgres:password@postgres:5432/vets_api_test
```

*If following the native flow, remove this setting from test.yml.*

The verification task can be run at any point in development, but it may be helpful to run frequently to point out failures during development iterations. 

***Troubleshooting*** You may encounter the following error in the console when following the Docker workflow:

``` 
     PG::ConnectionBad:
       could not connect to server: No such file or directory
       	Is the server running locally and accepting
       	connections on Unix domain socket "/var/run/postgresql/.s.PGSQL.5432"?

```

If you're still seeing the error after setting `test_database_url` in `config/test.yml`, try explicitly setting `test.url` in `config/database.yml` instead, but don't commit this change.

``` 
test:
  url: postgis://postgres:password@postgres:5432/vets_api_test

```


### 4. Verify your results

When the verification task completes, passing (green) and failing (red) interactions will display in the console. See example console output below. Verification status can also be viewed on the broker index page and in the verification matrix after commiting a change and CI runs the build workflow. See [broker matrix section](#broker-matrix-and-tagging) below.

The diff formatter when running the verification task can be configured for development purposes based on personal preference. The default configuration is the list format. See [diff_formatter documentation](https://docs.pact.io/implementation_guides/ruby/configuration/#diff_formatter)

![](https://i.imgur.com/7scCEhi.png)

### 5. Remove developer configurations

When your verification status is all green, please reconfigure your changes from step 2 in the by doing the following:
1. Reconfigure the `pact_broker_base_url`
2. Remove the temporary `pact_uri` definition. 

### Broker matrix and tagging

The verification matrix acts as a success metric for verification status (green or red). See the [search example](https://vagov-pact-broker.herokuapp.com/matrix/provider/VA.gov%20API/consumer/Search) in the pact broker for a provider verification matrix.

Additionally, each verification run is tagged with the Git branch name and Git SHA in the provider verification column to track provider version details. See details in the [pact_helper](https://github.com/department-of-veterans-affairs/vets-api/blob/master/spec/service_consumers/pact_helper.rb#L50-L51).

### CircleCI

A CircleCI config has been implemented to run workflows with various jobs as part of the build for vets-api. Additionally, a separate workflow (pact_verification) is implemented for webhook usage to auto run the pact verification task in CI when contracts are published to the pact broker.

The webhook is configured for contract changed events (when new or updated contracts are published). This webhook will trigger the pact_verification workflow when the verify_stable_pacts pipeline parameter is set to true. In the latest version of the CircleCI API, v2, the ability to trigger specific jobs is depreciated, so usage of pipeline parameters is the only way to mimic the behavior of triggering a specific job.

If the pacts are coming from a feature branch and the API has not been updated to match the new expectations, verification should be expected to fail. In that case, once the pacts have been published, the BE developers working on updating the API can run the following task during development to verify their changes:

```
bundle exec rake pact:verify:at[http://your-pact-broker/pacts/provider/PROVIDER/consumer/CONSUMER/version/CONSUMER_VERSION]
```

The actual URL should point to the pact that was published from the vets-website feature branch. A specific URL might look something like this, where the version is the commit hash on the feature branch that created the new pact:

```
https://vagov-pact-broker.herokuapp.com/pacts/provider/VA.gov%20API/consumer/HCA/version/d553c678bbdf1963fe3e27250eebc7c17b26fd55
```

After the vets-api feature branch is merged to master, that pact should be able to pass verification.

The CircleCI build for vets-api is still under construction with respect to parallelizing our test suite. Tests currently fail simplecov minimum coverage standards due to processes overriding results from other parallel test processes. A solution is in the works to properly merge simplecov results. 

The pact verification functionality is in working order for developer usage when a commit is pushed or when invoked via webhook from the pact broker during a contract content changed event. 

When the pact verification task runs in CircleCI (via the build or verification workflow), verification results are pushed back to the broker after the workflow completes.

### CircleCI and the Build Process

#### Deploy Strategy and Tagging
Every build of vets-website publishes pacts, tags the version with the name of the branch, and triggers the verification task from a master build of vets-api.


* Every build of vets-api verifies pacts tagged as `master` as well as [work-in-progress (WIP) pacts](https://docs.pact.io/pact_broker/advanced_topics/wip_pacts). If verification was successful, the build will the results with the name of the vets-api branch.
#### Frontend
If verification was successful, the CI pipeline will proceed to deploy. If it failed, the pipeline will stop the deploy. For a feature branch, a successful verification allows a PR to be merged while a failed verification blocks a PR from merging. If the CI job responsible for the verification task fails to publish the verification results, the can-i-deploy check would also fail, since it's looking for a passing verification. It would be possible to re-run the verification task job in Circle CI in the event that it fails.

#### Backend
The above steps apply to changes made from the FE. Assuming the pacts are already in place, the only relevant step for the BE CI pipeline is running the verification Rake task to ensure that any API changes don't break existing contracts in the appropriate environments.


### Pact FAQ

Additional questions, particularly around the design, architecture, and patterns in Pact, are answered in the [official FAQ](https://docs.pact.io/faq/).

Included below are some of the FAQs that are most relevant to us and our existing tools.

#### Can I generate my pact file from something like Swagger?
It’s generally not recommended to generate pact files from Swagger docs.
> The pact file is the artifact that keeps these two sets of tests in sync. To generate the pact file from anything other than the consumer tests (or to hand code it) would be to defeat the purpose of this type of contract testing. The reason the pact file exists is to ensure the tests in both projects are kept in sync - it is not an end in itself. Manually writing or generating a pact file from something like a Swagger document would be like marking your own exam, and would do nothing to ensure that the code in the consumer and provider are compatibile with each other.

#### Why doesn't Pact use JSON Schema?
> Whether you define a schema or not, you will still need a concrete example of the response to return from the mock server, and a concrete example of the request to replay against the provider. If you just used a schema, then the code would have to generate an example, and generated schemas are not very helpful when used in tests, nor do they give any readable, meaningful documentation. If you use a schema and an example, then you are duplicating effort. The schema can almost be implied from an example.
