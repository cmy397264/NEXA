# ---- build stage ----
FROM eclipse-temurin:21-jdk AS build
WORKDIR /app

# settings.gradle 의 includeBuild '../nexa-security' 용. compose 의 additional_contexts 로 전달된다
COPY --from=nexa-security . /nexa-security

# Resolve dependencies first so they are cached between source changes
COPY gradlew settings.gradle build.gradle ./
COPY gradle ./gradle
RUN chmod +x gradlew && ./gradlew --no-daemon dependencies > /dev/null || true

COPY src ./src
RUN ./gradlew --no-daemon bootJar -x test

# ---- runtime stage ----
FROM eclipse-temurin:21-jre
WORKDIR /app
RUN useradd --system --uid 1001 spring
COPY --from=build /app/build/libs/*.jar /app/app.jar
USER spring
EXPOSE 8083
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
