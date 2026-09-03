---
name: medusa-development
description: "Best practices for building commerce applications with Medusa v2, the headless e-commerce framework. Use when defining Medusa data models, writing workflows and steps with the Workflow SDK, creating API routes or subscribers, building module services that extend MedusaService, throwing MedusaError, or customizing the Medusa admin dashboard."
---

# Medusa Development

Medusa is a headless commerce framework built around modules, data models, and workflows; almost every piece of business logic — from an API route to a scheduled job — should be expressed as a workflow made of discrete, composable steps.

## Workflow for Building a Medusa Feature

1. **Define or extend the data model** — Use the `model` utility from `@medusajs/framework/utils` to declare the module's data model(s) under `src/modules/<module>/models/`.
2. **Write the module service** — Create a service in `src/modules/<module>/service.ts` that extends `MedusaService` when the module has data models, exposing async methods for domain operations.
3. **Register the module** — Add the module to `medusa-config.ts` so Medusa's container can resolve it.
4. **Build steps** — Define each unit of work as a step with `createStep` from `@medusajs/framework/workflows-sdk`, including a compensation function for anything that needs to be undone on failure.
5. **Compose the workflow** — Wire steps together with `createWorkflow`, using `transform` for data shaping and `when` for conditional branches.
6. **Expose the workflow** — Call the workflow from an API route, a scheduled job, or a subscriber — never put business logic directly in the route/job/subscriber handler.
7. **Read data with Query** — Use Medusa's Query (`req.scope.resolve("query")` or the workflow-level `useQueryGraphStep`) to fetch data instead of calling module services directly for reads.

## General Rules

- Don't use type aliases when importing files — import types and values directly from their source module rather than re-exporting through a local alias.
- When throwing errors, always throw `MedusaError` (from `@medusajs/framework/utils`) instead of a plain `Error`, so the API layer can map it to the correct HTTP status and error code.
- Always use Query to retrieve data rather than calling a module's service methods directly for reads — Query understands module links and can join data across modules in one call.

```ts
import { MedusaError } from "@medusajs/framework/utils"

if (!product) {
  throw new MedusaError(
    MedusaError.Types.NOT_FOUND,
    `Product with id "${productId}" was not found`
  )
}
```

## Data Model Rules

- Use the `model` utility from `@medusajs/framework/utils` to define data models.
- Data model variables should be camelCase; the name passed to `model.define` should be snake_case.
- When adding an `id` field to a data model, always make it a primary key with `.primaryKey()`.
- A data model can have only one `id` field — any other identifier should be a `text` field instead.
- Data model fields should be snake_case.

```ts
// src/modules/loyalty/models/loyalty-account.ts
import { model } from "@medusajs/framework/utils"

const LoyaltyAccount = model.define("loyalty_account", {
  id: model.id().primaryKey(),
  customer_id: model.text(),
  points_balance: model.number().default(0),
  tier: model.enum(["bronze", "silver", "gold"]).default("bronze"),
})

export default LoyaltyAccount
```

## Service Rules

- When creating a service, always make its methods async.
- If a module has data models, make the service extend `MedusaService` so it inherits generated CRUD methods for each model.

```ts
// src/modules/loyalty/service.ts
import { MedusaService } from "@medusajs/framework/utils"
import LoyaltyAccount from "./models/loyalty-account"

class LoyaltyModuleService extends MedusaService({
  LoyaltyAccount,
}) {
  async addPoints(accountId: string, points: number) {
    const account = await this.retrieveLoyaltyAccount(accountId)
    return await this.updateLoyaltyAccounts({
      id: account.id,
      points_balance: account.points_balance + points,
    })
  }
}

export default LoyaltyModuleService
```

## Workflow Rules

- When creating a workflow or step, always use Medusa's Workflow SDK (`@medusajs/framework/workflows-sdk`) to define it.
- When creating a feature in an API route, scheduled job, or subscriber, always create a workflow for it rather than inlining the logic in the handler.
- When creating a workflow, always create a step for each discrete unit of work in it.
- In workflows, use `transform` for any data transformation between steps — don't manipulate step output directly in the workflow function body.
- In workflows, use `when` to define conditional branches instead of a plain `if` around step calls.
- Don't use `await` when calling steps inside a workflow — step invocation returns a special reference the workflow engine resolves, and `await`-ing it breaks the workflow's ability to orchestrate compensation and retries.
- In workflows, don't make the workflow function itself `async` — the function body only describes the step graph, it doesn't execute imperatively.
- Don't add typing to a compensation function's input — the compensation function receives whatever the step's invoke function returned, and Medusa infers this automatically.
- Only use steps in a workflow — don't call services, Query, or other side-effecting code directly inside the workflow function; put that logic in a step.

```ts
// src/workflows/redeem-loyalty-points.ts
import {
  createStep,
  createWorkflow,
  StepResponse,
  transform,
  when,
  WorkflowResponse,
} from "@medusajs/framework/workflows-sdk"
import { MedusaError } from "@medusajs/framework/utils"
import { LOYALTY_MODULE } from "../modules/loyalty"
import LoyaltyModuleService from "../modules/loyalty/service"

type RedeemPointsInput = {
  accountId: string
  points: number
}

const deductPointsStep = createStep(
  "deduct-points-step",
  async (input: RedeemPointsInput, { container }) => {
    const loyaltyService: LoyaltyModuleService = container.resolve(LOYALTY_MODULE)
    const account = await loyaltyService.retrieveLoyaltyAccount(input.accountId)

    if (account.points_balance < input.points) {
      throw new MedusaError(
        MedusaError.Types.INVALID_DATA,
        "Insufficient points balance"
      )
    }

    const previousBalance = account.points_balance
    const updated = await loyaltyService.updateLoyaltyAccounts({
      id: account.id,
      points_balance: previousBalance - input.points,
    })

    return new StepResponse(updated, { accountId: account.id, previousBalance })
  },
  async (compensationInput, { container }) => {
    if (!compensationInput) return
    const loyaltyService: LoyaltyModuleService = container.resolve(LOYALTY_MODULE)
    await loyaltyService.updateLoyaltyAccounts({
      id: compensationInput.accountId,
      points_balance: compensationInput.previousBalance,
    })
  }
)

export const redeemLoyaltyPointsWorkflow = createWorkflow(
  "redeem-loyalty-points",
  (input: RedeemPointsInput) => {
    const account = deductPointsStep(input)

    const tierDowngrade = when(account, (acc) => acc.points_balance === 0)
      .then(() => {
        return transform({ account }, (data) => ({
          ...data.account,
          tier: "bronze" as const,
        }))
      })

    return new WorkflowResponse(account)
  }
)
```

## API Routes and Reading Data

- Expose workflows through API routes under `src/api/`; the route handler should validate input, call the workflow, and shape the HTTP response — nothing more.
- Always use Query to retrieve data for reads (list/detail endpoints) instead of resolving a module service directly.

```ts
// src/api/store/loyalty/[id]/route.ts
import type { MedusaRequest, MedusaResponse } from "@medusajs/framework/http"

export async function GET(req: MedusaRequest, res: MedusaResponse) {
  const query = req.scope.resolve("query")

  const { data: accounts } = await query.graph({
    entity: "loyalty_account",
    fields: ["id", "points_balance", "tier"],
    filters: { id: req.params.id },
  })

  res.json({ loyalty_account: accounts[0] })
}

export async function POST(req: MedusaRequest, res: MedusaResponse) {
  const { result } = await redeemLoyaltyPointsWorkflow(req.scope).run({
    input: req.validatedBody as { accountId: string; points: number },
  })

  res.json({ loyalty_account: result })
}
```

## Subscribers and Scheduled Jobs

- Subscribers and scheduled jobs should call a workflow, exactly like API routes — they are just a different trigger for the same business logic.

```ts
// src/subscribers/order-placed.ts
import type { SubscriberArgs, SubscriberConfig } from "@medusajs/framework"
import { redeemLoyaltyPointsWorkflow } from "../workflows/redeem-loyalty-points"

export default async function orderPlacedHandler({ event, container }: SubscriberArgs<{ id: string }>) {
  await redeemLoyaltyPointsWorkflow(container).run({
    input: { accountId: event.data.id, points: 0 },
  })
}

export const config: SubscriberConfig = { event: "order.placed" }
```

## Admin Customization Rules

- When sending requests from admin customizations (widgets, custom pages), always use Medusa's JS SDK (`@medusajs/js-sdk`) rather than raw `fetch`.
- Use TailwindCSS for styling admin customizations, matching the conventions of Medusa Admin's own UI.

```tsx
// src/admin/widgets/loyalty-widget.tsx
import { defineWidgetConfig } from "@medusajs/admin-sdk"
import { useQuery } from "@tanstack/react-query"
import { sdk } from "../lib/sdk"

const LoyaltyWidget = ({ data }: { data: { id: string } }) => {
  const { data: result } = useQuery({
    queryFn: () => sdk.admin.customer.retrieve(data.id),
    queryKey: ["customer", data.id],
  })
  return <p className="text-ui-fg-subtle">{result?.customer.email}</p>
}

export const config = defineWidgetConfig({ zone: "customer.details.after" })
export default LoyaltyWidget
```

## Common Mistakes

- Calling a module service directly from an API route handler instead of going through a workflow — this skips retries, compensation, and the standard event/observability hooks workflows provide.
- Throwing a plain `Error` instead of `MedusaError`, which loses the mapped HTTP status code and structured error type on the API response.
- Awaiting a step call inside a workflow function, which breaks the workflow engine's ability to build the step graph.
- Reading data by resolving a module service instead of using Query, which misses cross-module joins and links that Query resolves automatically.
- Naming a data model field or `model.define` name in camelCase instead of snake_case, causing inconsistency with the rest of the schema.

## Additional Resources

- Medusa Documentation: https://docs.medusajs.com/llms-full.txt
