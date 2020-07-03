# ⚡ Using GraphQL with React

## Quick notes on how to use graphql data in react.

### 📌 Graphql endpoint and server options

First and foremost, to use graphql data in react, we need a graphql endpoint. We can use any of the free online enpoints like [SWAPI](https://swapi.graph.cool/) or spin-up our own local endpoint using \``apollo server`\`

Check this [repo](https://github.com/jvikraman/apollo-server-ref) for a quick reference on setting up a local apollo server for our graphql needs.

---

## 📌 Graphql client options

### ✨ Basic client

We could do a simple non-react client using basic http request (or) head down the react path. Below is an example of a non-react client:

```javascript
// index.js file
const endpoint = `http://localhost:4000`;

const query = `
  {
    hello
  }
`;

const options = {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ query })
};

fetch(endpoint, options)
  .then(response => response.json())
  .then(({ data }) => console.log(data))
  .catch(console.error);
```

To simplify the code a bit, we can replace the `fetch` with `request` method from `graphql-request npm` package. Below is an example:

```javascript
import { request } from 'graphql-request';
...
...

request(endpoint, query)
  .then(({ data }) => console.log(data))
  .catch(console.error);
```

---

### ✨ Apollo client

The `apollo-boost` npm package provides a convenient zero-config way to use the apollo client in our applications. Start by installing the following npm packages:

```sh
npm install graphql apollo-boost
```

Then create an apollo client like so:

```javascript
import ApolloClient, { gql } from "apollo-boost";

// Specify the graphql endpoint to connect to.
const uri = `http://localhost:4000`;

// Then create the new apollo client with the endpoint.
const client = new ApolloClient({ uri });

const query = gql`
  query {
    hello
  }
`;

// Execute the query
client
  .query({ query })
  .then(({ data }) => console.log(data))
  .catch(console.error);
```

---

### ⚛ React apollo client

Next up, here's how to use a react apollo client.
Spin up a react starter project using `create-react-app` or your own setup and install the following dependencies:

```sh
npm install graphql apollo-boost react-apollo
```

From there onwards, there are several ways to consume graphql data in react:

### 1️⃣ Using `ApolloProvider` and `ApolloConsumer` from `react-apollo` package along with render-prop technique.

#### Example:

```javascript
import React, { Component } from 'react';
import ApolloClient from "apollo-boost";
import { ApolloProvider, ApolloConsumer } from "react-apollo";
import gql from "graphql-tag";

// Create a new apollo client using a specific endpoint
const client = new ApolloClient({
    uri: `https://swapi.graph.cool/`;
});

// App component using context feature along-with render-prop
// technique to execute the query & display the results.
class App extends Component {
  render() {
    return (
      <ApolloProvider client={client}>
        <ApolloConsumer>
          {client => {
            client
              .query({
                query: gql`
                  {
                    allFilms {
                      title
                      releaseDate
                    }
                  }
                `
              })
              .then(res => console.log(res));
            return null;
          }}
        </ApolloConsumer>
      </ApolloProvider>
    );
  }
}

export default App;
```

### 2️⃣ Using `<Query />` component from `react-apollo` package.

We can also use the `<Query />` component from `react-apollo` to fetch the data instead of `<ApolloConsumer />`. Using the previous example as a base, here's a modified version that retrieves a list of films from a graphql endpoint and renders them in the UI:

```javascript
import { ApolloProvider, Query } from 'react-apollo';
import { gql } from "apollo-boost";

const client = ...

class App extends Component {
  render() {
    return (
      <ApolloProvider client={client}>
        <Query
          query={gql`
            {
              allFilms {
                title
                releaseDate
              }
            }
          `}
        >
        {({ data, loading, error }) => {
          if (loading) return <p>Loading...</p>;

          if (error) return <p>Error: {error.message}</p>;

          if (data.allFilms === undefined) return null;

          return (
            {data.allFilms.map(({ title, releaseDate }) => (
              <li key={releaseDate}>title</li>
            ))}
          )
        }}
        </Query>
      </ApolloProvider>
    )
  }
}
```

### 💡Passing dynamic arguments using query variables

If we need to pass arguments dynamically to an apollo query, we can do it like the following:

```javascript
// Films.js file

...

// $director -> dynamic input variable

// String! -> indicates the input variable is mandatory and of type String

// filter: { ... } -> filter object to further refine the search query

const filmsQuery = gql`
  query films($director: String!) {
    allFilms(filter: {
      director_contains: $director
    }) {
      title
      releaseDate
    }
  }
`;

// variables -> a way to provide dynamic input to the graphql query

export default class extends Component {
  render() {
    return (
      <Query
        query={filmsQuery}
        variables={{ director: "George" }}
      >
      ...
      </Query>
    )
  }
}
```

---

### 📌 Using Mutations

Let's say our apollo server/graphql endpoint provides a couple `mutations` - one to increment a number and another to decrement. Here's how we can use them in our react component:

```javascript
import { Query, Mutation } from "react-apollo";
import { gql } from "apollo-boost";

// Queries
const TOTAL_COUNT_QUERY = gql`
  query count {
    count
  }
`;

// Mutations
const INCREMENT_MUTATION = gql`
  mutation inc {
    increment
  }
`;

const DECREMENT_MUTATION = gql`
  mutation dec {
    decrement
  }
`;

// In the app, use the <Mutation /> component to execute the mutations on the server & return the data.
const App = () => (
  <div>
    <Query query={TOTAL_COUNT_QUERY}>
      {({ data, loading, error }) => {
        if (loading) return <p>Loading...</p>;
        if (error) return <p>Error: {error.message}</p>;
        if (data === undefined) return null;

        return <p>Count: {data.count}</p>;
      }}
    </Query>
    <Mutation
      mutation={INCREMENT_MUTATION}
      refetchQueries={[{ query: TOTAL_COUNT_QUERY }]}
    >
      {increment => <button onClick={increment}>+</button>}
    </Mutation>
    <Mutation
      mutation={DECREMENT_MUTATION}
      refetchQueries={[{ query: TOTAL_COUNT_QUERY }]}
    >
      {decrement => <button onClick={decrement}>-</button>}
    </Mutation>
  </div>
);
```

### 💡Performance Optimization - Using local apollo cache for query management

With the `refetchQueries` property, there's an extra network request involved to keep the UI in sync with the endpoint data. We can avoid this extra request by opting in to write the query result to our local apollo cache. Using the above example as the base, below are the specific parts:

```javascript
...
// Local cache updater function
const updateLocalCount = (cache, { data }) => {
  const count = data.increment || data.decrement;
  cache.writeQuery({
    query: TOTAL_COUNT_QUERY,
    data: count
  });
};

// Inside our <App /> component
...
<Mutation
  update={updateLocalCount}
  mutation={INCREMENT_MUTATION}>
    {increment => (
      <button
        onClick={increment}
      >
        +
      </button>
    )}
</Mutation>
```

You may notice in the example above that `refetchQueries` prop has been replaced with a local cache updater function via `update` prop that receives the local apollo cache and the mutation result.

The cache updater function updates the query result locally via `cache.writeQuery()` method

### 📌 Returning objects and lists via Mutations

Mutations also support more complex data types like lists, arrays, objects etc.

Let's say our apollo server/graphql endpoint provides a mutation to accept a new product order and returns some information about the operation (for e.g. total order count, list of current orders etc.), here's how we can use such a mutation in our react component:

Let's start with the apollo server/graphql endpoint specific details. Our server/endpoint exposes the following queries:

1. `totalOrders` -> returns the total order count as an `Int` graphql type
2. `allOrders` -> returns the current list of orders as an `Order` graphql type

```javascript
// Graphql query to retrieve total order count
// and list of current orders.
query total {
  totalOrders
  allOrders {
    id
    date
    product
    status
  }
}

// server response
{
  "data": {
    "totalOrders": 3,
    "allOrders": [
      {
        "id": "ord-123",
        "date": "2020-03-23",
        "product": "Mountain Bike",
        "status": "PENDING"
      },
      {
        "id": "ord-456",
        "date": "2020-04-08",
        "product": "Desktop Monitor",
        "status": "COMPLETE"
      },
      {
        "id": "ord-789",
        "date": "2020-01-12",
        "product": "Charcoal Grill",
        "status": "PROCESSING"
      }
    ]
  }
}

```

Similarly, our server exposes the following mutations:

1. `addOrder` -> accepts a graphql input type named `AddOrderInput` with specific fields about a new order, adds it to the existing list of orders and returns the newly added order
2. `removeOrder` -> accepts a non-nullable `id` field of type ID containing the order id, removes it from the list of orders and returns details about the operation (for e.g. before and after order totals, removed order etc.)

```javascript
// server.js file

// `addOrder` mutation on the server
const typeDefs = `
  scalar Date

  type Order {
    id: ID!
    date: Date!
    product: String!
    status: Status
  }

  input AddOrderInput {
    date: Date!
    product: String!
    status: Status
  }

  enum Status {
    PROCESSING
    PENDING
    COMPLETED
  }

  type Mutation {
    addOrder(input: AddOrderInput!): Order
  }
`;

const resolvers = {
  Query: {...},

  Mutation: {
    addOrder: (parent, { input: { date, product, status }}) => {
      ...
    }
  }
};

// server response for the mutation
{
  "data": {
    "addOrder": {
      "id": "_t65M0uMM",
      "date": "2020-03-02",
      "product": "Trailer Hitch",
      "status": "PENDING"
    }
  }
}
```

### 💪 Below is a full-fledged example of using the mutation in a react component including retrieving and displaying the current list of orders in the UI, plus adding a new order via the `AddOrder` button:

```javascript
// Queries.js file

// Total orders query
export const TOTAL_ORDERS_QUERY = gql`
  query {
    totalOrders
    allOrders {
      id
      date
      product
      status
    }
  }
`;

// Total orders including the status enum values
export const TOTAL_ORDERS_WITH_STATUS_QUERY = gql`
  query {
    totalOrders
    allOrders {
      id
      date
      product
      status
    }
    status: __type(name: "Status") {
      enumValues {
        name
      }
    }
  }
`;
// End of file

// AddOrder.js file
import React, { Component } from 'react';
import { gql } from 'apollo-boost';
import { Mutation } from 'react-apollo';

// Total orders query
import { TOTAL_ORDERS_QUERY } from './Queries';

// `addOrder` mutation
const ADD_ORDER_MUTATION = gql`
  mutation($input: AddOrderInput!) {
    addOrder(input: $input) {
      id
      date
      product
      status
    }
  }
`;

// Main `AddOrder` component
export default class AddOrder extends Component {
  constructor(props) {
    super(props);
    this.state = {
      product: "",
      date: new Date().toISOString().substring(0, 10),
      status: this.props.status[0]
    }
  }

  render() {
    // Destructure the status array to render <option />
    // items for the <select /> tag
    const { status } = this.props;

    return (
      <form
        onSubmit={evt => {
          evt.preventDefault();
          this.setState({ product: "" })
        }}
        >
        <input
          type="date"
          value={this.state.date}
          onChange={e => this.setState({
            date: e.target.value
          })}
        />
        <select
          value={this.state.status}
          onChange={e => this.setState({
            status: e.target.value
          })}
          >
          {status.map(status => (
            <option
              key={status}
              value={status}>
              {status.toLowerCase()}
            </option>
          ))}
        </select>
        <Mutation
          mutation={ADD_ORDER_MUTATION}
          refetchQueries={[{ query: TOTAL_ORDERS_QUERY }]}>
          {addOrder =>
            <button
              onClick={() => addOrder({
                variables: {
                  input: this.state
                }
              })}
              >
              Add Order
            </button>
          }
        </Mutation>
      </form>
    );
  }
}
// End of file

// ListOrders.js file
import React from 'react';

// List orders functional component that renders
// the current list of orders to the UI.
const ListOrders = ({ totalOrders, allOrders }) => {
  return (
    <table border={1}>
      <thead>
        <tr>
          <th colspan={3}>Total Orders: {totalOrders}</th>
        </tr>
      </thead>
      <tbody>
        {allOrders.map({ id, date, product, status }) => (
          <tr key={id}>
            <td>{id}</td>
            <td>{date}</td>
            <td>{product}</td>
            <td>{status}</td>
          </tr>
        )}
      </tbody>
    </table>
  );
}

export default ListOrders;
// End of file

// Main App.js file
import React from 'react';
import { gql } from 'apollo-boost';
import { Query } from 'react-apollo';

import ListOrders from './ListOrders';
import AddOrder from './AddOrder';

// Total orders query
import { TOTAL_ORDERS_QUERY } from './Queries';

// Main App component
const App = () => (
  <div>
    <h1>Orders</h1>
    <Query query={TOTAL_ORDERS_QUERY}>
      {({ data, loading, error }) => {
        if(error) return <p>Error: {error.message}</p>;

        if(loading) return <p>Loading...</p>;

        const status = data
                        .status
                        .enumValues
                        .map(status => status.name);
        return (
          <div>
            <ListOrders
              totalOrders={data.totalOrders}
              allOrders={data.allOrders}
            />
            <AddOrder status={status} />
          </div>
        );
      }}
    </Query>
  </div>
);
```

In the above example, we could replace the `refetchQueries` prop with a local cache updater function to avoid an extra query to refetch and refresh the UI. Below is an example of how to use one in the `AddOrder` component:

```javascript
// AddOrder.js file

// Only the relevant pieces of code are shown below...
// ...
import { TOTAL_ORDERS_QUERY } from './Queries';

export default class AddOrder extends Component {
  ...

  // This updater function takes in the instantiated apollo
  // client and the data response from the mutation.
  updateLocalCache = (client, { data }) => {

    // First grab the data we need from the existing query in
    // local cache using client.readQuery() method.
    const { totalOrders, allOrders } = client.readQuery({
      query: TOTAL_ORDERS_QUERY
    });

    // Then we can update the query on the local cache using
    // client.writeQuery() method like so.
    client.writeQuery({
      query: TOTAL_ORDERS_QUERY,
      data: {
        totalOrders: totalOrders + 1,
        allOrders: [...allOrders, data.addOrder]
      }
    });
  }

  render() {
    return (
      ....
      <Mutation
        mutation={ADD_ORDER_MUTATION}
        update={this.updateLocalCache}
      >
        {addOrder => ...}
      </Mutation>
    )
  }
}
```

### 💡 Similar to the `addOrder` mutation, below is an example of how to use a `removeOrder` mutation in a react component to remove orders from the list:

```javascript
// RemoveOrder.js file

// Our queries
const TOTAL_ORDERS_QUERY = gql`
  query orders {
    totalOrders
    allOrders {
      id
      date
      product
      status
    }
  }
`;

// Our mutation
const REMOVE_ORDER_MUTATION = gql`
  mutation remove($id: ID!) {
    removeOrder(id: $id) {
      removed
      totalAfter
      removedOrder {
        id
        date
        product
        status
      }
    }
  }
`;

// Our local apollo cache updater function.
const updateLocalCache = (client, { data }) => {
  const { allOrders } = client.readQuery({
    query: TOTAL_ORDERS_QUERY
  });

  // If an order was removed via the endpoint, let's update
  // our local cache to reflect that change.
  if (data.removeOrder.removed) {
    client.writeQuery({
      query: TOTAL_ORDERS_QUERY,
      data: {
        totalOrders: data.removeOrder.totalAfter,
        allOrders: allOrders.filter(
          order => order.id !== data.removeOrder.removedOrder.id
        )
      }
    });
  }
};

// Our remove order component to remove an order from the list
// via a local graphql endpoint.
export const RemoveOrder = ({ orderId }) => (
  <Mutation mutation={REMOVE_ORDER_MUTATION} update={updateLocalCache}>
    {removeOrder => (
      <button
        onClick={() =>
          removeOrder({
            variables: { id: orderId }
          })
        }
      >
        ⛔️
      </button>
    )}
  </Mutation>
);
```

Since, we have made the `removeOrder` mutation reusable via the `<RemoveOrder />` component, other components can use this component by just passing in an `orderId` prop to specify the order to be removed.

### 👏 💥 🥁 🎉 🎊 🥳 Congratulations !! we made it through the end and that's a wrap of our quick reference guide on how to use GraphQL data with React !!

### 🏆 These notes are inspired by the amazing egghead tutorials on these topics by [Eve Porcello](https://egghead.io/instructors/eve-porcello) and [Alex Banks](https://egghead.io/instructors/alex-banks). They are a great resource for anyone starting with GraphQL. Be sure to check them out on egghead.io

### 🔥 If you like, you can also check out the sample [react-apollo app](https://github.com/jvikraman/react-apollo-graphql) that demonstrates the concepts discussed in this guide.

---
