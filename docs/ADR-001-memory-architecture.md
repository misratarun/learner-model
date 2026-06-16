# ADR-001: Memory Architecture for Longitudinal Student Modeling

## Status
Accepted

## Context

The core differentiation of this product is not tutoring. It is the accumulation of a structured model of how a specific child thinks, built from behavioral observation across sessions. This is what separates the product from generic AI assistants with chat memory, adaptive learning platforms that track performance scores, and human tutors whose model of the child is limited by session frequency and their own recall.

The technical question at this stage was: is a persistent, structured memory layer that accumulates behavioral knowledge about a specific child feasible with generative AI today?

The answer is yes, but feasibility depends on getting three things right. This document captures what those three things are and the reasoning behind the architectural direction chosen for each.

## Decision Drivers

- The memory layer must store patterns extracted from behavior, not raw session transcripts. A chat log is not a second brain. A structured model of observations extracted from that log is.
- The system must be able to reason against the memory layer in real time during a session.
- The memory layer must get more accurate over time, not noisier.
- The architecture must be buildable with the existing stack: LangChain, ChromaDB, Claude API, Python.

## The Three Problems and How We Are Approaching Each

### Problem 1: Persistent structured memory across sessions

This is solved with existing tools. After each session, an extraction step reads the session transcript and updates a structured student profile stored in a database. The profile contains observed patterns, not conversation history.

An observation looks like: "Student avoided problems where the first step was not immediately visible." Not: "In session 4, the student said they did not know where to start on problem 3."

At the start of each subsequent session, the relevant portions of the profile are injected into the system prompt. The LLM reasons against the profile during the session.

The profile schema is the critical design decision here. It cannot be a growing text file. It needs structure:

- Learning style signals, with confidence scores and observation count
- Persistent error patterns, organized by topic and type
- Avoidance behaviors, with the context in which they appear
- Intervention history, what was tried and whether it produced a change
- Open hypotheses that need more sessions to confirm or discard

A hybrid storage approach: vector store for retrieving semantically relevant past observations during a session, structured JSON profile for the core model updated after each session.

### Problem 2: Behavioral observation during problem solving

The system needs to distinguish between a child who got an answer wrong because they do not understand the concept, one who did not attempt the problem because the entry point was not visible, and one who started correctly then made a consistent error type.

This requires structured logging of every interaction during a session, not just the final answer. What did the student attempt, at what point did they stop, what did they ask for, how many hints before they moved on, did they attempt a similar problem differently.

This data feeds the memory extraction step after the session. The LLM can reason about these patterns accurately if the interaction data is captured cleanly. The challenge is instrumentation design, not AI capability.

### Problem 3: The memory layer getting more precise over time

Initial framing of this problem focused on noise accumulation and proposed confidence scoring and recency weighting as the mitigation mechanism.

This framing was revised. See ADR-002.

## Decision

Build the memory layer as a hybrid: structured JSON profile for the core student model, vector store for session-level observation retrieval. Profile is updated after each session through an LLM-driven extraction step that reads the session interaction log and updates the relevant fields. The profile schema is defined before any code is written and treated as a product surface, not an implementation detail.

## Consequences

The instrumentation design, specifically what gets logged during a session and at what granularity, is now a first-class product decision. It cannot be retrofitted after the tutoring interface is built.

The profile schema needs to be versioned from the start. As the product learns what signals are predictive and which are noise, the schema will evolve. Without versioning, early profile data becomes incompatible with later versions of the model.

The extraction prompt, the LLM instruction that reads a session log and updates the profile, is one of the most important prompts in the system. It needs its own evaluation framework.

## What Remains Open

- Final schema definition for the student profile
- Retrieval strategy for injecting relevant profile sections into the session prompt without exceeding context limits for longer-tenured students
- Evaluation framework for the extraction prompt
