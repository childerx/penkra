# Provider Connections manual QA

Run this matrix in one freshly started canonical Penkra Dev instance after the
focused automated suites and production builds pass. Capture the visible result
and relevant sanitized server/runtime diagnostics for every failed step. Never
record credentials, auth URLs, tokens, or raw child environments.

## Preconditions

- Managed ChatGPT, Claude, and OpenCode installations report ready.
- Test credentials are entered by the operator at the UI boundary.
- Available accounts: two ChatGPT sign-ins, two Claude account sign-ins, one
  OpenAI Platform API key, one deferred Anthropic API-key case, and OpenCode
  Zen/Go keys as available.
- Start with a known new-user account, then repeat persistence checks with an
  existing user database migrated on this machine.

## New-user and anonymous route

1. Complete account creation without adding a Connection. Only a harness with
   a usable anonymous route appears in the composer.
2. Verify OpenCode shows the live Zen free models returned by managed discovery;
   it must not show a fake `Free` Connection.
3. Send a message with a free model. Success means the Thread starts with a null
   Connection binding and the chosen exact model, with no Connection prompt.
4. Quit and reopen the same Dev instance. The Thread and native continuation
   remain usable.

## Named Connections and isolation

1. Add every Connection without asking for a custom name. Account Connections
   must use the provider-returned email; key Connections must use the declared
   provider/API prefix plus the final four credential characters. Cancel each
   inline flow once before completing it; cancellation must leave no active
   Connection.
2. Start one Thread per Connection and send a unique harmless prompt.
3. Inspect sanitized launch diagnostics. Each launch must name the exact
   installation/Connection/native-state identities and must not expose another
   Connection's credential or profile path.
4. Quit, reopen, and continue each Thread. Each must resume its original native
   identity and Connection.
5. Sign out or disconnect one Connection. It disappears from available choices;
   a bound Thread fails through the standard provider error on its next send and
   does not switch automatically.

## Space defaults

1. Set distinct defaults for Personal and Work, then create a Thread in each.
   The initial bindings must match their Space defaults.
2. Disconnect a default while another active Connection exists. The Space must
   select the newest remaining active Connection; existing Thread bindings must
   not change.
3. Disconnect the only Connection for a harness. The Space must have no default
   for it. Add a new Connection and verify that the ordinary first-Connection
   rule makes it the default.
4. With OpenCode Go as the Space default, choose a Zen free model for a new
   Thread. The first send must use the anonymous route, not Go.

## Mid-Thread model and Connection changes

1. Select another compatible Connection but do not send. Navigate away and
   back: the pending choice remains scoped to that Thread and does not appear in
   another Thread.
2. Send the next message. Success means the new turn uses the selected
   Connection, the durable binding revision advances once, and the transcript
   adds `Connection changed to <name>` only after the switch succeeds.
3. Change only the model and send. The harness remains fixed and a model-change
   activity appears after success.
4. Select a different Connection while a turn runs, then send normally. It is a
   queued next turn; the active turn stays on its current Connection.
5. Repeat with the explicit steer action. The send interrupts through the
   provider's steering path and the steered turn uses the selected Connection.
6. Force target validation or startup failure. The prior binding and transcript
   must remain unchanged and the composer shows the standard error.

## Update and recovery

1. Make a managed provider update available. Startup/hourly checking should
   download and activate the verified official artifact only when no affected
   turn is active.
2. Restart during each recoverable operation in a disposable QA database:
   static-key creation, managed login, Connection switch, and native-state
   cleanup. On restart, each journal either completes exactly once or remains a
   visible failure; it never creates a replacement identity heuristically.
3. Re-run the Thread continuation and disconnection scenarios after update.

Anthropic API-key coverage is intentionally deferred. Claude account sign-in,
isolation, restart, disconnection, and continuation remain required for this
pass.
