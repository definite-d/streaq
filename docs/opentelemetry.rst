OpenTelemetry Tracing
=====================

The |otel|_ package provides automated, end-to-end distributed tracing for
your streaQ task queues. It hooks directly into task lifecycles to capture producer
and consumer metrics, propagate trace contexts across network boundaries via Redis
Streams, and automatically track execution failures.

.. |otel| replace:: ``opentelemetry-instrumentation-streaq``
.. _otel: https://github.com/definite-d/opentelemetry-instrumentation-streaq

.. note::
   This integration is an independently maintained third-party package and is not part
   of the official ``opentelemetry-python-contrib`` repository.

   Because it is a third-party package, the ``opentelemetry-bootstrap -a install``
   command will **not** automatically find and install this package. It must be manually
   added to your project dependencies before you can use ``opentelemetry-instrument``.

Features
--------

* **Distributed Tracing:** Automatically creates producer and consumer spans for task queue operations.
* **Context Propagation:** Seamlessly propagates trace context from task producers to workers.
* **Semantic Conventions:** Follows OpenTelemetry messaging semantic conventions (``messaging.system="redis"``).
* **Error Tracking:** Records exceptions and task failures with full stack traces.

Installation
------------

Install the instrumentation package via ``pip``:

.. code-block:: bash

   pip install opentelemetry-instrumentation-streaq

Or with the optional instruments dependency:

.. code-block:: bash

   pip install opentelemetry-instrumentation-streaq[instruments]

Requirements
~~~~~~~~~~~~

* **Python:** 3.10+
* **streaQ:** ``>= 6.4.0, < 7.0``
* **OpenTelemetry API:** ``~= 1.12``

How It Works
------------

The instrumentation patches streaQ at two key points:

1. **Producer Side** (``Task._enqueue``): Creates ``PRODUCER`` spans when tasks are enqueued and injects trace context into task metadata.
2. **Consumer Side** (``Worker.run_task``): Extracts trace context and creates ``CONSUMER`` spans when tasks are processed by workers.

This ensures complete end-to-end tracing across the distributed task queue.

Usage
-----

Quick Start
~~~~~~~~~~~

Create a worker module (``worker.py``):

.. code-block:: python

   from streaq import Worker
   from opentelemetry.instrumentation.streaq import StreaqInstrumentor
   from opentelemetry.sdk.trace import TracerProvider
   from opentelemetry.sdk.trace.export import BatchSpanProcessor, ConsoleSpanExporter

   # Instrument streaQ
   tracer_provider = TracerProvider()
   tracer_provider.add_span_processor(BatchSpanProcessor(ConsoleSpanExporter()))
   StreaqInstrumentor().instrument(tracer_provider=tracer_provider)

   worker = Worker(redis_url="redis://localhost")

   @worker.task
   async def my_task(data: str) -> str:
       return f"Processed: {data}"

Run the worker:

.. code-block:: bash

   streaq run worker:worker

Queue a task (``script.py``):

.. code-block:: python

   from anyio import run
   from worker import worker, my_task

   async def main():
       async with worker:
           await my_task.enqueue("hello")

   run(main)

With Custom Tracer Provider
~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: python

   from opentelemetry.sdk.trace import TracerProvider
   from opentelemetry.instrumentation.streaq import StreaqInstrumentor

   provider = TracerProvider()
   StreaqInstrumentor().instrument(tracer_provider=provider)

Combining with Manual Instrumentation
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: python

   from opentelemetry import trace

   @worker.task
   async def my_task(data: str) -> str:
       tracer = trace.get_tracer(__name__)
       with tracer.start_as_current_span("business_logic"):
           return f"Processed: {data}"

Disabling Instrumentation
~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: python

   from opentelemetry.instrumentation.utils import disable_instrumentation

   disable_instrumentation("streaq")

Span Attributes
---------------

Producer Span Attributes
~~~~~~~~~~~~~~~~~~~~~~~~

These attributes are captured when tasks are enqueued:

+------------------------------+------------+-------------------------------------------------------+
| Attribute Key                | Type       | Description                                           |
+==============================+============+=======================================================+
| ``messaging.system``         | string     | Always ``"redis"``                                    |
+------------------------------+------------+-------------------------------------------------------+
| ``messaging.operation.type`` | string     | Always ``"send"``                                     |
+------------------------------+------------+-------------------------------------------------------+
| ``messaging.destination.name``| string    | The queue name (e.g., ``"normal"``)                   |
+------------------------------+------------+-------------------------------------------------------+
| ``messaging.message.id``     | string     | Unique message identifier                             |
+------------------------------+------------+-------------------------------------------------------+
| ``messaging.operation.name`` | string     | Name of the task function                             |
+------------------------------+------------+-------------------------------------------------------+
| ``streaq.task.max_retries``  | int        | Maximum retry attempts                                |
+------------------------------+------------+-------------------------------------------------------+
| ``streaq.task.delay_ms``     | int        | Task delay in milliseconds                            |
+------------------------------+------------+-------------------------------------------------------+
| ``streaq.task.timeout_ms``   | int        | Task timeout in milliseconds                          |
+------------------------------+------------+-------------------------------------------------------+
| ``streaq.task.ttl_ms``       | int        | Task TTL in milliseconds                              |
+------------------------------+------------+-------------------------------------------------------+
| ``streaq.task.expire_ms``    | int        | Task expiration in milliseconds                       |
+------------------------------+------------+-------------------------------------------------------+
| ``streaq.task.unique``       | boolean    | Whether task is unique                                |
+------------------------------+------------+-------------------------------------------------------+
| ``streaq.task.dependencies`` | string[]   | Dependency task IDs                                   |
+------------------------------+------------+-------------------------------------------------------+
| ``streaq.task.crontab``      | string     | Crontab schedule (if scheduled)                       |
+------------------------------+------------+-------------------------------------------------------+
| ``streaq.task.scheduled_time``| string    | Scheduled execution time (if delayed)                 |
+------------------------------+------------+-------------------------------------------------------+

Consumer Span Attributes
~~~~~~~~~~~~~~~~~~~~~~~~

These attributes are captured when tasks are executed:

+------------------------------+------------+-------------------------------------------------------+
| Attribute Key                | Type       | Description                                           |
+==============================+============+=======================================================+
| ``messaging.system``         | string     | Always ``"redis"``                                    |
+------------------------------+------------+-------------------------------------------------------+
| ``messaging.operation.type`` | string     | Always ``"process"``                                  |
+------------------------------+------------+-------------------------------------------------------+
| ``messaging.destination.name``| string    | The queue name                                        |
+------------------------------+------------+-------------------------------------------------------+
| ``messaging.message.id``     | string     | Unique message identifier                             |
+------------------------------+------------+-------------------------------------------------------+
| ``messaging.operation.name`` | string     | Name of the task function                             |
+------------------------------+------------+-------------------------------------------------------+
| ``messaging.consumer.id``    | string     | Worker consumer identifier                            |
+------------------------------+------------+-------------------------------------------------------+
| ``streaq.task.retry_count``  | int        | Current retry attempt                                 |
+------------------------------+------------+-------------------------------------------------------+
| ``streaq.task.timeout_ms``   | int        | Task timeout in milliseconds                          |
+------------------------------+------------+-------------------------------------------------------+
| ``error.type``               | string     | Exception class name (on failure only)                |
+------------------------------+------------+-------------------------------------------------------+

Completion Attributes
~~~~~~~~~~~~~~~~~~~~~

These attributes are added to consumer spans after task execution:

+-------------------------------------+------------+--------------------------------------------------+
| Attribute Key                       | Type       | Description                                      |
+=====================================+============+==================================================+
| ``streaq.task.success``             | boolean    | Whether task execution succeeded                 |
+-------------------------------------+------------+--------------------------------------------------+
| ``streaq.task.execution_duration_ms``| int       | Task execution duration in milliseconds          |
+-------------------------------------+------------+--------------------------------------------------+
| ``streaq.task.start_time``          | string     | Task start time (ISO format)                     |
+-------------------------------------+------------+--------------------------------------------------+
| ``streaq.task.finish_time``         | string     | Task finish time (ISO format)                    |
+-------------------------------------+------------+--------------------------------------------------+
| ``streaq.task.result_ttl``          | int        | Result TTL in milliseconds                       |
+-------------------------------------+------------+--------------------------------------------------+

API Reference
-------------

StreaqInstrumentor
~~~~~~~~~~~~~~~~~~

.. py:class:: StreaqInstrumentor(BaseInstrumentor)

   The main instrumentor class for streaQ.

   .. py:method:: instrument(**kwargs)

      Enable streaQ instrumentation.

      :param tracer_provider: Custom tracer provider. If not provided, uses the global provider.
      :param meter_provider: Custom meter provider for metrics (future).

   .. py:method:: uninstrument()

      Disable streaQ instrumentation and remove all patches.

Utilities
~~~~~~~~~

.. py:function:: inject_metadata(task_kwargs, metadata)

   Inject trace context metadata into task kwargs.

.. py:function:: extract_metadata(task_kwargs)

   Extract trace context metadata from task kwargs.

   :returns: Dict containing ``traceparent`` and ``tracestate`` if found.

.. py:data:: OTEL_METADATA_KEY
   :type: string
   :value: "__otel_metadata"

   The key used to store trace context in task kwargs (``"__otel_metadata"``).
