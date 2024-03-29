# SvelteKit & monday code hello world app

monday code is a framework whereby all your monday.com apps are deployed to monday.com's serverless infrastructure, bypassing the need for your own infrastructure.

This is a hello world app for deploying SvelteKit on monday.com's monday code infrastructure.

This app doesn't do anything useful, except help you understand how to deploy a SvelteKit app on monday code.

- [SvelteKit](https://kit.svelte.dev/)
- [monday code](https://developer.monday.com/apps/docs/quickstart-guide-for-monday-code)

## Create a SvelteKit app

The easiest way to start building a SvelteKit app is to run `npm create`:

```
% npm create svelte@latest sveltekit-monday-code-hello-world

create-svelte version 6.0.9

┌  Welcome to SvelteKit!
│
◇  Which Svelte app template?
│  SvelteKit demo app
│
◇  Add type checking with TypeScript?
│  Yes, using JavaScript with JSDoc comments
│
◇  Select additional options (use arrow keys/space bar)
│  none
│
└  Your project is ready!

✔ Type-checked JavaScript
  https://www.typescriptlang.org/tsconfig#checkJs

Install community-maintained integrations:
  https://github.com/svelte-add/svelte-add

Next steps:
  1: cd sveltekit-monday-code-hello-world
  2: npm install
  3: git init && git add -A && git commit -m "Initial commit" (optional)
  4: npm run dev -- --open

To close the dev server, hit Ctrl-C

Stuck? Visit us at https://svelte.dev/chat
% cd sveltekit-monday-code-hello-world
% npm install

added 111 packages, and audited 112 packages in 20s

11 packages are looking for funding
  run `npm fund` for details

found 0 vulnerabilities
% git init && git add -A && git commit -m "Initial commit"
% npm run dev -- --open
```

This now shows you the demo SvelteKit app in a new browser window:

**image**

## Deploy it to monday code

See [Getting started with monday code](https://developer.monday.com/apps/docs/quickstart-guide-for-monday-code)

### Install the monday code dependencies

Globally install the CLI:

```
npm install -g @mondaycom/apps-cli
```

### SvelteKit adapters

Monday code natively supports nodejs, and there's an [example project here](https://github.com/mondaycom/welcome-apps/tree/master/apps/github-monday-code) that you can use for that.

If we want to run a SvelteKit app on monday code, then using a [node adapter](https://kit.svelte.dev/docs/adapter-node) for SvelteKit is a nice start.

Install:

```
npm i -D @sveltejs/adapter-node
```

In `svelte.config.js`, add:

```
import adapter from '@sveltejs/adapter-node';

export default {
	kit: {
		adapter: adapter()
	}
};
```

**Note:** We'll likely need to expand on this for CSP and other security headers at some point later.

### Building and deploying your SvelteKit app

To build your SvelteKit app, you can run `npm run build`, which will build all the code into a native javascript application in the `build` directory.

To deploy you app to monday code, you would normally run `mapps code:push`. This takes a copy of all the files in the directory and bundles them into an archive `code.tar.gz`, uploads it onto the monday code infrastructure and then attempts to run whatever it has.


**This is not going to work with SvelteKit.**

Firstly, we need only the contents of the `/build` directory, and secondly, we need to give it some instructions on what we want to execute.

This is where a separate `package.json` comes in.

`package.build.json`:

```
{
  "name": "sveltekit-monday-code-hello-world",
  "description": "This is for the vanilla javascript app deployed on monday code infrastructure",
  "version": "0.0.1",
  "private": true,
  "engines": {
    "node": "20.x",
    "npm": "10.x"
  },
  "scripts": {
    "server": "NODE_ENV=production node ./index.js"
  },
  "dependencies": {},
  "type": "module"
}
```

The above file is the instruction for the monday code infrastructure that explains how to start the service – the `npm run start` instruction was copied from [here](https://github.com/mondaycom/welcome-apps/blob/master/apps/github-monday-code/package.json).

Now, we need to update the `scripts` object in our `package.json`:

```
{
  "name": "sveltekit-monday-code-hello-world",
  "version": "0.0.1",
  "scripts": {
    "dev": "vite dev",
    "build": "npm run build:default && cp package.build.json build/package.json",
    "build:default": "vite build",
    "preview": "vite preview",
    "check": "svelte-kit sync && svelte-check --tsconfig ./jsconfig.json",
    "check:watch": "svelte-kit sync && svelte-check --tsconfig ./jsconfig.json --watch",
    "mapps:install": "npm install -g @mondaycom/apps-cli",
    "mapps:init": "mapps init",
    "mapps:code:push": "npm run build && cd build && mapps code:push"
  },
  "devDependencies": {
    "@fontsource/fira-mono": "^4.5.10",
    "@neoconfetti/svelte": "^1.0.0",
    "@sveltejs/adapter-auto": "^3.0.0",
    "@sveltejs/kit": "^2.0.0",
    "@sveltejs/vite-plugin-svelte": "^3.0.0",
    "svelte": "^4.2.7",
    "svelte-check": "^3.6.0",
    "typescript": "^5.0.0",
    "vite": "^5.0.3"
  },
  "type": "module"
}
```

The `npm run build` script now runs the default build (`vite build`) and then copies `package.build.json` to `build/package.json`.

The `npm run mapps:code:push` script runs the build step, and then from the `/build` directory, runs `mapps code:push`.

This bundles the contents of the `/build` directory, which also contains a `package.json` with instructions on how to run the app, into a `code.tar.gz` archive, which is then sent over to monday code infrastructure to run there.

### Environment variables

We don't want any secret information such as OAuth client ID/secret or database connections and the like to be anywhere in our codebase, so we need a way to set these.

For local development, a `.env` file (which **should be ignored** using `.gitignore` – don't ever let this file into your repository) works nicely.

***Aside:** SvelteKit will just read the contents of a `.env` file automatically. It's not node/express, so there's no need to have a `dotenv` dependency here.*

`.env`:

```
MY_SECRET_CODE=ABC123
```

***Aside:** If you need to use the monday signing key, and/or OAuth key and secret for your app, this is a good place to store them. Be sure to also add them to monday code as described below.*

To use this in the monday code enviroment, you'll need to set this using:

```
mapps code:env -m set -k MY_SECRET_CODE -v ABC123 -i <app_id>
```

As an example, we've added the following to the `.env` file:

```
VITE_MY_VARIABLE=Hello World
MY_SECRET_CODE=ABC123
```

And then added the following code to `routes/+page.svelte`:

```
<p>VITE_MY_VARIABLE: {import.meta.env.VITE_MY_VARIABLE}</p>
<p>MY_SECRET_CODE: {import.meta.env.MY_SECRET_CODE}</p>
```

If we then also set the variables for monday code:

```
mapps code:env -m set -k VITE_MY_VARIABLE -v "Hello World" -i <app_id>
mapps code:env -m set -k MY_SECRET_CODE -v ABC123 -i <app_id>
```

Only the value of `VITE_MY_VARIABLE` will be allowed to be passed to the client to be displayed:

\*\* IMAGE GOES HERE \*\*

The point of this exercise is 2-fold:

1. To prove that our app can read environment variables from where they've been set
2. To also prove that only environment variables with the prefix `VITE_` can be passed through and displayed to the client browser

[More details on SvelteKit .env secrets](https://www.scottspence.com/posts/sveltekit-env-secrets).

### Dealing with form actions

If `adapter-node` can't correctly determine the URL of your deployment, you may experience this error when using [form actions](https://kit.svelte.dev/docs/form-actions):

> Cross-site POST form submissions are forbidden

To fix this, you'll need to set the `ORIGIN` in your monday code environment variables:

```
mapps code:env -m set -k ORIGIN -v https://instance-name-here.us.monday.app -i <app_id>
```
