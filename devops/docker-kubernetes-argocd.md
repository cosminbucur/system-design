Docker packages a Java app into a portable, self-contained image — the standard starting point for "code on your laptop" to "running in production" for a modern Java service.

## 1. Multi-Stage Builds

Build the app in one stage (with the full JDK + build tool), then copy only the compiled artifact into a slim runtime image — keeps the final image small and avoids shipping build tools into production.

```dockerfile
# Stage 1: build
FROM eclipse-temurin:21-jdk AS build
WORKDIR /app
COPY . .
RUN ./gradlew bootJar --no-daemon

# Stage 2: runtime — only the JRE and the built jar, nothing else
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /app/build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Using a `-jre` (not `-jdk`) base image for the runtime stage removes the compiler and build tooling from the final image — smaller attack surface, smaller image, faster pulls.

## 2. Layer Caching — Don't Rebuild Everything on Every Code Change

Docker caches each layer; if a layer's inputs haven't changed, it's reused. Order instructions so rarely-changing steps (dependency resolution) come before frequently-changing ones (your source code).

```dockerfile
FROM eclipse-temurin:21-jdk AS build
WORKDIR /app
COPY build.gradle settings.gradle ./
RUN ./gradlew dependencies --no-daemon   # cached as long as dependencies don't change
COPY src ./src
RUN ./gradlew bootJar --no-daemon        # only this layer rebuilds when source changes
```

Without this ordering, copying source code before resolving dependencies means every code change invalidates the dependency-download layer too — dramatically slower builds.

## 3. JVM Container Awareness

Modern JVMs (10+) are container-aware by default — they read the container's cgroup memory/CPU limits, not the host machine's. Still, be explicit rather than relying entirely on defaults:

```dockerfile
ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
```

`-XX:MaxRAMPercentage` sizes the heap as a percentage of the container's memory limit, not a fixed `-Xmx` value — so the same image behaves correctly whether the container is given 512Mi or 4Gi. A JVM unaware of its container limits (older JVMs, or misconfigured ones) can see the host's full memory/CPU and size thread pools or heap far larger than the container is actually allowed to use, leading to OOM-kills that look mysterious from inside the app. This also matters for thread pool sizing.

## 4. Best Practices

| Practice                                   | Recommendation                                                                                                     |
| ------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| Use multi-stage builds                     | Never ship JDK/build tools in the runtime image — only the JRE and the built artifact.                              |
| Use a specific version tag, never `latest` | `latest` is a moving target — you lose reproducibility and can't tell what's actually running.                     |
| Order instructions for layer caching        | Rarely-changing steps (dependency resolution) before frequently-changing ones (source code) — otherwise every code change invalidates the dependency layer too. |
| Size the JVM heap as a percentage, not a fixed value | `-XX:MaxRAMPercentage` keeps the same image correct whether the container is given 512Mi or 4Gi. |
| Run as a non-root user                     | A container running as root that gets compromised has root inside the container — add `USER appuser` in the Dockerfile. |
| Keep images minimal                        | Fewer packages = smaller attack surface and faster pulls; avoid installing debugging tools into production images. |
| Use `.dockerignore`                        | Prevents `.git`, local build artifacts, and secrets from accidentally being copied into the build context/image.    |
