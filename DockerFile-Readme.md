🐳 Deep Dive: The Frontend Dockerfile
This guide explains every line of the Dockerfile through the lens of Why we make these specific choices for a production-grade Next.js application.

Stage 1: Dependency Management (deps)
Goal: Create a clean environment to download the required libraries.

FROM node:20-alpine AS deps

What: Uses the Node.js 20 image based on Alpine Linux.

Why: We choose Alpine because it is incredibly small (~5MB), which makes your final image download faster. We use Node 20 because it is a stable Long-Term Support (LTS) version.

RUN apk add --no-cache libc6-compat

What: Installs a specific compatibility library.

Why: Alpine uses a different "C library" than most Linux versions. Many Node modules (like sharp for images) expect libc, so we add this to prevent crashes during installation.

WORKDIR /app

What: Creates and enters a folder named /app.

Why: It is best practice to keep your app in its own directory rather than the "root" of the container to avoid mixing your files with system files.

COPY ./src/frontend/package*.json ./

What: Copies only the package.json and package-lock.json.

Why: Docker Caching. Docker remembers each step. If you change your code but your libraries stay the same, Docker will skip the slow npm ci step and use the cached version from last time.

RUN npm ci

What: Installs dependencies.

Why: We use ci (Clean Install) instead of install because it ensures we get the exact versions locked in the package-lock.json, making the build predictable and stable.

Stage 2: The Build Room (builder)
Goal: Turn source code and .proto files into a runnable website.

FROM node:20-alpine AS builder

What: Starts a new stage with Node and Alpine.

Why: We start a fresh stage to keep things organized and ensure the build happens in a clean environment.

RUN apk add --no-cache libc6-compat protobuf-dev protoc

What: Installs the Protocol Buffer compiler.

Why: This specific repo uses gRPC to talk to other services. We need protoc to translate the demo.proto file into TypeScript files that our code can actually use.

COPY --from=deps /app/node_modules ./node_modules

What: Copies the libraries we just downloaded in Stage 1.

Why: This saves time. Instead of downloading them again, we just reuse what we already have.

COPY ./pb ./pb

What: Copies the protocol buffer definitions.

Why: The compiler needs these files to generate the necessary communication code for your services.

COPY ./src/frontend .

What: Copies your entire frontend source code.

Why: Now that we have the tools and libraries, we need the actual logic (the React components and API routes) to build the site.

RUN npm run grpc:generate

What: Runs the gRPC generation script.

Why: Without this, your code would have "broken links" because it wouldn't know how to talk to the Cart or Product services defined in your .proto file.

RUN npm run build

What: Compiles the Next.js app.

Why: This optimizes your code, minifies files, and prepares the "standalone" version of the site for the fastest possible performance.

Stage 3: The Production Runner (runner)
Goal: The final, secure container that goes live.

FROM node:20-alpine AS runner

What: A final fresh Node/Alpine environment.

Why: This is the most important "Why." By starting fresh, we leave behind all the "trash" (compilers, source code, caches) from previous stages. This makes the final image tiny and secure.

ENV NODE_ENV=production

What: Sets the environment to "production".

Why: Next.js and React perform much better in this mode because they turn off expensive developer warnings and debugging tools.

RUN addgroup -g 1001 -S nodejs & RUN adduser -S nextjs -u 1001

What: Creates a new, limited user account.

Why: Security. By default, Docker runs as "root." If a hacker finds a hole in your website, they would have full control. Running as a limited user (nextjs) prevents them from touching the rest of the system.

COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./

What: Copies the "baked" application.

Why: Next.js "standalone" mode includes only the files absolutely necessary to run. We use --chown to make sure our new nextjs user has permission to read these files.

USER nextjs

What: Switches from "root" to our limited user.

Why: To ensure the application actually runs with the restricted permissions we just set up.

ENTRYPOINT ["npm", "start"]

What: The command that runs the app.

Why: In this project, npm start is configured to run with Instrumentation.js, which automatically starts the OpenTelemetry monitoring so you can track errors and speed.
