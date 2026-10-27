# A self-hosted complete voice assistant stack

v3rt3p connects a Yandex Station to speech recognition, an OpenAI-compatible language model, and speech synthesis.

This is a large self-hosted setup, not a one-click home automation add-on. Use it if you want to run the services yourself, choose the language model, keep audio processing under your control, or develop the individual services.

## Who this is for

- operators comfortable with Docker, DNS, logs, and service credentials;
- people who need a private, customizable voice interface;
- developers working on station protocols, ASR, TTS, or LLM integrations.

You will need a machine that can run several containers, storage for model assets, a PostgreSQL volume, and access to an OpenAI-compatible LLM. CPU ASR works for a basic setup. The ASR service also has a CUDA image; the [getting started guide](getting-started.md) explains the prerequisites.

## Start here

1. Read [Getting started](getting-started.md).
2. Read [Architecture](architecture.md) to get better understanding of innerworkings.
3. Use [Services](services.md) to get configuration guide and documentation regarding all the protocols.

The project is licensed under the GNU GPLv3. See [`LICENSE`](LICENSE).
