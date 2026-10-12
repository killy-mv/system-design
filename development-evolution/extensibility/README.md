# Extensibility

> **Extensibility** = new behaviour can be **added without changing existing code**, often by people outside the core team (plugins, integrations, customers).
> "Can a partner add a new shipping carrier, or a customer react to 'order paid', without us editing and redeploying the core?"

Related: [Modifiability](../modifiability/README.md) (changing existing code safely), [Modularity](../modularity/README.md), [Simplicity](../simplicity/README.md) (the main risk: unnecessary extension points), [Evolvability](../evolvability/README.md).

---

## 1. Extension mechanisms

| Mechanism | How it works | Examples |
|---|---|---|
| **Strategy / interface + registry** | New implementations of an interface, registered by name | Payment methods, export formats, storage backends |
| **Hooks / events** | Core announces "something happened" or "about to do X"; extensions subscribe | WordPress actions and filters, Git hooks, domain events |
| **Plugins** | Separately packaged code loaded by the host, using a published plugin API | VS Code extensions, browser extensions, Grafana plugins, Terraform providers |
| **Webhooks** | Your system calls external URLs on events | Stripe, GitHub, Shopify webhooks |
| **Public APIs** | Others build on top of your data and actions | REST/GraphQL APIs, SDKs |
| **Configuration / rules** | Behaviour changed by data, not code | Pricing rules, workflow definitions, feature flags |
| **Scripting / embedded languages** | Users write small programs the host runs | Lua in Redis and Nginx (OpenResty), Excel formulas, Postgres extensions |
| **Middleware / pipelines** | Chain of handlers that can be added or reordered | HTTP middleware, Express/Koa, ASP.NET pipeline |

The **Open/Closed principle** sums it up: open for extension, closed for modification.

### Worked example: WordPress hooks

WordPress has tens of thousands of plugins, and almost none of them modify WordPress's own code. The core calls **hooks** at important moments:

```php
// Core, somewhere in rendering a post:
$content = apply_filters('the_content', $content);   // a filter: extensions can change a value
do_action('save_post', $post_id, $post);             // an action: extensions can react

// A plugin, in its own file:
add_filter('the_content', function ($content) {
    return $content . '<p>Thanks for reading!</p>';
});
add_action('save_post', 'notify_editors', 10, 2);
```

Thousands of independent developers extend behaviour without coordinating with the core team. The costs show up too: plugins depend on hook names and arguments forever (changing them breaks the ecosystem), plugin conflicts and ordering issues, and performance or security problems from third-party code running in-process.

---

## 2. Designing extension points well

- **Add them when there's real demand**: the second or third concrete use case, or a known ecosystem need. An extension point nobody uses is pure complexity (YAGNI).
- **Treat them as public APIs**: document, version, and keep them compatible; deprecate slowly.
- **Small, stable surface**: expose events and narrow interfaces, not your internals.
- **Isolate extensions**: separate processes, sandboxes or WebAssembly for untrusted code (VS Code runs extensions in a separate extension host process); timeouts and resource limits; failures in an extension must not crash the core.
- **Clear ordering and conflict rules** for hooks (priorities, documented order).
- **Security**: permissions per extension, signed packages, review for marketplaces.
- **Observability**: know which extension did what (timing and errors per plugin).

---

## 3. Anti-patterns

- **Speculative plugin architecture** for a product with one implementation.
- **Inner platform effect**: making the system so configurable it becomes a bad programming language.
- Extension points that expose internal classes, so internals can never change.
- Untrusted extensions running with full access in the core process.
- Breaking hook/plugin APIs without versioning.

---

## 4. Checklist

- [ ] Extension points exist only where there are real use cases
- [ ] Each extension point is documented, versioned and treated as a public API
- [ ] Extensions are isolated (process/sandbox, timeouts, permissions) when untrusted
- [ ] Extension failures can't take down the core
- [ ] Per-extension telemetry

---

## 5. Exercises

1. List the extension points in a product you use (VS Code, WordPress, Shopify, Grafana). Which mechanism does each use?
2. Design webhooks for messy-shop's order events: payload, signing, retries, versioning (see [Reliability §10.5](../../runtime-operational/reliability/README.md#105-webhooks-sending-to-external-systems)).
3. Find an extension point in your code that's never been used by more than one implementation. Would removing it make the code simpler?
4. A third-party plugin can slow down every page render. Design limits and isolation to protect the core.

---

## 6. Further reading

WordPress Plugin Handbook ("Hooks"); VS Code docs, "Extension Host"; Martin Fowler, "Inner Platform Effect" discussions and *Patterns of Enterprise Application Architecture* (plugin pattern); Robert C. Martin on the Open/Closed principle; Stripe and GitHub webhook documentation.
