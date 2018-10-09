# Screenshot Testing Infrastructure

This directory contains the core logic for running, capturing, diffing, updating screenshot tests.

Component test pages are located in [`test/screenshot/spec/`](/test/screenshot/spec).

## Shield generator

![shield generator](https://user-images.githubusercontent.com/409245/46484103-f0525f80-c7ad-11e8-90c0-593501a186e5.png)

### Public SVG images and URL redirects

The screenshot shield in the [main `README`](/README.md) is generated by two [Google Cloud Functions](https://cloud.google.com/functions/)
defined in [`test/screenshot/infra/index.js`](/test/screenshot/infra/index.js):

- `screenshotShieldSvg(req, res)` → [`screenshot-shield-svg`](https://us-central1-material-components-web.cloudfunctions.net/screenshot-shield-svg)
- `screenshotShieldUrl(req, res)` → [`screenshot-shield-url`](https://us-central1-material-components-web.cloudfunctions.net/screenshot-shield-url)

Unfortunately, the [`gh-badges`](https://www.npmjs.com/package/gh-badges) module hasn't been updated for over 2 years, and is
missing a fair amount of functionality compared to shields.io.

As a result, SVG images are transparently proxied to
[`https://img.shields.io/badge/screenshots-<message>-<color>.svg`](https://img.shields.io/badge/screenshots-581%20pass-brightgreen.svg).

Responses from shields.io are cached for the [lifetime of the Function](https://cloud.google.com/functions/docs/concepts/exec#function_instance_lifespan).

#### URL params

Both HTTP endpoints support the following special URL params:

| Key | Default | Values |
| -- | -- | -- |
| `ref` | `master` | Any valid git branch or full-length commit SHA | 
| `state` | `error`, `passed`, or `failed` | `unknown`, `starting`, `running`, `passed`, `failed`, `error` |

Example:

> [screenshot-shield-svg?ref=chore/infra/latest-shield&state=passed](https://us-central1-material-components-web.cloudfunctions.net/screenshot-shield-svg?ref=chore/infra/latest-shield&state=passed)
> 
> ![reserved URL params](https://us-central1-material-components-web.cloudfunctions.net/screenshot-shield-svg?ref=chore/infra/latest-shield&state=passed)

All other URL params are merged with the destination URL. For example:

> [screenshot-shield-svg?style=for-the-badge&state=passed](https://us-central1-material-components-web.cloudfunctions.net/screenshot-shield-svg?style=for-the-badge&ref=chore/infra/latest-shield&state=passed)
> 
> ![custom URL params](https://us-central1-material-components-web.cloudfunctions.net/screenshot-shield-svg?style=for-the-badge&ref=chore/infra/latest-shield&state=passed)

### Data source

The GCP Functions get their status data from a Google Cloud Datastore.

The Datastore is populated by [`StatusNotifier`](/test/screenshot/infra/lib/status-notifier.js) every time screenshot tests run on [Travis CI](https://travis-ci.com/material-components/material-components-web).

`StatusNotifier` also pushes progress updates to GitHub in the form of user-visible
[repo/commit status checks](https://developer.github.com/v3/repos/statuses/) on PRs, posted by [`screenshot-test/butter-bot`](https://github.com/mdc-web-bot):

![image](https://user-images.githubusercontent.com/409245/46483254-0d862e80-c7ac-11e8-974a-b667373e8a02.png)

### Testing and deploying Cloud Functions

#### Debugging locally

First, install the [Cloud Emulator](https://cloud.google.com/functions/docs/emulator):

```bash
npm install -g @google-cloud/functions-emulator
```

Then start up a local HTTP server:

```bash
test/screenshot/infra/deploy.sh --local
```

The HTTP endpoints should now be available at:

* http://localhost:8010/material-components-web/us-central1/screenshot-shield-svg
* http://localhost:8010/material-components-web/us-central1/screenshot-shield-url

You can also run the Functions directly from the Terminal:

```bash
functions call screenshot-shield-svg
```

To view the logs:

```bash
functions logs read
```

To debug the code interactively in a V8 inspector:

```bash
functions inspect screenshot-shield-svg
```

Then open the Chrome DevTools on any website (it doesn't matter which) and click the green Node.js icon to connect the debugger:

![Chrome DevTools Node.js icon](https://user-images.githubusercontent.com/409245/46485599-d5cdb580-c7b0-11e8-8723-441281743916.png)

After setting a breakpoint, invoke the Function from the Terminal to pause execution in the debugger:

```bash
functions call screenshot-shield-svg
```

#### Deploying to GCP

Run:

```bash
test/screenshot/infra/deploy.sh --prod
```

This will recompute Datastore indexes and deploy the Functions to their public URLs:

* https://us-central1-material-components-web.cloudfunctions.net/screenshot-shield-svg
* https://us-central1-material-components-web.cloudfunctions.net/screenshot-shield-url

#### Viewing production logs

In the Google Cloud Functions Console:

1. Click on the Function you want to inspect
2. Select the "Testing" tab
3. Click "Test the function"
4. Click "See all logs for this function execution"

![image](https://user-images.githubusercontent.com/409245/46485116-e7628d80-c7af-11e8-80fe-ae19d8c5bffc.png)