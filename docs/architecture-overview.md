# Text Generation Web UI – Architecture Overview

This document is meant as a guided tour of the repository for newcomers. It highlights the most important entry points, how the server is composed, and what to explore next when extending the project.

## Top-level layout

- **`server.py`** – launches the Gradio application, wires the UI tabs together, and coordinates shared state such as the active model, tokenizer, and interface-level caches. It is the canonical entry point when running `python server.py` or the platform-specific start scripts.
- **Startup scripts (`start_*.sh`, `start_*.bat`, `cmd_*`)** – wrappers that create or activate the managed Python environment for the "one-click" installer, then dispatch to `server.py` with any persisted launch flags found in `user_data/CMD_FLAGS.txt`.
- **`modules/`** – houses the bulk of the Python code, split into submodules that cover model loading, inference, UI construction, and utilities.
- **`extensions/`** – optional packages that hook into the web UI through a lightweight plugin system. Extensions can contribute CSS/JS snippets, Gradio components, and background workers.
- **`docs/`** – end-user documentation for individual tabs and features. This `architecture-overview.md` file complements those with a developer-focused view.
- **`user_data/`** – persisted runtime assets such as downloaded models, settings, chat histories, and temporary caches. Anything in this folder is considered mutable user content rather than source code.

## Runtime bootstrap

`server.py` performs a small amount of setup before instantiating the Gradio interface:

1. Configure cache directories (`user_data/cache/gradio`) and environment variables to keep Gradio self-contained.
2. Import `modules.shared` to parse command-line arguments (including aliases read from `user_data/CMD_FLAGS.txt`) and expose global state containers for the currently loaded model, tokenizer, and UI flags.
3. Lazily import Gradio with a request blocker that disables outbound network calls during startup, then apply optional extras such as Matplotlib for LaTeX rendering.
4. Compose the full interface by delegating to the tab-specific builders in `modules/ui_chat.py`, `modules/ui_notebook.py`, `modules/ui_model_menu.py`, and related files. Each module registers Gradio components and connects them with callbacks defined in the generation and inference modules.

Signals (Ctrl+C) are trapped to ensure models with background threads, such as `LlamaServer`, shut down cleanly when the process exits.

## Key Python packages

### Shared configuration and utilities

- **`modules/shared.py`** – centralizes argument parsing, default UI settings, and global references to the current model, tokenizer, and session state. Other modules read and mutate these globals via the imported `shared` object.
- **`modules/utils.py`** – helper routines for launching the web UI, including a thin wrapper around Gradio that standardizes themes and component creation.
- **`modules/logging_colors.py`** – configures the colorful console logger used throughout the application.

### Model and generation stack

- **`modules/models.py`** and **`modules/loaders.py`** – detect and load models across supported backends (Transformers, llama.cpp, ExLlama V2/V3, TensorRT-LLM, etc.), manage GPU offloading, and expose unified load/unload functions.
- **`modules/text_generation.py`** and **`modules/chat.py`** – orchestrate prompt assembly, call the active model to stream tokens, and maintain per-conversation histories.
- **`modules/LoRA.py`** and **`modules/models_settings.py`** – apply Low-Rank Adaptation adapters and persist per-model configuration defaults.
- **`modules/training.py`** and **`modules/evaluate.py`** – provide entry points for fine-tuning and evaluation workflows when the UI is in training mode.

### Front-end composition

- **`modules/ui.py`** – defines base CSS/JS snippets, shared Gradio theme configuration, and helper functions used by every tab.
- **`modules/ui_chat.py`, `modules/ui_notebook.py`, `modules/ui_default.py`, `modules/ui_parameters.py`, `modules/ui_session.py`, `modules/ui_model_menu.py`** – each file builds a Gradio `Blocks` layout for a specific tab, registers event handlers, and binds inputs/outputs to generation callbacks.
- **Static assets (`css/`, `js/`)** – additional styling and browser-side behavior, such as syntax highlighting and tab switching, injected by the UI modules.

### Integrations and optional features

- **`modules/extensions.py`** – discovers extension packages under `extensions/`, importing any `script.py` files and exposing hooks (`apply_extensions`, `load_extensions`) so the core UI can aggregate contributed assets.
- **`extensions/`** – bundled examples include text-to-speech (`coqui_tts`), translation (`google_translate`), alternative OpenAI-compatible APIs, and utility tools such as `long_replies`.
- **`modules/web_search.py`**, **`modules/image_utils.py`**, and **`modules/html_generator.py`** – power advanced features like web search, multimodal prompts, and HTML export of conversations.

## Developer workflow tips

1. **Follow the control flow**: Start in `server.py` to understand how the UI is composed, then trace into the specific tab module you care about. From there, follow callback definitions into the generation stack (`modules/text_generation.py`) to see how prompts hit the model loader.
2. **Inspect shared state**: `modules/shared.py` defines the globals that glue components together. Searching for `shared.` usages is a quick way to see how configuration flows across modules.
3. **Leverage extensions**: For isolated experiments, create a new folder under `extensions/` with a `script.py` that registers UI elements via the provided hooks. This keeps prototypes separate from core code.
4. **Read the user docs**: The existing markdown files in `docs/` explain each tab from an end-user perspective, which helps clarify the UX expectations before changing UI behavior.

## What to explore next

- Dive into `modules/text_generation.py` to understand the request/response loop, including how streaming works and how sampling parameters are applied.
- Review `modules/models.py` and the loader-specific helpers (`exllamav3.py`, `transformers_loader.py`, `llama_cpp_server.py`) if you need to add a new backend or tweak loading heuristics.
- Study `modules/extensions.py` alongside a simple extension (e.g., `extensions/example`) to learn how to distribute optional features.
- If you plan to work on training workflows, familiarize yourself with `modules/training.py` and the accompanying UI defined in `docs/05 - Training Tab.md`.
- For API work, check the OpenAI-compatible implementation documented in `docs/12 - OpenAI API.md` and wired through the API extension.

With this map in hand, you should be able to jump to the relevant module for most tasks while keeping the overall architecture in mind.
