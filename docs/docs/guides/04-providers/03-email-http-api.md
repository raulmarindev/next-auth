---
id: email-http
title: HTTP-based Email Provider
---

## Introduction

We have always had a built-in Email provider with which you could connect to the SMTP server of your choice to send "magic link" emails for sign-in purposes. However, the Email provider can also be used with HTTP-based email services, like AWS SES, Postmark, Sendgrid, etc. In this guide we are going to explain how to use our Email magic link provider with any of the more modern HTTP-based Email APIs.

For this example we will be using [Sendgrid](https://sendgrid.com), but any email service providing an HTTP API or JS client library will work.

## Setup

First, if you do not have a project using Auth.js, clone and setup a basic Auth.js project like the the one [provided in our example repo](https://github.com/nextauthjs/next-auth-example.git). We will need to make at least the following modifications to the example repository, or any other project you're adding this to, including:

- Install and configure any [Auth.js Database Adapter](/adapters/overview), as it is a requirement for the Email provider.
- Generate an API key from your cloud Email provider of choice and add it to your `.env.*` file. For example, mine is going to be called `SENDGRID_API`.

At this point, you should have a `[...nextauth].ts` file which looks something like this:

```ts title="pages/api/auth/[...nextauth].ts"
import NextAuth, { NextAuthOptions } from "next-auth"
import EmailProvider from "next-auth/providers/email"
import { PrismaAdapter } from "@next-auth/prisma-adapter"
import prisma from "../../../lib/db"

// For more information on each option (and a full list of options) go to
// https://next-auth.js.org/configuration/options
export const authOptions: NextAuthOptions = {
  providers: [
    EmailProvider({
      server: process.env.EMAIL_SERVER,
      from: process.env.EMAIL_FROM,
    }),
  ],
  // ...
}

export default NextAuth(authOptions)
```

## Customization

This is a standard Auth.js Email provider + Prisma adapter setup. At this point, we will customize a few things in order to avoid using the built-in SMTP connectivity to send emails, and instead use the HTTP API endpoint provided to us by our cloud Email provider of choice.

To make this change, first remove any of the existing options inside the object being passed to the `EmailProvider` and add `sendVerificationRequest`, like so:

```js title="pages/api/auth/[...nextauth].ts"
export const authOptions: NextAuthOptions = {
  ...,
  providers: [
    EmailProvider({
      sendVerificationRequest: async (params) => {
        const { identifier: email, url } = params

        // Coming soon...
      }
    })
  ],
  ...
}
```

This method allows us to override the sending behaviour of the email adapter. It was designed exactly for this reason, to be able to override the default `nodemailer` SMTP transport, so you could theoretically use another SMTP library, or send emails via a totally different route, as we are doing here.

The goal now is to simply call the HTTP endpoint from our cloud email provider to send a message and pass it the `to` address, the email `body`, and any other fields we may want to include.

As mentioned earlier, we're going to be using Sendgrid in this example, so the endpoint we need to use is `https://api.sendgrid.com/v3/mail/send` ([Docs](https://docs.sendgrid.com/for-developers/sending-email/api-getting-started)). Therefore, after pulling out some of the important information from the `params` parameter, we're going to continue by making a `fetch()` call to the previously mentioned API endpoint.

```js title="pages/api/auth/[...nextauth].ts"
export const authOptions: NextAuthOptions = {
  ...,
  providers: [
    EmailProvider({
      async sendVerificationRequest(params) {
        const { identifier: email, url } = params
        // Call the cloud Email provider API for sending messages
        const response = await fetch("https://api.sendgrid.com/v3/mail/send", {
          // The body format will vary depending on provider, please see their documentation
          // for further details.
          body: JSON.stringify({
            personalizations: [{ to: [{ email }] }],
            from: { email: "noreply@company.com" },
            subject: "Sign in to Your page",
            content: [
              {
                type: "text/plain",
                value: `Please click here to authenticate - ${url}`,
              },
            ],
          }),
          headers: {
            // Authentication will also vary from provider to provider, please see their docs.
            Authorization: `Bearer ${process.env.SENDGRID_API}`,
            "Content-Type": "application/json",
          },
          method: "POST",
        })

        if (!response.ok) {
          const { errors } = await response.json()
          throw new Error(JSON.stringify(errors))
        }
      },
    })
  ],
  ...
}
```

That's all we need to do to send Emails via an HTTP API like that from SendGrid, Postmark, AWS SES, etc. Note here that the example is only using `text/plain` as the body type. You'll probably want to change that to `text/html` and pass in a nice looking HTML email. See, for example, our `html` function in [the docs](/providers/email#customizing-emails).

## Further Reading

- [Email provider documentation with HTML generation and more](/reference/core/modules/providers_email)
- [SendGrid JSON Body documentation](https://docs.sendgrid.com/api-reference/mail-send/mail-send#body)