# Functions SDK Integration

Use Yarah SDK to invoke serverless functions from your frontend application.

## Setup

First, ensure your `.env` file is configured with your Yarah URL and anon key. Get the anon key with `npx -y @yarahdev/cli secrets get ANON_KEY`. See the main [SKILL.md](../SKILL.md) for framework-specific variable names and full setup steps.

```javascript
import { createClient } from '@yarahdev/sdk'

const yarah = createClient({
  baseUrl: process.env.NEXT_PUBLIC_YARAH_URL,       // adjust prefix for your framework
  anonKey: process.env.NEXT_PUBLIC_YARAH_ANON_KEY   // adjust prefix for your framework
})
```

## Invoke Function

```javascript
// POST with body (default)
const { data, error } = await yarah.functions.invoke('hello-world', {
  body: { name: 'World' }
})

// GET request
const { data, error } = await yarah.functions.invoke('get-stats', {
  method: 'GET'
})

// With custom headers
const { data, error } = await yarah.functions.invoke('api-endpoint', {
  method: 'PUT',
  body: { id: '123', status: 'active' },
  headers: { 'X-Custom-Header': 'value' }
})
```

## Important Notes

- SDK automatically includes user's auth token
- All methods return `{ data, error }` - always check for errors
- Function must be deployed and have `status: "active"` to be invokable

---

## Best Practices

1. **Verify function exists and is active before invoking**
   - Check available functions via CLI: `yarah functions list`
   - If no functions exist, create one first via admin API
   - Ensure function `status` is `"active"` (not `"draft"`)

2. **Handle errors gracefully**
   - Functions may fail due to code errors or timeouts
   - Always check `error` in response

## Common Mistakes

| Mistake | Solution |
|---------|----------|
| ❌ Invoking without checking function exists | ✅ Verify function via admin API first |
| ❌ Invoking a draft function | ✅ Ensure function status is `"active"` |
| ❌ Ignoring errors | ✅ Always handle `error` in response |

## Recommended Workflow

```
1. Check available functions → yarah functions list
2. If no function exists     → Create one first with status: "active"
3. Invoke function           → Use function slug
4. Handle response           → Check for errors
```
