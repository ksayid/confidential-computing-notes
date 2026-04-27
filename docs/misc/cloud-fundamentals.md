---
title: Cloud Fundamentals
parent: Reference
nav_order: 3
---

## Autoscaling
Autoscaling is the process of dynamically allocating and deallocating resources to match an application’s performance requirements.
The goal is to maintain desired performance levels and meet SLAs while minimizing cost. There are two ways to scale:
* Vertical (Scale Up/Down): Increase or decrease the capacity (e.g., VM size). Often requires redeployment and is less common to automate.
* Horizontal (Scale Out/In): Add or remove resource instances, typically without downtime. Most cloud-based autoscaling strategies focus on horizontal scaling.
Use instrumentation and monitoring for key metrics (e.g., CPU, memory, response time, queue length) to decide whether to scale. A threshold or schedule should already be defined and tuned.

### Best Practices
* Always combine scale-out and scale-in rules to ensure two-way scaling.
* Select a safe default instance count and optionally define min/max limits. Plan for graceful degradation if the max is reached.
* Avoid flapping (constant rapid scale in/out) by setting an adequate margin between CPU usage or other trigger thresholds.
* Orders per hour, average execution time of transactions, etc., must align with compute demands.
* Autoscaling can take time (provisioning new instances). Consider throttling or aggressive autoscaling strategies for extreme spikes.
* Monitoring & Logging: Track every autoscale event (what triggered it, resources changed, when it occurred). Use this data to tune rules over time.

### Application design considerations
* Stateless & Horizontally Scalable: The system must be designed to be horizontally scalable. Avoid making assumptions about instance affinity; do not design solutions that require that the code is always running in a specific instance of a process. The additional instances must be able to handle any request or message.
* Long-Running Tasks: Break tasks into smaller chunks or use checkpoints to avoid losing work when scaling in. Use frameworks like NServiceBus or MassTransit for checkpointing.
* Different Scaling Policies: UIs and background processes might require separate autoscaling strategies and thresholds.
* Queue & Critical Time: Beyond simple queue length, consider the time from when a message was sent to when it was processed as a meaningful trigger.
* Balance cost vs. performance needs for your business context.

## Background Jobs
A background job is a task or set of tasks that runs without requiring direct user interaction or blocking the user interface (UI). By offloading lengthy or resource-heavy processes to the background, you free up the UI thread so users can keep working, improving overall responsiveness and user experience.

Ideal Scenarios:
* Long-running computations (e.g., converting large files, data analysis).
* Repetitive batch tasks (e.g., nightly updates, invoice generation).
* Workflows that persist state over time (e.g., multi-step business processes).
* Sensitive data processing in a more secure or isolated context (e.g., using the Gatekeeper pattern).
Not Suitable if a user must see an immediate result (e.g., a quick calculation in a form field).

### Example
A user uploads an image to your web application, and you need to create a smaller thumbnail version.

Without a Background Job:
* The upload triggers the thumbnail generation synchronously.
* The user waits until the thumbnail is created.
* If the process is slow or fails, it affects the user’s experience directly.

With a Background Job:
* The user uploads the image.
* Your app queues a task or sends a message to a background worker.
* The user sees an immediate success message and can continue browsing.
* The background worker processes the image, generates the thumbnail, and saves it.
* (Optionally) The worker updates a database record and sends a completion notification (e.g., email or real-time UI update).

$\implies$ The UI remains responsive, the user is free to do other things, and the app can retry or handle errors internally if something goes wrong during thumbnail creation.

### Triggers
Event-Driven
* Triggered By: User action, workflow step, or message on a queue.
* Examples: Image uploads, form submissions, calls to an internal API.

Schedule-Driven
* Triggered By: A timer or schedule (e.g., every night at midnight, or once after a specified delay).
* Examples: Routine data updates, daily report generation, cleaning stale records.
* Caution: If the scheduler or compute instance scales out, you could unintentionally trigger multiple instances of the same task; consider concurrency control and idempotency.

https://learn.microsoft.com/en-us/azure/well-architected/reliability/design-patterns
https://learn.microsoft.com/en-us/security/zero-trust/apply-zero-trust-azure-services-overview
https://learn.microsoft.com/en-us/azure/architecture/best-practices/background-jobs
https://www.google.com/search?q=%22message+queue%22+site%3A.edu&oq=%22message+queue%22+site%3A.edu&gs_lcrp=EgZjaHJvbWUyBggAEEUYOTIHCAEQIRiPAtIBCDU2NTJqMGoxqAIAsAIA&sourceid=chrome&ie=UTF-8
https://www.google.com/search?q=%22Key-Value+Store%22+site%3A.edu&sca_esv=4e144505dd75033c&sxsrf=ADLYWIKvl6tAAv_zJaGWUe0MjoDg3abifA%3A1736517007911&ei=jyWBZ4WkN_i1wN4Pm8SP0QI&ved=0ahUKEwjFp5z7peuKAxX4GtAFHRviIyoQ4dUDCBA&uact=5&oq=%22Key-Value+Store%22+site%3A.edu&gs_lp=Egxnd3Mtd2l6LXNlcnAiGyJLZXktVmFsdWUgU3RvcmUiIHNpdGU6LmVkdUjUClAAWOkIcAB4AZABAZgB0AGgAZoKqgEFNS41LjG4AQPIAQD4AQGYAgagAt0GwgILEAAYgAQYkQIYigXCAgUQABiABMICBhAAGBYYHsICBRAhGKABwgILEAAYgAQYhgMYigXCAgUQABjvBZgDAJIHAzEuNaAHgis&sclient=gws-wiz-serp
https://samwho.dev/memory-allocation/?utm_source=blog.quastor.org&utm_medium=newsletter&utm_campaign=how-pinterest-stores-and-transfers-hundreds-of-terabytes-of-data-daily
https://github.com/sobolevn/awesome-cryptography?utm_source=blog.quastor.org&utm_medium=newsletter&utm_campaign=how-pinterest-stores-and-transfers-hundreds-of-terabytes-of-data-daily#readme
https://news.ycombinator.com/item?id=42799245
https://stackoverflow.com/questions/3538021/why-do-we-use-base64
https://news.ycombinator.com/item?id=24950437
https://chriskiehl.com/article/thoughts-after-10-years
https://newsletter.techworld-with-milan.com/p/computer-science-papers-every-developer
https://github.com/keyvanakbary/learning-notes/blob/master/books/an-elegant-puzzle.md#useful-papers
https://github.com/chiphuyen/aie-book/blob/main/resources.md
https://www.cse.msu.edu/~cse870/Public/Lectures/SS2010/Notes/09b-design-patterns-notes.pdf
https://web.eecs.umich.edu/~xwangsd/courses/w23/lectures/se-16-patterns.pdf
https://news.ycombinator.com/item?id=42456492
https://www.andrew.cmu.edu/course/14-712-s20/applications/ln/14712-l22.pdf
https://faculty.washington.edu/wlloyd/courses/tcss562/tcss562_lecture_17_f24_2up.pdf
https://websites.umich.edu/~eecs381/lecture/notes.html
https://courses.cs.washington.edu/courses/cse403/24wi/lectures/16-wrapup.pdf
