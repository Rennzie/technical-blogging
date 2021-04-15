# Writing Plan

## Authors

- Sean Rennie

## Title

- Using Slack for status notifications from any webapp

## Blurb

- Send status message or other notifications to Slack from any client side app.
- Sending messages to slack from any server side applications is simple using webhooks. Sending them from a webapp is not.

## Audience

- Javascript developers looking to send notifications to slack from deployed apps

## Angle

- Status notifications from running web applications
- This is an in-depth tutorial.

## Points to cover

- Setting up a Slack App (just point to relevant docs as it's been covered loads)
- Getting a typescript dev environment running for the express app.
- What are routes and how do we leverage them. Express for the win. But using it from a bolt app.
  - Making the distinction between `app` and `routes`. | Worth setting this language up at the start | Maybe I can find better names?
- Considerations for deploying. Module exports and running an app in development
- Suggested folder architecture
- Testing? | I can add my own tests and then write about it here
- Adding custom middleware to pass the web-api SDK to all endpoints
- Hacking a client side webhook with `'Content-type': 'application/x-www-form-urlencoded'` in a POST request
  - It makes it a simple request which disabled the preflight CORs check.

## Sections

- TL;DR
- What are we solving and why bother
- Webhooks for messaging a Slack channel and how not to do it from the client
- Scaffolding a Typescript, Bolt & Express app
- Express for Bolt
  - a. Leveraging middleware to overcome the dreaded CORS
  - b. Slacks web-api SDK
- Deploying the bot to Cloud Functions (GCP)
- Resources

## Notes

- It's likely that notifications from apps to slack should be infrequent enough that the channel is not overloaded.
  - Masses of messages would be better suited for tools like Sentry, Google Analytics or other logging/analytics tools
  - Uses cases could be in app error/bug reporting (one way comms), status notifications, internal tools
- Considerations  
  - Authentications of client side request
  - Better error handling
