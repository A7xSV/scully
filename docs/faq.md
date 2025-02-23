---
title: Frequently Asked Questions
published: true
lang: en
position: 200
---

# Frequently Asked Questions

## Ignoring Routes

<details>
<summary>Ignore routes without config</summary>

> I have a lot of routes I don't want Scully to handle.  
> How can I deal with this?

Scully will use the `default` plugin for any route that is not specified. When you want to have another way to handle defaults, you can replace this plugin with another one.  
For example, if you want to ignore all undefined routes you can do:

```typescript
registerPlugin('router', 'default', findPlugin('ignored'));
```

In case you want to have some more control, you can create a custom plugin:

```typescript
registerPlugin(
  'router',
  'default',
  async (route: string): Promise<HandledRoute[]> => {
    if (route === 'somethingSpecial') {
      return [{ route, type: 'somethingElse' }];
    }
    if (route === 'somethingSpecial/:id') {
      const data = httpGetJson('someEndPoint'); // fetch some json
      const { createPath } = routeSplit(route);
      const routes: HandledRoutes[] = [];
      for (const row of data) {
        routes.push({ route: createPath(row.id), type: 'default' });
      }
      return routes;
    }
    return [];
  },
  undefined,
  { replaceExistingPlugin: true }
);
```

</details>

## Plugins

<details>
<summary>How do I fix plugin build errors related to the `express-serve-static-core` module?</summary>

> Building a plugin results in a fatal error `Cannot find module 'express-serve-static-core'`, originating from `node_modules/@scullyio/scully/lib/utils/serverstuff/staticServer.d.ts`

To correct this, add the `skipLibCheck` and `skipDefaultLibCheck` flags to your `tsconfig.json` => `compilerOptions` like this:

```json
{
  "compileOnSave": false,
  "compilerOptions": {
    "skipLibCheck": true,
    "skipDefaultLibCheck": true
  }
}
```

</details>

<details>
<summary>Scully doesn't seem to find the plugins I just declared</summary>

> Running scully gives a fatal error: `Unknown type "myPlugin" in route "/aRoute"`

> I get this error:

```
--------------------------------------------------------------------------
you started scully outside of a scully project-folder,
or didn't install packages in this folder.
We can't find your local copy to start.
This can also happen on windows with PowerShell and mixed case path-names
--------------------------------------------------------------------------
```

This might happen when you started scully from within a different project, a subfolder that is too deeply nested.
Or you are on Windows, using Powershell and have a uppercase character in your path.
Scully will first try to start the local version, but if it can't find that, it errors out with this error.
The solution is that you should start Scully inside the root of your project with:

```bash
npx scully
```

That will use the local version of scully, and should solve the issue.

</details>

## Route Parameters

<details>
<summary>No configuration for route found</summary>

If you run Scully and the following warning is displayed, you need to teach Scully how to use the project's route parameters.

```bash
No configuration for route `/user/:userId` found. Skipping
```

The above error is given because Scully does not know all the possible values for `:userId`. Teach Scully how to get the list of `:userId`s from your app. Scully can turn `/user/:userId` into a list of meaningful pre-renderable routes like so:

```
/user/1
/user/2
/user/3
...
/user/100
```

Even small Angular projects have routes that contain route parameters. To stop Scully from skipping these routes, configure a [route plugin](/docs/Reference/plugins/types/router). Route plugins teach Scully how to fetch data and merges it into routes using parameters.

The easiest way to understand route plugin is by understanding the [`jsonPlugin`](/docs/Reference/plugins/built-in-plugins/json). It simply fetches data from any API that you specify, and it returns a list of properties that can be used to replace the route parameter. Checkout the [jsonPlugin docs](/docs/Reference/plugins/built-in-plugins/json) to see an example of how easy this configuration is.

</details>

<details>
<summary>Language params in routes</summary>

> I have a routing structure which looks like this:  
> `/:lang`  
> `/:lang/page1`  
> `/:lang/page2`  
> etc.  
> `:lang` can have few values (`'it'`, `'en'`, etc.)  
> I prefer to store `:lang` in the config, without a dedicated endpoint.  
> How can I solve this?

As the Scully config file is typescript, you can post-process the routing object.  
A very crude solution would be something like this:

```typescript
import { ScullyConfig } from '@scullyio/scully';

const preLangConfig: ScullyConfig = {
  /** settings */
  routes: {
    ':lang/route1': { type: 'default' },
    ':lang/route2': { type: 'default' },
    ':lang/route3': { type: 'default' },
    ':lang/route4': { type: 'default' },
  },
};
export const config = {
  ...preLangConfig,
  routes: Object.fromEntries(
    // make sure you use a node-version that supports this, or use a reduce.
    Object.entries(preLangConfig.routes).reduce((all, [route, config]) => {
      if (route.includes(':lang')) {
        ['it', 'en', 'nl', 'sp'].forEach(
          (
            lang // <-- language array
          ) => all.push([route.split(':lang').join(lang), config])
        );
      } else {
        all.push([route, config]);
      }
      return all;
    }, [])
  ),
};

console.log(config.routes);
```

It takes the `preLangConfig` and iterates over all the routes. When it finds the `:lang` parameter, it creates an entry with every value provided in the language array. That way the final config will have a route for every language available.

</details>

## Docker and CI/CD

<details>
<summary>Using Scully inside Docker, GitLab, or other CI/CD environments</summary>
> When I run Scully in XXX it gets stuck.

In all the cases we have seen around this, it is a problem with puppeteer running inside XXX. Most often it is missing the Chrome dependency.
A lot of information about this is on the [puppeteet troubleshooting page](https://github.com/puppeteer/puppeteer/blob/main/docs/troubleshooting.md)

We heard back from several users that a dockerfile like the below one works for them.

```docker
FROM node:12-alpine

RUN apk add --no-cache \
      chromium \
      ca-certificates

ENV PUPPETEER_SKIP_CHROMIUM_DOWNLOAD true
```

As a base docker config, and then make sure to set the environment correctly in the container that runs Scully:
In order to use this I create my projects' Docker file like this:

```docker
FROM aboveConfig
ENV SCULLY_PUPPETEER_EXECUTABLE_PATH '/usr/bin/chromium-browser'
... more docker stuff here
... in the end:
RUN npx scully
```

Also, make sure you add the following to your config:

```typescript
  puppeteerLaunchOptions: {
    args: ['--no-sandbox', '--disable-setuid--sandbox'],
  },
```

</details>

<details>
<summary>Scully inside GCE has timeout failures.</summary>

It seems that inside GCE sometimes the server takes a long time to properly come up. If this happens, you can extend the waiting time for the server with a command-line parameter like:

```bash
npx scully --handle404=index --hostName="${SSR_HOST_NAME}" --noPrompt  --serverTimeout=60000
```
The following puppeteer settings are reported to help in this case:
```typescript
export const config: ScullyConfig = {
  projectRoot: './pathToRoot',
  projectName: 'nameOfProject',
  outDir: './dist/static',
  routes: {
    /** your route config **/
  },
  defaultPostRenderers,
  puppeteerLaunchOptions: {
    executablePath: '/usr/bin/chromium-browser',
    args: [
      '--no-sandbox',
      '--disable-setuid--sandbox',
      '--headless',
      '--disable-gpu',
      '--disable-dev-shm-usage',
      '--no-default-browser-check',
      '--no-first-run',
      '--disable-default-apps',
      '--disable-popup-blocking',
      '--disable-translate',
      '--disable-background-timer-throttling',
      '--disable-renderer-backgrounding',
      '--disable-device-discovery-notifications',
      '--disable-web-security',
    ],
  },
};

```

</details>

## File locations

<details>
<summary>Dist folder</summary>
> Scully tells me I can't use the `dist` folder

As in some cases the Angular CLI puts the distribution files directly in the dist folder, and the Scully outputs its result in a subfolder of that by default.
As most operating systems will raise objections if you are trying to copy a folder into a subfolder of that same folder. Scully will raise an error.
To fix this error, you should open your `angular.json` and find the property `outputhPath`
Then change that from:

```json
  ...,
  "architect": {
    ...,
    "build" : {
      ...,
      "outputPath": "dist",
    }
  }

```

to:

```json
  ...,
  "architect": {
    ...,
    "build" : {
      ...,
      "outputPath": "dist/someName",
    }
  }

```

</details>

## Scully Command Line Interface

<details>
<summary>Why are the pair of hyphens `--` used when running `npm run Scully`?</summary>

The pair of hyphens i.e. `--` indicates to `NPM`-scripts that this is the end of node
options and every option after that should be passed to the script being run, in
this case Scully.

</details>

<details>
<summary>Scully is using stored routes despite running with --scanRoutes?</summary>

This is an issue with how NPM scripts are working and is not something Scully can solve.
If you use `npm run scully` you need to add `--` to tell NPM that you want to give parameters to the command. so run it like this:

```bash
npm run scully -- --scanRoutes
```

When using NPX, you don't need to add `--` and doing so might break things
To make the whole story more cumbersome, in different versions of NPM different things happened. For a while adding the `--` into the NPM script helped, and the user didn't need to provide them. But it turns out that this works differently, depending on OS and NPM version.
As a result we had the `--` in some versions of our schematics, and those ended up in the package.json scripts. If you have them there, remove them. 
An easy way to workaround this is updating your `package.json` script with the following:

```json
  "scripts": {
    "/** ","Other scripts are still here! **/",
    "scully": "npx scully",
    "scully.scan": "npx scully --scanRoutes",
    "scully.serve": "npx scully serve"
  },
```

Then, when you want to scan, you can run `npm run scully.scan`

</details>


