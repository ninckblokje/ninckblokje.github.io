---
layout: post
title:  "BPM 11g migration"
tags:
- oracle
- bpm suite
- bpm
disqus: true
---
## The problem

Oracle BPM Suite is one of the BPMN engines which allow BPMN processes to be executed. These processen usually contains steps with human interaction which can take up from several minutes to days or even weeks. The entire BPMN process might take weeks or months to complete. But the BPMN process of course might need bug fixes, new process staps or even a complete restructuring.

So how do you handle existing BPMN process instances? If the BPMN process was just a piece of paper a (radical) change is quite easy. Just create a new piece of paper and distribute it. But handling a BPMN process instance which runs inside of a BPMN engine is a bit more complicated. Nothing must be lost, data must be added, steps must be added and so on and so forth.

## Software requirements

This article is about Oracle BPM Suite, version 11g (11.1.1.7). The following patches need to be installed either in BPM Suite or in JDeveloper (or both):

## Reading material

There is some documentation from Oracle, but I personally find the A-Team blogs more useful:

