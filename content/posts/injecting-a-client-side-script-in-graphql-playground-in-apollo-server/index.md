---
title: "Injecting a client-side script in GraphQL Playground in Apollo Server"
subtitle: "Just a hack."
date: 2018-12-22T12:00:00+09:00
tags: ["apollo", "graphql", "playground", "express"]
---

This is a short article to explain how I inject a client-side script in Playground. Note that this is a workaround solution until I find a better way.

### Background

When developing a GraphQL server with Apollo Server, GraphQL Playground is a tool that you are likely to use. Now, suppose we have an Apollo server that authenticates with a token in an HTTP header. Luckily, Playground has an editor to specify HTTP headers. However, what if we need to call OAuth provider to get an accessToken? Surely, we could run `curl` to get a token, copy it in the terminal and paste it in the browser. It should be much easier if we could simply sign in in the browser.

----

A quick look into the Apollo Server code and the Playground code, there seems no configuration to inject a script that runs in the browser. While not being sure if there's a recommended way for this, I try to patch it.

### The example code

Here is the code based on the Apollo Server tutorial. It simply adds a string in HTML.

```javascript
const { ApolloServer, gql } = require('apollo-server-express');
const express = require('express');
const interceptor = require('express-interceptor');

const typeDefs = gql`
  type Query {
    "A simple type for getting started!"
    hello: String
  }
`;

const resolvers = {
  Query: {
    hello: () => 'world',
  },
};

const server = new ApolloServer({
  typeDefs,
  resolvers,
});

const app = express();
const port = process.env.PORT || 3000;

const SCRIPT_TO_INJECT = `
  <script>
    window.addEventListener('load', () => {
      console.log('script injected');
      const store = window.s;
      setTimeout(() => {
        const headers = '{"Authorization": "Bearer token"}';
        store.dispatch({ type: 'EDIT_HEADERS', payload: { headers } });
      }, 2000);
    });
  </script>
`;
const END_BODY = '\n  </body>\n  </html>';
app.use(interceptor((req, res) => ({
  isInterceptable: () => /text\/html/.test(res.get('content-type')),
  intercept: (body, send) => {
    send(body.replace(END_BODY, SCRIPT_TO_INJECT + END_BODY));
  },
})));

server.applyMiddleware({ app });

app.get('/', (req, res) => {
  res.redirect(server.graphqlPath);
});

app.listen(port, () => {
  console.log(`Open http://localhost:${port}${server.graphqlPath}`);
});
```

What is nice here is that Playground exposes the Redux store as `window.s`. Notice the string replacement is not robust and if Playground changes the spaces in HTML, it won't work.

### How to run

You can check out the GitHub repository and run this example.

https://github.com/dai-shi/apollo-server-patch-playground-example

You can also run it in codesandbox.

[apollo-server-patch-playground-example - CodeSandbox](https://codesandbox.io/s/github/dai-shi/apollo-server-patch-playground-example/tree/master/?module=%2Fsrc%2Fserver.js)

(Until now, I didn't realize that CodeSandbox is capable to run Apollo Server. How nice!)

### Final notes

As I noted, this is just a workaround at the point of writing. I'm pretty sure this is not what we want and would like to look for a recommended solution.
