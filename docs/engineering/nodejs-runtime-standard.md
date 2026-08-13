# Node.js Runtime Standard

Bond software that executes Node.js must use Node.js **24.19.0 (Krypton)**. Node.js 24 is the current Active LTS line and is supported through April 2028 according to the [Node.js release schedule](https://nodejs.org/en/about/previous-releases).

The standard applies to production services, web application builds and server runtimes, Lambda functions, infrastructure programs, GitHub Actions, command-line tools, published packages, tests, and release automation. A `package.json` used only as a Unity package manifest does not make a Unity repository a Node.js project.

## Required declarations

Each repository has one local version source of truth:

```text
# .nvmrc
24.19.0
```

Other declarations must agree with it:

- Set `engines.node` to `24.x`. This prevents accidental execution on another major while allowing managed platforms to apply security patches within Node 24.
- Configure GitHub Actions with `node-version-file: .nvmrc`. Avoid repeating a version literal in workflow files.
- Pin container images to `node:24.19.0-<variant>`. Preserve the existing distribution and variant unless the repository has a separate reason to change it.
- Configure AWS Lambda with the managed `nodejs24.x` runtime.
- Configure JavaScript GitHub Actions with `runs.using: node24` and build their committed distribution with Node 24.19.0.
- Keep the repository's existing package manager and lockfile. A Node runtime update is not permission to change package managers or dependencies.

Managed platforms such as [Vercel](https://vercel.com/docs/functions/runtimes/node-js/node-js-versions) and [AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtimes.html) expose only the Node major. In those environments, `24.x` is the precise supported declaration; local development, CI, and container builds remain pinned to 24.19.0.

## Validation

A runtime update must install from the committed lockfile and run the repository's relevant type checks, tests, build, and container build under Node 24.19.0. Native dependencies must be rebuilt rather than reused from a different Node major.

Infrastructure validation must stop at tests and read-only previews unless the normal production-approval process explicitly authorizes an update. A runtime-standard change does not authorize a deployment or a live infrastructure mutation.

## Updates and exceptions

Adopt a newer Node line only after it reaches Active LTS and every in-scope repository passes its normal validation on that line. Update this document and all runtime declarations as one coordinated migration.

An exception must identify the incompatible dependency or platform, its supported Node range, the owner, and a removal date. "It has always used this version" is not an exception. Repositories without a documented exception are expected to match this standard.
