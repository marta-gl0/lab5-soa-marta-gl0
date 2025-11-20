# Lab 5 Integration and SOA - Project Report

## 1. EIP Diagram (Before)

![Before Diagram](diagrams/before.png)

The starter code implements several message flows, but the overall integration logic contains multiple flaws that prevent correct routing and processing.

The main message source follows the Polling Consumer pattern: an `AtomicInteger` producer (`integerSource`) emits sequential integers starting at 0, incrementing every 100 ms. These values are intended to be routed based on parity, either to `evenChannel` or `oddChannel`. Additionally, a scheduled method emits random negative numbers every second through a Messaging Gateway, directly into `evenChannel`.

The critical issues detected are the following:

**1. Missing intermediate channel:** 
Messages from the polling source were sent directly to the router instead of passing through a proper intermediate channel, breaking the expected flow structure.

**2. Incorrect odd-channel filter:**
The filter on the odd channel was implemented incorrectly. It accepted values where `p % 2 == 0` instead of rejecting them. As a result, all valid odd numbers are filtered out, and because the discard channel is not connected, these messages simply disappear.

**3. Gateway bypassing the router:**
The Messaging Gateway injects messages straight into `evenChannel`, bypassing all routing and parity logic, meaning gateway-emitted negative numbers are always treated as even.

Overall, these issues result in odd numbers being lost entirely and gateway messages being classified incorrectly, leading to broken routing behavior throughout the system.

---

## 2. Analysis of Issues and Implementation Changes

Below are the defects found, the EIP pattern each relates to, the problem they caused, why they happened, and how they have been fixed.

**Bug 1 - Multiple ingress paths; no unified entry channel**

- **EIP involved:** Unified ingress / Single Routing Entry (Messaging Gateway + Polling Consumer + Router)

- **Problem:** The polled flow and the Gateway used different entry points: the poller sent messages through the router, while the Gateway injected messages directly into `evenChannel`, bypassing routing.

- **Why it happened:** There was no dedicated input channel to unify all message sources. The Gateway was configured to send messages to `evenChannel`, and the poller produced directly into the routing flow, creating two separate ingress paths.

- **How it has been fixed:**

    - Created a shared `numberChannel` to serve as the single entry point.

    - Created a separate `numberFlow` flow that routes the messages from `numberChannel` to `evenChannel` or `oddChannel`.    

    - Adjusted the polling flow `myFlow` to send its messages into `numberChannel`.
    
    - Reconfigured the Gateway’s `requestChannel` to `numberChannel` instead of `evenChannel`.
    
    This ensures that all messages now pass through the same parity-based routing logic.

**Bug 2 - oddChannel implemented as DirectChannel**

- **EIP involved:** Channel selection (Publish–Subscribe vs Direct Channel)

- **Problem:** `oddChannel` was configured as a `DirectChannel`, which delivers each message to only one subscriber.

- **Why it happened:** The flow mistakenly used a direct channel even though multiple consumers were intended to receive every odd message. As a result, Spring Integration’s direct-channel load balancing sent odd messages to only one handler at a time.

- **How it has been fixed:**

    - Replaced the `DirectChannel` with a `PublishSubscribeChannel` for odd messages.

    - Ensured all odd-processing components receive each odd number consistently.

    This restored the expected fan-out delivery semantics for the odd flow.

**Bug 3 - Wrong filter logic in odd flow**

- **EIP involved:** Filter (Message Filter Pattern)

- **Problem:** The odd flow’s filter used the predicate `p % 2 == 0`, which incorrectly passed even numbers instead of odd ones. The discard channel was also disconnected, causing rejected messages to vanish.

- **Why it happened:** The filter logic was inverted and unnecessary. After routing, parity was already guaranteed; adding a second, incorrect filter introduced redundancy and caused all legitimate odd messages to be dropped with no observable trail.

- **How it has been fixed:**

    - Deleted the configured filter in `oddFlow`.

    This prevented message loss.

**Bug 4 - Unused discardChannel and discard flow**

- **EIP involved:** Discard Channel

- **Problem:** A discard channel and related flow existed but were never connected to active filters. As a result, it was unused and filtered-out messages were not being logged or preserved.

- **Why it happened:** The discard path was commented out and never wired to the filter’s discardChannel attribute. When the faulty odd filter rejected messages, they had nowhere to go.

- **How it has been fixed:**
    
    - Deleted `discardChannel` and the logic related to its flow.

    This assured all the existing components are used.

---

## 3. Learning Outcomes

**What I learned about EIP:**

The most important aspect I learned is that maintaining a clear separation between message ingress, routing, and processing is crucial. It also became evident that choosing the correct channel semantics is essential, since using the wrong type can introduce subtle and hard-to-trace bugs. Likewise, filters must be applied with care; once a router has already constrained which messages enter a flow, adding redundant filters can unintentionally result in message loss.

**How Spring Integration works:**

Spring Integration provides a near one-to-one mapping to EIPs via its DSL: `IntegrationFlow`, `route`, `filter`, `transform`, and other terms used make it straightforward to express classical EIP architectures declaratively. I gained experience in understanding and using these elements.

**Challenges:**

The most challenging part of the project was understanding why the system behaved incorrectly, especially since multiple EIPs interacted in subtle ways. This required mentally simulating the routing logic, reconstructing the message paths from the logs, and analyzing how channel types affected delivery semantics.

Overall, the project provided valuable hands-on experience in designing, debugging, and extending a real integration system using classical Enterprise Integration Patterns.

---

## 4. AI Disclosure

**Did you use AI tools?**
Yes — I used ChatGPT.

**How it helped:**

- It assisted in improving the clarity, structure, and readability of the report.

**What I completed on my own:**

- I carried out the full analysis of the starter code and identified the issues in the integration flows.

- I implemented all modifications myself, including the routing redesign.

- I designed the final integration architecture, verified that each EIP pattern was applied correctly, tested the system manually, and resolved all unexpected behaviors.

- I produced the technical content of the report and created the requested diagram.

**My understanding of the code and patterns:**

I have a solid grasp of the following concepts:

- How messages enter through both the polling source and the gateway.

- How `numberChannel` serves as the unified ingress point.

- How routing decisions are made.

- How publish–subscribe channels distribute messages to multiple consumers. 

While AI tools helped me articulate the explanations more effectively; the architectural decisions, implementation work, and debugging were entirely performed based on my own understanding.

---

## Additional Notes

The final implementation now aligns with the expected EIP architecture. All messages enter through a single input channel (`numberChannel`), and a centralized router reliably directs them based on their content. The odd-number processing path correctly uses a Publish–Subscribe channel, ensuring every subscriber receives the same message. The incorrect filter logic has been removed, and unnecessary discard components are no longer part of the flow. As a result, the overall message pipeline behaves predictably, remains easy to trace, and fully reflects the requested EIP design.
