---
layout: post
title:  "AWS AppSync Authorization with a Second DynamoDB Table"
author: "interrobrian"
twitter: "interrobrian"
date:   2018-08-06 11:00:00 -0600
tags: [aws, appsync, graphql]
---

AWS AppSync is a powerful platform for making an offline-enabled backend for your app or website. However, for certain scenarios, I've had to change my thinking a bit.

## Simple ID-based Auth

Let's take a look at an authorization scenario. Suppose we have a collection of Project objects, and we want to make it so that when a project is added, it automatically puts the current user ID on the project as its owner. Using the [Authorization Use Cases](https://docs.aws.amazon.com/appsync/latest/devguide/security-authorization-use-cases.html) guide from Amazon, this is relatively simple:

### schema.graphql
{% highlight graphql %}
type Project {
  id: ID!
  name: String
  createdBy: String
}

type Query {
  getProject(id: ID)
}

type Mutation {
  createProject(name: String)
}

schema {
  query: Query
  mutation: Mutation
}
{% endhighlight %}

### createProject Request Resolver
{% highlight vtl %}
{
  "version" : "2017-02-28",
  "operation" : "PutItem",
  "key" : {
    "id": $util.dynamodb.toDynamoDBJson($util.autoId()),
  },
  "attributeValues": {
    "name": $util.dynamodb.toDynamoDBJson($context.args.name),
    "createdBy": $util.dynamodb.toDynamoDBJson($context.identity.sub),
  },
}
{% endhighlight %}

The core problem we've solved here is adding identity context to our DynamoDB resolver using the provided [$context.identity object](https://docs.aws.amazon.com/appsync/latest/devguide/resolver-context-reference.html#aws-appsync-resolver-context-reference-identity). We can also extend this to other requests pretty trivially, such as a get query:

### getProject Response Resolver
{% highlight vtl %}
#if($context.result["createdBy"] == $context.identity.sub)
    $utils.toJson($context.result);
#else
    $utils.unauthorized()
#end
{% endhighlight %}

## Auth with a Second Data Source

Using the `$context.identity` object works great and all- until you need user data that comes from one of your data sources and not the identity provider. Suppose we extend the previous example so that Projects belong to groups rather than users:

### schema.graphql
{% highlight graphql %}
type Group {
  id: ID!
  Users: [User]
}

type User {
  id: ID!
  group: Group
}

type Project {
  id: ID!
  name: String
  createdBy: ID
  groupId: ID
}

type Query {
  getProject(id: ID)
}

type Mutation {
  createProject(name: String)
}

schema {
  query: Query
  mutation: Mutation
}
{% endhighlight %}

Making this change presents a new issue - how do we know the current user's group ID within the context of `createProject`? We can't possibly get this information without cross-referencing the User collection, and AppSync Data Resolvers can only attach to one Data Source. We could use a Lambda Function to solve this, but that seems like an awful lot of work just to get the current user's Group.

To solve this, we can tweak our Mutation structure a bit:

### schema.graphql
{% highlight graphql %}
...

type UserMutation {
  createProject(name: String)
}

type Mutation {
  currentUser: UserMutation
}

...
{% endhighlight %}

This allows us to attach some additional context to the `createProject` mutation, thanks to the [$context.source object](https://docs.aws.amazon.com/appsync/latest/devguide/resolver-context-reference.html) in the resolver, which contains the result of the parent resolver:

### currentUser Request Resolver
{% highlight vtl %}
{
  "version": "2017-02-28",
  "operation": "GetItem",
  "key": {
    "id": $util.dynamodb.toDynamoDBJson($context.identity.sub),
  },
}
{% endhighlight %}

### createProject Request Resolver
{% highlight vtl %}
{
  "version" : "2017-02-28",
  "operation" : "PutItem",
  "key" : {
    "id": $util.dynamodb.toDynamoDBJson($util.autoId()),
  },
  "attributeValues": {
    "name": $util.dynamodb.toDynamoDBJson($context.args.name),
    "createdBy": $util.dynamodb.toDynamoDBJson($context.identity.sub"),
    "groupId": $util.dynamodb.toDynamoDBJson("$context.source.groupId"),
  },
}
{% endhighlight %}

Note how the value of `groupId` uses `$context.source.groupId` to get the `groupId` from the `User` object returned by `currentUser` (this is making an assumption that your DynamoDB Table uses a property called `groupId` internally to link a User to a Group).

In essence, you can think of the root `Mutation` type as being the "global" or "identity" context. If you need additional context for a mutation, you'll need to provide it - in our case, we've created a "current user" context, which contains additional user data.

As with the simple example, you can apply the same technique to queries:

### schema.graphql
{% highlight graphql %}
...

type User {
  id: ID!
  group: Group
  getProject(id: ID)
}

type Query {
  currentUser: User
}

...
{% endhighlight %}

### getProject Response Resolver
{% highlight vtl %}
#if($context.result["groupId"] == $context.source.groupId)
    $utils.toJson($context.result);
#else
    $utils.unauthorized()
#end
{% endhighlight %}

Hope this solution saves you some time on keeping your AppSync GraphQL APIs secure!
