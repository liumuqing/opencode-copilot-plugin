# GitHub Copilot Standalone Plugin

This directory contains a standalone GitHub Copilot auth/provider plugin for opencode.

The plugin exports two entry points:

```ts
import { CopilotAuthPlugin, createCopilotAuthPlugin } from "./github-copilot-standalone/copilot"
```

- `CopilotAuthPlugin` loads the plugin with the default behavior.
- `createCopilotAuthPlugin(options)` lets you wrap GitHub Copilot request bodies before they are sent.

## Basic Usage

Create a plugin file under `~/.config/opencode/plugins/`:

```ts
export { CopilotAuthPlugin as default } from "./github-copilot-standalone/copilot"
```

Restart opencode after adding or changing the plugin.

## Modify Request Body

Use `createCopilotAuthPlugin({ transform })` when you need to inspect or modify the final GitHub Copilot HTTP request body.

Example:

```ts
import { createCopilotAuthPlugin } from "./github-copilot-standalone/copilot"

export default createCopilotAuthPlugin({
  async transform(url, headers, body) {
    if (!url.includes("api.githubcopilot.com") && !url.includes("copilot-api.")) {
      return body
    }

    if (body?.messages) {
      return {
        ...body,
        messages: body.messages.map((message: any) => {
          if (message.role !== "user") return message
          return {
            ...message,
            content: typeof message.content === "string"
              ? message.content.replace(/secret_[a-z0-9]+/gi, "[REDACTED]")
              : message.content,
          }
        }),
      }
    }

    return body
  },
})
```

The `transform` function runs inside the Copilot auth fetch wrapper, after opencode has built the provider request body and before the request is sent.

## Transform Signature

```ts
type CopilotRequestTransform = (
  url: string,
  headers: Record<string, string>,
  body: any,
) => any | Promise<any>
```

Return the original `body` when no changes are needed. Return a new object when you want to modify the outgoing request.

## Notes

- This hook only applies to the GitHub Copilot provider loaded by this plugin.
- The body shape depends on the model API being used. Common shapes include `body.messages` and `body.input`.
- Because the transform runs before this plugin derives Copilot headers, body changes can affect headers such as `x-initiator`.
- Avoid logging full request bodies unless you have handled secrets and private code safely.
