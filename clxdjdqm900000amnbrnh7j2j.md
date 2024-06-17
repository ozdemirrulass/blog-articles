---
title: "Enterprise-Level Authentication in a Containerized Environment for Next.js 14 - AuthJS Patch"
seoTitle: "Enterprise-Level Authentication in a Containerized Environment NextJS"
seoDescription: "Integrate NextJS, Keycloak and MySQL services and handle authentication flow using AuthJS in a containerized environment."
datePublished: Thu Jun 13 2024 17:31:17 GMT+0000 (Coordinated Universal Time)
cuid: clxdjdqm900000amnbrnh7j2j
slug: enterprise-level-authentication-in-a-containerized-environment-for-nextjs-13-authjs-patch
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1718300660717/5db2bb5e-4bc3-40dc-a305-85934062c106.png
tags: docker, mysql, nextjs, keycloak, docker-compose, authjs

---

Recently, I've published a step by step guide on [Enterprise-Level Authentication in a Containerized Environment for NextJS](https://ulasozdemir.com.tr/enterprise-level-authentication-in-a-containerized-environment-for-nextjs-13) and while searching on the internet I saw a [post](https://keycloak.discourse.group/t/issues-running-keycloak-in-docker-and-authenticating-with-authjs/26443/8) on Keycloak groups. [benmarte](https://keycloak.discourse.group/u/benmarte/summary) was having a problem while integrating Keycloak using [auth.js](https://authjs.dev/) instead of [next-auth](https://next-auth.js.org/). In this post I'll explain how you can handle authentication using [auth.js](https://authjs.dev/).

To get the most out of this post please setup your environment as I shared in my [previous post](https://ulasozdemir.com.tr/enterprise-level-authentication-in-a-containerized-environment-for-nextjs-13).

## TL;DR

[https://github.com/ozdemirrulass/nextjs-keycloak-authjs](https://github.com/ozdemirrulass/nextjs-keycloak-authjs)

## NextJS 14 Keycloak Integration Using AuthJS

1. Install AuthJS as it is suggested in [official documentation](https://authjs.dev/getting-started/installation?framework=next.js):
    

```bash
npm install next-auth@beta
```

2. Create `.env.local` file in the root directory of your next-app project and add the following:
    

```plaintext
AUTH_SECRET=secret

NEXT_PUBLIC_KEYCLOAK_REALM=<realm-name>
NEXT_PUBLIC_KEYCLOAK_CLIENT_ID=<client-name>
KEYCLOAK_CLIENT_SECRET=<client-secret-from-keycloak>
NEXT_LOCAL_KEYCLOAK_URL="http://localhost:8080"
NEXT_CONTAINER_KEYCLOAK_ENDPOINT="http://keycloak:8080"
```

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">You can use following commands to produce AUTH_SECRET value. This secret is used to sign and encrypt cookies.</div>
</div>

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">You can see where to find other values in my <a target="_blank" rel="noopener noreferrer nofollow" href="https://ulasozdemir.com.tr/enterprise-level-authentication-in-a-containerized-environment-for-nextjs-13" style="pointer-events: none">PREVIOUS POST</a>.</div>
</div>

```bash
openssl rand -base64 33
```

or

```bash
npx auth secret
```

3. Create `/types` folder in your projects root and create `node-env.d.ts` file in it.
    

```typescript
// /types/node-env.d.ts
declare namespace NodeJS {
    export interface ProcessEnv {
      NEXT_PUBLIC_KEYCLOAK_CLIENT_ID: string
      KEYCLOAK_CLIENT_SECRET: string
      NEXT_LOCAL_KEYCLOAK_URL: string
      NEXT_PUBLIC_KEYCLOAK_REALM: string
      NEXT_CONTAINER_KEYCLOAK_ENDPOINT: string
    }
  }
```

4. Create `auth.ts` file under your `/src` directory and add the following code:
    

```typescript
// /src/auth.ts
import NextAuth from "next-auth"
import Keycloak from "next-auth/providers/keycloak";

export const { handlers, signIn, signOut, auth } = NextAuth({
  providers: [
    Keycloak({
      jwks_endpoint: `${process.env.NEXT_CONTAINER_KEYCLOAK_ENDPOINT}/realms/myrealm/protocol/openid-connect/certs`,
      wellKnown: undefined,
      clientId: process.env.NEXT_PUBLIC_KEYCLOAK_CLIENT_ID,
      clientSecret: process.env.KEYCLOAK_CLIENT_SECRET,
      issuer: `${process.env.NEXT_LOCAL_KEYCLOAK_URL}/realms/${process.env.NEXT_PUBLIC_KEYCLOAK_REALM}`,
      authorization: {
        params: {
          scope: "openid email profile",
        },
        url: `${process.env.NEXT_LOCAL_KEYCLOAK_URL}/realms/myrealm/protocol/openid-connect/auth`,
      },
      token: `${process.env.NEXT_CONTAINER_KEYCLOAK_ENDPOINT}/realms/myrealm/protocol/openid-connect/token`,
      userinfo: `${process.env.NEXT_CONTAINER_KEYCLOAK_ENDPOINT}/realms/myrealm/protocol/openid-connect/userinfo`,
    }),
  ],
});
```

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">This part is important! We must use any kind of <strong>parameter indicating urls</strong> as <code>localhost</code> but urls used for <strong>sending a request </strong>must use <code>container name</code> for connectivity purposes through <strong>docker network</strong>.</div>
</div>

5. Now it's time to create our `route.ts` and it must be in `/src/app/api/auth/[...nextauth]/route.ts`
    

```typescript
// /src/app/api/auth/[...nextauth]/route.ts

import { handlers } from "@/auth";
export const { GET, POST } = handlers;
```

6. We are going to need a Sign-In component! Let's create It. Create a `components` folder under your `/src` directory and create a file as `sign-in.tsx`.
    

```typescript
// /src/components/sign-in.tsx

import { signIn } from "@/auth"
 
export function SignIn() {
  return (
    <form
      action={async () => {
        "use server"
        await signIn("keycloak")
      }}
    >
      <button type="submit">Signin with Keycloak</button>
    </form>
  )
}
```

7. It's time to replace our `src/app/page.tsx` :
    

```typescript
// src/app/page.tsx
import { auth } from "@/auth";
import { SignIn } from "../components/sign-in";

export default async function Home() {
  const session = await auth();
  if (session) {
    return (
      <div>
        <div>Your name is {session.user?.name}</div>
      </div>
    );
  }
  return (
    <div>
      <SignIn />
    </div>
  );
}
```

Congratulations 👏 You've done it. You just implemented an Enterprise-Level Authentication in a Containerized Environment for Next.js 14 using AuthJS instead of [NextAuth](https://next-auth.js.org/) !

```bash
docker compose -f docker-compose.dev.yml build
```

```bash
docker compose -f docker-compose.dev.yml up -d
```

Thank you and see you soon!