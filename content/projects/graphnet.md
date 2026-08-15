---
title: "GraphNet"
tagline: "A social feed over GraphQL"
category: "Web Dev"
status: "archived"
type: "web-app"
description: "Posts, comments and likes behind an Apollo GraphQL API, with a React client. Built to learn GraphQL properly."
language: "JavaScript"
tech:
  - "Apollo Server"
  - "GraphQL"
  - "MongoDB"
  - "React"
  - "JWT"
github: "https://github.com/AbuCTF/GraphNet"
images:
  - "/images/projects/graphnet/atlas.jpg"
draft: false
---

Uniting communities with GraphQL-powered social networking, which in practice means posts, comments, likes and auth, built to learn GraphQL properly rather than to launch anything.

## The schema

Seven mutations, two queries, one subscription. That's the whole surface.

```graphql
type Mutation {
    register(registerInput: RegisterInput): User!
    login(username: String!, password: String!): User!
    createPost(body: String!): Post!
    deletePost(postId: ID!): String!
    createComment(postId: String!, body: String!): Post!
    deleteComment(postId: ID!, commentId: ID!): Post!
    likePost(postId: ID!): Post!
}
```

`likePost` is a single toggle rather than a like/unlike pair. It checks whether you're already in the post's `likes` array and adds or removes you. Fewer mutations, no way to get the two out of sync.

`likeCount` and `commentCount` aren't stored. They're resolvers over array length, computed per request:

```javascript
Post: {
    likeCount: (parent) => parent.likes.length,
    commentCount: (parent) => parent.comments.length,
},
```

Nothing to denormalise, nothing to drift.

## Auth

Register hashes with bcryptjs at 12 rounds, login compares and signs a JWT with a one-hour expiry. The client keeps it in localStorage and attaches it through an Apollo context link as a bearer token.

The client also decodes the token on load and throws it away if `exp` has passed, so a stale tab doesn't spend a request finding out it's logged out.

Deletes check ownership by comparing usernames: `deletePost` and `deleteComment` both refuse unless the caller's username matches the record's.

## Subscriptions

`newPost` publishes to a `NEW_POST` channel on every successful `createPost`. It's the piece that made GraphQL worth the trouble on this project: the feed updates without polling and without me writing any socket code.

## What's broken

Being honest about this one, since it's archived.

`config.js` holds the Mongo URI and the JWT secret, it's gitignored, and three files import it. A fresh clone will not start until you write it yourself.

The models are `models/user.js` and `models/post.js` on disk but imported as `../../models/User` and `../../models/Post`, which works on macOS and fails on Linux.

And `package.json` pins react-router-dom v6 while the routes still use the v5 API: `exact`, `component`, `render` props. `AuthRoute` imports `Navigate` from v6 and then uses it inside a v5 render prop. It needs a router migration, not a patch.
