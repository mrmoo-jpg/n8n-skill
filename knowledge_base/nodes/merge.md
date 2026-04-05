# Merge Node
<!-- Node type: n8n-nodes-base.merge -->
<!-- Sources: Report 2 -->

## Overview
Combines data from multiple inputs. In agentic workflows, serves as the cyclical entry point for looping patterns.

## Key Pattern: Cyclical Entry Point

In Plan-and-Execute architecture, the Merge node enables visual loops:

- **Input 1**: Linear flow from Planner phase (initial entry)
- **Input 2**: Loopback from Switch node (continuation after Replanner)

This creates the cycle: Merge → Executor → State Update → Replanner → Switch → (back to Merge)

### Configuration
- Mode: Use default merge behavior
- The node accepts data from whichever input fires, enabling both initial entry and loop continuation
