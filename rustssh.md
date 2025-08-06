# My Journey into Vibe Coding with Agentic AI: From Go to Rust in an Hour

The world of software development is ever-evolving, and the rise of AI is one of the most exciting recent developments. I recently embarked on a coding journey that took an unexpected, but fascinating, turn thanks to agentic AI. Here’s the story of how my simple SSH manager app idea blossomed into a testament to the power of "vibe coding" with an AI partner.

<iframe width="560" height="315" src="https://www.youtube.com/embed/1kWCu4cJ8XM?si=ZYklKNXb6YMqgFnc" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

Github Repo: https://github.com/mightywomble/sshtui

### The Spark of an Idea: An SSH Manager in the Terminal

It all started with a simple idea: an SSH manager app that I could spin up in my [warp.dev](http://warp.dev) terminal. I envisioned a clean Terminal User Interface (TUI) for adding SSH keys, and organizing hosts into groups. Something to make my life a little easier.

The initial development in Go was surprisingly quick. The core application was up and running in no time. But as with many projects, the devil was in the details.

-----

### The Frankenstein's Monster of a Virtual Terminal

The real challenge began when I tried to implement a virtual terminal. My initial attempt involved a side panel, which then evolved into a full-screen implementation. This solution, however, was clunky and ultimately launched a raw session. This was far from ideal because I couldn't get past the ANSI filtering to run essential tools like `htop` or even the humble `nano`. In the Go version, this meant a clunky user experience that required switching between different terminal modes: from the standard mode to a fullscreen mode, and then into a raw mode. This three-step process was far from the seamless experience I had envisioned.

I had a working app, but it was a Frankenstein's monster of code. It was functional, but not elegant.

-----

http://googleusercontent.com/image_generation_content/0

### A Glimmer of Hope: A Rust-y Solution

Feeling a bit stuck, I decided to consult an LLM (Sonnet) for its opinion. The AI suggested that Rust might have a better pseudo-terminal setup. The thought of migrating the entire application to a new language, especially with an AI, felt daunting. I wasn't in the mood for a night of migration, even with a digital assistant.

-----

### The Leap of Faith: AI-Powered Conversion

Then, a moment of inspiration struck. I was in my [warp.dev](http://warp.dev) terminal and thought, "Why not just ask the AI to do it?" I instructed the AI to convert my Go application to Rust, enabled the auto-answer option, and walked away to have dinner.

I returned an hour later, not knowing what to expect. To my astonishment, the AI had produced an almost like-for-like copy of my Go app, but in Rust. And the best part? It had a working SSH terminal. The very feature that had caused me so much grief was now functioning perfectly.

The Rust version provided a unified experience where the raw terminal worked directly in-panel. This was a significant technical achievement, as the Rust implementation demonstrated how to parse ANSI escape sequences using the `vte` crate, render styled terminal content within `ratatui` widget bounds, and coordinate between the TUI framework and raw terminal data. It also handled terminal resizing for both the UI and the SSH PTY and maintained precise cursor positioning and screen clearing within the panels.

This meant that I could now use `vim`, `nano`, or `emacs` perfectly within the terminal panel. Tools like `htop`, `top`, and `iotop` displayed correctly within the panel bounds, and `tmux` or `screen` sessions ran seamlessly. Even interactive shells like the Python REPL or database shells worked flawlessly, all while keeping the sidebar visible.

This experience was a revelation. It was a true "vibe coding" session where I set the direction, and the AI handled the heavy lifting. It was a glimpse into the future of software development, a future where human creativity and AI efficiency combine to create amazing things. This journey has not only given me a new and improved application but has also profoundly changed my perspective on the role of AI in the creative process of coding.