# OAuth2 Server Implementation Guide

In this lab, you will build an OAuth2 server step by step using Node.js and Express. This server will interact with Google's OAuth2 API to authenticate users.

## Step 1: Initial Setup

add the following basic setup code inside server.js:

```js
const express = require('express');
const cookieParser = require('cookie-parser');
const secret = require('./client_secret.json'); // ensure you have this file from Google's API console
const app = express();

app.use(cookieParser());

const PORT = process.env.PORT || 3000;
const CLIENT_ID = secret.client_id;
const CLIENT_SECRET = secret.client_secret;
const REDIRECT_URI = 'http://localhost:3000/auth/google/callback';

app.listen(PORT, () => {
    console.log('Server is running on http://localhost:' + PORT);
});
```

### Explanation

* **Express.js** sets up our server.
* **cookie-parse**r is middleware to parse cookies attached to the client request object.
* **querystring** helps in encoding and decoding query strings.
* **Client ID and Client Secret** are essential for securing OAuth2 flow and identifying the application to the Google OAuth2 server.

Running this code will crash the server becuase we are missing the `CLIENT_ID` and `CLIENT_SECRET`

## Step 2: Create A project At Google Developer Console

1. Navigate to the Google Developer Console (https://console.developers.google.com )
2. Create new project.
3. Go to "Credentials" -> "Create Credentials" -> "OAuth client ID".
4. Click on your OAuth 2.0 Client IDs.
5. Under "Authorized JavaScript origins", click "ADD URI" and enter your domain (e.g., http://localhost:3000 ).
6. Under "Authorized redirect URIs", click "ADD URI" and enter your redirect URI (e.g., http://localhost:3000/auth/google/callback). Note: if you deploy your app to a domain (e.g., mySite.com), you need to go add that domain as well.
7. Click "Create" to finalize your settings.
8. After creating the credentials, you'll have the option to download a JSON file containing your OAuth 2.0 client credentials. Click on the download icon to download the credentials.
9. Store the JSON content inside client_secret.json file.

## Step 3: Redirect to Google's OAuth 2.0 Server

When the user click on login in with google, we call this endpoint.

Add login with google button inside public\index.html

```html
    <button id="loginButton">Login with Google</button>
```

Now, add the js frontend handler to handle user click. add this code inside public\index.js

```js
document.getElementById('loginButton').addEventListener('click', () => {
    window.location.href = '/auth/google';
});
```

add this code in server.js to handle the auth request

```js
app.get('/auth/google', (req, res) => {
    const authorizationUrl = 'https://accounts.google.com/o/oauth2/v2/auth';
    const params = {
        client_id: CLIENT_ID,
        redirect_uri: REDIRECT_URI,
        response_type: 'code',
        scope: 'openid email profile',
        access_type: 'online'
    };
    res.redirect(`${authorizationUrl}?${querystring.stringify(params)}`);
});
```

### Explanation

* Authorization URL: This is the endpoint of Google's OAuth2 server.
* client_id & redirect_uri: Identify your application and where to send the user after authorization.
* response_type: Set to 'code' to indicate that you are initiating the authorization code flow.
* scope: Specifies the level of access that the application is requesting.
* access_type: Indicates whether your application needs to refresh tokens.

## Step 4: Handling the OAuth 2.0 Server Response

Handle the response from Google's OAuth server. This involves exchanging the authorization code received for an access token.
Add this code in server.js

```js
app.get('/auth/google/callback', async (req, res) => {
    const code = req.query.code;
    if (!code) {
        return res.status(400).send('Authorization code is missing');
    }
    try {
        const tokenResponse = await fetch('https://oauth2.googleapis.com/token', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/x-www-form-urlencoded',
            },
            body: querystring.stringify({
                code,
                client_id: CLIENT_ID,
                client_secret: CLIENT_SECRET,
                redirect_uri: REDIRECT_URI,
                grant_type: 'authorization_code'
            })
        });

        if (!tokenResponse.ok) {
            throw new Error(JSON.stringify(await tokenResponse.json()));
        }

        const tokenData = await tokenResponse.json();
        const accessToken = tokenData.access_token;

        if (!accessToken) {
            throw new Error('Access token is missing in the response');
        }

        res.cookie('token', accessToken, { httpOnly: true });
        res.redirect('/dashboard');
    } catch (error) {
        console.error(error);
        res.status(500).send('Internal Server Error');
    }
});
```

### Explanation

* Authorization code: A temporary code that the client will exchange for an access token.
* Access token: A token that grants temporary access to the user's resources.
* res.cookie('token', accessToken, { httpOnly: true }) : To save the token as Cookie in the browser.
* res.redirect('/dashboard') to redirect user to the dashboard page.

## Step 5: User Dashboard

Show a user dashboard that displays user information obtained through Google's API using the access token.

```js
app.get('/dashboard', async (req, res) => {
    if (!req.cookies.token) {
        return res.status(401).send('Unauthorized');
    }
    const accessToken = req.cookies.token;

    const userInfoResponse = await fetch('https://www.googleapis.com/oauth2/v1/userinfo', {
        headers: { Authorization: `Bearer ${accessToken}` }
    });

    if (!userInfoResponse.ok) {
        throw new Error('Failed to fetch user info');
    }

    const userData = await userInfoResponse.json();

    res.send(`
        <h1>Welcome to the Dashboard</h1>
        <h2>Email: ${userData.email}</h2>
        <h2>Name: ${userData.name}</h2>
        <img src="${userData.picture}" alt="Profile Picture" />
    `);
});
```

### Explanation

* req.cookies.token checks if the cookie exists
* Access Token: Used to securely call Google's APIs to retrieve user information.
* User Info Endpoint: Fetches user details like email and profile picture, which are displayed on the dashboard.

## Step 6: Run the Server and Inspect Behavior

To start your server and test the implementation:

1. **Install dependencies**: Ensure all required packages are installed:

   ```bash
   npm install
   ```
2. **Start the server:**: If you have set up a script in your package.json to start the server using npm run dev, use that command. If not, you can directly run::

   ```bash
    node server.js
   ```
3. **Testing:**: Open a web browser and navigate to http://localhost:3000  and click on login with google. This will start the OAuth2 authentication flow.
4. **Inspect cookies**:

* In your browser, open the developer tools (usually accessible by pressing F12 or right-clicking on the page and selecting "Inspect").
* Go to the 'Application' tab and look at the cookies under the 'Storage' section on the left sidebar. Find the cookie named 'token'.

5. **Observations to make**:

* Check the cookie value: Note the value of the token cookie. It should be a long string, which is the access token received from Google.
* Refresh the page: Refresh your browser while on the /dashboard route. Observe what happens to the cookie and the page content. Typically, the page should reload without asking for login again, indicating that the session is being maintained via the cookie.
* Change the value of the cookie and observe what this change causes.
