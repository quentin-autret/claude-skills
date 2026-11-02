# NestJS (@nestjs/bullmq)

## Module

Configure the connection once with `forRoot`, then register each queue and flow producer by name.

```typescript
BullModule.forRoot({ connection: { host: 'localhost', port: 6379 } });
BullModule.registerQueue({ name: 'queueName' });
BullModule.registerFlowProducer({ name: 'flowProducerName' });
```
https://docs.bullmq.io/guide/nestjs

## Consumers

A consumer is a class decorated with `@Processor('queueName')` that extends `WorkerHost` and implements `process(job)`. Worker events use `@OnWorkerEvent`. Register the class as a provider.

```typescript
@Processor('queueName')
class TestProcessor extends WorkerHost {
  async process(job: Job<any, any, string>): Promise<any> {
    // do some stuff
  }

  @OnWorkerEvent('completed')
  onCompleted() {}
}
```
https://docs.bullmq.io/guide/nestjs

## Producers

Inject with `@InjectQueue(name)` and `@InjectFlowProducer(name)`. The name must match `registerQueue()` or `registerFlowProducer()`.

```typescript
constructor(@InjectQueue('audio') private audioQueue: Queue) {}
await this.audioQueue.add('sample', { foo: 'bar' });

constructor(@InjectFlowProducer('flow') private fooFlowProducer: FlowProducer) {}
```
https://docs.bullmq.io/guide/nestjs/producers

## Queue-wide events

For events from every worker, decorate a class with `@QueueEventsListener('queueName')`, extend `QueueEventsHost` and use `@OnQueueEvent`. Register it as a provider. To type payloads with BullMQ's `QueueEventsListener` type, import it under an alias: the `@nestjs/bullmq` decorator has the same name.

```typescript
import {
  QueueEventsListener,
  QueueEventsHost,
  OnQueueEvent,
} from '@nestjs/bullmq';
import type { QueueEventsListener as QueueEventsListenerType } from 'bullmq';

type CompletedEvent = QueueEventsListenerType<{ result: number }>['completed'];
type CompletedEventArgs = Parameters<CompletedEvent>[0];

@QueueEventsListener('queueName')
export class TestQueueEvents extends QueueEventsHost {
  @OnQueueEvent('completed')
  onCompleted({ jobId, returnvalue }: CompletedEventArgs) {}
}
```
https://docs.bullmq.io/guide/nestjs/queue-events-listeners
