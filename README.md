# How To Serve React Application With NGINX and Docker

Date Added: September 19, 2022 2:00 PM

<aside>
💡 This template documents how to review code. Helpful for new and remote employees to get and stay aligned.

</aside>

### **Step 1: Create React App Using Vite (Skip this step if you already have a react app)**

```bash
npm create vite@latest
```

You'll be asked for

- App Name
- Which Framework to use like React, Angular, or Vue? Choose React
- Then, Typescript or Javascript. Choose as you wish

Vite Project Initialization

Now switch to the project directory

```bash
cd [your project name]
```

### **Step 2: Update vite.config File**

This step is required to map the port between Docker container and your React app

Now replace this code snippet in `vite.config`

```jsx
export default defineConfig({
  plugins: [react()],
});

```

to

```jsx
export default defineConfig({
  plugins: [react()],
  server: {
    watch: {
      usePolling: true,
    },
    host: true, // needed for the Docker Container port mapping to work
    strictPort: true,
    port: 5173, // you can replace this port with any port
  }

```

### **Step 3: Create a Dockerfile**

Create a file called `Dockerfile` in the root of your project directory.

### **Step 4: Add Commands to Dockerfile**

Copy these commands to your Dockerfile

```docker
# Stage 0, "build-stage", based on Node.js, to build and compile the frontend
FROM tiangolo/node-frontend:10 as build-stage

WORKDIR /app

COPY package*.json /app/

RUN npm install

COPY ./ /app/

RUN npm run build

# Stage 1, based on Nginx, to have only the compiled app, ready for production with Nginx
FROM nginx:1.15

COPY --from=build-stage /app/build/ /usr/share/nginx/html

# Copy the default nginx.conf provided by tiangolo/node-frontend
COPY --from=build-stage /nginx.conf /etc/nginx/conf.d/default.conf
```

The explanation for these commands:

- This will tell Docker that we will start with a base image **node:10-alpine** which in turn is based on the [Node.js official image](https://hub.docker.com/_/node/), notice that you won’t have to install and configure Node.js inside the Linux container or anything, Docker does that for you:

```docker
FROM node:10-alpine as builder
```

- Our working directory will be `/react-nginx-docker`. This Docker "instruction" will create that directory and go inside of it, all the next steps will "be" in that directory:

```docker
WORKDIR /react-nginx-docker
```

- Now, this instruction copies all the files that start with `package` and end with `.json` from your source to inside the container. With the `package*.json` it will include the `package.json` file and also the `package-lock.json` if you have one, but it won't fail if you don't have it. Just that file (or those 2 files), before the rest of the source code, because we want to install everything the first time, but not every time we change our source code. The next time we change our code and build the image, Docker will use the cached "layers" with everything installed (because the `package.json`hasn't changed) and will only compile our source code:

```docker
COPY package.json package-lock.json ./
```

- Install all the dependencies, this will be cached until we change the `package.json` file (changing our dependencies). So it won't take very long installing everything every time we iterate in our source code and try to test (or deploy) the production Docker image, just the first time and when we update the dependencies (installed packages):

```docker
RUN npm install
```

- Now, after installing all the dependencies, we can copy our source code. This section will not be cached that much, because we’ll be changing our source code constantly, but we already took advantage of Docker caching for all the package install steps in the commands above. So, let’s copy our source code:

```docker
COPY . .
```

- Then build the React app with:

```docker
RUN npm run build
```

…that will build our app, to the directory `./build/`. Inside the container will be in `/app/build/`.

- In the same `Dockerfile` file, we start another section (another "stage"), like if 2 `Dockerfile`s were concatenated. That's Docker multi-stage building. It almost just looks like concatenating `Dockerfile`s. So, let's start with an [official Nginx base image](https://hub.docker.com/_/nginx/) for this "stage":

```docker
FROM nginx:alpine
```

…if you were very concerned about disk space (and you didn’t have any other image that probably shares the same base layers), or if, for some reason, you are a fan of Alpine Linux, you could change that line and use an [Alpine version](https://hub.docker.com/_/nginx/).

- Here’s the Docker multi-stage trick. This is a normal `COPY`, but it has a `-from=builder`. That `builder` refers to the name we specified above in the `as builder`. Here, although we are in a Nginx image, starting from scratch, we can copy files from a previous stage. So, we can copy the compiled fresh version of our app. That compiled version is based on the latest source code, and that latest compiled version only lives in the previous Docker "stage", for now. But we'll copy it to the Nginx directory, just as static files:

```docker
COPY --from=builder /react-nginx-docker/build /usr/share/nginx/html
```

- This configuration file directs everything to `index.html`, so that if you use a router like [React router](https://reacttraining.com/react-router/) it can take care of it's routes, even if your users type the URL directly in the browser:

```docker
COPY ./.nginx/nginx.conf /etc/nginx/nginx.conf
```

…that’s it for the `Dockerfile`! Doing that with scripts or any other method would be a lot more cumbersome.

## **Step 5: Build the Dockerfile**

Now we can build our image, doing it will compile everything and create a Nginx image ready for serving our app.

- Build your image and tag it with a name:

```bash
docker build -t react-nginx-docker .
```

## **Step 6: Run the Docker Container**

To check that your new Docker image is working, you can start a container based on it and see the results.

- To test your image start a container based on it:

```bash
docker run -p 80:80 react-nginx-docker
```

…you won’t see any logs, just your terminal hanging there.

- Open your browser in [http://localhost](http://localhost/).

## **Step 7: Open the App in the Browser**

Open the Browser and access `http://localhost:[Port you mentioned in the docker run command]` as per the configuration we did so far it should be `<http://localhost:8080>`