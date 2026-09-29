---
layout: page
title: "Sequence Detector"
description: "A 52-page RTL interview handbook on sequence detectors: specifications, FSMs, SystemVerilog, verification, and PPA trade-offs."
permalink: /RTL/CommonQuestions/01_Sequence_Detector/
---

[Home]({{ '/' | relative_url }}) / [RTL]({{ '/RTL/' | relative_url }}) / [Common Questions]({{ '/RTL/CommonQuestions/' | relative_url }}) / Sequence Detector

*RTL Interview Prep, Volume 1 · Bharadwaj Sudula · September 2026*

A sequence detector recognizes a pattern in an incoming bit stream. This handbook follows the design from six specification questions through FSM construction, SystemVerilog, verification, and power, performance, and area (PPA).

**[Read the complete handbook (PDF, 52 pages)]({{ '/RTL/CommonQuestions/01_Sequence_Detector/assets/Sequence_Detector_Handbook.pdf' | relative_url }})**

<p><a href="{{ '/RTL/CommonQuestions/01_Sequence_Detector/assets/Sequence_Detector_Handbook.pdf' | relative_url }}" download="Sequence_Detector_Handbook.pdf">Download the handbook</a> (2.9 MB). The PDF includes the RTL listings, diagrams, waveforms, worked examples, and references.</p>

## Start with the specification

Before drawing the FSM, clarify:

1. Should matches overlap?
2. Should detection use Mealy or Moore timing?
3. Is the output a pulse or a level?
4. Which bit arrives first?
5. What are the reset timing and polarity?
6. Is there an input-valid signal, a programmable pattern, or a maximum pattern length?

## The central idea

Track the longest prefix of the target pattern that is also a suffix of the input seen so far. On each new bit, keep whatever partial match is still useful.

For `1011`, the transition from `S101` on a `0` is a useful test: the stream now ends in `10`, so the next state is `S10`. Returning to `IDLE` would discard a valid partial match.

The stream `1011011` also makes the overlap choice visible: an overlapping detector finds two matches, while a non-overlapping detector finds one.

<figure>
  <a href="{{ '/RTL/CommonQuestions/01_Sequence_Detector/assets/visual_drawn.svg' | relative_url }}">
    <img src="{{ '/RTL/CommonQuestions/01_Sequence_Detector/assets/visual_drawn.png' | relative_url }}" srcset="{{ '/RTL/CommonQuestions/01_Sequence_Detector/assets/visual_drawn.png' | relative_url }} 1x, {{ '/RTL/CommonQuestions/01_Sequence_Detector/assets/visual_drawn_2x.png' | relative_url }} 2x" width="1080" height="1350" style="display: block; width: 100%; height: auto;" loading="lazy" alt="An overlapping Mealy FSM for 1011 with states IDLE, S1, S10, and S101. The highlighted transition goes from S101 to S10 on input zero; a match returns to S1." />
  </a>
  <figcaption>The highlighted transition preserves the partial match. Select the illustration to open the scalable SVG.</figcaption>
</figure>

## Inside the handbook

| Topic | PDF page |
| --- | --- |
| How to use the handbook | [3]({{ '/RTL/CommonQuestions/01_Sequence_Detector/assets/Sequence_Detector_Handbook.pdf' | relative_url }}#page=3) |
| Cheat sheet | [4]({{ '/RTL/CommonQuestions/01_Sequence_Detector/assets/Sequence_Detector_Handbook.pdf' | relative_url }}#page=4) |
| The problem and the six specification questions | [5]({{ '/RTL/CommonQuestions/01_Sequence_Detector/assets/Sequence_Detector_Handbook.pdf' | relative_url }}#page=5) |
| The thought process: derivation, tables, and dry runs | [13]({{ '/RTL/CommonQuestions/01_Sequence_Detector/assets/Sequence_Detector_Handbook.pdf' | relative_url }}#page=13) |
| 24 interview framings, solved | [16]({{ '/RTL/CommonQuestions/01_Sequence_Detector/assets/Sequence_Detector_Handbook.pdf' | relative_url }}#page=16) |
| RTL coding and testbench practices | [41]({{ '/RTL/CommonQuestions/01_Sequence_Detector/assets/Sequence_Detector_Handbook.pdf' | relative_url }}#page=41) |
| PPA down to the transistor | [44]({{ '/RTL/CommonQuestions/01_Sequence_Detector/assets/Sequence_Detector_Handbook.pdf' | relative_url }}#page=44) |
| Rapid-fire questions, mistakes, practice, and glossary | [50]({{ '/RTL/CommonQuestions/01_Sequence_Detector/assets/Sequence_Detector_Handbook.pdf' | relative_url }}#page=50) |
| References | [52]({{ '/RTL/CommonQuestions/01_Sequence_Detector/assets/Sequence_Detector_Handbook.pdf' | relative_url }}#page=52) |

The worked framings include overlapping and non-overlapping detection, shift-register implementations, programmable patterns, long sync words, multiple patterns, input-valid bubbles, multiple bits per clock, glitch-free outputs, verification, and clock-domain crossings.

## Read on this page

<p>The full handbook is embedded below. If your browser does not display PDFs inline, <a href="{{ '/RTL/CommonQuestions/01_Sequence_Detector/assets/Sequence_Detector_Handbook.pdf' | relative_url }}">open the PDF directly</a> or use the download link above.</p>

<iframe src="{{ '/RTL/CommonQuestions/01_Sequence_Detector/assets/Sequence_Detector_Handbook.pdf' | relative_url }}#zoom=page-width" title="Sequence Detectors - The Interview Handbook, 52 pages" width="100%" height="760" loading="lazy" style="display: block; width: 100%; border: 1px solid #ddd; border-radius: 4px;"></iframe>

## Companion material

- [High-resolution illustration]({{ '/RTL/CommonQuestions/01_Sequence_Detector/assets/visual_drawn_2x.png' | relative_url }})
- [Vector illustration (SVG)]({{ '/RTL/CommonQuestions/01_Sequence_Detector/assets/visual_drawn.svg' | relative_url }})
- [Alternative AI-generated illustration]({{ '/RTL/CommonQuestions/01_Sequence_Detector/assets/visual_ai.png' | relative_url }})
- [LinkedIn post text]({{ '/RTL/CommonQuestions/01_Sequence_Detector/assets/post.txt' | relative_url }})

[Back to RTL Common Questions]({{ '/RTL/CommonQuestions/' | relative_url }})
