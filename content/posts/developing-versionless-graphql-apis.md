---
title: "Developing versionless APIs with GraphQL"
date: "2020-07-22"
authors: 
- github: xo426017
  name: "Bing Xiao"
categories: ["graphql", "API"]
featuredImage: "/images/developing-versionless-graphql-apis/featured.png"
---

GraphQL has recently made a huge splash on API world. It certainly helps address some of pain points in Rest API. In Rest API, basically every resource/action needs an API endpoint. When we need to add a new resource, the endpoint often must be versioned, and/or a brand-new endpoint must be created. This leads to huge maintainability and usability issues. 

To the contrary, GraphQL operates from a single /graphql endpoint. The new resource can be added via new types/fields in GraphQL schema. The schema also enables interactive documentation. The API consumers can use GraphiQL or Playground to get documentation for the query they are building. Therefore, designing an extensible schema is the most important part in building versionless APIs in GraphQL. 

There are a few things that can help when designing a schema. First and foremost, grasp the domain knowledge, try to understand the business domain as best as you can. Then, follow a few schema design principles for GraphQL. For instance, it is preferable to use more complex structures like an object type than simpler structures like an array. This enables that the new data fields can be added easily, and more complex business objects can be composed easily. Also whenever we could, use an Enum to represent a field, instead of a string. Using an Enum is especially helpful for API consumers, who can easily be aware the possible values for that Enum in the build-in online documentation.

I have a delightful experience when working on the queries in a properly designed StarWars GraphQL API. The query below tries to retrieve the name of the hero for a given episode, and if the hero is a droid, the primary function of the droid is returned; if the hero is a human, the height of the human is returned. In this example, the episode is an Enum type. The IntelliSense in Playground shows the currently available episodes in the system. A hero is a character, which can be either Human type or Droid type. In the schema, both Human and Droid implements an interface ICharacter. Another object type can be easily added to implement the same interface.

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

In two queries below, the first one is to add a review for an episode; the second one is to subscribe to the new review for an episode. The subscription query drastically reduces the code complexity in the use cases, in which the client application wants to subscribe to the server changes in real-time.

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

The query below allows the clients to retrieve the heroes for three different episodes in one request. This is one of the big advantages in GraphQL. It allows the clients to retrieve many resources in a single request, which can improve the performance substantially by reducing the number of API calls. This is especially important for mobile apps.

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

In summary, GraphQL provides a complete and understandable description of the data in your API. It makes it easier to evolve APIs over time and enables powerful developer tools. Writing your own GraphQL API is a bit harder than writing a REST API. In a lot cases, it is well worth it. Facebook’s mobile apps have been powered by GraphQL since 2012. Quite a few high profile companies, such as GitHub, airbnb, intuit, PayPal, etc., have switched to GraphQL for their APIs.

Thanks to [Josh Searles](https://github.com/jrsearles) for helping with this post.