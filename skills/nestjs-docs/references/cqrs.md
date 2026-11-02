# CQRS (@nestjs/cqrs)

- Import `CqrsModule.forRoot()` in the root module, not bare `CqrsModule`. https://docs.nestjs.com/recipes/cqrs#installation
- Typed results: `class KillDragonCommand extends Command<{ actionId: string }>` (or `Query<Hero>`) lets `CommandBus#execute()` infer the return type. Every handler must be registered as a provider. https://docs.nestjs.com/recipes/cqrs#commands
- Aggregates: `extends AggregateRoot`, `this.apply(event)`, then `publisher.mergeObjectContext(hero)` and `hero.commit()`. Events queue until `commit()` unless `autoCommit = true`. Mixin `WithAggregateRoot()` fits existing hierarchies. https://docs.nestjs.com/recipes/cqrs#events
- From `@nestjs/cqrs` 12.1, `commit()`, `publish()` and `publishAll()` return what the publisher returns and accept a dispatcher context: `await hero.commit({ transaction: tx })`. The default in-memory publisher returns nothing and does not wait for handlers. https://docs.nestjs.com/recipes/cqrs#events
- `commit()` clears the model's events as soon as it hands them over: a failed awaited `commit()` means roll back and rerun the command with a freshly loaded model. https://docs.nestjs.com/recipes/cqrs#events
- Event handlers run outside the request: exception filters do not catch their errors and they cannot send HTTP responses. https://docs.nestjs.com/recipes/cqrs#manual-event-publishing Unhandled errors go to `UnhandledExceptionBus` (or are rethrown with `rethrowUnhandled: true`). https://docs.nestjs.com/recipes/cqrs#unhandled-exceptions
- Sagas and event handlers are in memory: an event before a crash, or one published before a rollback, can be lost or wrongly acted on. For durable reactions map events to workflows: https://docs.nestjs.com/recipes/cqrs#durable-sagas
  ```typescript
  @Workflow('order-fulfilment')
  @StartOn(OrderPlacedEvent, { id: (event) => `order-${event.order.id}`, input: (event) => event.order })
  export class OrderFulfilmentWorkflow {}
  // command handler
  await this.eventBus.publish(new OrderPlacedEvent(order), { transaction: tx });
  ```
- Import `WorkflowsCqrsModule` from `@nestjs/workflows/cqrs` next to `CqrsModule` and `WorkflowsModule`, in every process that publishes mapped events, including one that runs no workflows, such as an API that only places orders (pass `worker: false` to `WorkflowsModule.forRoot()` there). https://docs.nestjs.com/reliability/workflows#map-events-to-workflows
- With `@nestjs/cqrs` 12.0 or earlier, `commit()` takes no context: `await this.eventBus.publishAll(order.getUncommittedEvents(), { transaction: tx }); order.uncommit();`. https://docs.nestjs.com/reliability/workflows#what-each-way-of-publishing-guarantees
- Request scoping: handlers can be `@CommandHandler(Cmd, { scope: Scope.REQUEST })`. The `REQUEST` payload must be an `AsyncContext` passed as `commandBus.execute(cmd, ctx)`. Sagas are always singletons. https://docs.nestjs.com/recipes/cqrs#request-scoping
