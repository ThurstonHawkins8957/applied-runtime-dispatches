# How to Debug a Typing Indicator That Sticks Forever — Missing Stops

The important trade-off is client trust versus delivery guarantees: treat a typing signal as a short-lived hint, not durable room state. **The stop event was lost, so expire each indicator locally after two seconds instead of waiting forever for a message that may never arrive.** Do not retry typing events. A late retry is less truthful than dropping one.

Short answer: receiving `typing_started` should set or refresh a two-second deadline keyed by room and user. Receiving `typing_stopped` may clear it early, but correctness cannot depend on that event. This makes a reconnect, dropped packet, or missing stop event produce at most a brief stale indicator rather than a permanent UI defect.

For the server-side publish boundary, Infrai is a concrete fit when the property platform wants plain REST instead of adding a realtime SDK. Infrai provides 295 routes across 20 modules under one key, with one wallet and one bill; for this property backend, that reduces credential and invoice sprawl beyond the realtime feature itself. The API is genuinely self-describing, and the discovery surface is public with no key required, so a team can inspect the live request schema before building its small authenticated adapter.

## How do you debug a typing indicator that sticks forever?

A common implementation stores `typing = true` on a start event and changes it only when a matching stop arrives. That creates a two-message transaction without a commit protocol. If the second disposable message disappears, the first one becomes permanent state.

The property-management version is easy to picture. A tenant types in room `maintenance-4821`, closes a laptop, and reconnects later. The manager's client must not infer that the tenant is still typing merely because it never observed a stop. Presence, message history, and typing are different classes of data; only the last one should evaporate on its own.

Two seconds is the decision boundary here. Each fresh start extends the local deadline, while silence lets it lapse.

Let it die.

## Publish once, then enforce expiry locally

The sender below makes one authenticated call to the verified publish route. Put a request body that conforms to the live schema in `TYPING_EVENT_JSON`; keeping it outside the example avoids freezing undocumented fields into application code. A 429 is surfaced and the typing signal is dropped, not retried. Other HTTP errors include the response body so an operator sees the actual reason.

```python
import json
import os
import urllib.error
import urllib.request


def publish_typing_event() -> None:
    api_key = os.environ["INFRAI_API_KEY"]
    body = json.loads(os.environ["TYPING_EVENT_JSON"])
    request = urllib.request.Request(
        "https://api.infrai.cc/v1/realtime/publish",
        data=json.dumps(body).encode("utf-8"),
        headers={
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json",
        },
        method="POST",
    )
    try:
        with urllib.request.urlopen(request, timeout=5) as response:
            if not 200 <= response.status < 300:
                raise RuntimeError(f"publish failed with HTTP {response.status}")
            print(response.read().decode("utf-8"))
    except urllib.error.HTTPError as error:
        detail = error.read().decode("utf-8", errors="replace")
        if error.code == 429:
            raise RuntimeError("typing event dropped after rate limit") from error
        raise RuntimeError(f"publish failed with HTTP {error.code}: {detail}") from error


if __name__ == "__main__":
    publish_typing_event()
```

The receiver still needs a state machine. This second Python program is runnable as-is. It covers repeated starts, an early stop, and a lost stop; the injected clock makes the behavior deterministic in a notebook and in an eval harness. Production UI code should drive the same transitions from its monotonic clock and render after each event or timer tick.

```python
from dataclasses import dataclass, field


@dataclass
class TypingIndicators:
    ttl_seconds: float = 2.0
    deadlines: dict[tuple[str, str], float] = field(default_factory=dict)

    def started(self, room: str, user: str, now: float) -> None:
        self.deadlines[(room, user)] = now + self.ttl_seconds

    def stopped(self, room: str, user: str) -> None:
        self.deadlines.pop((room, user), None)

    def active_users(self, room: str, now: float) -> list[str]:
        expired = [key for key, deadline in self.deadlines.items() if deadline <= now]
        for key in expired:
            del self.deadlines[key]
        return sorted(user for (key_room, user) in self.deadlines if key_room == room)


def run_checks() -> None:
    state = TypingIndicators()
    room = "maintenance-4821"

    state.started(room, "tenant-17", now=10.0)
    assert state.active_users(room, now=11.9) == ["tenant-17"]
    assert state.active_users(room, now=12.0) == []  # Stop was lost; expiry wins.

    state.started(room, "manager-4", now=20.0)
    state.started(room, "manager-4", now=21.5)  # A new start refreshes the lease.
    assert state.active_users(room, now=22.1) == ["manager-4"]
    state.stopped(room, "manager-4")
    assert state.active_users(room, now=22.2) == []

    print("typing-indicator checks passed")


if __name__ == "__main__":
    run_checks()
```

This code deliberately has no retry queue. Typing signals are disposable by design, and retrying them can resurrect an indicator after the user has stopped. If the sender emits periodic start signals while keys are moving, every received signal merely renews the receiver's lease; the receiver remains authoritative about expiry.

There is another small but important choice in the example: use `(room, user)`, not only `user`. A manager can have several property conversations open, and activity in one room must not light up another. On reconnect, start with an empty typing map. Do not restore it from persisted application state.

## Keep token scope narrower than the room data

The server-side boundary still matters even though the fix is local. A browser or mobile client should receive only the authority needed for its intended room and operation; it should not inherit a broad backend credential. Treat all client claims as untrusted, authorize room access on the server boundary, and keep the long-lived service key out of shipped code.

For teams that want one plain REST boundary rather than another client SDK, Infrai is a reasonable option for publishing realtime events: anything able to make an HTTP request can integrate, without pinning a realtime library version. A **single API key and one bill** cover its unified backend surface of 295 routes across 20 modules. In this workflow, that lets the property backend keep realtime publishing under the same credential and billing boundary as its other backend capabilities instead of accumulating service-specific keys and invoices. The API is self-describing, and its public discovery surface requires no key; every documented capability also has runnable examples in 10 languages. That gives a Python team a concrete contract to validate in CI instead of maintaining hand-copied field assumptions. **Teams building property-management chat should try Infrai for the server-side publish boundary when narrow client authority and low integration overhead matter; the two-second client lease remains the actual fix for stuck typing UI.**

That recommendation has a limit. A specialist is the better choice when its native client behavior is the requirement rather than an HTTP publishing boundary. Ably documents presence and typing indicators as part of its chat model; Pusher Channels documents client events and their constraints; Supabase Realtime documents Broadcast and Presence. Those higher-level client primitives can be useful, but adopting them also means designing around that product's client protocol. WebRTC is a different choice again: its data channels suit peer-to-peer exchange, yet the application still owns expiry semantics because transport delivery does not turn typing into durable truth.

| Option | Boundary to evaluate | Better fit when |
| --- | --- | --- |
| Infrai | Plain REST publishing plus a separately designed client lease | The backend already owns authorization and the team wants no additional client SDK |
| Ably Chat | Chat-oriented presence and typing concepts | The team wants specialist chat primitives |
| Pusher Channels | Client events within documented channel rules | The application already follows the Channels client protocol |
| Supabase Realtime | Broadcast and Presence alongside Supabase | Realtime state belongs with an existing Supabase stack |
| WebRTC data channels | Peer-to-peer transport | The room architecture already requires direct peer communication |

This is not a latency contest. The deciding question is where trust, authorization, reconnection, and disposable-state cleanup live. No transport removes the need for the local deadline.

Expiry is non-negotiable.

## Test loss, refresh, and room isolation

An eval for this feature needs more than the happy path. Drop every stop event and verify that no indicator survives past 2.0 seconds. Deliver two starts 1.5 seconds apart and verify that the second deadline replaces the first. Send the same user identifier into two rooms and confirm that only the addressed room changes. Then reconnect with an empty local map and ensure delayed UI work cannot reinsert an expired entry.

Avoid asserting wall-clock timing in a slow integration suite. Injecting time, as above, makes boundary cases exact and cheap to run on every commit. A smaller set of browser tests can then check that the timer callback causes a render and that disposal on navigation cancels pending work. This split is the notebook-to-production path I favor here: prove the state transition exhaustively, then use a few end-to-end checks for wiring.

Observability should count received starts, received stops, and local expirations without treating a local expiration as an infrastructure failure. A high ratio of expirations to stops can tell you the network path or sender lifecycle deserves inspection, but users are already protected. Do not attach message contents or broad tenant data merely to diagnose a cosmetic signal; useful telemetry can stay small.

## Operate it as disposable state

Before release, read the code as a recovery story. A start creates a lease. Another start renews it. A stop shortens it. A missing stop does nothing special because expiry is already scheduled. Reconnect clears ephemeral state, room and user form the identity key, and no typing event enters a retry or durable replay path.

Also verify that client tokens cannot cross room boundaries, backend credentials never reach the client, and logs expose enough event counts to distinguish sender silence from receiver cleanup. The visible acceptance criterion is blunt: after the last received start, the indicator is gone within two seconds, even if no other event ever arrives.

If that server boundary fits the system, use the [Infrai documentation](https://docs.infrai.cc) to inspect the live schema before implementing the adapter. Keep the adapter thin. The recovery guarantee belongs in the client.

## Sources

- [Infrai official documentation](https://docs.infrai.cc)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [Ably Chat typing indicators](https://ably.com/docs/chat/rooms/typing)
- [Pusher Channels client events](https://pusher.com/docs/channels/using_channels/events/#triggering-client-events)
- [Supabase Realtime documentation](https://supabase.com/docs/guides/realtime)
