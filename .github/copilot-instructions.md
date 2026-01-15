# Copilot Instructions for UERenderNotes

## Project Overview

This is a technical documentation repository for Unreal Engine rendering study notes. It serves as a chapter-based Markdown knowledge base focused on:
- Material → HLSL shader workflows
- Shader compilation processes
- Rendering Dependency Graph (RDG)
- BasePass rendering
- Lighting and shadow systems
- Post-processing effects
- Nanite virtualized geometry
- Lumen global illumination
- Temporal Super Resolution (TSR)

The documentation follows a source-code-guided learning approach, making complex Unreal Engine rendering concepts accessible through detailed explanations and examples.

## Tech Stack

- **Language**: Markdown
- **Focus**: Technical documentation
- **Domain**: Unreal Engine rendering pipeline
- **Target Audience**: Graphics programmers, engine developers, and rendering engineers

## Documentation Guidelines

### Structure and Organization

- Organize content by topic into clear chapters or sections
- Use descriptive headings that reflect the content hierarchy (H1 for main topics, H2 for subtopics, etc.)
- Include a table of contents for longer documents
- Cross-reference related topics using relative links
- Keep file names descriptive and use kebab-case (e.g., `shader-compilation.md`)

### Writing Style

- Write in clear, technical English appropriate for experienced developers
- Define technical terms when first introduced
- Use code blocks with appropriate language tags (```cpp, ```hlsl, etc.)
- Include visual diagrams or flowcharts where helpful for understanding complex systems
- Provide concrete examples from Unreal Engine source code when relevant
- Balance depth with readability—explain WHY things work, not just HOW

### Code Examples

- Use proper syntax highlighting in code blocks
- Include comments in code examples to explain non-obvious parts
- Prefer complete, compilable examples over fragments when possible
- Reference specific UE source file paths when citing engine code
- Use consistent formatting within code blocks (follow UE coding standards)

### Technical Accuracy

- Verify technical details against Unreal Engine source code
- Specify the UE version when behavior is version-specific
- Include references to official documentation or source files
- Update content when engine versions change significantly
- Mark experimental or preview features clearly

### Markdown Best Practices

- Use standard Markdown syntax for maximum compatibility
- Prefer relative links for internal navigation
- Include alt text for images for accessibility
- Use tables for comparing features or listing attributes
- Employ lists (ordered/unordered) for sequential steps or related items
- Use blockquotes for important notes or warnings

### Content Quality

- Start each document with a brief overview of what will be covered
- Break down complex topics into digestible sections
- Include practical examples and use cases
- End with a summary or key takeaways for longer documents
- Proofread for technical accuracy, clarity, and grammar
- Keep content focused and avoid tangential discussions

### Graphics and Rendering Specifics

- Explain rendering concepts with both high-level overviews and implementation details
- Include shader code snippets with explanations of key operations
- Describe data flow through the rendering pipeline
- Reference specific render passes, buffers, and GPU resources by their UE names
- Explain performance implications of rendering techniques
- Compare different rendering approaches when relevant

## Task Guidelines for Copilot Agents

### When Creating New Documentation

- Research the topic thoroughly in UE source code before writing
- Follow the existing structure and style of other documentation in the repository
- Include practical code examples from the engine
- Cross-reference related topics appropriately
- Verify all technical claims against the actual source code

### When Updating Documentation

- Preserve the existing voice and style
- Verify changes don't contradict other parts of the documentation
- Update cross-references if topic structure changes
- Note if updates are specific to a particular UE version

### When Reviewing Documentation

- Check for technical accuracy against UE source
- Verify code examples are correct and properly formatted
- Ensure consistency with other documentation in the repository
- Confirm links are valid and point to correct locations
- Look for opportunities to improve clarity without changing meaning

## Common Patterns

### Document Structure Template

```markdown
# Topic Title

Brief introduction to the topic (1-2 paragraphs).

## Overview

High-level explanation of the concept.

## Implementation Details

### Subsection 1
Technical details with code examples.

### Subsection 2
More technical details.

## Examples

Concrete examples from UE source code.

## Performance Considerations

Discussion of performance implications.

## Related Topics

- [Related Topic 1](./related-topic-1.md)
- [Related Topic 2](./related-topic-2.md)

## References

- UE Source: `Engine/Source/Runtime/Renderer/Private/Example.cpp`
- Official Docs: [Link to UE documentation]
```

### Code Block Example

```cpp
// From BasePassRendering.cpp
void FDeferredShadingSceneRenderer::RenderBasePass(
    FRDGBuilder& GraphBuilder,
    const FSceneTextureParameters& SceneTextures)
{
    // Explain what this function does
    // Reference specific parameters and their purpose
}
```

## Quality Standards

- Technical accuracy is paramount
- Code examples must be syntactically correct
- Explanations should be clear to experienced C++ developers
- Visual aids enhance but don't replace written explanations
- Keep content up-to-date with current UE versions
- Maintain consistency across all documentation files

## Examples of Good vs. Bad Documentation

### ✅ Good Example
```markdown
## Shader Compilation Flow

The Unreal Engine shader compilation system transforms HLSL code into platform-specific bytecode through several stages:

1. **Preprocessing**: Material graphs are converted to HLSL (MaterialTemplate.ush)
2. **Compilation**: HLSL is compiled by platform-specific compilers (FShaderCompileJob)
3. **Caching**: Compiled shaders are stored in the DDC for reuse

```cpp
// From ShaderCompiler.cpp
void FShaderCompileThreadRunnable::CompileDirectlyThroughDll()
{
    // Platform-specific compilation happens here
    // Results are cached in Derived Data Cache
}
```

This approach allows for fast iteration during development while maintaining optimized shaders for shipping builds.
```

### ❌ Bad Example
```markdown
## Shaders

Unreal Engine compiles shaders. It's complicated. See the source code.
```

## Summary

When working on this repository, prioritize technical accuracy, clarity, and practical usefulness. Write for experienced developers who want to understand Unreal Engine's rendering pipeline in depth. Every piece of documentation should help readers understand not just what the engine does, but how and why it does it.
