[Okta Developer home](https://developer.okta.com/)

Community

[Forum](https://devforum.okta.com/)

* * *

- [Okta Developers on GitHub](https://github.com/oktadev)
- [Okta Developers on X](https://x.com/OktaDev)
- [Okta Developers on YouTube](https://www.youtube.com/oktadev)
- [Okta Developers on LinkedIn](https://www.linkedin.com/company/oktadev/)

- [Blog](https://developer.okta.com/blog/)
- [Pricing(opens in a new tab)](https://developer.okta.com/pricing/)

- [Okta Docs(opens in a new tab)](https://help.okta.com/)
- [Okta(opens in a new tab)](https://okta.com/)
- [Sign in to Okta(opens in a new tab)](https://login.okta.com/)

[Sign Up](https://developer.okta.com/signup/)

- [Blog](https://developer.okta.com/blog/)
- [Pricing(opens in a new tab)](https://developer.okta.com/pricing/)
- [Okta Docs(opens in a new tab)](https://help.okta.com/)
- [Okta(opens in a new tab)](https://okta.com/)
- [Sign in to Okta(opens in a new tab)](https://login.okta.com/)
- [Forum](https://devforum.okta.com/)
- [Sign Up](https://developer.okta.com/signup/)

- [Okta Developers on GitHub](https://github.com/oktadev)
- [Okta Developers on X](https://x.com/OktaDev)
- [Okta Developers on YouTube](https://www.youtube.com/oktadev)
- [Okta Developers on LinkedIn](https://www.linkedin.com/company/oktadev/)

- [agentic-ai](https://developer.okta.com/blog/tags/agentic-ai/)
- [cross-app-access](https://developer.okta.com/blog/tags/cross-app-access/)
- [enterprise-ai](https://developer.okta.com/blog/tags/enterprise-ai/)
- [mcp](https://developer.okta.com/blog/tags/mcp/)
- [oauth](https://developer.okta.com/blog/tags/oauth/)
- [sso](https://developer.okta.com/blog/tags/sso/)
- [xaa](https://developer.okta.com/blog/tags/xaa/)

# Build Secure Agent-to-App Connections with Cross App Access (XAA) Using OIDC

[![Sohail Pathan](https://developer.okta.com/blog/img/avatar/qGi6HMtV9h-48.webp) Sohail Pathan](https://developer.okta.com/blog/authors/sohail-pathan/)\|September 3, 2025\|

Last Updated: [August 24, 2026](https://developer.okta.com/blog/2025/09/03/cross-app-access/#changelog)

\|

9 min read

\|[Share on X](https://twitter.com/intent/tweet?url=https%3A%2F%2Fdeveloper.okta.com%2Fblog%2F2025%2F09%2F03%2Fcross-app-access%2F&text=Build%20Secure%20Agent-to-App%20Connections%20with%20Cross%20App%20Access%20(XAA)%20Using%20OIDC)[Share on LinkedIn](https://www.linkedin.com/sharing/share-offsite/?url=https%3A%2F%2Fdeveloper.okta.com%2Fblog%2F2025%2F09%2F03%2Fcross-app-access%2F)

![](https://developer.okta.com/blog/img/cross-app-access/l8UWz9SZKe-100.webp)

## [⚠️ This guide is out of date](https://developer.okta.com/blog/2025/09/03/cross-app-access/\#%EF%B8%8F-this-guide-is-out-of-date)

> The Okta Admin Console steps in this guide no longer work: the **XAA Resource App** and **XAA Requesting App** App Catalog integrations, along with the **Manage Connections** tab, have been replaced by AI agent registration under **Directory > AI Agents**.
>
> Building the app that owns the API, validates the ID-JAG, and issues its own access token? Read [Add Cross App Access to Your OIDC Resource Application](https://developer.okta.com/blog/2026/08/24/xaa-oidc-resource).
>
> Building the app that acts on the user's behalf and exchanges a refresh token for an ID-JAG? Read [Add Cross App Access to Your OIDC Requesting Application](https://developer.okta.com/blog/2026/08/21/xaa-oidc-requesting).
>
> Both guides cover the current Admin Console configuration and let you verify the flow against a hosted test app instead of running a local sample.

So you want to get your AI agent in front of Okta customers or make your application securely accessible to agents — you're in the right place. Here's why it matters.

Secure access with enterprise IT oversight between independent applications that communicate with each other is a recognized gap in [OAuth 2.0](https://developer.okta.com/docs/concepts/oauth-openid/). Enterprises can't effectively regulate cross-app communication, as OAuth 2.0 consent screens rely on users granting access to their individual accounts. Now, with the advent of AI agents that communicate across systems, the need to solve the gap is even greater – especially given the growing importance of enterprise AI security in protecting sensitive data flows.

## [What is Cross App Access (XAA)?](https://developer.okta.com/blog/2025/09/03/cross-app-access/\#what-is-cross-app-access-xaa)

Cross App Access (XAA) is a new protocol that lets integrators enable secure agent-to-app and app-to-app access. Instead of scattered integrations and repeated logins, enterprise IT admins gain centralized control: they can decide what connects, enforce security policies, and see exactly what's being accessed. This unlocks seamless, scalable integrations across apps — whether it's just two like Google Calendar and Zoom, or hundreds across the enterprise. Read more about Cross App Access in this post:

[Integrate Your Enterprise AI Tools with Cross App Access (XAA)](https://developer.okta.com/blog/2025/06/23/enterprise-ai/)

Manage user and non-human identities, including AI in the enterprise with Cross App Access (XAA)

[![](https://developer.okta.com/blog/img/avatar/uir2rLXILh-40.webp) Semona Igama](https://developer.okta.com/blog/authors/semona-igama/)

Or watch the video about Cross App Access:

Secure Enterprise AI with MCP + Cross App Access (XAA) - YouTube

Tap to unmute

[Secure Enterprise AI with MCP + Cross App Access (XAA)](https://www.youtube.com/watch?v=3VLzeT1EGrg) [OktaDev](https://www.youtube-nocookie.com/channel/UC5AMiWqFVFxF1q9Ya1FuZ_Q)

![thumbnail-image](https://yt3.ggpht.com/06W3SQwZWkYVl56Oh6Pch_n1rFrS5AjmRUKlkzXNS7qLrdZibj-JVzPGxRGKb9Ts8nT_Rq3w=s68-c-k-c0x00ffffff-no-rj)

OktaDev67.3K subscribers

[Watch on](https://www.youtube.com/watch?v=3VLzeT1EGrg)

In this post, we'll go hands-on with Cross App Access. Using **Todo0** (the Resource App) and **Agent0** (the Requesting App) as our sample applications, and **Okta as the enterprise Identity Provider (IdP)**, we'll show you how to set up trust, exchange tokens, and enable secure API calls between apps that enable enterprise IT oversight. By the end, you'll not only understand how the protocol works but also have a working example you can adapt to your own integrations.

## [Prerequisites to set up the AI agent to app connections using Cross App Access (XAA)](https://developer.okta.com/blog/2025/09/03/cross-app-access/\#prerequisites-to-set-up-the-ai-agent-to-app-connections-using-cross-app-access-xaa)

To set up secure agent-to-app connections with Cross App Access (XAA), you'll need the following:

1. **Okta Developer Account (Integrator Free Plan):** You'll need an Okta Developer Account with the Integrator Free Plan. This account will act as your Identity Provider (IdP) for setting up Cross App Access.

   - If you don't already have an account, sign up for a new one here: [Okta Integrator Free Plan](https://developer.okta.com/signup)
   - Once created, sign in to your new org
2. **AWS Credentials:** You'll need an **AWS Access Key ID** and **AWS Secret Access Key**

   - The IAM user or role associated with these credentials must have access to **Amazon Bedrock,** specifically the **Claude 3.7 Sonnet model,** enabled
   - If you don't know how to obtain the credentials, [follow this guide](https://github.com/oktadev/okta-cross-app-access-mcp/blob/main/guide/aws-bedrock.md)
3. **Developer Tools:** These tools are essential for cloning, editing, building, and running your demo applications

   - **[Git](https://git-scm.com/downloads)** – to clone and manage the repository
   - **[VS Code](https://code.visualstudio.com/Download)** – for reading and modifying the sample source code
   - **[Dev Containers Extension (VS Code)](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers)** – recommended, as it automatically configures dependencies and environments when you open the project
   - **[Docker](https://www.docker.com/products/docker-desktop/)** – required by the Dev Container to build and run the sample applications in isolated environments

Table of Contents

- [What is Cross App Access (XAA)?](https://developer.okta.com/blog/2025/09/03/cross-app-access/#what-is-cross-app-access-xaa)
- [Prerequisites to set up the AI agent to app connections using Cross App Access (XAA)](https://developer.okta.com/blog/2025/09/03/cross-app-access/#prerequisites-to-set-up-the-ai-agent-to-app-connections-using-cross-app-access-xaa)
- [Use Okta to secure AI applications with OAuth 2.0 and OpenID Connect (OIDC)](https://developer.okta.com/blog/2025/09/03/cross-app-access/#use-okta-to-secure-ai-applications-with-oauth-20-and-openid-connect-oidc)
  - [Enable Cross App Access in your Okta org](https://developer.okta.com/blog/2025/09/03/cross-app-access/#enable-cross-app-access-in-your-okta-org)
  - [Create the resource app (Todo0)](https://developer.okta.com/blog/2025/09/03/cross-app-access/#create-the-resource-app-todo0)
  - [Create the requesting app (Agent0)](https://developer.okta.com/blog/2025/09/03/cross-app-access/#create-the-requesting-app-agent0)
  - [Establishing connections between Todo0 & Agent0](https://developer.okta.com/blog/2025/09/03/cross-app-access/#establishing-connections-between-todo0--agent0)
- [Set up a test user in Okta org](https://developer.okta.com/blog/2025/09/03/cross-app-access/#set-up-a-test-user-in-okta-org)
  - [Create the test user](https://developer.okta.com/blog/2025/09/03/cross-app-access/#create-the-test-user)
  - [Assign the Okta applications to the test user](https://developer.okta.com/blog/2025/09/03/cross-app-access/#assign-the-okta-applications-to-the-test-user)
- [Configure the Node.js Cross App Access project](https://developer.okta.com/blog/2025/09/03/cross-app-access/#configure-the-nodejs-cross-app-access-project)
  - [The Cross App Access MCP project at a glance](https://developer.okta.com/blog/2025/09/03/cross-app-access/#the-cross-app-access-mcp-project-at-a-glance)
- [Configure OAuth 2.0 and AI foundation models environment files](https://developer.okta.com/blog/2025/09/03/cross-app-access/#configure-oauth-20-and-ai-foundation-models-environment-files)
  - [Generate OIDC configuration and access token files](https://developer.okta.com/blog/2025/09/03/cross-app-access/#generate-oidc-configuration-and-access-token-files)
  - [Configure application connection values](https://developer.okta.com/blog/2025/09/03/cross-app-access/#configure-application-connection-values)
  - [Register OAuth 2.0 redirect URIs for both apps](https://developer.okta.com/blog/2025/09/03/cross-app-access/#register-oauth-20-redirect-uris-for-both-apps)
- [Initialize the database and run the project](https://developer.okta.com/blog/2025/09/03/cross-app-access/#initialize-the-database-and-run-the-project)
  - [Bootstrap the project](https://developer.okta.com/blog/2025/09/03/cross-app-access/#bootstrap-the-project)
  - [Run and access the apps in your browser](https://developer.okta.com/blog/2025/09/03/cross-app-access/#run-and-access-the-apps-in-your-browser)
- [Testing the XAA flow: From Bob to Agent0 to Todo0](https://developer.okta.com/blog/2025/09/03/cross-app-access/#testing-the-xaa-flow-from-bob-to-agent0-to-todo0)
  - [Interact with Todo0, the XAA resource app, by creating tasks](https://developer.okta.com/blog/2025/09/03/cross-app-access/#interact-with-todo0-the-xaa-resource-app-by-creating-tasks)
  - [Let the Agent0's AI agent access your todos](https://developer.okta.com/blog/2025/09/03/cross-app-access/#let-the-agent0s-ai-agent-access-your-todos)
- [Behind the scenes: the OAuth 2.0 Identity Assertion Authorization Grant](https://developer.okta.com/blog/2025/09/03/cross-app-access/#behind-the-scenes-the-oauth-20-identity-assertion-authorization-grant)
- [Need help setting up secure cross-domain enterprise agentic application access?](https://developer.okta.com/blog/2025/09/03/cross-app-access/#need-help-setting-up-secure-cross-domain-enterprise-agentic-application-access)
- [Learn more about Cross App Access, OAuth 2.0, and securing your applications](https://developer.okta.com/blog/2025/09/03/cross-app-access/#learn-more-about-cross-app-access-oauth-20-and-securing-your-applications)

## [Use Okta to secure AI applications with OAuth 2.0 and OpenID Connect (OIDC)](https://developer.okta.com/blog/2025/09/03/cross-app-access/\#use-okta-to-secure-ai-applications-with-oauth-20-and-openid-connect-oidc)

Before we dive into the code, we need to register our apps with Okta. In this demo:

- **Agent0**: the **agentic** **requesting app** (makes the API call on behalf of the user)
- **Todo0**: the **resource app** (owns the protected API)
- **Managed connection**: the trust relationship between the two apps, created in Okta

We'll create both apps in your Okta Integrator Free Plan account, grab their client credentials, and then connect them.

### [Enable Cross App Access in your Okta org](https://developer.okta.com/blog/2025/09/03/cross-app-access/\#enable-cross-app-access-in-your-okta-org)

> ⚠️ **Note:** Cross App Access is currently an **Early Access (EA) feature**. This guide covers the **OIDC** integration path and applies whether you're building a new OIDC-based application or adding XAA support to an existing one — as a resource app, a requesting app, or both.

**To get Cross App Access EA enabled for your org:**

- If you're an **Okta customer**: reach out to the customer support team
- If you're an **ISV or hold a developer integrator org**: reach out to [developers@okta.com](mailto:developers@okta.com)

### [Create the resource app (Todo0)](https://developer.okta.com/blog/2025/09/03/cross-app-access/\#create-the-resource-app-todo0)

1. In the Okta Admin console, navigate to **Applications > Applications**
2. Select **Browse App Catalog**
3. Search for **XAA Resource App**, and select it
4. Select **Add Integration**
5. Enter "Todo0" in the Application label field
6. In the **Issuer URL** field, enter `http://localhost:5001` (Todo0's Resource Authorization Server URL), then click **Done**
7. Click the **Sign On** tab to view the **Client ID** and **Client secret**. These are required to include in your `.env.todo`

![View Client ID and Client Secret for XAA Resource App in Okta Admin Console](https://developer.okta.com/blog/img/cross-app-access/B83_NVrIS6-800.webp)

### [Create the requesting app (Agent0)](https://developer.okta.com/blog/2025/09/03/cross-app-access/\#create-the-requesting-app-agent0)

1. Go back to **Applications > Applications**
2. Select **Browse App Catalog**
3. Search for **XAA Requesting App**, and select it
4. Select **Add Integration**
5. Enter **Agent0** in the Application label field
6. In the **Issuer URL** field, enter `http://localhost:5001` (Todo0's Resource Authorization Server URL)
7. In the **Client ID** field, enter `agent0-at-todo0` (Requesting App's Client ID in Resource Authorization Server), then click **Done**
8. Click the **Sign On** tab to view the **Client ID** and **Client secret**. These are required to be included in your `.env.agent`

![View Client ID and Client Secret for XAA Requesting App in Okta Admin Console](https://developer.okta.com/blog/img/cross-app-access/M6MZyMIlgK-800.webp)

### [Establishing connections between Todo0 & Agent0](https://developer.okta.com/blog/2025/09/03/cross-app-access/\#establishing-connections-between-todo0--agent0)

1. From the **Applications** page, select the **Agent0** app
2. Go to the **Manage Connections** tab
3. Under **App granted consent**, select **Add requesting apps**, select **Todo0**, then **Save**
4. Under **Apps providing consent**, select **Add resource apps**, select **Todo0**, then **Save**

![Connect Agent0 and Todo0 apps in Okta by managing connections](https://developer.okta.com/blog/img/cross-app-access/61sW0Dj5SI-800.webp)

Now **Agent0** and **Todo0** are connected **and Agent0's AI Agent can access Todo0's resources.** If you check the **Manage Connection** tab for either app, you'll see that the connection has been established.

## [Set up a test user in Okta org](https://developer.okta.com/blog/2025/09/03/cross-app-access/\#set-up-a-test-user-in-okta-org)

Now that the apps are in place, we need a test user who will sign in and trigger the Cross App Access flow.

### [Create the test user](https://developer.okta.com/blog/2025/09/03/cross-app-access/\#create-the-test-user)

1. In the **Okta Admin Console**, go to **Directory > People**
2. Select **Add Person**
3. Fill in the details:
   - **First name:** Bob
   - **Last name:** Tables
   - **Username / Email:**`bob@tables.fake`
4. Under **Activations**, select **Activate now**, mark **☑️ I will set password,** and create a temporary password
5. Optional: You can mark **☑️ User must change password on first login**
6. Select **Save** (If you don't see the new user right away, refresh the page)

![Create a test user Bob Tables in Okta Admin Console](https://developer.okta.com/blog/img/cross-app-access/sh-kZlyEhG-800.webp)

### [Assign the Okta applications to the test user](https://developer.okta.com/blog/2025/09/03/cross-app-access/\#assign-the-okta-applications-to-the-test-user)

1. Open the **Bob Tables** user profile
2. Select **Assign Applications**
3. Assign both **Agent0** (requesting app) and **Todo0** (resource app) to Bob

![Assign Agent0 and Todo0 applications to Bob Tables user in Okta](https://developer.okta.com/blog/img/cross-app-access/nutkMJ0un_-800.webp)

This ensures Bob can sign in to Agent0, and Agent0 can securely request access to Todo0 on his behalf.

> **⚠️ Note:** Bob will be the identity we use throughout this guide to demonstrate how Agent0 accesses Todo0's API through Cross App Access.

## [Configure the Node.js Cross App Access project](https://developer.okta.com/blog/2025/09/03/cross-app-access/\#configure-the-nodejs-cross-app-access-project)

With your Okta environment (apps and user) ready, let's set up the local project. Before we dive into configs, here's a quick look at what you'll be working with.

1. Clone the repository:


```shell
git clone https://github.com/oktadev/okta-cross-app-access-mcp
```

2. Change into the project directory:


```shell
cd okta-cross-app-access-mcp
```

3. Open **VS Code Command Palette** and run **"Dev Containers: Open Folder in Container"**

To open Command Palette, select View > Command Palette..., MacOS keyboard shortcut `Cmd+Shift+P`, or Windows keyboard shortcut `Ctrl+Shift+P`


> ⚠️ Note: This sets up all dependencies, including Node, Redis, Prisma ORM, and Yarn.

![Open project in VS Code Dev Container for local development](https://developer.okta.com/blog/img/cross-app-access/wVC6MgXZzP-800.webp)

### [The Cross App Access MCP project at a glance](https://developer.okta.com/blog/2025/09/03/cross-app-access/\#the-cross-app-access-mcp-project-at-a-glance)

```shell
okta-cross-app-access-mcp/
├─ packages/
│  ├─ agent0/               # Requesting app (UI + service) – runs on :3000
│  │  └─ .env               # Agent0 env (AWS creds)
│  ├─ todo0/                # Resource app (API/UI) – runs on :3001
│  ├─ authorization-server/ # Local auth server for ID-JAG + token exchange
│  │  └─ .env.agent         # IdP creds (Agent0 side)
│  │  └─ .env.todo          # IdP creds (Todo0 side)
│  ├─ id-assert-authz-grant-client/ # Implements Identity Assertion Authorization Grant client logic
├─ .devcontainer/           # VS Code Dev Containers setup
├─ guide/                   # Docs used by the README
├─ images/                  # Diagrams/screens used in README
├─ scripts/                 # Helper scripts
├─ package.json
└─ tsconfig.json
```

## [Configure OAuth 2.0 and AI foundation models environment files](https://developer.okta.com/blog/2025/09/03/cross-app-access/\#configure-oauth-20-and-ai-foundation-models-environment-files)

At this point, you have:

- **Client IDs and Client Secrets** for both **Agent0** and **Todo0** (from the Okta Admin Console)

- Your **Okta org URL**, visible in the Okta Admin Console profile menu of the Admin Console. It usually looks like


```text
https://integrator-123456.okta.com
```


This URL will be your **IdP issuer URL** and is shared across both apps.

### [Generate OIDC configuration and access token files](https://developer.okta.com/blog/2025/09/03/cross-app-access/\#generate-oidc-configuration-and-access-token-files)

From the project root, run:

```shell
yarn setup:env
```

This scaffolds the following files:

- `packages/authorization-server/.env.todo`
- `packages/authorization-server/.env.agent`
- `packages/agent0/.env`

### [Configure application connection values](https://developer.okta.com/blog/2025/09/03/cross-app-access/\#configure-application-connection-values)

Open each file and update the placeholder with your org-specific values:

**`authorization-server/.env.todo`**

```text
CUSTOMER1_EMAIL_DOMAIN=tables.fake
CUSTOMER1_AUTH_ISSUER=<Your integrator account URL>
CUSTOMER1_CLIENT_ID=<Todo0 client id>
CUSTOMER1_CLIENT_SECRET=<Todo0 client secret>
```

**`authorization-server/.env.agent`**

```text
CUSTOMER1_EMAIL_DOMAIN=tables.fake
CUSTOMER1_AUTH_ISSUER=<Your integrator account URL>
CUSTOMER1_CLIENT_ID=<Agent0 client id>
CUSTOMER1_CLIENT_SECRET=<Agent0 client secret>
```

**`agent0/.env`**

```text
AWS_ACCESS_KEY_ID=<your AWS access key id>
AWS_SECRET_ACCESS_KEY=<your AWS secret access key>
```

> **⚠️ Note:**
>
> 1. The **issuer URL** (`CUSTOMER1_AUTH_ISSUER`) is the same in both `.env.todo` and `.env.agent`
> 2. The **Client ID/Client secret** values differ because they come from the respective apps you created
> 3. AWS credentials are required only for Agent0 (requesting app)

### [Register OAuth 2.0 redirect URIs for both apps](https://developer.okta.com/blog/2025/09/03/cross-app-access/\#register-oauth-20-redirect-uris-for-both-apps)

Finally, we need to tell Okta where to send the authentication response for each app.

**For Agent0:**

1. From your Okta Admin Console, navigate to **Applications > Applications**

2. Open the **Agent0** app

3. Navigate to the **Sign On** tab

4. In the **Settings** section, select **Edit**

5. In the **Redirect URIs** field, add


```text
http://localhost:5000/openid/callback/customer1
```

6. Select **Save**


**Repeat the same steps for Todo0:**

1. Open the **Todo0** app

2. Go to the **Sign On** tab > **Settings** \> **Edit**

3. In the **Redirect URIs** field, add:


```text
http://localhost:5001/openid/callback/customer1
```

4. Select **Save**


Now both apps know where to redirect after authentication.

## [Initialize the database and run the project](https://developer.okta.com/blog/2025/09/03/cross-app-access/\#initialize-the-database-and-run-the-project)

With the apps and environment configuration in place, the next step is to prepare the local project, set up its databases, and bring both applications online.

### [Bootstrap the project](https://developer.okta.com/blog/2025/09/03/cross-app-access/\#bootstrap-the-project)

From the root of the repo, install all workspaces and initialize the databases:

```shell
yarn bootstrap
```

Since this is your first run, you'll be asked whether to reset the database. Type "`y`" for both Todo0 and Agent0.

### [Run and access the apps in your browser](https://developer.okta.com/blog/2025/09/03/cross-app-access/\#run-and-access-the-apps-in-your-browser)

Once the bootstrap is complete, start both apps (and their authorization servers) with:

```shell
yarn start
```

Open the following ports in your Chrome browser's tab:

- **Todo0 (Resource App):** [http://localhost:3001](http://localhost:3001/)
- **Agent0 (Requesting App):** [http://localhost:3000](http://localhost:3000/)

At this point, both apps should be live and connected through Okta. 🎉

## [Testing the XAA flow: From Bob to Agent0 to Todo0](https://developer.okta.com/blog/2025/09/03/cross-app-access/\#testing-the-xaa-flow-from-bob-to-agent0-to-todo0)

With everything configured, it's time to see Cross App Access in action.

### [Interact with Todo0, the XAA resource app, by creating tasks](https://developer.okta.com/blog/2025/09/03/cross-app-access/\#interact-with-todo0-the-xaa-resource-app-by-creating-tasks)

1. In the **Work Email** field, enter: `bob@tables.fake`, and select **Continue**![Sign in to Todo0 app using Bob Tables test user](https://developer.okta.com/blog/img/cross-app-access/W-o6Yt-CN2-800.webp)
2. You'll be redirected to the Okta Login page. Sign in with the test user credentials:
   - **Username:**`bob@tables.fake`
   - **Password:** the temporary password you created earlier
3. The first time you sign in, you'll be prompted to:
   - Set a new password
   - Enroll in [**Okta Verify**](https://help.okta.com/en-us/content/topics/mobile/okta-verify-overview.htm) for MFA
4. Once logged in, add several tasks to your to-do list
5. Select one of the tasks and mark it as complete to verify that the application updates the status accurately ![Add and complete tasks in Todo0 Resource App UI](https://developer.okta.com/blog/img/cross-app-access/4m-JDFjNB1-800.webp)

### [Let the **Agent0's** AI agent access your todos](https://developer.okta.com/blog/2025/09/03/cross-app-access/\#let-the-agent0s-ai-agent-access-your-todos)

1. Open the **Agent0** app in your browser ![Initialize AWS Bedrock client in Agent0 Requesting App](https://developer.okta.com/blog/img/cross-app-access/9ieuMxj_K6-800.webp)
2. Select **Initialize** to set up the AWS Bedrock client. Once connected, you'll see the following message:

`✅ Successfully connected to AWS Bedrock! You can now start chatting.`
3. Select the **Connect to IdP** button
   - Behind the scenes, Agent0 requests an identity assertion from Okta and exchanges it for an access token to Todo0
   - If everything is configured correctly, you'll see the following message

     `Authentication completed successfully! Welcome back.`![Authenticate Agent0 app with Okta and receive tokens](https://developer.okta.com/blog/img/cross-app-access/zBqWb2A3nR-800.webp)
4. To confirm that **Agent0** is actually receiving tokens from Okta:
   - Open a new browser tab and navigate to: `http://localhost:3000/api/tokens`
   - You should see a JSON payload containing: **`accessToken`, `jagToken`, and `idToken`** This verifies that Agent0 successfully authenticated through Okta and obtained the tokens needed to call Todo0 ![JSON payload containing tokens](https://developer.okta.com/blog/img/cross-app-access/XYE12Lqb2n-800.webp)
5. Now interact with Agent0 using natural prompts. For example: write this prompt

```text
What's on my plate in my to-do list?
```



> **⚠️ Note:** Agent0 will call the Todo0 API using the access token and return your pending tasks ![Agent0 app displays pending tasks from Todo0 using Cross App Access](https://developer.okta.com/blog/img/cross-app-access/LjNXKAj_UC-800.webp)

6. Let's try some more prompts
   - Ask Agent0 to **add a new task**
   - Ask it to **mark an existing task complete**
   - Refresh the Todo0 app — you'll see the changes reflected instantly ![Add and complete tasks in Todo0 Resource App UI](https://developer.okta.com/blog/img/cross-app-access/aYyUzV55sN-800.webp)

## [Behind the scenes: the OAuth 2.0 Identity Assertion Authorization Grant](https://developer.okta.com/blog/2025/09/03/cross-app-access/\#behind-the-scenes-the-oauth-20-identity-assertion-authorization-grant)

**✅ Bob Tables** logs in once with Okta

⏩ **Agent0 (requesting app)** gets an identity assertion from Okta

🔄 Okta vouches for Bob and exchanges that assertion for an access token

👋 **Agent0** uses that token to securely call the **Todo0 (resource app)** API

![Illustration showing secure agent-to-app connections using Okta Cross App Access](https://developer.okta.com/blog/img/cross-app-access/1HE5GsFmFI-1324.svg)

**🎉 Congratulations! You've successfully configured and run the Cross App Access project.**

## [Need help setting up secure cross-domain enterprise agentic application access?](https://developer.okta.com/blog/2025/09/03/cross-app-access/\#need-help-setting-up-secure-cross-domain-enterprise-agentic-application-access)

If you run into any issues while setting up or testing this project, feel free to post your questions on the forum: 👉 [Cross App Access (XAA) — Okta Developer Forum](https://devforum.okta.com/c/cross-app-access-xaa/33)

If you're interested in implementing **Cross App Access (XAA)** in your own application and integrating it into the **[Okta Integration Network (OIN)](https://www.okta.com/integrations/)** — whether as a **requesting app** or a **resource app** — and want to explore how Okta can support your use case, reach out to us at: 📩 **[xaa@okta.com](mailto:xaa@okta.com)**

## [Learn more about Cross App Access, OAuth 2.0, and securing your applications](https://developer.okta.com/blog/2025/09/03/cross-app-access/\#learn-more-about-cross-app-access-oauth-20-and-securing-your-applications)

If this walkthrough helped you understand how Cross App Access works in practice, you might enjoy diving deeper into the standards and conversations shaping it. Here are some resources to continue your journey

- 📘 [Cross App Access Documentation](https://help.okta.com/oie/en-us/content/topics/apps/apps-cross-app-access.htm) – official guides and admin docs to configure and manage Cross App Access in production
- 🎙️ [Developer Podcast on MCP and Cross App Access](https://www.youtube.com/watch?v=qKs4k5Y1x_s) – hear the backstory, use cases, and why this matters for developers
- 📄 [OAuth Identity Assertion Authorization Grant (IETF Draft)](https://datatracker.ietf.org/doc/draft-ietf-oauth-identity-assertion-authz-grant/) – the emerging standard that powers this flow

If you're new to OAuth or want to understand the basics behind secure delegated access, check out these resources:

- [What the Heck is OAuth?](https://developer.okta.com/blog/2017/06/21/what-the-heck-is-oauth)
- [What's the Difference Between OAuth, OpenID Connect, and SAML?](https://www.okta.com/identity-101/whats-the-difference-between-oauth-openid-connect-and-saml/)
- [Secure Your Express App with OAuth 2.0, OIDC, and PKCE](https://developer.okta.com/blog/2025/07/28/express-oauth-pkce)
- [Why You Should Migrate to OAuth 2.0 From Static API Tokens](https://developer.okta.com/blog/2023/09/25/oauth-api-tokens)
- [How to Get Going with the On-Demand SaaS Apps Workshops](https://developer.okta.com/blog/2023/07/27/enterprise-ready-getting-started)

Follow us on [LinkedIn](https://www.linkedin.com/company/oktadev), [Twitter](https://twitter.com/oktadev), and subscribe to our [YouTube](https://www.youtube.com/c/oktadev) channel for more developer content. If you have any questions, please leave a comment below!

Changelog:

- Aug 24, 2026: Added a notice directing readers to the current OIDC resource and requesting app guides, since the Admin Console steps in this post no longer apply. Changes to this article can be viewed in [oktadev/okta-blog#1701](https://github.com/oktadev/okta-blog/pull/1701).
- Jun 30, 2026: Removed the self-service EA enablement steps as Cross App Access is no longer self-service. Updated the section to direct Okta customers to the customer support team and ISVs to [developers@okta.com](mailto:developers@okta.com). Added a note clarifying that this guide covers the OIDC integration path. Changes to this article can be viewed in [oktadev/okta-blog#1663](https://github.com/oktadev/okta-blog/pull/1663).
- May 22, 2026: Updated the Okta Admin Console steps to reflect the new **XAA Resource App** and **XAA Requesting App** App Catalog integrations, and the new required **Issuer URL** and **Client ID** fields. Changes to this article can be viewed in [oktadev/okta-blog#1641](https://github.com/oktadev/okta-blog/pull/1641).

[![Sohail Pathan](https://developer.okta.com/blog/img/avatar/qGi6HMtV9h-48.webp)](https://developer.okta.com/blog/authors/sohail-pathan/) [Sohail Pathan](https://developer.okta.com/blog/authors/sohail-pathan/)
[GitHub Profile](https://github.com/iamspathan "GitHub Profile")[LinkedIn Profile](https://www.linkedin.com/in/iamspathan "LinkedIn Profile")[X Profile](https://twitter.com/iamspathan "X Profile")

Sohail is a Senior Developer Advocate at Okta with roots in mobile app development and hands-on experience designing, building, and publishing APIs. Now, he helps developers secure their apps by turning complex OAuth and API topics into clear, actionable guides. When he's not coding or speaking at conferences, you'll find him on a quest for the perfect plate of biryani.

[Previous post](https://developer.okta.com/blog/2025/08/20/ios-mfa/ "How to Build a Secure iOS App with MFA") [Next post](https://developer.okta.com/blog/2025/09/22/okta-mcp-server/ "Introducing the Okta Open Source MCP Server")

Okta Developer Blog Comment Policy

We welcome relevant and respectful comments. Off-topic comments may be removed.

Build Secure Agent-to-App Connections with Cross App Access (XAA) - Developer Blog Comments - Okta Developer Community[Continue Discussion](https://devforum.okta.com/t/build-secure-agent-to-app-connections-with-cross-app-access-xaa/34277/2) 11 replies[November 2025](https://devforum.okta.com/t/build-secure-agent-to-app-connections-with-cross-app-access-xaa/34277/2 "November 20, 2025, 10:16am")

[![](https://avatars.discourse-cdn.com/v4/letter/d/839c29/45.png)](https://devforum.okta.com/u/dharm.ace)

### [dharm.ace](https://devforum.okta.com/u/dharm.ace)

Hi,

I’ve been experimenting with Cross App Access based on this post and the Agent0/Todo0 sample, and I have a question about the required architecture.

In your example, Okta acts as the IdP and there’s a separate authorization server for the resource app (e.g., the Todo0 authorization server). In my setup I have:

- One Okta org (my “integrator” instance)

- Two apps in that same org:
  - A web UI app that does OIDC login with Okta (Auth Code + PKCE)

  - A backend “resource server” app that I want to call via Cross App Access

My questions are:

1. Can Cross App Access / ID-JAG be used when **both** the requesting app and the resource app live in the **same Okta org**, and the **resource’s authorization server is also that same org** (e.g., via a custom authorization server like `/oauth2/default`)?

2. Or is it a requirement/recommendation that the **resource app have its own authorization server**, separate from the enterprise IdP – like in your sample (a custom auth server) or possibly a **second Okta org** acting as the resource authorization server?

3. If using only a single Okta org is supported, is there a minimal example or configuration you can point to that shows ID-JAG working _without_ a separate non-Okta auth server?


I’m trying to understand the intended deployment model so I can get this working correctly with my own Okta “integrator” instance.

Thanks!

[November 2025](https://devforum.okta.com/t/build-secure-agent-to-app-connections-with-cross-app-access-xaa/34277/4 "November 21, 2025, 12:42pm")

[![](https://sea1.discourse-cdn.com/oktadev/user_avatar/devforum.okta.com/sohail.pathan/45/6116_2.png)](https://devforum.okta.com/u/sohail.pathan)

### [sohail.pathan](https://devforum.okta.com/u/sohail.pathan) Developer Advocacy Team

Hi [@dharm.ace](https://devforum.okta.com/u/dharm.ace), thanks for reaching out, and good to know you’re trying out the sample repo.

Regarding your questions:

**1\. Using Cross App Access / ID-JAG within a single Okta org**

Currently, Cross App Access is designed for scenarios where the requesting app and the resource app are **third-party to each other**, meaning each app has its **own** authorization server in a **separate trust domain**. We currently support the model in the Okta Integration Network where you add third-party apps using Okta SSO for your workforce.

So while both apps can live _as OIDC clients_ inside your same Okta org for login, their **authorization servers should not be the same**. We are actively working on supporting “first-party” / internal-only setups like yours, but that isn’t the intended model yet.

**2\. Whether the resource must have its own authorization server**

Yes – that is the current recommendation.

The requesting app and resource app should each have **their own** authorization server (their own issuer), separate from the enterprise IdP. This separation is what allows ID-JAG to model cross-app trust correctly.

**3\. Minimal example using a single Okta org**

Right now, there is **no minimal example** where both apps use the same Okta custom authorization server as their AS. Your scenario falls under “first-party/internal apps,” which is on the roadmap but not available yet.

[1 reply](https://devforum.okta.com/t/build-secure-agent-to-app-connections-with-cross-app-access-xaa/34277/4)

[December 2025](https://devforum.okta.com/t/build-secure-agent-to-app-connections-with-cross-app-access-xaa/34277/5 "December 5, 2025,  2:50am") [▶ sohail.pathan](https://devforum.okta.com/t/build-secure-agent-to-app-connections-with-cross-app-access-xaa/34277/4)

[![](https://avatars.discourse-cdn.com/v4/letter/i/8edcca/45.png)](https://devforum.okta.com/u/idseclabs)

### [idseclabs](https://devforum.okta.com/u/idseclabs)

Thank you so much for this clarification. There was some confusion in testing the demo and ID-JAG exchange wouldnt work until we explored the [Cross App Access (XAA) for Resource Applications Beta](https://auth0.com/docs/xaa-resource-app) . IDP for Requesting app and Auth Server for Resource app need to be separate trust domain which is where [GitHub - oktadev/okta-cross-app-access-mcp: The repository demonstrates a sophisticated enterprise integration pattern where an AI-enabled agent application (Agent0) can securely access resources from a separate todo management application (Todo0) without requiring users to authenticate separately to each system.](https://github.com/oktadev/okta-cross-app-access-mcp) was not working for us while using same Okta tenant.

Follow up question for the resource Authorization Server. Do the Requesting App IDP and Resource Authorization Server need to be federated as mentioned in this Okta(Requesting App IDP)+Auth0(Resource Auth Server) link [Cross App Access (XAA) for Resource Applications Beta](https://auth0.com/docs/xaa-resource-app)?

Regarding the roadmap, is there a page that could be referred to identify the following:

1. Support for first-party/internal apps
2. Remove dependency on using TODO0 and Agent0 as OIN apps and allowing the ability to Manage connections using Custom OIDC Apps?

[December 2025](https://devforum.okta.com/t/build-secure-agent-to-app-connections-with-cross-app-access-xaa/34277/6 "December 10, 2025, 11:10am")

[![](https://avatars.discourse-cdn.com/v4/letter/n/a183cd/45.png)](https://devforum.okta.com/u/nithishsubramani)

### [nithishsubramani](https://devforum.okta.com/u/nithishsubramani)

I need help understanding whether, in an OAuth/OIDC federation scenario where Okta is the Identity Provider and issues an ID-JAG token (ID-JAG) that the requesting app uses to obtain an access token from a downstream resource server, it is possible to customize the `sub` claim or include additional claims such as `email` or other profile attributes in the ID-JAG token. Specifically, I’d like to know if Okta supports overriding the `sub` value (e.g., mapping it to email, username, or a custom attribute) and whether additional claims can be injected into the ID-JAG token for this delegated authorization flow, and if so, where this should be configured—within the OIDC application settings.

[1 reply](https://devforum.okta.com/t/build-secure-agent-to-app-connections-with-cross-app-access-xaa/34277/6)

[December 2025](https://devforum.okta.com/t/build-secure-agent-to-app-connections-with-cross-app-access-xaa/34277/7 "December 11, 2025, 10:21am") [▶ nithishsubramani](https://devforum.okta.com/t/build-secure-agent-to-app-connections-with-cross-app-access-xaa/34277/6)

[![](https://sea1.discourse-cdn.com/oktadev/user_avatar/devforum.okta.com/sohail.pathan/45/6116_2.png)](https://devforum.okta.com/u/sohail.pathan)

### [sohail.pathan](https://devforum.okta.com/u/sohail.pathan) Developer Advocacy Team

[@nithishsubramani](https://devforum.okta.com/u/nithishsubramani) Answer to your questions:

- **The `sub` claim can’t be customized.** ID-JAG tokens always use the Okta user’s immutable Okta ID, and there’s no option to remap it to email, username, or any other attribute.

- **Additional claims (like email or profile fields) can’t be added** to the ID-JAG token right now. Custom admin-defined claims aren’t supported for this flow.


One thing to keep in mind is that the downstream OIDC step also relies on this same `sub` value, so both the requesting app and the resource server are expected to use that Okta ID as the stable identifier.

[1 reply](https://devforum.okta.com/t/build-secure-agent-to-app-connections-with-cross-app-access-xaa/34277/7)

[January 27](https://devforum.okta.com/t/build-secure-agent-to-app-connections-with-cross-app-access-xaa/34277/8 "January 27, 2026,  9:06pm")

[![](https://avatars.discourse-cdn.com/v4/letter/a/47e85d/45.png)](https://devforum.okta.com/u/abodell)

### [abodell](https://devforum.okta.com/u/abodell)

[@sohail.pathan](https://devforum.okta.com/u/sohail.pathan) Hi, appreciate the demo. I’m facing a few issues when using this. I’ve been trying to set up XAA between an Chat Application and one of our SaaS applications at my company. In order to do this, I believe I would need to be able to exchange the ID-JAG token for an access token from the SaaS application. However, I’ve noticed the ID-JAG token always has the audience listed as localhost:5001 and the client\_id is wiki0-at-todo0. I was wondering if there is a way to change this so the audience would be the token\_url from the SaaS application? Maybe I am having a misunderstanding of the implementation or these are just preconfigured settings that can’t be changed yet with the beta version.

[February 11](https://devforum.okta.com/t/build-secure-agent-to-app-connections-with-cross-app-access-xaa/34277/9 "February 11, 2026,  1:16pm")

[![](https://avatars.discourse-cdn.com/v4/letter/n/a183cd/45.png)](https://devforum.okta.com/u/nithishsubramani)

### [nithishsubramani](https://devforum.okta.com/u/nithishsubramani)

Hi Team,

I’m implementing the Identity Assertion JWT Authorization Grant (ID-JAG) flow for cross-app access.

Scenario:

- Two resource applications (Product A and Product B) are published as separate OIN apps
- Both share the same Authorization Server issuer URL: [https://api.vendor.example.com/](https://api.vendor.example.com/)
- A requesting app (Agent App) needs to get an ID-JAG to access either Product A or Product B

Token Exchange Request:

```auto
grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&requested_token_type=urn:ietf:params:oauth:token-type:id-jag
&audience=https://api.vendor.example.com/
&scope=product_a.read
&subject_token=<id_token>
&subject_token_type=urn:ietf:params:oauth:token-type:id_token
```

How does Okta identify which app (Product A or Product B) to apply the access policy for when:

- The audience parameter is the same for both apps
- The resource parameter (RFC 8707) is not supported in OIN submission
- Only scope values differ between the requests

I understand that the resource parameter (RFC 8707) can be used to differentiate between apps that share the same issuer URL.

Is there a provision to configure a resource identifier when publishing an OIN app so that Okta can use it to identify which app’s policy to apply during ID-JAG issuance?

What is the mechanism Okta uses to differentiate between these two apps during ID-JAG issuance?

Thanks!

[1 reply](https://devforum.okta.com/t/build-secure-agent-to-app-connections-with-cross-app-access-xaa/34277/9)

[February 12](https://devforum.okta.com/t/build-secure-agent-to-app-connections-with-cross-app-access-xaa/34277/10 "February 12, 2026,  4:08am")

[![](https://avatars.discourse-cdn.com/v4/letter/n/a183cd/45.png)](https://devforum.okta.com/u/nithishsubramani)

### [nithishsubramani](https://devforum.okta.com/u/nithishsubramani)

In the ID-JAG RFC, it is mentioned that the IdP can include the email attribute as a claim. Could you please confirm if this is the expected approach?

![](https://static.ietf.org/dt/12.58.0/ietf/images/ietf-logo-nor-32.png)[IETF Datatracker](https://datatracker.ietf.org/doc/draft-ietf-oauth-identity-assertion-authz-grant/#:~:text=sub%20should%20be%20an%20opaque%20ID%2C%20as%20iss%2Bsub%20is%20unique.%20%20The%20IdP%20might)

![](https://static.ietf.org/dt/12.58.0/ietf/images/ietf-logo-card.png)

### [Identity Assertion JWT Authorization Grant](https://datatracker.ietf.org/doc/draft-ietf-oauth-identity-assertion-authz-grant/\#:~:text=sub%20should%20be%20an%20opaque%20ID%2C%20as%20iss%2Bsub%20is%20unique.%20%20The%20IdP%20might)

This specification provides a mechanism for an application to use an identity assertion to obtain an access token for a third-party API by coordinating through a common enterprise identity provider using Token Exchange \[RFC8693\] and JWT Profile for...

[![image](https://global.discourse-cdn.com/oktadev/optimized/2X/6/641d1b4c4ac12c90127923c45c77f1d108ae6212_2_690x182.png)](https://global.discourse-cdn.com/oktadev/original/2X/6/641d1b4c4ac12c90127923c45c77f1d108ae6212.png "image")

[1 reply](https://devforum.okta.com/t/build-secure-agent-to-app-connections-with-cross-app-access-xaa/34277/10)

[March 11](https://devforum.okta.com/t/build-secure-agent-to-app-connections-with-cross-app-access-xaa/34277/11 "March 11, 2026,  7:49pm") [▶ nithishsubramani](https://devforum.okta.com/t/build-secure-agent-to-app-connections-with-cross-app-access-xaa/34277/9)

[![](https://sea1.discourse-cdn.com/oktadev/user_avatar/devforum.okta.com/sohail.pathan/45/6116_2.png)](https://devforum.okta.com/u/sohail.pathan)

### [sohail.pathan](https://devforum.okta.com/u/sohail.pathan) Developer Advocacy Team

[@nithishsubramani](https://devforum.okta.com/u/nithishsubramani)

Apologies for the late response on this!

First, a clarifying point — if both Product A and Product B share the same Authorization Server issuer URL, they don’t necessarily need to be two separate OIN apps. A single OIN app can

represent both products under one issuer. Is there a specific reason they need to be published as two separate apps?

That said, to answer your questions:

**How does Okta differentiate between apps during ID-JAG issuance?**

Okta uses a combination of:

- App assignments - which apps the user is assigned to
- SSO session - which app the user has actually signed into
- Manage Connections - which resource app is linked to the requesting app (Agent App)

These together determine which app’s policy applies when issuing the ID-JAG.

On the `resource` parameter:

Currently, the ID-JAG flow passes through the resource parameter as-is. If you include it in your token exchange request, that value will appear as a claim in the issued ID-JAG. Your

Authorization Server can then read this claim to determine which product (A or B) the ID-JAG is intended for and apply the right access rules on your end.

So even though the resource parameter (RFC 8707) isn’t part of OIN submission today, you can still use scopes (as you’re already doing) and the resource parameter in the token exchange request to signal which product the access is for, and handle the differentiation at your Authorization Server.

Thanks!

[March 11](https://devforum.okta.com/t/build-secure-agent-to-app-connections-with-cross-app-access-xaa/34277/12 "March 11, 2026,  7:51pm") [▶ nithishsubramani](https://devforum.okta.com/t/build-secure-agent-to-app-connections-with-cross-app-access-xaa/34277/10)

[![](https://sea1.discourse-cdn.com/oktadev/user_avatar/devforum.okta.com/sohail.pathan/45/6116_2.png)](https://devforum.okta.com/u/sohail.pathan)

### [sohail.pathan](https://devforum.okta.com/u/sohail.pathan) Developer Advocacy Team

Okta doesn’t include email in the sub claim. This is a very recent addition to the spec and it is optional, we have not added it to the `sub` claim.

[1 reply](https://devforum.okta.com/t/build-secure-agent-to-app-connections-with-cross-app-access-xaa/34277/12)

[March 17](https://devforum.okta.com/t/build-secure-agent-to-app-connections-with-cross-app-access-xaa/34277/13 "March 17, 2026,  4:02am") [▶ sohail.pathan](https://devforum.okta.com/t/build-secure-agent-to-app-connections-with-cross-app-access-xaa/34277/12)

[![](https://avatars.discourse-cdn.com/v4/letter/n/a183cd/45.png)](https://devforum.okta.com/u/nithishsubramani)

### [nithishsubramani](https://devforum.okta.com/u/nithishsubramani)

but in our case we only provide an **MCP server and act as an MCP Gateway**, and we **do not have a UI**. Because of this, we will **not support user SSO flows**, since there is no user interface where users can perform SSO authentication.

Given this architecture, I have a question regarding **user identification using IDJAG**.

Currently, **IDJAG does not contain the email identifier**, even though the **IDJAG specification mentions that the email claim can be optional**. Because of this, it is unclear how we should reliably identify the user.

Questions:

1. In this scenario, **how should we obtain the user details when we only receive IDJAG?**

2. Should we **call the `/userinfo` endpoint with IDJAG Jwt Token** to retrieve additional user attributes such as email?

3. What is the **best architectural approach to solve user identification for a service that acts only as an MCP Gateway and does not support user SSO?**


Could you please clarify the **recommended way to approach this problem**?

[![](https://global.discourse-cdn.com/oktadev/original/1X/0c6402653dfb70edc661d4976a43a46f33e5e919.png)](https://devforum.okta.com/)[Continue Discussion](https://devforum.okta.com/t/build-secure-agent-to-app-connections-with-cross-app-access-xaa/34277/13)

[Okta Developer](https://developer.okta.com/)

## Questions?

Ask us on the [forum](https://devforum.okta.com/).

## Connect with us

- [Okta Developers on GitHub](https://github.com/oktadev)
- [Okta Developers on X](https://x.com/OktaDev)
- [Okta Developers on YouTube](https://www.youtube.com/oktadev)
- [Okta Developers on LinkedIn](https://www.linkedin.com/company/oktadev/)
- [Okta Developer Forum](https://devforum.okta.com/)
- [Okta Developer blog RSS feed](https://developer.okta.com/feed.xml)

## Contact & Legal

- [Contact our team](https://www.okta.com/contact/)
- [Contact sales](https://www.okta.com/contact-sales/)
- [Integrator Free Plan terms](https://developer.okta.com/terms/)
- [Site terms](https://www.okta.com/terms-of-service/)
- [Privacy policy](https://www.okta.com/privacy-policy/)
- [Copyright & trademarks](https://developer.okta.com/copyright/)

## More info

- [All Communities](https://developer.okta.com/blog/communities/)
- [All Tags](https://developer.okta.com/blog/tags/)
- [All Authors](https://developer.okta.com/blog/authors/)
- [Pricing](https://developer.okta.com/pricing/)
- [Integrate with Okta](https://developer.okta.com/okta-integration-network/)
- [Change Log](https://developer.okta.com/docs/change-log/)
- [3rd-party notes](https://developer.okta.com/3rd_party_notices/)
- [Auth0(opens in a new tab)](https://developer.auth0.com/)

- [Okta.com](https://www.okta.com/)
Products, case studies, resources (opens in a new tab)

- [Support center](https://help.okta.com/)
Knowledgebase, roadmaps, and more (opens in a new tab)

- [Trust](https://trust.okta.com/)
System status, security, compliance (opens in a new tab)


* * *

Copyright © 2026 Okta. All rights reserved.