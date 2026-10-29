# Voice input

Transcription edits a composer draft. It does not submit an agent turn. Audio is
temporary client input, and only normal message submission sends the resulting
text. The current implementation transcribes locally on supported iOS devices;
environment-backed transcription is not implemented.

The [shared controller](../../packages/client-runtime/src/voice-input/controller.ts)
owns the operation while the client supplies capture and transcription. Preparation
binds the transcriber and resolved locale for the whole recording. Draft ownership,
text, and revision are captured before recording and checked before insertion, so
a late transcript cannot overwrite a draft that was edited or replaced.

Cancellation invalidates a result immediately, but resources stay owned until the
underlying work settles. Apple's native transcription call cannot be interrupted
once started. Releasing the session or deleting its recording when the abort signal
fires would race that work. The [transcription contract](../../packages/client-runtime/src/voice-input/transcription.ts)
therefore requires implementations to settle only after their work has stopped;
the [Apple binding](../../apps/mobile/src/native/voiceTranscription.ios.ts) checks
cancellation between native calls and discards late results.

## Live voice calls

A call is a Codex realtime session on the thread's native Codex thread. The client
owns the WebRTC media path to the provider; T3 relays only the SDP offer and answer.
[`LiveVoiceService`](../../apps/server/src/voice/LiveVoiceService.ts) is one
server-wide service keyed by thread, so a call outlives the WebSocket that started
it, and every subscription opens with a snapshot of live calls. While a call is
live it holds its provider session like a running turn, so idle release does not
end it.

Codex asks T3 to do real work through client-managed handoffs. The server runs the
request as an ordinary thread run with the settings the client last configured,
snapshotted when the handoff starts. The result, and check-ins for finished
subagents, can only reach the model through the browser's data channel, so the
owning client forwards each update once. A held call keeps that channel open and
its recording captures the spoken answer.

The [client registry](../../packages/client-runtime/src/liveVoice/registry.ts)
keeps one foreground call per environment; focus moves through hold, never a bare
mute. Finished phrases become runless thread rows. No run covers them, so a
provider switch adds them to the context handoff explicitly. The run's copy of a
spoken request is hidden, because its transcript row already shows it.
