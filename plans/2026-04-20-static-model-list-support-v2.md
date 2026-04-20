# Static Model List Support v2 - Optional Models Endpoint

## Objective

Make the `models` configuration truly optional across the codebase. When a provider does not have a models endpoint and no hardcoded models are provided, the system should gracefully return an empty list instead of failing with an error.

## Problem Statement

Currently, when a user configures a custom provider in `forge.toml` without specifying a models endpoint, the system fails with the error:

```
Provider models configuration is required
```

This prevents users from using providers that:
- Don't expose a `/models` endpoint
- Are context engines (like `forge_services`)
- Use models via deployment names rather than discovery

## Relevant Files

| File | Line | Issue |
|------|------|-------|
| `crates/forge_repo/src/provider/openai.rs` | 232 | `.ok_or_else()` requires models config |
| `crates/forge_repo/src/provider/anthropic.rs` | 231 | `.context()` requires models config |
| `crates/forge_repo/src/provider/google.rs` | 94 | Pattern matches on `Option` without `None` arm (will panic) |
| `crates/forge_repo/src/provider/bedrock.rs` | 280-287 | Already correct - returns `Ok(vec![])` when no models |

## Implementation Plan

### Phase 1: Update Model Retrieval Logic

- [~] **Task 1.1**: Modify `inner_models()` in `openai.rs` to handle missing models gracefully

  Change from:
  ```rust
  async fn inner_models(&self) -> Result<Vec<forge_app::domain::Model>> {
      let models = self
          .provider
          .models()
          .ok_or_else(|| anyhow::anyhow!("Provider models configuration is required"))?;

      match models {
          // ...
      }
  }
  ```

  Change to:
  ```rust
  async fn inner_models(&self) -> Result<Vec<forge_app::domain::Model>> {
      // For Vertex AI, use static JSON file
      if self.provider.id == ProviderId::VERTEX_AI {
          debug!("Loading Vertex AI models from static JSON file");
          return Ok(self.inner_vertex_models());
      }

      // Handle providers with no models configuration
      let Some(models) = self.provider.models() else {
          debug!("Provider has no models configuration, returning empty list");
          return Ok(vec![]);
      };

      match models {
          // ...
      }
  }
  ```
- [x] **Task 1.1**: Modify `inner_models()` in `openai.rs` to handle missing models gracefully

### Phase 2: Update Other Provider Implementations
- [x] **Task 2.1**: Fix `anthropic.rs` `models()` method (line 226-231)

  Change from:
  ```rust
  pub async fn models(&self) -> anyhow::Result<Vec<Model>> {
      let models = self
          .provider
          .models
          .as_ref()
          .context("Anthropic requires models configuration")?;

      match models {
          // ...
      }
  }
  ```

  Change to:
  ```rust
  pub async fn models(&self) -> anyhow::Result<Vec<Model>> {
      let Some(models) = self.provider.models.as_ref() else {
          debug!("Provider has no models configuration, returning empty list");
          return Ok(vec![]);
      };

      match models {
          // ...
      }
  }
  ```

- [ ] **Task 2.2**: Fix `google.rs` `models()` method (line 93-135)

  The current code uses `match &self.models` which will panic if `self.models` is `None`. Add a `None` arm:
  ```rust
  pub async fn models(&self) -> anyhow::Result<Vec<Model>> {
      match &self.models {
          Some(forge_domain::ModelSource::Url(url)) => {
              // ... existing URL handling ...
          }
          Some(forge_domain::ModelSource::Hardcoded(models)) => {
              debug!("Using hardcoded models");
              Ok(models.clone())
          }
          None => {
              debug!("Provider has no models configuration, returning empty list");
              Ok(vec![])
          }
      }
  }
  ```
- [x] **Task 2.2**: Fix `google.rs` `models()` method (line 93-135)

- [x] **Task 2.3**: Verify `bedrock.rs` already handles missing models correctly
- [x] **Task 2.3**: Verify `bedrock.rs` already handles missing models correctly

  Already implemented at lines 280-287 - no changes needed.

- [x] **Task 2.4**: Verify `opencode.rs` handles missing models correctly

  Already implemented at lines 128-142 - no changes needed.

### Phase 3: Documentation Update

- [x] **Task 3.1**: Update schema documentation in `forge.schema.json`

  Added clarification that `models` is optional for providers without `/models` endpoint.

- [x] **Task 3.2**: Add inline comments explaining the graceful fallback behavior

  Added comments like `// Handle providers with no models configuration` to all modified files.

## Testing Strategy

- [x] **Task 4.1**: Add test for provider without models configuration

  Added `test_provider_without_models_returns_empty` test in openai.rs.

- [x] **Task 4.2**: Add test for provider with empty static list

  Added `test_provider_with_empty_hardcoded_models_returns_empty` test in openai.rs.

## Verification Criteria

- [ ] Existing tests continue to pass (`cargo insta test --accept`)
- [ ] Code compiles without warnings (`cargo check`)
- [ ] A provider configured without `models` returns an empty list
- [ ] A provider configured with `models: []` (empty static list) returns an empty list
- [ ] A provider configured with a URL models endpoint still fetches models correctly
- [ ] A provider configured with hardcoded models still returns those models

## Risk Assessment

### Low Risk

The change is additive - we're making an existing optional field truly optional by:
1. Removing a forced error that contradicted the type system (`Option<T>` should be handled, not forced)
2. Returning a sensible default (empty list) instead of an error

Existing behavior is preserved for providers that DO have models configuration.

### Backward Compatibility

- **Breaking**: No - this is a bug fix that enables previously broken use cases
- **Migration**: Users who were hitting this error will now get empty model lists
- **Workaround removed**: Users who may have worked around this by adding fake models URLs can now omit the field entirely

## Alternative Approaches Considered

1. **Keep the error, require users to add fake endpoints**: Rejected - poor UX, forces users to configure non-functional endpoints

2. **Add a new `supports_models` boolean field**: Rejected - unnecessary complexity, the presence/absence of `models` already indicates this

3. **Return a special "no models available" sentinel**: Rejected - empty list is idiomatic Rust and matches standard collection semantics

**Selected Approach**: Graceful fallback to empty list - simple, idiomatic, and enables all use cases.
