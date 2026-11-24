# Local Voice Assistant Stack by Latency and Privacy Budget


> **Decision:** Use this guide when designing a local voice assistant against end-to-end latency, privacy, action approval, interruption, and failure budgets; skip it if you only want a speech-model list.
>
> **For:** local-AI builders, accessibility developers, and privacy-focused home-automation teams  ·  **Not for:** teams without a consent model or a way to measure each pipeline stage

A local voice assistant is a timed conversation, not a list of speech models. One utterance moves through capture, wake detection, speech activity detection, transcription, intent or model reasoning, action approval, synthesis, playback, and interruption. Each stage adds delay, error, data exposure, and a possible failure that the user must understand.

This guide follows that utterance and asks what to measure before calling a stack private, responsive, or useful. The included [`latency-budget.csv`](assets/latency-budget.csv) is the load-bearing artifact. It records observed p50/p95 timing, device, language, confidence, data egress, fallback, and confirmation for every stage; no result is prefilled.

## Establish a listening contract

Before selecting components, define:

- where microphones exist and who may be recorded;
- a visible or audible listening state;
- wake-only, push-to-talk, or continuous modes;
- maximum retained audio and transcript lifetime;
- whether audio, embeddings, transcripts, prompts, or diagnostics leave the device;
- supported speakers, accents, languages, noise, and accessibility needs;
- which actions require explicit confirmation;
- a hardware/software mute and a deletion path.

“Runs locally” is not enough. A local process may still send telemetry, download models, expose an API on the LAN, keep raw recordings, or pass text to a remote language model.

## The utterance path

### Capture and wake

[openWakeWord](https://github.com/dscripka/openWakeWord) is an upstream wake-word project. Measure false accepts per hour and missed activations in the actual acoustic environment. A wake model result should only open a short capture window; it should not authorize an action.

Keep a ring buffer only as long as necessary and disclose whether pre-roll audio is retained. Test television speech, children, guests, and similar phrases. Never hide the listening indicator merely to improve aesthetics.

### Voice activity detection

[Silero VAD](https://github.com/snakers4/silero-vad) is a speech-activity reference. VAD determines when speech may have started/stopped; it does not understand a completed request. Measure clipped first/last phonemes, interruption behavior, background music, overlapping speakers, and long pauses.

Author recommendation: budget endpointing separately from transcription. A “fast STT” result can be masked by slow end-of-speech detection.

### Speech recognition

[whisper.cpp](https://github.com/ggml-org/whisper.cpp) and [faster-whisper](https://github.com/SYSTRAN/faster-whisper) represent different local Whisper-family inference paths. Their documentation does not prove equal output, hardware support, memory, quantization behavior, or latency.

Record exact model artifact, runtime/commit, compute device, quantization, language setting, beam/sampling configuration, chunking, and whether partial transcripts are revised. Evaluate key entities and commands, not only aggregate word error. A confident wrong device name can be consequential.

### Intent, model, and action plan

[Home Assistant’s local voice documentation](https://www.home-assistant.io/voice_control/voice_remote_local_assistant/) describes an integrated local-assistant route. The [Home Assistant intents repository](https://github.com/home-assistant/intents) provides intent-related data. [Wyoming](https://github.com/rhasspy/wyoming) documents a voice-service protocol ecosystem.

Choose deterministic intents for narrow consequential controls where possible. If an LLM is introduced, separate its drafted interpretation from the action schema. Validate target, operation, parameters, and current state. Require a confirmation that names the exact effect for locks, purchases, messages, account changes, appliances with physical risk, or privacy-sensitive queries.

Untrusted speech can contain instructions aimed at the model. The model must not gain credentials, network destinations, or device capabilities from what it hears.

### Synthesis and playback

[Piper](https://github.com/OHF-Voice/piper1-gpl) and [Kokoro ONNX](https://github.com/thewh1teagle/kokoro-onnx) are local TTS references. [Speaches](https://github.com/speaches-ai/speaches) is an OpenAI-compatible local speech server reference. Check current licenses, model licenses, languages, voices, hardware, and API behavior before use.

Measure time to first audio and total synthesis separately. Support barge-in: a user must be able to stop playback and cancel a pending action. Do not clone or imitate a person’s voice without explicit authorization and a valid rights basis.

## Latency budget

Measure a monotonic clock at stage boundaries. At minimum:

1. wake phrase end → wake event;
2. speech start → first VAD event;
3. speech end → endpoint decision;
4. endpoint → stable transcript;
5. transcript → validated intent/plan;
6. plan → confirmation prompt;
7. confirmation → authorized tool result;
8. response text → first synthesized audio;
9. first audio → playback complete;
10. interruption → audio/action stop.

Report p50 and p95 over a declared corpus and device. Keep cold and warm runs separate. Record failures and timeouts rather than dropping them from latency statistics. Network and remote-provider time must be visible even when the rest of the stack is local.

## Privacy ledger

For each stage, record:

- data received and derived;
- in-memory and on-disk retention;
- process/user identity and listening interface;
- local socket, LAN, or internet destination;
- encryption and authentication;
- logs, telemetry, crash reports, and model downloads;
- deletion method and backup behavior;
- who can query history or replay audio;
- what happens during an outage.

Use synthetic utterances and fake entity names during integration. Inspect traffic rather than relying on a “privacy mode” label. Bind services to the narrowest interface and authenticate LAN APIs.

The [NIST Privacy Framework](https://www.nist.gov/privacy-framework) supplies a risk-management vocabulary, not a certification or a prefilled consent policy. The [FTC voice-cloning challenge notice](https://www.ftc.gov/system/files/ftc_gov/pdf/Voice-Cloning-Challenge.pdf) records concrete consumer-harm concerns, but does not define voice rights worldwide. Apply the law, consent standard, household policy, and accessibility requirements that govern the actual deployment.

## Component and transport map

| Layer | Candidates | Typical interface or artifact | What is supported by the linked source | What remains to test locally |
|---|---|---|---|---|
| Wake | openWakeWord | audio frames to wake event | wake-word project role | microphone format, model license, false accepts/misses, device acceleration |
| Endpointing | Silero VAD | audio frames to speech probability/events | VAD project role | sample-rate adapter, clipping, pause policy, noise robustness |
| STT runtime | whisper.cpp or faster-whisper | audio to text/segments | local Whisper-family inference role | model-format conversion, output equivalence, languages, hardware, quantization |
| Voice service transport | Wyoming | event/message protocol used by its ecosystem | protocol and service role | exact component version, discovery, authentication, encryption, retry semantics |
| Intent vocabulary | Home Assistant intents | structured sentences and intent data | intent-data role | supported language/domain coverage and application-specific slot validation |
| Integrated route | Home Assistant local voice | configured assistant pipeline | documented integrated path | chosen wake/STT/TTS combination, hardware budget, LAN boundary, action permissions |
| TTS runtime | Piper or Kokoro ONNX | text to audio | local synthesis role | voice/model license, pronunciation, languages, output format, compute demand |
| Speech API | Speaches | OpenAI-compatible speech service surface | local server/API role | endpoint coverage, auth/bind defaults, model compatibility, telemetry and failure mode |

Appearance in adjacent rows is not proof of compatibility. Record every adapter, socket, model format, license, authentication setting, and observed result in the worksheet before claiming a working stack.

## Failure ladder

| Failure | Safe response |
|---|---|
| uncertain wake/VAD | do not retain or act; provide a clear local indication |
| low-confidence or conflicting transcript | ask for repetition without guessing the consequential entity |
| unsupported language/entity | state the limit; do not silently map to a similar device/person |
| model/tool unavailable | offer a no-action fallback; do not queue an ambiguous future effect |
| confirmation mismatch | cancel and restate the proposed action |
| playback interrupted | stop speech and define whether the pending action is canceled |
| logging/storage full | follow the declared fail mode; never expose recordings through a fallback path |
| network unexpectedly required | disclose and hold rather than silently sending data |

## Evaluation corpus

Build an authorized corpus containing quiet/noisy rooms, near/far microphone positions, household voices who consented, supported accents/languages, similar device/contact names, negation, corrections, interruptions, and television/background speech. Keep a separate held-out set.

Measure wake false accepts/misses, endpoint clipping, key-entity accuracy, full task success, wrongful-action rate, p50/p95 latency, user correction turns, and data egress. Do not publish a universal ranking from one microphone and room.

## Consent and safety

- Obtain consent before recording or retaining another person’s voice.
- Make listening and retention visible and understandable.
- Do not use a cloned or recognizable voice without authorization.
- Do not infer identity, emotion, health, or protected traits merely because audio is available.
- Require independent confirmation and bounded permissions for consequential actions.
- Give users a reliable mute, cancel, history, and deletion route.
- Keep children, guests, shared spaces, and accessibility needs in the threat and usability model.
- This guide is not medical, emergency, legal, or security-system assurance.

## Evidence and maintenance

The ten precise records in [`sources/sources.json`](sources/sources.json) support component/protocol roles. The latency method, privacy ledger, failure ladder, and confirmation policy are author recommendations, not upstream guarantees.

Re-run when microphone, room, wake/VAD/STT/TTS model, runtime, quantization, language, intent set, LLM, service topology, tool permissions, or retention changes. Preserve cold/warm and failed results with their exact tuple.

## Adjacent guides

- [local-code-model-memory-planner](https://github.com/Ricardgu/local-code-model-memory-planner) — Use it when local reasoning or transcription models must fit a workstation or service memory budget.
- [open-source-agent-workflow-selector](https://github.com/Rubio199321/open-source-agent-workflow-selector) — Use it when approved voice actions must enter a durable or human-reviewed orchestration layer.
- [privacy-dns-stack-comparison](https://github.com/stormjilua/privacy-dns-stack-comparison) — Use it when the local assistant’s network path also needs explicit DNS privacy and routing layers.

## License

Original guide and latency worksheet: CC BY 4.0. Linked projects and documentation retain their terms.

If this saved you a decision, star the repo — it is how the next reader finds it.
