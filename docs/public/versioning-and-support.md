# Versioning and support

This documentation describes `novaos-agent-sdk` 2.x and its Agent2Agent (A2A) interfaces. Check the installed version with `novaos_client_sdk.__version__` before following an example.

## Select the current major version

For a new project, keep the dependency within this major version:

```text
novaos-agent-sdk>=2,<3
```

The distribution name is `novaos-agent-sdk`; the Python import is `novaos_client_sdk`.

## Compatibility policy

The package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html). Changes to the documented public API that break compatibility require a new major version. Review the [changelog](changelog.md) when upgrading and test the operations your application uses.

Documentation scope does not establish a maintenance or end-of-life commitment for earlier releases. No fixed end-of-life date for this major version is published here. If you need a specific maintenance commitment, confirm it with the SDK maintainer.

Some configuration helpers retain historical environment-variable defaults. The [authentication guide](guides/authentication.md) and [discovery configuration](guides/invoking-agents.md#discovery) document their actual behavior. A configuration name does not select an alternative wire protocol.

## Reporting a bug

Open an issue on this repository's [Issues](https://github.com/OrbitronAI-Repo/novaos-agent-sdk/issues) page with a minimal reproduction, the SDK version and your Python version. See [Security](security.md) instead if the issue is a vulnerability. Do not include credentials or private data.

There is no formal support SLA; the SDK is maintained on a best-effort basis.
