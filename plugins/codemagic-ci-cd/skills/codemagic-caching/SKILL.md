---
name: codemagic-caching
description: Support skill for configuring and troubleshooting dependency caching on Codemagic. Covers cache configuration for Flutter, React Native, Native Android, Native iOS, and common package managers. Includes cache invalidation, common issues, and debugging steps.
---

# Dependency Caching on Codemagic

## How Caching Works

Codemagic caches directories between builds on the same workflow. The cache is stored per app per workflow — different workflows do not share a cache. Cache is restored at the start of a build and saved at the end.

Docs: https://docs.codemagic.io/yaml-basic-configuration/dependency-caching/

---

## Basic Configuration

```yaml
workflows:
  my-workflow:
    cache:
      cache_paths:
        - ~/.pub-cache          # Flutter/Dart packages
        - ~/.gradle/caches      # Android/Gradle
        - ~/Library/Caches/CocoaPods  # iOS CocoaPods
        - node_modules          # npm/yarn (relative to repo root)
```

Cache paths can be absolute or relative to the repository root.

---

## Cache Paths by Platform

### Flutter
```yaml
cache:
  cache_paths:
    - ~/.pub-cache
```

### Android / Gradle
```yaml
cache:
  cache_paths:
    - ~/.gradle/caches
    - ~/.gradle/wrapper
```

### iOS / CocoaPods
```yaml
cache:
  cache_paths:
    - ~/Library/Caches/CocoaPods
    - Pods   # relative path — the Pods directory in your repo
```

### React Native / npm
```yaml
cache:
  cache_paths:
    - node_modules
    - ~/.npm
```

### React Native / Yarn
```yaml
cache:
  cache_paths:
    - node_modules
    - ~/.cache/yarn
```

### Combined Flutter + Android + iOS
```yaml
cache:
  cache_paths:
    - ~/.pub-cache
    - ~/.gradle/caches
    - ~/.gradle/wrapper
    - ~/Library/Caches/CocoaPods
```

---

## Common Issues

### Cache not being used / build still slow
- Check that the cache paths match exactly where the package manager stores files
- Paths are case-sensitive on macOS
- Relative paths are resolved from the repository root, not the home directory
- Verify the workflow name hasn't changed — cache is keyed per workflow name

### Cache causing build failures / stale cache
- A corrupted or outdated cache can cause unexpected failures
- Fix: clear the cache manually in Codemagic UI → App settings → Caching → Clear cache
- After clearing, the next build will rebuild the cache from scratch

### CocoaPods cache not helping
- `~/Library/Caches/CocoaPods` caches the downloaded pod sources
- The `Pods/` directory itself (installed pods) should also be cached if `pod install` is slow
- Both paths together give the best result:
  ```yaml
  cache:
    cache_paths:
      - ~/Library/Caches/CocoaPods
      - Pods
  ```

### Gradle cache growing too large
- Gradle caches can accumulate over time and slow down cache restore/save
- Limit to `~/.gradle/caches` rather than the entire `~/.gradle` directory
- Avoid caching `~/.gradle/caches/build-cache-*` if builds are slow to save cache

### npm/yarn `node_modules` cache inconsistency
- Caching `node_modules` directly can cause issues if `package.json` changes between builds
- Better approach: cache `~/.npm` or `~/.cache/yarn` (the package download cache) and let `npm install` / `yarn install` restore from there
- This is slightly slower than caching `node_modules` but more reliable

### Cache not shared between branches
- This is expected behaviour — Codemagic cache is per workflow, not per branch
- Each branch on the same workflow shares the same cache

### Cache not available on first build
- Also expected — the cache is built up on the first run and available from the second build onwards

---

## Cache Invalidation

Codemagic does not automatically invalidate the cache when dependencies change. If `pubspec.yaml`, `package.json`, or `build.gradle` changes, the cached packages may be outdated.

Options:
1. **Clear cache manually** in App settings → Caching → Clear cache before the next build
2. **Disable caching temporarily** by removing `cache_paths` to do a clean build, then re-enable

There is no built-in cache key based on lock file hash (unlike some other CI systems). Manual clearing is the only option when the cache needs to be invalidated.

---

## Debugging Checklist

1. Are the cache paths correct for the package manager being used?
2. Has the workflow name changed? (cache is keyed to workflow name)
3. Is the cache being restored? (check build log for "Restoring cache" step)
4. Is the cache being saved? (check build log for "Saving cache" step)
5. Has the cache grown stale after a dependency update? (clear it manually)
6. Are relative paths correct? (resolved from repo root, not home directory)

---

## Key Documentation Links

- Dependency caching: https://docs.codemagic.io/yaml-basic-configuration/dependency-caching/
