# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This repository is a general-purpose Claude workspace. It starts empty and is used as a working area for projects, experiments, and files created during Claude Code sessions. The OCI Data Science lab notebooks it previously contained have been removed.

## Conventions

- There is no build system, test suite, or linter at the workspace level. If a project added here introduces its own tooling, document its commands in this file.
- Keep the workspace organized: put each new project or experiment in its own top-level directory rather than scattering files at the root.
- Do not commit generated artifacts such as `.ipynb_checkpoints/`, caches, or large datasets (see `.gitignore`).
