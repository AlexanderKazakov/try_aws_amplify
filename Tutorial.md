# AWS Amplify Full-Stack (Serverless) Tutorial — A Gentle, Thorough Guide

This booklet is written for an experienced programmer (Python/C++) who is new to AWS and JavaScript/TypeScript. We’ll move carefully, repeat the essentials, and use analogies to familiar tooling you know (FastAPI, CMake, servers vs serverless) so everything clicks.

We’ll use your existing project (`try_aws_amplify/`) as the running example. The end goal is that you can read this once and then recognize what each file does, how the pieces connect, and when to use Amplify vs other AWS options.

References used and worth bookmarking:
- AWS Hands-on Tutorial (Generative AI, Amplify, Bedrock, Cognito, AppSync): `https://docs.aws.amazon.com/hands-on/latest/build-serverless-web-app-lambda-amplify-bedrock-cognito-gen-ai/build-serverless-web-app-lambda-amplify-bedrock-cognito-gen-ai.html`


## 0) The Big Picture First

You’re building a serverless web application. “Serverless” does not mean “no servers exist”; it means you don’t manage them. AWS manages the servers, scaling, and much of the glue, while you supply:
- A frontend (React) that runs in the browser
- A backend defined via “Infrastructure-as-Code” (IaC) using Amplify’s definitions
- Small bits of actual backend logic (handlers) that run inside AWS services (in your app, inside AppSync as JavaScript resolvers)

Think of this as replacing your familiar “FastAPI server + database + Nginx” with “AWS managed building blocks” wired together by configuration and small functions.

A very simplified data flow for your app:

Browser (React) → Amplify Client → AppSync (GraphQL API) → Your resolver code → Bedrock (Claude model) → back to Browser

No EC2, no Docker, no long-running server you maintain. AWS provides the runtime and scaling.


## 1) Tour of Your Project Structure

Your `try_aws_amplify/` folder has two worlds: frontend and backend (IaC + handlers).

- `index.html`: The page that loads the React app.
- `src/`: Your React frontend code
  - `main.tsx`: Entry point (like `int main()`); renders the app and wraps it in an Amplify `<Authenticator>` component (login UI provided by Amplify UI).
  - `App.tsx`: The main UI — your form where the user enters ingredients and receives an AI-generated recipe.
  - `App.css`, `index.css`, assets: Styling and images.
- `package.json`, `vite.config.ts`, `tsconfig.*.json`: Tooling and TypeScript configs — think of these like a combination of CMake + compiler flags + virtualenv metadata for the JavaScript ecosystem.

- `amplify/`: This is your backend definition via Amplify
  - `backend.ts`: The “backend main”. It imports and wires up the modules of your backend (auth + data). It also registers an HTTP data source pointing to Bedrock and grants it permission to call a specific model.
  - `auth/resource.ts`: Declares authentication (Cognito) settings — email signup, verification code content, etc.
  - `data/resource.ts`: Declares the GraphQL API schema (types, queries, auth), and points the `askBedrock` query to a custom handler.
  - `data/bedrock.js`: The actual backend code that runs at request time inside AppSync. It formats the request to Bedrock (request handler) and parses Bedrock’s reply (response handler).
  - `amplify/package.json`, `amplify/tsconfig.json`: Backend-side build/type settings for the code that runs in AWS (the resolvers).

Analogy: `amplify/` contains blueprints (declarations) describing which AWS services to create and how to connect them, plus small functions that actually execute at runtime. It’s as if you combined Terraform (for infra) with a few Python functions that run when a request arrives — except the functions here are JS resolvers run by AppSync.


## 2) Key AWS Services in Your App

- Amplify: The framework/CLI that lets you define and deploy full-stack apps on AWS quickly. It orchestrates the creation of services, generates configuration for the frontend, and provides typed clients to call your backend.
- Amazon Cognito: User authentication (signup, login, MFA, user pools). In your app, email-based login with verification codes.
- AWS AppSync: Managed GraphQL API service. It’s the “backend API server” you don’t have to host yourself. You define a schema (queries, mutations, types), and AppSync routes requests to resolvers/data sources.
- Amazon Bedrock: Managed access to foundation models (like Anthropic Claude). You call it via HTTP with signed requests. Your app uses Claude 3 Sonnet.

How they fit together:
Frontend (React + Amplify client) → AppSync (GraphQL) → Resolver (your JS code) → Bedrock (AI model) → AppSync → Frontend
Cognito is used by the frontend and AppSync to authenticate users and authorize requests.


## 3) “Resources”, “Schema”, and “Data Sources” — Terms Unpacked

- Resource: In AWS-speak, any managed component (Cognito user pool, AppSync API, S3 bucket, Lambda function). In your code, `auth` and `data` are resource modules that declare and configure such components.

- GraphQL Schema: The API contract for your backend. It declares what queries exist, what arguments they accept, and what types they return. In your project, `askBedrock(ingredients: [String]) => BedrockResponse` is declared in `amplify/data/resource.ts`.
  - Backend defines the schema.
  - The frontend imports the generated types and a typed client so your calls are type-safe and discoverable.

- Data Source: A thing your GraphQL API can fetch data from — a database, a Lambda, another HTTP API. You added an HTTP data source that points to the Bedrock runtime endpoint.
  - “bedrockDataSource” means “use Bedrock as a data source for AppSync”.


## 4) Resolvers vs Lambdas (Where Does Code Run?)

Your project uses **AppSync JavaScript resolvers** (the functions in `bedrock.js`). These are small JS functions that run inside the AppSync service. They:
- Inspect request context (arguments, identity)
- Build an outgoing HTTP request (to Bedrock)
- Parse the HTTP response and return data to the client

No AWS Lambda functions are created in your current setup.

Could you use Lambda? Yes. Amplify can point resolvers to Lambda handlers instead of in-AppSync JavaScript. You’d do that if you needed more complex compute, dependencies, long execution, or to share libraries across handlers. For this tutorial (simple proxy to Bedrock with minor transformation), in-API JS resolvers are simpler and faster to iterate on.


## 5) Infrastructure-as-Code (IaC) vs Application Code

- IaC (Declarations): `defineBackend`, `defineData`, `defineAuth` are declarative specifications. They don’t “run” at request time. Instead, the Amplify CLI reads them and translates them into CloudFormation templates to create AWS resources. Think of it like writing a CMakeLists.txt or Terraform files — you describe what should exist.

- Application Code: `data/bedrock.js` contains actual runtime code. When a user calls `askBedrock`, AppSync executes your `request` and `response` handlers.

A rule of thumb:
- If it mentions `define*` and looks like a configuration object, it’s IaC.
- If it exports `function request(ctx)` or `function response(ctx)`, it’s runtime code.


## 6) Frontend: How It Calls the Backend

In `App.tsx`, you create a client from the generated schema and call your query:

- `generateClient<Schema>({ authMode: "userPool" })` constructs a typed client that knows about `askBedrock`.
- On form submit, it calls `amplifyClient.queries.askBedrock({ ingredients: [...] })`.
- Amplify’s client handles auth tokens from Cognito and talks to AppSync.
- The result is displayed in the UI.

From your perspective, this feels much like calling a function stub — similar to having a typed gRPC client generated from `.proto` files.


## 7) GraphQL and AppSync (When and Why)

GraphQL is a query language for APIs. Instead of many endpoints (REST), there’s one endpoint (`/graphql`) and clients specify exactly which fields they need. AppSync is AWS’s managed GraphQL server.

Why teams like GraphQL:
- Clients can fetch nested data in one request.
- Strongly typed schema ensures consistency.
- Great DX (autocomplete, type safety).

When to use GraphQL/AppSync:
- Complex client apps with varying data needs (web/mobile)
- You want real-time subscriptions (AppSync supports them)
- You prefer a strongly-typed API surface

When to prefer REST (FastAPI, API Gateway + Lambda):
- Simple CRUD
- Public APIs with broad client support
- Heavy reliance on HTTP caching/CDNs for fixed payloads
- Your team’s experience is deep in REST and tooling

Amplify is not “built solely on AppSync,” but AppSync is the default for Amplify’s data layer. You can also integrate REST via API Gateway/Lambda within Amplify projects if needed.


## 8) Why Is the Module Named `data`?

In `amplify/data/resource.ts`, you define the “data layer”: GraphQL API, types, and (potentially) database models. The name “data” reflects that this module manages the API and its backing data sources (like DynamoDB or, in your case, HTTP to Bedrock). It’s broader than just “api”.

In `backend.ts`, you then do:

- `auth` — authentication module (Cognito)
- `data` — data/API module (AppSync + data sources)


## 9) Is This a Normal Project Structure?

Yes for Amplify projects. Amplify is frontend-first: you typically start with a React app, then add an `amplify/` folder for backend definitions and handlers. In other ecosystems, you might see separate `frontend/` and `backend/` directories. Here, the “backend” is mostly definitions + tiny resolvers, so living alongside the frontend is common.

If you later add a traditional backend (e.g., a FastAPI microservice), you might create a sibling folder like `services/api/` or a `backend/` directory to keep it separate.


## 10) Can I Click This Together in the AWS Console?

You can, but it’s tedious and error-prone for anything beyond a toy. You would manually create a Cognito user pool, an AppSync API, IAM roles and policies, an HTTP data source to Bedrock, resolver mappings, hosting, etc. It’s easy to misconfigure something.

IaC (Amplify/CDK/Terraform) is the recommended path for anything you’ll deploy more than once. It’s versioned, reviewable, and repeatable. Use the console mainly to inspect, debug, and learn.


## 11) When to Use Amplify vs Other AWS Paths

Use Amplify when:
- You want to ship a full-stack app quickly on AWS
- You need auth + data + hosting out of the box
- You’re okay with AWS opinionated defaults and directory structure
- Frontend frameworks are your main interface to the app (React, Vue)

Consider alternatives when:
- You want traditional servers/frameworks you already know (FastAPI, Django):
  - Deploy on Elastic Beanstalk or ECS; pair with RDS (PostgreSQL)
- You need precise control over infra, runtimes, networking:
  - Use CDK or Terraform with API Gateway + Lambda + DynamoDB
- You need containers or microservices:
  - ECS/EKS or App Runner

Special case — desktop app registration/billing page:
- Easiest path for you (given your background): FastAPI + Stripe, deploy on Elastic Beanstalk or Lightsail. Add Cognito or stick with your own auth if needed.
- Amplify shines if you later want a richer web app with hosted auth, file storage, and a typed data layer.


## 12) Walking Through Your Backend Code (Concrete)

- `amplify/backend.ts`:
  - Declares the backend with `auth` and `data` modules.
  - Adds an HTTP data source named `bedrockDS` pointing to `https://bedrock-runtime.eu-central-1.amazonaws.com`.
  - Grants the IAM permission for `bedrock:InvokeModel` on the Claude Sonnet model ARN.

- `amplify/data/resource.ts`:
  - Defines a GraphQL schema with a custom type `BedrockResponse { body: String, error: String }`.
  - Declares a query `askBedrock(ingredients: [String])` returning `BedrockResponse`.
  - Requires authenticated users.
  - Attaches a custom handler with `entry: "./bedrock.js"` and `dataSource: "bedrockDS"`.

- `amplify/data/bedrock.js` (runtime resolvers):
  - `request(ctx)`: builds the Bedrock HTTP request body using the provided ingredients, setting the model path and Anthropic/Claude message format.
  - `response(ctx)`: parses the Bedrock response JSON and extracts the text into `body`.

- `src/App.tsx`:
  - Configures Amplify using `amplify_outputs.json` (generated by Amplify after deploy/sandbox).
  - Creates a typed client and calls `askBedrock` on form submission.
  - Displays the result or a loading indicator.


## 13) Common Mental Models and Analogies

- IaC (Amplify) ≈ CMake/Terraform: You describe components and connections; a tool realizes them.
- AppSync GraphQL schema ≈ FastAPI route signatures + Pydantic models: Strongly-typed API boundary.
- Resolvers (JS) ≈ FastAPI route functions: Code that runs per request to fetch/transform data.
- Data sources ≈ database connections or external API clients.
- Amplify client ≈ gRPC/Thrift/OpenAPI-generated client: Typed calls from frontend to backend.


## 14) Pros and Cons of This Approach

Pros:
- Rapid full-stack development; a lot “just works” (auth, hosting, API)
- No server management; automatic scaling
- Strong typing from backend to frontend
- Clear security model via Cognito and IAM

Cons:
- AWS lock-in and Amplify conventions
- Debugging resolvers/IAM can be unfamiliar at first
- GraphQL learning curve if you come from REST-only
- Less control vs hand-rolled infra (until you mix in CDK/Lambda)


## 15) What to Try Next

- Add a new field to `BedrockResponse` (e.g., `modelName: string`) and return it from `response(ctx)` — see the types flow through to the frontend.
- Switch the resolver to a Lambda function (for comparison), then call Bedrock from Lambda using the AWS SDK for Bedrock.
- Replace the Bedrock model ARN/endpoint with another allowed model (ensure permissions match).
- Protect the API further (rate limits, input validation) in the resolver.


## 16) Quick Glossary

- Amplify: AWS toolkit for building and deploying full-stack apps.
- Cognito: AWS user authentication and identity management.
- AppSync: Managed GraphQL server.
- GraphQL: A query language for APIs, strongly typed, single endpoint.
- Resolver: Code that fulfills a GraphQL field/query/mutation.
- Data Source: Backend target for resolvers (DB, Lambda, HTTP, etc.).
- IaC: Infrastructure-as-Code; describe infra in files and deploy with tooling.


## 17) Final Words

If you’re comfortable with FastAPI and traditional servers, Amplify will feel different at first: fewer routes and servers you write, more declarations and small resolver functions. That’s by design. You trade some control for speed, scalability, and managed services.

This tutorial emphasized concepts and mental models over button clicks, so you can reason about what Amplify is doing on your behalf. Once these ideas land, the AWS Hands-on tutorial will feel much more straightforward.

Keep this booklet handy as you work — and don’t hesitate to revisit the analogies when something feels abstract. Repetition is a feature, not a bug, when learning a new platform.
