# Specification Development Guide for Claude Code Integration

## Overview

This guide provides a structured approach to creating effective specifications for developing AYBIZA components using Claude Code. Following this methodology ensures consistent, high-quality implementations that align with architectural principles while leveraging the power of AI-assisted development.

## Why Specifications Matter

Well-crafted specifications serve multiple purposes:

1. **Clear Direction**: Provides unambiguous requirements that Claude Code can interpret correctly
2. **Consistent Structure**: Ensures all team members follow the same development patterns
3. **Implementation Efficiency**: Facilitates rapid, accurate code generation
4. **Documentation**: Serves as living documentation of system components
5. **Quality Assurance**: Creates a reference point for validation and testing

## Specification Structure

Each specification document should follow this standardized structure:

### 1. Title and Identification

```markdown
# [Feature Name] Specification
> Specification #[Number] - v[Version]
```

### 2. High-Level Objective

A concise statement of the overall goal, written as a single bullet point:

```markdown
## High-Level Objective

- Create a [specific feature] that enables [specific capability]
```

### 3. Mid-Level Objectives

A breakdown of the high-level objective into concrete, measurable steps:

```markdown
## Mid-Level Objectives

- Build a [specific component] with [specific behavior]
- Implement [specific functionality] for [specific purpose]
- Integrate with [existing system component]
- Ensure comprehensive test coverage
```

### 4. Implementation Notes

Technical details, constraints, and standards that guide implementation:

```markdown
## Implementation Notes

- Use [specific technologies/frameworks]
- Follow [specific coding standards]
- Ensure compatibility with [specific systems]
- Performance requirements: [specific metrics]
- Security considerations: [specific requirements]
```

### 5. Context

Define the starting point and ending state of the codebase:

```markdown
## Context

### Beginning Context
- `path/to/existing/file1.ext`
- `path/to/existing/file2.ext`

### Ending Context
- `path/to/existing/file1.ext` (modified)
- `path/to/existing/file2.ext` (modified)
- `path/to/new/file3.ext` (new)
```

### 6. Low-Level Tasks

Detailed, sequential steps for implementation, formatted for Claude Code:

```markdown
## Low-Level Tasks
> Ordered from start to finish

1. [First task description]
```claude
UPDATE path/to/file.ext:
    CREATE/UPDATE function_name():
        Implement specific functionality
        Handle specific edge cases
```

2. [Second task description]
```claude
CREATE path/to/new/file.ext:
    IMPLEMENT specific_class:
        Add required methods
        Ensure proper error handling
```
```

## Integration with Claude Code

This section explains how to use these specifications with Claude Code for efficient implementation.

### Step 1: Create a Specification

Follow the structure above to create a complete specification document. Save it with the next available number following the established AYBIZA documentation convention (e.g., `39_feature_name_specification.md`).

### Step 2: Implement with Claude Code

1. Start Claude Code in your development environment:
   ```bash
   claude
   ```

2. Present the specification:
   ```
   I'd like to implement this specification: [paste specification or provide file path]
   ```

3. Guide Claude through the implementation process:
   ```
   Please review this specification and implement it step by step, starting with the first low-level task.
   ```

### Step 3: Validation and Testing

After Claude completes the implementation:

1. Run all tests to verify functionality:
   ```
   Please run the appropriate tests to verify this implementation.
   ```

2. Review code quality:
   ```
   Please review the code for adherence to our coding standards, potential edge cases, and performance considerations.
   ```

## Specification Examples

### Example 1: HTML Slider Output and Charts Feature

```markdown
# HTML Slider Output and Charts Feature

## High-Level Objective

- Create an interactive HTML output format with dynamic word frequency visualization

## Mid-Level Objective

- Build new HTML output format with slider-based word frequency filtering
- Add radial bar and bubble chart visualizations for word frequencies
- Extend CLI to support new output formats
- Ensure comprehensive test coverage

## Implementation Notes
- No need to import any external libraries beyond existing dependencies
- Comment every new function
- For CLI commands add usage examples starting with `uv run main.py`
- Follow existing code patterns and type safety practices
- Add tests for all new functionality

## Context

### Beginning context
- `src/let_the_code_write_itself/output_format.py`
- `src/let_the_code_write_itself/chart.py`
- `src/let_the_code_write_itself/main.py`
- `src/let_the_code_write_itself/tests/output_format_test.py`
- `src/let_the_code_write_itself/tests/chart_test.py`

### Ending context
- `src/let_the_code_write_itself/output_format.py`
- `src/let_the_code_write_itself/chart.py`
- `src/let_the_code_write_itself/main.py`
- `src/let_the_code_write_itself/tests/output_format_test.py`
- `src/let_the_code_write_itself/tests/chart_test.py`

## Low-Level Tasks
> Ordered from start to finish

1. Add new HTML output format with slider
```claude
UPDATE src/let_the_code_write_itself/output_format.py:
    CREATE format_as_html_with_slider_filter() function:
        Add HTML template with slider control
        Add JavaScript for dynamic filtering
        MIRROR format_as_html()
```

2. Add new chart visualizations
```claude
UPDATE src/let_the_code_write_itself/chart.py:
    CREATE create_radial_bar_chart(word_counts: WordCounts), create_bubble_chart(...)
```

3. Update CLI interface
```claude
UPDATE src/let_the_code_write_itself/main.py:
    ADD support for checking .htmlsld extension and calling format_as_html_with_slider_filter()
        Be sure to use .html when saving the file, .htmlsld is just for checking
    ADD support for 'radial' and 'bubble' choices and calling respective chart functions
```

4. Add comprehensive tests
```claude
UPDATE test files:
    ADD test_format_as_html_with_slider_filter()
    ADD test_create_radial_bar_chart()
    ADD test_create_bubble_chart()
```
```

### Example 2: Transcript Analytics Feature

```markdown
# Transcript Analytics Specification

## High-Level Objective

- Create a CLI transcript analytics application

## Mid-Level Objective

- Build a python MVP typer CLI application
- Accept a path to a text file
- Count the frequency of each word in a file, filter out common words, and limit by count threshold
- Use an OpenAI chat completion with structured output to analyze the transcript and word counts
- Rich print the frequency of each word to the terminal and the transcript analysis

## Implementation Notes
- No need to import any external libraries (see pyproject.toml for dependencies)
- Comment every function
- For typer commands add usage examples starting with `uv run main <func name dash sep and params>`
- When code block is given in low-level tasks, use it without making changes (Task 4)
- Carefully review each low-level task for exact code changes

## Context

### Beginning context
- `src/spec_based_ai_coding/main.py`
- `pyproject.toml` (readonly)

### Ending context
- `src/spec_based_ai_coding/main.py`
- `pyproject.toml`
- `src/spec_based_ai_coding/llm.py` (new file)
- `src/spec_based_ai_coding/word_counter.py` (new file)
- `src/spec_based_ai_coding/data_types.py` (new file)
- `src/spec_based_ai_coding/constants.py` (new file)

## Low-Level Tasks
> Ordered from start to finish

1. Create common word blacklist
```claude
CREATE src/spec_based_ai_coding/constants.py: 
    CREATE COMMON_WORDS_BLACKLIST = ['the', 'and', ...add 50 more common words]
```

2. Create our data types
```claude
CREATE src/spec_based_ai_coding/data_types.py:

    CREATE pydantic types:

        WordCounts(BaseModel): {count_to_word_map: Dict[str, int]},

        TranscriptAnalysis(BaseModel): {
            quick_summary: str
            bullet_point_highlights: List[str]
            sentiment_analysis: str
            keywords: List[str]
        }
```

3. Create our word counter & filter out & limit by count threshold
```claude
CREATE src/spec_based_ai_coding/word_counter.py:
    CREATE word_counter(script: str, min_count_threshold: int = 10) -> WordCounts:
        Remove punctuation from script and make all words lowercase,
        Use the COMMON_WORDS_BLACKLIST to filter out common words,
        Only include words that are greater than the min_count_threshold.
        Sort descending by count.
```

4. Create our LLM function using the code block below
```python
# CREATE src/spec_based_ai_coding/llm.py: Use code block below no changes.

    from typing import List
    from pydantic import BaseModel
    from openai import OpenAI
    from .data_types import TranscriptAnalysis

    client = OpenAI()


    def analyze_transcript(transcript: str, word_count: dict) -> TranscriptAnalysis:
        completion = client.beta.chat.completions.parse(
            model="gpt-4o-mini",
            messages=[
                {
                    "role": "system",
                    "content": "You are a helpful assistant analyzing transcripts.",
                },
                {"role": "user", "content": f"{transcript}\n\nWord Count: {word_count}"},
            ],
            response_format=TranscriptAnalysis,
        )

        message = completion.choices[0].message
        if message.parsed:
            return message.parsed
        else:
            raise ValueError("Failed to parse the response")
```

5. Update our main function to use new count and analysis functions

```claude
UPDATE src/spec_based_ai_coding/main.py:
    CREATE a new typer cli application:
        CREATE @app.command() def analyze_transcript(path_to_script_text_file, min_count_threshold: int = 10):
            Read file, count words, run analysis, rich print results,
            print words like '<word>: ###' where ### is count 3
```
```

## Best Practices for Specification Development

### 1. Clarity and Precision

- Use precise, unambiguous language
- Define terms clearly, especially domain-specific terminology
- Specify input and output expectations explicitly
- Include constraints and edge cases

### 2. Progressive Refinement

- Start with high-level objectives
- Refine into mid-level objectives
- Break down into low-level tasks
- Define specific code changes for each task

### 3. Consistency with Architecture

- Align with established architectural patterns
- Reference existing components and interfaces
- Follow naming conventions
- Maintain separation of concerns

### 4. Testability

- Include testing requirements
- Specify expected behaviors
- Define edge cases to test
- Include performance expectations

### 5. Claude Code Optimization

- Use specific, actionable language in low-level tasks
- Include explicit file paths and function names
- Provide clear context with beginning and ending states
- Use the `claude` code block format for precise instructions

## Implementing a Complete Project with Specifications

For larger projects requiring multiple specifications:

### 1. Create a Project Plan

Start by developing a high-level project plan that identifies major components and their relationships:

```markdown
# Project Implementation Plan

## Components
1. Component A - [Brief description]
2. Component B - [Brief description]
3. Component C - [Brief description]

## Implementation Order
1. Specification #40: Component A
2. Specification #41: Component B
3. Specification #42: Component C

## Integration Points
- Component A provides [specific interface] for Component B
- Component B provides [specific data] for Component C
```

### 2. Develop Individual Specifications

Create a separate specification document for each component following the structure outlined in this guide.

### 3. Implement Sequentially

Implement each specification in the defined order, ensuring that dependencies are satisfied:

1. Implement Specification #40
2. Validate and test
3. Implement Specification #41
4. Validate and test
5. Implement Specification #42
6. Validate and test

### 4. Final Integration Testing

After all components are implemented, perform comprehensive integration testing:

```markdown
# Integration Test Plan

## Test Scenarios
1. [Scenario 1 description]
2. [Scenario 2 description]
3. [Scenario 3 description]

## Expected Results
1. [Expected result 1]
2. [Expected result 2]
3. [Expected result 3]
```

## Using Claude Code Advanced Features for Implementation

When implementing specifications with Claude Code, leverage these advanced features for optimal results:

### Extended Thinking

For complex design decisions, prompt Claude to engage extended thinking:

```
Please think about the best approach for implementing [specific component] considering [specific requirements].
```

### Multi-Claude Workflows

For larger specifications, consider using multiple Claude instances in parallel:

1. One instance to implement code
2. Another to write tests
3. A third to review and suggest improvements

### Visual Feedback

For UI components, use screenshots and visual references:

```
Here's a mockup of the desired UI. Please implement the HTML and CSS to match this design.
[paste or link to image]
```

### Context Engineering with CLAUDE.md

Create a CLAUDE.md file in your project root with important implementation details:

```markdown
# Project Guidelines

## Coding Standards
- [List of coding standards]

## Common Patterns
- [List of common patterns]

## Important Commands
- [List of important commands]
```

## Conclusion

This specification development guide provides a structured approach to creating clear, actionable specifications that leverage Claude Code's capabilities. By following this methodology, you can ensure consistent, high-quality implementations that align with AYBIZA's architectural principles and quality standards.

When creating specifications, remember:
- Start with clear objectives
- Provide detailed context
- Break down into explicit tasks
- Format for optimal Claude Code interpretation
- Include validation requirements

The resulting code will be consistent, maintainable, and aligned with project requirements.

## References

- [35_anthropic_web_search_tool.md](35_anthropic_web_search_tool.md)
- [36_claude_code_best_practices.md](36_claude_code_best_practices.md)
- [37_claude_code_tech.md](37_claude_code_tech.md)
- [38_claude_code_tutorials.md](38_claude_code_tutorials.md)