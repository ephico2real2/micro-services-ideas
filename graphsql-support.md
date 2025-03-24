To add GraphQL support to your existing API Gateway layer, you'll need to implement it alongside your current REST API architecture. Here's how you can enhance your Next.js Backend API to support GraphQL:

## Adding GraphQL Support to Your Next.js API Gateway

### 1. Architecture Updates

```
      API GATEWAY LAYER                                 
                                                                               
     ┌───────────────────────────────────────────────────────────────────┐     
     │                                                                   │     
     │                    Next.js Backend API                            │     
     │   - Authentication & Authorization                                │     
     │   - API orchestration                                            │     
     │   - Request routing                                              │     
     │   - Response formatting                                          │     
     │                                                                   │     
     │   ┌─────────────────────┐      ┌──────────────────────┐          │     
     │   │   REST API Routes   │      │   GraphQL Endpoint   │          │     
     │   │   (/api/*)          │      │   (/api/graphql)     │          │     
     │   └─────────────────────┘      └──────────────────────┘          │     
     │                                         │                         │     
     │                                         ▼                         │     
     │                             ┌──────────────────────┐             │     
     │                             │   Schema & Resolvers │             │     
     │                             └──────────────────────┘             │     
     │                                                                   │     
     └───────────────────────────────────────────────────────────────────┘     
```

### 2. Implementation Steps

#### Install Required Dependencies

```bash
npm install graphql apollo-server-micro @graphql-tools/schema
```

#### Create a GraphQL Schema

Create a file at `graphql/schema.js`:

```javascript
const { gql } = require('apollo-server-micro');

const typeDefs = gql`
  type Query {
    hello: String
    # Add your query definitions here
  }
  
  type Mutation {
    # Add your mutation definitions here
  }
  
  # Define your types here
`;

module.exports = typeDefs;
```

#### Create Resolvers

Create a file at `graphql/resolvers.js`:

```javascript
const resolvers = {
  Query: {
    hello: () => 'Hello World!',
    // Add more query resolvers
  },
  Mutation: {
    // Add mutation resolvers
  },
  // Add type resolvers
};

module.exports = resolvers;
```

#### Set Up Apollo Server in Next.js

Create a file at `pages/api/graphql.js`:

```javascript
import { ApolloServer } from 'apollo-server-micro';
import { makeExecutableSchema } from '@graphql-tools/schema';
import typeDefs from '../../graphql/schema';
import resolvers from '../../graphql/resolvers';
import Cors from 'micro-cors';

// Import your authentication middleware
import { authenticateUser } from '../../middleware/auth';

const cors = Cors();

const schema = makeExecutableSchema({
  typeDefs,
  resolvers,
});

const apolloServer = new ApolloServer({
  schema,
  context: async ({ req }) => {
    // Add authentication to GraphQL
    try {
      const user = await authenticateUser(req);
      return { user };
    } catch (error) {
      // Handle authentication errors
      console.error('Authentication error:', error);
      return {};
    }
  },
});

const startServer = apolloServer.start();

export default cors(async (req, res) => {
  if (req.method === 'OPTIONS') {
    res.end();
    return false;
  }

  await startServer;
  await apolloServer.createHandler({
    path: '/api/graphql',
  })(req, res);
});

// Configure API route to handle Apollo Server
export const config = {
  api: {
    bodyParser: false,
  },
};
```

### 3. Integration with Existing Services

To leverage your existing services with GraphQL, modify your resolvers to call the same service functions your REST API uses:

```javascript
// graphql/resolvers.js
const { getUserData } = require('../services/userService');
const { getPreviewData } = require('../services/previewService');

const resolvers = {
  Query: {
    user: async (_, { id }, context) => {
      // Use the same authorization check as your REST endpoints
      if (!context.user) throw new Error('Not authenticated');
      
      // Call the same service function used by your REST API
      return await getUserData(id);
    },
    previews: async (_, { sourceId }, context) => {
      if (!context.user) throw new Error('Not authenticated');
      return await getPreviewData(sourceId);
    }
  },
  // More resolvers...
};
```

### 4. Type Definitions for Preview Service

For your Preview Generation Service, you would create these type definitions:

```graphql
type Preview {
  previewId: ID!
  sourceId: String!
  startTime: Float!
  duration: Float!
  position: String!
  width: Int
  height: Int
  format: String
  previewUrl: String!
  thumbnailUrl: String
}

type PreviewResult {
  success: Boolean!
  message: String
  previews: [Preview!]
}

input PreviewGenerationInput {
  sourceId: String!
  userId: String!
  previewCount: Int
  duration: Int
  quality: String
  format: String
}

extend type Query {
  previewsBySource(sourceId: String!): [Preview!]!
  preview(previewId: ID!): Preview
}

extend type Mutation {
  generatePreviews(input: PreviewGenerationInput!): PreviewResult!
  deletePreview(previewId: ID!): Boolean!
}
```

### 5. Supporting Both REST and GraphQL

Your API gateway can now support both REST and GraphQL simultaneously:

- **REST API**: Continues to work as before via `/api/*` routes
- **GraphQL API**: Available at `/api/graphql` endpoint

This allows clients to choose which API style works best for their needs.

### 6. Client Usage Example

```javascript
// Using the GraphQL API
const response = await fetch('/api/graphql', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`
  },
  body: JSON.stringify({
    query: `
      query GetPreviews($sourceId: String!) {
        previewsBySource(sourceId: $sourceId) {
          previewId
          sourceId
          previewUrl
          thumbnailUrl
          duration
          position
        }
      }
    `,
    variables: {
      sourceId: 'video-123'
    }
  })
});

const data = await response.json();
```

### 7. Benefits of Adding GraphQL

1. **Flexible data fetching**: Clients can request exactly what they need
2. **Reduced network requests**: Multiple resources in a single request
3. **Type safety**: Strongly typed schema provides better developer experience
4. **Introspection**: Built-in documentation through GraphQL Playground/Explorer
5. **Gradual adoption**: You can migrate endpoints gradually while maintaining REST compatibility

This approach allows you to enhance your API Gateway with GraphQL capabilities while preserving your existing REST API infrastructure.
