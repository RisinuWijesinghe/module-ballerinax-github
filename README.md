# Ballerina GitHub Connector

[![Build](https://github.com/ballerina-platform/module-ballerinax-github/actions/workflows/ci.yml/badge.svg)](https://github.com/ballerina-platform/module-ballerinax-github/actions/workflows/ci.yml)
[![codecov](https://codecov.io/gh/ballerina-platform/module-ballerinax-github/branch/master/graph/badge.svg)](https://codecov.io/gh/ballerina-platform/module-ballerinax-github)
[![GitHub Last Commit](https://img.shields.io/github/last-commit/ballerina-platform/module-ballerinax-github.svg)](https://github.com/ballerina-platform/module-ballerinax-github/commits/master)
[![GitHub Issues](https://img.shields.io/github/issues/ballerina-platform/ballerina-library/module/github.svg?label=Open%20Issues)](https://github.com/ballerina-platform/ballerina-library/labels/module%2Fgithub)

[GitHub](https://github.com/) is a widely used platform for version control and collaboration, allowing developers to work together on projects from anywhere. It hosts a vast array of both open-source and private projects, providing a suite of development tools for collaborative software development.

This Ballerina connector is designed to interface with [GitHub's REST API (version 2022-11-28)](https://docs.github.com/en/rest?apiVersion=2022-11-28), facilitating programmatic access to GitHub's services. It enables developers to automate tasks, manage repositories, issues, pull requests, and more, directly from Ballerina applications.

## Setup guide

To use the GitHub Connector in Ballerina you need a GitHub account. If you already have one, you can integrate the connector with your existing account. If not, you can create a new GitHub account by visiting [GitHub's Sign Up page](https://github.com/) and following the registration process.

The connector supports two ways of authenticating with the GitHub REST API:

| Authentication | When to use |
| -------------- | ----------- |
| **Personal Access Token (PAT)** | A static token bound to your own user account. Simplest option, suited for scripts and single-user automation. Also use this for an OAuth App token, which does not expire. |
| **OAuth2 refresh token grant** | A GitHub App user access token, which expires after 8 hours and is renewed automatically by the connector. Suited for applications acting on behalf of other users. Requires a GitHub App with expiring user authorization tokens enabled — OAuth Apps do not issue refresh tokens. |

### Option 1: Personal Access Token (PAT)

#### Step 1: Access GitHub Settings

1. Once logged in, click on the profile picture in the top-right corner of the page.
2. Select **Settings** from the dropdown menu.

#### Step 2: Navigate to Developer Settings

1. Scroll down in the sidebar on the left side of the settings page.
2. click on **Developer settings** located near the bottom.

#### Step 3: Go to Personal Access Tokens

1. Inside Developer Settings find and click on **Personal access tokens**.

    <img src=https://raw.githubusercontent.com/ballerina-platform/module-ballerinax-github/master/docs/setup/resources/1-developer-settings.png alt="GitHub Developer Settings" width="50%">

#### Step 4: Generate a New Token

1. Click on the **Generate new token** button (you might be asked to enter your password again for security purposes).

#### Step 5: Configure & Generate the Token

 - **Note**: Give your token a descriptive name so you can remember its purpose
 - **Expiration**: Select the duration before the token expires (e.g., 30 days, 60 days, 90 days, custom, or no expiration).
 - **Select Scopes**: Scopes control access for the token. Choose what you need the token for (e.g., repo access, user data access). For typical repository operations, selecting `repo` is often sufficient.

    <img src=https://raw.githubusercontent.com/ballerina-platform/module-ballerinax-github/master/docs/setup/resources/2-generate-token.png alt="Generate new PAT" width="50%">

### Option 2: GitHub App with the OAuth2 refresh token grant

#### Step 1: Register a GitHub App

1. Go to **Settings** → **Developer settings** → **GitHub Apps** and click **New GitHub App**.
2. Fill in the **GitHub App name**, **Homepage URL**, and the **Callback URL** that GitHub redirects to after a user authorizes the app.
3. Under **Identifying and authorizing users**, make sure **Expire user authorization tokens** is enabled. GitHub only issues refresh tokens when this option is on; without it, user access tokens never expire and there is nothing to refresh.

#### Step 2: Configure the app permissions

Under **Permissions**, grant the repository, organization, and account permissions your integration needs. A GitHub App user access token is limited to these permissions — GitHub Apps ignore the OAuth `scope` parameter, so the `scopes` field of the connector configuration is not used for them.

#### Step 3: Collect the client credentials

On the app's settings page, note the **Client ID** and click **Generate a new client secret** to obtain the **Client secret**.

#### Step 4: Install the app

Click **Install App** and install it on the user account or organization whose resources you want to access.

#### Step 5: Obtain a refresh token via the web application flow

1. Direct the user to the authorization page, replacing the placeholders with your own values. Generate `<RANDOM_STRING>` fresh for every authorization attempt and store it against the user's session — it is the CSRF guard for this flow:

    ```
    https://github.com/login/oauth/authorize?client_id=<CLIENT_ID>&redirect_uri=<CALLBACK_URL>&state=<RANDOM_STRING>
    ```

2. After the user approves, GitHub redirects to the callback URL with a temporary `code` query parameter and the `state` you sent.

    Compare the returned `state` against the value you stored and **abort without exchanging the code** if it is absent or does not match. Skipping this check lets an attacker feed their own `code` to your callback and bind the victim's session to an account the attacker controls. Discard the stored value once used, so a `state` cannot be replayed.

3. Exchange that code for tokens:

    ```bash
    curl -X POST https://github.com/login/oauth/access_token \
      -H "Accept: application/json" \
      -d "client_id=<CLIENT_ID>" \
      -d "client_secret=<CLIENT_SECRET>" \
      -d "code=<CODE>" \
      -d "redirect_uri=<CALLBACK_URL>"
    ```

    The response contains an `access_token` valid for 8 hours and a `refresh_token` valid for 6 months:

    ```json
    {
      "access_token": "ghu_...",
      "expires_in": 28800,
      "refresh_token": "ghr_...",
      "refresh_token_expires_in": 15897600,
      "token_type": "bearer"
    }
    ```

4. Keep the `client_id`, `client_secret`, and `refresh_token` — these are the three values the connector needs. The connector exchanges the refresh token for an access token on the first request and renews it automatically whenever it expires.

> **Note:** GitHub rotates refresh tokens. Every renewal returns a new refresh token and invalidates the previous one. The connector holds the new token in memory for the lifetime of the `github:Client` value; `ballerina/oauth2` provides no API to read it back, so it cannot be persisted. A process that restarts after a renewal therefore needs to be reauthorized with a freshly obtained refresh token.

## Quickstart

To use the `GitHub` connector in your Ballerina application, modify the `.bal` file as follows:

### Step 1: Import the connector

Import the `ballerinax/github` package into your Ballerina project.

```ballerina
import ballerinax/github;
```

### Step 2: Instantiate a new connector

Create a `github:ConnectionConfig` with the credentials obtained in the setup guide and initialize the connector with it.

If you are authenticating with a **Personal Access Token**, provide the token directly:

```ballerina
github:ConnectionConfig gitHubConfig = {
    auth: {
        token: authToken
    }
};
github:Client github = check new (gitHubConfig);
```

If you are authenticating a **GitHub App with the OAuth2 refresh token grant**, provide the client credentials and the refresh token instead:

```ballerina
github:ConnectionConfig gitHubConfig = {
    auth: {
        clientId,
        clientSecret,
        refreshToken
    }
};
github:Client github = check new (gitHubConfig);
```

The `refreshUrl` defaults to `https://github.com/login/oauth/access_token`, so it only needs to be set when you target a GitHub Enterprise Server instance:

```ballerina
github:ConnectionConfig gitHubConfig = {
    auth: {
        clientId,
        clientSecret,
        refreshToken,
        refreshUrl: "https://github.example.com/login/oauth/access_token"
    }
};
github:Client github = check new (gitHubConfig, "https://github.example.com/api/v3");
```

### Step 3: Invoke the connector operation

Now, utilize the available connector operations.

#### Get Private Repositories of Authenticated User

```ballerina
github:Repository[] userRepos = check github->/user/repos(visibility = "private", 'type = ());
```

#### Create a Private Repository

```ballerina
github:UserReposBody body = {
    name: "New Test Repo Name",
    'private: true,
    description: "New Test Repo Description"
};
github:Repository createdRepo = check github->/user/repos.post(body);
```

## Examples

The `GitHub` connector provides practical examples illustrating usage in various scenarios. Explore these [examples](https://github.com/ballerina-platform/module-ballerinax-github/tree/master/examples), covering use cases like initializing a new project, creating issues, and managing pull requests.

1. [Initialize a New GitHub Project](https://github.com/ballerina-platform/module-ballerinax-github/tree/master/examples/initialize-new-project) - Create a new repository on GitHub, initialize it with a README file, and add collaborators to the repository.

2. [Create and Assign an Issue in GitHub](https://github.com/ballerina-platform/module-ballerinax-github/tree/master/examples/create-and-assign-issue) - Create a new issue on GitHub, assign it to a specific user, and add labels.

3. [Create and Manage a PullRequest in GitHub](https://github.com/ballerina-platform/module-ballerinax-github/tree/master/examples/create-and-manage-pull-request) - Create a pull request on GitHub, and request changes as necessary.

4. [Star Ballerina-Platform Repositories](https://github.com/ballerina-platform/module-ballerinax-github/tree/master/examples/star-ballerina-repositories) - Fetch all repositories under the `ballerina-platform` organization on GitHub and star each of them

## Issues and projects

The **Issues** and **Projects** tabs are disabled for this repository as this is part of the Ballerina library. To report bugs, request new features, start new discussions, view project boards, etc., visit the Ballerina library [parent repository](https://github.com/ballerina-platform/ballerina-library).

This repository only contains the source code for the package.

## Build from the source

### Prerequisites

1. Download and install Java SE Development Kit (JDK) version 17. You can download it from either of the following sources:

   * [Oracle JDK](https://www.oracle.com/java/technologies/downloads/)
   * [OpenJDK](https://adoptium.net/)

    > **Note:** After installation, remember to set the `JAVA_HOME` environment variable to the directory where JDK was installed.

2. Download and install [Ballerina Swan Lake](https://ballerina.io/).

3. Download and install [Docker](https://www.docker.com/get-started).

    > **Note**: Ensure that the Docker daemon is running before executing any tests.

### Build options

Execute the commands below to build from the source.

1. To build the package:

   ```bash
   ./gradlew clean build
   ```

2. To run the tests:

   ```bash
   ./gradlew clean test
   ```

3. To build the without the tests:

   ```bash
   ./gradlew clean build -x test
   ```

4. To run tests against different environment:

   ```bash
   ./gradlew clean test -Pgroups=<Comma separated groups/test cases>
   ```

5. To debug package with a remote debugger:

   ```bash
   ./gradlew clean build -Pdebug=<port>
   ```

6. To debug with the Ballerina language:

   ```bash
   ./gradlew clean build -PbalJavaDebug=<port>
   ```

7. Publish the generated artifacts to the local Ballerina Central repository:

    ```bash
    ./gradlew clean build -PpublishToLocalCentral=true
    ```

8. Publish the generated artifacts to the Ballerina Central repository:

   ```bash
   ./gradlew clean build -PpublishToCentral=true
   ```

## Contribute to Ballerina

As an open-source project, Ballerina welcomes contributions from the community.

For more information, go to the [contribution guidelines](https://github.com/ballerina-platform/ballerina-lang/blob/master/CONTRIBUTING.md).

## Code of conduct

All the contributors are encouraged to read the [Ballerina Code of Conduct](https://ballerina.io/code-of-conduct).

## Useful links

* For more information go to the [`github` package](https://lib.ballerina.io/ballerinax/github/latest).
* For example demonstrations of the usage, go to [Ballerina By Examples](https://ballerina.io/learn/by-example/).
* Chat live with us via our [Discord server](https://discord.gg/ballerinalang).
* Post all technical questions on Stack Overflow with the [#ballerina](https://stackoverflow.com/questions/tagged/ballerina) tag.
