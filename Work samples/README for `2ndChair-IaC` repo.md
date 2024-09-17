# At a glance

## Major components in our stack

- SvelteKit
- AWS AppSync (GraphQL)
- DynamoDB

# Detailed walkthrough

## Website: SvelteKit on Vercel

We’ve built our website as a SvelteKit application. It deploys to Vercel.

We chose SvelteKit because its great developer experience reduces the cognitive load of frontend engineering.

We chose Vercel because its fully managed, serverless, auto-scaling nature reduces or eliminates the cognitive load of hosting and deploying a website.

## Auth: Auth.js

Our SvelteKit app uses the Auth.js library to create accounts and handle sign-in and sign-out actions.

We chose Auth.js because it’s free and open-source, and coding a homebrew solution for auth is strongly discouraged. Other out-of-the-box solutions carry a price tag.

We use only the “magic links” feature of Auth.js. Auth.js can support many OAuth providers (e.g. “sign in with Google”) out of the box, but we’ve decided not to incorporate any of those. This decision has simplified many aspects of our product design and engineering.

## Database: DynamoDB

Our database is DynamoDB.

We use a single-table design. This is a special kind of database schema unique to the NoSQL world. In practical terms, it means we give up having an abstract query language in order to gain benefits to cost, performance, and scalability. Each kind of database task (e.g. “fetch all messages in this chat“) requires a unique, specific sequence of instructions to the database, and the schema must be designed with every desired task in mind, to ensure those tasks can be achieved by coherent and efficient instructions. The schema and query plan must be documented so programmers will know how to perform the tasks they want. (Our docs are [here](https://docs.google.com/spreadsheets/d/1aDXAeuXisIPXqWs4dnzbgAaZNGYnFWXaVjeCuNN5ctc/).)

We chose DynamoDB with a single-table design because this solution auto-scales seamlessly with virtually zero effort or attention, reducing the cognitive load of backend DevOps. SQL databases, while easier to query, impose a higher DevOps burden because they were not designed for a serverless world. Learning how to work with NoSQL single-table design seems like the preferable pain compared to the constant mental load of DevOps. Also, DynamoDB has first-class integrations with both Auth.js (mentioned above) and AWS AppSync (mentioned later), as well as every other AWS service.

Auth.js interacts directly with DynamoDB when performing login-related actions.

### Blog table for marketing website

Separately from our app's data table, we have a small DynamoDB table that stores blog articles for our marketing website. It stores two kinds of record: a `CARD` type with metadata only (for efficiently fetching the list of articles), and a `PAGE` type with all metadata plus the article’s text content. To fetch the list of article cards, perform a DynamoDB “Query” action where the partition key is equal to “CARD” and the sort key is left unspecified. (Data can be sorted and filtered by the client afterward.) To fetch a particular article’s body text, perform a DynamoDB “GetItem” action where the partition key is “PAGE” and the sort key is the article’s URL slug.
If sorted or time-range-based access becomes a key product goal, we can add a GSI (global secondary index) to the table to enable this access pattern. This GSI might, for example, use a compound primary key based on a new attribute called yearMonth and an existing attribute called timestamp.

## Backend APIs: AppSync, GraphQL, real-time data

For all backend needs other than login-related-actions, our SvelteKit app talks to a GraphQL API hosted on AWS AppSync.

Our GraphQL API uses DynamoDB (the same single table mentioned above) as its data source.

At any time, our GraphQL API can talk to as many data sources as it wants. Currently, it needs only our DynamoDB table.

### Why AppSync?

We’re not using AWS AppSync because we need GraphQL; we’re using GraphQL because we need AWS AppSync.

We want to leverage one specific feature of AWS AppSync: real-time data subscriptions.

If you carefully set up a complete backend in AWS AppSync, then you can use the “AWS Amplify” code library in your Web app to listen for real-time updates to data of interest, with a vastly reduced amount of frontend engineering. This functionality is otherwise very hard and cumbersome to implement. A fully automatic, out-of-the-box solution for real-time data updates on the frontend is offered only by Amazon, and only through AppSync specifically.

By building around AppSync, we can support collaborative multiplayer features (and any other kind of real-time update) at the lowest possible effort cost. Sean’s research suggested that every alternative strategy for adding this functionality, either now or in the future, would impose a significantly higher effort cost, and the required architectural migrations would likely be very painful if the product were not built with this capability initially.

### GraphQL resolvers: APPSYNC_JS

Building a GraphQL solution always involves writing “resolvers”: small pieces of executable code that move requested data to/from a data source. In a GraphQL schema, each individual primitive (field, query, mutation, or subscription) can potentially have its own resolver. A single client request might be fulfilled by executing multiple resolvers and combining their results; the client has no knowledge of this, and sees only the returned data.

The GraphQL spec doesn’t give details about how resolver code should be implemented, stored, or run. It’s up to other players in the ecosystem to create a layer that meets these needs. Various options exist, mostly in the form of dedicated GraphQL servers you must provision and manage.

AWS AppSync takes a serverless approach. It provides a full-service solution for GraphQL, including resolvers. When building with AppSync, you write your resolvers in JavaScript. AppSync hosts your resolver functions in the cloud, right alongside your GraphQL schema. The AppSync platform listens for requests to your GraphQL API, then executes your resolvers on demand, using a special JavaScript runtime that Amazon built. That runtime is called APPSYNC_JS. It has a stripped-down feature set compared to Node or the browser environment. Before writing resolver functions, check [the docs](https://docs.aws.amazon.com/appsync/latest/devguide/supported-features.html) to learn what APPSYNC_JS can and can’t do.

## SAM / CloudFormation / IaC

We use an “Infrastructure as Code” (IaC) solution to unlock needed development capabilities for our AppSync configuration and its associated GraphQL schema and JavaScript resolvers.

These capabilities include version control, repeatable deployment, rollbacks, multi-author collaboration, pull requests, and more.

The solution we use is AWS SAM.

AWS SAM is a set of feature extensions added to AWS CloudFormation.

### CloudFormation and SAM

AWS CloudFormation is a system that allows you to declare your AWS resources in a YAML file, then feed that YAML file to Amazon’s servers, which will set up the declared resources in your AWS account on your behalf. By using this system or another one like it, you can graduate from the chore of setting up your backend by clicking on buttons in the AWS console, which is an untenable malpractice. Amazon knows this and puts little effort into making the console a good developer experience, further pushing customers toward IaC, which customers already expect to use anyway because it’s an industry-standard practice.

Your YAML file is called a “CloudFormation template”.

The CloudFormation template spec offers a syntax item to represent every possible kind of Amazon resource. That includes low-level AWS entities that don’t appear in our mental model of our product but are nonetheless required by AWS, like “IAM roles”. Our deployment must include these. Therefore, working with CloudFormation is kind of a pain, since it fails to free us from getting bogged down in nitty-gritty AWS details.

This is why AWS SAM was created. SAM stands for “Serverless Application Model”. The SAM template spec extends the basic CloudFormation template spec, offering some new syntax items at a higher level of abstraction that more closely matches how we think about our product. These SAM-specific items will automatically create and connect multiple lower-level items on your behalf, resulting in an “it just works” experience for mainstream use cases in the serverless world. So far, we are one of those use cases.

### Resolver pipelines

When you use AWS SAM to declare an AWS AppSync GraphQL API, you also declare all of the supplementary code assets that your API will use. That includes GraphQL resolvers, and also individual code files called “Functions” that you compose together to constitute the resolvers’ behavior. SAM exclusively supports the “pipeline resolver” architecture, which is the name for this pattern. (You may also hear about “unit resolvers”, which are not used in our project, because SAM does not support them.)

Every pipeline resolver is a chain of actions that begins with a pre-hook, then runs one or more “Functions”, then ends with a post-hook. Somewhere in the middle, data will travel to/from any number of remote data sources; at the end, all the fetched data will be massaged into the shape promised by your GraphQL schema for this particular client request.

#### Pipeline Inner “Functions”

Each capital-F “Function” is a standalone JavaScript code file.

In terms of its runtime behavior, each “Function” begins with its own inner pre-hook, then optionally sends a request to a remote data source (such as DynamoDB) and awaits a reply, then ends with its own inner post-hook.

A “Function’s” inner pre-hook, which is called `request()`, must return an instruction for the associated data source. Alternatively, the pre-hook can perform an “early return”, which skips the data source, mocks an output, and moves ahead to the next “Function”. This can be done conditionally according to whatever logic you want.

A “Function’s” inner post-hook, which is called `response()`, may inspect and modify the output returned from the data source. Whatever is returned from `response()` becomes an input to the next “Function” in the chain.

Either one of these hooks may raise an error, which can halt the pipeline or not, depending on how you write the code.

#### Pipeline Outer Prelude and Coda

The outer pre-hook and post-hook of the whole resolver also live in their own standalone JavaScript code file, which (again) contains two JavaScript code-functions. Confusingly, those code-functions are also named `request()` and `response()`, even though their role is different from the identically named code-functions inside of the “Function” files.

If you find the nomenclature here troubling, you’re not alone.

The pipeline’s outer `request()` code-function usually doesn’t do much, but it can short-circuit the entire pipeline using the “early return” feature or the “error” feature, and it can package input arguments so the “Functions” can find them.

The pipeline’s outer `response()` code-function must emit a JSON object that contains every field requested by the client. If the output of the last “Function” pipeline step happens to meet the requirements, then `response()` can pass it along unmodified.

#### Context object

When you write code for AppSync GraphQL resolver pipelines, your work will focus heavily on manipulating a JavaScript object called `ctx`. This object is provided by AppSync. It shuttles various important data through your pipeline.

Here are some of its features:

- Viewing the arguments given in the client’s request
- Inspecting the result of the previous pipeline step
- Passing arbitrary data to a future pipeline step

You can learn more about the context object by reading [these docs](https://docs.aws.amazon.com/appsync/latest/devguide/resolver-context-reference-js.html).

#### Sending instructions to DynamoDB

When you write a `request()` code-function for an inner “Function” step of a pipeline resolver, and you’re targeting a DynamoDB data source, you must return a JSON object that specifies the command you want DynamoDB to run.

Determining the requirements for this JSON object can be challenging for multiple reasons, described as follows. Amazon provides multiple helper libraries and supports multiple execution strategies. These all appear in different contexts. They all have separate documentation. The documentation assumes prior knowledge and is spread across multiple locations, which are confusingly organized—scattered in surprising ways and nested in surprising ways. Lastly, code examples you find across the Web will randomly use one specific approach without acknowledging the existence of the others, and your use case may or may not be compatible with that approach. I would not wish this ramp-up experience on my own enemies.

Below is Sean’s best effort at providing a map for beginners:

1. Your returned JSON object must conform to an object structure defined on [this page](https://docs.aws.amazon.com/appsync/latest/devguide/js-resolver-reference-dynamodb.html).
1. You’re free to code that object structure by hand, if you like, but you have to format the data values [a certain way](https://docs.aws.amazon.com/appsync/latest/devguide/js-resolver-reference-dynamodb.html#js-aws-appsync-resolver-reference-dynamodb-typed-values-request) in order for DynamoDB to accept them.
1. If formatting data values by hand is getting you down, AWS offers [helper functions](https://docs.aws.amazon.com/appsync/latest/devguide/dynamodb-helpers-in-util-dynamodb-js.html) that gobble up programmer-friendly JSON and spit out DynamoDB-friendly JSON. When you're using utility functions at this level, you'll probably reach for `util.dynamodb.toMapValues()` in most situations.
1. For some use cases, you can sidestep the formatting responsibility entirely by calling a method from a [more abstract method set](https://docs.aws.amazon.com/appsync/latest/devguide/built-in-modules-js.html). However, this method set only covers _some_ of DynamoDB’s capabilities; you will need to fall back to a lower-level approach if you need more customization than these methods can support, or if you need to run a DynamoDB command that isn’t represented by one of these methods.
1. The exact details and behaviors of the underlying DynamoDB system can be gleaned from [DynamoDB’s own docs](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_Operations_Amazon_DynamoDB.html), which exhaustively cover every requirement and capability. Unfortunately, these docs are written with the assumption that you’re talking to DynamoDB over the Internet via direct HTTP requests, not via JSON emissions from your AppSync resolver `request()` methods linked to a “DynamoDB data source”. As a result, the DynamoDB docs are actually incompatible with the I/O requirements of your context, which can be hard to tell at a glance. If you find yourself in this doc layer, you should probably back out.
1. HOWEVER, there may come a time (someday, somehow) when you want to do something really fancy that requires you to send a raw HTTP request from AppSync to DynamoDB, linking the latter to the former as an opaque “HTTP data source”. In that case, you would get access to the full feature set of DynamoDB, and you would also have to use the I/O format described in the raw DynamoDB docs. I hope that day never comes, but if it does, we’ll be ready.

### The SAM CLI

The AWS SAM product offering includes not only a template spec, but also a handy CLI you can use to feed your SAM templates to Amazon’s servers.

The core workflow is: hack on your template, then run `sam build`, then run `sam deploy`, then follow the on-screen prompts.

Once `sam deploy` reports success, your new backend is online and ready to receive requests.

Before you can perform this workflow for the first time, you must configure an AWS CLI "profile" on your local machine that contains the API key and secret key of our AWS resources. These keys **MUST NEVER BE COMMITTED TO VERSION CONTROL, EVER**. Keep them safe!

#### Debugging opaque error messages from SAM

If `sam deploy` gives you an opaque error message telling you there is something wrong with your resolver code, you can use the `evaluate-code` CLI tool to generate a more helpful error message. You must invoke the affected pipeline.

An example invocation is given here:

```
aws appsync evaluate-code --runtime "name=APPSYNC_JS,runtimeVersion=1.0.0" --context "{}" --profile 2nd_Chair_SAM_CLI --function "request" --code "$(cat ~/Documents/Dev/2nd\ Chair/2ndChair-IaC/2ndChairSAMApp/graphql/resolvers/pipelineSteps/TransferOrgOwnership.js)"
```

Replace the `--profile 2nd_Chair_SAM_CLI` portion, and the argument to `cat`, with your own configuration as appropriate.
