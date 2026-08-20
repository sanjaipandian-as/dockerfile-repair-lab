# Debugging Notes

## Dockerfile Repair Lab Fixes

Below is a detailed log of the issues discovered in the original `Dockerfile` and `.dockerignore`, along with how they were resolved.

---

### 1. Base Image Issue
*   **Original Code:** `FROM node:notfound`
*   **Diagnostic / Break Family:** Base Image Break
*   **The Problem:** `node:notfound` does not exist on Docker Hub, causing the build to crash immediately at Step 1.
*   **The Fix:** Changed the base image to `node:20-alpine`.
*   **Why:** Provides a stable, lightweight, and secure official Node.js 20 environment on Alpine Linux.

---

### 2. Layer Ordering / Workdir / Caching Issue
*   **Original Code:**
    ```dockerfile
    COPY . .
    WORKDIR /wrong
    ```
*   **Diagnostic / Break Family:** WORKDIR & Paths / Cache Optimization
*   **The Problem:** 
    1. By copying all files to the default root directory before setting `WORKDIR`, they are placed in the wrong place.
    2. Modifying any source file invalidates subsequent steps in the build cache, meaning `npm install` would have to be run on every build.
*   **The Fix:** 
    1. Defined `WORKDIR /usr/src/app` first.
    2. Copied only `package.json` and `package-lock.json` first, ran `npm ci`, and then copied the remaining project files.
*   **Why:** Leverages Docker's build cache. If code changes, but dependencies don't, Docker reuses the cached layer containing installed dependencies, accelerating subsequent builds.

---

### 3. Dependency Installation Issue
*   **Original Code:** `RUN npm install package-lock.json`
*   **Diagnostic / Break Family:** Dependencies Break
*   **The Problem:** `npm install package-lock.json` tries to fetch a package named `package-lock.json` from the npm registry instead of using the file to install the actual dependencies.
*   **The Fix:** Replaced with `RUN npm ci`.
*   **Why:** `npm ci` (clean install) reads the existing `package-lock.json` and installs the exact versions of the dependencies (like `express`) cleanly and reliably in production environments.

---

### 4. Build Context / File Not Found Issue
*   **Original Code:** `COPY missing-folder ./missing-folder`
*   **Diagnostic / Break Family:** Build Context
*   **The Problem:** The folder `missing-folder` does not exist in the repository, making the build fail.
*   **The Fix:** Removed this instruction.
*   **Why:** Docker builds fail when a specified copy source is missing in the build context.

---

### 5. .dockerignore / Excluded Source Issue
*   **Original Code:** `.dockerignore` contained `src`
*   **Diagnostic / Break Family:** WORKDIR / Paths
*   **The Problem:** The `src` directory containing vital application files (`routes.js`, etc.) was ignored. As a result, the files were never copied into the image, and the application crashed on start because it couldn't find `./src/routes`.
*   **The Fix:** Removed `src` from `.dockerignore`.
*   **Why:** Allows Docker to copy the required source files to run the application inside the container.

---

### 6. Incorrect Startup Command
*   **Original Code:** `CMD ["npm", "run", "production"]`
*   **Diagnostic / Break Family:** Startup Command
*   **The Problem:** There is no `production` script in `package.json` (only a `start` script). The container would crash instantly upon startup.
*   **The Fix:** Changed the command to `CMD ["npm", "start"]`.
*   **Why:** Correctly runs the application startup command defined in `package.json`.
