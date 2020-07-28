--- 
title: "Developing Versionless APIs with GraphQL" 
date: "2020-07-22" 
authors:  
- github: xo426017 
  name: "Bing Xiao" 
categories: ["graphql", "API"] 
featuredImage: "/images/developing-versionless-graphql-apis/featured.png" 
---

GraphQL has recently made a huge splash in the API world by addressing some of the pain points with REST APIs.

With REST APIs, every resource needs an API endpoint. When we need to add a new resource, the endpoint often must be versioned, and/or a brand-new endpoint must be created. This leads to many maintainability and usability issues.

However, GraphQL operates from a single `/graphql` endpoint. The new resource can be added via new types, or fields, in the GraphQL schema without creating a new endpoint.

The GraphQL schema also provides interactive documentation. API consumers can use [GraphiQL](https://www.electronjs.org/apps/graphiql) or the [GraphQL Playground](https://github.com/prisma-labs/graphql-playground) to view the documentation for the query that they are building.

That said, designing an extensible schema is the most important part of building versionless APIs in GraphQL.  

## Schema design

There are a few things that can help when designing a schema.

First and foremost, grasp the domain knowledge, and try to understand the business domain as best you can. Then, follow a few schema design principles for GraphQL. For instance, it is preferable to use more complex structures like an object type than simple structures, like an array. This ensures that new data fields can be added easily, and more complex business objects can be composed without much effort.

Additionally, an Enum should be preferred to represent a field instead of a string. Using an Enum is especially helpful for API consumers who can easily be aware of the possible values for that Enum in the build-in online documentation.

## Using the GraphQL API

I had a delightful experience when working on queries in a properly designed StarWars GraphQL API.

For example, the query below tries to retrieve the name of the hero for a given episode. If the hero is a droid, the primary function of the droid is returned; if the hero is a human, the height of the human is returned.  

```graphql
query HeroForEpisode($ep: Episode!) {
  hero(episode: $ep) {
    name
    ... on Droid {
      primaryFunction
    }
    ... on Human {
      height
    }
  }
}
```

```subtext
graphql
```

In the example above, the episode is an Enum type. A hero is a character, which can be either `Human` type or `Droid` type. In the schema, both Human and Droid implements an interface `ICharacter`. Another object type could be added to implement the same interface.

In the two queries below, the first one is to add a review for an episode; the second one is to subscribe to the new review for an episode. The subscription query drastically reduces the code complexity in the use cases where the client application needs to subscribe to the server changes in real-time.

```graphql
mutation CreateReviewForEpisode($ep: Episode!, $review: ReviewInput!) {
  createReview(episode: $ep, review: $review) {
    stars
    commentary
  }
}

subscription {
  onReview(episode: NEWHOPE) {
    stars
    commentary
  }
}
```

```subtext
graphql
```

The query below allows the clients to retrieve the heroes for three different episodes in one request. This is one of the big advantages of GraphQL. It allows the clients to retrieve many resources in a single request, which can improve the performance substantially by reducing the number of API calls. This is especially important for mobile apps.

```graphql
query Heros {
  empireHero: hero(episode: EMPIRE) {
    name
  }

  jediHero: hero(episode: JEDI) {
    name
  }

  newhopeHero: hero(episode: NEWHOPE) {
    name
  }
}
```

```subtext
graphql
```

## Summary

GraphQL provides a complete and understandable description of the data in your API. It makes it easier to evolve APIs over time and enables powerful developer tools.  

It is worth mentioning that quite a few high profile companies, such as GitHub, airbnb, intuit, and PayPal have switched to GraphQL for their APIs. GraphQL has powered Facebook's mobile apps since 2012!

Writing your own GraphQL API is a bit harder than writing a REST API, but in a lot of cases, it is well worth it.

Thanks to [Josh Searles](https://github.com/jrsearles) for helping with this post.