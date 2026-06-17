# WebGPU Three.js TSL eval rubric

Score each case from 1-5.

## API correctness

- 5: Uses current Three.js WebGPU and TSL APIs consistently with the repository references.
- 3: Mostly correct with minor version or import issues.
- 1: Uses obsolete WebGL-only or deprecated TSL patterns.

## Runnable structure

- 5: Produces code that has clear imports, initialization, render loop, and lifecycle handling.
- 3: Provides useful fragments but leaves integration gaps.
- 1: Produces disconnected pseudocode.

## Graphics reasoning

- 5: Explains shader, material, compute, or post-processing choices in implementation terms.
- 3: Explains the goal but misses important WebGPU constraints.
- 1: Gives vague graphics advice.

## Safety and scope

- 5: Avoids unrelated dependencies and warns about browser or device support limits.
- 3: Adds extra libraries without clear need.
- 1: Suggests unsafe browser flags or unsupported APIs as defaults.
