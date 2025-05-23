# Anthropic Official Documentation Reference Guide

## Purpose
This document consolidates all official Anthropic documentation URLs and key learnings for the AYBIZA platform implementation. Use this as a reference when implementing Claude 4 features.

## Core Documentation URLs

### 1. Extended Thinking
- **Main Doc**: https://docs.anthropic.com/en/docs/build-with-claude/extended-thinking
- **Cookbook**: https://github.com/anthropics/anthropic-cookbook/tree/main/extended_thinking
- **Tips**: https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/extended-thinking-tips

**Key Learnings**:
- Use `thinking` parameter with `budget_tokens` (minimum 1024)
- NOT time-based - it's token-based budgeting
- Start small and increase as needed
- Not compatible with temperature modifications

### 2. Prompt Caching
- **Documentation**: https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching
- **Notebook**: https://github.com/anthropics/anthropic-cookbook/blob/main/misc/prompt_caching.ipynb

**Key Learnings**:
- Default cache TTL: 5 minutes (NOT 1 hour)
- 1-hour caching is BETA feature
- Cache writes cost 25% more, hits cost 90% less
- Up to 4 cache breakpoints per request

### 3. Tool Use
- **Overview**: https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview
- **Code Execution**: https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/code-execution-tool
- **Web Search**: https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/web-search-tool
- **Cookbook**: https://github.com/anthropics/anthropic-cookbook/tree/main/tool_use

**Key Learnings**:
- Code execution requires beta header: `"anthropic-beta": "code-execution-2025-05-22"`
- Web search costs $10 per 1,000 searches
- Code execution limits: 1 GiB RAM, 5 GiB storage, 1 CPU
- Python 3.11.12 environment only

### 4. MCP (Model Context Protocol)
- **Introduction**: https://modelcontextprotocol.io/introduction
- **MCP Connector**: https://docs.anthropic.com/en/docs/agents-and-tools/mcp-connector
- **Remote Servers**: https://docs.anthropic.com/en/docs/agents-and-tools/remote-mcp-servers
- **Server Implementations**: https://github.com/modelcontextprotocol/servers

**Key Learnings**:
- Requires beta header: `"anthropic-beta": "mcp-client-2025-04-04"`
- Only URL-based servers (no local STDIO)
- NOT available on Amazon Bedrock
- Requires OAuth token authentication

### 5. AWS Bedrock Integration
- **Claude on Bedrock**: https://docs.anthropic.com/en/api/claude-on-amazon-bedrock

**Key Learnings**:
- Limited feature support compared to direct API
- Some features (MCP, Files API) not available
- Uses AWS authentication
- Model IDs confirmed correct

### 6. Agent Building
- **Agent Patterns**: https://github.com/anthropics/anthropic-cookbook/tree/main/patterns/agents
- **Building Effective Agents**: https://www.anthropic.com/engineering/building-effective-agents
- **Customer Support Example**: https://github.com/anthropics/anthropic-quickstarts/tree/main/customer-support-agent

**Key Learnings**:
- "Simplicity is paramount" - avoid over-engineering
- Start simple, add complexity only when needed
- Test extensively before adding features
- Prefer composable patterns over complex systems

### 7. Files API
- **Documentation**: https://docs.anthropic.com/en/docs/build-with-claude/files
- **PDF Support**: https://docs.anthropic.com/en/docs/build-with-claude/pdf-support

**Key Learnings**:
- 32 MB file size limit
- 100 GB total storage per organization
- NOT available on Bedrock
- Beta header required: `"anthropic-beta": "files-api-2025-04-14"`

### 8. Model Selection
- **Choosing a Model**: https://docs.anthropic.com/en/docs/about-claude/models/choosing-a-model
- **Claude 4 Best Practices**: https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/claude-4-best-practices

**Key Learnings**:
- Start with cheaper models (Haiku) and upgrade only if needed
- Balance intelligence, speed, and cost
- Test thoroughly before upgrading
- Be more explicit with Claude 4

### 9. API Reference
- **Messages API**: https://docs.anthropic.com/en/api/messages
- **Release Notes**: https://docs.anthropic.com/en/release-notes/api
- **Batch Processing**: https://docs.anthropic.com/en/docs/build-with-claude/batch-processing

### 10. Testing & Evaluation
- **Develop Tests**: https://docs.anthropic.com/en/docs/test-and-evaluate/develop-tests
- **Increase Consistency**: https://docs.anthropic.com/en/docs/test-and-evaluate/strengthen-guardrails/increase-consistency
- **Prompt Evaluations**: https://github.com/anthropics/anthropic-cookbook/tree/main/prompt_evaluations

### 11. Prompt Engineering
- **Overview**: https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview
- **Interactive Tutorial**: https://github.com/anthropics/anthropic-cookbook/tree/main/prompt_engineering_interactive_tutorial
- **Real World Prompting**: https://github.com/anthropics/anthropic-cookbook/tree/main/real_world_prompting

### 12. Multimodal/Vision
- **Vision Guide**: https://docs.anthropic.com/en/docs/build-with-claude/vision
- **Best Practices**: https://github.com/anthropics/anthropic-cookbook/blob/main/multimodal/best_practices_for_vision.ipynb
- **Chart Reading**: https://github.com/anthropics/anthropic-cookbook/blob/main/multimodal/reading_charts_graphs_powerpoints.ipynb

### 13. Embeddings & RAG
- **Embeddings**: https://docs.anthropic.com/en/docs/build-with-claude/embeddings
- **VoyageAI**: https://github.com/anthropics/anthropic-cookbook/blob/main/third_party/VoyageAI/how_to_create_embeddings.md
- **LlamaIndex**: https://github.com/anthropics/anthropic-cookbook/tree/main/third_party/LlamaIndex
- **MongoDB RAG**: https://github.com/anthropics/anthropic-cookbook/blob/main/third_party/MongoDB/rag_using_mongodb.ipynb

### 14. Computer Use
- **Documentation**: https://docs.anthropic.com/en/docs/agents-and-tools/computer-use
- **Demo**: https://github.com/anthropics/anthropic-quickstarts/tree/main/computer-use-demo

### 15. Admin & Prompt Tools
- **Admin API**: https://docs.anthropic.com/en/api/admin-api/users/get-user
- **Prompt Generate**: https://docs.anthropic.com/en/api/prompt-tools-generate
- **Prompt Improve**: https://docs.anthropic.com/en/api/prompt-tools-improve
- **Prompt Templatize**: https://docs.anthropic.com/en/api/prompt-tools-templatize

## Implementation Checklist

### Must Fix Immediately:
- [ ] Update extended thinking to use `budget_tokens` instead of time
- [ ] Clarify prompt caching is 5 minutes default (1 hour is beta)
- [ ] Add beta headers for code execution and MCP
- [ ] Update cost calculations for web search ($10/1000 searches)
- [ ] Note MCP and Files API not available on Bedrock

### Should Clarify:
- [ ] Verify "10 parallel tools" claim (no official documentation)
- [ ] Review model selection strategy (may be too aggressive with Opus 4)
- [ ] Add resource limits for code execution
- [ ] Update Files API limitations

### Best Practices to Emphasize:
- [ ] "Simplicity first" principle
- [ ] Start with cheaper models and upgrade only when needed
- [ ] Test thoroughly before adding complexity
- [ ] Be explicit with Claude 4 instructions

## Voice Agent Specific Considerations

While Anthropic's documentation doesn't specifically address voice agents, here are the relevant patterns:

1. **Latency Optimization**:
   - Use simpler models (Haiku) for acknowledgments
   - Reserve complex models for actual reasoning
   - Implement streaming for all responses

2. **Cost Management**:
   - Start with Haiku, upgrade only when needed
   - Use prompt caching for system prompts
   - Be aware of web search costs

3. **Tool Strategy**:
   - Simple acknowledgments during tool execution
   - Parallel execution not officially documented
   - Consider tool execution timeouts for voice

4. **Extended Thinking**:
   - May not be suitable for real-time voice
   - Use small token budgets if needed
   - Provide voice feedback during thinking

## Regular Review Schedule

This reference should be reviewed:
- Weekly: Check release notes for updates
- Monthly: Review cookbook for new patterns
- Quarterly: Full documentation review

Last Updated: [Current Date]
Next Review: [Date + 1 Week]