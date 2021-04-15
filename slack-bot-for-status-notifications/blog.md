# Insert Title Here

## TL;DR

If you are looking for a way to send messages to a Slack channel look no further than their webhooks. But if you need to send those message from a client side app running in a browser you should not use them. Webhook hooks are POST requests and will fail CORS pre-flight checks done by the browser. You could hack it by using a `'Content-type': 'application/x-www-form-urlencoded'` header which skips the check and POST's as a "simple" request. Please don't do this, CORS is there for a reason. Instead you'll need to build a Slack bot that acts as middleware between your Slack channels and your client app messaging them. Conveniently Slack has a node, python & Java SDK to do just that.

I'll be showcasing Node. It's an Express like framework that gives you the full power of Slacks web-api and app SDK. Using it to message channels from client apps is as simple as hitting an endpoint. Slacks block-kit-builder makes it straight forward to send rich well structured messages. You can also create slash commands for slack-to-bot communication, giving your bot the full power of your companies API. I'll show you how to deploy the bot using cloud function. We use GCP but in resources below theirs a link to Slacks curated list of deployment options for alternate providers.

This blog takes a look at why you shouldn't use webhooks in a client, even though you can. How to setup and build a Node slack bot that receives requests from a client app and posts to any Slack channel. In addition to Slack's Bolt.js SDK I'll be using Typescript, Express and Node. I'll also show you how we deploy it to Google cloud functions. I hope to follow this post with two others, 1) sending threaded slack messages from a client app to slack via our bot, 2) using our bot to communicate with your companies API from slack.

---

## What are we solving and why bother

At the company I work at we needed to receive notification from a semi-internal app we use for uploading drone imagery to our platform. An "upload" in this case consists of any number of images from a single flight, typically 100s - 1000s of images. We wanted notification to track the start, progress and completion of an upload and more importantly and failures. Users can also pause and resume an upload if their internet connection is week.

We looked at three options for notifications, Sentry, Slack and Google Analytics. We settled with Slack for x TK reasons 1) the primary target for the notification are non-technical people, 2)  implementation looked straight forward using webhooks, a false assumption at best, and 3) we wanted the results to be easily accessed by anyone in the company. Sentry was to technical leaning and Google Analytics is to out of the way in or company. TK

## Webhooks for messaging a Slack channel and why you should always test your assumptions

Slack has a TK (number of ways to communicate with slack) ways to interface with it's services. One of the simplest is channel scoped webhooks. All you need is a Slack app (creating one is well documented, see the resources) installed in the channel you want to message. Generate the incoming webhook and post to it with a JSON payload. Done! All I had to do was use `fetch` from my uploader app and post to the channel as event needed. Not so easy...

```bash
Access to XMLHttpRequest at 
'WEBHOOK_ADDRESS' 
from origin 'https://my.uploaderapp.com' has been blocked by 
CORS policy: Response to preflight request doesn't pass access control check: 
It does not have HTTP ok status.
```

This stumped me for an embarrassingly long time. I thought up numerous reasons why it wouldn't work:

- was it `https` in my local dev env - nope!
- maybe it was my local env? Deployed it and still no fix - damn!

I thought I'd try Slacks `@slack/web-api` Node package. When that didn't work, because it's only supported on the server, the penny slowly started to drop. But I didn't want to give up yet, I'd invested so much time trying to get my request to work...! Some deep googling eventually unsurfaced a content-type header that does not need a preflight request which was causing my CORS error.

```json
'Content-type': 'application/x-www-form-urlencoded'
```

Further research indicates it's common practice for API's to accept this content type. We didn't like the idea of using this to get around a CORs issue. We also don't have any control over Slacks services so there is no guarantee it will be supported in the future. Further more using the webhook this way did not feel like the idiomatic. There where fewer than a handful articles on using a webhook from a client app. And nothing official in the docs.

It was clear that we should instead build our own Slack bot and use it as middleware for notifications.

## Building a Slack bot using Typescript, Node and Bolt

TK: can this be structured better as a list?

Slack has a javascript framework called Bolt for building bots. It provides most of what we need to communicate with Slack and to receive requests from the internet. Its built on top of Express so if you're familiar with it you'll feel right at home.

We deploy our bot to Google Cloud functions which means out build needs to be ES5. My preference is to use Typescript because the Slack SDK's are also written in it, type safe code is awesome and the compiler is perfectly suitable for our requirements TK (wishy washy).

### 1. Scaffolding a Node app and setting up a development environment

Starting with an empty git repo and after I've initialised with yarn I install all these dependencies:

```bash
yarn install -D typescript nodemon tsc-watch @types/node @types/node-fetch
```

Typescript for compiling code to ES5. `nodemon` for restarting the node server when a file changes (think HMR for node apps). `tsc-watch` for running both together while using Typescript in watch mode.

I then initialised a Typescript project with `nps tsx --init`. I like doing it this way because the `.tsconfig.json` file it produces has most of the properties with comments included (all commented out except a handful). I then whittled down the config options to these:

```json
{
  "include": ["./src/**/*"],
  "exclude": ["node_modules"],
  "compilerOptions": {
    /* Visit https://aka.ms/tsconfig.json to read more about this file */

    /* Basic Options */
    "target": "ES2020" ,
    "module": "commonjs" , 
    "outDir": "./dist",
    "rootDir": "./src",

    /* Strict Type-Checking Options */
    "strict": true /* Enable all strict type-checking options. */,
    "types": [
      "node"
    ],
    "esModuleInterop": true,
     /* Advanced Options */
    "skipLibCheck": true /* Skip type checking of declaration files. */,
  }
}
```

The compiled files will go into the `dist` folder. Both for development and then when building to deploy. We need `commonjs` modules to satisfy the Google Cloud Functions requirement. The output code can be whatever you like. I'm choosing to use a `node>=14.1` runtime so I using a target of `ES2020` will be fine. Using a more modern version will increase the [speed of build](https://stackoverflow.com/questions/55742129/does-changing-typescript-target-affect-compilation-performance#55742488) too.

Typescript is ready to go. I created a simple function in `src/index.ts` to make sure it was working before I carried on.

I'll cover more details below but our app will run on a node server using the Bolt framework. While developing it will be great to have the server restart itself each time I make a change change to a file. We'll serve the app from the dist folder because Node does not support Typescript natively. I'll use `tsc --watch` to auto compile each time a file changes. And I'll use `nodemon` to run the development server. `tsc-watch` that allows me to run in watch mode and then also run another command on the same process. The script I use in my package.json looks like this:

```json
{
  "scripts": {
    "start": "NODE_ENV=development tsc-watch --onSuccess \"nodemon dist/index.js\""
  }
}
```

We'll see more of how this works as we build the app.

### 2. Setting up a Bolt app and running a local server

With the foundations in place I can make a start on building my bot. I'll need a few more dependencies:

```bash
yarn install @slack/bolt @slack/web-api body-parser cors dotenv express
```

I also need a Slack app. There are plenty of great resources to get you started so I won't cover it here. I recommend the bolt documentation [Getting Started](https://slack.dev/bolt-js/tutorial/getting-started) steps. It also covers many of the steps I'll show you but without the extra bits for setting up incoming requests and making it deployable to GCP functions.

```typescript
// src/index.ts

import { App, ExpressReceiver, LogLevel } from '@slack/bolt';
import dotenv from 'dotenv';

const PORT = 3000;

dotenv.config();

const token = process.env.SLACK_BOT_TOKEN;
const secret = process.env.SLACK_SIGNING_SECRET || '';

const receiver = new ExpressReceiver({
  signingSecret: secret,
});


/**
 * Initialise Bolt App
 */
const app = new App({
  token,
  receiver,
  logLevel: LogLevel.DEBUG,
});


// Initialises app in development
if (process.env.NODE_ENV === 'development')
  (async () => {
    await app.start(PORT);
    console.log(`Started slack bot 🚀 - PORT: ${PORT}`);
  })();

// Used by GCF to initialise function
module.exports = {
  slackBot: receiver.app,
};
```

The code speaks for itself, but a few things to point out. The `receiver` is an instance of `ExpressReceiver`. A built in class that gives us access to an Express app and `router` - handy for endpoints. TK (add the line) In development we'll need to start the app locally when running `yarn start`. Finally we need to export the module using commonjs syntax for Cloud Functions. Take not of the property name, `slackBot`, we'll need it later.

Running `yarn start` at this point will start the dev server and we should see `Started slack bot 🚀 - PORT: 3000` in the console. You'll also see out put from `tsc-watch` and nodemon. You can check the `dist` folder to see the compiled javascript that is being served.

### 3. Middleware for using Express

The bare necessities for an Express app as far as middleware goes is solving `CORS` and providing a way to parse response bodies. There nothing to complex here as I've already installed the packages we need, `cors` and `body-parser`. I'll apply them to the express receiver I created in step 2:

```typescript
// index.ts

import cors from 'cors';
import { json as jsonBodyParser } from 'body-parser';

receiver.router.use(cors());
receiver.router.use(jsonBodyParser());

// code removed for brevity...
```

## Using Slacks web-api to message any channel

Now that we've got a server running we can get to the fun stuff. Our bot can receive request from the internet thanks to the ExpressReceiver. I'll need to build an endpoint to hit and I'll need a way to send messages back to Slack.

### 1. Creating an Express router

Typically I would put all my routes in a different file to the main file we've. You can use Express to create a `Router` instance which I can add multiple routes to. I'll then pass this router into my primary catch all route back in the main file.

```typescript
// router.ts
import express from 'express';
const Router = express.Router();
Router.route('/some-route').post(/** some callback */)

export default Router;

// index.ts
import Router from './router.ts'

receiver.router.use('/', Router);
```

I can add as many routes as I need and they'll be tidily kept away in their own file.

### 2. Adding Slacks web-api to each request

To send messages to Slack we'll need access to the bolt apps `client`. The app isn't available on the express requests. So instead I instantiated Slacks web-api and added it to each request with some custom middleware.

```typescript
// index.ts
// ... other imports
import { WebClient } from '@slack/web-api';

const token = process.env.SLACK_BOT_TOKEN;

const webClient = new WebClient(token);

// receiver instantiation removed for brevity

// adds the webClient to `res.locals` for easy access by other routes
receiver.router.use((_, res, next) => {
  res.locals.slackWebClient = webClient;
  next();
});

// code removed for brevity

```

I've added the `webClient` to the response.locals object. I'd tried putting it directly onto the `request` object but the compiler was not happy with me. I couldn't find a workable way to extend the `Request` type itself, so after some googling I settled on using the locals options. The middleware runs before my routes, so every request will have access to the `webClient` which means I can interact with my Slack workspace with over 100 different methods.

I don't need 100 though, I only need the `webClient.chat.postMessage` method. This allows me to send a message to any channel the bot has been added to.

### 3. Messaging a channel with postMessage and an incoming POST request

`postMessage` is a POST request that requires a channel-id. It takes a json payload which you can send simple text or rich blocks. Because we're building "middleware" between client apps and Slack I'm not to concerned with what the body looks like - thats up to the client. I'll recieve the incoming request, make sure it's a POST, pass it's body on to `postMessage` and handle any errors. Job done.

```typescript
// router.ts
import express from 'express';
const Router = express.Router();

Router.route('/post-message').post( async (req, res) => {
  try {
    if (req.method !== 'POST') {
      const error = new Error('Only POST requests are accepted');
      error.code = 405;
      throw error;
    }

    // Pass message on to slack
    const slackResponse = await res.locals.slackWebClient.chat.postMessage(
      req.body,
    );

    // Return the slack response
    res.json(slackResponse);

    return await Promise.resolve();
  } catch (err) {
    res.status(err.code || 500).send(err);
    return Promise.reject(err);
  }
})

export default Router;
```

The naming convention I settled on matched the web-api method that it uses. In this case, `postMessage` becomes `/post-message`.
TK wishy washy 👇🏻
*I'd recommend adding in some authentication middleware to the incoming web requests.*
*If the client app has a user that's logged in with a token, forward it on here and validate it before you pass anything on to the Slack API.*
*It's probably also a good idea to validate that the incoming payload conforms to methods expected one*

That's it. My bot doesn't need anything else to be able to pass on notifications to my Slack workspace.

## Deploying to Google Cloud Functions

At my company we use GCP for all things cloud. Cloud functions are easy to deploy and cheep to run. You'll need to have a Google Cloud project,  `gcloud` cli installed on your machine and be authenticated. If you need it, [here's](https://cloud.google.com/sdk/docs/quickstart) the getting started guide for `gcloud`.

To deploy to cloud functions you can run this command:

```bash
gcloud functions deploy [function-name] \
--project [project-name] \
--runtime nodejs14 \
--trigger-http \
--source . \
--allow-unauthenticated \
--entry-point slackBot \
--region europe-west1
```

Give the function a name and point it at the correct project. If you only have one you don't need to set it as it's set when you authenticated int eh `gcloud` setup.

- `--source` is the directly that we want to upload to the function. Out built app is in the `dist` directory so originally I thought I'd just upload that. This was a mistake. The function needs a `package.json` in the root so it knows how to find the entry point - the value of to `main`. Set it to `dist/index.js` and upload the whole root directory along with package.json. To keep my deployment clean I use a `.gcloudignore` file and add all my directories and config files to it. The exceptions being `dist` and `package.json`.

- `--entry-point` is the name of the function we exported from index.ts using commonjs syntax. This is the entry point to the whole app and is called by the function when it initialises.

- **Environment variables** can be set in the deploy command using the `--set-env-vars` flag. But, you can also set them from the google cloud console. I prefer using the console because when I set-up CI/CD with CircleCI I don't have to manage adding them there too. Yes you'll need to add them for each separate deployment but, that's perfect, 12 Factor App for the win. To set variables in the console go to your function (you'll need to have run the script already). Click edit, then click the `RUNTIME, BUILD AND CONNECTIONS SETTINGS` drop down and add/edit any variables you need. Then click `NEXT`, then `DEPLOY`. This will re-deploy the function with the updated variables.

To run my deploy script I put it in a bash file called `deploy.sh` in the `scripts` directory. Then from the terminal run `bash ./scripts/deploy.sh`. The terminal will put out some messaging and eventually you'll get a success message. If you're setting the environment variables using the console your first deployment will fail. It's because the missing variables will cause the app to fail when it's initialised. Once you've set the variables you should be all set. Future deployments will work as expected!

## Resources

- Slack Webhooks
- Bot deployment options with common cloud service.
- Creating a Slack app
- Slack web-api documentation
- Bolt documentation
- Block Kit Builder
